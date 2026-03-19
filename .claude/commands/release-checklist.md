---
name: release-checklist
description: Pre-production release gate. Validates migrations safety, environment variables, feature flags, rollback plan, observability, documentation, release notes, and team readiness before deploying to production. Run after all features for a milestone are merged and CI is green. Triggers on "release checklist", "pre-release", "ready to deploy", "production checklist", or "release gate".
disable-model-invocation: true
argument-hint: [milestone-name or version]
---

# /release-checklist — Pre-Production Release Gate

Run after all milestone PRs are merged to main and CI is fully green.
No deployment should proceed without a completed checklist.

**Usage:** `/release-checklist v1.2.0` or `/release-checklist "Q2 Sprint 3"`

---

## Step 1 — Confirm all milestone work is merged

```bash
gh issue list --milestone "$ARGUMENTS" --state open
```

If any issues are still open: **STOP** — resolve before proceeding.

```bash
# Confirm CI is green on main
gh run list --branch main --limit 5 \
  --json status,conclusion,name \
  --jq '.[] | "\(.name): \(.conclusion)"'
```

All runs must show `success`. Any `failure` or `cancelled`: **STOP**.

---

## Step 2 — Database migration safety

```bash
# Review all pending migrations
npx prisma migrate status 2>&1
```

```bash
# List migration files added in this release
git diff origin/main~10..HEAD --name-only | grep "prisma/migrations"
```

For each migration, verify:

**Safe operations (green):**
- [ ] Adding new nullable columns
- [ ] Adding new tables
- [ ] Adding indexes (note: large tables may lock briefly)
- [ ] Updating column defaults (without rewriting data)

**Risky operations — require manual review:**
- [ ] **Column rename** → requires a 3-phase deploy (add new, backfill, remove old) — not a single migration
- [ ] **Column type change** → may require data conversion and a maintenance window
- [ ] **NOT NULL on existing column** → requires backfill before constraint is applied
- [ ] **Table drop** → confirm no code references the table in any deployed version
- [ ] **Large table index add** → use `CREATE INDEX CONCURRENTLY` in raw SQL, not Prisma migration

Flag any risky operation as **Critical** — confirm the deploy strategy before proceeding.

```bash
# Check for data migrations (large UPDATE or DELETE statements)
grep -l "UPDATE\|DELETE\|INSERT INTO" prisma/migrations/*/migration.sql 2>/dev/null
```

Data migrations must be tested against a production-size database snapshot, not just dev data.

---

## Step 3 — Environment variable verification

```bash
# List all env vars referenced in code
grep -rn "process\.env\." \
  --include="*.ts" --include="*.tsx" --include="*.js" \
  --exclude-dir=node_modules --exclude-dir=dist \
  . | grep -oE "process\.env\.[A-Z_]+" | sort -u
```

Compare against your environment variable checklist:
- [ ] All env vars from the above list are documented in `.env.example`
- [ ] All new env vars added in this release are provisioned in the production environment
- [ ] No `.env` file or real secrets are committed to the repository
- [ ] Secrets are stored in the secret manager (e.g., AWS Secrets Manager, Doppler, Vault) — not in CI env vars directly

```bash
# Confirm .env is gitignored
grep "^\.env" .gitignore || echo "WARNING: .env not in .gitignore"
```

---

## Step 4 — Feature flags

```bash
# Find feature flag references
grep -rn "featureFlag\|feature_flag\|FEATURE_\|isEnabled\|getFlag" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules . 2>/dev/null | head -20
```

For each active feature flag in this release:
- [ ] New features behind a flag: flag is **off** in production until explicitly enabled
- [ ] Flags being fully released: code is clean — no dead flag checks remaining after removal
- [ ] Flags being removed: old flag name is deprovisioned in the flag management system

---

## Step 5 — Rollback plan

Answer each question before proceeding:

**Database rollback:**
- [ ] If we need to roll back the code, will the new database schema still work with the old code?
  - New nullable columns: ✅ old code ignores them
  - New NOT NULL columns: ❌ old code won't set them — rollback may fail
  - Dropped columns old code reads: ❌ rollback will error
- [ ] If a data migration ran, is there a reverse migration script ready?

**Application rollback:**
- [ ] Previous Docker image tag (or deployment artifact) is available and ready to deploy
- [ ] Rollback can be executed in under 10 minutes

