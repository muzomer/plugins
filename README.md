# plugins

My personal agent skills I use for coding with Claude Code.

| Plugin                                               | Description                                                                                                                                                                                        |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`beads-workflow`](plugins/beads-workflow/README.md) | Implements [Beads](https://github.com/gastownhall/beads) issues and epics through implementer, reviewer and reviser agents, producing one reviewed jj or git commit per bead on a dedicated branch |

## Installation

Register the marketplace, then install a plugin from the table above.

```sh
claude plugin marketplace add muzomer/plugins  # append --scope project|local when required
claude plugin install <plugin>@muzomer          # use the same --scope value
```

The equivalent commands inside Claude Code are
`/plugin marketplace add muzomer/plugins` and `/plugin install <plugin>@muzomer`.
Each plugin's README lists its requirements and setup.

## License

Released under the [MIT License](LICENSE).
