# 10 — Foundation Stack: hermes-agent

## Primary Languages

| Language | Version | Role |
|----------|---------|------|
| Python | ≥3.11 | Core agent, all backend logic |
| TypeScript | ~5.x | TUI (Ink), web dashboard |
| Nix | — | Reproducible build system |
| Shell | bash | Install scripts, entry point shim |

**Evidence**: `pyproject.toml:requires-python = ">=3.11"`, `web/tsconfig.app.json`, `flake.nix`

---

## Package Managers and Build Tools

| Tool | Responsibility | Central or Replaceable |
|------|--------------|----------------------|
| `uv` | Python package install (fast pip replacement) | Central for dev; standard pip works for users |
| `setuptools` | Python package build backend | Standard; defined in `pyproject.toml` |
| `npm` | Node.js package management for TUI and web | Central for TUI mode |
| `vite` | Web dashboard bundler | Replaceable; only used for web UI |
| `tsc` (TypeScript) | TUI/web type checking | Central for TypeScript code |
| `nix` | Reproducible env for CI and NixOS users | Optional but supported via `flake.nix` |

**Evidence**: `pyproject.toml:[build-system]`, `package.json`, `web/vite.config.ts`, `flake.nix`, `setup-hermes.sh` (uses uv)

---

## Runtime Entrypoints

| Entrypoint | Command | File | Process Model |
|-----------|---------|------|---------------|
| `hermes` | `hermes` | `hermes_cli/main.py:main()` + shell shim `hermes` | Single Python process |
| `hermes-agent` | `hermes-agent` | `run_agent.py:main()` | Single Python process |
| `hermes-acp` | `hermes-acp` | `acp_adapter/entry.py:main()` | FastAPI + uvicorn |
| TUI | `hermes --tui` | `ui-tui/src/entry.tsx` + `tui_gateway/entry.py` | Node.js + Python child |
| Gateway | `hermes gateway start` | `gateway/run.py:start_gateway()` | Long-lived Python process + background threads |

**Evidence**: `pyproject.toml:[project.scripts]`, `AGENTS.md:Process Model`

---

## Process Model

- **CLI**: Single Python process, synchronous agent loop, prompt_toolkit REPL on main thread.
- **Gateway**: Single long-lived Python process with per-platform threads (asyncio event loop per platform), background cron thread, synchronous agent sessions per active conversation.
- **TUI**: Two-process: Node.js (Ink, stdio) + Python (tui_gateway, stdio JSON-RPC). Python spawns as a child process of Node.
- **Batch runner**: Multi-threaded Python (ThreadPoolExecutor), one `AIAgent` per parallel task.
- **ACP adapter**: FastAPI + uvicorn ASGI server.

---

## Agentic Framework(s)

Hermes **does not use** an external agentic framework (no LangChain, no LlamaIndex, no AutoGen). The agent loop is a custom implementation in `run_agent.py`. This is a deliberate choice — Hermes is the framework.

**Custom framework elements**:
- Agent loop: `run_agent.py:AIAgent.run_conversation()`
- Tool system: `tools/registry.py` + self-registration pattern
- Context management: `agent/context_engine.py` ABC + `agent/context_compressor.py`
- Memory: `agent/memory_manager.py` + `agent/memory_provider.py` ABC
- Multi-agent: `tools/delegate_tool.py` (child AIAgent spawning)

**Trade-off**: Full control and no framework churn, but all agent infrastructure must be maintained internally. No community plugins from LangChain ecosystem.

---

## API / Web Framework(s)

| Framework | Version | Role | Central or Replaceable |
|----------|---------|------|----------------------|
| `FastAPI` | ≥0.104.0 | ACP adapter HTTP server, web dashboard backend | Central for those features; not used in CLI/gateway |
| `uvicorn` | ≥0.24.0 | ASGI server for FastAPI | Coupled to FastAPI; replaceable (gunicorn, etc.) |
| `aiohttp` | ≥3.13.3 | Async HTTP for messaging adapters (Slack, HomeAssistant, SMS) | Central for those adapters |

**Evidence**: `pyproject.toml:web = ["fastapi>=0.104.0", "uvicorn..."]`, `pyproject.toml:messaging = ["aiohttp>=3.13.3"]`

---

## UI Framework(s)

| Framework | Role | Where Used | Central or Replaceable |
|----------|------|-----------|----------------------|
| `prompt_toolkit` | Interactive REPL with autocomplete | CLI (`cli.py`) | Central for classic CLI; bypassed in TUI mode |
| `rich` | Terminal formatting (panels, tables) | CLI banner, display formatting | Central for CLI output aesthetics |
| `Ink` (React for terminal) | Full-screen TUI | `ui-tui/` | Replaceable; only used in `--tui` mode |
| React + Vite | Web dashboard frontend | `web/src/` | Replaceable; only used for web UI |
| `Tailwind CSS` (inferred) | Web dashboard styling | `web/src/` | Replaceable |

**Evidence**: `pyproject.toml:"prompt_toolkit>=3.0.52"`, `pyproject.toml:"rich>=14.3.3"`, `ui-tui/package.json` (Ink), `web/vite.config.ts`

---

## Storage Technologies

| Technology | Version | Role | Central or Replaceable |
|-----------|---------|------|----------------------|
| SQLite (WAL + FTS5) | Built-in Python | Session store, message history, FTS search | Central; can be replaced behind SessionDB abstraction |
| Filesystem (Markdown files) | — | MEMORY.md, USER.md, SOUL.md, skills, config | Central for default memory layer |
| JSON files | — | Cron jobs, auth.json | Central for scheduling; replaceable |
| YAML files | — | config.yaml, skill manifests | Central for config |
| dotenv files | — | API keys in `.env` | Central for secrets |

