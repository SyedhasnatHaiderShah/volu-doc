# 04 · User Stories

**Status:** 🟢 Approved

Agile user stories grouped by epic. Each story has a stable ID (`US-XXX`), maps to one or more FRs, and includes acceptance criteria.

---

## EPIC: User Onboarding

### US-001 · As a new user, I want to sign up with my phone, so I can start exploring deals quickly.

**Acceptance:** UAE OTP flow works in < 30s; account created; can browse without buying.
**FRs:** FR-001, FR-005.

### US-002 · As a tourist, I want to use my foreign credit card, so I can buy deals during my visit.

**Acceptance:** International card Stripe flow works; 3DS handled; coupon delivered to email.
**FRs:** FR-001 (international SMS fallback), FR-406.

### US-003 · As a returning user, I want biometric unlock, so I don't retype credentials.

**Acceptance:** FaceID/fingerprint enabled in settings; opens app and reveals coupon PINs.
**FRs:** FR-006.

### US-004 · As a privacy-conscious user, I want to delete my account, so I control my data.

**Acceptance:** Account soft-deleted within seconds; PII anonymised; financial records retained per regulation; user receives confirmation email.
**FRs:** FR-009, NFR-501.

---

## EPIC: Discovery

### US-101 · As a user, I want to see flash deals with countdowns, so I can act before they expire.

**Acceptance:** Home shows "Today's Volu Drops" section with live countdown; tap opens detail.
**FRs:** FR-301, FR-203.

### US-102 · As a user, I want filters that respect my preferences, so I see relevant deals.

**Acceptance:** Filter by price, distance, rating, sub-category, dietary tags persist within session.
**FRs:** FR-304.

### US-103 · As an Arabic speaker, I want a fully localized RTL experience, so the app feels native.

**Acceptance:** Switch to AR; all screens RTL with translated copy; no truncation.
**FRs:** FR-302 (i18n), NFR-603.

### US-104 · As a user, I want to find deals near me, so I can act today.

**Acceptance:** "Near You" sorted by distance; map view shows deal pins.
**FRs:** FR-301, FR-305.

### US-105 · As a user, I want to save favourite deals, so I can come back to them.

**Acceptance:** Heart icon toggles save state; saved list in profile; notification when low inventory.
**FRs:** FR-308.

---

## EPIC: Purchase

### US-201 · As a user, I want to pay with Apple Pay, so checkout is one tap.

**Acceptance:** Apple Pay button on cart; biometric confirmation; coupon delivered.
**FRs:** FR-407, NFR-107.

### US-202 · As a user on a budget, I want to pay in instalments via Tabby, so I can buy now.

**Acceptance:** Tabby option visible when order ≥ AED 150; SDK flow returns approval; order completes.
**FRs:** FR-410.

### US-203 · As a user, I want a clear all-in price before paying, so there are no surprises.

**Acceptance:** Cart shows subtotal, promo discount, wallet credit, VAT, grand total separately.
**FRs:** FR-403, NFR-607.

### US-204 · As a user, I want to apply a promo code, so I can save more.

**Acceptance:** Promo input on cart; validates server-side; shows clear success or specific error.
**FRs:** FR-404.

### US-205 · As a referrer, I want to share my referral code, so I earn wallet credit.

**Acceptance:** Share sheet includes my code; referee gets new-user discount; I get credit on their first redemption.
**FRs:** FR-904, FR-905.

---

## EPIC: Wallet & Redemption

### US-301 · As a user, I want all my coupons in one place, so I never lose them.

**Acceptance:** "My Volus" tab; tabs for Active/Used/Expired/Refunded; sorted by expiry.
**FRs:** FR-502.

### US-302 · As a user, I want to add coupons to Apple Wallet, so I can quickly access them at the merchant.

**Acceptance:** "Add to Wallet" generates pkpass; QR + expiry shown in lock screen.
**FRs:** FR-505.

### US-303 · As a user, I want my PIN protected by biometric, so others can't redeem my coupons.

**Acceptance:** Coupon PIN hidden until biometric unlock.
**FRs:** FR-503, FR-006.

### US-304 · As a user, I want to gift a coupon, so I can treat someone.

**Acceptance:** Gift flow regenerates QR + PIN; old invalidated; recipient gets deep link.
**FRs:** FR-507.

### US-305 · As a cashier, I want to scan and validate coupons in seconds, so the queue moves.

