# gbs-TileRenderEventPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that lets you register scripts to run automatically every time the scroll engine renders a new column, a new row, or a full screen repaint of background tiles. This makes it possible to react to tile rendering with custom logic — replacing specific tiles on the fly as they scroll into view, implementing persistent destructible tiles, or procedurally modifying a scene's appearance without metatiles.

Compatibility variants are included for use alongside the [MetaTilePlugin](https://github.com/Mico27/gbs-MetatilePlugin), the [ScreenScrollPlugin](https://github.com/Mico27/gbs-ScreenScrollPlugin), or both at once.

https://github.com/user-attachments/assets/860d4490-b892-48a9-9bfb-fa75fca2777a

> This example uses this plugin together with the [SubmappingEx plugin](https://github.com/Mico27/GBS-copyBkgSubWinPlugin). The coins are background tiles, not metatiles. Triggers set a coin-collected flag and submap the coin tile away. When the player scrolls back, the render event fires for the newly loaded column or row and applies the submap again if the coin was already collected — so the tile stays removed even after scrolling away and back.

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [Memory Footprint](#memory-footprint)
6. [Bank 0 (HOME) Usage](#bank-0-home-usage)
7. [Changelog](#changelog)

---

## Concepts

### The scroll render pipeline

GB Studio's scroll system keeps only the currently visible 20×18 tiles in VRAM. When the camera moves, the engine loads new rows and columns from ROM into the off-screen edge. This plugin hooks into those moments — **after** the tile data has been written — so your script can immediately overwrite or supplement whatever was just rendered.

### Three render event slots

The plugin maintains three independent script slots:

| Slot | Fires when |
|------|------------|
| **Render column** | A single column of tiles is loaded (normal scroll, one tile at a time). |
| **Render row** | A single row of tiles is loaded (normal scroll, one tile at a time). |
| **Render all** | The entire visible screen is repainted (teleport, fast scroll, scene load repaint, or a multi-tile jump). |

Each slot holds one script. Registering a new script replaces the previous one, and slots are independent — you can have different scripts for columns and rows, or leave some empty.

---

## Project Setup

1. Install the plugin into your GB Studio project's `plugins` folder. The events appear in the **Screen** group automatically.
2. In the scene's **On Init** script, add an **Attach a Script to a tile rendering Event** event.
3. Choose the **Render Event** slot — column, row, or all.
4. Write the script to execute in the **On Render Event** tab.

The registered script fires every time that render event occurs, for the lifetime of the scene or until it is explicitly removed.

### Typical pattern: persistent tile replacement

```
On Init:
  Attach a Script to a tile rendering Event → Render all
    → [Check flags, call Set background tile / Submap as needed]
  Attach a Script to a tile rendering Event → Render column
    → [Same check-and-replace logic]
  Attach a Script to a tile rendering Event → Render row
    → [Same check-and-replace logic]
```

"Render all" fires on the initial scene repaint, and "Render column" / "Render row" fire as the player scrolls, so the same replacement logic covers both the initial load and later scroll-in events.

Call **Remove a Script from a tile rendering Event** on a slot when it is no longer needed — for example once all collectibles are gone — to avoid unnecessary work.

---

## Size Limits and Restrictions

### Render scripts are not re-entrant

If a render event fires again while the previous run of that slot's script is still going, the new invocation is silently skipped. Keep render event scripts short and fast; they should comfortably finish within a single frame.

### No positional context is passed

Render scripts receive no arguments telling them *which* row or column was just rendered. If your script needs the current scroll position it must read the scroll engine fields itself.

### Render all fires once per full repaint

The "Render all" slot fires once at the end of a full-screen repaint, not once per row within it. For per-row logic during a full repaint, either handle it in the column and row slots or have the "Render all" script iterate over the whole screen itself.

### One script per slot

Each slot holds exactly one script. Attaching to an already-occupied slot silently replaces the previous script.

### Render scripts run asynchronously

The script is launched as a new concurrent thread. It does not block the scroll engine — tile data is already in VRAM before the script starts, on the next VM tick.

---

## Events Reference

Both events are in the **Screen** group and are allowed before the init fade.

---

### Attach a Script to a tile rendering Event

**`MICO_EVENT_RENDER_SCRIPT`**

Registers a sub-script to run every time the selected render event fires.

| Field | Description |
|-------|-------------|
| Select Render Event | Which slot to attach the script to: **Render column**, **Render row**, or **Render all**. |
| On Render Event (script) | The script to execute when the event fires. |

---

### Remove a Script from a tile rendering Event

**`MICO_EVENT_RENDER_SCRIPT_CLEAR`**

Unregisters the script from the selected slot. After this call the slot is empty and the event no longer triggers any script.

| Field | Description |
|-------|-------------|
| Select Render Event | The slot to clear: **Render column**, **Render row**, or **Render all**. |

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine by `measure_plugin_memory.js` (per-file SDCC compile with GB Studio's own build flags, at default engine settings; report of 2026-08-13). Figures are this plugin's *delta* versus stock — a file that replaces a stock engine file counts only the difference, which is why a plugin can come out negative. Using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | 0 bytes |
| WRAM | +15 bytes |
| Banked ROM | +302 bytes |

- **Bank 0:** nothing. Every function the plugin adds is compiled into a switchable ROM bank.
- **WRAM:** 15 bytes of tile-render callback state.
- **Banked ROM:** 302 bytes, net of the stock `scroll.c` the plugin replaces.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922). With this plugin installed roughly **839 bytes** remain. That does not change with the number of global variables your project defines: the script memory array is a fixed 3,584 bytes at stock engine settings (VM_HEAP_SIZE + VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE = 768 + 16 × 64 words).
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Every one of its functions is compiled
into a switchable ROM bank; nothing it adds is resident in bank 0.
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-06-28

- Added ContinuousScenePlugin compatibility.

### 2025-02-24

- Initial release (1.0.0).
