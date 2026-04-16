# Phase 16: Integration Testing & Security Audit

## Dependencies
- Phase 14 and Phase 15 must be completed (all frontend + admin panel finished)

## Goals
- E2E tests (Playwright)
- Complete backend integration tests
- Security audit (OWASP Top 10)
- Performance check
- Cross-browser testing

## Tasks

### 1. Playwright E2E Tests

Playwright test files under `apps/web/e2e/`.

**Test scenarios:**

#### Auth flow (`auth.spec.ts`)
- Register → email verification → login → home page
- Google OAuth flow (mock or real, depending on test environment)
- Register with invalid credentials → error messages
- Login with wrong password → error message
- Token expiry → automatic refresh
- Logout → access protected page → redirect to login page

#### Topic flow (`topic.spec.ts`)
- Create topic (title + content + category + image) → visible on topic detail
- Edit topic (within 12 hours) → "düzenlendi" label
- Delete topic → removed from list
- Anonymous topic → author info hidden
- Category filter → correct topics displayed
- New user prerequisite → topic creation blocked

#### Voting flow (`voting.spec.ts`)
- Vote → counter updated
- Change vote → counters correct
- Withdraw vote → counters correct
- Vote on own topic → blocked

#### Comment flow (`comment.spec.ts`)
- Write comment → appears in list
- Write reply → nesting correct
- Like → counter increases
- Edit → text updated
- Delete → "bu yorum silindi"
- Anonymous comment → consistency

#### Profile flow (`profile.spec.ts`)
- View profile → info correct
- Edit profile → info updated
- Change avatar → avatar updated
- Other user's profile → firstName shown in full, lastName rendered as a single initial plus "." (e.g., "Ahmet D.")
- Cards (topic list, comment list, notification drawer) display only the username for the author — no firstName or lastName appears next to content

#### Notification flow (`notification.spec.ts`)
- Vote received → notification badge increases
- Click notification → navigate to relevant page + marked as read
- Mute → no notification received

#### Moderation flow (`moderation.spec.ts`)
- Block user → their topics disappear from feed
- Report → report created
- Restricted user → unable to perform action

#### Responsive test (`responsive.spec.ts`)
- Home page on mobile viewport → mobile nav visible
- Tablet viewport → appropriate layout
- Desktop → full layout

#### Admin panel flow (`admin.spec.ts`)
- Admin login → dashboard accessible
- View reports → filter by status → review report
- Apply restriction → restricted user cannot perform action
- View activity logs → filter by user/action
- Update XP action values → verify changes
- Non-admin user → admin panel returns 403

#### Search flow (`search.spec.ts`)
- Search by topic title → relevant results
- Empty search → empty state
- Search with special characters → no errors

#### Profile completion flow (`profile-completion.spec.ts`)
- OAuth user without username → blocked from creating topic → redirected to profile completion
- Complete profile → can now create topics
- User with complete profile → no redirect

### 2. Backend Integration Tests

In addition to unit tests written in Phases 3-10, end-to-end API test scenarios:

**Under `apps/api/test/` with Supertest:**

- Full register → login → create topic → vote → comment → XP check → rank check flow
- Moderator: view report → apply restriction → test restricted user
- Anonymous topic + comment → no user info leaking in responses
- Concurrent voting → no race condition
- Account deletion → anonymization correct

### 3. Security Audit (OWASP Top 10)

Check and fix for each item:

**A01: Broken Access Control**
- [ ] User can only edit their own data (IDOR check)
- [ ] Permission guards on all protected endpoints
- [ ] No user info leaking behind anonymous content
- [ ] Card responses (lists, feeds, notifications) never expose firstName or lastName — only admin endpoints return the full legal name
- [ ] Profile endpoints for other users return an abbreviated surname
- [ ] Deleted users cannot log in
- [ ] Normal users cannot access admin endpoints

**A02: Cryptographic Failures**
- [ ] Passwords hashed with bcrypt
- [ ] JWT secrets are of sufficient length
- [ ] HTTPS enforced (production)
- [ ] Sensitive data not returned in responses

