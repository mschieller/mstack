# mstack

Skills for the daily development loop, from settling a plan to a green PR, and the reviewer agents that `deep-review` runs in parallel.

```bash
claude plugin marketplace add mschieller/mstack
claude plugin install mstack@mstack
```

## Skills

Each skill triggers from its description, or you call it as `/mstack:<skill>`. The link opens the instructions Claude follows.

| Skill | What it does | Arguments |
|---|---|---|
| [grill](skills/grill/SKILL.md) | Settles every decision in a plan through rounds of numbered questions | The plan, issue or topic. `docs` first also records the decisions in the repo. `issues` first also drafts GitHub issues |
| [ask-me](skills/ask-me/SKILL.md) | Walks a list of findings or proposals one item at a time | Item numbers or a section name narrow the list. `decide` records the answers and changes nothing on disk |
| [architecture](skills/architecture/SKILL.md) | Finds restructuring candidates, settles the ones you pick and builds them | An area, a folder or `#123` for a PR. `drastic` widens the question to the whole service |
| [review](skills/review/SKILL.md) | Reviews the current branch or PR in one context before it ships | `#123` for a PR. `report` stops after the report. Other words are focus areas |
| [deep-review](skills/deep-review/SKILL.md) | Reviews a repo, PR, milestone or plan with parallel reviewer agents | A scope: `#123`, `branch`, `M2` for a milestone, a path or a plan file. `max N` sets the number of subagents, 6 by default. `report` changes nothing. Other words are focus areas |
| [docs-review](skills/docs-review/SKILL.md) | Checks the docs against each other and against the code | `docs` or `code` runs one pass only. `report` changes nothing. Doc names or areas narrow the findings |
| [accessibility-review](skills/accessibility-review/SKILL.md) | Reviews a frontend against WCAG 2.2 level AA and gives every criterion a status | URLs of a running app, and paths. `report` changes nothing |
| [ship](skills/ship/SKILL.md) | Commits, opens or updates the PR, and works until CI is green | An issue number or other context for the PR |
| [sync-base](skills/sync-base/SKILL.md) | Brings the branch up to date with its base and looks for conflicts git cannot see | `rebase` or `merge` sets the strategy. A branch name sets the base |
| [audit](skills/audit/SKILL.md) | Updates npm, pnpm and Go dependencies, the Node or Go version, GitHub Actions and Dependabot, and grades the tests that cover the changes | None |
| [visualize](skills/visualize/SKILL.md) | Draws how something works as a diagram anchored in the code | What to draw. Empty means the current topic |
| [unslop](skills/unslop/SKILL.md) | Removes AI patterns from prose. Claude loads it on its own, and it has no slash command | None. Ask "unslop" with a file or a piece of text |

## Agents

The reviewers review only and never edit. `deep-review` picks them for the scope, and you can start one yourself as `mstack:<agent>`. A repo's own `.claude/agents/<agent>.md` with the same name runs in its place under `deep-review` and `audit`. `audit` uses `test-reviewer` to grade the tests.

| Agent | Finds |
|---|---|
| `bug-reviewer` | Wrong behavior, races, broken error handling, data loss |
| `security-reviewer` | Auth gaps, injection, secrets, unsafe defaults, CI supply chain |
| `spec-reviewer` | Code that differs from its issue, milestone, plan or ADRs |
| `state-reviewer` | Implicit state machines and impossible states |
| `simplify-reviewer` | Duplication, dead code, needless abstraction |
| `perf-reviewer` | N+1 queries, repeated work, memory growth, blocking calls |
| `docs-reviewer` | Docs that no longer match the code |
| `test-reviewer` | Risky behavior with no test, tests that cannot fail |
| `ops-reviewer` | Unsafe rollouts and migrations, health checks, observability gaps |
| `accessibility-reviewer` | Failures of the WCAG 2.2 level A and AA criteria, with a status for each criterion. Runs only when asked, or under `accessibility-review` |

## What it needs

- `gh`, logged in, for the skills that read or open PRs.
- The [browser](../browser/README.md) plugin, for `accessibility-review` and `accessibility-reviewer` to check a running app. Install it with `claude plugin install browser@mstack`. The official Playwright plugin works too, except in a repo that pins pnpm in `devEngines`. Without either, they check the source only.
- T3 Code is optional. `ship`, `architecture`, `audit` and `visualize` use its tools when they are present and fall back to the terminal otherwise.