Document the rollback procedure:
```
ROLLBACK PLAN for [release]:
1. [Step 1 — e.g., "Revert to Docker image tag vX.Y.Z"]
2. [Step 2 — e.g., "Run migration rollback script: ..."]
3. [Step 3 — e.g., "Notify support team via #incidents channel"]
Estimated rollback time: [N] minutes
Point of no return: [e.g., "After data migration runs — no automatic rollback"]
```

---

## Step 6 — Observability and alerting

- [ ] New features have logging at key decision points (not verbose, but enough to debug in production)
- [ ] No sensitive data (PII, tokens, passwords) in log output
- [ ] Error paths log the error with sufficient context (not just `console.error(e)`)
- [ ] New external integrations have timeout handling and error logging
- [ ] Any new background jobs have failure alerting configured
- [ ] Dashboards/metrics updated if new measurable behaviors were added

```bash
# Check for bare catch blocks (swallowed errors)
grep -rn "catch\s*(.\{0,20\})\s*{$" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules . | head -10
```

---

## Step 7 — No dev artifacts in production

```bash
# Check for seed data calls, test accounts, or debug routes
grep -rn \
  -e "seedDatabase\|seed()\|createTestUser\|debugRoute\|\/debug\/" \
  -e "console\.log\|console\.debug" \
  --include="*.ts" \
  --exclude-dir=node_modules \
  --exclude-dir="*.test.ts" \
  --exclude-dir="*.spec.ts" \
  --exclude-dir=seed \
  . | grep -v "// production\|// intentional"
```

- [ ] No `console.log` in production code paths (use structured logger)
- [ ] No hardcoded test accounts, debug routes, or seed data calls in app code
- [ ] No `NODE_ENV !== 'production'` shortcuts left in production paths

---

## Step 8 — Documentation and release notes

```bash
# Check if CHANGELOG.md exists and has been updated
head -20 CHANGELOG.md 2>/dev/null || echo "No CHANGELOG.md found"
```

- [ ] CHANGELOG.md updated with this release's changes
- [ ] `README.md` updated if setup instructions, commands, or env vars changed
- [ ] API documentation updated if any endpoints were added, changed, or removed
- [ ] Any updated ADRs committed alongside the relevant changes

**Draft release notes:**

```bash
# Get commit history since last tag
git log $(git describe --tags --abbrev=0 2>/dev/null || echo "")..HEAD \
  --oneline --no-merges 2>/dev/null | head -30
```

Use the commit list to draft release notes for stakeholders. Group by: New Features / Improvements / Bug Fixes / Breaking Changes.

---

## Step 9 — Team readiness

- [ ] Support team briefed on new features (what's changing, what questions to expect)
- [ ] Rollback procedure shared with the on-call engineer
- [ ] Deployment window confirmed (avoid Fridays, end-of-month, major business events)
- [ ] Monitoring dashboard open and baselined before deploying

---

## Step 10 — Final release checklist output

```
═══════════════════════════════════════════════════════
RELEASE CHECKLIST — [milestone/version] — [date]
═══════════════════════════════════════════════════════

PRE-FLIGHT
  ✅ / ❌  All milestone issues closed
  ✅ / ❌  CI green on main

DATABASE
  ✅ / ❌  Migration safety reviewed
  ✅ / ❌  Rollback plan documented

ENVIRONMENT
  ✅ / ❌  All env vars provisioned in production
  ✅ / ❌  Feature flags configured correctly

QUALITY
  ✅ / ❌  No dev artifacts in production paths
  ✅ / ❌  Observability in place

DOCUMENTATION
  ✅ / ❌  Changelog updated
  ✅ / ❌  Release notes drafted

TEAM
  ✅ / ❌  Support briefed
  ✅ / ❌  On-call has rollback procedure

BLOCKING ITEMS (must resolve before deploy):
  🔴 [Item]

VERDICT: GO / NO-GO
═══════════════════════════════════════════════════════
```

---

## Step 11 — Tag the release

Once the checklist is fully green:

```bash
git tag -a "$ARGUMENTS" -m "Release $ARGUMENTS — $(date +%Y-%m-%d)"
git push origin "$ARGUMENTS"

gh release create "$ARGUMENTS" \
  --title "Release $ARGUMENTS" \
  --notes "[paste release notes here]"
```

---

## Next step

- **GO** → deploy and monitor for 30 minutes post-deploy before declaring success
- **NO-GO** → resolve blocking items, re-run the affected checklist sections, then re-evaluate
- Post-deploy issue → execute rollback plan immediately; do not attempt hot-fixes under pressure
