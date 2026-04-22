# Interview Story Bank: hermes-agent

Ready-to-use interview stories derived from hermes-agent architecture. Each story has a setup, technical depth, and reflection.

---

## System Design Stories

### Story SD-1: "Designing a Multi-Platform Agent Gateway"

**Setup**: "Tell me about a time you designed a system that needed to handle multiple interfaces or channels."

**The story**:
"I was studying hermes-agent, which needed to connect one AI agent core to 16+ messaging platforms — Telegram, Slack, Discord, WhatsApp, Matrix, and more.

The challenge: each platform has completely different APIs, authentication flows, message formats, and rate limits. The naive solution would be one giant message handler with a big if/else tree for platform-specific logic.

The design they used was cleaner: a `GatewayRunner` that manages platform adapters via a common interface. Each platform implements `send_message(user_id, text)`, `get_user_info(user_id)`, and `listen()`. The gateway is platform-agnostic — it only speaks the common interface. Adding a 17th platform is implementing the interface and registering it.

The agent side is equally clean: per-user `AIAgent` instances cached in an LRU cache (max 128, 1-hour TTL). The cache key is (platform, user_id). When a message arrives from Telegram user 12345, the gateway looks up or creates an agent for that key.

What I found interesting: they used `OrderedDict` for the LRU cache (not a dedicated LRU library), which made the eviction logic explicit and auditable. The `_enforce_agent_cache_cap()` method walks the ordered keys and evicts oldest entries that aren't mid-turn.

The lesson: multi-interface problems are adapter problems. Define a clean common interface first. Everything else follows."

---

### Story SD-2: "Designing for LLM Cost Optimization"

**Setup**: "How have you thought about cost efficiency in AI systems?"

**The story**:
"Hermes has an unusual architectural constraint: the system prompt cannot be modified during a conversation. This sounds like a limitation, but it's actually a cost optimization elevated to a design rule.

Anthropic models support prompt caching: if the first N tokens of a prompt match a previous request, the computation for those tokens is cached. The cost reduction is about 75% on cache hits.

For a multi-turn agent conversation, the system prompt (which can be 1000–3000 tokens) gets sent with every LLM call. Over 10 turns, that's 10x the system prompt tokens. With caching — and a stable system prompt — turns 2–10 hit the cache. You pay for the system prompt once.

The design constraint: any code that wants to inject dynamic content into the system prompt mid-session is a cache invalidation. The system routes these injections elsewhere: skills inject as user messages, memory context wraps in user-message fence tags.

What I found powerful was how the constraint disciplines the architecture. When someone proposed 'let's add platform hints to the system prompt dynamically', the answer was 'no, build it into the session-start prompt.' That forced cleaner thinking about what truly needs to be system-level vs turn-level.

The engineering lesson: the best cost optimizations are the ones that become architectural constraints, not post-hoc optimizations you layer on top."

---

### Story SD-3: "Designing a Self-Improving System"

**Setup**: "Have you worked on a system that gets better over time?"

**The story**:
"Hermes has a self-improvement loop built around 'skills' — markdown files the agent creates and stores. When the agent completes a complex task, it can create a skill describing how to do it. Next time a similar task comes up, the skill gets invoked.

The infrastructure is clever: skills are plain markdown files in `~/.hermes/skills/`. Creation is an agent tool call. Invocation is a slash command. The skill list is in the system prompt metadata; skill content injects as a user message when invoked.

But when I looked deeper, I found the quality control layer is missing. The agent creates skills with no validation, no deduplication, no quality scoring. A skill with a buggy command persists until a human deletes it. Over months, skill directories accumulate noise alongside signal.

There's also a more strategic piece: hermes has RL infrastructure (`batch_runner.py`, trajectory compression, Atropos integration) that lets the deployed agent generate training data for future model versions. The agent creates trajectories → filtered by quality → used for fine-tuning → better model → deployed → generates better trajectories. This is the closed-loop version of self-improvement.

The lesson about self-improving systems: the generation mechanism is the easy part. The quality gate is where most systems fail. If you can't distinguish good self-generated artifacts from bad, you've built an artifact accumulator, not a learning system."

---

## Reliability and Incident Stories

### Story R-1: "A Race Condition That Almost Made It to Production"

**Setup**: "Tell me about a bug you caught before it caused an incident."

**The story**:
"In hermes-agent, there's a module-level list called `_last_resolved_tool_names` in `model_tools.py`. It's a process-global — written every time `get_tool_definitions()` is called.

The `delegate_task` tool — which spawns a child AI agent — correctly saves and restores this global around the child run:

```python
saved = list(_last_resolved_tool_names)  # save
child_agent.run_conversation(task)        # run child
_last_resolved_tool_names[:] = saved     # restore
```

This works perfectly in single-threaded use. But hermes has a gateway mode where many user sessions run as concurrent threads. If two sessions both call `delegate_task` simultaneously:

- Thread A: saves [tool1, tool2] to local variable
- Thread B: saves [tool3, tool4] to local variable
- Thread A: child runs, rewrites global to [tool1_child, tool2_child]
- Thread B: child runs, rewrites global to [tool3_child, tool4_child]
- Thread A: restores... but now restores Thread A's original save over Thread B's active child state
- Thread B: restores... picks up Thread A's restoration

Neither agent has the correct parent tool list after restoration. In practice this hasn't caused a reported bug — delegation under concurrent gateway sessions is rare. But the race window exists and would cause mysterious tool access errors under load.

