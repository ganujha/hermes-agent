# 16 — Production Readiness Review: hermes-agent

## Assessment Framework

For each component, rated on:
- **P** = Production-grade (could serve paying customers today)
- **S** = Solid prototype (needs one or two targeted fixes)
- **D** = Development quality (needs significant work before production trust)

---

## Component Ratings

| Component | Rating | Rationale |
|-----------|--------|----------|
| Core agent loop (`AIAgent.run_conversation`) | P | Synchronous, well-tested, retry logic via tenacity, iteration budget enforced |
| Provider abstraction (20+ LLM providers) | P | Mature, battle-tested abstraction; failover logic present |
| SQLite session store | S | Works for moderate concurrency; WAL + jittered retry adequate for small teams; not for 50+ concurrent writers |
| Tool registry + execution | S | Solid for most tools; missing per-tool timeout enforcement |
| Context compression | P | Clean ABC, auxiliary model configurable, threshold configurable, compression chain tracked via parent_session_id |
| Memory system (builtin) | S | MEMORY.md works; concurrent write safety on gateway is weak; needs pruning/deduplication |
| Memory system (external providers) | S | Honcho/Hindsight/Mem0 integrations exist; memory sync blocks turn completion (should be async) |
| Gateway multi-platform | P | 16+ platform adapters, LRU agent cache, cron delivery; works in real deployments |
| Cron scheduler | S | File-based lock works on single host; not durable across crashes; no HA story |
| CLI interface | P | prompt_toolkit REPL, autocomplete, rich formatting, slash commands — mature UX |
| Config system | S | Two config loaders diverge; needs unification before adding more config options |
| Authentication / credential pool | P | Multiple auth flows (OAuth PKCE, device code), dotenv, auth.json; well-layered |
| Error handling | S | Error classifier and tenacity retries are good; no per-tool timeouts; no circuit breaker |
| Logging | S | Standard Python logging; no structured JSON; no distributed tracing; no metrics endpoint |
| Test coverage | S | pytest + asyncio + xdist; good unit coverage of core; integration tests require API keys; no end-to-end test of gateway |
| Skills system | D | No quality control, no deduplication, no automated review; skill pollution accumulates |
| ACP adapter (IDE integration) | D | Exists but under-documented and under-tested; unclear production usage |
| Web dashboard | D | React/Vite frontend exists; unclear test coverage or production usage |
| Security (prompt injection) | S | 10-pattern detection is basic; no sandboxing of tool execution (shell tools run in user context) |
| `_last_resolved_tool_names` concurrency | D | Latent race condition under concurrent delegation in gateway mode |

---

## Top 10 Upgrades for Production

Ordered by impact vs. effort:

### 1. Background Memory Sync (High impact / Low effort)
**Problem**: `MemoryManager.sync_all()` blocks turn completion. Slow external providers delay user-visible responses.
**Fix**: Spawn a `threading.Thread` (or `concurrent.futures.ThreadPoolExecutor`) for `sync_all()`. Return the response immediately; sync in background.
**Risk**: Turn N+1 memory prefetch may not see turn N memory. Acceptable for eventual-consistency memory systems.
**Files**: `run_agent.py:~11781`, `agent/memory_manager.py:210–219`

### 2. Per-Tool Timeout Enforcement (High impact / Medium effort)
**Problem**: A hanging tool (network I/O, subprocess) stalls the agent loop indefinitely.
**Fix**: Add `timeout_seconds: Optional[float]` to `ToolEntry`; wrap `_execute_single_tool()` in `concurrent.futures.Future.result(timeout=...)`.
**Files**: `tools/registry.py:ToolEntry`, `run_agent.py:_execute_single_tool()`

### 3. Fix `_last_resolved_tool_names` Race (High impact / Low effort)
**Problem**: Module-level list in `model_tools.py` is mutated by all threads; delegate_tool save/restore is not atomic.
**Fix**: Replace `_last_resolved_tool_names: List[str] = []` with `_last_resolved_tool_names = threading.local()` and update all read/write paths.
**Files**: `model_tools.py:159`, `tools/delegate_tool.py:100–113, 1155–1187`

### 4. Unify Config Loaders (Medium impact / Medium effort)
**Problem**: Two config loaders (`load_cli_config()` and `load_config()`) diverge silently; any new config option requires updating both.
**Fix**: Extract a single `load_config(mode="cli"|"library")` function; deprecate the old loaders.
**Files**: `cli.py:268–400`, `hermes_cli/config.py:3039–3065`

