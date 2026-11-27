# Audit: Node packages

The ecosystem part of the [audit skill](SKILL.md) for a repo with a `package.json`. Each heading names the step of the skill it belongs to, and step numbers in the text are those of the skill. [PNPM.md](PNPM.md) and [NPM.md](NPM.md) hold the commands for each package manager.

## Rules

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

## Step 1: Survey

- Find the package manager from the `packageManager` field and the lockfile. Read [PNPM.md](PNPM.md) for `pnpm-lock.yaml` and [NPM.md](NPM.md) for `package-lock.json`. With any other package manager, or none, skip steps 5 and 8 and the dependency majors in step 9, skip the install, audit, outdated and override items in this step and in step 10, leave the counts table out of the summary, and say so there.
- List every `package.json` in the repo and whether the repo is a workspace.
- Decide which packages are published. A package is published when it has no `"private": true` and either the registry has it or the repo publishes it. The registry has it when `npm view <name> name` succeeds and either the name is scoped, such as `@scope/name`, or `npm view <name> repository.url` names this repo. An unscoped name can belong to someone else's package, as `app` and `server` do. A restricted package needs the user's npm login, and without it `npm view` answers 404. The repo publishes it when a workflow or script runs `npm publish`, `pnpm publish` or `changeset publish`, or the package has a `publishConfig`. Every other package is an app.
- Run the frozen install. If it fails because the lockfile does not match `package.json`, add that to the questions in step 2, since every later step needs the install.
- When the install fails with a 401 or a 404 on a scoped package, run `npm whoami`. A 401 there means the npm login has expired. Ask the user with AskUserQuestion to run `npm login`, then run the install again. Never ask for a token or offer to take one.
- Record the audit counts for info, low, moderate, high and critical. These are the "before" numbers in the summary.
- Record the packages with a newer major, for the summary.
- List every override in the places the reference file names. Find why each one was added, from a comment next to it or from the commit that added it and that commit's PR. `git log -S'<name>' --reverse -- <file>` lists the commits that changed the name, oldest first. Record the reason as a fix, when the forced version fixes an advisory or a bug, or as a hold-back, when a newer version broke the repo. Record it as unknown when nothing says.
- Find every place that sets a Node version: `engines.node`, `.node-version`, `.nvmrc`, `.tool-versions`, `volta`, `node-version` inputs in workflows, every `FROM` in a Dockerfile, `@types/node` and a `@tsconfig/node<major>` base.
- Find the active LTS line of Node. In `https://raw.githubusercontent.com/nodejs/Release/main/schedule.json` it is the line whose `lts` date has passed and whose `maintenance` date has not. Its newest release is the first entry in `https://nodejs.org/dist/index.json` whose `lts` field holds that line's codename.
- When the repo has a `.changeset` folder, find the release tool, `@changesets/cli` and its major in `devDependencies` or pnpm's own `pnpm change`, and with Changesets read `ignore` and `privatePackages` from `.changeset/config.json`. Check the CI workflows for a step that fails a PR without a changeset.
- Run every script in each `package.json` that tests, lints, typechecks or builds. Record what already fails. This is the baseline for step 10. A `test` script that is npm's placeholder, `echo "Error: no test specified" && exit 1`, is a known failure, recorded without a question. Run `git status --short` again afterwards. A tracked file that was clean before and that a script changed goes back with `git restore <path>`, so it stays out of the first commit.
- When the repo already measures coverage, through a coverage script or a coverage setting in the test runner's config, run the tests once with coverage and copy the report to the scratchpad for the test grade in step 9. Install no coverage provider. This run is not part of the baseline, so step 10 does not repeat it. Then delete the coverage folder when git does not ignore it and it did not exist before, so it stays out of every commit.

## Step 2: Questions

- An override with an unknown reason. Offer to treat it as a fix, which step 8 removes when nothing needs it, or as a hold-back, which step 8 keeps. Recommend a fix. The baseline scripts in step 8 catch many wrong removals, while a wrongly kept override stays until someone checks it by hand.
- Tests that fail before any change, other than npm's placeholder script. Offer to continue with those failures noted, or to stop.
- A frozen install that fails. Offer to update the lockfile in a first commit of its own, or to stop. Recommend the update, because the repo's CI fails the same way until someone makes it.

