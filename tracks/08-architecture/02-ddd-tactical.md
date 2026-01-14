# 02. DDD Tactical Patterns

[← Назад к списку тем](README.md)

---

## Что такое DDD?

Domain-Driven Design (Eric Evans) — подход к разработке сложных систем, где в центре внимания находится domain (предметная область).

**Tactical patterns** — паттерны на уровне кода для моделирования domain.

---

## Entity

Объект с уникальной идентичностью, которая сохраняется на протяжении жизни объекта.

### Характеристики
- Имеет уникальный идентификатор
- Идентичность важнее атрибутов
- Мутабельный (состояние меняется со временем)

```go
// domain/order/entity.go
package order

import (
    "errors"
    "time"
)

type OrderID string

type Order struct {
    id          OrderID
    customerID  CustomerID
    items       []OrderItem
    status      Status
    totalAmount Money
    createdAt   time.Time
    updatedAt   time.Time
}

// Конструктор с валидацией инвариантов
func NewOrder(id OrderID, customerID CustomerID) (*Order, error) {
    if id == "" {
        return nil, errors.New("order ID is required")
    }
    if customerID == "" {
        return nil, errors.New("customer ID is required")
    }

    return &Order{
        id:         id,
        customerID: customerID,
        items:      make([]OrderItem, 0),
        status:     StatusDraft,
        createdAt:  time.Now(),
        updatedAt:  time.Now(),
    }, nil
}

// Методы изменяют состояние с валидацией бизнес-правил
func (o *Order) AddItem(item OrderItem) error {
    if o.status != StatusDraft {
        return errors.New("can only add items to draft orders")
    }
    if item.Quantity <= 0 {
        return errors.New("quantity must be positive")
    }

    o.items = append(o.items, item)
    o.recalculateTotal()
    o.updatedAt = time.Now()
    return nil
}

func (o *Order) Submit() error {
    if o.status != StatusDraft {
        return errors.New("only draft orders can be submitted")
    }
    if len(o.items) == 0 {
        return errors.New("cannot submit empty order")
    }

    o.status = StatusSubmitted
    o.updatedAt = time.Now()
    return nil
}

func (o *Order) Cancel() error {
    if o.status == StatusShipped || o.status == StatusDelivered {
        return errors.New("cannot cancel shipped or delivered order")
    }

    o.status = StatusCancelled
    o.updatedAt = time.Now()
    return nil
}

// Геттеры — приватные поля, публичный доступ через методы
func (o *Order) ID() OrderID         { return o.id }
func (o *Order) Status() Status      { return o.status }
func (o *Order) Items() []OrderItem  { return o.items }
func (o *Order) TotalAmount() Money  { return o.totalAmount }
```

### Equality

```go
// Entities сравниваются по ID, не по атрибутам
func (o *Order) Equals(other *Order) bool {
    if other == nil {
        return false
    }
    return o.id == other.id
}
```

---

## Value Object

Объект, определяемый своими атрибутами. Не имеет идентичности.

### Характеристики
- Идентичность через атрибуты
- Immutable (неизменяемый)
- Заменяется целиком, не модифицируется
- Может быть shared между entities

```go
// domain/shared/money.go
package shared

import (
    "errors"
    "fmt"
)

// Value Object - immutable
type Money struct {
    amount   int64  // cents
    currency string
}

// Конструктор с валидацией
func NewMoney(amount int64, currency string) (Money, error) {
    if currency == "" {
        return Money{}, errors.New("currency is required")
    }
    if len(currency) != 3 {
        return Money{}, errors.New("currency must be 3-letter code")
    }
    return Money{amount: amount, currency: currency}, nil
}

// Операции возвращают новый Value Object
func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, errors.New("cannot add different currencies")
    }
    return Money{amount: m.amount + other.amount, currency: m.currency}, nil
}

func (m Money) Multiply(factor int64) Money {
    return Money{amount: m.amount * factor, currency: m.currency}
}

// Equality по значениям
func (m Money) Equals(other Money) bool {
    return m.amount == other.amount && m.currency == other.currency
}

func (m Money) String() string {
    return fmt.Sprintf("%.2f %s", float64(m.amount)/100, m.currency)
}

// Геттеры
func (m Money) Amount() int64   { return m.amount }
func (m Money) Currency() string { return m.currency }
```

### Другие примеры Value Objects

```go
// Email
type Email struct {
    value string
}

func NewEmail(value string) (Email, error) {
    if !isValidEmail(value) {
        return Email{}, errors.New("invalid email format")
    }
    return Email{value: strings.ToLower(value)}, nil
}

func (e Email) String() string { return e.value }
func (e Email) Domain() string { return strings.Split(e.value, "@")[1] }

// Address
type Address struct {
    street     string
    city       string
    country    string
    postalCode string
}

func NewAddress(street, city, country, postalCode string) (Address, error) {
    // валидация
    return Address{street, city, country, postalCode}, nil
}

// DateRange
type DateRange struct {
    start time.Time
    end   time.Time
}

func NewDateRange(start, end time.Time) (DateRange, error) {
    if end.Before(start) {
        return DateRange{}, errors.New("end must be after start")
    }
    return DateRange{start, end}, nil
}

func (r DateRange) Contains(t time.Time) bool {
    return !t.Before(r.start) && !t.After(r.end)
}

func (r DateRange) Overlaps(other DateRange) bool {
    return !r.end.Before(other.start) && !other.end.Before(r.start)
}
```

