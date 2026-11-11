# 01 — Data Contract and Project Skeleton

> **Run order:** first, before any other prompt. Both team members review the
> output before anyone runs prompts 02 onward, because every later prompt builds
> against it.
>
> **Paste into a new session:** `00-conventions.md`, then this file.
>
> **Owner:** both (joint review)

---

## Context

Sound to Sensation is split into independently built parts: an audio engine,
the ear model, and six screen panels (Sound, Ear, Nerve, Sensation, Direction,
Controls/Footer). Two people build them in parallel. That only works if every
part agrees in advance on **what data passes between them**: names, shapes,
units, update rates and who produces what. That agreement is the *data
contract*.

## Goal

Create the project skeleton and the data contract:

1. `docs/data-contract.md` — the human-readable contract.
2. `src/s2s/contract.py` — the same contract as Python types.
3. `src/s2s/config.py` — every tunable constant in one frozen dataclass.
4. The package skeleton, `pyproject.toml`, and a CI workflow.

## Decisions already made (use these)

### Signal flow

```
mic (stereo) ─┐
              ├─> engine ─> AudioBlock ─┬─> ear ─> EarFrame ─> nerve ─> NerveFrame ─┐
presets ──────┘                         │                                          ├─> sensation ─> SensationFrame
                                        ├─> level meter ─> LevelFrame              │
                                        └─> direction ─> DirectionFrame            │
UI panels read the latest frames from the StateStore on a 30 Hz timer  <───────────┘
```

### Default configuration values

| Constant | Value | Reason |
| --- | --- | --- |
| `sample_rate_hz` | 48 000 | native rate of most laptop and USB mics |
| `block_size` | 1024 samples (≈21.3 ms) | low latency while keeping per-block overhead small |
| `n_input_channels` | 2 | stereo, needed for direction |
| `n_cochlear_channels` | 64 | matches Notebook 01 |
| `cf_min_hz`, `cf_max_hz` | 50, 16 000 | span of the Ear panel's frequency labels; use Notebook 01's values instead if they differ |
| `display_fps` | 30 | smooth on screen, light on CPU |
| `nerve_history_s` | 5.0 | width of the scrolling Nerve heat map |
| `sensation_window_s` / `sensation_update_s` | 1.0 / 0.25 | psychoacoustic metrics need ~1 s of audio; update four times a second |
| `mic_spacing_m` | 0.18 | roughly head width |
| `speed_of_sound_m_s` | 343.0 | air at about 20 °C |
| `full_scale_db_spl` | 120.0 (**placeholder**) | dB SPL that a full-scale digital sine represents; calibrated on site |
| `auto_tour_step_s` | 6.0 | time per sound in the Auto tour |

### Sign convention for direction
Azimuth is in degrees from **the viewer's point of view facing the screen**:
0° is straight ahead, **negative is the viewer's left**, positive the right,
range −90° to +90°. Input channel 0 is the left mic.

### Data types to define in `contract.py`

Use `@dataclass(frozen=True, slots=True)` and NumPy arrays. Validate shapes,
dtypes and finiteness in `__post_init__`, raising `ValueError` with a clear
message.

| Type | Fields (units) | Producer → consumers | Rate |
| --- | --- | --- | --- |
| `SourceKind` (Enum) | `MIC`, `PRESET` | — | — |
| `Preset` (Enum) | `WHISTLE`, `HUM`, `WHITE_NOISE`, plus test stimuli `MISSING_FUNDAMENTAL`, `AM_TONE` | — | — |
| `AppState` | `source`, `preset: Preset \| None`, `hearing_loss: bool`, `auto_tour: bool` | Controls/Footer → engine, panels | on change |
| `AudioBlock` | `t_start_s: float`, `samples_pa: ndarray (block_size, 2) float32` (pascals), `source` | engine → ear, level meter, direction | every block |
| `LevelFrame` | `t_s`, `spl_db: float` (125 ms "fast" RMS, re 20 µPa), `waveform_pa: ndarray (M,)` (mono, decimated for display) | level meter → Sound panel | every block |
| `EarFrame` | `t_s`, `center_freqs_hz: ndarray (N,)`, `bm_rms: ndarray (N,)` (basilar-membrane RMS per channel, Pa-equivalent), `bm_level_db: ndarray (N,)` | ear → Ear panel, nerve | every block |
| `NerveFrame` | `t_s`, `rate_hz: ndarray (N,)` (mean firing rate per channel over the block, spikes/s) | nerve → Nerve panel | every block |
| `SensationFrame` | `t_s`, `loudness_sone`, `pitch_hz: float \| None`, `pitch_strength: float` (0–1), `sharpness_acum`, `roughness_asper`, `window_s` | sensation → Sensation panel | every 0.25 s |
| `DirectionFrame` | `t_s`, `azimuth_deg`, `itd_s`, `confidence: float` (0–1), `active: bool` (sound present) | direction → Direction panel | every block |

