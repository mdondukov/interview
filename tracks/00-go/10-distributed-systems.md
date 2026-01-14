# 10 — Distributed Systems

Разбор вопросов о распределённых системах: консенсус, идемпотентность, saga, трейсинг, graceful degradation и обработка partial failures.

**Навигация:** [← 09-performance-profiling](./09-performance-profiling.md) | [Трек Go](./README.md) | [11-message-queues-events →](./11-message-queues-events.md)

---

## Вопросы и разборы

### 1. Как реализовать распределённую блокировку?

**Зачем спрашивают.**
Распределённые блокировки — базовый примитив координации. Интервьюер проверяет понимание проблем (split-brain, fencing tokens) и знание реализаций.

**Короткий ответ.**
Распределённая блокировка координирует доступ между процессами на разных машинах. Реализации: Redis (Redlock), etcd, Consul, ZooKeeper. Важны: TTL, fencing tokens, clock drift.

**Детальный разбор.**

**Проблемы распределённых блокировок:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Split-brain                                               │
│    Два процесса считают, что владеют блокировкой            │
│                                                              │
│ 2. Clock drift                                               │
│    Время на разных серверах расходится                      │
│                                                              │
│ 3. Process pause (GC, swap)                                  │
│    Процесс "заснул", блокировка истекла, другой взял        │
│                                                              │
│ 4. Network partition                                         │
│    Клиент не может связаться с lock service                 │
└─────────────────────────────────────────────────────────────┘
```

**Fencing tokens:**
```
┌─────────────────────────────────────────────────────────────┐
│ Client A                Storage                Client B      │
│     │                      │                      │          │
│     │──lock(token=33)─────►│                      │          │
│     │◄─────────OK──────────│                      │          │
│     │                      │                      │          │
│     │  [GC pause...]       │                      │          │
│     │                      │◄──lock(token=34)─────│          │
│     │                      │───────OK────────────►│          │
│     │                      │◄──write(token=34)────│          │
│     │                      │───────OK────────────►│          │
│     │                      │                      │          │
│     │──write(token=33)────►│                      │          │
│     │◄────REJECTED─────────│ (33 < 34)            │          │
│                                                              │
│ Storage отклоняет старые токены                             │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Redis (простая блокировка):**
```go
import "github.com/redis/go-redis/v9"

type RedisLock struct {
    client *redis.Client
    key    string
    value  string  // уникальный идентификатор владельца
    ttl    time.Duration
}

func NewRedisLock(client *redis.Client, key string, ttl time.Duration) *RedisLock {
    return &RedisLock{
        client: client,
        key:    "lock:" + key,
        value:  uuid.New().String(),
        ttl:    ttl,
    }
}

func (l *RedisLock) Lock(ctx context.Context) error {
    // SET NX — только если ключ не существует
    ok, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
    if err != nil {
        return err
    }
    if !ok {
        return ErrLockNotAcquired
    }
    return nil
}

func (l *RedisLock) Unlock(ctx context.Context) error {
    // Lua скрипт: удалить только если значение совпадает
    script := redis.NewScript(`
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `)

    result, err := script.Run(ctx, l.client, []string{l.key}, l.value).Int()
    if err != nil {
        return err
    }
    if result == 0 {
        return ErrLockNotOwned
    }
    return nil
}

// С retry
func (l *RedisLock) LockWithRetry(ctx context.Context, attempts int, delay time.Duration) error {
    for i := 0; i < attempts; i++ {
        if err := l.Lock(ctx); err == nil {
            return nil
        }

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(delay):
        }
    }
    return ErrLockNotAcquired
}
```

**etcd (с lease для автопродления):**
```go
import clientv3 "go.etcd.io/etcd/client/v3"

func AcquireLock(ctx context.Context, client *clientv3.Client, key string) (func(), error) {
    // Создаём lease с TTL
    lease, err := client.Grant(ctx, 10)  // 10 секунд
    if err != nil {
        return nil, err
    }

    // KeepAlive продлевает lease автоматически
    keepAliveCh, err := client.KeepAlive(ctx, lease.ID)
    if err != nil {
        return nil, err
    }

    // Пытаемся создать ключ с lease
    txn := client.Txn(ctx)
    txn.If(clientv3.Compare(clientv3.CreateRevision(key), "=", 0)).
        Then(clientv3.OpPut(key, "", clientv3.WithLease(lease.ID))).
        Else(clientv3.OpGet(key))

    resp, err := txn.Commit()
    if err != nil {
        return nil, err
    }

    if !resp.Succeeded {
        return nil, ErrLockNotAcquired
    }

    // Функция для освобождения
    unlock := func() {
        client.Delete(context.Background(), key)
        client.Revoke(context.Background(), lease.ID)
    }

    // Горутина для обработки KeepAlive
    go func() {
        for range keepAliveCh {
            // lease продлевается
        }
    }()

    return unlock, nil
}
```

**Redlock (несколько Redis инстансов):**
```go
import "github.com/go-redsync/redsync/v4"

func UseRedlock(pools []redsync.Pool) {
    rs := redsync.New(pools...)

    mutex := rs.NewMutex("resource-key",
        redsync.WithExpiry(10*time.Second),
        redsync.WithTries(3),
    )

    if err := mutex.Lock(); err != nil {
        log.Fatal(err)
    }
    defer mutex.Unlock()

    // критическая секция
}
```

**Типичные ошибки.**
1. **Нет TTL** — блокировка зависает навсегда при падении
2. **DEL без проверки владельца** — можно удалить чужую блокировку
3. **Не учитывать clock drift** — Redlock чувствителен к этому
4. **Бесконечный retry** — лучше fail fast с ошибкой

