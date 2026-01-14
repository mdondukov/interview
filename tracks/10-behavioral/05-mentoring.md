# 05. Mentoring

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Mentoring & Coaching Signals                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Investment in others' growth                                     │
│  ✓ Teaching and communication skills                                │
│  ✓ Patience and empathy                                             │
│  ✓ Adapting to different learning styles                            │
│  ✓ Giving actionable feedback                                       │
│  ✓ Force multiplier mindset                                         │
│  ✓ Building psychological safety                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a time you mentored someone"

### STAR Template

```
SITUATION:
- Who you mentored (level, context)
- What were they struggling with
- Why it mattered

TASK:
- What were you trying to help them achieve
- What was the timeframe

ACTION:
- How you assessed their needs
- What approaches you used
- How you adapted your style
- How you measured progress

RESULT:
- Their growth/outcome
- Impact on team/organization
- What you learned as a mentor
```

### Пример ответа

```
"I mentored a junior developer who was struggling with code reviews
and considering leaving the company.

SITUATION: She had joined 6 months prior but her PRs were regularly
blocked for days. She told me she felt 'constantly criticized' and
was losing confidence. The team was busy, and feedback was often
terse: 'fix this', 'wrong approach'.

TASK: Help her improve code quality while rebuilding her confidence.
The team couldn't afford to lose another junior (third in a year).

ACTION:
Assessment:
- Had a 1:1 to understand her perspective
- Reviewed her last 10 PRs to identify patterns
- Found: 80% of feedback was about same 3 issues (naming, error
  handling, test coverage)

Approach:
1. Weekly 30-min pair programming sessions
   - Showed her MY code before I polish it
   - 'This is my first draft - it's messy too'
   - Narrated my thought process while refactoring

2. Created 'Pre-submission Checklist'
   - Specific items for her common issues
   - She'd self-review before submitting

3. Changed how team gave feedback
   - I spoke to the team: 'Explain WHY, not just WHAT'
   - Modeled constructive feedback in my reviews

4. Celebrated progress
   - Acknowledged improvements publicly in standups
   - 'Great error handling in this PR'

RESULT:
- After 3 months, PR approval rate: 30% → 85% first-time approval
- She started mentoring our newest hire
- She's now a mid-level engineer and team culture improved
- I learned that psychological safety is prerequisite for growth"
```

---

## "How do you give feedback?"

### Framework: SBI (Situation-Behavior-Impact)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SBI Feedback Model                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SITUATION: When and where did this happen?                         │
│  BEHAVIOR: What specifically did they do? (Observable)              │
│  IMPACT: What was the effect? (On you, team, project)               │
│                                                                     │
│  Example (corrective):                                              │
│  ────────────────────                                               │
│  "In yesterday's design review [S],                                 │
│   you interrupted Alex twice while he was presenting [B].           │
│   This made him lose his train of thought and the team             │
│   didn't get to hear his full idea [I]."                           │
│                                                                     │
│  Example (positive):                                                │
│  ─────────────────                                                  │
│  "In the incident yesterday [S],                                    │
│   you calmly coordinated the response and kept everyone            │
│   updated every 15 minutes [B].                                    │
│   This reduced confusion and helped us resolve faster [I]."        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I follow a specific approach to giving feedback that I've developed
over years.

My principles:
1. TIMELY: As close to the event as possible
2. PRIVATE for critical feedback, public for praise
3. SPECIFIC: Behaviors, not personality
4. BALANCED: Always some positive context
5. ACTIONABLE: What can they do differently?

Example of feedback I gave recently:

'Hey, I wanted to share some feedback about the architecture
discussion this morning. [Setting context]

You came really well prepared - the diagrams were clear and the
trade-offs section was thorough. That's exactly what we need in
these discussions. [Positive, specific]

I noticed when Sarah asked about the caching approach, you said
'that's not how caching works' and moved on. [Behavior, observable]

Sarah looked frustrated and didn't ask any more questions. I think
we might have missed some valuable input. [Impact]

Next time, maybe try: 'That's a different approach - can you walk
me through your thinking?' This way we understand their perspective
before evaluating it. [Actionable suggestion]

What do you think?' [Invite dialogue]

The key: feedback should be a gift, not a punishment. Frame it
as 'I'm telling you this because I want you to succeed.'"
```

---

## "How do you handle underperformers?"

### Framework

```
1. DIAGNOSE
   - Is it skill or will?
   - External factors? (personal issues, bad tools)
   - Clear expectations set?

