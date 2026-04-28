# remote-agents

[English](https://github.com/vivaxy/remote-agents/blob/main/README.md) | [简体中文](https://github.com/vivaxy/remote-agents/blob/main/README.zh-CN.md)

> 在 Telegram 上驱动你的 coding agent CLI。持久化、可恢复、运行在服务器端。

## 为什么选择 remote-agents

- 让 Claude Code 或 Codex 在你的服务器上运行，随时随地通过 Telegram 操控。
- 状态在守护进程重启后依然保留 —— 会话、允许工具、权限模式、累计成本。
- macOS 一行命令安装，主机无需安装 Node.js。
- 可审计 —— 每次输入和输出都保存为带时间戳的 Markdown 文件。
- 项目级自定义斜杠命令。

## 快速开始

### 前置条件

至少安装并认证一个受支持的 agent runtime：

- [Claude CLI](https://claude.ai/code)，用于默认的 `claude` runtime
- [Codex CLI](https://developers.openai.com/codex/sdk)，用于 `codex` runtime

一个 Telegram bot token 和你的 Telegram 用户 ID —— 见下文 [设置 Telegram bot](#设置-telegram-bot)。

### 安装（macOS arm64 / x64）

```bash
curl -fsSL https://github.com/vivaxy/remote-agents/releases/latest/download/install.sh | sh
```

安装脚本会自动检测 CPU 架构，下载匹配的 tarball，移除 Gatekeeper 隔离属性，并将二进制安装到 `/usr/local/bin`（若该目录不可写则回退到 `~/.local/bin`）。在非 darwin 系统或不支持的 CPU 上会以清晰错误信息退出。

可选环境变量：

| 变量 | 作用 | 默认值 |
|------|------|--------|
| `VERSION` | 锁定到指定发行版标签，例如 `VERSION=v0.1.0` | 最新发行版 |
| `INSTALL_DIR` | 覆盖安装目录（须已存在且可写） | `/usr/local/bin`，回退 `$HOME/.local/bin`（自动创建） |

### 设置 Telegram bot

1. **创建 bot**：通过 [@BotFather](https://t.me/BotFather) 发送 `/newbot` 并按提示操作。保存 bot token（格式：`123456:ABC-DEF...`）。
2. **获取用户 ID**：向 [@userinfobot](https://t.me/userinfobot) 发送任意消息。复制你的用户 ID（数字，如 `987654321`）。

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

向你的 bot 发送任意消息 —— 每条消息立即触发执行。输出以 Telegram 消息流式返回，并保存到 `.remote-agents/YYYY-MM-DD-HH-mm-SS-mmm.md`。

### 微信

在 `remote-agents.json` 中设置 `channelType: "wechat"`：

```json
{
  "channelType": "wechat"
}
```

首次运行时，守护进程会在标准输出打印微信二维码 URL，用微信扫码授权。凭证保存到 `<remote-agents-dir>/wechat-credentials.json`，后续运行时自动复用。

与 Telegram 不同，微信频道无需配置白名单 —— 二维码登录会将 bot 绑定到你的微信账号，只有该账号能与之对话。

## 目录结构

```
<working_dir>/
└── .remote-agents/
    ├── YYYY-MM-DD-HH-mm-SS-mmm.md   # 编码代理 CLI 输出 + 存档输入
    ├── state.json            # 运行时状态（JSON）
    ├── YYYY-MM-DD.log        # 守护进程诊断日志（每天一个文件）
    └── remote-agents.json    # 配置文件（Telegram 凭证所需）
```

## 使用 bot

### 发送输入

- 你发送的每一条消息都会立即触发执行 —— 无需 `#submit`。
- 输出以 Telegram 消息流式返回。
- 快速发送的多条消息会被合并并一起处理。
- 使用 `/permission_mode` 获取内联键盘以选择权限模式。

### 工具审批

当 agent 请求使用某个工具时，会弹出四个按钮：

| 按钮 | 行为 |
|------|------|
| **✅ Allow** | 仅批准本次调用。下次遇到相同工具仍会询问。 |
| **✅ Allow (Session)** | 批准本次调用，并在本 session 内（直到 `/new` 或守护进程退出）自动通过该工具的后续调用。 |
| **✅ Allow (Always)** | 批准本次调用，并将该工具**持久化**写入项目的 `.claude/settings.local.json`，重启或 `/new` 后依然生效。 |
| **❌ Deny** | 拒绝本次调用；系统会提示你填写拒绝原因。 |

Session 内批准的工具会累积写入 `allowedTools`，直到执行 `/new` 命令将权限重置为配置默认值。

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
| `/verbose [true\|false]` | 查看或设置 verbose 模式。无参数时显示当前值并提供切换按钮。使用 `true` 或 `false` 直接设置，并持久化到 `<remote_agents_dir>/remote-agents.json` |
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

> **命令名约束：** 自定义命令名必须是小写 `snake_case` —— 只允许小写字母（`a-z`）、数字（`0-9`）和下划线（`_`）；不能以下划线开头或结尾，不能包含连续下划线。不合法的命令名会在启动时以警告形式跳过。

像内置命令一样使用：`/review` 后跟 `重点检查安全问题`。

## 配置

守护进程支持**两级配置级联**，允许全局默认值和项目特定覆盖：

1. **全局配置**：`~/.remote-agents/remote-agents.json`（所有项目共享）
2. **项目配置**：`<remote_agents_dir>/remote-agents.json`（项目特定）

项目设置会覆盖全局设置。

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
| `runtimeProvider` | agent runtime 提供者（`claude` 或 `codex`） | `"claude"` |
| `model` | 模型覆盖（如 `claude-sonnet-4-6`） | — |
| `agentEnvScript` | TS 脚本路径（绝对路径或相对于 `<remote_agents_dir>`），需 default-export `async (runtimeProvider): Promise<Record<string, string>>`。脚本会收到当前活动的 runtime provider（`'claude'` 或 `'codex'`），从而支持单脚本按 provider 返回不同的环境变量（例如 `claude` 时返回 `ANTHROPIC_API_KEY`，`codex` 时返回 `OPENAI_API_KEY`）。每次调用 agent 前执行，返回的环境变量会与 `process.env` 合并。 | — |
| `defaultAllowedTools` | 默认允许工具列表，启动时及 `/new` 后应用 | — |
| `defaultPermissionMode` | 首次运行或 `/new` 后的默认权限模式（`default`\|`acceptEdits`\|`bypassPermissions`\|`plan`\|`dontAsk`） | `default` |
| `telegramBotToken` | Telegram bot token | 必需 |
| `telegramAllowedUsers` | 允许使用 bot 的 Telegram 用户 ID 列表 | 必需 |
| `telegramPollTimeout` | Telegram `getUpdates` 长轮询超时时间（秒）。更高的值减少 QPS 同时保持实时响应。范围：1–60。 | `30` |
| `channelNetworkScript` | 一个 TS 文件路径（绝对路径，或相对于 `<remote_agents_dir>`），默认导出 `async function (channelType): Promise<{ proxy?: string; skipEtcHosts?: boolean } \| undefined>`，按频道返回网络配置对象：可选 `proxy`（http(s) URL，频道经此代理出网；`undefined` 表示直连，且不会从 `process.env.HTTPS_PROXY` 继承）与可选 `skipEtcHosts`（为 `true` 时，频道的 API 主机 DNS 解析会跳过 `/etc/hosts`，直接走 `dns.resolve4` / `dns.resolve6`；解析失败时频道会抛出明确错误，不会回退到默认解析器）。返回 `undefined` 或 `{}` 表示该频道使用默认网络行为。加载器采用严格模式：脚本文件加载失败、默认导出非函数、调用抛错、返回值非对象、对象包含未知键、`proxy` 不是合法 http(s) URL，或 `skipEtcHosts` 不是布尔值时，守护进程会在启动阶段以 `ParseError` 直接失败。**不兼容变更**：旧版本字段名为 `channelProxyScript`，返回 `string \| undefined`；请重命名字段并把 URL 包成对象：`return 'http://...'` 改为 `return { proxy: 'http://...' }`。 | — |
| `commands` | 自定义斜杠命令（见[自定义命令](#自定义命令)） | — |
| `verbose` | 为 `true` 时启用详细日志记录。 | `false` |

### 配置合并

守护进程从上述两个位置加载并合并配置。

**合并规则：**
- **标量字段**（字符串、数字、布尔值）：项目配置覆盖全局配置
- **数组字段**：项目配置完全替换全局配置（不拼接）
- **自定义命令**（`commands` 对象）：按命令名合并；项目配置覆盖同名的全局命令

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

这样你可以在全局配置中定义一次 Telegram 凭证，并在项目级覆盖模型、工具或命令而无需重复凭证。

## 许可证

GPL-3.0
