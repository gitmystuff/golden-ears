# 02 — Audio Engine

> **Run after:** 01 (data contract) has been reviewed and merged.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, then this file.
>
> **Owner:** ear-pipeline lead

---

## Context

The engine is the heart of the app. It does not do any hearing science itself.
Its job is to **get sound in, cut it into blocks, run the processing stages on
each block, and hand the results to the panels**, reliably and on time. Every
panel depends on it, so it must be dependable, well tested and easy to run
without the full app.

```
sound in ──> ring buffer ──> worker thread ──> stages ──> StateStore ──> panels
(mic / preset / WAV)          (one block at a time)  (ear, nerve,      (latest value
                                                      level, direction) + history)
```

## Goal

Build `src/s2s/engine/` so that, with any set of processing stages plugged in,
the app turns live or recorded audio into contract frames in real time.

## Interface (from the contract)

- **Produces:** `AudioBlock` (every block), and the `StateStore` implementation.
- **Reads:** `AppState` (source, preset), `Config`.
- **Runs:** any object that satisfies a `Stage` protocol you define here:
  `process(block: AudioBlock, state: AppState) -> list[Frame]`. Stages for the
  ear, nerve, level meter and direction come from later prompts. Provide one
  trivial example stage so the engine runs end to end today.

## Requirements

### 1. State store (`engine/store.py`)
Implement the contract's `StateStore` Protocol.
- Thread-safe (a single `threading.Lock` is fine; justify it).
- Bounded history per frame type, sized from `Config` (e.g. `nerve_history_s`),
  using `collections.deque(maxlen=...)` so memory cannot grow without limit.
- `subscribe_state` callbacks run outside the lock, so a slow callback cannot
  stall the audio worker.

### 2. Stimuli (`engine/stimuli.py`)
Pure functions that return mono signals **in pascals** for a given duration,
level (dB SPL) and sample rate:
- `pure_tone(freq_hz, level_db_spl, duration_s, fs)`
- `harmonic_complex(f0_hz, harmonics, level_db_spl, duration_s, fs)`, which
  supports the missing fundamental by omitting low harmonics
- `white_noise(level_db_spl, duration_s, fs, rng)`
- `am_tone(carrier_hz, mod_hz, depth, level_db_spl, duration_s, fs)`
- `level_to_rms_pa()` and `rms_pa_to_level()` helpers (re 20 µPa)

Short raised-cosine onset and offset ramps (e.g. 10 ms) prevent clicks. Explain
why clicks are a spectral problem, not just an annoyance.

Preset definitions (a table in code, not scattered literals):

| Preset | Signal | Level |
| --- | --- | --- |
| `WHISTLE` | pure tone, 2093 Hz (C7) | 70 dB SPL |
| `HUM` | pure tone, 147 Hz (D3) | 70 dB SPL |
| `WHITE_NOISE` | white noise | 70 dB SPL |
| `MISSING_FUNDAMENTAL` | harmonics 3–8 of 200 Hz | 60 dB SPL |
| `AM_TONE` | 1 kHz carrier, 100 % AM at 70 Hz | 60 dB SPL |

Presets loop seamlessly (generate a buffer whose length is a whole number of
periods, or crossfade the loop point; explain which you chose).

### 3. Sources (`engine/sources.py`)
Three interchangeable sources behind one interface that yields stereo blocks
in pascals:
- **Mic:** `sounddevice.InputStream`, stereo, `Config.sample_rate_hz` and
  `block_size`. Converts digital samples to pascals using
  `Config.full_scale_db_spl`.
- **Preset:** plays the preset through the speakers **and** feeds the same
  clean digital signal (identical in both channels) into the pipeline. Using
  the clean signal rather than re-recording it avoids room echo and feedback.
  Explain this choice.
- **WAV file:** reads a file with `soundfile`, optionally in real time or as fast
  as possible. This is how every other prompt tests without a mic.

Switching source follows `AppState` changes and must not glitch or crash.

### 4. Audio callback and ring buffer (`engine/audio_io.py`)
- The `sounddevice` callback **only copies** incoming samples into a
  preallocated ring buffer. No processing, no allocation, no locks held for long.
- Log input overflows and count them (they mean the worker fell behind).

### 5. Worker and pipeline (`engine/pipeline.py`)
- A worker thread pulls one block at a time, wraps it in an `AudioBlock`, runs
  each registered stage in order, and publishes every returned frame to the store.
- Timestamps come from a **sample counter** (`samples_so_far / fs`), not the wall
  clock, so results are reproducible.
- Measure each stage's processing time. Keep a rolling average, and log a warning
  if the total exceeds 80 % of the block duration.
- Clean start/stop: `Engine.start()`, `Engine.stop()`, usable as a context manager.

### 6. Calibration helper (`engine/calibration.py`)
A small CLI flow (`s2s calibrate`) that plays or records a reference and reports
the `full_scale_db_spl` value to put in config. Support: "hold a phone SPL-meter
app next to the mic, play pink noise, type the reading." State clearly that this
gives approximate absolute levels (±3 dB is realistic) and why that is fine for
a demo.

### 7. Headless runner
`python -m s2s.engine --wav tests/fixtures/<file>.wav --seconds 5` runs the
pipeline without a GUI and prints a short summary per frame type (count, rate,
mean processing time). Every panel owner uses this to test their stage.

## Performance budget

- Engine overhead (excluding stages) under **2 ms per block** on a typical laptop.
- Total time per block, stages included, under the block duration
  (≈21.3 ms), with a warning at 80 %.

## Acceptance criteria

- `pytest tests/engine` passes, including tests that:
  - check each stimulus has the requested RMS level within ±0.1 dB and the
    requested frequency (FFT peak within one bin);
  - show the missing-fundamental stimulus has **no energy at 200 Hz**;
  - check the store is thread-safe under concurrent publish/read (several threads,
    no exceptions, history never exceeds its bound);
  - check timestamps advance by exactly `block_size / fs` per block;
  - run a WAV source through the engine with a dummy stage and check the frame
    count matches the file length;
  - benchmark engine overhead per block.
- The headless runner works on a WAV fixture.
- Unplugging or lacking a mic produces a clear logged error, not a crash.
- Add test fixtures under `tests/fixtures/`: generate them with a script
  (`tests/fixtures/make_fixtures.py`) from the stimulus functions, plus a slot for
  one short real speech recording that the team adds.

## Explanation focus

Explain in the write-up:

- the producer/consumer pattern, and why the audio callback must stay tiny
  (what happens in an overflow);
- ring buffers versus queues for audio;
- why timestamps come from sample counts;
- digital full scale versus dB SPL, and what calibration can and cannot promise;
- why the preset source feeds the clean signal into the pipeline.
