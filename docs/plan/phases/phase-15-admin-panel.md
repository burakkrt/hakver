# Phase 15: Admin Panel

## Dependencies
- Phase 10 (notifications & activity logging — all backend admin APIs complete) and Phase 11 (frontend foundation — design system and components) must be completed

## Goals
- Separate Next.js admin application (`apps/admin`)
- Admin/Moderator authentication and role-based access
- Report management (list, filter, review)
- User restriction management (apply, remove, list)
- User management (search, view details, view restrictions)
- XP & Rank configuration
- Activity log viewer
- Dashboard with basic stats
- Reuse shared design system from `apps/web`

## Tasks

### 1. Admin App Setup (`apps/admin`)

Create a new Next.js application under `apps/admin`:

```
apps/admin/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                    # Dashboard
│   │   ├── (auth)/
│   │   │   └── login/page.tsx          # Admin login
│   │   ├── reports/
│   │   │   └── page.tsx                # Report management
│   │   ├── restrictions/
│   │   │   └── page.tsx                # Restriction management
│   │   ├── users/
│   │   │   ├── page.tsx                # User list/search
│   │   │   └── [username]/page.tsx     # User detail
│   │   ├── xp-settings/
│   │   │   └── page.tsx                # XP & Rank config
│   │   ├── activity-logs/
│   │   │   └── page.tsx                # Activity log viewer
│   │   └── categories/
│   │       └── page.tsx                # Category management
│   ├── components/
│   │   ├── layout/
│   │   │   ├── admin-sidebar.tsx       # Sidebar navigation
│   │   │   ├── admin-header.tsx        # Top bar
│   │   │   └── admin-layout.tsx        # Layout wrapper
│   │   └── shared/                     # Admin-specific components
│   ├── hooks/
│   ├── lib/
│   │   └── api.ts                      # Same API client pattern as apps/web
│   ├── stores/
│   │   └── auth-store.ts              # Admin auth store
│   └── providers/
├── .env
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

**`package.json`** — `name: "@hakver/admin"`. Shares `@hakver/shared` package. Uses the same shadcn/ui components and Tailwind configuration as `apps/web`.

**`next.config.ts`** — Turborepo transpile packages configuration for `@hakver/shared`.

### 2. Admin Authentication

**Login page** (`/login`):
- Email + password form (same as main app login)
- No Google OAuth (admin login is email/password only for security)
- After login → check user role: if `ADMIN` or `MODERATOR` → allow access, otherwise → "Yetkisiz erişim" error and redirect

**Auth guard:**
- All routes except `/login` are protected
- Check JWT token + role (`ADMIN` or `MODERATOR`)
- Expired token → redirect to login
- Token refresh works the same as main app

**Role-based UI:**
- ADMIN: sees everything
- MODERATOR: sees reports, restrictions, user details. Cannot see: XP settings, activity logs, category management

### 3. Admin Layout

**Sidebar navigation:**
- Dashboard (home icon)
- Raporlar (flag icon) — report count badge for PENDING reports
- Kısıtlamalar (shield icon)
- Kullanıcılar (users icon)
- Kategoriler (folder icon) — Admin only
- XP Ayarları (star icon) — Admin only
- Aktivite Logları (list icon) — Admin only

**Top bar:**
- Admin username + avatar
- Role badge (Admin / Moderatör)
- Logout button
- Link to main app

**Layout:** Sidebar (fixed, collapsible on mobile) + main content area. Clean, functional design — no decorative elements.

### 4. Dashboard (`/`)

Basic statistics overview:
- Total users (active)
- Total topics (active)
- Total comments (active)
- Pending reports count (highlighted)
- Recent activity (last 10 activity log entries)
- New users today/this week

**API endpoints needed** (add to backend if not existing):
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /admin/stats | Admin | Dashboard statistics |

Stats endpoint returns aggregated counts from the database. Cached in Redis (TTL: 5 minutes).

### 5. Report Management (`/reports`)

**Report list page:**
- Table view: ID, target type, target (link to content), reporter, category, status, date
- Filters: status (PENDING, RESOLVED, DISMISSED), targetType (TOPIC, COMMENT, USER)
- Sorting: createdAt DESC (default), status
- Pagination
- PENDING reports highlighted with a colored badge

**Report review:**
- Click a report → expand or modal showing:
  - Report details (category, description, reporter)
  - Target content preview (topic title/content or comment content)
  - Target author info (including anonymous identity reveal for moderators)
  - Action buttons: "Çözüldü" (RESOLVED) / "Reddedildi" (DISMISSED)
  - Option to apply restriction directly from the report (opens restriction form pre-filled with the target user)

### 6. Restriction Management (`/restrictions`)

**Active restrictions list:**
- Table view: user, restricted action, category, reason, applied by, expires at
- Filter: action type, category
- Sorting: expiresAt, createdAt
- "Kaldır" button for each restriction (Admin only)

**Apply new restriction:**
- Form: user search (autocomplete by username), action selection, category + reason selection, expiration date/time, note
- Validation: cannot restrict admin/moderator, expiration must be in the future
- Success → toast notification

### 7. User Management (`/users`)

**User list:**
- Table view: username, email, name, role, rank, totalXp, status (active/deleted), createdAt
- Search by username or email
- Pagination

**User detail** (`/users/[username]`):
- Profile info (all fields, no masking — admin can see everything)
- Role badges
- **XP Management section (Admin only):**
  - Current XP and rank display
  - XP adjustment form: type selector (Ekle / Çıkar / Ayarla), points input, reason textarea (mandatory)
  - "Ekle" adds points, "Çıkar" subtracts, "Ayarla" sets to exact value
  - Recent XP log table (last 20 entries — action, points, date, metadata)
  - Success toast with new XP value and rank after adjustment
  - Calls `PATCH /admin/users/:id/xp` (Phase 8 Task 5.1)
- Active restrictions list
- Recent activity log (last 20 entries for this user)
- Recent reports against this user
- Topics and comments by this user (with links)
- Action buttons: Apply restriction, Change role (Admin only)

### 8. Category Management (`/categories`) — Admin Only

- Category list (table view with drag-to-reorder for sortOrder)
- Create new category form
- Edit category (name, description, isActive toggle)
- Deactivate category (soft delete)

### 9. XP & Rank Settings (`/xp-settings`) — Admin Only

**XP Actions:**
- Table view: action name, points, isActive
- Edit points value (inline or modal)
- Toggle isActive

**Ranks:**
- Table view: name, minXp, sortOrder
- Edit rank values
- Visual preview: rank progression bar

### 10. Activity Log Viewer (`/activity-logs`) — Admin Only

- Table view: user, action, target, timestamp, IP address
- Filters: userId (search), action type, date range
- Pagination (mandatory — large dataset)
- Export to CSV (optional, nice-to-have)
- Tiered highlighting: security/moderation actions in a different color

### 11. Restriction Category & Reason Management

**Within the restrictions section:**
- Manage restriction categories (CRUD)
- Manage restriction reasons per category (CRUD)
- Toggle isActive on categories and reasons

## Security Checklist
- [ ] Admin panel only accessible to ADMIN and MODERATOR roles
- [ ] Role checks enforced on both frontend and backend
- [ ] MODERATOR cannot access admin-only pages (XP settings, activity logs, categories)
- [ ] Admin panel uses the same JWT mechanism as the main app
- [ ] No sensitive data exposed in client-side code
- [ ] CORS configured to allow admin panel origin
- [ ] Admin actions logged in activity log

## Test Plan

```bash
# 1. Build
cd apps/admin && pnpm build
# Does it compile without errors?

