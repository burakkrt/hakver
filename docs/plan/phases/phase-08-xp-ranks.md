# Phase 8: XP & Rank System

## Dependencies
- Phase 5 (topics), Phase 6 (voting), and Phase 7 (comments) must be completed

## Goals
- XP earning on actions (topic, vote, comment, like)
- XP revocation on vote withdrawal, unlike, and author self-delete
- Daily XP limit (1000, UTC-based)
- Single comment XP rule per topic (metadata.topicId)
- TOPIC_VOTE_RECEIVED unique-per-voter check (metadata.voterId)
- Minimum character rule (with trim)
- Race condition prevention (`withUserXpLock` — SELECT FOR UPDATE)
- Automatic rank assignment
- Displaying XP and rank info on user cards
- Admin XP management (add, subtract, set — with audit trail)
- XP reconciliation (daily cron job for drift detection)

## Tasks

### 1. XP Module

Under `apps/api/src/modules/xp/`:

```
xp/
├── xp.module.ts
├── xp.service.ts
└── xp.constants.ts
```

This module has no controller — it does not expose API endpoints directly. It is used by other modules (topic, vote, comment).

### 2. XP Award Service

**`xp.service.ts`** main methods:

**Shared lock helper — `withUserXpLock(userId, callback)`:**

Both `awardXp` and `revokeXp` modify `User.totalXp`. To prevent race conditions (concurrent award + revoke, or two concurrent awards exceeding the daily limit), all XP write operations use a shared locking mechanism:

```
async withUserXpLock<T>(userId: string, callback: (tx) => Promise<T>): Promise<T> {
  return prisma.$transaction(async (tx) => {
    // Acquire row-level lock on the User record
    await tx.$queryRaw`SELECT id FROM "User" WHERE id = ${userId} FOR UPDATE`;
    return callback(tx);
  });
}
```

- Acquires a `SELECT ... FOR UPDATE` row-level lock on the target User record
- Lock is held only for the duration of the transaction — other users' XP operations run in parallel
- Both `awardXp` and `revokeXp` call their logic inside this helper, ensuring mutual exclusion
- Admin XP adjustments (Task 5.1) also use this helper

**`awardXp(userId, action, referenceType, referenceId, metadata?)`:**

Runs inside `withUserXpLock(userId, ...)`:

1. Find the action from the `XpAction` table. If `isActive: false` → do not award XP
2. **Daily limit check (UTC):** Calculate the user's total XP earned today (`createdAt >= start of today in UTC`) from the `XpLog` table. All daily limit calculations use UTC timezone consistently. If `dailyTotal + basePoints > 1000` → do not award XP (skip silently, do not return error). **Note:** `ADMIN_ADJUST` actions are exempt from this limit
3. **Single comment XP per topic check:** If action is `COMMENT_CREATE` → check `XpLog` for existing positive entries with `(userId, action=COMMENT_CREATE)` and `metadata.topicId === currentTopicId`. **Only count positive (non-revoked) XP entries.** If the user previously earned XP but it was revoked (comment deleted by author), they can earn XP again. If a positive entry exists → do not award XP. The `metadata` JSON field stores `{ topicId }` to avoid joining the Comment table
4. **Minimum content quality check (XP gate only — comment is always stored):** If action is `COMMENT_CREATE`, evaluate the comment's **trimmed** content against all three conditions. Any failure skips XP. The comment itself is never rejected by this rule; Phase 7's Zod schema is the only authoritative validation for whether a comment saves:
   - Trimmed length ≥ 10 characters (whitespace-only bypass is prevented by the trim)
   - At least 3 distinct **non-whitespace** characters — blocks trivial spam like `"aaaaaaaaaa"` while letting natural short Turkish replies pass (any three-letter distinct set in "tamam mı", "evet evet", "harika" clears this easily)
   - Distinct non-whitespace ratio ≥ 25% of trimmed non-whitespace length — lenient enough to keep repeated emphasis patterns like "tamamm tamamm" (distinct `{t,a,m, }`; non-whitespace length 12, distinct non-whitespace count 3 → 25%) valid, while still rejecting `"hahahahahaha"` (2 distinct out of 12 → 16%). Whitespace is excluded from both numerator and denominator so sentence-level spacing does not skew the ratio
   
   The ratio was lowered from an earlier 30% draft after reviewing Turkish phrasing patterns — short genuine replies with limited alphabet (e.g., emoji clusters, repeated affirmations) were tripping the stricter threshold. 25% preserves spam resistance while respecting natural short Turkish text
