# beads-workflow

Converts Beads issues into reviewed commits, one bead at a time. The repository
owner remains the approver: the workflow never pushes and never closes a bead.

| Component | Name                            | Role                                                           |
| --------- | ------------------------------- | -------------------------------------------------------------- |
| Skill     | `/beads-workflow:run-bead <id>` | Runs a single bead                                             |
| Skill     | `/beads-workflow:run-epic <id>` | Runs every ready child of an epic, in dependency order         |
| Agent     | `beads-workflow:test-writer`    | Writes failing tests for the acceptance criteria, before code  |
| Agent     | `beads-workflow:implementer`    | Implements the change, without committing                      |
| Agent     | `beads-workflow:reviewer`       | Checks the acceptance criteria, then tries to break the change |
| Agent     | `beads-workflow:reviser`        | Resolves the blocker findings raised by the reviewer           |
| Agent     | `beads-workflow:diagnoser`      | Explains why a bead broke the full suite (read-only)           |
| Agent     | `beads-workflow:epic-reviewer`  | Reviews all beads of an epic together (read-only)              |

Each bead follows the same sequence: failing tests, implementation, quality
gate, review, revision (at most three rounds), and a commit with a Conventional
Commit message on a dedicated branch named `type-short-summary`, stacked on the
previous bead.
`run-bead` owns these per-bead steps and `run-epic` reuses them, so a change
to `run-bead` changes both.

After the last bead of an epic, `run-epic` runs every gate command of the
project once on top of the stack, to catch breakage between areas that the
per-bead gates miss. A failure is reported to the owner and is not fixed
automatically. The report names the cause: the bead that introduced the failure
(found with `jj bisect` or `git bisect`), a failure that already existed before
the epic, or a flaky test. When a bead is the cause, the read-only diagnoser
explains the likely reason and suggests fixes.

When the full suite passes, the read-only epic reviewer reads all beads of the
run together and reports problems between beads that per-bead reviews miss: a
missing piece, drift from the epic's design, inconsistency, duplication, and
leftovers. Its findings go to the owner and are not fixed automatically.

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

## Project setup

Every bead requires **acceptance criteria**. Beads without acceptance criteria
are flagged for a human decision instead of being implemented.

Beads are built test-first. The test-writer turns each acceptance criterion
into a test and confirms it fails before any code exists; it skips beads with
nothing to test. Its test files are then frozen: the workflow records their
checksum and stops the bead if the implementer or reviser changes them.

In round 1, the reviewer also reviews adversarially: it looks for bugs the
tests miss, and each blocker must name a concrete failing scenario. Before the
reviser fixes a blocker, the test-writer adds a failing test that reproduces
it, so every fix is proven by a test written by another agent.

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
command before starting. Prefer quiet test flags (for example `pytest -q`): the
workflow reads only the exit code and, on a failure, the last 30 lines.

The skills run the coordinator on Sonnet, because it only routes work; each
agent keeps its own model.

An optional `## Review checklist` section lists the project's own risks. The
reviewer checks each item, on top of its generic checks (inputs, error paths,
security, state, resources). Add an item whenever a bug reaches review that a
check would have caught:

```markdown
## Review checklist

- Every endpoint checks that the user belongs to the project's organisation.
- API responses stay backwards compatible: add fields, never rename or remove.
```

## Data recorded on beads

| Kind     | Name                           | Meaning                                                              |
| -------- | ------------------------------ | -------------------------------------------------------------------- |
| Label    | `needs-review`                 | Committed and awaiting review and closure by the owner               |
| Label    | `human`                        | Blocked on an owner decision; the open question is in the bead notes |
| Metadata | `gate`                         | Result of the last gate run: `pass` or `fail`                        |
| Metadata | `verdict`, `review_round`      | Result and round number of the last review                           |
| Metadata | `bookmark`, `change`, `rounds` | Branch name (jj or git), change or commit ID, and review rounds used |
| Metadata | `tdd`                          | Each test and why it failed before the code, or `skip: <reason>`     |
| Metadata | `tdd_files`, `tdd_sum`         | The frozen test files and their checksum                             |
| Metadata | `frozen_changed`               | Earlier beads whose frozen tests this bead changed                   |
| Metadata | `parked`                       | Where a stopped bead's partial work waits: jj change or git stash    |
| Metadata | `full_suite` (epic only)       | Result of the full suite after the last bead: `pass` or `fail`       |
| Metadata | `full_suite_culprit` (epic)    | Cause of a failure: a bead ID, `pre-existing`, `flaky`, or `unknown` |

To answer a bead labelled `human`, record the answer with
`bd update <id> --notes "..."`, remove the label with
`bd label remove <id> human`, and run the bead again. `bd human respond` is not
suitable, because the command closes the bead.

## License

Released under the [MIT License](../../LICENSE).
