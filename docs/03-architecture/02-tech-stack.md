# 02 · Tech Stack

**Status:** 🟢 Approved

Every technology choice with explicit rationale. New additions require an ADR.

---

## Mobile

### Flutter (Dart) — both apps

**Why:** Single codebase for iOS + Android; pixel-perfect UI consistency; 60–120 fps native-compiled performance; first-class Arabic RTL support; production-proven at MENA scale (Talabat, Alibaba); all critical SDKs available officially (Stripe, Tabby, Tamara, Firebase, Google Maps).

**Why not React Native:** Bridge (even with new architecture) adds latency for camera-heavy flows like the QR scanner. UI consistency requires more platform-specific code.

**Why not native (Swift + Kotlin):** Doubles team size and release cycle without proportional gain for our use case.

### Riverpod (state management)

**Why:** Compile-safe DI, easy testability, no `BuildContext` traps, proven at scale.

**Why not BLoC:** More boilerplate; pattern overkill for many simple screens.

**Why not Provider/GetX:** Less type-safe.

### GoRouter (navigation)

**Why:** Declarative, deep-link-first, supports nested routes, official package.

### Freezed (data classes / unions)

**Why:** Codegen for immutable models with copy/equality; sealed unions for state.

### Dio (HTTP client)

**Why:** Interceptor model perfect for auth refresh, retries, idempotency keys; widely supported.

### Hive (local storage)

**Why:** Fast key-value store for offline cart, redemption queue, cached data.

### shared_preferences + flutter_secure_storage

**Why:** OS-keychain-backed storage for tokens (`flutter_secure_storage`); plain prefs for non-sensitive settings.

### Mobile build tooling

- **Melos** — monorepo orchestration for shared packages
- **Fastlane** — release automation for App Store + Play Store
- **Codemagic or Bitrise** — CI for mobile (or GitHub Actions with macOS runners)
- **Shorebird** — OTA updates for emergency hotfixes

---

## Backend

### NestJS (TypeScript) — modular monolith

**Why:** Opinionated structure encourages module isolation; proven, stable; rich ecosystem; first-class DI; matches our admin stack (TS) for shared types; matches your existing ezeeFlight stack.

**Why not bare Express/Fastify:** No structure → divergent patterns across team.

**Why not Go/Rust:** Slower iteration speed; smaller talent pool in Dubai; we don't have CPU-bound problems.

### TypeScript strict mode

**Why:** Catches a category of bugs at compile time; mandatory for refactor safety in a codebase this size.

### PostgreSQL (RDS) — primary store

**Why:** ACID guarantees critical for money; rich feature set (JSONB, partial indexes, materialised views, full-text); proven at Volu's projected scale; horizontal-read scaling via replicas.

**Why not MongoDB:** Money + relational entities (orders, line items, ledgers) need transactions across collections; PostgreSQL handles this natively without sacrificing flexibility (JSONB for variable schemas).

### Redis (ElastiCache) — cache, queues, rate limit

**Why:** Industry-standard for these workloads; sub-ms latency; battle-tested; supports BullMQ.

### BullMQ (Redis-based job queue)

**Why:** First-class TypeScript support; reliable retries with exponential backoff; UI for inspection; production-proven.

**Why not SQS:** Tighter coupling with our service style; richer features (rate limits, repeated jobs).

### Meilisearch — search

**Why:** Self-hostable; sub-200ms; typo tolerance out of the box; Arabic support; simpler ops than Elasticsearch.

**Why not Elasticsearch:** Operational complexity outweighs feature benefit at our scale.

**Why not Postgres FTS only:** Faceting and ranking are weaker; latency degrades with corpus growth.

### Prisma (ORM)

**Why:** Type-safe queries (TS), great migration tooling, excellent DX.

**Why not TypeORM:** Less actively maintained.

**Why not raw SQL:** Type safety + migration discipline matter more than the small perf overhead.

**Caveat:** For hot paths or complex aggregation, drop down to raw SQL via `$queryRaw` — Prisma supports this.

### Pino — structured logging

**Why:** Fastest Node logger; JSON-native; integrates with all our log sinks.