**На интервью.**
Объясни проблему split-brain и решение через fencing tokens. Покажи разницу между Redis и etcd блокировками. Расскажи, когда Redlock нужен.

*Частые follow-up вопросы:*
- Когда использовать advisory locks в PostgreSQL?
- Как обрабатывать истечение блокировки во время работы?
- Почему Martin Kleppmann критикует Redlock?

---

### 2. Что такое консенсус и как работает Raft?

**Зачем спрашивают.**
Консенсус — фундамент распределённых систем (etcd, Consul, CockroachDB). Интервьюер проверяет понимание теории и практики.

**Короткий ответ.**
Консенсус — согласование значения между узлами несмотря на отказы. Raft — понятный алгоритм консенсуса с leader election и log replication. Используется в etcd, Consul, TiKV.

**Детальный разбор.**

**Роли в Raft:**
```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│     ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│     │ Follower │    │  Leader  │    │ Follower │            │
│     └────┬─────┘    └────┬─────┘    └────┬─────┘            │
│          │               │               │                   │
│          │◄──heartbeat───│───heartbeat──►│                   │
│          │               │               │                   │
│          │◄──replicate───│───replicate──►│                   │
│          │               │               │                   │
│                                                              │
│  • Leader: принимает запросы, реплицирует лог               │
│  • Follower: реплики, голосуют за leader                    │
│  • Candidate: претендент на leader при выборах              │
└─────────────────────────────────────────────────────────────┘
```

**Leader Election:**
```
┌─────────────────────────────────────────────────────────────┐
│ Term 1                                                       │
│                                                              │
│   Follower ──[timeout]──► Candidate ──[votes]──► Leader     │
│                               │                              │
│                               ▼                              │
│                        RequestVote RPC                       │
│                        к другим узлам                        │
│                                                              │
│ Побеждает если получил большинство голосов (N/2 + 1)        │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│ Term 2 (новый leader)                                        │
│                                                              │
│   При сбое leader → timeout → новые выборы                  │
└─────────────────────────────────────────────────────────────┘
```

**Log Replication:**
```
┌─────────────────────────────────────────────────────────────┐
│ Client ──► Leader: "set x=1"                                 │
│                                                              │
│ 1. Leader добавляет в свой лог                              │
│ 2. AppendEntries RPC к followers                            │
│ 3. Followers добавляют в лог, отвечают OK                   │
│ 4. Когда большинство ответило → entry committed             │
│ 5. Leader применяет к state machine                         │
│ 6. Отвечает клиенту                                         │
│                                                              │
│ Log:                                                         │
│ ┌─────┬─────┬─────┬─────┬─────┐                              │
│ │ 1   │ 2   │ 3   │ 4   │ 5   │  ← index                    │
│ │ T1  │ T1  │ T2  │ T2  │ T2  │  ← term                     │
│ │ x=1 │ y=2 │ z=3 │ x=4 │ y=5 │  ← command                  │
│ └─────┴─────┴─────┴─────┴─────┘                              │
│               ▲                                              │
│          committed                                           │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Использование etcd (Raft под капотом):**
```go
import clientv3 "go.etcd.io/etcd/client/v3"

func UseEtcd(ctx context.Context) error {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379", "localhost:2380", "localhost:2381"},
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        return err
    }
    defer client.Close()

    // Put — реплицируется на большинство узлов
    _, err = client.Put(ctx, "key", "value")
    if err != nil {
        return err
    }

    // Get — читает закоммиченное значение
    resp, err := client.Get(ctx, "key")
    if err != nil {
        return err
    }

    // Watch — подписка на изменения
    watchCh := client.Watch(ctx, "key")
    for wresp := range watchCh {
        for _, ev := range wresp.Events {
            fmt.Printf("%s %q : %q\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
        }
    }

    return nil
}
```

**Hashicorp Raft (встраиваемая библиотека):**
```go
import "github.com/hashicorp/raft"

// FSM — finite state machine, применяет команды
type MyFSM struct {
    mu   sync.Mutex
    data map[string]string
}

func (f *MyFSM) Apply(log *raft.Log) interface{} {
    f.mu.Lock()
    defer f.mu.Unlock()

    // Парсим команду и применяем
    cmd := parseCommand(log.Data)
    f.data[cmd.Key] = cmd.Value
    return nil
}

func (f *MyFSM) Snapshot() (raft.FSMSnapshot, error) {
    // Создаём snapshot для recovery
    f.mu.Lock()
    defer f.mu.Unlock()

    dataCopy := make(map[string]string)
    for k, v := range f.data {
        dataCopy[k] = v
    }
    return &MySnapshot{data: dataCopy}, nil
}

func (f *MyFSM) Restore(rc io.ReadCloser) error {
    // Восстанавливаем из snapshot
    // ...
}

