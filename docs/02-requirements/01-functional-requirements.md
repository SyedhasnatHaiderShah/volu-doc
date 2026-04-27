# 01 · Functional Requirements

**Status:** 🟢 Approved

Every functional requirement Volu must satisfy. Each has a stable ID (`FR-XXX`) that must be traceable to code, tests, and QA sign-off. IDs are never reused.

**Legend:** `MUST` = v1; `SHOULD` = v1 nice-to-have; `COULD` = post-v1.

---

## FR-0xx · Authentication & Identity

### FR-001 · User registration (phone) — MUST

Users can register via UAE mobile number. OTP sent via Unifonic SMS. OTP is 6 digits, valid for 5 minutes, max 3 attempts. On success, a user record is created and the device is bound to the account.

### FR-002 · User registration (email) — MUST

Users can register via email + password. Email verification link valid for 24h. Password policy: min 10 chars, 1 upper, 1 lower, 1 digit. Rejected against top-100k breached passwords list.

### FR-003 · User registration (Apple ID) — MUST

"Sign in with Apple" supported. Email relay accepted. If first-time, user is prompted to add a phone for verification.

### FR-004 · User registration (Google) — MUST

"Sign in with Google" supported. If first-time, user is prompted to add a phone for verification.

### FR-005 · Phone verification gate — MUST

No coupon purchase is allowed until phone is verified, regardless of signup method.

### FR-006 · Biometric app unlock — MUST

Users can enable FaceID / fingerprint unlock in settings. Toggleable per device. Falls back to PIN (not the coupon PIN — a separate app-unlock PIN).

### FR-007 · Session management — MUST

Users can view all active sessions and force-logout any device. Server must revoke all refresh tokens for that device.

### FR-008 · Password reset — MUST

"Forgot password" flow sends a time-limited magic link or OTP. Link/OTP is single-use.

### FR-009 · Account deletion — MUST

Users can request account deletion from settings. Soft-delete immediately (anonymise PII, retain financial records for regulatory retention per UAE law). Hard-delete after 30 days for data not subject to retention.

### FR-010 · Data export — MUST

Users can request a data export. Generated within 24h, delivered via signed S3 URL valid for 72h.

### FR-011 · Merchant signup — MUST

Business users can self-signup with business email, mobile, and business name. Status starts as `pending_kyc`.

### FR-012 · Merchant KYC upload — MUST

Merchant uploads: (a) trade licence PDF, (b) Emirates ID of authorised signatory (front+back), (c) VAT certificate if registered, (d) IBAN bank letter. Each doc is OCR'd (IDfy/Sumsub) and stored encrypted at rest.

### FR-013 · Merchant KYC review — MUST

Admin reviews docs, sees OCR-extracted fields alongside originals, can approve / reject with reason / request info. State transitions are audit-logged. SLA: 24 business hours.

### FR-014 · Merchant document expiry tracking — MUST

Each uploaded doc has an expiry date. 30 days before expiry, merchant is warned via in-app + email. On expiry, the merchant cannot publish new deals; existing live deals remain redeemable.

### FR-015 · Admin login with 2FA — MUST

Admin login requires email + password + TOTP. Failed attempts are rate-limited. 5 failures → 15-minute lockout.

### FR-016 · Admin RBAC — MUST

Admin sub-roles: Super Admin, Operations, Finance, Marketing, Support. Each has a scoped permission set. Role changes are audit-logged. Super Admin role assignment requires existing Super Admin approval.

### FR-017 · Merchant sub-roles — MUST

Merchant Owner can invite staff as: Branch Manager, Cashier, Accountant. Each sub-role has scoped permissions. Owner can revoke access instantly.

### FR-018 · JWT + refresh rotation — MUST

Access tokens are JWTs with 15-minute TTL. Refresh tokens are opaque, rotated on every use, bound to device fingerprint. Reuse of a rotated refresh token triggers full session invalidation.

---

## FR-1xx · Merchant & Store Profile

### FR-101 · Merchant profile — MUST

