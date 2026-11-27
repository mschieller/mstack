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
| [ask-me](skills/ask-me/SKILL.md) | Walks a list of findings or proposals one item at a time | Item numbers or a section name narrow the list. `decide` records the answers and changes nothing on disk. On an open `grill` round it works as under `decide` without the word |
| [architecture](skills/architecture/SKILL.md) | Finds restructuring candidates, settles the ones you pick and builds them | An area, a folder or `#123` for a PR. `drastic` widens the question to the whole service |
| [review](skills/review/SKILL.md) | Reviews the current branch or PR in one context before it ships | `#123` for a PR. `report` stops after the report. Other words are focus areas |
| [deep-review](skills/deep-review/SKILL.md) | Reviews a repo, PR, milestone or plan with parallel reviewer agents | A scope: `#123`, `branch`, `M2` for a milestone, a path or a plan file. `max N` sets the most subagents. The default is 6, and for a PR or `branch` scope 3 under 300 changed lines and 4 under 1500. `report` changes nothing. Other words are focus areas |
| [docs-review](skills/docs-review/SKILL.md) | Checks the docs against each other and against the code | `docs` or `code` runs one pass only. `report` changes nothing. Doc names or areas narrow the findings |
| [accessibility-review](skills/accessibility-review/SKILL.md) | Reviews a frontend against WCAG 2.2 level AA and gives every criterion a status | URLs of a running app, and paths. `report` changes nothing |
| [ship](skills/ship/SKILL.md) | Commits, opens or updates the PR, and works until CI is green | An issue number or other context for the PR |
| [tdd](skills/tdd/SKILL.md) | Builds a behaviour change test-first, one committed slice at a time | An issue number, a plan file or the behaviour to build |
| [bug](skills/bug/SKILL.md) | Takes a bug from a failing test through the root cause and the fix to a PR | The bug report, an issue number or a failing test. A "why" question stops at the root cause |
| [handoff](skills/handoff/SKILL.md) | Writes a note that lets a later session carry on, and finds it again | A slug for the task. `resume` first reads the newest matching note |
| [sync-base](skills/sync-base/SKILL.md) | Brings the branch up to date with its base and looks for conflicts git cannot see | `rebase` or `merge` sets the strategy. A branch name sets the base |
| [audit](skills/audit/SKILL.md) | Updates npm, pnpm and Go dependencies, the Node or Go version, GitHub Actions and Dependabot, and grades the tests that cover the changes | None |
| [visualize](skills/visualize/SKILL.md) | Draws how something works as a diagram anchored in the code | What to draw. Empty means the current topic |
| [unslop](skills/unslop/SKILL.md) | Removes AI patterns from prose. Claude loads it on its own, and it has no slash command | None. Ask "unslop" with a file or a piece of text |

## Agents

The reviewers review only and never edit. `deep-review` picks them for the scope, and you can start one yourself as `mstack:<agent>`. A repo's own `.claude/agents/<agent>.md` with the same name runs in its place under `deep-review`. `audit` uses `test-reviewer` to grade the tests, and a repo's own `test-reviewer` runs in its place there too.

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

## What a repo can tell mstack

The skills read the repo's `CLAUDE.md` or `AGENTS.md`. They look for these terms there:

| Term | Read by | What it gives |
|---|---|---|
| grill checklist | `grill` | Questions every plan in the repo must answer, as a list under that heading or a pointer to a file. It may also give a record format, and then `grill` writes one record in that format in every session, with or without `docs` |
| test command | `tdd`, `bug` | How to run one test, and how to run the tests of a package or module |
| test layers | `tdd`, `bug` | The kinds of test the repo requires. `bug` takes "the layer that should have caught it" from this list |
| commit rules | `ship`, and through it `tdd`, `bug` and `audit` | Types, scopes, title lengths and footers. They win over the defaults, except for the language |
| reviewers that always run | `deep-review`, `bug` | Reviewers that `deep-review` includes whatever the scope is, within `max N`, right after `bug-reviewer` |

A repo can also replace a reviewer. A file `.claude/agents/<name>.md` with the name of a reviewer above runs in its place under `deep-review`. Under `audit` only `test-reviewer` can be replaced, and `accessibility-review` always runs the plugin's `accessibility-reviewer`.

The skills also follow other instructions in those files, such as the check commands and where decisions are recorded. Only the terms above are stable.

## The reviewer output format

A repo's own reviewer returns findings in this form, because `deep-review` checks, rates and sorts them by it:

```
[blocker|should|nit] path/to/file:LINE: one-line claim
  Scenario: what triggers it, and what goes wrong
  Fix: the smallest concrete change
  Kind: mechanical | decision
```

