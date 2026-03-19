# Incident Response
> Reference doc — load on demand with `@docs/incident-response.md`
> For the slash command, use `/incident-response`

---

## The First 5 Minutes

Speed matters more than certainty at the start. Execute in order — don't skip steps.

```
Minute 0-1: DETECT
  → Confirm it's a real incident (not a monitoring false positive)
  → Define the blast radius: who is affected and how many?

Minute 1-2: NOTIFY
  → Post in #incidents: "🔴 INCIDENT: [what's broken] — investigating"
  → If client-facing: notify team lead immediately via phone/text, not just Slack

Minute 2-5: STABILIZE (rollback first, diagnose second)
  → Is rollback possible? If yes, do it NOW — don't wait for root cause
  → Rollback is always faster than a hot-fix under pressure
  → See rollback procedure in the relevant release checklist
```

**The most important rule:** rollback first, understand why later.

---

## Rollback Triggers

Roll back immediately if any of the following are true:
- Error rate increased by more than 5× baseline in the last 10 minutes
- Any client reports data loss or data corruption
- Payment processing is failing
- Authentication is broken for any user segment
- A deploy happened in the last 4 hours and errors started after it

Do NOT wait for root cause before rolling back if any of these apply.

---

## During the Incident

```bash
# Check application logs
# [CUSTOMIZE: your logging platform — e.g., Railway logs, Datadog, Logtail]

# Check recent deploys
gh run list --branch main --limit 10 \
  --json status,conclusion,name,createdAt \
  --jq '.[] | "\(.createdAt) \(.name): \(.conclusion)"'

# Check for database issues
npx prisma migrate status 2>&1

# Check error rates
# [CUSTOMIZE: your monitoring dashboard URL]
```

Keep a running log in the #incidents Slack thread. Every action you take, every finding, every hypothesis. This becomes the post-mortem source.

---

## Client Communication Template

Use this within 15 minutes of confirming a client-facing incident:

```
Subject: Service Disruption — [brief description]

We're aware of an issue affecting [what's affected] that began at approximately [time].

Current status: [Investigating / Mitigating / Resolved]

Impact: [Who is affected and how]

Next update: [specific time, e.g., "in 30 minutes or when resolved"]

We'll provide a full summary once resolved.

— [Your name], Code Impact
```

Follow up at the promised time — even if you have nothing new, send "Still investigating, next update at [time]."

---

## After the Incident

Run `/incident-response post-mortem` or complete this template manually:

```
INCIDENT POST-MORTEM — [date] — [brief title]
═══════════════════════════════════════════════

SUMMARY
  What broke: [one sentence]
  Duration:   [start time] → [end time] = [X minutes/hours]
  Impact:     [users affected, data affected, revenue impacted if known]
  Resolved by: [rollback / hotfix / config change]

TIMELINE
  [HH:MM] — [event or action]
  [HH:MM] — [event or action]

ROOT CAUSE
  [What actually caused the incident — be specific]

CONTRIBUTING FACTORS
  [Process gaps, missing monitoring, unclear runbooks, etc.]

WHAT WENT WELL
  [Detection was fast / rollback worked cleanly / communication was clear]

ACTION ITEMS (create GitHub issues for each)
  □ [Specific preventive action] — owner: [name] — due: [date]
  □ [Add monitoring for X] — owner: [name] — due: [date]
  □ [Add test for Y] — owner: [name] — due: [date]
```

Post the post-mortem in the #incidents channel and share with the client within 24 hours of resolution if the incident was client-facing.

---

## Severity Levels

| Level | Definition | Response Time | Client Notification |
|-------|-----------|--------------|---------------------|
| P1 | Production down or data loss | Immediate, all hands | Within 15 minutes |
| P2 | Major feature broken, workaround exists | Within 1 hour | Within 1 hour |
| P3 | Minor degradation, not blocking | Next business day | Only if client notices |

---

## Rules

- Never deploy a hotfix under pressure without `/verify` passing first — a broken hotfix makes things worse
- Never access production data directly without documenting why and what you did
- Post-mortems are blameless — the goal is systemic improvement, not individual accountability
- Every P1 and P2 requires a post-mortem, no exceptions
- Action items from post-mortems must become GitHub issues within 24 hours
