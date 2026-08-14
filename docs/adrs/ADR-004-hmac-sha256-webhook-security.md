# ADR-004: HMAC-SHA256 for Webhook Signature Verification

**Status:** Accepted

## Context

Webhook payloads contain sensitive order data (customer info, order totals, status transitions) and are delivered to endpoints outside our infrastructure. Clients must be able to:
1. Verify the request genuinely originated from our platform (authenticity)
2. Verify the payload was not tampered in transit (integrity)
3. Reject replayed requests (replay protection)

The security engineer (Sofia) mandated that each client endpoint have its own secret — a single global secret would create a blast radius where one leak compromises all clients.

## Decision

### Signing Algorithm
- **HMAC-SHA256** over the raw request body (JSON payload)
- Header: `X-Signature: sha256=<hex-encoded-hmac>`
- Clients verify by computing HMAC-SHA256(body, shared_secret) and comparing to header (constant-time comparison)

### Secret Management
- **Per-endpoint secret**: Each webhook configuration stores a unique `secret` (generated server-side, returned once on creation)
- **Secret rotation**: Clients can request a new secret via API
  - On rotation: new secret becomes active immediately
  - Old secret remains valid for **24-hour grace period** (worker tries both)
  - After 24h: old secret revoked
- **Secret storage**: Hashed at rest (bcrypt/argon2) like passwords; only plaintext returned once on creation/rotation

### Transport Security
- **TLS mandatory**: Webhook URL must be `https://` — rejected at validation (Zod schema)
- **No HTTP fallback**: Plaintext delivery is not supported

### Additional Headers for Security
- `X-Event-Id`: UUID generated at outbox insertion (replay detection, dedup)
- `X-Webhook-Id`: Webhook configuration ID (client knows which endpoint received it)
- `X-Timestamp`: ISO 8601 timestamp of delivery attempt (client can reject stale replays)

### Payload Size Limit
- **64 KB maximum**: Requests exceeding this are rejected with error (no truncation)
- Rationale: Events are small (~1-2 KB); larger payloads indicate a bug

## Alternatives Considered

### 1. Global Shared Secret
Single secret for all webhooks.

**Trade-off:** Rejected. One leak (e.g., in client logs) compromises all webhook endpoints. Per-endpoint secrets limit blast radius to one client.

### 2. JWT / OIDC Tokens
Sign payloads with asymmetric keys (RS256) or issue short-lived JWTs.

**Trade-off:** Rejected. Overengineering for server-to-server webhooks. HMAC is simpler, widely supported, and standard in the industry (Stripe, GitHub, Slack all use HMAC).

### 3. Mutual TLS (mTLS)
Require client certificates for webhook endpoints.

**Trade-off:** Rejected. High operational burden for clients (cert provisioning, rotation). HMAC with secret rotation achieves similar security with simpler client integration.

### 4. No Signature (Trust Network)
Rely on IP allowlists or network isolation.

**Trade-off:** Rejected. Clients are external B2B partners on public internet. IP allowlists are brittle (CDNs, load balancers change IPs). Signature verification works regardless of network path.

## Consequences

### Positive
- **Industry standard**: Clients can use existing HMAC libraries (every language has one)
- **Per-endpoint isolation**: Compromise of one client's secret doesn't affect others
- **Zero-downtime rotation**: 24h grace period allows clients to update their systems without failed deliveries
- **Replay protection**: `X-Event-Id` + `X-Timestamp` enable clients to detect and reject replays
- **Auditability**: Signature verification proves payload integrity at receipt time

### Negative
- **Client implementation burden**: Clients must implement HMAC verification (mitigated: documented in developer portal)
- **Secret storage**: Must hash secrets at rest; plaintext only returned once (like passwords)
- **Rotation complexity**: Worker must verify against both old and new secret during grace period
- **Clock skew**: `X-Timestamp` replay window requires reasonable clock sync (document acceptable skew, e.g., ±5 minutes)

## References
- Related: ADR-005 (At-Least-Once Delivery with X-Event-Id)
- Related: ADR-006 (Reuse Existing Patterns - AppError, Zod validation)
- Code: `src/modules/webhooks/webhook.service.ts` (HMAC generation/verification)
- Code: `src/modules/webhooks/webhook.schemas.ts` (Zod schema enforcing https://)
- Code: `src/middlewares/auth.middleware.ts` (requireRole for admin endpoints)
- Transcript: [09:19] Sofia, [09:20] Sofia, [09:21] Sofia, [09:22] Diego, [09:23] Sofia, [09:24] Diego