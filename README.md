# agentic-ops

[![CI](https://github.com/githubnext/agentic-ops/actions/workflows/ci.yml/badge.svg)](https://github.com/githubnext/agentic-ops/actions/workflows/ci.yml)

This repo contains a small set of GitHub Agentic Workflows for auditing Copilot token usage and highlighting workflows that should be optimized.

## Usage

To add one of these workflows to your repo, use `gh aw add <owner>/<repo>/<workflow-name>`.

```bash
gh aw add githubnext/agentic-ops/copilot-token-audit githubnext/agentic-ops/copilot-token-optimizer
```

This adds the workflow to `.github/workflows/`. For guided setup, use `gh aw add-wizard githubnext/agentic-ops/copilot-token-audit`.

Release history lives in [CHANGELOG.md](CHANGELOG.md).

## Workflows

| Workflow | What it does |
| ----- | --- |
| [`Daily Copilot Token Usage Audit`](https://github.com/githubnext/agentic-ops/blob/main/workflows/copilot-token-audit.md?plain=1) | Collects recent Copilot workflow usage and creates a daily audit snapshot. |
| [`Copilot Token Usage Optimizer`](https://github.com/githubnext/agentic-ops/blob/main/workflows/copilot-token-optimizer.md?plain=1) | Analyzes expensive workflows and proposes conservative token-reduction changes. |
