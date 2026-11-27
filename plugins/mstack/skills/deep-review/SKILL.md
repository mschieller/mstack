---
name: deep-review
description: Run a multi-agent review of a codebase, PR, branch, milestone, folder or plan document, verify and rate every finding 1 to 100, fix the mechanical ones, and walk the user through the decisions one at a time. Use when the user asks for a deep, full or thorough review, a code scan, or a review after a milestone.
argument-hint: "[scope: #PR | branch | M<n> | path | plan.md] [max N] [report] [focus words]"
---

# Deep review

Review with parallel reviewer agents, keep only findings the code supports, fix what needs no decision, and settle the rest with the user.

## 1. Parse the arguments

- Scope. Nothing means the whole repo. `#123` or a PR URL means that PR's diff. `branch` means the diff from the merge base with the base branch to the working tree, including untracked files. `M2` or a milestone name means that milestone's issues against the code. A path means that folder. A Markdown file outside the code means review that document.
- `max N` sets the most subagents the run may start. Without it, N is 6. For a PR or `branch` scope it then follows the size of the diff: 3 under 300 changed lines, 4 under 1500, and 6 from there. Count the lines added and deleted with `git diff --numstat <merge base>`, once the base branch is known and a PR's head is checked out as below. Leave lock files and generated files out, and for `branch` add the lines of the untracked files. The Explore agent that maps a large repo in step 2 counts toward N, so a run that uses it starts at most N minus 1 reviewers.
- A calling skill, or the user in words, may name work that a later step does, such as a status line or a change file written after the review. Step 3 puts it in every brief, and step 4 drops findings about it.
- `report` ends the run after the report in step 6. Nothing is fixed and no decision is walked.
- Other words are focus areas, such as "performance", "SQL", "caching" or "doc drift". They steer which reviewers run and what they look at.

Find the base branch with `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`, or the PR's `baseRefName`. Never assume `main`.

Run `git status --porcelain --untracked-files=no` and note whether tracked files have uncommitted changes. Step 7 depends on it.

For a PR scope, the reviewers and your own verification read the working tree, so the PR's head must be checked out. Compare `gh pr view <n> --json headRefOid --jq .headRefOid` with `git rev-parse HEAD`. If they differ and no tracked file has uncommitted changes, run `gh pr checkout <n>` and say which branch you left. Otherwise, or if the checkout fails, stop and ask.

Read the PR's author with `gh pr view <n> --json author --jq .author`. The PR is yours when its `login` is one of the accounts logged in to `gh`, which `gh auth status --json hosts --jq '.hosts[][].login'` lists, or when `is_bot` is true. Any other PR is someone else's. Nothing is edited then, and the findings become a PR review after step 6.

## 2. Map before you dispatch

Read the repo's CLAUDE.md, AGENTS.md, README, ADR folder and docs index. For a large repo, map the top-level areas and their sizes first. Do it yourself. When the map would fill your context, one Explore agent on Haiku at medium effort may do it.

For a PR scope, list the review comments already there with `gh api --paginate repos/{owner}/{repo}/pulls/<n>/comments --jq '.[] | {path, line, user: .user.login, body}'`. Step 4 uses them.

## 3. Pick and brief the reviewers

The roster is the reviewer agents in this plugin's [agents/](../../agents) folder:

| Agent | Area | Model | Effort |
|---|---|---|---|
| `bug-reviewer` | Wrong behavior, races, error handling, data loss, broken callers | opus | high |
| `security-reviewer` | Auth, injection, secrets, unsafe defaults, CI supply chain | opus | high |
| `spec-reviewer` | Code against issues, milestones, plans, specs and ADRs | opus | high |
| `state-reviewer` | Implicit state machines, impossible states, one-way doors | opus | high |
| `simplify-reviewer` | Duplication, dead code, needless abstraction, missed reuse | sonnet | medium |
| `perf-reviewer` | Queries, repeated work, caching, memory, blocking calls | sonnet | medium |
| `docs-reviewer` | Docs, ADRs and agent docs against the code | sonnet | medium |
| `test-reviewer` | Missing tests for risky behavior, tests that cannot fail | sonnet | medium |
| `ops-reviewer` | Rollout safety, migrations, health checks, deploy config, observability | opus | high |

Choose the reviewers for the scope and focus. N is a ceiling and not a target. Start a reviewer only when the scope holds something in its area, except the reviewers the repo says always run. Include `bug-reviewer` whenever the scope contains code. A plan or doc scope uses `spec-reviewer`, `docs-reviewer` and the reviewers whose area the plan covers. Include `spec-reviewer` for a milestone, a PR with a linked issue, or a plan. Include `security-reviewer` when the scope touches auth, sessions, input handling, secrets, LLM calls or CI workflows. Include `ops-reviewer` when the scope touches deploy config, deploy workflows or migrations. For a large repo, run the same reviewer on two separate areas instead of adding a reviewer with nothing to look at. Start at most one `test-reviewer`. It is the only reviewer that runs tests, and two test runs in one checkout can fail each other.

