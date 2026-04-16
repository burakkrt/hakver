# Phase 12: Frontend — Auth & User Pages

## Dependencies
- Phase 4 (backend auth + user APIs) and Phase 11 (frontend foundation) must be completed

## Goals
- Registration page (email + password)
- Login page
- Google OAuth flow (frontend)
- Email verification page
- Profile completion page (OAuth users)
- Consent text approval screen
- User profile page (self + others)
- Profile settings page
- TanStack Query hooks (auth + user)

## Tasks

### 1. Auth API Hooks

`src/hooks/use-auth.ts` — TanStack Query mutations:
- `useRegister()` → POST /auth/register
- `useLogin()` → POST /auth/login
- `useRefreshToken()` → POST /auth/refresh
- `useVerifyEmail()` → POST /auth/verify-email
- `useConfirmEmail()` → POST /auth/verify-email/confirm
- `useGoogleLogin()` → window.location.href → backend Google OAuth URL
- `useCompleteProfile()` → POST /auth/complete-profile
- `useAcceptConsent()` → POST /auth/accept-consent
- `useLogout()` → clear auth store, socket disconnect

On successful mutation → update auth store, show toast.
On error → show error message via toast.

### 2. User API Hooks

`src/hooks/use-user.ts` — TanStack Query:
- `useMe()` → GET /users/me (query)
- `useUser(username)` → GET /users/:username (query)
- `useUpdateProfile()` → PATCH /users/me (mutation)
- `useChangeUsername()` → PATCH /users/me/username (mutation)
- `useChangeEmail()` → PATCH /users/me/email (mutation)
- `useChangePassword()` → PATCH /users/me/password (mutation)
- `useChangeAvatar()` → PATCH /users/me/avatar (mutation)
- `useDeleteAccount()` → DELETE /users/me (mutation)
- `useCheckUsername(username)` → GET /users/check-username/:username (query, debounced)
- `useAvatars()` → GET /avatars (query)
- `useUserTopics(username)` → GET /users/:username/topics (query)
- `useUserVotes(username)` → GET /users/:username/votes (query, only on own profile)
- `useUserComments(username)` → GET /users/:username/comments (query)
- `useProfileStatus()` → GET /users/me/profile-status (query — returns which fields are missing for profile completion)
- `useBlockedUsers()` → GET /users/me/blocked (query — list of users the current user has blocked)
- `useRequestDataExport()` → POST /users/me/data-export (mutation — KVKK personal data export)
- `useMyBookmarks()` → GET /users/me/bookmarks (query, paginated — bookmarked topics)

### 3. Register Page (`/register`)

Form with React Hook Form + Zod (shared schema):
- Email, password, first name, last name, username, date of birth, gender
- Real-time username availability check on the username field (debounced, 500ms)
- Password strength indicator (weak/medium/strong)
- Date of birth: date picker (13-year age check on client-side)
- Gender: radio group (Erkek, Kadın, Belirtmek İstemiyorum)
- Client-side form validation + server error display
- Successful registration → update auth store → redirect to consent screen → after consent → redirect to email verification → after verification → redirect to home page. The order is: Register → Consent → Email Verification → Home.
- "Google ile Kayıt Ol" button
- Bottom: "Zaten hesabınız var mı? Giriş yapın" link

### 4. Login Page (`/login`)

- Email + password form
- "Şifremi Unuttum" link → redirect to `/reset-password` page
- Successful login → update auth store:
  - `requiresProfileCompletion` → redirect to `/complete-profile`
  - `!hasAcceptedConsent` → redirect to consent screen
  - `!emailVerifiedAt` → redirect to `/verify-email`
  - Otherwise → redirect to home page
- "Google ile Giriş Yap" button
- Bottom: "Hesabınız yok mu? Kayıt olun" link

### 5. Google OAuth Flow (Frontend)

- "Google ile Giriş/Kayıt" button → `window.location.href = NEXT_PUBLIC_API_URL + '/api/v1/auth/google'`
- Backend Google callback redirects to frontend: `FRONTEND_URL/login/callback?token=...&refresh=...&requiresCompletion=...`
- `src/app/(auth)/login/callback/page.tsx` → extract tokens from URL parameters, save to auth store, perform redirect

### 6. Email Verification Page (`/verify-email`)

- "Email adresinize gönderilen 6 haneli kodu girin" message
- 6-digit code input (OTP style — each digit in a separate input)
- "Kodu Doğrula" button
- "Kod gelmedi mi? Tekrar gönder" link (60-second cooldown timer)
- Successful verification → toast + redirect to home page
- 5 failed attempts → temporary block message

### 7. Complete Profile Page (`/complete-profile`)

For users who signed in via OAuth and haven't set a username yet:
- Username (real-time availability check)
- Gender selection
- Date of birth
- On success → redirect to consent screen or home page

**Important:** Profile completion is not just for OAuth users. Any user with missing required fields (username, firstName, lastName) is redirected here. The page checks which fields are missing and only shows those fields.

This page is also triggered when a user with an incomplete profile attempts a write action (create topic, comment, vote) — the auth guard intercepts and redirects here with a return URL, so the user is sent back to their original action after completing their profile.

### 8. Consent Screen

Modal or full page:
- Kullanım Koşulları, Gizlilik Politikası, Topluluk Kuralları links (link to separate pages for full text)
- Checkbox for each
- "Okudum ve kabul ediyorum" button (all checkboxes must be checked)
- After approval → redirect to home page

