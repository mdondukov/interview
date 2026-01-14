# 06. Integration Patterns

[← Назад к списку тем](README.md)

---

## Saga Pattern

Управление распределёнными транзакциями через последовательность локальных транзакций.

### Проблема

```
2PC (Two-Phase Commit) не подходит для микросервисов:
- Блокирует ресурсы
- Снижает доступность
- Сложно масштабировать
```

### Choreography-based Saga

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Choreography Saga                                │
│                                                                      │
│  1. Order         2. Payment           3. Inventory      4. Shipping│
│     Service          Service              Service          Service  │
│                                                                      │
│  CreateOrder    ─▶  ProcessPayment   ─▶  ReserveStock  ─▶  Ship     │
│       │                  │                   │              │        │
│       │                  │                   │              │        │
│       │            PaymentFailed        StockUnavailable   │        │
│       │                  │                   │              │        │
│       ◀──────────────────│                   │              │        │
│  CompensateOrder    ◀────┘              ◀────┘              │        │
│                    ReleaseStock     ReleaseStock            │        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

```go
// Order Service
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    order := NewOrder(req)

    // Local transaction
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }

    // Publish event for next step
    return s.eventBus.Publish(ctx, OrderCreatedEvent{
        OrderID:    order.ID,
        CustomerID: order.CustomerID,
        Items:      order.Items,
        Amount:     order.Total,
    })
}

// Payment Service listens and processes
func (h *PaymentHandler) OnOrderCreated(ctx context.Context, event OrderCreatedEvent) error {
    payment, err := h.service.ProcessPayment(ctx, event.CustomerID, event.Amount)
    if err != nil {
        // Publish failure event for compensation
        return h.eventBus.Publish(ctx, PaymentFailedEvent{
            OrderID: event.OrderID,
            Reason:  err.Error(),
        })
    }

    // Success - publish for next step
    return h.eventBus.Publish(ctx, PaymentCompletedEvent{
        OrderID:   event.OrderID,
        PaymentID: payment.ID,
    })
}

// Order Service handles compensation
func (h *OrderHandler) OnPaymentFailed(ctx context.Context, event PaymentFailedEvent) error {
    return h.service.CancelOrder(ctx, event.OrderID, event.Reason)
}
```

### Orchestration-based Saga

```go
// Saga orchestrator
type OrderSaga struct {
    orderService     OrderService
    paymentService   PaymentService
    inventoryService InventoryService
    shippingService  ShippingService
}

type SagaState struct {
    OrderID       string
    CurrentStep   string
    Completed     []string
    Failed        bool
    FailureReason string
}

func (s *OrderSaga) Execute(ctx context.Context, req CreateOrderRequest) error {
    state := &SagaState{OrderID: req.OrderID}

    // Step 1: Create Order
    if err := s.createOrder(ctx, req, state); err != nil {
        return err
    }

    // Step 2: Process Payment
    if err := s.processPayment(ctx, state); err != nil {
        s.compensate(ctx, state)
        return err
    }

    // Step 3: Reserve Inventory
    if err := s.reserveInventory(ctx, state); err != nil {
        s.compensate(ctx, state)
        return err
    }

    // Step 4: Arrange Shipping
    if err := s.arrangeShipping(ctx, state); err != nil {
        s.compensate(ctx, state)
        return err
    }

    return nil
}

func (s *OrderSaga) compensate(ctx context.Context, state *SagaState) {
    // Compensate in reverse order
    for i := len(state.Completed) - 1; i >= 0; i-- {
        step := state.Completed[i]
        switch step {
        case "shipping":
            s.shippingService.Cancel(ctx, state.OrderID)
        case "inventory":
            s.inventoryService.ReleaseReservation(ctx, state.OrderID)
        case "payment":
            s.paymentService.Refund(ctx, state.OrderID)
        case "order":
            s.orderService.Cancel(ctx, state.OrderID)
        }
    }
}
```

---

## Outbox Pattern

Гарантированная публикация событий вместе с изменением данных.

### Проблема

```
// Опасно: что если publish упадёт после commit?
func CreateOrder(ctx context.Context, order *Order) error {
    tx.Begin()
    db.Save(order)
    tx.Commit()        // ✓ Order сохранён

    eventBus.Publish(OrderCreated)  // ✗ Может упасть!
}
```

### Решение: Transactional Outbox

```
┌────────────────────────────────────────────────────────────────────┐
│                      Outbox Pattern                                 │
│                                                                    │
│  1. Single Transaction                                             │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  BEGIN TRANSACTION                                           │  │
│  │    INSERT INTO orders (...)                                  │  │
│  │    INSERT INTO outbox (event_type, payload, ...)            │  │
│  │  COMMIT                                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  2. Message Relay (separate process)                               │
│  ┌──────────────────────────────────────┐                         │
│  │  Outbox Table       Message Broker   │                         │
│  │  ┌─────────┐         ┌─────────┐    │                         │
│  │  │ event1  │ ──────▶ │  Kafka  │    │                         │
│  │  │ event2  │         │         │    │                         │
│  │  │ event3  │         └─────────┘    │                         │
│  │  └─────────┘                        │                         │
│  └──────────────────────────────────────┘                         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

```go
// infrastructure/outbox/outbox.go
type OutboxMessage struct {
    ID          string
    AggregateID string
    EventType   string
    Payload     []byte
    CreatedAt   time.Time
    PublishedAt *time.Time
}

