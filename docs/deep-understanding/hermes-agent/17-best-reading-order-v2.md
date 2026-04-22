# 17 — Best Reading Order v2: hermes-agent

This is an improved version of the first-pass reading order, correcting over-emphasis and under-emphasis based on second-pass analysis.

## What Changed from v1

**Over-emphasized in v1:**
- AGENTS.md before code — AGENTS.md is mostly correct but has subtle inaccuracies (skill injection, memory sync). Reading it first anchors you to wrong mental models before you see the code.
- Full `hermes_cli/commands.py` early — COMMAND_REGISTRY is important but not the first thing to understand; it's a product of the architecture, not the foundation.

**Under-emphasized in v1:**
- `model_tools.py` — the `_last_resolved_tool_names` global and `get_tool_definitions()` are central to understanding how tools get presented to the LLM, and the race condition is one of the most important latent bugs.
- `agent/memory_manager.py` — memory sync is blocking (not fire-and-forget); this changes how you reason about turn latency.
- `hermes_cli/config.py` vs `cli.py` — the config divergence is an ongoing maintenance risk, worth reading both early.

---

## Day 1 — The Core Loop (2–3 hours)

**Goal**: Understand what an "agent turn" looks like from user input to response.

1. **`hermes_constants.py`** (whole file, ~100 lines)
   - First, last, always: `get_hermes_home()` is the single most-called function in the repo
   - Understand: profile isolation via `HERMES_HOME` env var

2. **`run_agent.py`** — read these sections only:
   - Constructor: `AIAgent.__init__()` (lines ~700–850) — what an agent is at init time
   - Main loop: `run_conversation()` (lines 8464–9000) — the while loop, LLM call, tool dispatch
   - Parallel dispatch: `_should_parallelize_tool_batch()` (lines 299–340) — parallel vs sequential decision
   - Skip: streaming code, all helper methods until you need them

3. **`tools/registry.py`** (first 120 lines)
   - `ToolRegistry`, `ToolEntry`, `register()`, `discover_builtin_tools()`
   - The AST scan approach — understand why it doesn't import modules

4. **`model_tools.py`** — read:
   - `_last_resolved_tool_names` definition (line 159)
   - `get_tool_definitions()` (lines 330–360) — how registry becomes OpenAI format
   - The global write at lines 337–338 — this is the race condition site
   - Comment: this is the most important "hidden coupling" in the codebase

---

## Day 2 — State and Memory (2–3 hours)

**Goal**: Understand how the agent persists knowledge across turns and sessions.

5. **`hermes_state.py`** — read:
   - `SCHEMA_SQL` constant (lines 36–100) — the complete DB schema: sessions, messages, parent_session_id, FTS5
   - `SessionDB.__init__()` — WAL mode setup
   - `insert_message()` and `update_session()` — what gets written per turn
   - `_write_with_retry()` — jittered backoff for write contention
   - Skip: FTS search methods until Day 4

6. **`agent/memory_manager.py`** (whole file)
   - `MemoryManager.__init__()` — provider setup
   - `prefetch_all()` — memory injection before each turn
   - `build_memory_context_block()` — the `<memory-context>` fencing pattern
   - `sync_all()` — **read carefully**: this is BLOCKING (not fire-and-forget). Lines 210–219.
   - `MemoryProvider(ABC)` — the interface for external providers

7. **`agent/context_engine.py`** (whole file)
   - `ContextEngine(ABC)` lifecycle: session_start → token_tracking → should_compress → compress
   - `threshold_percent: float = 0.75` (class default)
   - Connect: when `should_compress()` fires, what happens next?

8. **`agent/context_compressor.py`** (skim)
   - How the auxiliary LLM summarizes the middle of a conversation
   - `parent_session_id` usage — how compression creates a session chain

---

## Day 3 — Configuration, Interfaces, and Gateway (2–3 hours)

**Goal**: Understand how configuration flows and how users interact with the agent.

9. **`hermes_cli/config.py`** — read:
   - `DEFAULT_CONFIG` constant (first 100 lines of the dict)
   - `load_config()` function (lines 3039–3065)
   - `_deep_merge()` and `_normalize_max_turns_config()`

10. **`cli.py`** — read:
    - `load_cli_config()` (lines 268–387) — compare to load_config() above; note the differences
    - `HermesCLI.chat()` — the REPL loop: how user input becomes run_conversation()
    - `HermesCLI._handle_slash_command()` — how slash commands are dispatched

