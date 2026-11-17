---
name: building-verification-loops
description: Use when deciding which tests or checks to write for a change, rewriting or migrating code that must keep its behaviour, facing a flaky or slow test, a check that keeps failing, adding a commit/CI gate or review rule, a bug that passed review or tests, or setting up a repo so agents can verify their own work with little human review.
---

# Building Verification Loops

A verification loop is a check that runs as software, can fail, and feeds its result back. The human defines "done" and reads the evidence; the agent produces it. Repo instructions (CLAUDE.md, a repo `verify` skill) win over this skill.

Source: https://claude.dev/blog/how-the-claude-code-team-thinks-about-verification/

**Related:** superpowers:verification-before-completion covers not claiming done without evidence. This skill covers what the evidence is and how to keep the checks honest.

## Which check to write

| Situation | Check | Expected values come from |
|---|---|---|
| A user-facing feature | End-to-end test of the real flow, reporting the step that failed | The flow the user performs |
| Parser, serializer, money, state, retries | Property test with generated inputs: round-trip, invariant, idempotency, equivalence to a naive version | A statement that must never be false |
| Rewrite or migration keeping behaviour | Differential test: old and new on the same generated or production inputs, noise normalized, approved differences listed | The old system, never a stale spec |

Unit tests alone can pass while the feature is broken.

## Running the change

Verification means running the changed thing: browser, simulator, the service locally, the real command.

- If the repo has a `verify` skill or documented boot procedure, use it before committing.
- If you had to work out how to boot and drive the app, leave that behind as a `verify` skill in the repo's `.claude/skills/`, next to the code it covers. A manual pass that leaves nothing behind has to be rediscovered by the next agent.
- Cheap checks first (tests, build, linter locally), then the shared ones (CI, review).
- A slow suite gets run less: keep a fast subset for every iteration and run the heavy checks before the commit.

## Rules that keep checks honest

1. **A change never edits the check that judges it.** No allowlist entry, loosened assertion, raised timeout or skipped test to get green. Gate rules are read from where the change cannot reach them (the base branch, server-side settings). A test whose expected value is wrong because the request itself changed the requirement is updated, and the PR says so.
2. **Three rounds, then stop.** A check still red after three fix attempts goes to the human with what was tried and the evidence.
3. **A flaky test is a bug.** Never rerun until it passes, add a sleep, or skip it. Find the root cause. If you cannot, report it as red with the measured failure rate; with the owner's approval, delete the test and write down why.
4. **A result is for one commit.** New commits after a review or test run void it.
5. **A check that escalates says why,** so the reviewer does not rebuild it from the diff.

## Adding a gate or review rule

- Give it three modes: off, observe, enforce. Start in observe, read what it would have blocked, then enforce.
- Tie each rule to a trigger readable from a diff and to the incident behind it, so only matching rules run.
- Validate against real misses: it must fail on the change that caused the incident and pass on a sample of clean merged changes. Keep a held-out set you do not tune against.
- Tune one reviewer with focused instructions and build each round on the previous one, before adding a second reviewer.
- Label findings important, nit, or pre-existing.

## After a miss

Every bug that got past the checks leaves a test or check that fails on that exact bug, in the place the next run reads it. A general review rule or hook is added on top when the pattern has occurred at least twice and can be detected from a diff. Prune rules, tests and instruction files routinely; they only grow otherwise.

## Red flags

- "Passed on retry"
- "I checked it by hand" with nothing left for the next run
- Green after editing the check, its allowlist or its timeout
- A fourth attempt at the same failing check
- A new gate that blocks from day one
