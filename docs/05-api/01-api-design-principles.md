# 01 · API Design Principles

**Status:** 🟢 Approved

How Volu APIs are designed, versioned, documented, and evolved.

---

## Style: REST + JSON

- **REST** with resource-oriented URLs.
- **JSON** for request and response bodies.
- **HTTP status codes** used semantically.
- **OpenAPI 3.1** as the contract; client SDKs generated from it.

We use REST not because it's perfect, but because it's the default that every developer understands, every tool supports, and Stripe / Twilio / GitHub have proven works at scale. GraphQL was considered and rejected for v1 ([Tech Stack](../03-architecture/02-tech-stack.md)).

---

## URL conventions

```
https://api.volu.ae/api/v1/<resource>
```

- Resources are plural nouns: `/deals`, `/orders`, `/coupons`.
- Single resource: `/deals/{id}`.
- Sub-resources: `/merchants/{id}/branches`.
- Actions on resources (rare; prefer state changes): `/payouts/{id}/approve`.
- No verbs in paths. State changes are POSTs to the resource or to a verb sub-path.

### Versioning

URL path versioning: `/api/v1/...`. Bumped only for breaking changes. Old versions supported for at least 6 months.

### Public vs Admin

- `/api/v1/...` — for end-user mobile app.
- `/api/v1/merchant/...` — for merchant mobile app.
- `/api/v1/admin/...` — for admin console.

These are logically separate even if served by the same backend; auth scope is enforced per surface.

---

## Methods

| Method   | Use                               |
| -------- | --------------------------------- |
| `GET`    | Read; safe; cacheable; idempotent |
| `POST`   | Create; or non-idempotent action  |
| `PUT`    | Replace entire resource (rare)    |
| `PATCH`  | Partial update                    |
| `DELETE` | Soft-delete                       |

---

## Status codes

| Code              | When                                         |
| ----------------- | -------------------------------------------- |
| `200`             | Successful read or update                    |
| `201`             | Successful resource creation                 |
| `202`             | Async work accepted (e.g., refund queued)    |
| `204`             | Successful delete or no-content response     |
| `400`             | Client validation error                      |
| `401`             | Authentication required or expired           |
| `403`             | Authenticated but not authorised             |
| `404`             | Resource not found                           |
| `409`             | Conflict (e.g., duplicate, version mismatch) |
| `410`             | Gone (resource intentionally removed)        |
| `422`             | Semantic validation error                    |
| `429`             | Rate-limited                                 |
| `500`             | Unhandled server error                       |
| `502 / 503 / 504` | Upstream / unavailable / timeout             |

---

## Request format

### Headers

| Header                            | Required               | Notes                                   |
| --------------------------------- | ---------------------- | --------------------------------------- |
| `Authorization: Bearer <jwt>`     | For authed endpoints   |                                         |
| `Content-Type: application/json`  | For POST/PATCH         |                                         |
| `Accept-Language: en` or `ar`     | Optional               | Drives EN/AR copy in responses          |
| `X-Idempotency-Key: <uuid>`       | Required for mutations | UUID v4                                 |
| `X-Request-Id: <uuid>`            | Optional               | Echoed in response; client traceability |
| `X-App-Version: <semver>+<build>` | Required from mobile   | For force-update gating                 |
| `X-Platform: ios / android / web` | Required               |                                         |
| `X-Device-Id: <fingerprint>`      | Required from mobile   |                                         |

### Body

JSON, camelCase keys.

```json
{
  "dealId": "01HXXXX...",
  "quantity": 2,
  "promoCode": "WELCOME10"
}
```

### Query parameters

camelCase. Standard params:

- `?limit=25` — page size (max 100)
- `?cursor=<opaque>` — cursor pagination (preferred over offset)
- `?sort=createdAt:desc` — sort field with direction
- `?filter[status]=active` — filter
- `?include=merchant,branches` — sparse expansion

---

## Response format

### Success (single resource)

```json
{
  "data": {
    "id": "01HXXXX...",
    "type": "deal",
    "title": "Spa Massage",
    "dealPrice": 15000,
    "currency": "AED",
    "merchantId": "01HXXXX..."
  }
}
```

### Success (list)

