# 主动代理：Cron 与 Heartbeat

> 核心问题：如何让 AI 主动做事，而不是被动响应？

## 问题

传统的 AI 助手是**被动**的：只有用户发送消息，它才会响应。

但很多场景需要**主动**行为：
- 每天早上发送天气预报
- 每小时检查服务器状态
- 定期提醒重要事项

nanobot 通过两个机制实现主动性：
1. **Cron**：定时任务
2. **Heartbeat**：周期性检查

---

## Cron：定时任务系统

### 三种调度方式

| 方式 | 语法 | 示例 |
|------|------|------|
| `every` | 间隔秒数 | 每 3600 秒执行 |
| `cron` | Cron 表达式 | 每天 9:00 执行 |
| `at` | 一次性任务 | 2024-01-20 15:00 执行 |

### CLI 命令

```bash
# 列出任务
nanobot cron list
nanobot cron list --all  # 包含禁用的

# 添加任务
nanobot cron add -n "Hourly check" -m "Check system status" -e 3600
nanobot cron add -n "Morning" -m "Good morning!" -c "0 9 * * *"
nanobot cron add -n "Reminder" -m "Don't forget!" --at "2024-01-20T15:00:00"

# 删除任务
nanobot cron remove abc1

# 启用/禁用
nanobot cron enable abc1
nanobot cron enable abc1 --disable

# 手动运行
nanobot cron run abc1
nanobot cron run abc1 -f  # 强制运行（即使已禁用）
```

### 数据结构

```python
@dataclass
class CronJob:
    id: str                    # 任务 ID
    name: str                  # 任务名称
    message: str               # 发送给代理的消息
    schedule: ScheduleType     # 调度类型
    enabled: bool = True       # 是否启用
    timezone: str = "UTC"      # 时区
    deliver: bool = False      # 是否发送结果到渠道
    channel: str | None = None # 目标渠道
    chat_id: str | None = None # 目标会话
    last_run: datetime | None = None
    next_run: datetime | None = None

class ScheduleType:
    every: int | None        # 间隔秒数
    cron: str | None         # Cron 表达式
    at: datetime | None      # 一次性时间
```

### CronService 实现

```python
class CronService:
    """定时任务服务"""

    def __init__(self, workspace: Path, agent_loop: AgentLoop):
        self.workspace = workspace
        self.agent = agent_loop
        self.jobs_file = workspace / "cron.json"
        self.jobs: list[CronJob] = []
        self._running = False

    async def start(self) -> None:
        """启动定时任务服务"""
        self._load_jobs()
        self._running = True

        while self._running:
            now = datetime.now(timezone.utc)

            for job in self.jobs:
                if not job.enabled:
                    continue

                if job.next_run and now >= job.next_run:
                    await self._run_job(job)

            await asyncio.sleep(1)  # 每秒检查一次

    async def _run_job(self, job: CronJob) -> None:
        """执行任务"""
        logger.info(f"Running cron job: {job.name}")

        # 通过 agent_loop 处理消息
        response = await self.agent.process_direct(
            content=job.message,
            session_key=f"cron:{job.id}",
        )

        # 更新下次运行时间
        job.last_run = datetime.now(timezone.utc)
        job.next_run = self._calculate_next_run(job)
        self._save_jobs()

        # 如果配置了发送结果
        if job.deliver and job.channel and job.chat_id:
            await self.agent.bus.publish_outbound(OutboundMessage(
                channel=job.channel,
                chat_id=job.chat_id,
                content=response,
            ))

    def _calculate_next_run(self, job: CronJob) -> datetime:
        """计算下次运行时间"""
        if job.schedule.every:
            return datetime.now(timezone.utc) + timedelta(seconds=job.schedule.every)
        elif job.schedule.cron:
            # 解析 cron 表达式
            cron = croniter(job.schedule.cron, datetime.now(timezone.utc))
            return cron.get_next(datetime)
        elif job.schedule.at:
            return None  # 一次性任务，不再运行
```

**源码位置**：`nanobot/cron/service.py`

---

## Heartbeat：心跳检查

### 概念

Heartbeat 是一个周期性（默认 30 分钟）的"心跳"，让 LLM 检查 `HEARTBEAT.md` 文件，决定是否执行某些任务。

### 为什么用 Heartbeat 而不是硬编码？

**硬编码方式（不灵活）**：

```python
# 每小时检查服务器
if time.hour == 0:
    check_server()
```

**Heartbeat 方式（灵活）**：

```python
# 让 LLM 决定要做什么
await agent.process_direct(
    content=f"Check HEARTBEAT.md and decide what to do. Current time: {now}"
)
```

LLM 读取 `HEARTBEAT.md`，根据其中的规则和当前状态决定行动。

### HEARTBEAT.md 示例