// Создание Raft node
func NewRaftNode(nodeID string, fsm raft.FSM) (*raft.Raft, error) {
    config := raft.DefaultConfig()
    config.LocalID = raft.ServerID(nodeID)

    // Transport для коммуникации между узлами
    addr, _ := net.ResolveTCPAddr("tcp", "127.0.0.1:7000")
    transport, _ := raft.NewTCPTransport(addr.String(), addr, 3, 10*time.Second, os.Stderr)

    // Хранилище для логов и stable store
    logStore := raft.NewInmemStore()
    stableStore := raft.NewInmemStore()
    snapshotStore := raft.NewInmemSnapshotStore()

    return raft.NewRaft(config, fsm, logStore, stableStore, snapshotStore, transport)
}
```

**Типичные ошибки.**
1. **Чтение без кворума** — stale reads
2. **Игнорировать partition** — split-brain при неправильной конфигурации
3. **Слишком маленький election timeout** — частые выборы
4. **Нечётное число узлов** — нет преимущества чётного

**На интервью.**
Объясни термины: term, leader election, log replication. Покажи, почему нужно N/2+1 узлов для кворума. Расскажи разницу между etcd и Consul.

*Частые follow-up вопросы:*
- Что происходит при network partition?
- Как Raft обрабатывает split-brain?
- Зачем нужны snapshots?

---

### 3. Как реализовать идемпотентность операций?

**Зачем спрашивают.**
Идемпотентность критична для надёжности (retries, at-least-once delivery). Интервьюер проверяет понимание паттернов и практик.

**Короткий ответ.**
Идемпотентная операция при повторном выполнении даёт тот же результат. Реализация: idempotency key в запросе, проверка в БД перед выполнением, возврат сохранённого результата.

**Детальный разбор.**

**Проблема без идемпотентности:**
```
┌─────────────────────────────────────────────────────────────┐
│ Client ────────► Server: "transfer $100"                     │
│                     │                                        │
│         [timeout]   │ (успешно выполнено)                   │
│                     │                                        │
│ Client ────────► Server: "transfer $100" (retry)            │
│                     │                                        │
│              $200 списано!                                   │
└─────────────────────────────────────────────────────────────┘
```

**Решение с idempotency key:**
```
┌─────────────────────────────────────────────────────────────┐
│ Client ────────► Server: "transfer $100, key=abc123"        │
│                     │                                        │
│                     ├─► Проверить: key=abc123 есть в БД?    │
│                     │   Нет → выполнить, сохранить результат│
│                     │                                        │
│         [timeout]   │                                        │
│                     │                                        │
│ Client ────────► Server: "transfer $100, key=abc123" (retry)│
│                     │                                        │
│                     ├─► Проверить: key=abc123 есть в БД?    │
│                     │   Да → вернуть сохранённый результат  │
│                                                              │
│              $100 списано (корректно)                        │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Таблица для idempotency:**
```sql
CREATE TABLE idempotency_keys (
    key VARCHAR(255) PRIMARY KEY,
    request_hash VARCHAR(64) NOT NULL,  -- hash тела запроса
    response_code INT NOT NULL,
    response_body JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
```

**Middleware для идемпотентности:**
```go
type IdempotencyMiddleware struct {
    db    *sql.DB
    ttl   time.Duration
}

func (m *IdempotencyMiddleware) Handle(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Только для мутирующих методов
        if r.Method == http.MethodGet || r.Method == http.MethodHead {
            next.ServeHTTP(w, r)
            return
        }

        key := r.Header.Get("Idempotency-Key")
        if key == "" {
            next.ServeHTTP(w, r)
            return
        }

        // Читаем body для hash
        body, _ := io.ReadAll(r.Body)
        r.Body = io.NopCloser(bytes.NewReader(body))
        requestHash := sha256Hash(body)

        // Проверяем существующий ключ
        existing, err := m.getIdempotencyRecord(r.Context(), key)
        if err == nil {
            // Проверяем, что запрос тот же
            if existing.RequestHash != requestHash {
                http.Error(w, "Idempotency key reused with different request", http.StatusUnprocessableEntity)
                return
            }

            // Возвращаем сохранённый ответ
            w.WriteHeader(existing.ResponseCode)
            w.Write(existing.ResponseBody)
            return
        }

        // Захватываем response
        rec := &responseRecorder{ResponseWriter: w}
        next.ServeHTTP(rec, r)

        // Сохраняем результат
        m.saveIdempotencyRecord(r.Context(), key, requestHash, rec.statusCode, rec.body.Bytes())
    })
}

type responseRecorder struct {
    http.ResponseWriter
    statusCode int
    body       bytes.Buffer
}

func (r *responseRecorder) WriteHeader(code int) {
    r.statusCode = code
    r.ResponseWriter.WriteHeader(code)
}

func (r *responseRecorder) Write(b []byte) (int, error) {
    r.body.Write(b)
    return r.ResponseWriter.Write(b)
}
```

