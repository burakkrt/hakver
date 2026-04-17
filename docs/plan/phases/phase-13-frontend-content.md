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
- `useSetTopicUpdateNote()` → PATCH /topics/:id/update-note (mutation)
- `useClearTopicUpdateNote()` → DELETE /topics/:id/update-note (mutation)
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
- `useBookmarkTopic()` → POST /topics/:id/bookmark (mutation)
- `useUnbookmarkTopic()` → DELETE /topics/:id/bookmark (mutation)
- `usePinTopic()` → PATCH /topics/:id/pin (mutation, moderator+)
- `useUserSearch(query)` → GET /users/search?q={query}&limit=5 (query, debounced — for @mention autocomplete)
- `useCreateComment()` → POST /topics/:topicId/comments (mutation)
- `useUpdateComment()` → PATCH /comments/:id (mutation)
- `useDeleteComment()` → DELETE /comments/:id (mutation)
- `useLikeComment()` → POST /comments/:id/like (mutation)
- `useUnlikeComment()` → DELETE /comments/:id/like (mutation)
- `useHighlightComment()` → PATCH /comments/:id/highlight (mutation — topic author only)
- `useUnhighlightComment()` → DELETE /comments/:id/highlight (mutation — topic author only)

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
- Sorting: "En Yeni" / "Popüler" / "Trend" / "Tartışmalı" tab or toggle. Default: "Trend" (time-weighted engagement score). "Tartışmalı" surfaces topics where the Hakli/Haksiz split is close to 50/50 and engagement is real — core Hakver identity move. Both "Trend" and "Tartışmalı" read the precomputed scores described in Phase 5 Section 10 so the listing stays cheap at scale
- Topic card list (card grid or list view)

**Topic card component** (`src/components/topic/topic-card.tsx`):
- Title (truncated)
- Content preview (first 150 characters)
- Author card (user-card or anonymous-card)
- Category badge
- Vote status: Haklı / Haksız buttons + counts (small version)
- **Majority verdict pill** (derived client-side from `voteCountRight` and `voteCountWrong`): renders as a small pill next to the counts. When total votes ≥ 10 (minimum engagement gate), compute `total = voteCountRight + voteCountWrong` and `rightRatio = voteCountRight / total`:
  - `rightRatio ≥ 0.60` → green pill "%{round(rightRatio * 100)} Haklı"
  - `rightRatio ≤ 0.40` → red/coral pill "%{round((1 - rightRatio) * 100)} Haksız"
  - Otherwise → neutral pill "Kararsız"
  - Total votes < 10 → pill is hidden entirely so early topics are not prematurely labeled
  - No backend change — the pill is a pure render derivation from fields the card already receives. Adds a clear "Hakli/Haksiz" identity signal to each card at zero infrastructure cost
- Comment count icon + count — renders `commentCount` (level-1 comments only). Replies are not included so the number on the card always matches the top-level comment rows the user sees when they open the topic
- Vote indicator: if authenticated, a small colored dot or icon showing the user's vote (green dot = Hakli, red dot = Haksiz, no dot = not voted). This comes from the `myVote` field in the topic list API response.
- Image thumbnail (if available, first image)
- Date (relative: "2 saat önce")
- Bookmark icon (filled if bookmarked, click to toggle)
- Pinned badge (if isPinned — "Sabitlenmiş" label, pinned topics shown at top of list)
- Update badge (if `hasUpdate` — "Güncellendi" label, signals that the author added a post-window update note)
- **`isEdited` is NOT surfaced on the card.** Edit history is meta-information; showing both "Güncellendi" and "Düzenlendi" at the same time clutters the card and dilutes the signal. The `isEdited` state only appears on the topic detail page (as a small "düzenlendi" hint next to the date). Cards prioritize the "Güncellendi" badge when `hasUpdate` is true
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
- Update-note section (if `updateNote` is populated) — rendered below the original content as a visually distinct block titled "Güncelleme", showing the update text (Twemoji render) and the relative `updateNoteAt` timestamp
- Action menu (if topic owner: Edit (within edit window), Güncelleme Ekle/Düzenle (any time — opens the update-note modal), Delete | non-author users: Takip Et / Takibi Bırak (toggles `TopicSubscription` so the user opts in to or out of `TOPIC_UPDATED` notifications; default state off), Share, Report, Mute, Bookmark | moderator+: Pin/Unpin). If the topic is outside its edit window and the user is the author, "Edit" is hidden and only "Güncelleme Ekle/Düzenle" is shown
- Subscription hint: non-authors who have not yet subscribed see a small inline pill next to the action menu "Bu konuya güncelleme bildirimleri almıyorsunuz" with a "Takip Et" shortcut. Authors and existing subscribers see "Güncellemeler için aboneliğiniz açık" / "Kendi konunuza aboneliğiniz açık" respectively. The pill disappears after the user subscribes to keep the detail page uncluttered

