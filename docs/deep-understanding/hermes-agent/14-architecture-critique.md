# 14 — Architecture Critique: hermes-agent

## Strengths

### 1. Synchronous Core with Clean Async Boundary
The agent loop (`run_conversation()`) is entirely synchronous. This is a deliberate and correct choice: it makes reasoning about state trivial, avoids async infection spreading through the whole codebase, and matches developer expectations for a Python tool. Async is used only where it genuinely belongs — per-platform messaging adapters (Telegram, Discord) and the Gateway's outer event loop. The boundary is clean.

### 2. `HERMES_HOME` Profile Isolation
Every state-writing path routes through `hermes_constants.get_hermes_home()`. A single env var (`HERMES_HOME=/tmp/test hermes`) creates a completely isolated agent. This enables: parallel test runs (the `_isolate_hermes_home` autouse fixture redirects all writes to a temp dir), multiple user profiles without code changes, and clean uninstall by deleting one directory. This is one of the most pragmatic single-var isolation patterns in any production agent codebase.

### 3. AST-Based Tool Discovery
`tools/registry.py` discovers tools via `ast.parse()` instead of Python imports. Benefits: no circular import risk, faster startup (modules not executed during discovery), and no global import side effects. The registry.register() call at module level is a self-describing contract — adding a tool is one function call.

### 4. Single COMMAND_REGISTRY Source of Truth
All five interfaces (CLI, gateway, Telegram menu, Slack routing, autocomplete) derive from `hermes_cli/commands.py:COMMAND_REGISTRY`. Adding a new command in one place makes it available everywhere. The `gateway_config_gate` field allows CLI-only defaults with gateway opt-in via config — this is a very clean extensibility mechanism that doesn't require adding if/else branches in five places.

### 5. Prompt Cache Invariant Enforced Architecturally
The prohibition on mid-conversation system prompt mutations is not just a policy — it's architecturally enforced by how system prompt is built (once at session start). This delivers ~75% cost reduction on multi-turn conversations via Anthropic prompt caching. Most agent frameworks let this degrade silently; Hermes elevates it to an invariant.

### 6. `_should_parallelize_tool_batch()` Safety Logic
Rather than always parallelizing or never parallelizing, the parallel/sequential decision is explicit: a blocklist of tools that must never run in parallel, an allowlist of tools safe for concurrency, and path-overlap detection for file tools. This avoids race conditions while maximizing throughput for safe operations.

### 7. Pluggable Context Engine
The `ContextEngine(ABC)` lifecycle is the strongest design abstraction in the codebase: session-start → token tracking → `should_compress()` → `compress()` → session-end. Swapping the compression strategy is a one-class change. The auxiliary model for summarization is configurable. This is how LLM infrastructure should be designed.

---

## Weaknesses

### 1. Process-Global `_last_resolved_tool_names` (Latent Race Condition)
`model_tools.py:159` defines `_last_resolved_tool_names` at module level. `delegate_tool.py` saves/restores this around child agent runs. But in gateway mode, multiple user sessions run as concurrent threads — each thread may call `get_tool_definitions()` (which writes the global) and `delegate_task` (which saves/restores) simultaneously. The save/restore is not atomic under concurrent access. No lock protects the global. This hasn't caused reported bugs because gateway delegation is rare, but the race window exists under load.

**Fix**: Replace with a threading.local() value or pass the tool list explicitly through the call stack.

### 2. Blocking Memory Sync Delays Turn Response
`MemoryManager.sync_all()` calls `provider.sync_turn()` in a blocking for-loop at the end of each turn. If an external provider (Honcho, Hindsight, Mem0) is slow (network latency, API throttling), the user sees the agent "processing" after the response has been generated. The first-pass documentation incorrectly described this as "fire-and-forget" — it is not.

**Fix**: Run `sync_all()` in a background thread and return the response immediately. Accept the race risk that memory may not be persisted before the next prefetch (acceptable for eventual-consistency memory systems).

### 3. Two Config Loaders with Divergent Logic
`cli.py:load_cli_config()` and `hermes_cli/config.py:load_config()` load from different paths, have different default handling, and different migration logic. CLI's loader includes personality templates; the library loader doesn't. This means: a config option added to one loader must be manually added to the other. Bugs from this kind of divergence will compound as the feature set grows.

**Fix**: Unify into a single config loader; pass a `mode` parameter or use composition.

### 4. No Per-Tool Timeout Enforcement
`ToolRegistry` stores tool metadata but no timeout field. The `_execute_tool_calls_sequential()` and parallel executor have no deadline mechanism. A tool that hangs (network I/O, subprocess blocking) will hang the agent loop indefinitely. In gateway mode with multiple sessions, this can starve other users' threads.

