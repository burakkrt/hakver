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
- rank: `{ name: string, iconSlug: string, color: string }` — rank metadata consumed by the frontend `<RankBadge />` component (Phase 11 Section 11) to render an icon + color-tinted pill. Example: `{ "name": "Yargıç", "iconSlug": "gavel", "color": "indigo" }`
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
- age: number — integer years computed from `dateOfBirth` on the server (`floor((now - dob) / 365.25 days)`). Exposes the account holder's age as a community signal without leaking the exact birth date. The raw `dateOfBirth` itself is still withheld from public responses
- Rendered together as "Ahmet D., 28" on the profile page. The backend never returns the untruncated family name or the raw birth date on this endpoint.

**UserPrivateResponseSchema** (when viewing own profile — the owner may see their full data):
- UserPublicResponseSchema fields, but `lastName` is returned in full
- email, dateOfBirth, gender, emailVerifiedAt, usernameChangedAt, emailChangedAt, hasAcceptedConsent
- `age` is inherited from the public schema; the owner also receives the raw `dateOfBirth` so the settings form can pre-populate the birth-date field

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

### 2.2. `@SelfOnly` Decorator

Endpoints whose response is tied to the authenticated user (e.g., `GET /users/:username/votes`) must reject requests where the URL parameter does not match the caller. Hand-rolled inline checks drift over time; a single decorator is the authoritative implementation.

**`@SelfOnly(paramName: string)`** — reads the specified route parameter from the request, compares it case-insensitively against `request.user.username`, and:
- When the values match → proceed to the controller
- When the values do not match → short-circuit with `404 Not Found` using the relevant resource's NOT_FOUND envelope (e.g., `{ code: "USER_NOT_FOUND", message: "Kullanıcı bulunamadı" }`). Returning 404 (not 403) prevents existence leakage — a curious caller cannot confirm which usernames have activity on the platform
- Moderators and admins also receive 404. This endpoint family is scoped to the owner only; admin and moderator surfaces use the dedicated admin endpoints (Phase 15) when they need the same data for moderation review. The self-only rule is deliberately strict so the same decorator protects future self-scoped endpoints without having to enumerate role exceptions

Apply to:
- `GET /users/:username/votes` — initial self-only endpoint in this phase
- Any future endpoint whose response body reflects the authenticated user's private signal surface (own activity feeds, private bookmark variants, etc.)

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
| GET | /users/:username/votes | JWT + `@SelfOnly('username')` (Section 2.2) | Topics the user voted on — returns 404 USER_NOT_FOUND for any caller other than the owner, including moderators and admins, to avoid existence leakage |
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
- age: integer years (computed server-side from `dateOfBirth`) — shown on the profile as a single number
- email: not returned
- dateOfBirth: not returned — only the computed `age` is exposed so the exact birth date is never leaked
- gender: not returned

**When viewing own profile (`GET /users/me` → UserPrivateResponseSchema):**
- All fields are returned in full, including the owner's full surname

**When representing a user alongside content (topic cards, comment cards, notification actors — UserPublicCardSchema):**
- id, username, avatar (url), rank (name, iconSlug, color), totalXp, isDeletedAuthor only
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
  - `avatar.url` → the system "Silinmiş Kullanıcı" placeholder avatar (`category: "system"`, maintained in the Avatar seed list and never selectable by users)
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
   - Call `invalidateOtherSessions(userId, { reason: "email-change" })` (Phase 3 Section 11.1) so any other device holding a session tied to the previous email is forced back to login on its next refresh. The current request's response carries the rotated access token and refresh cookie from the helper, so the caller stays logged in seamlessly
5. If no → 400 error

### 8. Password Change

1. `currentPassword` is verified with bcrypt
2. `newPassword` is hashed and updated
3. OAuth users (passwordHash null) cannot change password → 400 error
4. Call `invalidateOtherSessions(userId, { reason: "password-change" })` (Phase 3 Section 11.1) — increments `User.sessionVersion` so every other active session is cut at its next refresh. The caller receives a freshly rotated token pair in the same response and remains signed in on the current device

### 9. Avatar Change and Rank-Locked Avatar Gate

`PATCH /users/me/avatar`:

1. Request: `{ avatarId: UUID }`
2. Load the target avatar. Reject when any of the following fails:
   - `isActive === true`
   - `category !== 'system'` (system-category avatars are never selectable by users) — 403 `{ code: "AVATAR_LOCKED", message: "Bu avatar seçilemez" }`
