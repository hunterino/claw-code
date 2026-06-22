# Claw Code: CLI Interactions & Remote Control Proposal

## Part 1 — CLI Interaction Catalog

Every interaction between a human (or external system) and claw falls into one of the categories below. Each entry notes the **current interface** (stdin/stdout/files) and whether the interaction is **blocking** (waits for human input) or **fire-and-forget**.

### 1.1 Session Lifecycle

| Interaction | CLI Surface | Blocking? | Data Shape |
|---|---|---|---|
| **Start REPL** | `claw` (no args) | Yes (readline loop) | — |
| **One-shot prompt** | `claw prompt "text"` / `claw "text"` | No (exits on completion) | stdout text or JSON |
| **Resume session** | `claw --resume <ref> [commands]` | Depends on commands | JSONL session file |
| **List sessions** | `/session list` or `SessionStore::list_sessions()` | No | `Vec<ManagedSessionSummary>` |
| **Switch session** | `/session switch <id>` | No | Loads from `.claw/sessions/` |
| **Fork session** | `/session fork [name]` | No | New JSONL file, parent ref |
| **Delete session** | `/session delete <id>` | No (--force skips confirm) | Removes file |
| **Export session** | `claw export <ref> [path]` or `/export` | No | JSONL/JSON to file or stdout |
| **Compact session** | `/compact` or auto at token threshold | No | Rewrites session messages |

### 1.2 Conversation Turn

| Interaction | CLI Surface | Blocking? | Data Shape |
|---|---|---|---|
| **Submit prompt** | Readline input or `claw prompt` | No (after submission) | `String` → `TurnSummary` |
| **Stream response** | stdout (Markdown+ANSI or JSON) | — (push) | `AssistantEvent` stream |
| **Tool execution** | Automatic within turn loop | No | `ToolUse` → `ToolResult` |
| **Permission prompt** | stdin Y/N when mode != `danger-full-access` | **Yes — blocks turn** | `PermissionOutcome` |
| **Hook execution** | Shell scripts via `HookRunner` | No (timeout-bounded) | `HookRunResult` |
| **Auto-compaction** | Triggered when input tokens > threshold | No | `CompactionResult` |

### 1.3 Diagnostic / Read-Only Commands

| Command | CLI Surface | Output Formats | Notes |
|---|---|---|---|
| `claw status` | Subcommand | text, json | Model provenance, session info, MCP count |
| `claw doctor` | Subcommand | text, json | Auth health, config validation, env checks |
| `claw sandbox` | Subcommand | text, json | Filesystem isolation status |
| `claw state` | Subcommand | text, json | Reads `.claw/worker-state.json` |
| `claw config [section]` | Subcommand | text, json | Merged config from all sources |
| `claw diff` | Subcommand | text, json | Git diff of workspace |
| `claw version` | Subcommand | text, json | Binary version |
| `claw agents` | Subcommand | text, json | Registered agents |
| `claw mcp` | Subcommand | text, json | MCP server inventory |
| `claw skills` | Subcommand | text, json | Installed skills |
| `claw system-prompt` | Subcommand | text, json | Assembled system prompt |
| `claw dump-manifests` | Subcommand | text, json | Tool manifest extraction |
| `claw bootstrap-plan` | Subcommand | text, json | Planning output |

### 1.4 REPL Slash Commands (Interactive Only)

These are only available inside the readline loop. Key ones that modify state:

| Command | Effect | Stateful? |
|---|---|---|
| `/model <name>` | Switch model mid-session | Yes |
| `/permissions [mode]` | Change permission mode | Yes |
| `/clear --confirm` | Reset session | Yes |
| `/plugin install\|enable\|disable` | Plugin lifecycle | Yes |
| `/ultraplan <task>` | Extended reasoning turn | Yes (appends to session) |
| `/teleport <sym>` | Jump to file/symbol | Read-only |
| `/bughunter [scope]` | Bug scanning prompt | Yes (appends) |
| `/commit`, `/pr`, `/issue` | Git operations | Yes (side-effects) |
| `/subagent steer\|kill` | Agent control | Yes |
| `/review`, `/security-review` | Code analysis prompts | Yes (appends) |
| `/cost` | Token usage | Read-only |

