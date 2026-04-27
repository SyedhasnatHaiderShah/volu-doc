# 04 · Webhooks

**Status:** 🟢 Approved

How Volu sends and receives webhooks.

---

## Inbound webhooks (Volu receives)

We receive webhooks from Stripe, Tabby, Tamara, WhatsApp Business API, Meta, and TikTok.

### Pattern

```
Provider → POST /api/v1/webhooks/<provider>
  ↓
1. API Gateway routes to webhook handler.
2. Handler verifies signature using provider's mechanism.
3. If signature fails → 401, log, alert.
4. If signature passes:
     - Persist payload to inbound_webhooks table (with provider_event_id for dedup).
     - If provider_event_id already processed → return 200 immediately.
     - Else enqueue worker job; return 200 immediately.
5. Worker processes the event asynchronously:
     - Looks up affected entities.
     - Updates state.
     - Emits domain events.
6. On worker failure: retry with exponential backoff up to 5 attempts.
7. After max retries: send to DLQ; alert ops.
```

**Why return 200 immediately:** providers retry aggressively if we don't ack within ~5 seconds. Decoupling response from processing prevents cascading failures.

---

### Stripe webhook handling

Critical events:

| Event                           | Action                                         |
| ------------------------------- | ---------------------------------------------- |
| `payment_intent.succeeded`      | Mark order paid; emit `order.paid`             |
| `payment_intent.payment_failed` | Mark order failed; release inventory           |
| `charge.refunded`               | Reconcile refund record                        |
| `charge.dispute.created`        | Open dispute case; freeze merchant balance     |
| `charge.dispute.closed`         | Resolve dispute; unfreeze or finalise refund   |
| `payout.paid`                   | (Stripe Connect) Mark merchant payout paid     |
| `payout.failed`                 | Mark payout failed; refund balance to merchant |
| `account.updated` (Connect)     | Sync merchant Connect account state            |

Signature verification:

```ts
const event = stripe.webhooks.constructEvent(
  rawBody,
  req.headers["stripe-signature"],
  env.STRIPE_WEBHOOK_SECRET,
);
```

---

### Tabby webhook handling

| Event                | Action                                                     |
| -------------------- | ---------------------------------------------------------- |
| `payment.created`    | Order moves to `pending` (already in this state)           |
| `payment.authorized` | Mark order paid; trigger same fulfilment as Stripe success |
| `payment.closed`     | Settled                                                    |
| `payment.rejected`   | Order failed                                               |

Signature: HMAC of body using webhook secret.

---

### Tamara webhook handling

Similar to Tabby. Tamara sends `order_approved`, `order_rejected`, `order_canceled`, `payment_capture`, `payment_failure`.

---

### WhatsApp webhook

Two kinds:

- **Delivery status** — for our outbound WhatsApp messages.
- **Inbound replies** — when a user replies to our WhatsApp message; we route to support.

Signature: WhatsApp Business API verifies via challenge token initially; ongoing requests signed with `X-Hub-Signature-256`.

---

## Outbound webhooks (Volu sends)

For partners (e.g., merchant ERP integrations) who want to react to Volu events.

### Subscription model

Admin creates a subscription:

```http
POST /api/v1/admin/webhook-subscriptions
{
  "url": "https://partner.example.com/volu",
  "events": ["coupon.redeemed", "payout.paid"],
  "secret": "<base64 random 32 bytes>"
}
```

Subscription scoped to either platform-wide or a specific merchant.

### Delivery format

```http
POST /partner-url HTTP/1.1
Content-Type: application/json
X-Volu-Event: coupon.redeemed
X-Volu-Event-Id: 01HXX...
X-Volu-Signature: t=1716200000,v1=<hex_hmac>
X-Volu-Delivery-Id: 01HXY...

{
  "event": "coupon.redeemed",
  "version": 1,
  "occurredAt": "2026-04-23T14:30:00+04:00",
  "data": {
    "couponId": "01HXX...",
    "orderId": "01HXY...",
    "merchantId": "01HXZ...",
    "branchId": "01HXA...",
    "redeemedAt": "2026-04-23T14:29:55+04:00",
    "payoutAmount": { "amountFils": 12000, "currency": "AED" }
  }
}
```

### Signature

```
sigBase = `${timestamp}.${rawBody}`
sig = HMAC-SHA256(subscription.secret, sigBase)
header = `t=${timestamp},v1=${hex(sig)}`
```

Receiver validates:

1. Parse `t` and `v1` from header.
2. Recompute HMAC; constant-time compare.
3. Reject if `now - t > 5 minutes` (replay protection).

### Retry policy

If the partner endpoint returns non-2xx or times out:

| Attempt | Delay     |
| ------- | --------- |
| 1       | immediate |
| 2       | 1 min     |
| 3       | 5 min     |
| 4       | 15 min    |
| 5       | 1 hour    |
| 6       | 6 hours   |
| 7       | 24 hours  |

After 7 failures, the subscription is paused; partner notified.

### Delivery log

Every attempt logged: subscription_id, event_id, attempt_number, status_code, response_body (truncated), latency_ms, attempted_at.

Admin UI shows delivery history per subscription with replay-individual-event button.

---

## Standard event payloads

### `coupon.purchased`

```json
{
  "couponId": "01HXX...",
  "orderId": "01HXY...",
  "dealId": "01HXZ...",
  "merchantId": "01HXA...",
  "userId": "01HXB...",
  "shortCode": "...",
  "expiresAt": "2026-05-31T23:59:59+04:00",
  "payoutAmount": { "amountFils": 12000, "currency": "AED" }
}
```

### `coupon.redeemed`

```json
{
  "couponId": "01HXX...",
  "orderId": "01HXY...",
  "merchantId": "01HXZ...",
  "branchId": "01HXA...",
  "cashierUserId": "01HXB...",
  "redeemedAt": "2026-04-23T14:29:55+04:00",
  "usesRemaining": 0,
  "payoutAmount": { "amountFils": 12000, "currency": "AED" }
}
```

### `coupon.expired`

```json
{
  "couponId": "01HXX...",
  "merchantId": "01HXZ...",
  "expiredAt": "2026-05-31T23:59:59+04:00",
  "wasRedeemedTimes": 0,
  "breakageAmount": { "amountFils": 15000, "currency": "AED" }
}
```

### `coupon.refunded`

```json
{
  "couponId": "01HXX...",
  "orderId": "01HXY...",
  "refundId": "01HXZ...",
  "refundedAt": "2026-04-24T10:00:00+04:00",
  "amount": { "amountFils": 15000, "currency": "AED" },
  "reason": "user_self_service"
}
```

### `payout.paid`

```json
{
  "payoutId": "01HXX...",
  "merchantId": "01HXZ...",
  "amount": { "amountFils": 250000, "currency": "AED" },
  "bankReference": "TXN-...",
  "paidAt": "2026-04-23T16:00:00+04:00"
}
```

(Full catalogue lives in OpenAPI / event registry code.)

---

## See also

- [API Design Principles](./01-api-design-principles.md)
- [Event-Driven Design](../03-architecture/06-event-driven-design.md)
