# 05 — Interview Extraction: hermes-agent

## What Is Interview-Worthy in This Repo

- The **self-registering tool registry** with AST-based discovery is an elegant solution to a common "global list maintenance" problem.
- The **pluggable context engine** as a clean ABC with lifecycle hooks is a strong example of the Open/Closed Principle in a production system.
- The **single COMMAND_REGISTRY** that derives all downstream consumers (5+ interfaces) demonstrates DRY in a multi-interface system.
- The **memory fencing** (`<memory-context>` injection pattern) is a concrete solution to the prompt injection risk from retrieved context — interview-ready example of defensive AI system design.
- The **`HERMES_HOME` profile isolation** via env var with zero code duplication shows how to add multi-tenancy to an existing system without invasive changes.
- The **prompt caching invariant** as an architectural constraint shows mature thinking about LLM cost and latency in production.
- The **SQLite + WAL + FTS5** stack as a "good enough" persistence layer for agentic systems is a defensible architectural choice worth discussing.
- The **delegation pattern** (child AIAgent isolation) is a concrete implementation of hierarchical multi-agent orchestration.

---

## 10 Likely System Design Questions

**Q1: Design a persistent memory system for an AI agent.**
- Key points: Semantic (file-based) vs episodic (conversation) vs external (provider) tiers
- Fencing to prevent recalled memory from being treated as user discourse
- Single external provider constraint to avoid tool schema bloat
- Sync model: async post-turn writes, pre-turn prefetch
- Evidence: `agent/memory_manager.py`, `tools/memory_tool.py`

**Q2: Design a context window management system for a long-running agent.**
- Key points: Threshold-based trigger (75% of context limit)
- Two-phase approach: tool result pruning → auxiliary LLM summarization
- Protect first N and last N messages
- Session splitting via `parent_session_id` chain
- Compression summary framing ("different assistant instance")
- Evidence: `agent/context_engine.py`, `agent/context_compressor.py`

**Q3: Design a multi-platform messaging gateway for an AI agent.**
- Key points: Platform adapters behind a common interface
- Per-session agent cache (LRU + TTL eviction)
- Shared command registry across all platforms
- Cron delivery to "home channels"
- Evidence: `gateway/run.py`, `hermes_cli/commands.py`

**Q4: Design a tool registration system for an LLM agent that supports 50+ tools.**
- Key points: Self-registration at module level (no central list)
- AST-based discovery to avoid importing all modules eagerly
- Toolset grouping for platform-specific tool sets
- Check functions for conditional availability (API key present?)
- Evidence: `tools/registry.py`, `toolsets.py`

**Q5: Design a scheduled autonomous task system for an AI agent.**
- Key points: Cron expression + natural language schedules
- File-based locking to prevent concurrent ticks
- Multiple delivery targets (messaging platforms + local file)
- SILENT marker for audit-log-only jobs
- Evidence: `cron/scheduler.py`, `cron/jobs.py`

**Q6: Design a credential management system for an agent that supports 20+ LLM providers.**
- Key points: Credential sources (env, file, OAuth, CLI tool, config)
- Removal contracts that clean up external state
- Credential pool for round-robin / failover between API keys
- Evidence: `agent/credential_pool.py`, `agent/credential_sources.py`

**Q7: Design an agent system that can improve itself over time.**
- Key points: Skills as procedural memory extracted from completed tasks
- Nudge mechanism for proactive skill creation
- Session search (FTS5) for cross-session recall
- Skills Hub for community sharing (agentskills.io standard)
- Evidence: `tools/skill_manager_tool.py`, `skills/` directory, `hermes_state.py`

**Q8: Design a multi-agent delegation system within a single process.**
- Key points: Child AIAgent isolation (own state, tool access, iteration budget)
- Process-global state (tool names) save/restore around child runs
- Parallel delegation via ThreadPoolExecutor
- No shared memory between parent and child by default
- Evidence: `tools/delegate_tool.py`, `run_agent.py:_MAX_TOOL_WORKERS`

**Q9: How would you handle LLM provider failover in a production agent?**
- Key points: Error classification (rate limit vs auth vs network vs context limit)
- `FailoverReason` enum driving different recovery strategies
- Context-limit errors trigger compression rather than failover
- Auth failures try next credential in pool before switching providers
- Evidence: `agent/error_classifier.py`, `agent/credential_pool.py`