**Fix**: Add an optional `timeout_seconds` field to `ToolEntry`; wrap execution in `concurrent.futures.Future.result(timeout=...)`.

### 5. SQLite Single-Writer Architecture
WAL mode and jittered backoff handle moderate concurrency, but SQLite remains fundamentally single-writer. In gateway mode with 10+ active simultaneous conversations all writing to `state.db`, write contention will increase linearly. The retry backoff (20–150ms jitter) adds latency to every turn in a contention scenario.

**Mitigation available**: Replace `SessionDB` behind its interface with Postgres. The abstraction is there; the migration just hasn't been built.

### 6. Skill Quality Has No Enforcement Layer
Skills are LLM-generated markdown files stored in `~/.hermes/skills/`. There is no automated review, no schema validation, no deduplication check. The agent's self-improvement loop accumulates skills without hygiene. Over months of use, a skill directory can accumulate redundant, conflicting, or degraded entries. The "learning" accumulates noise alongside signal.

### 7. ACP Adapter Is Under-Documented and Under-Tested
The `acp_adapter/` integrates Hermes with VS Code/JetBrains via Agent Client Protocol. The adapter is present in the codebase but was not covered in first-pass documentation, and test coverage is unclear. For teams evaluating IDE integration, this is a critical but opaque feature.

---

## Hidden Coupling

### A. System Prompt and Session Identity
The system prompt is immutable during a session — this is the prompt cache invariant. But the system prompt includes memory context, which means memory recalled at session start is "baked in" to the cache key. A memory update mid-session (from a memory write tool) won't be visible in the system prompt until the next session. This coupling between memory and session identity is non-obvious.

### B. `model_tools.py` and `tools/registry.py` Coupling
`model_tools.py:get_tool_definitions()` queries the registry to build the OpenAI-format tool list. It also writes `_last_resolved_tool_names`. Any code that calls `get_tool_definitions()` has a side effect: it overwrites the global. This creates subtle action-at-a-distance bugs where a seemingly read-only operation has write effects.

### C. Platform Adapters and Session Cache
Gateway platform adapters call `GatewayRunner.get_or_create_agent(platform, user_id)` which may create new `AIAgent` instances. The `AIAgent` constructor opens a SQLite connection. If platform adapters are refactored without awareness of session cache behavior, connection leaks can occur.

### D. Cron Jobs and Gateway Adapters
Cron jobs that use `delivery: home_channel` require access to the gateway's platform adapters at tick time. The cron thread receives `adapters` as a parameter when the gateway starts. Cron jobs defined via the CLI (without the gateway running) cannot deliver to home channels — this dependency is implicit and not surfaced to users who create cron jobs from the CLI.

---

## Failure Modes

### High-probability, bounded impact
- **Slow external memory provider**: Blocks turn completion by seconds. User-visible as slow responses after the agent text appears. Fix: background thread.
- **SQLite write lock**: Jittered retry absorbs brief contention; extended contention (5+ retries) propagates as an error. Fix: Postgres backend.
- **Tool hangs indefinitely**: Stalls the agent session, potentially orphaning the session in SQLite. Fix: per-tool timeouts.

### Low-probability, high impact
- **`_last_resolved_tool_names` race under delegation concurrency**: Could cause child agents to run with wrong tool lists, producing incorrect behavior. Fix: threading.local().
- **Context compression compresses critical tool context**: If the summarizer loses essential tool result data from earlier in the conversation, the agent may behave incorrectly in later turns. Mitigation: keep last N turns always; increase auxiliary model quality.
- **MEMORY.md corruption**: Two concurrent agent sessions both writing to MEMORY.md (possible in gateway mode with concurrent sessions from the same user) could produce garbled memory. `atomic_json_write` helper exists but may not cover all write paths.

### Design-level risks
- **Skill pollution accumulation**: Self-created skills with errors get invoked as if correct. No review gate.
- **SQLite FTS5 index growth**: All message content is indexed. Very long tool results (web pages, file contents) bloat the FTS index without truncation. Large deployments will see slow search over time.

---

## Scale Bottlenecks

| Component | Single User | 10 Users | 100 Users |
|-----------|------------|----------|-----------|
| SQLite write | Fine (WAL) | Contention starts | Retry cascades |
| Memory sync blocking | Imperceptible | Noticeable (slow provider) | Significant latency |
| Tool parallelism (8 workers) | Fine | Per-session (isolated) | N/A (per-session) |
| Gateway agent LRU cache | Fine | Fine (128 slots) | Eviction churn |
| Cron thread (single) | Fine | Fine | Job queue can grow |
| FTS5 search index | Fast | Fast | Depends on message volume |

Hermes is designed for single-user or small-team deployments. 10–50 concurrent users: manageable with careful tuning. 100+ concurrent users: needs Postgres backend, distributed tracing, and likely a queue for cron execution.
