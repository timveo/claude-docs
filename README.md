# claude-docs

Claude Code configuration and command library for TypeScript/Node.js/React projects.
Built for coding agencies and product teams — optimized for velocity, production quality,
parallel development, and professional client delivery.

---

## Quickstart

```bash
# Copy into your project root
cp -r .claude/ your-project/      # includes skills/ directory with all 17 commands
cp CLAUDE.md your-project/
cp -r docs/ your-project/docs/

cd your-project
/project-init    # installs quality gates, GitHub templates, labels
```

Then customize every `[CUSTOMIZE]` section in `CLAUDE.md`.
New developers start at `docs/ONBOARDING.md`.

---

## File Structure

```
claude-docs/
├── CLAUDE.md                        # Project config — Claude reads this every session
├── .claude/
│   ├── settings.json                # Allowed bash commands
│   └── skills/                      # Slash commands — each is a directory with SKILL.md
│       ├── build-feature/
│       │   └── SKILL.md             # /build-feature [N]
│       ├── client-handoff/
│       │   └── SKILL.md             # /client-handoff [milestone]
│       ├── code-review/
│       │   └── SKILL.md             # /code-review
│       ├── create-pr/
│       │   └── SKILL.md             # /create-pr
│       ├── dependency-audit/
│       │   └── SKILL.md             # /dependency-audit
│       ├── feature-start/
│       │   └── SKILL.md             # /feature-start [N]
│       ├── human-test/
│       │   └── SKILL.md             # /human-test [N]
│       ├── incident-response/
│       │   └── SKILL.md             # /incident-response [live|post-mortem]
│       ├── parallel-status/
│       │   └── SKILL.md             # /parallel-status
│       ├── performance-review/
│       │   └── SKILL.md             # /performance-review
│       ├── pr-prep/
│       │   └── SKILL.md             # /pr-prep [N]
│       ├── prd-to-issues/
│       │   └── SKILL.md             # /prd-to-issues [path]
│       ├── project-init/
│       │   └── SKILL.md             # /project-init
│       ├── release-checklist/
│       │   └── SKILL.md             # /release-checklist [version]
│       ├── security-review/
│       │   └── SKILL.md             # /security-review
│       ├── tech-debt/
│       │   └── SKILL.md             # /tech-debt [scope]
│       └── verify/
│           └── SKILL.md             # /verify
└── docs/
    ├── ONBOARDING.md                # New developer setup and day-one checklist
    ├── plan-mode-prompt.md          # BIG/SMALL review framework
    ├── client-handoff.md            # Client delivery process reference
    ├── incident-response.md         # Production incident runbook
    ├── docker-workflow.md           # Compose patterns + environment strategy
    ├── db-connection.md             # PgBouncer + Prisma dual-URL setup
    └── db-rollback-process.md       # Migration rollback procedures
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
| `/pr-prep [N]` | After human test passes | Rebase + quality gates + code-review + create-pr + cleanup |

### Supporting commands

| Command | When | What |
|---------|------|------|
| `/code-review` | Before PRs / standalone | Pre-PR audit: correctness, error handling, type safety, security, architecture |
| `/create-pr` | Part of pr-prep / standalone | Push branch, write PR body, mark ready for review, assign reviewer |

### Production quality gates

| Command | When | What |
|---------|------|------|
| `/security-review` | **Required** for any PR touching auth, payments, user data, or integrations | OWASP Top 10 audit: auth, injection, secrets, CORS, headers, deps |
| `/performance-review` | **Required** for any PR touching queries, lists, bundles, or background jobs | N+1 detection, missing indexes, bundle size, React render efficiency |
| `/tech-debt [scope]` | Sprint retrospectives, or as issues accumulate | Code smell audit, complexity hotspots, TODO sweep, ADR creation |
| `/dependency-audit` | Monthly + before every release | npm audit, outdated packages, license compliance, bundle weight |
| `/release-checklist [version]` | After all milestone PRs merged, CI green | Pre-production gate: migrations, env vars, rollback plan, client sign-off |

### Client delivery and incidents

| Command | When | What |
|---------|------|------|
| `/client-handoff [milestone]` | After `/release-checklist` passes | Staging walkthrough → client access → feedback loop → go-live approval → notify |
| `/incident-response live` | Something breaks in production | Rollback decision, client communication, incident log |
| `/incident-response post-mortem` | After incident resolved | Timeline, root cause, action items, client communication |


---

## Workflow overview

```
Project setup (once)
  /project-init → /prd-to-issues
  New developers → docs/ONBOARDING.md

