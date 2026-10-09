# plugins

A Claude Code plugin marketplace maintained by [muzomer](https://github.com/muzomer)
and published under the marketplace name `muzomer`.

| Plugin | Description |
|---|---|
| [`beads-workflow`](#beads-workflow) | Implements [Beads](https://github.com/gastownhall/beads) issues and epics through implementer, reviewer and reviser agents, producing one reviewed jj or git commit per bead on a dedicated branch |

## Installation

Installation takes two steps: register the marketplace, then install the plugin.
The installation scope determines where the plugin is available:

| Scope | Availability | Flag |
|---|---|---|
| `user` (default) | The current user, in every project | none |
| `project` | Every collaborator on the repository; recorded in `.claude/settings.json`, which is committed | `--scope project` |
| `local` | The current user, in the current repository only; not committed | `--scope local` |

Both commands require the same `--scope` value, so that the marketplace is
registered at the same level as the plugin:

```sh
claude plugin marketplace add muzomer/plugins  # append --scope project|local when required
claude plugin install beads-workflow@muzomer    # use the same --scope value
```

The equivalent commands inside Claude Code are
`/plugin marketplace add muzomer/plugins` and `/plugin install beads-workflow@muzomer`.

For project-wide adoption, an alternative is to commit the following to
`.claude/settings.json`. Claude Code then prompts each collaborator to install
the plugin upon trusting the repository folder:

```json
{
  "extraKnownMarketplaces": {
    "muzomer": { "source": { "source": "github", "repo": "muzomer/plugins" } }
  },
  "enabledPlugins": { "beads-workflow@muzomer": true }
}
```

## beads-workflow

Converts Beads issues into reviewed commits, one bead at a time. The repository
owner remains the approver: the workflow never pushes and never closes a bead.

| Component | Name | Role |
|---|---|---|
| Skill | `/beads-workflow:run-bead <id>` | Runs a single bead |
| Skill | `/beads-workflow:run-epic <id>` | Runs every ready child of an epic, in dependency order |
| Agent | `beads-workflow:implementer` | Implements the change, without committing |
| Agent | `beads-workflow:reviewer` | Reviews the change against the acceptance criteria (read-only) |
| Agent | `beads-workflow:reviser` | Resolves the blocker findings raised by the reviewer |

Each bead follows the same sequence: implementation, quality gate, review,
revision (at most three rounds), and a commit with a Conventional Commit message
on a dedicated branch named `type-short-summary`, stacked on the previous bead.
In jj, the branch is a bookmark followed by `jj new`; in git, a new branch
created from the current `HEAD`. Any repository in which `jj root` succeeds is
treated as a jj repository, including repositories colocated with git.

### Requirements

- [`bd`](https://github.com/gastownhall/beads), written against version 1.2.2;
  `bd update --claim` and `--set-metadata` are required
- `jq`
- `jj` or `git`
- `hunk` (optional): when installed, the final report suggests `hunk show <branch>`

### Project setup

Every bead requires **acceptance criteria**. Beads without acceptance criteria
are flagged for a human decision instead of being implemented.

The project's root `CLAUDE.md` requires a `## Quality gates` section. The
workflow selects the row matching the area a bead touches and runs the
corresponding command after every change:

```markdown
## Quality gates

| Area | Gate command |
|---|---|
| `api/` | `cd api && go test ./... && golangci-lint run` |
| `web/` | `pnpm --dir web lint && pnpm --dir web test` |
```

When the section or the matching row is missing, the workflow requests the gate
command before starting.

### Data recorded on beads

| Kind | Name | Meaning |
|---|---|---|
| Label | `needs-review` | Committed and awaiting review and closure by the owner |
| Label | `human` | Blocked on an owner decision; the open question is in the bead notes |
| Metadata | `gate` | Result of the last gate run: `pass` or `fail` |
| Metadata | `verdict`, `review_round` | Result and round number of the last review |
| Metadata | `bookmark`, `change`, `rounds` | Branch name (jj or git), change or commit ID, and review rounds used |

Bead notes hold the reviewer's findings, in the format
`blocker|nit file:line - problem`, and the reviser's replies.

To answer a bead labelled `human`, record the answer with
`bd update <id> --notes "..."`, remove the label with
`bd label remove <id> human`, and run the bead again. `bd human respond` is not
suitable, because the command closes the bead.

## License

Released under the [MIT License](LICENSE).
