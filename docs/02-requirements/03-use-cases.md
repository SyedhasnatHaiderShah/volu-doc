# 03 · Use Cases

**Status:** 🟢 Approved

Detailed use cases for the most important user, merchant, and admin flows. Each use case has stable ID `UC-XXX`, actors, pre-conditions, main flow, alternate flows, post-conditions, and exception handling.

---

## USER USE CASES (UC-1xx)

---

### UC-101 · Register and verify (new user via phone)

**Primary Actor:** Prospective end-user
**Pre-conditions:** App installed; user has UAE mobile number; user has internet connection.
**Trigger:** User taps "Sign up" on welcome screen.

**Main Flow:**

1. User selects "Continue with phone."
2. User enters UAE mobile number (`+971XXXXXXXXX`).
3. System validates number format; sends OTP via Unifonic SMS.
4. User receives SMS with 6-digit OTP within 30 seconds.
5. User enters OTP.
6. System validates OTP (server-side, < 5 minutes since issuance).
7. System creates user record, marks phone as verified.
8. System prompts user for: name, optional email, language preference, notification preferences.
9. User completes profile.
10. System binds device to user account, issues JWT + refresh token.
11. User lands on Home with personalised feed.

**Alternate Flows:**

- **A1** (OTP not received): User taps "Resend." Cooldown of 60 seconds enforced; max 3 resends per 24h.
- **A2** (Sign in with Apple/Google instead): User selects social provider; system auto-creates account; still required to add and verify phone before purchasing.
- **A3** (Tourist with foreign number): System accepts international format; sends international SMS via Twilio fallback.

**Post-conditions:** User account created; user is signed in; can browse but cannot purchase until phone-verified.

**Exceptions:**

- **E1** (3 failed OTPs): Account temporarily locked for 15 minutes; user shown helpful error.
- **E2** (Phone already registered): User offered "Sign in" flow instead.
- **E3** (SMS provider failure): Fallback to Twilio; if both fail, in-app message: "We're having trouble sending your code. Please try email or contact support."

**Related:** FR-001, FR-005, NFR-606, NFR-607.

---

### UC-102 · Browse and discover deals

**Primary Actor:** Authenticated user
**Pre-conditions:** User signed in; location permission granted (or skipped).
**Trigger:** User opens app or navigates to Home.

**Main Flow:**

1. System loads home feed: hero carousel, Today's Volu Drops (flash deals with countdown), Trending, Near You (geo-sorted), New This Week, Categories.
2. User scrolls; system fetches additional sections lazily.
3. User taps a category chip; system navigates to category list view.
4. User applies filters (price, distance, rating, sub-category, badges).
5. User taps a deal card.
6. System opens deal detail page with hero gallery, price block, "What you get," fine print, branches, reviews, similar deals.

**Alternate Flows:**

- **A1** (Search): User taps search; types a term; sees autocomplete; selects a result.
- **A2** (Map view): User toggles to map; sees deal pins; taps a pin to see deal preview; taps preview to open detail.
- **A3** (Anonymous browse): Unauthenticated user can browse but not purchase. Tap "Buy" → Sign-up prompt.

**Post-conditions:** User has viewed deal detail; deal-view event recorded for analytics and Conversions API.

**Exceptions:**

- **E1** (No deals match filters): Empty state with "Clear filters" CTA.
- **E2** (Search service down): Fallback to category browse with notice.

**Related:** FR-301, FR-302, FR-303, FR-304, FR-305, FR-307.

---

### UC-103 · Purchase a coupon

**Primary Actor:** Phone-verified user
**Pre-conditions:** User signed in and phone-verified; valid payment method available; deal is live with inventory.
**Trigger:** User taps "Buy" on deal detail page.

**Main Flow:**

1. User selects quantity (default 1; max = min(per-user cap, remaining inventory)).
2. User taps "Add to cart" or "Buy now."
3. System opens cart with line item, subtotal, all-in price notice.
4. User optionally applies promo code or wallet credit.
5. User taps "Checkout."
6. System shows payment method options (Cards, Apple Pay, Google Pay, Tabby, Tamara, Careem Pay).
7. User selects method.
8. System creates Stripe PaymentIntent (or BNPL session) with idempotency key.
9. User completes payment (3DS challenge if required).
10. System receives Stripe webhook `payment_intent.succeeded`.
11. System: creates coupon records (one per quantity), generates QR token + PIN per coupon, debits merchant ledger `pending` row, issues raffle entries per active raffle's rules, sends push + email + in-app notification with coupons attached.
12. User sees confirmation screen with coupons.

