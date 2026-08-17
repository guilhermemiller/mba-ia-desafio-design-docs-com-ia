# ADR-001: Outbox Pattern with MySQL for Reliable Webhook Event Delivery

**Status:** Accepted

## Context

The team needs to notify B2B clients (Atlas Comercial, MaxDistribuição, Nova Cargo) when order status changes in near real-time (< 10 seconds). The current `OrderService.changeStatus` method performs a complex database transaction that:
- Updates the `orders` table status
- Inserts into `order_status_history` for audit
- Debits or replenishes product stock (`stock_quantity`)

Adding synchronous HTTP calls to external client endpoints within this transaction would block order status changes for all customers if any single client is slow or unavailable. A rollback of the status change due to a client timeout is unacceptable.

## Decision

Implement the **Transactional Outbox Pattern** using a `webhook_outbox` table in the existing MySQL database.

Within the same Prisma transaction that executes `changeStatus`:
1. Update order status
2. Insert into `order_status_history`
3. Adjust stock quantities
4. **Insert one row into `webhook_outbox` per matching webhook subscription** containing the fully rendered event payload

A separate worker process (`src/worker.ts`) polls the outbox table, delivers events via HTTP, and marks them as delivered.

## Alternatives Considered

### 1. Synchronous HTTP in the Transaction
Call client webhook URLs directly inside `OrderService.changeStatus` transaction.

**Trade-off:** Rejected because:
- Blocks the database transaction for the duration of HTTP calls (network latency, client processing time)
- Any client failure forces rollback of the order status change
- No retry mechanism without complex compensation logic
- Violates single responsibility: order service shouldn't know about external integrations

### 2. Redis Streams / External Message Broker (Kafka, RabbitMQ)
Use a dedicated message queue infrastructure.

**Trade-off:** Rejected because:
- Adds operational complexity (new infrastructure to provision, monitor, secure)
- Team is small; Redis Cluster or Kafka is overengineering for this feature scope
- Existing MySQL can handle the throughput (estimated < 100 events/second)
- Increases deployment surface and failure modes

## Consequences

### Positive
- **Atomicity**: Event is persisted if and only if the order status transaction commits. No inconsistency between order state and notification state.
- **No new infrastructure**: Leverages existing MySQL and Prisma.
- **Built-in audit trail**: Every event ever generated is queryable in `webhook_outbox`.
- **Decoupling**: Order service knows nothing about HTTP, retries, or client availability.
- **Replayability**: Failed events can be re-sent by resetting their status.

### Negative
- **Polling latency**: Worker introduces minimum 2-second delay (see ADR-002).
- **Table growth**: `webhook_outbox` requires archival/cleanup strategy for delivered events (post-MVP).
- **Worker process**: Requires separate deployment and monitoring (`src/worker.ts`).
- **Ordering guarantee limited**: Only guaranteed per `order_id` with single worker (see ADR-002).

## References
- Related: ADR-002 (Worker Polling Strategy)
- Related: ADR-007 (Snapshot Payload at Insertion)
- Code: `src/modules/orders/order.service.ts` (`changeStatus` method)
- Code: `prisma/schema.prisma` (Order, OrderStatusHistory, Product models)