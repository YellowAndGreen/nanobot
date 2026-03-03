# 配置哲学：简洁与灵活的平衡

> 核心问题：如何在简洁和灵活之间取得平衡？

## 设计原则

### 原则一：单文件配置

**一个文件**：`~/.nanobot/config.json`

为什么？
- 容易找到
- 容易备份
- 容易理解

### 原则二：合理的默认值

**大部分配置有默认值**，用户只需要配置必需项：

```json
{
  "model": "openrouter/auto",
  "providers": {
    "openrouter": { "apiKey": "sk-or-..." }
  }
}
```

其他都是可选的。

### 原则三：渐进式复杂度

**简单场景简单配置**：

```json
{ "model": "claude-3-opus" }
```

**复杂场景灵活配置**：

```json
{
  "model": "claude-3-opus",
  "providers": {
    "anthropic": { "apiKey": "sk-ant-..." }
  },
  "channels": {
    "telegram": { "enabled": true, "token": "..." },
    "discord": { "enabled": true, "token": "..." }
  },
  "agents": {
    "defaults": {
      "maxIterations": 50,
      "temperature": 0.1
    }
  }
}
```

---

## 配置结构

```
~/.nanobot/
├── config.json          # 主配置文件
├── workspace/           # 工作空间
│   ├── SOUL.md         # 人格模板
│   ├── USER.md         # 用户上下文
│   ├── TOOLS.md        # 工具指南
│   ├── memory/         # 记忆存储
│   │   ├── MEMORY.md
│   │   └── HISTORY.md
│   ├── sessions/       # 会话存储
│   └── skills/         # 用户技能
└── history/            # CLI 历史
```

---

## Config Schema

### Pydantic 定义

```python
@dataclass
class Config:
    """主配置"""

    model: str = "openrouter/auto"

    providers: ProvidersConfig = field(default_factory=ProvidersConfig)
    channels: ChannelsConfig = field(default_factory=ChannelsConfig)
    agents: AgentsConfig = field(default_factory=AgentsConfig)
    exec: ExecToolConfig = field(default_factory=ExecToolConfig)
    mcpServers: dict[str, Any] = field(default_factory=dict)

@dataclass
class ProvidersConfig:
    """提供商配置"""
    openrouter: ProviderConfig | None = None
    anthropic: ProviderConfig | None = None
    openai: ProviderConfig | None = None
    deepseek: ProviderConfig | None = None
    # ...

@dataclass
class ProviderConfig:
    """单个提供商配置"""
    apiKey: str | None = None
    apiBase: str | None = None

@dataclass
class ChannelsConfig:
    """渠道配置"""
    telegram: TelegramConfig | None = None
    discord: DiscordConfig | None = None
    slack: SlackConfig | None = None
    # ...

@dataclass
class TelegramConfig:
    """Telegram 配置"""
    enabled: bool = False
    token: str | None = None
    allowFrom: list[str] = field(default_factory=list)

@dataclass
class AgentsConfig:
    """代理配置"""
    defaults: AgentDefaults = field(default_factory=AgentDefaults)

@dataclass
class AgentDefaults:
    """代理默认值"""
    model: str | None = None
    maxIterations: int = 40
    temperature: float = 0.1
    maxTokens: int = 4096
    memoryWindow: int = 100
```

**源码位置**：`nanobot/config/schema.py`

---

## camelCase / snake_case 双支持

### 问题

JSON 习惯用 camelCase，Python 习惯用 snake_case。

### 解决方案

```python
# 两种写法都有效
{
  "apiKey": "...",      # camelCase
  "api_key": "..."      # snake_case
}
```

### 实现

```python
def normalize_config(data: dict) -> dict:
    """规范化配置键名"""
    normalized = {}
    for key, value in data.items():
        # camelCase → snake_case
        snake_key = re.sub(r'([a-z0-9])([A-Z])', r'\1_\2', key).lower()
        normalized[snake_key] = value
    return normalized
```

---

## 环境变量覆盖

### 支持的环境变量

```bash
# 提供商 API Keys
OPENROUTER_API_KEY=sk-or-...
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
DEEPSEEK_API_KEY=sk-...

# 模型配置
NANOBOT_MODEL=openrouter/auto

# 网络配置
HTTP_PROXY=http://proxy:8080
HTTPS_PROXY=http://proxy:8080
```

