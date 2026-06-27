# TanStack Query Patterns

## Purpose

Use TanStack Query with predictable cache ownership, clear query/mutation boundaries, and stable domain-facing APIs.

## Query Rules

- Use query key factories.
- Root query keys should follow domain/cache ownership, not raw endpoint names.
- Prefer `queryOptions()` factories when the project uses TanStack Query v5 patterns.
- Keep DTO-to-domain conversion behind the fetcher/adapter boundary.
- Let callers add view-specific UX options such as placeholder behavior when those options are screen-specific.

## Mutation Rules

- Use mutations for commands and writes.
- Name mutation variables by business intent.
- Invalidate cache keys intentionally; avoid broad invalidation unless the domain requires it.
- Keep alert, navigation, toast, and form reset behavior in the caller or UI orchestration layer.

## Hook Guidance

- Do not add custom query wrapper hooks by default if they only hide `useQuery`/`useSuspenseQuery` without adding domain value.
- Do add hooks when they compose multiple queries, own UI orchestration, encapsulate mutation side effects, or match an established project pattern.
- Prefer the project’s existing convention when it differs.

## Scaffold Shape

Typical domain server-state surface:

```text
domain/
  <feature>/
    api.ts
    types.ts
    queries.ts
    mutations.ts
    mappers.ts
    __tests__/
```

Adapt the shape to the target project instead of forcing this layout.
