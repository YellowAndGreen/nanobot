# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight personal AI assistant framework (~4,000 lines of core agent code). It supports multiple chat channels (Telegram, Discord, Slack, WhatsApp, Feishu, etc.) and LLM providers (OpenRouter, Anthropic, OpenAI, etc.).

## Commands

```bash
# Install (development)
pip install -e .

# Install with optional dependencies
pip install -e ".[dev]"      # pytest, ruff
pip install -e ".[matrix]"   # Matrix/Element channel

# Lint
ruff check nanobot/

# Run all tests
pytest

# Run specific test file
pytest tests/test_context_prompt_cache.py

# Run specific test with verbose output
pytest tests/test_commands.py -v

# Count core agent lines (excludes channels/, cli/, providers/)
bash core_agent_lines.sh
```

## Architecture

### Core Components

**Agent Loop** (`nanobot/agent/loop.py`)
- Central processing engine that receives messages, builds context, calls LLM, executes tools
- Uses `ToolRegistry` for built-in tools (exec, read/write/edit files, web fetch/search, cron, spawn)
- Manages sessions via `SessionManager` and subagents via `SubagentManager`

**Context Builder** (`nanobot/agent/context.py`)
- Assembles system prompts from templates (SOUL.md, USER.md, TOOLS.md, etc.)
- Loads memory from `memory/MEMORY.md` and skills from `skills/*/SKILL.md`
- Templates live in `nanobot/templates/` and are synced to `~/.nanobot/workspace/` on startup

**Provider Registry** (`nanobot/providers/registry.py`)
- Single source of truth for LLM provider metadata
- Adding a new provider: (1) add `ProviderSpec` to `PROVIDERS` tuple, (2) add field to `ProvidersConfig` in `config/schema.py`
- Supports gateways (OpenRouter), direct providers (custom), and OAuth providers (Codex)

**Message Bus** (`nanobot/bus/`)
- `events.py`: `InboundMessage` and `OutboundMessage` dataclasses
- `queue.py`: Async `MessageBus` for routing messages between channels and agent

**Channels** (`nanobot/channels/`)
- Each channel (telegram, discord, slack, etc.) extends `BaseChannel`
- Channels convert platform-specific events to/from `InboundMessage`/`OutboundMessage`
- Manager handles channel lifecycle and message routing

**Session Management** (`nanobot/session/manager.py`)
- Sessions keyed by `channel:chat_id`
- Messages stored in JSONL format (append-only for LLM cache efficiency)
- Consolidation writes summaries to MEMORY.md/HISTORY.md without modifying message history

**Skills System** (`nanobot/agent/skills.py`, `nanobot/skills/`)
- Skills are markdown files with YAML frontmatter (name, description, metadata)
- `metadata.nanobot.requires.bins` specifies required CLI tools
- `metadata.nanobot.install` provides installation instructions
- Skills can be `always` (auto-loaded) or on-demand (agent reads when needed)

**Cron & Heartbeat** (`nanobot/cron/`, `nanobot/heartbeat/`)
- Cron: Scheduled jobs with `at`, `every`, or cron expression schedules
- Heartbeat: Every 30 minutes, checks `HEARTBEAT.md` for periodic tasks via virtual tool call

### Data Flow

```
Channel (e.g., Telegram)
    ↓ InboundMessage
MessageBus
    ↓
AgentLoop.process()
    → ContextBuilder.build_system_prompt()
    → Session.get_history()
    → LLM Provider
    → ToolRegistry.execute()
    ↓ OutboundMessage
MessageBus
    ↓
Channel.send()
```

### Configuration

Config file: `~/.nanobot/config.json` (created by `nanobot onboard`)

Schema defined in `nanobot/config/schema.py` using Pydantic. Supports both camelCase and snake_case keys.

### Built-in Tools (`nanobot/agent/tools/`)

| Tool | Purpose |
|------|---------|
| `exec` | Shell command execution with safety limits |
| `read_file` / `write_file` / `edit_file` / `list_dir` | Filesystem operations |
| `web_fetch` / `web_search` | Web access (Brave Search API optional) |
| `cron` | Scheduled reminders |
| `spawn` | Background subagent tasks |
| `message` | Send messages proactively |
| `mcp` | Model Context Protocol tool integration |

## Key Patterns

### Adding a Provider

1. Add `ProviderSpec` to `PROVIDERS` in `nanobot/providers/registry.py`
2. Add field to `ProvidersConfig` in `nanobot/config/schema.py`

Key `ProviderSpec` fields:
- `keywords`: Model name patterns for auto-matching
- `litellm_prefix`: Auto-prefix for LiteLLM (e.g., "openrouter")
- `is_gateway`: Can route any model
- `is_direct`: Bypass LiteLLM entirely (for custom OpenAI-compatible endpoints)
- `is_oauth`: Uses OAuth flow instead of API key

### Skill Format

```markdown
---
name: skill-name
description: "What this skill does"
metadata:
  nanobot:
    emoji: 📦
    requires:
      bins: [required-cli-tool]
    install:
      - id: brew
        kind: brew
        formula: formula-name
        bins: [cli-tool]
        label: Install via brew
---
# Skill instructions here
```

### Tool Implementation

Tools extend `BaseTool` in `nanobot/agent/tools/base.py`. Register in `ToolRegistry` within `AgentLoop.__init__()`.

## Testing

Tests use pytest with async support (`pytest-asyncio`). Test files in `tests/` follow pattern `test_*.py`.

The `conftest.py` pattern is not used; fixtures are created inline in test files.
