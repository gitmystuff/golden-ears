# 13 — Panel 4: Sensation (loudness, pitch, sharpness, roughness)

> **Run after:** 01, 02, 03. Uses `hearing_loss_gain_db()` from prompt 11; if 11
> is not merged yet, stub it and note this in the PR.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/panels/base.py`,
> `src/s2s/ui/theme.py`, then this file.
>
> **Owner:** inputs/outputs lead

---

## Context

This is where the screen says what a listener would **report**. Four sensations
are predicted with standard psychoacoustic models, which are fitted to listening
tests. They are predictions, not measurements, and the panel says so.

The demo's central lesson lands here: **physics and sensation don't match.** A
147 Hz hum and a 2 kHz whistle at the same 70 dB SPL get very different loudness
values, because the ear is less sensitive to low pitches.

## Goal

1. `src/s2s/sensation/loudness.py`, `sharpness.py`, `roughness.py`, `pitch.py`.
2. `src/s2s/sensation/stage.py` — a `Stage` that buffers audio and a worker
   thread that computes a `SensationFrame` every `Config.sensation_update_s`.
3. `src/s2s/ui/panels/sensation.py` — the gauges.

## Interface

- **Reads:** `AudioBlock` (via the stage), `AppState.hearing_loss`, `Config`.
- **Produces:** `SensationFrame` every 0.25 s, computed on the most recent
  `Config.sensation_window_s` (1 s) of mono audio.

## Requirements

### 1. Threading
The metrics are too slow to compute inside the audio worker's block budget. The
stage only appends each block's mono signal to a 1 s ring buffer (cheap). A
separate worker thread wakes every 0.25 s, copies the window, computes all four
metrics, and publishes a `SensationFrame`. If a computation overruns, skip a
cycle rather than queueing (explain why dropping is better than lagging for a
live display).

### 2. Loudness (sones)
- ISO 532-1 (Zwicker) via MoSQITo. Check the installed MoSQITo version's API
  (function names have changed between releases), pin the version, and wrap it
  in your own function `loudness_sone(signal_pa, fs) -> float` so the rest of
  the code does not depend on MoSQITo's names.
- Choose the stationary or time-varying method for a 1 s window and justify it
  (accuracy, speed, what ISO 532-1 intends each for).

### 3. Sharpness (acum)
- DIN 45692 via MoSQITo, wrapped as `sharpness_acum(signal_pa, fs)`.

### 4. Roughness (asper)
- Daniel & Weber (1997) via MoSQITo, wrapped as `roughness_asper(signal_pa, fs)`.

### 5. Pitch (Hz, note name)
- Implement the **YIN** algorithm (de Cheveigné & Kawahara, 2002) yourself, in
  NumPy, on the 1 s window (or a shorter recent slice; justify). Return
  `pitch_hz` and `pitch_strength` (0–1, from YIN's aperiodicity measure).
- Return `pitch_hz = None` when `pitch_strength` is below a configurable threshold
  ("No clear pitch").
- Explain how YIN relates to the **summary autocorrelation** model of the
  auditory system (Meddis & O'Mard, 1997), which pools autocorrelation across
  cochlear channels, and why YIN was chosen for real time. Describe the summary
  autocorrelation model as a possible extension.
- `note_name(freq_hz) -> str` for 12-tone equal temperament with A4 = 440 Hz,
  using ♯ names (e.g. "A♯3").

### 6. Hearing-loss mode
These standard models assume normal hearing. When `AppState.hearing_loss` is on,
filter the analysis audio with the same frequency-dependent attenuation as the
Ear stage (`hearing_loss_gain_db()` from `ear/ohc.py`) before computing the
metrics. State clearly that this captures reduced audibility but not *loudness
recruitment* (the abnormally fast growth of loudness in impaired ears), so the
predictions are approximate.

### 7. Sensation panel (UI)
Match the mockup:
- header: badge **4** (filled accent), title "Sensation", subtitle "Predicted:
  what a listener would report";
- four gauges, each with: name, current value with unit (e.g. "6.1 sones",
  "A3 · 220 Hz", "1.4 acum", "0.15 asper"), a horizontal bar, and plain-language
  ends with the unit between them:
  | Gauge | Left | Right | Range |
  | --- | --- | --- | --- |
  | Loudness | quiet | loud | 0–20 sones |
  | Pitch | low | high | 50 Hz–5 kHz, log scale, plus a marker line |
  | Sharpness | dull | bright | 0–4 acum |
  | Roughness | smooth | harsh | 0–1 asper |
- Pitch shows "No clear pitch" and hides its marker when `pitch_hz` is `None`;
- display smoothing so the bars glide between 0.25 s updates; frames stay unsmoothed.

## Performance budget

- One full update (all four metrics on 1 s at 48 kHz) under **200 ms** in the
  worker thread. Measure each metric separately and report the numbers in the
  write-up. If loudness is the bottleneck, explain options (shorter window,
  stationary method, resampling) and pick one.
- Stage cost inside the audio worker: under **0.5 ms** per block.
- Panel refresh: under **2 ms**.

## Acceptance criteria

`pytest tests/sensation tests/ui/test_sensation_panel.py` passes, with tests that:
- **Loudness:** a 1 kHz tone at 40 dB SPL gives **1.0 sone ± 5 %** (the definition
  of the sone); at 50 dB SPL, 2.0 sones ± 10 %;
- **equal dB, different loudness:** at 70 dB SPL, the 147 Hz hum's loudness is
  lower than the 2093 Hz whistle's (expect roughly half; report the measured ratio);
- **Sharpness:** narrow-band noise one critical band wide centred at 1 kHz, at
  60 dB SPL, gives **1.0 acum ± 10 %** (the definition of the acum);
- **Roughness:** a 1 kHz tone, 100 % amplitude-modulated at 70 Hz, at 60 dB SPL,
  gives **1.0 asper ± 15 %** (the definition of the asper);
- **Pitch:** a 440 Hz tone gives 440 Hz ± 1 %; the **missing-fundamental** stimulus
  (harmonics 3–8 of 200 Hz) gives 200 Hz ± 2 %; white noise gives `None`;
- `note_name(440.0) == "A4"`, `note_name(261.63) == "C4"`, `note_name(233.08) == "A♯3"`;
- hearing-loss mode lowers loudness and sharpness for white noise and leaves a
  250 Hz tone's loudness within 10 %;
- a worker overrun skips a cycle instead of building a backlog;
- the panel builds offscreen and shows "No clear pitch" for a `None` pitch.

## Explanation focus

Explain:
- sensation versus perception, and why these outputs are *predicted sensations*;
- the sone and phon scales, equal-loudness contours and why loudness doubles
  roughly every 10 dB;
- what sharpness and roughness measure, and the critical-band idea behind both;
- YIN step by step (difference function, cumulative mean normalization, threshold,
  parabolic interpolation), and why it hears the missing fundamental;
- why the metrics run in their own thread at a lower rate;
- the limits of applying normal-hearing models to the hearing-loss mode.
