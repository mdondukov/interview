# 09. Testing Strategies

[← Назад к списку тем](README.md)

---

## Test Pyramid

```
                    ▲
                   /│\
                  / │ \
                 /  │  \       E2E Tests
                /   │   \      (few, slow, expensive)
               /────│────\
              /     │     \
             /      │      \   Integration Tests
            /       │       \  (some, medium speed)
           /────────│────────\
          /         │         \
         /          │          \ Unit Tests
        /           │           \(many, fast, cheap)
       /────────────│────────────\
```

| Level | Speed | Scope | Confidence | Quantity |
|-------|-------|-------|------------|----------|
| Unit | ms | Single function/class | Low-Medium | Many |
| Integration | seconds | Multiple components | Medium-High | Some |
| E2E | minutes | Full system | High | Few |

---

## Unit Tests

### Характеристики
- Тестируют одну единицу (функция, метод, класс)
- Изолированы от внешних зависимостей
- Быстрые (миллисекунды)
- Детерминированные

### Example: Testing Domain Logic

```go
// domain/order/order_test.go
func TestOrder_AddItem_Success(t *testing.T) {
    // Arrange
    order, _ := NewOrder("order-1", "customer-1")

    // Act
    err := order.AddItem("product-1", "Widget", Money{Amount: 1000, Currency: "USD"}, 2)

    // Assert
    assert.NoError(t, err)
    assert.Len(t, order.Items(), 1)
    assert.Equal(t, 2, order.Items()[0].Quantity)
}

func TestOrder_AddItem_FailsWhenNotDraft(t *testing.T) {
    order, _ := NewOrder("order-1", "customer-1")
    order.Submit() // Change status to submitted

    err := order.AddItem("product-1", "Widget", Money{Amount: 1000}, 2)

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "draft")
}

func TestOrder_Submit_FailsWhenEmpty(t *testing.T) {
    order, _ := NewOrder("order-1", "customer-1")

    err := order.Submit()

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "empty")
}
```

### Table-Driven Tests

```go
func TestMoney_Add(t *testing.T) {
    tests := []struct {
        name      string
        a         Money
        b         Money
        want      Money
        wantError bool
    }{
        {
            name: "same currency",
            a:    Money{Amount: 100, Currency: "USD"},
            b:    Money{Amount: 50, Currency: "USD"},
            want: Money{Amount: 150, Currency: "USD"},
        },
        {
            name:      "different currency",
            a:         Money{Amount: 100, Currency: "USD"},
            b:         Money{Amount: 50, Currency: "EUR"},
            wantError: true,
        },
        {
            name: "zero amount",
            a:    Money{Amount: 100, Currency: "USD"},
            b:    Money{Amount: 0, Currency: "USD"},
            want: Money{Amount: 100, Currency: "USD"},
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := tt.a.Add(tt.b)

            if tt.wantError {
                assert.Error(t, err)
                return
            }

            assert.NoError(t, err)
            assert.Equal(t, tt.want, result)
        })
    }
}
```

---

## Mocking

### Interface-based Mocking

```go
// Repository interface
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Mock implementation
type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepository) Save(ctx context.Context, user *User) error {
    args := m.Called(ctx, user)
    return args.Error(0)
}

// Test using mock
func TestUserService_CreateUser(t *testing.T) {
    // Arrange
    mockRepo := new(MockUserRepository)
    mockHasher := new(MockPasswordHasher)
    service := NewUserService(mockRepo, mockHasher)

    mockRepo.On("FindByEmail", mock.Anything, "test@example.com").Return(nil, nil)
    mockHasher.On("Hash", "password123").Return("hashed_password", nil)
    mockRepo.On("Save", mock.Anything, mock.AnythingOfType("*User")).Return(nil)

    // Act
    user, err := service.CreateUser(context.Background(), CreateUserInput{
        Email:    "test@example.com",
        Name:     "Test User",
        Password: "password123",
    })

    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "test@example.com", user.Email)
    mockRepo.AssertExpectations(t)
}
```

### Fake Implementation

```go
// Fake - working implementation for tests
type FakeUserRepository struct {
    users map[string]*User
    mu    sync.RWMutex
}

func NewFakeUserRepository() *FakeUserRepository {
    return &FakeUserRepository{
        users: make(map[string]*User),
    }
}

func (r *FakeUserRepository) FindByID(ctx context.Context, id string) (*User, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    user, ok := r.users[id]
    if !ok {
        return nil, nil
    }
    return user, nil
}

func (r *FakeUserRepository) Save(ctx context.Context, user *User) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    r.users[user.ID] = user
    return nil
}

// Test with fake
func TestUserService_GetUser_WithFake(t *testing.T) {
    repo := NewFakeUserRepository()
    repo.Save(context.Background(), &User{ID: "user-1", Name: "John"})

    service := NewUserService(repo, nil)

    user, err := service.GetUser(context.Background(), "user-1")

    assert.NoError(t, err)
    assert.Equal(t, "John", user.Name)
}
```

