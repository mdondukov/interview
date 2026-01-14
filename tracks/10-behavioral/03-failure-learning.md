# 03. Failure & Learning

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Failure & Learning Signals                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Self-awareness                                                   │
│  ✓ Ownership and accountability                                     │
│  ✓ Growth mindset                                                   │
│  ✓ Learning from mistakes                                           │
│  ✓ Resilience                                                       │
│  ✓ Honesty and humility                                             │
│  ✓ Preventing future failures                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a time you failed"

### STAR Template

```
SITUATION:
- Контекст проекта/задачи
- Stakes (почему это было важно)
- Your role and responsibility

TASK:
- Что нужно было достичь
- Какие были constraints

ACTION:
- Что пошло не так (конкретно)
- Ваша роль в failure
- Как отреагировали
- Что сделали для исправления

RESULT:
- Impact неудачи
- Что изменили в процессе
- Что узнали
- Как это повлияло на будущее
```

### Пример ответа

```
"I led a project to migrate our user authentication system that
resulted in a 4-hour outage affecting 50,000 users.

SITUATION: We were migrating from a legacy auth system to OAuth 2.0.
This was critical infrastructure - if auth is down, nobody can use
the product. I was the tech lead responsible for the migration plan.

TASK: Complete migration with zero downtime using a gradual rollout.

ACTION:
- I designed a migration plan with feature flags and gradual rollout
- However, I missed a critical edge case: users with special
  characters in their usernames wouldn't migrate correctly
- I had tested with 'typical' usernames, but not edge cases
- When we rolled out to 10%, the bug hit users who couldn't log in
- Because I hadn't set up proper rollback procedures, it took us
  4 hours to identify, fix, and recover

What I did wrong:
1. Insufficient test coverage (happy path only)
2. No automated rollback mechanism
3. Rolled out to 10% instead of 1% first
4. On-call engineers didn't have clear runbooks

RESULT:
- 4-hour partial outage, ~50K affected users
- I wrote a detailed postmortem and presented it to the engineering org
- We implemented mandatory rollback procedures for all migrations
- I created a 'migration checklist' that's now used company-wide
- I personally apologized to the on-call engineers I put in a tough spot

Most importantly, I learned that 'move fast' without safety nets
is just reckless. Now I always ask: 'What if this goes wrong?'
before shipping anything critical."
```

---

## "Tell me about a mistake you made"

### Types of Mistakes

```
Technical mistakes:
- Wrong architecture choice
- Missing edge cases
- Performance issues
- Security vulnerabilities

Process mistakes:
- Poor estimation
- Unclear requirements
- Skipped testing
- Missing communication

People mistakes:
- Micromanaging
- Not delegating
- Poor feedback delivery
- Misreading situations
```

### Пример ответа

```
"I made a significant mistake early in my career by choosing to
build a custom caching solution instead of using Redis.

SITUATION: We needed caching for a high-traffic feature. I was
excited about the technical challenge and convinced the team to
build our own solution - I argued it would be 'more tailored to
our needs.'

TASK: Implement caching to reduce database load by 80%.

ACTION:
- Spent 3 weeks building a custom in-memory cache
- It worked in development, but in production:
  * Memory leaks we hadn't anticipated
  * No cluster support (we scaled to multiple nodes)
  * Missing features (TTL, eviction policies)
- We ended up throwing it away and implementing Redis

What I learned:
1. NIH (Not Invented Here) syndrome is real and costly
2. Standard tools exist for a reason
3. My desire to build something 'cool' biased my technical judgment
4. I should have evaluated build vs buy more objectively

RESULT:
- Wasted 3 weeks of team time
- Delayed the actual feature by 1 month
- I wrote up the experience as a case study for our team
- Now I always start with 'what existing tool solves this?' before
  considering custom solutions

This taught me the value of boring technology choices. Sometimes
the best engineering is knowing when NOT to engineer."
```

---

## "Tell me about a project that didn't go as planned"

### Framework

```
1. Initial plan and expectations
2. What changed / went wrong
3. How you adapted
4. Final outcome
5. Lessons learned
```

### Пример ответа

