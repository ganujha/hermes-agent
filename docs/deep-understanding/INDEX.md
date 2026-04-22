# Deep Understanding Materials — Index

Generated analysis of the **hermes-agent** repository by Nous Research.

---

## Reading Order by Goal

### "I have 30 minutes — give me the core idea"
1. [`hermes-agent/00-system-overview.md`](hermes-agent/00-system-overview.md) — BLUF, problem, major subsystems
2. [`hermes-agent/01-architecture-map.md`](hermes-agent/01-architecture-map.md) — architecture diagram + control flow

### "I have 2 hours — I want to understand the architecture"
1. Above, then:
2. [`hermes-agent/02-agent-tool-memory-analysis.md`](hermes-agent/02-agent-tool-memory-analysis.md) — agent loop, tools, memory
3. [`hermes-agent/10-foundation-stack.md`](hermes-agent/10-foundation-stack.md) — technologies used and why
4. [`hermes-agent/04-design-patterns-and-antipatterns.md`](hermes-agent/04-design-patterns-and-antipatterns.md) — strong patterns + risks

### "I have a day — deep mastery"
1. All above, then:
2. [`hermes-agent/03-codebase-walkthrough.md`](hermes-agent/03-codebase-walkthrough.md) — recommended file reading order
3. [`hermes-agent/11-component-decomposition.md`](hermes-agent/11-component-decomposition.md) — bottom-up layer analysis
4. [`hermes-agent/09-teaching-note.md`](hermes-agent/09-teaching-note.md) — teach it from three perspectives
5. [`hermes-agent/08-runbook-for-learning.md`](hermes-agent/08-runbook-for-learning.md) — run it and observe

### "I'm preparing for an interview"
1. [`hermes-agent/05-interview-extraction.md`](hermes-agent/05-interview-extraction.md) — 30 questions with answer bullets
2. [`hermes-agent/04-design-patterns-and-antipatterns.md`](hermes-agent/04-design-patterns-and-antipatterns.md) — trade-offs
3. [`hermes-agent/09-teaching-note.md`](hermes-agent/09-teaching-note.md) — staff/CTO framing

### "I want to contribute to the codebase"
1. [`hermes-agent/raw-findings.md`](hermes-agent/raw-findings.md) — file map + key classes
2. [`hermes-agent/03-codebase-walkthrough.md`](hermes-agent/03-codebase-walkthrough.md) — top 20 files explained
3. [`hermes-agent/07-open-questions.md`](hermes-agent/07-open-questions.md) — unclear areas to investigate
4. [`hermes-agent/08-runbook-for-learning.md`](hermes-agent/08-runbook-for-learning.md) — setup + experiment commands

### "I want to design a similar system"
1. [`_best-patterns-playbook.md`](_best-patterns-playbook.md) — reusable architecture patterns
2. [`hermes-agent/11-component-decomposition.md`](hermes-agent/11-component-decomposition.md) — layer-by-layer decomposition
3. [`hermes-agent/02-agent-tool-memory-analysis.md`](hermes-agent/02-agent-tool-memory-analysis.md) — agent/tool/memory design choices

---

## Complete File Index

### hermes-agent Repo Analysis

| File | Description |
|------|-------------|
| [`hermes-agent/raw-findings.md`](hermes-agent/raw-findings.md) | Repo map, entrypoints, key files, classes, uncertainties |
| [`hermes-agent/evidence-table.md`](hermes-agent/evidence-table.md) | Claim → evidence → confidence → verification |
| [`hermes-agent/00-system-overview.md`](hermes-agent/00-system-overview.md) | BLUF, problem, users, subsystems, external dependencies |
| [`hermes-agent/01-architecture-map.md`](hermes-agent/01-architecture-map.md) | Mermaid diagrams, control flow, data flow, state locations |
| [`hermes-agent/02-agent-tool-memory-analysis.md`](hermes-agent/02-agent-tool-memory-analysis.md) | Agent loop, multi-agent patterns, tool invocation, memory model |
| [`hermes-agent/03-codebase-walkthrough.md`](hermes-agent/03-codebase-walkthrough.md) | Top 20 files, reading order (30min / 2hr / 1day) |
| [`hermes-agent/04-design-patterns-and-antipatterns.md`](hermes-agent/04-design-patterns-and-antipatterns.md) | 10 strong patterns, risks, failure modes, security gaps, hardening suggestions |
| [`hermes-agent/05-interview-extraction.md`](hermes-agent/05-interview-extraction.md) | 30 interview questions (system design, debugging, trade-offs) with answer bullets |
| [`hermes-agent/06-glossary.md`](hermes-agent/06-glossary.md) | Repo vocabulary, internal abstractions, execution lifecycle terms |
| [`hermes-agent/07-open-questions.md`](hermes-agent/07-open-questions.md) | Unclear areas, ambiguities, inferred vs confirmed claims |
| [`hermes-agent/08-runbook-for-learning.md`](hermes-agent/08-runbook-for-learning.md) | Exact commands, where logs appear, how to trigger workflows safely |
| [`hermes-agent/09-teaching-note.md`](hermes-agent/09-teaching-note.md) | Teach to engineer / staff+ / CTO — three framings |
| [`hermes-agent/10-foundation-stack.md`](hermes-agent/10-foundation-stack.md) | Languages, frameworks, storage, observability, test setup — with evidence |
| [`hermes-agent/11-component-decomposition.md`](hermes-agent/11-component-decomposition.md) | 7-layer bottom-up decomposition: infrastructure → interface |

### Synthesis Documents

| File | Description |
|------|-------------|
| [`_cross-repo-comparison.md`](_cross-repo-comparison.md) | Architecture comparison across repos (extend when more repos are analyzed) |
| [`_best-patterns-playbook.md`](_best-patterns-playbook.md) | Reusable architecture, memory, safety, and developer patterns |
| [`_my-learning-sequence.md`](_my-learning-sequence.md) | 5-day study plan: read → run → compare → explain → challenge |

---

## Key Concepts Quick Reference

| Concept | Where Defined | File |
|---------|--------------|------|
| Agent loop | `AIAgent.run_conversation()` | `run_agent.py` |
| Tool registry | `ToolRegistry` | `tools/registry.py` |
| Context engine | `ContextEngine` ABC | `agent/context_engine.py` |
| Context compression | `ContextCompressor` | `agent/context_compressor.py` |
| Memory manager | `MemoryManager` | `agent/memory_manager.py` |
| Memory fencing | `build_memory_context_block()` | `agent/memory_manager.py` |
| Profile isolation | `get_hermes_home()` | `hermes_constants.py` |
| Session store | `SessionDB` | `hermes_state.py` |
| Slash commands | `COMMAND_REGISTRY` | `hermes_cli/commands.py` |
| Gateway | `GatewayRunner` | `gateway/run.py` |
| Skill system | `skills/` + `tools/skill_manager_tool.py` | multiple |
| Cron scheduler | `tick()` | `cron/scheduler.py` |
| Delegation | `delegate_task` | `tools/delegate_tool.py` |
| Skin system | `skin_engine.py` | `hermes_cli/skin_engine.py` |
| Prompt caching | `apply_anthropic_cache_control()` | `agent/prompt_caching.py` |
