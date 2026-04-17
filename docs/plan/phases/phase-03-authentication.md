# Phase 3: Authentication System

## Dependencies
- Phase 2 must be completed (User, ConsentVersion, ConsentRecord tables)

## Goals
- Registration and login with email + password
- Registration/login with Google OAuth 2.0 + profile completion flow
- JWT access + refresh token mechanism
- Email verification (code delivery via Resend)
- Consent text approval mechanism
- Rate limiting (brute-force protection)
- Define auth DTO schemas in the shared package

## Tasks

### 1. Shared Zod Schemas (packages/shared)

In `packages/shared/src/schemas/auth.ts`:

**RegisterSchema:**
- email: email format, required
- password: min 8 characters, at least 1 uppercase, 1 lowercase, 1 digit
- firstName: min 2, max 50 characters
- lastName: min 2, max 50 characters
- username: min 3, max 30 characters, only letters/digits/underscores, normalized to lowercase
- dateOfBirth: ISO date string, 13-year age check
- gender: MALE | FEMALE | UNSPECIFIED

**LoginSchema:**
- email: email format
- password: string

**VerifyEmailSchema:**
- code: 6-digit string

**RefreshTokenSchema:**
- refreshToken: string

**ProfileCompletionSchema** (after Google OAuth):
- username: same rules as RegisterSchema
- gender: MALE | FEMALE | UNSPECIFIED
- dateOfBirth: ISO date string, 13-year age check

**ConsentAcceptSchema:**
- consentVersionIds: UUID array (min 1)

### 2. Auth Module (apps/api)

Under `apps/api/src/modules/auth/`:

```
auth/
├── auth.module.ts
├── auth.controller.ts
├── auth.service.ts
├── strategies/
│   ├── jwt.strategy.ts          # Access token verification
│   ├── jwt-refresh.strategy.ts  # Refresh token verification
│   └── google.strategy.ts       # Google OAuth
├── guards/
│   ├── jwt-auth.guard.ts
│   ├── jwt-refresh.guard.ts
│   └── google-auth.guard.ts
├── decorators/
│   ├── current-user.decorator.ts  # Get active user from request
│   ├── public.decorator.ts        # Endpoints that don't require auth
│   └── verified.decorator.ts      # Endpoints that require email verification
└── dto/
```

### 3. JWT Strategy

**Access Token:**
- Payload: `{ sub: userId, email, username, isVerified: boolean, sv: number }` — `sv` is `User.sessionVersion` (Phase 2) embedded at issue time
- Duration: `JWT_ACCESS_EXPIRATION` (15m)
- Secret: `JWT_SECRET`
- Transport to the frontend: returned inside the JSON response body. The frontend keeps it in memory (with an optional short-lived `sessionStorage` mirror that is cleared on tab close). It is **never** written to `localStorage`, because a successful XSS otherwise has full session take-over capability

**Refresh Token:**
- Payload: `{ sub: userId, sv: number }` — `sv` is the user's current `sessionVersion` at issue time
- Duration: `JWT_REFRESH_EXPIRATION` (7d)
- Secret: `JWT_REFRESH_SECRET`
- Session revocation is driven by the `sv` claim (see Section 11.1). `/auth/refresh` rejects tokens whose `sv` is lower than the user's current `User.sessionVersion`, so a password/email change can cut all other sessions by incrementing that single integer. No per-token blacklist table is needed
- Transport: the backend sets the refresh token as an **`httpOnly; Secure; SameSite=Strict`** cookie scoped to the auth path. JavaScript cannot read it, so XSS cannot exfiltrate it. The cookie's `Path` is locked to the auth endpoints (`/api/v1/auth`) so it is transmitted only when the browser calls `/auth/refresh` or `/auth/logout`. In development (`NODE_ENV=development`) the `Secure` flag is omitted so the cookie works over `http://localhost`; every other environment sets it

**Cookie configuration (`main.ts`):**
- Register `cookie-parser` middleware so guards and controllers can read the refresh cookie server-side
- CORS configuration (Phase 3 Section 14) must declare `credentials: true` and an explicit origin list — browsers refuse `SameSite=Strict` cookies on cross-origin requests when `*` is used as the allowed origin
- Expose the refresh endpoint at `POST /auth/refresh` only — the cookie is never accepted on any other route

