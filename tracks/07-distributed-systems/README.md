# Distributed Systems

Теоретический фундамент распределённых систем для Senior-level интервью.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте на интервью

## Темы и прогресс

### Основы
- [ ] [00-fallacies-fundamentals](00-fallacies-fundamentals.md) — 8 fallacies, latency numbers, failure modes
- [ ] [01-cap-theorem](01-cap-theorem.md) — CAP, PACELC, реальные примеры

### Консистентность
- [ ] [02-consistency-models](02-consistency-models.md) — strong, eventual, causal, linearizability
- [ ] [03-time-ordering](03-time-ordering.md) — Lamport clocks, vector clocks, hybrid clocks

### Консенсус и репликация
- [ ] [04-consensus](04-consensus.md) — Paxos basics, Raft в деталях, leader election
- [ ] [05-replication-theory](05-replication-theory.md) — state machine replication, chain replication

### Масштабирование
- [ ] [06-partitioning](06-partitioning.md) — consistent hashing, range, hash, rebalancing

### Транзакции и отказоустойчивость
- [ ] [07-distributed-transactions](07-distributed-transactions.md) — 2PC, 3PC, Saga, compensation
- [ ] [08-failure-detection](08-failure-detection.md) — heartbeats, phi accrual, gossip protocols

### Координация
- [ ] [09-coordination](09-coordination.md) — ZooKeeper, etcd, distributed locks, service discovery

## Формат вопроса

```markdown
## Тема: [Название]

### Концепция
[Теория и объяснение]

### Визуализация
[Диаграмма или ASCII-схема]

### Реальные примеры
[Как используется в production системах]

### На интервью
- Типичные вопросы
- Ключевые моменты
```

## Статистика

- **Модулей:** 10
- **Вопросов:** ~50
- **Уровень:** Senior → Staff
