# 记忆设计：双层记忆系统

> 核心问题：如何让 AI 拥有长期记忆？

## 问题

LLM 是无状态的。每次调用后，它就"忘记"了之前的对话。

如何让 AI 助手：
1. 记住用户的偏好？
2. 记住之前讨论过的重要信息？
3. 在对话无限增长时保持性能？

---

## 双层记忆设计

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory System                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Session                           │   │
│  │  短期记忆：JSONL 格式的完整对话历史                   │   │
│  │  - 精确的对话记录                                    │   │
│  │  - 追加写入，不修改历史                              │   │
│  │  - 有限窗口（最近 N 条）                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          │ 超过阈值时压缩                   │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   MEMORY.md                          │   │
│  │  长期记忆：结构化的事实存储                          │   │
│  │  - 用户偏好                                          │   │
│  │  - 重要决策                                          │   │
│  │  - 持久化知识                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                          +                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   HISTORY.md                         │   │
│  │  历史日志：可 grep 搜索的时间线                      │   │
│  │  - [YYYY-MM-DD HH:MM] 事件摘要                      │   │
│  │  - 粗粒度的活动记录                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Session：短期记忆

### 为什么用 JSONL？

```jsonl
{"_type": "metadata", "key": "telegram:123", "created_at": "2024-01-15T10:30:00"}
{"role": "user", "content": "你好", "timestamp": "2024-01-15T10:30:05"}
{"role": "assistant", "content": "你好！有什么可以帮你的？", "timestamp": "2024-01-15T10:30:10"}
{"role": "user", "content": "帮我查天气", "timestamp": "2024-01-15T10:30:15", "tools_used": ["web_search"]}
```

**选择 JSONL 的理由**：

| 特性 | JSONL | SQLite | Pickle |
|------|-------|--------|--------|
| 追加写入 | ✅ O(1) | ❌ 需要 INSERT | ❌ 需要重写 |
| LLM 缓存友好 | ✅ 文件不变则缓存有效 | ❌ 数据库变化 | ❌ 二进制 |
| 可读性 | ✅ 纯文本 | ❌ 需要工具 | ❌ 二进制 |
| 调试友好 | ✅ 可以直接看 | ⚠️ 需要 SQL | ❌ 需要 Python |

**源码位置**：`nanobot/session/manager.py:16-70`

### Session 数据结构

```python
@dataclass
class Session:
    key: str  # channel:chat_id
    messages: list[dict[str, Any]] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    metadata: dict[str, Any] = field(default_factory=dict)
    last_consolidated: int = 0  # 上次压缩时的消息数
```

### 历史窗口

```python
def get_history(self, max_messages: int = 500) -> list[dict]:
    """返回未压缩的消息，限制数量"""
    unconsolidated = self.messages[self.last_consolidated:]
    sliced = unconsolidated[-max_messages:]

    # 丢弃开头的非用户消息，避免孤立的 tool_result
    for i, m in enumerate(sliced):
        if m.get("role") == "user":
            sliced = sliced[i:]
            break

    return sliced
```

**源码位置**：`nanobot/session/manager.py:45-63`

---

## MEMORY.md：长期记忆

### 内容示例

```markdown
# Long-term Memory

## User Preferences
- Language: Chinese (Simplified)
- Timezone: Asia/Shanghai
- Code style: prefers type hints and docstrings

## Projects
### nanobot
- Location: /home/user/playground/nanobot
- Status: Active development
- Key decisions: Using LiteLLM for multi-provider support

## Important Dates
- 2024-01-10: User started the nanobot project
- 2024-01-15: First successful multi-channel test
```

### 读取和写入

```python
class MemoryStore:
    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")
        self.memory_file = self.memory_dir / "MEMORY.md"
        self.history_file = self.memory_dir / "HISTORY.md"

    def read_long_term(self) -> str:
        if self.memory_file.exists():
            return self.memory_file.read_text(encoding="utf-8")
        return ""

    def write_long_term(self, content: str) -> None:
        self.memory_file.write_text(content, encoding="utf-8")

    def get_memory_context(self) -> str:
        long_term = self.read_long_term()
        return f"## Long-term Memory\n{long_term}" if long_term else ""
```

**源码位置**：`nanobot/agent/memory.py:45-67`

---

## HISTORY.md：历史日志

### 内容示例

```markdown
[2024-01-15 10:30] User asked about weather in Beijing. Discussed the weather API integration options.

[2024-01-15 14:20] Helped debug a WebSocket connection issue in the Telegram channel. Root cause was a missing dependency.

[2024-01-16 09:00] User requested adding Discord channel support. Discussed architecture implications.
```

### 追加写入

```python
def append_history(self, entry: str) -> None:
    """追加历史条目"""
    with open(self.history_file, "a", encoding="utf-8") as f:
        f.write(entry.rstrip() + "\n\n")
```

**源码位置**：`nanobot/agent/memory.py:61-63`

### 为什么需要 HISTORY.md？

1. **可搜索**：用户可以用 `grep` 搜索历史
2. **可审计**：记录了所有重要交互
3. **轻量级**：不需要数据库

---

## 记忆压缩（Consolidation）

### 触发条件

```python
# nanobot/agent/loop.py
unconsolidated = len(session.messages) - session.last_consolidated
if unconsolidated >= self.memory_window and session.key not in self._consolidating:
    # 触发异步压缩
    asyncio.create_task(self._consolidate_memory(session))
```

### 压缩流程

