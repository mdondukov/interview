# 00. STAR Method

[← Назад к списку тем](README.md)

---

## Структура STAR

```
┌─────────────────────────────────────────────────────────────────────┐
│                        STAR Method                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  S - Situation (Ситуация)                                           │
│      Контекст: где, когда, что за проект/команда                    │
│      ~15-20% времени ответа                                         │
│                                                                     │
│  T - Task (Задача)                                                  │
│      Что нужно было сделать, какая была цель                        │
│      ~10-15% времени ответа                                         │
│                                                                     │
│  A - Action (Действие)                                              │
│      Что конкретно ВЫ сделали (не команда, а вы)                    │
│      ~50-60% времени ответа                                         │
│                                                                     │
│  R - Result (Результат)                                             │
│      Измеримый outcome, чему научились                              │
│      ~15-20% времени ответа                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Подготовка историй

### Матрица историй

```
┌─────────────────┬──────────────┬──────────────┬──────────────────────┐
│ Тема            │ История 1    │ История 2    │ История 3            │
├─────────────────┼──────────────┼──────────────┼──────────────────────┤
│ Leadership      │ Tech lead    │ Cross-team   │ Mentoring junior     │
│                 │ migration    │ initiative   │                      │
├─────────────────┼──────────────┼──────────────┼──────────────────────┤
│ Conflict        │ Disagreement │ Pushback on  │ Difficult            │
│                 │ with PM      │ design       │ stakeholder          │
├─────────────────┼──────────────┼──────────────┼──────────────────────┤
│ Failure         │ Production   │ Missed       │ Bad technical        │
│                 │ incident     │ deadline     │ decision             │
├─────────────────┼──────────────┼──────────────┼──────────────────────┤
│ Technical       │ Architecture │ Build vs buy │ Prioritization       │
│ Decision        │ choice       │              │                      │
├─────────────────┼──────────────┼──────────────┼──────────────────────┤
│ Ambiguity       │ New product  │ Unclear      │ Competing            │
│                 │ area         │ requirements │ priorities           │
└─────────────────┴──────────────┴──────────────┴──────────────────────┘

Подготовьте 8-10 историй, которые можно адаптировать под разные вопросы.
```

---

## Пример плохого ответа

```
Вопрос: "Tell me about a time you had to deal with a difficult coworker"

❌ Плохой ответ:
"Well, there was this guy on my team who was always late to meetings
and didn't do his work properly. I told him to be more responsible
but he didn't listen. Eventually he left the company which solved
the problem."

Проблемы:
- Нет конкретики (когда, какой проект)
- Blame на другого человека
- Нет описания своих действий
- Нет learning
- Пассивное решение (он ушёл)
```

---

## Пример хорошего ответа

```
Вопрос: "Tell me about a time you had to deal with a difficult coworker"

✅ Хороший ответ:

SITUATION:
"When I joined PaymentCo as a senior engineer in 2022, I was assigned
to work on the checkout team with a colleague who had been there for
5 years. He was initially resistant to my suggestions and would often
dismiss my ideas in code reviews without explanation."

TASK:
"I needed to find a way to build a productive working relationship
because we had to collaborate on the new payment gateway project,
which was critical for the company."

ACTION:
"First, I asked him for a 1:1 coffee chat to understand his perspective.
I learned he felt overlooked for promotion and was frustrated that
a new hire was making changes to 'his' codebase.

I acknowledged his expertise and asked if he could mentor me on the
system's history. I also started CC'ing him early on design docs and
explicitly asking for his input before team meetings.

When we had technical disagreements, I proposed we do small experiments
or POCs rather than debating in abstract. For one controversial change,
we agreed to deploy behind a feature flag and compare metrics."

RESULT:
"Within 2 months, our relationship completely transformed. He became
one of my strongest advocates and we successfully delivered the payment
gateway on time. He was promoted to tech lead the following quarter.

I learned that resistance often comes from feeling unheard, and that
investing time in understanding someone's perspective pays off hugely."
```

---

## Техники

### CAR (альтернатива STAR)

```
C - Challenge (вызов)
A - Action (действие)
R - Result (результат)

Более короткий формат, подходит для простых вопросов.
```

### Количественные результаты

```
Вместо:                          Лучше:
─────────                        ──────
"Improved performance"        →  "Reduced latency from 500ms to 50ms"
"Saved time"                  →  "Automated 20 hours/week of manual work"
"Good impact"                 →  "Increased conversion by 15%"
"Team liked it"               →  "NPS from 30 to 70"
```

### Я vs Мы

```
Вместо:                          Лучше:
─────────                        ──────
"We decided to..."            →  "I proposed that we..."
"The team implemented..."     →  "I led the implementation of..."
"We discovered..."            →  "I identified the root cause..."

Интервьюер хочет понять ВАШ вклад.
Но не преуменьшайте команду — покажите как вы с ней работали.
```

---

## Типы вопросов

### Tell me about a time when...

```
- ...you had to make a difficult decision
- ...you dealt with conflict
- ...you failed
- ...you influenced without authority
- ...you had to learn something quickly
```

### What would you do if...

```
Гипотетические вопросы. Можно:
1. Описать как бы подошли
2. Привести похожий реальный пример

"I haven't faced exactly that situation, but let me tell you about
a similar challenge I had..."
```

### Why...

```
- Why do you want to work here?
- Why are you leaving?
- Why this role?

Требуют подготовки, research компании.
```

---

## Чек-лист подготовки

```
□ Подготовить 8-10 историй (матрица выше)
□ Каждую историю записать по STAR
□ Практиковать вслух (засекать время: 2-3 минуты)
□ Подготовить числа и метрики
□ Research компании и её ценностей
□ Подготовить вопросы для интервьюера
□ Практика с другом или запись себя
```

---

## См. также

- [Leadership](./01-leadership.md) — вопросы о лидерстве и влиянии
- [Failure & Learning](./03-failure-learning.md) — как рассказывать об ошибках и уроках

---

## Red Flags

```
❌ Избегать:
- Blame на других
- Нет конкретики ("usually", "sometimes")
- Только "we", никогда "I"
- Нет результата или learning
- Негативные комментарии о предыдущих работодателях
- Слишком длинные ответы (>4 минут)
- Слишком короткие ответы (<1 минуты)
```

---

[← Назад к списку тем](README.md)
