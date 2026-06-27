---
name: design-tests
description: Design a test strategy before writing tests. Use for TDD planning, regression protection before refactors or migrations, flaky test hardening, UI/server-state wiring changes, coverage planning, or deciding unit vs integration vs E2E vs live-smoke evidence.
---

# Design Tests

Read relevant standards first:

- `${CLAUDE_PLUGIN_ROOT}/standards/test-driven-generation-workflow.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/testing-code-style.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/e2e-network-mocking.md` when E2E or browser behavior is involved

## Purpose

Design tests before writing them. The goal is to prevent low-value AI-generated tests by making the protected behavior, test layer, and evidence type explicit first.

This skill produces a test matrix. It does not implement tests unless the current user request already includes explicit approval to write them.

## Mode Selection

- `tdd`: new behavior or spec-first work. Design RED tests that should fail before implementation.
- `regression`: existing behavior changes, refactors, dependency swaps, server-state rewiring. Lock current user-visible behavior first.
- `hardening`: flaky or brittle tests. Identify instability cause and design a more reliable test shape.

## Intent Selection

- `behavior`: user actions, visible UI state, workflow orchestration, alerts/toasts/navigation, state transitions.
- `contract`: DTO/domain conversion, mapper policy, adapter shape, typed fixtures, schema compatibility.
- `connectivity`: live/staging smoke or real service integration. Mocked tests cannot satisfy this intent.

## Workflow

1. Inspect the task, changed surfaces, existing tests, and local test utilities.
2. Choose mode: `tdd`, `regression`, or `hardening`.
3. Choose intent: `behavior`, `contract`, or `connectivity`.
4. Identify protected behavior, policy, or integration contract.
5. Recommend the nearest useful test layer.
6. Exclude low-value tests.
7. Stop with a test matrix unless the user has already approved writing tests.

## Layer Decision

Prefer the nearest layer that proves the claim:

- Pure policy with no framework coupling -> unit test.
- Component/hook/server-state orchestration -> integration test.
- Full user workflow, routing, browser behavior, or cross-page state -> E2E.
- Real backend/service availability -> live smoke or connectivity check.

Do not jump to E2E because it feels more complete. Use E2E when lower layers cannot prove the user workflow.

## Low-Value Tests To Exclude

- Import/export existence checks.
- Library behavior checks.
- Tests that only assert `queryOptions`, `mutationOptions`, or a wrapper exists.
- Field-copy mapper tests with no policy.
- Snapshot-like DOM structure tests that do not protect user behavior.
- Mock-only tests used as connectivity evidence.

## Approval Gate

After producing the matrix, stop unless the user already asked to implement tests. If the user says "응", "진행해", or similar after the matrix, treat that as approval only for the proposed matrix.

## Output

```text
Mode:
Intent:
Scope:
Protected behaviors:
Recommended layers:
Test matrix:
Gate evidence:
Excluded tests:
Risks / gaps:
Approval needed:
```

Do not write test code from this skill unless approval is explicit in the current request.
