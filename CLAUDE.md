# Claude Code Style Kit Maintenance

This repository defines reusable Claude Code plugin assets. Keep the core plugin project-agnostic.

Rules:

- Standards are the source of truth.
- Skills apply standards to specific workflows.
- Agents provide role-specific review and execution lenses.
- Commands stay thin and route to skills or agents.
- Do not add company, client, product, domain, branch, or API-registry-specific rules to the core plugin.
- Replace fixed project paths with discovery-based guidance.
- When a rule depends on a specific stack, mark it as conditional and explain how to detect applicability.
- Do not make third-party companion plugins required for core behavior.
