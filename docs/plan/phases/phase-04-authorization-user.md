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

**UserPublicResponseSchema** (data returned when viewing another user):
- id, username, bio, avatarUrl, rankName, totalXp
- firstName → first character only + "." (e.g.: "B.")
- lastName → first character only + "." (e.g.: "K.")
- createdAt

**UserPrivateResponseSchema** (when viewing own profile):
- UserPublicResponseSchema + firstName (full), lastName (full), email, dateOfBirth, gender, emailVerifiedAt, usernameChangedAt, emailChangedAt

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
| GET | /avatars | Public | Available avatar list |

### 5. Profile Viewing Privacy Rules

**When viewing another user:**
- firstName → first character + "." (e.g.: "Burak" → "B.")
- lastName → first character + "." (e.g.: "Kurt" → "K.")
- email: not returned
- dateOfBirth: not returned
- gender: not returned

**When viewing own profile:**
- All fields are returned in full

This transformation is done **in the service layer**, not with Prisma select. This way business logic stays in one place.

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
4. Topics and comments created by the user are preserved but displayed with a "deleted user" profile card
5. All active sessions are invalidated (future: token blacklist)

### 11. Username Availability Check

`GET /users/check-username/:username` — Public endpoint. Checks if a username is available. Response: `{ available: boolean }`. Rate limited (10 requests/minute).

## Security Checklist
- [ ] Permission guards are active on all protected endpoints
- [ ] Profile privacy rules are applied in the service layer
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
GET /api/v1/users/me → 200, all fields (private)
GET /api/v1/users/testuser → 200, name-surname masked (public)
GET /api/v1/users/nonexistent → 404

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
- [ ] Restriction guard infrastructure is ready (to be used in Phase 9)
- [ ] Profile viewing privacy rules are applied
- [ ] Username and email change limits work
- [ ] Password change works
- [ ] Avatar selection works
- [ ] Account deletion with anonymization works
- [ ] All tests pass
