# 02 · Authentication

**Status:** 🟢 Approved

How identity, sessions, and tokens work on Volu.

---

## Goals

- Strong security: no PAN storage, mandatory verification, refresh-token theft detection.
- Frictionless UX: phone OTP, biometric unlock, social sign-in, "remember me" via long-lived refresh tokens.
- Correctness in edge cases: device changes, lost devices, tourists with foreign numbers.
- Auditability: every privileged action attributable.

---

## Token model

### Access token (JWT, RS256)

- Lifetime: 15 minutes.
- Bearer in `Authorization: Bearer <jwt>`.
- Claims:

```json
{
  "sub": "01HXX...",       // user id
  "iss": "volu-identity",
  "aud": "volu-api",
  "iat": 1716200000,
  "exp": 1716200900,
  "jti": "01HXY...",       // unique token id
  "role": "user",          // user | merchant_owner | merchant_branch_manager | merchant_cashier | merchant_accountant | admin
  "deviceId": "01HXZ...",
  "merchantId": "01HXA...", // for merchant roles only
  "scope": ["consumer:read", "consumer:write"],
  "kid": "v2"              // signing-key id for rotation
}
```

- Verified at API Gateway (cached public key); also verified at services for defence in depth.
- Stateless: no DB lookup per request.

### Refresh token (opaque)

- Lifetime: 30 days from issue; sliding window on use.
- Format: 256-bit random base64url string. Indistinguishable from random.
- Stored hashed (Argon2id) in `identity__sessions`.
- Single-use: each refresh issues a new refresh token and invalidates the previous.
- Bound to device fingerprint and stored at the OS keychain on mobile.

### Token signing keys

- RS256 keypair generated at platform setup.
- Rotated every 90 days.
- Signed JWTs include `kid` (key id); JWT verification looks up the right public key.
- Old keys retained for the lifetime of the longest-living JWT issued under them (15 min) plus a buffer.

---

## Sign-up flows

### Phone (UAE primary)

```
1. POST /api/v1/auth/phone/start
   { phone: "+9715xxxxxxxx" }
   → 200 { sessionId: "...", expiresInSeconds: 300 }
   (server sends OTP via Unifonic SMS)

2. POST /api/v1/auth/phone/verify
   { sessionId, otp: "123456" }
   → 200 {
       accessToken, refreshToken,
       user: { id, name, locale, isNew: true }
     }
   (server creates user record if new; binds device)

3. (if isNew) POST /api/v1/users/me/profile
   { name, locale, ... }
```

### Email + password

```
1. POST /api/v1/auth/email/register
   { email, password, name }
   → 200 { sessionId, verificationEmailSent: true }

2. (link in email) GET /verify-email?token=...
   → user marks email verified; auto-redirects to app.

3. POST /api/v1/auth/email/login
   { email, password }
   → 200 { accessToken, refreshToken, user }
```

### Social (Apple / Google)

```
1. Mobile SDK obtains identity token from Apple/Google.

2. POST /api/v1/auth/social
   { provider: "apple", idToken: "..." }
   → 200 {
       accessToken, refreshToken,
       user: { id, name, isNew: true|false, requiresPhone: true }
     }

3. (if requiresPhone) phone verification flow as above.
```

---

## OTP delivery

| Provider | Use |
|---|---|
| **Unifonic** (UAE) | Primary for +971 numbers — best UAE deliverability |
| **Twilio** | Fallback for Unifonic failures and international numbers |
| **WhatsApp Business API** | User-opt-in alternative channel — useful for tourists with WhatsApp |

OTP rules:
- 6 digits, numeric.
- Lifetime: 5 minutes.
- Max attempts: 3 per OTP.
- Cooldown between resends: 60 seconds.
- Daily limit per phone: 10 OTPs.
- Failed OTP rate-limit per IP: 30/min.

---

## Refresh flow with theft detection

```
Client has refresh token RT_n.
Access token expires.
Client calls POST /auth/refresh with RT_n.
  ↓
Server hashes RT_n, looks up session.
  ├─ If session.refreshTokenHash == hash(RT_n) and not revoked:
  │     - Mark RT_n as used (revokedAt = now, revocationReason = "rotated")
  │     - Issue new RT_{n+1}; store hash; create new session row
  │     - Issue new access token
  │     - Return both
  │
  └─ If session was already used (RT_n used > 1 time):
        - Suspect theft. Revoke ALL sessions for this user immediately.
        - Force re-authentication.
        - Push notification: "Your account was accessed from a new device — please sign in again."
```

