# Agent 核心模块详解

本文档详细解释 nanobot 的核心代理系统，包括所有主要组件的设计理念、架构和运行机制。阅读本文档后，您将对 nanobot 如何处理用户消息、执行工具调用、管理记忆和技能有完整的理解。

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [AgentLoop - 代理处理引擎](#2-agentloop---代理处理引擎)
3. [ContextBuilder - 系统提示词构建器](#3-contextbuilder---系统提示词构建器)
4. [MemoryStore - 持久化记忆管理](#4-memorystore---持久化记忆管理)
5. [SkillsLoader - 技能加载器](#5-skillsloader---技能加载器)
6. [SubagentManager - 后台任务执行](#6-subagentmanager---后台任务执行)
7. [Tool 工具系统](#7-tool-工具系统)
8. [Session 会话管理](#8-session-会话管理)
9. [MessageBus 消息总线](#9-messagebus-消息总线)
10. [消息处理完整流程](#10-消息处理完整流程)

---

## 1. 整体架构概览

### 1.1 核心组件关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户消息输入                                      │
│                  (来自 Telegram/Discord/Slack/CLI 等渠道)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MessageBus (消息总线)                                │
│                    异步队列，解耦渠道与代理核心                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AgentLoop.process_message()                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  1. ContextBuilder.build_messages()                                │    │
│  │     - 组装系统提示词 + 历史记录 + 运行时上下文                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  2. _run_agent_loop()                                              │    │
│  │     - 调用 LLM API                                                 │    │
│  │     - 执行工具调用 (ToolRegistry)                                   │    │
│  │     - 迭代直到完成                                                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    │                                         │
│                                    ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  3. 记忆整合 (MemoryStore.consolidate)                             │    │
│  │     - 将旧消息摘要写入 MEMORY.md / HISTORY.md                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OutboundMessage                                      │
│                   返回给用户的消息 (通过渠道发送)                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心设计理念

nanobot 的代理系统遵循以下设计原则：

1. **简洁优先**: 核心代码约 4000 行，易于理解和修改
2. **异步优先**: 基于 asyncio，支持高并发消息处理
3. **工具驱动**: LLM 通过工具调用与世界交互
4. **渐进加载**: 技能先加载摘要，按需读取完整内容
5. **记忆分层**: 短期会话记忆 + 长期记忆 + 历史日志
6. **解耦设计**: MessageBus 解耦渠道与核心逻辑

---

## 2. AgentLoop - 代理处理引擎

**文件位置**: `nanobot/agent/loop.py`

### 2.1 职责概述

`AgentLoop` 是整个代理系统的核心引擎，负责协调消息处理的完整生命周期：

1. **消息接收**: 从 MessageBus 获取入站消息
2. **上下文构建**: 委托给 ContextBuilder 组装提示词
3. **LLM 调用**: 与 LLM 提供商交互
4. **工具执行**: 通过 ToolRegistry 执行工具调用
5. **会话管理**: 与 SessionManager 协作管理对话
6. **记忆整合**: 触发 MemoryStore 进行记忆压缩
7. **子代理管理**: 通过 SubagentManager 管理后台任务

### 2.2 核心类结构

```python
class AgentLoop:
    """Central processing engine for agent messages."""

    def __init__(
        self,
        provider: LLMProvider,           # LLM 提供商
        bus: MessageBus,                  # 消息总线
        sessions: SessionManager,         # 会话管理器
        workspace: Path,                  # 工作区路径
        model: str,                       # 使用的模型
        # ... 其他配置
    ):
        self.provider = provider
        self.bus = bus
        self.sessions = sessions
        self.tools = ToolRegistry()       # 工具注册表
        self.context = ContextBuilder(workspace)  # 上下文构建器
        self.subagents = SubagentManager(...)    # 子代理管理器
        # ...

    async def run(self) -> None:
        """主循环：持续从消息总线消费消息并处理"""
        self._running = True
        while self._running:
            msg = await self.bus.consume_inbound()  # 阻塞等待消息
            if msg.content.strip().lower() == "/stop":
                await self._handle_stop(msg)
            else:
                asyncio.create_task(self._dispatch(msg))  # 异步处理
```

### 2.3 消息处理流程

```python
async def _process_message(self, msg: InboundMessage) -> OutboundMessage | None:
    """处理单条入站消息"""

    # 1. 解析会话键
    key = f"{msg.channel}:{msg.chat_id}"
    session = self.sessions.get_or_create(key)

    # 2. 处理命令
    if msg.content.strip().lower() == "/new":
        # 开启新会话，归档旧记忆
        return await self._handle_new_session(session, msg)

    if msg.content.strip().lower() == "/help":
        # 显示帮助
        return OutboundMessage(...)

    # 3. 检查是否需要记忆整合
    unconsolidated = len(session.messages) - session.last_consolidated
    if unconsolidated >= self.memory_window:
        # 异步触发记忆整合
        asyncio.create_task(self._consolidate_memory(session))

    # 4. 构建消息列表
    history = session.get_history(max_messages=self.memory_window)
    messages = self.context.build_messages(
        history=history,
        current_message=msg.content,
        channel=msg.channel,
        chat_id=msg.chat_id,
    )

    # 5. 运行代理循环
    final_content, tools_used, all_msgs = await self._run_agent_loop(messages)

    # 6. 保存会话
    self._save_turn(session, all_msgs, 1 + len(history))
    self.sessions.save(session)

    # 7. 返回响应
    return OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=final_content,
    )
```

### 2.4 代理循环核心逻辑

`_run_agent_loop()` 方法实现了 LLM 与工具调用的迭代：

```python
async def _run_agent_loop(
    self,
    initial_messages: list[dict],
    on_progress: Callable | None = None,  # 进度回调（流式输出）
) -> tuple[str | None, list[str], list[dict]]:
    """运行代理迭代循环"""

    messages = initial_messages
    iteration = 0

    while iteration < self.max_iterations:
        iteration += 1

        # 1. 调用 LLM
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),  # 工具定义
            model=self.model,
            temperature=self.temperature,
            # ...
        )

        # 2. 如果有工具调用
        if response.has_tool_calls:
            # 添加助手消息（含工具调用）
            messages = self.context.add_assistant_message(
                messages, response.content, tool_call_dicts
            )

            # 执行每个工具调用
            for tool_call in response.tool_calls:
                result = await self.tools.execute(
                    tool_call.name,
                    tool_call.arguments
                )
                # 添加工具结果到消息列表
                messages = self.context.add_tool_result(
                    messages, tool_call.id, tool_call.name, result
                )

            # 继续循环，让 LLM 基于工具结果做下一步决策

        # 3. 没有工具调用（完成）
        else:
            messages = self.context.add_assistant_message(messages, response.content)
            return response.content, tools_used, messages

    # 达到最大迭代次数
    return "我达到了最大工具调用次数...", tools_used, messages
```

### 2.5 关键设计点

#### 2.5.1 最大迭代次数保护

```python
self.max_iterations = 50  # 防止无限循环
```

这确保了：
- 代理不会陷入无限工具调用
- 避免资源耗尽
- 用户会收到明确的超时消息

#### 2.5.2 思考块 (Thinking Blocks) 处理

LLM 的推理过程可能包含 `<think>` 块，这些需要过滤：

```python
@staticmethod
def _strip_think(text: str) -> str | None:
    """移除 AI 推理过程中的思考块"""
    if not text:
        return None
    return re.sub(r"<think>[\s\S]*?
</think>", "", text).strip() or None
```

#### 2.5.3 进度回调 (Progress Callback)

支持流式输出进度：

```python
async def _bus_progress(content: str, *, tool_hint: bool = False) -> None:
    """通过消息总线发送进度更新"""
    meta = dict(msg.metadata or {})
    meta["_progress"] = True
    meta["_tool_hint"] = tool_hint
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel,
        chat_id=msg.chat_id,
        content=content,
        metadata=meta,
    ))
```

### 2.6 特殊消息处理

#### 2.6.1 系统消息

系统消息（如来自 cron 或子代理）有特殊处理：

```python
if msg.channel == "system":
    # 解析来源: "channel:chat_id"
    channel, chat_id = msg.chat_id.split(":", 1)
    # 创建独立会话处理
    session = self.sessions.get_or_create(f"{channel}:{chat_id}")
    # 处理完成后返回到原始会话
```

#### 2.6.2 命令处理

- `/new`: 开启新会话，归档旧记忆
- `/help`: 显示帮助信息
- `/stop`: 停止当前任务和子代理

---

## 3. ContextBuilder - 系统提示词构建器

**文件位置**: `nanobot/agent/context.py`

### 3.1 职责概述

`ContextBuilder` 负责组装发送给 LLM 的完整上下文消息，包括：

1. **系统提示词**: 从模板文件构建
2. **身份信息**: 运行时环境和工作区路径
3. **记忆上下文**: 从 MEMORY.md 读取长期记忆
4. **技能摘要**: 加载所有可用技能的概要
5. **会话历史**: 对话历史消息
6. **运行时上下文**: 当前时间、渠道、聊天 ID

### 3.2 引导文件 (Bootstrap Files)

系统提示词从以下模板文件按顺序组装：

```python
BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md", "IDENTITY.md"]
```

这些文件位于工作区 `~/.nanobot/workspace/`，在 nanobot 启动时同步。

### 3.3 系统提示词构建流程

```python
def build_system_prompt(self, skill_names: list[str] | None = None) -> str:
    """构建完整的系统提示词"""

    parts = []

    # 1. 核心身份
    parts.append(self._get_identity())

    # 2. 引导文件内容
    bootstrap = self._load_bootstrap_files()
    if bootstrap:
        parts.append(bootstrap)

    # 3. 长期记忆
    memory = self.memory.get_memory_context()
    if memory:
        parts.append(f"# Memory\n\n{memory}")

    # 4. 自动加载的技能 (always=true)
    always_skills = self.skills.get_always_skills()
    if always_skills:
        always_content = self.skills.load_skills_for_context(always_skills)
        if always_content:
            parts.append(f"# Active Skills\n\n{always_content}")

    # 5. 技能摘要 (供按需加载)
    skills_summary = self.skills.build_skills_summary()
    if skills_summary:
        parts.append(f"""# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.

{skills_summary}""")

    # 用分隔线连接各部分
    return "\n\n---\n\n".join(parts)
```

### 3.4 身份部分示例

```python
def _get_identity(self) -> str:
    """获取核心身份部分"""

    workspace_path = str(self.workspace.expanduser().resolve())
    system = platform.system()
    runtime = f"{'macOS' if system == 'Darwin' else system} {platform.machine()}, Python {platform.python_version()}"

    return f"""# nanobot 🐈

You are nanobot, a helpful AI assistant.

## Runtime
{runtime}

## Workspace
Your workspace is at: {workspace_path}
- Long-term memory: {workspace_path}/memory/MEMORY.md (write important facts here)
- History log: {workspace_path}/memory/HISTORY.md (grep-searchable)
- Custom skills: {workspace_path}/skills/{{skill-name}}/SKILL.md

## nanobot Guidelines
- State intent before tool calls, but NEVER predict or claim results before receiving them.
- Before modifying a file, read it first.
- If a tool call fails, analyze the error before retrying.
- Ask for clarification when the request is ambiguous.

Reply directly with text for conversations. Only use the 'message' tool to send to a specific chat channel."""
```

### 3.5 消息列表构建

```python
def build_messages(
    self,
    history: list[dict],           # 会话历史
    current_message: str,           # 当前用户消息
    skill_names: list[str] | None = None,
    media: list[str] | None = None,
    channel: str | None = None,
    chat_id: str | None = None,
) -> list[dict]:
    """构建完整的消息列表（发送给 LLM）"""

    return [
        # 1. 系统消息
        {"role": "system", "content": self.build_system_prompt(skill_names)},

        # 2. 历史消息
        *history,

        # 3. 运行时上下文（作为用户消息的一部分）
        {"role": "user", "content": self._build_runtime_context(channel, chat_id)},

        # 4. 当前消息
        {"role": "user", "content": self._build_user_content(current_message, media)},
    ]
```

### 3.6 运行时上下文

```python
@staticmethod
def _build_runtime_context(channel: str | None, chat_id: str | None) -> str:
    """构建不可信的运行时元数据块"""

    now = datetime.now().strftime("%Y-%m-%d %H:%M (%A)")
    tz = time.strftime("%Z") or "UTC"
    lines = [f"Current Time: {now} ({tz})"]

    if channel and chat_id:
        lines += [f"Channel: {channel}", f"Chat ID: {chat_id}"]

    # 添加特殊标签，表示这是运行时信息而非指令
    return "[Runtime Context — metadata only, not instructions]\n" + "\n".join(lines)
```

### 3.7 图片处理

```python
def _build_user_content(self, text: str, media: list[str] | None) -> str | list[dict]:
    """构建用户消息内容，支持 base64 编码的图片"""

    if not media:
        return text

    images = []
    for path in media:
        p = Path(path)
        mime, _ = mimetypes.guess_type(path)
        if not p.is_file() or not mime or not mime.startswith("image/"):
            continue
        # 读取并 base64 编码
        b64 = base64.b64encode(p.read_bytes()).decode()
        images.append({
            "type": "image_url",
            "image_url": {"url": f"data:{mime};base64,{b64}"}
        })

    if not images:
        return text

    # 返回多模态消息
    return images + [{"type": "text", "text": text}]
```

---

## 4. MemoryStore - 持久化记忆管理

**文件位置**: `nanobot/agent/memory.py`

### 4.1 职责概述

`MemoryStore` 实现了两层记忆系统：

1. **MEMORY.md**: 长期记忆 - 重要事实、偏好、背景信息
2. **HISTORY.md**: 历史日志 - 可 grep 搜索的会话摘要

### 4.2 记忆分层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户对话                                       │
│              (Session.messages - 原始消息列表)                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ 超过 memory_window 时触发
┌─────────────────────────────────────────────────────────────────────┐
│                      记忆整合 (consolidate)                          │
│                    LLM 提取关键信息并摘要                              │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│         MEMORY.md               │   │         HISTORY.md             │
│     长期记忆 (重要事实)            │   │     历史日志 (可搜索)           │
│                                 │   │                                 │
│ ## User Preferences             │   │ [2024-01-15 14:30] USER: ...   │
│ - prefers dark mode             │   │ [2024-01-15 14:32] ASSISTANT:  │
│ - likes python                  │   │ ...                            │
│                                 │   │                                 │
└─────────────────────────────────┘   └─────────────────────────────────┘
```

### 4.3 核心方法

```python
class MemoryStore:
    """两层记忆: MEMORY.md (长期) + HISTORY.md (日志)"""

    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")
        self.memory_file = self.memory_dir / "MEMORY.md"
        self.history_file = self.memory_dir / "HISTORY.md"

    def read_long_term(self) -> str:
        """读取长期记忆"""
        if self.memory_file.exists():
            return self.memory_file.read_text(encoding="utf-8")
        return ""

    def write_long_term(self, content: str) -> None:
        """写入长期记忆"""
        self.memory_file.write_text(content, encoding="utf-8")

    def append_history(self, entry: str) -> None:
        """追加历史日志"""
        with open(self.history_file, "a", encoding="utf-8") as f:
            f.write(entry.rstrip() + "\n\n")

    def get_memory_context(self) -> str:
        """获取记忆上下文（用于系统提示词）"""
        long_term = self.read_long_term()
        return f"## Long-term Memory\n{long_term}" if long_term else ""
```

### 4.4 记忆整合流程

```python
async def consolidate(
    self,
    session: Session,
    provider: LLMProvider,
    model: str,
    *,
    archive_all: bool = False,      # 是否归档所有消息
    memory_window: int = 50,        # 保留在会话中的消息数
) -> bool:
    """将旧消息整合到长期记忆"""

    # 1. 确定需要整合的消息
    if archive_all:
        old_messages = session.messages  # 全部归档（如 /new 命令）
        keep_count = 0
    else:
        keep_count = memory_window // 2  # 保留一半
        # 取已整合之后、保留之前的消息
        old_messages = session.messages[session.last_consolidated:-keep_count]

    if not old_messages:
        return True

    # 2. 格式化消息为文本
    lines = []
    for m in old_messages:
        if not m.get("content"):
            continue
        tools = f" [tools: {', '.join(m['tools_used'])}]" if m.get("tools_used") else ""
        lines.append(f"[{m.get('timestamp', '?')[:16]}] {m['role'].upper()}{tools}: {m['content']}")

    # 3. 调用 LLM 进行整合
    current_memory = self.read_long_term()
    prompt = f"""Process this conversation and call the save_memory tool with your consolidation.

## Current Long-term Memory
{current_memory or "(empty)"}

## Conversation to Process
{chr(10).join(lines)}"""

    # 4. LLM 应该调用 save_memory 工具
    response = await provider.chat(
        messages=[
            {"role": "system", "content": "You are a memory consolidation agent."},
            {"role": "user", "content": prompt},
        ],
        tools=_SAVE_MEMORY_TOOL,  # 定义了 save_memory 工具
        model=model,
    )

    # 5. 处理工具调用结果
    if response.has_tool_calls:
        args = response.tool_calls[0].arguments

        # 写入历史日志
        if entry := args.get("history_entry"):
            self.append_history(entry)

        # 更新长期记忆
        if update := args.get("memory_update"):
            if update != current_memory:  # 只有变化时才写
                self.write_long_term(update)

    # 6. 更新会话的已整合位置
    session.last_consolidated = 0 if archive_all else len(session.messages) - keep_count
```

### 4.5 保存记忆工具定义

```python
_SAVE_MEMORY_TOOL = [
    {
        "type": "function",
        "function": {
            "name": "save_memory",
            "description": "Save the memory consolidation result to persistent storage.",
            "parameters": {
                "type": "object",
                "properties": {
                    "history_entry": {
                        "type": "string",
                        "description": "A paragraph summarizing key events. Start with [YYYY-MM-DD HH:MM].",
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

### 4.6 整合触发条件

在 `AgentLoop._process_message()` 中：

```python
unconsolidated = len(session.messages) - session.last_consolidated
if unconsolidated >= self.memory_window:  # 默认 50 条消息
    # 异步触发整合（不阻塞当前请求）
    asyncio.create_task(self._consolidate_memory(session))
```

---

## 5. SkillsLoader - 技能加载器

**文件位置**: `nanobot/agent/skills.py`

### 5.1 职责概述

`SkillsLoader` 负责管理代理的技能（Skills）：

1. **发现技能**: 扫描工作区和内置技能目录
2. **依赖检查**: 检查技能所需的 CLI 工具和环境变量
3. **元数据解析**: 解析 YAML 前置元数据
4. **渐进加载**: 支持 always（自动加载）和按需加载

### 5.2 技能目录结构

```
~/.nanobot/workspace/skills/          # 用户技能（优先）
    └── my-skill/
        └── SKILL.md

nanobot/skills/                       # 内置技能
    ├── commit/
    │   └── SKILL.md
    ├── git/
    │   └── SKILL.md
    └── ...
```

### 5.3 技能文件格式

```markdown
---
name: skill-name
description: "技能描述"
metadata:
  nanobot:
    emoji: 📦
    always: false              # 是否自动加载
    requires:
      bins: [git, gh]         # 需要的 CLI 工具
      env: [GITHUB_TOKEN]     # 需要的环境变量
    install:
      - id: brew
        kind: brew
        formula: gh
        bins: [gh]
        label: Install via brew
---
# 技能指令内容

这里是技能的使用说明...
```

### 5.4 核心方法

```python
class SkillsLoader:
    def __init__(self, workspace: Path, builtin_skills_dir: Path | None = None):
        self.workspace = workspace
        self.workspace_skills = workspace / "skills"
        self.builtin_skills = builtin_skills_dir or BUILTIN_SKILLS_DIR

    def list_skills(self, filter_unavailable: bool = True) -> list[dict]:
        """列出所有可用技能"""
        skills = []

        # 1. 工作区技能（优先）
        if self.workspace_skills.exists():
            for skill_dir in self.workspace_skills.iterdir():
                if skill_dir.is_dir():
                    skill_file = skill_dir / "SKILL.md"
                    if skill_file.exists():
                        skills.append({
                            "name": skill_dir.name,
                            "path": str(skill_file),
                            "source": "workspace"
                        })

        # 2. 内置技能
        if self.builtin_skills.exists():
            for skill_dir in self.builtin_skills.iterdir():
                if skill_dir.is_dir():
                    skill_file = skill_dir / "SKILL.md"
                    if skill_file.exists() and not any(s["name"] == skill_dir.name for s in skills):
                        skills.append({
                            "name": skill_dir.name,
                            "path": str(skill_file),
                            "source": "builtin"
                        })

        # 3. 按依赖过滤
        if filter_unavailable:
            return [s for s in skills if self._check_requirements(self._get_skill_meta(s["name"]))]

        return skills

    def load_skill(self, name: str) -> str | None:
        """加载指定技能的完整内容"""
        # 先检查工作区
        workspace_skill = self.workspace_skills / name / "SKILL.md"
        if workspace_skill.exists():
            return workspace_skill.read_text(encoding="utf-8")

        # 再检查内置
        builtin_skill = self.builtin_skills / name / "SKILL.md"
        if builtin_skill.exists():
            return builtin_skill.read_text(encoding="utf-8")

        return None

    def get_always_skills(self) -> list[str]:
        """获取需要自动加载的技能"""
        result = []
        for s in self.list_skills(filter_unavailable=True):
            meta = self.get_skill_metadata(s["name"]) or {}
            skill_meta = self._parse_nanobot_metadata(meta.get("metadata", ""))
            if skill_meta.get("always") or meta.get("always"):
                result.append(s["name"])
        return result
```

### 5.5 技能摘要构建

```python
def build_skills_summary(self) -> str:
    """构建技能摘要（用于系统提示词）"""

    all_skills = self.list_skills(filter_unavailable=False)
    if not all_skills:
        return ""

    # XML 格式，便于 LLM 理解
    lines = ["<skills>"]
    for s in all_skills:
        name = escape_xml(s["name"])
        path = s["path"]
        desc = escape_xml(self._get_skill_description(s["name"]))
        available = self._check_requirements(self._get_skill_meta(s["name"]))

        lines.append(f'  <skill available="{str(available).lower()}">')
        lines.append(f"    <name>{name}</name>")
        lines.append(f"    <description>{desc}</description>")
        lines.append(f"    <location>{path}</location>")

        # 显示缺失的依赖
        if not available:
            missing = self._get_missing_requirements(self._get_skill_meta(s["name"]))
            if missing:
                lines.append(f"    <requires>{escape_xml(missing)}</requires>")

        lines.append("  </skill>")
    lines.append("</skills>")

    return "\n".join(lines)
```

### 5.6 按需加载机制

技能采用渐进加载策略：

1. **启动时**: 加载所有 always=true 的技能完整内容
2. **系统提示词**: 包含所有技能的摘要（XML 格式）
3. **按需**: 代理需要使用某技能时，用 `read_file` 工具读取完整内容

这种设计：
- 减少系统提示词长度
- 避免加载不需要的技能指令
- 让代理自主决定何时需要技能

---

## 6. SubagentManager - 后台任务执行

**文件位置**: `nanobot/agent/subagent.py`

### 6.1 职责概述

`SubagentManager` 负责在后台启动独立代理执行任务：

1. **任务Spawn**: 创建后台子代理处理长时间任务
2. **工具隔离**: 子代理只能使用受限工具集
3. **结果通知**: 任务完成后通过 MessageBus 通知主代理

### 6.2 为什么要子代理？

主代理循环是同步的——一条消息处理完才能处理下一条。如果有耗时的后台任务：
- 用户需要等待任务完成才能继续对话
- 可能超时

子代理允许：
- 后台并行执行任务
- 用户可以继续与主代理交互
- 任务完成后主动通知用户

### 6.3 Spawn 工具

```python
class SpawnTool(Tool):
    """用于启动子代理的工具"""

    async def execute(self, task: str, label: str | None = None, **kwargs) -> str:
        """启动子代理"""
        return await self._manager.spawn(
            task=task,
            label=label,
            origin_channel=self._origin_channel,
            origin_chat_id=self._origin_chat_id,
            session_key=self._session_key,
        )
```

### 6.4 子代理执行流程

```python
async def spawn(
    self,
    task: str,
    label: str | None = None,
    origin_channel: str = "cli",
    origin_chat_id: str = "direct",
    session_key: str | None = None,
) -> str:
    """Spawn 一个子代理"""

    task_id = str(uuid.uuid4())[:8]  # 生成唯一 ID

    # 创建后台任务
    bg_task = asyncio.create_task(
        self._run_subagent(task_id, task, display_label, origin)
    )

    # 跟踪任务
    self._running_tasks[task_id] = bg_task
    if session_key:
        self._session_tasks.setdefault(session_key, set()).add(task_id)

    return f"Subagent [{display_label}] started (id: {task_id}). I'll notify you when it completes."
```

### 6.5 子代理的独立循环

```python
async def _run_subagent(self, task_id: str, task: str, label: str, origin: dict) -> None:
    """执行子代理任务"""

    # 1. 构建受限工具集（不能发消息、不能 spawn）
    tools = ToolRegistry()
    allowed_dir = self.workspace if self.restrict_to_workspace else None
    tools.register(ReadFileTool(workspace=self.workspace, allowed_dir=allowed_dir))
    tools.register(WriteFileTool(workspace=self.workspace, allowed_dir=allowed_dir))
    tools.register(EditFileTool(workspace=self.workspace, allowed_dir=allowed_dir))
    tools.register(ListDirTool(workspace=self.workspace, allowed_dir=allowed_dir))
    tools.register(ExecTool(...))
    tools.register(WebSearchTool(...))
    tools.register(WebFetchTool(...))

    # 2. 构建系统提示词
    system_prompt = self._build_subagent_prompt()

    # 3. 运行代理循环（最多 15 次迭代）
    max_iterations = 15
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": task},
    ]

    while iteration < max_iterations:
        response = await self.provider.chat(
            messages=messages,
            tools=tools.get_definitions(),
            model=self.model,
            # ...
        )

        if response.has_tool_calls:
            # 执行工具...
            messages.append({"role": "assistant", "content": ..., "tool_calls": ...})
            for tool_call in response.tool_calls:
                result = await tools.execute(tool_call.name, tool_call.arguments)
                messages.append({"role": "tool", ...})
        else:
            final_result = response.content
            break

    # 4. 通知主代理结果
    await self._announce_result(task_id, label, task, final_result, origin, "ok")
```

### 6.6 结果通知机制

```python
async def _announce_result(
    self,
    task_id: str,
    label: str,
    task: str,
    result: str,
    origin: dict,
    status: str,
) -> None:
    """将结果通知给主代理"""

    # 构造通知内容
    announce_content = f"""[Subagent '{label}' completed]

Task: {task}

Result:
{result}

Summarize this naturally for the user. Keep it brief (1-2 sentences)."""

    # 作为系统消息注入到主代理
    msg = InboundMessage(
        channel="system",  # 特殊渠道
        sender_id="subagent",
        chat_id=f"{origin['channel']}:{origin['chat_id']}",
        content=announce_content,
    )

    await self.bus.publish_inbound(msg)
```

### 6.7 子代理工具限制

| 工具 | 主代理 | 子代理 |
|------|--------|--------|
| read_file | ✓ | ✓ |
| write_file | ✓ | ✓ |
| edit_file | ✓ | ✓ |
| list_dir | ✓ | ✓ |
| exec | ✓ | ✓ |
| web_search | ✓ | ✓ |
| web_fetch | ✓ | ✓ |
| cron | ✓ | ✗ |
| spawn | ✓ | ✗ |
| message | ✓ | ✗ |
| mcp | ✓ | ✗ |

这确保子代理：
- 可以执行任务
- 不能主动发消息给用户
- 不能启动更多子代理

---

## 7. Tool 工具系统

### 7.1 架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ToolRegistry                                 │
│  工具注册与执行中心                                                  │
│  - register() / unregister()                                       │
│  - get_definitions() -> 工具定义列表                                │
│  - execute(name, params) -> 执行结果                                │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│           Tool (抽象基类)         │   │      JSON Schema               │
│                                 │   │                                 │
│  + name: str                    │   │  {                              │
│  + description: str             │   │    "type": "object",           │
│  + parameters: dict             │   │    "properties": {...},        │
│  + execute(**kwargs) -> str    │   │    "required": [...]           │
│  + validate_params() -> []     │   │  }                              │
└─────────────────────────────────┘   └─────────────────────────────────┘
```

### 7.2 基础类定义

**文件**: `nanobot/agent/tools/base.py`

```python
class Tool(ABC):
    """工具抽象基类"""

    @property
    @abstractmethod
    def name(self) -> str:
        """工具名称（用于函数调用）"""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """工具描述（供 LLM 理解用途）"""
        pass

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]:
        """JSON Schema 格式的参数定义"""
        pass

    @abstractmethod
    async def execute(self, **kwargs: Any) -> str:
        """执行工具逻辑"""
        pass

    def to_schema(self) -> dict:
        """转换为 OpenAI 函数调用格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }
```

### 7.3 工具注册表

**文件**: `nanobot/agent/tools/registry.py`

```python
class ToolRegistry:
    """工具注册表"""

    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        """注册工具"""
        self._tools[tool.name] = tool

    def get_definitions(self) -> list[dict]:
        """获取所有工具定义（用于 LLM）"""
        return [tool.to_schema() for tool in self._tools.values()]

    async def execute(self, name: str, params: dict) -> str:
        """执行工具"""
        tool = self._tools.get(name)
        if not tool:
            return f"Error: Tool '{name}' not found."

        # 参数验证
        errors = tool.validate_params(params)
        if errors:
            return f"Error: Invalid parameters: " + "; ".join(errors)

        try:
            result = await tool.execute(**params)
            return result
        except Exception as e:
            return f"Error executing {name}: {str(e)}"
```

### 7.4 内置工具列表

| 工具 | 文件 | 功能 |
|------|------|------|
| exec | shell.py | 执行 Shell 命令 |
| read_file | filesystem.py | 读取文件内容 |
| write_file | filesystem.py | 写入文件内容 |
| edit_file | filesystem.py | 替换文件中文本 |
| list_dir | filesystem.py | 列出目录内容 |
| web_search | web.py | 网络搜索 (Brave API) |
| web_fetch | web.py | 获取网页内容 |
| cron | cron.py | 定时任务管理 |
| spawn | spawn.py | 启动子代理 |
| message | message.py | 发送消息给用户 |
| mcp | mcp.py | MCP 工具集成 |

### 7.5 文件系统工具

**文件**: `nanobot/agent/tools/filesystem.py`

提供基本的文件操作能力，带有安全限制：

```python
class ReadFileTool(Tool):
    """读取文件工具"""

    @property
    def name(self) -> str:
        return "read_file"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "文件路径"}
            },
            "required": ["path"],
        }

    async def execute(self, path: str, **kwargs) -> str:
        # 路径安全检查
        file_path = _resolve_path(path, self._workspace, self._allowed_dir)
        if not file_path.exists():
            return f"Error: File not found: {path}"
        return file_path.read_text(encoding="utf-8")
```

路径解析带安全检查：

```python
def _resolve_path(
    path: str,
    workspace: Path | None = None,
    allowed_dir: Path | None = None
) -> Path:
    """解析路径并检查权限"""
    p = Path(path).expanduser()
    if not p.is_absolute() and workspace:
        p = workspace / p  # 相对路径相对于工作区
    resolved = p.resolve()

    # 检查是否在允许目录内
    if allowed_dir:
        try:
            resolved.relative_to(allowed_dir.resolve())
        except ValueError:
            raise PermissionError(f"Path {path} is outside allowed directory")

    return resolved
```

### 7.6 Shell 执行工具

**文件**: `nanobot/agent/tools/shell.py`

```python
class ExecTool(Tool):
    """Shell 命令执行工具"""

    def __init__(
        self,
        timeout: int = 60,                    # 超时秒数
        working_dir: str | None = None,       # 工作目录
        deny_patterns: list[str] = [...],     # 禁止的模式
        restrict_to_workspace: bool = False,  # 是否限制在工作区
    ):
        # 危险命令黑名单
        self.deny_patterns = deny_patterns or [
            r"\brm\s+-[rf]{1,2}\b",        # rm -rf
            r"\bdel\s+/[fq]\b",            # del /f
            r"\bformat\b",                 # format
            r"\b(mkfs|diskpart)\b",        # 磁盘操作
            r"\bshutdown\b",               # 关机
            r":\(\)\s*\{.*\};\s*:",        # fork bomb
        ]

    async def execute(self, command: str, working_dir: str | None = None, **kwargs) -> str:
        # 安全检查
        guard_error = self._guard_command(command, working_dir or self.working_dir)
        if guard_error:
            return guard_error

        # 执行命令
        process = await asyncio.create_subprocess_shell(
            command,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=cwd,
        )

        # 等待结果
        try:
            stdout, stderr = await asyncio.wait_for(
                process.communicate(),
                timeout=self.timeout
            )
        except asyncio.TimeoutError:
            process.kill()
            return f"Error: Command timed out after {self.timeout} seconds"

        # 返回输出
        result = stdout.decode() if stdout else ""
        if stderr:
            result += f"\nSTDERR:\n{stderr.decode()}"
        return result or "(no output)"
```

### 7.7 Cron 工具

**文件**: `nanobot/agent/tools/cron.py`

```python
class CronTool(Tool):
    """定时任务管理工具"""

    async def execute(
        self,
        action: str,                    # add / list / remove
        message: str = "",              # 提醒消息
        every_seconds: int | None = None,   # 间隔秒数
        cron_expr: str | None = None,       # Cron 表达式
        tz: str | None = None,              # 时区
        at: str | None = None,              # 一次性执行时间
        job_id: str | None = None,          # 任务 ID（删除用）
        **kwargs
    ) -> str:
        if action == "add":
            # 添加任务
            if every_seconds:
                schedule = CronSchedule(kind="every", every_ms=every_seconds * 1000)
            elif cron_expr:
                schedule = CronSchedule(kind="cron", expr=cron_expr, tz=tz)
            elif at:
                schedule = CronSchedule(kind="at", at_ms=...)
                delete_after = True
            else:
                return "Error: 需要指定 every_seconds, cron_expr 或 at"

            job = self._cron.add_job(
                name=message[:30],
                schedule=schedule,
                message=message,
                channel=self._channel,
                to=self._chat_id,
            )
            return f"Created job '{job.name}' (id: {job.id})"

        elif action == "list":
            jobs = self._cron.list_jobs()
            return "\n".join(f"- {j.name} (id: {j.id})" for j in jobs)

        elif action == "remove":
            if self._cron.remove_job(job_id):
                return f"Removed job {job_id}"
            return f"Job {job_id} not found"
```

### 7.8 Web 工具

**文件**: `nanobot/agent/tools/web.py`

```python
class WebSearchTool(Tool):
    """网络搜索工具（使用 Brave Search API）"""

    @property
    def name(self) -> str:
        return "web_search"

    async def execute(self, query: str, count: int | None = None, **kwargs) -> str:
        if not self.api_key:
            return "Error: Brave Search API key not configured."

        # 调用 Brave Search API
        async with httpx.AsyncClient() as client:
            r = await client.get(
                "https://api.search.brave.com/res/v1/web/search",
                params={"q": query, "count": count or 5},
                headers={"X-Subscription-Token": self.api_key},
            )

        results = r.json().get("web", {}).get("results", [])
        # 格式化结果
        lines = [f"Results for: {query}\n"]
        for item in results:
            lines.append(f"{i}. {item.get('title')}\n   {item.get('url')}")
        return "\n".join(lines)


class WebFetchTool(Tool):
    """网页抓取工具（使用 Readability）"""

    @property
    def name(self) -> str:
        return "web_fetch"

    async def execute(self, url: str, extractMode: str = "markdown", **kwargs) -> str:
        # 获取网页
        async with httpx.AsyncClient(follow_redirects=True) as client:
            r = await client.get(url)

        # 使用 Readability 提取正文
        from readability import Document
        doc = Document(r.text)

        if extractMode == "markdown":
            content = self._to_markdown(doc.summary())
        else:
            content = _strip_tags(doc.summary())

        return json.dumps({
            "url": url,
            "title": doc.title(),
            "content": content,
        }, ensure_ascii=False)
```

---

## 8. Session 会话管理

**文件**: `nanobot/session/manager.py`

### 8.1 核心概念

- **会话键**: `channel:chat_id`（如 `telegram:123456`）
- **消息存储**: JSONL 格式（追加写入，利于 LLM 缓存）
- **持久化**: 保存到 `~/.nanobot/workspace/sessions/`

### 8.2 Session 数据结构

```python
@dataclass
class Session:
    key: str                           # channel:chat_id
    messages: list[dict] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    metadata: dict = field(default_factory=dict)
    last_consolidated: int = 0         # 已整合的消息数量
```

### 8.3 消息获取

```python
def get_history(self, max_messages: int = 500) -> list[dict]:
    """获取未整合的消息（用于 LLM 上下文）"""

    # 只取未整合的部分
    unconsolidated = self.messages[self.last_consolidated:]
    sliced = unconsolidated[-max_messages:]  # 限制数量

    # 对齐到用户消息（避免孤立的 tool_result）
    for i, m in enumerate(sliced):
        if m.get("role") == "user":
            sliced = sliced[i:]
            break

    # 提取必要字段
    out = []
    for m in sliced:
        entry = {"role": m["role"], "content": m.get("content", "")}
        for k in ("tool_calls", "tool_call_id", "name"):
            if k in m:
                entry[k] = m[k]
        out.append(entry)

    return out
```

### 8.4 会话持久化

```python
def save(self, session: Session) -> None:
    """保存会话到磁盘"""
    path = self._get_session_path(session.key)

    with open(path, "w", encoding="utf-8") as f:
        # 元数据行
        metadata_line = {
            "_type": "metadata",
            "key": session.key,
            "created_at": session.created_at.isoformat(),
            "updated_at": session.updated_at.isoformat(),
            "last_consolidated": session.last_consolidated,
        }
        f.write(json.dumps(metadata_line) + "\n")

        # 消息行
        for msg in session.messages:
            f.write(json.dumps(msg, ensure_ascii=False) + "\n")
```

---

## 9. MessageBus 消息总线

**文件**: `nanobot/bus/queue.py`

### 9.1 设计目的

MessageBus 解耦了：
- **渠道层**: 接收用户消息
- **核心层**: 处理消息

这允许：
- 独立运行渠道和代理
- 异步消息处理
- 简单添加新渠道

### 9.2 核心接口

```python
class MessageBus:
    """异步消息队列"""

    def __init__(self):
        self._inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()
        self._outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()

    async def publish_inbound(self, msg: InboundMessage) -> None:
        """发布入站消息"""
        await self._inbound.put(msg)

    async def consume_inbound(self) -> InboundMessage:
        """消费入站消息（阻塞等待）"""
        return await self._inbound.get()

    async def publish_outbound(self, msg: OutboundMessage) -> None:
        """发布出站消息"""
        await self._outbound.put(msg)

    async def consume_outbound(self) -> OutboundMessage:
        """消费出站消息"""
        return await self._outbound.get()
```

### 9.3 消息类型

```python
@dataclass
class InboundMessage:
    """入站消息（用户 -> 代理）"""
    channel: str       # 渠道类型: telegram, discord, cli, ...
    sender_id: str      # 发送者 ID
    chat_id: str       # 聊天 ID
    content: str       # 消息内容
    media: list[str] | None = None      # 附件
    metadata: dict | None = None         # 额外信息


@dataclass
class OutboundMessage:
    """出站消息（代理 -> 用户）"""
    channel: str       # 目标渠道
    chat_id: str       # 目标聊天
    content: str       # 消息内容
    media: list[str] | None = None
    metadata: dict | None = None
```

---

## 10. 消息处理完整流程

下面是一个完整的消息处理流程图：

```
1. 用户在 Telegram 发送消息
         │
         ▼
2. Telegram 渠道接收事件
         │
         ▼
3. 转换为 InboundMessage
   {
     channel: "telegram",
     sender_id: "user123",
     chat_id: "chat456",
     content: "帮我写一个 Python 函数"
   }
         │
         ▼
4. 推送到 MessageBus
         │
         ▼
5. AgentLoop.run() 消费消息
         │
         ▼
6. _process_message() 处理
   │
   ├─ 6.1 获取或创建会话
   │      key = "telegram:chat456"
   │
   ├─ 6.2 检查是否需要记忆整合
   │      (超过 memory_window)
   │
   ├─ 6.3 构建消息列表
   │      - system: 系统提示词
   │        (ContextBuilder.build_system_prompt)
   │      - history: 会话历史
   │        (Session.get_history)
   │      - user: 当前消息
   │
   ├─ 6.4 运行代理循环
   │      - 调用 LLM
   │      - 如有工具调用则执行
   │      - 重复直到完成
   │
   ├─ 6.5 保存会话
   │      - 写入新消息到 JSONL
   │
   └─ 6.6 触发记忆整合（异步）
          - 调用 LLM 摘要
          - 写入 MEMORY.md / HISTORY.md
         │
         ▼
7. 返回 OutboundMessage
   {
     channel: "telegram",
     chat_id: "chat456",
     content: "好的，我来帮你..."
   }
         │
         ▼
8. MessageBus 发布出站消息
         │
         ▼
9. Telegram 渠道发送消息给用户
```

---

## 总结

nanobot 的代理系统是一个精心设计的轻量级 AI 助手框架：

1. **AgentLoop** 是核心引擎，协调整个处理流程
2. **ContextBuilder** 组装智能的系统提示词
3. **MemoryStore** 实现两层记忆系统
4. **SkillsLoader** 支持可扩展的技能机制
5. **SubagentManager** 实现后台任务并行处理
6. **ToolRegistry** 提供丰富的工具能力
7. **SessionManager** 管理对话历史
8. **MessageBus** 解耦各组件

这套架构使得 nanobot 能够：
- 轻量（约 4000 行核心代码）
- 可扩展（工具、技能、渠道）
- 智能（记忆、推理、工具使用）
- 高效（异步处理、增量整合）

希望这份文档能帮助您全面理解 nanobot 的代理系统！
