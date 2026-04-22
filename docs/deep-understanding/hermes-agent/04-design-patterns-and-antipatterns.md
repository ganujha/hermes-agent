# 04 — Design Patterns and Anti-Patterns: hermes-agent

## Strong Design Patterns

### 1. Self-Registering Tool Registry with AST Discovery
**Pattern**: Each tool file calls `registry.register()` at module level. Discovery uses `ast.parse()` to find these calls without importing the module.
**Why it's strong**: Eliminates a manually maintained import list. Adding a tool requires only creating the file and calling `register()`. No central list to update, no import order issues. The AST-level check prevents false positives from registrations inside functions.
**Evidence**: `tools/registry.py:discover_builtin_tools()`, `_module_registers_tools()`

### 2. Abstract Context Engine with Plugin System
**Pattern**: `ContextEngine` ABC defines the compression lifecycle; the built-in `ContextCompressor` is just one implementation. Users or plugins can replace it via `plugins/context_engine/`.
**Why it's strong**: Context management strategy is fully decoupled from the agent loop. You can swap from LLM-based summarization to a vector-retrieval approach without touching `run_agent.py`.
**Evidence**: `agent/context_engine.py`, `AGENTS.md` plugin architecture

### 3. Single COMMAND_REGISTRY as Source of Truth
**Pattern**: `hermes_cli/commands.py:COMMAND_REGISTRY` is one list of `CommandDef` objects. CLI dispatch, gateway dispatch, Telegram BotCommand menu, Slack subcommand routing, autocomplete, and help text all derive from it.
**Why it's strong**: Adding a slash command in one place automatically propagates to all interfaces. Aliases, help text, and platform availability are properties of the command definition — not scattered across adapter files.
**Evidence**: `hermes_cli/commands.py`, `AGENTS.md:Adding a Slash Command`

### 4. Memory Context Fencing
**Pattern**: External memory provider output is wrapped in `<memory-context>...</memory-context>` with an explicit "this is recalled memory, NOT new user input" system note.
**Why it's strong**: Prevents the model from treating injected memory as new user instructions or discourse. The fence also prevents injection attacks from memory providers attempting to embed instructions in recalled content.
**Evidence**: `agent/memory_manager.py:build_memory_context_block()`, `sanitize_context()`

### 5. Compression Summary "Different Instance" Framing
**Pattern**: Compression summaries are prefixed with `[CONTEXT COMPACTION — REFERENCE ONLY]` and explicitly instruct the model not to answer questions from the summary and to resume from the active task.
**Why it's strong**: Without this framing, the model might hallucinate that old questions in the summary are pending, or use the summary as evidence. The framing prevents instruction bleed.
**Evidence**: `agent/context_compressor.py:38-50` (summary prefix constant)

### 6. Profile-Based Multi-Instance Isolation
**Pattern**: `HERMES_HOME` env var controls all state paths. `get_hermes_home()` is the single function for path resolution. `_apply_profile_override()` sets the env var before any imports at startup.
**Why it's strong**: Complete isolation with zero code duplication. 119+ state references work correctly without modification. Lock files prevent two profiles from using the same API credential.
**Evidence**: `hermes_constants.py:get_hermes_home()`, `hermes_cli/main.py:_apply_profile_override()`

### 7. Prompt Caching as First-Class Architectural Constraint
**Pattern**: AGENTS.md explicitly warns: "Do NOT implement changes that would alter past context mid-conversation, change toolsets mid-conversation, or reload memories mid-conversation." Cache-breaking is treated as a correctness bug.
**Why it's strong**: Forces disciplined context management. The only sanctioned context mutation is compression, which happens atomically.
**Evidence**: `AGENTS.md:Prompt Caching Must Not Break`, `agent/prompt_caching.py`

### 8. Jittered Backoff for SQLite Write Contention
**Pattern**: Write retries use random jitter (20–150ms) instead of fixed delays.
**Why it's strong**: Prevents thundering-herd on the SQLite write lock when multiple platform adapters flush sessions simultaneously. Simple, dependency-free, effective at the scale Hermes operates.
**Evidence**: `hermes_state.py` docstring: "Jittered retries (20-150ms random backoff)"

