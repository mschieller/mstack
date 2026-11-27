---
name: audit
description: Bring a repo's dependencies, Node version and GitHub Actions up to date in one run. Fixes what `pnpm audit` or `npm audit` reports with patch and minor updates, lists the open Dependabot alerts it did not close, asks before a major upgrade that is not free, removes overrides nothing needs any more, pins versions, moves Node to the active LTS with an exact version in apps and the active LTS major as the floor of a published package, moves GitHub Actions to their newest release pinned to a commit SHA, and sets up or updates Dependabot version updates with the minor and patch updates grouped into weekly PRs. Grades how far the tests cover what the run changed and names tests to add when they fall short. Surveys first, asks its questions in one round, works unattended until one round of questions about majors, then reports, picks a single PR or a stack, and asks once before it ships. Always runs the whole job. Use when the user says "audit", "audit the repo" or "fix the vulnerable dependencies". For a security review of code, use security-review.
---

# Audit

Do the whole job in one run and stop at the summary. The user reads it and confirms the ship before anything is pushed.

## Rules

- Commit locally, one commit per unit, in the order of the steps below. A stack can then be cut at commit boundaries.
- Before the first commit, read the title rules in step 3 of the `mstack:ship` skill, the repo's commitlint config and `git log --oneline -20`. Take the language from ship's step 3, and the types, scopes and lengths from the commitlint config and the log. Write every commit title in that form, so that no title needs a rewrite before the push.
- Run the repo's formatter on each file the run writes, before the commit that adds it: the repo's `format` script, or the formatter the repo uses, such as Prettier or Biome.
- Never push the audited repo or open a PR in it before the user has confirmed the ship in step 12.
- When a script or check needs Docker and `docker info` fails, record it as not run, and say in the summary that CI runs it. Never start, stop or poll Docker.

## Ecosystems

The steps below are the same for every repo. What differs between ecosystems is in a file of its own, under a heading that names the step:

- [NODE.md](NODE.md), for a repo with a `package.json`

The survey finds the ecosystems the repo has. Where a step says to do the ecosystem part, do what each file says under its heading for that step, once per ecosystem, and commit per ecosystem. A file with no heading for a step has no part in it. In a repo with none of these ecosystems, the ecosystem parts have nothing to do. Leave the counts table out of the summary then, and say so there.

## 1. Survey

Change no tracked file in this step. It creates the branch, installs and runs the baseline scripts, and writes nothing else.

