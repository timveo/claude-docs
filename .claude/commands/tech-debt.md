---
name: tech-debt
description: Identifies and catalogues technical debt in the codebase — code smells, complexity hotspots, TODO/FIXME tracking, duplicated logic, and outdated patterns. Outputs a prioritized debt backlog and creates GitHub issues for significant items. Also supports creating Architectural Decision Records (ADRs) to document deliberate trade-offs. Triggers on "tech debt", "code smells", "refactor audit", "TODO review", "create ADR", or "debt backlog".
disable-model-invocation: true
argument-hint: [scope: "current-branch" | "full-codebase" | "adr <title>"]
---

# /tech-debt — Technical Debt Audit and ADR Management

Use to audit debt in changed code (`current-branch`), sweep the full codebase (`full-codebase`),
or document a deliberate trade-off as an ADR (`adr <title>`).

**Usage:**
- `/tech-debt current-branch` — audit only the diff on the current branch
- `/tech-debt full-codebase` — sweep the whole codebase
- `/tech-debt adr "Use PgBouncer for connection pooling"` — create a new ADR

---

## Mode A: Debt Audit (current-branch or full-codebase)

### Step 1 — TODO / FIXME scan

```bash
# Count and list all TODO/FIXME/HACK/XXX comments
grep -rn "TODO\|FIXME\|HACK\|XXX\|TEMP\|@deprecated" \
  --include="*.ts" --include="*.tsx" --include="*.js" \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.next \
  . | sort
```

Categorize each:
- **FIXME / HACK / XXX** → likely active debt, flag for triage
- **TODO** → check age (if the line is in a commit > 90 days old, escalate)
- **@deprecated** → flag if the deprecated item is still being imported anywhere

```bash
# Check if deprecated items are still imported
grep -rn "@deprecated" --include="*.ts" --include="*.tsx" -l . \
  --exclude-dir=node_modules | while read file; do
    symbol=$(grep -o "function \w*\|const \w*\|class \w*" "$file" | head -1)
    echo "Deprecated in $file — checking usage..."
    grep -rn "$symbol" --include="*.ts" --include="*.tsx" . \
      --exclude-dir=node_modules | grep -v "$file"
done
```

---

### Step 2 — Complexity hotspots

```bash
# Find functions over 50 lines (complexity proxy)
awk '/^(export |async )?function |^const \w+ = (async )?\(/{start=NR} \
  start && NR-start>50{print FILENAME ":" start " — function is " NR-start " lines"; start=0}' \
  $(find . -name "*.ts" -o -name "*.tsx" | grep -v node_modules | grep -v dist)
```

```bash
# Find files over 300 lines (god file risk)
find . \( -name "*.ts" -o -name "*.tsx" \) \
  -not -path "*/node_modules/*" -not -path "*/dist/*" \
  | xargs wc -l 2>/dev/null | sort -rn | head -20
```

Flag files > 500 lines or functions > 80 lines as **Warning** — candidate for extraction.

---

### Step 3 — Duplication detection

```bash
# Find repeated patterns (5+ identical lines across files)
grep -rn --include="*.ts" --include="*.tsx" \
  -h "." --exclude-dir=node_modules --exclude-dir=dist \
  | sort | uniq -d -c | sort -rn | head -20
```

Look for:
- Repeated error handling patterns → candidate for shared middleware/utility
- Repeated data transformation logic → candidate for a shared utility function
- Copy-pasted validation logic → candidate for a shared Zod schema

---

### Step 4 — Outdated patterns

```bash
# Find patterns that should have been migrated
grep -rn \
  -e "require(" \
  -e "var " \
  -e "\.then(" \
  -e "callback(" \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules . | grep -v "// legacy\|// vendor\|test"
```

- `require()` in TypeScript → should be ES `import`
- `var ` → should be `const`/`let`
- `.then()` chains → should be `async/await` (except where intentional)

```bash
# Find any direct database calls bypassing the service layer
grep -rn "prisma\." \
  --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules \
  . | grep -v "repository\|service\|prisma\.ts\|schema"
```

Flag direct Prisma calls in route handlers or components as **Warning** — they bypass the service layer.

---

### Step 5 — Dead code detection

