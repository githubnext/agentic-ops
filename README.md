# agentic-ops — audit token spend, surface waste, and optimize agentic workflows with confidence

[![CI](https://github.com/githubnext/agentic-ops/actions/workflows/ci.yml/badge.svg)](https://github.com/githubnext/agentic-ops/actions/workflows/ci.yml)

`agentic-ops` is a focused bundle of GitHub Agentic Workflows for teams scaling agentic automation and wanting better visibility into cost, usage, and optimization opportunities. Instead of guessing which workflows are expensive or where token waste is hiding, this package gives you an audit trail, historical reporting, and conservative recommendations you can review before making changes.

## Introduction

It is built for platform engineers, developer productivity teams, and repository maintainers who are scaling agentic workflows and need a practical way to keep them efficient. The bundle helps solve a common problem with AI automation: token usage grows quickly, but the signals for where to improve are scattered across workflow runs and logs. With `agentic-ops`, you get repeatable workflows that make usage measurable, optimization opportunities actionable, and efficiency work easier to operationalize.

## Key Features

- **Clear operational visibility** with a daily token audit that captures usage, cost, trends, and workflow-level hotspots.
- **Actionable optimization guidance** that identifies high-token workflows and proposes safe, conservative improvements.
- **Faster cost control** by helping teams find waste before it becomes a recurring operational expense.
- **Built for real GitHub workflows** using GitHub Agentic Workflows, so installation and adoption fit naturally into existing repositories.
- **Useful historical context** through shared snapshots that support trend analysis instead of one-off debugging.
- **A focused bundle** that gives you both measurement and optimization, not just another standalone report.

## Quick Start

Install the bundle with `gh aw add`:

```bash
gh aw add githubnext/agentic-ops/copilot-token-audit githubnext/agentic-ops/copilot-token-optimizer

# Then compile the installed workflows in your repository
gh aw compile
```

After installation, you can use the included workflows to:

- run a daily audit of workflow token usage
- identify the workflows consuming the most tokens
- generate optimization recommendations grounded in recent run data

Included workflows:

| Workflow | What it does |
| ----- | --- |
| [`Daily Token Usage Audit`](https://github.com/githubnext/agentic-ops/blob/main/workflows/copilot-token-audit.md?plain=1) | Collects recent workflow token usage, stores historical snapshots, and publishes a daily audit summary. |
| [`Token Usage Optimizer`](https://github.com/githubnext/agentic-ops/blob/main/workflows/copilot-token-optimizer.md?plain=1) | Analyzes high-token workflows and recommends conservative efficiency improvements backed by recent run data. |

Release history lives in [CHANGELOG.md](CHANGELOG.md).

## License

MIT
