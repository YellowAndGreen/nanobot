# CLI 命令详解 (commands.py)

本文档详细解释 `nanobot/cli/commands.py` 文件，这是 nanobot 的命令行接口入口。

## 文件概述

| 属性 | 值 |
|------|-----|
| 路径 | `nanobot/cli/commands.py` |
| 行数 | ~1,117 行 |
| 框架 | Typer (基于 Click) |
| 作用 | 定义所有 CLI 命令 |

## 架构图

```
nanobot (主命令)
├── onboard          # 初始化配置
├── agent            # 与代理交互
├── gateway          # 启动网关服务
├── status           # 查看状态
├── channels         # 渠道管理 (子命令组)
│   ├── status       # 渠道状态
│   └── login        # WhatsApp 登录
├── cron             # 定时任务 (子命令组)
│   ├── list         # 列出任务
│   ├── add          # 添加任务
│   ├── remove       # 删除任务
│   ├── enable       # 启用/禁用任务
│   └── run          # 手动运行任务
└── provider         # 提供商管理 (子命令组)
    └── login        # OAuth 登录
```

## 核心组件

### 1. 主应用对象

```python
app = typer.Typer(
    name="nanobot",
    help=f"{__logo__} nanobot - Personal AI Assistant",
    no_args_is_help=True,
)
```

- 使用 Typer 框架创建 CLI 应用
- `no_args_is_help=True`: 无参数时显示帮助信息

### 2. 辅助函数

#### 终端输入处理

| 函数 | 作用 |
|------|------|
| `_flush_pending_tty_input()` | 清除终端未读取的输入（防止模型生成时用户输入干扰） |
| `_restore_terminal()` | 恢复终端原始状态 |
| `_init_prompt_session()` | 初始化 prompt_toolkit 会话（支持历史记录、粘贴） |

#### 交互式输入

```python
async def _read_interactive_input_async() -> str:
    """使用 prompt_toolkit 读取用户输入"""
    return await _PROMPT_SESSION.prompt_async(
        HTML("<b fg='ansiblue'>You:</b> "),
    )
```

支持的特性：
- 多行粘贴（bracketed paste mode）
- 历史记录导航（上下箭头）
- 干净的显示（无残留字符）

---

## 命令详解

### `nanobot onboard`

**用途**: 初始化 nanobot 配置和工作空间

**流程**:
```
1. 检查配置文件是否存在 (~/.nanobot/config.json)
   ├── 存在 → 询问是否覆盖或保留现有值
   └── 不存在 → 创建默认配置
2. 创建工作空间目录 (~/.nanobot/workspace/)
3. 同步模板文件 (SOUL.md, USER.md, TOOLS.md 等)
4. 显示下一步提示
```

**输出示例**:
```
✓ Created config at /home/user/.nanobot/config.json
✓ Created workspace at /home/user/.nanobot/workspace

🤖 nanobot is ready!

Next steps:
  1. Add your API key to ~/.nanobot/config.json
     Get one at: https://openrouter.ai/keys
  2. Chat: nanobot agent -m "Hello!"
```

---

### `nanobot agent`

**用途**: 直接与 AI 代理交互

**参数**:
| 参数 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--message` | `-m` | None | 单次消息模式 |
| `--session` | `-s` | `cli:direct` | 会话 ID |
| `--markdown` | | True | 是否渲染 Markdown |
| `--logs` | | False | 显示运行时日志 |

**两种模式**:

#### 单次消息模式
```bash
nanobot agent -m "你好！"
```
- 直接调用 `agent_loop.process_direct()`
- 不使用消息总线

#### 交互模式
```bash
nanobot agent
```
- 通过消息总线路由消息
- 支持上下箭头历史记录
- 输入 `exit` / `quit` / `:q` 退出

**交互模式流程**:
```
1. 初始化 prompt_toolkit 会话
2. 启动 agent_loop.run() 和消息消费协程
3. 循环读取用户输入
   ├── 发送到消息总线
   ├── 等待响应
   └── 显示结果
4. Ctrl+C 或 exit 退出
```

---

### `nanobot gateway`

**用途**: 启动多渠道网关服务

**参数**:
| 参数 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--port` | `-p` | 18790 | 网关端口 |
| `--verbose` | `-v` | False | 详细输出 |

**启动流程**:
```
1. 加载配置
2. 创建消息总线 (MessageBus)
3. 创建 LLM 提供商
4. 创建会话管理器 (SessionManager)
5. 创建定时任务服务 (CronService)
6. 创建代理循环 (AgentLoop)
7. 创建渠道管理器 (ChannelManager)
8. 创建心跳服务 (HeartbeatService)
9. 启动所有服务
```

**启动的服务**:
- `agent.run()` - 代理主循环
- `channels.start_all()` - 所有启用的渠道
- `cron.start()` - 定时任务服务
- `heartbeat.start()` - 心跳服务

**输出示例**:
```
🤖 Starting nanobot gateway on port 18790...
✓ Channels enabled: telegram, discord
✓ Cron: 3 scheduled jobs
✓ Heartbeat: every 1800s
```

---

### `nanobot status`

**用途**: 显示 nanobot 配置状态

**显示内容**:
- 配置文件路径和是否存在
- 工作空间路径和是否存在
- 当前使用的模型
- 各提供商 API Key 配置状态