```json
{
  "data": [
    { "id": "01HXX1", "type": "deal", "title": "..." },
    { "id": "01HXX2", "type": "deal", "title": "..." }
  ],
  "pagination": {
    "nextCursor": "eyJpZCI6IjAxSFh...",
    "previousCursor": null,
    "hasMore": true
  },
  "meta": {
    "totalCount": 1842,
    "filtersApplied": { "status": "active" }
  }
}
```

### Error

```json
{
  "error": {
    "code": "DEAL_INVENTORY_EXHAUSTED",
    "message": "This deal is sold out.",
    "messageAr": "نفدت هذه الصفقة.",
    "context": {
      "dealId": "01HXX...",
      "inventoryLeft": 0
    }
  },
  "traceId": "01HXY..."
}
```

Error codes are stable, documented, and mapped to user-facing copy by clients (mobile renders friendly messages; admin shows technical details).

---

## Authentication

Bearer JWT in `Authorization` header. JWTs are signed with RS256, public key cached at API Gateway.

JWT claims (relevant subset):

```json
{
  "sub": "01HXX...",
  "iss": "volu-identity",
  "aud": "volu-api",
  "iat": 1716200000,
  "exp": 1716200900,
  "role": "user",
  "deviceId": "...",
  "scope": ["consumer:read", "consumer:write"]
}
```

Access tokens expire in 15 minutes. Clients use refresh token endpoint to rotate. Refresh tokens are opaque, stored hashed, bound to device, and invalidated on reuse-after-rotation (theft detection).

Detailed in [Authentication](./02-authentication.md).

---

## Authorisation

Per-endpoint role checks via NestJS guards:

```ts
@Roles(AdminRole.OPERATIONS, AdminRole.SUPER)
@Post('merchants/:id/approve')
approve(...) { ... }
```

For resource-scoped checks (e.g., merchant can only see their own data), service layer enforces ownership:

```ts
async getDeal(dealId: string, currentUser: User) {
  const deal = await this.dealRepo.findById(dealId);
  if (!deal) throw new NotFoundError();
  if (currentUser.role === 'merchant' && deal.merchantId !== currentUser.merchantId) {
    throw new ForbiddenError();
  }
  return deal;
}
```

---

## Idempotency

Mandatory `X-Idempotency-Key` (UUID v4) on every mutating endpoint. Server caches the response keyed by `(idempotencyKey + endpoint + userId + bodyHash)` for 24h.

Retries with the same key:

- Same body → return cached response.
- Different body → `409 Conflict`.

Stripe-style: enables safe retries from flaky mobile networks without double-charging.

---

## Pagination

Cursor-based by default:

- Server returns `nextCursor` (opaque) in `pagination`.
- Client passes `?cursor=<value>` for the next page.
- No total count by default (expensive on big tables); `?withTotal=true` opt-in for admin.

For small admin lists, offset+limit is acceptable: `?page=2&limit=50`.

---

## Filtering and sorting

Filtering uses bracket-syntax: `?filter[status]=approved&filter[merchantId]=01HXX...`.

Multi-value: `?filter[status][in]=approved,scheduled`.

Sorting: `?sort=createdAt:desc,name:asc`.

Whitelisted fields per endpoint; arbitrary filtering not allowed.

---

## Sparse fields and includes

To reduce payload:

- `?fields=id,title,price` — only return listed fields.
- `?include=merchant,branches` — eager-load relations.

Each endpoint declares supported `fields` and `include` values.

---

## Rate limiting

Per-endpoint limits expressed in requests per minute. Common defaults:

| Endpoint class             | Anonymous | Authenticated |
| -------------------------- | --------- | ------------- |
| Auth (login, OTP, refresh) | 10/min/IP | 30/min/user   |
| Search                     | 30/min/IP | 60/min/user   |
| Read (deal, merchant)      | 60/min/IP | 200/min/user  |
| Cart, checkout, payment    | n/a       | 30/min/user   |
| Admin endpoints            | n/a       | 200/min/user  |

When rate-limited:

```
HTTP 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716200030
```

Implementation: Redis sliding-window per `(endpoint, key)` where key is IP or user ID.

---

## Internationalisation in responses

When the user has set a locale (via `Accept-Language` or stored preference), responses include localised content for translatable fields:

```json
{
  "id": "01HXX...",
  "title": "Spa Massage", // resolves to en or ar based on locale
  "titleEn": "Spa Massage", // always present
  "titleAr": "تدليك سبا" // always present if available
}
```

Error messages include both `message` (locale) and `messageAr` (always Arabic).

---

## Time and money in responses

