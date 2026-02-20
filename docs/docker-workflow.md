# Docker Workflow
> Reference doc — load on demand with `@docs/docker-workflow.md`

---

## File Structure
```
/your-project-root/
  docker-compose.yml                # Base service definitions
  docker-compose.dev.yml            # Dev overrides (hot reload, debug ports)
  docker-compose.staging.yml        # Staging overrides
  docker-compose.production.yml     # Production overrides
  /frontend/
    Dockerfile                      # Production image
    Dockerfile.dev                  # Dev image (hot reload)
    .dockerignore
  /backend/
    Dockerfile                      # Production image
    Dockerfile.dev                  # Dev image (hot reload)
    .dockerignore
```

---

## Services
| Service     | Container Name  | Internal Port | Exposed (dev only) |
|-------------|-----------------|---------------|--------------------|
| Frontend    | `app-frontend`  | `3000`        | `3000`             |
| Backend API | `app-backend`   | `4000`        | `4000`             |
| Redis       | `app-redis`     | `6379`        | never exposed      |

PostgreSQL → external only (RDS/Supabase). Never containerize it.

---

## Common Commands

### Development
```bash
# Start all services with hot reload
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# Stop services
docker-compose down

# Stop and remove volumes (dev only — destructive)
docker-compose down -v

# Rebuild a single service
docker-compose up --build backend

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Shell into a container
docker-compose exec backend sh
```

### Staging
```bash
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d --build
docker-compose logs -f
```

### Production
```bash
# Production deploys via CI/CD only — never run locally
docker-compose -f docker-compose.yml -f docker-compose.production.yml up -d
```

---

## Build & Registry
```bash
# Build images
docker build -t app-frontend ./frontend
docker build -t app-backend ./backend

# Tag for registry (replace with your actual registry URL)
docker tag app-backend your-registry/app-backend:v1.0.0
docker tag app-frontend your-registry/app-frontend:v1.0.0

# Push
docker push your-registry/app-backend:v1.0.0
docker push your-registry/app-frontend:v1.0.0
```
> Always use explicit version tags — never `latest` in staging or production.

---

## Compose File Rules
- `docker-compose.yml` — base definitions only, no env-specific values
- `docker-compose.dev.yml` — volume mounts, debug ports, relaxed limits
- `docker-compose.staging.yml` — mirrors production config, no debug ports
- `docker-compose.production.yml` — strict resource limits, health checks required, no volume mounts
- Never merge dev overrides into the base file

---

## Production Image Standards
- Multi-stage builds — dev dependencies never reach production image
- Non-root user — containers must not run as root
- Pinned base images — `node:20-alpine` not `node:latest`
- Health checks defined for every service
- Resource limits set (`mem_limit`, `cpus`)
- `.dockerignore` must exclude: `node_modules`, `.env*`, `/tests`, build artifacts