**Token return format (login / register / refresh responses):**
```json
{
  "accessToken": "...",
  "user": { "id", "email", "username", "isVerified", "firstName", "lastName" }
}
```
The refresh token is **not** in the JSON body — it ships in the `Set-Cookie` header as `refreshToken=...; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`.

**Refresh flow:** the frontend calls `POST /auth/refresh` with `credentials: "include"`; the browser attaches the refresh cookie automatically. The backend verifies the signature, then compares the token's `sv` claim against `User.sessionVersion`; a mismatch returns 401 `{ code: "AUTH_TOKEN_EXPIRED" }` and clears the refresh cookie so the client falls back to the login page. On success the backend issues a new access token, rotates the refresh cookie (writes a new one with a fresh 7-day TTL and the current `sv`), and returns the new access token in the response body. Token rotation limits the window in which a stolen refresh token could be reused.

**Concurrent refresh calls (multi-tab safety):** `apps/web` coalesces parallel refreshes into a single in-flight promise (Phase 11 Section 5). This prevents two simultaneous tabs from racing their `POST /auth/refresh` calls and causing one to overwrite the other's rotated cookie — only one request reaches the backend per refresh cycle.

**Logout flow:** `POST /auth/logout` sets the refresh cookie to an empty value with `Max-Age=0`, effectively deleting it. The frontend also drops the in-memory access token.

### 4. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /auth/register | Public | Register with email + password |
| POST | /auth/login | Public | Login with email + password |
| POST | /auth/refresh | Public | Get new access token with refresh token |
| POST | /auth/verify-email | JWT | Send email verification code |
| POST | /auth/verify-email/confirm | JWT | Confirm verification code |
| POST | /auth/verify-login-device | Public | Verify new-device code for authority-role login (Section 11.2) |
| GET | /auth/google | Public | Initiate Google OAuth |
| GET | /auth/google/callback | Public | Google OAuth callback |
| POST | /auth/complete-profile | JWT | Complete profile after OAuth |
| POST | /auth/accept-consent | JWT | Approve consent text |
| POST | /auth/logout | JWT | Logout (client-side token cleanup) |
| POST | /auth/forgot-password | Public | Send password reset email |
| POST | /auth/reset-password | Public | Reset password (with token) |

### 5. Registration Flow (Email + Password)

All writes in this flow happen inside a single `prisma.$transaction` so a partial failure cannot leave a half-created account.

1. Client sends data with `RegisterSchema`
2. Server-side validation (Zod)
3. Email and username uniqueness check
4. Password is hashed with bcrypt (salt rounds: 12)
5. User is created (`emailVerifiedAt: null`, `provider: LOCAL`, `totalXp: 0`)
6. Default avatar is assigned by drawing one random row from the free-pool: `WHERE category = 'free' AND isActive = true ORDER BY RANDOM() LIMIT 1`. All 15 free-pool avatars are interchangeable initial-pool candidates — the previous single-`isDefault` behaviour is retired along with the `isDefault` field (Phase 2 Section 3)
7. USER role is assigned
8. **Initial rank assignment:** Look up the lowest-threshold active rank from the `Rank` table (`ORDER BY minXp ASC, sortOrder ASC LIMIT 1`) and set `user.currentRankId` to its id. With the seeded rank table this resolves to "Gözlemci" (0 XP). The assignment is part of the same transaction so every registered account always has a non-null `currentRankId` by the time the transaction commits — downstream card responses (`UserPublicCardSchema.rank.name`) never need to handle a null rank. The Phase 2 rank seed and Phase 8's `checkAndUpdateRank` reuse the same ordering rule. `User.highestXpReached` defaults to `0` and begins tracking from the first `awardXp` call
9. 6-digit verification code is generated and stored in Redis (TTL: 10 minutes, key: `email-verify:{userId}`)
10. Verification code is sent to the user's email via Resend (React Email template)
11. Access + refresh tokens are returned
12. **Post-registration redirect order:** Register → Consent screen → Email verification. Consent approval must happen before email verification. The client redirects to the consent screen first, then to email verification.

