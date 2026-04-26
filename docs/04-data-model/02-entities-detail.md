# 02 · Entities — Detailed Schema

**Status:** 🟢 Approved

Every table, every column, every index, every constraint. This is the source of truth for the schema. Changes go through migrations, not edits to this doc — but this doc is updated alongside.

**Conventions:**
- Money in `INTEGER fils` (1 AED = 100 fils).
- IDs are `UUID` (UUID v7).
- All timestamps are `TIMESTAMPTZ` stored in UTC; rendered in `Asia/Dubai` at the edge.
- Soft delete via `deleted_at`; queries default-exclude soft-deleted.

---

## IDENTITY MODULE

### `identity__users`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | UUID v7 |
| `phone` | TEXT | UNIQUE | E.164 format `+9715XXXXXXXX` |
| `phone_verified_at` | TIMESTAMPTZ | NULL | |
| `email` | TEXT | UNIQUE NULLABLE | Lowercased |
| `email_verified_at` | TIMESTAMPTZ | NULL | |
| `password_hash` | TEXT | NULL | Argon2id; null if social-only |
| `name` | TEXT | NOT NULL | |
| `locale` | TEXT | NOT NULL DEFAULT 'en' | `en` or `ar` |
| `birthdate` | DATE | NULL | For age-restricted deals + raffle eligibility |
| `gender` | TEXT | NULL | `male`/`female`/`other`/`undisclosed` |
| `emirate` | TEXT | NULL | `dubai`/`abu_dhabi`/...; helps personalisation |
| `is_tourist` | BOOLEAN | NOT NULL DEFAULT FALSE | Inferred or self-declared |
| `referred_by_user_id` | UUID | FK | Self-FK |
| `social_apple_sub` | TEXT | UNIQUE NULLABLE | Apple ID subject |
| `social_google_sub` | TEXT | UNIQUE NULLABLE | Google subject |
| `metadata` | JSONB | NOT NULL DEFAULT '{}' | Tags, flags |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() | Trigger-updated |
| `deleted_at` | TIMESTAMPTZ | NULL | Soft delete |

**Indexes:**
- `idx_users_phone` UNIQUE on `(phone)` WHERE `deleted_at IS NULL`
- `idx_users_email` UNIQUE on `(email)` WHERE `deleted_at IS NULL`
- `idx_users_referred_by` on `(referred_by_user_id)`
- `idx_users_created_at` on `(created_at)`

---

### `identity__user_devices`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | → `identity__users.id` |
| `device_fingerprint` | TEXT | NOT NULL | Hashed device identifier |
| `platform` | TEXT | NOT NULL | `ios`/`android`/`web` |
| `os_version` | TEXT | NULL | |
| `app_version` | TEXT | NULL | |
| `push_token` | TEXT | NULL | FCM/APNs |
| `last_seen_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() | |
| `revoked_at` | TIMESTAMPTZ | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() | |

**Indexes:**
- `idx_user_devices_user` on `(user_id)`
- `idx_user_devices_fingerprint` on `(device_fingerprint)`
- UNIQUE `(user_id, device_fingerprint)`

---

### `identity__sessions`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | |
| `device_id` | UUID | FK NOT NULL | |
| `refresh_token_hash` | TEXT | NOT NULL | Hashed |
| `issued_at` | TIMESTAMPTZ | NOT NULL | |
| `expires_at` | TIMESTAMPTZ | NOT NULL | |
| `last_used_at` | TIMESTAMPTZ | NOT NULL | |
| `revoked_at` | TIMESTAMPTZ | NULL | |
| `revocation_reason` | TEXT | NULL | |

**Indexes:**
- `idx_sessions_user` on `(user_id)`
- `idx_sessions_token_hash` UNIQUE on `(refresh_token_hash)`
- `idx_sessions_active` on `(user_id, expires_at)` WHERE `revoked_at IS NULL`

---

### `identity__merchants`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `legal_name` | TEXT | NOT NULL | From trade licence |
| `brand_name` | TEXT | NOT NULL | Public-facing |
| `trn` | TEXT | UNIQUE NULLABLE | UAE VAT TRN |
| `trade_licence_number` | TEXT | NOT NULL | |
| `trade_licence_expiry` | DATE | NOT NULL | |
| `business_email` | TEXT | NOT NULL | |
| `business_phone` | TEXT | NOT NULL | |
| `description_en` | TEXT | NULL | |
| `description_ar` | TEXT | NULL | |
| `logo_image_key` | TEXT | NULL | S3 key |
| `cover_image_key` | TEXT | NULL | S3 key |
| `social_links` | JSONB | NOT NULL DEFAULT '{}' | `{ "instagram": "...", ... }` |
| `status` | enum | NOT NULL | `pending_kyc`/`under_review`/`approved`/`rejected`/`suspended`/`offboarded` |
| `kyc_status` | enum | NOT NULL | `pending`/`under_review`/`approved`/`rejected`/`info_requested` |
| `commission_override_bps` | INTEGER | NULL | Per-merchant override (basis points) |
| `payout_hold_days_override` | INTEGER | NULL | Per-merchant override |
| `category_id` | UUID | FK NOT NULL | Primary category |
| `signed_agreement_at` | TIMESTAMPTZ | NULL | |
| `agreement_version` | TEXT | NULL | |
| `internal_notes` | TEXT | NULL | Admin-only |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | NULL | |

**Indexes:**
- `idx_merchants_status` on `(status)`
- `idx_merchants_brand_name` on `(brand_name)` (text-search ready)
- `idx_merchants_category` on `(category_id)`
- `idx_merchants_kyc_status` on `(kyc_status)` WHERE `kyc_status != 'approved'`

**Constraints:**
- CHECK `commission_override_bps IS NULL OR (commission_override_bps >= 0 AND commission_override_bps <= 5000)`

---

### `identity__merchant_branches`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `name` | TEXT | NOT NULL | |
| `address_line` | TEXT | NOT NULL | |
| `city` | TEXT | NOT NULL | |
| `emirate` | TEXT | NOT NULL | |
| `country` | TEXT | NOT NULL DEFAULT 'AE' | |
| `lat` | DECIMAL(9,6) | NOT NULL | |
| `lng` | DECIMAL(9,6) | NOT NULL | |
| `phone` | TEXT | NULL | |
| `opening_hours` | JSONB | NOT NULL | Per-day hours |
| `photo_keys` | TEXT[] | NOT NULL DEFAULT '{}' | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_branches_merchant` on `(merchant_id)`
- `idx_branches_geo` GIST on `point(lng, lat)` (for nearest-neighbour queries)

