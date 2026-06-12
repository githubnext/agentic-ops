# `agentic-ops`
> "Audit AI credit spend, surface waste, and optimize agentic workflows with confidence!"

`agentic-ops` is a focused bundle of GitHub Agentic Workflows for teams scaling agentic automation and wanting better visibility into usage, trends, and optimization opportunities. Instead of guessing which workflows are driving the highest AI credit spend or where waste is hiding, this package gives you an audit trail, historical reporting, and conservative recommendations you can review before making changes.

## Introduction

It is built for platform engineers, developer productivity teams, and repository maintainers who are scaling agentic workflows and need a practical way to keep them efficient. The bundle helps solve a common problem with AI automation: token usage grows quickly, but the signals for where to improve are scattered across workflow runs and logs. With `agentic-ops`, you get repeatable workflows that make usage measurable, optimization opportunities actionable, and efficiency work easier to operationalize.

## Key Features

- **Clear operational visibility** with a daily audit that captures AI credit spend, usage trends, and workflow-level hotspots.
- **Actionable optimization guidance** that identifies high-AIC workflows and proposes safe, conservative improvements.
- **Faster efficiency improvements** by helping teams find waste before it becomes recurring operational overhead.
- **Built for real GitHub workflows** using GitHub Agentic Workflows, so installation and adoption fit naturally into existing repositories.
- **Useful historical context** through shared snapshots that support trend analysis instead of one-off debugging.
- **A focused bundle** that gives you both measurement and optimization, not just another standalone report.

## Quick Start

Prerequisites:

- Install the [GitHub Agentic Workflows CLI (`gh aw`)](https://github.github.com/gh-aw/introduction/installation/)
- Authenticate GitHub CLI (`gh auth login`) in the repository where you want to install the workflows

Install the package with `gh aw add`:

```bash
gh aw add githubnext/agentic-ops

# Then compile the installed workflows in your repository
gh aw compile
```

This repository publishes a single package at the repository root. You do not need to target a nested package path.

Required configuration after installation:

- No extra secrets are required beyond the default `GITHUB_TOKEN`.
- Review workflow schedules and adjust them to your repository needs before enabling them broadly.

After installation, you can use the included workflows to:

- run a daily audit of AI credit spend and workflow token usage
- identify the workflows consuming the most AI credits
- generate optimization recommendations grounded in recent run data
- surface workflows that are good candidates for inline sub-agent refactors

Included workflows:

| Workflow | What it does |
| ----- | --- |
| [`Daily Agentic Workflow AIC Usage Audit`](https://github.com/githubnext/agentic-ops/blob/main/.github/workflows/agentic-token-audit.md?plain=1) | Collects recent agentic workflow usage and creates a daily AIC spend snapshot. |
| [`Agentic Workflow AIC Usage Optimizer`](https://github.com/githubnext/agentic-ops/blob/main/.github/workflows/agentic-token-optimizer.md?plain=1) | Analyzes high-AIC workflows and proposes conservative efficiency changes, including inline sub-agent opportunities when they are a strong fit. |

## License

MIT