2. DIRECT CONVERSATION
   - Share observations (SBI)
   - Listen to their perspective
   - Agree on problem definition

3. CREATE PLAN
   - Specific, measurable goals
   - Timeline
   - Support needed
   - Check-in cadence

4. FOLLOW THROUGH
   - Regular check-ins
   - Document progress
   - Recognize improvement

5. ESCALATE IF NEEDED
   - PIP if no improvement
   - Involve HR/manager
   - Sometimes not right fit
```

### Пример ответа

```
"I worked with a mid-level engineer who was consistently missing
deadlines and delivering buggy code.

SITUATION: In 6 months, he'd missed 4 of 6 sprint commitments.
Other team members were frustrated, and morale was suffering.
My manager wanted to let him go.

TASK: Determine if we could turn this around and, if so, how.

ACTION:
Diagnosis:
- 1:1 conversation to understand his perspective
- He felt overwhelmed by the codebase complexity
- He was afraid to ask questions ('didn't want to look stupid')
- He was spending 50% of time debugging unfamiliar code

My approach:
1. Paired him with senior engineer for first week of each task
2. Created 'no stupid questions' Slack channel for team
3. Reduced his task scope: fewer items, more focused
4. Weekly 30-min 1:1s tracking specific metrics
5. Assigned him ownership of one small service (accountability)

Timeline: 3 months to see meaningful improvement

RESULT:
- Month 1: Still missing deadlines, but asking more questions
- Month 2: First sprint where he met all commitments
- Month 3: Delivered major feature on time with minimal bugs
- He became one of our more reliable engineers

What I learned: 'Underperformance' often has root causes that
aren't obvious. The same person can thrive or fail depending
on context and support."
```

---

## "Describe your approach to code reviews"

### Пример ответа

```
"I see code reviews as a teaching opportunity, not just quality gate.

My approach:

BEFORE REVIEW:
- Understand the context (read PR description, linked ticket)
- Check if it's a draft or ready for detailed review

DURING REVIEW:
1. Start with what's good
   - Acknowledge good patterns, clever solutions
   - 'Nice use of the strategy pattern here'

2. Categorize comments
   - 'nit:' - style/preference, optional
   - 'question:' - I'm curious, not blocking
   - 'blocker:' - must address before merge

3. Explain WHY, not just WHAT
   - Bad: 'Use a map instead of list'
   - Good: 'Map gives O(1) lookup vs O(n) - matters because
     this is called in a loop'

4. Offer suggestions, not mandates
   - 'Consider...' instead of 'You should...'
   - Include code snippets when helpful

5. Ask questions for context
   - 'I see you chose X, was Y considered?'
   - Assumes there might be good reasons I don't know

SPECIAL CASES:
- Junior developers: More explanatory, link to resources
- Senior developers: Trust their judgment, focus on architecture
- Time-sensitive: Prioritize blockers, save nits for follow-up

Example comment:
'This works! One thought: if we move the validation earlier
(before the API call), we can fail fast and save the network
round trip in error cases. Here's what that might look like:
[code snippet]. Thoughts?'

Code review is mentoring at scale - every review is a chance
to teach and learn."
```

---

## "How do you help someone grow their career?"

### Framework

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Career Development Approach                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. UNDERSTAND THEIR GOALS                                          │
│     - Where do they want to be in 2-3 years?                        │
│     - IC track vs management?                                       │
│     - What motivates them?                                          │
│                                                                     │
│  2. ASSESS CURRENT STATE                                            │
│     - Strengths to leverage                                         │
│     - Gaps to address                                               │
│     - Compare to next level criteria                                │
│                                                                     │
│  3. CREATE OPPORTUNITIES                                            │
│     - Stretch assignments                                           │
│     - Visibility projects                                           │
│     - Cross-functional work                                         │
│                                                                     │
│  4. PROVIDE SUPPORT                                                 │
│     - Regular feedback                                              │
│     - Resources (courses, books, conferences)                       │
│     - Air cover for failures                                        │
│                                                                     │
│  5. ADVOCATE                                                        │
│     - Sponsor in promotion discussions                              │
│     - Connect with right people                                     │
│     - Share their achievements                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример ответа

```
"I helped a senior engineer prepare for Staff promotion over 18 months.

SITUATION: Strong IC, excellent code quality, but not demonstrating
Staff-level impact. Promotion committee had rejected her once
citing 'needs more technical leadership.'

TASK: Help her develop and demonstrate Staff-level skills.

ACTION:
1. Gap analysis:
   - Compared her work to Staff-level criteria
   - Identified: strong execution, weak on influence and design docs
   - Discussed: she wanted Staff but wasn't sure what was different

2. Created development plan:
   - Lead one architecture design (not just implement)
   - Write 2 RFCs and get cross-team feedback
   - Mentor a junior developer
   - Present at engineering all-hands

3. Created opportunities:
   - Assigned her lead role on database migration
   - Connected her with Staff engineers in other teams
   - Asked her to represent our team in cross-team technical meetings

4. Supported actively:
   - Reviewed her RFC drafts before she shared widely
   - Gave feedback after her presentations
   - Shared 'this is what Staff thinking looks like' in our discussions

5. Advocated:
   - Shared her achievements in leadership meetings
   - Provided detailed promotion packet
   - Defended her case in committee

RESULT:
- She was promoted to Staff 18 months later
- The database migration she led became a company-wide case study
- She now mentors others pursuing Staff

My lesson: sponsorship (using your influence for them) is as
important as mentorship (advising them)."
```

---

## "How do you build a culture of learning on your team?"

### Пример ответа

```
"I've worked to create a learning culture through several mechanisms.

What I've implemented:

1. WEEKLY LEARNING TIME
   - Friday afternoons: 2 hours 'learning time'
   - Can work on side projects, take courses, read papers
   - Share learnings in Slack #til channel

2. FAILURE SHARES
   - Monthly 'failure of the month' presentation
   - I share MY failures first to set the tone
   - Celebrate what was learned, not the failure itself

3. CODE REVIEW AS LEARNING
   - Encouraged 'learning reviews' - request review to learn,
     not just for approval
   - Junior engineers review senior code too

4. KNOWLEDGE SHARING
   - Rotating 'tech talk' responsibility
   - 15 minutes, informal, any topic
   - Past talks: 'Redis internals', 'How I debug', 'Rust basics'

5. PSYCHOLOGICAL SAFETY
   - 'No stupid questions' channel
   - When someone asks a question I don't know: 'Great question,
     I don't know either, let's find out'
   - Never mock or dismiss questions

Results I've seen:
- Team started actively seeking feedback
- Knowledge silos decreased (less 'only X knows this')
- Onboarding time reduced (new hires ramp faster)
- Retention improved (people cite growth as reason to stay)

Key insight: Learning culture requires investment and modeling
from senior people. You can't just say 'we value learning' -
you have to demonstrate it."
```

---

## Red Flags

```
❌ Mistakes to avoid:

1. Only mentored once
   "I helped someone once in 2018"
   → Shows it's not a regular practice

2. Took credit for their success
   "I made them successful"
   → Should focus on THEIR growth

3. Only positive examples
   "Everyone I mentor succeeds"
   → What about when it didn't work?

4. Didn't adapt approach
   "I teach everyone the same way"
   → Different people need different styles

5. Mentoring = telling
   "I told them what to do"
   → Good mentoring involves asking, listening

6. No measurable outcomes
   "I think they improved"
   → What specifically changed?
```

---

## Mentoring vs Coaching vs Sponsoring

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Mentoring vs Coaching vs Sponsoring               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MENTORING (Advising)                                               │
│  - Share your experience and wisdom                                 │
│  - Give guidance based on your journey                              │
│  - "When I faced X, here's what I did..."                           │
│                                                                     │
│  COACHING (Developing)                                              │
│  - Ask questions to help them find answers                          │
│  - Build their capability to solve own problems                     │
│  - "What options have you considered?"                              │
│                                                                     │
│  SPONSORING (Advocating)                                            │
│  - Use your influence to create opportunities                       │
│  - Advocate for them in rooms they're not in                        │
│  - "You should put X on this project, here's why..."                │
│                                                                     │
│  All three are needed. Adjust based on:                             │
│  - Their level (junior needs more mentoring)                        │
│  - The situation (crisis = direct advice)                           │
│  - What they're asking for                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "What's the most challenging mentoring situation you've had?"
- "How do you mentor someone you don't manage?"
- "How do you balance mentoring with your own work?"
- "Have you ever had to give up on mentoring someone?"
- "How do you handle when your mentee disagrees with your advice?"
- "What have you learned FROM mentees?"
```

---

[← Назад к списку тем](README.md)
