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
gh run list --branch main --limit 5 \
  --json status,conclusion,name \
  --jq '.[] | "\(.name): \(.conclusion)"'
```

All runs must show `success`. Any `failure` or `cancelled`: **STOP**.

---

## Step 2 — Database migration safety

```bash
npx prisma migrate status 2>&1
git diff origin/main~10..HEAD --name-only | grep "prisma/migrations"
```

**Safe operations (green):**
- [ ] Adding new nullable columns
- [ ] Adding new tables
- [ ] Adding indexes (note: large tables may lock briefly)
- [ ] Updating column defaults (without rewriting data)

**Risky operations — require manual review:**
- [ ] **Column rename** → requires a 3-phase deploy (add new, backfill, remove old)
- [ ] **Column type change** → may require data conversion and a maintenance window
- [ ] **NOT NULL on existing column** → requires backfill before constraint is applied
- [ ] **Table drop** → confirm no code references the table in any deployed version
- [ ] **Large table index add** → use `CREATE INDEX CONCURRENTLY` in raw SQL, not Prisma migration

Flag any risky operation as **Critical** — confirm the deploy strategy before proceeding.

```bash
grep -l "UPDATE\|DELETE\|INSERT INTO" prisma/migrations/*/migration.sql 2>/dev/null
```

Data migrations must be tested against a production-size database snapshot, not just dev data.

---

## Step 3 — Environment variable verification

```bash
grep -rn "process\.env\." \
  --include="*.ts" --include="*.tsx" --include="*.js" \
  --exclude-dir=node_modules --exclude-dir=dist \
  . | grep -oE "process\.env\.[A-Z_]+" | sort -u
```

- [ ] All env vars are documented in `.env.example`
- [ ] All new env vars added in this release are provisioned in production
- [ ] No `.env` file or real secrets are committed to the repository
- [ ] Secrets are stored in the secret manager — not in CI env vars directly

```bash
grep "^\.env" .gitignore || echo "WARNING: .env not in .gitignore"
```

---

## Step 4 — Feature flags

```bash
grep -rn "featureFlag\|feature_flag\|FEATURE_\|isEnabled\|getFlag" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules . 2>/dev/null | head -20
```

- [ ] New features behind a flag: flag is **off** in production until explicitly enabled
- [ ] Flags being fully released: no dead flag checks remaining in code
- [ ] Flags being removed: old flag name is deprovisioned in the flag management system

---

## Step 5 — Rollback plan

**Database rollback:**
- [ ] New schema will work with the old code if rollback is needed?
- [ ] If a data migration ran, is there a reverse migration script ready?

**Application rollback:**
- [ ] Previous Docker image tag (or deployment artifact) is available
- [ ] Rollback can be executed in under 10 minutes

```
ROLLBACK PLAN for [release]:
1. [Step 1 — e.g., "Revert to Docker image tag vX.Y.Z"]
2. [Step 2 — e.g., "Run migration rollback script: ..."]
3. [Step 3 — e.g., "Notify team via #incidents channel"]
Estimated rollback time: [N] minutes
Point of no return: [e.g., "After data migration runs"]
```

---

## Step 6 — Observability and alerting

- [ ] New features have logging at key decision points
- [ ] No sensitive data (PII, tokens, passwords) in log output
- [ ] Error paths log with sufficient context (not just `console.error(e)`)
- [ ] New external integrations have timeout handling and error logging
- [ ] Any new background jobs have failure alerting configured

```bash
grep -rn "catch\s*(.\{0,20\})\s*{$" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules . | head -10
```

---

## Step 7 — No dev artifacts in production

```bash
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

- [ ] No `console.log` in production code paths
- [ ] No hardcoded test accounts, debug routes, or seed data calls
- [ ] No `NODE_ENV !== 'production'` shortcuts in production paths

---

## Step 8 — Documentation and release notes

```bash
head -20 CHANGELOG.md 2>/dev/null || echo "No CHANGELOG.md found"
git log $(git describe --tags --abbrev=0 2>/dev/null || echo "")..HEAD \
  --oneline --no-merges 2>/dev/null | head -30
```

- [ ] CHANGELOG.md updated with this release's changes
- [ ] `README.md` updated if setup instructions or env vars changed
- [ ] API documentation updated if endpoints were added, changed, or removed
- [ ] Any updated ADRs committed alongside the relevant changes

---

## Step 9 — Client handoff readiness

If this release delivers features to a client:

- [ ] Staging environment is up and matches production configuration
- [ ] Client staging access credentials are ready to send
- [ ] Plain-language summary of what's included is drafted
- [ ] Change requests discovered during review are documented as out-of-scope issues
- [ ] Written go-live approval has been received from the client

If any of these are unchecked, do not proceed to production deploy.
Run `/client-handoff "$ARGUMENTS"` to manage the full client delivery process.

---

## Step 10 — Team readiness

- [ ] Support team briefed on new features
- [ ] Rollback procedure shared with the on-call engineer
- [ ] Deployment window confirmed (avoid Fridays, end-of-month, major business events)
- [ ] Monitoring dashboard open and baselined before deploying

---

## Step 11 — Final release checklist output

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

CLIENT
  ✅ / ❌  Staging walkthrough passed
  ✅ / ❌  Written go-live approval received

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

## Step 12 — Tag the release

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
- **NO-GO** → resolve blocking items, re-run affected sections, then re-evaluate
- Post-deploy issue → run `/incident-response live` immediately; do not hot-fix under pressure
- Client-facing release → run `/client-handoff "$ARGUMENTS"` to complete the delivery