---

### `identity__merchant_users`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `user_id` | UUID | FK NOT NULL | → `identity__users.id` |
| `role` | enum | NOT NULL | `owner`/`branch_manager`/`cashier`/`accountant` |
| `branches_scope` | UUID[] | NOT NULL DEFAULT '{}' | Empty = all |
| `invited_at` | TIMESTAMPTZ | NULL | |
| `accepted_at` | TIMESTAMPTZ | NULL | |
| `revoked_at` | TIMESTAMPTZ | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- UNIQUE `(merchant_id, user_id)` WHERE `revoked_at IS NULL`
- `idx_merchant_users_user` on `(user_id)`

---

### `identity__kyc_documents`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `type` | enum | NOT NULL | `trade_licence`/`emirates_id_front`/`emirates_id_back`/`vat_certificate`/`iban_letter` |
| `s3_key` | TEXT | NOT NULL | Encrypted bucket |
| `mime_type` | TEXT | NOT NULL | |
| `size_bytes` | INTEGER | NOT NULL | |
| `expires_at` | DATE | NULL | For trade licence + Emirates ID |
| `ocr_data` | JSONB | NULL | Auto-extracted fields |
| `status` | enum | NOT NULL | `uploaded`/`under_review`/`approved`/`rejected`/`expired` |
| `reviewed_by` | UUID | FK NULL | → `identity__admin_users.id` |
| `reviewed_at` | TIMESTAMPTZ | NULL | |
| `rejection_reason` | TEXT | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_kyc_merchant` on `(merchant_id)`
- `idx_kyc_status` on `(status)`
- `idx_kyc_expires` on `(expires_at)` WHERE `expires_at IS NOT NULL`

---

### `identity__admin_users`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `email` | TEXT | UNIQUE NOT NULL | |
| `name` | TEXT | NOT NULL | |
| `password_hash` | TEXT | NOT NULL | Argon2id |
| `role` | enum | NOT NULL | `super`/`operations`/`finance`/`marketing`/`support` |
| `totp_secret_encrypted` | TEXT | NULL | KMS-encrypted |
| `totp_enabled_at` | TIMESTAMPTZ | NULL | |
| `last_login_at` | TIMESTAMPTZ | NULL | |
| `failed_attempts` | INTEGER | NOT NULL DEFAULT 0 | |
| `locked_until` | TIMESTAMPTZ | NULL | |
| `status` | enum | NOT NULL | `active`/`disabled` |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

---

## CATALOG MODULE

### `catalog__categories`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `parent_id` | UUID | FK NULL | Self-FK |
| `slug` | TEXT | UNIQUE NOT NULL | URL-safe |
| `name_en` | TEXT | NOT NULL | |
| `name_ar` | TEXT | NOT NULL | |
| `description_en` | TEXT | NULL | |
| `description_ar` | TEXT | NULL | |
| `icon` | TEXT | NULL | Lucide icon name or S3 key |
| `sort_order` | INTEGER | NOT NULL DEFAULT 0 | |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_categories_parent` on `(parent_id)`

---

