---
name: run-bead
description: Implement one Beads (`bd`) issue end to end - an implementer agent writes it, a reviewer agent checks it against the acceptance criteria, a reviser agent fixes findings (max 3 rounds) - then commit it on its own jj bookmark or git branch for the owner to review. Never pushes or closes the bead. Use when the user runs "/beads-workflow:run-bead <bead-id>", asks to run, implement, or process a bead by ID, or approves implementing a specific bead ID that was just discussed (e.g. "yes, implement bd-12", or "go ahead" right after bd-12 was proposed) - never implement a bead inline. Use run-epic instead when the ID is an epic.
---

# Run bead

Input: one bead ID. You are a thin coordinator. Agents do the work; you pass
bead IDs and read only their short returns and `VERDICT:` lines. Never read a diff
yourself. Never push.

This file owns the per-bead steps. `run-epic` follows them for each child, so a
change here changes both.

## Version control

Works in jj and git repos. Detect once per run: if `jj root` succeeds, use jj,
even when a `.git` sits next to it (colocated repo); otherwise use git. Pass
`vcs: jj` or `vcs: git` to every agent, so the check lives only here.

The steps below name a row of this table, never a raw command:

| Operation | jj | git |
|---|---|---|
| status | `jj status` | `git status --porcelain` |
| log | `jj log` | `git log --oneline` |
| commit | `jj describe -m "<msg>"` | `git switch -c <branch>`, `git add -A`, `git commit -m "<msg>"` |
| branch | `jj bookmark create <branch>` | nothing: `commit` already made it |
| next | `jj new` | nothing: HEAD is already on the new branch |
| change id | `jj log -r @- --no-graph -T 'change_id.short()'` | `git rev-parse --short HEAD` |
| parent | `<rev>-` | `<rev>~1` |
| checkout | `jj new <rev>` | `git switch --detach <rev>` |
| return | `jj new <top branch>` | `git switch <top branch>` |
| bisect | `jj bisect run --range '<good>..<bad>' -- bash -c '<command>'` | `git bisect start <bad> <good>`, `git bisect run bash -c '<command>'`, `git bisect reset` |

In git, each bead's branch starts from the previous bead's branch, so the
branches form the same stack as jj bookmarks. If `git switch -c` fails because
the branch exists, pick another name; never reset or overwrite a branch.

## Before you start (single bead only)

`run-epic` already picks beads that pass these checks; when run on its own, check
them and stop with a one-line reason if any fails:

- The bead is not closed, and has neither the `needs-review` nor the `human` label.
- Every `blocks` dependency is closed or has `needs-review`. A `needs-review`
  blocker counts as done because its commit is already in the stack; make sure
  the working copy (jj `@`, git `HEAD`) sits on top of that commit (`log`) so
  this bead stacks on it.

## Steps

1. **Pre-flight.** `bd show <id> --json | jq '.[0] | {title, acceptance_criteria, design}'`.
   No acceptance criteria, or the bead is clearly more than one small change:
   write the question in the notes, `bd label add <id> human`, stop this bead.
2. **Implement.** Run the `beads-workflow:implementer` agent with the bead ID, `vcs`, and the gate command
   for the touched app (see "Gate commands" below). `BLOCKED: human` -> stop this bead.
3. **Gate.** Run the gate yourself, each command as a plain line (never through a
   shell variable: shells such as fish do not split it into words) and check
   the exit code. Exit 127 means
   the command was not found, not a failed gate: fix the command. If the gate
   fails, write `GATE FAILED: <command> exited <code>` plus the error into the bead
   notes, run the `beads-workflow:reviser`, and skip the review this round. Record the result with
   `bd update <id> --set-metadata gate=pass|fail`.
4. **Review loop, max 3 rounds.** Before each review run
   `bd update <id> --unset-metadata verdict` so an old verdict cannot be read as new.
   Run the `beads-workflow:reviewer` (bead ID + round number + `vcs`), then read the verdict from the bead:
   `bd show <id> --json | jq -r '.[0].metadata.verdict // empty'`. If it is empty, use
   the `VERDICT:` line of the reviewer's return.
   - `CLEAN` -> step 5.
   - `ISSUES` -> run the `beads-workflow:reviser` (bead ID + `vcs` + gate command), run the gate, review again.
   - Still ISSUES after round 3, or no verdict anywhere: flag `human`, stop this bead.
5. **Commit.** First run `status`: every changed file must belong to this bead.
   Anything else (another session, a leftover agent) -> stop and ask the owner.
   Then run `commit` with a Conventional Commit message (`type(scope): summary`)
   plus the trailers from the session's attribution rules, using the branch name
   `<type-short-summary>`. Then `branch`, then `next`, so the next bead stacks
   on top. `bd label add <id> needs-review`. Record
   `bd update <id> --set-metadata bookmark=<branch> --set-metadata change=<change id> --set-metadata rounds=<n>`
   (the key is `bookmark` in git repos too, so older beads keep working).
   Leave the bead open.

An agent that stops at its turn limit returns partial output with an agent ID:
resume it once; if it still does not finish, flag `human`.

## When the owner answers a `human` bead

Never use `bd human respond`: it closes the bead. Instead write the answer with
`bd update <id> --notes "..."`, remove the flag with `bd label remove <id> human`,
and run the bead again.

## Gate commands

This skill is shared across projects, so the commands live with each project:
read the `## Quality gates` section of the project's root `CLAUDE.md` and pick
the row for the area the bead touches. No such section, or no row for that
area: ask the owner for the gate before step 2, and suggest adding the section.

## Final report (single bead only)

Short. Do not restate findings. Print one of:
- ready for review: the branch, the command to view it (`jj show <branch>` or
  `git show <branch>`; `hunk show <branch>` if `hunk` is installed), and rounds
  used (`metadata.rounds`)
- needs the owner's decision: the bead ID and the question from its notes
- not started: the failed check from "Before you start"

## Rules

- Never push, never close a bead (including via `bd human respond`): the owner approves first.
- One branch per bead, stacked in order.
- Stop every agent (`TaskStop <agent-id>`) as soon as you have read its return,
  before the next step. An agent can report DONE while its own background
  commands still run, and one that keeps going can edit files after the bead
  is committed, landing in the next bead's change.
- Keep your own context small: no diffs, no file reads, no copying agent output.
