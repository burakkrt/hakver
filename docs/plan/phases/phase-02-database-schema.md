# Phase 2: Database Schema & Prisma

## Dependencies
- Phase 1 must be completed

## Goals
- Set up Prisma ORM and Supabase connection
- Create schemas for all tables required for MVP (including infrastructure tables for future features)
- Run migrations
- Load initial data with seed data
- Integrate PrismaService as a NestJS module

## Tasks

### 1. Prisma Setup

Set up Prisma under `apps/api`.

**`prisma/schema.prisma`** — Datasource configuration:
```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

### 2. Enum Definitions

Define the following enums in the Prisma schema (must be consistent with `packages/shared` enums):

```
Gender: MALE, FEMALE, UNSPECIFIED
AuthProvider: LOCAL, GOOGLE
VoteType: RIGHT, WRONG
ReportTargetType: TOPIC, COMMENT, USER
ReportStatus: PENDING, REVIEWED, RESOLVED, DISMISSED
NotificationType: TOPIC_VOTED, TOPIC_COMMENTED, COMMENT_LIKED, COMMENT_REPLIED, MENTIONED, TOPIC_UPDATED, COMMENT_HIGHLIGHTED, ADMIN_BROADCAST, RANK_UP
ReferenceType: TOPIC, COMMENT, VOTE, COMMENT_LIKE, USER, RANK
```

### 3. Table Schemas

Create all the following tables. Every table has `createdAt` and `updatedAt` fields as standard.

#### User
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK, default uuid |
| email | String | unique |
| emailVerifiedAt | DateTime? | null = not verified |
| passwordHash | String? | null = OAuth user |
| firstName | String | |
| lastName | String | |
| username | String? | unique (when not null), index — nullable for OAuth users before profile completion |
| dateOfBirth | DateTime | |
| gender | Gender enum | |
| bio | String? | max 500 characters |
| avatarId | FK → Avatar | |
| totalXp | Int | default 0 |
| highestXpReached | Int | default 0 — monotonically increasing watermark of the highest `totalXp` the user has ever held. Updated in the same transaction as `totalXp` via `GREATEST(highestXpReached, totalXp)`. Never decreased — persists across XP revocation, admin SUBTRACT/SET, and reconciliation drift correction. Drives rank-gated avatar unlocks (Phase 4 Section 9) and `RANK_UP` notification idempotency (Phase 10) |
| currentRankId | FK → Rank? | |
| provider | AuthProvider enum | default LOCAL |
| providerId | String? | Google OAuth ID |
| usernameChangedAt | DateTime? | |
| emailChangedAt | DateTime? | |
| sessionVersion | Int | default 0 — incremented by `invalidateOtherSessions(userId)` (password change, password reset, email change). JWT access and refresh tokens carry this value as the `sv` claim; `/auth/refresh` rejects tokens whose embedded `sv` is below the current `User.sessionVersion`. Enables server-side session revocation on security events without introducing a per-token tracking table (Phase 3 Section 11.1) |
| deletedAt | DateTime? | soft delete |

#### Role
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| name | String | unique (ADMIN, MODERATOR, USER) |
| description | String? | |

#### Permission
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| action | String | unique (topic:create, topic:edit, topic:delete, comment:create, comment:delete, user:restrict, user:view-anonymous, report:review, category:manage, system:manage) |
| description | String? | |

#### UserRole
| Field | Type | Constraint |
|-------|------|-----------|
| userId | FK → User | |
| roleId | FK → Role | |
| assignedAt | DateTime | default now |
| | | @@unique([userId, roleId]) |

#### RolePermission
| Field | Type | Constraint |
|-------|------|-----------|
| roleId | FK → Role | |
| permissionId | FK → Permission | |
| | | @@unique([roleId, permissionId]) |

#### Avatar
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| name | String | Turkish display name (e.g., "Anonim Penguen", "Yargıç Baykuşu") |
| url | String | Served from `/avatars/{slug}.svg` (self-hosted; see Phase 17 build step) |
| category | String enum | `"free"` · `"rank-locked"` · `"system"` — controls visibility and selection eligibility |
| requiredRankId | FK → Rank? | null for `category="free"` or `"system"`; populated for `category="rank-locked"`. Viewer must satisfy `max(totalXp, highestXpReached) >= rank.minXp` to select |
| sortOrder | Int | default 0 — grid ordering within a category |
| isActive | Boolean | default true |

**Notes:**
- The old `isDefault` field is removed. Initial-pool candidate selection for new registrations uses `WHERE category = 'free' AND isActive = true ORDER BY RANDOM() LIMIT 1` (Phase 3 Section 5 step 6).
- `category: "system"` rows are excluded from the public `GET /avatars` response and can never be selected via `PATCH /users/me/avatar` — they are only ever assigned by backend services (e.g., the "Silinmiş Kullanıcı" placeholder referenced in Phase 4 Section 5.1).

#### Category
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| name | String | unique |
| slug | String | unique, index |
| description | String? | |
| isActive | Boolean | default true |
| sortOrder | Int | default 0 |

#### Topic
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| title | String | max 150 |
| content | String | max 3000 |
| slug | String | unique, index — composed as `{title-slug}-{shortKey}` for SEO-friendly URLs; see Phase 5 Section 6 for the generation rule |
| authorId | FK → User | index |
| categoryId | FK → Category | index |
| isAnonymous | Boolean | default false |
| isEdited | Boolean | default false |
| editableUntil | DateTime | createdAt + 12 hours |
| voteCountRight | Int | default 0, denormalized |
| voteCountWrong | Int | default 0, denormalized |
| commentCount | Int | default 0, denormalized — counts only level-1 (parent) comments that are NOT soft-deleted |
| replyCount | Int | default 0, denormalized — counts level-2 replies (`parentId IS NOT NULL`) that are NOT soft-deleted; exposed through the topic detail response but never on the topic card |
| trendingScore | Float | default 0, denormalized — time-weighted engagement score; recomputed every 5 minutes by the "Trending Score Recompute" cron (Phase 17 / Phase 5 Section 10). Older than 7 days → stays 0 |
| controversialScore | Float | default 0, denormalized — balance-weighted engagement score (high when RIGHT and WRONG votes are close AND the topic has real engagement); recomputed by the same cron job. Older than 7 days → stays 0 |
| coverImageId | FK → TopicImage? | null = first image is cover |
| isPinned | Boolean | default false |
| pinnedAt | DateTime? | when the topic was pinned |
| pinnedById | FK → User? | moderator/admin who pinned |
| updateNote | String? | max 500 — author-written update shown below the original content after `editableUntil` expires |
| updateNoteAt | DateTime? | timestamp of the most recent update-note change |
| deletedAt | DateTime? | soft delete |

#### TopicImage
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| topicId | FK → Topic | index |
| url | String | Cloudinary URL |
| publicId | String | Cloudinary public ID (for deletion) |
| sortOrder | Int | default 0 |

#### Vote
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | |
| topicId | FK → Topic | index |
| type | VoteType enum | RIGHT or WRONG |
| | | @@unique([userId, topicId]) |

#### Comment
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| content | String | max 1000 |
| authorId | FK → User | index |
| topicId | FK → Topic | index |
| parentId | FK → Comment? | null = level 1, populated = level 2 |
| isAnonymous | Boolean | default false |
| isEdited | Boolean | default false |
| lastEditedAt | DateTime? | |
| likeCount | Int | default 0, denormalized |
| isHighlighted | Boolean | default false — topic author marks a comment as "öne çıkarılan" (highlighted); see Phase 7 highlight flow |
| highlightedAt | DateTime? | timestamp of the highlight action; used to sort multiple highlighted comments |
| highlightedById | FK → User? | the user who highlighted this comment (always the topic author; preserved for audit and moderator review) |
| deletedAt | DateTime? | soft delete |

#### CommentLike
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | |
| commentId | FK → Comment | index |
| | | @@unique([userId, commentId]) |

#### AnonymousIdentity
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | |
| topicId | FK → Topic | |
| displayName | String | e.g.: "Anonim Penguen #4521" |
| iconType | String | animal type |
| | | @@unique([userId, topicId]) |

#### XpAction
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| action | String | unique (TOPIC_CREATE, COMMENT_CREATE, VOTE_CREATE, COMMENT_LIKE_RECEIVED, TOPIC_VOTE_RECEIVED) |
| points | Int | |
| description | String? | |
| isActive | Boolean | default true |

#### XpLog (hot table)
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| actionId | FK → XpAction | |
| points | Int | positive = earned, negative = revoked |
| referenceType | ReferenceType enum | |
| referenceId | UUID | related entity ID |
| metadata | Json? | additional context (e.g., `{ topicId }` for comment XP, `{ voterId }` for vote-received XP, `{ reason, adjustedBy }` for admin adjustments) |
| createdAt | DateTime | default now |

Holds the **last 12 months** of XP events in the primary database. Older rows migrate nightly to `XpLogArchive` — see below and Phase 10 Section 9.2 for the cron.

#### XpLogArchive (cold table)
Same columns as `XpLog`. Daily cron moves rows older than 12 months from `XpLog` into `XpLogArchive`, then deletes them from the hot table. Keeping the schema identical lets reconciliation, data-export, and admin-view queries reuse the same DTO shape; only the query target changes. The archive row retention strategy is **indefinite** at MVP — the table stays for KVKK data-export completeness, statistics, and auditability, and is vacuumed only when a user's account is fully retired (Phase 4 Section 10 Stage 2 cold archive).

Key consequences for downstream queries:
- Daily XP cap (Phase 8) only ever consults `XpLog` (today's window is well under 12 months)
- Reconciliation cron (Phase 10 Section 9) reads `SUM(XpLog.points) + SUM(XpLogArchive.points)` when comparing against `User.totalXp`
- Admin `/admin/users/:id/xp-logs` endpoint (Phase 8 Task 5.1) paginates over `XpLog` by default with a toggle to include archive rows when the admin needs long-history review
- KVKK personal data export (Phase 4 Section 11) iterates both tables so the user always receives their complete XP history

#### Rank
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| name | String | Turkish display name (e.g., "Yargıç", "Allâme") |
| minXp | Int | unique — non-negative; `0` for the starter rank |
| iconSlug | String | Lucide icon identifier rendered by the frontend `<RankBadge />` component (see Phase 11 Section 11). Example values seeded below |
| color | String | Semantic color token that drives the rank badge tint (e.g., `"zinc"`, `"violet"`, `"amber"`). Frontend maps the token to Tailwind classes |
| sortOrder | Int | |

#### RestrictionCategory
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| name | String | unique |
| description | String? | |
| isActive | Boolean | default true |

#### RestrictionReason
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| categoryId | FK → RestrictionCategory | |
| text | String | |
| isActive | Boolean | default true |

#### UserRestriction
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| restrictedAction | String | e.g.: "topic:create", "comment:create", "vote:create" |
| categoryId | FK → RestrictionCategory | |
| reasonId | FK → RestrictionReason | |
| appliedById | FK → User | admin who applied the restriction |
| expiresAt | DateTime | |
| note | String? | |

#### Report
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| reporterId | FK → User | |
| targetType | ReportTargetType enum | |
| targetId | UUID | |
| categoryId | FK → RestrictionCategory | report category |
| description | String? | max 500 |
| status | ReportStatus enum | default PENDING |
| reviewedById | FK → User? | |
| reviewedAt | DateTime? | |

#### UserBlock
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| blockerId | FK → User | |
| blockedId | FK → User | |
| | | @@unique([blockerId, blockedId]) |

#### Notification
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index (recipient) |
| type | NotificationType enum | |
| actorId | FK → User | the **latest** actor that triggered/updated this notification (for aggregation, this is the most recent contributor). Non-nullable: every MVP notification has a concrete acting user, including `ADMIN_BROADCAST` where the actor is the admin who dispatched the broadcast |
| referenceType | ReferenceType enum | includes `USER` when the reference is the admin that sent an `ADMIN_BROADCAST` |
| referenceId | UUID | |
| title | String? | optional short heading used for `ADMIN_BROADCAST` (e.g., "Bakım Duyurusu"); null for every other notification type |
| body | String? | optional multi-line body used for `ADMIN_BROADCAST` (max 1000 chars, HTML-stripped); null for every other notification type |
| isRead | Boolean | default false |
| aggregatedCount | Int | default 1 — total number of aggregated actors (grows as new same-type-same-target actions fold into this record) |
| updatedAt | DateTime | last time a new actor aggregated into this notification; used for sorting/dedup window checks |

**Aggregation policy:**
- When a new `TOPIC_VOTED`, `TOPIC_COMMENTED`, `COMMENT_LIKED`, or `COMMENT_REPLIED` notification is about to be created, first look for an existing UNREAD (`isRead=false`) record with the same `(userId, type, referenceType, referenceId)` and `updatedAt >= now() - 15min`. If found, append the new actor to `NotificationActor`, increment `aggregatedCount`, set `actorId` to the new actor, and bump `updatedAt`. Otherwise, create a fresh record
- `MENTIONED`, `TOPIC_UPDATED`, `COMMENT_HIGHLIGHTED` always create a single record — no aggregation
- Actor-target idempotency: within the same 24h window, the same `(userId, type, referenceId, actorId)` must not create a second record (prevents vote → withdraw → vote re-notifications). If an earlier notification from the same actor on the same target exists, reuse it (bump `updatedAt`) instead of creating a new record

#### NotificationActor
Junction table storing the last N actors aggregated into a notification. UI shows the most recent 3 actors by username followed by the Turkish suffix "ve N diğer kullanıcı" (user-facing copy stays in Turkish).

| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| notificationId | FK → Notification | index |
| actorId | FK → User | |
| isAnonymousForReference | Boolean | default false — written at insert time. `true` when the actor acted anonymously relative to the notification's reference: for `TOPIC` references (TOPIC_VOTED / TOPIC_COMMENTED), resolved from `Topic.isAnonymous` OR a matching `AnonymousIdentity(userId=actorId, topicId=reference.id)` row; for `COMMENT` references (COMMENT_LIKED / COMMENT_REPLIED), resolved from the triggering comment's `isAnonymous` field. Precomputed so notification listing (Phase 10 Section 5) derives `isAnonymousAggregate` without extra joins |
| createdAt | DateTime | default now |
| | | @@unique([notificationId, actorId]) — same actor counted once per notification |

#### UserNotificationPreference
Per-user, per-type toggles. Missing row = default enabled.

| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| type | NotificationType enum | |
| enabled | Boolean | default true |
| | | @@unique([userId, type]) |

`MENTIONED` and `TOPIC_UPDATED` are marked non-toggleable on the backend (personal signals); the settings UI hides the toggle for these two types.

#### TopicBookmark
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | |
| topicId | FK → Topic | |
| createdAt | DateTime | default now |
| | | @@unique([userId, topicId]) |

#### TopicSubscription
Explicit opt-in subscription that a non-author user can create on a topic to receive update notifications when the author edits the topic or adds an update note. Authors are subscribed implicitly to their own topics (no row needed; the notification service treats `topic.authorId === userId` as subscribed). Voters and commenters are **not** automatically subscribed — the default is no notification, the user must toggle subscription on the topic detail page.

| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| topicId | FK → Topic | index |
| createdAt | DateTime | default now |
| | | @@unique([userId, topicId]) — one subscription row per user/topic pair |

#### NotificationMute
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | |
| topicId | FK → Topic | |
| | | @@unique([userId, topicId]) |

#### ActivityLog
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| action | String | |
| targetType | String? | |
| targetId | UUID? | |
| metadata | Json? | additional context |
| ipAddress | String? | |
| userAgent | String? | |

#### ConsentVersion
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| type | String | TERMS_OF_SERVICE, PRIVACY_POLICY, COMMUNITY_GUIDELINES |
| version | String | e.g.: "1.0.0" |
| content | String | text content |
| publishedAt | DateTime | |

#### ConsentRecord
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| consentVersionId | FK → ConsentVersion | |
| acceptedAt | DateTime | default now |
| ipAddress | String? | |

> **Badge tables are intentionally not part of MVP scope.** The badge system (catalog, tier structure, XP bonus semantics, award mechanics) has not been designed yet. When the design is finalized, Badge and UserBadge tables will be added via a dedicated migration alongside the catalog and service layer. Until then, the schema omits these tables entirely to keep MVP surface area tight. See `plan.md` "Detailed Design Pending Items" for the placeholder.

### 4. Indexes

Additional indexes for performance:
- `Topic`: `(categoryId, createdAt DESC)` — category-based listing
- `Topic`: `(authorId, createdAt DESC)` — user topics
- `Topic`: `(isPinned DESC, trendingScore DESC)` — powers `sort=trending` listing (pinned first, then precomputed score)
- `Topic`: `(isPinned DESC, controversialScore DESC)` — powers `sort=controversial` listing (pinned first, then precomputed score)
- `Comment`: `(topicId, likeCount DESC)` — like-based sorting
- `Comment`: `(topicId, isHighlighted DESC, highlightedAt DESC)` — highlighted comments float to top of listing
- `Vote`: `(topicId)` — topic-based vote counting
- `XpLog`: `(userId, createdAt)` — daily XP calculation
- `ActivityLog`: `(userId, createdAt DESC)` — activity queries
- `Notification`: `(userId, isRead, createdAt DESC)` — unread notifications
- `Notification`: `(userId, type, referenceType, referenceId, isRead)` — aggregation lookup (find existing unread same-target record within 15min window)
- `NotificationActor`: `(notificationId, createdAt DESC)` — render last actors in UI
- `UserNotificationPreference`: `(userId, type)` — unique, preference lookup
- `UserRestriction`: `(userId, expiresAt)` — active restriction check
- `UserBlock`: `(blockedId)` — reverse block lookup for symmetric block checks
- `Avatar`: `(category, requiredRankId, sortOrder)` — avatar grid listing and rank-filter query

### 4.1. Search Index (pg_trgm)

Enable the `pg_trgm` PostgreSQL extension and create a GIN index for topic title search:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_topic_title_trgm ON "Topic" USING gin (title gin_trgm_ops);
```

