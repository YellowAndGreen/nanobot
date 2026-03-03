# 架构愿景：整体设计的顶层思考

> 核心问题：一个好的 AI 助手框架应该长什么样？

## 架构设计原则

### 原则一：核心极简

**目标**：~4000 行代码完成核心功能。

为什么？

1. **可读性**：一个人可以在一天内读完所有核心代码
2. **可维护性**：代码越少，bug 越少
3. **可理解性**：新人可以快速上手

```
核心模块（计入 4000 行）:
├── agent/loop.py        # 代理循环
├── agent/context.py     # 上下文构建
├── agent/memory.py      # 记忆管理
├── agent/skills.py      # 技能加载
├── bus/events.py        # 消息定义
├── bus/queue.py         # 消息总线
├── channels/base.py     # 通道抽象
├── providers/registry.py # 提供商注册
└── session/manager.py   # 会话管理

扩展模块（不计入）:
├── channels/telegram.py # 具体平台实现
├── channels/discord.py
├── providers/litellm_provider.py
└── ...
```

### 原则二：插件化扩展

**目标**：Channel、Provider、Tool、Skill 都是插件。

```python
# 添加新通道：继承 BaseChannel
class WeChatChannel(BaseChannel):
    name = "wechat"
    async def start(self): ...
    async def send(self, msg): ...

# 添加新提供商：添加 ProviderSpec
PROVIDERS = (
    ...,
    ProviderSpec(name="new_provider", keywords=("new",), ...),
)

# 添加新工具：继承 BaseTool
class MyTool(Tool):
    name = "my_tool"
    async def execute(self, **args): ...
```

### 原则三：异步优先

**目标**：所有 I/O 操作都是异步的。

为什么？

1. LLM 调用可能需要几十秒
2. 多个通道需要并发处理
3. 异步是 Python 现代 I/O 的标准

```python
# 所有 I/O 操作使用 async/await
async def process_message(self, msg: InboundMessage) -> OutboundMessage:
    response = await self.provider.chat(...)  # 异步 LLM 调用
    result = await self.tools.execute(...)    # 异步工具执行
    return OutboundMessage(...)
```

### 原则四：关注点分离

**目标**：通道无关、模型无关、存储无关。

| 层 | 职责 | 不关心 |
|----|------|--------|
| Channel | 消息收发 | LLM 如何处理 |
| Agent | 业务逻辑 | 消息来自哪个平台 |
| Provider | LLM 调用 | 消息如何返回 |

---

## 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      External World                          │
│  Telegram │ Discord │ Slack │ WhatsApp │ Feishu │ Email    │
└─────┬─────────┬─────────┬──────────┬─────────┬──────────────┘
      │         │         │          │         │
      ▼         ▼         ▼          ▼         ▼
┌─────────────────────────────────────────────────────────────┐
│                     Channel Layer                            │
│     BaseChannel → InboundMessage / OutboundMessage           │
│                                                              │
│  职责: 平台差异隔离、权限检查、消息格式转换                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Message Bus                              │
│              AsyncIO Queue (Inbound / Outbound)              │
│                                                              │
│  职责: 解耦通道与代理、异步消息传递、并发支持                 │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Agent Core                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ AgentLoop    │←→│ ContextBuild │←→│ MemoryStore  │       │
│  └──────┬───────┘  └──────────────┘  └──────────────┘       │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ ToolRegistry │  │ SkillsLoader │  │ SubagentMgr  │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  职责: 消息处理、上下文构建、工具调用、记忆管理               │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Provider Layer                           │
│  LiteLLMProvider → Anthropic │ OpenAI │ OpenRouter │ ...    │
│                                                              │
│  职责: 统一 LLM 接口、处理模型差异、管理认证                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 为什么这样分层？

### Channel Layer：隔离平台差异

**问题**：每个平台的消息格式不同。

```python
# Telegram 消息
{"message_id": 123, "from": {"id": 456}, "text": "Hello"}

# Discord 消息
{"id": "789", "author": {"id": "012"}, "content": "Hello"}
```

**解决方案**：统一为 `InboundMessage`。

```python
@dataclass
class InboundMessage:
    channel: str      # "telegram" / "discord" / ...
    sender_id: str    # "456" / "012" / ...
    chat_id: str      # 会话 ID
    content: str      # "Hello"
```

