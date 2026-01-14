# 02. Conflict Resolution

[← Назад к списку тем](README.md)

---

## Что проверяют

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Conflict Resolution Signals                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Emotional intelligence                                           │
│  ✓ Active listening                                                 │
│  ✓ Seeking to understand before being understood                    │
│  ✓ Finding win-win solutions                                        │
│  ✓ Professional maturity                                            │
│  ✓ Escalation judgment                                              │
│  ✓ Maintaining relationships                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## "Tell me about a time you had a disagreement with a coworker"

### STAR Template

```
SITUATION:
- Кто был вовлечён (роль, не имя)
- Что было причиной разногласия
- Почему это было важно

TASK:
- Что нужно было решить
- Какие были stakes

ACTION (ключевая часть):
- Как вы поняли их perspective
- Как выразили свою позицию
- Как искали общее решение
- Как пришли к согласию

RESULT:
- Outcome для проекта/команды
- Состояние отношений после
- Что узнали
```

### Пример ответа

```
"At my previous company, I had a significant disagreement with a
senior backend engineer about API design for a new feature.

SITUATION: We were building a new notification system. I proposed
a RESTful approach with separate endpoints for each notification
type. He strongly advocated for a single GraphQL endpoint. The
decision would affect the next 2 years of development.

TASK: We needed to reach a decision quickly as the sprint was
starting, but neither of us wanted to just 'give in' because
this was a foundational choice.

ACTION:
- First, I scheduled a 1:1 coffee chat outside of meetings to
  understand his perspective without an audience
- I learned his main concerns: frontend flexibility and reducing
  over-fetching - valid points I hadn't fully considered
- I shared my concerns: team's GraphQL inexperience, operational
  complexity, debugging difficulties
- Instead of debating, I proposed we list all requirements and
  score both solutions against them
- We discovered that 80% of use cases were simple and REST would
  work, but 20% (dashboards) really benefited from GraphQL
- We proposed a hybrid: REST for simple endpoints, GraphQL for
  complex queries

RESULT:
This hybrid approach was adopted and worked well. More importantly,
we became strong collaborators - he now seeks my input on API
decisions, and I've learned a lot about GraphQL from him.

The key lesson: disagreements often mean both people see part of
the picture. The goal isn't to win, but to find the best solution."
```

---

## "Tell me about a time you received difficult feedback"

### Key Points

```
Показать:
- Openness to feedback (not defensive)
- How you processed it
- Concrete actions taken
- Growth that resulted
- Gratitude for the feedback
```

### Пример ответа

```
"During my first performance review as a senior engineer, my manager
told me that while my technical work was excellent, I was seen as
'hard to collaborate with' by some team members.

SITUATION: This was surprising and honestly painful to hear. I
thought I was being helpful by quickly pointing out issues in code
reviews and design discussions.

TASK: I needed to understand specifically what I was doing wrong
and change my approach without compromising on technical quality.

ACTION:
- First, I asked my manager for specific examples - turns out my
  code review comments came across as dismissive ('just use X
  instead') without explaining why
- I asked two trusted colleagues for honest feedback. One said
  'You're always right, but you make people feel stupid'
- I started a practice: before submitting any code review, I'd
  re-read my comments asking 'would I want to receive this?'
- I changed from 'This is wrong' to 'Have you considered X?
  Here's why it might help...'
- I started explicitly acknowledging good parts of PRs before
  suggesting changes

RESULT:
Six months later, my manager noted significant improvement in team
feedback. I was nominated for a 'team player' award. Most importantly,
I realized that being right is worthless if you can't bring people
along with you.

I now actively seek feedback, especially negative, because that's
where growth happens."
```

---

## "Tell me about a time you pushed back on something"

### Framework

```
1. Почему вы были против (technical/process/values)
2. Как выразили несогласие (respectfully, with data)
3. К кому обратились (peer, manager, skip-level)
4. Результат (changed, compromised, accepted decision)
5. Как поддержали final decision
```

### Пример ответа

```
"Our VP of Engineering wanted to adopt a 'mandatory Friday
deployment freeze' policy for all teams.

SITUATION: While I understood the intent (reduce weekend incidents),
I believed this would hurt our team's velocity and didn't address
the root cause.

TASK: Express disagreement constructively while respecting the
VP's authority and concerns.

ACTION:
- First, I made sure I understood the full context. I asked about
  the incidents that prompted this - learned there were 3 major
  Friday deploys that caused weekend pages
- I analyzed our team's data: we did Friday deploys regularly with
  zero incidents due to our robust CI/CD and canary process
- I prepared a proposal: instead of blanket freeze, require Friday
  deploys to have additional review and monitoring commitments
- I asked my manager if I could present this in the next engineering
  leads meeting
- I presented with data, acknowledged the problem was real, and
  proposed a more targeted solution
- I made clear I'd fully support whatever decision was made

RESULT:
The VP appreciated the data-driven approach. We piloted my
alternative with our team for a month, tracked results, and it
became the standard policy - 'enhanced review for Friday deploys'
rather than a freeze.

Key learning: pushback is much more effective when you come with
alternatives, not just objections."
```