---

## Integration Tests

Тестируют взаимодействие нескольких компонентов.

### Database Integration Test

```go
// integration/user_repository_test.go
func TestUserRepository_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    // Setup test database
    db := setupTestDB(t)
    defer db.Close()

    repo := postgres.NewUserRepository(db)
    ctx := context.Background()

    // Test Save
    user := &User{
        ID:    "user-123",
        Email: "test@example.com",
        Name:  "Test User",
    }

    err := repo.Save(ctx, user)
    require.NoError(t, err)

    // Test FindByID
    found, err := repo.FindByID(ctx, "user-123")
    require.NoError(t, err)
    assert.Equal(t, user.Email, found.Email)

    // Test FindByEmail
    found, err = repo.FindByEmail(ctx, "test@example.com")
    require.NoError(t, err)
    assert.Equal(t, user.ID, found.ID)
}

func setupTestDB(t *testing.T) *sql.DB {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" {
        dsn = "postgres://test:test@localhost:5432/test?sslmode=disable"
    }

    db, err := sql.Open("postgres", dsn)
    require.NoError(t, err)

    // Run migrations
    runMigrations(db)

    // Clean up after test
    t.Cleanup(func() {
        db.Exec("TRUNCATE users CASCADE")
    })

    return db
}
```

### Testcontainers

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestWithPostgresContainer(t *testing.T) {
    ctx := context.Background()

    // Start PostgreSQL container
    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
    )
    require.NoError(t, err)
    defer container.Terminate(ctx)

    // Get connection string
    connStr, err := container.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    // Connect and test
    db, err := sql.Open("postgres", connStr)
    require.NoError(t, err)
    defer db.Close()

    // Run your tests
    repo := postgres.NewUserRepository(db)
    // ...
}
```

---

## Contract Tests

Проверяют контракт между сервисами (consumer и provider).

### Consumer-Driven Contract (Pact)

```go
// Consumer test
func TestOrderService_Consumer(t *testing.T) {
    // Create Pact client
    pact := dsl.Pact{
        Consumer: "order-service",
        Provider: "user-service",
    }
    defer pact.Teardown()

    // Define expected interaction
    pact.
        AddInteraction().
        Given("User 123 exists").
        UponReceiving("A request for user 123").
        WithRequest(dsl.Request{
            Method: "GET",
            Path:   dsl.String("/users/123"),
        }).
        WillRespondWith(dsl.Response{
            Status: 200,
            Body: dsl.Match(map[string]interface{}{
                "id":    dsl.Like("123"),
                "email": dsl.Like("user@example.com"),
                "name":  dsl.Like("John Doe"),
            }),
        })

    // Test
    err := pact.Verify(func() error {
        client := NewUserClient(pact.Server.URL)
        user, err := client.GetUser(context.Background(), "123")
        if err != nil {
            return err
        }
        if user.ID != "123" {
            return fmt.Errorf("unexpected user id")
        }
        return nil
    })

    assert.NoError(t, err)
}
```

### Provider Verification

```go
// Provider test
func TestUserService_Provider(t *testing.T) {
    // Start your actual service
    server := startTestServer()
    defer server.Close()

    // Verify against contracts
    pact.VerifyProvider(t, types.VerifyRequest{
        ProviderBaseURL: server.URL,
        PactURLs:        []string{"./pacts/order-service-user-service.json"},
        StateHandlers: types.StateHandlers{
            "User 123 exists": func(setup bool, s types.ProviderState) (types.ProviderStateResponse, error) {
                if setup {
                    // Setup state: create user 123 in test DB
                    createTestUser("123", "user@example.com", "John Doe")
                }
                return nil, nil
            },
        },
    })
}
```

---

## E2E Tests

```go
// e2e/order_flow_test.go
func TestOrderFlow_E2E(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping e2e test")
    }

    // Setup
    baseURL := os.Getenv("API_URL")
    if baseURL == "" {
        baseURL = "http://localhost:8080"
    }

    client := NewAPIClient(baseURL)

    // Create user
    user, err := client.CreateUser(CreateUserRequest{
        Email: "e2e-test@example.com",
        Name:  "E2E Test User",
    })
    require.NoError(t, err)
    defer client.DeleteUser(user.ID)

    // Create order
    order, err := client.CreateOrder(CreateOrderRequest{
        CustomerID: user.ID,
        Items: []OrderItem{
            {ProductID: "product-1", Quantity: 2},
        },
    })
    require.NoError(t, err)
    assert.Equal(t, "draft", order.Status)

    // Submit order
    order, err = client.SubmitOrder(order.ID)
    require.NoError(t, err)
    assert.Equal(t, "submitted", order.Status)

    // Verify order in list
    orders, err := client.ListOrders(user.ID)
    require.NoError(t, err)
    assert.Len(t, orders, 1)
    assert.Equal(t, order.ID, orders[0].ID)
}
```

---

## Test Doubles

```
┌────────────────────────────────────────────────────────────────────┐
│                       Test Doubles                                  │
├──────────┬─────────────────────────────────────────────────────────┤
│  Dummy   │ Passed but never used. Fills parameter list.            │
│          │ Example: logger in constructor that's not tested        │
├──────────┼─────────────────────────────────────────────────────────┤
│  Stub    │ Returns predefined values. No behavior verification.    │
│          │ Example: repo.FindByID() always returns same user       │
├──────────┼─────────────────────────────────────────────────────────┤
│  Spy     │ Records calls for later verification.                   │
│          │ Example: emailSender.Send() — verify it was called      │
├──────────┼─────────────────────────────────────────────────────────┤
│  Mock    │ Pre-programmed expectations + verification.             │
│          │ Example: mockRepo.On("Save").Return(nil)                │
├──────────┼─────────────────────────────────────────────────────────┤
│  Fake    │ Working implementation, simplified for testing.         │
│          │ Example: in-memory repository instead of PostgreSQL     │
└──────────┴─────────────────────────────────────────────────────────┘
```

---

## TDD (Test-Driven Development)

```
Red → Green → Refactor

