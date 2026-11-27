---
name: tdd
description: Build a behaviour change test-first, in red-green-refactor slices with one commit per slice. Use when the user says "tdd" or "test first", or asks to build an issue or a plan test-first, and when another skill hands over a list of acceptance criteria to build.
argument-hint: "[issue number, plan file or the behaviour to build]"
---

<!-- Adapted from mattpocock/skills tdd (MIT, see LICENSE). Changes: the test command and the test layers come from the repo's CLAUDE.md, slices come from written acceptance criteria, one commit per slice by the title rules of ship, a handoff note when the loop stops. -->

# TDD

Work one vertical slice at a time: a thin path through every layer it needs, proven by a test, before the next slice starts.

## Before the first slice

- Run the `mstack:handoff` skill with `resume` and the issue number, or the name of the plan or record that the arguments give. With neither, run it with `resume` alone, which takes the note of the current branch. When it finds a note, start at the note's next step. When it finds none, go on. Skip this bullet when `mstack:handoff resume` loaded this skill, or when the skill that called this one says it already ran it. Start then at the step and with the slices the caller names.
- Find the test command. The repo's CLAUDE.md or AGENTS.md gives it as its test command. Otherwise take it from the package scripts, the Makefile or the CI workflow. Find the form that runs one test, and the form that runs the package or module you are in. Find the option that lists each test with its result, such as `go test -v`, because a runner may print a pass for a package whose test was skipped. When the repo has no test runner or no way to run one test, stop and ask whether to set one up, and name the runner you would pick.
- Run the tests of the package or module once and record what already fails and what is skipped. A skipped test that a slice depends on, such as one that needs a database, has to run before the loop starts. Fix the setup, or stop and ask.
- Find the slices. They are the acceptance criteria of the issue, the decision record or the plan that the arguments name, one slice for each. When none are written down, write them as one line each, "given, when, then", show the list to the user and wait for an OK.
- On the default branch, create a branch in the form step 1 of the `mstack:ship` skill gives.

## Loop

1. Pick the next slice. Take the smallest first, which is often the plain successful case, then the errors, then the edge cases.
2. Red. Write the test and run it. It must fail for the right reason, so read the failure message in the output that lists each test. A test that fails on a typo or a missing import does not count. A skipped test has neither failed nor passed, so fix what makes it skip, or stop and ask. When the code under test does not exist yet, add the smallest stub that lets the test compile or load. The stub returns a zero value or does nothing, so that the assertion is what fails. Use a stub that panics or throws "not implemented" only where no zero value can be returned. When the test passes against a stub you just added, the behaviour does not exist yet. Make the stub return a value the assertion rejects, see the test fail, and go on with step 3. When the test passes at once against code that was already there, follow "A test that passes at once" below.
3. Green. Write the least code that makes it pass, in place of the stub when there is one. A hard-coded value is fine when the next test will force the general form. The test must show as run and passed in the output that lists each test. A skipped test is not a pass.
4. Refactor with the tests green. Remove the duplication you created, not duplication elsewhere.
5. Run the tests of the whole package or module, not only the new one. A test that failed before the first slice does not block the loop.
6. Commit the slice. Run `git status --short` first. Delete the files that a tool wrote while a test failed on purpose, such as a saved failing case, and ask about any other untracked file you did not write. Then commit the test and the code together, staged by path, with a title in the form step 3 of the `mstack:ship` skill gives.

Repeat until the list is done. When the repo's CLAUDE.md or AGENTS.md names test layers, check which of them the change needs beyond the tests the loop produced, and add those, each committed as a slice of its own.

## A test that passes at once

The behaviour already exists. Keep the test only when no other test covers the case, and show first that it can fail:

1. Run `git diff -- <file>` on the file you are about to break and keep its output.
2. Break the line the test guards, run the test and see it fail.
3. Undo the edit by hand. `git diff -- <file>` must print the kept output again.
4. Run the test again and see it pass.

Commit the test as step 6 of the loop says, with the type `test`, and go on to the next slice.

## Rules

- Write no production code without a failing test that needs it. Wiring, generated code and the stub of step 2 are the exceptions.
- Never weaken a test to make it pass. When a test is wrong, say so and change it in a commit of its own.
- Write tests the way the repo's own tests are written: their names, their helpers and their layout.
- After three red-green cycles on one slice with no progress, stop. The shape of the change is wrong, not the code. Say what blocks and ask.

## When you stop

When the loop has to stop before the list is done, write a note with the `mstack:handoff` skill. Give it `tdd` as the skill, "Loop, step 1" as the next step, the issue or record the slices came from as the ref, and the slices that are left.

## Report

Each slice with its test and its commit, the slices that are left, and anything you could not test. When the list is done, delete this task's handoff note, if one exists.
