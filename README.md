# pr-notifier

A local macOS tool that monitors all your open pull requests across a GitHub
organization and delivers native desktop notifications.

## What it does

Polls GitHub every 2 minutes and sends a macOS notification when:

- **CI fails** on any of your PRs (reports the failing check names)
- **CI passes** (all checks green)
- **Someone requests changes** on your PR
- **Someone leaves a review comment** on your PR
- **Someone comments** on your PR (human-only, bots filtered out)
- **A PR becomes ready to merge** (CI green + approved + not draft)
- **A PR is labeled** with a watched label (e.g., `design`, `hold/design`)
- **You are added as a reviewer** on a PR in a watched repo
- **A watched PR is closed or merged** (from either the label or reviewer set)

It does **not** notify for:

- Bot/automation comments (`dependabot[bot]`, `github-actions[bot]`, etc.)
- Approvals
- Your own comments or reviews
- PRs that aren't yours (unless they match watched labels or request your review)
- Label removals or review-request dismissals (silently dropped)

## Prerequisites

- macOS (uses `osascript` for native notifications)
- [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth login`)
- [`jq`](https://jqlang.github.io/jq/) (`brew install jq`)

## Installation

1. Clone or copy this repo somewhere on your machine (e.g. `~/pr-notifier`).

2. Configure your GitHub identity. You can either export environment variables
   (recommended) or edit the top of the `pr-notifier` script directly:

   ```sh
   export GITHUB_USER="your-github-username"
   export GITHUB_ORG="your-org"
   ```

   To persist these, add the exports to your shell profile (`~/.zshrc`,
   `~/.bashrc`, etc.) or to the launchd plist (see below).

3. Install `terminal-notifier` for clickable notifications (recommended):

   ```sh
   brew install terminal-notifier
   ```

   Without it, notifications still work via macOS built-in alerts but
   clicking them won't open the PR in your browser.

4. Make it executable:

   ```sh
   chmod +x pr-notifier
   ```

5. Create the state directory:

   ```sh
   mkdir -p ~/.local/share/pr-notifier
   ```

6. Install the launchd agent to run it automatically every 2 minutes:

   ```sh
   # Either copy or symlink the plist into LaunchAgents
   ln -s "$(pwd)/com.user.pr-notifier.plist" ~/Library/LaunchAgents/
   ```

   Edit `com.user.pr-notifier.plist` and replace all placeholder values:
   - `/path/to/pr-notifier/pr-notifier` → absolute path to the script
   - `YOUR_USERNAME` → your macOS username
   - `your-github-username` → your GitHub handle
   - `your-org` → the GitHub org to monitor

   To enable watched PRs, add these keys inside the `EnvironmentVariables`
   dict:

   ```xml
   <key>PR_NOTIFIER_WATCHED_LABELS</key>
   <string>design,hold/design</string>
   <key>PR_NOTIFIER_WATCHED_REPO</key>
   <string>your-org/your-repo</string>
   ```

   ```sh
   # Load (start running every 2 minutes)
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.pr-notifier.plist

   # Unload (stop)
   launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.user.pr-notifier.plist
   ```

## Usage

```sh
# Run manually (one poll cycle — only notifies on changes since last run)
./pr-notifier

# One-shot check: notify about ALL currently matching watched PRs
./pr-notifier --check

# View the log
cat ~/.local/share/pr-notifier/pr-notifier.log

# Tail the log live
tail -f ~/.local/share/pr-notifier/pr-notifier.log

# View tracked state
jq . ~/.local/share/pr-notifier/state.json

# Reset state (will re-notify on next run)
rm ~/.local/share/pr-notifier/state.json

# Check if the launchd agent is running
launchctl print gui/$(id -u)/com.user.pr-notifier

# View launchd stdout/stderr
cat ~/.local/share/pr-notifier/launchd-stdout.log
cat ~/.local/share/pr-notifier/launchd-stderr.log
```

### `--check` mode

Running `./pr-notifier --check` finds all PRs that currently match your
watched settings (labels and review-requested) and sends a notification for
each one. Unlike the normal poll mode, it does **not** compare against
previous state — you get notified about everything it finds. It also does
**not** update state, so the next normal poll cycle is unaffected.

## Configuration

All settings can be overridden via environment variables.

| Env variable | Script default | Purpose |
|---|---|---|
| `GITHUB_USER` | _(none — required)_ | Your GitHub username |
| `GITHUB_ORG` | _(none — required)_ | The GitHub org to monitor for your authored PRs |
| `PR_NOTIFIER_WATCHED_LABELS` | _(none — disabled)_ | Comma-separated labels to watch (OR logic). Must be set to enable label watching |
| `PR_NOTIFIER_WATCHED_REPO` | _(none)_ | Specific repo to watch for labels/reviews. Must be set (or set `PR_NOTIFIER_WATCHED_ORG`) to enable watched PRs |
| `PR_NOTIFIER_WATCHED_ORG` | _(empty)_ | Org to watch for labels/reviews (used if watched repo is empty) |
| `PR_NOTIFIER_STATE_DIR` | `~/.local/share/pr-notifier` | Directory for state and log files |
| `PR_NOTIFIER_MAX_LOG_LINES` | `500` | Log file rotation threshold |

`IGNORED_USERS` and the poll interval (launchd `StartInterval`) are configured
in the script and plist respectively.

## File locations

| File | Purpose |
|---|---|
| `pr-notifier` | The polling script |
| `com.user.pr-notifier.plist` | Example launchd plist (copy to `~/Library/LaunchAgents/`) |
| `~/.local/share/pr-notifier/state.json` | Tracked state between runs |
| `~/.local/share/pr-notifier/pr-notifier.log` | Application log |
| `~/.local/share/pr-notifier/launchd-stdout.log` | launchd stdout |
| `~/.local/share/pr-notifier/launchd-stderr.log` | launchd stderr |
