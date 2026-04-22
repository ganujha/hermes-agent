# 02 — Agent, Tool, and Memory Analysis: hermes-agent

## The Agent Loop

The core agent loop lives in `run_agent.py:AIAgent.run_conversation()`. It is a **synchronous Python while-loop** that runs inside whatever async context calls it (CLI REPL, gateway coroutine, batch runner thread).

```
while api_call_count < max_iterations and iteration_budget.remaining > 0:
    1. Check context compression threshold
    2. Apply prompt caching headers (Anthropic only)
    3. Call LLM API (provider-specific)
    4. Parse text + tool_calls from response
    5. Track tokens, update cost estimates
    6. If no tool_calls → break (return final text)
    7. Execute tool calls (parallel or sequential)
    8. Append tool results to messages
    9. Persist turn to SQLite (if session tracking enabled)
    10. Post-turn async memory sync
    continue
```

**Max iterations**: 90 by default (`AIAgent.__init__:max_iterations=90`). Configurable via `config.yaml`.

**Termination conditions**:
- Model returns no tool calls (natural completion)
- `max_iterations` reached (hard cap)
- `IterationBudget.remaining` exhausted (soft cap, refundable)
- `set_interrupt()` called (Ctrl+C or `/stop` from gateway)
- Unrecoverable API error after retries

---

## Single-Agent, Multi-Agent, or Hybrid?

**Hybrid**: Single-agent by default, multi-agent on demand via delegation.

- **Default mode**: One `AIAgent` instance per conversation. The agent loop handles all tool calls inline.
- **Delegation mode**: `delegate_task` tool creates a new `AIAgent` child instance inside `tools/delegate_tool.py:_run_single_child()`. The parent agent blocks while the child runs its own full loop, then receives the child's final response as a tool result.
- **Gateway mode**: `GatewayRunner` maintains a cache of up to 128 `AIAgent` instances (one per platform session), each running independently in response to messages.
- **Batch mode**: `batch_runner.py` runs many `AIAgent` instances in parallel threads for trajectory generation.

---

## Coordinator/Subagent Patterns

**`delegate_task` tool** (`tools/delegate_tool.py`):
- Parent agent calls `delegate_task(prompt, toolsets=[...])` as a tool call
- `_run_single_child()` instantiates a fresh `AIAgent` with isolated state
- Child has its own message history, tool access, and iteration budget
- Critical: `_last_resolved_tool_names` (process-global in `model_tools.py`) is saved and restored around each child run
- Parent receives child's final text response as a tool result
- No shared memory between parent and child by default

**Mixture of agents** (`tools/mixture_of_agents_tool.py`):
- Separate tool file suggesting MoA (mixture of agents) patterns exist
- Details require deeper inspection of this file

**Parallel delegation pattern** (inferred from docs):
- Parent agent can issue multiple `delegate_task` calls in one turn
- Tool parallel execution system executes them concurrently (up to 8 workers)
- Effective for parallelizing independent workstreams

---

## Tool Invocation Model

**Registration**: Self-registering. Each `tools/*.py` file calls `registry.register()` at module level. The registry is populated at import time.

**Discovery**: `tools/registry.py:discover_builtin_tools()` uses Python's `ast` module to scan `tools/*.py` for top-level `registry.register()` calls — no manual import list required.

**Schema exposure**: `model_tools.py:get_tool_definitions()` queries the registry, applies enabled/disabled toolset filters, and returns a list of JSON schemas in the format the LLM provider expects.

**Dispatch**: `model_tools.py:handle_function_call(tool_name, args, task_id)` looks up the handler in the registry and calls it. All handlers must return a JSON string.

**Parallel execution logic** (from `run_agent.py`):
- `_NEVER_PARALLEL_TOOLS`: set of tools that must run sequentially (e.g., `clarify`)
- `_PARALLEL_SAFE_TOOLS`: set of known read-only tools
- Path-scoped tools: checked for overlapping file paths to prevent races
- If a batch has any sequential-only tool, the entire batch runs sequentially
- `ThreadPoolExecutor(max_workers=8)` for parallel execution

**MCP tools** (`tools/mcp_tool.py` ~1050 lines):
- MCP (Model Context Protocol) client connects to external MCP servers
- Discovered tools appear alongside built-in tools in the tool list
- Config-driven server definitions in `~/.hermes/config.yaml`

**Plugin tools**:
- Discovered from `plugins/` directory at startup
- Same `registry.register()` interface as built-in tools

---

## Memory Model

Hermes has a **three-layer memory model**:

### Layer 1: Episodic (In-Process)
- `messages[]` list in `AIAgent` — the current conversation history
- Ephemeral: lost when the process exits
- Full OpenAI-format message list (system/user/assistant/tool roles)

### Layer 2: Semantic (File-Based)
- `~/.hermes/MEMORY.md` — free-form agent-curated notes
- `~/.hermes/USER.md` — user profile and preferences
- Written by the `memory` tool via explicit agent decision
- Injected into system prompt on every call
- Persists across sessions and process restarts

### Layer 3: Session History (SQLite)
- `~/.hermes/state.db` — all session messages persisted to SQLite
- `messages_fts` FTS5 virtual table enables full-text search
- `session_search` tool lets the agent search its own history
- Parent/child session chains via `parent_session_id` (compression splits sessions)

