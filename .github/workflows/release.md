---
name: Release
description: Create a GitHub release with curated, user-facing notes for the published agentic workflows
on:
  roles:
    - admin
    - maintainer
  workflow_dispatch:
    inputs:
      tag:
        description: Release tag to create (e.g., v1.0.0)
        required: true
        type: string
      target:
        description: Branch or commit to release from
        required: false
        default: main
        type: string
permissions:
  contents: read
timeout-minutes: 20
tools:
  bash:
    - "*"
safe-outputs:
  update-release:
  threat-detection: false
jobs:
  release:
    needs: ["pre_activation", "activation"]
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: write
    outputs:
      release_id: ${{ steps.create_release.outputs.release_id }}
      release_tag: ${{ steps.create_release.outputs.release_tag }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4.3.1
        with:
          persist-credentials: false
          ref: ${{ env.RELEASE_TARGET }}
          fetch-depth: 0
        env:
          RELEASE_TARGET: ${{ inputs.target }}

      - name: Set up gh-aw CLI
        uses: github/gh-aw-actions/setup-cli@f8495a686e66770ae977f82732f34d7340ee42a4 # v0.72.1
        with:
          version: v0.72.1
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Create GitHub release
        id: create_release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          RELEASE_TAG: ${{ inputs.tag }}
          RELEASE_TARGET: ${{ inputs.target }}
        run: |
          set -euo pipefail
          gh release create "$RELEASE_TAG" \
            --target "$RELEASE_TARGET" \
            --title "$RELEASE_TAG" \
            --notes "Release notes are being prepared." \
            --latest

          RELEASE_ID=$(gh release view "$RELEASE_TAG" --json databaseId --jq '.databaseId')
          echo "release_id=$RELEASE_ID" >> "$GITHUB_OUTPUT"
          echo "release_tag=$RELEASE_TAG" >> "$GITHUB_OUTPUT"

steps:
  - name: Prepare release context
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      RELEASE_ID: ${{ needs.release.outputs.release_id }}
      RELEASE_TAG: ${{ needs.release.outputs.release_tag }}
      RELEASE_TARGET: ${{ inputs.target }}
    run: |
      set -euo pipefail
      mkdir -p /tmp/gh-aw/release-data

      gh api "/repos/$GITHUB_REPOSITORY/releases/$RELEASE_ID" > /tmp/gh-aw/release-data/current_release.json

      gh release list --limit 10 --json tagName,name,publishedAt,isLatest \
        > /tmp/gh-aw/release-data/releases.json

      jq --arg tag "$RELEASE_TAG" '[.[] | select(.tagName != $tag)][0] // null' \
        /tmp/gh-aw/release-data/releases.json \
        > /tmp/gh-aw/release-data/previous_release.json

      PREVIOUS_TAG=$(jq -r '.tagName // empty' /tmp/gh-aw/release-data/previous_release.json)
      CURRENT_VERSION=${RELEASE_TAG#v}
      PREVIOUS_VERSION=${PREVIOUS_TAG#v}

      jq -n \
        --arg current_tag "$RELEASE_TAG" \
        --arg current_version "$CURRENT_VERSION" \
        --arg previous_tag "$PREVIOUS_TAG" \
        --arg previous_version "$PREVIOUS_VERSION" \
        --arg release_target "$RELEASE_TARGET" \
        '{
          current_tag: $current_tag,
          current_version: $current_version,
          previous_tag: ($previous_tag | select(length > 0)),
          previous_version: ($previous_version | select(length > 0)),
          release_target: $release_target
        }' > /tmp/gh-aw/release-data/semver_context.json

      if [ -n "$PREVIOUS_TAG" ]; then
        # The agent job checkout is shallow by default; fetch full history and tags
        # so that the "$PREVIOUS_TAG..$RELEASE_TARGET" revision range is resolvable.
        SERVER_HOST="${GITHUB_SERVER_URL#https://}"
        git remote set-url origin "https://x-access-token:${GH_TOKEN}@${SERVER_HOST}/${GITHUB_REPOSITORY}.git"
        git fetch --unshallow --tags 2>/dev/null || git fetch --tags
        git remote set-url origin "https://${SERVER_HOST}/${GITHUB_REPOSITORY}.git"

        git log --no-merges --reverse --pretty=format:'%H%x09%s' "$PREVIOUS_TAG..$RELEASE_TARGET" \
          > /tmp/gh-aw/release-data/commit_subjects.tsv
        git diff --name-only "$PREVIOUS_TAG..$RELEASE_TARGET" \
          > /tmp/gh-aw/release-data/changed_files.txt
      else
        : > /tmp/gh-aw/release-data/commit_subjects.tsv
        find workflows -maxdepth 1 -type f -name '*.md' | sort \
          > /tmp/gh-aw/release-data/changed_files.txt
      fi

      find workflows -maxdepth 1 -type f -name '*.md' | sort \
        > /tmp/gh-aw/release-data/workflow_sources.txt

---

# Release Notes Generator

Generate curated release notes for `${RELEASE_TAG}` in `${GITHUB_REPOSITORY}` and replace the placeholder release body.

## Data Available

- `/tmp/gh-aw/release-data/current_release.json` - the release that was just created
- `/tmp/gh-aw/release-data/previous_release.json` - the previous release, or `null` if this is the first one
- `/tmp/gh-aw/release-data/semver_context.json` - current and previous tags with plain semver values
- `/tmp/gh-aw/release-data/commit_subjects.tsv` - non-merge commit subjects since the previous release
- `/tmp/gh-aw/release-data/changed_files.txt` - files changed since the previous release
- `/tmp/gh-aw/release-data/workflow_sources.txt` - the source workflow files covered by this release

## What to Write

Write a concise, human changelog-style markdown body for the release.

Assume the audience is a maintainer or adopter deciding whether to upgrade, rerun, or reconfigure these workflows. Every sentence should help answer one of these questions:

- what changed in behavior or capability?
- does this affect how I use or operate the workflows?
- do I need to take action before or after upgrading?

Use this structure when there is meaningful content:

- `## Highlights` with 1 to 4 bullets focused on what users can now do, what improved, or what they need to know
- `## Upgrade Notes` only when users need to take action, should expect changed behavior, or there is a breaking change

Choose release-note content by user impact, not by commit count. Prefer one clear bullet that synthesizes several implementation commits over several low-value bullets.

You must also keep the checked-in changelog in sync with the release:

- update the repository root `CHANGELOG.md`
- add or refresh a Keep a Changelog entry for the current release using the heading format `## [x.y.z] - YYYY-MM-DD`
- use only the user-facing substance from this release; do not copy internal commit subjects or GitHub-generated metadata
- group bullets under only the relevant sections from `Added`, `Changed`, `Fixed`, `Removed`, `Security`
- omit empty sections
- if an entry for this version already exists, replace that entry instead of creating a duplicate
- insert the current release entry above the next older version entry

Use these Keep a Changelog sections with discipline:

- `Added` for new user-visible capabilities, new workflows, new supported inputs, new outputs, or new automation users can rely on
- `Changed` for meaningful behavior changes, revised defaults, performance changes users will notice, or notable UX/reporting improvements
- `Deprecated` only when something still works in this release but users should migrate away before a future removal
- `Removed` for functionality, configuration, or behavior that no longer exists for users
- `Fixed` for bug fixes that correct wrong behavior users could hit in normal usage
- `Security` only for user-relevant security fixes, hardening, or exposure changes users should know about

Do not create a section unless it has at least one strong user-facing entry.

The notes must be semver-aware:

- infer the intended semantic meaning from the current tag and the user-visible diff, not from commit prefixes
- treat tags like `v1.2.3` as semantic version `1.2.3`
- major release: emphasize breaking changes, removals, migration steps, and compatibility boundaries
- minor release: emphasize new capabilities, deprecations, and meaningful backward-compatible improvements
- patch release: emphasize bug fixes, polish, reliability, and narrowly scoped behavior corrections
- for `0.y.z` releases, note that the project is still pre-1.0; use the tag as a signal, but still call out any breakage plainly rather than hiding behind the leading zero
- for a major release, call out breaking changes only when the diff shows real user-facing breakage
- for a minor release, emphasize new capabilities and meaningful improvements
- for a patch release, emphasize fixes, polish, and small behavior improvements

Treat the public surface of this project as the workflow behavior users depend on: workflow names, supported triggers and inputs, required secrets or configuration, emitted issues/reports/artifacts, and operational behavior that changes how the workflows run in a repository.

The notes must be user-facing:

- include workflow names only when they help explain user value
- translate implementation changes into user impact instead of narrating commits or files
- if this is the first release, say so briefly
- if there are no user-visible changes, say the release is primarily maintenance and keep the body very short
- explain breaking or surprising changes in terms of effect first, implementation second
- prefer concrete effects such as "audit issues now distinguish empty windows from zero completed runs" over internal descriptions such as "refactored empty-window handling"
- mention configuration changes only when a user may need to add, remove, rename, or reconsider settings
- mention dependency, CI, or infrastructure changes only when they alter user-observable behavior, compatibility, reliability, or setup requirements

Selection rubric for whether a change belongs in the release notes or changelog:

- include it when a user would notice it by using the workflow, reading its issue/report output, configuring it, upgrading it, or debugging it
- include it when it changes expectations around compatibility, migration, deprecation, stability, security, or operating cost
- omit it when it only affects how the repository is built, tested, compiled, organized, or maintained internally
- omit it when it is merely a prerequisite for other work and has no standalone user-facing effect
- collapse related low-level changes into one user-facing outcome when they support the same improvement

Exclude internal-only changes unless they materially affect users:

- CI, build, test, refactor, repo maintenance, code movement, formatting, generated lockfile updates, dependency bumps, telemetry, observability, and other internal plumbing
- pull request numbers, commit SHAs, and contributor attribution
- GitHub usernames, handles, reviewers, or author mentions
- branch names, ticket IDs, implementation plans, or statements about code generation/compilation unless users need that information to operate the release

Release-note anti-patterns to avoid:

- rewriting commit subjects into bullets
- listing files changed as though they were user value
- overstating small maintenance changes as major improvements
- using vague filler such as "various fixes" or "multiple improvements" when the concrete effect can be named
- mixing user actions with internal housekeeping in the same bullet
- mentioning contributor credit inside the body of user-facing notes

Writing quality constraints:

- use plain English and strong verbs
- keep bullets parallel and scannable
- start with the outcome, then add the condition or scope if needed
- avoid hype, marketing language, and unverifiable claims
- avoid repeating the same fact in both `## Highlights` and `## Upgrade Notes`
- keep the release body short enough to scan quickly; prefer fewer, sharper bullets over exhaustive coverage

You may use `bash` to inspect targeted diffs or file content if the provided context is not enough, but keep the final notes concise.

After you finalize the release body, use `bash` to update `CHANGELOG.md` and push the change back to the repository when `semver_context.json` contains a branch-like `release_target` such as `main`.

- configure git with `github-actions[bot]`
- do not force-push
- commit message: `docs: update changelog for ${RELEASE_TAG}`
- push to `origin HEAD:${RELEASE_TARGET}`
- if `RELEASE_TARGET` is clearly a raw commit SHA instead of a branch name, skip the repository changelog update and still update the GitHub release body

## Output Requirements

Call the `update_release` MCP tool with:

- `tag`: `${RELEASE_TAG}`
- `operation`: `replace`
- `body`: the complete markdown release notes body

Do not include placeholders or analysis in the final body.