**源码位置**：`nanobot/channels/base.py:12-131`, `nanobot/bus/events.py:9-24`

### Message Bus：解耦通道与代理

**问题**：如果 Channel 直接调用 AgentLoop，会紧耦合。

```
# 紧耦合（不好）
TelegramChannel.agent_loop.process(msg)

# 松耦合（好）
TelegramChannel.bus.publish_inbound(msg)
```

**解决方案**：用 asyncio.Queue 作为消息总线。

```python
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()
```

**源码位置**：`nanobot/bus/queue.py:8-44`

### Agent Core：核心业务逻辑

**职责**：
1. 接收消息
2. 构建上下文（系统提示词 + 历史记录 + 记忆）
3. 调用 LLM
4. 执行工具
5. 管理会话

**源码位置**：`nanobot/agent/loop.py:35-498`

### Provider Layer：隔离模型差异

**问题**：不同 LLM 的 API 不同。

```python
# Anthropic
client.messages.create(model="claude-3-opus", messages=[...])

# OpenAI
client.chat.completions.create(model="gpt-4", messages=[...])
```

**解决方案**：统一接口 + LiteLLM 适配。

```python
class LiteLLMProvider:
    async def chat(self, messages, tools, model):
        # LiteLLM 统一处理不同提供商
        return await acompletion(model=model, messages=messages, tools=tools)
```

**源码位置**：`nanobot/providers/registry.py`, `nanobot/providers/litellm_provider.py`

---

## 数据流

```
用户发送消息
    │
    ▼
┌─────────────────┐
│ TelegramChannel │
│   receive()     │
└────────┬────────┘
         │ InboundMessage
         ▼
┌─────────────────┐
│   MessageBus    │
│ publish_inbound │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   AgentLoop     │
│ process_message │
│                 │
│  ┌───────────┐  │
│  │ContextBld │──┼──▶ 构建系统提示词
│  └───────────┘  │
│  ┌───────────┐  │
│  │  Session  │──┼──▶ 加载历史记录
│  └───────────┘  │
│  ┌───────────┐  │
│  │ Provider  │──┼──▶ 调用 LLM
│  └───────────┘  │
│  ┌───────────┐  │
│  │   Tools   │──┼──▶ 执行工具调用
│  └───────────┘  │
└────────┬────────┘
         │ OutboundMessage
         ▼
┌─────────────────┐
│   MessageBus    │
│publish_outbound │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ TelegramChannel │
│     send()      │
└─────────────────┘
         │
         ▼
用户收到响应
```

---

## 横切关注点

### Session：会话状态管理

```python
@dataclass
class Session:
    key: str  # channel:chat_id
    messages: list[dict]  # 对话历史
    last_consolidated: int  # 上次压缩位置
```

**源码位置**：`nanobot/session/manager.py:16-70`

### Config：全局配置访问

```python
@dataclass
class Config:
    model: str
    providers: ProvidersConfig
    channels: ChannelsConfig
    ...
```

**源码位置**：`nanobot/config/schema.py`

### Logging：统一的日志格式

```python
from loguru import logger

logger.info("Processing message from {}:{}", msg.channel, msg.sender_id)
```

---

## 依赖方向

```
Channel → Bus → Agent → Provider
              ↓
           Session
              ↓
            Config
```

**规则**：依赖只能向下，不能向上。

这保证了：
- Agent 不依赖具体的 Channel 实现
- Provider 不依赖 Agent
- 核心层可以独立测试

---

## 扩展点

| 扩展点 | 方式 | 文件位置 |
|--------|------|---------|
| 新通道 | 继承 `BaseChannel` | `nanobot/channels/` |
| 新提供商 | 添加 `ProviderSpec` | `nanobot/providers/registry.py` |
| 新工具 | 继承 `BaseTool` | `nanobot/agent/tools/` |
| 新技能 | 创建 Markdown 文件 | `nanobot/skills/` 或 `~/.nanobot/workspace/skills/` |

---

下一章：[03-LAYERED-DESIGN.md](./03-LAYERED-DESIGN.md) — 每一层的职责边界在哪里？
