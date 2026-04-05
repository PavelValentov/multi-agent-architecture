# Multi-Agent System — Full Architecture (Mermaid Diagrams)

> Conceptual architecture of a production-grade multi-agent AI system with orchestration, tool execution, permissions, and distributed execution.

## About This Document

This document contains **14 Mermaid diagrams** that describe the complete architecture of a multi-agent orchestration system. The diagrams are based on reverse-engineering of a real production codebase and verified against actual source code.

### How to Use

- **View diagrams**: Open this file in any Mermaid-compatible renderer — VS Code (Mermaid Preview extension), GitHub, [mermaid.live](https://mermaid.live), or any Markdown viewer with Mermaid support.
- **Navigate**: Each diagram covers a distinct architectural layer. Start with Diagram 1 for the high-level overview, then drill into specific areas.
- **Green elements** (`#00e676`): Indicate components that were discovered or significantly improved during verification — they represent findings that most public analyses missed.
- **Companion document**: See [ARCHITECTURE.md](ARCHITECTURE.md) for a full textual description of every component shown here.

### Diagram Index

| # | Diagram | Focus |
|---|---------|-------|
| 1 | High-Level System Architecture | All modules and connections |
| 2 | Query Loop Lifecycle | Conversation turn sequence |
| 3 | Agent Spawning Flow | 5 execution paths |
| 4 | Coordinator Mode | Multi-worker orchestration phases |
| 5 | Tool Registry & Dispatch | 50+ tools, execution pipeline |
| 6 | Permission System | 7 modes, rule priority flowchart |
| 7 | State Management | Central store, contexts, persistence |
| 8 | MCP Integration | 7 transports, tool marshaling |
| 9 | Plugin & Hook System | 27 events by category |
| 10 | Compaction Strategies | 5 context management approaches |
| 11 | Bridge / Remote Architecture | WebSocket remote execution |
| 12 | Message Type Hierarchy | Class diagram of all message types |
| 13 | Agent Communication System | Mailbox, notifications, routing |
| 14 | Prompt Cache Sharing | Cache optimization architecture |

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Entry["Entry Points"]
        CLI["CLI Bootstrap"]
        MAIN["Session Initializer"]
        RECOVERY["Recovery CLI"]
        BRIDGE_ENTRY["Remote Bridge"]
        DAEMON["Background Daemon"]
    end

    subgraph Core["Core Engine"]
        QE["Query Engine<br/>Session State Machine"]
        QL["Conversation Turn Loop<br/>Async Generator"]
        DEPS["Dependency Injection<br/>Model Calls, Compaction"]
    end

    subgraph ToolSystem["Tool Orchestration"]
        TOOLS_REG["Tool Registry<br/>Pool Assembly"]
        TOOL_ORCH["Tool Orchestrator<br/>Concurrency Partitioning"]
        STREAM_EXEC["Streaming Executor<br/>Ordered Buffering"]
        TOOL_EXEC["Tool Executor<br/>Permission Check + Dispatch"]
    end

    subgraph AgentSystem["Agent System"]
        AGENT_TOOL["Agent Spawner"]
        RUN_AGENT["Agent Runner<br/>Forked Engine"]
        FORK["Context Cloner<br/>Cache-Safe Fork"]
        COORD["Coordinator<br/>Multi-Worker Orchestration"]
        TEAMMATE["Teammate Spawner<br/>Named Agent Registry"]
    end

    subgraph TaskSystem["Task Management"]
        TASK_DEF["Task Definition<br/>7 Types / 5 Statuses"]
        TASK_REG["Task Registry<br/>Type Dispatch"]
        LOCAL_SHELL["Shell Task"]
        LOCAL_AGENT["Agent Task"]
        REMOTE_AGENT["Remote Agent Task"]
        DREAM["Dream Task"]
        WORKFLOW["Workflow Task ⚑"]
        MONITOR["Monitor Task ⚑"]
    end

    subgraph State["State Management"]
        APPSTATE["Central State Store"]
        MAILBOX["Agent Mailbox<br/>send / poll / receive"]
        NOTIFICATIONS["Notification System"]
    end

    subgraph Services["Service Layer"]
        API["API Client<br/>Streaming + Retry"]
        COMPACT["Compaction Engine<br/>5 Strategies:<br/>snip, micro, auto,<br/>contextCollapse, sessionMemory"]
        MCP["MCP Client<br/>7 Transports"]
        ANALYTICS["Analytics & Telemetry"]
        OAUTH["OAuth Service"]
        SESSION["Session Memory<br/>Transcript + History"]
    end

    subgraph Permissions["Permission System"]
        PERM_CTX["Permission Context<br/>7 Modes: default, acceptEdits,<br/>plan, dontAsk, bypass, auto ⚑, bubble"]
        CAN_USE["Permission Checker<br/>Triple Validation"]
        RULES["Permission Rules<br/>allow / deny / ask<br/>by source priority"]
    end

    subgraph Plugins["Plugin & Skill System"]
        PLUGIN_SYS["Plugin Manager<br/>builtin + marketplace + custom"]
        SKILL_SYS["Skill System<br/>bundled + disk skills"]
        HOOKS_SYS["Hook System<br/>27 Hook Events"]
    end

    subgraph UI["Terminal UI (React)"]
        APP["Root Component"]
        REPL["REPL Interface"]
        COMPONENTS["Component Library"]
        INK["Custom Renderer"]
    end

    CLI --> MAIN
    CLI -->|"bridge mode"| BRIDGE_ENTRY
    CLI -->|"recovery mode"| RECOVERY
    CLI -->|"daemon mode"| DAEMON
    MAIN --> QE
    QE --> QL
    QL --> DEPS
    DEPS --> API
    QL --> TOOL_ORCH
    TOOL_ORCH --> STREAM_EXEC
    TOOL_ORCH --> TOOL_EXEC
    TOOL_EXEC --> CAN_USE
    CAN_USE --> PERM_CTX
    PERM_CTX --> RULES
    TOOLS_REG --> TOOL_ORCH
    TOOL_EXEC --> AGENT_TOOL
    AGENT_TOOL --> RUN_AGENT
    AGENT_TOOL --> TEAMMATE
    RUN_AGENT --> FORK
    FORK -->|"new Query Engine"| QE
    AGENT_TOOL --> COORD
    COORD -->|"spawn workers"| AGENT_TOOL
    AGENT_TOOL --> TASK_REG
    TASK_REG --> LOCAL_AGENT
    TASK_REG --> REMOTE_AGENT
    TASK_REG --> LOCAL_SHELL
    TASK_REG --> DREAM
    TASK_REG --> WORKFLOW
    TASK_REG --> MONITOR
    TASK_DEF --> TASK_REG
    LOCAL_AGENT --> APPSTATE
    REMOTE_AGENT --> BRIDGE_ENTRY
    QL --> COMPACT
    QE --> SESSION
    MAIN --> APPSTATE
    APPSTATE --> MAILBOX
    APPSTATE --> NOTIFICATIONS
    MAIN --> MCP
    MCP --> TOOLS_REG
    MAIN --> PLUGIN_SYS
    PLUGIN_SYS --> SKILL_SYS
    PLUGIN_SYS --> HOOKS_SYS
    HOOKS_SYS --> TOOL_EXEC
    MAIN --> APP
    APP --> REPL
    REPL --> COMPONENTS
    REPL --> INK
    MAIN --> ANALYTICS

    style Entry fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
    style Core fill:#fff0f0,stroke:#e94560,stroke-width:3px,color:#333
    style AgentSystem fill:#f3f0ff,stroke:#6c5ce7,stroke-width:2px,color:#333
    style ToolSystem fill:#f5f3ff,stroke:#a29bfe,stroke-width:2px,color:#333
    style TaskSystem fill:#fff0f5,stroke:#fd79a8,stroke-width:2px,color:#333
    style State fill:#f0ffff,stroke:#00cec9,stroke-width:2px,color:#333
    style Services fill:#f0fff8,stroke:#55efc4,stroke-width:2px,color:#333
    style Permissions fill:#fff5f0,stroke:#e17055,stroke-width:2px,color:#333
    style Plugins fill:#f5f5fa,stroke:#a29bfe,stroke-width:1px,color:#333
    style UI fill:#f0f5ff,stroke:#74b9ff,stroke-width:2px,color:#333
    style TEAMMATE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style HOOKS_SYS fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style COMPACT fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 2. Query Loop — Conversation Turn Lifecycle

```mermaid
sequenceDiagram
    participant U as User / UI
    participant QE as Query Engine
    participant QL as Turn Loop
    participant API as LLM API
    participant TO as Tool Orchestrator
    participant T as Tool Executor
    participant P as Permissions
    participant C as Compaction

    U->>QE: submitMessage(prompt)
    activate QE
    QE->>QE: Process User Input<br/>slash commands, skills
    QE->>QE: Build System Prompt<br/>+ plugins + MCP + memory
    QE->>QL: start query loop
    activate QL

    loop Conversation Turns (while true)
        QL->>C: snipCompact / microcompact /<br/>autoCompact / contextCollapse /<br/>sessionMemoryCompact
        C-->>QL: compacted messages

        QL->>API: stream model request
        activate API
        API-->>QL: stream events<br/>(content blocks, tool_use blocks...)
        deactivate API

        QL-->>U: yield stream events<br/>(real-time tokens)

        Note over QL: tool_use blocks detected via<br/>streaming, NOT stop_reason<br/>(stop_reason is unreliable)

        alt tool_use blocks found in stream
            QL->>TO: execute tool calls
            activate TO

            TO->>TO: partition by concurrency<br/>safe → parallel, unsafe → serial

            par Concurrent Tools (max 10)
                TO->>P: check permission
                P-->>TO: allow / deny / ask
                TO->>T: execute tool
                T-->>TO: ToolResult
            end

            loop Serial Tools (one by one)
                TO->>P: check permission
                P-->>TO: allow / deny / ask
                TO->>T: execute tool
                T-->>TO: ToolResult
            end

            TO-->>QL: all ToolResults
            deactivate TO

            QL->>QL: push assistant + tool_result<br/>to messages
            QL->>QL: increment turn count
            Note over QL: continue loop
        else No tool_use blocks
            alt max_tokens reached
                QL->>QL: recovery attempt<br/>(max 3 retries)
                Note over QL: continue with prefix
            else end_turn
                QL-->>QE: Terminal { success: true }
                Note over QL: break loop
            end
        end
    end

    deactivate QL
    QE-->>U: yield result message
    deactivate QE
```

---

## 3. Agent Spawning & Execution Flow

```mermaid
graph TB
    subgraph Parent["Parent Agent Context"]
        PA_QE["Parent Query Engine"]
        PA_QL["Parent Turn Loop"]
        PA_TOOLS["Parent Tool Orchestrator"]
    end

    PA_QL --> PA_TOOLS
    PA_TOOLS -->|"tool_use: Agent"| AGENT_CALL

    AGENT_CALL{"Agent Spawner"}

    AGENT_CALL -->|"synchronous"| SYNC
    AGENT_CALL -->|"background=true"| ASYNC
    AGENT_CALL -->|"isolation=worktree"| WORKTREE
    AGENT_CALL -->|"isolation=remote"| REMOTE
    AGENT_CALL -->|"name + team"| TEAMMATE

    subgraph SyncPath["Synchronous Execution"]
        SYNC["Run Agent"]
        CLONE["Create Subagent Context<br/>- Clone File State<br/>- Clone Content Replacement<br/>- Fresh Denial Tracking<br/>- Freeze Cache-Safe Params"]
        CHILD_QE["Child Query Engine<br/>(isolated)"]
        CHILD_QL["Child Turn Loop"]
        RESULT["Return messages<br/>+ accumulated usage"]
    end

    subgraph AsyncPath["Background Execution"]
        ASYNC["Background Agent Task"]
        TASK_STATE["State Store: tasks[id]<br/>status: running"]
        OUTPUT_FILE["Output File<br/>streaming output"]
        NOTIFY["Task Notification<br/>to parent mailbox"]
    end

    subgraph WorktreePath["Git Worktree Isolation"]
        WORKTREE["Create Agent Worktree"]
        WT_BRANCH["Temp branch + worktree"]
        WT_AGENT["Run Agent in worktree"]
        WT_CLEANUP["Auto-cleanup if no changes"]
    end

    subgraph RemotePath["Remote Execution"]
        REMOTE["Remote Agent Task"]
        CCR_API["Remote API Submission"]
        WS_POLL["WebSocket Polling"]
        REMOTE_RESULT["Remote Result<br/>via Message Adapter"]
    end

    subgraph TeammatePath["Teammate Spawning"]
        TEAMMATE_SPAWN["Spawn Teammate<br/>Named Agent"]
        TM_REGISTER["Register in<br/>Agent Name Registry"]
        TM_PROCESS["In-Process Teammate<br/>Shared Terminal"]
        TM_NOTIFY["Teammate Spawned<br/>Notification"]
    end

    SYNC --> CLONE
    CLONE --> CHILD_QE
    CHILD_QE --> CHILD_QL
    CHILD_QL --> RESULT
    RESULT -->|"tool_result"| PA_QL

    ASYNC --> TASK_STATE
    TASK_STATE --> OUTPUT_FILE
    OUTPUT_FILE --> NOTIFY
    NOTIFY -->|"task-notification"| PA_QL

    WORKTREE --> WT_BRANCH
    WT_BRANCH --> WT_AGENT
    WT_AGENT --> WT_CLEANUP
    WT_CLEANUP -->|"tool_result + branch"| PA_QL

    REMOTE --> CCR_API
    CCR_API --> WS_POLL
    WS_POLL --> REMOTE_RESULT
    REMOTE_RESULT -->|"tool_result"| PA_QL

    TEAMMATE --> TEAMMATE_SPAWN
    TEAMMATE_SPAWN --> TM_REGISTER
    TM_REGISTER --> TM_PROCESS
    TM_PROCESS --> TM_NOTIFY
    TM_NOTIFY -->|"task-notification"| PA_QL

    subgraph CacheOptimization["Prompt Cache Sharing"]
        CACHE["Cache-Safe Parameters<br/>- System Prompt (frozen)<br/>- User Context<br/>- System Context<br/>- Tool Context<br/>- Fork Context Messages"]
    end

    CLONE -.->|"inherits parent cache"| CACHE

    style Parent fill:#fff0f0,stroke:#e94560,stroke-width:2px,color:#333
    style SyncPath fill:#f0f5ff,stroke:#74b9ff,stroke-width:2px,color:#333
    style AsyncPath fill:#f0ffff,stroke:#00cec9,stroke-width:2px,color:#333
    style WorktreePath fill:#f5f3ff,stroke:#a29bfe,stroke-width:2px,color:#333
    style RemotePath fill:#f0fff8,stroke:#55efc4,stroke-width:2px,color:#333
    style CacheOptimization fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
    style TeammatePath fill:#f0fff5,stroke:#00e676,stroke-width:3px,color:#333
    style TEAMMATE_SPAWN fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style TM_REGISTER fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style TM_PROCESS fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style TM_NOTIFY fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 4. Coordinator Mode — Multi-Worker Orchestration

```mermaid
graph TB
    subgraph Coordinator["Coordinator Agent"]
        C_QE["Coordinator Query Engine"]
        C_PROMPT["Role: Orchestrate,<br/>Synthesize, Direct"]
        C_TOOLS["Coordinator Tools:<br/>Worker Spawning, Messaging,<br/>Task Control, Task Monitoring"]
    end

    C_QE --> PHASE1
    C_QE --> PHASE2
    C_QE --> PHASE3
    C_QE --> PHASE4

    subgraph PHASE1["Phase 1: Research"]
        direction LR
        R1["Worker A<br/>explore codebase"]
        R2["Worker B<br/>read docs"]
        R3["Worker C<br/>analyze tests"]
    end

    subgraph PHASE2["Phase 2: Synthesis"]
        SYNTH["Coordinator reads findings<br/>forms implementation plan"]
    end

    subgraph PHASE3["Phase 3: Implementation"]
        direction LR
        I1["Worker D<br/>implement feature"]
        I2["Worker E<br/>update tests"]
        I3["Worker F<br/>update docs"]
    end

    subgraph PHASE4["Phase 4: Verification"]
        V1["Worker G<br/>run tests + typecheck"]
    end

    R1 -->|"task-notification"| SYNTH
    R2 -->|"task-notification"| SYNTH
    R3 -->|"task-notification"| SYNTH
    SYNTH --> I1
    SYNTH --> I2
    SYNTH --> I3
    I1 -->|"task-notification"| V1
    I2 -->|"task-notification"| V1
    I3 -->|"task-notification"| V1

    subgraph WorkerToolset["Worker Tool Set"]
        WT["Shell Execution, File I/O,<br/>Search & Navigation,<br/>Web Access, Notebook Editing,<br/>MCP Extensions"]
    end

    subgraph SharedState["Shared State & Communication"]
        AS["Task State Store"]
        MB["Agent Mailbox<br/>send / poll / receive"]
        SEND_MSG_NODE["Inter-Agent Messaging<br/>Continue / Redirect Workers"]
        TASK_STOP_NODE["Task Control<br/>Kill Unneeded Workers"]
        SCRATCH["Scratchpad<br/>Cross-Worker Knowledge"]
    end

    R1 -.-> WorkerToolset
    I1 -.-> WorkerToolset
    V1 -.-> WorkerToolset

    R1 --> AS
    I1 --> AS
    V1 --> AS
    AS --> MB
    MB --> C_QE

    C_QE -.->|"continue worker"| SEND_MSG_NODE
    SEND_MSG_NODE -.-> I1
    C_QE -.->|"stop worker"| TASK_STOP_NODE
    TASK_STOP_NODE -.-> R1
    I1 -.->|"shared knowledge"| SCRATCH
    I2 -.->|"shared knowledge"| SCRATCH

    style Coordinator fill:#fff0f0,stroke:#e94560,stroke-width:3px,color:#333
    style PHASE1 fill:#f0f5ff,stroke:#74b9ff,stroke-width:2px,color:#333
    style PHASE2 fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
    style PHASE3 fill:#f0f5ff,stroke:#74b9ff,stroke-width:2px,color:#333
    style PHASE4 fill:#f0fff8,stroke:#55efc4,stroke-width:2px,color:#333
    style WorkerToolset fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
    style SharedState fill:#f0ffff,stroke:#00cec9,stroke-width:2px,color:#333
    style SEND_MSG_NODE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style TASK_STOP_NODE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style SCRATCH fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 5. Tool Registry & Dispatch

```mermaid
graph LR
    subgraph Registry["Tool Registry"]
        GET_ALL["Collect All<br/>Base Tools"]
        GET_TOOLS["Filter by<br/>Permission Context"]
        ASSEMBLE["Assemble Tool Pool<br/>Built-in + MCP"]
        FILTER["Apply Deny Rules"]
    end

    subgraph BuiltInTools["Built-in Tools (~50+)"]
        subgraph IO["Core I/O"]
            IO_TOOLS["Shell Execution<br/>File Read / Edit / Write<br/>Pattern Search (Glob, Grep)"]
        end

        subgraph Web["Web & Network"]
            WEB_TOOLS["HTTP Fetch<br/>Web Search<br/>Browser Automation ⚑"]
        end

        subgraph Agent["Agent & Task Management"]
            AGENT_TOOLS["Agent Spawning<br/>Task CRUD<br/>Task Output Retrieval"]
        end

        subgraph Planning["Planning & Organization"]
            PLAN_TOOLS["Plan Mode Entry / Exit<br/>Brief Mode<br/>Todo Management"]
        end

        subgraph Specialized["Specialized"]
            SPEC_TOOLS["Notebook Editing<br/>Skill Invocation<br/>Deferred Tool Search<br/>Sleep ⚑ / Cron ⚑<br/>Configuration"]
        end

        subgraph CoordTools["Coordinator-Only"]
            COORD_TOOLS["Team Create / Delete<br/>Inter-Agent Messaging<br/>Synthetic Output"]
        end

        subgraph MCPBridge["MCP Bridge"]
            MCP_BRIDGE_TOOLS["Resource Listing / Reading<br/>LSP Integration"]
        end
    end

    subgraph External["External MCP Tools"]
        MCP_SERVER["External MCP Server A"]
        MCP_SERVER2["External MCP Server B"]
    end

    GET_ALL --> IO
    GET_ALL --> Web
    GET_ALL --> Agent
    GET_ALL --> Planning
    GET_ALL --> Specialized
    GET_ALL --> CoordTools
    GET_ALL --> MCPBridge
    GET_ALL --> GET_TOOLS
    GET_TOOLS --> FILTER
    FILTER --> ASSEMBLE
    MCP_SERVER --> ASSEMBLE
    MCP_SERVER2 --> ASSEMBLE

    subgraph Execution["Execution Pipeline"]
        PARTITION["Partition by Concurrency"]
        PAR["Parallel Batch<br/>(concurrency-safe)<br/>max 10"]
        SER["Serial Queue<br/>(non-concurrent)"]
    end

    ASSEMBLE --> PARTITION
    PARTITION --> PAR
    PARTITION --> SER

    style Registry fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
    style BuiltInTools fill:#fff0f0,stroke:#e94560,stroke-width:1px,color:#333
    style IO fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style Web fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style Agent fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style Planning fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style Specialized fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style CoordTools fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style MCPBridge fill:#f5f5fa,stroke:#666,stroke-width:1px,color:#333
    style External fill:#f0ffff,stroke:#00cec9,stroke-width:2px,color:#333
    style Execution fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
```

---

## 6. Permission System Flow

```mermaid
flowchart TD
    CALL["Tool Call Requested"]
    CALL --> VALIDATE["Input Validation<br/>Semantic Checks"]
    VALIDATE -->|"invalid"| DENY_RESULT["Deny: validation error"]
    VALIDATE -->|"valid"| CHECK_PERM

    CHECK_PERM["Tool-Specific<br/>Permission Check"]
    CHECK_PERM -->|"denied"| DENY_RESULT
    CHECK_PERM -->|"passed"| GENERAL

    GENERAL["General Permission Framework"]
    GENERAL --> MATCH_RULES

    MATCH_RULES{"Match Rules<br/>(by source priority)"}
    MATCH_RULES -->|"alwaysDeny matched"| DENY_RESULT
    MATCH_RULES -->|"alwaysAllow matched"| ALLOW_RESULT["Allow"]
    MATCH_RULES -->|"alwaysAsk matched"| ASK_USER
    MATCH_RULES -->|"no rule matched"| MODE_CHECK

    MODE_CHECK{"Permission Mode?"}
    MODE_CHECK -->|"bypassPermissions"| ALLOW_RESULT
    MODE_CHECK -->|"plan"| ASK_USER
    MODE_CHECK -->|"auto ⚑"| CLASSIFIER
    MODE_CHECK -->|"default"| ASK_USER
    MODE_CHECK -->|"dontAsk"| ALLOW_RESULT
    MODE_CHECK -->|"acceptEdits (file edit)"| ALLOW_RESULT
    MODE_CHECK -->|"acceptEdits (other)"| ASK_USER
    MODE_CHECK -->|"bubble (internal)"| BUBBLE

    CLASSIFIER["Transcript Classifier<br/>(hook-based)"]
    CLASSIFIER -->|"high confidence safe"| ALLOW_RESULT
    CLASSIFIER -->|"uncertain"| ASK_USER

    BUBBLE["Bubble to Parent Agent<br/>Delegate Decision"]
    BUBBLE -->|"parent allows"| ALLOW_RESULT
    BUBBLE -->|"parent denies"| DENY_RESULT

    ASK_USER{"Show Permission Dialog"}
    ASK_USER -->|"Allow Once"| ALLOW_RESULT
    ASK_USER -->|"Allow Always"| SAVE_RULE["Save alwaysAllow Rule"]
    ASK_USER -->|"Deny"| DENY_RESULT
    SAVE_RULE --> ALLOW_RESULT

    subgraph RuleSources["Rule Source Priority (high to low)"]
        direction LR
        S1["session"] --> S2["command"]
        S2 --> S3["cliArg"]
        S3 --> S4["policySettings"]
        S4 --> S5["flagSettings"]
        S5 --> S6["localSettings"]
        S6 --> S7["projectSettings"]
        S7 --> S8["userSettings"]
    end

    MATCH_RULES -.-> RuleSources

    style RuleSources fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
    style ALLOW_RESULT fill:#27ae60,stroke:#2ecc71,color:#333
    style DENY_RESULT fill:#e74c3c,stroke:#c0392b,color:#333
    style CLASSIFIER fill:#f39c12,stroke:#e67e22,color:#333
    style BUBBLE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 7. State Management & Data Flow

```mermaid
graph TB
    subgraph AppState["Central State Store"]
        SETTINGS["Settings Configuration"]
        TASKS["Task State Map<br/>id to TaskState"]
        AGENTS["Agent Name Registry<br/>Name-to-ID Mapping"]
        MCP_STATE["MCP State<br/>clients, tools, commands, resources"]
        PLUGINS["Plugin State<br/>enabled, disabled, commands"]
        PERMS["Permission Context"]
        BRIDGE_STATE["Bridge State"]
        UI_STATE["UI State<br/>views, modes, spinners"]
        COMPANION["Companion State"]
    end

    subgraph Contexts["React Contexts"]
        MAILBOX_CTX["Mailbox Provider<br/>send / poll / receive"]
        VOICE_CTX["Voice Provider"]
        MODAL_CTX["Modal Context"]
        OVERLAY_CTX["Overlay Context"]
        NOTIF_CTX["Notification Context"]
        QUEUE_CTX["Queued Message Context"]
        STATS_CTX["Stats Context (FPS)"]
    end

    subgraph Hooks["Key Hooks"]
        USE_CAN["Permission Checker"]
        USE_BRIDGE["Bridge Manager"]
        USE_VOICE["Voice Integration"]
        USE_TYPE["Typeahead"]
        USE_KEYS["Keybindings"]
        USE_INBOX["Inbox Poller"]
        USE_SCROLL["Virtual Scroll"]
    end

    subgraph Persistence["Persistence Layer"]
        CONFIG_JSON["User Configuration"]
        SETTINGS_JSON["Project Settings"]
        TRANSCRIPT["Session Transcripts"]
        FILE_HISTORY["File History"]
        COST_FILE["Session Costs"]
    end

    AppState --> Contexts
    Contexts --> Hooks
    USE_CAN --> PERMS
    USE_BRIDGE --> BRIDGE_STATE
    USE_INBOX --> MAILBOX_CTX
    AppState --> SETTINGS_JSON
    AppState --> CONFIG_JSON
    TASKS --> TRANSCRIPT
    TASKS --> COST_FILE

    style AppState fill:#fff0f0,stroke:#e94560,stroke-width:2px,color:#333
    style Contexts fill:#f3f0ff,stroke:#6c5ce7,stroke-width:2px,color:#333
    style Hooks fill:#f5f3ff,stroke:#a29bfe,stroke-width:2px,color:#333
    style Persistence fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
    style AGENTS fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 8. MCP Integration Architecture

```mermaid
graph TB
    subgraph Config["MCP Configuration Sources"]
        USER_CFG["User Settings<br/>MCP Server Definitions"]
        PROJECT_CFG["Project Settings<br/>MCP Server Definitions"]
        CLI_CFG["CLI Config Flag"]
        AGENT_CFG["Agent Frontmatter<br/>MCP Server References"]
        PLUGIN_CFG["Plugin Manifest<br/>MCP Server Definitions"]
    end

    subgraph Transports["Transport Layer (7 Types)"]
        STDIO["stdio"]
        SSE["SSE"]
        SSE_IDE["SSE-IDE"]
        HTTP["Streamable HTTP"]
        WS["WebSocket"]
        SDK["SDK (Internal)"]
        CLAUDE_PROXY["Managed Proxy"]
    end

    subgraph Client["MCP Client"]
        CONNECT["Connection Manager"]
        TOOLS_FETCH["Tool Fetcher"]
        RESOURCES["Resource Fetcher"]
        AUTH["Auth Handler<br/>OAuth + Token Refresh"]
        MARSHAL["Tool Marshaler<br/>Schema Translation"]
    end

    subgraph Integration["System Integration"]
        TOOL_REG["Tool Pool Assembly"]
        SYS_PROMPT["System Prompt<br/>Resource Injection"]
        PERM["Permission Callbacks"]
        ELICIT["Elicitation Handler<br/>OAuth URL Prompts"]
    end

    Config --> CONNECT
    CONNECT --> STDIO
    CONNECT --> SSE
    CONNECT --> SSE_IDE
    CONNECT --> HTTP
    CONNECT --> WS
    CONNECT --> SDK
    CONNECT --> CLAUDE_PROXY
    CONNECT --> AUTH
    TOOLS_FETCH --> MARSHAL
    MARSHAL --> TOOL_REG
    RESOURCES --> SYS_PROMPT
    Client --> PERM
    Client --> ELICIT

    style Config fill:#fff0f0,stroke:#e94560,stroke-width:1px,color:#333
    style Transports fill:#f0ffff,stroke:#00cec9,stroke-width:2px,color:#333
    style Client fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
    style Integration fill:#f0fff8,stroke:#55efc4,stroke-width:2px,color:#333
    style SSE_IDE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style CLAUDE_PROXY fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 9. Plugin & Hook System

```mermaid
graph TB
    subgraph PluginSources["Plugin Sources"]
        BUILTIN["Built-in Plugins"]
        MARKET["Marketplace Plugins"]
        CUSTOM["Custom Plugins"]
    end

    subgraph PluginComponents["Plugin Components"]
        P_CMDS["Commands"]
        P_AGENTS["Agents"]
        P_SKILLS["Skills"]
        P_HOOKS["Hooks"]
        P_MCP["MCP Servers"]
        P_LSP["LSP Servers"]
        P_STYLES["Output Styles"]
    end

    subgraph HookEvents["27 Hook Events"]
        subgraph LifecycleHooks["Lifecycle (5)"]
            H_SETUP["Setup"]
            H_SESSION_START["SessionStart"]
            H_SESSION_END["SessionEnd"]
            H_STOP["Stop"]
            H_STOP_FAIL["StopFailure"]
        end
        subgraph ToolHooks["Tool Events (3)"]
            H_PRE["PreToolUse"]
            H_POST["PostToolUse"]
            H_POST_F["PostToolUseFailure"]
        end
        subgraph PermHooks["Permission (2)"]
            H_PERM_R["PermissionRequest"]
            H_PERM_D["PermissionDenied"]
        end
        subgraph AgentHooks["Agent Events (5)"]
            H_SUB_START["SubagentStart"]
            H_SUB_STOP["SubagentStop"]
            H_TEAMMATE["TeammateIdle"]
            H_TASK_CREATE["TaskCreated"]
            H_TASK_DONE["TaskCompleted"]
        end
        subgraph CompactHooks["Compaction (2)"]
            H_PRE_COMPACT["PreCompact"]
            H_POST_COMPACT["PostCompact"]
        end
        subgraph InputHooks["Input Events (3)"]
            H_PROMPT["UserPromptSubmit"]
            H_ELICIT["Elicitation"]
            H_ELICIT_R["ElicitationResult"]
        end
        subgraph SystemHooks["System Events (5)"]
            H_NOTIF["Notification"]
            H_CONFIG["ConfigChange"]
            H_INSTRUCT["InstructionsLoaded"]
            H_CWD["CwdChanged"]
            H_FILE["FileChanged"]
        end
        subgraph WorktreeHooks["Worktree (2)"]
            H_WT_CREATE["WorktreeCreate"]
            H_WT_REMOVE["WorktreeRemove"]
        end
    end

    subgraph HookImpl["Hook Implementations"]
        HOOK_CMD["Command (Shell)"]
        HOOK_PROMPT["Prompt (LLM Hook)"]
        HOOK_HTTP["HTTP (Webhook)"]
        HOOK_AGENT["Agent (Verifier)"]
    end

    subgraph HookResponse["Hook Response"]
        CONTINUE["{ continue: true }"]
        BLOCK["{ continue: false }"]
        MODIFY["{ updatedInput: ... }"]
    end

    PluginSources --> PluginComponents
    P_HOOKS --> HookEvents
    HookEvents --> HookImpl
    HookImpl --> HookResponse

    style PluginSources fill:#fff0f0,stroke:#e94560,stroke-width:1px,color:#333
    style PluginComponents fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
    style HookEvents fill:#f0fff5,stroke:#00e676,stroke-width:3px,color:#333
    style LifecycleHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style ToolHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style PermHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style AgentHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style CompactHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style InputHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style SystemHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style WorktreeHooks fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style HookImpl fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
    style HookResponse fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
```

---

## 10. Compaction Strategies

```mermaid
flowchart LR
    subgraph Input["Context Budget Check"]
        TOKENS["Current tokens vs<br/>context window"]
    end

    Input -->|"tokens exceed threshold"| STRATEGIES

    subgraph STRATEGIES["5 Compaction Strategies"]
        SNIP["History Snip ⚑<br/>-----------<br/>Truncate old messages<br/>Keep recent N turns<br/>Feature-gated"]
        MICRO["Microcompact<br/>-----------<br/>Cache-aware compression<br/>Replace tool_use IDs<br/>Shrink tool results"]
        AUTO["Auto-Compact<br/>-----------<br/>LLM summarizes history<br/>Creates compact boundary<br/>Preserves key decisions"]
        CTX_COLLAPSE["Context Collapse ⚑<br/>-----------<br/>Granular context<br/>preservation over<br/>full history"]
        SESSION_MEM["Session Memory<br/>Compact<br/>-----------<br/>Extract durable memories<br/>Cross-session knowledge"]
    end

    SNIP -->|"applied first"| MICRO
    MICRO -->|"before API call"| API_CALL["LLM API Call"]
    AUTO -->|"reactive,<br/>after threshold"| API_CALL
    CTX_COLLAPSE -->|"feature-gated"| API_CALL
    SESSION_MEM -->|"periodic<br/>extraction"| MEMORY_STORE["Memory Store"]

    style Input fill:#f5f5fa,stroke:#888,stroke-width:1px,color:#333
    style STRATEGIES fill:#fff0f0,stroke:#e94560,stroke-width:2px,color:#333
    style SESSION_MEM fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style CTX_COLLAPSE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style MEMORY_STORE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## 11. Bridge / Remote Execution Architecture

```mermaid
graph TB
    subgraph Local["Local Machine"]
        LOCAL_CLI["CLI Client"]
        BRIDGE_MAIN["Bridge Loop<br/>Connection Manager"]
        BRIDGE_API["Bridge API Client<br/>JWT Authentication"]
        SESSION_RUN["Session Spawner"]
        REPL_BRIDGE["REPL Bridge<br/>Interactive Integration"]
        WS_TRANSPORT["WebSocket Transport"]
    end

    subgraph Remote["Remote Environment"]
        ENV["Environment<br/>(trusted device)"]
        SESSION["Session<br/>(isolated workspace)"]
        REMOTE_QE["Remote Query Engine"]
        REMOTE_TOOLS["Remote Tools<br/>(Shell, File ops)"]
    end

    subgraph Protocol["Communication Protocol"]
        SDK_MSG["SDK Messages"]
        CTRL_MSG["Control Messages<br/>(permission, cancel)"]
        POLL["Poll for Updates"]
        FORWARD["Forward Tool Results"]
    end

    LOCAL_CLI -->|"bridge mode"| BRIDGE_MAIN
    BRIDGE_MAIN --> BRIDGE_API
    BRIDGE_API -->|"register env"| ENV
    BRIDGE_MAIN --> SESSION_RUN
    SESSION_RUN -->|"spawn"| SESSION
    SESSION --> REMOTE_QE
    REMOTE_QE --> REMOTE_TOOLS
    REPL_BRIDGE --> WS_TRANSPORT
    WS_TRANSPORT -->|"Secure WebSocket"| SESSION

    SDK_MSG --> WS_TRANSPORT
    CTRL_MSG --> WS_TRANSPORT
    WS_TRANSPORT --> POLL
    WS_TRANSPORT --> FORWARD

    style Local fill:#fff0f0,stroke:#e94560,stroke-width:2px,color:#333
    style Remote fill:#f0ffff,stroke:#00cec9,stroke-width:2px,color:#333
    style Protocol fill:#fffef0,stroke:#fdcb6e,stroke-width:2px,color:#333
```

---

## 12. Message Type Hierarchy

```mermaid
classDiagram
    class Message {
        &lt;&lt;union&gt;&gt;
    }

    class UserMessage {
        type: user
        content: ContentBlock[]
        uuid: string
        isMeta: boolean
    }

    class AssistantMessage {
        type: assistant
        message: APIMessage
        costUSD: number
        durationMs: number
        usage: TokenUsage
        model: string
    }

    class AttachmentMessage {
        type: attachment
        content: ContentBlock[]
        source: string
    }

    class ProgressMessage {
        type: progress
        toolUseId: string
        data: ToolProgressData
    }

    class SystemMessage {
        type: system
        content: string
    }

    class ToolUseSummaryMessage {
        type: tool_use_summary
        summary: string
        toolNames: string[]
    }

    class SystemCompactBoundary {
        type: system_compact_boundary
        summary: string
    }

    class TombstoneMessage {
        &lt;&lt;internal&gt;&gt;
        type: tombstone
        originalType: string
    }

    class SDKUserMessage {
        &lt;&lt;sdk&gt;&gt;
        type: sdk_user
        content: ContentBlock[]
    }

    class SDKAssistantMessage {
        &lt;&lt;sdk&gt;&gt;
        message: APIMessage
        usage: TokenUsage
    }

    class SDKPartialAssistantMessage {
        &lt;&lt;sdk-streaming&gt;&gt;
        partial content blocks
        incremental deltas
    }

    Message <|-- UserMessage
    Message <|-- AssistantMessage
    Message <|-- AttachmentMessage
    Message <|-- ProgressMessage
    Message <|-- SystemMessage
    Message <|-- ToolUseSummaryMessage
    Message <|-- SystemCompactBoundary
    Message <|-- TombstoneMessage
    Message <|-- SDKUserMessage
    Message <|-- SDKAssistantMessage
    Message <|-- SDKPartialAssistantMessage

    class TaskState {
        &lt;&lt;base&gt;&gt;
        id: string
        type: TaskType
        status: TaskStatus
        description: string
        startTime: number
        endTime: number
        outputFile: string
        outputOffset: number
        notified: boolean
    }

    class TaskType {
        &lt;&lt;enum&gt;&gt;
        local_bash
        local_agent
        remote_agent
        in_process_teammate
        local_workflow
        monitor_mcp
        dream
    }

    class TaskStatus {
        &lt;&lt;enum&gt;&gt;
        pending
        running
        completed
        failed
        killed
    }

    TaskState --> TaskType
    TaskState --> TaskStatus
```

---

## 13. Agent Communication System

```mermaid
graph TB
    subgraph AgentComm["Agent Communication System"]
        subgraph MailboxSystem["Mailbox (Queue + Async Waiters)"]
            MB_CLASS["Mailbox Instance<br/>per session"]
            SEND["send(message)<br/>Inject + wake waiters"]
            POLL["poll(filter)<br/>Non-blocking check"]
            RECEIVE["receive(filter)<br/>Blocking wait with timeout"]
            SUBSCRIBE["subscribe()<br/>Reactive notifications"]
        end

        subgraph MessageFormat["Message Format"]
            MSG_FIELDS["id: string<br/>source: user | teammate | system | tick | task<br/>content: string<br/>from: agent name<br/>color: display color<br/>timestamp: ISO string"]
        end

        subgraph TaskNotifications["Task Notification Protocol"]
            TASK_NOTIF["XML Task Notifications"]
            NOTIF_FIELDS["Fields:<br/>task-id, status, summary,<br/>result, usage, output-file,<br/>tool-use-id"]
        end

        subgraph NameRegistry["Named Agent Registry"]
            REG_MAP["Name-to-ID Map<br/>registered at spawn time"]
            LOOKUP["Name-based Addressing<br/>via Inter-Agent Messaging tool"]
        end

        subgraph MessageRouting["Message Routing Patterns"]
            DIRECT["Direct: agent ID targeting"]
            NAMED["Named: agent name lookup"]
            BROADCAST["Broadcast: task completion<br/>notification to coordinator"]
        end
    end

    SEND --> MB_CLASS
    MB_CLASS --> POLL
    MB_CLASS --> RECEIVE
    MB_CLASS --> SUBSCRIBE
    TASK_NOTIF --> SEND
    NOTIF_FIELDS --> TASK_NOTIF
    REG_MAP --> LOOKUP
    LOOKUP --> NAMED
    DIRECT --> MB_CLASS
    NAMED --> MB_CLASS
    BROADCAST --> MB_CLASS

    style AgentComm fill:#f0fff5,stroke:#00e676,stroke-width:3px,color:#333
    style MailboxSystem fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style MessageFormat fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style TaskNotifications fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style NameRegistry fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style MessageRouting fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style MB_CLASS fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style SEND fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
    style POLL fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
    style RECEIVE fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
    style SUBSCRIBE fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
    style TASK_NOTIF fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
    style REG_MAP fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
    style LOOKUP fill:#00e676,stroke:#00c853,stroke-width:1px,color:#000
```

---

## 14. Prompt Cache Sharing

```mermaid
graph TB
    subgraph CacheSharing["Prompt Cache Sharing Architecture"]
        PARENT["Parent Agent<br/>Establishes Cache Breakpoints"]

        subgraph CacheSafe["Cache-Safe Parameters (Frozen at Fork)"]
            SYS_PROMPT["System Prompt<br/>(largest segment, cached first)"]
            USER_CTX["User Context<br/>(instructions, memory files)"]
            SYS_CTX["System Context<br/>(environment, capabilities)"]
            TOOL_CTX["Tool Use Context<br/>(tool definitions, schemas)"]
            FORK_MSG["Fork Context Messages<br/>(conversation prefix)"]
        end

        subgraph CacheKey["API Cache Key Composition"]
            KEY["Cache Key =<br/>System Prompt +<br/>Tool Definitions +<br/>Model ID +<br/>Message Prefix +<br/>Thinking Config"]
        end

        subgraph Children["Child Agents (Inherit Cache)"]
            CHILD_A["Child A"]
            CHILD_B["Child B"]
            CHILD_C["Child C"]
            CHILD_N["Child N..."]
        end

        subgraph Isolation["Isolation Guarantees"]
            ISO_FILE["File State Cache: cloned"]
            ISO_DENIAL["Denial Tracking: fresh instance"]
            ISO_CONTENT["Content Replacement: cloned"]
            ISO_STATE["App State writes: no-op"]
            ISO_ABORT["Abort Controller: child of parent"]
        end

        subgraph Benefit["Benefits"]
            SAVINGS["Cost Reduction<br/>Significant cache hit<br/>on shared prefix"]
            LATENCY["TTFT Improvement<br/>skip re-processing cached tokens"]
            SCALE["Horizontal Scaling<br/>N agents share 1 cache entry"]
        end
    end

    PARENT --> CacheSafe
    CacheSafe --> KEY
    CacheSafe -->|"frozen, identical"| CHILD_A
    CacheSafe -->|"frozen, identical"| CHILD_B
    CacheSafe -->|"frozen, identical"| CHILD_C
    CacheSafe -->|"frozen, identical"| CHILD_N
    CHILD_A --> Isolation
    CHILD_A --> SAVINGS
    CHILD_B --> SAVINGS
    CHILD_C --> SAVINGS
    SAVINGS --> LATENCY
    LATENCY --> SCALE

    style CacheSharing fill:#f0fff5,stroke:#00e676,stroke-width:3px,color:#333
    style CacheSafe fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style CacheKey fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style Children fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style Isolation fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style Benefit fill:#f0fff5,stroke:#00e676,stroke-width:1px,color:#333
    style PARENT fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style SAVINGS fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style LATENCY fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
    style SCALE fill:#00e676,stroke:#00c853,stroke-width:2px,color:#000
```

---

## Legend

| Symbol | Meaning |
|--------|---------|
| ![](https://img.shields.io/badge/green_node-00e676?style=for-the-badge) | New or improved element |
| ![](https://img.shields.io/badge/green_section-f0fff5?style=for-the-badge&logoColor=00e676) | New section (mint fill, green border) |
| ![](https://img.shields.io/badge/core_engine-fff0f0?style=for-the-badge) | Core engine group (rose border) |
| ![](https://img.shields.io/badge/agent_system-f3f0ff?style=for-the-badge) | Agent system group (purple border) |
| ![](https://img.shields.io/badge/sync_/_UI-f0f5ff?style=for-the-badge) | UI / sync execution group (blue border) |
| ![](https://img.shields.io/badge/state_/_async-f0ffff?style=for-the-badge) | State / async execution group (teal border) |
| ![](https://img.shields.io/badge/cache-fffef0?style=for-the-badge) | Persistence / cache group (yellow border) |
| ⚑ | Feature-gated (conditional loading) |
| **Solid arrow** (→) | Direct dependency or call |
| **Dashed arrow** (⇢) | Async or event-based communication |

14 diagrams: 12 core architecture + 2 agent infrastructure additions.
