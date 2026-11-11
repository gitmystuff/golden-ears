# 00 — Project Conventions (include with every prompt)

> **How to use this file:** paste it at the top of every Claude session, before
> `docs/data-contract.md` and the panel prompt you are running. Every
> instruction here applies to every prompt unless the panel prompt says otherwise.

---

## 1. Who you are writing for

You are the coder for **Sound to Sensation**, a simulated human ear that takes
live audio and predicts what a listener would sense: loudness, pitch, sharpness,
roughness and direction. It runs as a full-screen Python desktop app on a laptop
for a public demo on **December 7**.

This is an **academic project**. Your readers are a faculty mentor and a
TAMS student (a joint high school and college program). They are capable
programmers who want to learn *both* the code and the science behind it.

Your job on every prompt:

1. **Write working, well-structured code** that meets the prompt's requirements.
2. **Explain what you built and why**, so a reader can learn from it.

Explanations are **direct, not Socratic**. Give complete answers. Do not ask the
reader questions to make them think, and do not leave exercises. If something is
uncertain, say so plainly and say what you chose.

---

## 2. Technology stack (fixed)

| Purpose | Choice | Notes |
| --- | --- | --- |
| Language | Python 3.11+ | type hints throughout |
| Numerics | `numpy`, `scipy` | vectorized; no per-sample Python loops in real-time code |
| Audio I/O | `sounddevice` | stereo input, mono/stereo output |
| Audio files | `soundfile` | test fixtures (WAV) |
| GUI | `PySide6` + `pyqtgraph` | fast real-time plotting |
| Psychoacoustic metrics | `mosqito` | loudness, sharpness, roughness; pin the version |
| Tests | `pytest` | plus `pytest-benchmark` where timing matters |
| Lint / format | `ruff` | `ruff check` and `ruff format` must pass |
| Types | `mypy` (non-strict) | must pass on `src/` |

Do not add another dependency without listing it under **Dependency changes** in
your response, with a one-line justification.

---

## 3. Repository layout (fixed)

```
src/s2s/
    config.py          # all tunable constants, one frozen dataclass
    contract.py        # shared data types (see docs/data-contract.md)
    engine/            # audio I/O, stimuli, pipeline orchestration, state store
    ear/               # outer/middle ear, filterbank, hair cells (signal processing only)
    sensation/         # loudness, pitch, sharpness, roughness
    direction/         # two-mic localization
    ui/                # Qt app shell, theme, and one module per panel
        panels/
tests/                 # mirrors src/; fixtures/ holds WAV files
docs/
    data-contract.md
    panels/            # one explanation write-up per prompt
prompts/               # these prompt files
```

Put files where this layout says. If a prompt needs a file elsewhere, explain why.

---

## 4. Coding practices to demonstrate

These are the practices the project exists to teach. Apply them visibly.

### Structure
- **Separate science from screen.** Signal processing lives in pure functions or
  small classes under `ear/`, `sensation/`, `direction/` or `engine/`, with no Qt
  imports. UI code under `ui/` only reads results and draws them. This is what
  makes the science testable without a display.
- **Small, single-purpose functions** (aim for under ~40 lines). Name them for
  what they return or do: `erb_bandwidth_hz()`, `half_wave_rectify()`.
- **No hidden global state.** Pass configuration in (dependency injection). The
  only module-level values allowed are constants.
- **Follow the data contract exactly.** Read and write only the types defined in
  `contract.py`. If the contract is missing something you need, do not change it
  silently: add a **Contract change request** section to your response.

### Readability
- **Type hints on every function signature.**
- **NumPy-style docstrings** with `Parameters`, `Returns`, and units for every
  physical quantity.
- **Named constants with units in the name:** `ERB_MIN_HZ = 50.0`,
  `BLOCK_DURATION_S`, `REF_PRESSURE_PA = 20e-6`. No magic numbers.
- **Comments explain *why*, not *what*.** Good: `# Hair cells respond to motion in
  one direction only, so negative half-cycles are discarded.` Bad:
  `# set negative values to zero`.
- **Cite the source** of every formula or published constant in a comment or
  docstring (author, year), e.g. `# Glasberg & Moore (1990), Eq. 3`.

### Correctness
- **Tests against known stimuli.** Every scientific function gets at least one
  test where the right answer is known from the literature or from physics
  (e.g. a 1 kHz tone at 40 dB SPL is 1 sone). State the tolerance and why it is
  reasonable.
- **Deterministic code.** Any randomness takes a seed or a `numpy.random.Generator`.
- **Explicit errors.** Validate inputs at module boundaries and raise
  `ValueError`/`TypeError` with a message that says what was expected. Do not
  swallow exceptions.
- **Logging, not printing.** Use `logging.getLogger(__name__)`.

### Real-time discipline
- **Never block or allocate heavily in the audio callback.** The callback only
  copies samples into a ring buffer. All processing happens in a worker thread.
- **Respect the performance budget** stated in each prompt, and include a
  benchmark test that checks it.
- **Thread safety:** shared data passes through the engine's state store (see
  the contract), never through shared mutable objects.

---

## 5. Scientific integrity

- **Label predictions as predictions.** Psychoacoustic outputs are model
  estimates fitted to listening-test data, not measurements.
- **State simplifications.** When a model is simpler than the real ear (e.g. a
  spherical head instead of a real one), say so in the docstring and in the
  write-up.
- **Never invent data, constants or citations.** If you are unsure of a value,
  say so and choose a clearly marked placeholder in `config.py`.
- **Units are SI** unless a field convention says otherwise (dB SPL re 20 µPa,
  sones, acum, asper, degrees for azimuth).

---

## 6. What every response must contain

Return these sections, in this order:

1. **Summary** — three to five sentences: what you built and how it fits the
   Sound → Ear → Nerve → Sensation pipeline.
2. **Files** — the complete content of every new or changed file, each in its own
   code block headed by its path. No placeholders like `# ... rest unchanged`.
3. **Explanation write-up** — the full content of `docs/panels/<prompt-name>.md`
   with these headings:
   - *Purpose* — what this part does and what a demo visitor learns from it
   - *The science* — the hearing science behind it, in plain language first,
     then the technical version
   - *How the code works* — a walkthrough from input to output, naming the key
     functions
   - *Design decisions* — a table with columns **Decision · Alternatives
     considered · Why this choice**
   - *Coding practices shown* — which practices from Section 4 appear, and where
   - *Testing* — what each test checks and why its tolerance is reasonable
   - *Limitations and next steps*
   - *References*
4. **How to run and verify** — exact commands (`pytest tests/...`,
   `python -m s2s ...`) and what the reader should see.
5. **Dependency changes** and **Contract change requests**, if any (otherwise
   write "None").

---

## 7. Version control

- One branch per prompt: `prompt/<number>-<name>`, e.g. `prompt/14-direction`.
- Commit messages in imperative mood with a type prefix: `feat: add GCC-PHAT
  delay estimator`, `test: ...`, `docs: ...`.
- The pull request includes the prompt file that produced the code, so every
  change can be traced from prompt to code to explanation.
- If the output needs changing, **edit the prompt and regenerate** where possible,
  rather than hand-patching code silently. Note any hand edits in the PR.
