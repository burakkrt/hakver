# Phase 17: Deployment & CI/CD

## Dependencies
- Phase 16 must be completed (tests passing, security audit finished)

## Goals
- Frontend deployment to Vercel
- Backend deployment to Railway
- Complete GitHub Actions CI/CD pipeline
- Production environment variables
- Health check and monitoring
- Production smoke test

## Tasks

### 1. Backend Dockerfile

`apps/api/Dockerfile`:
- Multi-stage build (builder + runner)
- Builder: install dependencies, prisma generate, build
- Runner: only production dependencies + build output
- Non-root user
- Health check instruction

```dockerfile
# Basic structure:
FROM node:22-alpine AS builder
# pnpm install, prisma generate, nest build

FROM node:22-alpine AS runner
# Only dist/ and node_modules/
USER node
EXPOSE 3001
CMD ["node", "dist/main"]
```

### 2. Railway Backend Deployment

1. Create a new service in Railway dashboard
2. Connect GitHub repository
3. Root directory: `apps/api`
4. Build command: use Dockerfile
5. Add environment variables (production values):
   - `DATABASE_URL` — production Supabase pooled connection
   - `DIRECT_URL` — production Supabase direct connection
   - `REDIS_URL` — production Upstash URL
   - `JWT_SECRET`, `JWT_REFRESH_SECRET` — production secrets (different!)
   - `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` — production OAuth
   - `GOOGLE_CALLBACK_URL` — production callback URL
   - `CLOUDINARY_*` — production Cloudinary
   - `RESEND_API_KEY` — production Resend
   - `EMAIL_FROM` — production email
   - `FRONTEND_URL` — production frontend URL (Vercel)
   - `PORT` — Railway auto-assigns or 3001
   - `NODE_ENV=production`
6. Deploy trigger: main branch push
7. Health check: `GET /api/v1/health`

### 3. Prisma Migration (Production)

Migration to production database:
- During or before Railway deploy: `npx prisma migrate deploy`
- Seed data: be careful in production — only essential initial data (roles, permissions, categories, avatars, XP actions, ranks, consent text versions, restriction categories)
- Admin user creation: seed script or manual

### 4. Vercel Frontend Deployment

1. Create a new project in Vercel dashboard
2. Connect GitHub repository
3. Framework: Next.js
4. Root Directory: `apps/web`
5. Build Command: `cd ../.. && pnpm turbo build --filter=web`
6. Install Command: `pnpm install`
7. Environment variables:
   - `NEXT_PUBLIC_API_URL` — Railway backend URL (https)
   - `NEXT_PUBLIC_SOCKET_URL` — Railway backend URL (wss)
8. Deploy trigger: main branch push
9. Domain: Vercel-provided .vercel.app domain (custom domain later)

### 4.1. Vercel Admin Panel Deployment

1. Create a second project in Vercel dashboard for the admin panel
2. Connect the same GitHub repository
3. Framework: Next.js
4. Root Directory: `apps/admin`
5. Build Command: `cd ../.. && pnpm turbo build --filter=admin`
6. Install Command: `pnpm install`
7. Environment variables:
   - `NEXT_PUBLIC_API_URL` — Railway backend URL (same as main frontend)
8. Deploy trigger: main branch push
9. Domain: Separate Vercel subdomain (e.g., admin.hakver.com later)
10. **Access control:** Admin panel is only accessible to authenticated users with ADMIN or MODERATOR roles. Authentication is enforced in the app, not at the hosting level.

### 5. Google OAuth Production Configuration

1. In Google Cloud Console:
   - Add production callback URL to Authorized redirect URIs
   - Add production frontend URL to Authorized JavaScript origins
2. Switch OAuth consent screen to "Published" mode
3. Add required domains

### 6. CORS and URL Updates

In production environment:
- Backend CORS origin: production frontend URL
- Google OAuth callback: production backend URL
- Frontend API URL: production backend URL
- Links in emails: production frontend URL

### 7. GitHub Actions CI/CD Pipeline

Update `.github/workflows/ci.yml`:

```yaml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm build  # includes type-check

  test-backend:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter api test
      # E2E tests may require a test DB — Railway test environment or Supabase dev branch

  test-frontend:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter web test
      # Playwright E2E tests here or in a separate job

  test-admin:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter admin test

  # Deploys are triggered automatically by Vercel and Railway on main push
  # This CI only runs test and lint checks
```

### 7.1. Cron Jobs (Production)

The following scheduled jobs run in the backend (`@nestjs/schedule`):

