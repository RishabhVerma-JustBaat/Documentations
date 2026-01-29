
# DOOH Reporting Migration Guide (v2)

## 1. Introduction

This document explains the current DOOH reporting flow, its limitations, and the proposed optimized architecture using **hourly and daily pre-aggregated tables**. It is designed as a GitHub-ready README and follows an **incremental, low-compute architecture** to eliminate heavy scans on the `proof_of_play_events` table (≈14.4 GB).

----------

## 2. Current Reporting Flow

### 2.1 Overview

1.  **Proof of Play Events (POP)**
    
    -   Table: `proof_of_play_events`
        
    -   Stores raw playback-level events (`start`, `impression`, `complete`)
        
    -   Very large and continuously growing
        
2.  **Materialized View: `dooh_reporting_view`**
    
    -   Joins multiple entities:
        
        -   `devices`
            
        -   `campaigns`
            
        -   `creative_assets`, `creative_files`
            
        -   `screen_histories` (uptime)
            
    -   Aggregates data at playback level
        
    
    Refresh scheduler:
    
    ```sql
    REFRESH MATERIALIZED VIEW CONCURRENTLY dooh_reporting_view;
    
    ```
    
3.  **Reporting Queries**
    
    -   Dimensions: date, device, resolution, geo, campaign, creative
        
    -   Metrics: impressions, uptime %, cost, active campaigns
        

----------


  

### 2.2 Problems in Current Flow

  

1.  **Exponential Query Time Growth**

As `proof_of_play_events` grows, each materialized view refresh performs a full-table scan and multi-table joins. Runtime increases non‑linearly with data size, making the system increasingly fragile at scale.

  

2.  **Long-Running Jobs & Database Deadlocks**

Daily refresh jobs frequently run **18–21 hours**, holding locks on large relations and indexes. This leads to lock contention, blocked writes, and occasional deadlocks affecting production traffic.

----------

## 3. Optimized Reporting Flow

### 3.1 High-Level Design

**Goal:** Replace the monolithic materialized view with **incremental hourly and daily aggregates**.

Pipeline:

```
proof_of_play_events
        ↓        ↓
dooh_report_hourly (append-only)
        ↓
dooh_report_daily (reporting source)
        ↓
Reporting APIs

```

Key principles:

-   POP is scanned **only for new time windows**
    
-   Hourly table is **pipeline-internal**
    
-   Daily table is the **only reporting source**
    
-   No runtime joins in reporting APIs
    

----------

## 3.2 Table Schemas

### 3.2.1 Hourly Aggregate Table

> **Schema contract:** Hourly and Daily tables share the **exact same columns**, except `stat_hour`, which exists only in the hourly table. This guarantees 1:1 column correspondence and allows daily aggregation to be a pure roll-up with no schema transformation.

#### Create Table

```sql
CREATE TABLE dooh_report_hourly (
    stat_date DATE NOT NULL,
    stat_hour TIMESTAMP NOT NULL,
    device_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    creative_id TEXT,
    playlist_id TEXT,
    slot_index INT,

    device_name TEXT,
    partner_id TEXT,
    address_city TEXT,
    screen_size_in_inch INT,
    resolution_width INT,
    resolution_height INT,
    orientation TEXT,
    address_type TEXT,
    geo_latitude TEXT,
    geo_longitude TEXT,

    campaign_name TEXT,
    campaign_status TEXT,
    cost_type TEXT,
    cost_micro_amount BIGINT,

    creative_length INT,
    creative_format TEXT,

    uptime_pct NUMERIC,
    impressions INT,
    completes INT,
    play_seconds INT ,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

```

----------

### 3.2.2 Daily Aggregate Table (Reporting Source)

> Daily table contains **all hourly dimensions except hour**, plus aggregated metrics.

#### Create Table