**Идемпотентность на уровне БД:**
```go
func TransferMoney(ctx context.Context, db *sql.DB, req TransferRequest) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Проверяем, не выполнен ли уже этот transfer
    var exists bool
    err = tx.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM transfers WHERE idempotency_key = $1)",
        req.IdempotencyKey,
    ).Scan(&exists)
    if err != nil {
        return err
    }
    if exists {
        return nil  // уже выполнено
    }

    // Выполняем transfer
    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        req.Amount, req.FromAccount,
    )
    if err != nil {
        return err
    }

    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        req.Amount, req.ToAccount,
    )
    if err != nil {
        return err
    }

    // Сохраняем запись о выполнении
    _, err = tx.ExecContext(ctx,
        "INSERT INTO transfers (idempotency_key, from_account, to_account, amount) VALUES ($1, $2, $3, $4)",
        req.IdempotencyKey, req.FromAccount, req.ToAccount, req.Amount,
    )
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

**Типичные ошибки.**
1. **Key без TTL** — таблица растёт бесконечно
2. **Key без привязки к user** — один user может переиспользовать key другого
3. **Не проверять request hash** — разные запросы с одним key
4. **Не сохранять результат** — повторный запрос выполняется заново

**На интервью.**
Покажи структуру таблицы idempotency. Объясни, почему нужен request hash. Расскажи про TTL и cleanup.

*Частые follow-up вопросы:*
- Как генерировать idempotency key на клиенте?
- Как обрабатывать in-flight запросы с тем же key?
- Когда идемпотентность не нужна?

---

### 4. Что такое Saga pattern и как его реализовать?

**Зачем спрашивают.**
Saga — альтернатива распределённым транзакциям. Интервьюер проверяет понимание eventual consistency и компенсирующих транзакций.

**Короткий ответ.**
Saga — последовательность локальных транзакций с компенсирующими действиями при ошибке. Два подхода: Choreography (события) и Orchestration (центральный координатор).

**Детальный разбор.**

**Сравнение подходов:**
```
┌─────────────────────────────────────────────────────────────┐
│ Choreography (события)                                       │
│                                                              │
│ Service A ──event──► Service B ──event──► Service C          │
│     │                   │                   │                │
│     ◄──────────────compensate──────────────                 │
│                                                              │
│ + Loose coupling                                             │
│ - Сложно отслеживать flow                                   │
├─────────────────────────────────────────────────────────────┤
│ Orchestration (координатор)                                  │
│                                                              │
│              ┌──────────────┐                                │
│              │ Orchestrator │                                │
│              └──────┬───────┘                                │
│         ┌───────────┼───────────┐                            │
│         ▼           ▼           ▼                            │
│    Service A   Service B   Service C                         │
│                                                              │
│ + Понятный flow                                              │
│ - Single point of failure                                    │
└─────────────────────────────────────────────────────────────┘
```

**Пример Saga: оформление заказа:**
```
┌─────────────────────────────────────────────────────────────┐
│ Saga: CreateOrder                                            │
│                                                              │
│ Шаги:                          Компенсация:                  │
│ 1. CreateOrder                 → CancelOrder                 │
│ 2. ReserveInventory            → ReleaseInventory            │
│ 3. ProcessPayment              → RefundPayment               │
│ 4. ShipOrder                   → CancelShipment              │
│                                                              │
│ При ошибке на шаге 3:                                        │
│ 1. RefundPayment (если частично)                             │
│ 2. ReleaseInventory                                          │
│ 3. CancelOrder                                               │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Orchestrator-based Saga:**
```go
type OrderSaga struct {
    orderService     OrderService
    inventoryService InventoryService
    paymentService   PaymentService
    shippingService  ShippingService
}

type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context, data *SagaData) error
    Compensate func(ctx context.Context, data *SagaData) error
}

type SagaData struct {
    OrderID     string
    UserID      string
    Items       []OrderItem
    PaymentID   string
    ShipmentID  string
}

func (s *OrderSaga) Execute(ctx context.Context, data *SagaData) error {
    steps := []SagaStep{
        {
            Name: "CreateOrder",
            Execute: func(ctx context.Context, d *SagaData) error {
                order, err := s.orderService.Create(ctx, d.UserID, d.Items)
                if err != nil {
                    return err
                }
                d.OrderID = order.ID
                return nil
            },
            Compensate: func(ctx context.Context, d *SagaData) error {
                return s.orderService.Cancel(ctx, d.OrderID)
            },
        },
        {
            Name: "ReserveInventory",
            Execute: func(ctx context.Context, d *SagaData) error {
                return s.inventoryService.Reserve(ctx, d.OrderID, d.Items)
            },
            Compensate: func(ctx context.Context, d *SagaData) error {
                return s.inventoryService.Release(ctx, d.OrderID)
            },
        },
        {
            Name: "ProcessPayment",
            Execute: func(ctx context.Context, d *SagaData) error {
                payment, err := s.paymentService.Process(ctx, d.OrderID, d.UserID)
                if err != nil {
                    return err
                }
                d.PaymentID = payment.ID
                return nil
            },
            Compensate: func(ctx context.Context, d *SagaData) error {
                return s.paymentService.Refund(ctx, d.PaymentID)
            },
        },
        {
            Name: "CreateShipment",
            Execute: func(ctx context.Context, d *SagaData) error {
                shipment, err := s.shippingService.Create(ctx, d.OrderID)
                if err != nil {
                    return err
                }
                d.ShipmentID = shipment.ID
                return nil
            },
            Compensate: func(ctx context.Context, d *SagaData) error {
                return s.shippingService.Cancel(ctx, d.ShipmentID)
            },
        },
    }

    var completedSteps []SagaStep

    for _, step := range steps {
        if err := step.Execute(ctx, data); err != nil {
            // Компенсируем в обратном порядке
            for i := len(completedSteps) - 1; i >= 0; i-- {
                if compErr := completedSteps[i].Compensate(ctx, data); compErr != nil {
                    // Логируем, возможно retry позже
                    log.Printf("compensation failed for %s: %v", completedSteps[i].Name, compErr)
                }
            }
            return fmt.Errorf("saga failed at step %s: %w", step.Name, err)
        }
        completedSteps = append(completedSteps, step)
    }

    return nil
}
```

**Choreography-based Saga с событиями:**
```go
// Order Service
func (s *OrderService) HandleOrderCreated(ctx context.Context, event OrderCreatedEvent) error {
    // Публикуем событие для Inventory
    return s.publisher.Publish(ctx, "inventory.reserve", InventoryReserveEvent{
        OrderID: event.OrderID,
        Items:   event.Items,
    })
}

// Inventory Service
func (s *InventoryService) HandleReserveRequest(ctx context.Context, event InventoryReserveEvent) error {
    if err := s.reserve(ctx, event.OrderID, event.Items); err != nil {
        // Публикуем событие об ошибке
        return s.publisher.Publish(ctx, "order.cancel", OrderCancelEvent{
            OrderID: event.OrderID,
            Reason:  "inventory_unavailable",
        })
    }

    // Успех — публикуем для Payment
    return s.publisher.Publish(ctx, "payment.process", PaymentProcessEvent{
        OrderID: event.OrderID,
    })
}

// При ошибке Payment — Inventory слушает событие
func (s *InventoryService) HandlePaymentFailed(ctx context.Context, event PaymentFailedEvent) error {
    return s.release(ctx, event.OrderID)
}
```