### `catalog__deals`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `category_id` | UUID | FK NOT NULL | |
| `type` | enum | NOT NULL | `flash`/`standard` |
| `status` | enum | NOT NULL | `draft`/`submitted`/`changes_requested`/`approved`/`scheduled`/`live`/`paused`/`expired`/`rejected` |
| `title_en` | TEXT | NOT NULL | |
| `title_ar` | TEXT | NOT NULL | |
| `description_en` | TEXT | NOT NULL | |
| `description_ar` | TEXT | NOT NULL | |
| `included_en` | TEXT | NOT NULL | What's included |
| `included_ar` | TEXT | NOT NULL | |
| `excluded_en` | TEXT | NULL | What's excluded |
| `excluded_ar` | TEXT | NULL | |
| `fine_print_en` | TEXT | NOT NULL | |
| `fine_print_ar` | TEXT | NOT NULL | |
| `original_price_fils` | INTEGER | NOT NULL | |
| `deal_price_fils` | INTEGER | NOT NULL | |
| `commission_bps` | INTEGER | NOT NULL | Snapshot at deal creation |
| `usage_type` | enum | NOT NULL | `single_use`/`multi_use` |
| `uses_total` | INTEGER | NOT NULL DEFAULT 1 | For multi-use |
| `inventory_total` | INTEGER | NOT NULL | |
| `inventory_left` | INTEGER | NOT NULL | Atomic decrement on purchase |
| `max_per_user` | INTEGER | NOT NULL DEFAULT 5 | |
| `max_per_day_per_branch` | INTEGER | NULL | Optional cap |
| `valid_from` | TIMESTAMPTZ | NOT NULL | |
| `valid_until` | TIMESTAMPTZ | NOT NULL | |
| `redemption_rules` | JSONB | NOT NULL | Days, hours, dine-in/takeaway/delivery flags |
| `hero_image_key` | TEXT | NOT NULL | |
| `gallery_keys` | TEXT[] | NOT NULL DEFAULT '{}' | |
| `tags` | TEXT[] | NOT NULL DEFAULT '{}' | Denormalised for fast filter |
| `submitted_at` | TIMESTAMPTZ | NULL | |
| `approved_by` | UUID | FK NULL | |
| `approved_at` | TIMESTAMPTZ | NULL | |
| `rejected_reason` | TEXT | NULL | |
| `view_count` | INTEGER | NOT NULL DEFAULT 0 | |
| `purchase_count` | INTEGER | NOT NULL DEFAULT 0 | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_deals_status_valid` on `(status, valid_from, valid_until)` WHERE `status IN ('scheduled','live')`
- `idx_deals_merchant` on `(merchant_id)`
- `idx_deals_category` on `(category_id)`
- `idx_deals_inventory` on `(inventory_left)` WHERE `status = 'live'`
- GIN index on `tags`

**Constraints:**
- CHECK `deal_price_fils <= (original_price_fils * 0.75)`
- CHECK `inventory_left <= inventory_total`
- CHECK `valid_until > valid_from`

---

### `catalog__deal_versions`

Snapshot of every approved version of a deal. Coupons reference the version they were issued under so terms can't change retroactively.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `deal_id` | UUID | FK NOT NULL | |
| `version` | INTEGER | NOT NULL | Monotonic per deal |
| `snapshot` | JSONB | NOT NULL | Full deal payload at this version |
| `created_by` | UUID | FK NOT NULL | Admin user |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- UNIQUE `(deal_id, version)`

---

### `catalog__deal_branches`

Many-to-many between deals and branches.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `deal_id` | UUID | FK NOT NULL | |
| `branch_id` | UUID | FK NOT NULL | |

PRIMARY KEY `(deal_id, branch_id)`.

---

### `catalog__tags`

Curated badges (Halal-Certified, Family-Friendly, etc.) — distinct from free-text tags on deals.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `slug` | TEXT | UNIQUE NOT NULL | |
| `name_en` | TEXT | NOT NULL | |
| `name_ar` | TEXT | NOT NULL | |
| `icon` | TEXT | NULL | |
| `color` | TEXT | NULL | |

---

## COMMERCE MODULE

### `commerce__carts`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | |

UNIQUE `(user_id)`.

---

### `commerce__cart_items`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `cart_id` | UUID | FK NOT NULL | |
| `deal_id` | UUID | FK NOT NULL | |
| `quantity` | INTEGER | NOT NULL | |
| `added_at` | TIMESTAMPTZ | NOT NULL | |

UNIQUE `(cart_id, deal_id)`.

---

### `commerce__orders`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | |
| `status` | enum | NOT NULL | `pending`/`paid`/`failed`/`refunded`/`partially_refunded`/`cancelled` |
| `subtotal_fils` | INTEGER | NOT NULL | |
| `promo_discount_fils` | INTEGER | NOT NULL DEFAULT 0 | |
| `wallet_credit_fils` | INTEGER | NOT NULL DEFAULT 0 | |
| `vat_fils` | INTEGER | NOT NULL | 5% on (subtotal - promo) |
| `total_fils` | INTEGER | NOT NULL | What was charged |
| `currency` | TEXT | NOT NULL DEFAULT 'AED' | |
| `promo_code_id` | UUID | FK NULL | |
| `idempotency_key` | TEXT | NOT NULL | Client-provided |
| `referrer_user_id` | UUID | FK NULL | If purchase came via referral |
| `device_id` | UUID | FK NULL | Originating device |
| `metadata` | JSONB | NOT NULL DEFAULT '{}' | UTM, ad campaign, etc. |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `paid_at` | TIMESTAMPTZ | NULL | |

**Indexes:**
- UNIQUE `(user_id, idempotency_key)`
- `idx_orders_user_created` on `(user_id, created_at DESC)`
- `idx_orders_status_paid` on `(status, paid_at)`

---

### `commerce__order_items`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `order_id` | UUID | FK NOT NULL | |
| `deal_id` | UUID | FK NOT NULL | |
| `deal_version_id` | UUID | FK NOT NULL | Snapshot the user purchased |
| `merchant_id` | UUID | FK NOT NULL | Denormalised for query speed |
| `quantity` | INTEGER | NOT NULL | |
| `unit_price_fils` | INTEGER | NOT NULL | |
| `commission_bps_snapshot` | INTEGER | NOT NULL | Snapshot at purchase |
| `payout_per_unit_fils` | INTEGER | NOT NULL | Computed: unit_price - commission |

---

### `commerce__promo_codes`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `code` | TEXT | UNIQUE NOT NULL | |
| `scope` | enum | NOT NULL | `global`/`category`/`merchant`/`deal` |
| `scope_ref_id` | UUID | NULL | category_id/merchant_id/deal_id |
| `discount_type` | enum | NOT NULL | `flat`/`percent` |
| `value_fils` | INTEGER | NULL | For flat |
| `value_bps` | INTEGER | NULL | For percent (basis points) |
| `min_order_fils` | INTEGER | NULL | |
| `max_discount_fils` | INTEGER | NULL | Cap on percent |
| `max_uses_total` | INTEGER | NULL | |
| `used_count` | INTEGER | NOT NULL DEFAULT 0 | |
| `max_uses_per_user` | INTEGER | NULL | |
| `first_purchase_only` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `valid_from` | TIMESTAMPTZ | NOT NULL | |
| `valid_until` | TIMESTAMPTZ | NOT NULL | |
| `created_by` | UUID | FK NOT NULL | Admin |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

---

## PAYMENTS MODULE

### `payments__payments`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `order_id` | UUID | FK NOT NULL | |
| `provider` | enum | NOT NULL | `stripe`/`tabby`/`tamara`/`careem_pay` |
| `provider_intent_id` | TEXT | NOT NULL | E.g., Stripe PaymentIntent id |
| `status` | enum | NOT NULL | `created`/`requires_action`/`succeeded`/`failed`/`refunded` |
| `amount_fils` | INTEGER | NOT NULL | |
| `currency` | TEXT | NOT NULL | |
| `method` | TEXT | NULL | `card`/`apple_pay`/`google_pay`/`bnpl_tabby`/`bnpl_tamara`/`careem` |
| `card_brand` | TEXT | NULL | `visa`/`mastercard`/... (from Stripe; no PAN stored) |
| `card_last4` | TEXT | NULL | |
| `failure_code` | TEXT | NULL | |
| `failure_message` | TEXT | NULL | |
| `metadata` | JSONB | NOT NULL DEFAULT '{}' | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `completed_at` | TIMESTAMPTZ | NULL | |

**Indexes:**
- UNIQUE `(provider, provider_intent_id)`
- `idx_payments_order` on `(order_id)`

---

### `payments__inbound_webhooks`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `provider` | enum | NOT NULL | |
| `event_type` | TEXT | NOT NULL | |
| `provider_event_id` | TEXT | NOT NULL | For dedup |
| `signature_valid` | BOOLEAN | NOT NULL | |
| `payload` | JSONB | NOT NULL | |
| `received_at` | TIMESTAMPTZ | NOT NULL | |
| `processed_at` | TIMESTAMPTZ | NULL | |
| `process_attempts` | INTEGER | NOT NULL DEFAULT 0 | |
| `last_error` | TEXT | NULL | |

UNIQUE `(provider, provider_event_id)`.

---

## COUPONS MODULE

### `coupons__coupons`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | UUID v7 |
| `order_id` | UUID | FK NOT NULL | |
| `order_item_id` | UUID | FK NOT NULL | |
| `deal_id` | UUID | FK NOT NULL | |
| `deal_version_id` | UUID | FK NOT NULL | Snapshot |
| `merchant_id` | UUID | FK NOT NULL | Denormalised |
| `user_id` | UUID | FK NOT NULL | Owner; updated on gift |
| `original_user_id` | UUID | FK NOT NULL | Buyer; immutable |
| `short_code` | TEXT | UNIQUE NOT NULL | 10-char base32 |
| `qr_token_signature` | TEXT | NOT NULL | Stored for replay protection |
| `pin_hash` | TEXT | NOT NULL | Argon2id |
| `usage_type` | enum | NOT NULL | `single_use`/`multi_use` |
| `uses_total` | INTEGER | NOT NULL | |
| `uses_remaining` | INTEGER | NOT NULL | |
| `status` | enum | NOT NULL | `active`/`redeemed`/`used_up`/`expired`/`refunded`/`gifted_away` |
| `valid_from` | TIMESTAMPTZ | NOT NULL | |
| `expires_at` | TIMESTAMPTZ | NOT NULL | |
| `last_redeemed_at` | TIMESTAMPTZ | NULL | |
| `locked_until` | TIMESTAMPTZ | NULL | After PIN failures |
| `pin_attempts` | INTEGER | NOT NULL DEFAULT 0 | Reset on success |
| `payout_amount_fils` | INTEGER | NOT NULL | Per-redemption credit to merchant |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- UNIQUE `(short_code)`
- `idx_coupons_user_status` on `(user_id, status)`
- `idx_coupons_merchant_status` on `(merchant_id, status)`
- `idx_coupons_deal` on `(deal_id)`
- `idx_coupons_expires_active` on `(expires_at)` WHERE `status = 'active'`

**Constraints:**
- CHECK `uses_remaining >= 0 AND uses_remaining <= uses_total`

---

### `coupons__coupon_pin_attempts`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `coupon_id` | UUID | FK NOT NULL | |
| `result` | enum | NOT NULL | `success`/`wrong_pin`/`locked` |
| `source_ip` | INET | NULL | |
| `device_fingerprint` | TEXT | NULL | |
| `cashier_user_id` | UUID | FK NULL | |
| `attempted_at` | TIMESTAMPTZ | NOT NULL | |

`idx_pin_attempts_coupon_attempted` on `(coupon_id, attempted_at DESC)`.

---

## REDEMPTION MODULE

### `redemption__redemptions`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `coupon_id` | UUID | FK NOT NULL | |
| `branch_id` | UUID | FK NOT NULL | |
| `cashier_user_id` | UUID | FK NOT NULL | |
| `merchant_id` | UUID | FK NOT NULL | Denormalised |
| `device_fingerprint` | TEXT | NOT NULL | |
| `source` | enum | NOT NULL | `online`/`offline_synced` |
| `redeemed_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_redemptions_coupon` on `(coupon_id)`
- `idx_redemptions_merchant_redeemed` on `(merchant_id, redeemed_at DESC)`
- `idx_redemptions_branch_redeemed` on `(branch_id, redeemed_at DESC)`

---

### `redemption__queue_offline`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | Client-generated |
| `coupon_id` | UUID | NOT NULL | |
| `branch_id` | UUID | FK NOT NULL | |
| `cashier_user_id` | UUID | FK NOT NULL | |
| `device_fingerprint` | TEXT | NOT NULL | |
| `signature` | TEXT | NOT NULL | Device-signed |
| `attempted_at` | TIMESTAMPTZ | NOT NULL | When the cashier did it offline |
| `synced_at` | TIMESTAMPTZ | NULL | |
| `sync_result` | enum | NULL | `accepted`/`rejected_double_spend`/`rejected_invalid` |

---

## WALLET MODULE

### `wallet__merchant_ledger`

The merchant's running ledger. Every coupon purchase, redemption, refund, payout writes a row.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `entry_type` | enum | NOT NULL | `pending_credit`/`pending_release`/`available_credit`/`available_debit`/`paid_out`/`adjustment`/`reversal` |
| `amount_fils` | INTEGER | NOT NULL | Signed; positive credit, negative debit |
| `reference_kind` | enum | NOT NULL | `coupon_purchase`/`coupon_redemption`/`coupon_refund`/`payout`/`chargeback`/`adjustment` |
| `reference_id` | UUID | NOT NULL | Polymorphic FK |
| `available_after` | TIMESTAMPTZ | NULL | When pending becomes available |
| `description` | TEXT | NULL | |
| `created_by` | UUID | NULL | Admin if manual adjustment |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_ledger_merchant_created` on `(merchant_id, created_at DESC)`
- `idx_ledger_reference` on `(reference_kind, reference_id)`
- `idx_ledger_pending_release` on `(available_after)` WHERE `entry_type = 'pending_release'`

