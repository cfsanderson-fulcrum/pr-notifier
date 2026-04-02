# Internals

How the pr-notifier script works under the hood.

## Architecture

```
launchd (every 120s)
  └─ pr-notifier (bash)
       ├─ gh api search/issues  ←── single call finds all open PRs in the org
       │
       ├─ for each PR:
       │    ├─ gh pr view          ←── get head SHA, review decision, draft status
       │    ├─ gh pr checks        ←── get CI check run statuses
       │    ├─ gh api .../reviews  ←── get review submissions
       │    ├─ gh api .../comments ←── get issue comments
       │    │
       │    ├─ compare against state.json
       │    ├─ notify() on changes  ──→ osascript (macOS notification)
       │    └─ update state.json
       │
       └─ watched PRs (label + review-requested):
            ├─ gh api search/issues × N  ←── one call per watched label (OR)
            ├─ gh api search/issues      ←── review-requested:{user}
            │
            ├─ union label results, compare with previous state
            ├─ notify on new additions (labeled or review-requested)
            ├─ for disappeared PRs:
            │    ├─ gh pr view --json state  ←── check if closed/merged
            │    └─ notify if closed/merged, else silently drop
            └─ update state.json
```

## PR discovery

Instead of iterating every repo in the org (100+), the script makes a single
GitHub Search API call:

```
GET /search/issues?q=is:pr+is:open+author:{user}+org:{org}
```

This returns all open PRs authored by you across every repo in one request.
The `gh api` call uses `-X GET` explicitly — without it, `gh` sends `-f`
parameters as a POST body, which the search endpoint rejects with a 404.

The search API has a rate limit of 30 requests/minute, but we only make 1 per
poll cycle for discovery.

## Watched PRs (label + review-requested)

After processing authored PRs, the script runs a second discovery phase for
PRs matching watched labels or requesting your review.

### Label OR semantics

GitHub's search API treats multiple `label:` qualifiers as AND (all must
match). To achieve OR (any label matches), the script runs one search per
label and unions the results by PR number using `jq unique_by(.number)`.

### Scoping

`WATCHED_REPOS` (comma-separated) scopes queries to specific repos (e.g.,
`fulcrumapp/fulcrum,fulcrumapp/fulcrum-ios`). Each repo gets its own set of
search queries, and results are unioned with deduplication by `repo#number`.
The legacy `WATCHED_REPO` (singular) is still supported as a single-repo
fallback. If neither is set but `WATCHED_ORG` is, it uses `org:{org}` for
org-wide monitoring. All variables default to empty — watched-PR features are
disabled until explicitly configured.

### New/disappeared detection

The script stores the full set of matched PRs (as JSON arrays) in state.json
under `watched_labeled_prs` and `watched_review_prs`. Each poll cycle:

1. Fetches the current sets
2. Compares PR numbers against the previous sets
3. **New PRs** → notification (🏷️ for labels, 👀 for review-requested)
4. **Disappeared PRs** → fetches the PR state via `gh pr view --json state`:
   - `CLOSED` or `MERGED` → notification (✅ or 🎉)
   - Still `OPEN` → label was removed or review dismissed; silently dropped

### API calls

| Call | Count | Purpose |
|---|---|---|
| `search/issues` (labels) | R × L (per repo, per label) | Find labeled PRs |
| `search/issues` (reviews) | R (one per repo) | Find review-requested PRs |
| `gh pr view` (closures) | D (one per disappeared PR) | Check if closed/merged |

With 3 watched repos, 2 labels, and no disappearances, that's 9 extra calls
per cycle.

## State tracking

All state is stored in `~/.local/share/pr-notifier/state.json` as a flat
key-value map. Keys are namespaced by `{owner/repo}#{pr_number}_{field}`.

### Tracked fields per PR