*Voting section:*
- Two large buttons: "Haklı" (green tone) and "Haksız" (red tone)
- Vote counts + progress bar (percentage distribution) — reuses the same percentage math as the topic card verdict pill (Section 2). On detail we always show the full percentage bar; the verdict pill from the card is additionally rendered above the buttons once `total ≥ 10` so the summary stays consistent across surfaces
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
- @mention autocomplete: typing `@` shows dropdown with matching usernames (debounced 300ms). Selected username inserted as `@username`. Mentioned usernames rendered as highlighted links to user profiles.
- Sorting: "En Beğenilen" / "En Yeni" toggle
- Comment list (level 1 comments, replies under each)
- **Focus comment handling (from notification click):** The page reads the `focusCommentId` query parameter on mount and forwards it to `useComments()` → `GET /topics/:topicId/comments?focusCommentId=...`. The server pins that comment to the top of the payload (Phase 7 Section 7). After the list renders, the page scrolls to the `#comment-{id}` anchor. When the user refreshes or navigates away and comes back without the query parameter, the list renders in the standard order. The query parameter is intentionally sticky — it is not stripped from the URL so the user can share the URL with the focus intent preserved
- **Viewer-scoped own-comment pinning:** The comment list component reads the `isOwnComment` flag on each response row; within the highlight group the authenticated user's own comments float above the rest. This is per-viewer (other users see the normal order). The server already returns rows in this order for authenticated requests (Phase 7 Section 7 — "Viewer-scoped own-comment layer"), so the frontend only needs to render the order it receives

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

**Draft auto-save (localStorage, user-scoped, 7-day TTL):**

A stray refresh or an accidental navigation wipes 10 minutes of writing. The create form mitigates that with a local draft without introducing a server-side draft table.

- Storage key: `topic-draft:{userId}` — scoped to the authenticated user's id so a shared browser does not leak drafts across accounts. Unauthenticated visitors have no draft slot (the page redirects to login before reaching the form)
- Payload shape: `{ title, content, categoryId, isAnonymous, coverImageHint, updatedAt }`. Uploaded images are not persisted in localStorage (they already live in Cloudinary). `coverImageHint` stores the cover image's id so the preview reattaches when the user returns
- Save trigger: debounced 1 second after the last keystroke on any controlled input. Also saves on `visibilitychange` (tab hides) and `beforeunload`
- Restore trigger: on form mount, if a draft exists for the current user, show a subtle inline banner "Kaldığınız yerden devam edin" with two buttons `"Kaldığım yerden devam et"` / `"Yeni başla"`. Restore merges the draft into the form state; "Yeni başla" deletes the slot
- Clear trigger: successful submit deletes the slot. Explicit cancel from the author also deletes after confirmation
- TTL guard: on restore, if `updatedAt` is older than 7 days, the banner is suppressed and the slot is deleted silently — stale drafts never auto-populate
- Payload size cap: the serialized draft is size-limited at 8 KB (roughly twice the max body length in characters); exceeding the limit drops the oldest writes first. This prevents pathological growth from stripping out the storage quota

**Security hardening:**
- Never persist any token, refresh cookie reference, or credential in the draft slot; only the raw content fields
- On logout, `clearAuth()` in the auth store also sweeps every `topic-draft:*` key to remove authored drafts left behind by the user who signed out
- The draft feature is strictly local to the device; Phase 12 settings do not surface it — the KVKK export already covers on-server data

### 5. Topic Editing

When "Edit" button is clicked on the topic detail page:
- Inline editing or separate modal
- Title and content are editable
- 12-hour restriction: if backend returns 403 → "Düzenleme süresi dolmuştur" toast

