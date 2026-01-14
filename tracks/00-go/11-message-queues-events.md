# 11 — Message Queues & Events

Разбор вопросов о работе с очередями сообщений: Kafka, NATS, RabbitMQ, event sourcing, exactly-once семантика и обработка ошибок.

**Навигация:** [← 10-distributed-systems](./10-distributed-systems.md) | [Трек Go](./README.md) | [12-security-crypto →](./12-security-crypto.md)

---

## Вопросы и разборы

### 1. Как работать с Kafka в Go?

**Зачем спрашивают.**
Kafka — стандарт для event streaming. Интервьюер проверяет практический опыт работы с библиотеками и понимание концепций.

**Короткий ответ.**
Основные клиенты: sarama (чистый Go), confluent-kafka-go (обёртка над librdkafka). Producer отправляет сообщения в topics, Consumer читает из partitions. Важны: acks, offsets, consumer groups.

**Детальный разбор.**

**Архитектура Kafka:**
```
┌─────────────────────────────────────────────────────────────┐
│                         Topic: orders                        │
│                                                              │
│   Partition 0    Partition 1    Partition 2                 │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│   │ 0│1│2│3 │    │ 0│1│2│3 │    │ 0│1│2   │                │
│   └────┬────┘    └────┬────┘    └────┬────┘                │
│        │              │              │                       │
│        ▼              ▼              ▼                       │
│   Consumer A     Consumer B     Consumer C                   │
│   (Group: myapp)                                             │
│                                                              │
│   Key-based partitioning:                                   │
│   order_id % num_partitions → partition                     │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**confluent-kafka-go (рекомендуется для production):**
```go
import "github.com/confluentinc/confluent-kafka-go/v2/kafka"

// Producer
func NewProducer(brokers string) (*kafka.Producer, error) {
    return kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers":   brokers,
        "acks":                "all",        // ждать подтверждения от всех replicas
        "retries":             3,
        "retry.backoff.ms":    100,
        "enable.idempotence":  true,         // exactly-once на уровне producer
        "linger.ms":           5,            // batch небольших сообщений
        "compression.type":    "snappy",
    })
}

func Produce(p *kafka.Producer, topic string, key, value []byte) error {
    deliveryChan := make(chan kafka.Event)

    err := p.Produce(&kafka.Message{
        TopicPartition: kafka.TopicPartition{
            Topic:     &topic,
            Partition: kafka.PartitionAny,
        },
        Key:   key,
        Value: value,
    }, deliveryChan)

    if err != nil {
        return err
    }

    // Ждём подтверждения
    e := <-deliveryChan
    msg := e.(*kafka.Message)

    if msg.TopicPartition.Error != nil {
        return msg.TopicPartition.Error
    }

    return nil
}

// Async producer с callback
func ProduceAsync(p *kafka.Producer, topic string, messages []Message) {
    for _, msg := range messages {
        p.Produce(&kafka.Message{
            TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
            Key:            msg.Key,
            Value:          msg.Value,
        }, nil)
    }

    // Flush с таймаутом
    p.Flush(5000)
}

// Обработка delivery reports
func handleDeliveryReports(p *kafka.Producer) {
    for e := range p.Events() {
        switch ev := e.(type) {
        case *kafka.Message:
            if ev.TopicPartition.Error != nil {
                log.Printf("Delivery failed: %v", ev.TopicPartition.Error)
            } else {
                log.Printf("Delivered to %v", ev.TopicPartition)
            }
        }
    }
}
```

**Consumer:**
```go
func NewConsumer(brokers, groupID string) (*kafka.Consumer, error) {
    return kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers":  brokers,
        "group.id":           groupID,
        "auto.offset.reset":  "earliest",    // или "latest"
        "enable.auto.commit": false,          // manual commit
        "session.timeout.ms": 30000,
        "max.poll.interval.ms": 300000,       // max время между poll
    })
}

func Consume(c *kafka.Consumer, topics []string, handler func(msg *kafka.Message) error) error {
    if err := c.SubscribeTopics(topics, nil); err != nil {
        return err
    }

    for {
        msg, err := c.ReadMessage(time.Second)
        if err != nil {
            // Timeout — нормально
            if err.(kafka.Error).Code() == kafka.ErrTimedOut {
                continue
            }
            return err
        }

        // Обработка
        if err := handler(msg); err != nil {
            log.Printf("Handler error: %v", err)
            // Решаем: retry, DLQ, или пропустить
            continue
        }

        // Commit после успешной обработки
        if _, err := c.CommitMessage(msg); err != nil {
            log.Printf("Commit error: %v", err)
        }
    }
}
```

**sarama (чистый Go):**
```go
import "github.com/IBM/sarama"

func NewSaramaProducer(brokers []string) (sarama.SyncProducer, error) {
    config := sarama.NewConfig()
    config.Producer.RequiredAcks = sarama.WaitForAll
    config.Producer.Retry.Max = 3
    config.Producer.Return.Successes = true
    config.Producer.Idempotent = true
    config.Net.MaxOpenRequests = 1  // для idempotent

    return sarama.NewSyncProducer(brokers, config)
}

func NewSaramaConsumerGroup(brokers []string, groupID string) (sarama.ConsumerGroup, error) {
    config := sarama.NewConfig()
    config.Consumer.Group.Rebalance.Strategy = sarama.NewBalanceStrategyRoundRobin()
    config.Consumer.Offsets.Initial = sarama.OffsetOldest
    config.Consumer.Offsets.AutoCommit.Enable = false

    return sarama.NewConsumerGroup(brokers, groupID, config)
}

// ConsumerGroupHandler interface
type handler struct{}

func (h *handler) Setup(sarama.ConsumerGroupSession) error   { return nil }
func (h *handler) Cleanup(sarama.ConsumerGroupSession) error { return nil }

