# 上下文构建：让 LLM 理解世界

> 核心问题：如何让 LLM 理解它所处的环境？

## 问题

LLM 本身是无状态的。每次调用时，我们需要告诉它：

1. **你是谁**：身份和角色
2. **你在哪**：运行环境、工作空间
3. **你知道什么**：记忆和知识
4. **你能做什么**：工具和技能
5. **现在是什么时候**：当前时间

这就是上下文构建的核心任务。

---

## 系统提示词的组成

```
┌─────────────────────────────────────────────────────────────┐
│                     System Prompt                            │
├─────────────────────────────────────────────────────────────┤
│  1. Identity        # 基本身份（硬编码）                      │
│  2. Bootstrap Files # SOUL.md, USER.md, TOOLS.md, ...       │
│  3. Memory          # MEMORY.md（长期记忆）                  │
│  4. Active Skills   # always 类型的技能                      │
│  5. Skills Summary  # 可用技能列表                           │
└─────────────────────────────────────────────────────────────┘
```

**源码位置**：`nanobot/agent/context.py:26-53`

---

## 第一部分：Identity（身份）

### 内容

```markdown
# nanobot 🐈

You are nanobot, a helpful AI assistant.

## Runtime
macOS arm64, Python 3.11.0

## Workspace
Your workspace is at: /home/user/.nanobot/workspace
- Long-term memory: /home/user/.nanobot/workspace/memory/MEMORY.md
- History log: /home/user/.nanobot/workspace/memory/HISTORY.md
- Custom skills: /home/user/.nanobot/workspace/skills/{skill-name}/SKILL.md

## nanobot Guidelines
- State intent before tool calls, but NEVER predict results before receiving them.
- Before modifying a file, read it first.
- After writing or editing a file, re-read it if accuracy matters.
- If a tool call fails, analyze the error before retrying.
- Ask for clarification when the request is ambiguous.
```

### 代码

```python
def _get_identity(self) -> str:
    """Get the core identity section."""
    workspace_path = str(self.workspace.expanduser().resolve())
    system = platform.system()
    runtime = f"{'macOS' if system == 'Darwin' else system} {platform.machine()}, Python {platform.python_version()}"

    return f"""# nanobot 🐈

You are nanobot, a helpful AI assistant.

## Runtime
{runtime}

## Workspace
Your workspace is at: {workspace_path}
- Long-term memory: {workspace_path}/memory/MEMORY.md
- History log: {workspace_path}/memory/HISTORY.md
- Custom skills: {workspace_path}/skills/{{skill-name}}/SKILL.md

## nanobot Guidelines
- State intent before tool calls, but NEVER predict or claim results before receiving them.
- Before modifying a file, read it first. Do not assume files or directories exist.
- After writing or editing a file, re-read it if accuracy matters.
- If a tool call fails, analyze the error before retrying with a different approach.
- Ask for clarification when the request is ambiguous.

Reply directly with text for conversations. Only use the 'message' tool to send to a specific chat channel."""
```

**源码位置**：`nanobot/agent/context.py:55-81`

### 为什么部分硬编码？

1. **稳定性**：核心指令不应该被用户意外修改
2. **版本兼容**：升级时自动获得改进
3. **简洁性**：用户不需要关心这些细节

---

## 第二部分：Bootstrap Files（启动文件）

### 文件列表

| 文件 | 用途 |
|------|------|
| `SOUL.md` | 核心人格和行为准则 |
| `USER.md` | 用户上下文（偏好、背景） |
| `TOOLS.md` | 工具使用指南 |
| `AGENTS.md` | 子代理协作规则 |
| `IDENTITY.md` | 自定义身份覆盖 |

### 加载逻辑

```python
BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md", "IDENTITY.md"]

def _load_bootstrap_files(self) -> str:
    """Load all bootstrap files from workspace."""
    parts = []

    for filename in self.BOOTSTRAP_FILES:
        file_path = self.workspace / filename
        if file_path.exists():
            content = file_path.read_text(encoding="utf-8")
            parts.append(f"## {filename}\n\n{content}")

    return "\n\n".join(parts) if parts else ""
```

**源码位置**：`nanobot/agent/context.py:93-103`

### SOUL.md 示例

```markdown
# Soul

You are a thoughtful and precise assistant.

## Communication Style
- Be concise but thorough
- Explain your reasoning when it helps understanding
- Admit uncertainty when appropriate

## Problem Solving
- Break down complex problems into steps
- Verify assumptions before acting
- Learn from errors and adapt
```

### USER.md 示例

```markdown
# User Context

## Preferences
- Language: Chinese (Simplified)
- Timezone: Asia/Shanghai
- Code style: Google Python Style Guide

## Background
- Software engineer with 5 years experience
- Mainly works with Python and TypeScript
- Uses macOS for development
```

---

## 第三部分：Memory（记忆）

### 双层记忆

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory System                            │
├─────────────────────────────────────────────────────────────┤
│  MEMORY.md          # 长期记忆（结构化事实）                  │
│  - 用户偏好                                                  │
│  - 重要决策                                                  │
│  - 持久化知识                                                │
├─────────────────────────────────────────────────────────────┤
│  HISTORY.md         # 历史日志（可搜索）                     │
│  - [2024-01-15 10:30] 用户问了关于 X 的问题，讨论了 Y        │
│  - [2024-01-16 14:20] 帮助用户调试了 Z 问题                  │
└─────────────────────────────────────────────────────────────┘
```

### 加载代码

```python
def get_memory_context(self) -> str:
    """获取长期记忆上下文"""
    long_term = self.read_long_term()
    return f"## Long-term Memory\n{long_term}" if long_term else ""
