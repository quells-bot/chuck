# Agentic Loop Design — Data Room Search

**Date:** 2026-02-08
**Status:** Draft

## Overview

A conversational agent platform built in Go with Temporal as the orchestration layer. The initial use case is accelerating data room search across hundreds of documents spanning decades and multiple topics. The framework is designed to be domain-agnostic for future reuse.

Users start a conversation via a JSON/HTTP API. The agent reasons over the query using an LLM, invokes tools (semantic search, full-text search, code evaluation, memory, skills), and can spawn subagents to keep context under control. Responses are polled asynchronously.

## Architecture

### Request Flow

1. Client sends a message to a stateless API server.
1. API server starts a Temporal workflow (new conversation) or signals an existing one (follow-up message). Returns the conversation ID.
1. Client polls the API server for new messages. The API reads from DynamoDB.
1. The Temporal workflow runs the agentic loop internally: build context → call LLM → execute tools → repeat until done → persist response to DDB.

### Key Components

- **API Server** — Stateless HTTP service. Handles auth, starts/signals workflows, serves poll responses from DynamoDB.
- **Conversation Workflow** — One long-running Temporal workflow per conversation. Receives user messages via signals, runs the agent loop.
- **Agent Loop Activity** — Assembles context, calls the LLM, returns tool requests or a final response. Retryable as a Temporal activity.
- **Tool Executor Activities** — Each tool call is a separate Temporal activity, enabling parallel execution and independent retry/timeout policies.
- **Subagent Child Workflows** — Child workflows with their own agent loop, constrained to one level of nesting. Parent blocks until child completes.

## Data Model & Persistence

### DynamoDB Tables

**Conversations** — `PK: user_id, SK: conversation_id`

Conversation metadata: status (active/complete/failed), created/updated timestamps, title, model configuration, cumulative token counts (input/output), cumulative tool call count. Listing a user’s conversations is a simple partition key query.

**Messages** — `PK: conversation_id, SK: sequence_number`

User-visible messages only: role (user/assistant), content, timestamp, optional tool call summary (names and brief descriptions for UI display, not full payloads). Querying by conversation ID returns ordered history.

**Skills** — `PK: "skills", SK: skill_id`

System-level skill definitions: name, description (used by the LLM to decide when to load a skill), content (the full prompt text), version, active/inactive flag. Managed via admin API. Not tied to individual users or data rooms.

### S3

**Conversation internals bucket** — keyed by `conversations/{conversation_id}/turns/{turn_number}/`

Per-turn artifacts: full LLM request/response payloads, raw tool results, assembled context snapshots, token counts, latency, model used. This is debug and replay data, lazy-loaded only when reconstructing context for the next LLM call or for post-hoc analysis.

### The Split

DynamoDB stores everything the user should see. S3 stores everything internal. The poll endpoint only ever reads DDB. Turn internals go to S3 asynchronously after each turn completes.

## Agentic Loop

On each user message (received via Temporal signal):

1. **Build context** — Load conversation history from DDB Messages. Load relevant prior tool results from S3 if needed. Prepend system prompt with skill directory (names + descriptions of all active skills).
1. **Call LLM (activity)** — Send assembled context to the primary model. Response is either a final text answer or one or more tool call requests. Timeout: 2–5 minutes depending on model. Retry: limited retries with backoff.
1. **Execute tools (parallel activities)** — Each requested tool call runs as a separate Temporal activity, concurrently. Results are collected when all complete.
1. **Budget check** — Track tool calls and LLM iterations against per-turn limits. If budget is exhausted, strip tool definitions from the next LLM call to force a final response.
1. **Loop or finish** — If the model returned tool calls, append results to context, go to step 2. If it returned a final response, persist the user-visible message to DDB, persist turn internals to S3, update token/cost counters on the conversation record, and wait for the next signal.

### LLM Strategy

Multi-provider with a primary. The primary model (e.g., Claude) handles main reasoning. Cheaper/faster models can be used for summarization, relevance scoring, or subagent tasks where full reasoning power isn’t needed. Model selection is configurable per conversation and per subagent.

## Tool System

Each tool is defined by a spec (name, description, JSON schema — what the LLM sees) and an activity implementation. New tools are added by defining a spec and registering a Temporal activity.

### Built-in Tools

|Tool              |Description                                                                                                       |Timeout  |Retries|
|------------------|------------------------------------------------------------------------------------------------------------------|---------|-------|
|`semantic_search` |Vector search against the document corpus. Query string in, ranked chunks with source metadata out.               |10s      |2      |
|`full_text_search`|Keyword/phrase search with optional filters (date range, document type, topic).                                   |10s      |2      |
|`eval_code`       |Execute a code snippet in a sandboxed environment for data crunching. Strict resource limits.                     |30s      |0      |
|`store_memory`    |Persist a key-value pair to conversation-scoped memory in DDB.                                                    |5s       |3      |
|`recall_memory`   |Retrieve stored memories by key or list all.                                                                      |5s       |3      |
|`load_skill`      |Fetch a skill by ID from the Skills table, inject its content into the current context.                           |5s       |3      |
|`start_subagent`  |Start a child workflow with a goal, allowed tools, relevant context, and a budget. Returns the subagent’s summary.|inherited|0      |