- The label on the second line may differ. The plugin's reviewers use Scenario, Load, Source, Code, Evidence and Behavior.
- A fix is mechanical when it changes no behavior anyone relies on, no public API, no schema and no design. Anything else is a decision, and the Fix line then gives the options.
- Findings come most severe first. A reviewer may add sections of its own, such as a coverage table. The findings end with a "Hand-offs" list, one line for each thing outside the reviewer's area. With no findings, the findings part is one line.
- A reviewer never edits files, commits, pushes or starts a server. It runs no builds or installs, because the reviewers share one checkout. Only `test-reviewer` runs tests, limited to the files in scope. `security-reviewer` may run the repo's audit command, `simplify-reviewer` its dead-code tool, and `accessibility-reviewer` its accessibility lint and tests.
- `deep-review` maps other severity words to these three and works out the kind itself when a `Kind:` line is missing. A reviewer in this form leaves it nothing to guess.

## Commands a guard hook must allow

A repo that blocks risky git commands with a hook has to let these through:

- `git push --force-with-lease --force-if-includes origin HEAD`, which `sync-base` uses after a rebase or an amend.
- `git push --force-with-lease=<branch>:<sha> origin HEAD`, which `sync-base` uses after it cherry-picked commits from the remote. `--force-if-includes` would refuse there.
- `git commit --no-verify` with a message that starts with `sync-base: temporary`. `sync-base` sets uncommitted work aside in such commits before a rebase and removes them after it.
- `git commit --fixup` and `git rebase -i --autosquash` with `GIT_SEQUENCE_EDITOR=:`. `ship` and `audit` use them on commits that are not pushed, and `ship` adds `--autostash`. `sync-base` uses them after a rebase whose push then goes out with a lease.
- `git commit --amend` on the last commit when it is not pushed, and `git branch -m` on a branch with no upstream, in `ship`.
- `git rebase origin/<base>`, `git rebase --onto`, `git rebase --abort`, `git cherry-pick`, `git reset --soft HEAD~1` and `git reset --mixed HEAD~1`, in `sync-base`.
- `git restore <path>` on a tracked file that a script changed, in `audit`.
- `git bisect`, in `bug`.
- Writes to the `mstack/handoffs` folder inside the repo's git folder, in `handoff`.
- `gh stack submit`, `gh stack push`, `gh stack sync` and `gh stack rebase`, in the stack mode of `ship`. They rebase and push with a lease.

The skills never use a plain `--force`, never push to the default branch unless you ask, never use `--no-verify` on any other commit, and never run `git stash` or `git reset --hard`.

## What a calling skill can hand over

A skill of your own that calls these skills can pass them the following.

| To | What it hands over | What the skill does with it |
|---|---|---|
| `ship` | The tree hash its checks ran on | Skips its own checks when the tree still has that hash and nothing is uncommitted |
| `ship` | A draft of the title, the Summary or the Changes, and a section for the PR body | Keeps the facts of the draft in its own format, and copies the section word for word at the end of the body |
| `ship` | A layer plan for a stack | Opens the stack by that plan without asking for one |
| `deep-review` | Work that a later step does, such as a status line it updates after the review | Tells the reviewers not to report it, and names it in one line of the report |
| `grill` | That an approval stop of its own follows | Writes the summary and the records, and leaves the closing question to that stop |
| `tdd`, `bug` | That it already ran `mstack:handoff resume` | Does not run it again |
| `handoff` | Its name, its next step and the issue or record it works on | Writes them in the note's Skill and Ref lines, and `resume` loads that skill at that step |
| `ask-me` | A list of findings, and a rule for what follows each answer, such as a commit | Walks the list, follows the rule and returns the outcomes |

## What is stable

Other repos call these by name, so they change only with a new major version, and before 1.0 with a new minor version:

- the skill names and the arguments in the Skills table
- the names of the reviewer agents
- the terms under "What a repo can tell mstack"
- the reviewer output format
- `mstack:handoff resume`

The hand-overs under "What a calling skill can hand over" are not stable yet, apart from `mstack:handoff resume` itself, and may change in any release before 1.0.

Each release has a tag and release notes on GitHub.

## What it needs

- `gh`, logged in, for the skills that read or open PRs.
- The [browser](../browser/README.md) plugin, for `accessibility-review` and `accessibility-reviewer` to check a running app. Install it with `claude plugin install browser@mstack`. The official Playwright plugin works too, except in a repo that pins pnpm in `devEngines`. Without either, they check the source only.
- T3 Code is optional. `ship`, `architecture`, `audit` and `visualize` use its tools when they are present and fall back to the terminal otherwise.