### 1.5 File-Based Observability (Already Exists)

| File | Written By | Read By | Content |
|---|---|---|---|
| `.claw/worker-state.json` | Runtime (atomic writes) | `claw state`, external orchestrators | Worker status, current task |
| `.claw/sessions/<hash>/<id>.jsonl` | Session persistence | `claw --resume`, `/session` | Full conversation history |
| `.claw.json` / `.claw/settings.json` | User/tooling | Config loader | Model, permissions, MCP, hooks, plugins |

### 1.6 Existing MCP Server Mode

`run_mcp_serve()` at `main.rs:2036` already exposes claw's built-in tools (bash, read, write, grep, glob, web-search, web-fetch) as an MCP server over stdio JSON-RPC 2.0. This is tool-level access only — it does **not** expose session management or conversation turns.

---

## Part 2 — Interaction Classification for Remote Control

Every interaction above falls into one of these remote-control patterns:

| Pattern | Example | Challenge |
|---|---|---|
| **Request/Response** | Submit prompt → get turn summary | Easy — REST or MQTT req/res |
| **Streaming Push** | Token-by-token response | Needs SSE, WebSocket, or MQTT topic |
| **Blocking Prompt** | Permission approval | Needs async callback — the hardest one |
| **Fire-and-Forget** | Switch model, compact session | Easy — any protocol |
| **File Observation** | Poll worker-state.json | Already works; MQTT publish would be better |
| **Lifecycle** | Start/stop session, plugin management | REST CRUD or MQTT command topics |

---

## Part 3 — Proposed Remote Control Architecture

### 3.1 Design Principles

1. **Don't replace the CLI** — add a parallel control plane alongside it
2. **Reuse existing traits** — `ApiClient`, `ToolExecutor`, `PermissionPrompter` are already trait objects
3. **Session-first** — every remote interaction is scoped to a session ID
4. **MQTT for events, HTTP for commands** — MQTT excels at streaming/pub-sub; HTTP excels at request/response
5. **Match the FiveX platform** — use the same Mosquitto broker and goAuth JWT patterns from the parent monorepo

### 3.2 Protocol Choice: Both

| Concern | HTTP API | MQTT |
|---|---|---|
| Submit prompt | `POST /api/v1/sessions/{id}/turns` | `claw/{workspace}/command/turn` |
| Stream tokens | SSE on the POST response | `claw/{workspace}/events/stream/{session_id}` |
| Permission prompt | Returns `202 Accepted` + polls or WebSocket callback | `claw/{workspace}/events/permission_request/{session_id}` → reply on `claw/{workspace}/command/permission_response/{session_id}` |
| Session CRUD | REST (`GET/POST/DELETE /api/v1/sessions`) | `claw/{workspace}/command/session/{create\|delete\|list}` |
| Diagnostics | `GET /api/v1/status`, `/doctor`, `/config` | `claw/{workspace}/events/status` (periodic publish) |
| Worker state | `GET /api/v1/state` | `claw/{workspace}/events/worker-state` (on change) |
| Tool results | Included in turn response | `claw/{workspace}/events/tool/{session_id}` |
| Hook notifications | Included in turn events | `claw/{workspace}/events/hooks/{session_id}` |

### 3.3 HTTP API Design

