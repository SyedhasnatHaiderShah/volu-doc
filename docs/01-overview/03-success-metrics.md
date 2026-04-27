# 03 · Success Metrics

**Status:** 🟢 Approved

What we measure, why it matters, and what "good" looks like. These metrics drive the analytics schema, the admin dashboard, and every quarterly review.

---

## The North Star

**Monthly Successful Redemptions** — the count of coupons successfully redeemed at merchants in a calendar month.

**Why this and not GMV or MAU:**

- GMV alone is vanity — a sold-but-never-redeemed coupon is a short-term win and a long-term trust problem.
- MAU alone is vanity — we could have millions of browsers and zero business.
- A successful redemption means: merchant happy, user happy, money flowing, platform earning commission, breakage captured on the rest.

**12-month target:** 150,000 monthly successful redemptions.

---

## Input Metrics (levers we pull)

### Supply-side

| Metric                    | Definition                                                   | 12-month target     |
| ------------------------- | ------------------------------------------------------------ | ------------------- |
| Verified Active Merchants | Merchants with ≥1 active deal in the last 30 days            | 300+                |
| Live Deals                | Deals with `status = approved` and within valid-until window | 800+                |
| New Deals/Week            | Deals published per week                                     | 80+                 |
| Deal Approval SLA         | Median time from submission to approval decision             | < 24 business hours |
| Merchant Churn (monthly)  | Merchants who had a live deal 60 days ago but not today      | < 5%                |

### Demand-side

| Metric                     | Definition                                            | 12-month target |
| -------------------------- | ----------------------------------------------------- | --------------- |
| Monthly Active Users (MAU) | Distinct users who opened the app in a calendar month | 75,000          |
| Daily Active Users (DAU)   | Distinct users who opened the app that day            | 15,000          |
| DAU/MAU                    | Stickiness ratio                                      | ≥ 20%           |
| New Users/Month            | First-time registrations                              | 15,000          |
| Install → First Purchase   | Conversion rate from install to first coupon          | 8–12%           |

### Transaction

| Metric                     | Definition                             | 12-month target |
| -------------------------- | -------------------------------------- | --------------- |
| GMV (monthly)              | Sum of all coupon sales before refunds | AED 8M+         |
| Orders/Month               | Count of checkout-complete orders      | 50,000          |
| Average Order Value (AOV)  | GMV ÷ orders                           | AED 140–180     |
| Repeat Purchase Rate (30d) | % of users with ≥2 orders in 30 days   | ≥ 35%           |
| Payment Success Rate       | Successful charges ÷ attempted charges | ≥ 96%           |

---

## Output Metrics (results we see)

### Revenue

| Metric                 | Definition                                       | Target                      |
| ---------------------- | ------------------------------------------------ | --------------------------- |
| Net Platform Revenue   | Commission + breakage + featured + ads − refunds | AED 2.0M+/month by month 12 |
| Blended Take Rate      | Net revenue ÷ GMV                                | 28–35%                      |
| Breakage Revenue Share | Breakage revenue ÷ total revenue                 | 25–35%                      |
| Ad-Promoted GMV Share  | GMV from Meta/TikTok promoted deals ÷ total GMV  | 20–30% by month 12          |

### Quality / Trust

| Metric                             | Definition                                               | Target    |
| ---------------------------------- | -------------------------------------------------------- | --------- |
| Redemption Rate                    | Redeemed coupons ÷ sold coupons (within expiry)          | 65–80%    |
| Refund Rate                        | Refunded orders ÷ total orders                           | < 3%      |
| Chargeback Rate                    | Chargebacks ÷ total orders                               | < 0.5%    |
| Merchant Rejection Incidents       | Valid coupons rejected at merchant (per 10k redemptions) | < 5       |
| Support Ticket Resolution (median) | Time from ticket open to close                           | < 4 hours |
| App Store Rating                   | iOS + Android average                                    | ≥ 4.5     |

---

## Guardrail Metrics (don't break these)

If any of these breach, something is wrong — pause growth efforts and investigate.

