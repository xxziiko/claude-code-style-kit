---
name: dto-domain-boundary
description: Keep external DTOs, generated types, API responses, database rows, serialized storage, and third-party SDK objects from leaking into application/domain/UI code. Use when adding adapters, mappers, data fetching, API integrations, or reviewing boundary type usage.
---

# DTO / Domain Boundary

Read these original standards before acting:

- `${CLAUDE_PLUGIN_ROOT}/standards/api-domain-ui-standard.md`

Check:

- Where boundary data enters the app.
- Where defaults and null policy are handled.
- Whether UI code imports transport/generated types.
- Whether mapper tests protect real policy instead of field copying.

Keep normalization centralized and explicit. Mapper ownership follows the source standards even when the mapping is mechanically simple.

## Boundary Decision Steps

1. Identify the external shape: API response, generated type, database row, SDK object, or serialized storage.
2. Identify the application-facing type that callers should use.
3. Decide where normalization belongs: adapter, mapper, repository, server action, or equivalent project boundary.
4. Centralize null/default/error policy.
5. Keep UI/components from repeating transport cleanup.

## Mapper Test Guidance

Test mappers when they encode policy:

- null/default conversion
- enum/status translation
- date/time or unit conversion
- error normalization
- backward compatibility across response versions

Do not add tests for plain field copying unless it protects a known regression or complex compatibility case.

## Review Flags

- Generated/transport types imported into UI without a deliberate project policy.
- Multiple components repeat the same fallback/default logic.
- DTO field names drive public domain naming.
- Mapper logic is split across hooks/components.
- Tests assert mock response plumbing but miss domain policy.