**A03: Injection**
- [ ] SQL injection: Prisma parameterized queries (ORM protection)
- [ ] XSS: User content escaped, no dangerouslySetInnerHTML
- [ ] NoSQL injection: sanitization in Redis commands

**A04: Insecure Design**
- [ ] Rate limiting on all sensitive endpoints
- [ ] Email verification brute-force protection (5 attempt limit)
- [ ] Restriction system cannot be applied to admins/moderators

**A05: Security Misconfiguration**
- [ ] CORS allows only the declared allowed origins (main web app + admin panel + any other configured origin) — no wildcard
- [ ] Swagger enabled only in development
- [ ] Stack traces hidden in production (exception filter)
- [ ] Unnecessary HTTP headers removed (helmet middleware)

**A06: Vulnerable Components**
- [ ] Known vulnerabilities checked with `pnpm audit`
- [ ] Dependencies up to date

**A07: Authentication Failures**
- [ ] Brute-force protection (rate limiting)
- [ ] Strong password rules
- [ ] Token refresh mechanism works

**A08: Data Integrity Failures**
- [ ] All inputs validated with Zod
- [ ] File upload: type and size checks

**A09: Logging Failures**
- [ ] All actions in activity log
- [ ] Login attempts logged
- [ ] No sensitive data in logs (password, token)

**A10: SSRF**
- [ ] Cloudinary URLs validated on the backend
- [ ] No URL input accepted from users

### 4. NestJS Helmet Middleware

Add `helmet` middleware to `main.ts` (if not already present):
- X-Content-Type-Options: nosniff
- X-Frame-Options: DENY
- X-XSS-Protection enabled
- HSTS header (in production)

**Content Security Policy (CSP) configuration:**
```
default-src 'self';
script-src 'self';
style-src 'self' 'unsafe-inline';
img-src 'self' res.cloudinary.com data:;
font-src 'self';
connect-src 'self' {API_URL} {SOCKET_URL};
frame-src 'none';
object-src 'none';
base-uri 'self';
```
- Cloudinary image domain whitelisted in `img-src`
- API and WebSocket URLs whitelisted in `connect-src`
- `frame-src 'none'` prevents clickjacking
- Adjust as needed for Twemoji CDN, Google OAuth redirects, and other external resources
- CSP violations should be logged (report-uri or report-to) for monitoring

### 5. Performance Check

**Backend:**
- [ ] No N+1 query issues (Prisma include/select used correctly)
- [ ] Denormalized counters work correctly (no extra count queries)
- [ ] Redis cache active (frequently queried data: category list, avatar list, rank list)
- [ ] Pagination enforced (no unlimited data fetching)

**Frontend:**
- [ ] Lighthouse Performance score 80+
- [ ] Image optimization (next/image in use)
- [ ] Code splitting (dynamic import where needed)
- [ ] Bundle size check

### 6. Cross-Browser Test

Run E2E tests with Playwright across different browsers:
- Chromium
- Firefox
- WebKit (Safari)

## Security Checklist
This entire phase is a security check — all items in Task 3 constitute this list.

## Test Plan

```bash
# 1. E2E tests
cd apps/web && npx playwright test
# Do all scenarios pass?

# 2. Backend integration tests
cd apps/api && pnpm test:e2e
# Do all scenarios pass?

# 3. Security
pnpm audit
# No critical or high vulnerabilities

# 4. Performance
# Lighthouse → Performance 80+, Accessibility 90+, SEO 90+

# 5. Cross-browser
npx playwright test --project=firefox
npx playwright test --project=webkit
```

## Completion Criteria
- [ ] All E2E tests pass (Chromium, Firefox, WebKit)
- [ ] All backend integration tests pass
- [ ] OWASP Top 10 checklist completed, no open issues
- [ ] Helmet middleware active
- [ ] `pnpm audit` shows no critical vulnerabilities
- [ ] Lighthouse: Performance 80+, Accessibility 90+, SEO 90+
- [ ] No N+1 query issues
- [ ] Redis cache active
