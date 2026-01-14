# 00. Fallacies & Fundamentals

[← Назад к списку тем](README.md)

---

## 8 Fallacies of Distributed Computing

```
┌─────────────────────────────────────────────────────────────────────┐
│            8 Fallacies of Distributed Computing                     │
│                     (Peter Deutsch, 1994)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. The network is reliable                                         │
│     ❌ Packets get lost, connections drop                            │
│     ✅ Retry, timeout, circuit breaker                               │
│                                                                     │
│  2. Latency is zero                                                 │
│     ❌ Network calls take time                                       │
│     ✅ Batch requests, async processing, caching                     │
│                                                                     │
│  3. Bandwidth is infinite                                           │
│     ❌ Network has capacity limits                                   │
│     ✅ Compress, paginate, use CDN                                   │
│                                                                     │
│  4. The network is secure                                           │
│     ❌ Data can be intercepted/modified                              │
│     ✅ TLS, authentication, authorization                            │
│                                                                     │
│  5. Topology doesn't change                                         │
│     ❌ Nodes join/leave, routes change                               │
│     ✅ Service discovery, health checks                              │
│                                                                     │
│  6. There is one administrator                                      │
│     ❌ Multiple teams, organizations                                 │
│     ✅ Clear ownership, SLAs, contracts                              │
│                                                                     │
│  7. Transport cost is zero                                          │
│     ❌ Serialization, network usage costs                            │
│     ✅ Efficient serialization (protobuf), minimize calls            │
│                                                                     │
│  8. The network is homogeneous                                      │
│     ❌ Different protocols, versions, vendors                        │
│     ✅ Standard protocols, API versioning                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Latency Numbers Every Programmer Should Know

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Latency Numbers (2024)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  L1 cache reference                           0.5 ns                │
│  Branch mispredict                            5   ns                │
│  L2 cache reference                           7   ns                │
│  Mutex lock/unlock                           25   ns                │
│  Main memory reference                      100   ns                │
│  Compress 1K bytes with Snappy            3,000   ns    3 μs        │
│  Send 1K bytes over 1 Gbps network       10,000   ns   10 μs        │
│  Read 4K randomly from SSD              150,000   ns  150 μs        │
│  Read 1 MB sequentially from memory     250,000   ns  250 μs        │
│  Round trip within same datacenter      500,000   ns  500 μs        │
│  Read 1 MB sequentially from SSD      1,000,000   ns    1 ms        │
│  Disk seek                           10,000,000   ns   10 ms        │
│  Read 1 MB sequentially from disk    20,000,000   ns   20 ms        │
│  Send packet CA→Netherlands→CA      150,000,000   ns  150 ms        │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Key insights:                                                      │
│  - Memory is ~100x faster than SSD                                  │
│  - SSD is ~100x faster than HDD                                     │
│  - Network within DC: ~500μs                                        │
│  - Cross-region: ~150ms                                             │
│  - Caching is critical!                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implications

```go
// ❌ Bad: Sequential network calls
func getOrderDetails(orderID string) (*OrderDetails, error) {
    order := orderService.Get(orderID)        // 500μs
    user := userService.Get(order.UserID)     // 500μs
    products := productService.GetMany(order.ProductIDs)  // 500μs
    // Total: 1.5ms minimum
}

// ✅ Good: Parallel calls
func getOrderDetails(orderID string) (*OrderDetails, error) {
    order := orderService.Get(orderID)  // 500μs

    var wg sync.WaitGroup
    var user *User
    var products []*Product

    wg.Add(2)
    go func() {
        defer wg.Done()
        user = userService.Get(order.UserID)
    }()
    go func() {
        defer wg.Done()
        products = productService.GetMany(order.ProductIDs)
    }()
    wg.Wait()

    // Total: ~1ms (500μs + max(500μs, 500μs))
}
```

---

## Failure Modes

### Types of Failures

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Failure Types                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Crash Failure                                                      │
│  ─────────────                                                      │
│  Node stops and never recovers (or recovers later)                  │
│  Detectable: heartbeat timeout                                      │
│  Example: process crash, hardware failure                           │
│                                                                     │
│  Omission Failure                                                   │
│  ───────────────                                                    │
│  Node fails to send or receive messages                             │
│  Partial: network partition                                         │
│  Example: packet loss, network congestion                           │
│                                                                     │
│  Timing Failure                                                     │
│  ──────────────                                                     │
│  Response outside expected time bounds                              │
│  Example: slow network, overloaded node                             │
│                                                                     │
│  Byzantine Failure                                                  │
│  ─────────────────                                                  │
│  Node behaves arbitrarily (malicious or buggy)                      │
│  Hardest to handle                                                  │
│  Example: corrupted data, security breach                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Partial Failures

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Partial Failure Scenario                          │
│                                                                     │
│      Client                                                         │
│         │                                                           │
│         │ Request                                                   │
│         ▼                                                           │
│    ┌─────────┐                                                      │
│    │  Node A │ ─────────────────────▶ ?                             │
│    └─────────┘       Network                                        │
│         │            Partition         ┌─────────┐                  │
│         │                              │  Node B │                  │
│         │                              └─────────┘                  │
│         │                                                           │
│         ◀── Timeout! But did Node B:                                │
│             - Receive the request?                                  │
│             - Process it?                                           │
│             - Send response (that got lost)?                        │
│                                                                     │
│  "The message was sent. The message was not received.               │
│   Both are true. The uncertainty is fundamental."                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Handling Partial Failures

```go
// 1. Idempotency: safe to retry
type CreateOrderRequest struct {
    IdempotencyKey string  // Client-generated unique key
    UserID         string
    Items          []Item
}

