# Local SQLite-Backed AWS Mocks

**Date:** 2026-03-02
**Status:** Draft
**Builds on:** [2026-03-02 Agentic Loop State Management Design](2026-03-02-agentic-loop-state-management-design.md)

## Problem

We need S3 and DynamoDB for the agentic loop, but real AWS infrastructure is slow to iterate against and painful in unit tests. We want local-only implementations backed by SQLite so tests run fast, state is easy to inspect, and bad experiments are trivially thrown away.

## AWS APIs Required

### S3: 2 operations

| Operation | Used For |
|-----------|----------|
| **PutObject** | Write conversation snapshots, tool results, compactions |
| **GetObject** | Read them back by exact key |

No listing, no deletion, no multipart. The workflow tracks the latest S3 key as a single string — pure key-value access.

### DynamoDB: 4 operations

| Operation | Table | Used By | Purpose |
|-----------|-------|---------|---------|
| **PutItem** | Conversations | API layer | Create new conversation |
| **PutItem** | Messages | API layer, PublishProgress, PublishResponse | Write user/assistant/tool_progress messages |
| **GetItem** | Conversations | API layer | Fetch single conversation by (user_id, conversation_id) |
| **UpdateItem** (SET + ADD) | Conversations | PublishResponse | Atomic increment of `total_cost_estimate`, SET `updated_at`/`status` |
| **Query** (by PK) | Conversations | API layer | List conversations for a user |
| **Query** (by PK) | Messages | API layer | List messages for a conversation |

**Not needed:** BatchWriteItem, Scan, DeleteItem.

### Cross-table atomicity

`PublishResponse` must write an assistant message to Messages AND update conversation metadata (cost, updated_at, status) in Conversations. In real DynamoDB this is `TransactWriteItems`. In SQLite this is a plain SQL transaction. The interface exposes this as a single atomic method so neither implementation has to worry about partial writes.

## Interface Design

Domain-specific interfaces, not AWS SDK-shaped. This keeps the mock simple, tests readable, and avoids pulling in AWS SDK types at the interface boundary. Real AWS implementations wrap the SDK internally.

### BlobStore (S3 abstraction)

```go
package storage

import "context"

// BlobStore is a key-value store for opaque binary data.
// The S3 implementation uses a single bucket; the key is the full S3 object key.
type BlobStore interface {
    Put(ctx context.Context, key string, data []byte) error
    Get(ctx context.Context, key string) ([]byte, error)
}
```

- Treats all data as opaque `[]byte` — zstd compression is the caller's concern
- `Get` returns a sentinel error (`ErrNotFound`) when the key doesn't exist

### ConversationStore (DynamoDB abstraction)

```go
package storage

import (
    "context"
    "encoding/json"
)

type ConversationStore interface {
    // Conversations table
    CreateConversation(ctx context.Context, conv *Conversation) error
    GetConversation(ctx context.Context, userID, conversationID string) (*Conversation, error)
    ListConversations(ctx context.Context, userID string) ([]*Conversation, error)

    // Messages table
    CreateMessage(ctx context.Context, msg *Message) error
    ListMessages(ctx context.Context, conversationID string, after string) ([]*Message, error)

    // Cross-table atomic write for PublishResponse activity
    PublishResponse(ctx context.Context, msg *Message, costDelta float64) error
}

type Conversation struct {
    UserID            string
    ConversationID    string
    Status            string
    Title             string
    TotalCostEstimate float64
    Config            json.RawMessage
    CreatedAt         string
    UpdatedAt         string
}

type Message struct {
    ConversationID string
    MessageID      string
    Role           string
    Content        string
    SnapshotKey    string
    CostEstimate   float64
    CreatedAt      string
}
```

`PublishResponse` atomically:
1. Writes the assistant message to Messages
2. Increments `total_cost_estimate` by `costDelta` on the Conversation record
3. Updates `updated_at` and `status` on the Conversation record

## SQLite Schema

### blobs (S3 mock)

```sql
CREATE TABLE IF NOT EXISTS blobs (
    key  TEXT PRIMARY KEY,
    data BLOB NOT NULL
);
```

### conversations

```sql
CREATE TABLE IF NOT EXISTS conversations (
    user_id             TEXT NOT NULL,
    conversation_id     TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'active',
    title               TEXT NOT NULL DEFAULT '',
    total_cost_estimate REAL NOT NULL DEFAULT 0,
    config              TEXT NOT NULL DEFAULT '{}',
    created_at          TEXT NOT NULL,
    updated_at          TEXT NOT NULL,
    PRIMARY KEY (user_id, conversation_id)
);
```

### messages

```sql
CREATE TABLE IF NOT EXISTS messages (
    conversation_id TEXT NOT NULL,
    message_id      TEXT NOT NULL,
    role            TEXT NOT NULL,
    content         TEXT NOT NULL,
    snapshot_key    TEXT NOT NULL DEFAULT '',
    cost_estimate   REAL NOT NULL DEFAULT 0,
    created_at      TEXT NOT NULL,
    PRIMARY KEY (conversation_id, message_id)
);
```

Message IDs are ULIDs, so `ORDER BY message_id ASC` gives chronological order. The `after` parameter in `ListMessages` becomes `WHERE message_id > ?`.

## Package Structure

```
storage/
    blob.go              # BlobStore interface, ErrNotFound
    conversation.go      # ConversationStore interface, Conversation, Message
    sqliteblob/
        store.go         # SQLite BlobStore implementation
        store_test.go
    sqliteconv/
        store.go         # SQLite ConversationStore implementation
        store_test.go
```

## Implementation Notes

- **Pure-Go SQLite** via `modernc.org/sqlite` — no CGo, easy cross-compilation, no system library dependencies.
- **In-memory DBs** (`:memory:`) for tests — fast setup/teardown, no cleanup needed.
- **`Put` is INSERT OR REPLACE** — idempotent writes, matches S3 overwrite semantics.
- **`PublishResponse` uses a SQL transaction** — BEGIN → INSERT message → UPDATE conversation cost/updated_at/status → COMMIT. The real DDB implementation will use `TransactWriteItems`.

## Design Decisions

1. **Domain interfaces, not AWS-shaped** — No `*s3.GetObjectInput` / `*dynamodb.PutItemInput` at the boundary. Simpler mocks, no AWS SDK dependency in the interface package.
2. **`PublishResponse` as an explicit atomic method** — Makes the cross-table atomicity requirement visible in the interface rather than leaving it to callers to coordinate.
3. **Compression is the caller's problem** — BlobStore stores opaque bytes. Activities handle zstd encode/decode before calling Put/Get.
4. **No standalone `UpdateConversation`** — The only update path in the current design is the atomic cost+status update inside PublishResponse. Add it when needed.
5. **No `DeleteItem` / `DeleteObject`** — Not in the current design. Old snapshots remain for audit.
