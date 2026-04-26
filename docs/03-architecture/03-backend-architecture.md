# 03 · Backend Architecture

**Status:** 🟢 Approved

The detailed design of the Volu backend — modular monolith built with NestJS.

---

## Module structure

The backend is organised as **NestJS modules**, one per domain capability. Each module is a self-contained unit with strict boundaries.

```
volu-backend/
├── src/
│   ├── main.ts                  # bootstrap
│   ├── app.module.ts            # root module wiring
│   │
│   ├── modules/
│   │   ├── identity/
│   │   │   ├── identity.module.ts
│   │   │   ├── controllers/
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── users.controller.ts
│   │   │   │   └── merchants.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── otp.service.ts
│   │   │   │   ├── jwt.service.ts
│   │   │   │   └── kyc.service.ts
│   │   │   ├── repositories/    # Prisma access for module's tables
│   │   │   ├── domain/          # entities, value objects, business rules
│   │   │   ├── events/          # domain events emitted by this module
│   │   │   ├── dtos/            # request/response shapes
│   │   │   ├── guards/          # auth guards specific to this module
│   │   │   └── tests/
│   │   │
│   │   ├── catalog/
│   │   ├── commerce/
│   │   ├── payments/
│   │   ├── coupons/
│   │   ├── redemption/
│   │   ├── wallet/
│   │   ├── payouts/
│   │   ├── raffles/
│   │   ├── loyalty/
│   │   ├── reviews/
│   │   ├── notifications/
│   │   ├── ads/
│   │   ├── invoicing/
│   │   ├── audit/
│   │   └── reporting/
│   │
│   ├── shared/
│   │   ├── prisma/              # Prisma client + extensions
│   │   ├── events/              # event bus (Redis Streams + in-process)
│   │   ├── logger/              # Pino setup
│   │   ├── config/              # typed config from env
│   │   ├── errors/              # error classes + filters
│   │   ├── guards/              # JWT, roles, throttle (cross-cutting)
│   │   ├── interceptors/        # logging, idempotency, response shaping
│   │   ├── decorators/          # @CurrentUser, @Roles, @Idempotent
│   │   └── utils/
│   │
│   └── workers/                 # BullMQ workers (separate processes)
│       ├── coupon-expiry.worker.ts
│       ├── reminders.worker.ts
│       ├── stripe-reconcile.worker.ts
│       ├── notifications.worker.ts
│       └── ...
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── test/                         # E2E tests using supertest + Testcontainers
└── docker-compose.dev.yml        # Postgres + Redis + Meilisearch for local dev
```

---

## Module boundaries

### What's allowed across modules
- One module's controller or service can call **another module's service** through a public interface (export from the module).
- Modules emit **domain events** that other modules subscribe to.
- Cross-cutting concerns (logging, auth, config) are in `shared/`.

### What's forbidden across modules
- ❌ Reading another module's database tables directly. (Only the owning module touches its tables.)
- ❌ Importing internal services / repositories of another module.
- ❌ Modifying another module's database state synchronously without going through its public service.

### Why this discipline matters
This is what makes the monolith *modular*. When we extract a module to its own service later (e.g., Redemption to handle high RPS independently), the work is mechanical: replace direct service calls with HTTP calls, replace in-process events with Redis Stream events. The business logic doesn't change.

---

## Layers within a module

Each module follows a consistent four-layer structure:

```
controllers/   ← HTTP handlers (thin)
services/      ← business logic (thick)
repositories/  ← data access (Prisma)
domain/        ← entities, value objects, pure business rules
```

### Controllers

- Validate request shape using DTOs (Zod-backed).
- Authenticate + authorise via guards.
- Call one or two services.
- Return a DTO response.
- **Never** contain business logic.
- **Never** access the database directly.

### Services

- Implement one use case per public method.
- Orchestrate repositories, other services, and domain entities.
- Emit domain events.
- Wrap multi-step state changes in DB transactions.
- **Never** know about HTTP (no `req`, `res`).

### Repositories

- Wrap Prisma client calls.
- One repository per aggregate root (User, Merchant, Deal, Order, Coupon, ...).
- Define narrow query methods (`findActiveByMerchantId`), not generic CRUD.
- Returns domain objects, not raw Prisma types where it matters for type safety.

### Domain

- Pure TypeScript: no NestJS, no Prisma.
- Entities (rich models with behaviour, not anemic).
- Value objects (e.g., `Money`, `PhoneNumber`, `Pin`).
- Domain services for cross-entity logic.
- Easy to unit test in isolation.

---

