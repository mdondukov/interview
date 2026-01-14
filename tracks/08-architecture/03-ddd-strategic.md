# 03. DDD Strategic Patterns

[← Назад к списку тем](README.md)

---

## Обзор

Strategic DDD — паттерны для работы с большими системами, разбиения на части и организации взаимодействия между командами.

---

## Bounded Context

Явная граница, внутри которой термины и модели имеют конкретное значение.

### Проблема

В большой системе одно и то же слово означает разное:
- **Customer** в Sales: потенциальный покупатель, лиды
- **Customer** в Billing: плательщик, платёжные данные
- **Customer** в Support: пользователь с тикетами

### Решение

```
┌─────────────────────────────────────────────────────────────────────┐
│                          E-Commerce System                           │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │   Sales BC      │  │   Billing BC    │  │   Support BC    │     │
│  │                 │  │                 │  │                 │     │
│  │  Customer:      │  │  Customer:      │  │  Customer:      │     │
│  │  - Name         │  │  - AccountID    │  │  - UserID       │     │
│  │  - Email        │  │  - PaymentMethod│  │  - Tickets[]    │     │
│  │  - Leads[]      │  │  - Invoices[]   │  │  - Priority     │     │
│  │                 │  │                 │  │                 │     │
│  │  Product:       │  │  Product:       │  │  Product:       │     │
│  │  - SKU          │  │  - PriceID      │  │  (не нужен)     │     │
│  │  - Description  │  │  - TaxCategory  │  │                 │     │
│  │  - Price        │  │                 │  │                 │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Определение границ

```go
// sales/customer.go - Sales Bounded Context
package sales

type Customer struct {
    ID          CustomerID
    Name        string
    Email       string
    Phone       string
    Company     string
    Leads       []Lead
    Deals       []Deal
    AssignedTo  SalesRepID
}

// billing/customer.go - Billing Bounded Context
package billing

type Customer struct {
    AccountID      AccountID
    ExternalID     string        // ссылка на Sales Customer
    PaymentMethods []PaymentMethod
    BillingAddress Address
    TaxID          string
    Invoices       []Invoice
}

// support/customer.go - Support Bounded Context
package support

type Customer struct {
    UserID      UserID
    ExternalID  string           // ссылка на Sales Customer
    Tickets     []TicketID
    Priority    SupportPriority
    SLALevel    SLALevel
}
```

---

## Ubiquitous Language

Единый язык между разработчиками и экспертами предметной области внутри Bounded Context.

### Примеры

```
Плохо (технический жаргон):
"Нужно проапдейтить юзер-рекорд и триггернуть ивент"

Хорошо (ubiquitous language):
"Когда клиент размещает заказ, система должна зарезервировать товары"
```

### В коде

```go
// Плохо: технические термины
type OrderRecord struct {
    ID        int64
    UserID    int64
    ItemsList []ItemDTO
    StateFlag int
}

func (r *OrderRecord) UpdateState(newState int) error {
    r.StateFlag = newState
    return nil
}

// Хорошо: ubiquitous language
type Order struct {
    ID         OrderID
    CustomerID CustomerID
    Items      []OrderItem
    Status     OrderStatus
}

func (o *Order) Submit() error {
    // ...
}

func (o *Order) Ship() error {
    // ...
}

func (o *Order) Cancel(reason CancellationReason) error {
    // ...
}
```

---

## Context Mapping

Отношения между Bounded Contexts.

### Типы отношений

```
┌────────────────────────────────────────────────────────────────────┐
│                      Context Map                                    │
│                                                                    │
│    ┌──────────┐                         ┌──────────┐              │
│    │  Sales   │ ─── Customer/Supplier ──▶│  Billing │              │
│    │   (U)    │                         │   (D)    │              │
│    └──────────┘                         └──────────┘              │
│         │                                    │                     │
│         │ Shared Kernel                      │ ACL                 │
│         │                                    │                     │
│    ┌────▼─────┐                         ┌────▼─────┐              │
│    │ Product  │ ─── Conformist ─────────▶│ External │              │
│    │ Catalog  │                         │ Shipping │              │
│    └──────────┘                         └──────────┘              │
│                                                                    │
│    U = Upstream (поставщик)                                        │
│    D = Downstream (потребитель)                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 1. Partnership

