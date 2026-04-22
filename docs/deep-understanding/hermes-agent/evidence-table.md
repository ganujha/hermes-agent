# Evidence Table — hermes-agent

| Claim | Evidence File | Confidence | Verification Step |
|-------|--------------|------------|-------------------|
| Primary language is Python ≥ 3.11 | `pyproject.toml:requires-python = ">=3.11"` | High | `python --version` in venv |
| OpenAI SDK is primary LLM wire protocol | `pyproject.toml:openai>=2.21.0`, `run_agent.py:from openai import OpenAI` | High | Read `run_agent.py:1094-1274` |
| Anthropic SDK used natively for Claude | `pyproject.toml:anthropic>=0.39.0`, `agent/anthropic_adapter.py` | High | Read adapter file |
| AWS Bedrock supported | `pyproject.toml:bedrock = ["boto3>=..."]`, `agent/bedrock_adapter.py` | High | Read bedrock adapter |
| Main agent loop is synchronous | `run_agent.py` docstring: "entirely synchronous", `run_agent.py:8464` | High | Read `run_conversation()` body |
| Max 90 tool iterations per conversation | `run_agent.py:AIAgent.__init__ max_iterations=90` | High | Read `__init__` signature |
| SQLite WAL mode for session storage | `hermes_state.py:1-20` docstring: "WAL mode for concurrent readers + one writer" | High | Read SCHEMA_SQL in `hermes_state.py` |
| FTS5 full-text search on messages | `hermes_state.py:FTS_SQL` virtual table + triggers | High | Read FTS_SQL constant |
| Context compression via auxiliary model | `agent/context_compressor.py:compress()` calls auxiliary client | High | Read compress() method |
| Tools are self-registering via AST discovery | `tools/registry.py:discover_builtin_tools()` uses `ast.parse()` | High | Read `_module_registers_tools()` |
| Skills are markdown files with instructions | `skills/` directory contains `.md` files | High | `ls skills/software-development/` |
| Skills injected as user messages (not system) | `agent/skill_commands.py` + AGENTS.md note on prompt caching | High | Read skill_commands.py |
| MEMORY.md/USER.md are built-in memory | `tools/memory_tool.py`, `agent/prompt_builder.py:MEMORY_GUIDANCE` | High | Read memory_tool.py |
| Single external memory provider allowed | `agent/memory_manager.py:1-20` docstring: "Only ONE external...provider at a time" | High | Read MemoryManager.add_provider() |
| Memory context fenced with `<memory-context>` tags | `agent/memory_manager.py:build_memory_context_block()` | High | Read function |
| Prompt injection detection in context files | `agent/prompt_builder.py:_CONTEXT_THREAT_PATTERNS` | High | Read patterns list |
| Prompt caching enabled for Claude on Anthropic/OpenRouter | `run_agent.py:1000-1010` + `agent/prompt_caching.py` | High | Read apply_anthropic_cache_control() |
| All slash commands in single COMMAND_REGISTRY | `hermes_cli/commands.py:COMMAND_REGISTRY` | High | Read commands.py |
| Gateway caches AIAgent instances per session | `gateway/run.py:_AGENT_CACHE_MAX_SIZE=128` | High | Read run.py header |
| Gateway LRU eviction at 128 agents, 1h TTL | `gateway/run.py:_AGENT_CACHE_MAX_SIZE`, `_AGENT_CACHE_IDLE_TTL_SECS=3600.0` | High | Read constants at top of run.py |
| Cron tick runs every 60s from gateway background thread | `cron/scheduler.py` docstring: "every 60 seconds...gateway calls this" | High | Read scheduler.py |
| Cron uses file lock (`.tick.lock`) | `cron/scheduler.py` docstring: "file-based lock...only one tick runs" | High | Read tick() function |
| TUI uses Node.js Ink + JSON-RPC over stdio | `ui-tui/src/gatewayClient.ts`, `tui_gateway/entry.py`, `AGENTS.md:TUI Architecture` | High | Read entry.tsx and tui_gateway/server.py |
| Profile system via HERMES_HOME env var | `hermes_constants.py:get_hermes_home()` reads `HERMES_HOME` | High | Read hermes_constants.py |
| Tool parallel execution capped at 8 workers | `run_agent.py:_MAX_TOOL_WORKERS = 8` | High | Grep run_agent.py for _MAX_TOOL_WORKERS |
| `_NEVER_PARALLEL_TOOLS` list prevents parallel `clarify` | `run_agent.py:_NEVER_PARALLEL_TOOLS` | High | Read that constant |
| Skinning system is data-driven YAML | `hermes_cli/skin_engine.py`, AGENTS.md skin section | High | Read skin_engine.py |
| OpenClaw migration supported | `hermes_cli/main.py` + README migration section | High | Search for "openclaw" in cli.py |
| Delegation creates isolated subagents | `tools/delegate_tool.py:_run_single_child()` | High | Read delegate_tool.py |
| `_last_resolved_tool_names` is a process-global | AGENTS.md: "process-global in model_tools.py" | High | Grep model_tools.py |
| ACP adapter exposes Hermes as ACP server | `acp_adapter/server.py` + `pyproject.toml:acp = ["agent-client-protocol>=0.9.0"]` | High | Read acp_adapter/server.py |
| RL training uses Atropos submodule | `pyproject.toml:rl = ["atroposlib @ git+..."]`, `tinker-atropos/` submodule | High | Read pyproject.toml rl extra |
| Tests use pytest with xdist parallelism | `pyproject.toml:pytest-xdist`, `scripts/run_tests.sh` | High | Read pyproject.toml dev extra |
| Config version 5 triggers migration | AGENTS.md: "Bump `_config_version` (currently 5)" | Medium | Read hermes_cli/config.py DEFAULT_CONFIG |
| Jittered backoff 20-150ms for SQLite contention | `hermes_state.py` docstring: "Jittered retries (20-150ms)" | High | Read write retry logic in hermes_state.py |
| Gateway agent cache is LRU ordered | `gateway/run.py:from collections import OrderedDict` | High | Read _enforce_agent_cache_cap() |
| Ollama num_ctx auto-detected from /api/show | `agent/model_metadata.py:query_ollama_num_ctx()`, mentioned in run_agent.py comment | Medium | Read model_metadata.py |
| Compression summary uses "different assistant" framing | `agent/context_compressor.py:38-50` | High | Read summary prefix constant |
| Skills have frontmatter for platform/condition targeting | `agent/skill_utils.py:parse_frontmatter()`, `skill_matches_platform()` | High | Read skill_utils.py |
| Batch runner produces RL trajectories | `batch_runner.py` docstring | High | Read batch_runner.py header |
| MCP server mode exposes Hermes as MCP | `mcp_serve.py` | High | Read mcp_serve.py header |
| Web dashboard is React/Vite/TypeScript | `web/package.json`, `web/vite.config.ts` | High | Read web/package.json |
| Docusaurus documentation site | `website/docusaurus.config.ts` | High | Read website/ directory |
| NixOS / Nix flake support | `flake.nix`, `.github/workflows/nix.yml` | High | Read flake.nix |
| Docker deployment supported | `Dockerfile`, `docker/` directory | High | Read Dockerfile |