### 9. User Profile Page (`/profile/[username]`)

**Profile header:**
- Avatar (large)
- Username (prominent — the primary identifier)
- Name displayed as "Ahmet D." (firstName full + lastName first character + ".") when viewing another user's profile. The owner of the profile sees their full surname instead
- Age (calculated from dateOfBirth: `Math.floor((now - dob) / 365.25 / 24 / 60 / 60 / 1000)`)
- Bio
- Rank badge + XP
- Active restrictions (if any, warning banner)
- If own profile → "Profili Düzenle" button

Name rendering is driven by the API response shape. The frontend does not truncate the family name client-side — the backend returns the already-abbreviated string. This ensures the full surname is never present in client memory, devtools, or browser history for other users' profiles.

**Profile tabs:**
- Topics: list of topics created by the user
- Votes: topics the user voted on (only visible on own profile)
- Comments: topics the user commented on
- Kaydedilenler: bookmarked topics (only visible on own profile)

Each tab is paginated, with infinite scroll or a "Load more" button.

**If viewing another user's profile:**
- "Engelle" button (in dropdown menu)
- "Rapor Et" button (in dropdown menu)

### 10. Profile Settings Page (`/profile/settings`)

Sections:

**Personal Information:**
- First name, last name, date of birth edit form
- Bio edit (textarea, max 500 character counter)

**Avatar:**
- Current avatar display
- Preset avatars shown in a grid, click → select → save

**Username:**
- Current username
- Change form (if disabled → "XX gün sonra değiştirebilirsiniz" message)

**Email:**
- Current email
- Change form (password verification required)
- Verification status badge (verified/unverified)

**Password:**
- Current password + new password form
- This section is hidden for OAuth users

**Blocked Users:**
- "Engellenen Kullanıcılar" section
- List of blocked users (avatar, username, block date)
- "Engeli Kaldır" button per user → calls `DELETE /users/:username/block`
- Empty state: "Engellediğiniz kullanıcı yok"

**Veri Dışa Aktarma (KVKK):**
- "Verilerimi İndir" button
- Clicking sends `POST /users/me/data-export` → success toast: "Verileriniz hazırlanıyor, email adresinize gönderilecek"
- Disabled state with countdown if already requested within 7 days

**Account Deletion:**
- Red "Hesabımı Sil" button
- Confirmation dialog (irreversible action warning + password verification)

### 11. Legal & Utility Pages

- `/terms-of-service` → Kullanım Koşulları full text
- `/privacy-policy` → Gizlilik Politikası full text
- `/community-guidelines` → Topluluk Kuralları full text
- `/reset-password` → Two-step password reset page:
  - **Step 1 (no token in URL):** Email input form → "Şifre sıfırlama bağlantısı gönder" button. On success: "Email adresinize şifre sıfırlama bağlantısı gönderildi" message.
  - **Step 2 (token in URL query `?token=xxx`):** New password form (password + confirm password). On success: "Şifreniz başarıyla değiştirildi" + redirect to login page.
  - The page checks for `token` query parameter to decide which step to show.

Text content is fetched from the database (`ConsentVersion` table). Rendered with Server Component via SSR.

### 12. Auth Route Protection

**`src/components/shared/auth-guard.tsx`**:
- Client component, checks state from auth store
- Not logged in → redirect to `/login`
- Not verified → redirect to `/verify-email`
- Profile incomplete (missing username/firstName/lastName) → redirect to `/complete-profile` (with return URL)
- Profile not completed → redirect to `/complete-profile`
- Consent not accepted → show consent modal

Protected pages are wrapped with this guard.

## Security Checklist
- [ ] Password fields have `type="password"` and autocomplete="new-password"/"current-password"
- [ ] Form validation on both client and server side
- [ ] Google OAuth callback tokens are removed from URL after extraction
- [ ] Tokens stored in localStorage, with awareness of XSS protection
- [ ] Password verification in account deletion confirmation dialog
- [ ] Route protection cannot be bypassed

## Test Plan

```bash
# 1. Build
cd apps/web && pnpm build

# 2. Browser testing (with backend running):

# Registration flow
# /register → fill out form → register → consent screen → approve → verify-email → enter code → home page

# Login flow
# /login → email + password → login → home page

# Google OAuth
# /login → Google ile Giriş → Google consent → callback → complete-profile (new user) → home page

# Profile
# /profile/username → profile info, topics, comments tabs
# /profile/settings → edit info, change avatar, change password

# Auth guard
# /topics/create (without logging in) → redirect to /login

# Form validation
# Invalid email in registration form → error message
# 12-year-old date of birth → error message
# Taken username → real-time "Bu kullanıcı adı alınmış" message

# Account deletion
# /profile/settings → Hesabımı Sil → confirmation dialog → enter password → deleted → redirect to /login

# Mobile responsive
# Do all pages display correctly on mobile?
```

## Completion Criteria
- [ ] Registration flow works end-to-end (form → register → consent → verification)
- [ ] Login flow works
- [ ] Google OAuth flow works (new user + existing user)
- [ ] Profile completion works
- [ ] Email verification works
- [ ] Consent text approval works
- [ ] Profile page works (self + others, privacy rules)
- [ ] Profile settings works (all edit operations)
- [ ] Account deletion works
- [ ] Auth guard works on protected routes
- [ ] Legal text pages are displayed
- [ ] Mobile responsive
- [ ] `pnpm build` succeeds without errors
