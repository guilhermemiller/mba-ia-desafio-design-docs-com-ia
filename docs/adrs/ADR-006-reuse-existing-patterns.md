# ADR-006: Reuse Existing Project Patterns for Webhook Module

**Status:** Accepted

## Context

The codebase has well-established patterns across all existing modules (`auth`, `users`, `customers`, `products`, `orders`). Introducing a new module (`webhooks`) is an opportunity to either follow these patterns (consistency, lower cognitive load) or introduce new approaches (potential improvements but higher maintenance burden).

## Decision

The `webhooks` module **strictly follows all existing project patterns**:

### 1. Module Structure (`src/modules/webhooks/`)
```
webhook.controller.ts      # HTTP layer, request/response handling
webhook.service.ts         # Business logic, orchestration
webhook.repository.ts      # Data access (Prisma)
webhook.schemas.ts         # Zod validation schemas
webhook.routes.ts          # Route definitions
webhook.errors.ts          # WEBHOOK_* error classes (extends AppError)
webhook.processor.ts       # Core delivery logic (called by worker)
```

### 2. Error Handling
- All webhook-specific errors extend `src/shared/errors/app-error.ts` → `AppError`
- Error codes use `WEBHOOK_*` prefix (e.g., `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_DELIVERY_FAILED`, `WEBHOOK_REPLAY_FAILED`)
- Caught automatically by existing `src/middlewares/error.middleware.ts`
- No changes to error middleware needed

### 3. Validation
- Zod schemas in `webhook.schemas.ts`
- Used with existing `src/middlewares/validate.middleware.ts`
- Enforces HTTPS URLs, UUID formats, enum values

### 4. Authentication & Authorization
- Reuses `src/middlewares/auth.middleware.ts`:
  - `authenticate` middleware for JWT validation
  - `requireRole('ADMIN')` for DLQ replay endpoint
- No new auth logic

### 5. Logging
- Uses existing Pino logger (`src/shared/logger/`)
- Structured logging with existing redaction rules
- No new logger configuration

### 6. Dependency Injection
- Manual DI in `src/app.ts` (`buildControllers` function)
- Webhook controllers/services/repositories instantiated there
- Worker (`src/worker.ts`) creates its own instances (separate process)

### 7. Database
- Prisma Client (same version, same patterns)
- Worker uses **separate PrismaClient instance** (per-process)
- Migrations via `prisma migrate dev`

### 8. HTTP Responses
- Uses `src/shared/http/response.ts` helpers (`paginated`, etc.)
- Consistent response envelope across all endpoints

### 9. IDs
- All entities use UUID (`@default(uuid()) @db.Char(36)`)
- Consistent with `prisma/schema.prisma` patterns

## Alternatives Considered

### 1. New Error Handling Approach
Use a different error library or format (e.g., RFC 7807 Problem Details).

**Trade-off:** Rejected. Existing `AppError` + centralized middleware works well. Changing would require updating all modules or living with inconsistency.

### 2. New Validation Library
Use `class-validator` / decorators instead of Zod.

**Trade-off:** Rejected. Zod is already used everywhere; schema-first approach matches TypeScript types.

### 3. Worker as In-Process Background Job
Run worker as `setInterval` inside the API process.

**Trade-off:** Rejected. API restart = worker stop. Separate process is standard for reliability (transcript [09:11] Diego).

### 4. Shared PrismaClient Between API and Worker
Single instance across processes.

**Trade-off:** Rejected. PrismaClient is not process-safe. Each process needs its own connection pool (transcript [09:29] Bruno, [09:30] Diego).

## Consequences

### Positive
- **Zero new dependencies** for core patterns
- **Consistent developer experience**: Any engineer familiar with `orders` module can work on `webhooks`
- **Existing tests/middleware work automatically**: No changes to `error.middleware.ts`, `validate.middleware.ts`, `auth.middleware.ts`
- **Lower maintenance surface**: One way to do things
- **Faster onboarding**: Patterns documented implicitly by existing code

### Negative
- **Inflexibility**: Can't improve patterns without changing all modules
- **Carries legacy decisions**: If existing patterns have flaws, they propagate
- **Worker/process separation adds deployment complexity**: But this is necessary (not a pattern choice)

## References
- Code: `src/app.ts` (`buildControllers` pattern)
- Code: `src/modules/orders/order.service.ts` (transaction pattern, error usage)
- Code: `src/modules/orders/order.schemas.ts` (Zod schema pattern)
- Code: `src/middlewares/auth.middleware.ts` (auth/role pattern)
- Code: `src/middlewares/error.middleware.ts` (error handling)
- Code: `src/shared/errors/app-error.ts` (base error class)
- Code: `src/shared/errors/http-errors.ts` (specific error classes)
- Code: `src/shared/logger/` (Pino setup)
- Code: `src/shared/http/response.ts` (response helpers)
- Transcript: [09:27] Bruno, [09:28] Bruno, [09:29] Bruno, [09:30] Larissa