**Состояние Saga в БД:**
```sql
CREATE TABLE sagas (
    id UUID PRIMARY KEY,
    type VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,  -- STARTED, COMPLETED, COMPENSATING, FAILED
    current_step INT NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE saga_steps (
    id UUID PRIMARY KEY,
    saga_id UUID REFERENCES sagas(id),
    step_index INT NOT NULL,
    name VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,  -- PENDING, COMPLETED, COMPENSATED, FAILED
    result JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**Типичные ошибки.**
1. **Компенсация без retry** — может не выполниться
2. **Неидемпотентные шаги** — проблемы при retry
3. **Нет персистенции состояния** — при рестарте saga теряется
4. **Синхронный orchestrator** — блокирует при долгих операциях

**На интервью.**
Объясни разницу между Choreography и Orchestration. Покажи пример компенсирующей транзакции. Расскажи про eventual consistency.

*Частые follow-up вопросы:*
- Как обрабатывать ошибку в компенсации?
- Когда использовать 2PC вместо Saga?
- Как тестировать Saga?

---

### 5. Как работает распределённый трейсинг?

**Зачем спрашивают.**
Трейсинг — ключ к observability в микросервисах. Интервьюер проверяет понимание spans, propagation и инструментов.

**Короткий ответ.**
Распределённый трейсинг связывает запросы между сервисами через trace ID. Каждый сервис создаёт span с временными метками. OpenTelemetry — стандарт для инструментирования.

**Детальный разбор.**

**Структура trace:**
```
┌─────────────────────────────────────────────────────────────┐
│ Trace ID: abc123                                             │
│                                                              │
│ ├── Span: API Gateway (10ms)                                │
│ │   └── Span: User Service (5ms)                            │
│ │       └── Span: DB Query (2ms)                            │
│ │                                                            │
│ └── Span: Order Service (8ms)                               │
│     ├── Span: Inventory Check (3ms)                         │
│     └── Span: Payment Service (4ms)                         │
│         └── Span: External API (2ms)                        │
│                                                              │
│ Timeline:                                                    │
│ 0ms  2ms  4ms  6ms  8ms  10ms                               │
│ |====API Gateway=================|                          │
│   |===User===|                                              │
│     |DB|                                                    │
│       |=====Order=====|                                     │
│         |Inv|                                               │
│           |==Payment==|                                     │
│             |Ext|                                           │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**OpenTelemetry setup:**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

func initTracer(ctx context.Context, serviceName string) (func(), error) {
    // Exporter для отправки в collector
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    // Resource с метаданными сервиса
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion("1.0.0"),
        ),
    )
    if err != nil {
        return nil, err
    }

    // TracerProvider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.AlwaysSample()),  // или ParentBased
    )

    otel.SetTracerProvider(tp)

    // Shutdown function
    return func() {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        tp.Shutdown(ctx)
    }, nil
}
```

**Создание spans:**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("myservice")

func ProcessOrder(ctx context.Context, orderID string) error {
    // Создаём span
    ctx, span := tracer.Start(ctx, "ProcessOrder",
        trace.WithAttributes(
            attribute.String("order.id", orderID),
        ),
    )
    defer span.End()

    // Добавляем события
    span.AddEvent("Validation started")

    if err := validateOrder(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    span.AddEvent("Validation completed")

    // Вложенный span
    if err := processPayment(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    span.SetStatus(codes.Ok, "")
    return nil
}

func processPayment(ctx context.Context, orderID string) error {
    ctx, span := tracer.Start(ctx, "ProcessPayment")
    defer span.End()

    // HTTP вызов к другому сервису
    // ...
    return nil
}
```

**HTTP propagation:**
```go
import (
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/otel/propagation"
)

// Клиент с propagation
func NewHTTPClient() *http.Client {
    return &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
}

// Сервер с извлечением trace context
func NewHTTPHandler(handler http.Handler) http.Handler {
    return otelhttp.NewHandler(handler, "http-server")
}

// Ручное извлечение (если нужно)
func extractTraceContext(r *http.Request) context.Context {
    propagator := propagation.TraceContext{}
    return propagator.Extract(r.Context(), propagation.HeaderCarrier(r.Header))
}
```

**gRPC instrumentation:**
```go
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

// Сервер
server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

// Клиент
conn, err := grpc.Dial(address,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
)
```

**Sampling strategies:**
```go
// Всегда сэмплировать
sampler := sdktrace.AlwaysSample()

// Никогда
sampler := sdktrace.NeverSample()

// Вероятностный (10%)
sampler := sdktrace.TraceIDRatioBased(0.1)

// Parent-based (наследует решение родителя)
sampler := sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))
```

**Типичные ошибки.**
1. **Нет propagation** — traces обрываются между сервисами
2. **Слишком много spans** — overhead и шум
3. **Нет sampling** — огромный объём данных
4. **Не передавать ctx** — span не связывается с parent

**На интервью.**
Объясни структуру trace/span. Покажи, как context передаёт trace ID. Расскажи про sampling стратегии.

*Частые follow-up вопросы:*
- Как интегрировать с существующим logging?
- Что такое baggage?
- Как трейсить асинхронные операции?

---

### 6. Что такое Service Mesh и как Go-сервисы с ним работают?

**Зачем спрашивают.**
Service Mesh — современный подход к networking в Kubernetes. Интервьюер проверяет понимание архитектуры и влияния на код.

**Короткий ответ.**
Service Mesh (Istio, Linkerd) — инфраструктурный слой для service-to-service коммуникации. Sidecar proxy обрабатывает networking: mTLS, load balancing, retries, observability. Код приложения не меняется.

