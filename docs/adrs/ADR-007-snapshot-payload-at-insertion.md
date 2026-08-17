# ADR-007: Snapshot Payload at Outbox Insertion (Not at Delivery Time)

**Status:** Accepted

## Context

When an order status changes, the webhook event payload must reflect the order state at that exact moment. The team discussed whether the `webhook_outbox` should store:
1. **Rendered payload snapshot**: Full JSON payload generated at insertion time (inside the order status transaction)
2. **Reference only**: Store `order_id` + `from_status` + `to_status`; worker renders payload at delivery time by fetching current order state

The concern with option 2: if the order is modified after the status change but before delivery (e.g., notes added, items changed via admin), the delivered payload would reflect the *current* state, not the state at the moment of the status transition. This breaks auditability and confuses clients.

## Decision

**Store the fully rendered JSON payload at outbox insertion time.**

- The `publishWebhookEvent(tx, order, fromStatus, toStatus)` function (called inside the `changeStatus` transaction) serializes the complete event payload and inserts it into `webhook_outbox.payload` (JSON/TEXT column).
- The worker reads the pre-rendered payload and sends it directly — no additional database queries, no re-rendering.
- Payload includes: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` (per transcript [09:43] Diego — no items to keep payload small).

## Alternatives Considered

### 1. Store References Only, Render at Delivery
Outbox row contains only `order_id`, `from_status`, `to_status`, `event_id`. Worker fetches order at delivery time and builds payload.

**Trade-off:** Rejected because:
- **Auditability broken**: Delivered payload �� state at transition time
- **Race condition**: Order could be modified between status change and delivery (even seconds later)
- **Client confusion**: Client receives "status changed to SHIPPED" but payload shows different total/items than when it actually shipped
- **Extra DB load**: Worker must query order for every delivery (N+1 problem at scale)
- **Complexity**: Worker needs order context, joins, serialization logic

### 2. Hybrid: Store Minimal Data, Reconstruct at Delivery
Store only changed fields + event metadata; reconstruct full payload at delivery.

**Trade-off:** Rejected. Same auditability issues as option 1. Adds reconstruction complexity without benefit.

## Consequences

### Positive
- **Immutable audit trail**: Payload in outbox = exact state at status change moment
- **Worker simplicity**: Just reads `payload` column and sends; no Prisma queries in hot path
- **Performance**: Single-row insert in transaction; worker does zero reads for payload
- **Correctness**: Clients can trust payload matches the event they're being notified about
- **Debugging**: Outbox table is complete record — no need to correlate with order history

### Negative
- **Larger outbox rows**: Payload ~1-2 KB per event (acceptable; transcript [09:43] Diego confirms small payloads)
- **Schema evolution**: If payload format changes, old events in outbox/DLQ have old format (mitigated: version field in payload, workers handle multiple versions)
- **Storage**: 30-day retention of delivered events means more disk (mitigated: archival job, payloads are small)

## References
- Related: ADR-001 (Outbox Pattern — payload column in `webhook_outbox`)
- Related: ADR-002 (Worker Polling — worker reads pre-rendered payload)
- Code: `src/modules/orders/order.service.ts` (`changeStatus` transaction, lines 126-179)
- Code: `src/modules/webhooks/webhook.processor.ts` (delivery logic — reads payload, sends)
- Transcript: [09:51] Larissa, [09:52] Diego, [09:52] Bruno