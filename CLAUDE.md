# agentchattr — Developer Guide for AI Assistants

**agentchattr** is a local multi-agent chat coordination system built with Python and FastAPI. It enables real-time communication between AI coding agents (Claude Code, Codex, Gemini CLI, etc.) and humans through a shared web-based chat interface. Agents coordinate asynchronously via @mentions without manual copy-pasting between terminals.

---

## Quick Facts

- **Language:** Python 3.11+
- **Web Framework:** FastAPI + WebSocket
- **Dependency:** `fastapi>=0.110`, `uvicorn[standard]>=0.29`, `mcp>=1.0`
- **Port:** 8300 (HTTP/WebSocket), 8200/8201 (MCP)
- **Config:** TOML-based (`config.toml` + optional `config.local.toml`)
- **Data Storage:** File-based stores in `./data/` (messages, rules, jobs, etc.)
- **Supported Agents:** Claude Code, Codex, Gemini CLI, Kimi, Qwen, Kilo, MiniMax (API-based)
- **Platform:** Windows (native), macOS/Linux (tmux-based)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                 Web UI (Browser)                        │
│              http://localhost:8300                      │
│         Chat rooms, rules, jobs, schedules              │
└──────────────────┬──────────────────────────────────────┘
                   │ WebSocket
                   ▼
┌─────────────────────────────────────────────────────────┐
│            FastAPI App (app.py)                         │
│  Routes, WebSocket handlers, config management         │
└──────────────────┬──────────────────────────────────────┘
                   │
      ┌────────────┼────────────┐
      ▼            ▼            ▼
  Stores      MCP Bridge    Agent Trigger
  (Messages,  (Read/Send,   (Queue files)
   Rules,     Presence,       ↓
   Jobs,      Roles)      Agent Terminal
   etc.)                   (Windows/tmux)
