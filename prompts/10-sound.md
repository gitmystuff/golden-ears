# 10 — Panel 1: Sound

> **Run after:** 01, 02 and 03 are merged.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/panels/base.py`,
> `src/s2s/ui/theme.py`, then this file.
>
> **Owner:** inputs/outputs lead

---

## Context

The Sound panel is the first stage a visitor sees: **the physics going in**. It
shows the pressure wave reaching the "eardrum" and its level in dB SPL. Its
teaching job is to set up the contrast the rest of the screen pays off: *the
same dB number can feel very different to a listener.*

## Goal

1. A **level-meter stage** (`src/s2s/engine/level.py`) that turns each
   `AudioBlock` into a `LevelFrame`.
2. A **Sound panel** (`src/s2s/ui/panels/sound.py`) that draws it.

## Interface

- **Reads:** `AudioBlock` (via the stage), `LevelFrame` (via the store), `Config`.
- **Produces:** `LevelFrame` every block.

## Requirements

### Level-meter stage (science; no Qt)
1. Mix the two channels to mono by averaging. Explain why averaging, not summing,
   keeps the level meaning consistent.
2. Compute sound pressure level with **"fast" exponential time weighting
   (τ = 125 ms)**, as in sound level meters (IEC 61672-1), unweighted (Z). Keep
   the filter state across blocks so the meter is continuous.
3. Convert to dB SPL re 20 µPa. Clamp very quiet values to a floor (e.g. 0 dB SPL)
   rather than returning −∞, and explain why.
4. Produce `waveform_pa` for display as an **oscilloscope-style trace**:
   - the most recent ~40 ms of mono audio;
   - aligned on a rising zero crossing (a "trigger"), so periodic sounds stand
     still on screen instead of jittering;
   - decimated to about 240 points using **min/max pairs per bucket**, so short
     peaks are not lost. Explain why plain downsampling would alias or hide peaks.

### Sound panel (UI)
Match the mockup:
- header: badge **1**, title "Sound", subtitle "What goes into the ear";
- label "Air pressure at the eardrum" above a waveform plot (`pyqtgraph`,
  sound-blue line, fixed y-range so loudness changes are visible);
- label "Sound level", a large number (e.g. "68") with "dB SPL", and a horizontal
  bar from 0 to 120 dB with tick labels 0 / 60 / 120;
- caption: "This is **physics**: how strong the pressure wave is. The same number
  can feel very different to a listener."
- The dB number updates at most ~5 times a second (display smoothing), so it is
  readable. Keep the science value unsmoothed in the frame; smoothing is a display
  concern only.

## Performance budget

- Stage: under **1 ms** per block.
- Panel refresh: under **2 ms**.

## Acceptance criteria

`pytest tests/engine/test_level.py tests/ui/test_sound_panel.py` passes, with
tests that:
- a 1 kHz tone at 70 dB SPL reads **70.0 ± 0.3 dB** once settled (after > 1 s);
- after a step from silence to a steady tone, the meter is within 63 % ± 5 % of the
  final pressure level after 125 ms (time constant check, in the linear domain);
- silence reads the floor value and never NaN or −∞;
- for a steady 200 Hz tone, the trigger keeps the waveform's first rising zero
  crossing within one display point across consecutive frames;
- min/max decimation preserves a single-sample spike;
- processing a signal in blocks gives the same final level as processing it in
  one piece (state continuity);
- the panel builds offscreen and `refresh` updates the number and the trace from a
  fake store.

Manual check: `s2s run --windowed --wav tests/fixtures/<voice>.wav` shows a
moving voice waveform and a dB number that rises and falls with speech.

## Explanation focus

Explain:
- dB SPL, the 20 µPa reference, and why a logarithmic scale is used;
- exponential time weighting and why sound level meters use it;
- why the absolute level depends on calibration (prompt 02), and what "±3 dB"
  means for the demo;
- triggering and min/max decimation as display techniques borrowed from
  oscilloscopes;
- why science values and display smoothing are kept separate.
