# 01 · ERD Overview

**Status:** 🟢 Approved

The Volu data model. This document gives the high-level entity relationships; the detailed schema with every column lives in [02-entities-detail.md](./02-entities-detail.md).

---

## Naming convention

- Tables use `snake_case`. Module ownership prefix: `<module>__<table>`. E.g. `identity__users`, `catalog__deals`, `commerce__orders`.
- Primary keys are `id UUID` (UUID v7 for sortability + uniqueness).
- Foreign keys are `<entity>_id` referencing the related table's `id`.
- Timestamps: `created_at`, `updated_at`, soft delete `deleted_at` (nullable).
- Soft delete via `deleted_at`; hard delete only by retention job.
- Money columns: `INTEGER` storing fils (1 AED = 100 fils). No floating-point money. Ever.
- Enums: PostgreSQL native enums for stability; lookup tables only when admin needs to manage them.
- Indexes: every FK indexed; every `WHERE`-filter column indexed.

---

## Top-level entity map

```mermaid
erDiagram
    USERS ||--o{ ORDERS : "places"
    USERS ||--o{ COUPONS : "owns"
    USERS ||--o{ REVIEWS : "writes"
    USERS ||--o{ RAFFLE_ENTRIES : "earns"
    USERS ||--o{ FAVOURITES : "saves"
    USERS ||--o{ USER_DEVICES : "uses"
    USERS ||--o{ NOTIFICATIONS : "receives"

    MERCHANTS ||--o{ MERCHANT_BRANCHES : "operates"
    MERCHANTS ||--o{ MERCHANT_USERS : "employs"
    MERCHANTS ||--o{ DEALS : "publishes"
    MERCHANTS ||--o{ KYC_DOCUMENTS : "submits"
    MERCHANTS ||--o{ MERCHANT_LEDGER : "accrues"
    MERCHANTS ||--o{ PAYOUTS : "receives"
    MERCHANTS ||--o{ INVOICES : "issued"

    DEALS ||--o{ DEAL_VERSIONS : "snapshots"
    DEALS }o--|| MERCHANTS : "belongs to"
    DEALS }o--o{ MERCHANT_BRANCHES : "valid at"
    DEALS }o--|| CATEGORIES : "categorized by"
    DEALS ||--o{ ORDER_ITEMS : "appears in"
    DEALS ||--o{ AD_CAMPAIGNS : "promoted by"

    ORDERS ||--|{ ORDER_ITEMS : "contains"
    ORDERS ||--o{ COUPONS : "issues"
    ORDERS ||--o{ PAYMENTS : "settles"
    ORDERS }o--|| USERS : "placed by"
    ORDERS ||--o{ REFUNDS : "may have"

    COUPONS ||--o{ REDEMPTIONS : "redeemed via"
    COUPONS }o--|| ORDERS : "from order"
    COUPONS }o--|| DEALS : "for deal"
    COUPONS }o--|| USERS : "owned by"

    REDEMPTIONS }o--|| MERCHANT_BRANCHES : "at branch"
    REDEMPTIONS }o--|| MERCHANT_USERS : "by cashier"

    PAYOUTS }o--|| MERCHANTS : "to merchant"
    PAYOUTS ||--o{ MERCHANT_LEDGER : "settles entries"

    RAFFLES ||--o{ RAFFLE_ENTRIES : "collects"
    RAFFLES ||--o{ RAFFLE_PRIZES : "offers"
    RAFFLES ||--o{ RAFFLE_WINNERS : "selects"
    RAFFLE_WINNERS }o--|| USERS : "won by"
    RAFFLE_WINNERS }o--|| RAFFLE_PRIZES : "wins"

    REVIEWS }o--|| USERS : "written by"
    REVIEWS }o--|| MERCHANTS : "about merchant"
    REVIEWS }o--|| COUPONS : "tied to redemption"

    AD_CAMPAIGNS }o--|| DEALS : "promotes"

    PROMO_CODES ||--o{ ORDERS : "applied to"

    AUDIT_LOGS }o--|| USERS : "actor"
```

---

## Module ownership

Each table belongs to exactly one module:

