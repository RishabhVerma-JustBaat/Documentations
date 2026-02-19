# Creative Upload & Resolution-Based Selection

## Overview

This document covers two features in the DOOH platform:

1. **Enhanced Creative Upload** — upload multiple image files into a single named creative group
2. **Resolution-Based Selection** — automatically filter and alternate creatives based on screen resolution when assigning to a campaign slot

---

## Feature 1: Creative Upload Flow

### What Changed

Previously, the upload modal accepted a single creative with basic metadata. The new flow introduces a **Creative Group** — a named bundle of one or more image files. Each file's resolution is captured automatically at upload time and stored alongside the asset.

### How It Works

1. Open the **Upload Creative** modal
2. Fill in the required fields:
   - **Group Name** — unique name for this creative bundle
   - **Duration (sec)** — default display duration for all files in the group
   - **Description** *(optional)*
3. Drag & drop **any number of image files** onto the drop zone, or click **Browse Files** to select multiple
4. Each file appears in a staging list showing:
   - Thumbnail preview
   - Filename
   - Auto-detected resolution (W × H px)
   - Remove button
5. Click **Save** — all files are uploaded and the group is created

### Resolution Detection

Resolution is detected client-side before upload by reading the image's natural pixel dimensions. This value is stored with each file and used later for slot filtering.

> ⚠️ If detection fails (e.g. corrupted file), a warning is shown and the user can enter the resolution manually.

---

## Feature 2: Resolution-Based Creative Selection

### How Filtering Works

When opening the **Select Creative** modal for a campaign slot:

- The slot knows which **screen** it belongs to, and that screen has a known resolution (e.g. `1920 × 1080`)
- The creative library is **automatically filtered** to show only groups that contain at least one file matching that exact resolution
- Groups with no matching files are hidden (they still exist and are available for other slots)
- No manual filtering is needed — it happens on modal open

### Alternation Logic

Once a Creative Group is selected for a slot, the platform determines what to display at runtime:

| Matching Files in Group | Behaviour |
|---|---|
| Exactly 1 | That file is always shown |
| More than 1 | Files are **alternated in round-robin order** across ad plays |

**Example:** A group assigned to a `1920 × 1080` screen contains 3 files at that resolution. Play 1 shows file A, play 2 shows file B, play 3 shows file C, play 4 shows file A again, and so on.

> **Note:** Resolution matching is exact (pixel-perfect). A `1920 × 1080` screen will not match a `1918 × 1080` creative.

---

## Data Model (Summary)

```
CreativeGroup
  ├── id
  ├── name
  ├── description
  ├── createdAt
  └── files[]
        ├── id
        ├── url
        ├── filename
        ├── resolution  { width, height }
        └── duration
```

---

## Edge Cases

- **No creatives match the screen resolution** — the modal shows an empty state with a prompt to upload a compatible creative
- **All files in a group are removed** — the group cannot be saved; at least one file is required
- **Duplicate filenames** — automatically renamed with a counter suffix (e.g. `banner(2).png`)
- **Multiple slots, same screen** — the alternation index is tracked per slot independently
