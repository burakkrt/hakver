# Security Rules

- Auth guard on all protected endpoints
- Email verification required for write operations
- Permission guard for admin/moderator actions
- Restriction check for user-restrictable actions
- Rate limiting on auth, registration, email, and sensitive endpoints
- Strip HTML tags from all user text inputs (mandatory on backend, additional layer on frontend)
- All input validated with validation schema — both client and server
- No sensitive data in any response (password hash, tokens, other users' personal info)
- Anonymous content: user identity never exposed in API responses
- CORS restricted to frontend URL only