This is added via a Prisma migration using raw SQL. The `pg_trgm` GIN index dramatically speeds up `ILIKE` queries used in topic search (Phase 5). Without this index, `ILIKE` performs a full table scan. Supabase PostgreSQL supports `pg_trgm` natively.

### 5. PrismaService

Under `apps/api/src/prisma/`:

**`prisma.module.ts`** — Global module (`@Global()`)
**`prisma.service.ts`** — Service extending `PrismaClient` with `$extends` (Client Extensions). `$connect()` on `onModuleInit`, `$disconnect()` on `onModuleDestroy`. Soft delete via Client Extensions: all `findMany`, `findFirst`, `findUnique` queries automatically filter `deletedAt: null`. Override with custom methods like `findManyWithDeleted()` and `findFirstWithDeleted()` for moderator/admin access to deleted content. This approach is more modern and type-safe than Prisma middleware.

### 6. Seed Data

`prisma/seed.ts` file. Load the following data:

**Roles:**
- ADMIN, MODERATOR, USER

**Permissions:**
- topic:create, topic:edit, topic:delete, comment:create, comment:delete, vote:create, user:restrict, user:view-anonymous, report:review, category:manage, system:manage

**Role-Permission mappings:**
- ADMIN → all permissions
- MODERATOR → topic:delete, comment:delete, user:restrict, user:view-anonymous, report:review
- USER → topic:create, comment:create, vote:create