# 2. Dev server
pnpm dev
# http://localhost:3002 → page renders

# 3. Authentication
# Login as admin → dashboard visible
# Login as moderator → limited menu items
# Login as normal user → "Yetkisiz erişim" error
# Invalid credentials → error message

# 4. Reports
# View report list → filter by PENDING → review a report
# Apply restriction from report → restriction created

# 5. Restrictions
# Apply restriction → user is restricted
# Remove restriction (admin) → user can perform action again
# Try to restrict another admin → error

# 6. Users
# Search by username → results
# View user detail → all info visible
# Change user role (admin) → role updated

# 7. Categories
# Create category → visible in list
# Edit category → name updated
# Deactivate → isActive false

# 8. XP Settings
# Change XP points for an action → value updated
# Toggle isActive → action disabled

# 9. Activity Logs
# View logs → filter by user → filter by date range
# Security actions highlighted

# 10. Role-based access
# Moderator → cannot access /categories, /xp-settings, /activity-logs
# Admin → can access everything
```

## Completion Criteria
- [ ] Admin app builds and runs without errors
- [ ] Authentication works (admin + moderator, not regular users)
- [ ] Dashboard displays basic statistics
- [ ] Report management works (list, filter, review)
- [ ] Restriction management works (apply, remove, list)
- [ ] User management works (search, view, role change)
- [ ] Category management works (CRUD)
- [ ] XP & Rank settings work
- [ ] Activity log viewer works (filter, pagination)
- [ ] Role-based access control works (admin vs moderator)
- [ ] Mobile responsive (basic — table views may use horizontal scroll)
- [ ] `pnpm build` succeeds without errors