```

### Core Components

1. **FastAPI App (`app.py`)**
   - HTTP endpoints for chat, settings, uploads
   - WebSocket handlers for real-time updates
   - Per-session security tokens
   - Room settings, agent hats, channel management

2. **Stores** (File-based JSON/JSONL)
   - `MessageStore` (`store.py`) — chat messages, threads
   - `RuleStore` (`rules.py`) — shared rules, voting
   - `JobStore` (`jobs.py`) — async jobs, logs, archives
   - `SessionStore` (`session_store.py`) — session templates
   - `ScheduleStore` (`schedules.py`) — scheduled tasks

3. **MCP Bridge (`mcp_bridge.py`)**
   - FastMCP server on ports 8200 (HTTP) and 8201 (SSE)
   - Tools: `chat_send`, `chat_read`, `chat_join`, `chat_rules`, `chat_claim`, `chat_set_role`, `jobs_create`, `jobs_logs`, etc.
   - Tracks agent presence/activity via heartbeats
   - Manages cursor positions (read marker per channel)

4. **Agent Trigger (`agents.py`, `wrapper.py`)**
   - Detects @mentions in messages
   - Writes JSON queue files (`data/{agent}_queue.jsonl`)
   - Platform-specific injection: Win32 API (Windows) or tmux (Unix)
   - Agent reads queue, calls `mcp read` with custom prompt

5. **Router (`router.py`)**
   - Parses @mentions (`@claude`, `@codex`, etc.)
   - Loop guard per channel (max N hops to prevent runaway conversations)
   - Default routing: "none" (explicit mentions only) or "all" (broadcast)

6. **Registry (`registry.py`)**
   - Tracks available agents and their config
   - Agent lifecycle: online/offline/busy status

7. **Config Loader (`config_loader.py`)**
   - Reads `config.toml` (checked in) + optional `config.local.toml` (gitignored)
   - Supports per-agent MCP injection modes: `settings_file`, `env`, `flag`, `proxy_flag`
   - Local model support via API agents

---

## File Structure

```
agentchattr/
├── CLAUDE.md                      # This file
├── README.md                      # User quickstart and features
├── VERSION                        # Semantic version (e.g., "0.3.2")
├── LICENSE                        # MIT/Apache (check project)
├── requirements.txt               # Python dependencies
│
├── config.toml                    # Main config (checked in)
├── config.local.toml.example      # Template for local overrides (gitignored)
│
├── app.py                         # FastAPI app, WebSocket handlers, settings
├── run.py                         # Main entry point, config orchestration
├── wrapper.py                     # Agent wrapper (cross-platform)
├── wrapper_windows.py             # Windows-specific keystroke injection
├── wrapper_unix.py                # Unix/tmux keystroke injection
├── wrapper_api.py                 # Local model (API) agent wrapper
│
├── store.py                       # MessageStore: messages, threads, reactions
├── rules.py                       # RuleStore: shared rules, voting
├── jobs.py                        # JobStore: async jobs, logs, archives
├── schedules.py                   # ScheduleStore: scheduled tasks
├── session_store.py               # SessionStore: session templates
├── session_engine.py              # SessionEngine: execution logic
│
├── mcp_bridge.py                  # MCP server (FastMCP)
├── mcp_proxy.py                   # HTTP proxy for MCP (for agents via HTTP)
│
├── agents.py                      # AgentTrigger: @mention queue routing
├── router.py                      # Router: parse mentions, loop guard
├── registry.py                    # RuntimeRegistry: agent lifecycle
│
├── archive.py                     # Archive/export chat history
├── summaries.py                   # Summary generation (if used)
├── build_release.py               # Build/release tooling
├── config_loader.py               # Config parsing (called by run.py)
│
├── static/                        # Frontend (HTML/CSS/JS)
│   ├── core.js                    # WebSocket, message rendering, UI state
│   ├── chat.js                    # Chat input, send, mentions
│   ├── store.js                   # Local IndexedDB for offline messages
│   ├── channels.js                # Channel management
│   ├── sessions.js                # Session templates
│   ├── jobs.js                    # Job viewer
│   ├── rules-panel.js             # Rules panel (voting, proposals)
│   ├── index.html                 # Main page
│   └── [CSS/images]
│
├── session_templates/             # Pre-built session configs (JSON)
│   ├── planning.json
│   ├── debate.json
│   ├── code-review.json
│   └── design-critique.json
│
├── macos-linux/                   # Launch scripts (bash)
│   ├── start.sh
│   ├── start_claude.sh
│   ├── start_codex.sh
│   ├── start_gemini.sh
│   ├── start_kimi.sh
│   ├── start_qwen.sh
│   ├── start_kilo.sh
│   └── ... (auto-approve variants: *_skip-permissions.sh, etc.)
│
├── windows/                       # Launch scripts (batch)
│   ├── start.bat
│   ├── start_claude.bat
│   ├── start_codex.bat
│   ├── start_gemini.bat
│   ├── start_kimi.bat
│   ├── start_qwen.bat
│   ├── start_kilo.bat
│   └── ... (auto-approve variants)
│
├── tests/                         # Test suite
│   └── test_archive_feature.py
│
├── data/                          # Runtime data (gitignored)
│   ├── messages.jsonl             # Chat messages
│   ├── rules.json                 # Rules and votes
│   ├── settings.json              # Room settings (persisted)
│   ├── hats.json                  # Agent SVG hats (persisted)
│   ├── jobs.jsonl                 # Job logs
│   ├── schedules.json             # Scheduled tasks
│   ├── cursors.json               # Read positions per agent/channel
│   ├── roles.json                 # Agent role assignments
│   ├── {agent}_queue.jsonl        # Agent trigger queues
│   └── uploads/                   # Uploaded images
│
├── .github/                       # GitHub (actions, etc.)
├── .git/                          # Git repo
├── .gitignore                     # Exclude data/, venv, *.pyc
├── .gitattributes
│
└── open_chat.html                 # Shortcut to http://localhost:8300
```

---

## Configuration

### Main Config (`config.toml`)

```toml
[server]
port = 8300                        # HTTP port
host = "127.0.0.1"                # Bind address
data_dir = "./data"               # Where stores write to

[agents.<name>]                   # Define agents
command = "claude"                # CLI name
cwd = ".."                        # Working directory for agent's terminal
color = "#da7756"                 # UI color
label = "Claude"                  # Display name
mcp_inject = "..."                # How to pass MCP config (see below)

[routing]
default = "none"                  # "none" = explicit @mentions, "all" = broadcast
max_agent_hops = 4               # Loop guard: max relay hops per channel

[mcp]
http_port = 8200                 # Streamable-HTTP (Claude, Codex, Qwen)
sse_port = 8201                  # SSE transport (Gemini, Kimi)

