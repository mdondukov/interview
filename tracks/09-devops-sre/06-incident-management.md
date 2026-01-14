# 06. Incident Management

[← Назад к списку тем](README.md)

---

## Incident Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Incident Lifecycle                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. DETECTION                                                       │
│     - Monitoring alert                                              │
│     - Customer report                                               │
│     - Internal discovery                                            │
│                                                                     │
│  2. TRIAGE                                                          │
│     - Assess severity                                               │
│     - Assign incident commander                                     │
│     - Open communication channels                                   │
│                                                                     │
│  3. RESPONSE                                                        │
│     - Assemble team                                                 │
│     - Investigate                                                   │
│     - Communicate status                                            │
│     - Mitigate                                                      │
│                                                                     │
│  4. RESOLUTION                                                      │
│     - Implement fix                                                 │
│     - Verify recovery                                               │
│     - Stand down                                                    │
│                                                                     │
│  5. POST-INCIDENT                                                   │
│     - Write postmortem                                              │
│     - Action items                                                  │
│     - Share learnings                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Severity Levels

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Incident Severity                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SEV1 / P1 - Critical                                               │
│  ─────────────────────                                              │
│  - Complete service outage                                          │
│  - Major data loss/breach                                           │
│  - Revenue impact                                                   │
│  - All hands on deck                                                │
│  - Page immediately, any time                                       │
│                                                                     │
│  SEV2 / P2 - High                                                   │
│  ───────────────────                                                │
│  - Partial outage                                                   │
│  - Degraded service                                                 │
│  - Workaround exists                                                │
│  - Page during business hours                                       │
│                                                                     │
│  SEV3 / P3 - Medium                                                 │
│  ────────────────────                                               │
│  - Minor feature broken                                             │
│  - Limited impact                                                   │
│  - Handle during business hours                                     │
│                                                                     │
│  SEV4 / P4 - Low                                                    │
│  ──────────────────                                                 │
│  - Cosmetic issues                                                  │
│  - Tech debt                                                        │
│  - Schedule for later                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Incident Roles

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Incident Roles                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  INCIDENT COMMANDER (IC)                                            │
│  - Single point of authority                                        │
│  - Coordinates response                                             │
│  - Makes decisions                                                  │
│  - Does NOT debug (delegates)                                       │
│                                                                     │
│  TECHNICAL LEAD                                                     │
│  - Leads investigation                                              │
│  - Coordinates engineers                                            │
│  - Proposes solutions                                               │
│                                                                     │
│  COMMUNICATIONS LEAD                                                │
│  - Updates status page                                              │
│  - Coordinates with customer support                                │
│  - Internal stakeholder updates                                     │
│                                                                     │
│  SCRIBE                                                             │
│  - Documents timeline                                               │
│  - Records decisions                                                │
│  - Captures action items                                            │
│                                                                     │
│  SUBJECT MATTER EXPERTS                                             │
│  - Deep knowledge of specific systems                               │
│  - Called in as needed                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## On-Call

### On-Call Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                   On-Call Best Practices                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ROTATION                                                           │
│  - Weekly rotation (not longer)                                     │
│  - Primary + secondary                                              │
│  - Handoff documentation                                            │
│  - Follow-the-sun for global teams                                  │
│                                                                     │
│  EXPECTATIONS                                                       │
│  - Response time SLA (e.g., 15 min)                                 │
│  - Escalation path clear                                            │
│  - Tools accessible (laptop, VPN, creds)                            │
│                                                                     │
│  SUPPORT                                                            │
│  - Good runbooks                                                    │
│  - Easy escalation                                                  │
│  - Compensation (time off, pay)                                     │
│  - Quiet on-call = healthy system                                   │
│                                                                     │
│  SUSTAINABILITY                                                     │
│  - Track on-call load                                               │
│  - Reduce noise from alerts                                         │
│  - Fix recurring issues                                             │
│  - On-call shouldn't be miserable                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Runbook Template

```markdown
# Runbook: High Error Rate

## Alert Details
- **Alert:** HighErrorRate
- **Severity:** High
- **SLO Impact:** Yes

## Quick Assessment
1. Check error rate dashboard: [link]
2. Check recent deployments: [link]
3. Check dependent services: [link]

## Common Causes & Fixes

### Database Connection Issues
- **Symptom:** Connection timeout errors
- **Check:** `kubectl logs deployment/app | grep "connection"`
- **Fix:** Restart app or check DB health

### Memory Issues
- **Symptom:** OOM errors
- **Check:** `kubectl top pods`
- **Fix:** Scale up or restart pods

### Bad Deployment
- **Symptom:** Errors started after deploy
- **Check:** `kubectl rollout history deployment/app`
- **Fix:** `kubectl rollout undo deployment/app`

## Escalation
- If not resolved in 15 min, escalate to [team]
- Page secondary if primary unavailable

## Communication
- Update #incidents channel every 15 min
- Update status page if user-facing
```

---

## Postmortems

