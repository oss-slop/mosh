# Implementation Plan: SGR Reset Semantics (No OSC Scope)

## Goal
Implement the style-rendering change proven in `docs/reset-proof.md`:
- avoid unconditional `SGR 0` on every rendition update,
- emit transition-aware SGR for supported style state,
- keep scope strictly to SGR style rendering.

## Non-Goals
- No OSC changes.
- No transport/network changes.
- No unrelated parser rewrites.

## Intended Code Changes
1. `src/terminal/terminalframebuffer.h`
- Add a transition serializer API for renditions (current -> target).
- Keep public API narrow; transition logic stays inside `Renditions`.

2. `src/terminal/terminalframebuffer.cc`
- Keep SGR state mutation support (including faint + `22` coupling semantics).
- Add transition serializer implementation that:
  - diffs attributes and colors,
  - emits selective reset/set codes (`22/23/24/25/27/28`, `39/49`),
  - emits target color codes (`30-37`, `38;5;n`, `38;2;r;g;b`, and bg variants),
  - avoids unconditional leading `0` for incremental transitions.
- Keep absolute serializer (`sgr()`) for forced/unknown-state synchronization.

3. `src/terminal/terminaldisplay.cc`
- Use transition serializer when updating rendition from known prior state.
- Keep forced path behavior for unknown/initial synchronization.

4. `src/tests/*`
- Remove temporary narrow hack-style faint regression test introduced during exploration.
- Replace with transition-focused regression coverage that proves:
  - adding faint does not implicitly reset existing non-default foreground color,
  - SGR transition emitted by mosh is semantically equivalent to direct path.
- Update `src/tests/Makefile.am` test list accordingly.

## Validation Plan
1. Run focused emulation tests covering attributes and new transition regression.
2. Verify no unrelated files changed.
3. Keep changes local in the working tree until review is complete.

## Risk Notes
- Largest risk is attribute reset ordering, especially bold/faint via `22`.
- Secondary risk is over-emitting color resets when no color transition occurred.
- Mitigation: encode transition logic directly from `(A, Fg, Bg)` diff and test coupled intensity transitions.