**Acceptance:** Scan + PIN entry + result in ≤ 1.5s online; clear pass/fail banner.
**FRs:** FR-601, FR-603, NFR-104.

### US-306 · As a cashier, I want to keep redeeming when offline, so a network drop doesn't stop business.

**Acceptance:** Up to 4h offline; redemptions queued; sync on reconnect; no double-spend.
**FRs:** FR-606, NFR-307.

### US-307 · As a user, I want to refund within 24h if I change my mind, so I'm not stuck.

**Acceptance:** Self-service refund button on coupons < 24h old and unredeemed; processed via Stripe; raffle entries reversed.
**FRs:** FR-508.

---

## EPIC: Raffle & Loyalty

### US-401 · As a user, I want to see my raffle entries clearly, so I know my chances.

**Acceptance:** Raffles tab shows total entries, my entries, broken down by source, time to draw.
**FRs:** FR-806.

### US-402 · As a non-purchaser, I want a free way to enter, so the raffle is fair.

**Acceptance:** Daily check-in awards 1 free entry; tracked separately.
**FRs:** FR-805.

### US-403 · As a winner, I want to be notified immediately, so I can claim.

**Acceptance:** Push + SMS + email + in-app banner within seconds of draw.
**FRs:** FR-808.

### US-404 · As a sceptic, I want proof the draw was random, so I trust the platform.

**Acceptance:** Public audit log of seed commitment, reveal, block hash, entries.
**FRs:** FR-807.

### US-405 · As a frequent user, I want loyalty points so my spending is rewarded.

**Acceptance:** Points earned on every purchase; redeemable for discounts or extra raffle entries.
**FRs:** FR-901, FR-902.

---

## EPIC: Merchant Onboarding

### US-501 · As a merchant, I want a clear go-live checklist, so I know what to do.

**Acceptance:** Wizard with progress bar: profile → branches → KYC → bank → first deal.
**FRs:** FR-011–FR-012.

### US-502 · As a merchant, I want to know my KYC status in real time, so I'm not in the dark.

**Acceptance:** Status visible in app: pending / under review / approved / rejected / needs info; reason shown if not approved.
**FRs:** FR-013.

### US-503 · As a merchant, I want my trade licence to auto-extract data, so I don't retype.

**Acceptance:** OCR extracts and pre-fills business name, licence number, expiry; merchant verifies.
**FRs:** FR-012 (OCR via IDfy/Sumsub).

---

## EPIC: Merchant Deal Management

### US-601 · As a merchant, I want to create flash deals to clear midweek capacity.

**Acceptance:** Flash deal type with countdown + inventory cap; visible to users with countdown.
**FRs:** FR-201, FR-203.

### US-602 · As a merchant, I want to set per-day caps per branch, so I'm not overwhelmed.

**Acceptance:** Deal config supports max-per-day-per-branch; respected at purchase time.
**FRs:** FR-201.

### US-603 · As a multi-branch merchant, I want users to choose their branch at purchase, so I prevent confusion.

**Acceptance:** Multi-branch deals show branch selector at purchase; coupon bound to chosen branch.
**FRs:** FR-201, FR-602.

### US-604 · As a merchant, I want to preview my deal as users will see it, so I avoid mistakes.

**Acceptance:** Preview mode mirrors user app deal page exactly.
**FRs:** FR-206.

### US-605 · As a merchant, I want to pause a live deal when I'm overbooked.

**Acceptance:** Pause button hides deal from new buyers; existing coupons still redeemable.
**FRs:** FR-209.

### US-606 · As a merchant, I want a multi-use coupon for visit packs.

**Acceptance:** Configure usage_type=multi-use with N uses; cashier sees uses-remaining each scan.
**FRs:** FR-214.

---

## EPIC: Merchant Finance

### US-701 · As a merchant, I want to see real-time sales, so I can manage operations.

**Acceptance:** Dashboard shows sold today/7d/30d, redemption rate, revenue, pending payout — refreshed in < 10s.
**FRs:** FR-106, NFR-101.

### US-702 · As a merchant, I want to request a payout when I want, so cashflow works for me.

**Acceptance:** Self-service payout request when available ≥ AED 200; status visible until paid.
**FRs:** FR-703.

### US-703 · As a merchant, I want a clear PDF statement, so I can reconcile and file.

**Acceptance:** Per-payout PDF with TRN, line items, fees, VAT, net amount, bank reference.
**FRs:** FR-706.

