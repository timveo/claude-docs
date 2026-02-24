# Claude Docs

Claude Code configuration files and reference docs for development projects.

## Quick Start
1. Copy `CLAUDE.md` into the root of your project
2. Run through the **Customization Checklist** below
3. Copy any relevant `/docs/` files into your project's `/docs/` folder
4. Reference docs on demand in Claude Code with `@docs/filename.md`

## Customization Checklist

After copying `CLAUDE.md` into a project, search for `[CUSTOMIZE]` and update:

- [ ] **Project description** — What does the app do? Who uses it?
- [ ] **Deploy target** — Where does this deploy? (Railway, Vercel, AWS, etc.)
- [ ] **Commands** — Replace with your actual `package.json` scripts and working directories
- [ ] **Architecture tree** — Update folder structure to match your project
- [ ] **Dependency Preferences** — Uncomment and list libraries Claude should use (and avoid)
- [ ] **Known Gotchas** — Uncomment and add project-specific footguns as you discover them

### Optional: Remove sections that don't apply
- No Docker? Delete the Docker section
- No PgBouncer? Don't copy `docs/db-connection.md`
- No background jobs? Remove the `/workers/` line from the architecture tree

## Files
| File | Purpose |
|------|---------|
| `CLAUDE.md` | Main Claude Code configuration — copy to project root |
| `docs/plan-mode-prompt.md` | Garry Tan's structured code review framework |
| `docs/db-connection.md` | PgBouncer + Prisma dual-URL setup |
| `docs/db-rollback-process.md` | Prisma migration rollback procedures |
| `docs/docker-workflow.md` | Docker compose workflow reference |

## Design Principles

This template is optimized for **token efficiency** — CLAUDE.md is injected into every Claude Code conversation, so every line must earn its place:

- **Project-specific > generic** — Rules Claude can infer from config files (tsconfig, eslint) are omitted
- **Concrete > philosophical** — Actionable instructions over vague principles
- **No duplication** — Rules already built into Claude Code (don't commit secrets, confirm before deleting files, etc.) are not repeated
- **On-demand depth** — Deep references live in `/docs/` files loaded with `@docs/`, keeping the main file lean