1. RED: Write failing test
2. GREEN: Write minimal code to pass
3. REFACTOR: Improve code, keep tests green
```

```go
// 1. RED - Write test first
func TestCalculator_Add(t *testing.T) {
    calc := NewCalculator()

    result := calc.Add(2, 3)

    assert.Equal(t, 5, result)
}
// Test fails: Calculator doesn't exist

// 2. GREEN - Minimal implementation
type Calculator struct{}

func NewCalculator() *Calculator {
    return &Calculator{}
}

func (c *Calculator) Add(a, b int) int {
    return a + b
}
// Test passes

// 3. REFACTOR - Improve if needed
// (In this case, code is already clean)
```

---

## Best Practices

### AAA Pattern

```go
func TestSomething(t *testing.T) {
    // Arrange - setup test data and dependencies
    repo := NewFakeRepository()
    service := NewService(repo)
    input := CreateInput{Name: "test"}

    // Act - execute the code under test
    result, err := service.Create(context.Background(), input)

    // Assert - verify the results
    assert.NoError(t, err)
    assert.Equal(t, "test", result.Name)
}
```

### Test Isolation

```go
func TestWithIsolation(t *testing.T) {
    // Each test gets its own setup
    t.Run("scenario 1", func(t *testing.T) {
        db := setupFreshDB(t)
        // test...
    })

    t.Run("scenario 2", func(t *testing.T) {
        db := setupFreshDB(t)
        // test with clean state...
    })
}
```

### Meaningful Test Names

```go
// Good - describes behavior
func TestOrder_Submit_FailsWhenOrderIsEmpty(t *testing.T)
func TestUser_CanLogin_ReturnsFalseWhenBanned(t *testing.T)

// Bad - not descriptive
func TestOrder_Submit(t *testing.T)
func TestUser1(t *testing.T)
```

---

## См. также

- [Tooling & Testing в Go](../00-go/07-tooling-testing.md) — инструменты и тестирование в Go
- [Testing в Java](../01-java/12-testing.md) — тестирование в Java

---

## На интервью

### Типичные вопросы

1. **Test Pyramid — объясните**
   - Unit → Integration → E2E
   - Количество, скорость, стоимость

2. **Mock vs Fake vs Stub?**
   - Mock: expectations + verification
   - Fake: working simplified implementation
   - Stub: fixed responses

3. **Как тестировать external services?**
   - Mocks для unit tests
   - Contract tests (Pact)
   - Testcontainers для integration

4. **TDD — как и когда?**
   - Red → Green → Refactor
   - Полезно для complex logic
   - Не всегда практично для UI/exploratory

---

[← Назад к списку тем](README.md)