func (h *handler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for msg := range claim.Messages() {
        // Обработка
        process(msg)

        // Mark as processed
        session.MarkMessage(msg, "")
    }
    return nil
}
```

**Типичные ошибки.**
1. **auto.commit без обработки** — потеря сообщений при crash
2. **Синхронный produce в цикле** — низкая производительность
3. **Нет обработки rebalance** — теряются in-flight сообщения
4. **Большие сообщения** — лучше reference на storage

**На интервью.**
Объясни разницу между sarama и confluent-kafka-go. Покажи правильный commit после обработки. Расскажи про partitioning и ordering guarantees.

*Частые follow-up вопросы:*
- Как обеспечить ordering в Kafka?
- Что такое consumer lag?
- Как масштабировать consumers?

---

### 2. Что такое Consumer Groups и как балансировать нагрузку?

**Зачем спрашивают.**
Consumer groups — основа масштабирования. Интервьюер проверяет понимание rebalancing, partition assignment и координации.

**Короткий ответ.**
Consumer group — группа consumers, совместно читающих topic. Каждая partition назначается одному consumer в группе. При добавлении/удалении consumers происходит rebalance.

**Детальный разбор.**

**Partition assignment:**
```
┌─────────────────────────────────────────────────────────────┐
│ Topic: orders (6 partitions)                                 │
│                                                              │
│ Consumer Group A:                                            │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│ │Consumer 1│  │Consumer 2│  │Consumer 3│                    │
│ │ P0, P1   │  │ P2, P3   │  │ P4, P5   │                    │
│ └──────────┘  └──────────┘  └──────────┘                    │
│                                                              │
│ Consumer Group B (другое приложение):                        │
│ ┌──────────┐  ┌──────────┐                                  │
│ │Consumer 1│  │Consumer 2│                                  │
│ │P0, P1, P2│  │P3, P4, P5│                                  │
│ └──────────┘  └──────────┘                                  │
│                                                              │
│ Важно: consumers > partitions = idle consumers              │
└─────────────────────────────────────────────────────────────┘
```

**Rebalance:**
```
┌─────────────────────────────────────────────────────────────┐
│ Триггеры rebalance:                                          │
│ 1. Consumer join (новый instance)                           │
│ 2. Consumer leave (graceful shutdown)                       │
│ 3. Consumer crash (session timeout)                         │
│ 4. Topic изменения (новые partitions)                       │
│                                                              │
│ Во время rebalance:                                          │
│ • Все consumers останавливают обработку                     │
│ • Новое assignment вычисляется                              │
│ • Consumers получают новые partitions                       │
│ • Обработка возобновляется                                  │
│                                                              │
│ Проблема: Stop-the-world во время rebalance                 │
│ Решение: Cooperative rebalancing (incremental)              │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Cooperative rebalancing:**
```go
config := sarama.NewConfig()

// Cooperative sticky — минимизирует partition transfers
config.Consumer.Group.Rebalance.Strategy = sarama.NewBalanceStrategySticky()

// Или range для предсказуемого assignment
config.Consumer.Group.Rebalance.Strategy = sarama.NewBalanceStrategyRange()
```

**Graceful shutdown для минимизации rebalance:**
```go
func RunConsumer(ctx context.Context, group sarama.ConsumerGroup, topics []string) error {
    handler := &ConsumerHandler{
        ready: make(chan bool),
    }

    wg := &sync.WaitGroup{}
    wg.Add(1)

    go func() {
        defer wg.Done()
        for {
            // Consume блокируется до rebalance
            if err := group.Consume(ctx, topics, handler); err != nil {
                log.Printf("Error: %v", err)
            }

            if ctx.Err() != nil {
                return
            }

            handler.ready = make(chan bool)
        }
    }()

    <-handler.ready  // Ждём ready

    // Graceful shutdown
    <-ctx.Done()
    log.Println("Shutting down consumer...")

    // Даём время на commit текущих сообщений
    wg.Wait()

    return group.Close()
}

type ConsumerHandler struct {
    ready chan bool
}

func (h *ConsumerHandler) Setup(session sarama.ConsumerGroupSession) error {
    close(h.ready)  // Signal ready
    return nil
}

func (h *ConsumerHandler) Cleanup(session sarama.ConsumerGroupSession) error {
    // Commit pending offsets
    return nil
}

func (h *ConsumerHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for {
        select {
        case msg, ok := <-claim.Messages():
            if !ok {
                return nil
            }
            process(msg)
            session.MarkMessage(msg, "")

        case <-session.Context().Done():
            return nil
        }
    }
}
```

**Static membership (Kafka 2.3+):**
```go
// Static member — не триггерит rebalance при restart
config := &kafka.ConfigMap{
    "bootstrap.servers": brokers,
    "group.id":          groupID,
    "group.instance.id": hostname,  // Уникальный static ID
    "session.timeout.ms": 30000,
}

// При restart с тем же instance.id — partitions остаются
```

**Мониторинг consumer lag:**
```go
func GetConsumerLag(admin sarama.ClusterAdmin, group, topic string) (map[int32]int64, error) {
    // Получаем offsets группы
    offsets, err := admin.ListConsumerGroupOffsets(group, nil)
    if err != nil {
        return nil, err
    }

    // Получаем high watermarks
    client, _ := sarama.NewClient(brokers, nil)
    defer client.Close()

    lag := make(map[int32]int64)
    partitions, _ := client.Partitions(topic)

    for _, p := range partitions {
        hwm, _ := client.GetOffset(topic, p, sarama.OffsetNewest)
        committed := offsets.Blocks[topic][p].Offset

        lag[p] = hwm - committed
    }

    return lag, nil
}
```

**Типичные ошибки.**
1. **Больше consumers чем partitions** — часть idle
2. **Долгая обработка** — session timeout, rebalance loop
3. **Нет static membership** — частые rebalance при deploys
4. **Игнорировать lag** — consumers не успевают

