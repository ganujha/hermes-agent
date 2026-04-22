# 07 — Open Questions: hermes-agent

## Unclear Parts of the Codebase

**Q1: What exactly does `mixture_of_agents_tool.py` implement?**
The file exists in `tools/` but was not read in detail. It may implement a pattern where multiple models vote on a response, or where different agents with different toolsets collaborate. Needs direct inspection.
- Verification: `cat /home/user/hermes-agent/tools/mixture_of_agents_tool.py`

**Q2: How does `rl_cli.py` interact with the Atropos training loop?**
The RL training system involves `rl_cli.py`, `environments/`, `tinker-atropos/` submodule, and `pyproject.toml`'s `rl` extra. The exact data flow from agent trajectories to Atropos RL environment to model training updates is not fully traced.
- Verification: `cat rl_cli.py` and `ls environments/`

**Q3: What is the full ACP (Agent Client Protocol) integration?**
`acp_adapter/server.py` exposes Hermes as an ACP server, but the full protocol (how VS Code or JetBrains communicate with it, what capabilities are exposed) was not traced.
- Verification: `cat acp_adapter/server.py`, `cat acp_adapter/tools.py`

**Q4: How does the web dashboard backend API work?**
The `web/` directory has a React frontend. There's a FastAPI backend referenced in `pyproject.toml:web = ["fastapi>=0.104.0", "uvicorn..."]`. The exact API endpoints and how the dashboard connects to the running agent were not traced.
- Verification: Find and read the FastAPI app file in the project root or `hermes_cli/`

**Q5: How does `copilot_acp_client.py` relate to GitHub Copilot integration?**
The file `agent/copilot_acp_client.py` suggests Copilot-specific ACP client logic, but the relationship between this and the standard ACP adapter is unclear.
- Verification: `cat agent/copilot_acp_client.py`

**Q6: What is the `gateway/mirror.py` file?**
The gateway directory contains `mirror.py` which was not inspected. Could be a mirroring/forwarding feature between platforms.
- Verification: `cat gateway/mirror.py`

**Q7: How do `gateway/builtin_hooks/` work?**
There's a `gateway/builtin_hooks/` directory not inspected. Gateway hooks presumably fire on events like message received, session start, etc.
- Verification: `ls gateway/builtin_hooks/`

**Q8: What is `tirith_security.py` in tools/?**
The file `tools/tirith_security.py` suggests security scanning functionality. Tirith is an open-source policy enforcement tool. Details not inspected.
- Verification: `cat tools/tirith_security.py`

**Q9: What is `tools/osv_check.py`?**
OSV is the Open Source Vulnerability database. This file may implement dependency vulnerability scanning as an agent tool.
- Verification: `cat tools/osv_check.py`

**Q10: How does voice mode (`tools/voice_mode.py`) work end-to-end?**
The voice mode involves `faster-whisper` (STT), `edge-tts`/ElevenLabs (TTS), and `sounddevice`. The full pipeline from microphone input to agent response to audio output was not traced.
- Verification: `cat tools/voice_mode.py`, `cat tools/tts_tool.py`, `cat tools/transcription_tools.py`

---

## Ambiguous Architecture Areas

**Memory persistence timing**: It's unclear whether `sync_turn()` (post-turn memory write) is fully awaited before the next turn begins, or if it can overlap with the next API call. The docstring says "non-blocking" which suggests fire-and-forget, but this creates a race where memory written in turn N may not be available to prefetch in turn N+1.

**Tool result storage vs message list**: `tools/tool_result_storage.py` manages per-turn tool result budgets and potentially persists large results to disk. The exact boundary between what goes into the in-memory messages list and what is stored externally is not fully clear from documentation alone.

**Context references**: `agent/context_references.py` is listed but its exact role in the context pipeline (how it differs from `prompt_builder.py`) was not deeply traced.

**`google_code_assist.py` vs `gemini_native_adapter.py`**: Two Google adapters exist. The boundary between Google Code Assist and the native Gemini API adapter is unclear.

**`agent/insights.py`**: Not inspected. Likely related to the `/insights` slash command for usage analytics, but details unknown.

---

