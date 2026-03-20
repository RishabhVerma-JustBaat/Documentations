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


FIRST CAMPAIGN -- > 1765262250202_DOOH_JAGRAN_DIRECT_CAMPAIGN

