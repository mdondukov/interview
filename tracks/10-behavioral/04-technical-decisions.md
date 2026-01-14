# 04. Technical Decisions

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Technical Decision Signals                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Structured decision-making process                               │
│  ✓ Trade-off analysis                                               │
│  ✓ Data-driven approach                                             │
│  ✓ Stakeholder communication                                        │
│  ✓ Long-term thinking                                               │
│  ✓ Handling uncertainty                                             │
│  ✓ Knowing when to decide vs. gather more info                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a difficult technical decision you made"

### STAR Template

```
SITUATION:
- Business context
- Technical landscape
- Why decision was needed

TASK:
- What needed to be decided
- Constraints (time, resources, expertise)
- Stakes

ACTION:
- Options you considered
- How you evaluated each
- Who you involved
- How you reached the decision

RESULT:
- Outcome
- What worked / didn't work
- What you'd do differently
```

### Пример ответа

```
"I had to decide whether to rewrite our payment processing system
or continue patching the legacy system.

SITUATION: Our 7-year-old payment system had become a maintenance
nightmare - 30% of engineering time went to bug fixes, and we
couldn't add new payment methods without months of work. The
business wanted to expand to 5 new countries.

TASK: Recommend build-new vs. fix-existing, with full justification,
within 2 weeks. Wrong choice could cost 6+ months of wasted effort.

ACTION:
Options evaluated:
1. Big-bang rewrite (estimated 8 months)
2. Incremental strangler pattern (estimated 12 months)
3. Continue patching + hire more engineers

My process:
- Created decision matrix with criteria: time-to-market, risk,
  long-term maintainability, team morale, cost
- Interviewed 5 senior engineers on each option
- Did spike: 1 week prototyping strangler approach on one component
- Calculated TCO: patching actually more expensive over 2 years
- Talked to other companies who did similar migrations

Decision: Strangler pattern, even though slower initially
- Lower risk than big-bang
- Could deliver incremental value
- Team could learn as we went

RESULT:
- First new country launched in 4 months (vs. 8 for rewrite)
- Full migration completed in 14 months (2 months over estimate)
- Maintenance time dropped from 30% to 10%
- Decision framework became template for future major decisions

Key insight: When the decision is complex, invest in the decision
process itself. The spike saved us from a potential big-bang
disaster."
```

---

## "How do you evaluate build vs. buy?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Build vs. Buy Framework                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BUILD when:                    BUY when:                           │
│  ─────────────                  ───────────                         │
│  Core differentiator            Commodity functionality             │
│  Unique requirements            Standard problem                    │
│  You have expertise             Missing expertise                   │
│  Long-term investment           Time-sensitive                      │
│  Data/security concerns         Trusted vendors exist               │
│                                                                     │
│  Evaluation criteria:                                               │
│  ┌────────────────┬──────────────┬─────────────────┐               │
│  │                │    BUILD     │      BUY        │               │
│  ├────────────────┼──────────────┼─────────────────┤               │
│  │ Initial cost   │    High      │    Low-Med      │               │
│  │ Ongoing cost   │    Medium    │    Medium-High  │               │
│  │ Time to value  │    Long      │    Short        │               │
│  │ Customization  │    Full      │    Limited      │               │
│  │ Maintenance    │    Internal  │    Vendor       │               │
│  │ Risk           │    Higher    │    Lower        │               │
│  │ Control        │    Full      │    Limited      │               │
│  └────────────────┴──────────────┴─────────────────┘               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"We needed to decide whether to build our own feature flag system
or use a vendor like LaunchDarkly.

SITUATION: Product wanted to do more A/B testing and gradual
rollouts. We had a basic internal system, but it lacked targeting,
analytics, and multi-environment support.

TASK: Recommend solution within budget ($50K/year max).

ACTION:
I created a comparison:

Build:
- Estimated 3 engineer-months ($45K fully loaded)
- Ongoing: 0.5 FTE maintenance (~$75K/year)
- Full customization
- 4-6 months before feature-complete

Buy (LaunchDarkly):
- $30K/year at our scale
- Operational in 1 week
- Industry-standard features
- Less customizable

Key questions I asked:
1. Is feature flagging our core differentiator? No
2. Do we have special requirements? Not really
3. What's the cost of delay? 4 months of no A/B testing
4. Build expertise: We'd need to learn best practices

