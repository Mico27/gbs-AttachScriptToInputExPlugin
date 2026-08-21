# gbs-AttachScriptToInputExPlugin

**Version 1.2.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio events-only plugin that extends the built-in **Attach Script to Button** event with three subscript phases: **On Press**, **On Hold**, and **On Release**, and adds button combinations, double tap detection and input sequence (combo) detection. The standard event fires once per button-down edge. This plugin adds a per-frame hold loop, a release callback, and the input pattern matching that would otherwise need hand written polling loops.

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

Both events read the **raw current-frame joypad state** — which buttons are physically held right now — rather than the edge-detected value the input event system uses. They sample it once per frame and derive their own press edges from it, so they respond to the actual physical state regardless of what other input events are firing.

### Button combinations

With **Button Combination** enabled the selected buttons stop meaning "any of these" and start meaning "all of these, and nothing else":

- **On Press** only runs on the frame where every selected button is held at the same time. Buttons may be pressed one after the other, the event keeps waiting for the combination to be completed.
- Pressing a button that is **not** part of the combination cancels the event before On Press ever runs.
- Letting go of the whole combination before it is completed also cancels the event.
- **On Hold** runs for as long as every button of the combination stays held. Buttons outside the combination are ignored at this point.
- **On Release** runs as soon as **one** of the buttons of the combination is released.

### Double tap

With **Double Tap** enabled, the first press only arms the event. The button (or the whole combination) has to be released and pressed a second time within the **Double Tap Window** before **On Press** runs. If the second press does not arrive in time the event ends silently and no phase script runs at all. On Hold and On Release then follow the second press, so a double tap can be held down just like a single press.

### Input sequences (combos)

The **Input Sequence EX** event waits for a list of button presses to be entered in order — a fighting game motion, a cheat code, a lock combination. Every step is a button (or a combination), matched on the frame it is pressed:

- The **first** step waits forever, so the event can sit and listen for the sequence to start.
- Every later step has to be entered within the **Step Timeout**, which is read again at the start of each step. Using a variable there means the window can be widened or tightened while the sequence is running, for example from the **On Step** script.
- Pressing a button that does not belong to the current step fails the sequence immediately, as does letting a step time out.
- **On Success** runs once the last step has been matched, **On Failure** runs on a wrong button or a timeout, and **On Step** runs after every matched input.
- **Restart On Failure** turns the event into a permanent listener: after a failure it goes back to listening for the first step instead of ending. The input that caused the failure is tested against the first step first, so entering `UP UP DOWN` when the sequence is `UP UP` restarts correctly on the third press.

Buttons already held when the event starts count as freshly pressed, which is what makes the event work when it is placed inside an **Attach Script to Button** for the first button of the sequence.

---

## Project Setup

1. Copy the `AttachScriptToInputExPlugin` folder into your GB Studio project's `plugins/` directory. No engine files are modified.
2. Add a standard **Attach Script to Button** event to any script (actor On Update, scene On Init, etc.).
3. Inside that event's subscript, add an **Attach Script To Button EX** or an **Input Sequence EX** event.
4. Set the **Button** field to the same button (or set of buttons) you used in the outer event. For a combination or a sequence, the outer event should list every button that is allowed to start it.
5. Fill in the subscript tabs. Any tab can be left empty if that phase requires no action.

> ⚠️ **These events must be placed inside the subscript of a standard Attach Script to Button event.** They do not register their own input handler — they *are* the handler body. Placed outside an input attachment they simply run immediately when the script reaches them, not on button press.

The `AttachScriptToInputExExample` project in this repository demonstrates every feature: scenes 1 and 2 use the press/hold/release lifecycle, scene 3 uses an `A+B` combination and a double tap on `DOWN`, and scene 4 uses a five step input sequence with a timeout variable that tightens after the first input.

### About the example project's text

