# claude-docs

Claude Code configuration and command library for TypeScript/Node.js/React projects.
Optimized for token efficiency, parallel feature development, and team consistency.

---

## Quickstart

```bash
# Copy into your project root
cp -r .claude/ your-project/
cp CLAUDE.md your-project/
cp -r docs/ your-project/docs/

cd your-project
/project-init    # installs quality gates, GitHub templates, labels
```

Then customize every `[CUSTOMIZE]` section in `CLAUDE.md`.

---

## File Structure

```
claude-docs/
├── CLAUDE.md                        # Project config — Claude reads this every session
├── .claude/
│   ├── settings.json                # Allowed bash commands
│   └── commands/                    # Slash commands — one file = one /command
│       ├── project-init.md          # /project-init
│       ├── prd-to-issues.md         # /prd-to-issues
│       ├── feature-start.md         # /feature-start [N]
│       ├── build-feature.md         # /build-feature [N]
│       ├── verify.md                # /verify
│       ├── code-review.md           # /code-review
│       ├── create-pr.md             # /create-pr
│       ├── pr-prep.md               # /pr-prep [N]
│       ├── human-test.md            # /human-test [N]
│       └── parallel-status.md      # /parallel-status
└── docs/
    ├── plan-mode-prompt.md          # BIG/SMALL review framework (@docs/plan-mode-prompt.md)
    ├── docker-workflow.md           # Compose patterns + environment strategy (@docs/docker-workflow.md)
    ├── db-connection.md             # PgBouncer + Prisma dual-URL setup (@docs/db-connection.md)
    └── db-rollback-process.md       # Migration rollback procedures (@docs/db-rollback-process.md)
```

---

## Command Reference

### Project setup

| Command | When | What |
|---------|------|------|
| `/project-init` | Once per new project | Install Husky/commitlint, create GitHub templates and labels, scaffold CLAUDE.md |
| `/prd-to-issues` | After writing PRD.md | Parse PRD → GitHub milestones + issues with ACs, testing plans, dependency map |

### Feature development loop (repeat per issue)

| Command | When | What |
|---------|------|------|
| `/parallel-status` | Daily / before starting work | Dashboard: active worktrees, CI status, PRs in review, issues ready to start |
| `/feature-start [N]` | Starting a feature | Create worktree + branch + draft PR, load issue context |
| `/build-feature [N]` | Building a feature | Plan → TDD build → acceptance criteria verification → push |
| `/verify` | Anytime / called by build & pr-prep | Lint + typecheck + tests in parallel (backend and frontend) |
| `/human-test [N]` | After build + CI green | Start Docker stack, smoke tests, manual testing checklist |
| `/pr-prep [N]` | After human test passes | Rebase + code-review + create-pr + post-merge cleanup |

### Supporting commands

| Command | When | What |
|---------|------|------|
| `/code-review` | Before PRs / standalone | Pre-PR audit: correctness, error handling, type safety, security, architecture |
| `/create-pr` | Part of pr-prep / standalone | Push branch, write PR body, mark ready for review, assign reviewer |

### Production quality gates

| Command | When | What |
|---------|------|------|
| `/security-review` | Before any PR touching auth, payments, or user data | OWASP Top 10 audit: auth, injection, secrets, CORS, headers, deps |
| `/performance-review` | Before merging data-heavy or list features | N+1 detection, missing indexes, bundle size, React render efficiency |
| `/tech-debt [scope]` | Sprint retrospectives, or as issues accumulate | Code smell audit, complexity hotspots, TODO sweep, ADR creation |
| `/dependency-audit` | Monthly + before every release | npm audit, outdated packages, license compliance, bundle weight |
| `/release-checklist [version]` | After all milestone PRs merged, CI green | Pre-production gate: migrations, env vars, rollback plan, observability, release notes |

---

## Workflow overview

```
Project setup (once)
  /project-init → /prd-to-issues

Per feature (repeat × N, up to 4 in parallel)
  /parallel-status              ← check what's ready
  /feature-start [N]            ← worktree + branch + draft PR
  /build-feature [N]            ← plan → TDD → verify ACs → push
  /security-review              ← if touching auth / data / integrations
  /performance-review           ← if touching queries / lists / bundles
  /human-test [N]               ← docker stack + manual checklist
  /pr-prep [N]                  ← rebase → review → PR → cleanup

Per milestone / release
  /dependency-audit             ← vulnerability + license sweep
  /tech-debt full-codebase      ← debt triage before sprint planning
  /release-checklist [version]  ← production gate → tag → deploy
```

### Parallel development

Run up to 3–4 features simultaneously using git worktrees:

