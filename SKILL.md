---
name: claude-to-im
description: |
  This skill bridges Claude Code to IM platforms (Telegram, Discord, Feishu/Lark).
  It should be used when the user wants to start a background daemon that forwards
  IM messages to Claude Code sessions, or manage that daemon's lifecycle.
  Trigger on: "claude-to-im", "start bridge", "stop bridge", "bridge status",
  "查看日志", "启动桥接", "停止桥接", or any mention of IM bridge management.
  Subcommands: setup, start, stop, status, logs, reconfigure, doctor.
argument-hint: "setup | start | stop | status | logs [N] | reconfigure | doctor"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
  - Grep
  - Glob
---

# Claude-to-IM Bridge Skill

You are managing the Claude-to-IM bridge.
User data is stored at `~/.claude-to-im/`.

First, locate the skill directory by finding this SKILL.md file:
- Use Glob with pattern `**/skills/**/claude-to-im/SKILL.md` to find its path, then derive the skill root directory from it.
- Store that path mentally as SKILL_DIR for all subsequent file references.

## Command parsing

Parse the user's intent from `$ARGUMENTS` into one of these subcommands:

| User says (examples) | Subcommand |
|---|---|
| `setup`, `configure`, `配置` | setup |
| `start`, `start bridge`, `启动`, `启动桥接` | start |
| `stop`, `stop bridge`, `停止`, `停止桥接` | stop |
| `status`, `bridge status`, `状态` | status |
| `logs`, `logs 200`, `查看日志`, `查看日志 200` | logs |
| `reconfigure`, `修改配置` | reconfigure |
| `doctor`, `diagnose`, `诊断` | doctor |

Extract optional numeric argument for `logs` (default 50).

**IMPORTANT:** Before asking users for any platform credentials, first read `SKILL_DIR/references/setup-guides.md` to get the detailed step-by-step guidance for that platform. Present the relevant guide text to the user via AskUserQuestion so they know exactly what to do.

## Subcommands

### `setup`

Run an interactive setup wizard. Collect input **one field at a time** using AskUserQuestion. After each answer, confirm the value back to the user (masking secrets to last 4 chars only) before moving to the next question.

**Step 1 — Choose channels**

Ask which channels to enable (telegram, discord, feishu). Accept comma-separated input. Briefly describe each:
- **telegram** — Best for personal use. Streaming preview, inline permission buttons.
- **discord** — Good for team use. Server/channel/user-level access control.
- **feishu** (Lark) — For Feishu/Lark teams. Event-based messaging.

**Step 2 — Collect tokens per channel**

For each enabled channel, read `SKILL_DIR/references/setup-guides.md` and present the relevant platform guide to the user. Collect one credential at a time:

- **Telegram**: Bot Token → confirm (masked) → Allowed User IDs (optional)
- **Discord**: Bot Token → confirm (masked) → Allowed User IDs (optional) → Allowed Channel IDs (optional) → Allowed Guild IDs (optional)
- **Feishu**: App ID → confirm → App Secret → confirm (masked) → Domain (optional) → Allowed User IDs (optional). Guide through all 4 steps (A: batch permissions, B: enable bot, C: events & callbacks with long connection, D: publish version).

**Step 3 — General settings**

Ask for default working directory, model, and mode:
- **Working Directory**: default `$CWD`
- **Model**: `claude-sonnet-4-20250514` (default), `claude-opus-4-6`, `claude-haiku-4-5-20251001`
- **Mode**: `code` (default), `plan`, `ask`

**Step 4 — Write config and validate**

1. Show a final summary table with all settings (secrets masked to last 4 chars)
2. Ask user to confirm before writing
3. Use Bash to create directory structure: `mkdir -p ~/.claude-to-im/{data,logs,runtime,data/messages}`
4. Use Write to create `~/.claude-to-im/config.env` with all settings in KEY=VALUE format
5. Use Bash to set permissions: `chmod 600 ~/.claude-to-im/config.env`
6. Validate tokens:
   - Telegram: `curl -s "https://api.telegram.org/bot${TOKEN}/getMe"` — check for `"ok":true`
   - Feishu: `curl -s -X POST "${DOMAIN}/open-apis/auth/v3/tenant_access_token/internal" -H "Content-Type: application/json" -d '{"app_id":"...","app_secret":"..."}'` — check for `"code":0`
   - Discord: verify token matches format `[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+`
7. Report results with a summary table. If any validation fails, explain what might be wrong and how to fix it.

### `start`

Run: `bash "SKILL_DIR/scripts/daemon.sh" start`

Show the output to the user. If it fails, tell the user:
- Run `doctor` to diagnose: `/claude-to-im doctor`
- Check recent logs: `/claude-to-im logs`

### `stop`

Run: `bash "SKILL_DIR/scripts/daemon.sh" stop`

### `status`

Run: `bash "SKILL_DIR/scripts/daemon.sh" status`

### `logs`

Extract optional line count N from arguments (default 50).
Run: `bash "SKILL_DIR/scripts/daemon.sh" logs N`

### `reconfigure`

1. Read current config from `~/.claude-to-im/config.env`
2. Show current settings in a clear table format, with all secrets masked (only last 4 chars visible)
3. Use AskUserQuestion to ask what the user wants to change
4. When collecting new values, read `SKILL_DIR/references/setup-guides.md` and present the relevant guide for that field
5. Update the config file atomically (write to tmp, rename)
6. Re-validate any changed tokens
7. Remind user: "Run `/claude-to-im stop` then `/claude-to-im start` to apply the changes."

### `doctor`

Run: `bash "SKILL_DIR/scripts/doctor.sh"`

Show results and suggest fixes for any failures. Common fixes:
- SDK cli.js missing → `cd SKILL_DIR && npm install`
- dist/daemon.mjs stale → `cd SKILL_DIR && npm run build`
- Config missing → run `setup`

## Notes

- Always mask secrets in output (show only last 4 characters)
- If config.env doesn't exist and user runs start/status/logs, suggest running setup first
- The daemon runs as a background Node.js process managed by PID file
- Config persists at `~/.claude-to-im/config.env` — survives across sessions
