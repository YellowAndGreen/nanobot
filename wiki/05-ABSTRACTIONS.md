# 关键抽象：稳定与变化

> 核心问题：哪些概念是稳定的，哪些是变化的？

## 抽象的本质

好的抽象能**隔离变化**。在 nanobot 中，我们识别出以下稳定概念：

| 稳定概念 | 变化实现 |
|---------|---------|
| Channel（通道） | Telegram, Discord, Slack, WhatsApp, Feishu, CLI... |
| Provider（提供者） | Anthropic, OpenAI, OpenRouter, DeepSeek, vLLM... |
| Tool（工具） | read_file, exec, web_search, cron, spawn... |
| Skill（技能） | github, weather, tmux, docker... |
| Message（消息） | InboundMessage, OutboundMessage |

---

## Channel：通道抽象

### 稳定的是什么？

**所有聊天平台都有**：
- 接收消息
- 发送消息
- 发送者标识
- 会话标识

### 变化的是什么？

**平台差异**：
- API 格式（Telegram Bot API vs Discord Gateway）
- 认证方式（Token vs OAuth）
- 消息格式（纯文本 vs Markdown vs 富文本）
- 限流规则

### 抽象设计

```python
class BaseChannel(ABC):
    """通道抽象基类"""

    name: str  # 通道名称，如 "telegram"

    @abstractmethod
    async def start(self) -> None:
        """启动通道"""
        pass

    @abstractmethod
    async def stop(self) -> None:
        """停止通道"""
        pass

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        """发送消息"""
        pass

    async def _handle_message(
        self,
        sender_id: str,
        chat_id: str,
        content: str,
        media: list[str] | None = None,
        metadata: dict | None = None,
    ) -> None:
        """处理收到的消息，统一转换为 InboundMessage"""
        if not self.is_allowed(sender_id):
            return

        msg = InboundMessage(
            channel=self.name,
            sender_id=str(sender_id),
            chat_id=str(chat_id),
            content=content,
            media=media or [],
            metadata=metadata or {},
        )
        await self.bus.publish_inbound(msg)
```

**源码位置**：`nanobot/channels/base.py:12-131`

### 实现示例

```python
# Telegram 实现
class TelegramChannel(BaseChannel):
    name = "telegram"

    async def start(self):
        self.app = Application.builder().token(self.config.token).build()
        self.app.add_handler(MessageHandler(filters.TEXT, self._handle))
        await self.app.run_polling()

    async def send(self, msg: OutboundMessage):
        await self.app.bot.send_message(
            chat_id=int(msg.chat_id),
            text=msg.content,
            parse_mode="MarkdownV2",
        )

# Discord 实现
class DiscordChannel(BaseChannel):
    name = "discord"

    async def start(self):
        intents = discord.Intents.default()
        intents.message_content = True
        self.client = discord.Client(intents=intents)
        self.client.event(self._on_message)
        await self.client.start(self.config.token)

    async def send(self, msg: OutboundMessage):
        channel = self.client.get_channel(int(msg.chat_id))
        await channel.send(msg.content)
```

---

## Provider：提供者抽象

### 稳定的是什么？

**所有 LLM 都支持**：
- 接收消息列表
- 返回文本响应
- 支持工具调用（大部分）

### 变化的是什么？

**提供商差异**：
- API 端点
- 认证方式（API Key vs OAuth）
- 模型命名规则
- 特殊参数（temperature, max_tokens）

### 抽象设计

```python
@dataclass
class LLMResponse:
    """统一的 LLM 响应格式"""
    content: str | None
    tool_calls: list[ToolCall]
    finish_reason: str
    reasoning_content: str | None = None

    @property
    def has_tool_calls(self) -> bool:
        return bool(self.tool_calls)

class LLMProvider(Protocol):
    """LLM 提供者协议"""

    async def chat(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        model: str | None = None,
        **kwargs,
    ) -> LLMResponse:
        ...

    def get_default_model(self) -> str:
        ...
```

### ProviderSpec：元数据驱动

```python
@dataclass
class ProviderSpec:
    """提供商元数据，驱动自动化配置"""
    name: str                       # 配置字段名
    keywords: tuple[str, ...]       # 模型名匹配关键词
    env_key: str                    # 环境变量名
    litellm_prefix: str             # LiteLLM 前缀
    is_gateway: bool                # 是否是网关
    is_oauth: bool                  # 是否用 OAuth
    is_direct: bool                 # 是否直连
    ...
```

**源码位置**：`nanobot/providers/registry.py:20-66`

### 查找示例

```python
# 根据模型名找提供商
spec = find_by_model("claude-3-opus")
# → ProviderSpec(name="anthropic", env_key="ANTHROPIC_API_KEY", ...)

# 根据配置找网关
spec = find_gateway(api_key="sk-or-xxx")
# → ProviderSpec(name="openrouter", is_gateway=True, ...)
```

---

## Tool：工具抽象

### 稳定的是什么？

**所有工具都有**：
- 名称
- 描述
- 参数定义（JSON Schema）
- 执行方法

