# 03 · SLA & SLOs

**Status:** 🟢 Approved

How Volu measures reliability, the targets we hold ourselves to, and the contractual commitments we make to merchants.

---

## SLI vs SLO vs SLA — quick reminder

- **SLI** (Service Level Indicator) — what you measure. E.g., "% of HTTP requests returning 2xx/3xx."
- **SLO** (Service Level Objective) — internal target. E.g., "99.9% of requests succeed."
- **SLA** (Service Level Agreement) — external commitment, often with consequences for missing. Typically slightly looser than the SLO so we have a safety buffer.

**Rule:** SLA < SLO. Always.

---

## Service tiers

We classify surfaces by criticality:

| Tier | Surface | Why |
|---|---|---|
| **Tier 1 — Money-critical** | Payments, Redemption, Coupon-issuance, Payouts | Direct financial impact if broken |
| **Tier 2 — Core experience** | Auth, Catalog read, Cart, Search, Notifications | Users can't function without them |
| **Tier 3 — Supporting** | Reviews, Favourites, Loyalty, Recommendations | Degradation tolerated briefly |
| **Tier 4 — Internal** | Admin console, Reporting, Analytics | Internal users; SLAs informal |

---

## Service Level Objectives

### Tier 1 — Money-critical

| SLI | SLO | Window |
|---|---|---|
| Payment success rate (Stripe / Tabby / Tamara) | ≥ 99.95% | 30 days rolling |
| `POST /orders` availability | ≥ 99.95% | 30 days |
| `POST /redemptions/validate` availability | ≥ 99.95% | 30 days |
| `POST /redemptions/confirm` availability | ≥ 99.95% | 30 days |
| `POST /redemptions/validate` latency P95 | ≤ 1500 ms | 30 days |
| `POST /redemptions/confirm` latency P95 | ≤ 1500 ms | 30 days |
| Coupon issuance from payment success | ≤ 5 s P99 | 30 days |
| Payout queue → bank reference | ≤ 1 business day P95 | 30 days |
| Stripe webhook → ledger update | ≤ 30 s P95 | 30 days |

### Tier 2 — Core experience

| SLI | SLO | Window |
|---|---|---|
| API availability (overall, public endpoints) | ≥ 99.9% | 30 days |
| API latency P95 (read endpoints) | ≤ 500 ms | 30 days |
| API latency P95 (write endpoints) | ≤ 800 ms | 30 days |
| Search latency P95 | ≤ 300 ms | 30 days |
| Push notification delivery (FCM/APNs ack) | ≥ 99% | 30 days |
| Push notification delivery latency P95 | ≤ 60 s from event | 30 days |
| OTP delivery rate | ≥ 98% | 30 days |

### Tier 3 — Supporting

| SLI | SLO | Window |
|---|---|---|
| API availability | ≥ 99.5% | 30 days |
| API latency P95 | ≤ 1 s | 30 days |
| Reviews moderation throughput | ≤ 24h P95 | 30 days |

### Tier 4 — Internal

| SLI | SLO | Window |
|---|---|---|
| Admin availability (business hours) | ≥ 99.5% | 30 days |
| Reporting freshness | ≤ 15 min lag | continuous |

---

## Error budget

Every SLO implies an error budget — the allowed unreliability.

**Examples:**

- Tier 1 SLO of 99.95% over 30 days → error budget = 21 minutes 36 seconds of downtime per month.
- Tier 2 SLO of 99.9% over 30 days → error budget = 43 minutes 12 seconds per month.
- Tier 3 SLO of 99.5% over 30 days → error budget = 3 hours 36 minutes per month.

### Budget burn policy

| Burn rate | Action |
|---|---|
| < 50% of budget consumed mid-month | Normal operation |
| 50–80% consumed mid-month | Yellow alert: pause non-essential changes; review recent deploys |
| > 80% consumed mid-month | Red alert: full freeze on non-critical changes; root-cause review; SEV-2 incident |
| > 100% consumed | SLO miss for the month: post-mortem mandatory; published in retrospective |

