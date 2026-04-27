# 02 · Non-Functional Requirements

**Status:** 🟢 Approved

The qualities the system must have. Each requirement has a stable ID (`NFR-XXX`), a measurable target, and a verification method.

---

## NFR-1xx · Performance

### NFR-101 · API response time

- **Requirement:** P50 ≤ 150ms, P95 ≤ 500ms, P99 ≤ 1000ms for read endpoints under normal load.
- **Write endpoints:** P95 ≤ 800ms (excluding payment-gateway round-trips).
- **Verification:** Continuous load testing in staging; production monitoring via APM (Sentry + custom metrics).

### NFR-102 · Mobile app cold-start time

- **Requirement:** ≤ 2.5 seconds on iPhone 12 / Galaxy S21 with warm cache.
- **Verification:** Automated startup profiling on each release.

### NFR-103 · Mobile app screen transitions

- **Requirement:** 60 FPS sustained on iPhone 12 / Galaxy S21; no frame drops > 16ms during scroll on home feed.
- **Verification:** Flutter DevTools profiling on release builds.

### NFR-104 · QR scan to validation result

- **Requirement:** ≤ 1.5 seconds from QR detection to merchant seeing pass/fail banner (online); ≤ 500ms in offline mode (queued).
- **Verification:** Manual + automated timing tests.

### NFR-105 · Search response

- **Requirement:** Search results returned in ≤ 200ms P95 (Meilisearch).
- **Verification:** Continuous monitoring; benchmark on 1000-deal corpus.

### NFR-106 · Home feed load

- **Requirement:** First meaningful paint of home feed ≤ 1.5s on 4G; ≤ 500ms on WiFi.
- **Verification:** Real User Monitoring (RUM).

### NFR-107 · Checkout completion

- **Requirement:** From "Pay" tap to confirmation screen: P95 ≤ 6 seconds (includes Stripe round-trip + 3DS if needed).
- **Verification:** End-to-end timing in production.

### NFR-108 · Image delivery

- **Requirement:** Hero images served via CDN with WebP/AVIF, P95 fetch ≤ 200ms in UAE.
- **Verification:** Cloudflare R2 + Cloudflare Images metrics.

---

## NFR-2xx · Scalability

### NFR-201 · Concurrent users

- **Requirement:** Support 50,000 concurrent users at launch; scale to 250,000 within 12 months without re-architecture.
- **Verification:** Load tests at 2× target; horizontal scalability of stateless services validated.

### NFR-202 · Order throughput

- **Requirement:** 500 orders/minute peak (e.g., flash deal launch); sustained 100 orders/minute.
- **Verification:** Load test with simulated flash-drop scenario.

### NFR-203 · Database connection scaling

- **Requirement:** Use PgBouncer with transaction pooling; max 200 backend connections per app instance; auto-scale read replicas based on CPU.
- **Verification:** Connection pool metrics; replica lag monitoring.

### NFR-204 · Redis throughput

- **Requirement:** 100,000 ops/second; sub-millisecond latency P99.
- **Verification:** ElastiCache CloudWatch metrics.

### NFR-205 · Background job processing

- **Requirement:** BullMQ workers process queue depth back to zero within 5 minutes under peak load. Auto-scale workers on queue depth.
- **Verification:** Queue-depth alerting; load test.

### NFR-206 · Storage growth

- **Requirement:** Architecture supports 100M coupons, 10M users, 50K merchants without redesign.
- **Verification:** Capacity planning doc; partitioning strategy documented.

---

## NFR-3xx · Availability & Reliability

### NFR-301 · Uptime SLA (user-facing)

- **Requirement:** 99.9% monthly uptime for the user app, merchant app, and admin console (excluding scheduled maintenance windows).
- **Verification:** Synthetic monitoring from multiple geographies; status page.

### NFR-302 · Uptime SLA (payment processing)

- **Requirement:** 99.95% for the payment endpoints. Even during Volu maintenance, redemption must continue (read-only mode).
- **Verification:** Decoupled critical-path architecture; runbook for graceful degradation.