| Guardrail                  | Threshold          | Action if breached                                    |
| -------------------------- | ------------------ | ----------------------------------------------------- |
| Payment Success Rate       | < 94%              | Alert + investigate gateway / 3DS issues              |
| Redemption Rate            | < 55%              | Audit merchant quality and deal expiry comms          |
| Refund Rate                | > 5%               | Audit merchant quality; tighten deal approval         |
| Chargeback Rate            | > 0.8%             | Tighten Stripe Radar rules; review disputed merchants |
| Support Tickets/Day        | > 3% of DAU        | Investigate product bugs or merchant quality          |
| Merchant Payout Complaints | > 1/week           | Audit payout SLA and statement clarity                |
| App Crash Rate             | > 0.5% of sessions | Freeze release; investigate Sentry                    |
| API P95 Latency            | > 800ms            | Investigate DB / cache / load                         |
| API Error Rate (5xx)       | > 0.5%             | Page on-call                                          |

---

## Cohort Analysis

We track these on a weekly-cohort basis:

- **Retention curve** — % of each cohort active in week N
- **Revenue per cohort** — cumulative GMV contribution over time
- **Redemption rate by cohort** — trust trajectory
- **Raffle engagement** — % of cohort members with ≥1 raffle entry in month N

A healthy cohort shows **≥ 30% week-4 retention** and **≥ 20% week-12 retention**.

---

## Funnel Tracking

**Primary funnel: Install → Purchase → Redeem → Repeat**

| Step                                  | Target conversion |
| ------------------------------------- | ----------------- |
| Install → Registration                | ≥ 60%             |
| Registration → First deal viewed      | ≥ 85%             |
| First deal viewed → First cart add    | ≥ 25%             |
| Cart → Checkout complete              | ≥ 45%             |
| Purchase → Redemption (within expiry) | ≥ 65%             |
| First redemption → Second purchase    | ≥ 40%             |

---

## Merchant-side Metrics

What the merchant dashboard exposes:

- Coupons sold (today / 7d / 30d)
- Redemption rate (by deal, by branch)
- Time-to-redemption (median, p95)
- Repeat buyers (unique users with >1 purchase at this merchant)
- Views → purchase funnel per deal
- Earnings (pending, available, paid out, lifetime)
- Next payout eligibility date

---

## Raffle Metrics

- Total entries per raffle
- Entries per user (distribution)
- Raffle engagement rate (% of MAU with ≥1 entry)
- Draw-day DAU spike (leading indicator for raffle's retention effect)
- Winner fulfillment time (days from draw to prize delivered)

---

## Ads Metrics (Phase 4)

For every Meta/TikTok campaign:

- Spend
- Impressions, CPM
- Clicks, CTR
- App installs (new acquisition) or deal views (for existing users)
- Conversions: coupons sold for the promoted deal
- CPA (cost per acquired sale)
- ROAS = promoted-deal GMV ÷ ad spend

**Healthy ROAS target:** ≥ 3.0 for growth campaigns; ≥ 5.0 for repeat-user campaigns.

---

## How metrics flow through the system

```
Mobile app / Admin console
        │
        ├─► Backend events (Kafka-less: DB + Redis Streams)
        │         │
        │         ├─► PostHog (product analytics)
        │         ├─► Mixpanel (funnels & cohorts)
        │         └─► GA4 (acquisition attribution)
        │
        └─► OLAP export
                  │
                  └─► Admin dashboard (live SQL views on replicas)
```

- **Real-time** (admin dashboard KPIs): direct reads from read-replica with materialised views.
- **Near-real-time** (PostHog / Mixpanel): event ingestion within 60 seconds.
- **Daily** (cohorts, retention, ROAS): computed by scheduled job, stored in a reporting table.

---

## Metric Definitions — Single Source of Truth

Every metric listed above is defined in code as a named SQL view or computed metric inside the analytics service. See `/packages/analytics/metrics/` in the monorepo. Never invent a new metric name in a dashboard — reuse the canonical one.

---

## Review Cadence

| Review                    | Cadence             | Attendees        |
| ------------------------- | ------------------- | ---------------- |
| Daily Ops Standup         | Every morning 09:00 | Ops + Support    |
| Weekly Metrics Review     | Every Tuesday       | Entire team      |
| Monthly Business Review   | First week of month | Founders + leads |
| Quarterly Strategy Review | End of quarter      | Founders + board |
