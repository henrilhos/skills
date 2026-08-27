---
name: babysit-pr
description: "One pass over an open PR: syncs the branch with the base when CI is red and the branch is behind, resolves conflicts, then cycles the ready-to-test label"
user-invocable: true
argument-hint: "[pr-number|url] [--base <branch>] (defaults: the current branch's PR, main)"
---

# Babysit an open PR through CI

One **tick**: read the PR's state, take at most one action, report, stop. Recurrence belongs to `/loop`, not here.

## Usage

```
/loop 10m /babysit-pr            # the current branch's PR, base main
/loop 10m /babysit-pr 4359
/babysit-pr --base develop       # a single tick by hand
```

From `$ARGUMENTS`: the PR is the number or URL when given, otherwise the current branch's PR; the base is `--base <branch>` when given, otherwise `main`.

## 1. Read the state

Three calls, same message:

```fish
gh pr view <pr> --json number,url,headRefName,labels
gh pr checks <pr> --json name,bucket --jq '.[] | "\(.bucket)\t\(.name)"' | sort
git fetch origin <base>; and git rev-list --count HEAD..origin/<base>
```

Three facts come out of that, and they are the only ones the gate reads:

- **red** — any check in bucket `fail`. The buckets are `pass`, `fail`, `pending`, `skipping`, `cancel`.
- **behind** — `rev-list --count` above zero: commits on the base the branch doesn't have.
- **labelled** — `ready-to-test` present in `labels`. Step 4 needs this value from _before_ the push, so read it now.

## 2. Gate

| State                       | Action                                                                                            |
| --------------------------- | ------------------------------------------------------------------------------------------------- |
| Any check `pending`         | The run isn't over yet. Report "checks still running" and stop.                                    |
| Green — no `fail` bucket    | Report green and stop.                                                                            |
| **red** and **behind**      | Sync — step 3.                                                                                    |
| **red**, branch up to date  | The branch's own code broke it, and a sync fixes nothing. Report the failing check names and stop. |

## 3. Sync

Guard first — a tick may only sync the branch it is standing on:

```fish
git rev-parse --abbrev-ref HEAD    # must equal the PR's headRefName
```

A mismatch ends the tick: report it and stop. Syncing the wrong branch costs more than one red tick.

Then run `/update-branch <base>`. It owns the merge, the conflict resolution, the hidden-semantic-conflict check, the validation and the push — the whole sync, conflicts included. Reach for it instead of writing any of that here.

`/update-branch` stops and asks when two intents are genuinely incompatible, and when validation fails. Either way the branch was never pushed: report what it said and end the tick with the label untouched.

## 4. Cycle `ready-to-test`

The push in step 3 makes the repo's automation strip `ready-to-test`; putting it back is what re-arms the test pipeline. One push, one cycle — the clean path and the conflict path both land here exactly once.

**Not labelled before the push** → there is no strip to wait for. Add it and go to step 5:

```fish
gh pr edit <pr> --add-label ready-to-test
```

**Labelled before the push** → wait for the strip to land first. Adding while the label is still there is a no-op, and adding before the strip lets the strip eat the new label — the pipeline never fires.

Foreground `sleep` is blocked in this environment, so wait with a backgrounded loop (`Bash`, `run_in_background: true`) that exits once the label is gone, and take its completion notification as the signal:

```fish
for i in (seq 40)
    test (gh pr view <pr> --json labels --jq '[.labels[].name] | index("ready-to-test")') = null; and break
    sleep 15
end
```

A transient `gh` failure yields an empty string, which fails the `= null` test and keeps the loop waiting — the safe direction.

Ten minutes (40 × 15s) with the label still on means the automation never stripped it. Report that and stop: re-adding a label that never left fires nothing.

Otherwise add it back with the `gh pr edit` above, then confirm it is on the PR.

## 5. Report

One short paragraph: which gate branch the tick took and on which of the three facts, what `/update-branch` merged or resolved, and the label's final state. A tick that did nothing says so in a single line — it is the common case, and `/loop` will print it often.

## Guardrails

- **One action per tick.** Once the label is back on, stop. Leave the fresh pipeline alone; the next tick reads it.
- **Green and pending are no-op ticks.** Touch nothing — no sync, no label, no comment.

## Environment

- `gh` authenticated for the repo.
- Run from the checkout with the PR's branch checked out.
- The repo has a `ready-to-test` label, and automation that strips it when the branch is pushed.