**Alternate Flows:**

- **A1** (Tabby/Tamara approved): BNPL provider returns approval; flow continues from step 10.
- **A2** (Tabby/Tamara rejected): User returned to cart with message; can try alternate method.
- **A3** (Promo code redemption): Step 4 — code validated server-side; discount applied; persists to checkout.
- **A4** (Wallet credit): Applied after VAT calculation, reducing card charge.
- **A5** (Idempotent retry): If user retries payment with same idempotency key, no double charge.

**Post-conditions:** Order in `paid` status; coupons issued and delivered; merchant balance shows pending; analytics events fired (Conversions API to Meta + TikTok).

**Exceptions:**

- **E1** (Inventory exhausted between cart and checkout): Order rejected with clear message; user offered similar deals.
- **E2** (3DS failed): User returned to payment step; can retry or change method.
- **E3** (Payment gateway timeout): Idempotency key ensures safe retry; user sees "Processing..." for up to 30 seconds before actionable error.
- **E4** (Webhook delayed): User polls confirmation status; if no confirmation in 2 minutes, support escalation path shown.

**Related:** FR-401–FR-414, FR-501, NFR-107, NFR-306.

---

### UC-104 · Redeem a coupon at merchant

**Primary Actor:** User (with valid coupon)
**Secondary Actor:** Cashier (merchant staff)
**Pre-conditions:** User has an active coupon for this merchant/branch; cashier is logged into merchant app.
**Trigger:** User arrives at merchant; presents the coupon for redemption.

**Main Flow:**

1. User opens "My Volus" tab in app, finds the active coupon, taps to view full screen.
2. App requests biometric unlock (if enabled) to reveal PIN.
3. App displays large QR code and the 4-digit PIN.
4. Cashier opens merchant app, taps "Scan Volu" on home screen.
5. Cashier points camera at user's QR; app detects QR and sends to server.
6. Server validates: signature valid, coupon status active, branch eligible, within valid days/hours, uses_remaining > 0. Returns "PIN required" response.
7. App prompts cashier: "Ask the customer for their PIN."
8. User reads the 4-digit PIN aloud.
9. Cashier enters PIN.
10. Server validates PIN against the coupon's hashed PIN.
11. On success: server marks coupon redeemed (or decrements uses_remaining), credits merchant ledger pending row → available (after hold period via async job), records redemption with cashier id + branch id + timestamp.
12. Merchant app shows ✅ green banner: "Redeemed: [Deal Title] · [uses left if multi-use]."
13. User app refreshes coupon card to show redeemed/decremented state.

**Alternate Flows:**

- **A1** (Manual code entry): If camera fails, cashier taps "Enter code"; types the 10-char short code; same PIN step follows.
- **A2** (Multi-use coupon): Step 11 decrements uses_remaining by 1 instead of full redemption; cashier sees remaining count.
- **A3** (Offline mode): Server unreachable. Merchant app validates QR signature locally (using public key shipped in app), checks against local cache; queues redemption with cryptographic signature; syncs on reconnect.

**Post-conditions:** Coupon updated; merchant ledger credited; redemption record created; user can review merchant 24h later (post-redemption review prompt).

**Exceptions:**

- **E1** (Invalid QR): "Coupon not found" — cashier can rescan or enter code manually.
- **E2** (Already redeemed, single-use): "Already redeemed at [branch] on [date]."
- **E3** (Multi-use exhausted): "All uses used."
- **E4** (Expired): "Expired on [date]."
- **E5** (Wrong branch): "Not valid at this branch — valid at: [list]."
- **E6** (Outside valid hours): "Valid only [days/hours]."
- **E7** (Wrong PIN, 1st attempt): "Wrong PIN — 2 attempts remaining."
- **E8** (Wrong PIN, 3rd attempt): Coupon locked for 15 minutes; user notified in-app.
- **E9** (Offline sync conflict — double-spend attempt): Second redemption rejected on sync; merchant notified.

**Related:** FR-601–FR-609, NFR-104, NFR-307.

---

### UC-105 · Refund a coupon (self-service)

**Primary Actor:** User who purchased a coupon
**Pre-conditions:** Coupon purchased < 24h ago; coupon not yet redeemed.
**Trigger:** User taps "Refund" on the coupon card.