The fix is one line: change `_last_resolved_tool_names: List[str] = []` to `_last_resolved_tool_names = threading.local()`.

The lesson: process-global state is always a concurrency risk. The save/restore pattern that works perfectly in single-threaded code silently breaks under concurrency. Whenever I see a module-level mutable object, I now ask: 'is this accessed from multiple threads?'"

---

### Story R-2: "A Performance Issue That Looked Like a Bug"

**Setup**: "Tell me about a performance issue you diagnosed."

**The story**:
"The hermes documentation described memory sync as 'non-blocking fire-and-forget.' Reading it, I assumed external memory providers (Honcho, Hindsight, Mem0) were called asynchronously after the response was delivered to the user.

When I read the actual code in `agent/memory_manager.py`, I found:

```python
def sync_all(self, user_content: str, assistant_content: str) -> None:
    for provider in self._providers:
        try:
            provider.sync_turn(user_content, assistant_content)  # BLOCKING
        except Exception as e:
            logger.warning(...)
```

Plain `def`, blocking for-loop, no thread spawn. Memory sync happens BEFORE the turn ends. If an external provider has 500ms network latency, every turn takes 500ms longer than it should — after the response is already generated.

The symptom a user would see: the agent text appears, then there's a pause of several hundred milliseconds before the prompt returns. They'd investigate the LLM API latency, find it's fine, and be confused.

The documentation was optimistically wrong. The fix is straightforward: submit `sync_all()` to a thread pool after returning the response. Accept eventual consistency in memory recall (the next turn's prefetch might not see the previous turn's sync).

The lesson: 'fire-and-forget' claims in documentation should always be verified by reading the actual call site. Documentation about async behavior is frequently aspirational rather than accurate."

---

## Trade-off Stories

### Story T-1: "The Custom Framework Decision"

**Setup**: "When would you build a custom solution vs use an existing framework?"

**The story**:
"Hermes explicitly chose to not use LangChain, LlamaIndex, or any other agentic framework. The agent loop in `run_agent.py` is 12,000+ lines of custom code.

That sounds excessive. Let me explain why it's right for this system.

Hermes has specific architectural requirements: synchronous agent loop (no async infection), prompt cache invariant (system prompt immutable per session), multi-platform gateway (16 adapters with per-user agent caching), and a self-improvement loop (skills, session search, RL training). LangChain's async-first design, stateless tool model, and frequent API changes would have made each of these harder to build and maintain.

But the cost is real: every agent infrastructure component must be maintained. No LangChain community plugins. New team members need to learn Hermes patterns instead of industry-standard patterns.

The decision rule I take from this: use a framework when your requirements align with the framework's design center. Build custom when your requirements require fighting the framework. Hermes's requirements required too much fighting.

The signal that you're in 'build custom' territory: when you spend more time working around the framework than using it. When the framework's abstractions don't compose cleanly with your design constraints."

---

### Story T-2: "SQLite for a Multi-User System"

**Setup**: "Tell me about a storage architecture decision and the trade-offs."

**The story**:
"Hermes uses SQLite for all session storage — including multi-user gateway deployments with 16+ platforms.

SQLite is the right choice for this system, but the reasoning is specific: the target deployment is a single server, not a distributed system. SQLite with WAL mode handles concurrent readers well and multiple concurrent writers with jittered backoff retries. FTS5 gives full-text search across all conversations with zero infrastructure.

The benefits matter: hermes runs on developer laptops, Raspberry Pi, Android (Termux), and Docker with no external dependencies. Requiring Postgres for a personal AI assistant would eliminate most of the user base.

But the limitation is real: SQLite is single-writer. With 50+ concurrent sessions all writing messages, write lock contention grows. The retry backoff (20–150ms jitter) adds latency to every turn in a contention scenario.

The key architectural decision: `SessionDB` is the only interface to storage. The abstraction for a Postgres swap is in place — someone needs to implement `PostgresSessionDB` and add a config option to switch backends.

The lesson: sometimes the right storage choice for today's scale is wrong for future scale, and that's acceptable IF the abstraction for migration is in place. SQLite now, Postgres when you need it — and the Postgres migration won't require rewriting the application layer."

---

## Improvement Stories

### Story I-1: "What I'd Change First"

**Setup**: "If you joined this team, what would you change and why?"

**The story**:
"Three things, in priority order:

First: `threading.local()` for `_last_resolved_tool_names`. This is a one-line fix that prevents a race condition that could cause incorrect agent behavior under concurrent gateway load. The save/restore pattern in `delegate_tool.py` already shows the authors understand the concern — they just haven't taken it to its conclusion. This fix would take an afternoon and eliminates a class of hard-to-reproduce bug.

Second: Background memory sync. `MemoryManager.sync_all()` blocks turn completion. The documentation says it's non-blocking; the code says it's blocking. Moving this to a background thread makes every turn response faster by whatever the slowest memory provider's latency is. The engineering risk (eventual consistency in memory recall) is acceptable for memory systems that are already eventually consistent by nature.

Third: Structured logging. Right now debugging a production hermes deployment requires grepping unstructured log lines. Adding `structlog` or `python-json-logger` with consistent fields (session_id, turn_number, tool_name, latency_ms, tokens) would make every future debugging session faster. This is a one-day change that pays dividends indefinitely.

Everything else — Postgres, per-tool timeouts, skills quality control — is important but can wait. These three are low-effort, high-impact, and don't require architectural changes."
