# Phase 2 Study Pack: hermes-agent

Designed for active recall. Use this after reading the documentation — test yourself before interviews or architecture reviews.

---

## Top 20 Concepts to Know Cold

These are the concepts that come up in interviews about hermes or any similar agent system.

### 1. Synchronous Agent Loop
`run_agent.py:run_conversation()` — `def` (not `async def`), while loop, blocking LLM call, blocking tool execution, returns when pure text response or max_iterations hit. No `await` anywhere in the loop body.

### 2. `HERMES_HOME` Profile Isolation
`hermes_constants.get_hermes_home()` — called 119+ times. `HERMES_HOME` env var overrides `~/.hermes`. All state writes route through this. Test fixture `_isolate_hermes_home` uses this for full test isolation.

### 3. AST-Based Tool Discovery
`tools/registry.py:discover_builtin_tools()` — scans tool files via `ast.parse()` without importing them. `registry.register()` at module level is the self-registration contract. Zero circular import risk.

### 4. Prompt Cache Invariant
System prompt built once at session start in `_build_system_prompt()`. Never modified mid-session. Delivers ~75% cost reduction on Anthropic models. All dynamic content routes through user messages or tool results.

### 5. `_last_resolved_tool_names` Race Condition
`model_tools.py:159` — module-level (process-global) list. Written by `get_tool_definitions()`. Save/restored in `delegate_tool.py` around child agent runs. NOT thread-safe. Race window under concurrent gateway delegation. Fix: `threading.local()`.

### 6. Memory Sync Is Blocking
`agent/memory_manager.py:sync_all()` — plain `def`, blocking for-loop, called at end of each turn. Slow external providers block user response delivery. Documentation incorrectly describes as "non-blocking." Fix: background thread.

### 7. Context Compression at 75%
`agent/context_engine.py:59` — `threshold_percent: float = 0.75`. Auxiliary (cheaper) LLM summarizes the MIDDLE of the conversation. Creates new session with `parent_session_id`. Keeps system prompt + last N turns verbatim.

### 8. Three-Tier Memory
- Ephemeral: in-process `messages` list (session only)
- Semantic: `MEMORY.md` + `USER.md` flat files (across sessions)
- Episodic: SQLite FTS5 `messages_fts` table (keyword search across all sessions)

### 9. `<memory-context>` Fencing
`agent/memory_manager.py:build_memory_context_block()` — injected recalled memory is wrapped in `<memory-context>...</memory-context>` tags. Prevents injected memory from being treated as user instruction.

### 10. Skills Dual Injection
- `build_skills_system_prompt()` → goes to system prompt at session start as METADATA (list of available skills)
- `build_skill_invocation_message()` → goes to user message INPUT when skill is invoked (skill CONTENT)
Both paths preserve prompt cache validity.

### 11. Two Config Loaders
- `cli.py:load_cli_config()` — fallback to `./cli-config.yaml`, personality templates, inline defaults
- `hermes_cli/config.py:load_config()` — HERMES_HOME only, `_deep_merge()`, `max_turns` migration
Must stay in sync manually. Any new config option requires updating both.

### 12. Parallel Tool Safety Logic
`run_agent.py:_should_parallelize_tool_batch()` — returns False if: ≤1 tool, any tool in `_NEVER_PARALLEL_TOOLS`, any tool NOT in `_PARALLEL_SAFE_TOOLS`, path overlap detected. `_MAX_TOOL_WORKERS = 8` when parallel.

### 13. Gateway LRU Agent Cache
`gateway/run.py` — `OrderedDict` cache, `_AGENT_CACHE_MAX_SIZE = 128`, `_AGENT_CACHE_IDLE_TTL_SECS = 3600.0`. `_enforce_agent_cache_cap()` evicts oldest non-active entries. Cache key: `(platform, user_id)`.

### 14. SQLite WAL + FTS5
`hermes_state.py:SCHEMA_SQL` — `PRAGMA journal_mode=WAL`, jittered backoff (20–150ms) on write contention. `FTS_SQL` — `CREATE VIRTUAL TABLE messages_fts USING fts5(content=messages, ...)`. `parent_session_id` foreign key for compression chain.

