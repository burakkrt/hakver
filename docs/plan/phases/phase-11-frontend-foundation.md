# Phase 11: Frontend Foundation & Design System

## Dependencies
- Phase 1 must be completed (Next.js app scaffolding)

## Goals
- shadcn/ui + Tailwind CSS design system setup
- Layout components (header, mobile nav, page structure)
- API client (axios/fetch wrapper)
- TanStack Query provider
- Zustand auth store
- Socket.io client provider
- Twemoji integration
- Mobile-first responsive base structure
- Accessibility (WCAG AA) base setup
- Shared UI components (Button, Input, Card, Modal, Toast, Skeleton, Avatar, Badge)

## Tasks

### 1. shadcn/ui Setup

Init with shadcn/ui CLI:
- Style: Default
- Base color: color palette suitable for the project (to be decided, initially slate or zinc)
- CSS variables: Yes

Add the following shadcn/ui components:
`button`, `input`, `textarea`, `label`, `card`, `dialog`, `dropdown-menu`, `avatar`, `badge`, `toast`, `tabs`, `select`, `switch`, `checkbox`, `separator`, `skeleton`, `sheet` (mobile sidebar), `popover`, `tooltip`, `form`

### 1.1. Dark Mode (next-themes)

**Dark mode toggle:**
- Install `next-themes`
- Configure `ThemeProvider` in root layout with `attribute="class"` (for Tailwind `dark:` prefix)
- Default theme: system preference (`defaultTheme: "system"`)
- shadcn/ui already supports dark mode via CSS variables — no extra component work needed
- Theme toggle button in the header (sun/moon icon)
- Store preference in localStorage (next-themes handles this automatically)
- All custom Tailwind colors must have `dark:` variants defined in `tailwind.config.ts`

### 2. Tailwind Configuration

**`tailwind.config.ts`** extensions:
- Custom colors: primary, secondary, accent (brand colors)
- Font: **Nunito** (Google Fonts) — rounded, friendly, readable; matches the "Social, warm, friendly" direction in `.claude/rules/design-system.md` Typography. Authority for the font family is the design-system rule; this phase consumes it
- Container: centered container with responsive breakpoints
- Animations: fade-in, slide-in (for transitions)

### 3. Global Styles and Font

**`src/app/globals.css`**:
- Tailwind base/components/utilities
- CSS custom properties (color theme)
- Default typography

**Font loading:** Define with `next/font/google` using **Nunito** in `layout.tsx`. Weight subset: `400` (body), `600` (button), `700` (heading) — mirrors the weight guidance in `.claude/rules/design-system.md` Typography. Apply via `--font-nunito` CSS variable so Tailwind's `font-sans` resolves to Nunito app-wide.

### 4. Layout Components

**`src/app/layout.tsx`** (root layout):
- Font and metadata
- Provider wrapping (QueryProvider, AuthProvider, SocketProvider, Toaster)

**`src/components/layout/header.tsx`**:
- Logo (left)
- Search bar (center) — text input with search icon. On submit, navigates to `/search?q={query}`. Debounced autocomplete dropdown showing top 5 matching topic titles (calls `GET /topics?search={query}&limit=5`). On mobile: search icon that expands to full-width input.
- Right menu based on auth state:
  - Not logged in → Giriş Yap / Kayıt Ol buttons
  - Logged in → Notification icon (badge with unread count) + Avatar dropdown (Profil, Ayarlar, Çıkış)
- Mobile: hamburger menu → Sheet (slide-in sidebar)

**`src/components/layout/mobile-nav.tsx`**:
- Bottom navigation bar (mobile): Ana Sayfa, Keşfet/Kategoriler, Konu Aç (+), Bildirimler, Profil
- Only visible on mobile screens

**`src/components/layout/page-container.tsx`**:
- Max-width container, padding, responsive

### 5. API Client

