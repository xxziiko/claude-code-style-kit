---
name: commit
description: Prepare or create a focused git commit from the current worktree. Use when the user asks for /commit, commit help, staging guidance, commit message drafting, atomic commit splitting, or a safe commit that matches the repository's existing history style.
---

# Commit

Use this skill to turn local changes into a focused, reviewable commit.

## Core Rules

- Inspect the repository state before deciding what to commit.
- Treat already staged files as the user's intended commit unit unless evidence shows a problem.
- If nothing is staged, propose a commit unit from the worktree before staging.
- Stage only files that belong to the selected commit unit.
- Never include secrets, credentials, `.env` files, local editor files, logs, unrelated generated output, or unrelated user changes.
- Never use `--no-verify`.
- Match the repository's commit style from recent history unless the user or repo instructions specify a stricter format.
- Prefer commit messages that explain why the change exists, not just what files changed.
- If the repo defines a commit protocol, follow it exactly.

## Inspection Checklist

Run or inspect the equivalent of:

- current branch
- `git status --short`
- staged diff
- unstaged diff
- untracked files
- recent commits
- applicable repo instructions such as `AGENTS.md`, `CLAUDE.md`, or contribution docs

## Commit Unit Decision

Use one commit when the changes share one intent and can be reviewed or reverted together.

Split or ask for direction when:

- unrelated files changed
- formatting churn is mixed with behavior changes
- generated files are mixed with source edits
- docs, tests, and implementation tell different stories
- the staged set differs from the unstaged set in a way that could hide partial work

If the user explicitly asks you to commit and the unit is clear, proceed. If the unit is ambiguous or risky, stop with a proposed split.

## Message Drafting

Prefer the local style. If there is no obvious local style, use this shape:

```text
<intent line>

<body explaining context, constraints, and approach when useful>

Constraint: <constraint that shaped the decision>
Rejected: <alternative> | <reason>
Confidence: <low|medium|high>
Scope-risk: <narrow|moderate|broad>
Directive: <future maintainer guidance>
Tested: <verification performed>
Not-tested: <known gap>
```

Use only trailers that add real signal. Keep the first line concise and intent-focused.

## Approval And Execution

Before creating a commit, make sure the user can see:

- files included
- files intentionally excluded
- proposed message
- verification performed
- known gaps or risks

When the user has already asked to commit and the commit unit is safe, this can be part of the final execution summary rather than a separate approval prompt.

## Output

When preparing only:

```text
Commit unit:
Excluded:
Proposed message:
Verification:
Risks:
```

When committing:

```text
Commit:
Message:
Files:
Verification:
Risks:
```