```
"I led a project to rebuild our search feature that took 6 months
instead of the planned 2 months.

SITUATION: Our search was slow and inaccurate. I proposed Elasticsearch
as a solution and estimated 2 months based on similar migrations I'd
read about. The business was excited and committed to a launch date.

TASK: Replace MySQL full-text search with Elasticsearch.

ACTION:
Problems that emerged:
1. Data sync between MySQL and ES was complex (eventual consistency
   issues I hadn't anticipated)
2. Our product team kept adding requirements ('can we also add
   filters? faceted search? autocomplete?')
3. Elasticsearch query performance required expertise we didn't have

How I adapted:
- Week 3: Recognized timeline was unrealistic, informed stakeholders
  immediately rather than hoping to catch up
- Proposed phased approach: basic search first, advanced features later
- Brought in a consultant for 2 weeks to accelerate ES expertise
- Started weekly demos to product to manage scope creep
- Documented every change request with timeline impact

RESULT:
- Phase 1 (basic search): 3 months
- Phase 2 (advanced features): 3 more months
- End result exceeded expectations, but timeline was 3x estimate

What I learned:
1. Estimate based on YOUR team's experience, not industry benchmarks
2. Include 'unknown unknowns' buffer for new technology
3. Scope creep kills timelines - need formal change management
4. Communicate early when things go off track

I now explicitly break down estimates into 'known work', 'expected
unknowns', and 'buffer', making estimation assumptions visible."
```

---

## "Describe a time you had to deliver bad news"

### Пример ответа

```
"I had to inform my manager and stakeholders that a promised
feature would miss its deadline by 3 weeks.

SITUATION: We committed to a 'real-time notifications' feature
for a major client. Two weeks before deadline, I realized we'd
significantly underestimated the websocket infrastructure work.

TASK: Communicate the delay while maintaining trust and finding
a path forward.

ACTION:
- First, I validated my assessment with a senior engineer to
  make sure I wasn't missing something
- I prepared a document with:
  * Root cause analysis (what we missed in estimation)
  * Options: ship incomplete, extend timeline, reduce scope
  * My recommendation with reasoning
- I told my manager first (no surprises for them)
- In the stakeholder meeting, I:
  * Led with the news directly: 'We won't make the deadline'
  * Took full responsibility: 'I underestimated complexity'
  * Presented the options and recommendation
  * Asked for decision rather than forgiveness

RESULT:
- Stakeholders were disappointed but appreciated transparency
- We chose 'reduced scope' - core notifications in time, advanced
  features in a follow-up release
- Client was okay with this compromise
- I changed our estimation process to include infrastructure
  assessment for new technology

The key: bad news doesn't get better with time. Delivering it
early with options is always better than surprising people later."
```

---

## "How do you handle setbacks?"

### Framework

```
1. Acknowledge the setback (don't minimize)
2. Process emotions constructively
3. Analyze what happened objectively
4. Create action plan
5. Execute and follow through
6. Reflect and integrate learning
```

### Пример ответа

```
"When my promotion was delayed despite my expectation, I went
through a deliberate process to handle it constructively.

SITUATION: I had worked toward Staff Engineer for 18 months and
was told I was 'close.' The committee decided I needed another
cycle with more cross-team impact evidence.

TASK: Process this setback without becoming bitter or disengaged.

ACTION:
Step 1 - Acknowledge: I let myself be disappointed for a day.
Pretending I didn't care would be dishonest.

Step 2 - Understand: I scheduled a detailed feedback session
with my manager. I asked for specific gaps and examples, not
just vague feedback.

Step 3 - Plan: I identified 3 specific projects that would
demonstrate cross-team impact. I asked my manager to review
my plan and confirm it addressed the gaps.

Step 4 - Execute: Over the next 6 months, I led a cross-team
documentation initiative, mentored an engineer in another team,
and drove a shared library adoption.

Step 5 - Reflect: I realized the feedback was accurate. I had
been focused on depth (my team's work) not breadth (cross-team
impact).

RESULT:
I was promoted the next cycle. More importantly, I became a
better engineer. The cross-team work expanded my perspective
and network significantly.

The setback became a gift - it pushed me to grow in ways I
wouldn't have otherwise."
```

