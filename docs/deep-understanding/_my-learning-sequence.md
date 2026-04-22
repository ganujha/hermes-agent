# My Learning Sequence

A 5-day study plan for deeply understanding hermes-agent.

---

## Day 1: What to Read

**Goal**: Understand the architecture, the design decisions, and where things live.

**Morning — The Foundations (2h)**
1. `docs/deep-understanding/hermes-agent/00-system-overview.md` — BLUF and subsystems
2. `docs/deep-understanding/hermes-agent/01-architecture-map.md` — full architecture diagram, data/control flow
3. `AGENTS.md` (the repo's own developer guide) — read the Agent Loop, File Dependency Chain, and Known Pitfalls sections
4. `hermes_constants.py` — understand the HERMES_HOME pattern (10 min)
5. `pyproject.toml` — understand what's optional vs core (10 min)

**Afternoon — Core Mechanics (2h)**
6. `tools/registry.py` — read in full (30 min)
7. `agent/context_engine.py` — read in full (15 min)
8. `run_agent.py` lines 1–200 and `AIAgent.__init__` + `run_conversation()` — understand the loop (45 min)
9. `hermes_state.py` lines 1–120 — understand the session schema (15 min)
10. `docs/deep-understanding/hermes-agent/10-foundation-stack.md` — understand what third-party components are used

**Evening — Synthesis (1h)**
11. Read `docs/deep-understanding/hermes-agent/04-design-patterns-and-antipatterns.md`
12. Read `docs/deep-understanding/hermes-agent/06-glossary.md` — annotate any terms you found confusing

---

## Day 2: What to Run

**Goal**: Watch the system behave in practice. Confirm architectural claims from Day 1.

**Morning — Setup and First Contact (2h)**
1. Set up the environment: `source venv/bin/activate` (or `uv venv && uv pip install -e ".[all,dev]"`)
2. Run `hermes` — start a conversation, observe the spinner, tool calls, response streaming
3. Use the `terminal` tool in conversation: "what is in the current directory?"
4. Use the `web_search` tool: "search for recent news about AI agents"
5. After the session, inspect `~/.hermes/state.db` with sqlite3 (see runbook)

**Afternoon — Memory and Skills (2h)**
6. In conversation: "Please remember that I am a Python developer who prefers type annotations"
7. Exit hermes, then start again — verify the memory was loaded
8. Inspect `~/.hermes/MEMORY.md`
9. Ask hermes to create a skill: "create a skill for analyzing Python codebases"
10. Inspect `~/.hermes/skills/` — read the generated skill file
11. Invoke the skill: `/analyze-python-codebase` (or whatever it was named)

**Evening — Context Compression (1h)**
12. Set `context.threshold_percent: 0.1` in `~/.hermes/config.yaml`
13. Start a long conversation with many tool calls until compression fires
14. Find the `[CONTEXT COMPACTION — REFERENCE ONLY]` message in the transcript
15. Check the SQLite sessions table for the `parent_session_id` chain

---

## Day 3: What to Compare

**Goal**: Understand the design decisions by comparing alternatives and testing boundaries.

**Morning — Tool System Deep Dive (2h)**
1. Run the tool discovery code manually (see runbook) — count registered tools
2. Read `toolsets.py:_HERMES_CORE_TOOLS` — understand what's universal vs optional
3. Read `tools/delegate_tool.py` — understand child agent isolation
4. Compare: delegate a subtask vs asking the agent to do it directly. What's different?
5. Read `model_tools.py:get_tool_definitions()` — how toolset filtering works

**Afternoon — Memory Architecture Comparison (2h)**
6. Read `agent/memory_manager.py` in full — understand the fencing model
7. Compare MEMORY.md (semantic) vs FTS5 session search (episodic): when would you use each?
8. If you have a Honcho API key, configure it and observe how memory injection works
9. Read `agent/context_compressor.py:compress()` — compare to simple truncation alternatives

**Evening — Gateway and Multi-Platform (1h)**
10. Read `gateway/run.py` header and `GatewayRunner.__init__` — understand the LRU cache
11. Read `hermes_cli/commands.py:COMMAND_REGISTRY` — count how many consumers derive from it
12. Compare the CLI and gateway dispatch paths — how much do they share?

---

## Day 4: What to Explain Out Loud

**Goal**: Solidify understanding by articulating it. Use the interview questions as prompts.

**Morning — System Design Explanations (2h)**
Explain out loud (to a person, a rubber duck, or in writing):
1. "How does Hermes prevent context window overflow?" (compression pipeline)
2. "How does Hermes remember things across conversations?" (three memory tiers)
3. "How do tools get discovered and dispatched?" (AST registry, handle_function_call)
4. "How does the gateway serve 16 platforms from one agent?" (adapter pattern, shared session loop)
5. "Why can't you add a second memory provider?" (tool schema bloat, conflicting user models)

**Afternoon — Architecture Trade-Off Explanations (2h)**
6. "Why SQLite instead of Postgres?" — argue both sides
7. "Why is the agent loop synchronous?" — when would async matter?
8. "Why are skills injected as user messages?" — connect to prompt caching invariant
9. "What would break if the HERMES_HOME pattern didn't exist?" — walk through a profile scenario
10. "What is the biggest reliability risk in the current architecture?" (SQLite, process-global, no tool timeouts)

**Evening — Code Reading Practice (1h)**
11. Pick one tool you haven't read yet (e.g., `tools/browser_tool.py` or `tools/mcp_tool.py`)
12. Read it without looking at docs — reconstruct what it does from code alone
13. Compare your reconstruction to the registry entry and schema description

---

## Day 5: What to Challenge / Criticize

**Goal**: Develop critical judgment. Identify what you would change and why.

**Morning — Find the Weak Points (2h)**
1. Read `docs/deep-understanding/hermes-agent/07-open-questions.md` — pick 3 open questions and investigate them in code
2. Find the two-config-loader issue: compare `cli.py:load_cli_config()` vs `hermes_cli/config.py:load_config()` — what diverges?
3. Find `_last_resolved_tool_names` in `model_tools.py` — verify the race condition scenario described in AGENTS.md
4. Count how many lines of `run_agent.py` are in the `AIAgent` class vs helper functions — is it too monolithic?

**Afternoon — Design Alternatives (2h)**
5. Sketch: How would you replace SQLite with Postgres? What interface changes are needed in `hermes_state.py`?
6. Sketch: How would you add per-tool timeouts? What changes to `tools/registry.py` and `model_tools.py:handle_function_call()`?
7. Sketch: How would you fix the `_last_resolved_tool_names` global? Thread-local storage? Pass as argument?
8. Sketch: What would a structured logging system look like for this codebase? Where would you instrument?

**Evening — Teach It Back (1h)**
9. Write a 3-paragraph description of hermes-agent for:
   - A new engineer on the team
   - A staff engineer reviewing the architecture
   - A CTO deciding whether to adopt it
10. Compare your descriptions to `docs/deep-understanding/hermes-agent/09-teaching-note.md` — what did you miss or see differently?
11. Write 5 questions you still have about the codebase — add them to `docs/deep-understanding/hermes-agent/07-open-questions.md`
