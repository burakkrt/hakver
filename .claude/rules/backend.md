---
paths:
  - "apps/api/**"
---

# Backend Rules

- Environment variables must be validated at startup — app should not start if env is missing
- All database operations through ORM service, no raw SQL
- User-facing error messages in Turkish, seed data in Turkish
