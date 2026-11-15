---
name: run-epic
description: Run every ready child of a Beads (`bd`) epic, in dependency order, through the run-bead loop, each committed on its own stacked jj bookmark or git branch, then run the full test suite. Use when the user runs "/beads-workflow:run-epic <id>", asks to run or work through an epic by ID, or approves implementing an epic just discussed (e.g. "go ahead") - never implement an epic inline. Use run-bead for a single bead.
---

# Run epic

Input: an epic bead ID. Read `${CLAUDE_PLUGIN_ROOT}/skills/run-bead/SKILL.md`
once per run: its **Version control** (detect once per epic), **Gate
commands**, **Steps** and **Rules** apply here. Skip its "Before you start" and
"Final report": the query below does those checks, and the report is below.

## Per child bead

Pick the next bead with this query, and re-run it after every bead:

```bash
bd list --parent <epic> --status=all --json | jq -r '
  (map({key: .id, value: ((.status == "closed") or ((.labels // []) | index("needs-review") != null))}) | from_entries) as $done
  | .[]
  | select(.status != "closed")
  | select((.labels // []) | (index("human") or index("needs-review")) | not)
  | select([.dependencies[]? | select(.type == "blocks") | ($done[.depends_on_id] // false)] | all)
  | .id'
```

Not `bd ready`: it keeps a blocker open until it is closed, but a finished bead
stays open with `needs-review`, so a dependent chain would stop after one bead.
The query also skips `human` beads and their dependents. Run the first id
through the run-bead Steps; "stop this bead" means go to the next. Stop when
the query prints nothing.

## After the last bead: full suite

Each bead runs only its area's gate, so one area can break another unnoticed.
Skip this section if no bead was committed in this run.

1. Run each distinct command of the `## Quality gates` section once, on top of
   the stack, with the rules of the run-bead **Gate** step.
2. All pass: `bd update <epic> --set-metadata full_suite=pass`, then go to
   "Epic review".
3. A command fails: find the cause. Run `status` first: if the working copy is
   not clean, skip the search (moving between commits could lose changes) and
   use cause `unknown`. Otherwise stop at the first match:
   1. Run it again where you are. It passes: `flaky`.
   2. `checkout` the `parent` of the first bead committed in this run (the
      base) and run it. It fails: `pre-existing`. Required, because bisect
      assumes the base is good and would wrongly blame the first bead.
   3. `bisect` with the base as `<good>` and the top bead's branch as `<bad>`.
      The first bad commit maps to the bead whose `metadata.bookmark` points
      to it: that bead ID.

   Then `checkout` the top branch. Record
   `bd update <epic> --set-metadata full_suite=fail --set-metadata full_suite_culprit=<cause>`
   and keep `FULL SUITE FAILED: <command> exited <code>; caused by <cause>`
   plus the last lines of the error for the epic note. Do not run the reviser:
   the owner decides the fix.
4. The cause is a bead ID: run `beads-workflow:diagnoser` with that bead ID,
   its branch, `vcs`, the failing command and the error lines, then stop it
   (`TaskStop`). Its return (at most 5 lines) is an agent output you copy:
   keep it for the epic note and the report, labelled "likely cause".

## Epic review

Each bead was reviewed on its own, so problems between beads (a missing
piece, drift from the design, duplication) can slip through. Run this only
when the full suite passed: on a failure, that comes first.

Run `beads-workflow:epic-reviewer` with the epic ID, `vcs`, the base (the
`parent` of the first bead committed in this run) and the top branch, then
stop it (`TaskStop`). Its return (at most 10 lines) is the other agent output
you copy: keep it for the epic note and the report. Do not fix anything and
do not run the reviser: fixes often span beads, so the owner decides.

## Final report

Short. Do not restate findings. Print:
- first, the full suite result; on a failure, the command, its cause, and the
  diagnosis (if any); on a pass, the epic review lines
- beads ready for review, in stack order, each with its branch, view command
  and assumptions (as in the run-bead final report)
- beads that need the owner's decision (`bd human list`), with IDs
- `metadata.rounds` per bead

Write one note on the epic: the full suite failure line and diagnosis, or
the epic review lines; then order, branches, rounds.
