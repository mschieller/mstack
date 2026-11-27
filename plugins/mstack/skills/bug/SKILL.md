---
name: bug
description: Fix a bug end to end. Reproduces it as a failing test, finds the root cause, fixes it, adds the test at the layer that should have caught it, reviews the fix and hands over to ship for the PR. Use for a bug report, a failing or flaky test, a regression, or "why does X happen", or when the user says "bug".
argument-hint: "[the bug report, issue number or failing test]"
---

<!-- Adapted from mattpocock/skills diagnosing-bugs and from the how and why flow in cursor/plugins pstack (both MIT, see LICENSE). Changes: the gap test takes its layer from the repo's CLAUDE.md, a review before the PR, a hand-over to ship, a handoff note when the work stops. -->

# Bug

Take a bug from a report to an open PR. Drive every step yourself. When the user only asked why something happens, stop after step 4 and report.

## Before you start

- Look for a handoff note for this bug with the `mstack:handoff` skill and `resume`. When one exists, start at its next step.
- On the default branch, create a branch `fix/<slug>`, in the form step 1 of the `mstack:ship` skill gives.

## Steps

1. Restate the bug in one line: given this, when that, then the actual result in place of the expected one. If you cannot, ask the user. It is the only question you should need.
2. Reproduce it as a test. Write the smallest failing test at the lowest layer that shows the bug. Run it and confirm that it fails for the stated reason. Change nothing in the code under test before this test exists and fails. For a flaky bug, run the test many times with the runner's repeat and shuffle options and record the seed. Flakiness usually comes from shared state, real time or an assumption about order.
3. Hypothesise. List two to four candidate causes, each with the evidence that would confirm it or rule it out. Test them cheaply: read the code path, add a temporary log or assertion, or bisect with `git bisect run <the test command>`. Drop each one the evidence rules out.
4. Name the root cause, with its `file:line` and why it produces the symptom. "It works now" is not a root cause.
5. Fix it with the smallest diff that turns the test green, then run the tests of the whole package or module. Do not fix by retrying, sleeping or widening a timeout, unless the timeout is the bug. Change one thing at a time.
6. Close the gap. Ask which layer should have caught this. When the repo's CLAUDE.md or AGENTS.md names test layers, take the layer from that list. Otherwise it is the layer where a test would have failed before the bug reached a user. Add that test too, unless the test from step 2 already sits there.
7. Clean. Remove the temporary logs and assertions. The diff holds the fix and its tests and nothing else.
8. Commit, with the type `fix` and a title in the form step 3 of the `mstack:ship` skill gives.
9. Review the fix. When the repo's CLAUDE.md or AGENTS.md names reviewers that always run, use the `mstack:deep-review` skill with the scope `branch`. Otherwise use the `mstack:review` skill. Neither skill ends this one. Fix what it finds, and put its decisions to the user.
10. Hand over to the `mstack:ship` skill. Give it a draft of the PR's Summary that names the root cause and the test that would have caught it.

## Stop conditions

Stop and ask only when you cannot restate the bug, when you cannot reproduce it after trying the two lowest layers, or when the fix needs a change in behaviour that no recorded decision covers.

When you have to stop partway, write a note with the `mstack:handoff` skill. Give it `bug` as the skill, the next step, the hypotheses you ruled out and the evidence for each.

## Report

The root cause with its `file:line`, the reproducing test, the gap test and the PR link. After a "why" question, the root cause and the evidence, with the reproducing test left uncommitted for the user to keep or drop.
