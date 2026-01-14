# 01. Clean Architecture

[← Назад к списку тем](README.md)

---

## Основная идея

Clean Architecture (Robert C. Martin) — способ организации кода, при котором бизнес-логика независима от внешних деталей (UI, DB, frameworks).

### The Dependency Rule

**Зависимости направлены только внутрь — от внешних слоёв к внутренним.**

```
┌─────────────────────────────────────────────────────────────┐
│                    Frameworks & Drivers                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                Interface Adapters                      │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              Application Business Rules          │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │         Enterprise Business Rules          │  │  │  │
│  │  │  │              (Entities)                    │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  │                  (Use Cases)                    │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │   (Controllers, Gateways, Presenters)                 │  │
│  └───────────────────────────────────────────────────────┘  │
│   (Web, DB, External APIs, UI)                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Слои архитектуры

### 1. Entities (Domain Layer)

Самый внутренний слой. Содержит enterprise бизнес-правила.

```go
// domain/user/entity.go
package user

import (
    "errors"
    "time"
)

// Entity — чистая бизнес-логика, без зависимостей от внешнего мира
type User struct {
    ID        string
    Email     string
    Name      string
    Password  string  // hashed
    Status    Status
    CreatedAt time.Time
}

type Status string

const (
    StatusActive   Status = "active"
    StatusInactive Status = "inactive"
    StatusBanned   Status = "banned"
)

// Бизнес-правила в entity
func NewUser(email, name, hashedPassword string) (*User, error) {
    if email == "" {
        return nil, errors.New("email is required")
    }
    if !isValidEmail(email) {
        return nil, errors.New("invalid email format")
    }

    return &User{
        ID:        generateID(),
        Email:     email,
        Name:      name,
        Password:  hashedPassword,
        Status:    StatusActive,
        CreatedAt: time.Now(),
    }, nil
}

func (u *User) CanLogin() bool {
    return u.Status == StatusActive
}

func (u *User) Ban() {
    u.Status = StatusBanned
}

func (u *User) Activate() {
    u.Status = StatusActive
}
```

### 2. Use Cases (Application Layer)

Содержит application-specific бизнес-правила. Оркестрирует entities.

```go
// application/user/usecase.go
package user

import (
    "context"
    "errors"

    "myapp/domain/user"
)

// Input/Output DTOs для use case
type CreateUserInput struct {
    Email    string
    Name     string
    Password string
}

type CreateUserOutput struct {
    ID    string
    Email string
    Name  string
}

// Интерфейсы для внешних зависимостей (определяются в use case слое)
type UserRepository interface {
    Save(ctx context.Context, user *user.User) error
    FindByEmail(ctx context.Context, email string) (*user.User, error)
    FindByID(ctx context.Context, id string) (*user.User, error)
}

type PasswordHasher interface {
    Hash(password string) (string, error)
    Verify(password, hash string) bool
}

type EmailSender interface {
    SendWelcome(ctx context.Context, email, name string) error
}

// Use Case
type CreateUserUseCase struct {
    repo     UserRepository
    hasher   PasswordHasher
    emailer  EmailSender
}

func NewCreateUserUseCase(repo UserRepository, hasher PasswordHasher, emailer EmailSender) *CreateUserUseCase {
    return &CreateUserUseCase{
        repo:    repo,
        hasher:  hasher,
        emailer: emailer,
    }
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, input CreateUserInput) (*CreateUserOutput, error) {
    // 1. Проверить, что пользователь не существует
    existing, err := uc.repo.FindByEmail(ctx, input.Email)
    if err != nil {
        return nil, err
    }
    if existing != nil {
        return nil, errors.New("user with this email already exists")
    }

    // 2. Хешировать пароль
    hashedPassword, err := uc.hasher.Hash(input.Password)
    if err != nil {
        return nil, err
    }

    // 3. Создать entity
    newUser, err := user.NewUser(input.Email, input.Name, hashedPassword)
    if err != nil {
        return nil, err
    }

    // 4. Сохранить
    if err := uc.repo.Save(ctx, newUser); err != nil {
        return nil, err
    }

    // 5. Отправить welcome email (не критично)
    go uc.emailer.SendWelcome(ctx, input.Email, input.Name)

    return &CreateUserOutput{
        ID:    newUser.ID,
        Email: newUser.Email,
        Name:  newUser.Name,
    }, nil
}
```

### 3. Interface Adapters

Преобразуют данные между use cases и внешним миром.

**Controllers (HTTP handlers):**

```go
// adapters/http/user_handler.go
package http

