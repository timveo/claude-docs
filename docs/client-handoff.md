# Client Handoff Guide
> Reference doc — load on demand with `@docs/client-handoff.md`
> For the slash command, use `/client-handoff`

---

## Overview

Every feature or milestone delivered to a client follows this handoff process.
A clean handoff prevents "it worked on our end" situations and builds client trust.

---

## Pre-Handoff Requirements

Before initiating a client handoff, confirm:
- [ ] All milestone issues are closed and CI is green on main
- [ ] `/release-checklist` completed with GO verdict
- [ ] Feature is deployed to **staging** (not just passing local tests)
- [ ] Staging environment uses production-equivalent data (not seed data)

---

## Step 1 — Staging Walkthrough

Before involving the client, walk through the feature on staging yourself using the
acceptance criteria from each issue. This is your own QA pass — not Claude's, not CI's.

```
Staging Walkthrough — [feature name] — [date]

For each acceptance criterion:
  ✅ / ❌  [Criterion] — [what you observed on staging]

Edge cases tested:
  ✅ / ❌  [Edge case]

Known limitations or deferred items:
  • [anything the client should know before testing]
```

If anything fails on staging, fix it before scheduling the client review.

---

## Step 2 — Client Staging Access

Send the client their staging access:

```
Subject: [Project Name] — [Feature/Milestone] Ready for Review

Hi [Client name],

[Feature/milestone name] is ready for your review on our staging environment.

Staging URL: [URL]
Credentials: [username / password or "use your existing credentials"]
Available until: [date — typically 5 business days]

What to test:
• [Acceptance criterion 1 in plain language]
• [Acceptance criterion 2 in plain language]
• [Acceptance criterion 3 in plain language]

Known limitations in this release:
• [Anything deferred to a future sprint]

Please share feedback by [specific date]. We'll schedule a 30-minute walkthrough
call if that would be helpful.

— [Your name], Code Impact
```

---

## Step 3 — Feedback Collection

Collect client feedback in a structured way. Don't let it arrive as Slack messages
or email threads that get lost.

Options (pick one per project):
- GitHub issue with label `client-feedback` — client comments directly
- Shared doc (Notion, Google Docs) with a structured feedback form
- Loom video from the client showing what they're seeing

For each piece of feedback, classify it before responding:
- **Bug** (broken behavior vs. acceptance criteria) → fix before go-live
- **Change request** (new or different behavior from spec) → scope and schedule separately
- **Clarification** (they don't understand how it works) → provide documentation or a quick call

Never let "change request" slip through as a "bug fix" without explicit scope discussion.

---

## Step 4 — Go-Live Sign-Off

Get explicit written sign-off before deploying to production.

```
Subject: [Project Name] — [Feature/Milestone] Go-Live Approval

Hi [Client name],

Following your review, we're ready to deploy [feature/milestone] to production.

Summary of what's included:
• [Feature 1]
• [Feature 2]

Feedback addressed:
• [Issue they raised → how it was resolved]

Deferred to next sprint (out of scope for this release):
• [Item]

Planned deployment window: [date and time, e.g., "Tuesday March 24, 10am PT"]
Estimated downtime: [none / X minutes]

Please confirm approval by replying to this email.

— [Your name], Code Impact
```

**Do not deploy to production without written approval.** A Slack "sounds good" is not approval.

---

## Step 5 — Post-Deploy Verification

After deploying to production:

```bash
# Run smoke tests against production
curl -s -o /dev/null -w "%{http_code}" https://[production-url]/api/health

# Verify key workflows are functioning
# [CUSTOMIZE: list 2-3 critical paths to manually verify post-deploy]
```

Monitor for 30 minutes post-deploy before notifying the client.

---

## Step 6 — Client Go-Live Notification

```
Subject: [Project Name] — [Feature/Milestone] is Live

Hi [Client name],

[Feature/milestone name] is now live in production.

What's new:
• [Feature 1 — one sentence description]
• [Feature 2 — one sentence description]

Documentation:
• [Link to any user docs or release notes, if applicable]

If you experience any issues, reply to this email or reach us at [support contact].
We're monitoring closely for the next 24 hours.

— [Your name], Code Impact
```

---

## Handoff Quality Gates

| Gate | Required for |
|------|-------------|
| Staging walkthrough complete | Every handoff |
| Client staging access sent | Every handoff |
| Written go-live approval received | Every production deploy |
| Post-deploy verification run | Every production deploy |
| Client notified post-deploy | Every production deploy |
| Feedback classified (bug vs. change request) | Any client feedback received |

---

## Rules

- Staging must match production configuration — no "it'll work in prod" assumptions
- Never deploy to production without written client approval
- Change requests discovered during review are new scope — price and schedule them separately
- Every piece of client feedback gets a written response, even if the answer is "out of scope"
