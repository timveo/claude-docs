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
git diff main...HEAD | grep -E "findMany|findFirst|findUnique" -A 5
```

- [ ] Relations accessed in a loop are loaded via `include` or `select` — not fetched inside a `.map()`
- [ ] `findMany` without `take` on a user-facing endpoint → **Warning** (unbounded query)
- [ ] `findMany` without `take` in a background job → document expected volume

**Missing index detection**

```bash
git diff main...HEAD | grep -E "where:\s*\{" -A 3
grep -E "@index|@@index|@unique|@@unique" prisma/schema.prisma
```

Flag fields in `where`, `orderBy`, or `cursor` without an index as **Warning** if table > 10,000 rows.

**Prisma `select` hygiene**

- [ ] Queries use `select` to return only needed fields — not full model objects
- [ ] No `include: { user: true }` when only `user.name` is needed

**Transaction safety**

```bash
git diff main...HEAD | grep -E "\$transaction" -B 2 -A 10
```

- [ ] Multi-step writes are wrapped in `$transaction`
- [ ] Transactions don't include slow operations (external API calls, file I/O)

---

## Step 3 — API response time risks

**Pagination**

```bash
git diff main...HEAD | grep -E "findMany|getAll|list" -A 10 | grep -v "take\|limit\|pageSize"
```

- [ ] Every list endpoint enforces `limit`/`take` with a maximum (e.g., max 100)
- [ ] Default page size is reasonable (≤ 50 for UI lists)

**Parallel vs serial async**

```bash
git diff main...HEAD | grep -E "await " -A 1 | grep -B 1 "await "
```

Look for sequential `await` calls on independent operations — flag as **Warning**:
```typescript
// Serial — slow if independent
const user = await getUser(id);
const orders = await getOrders(id);
// Should be:
const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);
```

**Expensive synchronous operations**

- [ ] No `JSON.parse` / `JSON.stringify` of large objects in the request path
- [ ] No `readFileSync` in request handlers
- [ ] CPU-intensive operations offloaded to a background job

---

## Step 4 — Frontend performance audit

**Bundle size check**

```bash
npm run build 2>&1 | grep -E "chunk|bundle|size|kB|MB" | tail -20
```

Thresholds:
- Initial JS bundle > 200 kB (gzipped): **Warning**
- Initial JS bundle > 500 kB (gzipped): **Critical**
- Any single chunk > 1 MB: **Warning**

**Code splitting**

```bash
git diff main...HEAD | grep -E "^import " | grep -iE "(chart|editor|pdf|video|map|canvas)"
```

- [ ] Large, non-critical libraries are lazy-loaded with `React.lazy()` / `dynamic()`
- [ ] Route-level code splitting in place

**React render optimization**

- [ ] `useEffect` dependencies arrays are correct
- [ ] Expensive computations in render are wrapped in `useMemo`
- [ ] Callback functions passed to memoized children are wrapped in `useCallback`
- [ ] Lists of > 50 items use virtualization or pagination

---

## Step 5 — Memory leak checks

```bash
git diff main...HEAD | grep -E "addEventListener|setInterval|setTimeout|subscribe" -A 3
```

- [ ] Every `addEventListener` in `useEffect` has a cleanup function
- [ ] Every `setInterval` / `setTimeout` in `useEffect` is cleared in the cleanup
- [ ] Every subscription is unsubscribed/closed in cleanup

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

VERDICT: APPROVED / NEEDS FIXES / APPROVED WITH SCALE NOTES
═══════════════════════════════════════════════════════
```

---

## Severity guide

| Level | Meaning | Action |
|---|---|---|
| 🔴 **Critical** | Will degrade or fail at expected production load | Block merge — fix first |
| 🟡 **Warning** | Performance risk above a documented data volume threshold | Fix before merge, or document in a tech debt issue |
| 🔵 **Informational** | Nice-to-have optimization | Log in tech-debt backlog |

---

## Next step

- All Critical resolved, Warnings addressed → proceed with `/security-review` or `/pr-prep`
- Unresolved Critical → return to worktree and fix
- Warning items deferred → create a GitHub issue tagged `tech-debt` with the threshold documented
