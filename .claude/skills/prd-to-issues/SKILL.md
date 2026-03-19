---
name: prd-to-issues
description: Converts a PRD document into GitHub milestones and issues with full acceptance criteria, testing plans, and a dependency map. Use after writing a PRD or when creating a new sprint. Triggers on "convert PRD", "create issues from", "generate milestones", or any mention of turning a spec into GitHub issues.
disable-model-invocation: true
argument-hint: [path/to/PRD.md or URL]
---

# /prd-to-issues — Convert PRD into GitHub Milestones & Issues

Run this after writing `docs/PRD.md`. It reads the PRD, proposes a milestone structure,
and creates fully-specified GitHub issues with acceptance criteria, technical notes,
testing plans, and dependency mapping — so every Claude Code session has a clear target.

Good issues are the highest-leverage investment in this workflow. Vague issues produce
vague code. Specific, testable acceptance criteria produce verifiable results.

---

## Inputs

Provide one of:
- `@docs/PRD.md` — reference the PRD file directly
- A URL to a doc or GitHub issue
- Pasted PRD text

If nothing is provided, ask: "Please share your PRD — paste it here, give me the file path, or a URL."

---

## Step 1 — Read and understand

Read the full PRD. Identify:
- The primary problem being solved
- Major capability areas → these become **Milestones**
- Specific deliverables within each area → these become **Issues**
- Implicit dependencies between deliverables

If anything is underspecified, ask before creating issues. Don't create issues for things
that are too vague to build against.

---

## Step 2 — Propose milestone structure

Present the structure before creating anything:

```
Proposed Milestones:
1. [Name] — [one-sentence description] (~N issues)
2. [Name] — [one-sentence description] (~N issues)

Does this look right? Any changes before I create the issues?
```

Wait for confirmation.

---

## Step 3 — Create milestones

```bash
gh api repos/:owner/:repo/milestones \
  --method POST \
  --field title="[Milestone Name]" \
  --field description="[Description]"
```

---

## Step 4 — Create issues

Use this template for every issue. Be specific — Claude will use these as the primary
build context, so every acceptance criterion should be independently verifiable.

```bash
gh issue create \
  --title "[verb] [what]: [specific outcome]" \
  --label "feat" \
  --milestone "[Milestone Name]" \
  --body "$(cat <<'EOF'
## Goal
[What are we building and why? Include user-facing impact.]

## Acceptance Criteria
- [ ] [Observable behavior — not implementation detail]
- [ ] [Another testable criterion]
- [ ] Error states handled: [describe expected behavior]
- [ ] TypeScript compiles with no errors
- [ ] All unit tests pass

## Technical Notes
[Patterns to follow, constraints, files likely affected. Reference ARCHITECTURE.md if relevant.]
- Follow: [pattern]
- Touches: [files/modules]
- Do not: [anti-pattern to avoid]

## Testing Plan
### Unit tests
- [ ] [What to test]

### Manual testing
- [ ] [Step-by-step verification]

## Depends on
- #[issue number if applicable]

## Out of scope
- [Explicit exclusions]
EOF
)"
```

**Issue title format:** Action verbs — "Add user authentication", "Build orders API endpoint".
Never vague titles like "Auth work" or "API stuff".

---

## Step 5 — Produce dependency map

After all issues are created, output:

```
Dependency Map:
  #N depends on → #M
  #N depends on → #P

Immediately startable (no dependencies):
  → #X, #Y, #Z — can run in parallel

Sequenced (waiting on others):
  → #A after #M merges
  → #B after #X and #Y merge
```

---

## Step 6 — Summary

Report:
- Milestones created
- Issues created per milestone
- Which issues are immediately startable
- Suggested opening move: "Start issues #X, #Y, #Z in parallel using `/feature-start`"
