---
paths:
  - "apps/api/**"
---

# Backend Rules

- Defensive code: no silent catches, every error with a meaningful message
- Modular structure, separation of concerns
- Environment variables must be validated at startup — app should not start if env is missing
- All database operations through ORM service, no raw SQL
- ORM types never leak to frontend — convert to shared DTO schemas
- User-facing error messages in Turkish, seed data in Turkish
