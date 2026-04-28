# remote-agents

[English](https://github.com/vivaxy/remote-agents/blob/main/README.md) | [简体中文](https://github.com/vivaxy/remote-agents/blob/main/README.zh-CN.md)

> Run your coding agent on your machine. Control it from anywhere via messaging.

## Quick Start

### Prerequisites

One supported agent runtime installed and authenticated:

- [Claude CLI](https://claude.ai/code) for the default `claude` runtime
- [Codex CLI](https://developers.openai.com/codex/sdk) for the `codex` runtime

### Install (macOS arm64 / x64)

```bash
curl -fsSL https://github.com/vivaxy/remote-agents/releases/latest/download/install.sh | sh
```

### Set up your Telegram bot

1. **Create a bot** via [@BotFather](https://t.me/BotFather): send `/newbot` and follow the prompts. Save the token (format: `123456:ABC-DEF...`).
2. **Get your user ID**: send any message to [@userinfobot](https://t.me/userinfobot). Copy the number (e.g. `987654321`).

### Configure and run

```bash
cd /path/to/your/project
mkdir -p .remote-agents
cat > .remote-agents/remote-agents.json <<'EOF'
{
  "telegramBotToken": "YOUR_BOT_TOKEN",
  "telegramAllowedUsers": [YOUR_TELEGRAM_USER_ID]
}
EOF

remote-agents
```

Send any message to your bot — output streams back to Telegram and is saved to `.remote-agents/YYYY-MM-DD-HH-mm-SS-mmm.md`.

### WeChat

Set `channelType: "wechat"` in `remote-agents.json`:

```json
{
  "channelType": "wechat"
}
```

On first run the daemon prints a QR code in the terminal. Scan it with WeChat to authorize.

## Using the bot

### Sending input

- Every message triggers execution immediately.
- Output streams back as messages.
- Multiple messages sent quickly are merged and processed together.

### Tool approval

When the agent asks to use a tool, four buttons appear:

| Button | Behavior |
|--------|----------|
| **✅ Allow** | Approve this call. The same tool is asked again next time. |
| **✅ Allow (Session)** | Approve and auto-approve this tool for the rest of the session (until `/new` or daemon exit). |
| **✅ Allow (Always)** | Approve and persist the tool to `.claude/settings.local.json` — survives `/new` and restarts. |
| **❌ Deny** | Deny this call; you will be prompted for a reason. |

Session-approved tools accumulate in `allowedTools` until you run `/new`.

## Commands

| Command | Action |
|---|---|
| `/permission_mode default\|acceptEdits\|bypassPermissions\|plan\|dontAsk` | Switch permission mode; no agent invocation |
| `/allowed_tools [Tool1 ...]` | Overwrite session `allowedTools` entirely (empty arg = clear; **no arg = query current**) |
| `/add_allowed_tools [Tool1 ...]` | Append tools to session `allowedTools` (no arg = no-op) |
| `/remove_allowed_tools [Tool1 ...]` | Remove tools from session `allowedTools` (no arg = no-op) |
| `/new` | Reset session (clears stats, session ID, tools, mode) and shows new state |
| `/quit` | Exit the daemon cleanly |
| `/state` | Show current daemon state |
| `/help` | Show usage instructions and all commands |
| `/start` | Show welcome message and current state (status, session ID, permission mode, allowed tools, accumulated cost) |
| `/verbose [true\|false]` | View or set verbose mode. No argument shows current value and a toggle button. `true`/`false` sets and persists to `remote-agents.json` |
| *(no command)* | Reuse last mode + persisted tools; run the agent |

## Custom Commands

Define project-specific slash commands in `<remote_agents_dir>/remote-agents.json`:

```json
{
  "commands": {
    "review": {
      "permissionMode": "plan",
      "allowedTools": ["Read", "Grep", "Glob"],
      "addAllowedTools": ["Read", "Grep", "Glob"],
      "removeAllowedTools": [],
      "inputPrefix": "Please do a thorough code review:",
      "description": "Review code for quality"
    },
    "deploy": {
      "permissionMode": "default",
      "allowedTools": ["Bash(git:*)", "Bash(npm:*)"]
    }
  }
}
```

**Command name rules:** Names must be lowercase `snake_case` — only `a-z`, `0-9`, and `_`; no leading/trailing underscores, no consecutive underscores. Invalid names are skipped with a warning at startup.

Use them like built-in commands: `/review` followed by `Focus on security issues`.

## Configuration

Settings cascade from two locations:

1. **Global**: `~/.remote-agents/remote-agents.json` — shared across all projects
2. **Project**: `<remote_agents_dir>/remote-agents.json` — project-specific overrides

Project settings override global settings.

### CLI flags

```bash
remote-agents [--working-dir <path>] [--remote-agents-dir <path>]
```

| Flag | Description | Default |
|------|-------------|---------|
| `--working-dir` | Project source directory | current directory |
| `--remote-agents-dir` | State, logs, context, and input directory | `<working_dir>/.remote-agents` |

### `remote-agents.json`

| Key | Description | Default |
|-----|-------------|---------|
| `runtimeProvider` | Agent runtime: `claude` or `codex` | `"claude"` |
| `model` | Model override (e.g. `claude-sonnet-4-6`) | — |
| `agentEnvScript` | Path to a TS script (absolute or relative to `<remote_agents_dir>`) that default-exports `async (runtimeProvider): Promise<Record<string, string>>`. Called before each invocation; returned vars are merged into `process.env`. Receives the active provider so one script can return different keys for `claude` vs `codex`. | — |
| `defaultAllowedTools` | Allowed tools applied on startup and after `/new` | — |
| `defaultPermissionMode` | Permission mode on first run or after `/new`: `default`\|`acceptEdits`\|`bypassPermissions`\|`plan`\|`dontAsk` | `default` |
| `telegramBotToken` | Telegram bot token | Required |
| `telegramAllowedUsers` | Permitted Telegram user IDs | Required |
| `telegramPollTimeout` | Long-poll timeout in seconds (1–60). Higher values reduce QPS while keeping responses real-time. | `30` |
| `channelNetworkScript` | Path to a TS file (absolute or relative to `<remote_agents_dir>`) that default-exports `async (channelType): Promise<{ proxy?: string; skipEtcHosts?: boolean } \| undefined>`. Returns per-channel network config: `proxy` routes the channel through an HTTP(S) proxy; `skipEtcHosts` resolves the API host via DNS directly, bypassing `/etc/hosts`. Return `undefined` or `{}` for default behavior. A malformed script aborts daemon startup with a `ParseError`. | — |
| `commands` | Custom slash commands (see [Custom Commands](#custom-commands)) | — |
| `verbose` | Enable verbose logging | `false` |

### Configuration merging

**Merge rules:**
- **Scalar fields** (string, number, boolean): project overrides global
- **Array fields**: project replaces global entirely (no concatenation)
- **Custom commands** (`commands` object): merged by name; project overrides global for duplicates

**Example.** Global config (`~/.remote-agents/remote-agents.json`):
```json
{
  "telegramBotToken": "global-token",
  "model": "sonnet",
  "defaultAllowedTools": ["Read", "Grep"],
  "commands": {
    "review": {
      "permissionMode": "plan",
      "description": "Global review command"
    }
  }
}
```

Project config (`<remote_agents_dir>/remote-agents.json`):
```json
{
  "model": "opus",
  "defaultAllowedTools": ["Bash"],
  "commands": {
    "review": {
      "permissionMode": "acceptEdits",
      "description": "Project-specific review"
    },
    "deploy": {
      "permissionMode": "default",
      "description": "Deploy to production"
    }
  }
}
```

Final merged config:
```json
{
  "telegramBotToken": "global-token",
  "model": "opus",
  "defaultAllowedTools": ["Bash"],
  "commands": {
    "review": {
      "permissionMode": "acceptEdits",
      "description": "Project-specific review"
    },
    "deploy": {
      "permissionMode": "default",
      "description": "Deploy to production"
    }
  }
}
```

Define Telegram credentials once globally and override model, tools, or commands per project.

## Directory Layout

```
<working_dir>/
└── .remote-agents/
    ├── YYYY-MM-DD-HH-mm-SS-mmm.md   # Agent output + archived inputs
    ├── state.json                    # Runtime state (JSON)
    ├── YYYY-MM-DD.log                # Daemon diagnostics (one file per day)
    └── remote-agents.json            # Configuration
```

## License

GPL-3.0
