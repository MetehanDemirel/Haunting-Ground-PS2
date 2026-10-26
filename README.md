# Haunting Ground - 60 FPS Patch (.pnach) | IN BETA

A `.pnach` patch for **Haunting Ground** (*Demento*) on PCSX2 that unlocks 60 FPS gameplay.
It dynamically changes to 30 FPS during cutscenes and menus for better compatibility.

---

## Game Info & Compatibility

* **Title:** Haunting Ground (USA)
* **Serial:** `SLUS-21075`
* **CRC:** `901AAC09`
* **Target Framerate:** 60 FPS

> *Currently only supports the NTSC-U release.*

---

## Installation

1. Download the [latest release](https://github.com/MetehanDemirel/Haunting-Ground-PS2/releases/latest) and extract it.
2. In PCSX2, open **Tools → Open Data Directory** and place `SLUS-21075_901AAC09.pnach` in the `cheats` folder.
3. Right-click *Haunting Ground* in the game list → **Properties → Cheats**, turn on **Enable Cheats** and tick `60-fps-test`.
4. Start the game. PCSX2 shows *"1 cheat patch is active"*.

To compare against the original game, untick `60-fps-test` and restart.

### Important Note

**Use in-game saves (memory card).** PCSX2 savestates contain the patch's code. A savestate made with one version of the
patch can crash when loaded with another.

---

## Why 60 FPS was hard before?

Haunting Ground runs its whole simulation once per rendered frame and assumes 30 of them per second. Unlocking the frame
cap alone makes **everything** run twice as fast: animations, movement, AI timers, physics, effects, fades, sounds that are
started from game logic, even the play-time clock. Simple "60 FPS codes" for this kind of engine either leave the game
sped up or slow everything back down.

This patch keeps the rendering at 60 FPS and corrects each per-frame system individually, so one second of gameplay at 60
FPS does the same amount of work as one second at 30 FPS on a PS2.

---

<div align="center">

  <img src="./assets/fiona.jpg" alt="Fiona Belli Sketch by Koyorin" width="380" />

  *Fiona Belli Sketch by [Koyorin](https://x.com/koyoriin)*

---------

# For Nerds

---

## How it works

### Frame cap and mode switch

The render thread's frame limiter (`0x001BEF40`) is rewritten to wait for `1 + mode` vsyncs, where the mode word lives in
a free memory page at `0x000F8000` (`0` = 60 FPS, `1` = original 30 FPS). A small per-frame controller hooked into the
render loop sets the mode before any game logic runs:

* **Gameplay:** 60 FPS.
* **Menus, item screens, map, files:** automatically 30 FPS (game mode byte `0x0047B350` and the menu manager's fade at
  `*0x0044E988 + 0xA8C62`).
* **Cutscenes:** automatically 30 FPS (the event-script interpreter sets game mode 2 while an event runs).

### Patch layout

All added code lives in RAM the game never uses (`0x000F0000`–`0x00100000`): call trampolines, assembly stubs, jump
stubs, effect gates, a data page and compiled C code. Every hook replaces a single instruction (a `jal`, a vtable
pointer or a tail call) and the original is verified against the game executable when the patch is built.

### Correction techniques

| Technique | Used for |
|---|---|
| **Halve the per-frame step** at 60 FPS | Animation advance and root motion, fixed-speed moves, path following, turn rates, blend timers, fade steps, gravity and jump steps |
| **30 Hz gate**: run the update only on every other frame | Effect classes, event and room scripts, event triggers, the stalker encounter manager, blinks, chase gauge, play clock, music fades |
| **Freeze counters on odd frames** | 246 `±1` AI and state timers (stalker think timers, wait timers, stun, Hewie and Fiona timers, scene sequence waits) |
| **Exact per-second constants** | Exponential smoothing such as `0.75·prev + 0.25·measured` becomes `0.866 / 0.134` at 60 FPS (same result per second) |
| **Roll once per 30 Hz tick** | Per-frame random chances (Hewie's reactions and bite hold, blink timing) |
| **First-half-of-frame events** | One-shot animation events (hit frames, sound and rumble cues) fire once per animation frame, not twice |

---

## Systems covered

### Characters

* Animation playback, root motion, base and overlay layers, and all blend timers for every character.
* Animation frame events (hits, footsteps, rumble), so one-shot events fire once.
* **Fiona:** walk release slowdown, blend weight ease, fear/fatigue meter, mood cycle (calm/anxious phases, panic roll
  window), fear, panic, exhaustion and Hewie-bond timers, head look-at smoothing, eye blinks.
* **Hewie:** path speed (walk, trot and run measured equal), turning, state timers, jump, drop, leap, thrown and slide
  physics (gravity, height and horizontal steps), height blend, turn-speed ramp, reaction rolls to stalkers, bite-hold
  roll, action drains.
* **Stalkers (all 38 stalker-family classes):** think timers, state wait timers, alert stages, turn rates, look-around
  sweep, knockback impulses, chase gauge feed, per-chapter timers and tints.
* Ground-height smoothing on stairs and slopes, and soft-body chains (hair, clothes, tails).

### World and effects

* All particle and effect classes (fire, embers, glints, sparkles, water, dust, leaves), texture flipbooks (fireplaces,
  torches), twinkle sprites and flying creatures.
* Room emitters and gimmicks, animated props, map drawer drift.
* Event scripts, room scripts, event triggers and scene sequence waits.
* Room gimmick commands (elevators, gates, pressure plates, rotating parts), which run inside the gated scripts.
* Stalker encounter manager (spawn timer and encounter duration) and chase gauge.

### Camera, screen and sound

* Camera rail catch-up speed, camera switch blend and delay, camera shake.
* Screen fades, the pause overlay fade, pulsing screen tints.
* Music fades. Script-started ambient sounds are checked to start at the original rate.
* Play-time clock (saves and results screen show real time).

---

## Reverse engineering method

The game was analysed statically in Ghidra and dynamically in PCSX2 (memory reads over PINE, savestates as repeatable
test spots).

1. **Per-frame code inventory.** Every C++ class of the game was found through its vtable (602 classes) and through the
   state-machine tables (623 member-function-pointer states). From each class's methods, every function they reach was
   scanned for per-frame patterns: `±1` counters, `float += constant`, `field × constant` damping and per-frame random
   rolls. Every flagged function was reviewed by hand and is either fixed, already covered by a 30 Hz gate, or confirmed
   not to depend on frame rate.
2. **30 vs 60 comparisons in the running game.** The same savestate is run in both modes and compared per second:
   memory rates and step sizes for every word, animation frames per second, movement per animation frame, turn steps,
   blend lengths, timers and meters, sound starts, door and hide-spot timelines.
3. **Coverage.** A room teleport (retargeting a door link in the static room graph) loads any room so that its objects
   can be checked, and stalkers that are not active at a save point are spawned through the game's own encounter
   routines.

### Results

* Every animation of Fiona, Hewie and the stalkers plays at 30.0 animation frames per second in both modes.
* Hewie path speeds at 30 / 60: walk 8.0 / 7.9, trot 24.2 / 24.2, run 50.5 / 51.6 units/s.
* Fire flipbook: 29.9 / 30.4 frame changes per second. Play clock: 5.0 s per 5.0 s real time in both modes.
* Door transition and hide spot timelines match within one 30 FPS frame.
* Chase gauge rises at the same rate (0 → 6 in 5 s at both rates).
* Input: the game reacts within one vsync in both modes. The patch adds no input delay.

---

## Known limitations

* NTSC-U only.
* Savestates are not compatible between patch versions. Memory-card saves always are.
* One chapter-specific stalker is only checked in code: it appears through the story and could not be spawned for a live
  test.
* Rare cases that are only checked in code: a turning branch of Hewie that may still turn at double speed, and two Fiona
  interaction slides that may reach their target early.
* The frame-rate scan finds the common per-frame patterns. Frame-rate-dependent logic of another form can still exist,
  which is what this beta is for.

---

## Reporting issues

Please open an issue with:

* What happened and what you expected (for example *"crows fly faster than without the patch"*).
* Where: chapter and room, or a memory-card save if you can share one.
* Whether it also happens with the patch disabled.
* Your PCSX2 version and any other active patches (widescreen and similar).
* A short video if the problem is about speed or timing.