---

### `wallet__breakage_ledger`

Records platform breakage revenue per expired coupon.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `coupon_id` | UUID | FK NOT NULL | |
| `merchant_id` | UUID | FK NOT NULL | |
| `amount_fils` | INTEGER | NOT NULL | Full coupon payout amount that became breakage |
| `recognized_at` | TIMESTAMPTZ | NOT NULL | |

---

## PAYOUTS MODULE

### `payouts__bank_accounts`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `iban` | TEXT | NOT NULL | UAE format |
| `bank_name` | TEXT | NOT NULL | |
| `holder_name` | TEXT | NOT NULL | Must match merchant legal name |
| `is_primary` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `verified_at` | TIMESTAMPTZ | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

UNIQUE `(merchant_id, iban)`.

---

### `payouts__payouts`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `merchant_id` | UUID | FK NOT NULL | |
| `bank_account_id` | UUID | FK NOT NULL | |
| `amount_fils` | INTEGER | NOT NULL | Net payout |
| `provider_fees_fils` | INTEGER | NOT NULL DEFAULT 0 | |
| `status` | enum | NOT NULL | `requested`/`approved`/`in_transit`/`paid`/`failed`/`cancelled` |
| `provider` | enum | NOT NULL | `stripe_connect`/`manual_bank` |
| `provider_payout_id` | TEXT | NULL | |
| `bank_reference` | TEXT | NULL | |
| `requested_by` | UUID | FK NOT NULL | Merchant user |
| `approved_by` | UUID | FK NULL | Admin |
| `requested_at` | TIMESTAMPTZ | NOT NULL | |
| `approved_at` | TIMESTAMPTZ | NULL | |
| `paid_at` | TIMESTAMPTZ | NULL | |
| `failed_at` | TIMESTAMPTZ | NULL | |
| `failure_reason` | TEXT | NULL | |
| `statement_pdf_key` | TEXT | NULL | |

