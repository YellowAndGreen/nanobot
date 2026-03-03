# 技能架构：动态能力加载

> 核心问题：如何动态扩展 AI 的能力？

## Skill vs Tool

| 特性 | Tool | Skill |
|------|------|-------|
| 形式 | Python 类 | Markdown 文件 |
| 加载 | 代码注册 | 动态加载 |
| 内容 | 执行逻辑 | 知识/指令 |
| 扩展 | 需要改代码 | 只需写文件 |
| 示例 | read_file, exec | github, docker, weather |

**简单来说**：
- **Tool** 是 LLM 的"手脚"——执行具体操作
- **Skill** 是 LLM 的"知识"——告诉它如何使用某项能力

---

## Skill 文件格式

### 目录结构

```
~/.nanobot/workspace/skills/
├── github/
│   └── SKILL.md
├── docker/
│   └── SKILL.md
└── weather/
    └── SKILL.md

# 或内置技能
nanobot/skills/
├── github/
│   └── SKILL.md
└── ...
```

### 文件格式

```markdown
---
name: github
description: "GitHub operations via gh CLI"
metadata:
  nanobot:
    emoji: 🐙
    requires:
      bins: [gh]
    install:
      - id: brew
        kind: brew
        formula: gh
        bins: [gh]
        label: Install via brew
---

# GitHub Skill

Use the `gh` CLI to interact with GitHub.

## Common Operations

### List Repositories
```bash
gh repo list --limit 50
```

### Create Issue
```bash
gh issue create --title "Bug report" --body "Description..."
```

### Create Pull Request
```bash
gh pr create --title "Feature" --body "Description..." --base main
```

## Authentication

Check authentication status:
```bash
gh auth status
```

If not authenticated, run:
```bash
gh auth login
```
```

### YAML Frontmatter 字段

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | ✅ | 技能名称 |
| `description` | ✅ | 技能描述 |
| `metadata.nanobot.emoji` | ❌ | 显示用 emoji |
| `metadata.nanobot.requires.bins` | ❌ | 依赖的 CLI 工具 |
| `metadata.nanobot.install` | ❌ | 安装指令 |

---

## SkillsLoader

### 职责

1. 扫描技能目录
2. 解析 YAML frontmatter
3. 检查依赖可用性
4. 加载技能内容

### 实现

```python
class SkillsLoader:
    """技能加载器"""

    def __init__(self, workspace: Path):
        self.skills_dir = workspace / "skills"
        self._skills: list[SkillMeta] | None = None

    def scan_skills(self) -> list[SkillMeta]:
        """扫描所有技能"""
        if self._skills is not None:
            return self._skills

        skills = []

        # 扫描用户技能
        if self.skills_dir.exists():
            for skill_path in self.skills_dir.iterdir():
                if skill_path.is_dir():
                    skill_file = skill_path / "SKILL.md"
                    if skill_file.exists():
                        meta = self._parse_skill(skill_file)
                        if meta:
                            skills.append(meta)

        # 扫描内置技能
        builtin_dir = Path(__file__).parent.parent / "skills"
        if builtin_dir.exists():
            for skill_path in builtin_dir.iterdir():
                if skill_path.is_dir():
                    skill_file = skill_path / "SKILL.md"
                    if skill_file.exists():
                        meta = self._parse_skill(skill_file)
                        if meta and not any(s.name == meta.name for s in skills):
                            skills.append(meta)

        self._skills = skills
        return skills

    def _parse_skill(self, path: Path) -> SkillMeta | None:
        """解析技能文件"""
        content = path.read_text(encoding="utf-8")

        # 解析 YAML frontmatter
        if not content.startswith("---"):
            return None

        try:
            _, frontmatter, body = content.split("---", 2)
            data = yaml.safe_load(frontmatter)

            return SkillMeta(
                name=data.get("name", path.parent.name),
                description=data.get("description", ""),
                emoji=data.get("metadata", {}).get("nanobot", {}).get("emoji", ""),
                requires=data.get("metadata", {}).get("nanobot", {}).get("requires", {}),
                install=data.get("metadata", {}).get("nanobot", {}).get("install", []),
                path=path,
                content=body.strip(),
            )
        except Exception:
            return None
```

**源码位置**：`nanobot/agent/skills.py`

---

## 技能类型

### Always Skills（自动加载）

某些技能标记为 `always`，会自动加载到系统提示词中。

```python
def get_always_skills(self) -> list[SkillMeta]:
    """获取 always 类型的技能"""
    return [s for s in self.scan_skills() if s.load_policy == "always"]
```

### On-demand Skills（按需加载）

大多数技能是按需加载的。LLM 先看到技能摘要，需要时再读取详情。

