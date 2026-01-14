# 07. API Design

[← Назад к списку тем](README.md)

---

## REST API

### Richardson Maturity Model

```
Level 0: The Swamp of POX
  POST /api
  Body: { "action": "getUser", "userId": 123 }

Level 1: Resources
  POST /users/123/getDetails
  POST /users/123/update

Level 2: HTTP Verbs
  GET    /users/123
  PUT    /users/123
  DELETE /users/123
  POST   /users

Level 3: Hypermedia (HATEOAS)
  GET /users/123
  {
    "id": 123,
    "name": "John",
    "_links": {
      "self": { "href": "/users/123" },
      "orders": { "href": "/users/123/orders" },
      "update": { "href": "/users/123", "method": "PUT" }
    }
  }
```

### Resource Naming

```
Хорошо:
GET    /users                    # список пользователей
GET    /users/123                # конкретный пользователь
POST   /users                    # создать пользователя
PUT    /users/123                # обновить пользователя
DELETE /users/123                # удалить пользователя
GET    /users/123/orders         # заказы пользователя
POST   /users/123/orders         # создать заказ для пользователя

Плохо:
GET    /getUsers
POST   /createUser
GET    /user/123/getOrders
POST   /user/123/createNewOrder
```

### Query Parameters

```
# Pagination
GET /users?page=2&per_page=20
GET /users?offset=20&limit=20
GET /users?cursor=eyJpZCI6MTIzfQ==

# Filtering
GET /orders?status=pending&created_after=2024-01-01

# Sorting
GET /users?sort=created_at:desc,name:asc

# Field selection (sparse fieldsets)
GET /users/123?fields=id,name,email

# Expansion (include related)
GET /orders/123?include=customer,items
```

### HTTP Status Codes

```go
// Success
200 OK              // GET, PUT успех
201 Created         // POST успех, возвращает созданный ресурс
204 No Content      // DELETE успех, нет тела

// Client Errors
400 Bad Request     // Невалидный запрос
401 Unauthorized    // Не аутентифицирован
403 Forbidden       // Нет прав
404 Not Found       // Ресурс не найден
409 Conflict        // Конфликт (duplicate, optimistic lock)
422 Unprocessable   // Валидация не прошла
429 Too Many Requests // Rate limit

// Server Errors
500 Internal Error  // Неожиданная ошибка
502 Bad Gateway     // Upstream сервис недоступен
503 Service Unavailable // Временно недоступен
504 Gateway Timeout // Timeout от upstream
```

### Error Response Format

```go
// Структурированный формат ошибок
type ErrorResponse struct {
    Error   ErrorDetails   `json:"error"`
}

type ErrorDetails struct {
    Code    string         `json:"code"`              // "VALIDATION_ERROR"
    Message string         `json:"message"`           // Human-readable
    Details []FieldError   `json:"details,omitempty"` // Детали по полям
    TraceID string         `json:"trace_id"`          // Для отладки
}

type FieldError struct {
    Field   string `json:"field"`
    Code    string `json:"code"`
    Message string `json:"message"`
}

// Пример
{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Request validation failed",
        "details": [
            {
                "field": "email",
                "code": "INVALID_FORMAT",
                "message": "Email must be a valid email address"
            },
            {
                "field": "age",
                "code": "OUT_OF_RANGE",
                "message": "Age must be between 18 and 120"
            }
        ],
        "trace_id": "abc123"
    }
}
```

---

## API Versioning

### URL Versioning

```
GET /api/v1/users/123
GET /api/v2/users/123

Pros: Явно, просто
Cons: Меняет URL, кеширование сложнее
```

### Header Versioning

```
GET /api/users/123
Accept: application/vnd.myapi.v2+json

Pros: Чистые URL
Cons: Сложнее тестировать, менее явно
```

### Query Parameter

```
GET /api/users/123?version=2

Pros: Просто
Cons: Не очень RESTful
```

### Backward Compatibility

```go
// v1 response
type UserV1 struct {
    ID   int    `json:"id"`
    Name string `json:"name"`  // full name
}

// v2 response - breaking change: split name
type UserV2 struct {
    ID        int    `json:"id"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
}

// Стратегия: поддерживать оба
func (h *Handler) GetUser(w http.ResponseWriter, r *http.Request) {
    user := h.service.GetUser(r.Context(), userID)

    version := r.Header.Get("API-Version")
    switch version {
    case "2", "":  // v2 is default
        json.NewEncoder(w).Encode(UserV2{
            ID:        user.ID,
            FirstName: user.FirstName,
            LastName:  user.LastName,
        })
    case "1":
        json.NewEncoder(w).Encode(UserV1{
            ID:   user.ID,
            Name: user.FirstName + " " + user.LastName,
        })
    }
}
```

---

## gRPC

### Protobuf Definition

```protobuf
syntax = "proto3";

package user.v1;

option go_package = "myapp/gen/user/v1;userv1";

service UserService {
    // Unary RPC
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);

    // Server streaming
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client streaming
    rpc UploadUsers(stream User) returns (UploadUsersResponse);

    // Bidirectional streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetUserRequest {
    string user_id = 1;
}

message GetUserResponse {
    User user = 1;
}

message User {
    string id = 1;
    string email = 2;
    string name = 3;
    UserStatus status = 4;
    google.protobuf.Timestamp created_at = 5;
}

