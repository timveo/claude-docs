---
name: parallel-status
description: Shows a real-time dashboard of all active git worktrees, CI status, open PRs, issues ready to start, and rebase warnings. Use as a daily standup replacement or whenever you want to see what's in flight. Triggers on "what's in progress", "show me the status", "what can I start", or "standup".
disable-model-invocation: true
---

# /parallel-status — Parallel Work Dashboard

Run anytime — especially as a daily standup replacement — to see all active worktrees,
CI status, PRs in review, issues ready to start, and rebase warnings.

---

## Step 1 — Active worktrees

```bash
git worktree list
```

---

## Step 2 — Open PRs and CI status

```bash
gh pr list --state open \
  --json number,title,headRefName,isDraft,labels,checksState \
  --jq '.[] | "\(.number) [\(if .isDraft then "DRAFT" else "OPEN" end)] \(.title) — CI: \(.checksState)"'
```

Classify:
- 🟢 CI passing, not draft → **Ready for review**
- 🟡 CI pending → **Waiting on CI**
- 🔴 CI failing → **Needs attention**
- ⚫ Draft, CI not started → **In progress**

---

## Step 3 — Issues ready to start

```bash
gh issue list --state open --label "feat" --json number,title,body
```

For each issue listing a "Depends on #N", check:
```bash
gh issue view [N] --json state --jq '.state'
```

List which issues are **immediately startable** (no open dependencies).

---

## Step 4 — Rebase health

```bash
git fetch origin main
for branch in $(git branch -r | grep -v "main\|HEAD"); do
  BEHIND=$(git rev-list --count $branch..origin/main 2>/dev/null || echo 0)
  if [ "$BEHIND" -gt "5" ]; then
    echo "$branch is $BEHIND commits behind main — rebase recommended"
  fi
done
```

---

## Step 5 — Output dashboard

```
PARALLEL STATUS DASHBOARD — [date]
────────────────────────────────────────
Active worktrees: [N]  |  Open PRs: [N]  |  Issues ready to start: [N]

⚠️  CAPACITY CHECK: If 2+ PRs are awaiting review, clear them before starting new work.

IN PROGRESS:
  🔨 #42 "Add user auth"  →  feat/42-user-auth
     PR #15 (draft)  CI: ✅ passing

  🔨 #51 "Build dashboard"  →  feat/51-dashboard
     PR #16 (draft)  CI: ⏳ running

READY FOR REVIEW:
  👁  #38 "Fix login redirect"  →  fix/38-login-bug
     PR #14  CI: ✅  assigned to @reviewer

READY TO START (no dependencies):
  ▶  #55 "Add export feature"
  ▶  #56 "Notifications system"

BLOCKED:
  ⏳ #60 "Admin panel" — waiting on #42 and #43

NEEDS ATTENTION:
  ❌ feat/44-something is 12 commits behind main — rebase now

────────────────────────────────────────
Suggested next actions:
  1. Review PR #14 — green and waiting
  2. Start #55 or #56 in a new worktree (use /feature-start)
  3. Rebase feat/44-something against main
```

---

## Capacity guidance

Hard cap: **no new `/feature-start` if 2+ PRs are already awaiting review**.
Finishing is more valuable than starting — a review backlog kills velocity faster than anything else.

For a team of 2–6, the optimal number of parallel tracks is **3–4 maximum**.

A healthy state:
- 1–2 features actively being built
- 1 PR in human review
- 1 recently merged, monitoring CI/production

If more than 4 PRs are open simultaneously, flag it:
"You have [N] open PRs. Review and merge existing work before opening new tracks."