3. If `requiredRankId` is null (free pool) → proceed to step 5
4. If `requiredRankId` is populated:
   - Load the referenced `Rank`
   - Compute `gateValue = max(user.totalXp, user.highestXpReached)` — uses the monotonic watermark (Phase 2 `User.highestXpReached` field) so an admin SUBTRACT/SET or reconciliation drift never re-locks a previously earned avatar
   - If `gateValue < rank.minXp` → 403 `{ code: "AVATAR_LOCKED", message: "Bu avatar [Rank.name] rütbesine ulaşınca seçilebilir" }`
5. Update `user.avatarId`
6. Response: updated private profile response

### 9.1. `GET /avatars` Response Shape

The endpoint returns every avatar where `category IN ('free', 'rank-locked')` and `isActive = true`. System-category avatars are filtered out. Each row is hydrated with a per-viewer lock flag:

```json
{
  "data": [
    {
      "id": "...",
      "name": "Anonim Penguen",
      "url": "/avatars/anonim-penguen.svg",
      "category": "free",
      "isLocked": false,
      "requiredRank": null
    },
    {
      "id": "...",
      "name": "Yargıç Baykuşu",
      "url": "/avatars/yargic-baykusu.svg",
      "category": "rank-locked",
      "isLocked": true,
      "requiredRank": { "name": "Yargıç", "minXp": 1500, "iconSlug": "gavel", "color": "indigo" }
    }
  ]
}
```

- `isLocked` is computed per request from the viewer's `max(totalXp, highestXpReached)`. Unauthenticated callers receive `isLocked: true` for every `rank-locked` row so anonymous grid previews remain truthful
- The catalogue is bounded at 39 rows (15 free + 24 rank-locked), fits in a single payload — no pagination
- Cache key: `@CacheTTL(3600)` scoped by viewer tier, e.g., `avatars:v1:{rankId}` or `avatars:v1:anonymous`. Admin avatar or rank edits invalidate every scope via tag-based or prefix-based invalidation
- The frontend consumes this response to render the avatar grid in profile settings (Phase 12 Section 10)

### 10. Account Deletion (KVKK Anonymization + Cold Archive)

When `DELETE /users/me` is called the account is anonymized immediately and its lingering personal artefacts are scheduled for permanent removal. The content the account produced (topics, votes, comments) is preserved behind an anonymous presentation so the community record stays intact while the owner's identity disappears from the product surface. Everything in this section runs inside a single `prisma.$transaction` so either the account is fully retired or nothing changes.

**Stage 1 — Anonymize on the spot (synchronous, inside the delete request):**

1. User verifies their password (or confirms with a flag for OAuth users). The confirmation flow stays Turkish and warns that the action cannot be reversed
2. Personal data on the `User` row is anonymized:
   - `firstName = "Silinmiş"`
   - `lastName = "Kullanıcı"`
   - `email = "deleted_{uuid}@hakver.local"`
   - `username = "deleted_{uuid}"` — never rendered to other users; card responses substitute the fixed "Silinmiş Kullanıcı" label (see Phase 4 Section 5.1)
   - `bio = null`
   - `passwordHash = null`
   - `providerId = null`
   - `avatarId` kept as a reference to the dedicated "silinmis" avatar in the Avatar seed (card responses render this avatar)
   - `dateOfBirth` → set to epoch
   - `totalXp = 0`, `currentRankId = null` — XP and rank no longer surface on cards for this account
3. `deletedAt = now()` is set. The `User.deletedAt IS NOT NULL` predicate is the authoritative "account retired" flag used by every other module
4. **Owner-side artefact cleanup (delete within the same transaction — rows this user owns that no longer make sense after retirement):**
   - `TopicBookmark` where `userId = deleted user`
   - `NotificationMute` where `userId = deleted user`
   - `UserBlock` where `blockerId = deleted user` (rows where this user was blocking others)
   - `UserNotificationPreference` where `userId = deleted user`
   - `NotificationActor` rows authored by the deleted user are left in place so aggregated cards keep their structure; card hydration swaps the account for the "Silinmiş Kullanıcı" presentation (see below)
   - `Notification` rows whose recipient is the deleted user are deleted (no need to notify a retired account)
   - Pending `/users/me/data-export` jobs authored by the deleted user are cancelled
5. **Other-side artefact preservation (untouched):**
   - `UserBlock` where `blockedId = deleted user` (rows where other users blocked this account) — preserved so the other user's intent is honoured even if the target later no longer "exists" from the product's perspective. These rows render in the blocklist UI with the "Silinmiş Kullanıcı" card + a disabled "Engeli Kaldır" button (the other user can still click to remove the block manually, but the card itself carries no profile link)
   - `AnonymousIdentity` rows — preserved so anonymous topics/comments keep their "Anonim Penguen #4521" label. The unique `(userId, topicId)` constraint stays valid because the retired account cannot produce new rows
   - `Vote` rows — preserved so topic vote counts remain accurate
   - `Topic` / `Comment` rows — preserved; all previously non-anonymous posts are retroactively rendered as anonymous "Silinmiş Kullanıcı" cards (see Phase 4 Section 5.1 for the `isDeletedAuthor` flag propagation rule)
   - `Report` rows (against the deleted user) — preserved for moderator auditability
