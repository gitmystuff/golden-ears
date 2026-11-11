# 14 — Direction Card (two-mic localization)

> **Run after:** 01, 02, 03.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/panels/base.py`,
> `src/s2s/ui/theme.py`, then this file.
>
> **Owner:** inputs/outputs lead

---

## Context

We tell left from right because we have **two ears on opposite sides of the
head**. Sound reaches the nearer ear slightly earlier (the *interaural time
difference*, ITD, up to about 0.65 ms for a human head) and slightly louder (the
*interaural level difference*, ILD, mostly at high frequencies, where the head
casts an acoustic shadow). The brainstem compares the two ears in parallel with
the rest of the pathway, which is why this card has an eye icon rather than a
stage number.

In the demo, **two microphones play the part of the ears**, and two cartoon
eyes on screen follow a person walking and talking in front of the laptop.

## Goal

1. `src/s2s/direction/gcc_phat.py` — time-delay estimation.
2. `src/s2s/direction/tracker.py` — activity detection, smoothing and confidence.
3. `src/s2s/direction/stage.py` — a `Stage` that produces `DirectionFrame`.
4. `src/s2s/ui/panels/direction.py` — the eyes card.
5. `s2s mic-check` — a small CLI tool that confirms which input channel is left.

## Interface

- **Reads:** `AudioBlock` (stereo), `Config.mic_spacing_m`,
  `Config.speed_of_sound_m_s`.
- **Produces:** `DirectionFrame` every block. Sign convention from the contract:
  **negative azimuth is the viewer's left**; channel 0 is the left mic.

## Requirements

### 1. Time-delay estimation: GCC-PHAT
- Generalized cross-correlation with phase transform (Knapp & Carter, 1976) on a
  Hann-windowed analysis frame (about 2 blocks ≈ 43 ms, overlapping; justify).
- Search only physically possible lags: ±`mic_spacing_m / speed_of_sound_m_s`
  plus a small margin.
- Sub-sample accuracy by parabolic interpolation around the peak. Explain why
  this matters: at 48 kHz with 18 cm spacing the whole ±90° range spans only about
  ±25 samples.
- Compare GCC-PHAT with plain cross-correlation in the design table, especially
  for speech in a reverberant room.

### 2. Delay → angle
- Far-field free-field model: `azimuth = arcsin(c · τ / d)`, clamped to ±90°.
- Explain why a free-standing mic pair gives a smaller maximum ITD (~0.52 ms at
  18 cm) than a human head (~0.65 ms): sound travels around the head, which
  Woodworth's spherical-head formula describes.

### 3. Tracking
- **Activity detection:** update only when the frame's energy is clearly above an
  adaptive noise floor *and* the GCC peak is distinct. Set `active` accordingly.
- **Confidence:** from peak sharpness (e.g. peak-to-second-peak ratio), mapped to 0–1.
- **Smoothing:** a confidence-weighted exponential filter or a 1-D Kalman filter
  on azimuth (pick one and justify). When inactive, hold the last estimate for
  about 1 s, then ease back toward 0°.
- **Front–back ambiguity:** two mics cannot tell front from back (the *cone of
  confusion*). Document it; for the demo the talker is always in front.

### 4. Optional stretch: the brain's way
Describe, and implement only if time allows, a **per-frequency-channel**
comparison using the Ear stage's filterbank: cross-correlate left and right
channels (the Jeffress model of the medial superior olive) and compare their
levels. The expected result is the **duplex theory** (Rayleigh, 1907): timing
cues dominate in low channels, level cues in high ones.

### 5. Direction card (UI)
Match the mockup:
- header: an eye icon badge, title "Direction", subtitle "Two mics, like two
  ears, find the talker";
- two eyes (white circles with dark pupils and a small highlight), drawn with
  `QPainter`. Pupils move horizontally by `sin(azimuth)` times the maximum offset,
  so they look toward the talker from the viewer's point of view;
- readout: "35° right", "12° left" or "Straight ahead" (|azimuth| < 4°), and a
  line such as "The right mic hears it 0.31 ms sooner", or "Sound reaches both
  mics at the same moment";
- when inactive: pupils ease to centre and the readout says "Listening…";
- caption: "The eyes look where the model thinks the voice is, from tiny timing
  and loudness differences between the left and right mics."
- Pupil motion is smoothed for display (e.g. critically damped spring) so it
  looks natural; the frame values stay unsmoothed.

### 6. `s2s mic-check`
Prompts the user to tap or speak into the left mic, then the right, and reports
whether channel 0 is left. Explain why this matters: a swapped cable mirrors
every estimate.

## Performance budget

- Direction stage: under **2 ms** per block.
- Card refresh: under **1 ms**.

## Acceptance criteria

`pytest tests/direction tests/ui/test_direction_panel.py` passes, with tests that:
- white noise with the right channel delayed by k samples, k from −20 to +20,
  gives τ within **0.25 samples** of the truth;
- simulated sources at −60°, −30°, 0°, +30° and +60° (free-field delays) are
  estimated within **±2°**;
- the same at 10 dB SNR (independent noise in each channel) stays within **±5°**;
- identical left and right signals give 0°;
- **sign convention:** a source on the left (left channel leads) gives a negative
  azimuth;
- silence gives `active = False` and does not move the estimate;
- the card builds offscreen; at −30° the pupils sit left of centre.

**Manual test protocol** (record results in the write-up template): tape marks at
−60°, −30°, 0°, +30° and +60° on a 1.5 m arc in front of the laptop. Speak for
5 s at each mark. Report mean absolute error per mark. The target is ±10° in a
normal room.

## Explanation focus

Explain:
- ITD and ILD, and why each works best in a different frequency range (duplex theory);
- cross-correlation as "slide one signal over the other and find the best
  match", and what PHAT weighting adds (robustness to reverberation and to the
  talker's spectrum);
- sub-sample interpolation and angular resolution near the sides;
- free-field mic pairs versus a head, and why the cone of confusion appears;
- how turning the head would resolve front from back, which is why embodiment
  matters for robots.