**Main Flow:**

1. App shows confirmation: "Refund [Deal] for AED [amount]? This action cannot be undone."
2. User confirms.
3. System creates refund request via Stripe API with idempotency key.
4. Stripe processes refund (5–10 business days to user's card, but Volu marks immediately).
5. System: updates coupon status to `refunded`, returns inventory to deal, reverses raffle entries from this purchase, reverses merchant ledger pending entry, records refund in audit log.
6. User receives push + email confirmation.

**Alternate Flows:**

- **A1** (Outside 24h window): Self-service refund disabled. User shown "Contact support" CTA which opens in-app live chat.
- **A2** (Already redeemed): Self-service refund disabled. User shown contact support.

**Post-conditions:** Coupon refunded; user's payment method credited; merchant unaffected (since merchant wasn't yet credited).

**Exceptions:**

- **E1** (Stripe refund fails): User shown error; support team notified; manual refund initiated.

**Related:** FR-508, FR-711.

---

### UC-106 · Enter monthly raffle and win

**Primary Actor:** User who has purchased coupons
**Pre-conditions:** Active monthly raffle exists; user is UAE-resident-eligible per raffle T&Cs.
**Trigger:** Implicit — every qualifying purchase or referral generates raffle entries automatically.

**Main Flow:**

1. User completes a purchase (UC-103).
2. System evaluates raffle's rule engine against the purchase event.
3. System creates raffle_entry records per the rules (e.g., 1 entry per AED 10 spent, capped at 200 entries per user).
4. User views Raffles tab; sees: prize details, total raffle entries, my entries (broken down by source: purchases / referrals / bonuses), time to draw.
5. On draw day:
   - Admin triggers draw at scheduled time.
   - System publishes the previously-committed seed hash.
   - System fetches the public future value (e.g., specified Bitcoin block hash).
   - System combines: `winning_index = HMAC(seed, block_hash) mod total_entries`.
   - System selects winner(s); creates raffle_winner record.
6. Winner notified via push + SMS + email + in-app banner.
7. Winner has 7 days to claim from in-app banner.
8. Admin marks claim received; arranges prize fulfillment; uploads proof photo.
9. Public Past Winners gallery updated.

**Alternate Flows:**