```
POST   /api/v1/sessions                          # Create session
GET    /api/v1/sessions                          # List sessions
GET    /api/v1/sessions/{id}                     # Get session (messages, metadata)
DELETE /api/v1/sessions/{id}                     # Delete session
POST   /api/v1/sessions/{id}/fork                # Fork session
POST   /api/v1/sessions/{id}/compact             # Trigger compaction
POST   /api/v1/sessions/{id}/export              # Export session

POST   /api/v1/sessions/{id}/turns               # Submit a prompt turn
  Request:  { "prompt": "...", "model": "opus", "permission_mode": "danger-full-access" }
  Response: SSE stream of AssistantEvent, then final TurnSummary JSON

POST   /api/v1/sessions/{id}/permissions/{req_id} # Respond to permission prompt
  Request:  { "decision": "allow" | "deny" }

POST   /api/v1/sessions/{id}/slash               # Execute slash command
  Request:  { "command": "/commit", "args": "-m 'fix'" }

GET    /api/v1/status                             # claw status
GET    /api/v1/doctor                             # claw doctor
GET    /api/v1/config?section=hooks               # claw config
GET    /api/v1/state                              # Worker state
GET    /api/v1/models                             # Available models + aliases
GET    /api/v1/tools                              # Tool manifest
GET    /api/v1/mcp                                # MCP server inventory
GET    /api/v1/plugins                            # Plugin inventory
POST   /api/v1/plugins/{id}/enable                # Enable plugin
POST   /api/v1/plugins/{id}/disable               # Disable plugin
```

### 3.4 MQTT Topic Design

Follows FiveX conventions: `{service}/{workspace_hash}/{direction}/{entity}/{qualifier}`.

```
# Commands (external → claw)
claw/{workspace}/command/turn                    # { session_id, prompt, model?, permission_mode? }
claw/{workspace}/command/permission_response     # { session_id, request_id, decision }
claw/{workspace}/command/session/create          # { model?, workspace_root? }
claw/{workspace}/command/session/delete          # { session_id }
claw/{workspace}/command/session/fork            # { session_id, branch_name? }
claw/{workspace}/command/session/compact         # { session_id }
claw/{workspace}/command/slash                   # { session_id, command, args? }
claw/{workspace}/command/model                   # { session_id, model }
claw/{workspace}/command/cancel                  # { session_id }  — abort in-flight turn

# Events (claw → external)
claw/{workspace}/events/stream/{session_id}      # AssistantEvent (text_delta, tool_use, stop)
claw/{workspace}/events/tool/{session_id}        # { tool_name, input, output, is_error }
claw/{workspace}/events/permission/{session_id}  # { request_id, tool_name, input, mode }
claw/{workspace}/events/hook/{session_id}        # HookProgressEvent
claw/{workspace}/events/compaction/{session_id}  # CompactionResult
claw/{workspace}/events/worker-state             # WorkerStatus (on every state transition)
claw/{workspace}/events/session/created          # { session_id }
claw/{workspace}/events/session/deleted          # { session_id }
claw/{workspace}/events/error/{session_id}       # RuntimeError / ToolError

# Discovery
claw/{workspace}/discovery/announce              # Periodic heartbeat with capabilities
claw/{workspace}/discovery/health                # Health check response
```

### 3.5 Implementation Strategy

#### Phase 1: Non-blocking JSON API (minimal changes)

**What**: Add `claw serve` subcommand that starts an HTTP server (tiny — axum or warp) on a configurable port (default 4546). The server wraps the existing `LiveCli` / `ConversationRuntime` behind HTTP handlers.

**Why first**: Every diagnostic command already has `--output-format json`. The `ConversationRuntime::run_turn()` method already returns a `TurnSummary`. The hardest part — permission prompting — can initially be bypassed by requiring `permission_mode: "danger-full-access"` for API sessions.

**Key changes**:
- New `CliAction::Serve { port, permission_mode }` variant
- Axum router in `main.rs` wrapping existing functions
- `Session` gets an `Arc<Mutex<>>` wrapper for concurrent access
- Worker state transitions publish to an in-process event bus

**Files touched**: `main.rs` (new action + route handlers), `Cargo.toml` (add axum + tokio)

#### Phase 2: Streaming & Permission Callbacks

