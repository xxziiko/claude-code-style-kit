# Test-Driven Generation Workflow

## Purpose

Keep AI-generated tests meaningful by separating test design from test writing.

## Principles

- Specification comes before tests.
- Test design and test writing are separate steps.
- Existing behavior changes are regression-first.
- New behavior is test-plan-first or TDD when practical.
- Tests must protect user behavior, business policy, state transitions, or integration contracts.
- Every test plan should identify intent: `behavior`, `contract`, or `connectivity`.

## Modes

- `tdd`: define RED tests before implementation.
- `regression`: lock current behavior before changing it.
- `hardening`: make flaky or weak tests more reliable.

## Intents

- `behavior`: user-observable behavior, UI state, workflow orchestration.
- `contract`: mappers, adapters, DTO/domain conversion, typed fixtures.
- `connectivity`: live integration or smoke evidence against real services.

## Workflow

1. Read the task, local patterns, and existing tests.
2. Identify protected behavior or policy.
3. Choose layer: unit, integration, E2E, or live smoke.
4. Exclude low-value tests.
5. Produce a reviewable test matrix.
6. Write tests only after the matrix is approved or the user gave a concrete test target.
7. Run tests and report fresh evidence.

## Low-Value Tests

Avoid tests that only verify:

- Imports or exports.
- Library behavior.
- Query option existence without behavior.
- Field-copy mappers with no policy.
- DOM structure unrelated to user behavior.
