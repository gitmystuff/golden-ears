# 15 — Header Controls and Auto Tour

> **Run after:** 01, 02, 03.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/theme.py`,
> `src/s2s/ui/app.py`, then this file.
>
> **Owner:** inputs/outputs lead

---

## Context

The header lets a visitor choose what the ear hears: **Live mic**, **Whistle**,
**Hum** or **White noise**. An **Auto tour** cycles through the sounds on its own
so the laptop can run unattended at the demo. When a visitor clicks anything, the
tour pauses so they can explore; clicking Auto tour resumes it.

## Goal

1. `src/s2s/engine/tour.py` — the Auto tour as a pure state machine (no Qt).
2. `src/s2s/ui/panels/controls.py` — the header: title, subtitle, Auto tour
   button and source buttons.

## Interface

- **Reads and writes:** `AppState` (`source`, `preset`, `hearing_loss`,
  `auto_tour`) through the store.
- The engine (prompt 02) already switches sources when `AppState` changes. This
  prompt only changes state; it never talks to the audio system directly.

## Requirements

### 1. Tour state machine (logic; no Qt)
- Steps defined in `Config` as a tuple, default:
  1. Live mic
  2. Whistle
  3. Hum
  4. White noise
  5. Live mic with hearing loss on
- Each step lasts `Config.auto_tour_step_s` (6 s), then wraps around.
- `Tour.tick(now_s) -> AppState | None` returns a new state only when the step
  changes. **The clock is injected** (a `now_s` argument or a clock object), so
  tests run instantly and deterministically. Explain why this beats calling
  `time.time()` inside the class.
- `pause()` and `resume()`: resume continues from the step currently shown.
- Note: the live-mic steps show whatever the room hears. If the team wants an
  unattended voice, propose a `VOICE` preset (a bundled recording) as a
  **Contract change request**.

### 2. Header (UI)
Match the mockup:
- left: title "Sound to Sensation" (Space Grotesk, bold), subtitle "A simulated
  human ear · predicting what listeners hear";
- right: an **Auto tour** button (dashed outline; accent when on), a thin divider,
  "TRY A SOUND" label, then pill buttons **Live mic · Whistle · Hum · White noise**;
- the pill matching the current source or preset is filled with the accent color,
  **including when the tour changed it**. The UI follows `AppState`, never its own
  click history (unidirectional data flow; explain it);
- clicking a sound button sets that source or preset, sets `auto_tour = False`,
  and keeps `hearing_loss` as it is;
- clicking Auto tour toggles `auto_tour`;
- buttons are real `QPushButton`s, at least 44 px tall, reachable with Tab and
  Space, with accessible names;
- presenter shortcuts: **1–4** choose sounds, **T** toggles the tour, **H** toggles
  hearing loss.

### 3. Wiring
A `QTimer` (e.g. 10 Hz) calls `tour.tick()` while the tour is on and pushes any
new state to the store.

## Performance budget

Negligible; the header refreshes only on state changes.

## Acceptance criteria

`pytest tests/engine/test_tour.py tests/ui/test_controls.py` passes, with tests that:
- with a fake clock, steps change at exactly 6 s intervals and wrap from step 5 to 1;
- `pause()` stops changes; `resume()` continues from the shown step;
- step 5 sets `hearing_loss = True`, and leaving it sets it back to `False`;
- clicking "Hum" sets `source = PRESET`, `preset = HUM`, `auto_tour = False`;
- when the tour moves to Whistle, the Whistle pill shows as selected without any click;
- keyboard shortcuts 1–4, T and H work;
- every button has an accessible name and is at least 44 px tall.

## Explanation focus

Explain:
- state machines, and why the tour is one;
- dependency injection of the clock for testability;
- unidirectional data flow (state → UI, events → state) and the bugs it prevents;
- keyboard access and touch-target size as accessibility requirements, not polish.