```bash
# Find exported symbols that are never imported
# (approximation — run full treeshaking analysis for precision)
grep -rn "^export " --include="*.ts" --include="*.tsx" \
  --exclude-dir=node_modules . \
  | grep -v "index.ts" \
  | while IFS=: read file line content; do
    symbol=$(echo "$content" | grep -o "export \(function\|const\|class\|interface\|type\) \w*" \
      | awk '{print $NF}')
    count=$(grep -rn "$symbol" --include="*.ts" --include="*.tsx" \
      --exclude-dir=node_modules . | grep -v "$file" | wc -l)
    if [ "$count" -eq 0 ]; then
      echo "Possibly unused: $symbol in $file"
    fi
done
```

---

### Step 6 — Debt audit output

```
═══════════════════════════════════════════════════════
TECH DEBT AUDIT — [scope] — [date]
═══════════════════════════════════════════════════════

SUMMARY
  TODOs/FIXMEs: [N] items ([N] over 90 days old)
  Complexity hotspots: [N] files, [N] oversized functions
  Duplication candidates: [N] patterns
  Outdated patterns: [N] instances
  Dead code candidates: [N] exports

HIGH PRIORITY (address within current sprint or next):
  🔴 [Item] — [file] — [recommended action] — Est: [S/M/L]

MEDIUM PRIORITY (schedule within next 2 sprints):
  🟡 [Item] — [file] — [recommended action] — Est: [S/M/L]

LOW PRIORITY (backlog):
  🔵 [Item] — [file] — [note]

RECOMMENDED GITHUB ISSUES TO CREATE:
  □ "[Short title]" — label: tech-debt — [1-sentence description]
  □ "[Short title]" — label: tech-debt — [1-sentence description]
═══════════════════════════════════════════════════════
```

Ask: "Should I create GitHub issues for the High and Medium priority items?"

If yes:
```bash
gh issue create \
  --title "[debt item title]" \
  --label "tech-debt" \
  --body "## Debt Item\n\n**Location:** [file:line]\n**Issue:** [description]\n**Recommended fix:** [action]\n**Effort estimate:** [S/M/L]\n\nIdentified by /tech-debt on $(date +%Y-%m-%d)."
```

---

## Mode B: Create an ADR

**Usage:** `/tech-debt adr "Title of the decision"`

ADRs document deliberate architectural trade-offs so future team members understand *why* a decision was made, not just *what* was done.

### Step 1 — Check existing ADRs

```bash
ls docs/adr/ 2>/dev/null || echo "No ADR directory yet"
```

If the directory doesn't exist, create it:
```bash
mkdir -p docs/adr
```

### Step 2 — Determine the next ADR number

```bash
ls docs/adr/*.md 2>/dev/null | wc -l
```

### Step 3 — Create the ADR file

Create `docs/adr/[NNN]-[kebab-case-title].md` with this structure:

```markdown
# ADR-[NNN]: [Title]

**Date:** [YYYY-MM-DD]
**Status:** Accepted | Proposed | Deprecated | Superseded by ADR-[NNN]
**Deciders:** [names or roles]

---

## Context

[What is the issue or situation that requires this decision?
What forces are at play — technical, business, resource constraints?]

## Decision

[What decision was made? State it clearly and affirmatively.
"We will use X because..."]

## Consequences

**Positive:**
- [Benefit 1]
- [Benefit 2]

**Negative / trade-offs:**
- [Cost or limitation 1]
- [Cost or limitation 2]

**Debt incurred (if any):**
- [What future work is deferred or accepted as a result?]
- GitHub issue: #[N] (if one was created)

## Alternatives considered

| Option | Reason not chosen |
|---|---|
| [Alternative A] | [Why rejected] |
| [Alternative B] | [Why rejected] |

## References

- [Link to relevant docs, issues, or prior art]
```

### Step 4 — Stage the ADR

```bash
git add docs/adr/
git status
```

Remind the user to commit the ADR as part of the PR that implements the decision it documents.

---

## Next step

- After a debt audit: review the prioritized list, create issues for High/Medium items, then return to current sprint work
- After creating an ADR: commit it alongside the implementation PR so reviewers see the rationale
