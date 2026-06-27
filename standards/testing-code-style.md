# Testing Code Style

## General

- Prefer user behavior over implementation details.
- One test should protect one behavior or policy.
- Test names should describe expected behavior.
- Reuse existing test utilities and render helpers.
- Keep fixtures close to the behavior they explain, unless shared policy justifies centralization.

## React Testing

- Prefer role, label, and text queries over class names or DOM structure.
- Prefer `userEvent` over low-level event firing when testing user interaction.
- Use async assertions deliberately; avoid arbitrary sleeps.
- Add `data-testid` only when accessible queries cannot express the target.

## Mocking

- Mock at the narrowest boundary that still proves the behavior.
- Prefer network mocks for integration behavior involving server state.
- Prefer typed fixtures for adapter/mapper contract tests.
- Do not claim live connectivity from mocked tests.

## Regression Tests

- Lock the behavior before refactoring.
- Keep the test focused enough to fail for the intended regression.
- If a regression cannot be tested reasonably, document the gap and use the strongest available verification.
