# 01 · System Architecture

**Status:** 🟢 Approved

The high-level architecture of Volu. This is the map every engineer should have in their head.

---

## Architectural style

**Modular monolith on day one, designed for selective extraction.**

We start with a single deployable backend service organised into strict modules with hard internal boundaries (no cross-module DB reads, no shared state, all communication via internal APIs and events). When a module's load profile diverges from the rest (the redemption hot path is the prime candidate), it can be extracted to its own service without rewriting business logic.

**Why not microservices from day one:** premature service boundaries cause more pain than they solve at our scale. Network calls between services, distributed transactions, debugging across logs — all of this is real engineering tax that has zero customer value at < 100K users. Modular monolith gives us 90% of the architectural benefit at 10% of the operational cost.

---

## High-level diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                      │
│                                                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│   │  User App    │  │ Merchant App │  │  Admin Web   │              │
│   │  (Flutter)   │  │  (Flutter)   │  │  (Next.js)   │              │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
└──────────┼─────────────────┼─────────────────┼──────────────────────┘
           │                 │                 │
           │              HTTPS / TLS 1.3 (with cert pinning on mobile)
           │                 │                 │
           ▼                 ▼                 ▼
   ┌────────────────────────────────────────────────────┐
   │              CLOUDFLARE                            │
   │  (CDN, WAF, DDoS, Bot Mgmt, Image transforms)      │
   └────────────────────┬───────────────────────────────┘
                        │
                        ▼
   ┌────────────────────────────────────────────────────┐
   │          AWS Application Load Balancer             │
   │              (multi-AZ, health checks)             │
   └────────────────────┬───────────────────────────────┘
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
   ┌──────────────────┐    ┌──────────────────┐
   │  API Gateway     │    │  Admin Backend   │
   │  (NestJS, ECS)   │    │  (NestJS, ECS)   │
   │  ── auth, rate   │    │  ── admin-only   │
   │     limit, fwd   │    │     endpoints    │
   └────────┬─────────┘    └────────┬─────────┘
            │                       │
            └───────────┬───────────┘
                        ▼
   ┌──────────────────────────────────────────────────────┐
   │             VOLU CORE (NestJS Modular Monolith)      │
   │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
   │ │ Identity │ │  Catalog │ │ Commerce │ │Redemption│  │
   │ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
   │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
   │ │ Payments │ │  Wallet  │ │ Payouts  │ │ Raffles  │  │
   │ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
   │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
   │ │  Notify  │ │  Ads API │ │  Reviews │ │  Audit   │  │
   │ └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
   └────────┬─────────────────────────────────────────────┘
            │
   ┌────────┼────────┬────────────┬────────────┬─────────────┐
   ▼        ▼        ▼            ▼            ▼             ▼