**Детальный разбор.**

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────┐
│ Pod A                           Pod B                        │
│ ┌─────────────┐                 ┌─────────────┐             │
│ │   App A     │                 │   App B     │             │
│ │  (Go code)  │                 │  (Go code)  │             │
│ └──────┬──────┘                 └──────┬──────┘             │
│        │                               │                     │
│        ▼                               ▼                     │
│ ┌─────────────┐                 ┌─────────────┐             │
│ │   Envoy     │ ◄──────────────►│   Envoy     │             │
│ │  (sidecar)  │     mTLS        │  (sidecar)  │             │
│ └─────────────┘                 └─────────────┘             │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                     Control Plane                            │
│              ┌─────────────────────────┐                    │
│              │   Istio / Linkerd       │                    │
│              │   (config, certs, etc)  │                    │
│              └─────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

**Что делает Service Mesh:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ Функция         │ Описание                                 │
├─────────────────┼──────────────────────────────────────────┤
│ mTLS            │ Автоматическое шифрование между сервисами│
│ Load Balancing  │ Умная балансировка (least connections)   │
│ Retries         │ Автоматические retry с exponential backoff│
│ Circuit Breaker │ Защита от каскадных сбоев               │
│ Rate Limiting   │ Ограничение запросов                     │
│ Observability   │ Метрики, трейсинг, access logs          │
│ Traffic Split   │ Canary, A/B testing                      │
│ Authorization   │ Policy-based доступ                      │
└─────────────────┴──────────────────────────────────────────┘
```

**Пример.**

**Go-код без изменений:**
```go
// Service mesh прозрачен для приложения
func CallUserService(ctx context.Context, userID string) (*User, error) {
    // Обычный HTTP вызов
    // Envoy sidecar перехватит и добавит:
    // - mTLS
    // - Retry logic
    // - Tracing headers
    // - Metrics

    resp, err := http.Get(fmt.Sprintf("http://user-service/users/%s", userID))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }

    return &user, nil
}
```

**Istio VirtualService для traffic splitting:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10  # 10% трафика на v2
```

**DestinationRule для circuit breaker:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

**Использование headers для context:**
```go
// Service mesh добавляет headers для tracing
// Приложение должно их пропагировать

func Handler(w http.ResponseWriter, r *http.Request) {
    // Извлекаем trace headers
    traceHeaders := []string{
        "x-request-id",
        "x-b3-traceid",
        "x-b3-spanid",
        "x-b3-parentspanid",
        "x-b3-sampled",
        "x-b3-flags",
    }

    // Пропагируем в исходящие запросы
    req, _ := http.NewRequest("GET", "http://other-service/...", nil)
    for _, h := range traceHeaders {
        if v := r.Header.Get(h); v != "" {
            req.Header.Set(h, v)
        }
    }

    // Или используем OpenTelemetry propagation (автоматически)
}
```

**Health checks для service mesh:**
```go
// Readiness — готов ли принимать трафик
http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    if isReady() {
        w.WriteHeader(http.StatusOK)
        return
    }
    w.WriteHeader(http.StatusServiceUnavailable)
})

// Liveness — жив ли процесс
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
})
```

**Типичные ошибки.**
1. **Не пропагировать headers** — трейсинг обрывается
2. **Свой retry + mesh retry** — двойной retry
3. **Таймауты не согласованы** — сервис timeout < mesh retry timeout
4. **Игнорировать sidecar overhead** — дополнительная latency

**На интервью.**
Объясни, что делает sidecar proxy. Покажи, как код не меняется при использовании mesh. Расскажи про trade-offs (latency, complexity).

*Частые follow-up вопросы:*
- Когда service mesh не нужен?
- Как отлаживать проблемы в mesh?
- Istio vs Linkerd — когда что?

---

### 7. Как реализовать Graceful Degradation?

**Зачем спрашивают.**
Graceful degradation — ключ к resilience. Интервьюер проверяет понимание fallbacks, feature flags и приоритизации.

**Короткий ответ.**
Graceful degradation — продолжение работы с ограниченной функциональностью при сбоях зависимостей. Реализация: fallbacks, cached responses, feature flags, приоритизация критичных функций.

**Детальный разбор.**

**Стратегии degradation:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Fallback to cache                                         │
│    Recommendations недоступны → показать популярные          │
│                                                              │
│ 2. Default values                                            │
│    User preferences недоступны → использовать defaults       │
│                                                              │
│ 3. Feature disable                                           │
│    Search недоступен → скрыть поиск, показать каталог       │
│                                                              │
│ 4. Reduced functionality                                     │
│    Full checkout → simplified checkout                       │
│                                                              │
│ 5. Load shedding                                             │
│    Перегрузка → отклонять non-critical запросы              │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Fallback с cache:**
```go
type ProductService struct {
    client      ProductClient
    cache       Cache
    circuitBreaker *CircuitBreaker
}

func (s *ProductService) GetProduct(ctx context.Context, id string) (*Product, error) {
    // Пытаемся получить из основного сервиса
    product, err := s.circuitBreaker.Execute(func() (interface{}, error) {
        return s.client.GetProduct(ctx, id)
    })

    if err == nil {
        p := product.(*Product)
        // Обновляем cache
        s.cache.Set(ctx, "product:"+id, p, time.Hour)
        return p, nil
    }

    // Fallback to cache
    cached, cacheErr := s.cache.Get(ctx, "product:"+id)
    if cacheErr == nil {
        return cached.(*Product), nil
    }

    // Оба источника недоступны
    return nil, fmt.Errorf("product service unavailable: %w", err)
}
```