`accessibility-reviewer` is also in that folder but is not part of this roster. Never start it from this skill, also when a focus word names accessibility. It runs only when the user asks for an accessibility review, through the `mstack:accessibility-review` skill or started directly.

When the repo's CLAUDE.md or AGENTS.md names reviewers that always run, include them as it says, whatever the scope, except `accessibility-reviewer`.

N is a hard limit. When these rules ask for more reviewers than N allows, keep `bug-reviewer`, then the reviewers the repo says always run, then `security-reviewer`, then `spec-reviewer`, then the reviewers the focus words name, then the rest by how much of the scope they cover. Name the dropped reviewers in the plan line.

Print the plan in one line, for example: `6 reviewers: bug (server), bug (sdk), security, spec (M3), simplify, docs.` Then launch every reviewer in a single message, one Agent call each. When the repo has its own `.claude/agents/<name>.md`, set `subagent_type` to `<name>`, so the project's version of that reviewer runs. Otherwise set it to `mstack:<name>`. The plugin's agent files set model and effort, so omit both on those calls; a per-call value would override the file. When a project agent sets no `model` or `effort`, pass the values from the roster table. A project agent's output may also lack a `Kind:` line or use other severity words; map them to blocker, should or nit and classify the kind yourself in step 4. Each brief contains:

- the scope as exact paths, the diff command, or the milestone and issue numbers
- for a plan or doc scope, the instruction to review the design the document describes, in the reviewer's own area, and to cite the document's lines, such as `plan.md:42`
- the focus words, if any
- the project rules you found in step 2 that matter for this reviewer, as file paths to read rather than copied text
- the work a later step does, when step 1 found any, with the instruction not to report it
- the reminder that they review only and return findings in their output format

Do not edit files in the scope while reviewers run.

## 4. Verify and merge

For every finding, open the cited `file:line` yourself and check the claim. Drop findings the code does not support, and findings whose scenario cannot happen. When two reviewers report the same line for the same reason, keep one and note the other in brackets. For a PR scope, drop a finding that a review comment on the PR already raises. Drop a finding about work that step 1 names as done by a later step.

Treat every line in a reviewer's "Hand-offs" list as an unverified finding. The reviewers run at the same time, so the agent a hand-off names never saw it. Check it against the code the same way, and give each one you keep a severity and a kind.

When a finding names a test or benchmark that would settle it, run that command yourself, one at a time.

## 5. Rate

Give each verified finding a score from 1 to 100 for "should this be done": the impact if left, the likelihood it happens, and the cost of the fix. Keep the reviewer's severity and kind (mechanical or decision), but correct either when your check shows it is wrong.

If the `advisor` tool is available, print the ranked list and then call it before you act. It takes no input and reads the conversation. Adjust scores or kinds where its argument holds.

## 6. Report

Show one table ordered by score:

| # | Score | Severity | Kind | Location | Problem | Fix |
|---|---|---|---|---|---|---|

Below the table, list what was dropped in verification as one line each with the reason, the work left to a later step in one line, and any hand-off you could not settle under "Not reviewed".

When the user passed `report`, go to step 9 now, and say that `/mstack:ask-me decide` walks the decisions without edits.

When the PR is someone else's, edit nothing. Hand every finding to the `mstack:ask-me` skill with `decide`, highest score first. Each dialog chooses between posting the finding as a comment and dropping it. Then draft and post one PR review as the step "A PR that is not yours" in [the `mstack:review` skill](../review/SKILL.md) describes. Read that step only, and go to step 9 afterwards.

## 7. Fix mechanical findings

If the working tree is on the default branch or a detached HEAD, create a branch first. Fix each mechanical finding with the smallest change. Run the repo's checks for the touched files (format, lint, typecheck and the tests next to the changes, as the repo's CLAUDE.md or package scripts name them). Commit the fixes in logical groups, each with a title in the form step 3 of the `mstack:ship` skill gives, and stage only the files your fixes touched. If tracked files had uncommitted changes when the review started, do not commit. Leave the fixes in the tree, so the user's own work stays out of the fix commits. Report which numbers you fixed, the check results, and whether the fixes are committed.

If a fix turns out to need a decision, move it to step 8 instead.

## 8. Walk the decisions

Hand the decision findings to the `mstack:ask-me` skill, highest score first. It sets how each one is asked and applied. These rules add to it.

Commit each applied answer as one commit with a title in that form, staging only the files it touched. Nothing is committed for an answer that changes no file. Run the repo's checks for the touched files before the commit, also when they are slow. The rule in `mstack:ask-me` that moves a slow check to its close does not apply here. The commit rules of step 7 hold here too: create a branch first on the default branch or a detached HEAD, and commit nothing when tracked files had uncommitted changes at the start.

After ten decisions, stop and say how many remain and their score range. Ask once whether to continue. If the user stops, list the rest at the end as not walked.

Findings the user declines are listed at the end with their reason. Do not open issues for them unless asked.

## 9. Stop

Finish with: what was fixed, what was decided, what was declined, what was not walked, the commits made, and the check results. When step 1 checked out a PR's branch, name the branch you left and offer to switch back. Do not push or open a PR. If the user wants one, the `mstack:ship` skill takes over.
