# Multi-Agent Orchestration Architecture

> Reverse-engineered architectural blueprint of a production-grade multi-agent AI system, distilled into conceptual patterns and 14 detailed Mermaid diagrams.

## What Is This?

This repository documents the **architecture and design patterns** behind a production multi-agent system, extracted through careful analysis of real source code. All application-specific references have been removed — what remains is a **reusable conceptual reference** for building multi-agent systems on any stack.

This is **not** a framework, SDK, or runnable code. It's an architectural study.

## Why This Matters

Most "multi-agent frameworks" are just agents calling agents in messy loops. This architecture represents something closer to **distributed systems engineering for AI**:

- Real concurrency control with in-process semaphores
- Dependency-aware task management with 7 task types
- Generator-based streaming for zero-blocking pipelines
- Prompt cache sharing across agent forks (significant token savings)
- 27 hook events for extensibility
- 7 MCP transports for tool integration
- 5 context compaction strategies
- Multi-worker orchestration with 4-phase coordination

## Documents

| File | Description |
|------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Full architectural description (English) |
| [claude-code-architecture.md](claude-code-architecture.md) | 14 Mermaid diagrams with visual architecture |
| [TASK.md](TASK.md) | Task documentation and findings log |
| [POST.md](POST.md) | Social media post drafts |

## Architecture Overview

```
CLI Entry
  -> Session Initializer
    -> Query Engine (session state machine)
      -> Conversation Turn Loop (async generator)
        -> Tool Orchestration (parallel/serial execution)
          -> Agent Spawner -> Forked Query Engine (recursive)
          -> Shell, File I/O, Web, MCP tools...
        -> Compaction Engine (5 strategies)
        -> Cost Tracking
    -> Central State Store (tasks, agents, MCP, plugins)
    -> Permission System (7 modes, triple validation)
    -> Plugin & Hook System (27 events)
    -> MCP Client (7 transports)
    -> Terminal UI (React-based renderer)
```

## Key Architectural Patterns

### 1. Generator-Based Streaming
The entire conversation pipeline is built on `AsyncGenerator`, yielding events in real-time. This enables incremental UI rendering, streaming tool execution, and progressive result delivery without blocking.

### 2. Prompt Cache Sharing
When spawning subagents, the system freezes **Cache-Safe Parameters** (system prompt, user context, tool definitions) and passes them identically to children. This allows the LLM API cache to be reused, saving significant tokens on the shared prefix.

### 3. In-Process Concurrency Control
Tool calls are partitioned by concurrency safety: read-only tools run in parallel (up to 10), destructive tools execute serially. No external message queues needed — just a semaphore and batch partitioning.

### 4. Coordinator Mode
A multi-worker orchestrator that operates in 4 phases:
1. **Research** — Parallel workers explore the codebase
2. **Synthesis** — Coordinator reads findings, forms a plan
3. **Implementation** — Workers execute the plan
4. **Verification** — Workers run tests and typechecks

Workers communicate via task notifications and can be continued (SendMessage) or killed (TaskStop).

### 5. 5 Agent Execution Paths
- **Synchronous** — Inline, blocking execution
- **Background** — Async task with streaming output
- **Git Worktree** — Isolated branch + working directory
- **Remote** — Execution in a cloud environment
- **Teammate** — Named agent in a shared team

### 6. 27 Hook Events
Extensibility through 8 categories of hooks: Lifecycle, Tool, Permission, Agent, Compaction, Input, System, and Worktree events.

### 7. 5 Compaction Strategies
Context window management through: History Snip, Microcompact, Auto-Compact, Context Collapse, and Session Memory Compact.

## Diagrams

The [diagram file](claude-code-architecture.md) contains 14 Mermaid diagrams:

1. **High-Level System Architecture** — All modules and connections
2. **Query Loop Lifecycle** — Conversation turn sequence
3. **Agent Spawning Flow** — 5 execution paths
4. **Coordinator Mode** — Multi-worker orchestration phases
5. **Tool Registry & Dispatch** — 50+ tools, execution pipeline
6. **Permission System** — 7 modes, rule priority flowchart
7. **State Management** — Central store, contexts, persistence
8. **MCP Integration** — 7 transports, tool marshaling
9. **Plugin & Hook System** — 27 events by category
10. **Compaction Strategies** — 5 context management approaches
11. **Bridge / Remote Architecture** — WebSocket remote execution
12. **Message Type Hierarchy** — Class diagram of all message types
13. **Agent Communication System** — Mailbox, notifications, routing
14. **Prompt Cache Sharing** — Cache optimization architecture

Green-highlighted elements indicate new discoveries or improvements found during verification.

## Verification Methodology

Every claim was verified against actual source code:

| Common Claim | Verified Reality |
|-------------|-----------------|
| "13 hook events" | **27** hook events across 8 categories |
| "3 compaction strategies" | **5** strategies (Context Collapse + Session Memory live in separate services) |
| "6 permission modes" | **7** modes (bubble — parent delegation — was missed) |
| "6 MCP transports" | **7** transports (SSE-IDE + Managed Proxy were missed) |
| "stop_reason drives the loop" | **Streaming detection** of tool_use blocks (stop_reason is explicitly unreliable) |
| "4 agent execution paths" | **5** paths (Teammate spawning was missed) |
| "~45 tools" | **~50+** tools |

## License

This is an architectural study for educational purposes. The patterns described are general software engineering concepts. No proprietary code is included.

## Contributing

Found an inaccuracy? Have additional insights? Open an issue or PR. The goal is to make this the most accurate public documentation of production multi-agent architecture patterns.
