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
- The reference file for the package manager holds the commands. The repo pins its own package manager version, so check a command against its `--help` in the repo when it fails or behaves differently.
- A dependency version is written exact, as `1.2.3`, with no `^`, `~` or range. Two kinds of entry keep a range, as steps 5, 8 and 9 describe: every `peerDependencies` entry, and the `dependencies` and `optionalDependencies` of a published package. A consumer of a published package resolves those itself, and an exact version there forces a second copy on every consumer that has another version.
- When the repo has a `.changeset` folder, a commit that changes what a released package ships adds its changeset, as [Changesets](#changesets) describes.

## Changesets

A released package is one the release tool versions:

- With Changesets, a package without `"private": true` that is not in `ignore` in `.changeset/config.json`, and a private package when `privatePackages` in that file says so. Changesets 2 versions private packages unless `privatePackages` is `false` or sets `version: false`. Changesets 3 versions them only when `privatePackages` is `true` or sets `version: true`.
- With `pnpm change`, every package. It does not read `.changeset/config.json`.

A commit needs a changeset for each released package whose `package.json` it changes in `dependencies`, `optionalDependencies`, `peerDependencies` or `engines.node`, or whose `catalog:` entry in one of those fields it changes in `pnpm-workspace.yaml`. The bump is a patch for a dependency update or pin and for a changed `engines.node` in an app, and a minor for a raised or widened peer range or for an `engines.node` added to or raised in a published package. A major update of a dependency that reaches the package's public API, as step 9 checks, is a major. When a package gets several of these in one commit, take the largest. A released app also gets a patch changeset when the commit fixes an advisory in what the app installs through the lockfile or an override alone, because the app ships what its lockfile resolves. Every other change to `devDependencies`, overrides, settings, the lockfile, workflows, Dockerfiles, code owners and Dependabot needs none. A published package gets none for the lockfile or an override, because its consumers resolve their own versions.

Write the changeset as a file in `.changeset/`, in the commit it describes, so a stack keeps it with its layer. Any name ending in `.md` works except `README.md`. Changesets and `pnpm change` read the same format:

```markdown
---
"<package>": patch
---

<what changed for consumers, such as "Update fastify to 5.6.1">
```

The summary of a raised peer range says that consumers on an older version of that peer must update it, and names the advisory when one caused it. Check the release plan with `npx changeset status`, or `pnpm change status`. In a pnpm repo that uses Changesets, run `pnpm exec changeset` in place of `npx changeset`, because `npx` stops with `EBADDEVENGINES` when `devEngines.packageManager` names pnpm. When CI fails a PR without a changeset, give each layer without one a changeset that releases nothing: `npx changeset add --empty`, or `pnpm change <package> --bump none --summary <text>`.

## 1. Survey

Change no tracked file in this step. It creates the branch, installs and runs the baseline scripts, and writes nothing else.

- Run `git status --short`. If the tree has uncommitted changes, add that to the questions in step 2. If the current branch is the default branch, create `chore/audit` from it, or `chore/audit-2` and so on when an earlier run left that name taken. Record the merge base with the default branch, `git merge-base HEAD origin/<default>`, for steps 10 and 12.
- Find the package manager from the `packageManager` field and the lockfile. Read [PNPM.md](PNPM.md) for `pnpm-lock.yaml` and [NPM.md](NPM.md) for `package-lock.json`. With any other package manager, or none, skip steps 5 and 8 and the dependency majors in step 9, skip the install, audit, outdated and override items in this step and in step 10, leave the counts table out of the summary, and say so there.
- List every `package.json` in the repo and whether the repo is a workspace.
- Decide which packages are published. A package is published when it has no `"private": true` and either the registry has it or the repo publishes it. The registry has it when `npm view <name> name` succeeds and either the name is scoped, such as `@scope/name`, or `npm view <name> repository.url` names this repo. An unscoped name can belong to someone else's package, as `app` and `server` do. A restricted package needs the user's npm login, and without it `npm view` answers 404. The repo publishes it when a workflow or script runs `npm publish`, `pnpm publish` or `changeset publish`, or the package has a `publishConfig`. Every other package is an app.
- Run the frozen install. If it fails because the lockfile does not match `package.json`, add that to the questions in step 2, since every later step needs the install.
- When the install fails with a 401 or a 404 on a scoped package, run `npm whoami`. A 401 there means the npm login has expired. Ask the user with AskUserQuestion to run `npm login`, then run the install again. Never ask for a token or offer to take one.
- Record the audit counts for info, low, moderate, high and critical. These are the "before" numbers in the summary.
- Record the packages with a newer major, for the summary.
- Read the repo's open Dependabot alerts with `gh api --paginate 'repos/<owner>/<repo>/dependabot/alerts?state=open'`. Record each one's ecosystem, package, `manifest_path` and advisory id. When the call fails, alerts are off or the token lacks access, so record the message for the summary.
- List every override in the places the reference file names. Find why each one was added, from a comment next to it or from the commit that added it and that commit's PR. `git log -S'<name>' --reverse -- <file>` lists the commits that changed the name, oldest first. Record the reason as a fix, when the forced version fixes an advisory or a bug, or as a hold-back, when a newer version broke the repo. Record it as unknown when nothing says.
- Find every place that sets a Node version: `engines.node`, `.node-version`, `.nvmrc`, `.tool-versions`, `volta`, `node-version` inputs in workflows, every `FROM` in a Dockerfile, `@types/node` and a `@tsconfig/node<major>` base.
- Find the active LTS line of Node. In `https://raw.githubusercontent.com/nodejs/Release/main/schedule.json` it is the line whose `lts` date has passed and whose `maintenance` date has not. Its newest release is the first entry in `https://nodejs.org/dist/index.json` whose `lts` field holds that line's codename.
- When the repo has a `.changeset` folder, find the release tool, `@changesets/cli` and its major in `devDependencies` or pnpm's own `pnpm change`, and with Changesets read `ignore` and `privatePackages` from `.changeset/config.json`. Check the CI workflows for a step that fails a PR without a changeset.
- Find who owns the repo with `gh api repos/<owner>/<repo> --jq '.owner.login, .owner.type'`. Record the login for steps 4 and 7. The type is `Organization` or `User`.
- In a repo that an org owns, look for a `CODEOWNERS` file in `.github/`, at the root and in `docs/`, the three places GitHub reads. When there is none, collect candidates for the question in step 2: the user's own login from `gh api user`, and the user's teams from `gh api user/teams`, filtered to that org and to the teams that pass the access check in step 3. In a repo that a user owns, skip this item, the code owners question and step 3, because one owner for everything changes nothing there.
- Record two facts about update bots:
  - whether `.github/dependabot.yml` exists, and which ecosystems and directories its entries cover
  - whether a Renovate config exists: `renovate.json`, `renovate.json5`, `.renovaterc`, `.renovaterc.json` or `.renovaterc.json5`, at the root or under `.github/`, or a `renovate` key in the root `package.json`
- Run every script in each `package.json` that tests, lints, typechecks or builds. Record what already fails. This is the baseline for step 10. A `test` script that is npm's placeholder, `echo "Error: no test specified" && exit 1`, is a known failure, recorded without a question. Run `git status --short` again afterwards. A tracked file that was clean before and that a script changed goes back with `git restore <path>`, so it stays out of the first commit.
- When the repo already measures coverage, through a coverage script or a coverage setting in the test runner's config, run the tests once with coverage and copy the report to the scratchpad for the test grade in step 9. Install no coverage provider. This run is not part of the baseline, so step 10 does not repeat it. Then delete the coverage folder when git does not ignore it and it did not exist before, so it stays out of every commit.

## 2. Questions

Ask everything the survey raised in one round with AskUserQuestion, with the recommended option first. A round may take several dialogs, since one dialog holds four questions. Ask nothing the survey can answer. Typical questions:

- Whether to add Dependabot version updates, when the survey found no `dependabot.yml` and no Renovate config. Offer yes and no, and recommend yes, because the exact versions and commit SHAs this run writes go stale when no bot moves them. With no, step 4 is skipped.
- A missing `CODEOWNERS` file in a repo that an org owns. Ask whether to add one. If yes, ask who owns the code: one owner for everything, or different owners for different paths, and which teams or people. Offer the candidates from the survey and let the user name others. Recommend one team for `*` when the user is in a team with access to the repo.
- An override with an unknown reason. Offer to treat it as a fix, which step 8 removes when nothing needs it, or as a hold-back, which step 8 keeps. Recommend a fix. The baseline scripts in step 8 catch many wrong removals, while a wrongly kept override stays until someone checks it by hand.
- Tests that fail before any change, other than npm's placeholder script. Offer to continue with those failures noted, or to stop.
- A frozen install that fails. Offer to update the lockfile in a first commit of its own, or to stop. Recommend the update, because the repo's CI fails the same way until someone makes it.
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
    - package-ecosystem: "npm"
      directory: "/"
      schedule:
        interval: "weekly"
        day: "monday"
      groups:
        non-major:
          patterns: ["*"]
          update-types: ["minor", "patch"]
      ignore:
        - dependency-name: "@types/node"
          update-types: ["version-update:semver-major"]
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
      ignore:
        - dependency-name: "node"
          update-types: ["version-update:semver-major"]
  ```

  `npm` is the ecosystem for npm and for pnpm. Its `directory` is the folder that holds the lockfile. Leave the entry out with any other package manager.

  For `github-actions`, `directory: "/"` covers `.github/workflows` and an `action.yml` at the root. With `action.yml` files in other folders, write `directories` with `/` and each of those folders in place of `directory`, so the pins step 7 writes there stay current too. Write the entry when the repo has a workflow. `<owner>` is the owner's login from the survey, so the references step 7 leaves alone stay where they are. Dependabot updates a SHA pin and the version comment after it together.

  Write the `docker` entry when the repo has a Dockerfile, with the Dockerfile's folder as `directory`. With Dockerfiles in several folders, write `directories` with the list in place of `directory`.

  The `non-major` group puts the minor and patch updates of npm packages into one PR each Monday, and those of actions into another. A major gets a PR of its own, and so does each image update. Write no `cooldown`. Dependabot already waits 3 days after a release before it offers a version update, and that wait does not hold back a security update.

- The two `ignore` entries for Node turn off its majors. Step 6 moves Node to a new major by hand, in every place at once, and odd majors never become LTS. Dependabot cannot change `engines.node` or `.node-version`, so a major it took would move the image or the types alone. Write them when the survey found a Node version. When an image holds Node under another name, such as a mirrored image, add that name to the `docker` entry. A minor or patch update of the image moves the Dockerfile only, and step 6 brings every place back to one version on the next run.

- With pnpm or npm, add a group for each family below whose main package a `package.json` in the repo lists. Its plugins declare the major they support, so each major of the family lands in one PR:

  | Family | Main package | `patterns` |
  |---|---|---|
  | fastify | `fastify` | `["fastify", "fastify-*", "@fastify/*"]` |
  | vite | `vite` | `["vite", "@vitejs/*"]` |

  ```yaml
  <family>-major:
    patterns: [<from the table>]
    update-types: ["major"]
  ```

  Put each family group above `non-major` in `groups`. Dependabot puts a dependency in the first group it matches.

- Write no `versioning-strategy`. Dependabot's default raises the version in an app and widens the range in a library, so the ranges of a published package stay ranges. When the survey found published packages and an existing entry sets `versioning-strategy: increase`, report it in the summary and leave it. That value raises the lower bound of every range the consumers of those packages resolve.

- If the repo has a `dependabot.yml`, keep its entries and add what is missing: an entry for each ecosystem above that it lacks, and in the entries it has, the `ignore` entries, the `non-major` group for `npm` and `github-actions`, and the family groups for `npm`. Change no other key an entry sets, such as its `schedule`. Skip each of these when the config already covers it, and list each one you skip in the summary, with the lines that cover it:

  - `non-major`, when a group in the entry already takes the minor and patch updates of every dependency
  - a family, when a group already matches its main package on majors
  - an `ignore` entry, when one in the entry already covers the same name and update types

- Leave Dependabot alerts and security updates as they are. They are repo settings, not part of this file.
- Check the file. It parses as YAML, with `ruby -ryaml -e 'YAML.load_file(ARGV[0])' <file>` when `ruby` is installed and otherwise with a YAML parser that is installed. No two entries share a `package-ecosystem` and a directory. Each group name starts and ends with a letter. Dependabot reads the file from the default branch, so it reports a config error only after the merge, on the Dependabot tab under Insights and Dependency graph.
- With pnpm or npm, collect peer hints for the summary: pairs of direct dependencies whose majors may need to move together but that no group covers. Read `peerDependencies` and `peerDependenciesMeta` from the installed `package.json` of each direct dependency, in the `node_modules` next to each `package.json` in the repo. A pair counts when the peer is a direct dependency of the same `package.json` and `peerDependenciesMeta` does not mark it optional. Drop a pair with a `@types/` package, since types follow their library's major. Drop a pair whose two packages share a family above. Write one line per peer with the packages that need it. Write no group for them.
- Commit.

## 5. Pin and settings

- Apply the settings from the reference file.
- Write every `devDependencies` entry as the exact version the lockfile resolves today, and in an app every `dependencies` and `optionalDependencies` entry too. Do not strip the `^` or `~` from the range. That gives the lowest version the range allows, which can be older than the installed one.
- In a published package, leave `dependencies` and `optionalDependencies` as they are.
- Leave entries that are not registry versions as they are: `workspace:`, `catalog:`, `file:`, `link:`, git and URL entries.
- Leave `peerDependencies` as they are, also for a package that `devDependencies` lists too. The next two bullets give the one exception.
- In a published package from the survey, raise the lower bound of a peer dependency range when two things hold. The range covers exactly one whole major, 1 or higher, such as `^6`, `6.x`, `^6.0.0` or `>=6 <7`. And `devDependencies` lists the same package. Write the version that entry resolves to today as a caret range, such as `^6.4.0`. A bare `^6` promises that `6.0.0` works, but the package's own tests only ran against the `devDependencies` version.
- Leave every other peer range, so that no upper bound moves and later audits do not raise it again: a range below major 1, a tilde range, and a range that allows more than one major, such as `^5 || ^6` or `>=6`. Leave a peer that `devDependencies` does not list, because the lockfile then holds whichever release the package manager picked. Leave a range that the resolved version falls outside, and report it.
- Update the lockfile without installing. Its resolved versions must not change in this commit.
- Commit.

## 6. Node

- The target is the newest release of the active LTS line from the survey. A move to a new major must pass the free check in step 9, with the target Node on the `PATH`. When the target Node cannot be put on the `PATH`, the major is not free. When the check fails, take the newest release of the current line instead, and record the major for step 9.
- Update every place from the survey. Write the exact target version, such as `26.13.1`, also in place of a range such as `^26` or `>=24.11.0 <25`, and also when the place already names the target's major. The exception is `engines.node` in a published package, which the bullet below sets. An exact version makes a laptop, CI and the image run the same Node.
- When the survey found no place that sets a Node version for an app, not counting `@types/node` and a `@tsconfig/node<major>` base, add `engines.node` to the `package.json` next to the lockfile, with the target as an exact version, such as `26.13.1`. Add it to an app in no other case. One place is enough, and each extra place has to be kept in step with the others.
- In a published package, `engines.node` is the oldest Node its consumers may run, not the Node it builds with. Set it to `>=<major>` of the active LTS line, such as `>=26`. Add it when it is missing, and replace a lower bound whatever form it has, such as `>=18` or `>=22.12`. Leave a bound at or above that major as it is.
- In a Dockerfile, keep the registry and the image variant and write the exact target version, so `node:26-slim` becomes `node:26.13.1-slim`. Confirm that the tag exists in its registry before you write it.
- Set `@types/node` to the newest version with the same major as Node. In a published package, `@types/node` and a `@tsconfig/node<major>` base follow the major of the lower bound in its `engines.node` instead. Types and a compile target for a newer Node let code through that the lowest Node it allows cannot run.
- Commit.

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

Work through these in order and stop as soon as the audit is clean. Move a package only to fix an advisory, and leave every other update to the repo's update bot.

1. Run the audit fix that updates the lockfile. Then compare the lockfile with `HEAD`. A package whose major changed while every package that depends on it kept its version came in through a wide range such as `*`, or through a peer resolution. Treat each one as a major for step 9.
2. For each advisory that is left, find which direct dependency pulls the package in. Move that dependency to the newest patch or minor of its current major when that removes the advisory. Below 1.0.0 the minor counts as the major, as a caret range treats it.
3. When no update within the current major removes the advisory, add an override, but only when the lowest fixed version is in the same major as the version it replaces. Scope the key to the part of the advisory's vulnerable range inside that major, and write the lowest fixed version as the value, such as `"ms@>=2.0.0 <2.1.3": "2.1.3"`. Copies outside the range then keep their version. Record the package, the version and the advisory id for the summary.
4. An advisory that only a new major fixes, through an override or a direct dependency, goes to step 9.

Item 2 moves only the entry in `dependencies`, `devDependencies` or `optionalDependencies`. A `peerDependencies` entry for the same package keeps its range and changes only by the rule in step 5, the advisory rule below and the rule in step 9.

In a published package, a `dependencies` or `optionalDependencies` entry keeps its range. An update inside the range changes only the lockfile. When an advisory covers a version the range allows, raise the lower bound to the fixed version and keep the operator, so `^4.3.0` becomes `^4.4.4`. Consumers then cannot resolve the vulnerable version.

The same holds for a `peerDependencies` range of a published package when the advisory is in the peer itself. The first version that fixes it becomes the lower bound, so `^6.1.0` becomes `^6.4.3`, whatever form the range has. In a range that allows several majors, raise each part that allows a version the advisory covers. Leave a major that has no fixed version, and report it. Step 9 then widens the range by the major that has the fix.

Then check each override from the survey, also when the audit was clean from the start. Overrides from item 3 stay.

- Remove an override outside the root `package.json` and the root `pnpm-workspace.yaml`. npm and pnpm ignore it.
- Remove an override for a package that nothing in the tree depends on any more.
- When the audit reports the version an override forces, raise the override to the lowest version that fixes the advisory within the major it forces. Updating the parent cannot fix that. When only a newer major fixes it, leave the override and report it.
- Remove each other override, update the lockfile without installing, and compare the versions the package resolves to with and without it. Keep the override when a version drops or the audit reports a new advisory. When a kept override forces a major that the parent's range does not allow, record the parent version whose range does allow it, for the summary.
- Otherwise remove a fix and keep a hold-back, by the reason from the survey or the answer from step 2. An unchanged version does not show that a hold-back is no longer needed, because the lockfile keeps any version the parent's range allows until the next update.
- Run the install after the removals, then the baseline scripts. The lockfile-only update leaves `node_modules` on the overridden versions. When a script that passed now fails, put the removed overrides back one at a time until it passes. Keep the override whose return made it pass, remove the others again, and run the scripts once more. If they fail again, repeat with that override kept.

With pnpm, also update `packageManager` as the reference file's "Package manager version" section says.

Commit when done. The type is `fix` when an update closes an advisory in a package that ships: one an app installs outside `devDependencies`, or one a published package lists in `dependencies`, `optionalDependencies` or `peerDependencies`. Otherwise it is `chore`, also for a lockfile or override fix in a published package, because its consumers resolve their own versions. `fix` cuts a release in repos that release from commit types, and step 12 takes the PR title's type from this commit.

## 9. Majors

Attempt a dependency major only when an advisory cannot be fixed without it, whatever its severity and whether the package is a dev dependency. For every other package with a newer major, write one line for the summary with what the major changes for this repo, and leave it for the repo's update bot.

This step handles the majors of dependencies that step 8 left, and the Node and action majors that steps 6 and 7 recorded.

### Free majors

A major is free when all four hold:

- In a published package, the dependency does not reach the public API, by the check at the end of this step.
- The changelog or migration guide between the two versions removes or changes nothing the repo uses. Search the repo for each API, option and default it removes or changes, and check for a switch to ESM only.
- The new version's `engines.node` and peer dependencies are met without other changes.
- The install, build, typecheck and baseline scripts pass with no source edit.

For an action, the input and output check in step 7 is the free check.

### Targets

- For a direct dependency, the fixed major is the first one with a version outside the advisory's vulnerable range.
- For a transitive package, the fixed major is the first major of the direct dependency whose tree resolves to a fixed version. Read each major's newest release with `npm view <parent>@<version> dependencies`, and follow the chain down to the vulnerable package.
- The target is the newest release of the fixed major, not the registry's `latest`. Skip versions with a prerelease suffix such as `-rc`, `-beta` or `-alpha`.
- When no major fixes the advisory, leave it for the summary.

Install a target only after `npm view` shows that its tree resolves to a fixed version, as above. Upgrade to each dependency target, following the rules at the end of this step, and run the free check. Commit a free major on its own. Undo every other one, and write its brief.

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

These rules hold for free and approved majors alike:

- In a published package, check whether the dependency reaches the public API, meaning that the package re-exports its types or values. Record the result for the changeset.
- In `dependencies` or `optionalDependencies` of a published package, write the new version with the operator the entry has, such as `^5.1.0`.
- When the upgraded package is also a peer dependency of a published package, widen the peer range by the new major and change nothing else in it. Add a caret range with the version the lockfile resolves, the one the tests now run against: `^6.4.0` becomes `^6.4.0 || ^7.2.1`. The old major stays allowed although it has no fix, so that consumers can move on their own schedule. Record it for the summary.

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
- Compare the `peerDependencies` of each `package.json` with the merge base from the survey. Skip a `package.json` that does not exist at the merge base. The command prints nothing when they match:

  ```
  diff <(git show "<merge-base>:<path>" 2>/dev/null | jq -S '.peerDependencies // {}') <(jq -S '.peerDependencies // {}' "<path>")
  ```

  The only changes allowed are a lower bound raised by step 5 or for an advisory in step 8, and a range widened by step 9. Every entry must still be a range. Revert any other change in the commit that made it, and report it.
- Run the same comparison on `.dependencies` and `.optionalDependencies` of each published package. No entry may have lost its range.
- When a Dockerfile changed and Docker is available, build the image.
- When a workflow changed and `actionlint` is installed, run it. When it is not installed, skip it and leave it out of the summary.
- Trace each new failure to its commit. Fix it there, or undo that commit's change and record why.
- A fix or an undo from this step goes into the commit that caused it, never on top, so that a stack can still be cut at commit boundaries. Commit it with `git commit --fixup=<sha>`, then run `git rebase --autosquash <merge-base>`. The branch is not pushed, so rewriting it is safe.
- Record the audit counts again. These are the "after" numbers.
- Record `git rev-parse HEAD^{tree}` after the last verify run, after any fixup and autosquash. Ship skips its checks when it pushes this same tree.

## 11. Summary

In T3 Code, send the summary with `send_to_user` in parts of under 4000 characters, because the tool refuses a longer message.

Report in this order:

- The audit counts before and after, as a table with info, low, moderate, high and critical.
- Each advisory that is left, with the reason it could not be fixed. Add these details: the dependency chain, the newest version of the direct dependency and whether it still pulls in the vulnerable major, whether the package is a dev dependency or reaches production, and whether the repo calls the code the advisory names. When the direct dependency is deprecated on npm or has had no release in two years, also name a maintained replacement.
- Each open Dependabot alert from the survey that the run did not close, with the reason, or that alerts are off.
- Each updated package as `{name}: {old} to {new}`, under `dependencies` and `devDependencies`.
- Each raised or widened peer dependency range as `{name}: {old} to {new}`, and each peer major that stays allowed without a fix for an advisory.
- Each catalog entry left as a range because a published package uses it.
- Each changeset added, with its packages and bumps.
- Each package without a `private` field that the audit treated as an app.
- Node: the version before and after, and each file that changed.
- Workflows: each action as `{name}: {old} to {new}`, and each major not taken with its reason. The references left as they are because the repo's owner owns them.
- Code owners: the file added and its owners, or that the repo already had one, or that the user declined. The command that checks the file after the push. Leave this line out when a user owns the repo.
- Dependabot: added, updated or already correct, skipped because the repo uses Renovate, or declined by the user. When declined, say that nothing moves the exact versions and the SHA pins until the next audit. The groups and `ignore` entries not added because the config already covers them. An entry with `versioning-strategy: increase` in a repo with published packages. The peer hints. Where GitHub shows a config error after the merge.
- Majors: each free one done, each one approved or skipped in step 9, and each approved one undone with its reason.
- Overrides added, raised and removed, and each one kept with its reason. For a kept override that forces a major the parent's range does not allow, also give the parent version that would make it unnecessary.
- With pnpm: the `packageManager` version before and after, and each dependency build script the install did not run.
- Both test results from step 9, each with its count, its covered line and the names without tests or the gaps, the evidence for each touched item, and the line coverage per touched item when step 1 measured it. When the PR's changes are Partly covered or Not covered, or the repo's tests have gaps or little or none, up to 5 tests to add, picked from the agent's findings: code that calls a touched package first, then by risk. Each names the behavior, the file, and the lowest layer where a test would fail on the current code.
- What the user must test by hand because this run could not, including each runtime change without evidence.
- Each choice made without asking.
- The PR shape from step 12.

## 12. Ship

Count the change with `git diff --numstat <merge-base> HEAD`, leaving out lockfiles.

Pick a single PR when no major upgrade needed source changes and the count is under about 300 changed lines. Otherwise pick a stack, cut at commit boundaries in this order, bottom first:

1. code owners and Dependabot
2. pin and settings
3. Node
4. workflows
5. updates and fixes
6. one layer per major that changed source code

Fold a small layer into its neighbour. Say how many layers the stack has and why each one is separate.

Ask with AskUserQuestion whether to ship. Name the shape, and offer "Ship" first and "Don't ship yet". Stop when the user picks "Don't ship yet".

For a stack in a repo whose CI fails a PR without a changeset, give each layer that has none the empty changeset from [Changesets](#changesets) before the hand-over. Put it into that layer's last commit with the fixup and autosquash from step 10. `mstack:ship` does not know this rule, and would answer the failed check with a changeset that releases a package.

Then hand over to the `mstack:ship` skill. Give it the tree hash recorded in step 10, and these drafts for the PR:

- A title that leads with the outcome: how many `npm audit` or `pnpm audit` advisories the run closes, and the largest other change, such as `fix: close 11 advisories and move to node 26`. Its type is that of the updates commit from step 8, or `chore` when there is none.
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
