# 12 — Second-Pass Validation: hermes-agent

This document revisits every major first-pass claim and labels each with code-confirmed evidence. Corrections and nuances are called out explicitly.

**Evidence labels used:**
- ✅ **Confirmed from code** — direct source inspection with file:line
- 🔶 **Strong inference** — multiple indirect signals, no single definitive line
- ⚠️ **Weak inference** — plausible but unverified
- ❌ **Incorrect** — code contradicts the claim

---

## Agent Loop

| Claim | Status | Evidence |
|-------|--------|------|
| `run_conversation()` is synchronous (no `async def`, no `await`) | ✅ Confirmed | `run_agent.py:8464` — `def run_conversation(...)`, no `async`; grep for `await` returns zero hits in the method |
| Main loop is a `while` statement | ✅ Confirmed | `run_agent.py:8833` — `while iteration < max_iterations:` |
| Default `max_iterations = 90` | ✅ Confirmed | `run_agent.py:719` — `max_iterations: int = 90,` in constructor signature |
| Max iterations is configurable | ✅ Confirmed | `hermes_cli/config.py:3051–3056` — `max_turns` key merged to `agent.max_turns`; `cli.py:318` has matching default |
| Parallel tool execution uses 8 workers | ✅ Confirmed | `run_agent.py:269` — `_MAX_TOOL_WORKERS = 8`; used at line 7763 inside `ThreadPoolExecutor` |
| Parallel execution is conditional | ✅ Confirmed | `run_agent.py:299–340` — `_should_parallelize_tool_batch()` checks `_NEVER_PARALLEL_TOOLS`, path overlaps, `_PARALLEL_SAFE_TOOLS` allowlist |

---

## Tool Registry

| Claim | Status | Evidence |
|-------|--------|------|
| Tools self-register at import time | ✅ Confirmed | `tools/registry.py` — `registry.register()` called at module level |
| Discovery uses `ast.parse()` (no import) | ✅ Confirmed | `tools/registry.py:discover_builtin_tools()` — AST scan for `registry.register(` calls |

---

## Memory System

| Claim | Status | Evidence |
|-------|--------|------|
| Three-tier memory: ephemeral / semantic / episodic | ✅ Confirmed | `agent/memory_manager.py` (semantic via MEMORY.md), `hermes_state.py` (SQLite FTS5 episodic), in-process message list (ephemeral) |
| `<memory-context>` fencing prevents injection | ✅ Confirmed | `agent/memory_manager.py:build_memory_context_block()` — tags wrapping injected content |
| Single external memory provider constraint | ✅ Confirmed | `agent/memory_manager.py` docstring — "only one external provider active at a time" |
| Memory sync (`sync_turn`) is fire-and-forget | ❌ **INCORRECT** | `agent/memory_manager.py:210–219` — `sync_all()` is `def` (not `async def`), calls `provider.sync_turn()` in a blocking `for` loop. No thread spawn. Memory writes **block the agent loop turn**. Race risk (turn N memory not persisting before turn N+1) does NOT exist; but latency from slow providers DOES block response delivery. |

**Correction note:** The first-pass claim "non-blocking fire-and-forget" was inferred from the docstring description "sync." The code shows sequential blocking calls. Implication: a slow external memory provider (e.g., slow Honcho API call) directly delays the next user turn response.

---

## Context Compression

| Claim | Status | Evidence |
|-------|--------|------|
| `ContextEngine` is an ABC | ✅ Confirmed | `agent/context_engine.py` — `class ContextEngine(ABC)` |
| Default `threshold_percent = 0.75` (class default) | ✅ Confirmed | `agent/context_engine.py:59` — `threshold_percent: float = 0.75` |
| CLI default is 0.50 (not 0.75) | ✅ Confirmed | `cli.py:315` — `"threshold": 0.50` passed as `threshold_percent` |
| **Nuance:** Two different defaults exist | 🔶 | Class default is 0.75, but CLI applies 0.50 in practice. First-pass said "compression fires at 75% threshold" — true for the ABC, but CLI users get 50%. |
| Compression fires when `should_compress()` returns True | ✅ Confirmed | `agent/context_engine.py` — `should_compress()` method checks `last_prompt_tokens / context_length >= threshold_percent` |
| Auxiliary (secondary) LLM performs the summarization | ✅ Confirmed | `agent/context_compressor.py` — `auxiliary_client` instantiation |

---

## Skills Injection

| Claim | Status | Evidence |
|-------|--------|------|
| Skills injected as user messages | ⚠️ **Nuanced, partially incorrect** | Two code paths exist — see below |

**Detailed finding (important correction):**

Two distinct skills injection paths exist in the codebase:

1. **Preloaded skills list** (`build_skills_system_prompt()`): Called in `run_agent.py:4106–4113` inside `_build_system_prompt()`. Returns a text block listing available skills, which is **appended to `prompt_parts`** and becomes part of the system prompt. This is metadata about what skills exist.

2. **Slash-command invocation** (`build_skill_invocation_message()`): `agent/skill_commands.py:429–465` — labeled "Build the user message content for a skill slash command invocation." The actual skill body is injected as a user message when the user types `/skill-name`. This is the path AGENTS.md describes.

