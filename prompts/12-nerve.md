# 12 — Panel 3: Nerve (inner hair cells and the auditory nerve)

> **Run after:** 11 (Ear) is merged.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/panels/base.py`,
> `src/s2s/ui/theme.py`, `src/s2s/ear/stage.py`, then this file. Attach Notebook 01.
>
> **Owner:** ear-pipeline lead

---

## Context

This is **the handoff**, where sound becomes something the brain can use. When
the basilar membrane moves, the inner hair cells' stereocilia bend, ion channels
open, potassium flows in, and the cell releases neurotransmitter onto auditory
nerve fibers. The Nerve panel shows the result: a scrolling heat map of firing
rate per frequency channel over the last five seconds. Its label on screen is
"What your brain actually receives."

## Goal

1. `src/s2s/ear/hair_cell.py` — inner-hair-cell transduction.
2. `src/s2s/ear/nerve.py` — firing-rate model with adaptation.
3. `src/s2s/ear/nerve_stage.py` — an engine `Stage` that produces `NerveFrame`
   from the Ear stage's per-channel signals (as agreed in prompt 11).
4. `src/s2s/ui/panels/nerve.py` — the Nerve panel.

## Interface

- **Reads:** per-channel basilar-membrane signals from the Ear stage; `Config`.
- **Produces:** `NerveFrame` every block (mean rate per channel, spikes/s).

## Requirements

### 1. Inner-hair-cell transduction
- **Half-wave rectification:** hair cells depolarize for stereocilia deflection
  in one direction only.
- **Low-pass filter** (e.g. 2nd-order at about 1 kHz, configurable) to model the
  cell membrane's limited speed. This is why phase locking, the nerve's ability
  to fire in time with the waveform, fades above about 1–4 kHz (Johnson, 1980;
  Palmer & Russell, 1986). Carry filter state across blocks.
- Notebook 01's hair-cell stage also compressed the signal. Compression now lives
  in the Ear stage (prompt 11); do not apply it twice. State this in the write-up.

### 2. Firing-rate model
- A **saturating rate–level function** per channel: spontaneous rate (e.g.
  50 spikes/s, a high-spontaneous fiber) up to a maximum (e.g. 250 spikes/s),
  with a dynamic range of roughly 30 dB (Sachs & Abbas, 1974). Constants go in
  `Config` with sources.
- **Short-term adaptation:** a strong onset response that decays to a sustained
  rate with a time constant of tens of milliseconds (Westerman & Smith, 1984).
  Model it as a simple first-order adaptation stage; explain the choice.
- Output the **mean rate over the block** per channel as `NerveFrame.rate_hz`.
- This is a **rate model, not a spiking model.** Explain the trade-off, and point
  to the Zilany–Bruce–Carney model as the research-grade comparison (Level 2 of
  the project).

### 3. Nerve panel (UI)
Match the mockup:
- header: badge **3**, title "Nerve: the handoff", subtitle "What your brain
  actually receives"; at the right, a color legend from "less firing" to "more
  firing" using the theme heat scale;
- a heat map built from `store.history(NerveFrame, Config.nerve_history_s)`:
  rows = channels, **high frequencies at the top**, columns = time, newest at
  the right, so the image scrolls left as time passes;
- axis labels: "high", "pitch", "low" down the left; "5 seconds ago" and "now"
  along the bottom;
- color from the rate normalized between spontaneous and maximum rate, using the
  theme's 256-entry lookup table;
- use a `pyqtgraph.ImageItem` updated from a preallocated array (roll or ring
  index), not a widget per cell;
- caption: "Inner hair cells turn vibration into nerve signals. This is the moment
  sound becomes something the brain can use."

## Performance budget

- Nerve stage: under **3 ms** per block.
- Panel refresh: under **4 ms** (the heat map is the largest image on screen).

## Acceptance criteria

`pytest tests/ear/test_hair_cell.py tests/ear/test_nerve.py tests/ui/test_nerve_panel.py`
passes, with tests that:
- silence gives every channel the spontaneous rate ± 10 %;
- a 1 kHz tone at 80 dB SPL drives the 1 kHz channel to at least 90 % of the
  maximum rate (saturation);
- the rate–level function is monotonic and saturates: the rate for a tone 60 dB
  above the channel's threshold is within 5 % of the rate 80 dB above it;
- adaptation: for a 60 dB tone burst, the mean rate in the first 20 ms is at least
  1.5× the rate 200 ms later;
- phase locking: for a 200 Hz tone, the hair-cell output (before rate averaging)
  in the 200 Hz channel is strongly modulated at 200 Hz; for a 6 kHz tone, its
  modulation at 6 kHz is at least 10× smaller relative to its mean;
- block continuity, as in prompt 11;
- the panel builds offscreen and draws high-channel activity in the top rows.

Manual check: with a voice WAV, syllables appear as bright vertical bursts with
bright bands at the voice's harmonics and formants.

## Explanation focus

Explain:
- mechano-electrical transduction (stereocilia, tip links, potassium, the
  endocochlear potential as the "battery"), in plain language first;
- rectification and low-pass filtering, and how they produce phase locking and its
  high-frequency limit;
- rate–level functions, saturation and why the nerve has many fibers with
  different thresholds (the "dynamic range problem");
- adaptation and why onsets are emphasized;
- rate versus spike models, and when each is appropriate;
- the image-buffer technique for fast scrolling heat maps.