### 15. Delegation (Child Agent)
`tools/delegate_tool.py` — spawns isolated `AIAgent` instance. Child has own messages list, session, iteration budget. Shares Python process with parent. `_last_resolved_tool_names` save/restore is the concurrency risk.

### 16. Cron Background Thread
`gateway/run.py:11031–11038` — `threading.Thread(target=_start_cron_ticker, daemon=True, name="cron-ticker")`. 60s interval. `fcntl` file lock prevents concurrent ticks. Cron jobs get a fresh `AIAgent` per execution.

### 17. Prompt Injection Defense (10 Patterns)
`agent/prompt_builder.py:_CONTEXT_THREAT_PATTERNS` — 10 regex patterns: prompt_injection, deception_hide, sys_prompt_override, disregard_rules, bypass_restrictions, html_comment_injection, hidden_div, translate_execute, exfil_curl, read_secrets.

### 18. COMMAND_REGISTRY Single Source of Truth
`hermes_cli/commands.py:COMMAND_REGISTRY` — all `CommandDef` objects. Five interfaces derive from this: CLI help, gateway dispatch, Telegram menu, Slack routing, autocomplete. `gateway_config_gate` field for conditional activation.

### 19. TUI Two-Process Architecture
Node.js (Ink/React) + Python (`tui_gateway`) over JSON-RPC via stdio pipes. Node spawns Python as child process. Node renders UI, Python runs AIAgent. Communication is bidirectional streaming JSON-RPC.

### 20. RL Training Infrastructure
`batch_runner.py` (ThreadPoolExecutor, N agents in parallel) → `trajectory_compressor.py` → Atropos submodule. Deployed hermes generates training trajectories for future model versions. The virtuous loop: better agents generate better training data.

---

## 20 Follow-up Questions You Might Get

