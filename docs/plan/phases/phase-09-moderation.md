# Phase 9: Moderation & Social Safety

## Dependencies
- Phase 4, Phase 5 (topics), and Phase 7 (comments) must be completed

## Goals
- User restriction system (categorized, with reasons, time-limited)
- Restriction guard active on all protected endpoints
- User reporting system (topic, comment, user)
- User-to-user blocking with retroactive filters on topic and comment listing

## Tasks

### 1. Shared Zod Schemas

`packages/shared/src/schemas/moderation.ts`:

**ApplyRestrictionSchema:**
- userId: UUID
- restrictedAction: string (topic:create, comment:create, vote:create)
- categoryId: UUID
- reasonId: UUID
- expiresAt: ISO date (must be in the future)
- note: max 500 (optional)

**CreateReportSchema:**
- targetType: "TOPIC" | "COMMENT" | "USER"
- targetId: UUID
- categoryId: UUID
- description: max 500 (optional)

**ReviewReportSchema:**
- status: "RESOLVED" | "DISMISSED"

### 2. Moderation Module

Under `apps/api/src/modules/moderation/`:

```
moderation/
├── moderation.module.ts
├── moderation.controller.ts
├── moderation.service.ts
├── restriction/
│   ├── restriction.controller.ts
│   └── restriction.service.ts
├── report/
│   ├── report.controller.ts
│   └── report.service.ts
└── block/
    ├── block.controller.ts
    └── block.service.ts
```

### 3. Restriction System

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /moderation/restrictions | Moderator+ | Apply restriction |
| GET | /moderation/restrictions | Moderator+ | Active restrictions list |
| DELETE | /moderation/restrictions/:id | Admin | Remove restriction early |
| GET | /users/:username/restrictions | Public | User's active restrictions |
| GET | /moderation/restriction-categories | Public | Restriction categories and reasons |

**Restriction application flow:**
1. Validation with `ApplyRestrictionSchema`
2. Check if target user exists
3. Is the target user an admin or moderator? → 403 "Yöneticilere kısıtlama uygulanamaz"
4. Check if category and reason exist and are active
5. Check if `expiresAt` is in the future
6. Is there already an active restriction for the same action on the same user? If yes → 409
7. Create `UserRestriction` record
8. Save activity log

**Restriction check (RestrictionGuard):**
The guard whose infrastructure was set up in Phase 4 is activated here. On every protected endpoint:
1. Does the user have an active restriction for the relevant action?
2. Query in `UserRestriction` table: `userId, restrictedAction, expiresAt > now()`
3. If yes → 403 `{ code: "RESTRICTED", restriction: { category, reason, expiresAt } }` — for the client to show a meaningful message

**Restriction info on profile:**
`GET /users/:username/restrictions` → active restrictions (unexpired ones):
```json
[
  {
    "restrictedAction": "comment:create",
    "category": "Uygunsuz İçerik",
    "reason": "Hakaret veya küfür içeren paylaşım",
    "expiresAt": "2025-05-01T00:00:00Z"
  }
]
```

### 4. Reporting System

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /reports | JWT + Verified | Create report |
| GET | /reports | Moderator+ | Report list (filterable) |
| PATCH | /reports/:id | Moderator+ | Review report |

**Report creation:**
1. Validation with `CreateReportSchema`
2. Check if target (topic/comment/user) exists
3. Has the same user already reported the same target? → 409 "Bu içeriği zaten rapor ettiniz"
4. Self-report check (if targetType is USER) → 400
5. Create `Report` record (status: PENDING)

**Rate limiting:** Report creation endpoint (`POST /reports`) is rate limited to 5 requests per minute per user to prevent spam reports and moderator workload abuse.

**Report listing (moderator):**
- Filtering: status (PENDING, REVIEWED, RESOLVED, DISMISSED), targetType
- Sorting: createdAt DESC
- Pagination

**Report review:**
1. Validation with `ReviewReportSchema`
2. Check if report is in PENDING status
3. Update `status`, `reviewedById`, `reviewedAt`

### 5. Blocking System

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /users/:username/block | JWT + Verified | Block user |
| DELETE | /users/:username/block | JWT + Verified | Unblock user |
| GET | /users/me/blocked | JWT | Users I've blocked |

**Blocking:**
1. Check if target user exists
2. Self-block → 400
3. Unique constraint check
4. Create `UserBlock` record

**Effects of blocking (full blocking policy — retroactive to all existing modules):**
Blocking means "I don't want to see or interact with this person at all." The following filters must work:
- Topic listing: blocked users' topics are completely excluded
- Topic detail: blocked user's topic → 404 Not Found (as if it doesn't exist)
- Comment listing: blocked users' comments are excluded (except anonymous comments — identity is unknown)
- Voting: cannot vote on blocked user's topic (topic returns 404, so voting endpoint is unreachable)
- Commenting: cannot comment on blocked user's topic (topic returns 404)
- Profile: blocked user's profile → 404 Not Found
- Notifications: no notifications from blocked users (already defined in Phase 10)

This is a symmetric check: if User A blocks User B, then User A cannot see User B's content AND User B cannot see User A's content. This prevents harassment scenarios.

**BlockService** helper method:
`isBlocked(userId, targetUserId): boolean` — Other modules use this service to check block status.

### 6. Admin Restriction Category Management

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /admin/restriction-categories | Admin | Create new category |
| PATCH | /admin/restriction-categories/:id | Admin | Update category |
| POST | /admin/restriction-reasons | Admin | Create new reason |
| PATCH | /admin/restriction-reasons/:id | Admin | Update reason |

## Security Checklist
- [ ] Restrictions cannot be applied to moderators/admins
- [ ] Restriction check is active on all relevant endpoints
- [ ] Expired restrictions do not cause blocking
- [ ] User cannot report the same content multiple times
- [ ] Report review is moderator/admin only
- [ ] Block filters work correctly
- [ ] Admin endpoints are only open to the ADMIN role
- [ ] Restriction application is moderator+ only

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# API tests:

# 1. Applying restriction
POST /moderation/restrictions (moderator) → 201
POST /moderation/restrictions (normal user) → 403
POST /moderation/restrictions (restrict an admin) → 403

# 2. Restriction check
# Apply comment:create restriction to a user
POST /topics/:id/comments (restricted user) → 403, restriction details
# After time expires:
POST /topics/:id/comments (same user) → 201

# 3. Reporting
POST /reports { targetType: "TOPIC", targetId: "..." } → 201
POST /reports (same target, same user) → 409
POST /reports (self-report) → 400

# 4. Report review
PATCH /reports/:id { status: "RESOLVED" } (moderator) → 200
PATCH /reports/:id (normal user) → 403

# 5. Blocking
POST /users/testuser/block → 201
GET /topics (blocked user's topic should not appear)
GET /topics/:id (blocked user's topic) → 404
POST /topics/:id/comments (on blocked user's topic) → 403
DELETE /users/testuser/block → 200
GET /topics (topic visible again)

# 6. Restriction visible on profile
GET /users/testuser/restrictions → active restrictions list
```

## Completion Criteria
- [ ] Restriction application/removal works
- [ ] Restriction guard is active on all relevant endpoints
- [ ] Expired restrictions automatically stop blocking
- [ ] Reporting system works (create, list, review)
- [ ] Blocking system works
- [ ] Block filters work in topic and comment listing
- [ ] Restriction info is visible on profile
- [ ] All tests pass
