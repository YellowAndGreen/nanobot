# Nanobot Wiki

欢迎来到 Nanobot Wiki。这套文档从**顶级工程师的构思视角**，展示一个 AI 助手框架是如何从零开始一步步设计和构建的。

## 这套 Wiki 的特点

不同于传统的代码文档，这里我们关注的是：

- **为什么**要做这个设计决策？
- 这个抽象是**如何被识别**出来的？
- **权衡**了哪些替代方案？
- **演进**过程是怎样的？

## 目标读者

- 想要深入理解 nanobot 架构的开发者
- 准备扩展 nanobot 功能的贡献者
- 想要构建类似 AI 助手框架的工程师
- 对系统设计感兴趣的学习者

## 推荐阅读路径

### 路径一：按思维演进（推荐）

如果你想要理解"为什么这样设计"，按这个顺序阅读：

```
00-THINKING.md          → 从零开始构建的思维过程
01-PROBLEM-SPACE.md     → 我们要解决什么问题
02-ARCHITECTURE-VISION.md → 整体架构的顶层思考
03-LAYERED-DESIGN.md    → 四层架构详解
04-MINIMAL-VIABLE.md    → 最小可行产品的诞生
05-ABSTRACTIONS.md      → 关键抽象的识别
06-MESSAGE-FLOW.md      → 消息如何流转
07-CONTEXT-BUILDING.md  → 如何让 LLM 理解世界
08-MEMORY-DESIGN.md     → 双层记忆系统
09-TOOL-SYSTEM.md       → LLM 的手脚
10-SKILL-ARCHITECTURE.md → 动态能力加载
11-PROACTIVE-AGENT.md   → 主动性：Cron 与 Heartbeat
12-CONFIGURATION-PHILOSOPHY.md → 配置的哲学
13-EXTENSION-GUIDE.md   → 如何扩展
```

### 路径二：按模块快速查阅

如果你已经熟悉基本概念，只想查特定模块：

| 主题 | 章节 |
|------|------|
| 整体架构 | [02-ARCHITECTURE-VISION.md](./02-ARCHITECTURE-VISION.md) |
| 分层设计 | [03-LAYERED-DESIGN.md](./03-LAYERED-DESIGN.md) |
| 消息处理 | [06-MESSAGE-FLOW.md](./06-MESSAGE-FLOW.md) |
| 上下文构建 | [07-CONTEXT-BUILDING.md](./07-CONTEXT-BUILDING.md) |
| 记忆系统 | [08-MEMORY-DESIGN.md](./08-MEMORY-DESIGN.md) |
| 工具系统 | [09-TOOL-SYSTEM.md](./09-TOOL-SYSTEM.md) |
| 技能系统 | [10-SKILL-ARCHITECTURE.md](./10-SKILL-ARCHITECTURE.md) |
| 定时任务 | [11-PROACTIVE-AGENT.md](./11-PROACTIVE-AGENT.md) |
| 扩展开发 | [13-EXTENSION-GUIDE.md](./13-EXTENSION-GUIDE.md) |

### 路径三：快速参考

| 文档 | 内容 |
|------|------|
| [agent-core.md](./agent-core.md) | Agent 核心模块详解（代理引擎、上下文、记忆、技能） |
| [cli-commands.md](./cli-commands.md) | CLI 命令详解 |

## 源码索引

核心源码位置：

```
nanobot/
├── agent/
│   ├── loop.py           # AgentLoop - 代理处理引擎
│   ├── context.py        # ContextBuilder - 系统提示词构建
│   ├── memory.py         # MemoryStore - 持久化记忆
│   ├── skills.py         # SkillsLoader - 技能加载
│   ├── subagent.py       # SubagentManager - 后台任务
│   └── tools/            # 内置工具实现
├── bus/
│   ├── events.py         # InboundMessage / OutboundMessage
│   └── queue.py          # MessageBus - 消息总线
├── channels/
│   ├── base.py           # BaseChannel - 通道抽象基类
│   ├── manager.py        # ChannelManager - 通道管理
│   └── *.py              # 各平台实现
├── providers/
│   ├── registry.py       # ProviderSpec - 提供商元数据
│   └── litellm_provider.py # LiteLLM 封装
├── session/
│   └── manager.py        # Session / SessionManager
├── cron/
│   └── service.py        # CronService - 定时任务
├── heartbeat/
│   └── service.py        # HeartbeatService - 心跳
└── config/
    ├── schema.py         # Pydantic 配置模型
    └── loader.py         # 配置加载
```

## 文档约定

- `file_path:line_number` 格式引用源码位置
- Mermaid 图表展示流程和结构
- 每章以核心问题开头，驱动思考

## 快速开始

```bash
# 安装
pip install -e .

# 初始化配置
nanobot onboard

# 交互式聊天
nanobot agent

# 启动多渠道网关
nanobot gateway
```

## 核心设计理念

1. **核心极简** — ~4000 行代码完成核心功能
2. **插件化扩展** — Channel、Provider、Tool、Skill 都是插件
3. **异步优先** — 所有 I/O 操作都是异步的
4. **关注点分离** — 通道无关、模型无关、存储无关

---

开始阅读：[00-THINKING.md](./00-THINKING.md) — 从零开始构建的思维过程