**Categories:**
- Genel (slug: genel)
- İlişkiler (slug: iliskiler)

> **Seed order note:** `Rank` rows are seeded **before** `Avatar` rows because `Avatar.requiredRankId` is an FK to `Rank.id`. The seed script enforces this ordering; any reshuffling will fail the FK check.

**Ranks (8 tiers, Wisdom / Justice theme — Turkish display names):**

| name            | minXp  | iconSlug    | color   | sortOrder |
|-----------------|--------|-------------|---------|-----------|
| Gözlemci        | 0      | eye         | zinc    | 1         |
| Düşünür         | 150    | lightbulb   | violet  | 2         |
| Hakem           | 500    | scale       | blue    | 3         |
| Yargıç          | 1500   | gavel       | indigo  | 4         |
| Bilge           | 5000   | book-open   | emerald | 5         |
| Adalet Üstadı   | 15000  | landmark    | amber   | 6         |
| Bilgelik Piri   | 30000  | sparkles    | rose    | 7         |
| Allâme          | 60000  | crown       | gold    | 8         |

Progression: middle tiers step by ~3x (500→1500→5000→15000), the top two tiers step by 2x (30000→60000) so the summit stays reachable for dedicated users without collapsing the prestige gap. The starter rank (Gözlemci, 0 XP) is assigned automatically at registration (Phase 3 Section 5 step 8) and preserves the non-null `User.currentRankId` invariant.

