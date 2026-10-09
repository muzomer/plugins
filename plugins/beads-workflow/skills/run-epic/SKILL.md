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

## Final report

Short. Do not restate findings. Print:
- beads ready for review, in stack order, each with its branch and the command
  to view it (`jj show <branch>` or `git show <branch>`; `hunk show <branch>` if
  `hunk` is installed)
- beads that need the owner's decision (`bd human list`), with IDs
- rounds used per bead (`metadata.rounds`)

Also write one note on the epic: order, branches, rounds.