| Key suffix | Value | Purpose |
|---|---|---|
| `_sha` | Git SHA | Detect new pushes (reset CI tracking) |
| `_ci_state` | `none` / `pending` / `failed` / `passed` | Detect CI transitions |
| `_last_review_id` | Numeric ID | Only process reviews newer than this |
| `_last_comment_id` | Numeric ID | Only process comments newer than this |
| `_ready_to_merge` | `true` / `false` | Avoid repeat "ready" notifications |

### Tracked fields for watched PRs

| Key | Value | Purpose |
|---|---|---|
| `watched_labeled_prs` | JSON array of `{number, title, url, repo}` | Track label-matched PR set |
| `watched_review_prs` | JSON array of `{number, title, url, repo}` | Track review-requested PR set |

### State transitions

**CI tracking** uses a two-axis model: SHA + CI state.

- When the SHA changes (new push), CI state is reset. If the new SHA already
  has failures, a notification fires immediately.
- When the SHA is the same, we watch for transitions:
  - `pending → failed`: notify with the names of the first 3 failing checks.
  - `pending → passed`: notify that CI is green.
  - `failed → passed`: notify (re-run fixed it).
  - `passed → failed`: should not happen on the same SHA, but would notify.

**Review/comment tracking** uses monotonically increasing IDs. We store the
highest ID seen and only process items with a higher ID on the next poll. This
avoids duplicate notifications even if the API returns items we've already seen.

## Notification filtering

### Bot detection

GitHub API returns a `user.type` field (`"User"` or `"Bot"`). All `Bot` type
accounts are filtered by the jq query before the script even sees them.

### Service account detection

Some automation runs under regular `User` accounts. These
are caught by the `IGNORED_USERS` array, which is checked via the
`is_ignored_user()` function. This runs after the jq filter, so it catches
accounts that GitHub classifies as `User` but that are actually automation.

### Self-filtering

Your own comments and reviews are excluded:

- Comments: filtered in the jq query (`select(.user.login != "...")`)
- Reviews: filtered in the bash loop (skips `"$GITHUB_USER"`)

### Approval suppression

Reviews with state `APPROVED` are skipped in the bash loop, per configuration.

## API calls per poll cycle

For N open PRs, each cycle makes:

| Call | Count | Purpose |
|---|---|---|
| `search/issues` | 1 | Find all open authored PRs |
| `gh pr view` | N | Get SHA, reviewDecision, isDraft |
| `gh pr checks` | N | Get CI check statuses |
| `repos/.../reviews` | N | Get review submissions |
| `repos/.../comments` | N | Get issue comments |
| `search/issues` (labels) | R×L | Find watched-label PRs (per repo, per label) |
| `search/issues` (reviews) | R | Find review-requested PRs (per repo) |
| `gh pr view` (closures) | D | Check disappeared PRs (typically 0) |

**Total: 4N + R×L + R + D + 1 API calls per cycle.** With 9 open PRs,
3 watched repos, 2 labels, and no disappearances, that's 46 calls every
2 minutes. GitHub's authenticated rate limit is 5,000/hour, so this uses
about 1,380/hour — well within limits.

## Logging

The script writes structured log lines to `~/.local/share/pr-notifier/pr-notifier.log`.
Each line is prefixed with a timestamp. The log auto-rotates at 500 lines
(configurable via `MAX_LOG_LINES`) by keeping only the tail.

Lines prefixed with `NOTIFY:` indicate a macOS notification was sent. Other
lines are informational (PR processing, SHA resets, cycle start/end).

## launchd integration

The plist at configures:

- `StartInterval: 120` — run every 2 minutes
- `RunAtLoad: true` — run immediately on load (login or manual bootstrap)
- `PATH` includes `/opt/homebrew/bin` so `gh` and `jq` are found
- stdout/stderr are captured to separate log files for debugging launchd issues

launchd is preferred over cron on macOS because:

- It survives sleep/wake cycles (runs immediately after wake if overdue)
- Better integration with the macOS process lifecycle
- Has its own log capture built in
