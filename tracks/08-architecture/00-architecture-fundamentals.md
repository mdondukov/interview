# 00. Architecture Fundamentals

[← Назад к списку тем](README.md)

---

## Что такое хорошая архитектура?

Архитектура — это набор решений, которые сложно изменить позже. Хорошая архитектура минимизирует стоимость изменений и позволяет системе эволюционировать.

### Ключевые характеристики

1. **Maintainability** — легко понять и изменить
2. **Scalability** — справляется с ростом нагрузки
3. **Testability** — легко тестировать компоненты
4. **Flexibility** — легко добавлять новые функции
5. **Security** — защита от угроз

---

## SOLID Principles

### S — Single Responsibility Principle (SRP)

**Класс должен иметь только одну причину для изменения**

```go
// Плохо: один класс делает всё
type UserService struct{}

func (s *UserService) CreateUser(u User) error { ... }
func (s *UserService) SendEmail(u User) error { ... }
func (s *UserService) GenerateReport() error { ... }

// Хорошо: разделение ответственности
type UserRepository struct{}
func (r *UserRepository) Save(u User) error { ... }

type EmailService struct{}
func (e *EmailService) Send(to, subject, body string) error { ... }

type ReportGenerator struct{}
func (r *ReportGenerator) GenerateUserReport() error { ... }
```

### O — Open/Closed Principle (OCP)

**Открыт для расширения, закрыт для модификации**

```go
// Плохо: нужно менять код при добавлении типа
func CalculateArea(shape string, params map[string]float64) float64 {
    switch shape {
    case "rectangle":
        return params["width"] * params["height"]
    case "circle":
        return math.Pi * params["radius"] * params["radius"]
    // Добавление нового типа требует изменения функции
    }
    return 0
}

// Хорошо: расширяем через интерфейс
type Shape interface {
    Area() float64
}

type Rectangle struct {
    Width, Height float64
}
func (r Rectangle) Area() float64 { return r.Width * r.Height }

type Circle struct {
    Radius float64
}
func (c Circle) Area() float64 { return math.Pi * c.Radius * c.Radius }

// Новый тип просто реализует интерфейс
type Triangle struct {
    Base, Height float64
}
func (t Triangle) Area() float64 { return 0.5 * t.Base * t.Height }
```

### L — Liskov Substitution Principle (LSP)

**Объекты подтипа должны быть заменимы объектами базового типа**

```go
// Плохо: нарушение LSP
type Bird interface {
    Fly()
}

type Penguin struct{} // Пингвин не может летать!
func (p Penguin) Fly() { panic("penguins can't fly") }

// Хорошо: правильная иерархия
type Bird interface {
    Move()
}

type FlyingBird interface {
    Bird
    Fly()
}

type Penguin struct{}
func (p Penguin) Move() { /* walk/swim */ }

type Eagle struct{}
func (e Eagle) Move() { e.Fly() }
func (e Eagle) Fly() { /* fly */ }
```

### I — Interface Segregation Principle (ISP)

**Много специализированных интерфейсов лучше одного общего**

```go
// Плохо: толстый интерфейс
type Worker interface {
    Work()
    Eat()
    Sleep()
}

// Робот не ест и не спит
type Robot struct{}
func (r Robot) Work() { /* работает */ }
func (r Robot) Eat() { panic("robots don't eat") }
func (r Robot) Sleep() { panic("robots don't sleep") }

// Хорошо: разделение интерфейсов
type Worker interface {
    Work()
}

type Eater interface {
    Eat()
}

type Sleeper interface {
    Sleep()
}

type Human struct{}
func (h Human) Work() { /* работает */ }
func (h Human) Eat() { /* ест */ }
func (h Human) Sleep() { /* спит */ }

type Robot struct{}
func (r Robot) Work() { /* работает */ }
```

### D — Dependency Inversion Principle (DIP)

**Зависимости должны быть от абстракций, не от конкретных реализаций**

```go
// Плохо: зависимость от конкретной реализации
type UserService struct {
    db *PostgresDB  // привязка к PostgreSQL
}

func (s *UserService) GetUser(id string) (*User, error) {
    return s.db.Query("SELECT * FROM users WHERE id = $1", id)
}

// Хорошо: зависимость от абстракции
type UserRepository interface {
    FindByID(id string) (*User, error)
    Save(user *User) error
}

type UserService struct {
    repo UserRepository  // любая реализация
}

func (s *UserService) GetUser(id string) (*User, error) {
    return s.repo.FindByID(id)
}

// Реализации
type PostgresUserRepo struct{ db *sql.DB }
type MongoUserRepo struct{ client *mongo.Client }
type InMemoryUserRepo struct{ users map[string]*User }  // для тестов
```

