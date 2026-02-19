# Screen Slot Selection — New Flow

## Overview

This document describes the updated flow for the **on-screen slot selection** step in the campaign creation process. A global control bar sits above the slot grid with a **type dropdown** and a **reset button**. Slots can be selected and edited in groups via a modal that overlays the entire screen.

---

## 1. Global Controls Bar

A control bar is always visible above the slot grid. It contains two controls:

### A. Type Dropdown

A single dropdown always available to the user with three options:

| Option | Description |
|---|---|
| **Image / Video** | Upload a static image or video file |
| **VAST Tag** | Paste a VAST XML/URL for programmatic video ads |
| **Web URL** | Enter a web page URL to render in the slot |

**Critical behaviour:** Selecting a type from the dropdown **resets all slots immediately**, regardless of what was previously configured in any slot. This is a destructive action — all existing slot assignments are wiped and the grid starts fresh under the new type.

- The dropdown is always enabled — the user can change the type at any point in the flow.
- Every time the type changes, all slots reset to empty. No slot retains its previous creative.
- A confirmation prompt may be shown before resetting if slots already have creatives assigned, to prevent accidental data loss.

### B. Reset Button

- A standalone **Reset** button that clears all slots back to their empty default state.
- Works independently of the dropdown — the user can reset without changing the type.
- After reset, the type dropdown retains its current selection but all slot creatives are cleared.

---

## 2. Slot Selection — Group Behaviour

After setting the type, the user selects which slots to configure:

- **Single click** a slot tile to select it.
- **Multi-select** by clicking multiple tiles (or using a range/select-all mechanism).
- Selected slots are visually highlighted.
- The user then opens the modal to assign a creative to the selected group.

---

## 3. Modal Behaviour — Overlay on Open

When the user triggers the edit action on selected slots, a modal opens:

### A. Full-Screen Overlay
- A dark semi-transparent backdrop covers the entire screen, including the full slot grid and all controls beneath.
- Nothing on the page is interactive while the modal is open.

### B. Empty Modal State
- The modal always opens completely empty — no pre-filled or previously saved state is shown.
- The modal context reflects the active type (e.g. *"Assign Image / Video — 3 Slots Selected"*).

### C. Modal Actions
- **Cancel** — closes the modal and lifts the overlay with no changes applied.
- **Save / Confirm** — applies the configured creative to all selected slots; disabled until valid input is provided.

---

## 4. User Flow — Step by Step

```
1. User lands on the Screen Slot Selection screen.
   └─ Slot grid is visible, all slots empty.
   └─ Control bar is visible with type dropdown and Reset button.

2. User selects a type from the dropdown (Image/Video, VAST, or Web URL).
   └─ ALL slots immediately reset to empty.
   └─ The grid is now fresh under the new type.

3. User selects one or more slot tiles on the grid.
   └─ Selected slots highlight visually as a group.

4. User triggers edit on the selected group.
   └─ Dark overlay covers the entire screen.
   └─ An empty modal appears on top, centred.

5a. User provides the creative and clicks Save.
    └─ Creative is applied to all selected slots.
    └─ Modal closes, overlay lifts.

5b. User clicks Cancel or the backdrop.
    └─ Modal closes, overlay lifts.
    └─ Selected slots remain empty.

6. User selects a different type from the dropdown at any point.
   └─ ALL slots reset again regardless of current assignment state.
   └─ Flow returns to step 3.

7. User clicks the Reset button.
   └─ All slots are cleared.
   └─ Type dropdown keeps its current value.
```

---

## 5. Key Design Rules

- **Type change = full reset** — switching the type wipes every slot, always. There is no partial reset.
- **Reset button clears creatives only** — the selected type is preserved after a manual reset.
- **Dropdown is always accessible** — there is no locked or read-only state for the type selector.
- **Modal is always empty on open** — no previously saved state is ever pre-loaded into the modal.
- **Overlay blocks everything** — the backdrop covers the full screen while the modal is open, preventing any slot interaction.
