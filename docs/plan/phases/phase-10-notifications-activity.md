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

**NotificationResponseSchema** (response DTO):
- id: UUID
- type: NotificationType enum
- isRead: boolean
- createdAt: ISO date string
- actor: `UserPublicCardSchema | AnonymousCardSchema` (card shape — notification actors are never rendered with firstName/lastName)
- reference: `{ type: ReferenceType, id: UUID, title?: string }`
- message: string (Turkish, user-facing — generated from type + actor)

### 2. API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /notifications | JWT | Notification list (paginated) |
| GET | /notifications/unread-count | JWT | Unread notification count |
| PATCH | /notifications/:id/read | JWT | Mark notification as read |
| PATCH | /notifications/read-all | JWT | Mark all as read |
| POST | /topics/:topicId/mute | JWT | Mute topic notifications |
| DELETE | /topics/:topicId/mute | JWT | Unmute topic notifications |

### 3. Notification Creation Service

**`notification.service.ts`** — `createNotification(data)`:

1. **Mute check:** Does a recipient-topic pair exist in the `NotificationMute` table? If yes → do not create notification
2. **Self-notification prevention:** `actorId === userId` → do not create notification (e.g., liking own comment)
3. **Block check:** Has the recipient blocked the actor? → do not create notification
4. **Anonymous actor:** Notification is created but actor info is shown with an anonymous card
5. Create `Notification` record
6. Send in real-time via WebSocket

### 4. Notification Triggers

Add notification hooks to existing modules:

**Vote Module:**
- When a vote is cast → `TOPIC_VOTED` notification to topic owner
- On vote change → no new notification
- On vote withdrawal → no new notification

**Comment Module:**
- When a comment is posted → `TOPIC_COMMENTED` notification to topic owner
- When a reply is posted → `COMMENT_REPLIED` notification to parent comment owner
- When a like is given → `COMMENT_LIKED` notification to comment owner
- When a user is @mentioned in a comment → `MENTIONED` notification to the mentioned user. Reference: the comment containing the mention. Actor: the comment author (or anonymous card if anonymous comment).

**Topic Module:**
- When the author writes or replaces an update note (`PATCH /topics/:id/update-note`) → `TOPIC_UPDATED` notification fanned out to users who voted on the topic and users who commented on the topic, minus the author themselves, minus users who muted the topic, minus users who have a mutual block with the author. The fan-out runs asynchronously via a BullMQ job to keep the request fast and avoid blocking the author's response. Clearing the update note does not trigger a notification.
- Deduplication: if the author replaces the note while a previous `TOPIC_UPDATED` notification from the same topic is still unread by the recipient, update the existing notification's `createdAt` and keep it unread instead of creating a duplicate record. This prevents notification spam when the author iterates on the text.

### 5. Notification Listing

`GET /notifications?page=1&limit=20`:

```json
{
  "data": [
    {
      "id": "...",
      "type": "TOPIC_COMMENTED",
      "isRead": false,
      "createdAt": "...",
      "actor": {
        "username": "testuser",
        "avatarUrl": "...",
        "rank": "Aktif Üye"
      },
      "reference": {
        "type": "TOPIC",
        "id": "...",
        "title": "Konu başlığı..."
      },
      "message": "testuser konunuza yorum yaptı"
    }
  ],
  "meta": { "total": 50, "page": 1, "limit": 20 }
}
```

**Anonymous actor** case:
```json
"actor": {
  "displayName": "Anonim Penguen #4521",
  "iconType": "penguin"
}
```

**Notification messages** (in Turkish — user-facing):
- `TOPIC_VOTED`: "{actor} konunuza oy verdi"
- `TOPIC_COMMENTED`: "{actor} konunuza yorum yaptı"
- `COMMENT_LIKED`: "{actor} yorumunuzu beğendi"
- `COMMENT_REPLIED`: "{actor} yorumunuza yanıt verdi"
- `MENTIONED`: "{actor} sizi bir yorumda etiketledi"
- `TOPIC_UPDATED`: "{actor} ilgilendiğiniz konuya bir güncelleme ekledi"

### 6. WebSocket Notification Delivery

In addition to the gateway set up in Phase 6:

**User-based room:** Each authenticated user automatically joins the `user:{userId}` room (on connection establishment).

**Events:**
- Server → Client: `notification:new` → notification object
- Server → Client: `notification:unread-count` → `{ count: number }`

### 7. Topic-Based Muting

`POST /topics/:topicId/mute`:
1. Create `NotificationMute` record (`userId, topicId`)
2. Unique constraint → if already muted → 409

`DELETE /topics/:topicId/mute`:
1. Delete record
2. If not found → 404

Mute status is returned as a flag in the topic detail response:
```json
{
  "id": "...",
  "title": "...",
  "isMuted": true  // if authenticated and mute record exists
}
```

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

**IP and User-Agent:** Taken from the request via `req.ip` and `req.headers['user-agent']`. Can be automatically added via NestJS interceptor.

### 9.1. Data Retention Policy

**Activity logs — tiered retention:**
- **Regular activity logs** (profile update, login, topic/comment create/edit): **90 days** retention. Older records are automatically deleted.
- **Security & moderation logs** (restriction apply/remove, report create/review, block/unblock, account deletion, failed login attempts): **1 year** retention. Required for legal investigations and compliance (KVKK).
- **Authentication logs** (login, logout, token refresh, password change): **1 year** retention. Required for fraud/abuse investigation.

**Implementation:** A scheduled cron job (`@nestjs/schedule`) runs daily at 03:00 AM:
1. Delete regular activity logs older than 90 days
2. Delete security/moderation logs older than 1 year
3. Log the cleanup count for monitoring

The `action` field in the `ActivityLog` table is used to classify log type. Security/moderation actions: `restriction:*`, `report:*`, `block:*`, `account:delete`, `auth:login-failed`. Everything else is regular.

**Notification retention:**
- Notifications older than 90 days are automatically deleted by the same cron job
- This prevents the `Notification` table from growing unboundedly

**XP reconciliation** (daily at 04:30 AM UTC):
- Compare `User.totalXp` with `SUM(XpLog.points)` for all active users
- If a drift is detected (values don't match):
  1. Correct `User.totalXp` to match the actual `SUM(XpLog.points)` (minimum 0)
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
- [ ] All notification types are created (vote, comment, like, reply, mention, topic-updated)
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