### 5. Structured Logging + Metrics Endpoint (Medium impact / Medium effort)
**Problem**: Python `logging` produces unstructured text logs; no way to aggregate errors, latency, or token usage in an observability platform.
**Fix**: Add a structured logging formatter (`python-json-logger` or `structlog`); add a `/healthz` and `/metrics` endpoint to the ACP/web FastAPI app.
**Files**: `hermes_logging.py`, `acp_adapter/server.py`

### 6. Postgres Backend for SessionDB (Medium impact / High effort)
**Problem**: SQLite is single-writer; write contention grows with concurrent sessions.
**Fix**: Implement `PostgresSessionDB` behind the `SessionDB` interface; feature-flag between SQLite and Postgres via config.
**Files**: `hermes_state.py:SessionDB` interface, new `hermes_state_postgres.py`

### 7. Skill Quality Control Layer (Medium impact / Medium effort)
**Problem**: Self-created skills accumulate without review, creating signal/noise ratio degradation.
**Fix**: Add a skill validation step on creation (schema check, duplicate detection via embedding or hash), and a `skill doctor` CLI command to audit/prune the skills directory.
**Files**: `tools/skill_manage.py` (creation path), new `hermes_cli/skill_doctor.py`

### 8. Durable Cron Queue (Low-Medium impact / High effort)
**Problem**: In-memory cron scheduling doesn't survive gateway crashes; no HA story.
**Fix**: Persist cron execution state to SQLite (add `cron_runs` table tracking last execution per job); add crash-recovery logic to scheduler startup.
**Files**: `cron/scheduler.py`, `hermes_state.py:SCHEMA_SQL`

### 9. Tool Sandboxing (High impact / Very High effort)
**Problem**: Shell tools (`terminal`, `bash`) execute as the user process with no resource limits or capability restrictions.
**Fix**: Wrap shell tool execution in a subprocess with restricted environment; add a resource limits configuration (CPU, memory, allowed paths). Consider integrating with `nsjail` or Docker for stronger isolation.
**Files**: `tools/terminal_tool.py`, `tools/bash_tool.py`

### 10. End-to-End Integration Test Suite (Medium impact / High effort)
**Problem**: Integration tests are skipped without API keys; no automated test of the full gateway flow (message in → agent turn → response delivery).
**Fix**: Add a mock-provider mode (`HERMES_MOCK_LLM=1`) that returns deterministic responses; write E2E tests using mock providers for CI coverage of gateway, cron, and memory flows.
**Files**: `tests/integration/`, `run_agent.py:provider factory`

---

## What Is Prototype-Grade vs Production-Grade

### Production-Grade (deploy today for small teams)
- Core agent loop, LLM provider abstraction, streaming responses
- CLI interface (prompt_toolkit REPL)
- Session persistence (SQLite, WAL, FTS5)
- Multi-platform gateway (16+ platforms)
- Context compression (clean ABC, configurable threshold)
- Credential management (PKCE OAuth, dotenv, auth.json)
- Profile isolation (HERMES_HOME pattern)
- Cost tracking (token accounting, pricing estimates)

### Solid Prototype (one or two targeted fixes needed)
- Memory system (needs async sync, concurrent write safety)
- Config system (needs loader unification)
- Error handling (needs per-tool timeouts, circuit breaker)
- Cron scheduler (needs durable state for crash recovery)
- Logging (needs structured output for observability)

### Development Quality (significant work before production trust)
- Skills system (no quality control, no deduplication)
- ACP adapter / IDE integration (under-documented)
- Web dashboard (unclear test coverage)
- Tool sandboxing (runs as user process, no resource limits)
- Concurrent delegation safety (`_last_resolved_tool_names` race)

---

## Security Posture

| Concern | Current State | Risk Level |
|---------|--------------|----------|
| Prompt injection detection | 10 regex patterns; basic but present | Medium — regex is bypassable |
| Tool execution sandboxing | No sandboxing; runs as user | High for multi-tenant |
| API key storage | Dotenv + auth.json in HERMES_HOME | Acceptable for single-user |
| DM pairing (gateway) | PKCE OAuth per user | Acceptable |
| Memory injection fencing | `<memory-context>` tags | Good |
| Skill content review | None | Medium — LLM-generated code runs |
| Session data encryption | None (SQLite plaintext) | Medium for sensitive conversations |

**Bottom line**: Security posture is appropriate for self-hosted single-user or small-team use. NOT appropriate for multi-tenant public deployments without the sandboxing, encryption, and rate-limiting work.