**На интервью.**
Объясни, как работает partition assignment. Покажи, как минимизировать rebalance. Расскажи про static membership.

*Частые follow-up вопросы:*
- Сколько partitions нужно?
- Как обрабатывать rebalance в middleware?
- Что такое consumer lag и как мониторить?

---

### 3. Как гарантировать exactly-once семантику?

**Зачем спрашивают.**
Exactly-once — святой грааль message processing. Интервьюер проверяет понимание ограничений и практических подходов.

**Короткий ответ.**
True exactly-once невозможен в распределённых системах. Практические подходы: idempotent producer + transactional consumer, или idempotent processing на стороне consumer.

**Детальный разбор.**

**Семантики доставки:**
```
┌─────────────────────────────────────────────────────────────┐
│ At-most-once:                                                │
│ • Fire and forget                                           │
│ • Сообщение может потеряться                                │
│ • Никогда не дублируется                                    │
│                                                              │
│ At-least-once:                                               │
│ • Retry при ошибках                                         │
│ • Сообщение не теряется                                     │
│ • Может дублироваться                                       │
│                                                              │
│ Exactly-once:                                                │
│ • Каждое сообщение обрабатывается ровно один раз           │
│ • Требует специальных механизмов                            │
│ • Часто = at-least-once + idempotency                       │
└─────────────────────────────────────────────────────────────┘
```

**Kafka Transactions (EOS — Exactly-Once Semantics):**
```
┌─────────────────────────────────────────────────────────────┐
│ Transactional producer + consumer:                           │
│                                                              │
│ 1. Consumer читает из input topic                           │
│ 2. Producer начинает транзакцию                             │
│ 3. Producer отправляет в output topic                       │
│ 4. Producer commit'ит consumer offset                       │
│ 5. Producer commit'ит транзакцию                            │
│                                                              │
│ Всё атомарно: либо всё успешно, либо всё откатывается       │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Kafka Transactions:**
```go
func NewTransactionalProducer(brokers, transactionalID string) (*kafka.Producer, error) {
    p, err := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers":   brokers,
        "transactional.id":    transactionalID,  // Уникальный ID
        "enable.idempotence":  true,
        "acks":                "all",
    })
    if err != nil {
        return nil, err
    }

    // Инициализация транзакций
    if err := p.InitTransactions(nil); err != nil {
        return nil, err
    }

    return p, nil
}

func ProcessWithTransaction(
    producer *kafka.Producer,
    consumer *kafka.Consumer,
    inputTopic, outputTopic string,
    process func([]byte) ([]byte, error),
) error {
    for {
        msg, err := consumer.ReadMessage(time.Second)
        if err != nil {
            continue
        }

        // Начинаем транзакцию
        if err := producer.BeginTransaction(); err != nil {
            return err
        }

        // Обрабатываем
        result, err := process(msg.Value)
        if err != nil {
            producer.AbortTransaction(nil)
            continue
        }

        // Отправляем результат
        producer.Produce(&kafka.Message{
            TopicPartition: kafka.TopicPartition{Topic: &outputTopic, Partition: kafka.PartitionAny},
            Value:          result,
        }, nil)

        // Commit consumer offset в транзакции
        offsets := []kafka.TopicPartition{msg.TopicPartition}
        offsets[0].Offset++

        if err := producer.SendOffsetsToTransaction(nil, offsets, consumer.GetConsumerGroupMetadata()); err != nil {
            producer.AbortTransaction(nil)
            return err
        }

        // Commit транзакции
        if err := producer.CommitTransaction(nil); err != nil {
            producer.AbortTransaction(nil)
            return err
        }
    }
}
```

**Idempotent processing (практичный подход):**
```go
type IdempotentProcessor struct {
    db       *sql.DB
    producer Producer
}

func (p *IdempotentProcessor) Process(ctx context.Context, msg Message) error {
    tx, err := p.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Проверяем, не обработано ли уже
    var exists bool
    err = tx.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM processed_messages WHERE id = $1)",
        msg.ID,
    ).Scan(&exists)
    if err != nil {
        return err
    }
    if exists {
        return nil  // Уже обработано — idempotent
    }

    // Обрабатываем
    result, err := businessLogic(msg)
    if err != nil {
        return err
    }

    // Сохраняем результат и метку обработки в одной транзакции
    _, err = tx.ExecContext(ctx,
        "INSERT INTO processed_messages (id, processed_at) VALUES ($1, $2)",
        msg.ID, time.Now(),
    )
    if err != nil {
        return err
    }

    _, err = tx.ExecContext(ctx,
        "INSERT INTO results (message_id, data) VALUES ($1, $2)",
        msg.ID, result,
    )
    if err != nil {
        return err
    }

    // Отправляем в outbox (тоже в транзакции)
    _, err = tx.ExecContext(ctx,
        "INSERT INTO outbox (topic, payload) VALUES ($1, $2)",
        "results", result,
    )
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

**Deduplication на consumer:**
```go
type DeduplicationCache struct {
    cache *lru.Cache  // или Redis
    ttl   time.Duration
}

func (d *DeduplicationCache) IsDuplicate(messageID string) bool {
    _, exists := d.cache.Get(messageID)
    return exists
}

func (d *DeduplicationCache) MarkProcessed(messageID string) {
    d.cache.Add(messageID, struct{}{})
}

func ProcessWithDedup(consumer Consumer, dedup *DeduplicationCache, handler Handler) {
    for msg := range consumer.Messages() {
        if dedup.IsDuplicate(msg.ID) {
            consumer.Commit(msg)
            continue
        }

        if err := handler.Handle(msg); err != nil {
            // Не помечаем как обработанное — retry
            continue
        }

        dedup.MarkProcessed(msg.ID)
        consumer.Commit(msg)
    }
}
```

