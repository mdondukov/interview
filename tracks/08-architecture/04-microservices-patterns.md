# 04. Microservices Patterns

[← Назад к списку тем](README.md)

---

## Decomposition Strategies

### По бизнес-возможностям (Business Capability)

```
E-commerce разбивается по бизнес-функциям:

┌────────────────────────────────────────────────────────────────────┐
│                        E-Commerce Platform                          │
├────────────────┬────────────────┬────────────────┬────────────────┤
│   Product      │    Order       │   Inventory    │    Shipping    │
│   Catalog      │   Management   │   Management   │                │
├────────────────┼────────────────┼────────────────┼────────────────┤
│   Customer     │    Payment     │   Pricing &    │   Notification │
│   Management   │   Processing   │   Promotions   │                │
└────────────────┴────────────────┴────────────────┴────────────────┘
```

### По субдоменам (Subdomain)

```go
// Core Domain — уникальная бизнес-логика
// recommendation-service/
package recommendation

type Engine struct {
    mlModel    *MLModel
    userProfiles *UserProfileService
}

// Supporting Domain — важно, но не уникально
// order-service/
package order

type Service struct {
    repo Repository
    // ...
}

// Generic Domain — использовать готовое
// identity-service/ (или Auth0)
```

### Strangler Fig Pattern

Постепенная миграция от монолита к микросервисам.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Strangler Fig Migration                       │
│                                                                 │
│  Phase 1:                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Monolith                             │   │
│  │  [Users] [Orders] [Products] [Payments]                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Phase 2:                                                       │
│  ┌───────────────┐    ┌─────────────────────────────────────┐  │
│  │ User Service  │←───│            Monolith                 │  │
│  └───────────────┘    │  [Orders] [Products] [Payments]    │  │
│                       └─────────────────────────────────────┘  │
│                                                                 │
│  Phase 3:                                                       │
│  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐  │
│  │ User Service  │    │ Order Service │    │   Monolith    │  │
│  └───────────────┘    └───────────────┘    │  [Products]   │  │
│                                            │  [Payments]   │  │
│                                            └───────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

```go
// Facade перенаправляет трафик
type OrderFacade struct {
    legacyService *LegacyOrderService
    newService    *NewOrderService
    featureFlags  *FeatureFlags
}

func (f *OrderFacade) GetOrder(ctx context.Context, id string) (*Order, error) {
    if f.featureFlags.UseNewOrderService(id) {
        return f.newService.GetOrder(ctx, id)
    }
    return f.legacyService.GetOrder(ctx, id)
}
```

---

## Communication Patterns

### Synchronous (Request/Response)

**REST**
```go
// Прямой вызов между сервисами
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // Синхронный вызов Inventory Service
    available, err := s.inventoryClient.CheckAvailability(ctx, req.Items)
    if err != nil {
        return nil, err
    }
    if !available {
        return nil, ErrOutOfStock
    }

    // Создать заказ
    // ...
}
```

**gRPC**
```protobuf
service InventoryService {
    rpc CheckAvailability(CheckAvailabilityRequest) returns (CheckAvailabilityResponse);
    rpc ReserveItems(ReserveItemsRequest) returns (ReserveItemsResponse);
}
```

### Asynchronous (Event-Driven)

```go
// Публикация события
func (s *OrderService) PlaceOrder(ctx context.Context, order *Order) error {
    // Сохранить заказ
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }

    // Опубликовать событие (async)
    event := OrderPlacedEvent{
        OrderID:    order.ID,
        CustomerID: order.CustomerID,
        Items:      order.Items,
        Total:      order.Total,
        PlacedAt:   time.Now(),
    }

    return s.eventBus.Publish(ctx, "orders.placed", event)
}

// Подписчик в Inventory Service
func (h *InventoryHandler) OnOrderPlaced(ctx context.Context, event OrderPlacedEvent) error {
    // Зарезервировать товары
    return h.inventoryService.ReserveItems(ctx, event.Items)
}
```

### Request/Response vs Events

| Aspect | Sync (REST/gRPC) | Async (Events) |
|--------|------------------|----------------|
| Coupling | High | Low |
| Latency | Sum of all calls | Independent |
| Availability | All services must be up | Tolerant to failures |
| Complexity | Simple | Higher (eventual consistency) |
| Use case | Query, real-time | State changes, notifications |

---

## Data Management

### Database per Service

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Order Service  │  │ Product Service │  │ Customer Service│
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
    ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
    │Order DB │          │Product  │          │Customer │
    │(Postgres)│          │DB(Mongo)│          │DB(Postgres)│
    └─────────┘          └─────────┘          └─────────┘

Каждый сервис владеет своими данными
Нет прямого доступа к чужим БД
```

### Проблема: Cross-Service Queries

**API Composition**
```go
// BFF или API Gateway собирает данные
func (c *OrderComposer) GetOrderDetails(ctx context.Context, orderID string) (*OrderDetails, error) {
    // Параллельные запросы к разным сервисам
    var order *Order
    var customer *Customer
    var products []*Product

    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        var err error
        order, err = c.orderService.GetOrder(ctx, orderID)
        return err
    })

    g.Go(func() error {
        var err error
        customer, err = c.customerService.GetCustomer(ctx, order.CustomerID)
        return err
    })

    g.Go(func() error {
        var err error
        products, err = c.productService.GetProducts(ctx, order.ProductIDs)
        return err
    })

    if err := g.Wait(); err != nil {
        return nil, err
    }

    return &OrderDetails{
        Order:    order,
        Customer: customer,
        Products: products,
    }, nil
}
```

**CQRS (Command Query Responsibility Segregation)**
```go
// Write Model — отдельные сервисы
// Read Model — денормализованное view

