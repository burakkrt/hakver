# Phase 1: Project Foundation & Infrastructure

## Dependencies
None. This is the first phase.

## Goals
- Set up the Turborepo monorepo structure
- Set up Next.js, NestJS, and shared package scaffolding
- Configure TypeScript, ESLint, and Prettier
- Create `.env` files using values from `docs/environments.md`
- Prepare the local development environment with Docker Compose
- Set up Git hooks (husky + lint-staged)
- Create a basic CI pipeline

## Tasks

### 1. Monorepo Root Structure

Create the following files in the project root directory:

**`package.json`** — Monorepo root package. `private: true`, pnpm workspace support. Scripts: `dev`, `build`, `lint`, `test`, `format`.

**`pnpm-workspace.yaml`** — Workspace definition:
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

**`turbo.json`** — Pipeline definitions: `build` (dependency-ordered), `dev` (persistent), `lint`, `test`. Define environment variables in `globalEnv` and `env` fields.

**`.gitignore`** — Node.js, Next.js, NestJS, Prisma, IDE, OS files. Including `.env` and `docs/environments.md`.

**`.nvmrc`** — Node.js LTS version pinning (22.x).

### 2. TypeScript Configuration

**`tsconfig.json`** (root) — Base config. `strict: true`, `esModuleInterop: true`, `skipLibCheck: true`, `forceConsistentCasingInFileNames: true`.

Each app and package extends the root config in its own `tsconfig.json` file.

### 3. ESLint & Prettier

**`eslint.config.js`** (root) — ESLint flat config format. TypeScript + Prettier integration. Each app extends its own ESLint config.

**`.prettierrc`** — `semi: true`, `singleQuote: true`, `trailingComma: "all"`, `tabWidth: 2`, `printWidth: 100`.

**`.prettierignore`** — `node_modules`, `dist`, `.next`, `coverage`, `prisma/migrations`.

### 4. NestJS Backend (apps/api)

Create with NestJS CLI (`@nestjs/cli`). Structure:

```
apps/api/
├── src/
│   ├── app.module.ts
│   ├── main.ts
│   ├── common/
│   │   ├── decorators/
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts
│   │   ├── guards/
│   │   ├── interceptors/
│   │   │   └── logging.interceptor.ts
│   │   ├── pipes/
│   │   │   └── zod-validation.pipe.ts
│   │   └── constants/
│   ├── config/
│   │   └── configuration.ts
│   └── modules/
│       └── health/
│           ├── health.controller.ts
│           └── health.module.ts
├── test/
├── .env                          # Backend variables from docs/environments.md
├── tsconfig.json
├── tsconfig.build.json
├── nest-cli.json
└── package.json
```

**`main.ts`** configuration:
- CORS: origin from `[FRONTEND_URL, ADMIN_URL]` environment variables (array — allows both frontend and admin panel origins)
- Global prefix: `api/v1`
- Global validation pipe (Zod-based)
- Global exception filter
- Swagger setup (`@nestjs/swagger`)
- Port: from `PORT` environment variable
- `nestjs-pino` logger integration

**`configuration.ts`** — Centrally manage all environment variables with `@nestjs/config`. Apply runtime validation with Zod — throw an error before the application starts if an env variable is missing or invalid. Must validate `ADMIN_URL` alongside other required variables.

**Health endpoint** — `GET /api/v1/health` → `{ status: "ok", timestamp }`. For deployment verification.

### 5. Next.js Frontend (apps/web)

Create with Next.js App Router. Structure:

```
apps/web/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   └── ui/                   # shadcn/ui (to be added later)
│   ├── hooks/
│   ├── lib/
│   │   └── utils.ts
│   ├── providers/
│   ├── stores/
│   └── types/
├── public/
├── .env                          # Frontend variables from docs/environments.md
├── next.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── tsconfig.json
└── package.json
```

**`next.config.ts`** — Strict mode, Turborepo transpile packages configuration (`@hakver/shared`).

**`tailwind.config.ts`** — Content paths: `./src/**/*.{ts,tsx}`, shared package content.

### 6. Shared Package (packages/shared)

```
packages/shared/
├── src/
│   ├── schemas/                  # Zod validation schemas
│   ├── types/                    # TypeScript type definitions
│   ├── constants/                # Enums, constant values, error codes
│   │   ├── enums.ts
│   │   └── error-codes.ts
│   └── index.ts                  # Barrel export
├── tsconfig.json
└── package.json
```

**`package.json`** — `name: "@hakver/shared"`, `main: "./src/index.ts"`. Dependency: `zod`.

**`enums.ts`** — Enums matching the Prisma schema exactly: `Gender`, `AuthProvider`, `VoteType`, `ReportTargetType`, `ReportStatus`, `NotificationType`, `ReferenceType` (TOPIC, COMMENT, VOTE, COMMENT_LIKE, USER). Every enum defined in the Prisma schema must also be defined here.

