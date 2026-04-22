# 15 — Trade-off Analysis: hermes-agent

This document makes explicit the trade-offs that are baked into hermes-agent's architecture. Each dimension shows what was chosen, what was sacrificed, and the evidence.

---

## 1. Synchronous Agent Loop vs Async Agent Loop

**Choice**: Synchronous (`def run_conversation()`, no `async def`)

**Evidence**: `run_agent.py:8464` — plain `def`; zero `await` in loop body

**What you get:**
- Simple reasoning about state: no async infection, no race conditions in the agent loop
- Debuggability: synchronous stack traces are readable
- No event loop management complexity
- Blocking operations are safe to use everywhere

**What you sacrifice:**
- Cannot interleave agent work with I/O (e.g., can't start a second LLM call while a tool runs)
- Memory sync (`sync_all()`) blocks turn completion — slow providers add latency
- Tool execution, even with `ThreadPoolExecutor`, returns to a synchronous join before proceeding

**When this hurts**: When external memory providers have high latency. When tool execution involves waiting for external services and you want the agent to make progress on other work.

**Verdict**: Correct choice for the current use case. The system is single-agent-per-conversation. Async complexity would be wasted here and would make the codebase significantly harder to maintain.

---

## 2. SQLite vs Postgres

**Choice**: SQLite with WAL mode and FTS5

**Evidence**: `hermes_state.py:SCHEMA_SQL` — `PRAGMA journal_mode=WAL`; `FTS_SQL`

**What you get:**
- Zero infrastructure: no separate database server to install, configure, or maintain
- FTS5 full-text search built in — semantic search across all conversation history without a vector store
- WAL mode: concurrent readers + single writer without blocking reads
- Embedded: works on Termux (Android), offline, in Docker with no external dependencies

**What you sacrifice:**
- Single-writer architecture: write contention increases linearly with concurrent sessions
- No replication: no read replicas, no horizontal scaling
- No LISTEN/NOTIFY: gateway can't efficiently watch for external events
- No pg_trgm or vector extensions without plugin

**When this hurts**: Multi-user deployments (10+ concurrent sessions writing simultaneously). Teams needing HA/failover. Deployments where message volume grows to millions of rows.

**Mitigation designed in**: `SessionDB` is the only interface to storage — the abstraction for a Postgres swap exists, just not implemented.

**Verdict**: Correct for single-user and small team use. The zero-infra benefit is significant for an agent platform that must run on developer laptops, Raspberry Pi, and Android. Build the Postgres backend when you have 50+ users, not before.

---

## 3. Custom Agent Framework vs LangChain / LlamaIndex

**Choice**: Custom (`run_agent.py` is the framework)

**Evidence**: First-pass doc — "Hermes does not use an external agentic framework"

**What you get:**
- Full control over every aspect of the agent loop, tool execution, and context management
- No dependency on framework upgrade cycles breaking the system
- Can make unconventional architecture choices (synchronous core, prompt cache invariant, etc.)
- Deep optimization for specific use cases (gateway mode, self-improvement)

**What you sacrifice:**
- Must maintain all agent infrastructure internally
- Cannot use community tools built for LangChain/LlamaIndex ecosystems
- Onboarding requires learning Hermes internals (vs recognizable LangChain patterns)
- Every new feature (streaming, retries, multi-modal) must be built from scratch

**When this hurts**: When you want to integrate with a tool that's "LangChain-native." When you want to use pre-built evaluation frameworks. When team members have LangChain expertise.

**Verdict**: Correct given the self-improvement and multi-platform goals. LangChain's tool system wouldn't fit the gateway architecture. The tradeoff is appropriate — the framework IS the product.

---

## 4. Prompt Cache Invariant vs Flexible System Prompt

**Choice**: System prompt immutable during a session

**Evidence**: AGENTS.md explicit design rule; `_build_system_prompt()` called once at session start

**What you get:**
- ~75% cost reduction on Anthropic models via cache hits on cached system prompt segments
- Predictable agent behavior: no mid-conversation behavior drift from system prompt changes
- Simple reasoning about what the agent "knows" — the system prompt is deterministic

**What you sacrifice:**
- Cannot inject mid-session dynamic context into the system prompt (permissions changes, live config updates)
- Memory written mid-session doesn't update the system prompt until the next session
- Skills invoked mid-conversation inject via user message (not system prompt), slightly different semantics

**When this hurts**: When you want to dynamically update the agent's capabilities mid-session (e.g., user upgrades their permission level). When skills need to behave as "first-class" system instructions rather than user-level injections.

**Verdict**: Correct at current scale. 75% cost reduction is substantial. The tradeoff is worth accepting — the use cases that need mid-session system prompt mutation are edge cases, and the cost-saving benefit serves every session.

---

## 5. Skills as User Messages vs System Prompt Additions

**Choice**: Skill invocations inject content as user-level messages; skills list as system prompt metadata

**Evidence**: `agent/skill_commands.py:429–465` — `build_skill_invocation_message()` returns user message content; `cli.py:6172–6179` shows it going to `_pending_input.put(msg)` (user input queue)

**What you get:**
- System prompt remains unchanged when skills are invoked → prompt cache stays valid
- Skills can be created and invoked dynamically without session restart
- Skill content is part of the conversation history (visible in session search)

**What you sacrifice:**
- Skills don't have the semantic authority of system prompt instructions
- A sufficiently complex conversation context could dilute skill instructions
- The model might treat the skill as user content rather than a hard directive

**When this hurts**: Skills that need to establish hard behavioral constraints (e.g., security policies, always-on rules). These work better as system prompt additions, but that breaks the cache.

**Verdict**: Reasonable given the cost constraint, but creates a semantic ambiguity. The `[SYSTEM: The user has invoked...]` activation note in skill messages is a workaround, not a fix.

---

## 6. File-Based Memory vs External Vector Store

**Choice**: MEMORY.md (flat file) + SQLite FTS5 as the default memory layer; vector store as optional external provider

**Evidence**: `agent/memory_manager.py:BuiltinMemoryProvider`; `pyproject.toml:mem0`, `honcho`, `hindsight` as optional extras

**What you get:**
- Zero infrastructure for default memory: works offline, no API key, no extra service
- MEMORY.md is human-readable and user-editable — transparency over opacity
- FTS5 provides keyword-based recall across all conversation history

**What you sacrifice:**
- No semantic (embedding-based) retrieval in the default layer
- MEMORY.md can grow unbounded without built-in summarization or pruning
- Two agents writing MEMORY.md concurrently (e.g., gateway mode, same user) risk garbled output
- FTS5 keyword match can miss semantically relevant memories without exact term overlap

**When this hurts**: Long-running agents that accumulate large MEMORY.md files. Use cases where semantic similarity ("I want things related to Python performance") matters more than keyword match.

**Verdict**: Correct default for developer-use cases. Power users should enable Mem0 or Hindsight for semantic recall. The file-based default makes the memory layer transparent and debuggable.

---

## 7. Filesystem Skills vs Database-Managed Skills

**Choice**: Skills are markdown files in `~/.hermes/skills/`

**Evidence**: Skill creation writes to filesystem; `ls ~/.hermes/skills/` to inspect

**What you get:**
- Human-readable: anyone can inspect, edit, or delete a skill directly
- Git-friendly: skills can be version-controlled
- No schema required: skills are unstructured markdown

**What you sacrifice:**
- No deduplication: similar skills accumulate silently
- No quality scoring: bad skills persist alongside good ones
- No search across skill metadata without grep
- Concurrent skill writes from gateway mode can race

**When this hurts**: When the agent self-creates many skills over time. Without curation, the skills directory becomes noisy and skills with typos or outdated patterns continue being loaded.

---

## 8. Parallel Tool Execution vs Sequential Execution

**Choice**: Parallel when `_should_parallelize_tool_batch()` returns True; sequential otherwise

**Evidence**: `run_agent.py:299–340` — `_NEVER_PARALLEL_TOOLS`, `_PARALLEL_SAFE_TOOLS`, path overlap check; `_MAX_TOOL_WORKERS = 8`

**What you get:**
- Significant speedup for batches of independent read operations (multi-URL fetch, multi-file read)
- Safety guarantee: destructive or state-mutating tools default to sequential
- Deterministic behavior for path-overlapping file operations

**What you sacrifice:**
- Complexity: the safety logic (`_should_parallelize_tool_batch`) must be kept in sync with each new tool's safety characteristics
- A tool incorrectly placed in `_PARALLEL_SAFE_TOOLS` creates a race condition
- Debugging parallel tool failures is harder than sequential

**When this hurts**: When adding a new tool that has subtle state-sharing behavior not obvious from its name/category.

---

## 9. In-Process Cron Scheduler vs External Queue (Redis/Celery)

**Choice**: `threading.Thread` cron ticker in gateway process + file-based lock

**Evidence**: `cron/scheduler.py:tick()` + `gateway/run.py:_start_cron_ticker()` — daemon thread, fcntl lock

**What you get:**
- Zero external dependencies: no Redis, no Celery, no broker to manage
- Works offline and in simple deployments
- Cron results available within the same process for delivery to home channels

**What you sacrifice:**
- Single-process limitation: if the gateway crashes, pending cron jobs are lost (no durable queue)
- No distributed scheduling: multiple gateway instances would double-tick jobs
- File-based locking works on single host but not across hosts
- Long-running cron jobs block the cron thread

**When this hurts**: Multiple gateway instances (horizontal scaling). Cron jobs that take longer than 60 seconds (the next tick fires while the previous job runs). High-availability deployments where cron reliability is critical.

**Verdict**: Correct for single-server deployments. The file-based lock prevents double-execution on a single host. For HA, the architecture would need a distributed scheduler.
