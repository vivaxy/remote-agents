# remote-agents

[English](https://github.com/vivaxy/remote-agents/blob/main/README.md) | [简体中文](https://github.com/vivaxy/remote-agents/blob/main/README.zh-CN.md)

> Drive your coding agent CLI from Telegram. Persistent, recoverable, server-side.

## Why remote-agents

- Run Claude Code or Codex on your server, drive it from anywhere via Telegram.
- State persists across daemon restarts — sessions, allowed tools, permission mode, accumulated cost.
- One-line install on macOS, no Node.js needed on the host.
- Auditable — every input and output saved as a timestamped Markdown file.
- Custom slash commands per project.

## Quick Start

### Prerequisites

One supported agent runtime installed and authenticated:

- [Claude CLI](https://claude.ai/code) for the default `claude` runtime
- [Codex CLI](https://developers.openai.com/codex/sdk) for the `codex` runtime

A Telegram bot token and your Telegram user ID — see [Setting up your Telegram bot](#setting-up-your-telegram-bot) below.

### Install (macOS arm64 / x64)

```bash
curl -fsSL https://github.com/vivaxy/remote-agents/releases/latest/download/install.sh | sh
```

The install script auto-detects your CPU, downloads the matching tarball, strips the Gatekeeper quarantine attribute, and installs the binary to `/usr/local/bin` (falling back to `~/.local/bin` if that directory is not writable). It exits with a clear error on any non-darwin OS or unsupported CPU.

Optional environment variables:

| Variable | Effect | Default |
|----------|--------|---------|
| `VERSION` | Pin to a specific release tag, e.g. `VERSION=v0.1.0` | latest release |
| `INSTALL_DIR` | Override the install directory (must already exist and be writable) | `/usr/local/bin`, fallback `$HOME/.local/bin` (auto-created) |

### Setting up your Telegram bot

1. **Create a bot** via [@BotFather](https://t.me/BotFather): send `/newbot` and follow the prompts. Save the bot token (format: `123456:ABC-DEF...`).
2. **Get your user ID**: send any message to [@userinfobot](https://t.me/userinfobot). Copy your user ID (a number like `987654321`).

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

Send any message to your bot — every message triggers execution. Output streams back to Telegram and is saved to `.remote-agents/YYYY-MM-DD-HH-mm-SS-mmm.md`.

### WeChat

Set `channelType: "wechat"` in `remote-agents.json`:

```json
{
  "channelType": "wechat"
}
```

On first run, the daemon prints a WeChat QR code URL to stdout. Scan it with WeChat to authorize. Credentials are saved to `<remote-agents-dir>/wechat-credentials.json` and reused on subsequent runs.

Unlike Telegram, the WeChat channel has no allowed-users list — the QR-scan login binds the bot to your WeChat account, which is the only sender that can reach it.

## Directory Layout

```
<working_dir>/
└── .remote-agents/
    ├── YYYY-MM-DD-HH-mm-SS-mmm.md   # Coding agent CLI output + archived inputs
    ├── state.json            # Runtime state (JSON)
    ├── YYYY-MM-DD.log        # Daemon diagnostics (one file per day)
    └── remote-agents.json    # Configuration (required for Telegram credentials)
```

## Using the bot

### Sending input

- Every message you send triggers execution immediately — no `#submit` needed.
- Output is streamed back as Telegram messages.
- Multiple messages sent in quick succession are merged and processed together.
- Use `/permission_mode` to get an inline keyboard for mode selection.

### Tool approval

When the agent requests permission to use a tool, four buttons appear:

| Button | Behavior |
|--------|----------|
| **✅ Allow** | Approve this single tool call. The same tool is asked again next time. |
| **✅ Allow (Session)** | Approve now and auto-approve the same tool for the rest of this session (until `/new` or daemon exit). |
| **✅ Allow (Always)** | Approve now and remember persistently — writes the tool to your project's `.claude/settings.local.json` so it survives `/new` and restarts. |
| **❌ Deny** | Deny this call; you will be prompted to provide a reason. |

Session-approved tools accumulate in `allowedTools` for the lifetime of the current session. Use `/new` to reset all session permissions back to the configured defaults.

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
| `/verbose [true\|false]` | View or set verbose mode. With no argument, shows current value and a toggle button. With `true` or `false`, directly sets the value and persists to `<remote_agents_dir>/remote-agents.json` |
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

> **Command name rules:** Custom command names must be lowercase `snake_case` — only lowercase letters (`a-z`), digits (`0-9`), and underscores (`_`); no leading/trailing underscores, no consecutive underscores. Invalid names are skipped with a warning at startup.

Use them like built-in commands: `/review` followed by `Focus on security issues`.

## Configuration

The daemon supports **two-level configuration cascading** to allow global defaults and project-specific overrides:

1. **Global config**: `~/.remote-agents/remote-agents.json` (shared across all projects)
2. **Project config**: `<remote_agents_dir>/remote-agents.json` (project-specific)

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
| `runtimeProvider` | Agent runtime provider (`claude` or `codex`) | `"claude"` |
| `model` | Model override (e.g. `claude-sonnet-4-6`) | — |
| `agentEnvScript` | Path to a TS script (absolute or relative to `<remote_agents_dir>`) that default-exports an `async (runtimeProvider): Promise<Record<string, string>>`. Receives the active runtime provider (`'claude'` or `'codex'`) so a single script can return per-provider env vars (e.g. `ANTHROPIC_API_KEY` for `claude` vs `OPENAI_API_KEY` for `codex`). Called before each agent invocation; returned vars are merged with `process.env`. | — |
| `defaultAllowedTools` | Default allowed tools applied on startup and after `/new` | — |
| `defaultPermissionMode` | Default permission mode on first run or after `/new` (`default`\|`acceptEdits`\|`bypassPermissions`\|`plan`\|`dontAsk`) | `default` |
| `telegramBotToken` | Telegram bot token | Required |
| `telegramAllowedUsers` | List of permitted Telegram user IDs | Required |
| `telegramPollTimeout` | Telegram `getUpdates` long polling timeout (seconds). Higher values reduce QPS while keeping responses real-time. Range: 1–60. | `30` |
| `channelNetworkScript` | Path to a TS file (absolute, or relative to `<remote_agents_dir>`) whose default-export `async function (channelType): Promise<{ proxy?: string; skipEtcHosts?: boolean } \| undefined>` returns per-channel network config: optional `proxy` (http(s) URL — channel routes through this proxy; `undefined` means direct, regardless of `process.env.HTTPS_PROXY`) and optional `skipEtcHosts` (when `true`, the channel's API host DNS resolution skips `/etc/hosts` and goes straight to DNS via `dns.resolve4` / `dns.resolve6`; on failure the channel surfaces a loud error — no silent fallback). Returning `undefined` or `{}` means "default behavior" for that channel. The loader is strict: a malformed script (failed import, non-function default export, thrown invocation, non-object return, unknown object key, invalid http(s) URL, or non-boolean `skipEtcHosts`) aborts daemon startup with a clear `ParseError`. **Breaking change** from prior versions named `channelProxyScript` returning `string \| undefined` — rename the field and wrap the URL: `return 'http://...'` becomes `return { proxy: 'http://...' }`. | — |
| `commands` | Custom slash commands (see [Custom Commands](#custom-commands)) | — |
| `verbose` | When `true`, enables verbose logging. | `false` |

### Configuration merging

The daemon loads and merges configurations from the two locations above.

**Merge rules:**
- **Scalar fields** (string, number, boolean): project overrides global
- **Array fields**: project replaces global entirely (no concatenation)
- **Custom commands** (`commands` object): merged by command name; project overrides global for duplicate names

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

This lets you define Telegram credentials once globally and override the model, tools, or commands per project without duplicating credentials.

## License

GPL-3.0