**`src/lib/api.ts`**:
- Base URL: `NEXT_PUBLIC_API_URL`
- Axios instance (or fetch wrapper) configured with `withCredentials: true` / `credentials: "include"` so the backend-issued refresh cookie is attached on every request to the auth endpoints
- Request interceptor: attaches `Authorization: Bearer {accessToken}` header from the in-memory auth store
- Response interceptor:
  - `401 AUTH_TOKEN_EXPIRED` → call `POST /auth/refresh` (cookie is sent automatically); on success, store the new access token in memory and retry the original request once
  - Refresh itself returns `401` → `clearAuth()`, redirect to `/login` (the refresh cookie has already been invalidated by the backend; no client-side cookie clear needed)
- **Refresh coalescing (authoritative):** the client holds a module-scoped `inFlightRefresh: Promise<string> | null`. When `null`, a `401` hit starts a single `POST /auth/refresh`, assigns the promise, and retries the original request once the promise resolves. Any other request that hits `401` while the promise is live **awaits the same promise** instead of issuing its own refresh. The promise is cleared in both the success and failure branches. This prevents multi-tab / parallel-request races where two simultaneous refresh calls would rotate the refresh cookie twice and invalidate each other's session. Aligns with Phase 3 Section 3's refresh-flow description
- Error format standardization around the shared `ApiErrorResponse` shape
- **Important:** never read or write the refresh token from JavaScript — it lives exclusively in the `httpOnly; Secure; SameSite=Strict` cookie set by the backend (see Phase 3 Section 3). The auth store keeps only the access token and the user snapshot

### 6. Auth Store (Zustand)

**`src/stores/auth-store.ts`**:

```typescript
interface AuthState {
  user: User | null;
  accessToken: string | null;      // in-memory only — never persisted to localStorage
  isAuthenticated: boolean;
  isLoading: boolean;
  setAuth: (data: { user, accessToken }) => void;
  clearAuth: () => void;
  updateUser: (data: Partial<User>) => void;
}
```

The access token is held in memory only. The refresh token is **not** in this store — it lives server-side in an `httpOnly; Secure; SameSite=Strict` cookie (Phase 3 Section 3) that JavaScript cannot read or write. This removes the XSS-exfiltration surface that came with storing both tokens in `localStorage` in earlier drafts.

Zustand persist middleware persists only the `user` snapshot (id, username, avatar, rank, flags) so the header renders the logged-in state immediately on page reload. On mount the `AuthProvider` (below) rehydrates the access token by calling `POST /auth/refresh` — the browser sends the refresh cookie automatically, and the backend returns a fresh access token. If the refresh cookie is missing or invalid the response is `401` and `clearAuth()` runs, which also trims the persisted user snapshot.

### 7. Auth Provider

**`src/providers/auth-provider.tsx`**:
- On app mount, issue a silent `POST /auth/refresh` request with `credentials: "include"`. The browser attaches the httpOnly refresh cookie automatically; if the cookie is present and valid, the response carries a fresh access token and the user snapshot, and the store is hydrated
- If the refresh returns 401 (missing / expired cookie), the store stays cleared and any persisted `user` snapshot is dropped so the UI renders the anonymous state from the very first paint
- Loading spinner while the refresh is in flight; component children read from the resolved `isAuthenticated` afterwards
- Profile completion check: if the resolved user has `username`, `firstName`, or `lastName` null → set `requiresProfileCompletion: true` in auth state. Auth guard redirects to `/complete-profile` when a write action is attempted
- The provider wires the Socket.io provider to reconnect with the latest access token whenever it rotates, so the WebSocket periodic re-auth (Phase 6 Section 7) always sees the fresh token

### 8. TanStack Query Provider

**`src/providers/query-provider.tsx`**:
- `QueryClient` configuration
- Default options: `staleTime: 5 * 60 * 1000`, `retry: 1`
- Error handling: global error toast

### 9. Socket.io Client Provider

