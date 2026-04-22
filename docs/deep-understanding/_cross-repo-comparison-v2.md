# Cross-Repo Comparison v2: hermes-agent

This document provides deep architectural philosophy comparison for hermes-agent as a single-repo analysis, structured for comparing against other agent frameworks when additional repos are in scope.

---

## Architectural Philosophy

### hermes-agent: "The Framework IS the Product"

Hermes makes a clear meta-choice: by not using an external agentic framework (no LangChain, no LlamaIndex, no AutoGen), Hermes becomes the framework. Every architectural decision reflects this — the system can optimize for its specific goals without fighting an external abstraction.

**Core philosophy**:
- Synchronous core for simplicity + debuggability
- Everything under HERMES_HOME for profile isolation
- Custom tool registry with AST discovery for zero-import-side-effects
- Prompt cache invariant as a first-class architectural constraint
- Self-improvement loop (skills + session search + memory) as a differentiator

**What this buys**: Total control. When prompt caching becomes available, Hermes can build an architectural constraint around it immediately. When a new provider appears, it plugs in without waiting for LangChain to add an integration.

**What this costs**: All agent infrastructure must be maintained internally. No community plugins. Onboarding requires learning Hermes-specific patterns.

---

## Dimension-by-Dimension Analysis

### Agent Loop Design

**hermes-agent**: Synchronous `while` loop in `run_agent.py:run_conversation()`. Single LLM call per iteration. Tool calls dispatched synchronously (with optional parallel tool execution via ThreadPoolExecutor). Max 90 iterations (configurable). Clean: sequential conversation semantics match the synchronous model.

**What this looks like in comparison to alternatives**:
- LangChain/LangGraph: Async-first, graph-based, nodes and edges. More complex for simple cases; more powerful for DAG-structured workflows.
- AutoGen: Multi-agent by default, message-passing between agents. Hermes achieves multi-agent via delegation (one agent spawns another), not message passing.
- OpenAI Swarm: Handoffs-based multi-agent. Hermes delegation is conceptually similar but more flexible (no rigid handoff protocol).

### Context Management

**hermes-agent**: `ContextEngine(ABC)` with threshold-based compression. Auxiliary (secondary, cheaper) LLM summarizes the middle. Session chain tracked via `parent_session_id`. Clean lifecycle hooks. Default threshold: 0.75 (class) / configurable per deployment.

**Design choice**: Preserve recent turns verbatim; summarize middle. This prioritizes recency over completeness. Loss: detailed tool call results from compressed sessions. Gain: recent context always crisp.

### Memory Architecture

**hermes-agent**: Three-tier (ephemeral / semantic / episodic). Default semantic memory is a flat file (MEMORY.md) — human-readable, editable, no infrastructure. Episodic is SQLite FTS5 — keyword-based, zero infrastructure. External vector store is optional plugin (Mem0, Honcho, Hindsight).

**Design choice**: Transparency over sophistication. You can read your agent's memory with `cat ~/.hermes/MEMORY.md`. You can edit it. You can delete entries you don't want. No black box. The cost: no semantic similarity search in the default layer.

### State Persistence

**hermes-agent**: SQLite WAL + FTS5. Zero infrastructure. Full message history. FTS5 full-text search across all sessions. Parent-session chain for compression tracking. Cost: single-writer, no replication.

**Single-server constraint**: Explicit design choice for the target use case (developer laptop, Raspberry Pi, Android via Termux, Docker with no external deps). The abstraction for a Postgres swap is designed in but not built.

### Multi-Platform Interface

**hermes-agent**: 16+ messaging platform adapters via `gateway/platforms/`. Single `GatewayRunner` manages all. Per-user `AIAgent` LRU cache (128 agents, 1-hour TTL). Background cron thread for autonomous tasks.

**Design pattern**: Platform adapter implements a common interface; `GatewayRunner` is platform-agnostic. New platforms: implement the interface, register with the runner.

### Self-Improvement

**hermes-agent**: Skills (LLM-created markdown files), session search (FTS5 over history), MEMORY.md updates. RL infrastructure via `batch_runner.py` + Atropos submodule for training data generation. Skills accumulate without quality control — design gap.

