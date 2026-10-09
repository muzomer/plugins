---
name: run-epic
description: Run the child beads of an epic through implement -> review -> revise with the implementer, reviewer and reviser agents, then commit each bead and create a jj bookmark for the owner to review. Use when the user says "/beads-workflow:run-epic <epic-id>" or asks to run, work through, or process an epic. Also use it, without being named, whenever the user confirms or approves implementing an epic (e.g. "go ahead", "yes, implement it", "do <id>") — never implement an epic inline. Use run-bead instead when the approved ID is a single bead.
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

Run that bead with the **Steps** of the `run-bead` skill (read
`${CLAUDE_PLUGIN_ROOT}/skills/run-bead/SKILL.md` once per run). Skip its "Before you start" and "Final report" sections: the query
above already does those checks, and the report is the one below. When a step
says "stop this bead", move on to the next bead. The `human` answers, gate
commands and rules also live there.

## Final report

Short. Do not restate findings. Print:
- beads ready for review, in stack order, each with `hunk show <bookmark>`
- beads that need the owner's decision (`bd human list`), with IDs
- rounds used per bead (`metadata.rounds`)

Also write one note on the epic: order, bookmarks, rounds.