---

## "Tell me about something you'd do differently"

### Пример ответа

```
"Looking back, I would have advocated more strongly against a
microservices migration early in my career.

SITUATION: At a previous startup, we decided to migrate from
a monolith to microservices at ~15 engineers and moderate traffic.
I had concerns but didn't voice them strongly enough.

What I should have done differently:
1. SPEAK UP MORE FORCEFULLY
   - I mentioned concerns in passing but didn't push back
   - I should have written a structured argument against
   - I deferred to 'more experienced' people too easily

2. PROPOSE ALTERNATIVES
   - I criticized without offering alternatives
   - I could have proposed modular monolith as a middle ground

3. GATHER DATA
   - I had intuitions but no data
   - I should have researched similar-sized companies

What happened:
- Migration took 18 months (estimated 6)
- We spent more time on infrastructure than product
- Hiring became harder (needed distributed systems expertise)
- Eventually acquired; new parent company moved us back to monolith

RESULT:
I learned that staying quiet about concerns is its own failure.
Now I have a personal rule: if I disagree with a major decision,
I write up my concerns formally and share them. At minimum, it's
documented. Often, it surfaces important considerations.

Disagreeing diplomatically is a skill I've worked hard to develop."
```

---

## См. также

- [Incident Management](../09-devops-sre/06-incident-management.md) — управление инцидентами и постмортемы
- [Delivery & Execution](./08-delivery.md) — доставка под давлением и управление ожиданиями

---

## Red Flags

```
❌ Mistakes to avoid:

1. "I can't think of a failure"
   - Everyone fails. This suggests lack of self-awareness
   - Or you're not taking enough risks

2. Blaming others
   - "The team didn't deliver" ← Where was your ownership?
   - "PM gave bad requirements" ← Did you clarify?

3. Trivial failures
   - "Once I had a typo in production" ← Too minor
   - Pick something with real stakes

4. Failures without learning
   - "It didn't work out" ← So what did you learn?
   - Must show growth

5. Failures from too long ago
   - Recent failures (last 2-3 years) are more relevant
   - Shows current self-awareness

6. Not taking enough responsibility
   - "It was a team failure" ← What was YOUR part?
   - Focus on what YOU could have done differently
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "What would you do differently now?"
- "How did your team/manager react?"
- "What processes did you change as a result?"
- "Have you seen similar failures since? How did you prevent them?"
- "What's the biggest lesson you've learned from failure?"
- "How do you handle failure in your team members?"
```

---

## Growth Mindset Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Growth Mindset                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Fixed Mindset                   Growth Mindset                     │
│  ────────────                    ─────────────                      │
│  "I failed"          →          "I learned"                         │
│  "I'm not good at X" →          "I'm not good at X yet"             │
│  "This is too hard"  →          "This will take effort"             │
│  "Feedback is attack" →          "Feedback is data"                 │
│  "Others' success    →          "Others' success is                 │
│   threatens me"                   inspiring/educational"            │
│                                                                     │
│  Key practices:                                                     │
│  - Seek feedback actively                                           │
│  - Celebrate effort, not just results                               │
│  - View challenges as opportunities                                 │
│  - Learn from others' successes                                     │
│  - Embrace "productive struggle"                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Postmortem Culture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Blameless Postmortem                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Structure:                                                         │
│  1. Timeline of events                                              │
│  2. Impact (users, revenue, reputation)                             │
│  3. Root cause analysis (5 Whys)                                    │
│  4. What went well (yes, find positives!)                           │
│  5. What went wrong                                                 │
│  6. Action items with owners and deadlines                          │
│                                                                     │
│  Principles:                                                        │
│  - Focus on systems, not people                                     │
│  - "How did the system allow this?" not "Who did this?"             │
│  - Assume competent people trying their best                        │
│  - Document for learning, not blame                                 │
│  - Follow through on action items                                   │
│                                                                     │
│  Example question framing:                                          │
│  ❌ "Who deployed the broken code?"                                 │
│  ✅ "What made it possible for broken code to reach production?"    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

[← Назад к списку тем](README.md)
