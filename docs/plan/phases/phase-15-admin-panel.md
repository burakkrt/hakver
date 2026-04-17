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
- **New device verification (authoritative):** authority-role logins trigger the Phase 3 Section 11.2 new-device flow. When `POST /auth/login` returns `202 Accepted` with `{ requiresDeviceVerification: true, deviceHash, message }`, the UI replaces the password form with a 6-digit OTP-style input screen labelled "Yeni cihaz doğrulaması — email adresinize kod gönderildi". Submit calls `POST /auth/verify-login-device` with `{ deviceHash, code, email, password }`; success continues to the dashboard redirect, failure surfaces `AUTH_INVALID_CODE` with a Turkish toast. Remembers the device for 30 days so subsequent logins from the same admin workstation skip the code step

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

#### 4.1. Backend: `/admin/stats` Endpoint

The dashboard data is served by a new backend endpoint owned by this phase. Even though Phase 15 is frontend-heavy, the admin-only API surface that backs it lives here so the feature ships as one coherent unit.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /admin/stats | Admin + RequirePermissions(`system:manage`) | Dashboard aggregate counters |

**Response schema** (new in `packages/shared/src/schemas/admin.ts`):

```
AdminStatsResponseSchema = {
  users:     { total: number, activeLast30Days: number, newToday: number, newThisWeek: number },
  topics:    { total: number, activeLast30Days: number, newToday: number },
  comments:  { total: number, newToday: number },
  reports:   { pending: number, resolvedThisWeek: number, dismissedThisWeek: number },
  recentActivity: ActivityLogSummarySchema[]  // last 10 entries (action, actor username, timestamp, targetType)
}
```

**Implementation notes:**
- Each counter is a single `COUNT` or date-bounded `COUNT` query; the five-minute Redis cache (`@CacheTTL(300)`, key `admin:stats`) absorbs dashboard refreshes without hitting the database on every poll
- Recent activity pulls the latest 10 rows from `ActivityLog`, joined with `User` for the actor username
- **`users.activeLast30Days` derivation (authoritative):** `SELECT COUNT(DISTINCT "userId") FROM "ActivityLog" WHERE "createdAt" >= now() - interval '30 days'`. Source is `ActivityLog` — no `lastActiveAt` column on `User` is needed. The 90-day activity-log retention (Phase 10 Section 9.1) comfortably covers the 30-day window
- **`topics.activeLast30Days` derivation (authoritative):** `SELECT COUNT(DISTINCT "targetId") FROM "ActivityLog" WHERE "targetType" = 'TOPIC' AND "createdAt" >= now() - interval '30 days'` — counts topics that received any engagement (vote, comment, edit, update-note) in the window. A topic with no interaction in the last 30 days is not considered "active" for the dashboard purpose even if it was created earlier
- Cache invalidation is TTL-based only — admins who need up-to-the-second figures can trigger a stats refresh via a "Yenile" button in the UI, which bypasses the cache with a `?nocache=1` query parameter accepted by this endpoint (still admin-guarded, rate-limited to 1 request per 10 seconds per admin)
- Response DTO construction is handled in the service layer; no Prisma types leak

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
- **Admin** viewing the page → backend returns `UserAdminResponseSchema` with the full firstName and lastName
- **Moderator** viewing the page → backend returns `UserPublicResponseSchema` (firstName tam, lastName ilk harf + ".") — moderator sees the masked name because the legal-name access is reserved for admins and compliance reviews
- A dedicated "Kimlik Bilgileri" section on the admin view visually groups the full legal name and ID-adjacent fields so admins understand that this data is accessible only on admin routes. The moderator view hides this section entirely and shows the masked name inline with the profile summary
- **Data-access audit:** Every time an admin view of this endpoint is served (i.e., full firstName + lastName returned), the backend writes an `ActivityLog` entry with `action: "admin:user-view"`, `metadata: { viewedUserId, viewerId }`. Moderator views are not audited at this level (they do not receive the full legal name). This is a KVKK/veri-koruma denetim izi for legal-name access
- Role badges
- **XP Management section (Admin only):**
  - Current XP and rank display — shows both `totalXp` and `highestXpReached` side by side (e.g., `"Şu anki XP: 3.200 · Zirve XP: 5.100"`) so admins can reason about why a rank-gated avatar remains selectable after a SUBTRACT or SET-down
  - XP adjustment form: type selector (Ekle / Çıkar / Ayarla), points input, reason textarea (mandatory)
  - "Ekle" adds points, "Çıkar" subtracts, "Ayarla" sets to exact value
  - Recent XP log table (last 20 entries — action, points, date, metadata)
  - Success toast with new XP value and rank after adjustment
  - Calls `PATCH /admin/users/:id/xp` (Phase 8 Task 5.1)
  - **Idempotency note surfaced to the admin:** the inline helper text reads `"XP azaltmak kullanıcının önceden açtığı rozet ve avatarları geri kilitlemez. XP tekrar artırılırsa aynı rütbeye yeniden ulaşıldığında bildirim gönderilmez."`
