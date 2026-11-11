# 20 — Integration and Demo Readiness

> **Run after:** all of 01–16 are merged.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/app.py`,
> `src/s2s/engine/pipeline.py`, every `stage.py`, then this file.
>
> **Owner:** both

---

## Context

Each part was built and tested on its own. This prompt connects them into the
finished demo, checks that they work together in real time on the demo laptop,
and produces the documents needed on December 7.

## Goal

1. Register every stage with the engine in the right order and replace every
   placeholder panel with the real one.
2. End-to-end tests that run the whole pipeline on known stimuli.
3. A performance profile on the demo laptop.
4. A demo-day runbook.

## Requirements

### 1. Wiring (`src/s2s/app_factory.py`)
- One function, `build_app(config) -> (Engine, MainWindow)`, that creates the
  store, the stages (level, ear, nerve, direction, sensation), the panels and the
  tour, in one readable place. Explain why construction lives in one factory
  (the *composition root*) rather than spread across modules.
- Stage order: level and direction on the raw block; ear then nerve; the
  sensation stage only buffers. Justify the order.

### 2. End-to-end tests (`tests/e2e/`)
Run the full pipeline headless on generated fixtures and check, at the frame level:
- **Whistle** (2093 Hz, 70 dB): Ear peak in the channel nearest 2.1 kHz; Nerve rate
  highest there; Sensation pitch C7 ± 1 semitone; Direction 0° (identical channels).
- **Hum vs. whistle:** same `spl_db` (± 0.5 dB) but loudness ratio below 0.75.
- **White noise:** broad Ear activity (at least 80 % of channels above the midpoint
  of the color range); pitch `None`.
- **Missing fundamental:** Sensation pitch 200 Hz ± 2 % although no channel near
  200 Hz is strongly excited.
- **Hearing loss on:** high-frequency Ear channels drop; loudness and sharpness drop.
- **Moving talker:** a speech fixture panned from −60° to +60° over 6 s (a delay
  sweep between channels) gives a monotonic azimuth trend with error under 10°
  while active.

### 3. Performance profile
- `s2s profile --seconds 60` runs the live app and writes a report
  (`docs/performance.md`): mean and 95th-percentile time per stage and per panel
  refresh, input overflows, and CPU use.
- Every budget from the panel prompts must hold on the demo laptop. If one fails,
  propose the smallest fix (fewer channels, lower display rate, shorter window) and
  state its scientific cost.

### 4. Demo-day runbook (`docs/demo-runbook.md`)
A checklist a non-programmer can follow:
- hardware: laptop on power, stereo mic pair 15–20 cm apart, speaker or
  headphones, cable checked with `s2s mic-check`;
- calibration with `s2s calibrate`, and where the value goes;
- start command, full screen, Auto tour on;
- a two-minute talk track that walks a visitor across the screen
  (Sound → Ear → Nerve → Sensation → Direction), using the notices;
- troubleshooting: no mic, eyes mirrored, choppy display, everything too quiet.

## Acceptance criteria

- `pytest` passes in full, including `tests/e2e`.
- `s2s run` starts in full screen with every panel live and the tour running.
- `docs/performance.md` shows every budget met on the demo laptop, or documents
  the agreed trade-off.
- `docs/demo-runbook.md` is complete enough that someone who has never seen the
  code can set up and run the demo.

## Explanation focus

Explain:
- the composition root and why wiring is kept separate from logic;
- the difference between unit, integration and end-to-end tests, with examples
  from this project;
- how to read a performance profile (mean versus 95th percentile, and why the tail
  matters for real-time audio);
- what the end-to-end results show scientifically, in a form that can go straight
  onto a poster.
