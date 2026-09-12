# Changelog

All notable changes to progress-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `progstate` — `ProgUnit`, `ProgPhase`, `ProgSmoothing`, `ProgSample`
  and `ProgState`; `start`, `start_spinner`, `advance`,
  `set_position`, `finish`, `abandon` and the builders; and the
  measurements `fraction`, `remaining`, `elapsed`, `rate`, `mean_rate`,
  `eta`, `eta_at`, `has_eta`, `eta_duration`, `elapsed_duration`. Every
  function `[]`.
- `progtmpl` — `ProgToken`, `ProgTemplate`, `ProgTmplError` and
  `impl Error for ProgTmplError`; `parse` with an offset in every
  refusal; the three default templates including the one
  `novo pkg add` would use; `render_into`, `render`, `render_token`;
  the layout measurements `fixed_width` and `flexible_width`; the
  shared `width_rule` / `width_rule_cjk`; and the three formatters.
- `progstyle` — `ProgStyle`; `style`, `ascii`, `spinner_only` and the
  builders; `bar`, `bar_into`, `filled_parts`; and `spinner_frame`,
  which is a function of the instant rather than of a counter.
- `progdraw` — `ProgMode`, `ProgTerm` and `ProgTick`; the terminal and
  tick constructors; `should_redraw` and `milestone_due` at `[]`;
  `frame_into` and `clear_into` at `[]`; `draw_to`, `draw_if_due`,
  `finish_to` and `clear_to` at `[e]`; and `now()` at `[time]`.
- `progmulti` — `ProgLine` and `ProgMulti`; `of`, `add`, `add_with`,
  `update`, `line`, `line_count`, `retire_finished`, `all_done`,
  `height`; `move_up_into` and `frame_all_into` at `[]`; and
  `draw_all_to`, `draw_all_if_due`, `finish_all_to`, `clear_all_to` at
  `[e]`.

### Known

- **The instant is an argument.** Nothing in `progstate`, `progtmpl`,
  `progstyle` or the `*_into` functions reads a clock, so "after 30
  seconds and 300 of 1000 items the estimate is 70 seconds" is an
  assertion rather than a wait.
- **The writer is an argument, and effect-polymorphic**
  (`fn draw_to<W: Write[e]>(…) [e]`), so a draw costs what the caller's
  writer costs — `[io]` for stderr, nothing for a `Buffer`.
- **The terminal width is an argument.** This package asks the terminal
  nothing; a library that queried it would be `[io]` everywhere,
  untestable, and wrong inside a pipe.
- **`progdraw.now()` is the only `[time]` function**, and the only
  reason the layer is `host` rather than `core`. Not split, because a
  bar with no way to ask the time is one every caller would have to
  complete themselves.
- **`now()` is wall time, not monotonic**, because `std.time` publishes
  no monotonic clock; `elapsed` clamps at zero rather than reporting a
  negative.
- **`eta_duration` answers `HtDuration` and not `?HtDuration`.** A
  `@value` struct cannot be an optional payload (E2015), so `eta`
  carries the `None` and `has_eta` is the predicate.
- **A non-TTY writer gets milestones**, and in `ProgPlain`
  `novo pkg add` would write exactly the line it writes today.
- **The width is cells**, through unicode-nv into textwrap-nv's
  `WrapWidth` — the same rule table-nv uses.
- **The multi-bar's height is a value the caller carries.** There is no
  hidden state, so `draw_all_to` answers the new display and the caller
  keeps it.
- **Four `core` dependencies**: humantime-nv, unicode-nv, textwrap-nv,
  ansi-nv. No device claim: this is a terminal package.