### The state store (interface only)

Define a `StateStore` **Protocol** in `contract.py`. Prompt 02 implements it.

- `publish(frame) -> None` — store a frame as the latest of its type, and append it to that type's history.
- `latest(frame_type) -> frame | None`
- `history(frame_type, seconds: float) -> list[frame]` — oldest first
- `get_state() -> AppState` and `update_state(**changes) -> AppState`
- `subscribe_state(callback) -> None` — called on every `AppState` change

All methods must be safe to call from any thread.

## Decisions for you to make and justify

- How validation in `__post_init__` stays cheap enough to run every block.
- How `config.py` is loaded and overridden (e.g. `Config()` defaults plus
  `dataclasses.replace` for tests). Do not use environment variables or files yet.
- Any extra field you believe a consumer will need. List each one under
  **Contract change requests** rather than adding it silently. These are
  proposals for the two owners to accept or reject.

## Requirements

1. `pyproject.toml` with the stack from the conventions (pinned minimum
   versions), a `s2s` console entry point, and ruff/mypy/pytest config.
2. Package skeleton matching the layout in the conventions, with an
   `__init__.py` in each package and a one-line module docstring stating each
   package's role.
3. `contract.py` with every type and the `StateStore` Protocol above.
4. `config.py` with a frozen `Config` dataclass holding every constant above,
   each with a units comment.
5. `docs/data-contract.md` containing: the signal-flow diagram, a table per type
   (field, type, shape, units, meaning), producer/consumer and rates, the
   direction sign convention, the threading rules, and a **Change process**
   section (a change needs agreement from both owners plus a version bump).
6. `CONTRACT_VERSION = "1.0.0"` in `contract.py`, quoted in the doc.
7. `.github/workflows/ci.yml` running `ruff check`, `ruff format --check`,
   `mypy src` and `pytest` on push and pull request.
8. **Extend** the repository's existing `.gitignore` (GitHub's Python template)
   rather than replacing it. Append a clearly commented project section that
   ignores editor and OS files (`.vscode/`, `.idea/`, `.DS_Store`, `Thumbs.db`),
   local profiling output, and any `recordings/` folder of personal audio, while
   keeping the small WAV files in `tests/fixtures/` tracked. Explain each addition.

## Acceptance criteria

- `pip install -e .[dev]` succeeds in a fresh virtual environment.
- `pytest` passes, including tests that:
  - construct each frame type with valid data;
  - show that each frame type rejects a wrong shape, a wrong dtype and a NaN,
    with a helpful message;
  - show that `Config` is immutable.
- `ruff check`, `ruff format --check` and `mypy src` pass.
- Each type in `contract.py` matches its table in `docs/data-contract.md`
  exactly. Add a test that checks field names agree, by parsing the doc or by
  generating the doc tables from the dataclasses (explain which and why).

## Explanation focus

In the write-up (`docs/panels/01-data-contract.md`), explain:

- why a data contract lets two people build in parallel;
- why frozen dataclasses (immutability and thread safety) and why validation
  happens at construction;
- why units live in field names, docstrings and the doc;
- the producer/consumer pattern and the "latest value" store, compared with
  alternatives (direct function calls, queues, Qt signals everywhere).
