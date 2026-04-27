# 00 · Executive Summary

**Status:** 🟢 Approved
**Owner:** Hasnat
**Last updated:** 2026-04-23

---

## What we are building

**Volu** is an exclusive clearance marketplace for the UAE where verified merchants release limited-quantity deals at unbeatable prices, and every coupon purchase enters the buyer into a monthly raffle.

The platform is three connected experiences backed by a single service:

1. **User App** (Flutter, iOS + Android) — discovery, purchase, wallet, redemption, raffle.
2. **Merchant App** (Flutter, iOS + Android) — KYC, deal management, QR scanner, payouts.
3. **Admin Console** (Next.js web) — KYC approval, moderation, finance, raffles, ads, support.

All three are served by a **NestJS modular-monolith backend** with **PostgreSQL**, **Redis**, and **Meilisearch**, deployed on **AWS me-central-1 (Bahrain)** for UAE PDPL compliance.

---

## The problem

Three gaps in the live UAE market:

- **Trust is broken.** Cobone (Trustpilot 1.2–1.3★) and Groupon UAE both get repeated complaints: vouchers rejected at the merchant, hidden top-ups, support unreachable.
- **Engagement is dead.** Competitors email a daily-deals list. Nobody uses the app between purchases.
- **Merchants are in the dark.** Slow payouts, no real-time dashboard, no self-service pacing.

## The wedge

Every one of those complaints becomes a Volu feature:

| Competitor problem           | Volu solution                                                                 |
| ---------------------------- | ----------------------------------------------------------------------------- |
| Voucher rejected at merchant | KYC-gated merchants, contractually bound; 24h full-refund promise if rejected |
| Hidden top-ups at redemption | "All-in-price guarantee" — surcharges = auto-suspend + refund                 |
| Support unreachable          | In-app live chat 7 days/week                                                  |
| No merchant vetting          | Mandatory trade licence + Emirates ID + IBAN verification before going live   |
| No engagement between buys   | Monthly raffle with entries per purchase; real flash deals with countdowns    |
| Merchants paid slowly        | Real-time wallet ledger; self-service payout requests from AED 200            |

---

## Revenue model (hybrid)

| Stream             | How                                    | Rate                       |
| ------------------ | -------------------------------------- | -------------------------- |
| Sales commission   | % of each coupon sold                  | 15–25%                     |
| Breakage profit    | 100% of unredeemed coupon value        | on ~20–35% of sold coupons |
| Featured placement | Paid boost in home feed / category top | AED 300–2,500/week         |
| Performance ads    | Volu runs Meta/TikTok ads for a deal   | Cost-plus + media fee      |
| Volu Pro (future)  | Merchant subscription tier             | AED 299/month              |

**Illustrative:** 100 coupons sold at AED 150 with 20% commission and 70% redemption = **AED 6,600 revenue** vs **AED 3,000 commission-only**.

---

## Platform snapshot

| Dimension      | Choice                                                         |
| -------------- | -------------------------------------------------------------- |
| Geography      | UAE-first (Dubai)                                              |
| Currency       | AED                                                            |
| Languages      | English + Arabic (full RTL)                                    |
| Payment rails  | Stripe cards, Apple Pay, Google Pay, Careem Pay, Tabby, Tamara |
| Data residency | AWS me-central-1 (Bahrain)                                     |
| VAT            | 5% UAE VAT, FTA-compliant invoicing                            |

---

## Key product mechanics

1. **QR + PIN redemption.** User shows QR; cashier scans; user verbally states 4-digit PIN; cashier enters PIN; server validates. Fraud-resistant, works offline, supports multi-use coupons.
2. **Merchant-paid-only-on-redemption.** Platform holds funds until successful redemption. Breakage = 100% platform margin. This is the Groupon model done right.
3. **Configurable raffle engine.** Admin creates monthly raffles with custom entry rules (per-AED spent, per-purchase, bonus for referrals, category multipliers). Verifiable random draws using commit-reveal seeds.
4. **Full Meta + TikTok ads integration.** "Promote this deal" button in admin → creates campaign via API → ROAS dashboard ties spend back to actual coupon sales.

---

## Delivery plan

16-month phased build:

- **Phase 0** (weeks 1–4): Foundations, brand lock, infra, anchor merchant outreach.
- **Phase 1** (weeks 5–14): Core marketplace MVP — discover, buy, redeem (single-use).
- **Phase 2** (weeks 15–22): Multi-use coupons, payouts, invoicing, BNPL, reviews, live chat.
- **Phase 3** (weeks 23–32): Raffles, loyalty, referrals, WhatsApp, wallet passes, public launch.
- **Phase 4** (weeks 33–42): Meta + TikTok ads integration, advanced analytics.
- **Phase 5** (weeks 43–60): Scale, polish, Volu Pro, KSA readiness.

---

## Success looks like

12 months post-public-launch:

- **300+** verified active merchants
- **800+** live deals at any time
- **75,000+** monthly active users
- **≥ 35%** 30-day repeat purchase rate
- **65–80%** coupon redemption rate
- **28–35%** blended platform margin

---

## Deep dive

- Vocabulary: [Glossary](./01-glossary.md)
- Who uses Volu: [Personas](./02-personas.md)
- How we'll measure: [Success Metrics](./03-success-metrics.md)
- The full feature set: [Functional Requirements](../02-requirements/01-functional-requirements.md)
- How it's built: [System Architecture](../03-architecture/01-system-architecture.md)