- **Rank History section (Admin only):**
  - Populated from `ActivityLog` rows where `action === 'user:rank-up'` (Phase 10 Section 9)
  - Each row: `<RankBadge size="sm" />` with the rank name, achieved timestamp (absolute + relative), and a trigger tag: `"organik"` (user earned it), `"yönetici"` (admin XP adjustment), `"mutabakat"` (reconciliation drift correction). The mapping consumes the `metadata.trigger` field
  - Paginated (last 20 entries with load-more)
  - Empty state: `"Kullanıcı henüz bir rütbeye yükselmedi"` for brand-new Gözlemci accounts with no summit entries
  - History reflects first-time summits only — passive re-entries into previously held ranks are deliberately hidden (Phase 10 idempotency rule)
- Active restrictions list
- Recent activity log (last 20 entries for this user)
- Recent reports against this user
- Topics and comments by this user (with links)
- Action buttons: Apply restriction, Change role (Admin only)

### 7.1. Admin Role Change

`PATCH /admin/users/:id/role` (Admin only):

**Request body:**
```json
{
  "role": "USER" | "MODERATOR" | "ADMIN",
  "reason": "Topluluk moderasyonu için yeni moderatör atandı"
}
```

**Validation:**
- `role` must be one of the three enum values
- `reason`: min 5, max 500 characters, required
- Cannot demote the last remaining ADMIN (prevents locking out all admins) → 400 `{ code: "MODERATION_ROLE_CHANGE_INVALID", message: "Son yönetici hesabının rolü değiştirilemez" }`
- Cannot change your own role → 403 `{ code: "MODERATION_ROLE_CHANGE_INVALID", message: "Kendi rolünüzü değiştiremezsiniz" }`
- Target user must exist and not be soft-deleted

**Flow:**
1. Replace the user's role assignment (`UserRole` table — remove old role, insert new role)
2. **Restriction sweep on authority promotion:** when the new role is an authority role (MODERATOR, ADMIN, or any future role that carries authority permissions), delete every row from `UserRestriction` where `userId` equals the target user. An authority-carrying account must not be blocked from the same actions it is now expected to supervise. The sweep runs inside the same transaction as the role swap so the role and restriction states always stay consistent. When the new role is USER, existing restrictions are left untouched because a demoted user is still subject to community rules. Note: gaining an authority role grants only the permissions bound to that specific role — it does not cascade permissions from other authority roles. For example, promoting a user to MODERATOR unlocks moderator permissions only; it does not grant admin-only actions even though both are authority roles. Permission enforcement remains the responsibility of the PermissionGuard against each role's permission bindings
3. Invalidate all active sessions of the target user (forces re-login with new permissions, avoids stale JWT)
4. Write `ActivityLog` entry: `action: "user:role-change"`, `metadata: { previousRole, newRole, reason, clearedRestrictionIds }`. The `clearedRestrictionIds` field lists any `UserRestriction` records removed by the authority-promotion sweep, giving the audit trail a complete picture of what changed
5. Response: 200 with the updated user summary

Error code for missing reason: `{ code: "MODERATION_REASON_REQUIRED", message: "Rol değişikliği için sebep girilmelidir" }`

**UI** (admin user detail page, admin only):
- "Rol Değiştir" button opens a modal
- Role selector (dropdown), reason textarea (mandatory, min 5 chars)
- Submit → calls `PATCH /admin/users/:id/role`
- Success toast: "Kullanıcının rolü güncellendi"

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

### 12. Broadcast Notification Composer (`/broadcasts`) — Admin Only

Surface for the `POST /admin/notifications/broadcast` endpoint defined in Phase 10 Section 3.1.

**Compose screen:**
- Title input (Turkish, 5–120 characters) — example placeholder "Bakım Duyurusu"
- Body textarea (Turkish, 10–1000 characters, character counter) — supports plain text; no rich formatting at MVP
- Audience selector (radio group):
  - **"Tüm kullanıcılar"** (`scope: "ALL"`) — default
  - **"Seçilen kullanıcılar"** (`scope: "SELECTED_USERS"`) — opens a multi-select user search (username autocomplete, max 5000 entries, visual chip list)
  - **"Role göre"** (`scope: "ROLE"`) — dropdown of available roles (USER / MODERATOR / ADMIN)
- Reason textarea (Turkish, 5–500 characters) — required audit-trail field "Bu duyurunun nedeni"
- Estimated-recipient count displayed live (calls a lightweight `GET /admin/notifications/broadcast/preview` endpoint as the audience changes)
- Action buttons: "İptal" and "Gönder". "Gönder" opens a confirmation dialog that repeats the title, body, audience description, and recipient count before calling the broadcast endpoint

**History screen:**
- Table view of past broadcasts: title, audience scope, recipient count, broadcaster, date, delivered count
- Row click expands the full body + reason and shows the BullMQ job status (completed / failed) so admins can verify delivery
- Filter: date range, broadcaster, scope

**Security / UX notes:**
- Available only to ADMIN role (not MODERATOR). The sidebar entry is hidden for moderators
- Submit button is disabled unless every required field passes validation client-side; the server-side Zod refinement is authoritative
- On `429 RATE_LIMIT_EXCEEDED` the UI surfaces the formatted Turkish message (Phase 10 / `.claude/rules/security.md`) and starts a countdown on the button
- A visible "Yönetici Duyurusu" tag appears in the preview so admins understand that the message will be clearly marked as an admin broadcast on the recipient's notification list

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
