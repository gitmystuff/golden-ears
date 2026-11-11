# 11 — Panel 2: Ear (outer/middle ear, cochlea, outer hair cells)

> **Run after:** 01, 02 and 03 are merged.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/panels/base.py`,
> `src/s2s/ui/theme.py`, then this file. **Also attach Notebook 01
> (`01_ear_simulation_level1.ipynb`)** so its verified code can be ported.
>
> **Owner:** ear-pipeline lead

---

## Context

This is the scientific core. The cochlea sorts sound by frequency: each place
along the basilar membrane vibrates most for one frequency (*tonotopy*), high
pitches near the base and low pitches near the apex. The outer hair cells
amplify quiet sounds and compress loud ones. The Ear panel shows this as the
cochlea **uncoiled into a strip** that glows where the membrane vibrates.

Notebook 01 already built and verified stages 1–3b offline. This prompt turns
that work into **real-time, block-by-block code**, adds the outer-hair-cell
amplifier with a hearing-loss mode, and draws the panel.

## Goal

1. `src/s2s/ear/outer_middle_ear.py` — outer/middle-ear filter.
2. `src/s2s/ear/filterbank.py` — ERB-spaced gammatone filterbank.
3. `src/s2s/ear/ohc.py` — outer-hair-cell amplifier/compressor and hearing loss.
4. `src/s2s/ear/stage.py` — an engine `Stage` that produces `EarFrame`.
5. `src/s2s/ui/panels/ear.py` — the Ear panel.

## Interface

- **Reads:** `AudioBlock`, `AppState.hearing_loss`, `Config`.
- **Produces:** `EarFrame` every block. Also exposes the per-channel
  basilar-membrane signals (time domain, after compression) to the nerve stage
  (prompt 12) in-process. Propose how, as a **Contract change request** if it
  needs a new type; the simplest acceptable option is that prompt 12's stage
  wraps this one.

## Requirements

### Port from Notebook 01
Reuse its verified formulas and constants (ERB scale, gammatone filters,
middle-ear transfer function), keeping its built-in checks as unit tests. Where
you change something, say what and why in the write-up.

### 1. Outer/middle-ear filter
- Boosts the ear-canal resonance near 3 kHz (about +10 dB), with low-frequency
  roll-off (ossicle stiffness) and high-frequency roll-off (ossicle mass), as in
  Notebook 01.
- Implemented as IIR second-order sections (`scipy.signal.sosfilt`) **with state
  carried between blocks**.

### 2. Gammatone filterbank
- `Config.n_cochlear_channels` channels, centre frequencies equally spaced on the
  ERB-number scale from `cf_min_hz` to `cf_max_hz` (Glasberg & Moore, 1990).
- 4th-order gammatone filters with bandwidth 1.019 × ERB (Patterson et al., 1992).
- A recursive (IIR) implementation suitable for streaming, vectorized across
  channels, with per-channel state between blocks. Compare it with an FIR
  (impulse-response convolution) approach in the design-decisions table.
- Gain normalized so each channel passes its centre frequency at 0 dB.

### 3. Outer-hair-cell amplifier and compression
- A **broken-stick** input–output function per channel: linear gain below a knee,
  compressive above it (slope about 0.25–0.3 dB/dB), approximating
  basilar-membrane measurements (Ruggero et al., 1997) and the DRNL model's
  nonlinear path (Lopez-Poveda & Meddis, 2001). Apply it to each channel's
  instantaneous signal so phase is preserved.
- Notebook 01 placed compression in the hair-cell stage. Move it here, since
  outer hair cells on the basilar membrane do the compressing in the real ear, and
  the hearing-loss mode must act on it. Explain this change.
- **Hearing-loss mode** (`AppState.hearing_loss`): reduce the active gain in a
  sloping pattern like age-related hearing loss, little change below 1 kHz and
  rising to a configurable maximum (e.g. 40 dB) by 8 kHz. Expose
  `hearing_loss_gain_db(cf_hz) -> ndarray` as a public function, because
  prompt 13 reuses it. Explain that this is a simplified model, not a clinical one.
- Switching the mode must not click: ramp the gain change over ~50 ms.

### 4. Ear stage → `EarFrame`
- `center_freqs_hz`, `bm_rms` (per-channel RMS over the block), `bm_level_db`
  (dB SPL-equivalent).

### 5. Ear panel (UI)
Match the mockup:
- header: badge **2**, title "Ear", subtitle "The cochlea sorts sound by pitch";
  at the right, a small coiled-cochlea icon, an arrow, and "Cochlea, uncoiled";
- labels above the strip: "**High pitches** · base" (left) and "**Low pitches** ·
  apex" (right). The base is on the left, so **frequency decreases left to right**;
- the strip: one segment per channel (or about 48 merged segments), colored with
  `theme.heat_color` from `bm_level_db` mapped over a configurable range
  (e.g. 20–90 dB → 0–1);
- frequency labels under the strip at 16k, 8k, 4k, 2k, 1k, 500, 250, 125 Hz, each
  placed at the position of its channel (not evenly spaced, unless the channel
  spacing makes them so);
- caption: "Each pitch makes a different spot on the membrane vibrate. Brighter
  means stronger vibration."
- Display smoothing: fast attack, ~150 ms decay, so the strip reads clearly;
  science values stay unsmoothed.

## Performance budget

- Ear stage (all four steps, 64 channels × 1024 samples): under **6 ms** per block.
- Panel refresh: under **2 ms**.

## Acceptance criteria

`pytest tests/ear tests/ui/test_ear_panel.py` passes, with tests that:
- ERB bandwidths match Glasberg & Moore (1990) within 1 % at 100 Hz, 1 kHz and 8 kHz;
- pure tones at 250 Hz, 1 kHz and 4 kHz produce their largest `bm_rms` in the
  channel whose centre frequency is nearest (±1 channel);
- each channel's gain at its centre frequency is 0 dB ± 0.5 dB before the outer-hair-cell stage;
- the middle-ear response peaks between 2.5 and 3.5 kHz;
- **block continuity:** a signal processed in 1024-sample blocks matches the same
  signal processed in one piece (max absolute difference < 1e-6 relative to peak);
- compression: a 1 kHz tone 20 dB above the knee raises the 1 kHz channel's output
  by 5–7 dB (about 2× in amplitude), as Notebook 01 verified;
- hearing loss: at 60 dB SPL, channels above 4 kHz drop by at least 20 dB, while
  channels below 500 Hz change by less than 1 dB;
- toggling hearing loss produces no step larger than the ramp allows (no click);
- benchmark: stage under 6 ms per block;
- the panel builds offscreen, and a 4 kHz tone lights segments toward the left
  (base), while a 250 Hz tone lights segments toward the right (apex).

## Explanation focus

Explain:
- tonotopy, the ERB scale and why channels are not spaced evenly in hertz;
- what a gammatone filter is (a tone inside a gamma envelope) and the
  time–frequency trade-off it shows;
- IIR versus FIR for streaming, and why filter state must cross block boundaries;
- the outer hair cells as an amplifier, compression, and why loudness grows more
  slowly than intensity;
- what the hearing-loss mode does and does not capture;
- why compression moved from the hair-cell stage (Notebook 01) to here.
