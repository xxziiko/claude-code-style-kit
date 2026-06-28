---
name: commit
description: 프로젝트 커밋 컨벤션에 맞는 커밋 메시지 초안과 커밋 단위를 정리한다.
---

# Commit Command

This command is the repository entrypoint for preparing commits.

## Project-Specific Rules

- Use Conventional Commits: `type(scope): subject`.
- Prefer Korean subjects without a trailing period.
- Analyze staged changes first; if files are already staged, treat staged files as the intended commit unit.
- Do not commit secrets, credentials, `.env` files, or unrelated generated/local artifacts.
- Do not use `--no-verify`.
- Do not create a commit without explicit user approval.

## Workflow

1. Inspect the current branch, staged diff, unstaged diff, untracked files, and recent commits.
2. If files are already staged, treat the staged set as the proposed commit unit.
3. If nothing is staged, propose a commit unit from the working tree and stage only after explicit approval.
4. Reject or exclude secrets, credentials, `.env` files, unrelated local artifacts, and unrelated generated output.
5. Draft a Conventional Commit message that explains the intent of the change.
6. Show the commit unit, message, verification performed, and remaining risk.
7. Commit only after explicit user approval.

## Approval Gate

Before running `git commit`, show:

- Commit message
- Files to be committed
- Verification performed
- Known gaps or risks

Stop if approval is missing, if the file set is ambiguous, or if risky files are included.

## Output Contract

If preparing only:

- Proposed commit unit
- Proposed message
- Verification
- Blocked on human

If committing after approval:

- Commit hash
- Commit message
- Files committed
- Verification

Global commit tooling may be used for mechanics, but this file defines the repository commit contract.
