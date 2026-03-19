---
name: dependency-audit
description: Audits npm dependencies for security vulnerabilities, outdated packages, license compliance risks, and unnecessary bundle weight. Run monthly or before any production release. Triggers on "dependency audit", "npm audit", "check dependencies", "outdated packages", or "license check".
disable-model-invocation: true
---

# /dependency-audit — Dependency Health Check

Run monthly as routine hygiene, or before any production release. Covers security
vulnerabilities, version staleness, license compliance, and bundle weight.

---

## Step 1 — Security vulnerability scan

```bash
npm audit --audit-level=info 2>&1
```

Classify findings:

| Severity | Action |
|---|---|
| **critical** | Fix immediately — do not deploy |
| **high** | Fix before next release |
| **moderate** | Fix within 2 sprints or document accepted risk |
| **low** | Backlog item |

Attempt automatic fixes first:
```bash
npm audit fix --dry-run 2>&1
# If safe, apply:
npm audit fix
```

For breaking upgrades: `npm audit fix --force --dry-run 2>&1`
Do NOT run `--force` without reviewing the dry-run output.

---

## Step 2 — Outdated package review

```bash
npm outdated 2>&1
```

- **Patch** (1.2.3 → 1.2.4): Apply without review — bug fixes only.
- **Minor** (1.2.0 → 1.3.0): Review changelogs — usually safe.
- **Major** (1.x → 2.x): Treat as a migration task — create a GitHub issue.

```bash
npx npm-check-updates --target patch --upgrade
npm install
```

---

## Step 3 — License compliance check

```bash
npx license-checker --onlyAllow \
  "MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0;CC0-1.0;Unlicense;0BSD" \
  --excludePrivatePackages 2>&1 | head -40
```

**Acceptable:** MIT, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, CC0-1.0, Unlicense

**Requires review:** GPL, LGPL, AGPL, CC-BY-SA, Unknown/UNLICENSED

Flag any non-permissive license as **Critical** if it's a production dependency.

---

## Step 4 — Bundle size impact

```bash
du -sh node_modules/* 2>/dev/null | sort -rh | head -10
```

Flag packages where a lighter alternative exists or the package is only used in one place.

---

## Step 5 — Unused dependency detection

```bash
npx depcheck 2>&1 | head -40
```

For each flagged unused dependency: confirm it's truly unused, then `npm uninstall [package]`.

---

## Step 6 — Output

```
═══════════════════════════════════════════════════════
DEPENDENCY AUDIT — [date]
═══════════════════════════════════════════════════════

SECURITY
  Critical: [N]  ← must fix before deploy
  High:     [N]
  Moderate: [N]
  Low:      [N]

UPDATES AVAILABLE
  Patch: [N] packages  ← apply now
  Minor: [N] packages  ← review changelogs
  Major: [N] packages  ← migration tasks needed

LICENSE COMPLIANCE
  ✅ All production dependencies use permissive licenses
  — OR —
  🔴 [package] uses [license] — requires review

BUNDLE WEIGHT
  Packages > 100kB gzipped: [list]
  Unused packages: [list]

ACTIONS TAKEN:
  ✅ Applied [N] patch updates
  ✅ Fixed [N] vulnerabilities via npm audit fix

GITHUB ISSUES TO CREATE:
  □ "Upgrade [package] to v[N]" — label: dependency, tech-debt
═══════════════════════════════════════════════════════
```

---

## Step 7 — Create issues for major upgrades

```bash
gh issue create \
  --title "Upgrade [package] from v[current] to v[latest]" \
  --label "tech-debt,dependency" \
  --body "## Dependency Upgrade

**Package:** [package-name]
**Current:** [N]  **Latest:** [N]
**Migration guide:** [URL]
**Reason:** [Security fix / new features / EOL]
**Effort:** [S / M / L]

Identified by /dependency-audit on $(date +%Y-%m-%d)."
```

---

## Recommended schedule

| Cadence | Trigger |
|---|---|
| **Weekly** | `npm audit --audit-level=high` in CI (automatic) |
| **Monthly** | Full `/dependency-audit` sweep |
| **Before any release** | Full `/dependency-audit` as part of `/release-checklist` |
| **Immediately** | When a CVE is published for a package you use |

---

## Next step

- Critical/high fixed → re-run `npm audit` to confirm clean
- Major upgrade issues created → add to next sprint planning
- Full audit complete → proceed to `/release-checklist` if preparing a release