import (
    "encoding/json"
    "net/http"

    userUC "myapp/application/user"
)

type UserHandler struct {
    createUser *userUC.CreateUserUseCase
}

// Request/Response — HTTP-specific структуры
type CreateUserRequest struct {
    Email    string `json:"email"`
    Name     string `json:"name"`
    Password string `json:"password"`
}

type CreateUserResponse struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
}

type ErrorResponse struct {
    Error string `json:"error"`
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // 1. Parse HTTP request
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    // 2. Convert to use case input
    input := userUC.CreateUserInput{
        Email:    req.Email,
        Name:     req.Name,
        Password: req.Password,
    }

    // 3. Execute use case
    output, err := h.createUser.Execute(r.Context(), input)
    if err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }

    // 4. Convert to HTTP response
    resp := CreateUserResponse{
        ID:    output.ID,
        Email: output.Email,
        Name:  output.Name,
    }

    respondJSON(w, http.StatusCreated, resp)
}
```

**Repository implementations:**

```go
// adapters/postgres/user_repository.go
package postgres

import (
    "context"
    "database/sql"

    "myapp/domain/user"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

// Реализует интерфейс из use case слоя
func (r *UserRepository) Save(ctx context.Context, u *user.User) error {
    query := `
        INSERT INTO users (id, email, name, password, status, created_at)
        VALUES ($1, $2, $3, $4, $5, $6)
    `
    _, err := r.db.ExecContext(ctx, query,
        u.ID, u.Email, u.Name, u.Password, u.Status, u.CreatedAt)
    return err
}

func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*user.User, error) {
    query := `SELECT id, email, name, password, status, created_at FROM users WHERE email = $1`

    var u user.User
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &u.ID, &u.Email, &u.Name, &u.Password, &u.Status, &u.CreatedAt)

    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }

    return &u, nil
}
```

### 4. Frameworks & Drivers

Внешний слой — конкретные технологии: web frameworks, databases, etc.

```go
// cmd/api/main.go
package main

import (
    "database/sql"
    "log"
    "net/http"

    _ "github.com/lib/pq"

    userUC "myapp/application/user"
    httpAdapter "myapp/adapters/http"
    pgAdapter "myapp/adapters/postgres"
    "myapp/infrastructure/bcrypt"
    "myapp/infrastructure/smtp"
)

