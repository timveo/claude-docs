---
name: human-test
description: Starts the Docker stack and prints a manual testing checklist from the issue's acceptance criteria. Run after /build-feature passes CI. Triggers on "test this locally", "spin up docker", "human testing", "start the stack", or "validate issue #N".
disable-model-invocation: true
argument-hint: <issue-number>
---

# /human-test — Local Stack for Human Validation

Run after `/build-feature` passes CI. Starts the Docker environment, runs smoke tests,
and prints a manual testing checklist from the issue's testing plan. For Docker setup
details and compose file structure, see `@docs/docker-workflow.md`.

**Usage:** `/human-test 42`

---

## Step 1 — Read the testing plan

```bash
gh issue view $ARGUMENTS --json body --jq '.body'
```

Extract the **Manual Testing** section from the issue. If there isn't one, derive a
sensible checklist from the acceptance criteria.

---

## Step 2 — Verify Docker is ready

```bash
docker info > /dev/null 2>&1 && echo "Docker: running ✅" || echo "Docker: not running ❌"
ls docker-compose.yml docker-compose.dev.yml 2>/dev/null || echo "No compose file found"
```

If Docker isn't running, tell the user to start Docker Desktop and try again.

---

## Step 3 — Start the stack

```bash
docker compose down

# Dev compose uses volume mounts for hot reload — see docs/docker-workflow.md
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build -d

echo "Waiting for services to be healthy..."
sleep 5
docker compose ps
```

Report which services are running and on which ports (frontend :3000, backend :4000).

---

## Step 4 — Smoke test

```bash
# Health check
curl -s -o /dev/null -w "%{http_code}" http://localhost:4000/api/health \
  && echo " — API health: ✅" || echo " — API health: ❌"

# E2E smoke tests (if configured)
npm run test:e2e -- --grep "@smoke" 2>/dev/null || true
```

---

## Step 5 — Print the testing checklist

```
═══════════════════════════════════════════════
HUMAN TESTING CHECKLIST — Issue #[N]: [Title]
═══════════════════════════════════════════════
Frontend:  http://localhost:3000
API:       http://localhost:4000/api

Steps to verify:
□ [Step from testing plan]
□ [Step from testing plan]
□ [Edge case]
□ [Error state]

When done, report:
  - Which steps passed / failed
  - Any unexpected behavior
  - Screenshots if UI changed
═══════════════════════════════════════════════
```

---

## Step 6 — Document results

After the user reports results, add a comment to the PR:

```bash
gh pr comment [pr-number] --body "## Human Testing Results

Date: $(date +%Y-%m-%d)
Environment: Docker Compose (local)

### Results
[user-reported results]

### Status: PASSED / PASSED WITH NOTES / FAILED"
```

---

## Step 7 — Tear down

```bash
docker compose down
```

Ask if the user wants to leave the stack running or shut it down.

---

## Next step

- Testing **passed** → run `/pr-prep $ARGUMENTS`
- Testing **failed** → return to the worktree, fix, re-run `/build-feature $ARGUMENTS`