**`src/providers/socket-provider.tsx`**:
- Socket.io client connection: `NEXT_PUBLIC_SOCKET_URL`
- Connection with auth token: `auth: { token: accessToken }`
- Connect when authenticated, disconnect on logout
- Reconnect strategy
- Share socket instance via Context API

### 10. Twemoji Integration

**Self-hosted SVG assets (no CDN):**
- Twemoji SVG asset pack is copied into `apps/web/public/twemoji/` during the build (or via a one-time setup script that reads from the `@twemoji/svg` npm package). Assets are committed/released with the app — they are not fetched from a third-party CDN at runtime
- This keeps the CSP tight (no external `img-src` allowlist entry is needed for Twemoji) and guarantees offline-friendly rendering
- The Next.js image optimizer serves these assets; Tailwind config already allows `/public` as a same-origin source

**`src/lib/twemoji.ts`**:
- Twemoji parse function: converts Unicode emojis in text to Twemoji SVGs. The parser is configured with `base: "/twemoji/"` and `ext: ".svg"` so it points at the self-hosted copies
- React component: `<TwemojiText text="..." />` — renders emojis in text using the self-hosted assets

**Emoji Picker:**
- Emoji picker component with `emoji-mart` library (itself bundled locally, no external image fetch)
- `<EmojiPicker onSelect={(emoji) => ...} />`
- Shown inside a popover

### 11. Shared Components

**`src/components/shared/user-card.tsx`**:
- Compact / card variant (used in topic lists, topic detail author area, comment lists, notification actors, vote lists): Avatar + username + rank badge + totalXp. No firstName/lastName. Renders `UserPublicCardSchema`
- Full / profile header variant (used only on `/profile/[username]`): Avatar + username + rank badge + totalXp + the formatted name "Ahmet D." (firstName full + lastName first character + "."). Owner of the profile sees the full surname instead. Renders `UserPublicResponseSchema` for others, `UserPrivateResponseSchema` for owner
- Anonymous variant: animal icon, displayName (e.g., "Anonim Penguen #4521") — rank/XP suppressed
- **Deleted-author variant** (`isDeletedAuthor === true`): Render the card with the default "silinmis" avatar, the fixed label "Silinmiş Kullanıcı" in place of the username, and suppress rank and XP. The card remains clickable only inside admin views (which have separate handling); in the public feed, the card is non-interactive and does not link to a profile (the unavailable-profile page is surfaced from the global profile route handler instead)
- The compact variant must never receive firstName/lastName props — this is a platform-wide identity rule

**`src/components/shared/anonymous-card.tsx`**:
- Animal icon + "Anonim Penguen #4521" displayName
- Rank not shown

**`src/components/shared/rank-badge.tsx`**:
- Renders rank metadata as a tinted pill: Lucide icon (via `iconSlug`) + rank name text, with background/text color driven by the `color` semantic token from the `Rank` row
- Props: `rank: { name: string, iconSlug: string, color: string }`, `size?: "sm" | "md" | "lg"` (default `"md"`)
- Consumed by `user-card.tsx` (compact + profile variants), topic/comment author areas, the notification list RANK_UP card, the profile page header, and the admin user detail page
- Color token map lives in `src/lib/rank-colors.ts`: `{ zinc, violet, blue, indigo, emerald, amber, rose, gold }` → Tailwind class bundles for light + dark mode. Icons resolve through a small `src/lib/rank-icons.ts` map that statically imports the required Lucide components (`eye`, `lightbulb`, `scale`, `gavel`, `book-open`, `landmark`, `sparkles`, `crown`) so the tree-shaker keeps bundle size minimal
- Accessibility: `aria-label="[Rank.name] rütbesi"`; when inside interactive contexts, the rank pill is never the only target — the parent link/button holds the semantic role

