# Terminal Rendering Investigation: SGR Reset Canonicalization

## Scope

This document records the debugging process around style/color drift observed
in mosh sessions compared to OpenSSH passthrough, with focus on `SGR 2` (faint)
and `SGR 38;5;244` (gray).

## Reported Behavior

- OpenSSH path: expected gray/faint rendering (both inside and outside tmux).
- mosh path: text expected to appear gray/faint appears close to white.
- Issue reproduces in mosh alone and in mosh + tmux.
- TERM and color-count checks reported `xterm-256color` and `256`.

## Repro Snippets

```sh
printf '\e[2mexplain this codebase\e[0m\n'
printf '\e[38;5;244mexplain this codebase\e[0m\n'
```

## Test Matrix Snapshot (2026-03-01)

Observed user-level matrix so far:

1. Primitive faint test
```sh
printf '\e[2mexplain this codebase\e[0m\n'
```
- SSH: appears gray/faint as expected.
- mosh: appears close to white (problem case).

2. Primitive explicit gray test
```sh
printf '\e[38;5;244mexplain this codebase\e[0m\n'
```
- SSH: gray.
- mosh: gray.

3. Equivalent-style spellings test
```sh
printf '\e[38;5;244mgray244\e[0m  \e[90mbright-black\e[0m  \e[2mfaint\e[0m\n'
printf '\e[38;5;244mgray244\e[39m  \e[38;5;8mbright-black\e[39m  \e[2mfaint\e[22m\n'
```
- User report: these two render identically.
- Interpretation: `0` vs `39` and `0` vs `22` spelling alone does not explain the visual mismatch.

4. Colon-form vs semicolon-form color test
```sh
printf '\e[38:5:244mCOLON-244\e[0m  \e[38;5;244mSEMI-244\e[0m\n'
printf '\e[38:2::180:180:180mCOLON-RGB\e[0m  \e[38;2;180;180;180mSEMI-RGB\e[0m\n'
```
- User report: SSH and mosh looked identical.
- Lab note: in raw non-tmux mosh capture, colon-form colors are dropped while semicolon-form are preserved. This can be masked when running inside tmux (tmux may normalize before mosh sees output).

Current conclusion from matrix:
- The issue is not explained by simple reset spelling differences alone.
- The mismatch is likely context-dependent (full TUI render path and/or inherited style state), not a single isolated SGR primitive.

5. Intensity-only vs explicit-color+intensity (latest user check)
```sh
printf '\e[22m\e[2mfaint\e[0m\n'
printf '\e[2mfaint\e[0m\n'
printf '\e[38;5;244m\e[2mfaint-gray\e[0m\n'
```
- User report over mosh:
  - first two (`faint`) appear white,
  - `faint-gray` appears light gray.
- User report over SSH:
  - all three appear light gray.

Interpretation:
- `SGR 2` is not universally ignored in mosh path (it works when foreground is explicit gray).
- Remaining divergence appears tied to `dim + default-foreground` context rather than `dim` itself.

## Byte-Level Parity Notes

Additional capture on 2026-03-01 for `mosh -> tmux -> shell printf` showed:

- Payload rendered by mosh includes:
  - `\e[38;5;244mgray244\e[39m`
  - `\e[38;5;8mbright-black\e[39m`
  - `\e[2mfaint`
- Immediately after payload, tmux status line update emits:
  - `\e[22;30;42m...`

Interpretation:
- Faint is not missing from the byte stream in this layered path.
- Any remaining visual mismatch is likely due to contextual rendering interaction
  (timing/order/frame update behavior) rather than complete loss of `SGR 2`.

### Non-tmux Lab Repro (Important)

In raw non-tmux mosh capture, colon-form SGR color (`38:...` / `48:...`) is not
preserved, while semicolon-form (`38;...` / `48;...`) is preserved.
This is a separate parser-coverage issue and may or may not be the user's active
symptom depending on whether tmux normalization is in the path.

## Debugging Timeline (Objective)

1. Confirmed that OpenSSH preserves emitted bytes from applications.
2. Confirmed that mosh does not pass through raw SGR bytes; it parses terminal state and re-renders from framebuffer state.
3. Identified renderer path where SGR strings are generated:
   - `src/terminal/terminalframebuffer.cc` (`Renditions::sgr`)
   - `src/terminal/terminaldisplay.cc` (`FrameState::update_rendition`)
4. Verified current renderer emits canonicalized sequences starting with `\033[0` before active attributes/colors.
5. Initially investigated OSC52-related behavior in parallel; this was orthogonal to the faint/gray rendering issue.

