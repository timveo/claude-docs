# CLAUDE.md — Project Configuration

> **[CUSTOMIZE]** Replace placeholder commands, paths, and domain context with your project's specifics before use.
> This file is read automatically by Claude Code at the start of every session.
> Last updated: [DATE] — keep the "Current Sprint" section live at all times.

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
    /middleware/         # Auth, rate limiting, error handling
    /lib/                # Shared utilities — CHECK HERE before writing new ones
    /types/              # TypeScript type definitions
    /workers/            # Background jobs (BullMQ)
/prisma/                 # Schema + migrations (source of truth)
/docs/                   # Reference docs (load on demand, e.g. @docs/docker-workflow.md)
```
<!-- [CUSTOMIZE] Update the tree above to match your actual project structure -->

---

## Workflow

### Every Task
1. **Plan first** — Use Plan Mode for tasks with 3+ steps or architectural decisions. Stop and re-plan if approach isn't working.
2. **Check in** — Confirm approach with user before starting complex work
3. **Verify** — Run `/verify` (lint + typecheck + tests) before declaring done
4. **Use subagents** for research and exploration — keep main context clean

### Feature Development Loop
Each feature runs in its own git worktree — see **Parallel Development** below.

| Step | Command | What happens |
|------|---------|-------------|
| Start feature | `/feature-start [N]` | Worktree + branch + draft PR |
| Build | `/build-feature [N]` | Plan → TDD build → AC verify |
| Quality check | `/verify` | Lint + typecheck + tests |
| Security gate | `/security-review` | OWASP audit — **required** if touching auth/data/integrations |
| Performance gate | `/performance-review` | N+1, bundle, indexes — **required** if touching queries/lists |
| Human test | `/human-test [N]` | Docker stack + testing checklist |
| Merge | `/pr-prep [N]` | Rebase + review + create PR + cleanup |
| Client delivery | `/client-handoff` | Staging walkthrough → client review → go-live approval |

### Production Quality Gates
| Command | Cadence |
|---------|---------|
| `/tech-debt current-branch` | Per PR (optional) — debt in changed code |
| `/tech-debt full-codebase` | Monthly / pre-sprint — full triage |
| `/tech-debt adr "title"` | When making deliberate trade-offs |
| `/dependency-audit` | Monthly + before every release |
| `/release-checklist [version]` | Before every production deploy — GO / NO-GO |
| `/incident-response` | When something breaks in production |

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
- Branch naming: `feat/[issue-number]-[short-title]` · `fix/[issue-number]-[short-title]`
- Conventional commits: `type(scope): description`
- One concern per PR — squash merge to main
- Always open a draft PR when a branch is created (handled by `/feature-start`)

---

## Parallel Development
Multiple features can be developed simultaneously using git worktrees.
Each feature gets its own isolated directory and branch.

```
/desktop/development/
├── my-project/              ← main (always clean)
├── my-project-feat-42/      ← feat/42-user-auth  (worktree)
└── my-project-feat-51/      ← feat/51-dashboard  (worktree)
```

Rules:
- Max 3–4 parallel tracks for a team of 3–6
- Only parallelize issues that don't touch the same files
- Rebase against main daily to minimize conflicts
- Run `/parallel-status` as your daily standup dashboard
- **Hard cap: no new `/feature-start` if 2+ PRs are already awaiting review — clear the queue first**

---

## Testing
- Tests required for all new business logic
- Write tests before implementation (test-first)
- Unit tests colocated: `module.test.ts` next to the module
- Integration tests in `/tests/`
- **Minimum bar: every new service function gets a happy path test + one failure path test**
- **Every new API endpoint gets at least one integration test**
- Run `/verify` before marking any task done

---

## Dependency Preferences
<!-- [CUSTOMIZE] Preferred libraries — prevents Claude adding unwanted deps -->
<!-- e.g. Email: Resend · Validation: Zod · HTTP: Axios (installed) -->

---

## Known Gotchas
<!-- [CUSTOMIZE] Project-specific footguns discovered during development -->
<!-- e.g. sessions table has unique (userId, eventId) — always upsert -->

---

## Current Sprint
<!-- [CUSTOMIZE] Milestone: [NAME] · Due: [DATE] · GitHub: [URL] -->
<!-- Priority issues: #N, #N, #N -->