**Google OAuth new-account branch:** When the OAuth callback creates a fresh user (`provider: GOOGLE`, `emailVerifiedAt: now()`), the same initial rank assignment is applied inside the user-creation transaction. This keeps the invariant consistent across both registration paths.

### 6. Email Verification Flow

1. `POST /auth/verify-email` → generate new code, save to Redis, send email. Rate limit: 1 request per 60 seconds.
2. `POST /auth/verify-email/confirm` → verify code from Redis, update `emailVerifiedAt = now()`, delete code from Redis.
3. Wrong code entry: 5 attempts allowed (Redis counter, TTL: 10 min). If exceeded, temporary block.

### 7. Google OAuth Flow

**IMPORTANT:** Since the global prefix is `/api/v1`, the callback URL must be `http://localhost:3001/api/v1/auth/google/callback`. The `GOOGLE_CALLBACK_URL` in `docs/environments.md` and the redirect URI in Google Cloud Console must match this.

**Action required:** Update `GOOGLE_CALLBACK_URL` in `docs/environments.md` to `http://localhost:3001/api/v1/auth/google/callback` to match this requirement.

1. `GET /auth/google` → redirect to Google with Passport Google Strategy
2. Google callback → `GET /auth/google/callback`
3. Profile info from Google: `email`, `firstName`, `lastName`, `providerId`
4. Search for existing user by email:
   - **User exists, provider: GOOGLE** → log in, return tokens
   - **User exists, provider: LOCAL** → error: "Bu email adresi ile zaten bir hesabınız var. Lütfen şifrenizle giriş yapın." (error code: `AUTH_OAUTH_EMAIL_CONFLICT`). Frontend must display this error clearly to the user with a link/button to the login page.
   - **User doesn't exist** → create new user (`provider: GOOGLE`, `emailVerifiedAt: now()`, `username: null`)
5. If username is null → `requiresProfileCompletion: true` flag in response
6. Client redirects to profile completion page based on this flag

### 8. Profile Completion (OAuth)

`POST /auth/complete-profile`:
1. Validation with `ProfileCompletionSchema`
2. Username uniqueness check
3. Update user: `username`, `gender`, `dateOfBirth`
4. Return new tokens with updated user info

### 8.1. Profile Completion Guard

Users with incomplete profiles (missing `username`, `firstName`, or `lastName`) are blocked from all write actions across the platform (topic create, comment, vote). This is enforced:
- **Backend:** A `ProfileCompleteGuard` checks these fields before any write endpoint. Returns 403 with code `USER_PROFILE_INCOMPLETE` and a message: "Devam etmek için profilinizi tamamlayın".
- **Frontend:** Auth provider checks profile completeness and redirects to `/complete-profile` when a write action is attempted.

This guard is critical because OAuth users (Google) may have null `username` and potentially missing name fields. The guard ensures no anonymous-looking content (empty author info) appears on the platform.

### 9. Consent Text Approval

`POST /auth/accept-consent`:
1. Client sends current consent text version IDs
2. A `ConsentRecord` is created for each version (`ipAddress` is recorded)
3. After registration and verification, the client needs to show the consent screen
4. Consent status is returned as a flag in the user response (`hasAcceptedConsent: boolean`)

### 10. Email Delivery (Resend + React Email)

Under `apps/api/src/modules/mail/`:

**`mail.module.ts`** and **`mail.service.ts`**:
- Email delivery with Resend SDK
- Template rendering with React Email

**Templates:**
- `email-verification.tsx` — Verification code email. Platform logo, short message, 6-digit code, expiration time info.
- `welcome.tsx` — Welcome email sent after successful email verification. Content: "Hakver'e Hoş Geldiniz!", brief platform introduction, first steps (1. Bir konuya oy verin, 2. İlk konunuzu açın), link to home page. Sent via BullMQ queue (async, fire-and-forget).