Merchant has a brand name (EN + AR), logo, cover image, description (EN + AR), categories, social links, average rating.

### FR-102 · Branch management — MUST

A merchant can create, edit, and soft-delete branches. Each branch has: name, address, Google Maps pin, opening hours per day, phone number, photos.

### FR-103 · Branch manager assignment — MUST

Owner can assign a Branch Manager sub-role to a staff user, scoped to one or more branches.

### FR-104 · Public store page — MUST

Each merchant has a public profile in the user app showing: verified badge, categories, active deals, branches with map, rating, reviews.

### FR-105 · Verified badge logic — MUST

Badge is shown when: KYC approved AND no active suspensions AND documents not expired AND rating ≥ 3.5.

### FR-106 · Store metrics dashboard — MUST

Merchant sees: coupons sold (today/7d/30d), redemption rate, revenue earned, pending payout. Real-time (< 10s lag).

### FR-107 · Per-deal analytics — MUST

For each deal: views, cart adds, purchases, redemptions, refund count, average time-to-redeem, repeat buyers count.

---

## FR-2xx · Deal Creation & Lifecycle

### FR-201 · Create deal — MUST

Merchant creates a deal with: type (Flash / Standard), title (EN+AR), description (EN+AR), hero image, gallery (≤8), original price, Volu deal price, inventory total, max per user, max per day, valid-from, valid-until, eligible branches, redemption rules (days of week, hours, dine-in/takeaway/delivery), fine print (EN+AR), terms.

### FR-202 · Deal pricing validation — MUST

Volu deal price must be ≤ 75% of original price. Original price must be ≥ AED 10.

### FR-203 · Flash deal — MUST

Flash deals require: start timestamp, end timestamp, inventory cap. User UI shows countdown and remaining inventory.

### FR-204 · Standard deal — MUST

Standard deals require: valid-until date, optional inventory cap. No scarcity countdown shown to users.

### FR-205 · Deal submit for approval — MUST

Merchant clicks "Submit." State: `draft` → `submitted`. Admin notified. Merchant cannot edit while in `submitted`.

### FR-206 · Deal preview — MUST

Merchant can preview the deal exactly as a user will see it before submitting.

### FR-207 · Admin deal moderation — MUST

Admin sees: queue of submitted deals, preview pane, approve / request-changes / reject actions with reason templates. SLA: 24 business hours.

### FR-208 · Deal publishing — MUST

On approval, deal transitions: `approved` → `scheduled` (if future start) → `live` (at start time) → `expired` (at end time or inventory exhausted).

### FR-209 · Deal pause / resume — MUST

Merchant can pause a live deal (becomes invisible to new buyers; existing coupons still redeemable). Admin can also pause.

### FR-210 · Deal extension — MUST

Merchant can request an extension of `valid-until` before expiry. Admin approves or rejects. Deals cannot be extended beyond 90 days from original start.

### FR-211 · Deal versioning — MUST

