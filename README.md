# claude-docs — ARCHIVED

**This repository is no longer the source of anything.** As of 1 August 2026 it is read-only.

## Why

These 17 skills were one of **three divergent copies** of the same library — the others were
`impact-os/company/coding/commands` (15, in an older format) and a vendored copy inside
`Converge-NPS`. Each had behaviour the others lacked. `/pr-prep` existed only here and was
referenced by name in repos that did not have it; `/clean-worktree` existed only in impact-os and
was referenced by skills that lived here. Both were dangling references in production repos.

A fix to `/security-review` had to be made three times, and usually wasn't.

## What replaces it

**The `code-impact-cc` plugin.** A skill exists exactly once. Fixing it once fixes it in
Converge-NPS, Early-Alert and CI-Customer-Training at the same time — which is the whole point.

```bash
/plugin marketplace add timveo/code-impact-cc
/plugin install code-impact-cc@code-impact
```

## Where things went

| Was here | Now |
|---|---|
| The 17 skills | the plugin — merged with impact-os's versions, defects fixed |
| `CLAUDE.md` template | `templates/CLAUDE.md` in the plugin, under 200 lines, no `@imports` |
| `docs/*.md` reference files | `.claude/rules/` with `paths:` frontmatter — they load only when Claude touches a matching file, instead of at launch |
| `pr-prep` | split: gate applicability → `gate-status`, rebase → `bin/rebase`, branch delete → `retire-worktree` |

## Do not

Copy anything out of here. If something is missing from the plugin, add it to the plugin.

Process: the Code Impact handbook, https://github.com/codeimpact-ai/code-impact-handbook.
