# gbs-AttachScriptToInputExPlugin

**Version 1.1.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio events-only plugin that extends the built-in **Attach Script to Button** event with three subscript phases: **On Press**, **On Hold**, and **On Release**. The standard event fires once per button-down edge. This plugin adds a per-frame hold loop and a release callback, enabling complete button-lifecycle scripting without manual polling loops.

![image](https://github.com/user-attachments/assets/53ae20c3-201c-4e86-ad7f-5e81521f555b)

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

### Standard input attachment vs extended

The built-in GB Studio **Attach Script to Button** event runs a script each time the specified button transitions from not-held to held (a press edge). The script it runs has no built-in awareness of whether the button is still held while it runs, or when the button is eventually released.

The **Attach Script To Button EX** event is designed to be placed **inside** the subscript of a standard **Attach Script to Button** event. When the outer event fires on the press edge, the EX event's body implements a complete lifecycle:

1. **On Press** — runs once when the press is first confirmed.
2. **On Hold** — runs every frame for as long as the button remains held, waiting one frame between iterations.
3. **On Release** — runs once when the button is no longer held.

### Physical vs edge-detected input

Hold detection reads the **raw current-frame joypad state** — which buttons are physically held right now — rather than the edge-detected value the input event system uses. The hold loop therefore responds to the actual physical state every frame, regardless of what other input events are firing.

---

## Project Setup

1. Copy the `AttachScriptToInputExPlugin` folder into your GB Studio project's `plugins/` directory. No engine files are modified.
2. Add a standard **Attach Script to Button** event to any script (actor On Update, scene On Init, etc.).
3. Inside that event's subscript, add an **Attach Script To Button EX** event.
4. Set the EX event's **Button** field to the same button (or set of buttons) you used in the outer event.
5. Fill in the three subscript tabs:
   - **On Press** — actions to run once when the press is confirmed.
   - **On Hold** — actions to run every frame while the button is held, e.g. continuous movement or charge accumulation.
   - **On Release** — actions to run once when the button is released, e.g. firing a charged shot or ending a slide.

Any of the three tabs can be left empty if that phase requires no action.

> ⚠️ **The EX event must be placed inside the subscript of a standard Attach Script to Button event.** It does not register its own input handler — it *is* the handler body. Placed outside an input attachment it simply runs immediately when the script reaches it, not on button press.

---

## Size Limits and Restrictions

### Must be nested inside Attach Script to Button

This event does not register an input callback itself. The outer standard event provides the edge-triggered launch; the EX event implements the lifecycle once launched. Without the outer wrapper, its body executes immediately as ordinary sequential script code.

### On Hold runs at most once per frame

The hold loop waits one frame between iterations, so the On Hold script cannot execute more than once per frame — a minimum polling interval of roughly 16.7 ms at 60 fps.

### The first hold iteration is two frames after the press

One frame is also skipped after the On Press script completes, so the earliest On Hold execution happens two frames after the initial press detection.

### An empty button selection matches all buttons

If no buttons are selected in the **Button** field, all buttons are matched. This is deliberate: matching nothing would make the event skip all three phases immediately.

### The script thread blocks for the duration of the hold

While the button is held, the launched script thread is occupied by the hold loop. The outer **Attach Script to Button** event will not fire again for the same button until that thread terminates — that is, until the button is released and the On Release script completes. Plan accordingly for re-entrant input handling.

### Very short presses are skipped

If the button is already released by the time the launched script starts running, the whole event is skipped, including On Press and On Release.

---

## Events Reference

---

### Attach Script To Button EX

**`EVENT_SET_INPUT_SCRIPT_EX`** — group: **Input**

Implements a full press → hold loop → release input lifecycle. Must be placed inside the subscript of a standard **Attach Script to Button** event.

| Field | Default | Description |
|---|---|---|
| Button | B | The button(s) to monitor. Should match the button set on the outer **Attach Script to Button** event. |
| On Press | — | Script to run once when the button press is confirmed. |
| On Hold | — | Script to run every frame while the button remains held. |
| On Release | — | Script to run once when the button is released. |

---

## Memory Footprint

This plugin ships no engine sources, so it has no fixed memory cost at all — confirmed by `measure_plugin_memory.js` against the stock GB Studio **4.3.0-e1** engine (report of 2026-08-13).

- **Bank 0, WRAM, SRAM:** nothing. The plugin ships no engine sources at all.
- **ROM:** the events compile plain GBVM script into your project's script banks — a few bytes per call. There is no fixed engine cost.

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

### 2025-04-23

- Fixed a missing event name.

### 2025-02-24

- Initial release.