5. Create `XpLog` record (`points: basePoints`)
6. Update `User.totalXp += basePoints`
7. **Rank check:** `checkAndUpdateRank(userId)`

> **Badge system integration is intentionally deferred.** The badge catalog, per-badge XP bonus semantics, and the badge-earn award hook are not part of MVP — see `plan.md` "Detailed Design Pending Items". When the design ships, this flow will be revisited to insert the bonus-application and badge-award steps.

**`revokeXp(userId, action, referenceType, referenceId)`:**

Runs inside `withUserXpLock(userId, ...)`:

1. Find the related record in the `XpLog` table
2. If found → create a reverse XP log (negative points)
3. Update `User.totalXp -= points` (minimum 0)
4. **Rank check:** `checkAndUpdateRank(userId)`

**`checkAndUpdateRank(userId)`:**
1. Get the `User.totalXp` value
2. Find the highest rank from the `Rank` table where `minXp <= totalXp`
3. If current rank is different → update `User.currentRankId`

### 3. Integration with Existing Modules

Add XP hooks to modules created in Phases 5, 6, and 7:

**Topic Module (Phase 5):**
- After topic creation → `xpService.awardXp(userId, 'TOPIC_CREATE', 'TOPIC', topicId)`
- After topic deletion by author → `xpService.revokeXp(userId, 'TOPIC_CREATE', 'TOPIC', topicId)`
- After topic deletion by moderator → XP is preserved (no revocation)

**Vote Module (Phase 6):**
- After casting a vote → `xpService.awardXp(userId, 'VOTE_CREATE', 'VOTE', voteId)`
- After vote withdrawal → `xpService.revokeXp(userId, 'VOTE_CREATE', 'VOTE', voteId)`
- Vote change → no additional XP (existing XP is preserved)
- When a vote is received on a topic → `xpService.awardXp(topic.authorId, 'TOPIC_VOTE_RECEIVED', 'TOPIC', topicId, { voterId: currentUser.id })` **with unique per voter-topic check:** before awarding, check if this specific voter has already triggered `TOPIC_VOTE_RECEIVED` XP for this topic (check `XpLog` for `userId=topic.authorId, action=TOPIC_VOTE_RECEIVED, referenceId=topicId` with metadata containing `voterId`). If yes → do not award again. This prevents XP farming via vote/withdraw/re-vote cycles.
- When a vote is withdrawn, the topic owner's `TOPIC_VOTE_RECEIVED` XP: not revoked (the voter's own XP is revoked). Since the unique-per-voter check prevents re-awarding, farming is blocked without needing revocation.

**Comment Module (Phase 7):**
- After comment creation → `xpService.awardXp(userId, 'COMMENT_CREATE', 'COMMENT', commentId, { topicId })` — metadata includes `topicId` for single-comment-per-topic check
- After comment deletion by author → `xpService.revokeXp(userId, 'COMMENT_CREATE', 'COMMENT', commentId)`
- After comment deletion by moderator → XP is preserved (no revocation)
- When a comment is liked → `xpService.awardXp(comment.authorId, 'COMMENT_LIKE_RECEIVED', 'COMMENT_LIKE', commentLikeId)`
- When a comment is unliked → `xpService.revokeXp(comment.authorId, 'COMMENT_LIKE_RECEIVED', 'COMMENT_LIKE', commentLikeId)` (consistent with vote withdrawal XP revocation)

### 3.1. XP Revocation on Content Deletion

When content is soft-deleted, XP handling depends on who performed the deletion:

**User self-deletion (author deletes their own content):**
- Topic deleted by author → revoke `TOPIC_CREATE` XP (50 points)
- Comment deleted by author → revoke `COMMENT_CREATE` XP (10 points)
- This prevents XP farming (create → earn XP → delete → repeat)

**Moderator deletion:**
- Topic deleted by moderator → XP is NOT revoked (the user produced content, moderator enforced rules)
- Comment deleted by moderator → XP is NOT revoked

