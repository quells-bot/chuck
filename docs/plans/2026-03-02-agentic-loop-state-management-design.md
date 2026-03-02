# Agentic Loop State Management Design

**Date:** 2026-03-02
**Status:** Approved
**Builds on:** [2026-02-08 Agentic Loop Temporal Brainstorm](2026-02-08-agentic-loop-temporal-brainstorm.md)

## Problem

State management for the agentic loop within Temporal, minimizing workflow history size while maintaining full auditability. Temporal Cloud has stricter history size limits and higher per-event costs than self-hosted, so conversation content must live outside the workflow.

## Core Principle

The workflow is a thin orchestrator that passes S3 keys between activities. Heavy state (full conversation history, tool results, LLM payloads) lives in S3. User-visible data (messages, progress, cost) lives in DynamoDB. The workflow itself holds only a handful of scalars.

## LLM Integration

The LLM layer is a vendored copy of `github.com/quells-bot/unified-llm`, which wraps AWS Bedrock Converse with an immutable `Conversation` struct, tool calling support, and middleware. The `Conversation` is the unit of serialization — every S3 snapshot is a full compressed `Conversation`.

## S3 State Model

### Key Structure

```
conversations/{conversation_id}/
  snapshots/{ulid}.json.zst       # one per LLM call
  tool-results/{ulid}.json        # one per tool execution
  compactions/{ulid}.json.zst     # one per compaction run
```

All identifiers are ULIDs — lexicographically sortable, self-documenting timestamps, no coordination needed.

### What Gets Stored

| Object | Contents | Written By | Read By |
|--------|----------|------------|---------|
| `snapshots/{ulid}.json.zst` | Full compressed `llm.Conversation` after an LLM call | `AgentIteration` activity | `AgentIteration` (next iteration or next turn) |
| `tool-results/{ulid}.json` | Raw tool result JSON (tool call ID, content, is_error) | `ExecuteTool` activity | `AgentIteration` (next iteration) |
| `compactions/{ulid}.json.zst` | Full compressed `llm.Conversation` with middle turns summarized | `Compact` activity | `AgentIteration` (next turn after compaction) |

### Lifecycle

- **New conversation:** No snapshots exist. The first `AgentIteration` sees an empty snapshot key, creates a fresh `llm.Conversation` from config (system prompt, tool definitions, model ID), appends the user message, calls the LLM, and writes the first snapshot.
- **Within a turn:** Each `AgentIteration` loads the latest snapshot, appends tool results from the previous iteration (if any), calls the LLM, and writes a new snapshot.
- **Between turns:** The next turn's first `AgentIteration` loads the latest snapshot from the previous turn and appends the new user message.
- **After compaction:** `Compact` reads the latest snapshot, summarizes middle turns via a cheaper LLM, writes a new snapshot under `compactions/`. The workflow updates its pointer. Old snapshots remain for audit.

### How the Workflow Knows What to Load

The workflow state holds a single string — the S3 key of the latest Conversation snapshot. This gets updated after each `AgentIteration` or `Compact` activity. It can point to a snapshot or a compaction — the workflow doesn't distinguish between them.

## Workflow State

```go
type ConversationWorkflowState struct {
    ConversationID  string
    UserID          string
    LatestSnapshot  string      // S3 key; empty for brand new conversation
    Config          AgentConfig // system prompt, tool defs, model ID
}
```

No token tracking, no conversation content, no message history. Kilobytes at most.

## Activity Contracts

### AgentIteration

Transforms a Conversation through one LLM call.

```
Input:
  ConversationID  string
  SnapshotKey     string          // S3 key of latest Conversation; empty = new conversation
  ToolResultKeys  []string        // S3 keys of tool results to append; empty on first iteration
  UserMessage     string          // non-empty only on first iteration of a turn
  Config          AgentConfig     // system prompt, tool defs, model ID
  ForceFinal      bool            // if true, strip tools to force a text-only response

Output:
  SnapshotKey     string          // S3 key of the new snapshot
  Status          IterationStatus // "done" or "needs_tools"
  Response        string          // final text response (if done)
  ToolCalls       []ToolCallSpec  // tool name, ID, args (if needs_tools)
  CostEstimate    float64         // estimated cost of this LLM call in USD
```

