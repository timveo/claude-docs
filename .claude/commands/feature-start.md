---
name: feature-start
description: Starts a feature in an isolated git worktree — creates the branch, opens a draft PR for early CI feedback, and loads issue context. Use at the beginning of every new feature or bug fix. Triggers on "start feature", "begin issue", "work on #N", or "start a new branch".
disable-model-invocation: true
argument-hint: <issue-number>
---

# /feature-start — Start a Feature in an Isolated Worktree

Run this at the beginning of every feature. Creates a git worktree (isolated working
directory on its own branch), opens a draft PR immediately for early CI feedback, and
loads issue context so Claude is fully oriented before writing a line of code.

**Usage:** `/feature-start 42`

---

## Why worktrees?

A git worktree lets you work on multiple branches in separate directories simultaneously —
no stashing, no context-switching, no disrupting other in-progress work.

```
/desktop/development/
├── my-project/              ← main branch (always clean)
├── my-project-feat-42/      ← feat/42-user-auth  (worktree)
├── my-project-feat-51/      ← feat/51-dashboard  (worktree)
└── my-project-fix-38/       ← fix/38-login-bug   (worktree)
```

Each worktree is independent — you can run `npm run dev` in each one at the same time.

---

## Step 1 — Read the issue

```bash
gh issue view $ARGUMENTS
```

Read the full issue: goal, acceptance criteria, technical notes, testing plan, and dependencies.

If a dependency issue is listed, check whether it has merged:
```bash
gh issue view [dep-number] --json state --jq '.state'
```

If it's still open, warn the user and ask whether to proceed anyway.

---

## Step 2 — Create the worktree

From the repo root:
```bash
git fetch origin main

BRANCH_NAME="feat/$ARGUMENTS-$(gh issue view $ARGUMENTS --json title --jq '.title' \
  | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | cut -c1-40)"

WORKTREE_DIR="../$(basename $(pwd))-feat-$ARGUMENTS"

git worktree add "$WORKTREE_DIR" -b "$BRANCH_NAME" origin/main
```

Report the worktree path to the user.

---

## Step 3 — Install dependencies

```bash
cd "$WORKTREE_DIR" && npm install
```

---

## Step 4 — Open a draft PR immediately

Open the PR now, before any code is written. This starts CI baseline checks early and
flags environment issues before you're deep in the build.

```bash
cd "$WORKTREE_DIR"
git push -u origin "$BRANCH_NAME"

gh pr create \
  --draft \
  --title "feat: [issue title] (#$ARGUMENTS)" \
  --body "## What does this PR do?

Closes #$ARGUMENTS

## Status
🚧 Work in progress

## Changes
(to be filled in when ready for review)

## Acceptance criteria
(to be verified when ready)" \
  --label "in-progress"
```

Share the PR URL with the user.

---

## Step 5 — Assign and label the issue

```bash
gh issue edit $ARGUMENTS --add-assignee "@me" --add-label "in-progress"
```

---

## Step 6 — Orientation summary

```
✅ Feature #[N] ready to build

Worktree:   [path]
Branch:     [branch-name]
Draft PR:   [PR URL]

GOAL: [one sentence from issue]

Acceptance Criteria:
  • [criterion 1]
  • [criterion 2]
  ...

Technical notes: [from issue]

Next step: open VS Code in [worktree path] and run /build-feature $ARGUMENTS
```

---

## Step 7 — Open the worktree in VS Code

Tell the user:
```
Open this feature in a new VS Code window:
  cd [worktree path]
  code .

Start a Claude Code session there — CLAUDE.md will load automatically.
Run: /build-feature $ARGUMENTS
```
