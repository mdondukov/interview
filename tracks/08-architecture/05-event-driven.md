# 05. Event-Driven Architecture

[← Назад к списку тем](README.md)

---

## Основные концепции

### Event Types

```
┌────────────────────────────────────────────────────────────────────┐
│                        Types of Events                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. Domain Events                                                  │
│     "Что-то произошло в бизнес-домене"                             │
│     OrderPlaced, PaymentReceived, ItemShipped                      │
│                                                                    │
│  2. Integration Events                                             │
│     "Уведомление между bounded contexts"                           │
│     CustomerCreated (для других сервисов)                          │
│                                                                    │
│  3. Event-Carried State Transfer                                   │
│     "Событие несёт все данные"                                     │
│     CustomerUpdated { name, email, address, ... }                  │
│                                                                    │
│  4. Notification Events                                            │
│     "Минимум данных, получатель запрашивает остальное"             │
│     CustomerUpdated { id }                                         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Event Structure

```go
// Base event
type Event struct {
    ID            string                 `json:"id"`
    Type          string                 `json:"type"`
    Source        string                 `json:"source"`
    Time          time.Time              `json:"time"`
    CorrelationID string                 `json:"correlation_id"`
    Data          map[string]interface{} `json:"data"`
}

// Типизированный domain event
type OrderPlacedEvent struct {
    EventID       string    `json:"event_id"`
    EventType     string    `json:"event_type"` // "order.placed"
    Timestamp     time.Time `json:"timestamp"`
    CorrelationID string    `json:"correlation_id"`

    // Payload
    OrderID      string       `json:"order_id"`
    CustomerID   string       `json:"customer_id"`
    Items        []OrderItem  `json:"items"`
    TotalAmount  int64        `json:"total_amount"`
    Currency     string       `json:"currency"`
}
```

---

## Event Sourcing

Хранение состояния как последовательности событий.

### Концепция

```
Традиционный подход (Current State):
┌─────────────────────────────────────────┐
│ Order: { status: "shipped", items: [...] } │
└─────────────────────────────────────────┘

Event Sourcing (Event Log):
┌───────────────────────────────────────────────────────────┐
│ 1. OrderCreated { items: [...] }                          │
│ 2. ItemAdded { productId: "123", qty: 2 }                 │
│ 3. PaymentReceived { amount: 100 }                        │
│ 4. OrderShipped { trackingNumber: "XYZ" }                 │
└───────────────────────────────────────────────────────────┘

Current state = replay(events)
```

### Implementation

```go
// domain/order/events.go
type OrderEvent interface {
    EventType() string
    OccurredAt() time.Time
}

type OrderCreated struct {
    OrderID    string
    CustomerID string
    OccurredAt time.Time
}

type ItemAdded struct {
    OrderID   string
    ProductID string
    Quantity  int
    Price     int64
    OccurredAt time.Time
}

type OrderSubmitted struct {
    OrderID    string
    TotalAmount int64
    OccurredAt time.Time
}

// domain/order/aggregate.go
type Order struct {
    id         string
    customerID string
    items      []OrderItem
    status     Status
    total      int64

    changes []OrderEvent // uncommitted events
    version int          // for optimistic concurrency
}

// Apply event to rebuild state
func (o *Order) Apply(event OrderEvent) {
    switch e := event.(type) {
    case *OrderCreated:
        o.id = e.OrderID
        o.customerID = e.CustomerID
        o.status = StatusDraft
    case *ItemAdded:
        o.items = append(o.items, OrderItem{
            ProductID: e.ProductID,
            Quantity:  e.Quantity,
            Price:     e.Price,
        })
    case *OrderSubmitted:
        o.status = StatusSubmitted
        o.total = e.TotalAmount
    }
    o.version++
}

// Command creates event
func (o *Order) AddItem(productID string, qty int, price int64) error {
    if o.status != StatusDraft {
        return errors.New("cannot modify non-draft order")
    }

    event := &ItemAdded{
        OrderID:   o.id,
        ProductID: productID,
        Quantity:  qty,
        Price:     price,
        OccurredAt: time.Now(),
    }

    o.Apply(event)
    o.changes = append(o.changes, event)
    return nil
}

// Reconstruct from events
func LoadOrder(events []OrderEvent) *Order {
    order := &Order{}
    for _, event := range events {
        order.Apply(event)
    }
    order.changes = nil // clear uncommitted
    return order
}
```

### Event Store

```go
// infrastructure/eventstore/store.go
type EventStore interface {
    // Append events for aggregate
    Append(ctx context.Context, aggregateID string, events []Event, expectedVersion int) error

    // Load all events for aggregate
    Load(ctx context.Context, aggregateID string) ([]Event, error)

    // Subscribe to events
    Subscribe(ctx context.Context, fromPosition int64, handler EventHandler) error
}