**Типичные ошибки.**
1. **Надеяться на exactly-once без idempotency** — дубликаты будут
2. **Dedup только в памяти** — теряется при restart
3. **Не учитывать rebalance** — re-processing после rebalance
4. **TTL слишком маленький** — дубликаты после долгого retry

**На интервью.**
Объясни, почему true exactly-once невозможен. Покажи idempotent processing. Расскажи про Kafka Transactions и их ограничения.

*Частые follow-up вопросы:*
- Как выбрать TTL для deduplication?
- Когда Kafka Transactions не подходят?
- Как тестировать exactly-once?

---

### 4. Как реализовать паттерн Outbox для надёжной отправки событий?

**Зачем спрашивают.**
Outbox решает проблему dual write. Интервьюер проверяет понимание паттерна и практическую реализацию.

**Короткий ответ.**
Outbox — сохранение событий в БД вместе с бизнес-данными в одной транзакции. Отдельный процесс (relay) читает outbox и отправляет в message broker. Гарантирует at-least-once.

**Детальный разбор.**

**Проблема dual write:**
```
┌─────────────────────────────────────────────────────────────┐
│ Dual Write Problem:                                          │
│                                                              │
│   Service ─┬─► Database   (1. Save order)                   │
│            │                                                 │
│            └─► Kafka      (2. Send event)                   │
│                                                              │
│ Сценарии ошибок:                                            │
│ • DB успех, Kafka fail → событие потеряно                  │
│ • DB fail после Kafka → событие отправлено, данных нет     │
│ • Crash между операциями → неконсистентность               │
└─────────────────────────────────────────────────────────────┘
```

**Outbox pattern:**
```
┌─────────────────────────────────────────────────────────────┐
│ Outbox Solution:                                             │
│                                                              │
│   Service ──► Database (одна транзакция)                    │
│               ├── orders table                               │
│               └── outbox table                               │
│                      │                                       │
│                      ▼                                       │
│               Outbox Relay ──► Kafka                        │
│               (отдельный процесс)                           │
│                                                              │
│ Гарантии:                                                   │
│ • Атомарность: либо оба записаны, либо ничего              │
│ • At-least-once: relay retry пока не отправит              │
│ • Ordering: по created_at или sequence                      │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Схема таблицы:**
```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,  -- "Order", "User"
    aggregate_id VARCHAR(100) NOT NULL,     -- ID сущности
    event_type VARCHAR(100) NOT NULL,       -- "OrderCreated"
    payload JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    published_at TIMESTAMP WITH TIME ZONE,
    retry_count INT DEFAULT 0
);

CREATE INDEX idx_outbox_unpublished ON outbox(created_at)
    WHERE published_at IS NULL;
```

**Бизнес-операция с outbox:**
```go
type OrderService struct {
    db *sql.DB
}

func (s *OrderService) CreateOrder(ctx context.Context, order Order) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // 1. Сохраняем order
    _, err = tx.ExecContext(ctx, `
        INSERT INTO orders (id, user_id, total, status)
        VALUES ($1, $2, $3, $4)
    `, order.ID, order.UserID, order.Total, "created")
    if err != nil {
        return err
    }

    // 2. Сохраняем событие в outbox
    event := OrderCreatedEvent{
        OrderID: order.ID,
        UserID:  order.UserID,
        Total:   order.Total,
    }
    payload, _ := json.Marshal(event)

    _, err = tx.ExecContext(ctx, `
        INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
        VALUES ($1, $2, $3, $4)
    `, "Order", order.ID, "OrderCreated", payload)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

**Outbox Relay:**
```go
type OutboxRelay struct {
    db       *sql.DB
    producer Producer
    batchSize int
}

func (r *OutboxRelay) Run(ctx context.Context) error {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return nil
        case <-ticker.C:
            if err := r.processBatch(ctx); err != nil {
                log.Printf("Relay error: %v", err)
            }
        }
    }
}

func (r *OutboxRelay) processBatch(ctx context.Context) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // SELECT FOR UPDATE SKIP LOCKED для конкурентности
    rows, err := tx.QueryContext(ctx, `
        SELECT id, aggregate_type, aggregate_id, event_type, payload
        FROM outbox
        WHERE published_at IS NULL
        ORDER BY created_at
        LIMIT $1
        FOR UPDATE SKIP LOCKED
    `, r.batchSize)
    if err != nil {
        return err
    }
    defer rows.Close()

    var events []OutboxEvent
    for rows.Next() {
        var e OutboxEvent
        if err := rows.Scan(&e.ID, &e.AggregateType, &e.AggregateID, &e.EventType, &e.Payload); err != nil {
            return err
        }
        events = append(events, e)
    }

    if len(events) == 0 {
        return nil
    }

    // Отправляем в Kafka
    for _, e := range events {
        topic := fmt.Sprintf("%s-events", strings.ToLower(e.AggregateType))

        if err := r.producer.Produce(ctx, topic, e.AggregateID, e.Payload); err != nil {
            // При ошибке — retry на следующей итерации
            log.Printf("Failed to publish event %s: %v", e.ID, err)
            continue
        }

        // Помечаем как опубликованное
        _, err = tx.ExecContext(ctx,
            "UPDATE outbox SET published_at = $1 WHERE id = $2",
            time.Now(), e.ID,
        )
        if err != nil {
            return err
        }
    }

    return tx.Commit()
}
```

**Cleanup старых записей:**
```go
func (r *OutboxRelay) Cleanup(ctx context.Context, retention time.Duration) error {
    _, err := r.db.ExecContext(ctx, `
        DELETE FROM outbox
        WHERE published_at IS NOT NULL
        AND published_at < $1
    `, time.Now().Add(-retention))
    return err
}
```

