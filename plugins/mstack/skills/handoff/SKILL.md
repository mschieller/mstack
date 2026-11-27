---
name: handoff
description: Write a note that lets a later session or another agent carry on a task from cold, or find and read such a note with "resume". Use when stopping partway through a task, when the context is getting long, when the user says "handoff" or "write this up for the next session", and at the start of a session that continues earlier work. Other skills call it when they have to stop.
argument-hint: "[resume] [slug or a word from the task]"
---

<!-- Adapted from mattpocock/skills handoff (MIT, see LICENSE). Changes: notes live in the repo's shared git folder, a resume mode, a line that names the skill and its next step. -->

# Handoff

A handoff note holds what a new session cannot get from git: the goal, what is half done, what was decided and the next step.

## Where notes live

Notes go in the repo's shared git folder. Every worktree of the repo sees them there, a removed worktree does not take them along, and git can never commit them.

First run `git rev-parse --git-common-dir`. When it fails, there is no repo and no place for a note. Say so, give the note in your reply, and stop. Otherwise:

```
dir="$(git rev-parse --path-format=absolute --git-common-dir)/mstack/handoffs"
```

Run `mkdir -p "$dir"` before you write a note, and not for `resume`.

## Write

Without `resume`, write `<dir>/<yyyy-mm-dd>-<slug>.md`. The slug names the task in at most five words of kebab-case. A task has one note: when `ls "$dir"` shows a note with the same slug, replace it and remove the older file.

```markdown
# <the task in one line>

- Skill: <the skill that was running, or none>, next step: <its step number and name>
- Ref: <an issue, a PR or a decision record, when there is one>

## Goal

<what done looks like>

## State

- Branch: <name> at <short sha>, pushed: yes or no. Worktree: <path>
- Uncommitted: <the files with uncommitted changes, or none>
- Done: <each item with its evidence: a test name, or a command and its result line>
- In progress: <what, where as file:line, and what is half finished>
- Not started: <what is left>

## Decisions

<the choices made in this session and why, which ones the user confirmed, and what was ruled out>

## Open questions

<for the user, each one answerable in one line>

## Next step

<the single next action, concrete enough to start on without reading everything again>

## Gotchas

<what surprised you: a flaky test, an odd command, a misleading name>
```

- Write facts. Leave out the story of how you got here.
- Write no secret, token or password into the note.
- The note holds no code. Uncommitted changes stay in their worktree, so name that worktree and the files.
- When a skill calls this one, it gives its name and its next step for the Skill line, and what it has not finished.
- When a gotcha will matter to every later session, also propose a line for the repo's CLAUDE.md.

End by telling the user the note's path and how to pick it up: `/mstack:handoff resume <slug>`.

## Resume

With `resume`, list the notes newest first with `ls -t "$dir"`. When the folder does not exist, say that there are no notes. Take the newest note whose file name or first line matches the other words in the arguments. With no other words, take the newest note whose Branch line names the current branch. When nothing matches, say so, list the file names and load nothing. A skill that called this one then goes on from its own start. When several notes match, say which one you took and name the others.

Read the note, then check its State against the repo before you trust it:

- the branch exists and its sha is the note's sha or a descendant of it, which `git merge-base --is-ancestor <sha> <branch>` shows by succeeding
- the files under Uncommitted still have changes in the worktree the note names

When the branch no longer exists, the sha is not in it, or the worktree is gone, say what differs and ask whether to go on, to start over or to delete the note. Otherwise say what differs, switch to the note's worktree and branch, and go on at the note's next step. When its Skill line names a skill, load that skill and start at the step it names. Ask the note's open questions first when the next step depends on them.

Delete the note when its task is done.