**Implementation:** The `deleteBy` context (author vs moderator) is passed to the XP service. The rule is: **"If the actor is the content author (`authorId === currentUser.id`), always revoke XP — regardless of the actor's role."** This means a moderator who deletes their OWN topic/comment loses XP, just like any other user. The moderator privilege only applies when deleting SOMEONE ELSE's content — in that case, the content author's XP is preserved. This prevents XP farming by any user, including moderators.

### 4. Displaying Rank Info in Responses

Rank and XP are part of the `UserPublicCardSchema` defined in Phase 4. Every endpoint that returns a user card already emits this shape:

```json
{
  "id": "...",
  "username": "...",
  "avatar": { "url": "..." },
  "rank": { "name": "Jüri Adayı" },
  "totalXp": 750,
  "isDeletedAuthor": false
}
```

This info is visible in:
- Author info on topic cards
- Author info on comment cards
- Notification actor cards
- User profile page header (in addition to profile-only fields such as bio and masked name)

Rank is not shown on anonymous cards — they render the animal icon + anonymous display name instead of a username/rank pair.

### 5. Admin XP Configuration

Since XP values are managed from the database, admin endpoints:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /admin/xp-actions | Admin | XP action list |
| PATCH | /admin/xp-actions/:id | Admin | Update XP value |
| GET | /admin/ranks | Admin | Rank list |
| PATCH | /admin/ranks/:id | Admin | Update rank |

These endpoints are infrastructure only; the admin panel frontend will be built in Phase 15.

### 5.1. Admin XP Management

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| PATCH | /admin/users/:id/xp | Admin | Adjust user's XP (add, subtract, or set) |
| GET | /admin/users/:id/xp-logs | Admin | User's XP history (paginated) |

**`PATCH /admin/users/:id/xp`** — Admin XP adjustment:

Request body:
```json
{
  "type": "ADD" | "SUBTRACT" | "SET",
  "points": 5000,
  "reason": "Topluluk katkısı ödülü"
}
```

Flow:
1. Validate request (points > 0, reason required, max 500 chars)
2. Check target user exists and is not deleted
3. Inside a Prisma `$transaction` with `SELECT ... FOR UPDATE` on User row:
   - **ADD:** Create XpLog with `action=ADMIN_ADJUST`, `referenceType=USER`, `referenceId=targetUserId`, `points=+requestedPoints`, `metadata={ reason, adjustedBy: adminUserId, type: "ADD" }`
   - **SUBTRACT:** Create XpLog with `action=ADMIN_ADJUST`, `referenceType=USER`, `referenceId=targetUserId`, `points=-requestedPoints`, `metadata={ reason, adjustedBy: adminUserId, type: "SUBTRACT" }`. Ensure `User.totalXp` does not go below 0.
   - **SET:** Calculate difference (`requestedPoints - currentTotalXp`), create XpLog with `referenceType=USER`, `referenceId=targetUserId`, the difference as points, `metadata={ reason, adjustedBy: adminUserId, type: "SET", previousXp: currentTotalXp }`
4. Update `User.totalXp` accordingly
5. Run `checkAndUpdateRank(userId)` — rank may change after XP adjustment
6. Log the action in ActivityLog (security/moderation category — 1 year retention)

**Security:**
- Daily XP limit (1000) does NOT apply to admin adjustments
- Only ADMIN role can use this endpoint (not MODERATOR)
- Every adjustment is logged with admin identity, reason, and previous/new XP values
- Reason field is mandatory for audit trail

**Rank list caching:** Rank queries used in `checkAndUpdateRank()` should be cached (`@CacheTTL(3600)`). Invalidate cache when admin updates rank settings.

## Security Checklist
- [ ] Daily XP limit cannot be bypassed (UTC-based, checked within transaction lock)
- [ ] Multiple comment XP for the same topic is not awarded (metadata.topicId check)
- [ ] Minimum character check uses `trim()` — whitespace-only bypass prevented
- [ ] XP revocation does not produce negative total XP (min 0)
- [ ] XP logs cannot be manipulated (created by the system only)
- [ ] `withUserXpLock` is used by both `awardXp` and `revokeXp` — no race conditions
- [ ] Concurrent XP operations for the same user are serialized (SELECT FOR UPDATE)
- [ ] TOPIC_VOTE_RECEIVED unique-per-voter check prevents vote/withdraw/re-vote farming (metadata.voterId)
- [ ] Author self-delete always revokes XP regardless of role (moderator included)
- [ ] Admin XP adjustment endpoint is only open to ADMIN role (not MODERATOR)
- [ ] Admin XP adjustments are exempt from daily limit but fully logged (ActivityLog + XpLog)
- [ ] Admin XP adjustment requires mandatory reason field for audit trail
- [ ] XP reconciliation cron job detects and corrects totalXp drift

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# Integration tests:

