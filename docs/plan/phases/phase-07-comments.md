# Phase 7: Comment System

## Dependencies
- Phase 5 must be completed (topic module)

## Goals
- Comment CRUD (create, edit, delete)
- 2-level nesting (parent comment + reply)
- Comment likes
- Anonymous comments (consistent per topic, irreversible)
- Like-based sorting
- Edit rate limit
- Denormalized comment and like counts
- Define comment DTO schemas in the shared package

## Tasks

### 1. Shared Zod Schemas

`packages/shared/src/schemas/comment.ts`:

**CommentResponseSchema** (response DTO — used in REST responses and WebSocket `comment:created` event):
- id: UUID
- content: string | null (null if deleted)
- isAnonymous: boolean
- isEdited: boolean
- isDeleted: boolean
- lastEditedAt: ISO date string | null
- likeCount: number
- author: `UserPublicCardSchema | AnonymousCardSchema | null` (null if deleted — card shape, no firstName/lastName)
- isLiked: boolean (only when authenticated — whether current user liked this comment)
- replyCount: number (total reply count for level 1 comments)
- replies: `CommentResponseSchema[]` (first 5 replies, only for level 1 comments)
- createdAt: ISO date string
- _moderatorInfo: `{ realAuthorId, realUsername }` (only for users with `user:view-anonymous` permission, only when isAnonymous)

**AnonymousCardSchema** (shared anonymous identity response):
- displayName: string (e.g., "Anonim Penguen #4521")
- iconType: string (animal type)

**CreateCommentSchema:**
- content: min 1, max 1000 characters
- parentId: UUID (optional — null = level 1, populated = level 2)
- isAnonymous: boolean, default false

**UpdateCommentSchema:**
- content: min 1, max 1000 characters

**CommentQuerySchema:**
- page: number, default 1
- limit: number, default 20, max 50
- sort: "popular" | "newest" (default "popular")

### 2. Comment Module

Under `apps/api/src/modules/comment/`:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /topics/:topicId/comments | JWT + Verified | Post comment |
| GET | /topics/:topicId/comments | Public | List comments |
| PATCH | /comments/:id | JWT + Verified | Edit comment |
| DELETE | /comments/:id | JWT + Verified | Delete comment |
| POST | /comments/:id/like | JWT + Verified | Like comment |
| DELETE | /comments/:id/like | JWT + Verified | Remove like |

### 3. Comment Creation

1. Validation with `CreateCommentSchema`
2. Check if topic exists and `deletedAt: null`
3. **Profile completion check:** `ProfileCompleteGuard` ensures the commenter's profile is complete (guard chain order: JWT → Verified → ProfileComplete → Restriction → Permission)
4. **Restriction check:** `comment:create` action
5. **Nesting check:**
   - `parentId` null → level 1 comment
   - `parentId` populated → does parent comment exist? Is the parent itself level 1? (is parentId null?) If parent is already a reply (parentId populated) → use parent's parent as `parentId` (keep at level 2)
6. **Anonymous consistency check:**
   - Check if this user has previous comments on this topic
   - If yes: is the previous comment's `isAnonymous` value the same as the new comment's `isAnonymous` value? If different → 400 "Bu konudaki ilk yorumunuzda seçtiğiniz kimlik ayarını değiştiremezsiniz"
   - If no: first comment, choice is free
7. If `isAnonymous: true` → check if a record exists in the `AnonymousIdentity` table for this user+topic:
   - If exists: use existing anonymous identity
   - If not: create new anonymous identity (animal list from Phase 5 + number)
8. Create comment record
9. Update `topic.commentCount += 1` (transaction)
10. WebSocket event: `comment:created` → `{ topicId, comment: CommentResponseSchema }` (to topic room)

### 4. Comment Editing

1. Author check: `comment.authorId === currentUser.id`
2. `deletedAt: null` check
3. **Rate limit check:** has at least 30 seconds passed between `lastEditedAt` and current time? If not → 429 "Çok sık düzenleme yapıyorsunuz, lütfen bekleyin". Note: There is no time limit for comment editing (unlike topics which have a 12-hour window). Comments can be edited at any time, with only the 30-second cooldown between edits. The `isEdited` flag and `lastEditedAt` timestamp are always updated.
4. Validation with `UpdateCommentSchema`
5. Update: `content`, `isEdited = true`, `lastEditedAt = now()`

### 5. Comment Deletion

1. Author or `comment:delete` permission check
2. Soft delete: `deletedAt = now()`
3. Update `topic.commentCount -= 1` (transaction)
4. Child replies are preserved (replies under a deleted comment remain visible)
5. Deleted comment is returned as: `{ content: null, isDeleted: true, author: null }`

### 6. Comment Like

**Like:** `POST /comments/:id/like`
1. Check if comment exists and `deletedAt: null`
2. **Self-like prevention:** `comment.authorId === currentUser.id` → 403
3. Unique constraint: `(userId, commentId)`. Already liked → 409
4. Create `CommentLike` record
4. `comment.likeCount += 1` (transaction)

**Unlike:** `DELETE /comments/:id/like`
1. Does like record exist? If not → 404
2. Delete like record (hard delete)
3. `comment.likeCount -= 1` (transaction)

### 7. Comment Listing

`GET /topics/:topicId/comments`:

