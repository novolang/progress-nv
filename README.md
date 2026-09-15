# progress-nv

A progress bar is a line of terminal output that reports how far a long task has
got, and how long the rest of it will take. This package brings the shape of
[indicatif](https://docs.rs/indicatif) and [tqdm](https://github.com/tqdm/tqdm)
to novo-lang. It differs from both in three ways: the current time, the writer
and the terminal width all arrive as arguments rather than being read inside the
library.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

A **bar** reports work whose total is known: 300 of 1000 files unpacked. A
**spinner** reports work whose total is not known. Here they are one type. A
state with no length renders as a spinner because there is nothing to draw a bar
from.

The state of a bar is a plain value. It carries the position, the length, the
instant it started, the instant it was last updated, the unit it counts in and
the message on its line. Advancing it answers a new value. Nothing in it reads a
clock, writes a byte or knows what a terminal is.

A **template** is the text of the line, written with fields in braces:
`{spinner} {msg} {bar} {pos}/{len} ({percent}%) {eta}`. A template is parsed
once into a list of tokens, and rendered many times from that list. Exactly one
field in a template may expand to fill the width that the others leave.

The **estimate**, usually written ETA, is how long the rest will take. It is the
work remaining divided by the rate. The rate is **smoothed**, because a rate
taken from the last two samples alone produces an estimate that swings by
minutes. Smoothing here is a window of recent samples with a weight on the
newest one, and both numbers are fields a caller can change.

Width is counted in **cells**, the columns a character occupies on a terminal.
This is not the byte count and not the character count.

| Text | Bytes | Characters | Cells |
| --- | --- | --- | --- |
| `abc` | 3 | 3 | 3 |
| `世界` | 6 | 2 | 4 |

A **mode** says what the destination can take. A terminal is redrawn in place. A
file, a pipe or a continuous-integration log gets one line each time the bar
passes a milestone, and no escape sequences at all. A silent destination gets
nothing.

## Install

```
novo pkg add progress-nv
```

## Example

```novo
use std.list
use progstate
use progtmpl
use progdraw

fn main() [io, time]
    let items = ["alpha", "beta", "gamma"]

    // Stderr, by convention, so the bar does not land in a piped result.
    let err = FdStream.new(2)

    // The destination: a terminal 100 cells wide, redrawn in place.
    let term = progdraw.term_of(100)

    // The line's shape, parsed once. The loop below renders it many times.
    let tmpl = progtmpl.default_template()

    // Redraw at most twenty times a second, and always on a phase change.
    let policy = progdraw.tick()

    // A bar over a known count of items, started at the caller's instant.
    var st = progstate.start(list.len(items), progdraw.now())

    // The state the screen is showing, which the throttle compares against.
    var shown = st

    for name in items
        // One item done, with the item's name as the line's message.
        st = progstate.advance(progstate.with_message(st, name), 1, progdraw.now())

        // Draw only if the policy says a frame is due, and say whether it drew.
        match progdraw.draw_if_due(err, term, policy, tmpl, shown, st, progdraw.now())
            Err(e)   => println(e.message())
            Ok(drew) =>
                if drew
                    shown = st

    // The last line and a newline, so the next output starts on a clean line.
    let done = progstate.finish(st, progdraw.now())
    match progdraw.finish_to(err, term, tmpl, done, progdraw.now())
        Some(e) => println(e.message())
        None    => println("unpacked ${list.len(items)} items")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `progstate` | The bar as a value: the position, the length, the unit, the samples, and the measurements taken from them, including the rate and the estimate. |
| `progtmpl` | The line as a template: parsing one, measuring how wide each field will be, and rendering the whole line into a buffer. |
| `progstyle` | The characters a bar and a spinner are drawn with, and the spinner frame for a given instant. |
| `progdraw` | The destination: the mode, the width, the redraw policy, the functions that hand bytes to a writer, and the one function that reads the clock. |
| `progmulti` | Several bars drawn together, and the cursor movement that redraws them in place. |

## How to choose an entry point

**A loop with one bar calls `progstate` and `progdraw`.** Start a state, advance
it once per item, and call `progdraw.draw_if_due` after each advance. Finish
with `progdraw.finish_to`.

**A program running several tasks calls `progmulti`.** Add one line per task,
update a line by its index, and call `progmulti.draw_all_to`. Keep the display
it answers: the height of the last frame is carried on that value, and the next
frame's cursor movement is computed from it.

**A program that owns the screen itself calls the `*_into` functions.**
`progtmpl.render_into`, `progstyle.bar_into`, `progdraw.frame_into`,
`progmulti.frame_all_into` and `progmulti.move_up_into` all append into a buffer
the caller supplies and perform no input or output. A terminal user interface
takes those bytes and places them where it wants.

**A test calls the same drawing functions with a buffer.** Those functions take
the writer as an argument and cost exactly what that writer costs. Against a
`Buffer` they cost nothing, so a test asserts the exact bytes of a frame with no
terminal involved.

## The rules a user needs

1. **The current time is an argument.** Every measurement and every render takes
   the instant as a count of seconds. `progdraw.now()` is the only function in
   the package that reads a clock, and a caller that already has an instant never
   calls it.
2. **`progdraw.now()` is wall time, not monotonic**, because `std.time`
   publishes no monotonic clock. A clock that steps backwards would make an
   elapsed time negative. `progstate.elapsed` clamps at zero instead.
3. **The terminal width is an argument.** This package asks the terminal
   nothing. Read the real width once at the top of the program and pass it down.
   A program writing to a file passes the width it wants its lines to be.
4. **Width is counted in cells.** The rule is unicode-nv's character width,
   handed to textwrap-nv as a width rule, and it is the same rule table-nv uses.
   `progtmpl.width_rule_cjk` is for a terminal that draws East Asian Ambiguous
   characters two cells wide. Unicode Standard Annex #11, East Asian Width.
5. **A rendered line must not be wider than the width it was given.** A line one
   cell too long wraps, and a wrapped line makes a multi-bar's cursor movement
   wrong from then on. `progtmpl.fixed_width` and `progtmpl.flexible_width`
   answer how the width is spent.
6. **Exactly one field in a template may expand.** `{bar}` and `{msg}` take what
   is left after every other field is measured. A template with two expanding
   fields is refused with `ProgTooManyFlexible`, naming both.
7. **A template that does not parse is refused at parse time, with the byte
   offset.** A field name this package does not know is `ProgUnknownField`, and
   `progtmpl.field_names` lists the names it does know. indicatif drops a
   misspelled field silently.
8. **Parse a template once.** `progtmpl.parse` answers a list of tokens, and
   `progtmpl.render_into` walks it. A bar redrawn twenty times a second renders
   twelve hundred times off one parse.
9. **`progstate.eta` answers nothing in three cases**: the length is unknown,
   the rate is zero, and nothing has finished yet. A renderer draws `--` for
   those.
10. **`progstate.eta_duration` answers zero rather than nothing when there is no
    estimate.** Its return type cannot be optional. humantime-nv's duration is a
    `@value` struct, and a `@value` struct cannot be an optional payload
    ([E2015](https://novo-lang.org/docs/errors/E2xxx.html)). Ask
    `progstate.has_eta` first.
11. **A position may exceed its length.** A length is usually an estimate, so
    nothing clamps the position. `progstate.fraction` clamps its own answer at
    1.0.
12. **A state with no length is a spinner.** `progstate.start_spinner` makes
    one, and `progstate.has_length` is the question. A spinner's fraction,
    remaining count and estimate all have no answer.
13. **`progstate.rate` is zero until there are two samples.** A rate from one
    sample is where a bar's first absurd estimate comes from.
14. **A destination that is not a terminal gets milestones.** In `ProgPlain` one
    line is written each time the bar crosses a multiple of the milestone
    fraction, and nothing in between. A redraw loop into a log file otherwise
    writes a hundred thousand lines of carriage returns.
15. **The mode is a value you set.** This package cannot ask whether its writer
    is a terminal. `ProgSilent` is the third mode, so a quiet flag needs no
    branch at each call site.
16. **`progmulti.move_up_into(out, 0)` writes nothing.** A cursor-up sequence
    with a zero parameter moves one line on most terminals. ECMA-48, CUU.
17. **`progmulti.draw_all_to` answers the display, and the caller keeps it.**
    The height of the last frame lives on that value, and the next frame's
    cursor movement is computed from it.
18. **A style with no partial characters moves one whole cell at a time.**
    `progstyle.filled_parts` says how many distinct positions a bar of a given
    width can show. An ASCII bar twenty cells wide over ten thousand items
    redraws identically for five hundred of them.

## What is not included

- **Asking the terminal for its width.** A library that queried the terminal
  would make every render perform input and output, would be untestable, and
  would get the wrong answer inside a pipe.
- **Detecting whether the writer is a terminal.** The mode is a value the caller
  sets, for the same reason the width is.
- **A clock inside the bar.** `progdraw.now()` exists, and nothing else in the
  package calls it.
- **A monotonic clock.** `std.time` publishes none, so elapsed time is measured
  against wall time.
- **A background thread that redraws on its own.** This package holds no hidden
  state. Every value a draw changes is answered back to the caller.
- **Colour.** A style holds characters, not attributes. Colour sequences are
  [ansi-nv](https://novo-lang.org/packages/ansi-nv)'s, and a coloured message
  still measures correctly because an escape sequence costs no cells.
- **Fields beyond the ones `progtmpl.field_names` lists.** A caller that wants
  another arrangement renders its own line and calls `progstyle.bar_into` for
  the bar itself.
- **Running on a microcontroller.** No module claims it. This is a package about
  terminal output.

## Related packages

- [humantime-nv](https://novo-lang.org/packages/humantime-nv) formats the
  estimate and the elapsed time in words. A bar that formatted its own durations
  would disagree with every other tool in the ecosystem.
- [unicode-nv](https://novo-lang.org/packages/unicode-nv) supplies the character
  width, and [textwrap-nv](https://novo-lang.org/packages/textwrap-nv) supplies
  the width rule and the truncation of a message that does not fit.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) supplies the cursor movement
  and the erase, written into a caller's buffer, and the state machine that
  measures a coloured string.
- [table-nv](https://novo-lang.org/packages/table-nv) lays out columns on the
  same terminal under the same width rule, so a bar and a table cannot come to
  different answers about how wide a string is.
- `std.tty` in the standard library reports whether the output is a terminal and
  how wide it is. Call it once and pass the answers in.
- `std.cli` in the standard library parses command-line arguments. A quiet flag
  read from it selects `ProgSilent`.

## Tests

```bash
novo test tests                          # every suite
novo test tests/progstate_tests.nv       # the measurements and the estimate
novo test tests/progtmpl_tests.nv        # parsing a template, and the cell width
novo test tests/progstyle_tests.nv       # the glyphs and the spinner frame
novo test tests/progdraw_tests.nv        # the throttle and the three modes
novo test tests/progmulti_tests.nv       # the cursor movement
```

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
progress-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests
are the specification the implementation will have to satisfy.

The expected numbers are indicatif's and tqdm's own defaults: the smoothing
window, the weight on the newest sample, the redraw interval, the arrangement of
the default template, and the spinner frames. The cell widths come from the
Unicode character database through unicode-nv.

Because the instant is an argument, the suite asserts the arithmetic without
waiting for it. "After one second and 100 of 1000 items, nine seconds remain" is
one assertion over two integers and two floats. Because the writer is an
argument, every draw in the suite goes into a buffer, and the suite asserts the
exact bytes a frame contains with no terminal. Two assertions are worth naming
on their own. A rendered line is exactly as many cells as it was given. Moving
the cursor up zero lines writes no bytes at all.

## Implementation status

| Item | Implemented |
| --- | --- |
| `progstate.ProgUnit`, `.ProgPhase`, `.ProgSmoothing`, `.ProgSample`, `.ProgState` | declared |
| `progstate.smoothing`, `.uniform_smoothing`, `.start`, `.start_spinner` | no |
| `progstate.with_unit`, `.with_smoothing`, `.with_message`, `.with_length` | no |
| `progstate.advance`, `.set_position`, `.finish`, `.abandon` | no |
| `progstate.has_length`, `.is_running`, `.fraction`, `.remaining`, `.elapsed` | no |
| `progstate.rate`, `.mean_rate`, `.eta`, `.eta_at`, `.has_eta`, `.last_sample` | no |
| `progstate.eta_duration`, `.elapsed_duration` | no |
| `progtmpl.ProgToken`, `.ProgTemplate`, `.ProgTmplError`, `impl Error for ProgTmplError` | declared |
| `progtmpl.parse`, `.default_template`, `.field_names` | no |
| `progtmpl.default_template_text`, `.default_spinner_text`, `.download_template_text` | no |
| `progtmpl.render_into`, `.render`, `.render_token` | no |
| `progtmpl.width_rule`, `.width_rule_cjk`, `.fixed_width`, `.flexible_width` | no |
| `progtmpl.format_count`, `.format_rate`, `.format_seconds` | no |
| `progstyle.ProgStyle` | declared |
| `progstyle.style`, `.ascii`, `.spinner_only` | no |
| `progstyle.with_chars`, `.with_partials`, `.with_ends`, `.with_spinner` | no |
| `progstyle.bar`, `.bar_into`, `.filled_parts`, `.is_ascii` | no |
| `progstyle.spinner_frame`, `.finished_frame`, `.abandoned_frame` | no |
| `progdraw.ProgMode`, `.ProgTerm`, `.ProgTick` | declared |
| `progdraw.term`, `.term_of`, `.plain`, `.silent`, `.tick`, `.tick_at` | no |
| `progdraw.with_style`, `.with_width_rule`, `.with_milestone` | no |
| `progdraw.should_redraw`, `.milestone_due`, `.frame_into`, `.clear_into` | no |
| `progdraw.draw_to`, `.draw_if_due`, `.finish_to`, `.clear_to` | no |
| `progdraw.now` | no |
| `progmulti.ProgLine`, `.ProgMulti` | declared |
| `progmulti.multi`, `.of`, `.add`, `.add_with`, `.update`, `.line`, `.line_count` | no |
| `progmulti.retire_finished`, `.all_done`, `.height` | no |
| `progmulti.move_up_into`, `.frame_all_into` | no |
| `progmulti.draw_all_to`, `.draw_all_if_due`, `.finish_all_to`, `.clear_all_to` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
