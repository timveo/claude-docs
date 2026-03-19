---
name: performance-review
description: Performance audit of changes on the current branch. Covers database query efficiency (N+1, missing indexes, slow Prisma patterns), frontend bundle size, React render optimization, and API response time risks. Run before merging features that touch data-heavy pages, lists, or search. Triggers on "performance review", "check for N+1", "bundle analysis", "optimize queries", or "perf audit".
disable-model-invocation: true
---

# /performance-review — Performance Audit

Run before merging features that touch database queries, list/table rendering, search,
file processing, or any user-facing page that will handle significant data volume.

---

## Step 1 — Scope the review

```bash
git diff main...HEAD --name-only
```

Identify performance risk areas:

| Files changed | Performance risks |
|---|---|
| Prisma queries, repositories, services | N+1, missing includes, full-table scans |
| List/table components, infinite scroll | Virtualization, pagination, re-renders |
| API route handlers | Missing pagination, unbounded queries |
| `package.json` (new dependencies) | Bundle size impact |
| Background jobs, cron tasks | Memory leaks, blocking the event loop |
| Image handling, file uploads | Unoptimized assets, missing compression |

---

## Step 2 — Database query audit

**N+1 detection**

```bash
# Find Prisma findMany calls — check each for missing includes
git diff main...HEAD | grep -E "findMany|findFirst|findUnique" -A 5
```

For each query, check:
- [ ] Relations accessed in a loop are loaded via `include` or `select` in the parent query — not fetched inside a `.map()` or `forEach()`
- [ ] `findMany` without `take` on a user-facing endpoint → **Warning** (unbounded query)
- [ ] `findMany` without `take` in a background job processing all records → document expected volume

**Missing index detection**

```bash
# Find fields used in where clauses that may lack indexes
git diff main...HEAD | grep -E "where:\s*\{" -A 3
```

For each `where` clause on a Prisma model, verify the field has a `@index` or `@@index` in the schema:

```bash
# Check current schema indexes
grep -E "@index|@@index|@unique|@@unique" prisma/schema.prisma
```

Flag fields used in `where`, `orderBy`, or `cursor` without an index as **Warning** if the table will have > 10,000 rows.

**Prisma `select` hygiene**

- [ ] Queries fetching large models use `select` to return only needed fields — not full model objects
- [ ] No `include: { user: true }` when only `user.name` is needed — use `select: { user: { select: { name: true } } }`

**Transaction safety**

```bash
git diff main...HEAD | grep -E "\$transaction" -B 2 -A 10
```

- [ ] Multi-step writes that must be atomic are wrapped in `$transaction`
- [ ] Transactions don't include slow operations (external API calls, file I/O) that hold locks

---

## Step 3 — API response time risks

**Pagination**

```bash
git diff main...HEAD | grep -E "findMany|getAll|list" -A 10 | grep -v "take\|limit\|pageSize"
```

- [ ] Every list endpoint accepts and enforces `limit`/`take` with a maximum (e.g., max 100)
- [ ] Cursor-based or offset pagination implemented for large datasets
- [ ] Default page size is reasonable (≤ 50 for UI lists)

**Parallel vs serial async**

```bash
# Find sequential awaits that could be parallelized
git diff main...HEAD | grep -E "await " -A 1 | grep -B 1 "await "
```

Look for patterns like:
```typescript
// Serial — slow if independent
const user = await getUser(id);
const orders = await getOrders(id);

// Should be:
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
```

Flag sequential `await` calls on independent operations as **Warning**.

**Expensive synchronous operations**

- [ ] No `JSON.parse` / `JSON.stringify` of large objects in the request path
- [ ] No synchronous file reads (`readFileSync`) in request handlers
- [ ] CPU-intensive operations offloaded to a background job — not blocking the event loop

---

## Step 4 — Frontend performance audit

**Bundle size check**

```bash
# If build script exists, run and capture bundle stats
npm run build 2>&1 | grep -E "chunk|bundle|size|kB|MB" | tail -20
```

Thresholds:
- Initial JS bundle > 200 kB (gzipped): **Warning**
- Initial JS bundle > 500 kB (gzipped): **Critical**
- Any single chunk > 1 MB: **Warning**

**Code splitting**

```bash
# Check for large libraries imported without lazy loading
git diff main...HEAD | grep -E "^import " | grep -iE "(chart|editor|pdf|video|map|canvas)"
```

- [ ] Large, non-critical libraries (charts, rich text editors, PDFs) are lazy-loaded with `React.lazy()` / `dynamic()`
- [ ] Route-level code splitting in place (Next.js automatic or React Router lazy routes)

**React render optimization**

```bash
git diff main...HEAD | grep -E "useEffect|useState|useMemo|useCallback" -B 2 -A 5
```

- [ ] `useEffect` dependencies arrays are correct — no missing or excess dependencies
- [ ] Expensive computations in render are wrapped in `useMemo`
- [ ] Callback functions passed as props to memoized children are wrapped in `useCallback`
- [ ] Lists of > 50 items use virtualization (e.g., `react-window`, `react-virtual`) or pagination

```bash
# Check for array/object literals created inline in JSX (defeats memoization)
git diff main...HEAD | grep -E "=\{\[|\=\{\{" | grep -v "className\|style"
```

---

## Step 5 — Memory leak checks

```bash
git diff main...HEAD | grep -E "addEventListener|setInterval|setTimeout|subscribe" -A 3
```

- [ ] Every `addEventListener` in `useEffect` has a cleanup function removing it
- [ ] Every `setInterval` / `setTimeout` in `useEffect` is cleared in the cleanup
- [ ] Every subscription (Rx, EventEmitter, WebSocket) is unsubscribed/closed in cleanup
- [ ] No closures over large data structures in long-lived intervals

---

## Step 6 — Performance review output

```
═══════════════════════════════════════════════════════
PERFORMANCE REVIEW — [branch-name] — [date]
═══════════════════════════════════════════════════════

SCOPE: [list of files/areas reviewed]

CRITICAL (likely to cause production incidents under load):
  🔴 [Issue] — [file:line] — [remediation]

WARNING (will degrade at scale — fix before merge or document threshold):
  🟡 [Issue] — [file:line] — [remediation] — [acceptable scale: N rows/users]

INFORMATIONAL (optimization opportunities, low urgency):
  🔵 [Issue] — [note]

PASSED CHECKS:
  ✅ No N+1 queries detected
  ✅ All list endpoints paginated
  ✅ Bundle size within thresholds
  ✅ No synchronous blocking in request handlers
  [additional passing checks...]

VERDICT: APPROVED / NEEDS FIXES / APPROVED WITH SCALE NOTES
═══════════════════════════════════════════════════════
```

---

## Severity guide

| Level | Meaning | Action |
|---|---|---|
| 🔴 **Critical** | Will degrade or fail at expected production load | Block merge — fix first |
| 🟡 **Warning** | Performance risk above a documented data volume threshold | Fix before merge, or document the threshold in a tech debt issue |
| 🔵 **Informational** | Nice-to-have optimization | Log in tech-debt backlog |

---

## Next step

- All Critical resolved, Warnings addressed or documented → proceed with `/security-review` or `/pr-prep`
- Unresolved Critical → return to worktree and fix
- Warning items deferred → create a GitHub issue tagged `tech-debt` with the threshold documented
