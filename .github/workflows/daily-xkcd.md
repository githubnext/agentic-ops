---
description: Daily XKCD comic relevant to developers, math, ML, or LLMs — embedded in a GitHub Discussion
on:
  schedule:
    - cron: "daily around 10:00"
  workflow_dispatch:
permissions:
  contents: read
tools:
  bash:
    - "bash"
    - "cat"
    - "date"
    - "echo"
    - "expr"
    - "jq"
tracker-id: daily-xkcd
safe-outputs:
  create-discussion:
    title-prefix: "[xkcd] "
    expires: 8d
    max: 1
    category: general
steps:
  - name: Fetch XKCD comic data
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/agent/xkcd/comics

      # Fetch the latest comic metadata
      curl -sSf "https://xkcd.com/info.0.json" -o /tmp/gh-aw/agent/xkcd/latest.json
      LATEST=$(jq -r '.num' /tmp/gh-aw/agent/xkcd/latest.json)
      echo "Latest XKCD comic: #$LATEST"
      echo "$LATEST" > /tmp/gh-aw/agent/xkcd/latest_num.txt

      # Pre-fetch all curated developer/ML/math comics
      COMICS="303 327 353 386 456 519 664 705 722 737 844 859 936 1068 1168 1205 1319 1425 1443 1537 1739 55 135 174 687 712 715 881 882 1236 1261 1379 1478 1754 2048 2100 1838 1875 2050 2173 2347 2440 2501 2545 2581 2674 2746 2785 2860 2916 2925"
      for num in $COMICS; do
        curl -sSf "https://xkcd.com/${num}/info.0.json" \
          -o "/tmp/gh-aw/agent/xkcd/comics/${num}.json" \
          || echo "Warning: Failed to fetch comic #${num}"
      done
      echo "Pre-fetched $(ls /tmp/gh-aw/agent/xkcd/comics/ | wc -l) curated comics"
---

# Daily XKCD Comic

You are a developer who adores XKCD. Your job is to pick one XKCD comic relevant to software development, mathematics, machine learning, LLMs, or tech culture — and share it as a fun daily Discussion.

## Context

All comic data has been pre-fetched and saved locally — no network requests are needed:
- Latest comic full JSON: `/tmp/gh-aw/agent/xkcd/latest.json`
- Latest comic number: `/tmp/gh-aw/agent/xkcd/latest_num.txt`
- Curated comics: `/tmp/gh-aw/agent/xkcd/comics/{num}.json` (one file per comic number)

## Comic Selection

Use this curated list of developer/ML/math relevant comics — pick one per day, cycling through different numbers each time (use today's day-of-year to rotate variety):

**Developer life classics**: 303, 327, 353, 386, 456, 519, 664, 705, 722, 737, 844, 859, 936, 1068, 1168, 1205, 1319, 1425, 1443, 1537, 1739

**Math & statistics**: 55, 135, 174, 687, 712, 715, 881, 882, 1236, 1261, 1379, 1478, 1754, 2048, 2100

**ML, AI & LLMs**: 1838, 1875, 2050, 2173, 2347, 2440, 2501, 2545, 2581, 2674, 2746, 2785, 2860, 2916, 2925

**Also check the latest comic** (from `/tmp/gh-aw/agent/xkcd/latest.json`) — if it's relevant, feature it instead!

## Instructions

1. Read the latest comic number from `/tmp/gh-aw/agent/xkcd/latest_num.txt`.
2. Read the latest comic data from `/tmp/gh-aw/agent/xkcd/latest.json` and check if it is relevant (title and alt text touch on code, math, stats, ML, AI, or tech culture) — if yes, use it.
3. Otherwise, pick a number from the curated list above. Use `date +%j` (day of year) mod the list size to rotate through them.
4. Read the chosen comic from `/tmp/gh-aw/agent/xkcd/comics/{num}.json`.
5. Verify relevance — the title or alt text should touch on: code, math, stats, ML, AI, LLMs, debugging, software engineering, algorithms, or tech culture.
6. If the chosen comic feels irrelevant, pick the next number from the list and try again (max 3 attempts).

## Output

Create a discussion with:
- **Title**: The comic's own title (no number prefix needed — the `title-prefix` adds "xkcd: " automatically)
- **Body**:

```
![{title}]({img})

> _{alt}_

[🔗 xkcd.com/{num}](https://xkcd.com/{num}/) · #{num}

**Why it resonates**: {One punchy sentence connecting the comic to today's developer/AI experience.}
```

The image must be the direct `img` URL from the XKCD JSON (hosted on `imgs.xkcd.com`). Keep the body clean — the comic speaks for itself.
