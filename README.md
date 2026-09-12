# progress-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Progress bars and spinners for novo-lang —
[indicatif](https://docs.rs/indicatif)'s and
[tqdm](https://github.com/tqdm/tqdm)'s shape, with the clock, the
writer and the terminal width all moved outside.

- `progstate` — the bar as a value: position, length, samples, rate,
  ETA. Every function `[]`;
- `progtmpl` — the line as a template, parsed once and rendered into a
  buffer;
- `progstyle` — the glyphs, and the spinner as a function of an
  instant;
- `progdraw` — the writer, the redraw policy, the TTY and plain modes;
- `progmulti` — several bars, and the cursor arithmetic.

```
novo pkg add progress-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use std.io
use progstate
use progtmpl
use progdraw

fn download(items: [Str], err: FdStream) -> ?IoError [io, time]
    var st = progstate.start(list.len(items), progdraw.now())
    var shown = st
    var fault: ?IoError = None
    for name in items
        fetch(name)
        st = progstate.advance(progstate.with_message(st, name), 1, progdraw.now())
        match progdraw.draw_if_due(err, progdraw.term_of(100), progdraw.tick(),
                                   progtmpl.default_template(), shown, st,
                                   progdraw.now())
            Err(e)    => fault = Some(e)
            Ok(drew)  => if drew
                             shown = st
    let _ = progdraw.finish_to(err, progdraw.term_of(100),
                               progtmpl.default_template(),
                               progstate.finish(st, progdraw.now()),
                               progdraw.now())
    fault
```

The writer is the caller's — stderr here, by convention — and so is the
width and the clock.

## The load-bearing interface: `ProgState` is a value, rendering is pure

```novo ignore
pub fn advance(st: ProgState, delta: Int, at: Float) -> ProgState []
pub fn eta(st: ProgState, at: Float) -> ?Float []
pub fn render_into(out: [u8], t: ProgTemplate, st: ProgState, style: ProgStyle,
                   columns: Int, w: WrapWidth, at: Float) -> [u8] []
```

**The instant is an argument.** indicatif and tqdm both own the clock,
and the price is paid four times.

*A progress bar cannot be tested.* "After 30 seconds and 300 of 1000
items, the estimate is 70 seconds" is a statement about arithmetic, and
where the clock is inside the bar it can only be checked by waiting.
Here it is one assertion over two integers and a float.

*The rate smoothing is invisible.* Every bar smooths — an instantaneous
rate makes an estimate that jumps by minutes — and the smoothing is
what a reader most wants to see and least often can. `ProgSmoothing` is
a field.

*A bar cannot be driven by anything but wall time.* A download resumed
from a log, a replay of a build, a test of the formatting: each has an
instant, and none of them is `now`.

*The row spreads.* A bar that reads the clock charges `[time]` to every
caller of every function it has.

**The writer is an argument, and effect-polymorphic.**

```novo ignore
pub fn draw_to<W: Write[e]>(w: W, t: ProgTerm, tmpl: ProgTemplate,
                            st: ProgState, at: Float) -> ?IoError [e]
```

The bound binds the trait's effect parameter and the clause uses it, so
a draw costs what the caller's writer costs: `[io]` for stderr, nothing
at all for a `Buffer`. Drawing a bar inside a pure function is therefore
ordinary — which is how this package's own suite asserts what it draws.

**The terminal width is an argument.** This package never asks the
terminal anything. A program that wants the real width reads it once at
the top and passes it down; a program writing to a file passes the width
it wants its lines to be. A library that queried the terminal would make
every render `[io]`, make the output untestable, and get the wrong
answer inside a pipe.

## A non-TTY writer gets milestones

`ProgPlain` writes one line when a threshold is crossed and nothing in
between. A redraw loop into a log file produces a hundred thousand lines
of carriage returns, which is the commonest complaint about every
progress bar ever shipped.

The mode is a value the caller sets, for the same reason the width is:
this package cannot ask, and a caller who knows should not be overruled.
`ProgSilent` is the third mode, so a `--quiet` flag does not grow an
`if` at every call site.

## What `novo pkg add`'s download output would use

Today `novo pkg add` prints one line per package:

```
novo: fetching humantime-nv 0.0.1 from registry.novo-lang.org
```

With this package it is a `progmulti` display with one line per package
being fetched, on `progtmpl.download_template_text()`:

```
{spinner} {msg:24} {bytes}/{total_bytes} {bar} {rate} {eta}
```

— the package name in a fixed column so a list of them aligns, and the
byte counts before the bar so the numbers do not move when the bar does.

And in `ProgPlain` — which is what a continuous-integration log is —
it writes **exactly the line it writes today**, once per package. That
is the point of the mode: adopting this changes nothing about what a CI
log contains.

## The width is counted in cells

`世界` is six bytes, two codepoints and four cells, and a bar sized by
either of the first two does not line up. `unicode-nv`'s `char_width`,
handed to `textwrap-nv` as its `WrapWidth`, is the rule — the same
arrangement table-nv uses, so a bar and a table on the same terminal
cannot come to different answers, and no signature in either package
changed to make it so.

An ANSI escape in a message costs no width, and that is ansi-nv's state
machine rather than a regular expression.

## The layer, and the one function that decides it

| what | row |
| --- | --- |
| every measurement, every template, every render, every byte into a caller's buffer | `[]` |
| `draw_to`, `draw_if_due`, `finish_to`, `clear_to`, `draw_all_to` | `[e]` — the writer's |
| `progdraw.now()` | `[time]` |

**One function is the whole reason this package is `host`.** The
rendering half would be a `core` package on its own. It is not split,
because a progress bar with no way to ask the time is a library nobody
can use without writing that function themselves, and every one of them
would write it differently.

`now()` is wall time and not monotonic, because `std.time` publishes no
monotonic clock. A clock that steps backwards makes an elapsed time
negative, and `progstate.elapsed` clamps at zero rather than reporting
one. That is a real limitation, stated rather than hidden.

## One signature the language changed

`eta_duration` answers a `HtDuration` and not a `?HtDuration`.
humantime-nv's duration is a `@value` struct, and a `@value` struct
cannot be a `?T` payload ([E2015](https://novo-lang.org/docs/errors/E2xxx.html)):
it is unboxed and an optional position has no unboxed lowering. So
`eta` is the function that answers `None`, `has_eta` is the predicate,
and `eta_duration` is the formatting convenience beside them.

## Four dependencies, all `core`

**humantime-nv** for the ETA and the elapsed time in words — a bar that
formatted its own durations would disagree with every other tool in the
ecosystem after about a week. **unicode-nv** and **textwrap-nv** for the
cell width, shared with table-nv. **ansi-nv** for the cursor movement
and the erase, written into a caller's buffer, which is exactly the
shape the draw functions need.

## Reference implementation

[indicatif](https://docs.rs/indicatif) (Rust) for the template grammar,
the multi-bar and the default style;
[tqdm](https://github.com/tqdm/tqdm) (Python) for the rate smoothing and
the non-TTY behaviour. The departures are the instant as an argument,
the writer as an argument, and the width as an argument.

## Status

Interface only. Every body is `todo()`; `novo pkg build` type-checks and
effect-checks the whole surface, and `novo test tests` is red until the
bodies land.
