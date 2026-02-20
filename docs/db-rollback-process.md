# Database Rollback Process
> Reference doc — load on demand with `@docs/db-rollback-process.md`

---

## Important: Prisma Has No Auto-Rollback
Prisma does not support automatic migration rollback. All rollbacks are forward-only — you write a new corrective migration that undoes the change.

---

## Rollback Process

### Step 1 — Assess the damage
```bash
npx prisma migrate status
```
Identify which migration was applied and what it changed.

### Step 2 — Write a corrective migration
```bash
npx prisma migrate dev --name revert_describe_the_original_change
```
Manually edit the generated SQL to reverse the problematic change:
- Added a column → `ALTER TABLE x DROP COLUMN y`
- Dropped a column → `ALTER TABLE x ADD COLUMN y type`
- Changed a type → reverse the `ALTER COLUMN` statement
- Added a table → `DROP TABLE x`

### Step 3 — Verify locally
```bash
npx prisma migrate status        # Confirm clean state
npx prisma db pull               # Confirm schema matches DB
npm run typecheck                # Confirm Prisma client is valid
npm run test                     # Confirm nothing broken
```

### Step 4 — Deploy corrective migration
```bash
# Via CI/CD only — never run manually in production
npx prisma migrate deploy
```

---

## Emergency: Manual DB Fix (Production Only, Last Resort)
Only if a migration is partially applied and `migrate deploy` is failing:

1. Connect to the DB directly via `DIRECT_URL`
2. Manually reverse the SQL change
3. Delete the broken entry from `_prisma_migrations` table
4. Re-run `npx prisma migrate deploy`
5. Document exactly what was done and why in `/docs/incidents/`

**Never do this without notifying the team first.**

---

## Prevention
- Always test migrations against a staging DB before production
- Keep migrations small and reversible — one concern per migration
- Never combine data migrations with schema migrations in the same file
