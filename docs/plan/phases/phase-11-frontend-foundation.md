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

### 2. Tailwind Configuration

**`tailwind.config.ts`** extensions:
- Custom colors: primary, secondary, accent (brand colors)
- Font: Inter or a sans-serif font suitable for the project (Google Fonts)
- Container: centered container with responsive breakpoints
- Animations: fade-in, slide-in (for transitions)

### 3. Global Styles and Font

**`src/app/globals.css`**:
- Tailwind base/components/utilities
- CSS custom properties (color theme)
- Default typography

**Font loading:** Define with `next/font/google` using Inter or the chosen font in `layout.tsx`.

### 4. Layout Components

**`src/app/layout.tsx`** (root layout):
- Font and metadata
- Provider wrapping (QueryProvider, AuthProvider, SocketProvider, Toaster)

**`src/components/layout/header.tsx`**:
- Logo (left)
- Search (center — for the future, placeholder for now)
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
- Axios instance (or fetch wrapper)
- Request interceptor: `Authorization: Bearer {accessToken}` header
- Response interceptor:
  - 401 → attempt refresh with refresh token → if successful, retry original request
  - Refresh also 401 → logout, redirect to login page
- Error format standardization

### 6. Auth Store (Zustand)

**`src/stores/auth-store.ts`**:

```typescript
interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  setAuth: (data: { user, accessToken, refreshToken }) => void;
  clearAuth: () => void;
  updateUser: (data: Partial<User>) => void;
}
```

Tokens are stored in `localStorage`. With Zustand persist middleware.

### 7. Auth Provider

**`src/providers/auth-provider.tsx`**:
- On app mount, check token from localStorage
- If token exists → fetch user info with `/users/me`
- 401 → attempt refresh → if failed → clearAuth
- Loading spinner while auth state is loading

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

**`src/lib/twemoji.ts`**:
- Twemoji parse function: converts Unicode emojis in text to Twemoji SVGs
- React component: `<TwemojiText text="..." />` — renders emojis in text

**Emoji Picker:**
- Emoji picker component with `emoji-mart` library
- `<EmojiPicker onSelect={(emoji) => ...} />`
- Shown inside a popover

### 11. Shared Components

**`src/components/shared/user-card.tsx`**:
- Avatar, username, rank badge, name-surname (masked)
- Anonymous variant: animal icon, displayName
- Compact (inside comment/topic) and full (profile) variants

**`src/components/shared/anonymous-card.tsx`**:
- Animal icon + "Anonim Penguen #4521" displayName
- Rank not shown

**`src/components/shared/empty-state.tsx`**:
- Icon + title + description + optional action button
- For cases like "Henüz konu yok", "Bildirim yok"

**`src/components/shared/loading-spinner.tsx`**:
- For page and component loading states

**`src/components/shared/error-boundary.tsx`**:
- React error boundary, fallback UI on error

**`src/components/shared/seo-head.tsx`**:
- Helper for dynamic meta tags (title, description, OG)

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

### 13. Route Structure Preparation

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
├── categories/[slug]/page.tsx
├── terms-of-service/page.tsx
├── privacy-policy/page.tsx
├── community-guidelines/page.tsx
├── reset-password/page.tsx
└── not-found.tsx
```

Each page file contains only placeholder text for now. Actual content will be added in Phases 12-14.

## Security Checklist
- [ ] Tokens are only in localStorage (if not using httpOnly cookies, be cautious of XSS)
- [ ] API client handles token refresh on 401
- [ ] Socket.io connection is authenticated
- [ ] XSS protection: dangerouslySetInnerHTML is not used for user content rendering
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
- [ ] `pnpm build` succeeds without errors
