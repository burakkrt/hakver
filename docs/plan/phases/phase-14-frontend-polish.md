# Phase 14: Frontend — Polish, SEO & Social

## Dependencies
- Phase 10 (notifications backend) and Phase 13 (frontend content) must be completed

## Goals
- Notification page and real-time notification UI
- SEO optimization (meta tags, OG, structured data)
- Social media share cards and buttons
- Legal text page content
- Accessibility (WCAG AA) audit and fixes
- Completion of loading, error, and empty states
- 404 and error pages
- General responsive polish

## Tasks

### 1. Notification Hooks

`src/hooks/use-notifications.ts`:
- `useNotifications()` → GET /notifications (infinite query)
- `useUnreadCount()` → GET /notifications/unread-count (query, polling or WebSocket)
- `useMarkAsRead()` → PATCH /notifications/:id/read (mutation)
- `useMarkAllAsRead()` → PATCH /notifications/read-all (mutation)
- `useMuteTopic()` → POST /topics/:topicId/mute (mutation)
- `useUnmuteTopic()` → DELETE /topics/:topicId/mute (mutation)

### 2. Notifications Page (`/notifications`)

**Notification list (flat chronological list with visual importance hierarchy):**
- Single list — no tabs or grouping. Chronological order preserved (`updatedAt DESC`; aggregated cards sort by their latest actor's time)
- Unread notifications get a highlighted background tint
- **Important types** (`MENTIONED`, `COMMENT_HIGHLIGHTED`, `TOPIC_UPDATED`) are visually elevated: primary-tone vertical accent bar on the left edge of the card, primary-tone icon, and a soft background tint. Engagement types (`TOPIC_VOTED`, `TOPIC_COMMENTED`, `COMMENT_LIKED`, `COMMENT_REPLIED`) use the plain standard card style. The sort order stays chronological, but important cards catch the eye
- **`RANK_UP` type** — rendered with a celebration-style card: the avatar slot is replaced with the `<RankBadge size="md" />` of the newly reached rank (the frontend reads `reference.title` for the rank name and looks up `iconSlug`/`color` from the local ranks cache). Background uses the rank's `color` token at a softer tint, and a subtle sparkle decoration sits in the corner. Headline = `message` ("Yeni rütbe kazandın: [Rank Name]"). Secondary action link: `"Avatarları incele"` → `/profile/settings#avatar`. `actors` is empty for this type; the card does not render an actor avatar stack. Stale handling (`isStale`) does not apply — ranks never soft-delete
- **Aggregated card rendering:** when `aggregatedCount > 1`, the last 3 actor avatars stack horizontally with slight overlap and the headline shows "ayse_k ve {count-1} kullanıcı ...". For a single actor (`aggregatedCount === 1`) a single avatar + the standard message is shown
- **Anonymous aggregate:** when the payload carries `isAnonymousAggregate: true`, a generic anonymous icon (mask) replaces the avatar and the headline becomes "Bir kullanıcı ..." or "12 kullanıcı ..." depending on count
- **Stale notification (L-4):** cards with `isStale: true` render in grey/italic, avatar desaturated. Clicking does not navigate — a short info toast "Bu içerik artık mevcut değil" is shown instead. The card stays visible so the user keeps the history, but no actions are available
- Each notification shows: actor avatar (or anonymous/stale variant) + message + relative date + read/unread status
- Click → navigate to the referenced page (topic detail, etc.) + mark as read. For stale notifications navigation is skipped and the info toast is shown
- "Tümünü okundu işaretle" button (top bar — user-facing Turkish label)
- Infinite scroll

**Notification preferences link:** a small "Bildirim Ayarları" button at the top of the page opens `/profile/settings#notifications` (L-2, Phase 12).

**Notification message formats** (authoritative source is Phase 10; this list mirrors it for frontend reference):
- TOPIC_VOTED: "[Actor] konunuza oy verdi" → `/topics/:slug`
- TOPIC_COMMENTED: "[Actor] konunuza yorum yaptı" → `/topics/:slug`
- COMMENT_LIKED: "[Actor] yorumunuzu beğendi" → `/topics/:slug?focusCommentId={id}#comment-{id}`
- COMMENT_REPLIED: "[Actor] yorumunuza yanıt verdi" → `/topics/:slug?focusCommentId={id}#comment-{id}`
- MENTIONED (non-anonymous actor): "[Actor] sizi bir yorumda etiketledi" → `/topics/:slug?focusCommentId={id}#comment-{id}`
- MENTIONED (anonymous actor): "Bir kullanıcı sizi bir yorumda etiketledi" → same target URL; the anonymous actor's display name is deliberately not surfaced in the notification (see Phase 10 Section 4 "Anonymous actor privacy rule")
- TOPIC_UPDATED: "[Actor] ilgilendiğiniz konuya bir güncelleme ekledi" → `/topics/:slug#topic-update`
- COMMENT_HIGHLIGHTED: "[Actor] yorumunuzu öne çıkardı" → `/topics/:slug?focusCommentId={id}#comment-{id}`

**Focus behaviour:** Every comment-scoped notification URL carries the `focusCommentId` query parameter alongside the `#comment-{id}` anchor. The topic detail page (Phase 13) reads the query parameter, forwards it to `GET /topics/:topicId/comments?focusCommentId=...`, and the server pins the referenced comment to the first position of the response for that request only (see Phase 7 Section 7). The browser's native anchor scroll then brings the pinned card into view. A refresh that keeps the URL keeps the focus; a manual navigation that drops the parameter (the user scrolls to another topic and comes back via the feed) reverts to the standard listing order.

**Note:** Notifications older than 90 days are automatically cleaned up by the backend cron job (Phase 10). The frontend doesn't need to handle this — it simply fetches whatever notifications exist.

### 3. Header Notification Icon

Notification icon in the header:
- Unread count badge (red circle, number)
- Click → dropdown with last 5 notifications + "Tümünü gör" link
- **WebSocket:** `notification:new` event → increment badge count, show new notification in dropdown
- `notification:unread-count` event → update count

### 4. Mute on Topic Detail

Add to the action menu on the topic detail page:
- "Bildirimleri Sessize Al" / "Sessize Almayı Kaldır" toggle
- `isMuted` state comes from the topic detail response

### 5. SEO Optimization

**`src/app/layout.tsx`** — Global metadata:
```typescript
export const metadata: Metadata = {
  title: { default: 'Hakver', template: '%s | Hakver' },
  description: 'Haklı mısın, haksız mısın? Topluluğa sor.',
  // ... other global meta
}
```

**JSON-LD Structured Data** — Add Article schema to topic detail pages, Person schema to profile pages. For Google Rich Results compatibility.

**Topic detail page** — Dynamic metadata:
```typescript
export async function generateMetadata({ params }): Promise<Metadata> {
  const topic = await fetchTopic(params.id);
  return {
    title: topic.title,
    description: topic.content.substring(0, 160),
    openGraph: {
      title: topic.title,
      description: topic.content.substring(0, 160),
      type: 'article',
      images: topic.images?.[0]?.url ? [{ url: topic.images[0].url }] : [],
    },
    twitter: {
      card: 'summary_large_image',
      title: topic.title,
      description: topic.content.substring(0, 160),
    },
  }
}
```

**Category page** — Dynamic metadata: category name and description.

**User profile** — Dynamic metadata: username and bio.

**robots.txt** and **sitemap.xml** (under `src/app/`):
- robots.txt: allow all bots, sitemap link
- sitemap.xml: dynamic sitemap — topics, categories, profiles

### 6. Social Media Sharing

**`src/components/shared/share-button.tsx`**:
- "Paylaş" button → dropdown: Twitter/X, Facebook, WhatsApp, "Bağlantıyı Kopyala"
- Generate share URL for each platform:
  - Twitter: `https://twitter.com/intent/tweet?url={url}&text={title}`
  - Facebook: `https://www.facebook.com/sharer/sharer.php?u={url}`
  - WhatsApp: `https://wa.me/?text={title} {url}`
  - Copy: clipboard API

"Paylaş" button on topic detail and topic cards.

**Dynamic OG Image (Vercel OG):**
Generate a dynamic OG image using `@vercel/og` for eye-catching social media cards when a topic is shared:
- Topic title
- Vote distribution (Haklı %X / Haksız %Y)
- Comment count
- Platform logo
Route: `src/app/api/og/route.tsx` — SVG-based card generation with ImageResponse.

### 7. Legal Text Page Content

**`/terms-of-service`**, **`/privacy-policy`**, **`/community-guidelines`** pages:

- Server Component — SSR for SEO
- Content fetched from database (`ConsentVersion`)
- Text written in professional, formal Turkish
- Last updated date displayed

**Content was created in Phase 2 seed data.** In this phase, complete the frontend rendering of the pages.

### 8. Error and State Pages

**`src/app/not-found.tsx`** — 404 page:
- "Aradığınız sayfa bulunamadı" message
- Return to home page button
- Brand-consistent design

**`src/app/error.tsx`** — General error page:
- "Bir hata oluştu" message
- "Tekrar Dene" button
- Error details shown in development mode

**`src/app/loading.tsx`** — Global loading:
- Skeleton or spinner

### 9. Loading States

On all data-fetching pages and components:
- Skeleton loaders (topic card skeleton, comment skeleton, profile skeleton)
- Initial load: show skeleton
- Next page load: small spinner at the bottom
- Mutation (voting, posting comment): button loading state

### 10. Empty States

- Home page empty: "Henüz konu paylaşılmamış"
- Profile topics empty: "Henüz konu açmamış"
- Comments empty: "İlk yorumu siz yapın"
- Notifications empty: "Bildiriminiz yok"
- Search no results: "Aramanızla eşleşen konu bulunamadı. Farklı anahtar kelimeler deneyin." — implemented in Phase 13 (search results page). Phase 14 audits the component and ensures the shared `empty-state.tsx` renders it consistently

Each with an `empty-state.tsx` component, with appropriate icon and message.

### 11. Accessibility (a11y) Audit

Check across all pages:
- [ ] All images have meaningful `alt` text
- [ ] Form elements have matching `label`
- [ ] Error messages announced with `aria-live`
- [ ] Focus trap works in modals (including `<RankUpModal />`)
- [ ] Dropdown menus open and close with keyboard
- [ ] Tab order is logical
- [ ] Color contrast meets WCAG AA (4.5:1 text, 3:1 large text) — verify every `<RankBadge />` color token (zinc, violet, blue, indigo, emerald, amber, rose, gold) against both light and dark backgrounds
- [ ] Skip to content link works
- [ ] Button and link semantics used correctly (clickable → button, navigation → link)
- [ ] Avatar grid: radio-group semantics, keyboard arrow navigation, locked cards expose `aria-disabled="true"` with a descriptive `aria-label`, hover-reveal also triggers on keyboard focus so sighted keyboard users see the preview
- [ ] `<RankProgress />` exposes `role="progressbar"` plus `aria-valuenow`/`aria-valuemin`/`aria-valuemax`

### 12. PostHog Event Tracking

Add event tracking to key user actions (PostHog provider set up in Phase 11):
- `topic_viewed` — when user opens a topic detail page
- `topic_created` — when user creates a new topic
- `vote_cast` — when user votes (with vote type)
- `comment_created` — when user posts a comment
- `search_performed` — when user searches (with query)
- `bookmark_toggled` — when user bookmarks/unbookmarks a topic
- `share_clicked` — when user clicks a share button (with platform)
- `rank_up` — fired from the rank-up celebration flow (Phase 12 Section 13) once per notification key. Properties: `{ previousRank: string, newRank: string, totalXp: number, daysSinceRegistration: number, trigger: "organic" | "admin-adjust" | "reconciliation" }`. Guarded by the same Zustand celebration store that backs `<RankUpModal />` so retries do not double-fire

PostHog tracking respects cookie consent: only fires events if analytics consent is given.

### 12.1. Cookie Consent Banner (shipped in Phase 11)

The cookie consent banner and PostHog analytics provider were shipped together in Phase 11 Section 11.1. This section audits that they are still wired correctly at the end of the polish phase and documents the remaining follow-ups.

- The banner surface has been live since Phase 11; Phase 14's role is to confirm copy, focus behaviour, keyboard dismissal, and dark-mode rendering across every landing route
- Settings → "Çerez Tercihleri" entry point: verify that clicking it re-opens the banner and that switching consent state (`"all"` ↔ `"essential"`) triggers the PostHog provider's teardown / re-init flow from Phase 11
- Copy: heading `"Çerez Tercihleri"`, body `"Platformu geliştirmek ve deneyiminizi iyileştirmek için çerez kullanıyoruz."`, buttons `"Kabul Et"` / `"Sadece Zorunlu"`
- PostHog remains dormant until `cookie-consent === "all"` — no telemetry leaves the browser before consent (the Phase 14 QA suite includes a network-tab assertion on a fresh visit)

### 12.2. Static Pages (MDX)

"Hakkımızda" and "SSS (Sıkça Sorulan Sorular)" pages are static content managed via MDX files in the Next.js project:
- `src/app/about/page.mdx` — About page
- `src/app/faq/page.mdx` — FAQ page
- MDX files are committed to the repo and deployed with the app
- No admin CMS needed — content changes require a deploy
- Server Component rendering for SEO

### 13. Responsive Polish

Final check for all pages:
- Mobile (375px) — primary
- Tablet (768px)
- Desktop (1024px+)

Key areas to watch:
- Topic cards: full width on mobile, grid on desktop
- Comment nesting: collapsed indentation on mobile
- Vote buttons: full width on mobile
- Header: hamburger on mobile, full menu on desktop
- Image gallery: swipe on mobile, grid on desktop

## Security Checklist
- [ ] User-entered content in SEO meta tags is escaped
- [ ] Social share URLs are encoded
- [ ] Sensitive pages (settings, etc.) are not blocked in robots.txt (already auth-protected)

## Test Plan

```bash
# 1. Build
cd apps/web && pnpm build

# 2. Browser testing:

# Notifications
# /notifications → notification list
# Another user votes → badge increases in header, notification in dropdown
# "Tümünü okundu işaretle" → badge resets
# Mute → no notifications received

# SEO
# view-source:/topics/:slug → OG tags present?
# Check with Google Rich Results Test

# Sharing
# On topic detail "Paylaş" → Twitter/Facebook/WhatsApp links work
# "Bağlantıyı Kopyala" → URL in clipboard

# Legal texts
# /terms-of-service, /privacy-policy, /community-guidelines → content displayed

# 404
# /olmayan-sayfa → 404 page

# Loading
# On slow connection, are skeletons visible?

# Empty state
# Empty profile topics → empty state message

# Accessibility
# Tab navigation across all pages
# a11y score check with axe DevTools or Lighthouse
# Basic flows with screen reader

# Responsive
# Chrome DevTools → Mobile/Tablet/Desktop
# No broken layouts on any page
```

## Completion Criteria
- [ ] Notification page and real-time notifications work
- [ ] Header notification badge works
- [ ] Topic-level muting works
- [ ] SEO: dynamic meta tags, OG tags, sitemap
- [ ] Social media sharing works
- [ ] Legal text pages are displayed
- [ ] 404 and error pages work
- [ ] Loading skeletons on all pages
- [ ] Empty states for all empty conditions
- [ ] Accessibility: Lighthouse a11y score 90+
- [ ] Responsive: no breakage on mobile/tablet/desktop
- [ ] Search results page works with relevant results
- [ ] `pnpm build` passes without errors