```python
def build_skills_summary(self) -> str:
    """构建技能摘要表格"""
    skills = [s for s in self.scan_skills() if s.load_policy != "always"]

    if not skills:
        return ""

    lines = ["| Skill | Description | Available |", "|-------|-------------|-----------|"]

    for skill in skills:
        available = self._check_availability(skill)
        status = "✅" if available else "❌"
        lines.append(f"| {skill.emoji} {skill.name} | {skill.description} | {status} |")

    return "\n".join(lines)
```

### 依赖检查

```python
def _check_availability(self, skill: SkillMeta) -> bool:
    """检查技能依赖是否满足"""
    bins = skill.requires.get("bins", [])
    for bin in bins:
        if not shutil.which(bin):
            return False
    return True
```

---

## 技能加载流程

```
┌─────────────────────────────────────────────────────────────┐
│                     Skills Loading                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 启动时扫描                                              │
│     SkillsLoader.scan_skills()                              │
│     - 扫描 ~/.nanobot/workspace/skills/                     │
│     - 扫描 nanobot/skills/ (内置)                           │
│                                                             │
│  2. 构建 Always Skills                                      │
│     load_policy == "always" → 加入系统提示词                │
│                                                             │
│  3. 构建 Skills Summary                                     │
│     其他技能 → 生成摘要表格                                 │
│     - 检查依赖可用性                                        │
│     - 显示 ✅/❌ 状态                                       │
│                                                             │
│  4. LLM 看到摘要                                            │
│     "| 🐙 github | GitHub operations | ✅ |"                │
│                                                             │
│  5. LLM 决定使用某个技能                                    │
│     "让我读取 github 技能..."                               │
│                                                             │
│  6. LLM 调用 read_file                                      │
│     read_file("~/.nanobot/workspace/skills/github/SKILL.md")│
│                                                             │
│  7. LLM 获得完整技能内容                                    │
│     根据 SKILL.md 中的指令执行                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 技能示例

### GitHub Skill

```markdown
---
name: github
description: "GitHub operations via gh CLI"
metadata:
  nanobot:
    emoji: 🐙
    requires:
      bins: [gh]
---

# GitHub Skill

Interact with GitHub using the `gh` CLI.

## Prerequisites

Ensure `gh` is installed and authenticated:
```bash
gh auth status
```

## Common Operations

### Repositories
```bash
gh repo list                    # List your repos
gh repo create my-project       # Create new repo
gh repo clone owner/repo        # Clone repo
```

### Issues
```bash
gh issue list                   # List issues
gh issue create --title "..." --body "..."
gh issue close 123
```

### Pull Requests
```bash
gh pr list                      # List PRs
gh pr create --title "..." --body "..."
gh pr merge 123
```

### Workflows
```bash
gh workflow list                # List workflows
gh workflow run "CI"            # Trigger workflow
gh run watch                    # Watch run status
```
```

### Docker Skill

```markdown
---
name: docker
description: "Docker container management"
metadata:
  nanobot:
    emoji: 🐳
    requires:
      bins: [docker]
---

# Docker Skill

Manage Docker containers and images.

## Containers
```bash
docker ps                       # List running containers
docker ps -a                    # List all containers
docker logs <container>         # View logs
docker exec -it <container> sh # Execute command
```

## Images
```bash
docker images                   # List images
docker build -t name .          # Build image
docker rmi <image>              # Remove image
```

## Compose
```bash
docker compose up -d            # Start services
docker compose down             # Stop services
docker compose logs -f          # Follow logs
```
```

---

## 创建自定义技能

### 步骤

1. 创建目录：`~/.nanobot/workspace/skills/my-skill/`
2. 创建文件：`SKILL.md`
3. 编写 frontmatter 和内容

### 示例：公司内部 API 技能

```markdown
---
name: company-api
description: "Internal company API integration"
metadata:
  nanobot:
    emoji: 🏢
    requires:
      bins: [curl, jq]
---

# Company API Skill

Interact with internal company APIs.

## Authentication

All API calls require the `X-API-Key` header:
```bash
curl -H "X-API-Key: $COMPANY_API_KEY" https://api.company.com/...
```

## Endpoints

### List Projects
```bash
curl -H "X-API-Key: $COMPANY_API_KEY" \
  https://api.company.com/projects | jq
```

### Get Project Details
```bash
curl -H "X-API-Key: $COMPANY_API_KEY" \
  https://api.company.com/projects/{id} | jq
```

## Environment Variables

Set `COMPANY_API_KEY` in your shell profile.
```

---

## 源码索引

| 组件 | 文件位置 |
|------|---------|
| SkillsLoader | `nanobot/agent/skills.py` |
| 内置技能 | `nanobot/skills/*/SKILL.md` |
| 用户技能 | `~/.nanobot/workspace/skills/*/SKILL.md` |

---

下一章：[11-PROACTIVE-AGENT.md](./11-PROACTIVE-AGENT.md) — 如何让 AI 主动做事？
