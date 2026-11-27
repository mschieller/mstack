# mstack

Skills and reviewer agents for Claude Code. They cover the daily development loop, from settling a plan to a pull request with green CI.

## Install

```bash
claude plugin marketplace add mschieller/mstack
claude plugin install mstack@mstack
```

The `browser` plugin is optional. Install it with `claude plugin install browser@mstack`.

## Plugins

| Plugin | What it holds |
|---|---|
| [mstack](plugins/mstack/README.md) | Fifteen skills and ten reviewer agents: grill, ask-me, architecture, review, deep-review, docs-review, accessibility-review, tdd, bug, handoff, ship, sync-base, audit, visualize and unslop |
| [browser](plugins/browser/README.md) | The Playwright browser tools, started so that they also work in a repo that pins pnpm in `devEngines`. Optional. `accessibility-review` uses it to check a rendered page |

## License

MIT, see [LICENSE](LICENSE). The skills `architecture`, `bug`, `grill`, `handoff`, `ship`, `tdd`, `unslop` and `visualize` are adapted from other MIT-licensed work. Each keeps the original license in its folder and names its source at the top of its `SKILL.md`.
