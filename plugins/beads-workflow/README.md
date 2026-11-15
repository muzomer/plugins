# beads-workflow

Converts Beads issues into reviewed commits, one bead at a time. The repository
owner remains the approver: the workflow never pushes and never closes a bead.

| Component | Name                            | Role                                                           |
| --------- | ------------------------------- | -------------------------------------------------------------- |
| Skill     | `/beads-workflow:run-bead <id>` | Runs a single bead                                             |
| Skill     | `/beads-workflow:run-epic <id>` | Runs every ready child of an epic, in dependency order         |
| Agent     | `beads-workflow:implementer`    | Implements the change, without committing                      |
| Agent     | `beads-workflow:reviewer`       | Reviews the change against the acceptance criteria (read-only) |
| Agent     | `beads-workflow:reviser`        | Resolves the blocker findings raised by the reviewer           |

Each bead follows the same sequence: implementation, quality gate, review,
revision (at most three rounds), and a commit with a Conventional Commit message
on a dedicated branch named `type-short-summary`, stacked on the previous bead.

After the last bead of an epic, `run-epic` runs every gate command of the
project once on top of the stack, to catch breakage between areas that the
per-bead gates miss. A failure is reported to the owner and is not fixed
automatically. The report names the cause: the bead that introduced the failure
(found with `jj bisect` or `git bisect`), a failure that already existed before
the epic, or a flaky test.

## Installation

```sh
claude plugin marketplace add muzomer/plugins  # append --scope project|local when required
claude plugin install beads-workflow@muzomer    # use the same --scope value
```

## Requirements

- [`bd`](https://github.com/gastownhall/beads), written against version 1.2.2;
  `bd update --claim` and `--set-metadata` are required
- `jq`
- `jj` or `git`; with jj, a version that has `jj bisect run` (tested with 0.41)
- `bash`, used to run gate commands during bisect
- `hunk` (optional): when installed, the final report suggests `hunk show <branch>`

## Project setup

Every bead requires **acceptance criteria**. Beads without acceptance criteria
are flagged for a human decision instead of being implemented.

The project's root `CLAUDE.md` requires a `## Quality gates` section. The
workflow selects the row matching the area a bead touches and runs the
corresponding command after every change:

```markdown
## Quality gates

| Area   | Gate command                                   |
| ------ | ---------------------------------------------- |
| `api/` | `cd api && go test ./... && golangci-lint run` |
| `web/` | `pnpm --dir web lint && pnpm --dir web test`   |
```

When the section or the matching row is missing, the workflow requests the gate
command before starting.

## Data recorded on beads

| Kind     | Name                           | Meaning                                                              |
| -------- | ------------------------------ | -------------------------------------------------------------------- |
| Label    | `needs-review`                 | Committed and awaiting review and closure by the owner               |
| Label    | `human`                        | Blocked on an owner decision; the open question is in the bead notes |
| Metadata | `gate`                         | Result of the last gate run: `pass` or `fail`                        |
| Metadata | `verdict`, `review_round`      | Result and round number of the last review                           |
| Metadata | `bookmark`, `change`, `rounds` | Branch name (jj or git), change or commit ID, and review rounds used |
| Metadata | `full_suite` (epic only)       | Result of the full suite after the last bead: `pass` or `fail`       |
| Metadata | `full_suite_culprit` (epic)    | Cause of a failure: a bead ID, `pre-existing`, or `flaky`            |

To answer a bead labelled `human`, record the answer with
`bd update <id> --notes "..."`, remove the label with
`bd label remove <id> human`, and run the bead again. `bd human respond` is not
suitable, because the command closes the bead.

## License

Released under the [MIT License](../../LICENSE).