# 1. Topic creation XP
POST /topics → create topic
# Has User.totalXp increased by 50?
# Was an XpLog record created?

# 2. Voting XP
POST /topics/:id/votes → cast vote
# Voter: has totalXp increased by 5?
# Topic owner: has totalXp increased by 1? (TOPIC_VOTE_RECEIVED)

# 3. Vote withdrawal XP revocation
DELETE /topics/:id/votes → withdraw vote
# Voter: has totalXp decreased by 5?
# Is there a negative record in XpLog?

# 4. Comment XP
POST /topics/:id/comments → post comment
# Has totalXp increased by 10?
POST /topics/:id/comments → 2nd comment on same topic
# Did totalXp NOT increase? (single XP per topic)

# 5. Minimum character
POST /topics/:id/comments { content: "hi" } → comment is created but XP is not awarded

# 6. Like XP
POST /comments/:id/like → like
# Has the comment author's totalXp increased by 2?

# 7. Daily limit
# Perform actions until reaching 1000 XP
# Next action should not award XP

# 8. Rank assignment (10-tier Justice theme, see Phase 2 seed)
# totalXp 0 → rank "Yeni Vatandaş"
# totalXp 100+ → rank "Gözlemci"
# totalXp 300+ → rank "Tanık"
# totalXp 1750+ → rank "Jüri Üyesi"
# totalXp 8500+ → rank "Hakem"
# totalXp 70000+ → rank "Adalet Bilgesi"

# 9. Is rank visible in responses?
GET /users/:username → is rank info present?
GET /topics/:id → is rank present in author card?

# 10. Admin XP adjustment
PATCH /admin/users/:id/xp { type: "ADD", points: 100, reason: "test" } (admin) → 200, totalXp increased
PATCH /admin/users/:id/xp { type: "SUBTRACT", points: 50, reason: "test" } (admin) → 200, totalXp decreased
PATCH /admin/users/:id/xp { type: "SET", points: 5000, reason: "test" } (admin) → 200, totalXp set to 5000
PATCH /admin/users/:id/xp (moderator) → 403
PATCH /admin/users/:id/xp { points: 100 } (no reason) → 400
# Does rank update after XP adjustment?
# Is ActivityLog entry created?
# Is XpLog entry created with referenceType=USER?

# 11. Race condition test
# Two concurrent awardXp calls for the same user → totalXp correct? (no daily limit bypass?)
# Concurrent awardXp + revokeXp → totalXp consistent?

# 12. Metadata checks
# Comment XP → does XpLog contain metadata.topicId?
# TOPIC_VOTE_RECEIVED → does XpLog contain metadata.voterId?
# Vote/withdraw/re-vote → is TOPIC_VOTE_RECEIVED awarded only once per voter-topic?

# 13. Unlike XP revocation
DELETE /comments/:id/like → is COMMENT_LIKE_RECEIVED XP revoked?

# 14. Moderator self-delete
# Moderator deletes own topic → XP revoked (author rule applies)
# Moderator deletes other user's topic → XP preserved
```

## Completion Criteria
- [ ] Topic creation, voting, commenting awards XP
- [ ] Receiving likes awards XP
- [ ] Vote withdrawal revokes XP
- [ ] Unlike revokes COMMENT_LIKE_RECEIVED XP
- [ ] Daily 1000 XP limit works (UTC-based)
- [ ] Single comment XP per topic rule works (metadata.topicId)
- [ ] TOPIC_VOTE_RECEIVED unique-per-voter works (metadata.voterId)
- [ ] Minimum character rule works (with trim)
- [ ] Rank is automatically assigned
- [ ] Rank is visible on user cards
- [ ] `withUserXpLock` prevents race conditions (concurrent test passes)
- [ ] Author self-delete revokes XP regardless of role
- [ ] Admin XP/Rank configuration endpoints work
- [ ] Admin XP adjustment (ADD/SUBTRACT/SET) works
- [ ] Admin XP adjustment is logged and auditable
- [ ] XP reconciliation cron job is configured
- [ ] All tests pass
