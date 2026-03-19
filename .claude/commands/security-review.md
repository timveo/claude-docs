---
name: security-review
description: OWASP Top 10 aligned security audit of all changes on the current branch. Covers authentication, authorization, input validation, injection risks, secret exposure, API security, and dependency vulnerabilities. Run before any PR that touches auth, payments, user data, or external integrations. Triggers on "security review", "security audit", "check for vulnerabilities", or "OWASP review".
disable-model-invocation: true
---

# /security-review — OWASP-Aligned Security Audit

Run before merging any PR that touches authentication, authorization, user data, payments,
external integrations, or public API surfaces. Based on the OWASP Top 10 framework.

---

## Step 1 — Scope the review

```bash
git diff main...HEAD --name-only
git diff main...HEAD --stat
```

Identify which categories are in scope based on the files changed:

| Files changed | Risks in scope |
|---|---|
| `auth/`, `middleware/`, JWT/session code | A01: Broken Access Control, A07: Auth Failures |
| User input handlers, API routes | A03: Injection, A04: Insecure Design |
| `.env*`, config files, secrets | A02: Cryptographic Failures, secret exposure |
| `package.json`, `package-lock.json` | A06: Vulnerable Components |
| File upload, download, rendering | A05: Security Misconfiguration |
| Payment, PII, sensitive data flows | A02: Cryptographic Failures |

---

## Step 2 — Static secret scan

```bash
# Scan for exposed secrets, tokens, and credentials
git diff main...HEAD | grep -iE \
  "(api_key|apikey|secret|password|passwd|token|bearer|private_key|access_key|auth_token|client_secret)" \
  | grep -v "^---" | grep -v "^+++" | grep -v "placeholder\|example\|your_\|<YOUR\|TODO"
```

```bash
# Check for hardcoded connection strings or credentials
git diff main...HEAD | grep -iE \
  "(mongodb\+srv|postgres://|mysql://|redis://|amqp://)" \
  | grep -v "localhost\|127.0.0.1"
```

Flag any match as **Critical** — no exceptions.

---

## Step 3 — Authentication and authorization review

Audit changed auth-related code against these checks:

**A01 — Broken Access Control**
- [ ] Every route has explicit authorization middleware — no route is accidentally public
- [ ] Resource ownership is verified before read/write (user can only access their own data)
- [ ] Admin-only endpoints verify admin role, not just authentication
- [ ] No `isAdmin` or role checks in the frontend only — server enforces all permissions
- [ ] IDOR risk: IDs in URLs/params are validated against the authenticated user's access scope

**A07 — Identification and Authentication Failures**
- [ ] Passwords hashed with bcrypt (cost ≥ 12) or argon2 — never MD5/SHA1/plain
- [ ] JWT secrets loaded from env vars — not hardcoded
- [ ] JWT expiry is set (preferably ≤ 1 hour for access tokens)
- [ ] Refresh token rotation implemented if refresh tokens are used
- [ ] Account lockout or rate limiting on login endpoints
- [ ] Password reset tokens are single-use and expire within 30 minutes

---

## Step 4 — Input validation and injection review

**A03 — Injection (SQL, NoSQL, command, template)**

```bash
# Find raw query construction (potential SQL injection)
git diff main...HEAD | grep -E "(query|execute|raw)\s*\(" \
  | grep -v "prisma\.\|sequelize\.\|knex\."
```

- [ ] All database queries use parameterized queries or ORM methods — no string concatenation
- [ ] Prisma raw queries (`$queryRaw`, `$executeRaw`) use tagged template literals, not string concatenation
- [ ] No `eval()`, `new Function()`, or dynamic `require()` with user-controlled input
- [ ] Shell commands (`exec`, `spawn`, `execSync`) don't include user input without sanitization
- [ ] Template engines escape output by default — no manual `{{{ }}}` or `raw` with user data

**A04 — Insecure Design**
- [ ] File uploads validate MIME type server-side (not just client-side extension)
- [ ] Uploaded files stored outside the web root — not in `/public`
- [ ] File size limits enforced

---

## Step 5 — API security review

**CORS configuration**
```bash
git diff main...HEAD | grep -iE "cors|origin|allowedOrigins"
```

- [ ] CORS origin is a whitelist of specific domains — not `*` in production
- [ ] `credentials: true` is only set when necessary and paired with a specific origin

**Rate limiting**
- [ ] Auth endpoints (login, register, password reset) have rate limiting
- [ ] Public API endpoints have rate limiting
- [ ] Rate limit library is applied in middleware, not per-route

**Security headers**
```bash
git diff main...HEAD | grep -iE "helmet|Content-Security-Policy|X-Frame-Options|HSTS"
```

- [ ] `helmet` (or equivalent) is applied globally
- [ ] CSP header is configured — no `unsafe-inline` or `unsafe-eval` without justification

**HTTP method handling**
- [ ] Mutating operations (create/update/delete) use POST/PUT/PATCH/DELETE — not GET
- [ ] Sensitive actions require a CSRF token or `SameSite=Strict` cookie policy

---

## Step 6 — Cryptographic failures review

**A02 — Cryptographic Failures**
- [ ] Sensitive data (PII, payment info) encrypted at rest
- [ ] HTTPS enforced — no HTTP endpoints in production config
- [ ] No sensitive data logged (passwords, tokens, full card numbers, SSNs)
- [ ] Cookies storing session tokens set with `HttpOnly`, `Secure`, `SameSite=Strict`

```bash
# Check for console.log of potentially sensitive objects
git diff main...HEAD | grep "console\.log" \
  | grep -iE "(user|token|password|secret|key|auth|session)"
```

---

## Step 7 — Dependency vulnerability check

```bash
npm audit --audit-level=moderate 2>&1 | head -50
```

Flag any `critical` or `high` severity findings as **Critical**.
Flag `moderate` findings as **Warning**.

---

## Step 8 — Security review output

```
═══════════════════════════════════════════════════════
SECURITY REVIEW — [branch-name] — [date]
OWASP Top 10 Coverage: A01, A02, A03, A04, A06, A07
═══════════════════════════════════════════════════════

SCOPE: [list of files/categories reviewed]

CRITICAL (must fix before merge):
  🔴 [Issue] — [file:line] — [remediation]

WARNING (fix before merge, escalate if unsure):
  🟡 [Issue] — [file:line] — [remediation]

INFORMATIONAL (good to know, low urgency):
  🔵 [Issue] — [note]

PASSED CHECKS:
  ✅ No secrets detected in diff
  ✅ Auth middleware present on all routes reviewed
  ✅ Parameterized queries used throughout
  ✅ CORS origin whitelisted
  ✅ npm audit: no critical/high vulnerabilities
  [additional passing checks...]

VERDICT: APPROVED / NEEDS FIXES / BLOCKED
═══════════════════════════════════════════════════════
```

---

## Severity guide

| Level | Meaning | Action |
|---|---|---|
| 🔴 **Critical** | Exploitable, data exposure, auth bypass | Block merge — fix immediately |
| 🟡 **Warning** | Real risk, may not be immediately exploitable | Fix before merge or document accepted risk |
| 🔵 **Informational** | Security hygiene, defense in depth | Log in tech-debt backlog |

---

## Next step

- All Critical and Warning issues resolved → proceed with `/pr-prep`
- Any unresolved Critical → do not merge; return to the worktree and fix
- Accepted risk items → document in the PR body under a "Security Notes" section