Per feature (repeat × N, up to 4 in parallel)
  /parallel-status              ← check what's ready (hard cap: no new start if 2+ PRs await review)
  /feature-start [N]            ← worktree + branch + draft PR
  /build-feature [N]            ← plan → TDD → verify ACs → push
  /security-review              ← REQUIRED if touching auth / data / integrations
  /performance-review           ← REQUIRED if touching queries / lists / bundles
  /human-test [N]               ← docker stack + manual checklist
  /pr-prep [N]                  ← rebase → quality gates → review → PR → cleanup

Per milestone / release
  /dependency-audit             ← vulnerability + license sweep
  /tech-debt full-codebase      ← debt triage before sprint planning
  /release-checklist [version]  ← production gate → tag → deploy
  /client-handoff [milestone]   ← staging → client review → go-live approval → notify

When things go wrong
  /incident-response live       ← rollback decision + client comms
  /incident-response post-mortem ← root cause + action items
```

---

## Parallel development

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
- **Hard cap: no new `/feature-start` if 2+ PRs are already awaiting review**
- Use `/parallel-status` as your standup dashboard

---

## Reference docs

Load these on demand with `@docs/filename.md` — they stay out of context until needed:

| File | Contents |
|------|----------|
| `docs/ONBOARDING.md` | New developer setup, day-one checklist, daily rhythm |
| `docs/plan-mode-prompt.md` | Structured review framework — BIG vs SMALL change, 4-section analysis |
| `docs/client-handoff.md` | Client delivery process, email templates, feedback classification |
| `docs/incident-response.md` | Production incident runbook, post-mortem template, severity levels |
| `docs/docker-workflow.md` | Compose file structure, dev/staging/prod environment strategy |
| `docs/db-connection.md` | PgBouncer + Prisma dual-URL configuration |
| `docs/db-rollback-process.md` | Prisma migration rollback procedures |

---

## Anthropic alignment notes

This library was validated against Anthropic's official Claude Code documentation
(`code.claude.com/docs`) in March 2026. Key alignment points:

- `CLAUDE.md` at repo root is the recommended location for project-level context
- `.claude/skills/[name]/SKILL.md` is Anthropic's current preferred structure — each skill is a directory, enabling bundled supporting files alongside the command
- `@path/to/file` import syntax in CLAUDE.md is Anthropic's documented approach
- `settings.json` `permissions.allow/deny` with `Bash(command *)` format is correct
- Git worktrees are Anthropic's own recommended approach for parallel sessions
- `$ARGUMENTS` is Anthropic's native substitution mechanism for command arguments

**Three advanced patterns worth knowing:**

1. **`.claude/skills/`** — The current preferred structure, used throughout this repo. Each skill is a directory (`feature-start/SKILL.md`) that can bundle supporting files alongside the command. Claude Code discovers all `SKILL.md` files automatically.

2. **`.claude/rules/`** — Path-scoped rules that only load for matching files. Useful for monorepos (frontend-specific rules only load when editing React files).

3. **Auto memory** (`v2.1.59+`) — Claude writes its own notes between sessions. Stored in `~/.claude/projects/<repo>/memory/`. View with `/memory` in Claude Code.

---

## Design principles

- **Token efficiency** — CLAUDE.md stays lean; reference docs load only when needed
- **Concrete over generic** — every rule has a reason; every example is real
- **Plan before build** — no code written without an approved plan
- **Test first** — failing tests before implementation, every time
- **Parallel by default** — worktrees + draft PRs make multi-track work safe
- **Finish before starting** — no new features while PRs are waiting for review
- **Skills as institutional memory** — team patterns live in `.claude/skills/`, not in people's heads
- **Client delivery is part of the workflow** — shipping to a client is as structured as shipping code
