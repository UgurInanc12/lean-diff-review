---
name: lean-diff-review
description: "Review a diff: cut bloat, fix broken and missing parts."
version: 1.0.0
author: Ugur Inanc (UgurInanc12), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [code-review, diff, cleanup, yagni, correctness]
    related_skills: [simplify-code, requesting-code-review]
---

# Lean Diff Review

Single-pass review of one diff for three defect classes: code that is broken,
code that is missing, and code that should never have been written. It reports
one line per finding, applies the safe fixes, and hands the risky ones back to
the user.

It runs only when invoked. Nothing is injected into other turns, no always-on
ruleset, no plugin, no background cost.

The over-engineering lens (the `delete/stdlib/native/yagni/shrink` tags and the
"the diff's best outcome is getting shorter" framing) is adapted from
[ponytail](https://github.com/DietrichGebert/ponytail) (MIT). The `broken:` and
`missing:` tags are the local addition: ponytail's review scopes correctness out
on purpose, and a review that only deletes will happily ship a bug.

## When to Use

- "review this diff", "check the code you wrote", "lean review", "what can we delete"
- Before a commit or PR, on a change you or a subagent just produced
- After an agent-generated feature, where over-building is the default failure mode

Don't use for:

- Whole-repo audits of code nobody is changing right now (`inherited-codebase-audit`)
- The pre-commit security gate with an independent reviewer (`requesting-code-review`)
- Big multi-file cleanups worth four parallel reviewers (`simplify-code`)

This skill is the cheap one: one context, no subagents, minutes not tokens.

## Step 1: Collect the diff

Use `terminal` and take the first source that is non-empty:

```bash
git diff                 # uncommitted, tracked (default)
git diff HEAD            # + staged
git diff --staged        # "staged changes"
git diff HEAD~1          # "the last commit"
git diff main...HEAD     # "this branch" / "my PR"
git diff -- path/to/file # explicit scope
```

No repo and no changes: review the files the user named or that were written in
this session. Nothing at all: say so and stop.

Over ~1500 changed lines: review per directory or per commit instead of in one
pass, and say which slice you reviewed.

## Step 2: Read the context, not just the hunks

For every touched file, `read_file` around the hunks and `search_files` for the
callers of every function the diff changes. A hunk hides its callers, and half
the real findings (a sibling caller with the same bug, a helper that already
exists two files over) are invisible from the diff text alone.

Completion criterion: for each changed function you can name its callers, or
state that it has none.

## Step 3: Two passes, in this order

**Pass 1, correctness.** Does the code do what the change claims, on real input?
Wrong conditions, off-by-one, unhandled error path, missing guard at a trust
boundary, a fix applied to one caller while a sibling keeps the flaw.

**Pass 2, bloat.** Only after pass 1. Everything the change did not need.

Never run pass 2 first. A shorter diff that is still wrong is worse than the one
you started with.

## Finding format

`<file>:L<line>: <tag> <what>. <fix>. | <risk>`

Tags, reported worst first:

- `broken:` does not do what it claims. Name the input that breaks it.
- `missing:` error path, guard, or case the change needs. Name what is unhandled.
- `delete:` dead code, unused flexibility, speculative feature. Replacement: nothing.
- `stdlib:` hand-rolled thing the standard library ships. Name the function.
- `native:` dependency or code doing what the platform already does. Name the feature.
- `yagni:` abstraction with one implementation, config nobody sets, layer with one caller.
- `shrink:` same logic, fewer lines. Show the shorter form.

Risk tiers drive step 4:

- `SAFE` provably no behavior change (unused import, dead branch, pass-through wrapper).
- `CAREFUL` same semantics, different shape (stdlib swap, flatten, inline a one-use layer).
- `RISKY` may change behavior or breaks a contract (public API, signature, error semantics).

Examples:

```
api.py:L88: broken: retry loop swallows the last exception, a permanent 500 returns None. Re-raise after the final attempt. | CAREFUL
api.py:L34: missing: no timeout on the outbound call, a hung peer hangs the worker. timeout=10 on requests.get. | CAREFUL
validate.py:L12-38: stdlib: 27-line email validator class. "@" in value, 1 line, the confirmation mail is the real validation. | CAREFUL
repo.py:L5: yagni: AbstractRepository with one implementation. Inline it until a second one exists. | RISKY
util.js:L4: native: moment.js for one format call. Intl.DateTimeFormat, 0 deps. | CAREFUL
```

A finding with no `file:line` is noise. Drop it.

## Step 4: Apply

1. `broken:` and `missing:` first, always. Fix them even when that makes the diff longer.
2. `SAFE` bloat findings: apply.
3. `CAREFUL`: apply one file at a time, run the tests for that file after each.
4. `RISKY`: do not apply. List them and ask.

Never on the chopping block, no matter how removable they look: validation at a
trust boundary, error handling that prevents data loss, security checks,
accessibility, and the single smoke test or `assert` that covers the logic. That
is the difference between lazy and negligent.

Revert any fix whose test goes red, and report it instead of chasing it.

## Step 5: Verify and report

Run the project's tests for the touched files (not the whole suite) plus its
linter or type checker if one is configured. If the changed logic is non-trivial
and has no check at all, leave exactly one runnable check behind: the smallest
thing that fails if the logic breaks. No frameworks, no fixtures.

Close with the scoreboard:

```
net: -N lines | broken fixed: X | missing added: Y | risky, needs your call: Z
```

Nothing wrong and nothing to cut: `Lean already. Ship.` and stop.

## Pitfalls

- **Reviewing the diff without the callers.** The most expensive finding class,
  a sibling caller with the same bug, is only visible from the wider file.
- **Claiming `broken:` without running it.** A suspicious line is a hypothesis.
  Reproduce it with the smallest possible snippet and realistic input values
  before reporting it. Sort keys and comparison chains are the usual offenders:
  a wrong tuple order only misbehaves when the earlier field ties, so construct
  the tie deliberately instead of reading the code and guessing.
- **Deleting a guard because the happy path does not need it.** Trust-boundary
  validation, error handling, security and accessibility are out of scope for
  cutting. If it looks unnecessary, it is a `RISKY` finding, not a `SAFE` one.
- **Deleting the only test.** One assert-based self-check is the minimum, not bloat.
- **Chesterton's fence.** Before removing something odd, `git blame` the line.
  If you cannot explain why it exists, it is `RISKY`, not `delete:`.
- **Style churn.** Renames, import ordering and formatting are not findings.
- **Rewriting beyond the diff.** Scope is the change plus the minimum surrounding
  edit a fix requires. A deeper fix that is worth doing is a follow-up task, and
  you say so instead of starting it.
- **"Probably a helper exists for this."** Search and cite it, or drop the finding.

## Verification checklist

- [ ] Diff source stated (which git command, which scope)
- [ ] Callers identified for every changed function
- [ ] Correctness pass ran before the bloat pass
- [ ] Every finding has `file:line`, a concrete fix, and a risk tier
- [ ] All `broken:` and `missing:` findings fixed or explicitly deferred by the user
- [ ] No `RISKY` finding applied without approval
- [ ] Tests or a linter ran on the touched files after the edits
- [ ] Scoreboard line reported
