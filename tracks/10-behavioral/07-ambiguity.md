# 07. Dealing with Ambiguity

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Ambiguity Handling Signals                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Comfort with uncertainty                                         │
│  ✓ Problem structuring skills                                       │
│  ✓ Information gathering approach                                   │
│  ✓ Decision-making without complete information                     │
│  ✓ Balancing analysis vs. action                                    │
│  ✓ Scoping and prioritization                                       │
│  ✓ Communicating uncertainty to stakeholders                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a time you had to work with unclear requirements"

### STAR Template

```
SITUATION:
- What was unclear/ambiguous
- Why it was unclear (new domain, changing requirements, etc.)
- Stakes and timeline

TASK:
- What needed to be delivered
- Who was depending on it

ACTION:
- How you clarified/structured the problem
- How you made decisions without full information
- How you managed stakeholders through uncertainty

RESULT:
- What was delivered
- How close to actual need
- What you learned
```

### Пример ответа

```
"I was asked to 'improve our search' with minimal additional context.

SITUATION: Product said search was 'bad' but couldn't articulate
specifically what was wrong. Usage metrics showed declining search
usage, but no clear root cause. Leadership wanted it 'fixed' in Q3.

TASK: Define what 'improve search' means, then execute, with
limited time to investigate.

ACTION:
1. Structured the ambiguity
   - What could 'bad' mean? Speed? Relevance? UI? Coverage?
   - Created hypothesis list for each

2. Gathered data quickly
   - 5 user interviews (30 min each)
   - Analyzed search logs for patterns
   - Compared to competitor search behavior

3. Found patterns
   - Users: 'I can't find products I know exist'
   - Logs: 40% of searches return zero results
   - Root cause: search didn't handle typos or synonyms

4. Defined scope explicitly
   - Wrote one-pager: 'Search improvement = reduce zero-result
     searches from 40% to 10%'
   - Got sign-off from product before building
   - Made success measurable

5. Built incrementally
   - Week 1: Typo tolerance (biggest impact)
   - Week 2: Synonym handling
   - Week 3: Better relevance ranking
   - Each week: measure impact, adjust

RESULT:
- Zero-result rate: 40% → 8%
- Search usage increased 25%
- Stakeholders happy with outcome AND the process
- Created template: 'Clarify before commit'

Key learning: Ambiguity isn't a blocker - it's an invitation to
define the problem yourself. That's often the real work."
```

---