**Q10: Design a session storage system for a conversational AI agent deployed across 16 messaging platforms.**
- Key points: SQLite WAL for multi-reader, single-writer concurrency
- FTS5 for full-text search across message history
- Session tagging by source platform
- Parent/child chains for compression-split sessions
- Jittered backoff for write contention
- Evidence: `hermes_state.py`

---

## 10 Likely Debugging / Reliability Questions

**Q1: Users report the agent "forgets" its current task mid-conversation. How do you debug this?**
- Answer bullets: Check if compression fired unexpectedly (token threshold too low); verify compression summary quality (look at `[CONTEXT COMPACTION]` messages in session history); check if auxiliary LLM is producing low-quality summaries; verify `protect_last_n` setting preserves enough recent context.

**Q2: A tool call returns an error claiming the tool doesn't exist. What could cause this?**
- Answer bullets: Tool module failed to import at startup (check `logging.WARNING` for "Could not import tool module"); tool is in a disabled toolset; schema description references a cross-tool name that doesn't match the registered name; MCP server returned a tool schema with wrong name.

**Q3: The gateway becomes unresponsive after several hours of operation. How do you diagnose?**
- Answer bullets: Check agent cache eviction — if >128 sessions are active, LRU eviction is destroying live sessions; check `_AGENT_CACHE_IDLE_TTL_SECS` — sessions idle >1h are evicted; check memory growth if tools create large state; check for stuck tool calls that never return (no per-tool timeout).

**Q4: Cron jobs are firing twice. What's the mechanism and fix?**
- Answer bullets: `.tick.lock` file lock prevents overlapping ticks, but job execution inside a tick runs in parallel threads — two DIFFERENT ticks can't overlap, but if a job takes longer than 60s, the next tick starts a second instance; fix: add per-job execution lock.

**Q5: An agent conversation costs 10x more than expected. What do you investigate?**
- Answer bullets: Check if prompt caching is working (cache_read_tokens > 0 in usage); check if toolset changed mid-conversation (breaks cache); check if system prompt is being rebuilt unnecessarily; check compression count (each compression is a full auxiliary LLM call); check for token-heavy tool outputs bloating context.

**Q6: Skills are being applied to the wrong platform/context. How do you diagnose?**
- Answer bullets: Check skill frontmatter for platform conditions (`skill_matches_platform()` in `agent/skill_utils.py`); check if skill files have correct `platforms:` field; verify `get_disabled_skill_names()` is reading from correct profile's config.

**Q7: `state.db` grows indefinitely and disk fills up. What's happening?**
- Answer bullets: No automatic message pruning in the base schema; FTS5 triggers add `messages_fts` entries for every message; if many cron jobs run with large outputs, each turn is persisted; solution: add a session retention policy or archive old sessions.

**Q8: A user reports the agent is ignoring instructions in their AGENTS.md. How do you debug?**
- Answer bullets: Check `_scan_context_content()` — if AGENTS.md contains anything matching `_CONTEXT_THREAT_PATTERNS`, the file is sanitized/rejected; check if AGENTS.md contains invisible unicode characters (`_CONTEXT_INVISIBLE_CHARS`); verify the file is in the correct working directory.

**Q9: Memory content from one user is appearing in another user's session on the gateway. What happened?**
- Answer bullets: Profile isolation not configured — multiple users sharing the same `HERMES_HOME`; external memory provider not segmenting by user_id; memory tool writing to shared `MEMORY.md` without user-scoped path.

**Q10: A delegation chain is producing wrong tool outputs. How do you trace the issue?**
- Answer bullets: Check `_last_resolved_tool_names` global — if two concurrent delegations are running, one may have corrupted the other's tool schema view; check that `_run_single_child()` save/restore is working; verify child agent was initialized with correct toolset.

---

## 10 Likely Architecture Trade-Off Questions

**Q1: Why SQLite instead of Postgres for session storage?**
- Answer bullets: SQLite has zero deployment dependencies (no external service); WAL mode handles the concurrent-readers pattern; FTS5 is built-in; sufficient for personal/small-team use; trade-off: horizontal scaling requires external DB; the design explicitly acknowledges this is a "good enough" choice for the target deployment scale.