### 9. Skill Injection as User Messages
**Pattern**: Skills are injected as user messages, not system prompt additions.
**Why it's strong**: Avoids invalidating the cached system prompt when a skill is invoked. The system prompt stays stable across turns; only user-turn content changes when a skill is activated.
**Evidence**: `agent/skill_commands.py` + `AGENTS.md` note on prompt caching

### 10. Data-Driven Skin/Theme System
**Pattern**: Skins are pure YAML data — spinner faces, colors, branding strings — with no code changes needed to add new themes.
**Why it's strong**: Visual customization is fully decoupled from behavior. Users can drop a YAML file in `~/.hermes/skins/` and activate it with `/skin name`.
**Evidence**: `hermes_cli/skin_engine.py:_BUILTIN_SKINS`, user skin loading

---

## Weak Patterns / Risks / Technical Debt

### 1. Process-Global `_last_resolved_tool_names` in `model_tools.py`
**Risk**: This global is saved/restored around child agent runs in `delegate_tool.py`. Under concurrent gateway load (multiple sessions triggering delegation simultaneously), two threads could race on this global.
**Impact**: Child agent receives wrong tool schema; subtle tool-calling errors.
**Fix direction**: Pass resolved tool names as an argument rather than reading from a global.
**Evidence**: `AGENTS.md: _last_resolved_tool_names is a process-global`

### 2. SQLite Single-Writer Under Gateway Concurrency
**Risk**: WAL mode + jittered backoff handles moderate concurrency, but high-volume deployments (many active platform sessions writing simultaneously) will see write latency spikes.
**Impact**: Session persistence delays, potential data loss if process exits during backoff window.
**Fix direction**: Connection pool with write serialization, or Postgres backend for production deployments.
**Evidence**: `hermes_state.py` design note: "single writer"

### 3. 90-Iteration Hard Cap with No Checkpoint/Resume
**Risk**: Long-running autonomous tasks (large codebase refactors, batch processing) can be cut off at 90 iterations with no way to resume.
**Impact**: Partial work, wasted tokens, user frustration on complex tasks.
**Fix direction**: Checkpoint state to disk at intervals; resume protocol for interrupted sessions.
**Evidence**: `run_agent.py:max_iterations=90` default in constructor

### 4. Skills Are LLM-Created Without Quality Gating
**Risk**: The agent creates skills autonomously based on nudge intervals. No automated quality check, deduplication, or review gate.
**Impact**: Skills library accumulates redundant, contradictory, or incorrect procedural instructions over time, degrading agent performance.
**Fix direction**: Skills Hub integration (`agentskills.io`) for community curation; local deduplication check before save.
**Evidence**: `run_agent.py` skill nudge logic, skills directory structure

### 5. Two Separate Config Loaders
**Risk**: `load_cli_config()` (in `cli.py`) and `load_config()` (in `hermes_cli/config.py`) are separate implementations.
**Impact**: Config loading inconsistencies between CLI and gateway modes; difficult to maintain in sync.
**Fix direction**: Consolidate to one config loader.
**Evidence**: `AGENTS.md:Config loaders (two separate systems)` note

### 6. Cross-Tool Reference Anti-Pattern in Schemas
**Risk**: Tool schema descriptions must not name other tools (e.g., "prefer browser_navigate over web_search"). Those tools may not be enabled.
**Impact**: Model hallucinates calls to non-existent tools when a cross-referenced tool is disabled.
**Fix direction**: Dynamic post-processing in `get_tool_definitions()` to inject cross-references only when both tools are enabled.
**Evidence**: `AGENTS.md:DO NOT hardcode cross-tool references in schema descriptions`

### 7. Message List Grows Unboundedly Until Compression Threshold
**Risk**: Compression fires at ~75% context capacity. The 25% headroom may not be enough for tool-heavy turns with large outputs.
**Impact**: Compression can fail if a single turn's tool output + response exceeds the remaining context.
**Fix direction**: Per-turn tool result truncation (partially addressed by `tool_result_storage.py`), proactive compression at 60%.

---

## Hidden Assumptions

