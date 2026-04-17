# Phase 10: Notifications & Activity Logging

## Dependencies
- Phase 6 (voting), Phase 7 (comments), and Phase 9 (moderation) must be completed

## Goals
- Notification creation (vote, comment, like, reply)
- Notification listing and mark-as-read
- Topic-based notification muting
- Real-time notifications via WebSocket
- Activity logging (all user actions)

## Tasks

### 1. Notification Module

Under `apps/api/src/modules/notification/`:

```
notification/
├── notification.module.ts
├── notification.controller.ts
├── notification.service.ts
└── notification.gateway.ts
```

### 1.1. Shared Notification Response Schema

In `packages/shared/src/schemas/notification.ts`:

**NotificationResponseSchema** (response DTO — single canonical shape for every notification, single-actor and aggregated):
- id: UUID
- type: NotificationType enum
- isRead: boolean
- createdAt: ISO date string
- updatedAt: ISO date string (latest actor-append time; equals `createdAt` when `aggregatedCount === 1`)
- aggregatedCount: integer ≥ 1 (1 for single-actor notifications, >1 for aggregated)
- actors: `UserPublicCardSchema[]` (length 0..3). For `RANK_UP` the array is empty (the recipient is the only participant and the card is driven by the rank badge). The last three distinct actors sit here in most-recent-first order for engagement types. Elements are card-shape only — firstName and lastName are never included
- isAnonymousAggregate: boolean (true when every actor rolled into the notification is an anonymous author; in that case `actors` is returned as an empty array, see aggregation rules in Section 3 step 9)
- reference: `{ type: ReferenceType, id: UUID, title?: string }` — for `RANK_UP`, `type: "RANK"`, `id: rankId`, `title: rank.name` (e.g., `"Yargıç"`); the frontend hydrates the rank icon + color from its local ranks cache (`iconSlug`, `color` live on the `Rank` row)
- isStale: boolean (true when the referenced target is soft-deleted; see Section 5 L-4). Always `false` for `RANK_UP` — ranks are never deleted
- message: string (Turkish, user-facing — generated from type + actors array size + anonymity flags). For `ADMIN_BROADCAST` this field is left empty; the frontend renders `title` and `body` instead
- title: string | null (populated only for `ADMIN_BROADCAST`; see Section 3.1)
- body: string | null (populated only for `ADMIN_BROADCAST`; max 1000 characters)

> The singular `actor` field that earlier drafts described is deliberately absent. Frontend and backend both read and write `actors` only, which keeps client rendering logic uniform for single-actor and aggregated cards.

### 2. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /notifications | JWT | Notification list (paginated, aggregated + stale flag dahil) |
| GET | /notifications/unread-count | JWT | Unread notification count |
| PATCH | /notifications/:id/read | JWT | Mark notification as read |
| PATCH | /notifications/read-all | JWT | Mark all as read |
| POST | /topics/:topicId/mute | JWT | Mute topic notifications (author/subscriber opt-out for this specific topic) |
| DELETE | /topics/:topicId/mute | JWT | Unmute topic notifications |
| POST | /topics/:topicId/subscribe | JWT | Opt in to `TOPIC_UPDATED` notifications for this topic (non-authors only; authors are subscribed implicitly) |
| DELETE | /topics/:topicId/subscribe | JWT | Opt out of `TOPIC_UPDATED` notifications for this topic |
| GET | /users/me/notification-preferences | JWT | All notification types + their `enabled` status (missing rows default to enabled) |
| PATCH | /users/me/notification-preferences | JWT | Update the `enabled` flag for a single type. Body: `{ type: NotificationType, enabled: boolean }`. Sending `MENTIONED` or `TOPIC_UPDATED` returns 400 with message "Bu bildirim türü devre dışı bırakılamaz" (non-toggleable personal signals) |

### 3. Notification Creation Service

**`notification.service.ts`** — `createNotification(data)` (checks run in order; any negative check ends the call without creating a record):

