---
name: tanstack-query-patterns
description: Add or review TanStack Query server-state code, query key factories, queryOptions factories, mutations, invalidation, DTO normalization boundaries, and server-state scaffolds without assuming a project-specific API adoption workflow.
---

# TanStack Query Patterns

Read these original standards before acting:

- `${CLAUDE_PLUGIN_ROOT}/standards/tanstack-query-standard.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/api-domain-ui-standard.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`
- `${CLAUDE_PLUGIN_ROOT}/standards/scaffold-rest-domain-design.md` when creating a new server-state surface.

## Purpose

Create or review TanStack Query code so server-state behavior follows the original query/mutation ownership standard, cache ownership is explicit, and transport details do not leak into UI code.

## Workflow

1. Confirm the project uses TanStack Query.
2. Inspect nearby query/mutation patterns.
3. Preserve local conventions unless they are clearly broken.
4. Prefer query key factories and domain-owned query/mutation definitions.
5. Keep DTO mapping behind the adapter/fetcher boundary.
6. Put UI side effects in caller/orchestration code.

If the project does not use TanStack Query, do not introduce it without explicit user request.

## Query Checklist

- Query key root follows domain/cache ownership, not raw endpoint naming.
- Query key factory exposes broad and narrow invalidation targets.
- Query function returns normalized application/domain data when DTO cleanup is needed.
- `queryOptions()` is used when local project patterns support it.
- View-specific UX options, such as placeholder behavior, stay at the caller layer unless the project standard says otherwise.

## Mutation Checklist

- Mutation names describe business command intent.
- Mutation variables are named by caller intent, not transport payload mechanics.
- `onSuccess` invalidates the narrowest correct query key.
- UI side effects such as toast, dialog, navigation, and form reset are not hidden inside low-level API adapters.
- Error handling preserves useful domain/user information.

## Scaffold Guidance

When creating a new server-state surface, propose a shape before editing:

```text
domain-or-feature:
- types: application-facing types
- adapter/fetcher: transport call and DTO normalization
- query keys: cache ownership
- queries: queryOptions or local equivalent
- mutations: command functions and invalidation guidance
- tests: behavior or contract coverage
```

Adapt to the project’s existing folder structure. Do not force a `domain/` directory if the project uses another architecture.

## Avoid

- String query keys scattered across components.
- Generated DTO types imported directly into UI code when a domain boundary exists.
- Wrapper hooks that only hide `useQuery` or `useSuspenseQuery` without adding value.
- Broad invalidation when a narrower key is available.
- Treating HTTP method names as public domain names.
