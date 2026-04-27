# 03 · Endpoints Catalogue

**Status:** 🟢 Approved

A representative catalogue of REST endpoints. The complete contract is the OpenAPI spec at `/api/openapi.json` — this doc is for human navigation.

Format: `METHOD /path` — short description — auth.

---

## Public — User app `/api/v1/...`

### Auth

- `POST /auth/phone/start` — Send OTP — public
- `POST /auth/phone/verify` — Verify OTP, issue tokens — public
- `POST /auth/email/register` — Register with email — public
- `POST /auth/email/login` — Login with email — public
- `POST /auth/social` — Apple/Google sign-in — public
- `POST /auth/refresh` — Rotate refresh token — refresh-token only
- `POST /auth/logout` — Revoke current session — bearer
- `POST /auth/logout-all` — Revoke all sessions — bearer
- `POST /auth/password/reset/request` — Send reset link — public
- `POST /auth/password/reset/confirm` — Apply new password — reset-token only

### Users

- `GET /users/me` — Current user profile — bearer
- `PATCH /users/me` — Update profile — bearer
- `DELETE /users/me` — Request account deletion — bearer
- `GET /users/me/devices` — List sessions/devices — bearer
- `DELETE /users/me/devices/:id` — Revoke device session — bearer
- `GET /users/me/notification-preferences` — Read prefs — bearer
- `PATCH /users/me/notification-preferences` — Update prefs — bearer
- `POST /users/me/data-export` — Request data export — bearer

### Discovery

- `GET /home/feed` — Home sections — bearer (or guest with limited content)
- `GET /categories` — Category tree — public
- `GET /deals` — List deals (filter, sort, paginate) — bearer
- `GET /deals/:id` — Deal detail — bearer
- `GET /deals/:id/similar` — Similar deals — bearer
- `GET /merchants/:id` — Merchant store profile — bearer
- `GET /merchants/:id/deals` — Deals for a merchant — bearer
- `GET /merchants/:id/reviews` — Merchant reviews — bearer
- `GET /search` — Search across deals & merchants — bearer

### Favourites

- `GET /favourites/deals` — List favourited deals — bearer
- `POST /favourites/deals/:dealId` — Favourite a deal — bearer
- `DELETE /favourites/deals/:dealId` — Unfavourite — bearer
- `POST /favourites/merchants/:merchantId` — Follow merchant — bearer
- `DELETE /favourites/merchants/:merchantId` — Unfollow — bearer

### Cart

- `GET /cart` — Read cart — bearer
- `POST /cart/items` — Add item — bearer
- `PATCH /cart/items/:id` — Update quantity — bearer
- `DELETE /cart/items/:id` — Remove item — bearer
- `POST /cart/promo` — Apply promo code — bearer
- `DELETE /cart/promo` — Remove promo — bearer

### Orders & Payments

- `POST /orders` — Create order from cart, returns Stripe client secret — bearer + idempotency
- `GET /orders` — List orders — bearer
- `GET /orders/:id` — Order detail — bearer
- `POST /orders/:id/refund` — Self-service refund (within 24h) — bearer + idempotency
- `POST /payments/tabby/session` — Create Tabby BNPL session — bearer + idempotency
- `POST /payments/tamara/session` — Create Tamara BNPL session — bearer + idempotency

### Coupons

- `GET /coupons` — List user's coupons — bearer
- `GET /coupons/:id` — Coupon detail (includes QR + PIN if biometric-verified) — bearer
- `POST /coupons/:id/wallet-pass` — Generate Apple/Google Wallet pass — bearer
- `POST /coupons/:id/gift` — Gift to another user — bearer + idempotency

### Raffles

- `GET /raffles/current` — Active raffle + my entries — bearer
- `GET /raffles/:id` — Raffle detail — bearer
- `GET /raffles/:id/winners` — Past winners — public
- `POST /raffles/:id/free-entry` — Daily check-in free entry — bearer

### Loyalty & Referrals

- `GET /loyalty/balance` — Points + wallet credit — bearer
- `GET /loyalty/transactions` — History — bearer
- `GET /referrals/code` — My referral code + share link — bearer
- `GET /referrals` — My referrals + statuses — bearer

### Reviews

- `POST /reviews` — Submit review — bearer + idempotency
- `GET /reviews/me` — My reviews — bearer

### Notifications (in-app)

- `GET /notifications/in-app` — Unread + recent — bearer
- `POST /notifications/in-app/:id/read` — Mark read — bearer

### Devices (push)

- `POST /devices/push-token` — Register/update push token — bearer

### Misc

- `GET /content/:slug` — CMS static content (T&Cs, etc.) — public
- `GET /app-version` — Min/recommended version — public
- `GET /feature-flags` — Client-evaluated flags — bearer

---

## Merchant app `/api/v1/merchant/...`

### Auth