| Module | Tables |
|---|---|
| **Identity** | `users`, `user_devices`, `merchants`, `merchant_branches`, `merchant_users`, `kyc_documents`, `admin_users`, `sessions` |
| **Catalog** | `categories`, `deals`, `deal_versions`, `deal_branches`, `deal_tags`, `tags` |
| **Commerce** | `orders`, `order_items`, `carts`, `cart_items`, `promo_codes`, `promo_code_usages` |
| **Payments** | `payments`, `payment_methods`, `payment_intents`, `inbound_webhooks` |
| **Coupons** | `coupons`, `coupon_pin_attempts` |
| **Redemption** | `redemptions`, `redemption_queue_offline` |
| **Wallet** | `merchant_ledger`, `breakage_ledger` |
| **Payouts** | `payouts`, `payout_line_items`, `bank_accounts` |
| **Refunds** | `refunds` |
| **Raffles** | `raffles`, `raffle_prizes`, `raffle_rules`, `raffle_entries`, `raffle_winners`, `raffle_seed_commitments` |
| **Loyalty** | `loyalty_balances`, `loyalty_transactions`, `referrals` |
| **Reviews** | `reviews`, `review_responses` |
| **Notifications** | `notifications`, `notification_preferences`, `notification_templates`, `push_devices` |
| **Ads** | `ad_campaigns`, `ad_campaign_metrics`, `ad_audience_templates` |
| **Invoicing** | `invoices`, `invoice_line_items` |
| **Favourites** | `favourites`, `merchant_follows` |
| **Audit** | `audit_logs` |
| **System** | `outbox_events`, `feature_flags`, `app_versions` |

Cross-module reads via the owning module's repository, not direct SQL.

---

## Identity sub-graph

```mermaid
erDiagram
    USERS {
        uuid id PK
        text phone UK
        text email
        text name
        text locale
        date birthdate
        text gender
        text emirate
        timestamp phone_verified_at
        timestamp email_verified_at
        timestamp created_at
        timestamp deleted_at
    }
    USER_DEVICES {
        uuid id PK
        uuid user_id FK
        text device_fingerprint
        text platform
        text app_version
        text push_token
        timestamp last_seen_at
    }
    MERCHANTS {
        uuid id PK
        text legal_name
        text brand_name
        text trn
        text status
        text kyc_status
        int commission_override_bps
        timestamp created_at
    }
    MERCHANT_BRANCHES {
        uuid id PK
        uuid merchant_id FK
        text name
        text address_line
        text city
        decimal lat
        decimal lng
        jsonb opening_hours
        timestamp created_at
    }
    MERCHANT_USERS {
        uuid id PK
        uuid merchant_id FK
        uuid user_id FK
        text role
        uuid_array branches_scope
        timestamp created_at
    }
    KYC_DOCUMENTS {
        uuid id PK
        uuid merchant_id FK
        text type
        text s3_key
        date expires_at
        text status
        text reviewer_admin_id
        timestamp created_at
    }
    ADMIN_USERS {
        uuid id PK
        text email UK
        text name
        text role
        text totp_secret_encrypted
        text status
        timestamp created_at
    }

    USERS ||--o{ USER_DEVICES : ""
    MERCHANTS ||--o{ MERCHANT_BRANCHES : ""
    MERCHANTS ||--o{ MERCHANT_USERS : ""
    MERCHANTS ||--o{ KYC_DOCUMENTS : ""
    USERS ||--o{ MERCHANT_USERS : ""
```

---

## Catalog sub-graph

```mermaid
erDiagram
    CATEGORIES {
        uuid id PK
        uuid parent_id FK
        text slug UK
        text name_en
        text name_ar
        text icon
        int sort_order
        timestamp created_at
    }
    DEALS {
        uuid id PK
        uuid merchant_id FK
        uuid category_id FK
        text type
        text status
        text title_en
        text title_ar
        text description_en
        text description_ar
        text fine_print_en
        text fine_print_ar
        int original_price_fils
        int deal_price_fils
        int commission_bps
        text usage_type
        int uses_total
        int inventory_total
        int inventory_left
        int max_per_user
        int max_per_day
        timestamp valid_from
        timestamp valid_until
        jsonb redemption_rules
        text hero_image_key
        text_array gallery_keys
        uuid approved_by
        timestamp approved_at
        timestamp created_at
    }
    DEAL_VERSIONS {
        uuid id PK
        uuid deal_id FK
        int version
        jsonb snapshot
        uuid created_by
        timestamp created_at
    }
    DEAL_BRANCHES {
        uuid deal_id FK
        uuid branch_id FK
    }
    TAGS {
        uuid id PK
        text slug UK
        text name_en
        text name_ar
    }
    DEAL_TAGS {
        uuid deal_id FK
        uuid tag_id FK
    }

    CATEGORIES ||--o{ CATEGORIES : ""
    CATEGORIES ||--o{ DEALS : ""
    DEALS ||--o{ DEAL_VERSIONS : ""
    DEALS ||--o{ DEAL_BRANCHES : ""
    DEALS ||--o{ DEAL_TAGS : ""
    TAGS ||--o{ DEAL_TAGS : ""
```

---

## Commerce + Payments sub-graph