## Example: a coupon redemption end-to-end

```typescript
// redemption.controller.ts
@Controller('redemptions')
@UseGuards(MerchantJwtGuard)
export class RedemptionController {
  constructor(private readonly redemptionService: RedemptionService) {}

  @Post('validate')
  @UseInterceptors(IdempotencyInterceptor)
  async validate(
    @CurrentUser() cashier: AuthenticatedMerchantUser,
    @Body() dto: ValidateRedemptionDto,
  ): Promise<RedemptionValidationResponse> {
    return this.redemptionService.validateQrToken({
      qrToken: dto.qrToken,
      branchId: cashier.activeBranchId,
    });
  }

  @Post('confirm')
  @UseInterceptors(IdempotencyInterceptor)
  async confirm(
    @CurrentUser() cashier: AuthenticatedMerchantUser,
    @Body() dto: ConfirmRedemptionDto,
  ): Promise<RedemptionConfirmedResponse> {
    return this.redemptionService.confirmRedemption({
      qrToken: dto.qrToken,
      pin: dto.pin,
      branchId: cashier.activeBranchId,
      cashierId: cashier.id,
      deviceFingerprint: dto.deviceFingerprint,
    });
  }
}
```

```typescript
// redemption.service.ts
@Injectable()
export class RedemptionService {
  constructor(
    private readonly couponRepo: CouponRepository,
    private readonly redemptionRepo: RedemptionRepository,
    private readonly tokenSigner: QrTokenSigner,
    private readonly eventBus: DomainEventBus,
    private readonly clock: Clock,
  ) {}

  async confirmRedemption(input: ConfirmRedemptionInput): Promise<RedemptionConfirmedResponse> {
    // 1. Validate signed QR token
    const payload = this.tokenSigner.verify(input.qrToken);
    if (!payload.ok) throw new InvalidQrTokenError();

    // 2. Load coupon
    const coupon = await this.couponRepo.findByIdForUpdate(payload.value.couponId);
    if (!coupon) throw new CouponNotFoundError();

    // 3. Domain rule checks (pure logic in entity)
    coupon.assertEligibleForRedemption({
      branchId: input.branchId,
      now: this.clock.now(),
      pin: input.pin,
    });

    // 4. Atomic state change in transaction
    const redemption = await this.prisma.$transaction(async (tx) => {
      coupon.markRedeemed({ branchId: input.branchId, cashierId: input.cashierId });
      await this.couponRepo.save(coupon, tx);
      const redemption = Redemption.create({
        couponId: coupon.id,
        branchId: input.branchId,
        cashierId: input.cashierId,
        deviceFingerprint: input.deviceFingerprint,
        at: this.clock.now(),
      });
      await this.redemptionRepo.save(redemption, tx);
      return redemption;
    });

    // 5. Emit event for side effects
    await this.eventBus.publish(new CouponRedeemedEvent({
      couponId: coupon.id,
      orderId: coupon.orderId,
      merchantId: coupon.merchantId,
      branchId: input.branchId,
      cashierId: input.cashierId,
      payoutAmount: coupon.payoutAmount,
      at: redemption.at,
    }));

    // 6. Return response
    return RedemptionConfirmedResponse.from(coupon, redemption);
  }
}
```

```typescript
// coupon.entity.ts (domain — pure TypeScript)
export class Coupon {
  // ... fields ...

  assertEligibleForRedemption(ctx: { branchId: string; now: Date; pin: string }): void {
    if (this.status !== CouponStatus.Active) {
      throw new CouponNotActiveError(this.status);
    }
    if (this.expiresAt < ctx.now) {
      throw new CouponExpiredError(this.expiresAt);
    }
    if (this.usesRemaining <= 0) {
      throw new CouponNoUsesLeftError();
    }
    if (!this.eligibleBranchIds.includes(ctx.branchId)) {
      throw new BranchNotEligibleError(ctx.branchId);
    }
    if (!isWithinValidHours(this.redemptionRules, ctx.now, this.timezone)) {
      throw new OutsideValidHoursError();
    }
    if (!this.pinMatcher.matches(ctx.pin)) {
      this.pinAttempts += 1;
      if (this.pinAttempts >= 3) {
        this.lockUntil = addMinutes(ctx.now, 15);
        throw new CouponLockedError();
      }
      throw new WrongPinError(3 - this.pinAttempts);
    }
  }

  markRedeemed(ctx: { branchId: string; cashierId: string }): void {
    this.usesRemaining -= 1;
    if (this.usesRemaining === 0) {
      this.status = this.usageType === UsageType.SingleUse ? CouponStatus.Redeemed : CouponStatus.UsedUp;
    }
    this.lastRedeemedAt = new Date();
  }
}
```

