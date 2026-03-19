# Code Review Framework — Plan Mode
> Reference doc — load on demand with `@docs/plan-mode-prompt.md`

Use this framework for structured code reviews. Choose the review mode based on scope.

---

## Review Modes

**BIG CHANGE** — new feature, architectural change, new API surface, DB schema change
→ Full section-by-section analysis, up to 4 issues per section

**SMALL CHANGE** — bug fix, copy change, config tweak, minor enhancement
→ One focused check per section

---

## Core Engineering Values

- **DRY is important** — flag repetition aggressively. If logic appears twice, it belongs in a shared utility.
- **Well-tested code is non-negotiable** — err on the side of more tests, not fewer. Test edge cases, not just happy paths.
- **Explicit over clever** — readable code beats clever code. If you have to explain what it does, refactor it.
- **Handle edge cases comprehensively** — empty arrays, null values, network failures, concurrent requests.

---

## The Four Review Stages

Work through these in order. Pause after each stage and ask for feedback before continuing.

### Stage 1 — Architecture

Examine system design and structure:
- Does the change respect existing module boundaries?
- Are new dependencies necessary, or does something already do this?
- Does data flow through the correct layers? (controller → service → repository)
- Are there security implications — new attack surface, auth gaps, data exposure?
- Does this scale? (N+1 queries, unbounded queries, missing pagination)

### Stage 2 — Code Quality

Examine implementation:
- DRY violations — repeated logic that belongs in a shared utility
- Functions doing more than one thing (name has "and" in it → split it)
- Error handling — every failure path explicitly handled, no swallowed exceptions
- Naming clarity — variables, functions, and files named for what they are, not what they do
- Technical debt introduced — quick fixes that will need cleanup later (flag with a ticket)

### Stage 3 — Tests

Examine test coverage:
- Happy path covered for all new logic
- At least one failure path per function
- Edge cases: empty input, null, boundary values, concurrent access
- Are tests testing behavior or implementation? (behavior tests survive refactoring; implementation tests don't)
- Integration tests for new API endpoints

### Stage 4 — Performance

Examine runtime characteristics:
- N+1 queries — any loop that hits the database
- Missing database indexes on new query patterns
- Memory leaks — event listeners not cleaned up, large objects held in closures
- Unnecessary recomputation — expensive operations that could be memoized or moved outside loops

---

## Issue Documentation Format

For every issue found, provide:

1. **Location** — file name and line number
2. **Problem** — concrete description of what's wrong and why it matters
3. **Options** — 2–3 approaches, each with implementation effort and risk level
   - Include "leave as-is and track as debt" when genuinely acceptable
4. **Recommendation** — your preferred option with rationale

Then **pause** — ask which option to pursue before making any changes.
Do not assume timeline priorities. The developer may have context you don't.

---

## Example

```
Stage 2 — Code Quality finding:

Location: backend/src/services/orderService.ts, lines 45–67

Problem: getOrdersForUser() fetches all orders then filters in-memory. With large datasets
this will OOM. This logic is also duplicated in reportService.ts line 112.

Options:
  A) Push the filter into the Prisma query (where clause) — low effort, resolves both perf and DRY
  B) Extract a shared queryOrders(filter) utility in /src/lib/ — medium effort, better long-term
  C) Leave as-is and track as tech debt — low effort now, risk grows with data volume

Recommendation: Option A now, refactor to B in the next sprint when we add more query patterns.

Proceed with A, or would you like to discuss?
```
