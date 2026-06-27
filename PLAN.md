# Claude Code Style Kit Work Plan

## Goal

Create a reusable Claude Code plugin that makes code, tests, reviews, and TanStack Query server-state output consistent across new projects.

The plugin should preserve the original standard documents as the source of truth, while skills provide reusable execution guidance around those standards.

## Non-Goals

- Do not include project-specific API adoption workflows as executable commands.
- Do not require a specific codegen tool for installation or use.
- Do not require external companion plugins.
- Do not install third-party plugins automatically.
- Do not simplify source standards into lossy summaries.

## Architecture

- `standards/`: original source of truth for judgment.
- `skills/`: focused workflows that read and apply standards.
- `agents/`: role-specific review/execution lenses.
- `commands/`: thin entrypoints for common usage.
- `CLAUDE.md` / `AGENTS.md`: maintenance instructions for this plugin repository.

## MVP Assets

### Standards

- Original files from the source `docs/standards/` directory, preserving filenames and wording.
- Skills may explain how to apply these standards, but must not replace them with summaries.

### Skills

- `coding-style-guide`
- `function-naming`
- `dto-domain-boundary`
- `tanstack-query-patterns`
- `design-tests`
- `write-tests`

### Agents

- `architect`
- `code-reviewer`
- `critic`
- `verifier`
- `test-engineer`
- `integration-tester`
- `e2e-tester`
- `build-fixer`

### Commands

- `review`
- `test-plan`
- `write-tests`
- `verify`

## Companion Plugin Policy

Companion plugins are documented separately in `ops/claude-code-plugin-setup.md`.

The style kit may mention optional integrations, but it must not require or auto-install them.

Recommended companions:

- `oh-my-claudecode`
- `frontend-fundamentals`

## Generalization Rules

Do not rewrite source standards for generality. If a source standard contains project-specific examples, keep them as provenance and examples of the original engineering intent.

Generalization belongs in skills, agents, commands, and README usage guidance:

- discover local project patterns before editing
- apply stack-specific rules only when the stack exists
- treat source examples as judgment references, not mandatory folder names for unrelated projects
- require explicit evidence before claiming completion

## Validation Plan

1. Check plugin structure and manifest JSON.
2. Confirm `standards/` matches the source standards set.
3. Load locally with `claude --plugin-dir ./plugins/claude-code-style-kit` when Claude CLI is available.
4. Forward-test against a small generic React task:
   - ask for a test plan only and confirm no test code is written
   - approve a small matrix and confirm tests stay inside scope
   - ask for TanStack Query scaffold and confirm no fixed project path is assumed
   - ask for review and confirm findings use standards as lenses

## Current Status

- MVP scaffold created.
- Standards, skills, agents, commands, and companion setup doc drafted.
- Official Claude CLI validation is pending because `claude` is not available in the current shell PATH.
