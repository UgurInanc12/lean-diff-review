# lean-diff-review

A Hermes Agent skill that reviews one diff for three defect classes: code that is
broken, code that is missing, and code that should never have been written.

It reports one line per finding with a `file:line`, a concrete fix and a risk
tier, applies the safe ones, and hands the risky ones back to you.

## Why it exists

[Ponytail](https://github.com/DietrichGebert/ponytail) makes an agent stop
over-building. Its review skill is the sharpest part of it: tagged one-line
findings, `net: -N lines possible.`, no hedging prose. But it ships as an
always-on plugin that injects a ruleset into every turn, and its review scopes
correctness out on purpose.

Two things did not fit here:

1. This agent does more than write code. An always-on ruleset costs tokens on
   every unrelated turn and pulls the model toward terseness where terseness is
   wrong.
2. A review that only deletes will happily ship a bug. The shortest diff that is
   still broken is worse than the one you started with.

So this is the review lens on its own, invoked explicitly, with the correctness
pass put back in front of the bloat pass.

## Findings format

```
<file>:L<line>: <tag> <what>. <fix>. | <risk>
```

| Tag | Meaning |
| --- | --- |
| `broken:` | Does not do what it claims. Name the input that breaks it. |
| `missing:` | Error path, guard or case the change needs. |
| `delete:` | Dead code, unused flexibility, speculative feature. |
| `stdlib:` | Hand-rolled thing the standard library ships. |
| `native:` | Dependency doing what the platform already does. |
| `yagni:` | Abstraction with one implementation, layer with one caller. |
| `shrink:` | Same logic, fewer lines. |

Risk tiers: `SAFE` (apply), `CAREFUL` (apply with a test run), `RISKY` (ask
first). `broken:` and `missing:` are fixed before anything gets cut, even when
that makes the diff longer.

## What it will not cut

Trust-boundary validation, error handling that prevents data loss, security
checks, accessibility, and the one runnable check that covers the logic. Lazy
about the solution, never about the guardrails.

## Install

```bash
git clone https://github.com/UgurInanc12/lean-diff-review
cp -r lean-diff-review ~/.hermes/skills/software-development/lean-diff-review
```

Restart the agent, then invoke it with "review this diff", "lean review", or
"what can we delete".

## Real finding from the first run

Run on an uncommitted 405-line diff in a yt-dlp desktop app. The correctness pass
flagged an audio-track ranking function whose sort key compared filesize before
the DRC penalty:

```
download_options.py:L71: broken: rank() compares filesize before the DRC penalty, so at an
  equal bitrate the larger DRC file wins. Move the DRC term ahead of filesize. | CAREFUL
```

Reproduced with the real byte counts YouTube reports for format 140 (3,449,215)
and 140-drc (3,450,102), both at 129.472 kbps: the function returned `140-drc`,
the dynamic-range-compressed track, as "best audio". Fixed, regression test
added, suite went 110 -> 111 passing.

The existing test suite passed before the fix. The DRC test in it happened to
leave both filesizes `None`, which ties the field the bug hides behind.

## Credits

Tag vocabulary and the "the diff's best outcome is getting shorter" framing are
adapted from [ponytail](https://github.com/DietrichGebert/ponytail) by Dietrich
Gebert (MIT). The `broken:` and `missing:` tags, the two-pass ordering, the risk
tiers and the caller-tracing step are additions.

## License

MIT