**CDC (Change Data Capture) альтернатива:**
```
┌─────────────────────────────────────────────────────────────┐
│ Debezium CDC:                                                │
│                                                              │
│   Database ──► WAL/Binlog ──► Debezium ──► Kafka            │
│                                                              │
│ Преимущества:                                               │
│ • Не нужен relay процесс                                    │
│ • Меньше нагрузки на DB                                     │
│ • Ordering гарантирован                                     │
│                                                              │
│ Недостатки:                                                 │
│ • Сложнее настроить                                         │
│ • Зависимость от DB-specific формата                        │
│ • Нужен Kafka Connect                                       │
└─────────────────────────────────────────────────────────────┘
```

**Типичные ошибки.**
1. **Нет FOR UPDATE** — race condition между relay instances
2. **Нет SKIP LOCKED** — blocking при конкурентности
3. **Большой batch** — долгие транзакции
4. **Нет cleanup** — таблица растёт бесконечно

**На интервью.**
Покажи схему outbox таблицы. Объясни FOR UPDATE SKIP LOCKED. Сравни с CDC подходом.

*Частые follow-up вопросы:*
- Как гарантировать ordering событий?
- Когда использовать CDC вместо outbox?
- Как мониторить outbox lag?

---

### 5. NATS vs Kafka vs RabbitMQ — когда что использовать?

**Зачем спрашивают.**
Выбор message broker влияет на архитектуру. Интервьюер проверяет понимание trade-offs и критериев выбора.

**Короткий ответ.**
Kafka — для event streaming и replay. RabbitMQ — для традиционного message queuing. NATS — для простых pub/sub и request/reply с низкой latency.

**Детальный разбор.**

**Сравнение:**
```
┌─────────────────┬────────────────┬────────────────┬────────────────┐
│ Критерий        │ Kafka          │ RabbitMQ       │ NATS           │
├─────────────────┼────────────────┼────────────────┼────────────────┤
│ Модель          │ Log-based      │ Queue-based    │ Pub/Sub        │
│ Семантика       │ At-least-once  │ At-least-once  │ At-most-once*  │
│ Ordering        │ Per partition  │ Per queue      │ Нет гарантий   │
│ Replay          │ Да             │ Нет            │ JetStream      │
│ Throughput      │ Очень высокий  │ Высокий        │ Очень высокий  │
│ Latency         │ Средняя        │ Низкая         │ Очень низкая   │
│ Durability      │ Да             │ Да             │ JetStream      │
│ Сложность       │ Высокая        │ Средняя        │ Низкая         │
└─────────────────┴────────────────┴────────────────┴────────────────┘
* NATS Core — at-most-once, JetStream — at-least-once
```

**Когда что использовать:**
```
┌─────────────────────────────────────────────────────────────┐
│ Kafka:                                                       │
│ • Event streaming (logs, metrics, events)                   │
│ • Event sourcing                                            │
│ • Нужен replay (audit, debug, rebuild state)               │
│ • Высокий throughput важнее latency                         │
│ • Long-term retention                                       │
├─────────────────────────────────────────────────────────────┤
│ RabbitMQ:                                                    │
│ • Task queues (work distribution)                           │
│ • Сложная routing логика (exchanges, bindings)             │
│ • Priority queues                                           │
│ • Request/Reply patterns                                    │
│ • Традиционное enterprise messaging                         │
├─────────────────────────────────────────────────────────────┤
│ NATS:                                                        │
│ • Simple pub/sub                                            │
│ • Request/Reply (микросервисы)                              │
│ • IoT, edge computing                                       │
│ • Низкая latency критична                                   │
│ • Простота важнее гарантий                                  │
│ • JetStream для persistence                                 │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**NATS (простой pub/sub):**
```go
import "github.com/nats-io/nats.go"

func NATSExample() error {
    nc, err := nats.Connect(nats.DefaultURL)
    if err != nil {
        return err
    }
    defer nc.Close()

    // Subscribe
    sub, err := nc.Subscribe("orders.>", func(msg *nats.Msg) {
        log.Printf("Received: %s", string(msg.Data))
    })
    if err != nil {
        return err
    }
    defer sub.Unsubscribe()

    // Publish
    if err := nc.Publish("orders.created", []byte(`{"id": "123"}`)); err != nil {
        return err
    }

    // Request/Reply
    reply, err := nc.Request("user.get", []byte("123"), time.Second)
    if err != nil {
        return err
    }
    log.Printf("User: %s", string(reply.Data))

    return nil
}
```

**NATS JetStream (с persistence):**
```go
import "github.com/nats-io/nats.go/jetstream"

func JetStreamExample() error {
    nc, _ := nats.Connect(nats.DefaultURL)
    defer nc.Close()

    js, _ := jetstream.New(nc)

    // Создаём stream
    stream, _ := js.CreateStream(context.Background(), jetstream.StreamConfig{
        Name:     "ORDERS",
        Subjects: []string{"orders.>"},
        Storage:  jetstream.FileStorage,
        Retention: jetstream.LimitsPolicy,
        MaxAge:   24 * time.Hour,
    })

    // Publish
    js.Publish(context.Background(), "orders.created", []byte(`{"id": "123"}`))

    // Consumer
    consumer, _ := stream.CreateOrUpdateConsumer(context.Background(), jetstream.ConsumerConfig{
        Durable:   "order-processor",
        AckPolicy: jetstream.AckExplicitPolicy,
    })

    // Consume
    msgs, _ := consumer.Consume(func(msg jetstream.Msg) {
        log.Printf("Received: %s", string(msg.Data()))
        msg.Ack()
    })
    defer msgs.Stop()

    return nil
}
```

**RabbitMQ:**
```go
import amqp "github.com/rabbitmq/amqp091-go"