### Skill System

Skills are dynamic prompt text stored in DDB, managed via admin API. The full list of skill names and descriptions is included in the system prompt so the LLM knows what’s available. The `load_skill` tool fetches the full content on demand. Skills belong to the system, not to individual data rooms or users.

## Context Management

### Context Assembly Order

1. System prompt (static instructions + skill name/description directory)
1. Loaded skill contents (from prior `load_skill` calls in the conversation)
1. Conversation history from DDB Messages
1. Current turn’s tool results (in-memory workflow state)
1. Tool definitions (unless budget exhausted)

### Context Size Management

Token counts come directly from LLM API responses — no separate estimator needed. After each LLM call, check the returned input token count against the model’s context window. If approaching the limit before the next call, apply truncation.

**History truncation strategy:** Keep the first message (original ask) and the last N turns in full. Summarize the middle.

**Tool result truncation:** Large results are truncated with a note that full results are available. The model can request more via follow-up tool calls.

**Subagent offloading:** When the model needs a deep dive that would consume significant context, it spawns a subagent with a focused goal and clean context. Only the summary returns to the parent.

**Memory:** `store_memory` / `recall_memory` let the agent persist structured findings outside the context window, useful across turns without replaying full tool results.

## Subagents

Subagents are Temporal child workflows. The parent starts a child via `workflow.ExecuteChildWorkflow` and blocks until it completes.

- Child receives: goal string, allowed tool list, relevant context from parent, budget (subset of parent’s remaining per-turn budget).
- Child returns: summary string that becomes the tool result in the parent.
- Nesting: one level only. Subagents cannot spawn their own subagents.
- Cancellation: if the parent conversation is abandoned, child workflows are cancelled automatically by Temporal.

## Temporal Workflow Design

- **Workflow ID** = conversation ID. Natural deduplication.
- **Signals** for inbound user messages.
- **Continue-as-new** after every N turns (e.g., 20) to shed Temporal’s internal event history. Carries forward conversation ID and current state.
- **Parallel tool execution** via `workflow.Go` goroutines, results collected via futures.

### Activity Policies

|Activity    |Timeout|Retries     |Notes                         |
|------------|-------|------------|------------------------------|
|LLM Call    |2–5 min|2–3, backoff|Transient errors common       |
|Search tools|10s    |2           |                              |
|Code eval   |30s    |0           |Non-deterministic, don’t retry|
|Memory ops  |5s     |3           |                              |
|Skill load  |5s     |3           |                              |

## API Design

### Conversation Endpoints

`POST /conversations` — Start a new conversation. Body: `{ "message": "..." }`. Returns `{ "conversation_id": "..." }`. Starts the Temporal workflow and signals the first message.

`POST /conversations/{id}/messages` — Send a follow-up message. Body: `{ "message": "..." }`. Signals the existing workflow. Returns `202 Accepted`.

`GET /conversations/{id}/messages?after={sequence_number}` — Poll for new messages. Returns messages with sequence number greater than `after`. Status field indicates whether the agent is still thinking: `{ "messages": [...], "status": "thinking" | "ready" | "error" }`.

`GET /conversations` — List user’s conversations. Queries DDB by user ID.

### Admin Endpoints

`POST /skills` — Create a skill.
`GET /skills` — List all skills.
`PUT /skills/{id}` — Update a skill.
`DELETE /skills/{id}` — Deactivate a skill.

### Auth

User ID extracted from auth token (JWT or similar), used as the DDB partition key. Details TBD.

### Future Upgrade Path

Polling keeps the API server stateless and simple. SSE or WebSocket can be added later as an optional upgrade — the DDB-backed model supports both since the underlying data access is the same.

## Cost & Budget Controls

### Per-Turn Limits (hard)

- Max tool calls per turn (e.g., 10). When exhausted, strip tool definitions from the next LLM call.
- Max LLM iterations per turn (e.g., 5). Safety net against loops.

### Per-Conversation Tracking (informational)

- Running totals of input/output tokens and tool invocations stored on the conversation record in DDB.
- Estimated cost surfaced to the user in the UI. No hard stop — the tool saves employee time and has positive ROI, so the goal is visibility, not restriction.

### Subagent Budgets

Child workflows inherit a subset of the parent’s remaining per-turn budget, not their own full allocation.

### Observability

- Per-turn metrics (token counts, tool calls, latency, model) logged to S3 with turn internals.
- Aggregate metrics (spend per user, per conversation, per model) derived from DDB conversation records.