---

### `payouts__payout_line_items`

Links payouts to ledger entries that fund them.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `payout_id` | UUID | FK NOT NULL | |
| `ledger_entry_id` | UUID | FK NOT NULL | |
| `amount_fils` | INTEGER | NOT NULL | |

UNIQUE `(payout_id, ledger_entry_id)`.

---

## REFUNDS MODULE

### `refunds__refunds`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `order_id` | UUID | FK NOT NULL | |
| `coupon_id` | UUID | FK NULL | If coupon-specific |
| `amount_fils` | INTEGER | NOT NULL | |
| `reason` | enum | NOT NULL | `user_self_service`/`merchant_failure`/`misrepresentation`/`goodwill`/`duplicate`/`fraud`/`other` |
| `notes` | TEXT | NULL | |
| `initiated_by` | UUID | FK NOT NULL | User or admin |
| `initiated_by_role` | TEXT | NOT NULL | |
| `provider` | enum | NOT NULL | |
| `provider_refund_id` | TEXT | NULL | |
| `status` | enum | NOT NULL | `pending`/`completed`/`failed` |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `completed_at` | TIMESTAMPTZ | NULL | |

---

## RAFFLES MODULE

### `raffles__raffles`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `name_en` | TEXT | NOT NULL | |
| `name_ar` | TEXT | NOT NULL | |
| `description_en` | TEXT | NULL | |
| `description_ar` | TEXT | NULL | |
| `status` | enum | NOT NULL | `draft`/`scheduled`/`active`/`drawing`/`completed`/`cancelled` |
| `period_start` | TIMESTAMPTZ | NOT NULL | |
| `period_end` | TIMESTAMPTZ | NOT NULL | |
| `draw_at` | TIMESTAMPTZ | NOT NULL | |
| `seed_commitment_hash` | TEXT | NOT NULL | SHA-256 of seed, published at scheduling |
| `commitment_published_at` | TIMESTAMPTZ | NOT NULL | |
| `seed_revealed` | TEXT | NULL | After draw |
| `seed_revealed_at` | TIMESTAMPTZ | NULL | |
| `public_block_reference` | TEXT | NULL | E.g., Bitcoin block hash used |
| `tcs_url` | TEXT | NOT NULL | Versioned URL |
| `total_prize_value_fils` | INTEGER | NOT NULL | |
| `created_by` | UUID | FK NOT NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