```

**源码位置**：`nanobot/agent/memory.py:65-67`

---

## 第四部分：Skills（技能）

### 技能类型

1. **Always Skills**：自动加载到系统提示词
2. **On-demand Skills**：LLM 按需读取

### Always Skills

```python
def build_system_prompt(self, skill_names=None):
    parts = [
        self._get_identity(),
        self._load_bootstrap_files(),
        self.memory.get_memory_context(),
    ]

    # Always skills: 自动加载
    always_skills = self.skills.get_always_skills()
    if always_skills:
        always_content = self.skills.load_skills_for_context(always_skills)
        if always_content:
            parts.append(f"# Active Skills\n\n{always_content}")

    # Skills summary: 供 LLM 选择
    skills_summary = self.skills.build_skills_summary()
    if skills_summary:
        parts.append(f"""# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Skills with available="false" need dependencies installed first.

{skills_summary}""")

    return "\n\n---\n\n".join(parts)
```

**源码位置**：`nanobot/agent/context.py:26-53`

### Skills Summary 示例

```markdown
| Skill | Description | Available |
|-------|-------------|-----------|
| github | GitHub operations via gh CLI | ✅ |
| docker | Docker container management | ✅ |
| weather | Weather information lookup | ❌ (needs API key) |
```

---

## 运行时信息注入

### 问题

系统提示词是静态的，但有些信息需要动态注入：
- 当前时间
- 当前渠道
- 会话 ID

### 解决方案

在用户消息前注入一个"运行时上下文"块：

```python
@staticmethod
def _build_runtime_context(channel, chat_id) -> str:
    """构建运行时元数据块"""
    now = datetime.now().strftime("%Y-%m-%d %H:%M (%A)")
    tz = time.strftime("%Z") or "UTC"
    lines = [f"Current Time: {now} ({tz})"]
    if channel and chat_id:
        lines += [f"Channel: {channel}", f"Chat ID: {chat_id}"]
    return "[Runtime Context — metadata only, not instructions]\n" + "\n".join(lines)
```

**源码位置**：`nanobot/agent/context.py:83-91`

### 消息列表结构

```python
def build_messages(self, history, current_message, ...):
    return [
        {"role": "system", "content": self.build_system_prompt()},
        *history,
        {"role": "user", "content": self._build_runtime_context(...)},
        {"role": "user", "content": current_message},
    ]
```

### 实际效果

```
System: # nanobot 🐈
        ...（完整系统提示词）

User (history): 之前的消息...

User: [Runtime Context — metadata only, not instructions]
      Current Time: 2024-01-15 10:30 (Monday) (CST)
      Channel: telegram
      Chat ID: 123456789

User: 帮我查北京天气
```

---

## 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                   ContextBuilder.build_messages()            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐                                            │
│  │  Identity   │──▶ 基本身份 + 运行环境                      │
│  └─────────────┘                                            │
│         +                                                   │
│  ┌─────────────┐                                            │
│  │  Bootstrap  │──▶ SOUL.md + USER.md + TOOLS.md + ...      │
│  └─────────────┘                                            │
│         +                                                   │
│  ┌─────────────┐                                            │
│  │   Memory    │──▶ MEMORY.md（长期记忆）                   │
│  └─────────────┘                                            │
│         +                                                   │
│  ┌─────────────┐                                            │
│  │   Skills    │──▶ Always Skills + Skills Summary          │
│  └─────────────┘                                            │
│         =                                                   │
│  ┌─────────────┐                                            │
│  │   System    │──▶ 完整系统提示词                          │
│  └─────────────┘                                            │
│         +                                                   │
│  ┌─────────────┐                                            │
│  │   History   │──▶ 会话历史记录                            │
│  └─────────────┘                                            │
│         +                                                   │
│  ┌─────────────┐                                            │
│  │  Runtime    │──▶ 当前时间 + 渠道信息                     │
│  └─────────────┘                                            │
│         +                                                   │
│  ┌─────────────┐                                            │
│  │   User Msg  │──▶ 当前用户消息                            │
│  └─────────────┘                                            │
│         =                                                   │
│  ┌─────────────┐                                            │
│  │  Messages   │──▶ 发送给 LLM 的完整消息列表               │
│  └─────────────┘                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 模板同步

### 启动时同步

```python
# nanobot/cli/commands.py (onboard)
def sync_templates():
    """将内置模板同步到工作空间"""
    template_dir = Path(__file__).parent.parent / "templates"
    workspace_dir = Path.home() / ".nanobot" / "workspace"

    for template in template_dir.glob("*.md"):
        dest = workspace_dir / template.name
        if not dest.exists():
            shutil.copy(template, dest)
```

### 模板位置

- **内置模板**：`nanobot/templates/`
- **用户工作空间**：`~/.nanobot/workspace/`

### 用户可以自定义

用户可以编辑 `~/.nanobot/workspace/` 下的文件来自定义：
- `SOUL.md`：调整人格
- `USER.md`：添加个人背景
- `TOOLS.md`：自定义工具使用指南

---

## 源码索引

| 功能 | 文件位置 |
|------|---------|
| ContextBuilder | `nanobot/agent/context.py:15-165` |
| Identity 构建 | `nanobot/agent/context.py:55-81` |
| Bootstrap 加载 | `nanobot/agent/context.py:93-103` |
| 运行时注入 | `nanobot/agent/context.py:83-91` |
| 消息构建 | `nanobot/agent/context.py:105-120` |
| MemoryStore | `nanobot/agent/memory.py:45-150` |
| SkillsLoader | `nanobot/agent/skills.py` |
| 内置模板 | `nanobot/templates/` |

---

下一章：[08-MEMORY-DESIGN.md](./08-MEMORY-DESIGN.md) — 如何让 AI 拥有长期记忆？