**Q2: Why is the agent loop synchronous instead of async?**
- Answer bullets: Tool execution is inherently sequential within a turn (model calls tools, waits for results, decides next action); async wouldn't help within the loop; gateway uses threading (one thread per active session) rather than async coroutines; trade-off: high-concurrency deployments need more threads than coroutines would require.

**Q3: Why are skills injected as user messages instead of system prompt?**
- Answer bullets: System prompt changes invalidate Anthropic's prompt cache; user messages don't break cached prefixes; trade-off: model may give skills less authority than system-level instructions; skills are treated as conversational input rather than permanent rules.

**Q4: Why is there only one external memory provider allowed at a time?**
- Answer bullets: Multiple providers would expose multiple tool schemas (get_tool_schemas() per provider), bloating the context; conflicting memory backends create inconsistent user models; simplicity of single-source-of-truth for user modeling; trade-off: users who want both semantic retrieval (Mem0) and dialectic modeling (Honcho) can't use both.

**Q5: Why use an auxiliary LLM for context compression instead of deterministic truncation?**
- Answer bullets: LLM summarization preserves task context across compressions; deterministic truncation loses information in a binary way; the model can track resolved/pending questions; trade-off: compression depends on auxiliary LLM availability and quality; cost of compression call; potential for hallucinated summary.

**Q6: Why is the tool registry self-registering with AST discovery rather than a central import list?**
- Answer bullets: Eliminates maintenance overhead of keeping a central list in sync; AST scan catches registration at module body level only (prevents false positives); trade-off: slightly slower startup (AST parsing all tool files); less explicit than a central list for new developers.

**Q7: Why is the TUI implemented as a Node.js Ink process over stdio instead of a pure Python TUI?**
- Answer bullets: Ink provides React-like declarative UI for terminal (easier to build rich interactive components); Python TUI libraries (Textual, urwid) are less mature for the use case; trade-off: two-process model adds complexity (JSON-RPC bridge); Node.js dependency for users who just want the TUI.

**Q8: Why is the command registry a single global list rather than a plugin-based registration system?**
- Answer bullets: Commands are a finite, known set — not extensible by third parties; single source of truth prevents divergence across interfaces; trade-off: less flexible for plugins that want to add their own slash commands; central list can grow large.

**Q9: Why is prompt caching treated as an architectural invariant rather than an optimization?**
- Answer bullets: Cache misses at Claude's pricing (75% cost reduction on cache hits) mean unnecessary cache-breaking can double or triple costs in multi-turn conversations; treating it as a correctness constraint forces better context discipline; trade-off: limits mid-conversation context mutations (e.g., can't reload memories on demand).

**Q10: Why is the profile isolation implemented via HERMES_HOME env var instead of a database-level tenant ID?**
- Answer bullets: Filesystem isolation is simpler and more complete — each profile has isolated config, keys, memory, logs, and skills; no code changes needed for new state types (just call `get_hermes_home()`); trade-off: can't easily query across profiles; not suitable for multi-user server deployments where profiles share infrastructure.

---

## 5 "What Would You Improve?" Prompts

**P1: The compression system is a single point of failure.**
Improvements: Add a fallback simple-truncation mode when the auxiliary LLM is unavailable. Add compression pre-flight at 60% (not 75%) to buy more headroom. Consider caching compression summaries so they don't need recomputation if the session is resumed.

**P2: The tool execution model has no per-tool timeouts.**
Improvements: Add a configurable timeout per tool (or per toolset) in the registry. Implement a circuit breaker for tools that frequently time out. Surface slow-tool metrics via the web dashboard.

**P3: SQLite doesn't scale to multi-user deployments.**
Improvements: Abstract the session store behind an interface (already partially done via `SessionDB` class). Implement a Postgres backend for production deployments. Add a connection pool. Consider message archival (move old sessions to cold storage).

**P4: The skills system has no quality management.**
Improvements: Add a skill quality score (based on recency, usage frequency, and length). Surface low-quality skills for review. Add deduplication detection (semantic similarity check between new and existing skills). Integrate with Skills Hub for community ratings.

**P5: The process-global tool names variable is a latent concurrency bug.**
Improvements: Refactor `_last_resolved_tool_names` to be passed as an argument rather than read from a global. Add a thread-local storage fallback. Add a test that specifically exercises concurrent delegation to catch regressions.