---

## "Tell me about a difficult conversation you had to have"

### Types of Difficult Conversations

```
- Performance feedback to a peer
- Saying no to a stakeholder
- Admitting a mistake
- Raising concerns about a project
- Disagreeing with leadership
- Addressing behavior issues
```

### Пример ответа

```
"I had to tell a product manager that a feature he'd been promising
stakeholders for months wasn't technically feasible within the
timeline.

SITUATION: He had committed to a 'real-time analytics dashboard'
in 6 weeks without consulting engineering. Our data pipeline
couldn't support real-time - it was batch-processed daily.

TASK: Deliver this news without destroying the relationship and
find a path forward.

ACTION:
- I requested a private meeting rather than surprising him in
  a group setting
- I started by acknowledging his position: 'I understand you've
  made commitments and this puts you in a difficult spot'
- I explained the technical constraints clearly, avoiding jargon
- Instead of just saying 'no', I offered alternatives:
  * 6-hour refresh (achievable in 6 weeks)
  * True real-time in 3 months with additional resources
  * Hybrid: key metrics real-time, full dashboard daily
- I offered to join the stakeholder meeting to explain technical
  constraints and present options
- I took ownership: 'We should have been looped in earlier, and
  I'll make sure that happens going forward'

RESULT:
He was initially frustrated but appreciated that I came with
solutions. Stakeholders chose the hybrid option. We now have a
'technical feasibility check' as a required step before commitments.

The relationship actually improved because I showed I was trying
to help him succeed, not just block him."
```

---

## "Describe a time when you had to work with someone difficult"

### Key Points

```
Важно:
- Не критиковать человека (behaviors, not character)
- Показать empathy
- Фокус на ваших actions
- Результат должен быть constructive

Избегать:
- "They were a jerk"
- "They didn't listen to anyone"
- Passive waiting for them to leave
```

### Пример ответа

```
"I worked with a senior engineer who would frequently dismiss
ideas from junior team members in design reviews, sometimes
quite harshly.

SITUATION: This was affecting team morale. Junior engineers
stopped contributing in meetings. One told me they were
considering leaving.

TASK: Address the behavior without escalating it formally or
damaging my relationship with this engineer.

ACTION:
- I first tried to understand: I had lunch with him and learned
  he'd had negative experiences with 'inexperienced engineers
  making costly mistakes' at his previous company
- In the next design review, when he dismissed an idea, I
  intervened: 'Let's hear this out - what's the thinking behind
  this approach?'
- I spoke to him privately: 'I noticed you're quick to critique
  ideas. Your experience is valuable, but I think we're missing
  good input because people are hesitant to speak up'
- I suggested a format change: written proposals before meetings,
  so he could review calmly rather than react in the moment
- I also started explicitly asking junior engineers for input
  in meetings to create space for them

RESULT:
The dynamic improved significantly over a few months. He later
told me he hadn't realized how he was coming across and
appreciated me telling him directly. Two junior engineers who
were considering leaving decided to stay.

Key insight: difficult people often have reasons for their
behavior. Curiosity before judgment."
```

---

## Common Mistakes

```
❌ Mistakes to avoid:

1. Avoiding the conflict
   "I just let it go" ← shows you can't handle hard situations

2. Escalating too quickly
   "I went to HR" ← shows you can't solve problems yourself

3. Being too aggressive
   "I told them they were wrong" ← shows poor collaboration

4. Taking all the blame
   "It was probably my fault" ← shows lack of confidence

5. Claiming no conflicts
   "I've never had conflicts" ← either lying or not self-aware

6. Naming names / company details
   Keep it professional, use roles not identities
```

---

## Follow-up вопросы

```
Будьте готовы к:

- "What would you do if they hadn't changed?"
- "Have you ever had to escalate? When is that appropriate?"
- "How did the relationship evolve after?"
- "What did you learn about yourself?"
- "Would you handle it differently now?"
- "How do you prevent conflicts proactively?"
```

---

## Conflict Resolution Principles

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Effective Conflict Resolution                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. SEEK TO UNDERSTAND FIRST                                        │
│     - Listen before responding                                      │
│     - Ask clarifying questions                                      │
│     - Acknowledge their perspective                                 │
│                                                                     │
│  2. SEPARATE PEOPLE FROM PROBLEMS                                   │
│     - Focus on issues, not personalities                            │
│     - Assume positive intent                                        │
│     - Attack the problem, not the person                            │
│                                                                     │
│  3. FOCUS ON INTERESTS, NOT POSITIONS                               │
│     - "Why do you want that?" not "What do you want?"               │
│     - Find underlying needs                                         │
│     - Look for win-win                                              │
│                                                                     │
│  4. GENERATE OPTIONS                                                │
│     - Brainstorm alternatives                                       │
│     - Don't evaluate during generation                              │
│     - Expand the pie before dividing                                │
│                                                                     │
│  5. USE OBJECTIVE CRITERIA                                          │
│     - Data over opinions                                            │
│     - Industry standards                                            │
│     - Experiments and POCs                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

[← Назад к списку тем](README.md)
