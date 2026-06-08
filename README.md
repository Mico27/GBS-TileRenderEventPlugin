# gbs-TileRenderEventPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that lets you register scripts to run automatically every time the scroll engine renders a new column, a new row, or a full screen repaint of background tiles. This makes it possible to react to tile rendering with custom logic — for example, replacing specific tiles on the fly as they scroll into view, implementing persistent destructible tiles, or procedurally modifying a scene's appearance without metatiles.

`engineAlt` variants are included for compatibility when used alongside the [MetaTilePlugin](https://github.com/Mico27/gbs-MetatilePlugin), the [ScreenScrollPlugin](https://github.com/Mico27/gbs-ScreenScrollPlugin), or both at the same time.

All events are in the **Screen** group of the script editor.

https://github.com/user-attachments/assets/860d4490-b892-48a9-9bfb-fa75fca2777a

> This example is made with this plugin and the [SubmappingEx plugin](https://github.com/Mico27/GBS-copyBkgSubWinPlugin). Coins are background tiles (no metatiles). Triggers set a coin-collected flag and submap the coin tile away. When the player scrolls back, the render event fires for the newly-loaded column/row and applies the submap again if the coin was already collected — so the tile stays removed even after scrolling away and back.

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [How to Use](#how-to-use)
4. [Technicalities and Restrictions](#technicalities-and-restrictions)
5. [Events Reference](#events-reference)
6. [Inner Workings](#inner-workings)

---

## Concepts

### The Scroll Render Pipeline

GB Studio's scroll system keeps only the currently visible 20×18 tiles in VRAM at any one time. When the camera moves, the engine loads new rows and columns from ROM into the off-screen edge of the VRAM ring buffer. This plugin hooks into those load moments — **after** the ROM tile data is written — so your script can immediately overwrite or supplement whatever was just rendered.

### Three Render Event Slots

The plugin maintains three independent script slots:

| Slot | Index | Fires when |
|------|-------|------------|
| **Render column** | 0 | A single column of tiles is loaded (normal scroll, one tile at a time, or pending column flush). |
| **Render row** | 1 | A single row of tiles is loaded (normal scroll, one tile at a time, or pending row flush). |
| **Render all** | 2 | The entire visible screen is repainted (teleport, fast scroll, scene load repaint, or multi-tile jump). |

Each slot holds one script reference (`script_bank` + `script_addr`). Only one script per slot can be active at a time; registering a new script replaces the previous one. Slots are independent — you can have different scripts for columns and rows, or leave some slots empty.

---

## Project Setup

1. Install the plugin into your GB Studio project's `plugins` folder.
2. The two events will appear in the **Screen** group automatically.

---

## How to Use

### Registering a Render Script

In the scene's **On Init** script (it can run before the init fade), add an **Attach a Script to a tile rendering Event** event:

1. Choose the **Render Event** slot (column, row, or all).
2. Write the script to execute in the **On Render Event** tab.

The registered script will fire every time that render event occurs for the lifetime of the scene (or until it is explicitly removed).

### Typical Pattern: Persistent Tile Replacement

```
On Init:
  Attach a Script to a tile rendering Event → Render all
    → [Check flags, call Set background tile / Submap as needed]
  Attach a Script to a tile rendering Event → Render column
    → [Same check-and-replace logic]
  Attach a Script to a tile rendering Event → Render row
    → [Same check-and-replace logic]
```

Because "Render all" fires on the initial scene repaint and "Render column"/"Render row" fire as the player scrolls, the same replacement logic covers both the initial load and subsequent scroll-in events.

### Removing a Render Script

Call **Remove a Script from a tile rendering Event** on the same slot when the script is no longer needed (e.g. after all collectibles are already gone, to avoid unnecessary work each frame).

---

## Technicalities and Restrictions

### Script Is Not Re-Entrant

The `scroll_callback_execute` function checks whether the previous invocation of the script has finished (`SCRIPT_TERMINATED` flag) before starting a new one. If the render event fires again while the previous execution is still running, the new invocation is silently skipped. Keep render event scripts short and fast — they should complete well within a single frame.

### No Positional Context Passed

The render event scripts receive no arguments about *which* row or column was just rendered. If your script needs to know the current scroll position it must read `scroll_x`, `scroll_y`, or related engine fields from the script itself.

### "Render all" Fires Once Per Full Repaint

The "Render all" slot fires once at the end of a full-screen `scroll_render_rows` call, not once per row within it. If you need per-row logic during a full repaint you must handle it inside the column/row slots or design the "Render all" script to iterate over the whole screen.

### One Script Per Slot

Each slot holds exactly one script reference. Calling **Attach a Script to a tile rendering Event** on an already-occupied slot silently replaces the previous script without clearing the old execution handle.

### Script Runs Asynchronously

`script_execute` launches the script as a new concurrent thread. It does not block the scroll engine — tile data is already written to VRAM before the callback fires and the script begins executing on the next VM tick.

---

## Events Reference

Both events are in the **Screen** group and are allowed before the init fade.

---

### Attach a Script to a tile rendering Event

**`MICO_EVENT_RENDER_SCRIPT`**

Registers a sub-script to run every time the selected render event fires. The script is compiled as a sub-script and its bank/pointer are stored in the matching `render_events` slot at runtime.

| Field | Description |
|-------|-------------|
| Select Render Event | Choose which slot to attach the script to: **Render column** (slot 0), **Render row** (slot 1), or **Render all** (slot 2). |
| On Render Event (script) | The script to execute when the event fires. |

---

### Remove a Script from a tile rendering Event

**`MICO_EVENT_RENDER_SCRIPT_CLEAR`**

Unregisters the script from the selected slot by setting its `script_bank` to 0 and `script_addr` to `NULL`. After this call the slot is empty and the event will no longer trigger any script.

| Field | Description |
|-------|-------------|
| Select Render Event | The slot to clear: **Render column**, **Render row**, or **Render all**. |

---

## Inner Workings

### `render_events` Array

The plugin declares a static array of three `script_event_t` structs in `scroll.c`:

```c
script_event_t render_events[3];
```

Each entry holds:
- `script_bank` — ROM bank of the compiled sub-script.
- `script_addr` — pointer to the first instruction of the sub-script.
- `handle` — the execution handle returned by `script_execute`, used to detect when the previous run has terminated.

### `assign_render_script` / `clear_render_script`

These are the native functions called by the two events:

- `assign_render_script` reads the slot index, bank, and pointer from the VM stack and writes them into `render_events[slot]`.
- `clear_render_script` reads the slot index and zeroes out `script_bank` and `script_addr` for that slot.

### `scroll_callback_execute`

```c
static void scroll_callback_execute(UBYTE i)
{
    script_event_t *event = &render_events[i];
    if (!event->script_addr) return;
    if ((event->handle == 0) || ((event->handle & SCRIPT_TERMINATED) != 0))
    {
        script_execute(event->script_bank, event->script_addr, &event->handle, 0, 0);
    }
}
```

This is called from `scroll_viewport` immediately after each tile load operation:

- If `script_addr` is `NULL` the slot is empty and nothing happens.
- If the previous execution handle is still live (not yet terminated), the invocation is skipped to prevent re-entrance.
- Otherwise `script_execute` launches a new concurrent script thread. The `handle` output parameter is written back so future calls can check termination status.

### Where Each Slot Is Called in `scroll_viewport`

The `scroll_viewport` function is the heart of the scroll update loop. After every tile write operation it calls `scroll_callback_execute` with the appropriate slot:

| Condition | Tile write function | Callback slot |
|---|---|---|
| Column scroll by ±1 (parallax slice) | `scroll_load_col` | 0 (column) |
| Multi-column jump (parallax slice) | `scroll_render_rows` | 2 (all) |
| Column scroll by ±1 (main layer, pending flush) | `scroll_load_pending_col` | 0 (column) |
| Multi-column jump (main layer) | `scroll_render_rows` | 2 (all) |
| Row scroll by ±1 (pending flush) | `scroll_load_pending_row` | 1 (row) |
| Multi-row jump (main layer) | `scroll_render_rows` | 2 (all) |

The "Render all" callback is deliberately called only once per `scroll_render_rows` invocation (at the end of the branching logic that detected a multi-tile jump), not once per row written inside the function. This keeps the overhead proportional to scroll events rather than to screen size.
