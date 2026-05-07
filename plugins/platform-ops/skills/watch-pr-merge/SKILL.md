---
name: watch-pr-merge
description: Watch a GitHub PR in the background and raise a macOS desktop notification when it is merged. Use when the user asks to be notified, alerted, or watch for a PR merge, or provides a GitHub PR URL and mentions notifications.
---

# Watch PR Merge

Runs `watch_pr_merge_notify.sh` (bundled in this skill) as a background task. Sends a macOS notification (sound + alert) when the PR is merged or closed.

## Steps

1. Extract `REPO` (`owner/name`) and `PR` (number) from the GitHub URL the user provided.
2. Launch detached from the Cursor session using `nohup` + `disown`. This survives Cursor shell teardown:
   ```bash
   REPO=<owner/name> PR=<number> nohup bash /Users/pmartorell/.cursor/skills/watch-pr-merge/watch_pr_merge_notify.sh > /tmp/watch-pr-<number>.log 2>&1 & disown; echo "PID: $!"
   ```
3. Smoke-check that the process started (run this immediately after, no delay needed):
   ```bash
   sleep 2 && cat /tmp/watch-pr-<number>.log
   ```
   Expected first line: `HH:MM:SS Watching <REPO>#<PR> every 30s (max 24h). Ctrl+C to stop.`
   If the log is empty or shows an error, report it and do not tell the user the watcher is running.
4. Tell the user: watching `REPO#PR` every 30 s (max 24 h); they will get a macOS notification when it merges. Share the log path `/tmp/watch-pr-<number>.log` so they can tail it manually.

## Variables

| Var | Required | Default | Notes |
|-----|----------|---------|-------|
| `REPO` | yes | — | `owner/name` extracted from the GitHub URL |
| `PR` | yes | — | PR number extracted from the GitHub URL |
| `INTERVAL_SEC` | no | `30` | Pass if the user wants a different polling rate |
| `MAX_HOURS` | no | `24` | Pass if the user wants a longer or shorter timeout |

## Notes

- Requires `gh` (authenticated) and `jq` — both are available on this machine.
- macOS notification fires via `osascript`; sound **Glass** on merge, **Basso** on close-without-merge or timeout.
- No `chmod` needed — the script is already executable.
- **Do NOT use `block_until_ms: 0` alone** — Cursor kills those processes when the session goes idle. `nohup` + `disown` makes it a true system-level daemon.
