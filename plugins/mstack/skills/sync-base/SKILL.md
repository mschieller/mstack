---
name: sync-base
description: Bring the current branch up to date with its base branch (main, master or the PR's base) by rebasing onto it, or by merging it in when a rebase would rewrite what others build on. Picks the strategy itself. Reads what landed on the base and why first, resolves conflicts with both intents understood, then hunts for semantic conflicts git cannot see and runs the repo's checks. Use when the user asks to rebase on main or master, rebase onto the base, merge main or master in, fix conflicts, update or sync the branch, or when the `mstack:ship` skill finds the branch behind.
argument-hint: "[rebase|merge] [base branch, overrides detection]"
---

# Sync base

Bring the base branch into the current branch after understanding what landed on the base since this branch left it, and why. The dangerous part is not the conflicts git reports. It is the changes that apply cleanly and are still wrong.

## Rules

- Rebase unless step 2 picks a merge. A rebase keeps the branch's history a straight line on top of the base, so sync merges never reach the base's history in repos that merge PRs with merge commits. A merge rewrites nothing, so it is the choice when other work builds on the branch.
- Know what the base's side of a conflict was for before resolving it. Read the commit that introduced it, not only the hunk.
- Never take one whole side (`-X ours`, `-X theirs`, `git checkout --ours/--theirs`) of a file where both sides made real edits. Whole-file side-taking is fine only for generated files and for a file one side deleted. A rebase swaps git's names: `--ours` is the base and `--theirs` is the branch's commit. This skill says "the base's version" and "the branch's version" instead.
- Never discard the branch's work to make a conflict go away, and never abort the merge or rebase and report failure. Resolve it, or stop with the merge or rebase still in progress and explain what blocks you. The autosquash pass in step 6 is the one exception, because it only reorders commits that already exist.
- Never stash. Worktrees and other sessions share one stash stack, so a pop can take another session's entry. Before a rebase, set uncommitted changes aside in temporary commits as step 4 describes, and never push while they exist.

## 1. Preflight

1. Find the base: the base branch from the arguments when one is given, else the PR's `baseRefName` from `gh pr view --json baseRefName`, else `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`. If the current branch is the base, there is nothing to sync.
2. If `gh stack view` lists the current branch, it is in a stack. Stop and use `gh stack sync`, as the stack mode of the `mstack:ship` skill describes.
3. Find the remote. It is `origin`, unless the repo has none or `origin` is a fork of the repo that `gh repo view --json nameWithOwner` names. Then use the remote in `git remote -v` that points to that repo, and stop if none does. The steps below write `origin` for it.
4. Run `git fetch origin --prune` and work against `origin/<base>`, never a local copy that may be stale.
5. Run the [remote branch check](#remote-branch-check), so that nothing below rewrites commits the local branch has not seen.
6. If `git merge-base --is-ancestor origin/<base> HEAD` succeeds, report "already up to date" and stop.

## 2. Choose the strategy

When the branch has no commits of its own, which `git merge-base --is-ancestor HEAD origin/<base>` shows by succeeding, fast-forward with `git merge --ff-only origin/<base>` whatever the strategy. Then run steps 6 and 7 against the uncommitted changes, and make no commit in step 8.

Otherwise, `rebase` or `merge` in the arguments decides. Without one, merge when either of these holds, and rebase when neither does:

- **The branch already holds a merge.** `git rev-list --merges origin/<base>..HEAD` lists a commit. A plain rebase would drop that merge and replay the conflicts it resolved.
- **Other open PRs build on the branch.** `gh pr list --base <branch> --state open --json number,url` returns a PR. A rebase would rewrite the commits they are based on.

An explicit `rebase` overrides both. Say in the divergence brief how many merges the rebase flattens, and name each PR that builds on the branch.

For a merge, run `git status --porcelain --untracked-files=no`. If it lists a file that `git diff --name-only HEAD...origin/<base>` also lists, ask whether to commit the work first or stop. Ask the same when any change is staged, which `git diff --cached --quiet` shows by failing, because git refuses to merge with a staged change. Other uncommitted changes stay as they are, and git carries them through the merge. Untracked files do not count, because git refuses a merge that would overwrite one.

## 3. Divergence brief

Build this and show it before rebasing or merging.

- The strategy and why step 2 picked it.
- The merge base: `git merge-base HEAD origin/<base>`.
- The overlap set: the files changed on both sides, from `git diff --name-only <merge-base>..HEAD` intersected with `git diff --name-only <merge-base>..origin/<base>`. For each file, what the base changed and why, what the branch changed and why, and whether the intents are compatible, redundant or exclusive.
- The incoming commits from `git log --oneline <merge-base>..origin/<base>`, one line each. Read diffs in depth only for commits that touch the overlap set.
- Two lists recorded while reading: symbols the base renamed or removed, and schema changes (columns, tables, constraints, enum values).
- A premise check. Did the base already build what this branch builds, or change a decision in the docs that this branch implements the old way? If so, say it plainly and ask before going on. Continuing may be wrong.

If nothing in the brief is surprising, go on right after showing it. Pause only for a failed premise check.

## 4. Rebase or merge

**Rebase.** Record `git rev-parse HEAD` as the pre-sync SHA for the report. `git rebase` refuses to start with uncommitted changes to tracked files, so set them aside first:

1. If `git diff --cached --quiet` fails, run `git commit --no-verify -m "sync-base: temporary, staged changes"`.
2. If `git diff --quiet` fails, run `git commit --no-verify -am "sync-base: temporary, unstaged changes"`.

`--no-verify` keeps commit hooks from formatting or rejecting work that is not ready. Untracked files stay where they are. Then run `git rebase origin/<base>`, resolve each stop by the rules in step 5, stage the files and continue with `GIT_EDITOR=true git rebase --continue`, so that no editor opens. The temporary commits are replayed last. A conflict in them is between the base and the user's uncommitted work, so resolve it with the same care.

**Merge.** Run `git merge --no-ff --no-commit origin/<base>` and resolve the conflicts by the rules in step 5. Without `--no-commit`, a clean merge commits at once with git's default message, before steps 5 and 6 have run.

## 5. Resolve conflicts

A rebase stops once for each commit that conflicts, so the same file can come back several times. Classify each conflict:

- **Lockfiles**, such as `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `bun.lock`, `Cargo.lock`, `go.sum` and `uv.lock`. Take the base's version, then reinstall so the branch's dependency changes are resolved again. In a rebase, do this at each commit where the lockfile conflicts. The reinstall keeps the base's versions wherever the manifests allow them, so it drops a change the branch made only in the lockfile, such as an audit fix or an update inside a range. Find those commits with `git log <merge-base>..<pre-sync SHA> -- <lockfile>`, run their command again after the reinstall, and confirm that the package manager's audit reports nothing the branch had fixed.
- **Generated files** (route trees, OpenAPI or GraphQL clients, ORM clients, build output, `*.tsbuildinfo`). Take either side, then run the generator. Never hand-merge them.
- **Migrations.** Never hand-merge SQL. Check the ordering even when they apply cleanly: a branch migration may now sort before a base migration it depends on. If it does, regenerate the branch's migration on top of the new schema with the repo's ORM command. If the base squashed or regenerated its migration chain, regenerate the branch's schema change on top of the new chain.
- **Docs, CLAUDE.md and ADRs.** A conflict here is two decisions, not two texts. Keep each decision in one home doc with its reason, and make the other docs point to it. Registers of independent items, such as decision lists or roadmaps, usually keep both sides' entries; check that neither entry is stale.
- **Everything else.** Keep both sides' intent. Take one side only when the intents are exclusive. When a conflict hides a real design choice that changes behavior, ask with AskUserQuestion: the cause, both intents, the options with the recommended one first. Resolve obvious conflicts yourself.

## 6. Semantic conflict hunt

Run this whether or not git reported conflicts. Start with the repo's typecheck; it finds most of checks 1, 2 and 6 in seconds. Then work through the list, limited to the overlap set and what the branch added or deleted:

1. **Renamed, moved or deleted, in both directions.** Grep the branch's added lines for every symbol on the renamed or removed list. Then grep the base's added lines for everything the branch deleted: files (`git diff --diff-filter=D --name-only <merge-base>..<pre-sync SHA>`), exports and package entry points. A typecheck can pass while a bundler or test runner still fails on a removed package export.
2. **Changed signatures and shapes.** New required parameters, narrowed types, changed return shapes and widened unions that the branch handles only in part.
3. **Schema against queries.** The schema changes from step 3 against the queries and inserts the branch wrote, plus the migration ordering from step 5.
4. **New invariants.** Guards, permission checks, lint rules, validation or required fields the base added that the branch's new code does not satisfy. Read the guard's own source to see what it matches before searching for violations.
5. **Work done twice.** Both sides solved the same problem. Keep one implementation and say which and why.
6. **Config and environment.** New env vars, config keys, service definitions and changed script bodies. If the base wrapped a tool in a script, branch code that still calls the tool directly now skips the wrapper.
7. **Fixtures, seeds and test helpers** whose shape the base changed. A branch test that passes against a stale fixture is a false green.
8. **Docs decisions against the branch's code.** Where the base changed a decision, fix the branch's code when it is a spot fix, and report it otherwise. Where the base moved a decision to another doc, grep the branch's comments and docs for pointers to the old place.

Each hit is fixed as part of the sync when it is plain fallout, or reported as follow-up work. Say which for each hit.

Where a fix goes depends on the strategy. After a merge, it goes into the merge commit. After a rebase, it goes into the branch commit that wrote the code it fixes, so that commit builds on its own in repos that keep PR commits. Record `git rev-parse HEAD` as the post-rebase SHA before the first fix. Stage each fix by path and run `git commit --fixup=<that commit>`. A fix that no single commit owns becomes an ordinary commit with a conventional title. When every fix is in and any is a fixup, fold them in with `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash origin/<base>`. The `-i` with a no-op editor works on every git version, while `--autosquash` without `-i` needs git 2.44. If the pass conflicts, run `git rebase --abort`, which loses nothing, then put every fix on top as one ordinary commit: `git reset --soft <post-rebase SHA>` and `git commit -m "<conventional title>"`. Either way, `git diff --quiet <tip before the pass> HEAD` must succeed, because neither path changes the tree.

When `mstack:ship` runs this step on a stack, its stack mode decides where the fixes go.

## 7. Verify

Run the repo's checks in order of cost, as its CLAUDE.md, AGENTS.md or package scripts name them: typecheck, lint, format, then the tests for every package the sync touched. Run database or migration tests when anything under the schema or migrations changed. Fix what the sync broke, as step 6 describes. After a rebase, check the final commit only, and say in the report that the commits below it were not built.

When a check fails for a reason unrelated to the sync, check whether it fails on the base too, and report it as such.

## 8. Finish and report

**After a merge**, stage the resolutions and the fixes from step 6 by path, never with `git add -A`, so that changes the user left uncommitted stay out. Commit the merge with `git commit -m`. The title is `Merge origin/<base> into <branch>`, because commit linters skip a title in that form. The body says what came in and how the non-obvious conflicts were resolved.

**After a rebase**, restore the changes set aside in step 4. First record the SHAs of the temporary commits from `git log --format='%H %s' --grep '^sync-base: temporary'`, because they are the only copy of the user's uncommitted work.

- When the temporary commits are the top commits, undo the unstaged one with `git reset --mixed HEAD~1` and then the staged one with `git reset --soft HEAD~1`. Leave out the step for a commit that was not made.
- When an ordinary fix commit from step 6 sits above them, take them out of the branch with `git rebase --onto <first temporary commit>~1 <last temporary commit>`. Then apply the staged one with `git diff --binary <its parent> <it> | git apply --index` and the unstaged one with `git diff --binary <its parent> <it> | git apply`. If an apply fails, stop and give the user both SHAs. The work is safe in those commits, but no branch points at them.

Then confirm with `git status --short` that the tree shows what it showed before step 4.

Report:

- the strategy and why
- what landed from the base, summarized
- every conflict and how it was resolved, with the intent kept from each side
- every semantic hit, fixed or reported as follow-up, and for a rebase which commit each fix went into
- check results, including anything still failing and why
- after a rebase, the pre-sync SHA, which `git reset --hard <SHA>` returns to, and that the push needs a lease

Run on its own, ask before pushing. When the `mstack:ship` skill called this skill, return to `mstack:ship` without pushing. That skill owns the push. Either way, push as [Push](#push) describes.

## Remote branch check

`mstack:ship` runs this check too, before it pushes. It finds commits on `origin/<branch>` that the local branch lacks, such as a formatting bot's commit or a merge from GitHub's "Update branch" button. Run it after a fetch, and skip it when `origin/<branch>` does not exist.

1. If `git merge-base --is-ancestor origin/<branch> HEAD` succeeds, the remote holds nothing new.
2. If `git merge-base --is-ancestor HEAD origin/<branch>` succeeds, the local branch is only behind. Run `git merge --ff-only origin/<branch>`. If git refuses because of uncommitted changes, ask whether to commit the work first or stop.
3. Otherwise the two have diverged. Build two lists. The unseen commits are the remote's commits that the local branch never held, found through the branch's reflog. The missing commits are the remote's commits that `HEAD` lacks.

   ```
   git rev-list --reverse origin/<branch> --not origin/<base> $(git log -g --format=%H refs/heads/<branch>)
   git rev-list origin/<branch> --not origin/<base> HEAD
   ```

   Keep merge commits in both lists. A merge that exists only on the remote is exactly what the check must find.

   - **No unseen commits.** The local branch held every remote commit and rewrote them, as a rebase or amend does. Go on.
   - **Every missing commit is unseen.** The local branch rewrote nothing, and the remote only gained commits, as `git pull` expects. Bring them in without asking. Run `git merge origin/<branch>` when an unseen commit is a merge or the branch holds a merge as step 2 defines it. Otherwise set uncommitted changes aside as step 4 describes, run `git rebase origin/<branch>`, and restore them as step 8 describes.
   - **Some missing commits are unseen and some are not.** The local branch rewrote commits and the remote gained new ones. Show the unseen ones with `git log --oneline --no-walk <commits>` and ask whether to bring them in or stop. To bring them in, cherry-pick them in that order. If one is a merge, stop instead and say so, because a cherry-pick cannot carry a merge.

Record `git rev-parse origin/<branch>` as the checked remote tip, and which of these cases applied.

## Push

- When the checked remote tip is an ancestor of `HEAD`, or there is no remote branch, push with `git push -u origin HEAD`.
- When the check found the local branch had rewritten the remote's commits, push with `git push --force-with-lease --force-if-includes origin HEAD`. The lease refuses if the remote moved since the fetch, and `--force-if-includes` refuses if the local branch never held the remote tip.
- After the check cherry-picked unseen commits, push with `git push --force-with-lease=<branch>:<checked remote tip> origin HEAD`. `--force-if-includes` would refuse here, because a cherry-pick makes new commits and the remote tip never enters the reflog. The lease names the exact tip the check inspected, so the push still refuses if anyone pushed after it.

When a push is refused, fetch and run the remote branch check again. Never push while a `sync-base: temporary` commit or a `fixup!` commit is on the branch.
