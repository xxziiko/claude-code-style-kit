# Agent Instructions

Before editing this plugin, classify each rule as:

- `core`: reusable across typical TypeScript/React projects.
- `conditional`: reusable only when the stack is present.
- `overlay`: project-specific and not allowed in core assets.

Preserve the structure:

- `standards/` defines judgment.
- `skills/` applies judgment.
- `agents/` reviews or executes from a role perspective.
- `commands/` provides short entrypoints.

After substantial changes, forward-test the plugin against a small generic React task and check that it does not assume project-specific paths.