**Level 1 comments** (paginated):
- Comments with `parentId: null`
- Sorting: `sort=popular` → `likeCount DESC, createdAt ASC` | `sort=newest` → `createdAt DESC`
- For each comment: id, content (or null if deleted), author card (or anonymous card), isAnonymous, isEdited, likeCount, createdAt
- Under each level 1 comment, **level 2 replies** are also returned (no separate pagination, limit: first 5 replies + total reply count). Use Prisma `include` for eager loading: `include: { replies: { take: 5, orderBy: { createdAt: 'asc' } } }` to avoid N+1 queries.

**Level 2 reply expansion:**
`GET /comments/:id/replies?page=1&limit=20` — Fetch all replies for a parent comment, paginated. First 5 replies are shown inline in the topic comment list, "X more replies" button calls this endpoint.

**Anonymous comment response:**
- `isAnonymous: true` → anonymous card format from Phase 5 (displayName, iconType)
- If moderator → `_moderatorInfo` is included

**Block filter:**
- If current user exists, exclude comments from blocked users
- Anonymous comments are not subject to block filter (identity is unknown since they're anonymous)

### 8. Deleted Comment Display

A soft-deleted comment is returned in this format:
```json
{
  "id": "...",
  "content": null,
  "isDeleted": true,
  "author": null,
  "likeCount": 0,
  "replies": [...] // replies are still visible
}
```

This way the comment tree is not broken, and the user sees a "this comment was deleted" message.

### 9. @Mention System

When a comment is created or edited, the backend parses the content for `@username` patterns and creates notifications for mentioned users.

**Backend parsing:**
1. Extract all `@username` patterns from comment content using regex: `/(?<!\w)@([a-zA-Z0-9_]{3,30})(?!\w)/g`
2. Validate each username: check if user exists and is not deleted
3. Filter out: self-mention (author mentioning themselves), blocked users (mutual block), duplicate mentions (same user mentioned multiple times → notify once)
4. For each valid mention: create a `MENTIONED` notification (see Phase 10)

**On comment edit:**
- Re-parse the updated content for @mentions
- Compare with previous mentions (from the original content)
- Only notify NEW mentions (users who were not mentioned in the previous version)
- Do NOT re-notify users who were already mentioned

**Anonymous comment + @mention:**
- If the commenter is anonymous, the notification is sent with the anonymous card (e.g., "Anonim Penguen #4521 sizi bir yorumda etiketledi")
- The mentioned user does NOT learn who the anonymous commenter is

**Frontend autocomplete:**
- When user types `@` in the comment textarea, show a dropdown with matching usernames (debounced, 300ms, min 1 character after @)
- Autocomplete calls `GET /users/search?q={query}&limit=5` (new lightweight endpoint or reuse existing)
- Selected username is inserted into the textarea as `@username`
- Mentioned usernames are visually highlighted in rendered comments (link to user profile)

### 10. WebSocket Events

Additional events to the gateway set up in Phase 6:
- `comment:created` → `{ topicId, comment }` — when a new comment is posted
- `comment:deleted` → `{ topicId, commentId }` — when a comment is deleted

## Security Checklist
- [ ] Comment editing is author-only
- [ ] Comment deletion is by author or moderator
- [ ] Edit rate limit (30 seconds) is active
- [ ] Anonymous consistency check cannot be bypassed
- [ ] No user info in API response for anonymous comments
- [ ] Nesting cannot exceed 2 levels
- [ ] Like unique constraint exists
- [ ] Denormalized counters are updated with transactions
- [ ] No XSS risk (content is sanitized or stored as plain text)
- [ ] Blocked user filtering works

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# API tests:

# 1. Comment creation
POST /topics/:id/comments { content: "test" } → 201
POST /topics/:id/comments { content: "test", parentId: "..." } → 201 (reply)
POST /topics/:id/comments { parentId: "level2-comment-id" } → parentId normalized to level 1

# 2. Anonymous consistency
POST /topics/:id/comments { isAnonymous: true } → 201
POST /topics/:id/comments { isAnonymous: false } (same user, same topic) → 400

# 3. Comment editing
PATCH /comments/:id { content: "updated" } → 200, isEdited true
PATCH /comments/:id (again within 30 sec) → 429
PATCH /comments/:id (different user) → 403

# 4. Comment deletion
DELETE /comments/:id (author) → 200
DELETE /comments/:id (moderator) → 200
GET /topics/:id/comments → deleted comment { isDeleted: true, content: null }

# 5. Like
POST /comments/:id/like → 201, likeCount increased
POST /comments/:id/like (again) → 409
DELETE /comments/:id/like → 200, likeCount decreased

# 6. Listing
GET /topics/:id/comments?sort=popular → likeCount sorting
GET /topics/:id/comments?sort=newest → createdAt sorting
# Are replies present under each level 1 comment?

# 7. Denormalized counters
# Create comment → has topic.commentCount increased?
# Delete comment → has topic.commentCount decreased?
# Like → is comment.likeCount correct?
```

## Completion Criteria
- [ ] Comment creation works (level 1 and level 2)
- [ ] 2-level nesting limit is enforced
- [ ] Comment editing works (including rate limit)
- [ ] Comment deletion works (soft delete, replies preserved)
- [ ] Like add/remove works
- [ ] Anonymous comment consistency works
- [ ] Like-based sorting works
- [ ] Denormalized counters update correctly
- [ ] WebSocket events work
- [ ] All tests pass
