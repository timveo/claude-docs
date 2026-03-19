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
# Fix vulnerabilities that don't require semver-major bumps
npm audit fix --dry-run 2>&1

# If safe, apply:
npm audit fix
```

For vulnerabilities that require breaking upgrades:
```bash
npm audit fix --force --dry-run 2>&1
```

Do NOT run `npm audit fix --force` without reviewing the dry-run output — it may introduce breaking changes.

---

## Step 2 — Outdated package review

```bash
npm outdated 2>&1
```

Categorize updates:

**Patch updates** (e.g., 1.2.3 → 1.2.4): Apply without review — bug fixes only.
```bash
npx npm-check-updates --target patch --upgrade
npm install
```

**Minor updates** (e.g., 1.2.0 → 1.3.0): Review changelogs — usually safe but may have behavior changes.

**Major updates** (e.g., 1.x → 2.x): Treat as a migration task — create a GitHub issue, do not apply inline.

For each major update needed, check the package's migration guide and estimate effort:
```bash
# View a package's changelog before upgrading
npm info [package-name] changelog 2>/dev/null || \
  echo "Visit: https://github.com/[org]/[package-name]/releases"
```

---

## Step 3 — License compliance check

```bash
npx license-checker --summary 2>/dev/null || \
  npx license-checker --onlyAllow "MIT;ISC;BSD-2-Clause;BSD-3-Clause;Apache-2.0;CC0-1.0;Unlicense;0BSD;Python-2.0" \
  --excludePrivatePackages 2>&1 | head -40
```

**Acceptable licenses:** MIT, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, CC0-1.0, Unlicense

**Requires review:**
- GPL-2.0, GPL-3.0, LGPL: Copyleft — may require open-sourcing your code if bundled
- AGPL: Strong copyleft — flag for legal review before any commercial use
- CC-BY-SA: Check attribution requirements
- Unknown / UNLICENSED: Treat as proprietary — may not be usable

Flag any non-permissive license as **Critical** if it's a production dependency (not devDependency).

```bash
# List packages with non-standard licenses
npx license-checker --exclude "MIT,ISC,BSD-2-Clause,BSD-3-Clause,Apache-2.0" \
  --excludePrivatePackages --production 2>&1
```

---

## Step 4 — Bundle size impact

```bash
# Check for packages that contribute disproportionate bundle size
npx bundlephobia-cli $(cat package.json | \
  node -e "const d=require('/dev/stdin'); console.log(Object.keys(d.dependencies||{}).join(' '))") \
  2>/dev/null || echo "Install bundlephobia-cli for bundle analysis"
```

Manual check for known heavy packages:
```bash
# List top 10 largest packages by install size
du -sh node_modules/* 2>/dev/null | sort -rh | head -10
```

Flag packages where:
- A lighter alternative exists (e.g., `date-fns` instead of `moment`)
- The package is only used in one place and could be replaced with native code
- The package is a devDependency accidentally included in production dependencies

---

## Step 5 — Unused dependency detection

```bash
npx depcheck 2>&1 | head -40
```

For each flagged unused dependency:
- Confirm it's truly unused (depcheck has false positives with dynamic requires)
- If genuinely unused: `npm uninstall [package-name]`
- If used dynamically (plugins, peer deps): add to `depcheck` ignore list in `package.json`

---

## Step 6 — Dependency audit output

```
═══════════════════════════════════════════════════════
DEPENDENCY AUDIT — [date]
═══════════════════════════════════════════════════════

SECURITY
  Critical vulnerabilities: [N]  ← must fix before deploy
  High:    [N]
  Moderate: [N]
  Low:     [N]

UPDATES AVAILABLE
  Patch:  [N] packages  ← apply now
  Minor:  [N] packages  ← review changelogs
  Major:  [N] packages  ← migration tasks needed

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
  □ "Review [license] license for [package]" — label: legal, dependency

═══════════════════════════════════════════════════════
```

---

## Step 7 — Create issues for major upgrades

For each major version upgrade that needs a migration task:

```bash
gh issue create \
  --title "Upgrade [package] from v[current] to v[latest]" \
  --label "tech-debt,dependency" \
  --body "## Dependency Upgrade

**Package:** [package-name]
**Current version:** [N]
**Latest version:** [N]
**Migration guide:** [URL]

### Reason for upgrade
[Security fix / new features / maintenance / EOL of current version]

### Breaking changes summary
[Key changes from migration guide]

### Effort estimate
[S / M / L]

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

- Critical/high vulnerabilities fixed → re-run `npm audit` to confirm clean
- Major upgrade issues created → add to next sprint planning
- Full audit complete → proceed to `/release-checklist` if preparing a release
