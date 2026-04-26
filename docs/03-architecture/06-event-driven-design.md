# 06 · Event-Driven Design

**Status:** 🟢 Approved

How modules communicate via domain events, the catalogue of events, and the rules for evolving them.

---

## Why events

Events let one module react to changes in another without tight coupling. When a coupon is purchased, the Coupons module doesn't need to know about Notifications, Raffles, Wallet, Ads — it just emits `coupon.purchased`, and each interested module subscribes.

Benefits:
- **Decoupling**: modules added/removed without changing the producer.
- **Auditability**: every event is a stored fact.
- **Reliability**: events survive restart and retry.
- **Extensibility**: new behaviours added by adding subscribers.

---

## Event delivery tiers

We use two tiers based on consistency requirements:

### Tier 1 — In-process (synchronous, transactional)

Used when the side effect must happen as part of the same logical operation and we cannot tolerate it being delayed or lost.

- Implementation: NestJS `EventEmitterModule`.
- Same database transaction as the producer.
- Subscribers run in the same request lifecycle.
- Failure of subscriber = failure of producer (transaction rolls back).

**Examples:**
- Audit log entry on every privileged action.
- Inventory decrement when an order is created.

### Tier 2 — Out-of-process (asynchronous, eventually consistent)

Used for everything else. Subscribers are decoupled from the request lifecycle.

- Implementation: **Outbox pattern** writing to PostgreSQL, with an **OutboxPublisher** worker forwarding to **Redis Streams**.
- Subscribers run in BullMQ workers.
- At-least-once delivery; subscribers must be idempotent.

**Examples:**
- Send push notification when a coupon is purchased.
- Update Meilisearch index when a deal is approved.
- Forward conversion event to Meta + TikTok.
- Update merchant ledger when a coupon is redeemed.

---

## The Outbox pattern (in detail)

Problem: if we write a row and emit an event in two separate operations, either could fail independently — leaving the system inconsistent.

Solution: write the event into an `outbox_events` table **in the same transaction** as the business write. A worker polls the outbox and publishes to Redis Streams.

### Schema

```sql
CREATE TABLE outbox_events (
  id              UUID PRIMARY KEY,
  event_name      TEXT NOT NULL,
  event_version   INT NOT NULL,
  payload         JSONB NOT NULL,
  occurred_at     TIMESTAMPTZ NOT NULL,
  published_at    TIMESTAMPTZ,
  publish_attempts INT NOT NULL DEFAULT 0,
  last_error      TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_unpublished
  ON outbox_events (created_at)
  WHERE published_at IS NULL;
```

### Producer

```ts
@Injectable()
export class OrderService {
  async confirmPayment(orderId: string) {
    return this.prisma.$transaction(async (tx) => {
      const order = await this.orderRepo.markPaid(orderId, tx);
      const coupons = await this.couponService.issueForOrder(order, tx);

      await tx.outbox_events.create({
        data: {
          id: uuidv7(),
          event_name: 'coupon.purchased',
          event_version: 1,
          payload: { orderId: order.id, couponIds: coupons.map(c => c.id) },
          occurred_at: new Date(),
        },
      });

      return { order, coupons };
    });
  }
}
```

### Publisher worker

A small worker polls every 1 second:

```ts
async function publishOutbox() {
  const batch = await prisma.outbox_events.findMany({
    where: { published_at: null, publish_attempts: { lt: 5 } },
    orderBy: { created_at: 'asc' },
    take: 100,
  });

  for (const event of batch) {
    try {
      await redisStreams.add(event.event_name, {
        id: event.id,
        version: event.event_version,
        occurred_at: event.occurred_at,
        payload: event.payload,
      });
      await prisma.outbox_events.update({
        where: { id: event.id },
        data: { published_at: new Date() },
      });
    } catch (err) {
      await prisma.outbox_events.update({
        where: { id: event.id },
        data: {
          publish_attempts: { increment: 1 },
          last_error: String(err),
        },
      });
    }
  }
}
```

After 5 failed attempts, the event is sent to a dead-letter table for manual investigation.

---

## Event naming convention

`<aggregate>.<past-tense-verb>`

- `coupon.purchased`
- `coupon.redeemed`
- `coupon.expired`
- `coupon.refunded`
- `merchant.kyc_approved`
- `merchant.kyc_rejected`
- `merchant.suspended`
- `deal.submitted`
- `deal.approved`
- `deal.rejected`
- `deal.paused`
- `deal.expired`
- `order.paid`
- `order.refunded`
- `payout.requested`
- `payout.approved`
- `payout.paid`
- `payout.failed`
- `raffle.entry_created`
- `raffle.draw_completed`
- `raffle.winner_notified`
- `raffle.prize_fulfilled`
- `review.submitted`
- `review.published`
- `notification.sent`
- `chargeback.received`

