# 11 — Component Decomposition: hermes-agent

## Bottom-Up Decomposition

---

## Layer 1: Infrastructure / Runtime Layer

**What lives here**: OS primitives, filesystem access, environment variables, SQLite, network sockets, process management.

| Component | Custom vs Third-Party | Role |
|-----------|----------------------|------|
| Python stdlib (os, pathlib, threading, sqlite3, ast, asyncio) | Third-party (stdlib) | All core I/O, path resolution, concurrency, DB |
| `HERMES_HOME` path resolution | **Custom** (`hermes_constants.py`) | Profile-safe path abstraction |
| SQLite WAL + FTS5 | Third-party (sqlite3 stdlib) | Session persistence + full-text search |
| `fcntl` file locks | Third-party (stdlib) | Cron tick mutex |
| `concurrent.futures.ThreadPoolExecutor` | Third-party (stdlib) | Tool parallelism, batch execution |
| `_SafeWriter` | **Custom** (`run_agent.py`) | Broken pipe resilience for daemon mode |
| `atomic_json_write` | **Custom** (`utils.py`) | Race-safe JSON file writes |

**Architectural core**: `hermes_constants.py:get_hermes_home()` — zero-dependency, called 119+ times.

**Swappable**: SQLite could be replaced with Postgres behind the `SessionDB` class interface. The profile isolation mechanism would survive unchanged.

---

## Layer 2: Framework Layer

**What lives here**: HTTP clients, LLM SDKs, TUI libraries, web frameworks, tool SDKs.

| Component | Custom vs Third-Party | Role |
|-----------|----------------------|------|
| `openai` SDK (≥2.21.0) | Third-party | Primary LLM wire protocol (all non-native providers) |
| `anthropic` SDK (≥0.39.0) | Third-party | Native Anthropic Messages API + caching |
| `boto3` | Third-party | AWS Bedrock |
| `httpx[socks]` | Third-party | Async HTTP client for providers |
| `requests` | Third-party | Sync HTTP for some tool calls |
| `tenacity` | Third-party | Retry logic with backoff |
| `pydantic` | Third-party | Data validation (config, schemas) |
| `prompt_toolkit` | Third-party | Interactive REPL with autocomplete |
| `rich` | Third-party | Terminal formatting |
| `fire` | Third-party | CLI argument parsing |
| `fastapi` + `uvicorn` | Third-party | ACP adapter server, web dashboard backend |
| `jinja2` | Third-party | Template rendering (system prompts, config) |
| `pyyaml` | Third-party | Config file parsing |
| `croniter` | Third-party | Cron expression parsing |
| `mcp` SDK | Third-party | MCP protocol client |
| Ink (React for terminal) | Third-party | Full-screen TUI |
| Vite + React | Third-party | Web dashboard frontend |

**Architectural core**: `openai` SDK — the entire provider abstraction is built on OpenAI-wire format. Replacing this would require rewriting the message format layer.

**Swappable**: `anthropic` SDK could be replaced with a custom HTTP client. `prompt_toolkit` is only for the classic CLI (bypassed by TUI mode). `fastapi` only for ACP/web features.

---

## Layer 3: Application / Service Layer

**What lives here**: The core agent, tool implementations, platform adapters.

| Component | Custom vs Third-Party | Role |
|-----------|----------------------|------|
| `AIAgent` (`run_agent.py`) | **Custom** | Core agent loop, LLM routing, tool orchestration |
| `ContextCompressor` (`agent/context_compressor.py`) | **Custom** | Built-in context compression |
| `SessionDB` (`hermes_state.py`) | **Custom** | SQLite session store |
| Tool implementations (`tools/*.py`) | **Custom** (mostly) + SDK wrappers | 50+ tools for shell, file, web, browser, etc. |
| `HermesCLI` (`cli.py`) | **Custom** | Interactive REPL |
| `GatewayRunner` (`gateway/run.py`) | **Custom** | Multi-platform messaging gateway |
| Platform adapters (`gateway/platforms/`) | **Custom** + SDK wrappers | Per-platform message handling |
| Cron scheduler (`cron/scheduler.py`) | **Custom** | Tick-based autonomous execution |
| `MemoryManager` (`agent/memory_manager.py`) | **Custom** | Memory provider orchestration |
| `CredentialPool` (`agent/credential_pool.py`) | **Custom** | Multi-source credential management |
| `ToolRegistry` (`tools/registry.py`) | **Custom** | Self-registering tool catalog |

**Architectural core**: `AIAgent` in `run_agent.py`. Everything else serves or extends it.

---

## Layer 4: Orchestration Layer

**What lives here**: Multi-agent coordination, batch processing, RL training pipelines.

| Component | Custom vs Third-Party | Role |
|-----------|----------------------|------|
| `delegate_task` tool (`tools/delegate_tool.py`) | **Custom** | Spawn isolated child AIAgent instances |
| `batch_runner.py` | **Custom** | Parallel multi-agent trajectory generation |
| `rl_cli.py` + `environments/` | **Custom** | RL training orchestration |
| Atropos lib (`tinker-atropos/` submodule) | Third-party (Nous Research) | RL environment framework |
| Tinker lib | Third-party (third-party submodule) | RL tooling |
| `mixture_of_agents_tool.py` | **Custom** | MoA coordination (details to verify) |

**How components interact**: `batch_runner.py` creates `ThreadPoolExecutor` → spawns N `AIAgent` instances → each runs `run_conversation()` independently → saves trajectories. `delegate_task` creates a child `AIAgent` inline during a parent's tool execution phase.

**Swappable**: The RL layer is entirely optional (separate `rl` extra). Delegation pattern is a tool — any tool that spawns agents works the same way.

