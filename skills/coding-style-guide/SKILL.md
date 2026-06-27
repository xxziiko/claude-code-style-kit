---
name: coding-style-guide
description: Apply reusable TypeScript/React coding style standards for readability, predictability, cohesion, coupling, file organization, component boundaries, and type safety. Use when writing, reviewing, refactoring, or cleaning up application code.
---

# Coding Style Guide

Read these original standards before acting:

- `${CLAUDE_PLUGIN_ROOT}/standards/typescript-code-style.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`

Apply the project’s local conventions first when they are stricter. If local patterns conflict with this plugin, report the conflict and follow the project unless the user asks to migrate conventions.

## Purpose

Make implementation output consistent with the bundled source standards: readable names, cohesive files, predictable side effects, type-safe boundaries, and small reviewable diffs.

Before editing:

- Inspect nearby files.
- Reuse existing utilities and abstractions.
- Keep diffs scoped to the user’s task.

## Implementation Checklist

- Name values and functions by intent.
- Keep transport, server-state, and UI responsibilities separate according to the source standards.
- Prefer existing project helpers over new abstractions.
- Add an abstraction only when it removes real duplication or clarifies ownership.
- Keep component state as local as possible.
- Use explicit types at boundaries and avoid `any`.
- Preserve existing behavior unless the user asked to change it.
- Add or update tests when behavior changes.

## Refactor Checklist

- Lock behavior first when behavior is not already protected.
- Change one axis at a time: naming, extraction, boundary move, or behavior.
- Avoid mixing cleanup with feature work unless needed to finish safely.
- Leave unrelated cleanup in a follow-up note instead of expanding the diff.

Before finishing:

- Run the relevant lint/type/test command when available.
- Report any unverified assumptions.
