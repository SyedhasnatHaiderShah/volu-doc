# 03 · PCI & PDPL Compliance

**Status:** 🟢 Approved

How Volu meets the regulatory bar required to operate in the UAE — covering payment-card industry standards (PCI DSS), UAE Personal Data Protection Law (PDPL), and UAE Federal Tax Authority (FTA) requirements.

---

## PCI DSS

### Scope: SAQ-A

Volu is intentionally designed to qualify for **PCI SAQ-A** — the lightest PCI assessment level — by ensuring that **no cardholder data ever touches Volu's servers, network, or storage.**

How we achieve this:

| Step | Implementation |
|---|---|
| Card data never reaches our backend | Stripe Elements (web) and Stripe Mobile SDKs render the card form; PAN goes directly from the user's device to Stripe's PCI-DSS-Level-1 environment. |
| Tokenisation | Stripe returns a payment-method ID; only this token is sent to Volu. |
| No PAN in storage, logs, or backups | Audited via grep + DLP scanners on log streams; storage schemas reviewed for any field that could hold PAN. |
| No PAN in metadata | Stripe `metadata` fields populated only with our own identifiers (order ID, user ID); never card data. |
| Secure transmission to Stripe | Stripe's domain hosted directly in the iframe / SDK; HTTPS enforced. |
| Vulnerability management | Quarterly external scans (ASV); covered under our Snyk + Dependabot pipeline. |

### Controls explicitly required

- All payment forms served over HTTPS with valid certificates.
- 3DS / Strong Customer Authentication for high-value transactions (mandatory for SAR/AED transactions on most networks).
- Recurring annual SAQ-A self-assessment + supporting evidence (kept in `/compliance/pci/`).
- Stripe's Attestation of Compliance retained in compliance records.

### What we DO collect about payment

Without storing PAN:

- Card brand (Visa, Mastercard, Amex)
- Last 4 digits
- Expiry month/year
- Cardholder name (optional)
- Stripe payment-method token (`pm_...`)
- Stripe customer ID (`cus_...`)

These fields are stored in `payments__payments` and are explicitly out of PCI scope.

### Annual checklist

- [ ] Re-validate SAQ-A
- [ ] Receive Stripe AoC
- [ ] External ASV scan (quarterly)
- [ ] Internal vulnerability scan (quarterly)
- [ ] Penetration test
- [ ] Security awareness training for engineers
- [ ] Update incident response plan

---

## UAE PDPL

The UAE Personal Data Protection Law (Federal Decree-Law No. 45 of 2021) governs how personal data of UAE residents must be handled. Volu is a **data controller** for user data and a **processor** for some merchant-shared data.

### Lawful basis for processing

We rely on:

| Basis | Use case |
|---|---|
| Contract | Processing user data to deliver coupons, payments, and support |
| Consent | Marketing communications, optional features (e.g., location-based recommendations) |
| Legal obligation | Tax records, KYC retention, fraud investigation |
| Legitimate interest | Security analytics, fraud prevention (with clear notice) |

### Required notices

- **Privacy Notice** — published in EN + AR; linked from sign-up, settings, and footer.
- **Specific consents** — separate explicit opt-ins for: marketing push, marketing email, marketing SMS, marketing WhatsApp, location-based recommendations, sensitive data processing.
- **No pre-checked consent** — opt-in checkboxes default to unchecked.

### Data subject rights — implementation

| Right | Implementation |
|---|---|
| Right to be informed | Privacy Notice + just-in-time disclosures at point of data collection |
| Right of access | `POST /users/me/data-export` — JSON export delivered within 24 hours via signed URL (72h TTL) |
| Right to rectification | User can edit profile fields directly; for fields they can't edit (e.g., DOB after verification), support ticket |
| Right to erasure | `DELETE /users/me` — soft-delete immediately; PII anonymised; financial records retained 7 years per tax law; hard-delete other data after 30 days |
| Right to restrict processing | Supported via consent toggles in settings |
| Right to object | Marketing opt-out; support handles other objections case-by-case |
| Right to data portability | Same data export endpoint; format is structured JSON |

### Data residency

All personal data of UAE users stored in **AWS me-central-1 (Bahrain)**. Cross-region replication is for backup only and stays within ME zone (KSA).

Where third-party processors handle data outside the UAE (e.g., Sentry servers), we have processor agreements in place + transparent disclosure in the Privacy Notice.

### Sub-processors register

A live register of all sub-processors maintained at `/compliance/pdpl/sub-processors.md`:

- **AWS** (UAE/KSA regions) — infrastructure
- **Cloudflare** — CDN, WAF (transient request data)
- **Stripe** (UAE entity) — payments
- **Tabby**, **Tamara** — BNPL
- **Unifonic**, **Twilio** — SMS
- **Resend / AWS SES** — email
- **Firebase / APNs** — push
- **WhatsApp Business** — messaging
- **Sentry** — error monitoring (PII redacted before leaving our infra)
- **PostHog** — analytics (self-hosted in our AWS account; anonymised user IDs)
- **Mixpanel** — analytics (anonymised IDs)
- **GA4** — acquisition (consent-gated)
- **IDfy / Sumsub** — KYC OCR
- **Meta**, **TikTok** — advertising (Conversions API uses hashed identifiers)
- **Intercom** — support chat

