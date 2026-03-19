---
name: client-handoff
description: Structured client handoff process for a delivered feature or milestone. Covers staging walkthrough, client access, feedback classification, go-live sign-off, and post-deploy notification. Run after /release-checklist passes. Triggers on "client handoff", "send to client", "client review", "staging for client", "ready for client", or "go-live approval".
disable-model-invocation: true
argument-hint: [milestone-name or feature description]
---

# /client-handoff — Client Delivery and Go-Live Process

Run after `/release-checklist` produces a GO verdict and the feature is deployed to staging.
Nothing goes to production without written client approval.

**Usage:** `/client-handoff "Q1 Dashboard Feature"` or `/client-handoff v2.1.0`

Full reference: `@docs/client-handoff.md`

---

## Step 1 — Pre-handoff checklist

Confirm all of these before proceeding:

```bash
# Confirm milestone issues are closed
gh issue list --milestone "$ARGUMENTS" --state open

# Confirm CI is green
gh run list --branch main --limit 3 \
  --json status,conclusion,name \
  --jq '.[] | "\(.name): \(.conclusion)"'
```

- [ ] All milestone issues closed
- [ ] CI green on main
- [ ] Feature deployed to staging
- [ ] Staging uses production-equivalent configuration (not dev seed data)

If any item is unchecked: **STOP** — resolve before proceeding.

---

## Step 2 — Staging walkthrough

Walk through every acceptance criterion on staging before involving the client.

```bash
# Pull acceptance criteria from all closed milestone issues
gh issue list --milestone "$ARGUMENTS" --state closed \
  --json number,title,body \
  --jq '.[] | "Issue #\(.number): \(.title)"'
```

Output a self-QA checklist:

```
STAGING WALKTHROUGH — $ARGUMENTS — [date]
──────────────────────────────────────────
For each acceptance criterion:
  ✅ / ❌  [Criterion] — [what was observed]

Edge cases verified:
  ✅ / ❌  Empty states / error states
  ✅ / ❌  Mobile / responsive (if applicable)

Issues found (fix before client review):
  🔴 [issue] — must fix
  🟡 [issue] — minor, note for client
```

**Do not proceed if any 🔴 items exist.** Fix them first.

---

## Step 3 — Draft client staging email

Draft a staging access email. Ask the user to confirm recipients, staging URL, and deadline before sending.

```
Subject: [Project Name] — $ARGUMENTS Ready for Review

Hi [Client name],

$ARGUMENTS is ready for your review on our staging environment.

Staging URL: [URL]
Credentials: [credentials or "use your existing login"]
Available until: [5 business days from today]

What to test:
• [Acceptance criterion 1 — plain language]
• [Acceptance criterion 2 — plain language]

Known limitations in this release:
• [Deferred items — be specific]

Please share feedback by [date]. Happy to schedule a walkthrough call if useful.

— [Your name], Code Impact
```

**Do not send until user confirms** recipients, URL, and deadline.

---

## Step 4 — Track feedback

When feedback arrives, classify each item before responding:

| Type | Definition | Action |
|------|-----------|--------|
| Bug | Behavior doesn't match accepted ACs | Fix before go-live |
| Change request | New or different behavior from spec | New scope — price separately |
| Clarification | Client doesn't understand current behavior | Provide docs or a call |

Create a GitHub issue for every bug. Label change requests `client-feedback` + `out-of-scope`.

---

## Step 5 — Go-live approval email

Draft the go-live approval request. **Do not deploy to production without written reply.**

```
Subject: [Project Name] — $ARGUMENTS Go-Live Approval

Hi [Client name],

We're ready to deploy $ARGUMENTS to production.

What's included:
• [Feature 1]
• [Feature 2]

Feedback addressed:
• [Their concern → resolution]

Deferred to next sprint:
• [Out-of-scope items]

Planned deployment: [date and time, timezone]
Estimated downtime: [none / X minutes for maintenance window]

Please reply to approve.

— [Your name], Code Impact
```

---

## Step 6 — Post-deploy verification and notification

After production deploy:

```bash
# Health check
curl -s -o /dev/null -w "%{http_code}" https://[production-url]/api/health \
  && echo " — Production: ✅" || echo " — Production: ❌"
```

Monitor for 30 minutes. Then send go-live notification:

```
Subject: [Project Name] — $ARGUMENTS is Live

Hi [Client name],

$ARGUMENTS is now live.

What's new:
• [Feature 1]
• [Feature 2]

If you run into anything, reply here or reach us at [support contact].

— [Your name], Code Impact
```

---

## Summary output

```
CLIENT HANDOFF — $ARGUMENTS — [date]
═══════════════════════════════════════════
  ✅ / ❌  Staging walkthrough passed
  ✅ / ❌  Staging access sent to client
  ✅ / ❌  Feedback received and classified
  ✅ / ❌  Bugs fixed before go-live
  ✅ / ❌  Written go-live approval received
  ✅ / ❌  Deployed to production
  ✅ / ❌  Post-deploy verification passed
  ✅ / ❌  Client go-live notification sent

STATUS: COMPLETE / IN PROGRESS / BLOCKED
═══════════════════════════════════════════
```
