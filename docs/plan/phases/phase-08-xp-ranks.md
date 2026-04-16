# Phase 8: XP & Rank System

## Dependencies
- Phase 5 (topics), Phase 6 (voting), and Phase 7 (comments) must be completed

## Goals
- XP earning on actions
- XP revocation on vote withdrawal
- Daily XP limit (1000)
- Single comment XP rule per topic
- Minimum character rule
- Automatic rank assignment
- Displaying XP and rank info on user cards

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

**`awardXp(userId, action, referenceType, referenceId)`:**
1. Find the action from the `XpAction` table. If `isActive: false` → do not award XP
2. **Daily limit check:** Calculate the user's total XP earned today (`createdAt >= start of today`) from the `XpLog` table. If `totalXp + newXp > 1000` → do not award XP (skip silently, do not return error)
3. **Single comment XP per topic check:** If action is `COMMENT_CREATE` → is this the user's first comment on this topic? Check `XpLog` for `(userId, action=COMMENT_CREATE, referenceType=COMMENT)` entries related to this topic. **Only count positive (non-revoked) XP entries.** If the user previously earned XP but it was revoked (comment deleted by author), they can earn XP again for a new comment on the same topic. If a positive entry exists → do not award XP
4. **Minimum character check:** If action is `COMMENT_CREATE` → is the comment's character count less than 10? If yes → do not award XP
5. Create `XpLog` record (positive points)
6. Update `User.totalXp += points`
7. **Rank check:** `checkAndUpdateRank(userId)`

**`revokeXp(userId, action, referenceType, referenceId)`:**
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
- When a vote is received on a topic → `xpService.awardXp(topic.authorId, 'TOPIC_VOTE_RECEIVED', 'TOPIC', topicId)` **with unique per voter-topic check:** before awarding, check if this specific voter has already triggered `TOPIC_VOTE_RECEIVED` XP for this topic (check `XpLog` for `userId=topic.authorId, action=TOPIC_VOTE_RECEIVED, referenceId=topicId` with metadata containing `voterId`). If yes → do not award again. This prevents XP farming via vote/withdraw/re-vote cycles.
- When a vote is withdrawn, the topic owner's `TOPIC_VOTE_RECEIVED` XP: not revoked (the voter's own XP is revoked). Since the unique-per-voter check prevents re-awarding, farming is blocked without needing revocation.

**Comment Module (Phase 7):**
- After comment creation → `xpService.awardXp(userId, 'COMMENT_CREATE', 'COMMENT', commentId)`
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

**Implementation:** The `deleteBy` context (author vs moderator) is passed to the XP service. Use the `comment:delete` and `topic:delete` permission check result to determine the actor type. If the actor is the author (`authorId === currentUser.id`), revoke XP. If the actor has `topic:delete` or `comment:delete` permission (moderator/admin), keep XP.

### 4. Displaying Rank Info in Responses

Add rank info to all endpoints that return a user card:

```json
{
  "id": "...",
  "username": "...",
  "avatar": { "url": "..." },
  "rank": { "name": "Aktif Üye", "minXp": 500 },
  "totalXp": 750
}
```

This info is visible in:
- Author info on topic cards
- Author info on comment cards
- User profile page
- (Rank is not shown on anonymous cards)

### 5. Admin XP Configuration

Since XP values are managed from the database, admin endpoints:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /admin/xp-actions | Admin | XP action list |
| PATCH | /admin/xp-actions/:id | Admin | Update XP value |
| GET | /admin/ranks | Admin | Rank list |
| PATCH | /admin/ranks/:id | Admin | Update rank |

These endpoints are infrastructure only; the admin panel frontend will be built in Phase 15.

**Rank list caching:** Rank queries used in `checkAndUpdateRank()` should be cached (`@CacheTTL(3600)`). Invalidate cache when admin updates rank settings.

## Security Checklist
- [ ] Daily XP limit cannot be bypassed
- [ ] Multiple comment XP for the same topic is not awarded
- [ ] Minimum character check cannot be bypassed
- [ ] XP revocation does not produce negative total XP (min 0)
- [ ] XP logs cannot be manipulated (created by the system only)
- [ ] Admin endpoints are only open to the ADMIN role

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

# 8. Rank assignment
# totalXp 0 → rank "Çaylak"
# totalXp 100+ → rank "Üye"
# totalXp 500+ → rank "Aktif Üye"

# 9. Is rank visible in responses?
GET /users/:username → is rank info present?
GET /topics/:id → is rank present in author card?
```

## Completion Criteria
- [ ] Topic creation, voting, commenting awards XP
- [ ] Receiving likes awards XP
- [ ] Vote withdrawal revokes XP
- [ ] Daily 1000 XP limit works
- [ ] Single comment XP per topic rule works
- [ ] Minimum character rule works
- [ ] Rank is automatically assigned
- [ ] Rank is visible on user cards
- [ ] Admin XP/Rank configuration endpoints work
- [ ] All tests pass