// Order Service публикует события
// -> Projection Service слушает и строит read model

type OrderProjection struct {
    db *sql.DB
}

func (p *OrderProjection) OnOrderPlaced(event OrderPlacedEvent) error {
    // Денормализовать данные для быстрого чтения
    _, err := p.db.Exec(`
        INSERT INTO order_details_view
        (order_id, customer_name, product_names, total, status)
        VALUES ($1, $2, $3, $4, $5)
    `, event.OrderID, event.CustomerName, event.ProductNames, event.Total, "placed")
    return err
}
```

---

## Service Discovery

### Client-Side Discovery

```go
// Клиент запрашивает адреса у Service Registry
type ServiceDiscovery struct {
    consul *consul.Client
    cache  map[string][]string
}

func (d *ServiceDiscovery) GetInstances(serviceName string) ([]string, error) {
    services, _, err := d.consul.Health().Service(serviceName, "", true, nil)
    if err != nil {
        return nil, err
    }

    var addresses []string
    for _, s := range services {
        addr := fmt.Sprintf("%s:%d", s.Service.Address, s.Service.Port)
        addresses = append(addresses, addr)
    }

    return addresses, nil
}

// Клиент выбирает инстанс
func (c *OrderClient) GetOrder(ctx context.Context, id string) (*Order, error) {
    instances, _ := c.discovery.GetInstances("order-service")
    instance := c.loadBalancer.Choose(instances) // round-robin, random, etc.

    return c.httpClient.Get(instance + "/orders/" + id)
}
```

### Server-Side Discovery (Load Balancer)

```
┌────────┐      ┌──────────────┐      ┌─────────────────┐
│ Client │ ───▶ │ Load Balancer│ ───▶ │ Service Instance│
│        │      │ (Nginx/ALB)  │      │       1         │
└────────┘      │              │      └─────────────────┘
                │              │      ┌─────────────────┐
                │              │ ───▶ │ Service Instance│
                └──────────────┘      │       2         │
                        │             └─────────────────┘
                        ▼
                ┌──────────────┐
                │   Service    │
                │   Registry   │
                └──────────────┘
```

### Kubernetes Service Discovery

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order
  ports:
    - port: 80
      targetPort: 8080

# Другие сервисы обращаются по DNS:
# http://order-service.namespace.svc.cluster.local/orders
```

---

## API Gateway Pattern

```
┌────────────────────────────────────────────────────────────────┐
│                       API Gateway                               │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  - Authentication & Authorization                         │ │
│  │  - Rate Limiting                                          │ │
│  │  - Request Routing                                        │ │
│  │  - Load Balancing                                         │ │
│  │  - Response Caching                                       │ │
│  │  - Request/Response Transformation                        │ │
│  │  - SSL Termination                                        │ │
│  │  - Logging & Monitoring                                   │ │
│  └──────────────────────────────────────────────────────────┘ │
│                              │                                 │
│         ┌────────────────────┼────────────────────┐           │
│         │                    │                    │           │
│    ┌────▼────┐         ┌─────▼─────┐        ┌────▼────┐      │
│    │  User   │         │   Order   │        │ Product │      │
│    │ Service │         │  Service  │        │ Service │      │
│    └─────────┘         └───────────┘        └─────────┘      │
└────────────────────────────────────────────────────────────────┘
```

### Backend for Frontend (BFF)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Mobile App  │    │   Web App    │    │  Admin App   │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
  ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
  │ Mobile  │         │   Web   │         │  Admin  │
  │   BFF   │         │   BFF   │         │   BFF   │
  └────┬────┘         └────┬────┘         └────┬────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │  Services   │
                    └─────────────┘

Каждый BFF оптимизирован под свой клиент:
- Mobile BFF: меньше данных, pagination
- Web BFF: полные данные, SSR support
- Admin BFF: дополнительные endpoints
```

---

## Observability

### Distributed Tracing

```go
// OpenTelemetry instrumentation
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    ctx, span := tracer.Start(ctx, "CreateOrder")
    defer span.End()

    span.SetAttributes(
        attribute.String("customer_id", req.CustomerID),
        attribute.Int("item_count", len(req.Items)),
    )

    // Вызов другого сервиса — trace propagation
    available, err := s.inventoryClient.CheckAvailability(ctx, req.Items)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    // ...
}
```

### Structured Logging

```go
import "go.uber.org/zap"

func (s *OrderService) ProcessOrder(ctx context.Context, orderID string) error {
    logger := s.logger.With(
        zap.String("order_id", orderID),
        zap.String("trace_id", traceIDFromContext(ctx)),
    )

    logger.Info("processing order")

    if err := s.validate(order); err != nil {
        logger.Error("validation failed", zap.Error(err))
        return err
    }

    logger.Info("order processed successfully",
        zap.String("status", "completed"),
        zap.Duration("duration", time.Since(start)),
    )
    return nil
}
```

---

## На интервью

### Типичные вопросы

1. **Как разбить монолит на микросервисы?**
   - По бизнес-возможностям
   - По субдоменам
   - Strangler Fig Pattern

2. **Sync vs Async communication?**
   - Sync: простота, real-time
   - Async: loose coupling, resilience

3. **Как обеспечить consistency?**
   - Database per service
   - Saga pattern
   - Event Sourcing + CQRS

4. **Service Discovery подходы**
   - Client-side (Consul, Eureka)
   - Server-side (Load Balancer)
   - Kubernetes DNS

---

[← Назад к списку тем](README.md)