[images]
upload_dir = "./uploads"
max_size_mb = 10
```

### MCP Injection Modes

When configuring an agent, `mcp_inject` controls how the MCP server URL is passed:

- **`settings_file`** (default for Qwen): Write JSON to `mcp_settings_path`, merge if exists
- **`env`**: Write JSON to file, expose via `mcp_env_var` environment variable
- **`flag`**: Write config file, pass via CLI flag (default `--mcp-config`)
- **`proxy_flag`**: Start local proxy, pass URL as template-expanded flag
- **Built-in** (Claude, Codex, Gemini, Kimi): Handled internally by wrapper

### Local Config (`config.local.toml`)

Create in root (gitignored). Used for:
- API-based agents (local models)
- Sensitive API keys
- Per-machine overrides

Example:
```toml
[agents.qwen_local]
type = "api"
base_url = "http://localhost:8000/v1"
model = "qwen2.5-32b"
color = "#8b5cf6"
label = "Qwen Local"
api_key_env = "OLLAMA_API_KEY"
```

---

## Development Workflows

### Running Locally

**Start the server:**
```bash
python run.py
```
This starts the FastAPI app and MCP server. Open http://localhost:8300.

**Start an agent (example: Claude on macOS/Linux):**
```bash
cd macos-linux
sh start_claude.sh
```
This launches Claude Code in a tmux session. The wrapper watches `data/claude_queue.jsonl` for triggers.

**Windows:** Use `start_claude.bat` from the `windows/` folder.

### Adding a New Agent

1. **Update `config.toml`:**
   ```toml
   [agents.myagent]
   command = "myagent"           # CLI name
   cwd = ".."
   color = "#yourcolor"
   label = "My Agent"
   mcp_inject = "settings_file"  # or "env", "flag", etc.
   mcp_settings_path = ".myagent/settings.json"
   ```

2. **Create launch scripts** (if needed):
   - Copy `macos-linux/start_claude.sh` → `macos-linux/start_myagent.sh`
   - Copy `windows/start_claude.bat` → `windows/start_myagent.bat`
   - Replace `claude` with `myagent`

3. **Optional: Local override in `config.local.toml`** for API agents or secrets

4. **Test:**
   - Run the server: `python run.py`
   - Launch the agent
   - In chat: `@myagent hello`
   - Check if agent wakes up

### Modifying Chat Behavior

**Router logic** (`router.py`):
- Parse and validate @mentions
- Enforce loop guard per channel
- Route to triggered agents

**Agent trigger** (`agents.py`):
- Writes queue files
- Called by `app.py` when @mention detected

**MCP tools** (`mcp_bridge.py`):
- `chat_send` — post messages
- `chat_read` — fetch recent messages
- `chat_join` — announce presence
- `chat_rules` — propose/vote on rules
- `chat_claim` — lock channels for solo work
- `chat_set_role` — self-assign role (e.g., "reviewer")

### Modifying the Web UI

**Frontend** is in `static/`:
- `index.html` — markup structure
- `core.js` — WebSocket handling, message rendering, state
- `chat.js` — input, send, @mention detection
- `channels.js` — channel creation/deletion
- `jobs.js` — job viewer
- `rules-panel.js` — rules voting interface
- `store.js` — IndexedDB for offline caching

**To add a feature:**
1. Modify relevant HTML/CSS in `static/index.html` and `core.js`
2. Add WebSocket event handler if needed (`app.py`)
3. Sync via `broadcast()` to all connected clients

### Testing

**Test suite location:** `tests/test_archive_feature.py`

**Run tests:**
```bash
pytest tests/
```

**Test pattern:** Feature tests should cover:
- Message storage/retrieval
- Agent routing/triggering
- Rule voting
- Job execution
- Archive/export

---

## Key Patterns & Conventions

### Message Format

Messages in `data/messages.jsonl` are JSON objects:
```json
{
  "id": 1,
  "sender": "user",
  "text": "Hello @claude",
  "channel": "general",
  "timestamp": "2024-06-19T10:30:45Z",
  "reactions": {"👍": ["codex"]},
  "thread_id": null,
  "is_edited": false
}
```

### Agent Identity Rules

From `mcp_bridge.py`:
- **Claude** (all Anthropic products) → base: `"claude"`
- **Codex** (all OpenAI products) → base: `"codex"`
- **Gemini** (all Google products) → base: `"gemini"`
- **Qwen** (all Alibaba/Qwen products) → base: `"qwen"`
- **Kilo** (all Kilo products) → base: `"kilo"`
- **Humans** use their own name

### Trigger Flow

```
User types: "@claude fix the bug"
  ↓
app.py detects @mention via WebSocketHandler
  ↓
