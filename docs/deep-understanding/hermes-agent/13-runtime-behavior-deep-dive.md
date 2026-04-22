# 13 — Runtime Behavior Deep Dive: hermes-agent

## Startup Lifecycle

### CLI Mode (`hermes` command)

```
hermes (shell shim)
  → activates venv, exec python hermes_cli/main.py
    → fire.Fire(HermesCLI)
      → HermesCLI.__init__()
        → load_cli_config()           # reads ~/.hermes/config.yaml or ./cli-config.yaml
        → get_hermes_home()           # resolves HERMES_HOME → ~/.hermes
        → CredentialPool.build()      # loads API keys from .env, auth.json
        → ToolRegistry.discover()     # AST-scans tools/*.py for registry.register() calls
        → AIAgent.__init__()          # creates agent (no LLM call yet)
          → SessionDB.open()          # opens ~/.hermes/state.db with WAL mode
          → MemoryManager.build()     # loads active memory providers
          → ContextCompressor.__init__() # sets threshold, context_length
      → HermesCLI.chat()
        → PromptSession (prompt_toolkit)  # REPL starts
          → loop: read input → AIAgent.run_conversation() → print response
```

**Cold start time**: Dominated by tool discovery (AST scan of 50+ modules) and SQLite initialization. No LLM call during startup.

### Gateway Mode (`hermes gateway start`)

```
hermes gateway start
  → gateway/run.py:start_gateway()
    → GatewayRunner.__init__()
      → load platform configs from ~/.hermes/config.yaml
      → instantiate platform adapters (Telegram, Slack, Discord, ...)
      → self._agent_cache = OrderedDict()  # LRU cache, max 128
    → runner.start_all_platforms()
      → per-platform asyncio loop (Telegram uses python-telegram-bot, ...)
    → start cron thread:
        threading.Thread(target=_start_cron_ticker, daemon=True, name="cron-ticker").start()
    → await runner.run_forever()  # asyncio event loop, never returns
```

**Agent creation in gateway**: Lazy — `AIAgent` created on first message from a user, then cached. Cache key = (platform, user_id). Cache evicts LRU entries when `len > 128`.

---

## `run_conversation()` Control Flow

```
AIAgent.run_conversation(user_message)
  → Session.start_turn(user_message)    # persist to SQLite
  → MemoryManager.prefetch_all()        # inject recalled memory into message
  → _build_system_prompt()              # system prompt (CACHED — immutable during session)
    → skills_system_prompt()            # list of available skills (metadata, not content)
    → platform hints
    → memory context block
  → messages.append({"role":"user","content": user_message})  # with injected memory
  
  MAIN LOOP (while iteration < max_iterations and budget.remaining > 0):
    → LLM API call (blocking)
      → anthropic_adapter or openai_adapter selects provider
      → stream response tokens to stdout
    → if response has tool_calls:
        → _should_parallelize_tool_batch(tool_calls)
          → True: ThreadPoolExecutor(max_workers=min(n_tools, 8))
          → False: execute sequentially
        → per tool: tool_func(**args) → result
        → messages.append(tool_results)
        → loop continues
    → else (pure text response):
        → BREAK

  → Session.end_turn(final_response)    # persist to SQLite
  → MemoryManager.sync_all(user_msg, final_response)  # BLOCKING: syncs to providers
  → return final_response
```

**Key control flow invariants:**
- System prompt is built ONCE at session start and never modified during a session (prompt cache invariant)
- `max_iterations` default = 90; loop exits on pure-text response or budget/limit exhaustion
- Tool execution: parallel if `_should_parallelize_tool_batch()` returns True (checked per batch)
- Memory sync at END of turn — blocks until all providers acknowledge

---

## State Mutations Per Turn

| Stage | What Changes | Where |
|-------|-------------|-------|
| Turn start | New `messages` row in SQLite | `hermes_state.py:SessionDB.insert_message()` |
| User message | `messages` list extended in-process | `run_agent.py:messages.append()` |
| Memory prefetch | User message content modified (memory injected) | `agent/memory_manager.py:prefetch_all()` |
| Tool call | Tool may mutate filesystem, network, etc. | tool implementation |
| Tool result | `messages` list extended in-process | `run_agent.py:messages.append()` |
| LLM response | `messages` list extended; streamed to stdout | `run_agent.py:messages.append()` |
| Turn end | Final response row in SQLite | `hermes_state.py:SessionDB.insert_message()` |
| Token tracking | `sessions` row updated (input/output tokens) | `hermes_state.py:SessionDB.update_session()` |
| Memory sync | External provider updated (Honcho, Hindsight, Mem0) | `agent/memory_manager.py:sync_all()` |

---

## Error Propagation

### LLM API Errors
- Handled by `tenacity` retry decorator around API calls
- `agent/error_classifier.py` classifies errors (rate limit, context overflow, auth failure)
- Rate limits: exponential backoff, up to N retries (configured)
- Context overflow: triggers context compression before retry
- Auth failure: propagated to user immediately

