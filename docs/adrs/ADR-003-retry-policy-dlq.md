# ADR-003: Retry Policy with Exponential Backoff and Dead Letter Queue

**Status:** Accepted

## Context

Webhook delivery to external client endpoints can fail due to transient issues (network blips, client deployments, temporary overload) or permanent issues (client endpoint decommissioned, URL changed, client gone out of business). The system needs a retry strategy that:
- Recovers from transient failures without manual intervention
- Doesn't overwhelm a struggling client endpoint
- Doesn't hold resources indefinitely for permanently failed deliveries
- Provides visibility and manual recovery for permanently failed events

## Decision

### Retry Policy
- **Maximum attempts**: 5 total (1 initial + 4 retries)
- **Backoff schedule** (time since previous attempt):
  1. 1 minute
  2. 5 minutes
  3. 30 minutes
  4. 2 hours
  5. 12 hours
- **Total window**: ~15 hours from first failure to final attempt
- **HTTP timeout**: 10 seconds per attempt (client must respond within 10s)
- **Retry triggers**: Non-2xx HTTP status, timeout, network error, DNS failure

### Dead Letter Queue (DLQ)
After 5 failed attempts:
- Move event from `webhook_outbox` to `webhook_dead_letter` table
- DLQ row stores: full payload, failure reason (HTTP status, error message, stack trace), timestamp of final failure, attempt count
- DLQ is **separate table** (not a status in outbox) for cleaner queries and operational separation

### Manual Replay
- Admin endpoint: `POST /admin/webhooks/dead-letter/:id/replay`
- Requires `ADMIN` role (reuses existing `requireRole` middleware)
- Action: Copies event back to `webhook_outbox` with `PENDING` status, resets attempt counter
- Audit log: Records which admin user triggered replay and when

## Alternatives Considered

### 1. Three Retry Attempts (More Aggressive)
- **Trade-off**: Rejected. 3 attempts with similar backoff covers only ~30-40 minutes. Real incidents: client had 2-hour planned maintenance window (transcript [09:16] Diego). 3 attempts would have permanently failed a valid temporary outage.

### 2. Indefinite Retry with Backoff
- **Trade-off**: Rejected. Events would remain in retry limbo forever if client endpoint is permanently gone. Consumes worker capacity, pollutes metrics, no clear "this is dead" signal. No visibility into which deliveries are truly failed vs. just slow.

### 3. DLQ as Status in Same Table
Keep failed events in `webhook_outbox` with status `DEAD_LETTER`.

**Trade-off**: Rejected. Mixes active queue with archive. Queries for "what's pending" must filter out DLQ. Harder to reason about table size. Separate table provides cleaner operational model.

### 4. Automatic Replay After Client Recovery
Detect client health and auto-replay.

**Trade-off**: Rejected (deferred). Adds significant complexity (health checks, circuit breakers). Manual replay is sufficient for MVP; automation can be added if DLQ volume justifies it.

## Consequences

### Positive
- **Covers realistic outages**: 15-hour window handles extended maintenance, deployments, overnight incidents
- **Bounded resource consumption**: Max 5 attempts per event, then moves to DLQ
- **Operational clarity**: DLQ = "needs human attention"; outbox = "in progress"
- **Auditability**: Full failure context preserved (payload, error, timing)
- **Recovery path**: Simple admin action to re-trigger delivery after client fixes their endpoint

### Negative
- **Delivery delay**: Up to ~15 hours before event reaches DLQ (though most transient failures resolve in first 1-2 retries)
- **Manual intervention required**: DLQ doesn't self-heal; requires admin action
- **DLQ growth**: Needs monitoring and retention policy (30-day archive mentioned in transcript)
- **Duplicate risk on replay**: Replayed events get new `X-Event-Id`? No — must preserve original `event_id` for client dedup (see ADR-005)

## References
- Related: ADR-001 (Outbox Pattern)
- Related: ADR-002 (Worker Polling)
- Related: ADR-005 (At-Least-Once Delivery)
- Code: `src/modules/webhooks/webhook.processor.ts` (delivery logic)
- Transcript: [09:14] Larissa, [09:15] Diego, [09:16] Diego, [09:17] Diego, [09:18] Diego, [09:18] Bruno