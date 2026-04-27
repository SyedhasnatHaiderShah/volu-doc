# 05 · Critical Flows

**Status:** 🟢 Approved

The five flows that make or break Volu. Each is documented step-by-step with critical decision points and design rules.

---

## Flow 1 — First-time onboarding

**Goal:** install → first deal viewed in ≤ 90 seconds.

```
1. App launch (cold start)
   └─ Splash: Volu mark on slate background, ~1s

2. Welcome screen
   ├─ Hero: "Unbeatable deals. Just for the UAE."
   ├─ Primary CTA: "Get started"
   └─ Secondary: "I already have an account"

3. Sign-up choice
   ├─ Continue with phone (primary, top)
   ├─ Continue with Apple
   ├─ Continue with Google
   └─ Continue with email

4. Phone entry
   ├─ Country code (UAE prefilled)
   ├─ Phone number input (autoformats)
   └─ "Continue" CTA — disabled until valid format

5. OTP screen
   ├─ 6-digit OTP input (autofocuses; OS autofill from SMS)
   ├─ Countdown to resend (60s)
   ├─ "Didn't receive? Resend" (after 60s)
   └─ Auto-submit when 6 digits entered

6. Profile basics (only if new user)
   ├─ Name
   ├─ Locale toggle (EN / AR)
   └─ "Allow notifications" prompt (deferred 1 screen for context)

7. Permission requests (one at a time)
   ├─ Notifications: "We'll let you know when great deals drop near you."
   └─ Location (skippable): "See deals near you, faster."

8. Home feed
   └─ Personalised based on locale, location (if granted), default categories
```

**Rules:**

- Every screen has a "Skip" or "Back" affordance except OTP.
- Permission requests pre-prompted with context — not the bare OS dialog.
- New user → first deal in 7 taps maximum.

---

## Flow 2 — Discovery → Purchase

**Goal:** P95 from "viewing a deal" to "coupon in wallet" ≤ 60 seconds (excluding 3DS).

```
1. Home feed
   ├─ Hero carousel (top)
   ├─ Today's Volu Drops (flash cards w/ countdown)
   ├─ Trending
   ├─ Near You
   └─ Categories

2. Deal card tap → Deal detail
   ├─ Hero gallery (swipe horizontally)
   ├─ Merchant name + Verified badge
   ├─ Price block: original (struck) | Volu price | % saved
   ├─ Inventory: "47 left" if low, hidden if comfortable
   ├─ Countdown: live (only flash deals)
   ├─ "What you get" expandable
   ├─ "Fine print" expandable
   ├─ "How to redeem" expandable
   ├─ "Branches & map" expandable (with Open in Maps)
   ├─ Reviews (top 3 + see all)
   ├─ Similar deals
   └─ Sticky bottom: quantity stepper + "Buy now" CTA

3. Tap "Buy now" → Cart sheet (modal)
   ├─ Deal line item with quantity
   ├─ Subtotal
   ├─ "Apply promo code" (collapsed)
   ├─ "Use wallet credit X AED" (toggle if available)
   ├─ VAT 5%
   ├─ Grand total — bold, large
   ├─ "All-in price guaranteed" notice
   └─ "Continue to payment" CTA

4. Payment screen
   ├─ Saved cards (if any) — top
   ├─ Apple Pay / Google Pay buttons (1-tap)
   ├─ Tabby / Tamara (if eligible by amount)
   ├─ Add new card
   ├─ Careem Pay
   └─ Pay button — pressed → Stripe SDK takes over

5. 3DS challenge (if required)
   └─ Stripe-hosted, returns to app

6. Confirmation
   ├─ ✓ Big check
   ├─ "Your Volu is ready"
   ├─ Coupon card preview
   ├─ "View in wallet" CTA
   └─ "Add to Apple Wallet" / "Add to Google Wallet" CTAs
```

**Rules:**

- All-in price (with VAT) is shown before the "Pay" button.
- 3DS is the only place users are sent off-screen.
- Confirmation screen is the destination — not a redirect to home.
- Coupon is in wallet within 2 seconds of payment success (push + in-app).

---

## Flow 3 — Coupon Redemption (cashier-side)

**Goal:** scan to result in ≤ 1.5 seconds online.

```
1. Cashier home (merchant app)
   └─ Huge "Scan Volu" button (75% of screen)

2. Camera opens (pre-warmed at cashier login)
   ├─ Live preview
   ├─ Translucent QR-frame overlay
   └─ Auto-detects QR — no shutter button

3. QR detected
   ├─ Haptic
   ├─ Server validates QR signature + coupon state + branch eligibility
   └─ If invalid: full-screen red banner with reason; "Try again" / "Manual code"

4. PIN entry screen
   ├─ Voice prompt: "Ask the customer for their PIN."
   ├─ Large numpad
   ├─ 4 visible digit slots (cashier sees what they type to confirm with customer)
   └─ "Redeem" CTA — large green, bottom

5. Server validates PIN
   ├─ Success: full-screen ✓ green + "Redeemed: [Deal] · [N uses left if multi]"
   └─ Failure: full-screen ✗ red + reason + retry option

6. Cashier taps "Done" → returns to home
```

