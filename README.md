# OpenClaw Usage Notes

Personal notes and lessons learned from using [OpenClaw](https://github.com/anthropics/openclaw) — an AGI orchestration framework.

[中文版](./README_CN.md)

---

## Table of Contents

- [Claude Code Dispatch + Telegram Notification Setup](#claude-code-dispatch--telegram-notification-setup)

---

## Claude Code Dispatch + Telegram Notification Setup

> Integrating [claude-code-dispatch](https://github.com/win4r/claude-code-dispatch) with OpenClaw's Telegram channel for automatic task completion notifications.

### Overview

Claude Code Dispatch is a fire-and-forget task dispatching system for Claude Code. When paired with OpenClaw's Telegram integration, completed tasks automatically send rich notifications to your Telegram group.

### Prerequisites

| Dependency | Purpose |
|------------|---------|
| `claude` CLI | Claude Code runtime |
| `openclaw` CLI | Message sending & gateway |
| `python3` | PTY wrapper script |
| `jq`, `curl` | JSON processing & HTTP |
| `tmux` | Optional, for interactive mode |

### Step 1: Install Claude Code Dispatch

```bash
git clone https://github.com/win4r/claude-code-dispatch.git
cd claude-code-dispatch
chmod +x scripts/*.sh scripts/*.py
mkdir -p data/claude-code-results
```

### Step 2: Set CLAUDE_CODE_BIN

The PTY wrapper (`claude_code_run.py`) uses `script(1)` to allocate a pseudo-terminal. Child processes spawned by `script` may not inherit your full `$PATH`, so the `claude` binary can't be found.

**Fix:** Set `CLAUDE_CODE_BIN` to the absolute path in `~/.zshrc`:

```bash
export CLAUDE_CODE_BIN="$HOME/.nvm/versions/node/v22.22.0/bin/claude"
```

> **Lesson learned:** If you see `claude binary not found: claude`, this is always the cause.

### Step 3: Configure Claude Code Hooks

Edit `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "RESULT_DIR=$HOME/github/claude-code-dispatch/data/claude-code-results ~/github/claude-code-dispatch/scripts/notify-hook.sh",
            "timeout": 15
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "RESULT_DIR=$HOME/github/claude-code-dispatch/data/claude-code-results ~/github/claude-code-dispatch/scripts/notify-hook.sh",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

> **Critical pitfall — RESULT_DIR mismatch:**
>
> Both `dispatch.sh` and `notify-hook.sh` default `RESULT_DIR` to `$(pwd)/data/claude-code-results`. However:
> - `dispatch.sh` runs from the **project directory** (e.g. `~/github/claude-code-dispatch/`)
> - `notify-hook.sh` runs from Claude Code's **working directory** (e.g. `/tmp/my-project/`)
>
> They resolve to different paths, so the hook can't find `task-meta.json` and silently skips notifications.
>
> **Fix:** Always set an absolute `RESULT_DIR` in the hook command, as shown above.

### Step 4: OpenClaw Telegram Configuration

Three settings are needed in `~/.openclaw/openclaw.json`:

#### 4a. Enable sendMessage

By default, OpenClaw disables outbound Telegram messages. You **must** enable it:

```json
{
  "channels": {
    "telegram": {
      "actions": {
        "sendMessage": true
      }
    }
  }
}
```

> **Lesson learned:** Without this, `openclaw message send` fails silently with `Telegram sendMessage is disabled`. The hook logs it as `Telegram send failed` with no further detail (because stderr is suppressed with `2>/dev/null`).

#### 4b. Add Group to Allowlist

With `groupPolicy: "allowlist"`, your target group must be explicitly listed:

```json
{
  "channels": {
    "telegram": {
      "groupPolicy": "allowlist",
      "groups": {
        "-5175618539": {
          "requireMention": false,
          "enabled": true
        }
      }
    }
  }
}
```

**How to find your group ID:**

1. Add the bot to your Telegram group
2. Send any message in the group
3. Query the Bot API:

```bash
curl -s "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates" | jq '.result[] | {chat_id: .message.chat.id, chat_title: .message.chat.title}'
```

> Group IDs are negative numbers (e.g. `-5175618539`).

#### 4c. Ensure Bot Token is Set

```json
{
  "channels": {
    "telegram": {
      "name": "your-bot-name",
      "enabled": true,
      "botToken": "<your-bot-token>"
    }
  }
}
```

### Step 5: Auto-Callback Configuration

Place `dispatch-callback.json` in your project root for zero-config callback detection:

```json
{
  "type": "group",
  "group": "-5175618539"
}
```

This way you don't need to pass `-g` every time.

### Step 6: Test the Full Pipeline

```bash
# If running from inside Claude Code, unset the nesting guard
unset CLAUDECODE

./scripts/dispatch.sh \
  -p "Create hello.py that prints Hello World" \
  -n "test-task" \
  -w /tmp/test-project \
  -g "-5175618539" \
  --permission-mode bypassPermissions
```

**Verify:**
1. Check the Telegram group for the notification message
2. Check `data/claude-code-results/hook.log` for `Sent Telegram notification`
3. Check `data/claude-code-results/latest.json` for task results

### Troubleshooting Checklist

| Symptom | Cause | Fix |
|---------|-------|-----|
| `claude binary not found` | PATH not inherited in PTY | Set `CLAUDE_CODE_BIN` absolute path |
| `Cannot be launched inside another Claude Code session` | Nested session detection | `unset CLAUDECODE` before running |
| Hook fires but `telegram_group` is empty | RESULT_DIR mismatch | Set absolute `RESULT_DIR` in hook command |
| `Telegram send failed` in hook.log | `sendMessage: false` in config | Set `actions.sendMessage: true` |
| `Telegram sendMessage is disabled` | Same as above | Same as above |
| No hook log entry after task completes | Hook not configured | Check `~/.claude/settings.json` hooks |
| `Duplicate hook within Ns, skipping` | Normal dedup behavior | Not an error, first hook already processed |

### Architecture Reference

```
dispatch.sh                         notify-hook.sh
    │                                     │
    ├─ Write task-meta.json               ├─ Read task-meta.json
    ├─ Launch claude_code_run.py          ├─ Read task-output.txt
    ├─ Capture output to task-output.txt  ├─ Write latest.json
    │                                     ├─ openclaw message send (Telegram)
    └─ Update task-meta.json (done)       └─ Write pending-wake.json (fallback)
```

**Key:** Both scripts must agree on `RESULT_DIR` — this is the #1 integration gotcha.