```markdown
# Heartbeat Tasks

Check these items every heartbeat (30 minutes):

## Server Health
- If current time is on the hour, check server status
- Use `exec` tool to run `uptime` and `df -h`
- If any issue, send alert to Telegram

## Reminders
- Check if any reminders are due
- User's standing reminder: "Drink water" (every 2 hours)

## Notes
- Do NOT send messages unless necessary
- Be conservative with alerts
```

### HeartbeatService 实现

```python
class HeartbeatService:
    """心跳服务"""

    def __init__(self, agent_loop: AgentLoop, interval: int = 1800):
        self.agent = agent_loop
        self.interval = interval  # 默认 30 分钟
        self._running = False

    async def start(self) -> None:
        """启动心跳服务"""
        self._running = True

        while self._running:
            await asyncio.sleep(self.interval)

            if not self._running:
                break

            await self._heartbeat()

    async def _heartbeat(self) -> None:
        """执行心跳检查"""
        now = datetime.now().strftime("%Y-%m-%d %H:%M (%A)")

        # 构建心跳消息
        message = f"""[Heartbeat] Current time: {now}

Read HEARTBEAT.md from the workspace to check if any tasks need to be done.
Only take action if necessary. Reply briefly with what you checked."""

        try:
            await self.agent.process_direct(
                content=message,
                session_key="system:heartbeat",
            )
        except Exception as e:
            logger.error(f"Heartbeat error: {e}")

    def stop(self) -> None:
        self._running = False
```

**源码位置**：`nanobot/heartbeat/service.py`

---

## 虚拟工具调用

### 概念

Heartbeat 使用"虚拟工具调用"让 LLM 决定是否行动。

### 工作流程

```
1. Heartbeat 触发
   │
   ▼
2. 构建消息："当前时间...读取 HEARTBEAT.md..."
   │
   ▼
3. LLM 读取 HEARTBEAT.md
   │
   ▼
4. LLM 决定：
   - 无需行动 → 简短回复 "检查完毕，无需行动"
   - 需要行动 → 调用工具（exec, message 等）
   │
   ▼
5. 执行工具调用
   │
   ▼
6. 返回结果
```

### 示例交互

```
System: [Heartbeat] Current time: 2024-01-15 10:00 (Monday)
        Read HEARTBEAT.md from the workspace...

LLM: 让我读取 HEARTBEAT.md...

[调用 read_file("~/.nanobot/workspace/HEARTBEAT.md")]

LLM: 根据文件内容，现在正好是整点，需要检查服务器状态。

[调用 exec("uptime && df -h")]

LLM: 服务器状态正常：
     - uptime: 10 days
     - disk: 45% used

     无需发送警报。
```

---

## Cron vs Heartbeat

| 特性 | Cron | Heartbeat |
|------|------|-----------|
| 调度 | 精确时间 | 固定间隔 |
| 触发 | 系统自动 | 系统自动 |
| 决策 | 硬编码消息 | LLM 动态决策 |
| 灵活性 | 低 | 高 |
| 成本 | 低（每次都执行） | 中（LLM 推理） |
| 适用场景 | 固定提醒 | 条件性检查 |

### 什么时候用 Cron？

- 固定的提醒（"每天 9:00 发送天气预报"）
- 精确时间的任务（"2024-01-20 15:00 提醒开会"）

### 什么时候用 Heartbeat？

- 条件性检查（"如果服务器负载高，发送警报"）
- 复杂决策（"检查多个条件后决定是否行动"）
- 灵活规则（用户可以编辑 HEARTBEAT.md 调整行为）

---

## 系统消息路由

### Cron/Heartbeat 如何发送结果到渠道？

```python
# 系统消息：chat_id 包含原始渠道信息
msg = InboundMessage(
    channel="system",
    sender_id="cron",
    chat_id="telegram:123456789",  # "channel:chat_id" 格式
    content="Check system status",
)
```

AgentLoop 识别系统消息并路由：

```python
if msg.channel == "system":
    # 解析原始渠道
    channel, chat_id = msg.chat_id.split(":", 1)
    # 处理后返回到原始渠道
    return OutboundMessage(channel=channel, chat_id=chat_id, ...)
```

**源码位置**：`nanobot/agent/loop.py:338-354`

---

## 配置示例

### config.json

```json
{
  "cron": {
    "enabled": true,
    "timezone": "Asia/Shanghai"
  },
  "heartbeat": {
    "enabled": true,
    "interval": 1800
  }
}
```

---

## 源码索引

| 组件 | 文件位置 |
|------|---------|
| CronService | `nanobot/cron/service.py` |
| CronTool | `nanobot/agent/tools/cron.py` |
| HeartbeatService | `nanobot/heartbeat/service.py` |
| 系统消息路由 | `nanobot/agent/loop.py:338-354` |

---

下一章：[12-CONFIGURATION-PHILOSOPHY.md](./12-CONFIGURATION-PHILOSOPHY.md) — 如何在简洁和灵活之间取得平衡？
