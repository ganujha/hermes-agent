# 09 — Teaching Note: hermes-agent

---

## For a Strong Software Engineer New to Agentic Systems

Hermes Agent is what happens when you take the OpenAI tool-calling API pattern and build a full production system around it. If you've seen LLM demos where the model calls a function, gets a result, and continues — Hermes is that, but with every practical concern solved: persistence, memory, multi-platform access, autonomous scheduling, and self-improvement.

**The core pattern is a while-loop.** In `run_agent.py:AIAgent.run_conversation()`:
1. Send the conversation history (as an OpenAI-format message list) to an LLM.
2. If the model returns tool calls, execute them and append results to the history.
3. Loop until the model returns pure text (no tool calls), or until you hit the iteration limit.

That's the whole agent. Everything else is infrastructure around that loop.

**Three things to understand first:**

1. **The tool registry** (`tools/registry.py`): Tools self-register at import time using `registry.register()`. Discovery uses Python's `ast` module to find these calls without running the module — a clever way to avoid a manual import list.

2. **Context management** (`agent/context_engine.py`): LLMs have finite context windows. When conversations get long, Hermes uses a secondary (cheaper) LLM to summarize the middle of the conversation and replace it with a compact summary — all transparent to the user. The `ContextEngine` ABC defines a clean lifecycle for this.

3. **The HERMES_HOME pattern** (`hermes_constants.py`): All state (config, memory, sessions, keys) lives under a single directory. One env var (`HERMES_HOME`) overrides it. This enables profile isolation (multiple independent agent instances) with zero code duplication.

**Key insight**: Hermes is a framework for persistent, tool-using, self-improving agents. Most of the complexity is in managing what the model knows (context), what it remembers (memory), and what it can do (tools) — not in the model calls themselves.

**Start by reading**:
- `AGENTS.md` (agent loop section)
- `tools/registry.py` (first 100 lines)
- `run_agent.py` (`AIAgent.__init__` + `run_conversation()` body)

---

## For a Staff+ Engineer

Hermes is a full-stack agentic platform with production-grade architecture decisions — and some honest technical debt worth understanding.

**The architectural wins:**

The **pluggable context engine** (`agent/context_engine.py`) is the strongest design decision in the codebase. The abstract base defines a clean lifecycle (session start → token tracking → should_compress? → compress → session end) that decouples the agent loop from any specific context management strategy. The built-in LLM-based summarizer is just one implementation. This is the kind of abstraction that lets the system evolve without rewriting the core loop.

The **single COMMAND_REGISTRY** pattern in `hermes_cli/commands.py` solves a multi-interface synchronization problem elegantly. Five different consumers (CLI, gateway, Telegram menu, Slack routing, autocomplete) all derive from one list. The `CommandDef.gateway_config_gate` field is particularly clever: a command can be CLI-only by default but activate in the gateway if a config key is set — no code changes.

The **prompt caching invariant** is a rare example of an LLM cost concern being elevated to an architectural constraint. The system explicitly forbids mid-conversation system prompt mutations. This forces disciplined context management and delivers ~75% cost reduction on cache-hit turns.

**The honest risks:**

The **process-global `_last_resolved_tool_names`** in `model_tools.py` is a latent concurrency bug. The `delegate_tool.py` saves/restores it around child runs, but concurrent parent agents delegating simultaneously (in the gateway) create a race window. It hasn't caused a reported bug yet because delegation is relatively rare, but it's the kind of thing that surfaces under scale.

**Two config loaders** (`load_cli_config()` vs `load_config()`) indicates the CLI and gateway were developed somewhat independently. This kind of divergence tends to accumulate over time and makes adding config options error-prone.

**SQLite as the session store** is defensible for single-user or small-team use. The WAL mode + FTS5 combination is genuinely powerful (full-text search across all conversation history is a compelling feature). But the single-writer architecture and lack of connection pooling will become pain points in multi-user deployments.

**The meta-question worth asking**: Hermes positions itself as self-improving (skills created by the agent, session history searched by the agent). The infrastructure is there. But the quality control mechanisms are weak — skills are LLM-written without automated review, and there's no deduplication. The learning loop will work, but at scale it will accumulate noise. The architectural question is: should skill quality management live in the agent (LLM judges its own output) or in the infrastructure (heuristic scoring, embedding-based deduplication)?

**For production deployment**, the three things I'd address first: (1) Postgres backend for session storage, (2) per-tool timeouts in the registry, (3) fix the `_last_resolved_tool_names` global.

---

## For a CTO Evaluating the Architecture

Hermes Agent is a production-grade open-source framework for persistent, multi-platform, self-improving AI agents. Built by Nous Research, it's both a developer tool and an infrastructure layer for agentic deployments.

**Strategic position**: Hermes sits at the intersection of three trends — LLM-agnostic deployment (20+ providers, no lock-in), persistent AI assistants (memory across sessions), and autonomous agents (cron scheduling, delegation). The open standard (`agentskills.io`) for skills sharing suggests an ecosystem play.

**Architecture maturity**: The core agent loop is solid. The provider abstraction is mature (supports Anthropic, OpenAI, Bedrock, Gemini, and any OpenAI-compatible endpoint). The multi-platform gateway works across 16+ messaging platforms. The context compression system is well-designed with a clean plugin interface.

**Production readiness assessment**:
- ✅ Battle-tested provider abstraction and failover
- ✅ Clean session isolation via profile system
- ✅ Proven multi-platform messaging (16+ adapters)
- ✅ Built-in cost tracking and token accounting
- ⚠️ SQLite session store doesn't scale horizontally
- ⚠️ No built-in distributed tracing or structured logging
- ⚠️ No health/metrics endpoint for operational visibility
- ⚠️ Tool execution lacks per-tool timeouts

**Competitive differentiation**: The self-improvement loop (skills + session search + memory) is the strongest differentiator. Most commercial agents are stateless. Hermes accumulates institutional knowledge over time. The RL infrastructure (`batch_runner.py`, Atropos integration) suggests Nous Research is using this system to generate training data for better agents — a virtuous loop where the deployed agent improves future deployed agents.

**Team fit**: This is a good platform for teams who want to own their AI agent infrastructure, work across multiple LLM providers, and build autonomous internal automations. The learning curve is real (the codebase is large and non-trivial), but the documentation quality is above average for open-source AI tooling, and the `AGENTS.md` developer contract is particularly useful for onboarding contributors.

**Key risk**: The architecture is optimized for single-server, moderate-concurrency deployments. A team needing to run thousands of concurrent agent sessions would hit the SQLite and in-process limitations quickly. The design is modular enough to replace those components, but it's not turn-key for hyperscale.

**Bottom line**: For a team wanting a production-ready, self-hosted, multi-platform AI agent that they can extend and don't want to vendor-lock into a cloud platform — this is one of the strongest open-source options available. The code quality, documentation, and architectural discipline are significantly above the average AI agent project.