- Run `git status --short`. If the tree has uncommitted changes, add that to the questions in step 2. If the current branch is the default branch, create `chore/audit` from it, or `chore/audit-2` and so on when an earlier run left that name taken. Record the merge base with the default branch, `git merge-base HEAD origin/<default>`, for steps 10 and 12.
- Find the ecosystems the repo has, from the list under [Ecosystems](#ecosystems). Read the file of each one now, with its rules, before the survey goes on.
- Do the ecosystem part.
- Read the repo's open Dependabot alerts with `gh api --paginate 'repos/<owner>/<repo>/dependabot/alerts?state=open'`. Record each one's ecosystem, package, `manifest_path` and advisory id. When the call fails, alerts are off or the token lacks access, so record the message for the summary.
- Find who owns the repo with `gh api repos/<owner>/<repo> --jq '.owner.login, .owner.type'`. Record the login for steps 4 and 7. The type is `Organization` or `User`.
- In a repo that an org owns, look for a `CODEOWNERS` file in `.github/`, at the root and in `docs/`, the three places GitHub reads. When there is none, collect candidates for the question in step 2: the user's own login from `gh api user`, and the user's teams from `gh api user/teams`, filtered to that org and to the teams that pass the access check in step 3. In a repo that a user owns, skip this item, the code owners question and step 3, because one owner for everything changes nothing there.
- Record two facts about update bots:
  - whether `.github/dependabot.yml` exists, and which ecosystems and directories its entries cover
  - whether a Renovate config exists: `renovate.json`, `renovate.json5`, `.renovaterc`, `.renovaterc.json` or `.renovaterc.json5`, at the root or under `.github/`, or a `renovate` key in the root `package.json`

## 2. Questions

Ask everything the survey raised in one round with AskUserQuestion, with the recommended option first. A round may take several dialogs, since one dialog holds four questions. Ask nothing the survey can answer. Typical questions:

- Whether to add Dependabot version updates, when the survey found no `dependabot.yml` and no Renovate config. Offer yes and no, and recommend yes, because the exact versions and commit SHAs this run writes go stale when no bot moves them. With no, step 4 is skipped.
- A missing `CODEOWNERS` file in a repo that an org owns. Ask whether to add one. If yes, ask who owns the code: one owner for everything, or different owners for different paths, and which teams or people. Offer the candidates from the survey and let the user name others. Recommend one team for `*` when the user is in a team with access to the repo.
- The questions the ecosystem part lists.
- Uncommitted changes in the tree. Offer to stop so the user can commit or move them, or to commit them now as a first commit of their own, below every audit commit. Recommend stopping. The run cannot go on with a dirty tree, because the rebase in step 10 refuses one.

After this round the run is unattended until the majors question in step 9, and again after it. When something new comes up, take the option you would have recommended and list it in the summary.

## 3. Code owners

Skip this step when a user owns the repo, when the repo has a `CODEOWNERS` file, or when the user declined one in step 2.

- Write `.github/CODEOWNERS` in the form the user chose. A team is `@<org>/<team-slug>` and a person is `@<login>`. One owner for everything is a single line:

  ```
  * @<org>/<team-slug>
  ```

- With several owners, write one line per path. The last line that matches a file decides its owners, so put `*` first and the narrower paths after it.
- GitHub ignores an owner that has no write access to the repo. Check each one before you write it. `gh api -H 'Accept: application/vnd.github.v3.repository+json' orgs/<org>/teams/<team-slug>/repos/<org>/<repo> --jq .permissions.push` prints `true` for a team that can write, and fails for a team without access. Without that header the endpoint answers 204 with an empty body even when the team has access. `gh api repos/<org>/<repo>/collaborators/<login>/permission` gives a person's permission. Report an owner that fails the check and leave it out.
- GitHub validates the file only on a pushed branch. Put the command for that check in the summary: `gh api "repos/<org>/<repo>/codeowners/errors?ref=<branch>"`.
- Commit.

## 4. Dependabot

This run writes exact versions and commit SHAs, and Dependabot's version updates keep them current. Skip this step when the survey found a Renovate config, and leave that config as it is, because the repo already has an update bot. Skip it too when the user declined Dependabot in step 2.

- If the repo has no `.github/dependabot.yml`, write one with an entry for each ecosystem the repo has:

  ```yaml
  version: 2
  updates:
    - package-ecosystem: "github-actions"
      directory: "/"
      schedule:
        interval: "weekly"
        day: "monday"
      groups:
        non-major:
          patterns: ["*"]
          update-types: ["minor", "patch"]
      ignore:
        - dependency-name: "<owner>/*"
    - package-ecosystem: "docker"
      directory: "/"
      schedule:
        interval: "weekly"
        day: "monday"
  ```

  The example holds the entries that are the same in every repo. Write `github-actions` only when the repo has a workflow, and `docker` only when it has a Dockerfile. Each ecosystem part adds its own entry to `updates`, and may add an `ignore` list to the `docker` entry when the repo has that entry.

  For `github-actions`, `directory: "/"` covers `.github/workflows` and an `action.yml` at the root. With `action.yml` files in other folders, write `directories` with `/` and each of those folders in place of `directory`, so the pins step 7 writes there stay current too. Write the entry when the repo has a workflow. `<owner>` is the owner's login from the survey, so the references step 7 leaves alone stay where they are. Dependabot updates a SHA pin and the version comment after it together.

  Write the `docker` entry when the repo has a Dockerfile, with the Dockerfile's folder as `directory`. With Dockerfiles in several folders, write `directories` with the list in place of `directory`.

  The `non-major` group puts the minor and patch updates of an entry into one PR each Monday. A major gets a PR of its own, and so does each image update. Write no `cooldown`. Dependabot already waits 3 days after a release before it offers a version update, and that wait does not hold back a security update.

- Do the ecosystem part.

- If the repo has a `dependabot.yml`, keep its entries and add what is missing: an entry for each ecosystem above that it lacks, and in the entries it has, the `ignore` entries and, in every entry except `docker`, the `non-major` group. The ecosystem part may name more. Change no other key an entry sets, such as its `schedule`. Skip each of these when the config already covers it, and list each one you skip in the summary, with the lines that cover it:

  - `non-major`, when a group in the entry already takes the minor and patch updates of every dependency
  - an `ignore` entry, when one in the entry already covers the same name and update types

- Leave Dependabot alerts and security updates as they are. They are repo settings, not part of this file.
- Check the file. It parses as YAML, with `ruby -ryaml -e 'YAML.load_file(ARGV[0])' <file>` when `ruby` is installed and otherwise with a YAML parser that is installed. No two entries share a `package-ecosystem` and a directory. Each group name starts and ends with a letter. Dependabot reads the file from the default branch, so it reports a config error only after the merge, on the Dependabot tab under Insights and Dependency graph.
- Commit.

## 5. Pin and settings

Do the ecosystem part.

## 6. Runtime version

Do the ecosystem part. It moves the runtime the repo builds and runs on, such as Node.

## 7. Workflows

This step covers every `uses:` in `.github/workflows/` and in the repo's own `action.yml` and `action.yaml` files. Skip local paths such as `./.github/actions/build` and `docker://` images.

- A reference to an action or a reusable workflow that belongs to the repo's owner, from the survey, stays as it is, whether it names a branch, a tag or a SHA. The same owner controls both, and a pin would keep the owner's fixes from reaching the repo. List these references in the summary.
- Every other reference gets pinned as described below, GitHub's own actions included.

To pin an action:

- Take the highest version among the releases at least 3 days old. A release that is broken, or that an attacker published, is usually pulled within days, and Dependabot waits the same 3 days before it offers a version update. `gh api --paginate repos/<owner>/<repo>/releases` lists them with `published_at`, but not in version order, since a backport to an old major can come first. Skip drafts and prereleases. When the repo has no releases, take the highest version tag.
- Resolve the tag with `gh api repos/<owner>/<repo>/commits/<tag> --jq .sha`. This endpoint follows an annotated tag to its commit.
- Write the reference as `<owner>/<repo>/<path>@<full sha> # <tag>`, keeping the path when the reference has one, such as `/init` in `github/codeql-action/init`, and with the full tag such as `v4.2.1`. Dependabot updates the SHA and the comment together.

When a reference would move to a new major, read the release notes between the two versions. Compare the inputs the workflow passes and the outputs it reads with `action.yml` or `action.yaml` in the action's path at the new tag, fetched with `gh api -H 'Accept: application/vnd.github.raw' 'repos/<owner>/<repo>/contents/<path>/action.yml?ref=<tag>'`. For a reusable workflow, compare with its `on.workflow_call` block. Take the major when each one still exists with the same meaning. An input or output that the new version marks as deprecated counts as changed. Otherwise take the newest version in the current major. Record the major for step 9 when the version the repo ends up on has an advisory, or when the latest run of the workflow has an annotation that the action's Node runtime is deprecated. `gh run view <run-id>` lists the annotations. Record it for the summary otherwise.

Check each action against the GitHub Advisory Database at the version the repo ends up on: `gh api '/advisories?ecosystem=actions&affects=<owner>/<repo>@<version>'`, with the version written without a leading `v`. When an advisory has a fixed version in the current major, take that version or newer. When only a new major fixes it, record that major for step 9.

Commit when done.

## 8. Updates and fixes

Do the ecosystem part.

## 9. Majors

Do the ecosystem part for the majors of dependencies. It says when a major is attempted, when it is free, what its target is, and which rules every upgrade follows.

This step handles the majors of dependencies that step 8 left, and the Node and action majors that steps 6 and 7 recorded.

### Free majors

The ecosystem part says when a dependency major is free.

For an action, the input and output check in step 7 is the free check.

### The question

Write a brief for each major that is not free, whether it is a dependency, Node or an action:

- the advisories it fixes, with id and severity, or for a Node or action major without one, the reason step 6 or 7 recorded it, whether the package is a dev dependency or reaches production, and whether the repo calls the code the advisory names
- the current version and the target
- the peer dependencies the target declares, and whether the versions the repo installs meet them
- each breaking change between them and what it does
- the breaking changes that hit the repo, each with its `file:line`, and the edits the upgrade needs
- for a published package, whether the dependency reaches its public API, from the check below

Show all briefs, then ask with AskUserQuestion in one round, one question per major. A round may take several dialogs. Offer "Upgrade" and "Skip and report", with the one you recommend first.

### Upgrades

For each approved major:

- Upgrade to the target and make the edits in the brief. For Node, update every place step 6 lists. For an action, update the reference and the inputs and outputs the workflow uses.
- Run the baseline scripts. When the upgrade needs an edit the brief did not list, or a script that passed before still fails, undo it and record the reason.
- Commit each major on its own.

The ecosystem part holds the rules that free and approved majors follow alike.

### Test grade

Two results, each a label from the lists below. "This PR's changes" answers how far a green run of the baseline scripts proves this run's changes safe. The free check, the override removals in step 8 and the verify step all treat a green run as proof. "The repo's tests" judges the tests of the whole repo, so that a run that changed little still says whether the repo needs more tests. This section is the one place that defines the labels. The brief below copies it, and the legend in step 12 says the same in a reviewer's words.

Start the grading at the end of this step, also when the step had no majors, and go on with step 10 while the agent reads. Wait for its report before step 11. A change that step 10 undoes drops out of the touched set in the summary.

The touched set is every package the run moved, directly or through the lockfile or an override, plus Node and each Docker base image when their version changed. Group transitive packages under their direct parent, so the evidence check runs once per parent. Actions are left out, because the PR's own CI run exercises the workflows. The runtime changes are the direct and transitive packages, Node and the base images: what the app runs with. Dev tools and type packages are listed with their evidence but not counted, because a passing script is the only evidence they can have.

Evidence for a touched item is a test that would fail if the behavior the repo relies on changed:

- A direct dependency: tests of the code that imports it.
- A transitive package: tests of the code that calls its direct parent.
- A dev tool or type package, such as a linter, test runner, bundler, type checker or `@types/*`: the baseline script that runs it. A script that ran and passed is full evidence.
- Node: tests that can fail, which passed on the new version.
- A base image: a test or CI job that starts the app in the image. A build alone is no evidence.
- A test that failed in the baseline is no evidence.
- A test that cannot fail on a wrong value, or that asserts only on mocks or snapshots, is no evidence. Line coverage counts only together with assertions that check behavior.

This PR's changes, with a count of N runtime changes with evidence out of M:

- Covered: every runtime change has evidence.
- Partly covered: some runtime changes have evidence and some have none.
- Not covered: no runtime change has evidence. A green run then proves little beyond the install, the build and the typecheck.
- Nothing to cover: the run has no runtime change. It may still move dev tools, type packages and actions.

The repo's tests looks at the behavior that can break in the repo: business rules, permission checks, persistence and migrations, parsing of external formats and state transitions.

- Good: that behavior has tests that can fail.
- Has gaps: most of it has, and the gaps are named.
- Little or none: less than about half of it has, the repo has no tests, or most tests cannot fail.

Start one `mstack:test-reviewer` with the Agent tool, in the background, also when the repo has no tests. When the repo has its own `.claude/agents/test-reviewer.md`, set `subagent_type` to `test-reviewer` instead. Omit model and effort, because the agent file sets them. The brief contains:

- the touched set, each entry with its kind (direct, transitive with its chain, dev tool, Node or base image) and, in a workspace, the package it belongs to. In a workspace the scope is those packages, or the whole repo when the touched set is empty.
- the baseline results for each script, and the scratchpad path of the coverage report when step 1 made one
- the runtime changes, the evidence rules and both label lists above, copied in
- this rule: "Run nothing: no tests, no coverage, no install and no build. Another process reinstalls packages and rewrites git history while you read. Use the baseline results in this brief in place of your own test run, and leave out your usual test-run line."
- the output, these seven lines first, then one line per touched item with its name, its kind (direct, transitive, Node, base image, or dev tool, not counted), its evidence or "none", and its line coverage when the brief has a report, then the agent's usual findings. The lines that name things give names only, with no reasons, because they go into the PR word for word. The two covered lines are for the summary in step 11 only:

  ```
  PR changes: <label from the list>
  PR changes with tests: <N> of <M>
  PR changes without tests: <the names of the runtime changes without evidence, or none>
  PR changes covered: <what has evidence>
  Repo tests: <label from the list>
  Repo tests gaps: <the names of the behavior without tests that can fail, or none>
  Repo tests covered: <the behavior with tests that can fail>
  ```

When the agent fails or returns no result lines, the summary says that the results are missing and why. The summary gives the agent's results as it wrote them. When a repo's own test reviewer answers with strong, adequate or weak, read strong as Covered or Good, adequate as Partly covered or Has gaps, and weak as Not covered or Little or none, and say so in the summary. When you disagree with a result, say so on a line of its own.

## 10. Verify

- Run every script from the survey's baseline again. A script that passed before must pass now.
- Do the ecosystem part.
- When a Dockerfile changed and Docker is available, build the image.
- When a workflow changed and `actionlint` is installed, run it. When it is not installed, skip it and leave it out of the summary.
- Trace each new failure to its commit. Fix it there, or undo that commit's change and record why.
- A fix or an undo from this step goes into the commit that caused it, never on top, so that a stack can still be cut at commit boundaries. Commit it with `git commit --fixup=<sha>`, then run `git rebase --autosquash <merge-base>`. The branch is not pushed, so rewriting it is safe.
- Record the audit counts again. These are the "after" numbers.
- Record `git rev-parse HEAD^{tree}` after the last verify run, after any fixup and autosquash. Ship skips its checks when it pushes this same tree.

## 11. Summary

In T3 Code, send the summary with `send_to_user` in parts of under 4000 characters, because the tool refuses a longer message.

Report in this order:

- The lines of the ecosystem part.
- Each open Dependabot alert from the survey that the run did not close, with the reason, or that alerts are off.
- Workflows: each action as `{name}: {old} to {new}`, and each major not taken with its reason. The references left as they are because the repo's owner owns them.
- Code owners: the file added and its owners, or that the repo already had one, or that the user declined. The command that checks the file after the push. Leave this line out when a user owns the repo.
- Dependabot: added, updated or already correct, skipped because the repo uses Renovate, or declined by the user. When declined, say that nothing moves the exact versions and the SHA pins until the next audit. The groups and `ignore` entries not added because the config already covers them. An entry with `versioning-strategy: increase` in a repo with published packages. The peer hints. Where GitHub shows a config error after the merge.
- Majors: each free one done, each one approved or skipped in step 9, and each approved one undone with its reason.
- Both test results from step 9, each with its count, its covered line and the names without tests or the gaps, the evidence for each touched item, and the line coverage per touched item when step 1 measured it. When the PR's changes are Partly covered or Not covered, or the repo's tests have gaps or little or none, up to 5 tests to add, picked from the agent's findings: code that calls a touched package first, then by risk. Each names the behavior, the file, and the lowest layer where a test would fail on the current code.
- What the user must test by hand because this run could not, including each runtime change without evidence.
- Each choice made without asking.
- The PR shape from step 12.

## 12. Ship

Count the change with `git diff --numstat <merge-base> HEAD`, leaving out lockfiles.

Pick a single PR when no major upgrade needed source changes and the count is under about 300 changed lines. Otherwise pick a stack, cut at commit boundaries in this order, bottom first:

1. code owners and Dependabot
2. pin and settings
3. runtime version
4. workflows
5. updates and fixes
6. one layer per major that changed source code

Fold a small layer into its neighbour. Say how many layers the stack has and why each one is separate.

Ask with AskUserQuestion whether to ship. Name the shape, and offer "Ship" first and "Don't ship yet". Stop when the user picks "Don't ship yet".

Do the ecosystem part, when the file has one, before the hand-over.

Then hand over to the `mstack:ship` skill. Give it the tree hash recorded in step 10, and these drafts for the PR:

- A title that leads with the outcome: how many advisories the run closes, and the largest other change, such as `fix: close 11 advisories and move to node 26`. Its type is that of the updates commit from step 8, or `chore` when there is none.
- A Summary: the advisory counts before the run and the packages they came in through, then what the PR does and the counts it leaves. When the run changed more, one sentence names the other areas.
- Changes: one bullet per change, with the old and new version and the reason. When step 4 of `mstack:ship` groups the bullets, the labels are Dependencies, Node, and CI and repo config, which holds the workflows, Dependabot and code owners. A lone Node bullet joins Dependencies, and a label with no bullets is left out.
- After the merge, when the run wrote or changed `dependabot.yml`: where GitHub shows a config error.
- Verified: the scripts that passed on the new versions, and what was not run, such as a Docker build.

For a stack, give it the layer plan: for each layer, bottom to top, the branch name, its last commit, the PR title, and these drafts for that layer's changes. Each layer without the Audit section ends its body with `The audit results are in #<n>.`, where `<n>` is the PR of the layer that has the section. Ship fills in the number once `gh stack submit` has opened the PRs.

Give `mstack:ship` this section for the PR body, filled in from the summary, and tell it to copy the section word for word, also into a repo's PR template. In a stack it goes on the layer with the updates and fixes, or on the bottom layer when there is none.

```markdown
## Audit

### Results

| | Result |
|---|---|
| Advisories | <total before> to <total after>, <none high, or the high and critical left>. <how many advisories are left open, and whether dev only or in production>. |
| Dependabot | <n> open alerts close when this merges. |
| This PR's changes | <PR changes>: <N> of <M> runtime changes have tests. None for <PR changes without tests>. |
| The repo's tests | <Repo tests>: <Repo tests gaps> have no tests. |

<details>
<summary>Advisories, evidence and tests to add</summary>

#### Advisories

| | Info | Low | Moderate | High | Critical |
|---|---:|---:|---:|---:|---:|
| Before | <n> | <n> | <n> | <n> | <n> |
| After | <n> | <n> | <n> | <n> | <n> |

#### Left open

| Advisory | Package | Reaches the repo through | Fixed when |
|---|---|---|---|
| <id>, <severity>, <dev only or production> | <package> <version> | <dependency chain> | <what has to change before a fix reaches the repo> |

#### Evidence per runtime change

| Change | Kind | Evidence |
|---|---|---|
| <name and version> | <direct, transitive, Node or base image> | <evidence or none, and line coverage when measured> |

Dev tools, not counted: <name and version> (<the script that passed>).

#### Tests to add

- `<file:line>`: <the test>

<details>
<summary>What the results mean</summary>

**This PR's changes** counts runtime changes: the packages, Node versions and images the app runs with. Dev tools are not counted.

- **Covered.** Every runtime change has a test that fails when the behavior the repo relies on changes. Green CI is real evidence, so review the diff as usual.
- **Partly covered.** The runtime changes the row names have no such test. Green CI says nothing about them, so try the code that uses them by hand.
- **Not covered.** No runtime change has such a test. Green CI proves only that the app installs, builds and typechecks. Run the app and try its main paths before merging.
- **Nothing to cover.** The PR has no runtime change. Their own scripts check the dev tools it moves. Review the config changes by reading them.

**The repo's tests**

- **Good.** The behavior that can break, such as business rules, permission checks, persistence, parsing of external formats and state transitions, has tests that can fail.
- **Has gaps.** Most of that behavior has such tests, and the row names the gaps. A change in a named gap needs a check by hand.
- **Little or none.** Less than about half of that behavior has such tests, or the repo has none. Treat every change to this repo as untested.

</details>
</details>
```

Fill in the Results rows like this, and change no words of the reviewer's lines:

- Advisories: `None before or after.` when the audit found none. Leave the row out when step 1 skipped the dependency steps.
- Dependabot: `<n> open alerts, <m> close when this merges.` when some stay open, and each one that stays gets a row in "Left open". `No open alerts.` or `Alerts are off.` when that is the case.
- This PR's changes: leave out the second sentence when the result is Covered or Not covered. Nothing to cover reads `Nothing to cover: the PR has no runtime change.`
- The repo's tests: Good reads `Good.`

In the fold:

- Leave out each severity column that is 0 both before and after. Leave out "Advisories" when the Advisories row says none or is left out, and "Left open" when no advisory or alert is left.
- Copy the reviewer's item lines for runtime changes into "Evidence per runtime change" and the dev tools onto the line below it. Leave the part out when the result is Nothing to cover, and the dev tools line when there are none.
- Tests to add are plain bullets, not checkboxes, because GitHub counts every checkbox in the body in the PR's task badge. Leave the part out when the PR's changes are Covered or Nothing to cover and the repo's tests are Good.
- Copy "What the results mean" as it stands.
- The fold's `<summary>` names the parts it holds, such as "Tests to add". When it would hold only the legend, the legend's own fold takes its place.

Keep the blank line after each `<summary>` line, because GitHub renders a table or list inside `<details>` only after one.
