# Project Harness Standard

This standard defines the shared Sirloin OMS frontend harness surface.

## Purpose

The project harness keeps agent and teammate workflows reproducible without storing personal agent personas, local orchestration preferences, or private memory in the repository.

## Harness Layers

1. `AGENTS.md` - always-read project contract and task routing.
2. `docs/standards` and `docs/conventions` - durable implementation rules.
3. `.claude/commands` - project slash-command entrypoints.
4. `.claude/skills/project` - executable overlays for project standards and scaffolds.
5. `scripts/*` - deterministic checks and scaffold helpers.
6. Git hooks and CI - default verification path.

## Repository Boundary

Keep these in the project repository:

- Project standards and conventions.
- Project-shared Claude commands and project skills.
- Deterministic scripts that validate or scaffold shared project behavior.
- Local and CI hooks that enforce the shared harness surface.

Keep these outside the project repository:

- Personal agent personas.
- Local harness duplicates.
- Private memory, notes, and vault paths.
- Machine-specific orchestration preferences.

If a personal agent overlay contains a durable project rule, move the rule into `docs/standards`, `docs/conventions`, or a project skill before relying on it.

## Verification Boundary

Project-wide deterministic checks should validate shared project behavior, not
personal agent overlays or local orchestration preferences.

Use existing project checks such as:

```bash
yarn standards:check
```

Keep personal harness validation outside this repository. If a personal harness
rule becomes a durable project rule, move the rule into an active standard,
project command, project skill, or CI check before relying on it.

## Maintenance Rules

- Keep `AGENTS.md` compact; route to standards and project skills rather than duplicating their bodies.
- Keep command files thin but complete: include when to use them, workflow, gates, output contract, and prohibitions.
- Prefer one active standard over multiple overlapping design drafts.
- Archive or remove completed plans and superseded design drafts instead of listing them as active standards.
