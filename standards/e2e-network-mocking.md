# E2E Network Mocking

## Purpose

Avoid ambiguous E2E failures by making network mock ownership explicit.

## Rules

- For automated E2E, only one layer should own each API request.
- Prefer Playwright route ownership for per-test deterministic responses.
- Use service-worker or app-level mocks for local demo/manual mock apps, not mixed silently with per-test routes.
- If mixed ownership is unavoidable, document which URLs each layer owns.
- Connectivity evidence must come from live/staging smoke checks, not mocked E2E.

## E2E Writing Guidance

- Validate user-observable outcomes, not DOM internals.
- Capture console and network failures when diagnosing.
- Use screenshots/traces when they help reviewers confirm the scenario.
- Keep generated E2E specs separate from manually curated specs unless the project has an explicit merge policy.

## Review Checklist

- Which layer owns the API response?
- Is the scenario deterministic?
- Does the assertion prove user-visible behavior?
- Is mocked success being mistaken for live connectivity?
