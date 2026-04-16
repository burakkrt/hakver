# Security Rules

## Authentication & Authorization
- Auth guard on all protected endpoints
- Email verification required for write operations — unverified accounts cannot create topics, comment, or vote
- Profile completion required for write operations — accounts missing username, firstName, or lastName are blocked and redirected to profile completion
- Permission guard for admin/moderator actions
- Restriction check for user-restrictable actions
- Guard order on write endpoints: JWT → Verified → ProfileComplete → Restriction → Permission

## Rate Limiting
- Rate limiting on auth, registration, email, file upload, and other sensitive endpoints
- Rate-limit hits must return 429 with a Turkish user-facing message and a retry indicator; never silently drop requests

## Input Validation & Sanitization
- All input validated with a validation schema — both client and server
- Strip HTML tags from all user text inputs (mandatory on backend, additional layer on frontend)
- File uploads validated by actual content, not only the declared MIME type — verify magic bytes (file signature) before forwarding to external storage. Enforce size and type limits on the backend

## Response Hygiene
- No sensitive data in any response (password hash, tokens, other users' personal info, IP addresses, user-agent strings)
- Anonymous content: the real user identity is never exposed in API responses except to users with the view-anonymous permission through a dedicated moderator field

## Identity Display Rules
- Username is the primary public identifier across the platform — topic cards, comment cards, notification actors, and vote listings show only username
- Full name and surname are never shown outside the profile page. On the profile page, the viewer sees the given name in full and the family name abbreviated to a single initial with a period (e.g., "Ahmet D.")
- Own profile view exposes the owner's full name to the owner
- Admin dashboard may display the full legal name to authorized roles for moderation and compliance purposes
- Name masking is enforced in the service/DTO layer so that no endpoint, bug path, or error response can leak the full family name to other users

## XP & Concurrency
- XP mutations must be executed inside a database transaction with row-level protection against concurrent awards; a single action must not grant duplicate XP under race conditions

## Admin Action Accountability
- Admin and moderator actions that alter user state (restriction apply/remove, XP adjustment, role change, content removal) require a mandatory reason field recorded in the activity log

## Cross-Origin Access
- CORS is restricted to the platform's declared allowed origins (main web app, admin app, and any other configured origins); wildcard origins are forbidden

## Browser Security Policies
- Responses set security headers including a Content-Security-Policy that restricts script, style, image, and connect sources to trusted origins; X-Content-Type-Options, Referrer-Policy, and a strict transport policy in production
- User content rendered in the browser must never be rendered as raw HTML; always use the framework's safe text mechanism
