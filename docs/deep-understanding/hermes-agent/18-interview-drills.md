# 18 — Interview Drills: hermes-agent

Organized by difficulty. Each question includes what a strong answer covers.

---

## Level 1 — Foundational (10 questions)

These test whether you understand what the system does.

**Q1: What is hermes-agent?**
Strong answer covers: self-hosted AI agent framework, custom agent loop (no LangChain), multi-platform messaging gateway, self-improvement via skills, persistent memory across sessions, 20+ LLM providers. Mention: Nous Research, open source.

**Q2: What are the five main ways to interact with Hermes?**
Strong answer: CLI (`hermes`), Ink TUI (`hermes --tui`), messaging gateway (Telegram/Slack/Discord/etc.), ACP adapter (IDE integration), batch runner (parallel agent jobs). Each instantiates `AIAgent`.

**Q3: Describe the core agent loop in one sentence.**
Strong answer: "A synchronous while-loop that sends conversation history to an LLM, executes any tool calls, appends results, and repeats until the model returns a pure-text response or the iteration limit is reached."

**Q4: What is `HERMES_HOME` and why does it matter?**
Strong answer: An env var that overrides the default `~/.hermes` directory for all state (config, memory, sessions, API keys, skills). Enables profile isolation — `HERMES_HOME=/tmp/test hermes` creates a completely isolated agent instance. The `_isolate_hermes_home` test fixture uses this to prevent tests from writing to real user data.

**Q5: How does hermes manage multiple LLM providers?**
Strong answer: Provider abstraction built on the OpenAI-wire format. All non-native providers use the `openai` SDK (which speaks OpenAI-format to any compatible endpoint). Native Anthropic SDK used for Anthropic-specific features (prompt caching, extended thinking). Provider selected via `model.provider` config key.

**Q6: What is a "skill" in hermes?**
Strong answer: A markdown file in `~/.hermes/skills/` containing instructions. Invoked with a slash command (`/skill-name`). When invoked, the skill content is injected into the conversation as a user message (preserving system prompt cache). Skills can be LLM-generated (the agent creates its own). Available skills list is included as metadata in the system prompt.

**Q7: What is the purpose of `tools/registry.py`?**
Strong answer: Manages tool registration and discovery. Tools self-register by calling `registry.register()` at module level. Discovery uses `ast.parse()` to find these calls without executing the modules (avoiding circular imports and side effects). `get_tool_definitions()` converts registered tools to OpenAI-format tool schemas for the LLM API call.

**Q8: What is context compression and when does it fire?**
Strong answer: When the conversation grows to 75% of the model's context window (configurable via `threshold_percent`), an auxiliary (cheaper) LLM summarizes the middle portion of the conversation. The summary replaces the original messages. The compressed session gets a `parent_session_id` linking it to the original. The compression is transparent to the user.

**Q9: What storage technologies does hermes use?**
Strong answer: SQLite with WAL mode (session persistence, message history), SQLite FTS5 (full-text search across all conversations), filesystem markdown files (MEMORY.md, USER.md, skills), YAML files (config), JSON files (cron jobs, auth), dotenv files (API keys).

**Q10: What is the gateway mode?**
Strong answer: A long-lived Python process that listens to 16+ messaging platforms (Telegram, Slack, Discord, etc.) and routes messages to per-user `AIAgent` instances. Agents are cached in an LRU cache (max 128, 1-hour TTL). Includes a background cron thread for scheduled autonomous tasks.

---

## Level 2 — Intermediate (10 questions)

These test whether you understand architectural decisions.

**Q11: Why is the agent loop synchronous instead of async?**
Strong answer: The agent loop is a sequential conversation — each turn requires the previous turn's LLM response before starting. Async adds complexity without benefit for single-agent conversations. Async IS used for platform-specific adapters (Telegram, Discord) and the gateway event loop — but the agent loop itself stays synchronous for simplicity, debuggability, and clean reasoning about state.

**Q12: Why can't the system prompt be changed mid-conversation?**
Strong answer: Prompt caching invariant. Anthropic models cache the system prompt and return a cache hit (~75% cost reduction) when it doesn't change. Any mid-session system prompt mutation would break the cache prefix, forcing re-evaluation of the entire system prompt every turn. This architectural constraint is enforced by building the system prompt once at session start.

**Q13: How do parallel tool executions work, and what are the safety constraints?**
Strong answer: `_should_parallelize_tool_batch()` decides per-batch. Returns False if: only one tool, any tool is in `_NEVER_PARALLEL_TOOLS` (destructive tools), any tool is not in `_PARALLEL_SAFE_TOOLS`, or path-overlapping file operations detected. When parallel: `ThreadPoolExecutor(max_workers=min(n_tools, 8))`. The safety logic prevents race conditions on shared resources.

