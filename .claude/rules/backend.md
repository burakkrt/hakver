---
paths:
  - "apps/api/**"
---

# Backend Rules

## Core
- Environment variables must be validated at startup — app should not start if env is missing
- Database operations go through the ORM service. Raw queries are allowed only for scenarios the ORM does not support (row-level locking, extension management, full-text/trigram index creation); when used, document the reason in the migration or service file
- User-facing error messages in Turkish, seed data in Turkish

## Prisma Query Shape
- Every list / detail / card endpoint must declare an explicit `select` (or tightly scoped `include`) object. Implicit "return all scalar columns" reads are rejected in code review because they inflate payload, widen SQL, and accidentally surface columns like `email` or `passwordHash`
- Colocate reusable selects under `apps/api/src/common/prisma-selects/` (shared package cannot import Prisma). Each module owns a named select object per response shape (e.g., `topicCardSelect`, `topicDetailSelect`, `commentListSelect`) and imports it instead of redeclaring the same shape across services
- Prisma types must never leak to the frontend. The service layer maps the query result into the shared response DTO (`@hakver/shared`) before returning — no `Prisma.TopicGetPayload<...>` ever appears in controller return types

## Rate Limit Response Format
- When implementing a new rate-limited endpoint, follow the formatted Turkish 429 response template defined in `.claude/rules/security.md` → "Rate Limiting" section. The message must use the largest non-zero unit ("1 saat 15 dakika", "35 dakika", "45 saniye") and never render a zero-valued unit. The response envelope carries `code: "RATE_LIMIT_EXCEEDED"`, `message`, and `retryAfter` (seconds). Do not invent per-endpoint message formats

## HTTP Method Semantics (REST conventions)

Every new endpoint must pick the HTTP method that matches the operation's meaning. Choose based on the action's effect, not by habit:

- **GET** — Safe, idempotent read. No side effects. Used for resource fetch, list, detail, existence check. Never used to mutate state
- **POST** — Create a new resource or trigger a non-idempotent action that has no natural PUT/PATCH/DELETE semantics (e.g., initiate password reset, send verification email, request data export). Request body carries the input; response returns the created resource or acknowledgment
- **PATCH** — Partial update of an existing resource. Only the fields being changed are sent in the request body. Replaces/sets values; does not remove them via `null`. Never used to create or to clear an entire related resource
- **PUT** — Full replacement of an existing resource. Used rarely in this project; prefer PATCH for partial updates
- **DELETE** — Remove a resource, clear an optional field that has a dedicated lifecycle (e.g., update note, bookmark, vote withdrawal), or perform a soft delete. No request body beyond URL params
- **Idempotency** — GET, PUT, DELETE must be idempotent (repeated calls produce the same effect). POST and PATCH are not required to be idempotent but should avoid unintended duplication when the same payload arrives twice

### Resource and clear operations

When a field on a resource has its own lifecycle and can be explicitly removed (for example a topic's `updateNote`, a user's avatar override, a comment highlight), expose a dedicated `DELETE <resource>/<field>` endpoint for the clear action. Do not overload PATCH to accept `null` for clearing — this keeps the request schemas strict (`min 1`-style validation stays meaningful) and makes the API self-documenting.

### URL path conventions
- Paths in kebab-case (e.g., `/topics/:id/update-note`, `/users/me/data-export`)
- Plural nouns for collections (`/topics`, `/comments`), singular for sub-resources bound to one parent (`/topics/:id/cover`)
- Nested resources when the child only exists inside a parent (`/topics/:topicId/comments`). Flat paths when the child is addressable on its own (`/comments/:id`)

### Response codes
- `200` on successful GET / PATCH / DELETE with response body
- `201` on successful POST that creates a resource
- `202` on accepted but async operations (e.g., data export queued)
- `204` on successful DELETE / action without response body
- Error codes follow the error-codes.ts shared constants (see Phase 1) with Turkish `message` field

### Writing new endpoints

When proposing a new endpoint:
1. Identify the operation's effect (read / create / partial update / replace / remove)
2. Pick the method from the table above; do not invent composite methods
3. If the operation is "remove just this field", use DELETE on the sub-path, not PATCH with `null`
4. Confirm idempotency requirements for GET/PUT/DELETE
5. Match the path to the URL conventions above
