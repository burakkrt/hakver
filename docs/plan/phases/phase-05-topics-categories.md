# Phase 5: Topics & Categories

## Dependencies
- Phase 4 must be completed (authorization guards, user module)

## Goals
- Category CRUD (admin)
- Topic CRUD (create, list, detail, edit, delete)
- Image upload with Cloudinary
- Anonymous topic posting and anonymous identity system
- Topic editing (12-hour limit) and soft delete
- New user prerequisite check (at least 1 vote required to create a topic)
- Define topic and category DTO schemas in the shared package

## Tasks

### 1. Shared Zod Schemas

`packages/shared/src/schemas/topic.ts`:

**TopicResponseSchema** (response DTO — used in topic detail, topic list items, and WebSocket events):
- id: UUID
- title: string
- content: string
- slug: string
- isAnonymous: boolean
- isEdited: boolean
- editableUntil: ISO date string
- voteCountRight: number
- voteCountWrong: number
- commentCount: number
- category: `{ id, name, slug }`
- author: `UserPublicResponseSchema | AnonymousCardSchema` (based on isAnonymous)
- images: `{ id, url, sortOrder }[]`
- coverImage: `{ id, url }` | null
- myVote: `"RIGHT" | "WRONG" | null` (only when authenticated)
- isOwnTopic: boolean (only when authenticated and topic is anonymous)
- isMuted: boolean (only when authenticated, from Phase 10)
- createdAt: ISO date string

**TopicListItemSchema** (truncated version for list endpoints):
- id, title, slug, isAnonymous, voteCountRight, voteCountWrong, commentCount, category, author, coverImage, myVote, createdAt
- content: string (truncated to 200 characters)

`packages/shared/src/schemas/category.ts`:

**CategoryResponseSchema** (response DTO):
- id: UUID
- name: string
- slug: string
- description: string | null
- isActive: boolean
- sortOrder: number

**CreateTopicSchema:**
- title: min 5, max 150 characters
- content: min 50, max 3000 characters
- categoryId: UUID
- isAnonymous: boolean, default false

**UpdateTopicSchema:**
- title: min 5, max 150 (optional)
- content: min 50, max 3000 (optional)

**TopicQuerySchema** (listing filters):
- categorySlug: string (optional)
- page: number, default 1, min 1
- limit: number, default 20, min 1, max 50
- sort: "newest" | "popular" | "trending" (optional, default "newest")
- search: string (optional, min 2 characters)

`packages/shared/src/schemas/category.ts`:

**CreateCategorySchema:**
- name: min 2, max 50
- description: max 200 (optional)

### 2. Upload Module (Cloudinary)

Under `apps/api/src/modules/upload/`:

**`upload.module.ts`** and **`upload.service.ts`**:
- Cloudinary SDK configuration (`CLOUDINARY_CLOUD_NAME`, `API_KEY`, `API_SECRET`)
- `uploadImage(file, folder)` → `{ url, publicId }`
- `deleteImage(publicId)` → void
- File validation:
  - Maximum size: 5 MB
  - Allowed MIME types: `image/jpeg`, `image/png`, `image/webp`
  - GIF not accepted
  - **Magic bytes validation:** Use the `file-type` npm package to verify the actual file content matches the declared MIME type. MIME type from the request header can be spoofed — magic bytes (file signature) provide reliable validation. Check magic bytes BEFORE uploading to Cloudinary to prevent unnecessary traffic.
- Cloudinary upload options: `folder: "hakver/topics"`, automatic format conversion, quality optimization

### 3. Category Module

Under `apps/api/src/modules/category/`:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /categories | Public | Active category list (cached — `@CacheTTL(3600)`, invalidate on admin category changes) |
| POST | /categories | Admin | Create new category |
| PATCH | /categories/:id | Admin | Update category |
| DELETE | /categories/:id | Admin | Deactivate category (soft) |

- Slug is automatically derived from the name (Turkish character support: ö→o, ü→u, ş→s, ç→c, ğ→g, ı→i)
- When a category is deleted, `isActive = false` is set; topics under it are preserved

### 4. Topic Module

Under `apps/api/src/modules/topic/`:

```
topic/
├── topic.module.ts
├── topic.controller.ts
├── topic.service.ts
└── dto/
```

