# Claude Code Style Kit Work Plan

## Goal

Create a reusable Claude Code plugin that makes code, tests, reviews, and TanStack Query server-state output consistent across new projects.

The plugin should encode personal preferences as reusable standards without depending on any specific company, product, API registry, codegen tool, or repository path.

## Non-Goals

- Do not include project-specific API adoption workflows.
- Do not require a specific codegen tool.
- Do not require external companion plugins.
- Do not install third-party plugins automatically.
- Do not encode company-specific PR, branch, release, or domain rules.

## Architecture

- `standards/`: source of truth for judgment.
- `skills/`: focused workflows that read and apply standards.
- `agents/`: role-specific review/execution lenses.
- `commands/`: thin entrypoints for common usage.
- `CLAUDE.md` / `AGENTS.md`: maintenance instructions for this plugin repository.

## MVP Assets

### Standards

- `coding-style-guide.md`
- `function-naming.md`
- `dto-domain-boundary.md`
- `tanstack-query-patterns.md`
- `test-driven-generation-workflow.md`
- `testing-code-style.md`
- `e2e-network-mocking.md`

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

Remove or avoid:

- fixed project paths
- company or product names
- specific domain examples that imply one business system
- specific API registry or codegen assumptions
- hardcoded test helper names
- hardcoded dev server or environment variable assumptions

Replace with:

- project pattern discovery
- local convention precedence
- conditional guidance based on detected stack
- explicit evidence requirements

## Validation Plan

1. Check plugin structure and manifest JSON.
2. Search for project-specific terms.
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