```
┌─────────────────────────────────────────────────────────────┐
│                  Memory Consolidation                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 提取待压缩消息                                          │
│     messages[last_consolidated:-keep_count]                 │
│                                                             │
│  2. 构建 LLM 提示词                                         │
│     - 当前 MEMORY.md 内容                                   │
│     - 待压缩的对话记录                                      │
│                                                             │
│  3. LLM 调用（使用 save_memory 工具）                       │
│     - history_entry: 写入 HISTORY.md 的摘要                 │
│     - memory_update: 更新后的 MEMORY.md                     │
│                                                             │
│  4. 写入文件                                                │
│     - append_history(history_entry)                         │
│     - write_long_term(memory_update)                        │
│                                                             │
│  5. 更新 last_consolidated                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 压缩代码

```python
async def consolidate(
    self,
    session: Session,
    provider: LLMProvider,
    model: str,
    *,
    archive_all: bool = False,
    memory_window: int = 50,
) -> bool:
    """通过 LLM 工具调用压缩记忆"""

    # 确定要压缩的消息
    keep_count = 0 if archive_all else memory_window // 2
    old_messages = session.messages[session.last_consolidated:-keep_count]

    if not old_messages:
        return True

    # 格式化消息
    lines = []
    for m in old_messages:
        if not m.get("content"):
            continue
        tools = f" [tools: {', '.join(m['tools_used'])}]" if m.get("tools_used") else ""
        lines.append(f"[{m.get('timestamp', '?')[:16]}] {m['role'].upper()}{tools}: {m['content']}")

    # 构建 LLM 提示词
    current_memory = self.read_long_term()
    prompt = f"""Process this conversation and call the save_memory tool.

## Current Long-term Memory
{current_memory or "(empty)"}

## Conversation to Process
{chr(10).join(lines)}"""

    # 调用 LLM
    response = await provider.chat(
        messages=[
            {"role": "system", "content": "You are a memory consolidation agent."},
            {"role": "user", "content": prompt},
        ],
        tools=_SAVE_MEMORY_TOOL,  # 定义 save_memory 工具
        model=model,
    )

    # 处理结果
    if response.has_tool_calls:
        args = response.tool_calls[0].arguments
        if entry := args.get("history_entry"):
            self.append_history(entry)
        if update := args.get("memory_update"):
            if update != current_memory:
                self.write_long_term(update)
        session.last_consolidated = 0 if archive_all else len(session.messages) - keep_count
        return True

    return False
```

**源码位置**：`nanobot/agent/memory.py:69-150`

### save_memory 工具定义

```python
_SAVE_MEMORY_TOOL = [
    {
        "type": "function",
        "function": {
            "name": "save_memory",
            "description": "Save the memory consolidation result.",
            "parameters": {
                "type": "object",
                "properties": {
                    "history_entry": {
                        "type": "string",
                        "description": "A paragraph (2-5 sentences) summarizing key events. "
                        "Start with [YYYY-MM-DD HH:MM].",
                    },
                    "memory_update": {
                        "type": "string",
                        "description": "Full updated long-term memory as markdown.",
                    },
                },
                "required": ["history_entry", "memory_update"],
            },
        },
    }
]
```

**源码位置**：`nanobot/agent/memory.py:18-42`

---

## /new 命令：归档所有记忆

当用户发送 `/new` 时：

```python
if cmd == "/new":
    # 归档所有消息
    success = await self._consolidate_memory(session, archive_all=True)
    if success:
        session.clear()
        self.sessions.save(session)
        return OutboundMessage(content="New session started.")
    else:
        return OutboundMessage(content="Memory archival failed.")
```

**效果**：
1. 所有历史消息被压缩到 MEMORY.md 和 HISTORY.md
2. Session 被清空
3. 新的对话开始

---

## 为什么用 LLM 压缩？

### 替代方案

| 方案 | 优点 | 缺点 |
|------|------|------|
| 简单截断 | 快 | 丢失信息 |
| 关键词提取 | 快 | 语义丢失 |
| 摘要模型 | 中等 | 需要额外模型 |
| LLM 压缩 | 保留语义 | 慢、有成本 |

### 选择 LLM 的理由

1. **语义理解**：LLM 能理解什么重要
2. **结构化输出**：通过工具调用得到结构化结果
3. **一致性**：使用同一个模型，风格一致

---

## 记忆流程图

```
用户消息
    │
    ▼
┌─────────────┐
│   Session   │ 保存消息（追加）
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│        检查是否需要压缩              │
│  len(messages) - last_consolidated  │
│           >= memory_window?         │
└──────────────┬──────────────────────┘
               │
       ┌───────┴───────┐
       │               │
       ▼ No            ▼ Yes
   继续处理         ┌─────────────┐
                   │ 异步压缩    │
                   │ (LLM 调用)  │
                   └──────┬──────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
       ┌─────────────┐         ┌─────────────┐
       │ HISTORY.md  │         │  MEMORY.md  │
       │ (追加摘要)  │         │ (更新事实)  │
       └─────────────┘         └─────────────┘
              │                       │
              └───────────┬───────────┘
                          ▼
                   更新 last_consolidated
```

---

## 源码索引

| 功能 | 文件位置 |
|------|---------|
| Session 类 | `nanobot/session/manager.py:16-70` |
| SessionManager | `nanobot/session/manager.py:72-213` |
| MemoryStore | `nanobot/agent/memory.py:45-150` |
| save_memory 工具 | `nanobot/agent/memory.py:18-42` |
| 压缩触发 | `nanobot/agent/loop.py:396-412` |
| /new 命令 | `nanobot/agent/loop.py:364-391` |

---

下一章：[09-TOOL-SYSTEM.md](./09-TOOL-SYSTEM.md) — 如何让 LLM 拥有与外界交互的能力？
