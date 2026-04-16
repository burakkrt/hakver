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
- Payload: `{ sub: userId, email, username, isVerified: boolean }`
- Duration: `JWT_ACCESS_EXPIRATION` (15m)
- Secret: `JWT_SECRET`

**Refresh Token:**
- Payload: `{ sub: userId, tokenVersion: number }`
- Duration: `JWT_REFRESH_EXPIRATION` (7d)
- Secret: `JWT_REFRESH_SECRET`
- `tokenVersion` is not stored in the database initially — simple stateless refresh. Token revocation can be added in the future.

**Token Return Format:**
```json
{
  "accessToken": "...",
  "refreshToken": "...",
  "user": { "id", "email", "username", "isVerified", "firstName", "lastName" }
}
```

### 4. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /auth/register | Public | Register with email + password |
| POST | /auth/login | Public | Login with email + password |
| POST | /auth/refresh | Public | Get new access token with refresh token |
| POST | /auth/verify-email | JWT | Send email verification code |
| POST | /auth/verify-email/confirm | JWT | Confirm verification code |
| GET | /auth/google | Public | Initiate Google OAuth |
| GET | /auth/google/callback | Public | Google OAuth callback |
| POST | /auth/complete-profile | JWT | Complete profile after OAuth |
| POST | /auth/accept-consent | JWT | Approve consent text |
| POST | /auth/logout | JWT | Logout (client-side token cleanup) |
| POST | /auth/forgot-password | Public | Send password reset email |
| POST | /auth/reset-password | Public | Reset password (with token) |

### 5. Registration Flow (Email + Password)

1. Client sends data with `RegisterSchema`
2. Server-side validation (Zod)
3. Email and username uniqueness check
4. Password is hashed with bcrypt (salt rounds: 12)
5. User is created (`emailVerifiedAt: null`, `provider: LOCAL`)
6. Default avatar is assigned (the avatar with `isDefault: true`)
7. USER role is assigned
8. 6-digit verification code is generated and stored in Redis (TTL: 10 minutes, key: `email-verify:{userId}`)
9. Verification code is sent to the user's email via Resend (React Email template)
10. Access + refresh tokens are returned
11. **Post-registration redirect order:** Register → Consent screen → Email verification. Consent approval must happen before email verification. The client redirects to the consent screen first, then to email verification.

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
4. Generate random reset token, store in Redis (TTL: 30 minutes, key: `password-reset:{token}`)
5. Send password reset link via email: `FRONTEND_URL/reset-password?token={token}`
6. Rate limit: 1 request / 60 seconds (email-based)

`POST /auth/reset-password`:
1. Token and new password are received
2. Check if token exists in Redis
3. New password validation (same rules as RegisterSchema)
4. Hash password with bcrypt, update
5. Delete token from Redis
6. Response: 200

**Email template:** `password-reset.tsx` — Password reset link, expiration time info.

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
