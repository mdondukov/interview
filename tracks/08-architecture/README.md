# Трек: Architecture & Design Patterns

Фокус: архитектурные принципы, паттерны проектирования, DDD, и практики построения масштабируемых систем. Каждый модуль содержит теорию, примеры кода и типичные вопросы на интервью.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте на интервью

## Темы и прогресс

### Фундамент
- [ ] [00-architecture-fundamentals](00-architecture-fundamentals.md) — модульность, coupling/cohesion, SOLID

### Clean Architecture & DDD
- [ ] [01-clean-architecture](01-clean-architecture.md) — слои, boundaries, dependency rule
- [ ] [02-ddd-tactical](02-ddd-tactical.md) — entities, value objects, aggregates, repositories
- [ ] [03-ddd-strategic](03-ddd-strategic.md) — bounded contexts, context mapping, ACL

### Распределённые системы
- [ ] [04-microservices-patterns](04-microservices-patterns.md) — decomposition, communication, data management
- [ ] [05-event-driven](05-event-driven.md) — event sourcing, CQRS, eventual consistency
- [ ] [06-integration-patterns](06-integration-patterns.md) — saga, outbox, CDC, API composition

### API & Resilience
- [ ] [07-api-design](07-api-design.md) — REST maturity, gRPC, GraphQL, versioning
- [ ] [08-resilience-patterns](08-resilience-patterns.md) — circuit breaker, bulkhead, retry, timeout

### Тестирование
- [ ] [09-testing-strategies](09-testing-strategies.md) — test pyramid, contract tests, E2E, TDD

## Формат вопроса

```markdown
## Тема: [Название]

### Зачем это нужно
[Проблема, которую решает паттерн/концепция]

### Ключевые концепции
[Основные понятия и принципы]

### Пример кода

```go
// Go implementation
```

```python
# Python implementation
```

### Trade-offs
[Преимущества и недостатки]

### На интервью
[Типичные вопросы и как отвечать]
```

## Статистика

- **Модулей:** 10
- **Уровень:** Senior → Staff → Architect