### 变化的是什么？

**工具类型**：
- 文件操作：read_file, write_file, edit_file
- 命令执行：exec
- 网络访问：web_search, web_fetch
- 消息发送：message
- 任务调度：cron, spawn

### 抽象设计

```python
class BaseTool:
    """工具基类"""

    name: str
    description: str
    parameters: dict  # JSON Schema

    async def execute(self, **args) -> str:
        """执行工具，返回结果字符串"""
        raise NotImplementedError

    def get_definition(self) -> dict:
        """获取工具定义，用于 LLM 工具调用"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            }
        }
```

**源码位置**：`nanobot/agent/tools/base.py`

### 实现示例

```python
class ReadFileTool(BaseTool):
    name = "read_file"
    description = "Read the contents of a file"
    parameters = {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "The absolute path to the file",
            }
        },
        "required": ["path"],
    }

    def __init__(self, workspace: Path, allowed_dir: Path | None = None):
        self.workspace = workspace
        self.allowed_dir = allowed_dir

    async def execute(self, path: str) -> str:
        # 安全检查
        if self.allowed_dir and not path.startswith(str(self.allowed_dir)):
            return "Error: Access denied"

        file_path = Path(path)
        if not file_path.exists():
            return f"Error: File not found: {path}"

        return file_path.read_text()
```

**源码位置**：`nanobot/agent/tools/filesystem.py`

---

## Skill：技能抽象

### Tool vs Skill

| 特性 | Tool | Skill |
|------|------|-------|
| 形式 | Python 类 | Markdown 文件 |
| 加载 | 代码注册 | 动态加载 |
| 内容 | 执行逻辑 | 知识/指令 |
| 扩展 | 需要改代码 | 只需写文件 |

### 抽象设计

```markdown
---
name: github
description: "GitHub operations"
metadata:
  nanobot:
    emoji: 🐙
    requires:
      bins: [gh]
    install:
      - id: brew
        kind: brew
        formula: gh
        bins: [gh]
        label: Install via brew
---

# GitHub Skill

## Usage

Use `gh` CLI to interact with GitHub:

```bash
gh repo list
gh issue create --title "..." --body "..."
```
```

**源码位置**：`nanobot/skills/`

### 技能加载

```python
class SkillsLoader:
    def __init__(self, workspace: Path):
        self.skills_dir = workspace / "skills"

    def get_always_skills(self) -> list[SkillMeta]:
        """获取 always 类型的技能（自动加载）"""
        ...

    def build_skills_summary(self) -> str:
        """构建技能摘要，供 LLM 选择"""
        ...

    def load_skills_for_context(self, skills: list[SkillMeta]) -> str:
        """加载技能内容到上下文"""
        ...
```

**源码位置**：`nanobot/agent/skills.py`

---

## Message：消息抽象

### 稳定的是什么？

**所有消息都有**：
- 来源/目标
- 内容
- 时间戳

### 变化的是什么？

**消息类型**：
- 入站（用户 → AI）
- 出站（AI → 用户）

### 抽象设计

```python
@dataclass
class InboundMessage:
    """入站消息：从平台接收"""
    channel: str      # 来源渠道
    sender_id: str    # 发送者
    chat_id: str      # 会话 ID
    content: str      # 消息内容
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
    session_key_override: str | None = None

    @property
    def session_key(self) -> str:
        return self.session_key_override or f"{self.channel}:{self.chat_id}"

@dataclass
class OutboundMessage:
    """出站消息：发送到平台"""
    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None
    media: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
```

**源码位置**：`nanobot/bus/events.py:9-38`

---

## 抽象边界

```
┌─────────────────────────────────────────────────────────────┐
│                      外部世界                                │
│  Telegram │ Discord │ Claude │ GPT │ Files │ Web │ Cron    │
└─────┬─────────┬─────────┬──────────┬─────────┬──────────────┘
      │         │         │          │         │
      ▼         ▼         ▼          ▼         ▼
┌─────────────────────────────────────────────────────────────┐
│                      抽象层                                  │
│                                                             │
│  Channel ────▶ InboundMessage / OutboundMessage ◀─── Agent  │
│                                                             │
│  ProviderSpec ──▶ LLMProvider ◀─── AgentLoop               │
│                                                             │
│  BaseTool ──────▶ ToolRegistry ◀─── AgentLoop              │
│                                                             │
│  Skill ─────────▶ SkillsLoader ◀─── ContextBuilder         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**核心原则**：
- Agent 不直接依赖具体实现
- 所有交互通过抽象进行
- 新实现只需满足抽象接口

---

## 源码索引

| 抽象 | 文件位置 |
|------|---------|
| Channel | `nanobot/channels/base.py:12-131` |
| Message | `nanobot/bus/events.py:9-38` |
| Provider | `nanobot/providers/registry.py:20-66` |
| Tool | `nanobot/agent/tools/base.py` |
| Skill | `nanobot/agent/skills.py` |

---

下一章：[06-MESSAGE-FLOW.md](./06-MESSAGE-FLOW.md) — 一条消息是如何在系统中流动的？
