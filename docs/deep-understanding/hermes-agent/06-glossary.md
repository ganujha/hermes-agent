# 06 — Glossary: hermes-agent

## Repo-Specific Concepts

**Agent Loop**
The while-loop inside `AIAgent.run_conversation()` that cycles through: build prompt → LLM call → parse response → execute tool calls → append results → repeat. Terminates when the model returns no tool calls, the iteration budget is exhausted, or an interrupt is triggered.

**AIAgent**
The main class in `run_agent.py` that encapsulates a conversation session. One `AIAgent` = one conversation context. Multiple `AIAgent` instances can run concurrently (gateway per-session cache, batch runner, delegation).

**Auxiliary Client / Auxiliary Model**
A secondary, cheaper LLM client used for non-primary tasks: context compression summarization, vision fallback, and other background inference. Configured separately from the main model via `auxiliary_client` config. Lives in `agent/auxiliary_client.py`.

**Compression Chain**
When context compression fires, a new session is created in the database linked to the old session via `parent_session_id`. Following the chain reconstructs the full conversation history.

**Context Engine**
The pluggable component responsible for managing context window capacity. The abstract base is `agent/context_engine.py:ContextEngine`. The built-in implementation is `agent/context_compressor.py:ContextCompressor`. Can be replaced via `plugins/context_engine/`.

**Context Compaction**
The act of compressing the conversation history when approaching the context window limit. The default strategy: prune old tool results, then use an auxiliary LLM to summarize the middle portion, preserving first and last N messages intact.

**Credential Pool**
A collection of API keys/tokens for a single provider that can be rotated when one is rate-limited or exhausted. Managed by `agent/credential_pool.py`.

**Credential Source**
A named mechanism for obtaining a credential: `env:<VAR>`, `claude_code`, `hermes_pkce`, `device_code`, `gh_cli`, `config:<name>`, etc. Defined in `agent/credential_sources.py`.

**HERMES_HOME**
Environment variable that overrides the default `~/.hermes/` data directory. All paths in the system read from `get_hermes_home()` which checks this variable. The profile system works by setting `HERMES_HOME` to `~/.hermes/profiles/<name>` before imports.

**Home Channel**
The platform channel or chat ID configured for a given messaging platform where cron job outputs and bot notifications are delivered. Set via env vars like `TELEGRAM_HOME_CHANNEL`, `DISCORD_HOME_CHANNEL`, etc.

**Iteration Budget**
`IterationBudget` in `run_agent.py` — a thread-safe counter that limits total LLM API calls in a session. Refundable (tools can grant extra iterations). Distinct from `max_iterations` which is a hard per-conversation cap.

**KawaiiSpinner**
The animated terminal spinner in `agent/display.py` that shows while the LLM is processing. Displays animated "faces" and verbs during API calls. Customizable via the skin system.

**Memory Context Block**
The `<memory-context>...</memory-context>` fenced string injected into the user message at API call time when external memory has been prefetched. Never stored in the message history — only added transiently.

**Memory Manager**
`agent/memory_manager.py:MemoryManager` — orchestrates the built-in memory (MEMORY.md/USER.md) and at most one external memory provider. Single integration point in `run_agent.py`.

**Memory Provider**
An external service that manages cross-session memory (Honcho, Hindsight, Mem0). Implements the `agent/memory_provider.py:MemoryProvider` abstract base. At most one external provider active at a time.

**Nudge**
A periodic in-conversation prompt that reminds the agent to create a skill after completing a complex task. Frequency configured via `skills.nudge_interval` in config.yaml.

**Profile**
A fully isolated Hermes instance with its own `HERMES_HOME` directory (config, API keys, memory, sessions, skills, gateway settings). Created with `hermes -p <name>`. Accessed with `hermes -p <name> <command>`.

**Prompt Caching**
Anthropic's feature that caches the system prompt and early conversation turns server-side, reducing cost by ~75% on cache hits. Hermes treats cache validity as an architectural invariant — no mid-conversation system prompt mutations allowed.

**RemovalStep / RemovalResult**
Pattern in `agent/credential_sources.py` where each credential source registers cleanup logic. A `RemovalStep` knows how to remove its credential from external state (env file, OAuth file, auth.json). The result is a `RemovalResult` with cleaned items and hints.