**Internal flow:**
1. Load Conversation from S3 (or create fresh from Config)
2. Append tool results from `ToolResultKeys` (load each from S3)
3. Append user message (first iteration only)
4. If `ForceFinal`, strip tools from Conversation
5. Call LLM via `unified-llm` `client.Send()`
6. Compress and write new snapshot to S3
7. Parse response: extract tool calls or final text

### ExecuteTool

Executes a single tool and persists its result.

```
Input:
  ConversationID  string
  ToolCallID      string          // from the LLM's tool_use block
  ToolName        string
  Arguments       json.RawMessage

Output:
  ResultKey       string          // S3 key where the tool result was written
  IsError         bool            // whether the tool returned an error
  CostEstimate    float64         // estimated cost of this tool execution in USD
```

Each tool type has its own Temporal activity timeout and retry policy. Tool errors (no results, bad input, runtime exceptions) are successful activity completions that return error results — the LLM sees the error on the next iteration and adapts.

### PublishProgress

Writes tool progress to DDB so users can see what's happening.

```
Input:
  ConversationID  string
  ToolName        string
  Arguments       json.RawMessage
  MessageID       string          // ULID
  SnapshotKey     string          // S3 key for audit trail

Output:
  (none meaningful)
```

### PublishResponse

Writes the final assistant message and updates conversation metadata.

```
Input:
  ConversationID  string
  UserID          string
  Response        string
  MessageID       string          // ULID
  SnapshotKey     string          // S3 key for audit trail
  TurnCost        float64         // total cost for this turn

Output:
  (none meaningful)
```

Atomically updates the conversation's `total_cost_estimate` and `updated_at` in the Conversations table.

### Compact

Summarizes middle turns to reduce conversation size.

```
Input:
  ConversationID  string
  SnapshotKey     string          // latest full snapshot to compact
  Config          AgentConfig     // needs model ID for summarization LLM call

Output:
  SnapshotKey     string          // S3 key of the compacted snapshot
  CostEstimate    float64
```

Loads the full Conversation, identifies middle turns to summarize, calls a cheaper/faster LLM to produce summaries, replaces middle turns with summaries, writes new compacted snapshot. Original snapshots remain for audit.

## Workflow Loop

```
workflow ConversationWorkflow(state, initialMessage):
    maxIterations = 10  // hard-coded, adjustable

    // First turn: process the message that started the workflow
    if initialMessage != "":
        processTurn(state, initialMessage, maxIterations)

    // Subsequent turns: block on user message signals
    for each userMessage from signal channel:
        processTurn(state, userMessage, maxIterations)

        if shouldContinueAsNew(state):
            continueAsNew(state, "")  // empty initialMessage

function processTurn(state, userMessage, maxIterations):
    toolResultKeys = []
    turnCost = 0.0

    for iteration := 0; iteration < maxIterations; iteration++:
        forceFinal = (iteration == maxIterations - 1)

        result = activity AgentIteration(
            conversationID = state.ConversationID,
            snapshotKey    = state.LatestSnapshot,
            toolResultKeys = toolResultKeys,
            userMessage    = userMessage if iteration == 0 else "",
            config         = state.Config,
            forceFinal     = forceFinal,
        )
        state.LatestSnapshot = result.SnapshotKey
        turnCost += result.CostEstimate

        if result.Status == "done":
            activity PublishResponse(
                conversationID = state.ConversationID,
                userID         = state.UserID,
                response       = result.Response,
                messageID      = newULID(),
                snapshotKey    = result.SnapshotKey,
                turnCost       = turnCost,
            )
            return

        // Execute tools sequentially
        toolResultKeys = []
        for each tc in result.ToolCalls:
            activity PublishProgress(
                conversationID = state.ConversationID,
                toolName       = tc.Name,
                arguments      = tc.Arguments,
                messageID      = newULID(),
                snapshotKey    = result.SnapshotKey,
            )
            toolResult = activity ExecuteTool(
                conversationID = state.ConversationID,
                toolCallID     = tc.ID,
                toolName       = tc.Name,
                arguments      = tc.Arguments,
            )
            toolResultKeys = append(toolResultKeys, toolResult.ResultKey)
            turnCost += toolResult.CostEstimate

    // If we exit the loop, the last iteration had forceFinal=true
    // so the LLM produced a text response, published above.

    if shouldCompact(state):
        compactResult = activity Compact(
            conversationID = state.ConversationID,
            snapshotKey    = state.LatestSnapshot,
            config         = state.Config,
        )
        state.LatestSnapshot = compactResult.SnapshotKey
```

