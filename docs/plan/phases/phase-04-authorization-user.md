# Phase 4: Authorization & User Management

## Dependencies
- Phase 3 must be completed (auth module, JWT, user creation)

## Goals
- RBAC + Permission-based authorization guards
- User profile CRUD (viewing, editing, privacy rules)
- Avatar system
- Username and email change limits
- Account deletion (KVKK — anonymization)
- Password change
- Define user DTO schemas in the shared package

## Tasks

### 1. Shared Zod Schemas (packages/shared)

`packages/shared/src/schemas/user.ts`:

**UpdateProfileSchema:**
- firstName: min 2, max 50 (optional)
- lastName: min 2, max 50 (optional)
- dateOfBirth: ISO date, 13+ years (optional)
- bio: max 500 characters (optional)

**ChangeUsernameSchema:**
- username: min 3, max 30, letters/digits/underscores

**ChangeEmailSchema:**
- newEmail: email format
- password: string (for verification)

**ChangePasswordSchema:**
- currentPassword: string
- newPassword: same rules as RegisterSchema

**UserPublicCardSchema** (minimum author card — used in topic lists, topic details, comment lists, notification actors, vote lists, anywhere a user is represented alongside content):
- id: UUID
- username: string
- avatar: `{ url: string }`
- rank: `{ name: string }` — the rank display name (e.g., "Jüri Üyesi")
- totalXp: number
- isDeletedAuthor: boolean — `true` when the referenced user is soft-deleted; frontend uses this to render an "Silinmiş Kullanıcı" styled anonymous card instead of the real username/avatar

This schema intentionally excludes firstName and lastName. Full name is never shown next to content — the username is the primary public identifier across the platform. The card carries avatar, username, rank name, and total XP so gamification signals (rank badge, XP count) remain visible everywhere a user appears.

