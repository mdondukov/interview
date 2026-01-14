# 01. CAP Theorem

[← Назад к списку тем](README.md)

---

## CAP Theorem

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CAP Theorem                                    │
│                   (Eric Brewer, 2000)                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  В распределённой системе невозможно одновременно гарантировать     │
│  все три свойства. При network partition нужно выбрать:             │
│                                                                     │
│                      Consistency                                    │
│                          /\                                         │
│                         /  \                                        │
│                        /    \                                       │
│                       / CP   \                                      │
│                      /        \                                     │
│                     /    CA    \                                    │
│                    /____________\                                   │
│        Availability              Partition                          │
│                      \    AP    /  Tolerance                        │
│                       \        /                                    │
│                        \      /                                     │
│                         \    /                                      │
│                          \  /                                       │
│                           \/                                        │
│                                                                     │
│  CA = невозможно в реальных распределённых системах                 │
│       (partition всегда может случиться)                            │
│                                                                     │
│  CP = Consistency + Partition Tolerance                             │
│       При partition: отказываем в запросах                          │
│                                                                     │
│  AP = Availability + Partition Tolerance                            │
│       При partition: возвращаем stale данные                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Определения

### Consistency

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Consistency (Linearizability)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Все узлы видят одинаковые данные в одно и то же время.             │
│  После успешной записи все чтения возвращают новое значение.        │
│                                                                     │
│  Time →                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Write(x=1)                                                 │    │
│  │      │                                                      │    │
│  │      ▼                                                      │    │
│  │  ────●────────────────────────────────────────────────────  │    │
│  │      │         Read(x)=1    Read(x)=1    Read(x)=1          │    │
│  │      │              │            │            │             │    │
│  │      └──────────────●────────────●────────────●──────────── │    │
│  │                                                             │    │
│  │  После Write все Read возвращают 1                          │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Availability

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Availability                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Каждый запрос к работающему узлу получает ответ                    │
│  (не обязательно с самыми свежими данными).                         │
│                                                                     │
│  ┌─────────┐         ┌─────────┐                                    │
│  │ Client  │ ─────▶  │  Node   │                                    │
│  └─────────┘         └────┬────┘                                    │
│                           │                                         │
│                           ▼                                         │
│                     Response (always)                               │
│                                                                     │
│  100% availability = всегда отвечаем                                │
│  99.9% = 8.76 часов downtime в год                                  │
│  99.99% = 52.6 минут downtime в год                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Partition Tolerance

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Partition Tolerance                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Система продолжает работать при потере связи между узлами.         │
│                                                                     │
│       ┌─────────┐              ┌─────────┐                          │
│       │ Node A  │              │ Node B  │                          │
│       └────┬────┘              └────┬────┘                          │
│            │                        │                               │
│            │     ████████████       │                               │
│            │  ← Network Partition → │                               │
│            │     ████████████       │                               │
│            │                        │                               │
│       Clients A                Clients B                            │
│                                                                     │
│  Node A и Node B не могут общаться.                                 │
│  Каждый обслуживает своих клиентов.                                 │
│  Что делать?                                                        │
│                                                                     │
│  CP: Остановить операции (ждать восстановления)                     │
│  AP: Продолжать работать (возможна inconsistency)                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CP vs AP Systems

### CP Systems

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CP Systems                                     │
│              Consistency + Partition Tolerance                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  При partition: лучше вернуть ошибку, чем stale данные              │
│                                                                     │
│  Примеры:                                                           │
│  ─────────                                                          │
│  - ZooKeeper, etcd, Consul                                          │
│  - HBase, MongoDB (default)                                         │
│  - Spanner, CockroachDB                                             │
│  - Traditional RDBMS                                                │
│                                                                     │
│  Use cases:                                                         │
│  ───────────                                                        │
│  - Финансовые транзакции                                            │
│  - Inventory management                                             │
│  - Distributed locks                                                │
│  - Configuration management                                         │
│                                                                     │
│  Trade-off:                                                         │
│  ───────────                                                        │
│  - Ниже availability при partitions                                 │
│  - Выше latency (нужен consensus)                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### AP Systems

