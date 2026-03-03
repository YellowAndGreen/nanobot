# 扩展指南：添加新功能

> 核心问题：如何为这个项目添加新功能？

## 扩展点总览

nanobot 设计为高度可扩展。主要的扩展点：

| 扩展点 | 方式 | 难度 |
|--------|------|------|
| 新 Channel | 继承 `BaseChannel` | ⭐⭐ |
| 新 Provider | 添加 `ProviderSpec` | ⭐ |
| 新 Tool | 继承 `BaseTool` | ⭐⭐ |
| 新 Skill | 创建 Markdown 文件 | ⭐ |

---

## 添加新 Channel

### 步骤

1. 创建新文件：`nanobot/channels/newchannel.py`
2. 继承 `BaseChannel`
3. 实现 `start()`, `stop()`, `send()` 方法
4. 在 `ChannelManager` 中注册

### 示例：添加 IRC Channel

```python
# nanobot/channels/irc.py

import irc.client
from nanobot.channels.base import BaseChannel
from nanobot.bus.events import InboundMessage, OutboundMessage

class IRCChannel(BaseChannel):
    """IRC 渠道实现"""

    name = "irc"

    def __init__(self, config, bus):
        super().__init__(config, bus)
        self.client = None
        self.reactor = None

    async def start(self) -> None:
        """启动 IRC 客户端"""
        self.reactor = irc.client.Reactor()
        self.client = self.reactor.server()

        # 连接服务器
        self.client.connect(
            self.config.server,
            self.config.port,
            self.config.nickname,
            password=self.config.password,
        )

        # 加入频道
        for channel in self.config.channels:
            self.client.join(channel)

        # 注册消息处理器
        self.client.add_global_handler("pubmsg", self._on_message)

        # 启动事件循环
        self._running = True
        asyncio.create_task(self._run_reactor())

    async def _run_reactor(self):
        """运行 IRC 事件循环"""
        while self._running:
            self.reactor.process_once(timeout=0.1)
            await asyncio.sleep(0.01)

    def _on_message(self, connection, event):
        """处理收到的消息"""
        sender_id = event.source.nick
        chat_id = event.target
        content = event.arguments[0]

        # 转换为统一格式
        asyncio.create_task(self._handle_message(
            sender_id=sender_id,
            chat_id=chat_id,
            content=content,
        ))

    async def stop(self) -> None:
        """停止 IRC 客户端"""
        self._running = False
        if self.client:
            self.client.disconnect()

    async def send(self, msg: OutboundMessage) -> None:
        """发送消息到 IRC"""
        if self.client:
            self.client.privmsg(msg.chat_id, msg.content)
```

### 配置

```json
{
  "channels": {
    "irc": {
      "enabled": true,
      "server": "irc.libera.chat",
      "port": 6667,
      "nickname": "nanobot",
      "password": null,
      "channels": ["#nanobot"],
      "allowFrom": []
    }
  }
}
```

### 注册

```python
# nanobot/channels/manager.py

from nanobot.channels.irc import IRCChannel

CHANNEL_CLASSES = {
    "telegram": TelegramChannel,
    "discord": DiscordChannel,
    "irc": IRCChannel,  # 添加这行
    # ...
}
```

---

## 添加新 Provider

### 步骤

1. 在 `nanobot/providers/registry.py` 添加 `ProviderSpec`
2. 在 `nanobot/config/schema.py` 添加配置字段

### 示例：添加 Replicate Provider

```python
# nanobot/providers/registry.py

PROVIDERS = (
    # ... 现有 providers ...

    ProviderSpec(
        name="replicate",
        keywords=("replicate",),
        env_key="REPLICATE_API_KEY",
        display_name="Replicate",
        litellm_prefix="replicate",
        skip_prefixes=("replicate/",),
        is_gateway=False,
        is_local=False,
    ),
)
```

```python
# nanobot/config/schema.py

@dataclass
class ProvidersConfig:
    # ... 现有字段 ...
    replicate: ProviderConfig | None = None
```

### 使用

```json
{
  "model": "replicate/llama-70b",
  "providers": {
    "replicate": {
      "apiKey": "r8_..."
    }
  }
}
```

---

## 添加新 Tool

### 步骤

1. 创建新文件：`nanobot/agent/tools/mytool.py`
2. 继承 `BaseTool`
3. 在 `AgentLoop._register_default_tools()` 中注册

### 示例：添加 Database Tool

```python
# nanobot/agent/tools/database.py

import sqlite3
from pathlib import Path
from nanobot.agent.tools.base import BaseTool

class DatabaseTool(BaseTool):
    """SQLite 数据库查询工具"""

    name = "query_db"
    description = "Execute a SQL query on a SQLite database"
    parameters = {
        "type": "object",
        "properties": {
            "database": {
                "type": "string",
                "description": "Path to the SQLite database file",
            },
            "query": {
                "type": "string",
                "description": "SQL query to execute (SELECT only)",
            }
        },
        "required": ["database", "query"],
    }

    def __init__(self, workspace: Path, allowed_databases: list[str] | None = None):
        self.workspace = workspace
        self.allowed_databases = allowed_databases

    async def execute(self, database: str, query: str) -> str:
        """执行 SQL 查询"""
        # 安全检查
        if self.allowed_databases and database not in self.allowed_databases:
            return f"Error: Database '{database}' is not allowed"

        # 只允许 SELECT
        if not query.strip().upper().startswith("SELECT"):
            return "Error: Only SELECT queries are allowed"

        db_path = Path(database).expanduser()

        if not db_path.exists():
            return f"Error: Database not found: {database}"

        try:
            conn = sqlite3.connect(str(db_path))
            cursor = conn.cursor()
            cursor.execute(query)
            rows = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            conn.close()

            # 格式化输出
            result = [" | ".join(columns), "-" * len(" | ".join(columns))]
            for row in rows[:100]:  # 限制 100 行
                result.append(" | ".join(str(cell) for cell in row))

            return "\n".join(result)

        except Exception as e:
            return f"Error executing query: {e}"
```