### 5. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /topics | JWT + Verified | Create topic |
| GET | /topics | Public | Topic list (paginated, filterable) |
| GET | /topics/:idOrSlug | Public | Topic detail — accepts both UUID and slug. Backend auto-detects by UUID format (8-4-4-4-12 hex pattern). |
| PATCH | /topics/:id | JWT + Verified | Edit topic (12-hour limit, author only) |
| DELETE | /topics/:id | JWT + Verified | Delete topic (author or moderator only) |
| POST | /topics/:id/images | JWT + Verified | Add image to topic |
| DELETE | /topics/:id/images/:imageId | JWT + Verified | Remove image |
| PATCH | /topics/:id/cover | JWT + Verified | Set cover image (author only, within editableUntil) |
| GET | /topics/search | Public | Search topics by title (ILIKE query, paginated) — same query params as topic list + `search` parameter |
| POST | /topics/:id/bookmark | JWT + Verified | Bookmark topic |
| DELETE | /topics/:id/bookmark | JWT + Verified | Remove bookmark |
| GET | /users/me/bookmarks | JWT | My bookmarked topics (paginated) |
| PATCH | /topics/:id/pin | Moderator+ | Pin/unpin topic |

### 6. Topic Creation Flow

1. Validation with `CreateTopicSchema`
2. **New user prerequisite check:** Verify the user has cast at least 1 vote (check for records in the Vote table). If not → 403 "Konu açabilmek için en az bir konuya oy vermeniz gerekiyor"
3. **Profile completion check:** `ProfileCompleteGuard` ensures username, firstName, lastName are not null (guard chain order: JWT → Verified → ProfileComplete → Restriction → Permission)
4. **Restriction check:** `restriction.guard` checks for active restrictions (`topic:create` action)
5. Check if category exists and `isActive: true`
6. Create topic:
   - `slug` field is auto-generated from the title (Turkish character support, unique — appends `-2`, `-3` on collision)
   - `editableUntil = createdAt + 12 hours`
   - `voteCountRight = 0`, `voteCountWrong = 0`, `commentCount = 0`
7. If `isAnonymous: true` → create `AnonymousIdentity` record:
   - Select a random `iconType` from the animal list
   - Generate a random 4-digit number
   - `displayName = "Anonim {Animal} #{number}"` (e.g.: "Anonim Penguen #4521")
8. Return the topic with its ID
9. **Cover photo:** If images are uploaded, the first image (`sortOrder: 0`) is automatically set as the cover photo (`topic.coverImageId`). The author can change the cover selection via `PATCH /topics/:id/cover` within the editing window.

**Animal list** (for anonymous identities):
Penguen, Baykuş, Tilki, Panda, Koala, Yunus, Kelebek, Kartal, Kurt, Aslan, Kedi, Tavşan, Sincap, Flamingo, Papağan

### 7. Image Upload

`POST /topics/:id/images`:
1. Topic author check
2. Existing image count check (max 4)
3. File validation (size, type)
4. Upload to Cloudinary
5. Create `TopicImage` record
6. Response: `{ id, url, sortOrder }`

File received via multipart form-data (`@nestjs/platform-express` Multer).

Image add and remove operations follow the same 12-hour editing window (`editableUntil`) as text editing. After the window expires, images cannot be added or removed.

**Image removal:** When an image is explicitly removed via `DELETE /topics/:id/images/:imageId`, the image is also deleted from Cloudinary (`uploadService.deleteImage(publicId)`). This prevents orphan files and unnecessary Cloudinary storage usage. Note: when a topic is soft-deleted, images are NOT removed from Cloudinary (soft delete consistency).

**Rate limiting:** Image upload endpoint (`POST /topics/:id/images`) is rate limited to 10 requests per minute per user to prevent Cloudinary quota abuse.

### 8. Topic Editing

`PATCH /topics/:id`:
1. Topic author check (`authorId === currentUser.id`)
2. `editableUntil` check: if `now() < editableUntil` is false → 403 "Düzenleme süresi dolmuştur"
3. `deletedAt !== null` check
4. Validation with `UpdateTopicSchema`
5. Update, set `isEdited = true`

### 9. Topic Deletion

`DELETE /topics/:id`:
1. Author or user with `topic:delete` permission check
2. `deletedAt = now()` (soft delete)
3. Images are not deleted from Cloudinary (soft delete consistency)

### 10. Topic Listing

`GET /topics`:
- Pagination: offset-based (`page`, `limit`)
- Filtering: `categorySlug` (optional)
- Search: `search` query parameter (optional) — PostgreSQL ILIKE on title field. Minimum 2 characters.
- Sorting:
  - `newest`: `createdAt DESC`
  - `popular`: `voteCountRight + voteCountWrong DESC`
  - `trending`: time-weighted engagement score — `(voteCountRight + voteCountWrong + commentCount) / pow(hoursSinceCreation + 2, 1.5) DESC`. Calculated in-query using PostgreSQL `EXTRACT(EPOCH FROM (NOW() - "createdAt")) / 3600` for hours. This ensures recent topics with high engagement rank above old topics with accumulated votes.