RESULT:
Recommended buy. The math was clear: $30K/year vs. $75K/year
internal + opportunity cost. We bought, integrated in 2 weeks,
and immediately started experimenting.

Lesson: Buy boring infrastructure, build differentiating features.
Your time is better spent on what makes your product unique."
```

---

## "Tell me about a time you had to make a decision with incomplete information"

### Пример ответа

```
"I had to decide our database strategy for a new service before
we knew what our actual load patterns would be.

SITUATION: We were building a new analytics service. Projected
scale varied wildly: could be 1K QPS or 100K QPS depending on
customer adoption. Different scales called for different solutions.

TASK: Choose database architecture that wouldn't require complete
rewrite regardless of outcome.

ACTION:
My approach to uncertainty:
1. Identify what I DO know:
   - Read-heavy workload (90/10)
   - Time-series access pattern
   - Can tolerate eventual consistency

2. Identify decision points:
   - At 10K QPS: need read replicas
   - At 50K QPS: need sharding
   - At 100K QPS: need specialized time-series DB

3. Design for change:
   - Started with PostgreSQL (familiar, flexible)
   - Abstracted data layer with repository pattern
   - Set up monitoring for scale triggers
   - Documented migration paths for each scenario

4. Define decision triggers:
   - 'If we hit X, we'll evaluate Y'
   - Avoided premature optimization

RESULT:
- Started simple with PostgreSQL
- After 6 months, hit 15K QPS - added read replicas
- After 12 months, stable at 20K QPS - no need for sharding
- Saved months of premature work on complex architecture

Key learning: Make reversible decisions quickly, invest more time
in irreversible ones. Document your assumptions so future-you
knows what to revisit."
```

---

## "How do you convince others of a technical decision?"

### Framework

```
1. UNDERSTAND THEIR CONCERNS
   - What are they worried about?
   - What's their context?
   - What do they value?

2. BUILD SHARED UNDERSTANDING
   - Agree on the problem first
   - Establish evaluation criteria together
   - Use data they trust

3. PROPOSE, DON'T IMPOSE
   - Present as recommendation, not mandate
   - Invite challenges
   - Be open to being wrong

4. ADDRESS OBJECTIONS
   - Don't dismiss concerns
   - Acknowledge trade-offs
   - Propose mitigations

5. OFFER PROOF
   - POC or spike
   - Pilot program
   - Reversible experiment
```

### Пример ответа

```
"I wanted to introduce Kubernetes, but faced resistance from ops
and skepticism from management.

SITUATION: We were running on VMs, deployments took hours, and
scaling was manual. I believed K8s would solve these problems,
but ops team had concerns about complexity, and management
worried about the learning curve.

TASK: Build consensus for K8s adoption.

ACTION:
Understanding concerns:
- Ops: 'K8s is complex, will increase on-call burden'
- Management: 'We'll lose 3 months to migration'
- Dev team: 'Why change something working?'

My approach:
1. Quantified current pain: 15 hours/week on deployment issues,
   3 incidents/month from manual scaling
2. Addressed ops concerns: proposed training budget, gradual
   migration, keeping VM option as fallback
3. Proposed pilot: one low-risk service, 4 weeks, clear success
   criteria
4. Documented rollback plan: if it fails, we go back, lessons learned
5. Found internal champion in ops team who was curious about K8s

RESULT:
- Pilot succeeded: deployment time 2 hours → 5 minutes
- Ops engineer became K8s advocate after training
- Gradual rollout over 6 months
- Eventually all services on K8s