type PostgresEventStore struct {
    db *sql.DB
}

func (s *PostgresEventStore) Append(ctx context.Context, aggregateID string, events []Event, expectedVersion int) error {
    tx, _ := s.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // Optimistic concurrency check
    var currentVersion int
    err := tx.QueryRowContext(ctx,
        "SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1",
        aggregateID,
    ).Scan(&currentVersion)
    if err != nil {
        return err
    }

    if currentVersion != expectedVersion {
        return ErrConcurrencyConflict
    }

    // Append events
    for i, event := range events {
        _, err := tx.ExecContext(ctx, `
            INSERT INTO events (aggregate_id, version, event_type, data, timestamp)
            VALUES ($1, $2, $3, $4, $5)
        `,
            aggregateID,
            expectedVersion + i + 1,
            event.Type,
            event.Data,
            event.Timestamp,
        )
        if err != nil {
            return err
        }
    }

    return tx.Commit()
}
```

---

## CQRS (Command Query Responsibility Segregation)

Разделение моделей для чтения и записи.

```
┌─────────────────────────────────────────────────────────────────────┐
│                            CQRS Pattern                              │
│                                                                     │
│                           ┌─────────────┐                           │
│                           │   Client    │                           │
│                           └──────┬──────┘                           │
│                                  │                                  │
│              ┌───────────────────┴───────────────────┐              │
│              │                                       │              │
│       ┌──────▼──────┐                        ┌───────▼──────┐       │
│       │   Command   │                        │    Query     │       │
│       │   Handler   │                        │   Handler    │       │
│       └──────┬──────┘                        └───────┬──────┘       │
│              │                                       │              │
│       ┌──────▼──────┐                        ┌───────▼──────┐       │
│       │   Domain    │                        │  Read Model  │       │
│       │   Model     │                        │  (Projections)│      │
│       └──────┬──────┘                        └───────┬──────┘       │
│              │                                       ▲              │
│       ┌──────▼──────┐      Events            ┌───────┴──────┐       │
│       │   Event     │──────────────────────▶│  Projection  │       │
│       │   Store     │                        │   Handler    │       │
│       └─────────────┘                        └──────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Write Side (Commands)

```go
// application/commands/place_order.go
type PlaceOrderCommand struct {
    OrderID    string
    CustomerID string
    Items      []OrderItemDTO
}

type PlaceOrderHandler struct {
    eventStore EventStore
}

func (h *PlaceOrderHandler) Handle(ctx context.Context, cmd PlaceOrderCommand) error {
    // Load aggregate from events
    events, err := h.eventStore.Load(ctx, cmd.OrderID)
    if err != nil {
        return err
    }

    order := LoadOrder(events)

    // Execute business logic
    if err := order.Place(); err != nil {
        return err
    }

    // Save new events
    return h.eventStore.Append(ctx, cmd.OrderID, order.Changes(), order.Version())
}
```

### Read Side (Queries + Projections)

```go
// application/projections/order_details.go
type OrderDetailsProjection struct {
    db *sql.DB
}

func (p *OrderDetailsProjection) Handle(event Event) error {
    switch e := event.(type) {
    case *OrderCreated:
        return p.onOrderCreated(e)
    case *ItemAdded:
        return p.onItemAdded(e)
    case *OrderSubmitted:
        return p.onOrderSubmitted(e)
    }
    return nil
}

func (p *OrderDetailsProjection) onOrderCreated(e *OrderCreated) error {
    _, err := p.db.Exec(`
        INSERT INTO order_details (order_id, customer_id, status, created_at)
        VALUES ($1, $2, 'draft', $3)
    `, e.OrderID, e.CustomerID, e.OccurredAt)
    return err
}

func (p *OrderDetailsProjection) onItemAdded(e *ItemAdded) error {
    _, err := p.db.Exec(`
        UPDATE order_details
        SET item_count = item_count + 1,
            total_amount = total_amount + $2
        WHERE order_id = $1
    `, e.OrderID, e.Price * int64(e.Quantity))
    return err
}

// Query handler
type GetOrderDetailsHandler struct {
    db *sql.DB
}

func (h *GetOrderDetailsHandler) Handle(ctx context.Context, orderID string) (*OrderDetailsDTO, error) {
    var dto OrderDetailsDTO
    err := h.db.QueryRowContext(ctx, `
        SELECT order_id, customer_id, status, item_count, total_amount, created_at
        FROM order_details
        WHERE order_id = $1
    `, orderID).Scan(&dto.OrderID, &dto.CustomerID, &dto.Status, &dto.ItemCount, &dto.TotalAmount, &dto.CreatedAt)

    return &dto, err
}
```

