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

**Notification list:**
- Unread notifications highlighted (background color)
- Each notification: actor avatar/anonymous icon + message + date (relative) + read/unread status
- Click → navigate to the referenced page (topic detail, etc.) + mark as read
- "Tümünü okundu işaretle" button (top bar)
- Infinite scroll

**Notification message formats:**
- TOPIC_VOTED: "[Actor] konunuza oy verdi" → /topics/:slug
- TOPIC_COMMENTED: "[Actor] konunuza yorum yaptı" → /topics/:slug
- COMMENT_LIKED: "[Actor] yorumunuzu beğendi" → /topics/:slug
- COMMENT_REPLIED: "[Actor] yorumunuza yanıt verdi" → /topics/:slug

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
- "Paylaş" button → dropdown: Twitter/X, Facebook, WhatsApp, Bağlantıyı Kopyala
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
- Search no results (future): "Sonuç bulunamadı"

Each with an `empty-state.tsx` component, with appropriate icon and message.

### 11. Accessibility (a11y) Audit

Check across all pages:
- [ ] All images have meaningful `alt` text
- [ ] Form elements have matching `label`
- [ ] Error messages announced with `aria-live`
- [ ] Focus trap works in modals
- [ ] Dropdown menus open and close with keyboard
- [ ] Tab order is logical
- [ ] Color contrast meets WCAG AA (4.5:1 text, 3:1 large text)
- [ ] Skip to content link works
- [ ] Button and link semantics used correctly (clickable → button, navigation → link)

### 12. PostHog Event Tracking

Add event tracking to key user actions (PostHog provider set up in Phase 11):
- `topic_viewed` — when user opens a topic detail page
- `topic_created` — when user creates a new topic
- `vote_cast` — when user votes (with vote type)
- `comment_created` — when user posts a comment
- `search_performed` — when user searches (with query)
- `bookmark_toggled` — when user bookmarks/unbookmarks a topic
- `share_clicked` — when user clicks a share button (with platform)

PostHog tracking respects cookie consent: only fires events if analytics consent is given.

### 12.1. Cookie Consent Note

**MVP behavior:** The app uses localStorage for JWT tokens and essential session data only — no analytics or tracking cookies. A cookie consent banner is NOT required for MVP launch.

**When analytics is added:** If PostHog (or any analytics tool) is enabled, a cookie consent banner must be shown:
- Component: `src/components/shared/cookie-consent.tsx`
- Position: bottom bar, first visit
- Options: "Kabul Et" (all cookies) / "Sadece Zorunlu" (no analytics)
- Consent stored in localStorage: `cookie-consent: "all" | "essential"`
- PostHog only initializes if consent is `"all"`

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
