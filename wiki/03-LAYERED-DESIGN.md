# 分层设计：四层架构详解

> 核心问题：每一层的职责边界在哪里？

## 架构分层概览

```
┌─────────────────────────────────────────────────────────────┐
│                 第一层：Channel Layer                        │
│              （通道层 - 平台差异隔离）                        │
├─────────────────────────────────────────────────────────────┤
│                 第二层：Message Bus                          │
│              （消息总线 - 异步消息传递）                      │
├─────────────────────────────────────────────────────────────┤
│                 第三层：Agent Core                           │
│              （代理核心 - 业务逻辑处理）                      │
├─────────────────────────────────────────────────────────────┤
│                 第四层：Provider Layer                       │
│              （提供者层 - LLM 调用抽象）                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 第一层：Channel Layer（通道层）

### 职责

1. **接收平台消息** → 转换为 `InboundMessage`
2. **接收 `OutboundMessage`** → 转换为平台消息格式
3. **处理平台特有功能**（如 Telegram 的 MarkdownV2 转义）
4. **权限检查**（allow_from 白名单）

### 不关心

- 消息如何被处理
- LLM 是什么模型
- 记忆如何存储

### 关键抽象

```python
class BaseChannel(ABC):
    name: str = "base"

    @abstractmethod
    async def start(self) -> None:
        """启动通道，开始监听消息"""
        pass

    @abstractmethod
    async def stop(self) -> None:
        """停止通道，清理资源"""
        pass

    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        """发送消息到平台"""
        pass

    def is_allowed(self, sender_id: str) -> bool:
        """检查发送者是否有权限"""
        allow_list = getattr(self.config, "allow_from", [])
        if not allow_list:
            return True
        return str(sender_id) in allow_list
```

**源码位置**：`nanobot/channels/base.py:12-131`

### 设计决策

#### 为什么用 dataclass 而不是 dict？

```python
# 方案 A：dict
message = {"channel": "telegram", "content": "hello"}

# 方案 B：dataclass
@dataclass
class InboundMessage:
    channel: str
    content: str
```

**选择 dataclass 的理由**：
1. **类型安全**：IDE 自动补全、静态检查
2. **不可变性**：可以用 `frozen=True`
3. **文档性**：字段有明确的类型注解
4. **扩展性**：可以添加属性和方法

#### 为什么统一消息格式？

不同平台的消息格式差异很大：
- Telegram: `message.from.id`
- Discord: `message.author.id`
- Slack: `message.user`

统一为 `InboundMessage` 后，Agent 无需关心消息来源：

```python
# Agent 代码不需要知道是哪个平台
async def process_message(self, msg: InboundMessage):
    # msg.channel, msg.sender_id, msg.content 总是可用
    ...
```

### 实现示例

```python
class TelegramChannel(BaseChannel):
    name = "telegram"

    async def start(self):
        self.app = Application.builder().token(self.config.token).build()
        self.app.add_handler(MessageHandler(filters.TEXT, self._handle))
        await self.app.run_polling()

    async def _handle(self, update, context):
        msg = update.message
        await self._handle_message(
            sender_id=str(msg.from_user.id),
            chat_id=str(msg.chat.id),
            content=msg.text,
        )

    async def send(self, msg: OutboundMessage):
        await self.app.bot.send_message(
            chat_id=msg.chat_id,
            text=msg.content,
        )
```

**源码位置**：`nanobot/channels/telegram.py`

---

## 第二层：Message Bus（消息总线）

### 职责

1. **异步消息队列**
2. **解耦 Channel 和 Agent**
3. **支持多通道并发**

### 不关心

- 消息内容是什么
- 消息如何被处理

### 关键实现

```python
class MessageBus:
    """异步消息总线，解耦通道与代理核心"""

    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg: InboundMessage) -> None:
        """发布入站消息（Channel → Agent）"""
        await self.inbound.put(msg)

    async def consume_inbound(self) -> InboundMessage:
        """消费入站消息（阻塞直到有消息）"""
        return await self.inbound.get()

    async def publish_outbound(self, msg: OutboundMessage) -> None:
        """发布出站消息（Agent → Channel）"""
        await self.outbound.put(msg)

    async def consume_outbound(self) -> OutboundMessage:
        """消费出站消息（阻塞直到有消息）"""
        return await self.outbound.get()
