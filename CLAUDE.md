# CLAUDE.md — Project Configuration

> **[CUSTOMIZE]** Replace placeholder commands, paths, and domain context with your project's specifics before use.

<!-- PROJECT CONTEXT — highest-value section, customize first -->
## Project
- **What**: [CUSTOMIZE] One-line description of what the app does and who uses it
- **Stack**: Node.js/TypeScript monorepo — React frontend + Express API
- **Database**: PostgreSQL (external — RDS/Supabase), Redis for caching/queues
- **ORM**: Prisma · **Auth**: JWT + refresh tokens · **Realtime**: Socket.IO
- **Deploy**: [CUSTOMIZE] e.g., Railway (API), Vercel (frontend), CI/CD via GitHub Actions

---

## Commands
```bash
# [CUSTOMIZE] Replace with your actual scripts. Run from monorepo root or note the directory.
npm run dev              # Start dev server
npm run build            # Production build
npm run lint             # ESLint
npm run typecheck        # TypeScript check (tsc --noEmit)
npm run test             # Run tests
npm run db:migrate       # Prisma migrations
npm run db:generate      # Regenerate Prisma client after schema changes
```
> **Verify before done**: `npm run lint && npm run typecheck && npm run test`

---

## Architecture
```
/frontend/               # React app (Vite)
/backend/
  /src/
    /controllers/        # Route handlers — thin, delegate to services
    /services/           # Business logic — all domain logic lives here
    /routes/             # Express route definitions
    /middleware/          # Auth, rate limiting, error handling
    /lib/                # Shared utilities — CHECK HERE before writing new ones
    /types/              # TypeScript type definitions
    /workers/            # Background jobs (BullMQ)
/prisma/                 # Schema + migrations (source of truth)
/docs/                   # Reference docs (load with @docs/filename.md)
```
<!-- [CUSTOMIZE] Update the tree above to match your actual project structure -->

---

## Workflow

### Every Task
1. **Plan first** — Use Plan Mode for tasks with 3+ steps or architectural decisions. Stop and re-plan if approach isn't working.
2. **Check in** — Confirm approach with user before starting complex work
3. **Verify** — Run lint + typecheck + tests before declaring done
4. **Use subagents** for research and exploration — keep main context clean

### Code Reviews
- For every issue: describe concretely with file references, present 2–3 options with tradeoffs, give opinionated recommendation
- Pause after each major section for feedback — don't dump everything at once
- Full review framework: `@docs/plan-mode-prompt.md`

---

## Code Standards

### TypeScript
- No `any` types — use `unknown` and narrow, or define proper types
- `type` over `interface` unless extending
- Explicit error types — no bare `try/catch` swallowing errors in business logic
- Check `/src/lib/` for existing utilities before writing new ones

### Functions
- Single responsibility, under 40 lines — extract if longer
- Pure functions preferred; isolate side effects to service boundaries

### API Conventions
- Routes versioned: `/api/v1/resource`
- Controller → Service → Prisma (3-layer pattern — never put business logic in controllers)
- Response shapes:
  ```typescript
  { success: true, data: T }
  { success: false, error: { code: string, message: string } }
  ```
- Never expose Prisma models directly — map to response DTOs
- Status codes: 400 validation · 401 auth · 403 permissions · 404 not found · 500 server

### Logging
- Use the project logger (`/src/lib/logger.ts`) — never `console.log` in production code
- Include context: `logger.error('operation failed', { error, userId, requestId })`
- Never log passwords, tokens, full request bodies, or PII

---

## Database (PostgreSQL + Prisma)
- Schema is source of truth: `/prisma/schema.prisma`
- Run `npx prisma generate` after every schema change
- Use `prisma.$transaction()` for multi-step atomic writes
- Use `select` to limit fields — never fetch full records unnecessarily
- Use `upsert` in seed scripts for idempotency
- NEVER run `deleteMany({})` without a `where` clause
- NEVER use `prisma.$queryRaw` without parameterized inputs
- NEVER run `migrate reset` outside `development`
- Rollback procedure: `@docs/db-rollback-process.md`
- PgBouncer dual-URL setup: `@docs/db-connection.md`

---

## Docker
- PostgreSQL is external — never containerize it
- Pin image versions (e.g., `node:20-alpine`) — never use `latest`
- Never put secrets in Dockerfiles or compose files — inject at runtime
- Never expose Redis outside the internal Docker network
- Full workflow + compose patterns: `@docs/docker-workflow.md`

---

## Environment Guards
Environments: `development` · `staging` · `production`
- All dev-only scripts must check `NODE_ENV === 'development'` and throw otherwise
- Staging = near-production — no destructive scripts, no dev seed data
- If `DATABASE_URL` is unset, fail loudly at boot — never silently fall back
- `.env.staging` and `.env.production` must never be committed

---

## Git
- Branch naming: `feature/`, `fix/`, `chore/`
- Conventional commits: `type(scope): description`
- One concern per PR

---

## Testing
- Tests required for all new business logic
- Unit tests colocated: `module.test.ts` next to the module
- Integration tests in `/tests/`
- Minimum: happy path + one failure path per function
- Run full suite before marking any task done

---

## Dependency Preferences
<!-- [CUSTOMIZE] List your preferred libraries so Claude doesn't add unnecessary deps -->
<!-- Examples: -->
<!-- - Email: Resend — don't add SendGrid or other providers -->
<!-- - Validation: Zod — don't use Joi or Yup -->
<!-- - HTTP client: Axios (already installed) — don't add node-fetch or got -->

---

## Known Gotchas
<!-- [CUSTOMIZE] Document project-specific footguns discovered during development -->
<!-- Examples: -->
<!-- - The `sessions` table has a unique constraint on (userId, eventId) — always upsert -->
<!-- - Rate limiter uses Redis — tests need REDIS_URL or they'll hang -->
<!-- - Frontend proxy in vite.config.ts only works when backend runs on port 3000 -->