### OpenAPI / Swagger

**Why:** API contract first; codegen for clients (Dart for mobile, TS for admin) ensures type safety end-to-end.

### Zod — runtime validation

**Why:** TS-native schema validation; pairs perfectly with NestJS DTOs; same schema used at boundaries.

---

## Admin (Web)

### Next.js 14 (App Router)

**Why:** Server components reduce JS shipped to client; great for data-heavy admin tables; same React knowledge as the rest of the team; matches your ezeeFlight stack.

### TypeScript + Tailwind CSS + shadcn/ui

**Why:** TS for type safety; Tailwind for fast iteration; shadcn for accessible components without npm dependency hell (copy-paste, you own it).

### TanStack Query (React Query)

**Why:** Server-state caching, optimistic updates, refetch-on-focus.

### TanStack Table

**Why:** Headless, fast, supports virtualization for large datasets.

### Recharts (or Tremor for dashboards)

**Why:** React-native charting; integrates cleanly with Tailwind; good defaults for admin dashboards.

### Auth.js (NextAuth)

**Why:** Wide provider support; handles 2FA flow well; easy to combine with our backend JWT.

---

## Infrastructure

### AWS (me-central-1, Bahrain)

**Why:** UAE PDPL data residency; comprehensive service catalogue; broad team experience available.

**Why not Azure / GCP:** AWS has stronger MENA presence and the most mature ME region.

### ECS Fargate

**Why:** Container orchestration without managing nodes; pay-per-task; integrates with ALB, IAM, autoscaling.

**Why not Kubernetes (EKS):** Operational complexity not worth it at our scale; Fargate covers 95% of needs.

### Cloudflare (CDN + WAF)

**Why:** Best-in-class CDN; integrated DDoS protection; WAF rules for OWASP Top 10; image transformations for free.

### Cloudflare R2 (S3-compatible) — object storage

**Why:** Zero egress fees (vs S3 $$); S3 API compatible (no migration risk); good performance from UAE.

**Why not S3:** Egress costs add up at scale; R2's pricing wins.

### Terraform (infra-as-code)

**Why:** Versioned infra; reproducible environments; standard.

### GitHub Actions (CI/CD)

**Why:** Tight integration with code reviews; free for our scale; OIDC to AWS without long-lived secrets.

### Sentry — error monitoring

**Why:** Cross-platform (Flutter + Node + Next.js); excellent error grouping; release tracking.

### Grafana stack (Loki + Tempo + Mimir + Grafana)

**Why:** Open-source observability that we own; self-hosted on EKS later if needed; today, Grafana Cloud paid plan.

---

## External Services (with rationale)

### Stripe (payments + payouts)

**Why:** Best DX; comprehensive UAE-AED support; Stripe Connect Express handles merchant onboarding KYC + payouts; official Flutter SDK; webhook reliability.

### Tabby + Tamara (BNPL)

**Why:** UAE BNPL market leaders; both have official Flutter SDKs and AED support.

### Apple Pay + Google Pay (via Stripe)

**Why:** Tap-to-pay reduces checkout friction.

### Careem Pay

**Why:** Strong UAE wallet; reaches users who avoid card payments.

### Unifonic (SMS)

**Why:** Best UAE deliverability; OTP optimised; UAE-based support.

### Twilio (SMS fallback)

**Why:** International tourist coverage; mature fallback option.

### WhatsApp Business API (via 360dialog)

**Why:** WhatsApp is the dominant chat channel in UAE; 360dialog is a stable provider with WhatsApp Cloud API.

### Resend (transactional email)

**Why:** Developer-first DX; fast deliverability; React Email templates.

### AWS SES (email fallback)

**Why:** Inexpensive; reliable; good fallback.

### Firebase Cloud Messaging + APNs

**Why:** Standard for push; free at our scale.

### Google Maps Platform

**Why:** Best UAE map data; widely understood; geocoding accurate.

### IDfy or Sumsub (KYC OCR)

**Why:** Trade licence + Emirates ID auto-extraction speeds merchant onboarding; OCR accuracy critical.

### Meta Marketing API + Conversions API