// Service saves with outbox in single transaction
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    tx, _ := s.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // Create order
    order := NewOrder(req)
    _, err := tx.ExecContext(ctx,
        "INSERT INTO orders (id, customer_id, status, total) VALUES ($1, $2, $3, $4)",
        order.ID, order.CustomerID, order.Status, order.Total,
    )
    if err != nil {
        return err
    }

    // Create outbox message
    event := OrderCreatedEvent{OrderID: order.ID, /*...*/}
    payload, _ := json.Marshal(event)

    _, err = tx.ExecContext(ctx,
        "INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at) VALUES ($1, $2, $3, $4, $5)",
        uuid.New().String(), order.ID, "order.created", payload, time.Now(),
    )
    if err != nil {
        return err
    }

    return tx.Commit()
}

// Relay процесс (отдельный worker)
func (r *OutboxRelay) Run(ctx context.Context) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            r.processOutbox(ctx)
        }
    }
}

func (r *OutboxRelay) processOutbox(ctx context.Context) error {
    // Get unpublished messages
    rows, _ := r.db.QueryContext(ctx,
        "SELECT id, event_type, payload FROM outbox WHERE published_at IS NULL ORDER BY created_at LIMIT 100",
    )

    for rows.Next() {
        var msg OutboxMessage
        rows.Scan(&msg.ID, &msg.EventType, &msg.Payload)

        // Publish to message broker
        if err := r.broker.Publish(ctx, msg.EventType, msg.Payload); err != nil {
            continue // Retry next tick
        }

        // Mark as published
        r.db.ExecContext(ctx,
            "UPDATE outbox SET published_at = $1 WHERE id = $2",
            time.Now(), msg.ID,
        )
    }

    return nil
}
```

---

## Change Data Capture (CDC)

Захват изменений из БД и публикация как событий.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Change Data Capture                               │
│                                                                     │
│  ┌─────────────┐                                                    │
│  │  Service    │                                                    │
│  │  writes to  │                                                    │
│  │  database   │                                                    │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐        │
│  │  Database   │ ───▶ │   Debezium  │ ───▶ │    Kafka    │        │
│  │  (WAL/Binlog)│      │   (CDC)     │      │             │        │
│  └─────────────┘      └─────────────┘      └──────┬──────┘        │
│                                                    │                │
│                             ┌─────────────────────┬┴────────────┐  │
│                             │                     │             │  │
│                       ┌─────▼─────┐        ┌──────▼────┐  ┌─────▼──┐
│                       │ Consumer 1│        │Consumer 2 │  │Consumer│
│                       │ (Search)  │        │(Analytics)│  │   N    │
│                       └───────────┘        └───────────┘  └────────┘
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```yaml
# Debezium connector config
{
  "name": "orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "cdc_user",
    "database.password": "***",
    "database.dbname": "orders",
    "table.include.list": "public.orders,public.order_items",
    "topic.prefix": "orders",
    "plugin.name": "pgoutput"
  }
}
```

### CDC vs Outbox

| Aspect | Outbox | CDC |
|--------|--------|-----|
| Implementation | Application code | Infrastructure |
| Event format | Custom, business events | Raw DB changes |
| Coupling | Business logic aware | Schema coupled |
| Complexity | Simpler | Requires CDC tool |

---

## API Composition

Агрегация данных из нескольких сервисов.

```go
// BFF или API Gateway
type OrderDetailsComposer struct {
    orderClient     OrderServiceClient
    customerClient  CustomerServiceClient
    productClient   ProductServiceClient
}

func (c *OrderDetailsComposer) GetOrderDetails(ctx context.Context, orderID string) (*OrderDetailsResponse, error) {
    g, ctx := errgroup.WithContext(ctx)

    var order *Order
    var customer *Customer
    var products []*Product

    // Parallel calls
    g.Go(func() error {
        var err error
        order, err = c.orderClient.GetOrder(ctx, orderID)
        return err
    })

    g.Go(func() error {
        var err error
        // После получения order, получить customer
        // Нужна координация
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    // После получения order, запросить customer и products
    g2, ctx := errgroup.WithContext(ctx)

    g2.Go(func() error {
        var err error
        customer, err = c.customerClient.GetCustomer(ctx, order.CustomerID)
        return err
    })

    g2.Go(func() error {
        var err error
        productIDs := extractProductIDs(order.Items)
        products, err = c.productClient.GetProducts(ctx, productIDs)
        return err
    })

    if err := g2.Wait(); err != nil {
        return nil, err
    }

    return &OrderDetailsResponse{
        Order:    order,
        Customer: customer,
        Products: products,
    }, nil
}
```

---

## Idempotent Receiver

```go
type IdempotentHandler struct {
    db      *sql.DB
    handler EventHandler
}

func (h *IdempotentHandler) Handle(ctx context.Context, event Event) error {
    // Use database transaction for idempotency check + processing
    tx, _ := h.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // Check if already processed (SELECT FOR UPDATE to prevent race)
    var processed bool
    err := tx.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM processed_events WHERE event_id = $1) FOR UPDATE",
        event.ID,
    ).Scan(&processed)
    if err != nil {
        return err
    }

    if processed {
        return nil // Skip, already processed
    }

    // Process event
    if err := h.handler.Handle(ctx, event); err != nil {
        return err
    }

    // Mark as processed
    _, err = tx.ExecContext(ctx,
        "INSERT INTO processed_events (event_id, processed_at) VALUES ($1, $2)",
        event.ID, time.Now(),
    )
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

---

## На интервью

### Типичные вопросы

1. **Saga pattern — когда использовать?**
   - Распределённые транзакции
   - Choreography vs Orchestration trade-offs

2. **Outbox pattern — зачем?**
   - Гарантированная публикация событий
   - Atomicity между DB и message broker

3. **CDC vs Outbox?**
   - CDC: infrastructure-level, raw changes
   - Outbox: application-level, business events

4. **Как обеспечить idempotency?**
   - Unique event IDs
   - Idempotency keys
   - Deduplication table

---

[← Назад к списку тем](README.md)