Два контекста развиваются вместе, зависят друг от друга.

```go
// Sales и Shipping развиваются координированно
// Общие интерфейсы, синхронизированные релизы
```

### 2. Shared Kernel

Общий код между контекстами.

```go
// shared/types.go
package shared

type Money struct {
    Amount   int64
    Currency string
}

type Address struct {
    Street     string
    City       string
    Country    string
    PostalCode string
}

// Используется в Sales и Billing
```

### 3. Customer/Supplier

Upstream поставляет, Downstream потребляет.

```go
// Sales (Upstream) публикует события
type CustomerCreatedEvent struct {
    CustomerID string
    Email      string
    Name       string
}

// Billing (Downstream) подписывается и обрабатывает
func (h *BillingEventHandler) OnCustomerCreated(event CustomerCreatedEvent) {
    // Создать billing account для нового customer
    account := billing.NewAccount(event.CustomerID, event.Email)
    h.repo.Save(account)
}
```

### 4. Conformist

Downstream полностью принимает модель Upstream.

```go
// External shipping API определяет модель
// Наш код просто использует их структуры

type ShippingProvider interface {
    CreateShipment(order TheirOrderFormat) (TheirShipmentResponse, error)
}
```

### 5. Anti-Corruption Layer (ACL)

Downstream защищает свою модель от влияния внешних систем.

```go
// acl/legacy_adapter.go
package acl

import (
    "myapp/domain/order"
    "myapp/legacy"
)

// ACL переводит между нашей моделью и legacy системой
type LegacyOrderAdapter struct {
    legacyClient *legacy.Client
}

func (a *LegacyOrderAdapter) GetOrder(id order.OrderID) (*order.Order, error) {
    // 1. Получить данные из legacy
    legacyOrder, err := a.legacyClient.FetchOrder(string(id))
    if err != nil {
        return nil, err
    }

    // 2. Преобразовать в нашу доменную модель
    return a.translate(legacyOrder), nil
}

func (a *LegacyOrderAdapter) translate(lo *legacy.Order) *order.Order {
    // Legacy использует статусы как числа
    status := a.translateStatus(lo.StatusCode)

    // Legacy хранит деньги как float
    amount := order.NewMoney(int64(lo.TotalPrice*100), "USD")

    // Собираем наш Order
    return order.Reconstitute(
        order.OrderID(lo.ID),
        order.CustomerID(lo.CustomerNumber),
        a.translateItems(lo.LineItems),
        status,
        amount,
    )
}

func (a *LegacyOrderAdapter) translateStatus(code int) order.Status {
    switch code {
    case 0:
        return order.StatusDraft
    case 1:
        return order.StatusSubmitted
    case 2:
        return order.StatusShipped
    default:
        return order.StatusUnknown
    }
}
```

### 6. Open Host Service

Upstream предоставляет публичный API для всех потребителей.

```go
// Public API для внешних клиентов
// openapi/orders.yaml

paths:
  /api/v1/orders:
    get:
      summary: List orders
    post:
      summary: Create order

  /api/v1/orders/{id}:
    get:
      summary: Get order by ID
```

### 7. Published Language

Общий формат данных между контекстами.

```go
// published/events/order_events.go
package events

// Published Language — стабильный формат событий
type OrderPlacedV1 struct {
    Version     string    `json:"version"` // "1.0"
    OrderID     string    `json:"order_id"`
    CustomerID  string    `json:"customer_id"`
    TotalAmount int64     `json:"total_amount_cents"`
    Currency    string    `json:"currency"`
    PlacedAt    time.Time `json:"placed_at"`
}

// При изменении — новая версия
type OrderPlacedV2 struct {
    Version     string    `json:"version"` // "2.0"
    OrderID     string    `json:"order_id"`
    CustomerID  string    `json:"customer_id"`
    TotalAmount int64     `json:"total_amount_cents"`
    Currency    string    `json:"currency"`
    PlacedAt    time.Time `json:"placed_at"`
    // Новые поля в V2
    ShippingMethod string `json:"shipping_method"`
}
```

---

## Event Storming

Техника для обнаружения доменных событий и bounded contexts.