- **Filesystem access**: Assumes `~/.hermes/` is writable on the deployment machine. Breaks in read-only container filesystems unless `HERMES_HOME` is redirected.
- **Single Python process per profile**: Profiles use file-based locks, not database-level locks. Running two instances of the same profile concurrently causes undefined behavior.
- **LLM tool-calling support**: The entire agent loop assumes the model supports function/tool calling. Local models without tool support will break the loop.
- **UTF-8 filesystem**: Skill and memory files assume UTF-8. Non-UTF-8 locales or filenames can cause silent read failures.
- **Platform clock monotonicity**: Cron scheduler uses `time.time()` (wall clock). Clock jumps (NTP adjustments, VM migration) can cause jobs to fire prematurely or be skipped.
- **Auxiliary model availability**: Context compression requires the auxiliary LLM client to be available. If the auxiliary model is unreachable, compression fails and the conversation stalls when context is full.

---

## Likely Failure Modes

| Failure | Trigger | Impact |
|---------|---------|--------|
| Compression loop | Auxiliary LLM unreachable when context full | Agent loop hangs; conversation unrecoverable |
| Tool schema hallucination | Disabled tool referenced in schema description | Agent calls non-existent tool, gets error, loops |
| Memory injection attack | Malicious content in external memory provider | Model follows injected instructions |
| Credential exposure | API key in `MEMORY.md` or tool output | Logged to `state.db` FTS index |
| Session cache overflow | >128 concurrent gateway sessions | LRU eviction destroys active sessions |
| Cron job overlap | Slow job + 60s tick | Two instances of same job run concurrently (file lock prevents tick overlap but not job overlap) |
| Global state race in delegation | Concurrent parent agents delegating | Wrong tool schemas in child agents |

---

## Observability Gaps

- **No distributed tracing**: Tool execution spans, LLM latency, and compression events are logged but not traced with span IDs. Debugging multi-tool turns in gateway logs requires manual correlation.
- **No structured logging**: `logging.getLogger(__name__)` used throughout, but no JSON/structured format by default. Parsing logs from production deployments is manual.
- **Cost accounting is estimated, not verified**: Cost estimates use token counts × known pricing. Actual provider charges may differ (discounts, pricing changes).
- **No tool latency histograms**: Tool execution times are not systematically measured or reported. Identifying slow tools in production requires manual log analysis.
- **Cron output is only files**: Cron job outputs are saved to `~/.hermes/cron/output/` but not indexed or searchable without manual file inspection.
- **No health endpoint in gateway**: There's no `/health` HTTP endpoint to check if the gateway is alive and all platform adapters are connected.

---

## Security / Safety Risks

- **Prompt injection in context files**: `agent/prompt_builder.py:_CONTEXT_THREAT_PATTERNS` scans AGENTS.md, SOUL.md, etc. for injection patterns, but this is a blocklist — novel injection techniques bypass it.
- **Tool approval bypass**: Dangerous command detection is regex-based (`tools/approval.py`). Obfuscated commands (base64-encoded, split across calls) may bypass approval prompts.
- **MCP server trust**: The MCP client connects to configured external servers. A compromised or malicious MCP server can expose fake tools with arbitrary descriptions.
- **Skill injection**: A skill file crafted with injection patterns could influence agent behavior. Skill files from external sources (Skills Hub) are not sandboxed.
- **Memory loop risk**: External memory providers that return the agent's own past tool outputs could cause prompt injection via the recalled context injection pathway.
- **DM pairing gap**: The gateway's pairing mechanism requires users to DM the bot first, but this is a per-platform implementation — some platforms may lack robust verification.

---

## Suggestions for Hardening

1. **Replace process-global tool names with thread-local storage** — fix delegation race condition.
2. **Add a structured logging mode** — JSON log format option for production deployments.
3. **Add per-tool timeout enforcement** — currently tools can run indefinitely; add configurable timeouts via the registry.
4. **Implement compression pre-flight at 60%** — fire early before the last 25% of context fills in a single tool-heavy turn.
5. **Add a `/health` gateway endpoint** — simple HTTP endpoint reporting adapter connection status.
6. **Validate MCP tool schemas** — check returned schemas against JSON Schema spec before exposing to the model.
7. **Add skill quality scoring** — simple heuristics (length, reference quality, recency) to surface high-quality skills in the index.
8. **Periodic credential rotation check** — alert when API keys are approaching rate limit quotas or expiry.
9. **Compression fallback** — if auxiliary LLM is unreachable, fall back to simple truncation rather than stalling.
10. **SQLite WAL checkpoint tuning** — the current checkpoint-every-50-writes policy may not be optimal for high-volume deployments; make it configurable.