## What "Canonicalization" Means Here

Application-emitted bytes are parsed into internal style fields, then re-serialized.

Example forms:

- Input style add: `\e[2m` (faint)
- Rendered canonical form: `\e[0;2m`

- Input style add: `\e[38;5;244m`
- Rendered canonical form: `\e[0;38;5;244m`

This is syntactically valid SGR, but it is not byte-identical to OpenSSH passthrough.

## Why Leading `0` Can Matter

`SGR 0` is a global reset. Prepending it changes semantics from incremental
update to reset-and-reapply.

Concrete non-equivalent composition example:

```sh
# Preserves existing gray then adds faint:
printf '\e[38;5;244mA\e[2mB\e[0m\n'

# Reset + faint drops prior gray unless reapplied:
printf '\e[38;5;244mA\e[0;2mB\e[0m\n'
```

## Attempted Change(s) During Investigation

- Added faint handling in renderer/state path and a regression test
  (`emulation-faint.test`) in this working tree.
- This validated retention of faint state but did not fully explain parity
  differences with OpenSSH in all render stacks.
- OSC52 parsing/tests were explored in parallel, then deprioritized for this
  issue.

## Current Working Hypothesis

The primary parity gap is the reset-heavy SGR emission strategy (`\e[0;...m`)
rather than missing syntax support for faint itself.

## Risk Assessment (Pre-Implementation)

### Surface Area

- `src/terminal/terminalframebuffer.cc` (`Renditions::sgr`): main emission policy.
- `src/terminal/terminaldisplay.cc` (`update_rendition`): call-site frequency and ordering.
- Existing display tests in `src/tests/emulation-*` may require updates if emission strategy changes canonical output bytes.

### Technical Risks

1. **Behavioral drift in existing output diffs**
   - Many tests compare mosh vs direct capture. Changing canonical output can flip expected equality in edge cases.
2. **Attribute reset interactions**
   - Selective resets require careful use of SGR reset codes:
     - `22` resets both bold and faint.
     - `39`/`49` reset fg/bg only.
3. **State transition complexity**
   - A diff-based emitter must correctly handle all transitions (set, clear, color mode switches, truecolor/256/ANSI).
4. **Performance risk (low to moderate)**
   - Additional comparison logic per style update is small but non-zero in hot rendering paths.

### Mitigations

- Keep first implementation minimal: avoid unconditional leading `0`; emit selective resets/sets from previous→next rendition.
- Add targeted regression coverage for faint + 256-color + composition ordering.
- Validate with existing emulation suite before broad changes.

## Reset Philosophy (Design Guidance)

The renderer should treat SGR output as a state transition problem (`previous -> next`), not as "always rebuild from zero".

Principles:

- Preserve semantics first, minimize bytes second.
- Avoid `SGR 0` unless there is no safe selective expression for the transition.
- Prefer selective reset/set codes (`22`, `23`, `24`, `25`, `27`, `28`, `39`, `49`) over global reset.
- Keep emission deterministic so tests can assert exact output forms.

Practical casework required:

- Intensity (`bold`/`faint`): use `22` when either bit must be cleared, then reapply `1` and/or `2` as needed.
- Italic/underline/blink/inverse/invisible: use paired clear codes (`23`, `24`, `25`, `27`, `28`) and set codes (`3`, `4`, `5`, `7`, `8`) based on bit diffs.
- Foreground color: if changed, emit target color directly (`30-37`, `90-97`, `38;5;n`, `38;2;r;g;b`) or `39` for default.
- Background color: if changed, emit target background (`40-47`, `100-107`, `48;5;n`, `48;2;r;g;b`) or `49` for default.

When `SGR 0` is still acceptable:

- Terminal state is unknown or intentionally reinitialized.
- A one-shot full reset is explicitly desired at frame/session boundaries.

When `SGR 0` should be avoided:

- Mid-frame incremental style transitions where existing color/style context must be preserved.
- Any path where OpenSSH parity is a goal for style composition behavior.

## Proposed Next Sequence

1. Revert temporary/hack-style debug changes (faint-specific quick patch + temporary test additions) to establish clean baseline.
2. Design a delta SGR emitter strategy (previous rendition -> target rendition) with explicit reset semantics.
3. Implement in renderer path with focused tests for:
   - `2` vs `0;2` behavior across composition.
   - `38;5;244` persistence when additional attributes are added.
4. Run emulation tests and compare against OpenSSH behavior with the repro snippets.