enum UserStatus {
    USER_STATUS_UNSPECIFIED = 0;
    USER_STATUS_ACTIVE = 1;
    USER_STATUS_INACTIVE = 2;
}
```

### Go Implementation

```go
// Server
type userServer struct {
    userv1.UnimplementedUserServiceServer
    service UserService
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    user, err := s.service.GetUser(ctx, req.UserId)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, status.Error(codes.NotFound, "user not found")
        }
        return nil, status.Error(codes.Internal, "internal error")
    }

    return &userv1.GetUserResponse{
        User: toProto(user),
    }, nil
}

// Client
func (c *userClient) GetUser(ctx context.Context, userID string) (*User, error) {
    resp, err := c.client.GetUser(ctx, &userv1.GetUserRequest{
        UserId: userID,
    })
    if err != nil {
        st, ok := status.FromError(err)
        if ok && st.Code() == codes.NotFound {
            return nil, ErrNotFound
        }
        return nil, err
    }

    return fromProto(resp.User), nil
}
```

### gRPC vs REST

| Aspect | REST | gRPC |
|--------|------|------|
| Protocol | HTTP/1.1 (JSON) | HTTP/2 (Protobuf) |
| Performance | Good | Better (binary, multiplexing) |
| Streaming | Limited | Native support |
| Browser | Native | Requires grpc-web |
| Tooling | Widespread | Growing |
| Schema | OpenAPI (optional) | Required (proto) |

---

## GraphQL

### Schema

```graphql
type Query {
    user(id: ID!): User
    users(filter: UserFilter, pagination: Pagination): UserConnection!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
}

type User {
    id: ID!
    email: String!
    name: String!
    orders(first: Int, after: String): OrderConnection!
}

type Order {
    id: ID!
    total: Money!
    items: [OrderItem!]!
}

input CreateUserInput {
    email: String!
    name: String!
}

input UserFilter {
    status: UserStatus
    createdAfter: DateTime
}
```

### Resolver

```go
type Resolver struct {
    userService  UserService
    orderService OrderService
}

func (r *Resolver) User(ctx context.Context, args struct{ ID string }) (*UserResolver, error) {
    user, err := r.userService.GetUser(ctx, args.ID)
    if err != nil {
        return nil, err
    }
    return &UserResolver{user: user, orderService: r.orderService}, nil
}

type UserResolver struct {
    user         *User
    orderService OrderService
}

func (r *UserResolver) Orders(ctx context.Context, args struct{ First *int; After *string }) (*OrderConnectionResolver, error) {
    // N+1 problem - use DataLoader
    orders, err := r.orderService.GetUserOrders(ctx, r.user.ID, args.First, args.After)
    if err != nil {
        return nil, err
    }
    return &OrderConnectionResolver{orders: orders}, nil
}
```

### DataLoader (N+1 problem)

```go
// Без DataLoader: N+1 запросов
// Query { users { orders { ... } } }
// 1 запрос на users + N запросов на orders

// С DataLoader: 2 запроса
type OrderLoader struct {
    orderService OrderService
}

func (l *OrderLoader) Load(ctx context.Context, userIDs []string) ([][]*Order, []error) {
    // Batch load всех orders для всех userIDs
    ordersMap, err := l.orderService.GetOrdersByUsers(ctx, userIDs)
    if err != nil {
        errors := make([]error, len(userIDs))
        for i := range errors {
            errors[i] = err
        }
        return nil, errors
    }

    results := make([][]*Order, len(userIDs))
    for i, userID := range userIDs {
        results[i] = ordersMap[userID]
    }
    return results, nil
}
```

---

## OpenAPI / Swagger

```yaml
openapi: 3.0.3
info:
  title: User API
  version: 1.0.0

paths:
  /users:
    get:
      summary: List users
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [active, inactive]
        - name: page
          in: query
          schema:
            type: integer
            default: 1
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  users:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'

components:
  schemas:
    User:
      type: object
      required: [id, email, name]
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 1
          maxLength: 100

    CreateUserRequest:
      type: object
      required: [email, name]
      properties:
        email:
          type: string
          format: email
        name:
          type: string

  responses:
    BadRequest:
      description: Bad request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

---

## Best Practices

### Idempotency

```go
// Client sends idempotency key
// POST /orders
// Idempotency-Key: unique-client-generated-key

func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    idempotencyKey := r.Header.Get("Idempotency-Key")
    if idempotencyKey == "" {
        http.Error(w, "Idempotency-Key required", http.StatusBadRequest)
        return
    }

    // Check if already processed
    if cached, ok := h.cache.Get(idempotencyKey); ok {
        json.NewEncoder(w).Encode(cached)
        return
    }

    // Process
    order, err := h.service.CreateOrder(r.Context(), req)
    if err != nil {
        // ...
    }

    // Cache response
    h.cache.Set(idempotencyKey, order, 24*time.Hour)

    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(order)
}
```

### Rate Limiting Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704067200

HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

---

## На интервью

### Типичные вопросы

1. **REST best practices?**
   - Resource naming, HTTP verbs, status codes
   - Versioning strategies
   - Error handling

2. **gRPC vs REST?**
   - Performance, streaming, browser support
   - When to use each

3. **GraphQL pros/cons?**
   - Flexibility vs complexity
   - N+1 problem и DataLoader

4. **API versioning strategy?**
   - URL vs Header vs Query
   - Backward compatibility

---

[← Назад к списку тем](README.md)
