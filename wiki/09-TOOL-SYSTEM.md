# 工具系统：LLM 的手脚

> 核心问题：如何让 LLM 拥有与外界交互的能力？

## 问题

LLM 本身只能处理文本。但 AI 助手需要：
- 读写文件
- 执行命令
- 搜索网络
- 发送消息
- ...

这就是工具系统的职责。

---

## Tool 抽象设计

### 核心要素

每个工具都需要：

1. **name**：工具名称（LLM 调用时使用）
2. **description**：工具描述（告诉 LLM 这个工具做什么）
3. **parameters**：参数定义（JSON Schema）
4. **execute**：执行方法

### 基类定义

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

---

## 内置工具

| 工具 | 用途 | 文件位置 |
|------|------|---------|
| `read_file` | 读取文件内容 | `tools/filesystem.py` |
| `write_file` | 写入文件 | `tools/filesystem.py` |
| `edit_file` | 编辑文件（字符串替换） | `tools/filesystem.py` |
| `list_dir` | 列出目录内容 | `tools/filesystem.py` |
| `exec` | 执行 Shell 命令 | `tools/shell.py` |
| `web_search` | 搜索网络 | `tools/web.py` |
| `web_fetch` | 获取网页内容 | `tools/web.py` |
| `message` | 发送消息 | `tools/message.py` |
| `spawn` | 启动后台任务 | `tools/spawn.py` |
| `cron` | 定时任务管理 | `tools/cron.py` |

---

## 工具实现示例

### ReadFileTool

```python
class ReadFileTool(BaseTool):
    name = "read_file"
    description = "Read the contents of a file from the local filesystem."
    parameters = {
        "type": "object",
        "properties": {
            "path": {
                "type": "string",
                "description": "The absolute path to the file to read.",
            }
        },
        "required": ["path"],
    }

    def __init__(self, workspace: Path, allowed_dir: Path | None = None):
        self.workspace = workspace
        self.allowed_dir = allowed_dir

    async def execute(self, path: str) -> str:
        file_path = Path(path).expanduser().resolve()

        # 安全检查
        if self.allowed_dir and not str(file_path).startswith(str(self.allowed_dir)):
            return f"Error: Access denied. Path must be within {self.allowed_dir}"

        if not file_path.exists():
            return f"Error: File not found: {path}"

        if not file_path.is_file():
            return f"Error: Not a file: {path}"

        try:
            content = file_path.read_text(encoding="utf-8")
            return content
        except Exception as e:
            return f"Error reading file: {e}"
```

**源码位置**：`nanobot/agent/tools/filesystem.py`

### ExecTool

```python
class ExecTool(BaseTool):
    name = "exec"
    description = "Execute a shell command and return the output."
    parameters = {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "The command to execute.",
            },
            "timeout": {
                "type": "integer",
                "description": "Timeout in seconds (default: 30).",
                "default": 30,
            }
        },
        "required": ["command"],
    }

    def __init__(self, working_dir: str, timeout: int = 30,
                 restrict_to_workspace: bool = False,
                 path_append: str = ""):
        self.working_dir = working_dir
        self.timeout = timeout
        self.restrict_to_workspace = restrict_to_workspace
        self.path_append = path_append

    async def execute(self, command: str, timeout: int | None = None) -> str:
        timeout = timeout or self.timeout

        try:
            process = await asyncio.create_subprocess_shell(
                command,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                cwd=self.working_dir,
            )

            stdout, stderr = await asyncio.wait_for(
                process.communicate(),
                timeout=timeout
            )

            output = stdout.decode("utf-8", errors="replace")
            error = stderr.decode("utf-8", errors="replace")

            if process.returncode != 0:
                return f"Exit code: {process.returncode}\nStdout:\n{output}\nStderr:\n{error}"
            return output or "(no output)"

        except asyncio.TimeoutError:
            process.kill()
            return f"Error: Command timed out after {timeout} seconds"
        except Exception as e:
            return f"Error executing command: {e}"
```

**源码位置**：`nanobot/agent/tools/shell.py`

---

## ToolRegistry

### 职责

1. **注册工具**
2. **获取工具定义**（供 LLM 使用）
3. **执行工具调用**

### 实现

```python
class ToolRegistry:
    """工具注册表"""

    def __init__(self):
        self._tools: dict[str, BaseTool] = {}

    def register(self, tool: BaseTool) -> None:
        """注册工具"""
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool | None:
        """获取工具"""
        return self._tools.get(name)

    def get_definitions(self) -> list[dict]:
        """获取所有工具定义"""
        return [tool.get_definition() for tool in self._tools.values()]

    async def execute(self, name: str, args: dict) -> str:
        """执行工具"""
        tool = self._tools.get(name)
        if not tool:
            return f"Error: Unknown tool '{name}'"

        try:
            return await tool.execute(**args)
        except Exception as e:
            return f"Error executing {name}: {e}"
```

**源码位置**：`nanobot/agent/tools/registry.py`

### 注册默认工具

