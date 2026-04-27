# 01 · Threat Model

**Status:** 🟢 Approved

The Volu threat model, structured by **STRIDE**: Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege.

This model is reviewed quarterly and after every major architectural change. Each threat has an ID (`T-XXX`), a likelihood × impact rating, and a mitigation reference.

---

## Asset inventory

What an attacker would want from Volu:

| Asset                                      | Value to attacker             | Value to Volu                        |
| ------------------------------------------ | ----------------------------- | ------------------------------------ |
| Card data (PANs)                           | High — direct fraud           | Catastrophic if leaked               |
| User PII (phone, email, name, DOB)         | Medium — identity theft, spam | High — PDPL fines, reputation        |
| KYC documents (trade licence, Emirates ID) | High — identity-based fraud   | Catastrophic                         |
| Coupons (active, valuable)                 | Medium — resale               | High — revenue loss + merchant trust |
| Merchant balances                          | High — direct theft           | Catastrophic                         |
| Admin access                               | Critical — full control       | Catastrophic                         |
| Source code                                | Medium — find more vulns      | High                                 |
| Business intelligence (deal performance)   | Medium — competitive          | Medium                               |

---

## STRIDE catalogue

### S — Spoofing (impersonation)

| ID    | Threat                                                                 | Likelihood × Impact | Mitigations                                                                                                                                      |
| ----- | ---------------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| T-S01 | Attacker calls API with stolen JWT                                     | Med × High          | Short-lived access tokens (15m); refresh-token rotation with theft detection; device binding; ability to revoke all sessions                     |
| T-S02 | Attacker fakes Stripe webhook to mark unpaid order as paid             | Low × Critical      | Stripe signature verification; reject any unsigned webhook                                                                                       |
| T-S03 | Phishing — fake Volu admin login page                                  | Med × Critical      | Mandatory 2FA on admin; security-awareness training; SSO option for future                                                                       |
| T-S04 | SIM swap on a user's phone, OTP intercepted, account takeover          | Med × High          | Bind to device fingerprint at login; sensitive actions require re-verification; alert on new-device login; consider TOTP for high-value accounts |
| T-S05 | Cashier impersonation: stolen merchant device used to fake redemptions | Low × Med           | Device binding; per-device fingerprint; merchant can revoke staff; anomaly detection on redemption patterns                                      |
| T-S06 | Refresh token theft from compromised device                            | Med × High          | Single-use rotation; if stolen RT used after legitimate use, ALL sessions revoked + user notified                                                |

### T — Tampering (data modification)

| ID    | Threat                                               | Likelihood × Impact | Mitigations                                                                                                     |
| ----- | ---------------------------------------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| T-T01 | Mobile app modified to bypass payment                | Low × High          | Server-side enforcement (no client-trusted payment state); checkout validated against backend; signed responses |
| T-T02 | Attacker modifies QR token to mark a coupon redeemed | Low × High          | QR token is HMAC-signed; signature verification mandatory; PIN as second factor                                 |
| T-T03 | Attacker tampers with Stripe webhook payload         | Low × Critical      | Signature verification                                                                                          |
| T-T04 | Insider tampers with merchant balance                | Low × Critical      | RBAC + audit log immutable; sensitive operations require approval workflows; no direct DB writes from admin UI  |
| T-T05 | MITM on mobile app intercepts/modifies API calls     | Low × High          | TLS 1.3 + certificate pinning in production builds                                                              |
| T-T06 | SQL injection                                        | Low × Critical      | Parameterised queries (Prisma); input validation; SAST scanning                                                 |
| T-T07 | NoSQL/JSONB injection in JSONB filters               | Low × High          | Validated DTO inputs; never pass user input directly into raw SQL                                               |

### R — Repudiation (deniability)

| ID    | Threat                                | Likelihood × Impact | Mitigations                                                                         |
| ----- | ------------------------------------- | ------------------- | ----------------------------------------------------------------------------------- |
| T-R01 | Admin denies they approved a payout   | Med × High          | Audit log: actor, action, target, before, after, IP, UA, timestamp; immutable       |
| T-R02 | Cashier denies they redeemed a coupon | Low × Med           | Redemption record includes cashier_user_id and device_fingerprint; signed by device |
| T-R03 | Merchant disputes a sale              | Low × Med           | Order log with payment metadata; PDF statement with timestamps                      |
| T-R04 | User denies they purchased a coupon   | Low × Med           | Stripe-side payment record + Volu order record reconciled daily                     |

### I — Information Disclosure

