# 08. Delivery & Execution

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Delivery & Execution Signals                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Getting things done                                              │
│  ✓ Managing timelines and expectations                              │
│  ✓ Making scope/quality/time trade-offs                             │
│  ✓ Unblocking yourself and others                                   │
│  ✓ Operational excellence                                           │
│  ✓ Shipping iteratively                                             │
│  ✓ Handling pressure                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a time you delivered under a tight deadline"

### STAR Template

```
SITUATION:
- What was the deadline and why
- What were the constraints
- What was at stake

TASK:
- What needed to be delivered
- Your role in delivery

ACTION:
- How you planned/organized
- What trade-offs you made
- How you managed pressure
- How you kept quality

RESULT:
- What was delivered
- Was deadline met
- What did you learn
```

### Пример ответа

```
"I had to deliver a payment integration in 3 weeks that was
originally scoped for 6 weeks.

SITUATION: Our largest enterprise client needed Stripe integration
for their launch. Contract was signed with specific launch date.
Timeline was cut in half due to delayed contract negotiations.

TASK: Deliver a secure, reliable payment integration that could
process $1M+/day in transactions.

ACTION:
1. Ruthlessly scoped
   - Listed all features in original spec
   - Separated 'must have for launch' vs 'nice to have'
   - Cut: multiple currency, subscription billing, admin dashboard
   - Kept: one-time payments, refunds, basic reporting

2. Reduced uncertainty fast
   - Day 1: Spike on Stripe API integration
   - Day 2: Spike on our billing system integration
   - Day 3: Had realistic estimate of remaining work

3. Parallelized work
   - Split team: 2 on core integration, 1 on testing infrastructure
   - I handled stakeholder communication + filled gaps

4. Maintained quality where it mattered
   - Did NOT cut security review or load testing
   - DID cut UI polish, documentation, nice error messages

5. Communicated constantly
   - Daily status to client and leadership
   - 'Green/Yellow/Red' with specific risks
   - No surprises - they knew exactly where we were

RESULT:
- Shipped core integration on Day 19 (2 days buffer)
- Processed first real transaction on launch day
- Zero security incidents
- Added cut features over next 3 months

Key learning: Tight deadlines are about scope, not heroics.
Cutting the right things is the skill."
```

---

## "Tell me about a time you had to cut scope"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Scope Cutting Framework                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. UNDERSTAND THE CORE VALUE                                       │
│     - What's the ONE thing that must work?                          │
│     - What's the minimum viable solution?                           │
│                                                                     │
│  2. CATEGORIZE RUTHLESSLY                                           │
│     - Must have: literally cannot launch without                    │
│     - Should have: significant value, but survivable                │
│     - Nice to have: polish, improvements                            │
│                                                                     │
│  3. COMMUNICATE TRADE-OFFS                                          │
│     - What we're cutting                                            │
│     - Why it's the right cut                                        │
│     - What we'll lose                                               │
│     - When we'll add it back                                        │
│                                                                     │
│  4. CUT CLEANLY                                                     │
│     - Don't half-build cut features                                 │
│     - Remove completely or keep completely                          │
│     - Technical debt from half-done is worse                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I had to recommend cutting a major feature two weeks before launch.

SITUATION: Building a marketplace app. 'Saved searches' feature
was in scope but taking longer than expected. Team was working
overtime trying to ship everything.

TASK: Decide whether to push team harder, delay launch, or cut scope.

ACTION:
1. Assessed realistically
   - Saved searches: 2 more weeks needed
   - Team: already exhausted, quality declining
   - Launch delay: would miss holiday shopping season

2. Analyzed feature importance
   - Looked at comparable apps: saved searches is 'nice to have'
   - Our analytics: search → buy journey was fine without it
   - Customer interviews: 'helpful but not critical'

3. Made the recommendation
   - Presented to product: 'We should cut saved searches'
   - Showed data on usage from competitors
   - Proposed: launch without it, add in January

4. Handled the reaction
   - Product initially pushed back ('we promised stakeholders')
   - I offered to join stakeholder call to explain technical reality
   - Framed as: 'Launch a solid product now vs. delayed buggy product'

5. Executed the cut
   - Removed partial saved searches code (cleaner codebase)
   - Reallocated 1 engineer to polish core features
   - Used time buffer for more testing

RESULT:
- Launched on time with higher quality
- Zero critical bugs at launch
- Added saved searches in January
- Nobody complained about missing feature

Key insight: Saying no to features is often more valuable than
saying yes. What you don't build matters."
```

---

## "How do you handle blockers?"

### Пример ответа

```
"I was blocked on a critical API from a team that was unresponsive.