This pattern — thin controller, orchestrating service, pure domain entity — is the same across all modules.

---

## Idempotency

Every mutating endpoint accepts an `Idempotency-Key` header (UUID v4 from client). The `IdempotencyInterceptor`:

1. Reads the key.
2. Hashes (key + user ID + endpoint + body) to form a Redis cache key.
3. If hit: returns the cached response.
4. If miss: proceeds; on success, caches the response with 24h TTL.

For payment-creating endpoints, the idempotency key is also passed to Stripe so retries never double-charge.

---

## Transactions

We use Prisma's interactive transactions for any multi-step state change. Rules:

- Long-running external calls (Stripe, SMS) **never** happen inside a transaction.
- Maximum transaction duration: 5 seconds. Anything longer → break into compensating actions.
- Use `SELECT FOR UPDATE` (`Prisma.$queryRaw` if needed) on rows that have race conditions (inventory, balances).

---

## Domain events

Two-tier event system:

1. **In-process events** (synchronous within the same request): NestJS's built-in event emitter. Used for fire-and-forget side effects that *must* happen before the request returns (e.g., audit log).
2. **Out-of-process events** (asynchronous): Redis Streams + BullMQ. Subscribers run in worker processes. Used for everything else (notifications, search index updates, ad event forwarding).

All events are typed:

```typescript
abstract class DomainEvent {
  readonly eventId: string = uuidv7();
  readonly occurredAt: Date = new Date();
  abstract readonly eventName: string;
  abstract readonly version: number;
}

class CouponRedeemedEvent extends DomainEvent {
  readonly eventName = 'coupon.redeemed';
  readonly version = 1;

  constructor(
    public readonly payload: {
      couponId: string;
      orderId: string;
      merchantId: string;
      branchId: string;
      cashierId: string;
      payoutAmount: Money;
      at: Date;
    },
  ) {
    super();
  }
}
```

Event versioning: when payload shape changes, increment `version`. Subscribers handle multiple versions during a transition window.

---

## Configuration

Strongly-typed config loaded once at bootstrap from environment variables:

```typescript
@Injectable()
export class AppConfig {
  @IsString() @IsNotEmpty() readonly NODE_ENV: 'development' | 'staging' | 'production';
  @IsString() @IsNotEmpty() readonly DATABASE_URL: string;
  @IsString() @IsNotEmpty() readonly REDIS_URL: string;
  @IsString() @IsNotEmpty() readonly JWT_PRIVATE_KEY: string;
  @IsString() @IsNotEmpty() readonly JWT_PUBLIC_KEY: string;
  @IsInt() @Min(60) @Max(86400) readonly JWT_ACCESS_TTL_SECONDS: number = 900;
  // ... 80+ more
}
```

If any required config is missing or malformed, the app fails to start. No silent defaults.

---

## Error handling

Three error classes:

1. **DomainError** — business rule violation; HTTP 4xx; specific error code mapped to user-facing message.
2. **InfrastructureError** — external dependency failure; HTTP 5xx (or specific 5xx); retryable.
3. **UnknownError** — unexpected; HTTP 500; logged as error with full context; reported to Sentry.

