# Proof: Exhaustive SGR Style Coverage in Mosh Rendering

## Objective

Provide a protocol-level proof that style reset handling is exhaustive for mosh's
currently supported terminal styling semantics.

This is intentionally not an implementation architecture doc. It is a coverage
argument over the actual SGR domain that mosh models.

## Scope

In scope:

- `CSI ... m` (SGR) parsing, state mutation, and rendering.

Out of scope:

- OSC (`] ...`), including clipboard/title.
- Cursor movement, scrolling, mouse modes, alt-screen, and transport.
- Features not represented in mosh's style model.

Code anchors:

- Parser: `src/terminal/terminalfunctions.cc` (`CSI_SGR`)
- Style state type: `src/terminal/terminalframebuffer.h` (`Renditions`)
- Style mutation + serialization: `src/terminal/terminalframebuffer.cc`
  (`set_rendition`, `set_foreground_color`, `set_background_color`, `sgr`)
- Emission call site: `src/terminal/terminaldisplay.cc` (`update_rendition`)

## 1) Style State Domain Is Explicit and Finite

`Renditions` stores exactly:

- Attributes bitset `A = {bold, faint, italic, underlined, blink, inverse, invisible}`
- Foreground `Fg`
- Background `Bg`

Define style state as `R = (A, Fg, Bg)`.

There is no additional style field in this model. Therefore, any "complete style
proof" must be complete over transitions in this tuple and does not require any
fourth dimension.

## 2) Supported SGR Parameter Universe

`CSI_SGR` accepts a list of integer parameters and partitions them into exactly
three parse forms:

1. Extended 256-color:

- `38;5;n` -> `set_foreground_color(n)`
- `48;5;n` -> `set_background_color(n)`

2. Extended truecolor:

- `38;2;r;g;b` -> `set_foreground_color(truecolor)`
- `48;2;r;g;b` -> `set_background_color(truecolor)`

3. Scalar SGR `Ps` -> `set_rendition(Ps)`

Inside `set_rendition`, supported scalar set is:

- Reset: `0`
- Color defaults: `39`, `49`
- ANSI fg/bg: `30..37`, `40..47`
- Bright fg/bg aliases: `90..97`, `100..107`
- Attribute set: `1`, `2`, `3`, `4`, `5`, `7`, `8`
- Attribute clear: `22`, `23`, `24`, `25`, `27`, `28`

Any scalar outside this set is ignored (no mutation).

## 3) Closure Lemma (No Hidden Style Mutation Paths)

All supported style mutations reach state only through:

- `set_rendition`
- `set_foreground_color`
- `set_background_color`

No other parser pathway writes style bits/colors for terminal content.
Therefore the reachable supported style space is closed over `R = (A, Fg, Bg)`.

## 4) Emission Lemma (Renderer Depends Only on `R`)

`FrameState::update_rendition` emits style by calling `Renditions::sgr()` when
style changed (or forced). Thus output is a function of `R_prev -> R_next`, not
of the original byte spelling used by the application.

Consequence: proving exhaustive reset handling means proving all valid transitions
over `R`, not replaying all possible input byte sequences.

## 5) Exhaustiveness Theorem

If transition logic is correct for all changes in:

- attribute bitset `A`,
- foreground `Fg`, and
- background `Bg`,

then style reset behavior is exhaustive for all SGR semantics that mosh supports.

Proof sketch:

1. By Section 2, every supported SGR parameter mutates either `A`, `Fg`, or `Bg`.
2. By Section 3, no supported style mutation escapes this tuple.
3. By Section 4, rendered bytes are produced solely from tuple transitions.
4. Therefore, complete transition coverage of the tuple is complete coverage of
   supported style-reset semantics.

## 6) Concrete Transition Classes (Complete Partition)

To satisfy the theorem, implementation/testing must cover this exact partition:

1. Attribute transitions (`A_prev -> A_next`)

- Each bit toggled set/clear for:
  `bold, faint, italic, underlined, blink, inverse, invisible`
- Coupled reset rule:
  `22` clears both `bold` and `faint`

2. Foreground transitions (`Fg_prev -> Fg_next`)

- default <-> ANSI
- ANSI <-> bright alias
- ANSI/bright <-> indexed-256
- any <-> truecolor
- any -> default (`39`)

3. Background transitions (`Bg_prev -> Bg_next`)

- same category transitions as foreground
- any -> default (`49`)

No additional transition class exists in the current model.

## 7) Why OSC Is Provably Irrelevant Here

OSC is dispatched through a separate handler and does not mutate `Renditions`
style state. Therefore OSC cannot be a missing case in this proof.

## 8) Known Non-Covered Features (By Design, Not Omission)

ECMA-48 SGR features not represented in `Renditions` (for example, attributes
with no backing field) are outside this proof domain. They are unsupported
features, not missed reset cases.

## 9) Definition of "Done" for a Reset Policy

A reset/emission strategy is semantically complete in current mosh iff:

- It is correct for every transition class in Section 6.
- It honors `22` coupling (`bold` + `faint` clear).
- It distinguishes global reset (`0`) from selective color resets (`39`, `49`).
- It does not depend on OSC/cursor/mode logic to preserve style semantics.
