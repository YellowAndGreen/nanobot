# 最小可行：核心循环的诞生

> 核心问题：最简单的 AI 助手需要什么？

## 从伪代码开始

最简单的 AI 助手，核心循环是什么？

```python
while True:
    message = await receive_message()    # 1. 接收消息
    response = await llm.chat(message)   # 2. 调用 LLM
    await send_response(response)        # 3. 返回响应
```

三行伪代码，道尽 AI 助手的本质。

---

## 为什么选择异步架构？

### 同步 vs 异步

```python
# 同步（阻塞）
def process():
    message = receive_message()  # 阻塞等待
    response = llm.chat(message)  # 阻塞调用，可能需要 30 秒
    send_response(response)       # 阻塞发送

# 异步（非阻塞）
async def process():
    message = await receive_message()
    response = await llm.chat(message)  # 不阻塞其他任务
    await send_response(response)
```

### 为什么异步是必要的？

1. **LLM 调用慢**：一次调用可能需要几秒到几十秒
2. **多通道并发**：Telegram、Discord 可能同时有消息
3. **资源效率**：一个进程服务所有用户

### asyncio 的选择

Python 3.7+ 的 asyncio 已经足够成熟：

```python
import asyncio

async def main():
    # 并发处理多个任务
    await asyncio.gather(
        handle_telegram(),
        handle_discord(),
        run_agent_loop(),
    )

asyncio.run(main())
```

---

## 为什么选择 Python？

### 优点

1. **LLM 生态**：LiteLLM、LangChain 都以 Python 为主
2. **异步支持**：asyncio 足够成熟
3. **开发效率**：代码简洁，易于修改
4. **社区活跃**：遇到问题容易找到解决方案

### 缺点（及应对）

| 缺点 | 应对 |
|------|------|
| 性能不如 Go/Rust | AI 助手的瓶颈在网络 I/O，不在 CPU |
| 类型不安全 | 使用 Pydantic + type hints |
| 部署需要 Python 环境 | 可以用 Docker |

---

## 核心循环的实现

### 最简版本

```python
async def run(self):
    """运行代理循环"""
    self._running = True

    while self._running:
        # 从消息总线获取消息
        msg = await self.bus.consume_inbound()

        # 处理消息
        response = await self._process_message(msg)

        # 发送响应
        await self.bus.publish_outbound(response)
```

**源码位置**：`nanobot/agent/loop.py:259-276`

### 加入超时和优雅退出

```python
async def run(self):
    self._running = True

    while self._running:
        try:
            # 带超时的等待，允许定期检查 _running 状态
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )
        except asyncio.TimeoutError:
            continue  # 超时后继续循环，检查 _running

        if msg.content.strip().lower() == "/stop":
            await self._handle_stop(msg)
        else:
            # 创建任务处理消息，不阻塞主循环
            task = asyncio.create_task(self._dispatch(msg))
            self._active_tasks.setdefault(msg.session_key, []).append(task)
```

**源码位置**：`nanobot/agent/loop.py:259-276`

---

## 消息处理的核心逻辑

### 单轮对话

```python
async def _process_message(self, msg: InboundMessage) -> OutboundMessage:
    # 1. 获取会话
    session = self.sessions.get_or_create(msg.session_key)

    # 2. 获取历史记录
    history = session.get_history(max_messages=self.memory_window)

    # 3. 构建消息列表
    messages = self.context.build_messages(
        history=history,
        current_message=msg.content,
    )

    # 4. 调用 LLM
    response = await self.provider.chat(
        messages=messages,
        tools=self.tools.get_definitions(),
    )

    # 5. 保存历史
    session.add_message("user", msg.content)
    session.add_message("assistant", response.content)
    self.sessions.save(session)

    # 6. 返回响应
    return OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=response.content,
    )
```

### 多轮工具调用