SITUATION: Needed access to user profile API for our feature.
API team was overwhelmed with other priorities. My requests sat
in their backlog for 2 weeks. Launch was in 3 weeks.

TASK: Get unblocked without causing cross-team conflict or making
enemies.

ACTION:
1. Tried normal channels first
   - Slack message (no response)
   - Email to team lead (acknowledged, no action)
   - Jira ticket (sat in backlog)

2. Understood their constraint
   - Had coffee with their tech lead
   - They were firefighting production issues
   - Our request was 'important not urgent' for them

3. Made it easy
   - Wrote the API spec myself
   - Offered to pair program on implementation
   - 'I'll do 80% of the work, you just review'

4. Escalated appropriately
   - After 2.5 weeks, looped in both managers
   - Not as 'they're blocking me' but 'we have a priority conflict
     that needs leadership alignment'
   - Data: if not resolved by date X, we miss launch

5. Had backup plan
   - Designed feature to work without their API (degraded)
   - Could ship with workaround if needed

RESULT:
- Their manager freed up an engineer to help
- We finished together in 3 days
- I bought them coffee afterward
- Relationship strengthened, not damaged

Key learning: Unblocking yourself is your responsibility.
Waiting and complaining is not a strategy."
```

---

## "Tell me about managing a complex project"

### Пример ответа

```
"I led a system migration with 20+ components and zero tolerance
for downtime.

SITUATION: Migrating our core platform from AWS to GCP. 50+ services,
100TB data, processing $2M/day. Business said: 'no downtime, no
data loss, you figure out how.'

TASK: Plan and execute migration while system stayed operational.

ACTION:
1. Structured the complexity
   - Created component dependency graph
   - Identified critical path (8 services had to move together)
   - Found parallel workstreams (20 services could move independently)

2. Built confidence incrementally
   - Started with non-critical services
   - Each migration taught us something
   - Built runbooks and tooling iteratively

3. Created observability
   - Dashboard showing migration status
   - Real-time comparison: AWS metrics vs GCP metrics
   - Anomaly detection for data discrepancies

4. Planned for failure
   - Every component had rollback procedure
   - Tested rollbacks before migration, not during
   - Team drills: 'What if X fails? Who does what?'

5. Communicated status
   - Weekly migration update to all stakeholders
   - 'We're here on the timeline, this is next, here are risks'
   - No surprises - everyone knew the plan

6. Coordinated execution
   - Migration weekend: 12 people, clear roles
   - Checklist-driven execution
   - Decision tree for issues

RESULT:
- Completed migration in 4 months
- Zero customer-facing downtime
- Zero data loss
- $200K/month cost savings
- Post-migration stability actually improved

Key insight: Complex projects are about planning for what goes
wrong, not just what goes right."
```

---

## "How do you manage expectations?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Expectation Management                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  EARLY AND OFTEN                                                    │
│  - Set expectations at project start                                │
│  - Update regularly, not just when things change                    │
│  - Bad news early > bad news late                                   │
│                                                                     │
│  BE SPECIFIC                                                        │
│  - 'Done by Friday' vs 'Done by EOD Friday, 80% confidence'         │
│  - Include assumptions                                              │
│  - Define what 'done' means                                         │
│                                                                     │
│  UNDER-PROMISE, OVER-DELIVER                                        │
│  - Add buffer to estimates (not padding, realism)                   │
│  - Account for unknowns explicitly                                  │
│  - Communicate early if ahead/behind                                │
│                                                                     │
│  MANAGE SCOPE, NOT JUST TIME                                        │
│  - 'We can do A by X, or A+B by Y'                                  │
│  - Trade-offs, not just delays                                      │
│  - Give stakeholders choices                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I had to reset expectations on a project that was going to miss
its deadline.

SITUATION: 8-week project, end of week 5, team was ~40% done. At
current pace, we'd miss by 4 weeks. I hadn't communicated this
risk clearly yet.

TASK: Reset expectations without losing credibility.

ACTION:
1. Acknowledged my mistake
   - Should have raised flag earlier
   - Didn't cover up or blame team

2. Came with data, not excuses
   - Here's what we estimated
   - Here's actual velocity
   - Here's why (3 specific issues)

3. Presented options, not just problems
   A. Extend 4 weeks (all features)
   B. Cut features X and Y (2 week extension)
   C. Add 2 engineers (unknown impact, risky)

4. Made recommendation
   - Option B: delivers core value, realistic timeline
   - Explained reasoning

5. Owned the recovery plan
   - 'Here's how we'll track progress going forward'
   - Weekly updates with velocity metrics
   - Clear milestones, not just end date

RESULT:
- Stakeholders chose Option B
- Delivered 2 weeks late (vs 4)
- Trust was maintained because of transparency
- Implemented better tracking from project start

Key learning: Managing expectations is about proactive communication.
By the time something is a surprise, you've already failed."
```

