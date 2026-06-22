---
description: Daily optimizer that identifies a high-AIC agentic workflow, audits its runs, and recommends efficiency improvements including inline sub-agent refactors when warranted
on:
  schedule:
    - cron: "daily around 14:00 on weekdays"
  workflow_dispatch:
permissions:
  contents: read
  actions: read
  issues: read
  pull-requests: read
tracker-id: agentic-token-optimizer
tools:
  github:
    mode: gh-proxy
    toolsets: [issues]
  bash:
    - "*"
  repo-memory:
    branch-name: "memory/token-audit"
    description: "Historical daily workflow AIC snapshots (shared with agentic-token-audit)"
    file-glob: ["*.json", "*.jsonl", "*.csv", "*.md"]
    max-file-size: 102400
    max-patch-size: 51200
safe-outputs:
  create-issue:
    expires: 7d
    title-prefix: "[agentic-token-optimizer] "
    close-older-issues: true
    max: 1
  threat-detection: false
timeout-minutes: 30
steps:
  - name: Download recent agentic workflow logs
    env:
      GH_TOKEN: ${{ secrets.GH_AW_GITHUB_TOKEN || secrets.GITHUB_TOKEN }}
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/token-audit
      PARTS_DIR=/tmp/gh-aw/token-audit/log-parts
      mkdir -p "$PARTS_DIR"

      echo "📥 Downloading agentic workflow logs (last 7 days)..."

      # Determine which repositories to scan. By default this is just the
      # current repository (single-repo behavior, unchanged). When
      # `.github/agentic-ops.yml` lists `repos:`, scan each of them and
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

      # The repository that develops the AIC monitoring family (audit + optimizer).
      # In that repo the family workflows are valid optimization targets; in any
      # other repo they are excluded (optimize them in the source repo, not here).
      SOURCE_REPO=""
      if [ -f "$CONFIG_FILE" ]; then
        SOURCE_REPO=$(sed 's/#.*$//' "$CONFIG_FILE" | grep -E '^source-repo:' \
          | grep -oE '[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+' | head -n 1 || true)
      fi
      if [ -z "$SOURCE_REPO" ]; then
        SOURCE_REPO="githubnext/agentic-ops"
      fi
      echo "🗂️ Scanning repositories:"
      printf '  - %s\n' "${REPOS[@]}"
      echo "ℹ️ AIC monitoring family source repo: $SOURCE_REPO"

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
          --start-date -7d \
          --json \
          -c 50 \
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
      # The AIC monitoring family is skipped everywhere except the configured
      # source repo; the post-merge filter below is the safety net.
      for repo in "${REPOS[@]}"; do
        [ -n "$repo" ] || continue
        if [ "$repo" = "${GITHUB_REPOSITORY:-}" ] && [ -d .github/workflows ]; then
          for wf in .github/workflows/*.md; do
            [ -f "$wf" ] || continue
            wfid=$(sed -n 's/^tracker-id:[[:space:]]*//p' "$wf" | head -n 1 | tr -d '\r' | sed 's/[[:space:]]*$//')
            [ -n "$wfid" ] || continue
            if [ "$repo" != "$SOURCE_REPO" ] && { [ "$wfid" = "agentic-token-optimizer" ] || [ "$wfid" = "agentic-token-audit" ]; }; then
              echo "⏭️ Skipping $repo :: $wfid (AIC monitoring family — optimize in $SOURCE_REPO)"
              continue
            fi
            FOUND_WORKFLOW=1
            collect_one "$repo" "$wfid"
          done
        else
          while IFS= read -r wfname; do
            [ -n "$wfname" ] || continue
            if [ "$repo" != "$SOURCE_REPO" ] && { [ "$wfname" = "Agentic Workflow AIC Usage Optimizer" ] || [ "$wfname" = "Daily Agentic Workflow AIC Usage Audit" ]; }; then
              echo "⏭️ Skipping $repo :: $wfname (AIC monitoring family — optimize in $SOURCE_REPO)"
              continue
            fi
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
        ' "$PARTS_DIR"/*.json > /tmp/gh-aw/token-audit/all-runs.json
        TOTAL=$(jq '.runs | length' /tmp/gh-aw/token-audit/all-runs.json)
        echo "✅ Downloaded $TOTAL agentic workflow runs (last 7 days)"
      else
        if [ "$FOUND_WORKFLOW" -eq 0 ]; then
          echo "⚠️ No agentic workflow sources found under .github/workflows"
        fi
        echo '{"runs":[],"summary":{}}' > /tmp/gh-aw/token-audit/all-runs.json
      fi

      # Defensively drop any monitoring-family runs that slipped through from
      # repositories other than the source repo (e.g. a stale window or a renamed
      # workflow). Family runs from the source repo are retained as valid targets.
      BEFORE_COUNT=$(jq '(.runs // []) | length' /tmp/gh-aw/token-audit/all-runs.json)
      jq --arg src "$SOURCE_REPO" '
          (.runs // [])
          | map(select(
              ((.repository // "") == $src)
              or (
                (.workflow_path // "") != ".github/workflows/agentic-token-optimizer.lock.yml"
                and (.workflow_path // "") != ".github/workflows/agentic-token-audit.lock.yml"
                and (.workflow_name // "") != "Agentic Workflow AIC Usage Optimizer"
                and (.workflow_name // "") != "Daily Agentic Workflow AIC Usage Audit"
              )
            )) as $runs
          | {
              summary: {
                total_runs: ($runs | length),
                total_tokens: ($runs | map(.token_usage // 0) | add // 0),
                total_aic: ($runs | map(.aic // 0) | add // 0)
              },
              runs: $runs
            }
      ' /tmp/gh-aw/token-audit/all-runs.json > /tmp/gh-aw/token-audit/all-runs.filtered.json
      mv /tmp/gh-aw/token-audit/all-runs.filtered.json /tmp/gh-aw/token-audit/all-runs.json
      AFTER_COUNT=$(jq '(.runs // []) | length' /tmp/gh-aw/token-audit/all-runs.json)
      if [ "$BEFORE_COUNT" -ne "$AFTER_COUNT" ]; then
        echo "🚫 Excluded AIC monitoring family from candidate pool: $((BEFORE_COUNT - AFTER_COUNT)) run(s) removed"
      fi

  - name: Aggregate top workflows by AIC usage
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/token-audit

      jq '{
        generated_at: (now | todateiso8601),
        window_days: 7,
        top_workflows: (
          [.runs[]
            | select(.status == "completed")
              | select((.aic // 0) > 0)
            | {
                repo: (.repository // ""),
                workflow_name: .workflow_name,
                workflow_path: (.workflow_path // ""),
                ai_credits: (.aic // 0),
                tokens: (.token_usage // 0),
                turns: (.turns // 0),
                action_minutes: (.action_minutes // 0)
              }
          ]
          | group_by([.repo, .workflow_name])
          | map({
              repo: .[0].repo,
              workflow_name: .[0].workflow_name,
              workflow_path: (map(.workflow_path) | map(select(. != "")) | (.[0] // "")),
              run_count: length,
              total_ai_credits: (map(.ai_credits) | add),
              avg_ai_credits: ((map(.ai_credits) | add) / length),
              total_tokens: (map(.tokens) | add),
              avg_tokens: ((map(.tokens) | add) / length),
              total_turns: (map(.turns) | add),
              total_action_minutes: (map(.action_minutes) | add)
            })
          | sort_by(.total_ai_credits)
          | reverse
          | .[:10]
        )
      }' /tmp/gh-aw/token-audit/all-runs.json > /tmp/gh-aw/token-audit/top-workflows.json

      echo "✅ Generated top workflow summary at /tmp/gh-aw/token-audit/top-workflows.json"
      jq '.top_workflows' /tmp/gh-aw/token-audit/top-workflows.json

  - name: Load optimization history
    run: |
      set -euo pipefail

      OPT_LOG="/tmp/gh-aw/repo-memory/default/optimization-log.json"
      if [ -f "$OPT_LOG" ]; then
        echo "✅ Previous optimizations:"
        jq -r '.[] | "\(.date): \(.workflow_name)"' "$OPT_LOG"
      else
        echo "ℹ️ No previous optimization history found."
      fi
---

# Agentic Workflow AIC Usage Optimizer

You are the Agentic Workflow Optimizer. Pick one high-AIC workflow, audit recent runs, and create a conservative optimization issue with measurable improvements. Your recommendations may include prompt, tool, reliability, setup-prefix, and inline sub-agent improvements when the evidence supports them.

## Objectives

1. Select one workflow using repo-memory and pre-aggregated data.
2. Analyze AIC, tokens, turns, errors, tool usage patterns, and prompt structure across multiple runs.
3. Propose safe, high-impact optimizations with evidence, including inline sub-agent refactors only when they are a clear fit.
4. Publish one issue and update optimization history.

## Data Access Guidelines

All GitHub API access goes through the `gh` CLI via the cli-proxy — there are **no GitHub MCP tools** available. Always filter API responses with `--jq` or pipe through `jq` to extract only the fields you need. Loading full JSON payloads into context wastes tokens; every extra field is overhead.

**Preferred patterns:**

```bash
REPO="${{ github.repository }}"

# ✅ Extract only the fields you need from a file
gh api "repos/$REPO/contents/.github/workflows/my-workflow.md" \
  --jq '.content' | base64 -d

# ✅ List workflow runs — keep only essential metadata
gh api "repos/$REPO/actions/workflows/my-workflow.yml/runs?per_page=10" \
  --jq '.workflow_runs[] | {id, name, conclusion, run_started_at}'

# ✅ Combine multi-step reads into one bash block with pipes
gh api "repos/$REPO/contents/.github/workflows/my-workflow.md" \
  --jq '.content' | base64 -d | sed -n '1,/^---$/{ /^---$/d; p }' | head -40

# ❌ Never load full unfiltered responses — drops everything into context
gh api "repos/$REPO/actions/workflows/my-workflow.yml/runs"
```

Prefer `--jq` on `gh api` calls over a separate `| jq` step when the filter is simple — it avoids piping the full response through the shell. Use `| jq` for multi-step transformations or when chaining with other commands.

## Data Inputs

- `/tmp/gh-aw/token-audit/all-runs.json`: full 7-day run data (`gh aw logs --json`).
- `/tmp/gh-aw/token-audit/top-workflows.json`: pre-aggregated top 10 workflows by total AIC.
- `/tmp/gh-aw/repo-memory/default/YYYY-MM-DD.json`: daily audit snapshots.
- `/tmp/gh-aw/repo-memory/default/optimization-log.json`: prior optimizations (if present).

Treat missing numeric fields (`aic`, `token_usage`, `turns`, `action_minutes`) as `0`.

## Phase 1 — Select Target

- Start from `top-workflows.json`.
- Exclude workflows optimized in the last 14 days (use `optimization-log.json`).
- Exclude the AIC monitoring family — the `agentic-token-optimizer` and `agentic-token-audit` workflows (display names "Agentic Workflow AIC Usage Optimizer" and "Daily Agentic Workflow AIC Usage Audit") — for every repository **except the configured `source-repo`** (the repository that ships them; defaults to `githubnext/agentic-ops`). In other repositories these workflows are not valid optimization targets; suggestions for them belong in the source repo. They are pre-filtered from `all-runs.json` and `top-workflows.json`, but never select them even if a stale snapshot still lists them.
- Choose the highest AI-credit-spend workflow that remains. Candidates may span multiple repositories — each entry in `top-workflows.json` carries a `repo` field identifying where the workflow lives; preserve it through your analysis and reference it in the issue.
- If no snapshot/history exists, derive candidates directly from `all-runs.json` (grouping by `repository` + `workflow_name`).

Then collect run-level data for the selected workflow:

- run count
- total and average AIC
- total and average tokens
- total and average turns
- conclusions/error patterns

## Phase 2 — Analyze Runtime Behavior

Use this compact analysis matrix:

| Area | Required checks | Output |
|---|---|---|
| Tool usage | Compare configured tools from workflow source vs observed usage across multiple runs | Keep / Consider removing / Remove |
| AI credit spend | Evaluate AIC, token totals, cache efficiency, turns | Top spend drivers |
| Reliability | Repeated errors, warnings, retries, missing tools | AIC waste from failures |
| Prompt efficiency | Redundant instructions, overlong sections, avoidable iteration | Prompt reduction opportunities |
| Structural optimization | Repeated setup/tool-call prefixes and sections suited for inline sub-agents | Extract setup / Add sub-agent / Keep in main agent |

### Tool-Usage Efficiency Patterns

When auditing runs, check for these common anti-patterns that waste tokens:

- **Batch independent reads**: sequential file reads or API calls that could be requested in a single block
- **Chain bash commands**: separate bash calls that could be combined with `&&`
- **Prefer typed tools**: `bash cat`, `bash grep`, `bash find -name` where a more concise tool exists
- **Consolidate GitHub API sequences**: multiple `gh api` calls that could be reduced with `jq` filtering
- **Don't retry without diagnosing**: blind retries of the same failing operation without error analysis

Rules:

- Audit at least 5 runs when available before removal recommendations.
- Never recommend removing a tool used in any successful run unless there is strong contrary evidence.
- Only recommend inline sub-agents when the target workflow has no existing `## agent:` blocks and at least 3 major prompt sections.
- Prioritize highest expected savings first.

## Phase 3 — Read Workflow Source

Use `gh api` with `--jq` (via cli-proxy) to read the target workflow `.md` source. Extract only the sections you need — do not load the whole file if a targeted slice is sufficient.

```bash
# Read the SELECTED target workflow's source from its OWN repository. `repo` and
# `workflow_path` come from the chosen entry in top-workflows.json; the .md source
# is the .lock.yml path with the extension swapped. For single-repo audits this is
# simply the current repository.
REPO="<selected workflow's repo — owner/repo from top-workflows.json>"
WF_PATH="<selected workflow_path with .lock.yml replaced by .md>"

# Read the full source only when necessary
gh api "repos/$REPO/contents/$WF_PATH" --jq '.content' | base64 -d

# Extract frontmatter only
gh api "repos/$REPO/contents/$WF_PATH" --jq '.content' | base64 -d \
  | awk '/^---$/{n++; if(n==2) exit} n==1'

# Extract the prompt body only
gh api "repos/$REPO/contents/$WF_PATH" --jq '.content' | base64 -d \
  | awk 'f; /^---$/{f=1}'
```

Validate from the source:

- configured tools and feature flags
- imported shared components
- prompt structure and verbosity
- whether the prompt already uses inline sub-agents
- network/sandbox constraints relevant to recommendations

## Phase 4 — Structural Optimization Checks

### Common Setup Prefix Analysis

Split the prompt body into major sections (`##` and `###`). For each section, inspect the first 10 lines and note explicit setup instructions, tool invocations, file reads, or repeated shell snippets.

A setup extraction recommendation is warranted only when:

- at least 2 sections repeat the same opening tool calls or setup instructions, and
- moving them into a shared `## Setup` section would not change later section behavior.

If you recommend this optimization, capture:

- the shared setup text (quote the exact calls)
- the affected sections
- the proposed `## Setup` section text
- a conservative savings estimate (5–15% per duplicated call removed)

### Inline Sub-Agent Opportunity Analysis

If the workflow has no inline sub-agents yet, score major sections using these dimensions:

| Dimension | Meaning | Max |
|---|---|---|
| Independence | Can the section run without outputs from other sections? | 3 |
| Small-model adequacy | Is the work mostly extractive, classificatory, or formatting? | 3 |
| Parallelism | Could it run concurrently with other sections? | 2 |
| Size | Is the task substantial enough to justify an agent call? | 2 |

Scoring guidance:

- `6+`: strong candidate
- `4–5`: moderate candidate
- `<4`: keep in the main agent

Smaller models are a good fit for:

- summarizing one file or one code section
- extracting specific fields from structured text
- classifying items into a fixed set of categories
- checking whether something meets a stated criterion
- formatting already-derived data into tables or templates

Keep with the main agent when the section requires cross-referencing multiple heterogeneous sources, strategic synthesis, or writing the final authoritative issue body.

Recommend at most 3 inline sub-agents, and only when the combined opportunity is clearly material. Keep any proposed agent prompt concise and imperative.

## Phase 5 — Publish Optimization Issue

Create one issue with:

- **Target workflow (and its repository) + reason selected**
- **Analysis period + runs analyzed**
- **Spend profile table** (total AIC, avg AIC/run, total tokens, avg turns/run, cache efficiency)
- **Ranked recommendations** with:
  - title
  - estimated AIC savings per run
  - concrete action
  - evidence from observed runs
- **Optional structural optimizations** for shared setup prefixes and inline sub-agents when supported by the analysis
- **Caveats** (sampling limits, edge cases)

### Report Formatting Requirements

- Use `###` for main sections and `####` for subsections.
- Keep the selected workflow, token profile summary, and ranked recommendations visible without collapsible sections.
- Use `<details><summary>...</summary>` blocks for long supporting tables, raw run evidence, and lower-priority context.
- If you cite specific workflow runs, link them using each run's own `url` field (e.g. `[§12345](<run url>)`) so links resolve correctly even when runs come from multiple repositories. Include up to 3 under `**References:**`.
- If you recommend inline sub-agents, include each candidate's task, why a smaller model fits, score breakdown, and the exact invocation change you want made in the main prompt.

## Phase 6 — Update Optimization Log

Append one entry to `/tmp/gh-aw/repo-memory/default/optimization-log.json`:

`{"date":"YYYY-MM-DD","workflow_name":"...","total_ai_credits_analyzed":F,"total_tokens_analyzed":N,"runs_audited":N,"recommendations_count":N,"subagent_candidates":N,"estimated_aic_savings_per_run":F}`

Use `subagent_candidates` for the count of inline sub-agent candidates you actually recommend in the issue body.

Load the existing array if present, append, keep only the last 30 entries, and save.

## Guardrails

- Use pre-downloaded data; do not re-download logs.
- Keep recommendations evidence-based and low-risk.
- Do not modify audit snapshots; only update `optimization-log.json`.
- If the target workflow already has inline sub-agents, do not recommend adding more unless there is a clearly separate, still-extractive task.
- If no structural optimization is warranted, omit that section rather than padding the issue.
- If the selected workflow lives in another repository and its source cannot be read, continue with run-metrics-based analysis and note the limited source visibility in the issue rather than failing.