Specific automated alerts:

- Burn rate > 14× normal over 1 hour → page (will exhaust monthly budget in 2 days).
- Burn rate > 6× normal over 6 hours → page (will exhaust monthly budget in 1 week).

---

## Measurement methodology

### Availability

```
availability = successful_requests / total_requests
```

Where:
- "Successful" = HTTP 2xx or 3xx, OR 4xx that's not a server error (e.g., 401, 422 are user errors, not our fault).
- "Total" excludes synthetic monitor traffic and excludes retries from the same idempotency key.
- Measured at the load balancer (post-WAF, pre-app).

### Latency

P50/P95/P99 from histograms in Prometheus, with metric:
```
http_request_duration_seconds_bucket{path, method}
```

Latency excludes:
- Time spent in 3DS challenges (controlled by issuing bank).
- Time the user spends entering data (only server time matters).

### Payment success rate

```
success_rate = paid_orders / total_attempted_orders
```

Where "attempted" = Stripe `payment_intent` created. We exclude orders abandoned at the cart screen (user dropped off before submitting).

---

## Service Level Agreements (external)

For merchants and select partners, we publish SLAs. These are looser than internal SLOs and cover specific endpoints.

### Merchant-facing SLA

| Commitment | Target |
|---|---|
| Platform availability (per calendar month) | 99.5% |
| Coupon redemption endpoint availability | 99.9% |
| Coupon redemption latency P95 | < 2 s |
| Payout queue processing | ≤ 1 business day from approval |
| Support response — SEV-1 (merchant operations down) | ≤ 1 hour |
| Support response — SEV-2 (specific feature broken) | ≤ 4 hours |
| Support response — SEV-3 (everything else) | ≤ 1 business day |

### Service credits

If we miss an SLA in any month:

| Monthly availability | Service credit (% of fees waived) |
|---|---|
| < 99.5% but ≥ 99% | 10% |
| < 99% but ≥ 95% | 25% |
| < 95% | 50% |

(Calculated against the merchant's commission paid to Volu in that month.)

Exclusions: scheduled maintenance (advance notice given), force majeure, third-party outages outside Volu's reasonable control.

---

## Status page

`status.volu.ae` shows:

- Current status of every Tier 1 + Tier 2 surface.
- Active incidents with timeline.
- Past 90 days of uptime per surface.
- Subscription option for email/SMS notifications on incidents.

Updated automatically from synthetic monitor results, with manual overrides during declared incidents.

---

## Reporting cadence

| Report | Frequency | Audience |
|---|---|---|
| Weekly SLO report | Weekly | Engineering team |
| Monthly SLO retrospective | Monthly | Engineering + leadership |
| Quarterly reliability review | Quarterly | All-hands |
| Annual reliability summary | Annually | Public-facing (anonymised) |

The monthly retrospective covers:
- Did we meet each SLO?
- If not, why?
- Action items.
- Were error budgets healthy?
- Any SLO targets that need adjustment (loosen or tighten based on user impact)?

---

## Tightening over time

These SLOs are starting points, calibrated for a year-1 startup. As we scale:

- Year 2 target: tighten Tier 1 to 99.99% (4-nines).
- Add per-region SLOs once we expand beyond UAE.
- Add per-category SLOs (e.g., dedicated SLOs for the BNPL flow).

---

## Anti-patterns we avoid

- **SLO theatre**: green dashboards while users are frustrated. Our SLOs are based on real user-impact metrics, not internal abstractions.
- **One-size-fits-all SLOs**: a 99% SLO on a tier-1 surface is too loose; on a tier-3 surface it's too strict.
- **Punitive culture**: missed SLOs prompt root-cause investigation, not blame.
- **Gaming**: never re-define "successful" to make numbers look good.

---

## See also

- [Observability](../07-infrastructure/05-observability.md)
- [Runbooks](./02-runbooks.md)
- [On-call](./04-on-call.md)
- [Non-Functional Requirements](../02-requirements/02-non-functional-requirements.md)
