# 构建思维：如果让我从零开始

> 核心问题：如果让我从零开始构建一个 AI 助手，我会怎么想？

## 第一阶段：确定核心循环

任何交互系统都有一个核心循环。对于 AI 助手来说，最简单的循环是什么？

```python
while True:
    message = await receive_message()    # 1. 接收消息
    response = await llm.chat(message)   # 2. 调用 LLM
    await send_response(response)        # 3. 返回响应
```

这三行伪代码就是 AI 助手的本质。一切设计都围绕这个核心循环展开。

**关键洞察**：
- 这个循环必须是**异步**的 —— LLM 调用可能需要几秒甚至几十秒
- 这个循环必须是**可中断**的 —— 用户可能想停止当前任务
- 这个循环必须是**可扩展**的 —— 未来可能需要更多步骤

**源码位置**：`nanobot/agent/loop.py:180-257` (`_run_agent_loop`)

---

## 第二阶段：识别变化的维度

核心循环确定后，下一个问题是：**什么会变化？**

### 维度一：消息来源会变

用户可能从不同平台发起对话：
- Telegram
- Discord
- Slack
- WhatsApp
- CLI
- ...

**设计决策**：抽象出 `Channel` 概念，统一消息格式。

```python
@dataclass
class InboundMessage:
    channel: str      # 来源渠道
    sender_id: str    # 发送者
    chat_id: str      # 会话 ID
    content: str      # 消息内容
    media: list[str]  # 媒体附件
```

**源码位置**：`nanobot/bus/events.py:9-24`

### 维度二：LLM 会变

不同的 LLM 有不同的 API：
- Anthropic Claude
- OpenAI GPT
- DeepSeek
- 通过网关访问（OpenRouter）
- 私有部署（vLLM）

**设计决策**：抽象出 `Provider` 概念，统一调用接口。

```python
class LLMProvider(Protocol):
    async def chat(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        model: str | None = None,
    ) -> LLMResponse:
        ...
```

**源码位置**：`nanobot/providers/base.py`

### 维度三：能力会变

LLM 本身只能处理文本，但它需要：
- 读写文件
- 执行命令
- 搜索网络
- 发送消息
- ...

**设计决策**：抽象出 `Tool` 概念，让 LLM 通过工具调用与外界交互。

```python
class Tool:
    name: str
    description: str
    parameters: dict  # JSON Schema

    async def execute(self, **args) -> str:
        ...
```

**源码位置**：`nanobot/agent/tools/base.py`

---

## 第三阶段：隔离变化点

识别了变化的维度，下一步是**隔离它们**。

### 为什么需要 MessageBus？

如果我们让 Channel 直接调用 AgentLoop：

```
TelegramChannel → AgentLoop → TelegramChannel
```

问题：
1. AgentLoop 需要知道所有 Channel 类型
2. 无法支持多个 Channel 同时工作
3. 测试困难

**解决方案**：引入 MessageBus 作为中介。

```
TelegramChannel ↘
DiscordChannel  → MessageBus → AgentLoop
SlackChannel    ↗
```

**设计决策**：
- `inbound` 队列：Channel → Agent
- `outbound` 队列：Agent → Channel
- 使用 `asyncio.Queue` 实现

**源码位置**：`nanobot/bus/queue.py:8-44`

### 为什么需要 ProviderSpec？

不同的 LLM 有不同的：
- API Key 环境变量名
- 模型名前缀规则
- 认证方式（API Key vs OAuth）
- 特殊参数

**解决方案**：用 `ProviderSpec` 统一描述这些元数据。

```python
@dataclass
class ProviderSpec:
    name: str              # 配置字段名
    keywords: tuple[str]   # 模型名匹配关键词
    env_key: str           # 环境变量名
    litellm_prefix: str    # LiteLLM 前缀
    is_gateway: bool       # 是否是网关
    is_oauth: bool         # 是否用 OAuth
    ...
```

