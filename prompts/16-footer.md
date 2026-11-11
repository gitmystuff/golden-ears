# 16 — Footer: "What to Notice" Bar and Hearing-Loss Toggle

> **Run after:** 01, 02, 03. The toggle's effect on the signal is implemented in
> prompt 11 (Ear) and prompt 13 (Sensation); this prompt builds the control and
> the explanations.
>
> **Paste into a new session:** `00-conventions.md`, `docs/data-contract.md`,
> `src/s2s/contract.py`, `src/s2s/config.py`, `src/s2s/ui/theme.py`,
> `src/s2s/ui/app.py`, then this file.
>
> **Owner:** notice bar: inputs/outputs lead · toggle: ear-pipeline lead (pair on this one)

---

## Context

Visitors stay only a few seconds at a demo table. The footer gives each moment a
one-line takeaway, the **"What to notice"** bar, so the screen explains itself.
Next to it, the **Simulate hearing loss** toggle turns down the outer hair cells'
amplifier: the sound going in is identical, but the high-pitch end of the cochlea
fades and the predicted sensations drop.

## Goal

1. `src/s2s/ui/content/notices.py` — all notice text, keyed by state.
2. `src/s2s/ui/panels/footer.py` — the notice bar and the toggle.

## Interface

- **Reads:** `AppState`.
- **Writes:** `AppState.hearing_loss` (and `auto_tour = False` when the visitor
  uses the toggle, matching the controls' pause rule).

## Requirements

### 1. Notice content (data, not code)
Keep the text in one module as a mapping, so non-programmers can edit it
without touching UI code. Keys are the source/preset; a separate text is used
whenever hearing loss is on. Start from the mockup's wording:

| State | Notice |
| --- | --- |
| Live mic | Several spots glow at once. The pattern of peaks, not any single frequency, is what lets you recognize a voice. |
| Whistle | A pure whistle excites one narrow spot near the high-pitch end. One place on the membrane = one pitch you hear. |
| Hum | Same 70 dB as the whistle, yet about half as loud. Physics (dB) and sensation (sones) don't match: the ear is less sensitive to low pitches. |
| White noise | The whole membrane lights up and no single place wins, so there is no clear pitch, and it feels much louder and brighter. |
| Missing fundamental | There is no energy at 200 Hz, yet the model reports a 200 Hz pitch: pitch comes from the spacing of the harmonics. |
| AM tone | A tone that wobbles 70 times a second sounds rough: too fast to hear as a beat, too slow to blend into a smooth tone. |
| Any, hearing loss on | Hearing-loss mode turns down the outer hair cells' amplifier. The sound going in is identical, but the high-pitch end fades, and loudness and sharpness drop with it. |

Before the demo, check each claim against the real model's output (e.g. the
"about half as loud" ratio from prompt 13's test) and adjust the wording to match.

### 2. Footer (UI)
Match the mockup:
- left, filling the width: a card with "WHAT TO NOTICE" (accent, uppercase) and the
  current notice; a short cross-fade (~200 ms) when the text changes;
- right: a **Simulate hearing loss** switch (track + knob, accent when on),
  64 px tall, built as a checkable `QAbstractButton` subclass so it has real
  keyboard focus, Space to toggle and an accessible name and checked state;
- the footer follows `AppState`, including changes made by the Auto tour.

## Acceptance criteria

`pytest tests/ui/test_footer.py tests/ui/test_notices.py` passes, with tests that:
- **every `Preset` value and the live mic has a notice.** Iterate over the enum,
  so adding a preset without a notice fails the test (explain this pattern);
- the hearing-loss text overrides the others whenever `hearing_loss` is on;
- clicking or pressing Space on the switch flips `AppState.hearing_loss` and sets
  `auto_tour = False`;
- a state change from the tour updates the notice and the switch without a click;
- the switch exposes an accessible name and checked state;
- no notice is longer than fits on one line at 1280 px wide (measure with
  `QFontMetrics`) — or the bar wraps to two lines cleanly; pick one and test it.

## Explanation focus

Explain:
- separating content from presentation, and who benefits;
- the "every enum value is covered" test pattern;
- building an accessible custom switch on top of a real button class;
- why each notice must be checked against model output before the demo
  (scientific honesty in public communication).
