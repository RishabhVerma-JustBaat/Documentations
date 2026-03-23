# DOOH_JAGRAN Fix (SQL Only)

## Problem

Some rows in `playlist_items` have wrong `creative_id = 'DOOH_JAGRAN'`.
We need to:
- check counts per `(campaign_id, device_id)`
- update those bad rows to the expected `creative_id`

No slot-index mapping is used in this flow.

## 0) Backup target rows (before update)

Fill `VALUES` with the same `(campaign_id, device_id)` pairs you plan to update.

```sql
CREATE TABLE IF NOT EXISTS playlist_items_dooh_jagran_backup AS
SELECT *
FROM playlist_items
WHERE false;

WITH wanted(campaign_id, device_id) AS (
  VALUES
    ('1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1763625647938_3UXIFIYZ')
    -- , ('<campaign_id>', '<device_id>')
)
INSERT INTO playlist_items_dooh_jagran_backup
SELECT pi.*
FROM wanted w
JOIN playlists p
  ON p.target_type = 'DEVICE'
 AND p.target_id = w.device_id
JOIN playlist_items pi
  ON pi.playlist_id = p.id
 AND pi.campaign_id = w.campaign_id
 AND pi.creative_id = 'DOOH_JAGRAN';
```

## 1) Check counts (bad rows)

Fill `VALUES` with `(campaign_id, device_id)` pairs.

```sql
WITH wanted(campaign_id, device_id) AS (
  VALUES
    ('1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1763625647938_3UXIFIYZ')
    -- , ('<campaign_id>', '<device_id>')
)
SELECT
  w.campaign_id,
  w.device_id,
  COUNT(pi.id)::bigint AS bad_playlist_item_count
FROM wanted w
JOIN playlists p
  ON p.target_type = 'DEVICE'
 AND p.target_id = w.device_id
JOIN playlist_items pi
  ON pi.playlist_id = p.id
 AND pi.campaign_id = w.campaign_id
 AND pi.creative_id = 'DOOH_JAGRAN'
GROUP BY w.campaign_id, w.device_id
ORDER BY w.campaign_id, w.device_id;
```

## 2) Update bad rows

Fill `VALUES` with `(campaign_id, device_id, new_creative_id)` rows.

```sql
WITH wanted(campaign_id, device_id, new_creative_id) AS (
  VALUES
    ('1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1763625647938_3UXIFIYZ', '1764852429441_DOOH_JAGRAN_ASSET')
    -- , ('<campaign_id>', '<device_id>', '<correct_creative_id>')
)
UPDATE playlist_items pi
   SET creative_id = w.new_creative_id
  FROM wanted w
  JOIN playlists p
    ON p.id = pi.playlist_id
   AND p.target_type = 'DEVICE'
   AND p.target_id = w.device_id
 WHERE pi.campaign_id = w.campaign_id
   AND pi.creative_id = 'DOOH_JAGRAN';
```

## 3) Verify after update

Re-run the check query from step 1.  
Expected: `bad_playlist_item_count = 0` for all intended pairs.

## 4) Proof Of Play Events (`proof_of_play_events`)

Large table note: you mentioned ~25M rows in scope.  
Update only rows where `creative_id = 'DOOH_JAGRAN'` and set to the correct campaign creative.

### 4.1 Pre-check counts

```sql
SELECT
  campaign_id,
  COUNT(*)::bigint AS bad_rows
FROM proof_of_play_events
WHERE campaign_id IN (
  '1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN'
)
AND creative_id = 'DOOH_JAGRAN'
```

nearly 25M rows 

### 4.2 Backup target rows

```sql
CREATE TABLE IF NOT EXISTS proof_of_play_events_dooh_jagran_backup AS
SELECT *
FROM proof_of_play_events
WHERE false;

INSERT INTO proof_of_play_events_dooh_jagran_backup
SELECT *
FROM proof_of_play_events
WHERE campaign_id IN (
  '1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN'
)
AND creative_id = 'DOOH_JAGRAN';
```

### 4.3 Update bad POP rows (campaign → correct creative)

```sql
WITH campaign_creative_map(campaign_id, new_creative_id) AS (
  VALUES
    ('1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1764852429441_DOOH_JAGRAN_ASSET'),
    ('1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1768481200090_DOOH_JAGRAN_ASSET'),
    ('1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1768481246799_DOOH_JAGRAN_ASSET'),
    ('1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1768481304222_DOOH_JAGRAN_ASSET'),
    ('1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1770188095996_DOOH_JAGRAN_ASSET')
)
UPDATE proof_of_play_events poe
SET creative_id = m.new_creative_id
FROM campaign_creative_map m
WHERE poe.campaign_id = m.campaign_id
  AND poe.creative_id = 'DOOH_JAGRAN';
```

### 4.4 Verify after POP update

```sql
SELECT
  campaign_id,
  COUNT(*)::bigint AS remaining_bad_rows
FROM proof_of_play_events
WHERE campaign_id IN (
  '1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN'
)
AND creative_id = 'DOOH_JAGRAN'
GROUP BY campaign_id
ORDER BY campaign_id;
```

## 5) DOOH Report Daily (`dooh_report_daily`)

Apply the same campaign-to-creative correction on `dooh_report_daily`.

### 5.1 Pre-check counts

```sql
SELECT
  campaign_id,
  COUNT(*)::bigint AS bad_rows
FROM dooh_report_daily
WHERE campaign_id IN (
  '1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN'
)
AND creative_id = 'DOOH_JAGRAN'
GROUP BY campaign_id
ORDER BY campaign_id;
```

### 5.2 Backup target rows

```sql
CREATE TABLE IF NOT EXISTS dooh_report_daily_dooh_jagran_backup AS
SELECT *
FROM dooh_report_daily
WHERE false;

INSERT INTO dooh_report_daily_dooh_jagran_backup
SELECT *
FROM dooh_report_daily
WHERE campaign_id IN (
  '1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN'
)
AND creative_id = 'DOOH_JAGRAN';
```

Nearly 5k enteries

### 5.3 Update bad rows (campaign -> correct creative)

```sql
WITH campaign_creative_map(campaign_id, new_creative_id) AS (
  VALUES
    ('1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1764852429441_DOOH_JAGRAN_ASSET'),
    ('1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1768481200090_DOOH_JAGRAN_ASSET'),
    ('1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1768481246799_DOOH_JAGRAN_ASSET'),
    ('1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1768481304222_DOOH_JAGRAN_ASSET'),
    ('1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN', '1770188095996_DOOH_JAGRAN_ASSET')
)
UPDATE dooh_report_daily drd
SET creative_id = m.new_creative_id
FROM campaign_creative_map m
WHERE drd.campaign_id = m.campaign_id
  AND drd.creative_id = 'DOOH_JAGRAN';
```

### 5.4 Verify after update

```sql
SELECT
  campaign_id,
  COUNT(*)::bigint AS remaining_bad_rows
FROM dooh_report_daily
WHERE campaign_id IN (
  '1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481379635_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768481866818_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1768482110926_DOOH_JAGRAN_DIRECT_CAMPAIGN',
  '1770188126394_DOOH_JAGRAN_DIRECT_CAMPAIGN'
)
AND creative_id = 'DOOH_JAGRAN'
GROUP BY campaign_id
ORDER BY campaign_id;
```

