# Phase 13: Frontend — Content Pages

## Dependencies
- Phase 8 (XP), Phase 9 (moderation), and Phase 12 (frontend auth) must be completed

## Goals
- Home page / Feed (topic list)
- Topic detail page
- Topic creation page
- Voting UI (real-time)
- Comment section (nesting, likes, anonymous, sorting)
- Emoji picker integration
- Category filter
- Reporting and blocking UI

## Tasks

### 1. Content API Hooks

`src/hooks/use-topics.ts`:
- `useTopics(filters)` → GET /topics (infinite query, paginated)
- `useTopic(id)` → GET /topics/:id (query)
- `useCreateTopic()` → POST /topics (mutation)
- `useUpdateTopic()` → PATCH /topics/:id (mutation)
- `useDeleteTopic()` → DELETE /topics/:id (mutation)
- `useUploadTopicImage()` → POST /topics/:id/images (mutation)

`src/hooks/use-votes.ts`:
- `useMyVote(topicId)` → GET /topics/:topicId/votes/me (query)
- `useVote()` → POST /topics/:topicId/votes (mutation)
- `useChangeVote()` → PATCH /topics/:topicId/votes (mutation)
- `useWithdrawVote()` → DELETE /topics/:topicId/votes (mutation)

`src/hooks/use-comments.ts`:
- `useComments(topicId, filters)` → GET /topics/:topicId/comments (query)
- `useCommentReplies(commentId)` → GET /comments/:commentId/replies (query, paginated — for "X yanıt daha" expansion)
- `useCreateComment()` → POST /topics/:topicId/comments (mutation)
- `useUpdateComment()` → PATCH /comments/:id (mutation)
- `useDeleteComment()` → DELETE /comments/:id (mutation)
- `useLikeComment()` → POST /comments/:id/like (mutation)
- `useUnlikeComment()` → DELETE /comments/:id/like (mutation)

`src/hooks/use-categories.ts`:
- `useCategories()` → GET /categories (query)

`src/hooks/use-reports.ts`:
- `useCreateReport()` → POST /reports (mutation)

`src/hooks/use-blocks.ts`:
- `useBlockUser()` → POST /users/:username/block (mutation)
- `useUnblockUser()` → DELETE /users/:username/block (mutation)

### 2. Home Page / Feed (`/`)

**Server Component** (for SEO):
- First page data fetched server-side
- Subsequent pages loaded client-side with infinite scroll (TanStack Query)

**Structure:**
- Top: Category filters (horizontal scroll, chips: "Tümü", "Genel", "İlişkiler")
- Sorting: "En Yeni" / "Popüler" tab or toggle
- Topic card list (card grid or list view)

**Topic card component** (`src/components/topic/topic-card.tsx`):
- Title (truncated)
- Content preview (first 150 characters)
- Author card (user-card or anonymous-card)
- Category badge
- Vote status: Haklı / Haksız buttons + counts (small version)
- Comment count icon + count
- Vote indicator: if authenticated, a small colored dot or icon showing the user's vote (green dot = Hakli, red dot = Haksiz, no dot = not voted). This comes from the `myVote` field in the topic list API response.
- Image thumbnail (if available, first image)
- Date (relative: "2 saat önce")
- Card click → navigate to topic detail

**Infinite scroll:** Automatically load next page when approaching the bottom.

**Empty state:** "Henüz konu paylaşılmamış. İlk konuyu siz açın!" + CTA button.

### 3. Topic Detail Page (`/topics/[slug]`)

**Server Component** (for SEO):
- Initial data fetched server-side
- Dynamic meta tags (title, description, OG)

**Structure:**

*Top section — Topic:*
- Title (full)
- Content text (Twemoji render)
- Images (gallery/carousel)
- Cover image displayed prominently at the top (full-width banner). Other images shown in a smaller gallery/carousel below.
- Author card (or anonymous card)
- Category badge
- Date + "düzenlendi" label (if applicable)
- Action menu (if topic owner: Edit, Delete | everyone: Share, Report, Mute)

*Voting section:*
- Two large buttons: "Haklı" (green tone) and "Haksız" (red tone)
- Vote counts + progress bar (percentage distribution)
- Current user's vote highlighted
- After voting → button highlight changes, counters update
- Change vote → click the other button
- Withdraw vote → click the active button again (toggle)
- Own topic → buttons disabled, "Kendi konunuza oy veremezsiniz" tooltip
- Not logged in → clicking buttons → login modal/redirect
- **WebSocket:** `vote:updated` event → update counters in real-time

*Comment section:*
- Comment input area (textarea + emoji picker + anonymous switch)
- Anonymous switch: choice is made on the first comment, then locked (disabled + explanation tooltip)
- Sorting: "En Beğenilen" / "En Yeni" toggle
- Comment list (level 1 comments, replies under each)

### 4. Create Topic Page (`/topics/create`)

Protected with auth guard.

**Form:**
- Title input (character counter: 0/150)
- Content textarea (character counter: 0/3000, min 50)
- Emoji picker button (insert emoji into textarea)
- Category selection (select/dropdown)
- Image upload area:
  - Drag & drop or file picker
  - Uploaded images displayed as thumbnails (max 4)
  - "Remove" button on each image
  - File size/type error messages
  - **Cover photo selection:** After uploading images, each thumbnail shows a "Kapak Yap" button (or star icon). Clicking it sets that image as the cover photo. The selected cover image has a visible "Kapak" badge/overlay. If no cover is manually selected, the first uploaded image is automatically the cover.
  - When editing a topic, the cover selection is preserved. If the cover image is removed, the next image becomes the cover automatically.
  - Cover photo is displayed prominently at the top of the topic detail page and as the thumbnail in topic cards.