---

## Coupling & Cohesion

### Coupling (Связность)

Мера зависимости между модулями. **Стремимся к низкой связности.**

| Тип | Описание | Пример |
|-----|----------|--------|
| **Content** | Один модуль изменяет данные другого | Прямой доступ к приватным полям |
| **Common** | Модули делят глобальные данные | Глобальные переменные |
| **Control** | Один передаёт флаги управления другому | `func Process(data, isAdmin bool)` |
| **Stamp** | Передача структуры, когда нужна часть | Передача всего User, когда нужен только ID |
| **Data** | Передача только нужных данных | `func GetUser(id string)` ✓ |
| **Message** | Взаимодействие через сообщения | Event-driven ✓ |

```go
// Высокая связность (плохо)
type OrderService struct {
    userDB     *sql.DB  // Прямой доступ к БД пользователей
    inventory  *InventoryService
}

func (s *OrderService) CreateOrder(userID string, items []Item) error {
    // Напрямую лезем в таблицу users
    var user User
    s.userDB.QueryRow("SELECT * FROM users WHERE id = $1", userID).Scan(&user)

    // Напрямую вызываем внутренний метод inventory
    s.inventory.decrementStock(items)

    return nil
}

// Низкая связность (хорошо)
type OrderService struct {
    users     UserGateway      // абстракция
    inventory InventoryGateway // абстракция
}

func (s *OrderService) CreateOrder(userID string, items []Item) error {
    user, err := s.users.GetByID(userID)
    if err != nil {
        return err
    }

    if err := s.inventory.Reserve(items); err != nil {
        return err
    }

    return nil
}
```

### Cohesion (Сплочённость)

Мера связанности элементов внутри модуля. **Стремимся к высокой сплочённости.**

| Тип | Описание | Качество |
|-----|----------|----------|
| **Coincidental** | Элементы случайно вместе | Худший |
| **Logical** | Элементы логически похожи | Плохо |
| **Temporal** | Элементы выполняются вместе | Средне |
| **Procedural** | Элементы в одной последовательности | Средне |
| **Communicational** | Работают с одними данными | Хорошо |
| **Sequential** | Выход одного — вход другого | Хорошо |
| **Functional** | Все для одной задачи | Лучший |

```go
// Низкая сплочённость (плохо)
type Utils struct{}

func (u *Utils) FormatDate(t time.Time) string { ... }
func (u *Utils) HashPassword(pwd string) string { ... }
func (u *Utils) SendEmail(to, body string) error { ... }
func (u *Utils) ResizeImage(img []byte) []byte { ... }

// Высокая сплочённость (хорошо)
type DateFormatter struct {
    format string
}
func (f *DateFormatter) Format(t time.Time) string { ... }
func (f *DateFormatter) Parse(s string) (time.Time, error) { ... }

type PasswordHasher struct {
    cost int
}
func (h *PasswordHasher) Hash(pwd string) string { ... }
func (h *PasswordHasher) Verify(pwd, hash string) bool { ... }
```

---

## Модульность

### Признаки хорошего модуля

1. **Чёткие границы** — понятно, что внутри, что снаружи
2. **Минимальный API** — экспортируем только необходимое
3. **Скрытие реализации** — детали внутри модуля
4. **Независимость** — можно тестировать изолированно

### Структура Go-проекта

```
project/
├── cmd/
│   └── api/
│       └── main.go          # Entry point
├── internal/
│   ├── domain/              # Бизнес-логика
│   │   ├── user/
│   │   │   ├── entity.go
│   │   │   ├── repository.go
│   │   │   └── service.go
│   │   └── order/
│   │       └── ...
│   ├── infrastructure/      # Внешние зависимости
│   │   ├── postgres/
│   │   ├── redis/
│   │   └── kafka/
│   └── api/                 # HTTP/gRPC handlers
│       ├── http/
│       └── grpc/
├── pkg/                     # Переиспользуемые пакеты
│   └── logger/
├── configs/
└── go.mod
```

### Information Hiding