### 加载优先级

```
1. 环境变量（最高优先级）
2. config.json
3. 默认值（最低优先级）
```

---

## 配置验证

### Pydantic 验证

```python
from pydantic import BaseModel, field_validator

class Config(BaseModel):
    model: str

    @field_validator('model')
    @classmethod
    def validate_model(cls, v):
        if not v:
            raise ValueError('model cannot be empty')
        return v
```

### 启动时验证

```python
def load_config() -> Config:
    """加载并验证配置"""
    config_path = Path.home() / ".nanobot" / "config.json"

    if not config_path.exists():
        raise ConfigError("Config file not found. Run 'nanobot onboard' first.")

    with open(config_path) as f:
        data = json.load(f)

    # Pydantic 验证
    config = Config(**normalize_config(data))

    return config
```

**源码位置**：`nanobot/config/loader.py`

---

## onboard 命令

### 功能

`nanobot onboard` 初始化配置和工作空间。

### 流程

```
1. 检查配置文件是否存在
   ├── 存在 → 询问是否覆盖或保留现有值
   └── 不存在 → 创建默认配置

2. 创建工作空间目录
   ~/.nanobot/workspace/

3. 同步模板文件
   SOUL.md, USER.md, TOOLS.md, AGENTS.md

4. 显示下一步提示
```

### 输出示例

```
✓ Created config at /home/user/.nanobot/config.json
✓ Created workspace at /home/user/.nanobot/workspace

🤖 nanobot is ready!

Next steps:
  1. Add your API key to ~/.nanobot/config.json
     Get one at: https://openrouter.ai/keys
  2. Chat: nanobot agent -m "Hello!"
```

**源码位置**：`nanobot/cli/commands.py` (onboard 函数)

---

## 配置示例

### 最小配置

```json
{
  "model": "openrouter/auto",
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-..."
    }
  }
}
```

### 完整配置

```json
{
  "model": "openrouter/auto",

  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-..."
    },
    "anthropic": {
      "apiKey": "sk-ant-..."
    },
    "openai": {
      "apiKey": "sk-..."
    }
  },

  "channels": {
    "telegram": {
      "enabled": true,
      "token": "123456:ABC...",
      "allowFrom": ["123456789"]
    },
    "discord": {
      "enabled": true,
      "token": "Bot ..."
    },
    "slack": {
      "enabled": false,
      "botToken": "xoxb-...",
      "appToken": "xapp-..."
    }
  },

  "agents": {
    "defaults": {
      "maxIterations": 50,
      "temperature": 0.1,
      "maxTokens": 4096,
      "memoryWindow": 100
    }
  },

  "exec": {
    "timeout": 30,
    "pathAppend": "/usr/local/bin"
  },

  "mcpServers": {
    "filesystem": {
      "command": "mcp-filesystem",
      "args": ["/home/user/workspace"]
    }
  }
}
```

---

## 设计权衡

### 为什么用 JSON 而不是 YAML/TOML？

| 格式 | 优点 | 缺点 |
|------|------|------|
| JSON | 通用、解析快、无歧义 | 不支持注释 |
| YAML | 支持注释、更人性化 | 解析复杂、有歧义 |
| TOML | 支持注释、类型明确 | 不够通用 |

**选择 JSON 的理由**：
1. 最通用，所有语言都支持
2. 无歧义，不会因缩进出错
3. IDE 支持好（JSON Schema）

### 为什么用 Pydantic？

1. **类型安全**：自动类型转换和验证
2. **文档生成**：可以自动生成 JSON Schema
3. **错误信息**：验证失败时有清晰的错误提示

---

## 源码索引

| 组件 | 文件位置 |
|------|---------|
| Config Schema | `nanobot/config/schema.py` |
| Config Loader | `nanobot/config/loader.py` |
| onboard 命令 | `nanobot/cli/commands.py` |
| 模板文件 | `nanobot/templates/` |

---

下一章：[13-EXTENSION-GUIDE.md](./13-EXTENSION-GUIDE.md) — 如何为这个项目添加新功能？