### NFR-303 · Recovery Time Objective (RTO)

- **Requirement:** ≤ 1 hour for full service restoration after a regional incident.
- **Verification:** DR drills quarterly.

### NFR-304 · Recovery Point Objective (RPO)

- **Requirement:** ≤ 5 minutes data loss in worst-case regional failure.
- **Verification:** Continuous WAL streaming to standby; backup tested weekly.

### NFR-305 · Graceful degradation

- **Requirement:** If search is down, fallback to category browse. If recommendations are down, fallback to most-recent. If image CDN is slow, show placeholders. If payment fails, retry once with exponential backoff before surfacing error.
- **Verification:** Chaos engineering tests for each dependency.

### NFR-306 · Idempotency

- **Requirement:** All mutating endpoints accept an idempotency key and return cached responses for retries within 24h.
- **Verification:** API tests cover idempotent retry scenarios.

### NFR-307 · Offline merchant scanner

- **Requirement:** Merchant app must support 4 hours offline operation for redemptions; queue locally with cryptographic signing; sync on reconnect.
- **Verification:** Manual + automated offline test scenarios.

---

## NFR-4xx · Security

### NFR-401 · Encryption in transit

- **Requirement:** TLS 1.3 minimum for all client-server traffic. HSTS enabled. Certificate pinning in mobile apps for production builds.
- **Verification:** Automated TLS scan; certificate pinning unit tests.

### NFR-402 · Encryption at rest

- **Requirement:** All databases, object storage, and backups encrypted with AES-256. KMS-managed keys with 90-day rotation.
- **Verification:** AWS KMS audit reports.

### NFR-403 · PII protection

- **Requirement:** PII fields (phone, email, name, DOB, Emirates ID) encrypted at column level with separate KMS key. KYC documents stored in dedicated S3 bucket with private ACL and signed URL access.
- **Verification:** Schema review; access log audit.

### NFR-404 · Authentication strength

- **Requirement:** Admin: mandatory 2FA (TOTP). Merchant Owner: 2FA strongly recommended, mandatory after first AED 10K payout. User: phone OTP + biometric optional.
- **Verification:** Auth flow tests; admin login audit.

### NFR-405 · Authorisation enforcement

- **Requirement:** Every endpoint checks authorisation server-side; no client-side-only checks. RBAC enforced at controller and service layers (defence in depth).
- **Verification:** Authorisation matrix tested in integration suite.

### NFR-406 · Rate limiting

- **Requirement:** Per-IP and per-user limits on auth endpoints (10/min), search (60/min), payment creation (5/min). Configurable. Returns 429 with `Retry-After`.
- **Verification:** Load tests verify rate-limit enforcement.

### NFR-407 · Audit logging

- **Requirement:** Every privileged action (admin or merchant) is logged: who, what, when, before-state, after-state, IP, user-agent. Logs immutable for 12 months.
- **Verification:** Audit log schema validated; sampled review monthly.

### NFR-408 · Vulnerability scanning

- **Requirement:** SAST on every PR (CodeQL or Semgrep). Dependency scanning weekly (Dependabot + Snyk). DAST monthly. Penetration test annually by third party.
- **Verification:** CI/CD pipeline; pen-test report retention.

### NFR-409 · Secret management

- **Requirement:** No secrets in code or env files committed to git. AWS Secrets Manager for all production secrets. Local dev uses `.env.local` (gitignored).
- **Verification:** git-secrets pre-commit hook; secret-scanning CI step.

### NFR-410 · Session security

- **Requirement:** Session tokens HttpOnly + Secure + SameSite=Strict cookies (web admin). Mobile JWTs stored in OS-level secure storage (iOS Keychain, Android EncryptedSharedPreferences).
- **Verification:** Code review; security audit.

---

## NFR-5xx · Compliance & Privacy

### NFR-501 · UAE PDPL compliance

- **Requirement:** User consent captured for data collection. Privacy notice in EN + AR. Data export within 24h on request. Data deletion within 30 days on request (subject to financial-record retention).
- **Verification:** Privacy review; PDPL self-assessment.