**What**: SSE streaming on the turn endpoint. Permission prompts return `202 Accepted` with a `request_id`; the caller POSTs the decision back.

**Key changes**:
- New `RemotePermissionPrompter` implementing `PermissionPrompter` trait — instead of blocking on stdin, it publishes the permission request (HTTP: SSE event; MQTT: topic) and blocks on an `mpsc::Receiver` until the decision arrives via the callback endpoint
- `AssistantEvent` stream piped into SSE or chunked response

#### Phase 3: MQTT Integration

**What**: Add MQTT client (reuse `mqtt5_client` patterns from `fivex_ui` or `mod_dart`) alongside the HTTP server. Subscribe to `claw/{workspace}/command/#`, publish to `claw/{workspace}/events/#`.

**Key changes**:
- New `claw::mqtt` module with goAuth JWT authentication
- MQTT command handler dispatches to the same `ConversationRuntime` methods as HTTP
- Worker state transitions and AssistantEvents dual-publish to both HTTP SSE and MQTT topics
- Configuration in `.claw/settings.json`:

```json
{
  "serve": {
    "http": { "port": 4546, "bind": "127.0.0.1" },
    "mqtt": {
      "broker": "localhost:1883",
      "auth": "jwt",
      "workspace_topic_prefix": "claw/a1b2c3d4"
    }
  }
}
```

#### Phase 4: Multi-Session & Orchestration

**What**: Support multiple concurrent sessions behind the same `claw serve` instance. This enables orchestration patterns like: fivex_ui opens 3 sessions (planning, coding, reviewing) and steers them via MQTT.

**Key changes**:
- `SessionManager` holding `HashMap<String, Arc<Mutex<ConversationRuntime>>>`
- Session creation/destruction over API
- Subagent dispatch wired to session manager

### 3.6 Existing Seams to Exploit

The codebase already has these integration-friendly patterns that make remote control feasible without deep refactoring:

| Existing Pattern | Location | How to Exploit |
|---|---|---|
| `ApiClient` trait | `conversation.rs:53` | Runtime doesn't care who calls it |
| `ToolExecutor` trait | `conversation.rs:58` | Tools are already dispatch-by-name |
| `PermissionPrompter` trait | `permissions.rs` | Swap CLI prompter for remote prompter |
| `HookProgressReporter` trait | `hooks.rs:58` | Route hook events to MQTT/SSE |
| `--output-format json` on every command | `main.rs` passim | JSON output already exists for all diagnostics |
| `TurnSummary` return type | `conversation.rs:110` | Structured result, not printf |
| Worker state file | `main.rs:1999` | Already designed for external polling |
| MCP serve mode | `main.rs:2036` | Proves the binary can host a server |
| Session JSONL persistence | `session.rs` | Sessions already serializable |
| `SessionStore` | `session_control.rs:20` | Full CRUD on sessions already exists |

### 3.7 Security Considerations

1. **Auth**: HTTP API should require API key or JWT (reuse goAuth pattern). MQTT already has goAuth JWT.
2. **Bind address**: Default to `127.0.0.1` (localhost only). Expose on `0.0.0.0` only with explicit config.
3. **Permission mode**: API sessions should default to the most restrictive mode. `danger-full-access` requires explicit opt-in per session.
4. **Session isolation**: Each session is workspace-scoped. The API should not allow cross-workspace session access.
5. **Rate limiting**: Turn submissions should be rate-limited to prevent runaway token spend.
6. **Audit trail**: Every API/MQTT command should be logged to the session JSONL with a `source: "api"` or `source: "mqtt"` tag.

---

## Part 4 — Quick Wins (No Server Required)

Even without `claw serve`, external systems can drive claw today:

### 4.1 One-Shot Scripting (Works Now)