## Step 4: Dependabot

- With pnpm or npm, add this entry to `updates`:

  ```yaml
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
  ```

  `npm` is the ecosystem for npm and for pnpm. Its `directory` is the folder that holds the lockfile. Leave the entry out with any other package manager.

- When the survey found a Node version, add this to the `docker` entry:

  ```yaml
      ignore:
        - dependency-name: "node"
          update-types: ["version-update:semver-major"]
  ```

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

- In a `dependabot.yml` the repo already has, add the family groups to its `npm` entry. Skip a family when a group already matches its main package on majors.

- With pnpm or npm, collect peer hints for the summary: pairs of direct dependencies whose majors may need to move together but that no group covers. Read `peerDependencies` and `peerDependenciesMeta` from the installed `package.json` of each direct dependency, in the `node_modules` next to each `package.json` in the repo. A pair counts when the peer is a direct dependency of the same `package.json` and `peerDependenciesMeta` does not mark it optional. Drop a pair with a `@types/` package, since types follow their library's major. Drop a pair whose two packages share a family above. Write one line per peer with the packages that need it. Write no group for them.

## Step 5: Pin and settings

- Apply the settings from the reference file.
- Write every `devDependencies` entry as the exact version the lockfile resolves today, and in an app every `dependencies` and `optionalDependencies` entry too. Do not strip the `^` or `~` from the range. That gives the lowest version the range allows, which can be older than the installed one.
- In a published package, leave `dependencies` and `optionalDependencies` as they are.
- Leave entries that are not registry versions as they are: `workspace:`, `catalog:`, `file:`, `link:`, git and URL entries.
- Leave `peerDependencies` as they are, also for a package that `devDependencies` lists too. The next two bullets give the one exception.
- In a published package from the survey, raise the lower bound of a peer dependency range when two things hold. The range covers exactly one whole major, 1 or higher, such as `^6`, `6.x`, `^6.0.0` or `>=6 <7`. And `devDependencies` lists the same package. Write the version that entry resolves to today as a caret range, such as `^6.4.0`. A bare `^6` promises that `6.0.0` works, but the package's own tests only ran against the `devDependencies` version.
- Leave every other peer range, so that no upper bound moves and later audits do not raise it again: a range below major 1, a tilde range, and a range that allows more than one major, such as `^5 || ^6` or `>=6`. Leave a peer that `devDependencies` does not list, because the lockfile then holds whichever release the package manager picked. Leave a range that the resolved version falls outside, and report it.
- Update the lockfile without installing. Its resolved versions must not change in this commit.
- Commit.

## Step 6: Runtime version

- The target is the newest release of the active LTS line from the survey. A move to a new major must pass the free check in step 9, with the target Node on the `PATH`. When the target Node cannot be put on the `PATH`, the major is not free. When the check fails, take the newest release of the current line instead, and record the major for step 9.
- Update every place from the survey. Write the exact target version, such as `26.13.1`, also in place of a range such as `^26` or `>=24.11.0 <25`, and also when the place already names the target's major. The exception is `engines.node` in a published package, which the bullet below sets. An exact version makes a laptop, CI and the image run the same Node.
- When the survey found no place that sets a Node version for an app, not counting `@types/node` and a `@tsconfig/node<major>` base, add `engines.node` to the `package.json` next to the lockfile, with the target as an exact version, such as `26.13.1`. Add it to an app in no other case. One place is enough, and each extra place has to be kept in step with the others.
- In a published package, `engines.node` is the oldest Node its consumers may run, not the Node it builds with. Set it to `>=<major>` of the active LTS line, such as `>=26`. Add it when it is missing, and replace a lower bound whatever form it has, such as `>=18` or `>=22.12`. Leave a bound at or above that major as it is.
- In a Dockerfile, keep the registry and the image variant and write the exact target version, so `node:26-slim` becomes `node:26.13.1-slim`. Confirm that the tag exists in its registry before you write it.
- Set `@types/node` to the newest version with the same major as Node. In a published package, `@types/node` and a `@tsconfig/node<major>` base follow the major of the lower bound in its `engines.node` instead. Types and a compile target for a newer Node let code through that the lowest Node it allows cannot run.
- Commit.