Every edit to a deal after first approval is versioned. Previously-purchased coupons are bound to the version they were purchased under (terms can't change retroactively).

### FR-212 · Deal quality checks — MUST

On submit, automated checks: image min resolution (1200×800), banned-term detector in copy, duplicate-deal detector against the same merchant's history, price sanity vs merchant's historical data.

### FR-213 · Single-use coupons — MUST

Coupon can be redeemed exactly once. After redemption, status = `redeemed`, not redeemable again.

### FR-214 · Multi-use coupons — MUST

Coupon can specify `uses_total = N` (e.g., 5-visit pass). Each redemption decrements `uses_remaining`. When `uses_remaining = 0`, status = `used_up`. Expiry applies independently.

### FR-215 · Coupon transferability / gifting — SHOULD

User can gift a coupon to another user via phone or email. Transfer regenerates the QR token and PIN (old ones invalidated).

---

## FR-3xx · Discovery & Browse

### FR-301 · Home feed — MUST

Home shows: hero carousel, Today's Volu Drops (flash deals with countdown), Trending, Near You (geo-sorted), New This Week, Categories.

### FR-302 · Category tree — MUST

Top-level categories: Restaurants & Cafes, Spas & Salons & Wellness, Activities & Experiences, Retail & Shopping. Admin-configurable sub-categories. Badges: Halal-Certified, Family-Friendly, Pet-Friendly, Tourist-Friendly.

### FR-303 · Search — MUST

Full-text search across: deal title, merchant name, branch city, category, tags. Typo tolerance. Autocomplete. Recent and popular searches. Both EN and AR indexed.

### FR-304 · Filters — MUST

Price range, distance (radius from current location), rating, sub-category, dietary tags, valid-today, open-now.

### FR-305 · Map view — MUST

User can switch to map view showing deals as pins. Clustering at low zoom. Tap pin → deal detail. "Open in Google Maps" deep link.

### FR-306 · Sorting — MUST

Default: relevance (combination of popularity, proximity, freshness). User can change to: price low-to-high, price high-to-low, discount %, distance, newest.

### FR-307 · Deal detail page — MUST

Shows: hero gallery, merchant name + verified badge, rating, distance, price block (original, Volu, % saved, countdown), "What you get," "Fine print," "How to redeem," "Branches & map," "Reviews," "Similar deals," all-in-price notice, quantity selector, "Buy now" CTA.

### FR-308 · Favourites / saved — MUST

User can heart a deal or merchant. Favourites listed in profile. Notification when saved deal's inventory is running low.

### FR-309 · Share — MUST

User can share a deal via OS share sheet. Link includes referral attribution. Clicking the link either opens the app (if installed) or goes to a landing page with a smart install banner.

### FR-310 · Follow merchant — SHOULD

User can follow a merchant to see their new deals in a dedicated feed.

---

## FR-4xx · Cart, Checkout & Payment

### FR-401 · Add to cart — MUST

User can add a deal to cart with a specified quantity. Cart persists across sessions and devices.

### FR-402 · Quantity limits — MUST

Per-user cap (e.g., max 2 per user) and per-deal inventory are enforced. UI shows the lower of the two as the max quantity selector.

### FR-403 · Cart review — MUST

Cart shows line items, subtotal, applicable promo code, wallet credit applied, VAT (5%), grand total. Each line has an "all-in price" notice.

### FR-404 · Promo code application — MUST

User can apply one promo code per order. Validation: code exists, not expired, eligible for user, minimum order met. On failure, specific reason shown.

### FR-405 · Wallet credit — MUST

User can apply wallet credit (from refunds, referral bonuses, loyalty redemptions). Applied as discount before VAT calculation? **No** — applied after VAT. Wallet credit reduces what's charged to the card but doesn't reduce VAT liability.

### FR-406 · Stripe card payment — MUST

Visa / Mastercard / Amex accepted. 3DS required for transactions > AED 200. Card details tokenised via Stripe Elements; no PAN reaches Volu's servers.

### FR-407 · Apple Pay — MUST

Apple Pay button on cart and checkout. Uses Stripe's Apple Pay integration.

### FR-408 · Google Pay — MUST

Google Pay button on cart and checkout. Uses Stripe's Google Pay integration.

### FR-409 · Careem Pay — SHOULD

Careem Pay offered where supported.

### FR-410 · Tabby BNPL — MUST

Tabby offered for orders ≥ AED 150. Uses official Flutter SDK. On approval, order proceeds; on rejection, user returned to cart.

### FR-411 · Tamara BNPL — MUST

Tamara offered for orders ≥ AED 200. Uses official Flutter SDK.

### FR-412 · Payment idempotency — MUST

Every payment creation request includes an idempotency key. Retries with the same key never double-charge.

### FR-413 · Payment success handling — MUST

On Stripe webhook `payment_intent.succeeded`: order status → `paid`, coupons generated and delivered (push, email, in-app wallet), raffle entries created, merchant ledger debited `pending` row.

### FR-414 · Payment failure handling — MUST

On failure: clear error shown to user with retry CTA. Failed payment does not consume inventory. Common errors mapped to user-friendly messages (not Stripe codes).

### FR-415 · Guest checkout — COULD

Users can complete a purchase without full registration (phone verification only). Account is auto-created and linked if the same phone later signs up fully.

---

## FR-5xx · Coupon Wallet

### FR-501 · Coupon delivery — MUST

On successful purchase, each coupon is created with: UUID, short code (10-char base32), QR token (signed JWT-like), 4-digit random PIN. Delivered to: in-app wallet, push notification, email (PDF attachment), optional Apple/Google Wallet pass.

### FR-502 · Coupon wallet tabs — MUST

"My Volus" tabs: Active, Used, Expired, Refunded. Active sorted by expiry (soonest first).

### FR-503 · Coupon card display — MUST

Each coupon card shows: merchant logo, deal title, expiry countdown, large QR code, PIN (revealed only after biometric unlock in-app), branch list with "Open in Maps" CTA.

### FR-504 · Multi-use coupon display — MUST

Multi-use coupons show "X of Y uses remaining" and a timeline of past redemptions with branch and date.

### FR-505 · Apple Wallet pass — MUST

"Add to Apple Wallet" button generates a .pkpass file. Pass shows title, expiry, QR. Auto-updates on redemption (push-to-pass).

### FR-506 · Google Wallet pass — MUST

Equivalent for Android. "Add to Google Wallet" button.

### FR-507 · Coupon gift — SHOULD

User selects a coupon, taps "Gift." Enters recipient phone or email. Old QR/PIN invalidated; new ones generated for the recipient. If recipient is not on Volu, they receive an SMS/email with a deep link to claim.

### FR-508 · Coupon refund (pre-redemption, self-service) — MUST

Within 24h of purchase and before any redemption, user can request full refund in-app. Refund processed via Stripe. Raffle entries for this purchase are revoked. Inventory returned.

### FR-509 · Coupon refund (post-24h / post-redemption partial) — MUST

Requires support ticket. Support admin decides. If refund granted after merchant was already credited, merchant balance is debited.

---

## FR-6xx · Redemption

### FR-601 · Scan QR — MUST

Merchant app has a prominent "Scan Volu" button on cashier home. Opens camera, detects QR code.

### FR-602 · QR validation — MUST

Server validates QR token: signature valid, not expired, coupon status = active, branch eligible for this coupon, within valid days/hours per deal rules, uses_remaining > 0.

### FR-603 · PIN entry — MUST

After QR scan, cashier enters the 4-digit PIN the user speaks. Server validates PIN. 3 failed attempts lock the coupon for 15 minutes.

### FR-604 · Redemption result — MUST

On success: ✅ green banner with merchant + deal + uses_remaining. Cashier confirms. Coupon state updated. Merchant balance `pending` → `available` (after hold period).

On failure: specific error shown:

- ❌ "Coupon not found" (invalid QR)
- ⚠️ "Already redeemed" (single-use already used)
- ⚠️ "No uses left" (multi-use exhausted)
- ⏰ "Expired" (past valid-until)
- 🚫 "Not valid at this branch" (branch mismatch)
- 🚫 "Not valid at this time" (outside valid days/hours)
- ⛔ "Wrong PIN"

### FR-605 · Manual code entry fallback — MUST

If camera fails, cashier can type the short code instead. Same validation flow.

### FR-606 · Offline queue — MUST

If merchant app is offline, redemption is cryptographically signed with a device key and queued. On reconnect, synced to server. Server enforces single-source-of-truth, so a double-spend (same coupon redeemed on two offline devices) is rejected for the second.

### FR-607 · Redemption history (merchant) — MUST

Merchant can view all redemptions for any deal or date range, with cashier, branch, timestamp.

### FR-608 · Redemption history (user) — MUST

User's coupon card shows redemption timeline (especially relevant for multi-use).

### FR-609 · Fraud alerts — MUST

Server alerts admin if: >30 redemptions in 5 minutes from one device, same coupon scan attempt from >2 devices within 10 minutes, cashier redemption pattern deviates >3σ from merchant's baseline.

---

## FR-7xx · Merchant Payouts & Finance

### FR-701 · Merchant wallet view — MUST

Merchant sees: Available balance, Pending balance (redeemed but in hold period), Lifetime earnings, Next payout-eligible date.

### FR-702 · Ledger transparency — MUST

Every ledger entry is visible to the merchant: date, type (credit / debit / adjustment), amount, reference (coupon id, payout id, etc.).

### FR-703 · Payout request — MUST

Merchant requests payout when available ≥ threshold (default AED 200). Request includes amount (defaults to full available). Status: `requested` → `approved` → `in_transit` → `paid` / `failed`.

### FR-704 · Finance approval — MUST

Finance Admin sees the payout queue sorted by oldest. Views merchant details, KYC status, recent dispute history. Approves or rejects with reason.

### FR-705 · Bank transfer — MUST

Approved payouts are executed via Stripe Connect Express or direct bank transfer to the verified IBAN. Bank reference recorded.

### FR-706 · Payout statement PDF — MUST

Every payout generates a PDF statement with TRN, line-item redemptions contributing to the payout, fees, VAT, net amount, bank reference. Downloadable by merchant.

### FR-707 · Commission invoice (Volu → Merchant) — MUST

Monthly invoice to each merchant for Volu's commission, VAT-compliant with TRN.

### FR-708 · Platform-issued tax invoice — MUST

For end-users: VAT-compliant tax invoice PDF per order, with sequential numbering, TRN, QR e-invoice (FTA Phase 2-ready).

### FR-709 · VAT reports — MUST

Finance Admin can pull monthly VAT reports: output VAT collected, input VAT paid to merchants, net payable. Export in FTA-compatible CSV format.

### FR-710 · Stripe reconciliation — MUST

Daily scheduled job compares Stripe payouts to Volu's internal ledger. Discrepancies > AED 1 flagged to finance.

### FR-711 · Refund reversal — MUST

A user refund reverses: order total, Volu commission (negative), merchant ledger credit if already applied, raffle entries. Idempotent.

### FR-712 · Chargeback handling — MUST

On Stripe webhook `charge.dispute.created`: create dispute case, freeze merchant balance equal to disputed amount, notify admin, surface dispute evidence builder.

### FR-713 · Commission override per-merchant — MUST

Admin can set a custom commission rate for a specific merchant (e.g., 15% for anchor merchants for 90 days).

### FR-714 · Commission override per-category — MUST

Admin can set category-specific commission rates (e.g., 25% for spas, 18% for restaurants).

---

## FR-8xx · Raffles

### FR-801 · Raffle creation — MUST

Marketing Admin creates a raffle with: name (EN+AR), description, period start/end, prizes (title, value, photos, tier), entry rules (rule engine), draw date/time, T&Cs URL, promotional banners.

### FR-802 · Entry rules engine — MUST

Admin configures rules like: "1 entry per AED 10 spent on Volus," "5 bonus entries for first purchase this month," "2× multiplier for Activities category this weekend," "1 entry per successful referral." Cap: max N entries per user per raffle.

### FR-803 · Rule simulation — MUST

Before publishing, admin can simulate entries against historical data to see expected distribution.

### FR-804 · Entry ledger — MUST

Every raffle entry is a separate row with: raffle_id, user_id, source (purchase / referral / bonus / free_entry), weight, timestamp, source_ref (coupon_id, etc.).

### FR-805 · Free-entry alternative — MUST

For UAE prize-promotion compliance, every raffle must offer a no-purchase-necessary entry path (e.g., daily in-app check-in). Free entries are ledgered separately.

### FR-806 · User-facing raffle view — MUST

User sees: current raffle's prize, total entries, my entries (broken down by source), time to draw, rules, past winners gallery.

### FR-807 · Verifiable random draw — MUST

On draw date, system:

1. Before draw: committed a hash of seed to a public place (e.g., committed earlier on-chain or in a public log).
2. At draw: reveals seed.
3. Combines seed with a public future value (e.g., Bitcoin block hash at time of draw).
4. Uses combined value to deterministically pick winners.
   Audit log of the seed, hash commitment, block hash used, and entry ledger is published.

### FR-808 · Winner notification — MUST

Winners notified via push + SMS + email + in-app banner. Claim window (default 7 days). If not claimed, prize is re-drawn.

### FR-809 · Prize fulfillment tracking — MUST

Admin marks prize as "claimed," "fulfilled," with proof photo. Public "Past Winners" gallery populated.

### FR-810 · Consolation bonus — SHOULD

Non-winners receive bonus entries for the next raffle (configurable).

---

## FR-9xx · Loyalty, Referrals & Promotions

### FR-901 · Loyalty points earn — MUST

User earns X points per AED spent (configurable globally). Points issued on successful purchase (not pre-redemption).

### FR-902 · Loyalty points redeem — MUST

User can redeem points at checkout: Y points = AED 1 discount, or trade points for extra raffle entries. Configurable.

### FR-903 · Loyalty tiers — COULD

Bronze / Silver / Gold based on 12-month spend. Perks: higher earn rate, early access to flash deals, bonus raffle entries.

### FR-904 · Referral program — MUST

Every user has a unique referral code. Shared via deep link. Referee gets AED X off first purchase. Referrer earns AED Y wallet credit on referee's first successful redemption (not purchase).

### FR-905 · Referral abuse protection — MUST

Self-referral blocked (device fingerprint + phone + payment method). Referral credit held until referee's coupon is redeemed.

### FR-906 · Promo code engine — MUST

Admin creates codes with rules: value (flat AED or %), scope (global / category / specific merchant / specific deal), minimum order, max uses total, max uses per user, valid-from / valid-until, first-purchase-only toggle.

---

## FR-10xx · Notifications & Messaging

### FR-1001 · Push notifications — MUST

FCM (Android) + APNs (iOS). User opts in on first launch. Preference granularity: deal drops, flash deals, raffle updates, coupon expiry reminders, marketing.

### FR-1002 · Coupon expiry reminders — MUST

Scheduled pushes at 7 days, 3 days, 24 hours, and 3 hours before expiry (if not yet redeemed). Configurable per user.

### FR-1003 · Geo-fenced push — SHOULD

User near a partner merchant with active deal → contextual push. Opt-in.

### FR-1004 · Email templates — MUST

Transactional: OTP, purchase confirmation, coupon delivery, refund confirmation, payout confirmation, KYC status change. Marketing: weekly drops digest (opt-in).

### FR-1005 · SMS templates — MUST

OTP, payment confirmation, coupon expiry warning (last 24h), raffle winner notification.

### FR-1006 · WhatsApp notifications — MUST

Via WhatsApp Business API. Opt-in. Use cases: OTP fallback, coupon expiry reminders, raffle winner, support responses.

### FR-1007 · In-app banners — MUST

Admin can schedule home-screen banners: image, title, CTA, audience segment, start/end time.

### FR-1008 · In-app live chat — MUST

Intercom integration. User can start a chat from Help menu, from any coupon card, or from order history. SLA: 2-hour business-hours response target.

---

## FR-11xx · Admin Console (platform management)

### FR-1101 · Global dashboard — MUST

KPI strip (GMV, revenue, active deals, pending approvals, pending payouts, redemption rate, refund rate), time-series charts, funnel, cohort retention heatmap.

### FR-1102 · Merchant list & filters — MUST

Search by name / status / category / KYC state / performance band. Bulk actions: export, invite, message.

### FR-1103 · Merchant detail — MUST

Tabs: Profile, Branches, KYC Docs, Deals, Financials, Ratings, Support History, Internal Notes.

### FR-1104 · Coupon search — MUST

Search any coupon by code, user phone, merchant, order id, transaction id.

### FR-1105 · Manual coupon issuance — MUST

Admin can issue a comp coupon to a user (apology credit, marketing). Audit-logged.

### FR-1106 · Refund processing — MUST

Partial and full refunds. Reverses commission, merchant credit (if applied), raffle entries. Triggers Stripe refund.

### FR-1107 · Dispute evidence builder — MUST

For Stripe chargebacks, one-click export of: order details, delivery logs, redemption evidence, merchant response.

### FR-1108 · Feature flags — MUST

Admin can toggle feature flags globally or per-user-cohort. Flag changes propagate within 60s.

### FR-1109 · App version control — MUST

Admin sets minimum required version per platform. On launch, app checks; if below minimum, shows force-update screen.

---

## FR-12xx · Ads Integration (Phase 4)

### FR-1201 · Meta OAuth connect — MUST

Admin connects a Meta Business Manager account. Permissions requested: ads management, Conversions API, Catalog.

### FR-1202 · TikTok OAuth connect — MUST

Admin connects a TikTok Business account. Permissions: ads management, Events API.

### FR-1203 · "Promote this deal" flow — MUST

From any deal in admin, one-click "Promote" opens a wizard: choose platform (Meta / TikTok / both), objective (traffic / conversions / app installs), budget, schedule, audience template, creative (defaulted to deal hero + auto-copy).

### FR-1204 · Audience templates — MUST

Pre-built: "Dubai foodies," "Spa-goers female 25–45," "Tourists in UAE," "Active Volu users," "Lapsed users." Admin can create custom templates.

### FR-1205 · Conversions API — MUST

Volu sends events to Meta Conversions API and TikTok Events API: `ViewContent`, `AddToCart`, `InitiateCheckout`, `Purchase`. Enables attribution and lookalike audience building.

### FR-1206 · Campaign dashboard — MUST

For each active campaign: spend, impressions, CPM, clicks, CTR, conversions (tied to actual Volu coupon sales for this deal), CPA, ROAS, delta over baseline.

### FR-1207 · Auto-pause rules — MUST

Admin configures auto-pause: "pause if CPA > AED X for 48 hours" or "pause if ROAS < 2.0 after spending AED Y."

### FR-1208 · UTM auto-tagging — MUST

All ad destination URLs are auto-tagged with source, medium, campaign, content. Attribution persists through deferred deep linking to the deal.

---

## FR-13xx · Content & CMS

### FR-1301 · Category management — MUST

Admin creates/edits categories and sub-categories (EN + AR names, slug, sort order, icon).

### FR-1302 · Badge management — MUST

Admin defines badges (Halal-Certified, Family-Friendly, Pet-Friendly, Tourist-Friendly, etc.). Merchants can apply badges to deals; admin reviews.

### FR-1303 · Static pages — MUST

Help, T&Cs, Privacy, Merchant Agreement, Refund Policy — admin-editable with version history. Versions are immutable once published.

### FR-1304 · Translations workbench — MUST

EN / AR side-by-side editor for all CMS content and UI strings.

---

## FR-14xx · Reviews & Ratings

### FR-1401 · Post-redemption review prompt — MUST

24h after redemption, user is prompted (push + in-app) to rate 1–5 stars and optionally write a review.

### FR-1402 · Review publishing — MUST

Reviews go through lightweight moderation (profanity filter + first-review manual check). Published or auto-flagged.

### FR-1403 · Merchant response — MUST

Merchant can respond publicly to any review.

### FR-1404 · Review on merchant page — MUST

Public store page shows: average rating, total review count, sorted list (newest first, filterable by rating).

### FR-1405 · Rating impact — MUST

Merchant rating < 3.5 average over 30d → loses Verified badge, admin flagged. Rating < 3.0 → admin-initiated suspension review.

---

## Traceability

Every FR here must be traceable to:

- A unit test or integration test in the codebase.
- A test case in QA's test plan.
- A section in the user-facing help documentation (if consumer-facing).

PRs that implement an FR must reference its ID in the title (e.g., `FR-213: Single-use coupon redemption`).