```python
async def _run_agent_loop(self, messages, on_progress=None):
    """代理循环：可能包含多轮工具调用"""
    iteration = 0
    final_content = None

    while iteration < self.max_iterations:
        iteration += 1

        # 调用 LLM
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),
        )

        if response.has_tool_calls:
            # 有工具调用：执行工具，继续循环
            for tool_call in response.tool_calls:
                result = await self.tools.execute(
                    tool_call.name,
                    tool_call.arguments
                )
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result,
                })
        else:
            # 无工具调用：返回最终结果
            final_content = response.content
            break

    return final_content
```

**源码位置**：`nanobot/agent/loop.py:180-257`

---

## 最小可行的组成部分

```
┌─────────────────────────────────────────────────────────────┐
│                     最小可行 AI 助手                         │
├─────────────────────────────────────────────────────────────┤
│  必需组件                                                   │
│  ├── MessageBus      # 消息传递                             │
│  ├── AgentLoop       # 核心处理循环                         │
│  ├── LLMProvider     # LLM 调用（至少一个）                  │
│  └── ContextBuilder  # 系统提示词构建                       │
├─────────────────────────────────────────────────────────────┤
│  可选组件                                                   │
│  ├── Channel         # 具体平台（CLI 内置）                  │
│  ├── SessionManager  # 会话持久化                           │
│  ├── MemoryStore     # 长期记忆                             │
│  ├── ToolRegistry    # 工具系统                             │
│  └── SkillsLoader    # 技能加载                             │
└─────────────────────────────────────────────────────────────┘
```

### CLI 模式：最简启动

```bash
# 最简单的使用方式
nanobot agent -m "Hello!"
```

内部流程：

```
1. 解析命令行参数
2. 加载配置
3. 创建 LiteLLMProvider
4. 创建 AgentLoop
5. 调用 agent.process_direct("Hello!")
6. 打印响应
```

**源码位置**：`nanobot/cli/commands.py`

### Gateway 模式：多通道

```bash
# 启动多通道网关
nanobot gateway
```

内部流程：

```
1. 加载配置
2. 创建 MessageBus
3. 创建 AgentLoop
4. 创建 ChannelManager
5. 启动所有启用的 Channel
6. 运行 agent.run() 主循环
```

---

## 核心循环的演进

### 阶段 1：单次请求-响应

```python
response = await llm.chat(message)
```

这是最简单的形式，但缺少上下文。

### 阶段 2：加入历史记录

```python
messages = [
    {"role": "system", "content": "..."},
    *history,  # 之前的对话
    {"role": "user", "content": message},
]
response = await llm.chat(messages)
```

### 阶段 3：加入工具调用

```python
while not done:
    response = await llm.chat(messages, tools=tools)
    if response.has_tool_calls:
        for tc in response.tool_calls:
            result = await execute_tool(tc)
            messages.append({"role": "tool", "content": result})
    else:
        done = True
```

### 阶段 4：加入记忆压缩

```python
if len(history) > threshold:
    await consolidate_memory(session)
    history = history[-keep_count:]
```

---

## 代码行数统计

```bash
# 核心代理代码
bash core_agent_lines.sh

# 输出示例
# nanobot/agent/loop.py:      498 行
# nanobot/agent/context.py:   165 行
# nanobot/agent/memory.py:    151 行
# nanobot/bus/queue.py:        45 行
# nanobot/channels/base.py:   132 行
# nanobot/providers/registry.py: 463 行
# nanobot/session/manager.py: 213 行
# ...
# 总计: ~4000 行
```

---

## 小结

最小可行的 AI 助手只需要：

1. **消息接收**：MessageBus
2. **核心处理**：AgentLoop
3. **LLM 调用**：Provider
4. **上下文构建**：ContextBuilder

其他一切都是扩展：
- Session 是扩展（支持持久化）
- Memory 是扩展（支持长期记忆）
- Tools 是扩展（支持与外界交互）
- Skills 是扩展（支持动态能力）
- Channels 是扩展（支持多平台）

**这就是 nanobot 的核心理念**：核心极简，按需扩展。

---

下一章：[05-ABSTRACTIONS.md](./05-ABSTRACTIONS.md) — 哪些概念是稳定的，哪些是变化的？
