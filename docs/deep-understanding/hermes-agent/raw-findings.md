# Raw Findings — hermes-agent

## Repo Map

```
hermes-agent/
├── run_agent.py          12,053 lines  — AIAgent class, core loop
├── cli.py                11,045 lines  — HermesCLI, interactive REPL
├── model_tools.py         ~600 lines   — tool discovery, dispatch
├── toolsets.py            ~500 lines   — toolset group definitions
├── hermes_state.py        ~800 lines   — SQLite session store
├── hermes_constants.py    ~150 lines   — path resolution, HERMES_HOME
├── hermes_logging.py      ~300 lines   — logging config
├── hermes_time.py         ~50 lines    — time utilities
├── batch_runner.py       ~1,200 lines  — parallel batch trajectory generation
├── trajectory_compressor.py ~800 lines — trajectory compression for RL training
├── rl_cli.py             ~400 lines    — RL training CLI
├── mcp_serve.py          ~600 lines    — MCP server (expose Hermes as MCP)
├── mini_swe_runner.py    ~600 lines    — SWE-bench style task runner
├── utils.py              ~250 lines    — atomic writes, misc helpers
│
├── agent/                — Agent internals package
│   ├── anthropic_adapter.py    — Anthropic Messages API + OAuth
│   ├── bedrock_adapter.py      — AWS Bedrock converse API
│   ├── auxiliary_client.py     — Cheap model for compression/vision
│   ├── context_compressor.py   — Built-in context compression
│   ├── context_engine.py       — Abstract ContextEngine base class
│   ├── context_references.py   — Context file scanning
│   ├── credential_pool.py      — Unified credential management
│   ├── credential_sources.py   — Credential source types + removal contracts
│   ├── display.py              — KawaiiSpinner, tool display
│   ├── error_classifier.py     — API error classification + failover
│   ├── memory_manager.py       — Memory provider orchestration
│   ├── memory_provider.py      — Abstract MemoryProvider base
│   ├── model_metadata.py       — Model context lengths, token estimation
│   ├── prompt_builder.py       — System prompt assembly
│   ├── prompt_caching.py       — Anthropic cache_control injection
│   ├── skill_commands.py       — Skill slash commands
│   └── trajectory.py           — Trajectory saving helpers
│
├── tools/                — Tool implementations (self-registering)
│   ├── registry.py             — Central tool registry
│   ├── terminal_tool.py        — Shell execution
│   ├── file_tools.py           — File read/write/patch/search
│   ├── browser_tool.py         — Playwright/Browserbase automation
│   ├── web_tools.py            — Search + content extraction
│   ├── mcp_tool.py             — MCP client integration (~1050 lines)
│   ├── delegate_tool.py        — Subagent spawning
│   ├── memory_tool.py          — MEMORY.md/USER.md built-in memory
│   ├── skill_manager_tool.py   — Skill CRUD
│   ├── execute_code.py         — Python sandbox execution
│   ├── vision_tools.py         — Image analysis
│   ├── image_generation_tool.py — Text-to-image
│   ├── session_search_tool.py  — FTS5 session history search
│   ├── todo_tool.py            — Task tracking
│   ├── send_message_tool.py    — Cross-platform messaging
│   ├── cronjob_tools.py        — Cron job management
│   ├── approval.py             — Dangerous command detection
│   └── environments/           — Terminal backends (local, docker, ssh, modal, daytona, singularity)
│
├── gateway/              — Messaging platform gateway
│   ├── run.py                  — GatewayRunner, main loop, slash commands
│   ├── session.py              — Session storage + management
│   ├── config.py               — Platform configuration
│   ├── delivery.py             — Message routing
│   ├── stream_consumer.py      — Real-time event streaming
│   ├── platforms/              — Telegram, Discord, Slack, WhatsApp, Signal,
│   │                             Matrix, Mattermost, HomeAssistant, DingTalk,
│   │                             Feishu, WeChat, SMS, Email, Webhook, BlueBubbles, QQBot
│   └── hooks.py                — Gateway event hooks
│
├── cron/                 — Scheduled job execution
│   ├── scheduler.py            — tick() runner, job execution
│   └── jobs.py                 — Job storage, parsing, due detection
│
├── hermes_cli/           — CLI subcommands + config
│   ├── main.py                 — Entry point for `hermes` command
│   ├── config.py               — DEFAULT_CONFIG, config migration
│   ├── commands.py             — COMMAND_REGISTRY, CommandDef
│   ├── callbacks.py            — clarify, sudo, approval callbacks
│   ├── setup.py                — Interactive setup wizard
│   ├── auth.py                 — Provider credential resolution
│   ├── models.py               — Model catalog, provider model lists
│   ├── skin_engine.py          — Skin/theme engine
│   ├── skills_hub.py           — Skills Hub integration (agentskills.io)
│   └── tools_config.py         — Tool enable/disable config
│
├── ui-tui/               — Ink (React/TypeScript) TUI
│   ├── src/entry.tsx           — TTY gate, render entrypoint
│   ├── src/app.tsx             — State machine, main UI
│   ├── src/gatewayClient.ts    — Child process + JSON-RPC bridge
│   └── src/components/         — Ink UI components
│
├── tui_gateway/          — Python JSON-RPC backend for TUI
│   ├── entry.py                — stdio entrypoint
│   ├── server.py               — RPC handlers, session logic
│   └── slash_worker.py         — Persistent HermesCLI subprocess
│
├── acp_adapter/          — ACP server (IDE integrations)
│   ├── server.py               — FastAPI ACP endpoints
│   ├── session.py              — Session management
│   └── tools.py                — Tool proxy
│
├── skills/               — Skill library (27 category directories)
├── optional-skills/      — Optional skills (separate from built-in)
├── plugins/              — Plugin system
│   ├── context_engine/         — Pluggable context engines
│   └── memory/                 — Pluggable memory providers
│
├── environments/         — RL training environments (Atropos)
├── tests/                — Pytest suite (~3,000 tests)
├── web/                  — Web dashboard (React/TypeScript/Vite)
├── website/              — Docusaurus documentation site
├── docker/               — Docker configuration
└── packaging/            — Packaging scripts
```

