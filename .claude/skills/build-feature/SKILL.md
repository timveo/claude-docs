---
name: build-feature
description: Structured Plan → TDD Build → Acceptance Criteria Verification loop for a GitHub issue. Run inside a worktree after /feature-start. Nothing ships without every acceptance criterion verified. Triggers on "build issue", "implement #N", "work on this feature", or "start coding".
disable-model-invocation: true
argument-hint: <issue-number>
---

# /build-feature — Plan, Build, and Verify a Feature

Run this inside a worktree after `/feature-start`. Guides a structured
Plan → TDD Build → Acceptance Criteria Verification cycle. Nothing ships
without every acceptance criterion explicitly verified.

**Usage:** `/build-feature 42`

---

## Guiding principle

Every step maps back to the issue's acceptance criteria. Write the plan before
touching source files. Write tests before writing implementation. Verify every
acceptance criterion before declaring done.

---

## Phase 1: PLAN

### Read the issue
```bash
gh issue view $ARGUMENTS
```

### Explore the codebase
Before writing a plan, understand existing patterns:
- Find similar features already built — follow their patterns
- Check `/src/lib/` for existing utilities before writing new ones
- Read the relevant controller / service / route files

### Present the implementation plan

Structure the plan as:
1. **New files** — name, purpose, what it exports
2. **Existing files to modify** — what changes and why
3. **Types/interfaces** — define them explicitly upfront
4. **Test files** — what will be tested and how
5. **Sequence** — build order (dependencies first)

Ask: "Does this plan look right? Any changes before I start building?"
**Wait for confirmation before writing any code.**

Use Plan Mode for this step — stop and re-plan if the approach isn't working.

---

## Phase 2: BUILD (test-first)

### Write tests before implementation

For each piece of logic:
1. Create the test file
2. Write tests describing expected behavior — they will fail initially (that's correct)
3. Implement to make the tests pass

```bash
# Confirm tests fail before implementation (expected)
npm test -- --testPathPattern=[test-file] 2>&1 | tail -20
```

### Implement

- No `any` types — use `unknown` and narrow, or define proper types
- Controller → Service → Prisma (3-layer pattern — never business logic in controllers)
- Handle all error cases explicitly — no bare `try/catch` swallowing errors
- Use the project logger, never `console.log`
- Check `/src/lib/` before writing new utilities

### Type-check and lint as you go

After each logical chunk:
```bash
npm run typecheck 2>&1 | tail -20
npm run lint 2>&1 | tail -20
```

Fix errors immediately — never let them accumulate.

---

## Phase 3: VERIFY

### Run the full quality suite
```bash
/verify
```

All checks must pass. Fix anything failing before proceeding.

### Audit acceptance criteria

Go through each criterion from the issue and confirm it is met:

```
Acceptance Criteria Audit:
✅ [Criterion 1] — verified by: [test name or observable behavior]
✅ [Criterion 2] — verified by: [test name or manual check]
⚠️  [Criterion 3] — UNMET: [what's missing]
```

If any criterion is unmet, do not proceed — address it first.

---

## Phase 4: COMMIT

```bash
git add [specific files — never git add -A without reviewing]
git commit -m "feat([scope]): [what was built]

- [Key change 1]
- [Key change 2]

Refs #$ARGUMENTS"
```

The pre-commit hook runs lint-staged automatically.

---

## Phase 5: PUSH

```bash
git push
```

Check CI:
```bash
gh pr checks
```

Update the draft PR body with what was built and which acceptance criteria are met.

---

## Summary

Report:
- Files created/modified
- Tests written (count)
- Acceptance criteria status (all ✅ or note what's pending)
- CI status
- Next: `/human-test $ARGUMENTS` for human validation, then `/pr-prep $ARGUMENTS`
