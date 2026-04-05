# Architecture Evolution — Proposed Improvements

> Evolutionary improvements to the multi-agent orchestration architecture documented in [ARCHITECTURE.md](ARCHITECTURE.md). Each proposal builds on existing patterns — no rewrites, only extensions.

---

## Table of Contents

1. [Resilience Layer](#1-resilience-layer)
2. [Budget & Resource Management](#2-budget--resource-management)
3. [Coordinator Evolution](#3-coordinator-evolution)
4. [Communication Maturity](#4-communication-maturity)
5. [Observability Stack](#5-observability-stack)
6. [Adaptive Concurrency](#6-adaptive-concurrency)
7. [Agent Strategy Memory](#7-agent-strategy-memory)
8. [Security Hardening](#8-security-hardening)
9. [Cross-Cutting Concerns](#9-cross-cutting-concerns)

---

## 1. Resilience Layer

### Current State

The system has one recovery mechanism: `max_tokens` retry (up to 3 attempts with prefix accumulation). The abort controller cascades cancellation from parent to children. MCP Client has a Connection Manager (ARCHITECTURE.md, section 9), but its reconnection behavior is not specified beyond initial connection.

### Gaps

- **API failures** — No circuit breaker. If the API starts returning 5xx errors, every agent retries independently, amplifying load.
- **Worker crashes** — When a Coordinator worker fails mid-task, the Coordinator must manually reason about recovery. No structured retry or reassignment. Different task types (`dream` vs `local_agent`) have different failure semantics that aren't codified.
- **MCP degradation** — Connection Manager handles initial connection, but there is no documented strategy for graceful degradation when an established MCP server goes offline mid-session (tool state transition, backoff, recovery).
- **Partial output loss** — The system already writes streaming output to `outputFile` with `outputOffset` (ARCHITECTURE.md, section 6). However, failure notifications don't include a structured summary of what was completed vs. what remains.

### Proposed Improvements

**Circuit Breaker for API Calls**

```
States: CLOSED → OPEN → HALF_OPEN → CLOSED

CLOSED (normal):
  Track failure rate over sliding window (e.g., 10 calls)
  If failure rate > threshold (e.g., 50%) → transition to OPEN

OPEN (blocking):
  All calls fail immediately with CircuitOpenError
  After cooldown period (e.g., 30s) → transition to HALF_OPEN

HALF_OPEN (probing):
  Allow 1 probe call through
  If succeeds → CLOSED
  If fails → OPEN (reset cooldown)
```

Circuit breaker state is shared across all agents in a session — one breaker per API endpoint. This prevents thundering herd when the API is degraded.

**Agent Failure Recovery**

```
Worker fails → Coordinator receives failure notification
  → Classify failure by task type:

     local_agent / remote_agent:
       TRANSIENT (timeout, rate limit, OOM):
         → Retry same worker with SendMessage (preserves context)
         → Max 2 retries, then reassign to fresh worker
       PERSISTENT (invalid tool use, permission denied, logic error):
         → Spawn new worker with refined prompt
         → Include error context: "Previous attempt failed because: ..."
       FATAL (abort, budget exceeded):
         → Mark task as failed
         → Coordinator adjusts plan to work around missing result

     dream (speculative):
       ANY failure → mark as failed, no retry
       Dream results are optional — coordinator never blocks on them

     in_process_teammate:
       TRANSIENT → retry via SendMessage (shared process, low cost)
       PERSISTENT/FATAL → kill and respawn (same team context)
```

**MCP Graceful Degradation**

Extends the existing Connection Manager with state transitions:

```
MCP Server States (new):
  CONNECTED → DEGRADED → OFFLINE → RECONNECTING → CONNECTED

CONNECTED:
  Normal operation

CONNECTED → DEGRADED:
  Trigger: first failed call or heartbeat timeout
  Behavior: tools remain in registry but calls return fast error
  Emit hook: McpServerStateChanged(server, "degraded")

DEGRADED → OFFLINE:
  Trigger: 5 consecutive failed reconnection attempts
  Behavior: remove tools from pool
  Emit hook: McpServerStateChanged(server, "offline")

DEGRADED → RECONNECTING:
  Trigger: automatic, exponential backoff (1s, 2s, 4s, 8s, 16s, max 60s)
  On success: refresh tool list (schemas may have changed) → CONNECTED
  On failure: increment counter → stay DEGRADED or transition to OFFLINE
```

**Structured Failure Notifications**

Extends the existing `outputFile` + `outputOffset` mechanism (not a replacement):

```
Failure notification includes (new fields):
  <task-notification>
    <status>failed</status>
    <failure-type>TRANSIENT|PERSISTENT|FATAL</failure-type>
    <completed-steps>3 of 7 tool executions completed</completed-steps>
    <last-successful-output-offset>4096</last-successful-output-offset>
    <error-context>Rate limit exceeded after 3rd tool call</error-context>
  </task-notification>
```

Coordinator reads partial results via existing `TaskOutput` tool with `outputOffset`.

---

## 2. Budget & Resource Management

### Current State

Cost tracking records USD spent per assistant message. Usage metrics (input/output tokens) are accumulated per agent run. Analytics & Telemetry service exists (ARCHITECTURE.md, section 1). No enforcement — tracking is purely observational.

### Gaps

- No token budget per agent — a subagent with a broad prompt can consume unlimited context.
- No session-level cost ceiling — a Coordinator session with 10 workers has no spending limit.
- No priority-based resource allocation — all agents get the same model regardless of task criticality.
- Budget data is not connected to the existing Analytics & Telemetry pipeline.

### Proposed Improvements

**Hierarchical Budget Allocation**

Integrates with the existing Analytics & Telemetry service as a new budget dimension:

```
Session Budget
  ├── Coordinator Budget (allocated by session)
  │   ├── Worker A Budget (allocated by coordinator)
  │   ├── Worker B Budget (allocated by coordinator)
  │   └── Reserve Pool (unallocated, for retries/new workers)
  └── Background Agent Budgets (allocated by session)
```

Each budget has:
- `softLimit` — Warning threshold. Agent receives system message: "Approaching budget limit. Wrap up current task."
- `hardLimit` — Enforced ceiling. Turn loop exits with `budget_exceeded` status after current tool completes.
- `model` — Preferred model. Downgrade path when budget is low: opus → sonnet → haiku.

Budget events are emitted to Analytics & Telemetry:
```
budget_allocated, budget_warning, budget_exceeded, budget_returned
```

**Budget Inheritance Rules**

```
Parent spawns child:
  If parent specifies budget → child gets that budget
  If no budget specified:
    childBudget = max(
      minChildBudget,                          // Floor: 5K tokens (configurable)
      min(parent.remaining * 0.3, defaultChildBudget)
    )
  Child cannot exceed parent's remaining budget
  Unused child budget returns to parent's reserve pool on completion

Depth guard:
  At nesting depth > 3: warn coordinator, suggest flattening hierarchy
  At nesting depth > 5: refuse spawn, return error
```

The `minChildBudget` floor prevents exponential decay at deep nesting (0.3^3 = 2.7% without floor).

**Cost-Aware Model Routing**

```
Route model selection:
  effectiveCost = estimatedTokenCost * (1 - cacheHitRate)

  If effectiveCost / budget.remaining < 0.3 → use requested model
  If effectiveCost / budget.remaining 0.3-0.7 → downgrade one tier (opus → sonnet)
  If effectiveCost / budget.remaining > 0.7 → use cheapest model (haiku)
  If budget.remaining < minChildBudget → refuse new agent spawns, wrap up
```

Prompt Cache Sharing (ARCHITECTURE.md, section 13) significantly reduces real cost for subagents sharing the same prefix. The `cacheHitRate` factor prevents premature model downgrades when cache is effective.

---

## 3. Coordinator Evolution

### Current State

4 linear phases: Research → Synthesis → Implementation → Verification. Workers report via task notifications. Coordinator uses SendMessage to continue workers and TaskStop to kill them. Scratchpad provides unstructured file-based knowledge sharing.

### Gaps

- **No feedback loops** — If verification fails, the Coordinator must manually decide to re-implement. No structured protocol for iterative refinement.
- **No task dependencies** — All workers within a phase are independent. If Worker B needs Worker A's output, the Coordinator must manually sequence them.
- **Unstructured scratchpad** — No naming convention, no provenance tracking when multiple workers write to the same directory.
- **Fixed worker count** — Workers are spawned at phase start and don't scale dynamically.

### Proposed Improvements

**Verification Feedback Loop**

```
Phase 4 (Verification):
  Worker runs tests → reports results

  If ALL pass:
    → Coordinator marks session complete

  If SOME fail:
    → Coordinator analyzes failures
    → Classify each failure:
        IMPLEMENTATION_BUG: → SendMessage to original implementer with failure context
        MISSING_REQUIREMENT: → Spawn new worker for the gap
        TEST_ISSUE: → SendMessage to test worker to fix test
    → Re-run verification (max cycles configurable, default 3)

  If verification cycles exhausted OR remaining budget < threshold:
    → Coordinator reports partial success with failure summary
    → Human decides next steps
```

`maxVerificationCycles` is configurable and tied to budget: each cycle costs tokens, so the coordinator checks `budget.remaining > estimatedCycleCost` before retrying.

**Dependency-Aware Task Scheduling (Coordinator-scoped)**

Scope: only within a Coordinator session's workers. Does not affect global Task Management or other task types (dream, bash, etc.).

```
Coordinator-internal scheduling:
  TaskCreate:
    description: "Update API types"
    dependsOn: []  // no dependencies, can start immediately

  TaskCreate:
    description: "Implement endpoint handlers"
    dependsOn: ["task-id-types"]  // waits for types task to complete

  TaskCreate:
    description: "Write integration tests"
    dependsOn: ["task-id-types", "task-id-handlers"]  // waits for both

Scheduling logic:
  1. Build dependency graph from Coordinator's task declarations
  2. Identify tasks with no pending dependencies → spawn workers
  3. As tasks complete → check which blocked tasks are now unblocked → spawn
  4. Detect cycles at creation time → reject with error

Scope boundary:
  dependsOn[] only references tasks within the same Coordinator session.
  Cross-session or cross-type dependencies are not supported.
  No changes to the global Task State schema — dependsOn is a Coordinator-internal concept.
```

**Structured Scratchpad (Convention-based)**

Phase 1: Convention over configuration. No KV-store, no schema validation — just structured file naming:

```
Scratchpad file convention:
  {worker-name}--{topic}.md     // e.g., "worker-a--api-types.md"
  {worker-name}--{topic}.json   // For structured data

File header convention:
  ---
  author: worker-a
  timestamp: 2025-01-15T10:30:00Z
  topic: api-types
  ---

Workers write to their own namespace.
Coordinator reads all files when synthesizing.
```

Phase 2 (future): If convention proves insufficient, add `write(key, value)` / `read(key)` / `list()` operations with schema validation. Only after empirical evidence that convention-based approach has issues.

**Dynamic Worker Pool**

Constrained by circuit breaker state and budget:

```
Coordinator monitors worker progress:
  If workers completing faster than expected:
    → Reduce reserve budget allocation
  If workers slower than expected or blocked:
    → Check circuit breaker state (Section 1): if OPEN, do NOT spawn more workers
    → Check budget: must have > minChildBudget for new worker
    → Spawn additional workers for parallelizable subtasks
  If worker idle (waiting for dependency):
    → Reassign to independent task if available

Worker count bounded by:
  max_workers: 10 (configurable)
  available_budget: must have > minChildBudget per new worker
  circuit_breaker: must be CLOSED or HALF_OPEN
  active_tasks: no more workers than pending tasks
```

---

## 4. Communication Maturity

### Current State

Mailbox pattern with send/poll/receive/subscribe. Messages are typed by source (user, teammate, system, tick, task). Named Agent Registry enables routing by name. Task notifications use XML format. The `receive(filter)` method already supports predicate-based selective retrieval, providing de-facto message filtering. The architecture is fully in-process (ARCHITECTURE.md, section 1) — all communication is in-memory by design.

### Gaps

- **No request-response** — Coordinator sends a message via SendMessage but has no structured way to correlate the response. Must rely on the model interpreting task notifications.
- **No structured triage** — While `receive(filter)` allows predicate-based selection, there is no protocol for the Coordinator to systematically triage messages by urgency. Critical failures and routine updates arrive through the same path.

### Proposed Improvements

**Request-Response Overlay**

```
Coordinator sends:
  SendMessage(to: "worker-A", content: "...", correlationId: "req-001")

Worker-A responds (via task notification):
  <task-notification>
    <correlation-id>req-001</correlation-id>
    <status>completed</status>
    <result>...</result>
  </task-notification>

Coordinator awaits:
  receive(filter: msg => msg.correlationId === "req-001", timeout: 60s)
```

This adds one optional field (`correlationId`) to the existing message format. Fully backward-compatible.

**Structured Triage Protocol**

Instead of adding a priority field to messages (which adds complexity), define a triage protocol the Coordinator follows:

```
Coordinator triage loop:
  1. receive(filter: isFailureNotification, timeout: 0)   // Non-blocking: check failures first
  2. receive(filter: isCompletionNotification, timeout: 0) // Then: completed tasks
  3. receive(filter: isProgressUpdate, timeout: 0)          // Then: progress reports
  4. If no messages: receive(timeout: pollInterval)          // Block until next message

This uses existing receive(filter) capability — no Mailbox changes needed.
Just a documented pattern for Coordinator system prompts.
```

Note: Back-pressure is not needed at current scale (max 10 workers, 1-3 notifications per cycle). If Dynamic Worker Pool (section 3) raises the limit to 20+, back-pressure should be reconsidered.

---

## 5. Observability Stack

### Current State

Analytics & Telemetry service exists. Session transcripts record full conversation in conversation format. Cost tracking per message. Usage metrics (input/output tokens) per agent run. Hook system fires 27 events (including PermissionRequest and PermissionDenied). No structured queryable logging or distributed tracing.

### Gaps

- **No distributed tracing** — When a Coordinator spawns 5 workers, each running 10 tool calls, there's no way to trace a specific operation across the hierarchy.
- **Transcripts aren't queryable** — Session transcripts exist but are conversation-format, not structured events. You can replay a conversation but not answer "which tool calls took >5s?"
- **No performance baselines** — Tool execution latency, agent completion time, cache hit rates aren't tracked in a way that enables trend analysis or anomaly detection.

### Proposed Improvements

**Distributed Tracing**

```
Trace Structure:
  traceId: string     // One per top-level user request
  spanId: string      // One per operation (agent run, tool call, API call)
  parentSpanId: string // Links to parent operation

Span Types:
  SESSION    — Top-level user interaction
  TURN       — Single conversation turn
  AGENT      — Subagent execution
  TOOL       — Tool execution
  API_CALL   — LLM API request
  MCP_CALL   — MCP tool invocation
  HOOK       — Hook execution

Each span records:
  startTime, endTime, status
  attributes: { toolName, agentType, model, tokenCount, costUSD }
  events: [ { name, timestamp, attributes } ]
```

Integration with Context Cloning: `createSubagentContext()` (ARCHITECTURE.md, section 4) is extended to include tracing context in its cloned fields:

```
createSubagentContext() clones (existing):
  - File State Cache
  - Content Replacement State
  - Denial Tracking (fresh)
  - Rendered System Prompt (frozen)
  - App State Writes (no-op)
  - Abort Controller (child of parent)

createSubagentContext() adds (new):
  - Trace Context: { traceId (inherited), parentSpanId (parent's spanId) }
  - New spanId generated for the child agent
```

**Extended Session Transcripts**

Instead of a separate NDJSON event log (which would create a second parallel logging system), extend existing session transcripts with structured event annotations:

```
Existing transcript entry:
  { type: "assistant", message: ..., costUSD: 0.003, durationMs: 1200 }

Extended transcript entry (backward-compatible):
  { type: "assistant", message: ..., costUSD: 0.003, durationMs: 1200,
    trace: { traceId: "t-001", spanId: "s-005", parentSpanId: "s-001" },
    events: [
      { event: "tool_start", tool: "Grep", timestamp: "...", spanId: "s-006" },
      { event: "tool_end", tool: "Grep", durationMs: 45, spanId: "s-006" }
    ]
  }
```

The hook system (27 existing events) can forward these structured events to external collectors without adding a new log sink.

**Performance Metrics with Baselines**

Collect per tool type, per agent type:
```
Metrics:
  tool_execution_duration_ms  — histogram (p50, p95, p99) by tool name
  tool_call_count             — counter by tool name
  tool_error_rate             — ratio of failed / total
  cache_hit_rate              — prompt cache hits / total API calls
  agent_completion_time_ms    — histogram by agent type
  budget_utilization          — ratio of spent / allocated
  compaction_frequency        — compactions per N turns

Baseline establishment:
  First 100 operations per metric → establish baseline
  Subsequent operations → compare against rolling p95
  Alert if metric exceeds 2x baseline for 5 consecutive samples

Metrics are emitted through existing Analytics & Telemetry service.
```

---

## 6. Adaptive Concurrency

### Current State

Fixed semaphore: max 10 concurrent read-only tools, destructive tools run serially. The `isConcurrencySafe(input)` method is a function (not a property), allowing input-dependent decisions.

### Gaps

- **Static limit** — 10 is hardcoded. Optimal concurrency depends on API rate limits, system resources, and tool types.
- **No feedback loop** — Execution metrics don't influence future concurrency decisions.
- **No tool profiling** — All concurrent-safe tools are treated equally. A fast Glob and a slow WebFetch both count as 1 slot.

### Proposed Improvements

**Dynamic Semaphore**

```
Adaptive Concurrency Controller:

  initialConcurrency: 10
  minConcurrency: 2
  maxConcurrency: 20

  Adjustment algorithm (per batch):
    Measure: average tool latency in last batch
    If latency decreased and no errors:
      → Increase concurrency by 1 (up to max)
    If latency increased or error rate > 0:
      → Decrease concurrency by 2 (down to min)
    If rate limit detected (HTTP 429 or equivalent):
      → Check source:
         API endpoint 429 → decrease by 2 (API-wide throttle)
         MCP server 429 → decrease MCP-specific concurrency only
         Single tool failure → no global change, mark tool as slow
      → Gradual recovery over next 5 batches
```

**Latency-Adaptive Slot Weights**

Instead of hardcoded weights, derive slot costs from observed latency:

```
Initial weights (cold start, before profiling data):
  All tools: weight 1 (same as current behavior)

After profiling data (10+ calls per tool type):
  weight = ceil(observed_p95_ms / baseline_p95_ms)
  Where baseline = p95 of fastest tool category (typically Glob/Grep)

Example (after profiling):
  Glob: p95=15ms  → weight 1 (15/15)
  Grep: p95=20ms  → weight 1 (20/15, ceil)
  FileRead: p95=25ms → weight 2 (25/15, ceil)
  WebFetch: p95=800ms → weight 54... capped at max_weight=5
  MCP tool (stdio): p95=50ms → weight 4 (50/15, ceil)
  MCP tool (SSE remote): p95=300ms → weight 5 (capped)

max_weight: 5 (cap to prevent single tool from monopolizing)
Weights update every 50 calls (rolling window).
```

---

## 7. Agent Strategy Memory

### Current State

Session Memory Compact (ARCHITECTURE.md, section 11) extracts durable memories for cross-session knowledge. Memory files store user preferences, project context, and feedback. This is the only cross-session persistence mechanism.

### Gaps

- **No execution strategy learning** — The system doesn't learn which agent configurations (model, worker count, phase structure) work best for which task types.
- **No plan reuse** — A Coordinator builds a plan from scratch every time, even for recurring task patterns.

### Proposed Improvements

**Execution Pattern Index (Extension of Session Memory Compact)**

This is not a new mechanism — it's a new memory type extracted by the existing Session Memory Compact strategy. After each Coordinator session, the compaction process extracts:

```
{
  taskPattern: string,        // Classified: "refactor", "feature", "bugfix", "test"
  filePatterns: string[],     // Glob patterns of affected files (no full paths)
  workerCount: number,
  totalTokens: number,
  totalDuration: number,
  success: boolean,
  failureReason?: string
}
```

No `planSummary` field — plans often contain file paths, API endpoints, and business logic that are hard to anonymize reliably. The structured fields above provide sufficient signal without privacy risk.

Store in a per-project memory file (standard memory format). On new Coordinator session, the system prompt includes:
```
Previous similar tasks in this project:
- "refactor": 3 workers, 45K tokens, 12 min, succeeded
- "feature": 2 workers, 28K tokens, 8 min, succeeded
- "bugfix": 1 worker, 15K tokens, 5 min, failed → retried with 2 workers, succeeded
```

**Per-Pattern Agent Scoring**

Score is computed per task pattern, not globally:

```
For each (agentType, taskPattern) pair, track:
  successRate: number         // 0.0-1.0
  avgTokensPerTask: number
  avgDurationMs: number
  sampleCount: number         // Minimum 5 samples before score is used

Score = successRate * 0.6 + (1 / normalizedCost) * 0.2 + (1 / normalizedDuration) * 0.2

Example:
  ("Explore agent", "research") → score 0.85 (high success, fast)
  ("Explore agent", "implementation") → score 0.40 (often fails, not designed for this)
  ("developer agent", "implementation") → score 0.90 (designed for this)
```

The Agent Spawner uses scores as suggestions, not overrides — the model makes the final decision.

---

## 8. Security Hardening

### Current State

Triple permission validation. 7 modes with 8 rule sources. Bubble mode for parent delegation. Tool-specific permission checks. Session transcripts record full conversation history. Hook system fires PermissionRequest and PermissionDenied events. No structured, queryable audit format. No per-agent resource limits.

### Gaps

- **No structured audit log** — Session transcripts and hook events capture permission decisions, but in conversation format. Cannot easily answer "which tools were approved via bypass mode in the last 24 hours?"
- **No resource quotas** — An agent can read unlimited files, make unlimited network calls, consume unlimited disk.
- **No automated sensitive data detection** — Data flows freely between agents, to scratchpad, and to remote execution without classification.

### Proposed Improvements

**Structured Permission Audit Log**

Extends existing PermissionRequest/PermissionDenied hook events into a queryable format:

```
Every permission decision emits (via hook system):

{
  timestamp: ISO string,
  traceId: string,
  agentId: string,
  toolName: string,
  toolInput: { ... sanitized },   // Redact values matching secret patterns
  decision: "allow" | "deny" | "ask",
  reason: string,                  // Which rule matched, which mode
  ruleSource: string,              // "session", "projectSettings", etc.
  userAction?: "allowOnce" | "allowAlways" | "deny"
}

New hook event: PermissionAudit (category: System)
Written as structured field in session transcript (not a separate file).
External forwarding via hook system to SIEM if configured.
```

**Task-Type-Aware Resource Quotas**

Instead of fixed limits for all agents, define quota profiles by task type:

```
Quota Profiles:

  research (Coordinator Phase 1):
    maxFilesRead: 500           // Needs broad codebase access
    maxFilesWritten: 0          // Read-only phase
    maxNetworkCalls: 20
    maxBashCommands: 10         // git log, find, etc.

  implementation (Coordinator Phase 3):
    maxFilesRead: 200
    maxFilesWritten: 50
    maxNetworkCalls: 10
    maxBashCommands: 50         // Build, test commands

  verification (Coordinator Phase 4):
    maxFilesRead: 100
    maxFilesWritten: 5          // Only test result files
    maxNetworkCalls: 5
    maxBashCommands: 30

  background_agent (standalone):
    maxFilesRead: 100
    maxFilesWritten: 20
    maxNetworkCalls: 50
    maxBashCommands: 30

  dream (speculative):
    maxFilesRead: 50
    maxFilesWritten: 0          // Dreams don't write
    maxNetworkCalls: 10
    maxBashCommands: 5

Enforcement:
  Each tool execution increments the relevant counter
  softLimit (80%): system message warning
  hardLimit (100%): tool returns error, agent wraps up
  Coordinator can override defaults for specific workers
```

**Automated Sensitive Data Detection**

Tagging without auto-detection is useless (nobody will manually tag messages). Auto-detection strategy:

```
Detection layers:

1. Pattern-based (fast, low false-positive):
   - API keys: /[A-Za-z0-9_-]{32,}/ in tool results
   - Tokens: /Bearer [A-Za-z0-9_.-]+/
   - Connection strings: /postgres:\/\/|mysql:\/\/|mongodb:\/\//
   - AWS keys: /AKIA[0-9A-Z]{16}/
   - Private keys: /-----BEGIN (RSA |EC )?PRIVATE KEY-----/

2. Context-based (hook-driven):
   - Files matching: .env, .env.*, credentials.*, *secret*, *.pem, *.key
   - Tool results from: password managers, vault APIs, cloud IAM

3. Sensitivity tags (auto-applied):
   - Pattern match → tag as "restricted"
   - .env file read → tag as "restricted"
   - Default → "internal"

Restricted content:
  - Redacted in audit log entries
  - Warning if passed to WebFetch, external MCP servers, or remote agents
  - Not written to scratchpad
```

Detection runs as a PostToolUse hook — no core engine changes needed.

---

## 9. Cross-Cutting Concerns

### 9.1. Compaction Awareness

New mechanisms (Budget, Tracing) must survive compaction events.

```
After any compaction strategy runs (ARCHITECTURE.md, section 11):

Preserved (added to post-compact restoration):
  - Budget state (remaining, soft/hard limits) → always restored
  - Trace context (traceId, spanId) → always restored
  - Resource quota counters → always restored

Not preserved (acceptable loss):
  - Individual span events → written to transcript before compaction
  - Performance metric samples → aggregated into p50/p95 before compaction

Hook events PreCompact and PostCompact (already exist) can be used to
snapshot and restore these states.
```

### 9.2. Hook System Extensions

Proposed improvements introduce new hook events. These integrate into the existing 27-event registry (ARCHITECTURE.md, section 10):

```
New Hook Events:

  Category: System (currently 5 events → 8)
    McpServerStateChanged  — MCP server state transition (Section 1)
    PermissionAudit        — Structured permission decision (Section 8)
    BudgetWarning          — Agent approaching budget limit (Section 2)

  Category: Agent (currently 5 events → 6)
    VerificationCycleStart — Coordinator starts a verification retry (Section 3)

Total: 27 → 31 hook events
```

### 9.3. Migration Path

Each improvement can be adopted incrementally:

```
Budget:
  1. Add budget fields to Agent input schema (optional, no breaking change)
  2. Default: no budget (unlimited, same as current behavior)
  3. Enable via session config: { budgetEnabled: true, sessionBudget: 100000 }
  4. Existing sessions without budget config behave identically to today

Adaptive Concurrency:
  1. Replace hardcoded semaphore(10) with adaptiveSemaphore(initial=10)
  2. Cold start behavior is identical to current (10 slots, all weight 1)
  3. After 10+ calls, weights begin adjusting
  4. Config flag to force fixed mode: { adaptiveConcurrency: false }

Tracing:
  1. Add trace fields to createSubagentContext() (optional, null if tracing off)
  2. Add trace/events fields to transcript entries (backward-compatible extension)
  3. Enable via config: { tracingEnabled: true }
  4. Disabled by default — zero overhead when off

Resource Quotas:
  1. Add optional quota profile to Agent input schema
  2. Default: no quotas (unlimited, same as current behavior)
  3. Coordinator opt-in: specify profile per worker at spawn time
  4. Config for defaults: { defaultQuotaProfile: "background_agent" }
```

---

## Summary

| # | Area | Current | Proposed | Impact | Affected Diagrams |
|---|------|---------|----------|--------|-------------------|
| 1 | Resilience | max_tokens retry only | Circuit breaker, agent recovery, MCP degradation | High | 2 (Query Loop), 8 (MCP) |
| 2 | Budget | Passive cost tracking | Hierarchical budgets, cache-aware routing | High | 1 (Overview), 3 (Agent Spawning) |
| 3 | Coordinator | 4 linear phases | Feedback loops, scoped task DAG, dynamic scaling | High | 4 (Coordinator) |
| 4 | Communication | Mailbox with filter | Request-response overlay, triage protocol | Medium | 13 (Agent Communication) |
| 5 | Observability | Session transcripts | Distributed tracing, extended transcripts, metrics | Medium | 1 (Overview), 7 (State) |
| 6 | Concurrency | Fixed semaphore (10) | Adaptive sizing, latency-based weights | Medium | 5 (Tool Registry) |
| 7 | Strategy Memory | Session Memory Compact | Execution pattern index, per-pattern scoring | Low-Medium | 10 (Compaction) |
| 8 | Security | Triple permission check | Structured audit, quota profiles, auto-detection | Medium | 6 (Permissions), 9 (Hooks) |
| 9 | Cross-cutting | — | Compaction awareness, 4 new hooks, migration paths | — | 9 (Hooks), 10 (Compaction) |

### Implementation Priority

**Phase 1 — Foundations** (highest value, enables debugging of later phases):
- Budget & Resource Management
- Resilience Layer
- Structured Event Log (from Observability — prerequisite for debugging Budget and Resilience)

**Phase 2 — Coordination** (high value, moderate complexity):
- Coordinator feedback loops (with configurable cycle count)
- Dependency-aware task scheduling (Coordinator-scoped)
- Communication request-response overlay
- Structured Scratchpad (convention-based, no KV store)

**Phase 3 — Observability** (full tracing, metrics):
- Distributed tracing (trace context in Context Cloning)
- Performance metrics with baselines
- Permission audit log

**Phase 4 — Optimization** (refinements, requires empirical data from earlier phases):
- Adaptive concurrency (needs metrics from Phase 3)
- Agent strategy memory (needs execution data from Phase 1-2)
- Dynamic worker pool (needs Circuit Breaker + Budget from Phase 1)
- Latency-adaptive slot weights (needs profiling data from Phase 3)
