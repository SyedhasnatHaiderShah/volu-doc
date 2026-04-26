# 01 · Glossary

**Status:** 🟢 Approved

Every domain term used across the platform. If a word appears in code, UI copy, or documentation, it should be defined here. When in doubt, add it.

---

## A

**Admin.** A member of the Volu team with elevated privileges in the admin console. Sub-roles: Super Admin, Operations, Finance, Marketing, Support.

**ADR.** Architecture Decision Record. A short markdown document in `/docs/adr/` capturing a significant technical decision, its context, and consequences.

**Anchor Merchant.** One of the first 50–100 merchants signed through sales-led onboarding. Often on reduced-commission terms for 90 days.

**AOV.** Average Order Value. Sum of order totals divided by order count over a time window.

---

## B

**Branch.** A physical location of a merchant. A merchant can have many branches. Deals can be redeemed at one, some, or all branches.

**Breakage.** Coupon value that was paid for but never redeemed before expiry. Under Volu's revenue model, breakage is 100% platform margin.

**BNPL.** Buy Now Pay Later. Implemented via Tabby and Tamara.

---

## C

**Cashier.** A merchant staff member with the Scanner sub-role. Uses the merchant app to redeem coupons. Has no access to finance data.

**Commission.** The percentage of a coupon's sale price retained by Volu. Configurable globally, per-category, and per-merchant.

**Commit-Reveal.** A cryptographic scheme used for verifiable raffle draws. Volu commits to a seed hash before draw; reveals the seed after; seed combined with a public future value (e.g. Bitcoin block hash) produces the winner.

**Coupon.** A single unit of purchased deal value. Identified by a UUID, a short code, a QR payload, and a 4-digit PIN. A single purchase of quantity N creates N coupons.

**Coupon Wallet.** The "My Volus" section of the user app where active/used/expired coupons live.

---

## D

**Deal.** A merchant's offer published on Volu. Deals can be **Flash** (time-boxed with scarcity) or **Standard** (open availability within a window). Each deal has one or many coupons associated with its purchases.

**DLQ.** Dead-Letter Queue. Where failed background jobs go after exhausting retries.

**Domain Event.** An immutable record that something happened in the system (e.g., `CouponPurchased`, `CouponRedeemed`). Used to decouple side effects.

**Drop.** Colloquial for a new deal going live. "Today's Volu drops."

---

## E

**e-Invoice.** UAE FTA Phase-2-compliant electronic invoice format. Volu generates these for all VAT-relevant transactions.

---

## F

**FCM.** Firebase Cloud Messaging. Used for push notifications.

**Flash Deal.** A time-boxed deal with a visible countdown timer and limited inventory. Contrasted with a Standard Deal.

**FR.** Functional Requirement. Identified as `FR-XXX`.

**FTA.** UAE Federal Tax Authority. Owns VAT regulations and e-invoicing mandates.

---

## G

**GMV.** Gross Merchandise Value. Total value of all coupons sold before refunds. Reported in AED.

---

## H

**Hold Period.** The time between a successful redemption and the corresponding funds becoming withdrawable to the merchant's balance. Default 7 days; configurable.

---

## I

**IBAN.** International Bank Account Number. Required on merchant KYC for payout destination.

**Idempotency Key.** A client-provided unique key on mutating requests to ensure retries don't double-apply. Mandatory on all payment and payout endpoints.

---

## J

**JWT.** JSON Web Token. Used for authentication. Volu uses short-lived access tokens (15 min) + rotating refresh tokens (30 days).

---

## K

**KPI.** Key Performance Indicator. See `03-success-metrics.md`.

**KYC / KYB.** Know Your Customer / Know Your Business. The document-verification process merchants complete before going live: trade licence, Emirates ID of authorised signatory, VAT certificate (if registered), IBAN letter.

---

## L

**Ledger.** The merchant's running record of `pending` / `available` / `paid_out` / `adjustment` amounts. Every coupon purchase, redemption, and payout writes a row.

---

## M

**Merchant.** A verified UAE business that publishes deals on Volu. Has a legal entity, trade licence, one or more branches, and a bank account for payouts.

**Merchant Exclusivity Agreement.** The contractual commitment that a Volu deal is not simultaneously offered at the same or better price elsewhere (including the merchant's own website) during the deal window.

---

## N

**NFR.** Non-Functional Requirement. Identified as `NFR-XXX`.

---

## O

**Order.** A user's transaction that produces one or more coupons. An order can contain multiple line items across multiple deals.

---

## P

**Payout.** A transfer of available balance from Volu to the merchant's verified IBAN.

**PDPL.** UAE Personal Data Protection Law. Volu is a data controller for user data and must respect PDPL requirements (consent, export, deletion, data residency).

**PIN.** 4-digit numeric code associated with each coupon. User speaks it; cashier enters it. Stored hashed. Never transmitted in the QR payload.

---

## Q

**QR Token.** The signed payload encoded in a coupon's QR code. Contains `{ couponId, dealId, exp, sig }`. Useless without server verification.

---

## R

**Raffle.** A monthly prize draw. Users earn entries based on admin-configured rules (e.g., 1 entry per AED 10 spent). Winners chosen via commit-reveal verifiable randomness.

**Redemption.** The act of a cashier validating a coupon at a merchant branch. Records the coupon, branch, staff member, timestamp, and device metadata.

**RBAC.** Role-Based Access Control. How admin and merchant permissions are scoped.

**ROAS.** Return On Ad Spend. Revenue from promoted-deal sales ÷ ad spend.

---

## S

**SKILL.** See "Deal."  (Used in early docs; migrating away from this term.)

**Standard Deal.** A deal available for purchase anytime within its validity window, without flash-style scarcity. Contrasted with Flash Deal.

**Stripe Connect Express.** Stripe's marketplace payout product. Volu uses it to move merchant balances to merchant IBANs.

---

## T

**TRN.** Tax Registration Number. UAE VAT identifier. Required on invoices.

---

## U

**UC.** Use Case. Identified as `UC-XXX`.

**Unifonic.** UAE-local SMS provider. Primary OTP delivery channel.

**US.** User Story. Identified as `US-XXX`.

---

## V

**Verified Merchant.** A merchant whose KYC has been reviewed and approved by the Ops team. Displayed with a visible badge in the user app.

**Volu.** The product. See also: the brand name, the app icon, and the verb ("Volu'd it before it sold out").

---

## W

**Wallet Ledger.** See "Ledger."

**Webhook.** An outbound HTTP POST from Volu to a partner's endpoint when a domain event occurs.
