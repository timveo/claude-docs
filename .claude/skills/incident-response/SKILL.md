---
name: incident-response
description: Production incident response and post-mortem workflow. Covers detection, rollback decision, client communication, and post-mortem documentation. Triggers on "incident", "production down", "something broke", "rollback", "post-mortem", or "outage".
disable-model-invocation: true
argument-hint: ["live" for active incident | "post-mortem" for retrospective]
---

# /incident-response — Production Incident Response

Two modes:
- `/incident-response live` — active incident, execute immediately
- `/incident-response post-mortem` — document after resolution

Full reference: `@docs/incident-response.md`

---

## Mode: LIVE INCIDENT

### Minute 0-1 — Detect and size

Confirm it's real (not a monitoring fluke), then size the blast radius:

```bash
# Check recent deploys — did something ship before this started?
gh run list --branch main --limit 5 \
  --json status,conclusion,name,createdAt \
  --jq '.[] | "\(.createdAt) \(.name): \(.conclusion)"'

# Check application health
curl -s -o /dev/null -w "%{http_code}" https://[production-url]/api/health

# Check error rates
# [CUSTOMIZE: your monitoring dashboard or log query]
```

Post immediately in #incidents:
```
🔴 INCIDENT: [what's broken] — investigating
Detected: [time]
Affected: [who/what]
```

### Minute 1-3 — Rollback decision

**Roll back immediately** if any of these are true:
- Error rate 5× baseline since the last deploy
- Any data loss or data corruption reported
- Auth or payments broken
- A deploy happened in the last 4 hours

```bash
# Rollback: revert to previous deployment
# [CUSTOMIZE: your deployment platform rollback command]
# e.g., Railway: redeploy previous deployment from dashboard
# e.g., Vercel: `vercel rollback` from CLI
# e.g., Docker: docker pull [image:previous-tag] && docker-compose up -d
```

**If rolling back:** Notify #incidents: "🟡 Rolling back to [previous version] — ETA [X mins]"

**If NOT rolling back:** Document why in #incidents and continue investigating.

### Minute 3-15 — Diagnose (only if not rolling back)

```bash
# Check logs for errors starting at incident time
# [CUSTOMIZE: your log query]

# Check database
npx prisma migrate status 2>&1

# Check for recent code changes in the affected area
git log --oneline -20
```

### Notify the client (if P1 or P2)

Send within 15 minutes of confirming client impact:

```
Subject: Service Disruption — [brief description]

We're aware of an issue affecting [what] that began at approximately [time].

Status: Investigating / Mitigating
Impact: [who and how]
Next update: [specific time]

— [Your name], Code Impact
```

---

## Mode: POST-MORTEM

Run after the incident is resolved. Every P1 and P2 requires one.

```bash
# Pull the incident timeline from git and CI logs
gh run list --branch main --limit 20 \
  --json status,conclusion,name,createdAt \
  --jq '.[] | "\(.createdAt) \(.name): \(.conclusion)"'
```

Generate the post-mortem document:

```
INCIDENT POST-MORTEM — [date] — [brief title]
═══════════════════════════════════════════════

SUMMARY
  What broke:    [one sentence]
  Duration:      [start] → [end] = [X minutes]
  Impact:        [users affected, features broken]
  Resolution:    [rollback / hotfix / config change]
  Client impact: [yes/no — if yes, was client notified?]

TIMELINE
  [HH:MM] — [event or action]
  [HH:MM] — [event or action]
  [HH:MM] — Resolved

ROOT CAUSE
  [Specific technical cause — be precise, not vague]

CONTRIBUTING FACTORS
  [Process gaps, missing tests, missing monitoring, unclear docs]

WHAT WENT WELL
  [What worked — fast detection, clean rollback, good comms]

ACTION ITEMS
  □ [Preventive action] — owner: [name] — due: [date] — GitHub issue: #
  □ [Add monitoring for X] — owner: [name] — due: [date] — GitHub issue: #
  □ [Add test for edge case Y] — owner: [name] — due: [date] — GitHub issue: #
```

Create a GitHub issue for every action item — label `incident` + `tech-debt`.

Post in #incidents and share with client within 24 hours if client-facing.

---

## Severity Quick Reference

| Level | Definition | Client notification |
|-------|-----------|---------------------|
| P1 | Production down or data loss | Within 15 minutes |
| P2 | Major feature broken | Within 1 hour |
| P3 | Minor degradation | Only if they notice |

---

## Non-negotiable rules

- Roll back first — diagnose second. Always.
- Never hotfix to production without `/verify` passing.
- Never access production data directly without documenting it.
- Post-mortems are blameless. The goal is systemic improvement.
- Action items must become GitHub issues within 24 hours of the post-mortem.
