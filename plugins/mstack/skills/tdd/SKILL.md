---
name: tdd
description: Build a behaviour change test-first, in red-green-refactor slices with one commit per slice. Use when implementing a feature or a fix from acceptance criteria, an issue or a plan, or when the user says "tdd" or "test first".
argument-hint: "[issue number, plan file or the behaviour to build]"
---

<!-- Adapted from mattpocock/skills tdd (MIT, see LICENSE). Changes: the test command and the test layers come from the repo's CLAUDE.md, slices come from written acceptance criteria, one commit per slice by the title rules of ship, a handoff note when the loop stops. -->

# TDD

Work one vertical slice at a time: a thin path through every layer it needs, proven by a test, before the next slice starts.

## Before the first slice

- Look for a handoff note for this task with the `mstack:handoff` skill and `resume`. When one exists, start at its next step.
- Find the test command. The repo's CLAUDE.md or AGENTS.md gives it as its test command. Otherwise take it from the package scripts, the Makefile or the CI workflow. Find the form that runs one test, and the form that runs the package or module you are in.
- Find the slices. They are the acceptance criteria of the issue, the decision record or the plan that the arguments name, one slice for each. When none are written down, write them as one line each, "given, when, then", show the list to the user and wait for an OK.
- On the default branch, create a branch in the form step 1 of the `mstack:ship` skill gives.

## Loop

1. Pick the next slice. Take the smallest first, which is often the plain successful case, then the errors, then the edge cases.
2. Red. Write the test and run it. It must fail for the right reason, so read the failure message. A test that fails on a typo or a missing import does not count.
3. Green. Write the least code that makes it pass. A hard-coded value is fine when the next test will force the general form.
4. Refactor with the tests green. Remove the duplication you created, not duplication elsewhere.
5. Run the tests of the whole package or module, not only the new one.
6. Commit the slice, the test and the code together, with a title in the form step 3 of the `mstack:ship` skill gives.

Repeat until the list is done. When the repo's CLAUDE.md or AGENTS.md names test layers, check which of them the change needs beyond the tests the loop produced, and add those.

## Rules

- Write no production code without a failing test that needs it. Wiring and generated code are the exception.
- Never weaken a test to make it pass. When a test is wrong, say so and change it in a commit of its own.
- Write tests the way the repo's own tests are written: their names, their helpers and their layout.
- After three red-green cycles on one slice with no progress, stop. The shape of the change is wrong, not the code. Say what blocks and ask.

## When you stop

When the loop has to stop before the list is done, write a note with the `mstack:handoff` skill. Give it `tdd` as the skill, the next slice as the next step, and the slices that are left.

## Report

Each slice with its test and its commit, the slices that are left, and anything you could not test.
