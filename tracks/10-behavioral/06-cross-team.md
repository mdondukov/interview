# 06. Cross-Team Collaboration

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                Cross-Team Collaboration Signals                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Working across organizational boundaries                         │
│  ✓ Managing dependencies effectively                                │
│  ✓ Stakeholder communication                                        │
│  ✓ Building relationships outside your team                         │
│  ✓ Navigating competing priorities                                  │
│  ✓ Influencing without authority                                    │
│  ✓ Driving alignment                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a time you worked with other teams"

### STAR Template

```
SITUATION:
- What teams were involved
- What was the project/goal
- Why cross-team work was needed

TASK:
- Your role in the collaboration
- What needed to be coordinated
- Challenges anticipated

ACTION:
- How you built relationships
- How you communicated
- How you handled conflicts
- How you tracked progress

RESULT:
- Project outcome
- Relationship outcomes
- Process improvements
```

### Пример ответа

```
"I led a project to implement a new authentication system that
required coordination across 4 teams: platform, mobile, security,
and infrastructure.

SITUATION: We were migrating from session-based to JWT authentication.
Each team owned a piece: platform (APIs), mobile (apps), security
(token policies), infrastructure (secret management). No single
team owned the whole flow.

TASK: Coordinate the migration without formal authority over any
of the other teams.

ACTION:
1. Built relationships first
   - Had 1:1 coffee chats with leads from each team
   - Understood their priorities and constraints
   - Found 'what's in it for them' in this migration

2. Created shared understanding
   - Wrote RFC describing the full picture
   - Gathered input from all teams before finalizing
   - Visual diagrams showing dependencies

3. Established coordination mechanisms
   - Weekly cross-team sync (30 min, focused agenda)
   - Shared Slack channel for async questions
   - Shared timeline in project management tool

4. Handled conflicts
   - Security wanted 15-min token expiry, mobile wanted 24 hours
   - Facilitated discussion: understood constraints (security audit
     vs battery drain from frequent refreshes)
   - Found middle ground: 1-hour tokens with refresh tokens

5. Made dependencies visible
   - Created dependency graph
   - 'Our team is blocked on X from Team B'
   - Escalated blockers proactively

RESULT:
- Migration completed on schedule
- Zero authentication incidents during rollout
- Created template for future cross-team projects
- Built strong relationships with other teams

Key learning: Cross-team work is 70% relationship building,
30% technical coordination."
```

---

## "Tell me about managing dependencies"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Dependency Management                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. IDENTIFY EARLY                                                  │
│     - Map all dependencies at project start                         │
│     - Include people/approval dependencies, not just technical      │
│                                                                     │
│  2. COMMUNICATE CLEARLY                                             │
│     - What you need                                                 │
│     - When you need it                                              │
│     - Why it matters (impact of delay)                              │
│                                                                     │
│  3. BUILD BUFFER                                                    │
│     - Don't schedule on critical path without buffer                │
│     - Have fallback plans                                           │
│                                                                     │
│  4. TRACK ACTIVELY                                                  │
│     - Regular check-ins on dependency status                        │
│     - Escalate blockers early                                       │
│                                                                     │
│  5. REDUCE WHERE POSSIBLE                                           │
│     - Can you do it yourself?                                       │
│     - Can you decouple?                                             │
│     - Can you mock/stub temporarily?                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"Our project had a critical dependency on another team that was
deprioritizing our needs.

SITUATION: We needed a new API endpoint from the Platform team
to complete our checkout redesign. They had committed to it, but
it kept getting pushed as they dealt with incidents and their
own priorities.

TASK: Get the API delivered without damaging the relationship
or escalating prematurely.

ACTION:
1. Understood their situation
   - Had coffee with their tech lead
   - Learned they were drowning in operational issues
   - Our request was 'important but not urgent' for them

2. Made it easier for them
   - Wrote the API spec myself (usually their job)
   - Offered to write the implementation they'd review
   - 'How can we minimize your team's effort?'

3. Aligned incentives
   - Found a way the API benefited them too (reduced
     support tickets they were handling)
   - Got product to include their metrics in success criteria

4. Had backup plan
   - Designed our feature to work without the new API (degraded)
   - Could ship with workaround if needed

5. Made progress visible
   - Weekly status to both teams' leadership
   - 'Blocked on X, expected date Y, impact if missed Z'