---

## Event versioning

Every event has a `version` integer. When the payload shape changes incompatibly:

1. Bump the version (`coupon.purchased` v1 → v2).
2. Producer emits both v1 and v2 during the transition window.
3. Subscribers read both; new subscribers handle v2; old subscribers continue handling v1 until upgraded.
4. Once all subscribers handle v2, producer drops v1.

For backward-compatible additions (new optional fields), no version bump.

---

## Subscriber rules

1. **Idempotent.** A given event ID processed twice must produce the same end state.
2. **Fast.** A single subscriber should complete in < 5 seconds. Longer work goes into a child job.
3. **Independent.** Subscriber A failing must not prevent Subscriber B from running.
4. **Logged.** Every subscriber logs `event_received` and `event_processed` (or `event_failed`) with the event ID for traceability.
5. **Tested.** Every subscriber has unit tests for happy path, retry, and idempotency.

---

## Event catalogue (with subscribers)

### `coupon.purchased`
**Producer:** Commerce module on Stripe webhook → order paid.

| Subscriber | Action |
|---|---|
| Coupons module | Generate coupon records, QR tokens, PINs |
| Wallet module | Debit merchant `pending` ledger |
| Notifications module | Send push + email + in-app with coupons |
| Raffles module | Evaluate raffle rules, create entries |
| Loyalty module | Award loyalty points (issued, redeemed on success) |
| Ads module | Forward Purchase event to Meta + TikTok APIs |
| Reporting module | Update materialised views for dashboards |
| Ads module (referral) | Track referral attribution |

### `coupon.redeemed`
**Producer:** Redemption module on successful PIN validation.

| Subscriber | Action |
|---|---|
| Wallet module | Schedule `pending` → `available` transition after hold period |
| Notifications module | Push to user (post-redemption review prompt at +24h) |
| Notifications module | Add to merchant's daily redemption summary |
| Reviews module | Schedule review-prompt job for +24h |
| Reporting module | Update redemption metrics |

### `coupon.expired`
**Producer:** Coupon expiry worker (scheduled job).

| Subscriber | Action |
|---|---|
| Wallet module | Reverse merchant `pending` row; recognise breakage revenue |
| Notifications module | Send "expired" notification |
| Reporting module | Update breakage metrics |

### `merchant.kyc_approved`
**Producer:** Identity module on admin approval.

| Subscriber | Action |
|---|---|
| Notifications module | Email + in-app: "You're live!" |
| Catalog module | Mark merchant eligible to publish deals |

### `deal.approved`
**Producer:** Catalog module on admin approval.

| Subscriber | Action |
|---|---|
| Search module | Index deal in Meilisearch |
| Notifications module | Notify merchant; notify followers if scheduled |
| Ads module | Make deal available for "Promote this" |

### `deal.expired`
**Producer:** Deal lifecycle worker.

| Subscriber | Action |
|---|---|
| Search module | Remove from Meilisearch |
| Notifications module | Notify followers (optional) |

### `payout.paid`
**Producer:** Payouts module on bank confirmation.

| Subscriber | Action |
|---|---|
| Invoicing module | Generate PDF statement |
| Invoicing module | Issue commission invoice (Volu → merchant) |
| Notifications module | Notify merchant of payout |

### `chargeback.received`
**Producer:** Payments module on Stripe `charge.dispute.created` webhook.

| Subscriber | Action |
|---|---|
| Wallet module | Freeze merchant balance equal to disputed amount |
| Notifications module | Alert admin via Slack + in-app |
| Audit module | Open dispute case |

(Full catalogue maintained in code at `packages/events/catalog.ts`.)

---

## Replay & backfill

Sometimes a new subscriber wants to process historical events (e.g., when a new analytics service is added). We support:

1. Reading from `outbox_events` directly with a date range.
2. Replaying events through the new subscriber only.
3. Marking a "replay sentinel" so subscribers know it's a replay (vs live).

Replays must respect idempotency.

---

## Observability

- Every event publish + subscribe is traced via OpenTelemetry.
- Event lag (time from `occurred_at` to subscriber `processed_at`) is a key SRE metric.
- Stuck outbox rows (> 5 min unpublished) page the on-call.
- Dead-letter queue size is monitored.

---

## When NOT to use events

- **Synchronous request-response semantics**: if the caller needs an immediate result, call the service directly.
- **Strong consistency requirements within a single transaction**: use Tier 1 in-process events or just call the service.
- **Simple internal calls**: don't event-ify a "set this field" operation.

Events are powerful but add asynchrony. Reach for them when decoupling adds value.

---

## See also

- [Backend Architecture](./03-backend-architecture.md)
- [System Architecture](./01-system-architecture.md)
