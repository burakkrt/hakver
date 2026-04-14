---
paths:
  - "packages/shared/**"
---

# Shared Package Rules

## Allowed
- Validation schemas (request/response DTOs)
- TypeScript types
- Enums and constants
- WebSocket event names and payload types
- Error codes

## Not Allowed
- Business logic
- DB queries or ORM types
- React components or hooks
- Functions with side effects
- Platform-specific code (Node.js or browser)