- **Timestamps**: ISO-8601 with timezone, e.g., `"2026-04-23T14:30:00+04:00"`.
- **Money**: integer fils + currency: `{ "amountFils": 15000, "currency": "AED" }`. Clients render as AED 150.00. Never floats over the wire.

---

## Deprecation policy

When deprecating an endpoint:

1. Add `Deprecation: true` and `Sunset: <date>` headers to responses.
2. Document the replacement in OpenAPI.
3. Send weekly digest emails to API consumers (admin team).
4. Sunset minimum 6 months later.

---

## Documentation

OpenAPI 3.1 spec at `/api/openapi.json`. Auto-generated from NestJS controllers + decorators. Hosted Swagger UI at `/api/docs` (admin-only access in production).

Every endpoint must document:

- Path + method
- Description
- Request body schema (with examples)
- Response body schema (with examples)
- Possible error codes
- Required scope/role
- Idempotency key requirement (yes/no)
- Rate limit class

Generated TypeScript SDK and Dart SDK are checked into respective monorepos.

---

## Backwards compatibility

Within a major version (`v1`):

- ✅ Add new endpoints
- ✅ Add optional request fields
- ✅ Add response fields
- ✅ Add new enum values (clients must handle unknown values gracefully)
- ❌ Remove fields
- ❌ Change field types
- ❌ Change error semantics

Any breaking change requires `v2`.

---

## Webhook signing

Outbound webhooks signed with HMAC-SHA256 over `(timestamp + body)` using a shared secret per subscriber. Header: `X-Volu-Signature: t=<unix>,v1=<sig>`.

Detailed in [Webhooks](./04-webhooks.md).

---

## Examples

### Get a deal

```http
GET /api/v1/deals/01HXX... HTTP/1.1
Host: api.volu.ae
Authorization: Bearer eyJ...
Accept-Language: en
X-Request-Id: 01HXX...

HTTP/1.1 200 OK
Content-Type: application/json
X-Request-Id: 01HXX...

{
  "data": {
    "id": "01HXX...",
    "type": "deal",
    "merchantId": "01HXY...",
    "title": "Mediterranean Brunch for Two",
    "originalPrice": { "amountFils": 40000, "currency": "AED" },
    "dealPrice": { "amountFils": 19500, "currency": "AED" },
    "savingsPercent": 51,
    "validUntil": "2026-05-31T20:59:59+04:00",
    "inventoryLeft": 47,
    "branches": [
      { "id": "01HXZ...", "name": "Marina Branch", "lat": 25.080, "lng": 55.140 }
    ],
    "verifiedMerchant": true
  }
}
```

### Create an order

```http
POST /api/v1/orders HTTP/1.1
Host: api.volu.ae
Authorization: Bearer eyJ...
Content-Type: application/json
X-Idempotency-Key: 01HXY...
X-Request-Id: 01HXX...

{
  "items": [
    { "dealId": "01HXX...", "quantity": 2 }
  ],
  "promoCode": "WELCOME10"
}

HTTP/1.1 201 Created
Content-Type: application/json
X-Request-Id: 01HXX...

{
  "data": {
    "id": "01HXY...",
    "status": "pending",
    "subtotal": { "amountFils": 39000, "currency": "AED" },
    "promoDiscount": { "amountFils": 3900, "currency": "AED" },
    "vat": { "amountFils": 1755, "currency": "AED" },
    "total": { "amountFils": 36855, "currency": "AED" },
    "stripeClientSecret": "pi_xxx_secret_yyy",
    "expiresAt": "2026-04-23T14:45:00+04:00"
  }
}
```

### Validate redemption

```http
POST /api/v1/merchant/redemptions/validate HTTP/1.1
Authorization: Bearer eyJ...merchant...
Content-Type: application/json
X-Idempotency-Key: 01HX...

{
  "qrToken": "eyJjb3Vwb25JZCI6...",
  "branchId": "01HXZ..."
}

HTTP/1.1 200 OK
{
  "data": {
    "couponId": "01HXX...",
    "dealTitle": "Mediterranean Brunch for Two",
    "usesRemaining": 1,
    "requiresPin": true
  }
}
```

---

## See also

- [Authentication](./02-authentication.md)
- [Endpoint Catalogue](./03-endpoints-catalog.md)
- [Webhooks](./04-webhooks.md)
- [Backend Architecture](../03-architecture/03-backend-architecture.md)