**Async email processing with BullMQ:**
- Install `@nestjs/bullmq` + `bullmq`
- Create `MailQueue` with Redis connection (`REDIS_URL`)
- Email sending jobs are added to the queue instead of sending synchronously
- API responses don't wait for email delivery — fire and forget via queue
- Retry: 3 attempts with exponential backoff
- Dead letter queue for failed emails (logged for debugging)
- This ensures API response times aren't affected by email service latency

### 11. Password Reset Flow

`POST /auth/forgot-password`:
1. Email address is received
2. Check if user exists (return 200 even if not — email enumeration prevention)
3. If OAuth user → return 200 but don't send email
4. Generate a cryptographically strong random reset token (32 bytes, URL-safe Base64), store in Redis with **TTL 15 minutes** (key: `password-reset:{token}`, value: target `userId`). Every reset request invalidates any prior outstanding token for the same user (the service deletes `password-reset:*` entries whose value matches the user id before writing the new key) so an attacker cannot accumulate multiple valid tokens
5. Send password reset link via email using the **URL fragment** form: `FRONTEND_URL/reset-password#token={token}`. Fragments never travel over the wire to the backend, never appear in reverse-proxy or Vercel access logs, and are not sent as `Referer` to third-party assets. The email template mentions both the link and the 15-minute validity window
6. Rate limit: 1 request / 60 seconds (email-based)

`POST /auth/reset-password`:
1. Token and new password are received in the request body (the frontend reads `window.location.hash`, strips it via `history.replaceState`, and POSTs the token + new password). Never accept the token via query string — if an incoming request has `?token=` only, return 400 `{ code: "VALIDATION_INVALID_INPUT" }`
2. Look up the token in Redis. When the key is missing or the value does not resolve to an existing non-deleted user → return 400 `{ code: "AUTH_RESET_TOKEN_INVALID", message: "Bağlantının süresi dolmuş veya geçersiz" }` so the frontend can render the expired-link screen
3. New password validation (same rules as RegisterSchema)
4. Hash password with bcrypt, update the user's `passwordHash`
5. Delete the token from Redis (single-use) and call `invalidateOtherSessions(userId, { reason: "password-reset" })` (Section 11.1) — increments `User.sessionVersion` so every other live session fails its next refresh and is forced back to the login page
6. Response: 200

**Frontend behaviour** (Phase 12):
- `/reset-password` page reads `window.location.hash`; if empty, shows the Step 1 email form. If present, shows the Step 2 new-password form
- Step 2 calls `POST /auth/reset-password` with the token in the JSON body (not the URL). On success: toast "Şifreniz başarıyla değiştirildi" + redirect to `/login`
- On `AUTH_RESET_TOKEN_INVALID`: render a dedicated "Bağlantının süresi dolmuş" screen with a single CTA "Yeni bağlantı al" that re-opens the Step 1 form

**Email template:** `password-reset.tsx` — Turkish copy, reset link using the fragment form, explicit "15 dakika içinde kullanılmalı" note, small footer clarifying "Bu isteği siz yapmadıysanız dikkate almayın".

### 11.1. Session Invalidation Helper (`invalidateOtherSessions`)

Security-critical account changes (password reset, password change, email change, future 2FA enrolment) must be able to revoke other live sessions. Without a per-token table, revocation is driven by a monotonic `User.sessionVersion` counter (Phase 2) that every issued JWT embeds as the `sv` claim. A single integer per user is enough — no token-tracking table, no stateful refresh-chain infrastructure.

**Contract:** `invalidateOtherSessions(userId: string, { reason: "password-reset" | "password-change" | "email-change" }): Promise<{ accessToken: string; refreshTokenCookie: string }>`

**Behavior:**
1. `UPDATE "User" SET "sessionVersion" = "sessionVersion" + 1 WHERE id = :userId RETURNING *` — atomic at row level, no `FOR UPDATE` needed
2. Issue a fresh access + refresh token pair carrying the new `sv` value so the caller's current session continues seamlessly
3. Write an ActivityLog entry: `action = "auth:sessions-invalidated"`, `metadata: { reason }` (security/moderation category, 1 year retention per Phase 10 Section 9.1)
4. Return the token pair to the controller; the controller sets the new refresh cookie and returns the new access token in the response body