### NFR-502 · Data residency

- **Requirement:** All user PII and transactional data stored in AWS me-central-1 (Bahrain). No cross-border replication of personal data without explicit user consent.
- **Verification:** AWS region audit; data-flow diagram.

### NFR-503 · UAE VAT compliance

- **Requirement:** All applicable transactions issued with TRN, sequential invoice numbers, tax breakdown. FTA Phase-2 e-invoice ready (XML format).
- **Verification:** Sample invoice review by UAE tax advisor.

### NFR-504 · PCI DSS scope minimisation

- **Requirement:** Volu does not store PAN (primary account number). All card data tokenised by Stripe Elements client-side. SAQ-A scope only.
- **Verification:** PCI scope audit annually.

### NFR-505 · KYC retention

- **Requirement:** Merchant KYC documents retained for 7 years post-merchant-offboarding (UAE financial-records retention requirement).
- **Verification:** Document lifecycle policy.

### NFR-506 · Marketing consent

- **Requirement:** Marketing push, email, SMS, WhatsApp require explicit opt-in (not pre-checked). Opt-out one-tap in any message and in-app settings.
- **Verification:** Consent log review.

### NFR-507 · Cookie consent (web admin)

- **Requirement:** Web admin shows cookie banner on first visit; only essential cookies until consent.
- **Verification:** Web compliance audit.

---

## NFR-6xx · Usability & Accessibility

### NFR-601 · WCAG 2.1 AA

- **Requirement:** All consumer-facing screens meet WCAG 2.1 AA. Minimum: contrast ratios, alt text, focus indicators, screen reader labels, captions for any video.
- **Verification:** Automated (axe-core) on every PR; manual screen-reader testing per release.

### NFR-602 · Tap target size

- **Requirement:** Minimum 44×44pt (iOS) / 48×48dp (Android) for any tappable element.
- **Verification:** Design system enforces; visual regression tests.

### NFR-603 · Bilingual parity

- **Requirement:** Every user-facing string available in EN and AR. RTL layout pixel-equivalent to LTR. No truncation in either language.
- **Verification:** i18n linter; per-release Arabic UAT.

### NFR-604 · Internationalisation

- **Requirement:** All dates, times, numbers, currency formatted per locale. Calendar respects user's locale (Hijri shown alongside Gregorian in AR mode where appropriate).
- **Verification:** i18n unit tests.

### NFR-605 · Onboarding time

- **Requirement:** New user from app open to viewing first deal: ≤ 90 seconds (without skipping mobile verification).
- **Verification:** UX session recordings; first-week metrics.

### NFR-606 · Empty states

- **Requirement:** Every list view that can be empty has a designed empty state with helpful copy and a CTA.
- **Verification:** UX checklist per screen.

### NFR-607 · Error states

- **Requirement:** Every error state shows user-friendly copy (no stack traces, no error codes alone), what happened, what to do next.
- **Verification:** Error-state catalogue; copy review.

### NFR-608 · Loading states

- **Requirement:** Any operation > 300ms shows a loading indicator. Skeleton screens preferred over spinners for content-heavy views.
- **Verification:** UX checklist.

---

## NFR-7xx · Maintainability

### NFR-701 · Test coverage

- **Requirement:** Backend unit test coverage ≥ 80% for business logic (services, domain). Integration tests for every API endpoint. E2E tests for critical user flows (signup, purchase, redeem, payout).
- **Verification:** Coverage reports in CI; coverage gate on PRs.

### NFR-702 · Code review

- **Requirement:** Every PR requires at least one approval from a code owner. Auto-merge disabled. Direct push to `main` blocked.
- **Verification:** GitHub branch protection rules.

### NFR-703 · Documentation freshness

- **Requirement:** Public API docs auto-generated from OpenAPI spec on merge. Internal docs reviewed quarterly. Outdated docs marked 🔴 Deprecated.
- **Verification:** Quarterly doc audit.

### NFR-704 · Dependency hygiene