**Feature flags для degradation:**
```go
type FeatureFlags struct {
    mu    sync.RWMutex
    flags map[string]bool
}

func (f *FeatureFlags) IsEnabled(feature string) bool {
    f.mu.RLock()
    defer f.mu.RUnlock()
    enabled, ok := f.flags[feature]
    return ok && enabled
}

// Health monitor обновляет flags
func (f *FeatureFlags) MonitorHealth(ctx context.Context, services []Service) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            for _, svc := range services {
                healthy := svc.HealthCheck(ctx)
                f.SetFlag(svc.FeatureFlag(), healthy)
            }
        }
    }
}

// Использование
func (h *Handler) HandleRequest(w http.ResponseWriter, r *http.Request) {
    resp := Response{
        Products: h.getProducts(r.Context()),
    }

    // Рекомендации только если сервис доступен
    if h.features.IsEnabled("recommendations") {
        resp.Recommendations = h.getRecommendations(r.Context())
    }

    // Персонализация только если сервис доступен
    if h.features.IsEnabled("personalization") {
        resp.PersonalizedContent = h.getPersonalized(r.Context())
    }

    json.NewEncoder(w).Encode(resp)
}
```

**Load shedding:**
```go
type LoadShedder struct {
    maxConcurrent int64
    current       atomic.Int64
    critical      map[string]bool  // критичные endpoints
}

func (l *LoadShedder) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Критичные запросы всегда пропускаем
        if l.critical[r.URL.Path] {
            next.ServeHTTP(w, r)
            return
        }

        // Проверяем нагрузку
        current := l.current.Add(1)
        defer l.current.Add(-1)

        if current > l.maxConcurrent {
            // Отклоняем non-critical
            w.Header().Set("Retry-After", "5")
            http.Error(w, "Service overloaded", http.StatusServiceUnavailable)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

**Priority queues:**
```go
type PriorityRequest struct {
    Priority int  // 1 = highest
    Request  *http.Request
    Response chan Response
}

type PriorityHandler struct {
    queue    *PriorityQueue
    workers  int
}

func (h *PriorityHandler) HandleRequest(w http.ResponseWriter, r *http.Request) {
    priority := h.getPriority(r)  // из headers или path

    respChan := make(chan Response, 1)
    h.queue.Push(PriorityRequest{
        Priority: priority,
        Request:  r,
        Response: respChan,
    })

    select {
    case resp := <-respChan:
        json.NewEncoder(w).Encode(resp)
    case <-r.Context().Done():
        http.Error(w, "timeout", http.StatusGatewayTimeout)
    }
}

func (h *PriorityHandler) worker() {
    for {
        req := h.queue.Pop()  // блокируется
        // Обрабатываем запрос
        resp := h.process(req.Request)
        req.Response <- resp
    }
}
```

**Типичные ошибки.**
1. **Нет fallback** — один сбой ломает всё
2. **Fallback с ошибкой** — cache тоже может быть недоступен
3. **Нет мониторинга degradation** — не знаем, что в degraded mode
4. **Всё critical** — load shedding не работает

**На интервью.**
Покажи примеры fallback стратегий. Объясни, как определить critical vs non-critical функции. Расскажи про мониторинг degraded state.

*Частые follow-up вопросы:*
- Как тестировать degradation?
- Как избежать cascading degradation?
- Когда лучше fail fast вместо degrade?

---

### 8. Как обрабатывать Partial Failures в распределённых системах?

**Зачем спрашивают.**
Partial failures — реальность распределённых систем. Интервьюер проверяет понимание стратегий обработки и восстановления.

**Короткий ответ.**
Partial failure — когда часть операции успешна, часть нет. Стратегии: retry с idempotency, compensating transactions, async reconciliation, dead letter queues.

**Детальный разбор.**

**Типы partial failures:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Request partial success                                   │
│    Batch operation: 8 из 10 записей сохранились             │
│                                                              │
│ 2. Distributed transaction partial commit                    │
│    Order created, но Payment failed                         │
│                                                              │
│ 3. Network timeout ambiguity                                 │
│    Запрос отправлен, ответ не получен — успех или нет?      │
│                                                              │
│ 4. Cascading partial failures                                │
│    Service A частично → Service B частично → ...            │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Batch операция с partial success:**
```go
type BatchResult struct {
    Successful []string
    Failed     []BatchError
}

type BatchError struct {
    ID    string
    Error error
}

func ProcessBatch(ctx context.Context, items []Item) (*BatchResult, error) {
    result := &BatchResult{
        Successful: make([]string, 0, len(items)),
        Failed:     make([]BatchError, 0),
    }

    var wg sync.WaitGroup
    var mu sync.Mutex

    sem := make(chan struct{}, 10)  // concurrency limit

    for _, item := range items {
        wg.Add(1)
        go func(it Item) {
            defer wg.Done()

            sem <- struct{}{}
            defer func() { <-sem }()

            err := processItem(ctx, it)

            mu.Lock()
            defer mu.Unlock()

            if err != nil {
                result.Failed = append(result.Failed, BatchError{ID: it.ID, Error: err})
            } else {
                result.Successful = append(result.Successful, it.ID)
            }
        }(item)
    }

    wg.Wait()

    // Возвращаем результат даже при частичных ошибках
    // Caller решает, что делать
    return result, nil
}

// Использование
result, _ := ProcessBatch(ctx, items)

