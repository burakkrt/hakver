# Phase 6: Voting System

## Dependencies
- Phase 5 must be completed (topic module)

## Goals
- Right/Wrong voting
- Vote changing and withdrawal
- Block voting on own topic
- Update denormalized vote counts
- WebSocket infrastructure for real-time vote updates
- Define vote DTO schemas in the shared package

## Tasks

### 1. Shared Zod Schemas

`packages/shared/src/schemas/vote.ts`:

**VoteResponseSchema** (response DTO):
- vote: `{ id, type: "RIGHT" | "WRONG", createdAt }`
- topic: `{ voteCountRight: number, voteCountWrong: number }`

**MyVoteResponseSchema** (for `GET /topics/:topicId/votes/me`):
- type: `"RIGHT" | "WRONG" | null`

**CreateVoteSchema:**
- topicId: UUID
- type: "RIGHT" | "WRONG"

**UpdateVoteSchema:**
- type: "RIGHT" | "WRONG"

### 2. Vote Module

Under `apps/api/src/modules/vote/`:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /topics/:topicId/votes | JWT + Verified | Cast vote |
| PATCH | /topics/:topicId/votes | JWT + Verified | Change vote |
| DELETE | /topics/:topicId/votes | JWT + Verified | Withdraw vote |
| GET | /topics/:topicId/votes/me | JWT | My vote status |

### 3. Casting a Vote

1. Validation with `CreateVoteSchema`
2. **Own topic check:** `topic.authorId === currentUser.id` → 403 "Kendi konunuza oy veremezsiniz"
3. **Duplicate vote check:** `(userId, topicId)` unique constraint in `Vote` table. If vote already exists → 409 "Bu konuya zaten oy verdiniz"
4. **Profile completion check:** `ProfileCompleteGuard` ensures the voter's profile is complete (guard chain order: JWT → Verified → ProfileComplete → Restriction → Permission)
5. **Restriction check:** check for active restriction on `vote:create` action
5. Create vote record
6. Update denormalized counter on Topic:
   - `type === RIGHT` → `voteCountRight += 1`
   - `type === WRONG` → `voteCountWrong += 1`
7. **Emit WebSocket event:** `vote:updated` → `{ topicId, voteCountRight, voteCountWrong }`
8. Response: `{ vote, topic: { voteCountRight, voteCountWrong } }`

**Denormalized counter update** must be done inside a Prisma `$transaction` — vote creation and counter update must be atomic.

### 4. Changing a Vote

1. Check if existing vote exists. If not → 404
2. If already the same type → 400 "Oyunuz zaten bu yönde"
3. Inside transaction:
   - Decrement old counter, increment new counter
   - Update vote type
4. Emit WebSocket event
5. **XP:** Vote changing does not earn additional XP (to be implemented in Phase 8)

### 5. Withdrawing a Vote

1. Check if existing vote exists. If not → 404
2. Inside transaction:
   - Decrement the relevant counter
   - Delete vote record (hard delete — plan.md rule 5 exception: a vote is a preference record, not content)
3. Emit WebSocket event
4. **XP:** On withdrawal, earned XP is revoked (to be implemented in Phase 8)

### 6. Own Vote Status

`GET /topics/:topicId/votes/me`:
- If authenticated → user's vote on this topic (`RIGHT`, `WRONG`, or `null`)
- If not authenticated → 401

### 7. WebSocket Gateway

Under `apps/api/src/websocket/`:

**`events.gateway.ts`** — Socket.io gateway:
- Namespace: `/` (default)
- Room structure: `topic:{topicId}` room for each topic
- Client joins room with `joinTopic` event when entering topic page
- Client leaves room with `leaveTopic` event when leaving page

**WebSocket JWT Verification:**
- Client sends `auth: { token: accessToken }` when establishing connection
- JWT is verified during `handleConnection` in the gateway (`JwtService.verify`)
- Invalid or expired token → connection rejected (`socket.disconnect()`)
- When token is refreshed, client must `reconnect` (to be handled in frontend)

**Events:**
- Server → Client: `vote:updated` → `{ topicId, voteCountRight, voteCountWrong }`
- Event names and payload types are imported from `@hakver/shared/websocket-events.ts`

**Redis Adapter:**
- Install `@socket.io/redis-adapter`
- Configure with `REDIS_URL` (Upstash Redis)
- Required for production scaling — without Redis adapter, WebSocket events are not shared across multiple backend instances
- Setup in the gateway module initialization

Only the vote event is set up in this phase. Other events (comment, notification) will be added in subsequent phases.

### 8. Block Filter

- If the voter has blocked the topic author → voting is blocked (topic returns 404 due to full blocking policy)
- If the topic author has blocked the voter → voting is blocked (topic returns 404 due to full blocking policy)
- Full blocking means all interactions are prevented: the blocked user's content is completely invisible and inaccessible

## Security Checklist
- [ ] User cannot vote on their own topic
- [ ] Cannot cast multiple votes on the same topic (unique constraint)
- [ ] Counter update is atomic inside a transaction
- [ ] Rate limiting is active (vote spam prevention)
- [ ] WebSocket connection is authenticated (with JWT token)
- [ ] Restriction check is active

## Test Plan

```bash
# Unit tests
cd apps/api && pnpm test

# API tests:

# 1. Casting a vote
POST /topics/:id/votes { type: "RIGHT" } → 201
POST /topics/:id/votes (same topic, same user) → 409
POST /topics/:id/votes (own topic) → 403
POST /topics/:id/votes (unverified user) → 403

# 2. Changing a vote
PATCH /topics/:id/votes { type: "WRONG" } → 200
PATCH /topics/:id/votes { type: "WRONG" } (already WRONG) → 400

# 3. Withdrawing a vote
DELETE /topics/:id/votes → 200
DELETE /topics/:id/votes (no vote) → 404

# 4. Counter verification
# Cast vote → has topic.voteCountRight increased?
# Withdraw vote → has topic.voteCountRight decreased?
# Change vote → has old counter decreased and new counter increased?

# 5. Own vote status
GET /topics/:id/votes/me → { type: "RIGHT" } or { type: null }

# 6. WebSocket
# Connect to topic room with Socket.io client
# Cast vote → did vote:updated event arrive?
# Are the numbers in the event correct?

# 7. Concurrency test
# 10 users vote simultaneously → are counters correct? (no race condition?)
```

## Completion Criteria
- [ ] Right/Wrong voting works
- [ ] Vote changing works
- [ ] Vote withdrawal works
- [ ] Block on voting own topic works
- [ ] Denormalized counters update correctly
- [ ] Atomic update with transaction is working
- [ ] Real-time vote update via WebSocket works
- [ ] Race condition tests pass
- [ ] All tests pass