**`error-codes.ts`** — Standard error codes. These codes are used in backend responses across all phases:
- `AUTH_*`: INVALID_CREDENTIALS, EMAIL_NOT_VERIFIED, TOKEN_EXPIRED, OAUTH_EMAIL_CONFLICT
- `VALIDATION_*`: INVALID_INPUT, DUPLICATE_USERNAME, DUPLICATE_EMAIL
- `TOPIC_*`: NOT_FOUND, EDIT_EXPIRED, PREREQUISITE_NOT_MET, UPDATE_NOTE_NOT_FOUND
- `VOTE_*`: ALREADY_VOTED, SELF_VOTE, NOT_FOUND
- `COMMENT_*`: EDIT_COOLDOWN, ANONYMOUS_LOCKED, NOT_AUTHOR_TOPIC
- `USER_*`: NOT_FOUND, RESTRICTED, BLOCKED, CHANGE_COOLDOWN, PROFILE_INCOMPLETE, ACCOUNT_DELETED, ACCOUNT_UNAVAILABLE
- `REPORT_*`: ALREADY_REPORTED, SELF_REPORT
- `MODERATION_*`: REASON_REQUIRED (moderator/admin action missing the mandatory reason field), ROLE_CHANGE_INVALID

**`websocket-events.ts`** — WebSocket event names and payload types:
- `vote:updated` → `{ topicId: string, voteCountRight: number, voteCountWrong: number }`
- `comment:created` → `{ topicId: string, comment: CommentResponse }`
- `comment:deleted` → `{ topicId: string, commentId: string }`
- `notification:new` → `NotificationResponse`
- `notification:unread-count` → `{ count: number }`

**`api-response.ts`** — Standard API response and error formats:
- `ApiErrorResponse`: `{ statusCode: number, code: string, message: string, errors?: { field: string, message: string }[] }`
- `PaginatedResponse<T>`: `{ data: T[], meta: { total: number, page: number, limit: number, totalPages: number, hasNext: boolean } }`
- These types are used by all API endpoints for consistent responses.

### 7. Environment Variables

Create `.env.example` files with placeholder values in the repository. Actual values come from `docs/environments.md`:
- `apps/api/.env.example` — All backend variables with placeholder values
- `apps/web/.env.example` — All frontend variables with placeholder values

`.env.example` files are committed to the repo with placeholder values. `docs/environments.md` and `.env` files must be in `.gitignore`.

**Actual `.env` files** must be created during initial setup using the values from `docs/environments.md`. Each developer creates their own `.env` files by copying `.env.example` and filling in the real values from `docs/environments.md`.

### 8. Redis Cache Module

Set up `nestjs/cache-manager` + `cache-manager-ioredis-yet` in `apps/api`:
- Redis connection with `CacheModule.registerAsync()` (`REDIS_URL`)
- Register globally
- Default TTL: 5 minutes
- Use `@CacheTTL()` decorator for infrequently changing data (category list, avatar list, rank list, etc. — to be applied in relevant phases)

### 9. Pino Logger Configuration

Detailed `nestjs-pino` configuration:
- Development: human-readable log output with `pino-pretty`
- Production: structured log in JSON format
- Log levels: `error`, `warn`, `info`, `debug` (debug in development, info in production)
- Request/response logging: every request is automatically logged (excluding body — sensitive data protection)
- Fields like password and token are redacted from logs

### 10. Docker Compose (Optional Local Development)

`docker-compose.yml` (root) — For local development only. Redis container (can be used for local testing instead of Upstash). Backend and frontend are not Docker-dependent, they run directly.

### 11. Git Hooks

**Husky + lint-staged** setup:
- Pre-commit: lint-staged (ESLint + Prettier on relevant files)
- Commit-msg: commitlint (conventional commits format)

### 12. GitHub Actions CI

`.github/workflows/ci.yml`:
- Trigger: push and PR (all branches)
- Steps: checkout, pnpm setup, install, lint, type-check, test
- Matrix: Node.js 22 LTS

## Security Checklist
- [ ] `.env` files are in `.gitignore`
- [ ] `docs/environments.md` and all `.env` files are in `.gitignore` — only `.env.example` files with placeholder values are committed
- [ ] `configuration.ts` prevents application startup on missing env variables
- [ ] CORS only allows `FRONTEND_URL` and `ADMIN_URL`
- [ ] Swagger is only active in development mode

## Test Plan

```bash
# 1. Are dependencies installed?
pnpm install

# 2. Does lint pass?
pnpm lint

# 3. Does TypeScript compile?
pnpm build

# 4. Does the backend start?
cd apps/api && pnpm start:dev
# GET http://localhost:3001/api/v1/health → { status: "ok" }

# 5. Does the frontend start?
cd apps/web && pnpm dev
# http://localhost:3000 → Page renders

# 6. Can the shared package be imported?
# @hakver/shared import works from both apps/api and apps/web
```

## Completion Criteria
- [ ] `pnpm install` completes without errors
- [ ] `pnpm build` succeeds in all workspaces
- [ ] `pnpm lint` passes without errors
- [ ] Backend health endpoint returns `200 OK`
- [ ] Frontend renders on localhost:3000
- [ ] `@hakver/shared` can be imported from both apps
- [ ] `.env` files are in `.gitignore` and not committed to the repo
- [ ] Environment variables are validated at runtime