**输出示例**:
```
🤖 nanobot Status

Config: /home/user/.nanobot/config.json ✓
Workspace: /home/user/.nanobot/workspace ✓
Model: openrouter/auto
OpenRouter: ✓
Anthropic: not set
OpenAI: not set
```

---

### `nanobot channels`

**用途**: 管理聊天渠道

#### `nanobot channels status`

显示所有渠道的配置状态：

```
Channel Status
┌───────────┬─────────┬──────────────────────────────┐
│ Channel   │ Enabled │ Configuration                │
├───────────┼─────────┼──────────────────────────────┤
│ WhatsApp  │ ✓       │ ws://localhost:8080          │
│ Discord   │ ✗       │                              │
│ Telegram  │ ✓       │ token: 123456789...          │
│ Slack     │ ✗       │ not configured               │
│ Feishu    │ ✗       │ not configured               │
│ ...       │ ...     │ ...                          │
└───────────┴─────────┴──────────────────────────────┘
```

#### `nanobot channels login`

通过二维码链接 WhatsApp 设备：

1. 检查并构建 Node.js bridge
2. 启动 bridge 服务
3. 显示二维码供扫描

---

### `nanobot cron`

**用途**: 管理定时任务

#### `nanobot cron list`

列出所有定时任务：

```bash
nanobot cron list        # 只显示启用的
nanobot cron list --all  # 包含禁用的
```

```
Scheduled Jobs
┌──────┬─────────────┬───────────────────┬─────────┬─────────────────┐
│ ID   │ Name        │ Schedule          │ Status  │ Next Run        │
├──────┼─────────────┼───────────────────┼─────────┼─────────────────┤
│ abc1 │ Morning     │ 0 9 * * * (UTC)   │ enabled │ 2024-01-15 09:00│
│ def2 │ Hourly      │ every 3600s       │ enabled │ 2024-01-15 10:00│
│ ghi3 │ One-time    │ one-time          │ enabled │ 2024-01-20 15:00│
└──────┴─────────────┴───────────────────┴─────────┴─────────────────┘
```

#### `nanobot cron add`

添加定时任务：

```bash
# 每小时执行
nanobot cron add -n "Hourly check" -m "Check system status" -e 3600

# 每天 9:00 执行（Cron 表达式）
nanobot cron add -n "Morning" -m "Good morning!" -c "0 9 * * *"

# 指定时区
nanobot cron add -n "Morning" -m "Good morning!" -c "0 9 * * *" --tz Asia/Shanghai

# 一次性任务
nanobot cron add -n "Reminder" -m "Don't forget!" --at "2024-01-20T15:00:00"

# 发送结果到渠道
nanobot cron add -n "Alert" -m "Check the server" -e 3600 -d --to "user123" --channel telegram
```

**参数**:
| 参数 | 说明 |
|------|------|
| `--name, -n` | 任务名称 |
| `--message, -m` | 发送给代理的消息 |
| `--every, -e` | 间隔秒数 |
| `--cron, -c` | Cron 表达式 |
| `--tz` | 时区 (IANA 格式) |
| `--at` | 一次性任务时间 (ISO 格式) |
| `--deliver, -d` | 是否发送结果到渠道 |
| `--to` | 接收者 |
| `--channel` | 发送渠道 |

#### `nanobot cron remove`

删除任务：
```bash
nanobot cron remove abc1
```

#### `nanobot cron enable`

启用/禁用任务：
```bash
nanobot cron enable abc1        # 启用
nanobot cron enable abc1 --disable  # 禁用
```

#### `nanobot cron run`

手动运行任务：
```bash
nanobot cron run abc1       # 运行启用的任务
nanobot cron run abc1 -f    # 强制运行（即使已禁用）
```

---

### `nanobot provider`

**用途**: 管理提供商认证

#### `nanobot provider login`

OAuth 登录：

```bash
nanobot provider login openai-codex
nanobot provider login github-copilot
```

支持的 OAuth 提供商：
- `openai-codex` - OpenAI Codex
- `github-copilot` - GitHub Copilot

---

## 内部函数

### `_make_provider()`

根据配置创建 LLM 提供商：

```python
def _make_provider(config: Config):
    model = config.agents.defaults.model
    provider_name = config.get_provider_name(model)

    # OpenAI Codex (OAuth)
    if provider_name == "openai_codex":
        return OpenAICodexProvider(default_model=model)

    # Custom: 直连 OpenAI 兼容端点
    if provider_name == "custom":
        return CustomProvider(...)

    # 默认: LiteLLM
    return LiteLLMProvider(...)
```

---

## 依赖关系

```
commands.py
    │
    ├── typer           # CLI 框架
    ├── prompt_toolkit  # 交互式输入
    ├── rich            # 终端美化输出
    │
    └── nanobot 模块
        ├── agent.loop         # AgentLoop
        ├── bus.queue          # MessageBus
        ├── channels.manager   # ChannelManager
        ├── config.loader      # 配置加载
        ├── cron.service       # CronService
        ├── heartbeat.service  # HeartbeatService
        ├── session.manager    # SessionManager
        └── providers.*        # LLM 提供商
```

---

## 退出命令

交互模式下支持的退出命令：
- `exit`
- `quit`
- `/exit`
- `/quit`
- `:q`
- `Ctrl+C`
- `Ctrl+D` (EOF)

---

## 历史记录

交互模式的输入历史保存在：
```
~/.nanobot/history/cli_history
```

使用上下箭头可以导航历史记录。