**SOUL.md**
User-editable file at `~/.hermes/SOUL.md` that defines the agent's persona. Injected into the system prompt at the start of every conversation.

**Skill**
A markdown file (`.md`) containing procedural instructions for a specific domain task. Stored in `skills/` (built-in) or `~/.hermes/skills/` (user-created). Injected as a user message when invoked via `/<skill-name>`. Skills can have frontmatter for platform targeting.

**Skills Hub**
Community platform at `agentskills.io` for sharing and discovering skills. Hermes integrates via `tools/skills_hub.py` and the `/skills` slash command.

**Skin**
A YAML data file that customizes CLI visual appearance: colors, spinner faces, branding strings, tool emojis. Built-in skins: `default`, `ares`, `mono`, `slate`. User skins in `~/.hermes/skins/`.

**Tick**
The 60-second heartbeat in `cron/scheduler.py:tick()`. Called by the gateway background thread. Checks for due cron jobs and executes them. Protected by a file lock to prevent concurrent ticks.

**Toolset**
A named group of tools. Examples: `web` (web_search + web_extract), `browser` (full browser suite), `terminal` (terminal + process). Toolsets can be enabled or disabled per platform. Defined in `toolsets.py`.

**Trajectory**
A recorded conversation including all messages, tool calls, tool results, and metadata. Saved via `agent/trajectory.py`. Used by `batch_runner.py` and `trajectory_compressor.py` to generate RL training data.

**TUI**
Terminal User Interface — the full-screen Ink (React for terminal) application activated with `hermes --tui`. Runs as a Node.js process that communicates with a Python backend (`tui_gateway/`) via JSON-RPC over stdio.

**tui_gateway**
Python component (`tui_gateway/`) that serves as the backend for the TUI. Handles sessions, tool execution, LLM calls, and slash commands via JSON-RPC. Bridges Ink UI with `AIAgent`.

**WAL mode**
SQLite's Write-Ahead Logging mode, used in `hermes_state.py`. Allows concurrent readers while a single writer is active. Prevents database locks from blocking read queries during write operations.

---

## Internal Abstractions

**`_HERMES_CORE_TOOLS`**
List in `toolsets.py` of all tools enabled for both CLI and messaging platforms by default. The single place to add a tool that should be universally available.

**`_NEVER_PARALLEL_TOOLS`**
Set in `run_agent.py` of tool names that must never run in parallel (e.g., `clarify` which requires human input).

**`_PARALLEL_SAFE_TOOLS`**
Set in `run_agent.py` of tool names that are always safe to run in parallel (read-only tools).

**`CommandDef`**
Dataclass in `hermes_cli/commands.py` defining a slash command: name, description, category, aliases, platform availability flags, config gate.

**`COMMAND_REGISTRY`**
Single source of truth list of all `CommandDef` objects. All CLI and gateway command dispatch, help text, and platform menus derive from this list.

**`ContextEngine`**
Abstract base class for context management. Defines the interface: `update_from_response`, `should_compress`, `compress`, lifecycle hooks.

**`MemoryProvider`**
Abstract base class for external memory backends. Defines lifecycle: `is_available`, `initialize`, `prefetch`, `sync_turn`, `get_tool_schemas`, `on_session_end`, `shutdown`.

**`ToolEntry`**
Dataclass in `tools/registry.py` holding tool metadata: name, toolset, schema, handler function, check function, required env vars.

**`SessionDB`**
Class in `hermes_state.py` wrapping SQLite with WAL mode, FTS5, and write contention handling.

---

## Execution Lifecycle Terms

**`on_session_start(session_id)`** — called when a conversation begins; used by context engines to load persisted state.

**`on_session_end(session_id, messages)`** — called at real session boundaries (CLI exit, `/reset`, gateway session expiry). NOT called per-turn.

**`on_session_reset()`** — called on `/new` or `/reset`; resets per-session state (compression count, token tracking).

**`prefetch(user_message)`** — memory provider method called before each LLM turn; background-loads relevant memories.

**`sync_turn(user_msg, assistant_response)`** — memory provider method called after each turn; writes new memories asynchronously.

**`compress(messages)`** — context engine method; receives full message list, returns shorter list.

**`should_compress(prompt_tokens)`** — context engine method; returns True when compression should fire.
