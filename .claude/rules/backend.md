---
paths:
  - "apps/api/**"
---

# Backend Rules

- Environment variables must be validated at startup — app should not start if env is missing
- Database operations go through the ORM service. Raw queries are allowed only for scenarios the ORM does not support (row-level locking, extension management, full-text/trigram index creation); when used, document the reason in the migration or service file
- User-facing error messages in Turkish, seed data in Turkish