**Q14: How does the gateway know when to evict a cached agent?**
Strong answer: `OrderedDict` for the cache — oldest entries at the front. `_enforce_agent_cache_cap()` runs when cache size exceeds 128. It iterates `list(_cache.keys())[:excess]` (LRU order), skips entries in `_running_agents` (currently mid-turn), and pops the rest. TTL-based eviction (1-hour idle) runs as a background daemon thread.

**Q15: What is `_last_resolved_tool_names` and why is it potentially dangerous?**
Strong answer: A module-level (process-global) list in `model_tools.py`. Written by `get_tool_definitions()` after every tool resolution. Read by `delegate_tool.py` to save/restore parent tool context around child agent runs. The save/restore pattern is correct for single-threaded delegation, but in gateway mode with concurrent sessions, multiple threads may interleave their saves and restores, causing one agent to restore the wrong tool list.

**Q16: How does memory injection work and how is it protected against prompt injection?**
Strong answer: Before each turn, `MemoryManager.prefetch_all()` fetches recalled memory from providers. The recalled text is wrapped in `<memory-context>...</memory-context>` tags. This tag boundary signals to the model that the content is recalled memory, not user instruction — preventing injected memory from being treated as a command. The model's interpretation of the fence is a design convention, not a hard technical barrier.

**Q17: How does skill invocation preserve the prompt cache?**
Strong answer: Skill content is injected as a user message (not the system prompt). The system prompt includes only a list of available skill names as metadata — this is set at session start and doesn't change. When a user types `/skill-name`, the skill body is added to the user message for that turn. Since the system prompt is unchanged, the Anthropic prompt cache hit is preserved.

**Q18: What are the two config loaders and why do they diverge?**
Strong answer: `load_cli_config()` (in `cli.py`) — used by the interactive CLI; falls back to `./cli-config.yaml`, includes personality templates. `load_config()` (in `hermes_cli/config.py`) — used by the library/gateway; only reads `~/.hermes/config.yaml`, performs backward-compatibility migration (`max_turns` → `agent.max_turns`), uses `_deep_merge()`. They diverged because CLI and gateway were developed somewhat independently. Any new config option requires updating both.

**Q19: Why does Hermes use FTS5 instead of a vector database for session search?**
Strong answer: FTS5 is built into SQLite — zero extra infrastructure. For keyword-based recall ("find conversations about Python performance"), BM25 full-text search is effective. No embedding model or vector store server needed. The tradeoff: no semantic similarity search. If you need "related topics" recall, external providers (Mem0, Honcho, Hindsight) add vector/embedding capabilities as optional plugins.

**Q20: How does context compression create a session chain?**
Strong answer: When compression fires, a new session is created in the `sessions` table with `parent_session_id` pointing to the original session. The compressed messages (summary + recent) become the new session's history. The `parent_session_id` foreign key + index allows reconstruction of the full compression chain. The `[CONTEXT COMPACTION — REFERENCE ONLY]` marker in the conversation helps the agent understand that earlier context was summarized.

---

## Level 3 — Advanced (10 questions)

These test architectural depth and critical thinking.

**Q21: How would you fix the `_last_resolved_tool_names` race condition?**
Strong answer: Replace the module-level list with `threading.local()` — each thread gets its own copy. `get_tool_definitions()` would write to `threading.local().last_resolved_tool_names`. Delegation save/restore in `delegate_tool.py` would read from the same thread-local. This eliminates the cross-thread interference. The alternative (passing tool names explicitly through the call stack) would require more invasive changes but is architecturally cleaner.

**Q22: What would you change to make memory sync non-blocking?**
Strong answer: In `run_agent.py`, after generating the final response, instead of calling `self._memory_manager.sync_all()` directly, submit it to a `concurrent.futures.ThreadPoolExecutor` (one-shot, fire-and-forget). The response is returned immediately to the user. Accept: turn N memory may not be visible to turn N+1 prefetch if N+1 starts before the background sync completes. Mitigation: use a per-session queue to serialize syncs for a given user.

**Q23: Hermes uses SQLite for session storage. Describe a Postgres migration path.**
Strong answer: `SessionDB` (in `hermes_state.py`) is the abstraction layer — all DB operations go through it. Steps: (1) implement `PostgresSessionDB` class with the same interface; (2) add a config option `storage.backend: sqlite|postgres`; (3) add connection pooling (asyncpg or psycopg2 with pooling); (4) handle FTS differently — Postgres full-text search uses `tsvector`/`tsquery`, not FTS5. Migration of existing data: write a one-time migration script that reads from SQLite and bulk-inserts to Postgres.