1. How would you fix the `_last_resolved_tool_names` race? (Answer: threading.local())
2. Why is memory sync blocking? Could you make it async? (Background thread, accept eventual consistency)
3. What happens when the gateway has 129 users? (LRU evicts oldest non-active agent; conversation continues from SQLite history)
4. Why does the CLI default use 0.50 threshold while the class default is 0.75? (CLI applies its own config; class default is for programmatic use)
5. How do skills preserve the prompt cache? (Content injects as user message, not system prompt)
6. What would break if you added `async` to `run_conversation()`? (All callers, all tests, all platform adapters — full async infection)
7. How does context compression affect session search? (Parent session and child session both searchable; `parent_session_id` links them)
8. Can two users on the same gateway instance see each other's memories? (No — MEMORY.md is per-user if using separate HERMES_HOME; gateway sessions are isolated by user_id cache key)
9. What happens to an active agent session when the gateway receives SIGTERM? (Current tool call may complete; session is marked ended in SQLite; next user message creates a new session restoring from SQLite history)
10. Why does hermes use `threading.Thread` for cron instead of asyncio.create_task? (Agent loop is synchronous; cron tick function calls `run_conversation()` which is blocking; asyncio can't await blocking calls without a thread)
11. How would you add a new messaging platform to the gateway? (Implement the platform adapter interface; register with GatewayRunner; no core changes needed)
12. What prevents a malicious MEMORY.md from injecting instructions? (`<memory-context>` fencing; model trained to interpret tags; not a hard technical barrier)
13. How does FTS5 handle tool results? (If stored in message `content` field, they're indexed. Large tool results bloat the FTS index — known gap)
14. Why does hermes support 20+ LLM providers? (OpenAI-wire format compatibility; any OpenAI-compatible endpoint works with no code changes; native SDK for Anthropic-specific features)
15. What is `IterationBudget` and why is it separate from `max_iterations`? (Budget can be shared with child agents; budget.remaining decreases as child agents consume iterations; prevents delegation from circumventing iteration limits)
16. How would you debug a slow turn response? (Check tool execution time via logs; check if memory sync is the bottleneck; check SQLite write retry rates; check LLM API latency separately)
17. What is `_SafeWriter` in `run_agent.py`? (Broken pipe resilience for daemon/gateway mode — SIGPIPE doesn't crash the process when stdout is closed)
18. How does hermes handle a tool that raises an exception? (Exception caught in execute wrapper; returns error as tool result message; model sees the error and can retry or adapt)
19. What is `atomic_json_write` in `utils.py`? (Write to temp file, rename to target — makes JSON writes atomic; prevents partial writes from corrupting JSON files)
20. Why does skill invocation use `_pending_input.put(msg)` rather than calling the agent directly? (Skills invoke mid-REPL; the pending input queue is the natural injection point into the REPL loop without bypassing the CLI input handling logic)

---

## 10 Mental Diagrams to Draw

Drawing these from memory is a strong indicator of architectural understanding.

1. **The agent turn lifecycle**: User input → memory prefetch → LLM call → tool dispatch (parallel/sequential) → tool results → loop back or end → memory sync → SQLite persist

2. **The memory injection pipeline**: MEMORY.md read → FTS5 search → external provider prefetch → `<memory-context>` fence → inject into user message

3. **The compression chain**: Original session (ID=A) → threshold hit → auxiliary LLM summary → new session (ID=B, parent_session_id=A) → continued conversation

4. **The gateway request flow**: Telegram message → platform adapter → GatewayRunner.get_or_create_agent(platform, user_id) → check LRU cache → AIAgent.run_conversation() → response → platform adapter → Telegram reply

5. **The tool registry chain**: Module-level `registry.register()` call → AST discovery scans files → ToolRegistry._tools dict → `get_tool_definitions()` builds OpenAI schemas → LLM receives tool list → LLM requests tool call → tool executed

6. **The config load path**: HERMES_HOME → config.yaml → `load_config()` → `_deep_merge(DEFAULT_CONFIG, user_config)` → `_normalize_max_turns_config()` → return merged dict

7. **The delegation flow**: Parent agent → delegate_task tool call → save `_last_resolved_tool_names` → spawn child AIAgent → child.run_conversation() → restore `_last_resolved_tool_names` → return child's final response as tool result to parent

8. **The skills dual-path**: Session start → `build_skills_system_prompt()` → skill metadata to system prompt (static). User types `/skill-name` → `build_skill_invocation_message()` → skill content to user message input queue (dynamic)

9. **The LRU cache**: New user message → `get_or_create_agent(platform, user_id)` → check OrderedDict → hit: move_to_end, return agent → miss: create new AIAgent → `_enforce_agent_cache_cap()` if >128 → evict oldest non-active

10. **The prompt injection defense**: External content (web page, file read, memory recall) → scan with `_CONTEXT_THREAT_PATTERNS` (10 regexes) → if match: replace with `[BLOCKED: ...]` → if no match: pass through → inject into message

---

## 10 Common Mistakes

Things people get wrong about hermes architecture.

1. **"The agent loop is async"** — It's synchronous. `run_conversation()` is `def`, not `async def`. Zero `await` calls.

2. **"Memory sync is fire-and-forget"** — It's blocking. `sync_all()` loops over providers synchronously. Slow providers delay turn completion.

3. **"Skills go into the system prompt"** — Skills CONTENT injects as user messages. Only the skills METADATA list is in the system prompt.

4. **"The compression threshold is always 75%"** — Class default is 0.75 but the CLI applies its own configured value. The threshold is configurable.

5. **"All tools run in parallel"** — Only when `_should_parallelize_tool_batch()` returns True. Many tools are in `_NEVER_PARALLEL_TOOLS` or not in `_PARALLEL_SAFE_TOOLS`.

6. **"`_last_resolved_tool_names` is thread-safe"** — It's a module-level list with no lock. Thread-safe in the delegate_tool save/restore pattern only for single-threaded delegation.

7. **"SQLite can't handle the gateway"** — WAL mode + jittered backoff handles moderate concurrency. The limit is 50+ concurrent writers, not 5–10.

8. **"hermes is built on LangChain"** — Deliberately avoids external agent frameworks. The agent loop is 100% custom.

9. **"Config changes take effect immediately"** — Config is loaded at session start. Config changes require a new session (CLI restart or new gateway conversation).

10. **"The gateway agent cache is infinite"** — Max 128 agents. Eviction happens when the 129th user sends a message. The evicted agent's session persists in SQLite — the conversation isn't lost, but the warm agent is.
