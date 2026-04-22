# 00 — System Overview: hermes-agent

## BLUF

Hermes Agent is a self-hosted, self-improving AI agent framework built by Nous Research that runs a tool-calling conversation loop against any OpenAI-compatible LLM, persists session memory across conversations, autonomously creates and improves domain-specific “skills,” schedules unattended tasks via a built-in cron engine, and exposes the same agent across a dozen messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal, and more) via a single gateway process — all deployable on a $5 VPS or serverless infrastructure.

---

## Problem the Repo is Solving

Most LLM chat tools are stateless, single-provider, laptop-bound, and have no mechanism for the agent to improve over time. Hermes solves five distinct problems simultaneously:

1. **Statelessness** — agents forget everything between sessions. Hermes maintains `MEMORY.md`, `USER.md`, SQLite session history with FTS5 search, and optional external memory backends (Honcho, Mem0).
2. **Single-provider lock-in** — switching from OpenAI to Claude requires code changes. Hermes abstracts providers behind a unified OpenAI-wire interface with native adapters for Anthropic, Bedrock, and Gemini.
3. **Interface fragmentation** — users want to chat from Telegram while the agent runs on a server. The gateway process bridges 16+ messaging platforms to the same core agent.
4. **Static capability** — agents can’t learn new procedures. The Skills system creates and refines procedural knowledge from completed tasks, stored as `.md` files and injected at runtime.
5. **Always-on limitation** — most agents require a human initiator. The cron scheduler enables fully autonomous, unattended task execution with platform delivery.

---

## Who the User/Operator Is

- **Individual power users**: Developers and researchers who want a persistent, cloud-hosted personal AI assistant accessible from any device.
- **Team operators**: Teams that deploy Hermes as a shared Telegram/Slack bot for internal automation.
- **ML researchers**: Nous Research using the batch runner and RL environments to generate training trajectories for next-generation tool-calling models.
- **Developers extending the system**: Contributors adding new tools, skills, memory providers, or platform adapters.

---

## Major Subsystems

| Subsystem | Entry Point | Purpose |
|-----------|-------------|--------|
| **Agent Loop** | `run_agent.py:AIAgent` | Core LLM ↔ tool conversation loop |
| **CLI** | `hermes_cli/main.py` + `cli.py` | Interactive terminal REPL |
| **TUI** | `ui-tui/` + `tui_gateway/` | Full-screen Ink terminal UI |
| **Gateway** | `gateway/run.py` | Multi-platform messaging bridge |
| **Skills** | `skills/` + `tools/skill_manager_tool.py` | Procedural memory, domain knowledge |
| **Memory** | `tools/memory_tool.py` + `plugins/memory/` | Cross-session knowledge persistence |
| **Context Engine** | `agent/context_engine.py` + `agent/context_compressor.py` | Context window management |
| **Cron Scheduler** | `cron/scheduler.py` + `cron/jobs.py` | Autonomous scheduled execution |
| **Session Store** | `hermes_state.py` | SQLite conversation history |
| **Tool Registry** | `tools/registry.py` + `model_tools.py` | Tool discovery, dispatch, toolsets |
| **Credential System** | `agent/credential_pool.py` + `agent/credential_sources.py` | Multi-provider auth |
| **ACP Adapter** | `acp_adapter/` | IDE integration (VS Code, JetBrains, Zed) |
| **Web Dashboard** | `web/` | React admin UI |
| **Batch Runner** | `batch_runner.py` | Parallel trajectory generation for RL |

---

## External Dependencies / Integrations

**LLM Providers**: Anthropic (Claude), OpenAI, AWS Bedrock, Google Gemini, xAI (Grok), Nous Portal, OpenRouter (200+ models), NVIDIA NIM, HuggingFace, Kimi/Moonshot, Alibaba Qwen, MiniMax, GLM/z.ai, GitHub Copilot, any OpenAI-compatible endpoint.

**Tool-Level Integrations**: Exa (web search), Firecrawl (web extraction), Parallel.ai (web), fal.ai (image generation), ElevenLabs / Edge TTS (text-to-speech), Browserbase (browser automation), Modal (serverless compute), Daytona (development environments).

**Memory Providers**: Honcho (dialectic user modeling), Hindsight, Mem0 — all via plugin architecture.

**Messaging Platforms**: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, Home Assistant, DingTalk, Feishu, WeChat (WeChat Official Account + WeCom), SMS, Email, Webhooks, BlueBubbles, QQBot.

**Infrastructure**: SQLite (state), Playwright (browser), MCP protocol (tool extension), Nix (reproducible builds), Docker (containerization), GitHub Actions (CI).

---

## What Makes This Interesting from an Agentic-Systems Perspective

1. **Closed learning loop**: The agent can write new skills after complex tasks, retrieve them in future sessions, improve them during use, and search past sessions. This is a rare self-improvement architecture in open-source agents.

2. **Pluggable context engine**: The `ContextEngine` abstract base class lets you replace the built-in LLM summarizer with entirely different compression strategies (DAG-based, vector-indexed, etc.) by dropping a plugin into `plugins/context_engine/`.

3. **Provider abstraction with caching invariants**: Prompt caching is a first-class design constraint — the system explicitly warns developers against breaking cache validity mid-conversation, and the architecture enforces this by never modifying past context except during compression.

4. **Multi-platform identity**: The same agent handles CLI, 16+ messaging platforms, and IDE integration. The session model and command registry are shared; platform adapters are thin.

5. **Delegation and parallelism**: `delegate_task` spawns fully isolated child `AIAgent` instances, enabling hierarchical multi-agent workflows within a single process.

6. **Research-grade trajectory infrastructure**: `batch_runner.py` and `trajectory_compressor.py` suggest the architecture is being used to generate training data for future models — meta-use of the agent to improve agents.
