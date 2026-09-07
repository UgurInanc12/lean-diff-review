# lean-diff-review

A Hermes Agent skill that reviews one diff for three things: code that is broken,
code that is missing, and code that should never have been written.

One line per finding, then it applies the safe fixes and hands the risky ones back.

```
<file>:L<line>: <tag> <what>. <fix>. | SAFE | CAREFUL | RISKY
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

Correctness pass runs before the bloat pass, always. A shorter diff that is still
wrong is worse than the one you started with.

Never cut, however removable they look: trust-boundary validation, error handling
that prevents data loss, security, accessibility, and the one check covering the
logic.

## Install

```
git clone https://github.com/UgurInanc12/lean-diff-review
cp -r lean-diff-review ~/.hermes/skills/software-development/lean-diff-review
```

Restart the agent, then say "review this diff" or "lean review".

## Credits

The cut-the-bloat tags come from [ponytail](https://github.com/DietrichGebert/ponytail)
(MIT). Taken as an on-demand skill rather than its always-on plugin, with
`broken:` and `missing:` added because a review that only deletes will happily
ship a bug.

MIT
