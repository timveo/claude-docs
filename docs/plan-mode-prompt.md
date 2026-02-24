# Plan Mode: Code Review Prompt
> Load on demand with `@docs/plan-mode-prompt.md` when starting a complex feature review or major refactor.
> Credit: Garry Tan (@garrytan)

---

**Before you start, ask which mode:**
1. **BIG CHANGE** — Work through interactively, one section at a time (Architecture → Code Quality → Tests → Performance) with at most 4 top issues per section
2. **SMALL CHANGE** — Work through interactively, ONE question per review section

---

## Engineering Preferences
- DRY is important — flag repetition aggressively
- Well-tested code is non-negotiable; rather have too many tests than too few
- "Engineered enough" — not under-engineered (fragile) and not over-engineered (premature abstraction)
- Err on the side of handling more edge cases, not fewer; thoughtfulness > speed
- Bias toward explicit over clever

---

## Review Stages

### 1. Architecture Review
- System design and component boundaries
- Dependency graph and coupling concerns
- Data flow patterns and potential bottlenecks
- Scaling characteristics and single points of failure
- Security architecture (auth, data scope, API boundaries)

### 2. Code Quality Review
- Code organization and module structure
- DRY violations — be aggressive
- Error handling patterns and missing edge cases (call out explicitly)
- Technical debt hotspots
- Over-engineered or under-engineered areas

### 3. Test Review
- Coverage gaps (unit, integration, e2e)
- Test quality and assertion strength
- Missing edge case coverage
- Untested failure modes and error paths

### 4. Performance Review
- N+1 queries and database access patterns
- Memory usage concerns
- Caching opportunities
- Slow or high-complexity code paths

---

## For Each Issue Found
- Describe the problem concretely, with file and line references
- Present 2–3 options, including "do nothing" where reasonable
- For each option: implementation effort, risk, impact on other code, maintenance burden
- Give recommended option and why, mapped to preferences above
- Ask whether I agree or want a different direction before proceeding

---

## Interaction Rules
- Do not assume priorities on timeline or scale
- After each section, pause and ask for feedback before moving on