## Step 8: Updates and fixes

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

## Step 9: Majors

Attempt a dependency major only when an advisory cannot be fixed without it, whatever its severity and whether the package is a dev dependency. For every other package with a newer major, write one line for the summary with what the major changes for this repo, and leave it for the repo's update bot.

### Free majors

A major is free when all four hold:

- In a published package, the dependency does not reach the public API, by the check at the end of this step.
- The changelog or migration guide between the two versions removes or changes nothing the repo uses. Search the repo for each API, option and default it removes or changes, and check for a switch to ESM only.
- The new version's `engines.node` and peer dependencies are met without other changes.
- The install, build, typecheck and baseline scripts pass with no source edit.

### Targets

- For a direct dependency, the fixed major is the first one with a version outside the advisory's vulnerable range.
- For a transitive package, the fixed major is the first major of the direct dependency whose tree resolves to a fixed version. Read each major's newest release with `npm view <parent>@<version> dependencies`, and follow the chain down to the vulnerable package.
- The target is the newest release of the fixed major, not the registry's `latest`. Skip versions with a prerelease suffix such as `-rc`, `-beta` or `-alpha`.
- When no major fixes the advisory, leave it for the summary.

Install a target only after `npm view` shows that its tree resolves to a fixed version, as above. Upgrade to each dependency target, following the rules at the end of this step, and run the free check. Commit a free major on its own. Undo every other one, and write its brief.

### Rules for every major

These rules hold for free and approved majors alike:

- In a published package, check whether the dependency reaches the public API, meaning that the package re-exports its types or values. Record the result for the changeset.
- In `dependencies` or `optionalDependencies` of a published package, write the new version with the operator the entry has, such as `^5.1.0`.
- When the upgraded package is also a peer dependency of a published package, widen the peer range by the new major and change nothing else in it. Add a caret range with the version the lockfile resolves, the one the tests now run against: `^6.4.0` becomes `^6.4.0 || ^7.2.1`. The old major stays allowed although it has no fix, so that consumers can move on their own schedule. Record it for the summary.

## Step 10: Verify

- Compare the `peerDependencies` of each `package.json` with the merge base from the survey. Skip a `package.json` that does not exist at the merge base. The command prints nothing when they match:

  ```
  diff <(git show "<merge-base>:<path>" 2>/dev/null | jq -S '.peerDependencies // {}') <(jq -S '.peerDependencies // {}' "<path>")
  ```

  The only changes allowed are a lower bound raised by step 5 or for an advisory in step 8, and a range widened by step 9. Every entry must still be a range. Revert any other change in the commit that made it, and report it.
- Run the same comparison on `.dependencies` and `.optionalDependencies` of each published package. No entry may have lost its range.

## Step 11: Summary

- The audit counts before and after, as a table with info, low, moderate, high and critical.
- Each advisory that is left, with the reason it could not be fixed. Add these details: the dependency chain, the newest version of the direct dependency and whether it still pulls in the vulnerable major, whether the package is a dev dependency or reaches production, and whether the repo calls the code the advisory names. When the direct dependency is deprecated on npm or has had no release in two years, also name a maintained replacement.
- Each updated package as `{name}: {old} to {new}`, under `dependencies` and `devDependencies`.
- Each raised or widened peer dependency range as `{name}: {old} to {new}`, and each peer major that stays allowed without a fix for an advisory.
- Each catalog entry left as a range because a published package uses it.
- Each changeset added, with its packages and bumps.
- Each package without a `private` field that the audit treated as an app.
- Node: the version before and after, and each file that changed.
- Overrides added, raised and removed, and each one kept with its reason. For a kept override that forces a major the parent's range does not allow, also give the parent version that would make it unnecessary.
- With pnpm: the `packageManager` version before and after, and each dependency build script the install did not run.

## Step 12: Ship

For a stack in a repo whose CI fails a PR without a changeset, give each layer that has none the empty changeset from [Changesets](#changesets) before the hand-over. Put it into that layer's last commit with the fixup and autosquash from step 10. `mstack:ship` does not know this rule, and would answer the failed check with a changeset that releases a package.
