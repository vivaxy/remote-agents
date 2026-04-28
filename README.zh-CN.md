# remote-agents

[English](https://github.com/vivaxy/remote-agents/blob/main/README.md) | [简体中文](https://github.com/vivaxy/remote-agents/blob/main/README.zh-CN.md)

> 在你的机器上运行 coding agent，通过消息渠道随时随地操控。

## 快速开始

### 前置条件

至少安装并认证一个受支持的 agent runtime：

- [Claude CLI](https://claude.ai/code)，用于默认的 `claude` runtime
- [Codex CLI](https://developers.openai.com/codex/sdk)，用于 `codex` runtime

### 安装（macOS arm64 / x64）

```bash
curl -fsSL https://github.com/vivaxy/remote-agents/releases/latest/download/install.sh | sh
```

### 设置 Telegram bot

1. **创建 bot**：通过 [@BotFather](https://t.me/BotFather) 发送 `/newbot` 并按提示操作。保存 token（格式：`123456:ABC-DEF...`）。
2. **获取用户 ID**：向 [@userinfobot](https://t.me/userinfobot) 发送任意消息。复制数字 ID（如 `987654321`）。

### 配置并运行

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

向 bot 发送任意消息 — 输出以 Telegram 消息流式返回，并保存到 `.remote-agents/YYYY-MM-DD-HH-mm-SS-mmm.md`。

### 微信

在 `remote-agents.json` 中设置 `channelType: "wechat"`：

```json
{
  "channelType": "wechat"
}
```

首次运行时，守护进程会在终端打印二维码。用微信扫码授权。

## 使用 bot

### 发送输入

- 每条消息立即触发执行。
- 输出以消息流式返回。
- 快速发送的多条消息会合并处理。

### 工具审批

当 agent 请求使用某个工具时，会出现四个按钮：

| 按钮 | 行为 |
|------|------|
| **✅ Allow** | 仅批准本次调用。下次遇到相同工具仍会询问。 |
| **✅ Allow (Session)** | 批准本次调用，并在本 session 内（直到 `/new` 或守护进程退出）自动通过该工具。 |
| **✅ Allow (Always)** | 批准并持久化写入项目的 `.claude/settings.local.json`，重启或 `/new` 后依然生效。 |
| **❌ Deny** | 拒绝本次调用；系统会提示填写拒绝原因。 |

Session 内批准的工具累积在 `allowedTools` 中，直到执行 `/new`。

## 命令

| 命令 | 动作 |
|---|---|
| `/permission_mode default\|acceptEdits\|bypassPermissions\|plan\|dontAsk` | 切换权限模式；不调用 agent |
| `/allowed_tools [Tool1 ...]` | 完全覆盖会话 `allowedTools`（空参数 = 清空；**无参数 = 查询当前值**） |
| `/add_allowed_tools [Tool1 ...]` | 追加工具到会话 `allowedTools`（无参数 = 无操作） |
| `/remove_allowed_tools [Tool1 ...]` | 从会话 `allowedTools` 中移除工具（无参数 = 无操作） |
| `/new` | 重置会话（清空统计、会话 ID、工具、模式）并显示新状态 |
| `/quit` | 干净退出守护进程 |
| `/state` | 显示当前守护进程状态 |
| `/help` | 显示使用说明及所有命令 |
| `/start` | 显示欢迎消息和当前状态（状态、会话 ID、权限模式、允许工具、累计成本） |
| `/verbose [true\|false]` | 查看或设置 verbose 模式。无参数时显示当前值并提供切换按钮。`true`/`false` 直接设置并持久化到 `remote-agents.json` |
| *（无命令）* | 复用上次模式和持久化工具；调用 agent |

## 自定义命令

在 `<remote_agents_dir>/remote-agents.json` 中定义项目专属斜杠命令：

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

**命令名约束：** 名称必须是小写 `snake_case` —— 只允许 `a-z`、`0-9` 和 `_`；不能以下划线开头或结尾，不能有连续下划线。非法名称在启动时以警告形式跳过。

像内置命令一样使用：`/review` 后跟 `重点检查安全问题`。

## 配置

设置从两个位置级联：

1. **全局配置**：`~/.remote-agents/remote-agents.json` —— 所有项目共享
2. **项目配置**：`<remote_agents_dir>/remote-agents.json` —— 项目特定覆盖

项目配置覆盖全局配置。

### 命令行参数

```bash
remote-agents [--working-dir <path>] [--remote-agents-dir <path>]
```

| 标志 | 说明 | 默认值 |
|------|------|--------|
| `--working-dir` | 项目源码目录 | 当前目录 |
| `--remote-agents-dir` | 状态、日志、上下文和输入文件目录 | `<working_dir>/.remote-agents` |

### `remote-agents.json`

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `runtimeProvider` | Agent runtime：`claude` 或 `codex` | `"claude"` |
| `model` | 模型覆盖（如 `claude-sonnet-4-6`） | — |
| `agentEnvScript` | TS 脚本路径（绝对路径或相对于 `<remote_agents_dir>`），需 default-export `async (runtimeProvider): Promise<Record<string, string>>`。每次调用 agent 前执行，返回的环境变量合并入 `process.env`。脚本收到当前 provider，可按 `claude`/`codex` 返回不同的环境变量。 | — |
| `defaultAllowedTools` | 启动时及 `/new` 后应用的默认允许工具列表 | — |
| `defaultPermissionMode` | 首次运行或 `/new` 后的默认权限模式：`default`\|`acceptEdits`\|`bypassPermissions`\|`plan`\|`dontAsk` | `default` |
| `telegramBotToken` | Telegram bot token | 必需 |
| `telegramAllowedUsers` | 允许的 Telegram 用户 ID 列表 | 必需 |
| `telegramPollTimeout` | 长轮询超时（秒，1–60）。更高的值减少 QPS 同时保持实时响应。 | `30` |
| `channelNetworkScript` | TS 文件路径（绝对路径或相对于 `<remote_agents_dir>`），需 default-export `async (channelType): Promise<{ proxy?: string; skipEtcHosts?: boolean } \| undefined>`。返回各频道的网络配置：`proxy` 让频道通过 HTTP(S) 代理出网；`skipEtcHosts` 让 API 主机 DNS 解析跳过 `/etc/hosts` 直接走 DNS。返回 `undefined` 或 `{}` 表示使用默认行为。脚本格式错误时守护进程启动阶段以 `ParseError` 终止。 | — |
| `commands` | 自定义斜杠命令（见[自定义命令](#自定义命令)） | — |
| `verbose` | 启用详细日志记录 | `false` |

### 配置合并

**合并规则：**
- **标量字段**（字符串、数字、布尔值）：项目覆盖全局
- **数组字段**：项目完全替换全局（不拼接）
- **自定义命令**（`commands` 对象）：按命令名合并；项目覆盖同名全局命令

**示例。** 全局配置（`~/.remote-agents/remote-agents.json`）：
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

项目配置（`<remote_agents_dir>/remote-agents.json`）：
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

最终合并配置：
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

在全局配置中定义一次 Telegram 凭证，在项目级覆盖模型、工具或命令，无需重复凭证。

## 目录结构

```
<working_dir>/
└── .remote-agents/
    ├── YYYY-MM-DD-HH-mm-SS-mmm.md   # agent 输出 + 存档输入
    ├── state.json                    # 运行时状态（JSON）
    ├── YYYY-MM-DD.log                # 守护进程诊断日志（每天一个文件）
    └── remote-agents.json            # 配置文件
```

## 许可证

GPL-3.0
