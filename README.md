# gbs-AttachScriptToInputExPlugin

**Requires GB Studio ≥ 4.3.0**

A GB Studio events-only plugin that extends the built-in **Attach Script to Button** event with three subscript phases: **On Press**, **On Hold**, and **On Release**. The standard event fires once per button-down edge. This plugin adds a per-frame hold loop and a release callback, enabling complete button-lifecycle scripting without manual polling loops.

![image](https://github.com/user-attachments/assets/53ae20c3-201c-4e86-ad7f-5e81521f555b)

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

### Standard Input Attachment vs Extended

The built-in GB Studio **Attach Script to Button** event registers a concurrent script that is launched each time the specified button transitions from not-held to held (a press edge). The script it runs has no built-in awareness of whether the button is still held when it runs, or when the button is eventually released.

The **Attach Script To Button EX** event is designed to be placed **inside** the subscript of a standard **Attach Script to Button** event. When the outer event fires on the press edge, the EX event's body runs and implements a complete press → hold loop → release lifecycle:

1. **On Press** — runs once when the press is first confirmed.
2. **On Hold** — runs every frame for as long as the button remains held. The script waits one frame between each iteration.
3. **On Release** — runs once when the button is no longer held.

### Joypad Polling

The hold detection reads `_joypads + 1` (the raw current-frame joypad state, i.e. which buttons are physically held right now) and masks it with the configured button set. This is distinct from the edge-detected `joy` value that the input event system uses. As a result, the hold loop responds to the actual physical state every frame regardless of other input events firing.

---

## Project Setup

1. Copy the `AttachScriptToInputExPlugin` folder into your GB Studio project's `plugins/` directory.
2. No engine files are modified — this plugin adds only a new event definition.

---

## How to Use

1. Add a standard **Attach Script to Button** event to any script (actor On Update, scene On Init, etc.).
2. Inside that event's subscript, add an **Attach Script To Button EX** event.
3. Configure the **Button** field in the EX event to the same button (or set of buttons) you used in the outer event.
4. Fill in the three subscript tabs:
   - **On Press** — actions to run once when the press is confirmed.
   - **On Hold** — actions to run every frame while the button is held (e.g. continuous movement, charge accumulation).
   - **On Release** — actions to run once when the button is released (e.g. fire a charged shot, end a slide).

Any of the three tabs can be left empty if that phase requires no action.

> ⚠️ **The EX event must be placed inside the subscript of a standard Attach Script to Button event.** It does not register its own input handler — it IS the handler body. Placing it outside an input attachment will cause it to run immediately when the script reaches it, not on button press.

---

## Technicalities and Restrictions

### Must Be Nested Inside Attach Script to Button

This event does not register an input callback itself. The outer standard **Attach Script to Button** event provides the edge-triggered launch. The EX event implements the full lifecycle once launched. Without the outer wrapper, the EX event's body executes immediately as normal sequential script code.

### On Hold Runs at Most Once Per Frame

The hold loop calls `_idle()` after each On Hold iteration, suspending the script thread for one frame before the next check. This means the On Hold script cannot execute more than once per frame, and the minimum hold polling interval is exactly one frame (approximately 16.7 ms at 60 fps).

### On Press Also Waits One Frame Before the Hold Loop Begins

After the On Press script completes, `_idle()` is called before the hold loop starts checking the button state. This means the first On Hold execution happens at the earliest two frames after the initial press detection.

### Empty Button Selection Matches All Buttons

If no buttons are selected in the **Button** field, the bitmask defaults to `0xFF` (all buttons). This is an intentional safety measure — a mask of `0` would cause the hold check to never match and the script to skip all phases immediately.

### The Hold Check Uses Raw Current-Frame State

The hold detection reads the raw joystick register (`_joypads + 1`), not the edge-detected values. This means it reflects the physical state at the start of each frame, independent of how many scripts are polling input.

### Script Thread Blocks for the Duration of the Hold

While the button is held, the launched script thread is occupied in the hold loop. The outer **Attach Script to Button** event will not fire again for the same button until this thread terminates (the button is released and the On Release script completes). Plan accordingly for re-entrant input handling.

### No Engine Files Modified

This is a pure events plugin. No engine C files are patched or replaced. There are no `engineAlt` variants.

---

## Events Reference

### Attach Script To Button EX

**Event ID:** `EVENT_SET_INPUT_SCRIPT_EX`  
**Group:** Input

Implements a full press → hold loop → release input lifecycle. Must be placed inside the subscript of a standard **Attach Script to Button** event.

| Field | Type | Default | Description |
|---|---|---|---|
| Button | Input (buttons) | B | The button(s) to monitor. Should match the button set on the outer **Attach Script to Button** event. |
| On Press | Events (subscript) | — | Script to run once when the button press is confirmed. |
| On Hold | Events (subscript) | — | Script to run every frame while the button remains held. |
| On Release | Events (subscript) | — | Script to run once when the button is released. |

---

## Inner Workings

### Compiled Script Structure

The compile function generates the following logical flow (labels added for clarity):

```
read _joypads+1 → inputRef
ARG0 = inputRef AND buttonMask
if ARG0 != 0: jump pressLabel
jump endLabel

pressLabel:
  [On Press script]
  idle()                    ← wait one frame

loopLabel:
  read _joypads+1 → inputRef
  ARG0 = inputRef AND buttonMask
  if ARG0 != 0: jump holdLabel
  jump releaseLabel

holdLabel:
  [On Hold script]
  idle()                    ← wait one frame
  jump loopLabel

releaseLabel:
  [On Release script]

endLabel:
```

### Initial Press Confirmation

On entry, the current joypad state is read and ANDed with the button mask. If the result is zero (button already released by the time this script was scheduled), the entire event is skipped and the script jumps to `endLabel`. This guards against the rare case where a very short press fires the outer event but the button is released before this script begins executing.

### Hold Loop Mechanics

The hold loop reads `_joypads + 1` each iteration. `_joypads + 1` is the address of `joypads.joy0` in RAM — the raw byte reflecting which buttons are currently pressed on the hardware. Using this rather than the edge-detected `joy` variable ensures the hold check responds to the physical state rather than the input event system's processed state.

The `_idle()` call after On Hold compiles to a GBVM `VM_IDLE` instruction, which suspends the current script context until the next frame. This is the same mechanism used by all waitable events in GB Studio.

### Button Bitmask Encoding

```
Bit 0 (0x01): Right
Bit 1 (0x02): Left
Bit 2 (0x04): Up
Bit 3 (0x08): Down
Bit 4 (0x10): A
Bit 5 (0x20): B
Bit 6 (0x40): Select
Bit 7 (0x80): Start
```

Multiple buttons are OR'd together. If all bits are zero (no buttons selected), the mask defaults to `0xFF` to prevent the script from hanging.

### Relationship with the Outer Attach Script to Button Event

The standard **Attach Script to Button** event compiles to a VM instruction that registers a concurrent script callback. When the specified button transitions from not-held to held (a new-press edge), GB Studio's event system launches the registered script as a new concurrent thread. The EX event's compiled bytecode is the body of that thread — it receives no special arguments and simply runs as a normal script once triggered.

