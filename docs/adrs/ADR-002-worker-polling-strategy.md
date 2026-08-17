# ADR-002: Worker Polling Strategy for Outbox Event Processing

**Status:** Accepted

## Context

The outbox pattern (ADR-001) requires a mechanism to read pending events from the `webhook_outbox` table and deliver them via HTTP to client endpoints. The team evaluated how the worker should be triggered.

MySQL does not support native notification mechanisms like PostgreSQL's `LISTEN/NOTIFY`. The team discussed using database triggers, but MySQL triggers cannot invoke external processes—they only execute SQL statements. Any workaround (writing to a file, calling an HTTP endpoint) would be fragile and non-standard.

## Decision

Implement a **polling-based worker** running as a separate Node.js process (`src/worker.ts`) with the following characteristics:

- **Polling interval**: 2 seconds
- **Batch size**: Small batch of oldest `PENDING` events (e.g., 10-50)
- **Processing order**: FIFO by `created_at`
- **Status transitions**: `PENDING` → `PROCESSING` → `DELIVERED` (or `FAILED` for retry/DLQ)
- **Single worker instance** for MVP (ordering guarantee per `order_id`)

The worker is deployed independently from the API server (`npm run worker`), with its own PrismaClient instance connected to the same database.

## Alternatives Considered

### 1. MySQL Triggers with External Notification
Create a trigger on `webhook_outbox` insert that attempts to notify an external process.

**Trade-off:** Rejected because:
- MySQL triggers cannot execute external programs or make HTTP calls
- Workarounds (e.g., trigger writes to a file watched by worker, or calls a UDF) are hacky, non-portable, and hard to debug
- Adds coupling between database and application layer

### 2. PostgreSQL Migration for `LISTEN/NOTIFY`
Migrate from MySQL to PostgreSQL to leverage native pub/sub.

**Trade-off:** Rejected because:
- Migration scope is disproportionate to the feature
- Team has MySQL expertise and existing data
- No other driver for migration

### 3. Redis Pub/Sub as Notification Layer
Use Redis to notify workers of new outbox events.

**Trade-off:** Rejected because:
- Adds Redis infrastructure (see ADR-001 rejection of external message brokers)
- Dual-write problem: must write to MySQL outbox AND publish to Redis atomically
- Polling is sufficiently fast for the <10s latency requirement

## Consequences

### Positive
- **Simplicity**: Pure application-level logic, no database-specific features
- **Meets latency**: 2s poll + processing < 10s requirement comfortably
- **No new infrastructure**: Uses existing MySQL connection
- **Operational visibility**: Easy to monitor (query `PENDING` count = lag)
- **Fault tolerance**: Worker crash = events accumulate in `PENDING`; restart resumes automatically
- **Ordering (single worker)**: Events processed FIFO by `created_at`, preserving per-`order_id` sequence

### Negative
- **Minimum 2s latency**: Even with zero queue, events wait up to 2 seconds
- **Polling overhead**: Continuous queries even when idle (mitigated by small interval and indexed `status` + `created_at`)
- **No horizontal scaling guarantee**: Single worker for MVP; multi-worker requires partitioning strategy (future work)
- **Worker must be separate process**: Cannot run inside API server (API restart would stop delivery)

## References
- Related: ADR-001 (Outbox Pattern)
- Related: ADR-003 (Retry Policy and DLQ)
- Code: `src/modules/orders/order.service.ts` (integration point)
- Transcript: [09:09] Diego, [09:11] Diego, [09:12] Diego