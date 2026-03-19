---
name: create-pr
description: Creates or finalizes a pull request — writes a quality PR body, marks it ready for review, and assigns a reviewer. Called as part of /pr-prep. Also triggers on "open a PR", "submit this for review", "create pull request", or "make the PR".
disable-model-invocation: true
---

# /create-pr — Create a Pull Request

Run after `/code-review` passes and you're ready to submit for human review.
Called as part of `/pr-prep`. Can also be run standalone.

---

## Step 1 — Understand all changes

```bash
git log main..HEAD --oneline
git diff main...HEAD --stat
```

---

## Step 2 — Push the branch

```bash
git push
```

If the branch has already been pushed (as a draft PR), just ensure it's up to date.

---

## Step 3 — Create or update the PR

If a **draft PR already exists** (created by `/feature-start`), update it and mark ready:
```bash
gh pr edit --title "feat([scope]): [description under 70 chars]" \
  --body "$(cat <<'EOF'
## What does this PR do?

[2-3 sentences describing the feature and its user-facing value.]

Closes #[issue-number]

## Changes

### New files
- `[path]` — [purpose]

### Modified files
- `[path]` — [what changed and why]

## Quality gates
- [x] /verify passes (lint + typecheck + tests)
- [x] /security-review run — or N/A (reason: _______________)
- [x] /performance-review run — or N/A (reason: _______________)
- [x] /human-test completed
- [x] All acceptance criteria met
EOF
)"

gh pr ready
gh pr edit --remove-label "in-progress" --add-label "ready-for-review"
```

If **no PR exists yet**, create it:
```bash
gh pr create \
  --title "feat([scope]): [description under 70 chars]" \
  --body "[same body as above]" \
  --label "ready-for-review"
```

---

## Step 4 — Assign a reviewer

```bash
gh pr edit --add-reviewer [github-username]
```

---

## Step 5 — Return the PR URL

Report the PR URL so the user can share it or monitor it directly.