### 5.1. Update Note Modal

`src/components/topic/topic-update-note-modal.tsx` — opened from the author's "Güncelleme Ekle" or "Güncelleme Düzenle" action.

- Textarea with a 0/500 character counter, Turkish helper text explaining that the note appears below the original content and notifies voters and commenters
- Submit button "Kaydet" (disabled while empty and unchanged)
- If the note already exists, a secondary "Güncellemeyi Kaldır" button calls `useClearTopicUpdateNote()` after a confirmation dialog
- Success → close modal, refresh the topic query, show toast "Güncellemeniz paylaşıldı" (or "Güncelleme kaldırıldı" on clear)
- Errors: 429 → "Çok sık güncelleme yapıyorsunuz, biraz bekleyin" toast; 403 → "Bu konuya güncelleme ekleyemezsiniz" toast
- The modal is accessible (focus trap, Escape to close, ARIA labels)

### 6. Comment Components

**`src/components/comment/comment-item.tsx`**:
- Author card (or anonymous card)
- **"Konu Sahibi" badge** (when `isAuthor === true` on the response): small, professional pill rendered next to the author card — muted primary-tone background, small label "Konu Sahibi", Lucide icon (e.g., user-check). Never shown on anonymous topics (the `isAuthor` flag is server-side forced to `false` there)
- **"Öne Çıkarılan" badge** (when `isHighlighted === true`): rendered on the comment card with a star icon (Lucide `star`, filled, primary-tone). The whole comment card gets a subtle accent border/background so the highlight is visually obvious in a long list. Tooltip on the star: "Konu sahibi bu yorumu öne çıkardı"
- Comment text (Twemoji render)
- "düzenlendi" label (if applicable) — small meta hint, rendered subtly next to the date. Remains on comments on both cards and detail, unlike topic cards which hide it (comments don't have an independent "güncellendi" signal)
- Like button + count (filled heart if liked)
- "Reply" button → opens reply input area
- Action menu:
  - If own comment: Edit, Delete, Report
  - Everyone: Report
  - **If viewer is the topic author** (and the comment is on their topic): additional "Öne Çıkar" / "Öne Çıkarmayı Kaldır" menu item. Button label toggles based on current `isHighlighted` state. Kept inside the action menu (not as a prominent button) to match the discreet professional pattern — highlighting is a considered action, not an impulse button
  - Confirmation toast after highlight: "Yorum öne çıkarıldı" / "Öne çıkarma kaldırıldı"
- Deleted comment: "Bu yorum silindi" gray text
- If `isAuthor` and `isHighlighted` are both true, both badges show side by side (topic author highlighted their own follow-up) — both pills sit on the same row, spacing consistent

**`src/components/comment/comment-reply.tsx`**:
- Level 2 comment component (slightly indented)
- "@username" mention highlight
- Same isAuthor and isHighlighted treatment as level 1 (reply from topic author gets the "Konu Sahibi" badge; replies can also be highlighted)
- Same actions (like, edit, delete, report, and the topic-author-only highlight toggle)

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

# Update note
# Topic detail (author, after edit window) → "Güncelleme Ekle" action → modal → type 200 chars → Kaydet → "Güncelleme" block appears below content with relative timestamp
# Card list → topic shows "Güncellendi" badge
# Voter receives a `TOPIC_UPDATED` notification in real-time

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
- [ ] "Konu Sahibi" badge renders on comments written by the topic author (non-anonymous topics only)
- [ ] "Öne Çıkarılan" star badge renders on highlighted comments; highlighted comments float to the top of the comment list regardless of sort
- [ ] Topic author can highlight/unhighlight comments from the action menu; toast confirmation shown
- [ ] Update-note modal works (add, edit, clear) and the "Güncelleme" block renders on the detail page with the "Güncellendi" badge on cards
- [ ] "Düzenlendi" label suppressed on topic cards and only shown on topic detail page
- [ ] 2-level nesting renders correctly
- [ ] Anonymous cards display correctly
- [ ] Deleted author cards render as "Silinmiş Kullanıcı" (no real username/avatar leak)
- [ ] Emoji picker works
- [ ] WebSocket real-time updates work
- [ ] Reporting and blocking UI works
- [ ] Mobile responsive
- [ ] `pnpm build` passes without errors
