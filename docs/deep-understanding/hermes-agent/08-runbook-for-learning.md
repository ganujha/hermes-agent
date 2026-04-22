# 08 — Runbook for Learning: hermes-agent

## Environment Setup

```bash
cd /home/user/hermes-agent

# Activate the virtual environment
source venv/bin/activate   # if venv exists
# OR install fresh:
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"

# Verify install
python -c "from run_agent import AIAgent; print('OK')"
hermes --version   # or: python hermes_cli/main.py --version
```

---

## Exact Commands to Run

### 1. Start Interactive CLI (Core Agent Loop)
```bash
hermes
# OR from repo root:
python hermes_cli/main.py
```
Observes: System prompt construction, tool call display, KawaiiSpinner animation, response streaming.

### 2. Inspect the SQLite Session Database
```bash
# After running at least one conversation:
sqlite3 ~/.hermes/state.db

# In sqlite3:
.schema
SELECT id, source, model, started_at, message_count FROM sessions ORDER BY started_at DESC LIMIT 5;
SELECT session_id, role, content FROM messages WHERE session_id = '<id from above>' LIMIT 10;

# Full-text search across all sessions:
SELECT m.session_id, m.role, m.content 
FROM messages_fts 
JOIN messages m ON m.id = messages_fts.rowid 
WHERE messages_fts MATCH 'python';
.quit
```

### 3. Run the Test Suite
```bash
# Use the wrapper script (CI-parity mode):
./scripts/run_tests.sh

# Run a specific test file:
./scripts/run_tests.sh tests/agent/test_context_compressor.py

# Verbose output:
./scripts/run_tests.sh -v --tb=long
```
Observes: Which tests cover the agent loop, compression, memory, and tools. Look for what's NOT tested (gaps).

### 4. Trace Tool Discovery
```bash
python3 -c "
from tools.registry import discover_builtin_tools, registry
mods = discover_builtin_tools()
print(f'Discovered {len(mods)} tool modules:')
for m in mods:
    print(f'  {m}')
print(f'Registered tools: {len(registry._tools)}')
for name, entry in list(registry._tools.items())[:5]:
    print(f'  {name} (toolset={entry.toolset})')
"
```
Observes: How many tools self-register, which modules register them.

### 5. Inspect the Config System
```bash
# See the full default config:
python3 -c "
from hermes_cli.config import DEFAULT_CONFIG
import json
print(json.dumps(DEFAULT_CONFIG, indent=2, default=str))
" | head -80

# See what config file would be used:
python3 -c "
from hermes_constants import get_hermes_home
print(get_hermes_home() / 'config.yaml')
"
```

### 6. Observe Context Compression
```bash
# Set compression to trigger early for observation:
hermes config set context.threshold_percent 0.1

# Start a conversation and do many tool calls until compression fires:
hermes
# In chat: "search the web for 10 recent AI papers and summarize each one"
# Watch for: [CONTEXT COMPACTION] in the conversation
```
Observes: When compression fires, what the summary looks like, how the conversation continues.

### 7. Create and Test a Skill
```bash
# In a conversation:
hermes
# In chat: "Create a skill called 'repo-audit' that shows how to analyze a git repository"

# Then invoke it:
/repo-audit

# Inspect the created skill file:
ls ~/.hermes/skills/
cat ~/.hermes/skills/repo-audit.md   # or wherever it was saved
```

### 8. Inspect Memory Files
```bash
# See what the agent has remembered:
cat ~/.hermes/MEMORY.md
cat ~/.hermes/USER.md

# Ask the agent to remember something:
hermes
# In chat: "Please remember that I prefer Python 3.12 and use pytest for testing"
# Exit and re-open hermes -- the memory should persist
```

### 9. Test the Cron Scheduler
```bash
# Create a test cron job:
hermes
# In chat: "Create a cron job to run in 2 minutes that writes 'hello from cron' to a test file"

# Check the jobs:
cat ~/.hermes/cron/jobs.json

# Wait 2 minutes, then check output:
ls ~/.hermes/cron/output/
```

### 10. Gateway Mode (if you have a Telegram bot token)
```bash
# Setup (interactive):
hermes gateway setup

# Start the gateway:
hermes gateway start

# Observe logs:
tail -f ~/.hermes/logs/gateway.log   # if logging to file
```

### 11. Run a Batch Job
```bash
# Run multiple agent tasks in parallel:
python batch_runner.py \
  --tasks "What is 2+2?" "What is the capital of France?" \
  --model "anthropic/claude-haiku-4-5" \
  --output-dir /tmp/hermes_batch

# Inspect output:
ls /tmp/hermes_batch/
```