**Q24: How would you add per-tool timeout enforcement?**
Strong answer: (1) Add `timeout_seconds: Optional[float] = None` to `ToolEntry` in `tools/registry.py`. (2) Tools annotate their registration with a timeout. (3) In `_execute_single_tool()` in `run_agent.py`, wrap the call in `concurrent.futures.ThreadPoolExecutor.submit(tool_func, ...).result(timeout=entry.timeout_seconds)`. (4) Catch `concurrent.futures.TimeoutError` and return an error tool result. (5) The model sees the timeout error and can decide to retry or abandon.

**Q25: The system prompt is cached. What happens if a user's permissions change mid-session?**
Strong answer: The current design doesn't handle this case — permission changes would only take effect at the start of the next session (when the system prompt is rebuilt). This is a known limitation of the prompt cache invariant. Options: (1) Accept the limitation and document it; (2) include a permission "version" in the system prompt and invalidate the cache on version change; (3) use a separate channel (tool result or system message injection) to communicate permission changes without modifying the cached system prompt prefix.

**Q26: How does hermes handle a model that returns malformed tool call JSON?**
Strong answer: The `_should_parallelize_tool_batch()` function includes JSON validation of tool call arguments before dispatch (lines 299–340 check argument parseability). In `_execute_single_tool()`, if argument parsing fails, the error is returned as a tool result (the model sees it and can correct itself). The agent loop doesn't crash — the error becomes part of the conversation context for the LLM to reason about.

**Q27: Why does hermes inject skills as user messages instead of system prompt entries?**
Strong answer: Two reasons: (1) Prompt cache preservation — adding skill content to the system prompt would change the cache key, losing the ~75% cost reduction on every turn. (2) Flexibility — skills can be invoked at any point in a conversation without restarting the session. The tradeoff is semantic authority: user messages have slightly different semantic weight than system prompt entries for some models.

**Q28: How would you add distributed multi-instance gateway support?**
Strong answer: The current single-server architecture has two main blockers: (1) SQLite (single-writer) → needs Postgres with connection pooling; (2) In-memory agent cache (`OrderedDict`) → needs an external cache (Redis) with session affinity or distributed lock. The cron scheduler also needs to be externalized (Redis-based distributed lock or a proper job queue). The `GatewayRunner._agent_cache` interface would need to become a distributed cache client. Platform adapters can run independently once session state is external.

**Q29: What would a robust self-improvement loop look like for the skills system?**
Strong answer: Current state: LLM creates skills on demand, stored as markdown files, no quality control. Improved loop: (1) Skill creation: validate markdown structure (schema), run against a test prompt before saving; (2) Deduplication: embed skill name/description, compare cosine similarity to existing skills, block creation if duplicate above threshold; (3) Quality scoring: after N invocations, prompt a judge model to rate the skill effectiveness; (4) Pruning: `skill doctor` command that removes low-scored skills and merges near-duplicates; (5) Versioning: skills stored with creation date and version history.

**Q30: The gateway caches up to 128 agents. What happens when the 129th user sends a message?**
Strong answer: `_enforce_agent_cache_cap()` is called, which identifies the oldest (LRU) entry in the `OrderedDict` cache. If that entry is not currently processing a message (not in `_running_agents`), it's evicted — the agent object is discarded (garbage collected). The 129th user gets a new agent instance created from scratch. That new agent loads context from SQLite (the session history persists), so the conversation continues from where it left off. The "warmup" cost is the agent initialization, not the conversation history — history is re-loaded.

---

## Level 4 — System Design (5 questions)

These are open-ended design questions.

**Q31: Design a production-ready multi-tenant deployment of hermes.**
Key elements to cover: Postgres backend for sessions, Redis for agent cache (sticky sessions or distributed cache), horizontal gateway instances behind a load balancer, centralized logging (structured JSON → Elasticsearch/Loki), metrics (Prometheus + Grafana), per-tenant rate limiting, per-tenant HERMES_HOME isolation (or per-tenant DB schema), queue-based cron execution (not in-process). Security: tenant data isolation (row-level security in Postgres), tool execution sandboxing (Docker per job), API key management (Vault or AWS Secrets Manager).

**Q32: How would you add real-time observability to hermes?**
Key elements: (1) Structured logging with `structlog` or `python-json-logger` — emit JSON with session_id, user_id, tool_name, latency_ms, token_count; (2) OpenTelemetry traces — wrap `run_conversation()` as a parent span; each tool call a child span; (3) Prometheus metrics — counters for turn count, tool call count, tool errors; histograms for turn latency, token count distribution; (4) `/healthz` endpoint in ACP/gateway; (5) Alerting on: tool error rate > 5%, turn latency p99 > 30s, SQLite write retry rate > 10%.