---

### `raffles__raffle_prizes`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `raffle_id` | UUID | FK NOT NULL | |
| `tier` | INTEGER | NOT NULL | 1 = grand, 2 = second, ... |
| `title_en` | TEXT | NOT NULL | |
| `title_ar` | TEXT | NOT NULL | |
| `description_en` | TEXT | NULL | |
| `description_ar` | TEXT | NULL | |
| `value_fils` | INTEGER | NOT NULL | |
| `quantity` | INTEGER | NOT NULL DEFAULT 1 | Number of winners for this tier |
| `photo_keys` | TEXT[] | NOT NULL DEFAULT '{}' | |

---

### `raffles__raffle_rules`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `raffle_id` | UUID | FK NOT NULL | |
| `rule_dsl` | JSONB | NOT NULL | Structured rule |
| `sort_order` | INTEGER | NOT NULL DEFAULT 0 | |

Example `rule_dsl`:
```json
{
  "trigger": "coupon_purchased",
  "entries_per": { "type": "per_aed_spent", "amount_fils": 1000, "entries": 1 },
  "category_filter": null,
  "max_per_user_total": 200
}
```

---

### `raffles__raffle_entries`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `raffle_id` | UUID | FK NOT NULL | |
| `user_id` | UUID | FK NOT NULL | |
| `source` | enum | NOT NULL | `purchase`/`referral`/`bonus`/`free_entry` |
| `source_ref` | UUID | NULL | order_id, referral_id, etc. |
| `weight` | INTEGER | NOT NULL DEFAULT 1 | Multiplier |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_entries_raffle` on `(raffle_id)`
- `idx_entries_user_raffle` on `(user_id, raffle_id)`

---

### `raffles__raffle_winners`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `raffle_id` | UUID | FK NOT NULL | |
| `prize_id` | UUID | FK NOT NULL | |
| `user_id` | UUID | FK NOT NULL | |
| `winning_entry_id` | UUID | FK NOT NULL | |
| `drawn_at` | TIMESTAMPTZ | NOT NULL | |
| `notified_at` | TIMESTAMPTZ | NULL | |
| `claimed_at` | TIMESTAMPTZ | NULL | |
| `fulfilled_at` | TIMESTAMPTZ | NULL | |
| `proof_photo_key` | TEXT | NULL | |
| `notes` | TEXT | NULL | |

---

## LOYALTY MODULE

### `loyalty__loyalty_balances`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `user_id` | UUID | PK | One row per user |
| `points` | INTEGER | NOT NULL DEFAULT 0 | |
| `wallet_credit_fils` | INTEGER | NOT NULL DEFAULT 0 | |
| `tier` | enum | NOT NULL DEFAULT 'bronze' | `bronze`/`silver`/`gold` |
| `lifetime_spent_fils` | INTEGER | NOT NULL DEFAULT 0 | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | |

---

### `loyalty__loyalty_transactions`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | |
| `kind` | enum | NOT NULL | `earned`/`redeemed`/`expired`/`adjustment` |
| `points_delta` | INTEGER | NOT NULL | Signed |
| `wallet_credit_delta_fils` | INTEGER | NOT NULL DEFAULT 0 | Signed |
| `reference_kind` | enum | NULL | |
| `reference_id` | UUID | NULL | |
| `description` | TEXT | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

---

### `loyalty__referrals`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `referrer_user_id` | UUID | FK NOT NULL | |
| `referee_user_id` | UUID | FK NOT NULL | |
| `referrer_credit_fils` | INTEGER | NOT NULL | |
| `referee_credit_fils` | INTEGER | NOT NULL | |
| `referee_first_redemption_at` | TIMESTAMPTZ | NULL | When credit awarded |
| `status` | enum | NOT NULL | `pending`/`credited`/`abandoned`/`fraudulent` |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

UNIQUE `(referee_user_id)`.

---

## REVIEWS MODULE

### `reviews__reviews`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | |
| `merchant_id` | UUID | FK NOT NULL | |
| `coupon_id` | UUID | FK NOT NULL | Tied to a redemption |
| `rating` | SMALLINT | NOT NULL | 1–5 |
| `body_en` | TEXT | NULL | |
| `language` | TEXT | NOT NULL DEFAULT 'en' | Detected |
| `status` | enum | NOT NULL | `pending`/`published`/`hidden`/`flagged` |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

**Indexes:**
- `idx_reviews_merchant_status` on `(merchant_id, status)`
- UNIQUE `(coupon_id)` (one review per redemption)

---

### `reviews__review_responses`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `review_id` | UUID | FK NOT NULL | |
| `merchant_user_id` | UUID | FK NOT NULL | |
| `body` | TEXT | NOT NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

---

## NOTIFICATIONS MODULE

### `notifications__notifications`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `user_id` | UUID | FK NOT NULL | |
| `channel` | enum | NOT NULL | `push`/`email`/`sms`/`whatsapp`/`in_app` |
| `template_key` | TEXT | NOT NULL | |
| `payload` | JSONB | NOT NULL | Variables for template |
| `status` | enum | NOT NULL | `queued`/`sent`/`delivered`/`failed` |
| `provider_message_id` | TEXT | NULL | |
| `failure_reason` | TEXT | NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |
| `sent_at` | TIMESTAMPTZ | NULL | |

Range-partitioned by `created_at` weekly; pruned after 90 days.

---

### `notifications__notification_preferences`

One row per user.

| Column | Type | Constraints |
|---|---|---|
| `user_id` | UUID | PK |
| `push_drops` | BOOLEAN | DEFAULT TRUE |
| `push_flash` | BOOLEAN | DEFAULT TRUE |
| `push_raffle` | BOOLEAN | DEFAULT TRUE |
| `push_expiry` | BOOLEAN | DEFAULT TRUE |
| `push_marketing` | BOOLEAN | DEFAULT FALSE |
| `email_marketing` | BOOLEAN | DEFAULT FALSE |
| `sms_marketing` | BOOLEAN | DEFAULT FALSE |
| `whatsapp_marketing` | BOOLEAN | DEFAULT FALSE |

---

### `notifications__push_devices`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `user_id` | UUID | FK NOT NULL |
| `platform` | TEXT | NOT NULL `ios`/`android` |
| `token` | TEXT | NOT NULL |
| `app_version` | TEXT | NOT NULL |
| `registered_at` | TIMESTAMPTZ | NOT NULL |
| `last_seen_at` | TIMESTAMPTZ | NOT NULL |

UNIQUE `(token)`.

---

## ADS MODULE

### `ads__ad_campaigns`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | UUID | PK | |
| `deal_id` | UUID | FK NOT NULL | |
| `channel` | enum | NOT NULL | `meta`/`tiktok` |
| `provider_campaign_id` | TEXT | NOT NULL | |
| `objective` | enum | NOT NULL | `traffic`/`conversions`/`installs` |
| `audience_template_id` | UUID | FK NULL | |
| `budget_fils_daily` | INTEGER | NULL | |
| `budget_fils_lifetime` | INTEGER | NULL | |
| `start_at` | TIMESTAMPTZ | NOT NULL | |
| `end_at` | TIMESTAMPTZ | NULL | |
| `status` | enum | NOT NULL | `pending`/`active`/`paused`/`completed`/`failed` |
| `auto_pause_rules` | JSONB | NULL | E.g., `{ "pause_if_cpa_gt_fils": 5000, "after_spend_fils": 50000 }` |
| `created_by` | UUID | FK NOT NULL | |
| `created_at` | TIMESTAMPTZ | NOT NULL | |

---

### `ads__ad_campaign_metrics`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `campaign_id` | UUID | FK NOT NULL |
| `metric_date` | DATE | NOT NULL |
| `spend_fils` | INTEGER | NOT NULL |
| `impressions` | INTEGER | NOT NULL |
| `clicks` | INTEGER | NOT NULL |
| `installs` | INTEGER | NOT NULL DEFAULT 0 |
| `purchases` | INTEGER | NOT NULL DEFAULT 0 |
| `attributed_revenue_fils` | INTEGER | NOT NULL DEFAULT 0 |

UNIQUE `(campaign_id, metric_date)`.

---

### `ads__audience_templates`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `name` | TEXT | NOT NULL |
| `description` | TEXT | NULL |
| `meta_spec` | JSONB | NULL |
| `tiktok_spec` | JSONB | NULL |
| `created_at` | TIMESTAMPTZ | NOT NULL |

---

## INVOICING MODULE

### `invoicing__invoices`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `invoice_number` | TEXT | UNIQUE NOT NULL |
| `type` | enum | NOT NULL `consumer_tax`/`merchant_commission`/`merchant_payout_statement` |
| `party_user_id` | UUID | FK NULL |
| `party_merchant_id` | UUID | FK NULL |
| `subtotal_fils` | INTEGER | NOT NULL |
| `vat_fils` | INTEGER | NOT NULL |
| `total_fils` | INTEGER | NOT NULL |
| `currency` | TEXT | NOT NULL |
| `pdf_key` | TEXT | NOT NULL |
| `xml_key` | TEXT | NULL |
| `issued_at` | TIMESTAMPTZ | NOT NULL |
| `metadata` | JSONB | NOT NULL DEFAULT '{}' |

---

### `invoicing__invoice_line_items`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `invoice_id` | UUID | FK NOT NULL |
| `description` | TEXT | NOT NULL |
| `quantity` | INTEGER | NOT NULL |
| `unit_price_fils` | INTEGER | NOT NULL |
| `vat_rate_bps` | INTEGER | NOT NULL DEFAULT 500 |
| `subtotal_fils` | INTEGER | NOT NULL |
| `vat_fils` | INTEGER | NOT NULL |
| `total_fils` | INTEGER | NOT NULL |
| `reference_kind` | TEXT | NULL |
| `reference_id` | UUID | NULL |

---

## FAVOURITES & FOLLOWS

### `favourites__favourites`

| Column | Type | Constraints |
|---|---|---|
| `user_id` | UUID | FK NOT NULL |
| `deal_id` | UUID | FK NOT NULL |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() |

PRIMARY KEY `(user_id, deal_id)`.

---

### `favourites__merchant_follows`

| Column | Type | Constraints |
|---|---|---|
| `user_id` | UUID | FK NOT NULL |
| `merchant_id` | UUID | FK NOT NULL |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() |

PRIMARY KEY `(user_id, merchant_id)`.

---

## AUDIT MODULE

### `audit__audit_logs`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `actor_user_id` | UUID | NULL |
| `actor_role` | TEXT | NOT NULL |
| `action` | TEXT | NOT NULL |
| `target_type` | TEXT | NOT NULL |
| `target_id` | UUID | NULL |
| `before_state` | JSONB | NULL |
| `after_state` | JSONB | NULL |
| `source_ip` | INET | NULL |
| `user_agent` | TEXT | NULL |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT NOW() |

Range-partitioned by `created_at` monthly. Append-only — no updates allowed.

**Indexes:**
- `idx_audit_actor_created` on `(actor_user_id, created_at DESC)`
- `idx_audit_target` on `(target_type, target_id)`

---

## SYSTEM TABLES

### `system__outbox_events`

(Already documented in [Event-Driven Design](../03-architecture/06-event-driven-design.md))

---

### `system__feature_flags`

| Column | Type | Constraints |
|---|---|---|
| `id` | UUID | PK |
| `key` | TEXT | UNIQUE NOT NULL |
| `enabled` | BOOLEAN | NOT NULL DEFAULT FALSE |
| `rules` | JSONB | NOT NULL DEFAULT '{}' |
| `description` | TEXT | NULL |
| `updated_by` | UUID | FK NULL |
| `updated_at` | TIMESTAMPTZ | NOT NULL |

---

### `system__app_versions`

| Column | Type | Constraints |
|---|---|---|
| `platform` | TEXT | PK `ios`/`android` |
| `minimum_version` | TEXT | NOT NULL |
| `recommended_version` | TEXT | NOT NULL |
| `force_update_message_en` | TEXT | NULL |
| `force_update_message_ar` | TEXT | NULL |
| `updated_at` | TIMESTAMPTZ | NOT NULL |

---

## Schema invariants enforced at DB level

- Money columns are integer, never float.
- Status columns are enums, never raw text.
- All FKs `ON DELETE` is `RESTRICT` unless explicitly noted (we do not cascade-delete; we soft-delete and let retention jobs hard-delete).
- All timestamps are `TIMESTAMPTZ` (with timezone), never `TIMESTAMP` without.
- Unique business identifiers (phone, email, short_code, TRN) have UNIQUE indexes scoped to non-deleted rows.

---

## See also

- [ERD overview](./01-erd-overview.md)
- [Migrations strategy](./03-migrations-strategy.md)