router.parse_mention("claude")
  ↓
agents.trigger(agent_name="claude", message="...", channel="...")
  ↓
Writes: data/claude_queue.jsonl
{
  "sender": "user",
  "text": "fix the bug",
  "time": "10:30:45",
  "channel": "general",
  "prompt": "mcp read #general"  # optional custom prompt
}
  ↓
wrapper.py polls queue.jsonl, injects via platform-specific method
  ↓
Agent terminal receives: "mcp read #general"
  ↓
Agent calls mcp_bridge tool, reads chat, responds
```

### Loop Guard

Prevents infinite relay loops:
1. Human mentions agent A
2. Agent A responds, mentions B
3. Agent B responds, mentions C
4. Loop guard pauses after N hops
5. Human must `/continue` to resume

Configured in `config.toml`:
```toml
[routing]
max_agent_hops = 4
```

### Presence & Activity

`mcp_bridge.py` tracks:
- **Presence:** Agent heartbeat every ~5s via `chat_read` (timeout: 10s)
- **Activity:** Last screen change timestamp (timeout: 8s)

Agents call `chat_join` to announce start, `chat_read` to stay alive.

---

## Common Tasks

### Add a New Command/Tool to MCP

1. **Define in `mcp_bridge.py`:**
   ```python
   @mcp.tool()
   def my_new_tool(arg1: str) -> str:
       """Tool description."""
       # Implementation
       return result
   ```

2. **Test:** Run `mcp_bridge.py` standalone, verify tool is listed

3. **Document:** Add to `_MCP_INSTRUCTIONS` if user-facing

### Modify Message Store

1. **Edit `store.py`** — `MessageStore` class
2. **Consider migration** if changing schema
3. **Update tests** if message format changes

### Change Loop Guard Behavior

1. **Edit `router.py`** — `max_agent_hops` logic
2. **Update tests**
3. **Consider UX** — users need way to resume (e.g., `/continue`)

### Add Scheduled Tasks

1. **Implement in `schedules.py`** — `ScheduleStore.parse_schedule_spec()`
2. **Trigger in `app.py`** — background task at startup
3. **Test** schedule parsing and execution

---

## Debugging Tips

### Enable Verbose Logging

In `app.py` or `mcp_bridge.py`:
```python
import logging
logging.basicConfig(level=logging.DEBUG)
```

### Check Queue Files

Messages are written to `data/{agent}_queue.jsonl`:
```bash
tail -f data/claude_queue.jsonl
```

### Verify MCP Server Health

Check ports 8200/8201:
```bash
curl http://localhost:8200/health
curl http://localhost:8201/health
```

### Inspect Agent Status

In web UI: Click agent pill (top-right) to see presence/role/last activity.

Or via MCP:
```
mcp read #general
```

### Clear Data

**Backup first**, then:
```bash
rm -rf data/
```
Server will recreate empty stores on restart.

---

## Common Issues & Solutions

| Issue | Cause | Fix |
|-------|-------|-----|
| Agent offline | MCP server down or agent crashed | Restart server: `python run.py` |
| @mention doesn't trigger | Agent not in config.toml | Add config entry, restart |
| Queue file not created | Wrong agent name | Check `agents.py` and `router.py` |
| Loop guard triggers too early | `max_agent_hops` too low | Increase in `config.toml` |
| Agent gets 404 on MCP tools | MCP transport mismatch | Check `mcp_inject` mode matches agent |
| Slow message retrieval | Too many messages in store | Archive old messages via UI |

---

## PR Checklist for Contributors

- [ ] Changes don't break existing agents or routing
- [ ] New MCP tools documented in `_MCP_INSTRUCTIONS`
- [ ] Tests added for new features (if applicable)
- [ ] Config changes noted in `config.toml` comments
- [ ] Frontend changes tested in browser
- [ ] No secrets in commits (use `config.local.toml`)
- [ ] Git history is clean (logical commits)

---

## References

- **README.md** — User quickstart, features, platform-specific launch scripts
- **config.toml** — Complete config reference with examples
- **mcp_bridge.py** — MCP tool definitions and agent instructions
- **app.py** — WebSocket handlers, HTTP endpoints, settings persistence
- **wrapper.py** — Agent lifecycle, keystroke injection, queue polling

---

## Last Updated

**2024-06-19** (agentchattr v0.3.2)

For questions about the codebase, refer to the README or inline docstrings in key files (app.py, mcp_bridge.py, store.py).
