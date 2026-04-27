# 02 · Security Controls

**Status:** 🟢 Approved

Every security control we implement, mapped to threats and to NFRs.

---

## Control catalogue

### Identity & Access

#### SC-IA-01 · Strong password storage

- **What:** Argon2id with sensible work factors (memory 64MB, iterations 3, parallelism 4).
- **Where:** All password storage (users, admins).
- **Mitigates:** T-S03, T-I02.
- **Verify:** Code review of password-set/change paths.

#### SC-IA-02 · Mandatory phone verification

- No purchase, no payout, no privileged action without verified phone.
- **Mitigates:** T-S04.

#### SC-IA-03 · Mandatory 2FA for admins

- TOTP only (no SMS-based 2FA for admins — known SIM-swap risk).
- TOTP secret stored encrypted with KMS.
- **Mitigates:** T-S03, T-E01.

#### SC-IA-04 · Refresh-token rotation with theft detection

- Single-use refresh tokens; using a previously rotated token invalidates entire user session and pushes alert.
- **Mitigates:** T-S06.

#### SC-IA-05 · Device binding

- Refresh tokens bound to device fingerprint; mismatched device = re-auth.
- **Mitigates:** T-S01.

#### SC-IA-06 · Session management UI

- User can list and revoke active sessions; admin can force logout.
- **Mitigates:** T-S01.

#### SC-IA-07 · Brute-force lockout

- 5 failed admin logins → 15-min lockout per email.
- 3 failed PIN attempts on a coupon → 15-min lock.
- 10 failed OTPs per phone per 24h → block until manual review.
- **Mitigates:** T-S03, T-S05.

#### SC-IA-08 · RBAC enforced server-side

- Every endpoint declares required role/scope; defence in depth at controller AND service.
- **Mitigates:** T-E01, T-E02, T-E03.

#### SC-IA-09 · Resource ownership checks

- User can only access their own orders/coupons; merchant only their own deals/branches; cross-tenant access rejected with 403.
- **Mitigates:** T-I04, T-I08.

#### SC-IA-10 · IP allowlist for Super Admin

- Optional but enabled for Super Admin role; only configured corporate IPs allowed.
- **Mitigates:** T-S03, T-E03.

---

### Network & Edge

#### SC-NE-01 · TLS 1.3 minimum

- All client-server traffic. HSTS preload. Modern cipher suites only.
- **Mitigates:** T-T05.

#### SC-NE-02 · Certificate pinning (mobile)