- **A1** (User opts in for free entry): Daily in-app check-in awards 1 entry (no purchase). Tracked as `source = free_entry`.
- **A2** (Winner doesn't claim within 7 days): Automatic re-draw from same entry pool, excluding the original winner.
- **A3** (Tiered prizes): Multiple winners selected for different tiers (Grand prize, 5 runners-up, etc.) — each drawn independently from the entry pool.

**Post-conditions:** Winner notified; entries archived; new raffle for next month auto-created if scheduled.

**Exceptions:**

- **E1** (Draw infrastructure issue): Draw rescheduled within 24h with public communication. Seed commitment preserved.
- **E2** (Eligibility dispute): Admin can investigate via audit log of entries.

**Related:** FR-801–FR-810, FR-805 (free-entry compliance).

---

## MERCHANT USE CASES (UC-2xx)

---

### UC-201 · Onboard merchant (KYC)

**Primary Actor:** Prospective merchant business owner
**Pre-conditions:** Merchant has UAE trade licence, Emirates ID, IBAN.
**Trigger:** User installs merchant app and taps "Sign up."

**Main Flow:**

1. Merchant signs up with business email + mobile.
2. System verifies mobile via OTP (UC-101 reused).
3. Merchant lands on "Go-Live Checklist": (a) Business profile (b) Branches (c) KYC documents (d) Bank details (e) First deal.
4. Merchant completes Business profile: legal name, brand name, categories, contact.
5. Merchant adds at least one Branch with address, Google Maps pin, opening hours.
6. Merchant uploads KYC pack: trade licence PDF, Emirates ID front+back, VAT certificate (if registered), IBAN bank letter.
7. System runs IDfy/Sumsub OCR on documents; extracts and pre-fills key fields for merchant verification.
8. Merchant submits.
9. Status: `pending_kyc` → `under_review`.
10. Operations Admin reviews docs (UC-301); approves, rejects with reason, or requests info.
11. On approval: merchant status → `approved`; merchant receives email + in-app notification "You're live! Create your first deal."
12. Merchant proceeds to Deal Creation (UC-202).

**Alternate Flows:**

- **A1** (Doc rejected): Merchant receives notification with reason; re-uploads.
- **A2** (Sales-led onboarding): Volu Sales team initiates; merchant gets a streamlined link with pre-filled data.

**Post-conditions:** Merchant approved; can publish deals; verified badge active.

**Exceptions:**

- **E1** (KYC SLA breach > 24 business hours): Auto-escalation to Ops Lead.
- **E2** (Document expired during review): Merchant notified to upload fresh.

**Related:** FR-011–FR-014.

---

### UC-202 · Create and publish a deal

**Primary Actor:** Merchant Owner or Branch Manager
**Pre-conditions:** Merchant approved; valid trade licence not expired.
**Trigger:** Merchant taps "Create Deal" in merchant app.

**Main Flow:**

1. Merchant selects deal type: Flash or Standard.
2. Merchant fills form:
   - Title (EN+AR), description (EN+AR)
   - Hero image (uploaded; min 1200×800), gallery (≤8)
   - Original price (AED), Volu deal price
   - Inventory total, max per user, max per day
   - Valid-from date, Valid-until date
   - Eligible branches
   - Redemption rules: days of week, hours, dine-in/takeaway/delivery flags
   - Fine print, what's included, what's excluded, "good to know"
   - Coupon usage type: single-use or multi-use (with uses count)
3. App shows commission preview and estimated payout per redemption.
4. Merchant taps "Preview" to see how users will see it.
5. Merchant taps "Submit for Review."
6. System runs automated quality checks (image resolution, banned terms, duplicate, price sanity).
7. If checks pass, status → `submitted`; admin notified.
8. Admin reviews (UC-302); approves / requests changes / rejects.
9. On approval: status → `scheduled` (if start in future) → `live` (at start time).
10. Merchant sees the deal go live in their dashboard.

**Alternate Flows:**

- **A1** (Save as draft): Merchant can save without submitting; resume later.
- **A2** (Auto-quality-check fails): Merchant shown specific issues to fix before resubmit.
- **A3** (Changes requested): Merchant receives feedback; edits; resubmits.

**Post-conditions:** Deal in user-facing catalogue when live; merchant sees real-time analytics.

**Exceptions:**

- **E1** (Pricing rule violation: deal price > 75% of original): Form rejects submission with clear error.
- **E2** (Trade licence expired): Cannot submit; force renewal flow.

**Related:** FR-201–FR-212.

---

### UC-203 · Cashier scans and validates coupon

(See UC-104 from cashier perspective.)

---

### UC-204 · Request payout

**Primary Actor:** Merchant Owner
**Pre-conditions:** Merchant approved; available balance ≥ AED 200; verified IBAN on file.
**Trigger:** Merchant taps "Request Payout" in merchant wallet.

**Main Flow:**

1. App shows: Available balance, suggested amount (defaults to full available), bank account on file.
2. Merchant confirms amount and IBAN.
3. System creates payout request with status `requested`.
4. Finance Admin notified.
5. Finance Admin reviews (UC-303); approves.
6. System initiates Stripe Connect payout (or manual bank transfer if outside Stripe).
7. Status: `requested` → `approved` → `in_transit`.
8. On bank confirmation: status → `paid`; merchant notified with reference number.
9. System generates PDF statement with line-item redemptions, fees, VAT, net amount.

**Alternate Flows:**

- **A1** (Below threshold): "Available balance must be ≥ AED 200." Merchant sees how much more is needed.
- **A2** (Held due to dispute): Pending balance shown but available is reduced; merchant notified of held amount + reason.
- **A3** (IBAN not yet verified): Payout request blocked; KYC update path shown.

**Post-conditions:** Funds transferred to merchant; ledger updated; PDF statement available; commission invoice from Volu issued.

**Exceptions:**

- **E1** (Payout rejected by finance): Reason shown; merchant can resolve and re-request.
- **E2** (Bank transfer fails): Status → `failed`; finance investigates; reverses ledger.

**Related:** FR-701–FR-707.

---

## ADMIN USE CASES (UC-3xx)

---

### UC-301 · Review and approve merchant KYC

**Primary Actor:** Operations Admin
**Pre-conditions:** Merchant submitted KYC; admin logged in with 2FA.
**Trigger:** Admin opens KYC queue.

**Main Flow:**

1. Admin sees queue sorted by oldest pending.
2. Admin opens a merchant case.
3. Side-by-side view: original docs (PDF/image) and OCR-extracted fields.
4. Admin checks: trade licence valid + matches business name; Emirates ID matches authorised signatory; IBAN letter valid + matches business; VAT certificate valid (if applicable).
5. Admin clicks Approve / Request Info / Reject.
6. If Reject or Request Info: admin selects template reason; can add custom note.
7. System updates merchant status; sends email + in-app notification.
8. Audit log records: admin id, action, before/after state, timestamp.

**Alternate Flows:**

- **A1** (Suspect document): Admin can flag for second review by Ops Lead.
- **A2** (Bulk approve): For pre-vetted batches (e.g., chamber-of-commerce member list), admin can bulk approve.

**Post-conditions:** Merchant transitioned; SLA met or breached (logged for ops review).

**Related:** FR-013, NFR-407.

---

### UC-302 · Moderate deal approval queue

**Primary Actor:** Operations Admin
**Pre-conditions:** Deals in `submitted` state.
**Trigger:** Admin opens deal moderation queue.

**Main Flow:**

1. Queue sorted by submission time (oldest first).
2. Admin opens a deal: side-by-side preview (as user will see it) + form data + automated quality check results.
3. Admin checks: copy is accurate and not misleading; images are appropriate; price math is reasonable; fine print is clear; merchant has no recent quality issues.
4. Admin clicks Approve / Request Changes / Reject.
5. For Request Changes / Reject: admin selects template reason; adds custom notes.
6. System updates deal status; merchant notified.
7. On approval: deal scheduled or goes live immediately based on valid-from.

**Alternate Flows:**

- **A1** (Featured deal): Admin can also tag as Featured during approval, scheduling it for hero carousel.
- **A2** (Edit suggestion): Admin can suggest edits inline; merchant accepts in one click.

**Post-conditions:** Deal in correct lifecycle state; merchant receives next-action clarity.

**Related:** FR-207, FR-208.

---

### UC-303 · Process merchant payouts

**Primary Actor:** Finance Admin
**Pre-conditions:** Pending payout requests exist.
**Trigger:** Admin opens payout queue.

**Main Flow:**

1. Queue sorted by oldest request.
2. Admin opens a payout: shows merchant details, requested amount, available balance, recent dispute history, KYC status, IBAN.
3. Admin verifies: balance accurate, no active disputes freezing the funds, IBAN verified.
4. Admin clicks Approve.
5. System initiates Stripe Connect payout (or schedules manual bank transfer).
6. Status: `requested` → `approved` → `in_transit`.
7. On bank confirmation webhook: status → `paid`; merchant notified.
8. Generates payout statement PDF.

**Alternate Flows:**

- **A1** (Reject): Admin selects reason, e.g., "Active dispute on order #X." Merchant notified.
- **A2** (Bulk approve): Admin can approve multiple low-risk payouts in batch.

**Post-conditions:** Funds disbursed; merchant statement issued; commission invoice from Volu generated.

**Related:** FR-703–FR-706.

---

### UC-304 · Create and run a monthly raffle

**Primary Actor:** Marketing Admin
**Trigger:** Admin opens Raffle Builder.

**Main Flow:**

1. Admin creates raffle: name (EN+AR), period (start/end), prizes (title, value, photos, tier), draw date/time, T&Cs.
2. Admin configures entry rules in rule builder (e.g., "1 entry per AED 10 spent on Volus" + "5 bonus entries for first purchase this month" + cap "max 200 entries per user").
3. Admin runs simulation against last month's data; sees expected entry distribution.
4. Admin commits seed hash (system stores commitment publicly visible).
5. Admin publishes raffle.
6. During raffle period: every qualifying user action → entries created in raffle_entries.
7. On draw date/time:
   - Admin clicks "Run Draw."
   - System reveals seed; combines with public future value (Bitcoin block hash at time of draw).
   - Selects winner(s) deterministically.
   - Creates raffle_winner records.
8. System notifies winners; updates Past Winners gallery after fulfillment.

**Alternate Flows:**

- **A1** (Tiered prizes): Multiple winner selections per tier; same source pool, sequential exclusion.
- **A2** (Re-draw on no-claim): Same flow, excluding original winner.

**Post-conditions:** Winners selected verifiably; entry ledger archived; audit trail published.

**Related:** FR-801–FR-810.

---

### UC-305 · Promote deal via Meta/TikTok ads

**Primary Actor:** Marketing Admin
**Pre-conditions:** Meta Business and TikTok Business accounts connected via OAuth; deal is approved and live.
**Trigger:** Admin opens a deal and clicks "Promote."

**Main Flow:**

1. Wizard: choose platform (Meta / TikTok / both).
2. Choose objective: Traffic (deal views), Conversions (coupon sales), App Installs.
3. Set budget (daily or lifetime), schedule (start/end), geo (UAE city granularity).
4. Choose audience template ("Dubai foodies," "Spa-goers female 25–45," etc.) or custom.
5. Choose creative: defaults to deal hero image + auto-generated copy from title and price; admin can override.
6. Review estimated reach.
7. Click Launch.
8. System calls Meta Marketing API + TikTok Marketing API to create the campaign(s).
9. UTM tags auto-applied; deferred deep links generated.
10. System sends events to Meta Conversions API + TikTok Events API for ViewContent / AddToCart / InitiateCheckout / Purchase.
11. Admin sees campaign in dashboard with: spend, CPM, CTR, conversions (real Volu sales), CPA, ROAS.
12. Auto-pause rules trigger if CPA / ROAS thresholds breached.

**Post-conditions:** Campaign live; attributed sales tracked; ROAS visible.

**Related:** FR-1201–FR-1208.

---

### UC-306 · Handle a customer dispute / chargeback

**Primary Actor:** Support Admin → Finance Admin (escalation)
**Trigger:** Stripe webhook `charge.dispute.created` OR user files complaint via in-app chat.

**Main Flow:**

1. System receives Stripe dispute webhook; creates dispute case; freezes merchant balance equal to disputed amount.
2. Support admin assigned; reviews order, redemption logs, merchant response, user message history.
3. Admin uses Dispute Evidence Builder: one-click export of order details, delivery logs, redemption evidence, communication log.
4. Admin submits evidence to Stripe via dispute API.
5. Stripe rules; webhook updates case to `won` or `lost`.
6. If won: merchant balance unfrozen; user notified.
7. If lost: refund finalised; merchant balance debited (if previously credited); user notified.

**Alternate Flows:**

- **A1** (Quality complaint, not chargeback): Support handles directly; may issue refund without Stripe dispute.
- **A2** (Suspected fraud): Escalated; user account flagged for review; merchant flagged if pattern emerges.

**Post-conditions:** Case resolved; ledgers reconciled; learnings captured for merchant or fraud rules.

**Related:** FR-712, FR-1107.

---

## SYSTEM USE CASES (UC-4xx — automated/scheduled)

---

### UC-401 · Process coupon expiry (scheduled job)

**Trigger:** BullMQ job runs every 5 minutes.

**Main Flow:**

1. Job queries coupons with `status = active` AND `expires_at < NOW()`.
2. For each: status → `expired`; merchant ledger pending row reversed; breakage revenue recognised.
3. User notified that coupon expired.
4. Inventory not returned (it was consumed at purchase).
5. Audit logged.

---

### UC-402 · Send expiry reminder pushes (scheduled job)

**Trigger:** Hourly job.

**Main Flow:**

1. Find coupons in time windows: 7 days / 3 days / 24 hours / 3 hours before expiry.
2. For each: send push notification per user's preferences (skip if user opted out).
3. Notification content: "[Deal Title] expires in [X]. Don't miss it!"
4. Deduplicate so same coupon doesn't get same window twice.

---

### UC-403 · Daily Stripe reconciliation (scheduled job)

**Trigger:** Daily at 02:00 UAE time.

**Main Flow:**

1. Job pulls Stripe payouts and balance transactions for the previous day.
2. Compares to internal ledger entries.
3. Discrepancies > AED 1 flagged to finance via Slack alert.
4. Report stored for audit.

---

### UC-404 · Merchant document expiry alert (scheduled job)

**Trigger:** Daily.

**Main Flow:**

1. Find merchants with KYC documents expiring in next 30 days.
2. Send in-app + email warning to merchant.
3. On day of expiry: merchant cannot publish new deals (existing deals still redeemable until they expire).

---

## Cross-Cutting

### Use case to FR traceability

Every use case references the FRs it satisfies. PRs implementing a use case must reference the use case ID.

### Use case test coverage

Every use case must have at least one E2E test covering the main flow and at least one critical alternate flow. Exception flows are unit/integration tested.
