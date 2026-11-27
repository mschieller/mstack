# Audit: Go modules

The ecosystem part of the [audit skill](SKILL.md) for a repo with a `go.mod`. Each heading names the step of the skill it belongs to, and step numbers in the text are those of the skill.

## Rules

- Do each part once per `go.mod` in the repo, from the folder that holds it. With a `go.work` file, still work per module, and leave `go.work` and `go.work.sum` to the `go` command.
- Change `go.mod` with `go get` or `go mod edit`, never by hand, and run `go mod tidy` after each change. `go.mod` holds exact versions, so nothing needs pinning.
- When git tracks a `vendor` folder, run `go mod vendor` after each change to `go.mod`, and put the result in the same commit.
- The scan is `govulncheck`, which is not part of Go. Look for it on the `PATH`, then in `$(go env GOBIN)` when that is set, then in `$(go env GOPATH)/bin`, and run it by its full path when it is not on the `PATH`. When it is in none of them, ask the user with AskUserQuestion to install it with `go install golang.org/x/vuln/cmd/govulncheck@latest`, and skip the Go parts when they decline. Never install it yourself.
- `govulncheck` cannot scan a module that needs a newer Go than the one it was built with. `go version <path to govulncheck>` prints that Go. When it is older than the app target of step 6, ask the user in step 2 to build it again with `GOTOOLCHAIN=go<target> go install golang.org/x/vuln/cmd/govulncheck@latest`. When they decline, step 6 leaves the Go minor where it is, and the summary says why.
- When git tracks a `vendor` folder, add `-mod=mod` to every `go list -m` command in this file. Without it the command fails, because it cannot look up versions from the vendor folder.
- A module is a library when it has a package that is neither `main` nor under `internal/`, and the repo has a version tag for it, such as `v1.2.3` or `<folder>/v1.2.3`. Every other module is an app.
- When a download fails because a module is private, ask the user to fix the access, such as `GOPRIVATE` and their git login. Never ask for a token or offer to take one.

## The scan

Run the scan twice, and read the JSON, because with `-format json` the exit code is 0 whatever the scan finds. Any other exit code means the scan itself failed, for example because the packages do not load. A failed scan prints no findings, and its error output names the cause after the first line. Record it as not run, never as zero, and say so in the summary and in the PR:

```
govulncheck -format json ./...
govulncheck -format json -test ./...
```

The output is a stream of JSON objects. Each `finding` has an `osv` id, a `fixed_version` when a fix exists, and a `trace` whose first frame names the `module` and the `version` found. The first frame also gives the level:

- Called: the frame has a `function`. The code calls the vulnerable function.
- Imported: the frame has a `package` and no `function`. The code imports the package and calls nothing vulnerable in it.
- Required: the frame has neither. `go.mod` requires the module and the code imports nothing vulnerable from it.

One advisory can show up at several levels. Count it once per module, at its highest level. Take the counts and the modules to fix from the second scan, which holds everything the first one does. A finding is test only when it is called in the second scan and not in the first.

Neither scan sees a module that only test files import. `-test` adds the calls that tests make into modules the other code already imports, and nothing else. List the modules the scan cannot see by comparing these two outputs, and name them in the summary as not scanned:

```
go list -deps -f '{{with .Module}}{{.Path}}{{end}}' ./... | sort -u
go list -deps -test -f '{{with .Module}}{{.Path}}{{end}}' ./... | sort -u
``` With several `go.mod` files, keep the counts per `go.mod`, and give their sum where one number is asked for.

The module `stdlib` is the Go standard library, and its `version` is the Go release that ran the scan, which `go version` prints in that folder. A `stdlib` finding is fixed by step 6, not by `go get`. In a library, leave the `stdlib` findings out of every count, before and after. They follow the Go on the machine and say nothing about the module.

## Step 1: Survey

- Check that `govulncheck` is there, as the rules say.
- List every `go.mod`, whether git tracks a `vendor` folder next to it, and whether the repo has a `go.work`.
- Decide for each module whether it is a library or an app, as the rules say.
- Run `go mod download`. If it fails, add that to the questions in step 2, since every later step needs the modules.
- Run both scans. Record the count of findings per level, with the test-only ones named. These are the "before" numbers in the summary. Record the modules that only test files import.
- Record the modules with a newer version, from `go list -m -u -json all`. Each object has `Path`, `Version`, `Indirect` and, when a newer version exists, `Update.Version`. A newer major has another module path, such as `example.com/mod/v2`, so this command does not show it.
- List every `replace` and `exclude` line in `go.mod`. A `replace` to another module path or to a folder is a fork or a local copy. A `replace` of a module by the same path at another version is a pin. Find why each pin and each `exclude` was added, from a comment next to it or from the commit that added it, with `git log -S'<module>' --reverse -- go.mod`. Record the reason as a fix, as a hold-back, or as unknown.
- Find every place that sets a Go version: the `go` and `toolchain` lines in `go.mod`, `.go-version`, `.tool-versions`, `go-version` and `go-version-file` inputs in workflows, and every `FROM` in a Dockerfile that names a `golang` image.
- Find the supported Go releases. `https://go.dev/dl/?mode=json` lists the newest release of each supported minor, the newest minor first.
- Run the baseline: `go build ./...`, `go vet ./...` and `go test ./...`, and each check the repo's Makefile or CI workflow runs for Go, such as a linter. Record what already fails. This is the baseline for step 10. Skip a target that starts a server, deploys or writes outside the repo, and name it. Run `git status --short` again afterwards. A tracked file that was clean before and that a check changed goes back with `git restore <path>`.
- When the repo already measures coverage, through a Makefile target or a CI step with `-cover`, run the tests once with `go test -coverprofile=<scratchpad>/cover.out ./...` for the test grade in step 9.

