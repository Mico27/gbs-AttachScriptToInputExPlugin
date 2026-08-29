# gbs-AttachScriptToInputExPlugin

**Version 1.2.0. Requires GB Studio 4.3.0 or newer.**

Turns a button press into three scripts: **On Press**, **On Hold** which repeats every frame while
the button is down, and **On Release**. It also detects button combinations, double taps and
ordered input sequences.

That covers a charged shot that builds while B is held, a run button, a crouch that ends when the
player lets go, an A plus B special move, a double tap to dash, and a cheat code entered on the
title screen. GB Studio's own **Attach Script to Button** fires once on the press and tells you
nothing after that.

![image](https://github.com/user-attachments/assets/53ae20c3-201c-4e86-ad7f-5e81521f555b)

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [FAQ](#faq)
6. [Memory Footprint](#memory-footprint)
7. [Bank 0 (HOME) Usage](#bank-0-home-usage)
8. [Changelog](#changelog)

---

## Concepts

### The three phases

**Attach Script To Button EX** goes **inside** the script of a standard **Attach Script to Button**
event. The outer event launches it on the press, and the EX event then runs the full life of that
press:

1. **On Press** runs once, when the press is confirmed.
2. **On Hold** runs every frame for as long as the button stays down.
3. **On Release** runs once, when the button comes up.

### Reading the buttons

Both events read which buttons are physically down this frame, and work out their own press moments
from that. They are unaffected by what other input events are doing.

### Button combinations

With **Button Combination** on, the selected buttons mean "all of these together, and nothing
else":

- **On Press** runs on the frame every selected button is down at once. They may be pressed one
  after another, and the event keeps waiting.
- Pressing a button outside the combination cancels the event before On Press runs.
- Letting go of everything before the combination is complete also cancels it.
- **On Hold** runs while every button of the combination stays down. Other buttons are ignored by
  then.
- **On Release** runs as soon as one of the combination's buttons comes up.

### Double tap

With **Double Tap** on, the first press only arms the event. The button, or the whole combination,
has to be released and pressed again inside the **Double Tap Window** before **On Press** runs. A
second press that never arrives ends the event quietly with no script run at all. On Hold and On
Release then follow the second press, so a double tap can be held down like a normal press.

### Input sequences

**Input Sequence EX** waits for a list of presses in order: a fighting game motion, a cheat code, a
combination lock. Each step is a button, or a combination, matched on the frame it goes down.

- The **first** step waits forever, so the event can sit and listen for the sequence to begin.
- Every later step has to arrive within the **Step Timeout**, which is read again at the start of
  each step. Putting a variable there lets the window widen or tighten while the sequence runs, for
  instance from the **On Step** script.
- A button that does not belong to the current step fails the sequence at once, and so does a step
  running out of time.
- **On Success** runs after the last step matches, **On Failure** runs on a wrong button or a
  timeout, and **On Step** runs after every match.
- **Restart On Failure** turns the event into a permanent listener. After a failure it goes back to
  waiting for the first step. The input that caused the failure is tested against the first step
  first, so entering `UP UP DOWN` when the sequence is `UP UP` restarts correctly on the third
  press.

Buttons already down when the event starts count as freshly pressed, which is what makes it work
inside an **Attach Script to Button** for the sequence's first button.

---

## Project Setup

1. Copy the `AttachScriptToInputExPlugin` folder into your project's `plugins` folder. No engine
   files are changed.
2. Add a standard **Attach Script to Button** event to any script, such as an actor's On Update or
   a scene's On Init.
3. Inside that event's script, add **Attach Script To Button EX** or **Input Sequence EX**.
4. Set the **Button** field to the same button, or set of buttons, as the outer event. For a
   combination or a sequence, the outer event should list every button that can start it.
5. Fill in the tabs you need. Leave a tab empty when that phase does nothing.

> **These events must sit inside a standard Attach Script to Button event.** They do not register
> a handler of their own. They are the handler's body. Placed anywhere else they simply run when
> the script reaches them.

The `AttachScriptToInputExExample` project shows every feature. Scenes 1 and 2 use press, hold and
release, scene 3 uses an A plus B combination and a double tap on DOWN, and scene 4 uses a five
step sequence with a timeout variable that tightens after the first input.

### About the example project's text

The example also ships [UI Alt Display Text](https://github.com/Mico27/gbs-uiAltDisplayTextPlugin)
and draws its on-screen text with **Alt Load and Display Text To Background Instantly**. The stock
renderer rebuilds each letter's pixels every time a line is drawn, which is wasteful for feedback
redrawn on every button press. The Alt renderer writes tile numbers only, so an update costs a
handful of writes.

Each scene's init script starts with **Alt Load Font tiles** copying GBS Mono into tiles 8 to 103.
Only Scene 1's event has **Adjust font mapping with offset on compile** ticked, because that option
rewrites the font's character table for the whole build and may be applied once per font. The tile
copy itself still has to happen in every scene.

Neither of this plugin's events depends on that. Swap the Alt text events for stock **Draw Text**
events and delete the `UIAltDisplayTextPlugin` folder if you only want the input handling.

---

## Size Limits and Restrictions

### They must be nested inside Attach Script to Button

The outer event provides the press that launches them. Without it, their contents run as ordinary
script code the moment the script reaches them.

### On Hold runs once per frame at most

The hold loop waits a frame between passes, so On Hold runs at most 60 times a second. Every wait
loop in these events samples the buttons once per frame.

### The first hold pass is two frames after the press

A frame is also skipped after the On Press script finishes, so the earliest On Hold is two frames
after the press was detected.

### Selecting no buttons matches every button

An empty **Button** field matches all buttons, because matching nothing would make the event skip
straight to the end. **Button Combination** is ignored in that case.

### The script is busy for the whole event

While the button is held, the launched script is occupied by the hold loop, and the outer **Attach
Script to Button** will not fire again for that button until it finishes, meaning until the button
is released and On Release completes. The same holds while a combination is being formed, while a
double tap window is open, and for the whole of an input sequence.

With **Restart On Failure** on, a sequence ends only on success, so its script stays resident until
then.

### Very short presses are skipped

If the button is already up by the time the launched script starts, the whole event is skipped,
On Press and On Release included.

### A half-held combination waits forever

While waiting for a combination, the event gives up when every one of its buttons is released, or
when a button outside it is pressed. There is no timer. Holding one button of the combination keeps
it waiting indefinitely. Use **Double Tap** or a sequence when you want a time limit.

### The first step of a sequence has no timeout

**Step Timeout** applies from the second step on. The first waits as long as it takes, which is
what makes a permanent combo listener possible. A timeout of `0` turns it off for the later steps
too.

### A sequence holds up to 12 steps

**Sequence Length** accepts 1 to 12. The **On Step** script is compiled once and shared by every
step, so a longer sequence does not make it bigger.

---

## Events Reference

### Attach Script To Button EX

Group: **Input**.

Runs a press, hold and release cycle, optionally behind a combination, a double tap, or both. Place
it inside the script of a standard **Attach Script to Button** event.

| Field | Default | Description |
|---|---|---|
| Button | B | The buttons to watch. Match the outer event's buttons. |
| Button Combination | off | Every selected button must be down at once, with nothing else down, before On Press runs. |
| Double Tap | off | The button, or combination, must be released and pressed again before On Press runs. |
| Double Tap Window | 15 | Frames allowed between the two presses. Accepts a variable. Shown only with **Double Tap** on. |
| On Press | none | Runs once when the press is confirmed. |
| On Hold | none | Runs every frame while the button stays down. |
| On Release | none | Runs once when the button comes up. |

### Input Sequence EX

Group: **Input**.

Waits for a list of presses in order and branches on success or failure. Place it inside the script
of a standard **Attach Script to Button** event, or anywhere a script is free to wait.

| Field | Default | Description |
|---|---|---|
| Sequence Length | 3 | How many inputs make up the sequence, 1 to 12. |
| Step Timeout | 30 | Frames allowed for each input after the first. `0` means no limit. Accepts a variable, read again at every step, so it can change while the sequence runs. |
| Restart On Failure | off | After a failure, go back to waiting for the first input instead of ending. The failing input is still tested against the first step. |
| Step *n* | A | The button for step *n*. Left empty, any button matches. |
| Step *n* Combination | off | Every button selected for step *n* must be down at once, and the step matches on the frame that happens. |
| On Success | none | Runs once the whole sequence is entered. |
| On Failure | none | Runs on a wrong button or a timeout. |
| On Step | none | Runs after every matched input, before the next step's timeout is read. |

---

## FAQ

**How do I make a charged shot that builds while B is held?**
Put **Attach Script To Button EX** inside an **Attach Script to Button** for B. Reset a charge
variable in **On Press**, add to it in **On Hold**, and fire based on its value in **On Release**.

**How do I detect a special move on A plus B?**
Set the outer event to A and B, then in the EX event select both buttons and tick **Button
Combination**. On Press runs on the frame both are down together.

**How do I add a double tap to dash?**
Tick **Double Tap** and set the button to a direction. Set **Double Tap Window** to about 15 frames
to start with, and adjust to taste.

**How do I add a cheat code?**
Use **Input Sequence EX** with a step per button, and put the reward in **On Success**. Tick
**Restart On Failure** so it keeps listening on the title screen.

**Nothing happens when I add these events.**
They have to sit inside a standard **Attach Script to Button** event. On their own they run
immediately, as normal script steps.

**My button script does not fire again while the button is held.**
That is by design. The script stays busy for the whole press. It becomes available again once the
button is released and **On Release** has finished.

**How often does On Hold run?**
Once per frame at most, so 60 times a second. The first pass is two frames after the press.

**How do I put a time limit on a combination?**
Combinations have no timer of their own. Use **Double Tap**, or an input sequence with a **Step
Timeout**.

**Can the timeout change partway through a sequence?**
Yes. Put a variable in **Step Timeout** and change it from **On Step**. The value is read again at
the start of every step, so the first input can be generous and later ones tight.

**Can I use a directional input in a sequence, like a quarter circle?**
Yes. Give each step the direction it needs, and use per-step combinations for diagonals.

**Does it cost ROM?**
Only for the events you use, a few bytes each. There is no fixed cost, and no engine files are
changed.

---

## Memory Footprint

This plugin ships no engine code, so it has no fixed cost. Measured against the stock GB Studio
**4.3.0-e1** engine, report of 2026-08-13.

- **Bank 0, WRAM, SRAM:** nothing.
- **ROM:** the events compile a few bytes of script per call into your project. There is no fixed
  cost.

Both events use script-local storage and no global variables. **Attach Script To Button EX** takes
1 local, or 2 with **Double Tap**. **Input Sequence EX** takes 3, plus 1 for the step timeout and 1
more when an **On Step** script is used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Everything it adds is compiled into a
switchable ROM bank.
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version bumps, patch
regeneration, packaging fixes and documentation edits are omitted.

### 2026-08-20

- Added **Button Combination** to **Attach Script To Button EX**. On Press runs once every selected
  button is down at the same time with nothing else down, On Hold runs while the combination holds,
  and On Release runs as soon as one of its buttons comes up.
- Added **Double Tap**, with an editable window, usable on its own or with a combination.
- Added **Input Sequence EX**: up to 12 ordered inputs with per-step combinations, a step timeout
  that can come from a variable and change while the sequence runs, On Success, On Failure and
  On Step scripts, and an optional restart on failure.

### 2025-04-23

- Fixed a missing event name.

### 2025-02-24

- Initial release.