**UserPublicResponseSchema** (data returned when viewing another user's full profile):
- UserPublicCardSchema fields
- bio: string | null
- totalXp: number
- createdAt: ISO date string
- firstName: string (full — e.g.: "Ahmet")
- lastName: string (first character + "." — e.g.: "Demir" → "D.")
- Rendered together as "Ahmet D." on the profile page. The backend never returns the untruncated family name on this endpoint.

**UserPrivateResponseSchema** (when viewing own profile — the owner may see their full data):
- UserPublicResponseSchema fields, but `lastName` is returned in full
- email, dateOfBirth, gender, emailVerifiedAt, usernameChangedAt, emailChangedAt, hasAcceptedConsent

**UserAdminResponseSchema** (admin-only endpoints — the admin dashboard needs the full legal name for compliance and moderation):
- UserPrivateResponseSchema fields (target user's full firstName and lastName)
- role, active restrictions, recent XP logs, recent reports (populated by admin endpoints that return this shape)
- Never exposed through public or self endpoints — only admin routes. Moderator routes receive `UserPublicResponseSchema` instead (masked surname, no full legal name)

**ProfileUnavailableResponseSchema** (returned by `GET /users/:username` when the profile cannot be shown):
- accountUnavailable: `true`
- reason: `"DELETED" | "UNAVAILABLE"`
  - `DELETED` → the target user is soft-deleted (`User.deletedAt IS NOT NULL`). Frontend renders "Bu kullanıcı hesabını silmiştir" info page
  - `UNAVAILABLE` → a block relationship exists between the viewer and the target in either direction, OR the profile is otherwise restricted. Frontend renders a generic "Bu kullanıcı profili görüntülenemiyor" info page with no mention of blocking (prevents the blocked party from discovering they are blocked)

Both variants share the same page layout; only the heading and help copy differ. The profile page never uses different HTTP status codes to distinguish these cases — 200 with the structured flag keeps the frontend behaviour uniform and avoids leaking block signals via 403 vs 404 timing.

### 2. Permission Guard System

Under `apps/api/src/common/guards/`:

**`roles.guard.ts`** — Works with the `@Roles('ADMIN', 'MODERATOR')` decorator. Gets userId from JWT, checks the user's roles.

**`permissions.guard.ts`** — Works with the `@RequirePermissions('topic:delete', 'user:restrict')` decorator. Checks the permissions that the user's roles have.

**`verified.guard.ts`** — For endpoints that require email verification. Checks `emailVerifiedAt !== null`.

**`restriction.guard.ts`** — Checks whether the user has an active restriction. Checks the restricted action based on the endpoint. Expired restrictions are not considered valid.

**Decorators:**
- `@Public()` — No auth required
- `@Roles(...roles)` — Specific roles
- `@RequirePermissions(...permissions)` — Specific permissions
- `@RequireVerified()` — Email verification required
- `@CurrentUser()` — User object from request

### 2.1. Profile Completion Guard

**`profile-complete.guard.ts`** — Checks that the user's profile is complete before allowing write actions. Required fields: `username`, `firstName`, `lastName`.

- Applied to all endpoints that create or modify content (topic create, comment create, vote create)
- Returns 403 with `{ code: "USER_PROFILE_INCOMPLETE", message: "Devam etmek için profilinizi tamamlayın", requiredFields: ["username", "firstName", "lastName"] }` if any required field is null/empty
- Works alongside existing guards: JWT → Verified → ProfileComplete → Restriction → Permission
- Decorator: `@RequireCompleteProfile()` — applied on write endpoints

This is critical for OAuth users who may register with incomplete data (Google OAuth provides email but username is set later via profile completion).

### 3. User Module

Under `apps/api/src/modules/user/`:

```
user/
├── user.module.ts
├── user.controller.ts
├── user.service.ts
└── dto/
```

### 4. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /users/me | JWT | Own profile info (private response) |
| PATCH | /users/me | JWT + Verified | Update profile (name, surname, date of birth, bio) |
| PATCH | /users/me/username | JWT + Verified | Change username (30-day restriction) |
| PATCH | /users/me/email | JWT + Verified | Change email (30-day restriction, re-verification) |
| PATCH | /users/me/password | JWT + Verified | Change password |
| PATCH | /users/me/avatar | JWT + Verified | Change avatar |
| DELETE | /users/me | JWT + Verified | Delete account (KVKK anonymization) |
| GET | /users/:username | Public | User profile (public response) |
| GET | /users/:username/topics | Public | Topics created by the user |
| GET | /users/:username/votes | JWT (self only) | Topics the user voted on |
| GET | /users/:username/comments | Public | Topics the user commented on |
| GET | /users/me/profile-status | JWT | Profile completion status (which fields are missing) |
| GET | /avatars | Public | Available avatar list (cached — `@CacheTTL(3600)`, invalidate on admin avatar changes) |
| POST | /users/me/data-export | JWT + Verified | Request personal data export (KVKK) |

### 5. Profile Viewing Privacy Rules

**`GET /users/:username` response branching** (determined by the service layer before field masking):
1. Target user is soft-deleted → return `ProfileUnavailableResponseSchema` with `reason: "DELETED"`
2. A block relationship exists between viewer and target (either direction) → return `ProfileUnavailableResponseSchema` with `reason: "UNAVAILABLE"`
3. Viewer is the target (viewing own profile) → return `UserPrivateResponseSchema`
4. Viewer is an admin → return `UserAdminResponseSchema` (audited via `admin:user-view` ActivityLog, see Phase 15)
5. Otherwise (normal public view) → return `UserPublicResponseSchema`

**When viewing another user's profile (`GET /users/:username` → UserPublicResponseSchema):**
- firstName: full (e.g.: "Ahmet") — the first name is not secret; only the family name is abbreviated for identity protection
- lastName: first character + "." (e.g.: "Demir" → "D.") — rendered together as "Ahmet D."
- email: not returned
- dateOfBirth: not returned
- gender: not returned

**When viewing own profile (`GET /users/me` → UserPrivateResponseSchema):**
- All fields are returned in full, including the owner's full surname

**When representing a user alongside content (topic cards, comment cards, notification actors — UserPublicCardSchema):**
- id, username, avatar (url), rank (name), totalXp, isDeletedAuthor only
- firstName and lastName are NOT returned on card responses. Any list, feed, or WebSocket event that includes a user reference must use the card schema, not the profile schema
- This is a security-critical rule: the family name must not leak into feeds, notifications, search results, or any non-profile response

**When accessed by an admin or moderator (UserAdminResponseSchema on admin endpoints):**
- Full firstName and lastName are returned for moderation/compliance review
- These endpoints require the admin role and are audited via the activity log

All masking transformations are performed **in the service / DTO mapping layer**, not with Prisma `select`. This keeps the business rule in a single place and prevents accidental leakage when a new endpoint is added.

### 5.1. Deleted-Author Propagation on Card Responses

Every endpoint that returns a `UserPublicCardSchema` populates `isDeletedAuthor` based on `User.deletedAt`:
- Active user → `isDeletedAuthor: false`, real username / avatar / rank / totalXp shown
- Soft-deleted user → `isDeletedAuthor: true`. The backend still returns a schema-compliant card, but substitutes the display fields so no leaked username or avatar appears:
  - `username` → `"Silinmiş Kullanıcı"` (constant label; the raw `deleted_{uuid}` username is never returned to clients)
  - `avatar.url` → the default "silinmis" avatar (maintained in the Avatar seed list)
  - `rank.name` → `null`-replaced with an empty string, or the card simply omits the rank badge — frontend hides rank when `isDeletedAuthor` is `true`
  - `totalXp` → `0` (deleted accounts do not expose XP)

Deleted users do NOT receive notifications, do NOT trigger fan-out, and do NOT appear in search, trending, or bookmark listings. See Phase 10 fan-out rules for the recipient-side filter.

### 6. Username Change

1. New username is validated with `ChangeUsernameSchema`
2. Uniqueness check
3. `usernameChangedAt` is checked: has 30 days passed since the last change?
4. If yes → update, set `usernameChangedAt = now()`
5. If no → 400 "Kullanıcı adınızı {tarih} tarihinden önce değiştiremezsiniz"

### 7. Email Change

1. Current password is verified
2. New email uniqueness check
3. `emailChangedAt` is checked: has 30 days passed since the last change?
4. If yes:
   - Email is updated
   - `emailVerifiedAt = null` is set (re-verification required)
   - `emailChangedAt = now()`
   - Verification code is sent to the new email
5. If no → 400 error

### 8. Password Change

1. `currentPassword` is verified with bcrypt
2. `newPassword` is hashed and updated
3. OAuth users (passwordHash null) cannot change password → 400 error

### 9. Avatar Change

1. Request: `{ avatarId: UUID }`
2. Check that the avatar has `isActive: true`
3. Update the user's `avatarId` field

### 10. Account Deletion (KVKK Anonymization)

When `DELETE /users/me` is called:

1. User verifies their password (or confirmation flag for OAuth users)
2. Personal data is anonymized:
   - `firstName = "Silinmiş"`
   - `lastName = "Kullanıcı"`
   - `email = "deleted_{uuid}@hakver.local"`
   - `username = "deleted_{uuid}"`
   - `bio = null`
   - `passwordHash = null`
   - `providerId = null`
   - `dateOfBirth` → set to epoch date
3. `deletedAt = now()` is set
4. Topics and comments created by the user are preserved but displayed with an anonymous "Silinmiş Kullanıcı" profile card. All non-anonymous topics and comments by this user are retroactively displayed as anonymous (author info hidden). Anonymous content continues to show its existing AnonymousIdentity (e.g., "Anonim Penguen #4521").
5. `AnonymousIdentity` records are preserved — they reference the anonymized user via FK. Since the user is soft-deleted, the unique constraint `(userId, topicId)` will not be violated (deleted users cannot create new content).
6. Votes by the deleted user are preserved (vote counts remain accurate).
7. All active sessions are invalidated (future: token blacklist)

### 11. KVKK Personal Data Export

`POST /users/me/data-export`:
1. Rate limit: 1 request per 7 days per user
2. Create a BullMQ job to compile the user's personal data:
   - Profile info (name, email, username, dateOfBirth, gender, bio)
   - Topics created (title, content, dates)
   - Comments written (content, dates)
   - Votes cast (topic, type, dates)
   - XP history (action, points, dates)
   - Likes given/received
   - Block list
   - Consent records
3. Generate a JSON file with all data
4. Send download link via email (Resend) — link valid for 24 hours
5. Response: 202 Accepted `{ message: "Verileriniz hazırlanıyor, email adresinize gönderilecek" }`
6. Email template: `data-export.tsx` — download link + expiration info

**Security:** Download link uses a signed JWT token (24-hour expiry). File is temporarily stored and auto-deleted after download or expiry.

### 12. Username Availability Check

`GET /users/check-username/:username` — Public endpoint. Checks if a username is available. Response: `{ available: boolean }`. Rate limited (10 requests/minute).

## Security Checklist
- [ ] Permission guards are active on all protected endpoints
- [ ] Profile privacy rules are applied in the service layer
- [ ] Card responses (topic, comment, notification, vote lists) use `UserPublicCardSchema` and do NOT return firstName or lastName
- [ ] Profile responses for other users return full firstName but abbreviate lastName to a single initial
- [ ] Only admin/moderator endpoints return the full legal name; this access is audit-logged
- [ ] User can only edit their own profile (userId match)
- [ ] Current password is verified on password change
- [ ] Current password is verified on email change
- [ ] Data is anonymized on account deletion, no hard delete
- [ ] Password hash is never returned in any response
- [ ] Users with `deletedAt` cannot log in
- [ ] Username check endpoint is rate limited

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# API tests:

# 1. Profile viewing
GET /api/v1/users/me → 200, all fields (private), full lastName
GET /api/v1/users/testuser → 200, firstName full + lastName initial (e.g. "Ahmet D."), no email/dateOfBirth/gender
GET /api/v1/users/nonexistent → 404
# Card shape: any list endpoint's author object has no firstName/lastName keys
GET /api/v1/topics → each topic.author contains { id, username, avatar: { url }, rank: { name }, totalXp, isDeletedAuthor } only

# 2. Profile update
PATCH /api/v1/users/me { bio: "test" } → 200
PATCH /api/v1/users/me (unverified user) → 403

# 3. Username change
PATCH /api/v1/users/me/username { username: "yeni" } → 200
PATCH /api/v1/users/me/username (before 30 days) → 400
PATCH /api/v1/users/me/username (taken username) → 409

# 4. Permission guard
DELETE /api/v1/topics/:id (USER role) → 403
DELETE /api/v1/topics/:id (MODERATOR role) → 200

# 5. Account deletion
DELETE /api/v1/users/me → 200
GET /api/v1/users/me (with deleted account token) → 401

# 6. Avatar
GET /api/v1/avatars → 200, avatar list
PATCH /api/v1/users/me/avatar { avatarId: "..." } → 200

# 7. Username availability
GET /api/v1/users/check-username/test → { available: true/false }
```

## Completion Criteria
- [ ] RBAC + Permission guards work
- [ ] Verified guard works (unverified users are blocked)
- [ ] ProfileComplete guard works (incomplete profiles are blocked from write actions)
- [ ] Restriction guard infrastructure is ready (to be used in Phase 9)
- [ ] Profile viewing privacy rules are applied (firstName full, lastName initial on other-user profiles; card responses expose no name fields)
- [ ] UserPublicCardSchema, UserPublicResponseSchema, UserPrivateResponseSchema, and UserAdminResponseSchema are all defined in `@hakver/shared` and used by the right endpoints
- [ ] Username and email change limits work
- [ ] Password change works
- [ ] Avatar selection works
- [ ] Account deletion with anonymization works
- [ ] All tests pass