| ID    | Threat                                                       | Likelihood × Impact | Mitigations                                                                                 |
| ----- | ------------------------------------------------------------ | ------------------- | ------------------------------------------------------------------------------------------- |
| T-I01 | KYC documents leaked                                         | Low × Critical      | Encrypted at rest with separate KMS key; private S3 bucket; signed URLs only; access logged |
| T-I02 | User PII leaked from API response                            | Med × High          | Field-level access control; admin-only fields hidden in user responses; redaction in logs   |
| T-I03 | PII in logs                                                  | Med × Med           | Structured logger with PII redaction filter; PII scanner on log streams                     |
| T-I04 | Cross-tenant data leak (merchant A sees merchant B's data)   | Low × High          | Resource-scoped checks in service layer; integration tests cover negative cases             |
| T-I05 | API enumeration via incremental IDs                          | Low × Med           | UUIDs as PKs (UUID v7); rate limiting; resource-level auth                                  |
| T-I06 | Stack traces or internal errors exposed to clients           | Med × Med           | Global exception filter sanitises responses; details only in server logs                    |
| T-I07 | Source code containing secrets pushed                        | Low × High          | Pre-commit hook (git-secrets); CI secret scan; AWS Secrets Manager mandatory for runtime    |
| T-I08 | Merchant browses other merchant's deals via path enumeration | Low × Med           | Server-side ownership check on every merchant endpoint                                      |
| T-I09 | Side-channel: timing attack reveals if user exists           | Low × Low           | Constant-time compares; uniform error responses for "user not found" vs "wrong password"    |

### D — Denial of Service

| ID    | Threat                                                                    | Likelihood × Impact | Mitigations                                                                             |
| ----- | ------------------------------------------------------------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| T-D01 | Volumetric DDoS                                                           | Med × High          | Cloudflare DDoS protection; ALB scaling; rate limiting at multiple layers               |
| T-D02 | App-layer flood (slow loris, complex queries)                             | Med × Med           | Cloudflare WAF rules; per-endpoint rate limits; request timeouts; query complexity caps |
| T-D03 | Inventory-locking attack: rapid add-to-cart on flash deals to deny others | Med × Med           | Inventory reserved only at checkout (not cart); cart items expire; rate limit per user  |
| T-D04 | OTP flood to drain SMS budget                                             | Med × High          | Per-phone OTP rate limit; per-IP limit; daily caps; CAPTCHA after suspicious patterns   |
| T-D05 | Webhook flood from malicious source                                       | Low × Med           | Provider whitelist; signature verification immediately rejects unsigned                 |
| T-D06 | Resource exhaustion via large payloads                                    | Low × Med           | Body-size limits (10MB default; lower for non-upload endpoints)                         |
| T-D07 | DB connection pool exhaustion                                             | Low × High          | PgBouncer; connection limits; long-running queries killed; query timeout                |
| T-D08 | Worker queue flooding (event storm)                                       | Low × Med           | Queue rate limits; DLQ; auto-scale workers on depth                                     |

### E — Elevation of Privilege

| ID    | Threat                                      | Likelihood × Impact | Mitigations                                                                                      |
| ----- | ------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------ |
| T-E01 | User token used on admin endpoint           | Low × Critical      | Role claim in JWT; per-endpoint role guard; admin endpoints reject non-admin tokens              |
| T-E02 | Cashier escalates to merchant owner         | Low × High          | Role-stamped JWT; service-level role check; staff invite + accept flow audited                   |
| T-E03 | Admin role-changes themselves to Super      | Low × Critical      | Role assignment guarded; Super role assignment requires existing Super; alert on any role change |
| T-E04 | RCE via vulnerable dependency               | Low × Critical      | Snyk + Dependabot; auto-PRs; high-severity CVEs SLA 7 days                                       |
| T-E05 | RCE via image-processing library (KYC docs) | Low × Critical      | Use vetted libraries (Sharp); dedicated worker process with reduced privileges; sandboxing       |
| T-E06 | Insider promotes themselves via direct DB   | Low × Critical      | DB writes from prod restricted to app role; admin DB access requires JIT approval + audit        |

---

## Specific risk: coupon fraud

A focused threat surface because it's our business.

### F-01 — Sharing a coupon (intended use? or fraud?)

**Scenario:** User buys a coupon, screenshots the QR + PIN, sends to a friend who redeems first.

**Risk:** Low — PIN gating means screenshot alone isn't enough; user must voluntarily reveal PIN. Each redemption ties the coupon to that one transaction.

**Mitigation:**

- PIN regenerates when user explicitly gifts coupon.
- Multi-use coupons can be deliberate (e.g., 5-visit pass).
- Single-use coupons can only be redeemed once regardless of who shows it.

### F-02 — Duplicate coupons via QR replay

**Scenario:** Attacker captures the QR token and tries to redeem it twice.

**Mitigation:**

- Server enforces single-use via DB constraint (`SELECT FOR UPDATE` on coupon row).
- After redemption, status moves to `redeemed`; subsequent redemptions rejected.

### F-03 — Cashier collusion

**Scenario:** A cashier "redeems" a coupon for cash without the customer present, pocketing the difference.

**Mitigation:**

- All redemptions logged with cashier ID, branch, device, timestamp.
- Anomaly detection on cashier redemption patterns vs merchant baseline.
- Random sampling for spot-audit.
- Customer review prompt at +24h flags missing service.
- Merchant balance only credited after redemption — incentive for merchant to monitor staff.

### F-04 — Offline double-spend

**Scenario:** Merchant device offline; same coupon scanned at two different branches simultaneously.

**Mitigation:**

- Server is single source of truth; on sync, second redemption rejected with `rejected_double_spend`.
- Merchant app shows the rejected redemption to staff for awareness.
- Merchant policy: customer presents coupon = service rendered, even if rejected on sync; rare event, absorbed by platform.

### F-05 — Refund-then-redeem race

**Scenario:** User requests refund AND tries to redeem the coupon at a merchant simultaneously.

**Mitigation:**

- Refund request locks coupon (`status='refund_pending'`); redemption rejected.
- Redemption locks coupon during transaction; refund request rejected.
- Optimistic locking with version increments + retry.

---

## Threats we accept

Not every threat is mitigated. Conscious acceptances:

- **Lost devices**: user loses phone with active sessions. Mitigated by user-initiated logout-all and device list management. Time-to-detect determined by user.
- **Sophisticated nation-state actor**: outside threat model. We design to industry standard, not military.
- **Coordinated insider conspiracy**: one rogue employee bounded; multiple colluding requires governance + audit.

These are documented; reviewed annually for whether to upgrade.

---

## Review cadence

- Quarterly threat model review by Security Lead + Tech Lead.
- After every major architectural change.
- After every security incident.
- Annual external pen-test informs updates.

---

## See also

- [Security Controls](./02-security-controls.md)
- [PCI / PDPL Compliance](./03-pci-pdpl-compliance.md)
- [Incident Response](./04-incident-response.md)
