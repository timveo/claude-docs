---
name: code-review
description: Pre-PR code audit covering correctness, error handling, type safety, security, and architecture using the full review framework in @docs/plan-mode-prompt.md. Called as part of /pr-prep. Also triggers on "review my code", "check for issues", "audit this PR", or "is this ready to merge".
---

# /code-review — Pre-PR Code Audit

Run before every PR to catch issues before a human reviewer sees them.
Called as part of `/pr-prep`. Can also be run standalone at any time.

For the full structured review framework (BIG vs SMALL change, four-section analysis),
see: `@docs/plan-mode-prompt.md`

---

## Step 1 — Survey all changes

```bash
git diff main...HEAD
git diff
```

Understand the full scope: what changed, how many files, how much logic.

Classify the change:
- **BIG**: new feature, architecture change, new API surface, DB schema change
- **SMALL**: bug fix, copy change, config tweak, minor enhancement

Use `@docs/plan-mode-prompt.md` for BIG changes (section-by-section analysis).
For SMALL changes, one focused check per section below.

---

## Step 2 — Review each modified file across these dimensions

For each file:

**Correctness**
- Bugs, logic errors, off-by-one, wrong conditionals

**Error handling**
- Missing error handlers, unhandled promise rejections, swallowed exceptions
- Edge cases: empty arrays, null values, network failures

**Code hygiene**
- Debug artifacts (`console.log`, commented-out code, TODO without a ticket)
- Dead code that was never cleaned up

**Type safety**
- `any` types — should be `unknown` with narrowing or a proper type definition
- Missing return type annotations on exported functions
- Unsafe type assertions (`as SomeType` without validation)

**Security**
- Unvalidated user input reaching a database query, file path, or shell command
- Hardcoded credentials, tokens, or secrets
- Missing auth checks on new routes
- SQL injection risk in raw queries

**API contract**
- New routes follow the versioning convention (`/api/v1/...`)
- Response shape matches `{ success: true, data: T }` / `{ success: false, error: {...} }`
- No Prisma models returned directly — DTOs used

**Architecture**
- Business logic placed in controllers instead of services
- Direct DB access outside the service layer
- New utilities written when an equivalent exists in `/src/lib/`

---

## Step 3 — Classify findings

- **Critical**: must fix before merging (security issue, broken functionality, data corruption risk)
- **Warning**: should fix before merging (code quality, missing test coverage)
- **Nit**: optional improvement (style, naming)

---

## Step 4 — Output

List all findings with:
- File and line reference
- Issue description
- 2–3 options with tradeoffs (include "leave as-is" only if truly acceptable)
- Recommended action

Pause after listing all findings — ask which to address before making any changes.

If there are no findings: "✅ Code review passed — no issues found."
