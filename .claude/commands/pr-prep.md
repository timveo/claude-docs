---
name: pr-prep
description: Full pre-merge workflow for a GitHub issue — rebase, /verify, acceptance criteria audit, /code-review, /create-pr, and post-merge cleanup. Run after /human-test passes. Triggers on "ready to merge", "prepare the PR", "merge this", or "finalize #N".
disable-model-invocation: true
argument-hint: <issue-number>
---

# /pr-prep — Prepare PR for Merge

Full pre-merge workflow: rebase, quality suite, acceptance criteria audit, code review,
PR creation, and post-merge cleanup. Run this after `/human-test` passes.

**Usage:** `/pr-prep 42`

This command orchestrates: `/verify` → `/code-review` → `/create-pr` → cleanup.

---

## Step 1 — Sync with main

```bash
git fetch origin main
git rebase origin/main
```

If there are conflicts, resolve them carefully. For any non-obvious conflict, ask the
user for guidance rather than guessing at intent.

---

## Step 2 — Full quality suite

```bash
/verify
```

All checks must be green before proceeding. Fix anything failing.

---

## Step 3 — Acceptance criteria audit

```bash
gh issue view $ARGUMENTS
```

Go through every acceptance criterion and confirm it is satisfied:

```
Final Acceptance Criteria Audit — Issue #[N]:
✅ [Criterion 1] — [test name or observable behavior]
✅ [Criterion 2] — [test name or observable behavior]
✅ TypeScript: no errors
✅ All tests passing
```

If any criterion is unmet, fix it now. Do not proceed with an unmet criterion.

---

## Step 4 — Code review

```bash
/code-review
```

Address any Critical or Warning findings before marking the PR ready.

---

## Step 5 — Create / finalize PR

```bash
/create-pr
```

---

## Step 6 — Wait for approval and merge

CI will run. Once a reviewer approves and all checks are green, the PR can be merged.

Merge strategy: **squash merge** — keeps main history clean, one commit per feature.

---

## Step 7 — Post-merge cleanup

Once merged:

```bash
# Remove the worktree (run from the main repo directory, not the worktree)
git worktree remove [worktree-path]
git branch -d [branch-name]
```

Close the issue if not auto-closed by "Closes #N":
```bash
gh issue close $ARGUMENTS \
  --comment "Completed in PR #[pr-number] — merged to main."
```

Check if the milestone is now complete:
```bash
gh issue list --milestone "[milestone-name]" --state open
```

If all issues are closed:
```bash
gh api repos/:owner/:repo/milestones/[milestone-number] \
  --method PATCH \
  --field state="closed"
```

---

## Step 8 — What's next

```bash
/parallel-status
```

See what other features are in progress, what's ready to start, and what needs attention.