**Trade-offs of SQLite**: Zero external dependencies, WAL mode handles moderate concurrency, FTS5 built-in. Limits: single-writer, no replication, no horizontal scaling.

**Evidence**: `hermes_state.py:DEFAULT_DB_PATH`, `hermes_state.py:SCHEMA_SQL`

---

## Vector / Memory Technologies

| Technology | Role | Status |
|-----------|------|--------|
| SQLite FTS5 | Full-text search on message history (BM25, no embeddings) | Active, built-in |
| Honcho | Dialectic user modeling external provider | Optional plugin |
| Hindsight | Memory extraction external provider | Optional plugin |
| Mem0 | Vector-based memory retrieval external provider | Optional plugin |

Note: No embedding/vector store is built into the core. FTS5 provides keyword-based recall. Semantic retrieval requires an external memory provider.

**Evidence**: `pyproject.toml:honcho = ["honcho-ai>=2.0.1"]`, `hermes_state.py:FTS_SQL`

---

## Queue / Job / Scheduling Technologies

| Technology | Role | Evidence |
|-----------|------|---------|
| Built-in cron scheduler | Job scheduling + execution | `cron/scheduler.py`, `cron/jobs.py` |
| `croniter` | Parsing cron expressions | `pyproject.toml:cron = ["croniter>=6.0.0"]` |
| `concurrent.futures.ThreadPoolExecutor` | Parallel tool execution + batch runner | `run_agent.py:_MAX_TOOL_WORKERS=8`, `batch_runner.py` |
| File-based lock (`.tick.lock`) | Prevent concurrent cron ticks | `cron/scheduler.py` |

No external queue (Redis, Celery, etc.) is used. All job scheduling is in-process with file-based coordination.

---

## Auth / Session Technologies

| Technology | Role | Evidence |
|-----------|------|---------|
| `PyJWT[crypto]` | GitHub App JWT auth for Skills Hub | `pyproject.toml:PyJWT[crypto]>=2.12.0` |
| PKCE OAuth flow | Anthropic OAuth authentication | `agent/credential_sources.py:hermes_pkce` |
| Device code flow | Some provider auth | `agent/credential_sources.py:device_code` |
| dotenv-based secrets | Primary API key management | `~/.hermes/.env` |
| `auth.json` | Persisted credential index | `~/.hermes/auth.json` |
| DM pairing (gateway) | Per-user authentication in messaging platforms | `gateway/pairing.py` |

---

## Observability / Logging / Tracing Stack

| Tool | Role | Evidence |
|------|------|---------|
| Python `logging` | All server-side logging | `logging.getLogger(__name__)` throughout |
| `hermes_logging.py` | Logging configuration | Root module |
| Cost tracking (in-process) | Token usage + cost estimate per session | `agent/usage_pricing.py`, `hermes_state.py` |
| `/insights` slash command | Usage analytics CLI | `agent/insights.py` |
| Session DB | Implicit audit log of all turns | `hermes_state.py` |

**Gaps**: No distributed tracing (OpenTelemetry), no structured JSON logs, no metrics endpoint, no alerting integration.

---

## Test Frameworks and Test Organization

| Tool | Version | Role | Evidence |
|------|---------|------|----------|
| `pytest` | ≥9.0.2 | Test runner | `pyproject.toml:dev` extra |
| `pytest-asyncio` | ≥1.3.0 | Async test support | `pyproject.toml:dev` extra |
| `pytest-xdist` | ≥3.0 | Parallel test execution (4 workers) | `pyproject.toml:dev` extra |
| `vitest` | — | TypeScript/Ink unit tests | `ui-tui/package.json` |

**Test organization** (`tests/` directory):
```
tests/
├── conftest.py           — _isolate_hermes_home autouse fixture
├── agent/                — Agent internals tests
├── gateway/              — Gateway tests
├── tools/                — Tool tests
├── hermes_cli/           — CLI command tests
├── cron/                 — Scheduler tests
└── integration/          — Integration tests (marked, skipped in CI without keys)
```

**Key fixture**: `tests/conftest.py:_isolate_hermes_home` redirects `HERMES_HOME` to a temp dir for every test. Tests that write to `~/.hermes/` will fail.

---

## Deployment / Runtime Assumptions

| Assumption | Detail | Evidence |
|-----------|--------|----------|
| Single-server deployment | SQLite is single-process primary writer | `hermes_state.py` design |
| Filesystem access | All state in `HERMES_HOME`; must be writable | `hermes_constants.py` |
| Internet access | LLM API calls require outbound HTTP/HTTPS | `run_agent.py` provider clients |
| Python 3.11+ | Required for match statements, newer stdlib | `pyproject.toml` |
| Unix-like OS | PTY support, file locking via `fcntl` | `pyproject.toml:pty` extra, `cron/scheduler.py:fcntl` |
| Docker supported | `Dockerfile` + `docker/` directory | `Dockerfile` |
| Nix supported | `flake.nix` + CI workflows | `.github/workflows/nix.yml` |
| Termux (Android) supported | Curated `[termux]` extra | `pyproject.toml:termux` extra |
| Windows: WSL2 only | Native Windows not supported | `README.md` |

**Serverless deployment**: Modal and Daytona terminal backends allow the agent to run in serverless environments that hibernate when idle, but the gateway process itself must be persistent.