**`src/components/shared/rank-progress.tsx`**:
- Inline progress bar showing distance to the next rank
- Props: `currentXp: number`, `currentRank: { name, minXp, iconSlug, color }`, `nextRank: { name, minXp, iconSlug, color } | null` (null when the viewer already holds the summit rank)
- Layout (mobile-first): current rank badge left, slim filled bar center, next-rank badge right. Below the bar a Turkish helper line: `"{nextRank.name} rütbesine {remainingXp.toLocaleString('tr-TR')} XP kaldı"`, or `"Zirve rütbeye ulaştınız"` when `nextRank === null`
- Percentage fill = `(currentXp - currentRank.minXp) / (nextRank.minXp - currentRank.minXp)` clamped to `[0, 1]`. When `nextRank === null`, the bar renders fully filled with a gold accent
- Accessibility: root element uses `role="progressbar"`, `aria-valuenow`, `aria-valuemin={currentRank.minXp}`, `aria-valuemax={nextRank?.minXp ?? currentRank.minXp}`, plus an aria-label describing the next-rank target
- Consumed on the profile page header (Phase 12 Section 9); reusable elsewhere if future screens require a rank-progress surface

**Lucide icon coverage:** the 8 rank icons (`eye`, `lightbulb`, `scale`, `gavel`, `book-open`, `landmark`, `sparkles`, `crown`) are all part of the standard `lucide-react` export surface — no custom SVGs needed. Lock icons (`Lock`) for the avatar grid and rank/star icons for notifications also come from the same package, keeping the icon dependency single-sourced.

**`src/components/shared/empty-state.tsx`**:
- Icon + title + description + optional action button
- For cases like "Henüz konu yok", "Bildirim yok"

**`src/components/shared/loading-spinner.tsx`**:
- For page and component loading states

**`src/components/shared/error-boundary.tsx`**:
- React error boundary, fallback UI on error

**`src/components/shared/seo-head.tsx`**:
- Helper for dynamic meta tags (title, description, OG)

### 11.1. Cookie Consent Banner + PostHog Analytics Provider

PostHog analytics and the cookie-consent banner ship together in this phase. PostHog sets tracking cookies, so the KVKK / ePrivacy expectation is that it must not initialise until the user has explicitly accepted. The banner and the provider are wired as a single unit to remove any drift between "analytics is live" and "consent banner is live".

**`src/components/shared/cookie-consent.tsx`** — the banner:
- Rendered inside the root layout; displayed to every visitor whose `cookieConsent` preference (localStorage key `cookie-consent`) is missing
- Turkish copy: heading "Çerez Tercihleri", body "Platformu geliştirmek ve deneyiminizi iyileştirmek için çerez kullanıyoruz."
- Two buttons: `"Kabul Et"` → writes `cookie-consent: "all"`, hides banner, triggers the PostHog provider to initialise. `"Sadece Zorunlu"` → writes `cookie-consent: "essential"`, hides banner, PostHog stays dormant
- Accessible: focus trap on first render, restores focus to the triggering element on dismissal, `role="dialog"`, labelled title + description

**`src/providers/posthog-provider.tsx`**:
- Reads `cookie-consent` on mount; only initialises PostHog when the value equals `"all"`
- PostHog client initialization with `NEXT_PUBLIC_POSTHOG_KEY` (project ID: 160520, EU Cloud) and `NEXT_PUBLIC_POSTHOG_HOST` (`https://eu.i.posthog.com`)
- Auto page view tracking enabled
- User identification: after login, call `posthog.identify(userId, { username, email })`. On logout, call `posthog.reset()`
- When the user later changes the preference (e.g., via the profile settings → "Çerez Tercihleri" link), the provider tears down and re-initialises in place: switching from `"all"` to `"essential"` calls `posthog.reset()` + `posthog.opt_out_capturing()`; switching back re-initialises with the stored user context

This replaces the earlier "defer banner to Phase 14" note — Phase 14 Section 12.1 now documents the already-shipped banner rather than deferring it.

### 12. Vitest Setup

