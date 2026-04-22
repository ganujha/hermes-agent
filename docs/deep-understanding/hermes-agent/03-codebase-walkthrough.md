# 03 — Codebase Walkthrough: hermes-agent

## Recommended Reading Order

### The 30-Minute Path: Core Agent
1. `hermes_constants.py` — understand `get_hermes_home()` and `HERMES_HOME` pattern (5 min)
2. `tools/registry.py` (first 100 lines) — understand the tool registration contract (5 min)
3. `run_agent.py` (first 200 lines + `run_conversation()`) — understand the agent loop structure (10 min)
4. `agent/context_engine.py` — understand the compression abstraction (5 min)
5. `AGENTS.md` (Agent Loop section) — read the authoritative design notes (5 min)

### The 2-Hour Path: Full Architecture
1. **30-min path first**
2. `hermes_state.py` (first 120 lines) — understand SQLite session schema (10 min)
3. `toolsets.py` (first 100 lines) — understand `_HERMES_CORE_TOOLS` and toolset groups (10 min)
4. `agent/prompt_builder.py` (first 80 lines) — understand system prompt assembly + injection detection (10 min)
5. `agent/memory_manager.py` (first 80 lines) — understand memory orchestration + fencing (10 min)
6. `hermes_cli/commands.py` — understand the single `COMMAND_REGISTRY` design (10 min)
7. `gateway/run.py` (first 80 lines) — understand LRU agent cache + gateway lifecycle (10 min)
8. `cron/scheduler.py` (first 80 lines) — understand tick-based scheduling + file lock (10 min)
9. One representative skill: `skills/software-development/*.md` (5 min)

### The 1-Day Path: Deep Mastery
1. **2-hour path first**
2. `agent/context_compressor.py` — full read: compression algorithm, auxiliary LLM, summary framing (30 min)
3. `agent/anthropic_adapter.py` — native Anthropic API + OAuth flow (20 min)
4. `agent/credential_pool.py` + `agent/credential_sources.py` — credential lifecycle + removal contracts (20 min)
5. `tools/mcp_tool.py` — MCP client implementation (20 min)
6. `tools/delegate_tool.py` — subagent delegation, global state save/restore (15 min)
7. `cli.py` (key sections) — HermesCLI, slash command dispatch, KawaiiSpinner (20 min)
8. `agent/model_metadata.py` — model context lengths, token estimation, Ollama integration (15 min)
9. `batch_runner.py` (first 100 lines) — trajectory generation architecture (10 min)
10. `tests/` — browse test directory structure to understand coverage gaps (20 min)
11. `hermes_cli/skin_engine.py` — theme/skin system as a pattern example (10 min)
12. `acp_adapter/server.py` — ACP integration pattern (10 min)
13. `.plans/` directory — read in-progress design plans (10 min)

---

## Top 20 Most Important Files

| # | File | Why It Matters |
|---|------|---------------|
| 1 | `run_agent.py` | **The entire agent loop.** Every architectural decision (loop termination, tool parallel execution, compression trigger, provider routing, token accounting) lives here. 12,053 lines — the true center of gravity. |
| 2 | `tools/registry.py` | **The tool contract.** Defines how tools self-register, how discovery works (AST scan), and how dispatch is handled. Understanding this unlocks how all 50+ tools relate. |
| 3 | `agent/context_engine.py` | **The extensibility contract for context management.** A clean 120-line abstract base that defines the entire compression lifecycle. Read this before reading the concrete implementation. |
| 4 | `hermes_constants.py` | **The path resolution contract.** `get_hermes_home()` is called 119+ times. Profile isolation lives here. Any developer adding state storage must understand this file. |
| 5 | `hermes_state.py` | **Session persistence.** SQLite schema, FTS5 setup, WAL mode, write contention handling. Understanding this reveals what "session" means in Hermes. |
| 6 | `agent/context_compressor.py` | **The default context engine.** Auxiliary LLM-based summarization, tool result pruning, compression framing, token budgeting. The most complex single piece of logic. |
| 7 | `agent/prompt_builder.py` | **System prompt assembly.** All the pieces that make the agent's identity: persona, memory guidance, skills guidance, context file injection, prompt injection detection. |
| 8 | `agent/memory_manager.py` | **Memory orchestration.** The fence-based injection model, the single-external-provider constraint, and how built-in + external memory coexist. |
| 9 | `toolsets.py` | **Tool groups.** `_HERMES_CORE_TOOLS` defines what every platform gets. Individual toolsets gate optional capabilities. |
| 10 | `hermes_cli/commands.py` | **Slash command registry.** Single source of truth for all `/commands`. The pattern of deriving all downstream consumers (CLI, gateway, Telegram menu, Slack map, autocomplete) from one registry is worth studying. |
| 11 | `gateway/run.py` | **Multi-platform gateway.** LRU agent cache, session lifecycle, platform adapter management, cron thread. Shows how the agent loop is turned into a long-lived service. |
| 12 | `model_tools.py` | **Tool discovery and dispatch.** `get_tool_definitions()` + `handle_function_call()` + toolset resolution. The bridge between agent loop and tool layer. |
| 13 | `agent/anthropic_adapter.py` | **Provider-specific API handling.** OAuth flows, native Anthropic Messages API, cache control. Template for understanding other provider adapters. |
| 14 | `agent/credential_pool.py` | **Credential management.** Multi-source credential resolution, rotation, removal contracts. Shows how Hermes handles 20+ provider auth patterns. |
| 15 | `cron/scheduler.py` | **Autonomous execution.** Tick-based scheduling, file locking, job delivery to messaging platforms. Enables unattended agent operation. |
| 16 | `tools/delegate_tool.py` | **Subagent delegation.** How the system spawns isolated child agents, passes context, and handles the process-global state issue. |
| 17 | `cli.py` | **Interactive REPL.** prompt_toolkit integration, KawaiiSpinner, slash command dispatch, session management from the user perspective. |
| 18 | `hermes_cli/config.py` | **Configuration contract.** `DEFAULT_CONFIG` defines all configurable options + their defaults. Config migration versioning. |
| 19 | `agent/error_classifier.py` | **Failure handling.** How API errors are classified and whether they trigger retry, failover, compression, or abort. |
| 20 | `AGENTS.md` | **Developer contract.** Contains architectural invariants, known pitfalls, development patterns, and the authoritative diagram of system components. Unusually high signal-to-noise for developer docs. |