**源码位置**：`nanobot/providers/registry.py:20-66`

---

## 第四阶段：解决状态问题

AI 助手需要"记住"之前的对话。但怎么存储？

### 问题：会话如何管理？

每个会话由 `channel:chat_id` 唯一标识：
- `telegram:123456789`
- `discord:987654321`
- `cli:direct`

**设计决策**：
- 会话存储为 JSONL 文件（追加写入，LLM 缓存友好）
- 内存中缓存活跃会话
- 支持会话恢复

**源码位置**：`nanobot/session/manager.py:16-70`

### 问题：记忆如何持久化？

会话历史会无限增长，怎么办？

**设计决策**：双层记忆系统
1. **MEMORY.md**：长期事实（用户偏好、重要信息）
2. **HISTORY.md**：可搜索的日志

当历史超过阈值时，用 LLM 压缩：
```
100 条消息 → LLM 总结 → 更新 MEMORY.md + HISTORY.md → 保留最近 50 条
```

**源码位置**：`nanobot/agent/memory.py:45-150`

---

## 第五阶段：赋予主动性

被动响应的 AI 助手是不够的。我们需要它能够：
- 定时提醒
- 周期性检查
- 主动报告

### Cron：定时任务

```python
# 每小时检查
nanobot cron add -n "Hourly check" -m "Check system status" -e 3600

# 每天 9:00 提醒
nanobot cron add -n "Morning" -m "Good morning!" -c "0 9 * * *"
```

**设计决策**：
- 支持三种调度：`every`、`cron`、`at`
- 任务触发时发送消息给 AgentLoop
- 结果可选发送到指定渠道

**源码位置**：`nanobot/cron/service.py`

### Heartbeat：心跳检查

每 30 分钟，系统会让 LLM 检查 `HEARTBEAT.md`，决定是否执行某些任务。

**设计决策**：
- 用"虚拟工具调用"让 LLM 决定是否行动
- 避免硬编码逻辑

**源码位置**：`nanobot/heartbeat/service.py`

---

## 架构演进总结

```
阶段 1：核心循环
    消息进 → LLM 处理 → 响应出

阶段 2：识别变化
    Channel 变化 + Provider 变化 + Tool 变化

阶段 3：隔离变化
    MessageBus 解耦 + ProviderSpec 元数据

阶段 4：状态管理
    Session 会话 + Memory 记忆

阶段 5：主动性
    Cron 定时 + Heartbeat 心跳
```

最终架构：

```
┌─────────────────────────────────────────────────────────────┐
│                      External World                          │
│  Telegram │ Discord │ Slack │ WhatsApp │ CLI │ ...         │
└─────┬─────────┬─────────┬──────────┬─────────┬──────────────┘
      │         │         │          │         │
      ▼         ▼         ▼          ▼         ▼
┌─────────────────────────────────────────────────────────────┐
│                     Channel Layer                            │
│     BaseChannel → InboundMessage / OutboundMessage           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Message Bus                              │
│              AsyncIO Queue (Inbound / Outbound)              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Agent Core                               │
│  AgentLoop │ ContextBuilder │ MemoryStore │ ToolRegistry    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                     Provider Layer                           │
│  LiteLLMProvider → Anthropic │ OpenAI │ OpenRouter │ ...    │
└─────────────────────────────────────────────────────────────┘
```

---

## 设计原则总结

1. **核心极简**：~4000 行代码完成核心功能
2. **插件化扩展**：Channel、Provider、Tool、Skill 都是插件
3. **异步优先**：所有 I/O 操作都是异步的
4. **关注点分离**：通道无关、模型无关、存储无关
5. **追加写入**：JSONL 格式，LLM 缓存友好
6. **元数据驱动**：ProviderSpec 让配置自动化

---

下一章：[01-PROBLEM-SPACE.md](./01-PROBLEM-SPACE.md) — 我们到底要解决什么问题？