func createOrder(req CreateOrderRequest) (*Order, error) {
    // Check if already processed
    if order := cache.Get(req.IdempotencyKey); order != nil {
        return order, nil
    }

    // Process...
    order := processOrder(req)

    // Cache result
    cache.Set(req.IdempotencyKey, order, 24*time.Hour)

    return order, nil
}

// 2. Exactly-once semantics (if possible)
// - Idempotent operations
// - Transactional outbox
// - Deduplication at receiver

// 3. At-least-once with reconciliation
// - Retry until success
// - Periodic reconciliation job
// - Manual intervention for edge cases
```

---

## Synchronous vs Asynchronous

```
┌─────────────────────────────────────────────────────────────────────┐
│              Synchronous vs Asynchronous                             │
├───────────────────────────────┬─────────────────────────────────────┤
│        Synchronous            │        Asynchronous                 │
├───────────────────────────────┼─────────────────────────────────────┤
│ Request → Wait → Response     │ Request → Continue → Callback       │
│ Simpler mental model          │ Complex flow                        │
│ Tighter coupling              │ Loose coupling                      │
│ Lower latency (if fast)       │ Better throughput                   │
│ Failure propagates            │ Failure isolated                    │
│                               │                                     │
│ HTTP REST, gRPC               │ Message queues, events              │
└───────────────────────────────┴─────────────────────────────────────┘
```

### When to use what

```
Synchronous:
✅ User-facing requests needing immediate response
✅ Strong consistency required
✅ Simple CRUD operations
✅ Read operations (queries)

Asynchronous:
✅ Long-running operations
✅ High throughput needed
✅ Eventual consistency acceptable
✅ Decoupling services
✅ Handling spikes in traffic
```

---

## Scalability Patterns

### Vertical vs Horizontal

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Scaling Patterns                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Vertical Scaling (Scale Up)                                        │
│  ───────────────────────────                                        │
│  ┌──────────┐      ┌──────────────┐                                 │
│  │ 4 cores  │  →   │  64 cores    │                                 │
│  │ 16GB RAM │      │  512GB RAM   │                                 │
│  └──────────┘      └──────────────┘                                 │
│  + Simple, no code changes                                          │
│  - Limited by hardware, expensive, SPOF                             │
│                                                                     │
│  Horizontal Scaling (Scale Out)                                     │
│  ─────────────────────────────                                      │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                        │
│  │ N1  │  │ N2  │  │ N3  │  │ N4  │  │ N5  │                        │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘                        │
│  + Linear scaling, fault tolerant                                   │
│  - Complex, stateless requirement                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Stateless vs Stateful

```go
// Stateless: любой инстанс может обработать любой запрос
// Состояние хранится externally (DB, cache)

type OrderHandler struct {
    db    *sql.DB
    cache *redis.Client
}

func (h *OrderHandler) GetOrder(w http.ResponseWriter, r *http.Request) {
    orderID := chi.URLParam(r, "id")

    // State from external store
    order, err := h.db.GetOrder(r.Context(), orderID)
    // ...
}

// Stateful: нужен sticky sessions или routing
// Сложнее масштабировать

type WebSocketHandler struct {
    connections map[string]*websocket.Conn  // In-memory state!
}
// Клиент должен всегда попадать на тот же инстанс
```

---

## На интервью

### Типичные вопросы

1. **8 Fallacies — какие помните?**
   - Network unreliable, latency exists
   - Bandwidth limited, security needed
   - Topology changes, multiple admins
   - Transport costs, heterogeneous network

2. **Latency numbers — порядки величин?**
   - Memory: ~100ns
   - SSD: ~100μs
   - Network (DC): ~500μs
   - Network (cross-region): ~150ms

3. **Partial failures — как обрабатывать?**
   - Timeouts, retries
   - Idempotency
   - Circuit breaker
   - Reconciliation

4. **Sync vs Async — когда что?**
   - Sync: immediate response, strong consistency
   - Async: throughput, decoupling, long-running

5. **Horizontal scaling — requirements?**
   - Stateless services
   - External state storage
   - Load balancing
   - Service discovery

---

[← Назад к списку тем](README.md)
