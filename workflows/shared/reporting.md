---
# Reporting Shared Component
# Provides common issue-reporting conventions for agentic workflows.
#
# Usage:
#   imports:
#     - shared/reporting.md
#
# This import provides:
# - Standard formatting guidelines for issues and discussions
# - Common markdown templates for tables, summaries, and recommendations

safe-outputs:
  create-issue:
    close-older-issues: true
    expires: 7d
---

# Reporting Guidelines

Use the following conventions when generating issues, discussions, or step summaries.

## Issue Structure

Every generated issue should include:

1. **Executive Summary** — two-to-three sentence overview of findings.
2. **Data Table** — key metrics in a compact markdown table.
3. **Recommendations** — ranked list with estimated impact and concrete action.
4. **Caveats** — sampling limits, missing data, or edge cases that affect confidence.

Use `<details>` blocks for long supporting tables to keep the top-level view concise.

## Markdown Conventions

- Use `**bold**` for metric names and important values.
- Format large numbers with commas (e.g., `1,234,567`).
- Format costs as `$X.XX` (two decimal places).
- Use emoji sparingly: ✅ success, ⚠️ warning, ❌ failure, 📊 data, 💡 insight.
- Link to relevant runs, PRs, or issues where possible.

## Table Template

```markdown
| Metric | Value | Notes |
|---|---|---|
| Total runs | N | Last N days |
| Total tokens | N | Formatted with commas |
| Total cost | $X.XX | USD |
| Avg tokens/run | N | |
```

## Recommendations Template

```markdown
### Recommendations

1. **[Title]** — estimated savings: ~N tokens/run
   - **Action**: ...
   - **Evidence**: ...

2. **[Title]** — estimated savings: ~N tokens/run
   - **Action**: ...
   - **Evidence**: ...
```