func RabbitMQExample() error {
    conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
    defer conn.Close()

    ch, _ := conn.Channel()
    defer ch.Close()

    // Declare exchange
    ch.ExchangeDeclare("orders", "topic", true, false, false, false, nil)

    // Declare queue
    q, _ := ch.QueueDeclare("order-processor", true, false, false, false, nil)

    // Bind queue to exchange
    ch.QueueBind(q.Name, "orders.#", "orders", false, nil)

    // Publish
    ch.PublishWithContext(context.Background(), "orders", "orders.created",
        false, false,
        amqp.Publishing{
            ContentType:  "application/json",
            Body:         []byte(`{"id": "123"}`),
            DeliveryMode: amqp.Persistent,
        },
    )

    // Consume
    msgs, _ := ch.Consume(q.Name, "", false, false, false, false, nil)

    for msg := range msgs {
        log.Printf("Received: %s", string(msg.Body))
        msg.Ack(false)
    }

    return nil
}
```

**Типичные ошибки.**
1. **Kafka для tasks** — overhead не оправдан
2. **RabbitMQ для event sourcing** — нет replay
3. **NATS Core для critical** — at-most-once потеряет сообщения
4. **Игнорировать operational complexity** — Kafka сложнее в ops

**На интервью.**
Объясни разницу между log-based и queue-based. Покажи, когда replay критичен. Расскажи про NATS JetStream.

*Частые follow-up вопросы:*
- Как мигрировать с одного broker на другой?
- Можно ли использовать несколько brokers?
- Что такое virtual topics в RabbitMQ?

---

### 6. Как обрабатывать Poison Messages?

**Зачем спрашивают.**
Poison messages блокируют обработку. Интервьюер проверяет понимание стратегий: DLQ, retry limits, circuit breaker.

**Короткий ответ.**
Poison message — сообщение, которое всегда вызывает ошибку при обработке. Решения: ограничение retries, Dead Letter Queue для анализа, circuit breaker для предотвращения каскадных сбоев.

**Детальный разбор.**

**Типы poison messages:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Invalid format                                            │
│    Некорректный JSON, отсутствующие поля                    │
│                                                              │
│ 2. Business validation failure                               │
│    Данные не проходят бизнес-правила                        │
│                                                              │
│ 3. Dependency failure                                        │
│    Внешний сервис недоступен (transient)                    │
│                                                              │
│ 4. Bug in handler                                            │
│    Код обработки падает на определённых данных              │
│                                                              │
│ 5. Resource exhaustion                                       │
│    OOM, timeout, rate limit                                 │
└─────────────────────────────────────────────────────────────┘
```

**Стратегия обработки:**
```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   Message ──► Handler                                        │
│                 │                                            │
│                 ├── Success ──► Commit                       │
│                 │                                            │
│                 └── Error                                    │
│                       │                                      │
│                       ├── Retryable?                         │
│                       │     │                                │
│                       │     ├── Yes ──► Retry Queue          │
│                       │     │           (с backoff)          │
│                       │     │                                │
│                       │     └── No ──► DLQ                   │
│                       │                 (для анализа)        │
│                       │                                      │
│                       └── Max retries? ──► DLQ               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Retry с exponential backoff:**
```go
type RetryConfig struct {
    MaxRetries     int
    InitialBackoff time.Duration
    MaxBackoff     time.Duration
    Multiplier     float64
}

type MessageHandler struct {
    handler    func(ctx context.Context, msg Message) error
    retryQueue Queue
    dlq        Queue
    config     RetryConfig
}

func (h *MessageHandler) Handle(ctx context.Context, msg Message) error {
    retries := msg.RetryCount()

    // Проверяем лимит
    if retries >= h.config.MaxRetries {
        return h.sendToDLQ(ctx, msg, "max retries exceeded")
    }

    err := h.handler(ctx, msg)
    if err == nil {
        return nil
    }

    // Классифицируем ошибку
    if isRetryable(err) {
        backoff := h.calculateBackoff(retries)
        return h.scheduleRetry(ctx, msg, backoff)
    }

    // Non-retryable — сразу в DLQ
    return h.sendToDLQ(ctx, msg, err.Error())
}

func (h *MessageHandler) calculateBackoff(retries int) time.Duration {
    backoff := float64(h.config.InitialBackoff) * math.Pow(h.config.Multiplier, float64(retries))
    if backoff > float64(h.config.MaxBackoff) {
        backoff = float64(h.config.MaxBackoff)
    }
    // Jitter
    jitter := rand.Float64() * 0.3 * backoff
    return time.Duration(backoff + jitter)
}

func isRetryable(err error) bool {
    // Network errors, timeouts — retryable
    // Validation errors, business errors — not retryable
    var netErr net.Error
    if errors.As(err, &netErr) {
        return netErr.Temporary()
    }

    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        return false
    }

    return true  // default: retry
}

func (h *MessageHandler) sendToDLQ(ctx context.Context, msg Message, reason string) error {
    dlqMessage := DLQMessage{
        OriginalTopic:   msg.Topic(),
        OriginalPayload: msg.Payload(),
        ErrorReason:     reason,
        Retries:         msg.RetryCount(),
        FailedAt:        time.Now(),
    }

    return h.dlq.Send(ctx, dlqMessage)
}
```

**DLQ с анализом:**
```go
type DLQProcessor struct {
    db       *sql.DB
    alerter  Alerter
}

func (p *DLQProcessor) Process(ctx context.Context, msg DLQMessage) error {
    // Сохраняем для анализа
    _, err := p.db.ExecContext(ctx, `
        INSERT INTO dead_letters (
            original_topic, payload, error_reason, retries, failed_at
        ) VALUES ($1, $2, $3, $4, $5)
    `, msg.OriginalTopic, msg.OriginalPayload, msg.ErrorReason, msg.Retries, msg.FailedAt)
    if err != nil {
        return err
    }

    // Alert при пороге
    count := p.countRecentErrors(ctx, msg.OriginalTopic)
    if count > 10 {
        p.alerter.Alert(fmt.Sprintf("High DLQ rate for topic %s: %d messages", msg.OriginalTopic, count))
    }

    return nil
}