| Job | Schedule | Description |
|-----|----------|-------------|
| Activity Log Cleanup | Daily 03:00 AM | Delete regular activity logs > 90 days, security logs > 1 year |
| Notification Cleanup | Daily 03:30 AM | Delete notifications > 90 days |
| Expired Restriction Cleanup | Daily 04:00 AM | Clean up expired restriction records (optional, soft cleanup) |
| XP Reconciliation | Daily 04:30 AM | Compare User.totalXp with SUM(XpLog.points), fix drift, log corrections |

These are configured in a `ScheduleModule` within the backend. Ensure Railway keeps at least one instance running for cron jobs to execute.

### 8. Production Smoke Test

Post-deployment manual checklist:

```bash
# 1. Backend health
curl https://[backend-url]/api/v1/health
# → { "status": "ok" }

# 2. Frontend
# https://[frontend-url] → page loads

# 3. Registration flow
# Create new account → did email arrive → verification → login

# 4. Google OAuth
# Sign in with Google → callback → complete profile

# 5. Create topic
# Create topic → vote → comment

# 6. WebSocket
# Two different devices/tabs → vote → real-time update

# 7. Image upload
# Upload image during topic creation → is it in Cloudinary

# 8. Email
# Did email verification code arrive

# 9. Notifications
# Did notification arrive when voted on

# 10. Mobile
# From phone browser → is responsive layout correct

# 11. Admin Panel
# https://[admin-url] → page loads
# Login as admin → dashboard accessible
# View reports → review a report
# Login as normal user → access denied

# 12. Cron Jobs
# Check Railway logs → did cleanup jobs run at scheduled times?
# Activity logs older than 90 days → cleaned up?
```

### 9. Monitoring and Error Tracking

**Starting level (free):**
- Railway dashboard: CPU, memory, request count monitoring
- Vercel dashboard: deployment status, edge function analytics
- NestJS Pino logger: structured JSON logs visible in Railway

**To be added later (optional):**
- Sentry (error tracking)
- Uptime monitoring (UptimeRobot or similar)

### 10. Production Migration Strategy

Strategy for database schema changes in future updates:

1. **Create migration:** In development, create migration with `prisma migrate dev --name <name>`
2. **Test:** Test the migration on the development database
3. **PR:** Migration file is included in the PR, code review
4. **Production deploy:** `prisma migrate deploy` runs automatically during Railway deploy (in Dockerfile CMD or Railway start command)
5. **Rollback:** Prisma does not natively support rollback. If issues arise, write a reverse migration (`prisma migrate dev --name revert_<name>`)
6. **Breaking changes:** Major schema changes are done in multiple steps (e.g., add new column → update code → remove old column). For zero-downtime deploys, column deletion/renaming is done in a separate deploy.

### 11. Production Branch Protection

GitHub repository settings:
- `main` branch protection rules:
  - Require pull request reviews (min 1)
  - Require status checks to pass (CI)
  - No direct push

## Security Checklist
- [ ] Production environment variables differ from development (especially JWT secrets)
- [ ] HTTPS enforced (Vercel and Railway provide this automatically)
- [ ] Swagger disabled in production
- [ ] CORS restricted to production frontend URL(s) — both main frontend and admin panel
- [ ] Google OAuth in production mode (Published)
- [ ] .env files not in the repository
- [ ] Branch protection active
- [ ] Admin panel accessible only to admin/moderator users

## Test Plan

```bash
# 1. Production smoke test (Task 8 above)
# All items should be checked

# 2. SSL check
# https://[backend-url] → valid SSL certificate
# https://[frontend-url] → valid SSL certificate

# 3. CORS check
# Request to API from different origin → is it blocked?

# 4. Swagger
# https://[backend-url]/api/docs → 404 or access denied

# 5. GitHub Actions
# Open PR → did CI run → did tests pass
# Merge to main → was deploy triggered
```

## Completion Criteria
- [ ] Backend running on Railway, health endpoint returns 200
- [ ] Frontend running on Vercel, page loads
- [ ] Admin panel running on Vercel, accessible to admin/moderator only
- [ ] Production database migrated and seed data loaded
- [ ] Google OAuth works in production
- [ ] Email sending works in production
- [ ] Image upload works in production
- [ ] WebSocket works in production
- [ ] CI pipeline runs on PRs (including admin panel)
- [ ] Main branch protected
- [ ] Production smoke test completed (including admin panel)
- [ ] HTTPS active
- [ ] Swagger disabled in production
- [ ] Cron jobs running (activity log + notification cleanup)
- [ ] Data retention policy enforced