- Production builds pin api.volu.ae cert (or its issuer's intermediate).
- Failsafe rotation plan documented.
- **Mitigates:** T-T05.

#### SC-NE-03 · Cloudflare WAF

- OWASP Top 10 ruleset enabled. Custom rules for our endpoints. Bot management on auth.
- **Mitigates:** T-D01, T-D02, T-T06.

#### SC-NE-04 · DDoS protection

- Cloudflare edge + AWS Shield Standard. Rate limits at multiple layers.
- **Mitigates:** T-D01.

#### SC-NE-05 · Per-endpoint rate limiting

- Redis sliding window per (endpoint, key). See [API Design Principles](../05-api/01-api-design-principles.md#rate-limiting).
- **Mitigates:** T-D04, T-D02.

#### SC-NE-06 · Body size limits

- 10MB default; 1MB for non-upload endpoints; 25MB for KYC document uploads.
- **Mitigates:** T-D06.

#### SC-NE-07 · Security headers

- HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, CSP for admin web.
- **Mitigates:** Various web vectors.

---

### Application

#### SC-AP-01 · Input validation everywhere

- Every endpoint declares Zod-validated DTOs. Reject before reaching business logic.
- **Mitigates:** T-T06, T-T07.

#### SC-AP-02 · Parameterised queries

- Prisma; no raw string-interpolated SQL with user input.
- **Mitigates:** T-T06.

#### SC-AP-03 · Output encoding

- Admin web uses React (auto-escaping). API responses are JSON; no template-rendered HTML.
- **Mitigates:** XSS.

#### SC-AP-04 · CSRF protection (admin web)

- Same-site=Strict cookies + double-submit token pattern for state-changing requests.
- **Mitigates:** CSRF.

#### SC-AP-05 · Idempotency keys mandatory on mutations

- Prevents accidental double-charges, double-payouts.
- **Mitigates:** T-T01 (partial), correctness threats.

#### SC-AP-06 · Webhook signature verification

- Every inbound webhook signature checked before processing.
- **Mitigates:** T-S02, T-T03.

#### SC-AP-07 · QR token signing (HMAC)

- Coupon QR tokens are HMAC-signed with server-only key; tampering rejected.
- **Mitigates:** T-T02.

#### SC-AP-08 · PIN as second factor

- Coupon redemption requires both QR and PIN; QR alone is insufficient.
- **Mitigates:** F-01, F-02.

#### SC-AP-09 · Pessimistic locking on critical writes

- Coupon redemption uses `SELECT FOR UPDATE` to prevent race conditions / double-spend.
- **Mitigates:** F-02, F-04, F-05.

#### SC-AP-10 · Optimistic concurrency on inventory

- Inventory decrement uses `UPDATE ... WHERE inventory_left > 0` to prevent oversell.
- **Mitigates:** Race conditions, T-D03.

---

### Data Protection

#### SC-DP-01 · Encryption at rest (storage)

- RDS, ElastiCache, S3 — all AES-256 with KMS-managed keys.
- KMS keys rotate every 90 days (envelope encryption).
- **Mitigates:** T-I01.

#### SC-DP-02 · Encryption at rest (column-level for sensitive PII)

- Phone, email, name, DOB, Emirates ID number — encrypted with separate KMS key before storage.
- **Mitigates:** T-I02.

#### SC-DP-03 · KYC documents in dedicated bucket

- Private S3 bucket with bucket-policy restricting access; no public ACLs.
- Only signed URLs (TTL 5 minutes) for retrieval.
- Bucket access logged.
- **Mitigates:** T-I01.

#### SC-DP-04 · No PAN storage

- Volu never stores card primary account numbers. All card data tokenised by Stripe Elements client-side.
- PCI scope: SAQ-A.
- **Mitigates:** Catastrophic financial breach.

#### SC-DP-05 · Secret management

- AWS Secrets Manager for all production secrets. No `.env.production` files in git.
- IAM-based access; short-lived credentials only.
- **Mitigates:** T-I07.

#### SC-DP-06 · Log redaction

- Pino redaction filter scrubs known PII fields from logs (phone, email, name, IP partial-mask, card-related fields, JWT tokens, OTP codes).
- **Mitigates:** T-I03.

#### SC-DP-07 · Encrypted backups

- RDS automated backups encrypted; cross-region snapshots encrypted.
- **Mitigates:** T-I01.

#### SC-DP-08 · Data export & deletion

- User can request data export (delivered as JSON via signed URL with 72h TTL).
- User can request account deletion: soft-delete immediately; hard-delete after 30 days for non-financial data; financial records retained 7 years per UAE tax law.
- **Mitigates:** PDPL compliance.

#### SC-DP-09 · Privacy-by-default in client storage

- Mobile: refresh token in OS keychain; access token in memory only.
- No PII in `shared_preferences` or unencrypted Hive boxes.
- **Mitigates:** Client-side leak.

---

### Auditability

#### SC-AU-01 · Privileged-action audit log

- Every admin or merchant-owner action that changes state writes an audit entry: actor, action, target_type, target_id, before, after, IP, UA, timestamp.
- Append-only; no updates allowed.
- Retained 12 months hot, archived to cold storage for 7 years total.
- **Mitigates:** T-R01, T-R02, T-R03.

#### SC-AU-02 · Authentication event logging

- Login successes/failures, refresh, logout, theft detection, role changes, 2FA enrol/disable.
- **Mitigates:** T-R01, T-S03.

#### SC-AU-03 · Webhook delivery log

- Every inbound + outbound webhook delivery logged with status, latency, response.
- **Mitigates:** T-R03, debugging.

#### SC-AU-04 · Stripe reconciliation log

- Daily Stripe-vs-ledger diff stored with discrepancies flagged.
- **Mitigates:** T-T04, T-R03.

---

### Code & Build

#### SC-CB-01 · SAST in CI

- CodeQL or Semgrep runs on every PR; high-severity findings block merge.
- **Mitigates:** T-T06, T-T07, T-E04.

#### SC-CB-02 · Dependency scanning

- Snyk + Dependabot; weekly scan; auto-PRs for safe updates; high-severity SLA 7 days.
- **Mitigates:** T-E04.

#### SC-CB-03 · Secret scanning

- git-secrets pre-commit hook; GitHub secret scanning; CI step.
- **Mitigates:** T-I07.

#### SC-CB-04 · Code review required

- ≥1 code-owner approval; auto-merge disabled; direct push to `main` blocked.
- **Mitigates:** Human error vector.

#### SC-CB-05 · Container image scanning

- Trivy scan on every image build; high-severity blocks deployment.
- **Mitigates:** T-E04, T-E05.

#### SC-CB-06 · Reproducible builds

- Lockfiles committed (`package-lock.json`, `pubspec.lock`). Pinned versions.
- **Mitigates:** Supply-chain.

#### SC-CB-07 · Signed releases

- Mobile: Play App Signing (Google manages key) + App Store Connect.
- Container images signed with Cosign.
- **Mitigates:** Supply-chain.

---

### Operations

#### SC-OP-01 · Least-privilege IAM

- Service roles scoped to exactly what they need; no wildcard policies.
- **Mitigates:** T-E06.

#### SC-OP-02 · No prod DB write from admin UI

- Admin UI calls APIs; APIs enforce RBAC + audit. No direct DB tools allowed against production.
- **Mitigates:** T-T04, T-E06.

#### SC-OP-03 · JIT prod database access

- Engineers needing prod DB access request via approval workflow; granted for limited time; all sessions logged.
- **Mitigates:** T-T04, T-E06.

#### SC-OP-04 · Audit IAM changes

- Any IAM policy change in AWS triggers Slack alert + monthly review.
- **Mitigates:** T-E06.

#### SC-OP-05 · Annual penetration test

- Independent third party performs full pen test annually + after major changes.
- **Mitigates:** Unknown unknowns.

#### SC-OP-06 · Bug bounty program (post-launch)

- Vetted researchers via HackerOne or self-managed; tiered rewards.
- **Mitigates:** Unknown unknowns.

#### SC-OP-07 · Security training

- Quarterly training for all engineers; annual phishing simulation.
- **Mitigates:** T-S03.

#### SC-OP-08 · Incident response plan

- Documented playbook ([04-incident-response.md](./04-incident-response.md)); on-call rotation; post-mortem culture.

---

### Monitoring & Detection

#### SC-MD-01 · Failed-login alerting

- Alert if failed login rate spikes or hits per-account threshold (potential brute force).

#### SC-MD-02 · Privilege escalation alerting

- Alert on any role change, especially Super Admin assignments.

#### SC-MD-03 · Unusual data access

- Alert on bulk reads of users / KYC documents (potential exfiltration).

#### SC-MD-04 · Payment anomaly detection

- Alert on refund-rate spikes per merchant; chargeback patterns; payment-method-mix shifts.

#### SC-MD-05 · Coupon fraud detection

- Alert on >30 redemptions in 5 minutes from one device; same coupon scan attempt from >2 devices in 10 minutes; cashier pattern deviates >3σ from merchant baseline.

#### SC-MD-06 · WAF alert routing

- Cloudflare WAF blocks → Slack channel for review.

#### SC-MD-07 · KMS access alerting

- Any KMS Decrypt outside expected app pattern → alert.

---

### Mobile-specific

#### SC-MO-01 · No sensitive logs in production

- Sentry breadcrumbs scrub PII before sending; debug logging disabled in release builds.

#### SC-MO-02 · Jailbreak / root detection (advisory)

- App detects jailbreak/root and warns user; does NOT block (false positives high) — but disables coupon-PIN auto-fill on rooted devices.

#### SC-MO-03 · Code obfuscation

- Dart obfuscation enabled in release builds. R8 for Android. Symbol files retained for crash debugging.

#### SC-MO-04 · App Transport Security (iOS) + cleartext disabled (Android)

- TLS-only enforced at OS level.

#### SC-MO-05 · Biometric for sensitive actions

- Coupon PIN reveal, payout request, password reset — biometric prompt.

---

## Mapping: NFR ↔ Control

| NFR                          | Controls satisfying it                 |
| ---------------------------- | -------------------------------------- |
| NFR-401 (TLS in transit)     | SC-NE-01, SC-NE-02, SC-MO-04           |
| NFR-402 (Encryption at rest) | SC-DP-01, SC-DP-02, SC-DP-07           |
| NFR-403 (PII protection)     | SC-DP-02, SC-DP-03, SC-DP-06           |
| NFR-404 (Auth strength)      | SC-IA-01, SC-IA-02, SC-IA-03           |
| NFR-405 (Authorisation)      | SC-IA-08, SC-IA-09                     |
| NFR-406 (Rate limiting)      | SC-NE-05                               |
| NFR-407 (Audit logging)      | SC-AU-01, SC-AU-02                     |
| NFR-408 (Vuln scanning)      | SC-CB-01, SC-CB-02, SC-CB-05, SC-OP-05 |
| NFR-409 (Secrets)            | SC-DP-05, SC-CB-03                     |
| NFR-410 (Session security)   | SC-IA-04, SC-IA-05, SC-IA-06, SC-MO-01 |
| NFR-501 (PDPL)               | SC-DP-08                               |
| NFR-504 (PCI scope)          | SC-DP-04                               |

---

## See also

- [Threat Model](./01-threat-model.md)
- [PCI / PDPL Compliance](./03-pci-pdpl-compliance.md)
- [Incident Response](./04-incident-response.md)
- [Non-Functional Requirements](../02-requirements/02-non-functional-requirements.md)
