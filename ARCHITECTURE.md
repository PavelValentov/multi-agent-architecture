# Multi-Agent Orchestration Architecture

> A comprehensive architectural reference for production-grade multi-agent AI systems, based on reverse-engineering of a real-world implementation.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Core Engine](#2-core-engine)
3. [Tool System](#3-tool-system)
4. [Agent System](#4-agent-system)
5. [Coordinator Mode](#5-coordinator-mode)
6. [Task Management](#6-task-management)
7. [Permission System](#7-permission-system)
8. [State Management](#8-state-management)
9. [MCP Integration](#9-mcp-integration)
10. [Plugin & Hook System](#10-plugin--hook-system)
11. [Compaction Engine](#11-compaction-engine)
12. [Agent Communication](#12-agent-communication)
13. [Prompt Cache Sharing](#13-prompt-cache-sharing)
14. [Bridge / Remote Execution](#14-bridge--remote-execution)
15. [Message Types](#15-message-types)

---

## 1. System Overview

The system is a **generator-based async orchestrator** built around these layers:

```
Entry Points (CLI, Bridge, Daemon, Recovery)
  |
  v
Core Engine (Query Engine + Turn Loop)
  |
  +---> Tool Orchestration (Registry, Concurrency, Streaming)
  |       +---> Agent Spawner (5 execution paths)
  |       +---> 50+ Built-in Tools + MCP Extensions
  |
  +---> Compaction Engine (5 strategies)
  +---> Cost Tracking
  +---> State Store (tasks, agents, MCP, plugins)
  +---> Permission System (7 modes)
  +---> Plugin System (hooks, skills, commands)
  +---> Terminal UI (React-based renderer)
```

**Key design principles:**
- **Streaming-first**: Everything is an `AsyncGenerator` yielding events in real-time
- **In-process**: No CLI spawning, no subprocesses for orchestration, no external queues
- **Cache-optimized**: Subagents inherit parent's prompt cache via frozen parameters
- **Permission-aware**: Triple validation before every tool execution
- **Extensible**: 27 hook events, plugin system, MCP protocol support

---

## 2. Core Engine

### Query Engine

The Query Engine owns the lifecycle and session state for a single conversation. It is the top-level coordinator.

**Responsibilities:**
- Persistent state across turns (messages, file cache, usage, permissions)
- System/user context loading and assembly
- File state cache management (LRU for read deduplication)
- Session storage (transcript recording, file history, attribution)
- Plugin/MCP loading and tool pool assembly
- Orphaned permission handling

**Key method:** `submitMessage(prompt)` — an async generator that:
1. Processes user input (slash commands, skill discovery)
2. Builds system prompt (static prompts + plugin hooks + MCP resources + memory)
3. Delegates to the Turn Loop
4. Yields SDK messages as they arrive
5. Returns terminal state with usage metrics

### Conversation Turn Loop

The heart of the system. An infinite `while(true)` loop that manages each conversation turn:

**Per-turn sequence:**
1. **Compaction** — Apply snip/microcompact/autocompact/contextCollapse/sessionMemory as needed
2. **API Call** — Stream model request with tool definitions
3. **Tool Detection** — Detect `tool_use` blocks via streaming (NOT via `stop_reason`, which is unreliable)
4. **Tool Execution** — Partition and execute tools (concurrent-safe in parallel, others serial)
5. **Message Accumulation** — Push assistant + tool_result messages to state
6. **Loop Decision** — Continue if tool_use blocks found, else check for max_tokens recovery or exit

**Recovery mechanism:** If the API returns `max_tokens`, the system retries up to 3 times, accumulating output as a prefix for the next request.

### Dependency Injection

The query loop receives its dependencies via injection:
- `callModel` — The actual API call function
- `microcompact` — Message compression function
- `autocompact` — History summarization function
- `uuid` — Identity generation

This enables clean testing and production factory patterns.

---

## 3. Tool System

### Tool Interface

Each tool implements a rich interface:

- **Schema & Validation**: Input schema (Zod) + optional JSON Schema for MCP
- **Core Methods**: `call()` for execution, `description()` for dynamic descriptions, `prompt()` for system prompt notices
- **Permissions**: `checkPermissions()` + `validateInput()` for pre-execution checks
- **Rendering**: Full reactive UI rendering for tool use, progress, results, errors
- **Metadata**:
  - `isConcurrencySafe(input)` — Method (not property) returning whether the tool can run in parallel given specific input
  - `isDestructive(input)` — Optional method for irreversible operations
  - `interruptBehavior()` — Method returning `'cancel'` or `'block'`
  - `shouldDefer` — Property; when true, tool requires explicit ToolSearch before invocation
  - `alwaysLoad` — Property; tool is never deferred

### Tool Registry

- **Collect All Base Tools** — Exhaustive list of ~50+ available tools, respecting feature gates
- **Filter by Permission Context** — Removes tools blocked by deny rules
- **Assemble Tool Pool** — Combines built-in + MCP tools, deduplicates by name, sorts for cache stability

### Tool Categories (~50+ tools)

| Category | Tools |
|----------|-------|
| Core I/O | Shell execution, file read/edit/write, pattern search (glob, grep) |
| Web & Network | HTTP fetch, web search, browser automation (feature-gated) |
| Agent & Tasks | Agent spawning, task CRUD (create/get/update/list/stop), task output |
| Planning | Plan mode entry/exit, brief mode, todo management |
| Specialized | Notebook editing, skill invocation, deferred tool search, sleep, cron, configuration |
| Coordinator-Only | Team create/delete, inter-agent messaging, synthetic output |
| MCP Bridge | Resource listing/reading, LSP integration |

### Tool Execution Pipeline

**Two execution patterns:**

1. **Batch Orchestration** — Partitions tool calls into batches. Concurrency-safe tools run in parallel (max 10), non-concurrent tools run serially with exclusive access.

2. **Streaming Executor** — Buffers results until ordered emission (preserving input order). Tracks tool status: `queued -> executing -> completed -> yielded`. Exclusive sibling abort controller for error handling.

---

## 4. Agent System

### Agent Spawner

The Agent Spawner creates subagents with 5 execution paths:

| Path | Trigger | Behavior |
|------|---------|----------|
| **Synchronous** | Default (no flags) | Inline execution, returns immediately |
| **Background** | `run_in_background=true` | Async task with streaming output file |
| **Git Worktree** | `isolation=worktree` | Temporary git branch + worktree, auto-cleanup |
| **Remote** | `isolation=remote` | Cloud environment execution via WebSocket |
| **Teammate** | `name + team_name` | Named agent in shared team, registered in Agent Name Registry |

### Agent Input Schema

```
description: string         // 3-5 word task summary
prompt: string             // Complete task specification
subagent_type?: string     // Specialized agent type
model?: string             // Model override (sonnet/opus/haiku)
run_in_background?: bool   // Async background execution
name?: string              // Named agent for message routing
team_name?: string         // Team context
mode?: string              // Permission mode override
isolation?: string         // worktree or remote
cwd?: string               // Working directory override
```

### Agent Runner

Orchestrates the full query loop for subagents:
1. Initialize MCP servers (parent + agent-specific)
2. Create subagent context (cloned file/replacement state, fresh denial tracking)
3. Run query loop with isolated context
4. Track usage metrics
5. Record sidechain transcript
6. Return all messages + accumulated usage

### Context Cloning

`createSubagentContext()` creates isolated contexts:
- **File State Cache** — Cloned (LRU, size-limited)
- **Content Replacement State** — Cloned (tool result budget sharing)
- **Denial Tracking** — Fresh instance (isolated permission fallback-to-prompting)
- **Rendered System Prompt** — Frozen (cache-critical snapshot)
- **App State Writes** — No-op (isolated state; only task state reaches root store)
- **Abort Controller** — Child of parent (cascading cancellation)

---

## 5. Coordinator Mode

### Architecture

The Coordinator is a special agent role that orchestrates multiple worker agents:

- **Coordinator** receives: Agent Spawner, SendMessage, TaskStop, TaskList tools
- **Workers** receive: Shell, File I/O, Search, Web, Notebook, MCP tools
- Workers cannot see the coordinator's conversation — prompts must be self-contained

### 4-Phase Workflow

1. **Research** — Parallel workers explore the codebase, read docs, analyze tests
2. **Synthesis** — Coordinator reads all findings, forms implementation plan
3. **Implementation** — Workers execute features, tests, documentation in parallel
4. **Verification** — Workers run tests, typechecks, prove code works

### Communication Patterns

- **Task Notifications** — Workers report results via XML `<task-notification>` messages containing task-id, status, summary, result, usage, output-file, and tool-use-id
- **SendMessage** — Coordinator continues an existing worker with new instructions (preserves context)
- **TaskStop** — Coordinator kills unneeded workers
- **Scratchpad** — Optional shared directory for cross-worker knowledge

### Key Principles

- High context overlap -> continue existing worker (SendMessage)
- Low context overlap -> spawn fresh worker
- Real verification proves code works (runs tests), not rubber-stamping
- Workers are internal signals — coordinator never "thanks" them

---

## 6. Task Management

### Task Types (7)

| Type | ID Prefix | Description |
|------|-----------|-------------|
| `local_bash` | `b` | Shell command execution |
| `local_agent` | `a` | Local AI agent execution |
| `remote_agent` | `r` | Remote cloud agent execution |
| `in_process_teammate` | `t` | Inline agent execution in shared team |
| `local_workflow` | `w` | Workflow automation (feature-gated) |
| `monitor_mcp` | `m` | MCP server monitoring (feature-gated) |
| `dream` | `d` | Speculative background execution |

### Task Lifecycle

```
pending -> running -> completed | failed | killed
```

Terminal states: `completed`, `failed`, `killed`. The system guards against injecting messages into dead agents.

### Task State

```
id: string              // prefix + 8 random alphanumeric chars (~2.8T combinations)
type: TaskType
status: TaskStatus
description: string
toolUseId?: string      // Links to model's tool_use block
startTime: number
endTime?: number
totalPausedMs?: number
outputFile: string      // Disk path for streaming output
outputOffset: number    // Streaming resume point
notified: boolean       // UI notification flag
```

### Task Registration

Tasks are registered through a factory pattern with feature-gating:
- Always loaded: Shell, Agent, Remote Agent, Dream
- Feature-gated: Workflow, Monitor MCP

---

## 7. Permission System

### 7 Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Ask user for permission on each tool use |
| `acceptEdits` | Auto-approve file edits, ask for others |
| `plan` | Plan-mode approval required before execution |
| `dontAsk` | Trust mode (deprecated) |
| `bypassPermissions` | Admin bypass (requires explicit flag) |
| `auto` | Classifier auto-approve for safe patterns (feature-gated) |
| `bubble` | Delegate permission decision to parent agent (internal) |

### Triple Validation

1. **Input Validation** — Semantic checks on tool input
2. **Tool-Specific Permission Check** — Tool's own permission logic
3. **General Permission Framework** — Rule matching, mode checking, classifier

### Rule Source Priority (high to low)

1. `session` — Current session overrides
2. `command` — Slash command context
3. `cliArg` — Command-line arguments
4. `policySettings` — Organizational policies
5. `flagSettings` — Feature flag settings
6. `localSettings` — Project-local settings (gitignored)
7. `projectSettings` — Shared project settings
8. `userSettings` — User-level global settings

### Permission Decision Flow

```
Tool Call Requested
  -> Input Validation (invalid -> deny)
  -> Tool-Specific Check (denied -> deny)
  -> Rule Matching:
      alwaysDeny matched -> deny
      alwaysAllow matched -> allow
      alwaysAsk matched -> ask user
      no rule matched -> check mode:
          bypass -> allow
          plan -> ask user
          auto -> transcript classifier (safe -> allow, uncertain -> ask)
          default -> ask user
          acceptEdits (file) -> allow, (other) -> ask
          bubble -> delegate to parent agent
  -> User Dialog:
      Allow Once -> allow
      Allow Always -> save rule + allow
      Deny -> deny
```

---

## 8. State Management

### Central State Store

A Zustand-like reactive store holding:

- **Settings & Config** — Settings, model selection, verbose mode
- **Task State** — Map of task ID to TaskState (all running/pending tasks)
- **Agent Name Registry** — Map of agent name to AgentId for message routing
- **MCP State** — Connected clients, tools, commands, resources
- **Plugin State** — Enabled/disabled plugins, commands, errors
- **Permission Context** — Current mode, rules, working directories
- **Bridge State** — Remote session connection status
- **UI State** — View modes, spinners, selection state
- **Companion State** — Buddy system metadata

### React Contexts

- **Mailbox Provider** — Agent communication queue
- **Voice Provider** — Voice I/O integration
- **Modal/Overlay Context** — UI overlays
- **Notification Context** — Toast messages + task signals
- **Queued Message Context** — Message buffering
- **Stats Context** — FPS metrics for UI profiling

### Persistence Layer

- User configuration file
- Project settings file
- Session transcripts
- File history
- Session cost tracking

---

## 9. MCP Integration

### 7 Transport Types

| Transport | Config |
|-----------|--------|
| `stdio` | command, args, env |
| `sse` | url, headers |
| `sse-ide` | url (IDE-only variant) |
| `http` | url, headers (streamable HTTP) |
| `ws` | url, headers (WebSocket) |
| `sdk` | name (internal SDK server) |
| `managed-proxy` | Managed server access |

### Configuration Sources

1. User settings — Global MCP server definitions
2. Project settings — Project-scoped MCP servers
3. CLI flag — Runtime config override
4. Agent frontmatter — Agent-specific MCP servers (by reference or inline)
5. Plugin manifest — Plugin-provided MCP servers

### Tool Marshaling

MCP tools are automatically converted to the internal Tool interface:
- Names prefixed: `mcp__servername__toolname`
- Schemas translated: MCP JSON Schema -> internal Zod representation
- Results mapped through result content transformer
- Memoized with LRU cache for performance

### Resource Injection

Server resources are fetched per MCP connection and injected into the system prompt as "Available resources" sections. The model can access them via Resource Reading tools.

### Auth & Elicitation

- OAuth flow management with token refresh
- Step-up authentication detection
- Elicitation handler for interactive OAuth URL prompts
- Permission callbacks over messaging channels

---

## 10. Plugin & Hook System

### Plugin Architecture

**Plugin sources:**
- **Built-in** — Ship with the system, toggleable
- **Marketplace** — Downloaded from plugin repositories
- **Custom** — User-managed plugin directory

**Plugin components:**
- Commands (slash commands)
- Agents (AI agent definitions)
- Skills (reusable prompts/actions)
- Hooks (event handlers)
- MCP Servers
- LSP Servers (language server protocol)
- Output Styles (rendering themes)

### 27 Hook Events

| Category | Events |
|----------|--------|
| **Lifecycle** (5) | Setup, SessionStart, SessionEnd, Stop, StopFailure |
| **Tool** (3) | PreToolUse, PostToolUse, PostToolUseFailure |
| **Permission** (2) | PermissionRequest, PermissionDenied |
| **Agent** (5) | SubagentStart, SubagentStop, TeammateIdle, TaskCreated, TaskCompleted |
| **Compaction** (2) | PreCompact, PostCompact |
| **Input** (3) | UserPromptSubmit, Elicitation, ElicitationResult |
| **System** (5) | Notification, ConfigChange, InstructionsLoaded, CwdChanged, FileChanged |
| **Worktree** (2) | WorktreeCreate, WorktreeRemove |

### Hook Implementations

- **Command** — Shell command execution (bash/sh)
- **Prompt** — LLM prompt hook (model-based evaluation)
- **HTTP** — HTTP webhook to external service
- **Agent** — Agentic verifier hook (autonomous verification)

### Hook Response Schema

```
{
  continue?: boolean          // Continue after hook? (default: true)
  decision?: 'approve'|'block'  // For PreToolUse
  reason?: string
  systemMessage?: string      // Warning to user
  hookSpecificOutput?: {
    updatedInput?: object     // Modify tool input
    additionalContext?: string
    watchPaths?: string[]     // FileChanged watches
  }
}
```

### Skills vs Plugins

- **Skills** are reusable prompts/actions invoked explicitly (like commands). They support tool filtering, model override, and execution context control.
- **Plugins** are packages that provide skills + hooks + MCP servers + commands + agents. They are the distribution unit.

---

## 11. Compaction Engine

### 5 Strategies

| Strategy | Trigger | Behavior |
|----------|---------|----------|
| **History Snip** (feature-gated) | Applied first | Truncate old messages, keep recent N turns |
| **Microcompact** (always available) | Before API call | Cache-aware compression, replace tool_use IDs, shrink tool results |
| **Auto-Compact** (always available) | When context exceeds threshold | LLM summarizes history, creates compact boundary, preserves key decisions |
| **Context Collapse** (feature-gated) | Feature-gated activation | Granular context preservation over full history |
| **Session Memory Compact** | Periodic extraction | Extract durable memories for cross-session knowledge |

### Post-Compact Restoration

After compaction, the system restores critical context:
- Up to 5 recently-modified files (5K tokens each, 50K total budget)
- Active plan attachment
- Active skill attachment
- Running async agent status
- Skill token budget: 25K tokens

### Compaction Flow

```
1. History Snip (if enabled) — first pass, removes oldest messages
2. Microcompact — before every API call, compresses cached content
3. Auto-Compact — reactive, triggered when context exceeds threshold
4. Context Collapse — feature-gated, alternative to auto-compact
5. Session Memory — periodic, extracts long-term knowledge to memory store
```

---

## 12. Agent Communication

### Mailbox System

Each session has a singleton Mailbox instance with queue + async waiter pattern:

- `send(message)` — Inject message, wake waiting receivers
- `poll(filter)` — Non-blocking receive matching predicate
- `receive(filter)` — Blocking async-wait receive matching predicate
- `subscribe()` — Reactive notifications on message arrival

### Message Format

```
id: string
source: 'user' | 'teammate' | 'system' | 'tick' | 'task'
content: string
from?: string        // Sender agent name
color?: string       // Display color
timestamp: string    // ISO timestamp
```

### Task Notification Protocol

Workers report results to the coordinator via XML task notifications:

```xml
<task-notification>
  <task-id>a1b2c3d4</task-id>
  <status>completed</status>
  <summary>Implement user auth</summary>
  <result>Successfully implemented OAuth2 flow...</result>
  <usage>Input: 15K, Output: 3K tokens</usage>
</task-notification>
```

### Named Agent Registry

A Map of agent name to agent ID, populated at spawn time. Enables:
- **Direct routing** — Target by agent ID
- **Named routing** — Target by human-readable name via SendMessage tool
- **Broadcast** — Task completion notifications to coordinator

---

## 13. Prompt Cache Sharing

### The Problem

When spawning N subagents, each shares the same system prompt, tool definitions, and context prefix. Without optimization, each agent pays full token cost for this shared prefix.

### The Solution: Cache-Safe Parameters

At fork time, the system freezes a set of parameters that compose the API cache key:

```
Cache-Safe Parameters:
  - System Prompt (largest segment, cached first)
  - User Context (instructions, memory files)
  - System Context (environment info, capabilities)
  - Tool Use Context (tool definitions, schemas)
  - Fork Context Messages (conversation prefix)
```

If a fork uses identical Cache-Safe Parameters as the parent, it inherits the parent's prompt cache. The API recognizes the shared prefix and skips re-processing.

### Isolation Guarantees

While cache is shared, execution state is isolated:
- File State Cache — Cloned (changes don't affect parent)
- Denial Tracking — Fresh instance (permission state isolated)
- Content Replacement — Cloned (tool result budgets independent)
- App State Writes — No-op (subagent can't mutate parent state)
- Abort Controller — Child of parent (parent cancellation cascades to children)

### Benefits

- **Cost reduction** — significant cache hit rate on shared prefix
- **Latency reduction** — TTFT (time to first token) improvement
- **Horizontal scaling** — N agents share 1 cache entry

---

## 14. Bridge / Remote Execution

### Architecture

The Bridge connects a local CLI session to a remote cloud environment:

**Local side:**
- Bridge Loop — Connection manager with backoff
- Bridge API Client — JWT authentication
- Session Spawner — Creates remote sessions
- REPL Bridge — Interactive integration
- WebSocket Transport — V1/V2 transport factories

**Remote side:**
- Environment — Trusted device registration
- Session — Isolated workspace
- Remote Query Engine — Full conversation capability
- Remote Tools — Shell, file operations

### Communication Protocol

- **SDK Messages** — Standard message format over WebSocket
- **Control Messages** — Permission requests/responses, cancellation
- **Polling** — Server-sent updates for long-running operations
- **Tool Result Forwarding** — Relay execution results back to local

### Session Lifecycle

```
1. Register environment (trusted device + JWT)
2. Spawn session (local or cloud)
3. Poll for updates via WebSocket
4. Forward tool results between local and remote
5. Cleanup on disconnect
```

---

## 15. Message Types

### Internal Message Types

| Type | Description |
|------|-------------|
| `UserMessage` | User input with content blocks and UUID |
| `AssistantMessage` | Model response with cost, duration, usage, model info |
| `AttachmentMessage` | Memory/context attachments with source tracking |
| `ProgressMessage` | Tool execution progress updates |
| `SystemMessage` | System notifications and warnings |
| `ToolUseSummaryMessage` | Compact summary of tool executions |
| `SystemCompactBoundary` | Marker for compaction boundaries |
| `TombstoneMessage` | Internal placeholder for removed messages |

### SDK Message Types

| Type | Description |
|------|-------------|
| `SDKUserMessage` | User message for SDK/headless use |
| `SDKAssistantMessage` | Full assistant response |
| `SDKPartialAssistantMessage` | Streaming partial response with incremental deltas |

### Task State

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Prefix + 8 random chars (~2.8T combinations) |
| `type` | TaskType | One of 7 task types |
| `status` | TaskStatus | pending / running / completed / failed / killed |
| `description` | string | Human-readable task description |
| `startTime` | number | Unix timestamp |
| `endTime` | number | Unix timestamp (when terminal) |
| `outputFile` | string | Disk path for streaming output |
| `outputOffset` | number | Resume point for streaming |
| `notified` | boolean | Whether UI was notified of completion |

---

## Architectural Principles Summary

1. **Streaming-first** — AsyncGenerator everywhere, zero-blocking pipelines
2. **In-process** — No external orchestration, serverless-friendly
3. **Cache-optimized** — Prompt cache sharing across agent forks
4. **Permission-aware** — Triple validation, 7 modes, 8 rule sources
5. **Extensible** — 27 hooks, plugin system, MCP protocol
6. **Resilient** — Max-tokens recovery, 5 compaction strategies, cascading abort
7. **Observable** — Cost tracking, usage metrics, session transcripts, analytics