Global exception filter:

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    if (exception instanceof DomainError) {
      return response.status(exception.httpStatus).json({
        error: {
          code: exception.code,
          message: exception.message,
          messageAr: this.translate(exception.code, 'ar'),
          context: exception.context,
        },
        traceId: request.id,
      });
    }

    if (exception instanceof InfrastructureError) {
      this.logger.warn({ exception, traceId: request.id }, 'Infrastructure error');
      return response.status(exception.httpStatus).json({
        error: {
          code: exception.code,
          message: 'Service temporarily unavailable. Please try again.',
        },
        traceId: request.id,
      });
    }

    // Unknown
    this.logger.error({ exception, traceId: request.id }, 'Unhandled error');
    Sentry.captureException(exception);
    return response.status(500).json({
      error: { code: 'INTERNAL_ERROR', message: 'Something went wrong. We have been notified.' },
      traceId: request.id,
    });
  }
}
```

Error code catalogue lives in `shared/errors/codes.ts`. Every code has EN + AR copy.

---

## Database access patterns

### Read patterns

| Pattern | Use case | Implementation |
|---|---|---|
| Direct read by primary key | Hot lookup | Prisma; cached in Redis with TTL 5 min for hot entities |
| Indexed read (e.g., user's orders) | List view | Prisma; backed by composite indexes |
| Aggregation (e.g., merchant dashboard KPIs) | Dashboards | Materialised views refreshed every minute |
| Complex analytics | Admin reports | Read replica, written in raw SQL or via reporting service |

### Write patterns

| Pattern | Use case | Implementation |
|---|---|---|
| Single-row insert/update | Most writes | Prisma standard methods |
| Multi-row transaction | Order + coupons + ledger | Prisma `$transaction` |
| Optimistic concurrency | Inventory decrement | `UPDATE ... WHERE inventory_left > 0 RETURNING` |
| Pessimistic locking | Coupon redemption | `SELECT ... FOR UPDATE` |
| Write-and-emit-event | Most domain operations | Outbox pattern (write event row in same TX, publish from outbox worker) |

### Outbox pattern (for reliable event publication)

```sql
-- inside the same transaction as the business write:
INSERT INTO outbox_events (id, event_name, payload, created_at) VALUES (...);
```

A separate `OutboxWorker` polls the outbox every second, publishes to Redis Streams, and marks rows published. This guarantees no event is lost if Redis is down at write time.

---

## Caching strategy

| Cache key | TTL | Invalidation |
|---|---|---|
| `deal:{id}` | 5 min | Event-driven on deal update |
| `merchant:{id}` | 10 min | Event-driven on merchant profile change |
| `category:tree` | 1 hour | Manual on admin change |
| `user:permissions:{id}` | 5 min | Event-driven on role change |
| `feature_flags` | 60s | Pub/sub broadcast |
| `meilisearch:results:{query_hash}` | 1 min | Time-based only |

Cache layer: Redis. Library: a thin wrapper around `ioredis` with consistent key prefixing, JSON serialisation, and stale-while-revalidate semantics.

---

## Background workers

Workers are separate ECS Fargate tasks consuming BullMQ queues. Each worker is single-purpose:

| Worker | Trigger | Concurrency |
|---|---|---|
| coupon-expiry | Cron every 5 min | 1 |
| expiry-reminders | Cron hourly | 1 |
| stripe-reconcile | Cron daily 02:00 | 1 |
| notification-delivery | Event-driven | Auto-scale on queue depth |
| search-index | Event-driven | 4 |
| image-processing | Event-driven | 2 |
| ads-event-forward | Event-driven | 4 |
| outbox-publisher | Continuous polling | 2 |
| document-expiry-check | Cron daily | 1 |
| raffle-draw | Cron at draw times | 1 |

Each worker has graceful shutdown (in-flight jobs allowed to finish), idempotent job processing (replays don't double-effect), and dead-letter queue for failed jobs after 5 retries with exponential backoff.

---

## Webhooks (inbound)

Stripe, Tabby, Tamara, WhatsApp send webhooks to dedicated endpoints. Pattern:

1. Webhook endpoint verifies signature using each provider's mechanism.
2. Returns 200 immediately if signature valid.
3. Persists the webhook payload to an `inbound_webhooks` table.
4. Enqueues a worker job to process the webhook.

This decouples webhook latency from processing latency and ensures we never lose a webhook.

---

## Webhooks (outbound)

We expose webhook subscriptions for partners (e.g., merchant ERP integrations). Pattern:

1. Partner registers a webhook URL + events of interest.
2. We sign each delivery with HMAC-SHA256 of the body.
3. We retry on failure with exponential backoff up to 24 hours.
4. After 24h of failure, deliveries are paused and partner notified.

Detailed in [05-api/04-webhooks.md](../05-api/04-webhooks.md).

---

## Health & readiness

```typescript
@Controller()
export class HealthController {
  @Get('health')   // liveness — is the process alive?
  health() { return { status: 'ok' }; }

  @Get('ready')    // readiness — can it serve traffic?
  async ready() {
    await Promise.all([
      this.prisma.$queryRaw`SELECT 1`,
      this.redis.ping(),
      this.meilisearch.health(),
    ]);
    return { status: 'ready' };
  }
}
```

ALB uses `/ready` for routing decisions.

---

## API versioning

URL versioning: `/api/v1/...`. Breaking changes require `/api/v2/...` with at least 6 months of overlap. Mobile clients call only the version their build was tested against; admin/server-to-server can call latest.

---

## See also

- [Mobile Architecture](./04-mobile-architecture.md)
- [Event-Driven Design](./06-event-driven-design.md)
- [Data Model](../04-data-model/01-erd-overview.md)
- [API Design Principles](../05-api/01-api-design-principles.md)