### 注册

```python
# nanobot/agent/loop.py

from nanobot.agent.tools.database import DatabaseTool

def _register_default_tools(self) -> None:
    # ... 现有工具 ...

    # 添加数据库工具
    self.tools.register(DatabaseTool(
        workspace=self.workspace,
        allowed_databases=self.config.allowed_databases,
    ))
```

---

## 添加新 Skill

### 步骤

1. 创建目录：`~/.nanobot/workspace/skills/my-skill/`
2. 创建文件：`SKILL.md`
3. 编写 YAML frontmatter 和内容

### 示例：添加 Jira Skill

```markdown
---
name: jira
description: "Jira issue tracking integration"
metadata:
  nanobot:
    emoji: 📋
    requires:
      bins: [jira]
    install:
      - id: npm
        kind: npm
        formula: jira-cli
        bins: [jira]
        label: Install via npm
---

# Jira Skill

Interact with Jira using the `jira` CLI.

## Prerequisites

```bash
# Install jira-cli
npm install -g jira-cli

# Authenticate
jira login
```

## Common Operations

### List Issues
```bash
jira issue list --assignee me --status "In Progress"
```

### Create Issue
```bash
jira issue create --project PROJ --type Bug --summary "Bug title" --description "Details..."
```

### Update Issue
```bash
jira issue update PROJ-123 --status "In Progress"
```

### Add Comment
```bash
jira issue comment add PROJ-123 --comment "Updated status..."
```

### Search
```bash
jira search "project = PROJ AND status = Open"
```
```

### 无需代码修改

Skill 的优势是不需要修改任何代码，只需创建 Markdown 文件。

---

## 测试扩展

### 测试 Channel

```python
# tests/test_irc_channel.py

import pytest
from nanobot.channels.irc import IRCChannel
from nanobot.bus.queue import MessageBus

@pytest.fixture
def irc_channel():
    bus = MessageBus()
    config = IRCConfig(
        server="irc.example.com",
        port=6667,
        nickname="testbot",
        channels=["#test"],
    )
    return IRCChannel(config, bus)

@pytest.mark.asyncio
async def test_irc_send(irc_channel):
    """测试发送消息"""
    await irc_channel.start()

    # 模拟发送消息
    from nanobot.bus.events import OutboundMessage
    msg = OutboundMessage(channel="irc", chat_id="#test", content="Hello")

    await irc_channel.send(msg)
    # 验证消息已发送
```

### 测试 Tool

```python
# tests/test_database_tool.py

import pytest
import sqlite3
from pathlib import Path
from nanobot.agent.tools.database import DatabaseTool

@pytest.fixture
def test_db(tmp_path):
    """创建测试数据库"""
    db_path = tmp_path / "test.db"
    conn = sqlite3.connect(str(db_path))
    conn.execute("CREATE TABLE users (id INTEGER, name TEXT)")
    conn.execute("INSERT INTO users VALUES (1, 'Alice')")
    conn.execute("INSERT INTO users VALUES (2, 'Bob')")
    conn.commit()
    conn.close()
    return str(db_path)

@pytest.mark.asyncio
async def test_query(test_db):
    """测试数据库查询"""
    tool = DatabaseTool(workspace=Path("/tmp"))
    result = await tool.execute(
        database=test_db,
        query="SELECT * FROM users"
    )

    assert "Alice" in result
    assert "Bob" in result
```

---

## 最佳实践

### Channel 开发

1. **错误处理**：网络断开时自动重连
2. **限流**：遵守平台的 API 限制
3. **日志**：记录关键事件
4. **测试**：使用 mock 测试，不依赖真实平台

### Tool 开发

1. **安全检查**：验证输入，防止注入
2. **超时**：长时间操作设置超时
3. **错误信息**：返回清晰的错误信息
4. **幂等性**：相同输入产生相同输出

### Skill 开发

1. **简洁**：只包含必要的信息
2. **示例**：提供常用命令示例
3. **依赖**：明确列出依赖的 CLI 工具
4. **安装**：提供安装指令

---

## 贡献代码

### 流程

1. Fork 仓库
2. 创建功能分支：`git checkout -b feature/new-channel`
3. 编写代码和测试
4. 运行测试：`pytest`
5. 运行 lint：`ruff check nanobot/`
6. 提交 PR

### 代码风格

- 使用 ruff 格式化
- 添加类型注解
- 编写文档字符串
- 添加测试

---

## 源码索引

| 扩展点 | 模板位置 |
|--------|---------|
| Channel | `nanobot/channels/base.py` |
| Provider | `nanobot/providers/registry.py` |
| Tool | `nanobot/agent/tools/base.py` |
| Skill | `nanobot/skills/*/SKILL.md` |

---

恭喜你完成了整个 Wiki 的阅读！

现在你对 nanobot 的架构和设计有了全面的理解。开始构建你自己的扩展吧！