### Key Workflow Details

- **First message** arrives as a workflow input parameter, not a signal. This is what starts the workflow.
- **Continue-as-new** passes `state` with an empty `initialMessage`, so the continued workflow goes straight to the signal loop. This sheds workflow history.
- **Sequential tool execution** — no parallelism yet. Each tool runs as its own activity with independent timeout and retry.
- **Iteration limit** is a hard cap to prevent runaway tool-calling loops. On the last allowed iteration, `forceFinal=true` strips tools so the LLM must respond with text.

## DynamoDB Data Model

### Conversations Table

```
PK: user_id (string)
SK: conversation_id (ULID)

Attributes:
  status:              string  // "active", "waiting", "error", "closed"
  title:               string  // user-provided or LLM-generated
  total_cost_estimate:  float   // cumulative USD cost, updated at end of each turn
  created_at:          string  // ISO 8601
  updated_at:          string  // ISO 8601
  config:              map     // system prompt ref, model ID, tool config
```

### Messages Table

```
PK: conversation_id (ULID)
SK: message_id (ULID)

Attributes:
  role:           string  // "user", "assistant", "tool_progress"
  content:        string  // message text, or tool name + args for progress
  snapshot_key:   string  // S3 key of the Conversation snapshot that produced this message
  cost_estimate:  float   // incremental USD cost of this specific step
  created_at:     string  // ISO 8601
```

### Who Writes What

| Writer | Table | When |
|--------|-------|------|
| API layer | Messages (user role) | When user sends a message |
| `PublishProgress` activity | Messages (tool_progress role) | After each tool call is requested |
| `PublishResponse` activity | Messages (assistant role) + Conversations (metadata) | After final response |

The workflow never reads from DDB. It is a write-only sink from the workflow's perspective.

### Cost Tracking

- **Per-step:** Each message has a `cost_estimate` attribute — the incremental cost of that specific LLM call or tool execution.
- **Per-conversation:** The Conversations table has a `total_cost_estimate` attribute, atomically updated by `PublishResponse` at the end of each turn.

## Error Handling

### Activity-Level Policies

| Activity | Timeout | Retries | On Failure |
|----------|---------|---------|------------|
| `AgentIteration` | 120s | 1 | Publish error response to user, end turn |
| `ExecuteTool` | Per-tool (10-30s) | Per-tool (0-3) | Return error result; LLM adapts on next iteration |
| `PublishProgress` | 5s | 3 | Non-critical; workflow continues |
| `PublishResponse` | 5s | 3 | Critical; aggressive retry |
| `Compact` | 120s | 1 | Non-critical; deferred to next turn |

### Key Principles

- **Tool errors are not activity errors.** A search returning no results or code eval throwing an exception is a successful tool execution that returns an error result. The LLM sees it and adapts.
- **LLM call retries are safe.** If the LLM call succeeds but the S3 write fails, the activity retries from scratch. The `Conversation` is immutable, so re-calling the LLM is safe (the response may differ, but nothing is inconsistent).
- **Workflow replay is fast.** Since all heavy data is in S3 and workflow state is small, Temporal replay after a crash completes quickly.

## Compaction

Compaction is a separate activity that runs at the end of a turn when the Conversation snapshot exceeds a size or turn-count threshold.

- Loads the full Conversation from S3
- Keeps the system prompt, first user message, and last N turns intact
- Summarizes middle turns using a cheaper/faster LLM
- Writes the compacted Conversation as a new snapshot under `compactions/`
- Old snapshots remain in S3 for audit and debugging
- The workflow updates its `LatestSnapshot` pointer to the compacted version