- `POST /merchant/auth/phone/start` — Same as user but routes to merchant context — public
- `POST /merchant/auth/phone/verify` — Returns merchant-scoped JWT — public
- `GET /merchant/auth/memberships` — List merchants this user belongs to — bearer
- `POST /merchant/auth/switch-merchant` — Switch active merchant — bearer

### Onboarding

- `POST /merchant/signup` — Create merchant — bearer
- `PATCH /merchant/profile` — Update profile — owner
- `POST /merchant/branches` — Add branch — owner
- `PATCH /merchant/branches/:id` — Edit — owner/branch_manager
- `POST /merchant/kyc-documents` — Upload KYC doc — owner
- `GET /merchant/kyc-status` — KYC status — owner
- `POST /merchant/bank-accounts` — Add bank account — owner
- `POST /merchant/staff` — Invite staff — owner
- `DELETE /merchant/staff/:userId` — Revoke staff — owner

### Deals

- `GET /merchant/deals` — List my deals — owner/branch_manager
- `POST /merchant/deals` — Create draft — owner/branch_manager
- `GET /merchant/deals/:id` — Detail — owner/branch_manager
- `PATCH /merchant/deals/:id` — Edit (draft or after changes-requested) — owner/branch_manager
- `POST /merchant/deals/:id/submit` — Submit for approval — owner/branch_manager
- `POST /merchant/deals/:id/pause` — Pause live deal — owner/branch_manager
- `POST /merchant/deals/:id/resume` — Resume — owner/branch_manager
- `POST /merchant/deals/:id/extension-request` — Request extension — owner/branch_manager
- `GET /merchant/deals/:id/coupons` — Coupons issued — owner/branch_manager
- `GET /merchant/deals/:id/analytics` — Funnel + redemption — owner/branch_manager

### Redemption (cashier)

- `POST /merchant/redemptions/validate` — Validate QR token — cashier+
- `POST /merchant/redemptions/confirm` — Confirm with PIN — cashier+ + idempotency
- `POST /merchant/redemptions/manual-code` — Validate by short code — cashier+
- `POST /merchant/redemptions/offline-sync` — Sync queued offline redemptions — cashier+ + idempotency
- `GET /merchant/redemptions` — Recent redemptions list — cashier+

### Wallet, Payouts

- `GET /merchant/wallet` — Available, pending, lifetime — owner/accountant
- `GET /merchant/wallet/ledger` — Ledger entries — owner/accountant
- `POST /merchant/payouts` — Request payout — owner
- `GET /merchant/payouts` — Payout history — owner/accountant
- `GET /merchant/payouts/:id/statement.pdf` — Download PDF — owner/accountant

### Insights & Messages

- `GET /merchant/dashboard` — KPI overview — owner/branch_manager/accountant
- `GET /merchant/messages` — Admin messages — owner
- `GET /merchant/reviews` — Reviews of this merchant — owner/branch_manager
- `POST /merchant/reviews/:id/response` — Respond to review — owner/branch_manager

---

## Admin console `/api/v1/admin/...`

### Auth

- `POST /admin/auth/login` — Email + password (challenge-token only) — public
- `POST /admin/auth/2fa` — TOTP, issue tokens — challenge-token
- `POST /admin/auth/refresh` — Rotate — refresh
- `POST /admin/auth/logout` — — admin

### Dashboard

- `GET /admin/dashboard/kpis` — Top-strip KPIs — admin
- `GET /admin/dashboard/timeseries?metric=&from=&to=` — Charts — admin
- `GET /admin/dashboard/funnel` — Install→purchase→redeem funnel — admin
- `GET /admin/dashboard/cohorts` — Retention heatmap — admin
- `GET /admin/dashboard/anomalies` — Active anomaly alerts — admin

### Merchants

- `GET /admin/merchants` — List + filter — ops/super
- `GET /admin/merchants/:id` — Detail — ops/super
- `POST /admin/merchants/:id/approve-kyc` — Approve KYC — ops/super
- `POST /admin/merchants/:id/reject-kyc` — Reject — ops/super
- `POST /admin/merchants/:id/request-info` — Request more info — ops/super
- `POST /admin/merchants/:id/suspend` — Suspend — ops/super
- `POST /admin/merchants/:id/unsuspend` — Restore — ops/super
- `PATCH /admin/merchants/:id/commission-override` — Custom commission — finance/super
- `GET /admin/merchants/:id/notes` — Internal notes — ops/super
- `POST /admin/merchants/:id/notes` — Add note — ops/super
- `GET /admin/kyc-queue` — Pending KYC queue — ops/super

### Deals

