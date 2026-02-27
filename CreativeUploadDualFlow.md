# Creative File Dual-Mode Changes

This document describes all changes required to support **file-level** creative reporting alongside the existing **asset-level** reporting. The design uses two columns: `creative_id` (asset) and `creative_file_id` (file), with a simple fallback rule for backward compatibility.

---

## Table of Contents

1. [Overview](#overview)
2. [Design Principle](#design-principle)
3. [Schema Changes](#schema-changes)
4. [API Body Structure](#api-body-structure)
5. [Type Definition Changes](#type-definition-changes)
6. [Service Changes](#service-changes)
7. [Analytics & Reporting](#analytics--reporting)
8. [Migration Checklist](#migration-checklist)

---

## Overview

### Old Model (Asset-Centric)
- **CreativeAsset** = one creative with multiple aspect-ratio variants
- Selection: choose asset → system picks file by device aspect ratio
- Reporting: asset-level only (`creative_id`)

### New Model (File-Centric)
- **CreativeAsset** = group/container
- **CreativeFile** = individual creatives (one per aspect ratio or variant)
- Selection: user selects specific file directly
- Reporting: file-level when `creative_file_id` is present, else asset-level

### Backward Compatibility
- Old campaigns/playlists: `creative_file_id = NULL` → reporting uses `creative_id`
- New campaigns/playlists: both populated → reporting uses `creative_file_id`
- **Rule:** `COALESCE(creative_file_id, creative_id)` = reporting creative ID

---

## Design Principle

| Column | Meaning |
|--------|---------|
| `creative_id` | CreativeAsset ID (logical grouping, legacy) |
| `creative_file_id` | CreativeFile ID (actual executed creative for playback & reporting) |

**One-line rule:** *Reporting is file-level when available, otherwise asset-level.*

---

## Schema Changes

### 1. Prisma Schema (`prisma/schema.prisma`)

#### PlaylistItem (lines 310-311)
```prisma
model PlaylistItem {
  creativeId     String?  @map("creative_id")        # CreativeAsset ID
  creativeFileId String?  @map("creative_file_id")    # CreativeFile ID (NEW)
  creativeSrc    String?  @map("creative_src")
  # ...
  @@index([creativeId])
  @@index([creativeFileId])   # ADD: for query performance
}
```

#### ProofOfPlayEvent (lines 254-255)
```prisma
model ProofOfPlayEvent {
  creativeId     String?  @map("creative_id")        # CreativeAsset ID
  creativeFileId String?  @map("creative_file_id")    # CreativeFile ID (NEW)
  # ...
  @@index([creativeId])
  @@index([creativeFileId])   # ADD: for query performance
}
```

#### dooh_report_daily (lines 415-417)
```prisma
model dooh_report_daily {
  creative_id      String?  # CreativeAsset ID
  creative_file_id String?  # CreativeFile ID (NEW)
  # ...
}
```

#### dooh_report_hourly (lines 450-452)
```prisma
model dooh_report_hourly {
  creative_id      String?
  creative_file_id String?
  # ...
}
```

### 2. Database Migration (if not already applied)

If `creative_file_id` does not exist in your database, create a migration:

```sql
-- Add creative_file_id to playlist_items
ALTER TABLE "playlist_items" ADD COLUMN IF NOT EXISTS "creative_file_id" TEXT;

-- Add creative_file_id to proof_of_play_events
ALTER TABLE "proof_of_play_events" ADD COLUMN IF NOT EXISTS "creative_file_id" TEXT;

-- Add creative_file_id to dooh_report_daily
ALTER TABLE "dooh_report_daily" ADD COLUMN IF NOT EXISTS "creative_file_id" TEXT;

-- Add creative_file_id to dooh_report_hourly
ALTER TABLE "dooh_report_hourly" ADD COLUMN IF NOT EXISTS "creative_file_id" TEXT;

-- Optional: indexes for reporting queries
CREATE INDEX IF NOT EXISTS "playlist_items_creative_file_id_idx" ON "playlist_items"("creative_file_id");
CREATE INDEX IF NOT EXISTS "proof_of_play_events_creative_file_id_idx" ON "proof_of_play_events"("creative_file_id");
CREATE INDEX IF NOT EXISTS "dooh_report_daily_creative_file_id_idx" ON "dooh_report_daily"("creative_file_id");
CREATE INDEX IF NOT EXISTS "dooh_report_hourly_creative_file_id_idx" ON "dooh_report_hourly"("creative_file_id");
```

**Note:** Run `npx prisma migrate dev --name add_creative_file_id` to generate migration from schema, or apply the SQL above manually if needed.

---

## API Body Structure

### New Creative File Object (Flat Structure)

When returning creatives from the Creative Library API or when selecting creatives for campaigns/bookings:

```json
{
  "id": "1767098995350_JUSTBAAT_TEST_ASSET_FILE_RATIO_1_1",
  "creativeId": "1767098995350_JUSTBAAT_TEST_ASSET",
  "partnerId": "JUSTBAAT_TEST",
  "type": "VIDEO",
  "name": "1767098984580_creative6.mp4",
  "assetName": "N6",
  "aspectRatio": "1:1",
  "src": "https://jb-optimus.s3.ap-south-1.amazonaws.com/.../creative6.mp4",
  "resolutionHeight": 1920,
  "resolutionWidth": 1080,
  "status": "READY",
  "size": 4907336,
  "createdAt": "2026-02-18T10:04:17.022Z",
  "updatedAt": "2026-02-18T10:04:17.022Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | **CreativeFile ID** — use this as `creative_file_id` when saving |
| `creativeId` | string | **CreativeAsset ID** — use this as `creative_id` when saving |
| `assetName` | string | Asset/group name (for UI display) |
| `name` | string | File name |
| `src` | string | Media URL |

**Important:** No separate `creativeFileId` in the body — `id` **is** the file ID.

---

## Type Definition Changes

### 1. `src/types/playlist.types.ts`

```typescript
export interface PlaylistCreativeRef {
  creativeId?: string;      // Asset ID
  creativeFileId?: string;  // File ID (id from file object)
  src?: string;             // Fallback direct URL
}
```

### 2. `src/types/analytics.types.ts`

```typescript
export interface ProofOfPlayEvent {
  id: string;
  deviceId: string;
  playlistId: string;
  slotIndex: number;
  creativeId?: string;
  creativeFileId?: string;  // ADD
  campaignId?: string;
  event?: string;
  type?: string;
  positionMs?: number;
  durationMs?: number;
  timestamp?: string;
}
```

### 3. `src/types/booking.types.ts` (Booking Creative)

When creatives are file objects:
- `creative.id` = file ID
- `creative.creativeId` = asset ID

---

## Service Changes

### 1. Creative Library Service (`src/services/creative.library.service.ts`)

**Change:** Return flat array of files instead of nested structure.

```typescript
// In listByPartner - flatten files
const formattedFiles: any[] = [];
creatives.forEach((asset: any) => {
  if (asset.files && asset.files.length > 0) {
    asset.files.forEach((file: any) => {
      formattedFiles.push({
        id: file.id,
        creativeId: asset.id,
        partnerId: asset.partnerId,
        type: file.type,
        name: file.src?.split('/').pop() || file.id,
        assetName: asset.name,
        aspectRatio: convertAspectRatioFromPrisma(file.aspectRatio),
        src: file.src,
        resolutionHeight: file.resolutionHeight,
        resolutionWidth: file.resolutionWidth,
        status: file.status,
        size: file.size,
        createdAt: elasticDateFormat(file.createdAt),
        updatedAt: elasticDateFormat(file.updatedAt),
      });
    });
  }
});
```

### 2. Campaign Service (`src/services/campaign.service.ts`)

**Line ~723-726:** Fix typo `creativeFilesId` → `creativeFileId`, use `creativeId` from body:

```typescript
// BEFORE
creativeRef = {
  creativeId: creative.creativeAssetId,
  src: creative?.src,
  creativeFilesId: creative.id  // typo
};

// AFTER
creativeRef = {
  creativeId: creative.creativeId,   // Asset ID from body
  creativeFileId: creative.id,      // File ID (id = file ID)
  src: creative?.src,
};
```

**Line ~778:** Ensure `creativeFileId` is passed when building `item.creative`:

```typescript
creative: {
  creativeId: creativeRef.creativeId,
  creativeFileId: creativeRef.creativeFileId,
  src: creativeRef.src,
},
```

### 3. Playlist Service (`src/services/playlist.service.ts`)

**Line ~762-790 (`injectInPlaceForTargetSlot`):** Add `creativeFileId` when saving:

```typescript
await prismaClient.playlistItem.create({
  data: {
    // ...
    creativeId: item.creative?.creativeId,
    creativeFileId: item.creative?.creativeFileId,  // ADD
    creativeSrc: normalizedCreativeSrc,
    // ...
  },
});
```

**Line ~536-539 (`mapDbItemToTyped`):** Read `creativeFileId` when mapping:

```typescript
creative:
  db.creativeId || db.creativeFileId || db.creativeSrc
    ? {
        creativeId: db.creativeId || undefined,
        creativeFileId: db.creativeFileId || undefined,
        src: db.creativeSrc || undefined,
      }
    : undefined,
```

### 4. Booking Types (`src/types/booking.types.ts`)

**Line ~99 (`toPlaylistItem`):** Set both IDs when converting:

```typescript
creative: {
  creativeId: creative.creativeId ?? creative.creativeAssetId,
  creativeFileId: creative.id,
  src: creative.src || srcVal,
},
```

### 5. Analytics Service (`src/services/analytics.service.ts`)

**`logPlayback`:** Already stores `creativeFileId` if provided. Ensure devices send it:

```typescript
dataToInsert.push({
  // ...
  creativeId: event.creativeId && event.creativeId.trim() !== '' ? event.creativeId : null,
  creativeFileId: event.creativeFileId && event.creativeFileId.trim() !== '' ? event.creativeFileId : null,
  // ...
});
```

**`refreshHourlyAggregate` / `refreshDailyAggregate`:** Already include `creative_file_id` in CTEs and INSERTs (verify they match schema).

---

## Analytics & Reporting

### Reporting Query Pattern

Use `COALESCE(creative_file_id, creative_id)` for dual-mode reporting:

```sql
SELECT
  s.stat_date,
  s.campaign_id,
  s.device_id,
  s.playlist_id,
  s.slot_index,
  s.creative_id AS creative_asset_id,
  s.creative_file_id,
  COALESCE(s.creative_file_id, s.creative_id) AS reporting_creative_id,
  SUM(s.impressions) AS total_impressions,
  SUM(s.completes) AS total_completes,
  SUM(s.play_seconds) AS total_play_seconds,
  SUM(s.cost) AS total_cost
FROM dooh_report_daily s
WHERE s.campaign_id = :campaignId
  AND s.stat_date BETWEEN :fromDate AND :toDate
GROUP BY
  s.stat_date,
  s.campaign_id,
  s.device_id,
  s.playlist_id,
  s.slot_index,
  s.creative_id,
  s.creative_file_id
ORDER BY s.stat_date, s.slot_index;
```

### Row ID for Daily Aggregation

The `row_id` in `dooh_report_daily` must include `creative_file_id` when present for uniqueness:

```sql
CASE
  WHEN creative_file_id IS NULL THEN
    stat_date || '_' || device_id || '_' || campaign_id || '_' ||
    creative_id || '_' || playlist_id || '_' || slot_index
  ELSE
    stat_date || '_' || device_id || '_' || campaign_id || '_' ||
    creative_id || '_' || creative_file_id || '_' ||
    playlist_id || '_' || slot_index
END AS row_id
```

---

## Migration Checklist

### Schema
- [ ] Prisma schema has `creativeFileId` on `PlaylistItem`
- [ ] Prisma schema has `creativeFileId` on `ProofOfPlayEvent`
- [ ] Prisma schema has `creative_file_id` on `dooh_report_daily`
- [ ] Prisma schema has `creative_file_id` on `dooh_report_hourly`
- [ ] Run migration: `npx prisma migrate dev --name add_creative_file_id` (if columns missing)

### Types
- [ ] `PlaylistCreativeRef` includes `creativeFileId`
- [ ] `ProofOfPlayEvent` includes `creativeFileId`

### Services
- [ ] Creative Library: return flat file list with `id` and `creativeId`
- [ ] Campaign Service: fix `creativeFilesId` typo, pass `creativeFileId`
- [ ] Playlist Service: save and read `creativeFileId`
- [ ] Booking: set `creativeFileId` in `toPlaylistItem`
- [ ] Analytics: `logPlayback` stores `creativeFileId`

### Aggregation
- [ ] `refreshHourlyAggregate`: includes `creative_file_id` in SELECT, GROUP BY, INSERT
- [ ] `refreshDailyAggregate`: includes `creative_file_id` in SELECT, GROUP BY, INSERT, row_id

### Device/SDK
- [ ] Playback events include `creativeFileId` when available (from playlist item)

---

## Files Summary

| File | Changes |
|------|---------|
| `prisma/schema.prisma` | Already has `creativeFileId` / `creative_file_id` |
| `src/types/playlist.types.ts` | Add `creativeFileId` to `PlaylistCreativeRef` |
| `src/types/analytics.types.ts` | Add `creativeFileId` to `ProofOfPlayEvent` |
| `src/services/creative.library.service.ts` | Return flat file list |
| `src/services/campaign.service.ts` | Fix typo, pass `creativeFileId` |
| `src/services/playlist.service.ts` | Save/read `creativeFileId` |
| `src/types/booking.types.ts` | Set `creativeFileId` in `toPlaylistItem` |
| `src/services/analytics.service.ts` | Verify `creativeFileId` in logPlayback & aggregates |

---

## Quick Reference

| Context | creative_id | creative_file_id |
|---------|-------------|------------------|
| API body | `creativeId` (asset) | `id` (file) — no separate field |
| PlaylistItem | Asset ID | File ID |
| ProofOfPlayEvent | Asset ID | File ID |
| dooh_report_* | Asset ID | File ID |
| Reporting | Fallback | Primary when present |

**Rule:** `COALESCE(creative_file_id, creative_id)` = reporting creative ID.
