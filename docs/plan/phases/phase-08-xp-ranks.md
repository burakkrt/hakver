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
3. **Single comment XP per topic check:** If action is `COMMENT_CREATE` → is this the user's first comment on this topic? Check topics of comments related to `(userId, action=COMMENT_CREATE, referenceType=COMMENT)` in the `XpLog` table. Has XP already been earned for the same topic? If yes → do not award XP
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

**Vote Module (Phase 6):**
- After casting a vote → `xpService.awardXp(userId, 'VOTE_CREATE', 'VOTE', voteId)`
- After vote withdrawal → `xpService.revokeXp(userId, 'VOTE_CREATE', 'VOTE', voteId)`
- Vote change → no additional XP (existing XP is preserved)
- When a vote is received on a topic → `xpService.awardXp(topic.authorId, 'TOPIC_VOTE_RECEIVED', 'TOPIC', topicId)`
- When a vote is withdrawn, the topic owner's XP: not revoked (only the voter's XP is revoked)

**Comment Module (Phase 7):**
- After comment creation → `xpService.awardXp(userId, 'COMMENT_CREATE', 'COMMENT', commentId)`
- When a comment is liked → `xpService.awardXp(comment.authorId, 'COMMENT_LIKE_RECEIVED', 'COMMENT_LIKE', commentLikeId)`

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

These endpoints are infrastructure only; the admin panel frontend will be built in Phase 14 or a future version.

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