**Effect on other live sessions:**
- Any other device holding an older-`sv` access token keeps working until its 15-minute expiry (acceptable minor window — the access token is short-lived by design)
- The next `/auth/refresh` from that device compares the refresh token's `sv` against the updated `User.sessionVersion`, fails, returns 401 `AUTH_TOKEN_EXPIRED`, clears the refresh cookie, and the device is forced back to the login page

**Callers (MVP):**
- `POST /auth/reset-password` (Section 11) — `reason: "password-reset"`
- `PATCH /users/me/email` (Phase 4 Section 7) — `reason: "email-change"`
- `PATCH /users/me/password` (Phase 4 Section 8) — `reason: "password-change"`

Future callers (2FA enrolment, phone attach, admin role change impact on the affected user) adopt the same helper without schema changes.

### 11.2. New Device Login Verification (Authority Roles)

Admin and moderator logins carry higher impact than regular-user logins (role assignment, XP adjustment, broadcast dispatch), so MVP adds an email-code verification step whenever an authority account logs in from a device that has not been seen for this user in the last 30 days. Regular-user new-device verification is deferred to post-MVP; the infrastructure below will extend there when rolled out.

**Device fingerprint:** `deviceHash = SHA-256(userId + userAgent + truncatedIp)` where `truncatedIp` collapses the request IP to its `/24` (IPv4) or `/64` (IPv6) subnet so natural ISP-assigned rotations do not re-trigger verification every session.

**Storage:** Redis key `trusted-device:{userId}:{deviceHash}` with TTL 30 days. Presence means the device has passed verification for this user within the last 30 days. Verification code storage: `login-device-verify:{userId}:{deviceHash}` with TTL 10 minutes.

**Login flow (authority accounts only — USER role logs in normally):**
1. After password/OAuth verification succeeds (Section 5 / Section 7), check the caller's roles. If the user holds `ADMIN` or `MODERATOR` → continue to step 2; otherwise issue tokens normally and return
2. Compute `deviceHash`. If `trusted-device:{userId}:{deviceHash}` exists → issue tokens normally and return
3. Otherwise: generate a 6-digit code, store it at the `login-device-verify:*` key, and enqueue an email via the existing `mail.service.ts` (new template `login-device-verify.tsx` — Turkish copy: "Yeni cihazdan giriş isteği. Doğrulama kodu: XXXXXX. 10 dakika içinde kullanılmalı")
4. Respond with `202 Accepted` and body `{ requiresDeviceVerification: true, deviceHash, message: "Giriş için email adresinize gönderilen kodu girin" }`. No access or refresh tokens are issued yet; the response does not set the refresh cookie
5. Client submits the code via `POST /auth/verify-login-device` with `{ deviceHash, code, email, password }` (the re-authentication fields protect against session fixation — the backend re-verifies credentials before accepting the code). On match: delete the verification code from Redis, write `trusted-device:{userId}:{deviceHash}` with 30-day TTL, issue the normal token pair
6. Wrong code: 400 `{ code: "AUTH_INVALID_CODE", message: "Doğrulama kodu geçersiz" }`. A Redis counter tracks failures; 5 failed attempts per `(userId, deviceHash)` within the 10-minute window locks the attempt and invalidates the code — the admin must start the login again and wait for a new code

**Endpoints:**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /auth/verify-login-device | Public | Verify the new-device code; issue full session on success |

**Error codes (added to `error-codes.ts` in Phase 1):**
- `AUTH_DEVICE_VERIFICATION_REQUIRED` — login response flag for the client to show the code input screen
- `AUTH_INVALID_CODE` — wrong code on the verify endpoint

**Admin panel (Phase 15):** the admin login screen consumes this API automatically — when the login response carries `requiresDeviceVerification: true`, the UI swaps to the 6-digit input form, keeps the email/password in state, and calls `POST /auth/verify-login-device` with the captured values. After success, the standard dashboard redirect runs.

**Rate limits:** the auth endpoint's 5 rpm IP limit from Section 13 already bounds code generation. The 5-attempt lock on the verify endpoint prevents code brute-forcing.

### 12. Input Sanitization