```

**源码位置**：`nanobot/bus/queue.py:8-44`

### 设计决策

#### 为什么用 asyncio.Queue？

**替代方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| `asyncio.Queue` | 简单、内置、无锁 | 单进程 |
| `multiprocessing.Queue` | 多进程支持 | 需要 pickle、复杂 |
| Redis Queue | 分布式支持 | 依赖外部服务 |
| ZeroMQ | 高性能 | 依赖外部库 |

**选择 asyncio.Queue 的理由**：
1. **简单**：Python 内置，无需依赖
2. **高效**：内存操作，无序列化开销
3. **足够**：单机部署足够用

#### 为什么分离 inbound/outbound？

```python
# 方案 A：单一队列
bus = MessageBus()
await bus.publish(msg)  # 入站和出站混在一起

# 方案 B：分离队列
bus = MessageBus()
await bus.publish_inbound(msg)   # 入站
await bus.publish_outbound(msg)  # 出站
```

**选择分离的理由**：
1. **方向清晰**：入站和出站流向不同
2. **独立消费**：Agent 只关心入站，Channel 只关心出站
3. **便于监控**：可以分别统计队列大小

### 消费模式

```python
# AgentLoop 消费入站消息
async def run(self):
    while self._running:
        try:
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )
            await self._dispatch(msg)
        except asyncio.TimeoutError:
            continue

# Channel 消费出站消息
async def _consume_loop(self):
    while self._running:
        msg = await self.bus.consume_outbound()
        if msg.channel == self.name:
            await self.send(msg)
```

---

## 第三层：Agent Core（代理核心）

### 职责

1. **消息处理循环**
2. **上下文构建**（系统提示词 + 历史 + 记忆）
3. **工具调用**
4. **记忆管理**
5. **子代理调度**

### 不关心

- 消息来自哪个平台
- LLM 是哪个提供商

### 核心组件

```python
class AgentLoop:
    def __init__(self, bus, provider, workspace, ...):
        self.bus = bus                    # 消息总线
        self.provider = provider          # LLM 提供商
        self.context = ContextBuilder(workspace)  # 上下文构建器
        self.sessions = SessionManager(workspace) # 会话管理器
        self.tools = ToolRegistry()       # 工具注册表
        self.subagents = SubagentManager(...)  # 子代理管理器
```

**源码位置**：`nanobot/agent/loop.py:35-113`

### 设计决策

#### 为什么把 ContextBuilder 独立出来？

```python
# 方案 A：在 AgentLoop 内部构建
class AgentLoop:
    def _build_system_prompt(self):
        # 所有逻辑混在一起
        ...

# 方案 B：独立的 ContextBuilder
class ContextBuilder:
    def build_system_prompt(self):
        ...

class AgentLoop:
    def __init__(self):
        self.context = ContextBuilder(workspace)
```

**选择独立的理由**：
1. **单一职责**：ContextBuilder 只负责上下文构建
2. **可测试**：可以单独测试上下文构建逻辑
3. **可复用**：子代理也可以使用同样的上下文构建

#### 为什么用 ToolRegistry 而不是直接调用？

```python
# 方案 A：直接调用
if tool_name == "read_file":
    result = await read_file(**args)
elif tool_name == "exec":
    result = await exec(**args)

# 方案 B：ToolRegistry
result = await self.tools.execute(tool_name, args)
```

**选择 ToolRegistry 的理由**：
1. **可扩展**：添加新工具只需注册
2. **统一接口**：所有工具遵循相同的执行模式
3. **元数据管理**：工具描述、参数 schema 统一管理

### 处理流程

```python
async def _process_message(self, msg: InboundMessage) -> OutboundMessage:
    # 1. 获取或创建会话
    session = self.sessions.get_or_create(msg.session_key)

    # 2. 构建上下文
    history = session.get_history(max_messages=self.memory_window)
    messages = self.context.build_messages(
        history=history,
        current_message=msg.content,
    )

    # 3. 运行代理循环
    final_content, tools_used, all_msgs = await self._run_agent_loop(messages)

    # 4. 保存会话
    self._save_turn(session, all_msgs, len(history))
    self.sessions.save(session)

    # 5. 返回响应
    return OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=final_content,
    )
