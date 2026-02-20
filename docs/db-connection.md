# Database Connection Guide
> Reference doc — load on demand with `@docs/db-connection.md`

---

## Overview
All runtime queries route through PgBouncer. Migrations connect directly to PostgreSQL via `DIRECT_URL`. PostgreSQL itself is hosted externally (RDS/Supabase) — never containerized.

---

## Environment Variables
```env
# PgBouncer (runtime queries)
DATABASE_URL="postgresql://user:pass@pgbouncer-host:6432/dbname?pgbouncer=true&connection_limit=1"

# Direct PostgreSQL (migrations only)
DIRECT_URL="postgresql://user:pass@postgres-host:5432/dbname"
```

## Prisma Schema Config
```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}
```

---

## Rules
- `DATABASE_URL` → PgBouncer → used for all runtime queries
- `DIRECT_URL` → PostgreSQL directly → used by `prisma migrate deploy` only
- PgBouncer runs in **transaction mode** — Prisma handles prepared statement compatibility automatically when `?pgbouncer=true` is set
- `connection_limit=1` per serverless instance — prevents pool exhaustion
- Never run migrations using `DATABASE_URL` — always use `DIRECT_URL` in migration contexts
- Alert if pool connections regularly exceed 80% capacity

---

## Troubleshooting
| Symptom | Likely Cause | Fix |
|---|---|---|
| Migration hangs | Running via PgBouncer | Switch to `DIRECT_URL` |
| Prepared statement error | Missing `?pgbouncer=true` | Add to `DATABASE_URL` |
| Pool exhaustion | `connection_limit` too high | Set to `1` per instance |
| Connection refused | PgBouncer down | Check container health |
