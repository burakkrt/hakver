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
NotificationType: TOPIC_VOTED, TOPIC_COMMENTED, COMMENT_LIKED, COMMENT_REPLIED
ReferenceType: TOPIC, COMMENT, VOTE, COMMENT_LIKE
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
| actorId | FK → User | the user who triggered the notification |
| referenceType | ReferenceType enum | |
| referenceId | UUID | |
| isRead | Boolean | default false |

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

### 4. Indexes

Additional indexes for performance:
- `Topic`: `(categoryId, createdAt DESC)` — category-based listing
- `Topic`: `(authorId, createdAt DESC)` — user topics
- `Comment`: `(topicId, likeCount DESC)` — like-based sorting
- `Vote`: `(topicId)` — topic-based vote counting
- `XpLog`: `(userId, createdAt)` — daily XP calculation
- `ActivityLog`: `(userId, createdAt DESC)` — activity queries
- `Notification`: `(userId, isRead, createdAt DESC)` — unread notifications
- `UserRestriction`: `(userId, expiresAt)` — active restriction check
- `UserBlock`: `(blockedId)` — reverse block lookup for symmetric block checks

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

**Ranks:**
- Çaylak: 0 XP
- Üye: 100 XP
- Aktif Üye: 500 XP
- Deneyimli: 1500 XP
- Uzman: 5000 XP
- Usta: 15000 XP

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
