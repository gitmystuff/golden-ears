# 03 — App Shell and Theme

> **Run after:** 01 (data contract). Can run in parallel with 02.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, then this file. Attach a
> screenshot of the mockup if you have one.
>
> **Owner:** inputs/outputs lead

---

## Context

The demo runs as one full-screen window on a laptop. It reads left to right as
the auditory pathway: **1 Sound → 2 Ear → 3 Nerve → 4 Sensation**, with a
**Direction** card under Sensation, a header with sound controls and a footer
with a "What to notice" bar and a hearing-loss toggle. Each panel is built by a
separate prompt, so the shell must give every panel the same frame to slot into.

### Layout (1440 × 900 reference; must scale to other screen sizes)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Sound to Sensation              [Auto tour] | Live mic  Whistle  Hum  Noise │  header
├────────────┬──────────────────────────────────┬──────────────────────────┤
│ 1 Sound    │ 2 Ear (uncoiled cochlea strip)   │ 4 Sensation (4 gauges)   │
│ waveform   ├──────────────────────────────────┤                          │
│ dB SPL     │ 3 Nerve (scrolling heat map)     ├──────────────────────────┤
│            │                                  │ Direction (two eyes)     │
├────────────┴──────────────────────────────────┴──────────────────────────┤
│ WHAT TO NOTICE  one-line explanation…              [ ◯ Simulate hearing loss ] │  footer
└──────────────────────────────────────────────────────────────────────────┘
```

### Visual style (from the mockup)

| Token | Value | Use |
| --- | --- | --- |
| background | `#0E1116` | window |
| panel | `#161B22`, border `#232A35`, radius 16 px | cards |
| text | `#E6EDF3`; secondary `#B8C1CC`; muted `#8B95A3` | all text passes 4.5:1 contrast on panel |
| accent | `#F5A524` | gauges, selected buttons, step numbers |
| sound blue | `#7FB3FF` | waveform and dB meter |
| heat scale | `#1A2029 → #7A3410 → #F08A24 → #FFE9B0` | Ear and Nerve activity |
| fonts | Space Grotesk (headings), Inter (body); fall back to system sans | bundled under `ui/assets/fonts/` with their licenses |

## Goal

Build `src/s2s/ui/app.py`, `ui/theme.py` and `ui/panels/base.py`, so that each
later panel prompt only has to write one panel class.

## Requirements

1. **Theme module** (`ui/theme.py`): every color, font, size and spacing above as
   named constants, plus a `heat_color(value: float) -> QColor` function and a
   precomputed 256-entry lookup table for fast heat maps. No colors hard-coded
   anywhere else in `ui/`.
2. **Panel base class** (`ui/panels/base.py`): `Panel(QWidget)` with
   - a header row: step badge (number or icon), title, one-line subtitle;
   - a content area for the subclass;
   - an optional caption line at the bottom;
   - `refresh(store: StateStore) -> None`, which subclasses override and which
     must not block (read the latest frames, update the drawing, return).
3. **Main window** (`ui/app.py`):
   - builds the layout above with Qt layouts (no absolute positioning), so it
     scales to 1280 × 800 through 1920 × 1080;
   - one `QTimer` at `Config.display_fps` that calls `refresh(store)` on every panel;
   - placeholder panels (title plus "coming soon") for any panel not yet built,
     so the app runs from day one;
   - full-screen toggle (F11) and Esc to leave full screen;
   - starts and stops the engine cleanly when the window opens and closes.
4. **Entry point:** `s2s run` launches the app; `--wav PATH` uses a WAV source;
   `--windowed` skips full screen.
5. **Frame-time monitor:** an optional overlay (toggle with F12) showing refresh
   time per panel, so slow panels are easy to spot.

## Performance budget

All panel refreshes together must finish in under **12 ms** (leaving headroom in
a 33 ms frame at 30 fps). The base class measures each panel's refresh time.

## Acceptance criteria

- `s2s run --windowed --wav tests/fixtures/<any>.wav` opens the full layout with
  placeholder panels and no errors.
- `pytest tests/ui` passes, using `pytest-qt` (add it as a dev dependency), with tests that:
  - construct the main window offscreen (`QT_QPA_PLATFORM=offscreen`);
  - check the timer calls `refresh` on every panel;
  - check `heat_color(0)` and `heat_color(1)` return the end colors and the
    lookup table is monotonic in brightness;
  - check every theme text color meets 4.5:1 contrast against the panel color
    (compute WCAG relative luminance).
- Resizing between 1280 × 800 and 1920 × 1080 keeps every panel visible.

## Explanation focus

Explain:

- the template-method pattern in `Panel` (base class owns the frame; subclasses
  fill the content), and why it keeps six panels consistent;
- why one display timer polls the store instead of every frame triggering a
  redraw (frame rate decoupled from block rate);
- why design tokens live in one theme module;
- how the contrast test turns an accessibility rule into an automated check.