Adding a new sub-processor requires Privacy Notice update + 30 days' notice to users where required.

### Breach notification

If a personal data breach occurs:

1. Contain the breach.
2. Within 72 hours: notify the UAE Data Office (or the relevant authority).
3. Notify affected users without undue delay if there's high risk to their rights.
4. Document the breach in the breach register.

Detailed runbook in [Incident Response](./04-incident-response.md).

### Data Protection Impact Assessment (DPIA)

A DPIA is conducted for any processing that's likely high risk:

- New types of profiling
- Sharing data with new categories of recipients
- New use of biometric data (e.g., face verification for payouts — when introduced)
- Processing of children's data (raffle eligibility check)

DPIAs stored in `/compliance/pdpl/dpia/`.

### Records of Processing Activities (ROPA)

Maintained at `/compliance/pdpl/ropa.md`. Documents every processing activity, lawful basis, retention period, and recipients.

---

## UAE FTA — VAT & e-Invoicing

### VAT registration

- Volu registers for VAT once turnover threshold is exceeded (mandatory at AED 375K annual; voluntary above AED 187,500).
- TRN obtained and displayed on every invoice and on the website footer.
- VAT is 5% on most B2C and B2B transactions in scope.

### Invoice requirements

Every invoice must include:

- TRN of supplier (Volu)
- TRN of recipient (if VAT-registered business)
- Sequential invoice number
- Date of issue + date of supply
- Description of goods/services (deal title)
- Unit price + quantity
- VAT rate (5%) + VAT amount
- Total inclusive of VAT
- Currency (AED)

Three invoice types we issue:

1. **Consumer tax invoice** — to end-user per order.
2. **Merchant commission invoice** — Volu → merchant, monthly, for commission earned.
3. **Merchant payout statement** — Volu → merchant per payout, listing redemptions.

### e-Invoicing (FTA Phase 2)

UAE is rolling out mandatory e-invoicing (Phase 2). Volu's invoice generator outputs both PDF and structured XML compatible with the expected format. We connect to a certified service provider (CSP) when the mandate becomes effective.

### VAT reporting

- Monthly VAT return prepared from `invoicing__invoices` data.
- Output VAT (5% on coupon sales) - Input VAT (paid to merchants) = net payable.
- Filed via FTA portal by Finance Admin.

### Retention

Tax records retained 7 years. KYC records retained 7 years post-merchant-offboarding.

---

## Anti-Money Laundering (AML)

While Volu is not a regulated financial institution, we apply prudent AML hygiene:

- Merchant KYC verifies legitimate UAE business identity.
- Payouts only to verified IBANs matching the merchant's legal name.
- Transaction monitoring: alerts on unusual patterns (e.g., abnormally high refund rates, sudden deal price changes).
- Records retained per UAE financial regulation (7 years).

If volume grows or product changes (e.g., introducing peer-to-peer transfers), formal AML programme will be put in place.

---

## Consumer Protection

UAE Consumer Protection Law applies. Volu's practices:

- Clear refund policy displayed at point of sale.
- All-in price (including VAT) shown before payment.
- No hidden fees or surcharges.
- Transparent T&Cs in EN + AR.
- Dispute resolution via in-app chat + UAE consumer-protection authority as last resort.
- Misleading advertising prohibited (deal claims auto-checked + manually moderated).

---

## Prize-promotion (raffle) compliance

UAE has restrictions on lotteries and prize promotions:

- Raffles operated as prize promotions, not lotteries.
- **Free-entry alternative path** offered for every raffle (daily check-in awards entry without purchase).
- T&Cs of every raffle published in EN + AR including: eligibility, period, prize, draw method, claim window.
- Verifiable random draws (commit-reveal seed + public block hash) provide transparency.
- Where local guidance changes, we adapt — likely we'll engage local counsel for specific raffles with prizes > AED 50,000.

---

## Internal controls

### Compliance team
A designated compliance owner (Finance / Operations Lead) maintains the compliance calendar, sub-processor register, ROPA, DPIAs, and incident log.

### Compliance calendar
| Activity | Cadence |
|---|---|
| Privacy Notice review | Annually + on material change |
| Sub-processor register review | Quarterly |
| ROPA review | Annually |
| DPIA review for processing | On change |
| VAT return | Monthly / quarterly per FTA |
| PCI SAQ-A self-assessment | Annually |
| Pen test | Annually |
| Security awareness training | Quarterly |
| Phishing simulation | Annually |
| Disaster recovery drill | Quarterly |
| Backup restore test | Monthly |
| Access review | Quarterly (every employee's access reviewed) |

### Document repository
All compliance evidence in `/compliance/` (separate private repo with restricted access).

---

## See also

- [Threat Model](./01-threat-model.md)
- [Security Controls](./02-security-controls.md)
- [Incident Response](./04-incident-response.md)
- [DR & Backup](../07-infrastructure/06-dr-backup.md)