Vitest + React Testing Library setup for `apps/web`:
- `vitest.config.ts` — jsdom environment, path aliases (@/ → src/)
- `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`
- `src/test/setup.ts` — global test setup (jest-dom matchers)
- Script: `pnpm --filter web test` → `vitest run`, `pnpm --filter web test:watch` → `vitest`

### 13. Accessibility (a11y) Foundation

- shadcn/ui (Radix) provides automatic ARIA support
- All interactive elements should have `focus-visible` outline
- Color contrast WCAG AA: contrast-checked colors in `tailwind.config`
- Skip to content link (hidden in header, visible on Tab)
- `alt` text required on all images

### 14. Route Structure Preparation

App Router directory structure (empty page files, content in subsequent phases):

```
src/app/
├── layout.tsx                          # Root layout
├── page.tsx                            # Home page (topic list)
├── (auth)/
│   ├── login/page.tsx
│   ├── register/page.tsx
│   ├── verify-email/page.tsx
│   └── complete-profile/page.tsx
├── topics/
│   ├── [slug]/page.tsx                 # Topic detail (SEO-friendly slug URL)
│   └── create/page.tsx                 # Create topic
├── profile/
│   ├── [username]/page.tsx             # User profile
│   └── settings/page.tsx               # Profile settings
├── notifications/page.tsx
├── search/page.tsx                      # Search results
├── categories/[slug]/page.tsx
├── terms-of-service/page.tsx
├── privacy-policy/page.tsx
├── community-guidelines/page.tsx
├── reset-password/page.tsx
└── not-found.tsx
```

Each page file contains only placeholder text for now. Actual content will be added in Phases 12-14.

## Security Checklist
- [ ] Access token stays in memory only; no write to `localStorage` or `sessionStorage` for the token itself
- [ ] Refresh token is never visible to JavaScript — it lives in the backend-issued `httpOnly; Secure; SameSite=Strict` cookie
- [ ] API client uses `credentials: "include"` so the refresh cookie ships on `/auth/refresh` and `/auth/logout`, and only those calls
- [ ] API client handles token refresh on 401 and retries the original request exactly once
- [ ] Socket.io connection is authenticated (handshake + periodic re-auth with the rotated access token)
- [ ] XSS protection: `dangerouslySetInnerHTML` is not used for user content rendering
- [ ] No script injection risk in Twemoji-parsed text

## Test Plan

```bash
# 1. Build check
cd apps/web && pnpm build
# Does it compile without errors?

# 2. Dev server
pnpm dev
# http://localhost:3000 → does the page render?

# 3. Layout
# Is the header visible?
# Does the hamburger menu work on mobile?
# Is the mobile nav bar visible (on mobile screens)?

# 4. shadcn/ui components
# Do Button, Input, Card, Dialog, Toast work?

# 5. API client
# Request with invalid token → 401 → refresh → retry

# 6. Twemoji
# Are emojis in text rendered as Twemoji SVGs?

# 7. Responsive
# Is the layout correct in desktop, tablet, mobile views?

# 8. Accessibility
# Does Tab navigation work?
# Is focus outline visible?
# Can it be read with a screen reader? (basic check)
```

## Completion Criteria
- [ ] shadcn/ui is installed and components work
- [ ] Layout components render (header, mobile nav)
- [ ] API client works (including interceptors)
- [ ] Auth store works (including persist)
- [ ] TanStack Query provider works
- [ ] Socket.io client connection is established
- [ ] Twemoji rendering works
- [ ] Emoji picker works
- [ ] Route structure is ready (placeholder pages)
- [ ] Mobile-first responsive layout works
- [ ] Dark mode toggle works (light/dark/system)
- [ ] Search bar navigates to search results page
- [ ] `<RankBadge />` renders the correct icon and color for each of the 8 seed ranks
- [ ] `<RankProgress />` renders partial fill, summit state, and Turkish helper copy; `aria-valuenow`/`aria-valuemax` populated correctly
- [ ] `pnpm build` succeeds without errors
