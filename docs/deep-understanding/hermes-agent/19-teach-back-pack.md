# 19 — Teach-Back Pack: hermes-agent

Explanations at multiple depths. Use for interviews, team onboarding, or checking your own understanding.

---

## 2-Minute Explanation (Twitter / Elevator Pitch)

"Hermes Agent is an open-source AI assistant that remembers you, runs on any messaging platform, and improves itself over time.

You talk to it through Telegram, Slack, Discord, or a terminal. It stores conversation history in a local database. It builds a personal memory file that persists across sessions. It can schedule itself to run tasks while you sleep. And it creates its own tools — called skills — to get better at things you ask it to do repeatedly.

Under the hood, it's a Python program with a simple while-loop: send conversation to an LLM, run the tools the LLM requests, append results, repeat. No magical frameworks. Just careful engineering around that loop."

---

## 5-Minute Explanation (Technical Interview Intro)

"Hermes is a self-hosted AI agent framework with a custom architecture — no LangChain, no AutoGen.

The core is a synchronous while-loop in `run_agent.py:run_conversation()`. Each turn: build a message list from conversation history, call the LLM API, parse any tool calls, execute them (in parallel if safe), append results, loop. When the model returns pure text with no tool calls, the turn ends.

Around that loop, there are five systems:

**State**: SQLite with WAL mode stores every session and message. SQLite FTS5 gives full-text search across all conversation history — so the agent can search its own past.

**Memory**: Three tiers. Ephemeral — the in-process message list. Semantic — MEMORY.md and USER.md files that the agent reads and writes. Episodic — FTS5 keyword search over SQLite. External vector stores (Mem0, Honcho) are optional plugins.

**Context management**: When conversations get long, a secondary (cheaper) LLM summarizes the middle. The `ContextEngine(ABC)` is a clean lifecycle abstraction — compression is pluggable.

**Multi-platform gateway**: A long-lived Python process that connects to 16+ messaging platforms. Each user gets their own `AIAgent` instance, cached in an LRU cache (max 128, 1-hour TTL). A background thread ticks a cron scheduler every 60 seconds for autonomous tasks.

**Self-improvement**: Skills are markdown files in `~/.hermes/skills/`. The agent creates them. Users invoke them with slash commands. They inject as user messages (not system prompt) to preserve Anthropic prompt caching — that's a ~75% cost reduction on multi-turn conversations.

The design is optimized for single-server, moderate-concurrency deployments. SQLite can't scale horizontally, there are no per-tool timeouts, and the config system has two diverging loaders. But for a self-hosted personal or team agent, it's production-ready."

---

## 15-Minute Deep Dive (Architecture Review / Staff Engineer Context)

### The Architectural Core

Hermes makes three unusual choices that define its character:

**1. Synchronous core, async boundary**
The `AIAgent.run_conversation()` method is entirely synchronous — no `async def`, no `await` anywhere in the loop body. This is correct. A single agent conversation is inherently sequential — you can't start turn 2 before turn 1's response arrives. Async would complicate reasoning about state without providing any benefit. The async boundary sits at the Gateway level: platform adapters (Telegram bot, Discord) use asyncio, but they deliver messages synchronously to the agent.

Consequence: `MemoryManager.sync_all()` is a blocking call at the end of each turn. If a slow external provider (Honcho, Hindsight) takes 500ms, the user sees the agent "processing" after the response already appeared. First-pass documentation wrongly called this "fire-and-forget." It's blocking. Fix: submit `sync_all()` to a background thread after returning the response.

**2. Prompt cache invariant**
The system prompt is built once at session start and never modified mid-session. This is an architectural invariant, not a policy. Consequence: ~75% cost reduction on Anthropic models via cache hits. The design around this constraint is elegant: skills invoke as user messages (not system prompt additions), memory updates wait for next session. Two code paths exist for skills: `build_skills_system_prompt()` lists available skills as metadata (static, goes to system prompt), and `build_skill_invocation_message()` injects skill content as a user message when invoked (dynamic, preserves cache).

**3. Process-global tool state (known risk)**
`model_tools.py:_last_resolved_tool_names` is a module-level list. Written by `get_tool_definitions()` after every tool resolution. Read by `delegate_tool.py` to save/restore parent tool context around child agent runs. The save/restore is correct for single-threaded delegation. But in gateway mode with concurrent sessions, multiple threads share this global. A race window exists when two agents delegate simultaneously. No lock protects it. The fix is threading.local() — straightforward, not yet done.

### The Memory Architecture

Three tiers with different characteristics:

| Tier | Storage | Access | Persistence |
|------|---------|--------|--------------|
| Ephemeral | In-process list | O(1) | Session only |
| Semantic | MEMORY.md flat file | Read-on-session-start | Across sessions |
| Episodic | SQLite FTS5 | BM25 keyword search | Across sessions |

Memory injection uses `<memory-context>` tags as a fence — recalled memory is wrapped in these tags before being appended to the user message. The model is expected to treat the tag content as recalled memory, not user instruction. This is a convention, not a technical barrier. But it's better than nothing.

Single external provider constraint: only one of {Honcho, Hindsight, Mem0} can be active at once. If you want semantic AND dialectic memory, you need to build a provider that wraps both.

### The Config Problem

Two config loaders exist and diverge:
- `cli.py:load_cli_config()` — interactive CLI config; falls back to `./cli-config.yaml`; includes personality templates (kawaii, pirate, etc.)
- `hermes_cli/config.py:load_config()` — library/gateway config; HERMES_HOME only; handles `max_turns` migration; uses `_deep_merge()`

They were developed somewhat independently. Any new config option requires updating both. This is a debt that compounds linearly with features.

### The Self-Improvement Loop