```
┌─────────────────────────────────────────────────────────────────────┐
│                       AP Systems                                     │
│              Availability + Partition Tolerance                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  При partition: лучше вернуть stale данные, чем ошибку              │
│                                                                     │
│  Примеры:                                                           │
│  ─────────                                                          │
│  - Cassandra, DynamoDB                                              │
│  - CouchDB, Riak                                                    │
│  - DNS                                                              │
│  - CDN caches                                                       │
│                                                                     │
│  Use cases:                                                         │
│  ───────────                                                        │
│  - Social media feeds                                               │
│  - Product catalogs                                                 │
│  - Shopping carts                                                   │
│  - User preferences                                                 │
│                                                                     │
│  Trade-off:                                                         │
│  ───────────                                                        │
│  - Eventual consistency                                             │
│  - Conflict resolution needed                                       │
│  - Read-your-writes не гарантирован                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## PACELC

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PACELC                                       │
│                    (Daniel Abadi, 2012)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Расширение CAP: что происходит когда partition НЕТ?                │
│                                                                     │
│  if (Partition) {                                                   │
│      choose between Availability and Consistency                    │
│  } else {                                                           │
│      choose between Latency and Consistency                         │
│  }                                                                  │
│                                                                     │
│  P + A / E + L  (PA/EL)                                             │
│  ───────────────────────                                            │
│  При partition: Availability                                        │
│  Без partition: Latency (быстрые ответы)                            │
│  Примеры: Cassandra, DynamoDB                                       │
│                                                                     │
│  P + A / E + C  (PA/EC)                                             │
│  ───────────────────────                                            │
│  При partition: Availability                                        │
│  Без partition: Consistency                                         │
│  Примеры: редко используется                                        │
│                                                                     │
│  P + C / E + L  (PC/EL)                                             │
│  ───────────────────────                                            │
│  При partition: Consistency                                         │
│  Без partition: Latency                                             │
│  Примеры: PNUTS (Yahoo)                                             │
│                                                                     │
│  P + C / E + C  (PC/EC)                                             │
│  ───────────────────────                                            │
│  При partition: Consistency                                         │
│  Без partition: Consistency                                         │
│  Примеры: Traditional RDBMS, Spanner                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Реальные примеры

### DynamoDB (AP / PA/EL)

```go
// Eventually consistent read (default) - fast
item, _ := client.GetItem(&dynamodb.GetItemInput{
    TableName: aws.String("users"),
    Key: map[string]*dynamodb.AttributeValue{
        "id": {S: aws.String("user-123")},
    },
    // ConsistentRead: default false
})

// Strongly consistent read - slower
item, _ := client.GetItem(&dynamodb.GetItemInput{
    TableName: aws.String("users"),
    Key: map[string]*dynamodb.AttributeValue{
        "id": {S: aws.String("user-123")},
    },
    ConsistentRead: aws.Bool(true),  // 2x latency, 2x cost
})
```

### MongoDB (CP by default)

```go
// Write concern controls consistency
collection.InsertOne(ctx, doc, options.InsertOne().
    SetWriteConcern(writeconcern.New(
        writeconcern.WMajority(),        // Wait for majority
        writeconcern.WTimeout(5000),     // 5 second timeout
    )),
)

// Read concern
collection.Find(ctx, filter, options.Find().
    SetReadConcern(readconcern.Majority()),  // Read from majority
)
```

### Cassandra (AP / PA/EL)

```sql
-- Tunable consistency per query
-- Consistency level: ONE, QUORUM, ALL

-- Fast write (AP behavior)
INSERT INTO users (id, name) VALUES ('123', 'John')
USING CONSISTENCY ONE;

-- Strong write (CP-like behavior)
INSERT INTO users (id, name) VALUES ('123', 'John')
USING CONSISTENCY QUORUM;

-- Fast read
SELECT * FROM users WHERE id = '123'
USING CONSISTENCY ONE;

-- Strong read (read-your-writes)
SELECT * FROM users WHERE id = '123'
USING CONSISTENCY QUORUM;
```

---

## Выбор CP vs AP

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Decision Framework                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Choose CP when:                                                    │
│  ───────────────                                                    │
│  - Data correctness critical (money, inventory)                     │
│  - Business logic requires consistency                              │
│  - Can tolerate temporary unavailability                            │
│  - Users can retry failed operations                                │
│                                                                     │
│  Choose AP when:                                                    │
│  ───────────────                                                    │
│  - Availability critical (user-facing apps)                         │
│  - Stale data acceptable (social feeds, caches)                     │
│  - Can handle conflicts later                                       │
│  - Global distribution required                                     │
│                                                                     │
│  Hybrid approach:                                                   │
│  ────────────────                                                   │
│  - Different consistency for different operations                   │
│  - Read: eventually consistent (fast)                               │
│  - Write: strongly consistent (safe)                                │
│  - Critical paths: CP                                               │
│  - Non-critical: AP                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Consistency Models](./02-consistency-models.md) — модели консистентности и их спектр от сильной до слабой
- [Replication](../06-databases/05-replication.md) — репликация данных в базах данных

---

## На интервью

### Типичные вопросы

1. **Что такое CAP theorem?**
   - Consistency, Availability, Partition Tolerance
   - Можно выбрать только 2 из 3 при partition
   - P всегда нужен → выбор между C и A

2. **CP vs AP — примеры систем?**
   - CP: ZooKeeper, PostgreSQL, MongoDB
   - AP: Cassandra, DynamoDB, DNS

3. **Что такое PACELC?**
   - Расширение CAP
   - При partition: A vs C
   - Без partition: L vs C

4. **Когда выбрать AP?**
   - Availability важнее
   - Stale данные OK
   - Global distribution

5. **Может ли система быть CA?**
   - Теоретически да (single node)
   - В распределённой системе — нет
   - Partition всегда может случиться

---

[← Назад к списку тем](README.md)