## Step 2: Questions

- A pin or an `exclude` with an unknown reason. Offer to treat it as a fix, which step 8 removes when nothing needs it, or as a hold-back, which step 8 keeps. Recommend a fix.
- A `govulncheck` built with an older Go than the app target, as the rules say. Offer to wait while the user builds it again, or to go on without the move to a new Go minor. Recommend the rebuild.
- Baseline checks that fail before any change. Offer to continue with those failures noted, or to stop. Recommend continuing when the build passes, and stopping when it does not, because step 8 cannot tell a broken update from a build that was already broken.
- A `go mod download` that fails. Offer to stop. There is no other option.

## Step 4: Dependabot

- Add one `gomod` entry to `updates`. The snippet is indented two spaces more than in the file, as the example in the skill is:

  ```yaml
    - package-ecosystem: "gomod"
      directory: "/"
      schedule:
        interval: "weekly"
        day: "monday"
      groups:
        non-major:
          patterns: ["*"]
          update-types: ["minor", "patch"]
  ```

  `directory` is the folder with the `go.mod`. With several `go.mod` files, write `directories` with the list in place of `directory`, such as `directories: ["/", "/lib"]`.

- When the survey found a `golang` image and the repo has a `docker` entry, add this to that entry:

  ```yaml
      ignore:
        - dependency-name: "golang"
          update-types: ["version-update:semver-major", "version-update:semver-minor"]
  ```

  Step 6 moves Go to a new minor by hand, in every place at once. Dependabot cannot change the `go` line together with the image, so a minor it took would move the image alone.

## Step 5: Pin and settings

Nothing to do. `go.mod` holds exact versions, and Go has no setting for a minimum release age.

## Step 6: Runtime version

- For an app, the target is the newest release of the newest supported minor from the survey, such as `1.27.4`. Set it with `go mod edit -go=<version>`, and remove a `toolchain` line with `go mod edit -toolchain=none`, because the `go` line already selects that release.
- A move to a new minor must pass this check: `go build ./...`, `go vet ./...` and `go test ./...` pass with no source edit, every other baseline check that passed before still passes, and both scans still run. `go` downloads the release the `go` line names on its own. A scan can fail on a new minor until `govulncheck` supports it, also when it was built with that Go, and a module that cannot be scanned is worse off than one on the older minor. When the check fails, take the newest release of the minor the module is on, and record the minor and the reason for step 9. When that minor is no longer supported, record that too.
- For a library, the `go` line is the oldest Go its consumers may use, not the Go it builds with. Set it to the oldest supported minor from the survey with a zero patch, such as `1.26.0`. Leave a `go` line that is already at or above it. Add no `toolchain` line and leave one that is there, so the library can still be tested on its floor. Run the check from the bullet above once with `GOTOOLCHAIN=go<floor>` in the environment, which makes `go` use that release.
- Update every other place from the survey to the version the app's `go` line now holds, written exact, such as `go-version: "1.27.4"`. Leave a `go-version-file` input as it is, since it follows `go.mod`. In a Dockerfile, keep the registry and the image variant, so `golang:1.26-alpine` becomes `golang:1.27.4-alpine`. Confirm that the tag exists in its registry before you write it.
- Run `go mod tidy`, then the scan again. For an app, every `stdlib` finding whose `fixed_version` is at or below the target is gone. One that is left has no fix in a supported release. Report it.
- Commit.

## Step 8: Updates and fixes

Work through these in order and stop as soon as the scan is clean. Move a module only to fix a finding, and leave every other update to the repo's update bot.

1. For each module with a finding at any level, the target is the lowest version that clears every finding of that module: the highest `fixed_version` among them. A finding with no `fixed_version` has no fix on this module path. Find its advisory among the `osv` objects of the same output, by `id`. When its `affected` list names the module under a higher major path, such as `example.com/mod/v4`, with a `fixed` event, a new major clears it, and it goes to step 9. Otherwise report it as having no fix. Before the update, read the Go that the target needs with `go list -m -json <module>@<version>`, in `GoVersion`. `go get` raises the `go` line to it without asking. In a library that raises the floor for its consumers, so record it for the summary.
2. Run `go get <module>@<version>` for each one, also for a module that `go.mod` marks `// indirect`. Go has no overrides. A requirement on an indirect module does the same job. Then run `go mod tidy`.
3. A finding that only a new major clears, which is a new module path such as `/v2`, goes to step 9.

