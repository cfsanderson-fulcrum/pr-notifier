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

It does **not** notify for:

- Bot/automation comments (`dependabot[bot]`, `github-actions[bot]`, etc.)
- Approvals
- Your own comments or reviews
- PRs that aren't yours

## Prerequisites

- macOS (uses `osascript` for native notifications)
- [`gh`](https://cli.github.com/) CLI, authenticated (`gh auth login`)
- [`jq`](https://jqlang.github.io/jq/) (`brew install jq`)

## Installation

1. Clone or copy this repo somewhere on your machine (e.g. `~/pr-notifier`).

2. Configure the script — edit the top of `pr-notifier` and set `GITHUB_USER`
   and `GITHUB_ORG`:

   ```sh
   GITHUB_USER="your-github-username"
   GITHUB_ORG="your-org"
   ```

3. Make it executable:

   ```sh
   chmod +x pr-notifier
   ```

4. Create the state directory:

   ```sh
   mkdir -p ~/.local/share/pr-notifier
   ```

5. Install the launchd agent to run it automatically every 2 minutes:

   ```sh
   # Copy the example plist and fill in your paths
   cp com.user.pr-notifier.plist ~/Library/LaunchAgents/
   ```

   Edit `~/Library/LaunchAgents/com.user.pr-notifier.plist` and replace
   `YOUR_USERNAME` with your macOS username (or use absolute paths to
   wherever you placed the repo).

   ```sh
   # Load (start running every 2 minutes)
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.user.pr-notifier.plist

   # Unload (stop)
   launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.user.pr-notifier.plist
   ```

## Usage

```sh
# Run manually (one poll cycle)
./pr-notifier

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

## Configuration

Edit the top of `pr-notifier`:

| Variable | Purpose |
|---|---|
| `GITHUB_USER` | Your GitHub username |
| `GITHUB_ORG` | The GitHub org to monitor |
| `IGNORED_USERS` | Array of usernames to treat as bots |
| `MAX_LOG_LINES` | Log file rotation threshold (default: 500) |

The poll interval (default: 120s) is set in the launchd plist under
`StartInterval`.

## File locations

| File | Purpose |
|---|---|
| `pr-notifier` | The polling script |
| `com.user.pr-notifier.plist` | Example launchd plist (copy to `~/Library/LaunchAgents/`) |
| `~/.local/share/pr-notifier/state.json` | Tracked state between runs |
| `~/.local/share/pr-notifier/pr-notifier.log` | Application log |
| `~/.local/share/pr-notifier/launchd-stdout.log` | launchd stdout |
| `~/.local/share/pr-notifier/launchd-stderr.log` | launchd stderr |
