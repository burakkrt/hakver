# Coding Standards

## Framework Conventions
- Follow official conventions and best practices of the frameworks used (NestJS, Next.js, Prisma, etc.) when creating new structures
- Framework documentation takes precedence, then project-internal consistency

## Naming
- Components: PascalCase
- Hooks/utilities/modules/services: kebab-case
- Service methods: camelCase
- API paths: kebab-case
- All names (variables, functions, files, folders) should be descriptive, meaningful, and aligned with industry standards
- TypeScript strict mode — no `any`, no type assertions without justification

## Shared Package
- All request/response DTOs defined in `@hakver/shared` and imported from there by both frontend and backend
- Schema additions, modifications, and deletions should be maintained in a shared context across FE and BE — never duplicate

## ORM Type Isolation
- ORM (Prisma) types never leak to frontend — convert to shared DTO schemas at the service layer
- Every list / detail endpoint must declare an explicit Prisma `select` (or tightly scoped `include`) object; implicit "return all columns" queries are rejected in code review to keep payloads lean and prevent accidental PII leaks. Reusable selects live under `apps/api/src/common/prisma-selects/` (detailed guidance in `.claude/rules/backend.md`)

## API Response Standard
- All API responses should be consistent, predictable, and follow REST standards. Error responses should include error codes from the shared package and Turkish user-facing messages. Pagination meta should be considered for list endpoints.

## Code Quality
- No dead code or unused imports
- Consistent error handling — no silent catches
- N+1 query prevention
- Cache for frequently accessed static data
- Files should not be excessively long — split responsibilities into components
- Write sustainable, readable, and performant code
- Pagination on every list endpoint
- Denormalized counters updated inside transactions