**Strategic intent**: The self-improvement loop is the strategic moat. Deployed agents generate training data for better models, which get deployed, which generate better training data. This is the "virtuous loop" that most commercial agent platforms lack.

### Tool System

**hermes-agent**: Self-registering tools via `registry.register()` at module level. AST-based discovery (no import). 50+ built-in tools covering shell, file, web, browser, memory, delegation, voice. Tool safety metadata per tool (parallel-safe, never-parallel, path-scoped). OpenAI-format tool schemas for all providers.

**Design insight**: Registering a tool is one function call. Discovering tools doesn't execute modules. Adding a new tool is additive, not modifying existing code.

### Provider Abstraction

**hermes-agent**: All providers via OpenAI wire format (OpenAI SDK). Native Anthropic SDK for Anthropic-specific features (caching, extended thinking, native tools). Bedrock, Gemini, local Ollama, and any OpenAI-compatible endpoint work out of the box.

**Design choice**: OpenAI format as the lingua franca of LLM APIs. This was a correct bet — nearly every new provider implements OpenAI-compatible endpoints. The cost: native features (Anthropic computer use, Google Gemini grounding) require provider-specific adapter paths.

---

## Unique Patterns Worth Borrowing

### 1. `HERMES_HOME` Single-Var Isolation
Any system that writes persistent state should have a single env var that redirects all writes. Enables: isolated test runs, multiple user profiles, clean demo environments. Cost: negligible. Value: enormous for testing and operations.

### 2. AST-Based Tool Discovery
Tools that need to self-register without being imported can use AST scanning of their own file. No manual registry, no import side effects, no circular dependency risk. Applicable to: plugin systems, command registries, event handler registries.

### 3. Prompt Cache Invariant as Architecture
Don't treat "minimize LLM cost" as an optimization concern — elevate it to an architectural constraint. Make the constraint explicit, document it, enforce it in code review. The 75% cost reduction delivers itself automatically.

### 4. `<memory-context>` Fencing
Any system that injects external content into LLM context should fence it with semantic tags. The tags signal to the model that the content is recalled/external, not user instruction. Simple to implement, reduces prompt injection risk.

### 5. COMMAND_REGISTRY Single Source of Truth
When multiple interfaces (CLI, bot, API, webhook) need to share command definitions, use a single list of command objects. Each object carries all metadata: name, description, aliases, interface-specific config gates. Eliminates the "I added the command to CLI but forgot the gateway" class of bug.

---

## Weaknesses Worth Knowing

### 1. No Structured Observability
No JSON logs, no OpenTelemetry, no metrics endpoint. Debugging production issues requires log-grepping. For a framework that runs autonomous background tasks (cron), this is a significant operational gap.

### 2. Config System Divergence
Two config loaders that must stay in sync manually. Every new config option is a potential divergence point. This is technical debt that compounds with feature growth.

### 3. Memory Sync Blocks Turn Completion
The most surprising finding from second-pass analysis: memory sync is blocking. This means slow external memory providers add latency to every turn response. The first-pass documentation (and AGENTS.md) incorrectly described this as non-blocking.

### 4. No per-Tool Timeouts
Tools can hang indefinitely. In a shared server (gateway mode), one hanging tool in one session can starve other sessions' thread pool workers.

---

## Maturity Signals

| Signal | hermes-agent |
|--------|-------------|
| Test isolation fixture | Yes (`_isolate_hermes_home`) |
| Abstract interfaces for swappable components | Yes (`ContextEngine`, `MemoryProvider`, `SessionDB`) |
| Documented architectural invariants | Yes (AGENTS.md, prompt cache rule) |
| Known bugs documented | Yes (`_last_resolved_tool_names` in AGENTS.md) |
| Cost-aware design | Yes (prompt caching, token tracking) |
| Observability | No (no structured logs, no metrics) |
| Horizontal scaling story | No (SQLite single-writer) |
| Security sandboxing | No (tools run as user process) |