This is the standard rotating-refresh-token theft-detection pattern from OAuth 2.1.

---

## Device binding

Every refresh token is bound to a `deviceId` (device fingerprint hashed). Refresh requests must come from the same device. If the device changes (e.g., user moved to a new phone), they re-authenticate.

We don't bind access tokens to devices because they're short-lived; the binding via refresh tokens is sufficient.

---

## Biometric unlock (mobile app PIN)

Separate from authentication. After login, the user can enable biometric unlock in settings:

- App stores a separate "app PIN" hash in OS keychain.
- On cold start, app shows biometric prompt; on success, releases JWT for use.
- Coupon PINs (4-digit codes for redemption) are gated behind biometric unlock — visible only after FaceID/fingerprint succeeds.

---

## Logout

```
POST /api/v1/auth/logout
Authorization: Bearer <access>
{ "refreshToken": "<rt>" }
  ↓
Server: revoke session for this RT; access token expires naturally in ≤ 15 min.
```

For "logout all devices":

```
POST /api/v1/auth/logout-all
  ↓
Server: revoke ALL sessions for this user.
```

---

## Admin authentication

Admin login requires:

1. Email + password (Argon2id).
2. TOTP 2FA (mandatory).
3. Optional: IP allowlist for Super Admin role.

Admin tokens:
- Access token: 30 minutes (longer than user; admins do longer focused work).
- Refresh token: 8 hours.
- Idle session timeout: 30 minutes.
- Hard session limit: 8 hours then re-login.

Admin login flow:

```
1. POST /api/v1/admin/auth/login
   { email, password }
   → 200 { challengeToken, requires2fa: true }
   (no access token issued yet)

2. POST /api/v1/admin/auth/2fa
   { challengeToken, totpCode: "123456" }
   → 200 { accessToken, refreshToken, admin: { id, role } }
```

---

## Merchant authentication

Same flow as user: phone OTP. After login, if the user is associated with one or more merchants, the JWT includes:

```json
{
  "sub": "<userId>",
  "role": "merchant_owner",
  "merchantId": "<merchantId>",
  "branchScope": ["<branchId>", "<branchId>"]
}
```

If the user is associated with multiple merchants, they choose at login (small "switch merchant" picker).

Cashier role: Owner invites a cashier; cashier signs in; sees only the scanner UI.

Owner can invite/revoke any time; revocation invalidates all sessions for that merchant role.

---

## Token revocation

Mechanisms:

- **Logout** — single session revoked.
- **Logout-all** — all sessions revoked.
- **Theft detection** — automatic.
- **Admin force-logout** — admin can revoke all sessions for any user (after suspect activity).
- **Merchant role revocation** — when a merchant fires a cashier, all their merchant-scoped sessions revoked.
- **Account deletion** — all sessions revoked.

Revocation list cached in Redis; access tokens that haven't expired but belong to a revoked session are rejected by API Gateway.

---

## Rate limits

| Endpoint | Limit |
|---|---|
| `/auth/phone/start` | 5/hour/phone, 30/hour/IP |
| `/auth/phone/verify` | 10/hour/sessionId, 30/hour/IP |
| `/auth/email/login` | 10/min/email, 30/min/IP |
| `/auth/social` | 30/min/IP |
| `/auth/refresh` | 60/min/user |
| `/admin/auth/login` | 5/min/email, 5 failed → 15-min lockout |

---

## Logging and audit

Every authentication event logged:

- Successful login (user id, device, IP, method).
- Failed login (email/phone, IP, reason).
- Token refresh (user id, device).
- Logout (user id, scope).
- Theft detection trigger (user id, sessions revoked).
- Admin 2FA enrol/disable.
- Admin role change (by/of/timestamp).

---

## Security headers (web admin)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), camera=(), microphone=()
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' https://js.stripe.com; ...
```

Mobile apps: certificate pinning to api.volu.ae's leaf certificate (or its issuer's intermediate, with fallback).

---

## See also

- [API Design Principles](./01-api-design-principles.md)
- [Security Controls](../06-security/02-security-controls.md)
- [Threat Model](../06-security/01-threat-model.md)
