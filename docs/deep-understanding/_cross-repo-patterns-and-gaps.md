# Cross-Repo Patterns and Gaps: hermes-agent

Patterns worth extracting for any agent or AI-powered system. Gaps that any team building in this space should address.

---

## Best Patterns (Extractable to Any Codebase)

### Pattern 1: Single-Var State Isolation
**What**: One environment variable (`HERMES_HOME`) controls where all persistent state writes go. Every file write, DB connection, and config load routes through `get_hermes_home()`.

**Why it works**: Test isolation is free — redirect the var to a temp dir. Multiple user profiles need no code changes. Clean demos need no state cleanup. Uninstall is `rm -rf ~/.hermes`.

**How to implement in any system**:
```python
# Single resolution function
def get_app_home() -> Path:
    return Path(os.environ.get("APP_HOME", "~/.myapp")).expanduser()

# Autouse test fixture
@pytest.fixture(autouse=True)
def _isolate_app_home(tmp_path, monkeypatch):
    monkeypatch.setenv("APP_HOME", str(tmp_path / "app"))
```

**Applicable to**: Any system that writes config, cache, logs, or state to the filesystem.

---

### Pattern 2: AST-Based Self-Registration
**What**: Module-level calls to a registry function (`registry.register(name, func, schema)`). Discovery uses `ast.parse()` to find these calls without executing the modules.

**Why it works**: Zero circular import risk. Zero startup side effects from unneeded modules. Adding a new plugin/tool is additive (one file, one registration call). Discovery is grep-equivalent but structured.

**How to implement**:
```python
# Discovery (no import)
def discover_plugins(plugin_dir: Path) -> list[str]:
    modules = []
    for py_file in plugin_dir.glob("*.py"):
        tree = ast.parse(py_file.read_text())
        for node in ast.walk(tree):
            if (isinstance(node, ast.Call) and
                hasattr(node.func, 'attr') and
                node.func.attr == 'register'):
                modules.append(str(py_file))
    return modules
```

**Applicable to**: Plugin systems, command registries, event handler registries, tool catalogs.

---

### Pattern 3: Prompt Cache Invariant as Architecture
**What**: System prompt is built once at session/request start and never modified. All dynamic content uses alternative injection channels (user message fencing, tool results).

**Why it works**: ~75% cost reduction on multi-turn LLM conversations via Anthropic prompt caching. Predictable agent behavior (no mid-session system prompt drift). Forces disciplined context design.

**How to implement**:
- Document the invariant explicitly in architecture docs
- Code review: any function that modifies the system prompt mid-session is a violation
- Test: assert that system prompt hash doesn't change between turns in a session
- Alternative channels: user message `<context-fence>` tags, tool result injection

**Applicable to**: Any multi-turn LLM system using Anthropic models.

---

### Pattern 4: Context Fence for Injected Content
**What**: External content injected into LLM context is wrapped in semantic tags (`<memory-context>`, `<retrieved-docs>`, etc.). The model's training is relied upon to interpret the fence semantics.

**Why it works**: Signals to the model that the fenced content is recalled/injected, not user instruction. Reduces prompt injection risk from malicious content in retrieved data. Provides structure for the model to differentiate content sources.

**How to implement**:
```python
def build_context_block(recalled_text: str, source: str = "memory") -> str:
    return f"<{source}-context>\n{recalled_text}\n</{source}-context>"

# In message assembly:
user_content = f"{context_block}\n\n{user_message}"
```

**Applicable to**: RAG systems, memory injection, tool result injection, multi-source context assembly.

---

### Pattern 5: Single Command Registry for Multi-Interface Systems
**What**: A single list of command definitions (`COMMAND_REGISTRY`) drives all interfaces: CLI help text, bot menus, API routing, autocomplete, gateway dispatch.

**Why it works**: Single source of truth — add a command once, available everywhere. Per-interface config gates (`gateway_config_gate`) allow CLI-only defaults with opt-in activation. Documentation is auto-generated from the registry.

**How to implement**:
```python
@dataclass
class CommandDef:
    name: str
    description: str
    handler: str  # import path
    interfaces: list[str]  # ["cli", "gateway", "telegram"]
    config_gate: Optional[str] = None  # "features.gateway_commands"

COMMAND_REGISTRY = [
    CommandDef(name="search", description="Search conversation history", ...),
    ...
]
```

**Applicable to**: Any system with multiple interfaces (CLI + API + bot + webhook) sharing command semantics.

---

### Pattern 6: Parallel Safety Allowlist + Blocklist for Concurrent Operations
**What**: Rather than allowing all operations to run in parallel (unsafe) or forcing all operations sequential (slow), maintain: (1) explicit blocklist of never-parallel operations, (2) explicit allowlist of safe-for-parallel operations, (3) path-overlap detection for file operations.

**Why it works**: Maximizes throughput for safe operations. Prevents race conditions for unsafe ones. Explicit lists make safety reasoning auditable.