**Avatars (40 rows total — 15 free + 24 rank-locked + 1 system):**

- **Free pool (15 rows, `category: "free"`, `requiredRankId: null`, `isActive: true`):** community-friendly illustrations used as the initial-pool candidates for new registrations. Turkish display names (e.g., "Anonim Penguen", "Sakin Tilki", "Meraklı Baykuş"). Registration draws one at random via `WHERE category = 'free' AND isActive = true ORDER BY RANDOM() LIMIT 1` (Phase 3 Section 5 step 6). All 15 rows are interchangeable; there is no single "default" row.
- **Rank-locked (24 rows, `category: "rank-locked"`):** 3 avatars per rank (8 × 3 = 24), each with `requiredRankId` pointing to the relevant `Rank.id`. Themed to the rank identity — e.g., Gözlemci avatars evoke observer/binocular motifs, Yargıç avatars evoke gavel/scale/owl motifs, Allâme avatars evoke crown/star/manuscript motifs. The UI surfaces them as locked until the viewer's `max(totalXp, highestXpReached)` reaches `requiredRank.minXp`.
- **System (1 row, `category: "system"`, `isActive: true`, `requiredRankId: null`):** the "Silinmiş Kullanıcı" placeholder referenced by Phase 4 Section 5.1. Hidden from `GET /avatars` and never selectable via `PATCH /users/me/avatar`; only assigned internally by services that hydrate deleted-author cards.