### External Memory Providers (Optional)
- One provider at a time (enforced by `MemoryManager`)
- Providers: Honcho (dialectic user modeling), Hindsight, Mem0
- Lifecycle: `prefetch()` before turn → `sync_turn()` after turn → `on_session_end()`
- Memory injected as `<memory-context>` fenced block into the user message
- Fencing prevents the model from treating recalled context as new user discourse

---

## Session Model

**CLI mode**: One session per invocation. Session ID created at start, saved to `state.db`. `/new` or `/reset` creates a new session. Context compression splits session into parent+child chain.

**Gateway mode**: One session per platform+user+chat combination. Sessions cached in `GatewayRunner._agent_cache` (OrderedDict, LRU). Max 128 active sessions. Sessions idle >1 hour are evicted.

**Session fields** (from `hermes_state.py:SCHEMA_SQL`):
- `source` (platform: `cli`, `telegram`, `discord`, etc.)
- `user_id` (platform user identifier)
- `model` (model used)
- `parent_session_id` (compression chain)
- `started_at`, `ended_at`, `end_reason`
- Token counts (input, output, cache_read, cache_write, reasoning)
- Cost tracking (`estimated_cost_usd`, `actual_cost_usd`)
- `title` (auto-generated from first user message)

**Profile isolation**: Each Hermes profile (`hermes -p <name>`) gets its own `HERMES_HOME` directory, so sessions, memory, config, and credentials are fully isolated.

---

## Context-Passing Model

Messages follow OpenAI format throughout. The system assembles:
```
[
  {"role": "system", "content": "<assembled system prompt>"},
  {"role": "user",   "content": "<memory context> + user message"},
  {"role": "assistant", "content": "...", "tool_calls": [...]},
  {"role": "tool", "tool_call_id": "...", "content": "..."},
  ...repeat...
]
```

**Skills** are injected as user messages (not system), specifically to preserve prompt caching validity. Changing the system prompt mid-conversation invalidates all cached prefixes.

**Compression summary** is inserted as an assistant message with a `[CONTEXT COMPACTION — REFERENCE ONLY]` prefix, framing it as notes from a "previous assistant instance" — this prevents instruction bleed from the old context.

**Memory context** is injected at API-call time only, not persisted in the message list, to prevent the model from treating recalled memory as user-said discourse.

---

## Retry / Recovery Model

**API-level retries**:
- `tenacity` library used for retries (`pyproject.toml:tenacity>=9.1.4`)
- `agent/retry_utils.py:jittered_backoff()` implements exponential backoff with jitter
- `agent/error_classifier.py:classify_api_error()` determines if error is retryable vs failover-worthy

**Provider failover** (`agent/error_classifier.py:FailoverReason`):
- Rate limit → try next credential from pool
- Authentication failure → try next credential or next provider
- Context overflow → trigger compression, retry
- Model not found → try alternate model
- Network timeout → retry with longer timeout

**SQLite write contention** (`hermes_state.py`):
- Jittered backoff: 20–150ms random sleep between retries
- WAL mode reduces contention between concurrent readers and the single writer

**Tool errors**:
- Tool handlers wrapped to return error JSON on exception
- `tools/registry.py:tool_error()` formats error responses
- Agent sees error as tool result, can retry or adapt

---

## Points of Determinism vs Model Discretion

**Deterministic** (code-enforced):
- Which tools are available (toolset config)
- Max iterations (hard cap at 90)
- When compression fires (token threshold)
- Tool parallel/sequential execution rules
- Command approval for dangerous operations
- Session storage and retrieval
- Memory injection format (`<memory-context>` fence)

**Model discretion** (prompt-guided only):
- Which tools to call and when
- When to write to memory vs not
- When to create a skill
- Whether to delegate a subtask
- When to use `clarify` to ask for input
- How to handle ambiguous instructions
- When to stop vs keep iterating

**Hybrid** (prompt-guided with code enforcement):
- Context compression: code decides WHEN, LLM decides HOW (summarization content)
- Tool selection: code provides schema and toolset restrictions, LLM decides which to call
- Skill creation: nudge interval is code-enforced, actual skill writing is LLM-driven

---

## What Would Break First Under Scale or Ambiguity

1. **SQLite as session store**: Single-writer model fine for personal use, but multi-user deployments under high concurrency will see write contention. WAL + jittered backoff buys time but doesn't scale horizontally.

2. **In-memory message list**: Context grows with conversation length until compression fires. Deep research sessions with many tool calls can hit 50% of context before compression, then the summary becomes the single point of failure for conversation continuity.

3. **Process-global `_last_resolved_tool_names`**: Delegate tool saves/restores this, but under multi-threaded gateway load (multiple sessions firing concurrently), a race condition here could cause tool schema corruption in child agents.

4. **Single external memory provider enforcement**: Architectural simplicity, but limits capability. Users who want both Honcho (user modeling) and Mem0 (retrieval) cannot use both simultaneously.

5. **90-iteration hard cap**: Legitimate long-running autonomous tasks can be truncated. There's no native checkpoint/resume for mid-task interruptions.

6. **LLM decides when to create skills**: Under heavy use, the agent may create redundant or low-quality skills. There's no automated quality gate or deduplication in the base architecture.

7. **Tool output size**: Large tool results (e.g., full file reads, long terminal output) can rapidly exhaust context. `tools/tool_result_storage.py:enforce_turn_budget()` manages per-turn limits, but aggressive tasks can still consume context quickly.