```bash
# Submit a prompt, get JSON back
claw prompt "List all TODO items in src/" --output-format json --permission-mode danger-full-access

# Resume a session and run a slash command
claw --resume latest /diff --output-format json

# Chain commands
SESSION=$(claw prompt "Create a test for auth.rs" --output-format json | jq -r .session_id)
claw --resume "$SESSION" "Now run the test" --output-format json
```

### 4.2 File-Based Orchestration (Works Now)

```bash
# Poll worker state
watch -n 2 'cat .claw/worker-state.json | jq .'

# List sessions programmatically
claw session list --output-format json | jq '.[] | {id, updated_at, message_count}'
```

### 4.3 MCP-Based Tool Access (Works Now)

```bash
# Use claw as an MCP server for another AI tool
echo '{"jsonrpc":"2.0","method":"tools/list","id":1}' | claw acp serve
```

---

## Part 5 — Interaction Sequence Diagrams

### 5.1 HTTP Turn with Permission Prompt

```
Client                          claw serve                     Claude API
  │                                │                               │
  │  POST /sessions/{id}/turns     │                               │
  │  { prompt: "delete old logs" } │                               │
  │ ──────────────────────────────>│                               │
  │                                │  stream(ApiRequest)           │
  │                                │ ─────────────────────────────>│
  │                                │  AssistantEvent::ToolUse      │
  │                                │  (bash: "rm -rf /var/log/*")  │
  │                                │ <─────────────────────────────│
  │  SSE: tool_use { bash, rm.. }  │                               │
  │ <──────────────────────────────│                               │
  │                                │  PermissionPrompter::prompt() │
  │  SSE: permission_request       │  (blocks on channel)          │
  │  { req_id: "p1", tool: bash }  │                               │
  │ <──────────────────────────────│                               │
  │                                │                               │
  │  POST /sessions/{id}/perms/p1  │                               │
  │  { decision: "deny" }          │                               │
  │ ──────────────────────────────>│                               │
  │                                │  channel.send(Deny)           │
  │                                │  → ToolResult { is_error }    │
  │  SSE: tool_result { denied }   │                               │
  │ <──────────────────────────────│                               │
  │  SSE: turn_summary { ... }     │                               │
  │ <──────────────────────────────│                               │
```

### 5.2 MQTT Orchestration from fivex_ui

```
fivex_ui (Flutter)              Mosquitto                      claw serve
  │                                │                               │
  │  PUB claw/ws/command/turn      │                               │
  │  { session_id, prompt }        │                               │
  │ ──────────────────────────────>│  SUB claw/ws/command/#        │
  │                                │ ─────────────────────────────>│
  │                                │                               │  run_turn()
  │  SUB claw/ws/events/stream/s1  │  PUB events/stream/s1        │
  │ <──────────────────────────────│ <─────────────────────────────│
  │  (token deltas arrive)         │                               │
  │                                │  PUB events/tool/s1           │
  │ <──────────────────────────────│ <─────────────────────────────│
  │  (tool execution visible)      │                               │
  │                                │  PUB events/stream/s1 [stop]  │
  │ <──────────────────────────────│ <─────────────────────────────│
  │  (turn complete)               │                               │
```

---

## Summary

Claw Code's CLI interactions decompose cleanly into **session lifecycle**, **conversation turns** (with streaming and permission blocking), **diagnostics**, and **slash commands**. The existing codebase is unusually well-prepared for remote control because:

1. Every command already has `--output-format json`
2. Core runtime uses traits (`ApiClient`, `ToolExecutor`, `PermissionPrompter`) not concrete types
3. Worker state is already a file-based observability surface designed for external polling
4. An MCP server mode already exists, proving the binary can host a protocol server
5. Session persistence is structured JSONL with full CRUD via `SessionStore`

The recommended approach: **HTTP API first** (Phase 1-2) for immediate scriptability and `fivex_ui` integration, then **MQTT overlay** (Phase 3) for real-time event streaming that matches the FiveX platform's Mosquitto/goAuth infrastructure.