**Resolution:** AGENTS.md is correct for invocation (user message), but incomplete — skills are also listed in the system prompt. The system prompt receives skill *metadata* (available skills list); user messages receive skill *content* (when invoked). The first-pass claim from AGENTS.md ("skills injected as user messages") is correct for invocation but overstated as the complete picture.

**Implication for prompt caching:** Since skill invocation modifies the user message (not system prompt), prompt cache validity is preserved for cached system prompt segments.

---

## `_last_resolved_tool_names` Global

| Claim | Status | Evidence |
|-------|--------|------|
| Process-global variable in `model_tools.py` | ✅ Confirmed | `model_tools.py:159` — `_last_resolved_tool_names: List[str] = []` |
| Written in `get_tool_definitions()` | ✅ Confirmed | `model_tools.py:337–338` — `global _last_resolved_tool_names; _last_resolved_tool_names = [...]` |
| `delegate_tool.py` saves/restores around child runs | ✅ Confirmed | `tools/delegate_tool.py:100–113` |
| Race condition under concurrent delegation | ✅ Confirmed (as risk) | Save/restore pattern in delegate_tool is not atomic; concurrent parent agents delegating simultaneously share the same module-level list |

---

## Configuration System

| Claim | Status | Evidence |
|-------|--------|------|
| Two separate config loaders exist | ✅ Confirmed | `cli.py:load_cli_config()` vs `hermes_cli/config.py:load_config()` |
| `load_cli_config()` fallback to `./cli-config.yaml` | ✅ Confirmed | `cli.py:268–400` — checks `~/.hermes/config.yaml` then `./cli-config.yaml` |
| `load_config()` only uses `~/.hermes/config.yaml` | ✅ Confirmed | `hermes_cli/config.py:3039–3066` — uses `get_config_path()` only |
| `load_config()` migrates legacy `max_turns` key | ✅ Confirmed | `hermes_cli/config.py:3051–3056` — `_normalize_max_turns_config()` |
| CLI loader includes personality templates | ✅ Confirmed | `cli.py:324–339` — kawaii, catgirl, pirate, shakespeare, surfer, noir, uwu, philosopher, hype |
| Only `load_config()` uses `_deep_merge()` | ✅ Confirmed | `hermes_cli/config.py` has `_deep_merge()` call; CLI uses simpler dict operations |

---

## Gateway

| Claim | Status | Evidence |
|-------|--------|------|
| LRU cache max 128 agents | ✅ Confirmed | `gateway/run.py` — `_AGENT_CACHE_MAX_SIZE = 128` |
| 1-hour TTL | ✅ Confirmed | `gateway/run.py` — `_AGENT_CACHE_IDLE_TTL_SECS = 3600.0` |
| `OrderedDict` implements LRU | ✅ Confirmed | `gateway/run.py:675` — `self._agent_cache: OrderedDict[str, tuple] = OrderedDict()` |
| `_enforce_agent_cache_cap()` evicts oldest | ✅ Confirmed | `gateway/run.py:8738–8798` — iterates `list(_cache.keys())[:excess]` for eviction; skips active agents |
| Cron runs as a daemon thread | ✅ Confirmed | `gateway/run.py:11031–11038` — `threading.Thread(target=_start_cron_ticker, daemon=True, name="cron-ticker")` |
| Cron tick interval is 60 seconds | ✅ Confirmed | `cron/scheduler.py` — 60s interval with `fcntl` file lock |

---

## SQLite / State

| Claim | Status | Evidence |
|-------|--------|------|
| WAL mode enabled | ✅ Confirmed | `hermes_state.py:SCHEMA_SQL` — `PRAGMA journal_mode=WAL` |
| FTS5 full-text search | ✅ Confirmed | `hermes_state.py:FTS_SQL` — `CREATE VIRTUAL TABLE messages_fts USING fts5(content=messages, ...)` |
| `parent_session_id` foreign key in schema | ✅ Confirmed | `hermes_state.py:36–98` — `parent_session_id TEXT REFERENCES sessions(id)`, indexed at `idx_sessions_parent` |
| Jittered backoff on write contention | ✅ Confirmed | `hermes_state.py` — 20–150ms random sleep before retry on `OperationalError` |

---

## Prompt Injection Detection

| Claim | Status | Evidence |
|-------|--------|------|
| 10 regex threat patterns | ✅ Confirmed | `agent/prompt_builder.py:36–47` — exactly 10 patterns: `prompt_injection`, `deception_hide`, `sys_prompt_override`, `disregard_rules`, `bypass_restrictions`, `html_comment_injection`, `hidden_div`, `translate_execute`, `exfil_curl`, `read_secrets` |
| Blocked content replaced with warning string | ✅ Confirmed | `agent/prompt_builder.py:65–71` — `[BLOCKED: {filename} contained potential prompt injection ...]` |

---

## First-Pass Correction Summary

| # | Original Claim | Correction |
|---|---------------|----------|
| 1 | Memory sync is fire-and-forget / non-blocking | **WRONG** — it's blocking sequential in a `for` loop. Slow providers delay turn responses. |
| 2 | Skills are injected exclusively as user messages | **INCOMPLETE** — slash-command invocations → user messages; preloaded skills list → system prompt metadata |
| 3 | Compression fires at 75% threshold | **PARTIALLY WRONG** — class default is 0.75, but CLI overrides to 0.50. In practice, CLI users compress at 50%. |
| 4 | All other major claims | ✅ Confirmed |
