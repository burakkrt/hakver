# Coding Standards

## Naming
- Components: PascalCase
- Hooks/utilities/modules/services: kebab-case
- Service methods: camelCase
- API paths: kebab-case
- TypeScript strict mode — no `any`, no type assertions without justification

## Code Quality
- No dead code or unused imports
- Consistent error handling — no silent catches
- N+1 query prevention
- Cache for frequently accessed static data
- Files should not be excessively long — split responsibilities into components
- Write sustainable, readable, and performant code
- Pagination on every list endpoint
- Denormalized counters updated inside transactions