---

## Aggregate

Кластер связанных объектов, которые рассматриваются как единое целое для изменения данных.

### Характеристики
- Имеет Aggregate Root — единственная точка входа
- Boundary определяет, что входит в aggregate
- Транзакционная консистентность внутри aggregate
- Eventual consistency между aggregates

```go
// domain/order/aggregate.go
package order

// Order — Aggregate Root
// OrderItem — внутри aggregate
// Product — ссылка по ID, не входит в aggregate

type Order struct {
    id         OrderID
    customerID CustomerID  // ссылка на другой aggregate
    items      []OrderItem // часть этого aggregate
    status     Status
    // ...
}

type OrderItem struct {
    productID ProductID  // ссылка, не сам Product
    name      string
    price     Money
    quantity  int
}

// Внешний мир взаимодействует только через Aggregate Root
func (o *Order) AddItem(productID ProductID, name string, price Money, qty int) error {
    // Бизнес-правила aggregate
    if o.status != StatusDraft {
        return ErrCannotModify
    }

    // Проверка инвариантов aggregate
    item := OrderItem{
        productID: productID,
        name:      name,
        price:     price,
        quantity:  qty,
    }

    o.items = append(o.items, item)
    o.recalculateTotal()
    return nil
}

// Нельзя получить прямой доступ к item и изменить его
func (o *Order) GetItem(productID ProductID) (OrderItem, bool) {
    for _, item := range o.items {
        if item.productID == productID {
            return item, true
        }
    }
    return OrderItem{}, false
}
```

### Правила проектирования Aggregates

1. **Protect invariants** — aggregate гарантирует свои бизнес-правила
2. **Small aggregates** — только минимально необходимое
3. **Reference by ID** — ссылки на другие aggregates по ID
4. **Update one aggregate per transaction**

```go
// Плохо: большой aggregate
type Order struct {
    Customer Customer  // весь Customer внутри
    Items    []OrderItem
    Payment  Payment   // весь Payment внутри
}

// Хорошо: маленький aggregate с ссылками
type Order struct {
    CustomerID CustomerID  // только ID
    Items      []OrderItem
    PaymentID  *PaymentID  // только ID, nullable
}
```

---

## Repository

Абстракция для доступа к aggregates. Скрывает детали persistence.

### Характеристики
- По одному Repository на Aggregate Root
- Interface в domain, реализация в infrastructure
- Работает с целым aggregate

```go
// domain/order/repository.go
package order

import "context"

// Repository interface — в domain layer
type Repository interface {
    // Save сохраняет весь aggregate
    Save(ctx context.Context, order *Order) error

    // FindByID возвращает весь aggregate
    FindByID(ctx context.Context, id OrderID) (*Order, error)

    // Query methods возвращают aggregates
    FindByCustomer(ctx context.Context, customerID CustomerID) ([]*Order, error)
    FindPending(ctx context.Context) ([]*Order, error)

    // NextID генерирует новый ID
    NextID() OrderID
}

// Реализация — в infrastructure layer
// infrastructure/postgres/order_repository.go

type PostgresOrderRepository struct {
    db *sql.DB
}

func (r *PostgresOrderRepository) Save(ctx context.Context, order *Order) error {
    tx, _ := r.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    // Сохраняем aggregate root
    _, err := tx.ExecContext(ctx, `
        INSERT INTO orders (id, customer_id, status, total_amount, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6)
        ON CONFLICT (id) DO UPDATE SET
            status = $3, total_amount = $4, updated_at = $6
    `, order.ID(), order.CustomerID(), order.Status(), order.TotalAmount(), order.CreatedAt(), order.UpdatedAt())
    if err != nil {
        return err
    }

    // Сохраняем items (часть aggregate)
    _, err = tx.ExecContext(ctx, "DELETE FROM order_items WHERE order_id = $1", order.ID())
    if err != nil {
        return err
    }

    for _, item := range order.Items() {
        _, err = tx.ExecContext(ctx, `
            INSERT INTO order_items (order_id, product_id, name, price, quantity)
            VALUES ($1, $2, $3, $4, $5)
        `, order.ID(), item.ProductID, item.Name, item.Price.Amount(), item.Quantity)
        if err != nil {
            return err
        }
    }

    return tx.Commit()
}

func (r *PostgresOrderRepository) FindByID(ctx context.Context, id OrderID) (*Order, error) {
    // Загружаем весь aggregate: order + items
    // Reconstruct domain object from DB rows
    // ...
}
```

### Repository vs DAO