```
/desktop/development/
├── my-project/              ← main (always clean)
├── my-project-feat-42/      ← worktree: feat/42-user-auth
├── my-project-feat-51/      ← worktree: feat/51-dashboard
└── my-project-fix-38/       ← worktree: fix/38-login-bug
```

Rules:
- Only parallelize issues that don't touch the same files
- Rebase against main daily
- Draft PRs open immediately — CI runs from day one
- Use `/parallel-status` as your standup dashboard

---

## Cowork PM skills — where they fit

These Cowork skills handle the planning and communication layers that surround the code workflow:

| Phase | Cowork skill | When to use |
|-------|-------------|-------------|
| **Pre-PRD research** | `product-management:user-research-synthesis` | Synthesize user interviews and support tickets into themes before writing a PRD |
| **PRD creation** | `product-management:feature-spec` | Write a structured PRD with problem statement, user stories, requirements, and success metrics — then hand it to `/prd-to-issues` |
| **Sprint planning** | `product-management:roadmap-management` | Prioritize issues using RICE/MoSCoW, plan the milestone before `/feature-start` |
| **During build** | `product-management:metrics-tracking` | Define the success metrics and OKRs for a feature while it's being built |
| **Post-merge comms** | `product-management:stakeholder-comms` | Draft milestone release notes, executive summaries, and customer-facing announcements after `/release-checklist` completes |
| **Competitive context** | `product-management:competitive-analysis` | Research competitor capabilities when scoping a new feature area |

---

## Reference docs

Load these on demand with `@docs/filename.md` — they stay out of context until needed:

| File | Contents |
|------|----------|
| `docs/plan-mode-prompt.md` | Structured review framework — BIG vs SMALL change, 4-section analysis |
| `docs/docker-workflow.md` | Compose file structure, dev/staging/prod environment strategy |
| `docs/db-connection.md` | PgBouncer + Prisma dual-URL configuration |
| `docs/db-rollback-process.md` | Prisma migration rollback procedures |

---

## Anthropic alignment notes

This library was validated against Anthropic's official Claude Code documentation
(`code.claude.com/docs`) in March 2026. Key alignment points:

**What's confirmed correct:**
- `CLAUDE.md` at repo root is the recommended location for project-level context
- `.claude/commands/*.md` is fully supported (Anthropic notes it's compatible with the newer `.claude/skills/` structure)
- `@path/to/file` import syntax in CLAUDE.md is Anthropic's documented approach
- `settings.json` `permissions.allow/deny` with `Bash(command *)` format is correct
- CLAUDE.md at 196 lines is just under Anthropic's recommended 200-line limit
- Git worktrees are Anthropic's own recommended approach for parallel sessions

**What was corrected after doc review:**
- Added **YAML frontmatter** (`name`, `description`, `argument-hint`) to all 10 commands — this is how Claude knows when to offer or auto-invoke a skill
- Added **`disable-model-invocation: true`** to 8 workflow commands (feature-start, build-feature, pr-prep, human-test, etc.) — prevents Claude from running side-effect commands automatically
- Replaced `[issue-number]` placeholders with **`$ARGUMENTS`** — Anthropic's native substitution mechanism

**Three additional patterns from Anthropic docs worth knowing:**

1. **`.claude/skills/`** — The current preferred structure over `.claude/commands/`. Each skill becomes a directory (`deploy/SKILL.md`) and can bundle supporting files alongside it. Migrate when your commands need multi-file supporting assets.
   ```
   .claude/skills/
   └── feature-start/
       ├── SKILL.md          ← the command
       └── templates/        ← supporting files
   ```

2. **`.claude/rules/`** — Path-scoped rules that only load into context for matching files. Useful for monorepos (e.g., frontend-specific rules only load when editing React files).
   ```yaml
   # .claude/rules/api.md
   ---
   paths:
     - "src/api/**/*.ts"
   ---
   All API endpoints must validate input with Zod...
   ```

3. **Auto memory** (`v2.1.59+`) — Claude writes its own notes between sessions (debugging insights, patterns it discovers). Stored in `~/.claude/projects/<repo>/memory/`. View and manage with `/memory` in Claude Code. Complements CLAUDE.md — you write CLAUDE.md, Claude writes auto memory.

---

## Design principles

- **Token efficiency** — CLAUDE.md stays lean; reference docs load only when needed
- **Concrete over generic** — every rule has a reason; every example is real
- **Plan before build** — no code written without an approved plan
- **Test first** — failing tests before implementation, every time
- **Parallel by default** — worktrees + draft PRs make multi-track work safe
- **Commands as institutional memory** — team patterns live in `.claude/commands/`, not in people's heads