```go
// package user

// Экспортируемый интерфейс (публичный контракт)
type Service interface {
    Create(ctx context.Context, input CreateInput) (*User, error)
    GetByID(ctx context.Context, id string) (*User, error)
}

// Внутренняя реализация (скрыта)
type service struct {
    repo       Repository
    hasher     PasswordHasher
    validator  Validator
}

// Конструктор — единственный способ создать сервис
func NewService(repo Repository, hasher PasswordHasher) Service {
    return &service{
        repo:      repo,
        hasher:    hasher,
        validator: newValidator(),
    }
}
```

---

## Composition vs Inheritance

Go не имеет классического наследования. Используем композицию и интерфейсы.

### Embedding (Встраивание)

```go
// Переиспользование через встраивание
type Logger struct{}
func (l Logger) Log(msg string) { fmt.Println(msg) }

type UserService struct {
    Logger  // встроенный тип
    repo UserRepository
}

func (s *UserService) CreateUser(u User) error {
    s.Log("Creating user")  // метод доступен напрямую
    return s.repo.Save(u)
}
```

### Композиция через интерфейсы

```go
// Маленькие интерфейсы
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Композиция интерфейсов
type ReadWriter interface {
    Reader
    Writer
}

// Функция принимает минимально необходимый интерфейс
func CopyData(dst Writer, src Reader) error {
    _, err := io.Copy(dst, src)
    return err
}
```

---

## Архитектурные стили

### Monolith

```
┌────────────────────────────────────────┐
│              Monolith                   │
│  ┌──────┐  ┌──────┐  ┌──────────────┐ │
│  │ User │  │Order │  │   Payment    │ │
│  │Module│  │Module│  │    Module    │ │
│  └──────┘  └──────┘  └──────────────┘ │
│                 │                      │
│           ┌─────▼─────┐               │
│           │  Database │               │
│           └───────────┘               │
└────────────────────────────────────────┘

+ Простота разработки и deploy
+ Легче отлаживать
- Масштабирование всего приложения
- Сложно менять технологии
```

### Microservices

```
┌─────────┐   ┌─────────┐   ┌─────────┐
│  User   │   │  Order  │   │ Payment │
│ Service │   │ Service │   │ Service │
└────┬────┘   └────┬────┘   └────┬────┘
     │             │             │
┌────▼────┐   ┌────▼────┐   ┌────▼────┐
│ User DB │   │Order DB │   │Pay DB   │
└─────────┘   └─────────┘   └─────────┘

+ Независимое масштабирование
+ Независимый deploy
+ Выбор технологий per service
- Сложность операций
- Распределённые транзакции
```

### Modular Monolith

```
┌────────────────────────────────────────┐
│           Modular Monolith              │
│  ┌──────────┐  ┌──────────┐           │
│  │   User   │  │   Order  │           │
│  │  Module  │←→│  Module  │  API      │
│  └────┬─────┘  └────┬─────┘  boundary │
│       │             │                  │
│  ┌────▼─────┐  ┌────▼─────┐           │
│  │User Table│  │Order Tbl │  Shared   │
│  └──────────┘  └──────────┘  DB       │
└────────────────────────────────────────┘

+ Чёткие границы модулей
+ Проще, чем microservices
+ Легко извлечь в сервис позже
```

---

## См. также

- [Design Fundamentals](../03-system-design/00-design-fundamentals.md) — основы проектирования систем и ключевые принципы
- [Clean Architecture](./01-clean-architecture.md) — организация кода с независимой бизнес-логикой

---

## На интервью

### Типичные вопросы

1. **Объясните SOLID принципы с примерами**
   - Каждый принцип + практический пример
   - Когда принцип может быть нарушен (trade-offs)

2. **Что такое coupling и cohesion?**
   - Определения
   - Как достичь low coupling, high cohesion

3. **Как вы структурируете Go/Python проект?**
   - Показать понимание модульности
   - Объяснить разделение на слои

4. **Monolith vs Microservices**
   - Trade-offs каждого подхода
   - Когда что выбирать
   - Modular monolith как middle ground

### Red Flags (чего избегать)

- Слепое следование паттернам без понимания "зачем"
- Over-engineering простых задач
- Игнорирование trade-offs

### Ключевые фразы

```
"The goal of architecture is to minimize the cost of change"
"Prefer composition over inheritance"
"Depend on abstractions, not concretions"
"Make things easy to change, not easy to write"
```

---

[← Назад к списку тем](README.md)
