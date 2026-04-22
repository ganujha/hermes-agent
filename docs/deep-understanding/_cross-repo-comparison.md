# Cross-Repo Comparison

> **Note**: This analysis covers a single repository: **hermes-agent**. This document is structured for comparison readiness — when additional repositories are analyzed, this file should be expanded with comparative columns.

---

## hermes-agent — Architecture Summary for Comparison

| Dimension | hermes-agent | Notes |
|-----------|-------------|-------|
| **Agent Architecture** | Single-agent with on-demand multi-agent delegation. One `AIAgent` class, `delegate_task` tool spawns child instances. | Explicit coordinator/subagent: parent blocks while child runs independently |
| **Memory Design** | Three-tier: in-process messages (ephemeral), file-based MEMORY.md/USER.md (semantic), SQLite FTS5 (episodic recall). One external provider slot (Honcho, Mem0, etc.). | Memory fencing with `<memory-context>` prevents injection. No vector store built-in — keyword search only in core. |
| **Tool Design** | Self-registering via AST discovery. 50+ built-in tools. Toolset groups for platform control. MCP client for external tools. Plugin tool discovery. | Clean registry abstraction. Schema + handler + check_fn per tool. All handlers return JSON strings. |
| **Developer Workflow** | Python CLI (`hermes`), full TUI (`hermes --tui`), gateway for messaging platforms. Single `COMMAND_REGISTRY` drives all interfaces. | Two config loaders is a known issue. `scripts/run_tests.sh` enforces CI-parity. |
| **Reliability Model** | Tenacity retries, jittered backoff for SQLite writes, error classification for LLM errors, file locks for cron mutex. | No per-tool timeouts. SQLite single-writer limitation. 90-iteration hard cap. |
| **Observability** | Python logging throughout, session DB as implicit audit log, `/insights` command, per-session cost tracking. | No structured/JSON logs, no distributed tracing, no health endpoint, no metrics export. |
| **Extensibility** | Plugin ABCs for context engine and memory. Self-registering tools. Skills as hot-loaded markdown. Platform adapters behind common interface. Skin/theme as YAML data. | Pluggable at 5 levels. Central registries prevent fragmentation. |
| **Security Posture** | Prompt injection detection in context files. Memory context fencing. DM pairing for gateway users. Command approval patterns. | Defense-in-depth on LLM-facing inputs. MCP server trust not validated. Skills from external sources not sandboxed. |
| **Production Readiness** | Battle-tested for single-server, moderate-concurrency deployments. SQLite limits horizontal scaling. No health endpoint. | Strong for personal/team use. Not turnkey for multi-user SaaS. |
| **Best Interview Stories** | Self-registering tool registry with AST discovery. Pluggable context engine ABC. Single COMMAND_REGISTRY across 5 interfaces. Memory fencing for injection prevention. Prompt caching as architectural invariant. HERMES_HOME profile isolation. |

---

## Key Patterns Unique to hermes-agent

1. **Skills as self-improving procedural memory**: Agent writes and refines its own `.md` instruction files. No equivalent in most agent frameworks — this is a genuine self-improvement loop.

2. **Compression as a first-class abstraction**: `ContextEngine` ABC with full lifecycle hooks is more principled than ad-hoc truncation strategies common in other frameworks.

3. **Prompt caching treated as a correctness constraint**: Not just an optimization flag — the architecture is explicitly designed to preserve cache validity. AGENTS.md documents this as an invariant developers must not break.

4. **Multi-platform with shared command registry**: 16+ messaging platforms sharing one command registry is unusual. Most frameworks treat each platform as a fully custom integration.

5. **RL trajectory infrastructure built-in**: `batch_runner.py` and trajectory compression for model training suggests the deployed agent is part of a training loop for future agents.