All 40 rows use `url = "/avatars/{slug}.svg"`. The SVG assets are generated via DiceBear at build time and committed to `apps/web/public/avatars/` — see Phase 17 Section 7.2 for the `pnpm generate:avatars` build step. No external CDN dependency; Phase 16 CSP remains unchanged.

**XP Actions:**
- TOPIC_CREATE: 50 points
- COMMENT_CREATE: 10 points
- VOTE_CREATE: 5 points
- COMMENT_LIKE_RECEIVED: 2 points
- TOPIC_VOTE_RECEIVED: 1 point
- ADMIN_ADJUST: 0 points (actual points are set per request by admin — this action is exempt from daily XP limits)

**Restriction Categories and Reasons:**
- Uygunsuz İçerik → "Hakaret veya küfür içeren paylaşım", "Şiddete teşvik eden içerik", "Cinsel içerikli paylaşım"
- Spam / Reklam → "Ticari reklam içerikli paylaşım", "Tekrarlayan anlamsız içerik", "Dış bağlantı spamı"
- Kişisel Bilgi İhlali → "Başka bir kullanıcının kişisel bilgilerini paylaşma", "Gizlilik ihlali"
- Yanıltıcı İçerik → "Kasıtlı yanlış bilgi paylaşımı", "Sahte veya uydurma içerik"