11. **`hermes_cli/commands.py`** (skim)
    - `COMMAND_REGISTRY` — the list of `CommandDef` objects
    - `gateway_config_gate` field — how gateway enables/disables commands via config

12. **`gateway/run.py`** — read these sections:
    - `_AGENT_CACHE_MAX_SIZE`, `_AGENT_CACHE_IDLE_TTL_SECS` constants
    - `GatewayRunner.__init__()` — agent cache setup (OrderedDict)
    - `get_or_create_agent()` — lazy agent creation, LRU logic
    - `_enforce_agent_cache_cap()` — eviction logic (lines 8738–8812)
    - `_start_cron_ticker()` and thread start (lines 10724–11038)

---

## Day 4 — Delegation, Tools, and Advanced Patterns (2–3 hours)

**Goal**: Understand multi-agent delegation, the prompt injection defense, and advanced tool patterns.

13. **`tools/delegate_tool.py`** — read:
    - `delegate_task()` function signature and docstring
    - Save/restore of `_last_resolved_tool_names` (lines ~100–113, 1155–1187)
    - How a child `AIAgent` is spawned and run to completion
    - Return value path: child response → parent tool result

14. **`agent/prompt_builder.py`** — read:
    - `_CONTEXT_THREAT_PATTERNS` (lines 36–47) — the 10 injection patterns
    - `_scan_for_injection()` usage — where it's called
    - `PLATFORM_HINTS` dict — how platform-specific behavior is injected

15. **`cron/scheduler.py`** (skim)
    - `tick()` function — the 60-second tick
    - `fcntl` file lock — single-process guarantee
    - Job execution model — fresh `AIAgent` per cron job

16. **`agent/skill_commands.py`** — read:
    - `build_skill_invocation_message()` (lines 429–465) — user message injection
    - `build_preloaded_skills_prompt()` — note this goes to system prompt (contrast above)
    - The activation note format: `[SYSTEM: The user has invoked...]`

---

## Day 5 — Tests, Production Gaps, and the Full Picture (1–2 hours)

**Goal**: Understand quality, test coverage, and what's missing.

17. **`tests/conftest.py`** (whole file)
    - `_isolate_hermes_home` autouse fixture — this is how all tests stay isolated
    - Understand: if a test writes to `~/.hermes/` directly, it breaks

18. **Read `14-architecture-critique.md`** (this learning pack)
    - Weaknesses and hidden coupling sections
    - Failure modes section — understand what can go wrong in production

19. **Run the test suite:**
    ```bash
    ./scripts/run_tests.sh -v
    ```
    - Look for which areas have thin coverage
    - Check integration tests — what's marked as skipped?

20. **Browse `tests/` subdirectory structure:**
    ```bash
    find tests/ -name "*.py" | head -30
    ```
    - Note what's well-covered (agent, tools, cron)
    - Note what's sparse (gateway, ACP adapter, web)

---

## Quick Reference: File → Concept Map

| If you want to understand... | Read... |
|-----------------------------|--------|
| How turns work | `run_agent.py:run_conversation()` |
| Tool registration | `tools/registry.py:ToolRegistry` |
| Tool → LLM format | `model_tools.py:get_tool_definitions()` |
| Memory injection | `agent/memory_manager.py:prefetch_all()` |
| Memory security | `agent/memory_manager.py:build_memory_context_block()` |
| Context compression | `agent/context_engine.py` + `context_compressor.py` |
| Config loading | `hermes_cli/config.py:load_config()` |
| CLI config (different!) | `cli.py:load_cli_config()` |
| Session schema | `hermes_state.py:SCHEMA_SQL` |
| Gateway agent cache | `gateway/run.py:GatewayRunner` |
| Delegation | `tools/delegate_tool.py` |
| Parallel tool safety | `run_agent.py:_should_parallelize_tool_batch()` |
| Concurrency race condition | `model_tools.py:_last_resolved_tool_names` |
| Prompt injection defense | `agent/prompt_builder.py:_CONTEXT_THREAT_PATTERNS` |
| Cron execution | `cron/scheduler.py:tick()` |
| Skill invocation | `agent/skill_commands.py:build_skill_invocation_message()` |
| Profile isolation | `hermes_constants.py:get_hermes_home()` |
| Test isolation | `tests/conftest.py:_isolate_hermes_home` |