## Entrypoints

| Command | File | Function |
|---------|------|----------|
| `hermes` | `hermes_cli/main.py` | `main()` |
| `hermes-agent` | `run_agent.py` | `main()` |
| `hermes-acp` | `acp_adapter/entry.py` | `main()` |
| `hermes --tui` | `ui-tui/src/entry.tsx` + `tui_gateway/entry.py` | Ink process + Python RPC |
| `hermes gateway start` | `gateway/run.py` | `start_gateway()` |
| Shell script | `hermes` (root) | Detects venv, calls Python |

## Key Configuration Files

| File | Purpose |
|------|--------|
| `~/.hermes/config.yaml` | Primary user config (model, tools, memory, display) |
| `~/.hermes/.env` | API keys and secrets |
| `~/.hermes/MEMORY.md` | Persistent agent memory |
| `~/.hermes/USER.md` | User profile/preferences |
| `~/.hermes/SOUL.md` | Agent persona/personality |
| `~/.hermes/AGENTS.md` | Agent-level context files (per project) |
| `~/.hermes/state.db` | SQLite session database |
| `~/.hermes/cron/jobs.json` | Scheduled jobs |
| `~/.hermes/skills/` | User-created skills |
| `~/.hermes/skins/` | Custom skin YAML files |
| `~/.hermes/profiles/` | Multi-instance profile directories |
| `cli-config.yaml.example` | Example config with all options |
| `pyproject.toml` | Python package manifest + dependencies |
| `package.json` | Node.js manifest for TUI |

## Key Classes and Functions

### run_agent.py
- `AIAgent.__init__()` — constructs agent with provider, model, toolsets, memory, session
- `AIAgent.run_conversation()` — main loop: build prompt → LLM call → tool execution → repeat
- `AIAgent.chat()` — simple interface returning final string
- `IterationBudget` — thread-safe iteration counter with refund mechanism
- `_SafeWriter` — resilient stdout wrapper for broken pipes in daemon mode

### agent/context_engine.py
- `ContextEngine(ABC)` — abstract base for all context engines
- Key abstract methods: `update_from_response()`, `should_compress()`, `compress()`
- Lifecycle hooks: `on_session_start()`, `on_session_end()`, `on_session_reset()`

### agent/context_compressor.py
- `ContextCompressor(ContextEngine)` — built-in summarization-based compression
- `compress()` — prune old tool results → auxiliary LLM summarization → return trimmed list
- `should_compress()` — checks `last_prompt_tokens / context_length > threshold_percent`
- Summary prefix: `[CONTEXT COMPACTION — REFERENCE ONLY]` with explicit "do not answer old questions" framing

