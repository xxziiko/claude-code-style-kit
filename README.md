# Xxziiko

Practical Claude Code plugin assets for running project work through a clear flow: scan, implement, fix, check, review, verify, and commit.

## Includes

- Original harness standards needed by the bundled skills: TypeScript style, function naming, TanStack Query, test-driven generation, and E2E network mocking.
- Skills that apply those standards in focused workflows.
- Agents for review, verification, testing, architecture, and build repair.
- Thin commands for common entrypoints, including `/scan`, `/do`, `/fix`, `/check`, and `/commit`.

## Does Not Include

- API adoption workflows tied to a specific codegen or registry.
- Automatic installation of third-party plugins.
- Company-specific PR, branch, or release conventions.

## Suggested Companion Plugins

Keep companion plugin setup in a separate bootstrap README or dotfiles repo. This plugin should remain independently usable.

Recommended optional companions:

- `oh-my-claudecode` for multi-agent orchestration.
- `frontend-fundamentals` for additional frontend quality review lenses.
- `ponytail` for over-engineering reduction, deletion-first review, and minimum viable diffs.

See `COMPANIONS.md` for practical playbooks that combine these companions with the `xxziiko` workflow.

## Install

Add this public marketplace in Claude Code:

```text
/plugin marketplace add xxziiko/claude-code-style-kit
```

Then install the plugin:

```text
/plugin install xxziiko@xxziiko-style-kit
```

Because this repository is public, installation does not require GitHub read authentication.

## Local Test

```bash
claude --plugin-dir .
```

Then run:

```text
/xxziiko:scan
/xxziiko:do
/xxziiko:fix
/xxziiko:check
/xxziiko:test-plan
/xxziiko:review
/xxziiko:verify
/xxziiko:commit
```

## Workflow Commands

Use these commands as a practical execution flow for real project work:

- `/scan`: inspect the project and requirements without editing, then define success criteria and a small implementation plan.
- `/do`: implement the smallest safe change using existing patterns and no new dependencies unless explicitly requested.
- `/fix`: debug from logs and reproduction evidence, fix the root cause, and rerun focused verification.
- `/check`: inspect the diff, run available verification, and summarize changes, evidence, and remaining risks.