```sql
CREATE TABLE dooh_report_daily (
    -- NOTE: Column list is identical to dooh_report_hourly except `stat_hour`
    id BIGSERIAL  PRIMARY KEY,
    stat_date DATE NOT NULL,
    device_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    creative_id TEXT,
    playlist_id TEXT,
    slot_index INT,

    impressions INT,
    completes INT,
    play_seconds INT,
    uptime_pct NUMERIC,
    cost NUMERIC,
    active_campaigns INT,

    device_name TEXT,
    partner_id TEXT,
    address_city TEXT,
    screen_size_in_inch INT,
    resolution_width INT,
    resolution_height INT,
    orientation TEXT,
    address_type TEXT,
    geo_latitude TEXT,
    geo_longitude TEXT,

    campaign_name TEXT,
    campaign_status TEXT,
    cost_type TEXT,
    cost_micro_amount BIGINT,
    creative_length INT,
    creative_format TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);


```

----------

## 3.3 Data Processing Pipeline

### 3.3.1 POP → Hourly Aggregation

#### Insert Query

```sql
INSERT INTO dooh_report_hourly (
    stat_date,
    stat_hour,

    device_id,
    campaign_id,
    creative_id,
    playlist_id,
    slot_index,

    impressions,
    completes,
    play_seconds,

    uptime_pct,

    device_name,
    partner_id,
    address_city,
    screen_size_in_inch,
    resolution_width,
    resolution_height,
    orientation,
    address_type,
    geo_latitude,
    geo_longitude,

    campaign_name,
    campaign_status,
    cost_type,
    cost_micro_amount,

    creative_length,
    creative_format
)
WITH pulse_expanded AS (
    SELECT
        DATE(sh.created_at) AS stat_date,
        sh.device_id,
        (p.pulse_elem ->> 'status') AS pulse_status
    FROM screen_histories sh
    CROSS JOIN LATERAL unnest(sh.pulses) p(pulse_elem)
),
uptime_calc AS (
    SELECT
        stat_date,
        device_id,
        COUNT(*) FILTER (WHERE pulse_status = 'ACTIVE')::NUMERIC
        / NULLIF(COUNT(*), 0) * 100.0 AS uptime_pct
    FROM pulse_expanded
    GROUP BY stat_date, device_id
),
start_events AS (
    SELECT DISTINCT ON (e.device_id, e."slotIndex", e."timestamp")
        e.device_id,
        e.campaign_id,
        e.creative_id,
        e.playlist_id,
        e."slotIndex" AS slot_index,
        e."timestamp" AS started_at
    FROM proof_of_play_events e
    WHERE lower(e.event) IN ('start', 'impression')
      AND e."timestamp" >= now() - interval '1 hour'
      AND e."timestamp" < now()
    ORDER BY
        e.device_id,
        e."slotIndex",
        e."timestamp",
        CASE WHEN lower(e.event) = 'start' THEN 1 ELSE 2 END
),
matched_events AS (
    SELECT
        s.*,
        (
            SELECT c."timestamp"
            FROM proof_of_play_events c
            WHERE lower(c.event) = 'complete'
              AND c.device_id = s.device_id
              AND c."slotIndex" = s.slot_index
              AND c."timestamp" >= s.started_at
              AND c."timestamp" <= s.started_at + (COALESCE(get_creative_duration(s.creative_id), 60) + 10) * INTERVAL '1 second'
            ORDER BY ABS(EXTRACT(EPOCH FROM (c."timestamp" - s.started_at)))
            LIMIT 1
        ) AS completed_at
    FROM start_events s
)
SELECT
    DATE(m.started_at) AS stat_date,
    DATE_TRUNC('hour', m.started_at) AS stat_hour,

    m.device_id,
    m.campaign_id,
    m.creative_id,
    m.playlist_id,
    m.slot_index,

    COUNT(*) AS impressions,
    COUNT(m.completed_at) AS completes,

    SUM(
        CASE
            WHEN m.completed_at IS NOT NULL
                THEN EXTRACT(EPOCH FROM m.completed_at - m.started_at)
            ELSE COALESCE(ca.duration, 0)
        END
    )::INT AS play_seconds,

    MAX(COALESCE(u.uptime_pct, 0.0)) AS uptime_pct,

    MAX(d.name) AS device_name,
    MAX(d.partner_id),
    MAX(d.address_city),
    MAX(d.screen_size_in_inch),
    MAX(d.resolution_width),
    MAX(d.resolution_height),
    MAX(d.orientation),
    MAX(d.address_type),
    MAX(d.geo_latitude),
    MAX(d.geo_longitude),

    MAX(c.name) AS campaign_name,
    MAX(c.status) AS campaign_status,
    MAX(c."costType") AS cost_type,
    MAX(c.cost_micro_amount),

    MAX(ca.duration) AS creative_length,
    MAX(cf.type) AS creative_format
FROM matched_events m
JOIN devices d ON d.id = m.device_id
JOIN campaigns c ON c.id = m.campaign_id
LEFT JOIN creative_assets ca ON ca.id = m.creative_id
LEFT JOIN creative_files cf ON cf.creative_asset_id = m.creative_id
LEFT JOIN uptime_calc u ON u.device_id = m.device_id
  AND u.stat_date = DATE(m.started_at)
GROUP BY
    DATE(m.started_at),
    DATE_TRUNC('hour', m.started_at),
    m.device_id,
    m.campaign_id,
    m.creative_id,
    m.playlist_id,
    m.slot_index;



```