---

## File-by-File Reading Notes

### `hermes_constants.py`
Start here. `get_hermes_home()` reads `HERMES_HOME` env var. Profile system works by setting this env var before any imports. Every file that stores state must call this function — never hardcode `~/.hermes`. This file has zero dependencies, making it safe to import anywhere.

### `tools/registry.py`
The circular-import-safe foundation of the tool layer. The `discover_builtin_tools()` function is architecturally interesting: it uses `ast.parse()` to find `registry.register()` calls at module body level (not inside functions). This avoids a manual import list while also catching accidental registrations inside functions. The `ToolEntry` dataclass holds the handler, schema, check function, and required env vars for each tool.

### `run_agent.py`
Do not try to read this linearly. Instead, read:
- Lines 1–200: imports, `_SafeWriter`, proxy detection
- Lines 690–800: `AIAgent.__init__` — understand constructor parameters
- `run_conversation()` method: the loop body — this is the architectural core
- `_build_system_prompt()` method: how context is assembled before each API call
- Provider routing section (lines ~1094–1274): how `api_mode` selects the right client
- Compression trigger section: how `context_engine.should_compress()` gates compression

### `agent/context_engine.py`
120 lines, pure abstract base. Read in full. Pay attention to the lifecycle comments: `on_session_start`, `on_session_end`, `on_session_reset` are called at different granularities. The base class constants (`threshold_percent=0.75`, `protect_first_n=3`) are defaults that concrete engines can override.

### `agent/context_compressor.py`
Read the summary prefix constant first (around line 38–50). The `[CONTEXT COMPACTION — REFERENCE ONLY]` framing with explicit "do NOT answer questions from this summary" instruction is a key safety pattern. Then read `should_compress()` (simple threshold check) and `compress()` (the full pipeline: prune → summarize → return).

### `hermes_state.py`
Read `SCHEMA_SQL` to understand what data is stored. Read the FTS triggers — they maintain the search index automatically. Look for the jittered backoff write logic. The schema's `parent_session_id` column enables following compression chains.

### `hermes_cli/commands.py`
Read the `CommandDef` dataclass and `COMMAND_REGISTRY` list. Then trace how `resolve_command()` is used in `cli.py` and `gateway/run.py` — same function, same registry, two consumers. The `gateway_config_gate` field is an elegant pattern: a command can be CLI-only by default but gateway-enabled if a config key is set.

### `gateway/run.py`
Read the constants at the top (`_AGENT_CACHE_MAX_SIZE=128`, `_AGENT_CACHE_IDLE_TTL_SECS=3600.0`). Then read `GatewayRunner.__init__`. The LRU cache (`OrderedDict`) + idle TTL eviction shows two complementary eviction strategies: hard cap (128) and time-based (1h). Each platform adapter is started independently; a failure in one doesn't kill others.

---

## "Read This First / Second / Third" Guidance

**First** (orientation):
- `AGENTS.md` — the developer's own architectural notes
- `hermes_constants.py` — the universal path contract

**Second** (core mechanics):
- `tools/registry.py` — tool system foundation
- `agent/context_engine.py` — context management abstraction
- `run_agent.py` (`AIAgent.__init__` + `run_conversation()`) — the loop

**Third** (production concerns):
- `hermes_state.py` — persistence
- `gateway/run.py` — service mode
- `agent/context_compressor.py` — what happens when context fills up
- `agent/memory_manager.py` — what persists across sessions
