---
description: Daily audit of AI Credit (AIC) usage across all agentic workflows with historical trend tracking
on:
  schedule:
    - cron: "daily around 12:00 on weekdays"
  workflow_dispatch:
permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read
network:
  allowed:
    - defaults
    - python
tracker-id: agentic-token-audit
safe-outputs:
  create-issue:
    expires: 3d
    title-prefix: "[agentic-token-audit] "
    max: 1
    close-older-issues: true
  upload-asset:
    max: 5
    allowed-exts: [.png, .jpg, .jpeg, .svg]
tools:
  agentic-workflows:
  bash:
    - "*"
  repo-memory:
    branch-name: "memory/token-audit"
    description: "Historical daily workflow AIC snapshots (shared with agentic-token-optimizer)"
    file-glob: ["*.json", "*.jsonl", "*.csv", "*.md"]
    max-file-size: 102400
    max-patch-size: 51200
steps:
  - name: Setup Python
    uses: actions/setup-python@v6.2.0
    with:
      python-version: "3.12"
  - name: Setup local chart workspace
    run: |
      mkdir -p /tmp/gh-aw/token-audit/charts /tmp/gh-aw/token-audit/site-packages
  - name: Install Python chart dependencies
    run: |
      python3 -m pip install --quiet --target /tmp/gh-aw/token-audit/site-packages pandas matplotlib seaborn
  - name: Download agentic workflow logs
    env:
      GH_TOKEN: ${{ secrets.GH_AW_GITHUB_TOKEN || secrets.GITHUB_TOKEN }}
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/token-audit
      PARTS_DIR=/tmp/gh-aw/token-audit/log-parts
      mkdir -p "$PARTS_DIR"

      # Determine which repositories to audit. By default this is just the
      # current repository (single-repo behavior, unchanged). When
      # `.github/agentic-ops.yml` lists `repos:`, audit each of them and
      # aggregate centrally. See the README "Auditing multiple repositories".
      CONFIG_FILE=".github/agentic-ops.yml"
      REPOS=()
      if [ -f "$CONFIG_FILE" ]; then
        while IFS= read -r repo_line; do
          if [ -n "$repo_line" ]; then REPOS+=("$repo_line"); fi
        done < <(awk '/^repos:[[:space:]]*$/{f=1;next} /^[^[:space:]#]/{f=0} f' "$CONFIG_FILE" \
          | sed 's/#.*$//' \
          | grep -oE '[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+' || true)
      fi
      if [ "${#REPOS[@]}" -eq 0 ]; then
        REPOS=("${GITHUB_REPOSITORY:-}")
      fi
      echo "🗂️ Auditing repositories:"
      printf '  - %s\n' "${REPOS[@]}"

      # Fetch one workflow's logs, stamp each run with its source repository, and
      # keep the part only if it has runs. $1=repo, $2=workflow identifier; any
      # further args are passed through to `gh aw logs` (e.g. --repo for other repos).
      FOUND_WORKFLOW=0
      collect_one() {
        local repo="$1" wfid="$2"; shift 2
        local safe_repo safe_id part exit_code count
        safe_repo=$(printf '%s' "$repo" | tr -cs 'A-Za-z0-9._-' '_')
        safe_id=$(printf '%s' "$wfid" | tr -cs 'A-Za-z0-9._-' '_')
        part="$PARTS_DIR/${safe_repo}__${safe_id}.json"
        exit_code=0
        gh aw logs "$wfid" "$@" \
          --start-date -1d \
          --json \
          -c 100 \
          > "$part" || exit_code=$?
        if ! jq -e . "$part" >/dev/null 2>&1; then
          echo "⚠️ $repo :: $wfid: invalid log JSON (exit code $exit_code)"
          rm -f "$part"
          return 0
        fi
        # Stamp each run with its source repository for cross-repo aggregation.
        if jq --arg repo "$repo" '.runs = ((.runs // []) | map(.repository //= $repo))' \
          "$part" > "$part.tagged"; then
          mv "$part.tagged" "$part"
        else
          rm -f "$part.tagged"
        fi
        count=$(jq '(.runs // []) | length' "$part")
        if [ "$count" -gt 0 ]; then
          echo "✅ $repo :: $wfid: downloaded $count run(s) (exit code $exit_code)"
        else
          echo "⚠️ $repo :: $wfid: no log data (exit code $exit_code)"
          rm -f "$part"
        fi
      }

      # Fetch logs per workflow (avoids repo-wide pagination truncation in busy
      # repos). For the current repo, resolve agentic workflows from the local
      # checkout by tracker-id — unchanged single-repo behavior. For any other
      # repo, resolve them by display name via the GitHub Actions API and pass
      # --repo, because `gh aw logs` resolves a remote workflow only by its name.
      for repo in "${REPOS[@]}"; do
        [ -n "$repo" ] || continue
        if [ "$repo" = "${GITHUB_REPOSITORY:-}" ] && [ -d .github/workflows ]; then
          for wf in .github/workflows/*.md; do
            [ -f "$wf" ] || continue
            wfid=$(sed -n 's/^tracker-id:[[:space:]]*//p' "$wf" | head -n 1 | tr -d '\r' | sed 's/[[:space:]]*$//')
            [ -n "$wfid" ] || continue
            FOUND_WORKFLOW=1
            collect_one "$repo" "$wfid"
          done
        else
          while IFS= read -r wfname; do
            [ -n "$wfname" ] || continue
            FOUND_WORKFLOW=1
            collect_one "$repo" "$wfname" --repo "$repo"
          done < <(gh api "repos/$repo/actions/workflows?per_page=100" \
            --jq '.workflows[] | select(.path | endswith(".lock.yml")) | .name' 2>/dev/null || true)
        fi
      done

      if [ "$FOUND_WORKFLOW" -eq 1 ] && ls "$PARTS_DIR"/*.json >/dev/null 2>&1; then
        jq -s '
          (map(.runs // []) | add // [] | unique_by([.repository, .run_id])) as $runs |
          {
            summary: {
              total_runs: ($runs | length),
              total_tokens: ($runs | map(.token_usage // 0) | add // 0),
              total_aic: ($runs | map(.aic // 0) | add // 0)
            },
            runs: $runs
          }
        ' "$PARTS_DIR"/*.json > /tmp/gh-aw/token-audit/workflow-logs.json
        TOTAL=$(jq '.runs | length' /tmp/gh-aw/token-audit/workflow-logs.json)
        echo "✅ Downloaded $TOTAL agentic workflow runs (last 24 hours)"
      else
        if [ "$FOUND_WORKFLOW" -eq 0 ]; then
          echo "⚠️ No agentic workflow sources found under .github/workflows"
        fi
        echo '{"runs":[],"summary":{}}' > /tmp/gh-aw/token-audit/workflow-logs.json
      fi
timeout-minutes: 25
---

# Daily Agentic Workflow AIC Usage Audit

You are the Agentic Workflow Auditor — a workflow that tracks daily AI Credit (AIC) spend and token consumption across all agentic workflows in this repository and maintains a historical record for trend analysis.

## Mission

1. Parse the pre-downloaded agentic workflow logs and compute per-workflow AIC spend and token usage metrics.
2. Persist today's snapshot to repo-memory so the optimizer (and future runs of this audit) can read historical data.
3. Publish a concise audit issue summarizing today's AIC spend and trend highlights.

## Data Sources

### Pre-downloaded logs

The workflow logs are at `/tmp/gh-aw/token-audit/workflow-logs.json`. The file is the raw JSON output of `gh aw logs --json` with this top-level shape:

```json
{
  "summary": { "total_runs": N, "total_tokens": N, ... },
  "runs": [ ... ],
  "tool_usage": [ ... ],
  "mcp_tool_usage": { ... },
  ...
}
```

Each element of `.runs` is a `RunData` object with (among others):

| Field | Type | Notes |
|---|---|---|
| `workflow_name` | string | Human-readable name |
| `workflow_path` | string | `.github/workflows/....lock.yml` |
| `repository` | string | `owner/repo` the run belongs to (used to group spend by repository when auditing multiple repos) |
| `aic` | float | AI Credits (AIC) consumed (primary billing metric; 1 AIC = $0.01 USD) |
| `token_usage` | int | Total tokens (`omitempty` — treat missing/null as 0) |
| `effective_tokens` | int | Legacy normalized token metric (deprecated; use `aic` for billing) |
| `action_minutes` | float | Billable GitHub Actions minutes |
| `turns` | int | Number of agent turns |
| `duration` | string | Human-readable duration |
| `created_at` | ISO 8601 | Run creation time |
| `run_id` | int64 | Unique run ID |
| `url` | string | Link to the run |
| `status` | string | `completed`, `in_progress`, etc. |
| `conclusion` | string | `success`, `failure`, etc. |
| `error_count` | int | Errors encountered |
| `warning_count` | int | Warnings encountered |
| `token_usage_summary` | object or null | Firewall-level breakdown by model |

### Repo-memory (historical snapshots)

Previous snapshots live at `/tmp/gh-aw/repo-memory/default/`. Each daily snapshot is stored as a JSON file named `YYYY-MM-DD.json` with the schema below.

## Phase 1 — Process Logs

Write a Python script to `/tmp/gh-aw/token-audit/process_audit.py` and run it. The script must:

1. Load `/tmp/gh-aw/token-audit/workflow-logs.json` and extract `.runs`.
2. Filter to `status == "completed"` runs only.
3. Group by `repository` + `workflow_name` (so identically named workflows in different repositories are never conflated) and compute per-workflow aggregates:
   - `repo`, `run_count`, `total_ai_credits`, `avg_ai_credits`, `total_tokens`, `avg_tokens`, `total_turns`, `avg_turns`, `total_action_minutes`, `error_count`, `warning_count`
4. Compute an overall summary: total runs, total AI credits, total tokens, total action minutes.
5. Sort workflows descending by `total_ai_credits`.
6. Save the result to `/tmp/gh-aw/token-audit/audit_snapshot.json` with this shape:

```json
{
  "date": "YYYY-MM-DD",
  "period_days": 30,
  "overall": {
    "total_runs": N,
    "total_ai_credits": F,
    "total_tokens": N,
    "total_action_minutes": F
  },
  "workflows": [
    {
      "repo": "owner/repo",
      "workflow_name": "...",
      "run_count": N,
      "total_ai_credits": F,
      "avg_ai_credits": F,
      "total_tokens": N,
      "avg_tokens": N,
      "total_turns": N,
      "avg_turns": F,
      "total_action_minutes": F,
      "error_count": N,
      "warning_count": N,
      "latest_run_url": "..."
    }
  ]
}
```

Handle null/missing `aic` and `token_usage` by treating them as 0.

When runs span more than one repository, also build a top-level `repos` array — one entry per repository with `repo`, `total_runs`, `total_ai_credits`, `total_tokens`, `total_action_minutes`, and `active_workflows` — so the issue can present a per-repository rollup. When all runs come from a single repository, omit `repos`.

## Phase 2 — Persist Snapshot to Repo-Memory

1. Read the snapshot from `/tmp/gh-aw/token-audit/audit_snapshot.json`.
2. Copy it to `/tmp/gh-aw/repo-memory/default/YYYY-MM-DD.json` (today's UTC date).
3. This file is what the optimizer workflow reads to identify high-usage workflows.

Also maintain a rolling summary file at `/tmp/gh-aw/repo-memory/default/rolling-summary.json` that contains an array of daily overall totals (date, total_ai_credits, total_tokens, total_runs, total_action_minutes, active_workflows) for the last 90 entries. `active_workflows` must be the count of distinct workflows with `run_count >= 1` in that day's snapshot. Load the existing file, append today's entry, trim to 90, and save.

Do not append a synthetic zero-valued entry to `rolling-summary.json` when either of these conditions is true:

- the raw `.runs` array is empty
- the raw `.runs` array is non-empty but there are zero completed runs in the current window

Report those two cases differently in the issue as described below so the empty-window diagnosis stays precise while the historical trend remains unchanged.

## Phase 3 — Generate Charts

Create up to two chart images in `/tmp/gh-aw/token-audit/charts/` using Python, `matplotlib`, and `seaborn` with `whitegrid` styling:

1. **AI credit spend by workflow** (`ai_credits_by_workflow.png`): a horizontal bar chart of the top 15 workflows by total AI credits from `audit_snapshot.json`.
2. **Historical AI credit trend** (`ai_credits_trend.png`): a dual-axis line chart from `rolling-summary.json` with total AI credits on the primary y-axis and active workflows/day on the secondary y-axis.

Chart requirements:

- The preinstalled Python packages live in `/tmp/gh-aw/token-audit/site-packages`. Set `PYTHONPATH=/tmp/gh-aw/token-audit/site-packages${PYTHONPATH:+:$PYTHONPATH}` for every Python command that imports `pandas`, `matplotlib`, or `seaborn`, for example: `PYTHONPATH=/tmp/gh-aw/token-audit/site-packages${PYTHONPATH:+:$PYTHONPATH} python3 /tmp/gh-aw/token-audit/process_audit.py`.
- Use 300 DPI and a white background.
- Add clear axis labels and titles.
- Save only PNG files.
- For `ai_credits_trend.png`, label the secondary y-axis as `Active workflows/day` (or `Distinct workflows executed`) and plot daily distinct executed workflows from `active_workflows`.
- Do not plot the total number of workflows defined in the repository.
- If there are fewer than 2 rolling-summary points, skip the trend chart and explain why in the issue.
- After generating each chart, call `upload_asset` with its file path.
- In the issue template below, replace `UPLOAD_URL_WORKFLOW_PLACEHOLDER` with the URL returned for `ai_credits_by_workflow.png`.
- In the issue template below, replace `UPLOAD_URL_TREND_PLACEHOLDER` with the URL returned for `ai_credits_trend.png`.
- If a chart is skipped, omit that image markdown line entirely instead of leaving a placeholder behind.

## Phase 4 — Publish Audit Issue

Create an issue with these sections:

### Formatting Requirements

- Use `###` for main sections and `####` for subsections inside the issue body.
- Keep the executive summary and final observations visible without collapsible sections.
- Put verbose tables or supporting detail inside `<details><summary>...</summary>` blocks.
- If you cite specific workflow runs, link them using each run's own `url` field (e.g. `[§12345](<run url>)`) so links resolve correctly even when runs come from multiple repositories. Include up to 3 under `**References:**`.

### Report Template

```
### 📊 Executive Summary

- **Period**: last 24 hours (YYYY-MM-DD to YYYY-MM-DD)
- **Repositories audited**: N (include this line only when auditing more than one)
- **Total runs**: N
- **Total AI credits**: N.NN AIC
- **Total tokens**: N (formatted with commas)
- **Total Actions minutes**: X.X min
- **Active workflows**: N

### 🏆 Top 5 Workflows by AI Credit Spend

| Workflow | Runs | Total AI Credits | Avg AI Credits |
|---|---|---|---|
| ... | ... | ... | ... |

When auditing more than one repository, add a leading **Repository** column to this table (and to the full per-workflow breakdown), and include the per-repository section below.

### 🗂️ By Repository

_Include this section only when auditing more than one repository, built from the snapshot `repos` array._

| Repository | Runs | Total AI Credits | Active Workflows |
|---|---|---|---|
| ... | ... | ... | ... |

### 📈 Trends

Embed chart images using uploaded asset URLs when available:

![AI Credit Spend by Workflow](UPLOAD_URL_WORKFLOW_PLACEHOLDER)

![Historical AI Credit Trend](UPLOAD_URL_TREND_PLACEHOLDER)

Summarize AI credit, token, and active-workflow changes from `rolling-summary.json` when historical data is available.

<details>
<summary><b>Full Per-Workflow Breakdown</b></summary>

[Complete table of all workflows sorted by total AI credits]

</details>

### 💡 Observations

- Identify any workflow with >30% of total AI credits as a "heavy hitter"
- Note workflows with high error/warning counts relative to runs
- Flag any workflow whose avg AI credits per run exceeds 1.00 AIC

**Data snapshot**: `memory/token-audit/YYYY-MM-DD.json`
```

## Important Notes

- Use `// 0` (null coalescing) in jq and `.get(field, 0)` in Python for nullable numeric fields (`aic`, `token_usage`).
- Distinguish between these two cases in the issue:
  - the raw `.runs` array is empty
  - the raw `.runs` array is non-empty but none of the runs are `status == "completed"`
- Report those cases differently:
  - if `len(runs) == 0` (or `jq '.runs | length' == 0`), say the collection window returned no runs
  - if `len(runs) > 0` and there are zero completed runs, say the collection window had runs but none completed yet
- Do not claim the raw log file was empty unless you verified `len(runs) == 0` (or `jq '.runs | length' == 0`).
- Keep the issue concise — the optimizer workflow will do the deep analysis.

## Experiment OTEL Span Attributes

If the file `/tmp/gh-aw/experiments/assignments.json` exists, read it and emit one OTEL span
attribute per experiment using the `otlp.cjs` `logSpan` helper:

```javascript
const fs = require('fs');
const assignmentsFile = '/tmp/gh-aw/experiments/assignments.json';
if (fs.existsSync(assignmentsFile)) {
  const assignments = JSON.parse(fs.readFileSync(assignmentsFile, 'utf8'));
  const names = Object.keys(assignments).sort();
  if (names.length > 0) {
    const attrs = { 'gh_aw.experiment.names': names.join(',') };
    for (const name of names) {
      attrs[`gh_aw.experiment.${name}`] = assignments[name];
    }
    const otlp = require('/tmp/gh-aw/actions/otlp.cjs');
    await otlp.logSpan('experiment', attrs);
  }
}
```

This enables filtering workflow runs by experiment variant in Datadog, Honeycomb, or any
OTLP-compatible backend. Attribute keys follow the pattern `gh_aw.experiment.<name>` with the
assigned variant as the value, plus `gh_aw.experiment.names` as a comma-separated index.