Key insight: Selling technical change is about addressing fears,
not just presenting benefits. 'What could go wrong?' matters more
than 'What could go right?' to risk-averse stakeholders."
```

---

## "Describe a technical trade-off you had to make"

### Common Trade-offs

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Common Technical Trade-offs                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Speed vs Quality           Monolith vs Microservices               │
│  Build vs Buy               SQL vs NoSQL                            │
│  Consistency vs Availability  Sync vs Async                         │
│  Simplicity vs Flexibility    DRY vs Readability                    │
│  Optimize vs Ship           Custom vs Standard                      │
│                                                                     │
│  There's no right answer - only contextually appropriate ones       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I had to choose between eventual consistency and strong consistency
for our inventory system.

SITUATION: E-commerce platform, 50K SKUs, multiple warehouses.
Product wanted real-time inventory display. Engineering concerned
about performance at scale.

TRADE-OFF:
- Strong consistency: always accurate, but 3x latency, complex
  distributed locking
- Eventual consistency: fast, simpler, but could show stale data
  (risk: overselling)

ACTION:
I analyzed the actual business impact:
1. How often do we oversell? Checked: 0.1% of orders
2. Cost of oversell: $50 avg to apologize and compensate
3. Cost of slow page: 2% conversion drop per 100ms

Calculated:
- Eventual: $50 × 0.1% = $0.05/order in oversell cost
- Strong: 2% conversion loss × $30 AOV = $0.60/order

Decision: Eventual consistency with safeguards:
- 'Low stock' warning at <10 items
- Final check at checkout (strong consistency only there)
- Fast reconciliation job (every 5 mins)

RESULT:
- Page load 50ms faster
- Overselling dropped to 0.02% with safeguards
- Best of both worlds

Key learning: Trade-offs aren't always 'pick one'. Sometimes you
can find hybrid solutions that capture most benefits of both."
```

---

## "How do you make decisions under time pressure?"

### Пример ответа

```
"During a production incident, I had to decide between a quick
fix that might cause data inconsistency vs. a proper fix that
would extend downtime.

SITUATION: Our order processing was stuck. 500 orders queued,
customers waiting. I identified two paths:
- Path A: Skip validation, process immediately (5 min)
- Path B: Fix validation bug properly (45 min)

TASK: Minimize customer impact while not creating bigger problems.

ACTION:
My rapid decision process:
1. What's the blast radius?
   - Path A: potentially 500 orders with bad data, cleanup needed
   - Path B: 45 min delay for 500 customers

2. Is it reversible?
   - Path A: Data issues are hard to fix
   - Path B: Delay is temporary, no lasting damage

3. What do we know vs. assume?
   - I knew validation bug affected 5% of orders
   - I didn't know which 5%

4. Who else should input?
   - Quick Slack to senior engineer: 'sanity check, 30 seconds?'
   - She agreed: proper fix, communicate to customers

Decision: Path B with immediate customer communication

RESULT:
- 45 min fix, zero data issues
- Customer support sent proactive apology with discount code
- Would have taken 3 days to clean up bad data from Path A

Framework I use under pressure:
1. Reversible? Bias toward caution if not
2. Blast radius? Smaller = more risk tolerance
3. 30-second sanity check if possible
4. Decide and commit - indecision is also a cost"
```

---

## Technical Decision Document (ADR)

```markdown
# ADR-001: [Decision Title]

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Context
[What is the issue we're facing? Business and technical context]

## Decision Drivers
- [Driver 1]
- [Driver 2]

## Considered Options
1. [Option 1]
2. [Option 2]
3. [Option 3]

## Decision
[Which option was chosen and why]

## Consequences

### Positive
- [Benefit 1]
- [Benefit 2]

### Negative
- [Trade-off 1]
- [Trade-off 2]

### Risks
- [Risk 1 and mitigation]

## Links
- [Related ADRs]
- [Relevant documentation]
```

---

## См. также

- [Architecture Fundamentals](../08-architecture/00-architecture-fundamentals.md) — основы архитектурных решений
- [Leadership](./01-leadership.md) — влияние и убеждение в технических решениях

---

## Red Flags

```
❌ Mistakes to avoid:

1. Decision without trade-off discussion
   "We chose X because it's the best"
   → What did you give up?

2. Not involving stakeholders
   "I decided and told the team"
   → Good decisions need buy-in

3. Analysis paralysis
   "We're still evaluating options after 3 months"
   → Sometimes you need to decide with imperfect info

4. Not documenting decisions
   "We went with X, I forget why"
   → ADRs prevent reinventing the wheel

5. Ignoring reversibility
   One-way doors need more analysis than two-way doors

6. No success criteria
   "We'll see how it goes"
   → How will you know if the decision was right?
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "Looking back, would you make the same decision?"
- "How did you handle disagreement about the decision?"
- "What were the biggest risks? How did you mitigate them?"
- "How did you measure if the decision was successful?"
- "What would you do if new information contradicted your decision?"
- "How do you know when you have enough information to decide?"
```

---

[← Назад к списку тем](README.md)