if len(result.Failed) > 0 {
    // Retry failed items
    retryItems := extractItems(items, result.Failed)
    retryResult, _ := ProcessBatch(ctx, retryItems)

    // После retry — в DLQ
    for _, failed := range retryResult.Failed {
        dlq.Send(failed)
    }
}
```

**Outbox pattern для надёжной отправки:**
```go
// Транзакционно сохраняем событие вместе с бизнес-данными
func CreateOrder(ctx context.Context, db *sql.DB, order Order) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Сохраняем order
    _, err = tx.ExecContext(ctx,
        "INSERT INTO orders (id, user_id, total) VALUES ($1, $2, $3)",
        order.ID, order.UserID, order.Total,
    )
    if err != nil {
        return err
    }

    // Сохраняем событие в outbox
    event := OrderCreatedEvent{OrderID: order.ID, UserID: order.UserID}
    eventData, _ := json.Marshal(event)

    _, err = tx.ExecContext(ctx,
        "INSERT INTO outbox (id, event_type, payload, created_at) VALUES ($1, $2, $3, $4)",
        uuid.New(), "order.created", eventData, time.Now(),
    )
    if err != nil {
        return err
    }

    return tx.Commit()
}

// Отдельный процесс публикует события
func OutboxProcessor(ctx context.Context, db *sql.DB, publisher Publisher) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            processOutbox(ctx, db, publisher)
        }
    }
}

func processOutbox(ctx context.Context, db *sql.DB, publisher Publisher) {
    tx, _ := db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // SELECT FOR UPDATE SKIP LOCKED для конкурентной обработки
    rows, _ := tx.QueryContext(ctx, `
        SELECT id, event_type, payload
        FROM outbox
        WHERE processed_at IS NULL
        ORDER BY created_at
        LIMIT 100
        FOR UPDATE SKIP LOCKED
    `)
    defer rows.Close()

    for rows.Next() {
        var id, eventType string
        var payload []byte
        rows.Scan(&id, &eventType, &payload)

        if err := publisher.Publish(ctx, eventType, payload); err != nil {
            continue  // retry later
        }

        tx.ExecContext(ctx, "UPDATE outbox SET processed_at = $1 WHERE id = $2", time.Now(), id)
    }

    tx.Commit()
}
```

**Dead Letter Queue:**
```go
type DLQMessage struct {
    OriginalTopic string
    Payload       []byte
    Error         string
    Attempts      int
    CreatedAt     time.Time
    LastAttempt   time.Time
}

type DLQHandler struct {
    db        *sql.DB
    processor MessageProcessor
    maxRetries int
}

func (h *DLQHandler) Process(ctx context.Context) error {
    messages, err := h.fetchMessages(ctx)
    if err != nil {
        return err
    }

    for _, msg := range messages {
        if msg.Attempts >= h.maxRetries {
            // Переводим в dead-dead (требует ручного вмешательства)
            h.markAsDead(ctx, msg)
            continue
        }

        err := h.processor.Process(ctx, msg.OriginalTopic, msg.Payload)
        if err != nil {
            h.incrementAttempts(ctx, msg)
            continue
        }

        h.markAsProcessed(ctx, msg)
    }

    return nil
}

// Async reconciliation
func Reconciler(ctx context.Context, orderRepo OrderRepo, paymentRepo PaymentRepo) {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            reconcile(ctx, orderRepo, paymentRepo)
        }
    }
}

func reconcile(ctx context.Context, orderRepo OrderRepo, paymentRepo PaymentRepo) {
    // Найти orders без payment
    pendingOrders, _ := orderRepo.FindPendingPayment(ctx, 30*time.Minute)

    for _, order := range pendingOrders {
        payment, _ := paymentRepo.FindByOrderID(ctx, order.ID)

        if payment == nil {
            // Payment не создан — retry
            createPayment(ctx, order)
        } else if payment.Status == "completed" {
            // Payment есть, order не обновлён — fix
            orderRepo.MarkAsPaid(ctx, order.ID)
        }
    }
}
```

**Типичные ошибки.**
1. **Игнорировать partial success** — теряются успешные операции
2. **Бесконечный retry** — нет DLQ
3. **Нет reconciliation** — inconsistency накапливается
4. **Синхронная обработка DLQ** — блокирует main flow

**На интервью.**
Объясни outbox pattern. Покажи, как работает DLQ. Расскажи про reconciliation для eventual consistency.

*Частые follow-up вопросы:*
- Как мониторить partial failures?
- Когда использовать sync vs async retry?
- Как тестировать partial failure scenarios?

---

## См. также

- [CAP Theorem](../07-distributed-systems/01-cap-theorem.md) — теорема CAP
- [Consensus](../07-distributed-systems/04-consensus.md) — алгоритмы консенсуса
- [Replication](../06-databases/05-replication.md) — репликация данных
- [Distributed Transactions](../07-distributed-systems/07-distributed-transactions.md) — распределённые транзакции
- [Message Queues & Events](./11-message-queues-events.md) — асинхронное взаимодействие

---

## Практика

1. **Distributed Lock:** Реализуй Redis lock с fencing token. Протестируй с network partition.

2. **Saga:** Реализуй Order Saga с orchestrator. Добавь compensating transactions.

3. **Tracing:** Настрой OpenTelemetry с Jaeger. Добавь custom spans и events.

4. **Idempotency:** Реализуй idempotency middleware. Добавь cleanup job.

5. **Graceful Degradation:** Добавь feature flags и fallback cache. Симулируй сбой dependency.

6. **Partial Failures:** Реализуй batch processing с DLQ. Добавь reconciler.

---

## Дополнительные материалы

- [Designing Data-Intensive Applications](https://dataintensive.net/) — Martin Kleppmann
- [The Raft Consensus Algorithm](https://raft.github.io/)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [OpenTelemetry Go](https://opentelemetry.io/docs/instrumentation/go/)
- [Istio Documentation](https://istio.io/latest/docs/)
- [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- Talks: "Distributed Systems in One Lesson" — Tim Berglund