- Pinned topics always appear at the top regardless of sort order: `ORDER BY isPinned DESC, {selected sort}`
- `deletedAt: null` filter (soft delete middleware)
- For each topic: id, title, content (truncated 200 char), author card (or anonymous card), category, vote counts, comment count, image count, createdAt
- If authenticated: `myVote: "RIGHT" | "WRONG" | null` — the current user's vote on this topic (requires a batch query or LEFT JOIN on the Vote table)
- **Block filter:** If current user exists, exclude topics from blocked users

### 11. Topic Detail

`GET /topics/:id`:
- All topic info + images
- Author card: if `isAnonymous` → `displayName` and `iconType` from `AnonymousIdentity`, no user info returned
- Author card: if not `isAnonymous` → public user info (masked name-surname, username, avatar, rank)
- If the viewing user is the topic author and topic is anonymous → `isOwnTopic: true` flag in response (for the client to show "this is your topic")

### 12. Anonymous Topic Response Rules

**Response for normal users:**
```json
{
  "id": "...",
  "title": "...",
  "content": "...",
  "isAnonymous": true,
  "author": {
    "displayName": "Anonim Penguen #4521",
    "iconType": "penguin"
  }
}
```

**Response for moderators (user:view-anonymous permission):**
```json
{
  "id": "...",
  "isAnonymous": true,
  "author": {
    "displayName": "Anonim Penguen #4521",
    "iconType": "penguin"
  },
  "_moderatorInfo": {
    "realAuthorId": "...",
    "realUsername": "..."
  }
}
```

The `_moderatorInfo` field is only returned to users with the `user:view-anonymous` permission.

## Security Checklist
- [ ] User info is not returned in API response for anonymous topics (except moderators)
- [ ] Image upload: file type and size are validated on the backend
- [ ] Cloudinary upload goes through the backend (no direct frontend access)
- [ ] Topic editing can only be done by the author, within 12 hours
- [ ] Topic deletion can only be done by the author or moderator/admin
- [ ] New user prerequisite check cannot be bypassed
- [ ] Restriction check is active
- [ ] Blocked users' content is filtered
- [ ] No SQL injection risk (Prisma parameterized queries)

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# API tests:

# 1. Category
GET /api/v1/categories → 200, 2 categories (Genel, İlişkiler)
POST /api/v1/categories (admin) → 201
POST /api/v1/categories (user) → 403

# 2. Topic creation
POST /api/v1/topics (user who has never voted) → 403
POST /api/v1/topics (user who has voted) → 201
POST /api/v1/topics (content under 50 chars) → 400
POST /api/v1/topics (anonymous) → 201, AnonymousIdentity created

# 3. Topic listing
GET /api/v1/topics → 200, paginated
GET /api/v1/topics?categorySlug=genel → 200, filtered

# 4. Topic detail
GET /api/v1/topics/:id → 200
GET /api/v1/topics/:id (anonymous, normal user) → author info hidden
GET /api/v1/topics/:id (anonymous, moderator) → _moderatorInfo present

# 5. Topic editing
PATCH /api/v1/topics/:id (author, within 12 hours) → 200, isEdited true
PATCH /api/v1/topics/:id (author, after 12 hours) → 403
PATCH /api/v1/topics/:id (different user) → 403

# 6. Topic deletion
DELETE /api/v1/topics/:id (author) → 200
DELETE /api/v1/topics/:id (moderator) → 200
GET /api/v1/topics/:id (deleted) → 404

# 7. Image upload
POST /api/v1/topics/:id/images (JPEG, 3MB) → 201
POST /api/v1/topics/:id/images (GIF) → 400
POST /api/v1/topics/:id/images (6MB) → 400
POST /api/v1/topics/:id/images (5th image) → 400
```

## Completion Criteria
- [ ] Category CRUD works (admin protected)
- [ ] Topic CRUD works (create, list, detail, edit, delete)
- [ ] Image upload works (Cloudinary integration)
- [ ] Anonymous identity system works (consistent per topic)
- [ ] Anonymous privacy rules are correctly applied
- [ ] 12-hour editing restriction works
- [ ] New user prerequisite check works
- [ ] Soft delete works
- [ ] Block filter works in listing
- [ ] All tests pass
