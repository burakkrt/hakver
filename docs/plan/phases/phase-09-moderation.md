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
- note: max 500 (optional free-text — the structured `reasonId` already satisfies the mandatory-reason security rule; `note` adds moderator-specific context)

**RemoveRestrictionSchema** (body of `DELETE /moderation/restrictions/:id`):
- reason: min 5, max 500 characters, required (Turkish user-facing text). Lifting an active restriction is an accountability event; moderators/admins must document why the restriction ended early so the action shows up in the activity log audit trail

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
| DELETE | /moderation/restrictions/:id | Admin | Remove restriction early (body requires `reason`) |
| GET | /users/:username/restrictions | Public | User's active restrictions |
| GET | /moderation/restriction-categories | Public | Restriction categories and reasons |

**Authority-promotion interaction (authoritative reference, Phase 15):** When an account gains an authority role (MODERATOR, ADMIN, future authority roles) through `PATCH /admin/users/:id/role`, all of its `UserRestriction` rows are deleted inside the same transaction that swaps the role. This prevents a promoted account from being blocked from the actions it is now expected to supervise. A demotion to USER does not restore the cleared restrictions. See Phase 15 Section 7.1 for the full role-change flow.

**Restriction application flow:**
1. Validation with `ApplyRestrictionSchema`
2. Check if target user exists
3. Is the target user an admin or moderator? → 403 "Yöneticilere kısıtlama uygulanamaz"
4. Check if category and reason exist and are active
5. Check if `expiresAt` is in the future
6. Is there already an active restriction for the same action on the same user? If yes → 409
7. Create `UserRestriction` record
8. Save activity log

**Restriction removal flow** (`DELETE /moderation/restrictions/:id`):
1. Validation with `RemoveRestrictionSchema` — `reason` field is required (min 5, max 500)
2. Check if the restriction exists and is still active
3. Hard delete (or set `revokedAt`) the `UserRestriction` record
4. Write `ActivityLog` entry: `action: "restriction:remove"`, `metadata: { reason, previousExpiresAt, removedBy }`
5. Response: 200 with the removed restriction summary
6. Validation error when `reason` missing or too short: 400 `{ code: "MODERATION_REASON_REQUIRED", message: "Kısıtlamayı kaldırmak için sebep girilmelidir" }`

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
Blocking means "I don't want to see or interact with this person at all." The check is always **symmetric**: if User A blocks User B, User A cannot see User B's content AND User B cannot see User A's content. This prevents harassment scenarios and avoids information leakage about who blocked whom.

The following touch points must all call `BlockService.isBlocked(viewerId, targetId)` and honour its verdict:
- Topic listing: blocked users' topics are completely excluded from the listing payload
- Topic detail (`GET /topics/:idOrSlug`): returns `404 Not Found` with the standard `TOPIC_NOT_FOUND` envelope (see Phase 5 Section 11). Not 403 — the blocker/blocked relationship stays opaque
- Comment listing: blocked users' comments are excluded from the response (exception: anonymous comments, since their author is concealed by design)
- Voting (`POST|PATCH|DELETE /topics/:topicId/votes`): unreachable because the topic resolves to 404 under the symmetric block
- Commenting (`POST /topics/:topicId/comments`): unreachable for the same reason
- Comment like (`POST|DELETE /comments/:id/like`): the containing topic is unreachable; even on direct hit the service re-checks the block and returns 404
- Comment highlight (`PATCH|DELETE /comments/:id/highlight`): same 404 behaviour when the viewer/author pair is symmetrically blocked
- Bookmark, mute, report endpoints that resolve a topic or user: all return 404 under a symmetric block
- Profile (`GET /users/:username`): returns `ProfileUnavailableResponseSchema` with `reason: "UNAVAILABLE"` (Phase 4). The envelope is deliberately identical to the one used for other "unavailable" cases so the blocked party cannot deduce they were blocked
- Mention autocomplete / user search: candidates under a symmetric block with the viewer are filtered out before the response is emitted
- Notifications: already covered by the fan-out and creation filters (Phase 10 Section 3 step 3)
- WebSocket rooms: server refuses `joinTopic` when the topic resolves to 404 for the viewer under a symmetric block

**BlockService helper (authoritative):**
```
isBlocked(viewerId: string, targetId: string): Promise<boolean>
```
Checks `UserBlock` for a row matching `(blockerId = viewerId AND blockedId = targetId)` OR `(blockerId = targetId AND blockedId = viewerId)`. Returns `true` if either direction exists. The helper is the single point of truth — every module calls it rather than hand-rolling the `OR` predicate, so a future change (e.g., block grace period, block tiers) only touches one file.

The helper is also exposed as a `@WithBlockCheck('paramName')` decorator that modules can attach to controller handlers to short-circuit to `404 TOPIC_NOT_FOUND` (or the relevant resource's NOT_FOUND) automatically.

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