- Anonymous post switch + description: "Bu konu anonim olarak paylaşılacak. Bu seçim geri alınamaz."
- "Paylaş" button

**New user check:** If backend returns 403 → "Konu açabilmek için en az bir konuya oy vermeniz gerekiyor" warning + link to redirect to home page.

### 5. Topic Editing

When "Edit" button is clicked on the topic detail page:
- Inline editing or separate modal
- Title and content are editable
- 12-hour restriction: if backend returns 403 → "Düzenleme süresi dolmuştur" toast

### 6. Comment Components

**`src/components/comment/comment-item.tsx`**:
- Author card (or anonymous card)
- Comment text (Twemoji render)
- "düzenlendi" label (if applicable)
- Like button + count (filled heart if liked)
- "Reply" button → opens reply input area
- Action menu (if own comment: Edit, Delete | everyone: Report)
- Deleted comment: "Bu yorum silindi" gray text

**`src/components/comment/comment-reply.tsx`**:
- Level 2 comment component (slightly indented)
- "@username" mention highlight
- Same actions (like, edit, delete, report)

**`src/components/comment/comment-form.tsx`**:
- Textarea + emoji picker
- Anonymous switch (first comment check)
- "Gönder" button
- Character counter

**`src/components/comment/comment-list.tsx`**:
- Render level 1 comments
- Replies under each (first 5 + "X yanıt daha" expand button)
- "Load more comments" button (pagination)

### 7. Report Modal

**`src/components/shared/report-modal.tsx`**:
- Category selection (dropdown)
- Description (optional textarea)
- "Rapor Et" button
- Shared component for topic, comment, or user reporting (`targetType` and `targetId` props)

### 8. Block Confirmation Dialog

**`src/components/shared/block-dialog.tsx`**:
- "Bu kullanıcıyı engellemek istediğinize emin misiniz?" confirmation
- Explanation of blocking effects
- "Engelle" / "İptal" buttons

### 9. Category Page (`/categories/[slug]`)

Same structure as the home page but filtered by a specific category. Category name as page title. Server Component (SEO).

### 10. Search Results Page (`/search`)

**Query parameter:** `q` — search query string

**Structure:**
- Search input (pre-filled with query)
- Results count: "{n} sonuç bulundu" or "Sonuç bulunamadı"
- Topic card list (same component as home page)
- Pagination (same as home page)

**Server Component** for SEO: `<title>"{query}" araması | Hakver</title>`

**Empty state:** "Aramanızla eşleşen konu bulunamadı. Farklı anahtar kelimeler deneyin." with a link to create a new topic.

### 11. WebSocket Integration

On the topic detail page:
- Page mount → send `joinTopic(topicId)` event
- Listen for `vote:updated` → update voting UI
- Listen for `comment:created` → add new comment (or show "X yeni yorum" banner)
- Listen for `comment:deleted` → update comment as "deleted"
- Page unmount → send `leaveTopic(topicId)` event

## Security Checklist
- [ ] Image upload restricted to allowed types/sizes only
- [ ] User content rendered safely against XSS (no dangerouslySetInnerHTML)
- [ ] "Irreversible" warning for anonymous switch is displayed
- [ ] Vote button disabled for own topic
- [ ] Proper redirect when unauthenticated user attempts interaction

## Test Plan

```bash
# 1. Build
cd apps/web && pnpm build

# 2. Browser testing (with backend running):

# Home page
# / → topic list is displayed
# Category filter works
# Sorting works
# Infinite scroll works

# Topic creation
# /topics/create → fill form → upload images → select anonymous → post → redirect to detail page
# New user (never voted) → warning message

# Topic detail
# /topics/:slug → title, content, images, author card, voting, comments are displayed
# Vote → button highlighted, counters updated
# Change vote → click second button → updated
# Withdraw vote → click active button → reset

# Comments
# Write comment → comment appears in list
# Reply → level 2 comment created
# Like → heart fills, counter increases
# Edit → text updated, "düzenlendi" label
# Delete → "Bu yorum silindi" message

# Anonymous
# Anonymous topic → anonymous card displayed, no user info
# Anonymous comment → consistent anonymous identity within same topic

# WebSocket
# Two browser tabs → vote in one → counter updates in real-time in the other

# Reporting
# Report topic/comment/user → modal → select category → submit → toast

# Blocking
# Block user → their topics disappear from feed → unblock → they return

# Mobile responsive
# All pages look correct on mobile
```

## Completion Criteria
- [ ] Home page topic list works (pagination, filtering, sorting)
- [ ] Topic detail page works (info, images, author card)
- [ ] Topic creation works (form, image upload, anonymous, category)
- [ ] Voting UI works (vote, change, withdraw, real-time)
- [ ] Comment system works (create, reply, like, edit, delete)
- [ ] 2-level nesting renders correctly
- [ ] Anonymous cards display correctly
- [ ] Emoji picker works
- [ ] WebSocket real-time updates work
- [ ] Reporting and blocking UI works
- [ ] Mobile responsive
- [ ] `pnpm build` passes without errors
