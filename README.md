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

## Auditing multiple repositories

By default each workflow audits only the repository it runs in. To monitor AI-credit spend across **many repositories from one central repository**, add a `.github/agentic-ops.yml` config to the repo where the workflows run:

```yaml
repos:
  - your-org/repo-a
  - your-org/repo-b
  - your-org/repo-c
```

With `repos` set, the audit and optimizer collect each listed repository's agentic-workflow logs via `gh aw logs --repo` and aggregate them into a single report broken down by repository and workflow. Leave the file out (or leave `repos` empty) to keep the default single-repo behavior — the feature is fully opt-in and backward compatible.

Multi-repo collection reads each listed repository's GitHub Actions API, so it needs a token with **`actions: read` on every listed repo** (the default `GITHUB_TOKEN` only covers the current repository). These workflows use gh-aw's standard [`GH_AW_GITHUB_TOKEN`](https://github.github.com/gh-aw/reference/auth/) "magic" secret — set it to a PAT (classic `repo` scope, or a fine-grained PAT with Actions read) or a GitHub App token with access to the listed repos:

```bash
gh aw secrets set GH_AW_GITHUB_TOKEN --value "<token>"
```

The workflows fall back to `GITHUB_TOKEN` (current repo only) when `GH_AW_GITHUB_TOKEN` is unset.

Optional keys in `.github/agentic-ops.yml`:

- `source-repo` — the repository that develops the audit/optimizer workflows themselves (defaults to `githubnext/agentic-ops`). The optimizer keeps the monitoring workflows eligible for optimization only in that repository and excludes them everywhere else.

## License

MIT