---

## "Tell me about shipping iteratively"

### Пример ответа

```
"I shifted our team from big-bang releases to continuous delivery.

SITUATION: Team had habit of 'saving up' features for quarterly
releases. Releases were painful: 3-day testing cycles, frequent
rollbacks, high stress.

TASK: Move to more frequent, smaller releases.

ACTION:
1. Started with myself
   - My next feature: shipped in 3 increments
   - Showed it was possible and safe

2. Created safety net
   - Improved CI/CD pipeline
   - Feature flags for incomplete features
   - Automated smoke tests
   - 1-click rollback

3. Changed the culture
   - 'What's the smallest shippable increment?'
   - Daily standups: 'What's blocking deployment?'
   - Celebrated small frequent deploys

4. Measured and shared progress
   - Tracked deploy frequency
   - Tracked rollback rate
   - Tracked time-to-recover
   - All improved

5. Handled resistance
   - Some engineers liked big releases ('more satisfying')
   - Showed data: small = safer
   - Let them see benefits themselves

RESULT:
- Deploy frequency: monthly → daily
- Rollback rate: 20% → 3%
- Time to recover: 4 hours → 15 minutes
- Team stress around releases eliminated

Key insight: Shipping small and often is safer than shipping big
and rarely. More deploys = less risk per deploy."
```

---

## "How do you work under pressure?"

### Пример ответа

```
"During a major production incident, I had to stay calm while
coordinating recovery.

SITUATION: Database corruption affecting 10% of users. CEO watching,
support overwhelmed, engineers panicking. 2 AM on a Sunday.

TASK: Coordinate incident response and get service restored.

ACTION:
1. Took control calmly
   - 'Everyone stop. Let's structure this.'
   - Not panicking gave others permission to not panic

2. Established structure
   - Incident commander: me
   - Technical lead: senior engineer
   - Communications: product manager
   - Clear roles, no confusion

3. Made decisions systematically
   - First: stop the bleeding (disable writes)
   - Second: assess damage (which data affected?)
   - Third: recovery plan
   - Fourth: root cause (AFTER recovery, not during)

4. Communicated crisply
   - Every 15 minutes: status update
   - 'Here's what we know. Here's what we're doing. Here's ETA.'
   - No speculation, no blame

5. Took care of people
   - Ordered food (it was 2 AM)
   - Let tired people tap out
   - 'We'll handle this, you rest'

RESULT:
- Full recovery in 4 hours
- Data loss: 0.02% of transactions (reconciled manually later)
- Postmortem identified root cause, fixed
- Team thanked me for keeping them calm

Key learning: Under pressure, slow down. Panic is contagious, but
so is calm. Your job is to bring calm."
```

---

## Red Flags

```
❌ Mistakes to avoid:

1. Heroics as strategy
   "I worked 80-hour weeks to deliver"
   → Should have managed scope or timeline

2. Blaming external factors
   "We missed because of X team"
   → What did YOU do to mitigate?

3. No trade-off discussion
   "We delivered everything on time"
   → Always trade-offs, be honest

4. Chaos as normal
   "We always work like this"
   → Should be improving process

5. Quality sacrificed
   "We shipped but it was buggy"
   → Quality is part of delivery

6. No reflection
   "We delivered, next project"
   → What did you learn?
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "What would you have done differently?"
- "How did you make sure quality didn't suffer?"
- "How did you prioritize what to cut?"
- "How do you estimate accurately?"
- "What do you do when things slip?"
- "How do you keep the team motivated under pressure?"
```

---

## Delivery Anti-Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Delivery Anti-Patterns                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ANTI-PATTERN              WHAT TO DO INSTEAD                       │
│  ─────────────             ────────────────────                     │
│                                                                     │
│  90% done for weeks        Define 'done' clearly upfront            │
│                                                                     │
│  Estimate then pad         Estimate with uncertainty range          │
│                                                                     │
│  Saving up changes         Ship small, ship often                   │
│                                                                     │
│  Last-minute testing       Test continuously                        │
│                                                                     │
│  Hero culture              Sustainable pace, no surprises           │
│                                                                     │
│  Scope creep without       Every addition trades off something      │
│  timeline change                                                    │
│                                                                     │
│  Quality vs speed          False dichotomy - both or neither        │
│  trade-off                                                          │
│                                                                     │
│  "Just ship it"            Ship the right thing, right way          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

[← Назад к списку тем](README.md)