// Replay из DLQ
func (p *DLQProcessor) Replay(ctx context.Context, id string, producer Producer) error {
    var msg DLQMessage
    err := p.db.QueryRowContext(ctx,
        "SELECT original_topic, payload FROM dead_letters WHERE id = $1",
        id,
    ).Scan(&msg.OriginalTopic, &msg.OriginalPayload)
    if err != nil {
        return err
    }

    // Отправляем обратно
    if err := producer.Send(ctx, msg.OriginalTopic, msg.OriginalPayload); err != nil {
        return err
    }

    // Помечаем как replayed
    _, err = p.db.ExecContext(ctx,
        "UPDATE dead_letters SET replayed_at = $1 WHERE id = $2",
        time.Now(), id,
    )
    return err
}
```

**Circuit breaker для handlers:**
```go
type CircuitBreakerHandler struct {
    handler Handler
    cb      *gobreaker.CircuitBreaker
    dlq     Queue
}

func NewCircuitBreakerHandler(handler Handler, dlq Queue) *CircuitBreakerHandler {
    settings := gobreaker.Settings{
        Name:        "message-handler",
        MaxRequests: 5,                // в half-open
        Interval:    10 * time.Second, // reset window
        Timeout:     30 * time.Second, // half-open timeout
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 5
        },
    }

    return &CircuitBreakerHandler{
        handler: handler,
        cb:      gobreaker.NewCircuitBreaker(settings),
        dlq:     dlq,
    }
}

func (h *CircuitBreakerHandler) Handle(ctx context.Context, msg Message) error {
    _, err := h.cb.Execute(func() (interface{}, error) {
        return nil, h.handler.Handle(ctx, msg)
    })

    if err == gobreaker.ErrOpenState {
        // Circuit open — в DLQ для retry позже
        return h.dlq.Send(ctx, msg)
    }

    return err
}
```

**Типичные ошибки.**
1. **Бесконечный retry** — блокирует обработку
2. **Нет backoff** — перегружает систему
3. **DLQ без мониторинга** — проблемы накапливаются
4. **Нет replay механизма** — сообщения теряются навсегда

**На интервью.**
Покажи стратегию retry с backoff. Объясни, когда retry, когда DLQ. Расскажи про replay из DLQ.

*Частые follow-up вопросы:*
- Как определить retryable vs non-retryable?
- Как мониторить DLQ?
- Что делать с очень старыми DLQ сообщениями?

---

### 7. Как реализовать Event Sourcing в Go?

**Зачем спрашивают.**
Event sourcing — мощный паттерн для audit и temporal queries. Интервьюер проверяет понимание концепций и практические аспекты реализации.

**Короткий ответ.**
Event sourcing — хранение состояния как последовательности событий. Текущее состояние восстанавливается replay событий. Даёт полный audit trail и возможность rebuild состояния на любой момент.

**Детальный разбор.**

**Сравнение с CRUD:**
```
┌─────────────────────────────────────────────────────────────┐
│ CRUD (State-based):                                          │
│                                                              │
│   Account { balance: 100 }                                   │
│   UPDATE → Account { balance: 150 }                          │
│   UPDATE → Account { balance: 120 }                          │
│                                                              │
│   Текущее состояние: 120                                    │
│   История: потеряна                                         │
├─────────────────────────────────────────────────────────────┤
│ Event Sourcing:                                              │
│                                                              │
│   Event 1: AccountOpened { initial: 100 }                   │
│   Event 2: MoneyDeposited { amount: 50 }                    │
│   Event 3: MoneyWithdrawn { amount: 30 }                    │
│                                                              │
│   Текущее состояние: replay → 120                           │
│   История: полностью сохранена                              │
│   Любой момент: replay до event N                           │
└─────────────────────────────────────────────────────────────┘
```

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   Command ──► Aggregate ──► Events ──► Event Store          │
│                   │                         │                │
│                   │                         ▼                │
│                   │                    Event Bus ──► Projections
│                   │                                          │
│                   └──────── Rebuild state from events       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Event definitions:**
```go
// Базовый интерфейс события
type Event interface {
    EventType() string
    AggregateID() string
    Version() int
}

// Базовая структура
type BaseEvent struct {
    ID          string    `json:"id"`
    Type        string    `json:"type"`
    AggregateID string    `json:"aggregate_id"`
    Version     int       `json:"version"`
    Timestamp   time.Time `json:"timestamp"`
    Data        []byte    `json:"data"`
}

// Конкретные события
type AccountOpened struct {
    AccountID     string `json:"account_id"`
    OwnerName     string `json:"owner_name"`
    InitialBalance int64  `json:"initial_balance"`
}

type MoneyDeposited struct {
    AccountID string `json:"account_id"`
    Amount    int64  `json:"amount"`
}

type MoneyWithdrawn struct {
    AccountID string `json:"account_id"`
    Amount    int64  `json:"amount"`
}
```

**Aggregate:**
```go
type Account struct {
    id      string
    name    string
    balance int64
    version int
    changes []Event  // uncommitted events
}

func NewAccount(id, name string, initialBalance int64) *Account {
    a := &Account{}
    a.Apply(AccountOpened{AccountID: id, OwnerName: name, InitialBalance: initialBalance})
    return a
}

func (a *Account) Deposit(amount int64) error {
    if amount <= 0 {
        return errors.New("amount must be positive")
    }
    a.Apply(MoneyDeposited{AccountID: a.id, Amount: amount})
    return nil
}

