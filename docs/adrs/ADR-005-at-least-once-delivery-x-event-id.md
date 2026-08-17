# ADR-005: At-Least-Once Delivery with X-Event-Id for Client-Side Deduplication

**Status:** Accepted

## Context

Network failures, client timeouts, and retries mean the same webhook event may be delivered multiple times. The team needed to decide on a delivery guarantee and how clients should handle duplicates.

Options on the spectrum:
- **At-most-once**: Fire and forget; no retry. Events lost on failure.
- **At-least-once**: Retry until success; duplicates possible.
- **Exactly-once**: Complex coordination; typically requires two-phase commit or client-side acknowledgment protocol.

## Decision

**Guarantee: At-least-once delivery.**

- The worker retries failed deliveries per ADR-003 (5 attempts, exponential backoff)
- The same event (same `X-Event-Id`) may be delivered multiple times
- **Client responsibility**: Deduplicate using `X-Event-Id` header

### X-Event-Id Specification
- **Generated at outbox insertion time** (inside the order status transaction)
- **UUID v4** (random, 122 bits of entropy)
- **Unique per event** — not per delivery attempt
- Included in **every** delivery attempt (initial + all retries + manual replay)
- Preserved when replaying from DLQ (same `event_id` reinserted into outbox)

### Client Integration Guidance (to be documented in developer portal)
1. Store processed `X-Event-Id` values (with TTL, e.g., 30 days)
2. On webhook receipt: check if `X-Event-Id` already processed
3. If yes: return 2xx immediately, skip processing
4. If no: process payload, store `X-Event-Id`, return 2xx

### Additional Headers Supporting Deduplication
- `X-Timestamp`: ISO 8601 send time — client can reject events older than acceptable window
- `X-Webhook-Id`: Configuration ID — client can scope dedup per endpoint if desired

## Alternatives Considered

### 1. Exactly-Once Delivery
Implement a two-phase protocol: worker sends event → client acknowledges → worker marks delivered. Or use idempotency keys with client-side storage coordination.

**Trade-off:** Rejected because:
- Requires client to implement acknowledgment endpoint (additional contract)
- Client must persist acknowledgment state durably before responding
- Network partition during ack = uncertainty; requires reconciliation
- Significantly more complex for both sides
- Industry standard (Stripe, GitHub, Slack, Twilio) is at-least-once + client dedup

### 2. At-Most-Once (No Retry)
Send once; if fails, log and move on (or DLQ immediately).

**Trade-off:** Rejected because:
- Transient failures (network blip, client restart) would permanently lose events
- Unacceptable for B2B integrations where reliability is a selling point
- Clients explicitly asked for "not having to poll manually"

### 3. Deduplication on Server Side
Worker tracks delivered `event_id` + `webhook_id` and skips if already delivered.

**Trade-off:** Rejected because:
- Requires durable storage of delivery receipts (new table, writes on every delivery)
- Doesn't solve client-side duplicate processing (client could still crash after processing but before responding)
- Client-side dedup is still needed for exactly-once semantics
- Adds complexity without eliminating client responsibility

## Consequences

### Positive
- **Simple implementation**: Worker just retries; no coordination protocol
- **Industry compatible**: Matches Stripe, GitHub, AWS EventBridge, etc.
- **Client control**: Clients choose dedup strategy (in-memory, Redis, DB) and TTL
- **Works with retry/DLQ**: Same `event_id` flows through retries, DLQ, and manual replay
- **Audit trail**: `X-Event-Id` correlates outbox row, delivery logs, and client processing

### Negative
- **Client must implement dedup**: Not all clients will do it correctly
- **Storage on client**: Clients must persist processed IDs (mitigated: TTL limits growth)
- **Duplicate processing possible**: If client crashes after processing but before storing ID, event reprocessed on retry
- **Replay from DLQ sends same ID**: Correct behavior, but clients must handle (dedup covers it)

## References
- Related: ADR-001 (Outbox — event_id generated at insertion)
- Related: ADR-003 (Retry Policy — same event_id on retries)
- Related: ADR-004 (HMAC — X-Event-Id in signed payload)
- Code: `src/modules/webhooks/webhook.processor.ts` (adds headers)
- Transcript: [09:24] Diego, [09:25] Diego, [09:25] Sofia, [09:26] Marcos