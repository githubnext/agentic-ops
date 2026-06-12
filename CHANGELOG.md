# Changelog

All notable user-facing changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Changelog entries should describe what changed for users of these workflows, not how the repository was maintained behind the scenes.

- Major releases document breaking changes and required migration steps.
- Minor releases document new workflow capabilities and meaningful improvements.
- Patch releases document bug fixes and smaller behavior improvements.
- For pre-1.0 releases, still call out breaking changes plainly; do not hide user impact behind `0.x` versioning.
- Every release should have an entry, even if the release is mostly maintenance.
- Use ISO 8601 dates in headings: `YYYY-MM-DD`.
- Prefer one clear bullet per user-visible outcome over several bullets that mirror commit history.
- Group changes under only the relevant Keep a Changelog sections: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.
- Use `Deprecated` when something still works but users should migrate; use `Removed` when it is gone in this release.
- If a change requires action, say what the user needs to do.
- If a change is breaking, say what breaks and how to adapt.
- Include workflow names, inputs, outputs, or configuration keys only when they help users identify impact quickly.
- Omit internal-only changes such as CI, refactors, tests, docs-only edits, dependency churn, generated lockfile updates, and similar maintenance unless they materially affect users.
- Omit commit-by-commit narration, PR numbers, branch names, file-by-file inventories, and implementation detail that does not change user behavior.
- Omit vague filler like "various fixes" when the actual effect can be named.
- Do not include contributor handles, reviewer names, or other attribution in release entries.

## [0.3.2] - 2026-06-12

### Changed

- Consolidated published workflow sources under `.github/workflows/` and removed the legacy `workflows/` directory. If you reference workflow files by path, update those references to `.github/workflows/...`.

## [0.3.1] - 2026-06-09

### Fixed

- Kept the published audit and optimizer workflows compatible with the current upstream `gh-aw` AIC/runtime log schema updates without changing the existing snapshot and summary fields that downstream reporting depends on.
- Restored the audit workflow's explicit Python network allowlist so chart dependency installation remains supported in the packaged workflow.

## [0.3.0] - 2026-06-08

### Changed

- Updated both workflows to track and report AI credits (AIC) as the primary billing metric, aligned with the GitHub Copilot AI credits billing model introduced on June 1, 2026 (1 AIC = $0.01 USD). The daily audit snapshot and rolling summary now include `total_ai_credits`; audit and optimizer issues now lead with AI credit spend. Effective tokens remain available as a legacy compatibility field in the run data but are no longer the primary reporting metric.

## [0.2.2] - 2026-06-01

### Changed

- Removed dollar-denominated cost estimates from workflow output and documentation. Cost calculations are now expressed only as token counts.

## [0.2.1] - 2026-05-29

### Fixed

- Internal maintenance fix for agent job reliability.

## [0.2.0] - 2026-05-22

### Added

- Support for installing workflows as named sources via `gh aw add owner/repo/workflow-name`.
- Daily Sub-Agent Optimizer workflow for analyzing and optimizing high-token workflows.

### Changed

- Consolidated separate optimizer workflows into a single unified workflow.
- Made all published workflows agent-agnostic, eliminating agent-specific configuration requirements.

## [0.1.1] - 2026-05-14

### Changed

- Simplified the published workflow package so `copilot-token-audit` no longer depends on the removed Copilot requests feature.

### Fixed

- Improved `copilot-token-audit` startup reliability when optional observability settings are not configured.

## [0.1.0] - 2026-05-08

### Added

- Initial published set of GitHub Agentic Workflows for Copilot token auditing and optimization.
- `copilot-token-audit`, which collects recent Copilot workflow usage and stores daily audit snapshots.
- `copilot-token-optimizer`, which analyzes high-token workflows and proposes conservative efficiency improvements.