### US-704 · As a merchant, I want to see every commission charged, so I trust the math.

**Acceptance:** Per-coupon ledger entry; per-payout commission detail.
**FRs:** FR-702, FR-707.

---

## EPIC: Admin

### US-801 · As an admin, I want a single dashboard with key KPIs, so I know how the platform is doing.

**Acceptance:** GMV, revenue, active deals, pending approvals, pending payouts, redemption rate, refund rate, with time-series charts.
**FRs:** FR-1101.

### US-802 · As an Operations admin, I want to clear KYC and deal queues efficiently, so SLAs are met.

**Acceptance:** Both queues sorted oldest-first; bulk approve where possible; templated reasons for reject/info.
**FRs:** FR-013, FR-207.

### US-803 · As a Finance admin, I want a daily reconciliation report, so discrepancies don't accumulate.

**Acceptance:** Stripe vs ledger diff > AED 1 flagged daily; investigated within 1 day.
**FRs:** FR-710.

### US-804 · As a Marketing admin, I want to launch ads from inside the deal page, so I save context-switching time.

**Acceptance:** "Promote" button opens wizard; campaign live in 2 minutes; ROAS in dashboard.
**FRs:** FR-1203, FR-1206.

### US-805 · As a Marketing admin, I want to test raffle entry rules before publishing.

**Acceptance:** Simulation against historical data shows expected entry distribution.
**FRs:** FR-803.

### US-806 · As a Support admin, I want to find any coupon by any identifier, so I can help a user fast.

**Acceptance:** Search by short code, user phone, merchant, order id, transaction id; results in < 1s.
**FRs:** FR-1104.

### US-807 · As a Super admin, I want every privileged action logged, so we have an audit trail.

**Acceptance:** Audit log shows actor, action, target, before/after, IP, timestamp; immutable, queryable.
**FRs:** NFR-407.

---

## EPIC: Trust & Safety

### US-901 · As a user, I want a clear refund promise, so I'm not afraid to buy.

**Acceptance:** "Refund Promise" surfaced on home and deal page; 24h self-service refund + support escalation otherwise.
**FRs:** FR-508, FR-509.

### US-902 · As a user, I want live chat support 7 days a week, so I'm never stuck.

**Acceptance:** Chat available in-app; median response < 4h business hours.
**FRs:** FR-1008, NFR-303 (this is product not infra but support SLO).

### US-903 · As a user, I want to know merchants are verified, so I trust the listing.

**Acceptance:** Verified badge with tooltip explaining what was checked.
**FRs:** FR-105.

### US-904 · As a user, I want my data secured, so I'm comfortable shopping.

**Acceptance:** TLS 1.3, biometric lock, no PAN storage, PDPL compliance.
**FRs:** NFR-401–404, NFR-501–502.

---

## EPIC: Notifications

### US-1001 · As a user, I want to control what I'm notified about.

**Acceptance:** Settings let me toggle: deal drops, flash, raffle, coupon expiry, marketing — separately.
**FRs:** FR-1001.

### US-1002 · As a user, I want a reminder before my coupon expires, so I don't waste it.

**Acceptance:** Pushes at 7d / 3d / 24h / 3h before expiry, respecting prefs.
**FRs:** FR-1002.

### US-1003 · As a merchant, I want a daily redemption summary, so I can review at end of day.

**Acceptance:** End-of-business-day push + email summary.
**FRs:** FR-1004.

---

## Story sizing & sequencing

Stories sized in points (Fibonacci 1, 2, 3, 5, 8, 13). Stories > 13 points must be split. Stories grouped into sprints (2-week) and mapped to roadmap phases.

| Phase         | Major epics                                                                                         |
| ------------- | --------------------------------------------------------------------------------------------------- |
| Phase 1 (MVP) | Onboarding, Discovery, Purchase, Wallet & Redemption, Merchant Onboarding, Merchant Deal Management |
| Phase 2       | Merchant Finance, Trust & Safety (refunds + reviews + live chat), BNPL/wallets                      |
| Phase 3       | Raffle & Loyalty, Notifications (advanced), referrals                                               |
| Phase 4       | Admin (Ads), advanced analytics                                                                     |
| Phase 5       | Polish, Volu Pro                                                                                    |

---

## Story to test mapping

Every story has a corresponding E2E test in the test plan and can be demoed independently. PR title format: `[US-XXX] short description`.
