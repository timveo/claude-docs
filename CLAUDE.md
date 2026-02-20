# CLAUDE.md — Project Configuration

> Node.js/TypeScript monorepo: React frontend + Node API. Read this before every session. Check `tasks/lessons.md` for project-specific corrections before starting work.

---

## Core Principles
- **Simplicity First** — Minimal code change that solves the problem. No over-engineering.
- **No Laziness** — Find root causes. No temporary fixes. Senior developer standards.
- **Minimal Impact** — Only touch what's necessary. Avoid side effects.
- **Verify Before Done** — Never mark complete without proving it works. Ask: *"Would a staff engineer approve this?"*

---

## Workflow

### Every Task
1. **Plan first** — Write `[ ]` checklist to `tasks/todo.md` before writing any code
2. **Check in** — Confirm plan with user on complex tasks before starting
3. **Track** — Mark `[x]` as you go; summarize changes at each step
4. **Verify** — Run lint + tests + typecheck before declaring done
5. **Capture** — Update `tasks/lessons.md` after any user correction

### Plan Mode
- Use Plan Mode (Shift+Tab twice) for any task with 3+ steps or architectural decisions
- STOP and re-plan if something goes sideways — don't keep pushing
- Use subagents for research and exploration to keep main context clean

### Self-Improvement
- After ANY correction: log the pattern in `tasks/lessons.md`
- Review `tasks/lessons.md` at session start — apply learned rules immediately

---

## Commands
```bash
npm run dev           # Start dev server
npm run build         # Production build
npm run lint          # ESLint
npm run typecheck     # TypeScript check
npm run test          # Run tests
npm run db:migrate    # Run Prisma migrations
npm run db:seed       # Seed reference data (all envs)
npm run db:seed:dev   # Seed dev data (dev only)
```
> Always run `npm run lint && npm run typecheck && npm run test` before marking done.

---

## Architecture
```
/frontend/            # React app
/backend/
  /src/
    /agents/          # AI agent modules (PCMA)
    /api/             # Routes and controllers
    /lib/             # Shared utilities — check here before writing new ones
    /types/           # TypeScript types
/prisma/              # Schema and migrations
/scripts/db/          # Dev-only DB scripts
/tasks/               # todo.md + lessons.md
/docs/                # Architecture, ADRs, reference docs
```
For full architecture: `@docs/architecture.md`

---

## Code Standards

### TypeScript
- Strict mode — no `any` types, ever
- Named exports only — no default exports
- `type` over `interface` unless extending
- Explicit error types — no bare `try/catch` in business logic

### Functions
- Single responsibility — one purpose per function
- Under 40 lines — extract if longer
- Pure functions preferred; isolate side effects to boundaries
- Check `/src/lib/` for existing utilities before writing new ones

### Error Handling
- Never swallow errors silently
- Log with context: `logger.error('operation failed', { error, userId, requestId })`
- API routes always return typed error responses

### Security
- NEVER commit secrets, `.env` files, or API keys
- Validate all external inputs at the boundary
- Sanitize before any DB write or HTML render
- Hash/encrypt sensitive data before storing — never plaintext

---

## Logging
- Use the project logger (`/src/lib/logger.ts`) — never `console.log` in production code
- Every log entry must include: `{ requestId, userId?, environment, timestamp }`
- Log levels: `error` (failures) · `warn` (recoverable issues) · `info` (key events) · `debug` (dev only)
- NEVER log passwords, tokens, full request bodies, or PII
- Agent decisions must always be logged at `info` level with full context for auditability

---

## API Conventions
- All routes versioned: `/api/v1/resource`
- HTTP methods: GET (read) · POST (create) · PUT (replace) · PATCH (update) · DELETE (remove)
- Success response shape:
  ```typescript
  { success: true, data: T }
  ```
- Error response shape:
  ```typescript
  { success: false, error: { code: string, message: string } }
  ```
- NEVER expose Prisma models or DB internals directly — always map to a response DTO
- Return 400 for validation errors, 401 for auth, 403 for permissions, 404 for not found, 500 for server errors

---

## Testing
- Tests for all new business logic — no exceptions
- Unit tests colocated: `module.test.ts`
- Integration tests in `/tests/`
- Minimum: happy path + one failure path per function
- Run full suite before marking any task done

---

## Git
- Branch per task: `feature/`, `fix/`, `chore/`
- Conventional commits: `type(scope): description`
- Never push directly to `main` — PRs only, one concern per PR

---

## Database (PostgreSQL + Prisma)
- Schema is source of truth: `/prisma/schema.prisma` — never edit migrations manually
- Run `npx prisma generate` after every schema change
- Use `prisma.$transaction()` for multi-step atomic writes
- Use `select` to limit fields — never fetch full records unnecessarily
- Use `upsert` in all seed scripts — never bare `create` (idempotency)
- NEVER run `deleteMany({})` without a `where` clause
- NEVER use `prisma.$queryRaw` without parameterized inputs
- NEVER run `migrate reset` or dev seed scripts outside `development`
- Migrations in production via CI/CD only — never manually
- For migration rollback procedure: `@docs/db-rollback-process.md`
- PgBouncer config and dual-URL setup: `@docs/db-connection.md`

---

## Docker
- Environments map to compose files: `docker-compose.[dev|staging|production].yml`
- PostgreSQL is external (RDS/Supabase) — never containerize it
- NEVER expose Redis (`6379`) outside the internal Docker network
- NEVER use `latest` image tag — pin versions (e.g. `node:20-alpine`)
- NEVER put secrets in Dockerfiles or compose files — inject at runtime
- NEVER run `docker-compose down -v` outside development — destroys volumes
- Production deploys via CI/CD only — never build and push manually
- Full Docker workflow: `@docs/docker-workflow.md`

---

## Environment Guards
Environments: `development` · `staging` · `production`

```typescript
// /src/lib/env.ts
export const ENV = process.env.NODE_ENV ?? 'development'
export const isProd = ENV === 'production'
export const isStaging = ENV === 'staging'
export const isDev = ENV === 'development'
```
- All `/scripts/db/` scripts must throw if `NODE_ENV !== 'development'`
- Staging = near-production — no destructive scripts, no dev seed data
- `.env.staging` and `.env.production` must never be committed to Git
- If `DATABASE_URL` is unset, fail loudly at boot — never silently fall back

---

## AI Agents (PCMA)
- Each agent module must be independently testable
- Interfaces defined in `/src/types/agents.ts`
- Human approval gates required for all destructive operations — never auto-approve
- All agent decisions logged at `info` level with full context
- Full orchestration patterns: `@docs/agent-architecture.md`

---

## Critical Rules
- NEVER modify `.env` files — ask the user
- NEVER delete files without explicit confirmation
- NEVER push to `main` directly
- ALWAYS check `/src/lib/` before writing new utilities
- ALWAYS run lint + typecheck + tests before declaring done
