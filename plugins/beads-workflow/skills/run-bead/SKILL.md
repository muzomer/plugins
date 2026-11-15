---
name: run-bead
# The coordinator only routes work, so Sonnet is enough; each agent keeps its own model.
model: sonnet
description: Run one Beads (`bd`) issue through test-writer, implementer, reviewer and reviser agents (max 3 review rounds), then commit it on its own jj bookmark or git branch for the owner to review. Use when the user runs "/beads-workflow:run-bead <id>", asks to run or implement a bead by ID, or approves implementing a bead just discussed (e.g. "go ahead") - never implement a bead inline. Use run-epic for an epic.
---

# Run bead

Input: one bead ID. You are a thin coordinator: agents do the work, you pass
bead IDs and read their short returns (see "Rules").

## Version control

Detect once per run: if `jj root` succeeds, use jj, even when a `.git` sits
next to it (colocated repo); otherwise use git. Pass `vcs: jj` or `vcs: git` to
every agent, so the check lives only here. The steps name a row of this table,
never a raw command:

| Operation | jj | git |
|---|---|---|
| status | `jj status` | `git status --porcelain` |
| log | `jj log` | `git log --oneline` |
| commit | `jj describe -m "<msg>"` | `git switch -c <branch>`, `git add -A`, `git commit -m "<msg>"` |
| branch | `jj bookmark create <branch>` | nothing: `commit` already made it |
| next | `jj new` | nothing: HEAD is already on the new branch |
| change id | `jj log -r @- --no-graph -T 'change_id.short()'` | `git rev-parse --short HEAD` |
| parent | `<rev>-` | `<rev>~1` |
| checkout | `jj new <rev>` | `git switch --detach <rev>`; to go back to a branch tip, drop `--detach` |
| bisect | `jj bisect run --range '<good>..<bad>' -- bash -c '<command>'` | `git bisect start <bad> <good>`, `git bisect run bash -c '<command>'`, `git bisect reset` |
| park | `jj describe -m "wip(<id>): parked"`, note the ref `jj log -r @ --no-graph -T 'change_id.short()'`, then `jj new @-` | `git stash push -u -m "wip <id>"`; the ref is `wip <id>` |
| unpark | `jj squash --from <ref> --into @` | `git stash list` to find the entry `wip <id>`, then `git stash pop stash@{<n>}` |

In git, each bead's branch starts from the previous bead's branch, so the
branches stack like jj bookmarks. If `git switch -c` fails because the branch
exists, pick another name; never reset or overwrite a branch.

## Before you start (single bead only)

Stop with a one-line reason if any check fails:

- The bead is not closed, and has neither the `needs-review` nor the `human` label.
- Every `blocks` dependency is closed or has `needs-review`. A `needs-review`
  blocker counts as done because its commit is already in the stack: check
  with `log` that the working copy (jj `@`, git `HEAD`) sits on top of it.

## Gate commands

Read the `## Quality gates` section of the project's root `CLAUDE.md` and pick
the row for the area the bead touches. No section or no matching row: ask the
owner before step 2, and suggest adding the section.

## Steps

1. **Pre-flight.** `bd show <id> --json | jq '.[0] | {title, acceptance_criteria, design, parked: .metadata.parked}'`.
   `parked` set (a re-run): `unpark` it so the earlier work and tests come
   back, then `bd update <id> --unset-metadata parked`.
   No acceptance criteria, or clearly more than one small change: write the
   question in the notes, `bd label add <id> human`, stop this bead.
2. **Write tests.** Run `beads-workflow:test-writer` with the bead ID, `vcs`
   and the gate command. `BLOCKED: human` -> stop this bead. `SKIP` -> step 3.
   `DONE` -> freeze the tests: read `.[0].metadata.tdd_files` from `bd show`,
   run `shasum <files> | shasum` with the paths written out, then copy the
   printed hash into `bd update <id> --set-metadata tdd_sum=<hash>`.
3. **Implement.** Run `beads-workflow:implementer` with the bead ID, `vcs` and
   the gate command. `BLOCKED: human` -> stop this bead.