1. **Recipient health check:** skip when `User.deletedAt IS NOT NULL` — soft-deleted accounts never receive notifications
2. **Self-notification prevention:** `actorId === userId` → skip (e.g., liking own comment). **Exception: `RANK_UP`** — this type is a self-signal about the user's own progression; the check is explicitly bypassed so the recipient receives the notification even though `actorId === userId`
3. **Block check:** skip if a `UserBlock` row exists between recipient and actor in **either direction**. Does not apply to `RANK_UP` (actor is the recipient themselves)
4. **Per-type preference check:** skip if `UserNotificationPreference(userId, type)` exists with `enabled=false`. Absent row means enabled by default. `MENTIONED` and `TOPIC_UPDATED` are non-toggleable on the backend — the preference check is bypassed for these two types. `RANK_UP` is toggleable (default on) — users who want a silent progression experience can disable it from the settings page
5. **Mute check** (topic-based): skip if `NotificationMute(userId, topicId)` exists for the referenced topic. **Exception: `MENTIONED` bypasses this check** — @mention is a personal signal and punches through topic mute. `RANK_UP` has no topic reference so this check is a no-op
6. **Actor-target idempotency** (L-5): applies to `TOPIC_VOTED`, `COMMENT_LIKED`, `TOPIC_COMMENTED`, and `COMMENT_REPLIED`. If the same `(userId=recipient, type, referenceId, actorId)` quadruple already has a record (read or unread) within the last 24 hours, **do not create a new record**; instead bump the existing record's `updatedAt`. This suppresses repeats across vote-withdraw-revote, like-unlike-like, and comment/reply delete-and-repost cycles (Phase 7 allows rapid soft-delete + recreate, which without this guard would spam the topic author with repeated notifications for the same adversary). Edge cases:
   - **Comment was soft-deleted then recreated:** the second `TOPIC_COMMENTED` within 24h matches the first via `referenceId = topicId` and `actorId = same user`; the existing (possibly read) notification is not duplicated
   - **Comment was soft-deleted then *edited* (author edits the original, not re-create):** no new notification fires — edit is not a trigger in Phase 7
   - **Two distinct comments from the same actor on the same topic within 24h:** still a single notification record (the semantic is "user X is discussing your topic", one card is enough). The `aggregatedCount` / `NotificationActor` row counts stay at 1 since the same actor is not appended twice (the `@@unique([notificationId, actorId])` constraint would reject a duplicate actor row anyway)
   - **Different parent comments from the same actor for `COMMENT_REPLIED`:** idempotency keys on `referenceId = parentCommentId`, so replies under different parents each produce their own notification to the respective parent author
7. **Aggregation check** (L-1): if `type ∈ { TOPIC_VOTED, TOPIC_COMMENTED, COMMENT_LIKED, COMMENT_REPLIED }`, look for an existing record with the same `(userId, type, referenceType, referenceId)`, `isRead=false`, and `updatedAt >= now() - 15min`:
   - **Found:** append the new actor to `NotificationActor` (the `@@unique` constraint drops duplicate actor rows), `aggregatedCount += 1`, set `actorId` to the new actor, bump `updatedAt = now()`. Emit a `notification:updated` WebSocket event so the client refreshes the existing card in place (L-8: no new card inserted)
   - **Not found:** fall through to step 8 and create a fresh record