---

## Layer 5: Memory / Context Layer

**What lives here**: Everything that manages what the agent knows and remembers.

| Component | Custom vs Third-Party | Role |
|-----------|----------------------|------|
| `MEMORY.md` / `USER.md` | **Custom** (flat files) | Semantic long-term memory |
| `state.db` FTS5 | **Custom** (SQLite FTS5) | Episodic recall by keyword |
| `ContextEngine` ABC | **Custom** | Context window management abstraction |
| `ContextCompressor` | **Custom** | LLM-based context summarization |
| `MemoryManager` | **Custom** | Memory tier orchestration |
| `BuiltinMemoryProvider` | **Custom** | MEMORY.md / USER.md as a provider |
| Honcho provider | Third-party + **custom** adapter | Dialectic user modeling |
| Hindsight provider | Third-party + **custom** adapter | Memory extraction |
| Mem0 provider | Third-party + **custom** adapter | Vector-based retrieval |
| `<memory-context>` fencing | **Custom** (in-process) | Injection prevention for recalled context |
| `session_search` tool | **Custom** | FTS5 search exposed as an agent tool |

**How components interact**:
```
Before turn: MemoryManager.prefetch_all() → external provider recall → build_memory_context_block() → inject into user message
After turn:  MemoryManager.sync_turn() → external provider async write
               built-in: agent explicitly calls memory tool → writes to MEMORY.md
At context limit: ContextEngine.compress() → auxiliary LLM → compressed message list
```

**Architectural core**: The `<memory-context>` fencing pattern. Everything else — providers, storage backends — is swappable. The fencing is the invariant that keeps recalled memory safe.

---

## Layer 6: Interface Layer

**What lives here**: All user-facing interfaces and their adapters.

| Component | Custom vs Third-Party | Role |
|-----------|----------------------|------|
| Classic CLI (`cli.py`) | **Custom** + prompt_toolkit | Interactive REPL |
| Ink TUI (`ui-tui/`) | **Custom** + Ink/React | Full-screen terminal UI |
| TUI gateway (`tui_gateway/`) | **Custom** | Python JSON-RPC backend for TUI |
| Messaging gateway (`gateway/`) | **Custom** | Multi-platform bot server |
| ACP adapter (`acp_adapter/`) | **Custom** | IDE integration server |
| Web dashboard (`web/`) | **Custom** + React/Vite | Admin UI |
| `hermes` shell script | **Custom** | Entry point with venv detection |
| Slash command registry (`hermes_cli/commands.py`) | **Custom** | Single source of truth for all commands |
| Skin/theme engine (`hermes_cli/skin_engine.py`) | **Custom** | Data-driven visual customization |

**How components interact**: All interfaces instantiate `AIAgent` and call `run_conversation()`. The gateway additionally manages session state and platform-specific message formatting. The TUI bridges the gap via JSON-RPC (Ink can't directly call Python APIs).

**Architectural core**: The `COMMAND_REGISTRY` — all interfaces share the same command definitions, ensuring behavioral consistency.

---

## Layer 7: Cross-Cutting Concerns

| Concern | Implementation | Files |
|---------|---------------|-------|
| **Configuration** | Two loaders (known issue), YAML + dotenv | `hermes_cli/config.py`, `cli.py:load_cli_config()` |
| **Authentication** | `CredentialPool` + per-source removal contracts | `agent/credential_pool.py`, `agent/credential_sources.py` |
| **Logging** | Python `logging` module, per-module loggers | `hermes_logging.py`, every module |
| **Error handling** | `error_classifier.py` + tenacity retries | `agent/error_classifier.py`, `agent/retry_utils.py` |
| **Security** | Prompt injection detection, DM pairing, approval patterns | `agent/prompt_builder.py`, `gateway/pairing.py`, `tools/approval.py` |
| **Testing** | pytest + xdist, `_isolate_hermes_home` fixture | `tests/conftest.py` |
| **Profile isolation** | `HERMES_HOME` env var + `get_hermes_home()` | `hermes_constants.py` |
| **Cost tracking** | Token counting + pricing estimates per session | `agent/usage_pricing.py`, `hermes_state.py` |
| **Interruption** | `set_interrupt()` + `IterationBudget` | `tools/interrupt.py`, `run_agent.py` |
| **Platform hints** | `PLATFORM_HINTS` dict keyed by platform string | `agent/prompt_builder.py` |

---

## What Is the Architectural Core vs Replaceable

### Architectural Core (high coupling, cannot replace without major refactor)
- `run_agent.py:AIAgent` — everything depends on this
- `tools/registry.py:ToolRegistry` — all tools, model_tools, and agents depend on this
- `hermes_constants.py:get_hermes_home()` — all state access depends on this
- OpenAI message format (`{"role": ..., "content": ...}`) — entire message pipeline assumes this format
- `hermes_cli/commands.py:COMMAND_REGISTRY` — all interfaces derive from this

### Likely Replaceable (low coupling, clean interfaces)
- `agent/context_compressor.py` — swap via `ContextEngine` ABC
- `agent/memory_manager.py` providers — swap via `MemoryProvider` ABC
- `hermes_state.py:SessionDB` — abstract behind interface for Postgres backend
- `agent/anthropic_adapter.py`, `bedrock_adapter.py`, etc. — provider-specific, switchable
- Skin system (`hermes_cli/skin_engine.py`) — pure data layer
- Web dashboard (`web/`) — decoupled frontend
- Platform adapters (`gateway/platforms/`) — each implements common interface
- `agent/context_compressor.py:auxiliary_client` — the summarization model is configurable
