# Internals

How the pr-notifier script works under the hood.

## Architecture

```
launchd (every 120s)
  └─ pr-notifier (bash)
       ├─ gh api search/issues  ←── single call finds all open PRs in the org
       │
       └─ for each PR:
            ├─ gh pr view          ←── get head SHA, review decision, draft status
            ├─ gh pr checks        ←── get CI check run statuses
            ├─ gh api .../reviews  ←── get review submissions
            ├─ gh api .../comments ←── get issue comments
            │
            ├─ compare against state.json
            ├─ notify() on changes  ──→ osascript (macOS notification)
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
| `search/issues` | 1 | Find all open PRs |
| `gh pr view` | N | Get SHA, reviewDecision, isDraft |
| `gh pr checks` | N | Get CI check statuses |
| `repos/.../reviews` | N | Get review submissions |
| `repos/.../comments` | N | Get issue comments |

**Total: 4N + 1 API calls per cycle.** With 9 open PRs, that's 37 calls
every 2 minutes. GitHub's authenticated rate limit is 5,000/hour, so this
uses about 1,110/hour — well within limits.

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