| Repository | DAO |
|------------|-----|
| Domain объекты | Data structures |
| Aggregate целиком | CRUD для таблицы |
| В domain layer | В data layer |
| Абстракция | Прямой доступ к DB |

---

## Domain Service

Бизнес-логика, которая не принадлежит ни одному Entity/Value Object.

### Когда использовать
- Операция над несколькими aggregates
- Логика не подходит ни одному entity
- Stateless операция

```go
// domain/payment/service.go
package payment

import "context"

// Domain Service - stateless
type PaymentProcessor interface {
    Process(ctx context.Context, order *order.Order, method PaymentMethod) (*Payment, error)
}

type paymentProcessor struct {
    orderRepo   order.Repository
    paymentRepo Repository
}

func NewPaymentProcessor(or order.Repository, pr Repository) PaymentProcessor {
    return &paymentProcessor{
        orderRepo:   or,
        paymentRepo: pr,
    }
}

func (p *paymentProcessor) Process(ctx context.Context, ord *order.Order, method PaymentMethod) (*Payment, error) {
    // Бизнес-логика, затрагивающая несколько aggregates
    if ord.Status() != order.StatusSubmitted {
        return nil, errors.New("order must be submitted")
    }

    // Создать payment
    payment := NewPayment(p.paymentRepo.NextID(), ord.ID(), ord.TotalAmount(), method)

    // Валидация
    if err := payment.Validate(); err != nil {
        return nil, err
    }

    // Обработка через внешний gateway (будет в application service)
    // ...

    return payment, nil
}
```

---

## Domain Event

Что-то важное произошло в домене.

```go
// domain/order/events.go
package order

import "time"

// Base event
type DomainEvent interface {
    OccurredAt() time.Time
    AggregateID() string
}

type baseEvent struct {
    occurredAt  time.Time
    aggregateID string
}

func (e baseEvent) OccurredAt() time.Time { return e.occurredAt }
func (e baseEvent) AggregateID() string   { return e.aggregateID }

// Specific events
type OrderSubmitted struct {
    baseEvent
    CustomerID CustomerID
    TotalAmount Money
}

type OrderCancelled struct {
    baseEvent
    Reason string
}

type ItemAdded struct {
    baseEvent
    ProductID ProductID
    Quantity  int
}

// Aggregate собирает events
type Order struct {
    // ...
    events []DomainEvent
}

func (o *Order) Submit() error {
    if o.status != StatusDraft {
        return errors.New("only draft orders can be submitted")
    }

    o.status = StatusSubmitted
    o.updatedAt = time.Now()

    // Record event
    o.events = append(o.events, OrderSubmitted{
        baseEvent:   baseEvent{occurredAt: time.Now(), aggregateID: string(o.id)},
        CustomerID:  o.customerID,
        TotalAmount: o.totalAmount,
    })

    return nil
}

func (o *Order) Events() []DomainEvent {
    return o.events
}

func (o *Order) ClearEvents() {
    o.events = nil
}
```

---

## Factory

Создание сложных объектов и aggregates.

```go
// domain/order/factory.go
package order

type Factory struct {
    idGenerator IDGenerator
}

func NewFactory(idGen IDGenerator) *Factory {
    return &Factory{idGenerator: idGen}
}

// Создание нового order
func (f *Factory) CreateOrder(customerID CustomerID) (*Order, error) {
    id := f.idGenerator.Generate()
    return NewOrder(id, customerID)
}

// Создание из существующих данных (восстановление из DB)
func (f *Factory) Reconstitute(
    id OrderID,
    customerID CustomerID,
    items []OrderItem,
    status Status,
    totalAmount Money,
    createdAt, updatedAt time.Time,
) *Order {
    return &Order{
        id:          id,
        customerID:  customerID,
        items:       items,
        status:      status,
        totalAmount: totalAmount,
        createdAt:   createdAt,
        updatedAt:   updatedAt,
    }
}
```

---

## См. также

- [DDD Strategic Patterns](./03-ddd-strategic.md) — стратегические паттерны для больших систем
- [Data Modeling](../06-databases/04-data-modeling.md) — моделирование данных в базах данных

---

## На интервью

### Типичные вопросы

1. **Entity vs Value Object**
   - Entity: идентичность, мутабельный, lifecycle
   - VO: атрибуты, immutable, заменяемый

2. **Что такое Aggregate?**
   - Кластер объектов с одним корнем
   - Транзакционная граница
   - Invariants protection

3. **Где размещать бизнес-логику?**
   - В Entity/VO — логика связанная с объектом
   - В Domain Service — логика между aggregates
   - В Application Service — orchestration

4. **Почему aggregates должны быть маленькими?**
   - Concurrency — меньше конфликтов
   - Performance — меньше данных грузить
   - Scalability — легче распределять

### Red Flags

- Анемичная доменная модель (data classes + services)
- Большие aggregates (весь граф объектов)
- CRUD-мышление вместо domain modeling

---

[← Назад к списку тем](README.md)
