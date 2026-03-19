---
name: project-init
description: One-time project setup. Installs quality gates (Husky, lint-staged, commitlint), creates GitHub issue/PR templates and labels, and scaffolds CLAUDE.md from the team template. Use when starting a new project or repo.
disable-model-invocation: true
---

# /project-init — One-Time Project Setup

Run this once when starting a new project from the claude-docs template. It installs
quality gates, creates GitHub templates and labels, and scaffolds CLAUDE.md so every
Claude Code session starts with full context.

---

## Step 1 — Confirm project details

Collect (or infer from existing files):
- Project name and one-line description
- GitHub repo URL
- Deployment target (Railway, Vercel, etc.)
- Any architecture decisions already made

---

## Step 2 — Customize CLAUDE.md

Open `CLAUDE.md` and replace every `[CUSTOMIZE]` section with project specifics:
- What the app does and who uses it
- Actual deploy targets
- Real npm scripts (verify with `cat package.json`)
- Dependency preferences (email provider, validation library, HTTP client, etc.)
- Known gotchas (add these as you discover them throughout the project)

---

## Step 3 — Install quality gates

```bash
npm install --save-dev husky lint-staged @commitlint/cli @commitlint/config-conventional
npx husky init
```

Create `.husky/pre-commit`:
```sh
#!/usr/bin/env sh
npx lint-staged
```

Create `.husky/commit-msg`:
```sh
#!/usr/bin/env sh
npx --no -- commitlint --edit $1
```

Add to `package.json`:
```json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{js,json,md}": ["prettier --write"]
  },
  "commitlint": {
    "extends": ["@commitlint/config-conventional"]
  }
}
```

---

## Step 4 — GitHub templates

Create `.github/PULL_REQUEST_TEMPLATE.md`:
```markdown
## What does this PR do?

Closes #[ISSUE_NUMBER]

## Changes
-

## Testing
- [ ] `/verify` passes (lint + typecheck + tests)
- [ ] `/human-test` completed
- [ ] All acceptance criteria met

## Notes
```

Create `.github/ISSUE_TEMPLATE/feature.md`:
```markdown
---
name: Feature
about: New feature or enhancement
labels: feat
---

## Goal

## Acceptance Criteria
- [ ]

## Technical Notes

## Testing Plan
### Unit tests
- [ ]
### Manual testing
- [ ]

## Depends on
- #
```

---

## Step 5 — GitHub labels

```bash
gh label create "feat" --color "0075ca" --description "New feature"
gh label create "fix" --color "d73a4a" --description "Bug fix"
gh label create "chore" --color "e4e669" --description "Maintenance"
gh label create "blocked" --color "b60205" --description "Blocked by dependency"
gh label create "ready-for-review" --color "0e8a16" --description "Ready for human review"
gh label create "in-progress" --color "fbca04" --description "Actively being worked on"
```

---

## Step 6 — Validate setup

```bash
git add .
git commit -m "chore: initialize claude-docs workflow"
```

The pre-commit hook should run and pass. If it fails, fix the issue before proceeding.

---

## Done — next steps

1. Write `PRD.md` and `ARCHITECTURE.md` and commit them to `/docs`
2. Run `/prd-to-issues` to generate GitHub milestones and issues from the PRD
3. Run `/parallel-status` to see what's ready to start
