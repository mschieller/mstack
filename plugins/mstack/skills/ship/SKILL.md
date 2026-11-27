---
name: ship
description: Take finished work to a pull request with green CI. Runs the checks, commits, opens or updates the PR with a concise body, watches CI, fixes failures and the real review findings that are already there, and stops when CI is green. Opens a stack of PRs with `gh stack` when asked for one. Never waits for a reviewer. Never merges unless told. Use when the user says "ship", "create the PR", "push and create the PR", "commit and push", "ship as a stack", "watch the CI" or "CI errored".
argument-hint: "[issue number or extra context]"
---

<!-- PR body and diagram guidance adapted from mattpocock/skills pr (MIT, see LICENSE), which credits Dex Horthy's show-me skill. Review and CI rules adapted from t3dotgg/tstack file-pr and babysit-pr (MIT, see LICENSE). -->

# Ship

Own the work from the current working tree to a PR with green CI. The user merges.

Stop at the step the request names. "Commit" stops after step 3. "Commit and push" or "push the changes" stops after the push in step 4, without opening a PR. "Watch the CI" or "CI errored" starts at step 5 on the existing PR. "Ship", "create the PR" or "push and create the PR" runs every step. A request for a stack, or a layer plan from another skill, runs the steps as [Stack mode](#stack-mode) changes them. A later change to the branch of an open PR, such as a fix the user asks for after ship stopped, runs ship again from step 2, so the change gets the checks, the title check, the body update and a watch.

## 1. Preflight

- Run `git status --short`. If it shows changes unrelated to the task, ask which to include. Never stash or discard the user's changes.
- Find the base branch: the PR's `baseRefName` if a PR exists, otherwise `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`. Never assume `main`.
- Name the branch `<type>/<slug>`, such as `fix/login-redirect-loop`. The type is a conventional commit type (`feat`, `fix`, `chore`, `docs`, `style`, `refactor`, `test`, `perf`, `ci`, `build` or `revert`). The slug is kebab-case, at most five words, and says what the change does.
- On the base branch or a detached HEAD, create the branch with `git switch -c <name>`. On any other branch, look up its PR with `gh pr view --json state`. If the PR is `OPEN` or `CLOSED`, keep the name, because the PR is tied to it. If the branch has an upstream, so `git rev-parse --abbrev-ref '@{upstream}'` succeeds, keep the name and never delete it from the remote, because other people may use it. When there is no PR or it is `MERGED`, and the branch has no upstream, check the name:
  - A name that starts with a type and a slash stays as it is, also when a better slug exists.
  - Any other name, such as `t3/96c3b1cf` or `login-fix`, gets a new one in the form above, with `git branch -m <name>`.

## 2. Checks

Run what CI runs, limited to the touched files where the repo allows it: format, lint, typecheck and the tests next to the changes. Find the commands in the repo's CLAUDE.md, AGENTS.md, package scripts or CI workflow. Fix failures before committing. Do not run the full test suite unless CI only offers that.

When the caller hands over the tree hash its checks ran on, `git rev-parse HEAD^{tree}` prints the same hash, and `git status --short` shows nothing to commit, skip the checks and say so in the report. Run them in every other case.

## 3. Commit

Write in English everything ship writes: commit titles and bodies, and in the later steps the PR title, the PR body, review replies and changeset text. This holds when the repo's log, its PR template or a title the caller suggests uses another language.

Group the changes into logical commits. Write every commit as a conventional commit:

- The title is `<type>: <summary>` or `<type>(<scope>): <summary>`, with a type from the list in step 1. It is at most 72 characters.
- Pick the type by what the change does for the people who use the code, because release tools build the changelog and the next version from it. `feat` adds behavior they can use. `fix` repairs behavior that was wrong for them. A change they cannot observe is `refactor`, `perf`, `test`, `docs`, `style`, `build`, `ci` or `chore`.
- The summary starts with a lowercase word and is in the imperative, with no period at the end. Names inside it keep their case, such as Node, Renovate and GitHub.
- Use a scope only when the repo's log already uses scopes, with the names it uses. `git log --oneline -20` shows them.
- A breaking change gets `!` before the colon and a footer `BREAKING CHANGE: <what breaks and what a user must do>`. Some release tools read only one of the two.
- A revert is `revert: <what is undone>`, a summary in your own words that follows the same rules, with `This reverts commit <sha>.` in the body.
- The body follows one blank line after the title and states what changed and why. Footers such as `BREAKING CHANGE:` and `Refs #<issue>` follow one blank line after the body.
- Add no attribution trailer or "generated with" line.

The repo's own commit rules win over this list for the types, scopes, lengths and footers they allow, but never for the language. Take them from the commit rules in the repo's CLAUDE.md or AGENTS.md, which may also name footers, and from a commitlint config, a `commitlint` key in `package.json`, and a workflow that checks the PR title. This command lists the commitlint configs and the PR templates in the repo. It uses no shell glob, because zsh aborts a command whose glob matches nothing:

```
git ls-files | grep -iE '(^|/)(commitlint\.config\.[a-z]+|\.commitlintrc[^/]*|pull_request_template[^/]*|PULL_REQUEST_TEMPLATE/.*)$'
```

It exits with code 1 when it finds nothing, so run it on its own and not before `&&`.

Check each title right after the commit, and amend the commit when it breaks the form. A commit that is not in English breaks it too. To reword a commit that is not pushed and is not the last one, write the new message to a file whose first line is `amend! <old subject>`, followed by a blank line and the new message, then run:

```
GIT_EDITOR="cp <file>" git commit --fixup=reword:<sha>
GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash --autostash "$(git merge-base origin/<base> HEAD)"
```

`GIT_SEQUENCE_EDITOR=:` accepts the todo list as it is, so no editor opens. `--autostash` sets the changes left out in step 1 aside and puts them back, because a rebase refuses to start while they are in the tree. This does not break the rule against stashing. Git keeps the entry in the rebase's own state and applies it back by SHA, and puts it on the stash stack only when that fails. A reword changes no file, so it cannot fail here. The committed tree stays the same. Never reword with `git filter-branch`, `git commit-tree` or `git reset --hard`.

Before a push, list every title on the branch that breaks the form. A title left as `amend! ...` means the subject in the file did not match:

```
git log --format=%s origin/<base>..HEAD | awk '!/^Merge / && (length > 72 || !/^(feat|fix|chore|docs|style|refactor|test|perf|ci|build|revert)(\([^)]+\))?!?: [^A-Z].*[^.]$/)'
```

No output means every title passes. A title that starts with `Merge ` is a merge commit and is left out. The command cannot see the language, so read the titles as well. In a repo with commitlint, run `npx commitlint --from origin/<base> --to HEAD` in place of this command. Report a failing commit that was there before this run or that is already pushed, and do not rewrite it.

## 4. Open or update the PR

Run `git fetch origin`, or fetch the remote that the Preflight step of `mstack:sync-base` names when `origin` is not the PR's repo. If the branch is behind the base, bring it up to date with the `mstack:sync-base` skill before pushing. The sync runs here, after the commit, so that only the changes you left out in step 1 are uncommitted, and `mstack:sync-base` sets those aside or carries them through itself. When the branch is not behind, run the remote branch check of `mstack:sync-base` instead, because the remote copy of the branch may hold commits the local one lacks.

Check for an existing PR first: `gh pr view --json number,url,state,body`. The command also returns merged and closed PRs. If an open one exists, push, and update the body when the scope has changed. If none exists or the PR is `MERGED`, push and open a real PR, not a draft, so review bots run. If the PR is `CLOSED`, ask whether to reopen it or open a new one.

Push as the Push section of `mstack:sync-base` describes: a plain push when the remote copy of the branch is behind the local one, and a push with a lease after a rebase or amend. Run the checks from step 2 again when the remote branch check brought commits in. Outside these cases and stack mode, never force-push.

Title: a conventional commit title in the form from step 3. Its type is that of the PR's most significant commit: a breaking change first, then `feat`, then `fix`, then the rest. A squash merge makes it the commit title on the base branch. Say why the change matters, not which mechanism changed. A title the caller suggests is a draft: apply these rules to it, and say in the report what you changed. Check the PR title with the command from step 3, with `printf '%s\n' "<title>"` in place of `git log`. Double quotes keep an apostrophe in the title, such as in "don't", from ending the string.

Body, written with the `mstack:unslop` skill and checked for secrets and private details before posting:

```markdown
> [!WARNING]
> <merge danger, only for one-way doors: migrations, data deletion, public API removals, infra changes. Say what cannot be undone and what it affects.>

## Summary

<the problem and its cause, then what this PR does about it, in one to four sentences>

<optional visual, see below>

## Changes

- <one bullet per change a reviewer needs to know: what changed, the old and new version when there is one, and why>

**After the merge**

- <what someone has to do or check once this merges, only when there is something>

## Verified

<one or two lines: the checks, tests or manual runs that show it works. Then what was not run, if anything.>

<a section the caller hands over>

Closes #<issue>
```

Write the Changes bullets as one flat list. Group them under bold labels only when there are five or more, not counting After the merge, and they fall in at least two of these areas: app code, tests, dependencies, Node and images, CI, deploy and platform config, and docs. Name each label after its area. Every group holds at least two bullets, and a lone bullet joins the group closest to it. When something has to be done or checked after the merge, it goes under the After the merge label, also below a flat list, because it lists things to do, not changes. Leave it out when there is nothing.

When the change makes a choice reviewers may question, such as where code lives, a new dependency or an API shape, its Changes bullet states the choice, why, and the alternative you would accept. Ask an open question only when the answer depends on something only the reviewers know.

A title, Summary or Changes the caller drafts, such as `mstack:audit` does, is a draft. Keep its facts, labels and reasons, and apply this format and the title rules to it.

When the caller hands over a section for the body, such as the `## Audit` section from `mstack:audit`, copy it word for word. It goes at the end of the body, after `## Verified` and before `Closes #<issue>`. In a stack it goes on the layer the caller names.

When the repo has a PR template, fill it in and do not use the format above. The command from step 3 finds it. GitHub reads `pull_request_template.md`, in any letter case, from `.github/`, the repo root and `docs/`. When `.github/PULL_REQUEST_TEMPLATE/` holds several templates, take the one that fits the change. Keep the template's sections and checklists in their order, write their headings and items in English, and tick only the boxes that are true. Put the summary, the changes, the verification and the visual into the sections that ask for them, each as short as the format above makes it. When the template has no section for the changes, they follow the summary in its section, with their group labels and without the `## Changes` heading. A section the caller hands over goes at the end, after every template section. When the template has no place for the rest, add the merge danger block above its first heading and `Closes #<issue>` at the end. Remove the template's instruction comments and leave no placeholder text.

Add a visual only when it shows the reviewer something the bullets cannot: how files moved, a new call path, a state change, or how components interact. Pick the smallest form that makes the point:

- a `diff` sketch of a file tree, a call tree or a component tree when the change adds, moves or removes parts of an existing shape
- pseudocode for a changed algorithm or control flow
- a Mermaid sequence or flow diagram when several services or components talk to each other
- a before and after screenshot for visual changes, when you can upload one

Keep only the files, calls and states the reviewer needs. One visual is normal. Most small PRs need none.

In T3 Code, `pull_request_create` links the PR to the thread. After `gh pr create`, link it with `link_pull_request`.

## 5. Watch

First act on what is already there, as step 6 describes: failed checks on the head commit, and comments nobody has answered. A watch reports only what happens after it starts. A repo without CI has nothing to wait for: when the head commit has no workflow file under `.github/workflows` and `gh pr checks <n>` reports no checks, go to step 7. `gh pr checks` alone does not show that, because a PR opened seconds ago reports no checks either.

Then, in T3 Code, call `watch_pull_request` and end your turn. T3 wakes you when a check fails, checks pass, someone comments or reviews, or the branch conflicts. Outside T3, run `gh pr checks <n> --watch --fail-fast > /dev/null 2>&1` as a background command. It exits when the checks finish, with a non-zero code when one failed. Then list the comments once, because this command never reports them. Never write `sleep` or polling loops.

When a wake needs no action, end the turn without a message.

## 6. On each wake

Act on failed checks for the head commit, and on every review thread or PR comment that is unresolved and that you have not answered. Ignore a failure on an older commit.

List the conversation comments and reviews with `gh pr view <n> --json comments,reviews`. `--comments` and `--json` do not combine, and neither shows review threads. List the unresolved ones with:

```
gh api graphql -F owner='{owner}' -F repo='{repo}' -F number=<n> -f query='
query($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      reviewThreads(first: 100) {
        nodes { id isResolved path line comments(first: 20) { nodes { author { login } body } } }
      }
    }
  }
}' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved | not)'
```

Resolve a thread with its `id`:

```
gh api graphql -f threadId=<id> -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: {threadId: $threadId}) { thread { isResolved } }
}'
```

Reply inside a thread with the same `id`. `gh pr comment` posts to the conversation instead, outside the thread:

```
gh api graphql -f threadId=<id> -F body=@<file> -f query='
mutation($threadId: ID!, $body: String!) {
  addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: $threadId, body: $body}) { comment { url } }
}'
```

- **Failed check.** Read the failed log with `gh run view <run-id> --log-failed`. Decide whether it is a real failure or an infrastructure flake. A flake is a runner that died, a network error or a rate limit. A test that fails now and then is a real failure. When the same test also fails on the base branch, report it as a flaky test this PR did not cause, and leave its fix to a PR of its own. Fix real failures, run the checks from step 2, commit and push. Re-run a flake once with `gh run rerun <run-id> --failed`; if it fails again, treat it as real.
- **Conflict or branch behind.** Use the `mstack:sync-base` skill, run the checks, and push as step 4 describes.
- **Comment from `changeset-bot`.** Do not reply. It says whether the PR has a changeset. Act only when it finds no changeset and the PR needs one. The REST API adds `[bot]` to the login, so it reads `changeset-bot[bot]` there.
- **Review comment from a bot.** Verify the finding against the code. When it is real, fix it, and after the push reply in the thread with the commit SHA and resolve the thread. When it is not, or when it asks for work outside the PR's goal, reply with a one-line reason and resolve the thread.
- **Review comment from a person.** Verify it against the code. When it is real, fix it and draft a reply that names the commit SHA. Before posting any reply, show the user the draft and post only after approval. Leave resolving the thread to the person.

Keep the PR at the size its goal needs. Decline extra tests, guards for cases that do not matter, and refactors the goal did not ask for, and list them as follow-ups in the report.

## 7. Stop

Ship is done when the required checks pass on the head commit, or when the repo has no CI, as step 5 defines it. Before you stop, handle the review comments that are already there, as in step 6. A fix starts another CI round. Never wait for a review that has not arrived, and never wait for an approval.

In T3 Code, call `unwatch_pull_request` before you stop, also when you stop to ask the user something. Report the PR link, the head commit, the check results, what you fixed along the way, and any follow-ups you declined.

Never merge, enable auto-merge or push to the base branch unless the user asked for it in this session. When asked to merge, run `git fetch`, confirm the PR is still green and not behind, and use the merge method the repo uses. Never bypass branch protection.

## Stack mode

A stack is a chain of PRs in which each one builds on the one below. The `gh stack` extension creates and updates it. Use this mode when the user asks for a stack, or when a skill hands over a layer plan. A layer plan names each layer from the bottom up: the branch name, its last commit, the PR title and what the body should say. With no plan, propose one and ask before you create branches.

The commits must already be in layer order on the current branch, so that each layer ends at one commit. If they are not, stop and ask, because reordering them rewrites history.

Steps 1 to 3 run as written, with one branch name per layer in the form from step 1. A layer plan from another skill keeps its names. These steps replace step 4:

1. Create each layer's branch at its last commit with `git branch <name> <commit>`. The working branch stays local and is never pushed, so give no layer its name.
2. Run `gh stack init --base <base> <bottom> ... <top>`. It adopts the branches as a stack and checks out the top layer. Stay on a branch of the stack for the `gh stack` commands that follow.
3. If the base has moved since the branch left it, run `gh stack rebase`.
4. Run `gh stack submit --auto --open`. It pushes every branch and opens one PR per layer, each based on the layer below. `--auto` skips the interactive editor. `--open` makes the PRs ready for review, so review bots run.
5. Set each PR's title and body with `gh pr edit <n> --title <title> --body-file <file>`, in the title form and the body format of step 4.
6. In T3 Code, link every PR to the thread.

Steps 5 to 7 run for every PR in the stack, with these differences:

- **A fix.** Put it in the lowest layer it applies to. Check out that branch, commit, run `gh stack rebase --upstack` to carry the fix into the layers above, then `gh stack push`. This rebases and force-pushes with a lease. It is allowed only on the branches of a stack.
- **Base moved, conflict, or a lower layer merged.** Run `gh stack sync` in place of the `mstack:sync-base` skill. On a rebase conflict it restores the branches and stops. Then run `gh stack rebase`, resolve each conflict by the rules under "Resolve conflicts" in `mstack:sync-base`, stage the files, and run `GIT_EDITOR=true gh stack rebase --continue` so that no editor opens, and finish with `gh stack push`. A rebase that applies cleanly can still break the code, and a clean `gh stack sync` has already pushed. So after either path, run the "Semantic conflict hunt" and "Verify" steps of `mstack:sync-base` on each layer the rebase changed, and send what they find as a fix, as above.
- **Stop.** Ship is done when the required checks pass on the head commit of every layer. Report every PR link, bottom to top.

Never run `gh stack merge` unless the user asked for it in this session.