### agent/memory_manager.py
- `MemoryManager` — coordinates built-in + at most one external provider
- `build_memory_context_block()` — wraps recalled memory in `<memory-context>` fence
- `sanitize_context()` — strips fence tags to prevent injection via provider output

### tools/registry.py
- `ToolRegistry` — central tool registry
- `discover_builtin_tools()` — AST-scans `tools/*.py` for top-level `registry.register()` calls
- `ToolEntry` — metadata: name, toolset, schema, handler, check_fn

### hermes_state.py
- `SessionDB` — SQLite WAL-mode session store
- Schema: `sessions`, `messages`, `state_meta` + FTS5 `messages_fts` virtual table
- FTS triggers for INSERT/UPDATE/DELETE
- Jittered backoff (20–150ms) for write contention

### hermes_cli/commands.py
- `CommandDef` — single command descriptor (name, aliases, category, cli_only, etc.)
- `COMMAND_REGISTRY` — single source of truth for all slash commands
- Consumers: CLI dispatch, gateway dispatch, Telegram BotCommand menu, Slack subcommand map, autocomplete

### gateway/run.py
- `GatewayRunner` — manages platform adapter lifecycle
- Per-session `AIAgent` cache (LRU, max 128, 1-hour idle TTL)
- `_enforce_agent_cache_cap()` — LRU eviction
- `_session_expiry_watcher()` — idle TTL eviction

## Prompts / Rules / Skills Locations

| Location | Content |
|----------|--------|
| `agent/prompt_builder.py` | `DEFAULT_AGENT_IDENTITY`, `PLATFORM_HINTS`, `MEMORY_GUIDANCE`, `SKILLS_GUIDANCE` |
| `~/.hermes/SOUL.md` | User-editable persona file |
| `~/.hermes/MEMORY.md` | Persistent memory |
| `~/.hermes/USER.md` | User profile |
| `AGENTS.md` (project root) | Project-level context |
| `.cursorrules`, `.cursor/rules/*.mdc` | Editor context files |
| `skills/*.md` | Skill instructions injected as user messages |
| `agent/context_compressor.py:38-50` | Compression summary framing |
| `agent/prompt_builder.py:_CONTEXT_THREAT_PATTERNS` | Injection detection patterns |

## Memory / State Locations

| Location | Mechanism |
|----------|----------|
| `~/.hermes/MEMORY.md` | Markdown file, read/written by `memory` tool |
| `~/.hermes/USER.md` | User profile markdown |
| `~/.hermes/state.db` | SQLite: sessions + messages + FTS5 |
| `~/.hermes/cron/jobs.json` | JSON: scheduled job definitions |
| `~/.hermes/cron/output/` | Cron job output files |
| `plugins/memory/` | External providers: Honcho, Hindsight, Mem0 |
| In-process `messages[]` | Ephemeral: conversation history list |

## Tool Integration Locations

| Area | File |
|------|------|
| Tool registry | `tools/registry.py` |
| Tool discovery | `tools/registry.py:discover_builtin_tools()` |
| Tool dispatch | `model_tools.py:handle_function_call()` |
| Toolset definitions | `toolsets.py` |
| Tool schemas exposed | `model_tools.py:get_tool_definitions()` |
| MCP client | `tools/mcp_tool.py` |
| Plugin tools | `model_tools.py` (plugin discovery section) |

## Runtime Assumptions

- Python ≥ 3.11
- Unix-like OS preferred (Windows: WSL2 recommended)
- `~/.hermes/` directory created on first run
- `HERMES_HOME` env var overrides default `~/.hermes/`
- At least one LLM API key required (multiple provider support)
- SQLite WAL mode requires filesystem-level locking support
- Playwright install required for browser tools
- Node.js ≥ 18 required for `--tui` mode

## Uncertainties

- Exact RL training workflow (atropos submodule not fully explored)
- Honcho dialectic user modeling depth (external service)
- Full MCP tool proxy implementation details
- ACP adapter full capabilities (VS Code / JetBrains integration)
- Web dashboard backend API contract (FastAPI server in `web/`)
- Daytona/Modal serverless environment details
- Tinker-Atropos submodule interaction with training loop
