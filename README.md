# plugins

Claude Code plugins by [muzomer](https://github.com/muzomer), published as the
`muzomer` plugin marketplace.

| Plugin | What it does |
|---|---|
| [`beads-workflow`](#beads-workflow) | Implement [Beads](https://github.com/gastownhall/beads) issues and epics with implementer, reviewer and reviser agents; each bead becomes one reviewed jj or git commit on its own branch |

## Install

Add the marketplace once, then install a plugin. Pick the scope that fits:

| Scope | Who gets it | Flag |
|---|---|---|
| `user` (default) | you, in every project | no flag |
| `project` | everyone who opens this repo (written to `.claude/settings.json`, commit it) | `--scope project` |
| `local` | you, in this repo only (not committed) | `--scope local` |

Pass the same `--scope` to both commands, so the marketplace is registered
where the plugin is enabled:

```sh
claude plugin marketplace add muzomer/plugins  # add --scope project|local if needed
claude plugin install beads-workflow@muzomer    # same --scope here
```

Inside Claude Code the same works with `/plugin marketplace add muzomer/plugins`
and `/plugin install beads-workflow@muzomer`.

For a project, you can instead commit this to `.claude/settings.json`, so
teammates are asked to install the plugin when they trust the folder:

```json
{
  "extraKnownMarketplaces": {
    "muzomer": { "source": { "source": "github", "repo": "muzomer/plugins" } }
  },
  "enabledPlugins": { "beads-workflow@muzomer": true }
}
```

## beads-workflow

Turns Beads issues into reviewed commits, one bead at a time. You stay the
approver: it never pushes and never closes a bead.

| Component | Name | Role |
|---|---|---|
| Skill | `/beads-workflow:run-bead <id>` | Run one bead |
| Skill | `/beads-workflow:run-epic <id>` | Run every ready child of an epic, in dependency order |
| Agent | `beads-workflow:implementer` | Writes the change (uncommitted) |
| Agent | `beads-workflow:reviewer` | Read-only review against the acceptance criteria |
| Agent | `beads-workflow:reviser` | Fixes the reviewer's blocker findings |

Per bead: implement -> run the quality gate -> review -> revise (max 3 rounds)
-> commit with a Conventional Commit message on its own branch
(`type-short-summary`), stacked on the previous bead. In jj this is a bookmark
plus `jj new`; in git, a new branch from the current `HEAD`. A repo where
`jj root` works is treated as jj, even when it is colocated with git.

### Requirements

- [`bd`](https://github.com/gastownhall/beads) (written against bd 1.2.2; needs
  `bd update --claim` and `--set-metadata`)
- `jq`
- `jj` or `git`
- Optional: `hunk` - the final report
  suggests `hunk show <branch>` when it is installed

### Set up your project

Each bead must have **acceptance criteria**; beads without them are flagged for
you instead of being implemented.

Add a `## Quality gates` section to the project's root `CLAUDE.md`. The skill
picks the row for the area a bead touches and runs it after every change:

```markdown
## Quality gates

| Area | Gate command |
|---|---|
| `api/` | `cd api && go test ./... && golangci-lint run` |
| `web/` | `pnpm --dir web lint && pnpm --dir web test` |
```

Without this section the skill asks you for the gate before it starts.

### What it writes to your beads

| Kind | Name | Meaning |
|---|---|---|
| Label | `needs-review` | Committed; waiting for you to review and close |
| Label | `human` | Blocked on your decision; the question is in the bead notes |
| Metadata | `gate` | `pass` or `fail` of the last gate run |
| Metadata | `verdict`, `review_round` | Last review result and round |
| Metadata | `bookmark`, `change`, `rounds` | Branch name (jj or git), change or commit id, review rounds used |

Notes hold the reviewer's findings (`blocker|nit file:line - problem`) and the
reviser's replies.

To answer a `human` bead: write the answer with `bd update <id> --notes "..."`,
remove the label with `bd label remove <id> human`, and run the bead again.
Don't use `bd human respond`: it closes the bead.

## License

[MIT](LICENSE)