```
┌────────────────────────────────────────────────────────────────────┐
│                     Event Storming Board                            │
│                                                                    │
│  Timeline ─────────────────────────────────────────────────────▶   │
│                                                                    │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │   Customer   │    │    Order     │    │   Payment    │        │
│  │  Registered  │ ──▶│    Placed    │ ──▶│   Received   │        │
│  │   (Orange)   │    │   (Orange)   │    │   (Orange)   │        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│         │                   │                   │                  │
│         ▼                   ▼                   ▼                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │   Register   │    │  Place Order │    │  Charge Card │        │
│  │   (Blue)     │    │    (Blue)    │    │    (Blue)    │        │
│  │   Command    │    │   Command    │    │   Command    │        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│         │                   │                   │                  │
│         ▼                   ▼                   ▼                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐        │
│  │  Registration│    │    Order     │    │   Payment    │        │
│  │    Form      │    │    Cart      │    │   Gateway    │        │
│  │  (Yellow)    │    │  (Yellow)    │    │  (Pink/Ext)  │        │
│  └──────────────┘    └──────────────┘    └──────────────┘        │
│                                                                    │
│  Legend:                                                           │
│  🟠 Orange = Domain Event                                          │
│  🔵 Blue = Command                                                 │
│  🟡 Yellow = Aggregate / Actor                                     │
│  🟣 Pink = External System                                         │
│  🟢 Green = Read Model / Query                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Subdomains

Разделение бизнеса на области.

### Core Domain

Ключевое конкурентное преимущество. Максимум внимания.

```go
// Core Domain для e-commerce: персонализация и рекомендации
package recommendations

type RecommendationEngine struct {
    // Сложная логика, уникальная для нашего бизнеса
    mlModel     MLModel
    userProfile UserProfiler
    realtime    RealtimeScorer
}
```

### Supporting Subdomain

Важно для бизнеса, но не уникально.

```go
// Supporting: управление заказами
package orders

type OrderService struct {
    // Стандартная логика, можно использовать паттерны
}
```

### Generic Subdomain

Общая функциональность, не специфичная для бизнеса.

```go
// Generic: аутентификация — использовать готовое решение
// Auth0, Keycloak, или стандартная библиотека
```

---

## Microservices и Bounded Contexts

```
┌────────────────────────────────────────────────────────────────────┐
│                1 Bounded Context ≈ 1 Microservice                  │
│                                                                    │
│    ┌─────────────────┐           ┌─────────────────┐              │
│    │  Sales Context  │           │ Billing Context │              │
│    │    (Service)    │───────────│    (Service)    │              │
│    │                 │  Events   │                 │              │
│    │  ┌───────────┐  │           │  ┌───────────┐  │              │
│    │  │ Sales DB  │  │           │  │Billing DB │  │              │
│    │  └───────────┘  │           │  └───────────┘  │              │
│    └─────────────────┘           └─────────────────┘              │
│                                                                    │
│    Но: можно несколько BC в одном сервисе (modular monolith)      │
│    Или: один BC разбит на несколько сервисов (для scale)          │
└────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [DDD Tactical Patterns](./02-ddd-tactical.md) — тактические паттерны на уровне кода
- [Microservices Patterns](./04-microservices-patterns.md) — паттерны микросервисной архитектуры

---

## На интервью

### Типичные вопросы

1. **Что такое Bounded Context?**
   - Граница, внутри которой модель консистентна
   - Ubiquitous language внутри BC
   - Пример: Customer в разных контекстах

2. **Как определить границы BC?**
   - Event Storming
   - По командам/владению
   - По ubiquitous language

3. **Что такое ACL и когда использовать?**
   - Защита от внешней/legacy модели
   - Translation layer
   - Когда внешняя модель не соответствует нашей

4. **Context Map и типы отношений**
   - Перечислить типы: Partnership, Shared Kernel, Customer/Supplier, etc.
   - Объяснить когда какой

### Ключевые выводы

```
1. Bounded Context = граница модели и языка
2. Один контекст — один ubiquitous language
3. ACL защищает нашу модель от внешних влияний
4. Context Map показывает отношения между контекстами
5. Core Domain — куда инвестировать больше всего
```

---

[← Назад к списку тем](README.md)