## "How do you handle changing requirements?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Handling Changing Requirements                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DON'T:                                                             │
│  - Complain that requirements changed                               │
│  - Silently absorb scope without adjusting timeline                 │
│  - Assume bad intent from product/stakeholders                      │
│                                                                     │
│  DO:                                                                │
│  - Expect change (it's normal, not exceptional)                     │
│  - Clarify impact: "Adding X means Y or Z tradeoff"                 │
│  - Document decisions and reasoning                                 │
│  - Build for flexibility where cost is low                          │
│  - Push back on change that doesn't make sense                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"Midway through a 3-month project, our main requirement changed
completely.

SITUATION: Building an inventory management system. Original
requirement: single warehouse. Month 2: 'Actually, we need to
support 5 warehouses across different regions.'

Multi-warehouse wasn't just more work - it required different
data model, different sync logic, different UI.

TASK: Adapt without throwing away work or missing deadline.

ACTION:
1. Assessed impact honestly
   - Current design: fundamentally single-warehouse
   - Multi-warehouse: 60% rework estimate
   - Documented the gap clearly

2. Understood WHY requirements changed
   - Business was acquiring a competitor with 4 warehouses
   - This was known but not communicated early
   - Changed conversation: 'How do we support acquisition?'

3. Proposed options
   A. Rebuild from scratch (4 months)
   B. Ship single-warehouse, then extend (3+2 months)
   C. Minimal multi-warehouse now, enhance later (3.5 months)

4. Recommended option C with rationale
   - Met acquisition timeline
   - Some technical debt, but manageable
   - Could iterate on experience

5. Prevented future surprises
   - Asked to be included in business planning discussions
   - 'If I'd known about acquisition, I'd have designed differently'
   - Not blaming, just fixing process

RESULT:
- Shipped multi-warehouse in 3.5 months
- Supported acquisition successfully
- Some tech debt we paid down in Q4
- Now included in strategic planning meetings

Key learning: Changing requirements often mean someone else has
information you don't. Find that information source."
```

---

## "Tell me about working in a new domain"

### Пример ответа

```
"I took on a project in healthcare domain with zero prior experience.

SITUATION: Our company was launching a patient scheduling system.
I was assigned as tech lead despite having no healthcare background.
Terms like 'HIPAA', 'HL7', 'provider credentialing' were foreign
to me.

TASK: Lead technical design and implementation for a compliant
healthcare product.

ACTION:
1. Acknowledged what I didn't know
   - Made list of 'known unknowns'
   - Didn't pretend expertise I didn't have
   - Asked 'stupid' questions early

2. Accelerated learning
   - Read 3 books on healthcare IT in 2 weeks
   - Took online HIPAA certification course
   - Interviewed 5 people at healthcare companies

3. Built expert network
   - Found internal expert (PM had healthcare background)
   - Engaged healthcare compliance consultant for reviews
   - Joined healthcare tech Slack community

4. Designed for what I didn't know
   - Built abstraction layers for healthcare-specific logic
   - Made compliance requirements configurable
   - Created 'compliance checkpoint' in every sprint review

5. Validated continuously
   - Showed designs to healthcare experts early
   - 'What am I missing?' as standing question
   - Found several critical gaps this way

RESULT:
- Launched compliant product on schedule
- Passed external security audit first try
- No HIPAA incidents in first year
- I became the team's healthcare tech expert

Key learning: You don't need to be a domain expert to work in a
new domain. You need to be good at learning and know when to
seek expertise."
```

---

## "How do you prioritize when everything is important?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Prioritization Framework                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                     High Urgency        Low Urgency                 │
│                  ┌─────────────────┬─────────────────┐              │
│    High Impact   │  DO FIRST       │  SCHEDULE       │              │
│                  │  (Crisis,        │  (Strategic     │              │
│                  │   Deadlines)     │   initiatives)  │              │
│                  ├─────────────────┼─────────────────┤              │
│    Low Impact    │  DELEGATE or    │  ELIMINATE or   │              │
│                  │  QUICK WIN      │  DEFER          │              │
│                  │                 │                 │              │
│                  └─────────────────┴─────────────────┘              │
│                                                                     │
│  Questions to ask:                                                  │
│  - What's the cost of NOT doing this?                               │
│  - What's the cost of delay?                                        │
│  - Who else is blocked?                                             │
│  - Is this reversible?                                              │
│  - What's the minimum viable version?                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I joined a team with 15 'P0 critical' projects and was told to
pick what to work on.

SITUATION: Previous tech lead had left, backlog was enormous,
everything was labeled critical by different stakeholders. Team
was paralyzed - working on everything, finishing nothing.

TASK: Establish clear priority and focus the team.

ACTION:
1. Refused to accept 'everything is P0'
   - Met with each stakeholder: 'If you could only have ONE thing,
     what would it be?'
   - 15 'critical' items → 5 truly critical, 10 important

2. Created forcing function
   - 'We can complete 3 things this quarter. Which 3?'
   - Made stakeholders prioritize against each other
   - I facilitated, didn't decide

3. Established criteria
   - Revenue impact (quantified)
   - Customer pain (NPS, support tickets)
   - Technical risk (security, stability)
   - Dependencies (blocking other teams)

4. Made priorities visible
   - Published ranked list
   - Explained WHY for each ranking
   - Stakeholders could see their item and why it was where it was

5. Protected focus
   - New requests evaluated against existing priorities
   - 'Yes, we can do X. What should we drop?'
   - Said no with explanation, not just rejection

RESULT:
- Completed 4 major projects (up from ~1 in previous quarters)
- Stakeholder satisfaction improved (clear expectations)
- Team morale improved (sense of progress)
- Prioritization process adopted by other teams

Key insight: If everything is priority 1, nothing is. Your job is
to force clarity."
```

---

## "Tell me about scoping a large/vague project"

### Пример ответа

```
"I was asked to 'modernize our data pipeline' - a massive, vague scope.

SITUATION: Our data pipeline was 5 years old, different teams
complained about different problems. No clear definition of 'modern'
or 'done'. Previous attempt failed after 8 months with nothing shipped.

TASK: Scope this into something achievable and valuable.

ACTION:
1. Decomposed the problem
   - Listed all complaints/issues (30+ items)
   - Grouped by theme: speed, reliability, observability, cost
   - Mapped dependencies between issues

2. Found the 80/20
   - Interviewed data consumers: 'What's your biggest pain?'
   - Answer: 'Jobs fail silently, we don't know until data is wrong'
   - Single issue caused 70% of the pain

3. Defined success narrowly
   - 'Modern pipeline' → 'Pipeline with <5% silent failures'
   - Measurable, achievable, valuable
   - Got sign-off on this definition

4. Created phases
   - Phase 1: Monitoring and alerting (6 weeks)
   - Phase 2: Self-healing for common failures (8 weeks)
   - Phase 3: Re-evaluate remaining needs

5. Built incrementally
   - Shipped monitoring after week 3
   - Immediate value: could now SEE failures
   - Trust built → more time/resources for next phases

RESULT:
- Phase 1 alone reduced data quality incidents by 60%
- Phase 2 further reduced to 90%
- Phase 3 wasn't needed - remaining issues were minor
- Total: 14 weeks, delivered value continuously

Key learning: 'Modernization' projects fail when scoped as
modernization. Scope as specific problems to solve."
```

---

## "How do you make decisions with limited information?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│              Decision Making Under Uncertainty                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. DETERMINE: What TYPE of decision is this?                       │
│     - One-way door (hard to reverse) → more analysis                │
│     - Two-way door (easy to change) → bias toward action            │
│                                                                     │
│  2. DEFINE: What information would change the decision?             │
│     - If nothing would change it → decide now                       │
│     - If something would → go get that specific info                │
│                                                                     │
│  3. TIMEBOX: How long can you wait?                                 │
│     - Cost of delay vs. cost of wrong decision                      │
│     - Often waiting costs more than being slightly wrong            │
│                                                                     │
│  4. DECIDE: Make explicit choice                                    │
│     - Document assumptions                                          │
│     - Define what would make you reconsider                         │
│     - Commit and execute                                            │
│                                                                     │
│  5. LEARN: Revisit and adjust                                       │
│     - Was the decision right?                                       │
│     - What did you learn?                                           │
│     - Update mental models                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I had to choose our message queue technology without knowing our
actual scale requirements.

SITUATION: Greenfield project, unknown traffic. Could be 1K/sec or
100K/sec. Different solutions optimal at different scales. Product
wanted decision in 1 week.

TASK: Choose message queue that wouldn't be catastrophically wrong
regardless of actual scale.

ACTION:
1. Identified what I DID know
   - Message patterns (events, not RPC)
   - Persistence needed (yes)
   - Team expertise (Kafka: 2 people, RabbitMQ: 4)

2. Identified what I DIDN'T know
   - Actual traffic
   - Peak patterns
   - Growth trajectory

3. Reframed the question
   - Not 'what's optimal?' but 'what's safe?'
   - What's the cost of being wrong?

4. Analyzed reversibility
   - Queue migration is painful but possible
   - Data loss during migration is the real risk
   - Abstraction layer could reduce migration cost

5. Made pragmatic decision
   - RabbitMQ: team knows it, scales to 10K/sec easily
   - Design: abstracted behind interface
   - Trigger: if we hit 5K/sec sustained, evaluate Kafka
   - Documented assumptions and decision

RESULT:
- 18 months later: stable at 3K/sec
- Never needed to migrate
- Abstraction layer helped when we added second queue type
- Decision documented, easy to explain to new team members

Key learning: Perfect information is expensive. 'Good enough' decisions
with clear reversibility plans usually win."
```

---

## Red Flags

```
❌ Mistakes to avoid:

1. Paralysis
   "I waited for requirements to be finalized"
   → Requirements are never finalized

2. Blame product/stakeholders
   "Product didn't know what they wanted"
   → It's partly your job to help clarify

3. Didn't document assumptions
   "We just started building"
   → How do you know when assumptions are wrong?

4. Over-engineering for uncertainty
   "We built it to handle anything"
   → Flexibility has cost too

5. Under-communicating uncertainty
   "We said we'd deliver X"
   → Stakeholders should know confidence level

6. Not seeking clarification
   "We just guessed"
   → Should exhaust reasonable information sources first
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "How do you know when you have enough information to decide?"
- "What do you do when stakeholders can't clarify requirements?"
- "How do you communicate uncertainty to non-technical stakeholders?"
- "Tell me about a time your assumptions were wrong. What happened?"
- "How do you balance analysis with getting things done?"
- "What's your process for breaking down a big ambiguous problem?"
```

---

## Communicating Uncertainty

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Communicating Uncertainty                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DON'T say:              DO say:                                    │
│  ──────────              ───────                                    │
│  "It'll take 3 weeks"    "3-4 weeks, depending on X"                │
│  "I'm not sure"          "I'm 70% confident because of Y"           │
│  "Maybe"                 "If A, then X. If B, then Y"               │
│  "I don't know"          "I don't know yet. I'll find out by Z"     │
│                                                                     │
│  Techniques:                                                        │
│  - Use ranges instead of points                                     │
│  - Quantify confidence (%, likelihood)                              │
│  - State assumptions explicitly                                     │
│  - Define triggers for re-evaluation                                │
│  - Separate what you know from what you believe                     │
│                                                                     │
│  Example:                                                           │
│  "Based on similar projects, I estimate 4-6 weeks.                  │
│   The main uncertainty is the third-party API - if it               │
│   works as documented, we're at 4 weeks. If we need                 │
│   workarounds, closer to 6. I'll know more after the                │
│   spike next week and will update the estimate."                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

[← Назад к списку тем](README.md)