**Rules:**

- Single-handed operation. All controls reachable with right thumb.
- Haptic + audio + visual on every state.
- Errors are bigger and more obvious than success states (cashiers need to NOT make a mistake here).
- "Manual code" is a one-tap escape if the camera fails.
- Offline mode: same UI; offline indicator banner; queued for sync.

---

## Flow 4 — Merchant Onboarding & First Deal

**Goal:** signup to live deal in 24 hours (KYC-bound).

```
1. Merchant app install + signup
   ├─ Business email + mobile
   └─ OTP

2. Business profile
   ├─ Legal name (from trade licence)
   ├─ Brand name
   ├─ Categories
   └─ Save

3. Add first branch
   ├─ Name (e.g., "Marina Branch")
   ├─ Address auto-suggest (Google Places)
   ├─ Pin on map (drag to confirm)
   ├─ Opening hours per day
   └─ Save

4. KYC documents
   ├─ Trade licence: upload PDF/photo
   ├─ Emirates ID front + back
   ├─ VAT certificate (if registered)
   ├─ IBAN bank letter
   └─ OCR pre-fills key fields; merchant verifies

5. Bank details
   ├─ IBAN (auto-validated format)
   ├─ Bank name (from IBAN lookup)
   └─ Holder name (must match legal entity)

6. Submit for review
   └─ Status: "Under review — typical SLA 24 business hours"

7. Admin reviews → approved
   └─ Merchant receives push + email: "You're live! Create your first deal."

8. Create first deal (guided wizard)
   ├─ Step 1: Type (Flash / Standard)
   ├─ Step 2: Title + description (EN + AR)
   ├─ Step 3: Photos (hero + gallery)
   ├─ Step 4: Pricing (original, Volu price, commission preview)
   ├─ Step 5: Inventory + dates
   ├─ Step 6: Branches + redemption rules
   ├─ Step 7: Fine print
   ├─ Step 8: Preview as user will see it
   └─ Step 9: Submit for approval

9. Admin reviews → approved → live
   └─ Merchant sees deal go live in their dashboard
```

**Rules:**

- Progress indicator across all KYC + first-deal steps (e.g., "3 of 9").
- Save-and-resume at every step.
- KYC OCR shows confidence; merchant confirms or corrects.
- Real-time preview during deal creation.

---

## Flow 5 — User Refund Self-Service

**Goal:** within 24h of purchase, user can refund without contacting support.

```
1. User opens "My Volus" → coupon detail
   └─ "Refund this coupon" link visible if eligible (≤ 24h, not redeemed)

2. Tap refund
   ├─ Bottom sheet: "Refund [Deal] for AED [X]?"
   ├─ "This will return your money to your card within 5–10 business days."
   ├─ "Your raffle entries from this purchase will also be revoked."
   └─ Confirm / Cancel

3. Confirm
   ├─ System creates Stripe refund (idempotent)
   ├─ Coupon status → refunded
   ├─ Inventory returned
   ├─ Raffle entries reversed
   ├─ Merchant ledger pending entry reversed
   └─ Spinner ~2s

4. Confirmation
   ├─ ✓ "Refund processed"
   ├─ "AED [X] will appear on your card within 5–10 days."
   └─ "Done" → returns to wallet
```

**Rules:**

- Self-service refund is one of three taps from any active coupon.
- Beyond 24h or after redemption, "Contact support" CTA opens live chat.

---

## Flow 6 — Raffle entry & winner reveal (bonus)

```
1. After every purchase
   └─ Toast: "You earned X raffle entries!"

2. Raffles tab
   ├─ Active raffle prize (photo, value, draw date)
   ├─ "My entries: X / Y total in pool"
   ├─ Breakdown: from purchases, referrals, free entries
   ├─ Time to draw: countdown
   └─ "How to earn more entries" link

3. Daily check-in
   └─ "Check in for 1 free entry" — once-a-day button

4. Draw day
   └─ Push: "Today's draw at 6 PM!"

5. Winner experience
   ├─ Push: "🎉 You won [prize]!"
   ├─ App opens to winner screen with confetti
   ├─ Claim CTA + 7-day countdown
   └─ Form for fulfillment

6. Non-winner experience
   ├─ Push: "Today's winner is announced — see them on Past Winners!"
   └─ Bonus entries credit for next month: "+5 entries for May"
```

---

## Cross-flow rules

- **Critical confirmations** (purchase, refund, payout): always show all-in amount + currency + clear "what happens next."
- **Long-running async** (refund, payout): show realistic ETA ("5–10 business days") rather than vague "processing."
- **Empty states**: every list has a defined empty state with helpful CTA.
- **Error recovery**: every error tells the user what to do next.
- **Offline tolerance**: every screen renders cached state; explicit retry.

---

## See also

- [Design Principles](./01-design-principles.md)
- [Design System](./02-design-system.md)
- [Use Cases](../02-requirements/03-use-cases.md)
- [User Stories](../02-requirements/04-user-stories.md)