Then check each pin and each `exclude` from the survey, also when the scan was clean from the start.

- Never change a `replace` to another module path or to a folder. Report it.
- Remove a pin or an `exclude` for a module that `go mod why -m <module>` says the main module does not need.
- Keep a pin or an `exclude` whose reason is a hold-back, from the survey or the answer from step 2.
- Try to remove each one whose reason is a fix: `go mod edit -dropreplace=<module>` for a pin and `go mod edit -dropexclude=<module>@<version>` for an `exclude`, then `go mod tidy`, the baseline checks and both scans. Keep the removal when the checks that passed before still pass and the scan shows no new finding. Otherwise put the line back.

Commit when done. The type is `fix` when an update clears a finding that is called or imported outside test files, in a module that ships. Otherwise it is `chore`, also for a finding that only tests reach.

## Step 9: Majors

Attempt a major only for a finding that is called and that is not test only. For every other module with a newer major, write one line for the summary, and leave it for the repo's update bot. A newer major has another module path, so look for it by name: `go list -m -versions <module>/v<N+1>` for a path such as `example.com/mod/v2`, and `gopkg.in/<name>.v<N+1>` for a `gopkg.in` path. The command fails, or prints no versions, when that major does not exist.

### Free majors

No major of a Go module is free. A major has a new module path, so the upgrade rewrites every import of it, and each one gets a brief and a question.

A new Go minor that failed the check in step 6 gets a brief and a question too. Its brief holds the check that failed and the line of the error that names the cause. Recommend "Skip and report" when the scan is what failed, because the module could not be scanned after the move.

### Targets

- The fixed major is the lowest major path for which the advisory's `affected` list has a `fixed` event.
- The target is the newest release of the fixed major, from `go list -m -versions <module path of that major>`. Skip a version with a prerelease suffix.
- When no major clears the finding, leave it for the summary.

### Rules for every major

- In the brief, leave out the lines about peer dependencies. Go has none. "Reaches the public API" means that an exported function, type or field of a library names a type from the module. Go's database gives no severity. Take it from the GHSA id among the advisory's `aliases`, with `gh api /advisories/<GHSA id> --jq .severity`, and leave it out when there is none.
- For a module major, recommend the upgrade in the question, since the finding is one the code calls. Undo an upgrade when a check that passed before fails or a scan no longer runs.
- To upgrade, run `go get <new module path>@<version>`, rewrite each import of the old path in the `.go` files outside `vendor`, run `gofmt -l` on the files you changed, and run `go mod tidy`, which drops the old requirement once nothing imports it.
- Commit each major on its own, with the type `fix`.

### Test grade

The touched set is every module the run moved, direct or indirect, plus Go and each base image when their version changed. Group an indirect module under the direct module that requires it, from `go mod graph`. A module that only test files import is a dev tool. Evidence for Go itself is tests that can fail and that passed on the new release.

## Step 10: Verify

- Run both scans again. Record the count per level. These are the "after" numbers.
- Run `go mod verify`, and `go mod tidy` followed by `git diff --exit-code go.mod go.sum`. A diff means an earlier step left `go.mod` untidy. Fix it in the commit that caused it.
- When git tracks a `vendor` folder, run `go mod vendor` with a clean tree, followed by `git status --short vendor`. It prints nothing when the folder is current.

## Step 11: Summary

- The scan counts before and after, as a table with called, imported and required, and the test-only findings named. The modules that only test files import, as not scanned.
- Each finding that is left, with the reason it could not be fixed: no fixed version, only a new major that was not taken, or a fork or a local copy that holds the module back. Add the module that pulls it in, from `go mod why -m <module>`, and its level.
- Each updated module as `{module}: {old} to {new}`, with direct and indirect ones apart.
- Each `go` line that an update raised in a library, with the module that needed it.
- Go: the version before and after, and each file that changed. For a library, the `go` line before and after.
- Each pin and `exclude` removed, and each one kept with its reason. Each fork and local copy, reported and left.
- Each module with a newer version that the run left for the update bot, as a count, and each newer major by name.

## Step 12: Ship

Leave the `vendor` folder and `go.sum` out of the line count that picks a single PR or a stack.

In the Audit section of the PR body, the row and the table for Go are these. Write `not measured` in place of a count when a scan failed. With several ecosystems, put the name of each in front of its row, and give each its own table under "Advisories".

```markdown
| Go findings | <called before> called, <imported before> imported and <required before> required, to <the same after>. <how many are left open, and whether any is called>. |
```

```markdown
| | Called | Imported | Required |
|---|---:|---:|---:|
| Before | <n> | <n> | <n> |
| After | <n> | <n> | <n> |
```

In "Left open", the first cell is `<id>, <level>, <test only or production>`.
