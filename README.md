# Claude Code Style Kit

Reusable Claude Code plugin assets for producing consistent code, tests, review output, and server-state patterns across new projects.

## Includes

- Original harness standards needed by the bundled skills: TypeScript style, function naming, TanStack Query, test-driven generation, and E2E network mocking.
- Skills that apply those standards in focused workflows.
- Agents for review, verification, testing, architecture, and build repair.
- Thin commands for common entrypoints, including `/commit`.

## Does Not Include

- API adoption workflows tied to a specific codegen or registry.
- Automatic installation of third-party plugins.
- Company-specific PR, branch, or release conventions.

## Suggested Companion Plugins

Keep companion plugin setup in a separate bootstrap README or dotfiles repo. This plugin should remain independently usable.

Recommended optional companions:

- `oh-my-claudecode` for multi-agent orchestration.
- `frontend-fundamentals` for additional frontend quality review lenses.

## Install

Add this public marketplace in Claude Code:

```text
/plugin marketplace add xxziiko/claude-code-style-kit
```

Then install the plugin:

```text
/plugin install claude-code-style-kit@xxziiko-style-kit
```

Because this repository is public, installation does not require GitHub read authentication.

## Local Test

```bash
claude --plugin-dir .
```

Then run:

```text
/claude-code-style-kit:test-plan
/claude-code-style-kit:review
/claude-code-style-kit:verify
/claude-code-style-kit:commit
```
