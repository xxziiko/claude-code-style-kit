---
name: function-naming
description: Name and review functions, hooks, query/mutation factories, mappers, event handlers, adapters, and public APIs by caller intent rather than transport, generated, framework, batching, or implementation details.
---

# Function Naming

Read `${CLAUDE_PLUGIN_ROOT}/standards/function-naming.md`.

Use this skill when a name crosses a boundary:

- Generated or vendor API to app adapter.
- Adapter to domain API.
- Domain API to hook or component.
- Component interaction to handler.

Prefer caller intent over transport or implementation detail. Preserve generated/vendor names only at their boundary.

## Naming Decision Steps

1. Identify the layer: generated/vendor, adapter, domain/service, query/mutation, hook, UI handler, mapper, or test helper.
2. Identify the caller and what the caller needs to know.
3. Remove details that are only transport, framework, batching, or version mechanics.
4. Keep details that change caller behavior or domain meaning.
5. Compare with nearby names in the same layer.

## Review Flags

- A public name includes generated operation names without adding business meaning.
- A hook name hides whether it reads server state, mutates server state, or controls local UI state.
- A handler name describes implementation instead of user action.
- Similar functions in one layer use different verbs for the same concept.
- A short name is only understandable after reading the implementation.

## Output When Reviewing

```text
Current name:
Layer:
Caller intent:
Issue:
Suggested name:
Reason:
```