```python
def _register_default_tools(self) -> None:
    """注册默认工具集"""
    allowed_dir = self.workspace if self.restrict_to_workspace else None

    # 文件系统工具
    for cls in (ReadFileTool, WriteFileTool, EditFileTool, ListDirTool):
        self.tools.register(cls(workspace=self.workspace, allowed_dir=allowed_dir))

    # Shell 执行
    self.tools.register(ExecTool(
        working_dir=str(self.workspace),
        timeout=self.exec_config.timeout,
        restrict_to_workspace=self.restrict_to_workspace,
    ))

    # 网络工具
    self.tools.register(WebSearchTool(api_key=self.brave_api_key))
    self.tools.register(WebFetchTool())

    # 消息发送
    self.tools.register(MessageTool(send_callback=self.bus.publish_outbound))

    # 后台任务
    self.tools.register(SpawnTool(manager=self.subagents))

    # 定时任务
    if self.cron_service:
        self.tools.register(CronTool(self.cron_service))
```

**源码位置**：`nanobot/agent/loop.py:115-131`

---

## 工具调用流程

```
┌─────────────────────────────────────────────────────────────┐
│                     Tool Call Flow                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. LLM 返回 tool_calls                                     │
│     Response(                                               │
│       content=None,                                         │
│       tool_calls=[ToolCall(name="read_file", args=...)]     │
│     )                                                       │
│                                                             │
│  2. AgentLoop 提取 tool_calls                               │
│     for tc in response.tool_calls:                          │
│       result = await tools.execute(tc.name, tc.args)        │
│                                                             │
│  3. ToolRegistry 查找并执行                                  │
│     tool = registry.get("read_file")                        │
│     result = await tool.execute(**args)                     │
│                                                             │
│  4. 将结果追加到消息列表                                     │
│     messages.append({                                       │
│       "role": "tool",                                       │
│       "tool_call_id": tc.id,                                │
│       "content": result                                     │
│     })                                                      │
│                                                             │
│  5. 再次调用 LLM                                            │
│     response = await provider.chat(messages, tools)         │
│                                                             │
│  6. 重复直到 LLM 不再调用工具                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 代码实现

```python
async def _run_agent_loop(self, messages, on_progress=None):
    iteration = 0

    while iteration < self.max_iterations:
        iteration += 1

        # 调用 LLM
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),
        )

        if response.has_tool_calls:
            # 格式化 tool_calls
            tool_call_dicts = [
                {
                    "id": tc.id,
                    "type": "function",
                    "function": {
                        "name": tc.name,
                        "arguments": json.dumps(tc.arguments)
                    }
                }
                for tc in response.tool_calls
            ]

            # 添加 assistant 消息
            messages = self.context.add_assistant_message(
                messages, response.content, tool_call_dicts
            )

            # 执行每个工具调用
            for tc in response.tool_calls:
                result = await self.tools.execute(tc.name, tc.arguments)
                messages = self.context.add_tool_result(
                    messages, tc.id, tc.name, result
                )
        else:
            # 无工具调用，返回结果
            return response.content
```

**源码位置**：`nanobot/agent/loop.py:180-257`

---

## 安全边界

### exec 工具的限制

```python
# 配置项
class ExecToolConfig:
    timeout: int = 30          # 默认超时
    path_append: str = ""      # 额外的 PATH

# 使用限制
- 超时限制（默认 30 秒）
- 工作目录限制（可选 restrict_to_workspace）
- 不允许交互式命令
```

### 文件系统工具的限制

```python
# restrict_to_workspace 模式
if self.allowed_dir and not str(file_path).startswith(str(self.allowed_dir)):
    return "Error: Access denied"
```

---

## MCP：扩展工具生态

### 什么是 MCP？

Model Context Protocol (MCP) 是一个开放协议，允许 LLM 连接外部工具和数据源。

### nanobot 的 MCP 支持

```python
# 配置
{
  "mcpServers": {
    "filesystem": {
      "command": "mcp-filesystem",
      "args": ["/home/user/workspace"]
    }
  }
}

# 连接
async def _connect_mcp(self):
    if self._mcp_servers:
        await connect_mcp_servers(self._mcp_servers, self.tools, self._mcp_stack)
```

**源码位置**：`nanobot/agent/tools/mcp.py`

---

## 添加新工具

### 步骤

1. 创建工具类（继承 `BaseTool`）
2. 在 `AgentLoop._register_default_tools()` 中注册

### 示例

```python
# nanobot/agent/tools/my_tool.py
class MyTool(BaseTool):
    name = "my_tool"
    description = "Do something useful"
    parameters = {
        "type": "object",
        "properties": {
            "input": {"type": "string"}
        },
        "required": ["input"],
    }

    async def execute(self, input: str) -> str:
        return f"Processed: {input}"

# nanobot/agent/loop.py
def _register_default_tools(self):
    ...
    self.tools.register(MyTool())
```

---

## 源码索引

| 组件 | 文件位置 |
|------|---------|
| ToolRegistry | `nanobot/agent/tools/registry.py` |
| BaseTool | `nanobot/agent/tools/base.py` |
| Filesystem Tools | `nanobot/agent/tools/filesystem.py` |
| ExecTool | `nanobot/agent/tools/shell.py` |
| Web Tools | `nanobot/agent/tools/web.py` |
| MessageTool | `nanobot/agent/tools/message.py` |
| SpawnTool | `nanobot/agent/tools/spawn.py` |
| CronTool | `nanobot/agent/tools/cron.py` |
| MCP Tools | `nanobot/agent/tools/mcp.py` |

---

下一章：[10-SKILL-ARCHITECTURE.md](./10-SKILL-ARCHITECTURE.md) — 如何动态扩展 AI 的能力？