**Consent Text Versions:**
- TERMS_OF_SERVICE v1.0.0
- PRIVACY_POLICY v1.0.0
- COMMUNITY_GUIDELINES v1.0.0
Each should be written in a professional and formal tone (in Turkish). They should be as detailed as if prepared by a real lawyer.

**Admin user:**
- email: read from the `ADMIN_EMAIL` env variable (validated at startup by Phase 1 configuration)
- password: read from the `ADMIN_PASSWORD` env variable, hashed with bcrypt (same salt rounds as normal registration)
- username: `admin`
- firstName / lastName: `"Yönetici"` / `"Hakver"` placeholder values (admin can edit post-login)
- role: ADMIN
- `emailVerifiedAt`: set to `now()` so the admin account can perform write actions without going through the verification flow
- `currentRankId`: the id of the lowest-threshold rank (Gözlemci, 0 XP) — satisfies the non-null `currentRankId` invariant applied to every account (Phase 3 Section 5 step 8)
- `highestXpReached`: 0
- `avatarId`: drawn from the free-pool using the same random-selection rule as new registrations (`WHERE category = 'free' AND isActive = true ORDER BY RANDOM() LIMIT 1`)

The seed script must skip admin creation when a user with the `ADMIN_EMAIL` address already exists (idempotent re-seed). The plaintext `ADMIN_PASSWORD` is never persisted — only the bcrypt hash is written to the database.

**`UserNotificationPreference` seed defaults:** The table is empty when a user registers (absent row = default enabled). The registration flow does not insert preference rows; the first toggle on the settings page creates the row on demand.

## Security Checklist
- [ ] Soft delete middleware is active and working correctly
- [ ] UUIDs are used (instead of auto-increment — prevents ID predictability)
- [ ] Unique constraints are correctly defined
- [ ] Cascade delete rules are carefully configured (content should be preserved when a user is deleted)
- [ ] Admin password in seed data is read from `.env`

## Test Plan

```bash
# 1. Does migration run?
cd apps/api && npx prisma migrate dev --name init

# 2. Was Prisma Client generated?
npx prisma generate

# 3. Does seed run?
npx prisma db seed

# 4. Is the database connection correct?
npx prisma db pull  # Verify the existing schema

# 5. Does PrismaService work in NestJS?
# Start the backend, call the health endpoint — no errors
pnpm --filter api start:dev

# 6. Is seed data correct?
# Check with Prisma Studio:
npx prisma studio
# Is there data in the Roles, Permissions, Categories, Avatars, XpActions, Ranks tables?
```

## Completion Criteria
- [ ] Prisma schema contains all tables
- [ ] `prisma migrate dev` completes without errors
- [ ] `prisma db seed` loads all initial data
- [ ] PrismaService is globally accessible in the backend
- [ ] Soft delete middleware is active
- [ ] All tables and seed data are visible in Prisma Studio
- [ ] Backend starts without errors