- `GET /admin/deals` — List + filter — ops/super
- `GET /admin/deals/queue` — Submitted-for-approval queue — ops/super
- `POST /admin/deals/:id/approve` — Approve — ops/super
- `POST /admin/deals/:id/request-changes` — Request changes — ops/super
- `POST /admin/deals/:id/reject` — Reject — ops/super
- `POST /admin/deals/:id/feature` — Feature on home/category — marketing/super
- `POST /admin/deals/:id/pause` — Force pause — ops/super
- `POST /admin/deals/:id/promote` — Create Meta/TikTok ad campaign — marketing/super

### Coupons / Orders / Refunds

- `GET /admin/coupons?search=` — Search by short_code/user/merchant/order — ops/support/super
- `GET /admin/coupons/:id` — Coupon detail with full history — ops/support/super
- `POST /admin/coupons/issue` — Manually issue (compensation) — support/super
- `GET /admin/orders` — List orders — finance/support/super
- `POST /admin/orders/:id/refund` — Process refund — finance/support/super
- `GET /admin/refunds` — Refund queue + history — finance/super
- `GET /admin/disputes` — Stripe disputes — finance/super
- `POST /admin/disputes/:id/submit-evidence` — Submit dispute evidence — finance/super

### Payouts & Finance

- `GET /admin/payouts/queue` — Pending payout requests — finance/super
- `POST /admin/payouts/:id/approve` — Approve payout — finance/super
- `POST /admin/payouts/:id/reject` — Reject — finance/super
- `POST /admin/payouts/:id/mark-paid` — Mark manually paid — finance/super
- `GET /admin/finance/vat-report?month=` — VAT report — finance/super
- `GET /admin/finance/reconciliation?date=` — Stripe vs ledger diff — finance/super
- `GET /admin/finance/invoices` — Invoice list — finance/super

### Raffles

- `POST /admin/raffles` — Create raffle — marketing/super
- `GET /admin/raffles` — List — marketing/super
- `GET /admin/raffles/:id` — Detail — marketing/super
- `POST /admin/raffles/:id/simulate` — Simulate entry distribution — marketing/super
- `POST /admin/raffles/:id/publish` — Publish (commit seed hash) — marketing/super
- `POST /admin/raffles/:id/draw` — Run draw — marketing/super
- `POST /admin/raffles/:id/winners/:winnerId/fulfil` — Mark prize fulfilled — marketing/super

### Marketing

- `POST /admin/notifications/push` — Send push campaign — marketing/super
- `POST /admin/notifications/email` — Send email campaign — marketing/super
- `POST /admin/notifications/whatsapp` — Send WhatsApp campaign — marketing/super
- `POST /admin/banners` — Create in-app banner — marketing/super
- `POST /admin/promo-codes` — Create promo code — marketing/super

### Ads

- `POST /admin/ads/oauth/meta/start` — Begin Meta OAuth — marketing/super
- `POST /admin/ads/oauth/tiktok/start` — Begin TikTok OAuth — marketing/super
- `POST /admin/ads/campaigns` — Create campaign — marketing/super
- `GET /admin/ads/campaigns` — List — marketing/super
- `POST /admin/ads/campaigns/:id/pause` — Pause — marketing/super
- `POST /admin/ads/audience-templates` — Create audience — marketing/super
- `GET /admin/ads/dashboard` — Spend, ROAS, etc. — marketing/super

### Content / CMS

- `GET /admin/categories` — List — ops/super
- `POST /admin/categories` — Create — ops/super
- `PATCH /admin/categories/:id` — Edit — ops/super
- `GET /admin/tags` — List — ops/super
- `POST /admin/tags` — Create — ops/super
- `GET /admin/content/:slug` — Read versioned content — ops/super
- `POST /admin/content/:slug/version` — Publish new version — ops/super

### System

- `GET /admin/feature-flags` — List — super
- `PATCH /admin/feature-flags/:key` — Toggle — super
- `GET /admin/app-versions` — Min/recommended — super
- `PATCH /admin/app-versions/:platform` — Update — super
- `GET /admin/audit-log?actor=&action=&from=&to=` — Audit search — super (or scoped)
- `GET /admin/system/health` — System health — admin
- `GET /admin/system/queues` — BullMQ queue depths — super

---

## Webhooks (inbound)

These are public endpoints; verified by signature.

- `POST /api/v1/webhooks/stripe` — Stripe events
- `POST /api/v1/webhooks/tabby` — Tabby events
- `POST /api/v1/webhooks/tamara` — Tamara events
- `POST /api/v1/webhooks/whatsapp` — WhatsApp delivery + replies
- `POST /api/v1/webhooks/meta-conversions` — Meta Conversions API responses
- `POST /api/v1/webhooks/tiktok-events` — TikTok Events API responses

---

## Health

- `GET /health` — Liveness — public
- `GET /ready` — Readiness — public
- `GET /api/openapi.json` — OpenAPI spec — public

---

## See also

- [API Design Principles](./01-api-design-principles.md)
- [Authentication](./02-authentication.md)
- [Webhooks](./04-webhooks.md)