----------

### 3.3.2 Hourly → Daily Aggregation

#### Insert Query

```sql
INSERT INTO dooh_report_daily (
                stat_date,
                device_id,
                campaign_id,
                creative_id,
                playlist_id,
                slot_index,

                impressions,
                completes,
                play_seconds,
                uptime_pct,
                cost,
                active_campaigns,

                device_name,
                partner_id,
                address_city,
                screen_size_in_inch,
                resolution_width,
                resolution_height,
                orientation,
                address_type,
                geo_latitude,
                geo_longitude,

                campaign_name,
                campaign_status,
                cost_type,
                cost_micro_amount,
                creative_length,
                creative_format
            )
SELECT
                stat_date,
                device_id,
                campaign_id,
                creative_id,
                playlist_id,
                slot_index,

               SUM(impressions) AS impressions,
               SUM(completes) AS completes,

                SUM(play_seconds
                )::INT AS play_seconds,

                MAX(uptime_pct) AS uptime_pct,

                MAX(cost_micro_amount) / 1000000.0 AS cost,

                COUNT(DISTINCT campaign_id)
                    FILTER (WHERE campaign_status = 'RUNNING') AS active_campaigns,

                MAX(device_name) as device_id,
                MAX(partner_id) as partner_id,
                MAX(address_city) as address_city,
                MAX(screen_size_in_inch) as screen_size_in_inch,
                MAX(resolution_width) as resolution_width,
                MAX(resolution_height) as resolution_height,
                MAX(orientation) as orientation,
                MAX(address_type) as address_type,
                MAX(geo_latitude) as geo_latitude,
                MAX(geo_longitude) as geo_longitude,

                MAX(campaign_name) as campaign_name,
                MAX(campaign_status) as campaign_status,
                MAX(cost_type) as cost_type,
                MAX(cost_micro_amount) as cost_micro_amount,
                MAX(creative_length) as creative_length,
                MAX(creative_format) as creative_format

            FROM dooh_report_hourly
            WHERE stat_date = CURRENT_DATE - 1
            GROUP BY
                stat_date,
                device_id,
                campaign_id,
                creative_id,
                playlist_id,
                slot_index;
```

----------

## 5. Reporting Model

All reporting APIs read **only from `dooh_playback_daily`**.

  
----------

## 6. Example Reporting Query

```sql
SELECT *
FROM dooh_playback_daily
WHERE stat_date BETWEEN '2026-01-01' AND '2026-01-22'
  AND device_id = 'D1'
  AND campaign_id = 'C1'
  AND creative_format = 'video';

```