### 12. Test Profile Isolation
```bash
# Create two profiles:
hermes -p work config set model.provider "openai"
hermes -p personal config set model.provider "anthropic"

# Verify isolation:
hermes -p work config get model.provider
hermes -p personal config get model.provider
ls ~/.hermes/profiles/
```

---

## Where Logs Appear

| Log Type | Location | Notes |
|----------|----------|-------|
| Python logging | stderr (default) | Configured via `hermes_logging.py` |
| Tool execution traces | `agent/display.py:KawaiiSpinner` | Terminal stdout |
| Session data | `~/.hermes/state.db` | SQLite (inspect with sqlite3) |
| Cron output | `~/.hermes/cron/output/{job_id}/{timestamp}.md` | Markdown files |
| Gateway logs | stderr / platform-specific | Depends on config |
| Compression events | Logged as `INFO` in Python logger | Search for "compress" in logs |

---

## Where State Changes Can Be Seen

| State Change | How to Observe |
|-------------|---------------|
| New conversation session | `SELECT * FROM sessions ORDER BY started_at DESC LIMIT 1;` in sqlite3 |
| Memory update | `cat ~/.hermes/MEMORY.md` before and after a memory write |
| Skill creation | `ls ~/.hermes/skills/` |
| Config change | `cat ~/.hermes/config.yaml` |
| Cron job added | `cat ~/.hermes/cron/jobs.json` |
| Compression fired | Look for `parent_session_id` in sessions table |
| Token usage | `SELECT input_tokens, output_tokens, cache_read_tokens FROM sessions ORDER BY started_at DESC LIMIT 1;` |

---

## How to Trigger Important Workflows

| Workflow | Method |
|---------|--------|
| Context compression | Set `context.threshold_percent: 0.1` in config, run a long conversation |
| Memory write | In chat: "Please remember that [fact]" |
| Skill creation | In chat: "Create a skill for [task]" or use `skill_manage` tool |
| Delegation | In chat: "Use delegate_task to run [subtask] in parallel" |
| Session search | In chat: "Search my previous conversations for [topic]" |
| Cron job | In chat: "Schedule a daily summary at 9am" |
| Model switch | `/model` slash command |
| Tool approval | Use `terminal` tool with a sudo command |
| Interrupt | `Ctrl+C` during tool execution |

---

## How to Inspect Memory / Tool Use / Agent Behavior

### Inspect tool calls in a session:
```bash
sqlite3 ~/.hermes/state.db "
SELECT m.role, m.tool_name, substr(m.content, 1, 200)
FROM messages m
WHERE m.session_id = (SELECT id FROM sessions ORDER BY started_at DESC LIMIT 1)
ORDER BY m.timestamp;
"
```

### See what tools were available in a session:
```bash
python3 -c "
from model_tools import get_tool_definitions
tools = get_tool_definitions()
print(f'{len(tools)} tools available:')
for t in tools[:10]:
    print(f'  {t[\"name\"]}')
"
```

### Trace the system prompt:
Add a debug print in `agent/prompt_builder.py` or use Python's logging at DEBUG level:
```bash
PYTHONLOGLEVEL=DEBUG hermes 2>&1 | grep -A5 "system prompt"
```

### Watch compression decisions:
```bash
python3 -c "
from agent.context_compressor import ContextCompressor
from agent.auxiliary_client import build_auxiliary_client
# Create a compressor and test should_compress():
comp = ContextCompressor(context_length=200000)
comp.last_prompt_tokens = 160000  # 80% full
print('Should compress:', comp.should_compress())
"
```

---

## How to Safely Experiment Without Breaking the Repo

1. **Use profiles**: Create a `test` profile (`hermes -p test`) — all state is isolated.
2. **Use temp HERMES_HOME**: `HERMES_HOME=/tmp/hermes_test hermes` — completely isolated, delete when done.
3. **Mock API calls in tests**: Use the test fixtures in `tests/conftest.py` — `_isolate_hermes_home` autouse fixture redirects all state to a temp dir.
4. **Run tests, not live agent**: For tool behavior, write a test in `tests/` rather than running the live agent.
5. **Dry-run cron jobs**: Use `/insights` or inspect `jobs.json` before any job fires.
6. **Backup state.db**: `cp ~/.hermes/state.db ~/.hermes/state.db.backup` before experiments.
7. **Don't edit built-in skills**: Place experimental skills in `~/.hermes/skills/` not in the repo's `skills/` directory.