### Tool Execution Errors
- Each tool call wrapped in try/except inside `_execute_tool_calls_*`
- Tool errors return error message as tool result (model sees the error and can decide to retry/adapt)
- Tool timeouts: **no per-tool timeout enforced** — known production gap
- Interrupted: `Ctrl+C` sets interrupt flag via `tools/interrupt.py:set_interrupt()`, checked at loop head

### SQLite Write Contention
- `hermes_state.py`: jittered backoff (20–150ms) on `OperationalError` (locked DB)
- Up to N retries; after all retries fail, exception propagates to turn end handler

### Cron Tick Errors
- `cron/scheduler.py:tick()` wrapped in try/except in `_start_cron_ticker`
- Individual job errors logged as `DEBUG`, do not crash the cron thread
- The file lock (`.tick.lock`) is released in a `finally` block

---

## Memory Prefetch + Inject Pipeline

```
Before each user turn:
  MemoryManager.prefetch_all(user_message)
    → BuiltinMemoryProvider.prefetch():
        read MEMORY.md → recent entries → return text block
    → ExternalProvider.prefetch() (if configured):
        network call to Honcho/Hindsight/Mem0
        → returns recalled memory text
    → build_memory_context_block(recalled_texts):
        wraps in <memory-context>...</memory-context> tags
        appends to user_message content

Result: user sees their own message; model sees:
  [MEMORY CONTEXT]
  <memory-context>
  ... recalled facts ...
  </memory-context>
  
  [USER MESSAGE]
  <user message text>
```

**Important**: Memory injection happens AFTER the session prompt is built (system prompt is immutable). The user message content is the injection vehicle.

---

## Context Compression Trigger

```
After each LLM response:
  ContextEngine.should_compress(current_token_count):
    → if current_token_count / context_length >= threshold_percent:
        return True

If True:
  ContextCompressor.compress(messages):
    → Keep: system prompt (always), last N messages (recent context)
    → Summarize: middle messages via auxiliary LLM (cheaper model)
    → New session created with parent_session_id = original session
    → Compressed messages replace original message list
    → [CONTEXT COMPACTION — REFERENCE ONLY] marker inserted
```

**Default threshold**: 0.75 (class default); CLI may configure differently via `context.threshold_percent`.

---

## Delegation (Child Agent) Flow

```
Parent agent turn: model calls delegate_task(task="...", tools=[...])

delegate_tool.py:
  → save: _parent_tool_names = list(model_tools._last_resolved_tool_names)
  → child_agent = AIAgent(...)  # fresh instance, isolated session
    → child runs its own run_conversation() to completion
  → restore: model_tools._last_resolved_tool_names = _parent_tool_names
  → return child.final_response to parent as tool result

Parent loop continues with child's response as a tool result message.
```

**Isolation**: Child has its own `messages` list, `SessionDB` session, and iteration budget. Parent and child share the process (same Python process, different stack frames). The process-global `_last_resolved_tool_names` is the only shared mutable state — the save/restore handles single-threaded delegation but creates a race window for concurrent gateway delegation.

---

## TUI Mode Architecture

```
Node.js process (Ink/React):          Python process (tui_gateway):
  renders terminal UI            <→>    manages AIAgent sessions
  listens for keypresses               handles tool execution
  
Communication: JSON-RPC over stdio pipes

Message flow:
  User types → Ink captures keypress
  → JSON-RPC request: {"method": "chat", "params": {"message": "..."}}
  → Python tui_gateway receives → creates/retrieves AIAgent → run_conversation()
  → Streams responses back as JSON-RPC notifications
  → Ink renders streaming tokens in real-time
```

**Process lifecycle**: Node.js spawns Python as a child process via `child_process.spawn`. Node is the process owner. If Node exits, Python child gets SIGHUP.

---

## Cron Job Execution Flow

```
Every 60 seconds (cron-ticker thread):
  cron/scheduler.py:tick()
    → acquire .tick.lock (fcntl, non-blocking)
    → load ~/.hermes/cron/jobs.json
    → for each job: check if due (croniter)
    → for due jobs:
        → create fresh AIAgent()
        → run_conversation(job.prompt)
        → write output to ~/.hermes/cron/output/{job_id}/{timestamp}.md
        → if job.delivery: send result to gateway home channel
    → release .tick.lock
```

**Home channel delivery**: Cron output can be delivered to the user's "home" messaging platform (e.g., Telegram DM). Configured via `_HOME_TARGET_ENV_VARS` in `cron/scheduler.py`.

---

## Shutdown Sequence

### CLI
- `Ctrl+C` → `KeyboardInterrupt` in REPL → prompt_toolkit handles → clean exit
- Active tool call interrupted: `set_interrupt()` → checked at loop head on next iteration

### Gateway
- SIGTERM → asyncio cancels all tasks → `cron_stop.set()` → `cron_thread.join(timeout=5)` → exit
- In-flight agent sessions: current tool call may complete (no pre-emption); session marked as ended in SQLite

### Crash Recovery
- No explicit crash recovery mechanism
- SQLite WAL mode ensures DB consistency across crashes
- `parent_session_id` allows linking pre-crash sessions to post-restart sessions (manual recovery)
- Session `end_reason` column can record why a session ended