**Q33: The RL infrastructure suggests Hermes generates training data. How would you build a closed-loop improvement system?**
Key elements: `batch_runner.py` runs N agents in parallel → generates trajectories → `trajectory_compressor.py` formats for training. Add: (1) Quality signal: human preference labels via `/feedback` command; (2) Automated quality: reward model that scores trajectories (was the task completed? how many tool calls?); (3) Filtering: keep top-K trajectories by reward; (4) SFT training via Atropos on filtered data; (5) Deploy updated model; (6) Measure win rate on benchmark tasks before/after. The key challenge is reward modeling — defining "good" agent behavior programmatically.

**Q34: How would you design a plugin system for hermes tools?**
Key elements: Current system requires tools to be in the `tools/` directory. Plugin design: (1) Define a tool plugin manifest (YAML) with: module path, dependencies, tool names, min hermes version; (2) Plugin discovery: scan `~/.hermes/plugins/` for manifests; (3) Installation: `hermes plugin install <url>` downloads module to plugins dir; (4) Security: plugins run in a restricted subprocess (sandboxed); (5) Versioning: pin plugin versions in `~/.hermes/plugins.lock`; (6) `agentskills.io` hub as the distribution channel (already referenced in README).

**Q35: A user reports that after 2 hours of usage, their agent "forgets" recent conversations. Debug the issue.**
Diagnostic approach: (1) Check if context compression fired: `SELECT id, parent_session_id FROM sessions WHERE parent_session_id IS NOT NULL ORDER BY started_at DESC LIMIT 5;` — if many compressed sessions, check summary quality; (2) Check MEMORY.md size: large MEMORY.md may be truncated in the memory context block; (3) Check memory provider: if Honcho/Hindsight is configured, check if `prefetch_all()` is returning stale data; (4) Check session search: `session_search` tool with recent terms — if FTS returns nothing, index may be stale; (5) Verify `sync_all()` timing: if a slow provider is timing out, memory writes may be failing silently (errors are logged as WARNING, not ERROR). The most likely culprit: context compression with a poor auxiliary LLM producing vague summaries.

---

## Level 5 — Behavioral / Trade-off (5 questions)

**Q36: What are the three biggest risks in this codebase and how would you fix them?**
Strong answer: (1) `_last_resolved_tool_names` race → threading.local(); (2) Blocking memory sync → background thread with per-session queue; (3) No per-tool timeouts → add timeout to ToolEntry + Future.result(timeout=...). These three changes would address the most likely production failures under moderate load.

**Q37: You need to add a new tool that reads a large database query result. What concerns do you have?**
Strong answer: (1) Result size — large DB results go into the message list, bloating context; need to summarize or paginate. (2) SQLite contention — if the tool writes to `state.db`, check WAL mode handles concurrent writes from other sessions. (3) Parallel safety — if this tool reads/writes state, ensure it's in `_NEVER_PARALLEL_TOOLS` or `_PARALLEL_SAFE_TOOLS` appropriately. (4) Timeout — DB queries can run indefinitely; add `timeout_seconds` to the tool registration.

**Q38: A team is considering hermes for a 200-user internal deployment. What would you tell them?**
Strong answer: The core agent loop and provider abstraction are production-ready. The blocking points for 200 users: SQLite write contention (needs Postgres), no per-tool timeouts (risky for shared infra), no structured logging or metrics (hard to operate at scale). Estimated effort for Postgres + timeouts + structured logging: 2–3 engineer-weeks. With those three changes, 200 users is manageable. Without them: expect intermittent latency spikes and opaque debugging.

**Q39: Why doesn't hermes use LangChain? What did they gain and lose?**
Strong answer: Gain: Full control over the agent loop, tool system, context management, and memory. No dependency on LangChain upgrade cycles. Can implement non-standard patterns (prompt cache invariant, gateway-mode caching, AST tool discovery). Lose: Community integrations, pre-built evaluation tools, developer familiarity. The fact that Hermes IS the framework (not built on one) means it can optimize for its specific use case — persistent, self-improving, multi-platform agents — without fighting the abstraction.

**Q40: The agent creates its own skills over time. What are the long-term quality implications?**
Strong answer: Without a quality control layer, skill quality degrades over time via: (1) Skill pollution — similar skills accumulate; the LLM may invoke the wrong version; (2) Outdated skills — a skill created with an old API pattern continues being invoked; (3) Noisy memory injection — the available skills list in the system prompt grows, consuming tokens; (4) Hallucination amplification — a skill with incorrect instructions gets treated as authoritative. The architecture supports improvement (skills are files, easy to prune), but the pruning must be intentional. Recommendation: add a skill quality audit (scheduled cron job) that uses a judge model to rate each skill.