---

## Eventual Consistency

### Проблема

```
User создаёт order -> Order event опубликован -> Inventory ещё не обновлён
User проверяет inventory -> Показывает старое значение
```

### Стратегии

**1. UI показывает pending state**
```javascript
// После создания заказа
setOrderStatus("processing");
// Пока не получим подтверждение через WebSocket
onWebSocketMessage((msg) => {
    if (msg.type === "order_confirmed") {
        setOrderStatus("confirmed");
    }
});
```

**2. Causal consistency через read-your-writes**
```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    order := createOrder(r)

    // Возвращаем version/position
    response := OrderResponse{
        OrderID: order.ID,
        Version: order.Version,  // клиент использует для следующих запросов
    }
}

func (h *Handler) GetOrder(w http.ResponseWriter, r *http.Request) {
    minVersion := r.Header.Get("X-Min-Version")

    // Ждём пока read model догонит
    order := h.waitForVersion(orderID, minVersion)
}
```

**3. Synchronous update критичных read models**
```go
func (h *PlaceOrderHandler) Handle(ctx context.Context, cmd PlaceOrderCommand) error {
    // ... создать order ...

    // Синхронно обновить критичную проекцию
    if err := h.inventoryProjection.UpdateSync(order.Items()); err != nil {
        // Rollback или retry
    }

    // Async обновить остальные
    h.eventBus.Publish(orderPlacedEvent)

    return nil
}
```

---

## Event-Driven Patterns

### Choreography vs Orchestration

**Choreography (децентрализованно)**
```
┌─────────┐    OrderPlaced    ┌─────────┐    PaymentReceived    ┌─────────┐
│  Order  │─────────────────▶│ Payment │───────────────────────▶│Shipping │
│ Service │                   │ Service │                       │ Service │
└─────────┘                   └─────────┘                       └─────────┘

Каждый сервис знает, на какие события реагировать.
+ Loose coupling
- Сложно отследить flow
```

**Orchestration (централизованно)**
```
                      ┌─────────────┐
                      │ Orchestrator│
                      │   (Saga)    │
                      └──────┬──────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
    │  Order  │         │ Payment │         │Shipping │
    │ Service │         │ Service │         │ Service │
    └─────────┘         └─────────┘         └─────────┘

Orchestrator управляет последовательностью.
+ Понятный flow
- Single point of coordination
```

### Idempotency

```go
// Consumer должен быть idempotent
type OrderEventHandler struct {
    processedEvents map[string]bool
    db              *sql.DB
}

func (h *OrderEventHandler) Handle(ctx context.Context, event Event) error {
    // Check if already processed
    var exists bool
    h.db.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM processed_events WHERE event_id = $1)",
        event.ID,
    ).Scan(&exists)

    if exists {
        return nil // Already processed, skip
    }

    // Process event
    if err := h.processEvent(event); err != nil {
        return err
    }

    // Mark as processed
    h.db.ExecContext(ctx,
        "INSERT INTO processed_events (event_id, processed_at) VALUES ($1, $2)",
        event.ID, time.Now(),
    )

    return nil
}
```

---

## Trade-offs

### Event Sourcing

| Pros | Cons |
|------|------|
| Complete audit trail | Complexity |
| Time travel (debugging) | Event schema evolution |
| Replay for projections | Eventual consistency |
| Natural fit for ES+CQRS | More storage |

### CQRS

| Pros | Cons |
|------|------|
| Optimized read models | Complexity |
| Scale reads independently | Eventual consistency |
| Different storage for R/W | More infrastructure |

---

## См. также

- [Message Queues & Events в Go](../00-go/11-message-queues-events.md) — работа с очередями сообщений и событиями в Go
- [Distributed Transactions](../07-distributed-systems/07-distributed-transactions.md) — распределённые транзакции и консистентность

---

## На интервью

### Типичные вопросы

1. **Что такое Event Sourcing?**
   - Хранение состояния как последовательности событий
   - Replay для восстановления state
   - Append-only event log

2. **CQRS — зачем разделять?**
   - Разные требования к read/write
   - Оптимизация read models
   - Масштабирование независимо

3. **Eventual consistency — как справляться?**
   - UI feedback (pending states)
   - Read-your-writes
   - Synchronous critical paths

4. **Choreography vs Orchestration?**
   - Trade-offs каждого
   - Когда что использовать

---

[← Назад к списку тем](README.md)