```mermaid
erDiagram
    CARTS {
        uuid id PK
        uuid user_id FK
        timestamp created_at
        timestamp updated_at
    }
    CART_ITEMS {
        uuid id PK
        uuid cart_id FK
        uuid deal_id FK
        int quantity
        timestamp added_at
    }
    ORDERS {
        uuid id PK
        uuid user_id FK
        text status
        int subtotal_fils
        int promo_discount_fils
        int wallet_credit_fils
        int vat_fils
        int total_fils
        text currency
        uuid promo_code_id FK
        text idempotency_key
        timestamp created_at
        timestamp paid_at
    }
    ORDER_ITEMS {
        uuid id PK
        uuid order_id FK
        uuid deal_id FK
        uuid deal_version_id FK
        int quantity
        int unit_price_fils
        int commission_bps_snapshot
    }
    PAYMENTS {
        uuid id PK
        uuid order_id FK
        text provider
        text provider_intent_id
        text status
        int amount_fils
        text currency
        text method
        jsonb metadata
        timestamp created_at
        timestamp completed_at
    }
    PROMO_CODES {
        uuid id PK
        text code UK
        text scope
        int value_fils
        int value_bps
        int max_uses_total
        int used_count
        int max_uses_per_user
        timestamp valid_from
        timestamp valid_until
        bool first_purchase_only
        timestamp created_at
    }
    REFUNDS {
        uuid id PK
        uuid order_id FK
        int amount_fils
        text reason
        text status
        text provider_refund_id
        uuid initiated_by
        timestamp created_at
        timestamp completed_at
    }

    CARTS ||--o{ CART_ITEMS : ""
    ORDERS ||--|{ ORDER_ITEMS : ""
    ORDERS ||--o{ PAYMENTS : ""
    ORDERS ||--o{ REFUNDS : ""
    PROMO_CODES ||--o{ ORDERS : ""
```

---

## Coupons + Redemption sub-graph

```mermaid
erDiagram
    COUPONS {
        uuid id PK
        uuid order_id FK
        uuid order_item_id FK
        uuid deal_id FK
        uuid deal_version_id FK
        uuid user_id FK
        uuid merchant_id FK
        text short_code UK
        text qr_token_hash
        text pin_hash
        text usage_type
        int uses_total
        int uses_remaining
        text status
        timestamp valid_from
        timestamp expires_at
        timestamp last_redeemed_at
        timestamp locked_until
        int payout_amount_fils
        timestamp created_at
    }
    COUPON_PIN_ATTEMPTS {
        uuid id PK
        uuid coupon_id FK
        text result
        text source_ip
        text device_fingerprint
        timestamp attempted_at
    }
    REDEMPTIONS {
        uuid id PK
        uuid coupon_id FK
        uuid branch_id FK
        uuid cashier_user_id FK
        text device_fingerprint
        text source
        timestamp redeemed_at
    }
    REDEMPTION_QUEUE_OFFLINE {
        uuid id PK
        uuid coupon_id
        uuid branch_id FK
        uuid cashier_user_id FK
        text device_fingerprint
        text signature
        timestamp queued_at
        timestamp synced_at
        text sync_result
    }

    COUPONS ||--o{ COUPON_PIN_ATTEMPTS : ""
    COUPONS ||--o{ REDEMPTIONS : ""
```

---

## Wallet + Payouts sub-graph

```mermaid
erDiagram
    MERCHANT_LEDGER {
        uuid id PK
        uuid merchant_id FK
        text type
        int amount_fils
        text reference_kind
        uuid reference_id
        timestamp available_after
        timestamp created_at
    }
    BREAKAGE_LEDGER {
        uuid id PK
        uuid coupon_id FK
        int amount_fils
        timestamp recognized_at
    }
    BANK_ACCOUNTS {
        uuid id PK
        uuid merchant_id FK
        text iban
        text bank_name
        text holder_name
        text status
        timestamp verified_at
    }
    PAYOUTS {
        uuid id PK
        uuid merchant_id FK
        uuid bank_account_id FK
        int amount_fils
        text status
        text provider
        text provider_payout_id
        text bank_reference
        uuid approved_by
        timestamp requested_at
        timestamp approved_at
        timestamp paid_at
        timestamp failed_at
        text failure_reason
    }
    PAYOUT_LINE_ITEMS {
        uuid id PK
        uuid payout_id FK
        uuid ledger_entry_id FK
        int amount_fils
    }

    MERCHANT_LEDGER ||--o{ PAYOUT_LINE_ITEMS : ""
    PAYOUTS ||--|{ PAYOUT_LINE_ITEMS : ""
    BANK_ACCOUNTS ||--o{ PAYOUTS : ""
```