┌──────┐ ┌──────┐ ┌────────┐ ┌──────────┐ ┌─────────┐ ┌───────────┐
│Postgres│ │Redis │ │Meili-  │ │Cloudflare│ │ BullMQ  │ │S3 (R2)    │
│(RDS,   │ │(EC,  │ │search  │ │Stream    │ │Workers  │ │buckets    │
│Multi-AZ│ │ Multi│ │        │ │(WS, real-│ │ (ECS)   │ │ (KYC,     │
│ + read │ │  AZ) │ │        │ │ time)    │ │         │ │ images,   │
│replicas│ │      │ │        │ │          │ │         │ │ invoices) │
└────────┘ └──────┘ └────────┘ └──────────┘ └─────────┘ └───────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │   EXTERNAL SERVICES    │
                │ Stripe · Tabby · Tamara│
                │ Unifonic · Twilio · WA │
                │ FCM · APNs · Resend    │
                │ Meta API · TikTok API  │
                │ Google Maps · IDfy     │
                │ Sentry · PostHog · Mix │
                └────────────────────────┘
```

---

## Components

### Clients

**User App (Flutter, iOS + Android).**
The primary surface for end-users. Discovery, purchase, wallet, redemption, raffle. Optimised for performance on mid-range devices. Offline-tolerant for browsing cached content.

**Merchant App (Flutter, iOS + Android).**
Cashier-focused: scanner is the most important screen. Also: deal management (Owner role), payouts, real-time dashboard. Critical: works offline for redemption.

**Admin Web (Next.js + React + Tailwind + shadcn/ui).**
Internal-only browser-based console. Server-rendered for SEO-irrelevant routes; SPA for data-heavy dashboards.

### Edge layer

**Cloudflare.**
CDN for static assets and image delivery (with on-the-fly resize/format conversion via Cloudflare Images). WAF rules for OWASP Top 10. Bot management. DDoS protection.

### Application layer

**AWS ALB.** Multi-AZ load balancer in front of all backend services. Health checks at `/ready`.

**API Gateway (NestJS).** Public-facing API for user app and merchant app. Handles JWT validation, rate limiting, request normalisation, then forwards to the core service. Stateless. Horizontally scaled.

**Admin Backend (NestJS).** Separate ECS service for admin endpoints. Same code module imports as core, but with stricter auth and audit logging on every action. Isolated so a flood of consumer traffic never starves admin operations.

**Volu Core (NestJS).** The modular monolith. Every domain capability lives in a NestJS module:

| Module        | Owns                                                             |
| ------------- | ---------------------------------------------------------------- |
| Identity      | Users, merchants, admin users, sessions, RBAC, KYC documents     |
| Catalog       | Categories, deals, deal versions, branches, store profiles       |
| Commerce      | Cart, orders, checkout orchestration, promo codes                |
| Payments      | Stripe / Tabby / Tamara / wallet integrations                    |
| Coupons       | Coupon issuance, QR/PIN generation, lifecycle                    |
| Redemption    | QR validation, PIN check, redemption recording, offline sync     |
| Wallet        | Merchant ledger entries (pending / available / paid_out)         |
| Payouts       | Payout requests, approvals, Stripe Connect transfers, statements |
| Raffles       | Raffle configuration, rule engine, entries, draws, winners       |
| Loyalty       | Points, referrals, tier management                               |
| Reviews       | Ratings, moderation, merchant responses                          |
| Notifications | Push, email, SMS, WhatsApp, in-app banners                       |
| Ads           | Meta + TikTok API integration, campaigns, reporting              |
| Invoicing     | Tax invoice + commission invoice generation, e-invoice           |
| Audit         | Privileged action audit log                                      |
| Reporting     | Analytics, materialised views, export jobs                       |

Each module exposes:

- A **public API** (controllers): HTTP endpoints
- An **internal API** (services): for other modules to call
- **Domain events**: emitted for side effects
- **Database schema**: owned by the module; no other module can read/write its tables

### Data layer

**PostgreSQL (RDS, multi-AZ).** Primary database. One logical database; modules namespace their tables (`identity__users`, `catalog__deals`, etc.). Read replicas for analytics queries and admin dashboard reads. Automatic backups; point-in-time recovery.

**Redis (ElastiCache, multi-AZ).**

- Cache layer for hot reads (deal details, merchant profiles).
- Session storage for refresh-token blacklist.
- Rate limiting counters.
- BullMQ job queue.
- Pub/Sub for low-latency cross-instance signals (e.g., feature flag invalidation).

**Meilisearch.** Full-text search engine for deals. Indexed in EN and AR. Sub-200ms responses. Triggered on deal create/update/delete via domain events.

**Cloudflare Stream / WebSocket gateway.** Realtime: live deal countdowns, redemption confirmation push to merchant dashboard.

**S3 (Cloudflare R2 — S3-compatible).**

- KYC documents (private, encrypted, signed-URL access)
- Deal hero + gallery images (public, served via Cloudflare CDN)
- Generated PDFs (invoices, statements, wallet passes)
- Backups

### Background processing

**BullMQ Workers (ECS Fargate).** Process scheduled and event-driven jobs:

- Coupon expiry processing (every 5 min)
- Coupon expiry reminders (hourly)
- Daily Stripe reconciliation (02:00 UAE)
- Document expiry checks (daily)
- Payout disbursement (triggered)
- Raffle draws (scheduled)
- Notification delivery (event-driven)
- Image processing (event-driven)
- Search index updates (event-driven)

Auto-scaled by queue depth.

### External services

| Capability              | Provider                                                                |
| ----------------------- | ----------------------------------------------------------------------- |
| Card payments + payouts | Stripe (UAE entity) + Stripe Connect Express                            |
| BNPL                    | Tabby + Tamara                                                          |
| Wallets                 | Apple Pay + Google Pay (via Stripe), Careem Pay                         |
| SMS                     | Unifonic (UAE primary), Twilio (international fallback)                 |
| WhatsApp                | WhatsApp Business API via Meta or 360dialog                             |
| Email                   | Resend (primary), AWS SES (fallback)                                    |
| Push                    | Firebase Cloud Messaging + APNs                                         |
| Maps                    | Google Maps Platform                                                    |
| KYC OCR                 | IDfy or Sumsub                                                          |
| Ads                     | Meta Marketing API + Conversions API; TikTok Marketing API + Events API |
| Errors                  | Sentry                                                                  |
| Product analytics       | PostHog (self-hosted) + Mixpanel + GA4                                  |
| Customer support        | Intercom                                                                |

---

## Request flow examples

### Reading a deal

1. User app → Cloudflare → ALB → API Gateway.
2. API Gateway validates JWT (cached pubkey).
3. Forwards to Volu Core `/api/v1/deals/:id`.
4. Catalog module checks Redis cache.
5. If cache miss: reads from PostgreSQL replica; populates cache (TTL 5 min).
6. Returns to user app.

**Target latency:** P95 ≤ 200ms end-to-end (UAE → infra → UAE).

### Purchasing a coupon

1. User taps "Pay" → app initiates Stripe PaymentIntent via Volu Core.
2. Commerce module creates an order in `pending` state, reserves inventory atomically (Postgres `UPDATE ... RETURNING`), generates idempotency key.
3. Returns Stripe client secret to app.
4. App completes payment with Stripe (3DS if needed).
5. Stripe sends webhook → Volu Core webhook endpoint.
6. Webhook handler verifies signature, looks up order by Stripe PI id, transitions state to `paid`.
7. Emits `OrderPaid` domain event.
8. Subscribers fire:
   - Coupons module → generates coupon records, QR tokens, PINs
   - Wallet module → debits merchant `pending` ledger
   - Raffles module → evaluates rules, creates entries
   - Notifications module → push + email + in-app
   - Ads module → fires Conversions API event
9. User app polls or receives WebSocket push; shows confirmation.

**Target end-to-end:** P95 ≤ 6 seconds from "Pay" tap to confirmation.

### Redeeming a coupon

1. Cashier scans QR → merchant app sends QR token to Volu Core `/api/v1/redemptions/validate`.
2. Redemption module validates JWT-like token signature (HMAC).
3. Reads coupon from Postgres (cached if hot); checks status, expiry, branch eligibility, valid hours.
4. Returns "PIN required."
5. Cashier enters PIN → app sends to `/api/v1/redemptions/confirm` with idempotency key.
6. Redemption module validates PIN against hashed PIN; on success, atomic update: status → redeemed (or decrement uses_remaining), insert redemption record.
7. Emits `CouponRedeemed` event.
8. Subscribers fire:
   - Wallet module → schedules `pending` → `available` after hold period
   - Notifications module → user push, daily summary entry
   - Reporting module → metrics, analytics
9. Returns success to merchant app.

**Target latency:** P95 ≤ 1.5 seconds online; ≤ 500ms offline (queued).

---

## Cross-cutting concerns

### Authentication

Single JWT issuance service (Identity module). Access tokens 15-min TTL, signed with rotating keypair. Public key cached at API Gateway for stateless verification. Refresh tokens are opaque, rotated on every use, bound to device fingerprint.

### Authorisation

RBAC enforced in every controller via NestJS guards. Defence in depth: also enforced in service layer for sensitive operations.

### Rate limiting

Edge: Cloudflare WAF rules. App: per-user and per-IP via Redis counter sliding window.

### Logging

Structured JSON logs (Pino). Correlation ID propagated from edge through every service call. PII redacted at log-emit time. Shipped to AWS CloudWatch + Grafana Loki.

### Tracing

OpenTelemetry instrumentation. Traces shipped to Grafana Tempo. 10% sample in production, 100% in staging.

### Metrics

Prometheus-format metrics from every service. Scraped by Grafana Cloud or self-hosted Mimir. Dashboards per module.

### Error tracking

Sentry SDK in mobile, backend, admin. Errors grouped, assigned, and SLA'd.

### Feature flags

Service: GrowthBook (open-source) or LaunchDarkly. Flags evaluated client-side or server-side. Cached in Redis with 60-second propagation.

---

## Deployment topology

**Production:** AWS me-central-1 (Bahrain), 3 availability zones.

**Compute:** ECS Fargate for stateless services (API Gateway, Volu Core, Admin Backend, Workers). Auto-scaling group based on CPU + custom metrics (queue depth for workers, request rate for APIs).

**Data:**

- PostgreSQL: RDS Multi-AZ primary in AZ-1, standby in AZ-2, two read replicas across AZs.
- Redis: ElastiCache cluster mode disabled, Multi-AZ with automatic failover.
- Meilisearch: ECS task with EBS-backed persistence; nightly snapshots to S3.
- S3 (R2): single region (UAE) with cross-region replication (KSA region) for backups.

**Networking:** VPC with public, private app, and private data subnets. NAT Gateways for outbound. PrivateLink for AWS service access where available.

**CDN:** Cloudflare in front of public assets and API.

**Staging environment:** AWS me-central-1, scaled-down mirror.

**Local dev environment:** Docker Compose with all dependencies (Postgres, Redis, Meilisearch, MailHog).

---

## Selective extraction roadmap

When (not if) we extract a module to its own service:

| Module        | Trigger to extract                            | Why                                                                          |
| ------------- | --------------------------------------------- | ---------------------------------------------------------------------------- |
| Redemption    | When merchant app sees > 1000 RPS during peak | Hot path; isolation prevents user-side incidents from impacting cashier flow |
| Notifications | When push/email volume > 1M/day               | Decouples notification delivery latency from API response time               |
| Ads           | Right before Phase 4 launch                   | Independent deployment cadence; different team ownership                     |
| Reporting     | When read load impacts replicas > 50%         | Read-heavy; can sit on dedicated read-only fleet                             |

Extractions happen via the strangler pattern: extract one endpoint at a time behind the same API surface; keep events compatible.

---

## What's explicitly NOT in v1

- Multi-region active-active. (We do have multi-AZ within UAE.)
- Microservices (we have a modular monolith).
- gRPC. (Internal communication is direct method calls within the monolith; HTTP/REST for external.)
- Kafka or other event-streaming infrastructure. (Postgres + Redis Streams is enough at our scale.)
- Service mesh.
- Custom AI / ML models. (Recommendations in v1 are hand-curated + popularity-based.)

We will revisit each of these when there's a measurable need, not before.

---

## See also

- [Tech Stack](./02-tech-stack.md)
- [Backend Architecture](./03-backend-architecture.md)
- [Mobile Architecture](./04-mobile-architecture.md)
- [Admin Architecture](./05-admin-architecture.md)
- [Event-Driven Design](./06-event-driven-design.md)
- [Cloud Topology](../07-infrastructure/01-cloud-topology.md)