**Why:** Ads integration built into product; Conversions API enables attribution and lookalikes.

### TikTok Marketing API + Events API

**Why:** Same, for TikTok which is rising fast in UAE.

### PostHog (product analytics)

**Why:** Self-hostable for data ownership; full feature set (funnels, cohorts, replay, feature flags); generous free tier.

### Mixpanel (alt analytics)

**Why:** Stronger funnel + retention features for non-technical users; good for marketing team.

### GA4 (acquisition attribution)

**Why:** Necessary for ad attribution; web + mobile.

### Intercom (customer support)

**Why:** In-app chat embeddable in Flutter; merchant-side ticket workflows.

---

## Testing

| Layer               | Tool                                              |
| ------------------- | ------------------------------------------------- |
| Backend unit        | Jest                                              |
| Backend integration | Jest + supertest + Testcontainers (real Postgres) |
| Mobile unit         | flutter_test                                      |
| Mobile widget       | flutter_test                                      |
| Mobile E2E          | Patrol or Maestro                                 |
| Admin unit          | Jest + React Testing Library                      |
| Admin E2E           | Playwright                                        |
| Load                | k6                                                |
| Visual regression   | Chromatic for admin; Percy for mobile (optional)  |
| Mutation testing    | Stryker (optional)                                |

---

## Coding & quality

| Concern             | Tool                                 |
| ------------------- | ------------------------------------ |
| TS lint             | ESLint + typescript-eslint           |
| TS format           | Prettier                             |
| Dart lint           | flutter_lints + custom rules         |
| Dart format         | dart format                          |
| Pre-commit          | Husky + lint-staged                  |
| Commit messages     | Conventional Commits + commitlint    |
| Dependency security | Dependabot + Snyk                    |
| Secrets scanning    | git-secrets + GitHub secret scanning |
| SAST                | CodeQL or Semgrep                    |
| License compliance  | license-checker (Node), pana (Dart)  |

---

## Why we said no to popular alternatives

| Alternative considered             | Verdict                         | Why                                                                                   |
| ---------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------- |
| GraphQL (Apollo / Hasura)          | No (REST + OpenAPI)             | Adds complexity; we don't have over-fetching pain at our scale.                       |
| MongoDB                            | No (Postgres)                   | Money + relations need transactions; Postgres covers our flexibility needs via JSONB. |
| Microservices from day one         | No (modular monolith)           | Network calls + distributed transactions are a tax we shouldn't pay before forced to. |
| Kubernetes                         | No (ECS Fargate)                | Operational complexity not justified.                                                 |
| gRPC internal                      | No (TypeScript modules)         | We're a monolith; gRPC adds nothing.                                                  |
| Custom auth                        | No (built on Passport.js + JWT) | Auth is too security-sensitive to home-grow.                                          |
| Stream processing (Kafka, Kinesis) | No (Redis + Postgres)           | Our event volume doesn't justify the operational cost.                                |

---

## Tech-stack version pinning

All third-party dependencies are pinned to specific versions in `package.json`, `pubspec.yaml`, `Dockerfile`, and Terraform. Renovatebot or Dependabot opens PRs for updates; security patches auto-merge after CI green; major-version bumps require explicit review.

---

## Versions targeted (snapshot)

| Tech        | Version                |
| ----------- | ---------------------- |
| Node.js     | LTS (e.g., 20.x at v1) |
| TypeScript  | 5.x                    |
| NestJS      | 10.x                   |
| Prisma      | 5.x                    |
| PostgreSQL  | 16                     |
| Redis       | 7                      |
| Meilisearch | 1.x                    |
| Flutter     | latest stable          |
| Dart        | sound null safety      |
| Next.js     | 14.x (App Router)      |
| Tailwind    | 3.x                    |
| Terraform   | 1.6+                   |

These versions will be revised but the discipline of pinning is permanent.

---

## See also

- [System Architecture](./01-system-architecture.md)
- [Backend Architecture](./03-backend-architecture.md)
- [Mobile Architecture](./04-mobile-architecture.md)
- [Coding Standards](../09-development/02-coding-standards.md)
