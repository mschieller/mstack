---
name: bug
description: Fix a bug end to end. Reproduces it as a failing test, finds the root cause, fixes it, adds the test at the layer that should have caught it, reviews the fix and hands over to ship for the PR. Use for a bug report, a failing or flaky test, a regression, or "why does X happen", or when the user says "bug".
argument-hint: "[the bug report, issue number or failing test]"
---

<!-- Adapted from mattpocock/skills diagnosing-bugs and from the how and why flow in cursor/plugins pstack (both MIT, see LICENSE). Changes: the gap test takes its layer from the repo's CLAUDE.md, a review before the PR, a hand-over to ship, a handoff note when the work stops. -->

# Bug

Take a bug from a report to a reviewed fix, and to an open PR when the user wants one. Drive every step yourself. When the user only asked why something happens, stop after step 4, remove the temporary logs and assertions, and report.

## Before you start

- Run the `mstack:handoff` skill with `resume` and the issue number or the slug of this bug. When it finds a note, start at the note's next step. When it finds none, go on. Skip this bullet when `mstack:handoff resume` loaded this skill.
- Find the test command as the `mstack:tdd` skill does under "Before the first slice".
- On the default branch, create a branch `fix/<slug>` before step 5, in the form step 1 of the `mstack:ship` skill gives. On another branch, when `git log origin/<base>..HEAD` or `git status --short` shows work that has nothing to do with this bug, ask whether to branch `fix/<slug>` from the base or to fix it here.

## Steps

1. Restate the bug in one line: given this, when that, then the actual result in place of the expected one. If you cannot, ask the user.
2. Reproduce it as a test. Write the smallest failing test at the lowest layer that shows the bug. Run it and confirm that it fails for the stated reason. Change nothing in the code under test before this test exists and fails. For a flaky bug, run the test many times with the runner's repeat and shuffle options and record the seed. Flakiness usually comes from shared state, real time or an assumption about order.
3. Hypothesise. List two to four candidate causes, each with the evidence that would confirm it or rule it out. Test them cheaply: read the code path, add a temporary log or assertion, or bisect. To bisect, copy the test from step 2 to a place outside the repo, run `git bisect start HEAD <a good commit>`, then `git bisect run` with a script that copies the test in and runs it, then `git bisect reset`. Drop each one the evidence rules out.
4. Name the root cause, with its `file:line` and why it produces the symptom. "It works now" is not a root cause.
5. Fix it with the smallest diff that turns the test green, then run the tests of the whole package or module. Do not fix by retrying, sleeping or widening a timeout, unless the timeout is the bug. Change one thing at a time.
6. Close the gap. Ask which layer should have caught this. When the repo's CLAUDE.md or AGENTS.md names test layers, take the layer from that list. Otherwise it is the layer where a test would have failed before the bug reached a user. Add that test too, unless the test from step 2 already sits there.
7. Clean. Remove the temporary logs and assertions. The diff holds the fix and its tests and nothing else.
8. Commit, with a title in the form step 3 of the `mstack:ship` skill gives. The type is `fix`, or `test` when the only change is to a test.
9. Review the fix. When the repo's CLAUDE.md or AGENTS.md names reviewers that always run, use the `mstack:deep-review` skill with the scope `branch`. Otherwise use the `mstack:review` skill. Neither skill ends this one. Each fixes the mechanical findings and walks its decisions with the user itself. Go on with step 10 after it.
10. Hand over to the `mstack:ship` skill when the user asked for a PR or called this skill by name. Otherwise stop here, report, and offer the PR. Give ship a draft of the PR's Summary that names the root cause and the test that would have caught it.

## Stop conditions

Stop and ask only when you cannot restate the bug, when you cannot reproduce it after trying the two lowest layers, or when the fix needs a change in behaviour that no recorded decision covers.

When you have to stop partway, write a note with the `mstack:handoff` skill. Give it `bug` as the skill, the next step, the hypotheses you ruled out and the evidence for each.

## Report

The root cause with its `file:line`, the reproducing test, the gap test and the PR link. After a "why" question, the root cause and the evidence, with the reproducing test left uncommitted for the user to keep or drop. When the fix is done, delete this bug's handoff note, if one exists.
