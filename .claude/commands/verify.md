---
name: verify
description: Runs the full quality suite — lint, TypeScript typecheck, and tests for both frontend and backend in parallel. Called automatically at the end of /build-feature and /pr-prep. Also triggers on "run tests", "check everything", "does it pass", or "verify before PR".
---

# /verify — Full Quality Suite

Run the complete lint + typecheck + test pipeline before any PR or deployment.
Called automatically at the end of `/build-feature` and `/pr-prep`.

---

## [CUSTOMIZE] Adapt the paths below to your monorepo structure.

Run backend and frontend checks in parallel:

**Backend** (from `/backend`):
```bash
npm run lint
npm run typecheck
npm run test
```

**Frontend** (from `/frontend`):
```bash
npm run lint
npm run typecheck
npm run test
```

Run both sets in parallel where possible to save time.

---

## Output

- Document any failures with file name, line number, and error message
- If all checks pass, report: "✅ Verify passed — lint, typecheck, and tests clean"
- If any fail, stop and fix before proceeding — do not mark a task done with failing checks