### Blameless Postmortem

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Blameless Culture                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRINCIPLES                                                         │
│  ──────────                                                         │
│  - Focus on systems, not individuals                                │
│  - "How did the system allow this?" not "Who did this?"             │
│  - Assume competent people trying their best                        │
│  - Every failure is a learning opportunity                          │
│                                                                     │
│  BAD QUESTIONS                        GOOD QUESTIONS                │
│  ─────────────                        ──────────────                │
│  "Who deployed the bad code?"         "What made it possible for    │
│                                        bad code to reach prod?"     │
│                                                                     │
│  "Why didn't you notice?"             "What signals were missing?"  │
│                                                                     │
│  "Who approved this?"                 "What could the review        │
│                                        process have caught?"        │
│                                                                     │
│  WHY IT MATTERS                                                     │
│  ──────────────                                                     │
│  - Blame → hiding → slower detection                                │
│  - Blameless → transparency → faster fixes                          │
│  - Trust enables honesty enables improvement                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Postmortem Template

```markdown
# Postmortem: [Incident Title]

**Date:** YYYY-MM-DD
**Duration:** HH:MM
**Severity:** SEV1/2/3
**Authors:** [Names]

## Summary
[1-2 sentence summary of what happened]

## Impact
- Users affected: X
- Duration: Y minutes
- Revenue impact: $Z (if applicable)
- SLA breached: Yes/No

## Timeline (all times UTC)
| Time  | Event |
|-------|-------|
| 14:00 | Alert fired for high error rate |
| 14:05 | On-call acknowledged |
| 14:15 | Root cause identified |
| 14:30 | Fix deployed |
| 14:35 | Service recovered |

## Root Cause
[Detailed technical explanation]

## What Went Well
- Alert fired quickly
- Team coordinated effectively
- Communication was clear

## What Went Poorly
- Runbook was outdated
- Took 15 min to identify cause
- Status page updated late

## Lessons Learned
- [Key insight 1]
- [Key insight 2]

## Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| Update runbook | @alice | 2024-01-15 |
| Add alert for X | @bob | 2024-01-20 |
| Improve monitoring | @carol | 2024-01-25 |

## Supporting Information
- [Link to dashboard]
- [Link to logs]
- [Link to Slack thread]
```

---

## Communication During Incidents

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Incident Communication                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  INTERNAL (Slack/Teams)                                             │
│  ─────────────────────                                              │
│  - Dedicated incident channel                                       │
│  - Regular updates (every 15-30 min)                                │
│  - Clear status: investigating/identified/mitigating/resolved       │
│  - Tag stakeholders appropriately                                   │
│                                                                     │
│  EXTERNAL (Status Page)                                             │
│  ────────────────────────                                           │
│  - Update within 5 minutes of detection                             │
│  - Honest about impact                                              │
│  - ETA if known (or "investigating")                                │
│  - Follow-up when resolved                                          │
│                                                                     │
│  TEMPLATE:                                                          │
│  ──────────                                                         │
│  "[TIME] - Investigating elevated error rates affecting             │
│  [service]. Users may experience [impact]. We're actively           │
│  working to resolve. Next update in 30 minutes."                    │
│                                                                     │
│  "[TIME] - Root cause identified. Deploying fix.                    │
│  Expected resolution in [time]."                                    │
│                                                                     │
│  "[TIME] - Issue resolved. Service operating normally.              │
│  We'll publish a postmortem within 48 hours."                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Incident Response Checklist

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Response Checklist                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DETECTION                                                          │
│  □ Acknowledge alert                                                │
│  □ Assess severity                                                  │
│  □ Open incident channel                                            │
│                                                                     │
│  TRIAGE                                                             │
│  □ Assign incident commander                                        │
│  □ Page relevant teams                                              │
│  □ Start timeline documentation                                     │
│                                                                     │
│  INVESTIGATION                                                      │
│  □ Check recent changes (deploys, config)                           │
│  □ Check dashboards and logs                                        │
│  □ Check dependent services                                         │
│  □ Identify affected scope                                          │
│                                                                     │
│  MITIGATION                                                         │
│  □ Implement fix or workaround                                      │
│  □ Verify fix in staging (if possible)                              │
│  □ Deploy fix                                                       │
│  □ Verify recovery                                                  │
│                                                                     │
│  COMMUNICATION                                                      │
│  □ Update status page                                               │
│  □ Notify stakeholders                                              │
│  □ Regular updates every 15-30 min                                  │
│                                                                     │
│  RESOLUTION                                                         │
│  □ Confirm service healthy                                          │
│  □ Update status page: resolved                                     │
│  □ Stand down team                                                  │
│  □ Schedule postmortem                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Monitoring & Observability](./05-monitoring-observability.md) — инструменты для обнаружения и диагностики инцидентов
- [Capacity & Reliability](./07-capacity-reliability.md) — предотвращение инцидентов через планирование ёмкости

---

## На интервью

### Типичные вопросы

1. **Walk me through your incident response process**
   - Detection, Triage, Response, Resolution, Post-incident
   - Clear roles (IC, Tech Lead, Comms)
   - Communication cadence

2. **What makes a good postmortem?**
   - Blameless
   - Focus on systems, not people
   - Clear timeline
   - Actionable items with owners
   - Shared broadly

3. **How do you handle on-call?**
   - Weekly rotation
   - Clear escalation
   - Good runbooks
   - Track and reduce noise

4. **How do you prevent recurring incidents?**
   - Thorough postmortems
   - Action items tracked to completion
   - Automated prevention (tests, alerts)
   - Share learnings across teams

5. **Tell me about a major incident you handled**
   - Describe the situation
   - Your role and actions
   - What went well, what didn't
   - What you learned

6. **Blameless culture - why?**
   - Blame → hiding → slower detection
   - Trust enables honesty
   - Focus on systemic fixes, not punishment

---

[← Назад к списку тем](README.md)