4. **Gate.** Run each gate command yourself as a plain line (see "Rules"),
   with its output sent to a file: `<command> > /tmp/bd-gate-<id>.log 2>&1`.
   Test logs are long and would fill your context on every run, so use only
   the exit code the Bash tool reports; on a failure, read
   `tail -n 30 /tmp/bd-gate-<id>.log`. Exit 127 means command not found, not
   a failed gate: fix the command.
   Record `bd update <id> --set-metadata gate=pass|fail`. On a failure, write
   `GATE FAILED: <command> exited <code>` plus those lines into the bead notes,
   run `beads-workflow:reviser`, and skip the review this round.
5. **Review loop, max 3 rounds.** Before each review, if `tdd_sum` is set, run
   the same `shasum` command again. A different hash means the frozen tests
   were changed: write `blocker - test files changed after test-writer` in the
   notes, flag `human`, stop this bead (the originals cannot be restored, so
   the owner decides). Then `bd update <id> --unset-metadata verdict` so an old
   verdict cannot be read as new. Run `beads-workflow:reviewer`
   (bead ID, round number, `vcs`), then read
   `bd show <id> --json | jq -r '.[0].metadata.verdict // empty'`; if empty,
   use the `VERDICT:` line of its return.
   - `CLEAN` -> step 6.
   - `ISSUES` with a blocker marked `needs owner` in the notes -> flag `human`,
     stop this bead. No agent can fix it, so the fix loop would only cost runs.
   - Other `ISSUES` -> run `beads-workflow:test-writer` with `mode: findings` (bead
     ID, `vcs`, gate command), so each testable blocker gets a failing test
     from an agent that did not write the code. `BLOCKED: human` -> stop this
     bead. Freeze again: the step 2 `shasum` over the new `tdd_files`, record
     `tdd_sum`. Then run `beads-workflow:reviser` (bead ID, `vcs`, gate
     command), run the gate, review again.
   - Still `ISSUES` after round 3, or no verdict anywhere: flag `human`, stop this bead.
6. **Commit.** Run `status`: every changed file must belong to this bead;
   anything else (another session, a leftover agent) -> stop and ask the owner.
   Then check earlier beads' frozen tests: for each bead with `needs-review`
   and a `tdd_sum` (`bd list --label needs-review --json`), run the step 2
   `shasum` over its `tdd_files`. A different hash (or a missing file) means
   this bead changed them. Do not block: a change of shape can require it,
   and the reviewer has checked it. Record
   `bd update <id> --set-metadata frozen_changed="<bead-id> ..."`, store the
   new hash as that bead's `tdd_sum`, and name it in the report.
   Run `commit` with a Conventional Commit message (`type(scope): summary`) plus
   the session's attribution trailers, branch name `<type-short-summary>`; then
   `branch` and `next`, so the next bead stacks on top. Then
   `bd label add <id> needs-review` and
   `bd update <id> --set-metadata bookmark=<branch> --set-metadata change=<change id> --set-metadata rounds=<n>`
   (key `bookmark` in git too). Leave the bead open.

**Stopping a bead.** Whenever a step says "stop this bead", run `status`
first. If agents changed files, `park` them and record
`bd update <id> --set-metadata parked=<ref>`. Partial work left in the
working copy would leak into the next bead's commit, the full suite and the
epic review; parked, it waits for the re-run.

An agent that stops at its turn limit returns partial output with an agent ID:
resume it once; if it still does not finish, flag `human`.

## Final report (single bead only)

Short. Do not restate findings. Print one of:
- ready for review: the branch, the command to view it (`jj show <branch>` or
  `git show <branch>`), `metadata.rounds`, the beads in `frozen_changed`
  (their frozen tests were changed), and each `assumption:` from
  `metadata.tdd` (the frozen tests encode them, so the owner must see them)
- needs the owner's decision: the bead ID and the question from its notes
- not started: the failed check from "Before you start"

## Rules

- Never push, never close a bead: the owner approves first. This includes
  `bd human respond`, which closes the bead. When the owner answers a `human`
  bead, write the answer with `bd update <id> --notes "..."`, run
  `bd label remove <id> human`, and run the bead again.
- One branch per bead, stacked in order.
- Stop every agent (`TaskStop <agent-id>`) as soon as you have read its return.
  An agent can report DONE while its background commands still run, and keep
  editing files after the commit, landing them in the next bead's change.
- Keep your own context small: never read a diff or files, never copy agent output.
- Write every command out in full: no shell variables (`$?`, `H=$(...)`),
  loops or heredocs. Claude Code cannot check them and asks for approval, and
  fish does not split a variable into words. Copy values such as a hash or a
  change id from one command's output into the next command.
