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
NotificationType: TOPIC_VOTED, TOPIC_COMMENTED, COMMENT_LIKED, COMMENT_REPLIED, MENTIONED, TOPIC_UPDATED, COMMENT_HIGHLIGHTED
ReferenceType: TOPIC, COMMENT, VOTE, COMMENT_LIKE, USER
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
| currentRankId | FK → Rank? | |
| provider | AuthProvider enum | default LOCAL |
| providerId | String? | Google OAuth ID |
| usernameChangedAt | DateTime? | |
| emailChangedAt | DateTime? | |
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
| name | String | |
| url | String | |
| isDefault | Boolean | default false |
| isActive | Boolean | default true |

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
| slug | String | unique, index — auto-generated from title, for SEO-friendly URLs |
| authorId | FK → User | index |
| categoryId | FK → Category | index |
| isAnonymous | Boolean | default false |
| isEdited | Boolean | default false |
| editableUntil | DateTime | createdAt + 12 hours |
| voteCountRight | Int | default 0, denormalized |
| voteCountWrong | Int | default 0, denormalized |
| commentCount | Int | default 0, denormalized |
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

#### XpLog
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| actionId | FK → XpAction | |
| points | Int | positive = earned, negative = revoked |
| referenceType | ReferenceType enum | |
| referenceId | UUID | related entity ID |
| metadata | Json? | additional context (e.g., `{ topicId }` for comment XP, `{ voterId }` for vote-received XP, `{ reason, adjustedBy }` for admin adjustments) |

#### Rank
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| name | String | |
| minXp | Int | unique |
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
| actorId | FK → User | the **latest** actor that triggered/updated this notification (for aggregation, this is the most recent contributor). Nullable for system notifications |
| referenceType | ReferenceType enum | |
| referenceId | UUID | |
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

#### Badge (badge definition)
Comprehensive, tiered badge system. The base tables live here; the full badge catalog and the XP bonus / advantage semantics tied to some badges are finalized in a **separate design document** (see "Detailed Design Pending Items" in `plan.md`).

| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| code | String | unique — program-readable identifier (e.g., `loyal_bronze`, `loyal_silver`, `vote_master_gold`) |
| name | String | Turkish user-facing display name (e.g., "Sadık", "Oy Ustası") |
| description | String | max 500 — Turkish description shown when the badge is awarded or displayed on the profile |
| icon | String | icon identifier (Lucide name or custom SVG key) |
| tier | String? | null = tierless single badge. Otherwise: BRONZE / SILVER / GOLD |
| group | String? | groups a tiered chain under one code (e.g., `loyal` aggregates all "Sadık" tiers) |
| conditionType | String | action type (TOPIC_CREATE, VOTE_CREATE, COMMENT_LIKE_RECEIVED, STREAK_DAYS, XP_THRESHOLD, MANUAL, …) |
| conditionValue | Int | threshold value (e.g., 100 votes) |
| bonusEffect | Json? | **Pending design — placeholder field.** Carries the bonus semantics for badges that grant an XP gain percentage or other advantage. Example shape: `{ type: "XP_GAIN_MULTIPLIER", scope: "VOTE_CREATE", percent: 10 }`. Stored as JSON so that the full catalog and effect semantics can be finalized in the separate design before launch without a schema migration |
| sortOrder | Int | UI ordering |
| isActive | Boolean | default true |

#### UserBadge (badges awarded to a user)
| Field | Type | Constraint |
|-------|------|-----------|
| id | UUID | PK |
| userId | FK → User | index |
| badgeId | FK → Badge | |
| awardedAt | DateTime | default now |
| | | @@unique([userId, badgeId]) — the same badge is never awarded twice to the same user |

### 4. Indexes

Additional indexes for performance:
- `Topic`: `(categoryId, createdAt DESC)` — category-based listing
- `Topic`: `(authorId, createdAt DESC)` — user topics
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
- `Badge`: `(group, tier)` — tier chain queries
- `UserBadge`: `(userId, awardedAt DESC)` — profile badge listing

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

**Avatars:**
- 6-8 default avatars (placeholder URLs, real images to be added later). One with `isDefault: true`.

**XP Actions:**
- TOPIC_CREATE: 50 points
- COMMENT_CREATE: 10 points
- VOTE_CREATE: 5 points
- COMMENT_LIKE_RECEIVED: 2 points
- TOPIC_VOTE_RECEIVED: 1 point
- ADMIN_ADJUST: 0 points (actual points are set per request by admin — this action is exempt from daily XP limits)

**Ranks (10 tiers, Justice / Jury theme — Turkish display names):**
- Yeni Vatandaş: 0 XP
- Gözlemci: 100 XP
- Tanık: 300 XP
- Jüri Adayı: 750 XP
- Jüri Üyesi: 1750 XP
- Kıdemli Jüri: 4000 XP
- Hakem: 8500 XP
- Bilirkişi: 17500 XP
- Yargıç: 35000 XP
- Adalet Bilgesi: 70000 XP

Progression: each tier is roughly 2–2.3x the previous threshold. Early tiers reward quickly to motivate new users; upper tiers act as long-term prestige. The top tier (70k XP) is aimed at the top ~1% of users. The curve is slightly steeper than the original 5x step because the user asked for 10 tiers instead of 6; overall gradient stays close to ~2.2x per tier.

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
- email: from environment variable or default
- username: admin
- role: ADMIN

**Badge seed (placeholder — detailed design pending):**

The badge system is part of the MVP per the user's decision. A comprehensive + tiered structure, together with XP gain bonuses on certain badges, is planned. The full catalog (name, condition, tier thresholds, bonus effects) is finalized in a **separate design document**. For the seed stage:

- Starter milestone badges (Bronze tier only) are included in the seed: "Hoş Geldin" (email verification), "İlk Oy", "İlk Konu", "İlk Yorum" — `conditionType: MANUAL` or the matching action + `conditionValue: 1`, `bonusEffect: null`
- The full tiered badge catalog (Sadık Bronze/Silver/Gold, Oy Ustası, Halk Kahramanı, etc.) and XP bonus effects will be added via a separate design document + PR before launch. This placeholder keeps the schema exercised and provides enough badges for MVP smoke tests

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