**How to implement**:
```python
_NEVER_PARALLEL = {"delete_file", "write_file", "execute_shell"}
_PARALLEL_SAFE = {"read_file", "fetch_url", "query_db_read"}

def should_parallelize(operations: list[str]) -> bool:
    if any(op in _NEVER_PARALLEL for op in operations):
        return False
    if not all(op in _PARALLEL_SAFE for op in operations):
        return False
    return True
```

**Applicable to**: Tool execution engines, workflow orchestrators, test runners.

---

### Pattern 7: Iteration Budget for Agent Loop Safety
**What**: The agent loop has a hard cap on iterations (`max_iterations = 90`) plus an `IterationBudget` object that tracks remaining budget and can be shared with child agents.

**Why it works**: Prevents infinite loops from bugs or adversarial inputs. Budget sharing ensures a parent + its child agents can't collectively exceed the budget by delegation.

**How to implement**:
```python
class IterationBudget:
    def __init__(self, total: int):
        self.total = total
        self.used = 0
    
    @property
    def remaining(self) -> int:
        return self.total - self.used
    
    def consume(self, n: int = 1):
        self.used += n
```

**Applicable to**: Any agent loop, recursive workflow system, LLM-powered automation.

---

## Recurring Gaps (What's Missing Across Agent Systems)

### Gap 1: Per-Operation Timeout Enforcement
**What's missing**: No per-tool or per-operation timeout in hermes. Tools can hang indefinitely.

**Why it's everywhere**: Timeout enforcement requires wrapping every operation in a future with a deadline. Most systems add this retroactively after the first production hang.

**Standard fix**:
```python
with concurrent.futures.ThreadPoolExecutor(max_workers=1) as ex:
    future = ex.submit(tool_func, **args)
    try:
        result = future.result(timeout=tool_entry.timeout_seconds)
    except concurrent.futures.TimeoutError:
        return ToolResult(error=f"Tool {tool_name} timed out after {timeout}s")
```

---

### Gap 2: Structured Observability
**What's missing**: Python `logging` produces unstructured text. No OpenTelemetry spans, no Prometheus metrics, no `/healthz` endpoint.

**Why it's everywhere**: Observability is treated as ops concern, not engineering concern. Until the first production incident where you can't tell what happened, it stays unstructured.

**Minimum viable observability for an agent system**:
- Structured JSON logs with: session_id, turn_number, tool_name, latency_ms, token_count, error
- One histogram: turn_latency_seconds
- One counter: tool_calls_total{tool_name, success}
- One gauge: active_sessions
- One health endpoint: returns 200 with { "db": "ok", "memory": "ok" }

---

### Gap 3: Async Memory Sync
**What's missing**: Memory providers are called synchronously at end of turn, blocking response delivery.

**Why it's everywhere**: Memory sync was designed as "just a write" — seems like it should be fast. External API latency reveals the assumption was wrong.

**Standard fix**: Background thread pool for async writes. Accept eventual consistency in memory recall.

---

### Gap 4: Config Unification
**What's missing**: Multiple config loaders with divergent logic that must be manually kept in sync.

**Why it's everywhere**: Config systems grow organically as different subsystems add their own loading logic. Unification is boring work that doesn't ship features.

**Standard fix**: Enforce a single config loader with a mode parameter from day one. All callers use the same function.

---

### Gap 5: Quality Gates for Self-Generated Content
**What's missing**: No automated review of LLM-created skills, memories, or tools.

**Why it's everywhere**: Self-improvement is often treated as the "fun" part of the system. Quality control on generated artifacts is unglamorous.

**Standard fix for skills specifically**:
1. Schema validation on creation (does the skill have required fields?)
2. Semantic deduplication (cosine similarity check against existing skills)
3. Periodic quality audit (judge model rates skill effectiveness after N invocations)
4. TTL: skills expire after 90 days unless recently invoked

---

## Combined Architecture (What a Production-Grade Version Would Look Like)

Taking hermes's best patterns and filling the gaps:

```
User interfaces (CLI, TUI, Gateway, ACP)
    ↓ (COMMAND_REGISTRY single source of truth)
AIAgent.run_conversation() (synchronous core)
    ↓
Tool execution (parallel + sequential, WITH per-tool timeouts)
    ↓ (PARALLEL_SAFE allowlist + NEVER_PARALLEL blocklist)
Provider abstraction (OpenAI wire format, native Anthropic SDK)
    ↓ (prompt cache invariant enforced)
Context management (ContextEngine ABC, threshold compression)
    ↓
Session storage (Postgres with connection pool OR SQLite for dev)
    ↓ (SessionDB interface hides backend)
Memory sync (background thread pool, eventual consistency)
    ↓ (<memory-context> fencing for injection)
Observability (structured JSON logs, OTel spans, Prometheus metrics)
    ↓
State isolation (HERMES_HOME single-var pattern)
```

The hermes architecture is 80% of the way there. The remaining 20%: async memory sync, per-tool timeouts, Postgres backend, structured observability, and skills quality control.
