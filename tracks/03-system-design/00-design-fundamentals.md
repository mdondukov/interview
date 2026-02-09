# 00 — System Design Fundamentals

Технические основы system design: CAP теорема, модели консистентности, выбор баз данных, паттерны масштабирования, consistent hashing.

**Навигация:** [Трек System Design](./README.md) · Следующая тема: [01-url-shortener](./01-url-shortener.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**CAP теорема** — фундаментальный принцип распределённых систем, сформулированный Эриком Брюером в 2000 году. Теорема утверждает, что любая распределённая система может одновременно обеспечить только два из трёх свойств: консистентность (Consistency), доступность (Availability) и устойчивость к разделению сети (Partition Tolerance). Поскольку сетевые разделения в реальных системах неизбежны, на практике приходится выбирать между консистентностью и доступностью. Это не означает, что система полностью теряет одно из свойств — речь о гарантиях в момент сбоя сети.

**Consistency (консистентность)** — свойство системы, при котором все узлы видят одни и те же данные в один момент времени. После успешной записи любое последующее чтение вернёт это новое значение, независимо от того, к какому узлу обратился клиент. Это критично для финансовых операций: если пользователь перевёл деньги, он должен сразу видеть обновлённый баланс, а не старое значение с другой реплики.

**Availability (доступность)** — гарантия того, что каждый запрос к работающему узлу получит ответ (успех или ошибка), без гарантии, что ответ содержит самые свежие данные. Высокая доступность означает, что система продолжает обслуживать запросы даже при частичных сбоях. Для пользовательских приложений это часто важнее абсолютной консистентности: лучше показать слегка устаревшую ленту новостей, чем выдать ошибку.

**Partition Tolerance (устойчивость к разделению)** — способность системы продолжать работу, когда сетевое соединение между узлами временно разорвано. В реальных распределённых системах сетевые разделения неизбежны: кабели рвутся, маршрутизаторы падают, датацентры теряют связь. Поэтому любая распределённая система должна быть устойчива к partition — это не опциональное свойство, а необходимость.

**Linearizability (линеаризуемость)** — самая строгая модель консистентности, при которой все операции выглядят так, будто выполняются на одной машине в определённом порядке, соответствующем реальному времени. Если операция A завершилась до начала операции B, то все наблюдатели увидят эффект A перед эффектом B. Это даёт максимальные гарантии, но требует синхронизации между узлами, что увеличивает задержки и снижает доступность.

**Eventual consistency (согласованность в конечном счёте)** — модель, при которой система гарантирует, что если новые записи прекратятся, то со временем все реплики придут к одинаковому состоянию. В промежутке разные узлы могут возвращать разные значения. Это позволяет строить высокодоступные системы с минимальными задержками, но требует от приложения умения работать с потенциально устаревшими данными.

**Read-Your-Writes** — промежуточная модель консистентности, гарантирующая, что клиент всегда видит результаты своих собственных записей. Другие клиенты могут временно видеть устаревшие данные, но автор изменений сразу видит их эффект. Это хороший баланс для веб-приложений: пользователь обновил профиль и сразу видит изменения, а другие пользователи увидят их с небольшой задержкой.

**Causal consistency (причинная консистентность)** — модель, сохраняющая причинно-следственные связи между операциями. Если операция B зависит от операции A (например, ответ на комментарий), все узлы увидят A перед B. Операции без причинной связи могут наблюдаться в разном порядке. Это важно для социальных сетей: нельзя показать ответ на сообщение раньше самого сообщения.

**Replication (репликация)** — копирование данных на несколько узлов для повышения доступности и отказоустойчивости. Если один узел выходит из строя, данные остаются доступны на других. Репликация бывает синхронной (запись подтверждается после копирования на все реплики) и асинхронной (подтверждение сразу, копирование в фоне). Синхронная даёт консистентность, асинхронная — скорость.

**Quorum (кворум)** — минимальное количество узлов, которые должны подтвердить операцию для её успеха. Типичная формула: для N реплик требуется W узлов для записи и R для чтения, где W + R > N. Это гарантирует, что хотя бы один узел участвовал и в записи, и в чтении, обеспечивая консистентность. Например, при 3 репликах можно требовать 2 для записи и 2 для чтения — даже при падении одного узла система работает корректно.

---

## Вопросы и разборы

### 1. Как работает CAP теорема?

**Зачем спрашивают.** CAP теорема — фундаментальное ограничение распределённых систем. Интервьюер проверяет понимание trade-offs между Consistency, Availability и Partition Tolerance.

**Короткий ответ.** CAP теорема гласит: в распределённой системе при network partition нельзя гарантировать одновременно Consistency и Availability. Выбираешь либо CP (Consistency + Partition), либо AP (Availability + Partition). CA не возможен в сетевых системах.

**Детальный разбор.**

**Определения:**

```
Consistency (C):
- Все узлы видят одни и те же данные
- После write, все read возвращают последнее значение
- Пример: банковский счёт, ACID базы

Availability (A):
- Каждый запрос получает ответ (success или failure)
- Система всегда доступна и отзывчива
- Пример: кэш, eventual consistency базы

Partition Tolerance (P):
- Система продолжает работать при network partition
- Узлы не могут общаться между собой
- В реальных распределённых системах P обязателен!
```

**Визуализация CAP теоремы:**

```
                        Consistency (C)
                             ▲
                            / \
                           /   \
                          /     \
                         /       \
                        /    CP   \
                       /           \
                      /             \
                     /               \
                    ├─────────────────┤
                   /                   \
                  /         Must        \
                 /        Choose         \
                /                         \
               /                           \
              /                             \
             /              AP               \
            /                                 \
           /                                   \
          /                                     \
         /_______________________________________\
        Availability (A)      Partition Tolerance (P)

NB: CA (Consistency + Availability без Partition Tolerance) = impossible в networked systems
    → Если нет network partition, CA достижимо (монолитная система)
    → Но в любой распределённой системе P нужен
```

**Три архетипа:**

```
1. CP (Consistency + Partition):
   ├─ При partition: sacrifice Availability
   ├─ Примеры: MongoDB (strict mode), HBase, Consul
   ├─ Поведение:
   │  - Write path: успех, если большинство узлов доступно
   │  - Read path: всегда читаешь актуальное значение
   │  - При partition: один side может быть offline
   └─ Когда использовать: financial systems, inventory

2. AP (Availability + Partition):
   ├─ При partition: sacrifice Consistency
   ├─ Примеры: Cassandra, DynamoDB, Riak
   ├─ Поведение:
   │  - Write path: успех на любом узле (даже если partition)
   │  - Read path: может возвращать stale data
   │  - Eventual consistency: со временем все синхронизируются
   └─ Когда использовать: social feeds, recommendations

3. CA (Consistency + Availability):
   ├─ Не существует при network partition
   ├─ Возможен только в:
   │  - Монолитных системах (нет сети)
   │  - Полностью синхронных системах (нельзя обойти задержку)
   └─ На практике: переходит в CP или AP при partition
```

**Пример: распределённый банк с partition:**

```
┌─────────────────────────────────────────────────────────┐
│ Сценарий: Account balance = $1000, network partition    │
├─────────────────────────────────────────────────────────┤

Normal state (no partition):
┌─────────────────┐                 ┌─────────────────┐
│  Data Center 1  │◄─ replication ─►│  Data Center 2  │
│  balance: $1000 │                 │  balance: $1000 │
└─────────────────┘                 └─────────────────┘

Network partition occurs:
┌─────────────────┐   PARTITION     ┌─────────────────┐
│  Data Center 1  │─────────────────│  Data Center 2  │
│  balance: $1000 │                 │  balance: $1000 │
└─────────────────┘                 └─────────────────┘

CP Choice (Consistency + Partition):
Client tries to withdraw $500 from DC1:
- DC1: "I need to confirm with DC2..."
- DC2: "I can't reach DC2, rejecting write"
- Result: ERROR "Unable to process"
- ✓ Consistency: balance unchanged, not double-spent
- ✗ Availability: client can't withdraw

AP Choice (Availability + Partition):
Client tries to withdraw $500 from DC1:
- DC1: "OK, processing withdrawal"
- DC1: balance = $500
- (Later tries to withdraw $400 from DC2, which succeeds)
- DC2: balance = $600
- Result: ✓ Availability: both withdrawals succeed
- ✗ Consistency: total = $1100 (money duplicated!)
- Eventually sync: need conflict resolution
```

**PACELC Theorem (улучшенная версия CAP):**

```
CAP теорема предполагает, что partition всегда происходит.
На практике:

PACELC:
├─ If there is a Partition:
│  ├─ Choose Availability or Consistency
│  └─ (Same as CAP)
│
└─ Else (no partition):
   └─ Choose Latency or Consistency
      (Even without partition, есть trade-off)

Пример:
Google Bigtable:
- If partition → CP (consistent, reject writes)
- If no partition → Usually LC (low latency, eventual consistency)
  (Can write to any replica immediately, sync later)
```

**Пример на интервью:**

```
Интервьюер: "Design a system that must be consistent.
Does CAP theorem limit us?"

Мой ответ:

"CAP theorem says we cannot guarantee both consistency and
availability during network partition. If consistency is critical:

1. Architecture choice: CP (Consistency + Partition Tolerance)
   - Sacrifice Availability during partition
   - Use quorum-based writes: require majority vote before commit
   - Example: Write succeeds only if (replicas written / total_replicas) > 0.5

2. Typical implementation:
   - Leader-based replication (1 primary)
   - Write: primary commits, then replicas
   - If primary fails: elect new leader
   - During election: writes are blocked (sacrifice A)
   - After new leader elected: writes resume with C guaranteed

3. Real world trade-off:
   - Partition probability is low (99.9% network uptime)
   - 99.9% of time: system is both consistent and available
   - 0.1% of time: inconsistent network partition → choose consistency
   - Users won't mind occasional unavailability if consistency is guaranteed

4. Mitigation strategies:
   - Minimize partition probability: multi-path networking
   - Minimize partition duration: fast failover
   - Multiple data centers: increases partition risk but improves availability
   - Hybrid approach: different guarantees for different data
     (Transactions = CP, recommendations = AP)"
```

**Типичные ошибки.**

- Думать, что P можно выбрать или нет — в распределённых системах P обязателен, выбираешь только между C и A.
- Путать CAP с других трёхбуквенных теоремами (BASE, ACID) — это разные концепции.
- Говорить "наша система CA" о распределённой системе — невозможно.
- Игнорировать практическое влияние: partition вероятность низка (0.1%), но последствия серьёзны.

**На интервью.**

- Нарисуй треугольник CAP и объясни каждую сторону.
- Приведи примеры реальных систем (MongoDB = CP по умолчанию, Cassandra = AP).
- Упомяни, что выбор влияет на архитектуру: если CP → quorum writes, если AP → eventual consistency.
- Уточняющий вопрос: "How do we handle conflicts in AP systems?" → версионирование, CRDT, last-write-wins.

---

### 2. В чём разница между consistency моделями?

**Зачем спрашивают.** Consistency модели определяют гарантии, которые система даёт о порядке операций. Интервьюер проверяет понимание trade-offs между сильной и слабой консистентностью.

**Короткий ответ.** Consistency модели варьируются от Strong (все видят одно и то же) до Eventual (со временем синхронизируются). Промежуточные: Read-Your-Writes, Causal, Timeline. Выбор зависит от use case и acceptable latency.

**Детальный разбор.**

**Иерархия consistency моделей:**

```
┌─────────────────────────────────────────────────────────┐
│           Consistency Models Spectrum                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Strong ────────────────────────────────── Weak          │
│                                                         │
│ ├─ Linearizability / Strong Consistency                 │
│ ├─ Sequential Consistency                               │
│ ├─ Causal Consistency                                   │
│ ├─ Timeline (Session) Consistency                       │
│ ├─ Read-Your-Writes (RYW)                               │
│ ├─ Monotonic Read Consistency                           │
│ └─ Eventual Consistency                                 │
│                                                         │
│ Tradeoff:                                               │
│ Stronger → Lower latency ✗, Higher cost ✗               │
│ Weaker  → Higher latency ✓, Lower cost ✓                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**1. Linearizability (Strong Consistency):**

```
Определение:
- Все операции выполняются как на single machine
- Есть global order для всех операций
- Каждое чтение видит последний write

Гарантия:
  Write(x=1) at t1 → Read(x) at t2 → always returns 1

Пример:
┌─────────────────────────────────────┐
│ Thread 1: write(x=1) ─────────┐     │
│                               │     │
│ Thread 2:                    read(x) → returns 1
│                               │     │
│ Thread 3: write(x=2) ─────────┘     │
└─────────────────────────────────────┘

Реальность:
- Требует синхронной репликации
- Write latency = wait for all replicas
- Недостижимо в географически распределённых системах
- Примеры: традиционные ACID БД, ZooKeeper

Использовать когда:
- Финансовые транзакции
- Inventory management
- Когда логика требует видеть актуальное состояние
```

**2. Sequential Consistency:**

```
Определение:
- Результаты выглядят как если бы на одной машине
- Но порядок операций разных потоков может отличаться

Гарантия:
  Каждый процесс видит операции в одинаковом порядке
  Но это не обязательно реальный хронологический порядок

Пример:
  Thread 1: write(x=1), write(y=1)
  Thread 2: read(y) → 1
           read(x) → ??? (может быть 0 или 1)

  Sequential: если read(y) видит 1, то thread 1 и thread 2
  видят одинаковый порядок операций

  Но это порядок может быть не реальный во времени

Примеры: некоторые memory models в процессорах

Использовать когда: редко, обычно нужна либо strong, либо weaker
```

**3. Causal Consistency:**

```
Определение:
- Если operation A причинно зависит от B, все процессы
  видят B перед A
- Непричинно связанные операции могут видеться в разном порядке

Гарантия:
  write(x=1) ─ causally depends ─► write(y=2)
  Все процессы: видят write(x) перед write(y)

Пример (Social Network):
  Alice posts message: write("msg: hello")
  Bob likes message: write("liked: true") — causally depends on Alice's write
  Charlie reads:
  - видит Alice's message
  - видит Bob's like
  - видит в правильном порядке (message → like)

Практика:
  - Requires versioning / vector clocks
  - More efficient than linearizability
  - Still ensures important causality

Примеры: Dynamo (with vector clocks), some NoSQL DBs

Использовать когда:
- Социальные сети (comments on posts)
- Discussion threads
- Когда нужна причинность, но не полный порядок
```

**4. Read-Your-Writes (RYW / Session Consistency):**

```
Определение:
- Клиент видит свои собственные writes немедленно
- Другие могут видеть stale data

Гарантия:
  write(x=1) by client
  read(x) by same client → always returns 1

  read(x) by other client → может вернуть 0 (stale)

Пример:
  User updates profile (write)
  Same user views profile (read) → видит свои изменения
  Other user views profile (read) → может видеть старый профиль
  Eventually → все видят новый профиль

Реализация:
  - Client-side tracking: remember which servers has writes
  - Server reads only from servers that have client's writes
  - Cookie-based: store version vector

Примеры: Cassandra, DynamoDB (with write-after-write)

Использовать когда:
- Большинство веб приложений
- Email (пользователь видит отправленное письмо)
- Социальные сети (видишь свой пост, но друзья видят с задержкой)
```

**5. Monotonic Read Consistency:**

```
Определение:
- Если read вернул версию V, следующие reads вернут версию ≥ V
- Никогда не видишь более старую версию, чем уже видел

Гарантия:
  read(x) → returns version 3
  read(x) → returns version 4 or higher (never 2!)

Пример:
  User views notification count = 5
  Refreshes page → notification count = 3 ??? (should be ≥ 5)
  This violates monotonic read consistency

Реализация:
  - Client remembers latest version read
  - Routes next read to replica with ≥ that version

Примеры: Многие распределённые кэши

Использовать когда:
- Когда видишь изменения, не хочешь видеть откат
- Примеры: score в рейтинге, quantity в inventory
```

**6. Eventual Consistency:**

```
Определение:
- Если нет новых writes, со временем все видят одно и то же
- На данный момент: могут быть различия
- Guarantee: eventually они синхронизируются

Гарантия:
  write(x=1) at t1
  read(x) at t2 (t2 > t1) → может вернуть 0 или 1
  read(x) at t3 (t3 → ∞) → вернёт 1

Пример:
  Alice posts photo
  Bob sees photo after 0.5 seconds
  Charlie sees photo after 2 seconds
  - система выбрала eventual consistency
  - все с течением времени видят фото

Реализация:
  - Async replication
  - Gossip protocol / change log
  - Reconciliation / conflict resolution

Примеры: Cassandra, DynamoDB, S3

Использовать когда:
- High availability нужнее consistency
- Социальные сети (лайки, комментарии)
- Реклама, аналитика
- Рекомендации

Проблемы:
- Conflicts: если 2 users пишут разные значения
  Solution: last-write-wins, CRDT, версионирование
- Stale reads: клиент может видеть старые данные
  Solution: RYW, Causal, Monotonic for critical paths
```

**Сравнительная таблица:**

```
┌─────────────────────┬──────────┬──────────┬──────────────┬─────────┐
│ Model               │ Latency  │ Cost     │ Real-World   │ Typical │
│                     │          │          │ Availability │ Use     │
├─────────────────────┼──────────┼──────────┼──────────────┼─────────┤
│ Linearizability     │ High ✗   │ High ✗   │ 99%          │ Finance │
│ Sequential          │ Medium   │ Medium   │ ~99%         │ Rare    │
│ Causal              │ Medium   │ Medium   │ 99.9%        │ Forums  │
│ RYW / Session       │ Low ✓    │ Low ✓    │ 99.99% ✓     │ Web     │
│ Monotonic Read      │ Low ✓    │ Low ✓    │ 99.99% ✓     │ Cache   │
│ Eventual            │ Very Low │ Very Low │ 99.999% ✓✓   │ Social  │
└─────────────────────┴──────────┴──────────┴──────────────┴─────────┘
```

**Пример на интервью:**

```
Интервьюер: "Design a social network. What consistency model do you use?"

Мой ответ:

"Depends on the component:

1. User Authentication (Strong Consistency / Linearizability):
   - If user registers and logs in, must see immediate effect
   - Implement: synchronous writes to primary DB + cache
   - Latency acceptable: 100ms is OK for auth

2. Feed (Eventual Consistency):
   - When Alice posts, Bob sees within seconds
   - Async replication: Alice writes → Message Queue → Fan-out to followers
   - Followers' feeds updated eventually
   - Acceptable latency: 1-5 seconds

3. User Profile Read (Read-Your-Writes):
   - User updates profile (write)
   - Same user views profile (read) → must see changes immediately
   - Other users might see old profile temporarily
   - Route user's reads to replica that has processed user's writes

4. Like Counter (Eventual Consistency + Monotonic Reads):
   - When user likes post, counter increments eventually
   - User never sees counter go down (monotonic reads)
   - Implement: version vector, gossip protocol

5. Comment Threads (Causal Consistency):
   - Comments have dependencies (replies to comments)
   - All users should see causal order
   - Use vector clocks to track causality
   - If you see reply, you see parent comment

Резюме trade-off:
- 80% системы: Eventual consistency (scalable)
- 15%: RYW + Monotonic reads (good UX)
- 5%: Strong consistency (critical operations)
- Результат: высокодоступная система с хорошим UX"
```

**Типичные ошибки.**

- Путать "consistency" в ACID с consistency моделями в распределённых системах — разные понятия.
- Выбирать Strong Consistency везде — система будет медленная и unavailable.
- Выбирать Eventual везде — пользователи будут видеть смешные баги (например, лайки исчезают).
- Не ясно понимать разницу между RYW и Monotonic — близко, но важные детали.

**На интервью.**

- Нарисуй спектр consistency моделей.
- Поясни trade-off: сильнее консистентность → выше latency / ниже availability.
- Приведи примеры из реальной жизни (банк = strong, соцсеть = eventual).
- Обсуди per-component выбор: разные части системы могут использовать разные модели.
- Уточняющий вопрос: "How do you handle conflicts in eventual consistency?" → versioning, CRDT, last-write-wins.

---

### 3. Как выбирать между SQL и NoSQL?

**Зачем спрашивают.** SQL и NoSQL имеют разные силы и слабости. Интервьюер проверяет понимание trade-offs и умение выбирать правильный инструмент.

**Короткий ответ.** Используй SQL (PostgreSQL, MySQL) для structured data, ACID transactions, complex queries. Используй NoSQL (MongoDB, Cassandra, DynamoDB) для unstructured data, horizontal scaling, high throughput. Выбор зависит от access patterns, consistency requirements, scale.

**Детальный разбор.**

**SQL Database Strengths & Weaknesses:**

```
SQL (PostgreSQL, MySQL, Oracle):

✓ ACID Transactions:
  - Atomicity: all or nothing
  - Consistency: constraints guaranteed
  - Isolation: no dirty reads
  - Durability: written to disk

✓ Complex Queries:
  - JOINs: multiple tables
  - GROUP BY, aggregate functions
  - Subqueries, window functions
  - Flexible WHERE clauses

✓ Schema Enforcement:
  - Columns must match type
  - Foreign key constraints
  - Data integrity at DB level

✓ Mature & Stable:
  - Decades of optimization
  - Battle-tested in production
  - Good tooling, libraries

✗ Horizontal Scaling:
  - Sharding is complex
  - Transactions across shards are hard
  - Usually vertical scaling

✗ Flexible Schema:
  - Schema changes require migrations
  - Adding column to 1B row table = slow
  - Downtimes possible

✗ High Write Throughput:
  - ACID guarantees have cost
  - Replication + consistency checking = latency
  - Typically 10K-100K QPS per server

✗ Unstructured Data:
  - Not designed for JSON, blobs, etc.
  - Could store as TEXT, but lose benefits
```

**NoSQL Database Types & Strengths:**

```
1. Key-Value (Redis, Memcached, DynamoDB):

✓ Ultra-fast reads/writes
  - Data in memory (Redis)
  - Hash table access = O(1)
  - Throughput: 1M+ QPS per server

✓ Flexible values
  - String, hash, list, set, sorted set (Redis)
  - JSON (DynamoDB)
  - Anything serializable

✓ Horizontal scaling
  - Sharding by key = easy
  - No cross-key transactions (usually)

✗ Limited queries
  - Can't JOIN
  - Can't aggregate across keys
  - Need to denormalize

✗ Limited consistency
  - Often eventual consistency
  - Single-key atomicity but multi-key is hard


2. Document (MongoDB, Couchbase, Firestore):

✓ Flexible schema
  - Each document can have different structure
  - JSON-like format
  - Easy to evolve

✓ Rich queries
  - Can query nested fields
  - Aggregation pipeline
  - More flexible than key-value

✓ Indexes
  - Support multiple indexes
  - Can still query efficiently

✗ Still not SQL-like
  - No JOIN (until MongoDB 3.6 $lookup)
  - Aggregations more complex than SQL
  - Denormalization often needed

✗ Transactions still limited
  - Multi-document ACID in some (MongoDB 4.0+)
  - But slower than SQL


3. Wide-Column (Cassandra, HBase, Bigtable):

✓ Extreme scale & throughput
  - Designed for 1000s of nodes
  - Write throughput: 1M+ QPS cluster
  - Store petabytes

✓ High availability
  - No single point of failure
  - Automatic replication
  - Handles failures gracefully

✓ Time-series data
  - Optimized for time-series
  - Good for logs, metrics, events
  - TTL built-in

✗ Limited queries
  - Must query by partition key
  - No arbitrary WHERE
  - Must denormalize heavily

✗ Complex to use
  - Consistent hashing, rebalancing
  - Need to understand internals
  - Operational complexity


4. Graph (Neo4j, ArangoDB):

✓ Relationships
  - Traversals are fast
  - Relationships are first-class
  - Social networks, recommendations

✓ Path queries
  - "Find friends of friends"
  - Complex traversals in single query

✗ Overkill for most use cases
  - Need actual graph to benefit
  - If data is hierarchical (tree) → use SQL

✗ Scaling challenges
  - Usually not horizontally scalable
  - High memory requirements
```

**Comparison Matrix:**

```
┌──────────────────────┬────────┬────────┬───────┬──────┬────────┐
│ Aspect               │ SQL    │ KV     │ Doc   │ Wide │ Graph  │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Transactions         │ Excellent
│                      │ ACID   │ Limited│ Good  │ Poor │ Good   │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Query Flexibility    │ Very High
│                      │ ✓✓✓    │ ✗      │ ✓✓    │ ✗✗   │ ✓✓✓    │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Horizontal Scale     │ Hard   │ Easy   │ Easy  │ ✓✓✓  │ Hard   │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Schema Flexibility   │ Low    │ Very H │ High  │ Med  │ High   │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Read Throughput      │ High   │ ✓✓✓    │ High  │ ✓✓   │ Med    │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Write Throughput     │ Med    │ ✓✓✓    │ High  │ ✓✓✓  │ Med    │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Data Consistency     │ Strong │ Varies │ Varies│ Weak │ Strong │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Operational Ease     │ Easy   │ Easy   │ Med   │ Hard │ Easy   │
├──────────────────────┼────────┼────────┼───────┼──────┼────────┤
│ Memory Efficiency    │ High   │ Low    │ Med   │ High │ Low    │
└──────────────────────┴────────┴────────┴───────┴──────┴────────┘
```

**Decision Tree:**

```
Начало: Нужно хранить данные?
│
├─ Is it highly relational?
│  ├─ YES: Users ←→ Orders ←→ Products
│  │  └─► SQL (PostgreSQL)
│  │
│  └─ NO: Mostly independent entities
│     └─ Continue...

├─ Do you need ACID transactions?
│  ├─ YES: Money, inventory, reservations
│  │  └─► SQL (PostgreSQL with sharding if needed)
│  │
│  └─ NO: Recommends, feeds, events
│     └─ Continue...

├─ What's your scale?
│  ├─ < 1M QPS, < 10TB: SQL is fine
│  ├─ 1M+ QPS read-heavy: Cache (Redis) + SQL
│  ├─ 1M+ QPS write-heavy: NoSQL
│  └─ 10M+ QPS, petabytes: Wide-column (Cassandra)

├─ Do you have many JOINs?
│  ├─ YES: SQL (PostgreSQL)
│  ├─ NO: Denormalization OK: Consider NoSQL
│  └─ Lots of relationships: Graph DB

└─ Schema?
   ├─ Fixed & stable: SQL
   ├─ Evolving, flexible: Document (MongoDB)
   └─ Simple key-value: Redis
```

**Пример на интервью:**

```
Интервьюер: "Design e-commerce system. SQL or NoSQL?"

Мой ответ:

"Depends on the component. I'd use both:

1. User Accounts & Orders (SQL - PostgreSQL):
   - ACID transactions critical
   - Orders have status, payments, shipments
   - Complex queries: "show my orders with expensive items"
   - Foreign keys: users → orders → items → inventory
   - Use SQL

2. Product Catalog (SQL or Document):
   - Products have different attributes
   - Electronics: brand, specs, color
   - Clothing: size, material, style
   - Option A: SQL with flexible JSONB column (PostgreSQL)
   - Option B: MongoDB for more flexibility
   - I'd choose PostgreSQL JSONB for consistency

3. Session & Cache (Redis):
   - User sessions, shopping cart
   - Need fast reads/writes
   - No durability critical (can lose sessions)
   - Use Redis key-value store
   - TTL for auto-expiration

4. Recommendations & Analytics (NoSQL - Cassandra or analytical DB):
   - "Users who bought X also bought Y"
   - "Product views per day" (time-series)
   - Write-heavy: logs, events
   - Use Cassandra for time-series or
   - Use ClickHouse for analytics queries
   - Eventual consistency OK

5. Search (Elasticsearch):
   - Full-text search on products
   - Not a primary DB, derivative of catalog
   - Use Elasticsearch + indexing from PostgreSQL

Architecture:
- PostgreSQL: Users, Orders, Payments, Inventory (ACID)
- Redis: Sessions, Cart, Cache
- Cassandra or ClickHouse: Logs, analytics, recommendations
- Elasticsearch: Product search

This gives us:
- Consistency where it matters (orders, inventory)
- Speed where needed (cache, sessions)
- Scale for analytics (Cassandra)"
```

**Hybrid Approach (OLTP + OLAP):**

```
┌─────────────────────────────────────────────────────────┐
│ Online Transaction Processing (OLTP)                    │
│ - Primary database: PostgreSQL                          │
│ - Optimized for: frequent, small updates                │
│ - Workload: read-modify-write                           │
│ - Consistency: strong                                   │
│ - Example: credit card payment                          │
└──────────────┬──────────────────────────────────────────┘
               │
        [ETL Pipeline]
               │
┌──────────────▼──────────────────────────────────────────┐
│ Online Analytical Processing (OLAP)                     │
│ - Analytics database: ClickHouse, Snowflake             │
│ - Optimized for: complex aggregations                   │
│ - Workload: read-only bulk analysis                     │
│ - Consistency: eventual                                 │
│ - Example: "revenue by region per month"                │
└─────────────────────────────────────────────────────────┘
```

**Типичные ошибки.**

- Выбрать NoSQL для любых данных — многие "NoSQL" проекты потом сожалели.
- Забыть про write sharding в SQL — если write QPS высокий, SQL требует особого внимания.
- Использовать SQL для 1B+ documents без шардинга — тупик.
- Выбрать Graph DB потому что это модно — нужны реальные отношения.
- Забыть про hybrid approach — большинство крупных систем используют несколько БД.

**На интервью.**

- Спроси о access patterns: как читают, как пишут?
- Спроси о scale: DAU, QPS, growth rate?
- Спроси о consistency requirements.
- Приведи примеры реальных систем: Uber (SQL + NoSQL hybrid), Netflix (Cassandra for speed).
- Не бойся менять мнение: "Actually, given write QPS is 1M, NoSQL is better..."
- Уточняющий вопрос: "How would you denormalize data in NoSQL?" или "How would you shard this in SQL?"

---

### 4. Какие есть паттерны масштабирования?

**Зачем спрашивают.** Масштабирование — не о выборе большего сервера. Паттерны масштабирования определяют архитектурные решения. Интервьюер проверяет знание horizontal vs vertical, load balancing, sharding.

**Короткий ответ.** Основные паттерны: Vertical Scaling (больше CPU/RAM одного сервера), Horizontal Scaling (больше серверов), Read Replicas (для read-heavy), Database Sharding (для write-heavy), Caching (для горячих данных), CDN (для статики).

**Детальный разбор.**

**1. Vertical Scaling vs Horizontal Scaling:**

```
┌─────────────────────────────────────────────────────────────┐
│ Vertical Scaling (Scale Up)                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Before:        After:                                       │
│ ┌──────────┐   ┌────────────────┐                           │
│ │ Server   │   │ Server         │                           │
│ │ CPU: 4   │   │ CPU: 32        │                           │
│ │ RAM: 8GB │   │ RAM: 256GB     │                           │
│ └──────────┘   └────────────────┘                           │
│                                                             │
│ ✓ Pros:                      ✗ Cons:                        │
│ - Simple: no code changes    - Expensive (exponential)      │
│ - No distributed complexity  - Has ceiling (biggest server) │
│ - No sharding needed         - Single point of failure      │
│                              - Downtime for upgrade         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Horizontal Scaling (Scale Out)                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Before:        After:                                       │
│ ┌──────────┐   ┌───────┬───────┬───────┐                    │
│ │ Server   │   │Server │Server │Server │                    │
│ │ (1x)     │   │ (1x)  │ (1x)  │ (1x)  │                    │
│ └──────────┘   └───────┴───────┴───────┘                    │
│                                                             │
│ ✓ Pros:                      ✗ Cons:                        │
│ - Nearly unlimited scale      - Complex architecture        │
│ - Cost linear                 - Distributed system issues   │
│ - No single point of failure  - Sharding strategy needed    │
│ - Good for cloud              - Debugging harder            │
│                               - More moving parts           │
└─────────────────────────────────────────────────────────────┘
```

**Vertical Scaling Strategy:**

```
Starting point: 1 server
┌─────────────┐
│ Web/API     │ ← CPU bound? Add CPU
│ Database    │ ← Memory bound? Add RAM
│ Cache       │ ← Storage bound? Add disk
└─────────────┘

Limitations:
1. Cost: 64-core machine costs > 8 × 8-core machine
2. Physics: Biggest server ≈ 1-2TB RAM, 128 CPU
3. Downtime: Upgrade requires restart (if single instance)
4. Availability: Single machine can fail

Когда использовать:
- Early stage: < 10M QPS
- Starting point before horizontal scaling
- Simple applications

Example:
- 100K QPS → One big server might work ($10K/month)
- 1M QPS → Vertical scaling at limit, need horizontal
```

**Horizontal Scaling Strategy:**

```
Starting point: 1 server at limit
   ↓ (Add load balancer)
┌──────────────────────────────────────────┐
│           Load Balancer                  │
│  (Nginx, HAProxy, cloud LB)              │
└────┬────────────┬─────────────┬──────────┘
     │            │             │
┌────▼──┐   ┌─────▼──┐   ┌──────▼──┐
│Server1│   │Server2 │   │Server N │
└───────┘   └────────┘   └─────────┘

Ключевое правило: stateless серверы
- Каждый запрос может идти на любой сервер
- Не требуется состояние между серверами
- Легко добавлять/удалять серверы

Стратегии load balancing:
1. Round-robin: server1, server2, server1, server2...
2. Least connections: pick server with fewest active connections
3. IP hash: same client → same server (sticky sessions)
4. Weighted: big server gets more traffic

Когда использовать:
- После достижения предела вертикального масштабирования
- Требуется высокая доступность
- Можно сделать серверы stateless
```

**2. Read Replicas (для read-heavy систем):**

```
Архитектура:
┌────────────────────────────────────────────┐
│ Application                                │
├────────────────────────────────────────────┤
│         Writes → [Primary DB]              │
│         Reads  → [Read Replicas]           │
└────┬──────────────────────┬────────────────┘
     │                      │
     │                      │
┌────▼──────────────┐   ┌───▼────┬─────────┐
│  Primary (Write)  │   │ Replica│ Replica │
│  PostgreSQL       │───►   1    │   2     │
└───────────────────┘   └────────┴─────────┘
                         (read-only)

Как это работает:
1. Write → Primary
2. Primary → Write-Ahead Log (WAL)
3. Primary → Stream WAL to replicas (replication lag = 100ms)
4. Reads → Can go to any replica
5. If primary fails → Elect new primary from replicas

Преимущества:
- Read throughput = replicas × primary_throughput
- Primary не перегружена читами
- Read failover (если одна replica упадёт, другие работают)

Недостатки:
- Replication lag: read might return old data
- Consistency challenges: reads see different versions
- Master failure requires election (downtime)

Example:
- Primary: 1K write QPS, 100K read QPS → primary bottleneck
- Solution: Add 3 replicas → 300K read capacity
- Result: Primary at 1K QPS, replicas handle 99K QPS each

Использовать когда:
- Read-heavy рабочая нагрузка (read:write > 10:1)
- Можно допустить eventual consistency
- Примеры: Twitter (feed reads), YouTube (video plays)
```

**3. Database Sharding (для write-heavy систем):**

```
Sharding = Горизонтальное партиционирование базы данных

Before (bottleneck единственной базы данных):
┌──────────────────────────────────────┐
│ All users → Single Database          │
│ write QPS limit ≈ 10K-50K per server │
│ If you need 100K write QPS = stuck!  │
└──────────────────────────────────────┘

After (sharded):
┌────────────────────────────────────────────────────────┐
│ Hash(user_id) % 4 = shard number                       │
├────┬────┬────┬────┬────────────────────────────────────┤
│ 0  │ 1  │ 2  │ 3  │                                    │
└─┬──┴─┬──┴─┬──┴─┬──┘                                    │
  │    │    │    │                                       │
┌─▼─┐┌─▼─┐┌─▼─┐┌─▼─┐                                     │
│DB0││DB1││DB2││DB3│  Each shard = independent DB        │
│50K││50K││50K││50K│ QPS  Total: 200K write QPS!         │
└───┘└───┘└───┘└───┘                                     │

Стратегии выбора sharding key:
1. Hash-based: hash(user_id) % num_shards
   - Равномерно распределяет
   - Проблема: если num_shards меняется, нужен re-hash

2. Range-based: диапазоны user_id
   - [0-1M] → shard 0
   - [1M-2M] → shard 1
   - Проблема: может стать несбалансированным ("hot shard")

3. Directory-based: таблица поиска
   - user_id → shard_id (хранится в metadata service)
   - Гибко: можно менять назначения
   - Проблема: дополнительный lookup

4. Geographic: регион пользователя
   - EU пользователи → EU shard
   - US пользователи → US shard
   - Преимущество: data residency, latency

Трудности sharding:
1. Кросс-shard запросы становятся сложными
   - "Найти пользователя с наивысшим баллом" = нужно запросить все shards
   - Решение: scatter-gather запрос

2. Joins не работают между shards
   - Решение: denormalize, async resolution

3. Неравномерное распределение (hot shards)
   - Решение: re-sharding (дорого), consistent hashing

4. Транзакции между shards
   - Решение: 2-phase commit (медленно) или eventual consistency

Пример: Instagram с 1 миллиардом пользователей
- Невозможно на одной БД
- Решение: Shard по user_id, 256 shards
- Каждый shard на отдельном кластере
- Каждый кластер сам реплицирован для HA
```

**4. Caching (для горячих данных):**

```
Размещение cache:

┌──────────────────────────────────────────────────┐
│ Client              Browser Cache                │
│ (Cache Busting: add ?v=123 to URLs)              │
└──────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│ CDN Cache           CloudFront, Akamai           │
│ (Static assets, nearby servers)                  │
└──────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│ Application Cache   Redis, Memcached             │
│ (Hot data, nearby app server)                    │
└──────────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│ Database            PostgreSQL (slow)            │
└──────────────────────────────────────────────────┘

Ключевые insights кэширования:
- Hit ratio: выше лучше
  - 80% hit ratio: 80% reads из cache
  - Означает: 80% избежали медленной БД
  - 20% попадают в cache, 80% × скорость = в 4× быстрее в среднем

- Инвалидация: когда удалять cached данные
  Вариант 1: TTL (Time-To-Live)
    cache.set("user:123", data, ttl=3600)  # 1 час
    Просто, но может отдавать stale данные

  Вариант 2: Event-based
    Когда пользователь обновляет → delete cache
    Сразу консистентно, но сложно

  Вариант 3: Cache-aside с версионированием
    cache.set("user:123:v5", data)
    Обновляем schema → v6
    Старая версия естественно expires из памяти

Пример: Social Media Profile
- 1M QPS read запросов для профилей
- Без cache: 1M запросов к БД/секунду = медленно
- С cache: 95% hit ratio
  - 950K из cache (быстро)
  - 50K из database
  - Средняя latency: 95% × 10ms + 5% × 100ms = 14.5ms
  - Без cache: 100ms всегда
  - Результат: в 7 раз быстрее!
```

**5. CDN для статического контента:**

```
Без CDN:
User в Австралии → запрашивает изображение → сервер в Калифорнии
            Network latency = 150ms
            ← изображение (150ms позже)

С CDN:
User в Австралии → запрос → CDN edge в Сиднее
            Network latency = 10ms
            ← изображение из cache (10ms)

Как это работает:
1. Загружаем assets на origin сервер
2. CDN кэширует на edge locations по всему миру
3. User запросы → маршрутизируются на ближайший edge
4. Edge имеет → служит немедленно (cache hit)
5. Edge не имеет → fetch с origin, cache, serve

Преимущества:
- Latency: served из ближайшего сервера
- Origin нагрузка: в 1000 раз меньше запросов к origin
- Bandwidth: дешевле с CDN
- Resilience: outage одного origin не влияет на edge caches

Иерархия cache:
Browser cache (контролируешь ты, etag)
   ↑
CDN (контролирует провайдер, обычно 24 часа)
   ↑
Origin (твой сервер)

Пример: Netflix
- Хранит 100+ PB фильмов глобально
- Использует Open Connect CDN (кэширует на уровне ISP)
- 99% трафика served из ISP cache
- Origin редко hit
- Результат: fast playback, низкая latency
```

**Complete Scaling Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│ Clients (Web, Mobile, Desktop)                               │
└────────────────┬─────────────────────────────────────────────┘
                 │
     ┌───────────▼───────────┐
     │   CDN (CloudFront)    │  ← Static assets
     │   Cache static files  │
     └───────────┬───────────┘
                 │
     ┌───────────▼──────────────┐
     │  Load Balancer (Nginx)   │  ← Распределяем трафик
     └───────────┬──────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼──┐    ┌────▼───┐   ┌────▼───┐
│App 1 │    │ App 2  │   │ App N  │   ← Horizontal scaling
│stat- │    │stat-   │   │stat-   │     (stateless серверы)
│less  │    │less    │   │less    │
└───┬──┘    └────┬───┘   └────┬───┘
    │            │            │
    └────────────┼────────────┘
                 │
     ┌───────────▼───────────────────────┐
     │  Cache Layer (Redis)              │  ← Hot data
     │  (user sessions, popular items)   │
     └───────────┬───────────────────────┘
                 │
    ┌────────────┼──────────────────┐
    │            │                  │
┌───▼──────┐ ┌───▼──────┐    ┌──────▼──┐
│Primary DB│ │Replica 1 │    │Replica 2│  ← Read replicas
│(write)   │ │ (read)   │    │  (read) │
└────┬─────┘ └──────────┘    └─────────┘
     │
 [Replication] ← WAL streaming


Для write-heavy системы добавьте sharding:

Primary будет разделён на:
┌─────────────────────────────────────────────────┐
│ Shard assignment based on user_id               │
├───────┬───────┬───────┬───────┐                 │
│Shard0 │Shard1 │Shard2 │Shard3 │ каждый shard    │
│Primary│Primary│Primary│Primary│ имеет replicas  │
└────┬──┴───┬───┴───┬───┴───┬───┘                 │
     │      │       │       └─────────────────────┘
   Replicas для каждого shard
```

**Типичные ошибки.**

- Вертикальное масштабирование как решение навсегда — в какой-то момент упирается в потолок.
- Горизонтальное масштабирование без stateless design — горутины теряются между серверами.
- Забыть про read replicas для read-heavy систем — primary перегружена.
- Шардинг без надлежащей планировки — uneven shard distribution (hot shards).
- Cache без инвалидации — stale data на неделю.

**На интервью.**

- Нарисуй диаграмму с load balancer, multiple app servers, cache, replicas.
- Объясни, как каждый компонент решает определённый bottleneck.
- Обсуди, когда какой паттерн применяется (vertical → horizontal → read replicas → sharding → cache → CDN).
- Упомяни trade-offs: сложность operational vs масштабируемость.
- Уточняющий вопрос: "What if you shard by user_id but need to query across all users?" → scatter-gather, analytics DB.

---

### 5. Как работает consistent hashing?

**Зачем спрашивают.** Consistent hashing решает проблему переболивания (rehashing) при добавлении новых серверов. Интервьюер проверяет понимание распределения нагрузки и масштабирования.

**Короткий ответ.** Consistent hashing размещает ключи и серверы на кольце. При добавлении/удалении сервера только часть ключей (1/n) переназначается, вместо полного re-hash. Используется для кэшей, шардинга, load balancing.

**Детальный разбор.**

**Проблема с naive hashing:**

```
Naive approach: hash(key) % num_servers

Example: 3 servers, 100 keys
┌──────────────────────────────────┐
│ Server mapping:                  │
│ key % 3 = 0 → Server 0           │
│ key % 3 = 1 → Server 1           │
│ key % 3 = 2 → Server 2           │
└──────────────────────────────────┘

Distribution: 33 keys per server (balanced)

Problem: Add one server
┌──────────────────────────────────┐
│ Now 4 servers, but formula:      │
│ key % 4 = ??? (completely different!)
│                                  │
│ key % 3 = 0 → used Server 0      │
│ key % 4 = 0 → Server 0 (maybe)   │
│ key % 4 = 1 → ??? (98 keys move) │
│ key % 4 = 2 → ???                │
│ key % 4 = 3 → ???                │
│                                  │
│ Result: 99% cache miss!          │
│ All keys must be re-hashed       │
│ Network: 100K keys × network     │
│ = massive shuffle, database hits │
└──────────────────────────────────┘
```

**Consistent Hashing Solution:**

```
Concept: Arrange servers and keys on a ring (0 to 2^32-1)

Step 1: Hash servers & keys to ring
┌─────────────────────────────────────┐
│      Consistent Hash Ring           │
│                                     │
│            0: Server0_hash          │
│                                     │
│   100:                  |           │
│                     Key5 (hash=87)  │
│                                     │
│   200: Server1_hash                 │
│                          Key3       │
│                        (hash=180)   │
│                                     │
│   300: Key1 (hash=250)              │
│        |                            │
│        Server2_hash                 │
│        (hash=320)                   │
│                                     │
│        Key2 (hash=400)              │
│                                     │
│   2^32-1 (= 0, wrap around)         │
└─────────────────────────────────────┘

Step 2: Assign key to server
Rule: Find closest server clockwise from key

┌─────────────────────────────────────┐
│ Key assignments:                    │
│                                     │
│ Key5 (hash=87) → Server0 (hash=0)   │
│ Key3 (hash=180) → Server1 (hash=200)
│ Key1 (hash=250) → Server2 (hash=320)
│ Key2 (hash=400) → Server0 (wrap)    │
└─────────────────────────────────────┘
```

**Adding Server (scaling without rehash):**

```
Before: 3 servers
┌─────────────────────────────┐
│ 0: Server0  (65 keys)       │
│ 200: Server1 (35 keys)      │
│ 320: Server2 (0 keys)       │
│ (other keys wrap to Server0)│
└─────────────────────────────┘

Add Server3 at hash=150
┌─────────────────────────────┐
│ 0: Server0                  │
│ 150: Server3 (NEW!)         │
│ 200: Server1                │
│ 320: Server2                │
└─────────────────────────────┘

Воздействие:
Only keys between 0 and 150 move from Server0 to Server3
Keys between 150 and 200 move from Server0 to Server3
Результат: Only ~33% of keys rehashed (instead of 99%!)

Actual: ~1/n of keys rehash when adding 1 new server
n = number of servers
```

**Ring Implementation Detail:**

```python
class ConsistentHash:
    def __init__(self, servers, replicas=3):
        """
        replicas: number of virtual nodes per server
        Virtual nodes improve balance and reduce hotspots
        """
        self.ring = {}  # hash -> server
        self.servers = servers
        self.replicas = replicas

        for server in servers:
            for i in range(replicas):
                # Create virtual nodes
                virtual_key = f"{server}#{i}"
                hash_value = hash(virtual_key)
                self.ring[hash_value] = server

        self.sorted_keys = sorted(self.ring.keys())

    def get_server(self, key):
        """Find server for given key"""
        hash_value = hash(key)

        # Find first server hash >= key hash (clockwise)
        for server_hash in self.sorted_keys:
            if server_hash >= hash_value:
                return self.ring[server_hash]

        # Wrap around to first
        return self.ring[self.sorted_keys[0]]

# Example
servers = ["server0", "server1", "server2"]
ch = ConsistentHash(servers, replicas=3)

for key_id in range(100):
    server = ch.get_server(f"key_{key_id}")
    print(f"key_{key_id} → {server}")
```

**Benefits vs Naive Hashing:**

```
┌──────────────────┬──────────────────┬───────────────────┐
│ Operation        │ Naive % hashing  │ Consistent Hash   │
├──────────────────┼──────────────────┼───────────────────┤
│ Add 1 server     │ 99% keys rehash  │ 1/n keys rehash   │
│ (n=10 servers)   │ (massive pain)   │ (10% keys)        │
├──────────────────┼──────────────────┼───────────────────┤
│ Remove server    │ Same 99% rehash  │ 1/(n-1) keys      │
├──────────────────┼──────────────────┼───────────────────┤
│ Scaling up/down  │ Not feasible     │ Graceful          │
├──────────────────┼──────────────────┼───────────────────┤
│ Load balance     │ OK (if n stable) │ Better (virtual   │
│                  │                  │ nodes)            │
└──────────────────┴──────────────────┴───────────────────┘
```

**Real-World Examples:**

```
1. Cache distribution (Redis, Memcached):
   ├─ N cache servers
   ├─ Key → hash → server
   ├─ Client code:
   │  server = get_consistent_hash_server(key)
   │  value = redis_servers[server].get(key)
   │
   └─ Benefit: Add cache server without flushing everything

2. Database sharding:
   ├─ N database shards
   ├─ User ID → hash → shard
   ├─ Benefit: Adding shard only migrates ~10% of data
   │
   └─ Problem: still need migration tool to move data

3. Load balancing:
   ├─ N backend servers
   ├─ Client IP → hash → server
   ├─ Sticky sessions: same client → same server
   │ (without consistent hash, client could jump between servers)
   │
   └─ Benefit: Session affinity with minimal rehashing

4. Distributed systems (Cassandra, DynamoDB):
   ├─ Nodes arranged on ring
   ├─ Data partitioned by consistent hash
   ├─ Replication: key goes to N nodes (e.g., 3)
   │
   └─ Benefit: Node failure only affects small fraction of data
```

**Virtual Nodes для балансировки:**

```
Without virtual nodes:
┌────────────────────────────────┐
│ Ring with 3 physical servers   │
│                                │
│ 0: Server0                     │
│ 500: Server1                   │
│ 700: Server2                   │
│                                │
│ Imbalance: if Server0 hash     │
│ is far from others, it gets    │
│ more keys                      │
└────────────────────────────────┘

With virtual nodes (3 per server):
┌────────────────────────────────────┐
│ Ring with 3 servers, 3 vnodes each │
│                                    │
│ 0: Server0                         │
│ 100: Server0 (virtual)             │
│ 200: Server0 (virtual)             │
│ 300: Server1                       │
│ 400: Server1 (virtual)             │
│ 500: Server1 (virtual)             │
│ 600: Server2                       │
│ 700: Server2 (virtual)             │
│ 800: Server2 (virtual)             │
│                                    │
│ Better balance: replicas spread    │
│ Keys distributed evenly            │
└────────────────────────────────────┘

Result:
- Server2 gets ~1/3 of keys (fair)
- Instead of 0 keys (if hash poorly placed)
```

**Пример на интервью:**

```
Интервьюер: "Design cache layer for 1M QPS. How would you
distribute keys across cache servers?"

Мой ответ:

"I'd use consistent hashing:

1. Architecture:
   - N cache servers (e.g., 10)
   - Each key hashed to server
   - Consistent hash ring with virtual nodes

2. Implementation:
   ring = ConsistentHash(servers=['cache0', 'cache1', ...],
                         replicas=3)

   For write:
   server = ring.get_server(key)
   server.set(key, value, ttl=3600)

   For read:
   server = ring.get_server(key)
   value = server.get(key)

3. Why consistent hashing?
   - Current: 10 servers, balanced
   - Add server 11: only ~10% of keys rehash
   - Without it: 90% would move (catastrophic)
   - Network saved: 100M keys × 90% × network = avoided

4. Replication for HA:
   - Store key on 3 closest servers in ring
   - If cache0 down, read from cache1 or cache2
   - Automatic failover

5. Scaling:
   - Add cache server → 10% cache keys migrate
   - Old servers → warm new server with data
   - No downtime

Компромиссы:
- Small complexity in client code
- vs massive rehashing burden (naive approach)
- Good investment for scale"
```

**Типичные ошибки.**

- Неправильная реализация поиска server на кольце — off-by-one errors при бинарном поиске.
- Забыть про virtual nodes — неравномерное распределение.
- Забыть про replication — single server failure = data loss.
- Использовать weak hash function — скопление ключей.

**На интервью.**

- Объясни, почему consistent hash лучше naive % hashing.
- Нарисуй ring с несколькими серверами и ключами.
- Покажи, как добавление сервера влияет на распределение.
- Упомяни virtual nodes для балансировки.
- Уточняющий вопрос: "What if we need replication?" → multiple nodes clockwise.

---

## См. также

- [Основы архитектуры](../08-architecture/00-architecture-fundamentals.md) — паттерны и принципы построения архитектуры систем
- [Заблуждения о распределённых системах](../07-distributed-systems/00-fallacies-fundamentals.md) — типичные ошибки при проектировании распределённых систем

---

## Практика

1. **CAP Trade-offs** — какую сторону CAP выбрать для: (a) payment system, (b) recommendations, (c) activity feed? Объясни решение.

2. **Consistency Models** — для каждого компонента социальной сети выбери consistency model:
   - User authentication
   - Friend connections
   - Post feeds
   - Like counter
   - Comments

3. **SQL vs NoSQL** — спроектируй schema для e-commerce:
   - Users (fixed schema) — SQL or NoSQL?
   - Products (flexible attributes) — SQL or NoSQL?
   - Inventory (transactional) — SQL or NoSQL?
   - Search index — SQL or NoSQL?

4. **Scaling Design** — дизайн scalability для 1M QPS:
   - Вертикальное масштабирование: почему недостаточно?
   - Горизонтальное: как сделать servers stateless?
   - Caching: где размещать? Как инвалидировать?
   - Sharding: стратегия выбора ключа?

5. **Consistent Hashing** — реализуй simple consistent hash ring для cache servers:
   - Добавь 5 серверов на ring
   - Распредели 100 ключей
   - Добавь 6-й сервер: сколько ключей переместилось?

---

## Дополнительные материалы

- [Designing Data-Intensive Applications](https://dataintensive.net/) — классическая книга про distributed systems
- [System Design Primer](https://github.com/donnemartin/system-design-primer) — GitHub собрание ресурсов
- [Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview) — практический курс с примерами
- [High Scalability Blog](http://highscalability.com/) — case studies реальных систем
- [Google Cloud Architecture Center](https://cloud.google.com/architecture) — примеры production архитектур

---

← [Назад к списку тем](./README.md) | [Трек System Design](./README.md) | [01-url-shortener](./01-url-shortener.md) →