func main() {
    // Frameworks & Drivers
    db, err := sql.Open("postgres", "postgres://...")
    if err != nil {
        log.Fatal(err)
    }

    // Interface Adapters (implementations)
    userRepo := pgAdapter.NewUserRepository(db)
    hasher := bcrypt.NewHasher(10)
    emailer := smtp.NewEmailSender("smtp://...")

    // Application (Use Cases)
    createUserUC := userUC.NewCreateUserUseCase(userRepo, hasher, emailer)

    // Interface Adapters (controllers)
    userHandler := httpAdapter.NewUserHandler(createUserUC)

    // Wire up HTTP
    http.HandleFunc("/users", userHandler.CreateUser)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## Структура проекта

```
myapp/
├── cmd/
│   └── api/
│       └── main.go                 # Entry point, wiring
├── domain/                         # Entities (innermost)
│   └── user/
│       ├── entity.go
│       └── value_objects.go
├── application/                    # Use Cases
│   └── user/
│       ├── create_user.go
│       ├── get_user.go
│       └── interfaces.go           # Repository, Service interfaces
├── adapters/                       # Interface Adapters
│   ├── http/
│   │   ├── user_handler.go
│   │   └── middleware.go
│   ├── grpc/
│   │   └── user_server.go
│   └── postgres/
│       └── user_repository.go
├── infrastructure/                 # Implementations of interfaces
│   ├── bcrypt/
│   │   └── hasher.go
│   └── smtp/
│       └── email_sender.go
└── go.mod
```

---

## Dependency Injection

### Manual DI (Preferred in Go)

```go
// Явное внедрение зависимостей через конструкторы
func main() {
    // Create dependencies
    db := connectDB()
    cache := connectRedis()

    // Inject into repositories
    userRepo := postgres.NewUserRepository(db)
    sessionRepo := redis.NewSessionRepository(cache)

    // Inject into use cases
    createUserUC := user.NewCreateUserUseCase(userRepo)
    loginUC := auth.NewLoginUseCase(userRepo, sessionRepo)

    // Inject into handlers
    userHandler := http.NewUserHandler(createUserUC)
    authHandler := http.NewAuthHandler(loginUC)

    // Setup router
    router := setupRoutes(userHandler, authHandler)

    http.ListenAndServe(":8080", router)
}
```

### Wire (Google's DI tool)

```go
// wire.go
//go:build wireinject

package main

import (
    "github.com/google/wire"

    "myapp/adapters/http"
    "myapp/adapters/postgres"
    "myapp/application/user"
)

func InitializeApp() (*App, error) {
    wire.Build(
        // Database
        provideDB,

        // Repositories
        postgres.NewUserRepository,

        // Use Cases
        user.NewCreateUserUseCase,

        // Handlers
        http.NewUserHandler,

        // App
        NewApp,
    )
    return nil, nil
}
```

---

## Тестирование

### Unit тесты для Use Case

```go
// application/user/create_user_test.go
func TestCreateUser_Success(t *testing.T) {
    // Arrange
    mockRepo := &MockUserRepository{
        FindByEmailFunc: func(ctx context.Context, email string) (*user.User, error) {
            return nil, nil // User not found
        },
        SaveFunc: func(ctx context.Context, u *user.User) error {
            return nil
        },
    }
    mockHasher := &MockHasher{
        HashFunc: func(pwd string) (string, error) {
            return "hashed_" + pwd, nil
        },
    }
    mockEmailer := &MockEmailSender{}

    uc := NewCreateUserUseCase(mockRepo, mockHasher, mockEmailer)

    // Act
    output, err := uc.Execute(context.Background(), CreateUserInput{
        Email:    "test@example.com",
        Name:     "Test User",
        Password: "password123",
    })

    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "test@example.com", output.Email)
    assert.True(t, mockRepo.SaveCalled)
}

func TestCreateUser_UserAlreadyExists(t *testing.T) {
    mockRepo := &MockUserRepository{
        FindByEmailFunc: func(ctx context.Context, email string) (*user.User, error) {
            return &user.User{Email: email}, nil // User exists
        },
    }

    uc := NewCreateUserUseCase(mockRepo, nil, nil)

    output, err := uc.Execute(context.Background(), CreateUserInput{
        Email: "existing@example.com",
    })

    assert.Error(t, err)
    assert.Nil(t, output)
    assert.Contains(t, err.Error(), "already exists")
}
```

### Integration тесты для Repository

```go
// adapters/postgres/user_repository_test.go
func TestUserRepository_Save(t *testing.T) {
    // Setup test database
    db := setupTestDB(t)
    defer db.Close()

    repo := NewUserRepository(db)

    user := &user.User{
        ID:    "test-id",
        Email: "test@example.com",
        Name:  "Test User",
    }

    err := repo.Save(context.Background(), user)
    assert.NoError(t, err)

    // Verify
    found, err := repo.FindByID(context.Background(), "test-id")
    assert.NoError(t, err)
    assert.Equal(t, user.Email, found.Email)
}
```

---

## Trade-offs

### Pros

- **Testability** — легко тестировать бизнес-логику в изоляции
- **Flexibility** — легко менять инфраструктуру (DB, frameworks)
- **Independence** — domain не зависит от внешних деталей
- **Clear boundaries** — понятно, где что находится

### Cons

- **More code** — больше файлов, структур, преобразований
- **Learning curve** — нужно понимать, где что размещать
- **Overkill for simple apps** — избыточно для CRUD-приложений
- **DTO mapping** — много преобразований между слоями

---

## Когда использовать

| Ситуация | Clean Architecture? |
|----------|---------------------|
| Простой CRUD сервис | Нет, overkill |
| Сложная бизнес-логика | Да |
| Долгоживущий проект | Да |
| Меняющиеся требования | Да |
| MVP / Прототип | Нет |
| Микросервис без логики | Нет |

---

## См. также

- [DDD Tactical Patterns](./02-ddd-tactical.md) — тактические паттерны для моделирования домена
- [OOP Principles](../01-java/01-oop-principles.md) — принципы ООП в Java

---

## На интервью

### Типичные вопросы

1. **Объясните Clean Architecture**
   - Dependency Rule
   - Слои и их ответственность
   - Почему зависимости внутрь

2. **Как тестировать use cases?**
   - Mock interfaces для зависимостей
   - Показать пример теста

3. **Где размещается валидация?**
   - Базовая (формат) — в entity
   - Бизнес-правила — в use case
   - HTTP-специфичная — в handler

4. **Как избежать boilerplate?**
   - Code generation для DTOs
   - Generic helpers
   - Но не жертвовать явностью ради краткости

---

[← Назад к списку тем](README.md)