8. **Create a new record:** insert `Notification` (`aggregatedCount=1`, first actor row in `NotificationActor`). Emit `notification:new` WebSocket event
9. **Anonymous actor** (L-9): when the actor acts anonymously (anonymous topic/comment author), aggregation is **mandatory** — a single anonymous actor never renders as a standalone identity card. Even one anonymous actor folds into the aggregated shape; the message format is always "Bir kullanıcı konunuza oy verdi" (single) or "N kullanıcı konunuza oy verdi" (multi), never the real anonymous display name like "Anonim Penguen #4521". This rule does **not** apply to `MENTIONED` (mention targets a specific user; the anonymous card is shown on that user's notification because the timing correlation risk is an accepted trade-off for explicit mentions). **Security note:** on `MENTIONED` notifications the anonymous actor's display name (e.g., "Anonim Penguen #4521") is returned only to the mentioned user; no other API response pairs this name with the comment. The frontend scrolls from the mention notification to the comment where the anonymous card is still rendered as anonymous — the real identity never leaks

**`NotificationActor.isAnonymousForReference` computation (authoritative):** whenever a `NotificationActor` row is inserted (aggregation append in step 7 OR fresh record in step 8), the notification service writes the precomputed `isAnonymousForReference` flag in the same statement:
- Reference type `TOPIC` (`TOPIC_VOTED`, `TOPIC_COMMENTED`): `isAnonymousForReference = Topic.isAnonymous OR EXISTS(SELECT 1 FROM "AnonymousIdentity" WHERE "userId" = :actorId AND "topicId" = :referenceId)`. Both branches are already loaded by the caller (the voting/commenting service already fetched the topic row; anonymous-identity creation happens right before the notification call), so no extra query is needed
- Reference type `COMMENT` (`COMMENT_LIKED`, `COMMENT_REPLIED`): `isAnonymousForReference = triggeringComment.isAnonymous` — the comment being liked/replied-to is already loaded by the caller
- Reference type `USER` (`ADMIN_BROADCAST`) and `RANK` (`RANK_UP`): `isAnonymousForReference = false` (broadcast and rank-up actors are never anonymous)
- `MENTIONED`: the actor's anonymity is resolved from the comment containing the mention and stored on the actor row; the mention-side rendering rule from step 9 still applies at listing time

The flag is written once at insert time so `Phase 10 Section 5` listing reads it directly without any join to `AnonymousIdentity` or `Comment`.

**Actor hydration** (L-6): each element of the `actors` array in notification listings and WebSocket event payloads is **always** hydrated via `UserPublicCardSchema`. For soft-deleted actors the schema rule returns `isDeletedAuthor: true` and masks the card fields ("Silinmiş Kullanıcı" label, default "silinmis" avatar, rank omitted, `totalXp: 0`). Anonymous actors never appear hydrated in the array — when the actor is anonymous the record is flagged with `isAnonymousAggregate: true` and `actors` returns `[]` (see Section 5 and the L-9 security rule below). No code path leaks raw username / full name to the frontend.

### 3.1. Admin Broadcast Notification

Users with the `system:manage` permission (ADMIN role at MVP; future authority roles may also gain it) can dispatch an `ADMIN_BROADCAST` notification to an explicit recipient set. This is the MVP mechanism for service announcements ("Bakım penceresi gece 02:00–03:00 arasında", "Yeni topluluk kuralı yayınlandı").

**Endpoint:** `POST /admin/notifications/broadcast` (ADMIN only, requires `system:manage` permission).

**Request schema** (new in `packages/shared/src/schemas/admin.ts`):

```
AdminBroadcastRequestSchema = {
  title: string          // 5..120 chars, HTML-stripped, Turkish user-facing
  body: string           // 10..1000 chars, HTML-stripped, Turkish user-facing
  audience: {
    scope: "ALL" | "SELECTED_USERS" | "ROLE"
    userIds?: UUID[]     // required when scope = SELECTED_USERS (max 5000 ids per request)
    roleName?: string    // required when scope = ROLE (e.g., "MODERATOR")
  }
  acknowledgementReason: string  // 5..500 chars — audit trail, stored in ActivityLog
}
```

**Flow:**
1. Validate request. `audience.scope` and the accompanying field must be consistent (Zod refinement returns 400 `{ code: "VALIDATION_INVALID_INPUT" }` on mismatch)
2. Build the recipient id list from the scope. Exclude the broadcasting admin themselves, soft-deleted accounts, and — for `SELECTED_USERS` — deduplicate and drop ids that do not exist
3. Enqueue a BullMQ `admin-broadcast-fanout` job carrying `{ actorId, title, body, recipientBatch }` in 500-recipient chunks, mirroring the Phase 10 `TOPIC_UPDATED` fan-out worker pattern. This keeps individual transactions bounded and isolates batch failures
4. Each worker inserts `Notification` rows with `type: ADMIN_BROADCAST`, `actorId: broadcaster`, `referenceType: USER`, `referenceId: broadcaster`, the Turkish `title` and `body` fields populated, and `aggregatedCount: 1`. Aggregation, actor-target idempotency, and topic-mute checks **do not apply** — broadcasts are always delivered in full, once per recipient
5. Per-type preference check still applies: `UserNotificationPreference(userId, type=ADMIN_BROADCAST)` with `enabled=false` suppresses delivery. The preference defaults to `true` (users receive broadcasts unless they explicitly disable them) and `ADMIN_BROADCAST` is **toggleable** from the notification settings UI, unlike `MENTIONED` and `TOPIC_UPDATED`
6. The service writes an ActivityLog entry `action: "admin:notifications-broadcast"` carrying `{ broadcasterId, audienceScope, recipientCount, title, acknowledgementReason }`. This is a mandatory audit step — the reason field exists precisely for accountability
7. WebSocket delivery follows the standard `notification:new` event so online recipients see the card appear immediately
8. Response: 202 Accepted with `{ jobId, estimatedRecipients }`. Actual delivery happens asynchronously; the admin UI polls the job status endpoint (`GET /admin/notifications/broadcasts/:jobId`) to show progress

**Error handling:**
- `403 FORBIDDEN` when the caller lacks `system:manage` permission (ADMIN-only at MVP)
- `400 MODERATION_REASON_REQUIRED` when `acknowledgementReason` is absent or too short (reuses the existing moderator-action reason pattern)
- `400 VALIDATION_INVALID_INPUT` for schema issues
- `429 RATE_LIMIT_EXCEEDED` when the admin exceeds 5 broadcasts per hour per admin (prevents abuse). 429 message follows the Turkish format described in `.claude/rules/security.md`

**Message rendering:** The frontend renders `ADMIN_BROADCAST` notifications with a distinct visual treatment — primary-tone accent bar plus a "Yönetici Duyurusu" chip — so they are never confused with engagement notifications. The `message` field is ignored for this type; the UI reads `title` and `body` directly from the notification payload.

### 4. Notification Triggers

Add notification hooks to existing modules:

**Vote Module:**
- When a vote is cast → `TOPIC_VOTED` notification to topic owner. Aggregation applies (L-1); popular topics collapse into "ayse_k ve 14 kullanıcı konunuza oy verdi". Actor-target idempotency (L-5): vote-withdraw-revote cycle resolves to a single notification
- On vote change → no new notification
- On vote withdrawal → no new notification. The existing record is not deleted; idempotency ensures a subsequent re-vote does not repeat the notification

**Comment Module:**
- When a comment is posted → `TOPIC_COMMENTED` notification to topic owner. Aggregation applies
- When a reply is posted → `COMMENT_REPLIED` notification to parent comment owner. Aggregation applies
- When a like is given → `COMMENT_LIKED` notification to comment owner. Aggregation + actor-target idempotency apply (like-unlike-like cycle resolves to a single notification)
- When a user is @mentioned in a comment → `MENTIONED` notification to the mentioned user. Reference: the comment containing the mention. **Aggregation does not apply** — each mention is a single notification. **Topic mute exception:** mention notifications are emitted even on muted topics (L-3). **Anonymous actor privacy rule (updated):** when the commenter is anonymous, the notification is persisted with `isAnonymousAggregate: true` and an empty `actors` array. The rendered Turkish message becomes `"Bir kullanıcı sizi bir yorumda etiketledi"` — the anonymous display name (`"Anonim Penguen #4521"`) is never surfaced on this notification. The mention recipient still clicks through to the comment, where the anonymous card remains anonymous; they never get an extra "which anonymous identity mentioned me" hint that could help triangulate the real author on a topic the user is also watching
- When the topic author highlights a comment (`PATCH /comments/:id/highlight`) → `COMMENT_HIGHLIGHTED` notification to the comment author. Aggregation does not apply. Skipped when the topic author highlights their own comment (self-notification prevention). Removing the highlight does not trigger a notification. Idempotent PATCH (re-highlighting an already-highlighted comment) does not produce a duplicate notification

**Topic Module:**
- When the author writes or replaces an update note (`PATCH /topics/:id/update-note`) → `TOPIC_UPDATED` notifications are fanned out asynchronously via BullMQ. Clearing the update note does not trigger a notification.

**XP Module (Phase 8):**
- When `checkAndUpdateRank` detects a first-time rank summit (new rank's `minXp` exceeds the previous `highestXpReached`) → `RANK_UP` notification to the user. `actorId` is set to the user themselves (self-notification prevention is explicitly bypassed for this type in Section 3 step 2). `referenceType: RANK`, `referenceId: newRankId`, `title: rank.name`. `actors` is stored as an empty set — the recipient is the only participant and the frontend renders a rank-badge card instead of an actor card. Aggregation does not apply. Actor-target idempotency is already enforced upstream via the `highestXpReached` watermark (Phase 8 Section 2), so notification-side idempotency is not needed. Per-type preference applies (default on; user may disable via `/users/me/notification-preferences`).

**Default-off subscription policy (authoritative):**

Voting on or commenting on a topic does **not** subscribe a user to that topic's update notifications. The platform-wide default is: non-author users receive no `TOPIC_UPDATED` notification unless they explicitly subscribed via the topic detail page. This keeps notification volume predictable for popular threads — a topic with thousands of voters does not inundate all of them every time the author edits the note.

- Authors are always considered subscribed to their own topics (implicit; no `TopicSubscription` row needed). Self-notification prevention still filters them out of update-note fan-out because they are the actor
- Non-author users opt in by calling `POST /topics/:topicId/subscribe` (Phase 10 Section 7 endpoints), which creates a `TopicSubscription` row. They can opt out via `DELETE /topics/:topicId/subscribe`
- `NotificationMute` remains available for authors who want to silence notifications about activity on their own topics (e.g., voting/commenting noise); it is **not** the primary opt-out path for TOPIC_UPDATED because non-authors are opt-in by default
- Engagement-type notifications targeted at the topic author (`TOPIC_VOTED`, `TOPIC_COMMENTED`) stay default-on for the author — they still learn when their topic receives votes and comments. The author can disable those via the per-type toggle in notification settings (Phase 12)

**Fan-out recipient rules (`TOPIC_UPDATED`):**
- Include: users with a `TopicSubscription` row for this topic
- Exclude the topic author (they are the actor who triggered the update)
- Exclude users whose `User.deletedAt IS NOT NULL`
- Exclude users who muted the topic (`NotificationMute` row) — mute wins over subscription, so a subscriber who later mutes stops receiving updates without needing to unsubscribe
- Exclude any user under a symmetric block with the author (see Phase 9 `BlockService.isBlocked`)
- Anonymous commenters who explicitly subscribed still count — the anonymity applies only to how they appear in the discussion, not to whether they receive notifications they actively opted into

**Fan-out execution (BullMQ):**
- The fan-out producer runs the single recipient query (`SELECT userId FROM TopicSubscription WHERE topicId = :topicId` joined with the exclusion filters above) and splits the result into **batches of 500 recipients**
- Each batch is enqueued as its own BullMQ job (`topic-updated-fanout` queue). Jobs run in parallel with the worker concurrency limit
- **Recipient recheck inside each job (authoritative):** immediately before the `createMany` call, the worker runs `SELECT id FROM "User" WHERE id IN (:batchIds) AND "deletedAt" IS NULL` and keeps only the rows that still resolve to active accounts. This closes the race where a recipient soft-deletes their account between producer enqueue and worker execution — KVKK requires that no new personal data (a notification row referencing the user) be written for a retired account. The same recheck pattern applies to the `admin-broadcast-fanout` worker (Section 3.1)
- Each job then inserts the filtered batch of `Notification` rows in a single `createMany` and pushes WebSocket events to each online recipient
- Failed jobs use the standard retry policy (3 attempts, exponential backoff) and fall back to the dead-letter queue
- Chunking keeps individual transactions bounded, isolates batch failures, and lets the worker concurrency dictate throughput

**Deduplication (DB-enforced):**
- A partial unique index on `Notification` guarantees at most one unread `TOPIC_UPDATED` per recipient per topic: `UNIQUE (userId, referenceId, type) WHERE type = 'TOPIC_UPDATED' AND isRead = false`. This is a raw SQL migration (Prisma declarative syntax does not yet cover partial indexes in all versions; `prisma migrate` supports them through `@@map` + SQL). The index turns a duplicate insert into a caught exception that the worker converts into an `updatedAt` bump on the existing row
- **Producer-side debounce:** when the author issues a second `PATCH /topics/:id/update-note` within 10 seconds of the previous one, the update-note service replaces the in-flight fan-out job instead of enqueuing a new one. Implementation: each fan-out job carries a `jobId` derived from `topic-updated:{topicId}`; enqueue with `removeOnComplete` + `removeOnFail` and BullMQ's `upsert` / deterministic job id so a second enqueue inside the debounce window replaces the pending job. The final job wins
- Combined, the partial index is the safety net (catches race conditions across workers) and the debounce is the efficiency layer (avoids wasted work when the author edits quickly)

### 5. Notification Listing

`GET /notifications?page=1&limit=20`:

```json
{
  "data": [
    {
      "id": "...",
      "type": "TOPIC_VOTED",
      "isRead": false,
      "isStale": false,
      "createdAt": "...",
      "updatedAt": "...",
      "aggregatedCount": 15,
      "actors": [
        {
          "id": "...",
          "username": "ayse_k",
          "avatar": { "url": "/avatars/meraklı-baykus.svg" },
          "rank": { "name": "Yargıç", "iconSlug": "gavel", "color": "indigo" },
          "totalXp": 1750,
          "isDeletedAuthor": false
        }
      ],
      "reference": {
        "type": "TOPIC",
        "id": "...",
        "title": "Konu başlığı..."
      },
      "message": "ayse_k ve 14 kullanıcı konunuza oy verdi"
    }
  ],
  "meta": { "total": 50, "page": 1, "limit": 20 }
}
```

**`actors` field:** for an aggregated notification, the latest 3 actors (most recent `NotificationActor` rows) are returned hydrated via the card schema. If `aggregatedCount > 3`, the first 3 are shown and the rest is surfaced through the user-facing Turkish suffix "ve N diğer kullanıcı" inside the message. For single-actor notifications (`aggregatedCount === 1`) the `actors` array contains a single element.

**Listing query plan (authoritative — avoids N+1):** the listing service runs **three bounded queries per page** regardless of page size:
1. Notification page — standard paginated query: `SELECT ... FROM "Notification" WHERE "userId" = :me ORDER BY "updatedAt" DESC LIMIT :limit OFFSET :offset`. Collect the page's notification ids
2. Actor rows for the whole page in one shot: `SELECT "notificationId", "actorId", "isAnonymousForReference", "createdAt" FROM "NotificationActor" WHERE "notificationId" IN (:pageIds) ORDER BY "notificationId", "createdAt" DESC`. In the service layer, group by `notificationId` and keep the top 3 per group. Collect the distinct `actorId` set across all groups
3. Actor hydration in one batch: `SELECT ... FROM "User" WHERE id IN (:distinctActorIds)` — mapped to `UserPublicCardSchema` (with `isDeletedAuthor` propagation from Phase 4 Section 5.1)

The service then assembles each notification's `actors` array from the hydrated user rows, derives `isAnonymousAggregate` as "every top-3 actor row carries `isAnonymousForReference = true`" (single pass over the already-loaded rows — no extra query), and emits the final payload. `RANK_UP` short-circuits this plan entirely because its `actors` array is always empty. This 3-query ceiling keeps the notification list fast regardless of how many aggregated rows each card carries.

**`isStale` field** (L-4): during notification listing, if the `reference` target (topic or comment) is soft-deleted (`deletedAt IS NOT NULL`), the response flags `isStale: true`. The frontend renders the card grey/italic, disables navigation, or shows a toast "Bu içerik kaldırıldı" on click (details: Phase 14).

**Anonymous actor — aggregation (L-9 security rule):** when the actor(s) are anonymous, the `actors` array is omitted; only a count is returned and the message is crafted to hide the anonymous identity:
```json
{
  "type": "TOPIC_COMMENTED",
  "aggregatedCount": 1,
  "actors": [],
  "isAnonymousAggregate": true,
  "message": "Bir kullanıcı konunuza yorum yaptı"
}
```
When `aggregatedCount > 1` the message becomes "N kullanıcı konunuza yorum yaptı". This prevents leakage of a lone anonymous actor's display name (e.g., "Anonim Penguen #4521") through the notification channel — the anonymous card on the topic detail page still protects identity.

**Anonymous `MENTIONED` also hides the display name** (updated rule): when the mention comes from an anonymous commenter, the frontend renders `"Bir kullanıcı sizi bir yorumda etiketledi"` instead of prefixing the anonymous display name. The earlier draft's "accepted trade-off" exception is retired — there is no product gain in revealing which anonymous identity mentioned the user; the recipient can still click through to the comment where the anonymous card stays intact, and removing the name from the notification closes the remaining triangulation vector on topics the mention recipient is also browsing.

**Notification messages** (Turkish user-facing strings):

Singular (`aggregatedCount === 1`):
- `TOPIC_VOTED`: "{actor} konunuza oy verdi"
- `TOPIC_COMMENTED`: "{actor} konunuza yorum yaptı"
- `COMMENT_LIKED`: "{actor} yorumunuzu beğendi"
- `COMMENT_REPLIED`: "{actor} yorumunuza yanıt verdi"
- `MENTIONED`: "{actor} sizi bir yorumda etiketledi"
- `TOPIC_UPDATED`: "{actor} ilgilendiğiniz konuya bir güncelleme ekledi"
- `COMMENT_HIGHLIGHTED`: "{actor} yorumunuzu öne çıkardı"
- `RANK_UP`: "Yeni rütbe kazandın: {rank.name}" — `{rank.name}` is pulled from the notification's `reference.title` populated at creation time

Aggregated (`aggregatedCount > 1`, only for the 4 aggregatable types):
- `TOPIC_VOTED`: "{latest_actor} ve {count-1} kullanıcı konunuza oy verdi"
- `TOPIC_COMMENTED`: "{latest_actor} ve {count-1} kullanıcı konunuza yorum yaptı"
- `COMMENT_LIKED`: "{latest_actor} ve {count-1} kullanıcı yorumunuzu beğendi"
- `COMMENT_REPLIED`: "{latest_actor} ve {count-1} kullanıcı yorumunuza yanıt verdi"

Anonymous aggregate (all actors are anonymous):
- Singular (count=1): "Bir kullanıcı konunuza oy verdi / yorum yaptı / ..."
- Aggregated (count>1): "{count} kullanıcı konunuza oy verdi / ..."

### 6. WebSocket Notification Delivery

In addition to the gateway set up in Phase 6:

**User-based room:** Each authenticated user automatically joins the `user:{userId}` room (on connection establishment).

**Events:**
- Server → Client: `notification:new` → notification object
- Server → Client: `notification:unread-count` → `{ count: number }`

### 7. Topic-Based Muting and Subscription

**Muting** silences every notification the current user would otherwise receive about a specific topic, regardless of why they would receive it. It is the catch-all opt-out path — useful for authors who want to ignore a noisy topic they created, or for subscribers who no longer want the updates.

`POST /topics/:topicId/mute`:
1. Create `NotificationMute` record (`userId, topicId`)
2. Unique constraint → if already muted → 409

`DELETE /topics/:topicId/mute`:
1. Delete record
2. If not found → 404

**Subscription** (non-authors only) is the opt-in path for `TOPIC_UPDATED`. Without a subscription, a user who voted or commented on a topic receives no update notifications by default.

`POST /topics/:topicId/subscribe`:
1. Verify the caller is not the topic author (authors are implicitly subscribed; a self-subscribe attempt returns 400 `{ code: "VALIDATION_INVALID_INPUT", message: "Kendi konunuza abone olamazsınız" }`)
2. Create `TopicSubscription` row (`userId, topicId`)
3. Unique constraint → already subscribed → 409
4. Response: 201

`DELETE /topics/:topicId/subscribe`:
1. Delete the `TopicSubscription` row
2. If not found → 404
3. Response: 200

Both mute and subscription states are returned as flags on the topic detail response:
```json
{
  "id": "...",
  "title": "...",
  "isMuted": true,       // if authenticated and NotificationMute row exists
  "isSubscribed": false  // if authenticated and non-author; for the author this field is always true
}
```

**Interaction between mute and subscription:** Mute wins. A user who is subscribed AND has muted the topic receives no notification — they do not need to unsubscribe first. This keeps the opt-out path predictable when the user just wants the notifications to stop.

### 8. Activity Log Module

Under `apps/api/src/modules/activity-log/`:

```
activity-log/
├── activity-log.module.ts
├── activity-log.service.ts
└── activity-log.interceptor.ts
```

### 9. Activity Log Service

**`activity-log.service.ts`** — `log(data)`:

```typescript
interface ActivityLogData {
  userId: string;
  action: string;         // e.g.: "topic:create", "vote:create", "comment:like"
  targetType?: string;    // e.g.: "TOPIC", "COMMENT"
  targetId?: string;
  metadata?: Record<string, any>;
  ipAddress?: string;
  userAgent?: string;
}
```

**Logging points** (add to existing modules):
- Topic create, edit, delete
- Vote cast, change, withdraw
- Comment create, edit, delete
- Comment like/unlike
- Profile update
- Username/email change
- Account deletion
- Restriction apply/remove
- Report create/review
- Block/unblock
- Login
- **Rank change — first-time summit** (`user:rank-up`): written by `checkAndUpdateRank` (Phase 8 Section 2) when the new rank's `minXp` exceeds `previousHighestXpReached`. `metadata: { previousRankId, newRankId, trigger: "organic" | "admin-adjust" | "reconciliation" }`. Silent drift updates that re-enter a previously held rank do NOT produce a log entry (they are passive bookkeeping)

**IP and User-Agent:** Taken from the request via `req.ip` and `req.headers['user-agent']`. Added automatically via the post-response interceptor described below.

**Writing strategy — post-response interceptor (MVP):**

Activity log writes run **after the HTTP response has been sent**, never inside the hot path's business transaction. This keeps the user-facing latency untouched by the extra insert while still keeping the log in the same NestJS request context.

- Implementation: a `ActivityLogInterceptor` (NestJS) observes successful controller handlers that opt-in (decorator `@LogActivity('action-name')` or an ordered interceptor list). After the response stream completes it enqueues the log payload through the `ActivityLogService.log()` method, which writes to the `ActivityLog` table asynchronously. The write is fire-and-forget from the interceptor's perspective
- The service uses a short in-memory buffer flushed every 500 ms (or on graceful shutdown) to coalesce multiple inserts into a single `createMany`. This keeps per-action overhead low without needing BullMQ for the MVP
- If activity-log write volume grows beyond what the in-memory buffer can handle, the follow-up is to migrate behind a BullMQ queue (same pattern as the email / TOPIC_UPDATED fan-out queues); this upgrade is tracked in `docs/post-mvp.md` if and when it becomes necessary
- Failures in the activity-log write path are logged to Pino but never surfaced to the client — the user's action has already succeeded by the time the interceptor runs

**Actor-self notification exception clarification (cross-reference to E3 behaviour):** the "Self-notification prevention" rule in Section 3 step 2 already ensures the voter/commenter does not receive a notification about their own action — only the *counterpart* (topic author for TOPIC_VOTED / TOPIC_COMMENTED, comment author for COMMENT_LIKED / COMMENT_REPLIED) receives one. Both parties still earn XP. Authors who find the notifications noisy can disable `TOPIC_VOTED` / `TOPIC_COMMENTED` individually from the notification settings page (Phase 12), because the per-type toggle is "on" by default.

### 9.1. Data Retention Policy

**Activity logs — tiered retention:**
- **Regular activity logs** (profile update, login, topic/comment create/edit): **90 days** retention. Older records are automatically deleted.
- **Security & moderation logs** (restriction apply/remove, report create/review, block/unblock, account deletion, failed login attempts): **1 year** retention. Required for legal investigations and compliance (KVKK).
- **Authentication logs** (login, logout, token refresh, password change): **1 year** retention. Required for fraud/abuse investigation.

**Implementation:** A scheduled cron job (`@nestjs/schedule`) runs daily at 03:00 AM. **Batched delete pattern (authoritative):** every cleanup step below deletes rows in chunks of 5000 inside its own short transaction, looping until a pass affects zero rows. The chunked shape matches the `XpLog` archival job (Section 9.2 and Phase 17 Section 7.1) and keeps long locks off the hot tables even when the backlog reaches millions of rows. Example SQL for each pass: `DELETE FROM "ActivityLog" WHERE id IN (SELECT id FROM "ActivityLog" WHERE createdAt < :cutoff AND <category predicate> LIMIT 5000)`.

1. Delete regular activity logs older than 90 days — batched
2. Delete security/moderation logs older than 1 year — batched
3. Log the cumulative cleanup count for monitoring

The `action` field in the `ActivityLog` table is used to classify log type. Security/moderation actions: `restriction:*`, `report:*`, `block:*`, `account:delete`, `auth:login-failed`. Everything else is regular, including `user:rank-up`.

**Notification retention:**
- Notifications older than 90 days are automatically deleted by the same cron job, using the same 5000-row batched delete pattern (each pass wrapped in its own short transaction)
- This prevents the `Notification` table from growing unboundedly

**XpLog hot/cold archival** (daily at 04:15 AM UTC):
- Move rows from `XpLog` whose `createdAt < now() - interval '12 months'` into `XpLogArchive`
- Execute in batches of 5000 inside short transactions (`INSERT INTO XpLogArchive SELECT ... LIMIT 5000; DELETE FROM XpLog WHERE ...`) so the cron stays bounded even on large migrations
- Archive schema is identical to the hot schema so the move is a straight copy; no field mapping required
- Log the count of rows archived per run for observability

**XP reconciliation** (daily at 04:30 AM UTC):
- Compare `User.totalXp` with `SUM(XpLog.points) + SUM(XpLogArchive.points)` for all active users (the archive sum keeps reconciliation correct after archival runs)
- If a drift is detected (values don't match):
  1. Correct `User.totalXp` to match the actual sum (minimum 0)
  2. Run `checkAndUpdateRank()` for affected users
  3. Create an ActivityLog entry with action `xp:reconciliation`, metadata including `{ userId, previousXp, correctedXp, drift }`
- Log the total number of reconciled users for monitoring
- This catches any drift caused by partial transaction failures, bugs, or edge cases

### 10. Topic Detail Response Update

Add `isMuted` field to the `GET /topics/:id` response created in Phase 5:
- If authenticated → check if `(userId, topicId)` record exists in the `NotificationMute` table
- If exists → `isMuted: true`, if not → `isMuted: false`
- If not authenticated → field is not returned

### 11. Admin Activity Log Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /admin/activity-logs | Admin | Activity logs (filterable) |

Filtering: userId, action, targetType, date range. Pagination is mandatory (large dataset).

## Security Checklist
- [ ] User can only see their own notifications
- [ ] No self-notifications are sent
- [ ] No notifications from blocked users
- [ ] No notification is created when muted
- [ ] Anonymous actor info is hidden in notifications
- [ ] WebSocket connection is authenticated
- [ ] Activity logs can only be viewed by admin
- [ ] IP/User-Agent is logged but not shown to users

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# API tests:

# 1. Notification creation
# Vote on a topic → did the topic owner receive a notification?
GET /notifications (topic owner) → TOPIC_VOTED notification present

# 2. No self-notification
# Comment on your own topic → no notification

# 3. Mark as read
GET /notifications/unread-count → { count: 1 }
PATCH /notifications/:id/read → 200
GET /notifications/unread-count → { count: 0 }

# 4. Mark all as read
PATCH /notifications/read-all → 200

# 5. Muting
POST /topics/:id/mute → 201
# Post a comment on this topic (another user) → notification not created
DELETE /topics/:id/mute → 200
# Post a comment → notification created

# 6. Block + notification
# User A blocks User B
# User B votes on User A's topic → no notification

# 7. WebSocket
# Connect to user room with Socket.io client
# Another user votes → did notification:new event arrive?

# 8. Activity log
GET /admin/activity-logs (admin) → log records
GET /admin/activity-logs (normal user) → 403
# Create topic → is there a record in the activity log?
```

## Completion Criteria
- [ ] All notification types are created (vote, comment, like, reply, mention, topic-updated, rank-up)
- [ ] Notification listing and mark-as-read works
- [ ] Topic-based muting works
- [ ] Real-time notification via WebSocket works
- [ ] No self-notifications or notifications from blocked users
- [ ] Anonymous notifications display correctly
- [ ] `TOPIC_UPDATED` fan-out reaches voters and commenters (minus author, muted users, blocked) and deduplicates unread repeats
- [ ] Activity logging works on all actions
- [ ] Admin log endpoint works
- [ ] Data retention cron job is configured and tested
- [ ] All tests pass