Skills are LLM-created markdown files. The creation path: agent reasons it would benefit from a skill → calls `skill_manage` tool → writes markdown → file persists. Next time a similar task comes up, the skill is listed in the system prompt (metadata) and the agent may invoke it.

Problems: no deduplication, no quality scoring, no automated review. A skill with a typo in a command gets used until a human notices. Over months of use, skill directories accumulate noise.

The RL infrastructure (`batch_runner.py`, `trajectory_compressor.py`, Atropos submodule) suggests Nous Research uses hermes to generate training data for better models. The deployed agent generates trajectories → filtered → used for fine-tuning → better model deployed. This is the strategic differentiator: the agent improves the models that power agents.

---

## Common Misconceptions

### "Memory is always up-to-date within a session"
**Reality**: Memory recalled at session start is baked into the user message context (not system prompt). Memory WRITTEN mid-session (by the agent calling a memory tool) takes effect immediately for the agent's in-process messages, but the semantic memory files (MEMORY.md) are updated synchronously. However, context compression may summarize away recent memory writes from earlier in the session. New sessions always get fresh memory prefetch.

### "All tools run in parallel"
**Reality**: Parallel execution is conditional. `_should_parallelize_tool_batch()` checks against blocklists, allowlists, and path overlap. Many tools are not in `_PARALLEL_SAFE_TOOLS` and execute sequentially. Single-tool batches always run sequentially. The maximum parallelism is 8 workers, and that only applies when all tools in a batch pass the safety checks.

### "Context compression means the agent forgets old information"
**Reality**: Compression replaces the MIDDLE of the conversation with a summary (not the beginning or end). The most recent N turns are always kept. The summary is generated by an auxiliary LLM. What's lost is the verbatim tool call results and intermediate reasoning from compressed turns. The agent has access to the summary. For important long-running context, the agent should write to MEMORY.md before compression fires.

### "Skills are system prompt instructions"
**Reality**: Two mechanisms: (1) The list of available skills is included in the system prompt as metadata (static, built at session start). (2) Skill CONTENT is injected as a user message when invoked. The user message injection preserves the system prompt cache. The activation note in skill messages uses `[SYSTEM: The user has invoked...]` prefix — this is a convention signal to the model, not an actual system-level message.

### "hermes uses LangChain"
**Reality**: Deliberately avoids it. The agent loop, tool system, context management, and memory are all custom implementations. This gives full control but requires maintaining all infrastructure internally. The `openai` SDK is used only for its wire protocol format (all providers speak OpenAI format).

### "Memory sync is async / fire-and-forget"
**Reality**: `MemoryManager.sync_all()` is a synchronous blocking call in a for-loop. Slow external providers block the turn response. This is a known production gap. The documentation (including AGENTS.md section descriptions) implies non-blocking — but the code is blocking.

---

## Interview Stories

### Story: "The Prompt Cache Invariant"
Context: You're asked about cost optimization in LLM systems.

"In hermes, we have a hard architectural rule: the system prompt cannot change during a session. Every team member knows this rule. When you add a feature — skills, memory injection, platform hints — you work around the invariant rather than through it.

The reason is simple: Anthropic's prompt caching reduces LLM costs by about 75% on multi-turn conversations where the system prompt doesn't change. A 1000-token system prompt, re-sent 10 times without caching, costs 10x. With caching, 9 of those 10 calls hit the cache.

The interesting engineering comes from maintaining the invariant under pressure. Skills can't go into the system prompt (they're dynamic). We inject them as user messages. Memory recalled mid-session can't update the system prompt. We fence it with `<memory-context>` tags in the user message.

What I learned: the most productive architectural constraints are the ones that have clear economic justification. When someone asks 'why can't we put X in the system prompt?', the answer 'because it would cost us 4x more' shuts down the debate quickly."

### Story: "The Synchronous Core vs Async Complexity"
Context: You're asked about concurrency architecture decisions.

"When I first looked at hermes, I expected the agent loop to be async — it makes network calls, runs tools, waits for LLM responses. Everything about it screams 'async.' But `run_conversation()` is a plain synchronous function.

The insight: async concurrency is valuable when you want to interleave work. But an agent conversation is inherently sequential — turn 2 waits for turn 1. There's nothing to interleave. Making the loop async would only add complexity for the two phases where the loop genuinely does I/O in parallel: tool execution (handled by ThreadPoolExecutor, no async needed) and streaming (handled by the provider SDK).

The lesson: async is a tool for concurrency, not for I/O. If you're doing sequential work with one thread per conversation, synchronous code is easier to debug, profile, and reason about. The async layer belongs at the boundary (gateway event loop) not in the core."

### Story: "The Process-Global Race Condition"
Context: You're asked about bugs you found or production incidents.

"In hermes's multi-agent delegation system, there's a module-level list called `_last_resolved_tool_names` in `model_tools.py`. It stores which tools were resolved in the last `get_tool_definitions()` call.

When a parent agent spawns a child agent via `delegate_task`, the delegate tool saves this list, runs the child agent, then restores the list. That pattern works perfectly for single-threaded use.

But in gateway mode, multiple user sessions run as concurrent threads. If two sessions both call `delegate_task` at the same moment, the saves and restores interleave. Thread A saves, Thread B saves (overwriting Thread A's state), Thread A restores B's value (wrong), Thread B restores A's value (wrong). Neither agent gets the correct parent tool list after restoration.

No reported bug yet — delegation under concurrent gateway sessions is rare. But the race window is real. The fix is one line: change `_last_resolved_tool_names: List[str] = []` to `_last_resolved_tool_names = threading.local()`. Sometimes the most dangerous bugs are the ones that only show up under specific concurrency conditions that almost never happen in practice."