func (a *Account) Withdraw(amount int64) error {
    if amount <= 0 {
        return errors.New("amount must be positive")
    }
    if a.balance < amount {
        return errors.New("insufficient funds")
    }
    a.Apply(MoneyWithdrawn{AccountID: a.id, Amount: amount})
    return nil
}

// Apply применяет событие и добавляет в uncommitted
func (a *Account) Apply(event interface{}) {
    a.when(event)
    a.version++
    a.changes = append(a.changes, event.(Event))
}

// when — чистая функция обновления состояния
func (a *Account) when(event interface{}) {
    switch e := event.(type) {
    case AccountOpened:
        a.id = e.AccountID
        a.name = e.OwnerName
        a.balance = e.InitialBalance
    case MoneyDeposited:
        a.balance += e.Amount
    case MoneyWithdrawn:
        a.balance -= e.Amount
    }
}

// UncommittedChanges возвращает новые события
func (a *Account) UncommittedChanges() []Event {
    return a.changes
}

// ClearChanges после сохранения
func (a *Account) ClearChanges() {
    a.changes = nil
}
```

**Event Store:**
```go
type EventStore interface {
    Save(ctx context.Context, aggregateID string, events []Event, expectedVersion int) error
    Load(ctx context.Context, aggregateID string) ([]Event, error)
}

type PostgresEventStore struct {
    db *sql.DB
}

func (s *PostgresEventStore) Save(ctx context.Context, aggregateID string, events []Event, expectedVersion int) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Optimistic concurrency check
    var currentVersion int
    err = tx.QueryRowContext(ctx,
        "SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1",
        aggregateID,
    ).Scan(&currentVersion)
    if err != nil {
        return err
    }

    if currentVersion != expectedVersion {
        return fmt.Errorf("concurrency conflict: expected %d, got %d", expectedVersion, currentVersion)
    }

    // Insert events
    for i, event := range events {
        data, _ := json.Marshal(event)
        _, err = tx.ExecContext(ctx, `
            INSERT INTO events (aggregate_id, version, event_type, data, created_at)
            VALUES ($1, $2, $3, $4, $5)
        `, aggregateID, expectedVersion+i+1, event.EventType(), data, time.Now())
        if err != nil {
            return err
        }
    }

    return tx.Commit()
}

func (s *PostgresEventStore) Load(ctx context.Context, aggregateID string) ([]Event, error) {
    rows, err := s.db.QueryContext(ctx, `
        SELECT event_type, data FROM events
        WHERE aggregate_id = $1
        ORDER BY version
    `, aggregateID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var events []Event
    for rows.Next() {
        var eventType string
        var data []byte
        if err := rows.Scan(&eventType, &data); err != nil {
            return nil, err
        }

        event, err := deserializeEvent(eventType, data)
        if err != nil {
            return nil, err
        }
        events = append(events, event)
    }

    return events, nil
}
```

**Repository:**
```go
type AccountRepository struct {
    store EventStore
}

func (r *AccountRepository) Save(ctx context.Context, account *Account) error {
    changes := account.UncommittedChanges()
    if len(changes) == 0 {
        return nil
    }

    err := r.store.Save(ctx, account.id, changes, account.version-len(changes))
    if err != nil {
        return err
    }

    account.ClearChanges()
    return nil
}

func (r *AccountRepository) Load(ctx context.Context, id string) (*Account, error) {
    events, err := r.store.Load(ctx, id)
    if err != nil {
        return nil, err
    }

    if len(events) == 0 {
        return nil, ErrNotFound
    }

    account := &Account{}
    for _, event := range events {
        account.when(event)
        account.version++
    }

    return account, nil
}
```

**Projection:**
```go
type AccountBalanceProjection struct {
    db *sql.DB
}

func (p *AccountBalanceProjection) Handle(event Event) error {
    switch e := event.(type) {
    case AccountOpened:
        _, err := p.db.Exec(`
            INSERT INTO account_balances (account_id, balance)
            VALUES ($1, $2)
        `, e.AccountID, e.InitialBalance)
        return err

    case MoneyDeposited:
        _, err := p.db.Exec(`
            UPDATE account_balances SET balance = balance + $1 WHERE account_id = $2
        `, e.Amount, e.AccountID)
        return err

    case MoneyWithdrawn:
        _, err := p.db.Exec(`
            UPDATE account_balances SET balance = balance - $1 WHERE account_id = $2
        `, e.Amount, e.AccountID)
        return err
    }

    return nil
}
```

**Типичные ошибки.**
1. **Мутации без событий** — нарушает event sourcing
2. **Большие агрегаты** — медленный replay
3. **Нет snapshots** — долгая загрузка старых агрегатов
4. **Events с логикой** — events должны быть immutable data

**На интервью.**
Покажи Apply/when разделение. Объясни optimistic concurrency. Расскажи про projections и CQRS.

*Частые follow-up вопросы:*
- Как делать миграции схемы событий?
- Когда нужны snapshots?
- Как масштабировать event store?

---

## Практика

1. **Kafka:** Реализуй producer/consumer с manual commit. Добавь graceful shutdown.

2. **Consumer Groups:** Настрой static membership. Измерь время rebalance.

3. **Exactly-Once:** Реализуй idempotent processing с deduplication в Redis.

4. **Outbox:** Реализуй outbox pattern с relay. Добавь cleanup job.

5. **DLQ:** Добавь retry с backoff и DLQ. Реализуй replay механизм.

6. **Event Sourcing:** Реализуй простой aggregate с event store и projection.

---

## Дополнительные материалы

- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)
- [sarama](https://github.com/IBM/sarama)
- [NATS documentation](https://docs.nats.io/)
- [RabbitMQ tutorials](https://www.rabbitmq.com/getstarted.html)
- [Event Sourcing in Practice](https://eventstore.com/event-sourcing/)
- [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- Talks: "Event Sourcing You are doing it wrong" — David Schmitz
