# OpenClaw 使用笔记

使用 [OpenClaw](https://github.com/anthropics/openclaw)（AGI 编排框架）过程中的个人经验和踩坑记录。

[English Version](./README.md)

---

## 目录

- [Claude Code Dispatch + Telegram 通知配置](#claude-code-dispatch--telegram-通知配置)

---

## Claude Code Dispatch + Telegram 通知配置

> 将 [claude-code-dispatch](https://github.com/win4r/claude-code-dispatch) 与 OpenClaw 的 Telegram 频道集成，实现任务完成后自动发送通知。

### 概述

Claude Code Dispatch 是一个 Claude Code 的异步任务调度系统，支持"提交即走"的工作模式。配合 OpenClaw 的 Telegram 集成，任务完成后会自动向 Telegram 群组发送详细的通知消息。

### 前置依赖

| 依赖 | 用途 |
|------|------|
| `claude` CLI | Claude Code 运行时 |
| `openclaw` CLI | 消息发送和网关 |
| `python3` | PTY 包装器脚本 |
| `jq`、`curl` | JSON 处理和 HTTP 请求 |
| `tmux` | 可选，交互模式使用 |

### 第一步：安装 Claude Code Dispatch

```bash
git clone https://github.com/win4r/claude-code-dispatch.git
cd claude-code-dispatch
chmod +x scripts/*.sh scripts/*.py
mkdir -p data/claude-code-results
```

### 第二步：设置 CLAUDE_CODE_BIN

PTY 包装器（`claude_code_run.py`）使用 `script(1)` 分配伪终端。`script` 派生的子进程可能无法继承完整的 `$PATH`，导致找不到 `claude` 二进制文件。

**解决方法：** 在 `~/.zshrc` 中设置绝对路径：

```bash
export CLAUDE_CODE_BIN="$HOME/.nvm/versions/node/v22.22.0/bin/claude"
```

> **踩坑记录：** 如果看到 `claude binary not found: claude`，一定是这个原因。

### 第三步：配置 Claude Code Hooks

编辑 `~/.claude/settings.json`：

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

> **关键坑点 — RESULT_DIR 路径不一致：**
>
> `dispatch.sh` 和 `notify-hook.sh` 都默认使用 `$(pwd)/data/claude-code-results` 作为 `RESULT_DIR`。但是：
> - `dispatch.sh` 从**项目目录**运行（如 `~/github/claude-code-dispatch/`）
> - `notify-hook.sh` 从 Claude Code 的**工作目录**运行（如 `/tmp/my-project/`）
>
> 两者解析到不同的路径，导致 hook 找不到 `task-meta.json`，静默跳过通知发送。
>
> **解决方法：** 必须在 hook 命令中设置绝对路径的 `RESULT_DIR`，如上所示。

### 第四步：OpenClaw Telegram 配置

需要在 `~/.openclaw/openclaw.json` 中修改三个配置：

#### 4a. 启用 sendMessage

OpenClaw 默认禁用 Telegram 消息发送功能，**必须**手动开启：

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

> **踩坑记录：** 不开启此项，`openclaw message send` 会静默失败，报错 `Telegram sendMessage is disabled`。hook 日志中只会显示 `Telegram send failed`，没有更多细节（因为 stderr 被 `2>/dev/null` 抑制了）。

#### 4b. 将群组添加到白名单

如果 `groupPolicy` 设为 `"allowlist"`，目标群组必须显式列出：

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

**如何获取群组 ID：**

1. 将 Bot 添加到 Telegram 群组
2. 在群里发送任意消息
3. 通过 Bot API 查询：

```bash
curl -s "https://api.telegram.org/bot<你的BOT_TOKEN>/getUpdates" | jq '.result[] | {chat_id: .message.chat.id, chat_title: .message.chat.title}'
```

> 群组 ID 是负数（如 `-5175618539`）。

#### 4c. 确认 Bot Token 已配置

```json
{
  "channels": {
    "telegram": {
      "name": "your-bot-name",
      "enabled": true,
      "botToken": "<你的bot-token>"
    }
  }
}
```

### 第五步：自动回调配置

在项目根目录放置 `dispatch-callback.json`，实现零参数回调检测：

```json
{
  "type": "group",
  "group": "-5175618539"
}
```

这样就不需要每次都传 `-g` 参数了。

### 第六步：测试完整流程

```bash
# 如果在 Claude Code 内部运行，需要取消嵌套检测
unset CLAUDECODE

./scripts/dispatch.sh \
  -p "Create hello.py that prints Hello World" \
  -n "test-task" \
  -w /tmp/test-project \
  -g "-5175618539" \
  --permission-mode bypassPermissions
```

**验证方式：**
1. 检查 Telegram 群组是否收到通知消息
2. 查看 `data/claude-code-results/hook.log` 中是否有 `Sent Telegram notification`
3. 查看 `data/claude-code-results/latest.json` 中的任务结果

### 故障排查速查表

| 现象 | 原因 | 解决方法 |
|------|------|----------|
| `claude binary not found` | PTY 子进程未继承 PATH | 设置 `CLAUDE_CODE_BIN` 绝对路径 |
| `Cannot be launched inside another Claude Code session` | 嵌套会话检测 | 运行前 `unset CLAUDECODE` |
| Hook 触发但 `telegram_group` 为空 | RESULT_DIR 路径不一致 | Hook 命令中设置绝对路径 `RESULT_DIR` |
| hook.log 中显示 `Telegram send failed` | 配置中 `sendMessage: false` | 设为 `actions.sendMessage: true` |
| `Telegram sendMessage is disabled` | 同上 | 同上 |
| 任务完成后 hook.log 没有新记录 | Hook 未配置 | 检查 `~/.claude/settings.json` 中的 hooks |
| `Duplicate hook within Ns, skipping` | 正常的去重行为 | 不是错误，第一次 hook 已处理 |

### 架构参考

```
dispatch.sh                         notify-hook.sh
    │                                     │
    ├─ 写入 task-meta.json                ├─ 读取 task-meta.json
    ├─ 启动 claude_code_run.py            ├─ 读取 task-output.txt
    ├─ 捕获输出到 task-output.txt          ├─ 写入 latest.json
    │                                     ├─ openclaw message send（Telegram）
    └─ 更新 task-meta.json（完成）         └─ 写入 pending-wake.json（兜底）
```

**关键：** 两个脚本必须使用相同的 `RESULT_DIR` — 这是集成中的第一大坑。