---

## Raffles sub-graph

```mermaid
erDiagram
    RAFFLES {
        uuid id PK
        text name_en
        text name_ar
        text description_en
        text description_ar
        text status
        timestamp period_start
        timestamp period_end
        timestamp draw_at
        text seed_commitment_hash
        text seed_revealed
        text public_block_reference
        timestamp commitment_published_at
        timestamp seed_revealed_at
        text tcs_url
        timestamp created_at
    }
    RAFFLE_PRIZES {
        uuid id PK
        uuid raffle_id FK
        text title_en
        text title_ar
        int value_fils
        int tier
        text photo_key
        text description_en
        text description_ar
    }
    RAFFLE_RULES {
        uuid id PK
        uuid raffle_id FK
        jsonb rule_dsl
        int sort_order
    }
    RAFFLE_ENTRIES {
        uuid id PK
        uuid raffle_id FK
        uuid user_id FK
        text source
        uuid source_ref
        int weight
        timestamp created_at
    }
    RAFFLE_WINNERS {
        uuid id PK
        uuid raffle_id FK
        uuid prize_id FK
        uuid user_id FK
        timestamp drawn_at
        timestamp notified_at
        timestamp claimed_at
        timestamp fulfilled_at
        text proof_photo_key
    }

    RAFFLES ||--o{ RAFFLE_PRIZES : ""
    RAFFLES ||--o{ RAFFLE_RULES : ""
    RAFFLES ||--o{ RAFFLE_ENTRIES : ""
    RAFFLES ||--o{ RAFFLE_WINNERS : ""
    RAFFLE_PRIZES ||--o{ RAFFLE_WINNERS : ""
```

---

## Notifications + audit + system

```mermaid
erDiagram
    NOTIFICATIONS {
        uuid id PK
        uuid user_id FK
        text channel
        text template_key
        jsonb payload
        text status
        text provider_message_id
        timestamp sent_at
        timestamp created_at
    }
    NOTIFICATION_PREFERENCES {
        uuid user_id PK
        bool push_drops
        bool push_flash
        bool push_raffle
        bool push_expiry
        bool email_marketing
        bool sms_marketing
        bool whatsapp_marketing
    }
    PUSH_DEVICES {
        uuid id PK
        uuid user_id FK
        text platform
        text token
        text app_version
        timestamp registered_at
        timestamp last_seen_at
    }
    AUDIT_LOGS {
        uuid id PK
        uuid actor_user_id
        text actor_role
        text action
        text target_type
        uuid target_id
        jsonb before_state
        jsonb after_state
        text source_ip
        text user_agent
        timestamp created_at
    }
    OUTBOX_EVENTS {
        uuid id PK
        text event_name
        int event_version
        jsonb payload
        timestamp occurred_at
        timestamp published_at
        int publish_attempts
        text last_error
        timestamp created_at
    }
    FEATURE_FLAGS {
        uuid id PK
        text key UK
        bool enabled
        jsonb rules
        timestamp updated_at
    }
    APP_VERSIONS {
        uuid id PK
        text platform
        text minimum_version
        text recommended_version
        timestamp updated_at
    }

    NOTIFICATIONS }o--|| USERS : ""
```

---

## Sizing & growth assumptions

| Table | Rows after 12 months (target) | Strategy |
|---|---|---|
| `users` | ~150,000 | Single table; B-tree on `phone`, `email` |
| `merchants` | ~600 | Trivial |
| `deals` | ~5,000 active + ~30,000 historical | Partial index on `status='approved'` |
| `orders` | ~2,000,000 | Range-partitioned by `created_at` quarterly after 6 months |
| `coupons` | ~3,000,000 | Range-partitioned by `created_at` quarterly after 6 months |
| `redemptions` | ~2,000,000 | Range-partitioned by `redeemed_at` quarterly |
| `merchant_ledger` | ~6,000,000 | Range-partitioned by `created_at` quarterly |
| `raffle_entries` | ~10,000,000 | Range-partitioned by `raffle_id` |
| `audit_logs` | ~5,000,000 | Append-only; range-partitioned by `created_at` monthly; archived after 12 months |
| `notifications` | ~50,000,000 | Range-partitioned by `created_at` weekly; pruned after 90 days |

---

## Migration strategy

See [03-migrations-strategy.md](./03-migrations-strategy.md). All schema changes go through Prisma migrations; production migrations are expand-then-contract to allow zero-downtime deploys.

---

## See also

- [Detailed entity definitions](./02-entities-detail.md)
- [Migrations strategy](./03-migrations-strategy.md)
- [Backend Architecture](../03-architecture/03-backend-architecture.md)
