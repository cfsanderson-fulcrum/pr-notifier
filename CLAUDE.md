# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-file macOS bash script (`pr-notifier`) that polls GitHub every 2 minutes via launchd and sends native desktop notifications about PR activity. No build system, no dependencies beyond `gh`, `jq`, and optionally `terminal-notifier`.

## Running & Testing

```sh
# Run one poll cycle manually
./pr-notifier

# One-shot check for all currently watched PRs (no state update)
./pr-notifier --check

# Tail live log
tail -f ~/.local/share/pr-notifier/pr-notifier.log

# Inspect state
jq . ~/.local/share/pr-notifier/state.json

# Reset state (re-notifies everything on next run)
rm ~/.local/share/pr-notifier/state.json

# Reload launchd agent after plist edits
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.user.pr-notifier.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.pr-notifier.plist

# Check agent status
launchctl print gui/$(id -u)/com.user.pr-notifier
```

There is no test suite. Manual testing means running `./pr-notifier` and checking the log.

## Architecture

The script runs in five sequential phases per poll cycle:

1. **PR discovery** — single `gh api -X GET search/issues` call finds all open PRs you authored across the entire org
2. **Per-PR checks** — for each PR: CI status (via `statusCheckRollup`), reviews, issue comments
3. **Ready-to-merge detection** — `!draft && ci=passed && reviewDecision=APPROVED`
4. **Watched label PRs** — one search query per label per repo, OR-unioned with `jq unique_by`
5. **Watched review-requested PRs** — one search query per repo for `review-requested:{user}`

State is persisted as a flat JSON key-value map at `~/.local/share/pr-notifier/state.json`. Keys use `{owner/repo}#{number}_{field}` for authored PR state, and `watched_labeled_prs` / `watched_review_prs` (JSON arrays) for watched PR sets.

## Key Gotchas

- **`gh api` requires `-X GET`** — without it, `-f` params are sent as POST body and the search endpoint returns 404.
- **Label OR semantics** — GitHub's search treats `label:X label:Y` as AND. The script runs one query per label and unions results with `jq unique_by`.
- **launchd env vars** — shell profile (`~/.zshrc`) vars are not available to launchd agents. Use the `EnvironmentVariables` dict in the plist, or a `~/.local/share/pr-notifier/config` file (sourced with `set -a`).
- **terminal-notifier `-group`** — the group ID is per-PR (`pr-{repo}-{number}`) so notifications replace previous ones for the same PR rather than stacking.
- **`${VAR+x}` vs `${VAR:-}`** — the script uses `+x` to distinguish "not set" from "set to empty string" for `WATCHED_LABELS`.

## Configuration

All config via environment variables. Required: `GITHUB_USER`, `GITHUB_ORG`. Optional: `PR_NOTIFIER_WATCHED_LABELS` (comma-separated), `PR_NOTIFIER_WATCHED_REPOS` (comma-separated `org/repo`), `PR_NOTIFIER_WATCHED_ORG`, `PR_NOTIFIER_STATE_DIR`, `PR_NOTIFIER_MAX_LOG_LINES`.

## Conventions

- Section headers use `# ─── Section Name ───` with box-drawing chars
- Notification emojis: 🏷️ labeled, 👀 review-requested, ✅ closed, 🎉 merged, ❌ CI failed, 🔃 changes requested, 💬 comment, 🚀 ready to merge
- Log lines: `[timestamp] message`; notifications additionally prefixed `NOTIFY:`

## Remotes

- `origin` — `corporealshift/pr-notifier` (upstream)
- `fork` — `cfsanderson-fulcrum/pr-notifier` (Caleb's fork)