## Assumptions Made During Analysis

1. **The agent loop is synchronous** — inferred from the `AGENTS.md` statement "entirely synchronous." Not verified by reading the full `run_conversation()` body.

2. **Gateway uses threading (not asyncio)** — inferred from `threading.Thread` imports in `gateway/run.py` header and the synchronous nature of the agent loop. Asyncio is used for platform-specific adapters (Telegram bot uses `asyncio`) but the agent itself runs synchronously.

3. **Skills are injected as user messages at invocation time only** — inferred from AGENTS.md note about prompt caching. The exact mechanism (when and how the skill content is added to messages) was confirmed in `agent/skill_commands.py` but full detail not read.

4. **Context compression is the only sanctioned mid-conversation context mutation** — stated explicitly in AGENTS.md. Treated as a design invariant.

5. **The `_AGENT_CACHE_MAX_SIZE=128` is per-gateway-process** — seems clear from `gateway/run.py` constants, but not verified for multi-process deployments.

6. **FTS5 search is over message content only** — the schema shows `content=messages` in the FTS5 definition, so tool results, tool call arguments, and reasoning content are also indexed if stored in the `content` field.

---

## What Should Be Validated by Running the System

1. **Compression trigger behavior**: Run a long conversation until compression fires; inspect the `[CONTEXT COMPACTION — REFERENCE ONLY]` message content and verify the summary quality.

2. **SQLite write contention**: Run gateway with 5+ active simultaneous sessions and measure write retry rates.

3. **Skill creation nudge**: Observe whether the agent proactively suggests creating a skill after completing a complex task, and what the created skill looks like.

4. **Memory provider injection**: Configure Honcho and send a multi-session conversation; verify memory is recalled correctly with the `<memory-context>` fence.

5. **Delegation chain**: Use `delegate_task` in a conversation; verify the child agent has isolated tool state and the parent receives the child's final response.

6. **Cron job execution**: Create a cron job, wait for it to fire, check `~/.hermes/cron/output/` and verify delivery.

7. **Prompt caching effectiveness**: Observe `cache_read_tokens` vs `cache_write_tokens` in multi-turn conversations; confirm 75% cost reduction on repeated turns.

8. **Profile isolation**: Create two profiles, add different API keys to each, verify they don't cross-contaminate.

---

## What Is Inferred vs Directly Confirmed from Code

| Claim | Status |
|-------|--------|
| Agent loop is synchronous | Confirmed (AGENTS.md + run_agent.py docstring) |
| Max 90 iterations default | Confirmed (constructor signature) |
| SQLite WAL mode | Confirmed (hermes_state.py docstring + SCHEMA_SQL) |
| FTS5 full-text search | Confirmed (FTS_SQL constant) |
| Memory fencing via `<memory-context>` | Confirmed (memory_manager.py code) |
| Prompt injection detection | Confirmed (prompt_builder.py patterns) |
| Tool self-registration via AST | Confirmed (registry.py code) |
| Single external memory provider limit | Confirmed (memory_manager.py docstring) |
| LRU cache with 128 max, 1h TTL | Confirmed (gateway/run.py constants) |
| Process-global tool names race risk | Confirmed (AGENTS.md explicit warning) |
| Compression fires at 75% threshold | Confirmed (context_engine.py default) |
| Skills injected as user messages | Confirmed (AGENTS.md explicit note) |
| Parallel tool execution with 8 workers | Confirmed (run_agent.py constant search) |
| Cron tick every 60s with file lock | Confirmed (scheduler.py docstring) |
| ACP adapter uses FastAPI | Inferred from pyproject.toml + acp_adapter/server.py existence |
| Voice pipeline uses faster-whisper | Confirmed (pyproject.toml voice extra) |
| RL training via Atropos | Confirmed (pyproject.toml rl extra + tinker-atropos submodule) |
| Web dashboard is React/Vite | Confirmed (web/package.json, web/vite.config.ts) |
| Honcho provides dialectic user modeling | Confirmed (pyproject.toml + README) |
| Gateway cron delivery to home channels | Confirmed (cron/scheduler.py _HOME_TARGET_ENV_VARS) |