All user text inputs (topic title, content, comment, bio) are stripped of HTML tags on the backend. A simple `stripHtml()` helper function is sufficient — tags like `<script>`, `<img>`, etc. are removed. This is an additional security layer on top of React's default text escaping. Applied during validation via Zod pipe transform.

### 13. Rate Limiting

With `@nestjs/throttler`:

Uses `@nestjs/throttler` with `@nestjs/throttler-storage-redis` for distributed rate limiting. In-memory throttler doesn't work across multiple instances in production. Redis store shares rate limit counters across all backend instances.

- Global: 60 requests / minute (IP-based)
- Auth endpoints: 5 requests / minute (IP-based)
- Email delivery: 1 request / 60 seconds (user-based)

**Redis failure strategy (layered approach):**
- **Auth endpoints** (login, register, verify-email, forgot-password): **fail-closed** — if Redis is unavailable, return 503 Service Unavailable. Security takes priority; brute-force protection must not be bypassed.
- **All other endpoints**: **fail-open** — if Redis is unavailable, allow the request without rate limiting. Service availability takes priority for short Redis outages.
- Implementation: wrap the Redis throttler storage with a try-catch. On Redis connection error, check if the endpoint is in the auth fail-closed list.

### 14. CORS Configuration

CORS settings in `main.ts`:
- Origin: `[FRONTEND_URL, ADMIN_URL]` (array — allows both frontend and admin panel origins)
- Credentials: true
- Allowed headers: `Content-Type`, `Authorization`

## Security Checklist
- [ ] Passwords are hashed with bcrypt (salt rounds: 12)
- [ ] JWT secrets are sufficiently long (64+ characters)
- [ ] Refresh token uses a separate secret
- [ ] Email verification code is stored in Redis with TTL (no memory leak)
- [ ] Brute-force protection: rate limiting is active
- [ ] Verification code has an attempt limit (5 attempts)
- [ ] Google OAuth callback URL is correctly configured
- [ ] OAuth email conflict check is performed with LOCAL provider
- [ ] Password hash is never returned in responses
- [ ] Token payload contains no sensitive info (password, full name, etc.)
- [ ] Consent IP address is recorded
- [ ] Password reset token is single-use and time-limited
- [ ] Input sanitization is active on all text fields
- [ ] Email enumeration protection (forgot-password returns 200 in all cases)
- [ ] Every newly created account — email/password or Google OAuth — has a non-null `currentRankId` by the time its creation transaction commits

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# Test the following scenarios (Supertest or manual curl):

# 1. Registration
POST /api/v1/auth/register → 201, token returned
POST /api/v1/auth/register (same email) → 409 Conflict
POST /api/v1/auth/register (same username) → 409 Conflict
POST /api/v1/auth/register (12 years old) → 400 Bad Request
POST /api/v1/auth/register (invalid email) → 400 Bad Request
POST /api/v1/auth/register (weak password) → 400 Bad Request

# 2. Login
POST /api/v1/auth/login → 200, token returned
POST /api/v1/auth/login (wrong password) → 401
POST /api/v1/auth/login (non-existent email) → 401

# 3. Token refresh
POST /api/v1/auth/refresh → 200, new access token
POST /api/v1/auth/refresh (invalid token) → 401

# 4. Email verification
POST /api/v1/auth/verify-email → 200, email sent
POST /api/v1/auth/verify-email/confirm (correct code) → 200
POST /api/v1/auth/verify-email/confirm (wrong code) → 400
POST /api/v1/auth/verify-email/confirm (5x wrong) → 429

# 5. Rate limiting
6x POST /api/v1/auth/login (within 1 min) → 429 Too Many Requests

# 6. Consent
POST /api/v1/auth/accept-consent → 200
```

## Completion Criteria
- [ ] Registration and login with email + password works
- [ ] JWT access + refresh token mechanism works
- [ ] Google OAuth flow works (redirect to Google and callback)
- [ ] Profile completion endpoint works
- [ ] Email verification code is sent and confirmed
- [ ] Consent text approval mechanism works
- [ ] Rate limiting is active
- [ ] All validation errors return meaningful messages
- [ ] Shared schemas are used in the backend
- [ ] Unit tests pass
- [ ] API tests pass
