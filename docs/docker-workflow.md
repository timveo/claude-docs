# Docker Workflow
> Reference doc — load on demand with `@docs/docker-workflow.md`

---

## File Structure
```
docker-compose.yml                # Base service definitions
docker-compose.dev.yml            # Dev overrides (hot reload, debug ports)
docker-compose.staging.yml        # Staging overrides
docker-compose.production.yml     # Production overrides
/frontend/Dockerfile              # Production (multi-stage)
/frontend/Dockerfile.dev          # Dev (hot reload)
/backend/Dockerfile               # Production (multi-stage)
/backend/Dockerfile.dev           # Dev (hot reload)
```

## Services
| Service | Container | Internal Port | Exposed (dev only) |
|---|---|---|---|
| Frontend | `app-frontend` | `3000` | `3000` |
| Backend | `app-backend` | `4000` | `4000` |
| Redis | `app-redis` | `6379` | never |

PostgreSQL is external (RDS/Supabase) — never containerize it.

---

## Commands

### Development
```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up --build
docker-compose down                    # stop
docker-compose down -v                 # stop + destroy volumes (dev only)
docker-compose up --build backend      # rebuild single service
docker-compose logs -f backend         # tail logs
docker-compose exec backend sh         # shell into container
```

### Staging / Production
```bash
# Staging
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d --build

# Production — CI/CD only, never run locally
docker-compose -f docker-compose.yml -f docker-compose.production.yml up -d
```

---

## Compose File Rules
- `docker-compose.yml` — base definitions only, no env-specific values
- `docker-compose.dev.yml` — volume mounts, debug ports, relaxed limits
- `docker-compose.staging.yml` — mirrors production, no debug ports
- `docker-compose.production.yml` — resource limits, health checks, no volume mounts
- Never merge dev overrides into the base file

## Production Image Standards
- Multi-stage builds — dev dependencies never reach production
- Non-root user in containers
- Pinned base images (`node:20-alpine`, not `node:latest`)
- Health checks on every service
- `.dockerignore` excludes: `node_modules`, `.env*`, `/tests`, build artifacts
