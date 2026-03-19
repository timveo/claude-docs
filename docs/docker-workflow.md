# Docker Workflow Reference
> Reference doc — load on demand with `@docs/docker-workflow.md`

---

## Architecture

Three compose override files layer on top of a shared base. PostgreSQL is always external
(RDS/Supabase) — never containerized.

```
docker-compose.yml               # Base definitions only — no env-specific values
docker-compose.dev.yml           # Dev overrides: hot reload, debug ports, volume mounts
docker-compose.staging.yml       # Staging overrides: mirrors prod, no debug ports
docker-compose.prod.yml          # Prod overrides: hardened, CI/CD only
```

---

## Services

| Service | Container name | Port | Notes |
|---------|---------------|------|-------|
| Frontend | `app-frontend` | 3000 | React/Vite |
| Backend | `app-backend` | 4000 | Express API |
| Redis | `app-redis` | 6379 | Never exposed externally |
| PostgreSQL | — | — | External (RDS/Supabase) — not containerized |

---

## Development

```bash
# Start full stack with hot reload
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# Rebuild a single service (after dependency changes)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build app-backend

# Tail logs for a service
docker compose logs -f app-backend

# Stop everything
docker compose down
```

Dev-specific config enables:
- Volume mounts for hot reload (source changes reflect immediately)
- Debug ports exposed
- Relaxed resource constraints

---

## Staging

```bash
# Staging mirrors production minus debug ports
docker compose -f docker-compose.yml -f docker-compose.staging.yml up --build
```

Rules:
- No debug ports
- No dev seed data
- Treat as near-production — no destructive scripts

---

## Production

Production deployments **only** through CI/CD pipelines — never run locally.

Production build requirements:
- Multi-stage builds — dev dependencies must never reach the production image
- Non-root user inside containers
- Pinned base image versions (e.g., `node:20-alpine`) — never `latest`
- Health checks required on all services
- Secrets injected at runtime via environment — never in Dockerfiles or compose files

---

## .dockerignore

Always exclude from Docker build context:
```
node_modules
.env*
*.test.ts
*.spec.ts
/tests
/coverage
.git
dist
build
```

---

## Rules (from CLAUDE.md)

- PostgreSQL is external — never containerize it
- Pin image versions — never use `latest`
- Never put secrets in Dockerfiles or compose files — inject at runtime
- Never expose Redis outside the internal Docker network
- Multi-stage builds for production — dev dependencies never reach prod
