---
name: run-epic
description: Implement every ready child of a Beads (`bd`) epic in dependency order, each through the run-bead loop (implement -> review -> revise), and commit each child on its own stacked jj bookmark or git branch for the owner to review. Never pushes or closes beads; skips beads that need a human decision. Use when the user runs "/beads-workflow:run-epic <epic-id>", asks to run, work through, or process an epic by ID, or approves implementing a specific epic ID that was just discussed (e.g. "yes, run epic bd-3", or "go ahead" right after epic bd-3 was proposed) - never implement an epic inline. Use run-bead instead when the ID is a single bead.
---

# Run epic

Input: an epic bead ID. You are a thin coordinator. Agents do the work; you pass
bead IDs and read only their short returns and `VERDICT:` lines. Never read a diff
yourself. Never push.

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

Not `bd ready`: it treats a blocker as open until it is closed, but a finished
bead stays open with `needs-review` until the owner approves, so a chain of
dependent beads would stop after the first. A `needs-review` blocker counts as
done because its commit is already in the stack below. The query also skips
`human` beads and anything that depends on them. Take the first id; stop when
it prints nothing.

Run that bead with the **Version control** section (detect once per epic) and
the **Steps** of the `run-bead` skill (read
`${CLAUDE_PLUGIN_ROOT}/skills/run-bead/SKILL.md` once per run). Skip its "Before you start" and "Final report" sections: the query
above already does those checks, and the report is the one below. When a step
says "stop this bead", move on to the next bead. The `human` answers, gate
commands and rules also live there.

## After the last bead: full suite

Each bead runs only the gate row for the area it touches, so a change in one
area can break another area unnoticed. Once the query prints nothing, check the
whole stack:

1. Skip this section if no bead was committed in this run: nothing new to check.
2. Read every row of the project's `## Quality gates` section and run each
   distinct command once, on top of the stack. Run each command as a plain line
   and check its exit code, as in the run-bead **Gate** step. Exit 127 means
   the command was not found: fix the command, do not count it as a failure.
3. All pass: `bd update <epic> --set-metadata full_suite=pass`.
4. Any fails: `bd update <epic> --set-metadata full_suite=fail`, and keep the
   line `FULL SUITE FAILED: <command> exited <code>` plus the last lines of the
   error for the epic note (see "Final report"). Do not run the reviser: the
   cause can be in any bead of the stack, so the owner decides the fix.

## Final report

Short. Do not restate findings. Print:
- the full suite result first, when it ran; on a failure, the failing command
- beads ready for review, in stack order, each with its branch and the command
  to view it (`jj show <branch>` or `git show <branch>`; `hunk show <branch>` if
  `hunk` is installed)
- beads that need the owner's decision (`bd human list`), with IDs
- rounds used per bead (`metadata.rounds`)

Also write one note on the epic: the full suite failure line (if any), order,
branches, rounds.