```

**源码位置**：`nanobot/agent/loop.py:330-453`

---

## 第四层：Provider Layer（提供者层）

### 职责

1. **统一 LLM 调用接口**
2. **处理不同模型的差异**
3. **管理认证和限流**

### 不关心

- 消息内容是什么
- 工具如何执行

### 关键抽象

```python
@dataclass
class ProviderSpec:
    """LLM 提供商元数据"""
    name: str                    # 配置字段名
    keywords: tuple[str, ...]    # 模型名匹配关键词
    env_key: str                 # 环境变量名
    litellm_prefix: str          # LiteLLM 前缀
    is_gateway: bool             # 是否是网关（如 OpenRouter）
    is_oauth: bool               # 是否用 OAuth
    is_direct: bool              # 是否直连（绕过 LiteLLM）
    ...
```

**源码位置**：`nanobot/providers/registry.py:20-66`

### 设计决策

#### 为什么用 LiteLLM？

**问题**：不同 LLM 的 API 差异很大。

```python
# Anthropic
client.messages.create(
    model="claude-3-opus",
    messages=[...],
    system="...",
)

# OpenAI
client.chat.completions.create(
    model="gpt-4",
    messages=[...],  # system 在 messages 里
)
```

**解决方案**：LiteLLM 统一接口。

```python
# 统一调用
from litellm import acompletion

response = await acompletion(
    model="claude-3-opus",  # 自动识别提供商
    messages=[...],
    tools=[...],
)
```

#### 为什么需要 ProviderSpec？

**问题**：不同提供商需要不同的配置。

| 提供商 | 环境变量 | 模型前缀 |
|--------|---------|---------|
| Anthropic | `ANTHROPIC_API_KEY` | 无 |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek/` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter/` |

**解决方案**：元数据驱动。

```python
PROVIDERS = (
    ProviderSpec(
        name="anthropic",
        keywords=("claude",),
        env_key="ANTHROPIC_API_KEY",
        litellm_prefix="",
    ),
    ProviderSpec(
        name="deepseek",
        keywords=("deepseek",),
        env_key="DEEPSEEK_API_KEY",
        litellm_prefix="deepseek/",
    ),
)
```

#### 为什么支持 Direct 模式？

**场景**：私有部署的 OpenAI 兼容端点。

```python
ProviderSpec(
    name="custom",
    is_direct=True,  # 绕过 LiteLLM
)
```

直连模式下，可以直接调用自定义端点：

```python
class CustomProvider:
    async def chat(self, messages, ...):
        async with aiohttp.ClientSession() as session:
            async with session.post(
                self.api_base + "/chat/completions",
                json={"messages": messages, ...},
            ) as resp:
                return await resp.json()
```

---

## 横切关注点

### Session：跨层的会话状态管理

```python
@dataclass
class Session:
    key: str  # channel:chat_id
    messages: list[dict]  # JSONL 格式存储
    last_consolidated: int  # 上次压缩位置
```

**为什么用 JSONL？**

```jsonl
{"_type": "metadata", "key": "telegram:123", "created_at": "..."}
{"role": "user", "content": "Hello", "timestamp": "..."}
{"role": "assistant", "content": "Hi!", "timestamp": "..."}
```

1. **追加写入**：新消息直接 append，不需要重写整个文件
2. **LLM 缓存友好**：文件内容不变时，缓存仍然有效
3. **可读性**：每行一个 JSON，便于调试

**源码位置**：`nanobot/session/manager.py`

### Config：全局配置访问

```python
@dataclass
class Config:
    model: str
    providers: ProvidersConfig
    channels: ChannelsConfig
    agents: AgentsConfig
```

**设计特点**：
1. **Pydantic 验证**：类型安全、自动校验
2. **camelCase/snake_case 双支持**：兼容不同的配置风格
3. **环境变量覆盖**：敏感信息可以从环境变量读取

**源码位置**：`nanobot/config/schema.py`

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
- **Channel** 不依赖 Agent 实现
- **Agent** 不依赖具体 Channel
- **Provider** 不依赖 Agent
- **核心层** 可以独立测试

---

## 分层的好处

### 可测试性

```python
# 测试 Agent，不需要真实的 Channel
bus = MessageBus()
agent = AgentLoop(bus=bus, provider=mock_provider, ...)
await bus.publish_inbound(InboundMessage(...))
response = await bus.consume_outbound()
```

### 可扩展性

```python
# 添加新 Channel，不需要修改 Agent
class NewChannel(BaseChannel):
    ...
```

### 可替换性

```python
# 替换 Provider，不需要修改其他层
agent = AgentLoop(provider=NewProvider(), ...)
```

---

下一章：[04-MINIMAL-VIABLE.md](./04-MINIMAL-VIABLE.md) — 最小可行产品的诞生