RESULT:
- They delivered 1 week late (vs 3+ weeks if I'd just waited)
- Relationship strengthened - they appreciated I didn't just complain
- My implementation spec saved them time, they were grateful

Lesson: When depending on others, do everything you can to make
their job easier. Dependencies are relationships, not transactions."
```

---

## "Tell me about working with a difficult stakeholder"

### Пример ответа

```
"I had to work with a VP who was known for changing requirements
mid-project and not respecting engineering constraints.

SITUATION: He wanted to launch a feature for a major client. He
kept adding 'small' requirements that weren't small, and pushed
back on any timeline changes. Engineering-product relationship
was strained.

TASK: Deliver the feature while establishing better working
patterns.

ACTION:
1. Sought to understand his pressure
   - Found out: he had committed to the client personally
   - His bonus was tied to this deal
   - 'Difficult' behavior was stress, not malice

2. Changed communication approach
   - Instead of 'that's 2 more weeks', showed trade-off table
   - 'Adding X: we can do A (drop Y), B (delay 2 weeks), C (reduce quality)'
   - Let HIM make the priority call

3. Created visibility
   - Daily 1-paragraph status updates
   - 'Today: completed X. Blockers: Y. Need from you: Z'
   - He could see progress and felt in control

4. Set boundaries respectfully
   - 'We can add this feature, but I need to be honest: it puts
     the launch date at risk. Here's why...'
   - Provided data, not opinions

5. Found wins for him
   - Identified 2 features we could ship early
   - He could show progress to the client
   - Built trust that we were trying to help him succeed

RESULT:
- Delivered on time (original scope + 2 additions, - 1 cut feature)
- He became an advocate for engineering needs
- He started consulting engineering earlier in planning
- I learned: 'difficult people' often have difficult pressures"
```

---

## "How do you build alignment across teams?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Building Cross-Team Alignment                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. START WITH WHY                                                  │
│     - Shared understanding of the problem                           │
│     - Agree on success criteria before solutions                    │
│                                                                     │
│  2. UNDERSTAND EACH TEAM'S CONTEXT                                  │
│     - What are their priorities?                                    │
│     - What constraints do they have?                                │
│     - What do they care about?                                      │
│                                                                     │
│  3. FIND SHARED GOALS                                               │
│     - How does this help ALL teams?                                 │
│     - Frame in terms of company goals, not team goals               │
│                                                                     │
│  4. CREATE SHARED ARTIFACTS                                         │
│     - Joint design docs                                             │
│     - Shared roadmaps                                               │
│     - Common metrics/dashboards                                     │
│                                                                     │
│  5. ESTABLISH COMMUNICATION CADENCE                                 │
│     - Regular syncs                                                 │
│     - Clear escalation paths                                        │
│     - Async-first where possible                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I drove alignment between engineering, product, and design on our
API versioning strategy - a topic with historically strong disagreements.

SITUATION: Product wanted frequent breaking changes for new features.
Engineering wanted stability. Design wanted flexibility. We'd been
arguing about each API change individually for months.

TASK: Create a shared policy everyone could support.

ACTION:
1. Framed the problem collaboratively
   - 'None of us want customer pain. Let's find a way to move
     fast AND not break things.'
   - Not 'engineering vs product' but 'us vs the problem'

2. Gathered each perspective formally
   - Asked each group to write their top 3 requirements
   - Product: 'Ship features quickly'
   - Engineering: 'Don't break existing integrations'
   - Design: 'Don't be constrained by legacy decisions'

3. Found the actual conflict
   - Realized: only ~20% of changes were truly breaking
   - Most arguments were about edge cases
   - We were solving the wrong problem

4. Proposed framework together
   - Workshop session with representatives from each team
   - Created decision tree for when breaking changes are okay
   - Defined deprecation timeline (6 months warning)

5. Documented and socialized
   - Wrote RFC together - all three leads as co-authors
   - Presented at all-hands to create shared understanding
   - Created Slack channel for versioning questions

RESULT:
- API arguments dropped 80%
- Time to ship new features didn't increase
- No customer complaints about breaking changes in 12 months
- Framework became template for other cross-functional policies

Key insight: Alignment isn't about compromise - it's about finding
solutions that meet everyone's actual needs."
```

---

## "Tell me about navigating competing priorities"

### Пример ответа

```
"Three teams simultaneously needed the same platform resource -
our search infrastructure team's time.

SITUATION: Growth team wanted search performance improvements.
Monetization wanted ad placement in search. Discovery wanted
new search features. All were Q3 priorities. Search team could
only handle one major project.

TASK: As search team lead, help determine priority without
creating resentment.

ACTION:
1. Made trade-offs explicit
   - Created one-pager showing impact of each project
   - Revenue estimates, user impact, technical complexity
   - Not my opinion - data from each team

2. Facilitated prioritization session
   - Got all three leads in a room with product leadership
   - Ground rules: 'We're all on the same side'
   - Went through company goals to ground the discussion

3. Surfaced hidden dependencies
   - Turns out ad placement needed performance work first
   - Sequential dependency nobody had noticed
   - Changed the conversation

4. Proposed creative solutions
   - What if we do performance Q3, ads Q4?
   - Growth can help with some performance work (they had capacity)
   - Discovery features could be 70% self-service

5. Let leadership decide
   - I presented options and recommendations
   - Didn't advocate for my preference
   - Leadership picked based on business context I didn't have

RESULT:
- Clear priority order that everyone understood reasoning for
- No resentment because process was transparent
- Discovered dependency saved us from failure
- Teams collaborated on the solution rather than competing

Lesson: In priority conflicts, be a facilitator, not a combatant.
Make trade-offs visible, let leaders decide."
```

---

## "How do you handle cross-team conflict?"

### Пример ответа

```
"Two teams had an ongoing conflict about who owned a shared
service that was causing both teams operational pain.

SITUATION: A legacy service sat between payments and orders teams.
Both used it, neither wanted to maintain it. Bug fixes got delayed
because each team said it was the other's responsibility.

TASK: Resolve ownership and stop the finger-pointing.

ACTION:
1. Diagnosed the root cause
   - Not laziness - both teams had capacity constraints
   - No clear ownership model for shared services
   - Organizational gap, not people gap

2. Facilitated joint discussion
   - 'Let's agree on the facts first'
   - Listed all the pain points both teams experienced
   - Found common ground: both wanted the service better

3. Explored options together
   - Team A owns: but they're at capacity
   - Team B owns: but code is far from their expertise
   - Create new team: overkill for one service
   - Deprecate and split functionality: technically possible

4. Proposed ownership model
   - Split the service by domain
   - Payment-related logic → payments team
   - Order-related logic → orders team
   - 6-month project to decouple, shared responsibility during

5. Got leadership buy-in
   - Presented plan to both directors
   - Got commitment for resources
   - Created shared metrics for transition

RESULT:
- Service split completed in 8 months
- Operational incidents dropped 70%
- Created template for future shared service ownership discussions
- Teams now collaborate better after working through this together

Key learning: Cross-team conflict is usually organizational, not
personal. Fix the system, not the people."
```

---

## См. также

- [Conflict Resolution](./02-conflict-resolution.md) — разрешение конфликтов между командами
- [Leadership](./01-leadership.md) — влияние без формальной власти

---

## Red Flags

```
❌ Mistakes to avoid:

1. Only worked within your team
   "I mostly work on my team's projects"
   → Senior roles require cross-team effectiveness

2. Blamed other teams
   "They didn't deliver on time"
   → What did YOU do to help?

3. Escalated immediately
   "I told my manager to handle it"
   → Should try to resolve at your level first

4. No relationship building
   "I sent them a Jira ticket"
   → Cross-team work requires relationships

5. Focused only on your needs
   "I needed X from them"
   → What did you offer in return?

6. Avoided difficult conversations
   "We just worked around it"
   → Should address conflicts directly
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "How do you build trust with teams you don't interact with daily?"
- "What do you do when another team's priorities conflict with yours?"
- "How do you escalate effectively without damaging relationships?"
- "Tell me about a cross-team project that failed. What would you do differently?"
- "How do you influence teams you have no authority over?"
- "How do you maintain alignment as organizations grow?"
```

---

## Cross-Team Communication Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                Cross-Team Communication                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ASYNC FIRST                                                        │
│  - Write things down (RFCs, docs)                                   │
│  - Shared channels, not DMs                                         │
│  - Meetings for discussion, not information                         │
│                                                                     │
│  OVER-COMMUNICATE                                                   │
│  - Repeat important information                                     │
│  - Assume people miss messages                                      │
│  - Different formats for different people                           │
│                                                                     │
│  MAKE IT VISUAL                                                     │
│  - Diagrams > text for architecture                                 │
│  - Timelines > lists for schedules                                  │
│  - Dashboards > reports for status                                  │
│                                                                     │
│  BE EXPLICIT                                                        │
│  - "I need X by Y for Z reason"                                     │
│  - "My understanding is... please correct if wrong"                 │
│  - "Decision needed: options are A, B, C"                           │
│                                                                     │
│  FOLLOW UP                                                          │
│  - Recap decisions in writing                                       │
│  - Check understanding                                              │
│  - Verify commitments                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

[← Назад к списку тем](README.md)