The example ships a second plugin, [UI Alt Display Text](https://github.com/Mico27/gbs-uiAltDisplayTextPlugin), and draws all of its on-screen text with **Alt Load and Display Text To Background Instantly** instead of the stock **Draw Text**. The stock renderer writes glyph pixels into the VWF tile buffer every time a line is drawn, which is wasteful for feedback that is redrawn on every button press; the Alt renderer only writes tile IDs to the tilemap, so a text update costs a handful of VRAM writes.

Each scene's init script starts with an **Alt Load Font tiles** event that copies GBS Mono into VRAM tiles 8–103 (ASCII 32–127 are unique tiles 0–95 of the compiled font, the background itself only uses tiles 0–3, the UI frame sits at `0xC0` and the stock VWF buffer at `0xCC`). Only the event in Scene 1 has **Adjust font mapping with offset on compile** checked — that option rewrites the font's character-to-tile table globally at build time and may only be applied once per font per build, while the runtime tile copy still has to happen in every scene.

Neither of the plugin's own events depends on this. Replace the Alt text events with stock **Draw Text** events and remove the `UIAltDisplayTextPlugin` folder if you only want the input handling.

---

## Size Limits and Restrictions

### Must be nested inside Attach Script to Button

These events do not register an input callback themselves. The outer standard event provides the edge-triggered launch; the EX events implement the lifecycle once launched. Without the outer wrapper, their bodies execute immediately as ordinary sequential script code.

### On Hold runs at most once per frame

The hold loop waits one frame between iterations, so the On Hold script cannot execute more than once per frame — a minimum polling interval of roughly 16.7 ms at 60 fps. The same applies to every wait loop in these events: input is sampled once per frame.

### The first hold iteration is two frames after the press

One frame is also skipped after the On Press script completes, so the earliest On Hold execution happens two frames after the initial press detection.

### An empty button selection matches all buttons

If no buttons are selected in a **Button** field, all buttons are matched. This is deliberate: matching nothing would make the event skip all its phases immediately. Because "any button" cannot be combined, **Button Combination** is ignored when no button is selected.

### The script thread blocks for the duration of the event

While the button is held, the launched script thread is occupied by the hold loop. The outer **Attach Script to Button** event will not fire again for the same button until that thread terminates — that is, until the button is released and the On Release script completes. The same is true while a combination is being formed, while a double tap window is open, and for the whole duration of an input sequence. Plan accordingly for re-entrant input handling.

With **Restart On Failure** enabled an input sequence only ends on success, so its thread stays resident until the sequence is completed.

### Very short presses are skipped

If the button is already released by the time the launched script starts running, the whole event is skipped, including On Press and On Release.

### A partly held combination waits indefinitely

While waiting for a combination to be completed, the event gives up when every button of the combination has been released or when a button outside the combination is pressed — but not on a timer. Holding one button of the combination forever keeps the event waiting forever. Use **Double Tap** or an input sequence if you need a time limit on the entry.

### The first step of a sequence has no timeout

The **Step Timeout** applies from the second step onwards. The first step waits for as long as it takes, which is what allows the event to be used as a permanent combo listener. A timeout of `0` disables the timeout for the later steps as well.

### Sequences are limited to 12 steps

**Sequence Length** accepts 1 to 12 inputs. The **On Step** script is compiled once and shared by every step, so its size does not grow with the sequence length.

---

## Events Reference

---

### Attach Script To Button EX

**`EVENT_SET_INPUT_SCRIPT_EX`** — group: **Input**

Implements a full press → hold loop → release input lifecycle, optionally gated behind a button combination and/or a double tap. Must be placed inside the subscript of a standard **Attach Script to Button** event.

| Field | Default | Description |
|---|---|---|
| Button | B | The button(s) to monitor. Should match the button set on the outer **Attach Script to Button** event. |
| Button Combination | off | When enabled every selected button must be held at the same time, and no other button may be held, before On Press runs. |
| Double Tap | off | When enabled the button (or combination) must be released and pressed a second time before On Press runs. |
| Double Tap Window | 15 | Frames allowed between the first press and the second press. Accepts a variable. Only shown when **Double Tap** is enabled. |
| On Press | — | Script to run once when the press is confirmed. |
| On Hold | — | Script to run every frame while the button remains held. |
| On Release | — | Script to run once when the button is released. |

---

### Input Sequence EX

**`EVENT_INPUT_SEQUENCE_EX`** — group: **Input**

Waits for a list of button presses to be entered in order and branches on success or failure. Must be placed inside the subscript of a standard **Attach Script to Button** event, or used in a script that is free to block.

| Field | Default | Description |
|---|---|---|
| Sequence Length | 3 | Number of inputs that make up the sequence, 1 to 12. |
| Step Timeout | 30 | Frames allowed to enter each input after the first one. `0` means no timeout. Accepts a variable, and is read again at the start of every step so the value can change while the sequence is running. |
| Restart On Failure | off | When enabled the sequence starts listening again from the first input after a failure instead of ending. The failing input is still tested against the first step. |
| Step *n* | A | Button that has to be pressed for step *n*. Leaving it empty accepts any button. |
| Step *n* Combination | off | When enabled every button selected for step *n* must be held at the same time, and the step only matches on the frame the combination is completed. |
| On Success | — | Script to run once the whole sequence has been entered. |
| On Failure | — | Script to run when a wrong button is pressed or a step times out. |
| On Step | — | Script to run after every matched input, before the timeout of the next step is read. |

---

## Memory Footprint

This plugin ships no engine sources, so it has no fixed memory cost at all — confirmed by `measure_plugin_memory.js` against the stock GB Studio **4.3.0-e1** engine (report of 2026-08-13).

- **Bank 0, WRAM, SRAM:** nothing. The plugin ships no engine sources at all.
- **ROM:** the events compile plain GBVM script into your project's script banks — a few bytes per call. There is no fixed engine cost.

Both events use script locals only, no globals: **Attach Script To Button EX** reserves 1 local (2 with **Double Tap**), and **Input Sequence EX** reserves 3 locals, plus 1 for the step timeout and 1 more when an **On Step** script is used.

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

### 2026-08-20

- Added a **Button Combination** option to **Attach Script To Button EX**: On Press only runs once every selected button is held at the same time and no other button is held, On Hold runs while the combination stays formed and On Release runs as soon as one of its buttons is released.
- Added a **Double Tap** option with an editable window to **Attach Script To Button EX**, usable on its own or together with a combination.
- Added the **Input Sequence EX** event: up to 12 ordered inputs with per-step combinations, a step timeout that can be driven by a variable and changed while the sequence runs, On Success / On Failure / On Step scripts and an optional restart on failure.

### 2025-04-23

- Fixed a missing event name.

### 2025-02-24

- Initial release.