6. All active sessions of the deleted account are invalidated (future: token blacklist; for MVP the refresh token's cookie is cleared by the response and the short-lived access token expires within 15 minutes)
7. The account transition is written to `ActivityLog` (`action: "account:delete"`, category: security, 1-year retention)

**Stage 2 — Cold archive + full personal-data removal (post-MVP cron, placeholder now):**

A scheduled job will eventually carry retired accounts through their KVKK-compliant final step: the residual FK on content rows (`Topic.authorId`, `Comment.authorId`, `Vote.userId`, `NotificationActor.actorId`, etc.) is copied into a detached archive database (no linkage back to `User.id`) and the primary `User` row is hard-deleted. The archive rows carry only the fields required to keep anonymous presentation working (for example `Topic.authorDisplay = "Silinmiş Kullanıcı"`, `Vote.topicId`, `AnonymousIdentity.displayName`), never the former personal data. After this step:

- The hot database no longer contains any row that can reconstruct the deleted user's identity
- Listings and content continue to render with the "Silinmiş Kullanıcı" anonymous card, now sourced from the archive mirror
- Any URL targeting the former username (`/users/deleted_{uuid}`) responds with `ProfileUnavailableResponseSchema { accountUnavailable: true, reason: "DELETED" }` with `<meta name="robots" content="noindex">` on the page

This cold-archive pipeline is **not part of MVP scope** — Stage 1 alone is sufficient for launch because the anonymization completely removes personal surfaces from the product. The Stage 2 cron is tracked as a KVKK follow-up in `docs/post-mvp.md` (see "Deleted-Account Cold Archive Pipeline"). Phase 4 references the eventual existence of this pipeline so the schema and the card hydration rule are designed to survive the transition without a second migration.

**Anonymous card enforcement on listings (MVP-visible):**
- Every endpoint that returns a `UserPublicCardSchema` calls the deleted-author propagation rule (Phase 4 Section 5.1) and substitutes: fixed label "Silinmiş Kullanıcı", default "silinmis" avatar, rank omitted, `totalXp: 0`, card non-clickable (no link to a profile, bookmarks page, or any other surface). Any card UI component treats `isDeletedAuthor === true` as an instruction to render the locked-down anonymous presentation
- The `username` surfaced by card responses for deleted accounts is **always** the constant string "Silinmiş Kullanıcı" — the raw `deleted_{uuid}` form is never leaked to the client. This prevents both accidental display and URL probing
- Security surfaces: attempting to access `/users/deleted_{uuid}`, `/users/deleted_{uuid}/topics`, `/users/deleted_{uuid}/comments`, or `/users/deleted_{uuid}/votes` returns `ProfileUnavailableResponseSchema` with `reason: "DELETED"` (the underlying username is not echoed back, so a probe cannot confirm existence)

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
- [ ] `PATCH /users/me/avatar` rejects rank-locked avatars whose `requiredRank.minXp` exceeds `max(user.totalXp, user.highestXpReached)` with 403 `AVATAR_LOCKED`
- [ ] `category: "system"` avatars cannot be selected via `PATCH /users/me/avatar` under any circumstance
- [ ] `GET /avatars` filters out `category: "system"` rows and hydrates `isLocked` using the viewer's monotonic XP watermark

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
GET /api/v1/topics → each topic.author contains { id, username, avatar: { url }, rank: { name, iconSlug, color }, totalXp, isDeletedAuthor } only

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
GET /api/v1/avatars → 200, 39 rows (15 free + 24 rank-locked); system avatars excluded
# Each row carries isLocked flag; unauthenticated viewer sees every rank-locked row as isLocked: true
PATCH /api/v1/users/me/avatar { avatarId: "<free-avatar>" } → 200
PATCH /api/v1/users/me/avatar { avatarId: "<locked-above-current-rank>" } → 403 AVATAR_LOCKED
PATCH /api/v1/users/me/avatar { avatarId: "<system-avatar-id>" } → 403 AVATAR_LOCKED
# After user.highestXpReached crosses the avatar's rank threshold once, subsequent reselection succeeds even if totalXp has been reduced

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
- [ ] Avatar selection works, including rank-lock gate (AVATAR_LOCKED for premature selection, system avatar always rejected, `highestXpReached` preserves access after XP reductions)
- [ ] `GET /avatars` returns `isLocked` and `requiredRank` per row and filters out system avatars
- [ ] Account deletion with anonymization works
- [ ] All tests pass
