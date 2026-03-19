# Developer Onboarding Guide
> Read this on day one. You should be running your first `/feature-start` by end of day.

---

## Prerequisites

Install these before anything else:

```bash
# Node.js (v20+)
node --version   # must be 20.x or higher

# Claude Code
npm install -g @anthropic-ai/claude-code
claude --version

# GitHub CLI
brew install gh
gh auth login

# Docker Desktop
# Download from https://www.docker.com/products/docker-desktop/
docker --version

# Verify all tools
node --version && claude --version && gh --version && docker --version
```

---

## Day 1 Checklist

- [ ] Clone the project repo and run `npm install` from the root
- [ ] Copy `.env.example` to `.env` and fill in your local values (ask the team lead for any secrets)
- [ ] Run `npm run dev` — confirm the dev server starts without errors
- [ ] Run `npm run test` — confirm all tests pass on main
- [ ] Open a Claude Code session: `cd [project-root] && claude`
- [ ] Run `/parallel-status` — familiarize yourself with what's in flight
- [ ] Read `CLAUDE.md` in full — this is Claude's project context, it's also yours
- [ ] Ask the team lead to assign you a "good first issue" labeled `onboarding`

---

## Starting a Claude Code Session

Every feature runs inside its own Claude Code session in its own worktree directory.

```bash
# Always start from the project root (main branch, always clean)
cd /desktop/development/[project-name]
claude

# Claude reads CLAUDE.md automatically — you're ready to work
# Run your first command:
/parallel-status
```

---

## Your First Feature

```bash
# Pick an issue number from /parallel-status or GitHub
/feature-start [issue-number]

# Claude will:
# 1. Create a worktree at ../[project]-feat-[N]/
# 2. Create branch feat/[N]-[title]
# 3. Open a draft PR
# 4. Load the issue context

# Open VS Code in the new worktree directory
code ../[project]-feat-[N]/

# Start a Claude session in that directory
cd ../[project]-feat-[N]/
claude

# Build the feature
/build-feature [issue-number]
```

---

## The Daily Rhythm

```
Morning
  → /parallel-status          check what's in flight, what's blocked, what's ready
  → Pick up your worktree     cd ../[project]-feat-[N]/ && claude

During the day
  → /build-feature            Claude plans, writes tests, implements
  → /verify                   run this any time you want a quality check
  → git push                  push often — CI runs on every push

When done building
  → /security-review          if touching auth, data, or integrations
  → /performance-review       if touching queries, lists, or bundles
  → /human-test [N]           start Docker stack, walk through the checklist
  → /pr-prep [N]              rebase, review, open PR for human approval

End of day
  → Push your branch
  → Update the draft PR body with where you left off
  → Run /parallel-status to see the overall state
```

---

## Parallel Work Rules

You can work on multiple features simultaneously. Each gets its own worktree.

```
/desktop/development/
├── [project]/              ← main — never work here directly
├── [project]-feat-42/      ← your feature
└── [project]-feat-51/      ← another feature (if applicable)
```

Hard rules:
- Never work directly on `main` — always in a worktree
- Max 2 active worktrees per developer (team cap is 3–4 total)
- No new `/feature-start` if you already have 2 PRs awaiting review — clear the queue first
- Rebase your branches against main daily: `git fetch origin main && git rebase origin/main`

---

## Commit Format

We use Conventional Commits. commitlint enforces this automatically.

```bash
# Valid formats
feat(auth): add refresh token rotation
fix(dashboard): correct chart data aggregation
chore(deps): bump typescript to 5.4
docs(api): update endpoint reference

# Invalid — will be rejected by the pre-commit hook
updated stuff
WIP
fix
```

---

## When Something Goes Wrong

**Tests failing on main?** Don't pull and ignore — tell the team in Slack immediately. Broken main blocks everyone.

**Merge conflict you're unsure about?** Ask before guessing. Guessing at intent in a conflict is how bugs get introduced.

**Something broken in production?** See `@docs/incident-response.md` — follow it exactly. First 5 minutes matter.

**Unsure about an architectural decision?** Check `docs/adr/` for existing decisions. If there's no ADR covering it, ask before building — don't invent patterns.

---

## Who to Ask

| Question | Ask |
|----------|-----|
| Project architecture, design decisions | Team lead |
| Client requirements, acceptance criteria | Project manager / team lead |
| Stuck on a bug for > 30 mins | Post in Slack with context, don't silently spin |
| Security or deployment questions | Team lead only — never improvise here |

---

## Next Steps

Once you've shipped your first feature through `/pr-prep`, you're fully onboarded.
Ask your team lead to remove the `onboarding` label from your issue and assign you to the regular sprint backlog.
