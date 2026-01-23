
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
        ↓
matched_playbacks (incremental window)
        ↓
dooh_playback_hourly (append-only)
        ↓
dooh_playback_daily (reporting source)
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
CREATE TABLE dooh_playback_hourly (
    stat_date DATE NOT NULL,
    stat_hour TIMESTAMP NOT NULL,
    device_id TEXT NOT NULL,
    campaign_id TEXT NOT NULL,
    creative_id TEXT,
    playlist_id TEXT,
    slot_index INT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,

    device_name TEXT,
    partner_id TEXT,
    address_city TEXT,
    screen_size_in_inch INT,
    resolution_width INT,
    resolution_height INT,
    orientation TEXT,
    address_type TEXT,
    geo_latitude DOUBLE PRECISION,
    geo_longitude DOUBLE PRECISION,

    campaign_name TEXT,
    campaign_status TEXT,
    cost_type TEXT,
    cost_micro_amount BIGINT,

    creative_length INT,
    creative_format TEXT,

    uptime_pct NUMERIC
);

```

----------

### 3.2.2 Daily Aggregate Table (Reporting Source)

> Daily table contains **all hourly dimensions except hour**, plus aggregated metrics.

#### Create Table

```sql
CREATE TABLE dooh_playback_daily (
    -- NOTE: Column list is identical to dooh_playback_hourly except `stat_hour`
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
    geo_latitude DOUBLE PRECISION,
    geo_longitude DOUBLE PRECISION,

    campaign_name TEXT,
    campaign_status TEXT,
    cost_type TEXT,
    cost_micro_amount BIGINT,
    creative_length INT,
    creative_format TEXT
);

```

----------

## 3.3 Data Processing Pipeline

### 3.3.1 POP → Hourly Aggregation

#### Insert Query

```sql
INSERT INTO dooh_playback_hourly (
    stat_date,
    stat_hour,
    device_id,
    campaign_id,
    creative_id,
    playlist_id,
    slot_index,
    started_at,
    completed_at,

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
    creative_format,

    uptime_pct
)
SELECT
    DATE(mp.started_at AT TIME ZONE 'Asia/Kolkata'),
    DATE_TRUNC('hour', mp.started_at AT TIME ZONE 'Asia/Kolkata'),

    mp.device_id,
    mp.campaign_id,
    mp.creative_id,
    mp.playlist_id,
    mp.slot_index,
    mp.started_at,
    mp.completed_at,

    d.name,
    d.partner_id,
    d.address_city,
    d.screen_size_in_inch,
    d.resolution_width,
    d.resolution_height,
    d.orientation,
    d.address_type,
    d.geo_latitude,
    d.geo_longitude,

    c.name,
    c.status,
    c."costType",
    c.cost_micro_amount,

    ca.duration,
    cf.type,

    COALESCE(u.uptime_pct, 0.0)

FROM matched_playbacks mp
JOIN devices d ON d.id = mp.device_id
JOIN campaigns c ON c.id = mp.campaign_id
LEFT JOIN creative_assets ca ON ca.id = mp.creative_id
LEFT JOIN creative_files cf ON cf.creative_asset_id = mp.creative_id
LEFT JOIN uptime_calc u
  ON u.device_id = mp.device_id
 AND u.stat_date = DATE(mp.started_at AT TIME ZONE 'Asia/Kolkata')

WHERE mp.started_at >= date_trunc('hour', now() - interval '1 hour')
  AND mp.started_at <  date_trunc('hour', now());

```

----------

### 3.3.2 Hourly → Daily Aggregation

#### Insert Query

```sql
INSERT INTO dooh_playback_daily (
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

    COUNT(*) AS impressions,
    COUNT(completed_at) AS completes,

    SUM(
        CASE
            WHEN completed_at IS NOT NULL
              THEN EXTRACT(EPOCH FROM completed_at - started_at)
            ELSE creative_length
        END
    )::INT AS play_seconds,

    MAX(uptime_pct) AS uptime_pct,

    SUM(cost_micro_amount) / 1000000.0 AS cost,

    COUNT(DISTINCT campaign_id)
        FILTER (WHERE campaign_status = 'RUNNING') AS active_campaigns,

    MAX(device_name),
    MAX(partner_id),
    MAX(address_city),
    MAX(screen_size_in_inch),
    MAX(resolution_width),
    MAX(resolution_height),
    MAX(orientation),
    MAX(address_type),
    MAX(geo_latitude),
    MAX(geo_longitude),

    MAX(campaign_name),
    MAX(campaign_status),
    MAX(cost_type),
    MAX(cost_micro_amount),
    MAX(creative_length),
    MAX(creative_format)

FROM dooh_playback_hourly
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
