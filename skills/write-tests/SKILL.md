---
name: write-tests
description: Write and run tests from an approved test matrix, concrete bounded test request, or existing failing/flaky test. Use for TDD RED/GREEN tests, regression tests, integration tests, E2E tests, contract tests, and hardening brittle tests.
---

# Write Tests

Read relevant standards first:

- `${CLAUDE_PLUGIN_ROOT}/standards/test-driven-generation-workflow.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/e2e-network-mocking-standard.md` when E2E or browser behavior is involved

## Purpose

Turn an approved test matrix or concrete test request into executable tests, then run them and report fresh evidence.

The skill must preserve test intent. It should not expand scope just because nearby behavior looks testable.

## Preconditions

Proceed when one is true:

- The user approved a test matrix.
- The user gave a concrete, bounded test target.
- Existing failing/flaky tests define the task.

If the target is broad or ambiguous, route to `design-tests` first.

## Mode Rules

- `tdd`: write the failing test first, run it, confirm RED, then hand off or implement only if the user asked for implementation too.
- `regression`: lock existing behavior before changing code. If the test fails because the current behavior differs from the assumed behavior, pause and report the mismatch.
- `hardening`: reduce brittleness. Do not mask flakes with sleeps, broad retries, or weaker assertions unless explicitly justified.

## Test Quality Rules

- One test protects one behavior or policy.
- Test names describe expected behavior, not implementation details.
- Reuse the project’s existing render helpers, mock setup, fixture builders, and test conventions.
- Prefer accessible/user-level queries for UI tests.
- Prefer `userEvent`-style interaction over low-level event dispatch when testing user behavior.
- Avoid production `data-testid` additions unless accessible queries cannot express the target and the user approves.
- For contract tests, assert policy: null/default conversion, enum mapping, error normalization, date/unit conversion, or compatibility behavior.

## Workflow

1. Reuse existing test utilities and local patterns.
2. Write only tests inside the approved scope.
3. For TDD, run the test and confirm RED before implementation handoff.
4. For regression, confirm the behavior is protected.
5. For hardening, explain the flaky cause and why the new shape is more stable.
6. Run the relevant test command and report fresh output.

## Do Not

- Add tests for unrelated behavior.
- Change production code to make a poorly designed test pass.
- Replace a behavior test with a weaker implementation-detail test.
- Claim connectivity from mocked tests.
- Delete or loosen existing assertions without explaining the behavioral reason.

## Output

```text
Mode:
Intent:
Tests written:
Execution:
Coverage notes:
Remaining gaps:
```