- **Requirement:** No critical CVEs in production dependencies. High-severity CVEs patched within 7 days.
- **Verification:** Snyk + Dependabot monitoring.

### NFR-705 · Linting & formatting

- **Requirement:** All code passes lint and format checks before merge. Format auto-applied via pre-commit hooks.
- **Verification:** CI pipeline.

### NFR-706 · Type safety

- **Requirement:** TypeScript strict mode in backend and admin. Dart sound null safety in mobile. No `any` types in business logic.
- **Verification:** tsconfig + analysis_options.yaml; PR reviews.

### NFR-707 · Logging

- **Requirement:** Structured JSON logs with correlation IDs. Log levels: debug / info / warn / error. PII redacted from logs.
- **Verification:** Log sampling review; PII scanner on log streams.

---

## NFR-8xx · Observability

### NFR-801 · Metrics

- **Requirement:** Every service exposes Prometheus metrics: request rate, error rate, latency, custom business metrics. Dashboards for each service.
- **Verification:** Grafana dashboards; runbook references.

### NFR-802 · Distributed tracing

- **Requirement:** OpenTelemetry tracing across services. P95 trace sampling 10% in production, 100% in staging.
- **Verification:** Trace UI accessible; sampled trace review.

### NFR-803 · Error tracking

- **Requirement:** All runtime errors captured by Sentry across mobile, backend, admin. Error grouping; assignee per error; SLA on error triage.
- **Verification:** Sentry quota; weekly error review.

### NFR-804 · Alerting

- **Requirement:** Alerts paged to on-call for: API error rate > 1%, P95 latency > SLA, payment failure rate > 5%, queue depth backlog, DB replication lag > 30s. Non-pageable alerts to a #ops Slack channel.
- **Verification:** Alert runbook; paging response drills.

### NFR-805 · Health checks

- **Requirement:** Each service exposes `/health` (liveness) and `/ready` (readiness) endpoints. Used by load balancer and orchestrator.
- **Verification:** Endpoint tests.

---

## NFR-9xx · Cost

### NFR-901 · Per-order cost

- **Requirement:** Cloud + payment + SMS infra cost per order ≤ AED 1.50 at scale (target month 12).
- **Verification:** Monthly cost-per-unit tracking.

### NFR-902 · Storage cost

- **Requirement:** Use lifecycle policies to move old logs to cheap storage; cold-tier for backups > 90 days.
- **Verification:** AWS Cost Explorer reviews.

### NFR-903 · Cost alerts

- **Requirement:** AWS Budget alerts at 80% and 100% of monthly forecast.
- **Verification:** Budget configuration audit.

---

## NFR-10xx · Operability

### NFR-1001 · Zero-downtime deploys

- **Requirement:** Backend deploys with rolling restart, no user-visible downtime. DB migrations are backward-compatible (expand-then-contract pattern).
- **Verification:** Deploy timing in production; canary deployments.

### NFR-1002 · Feature flag toggles

- **Requirement:** Risky features behind flags. Flag changes propagate within 60s without redeploy.
- **Verification:** Flag service uptime; rollout playbook.

### NFR-1003 · Rollback capability

- **Requirement:** Any deploy can be rolled back within 5 minutes via single command.
- **Verification:** Rollback drills.

### NFR-1004 · Mobile force-update

- **Requirement:** Admin can set minimum required app version per platform; old clients shown a force-update screen.
- **Verification:** Force-update flow tested.

### NFR-1005 · OTA hotfix capability (mobile)

- **Requirement:** Critical bug fixes deployable to mobile without app-store review via Shorebird (Flutter OTA).
- **Verification:** OTA capability documented; only used for true emergencies.

---

## Acceptance

This document is the contract between Product, Engineering, and Operations. Every NFR has an owner who signs off on its verification.

| Area                        | Owner              |
| --------------------------- | ------------------ |
| Performance                 | Eng Lead (Backend) |
| Security                    | Security Lead      |
| Compliance                  | Legal + CTO        |
| UX / Accessibility          | Design Lead        |
| Observability + Operability | SRE Lead           |
