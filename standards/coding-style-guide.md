# Coding Style Guide

## Purpose

Keep generated code readable, predictable, cohesive, and easy to review.

## Core Rules

- Prefer explicit business intent over implementation-shaped names.
- Keep files cohesive; split when multiple reasons to change appear in one file.
- Prefer named constants for non-obvious values.
- Avoid `any`; use `unknown`, typed adapters, or type guards.
- Do not add wrapper layers unless they remove real duplication or clarify ownership.
- Reuse existing project utilities and component patterns before adding new ones.
- Keep side effects visible at orchestration boundaries.
- Avoid barrel exports unless the project already relies on them consistently.

## React Rules

- Separate data orchestration from presentational rendering when the component becomes hard to scan.
- Prefer composition over boolean prop proliferation.
- Keep UI state local unless multiple surfaces genuinely share it.
- Do not introduce global state for transient component concerns.
- Use existing design-system components before creating new primitives.

## Discovery

Before changing code, inspect:

- Existing nearby files.
- Existing test helpers.
- Existing naming and folder patterns.
- Project lint/typecheck/test commands.

If the project has stricter local rules, follow the local rules.
