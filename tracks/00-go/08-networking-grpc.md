# 08 — Networking & gRPC

Вопросы и ответы про построение сетевых сервисов на Go: устройство `net/http`, клиенты, сериализацию данных, gRPC, protobuf и наблюдаемость.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [07-tooling-testing](./07-tooling-testing.md) · Следующая тема: [09-performance-profiling](./09-performance-profiling.md)

---

## Вопросы и разборы

### 1. Как работает `net/http` сервер внутри?

**Зачем спрашивают.**
Понимание внутреннего устройства HTTP сервера критично для настройки производительности и отладки. Интервьюер проверяет знание модели concurrency и жизненного цикла соединений.

**Короткий ответ.**
HTTP сервер принимает соединения на `net.Listener`. Каждое соединение обслуживается отдельной горутиной. Keep-alive позволяет обрабатывать несколько запросов на одном соединении.

**Детальный разбор.**

**Архитектура net/http сервера:**
```
┌─────────────────────────────────────────────────────────────┐
│                     http.Server                              │
├─────────────────────────────────────────────────────────────┤
│  net.Listener.Accept() ─────┬─────────────────────────────  │
│                             │                                │
│            ┌────────────────┼────────────────┐               │
│            ▼                ▼                ▼               │
│      goroutine 1      goroutine 2      goroutine 3          │
│      (conn 1)         (conn 2)         (conn 3)             │
│            │                │                │               │
│      ┌─────┴─────┐    ┌─────┴─────┐    ┌─────┴─────┐        │
│      │ Request 1 │    │ Request 1 │    │ Request 1 │        │
│      │ Request 2 │    │ Request 2 │    └───────────┘        │
│      │ Request 3 │    └───────────┘                         │
│      └───────────┘                                           │
│                           HTTP/1.1 Keep-Alive                │
└─────────────────────────────────────────────────────────────┘
```

**Жизненный цикл запроса:**
```
1. Listener.Accept() — принять TCP соединение
2. go conn.serve() — горутина на соединение
3. conn.readRequest() — парсинг HTTP запроса
4. Handler.ServeHTTP() — вызов обработчика
5. Response.Write() — отправка ответа
6. Повторить 3-5 для keep-alive или закрыть
```

**Ключевые настройки:**
```
┌───────────────────────┬─────────────────────────────────────┐
│ Параметр              │ Назначение                          │
├───────────────────────┼─────────────────────────────────────┤
│ ReadTimeout           │ Время на чтение запроса             │
│ WriteTimeout          │ Время на запись ответа              │
│ IdleTimeout           │ Время keep-alive соединения         │
│ ReadHeaderTimeout     │ Время на чтение заголовков          │
│ MaxHeaderBytes        │ Лимит размера заголовков            │
└───────────────────────┴─────────────────────────────────────┘
```

**Пример.**
```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       120 * time.Second,
    ReadHeaderTimeout: 2 * time.Second,
    MaxHeaderBytes:    1 << 20, // 1MB
}

// Graceful shutdown
go func() {
    if err := srv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatalf("HTTP server error: %v", err)
    }
}()

// Ожидание сигнала
<-ctx.Done()

// Graceful shutdown с таймаутом
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Printf("HTTP shutdown error: %v", err)
}
```

**Типичные ошибки.**
1. **Не устанавливать таймауты** — медленные клиенты держат горутины вечно
2. **WriteTimeout < время обработки** — ответ обрезается
3. **Забывать про graceful shutdown** — теряются in-flight запросы
4. **Не ограничивать body size** — DoS через большие запросы

**На интервью.**
Объясни, почему одна горутина на соединение, а не на запрос. Расскажи про keep-alive и HTTP/2 мультиплексирование. Покажи, как таймауты защищают от slowloris атак.

*Частые follow-up вопросы:*
- Как HTTP/2 меняет модель горутин?
- Почему важен ReadHeaderTimeout отдельно от ReadTimeout?
- Как отследить количество активных соединений?

---

### 2. Как реализовать middleware/chain-of-responsibility?

**Зачем спрашивают.**
Middleware — стандартный паттерн для cross-cutting concerns (логирование, авторизация, метрики). Интервьюер проверяет понимание композиции и порядка выполнения.

**Короткий ответ.**
Middleware — функция, принимающая `http.Handler` и возвращающая `http.Handler`. Цепочка строится вложенными вызовами или с помощью библиотек (chi, alice).

**Детальный разбор.**

**Структура middleware:**
```
┌─────────────────────────────────────────────────────────────┐
│ Request Flow:                                                │
│                                                              │
│   Request → [Logging] → [Auth] → [RateLimit] → [Handler]    │
│                                                              │
│   Response ← [Logging] ← [Auth] ← [RateLimit] ← [Handler]   │
│                                                              │
│ Порядок выполнения:                                         │
│   Logging.Before → Auth.Before → RateLimit.Before → Handler │
│   Handler → RateLimit.After → Auth.After → Logging.After    │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**
```go
// Базовый middleware
func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Обёртка для захвата status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(wrapped, r)

        log.Printf("%s %s %d %v",
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            time.Since(start),
        )
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// Recovery middleware
func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Auth middleware с context
func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        user, err := validateToken(token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        // Добавляем user в context
        ctx := context.WithValue(r.Context(), userKey, user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**Композиция middleware:**
```go
// Ручная композиция
handler := Logging(Recovery(Auth(businessHandler)))

// С помощью функции Chain
func Chain(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

handler := Chain(businessHandler, Logging, Recovery, Auth)

// С chi router
r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)
r.Use(Auth)
r.Get("/api/users", usersHandler)
```

**Типичные ошибки.**
1. **Неправильный порядок** — Recovery должен быть первым (внешним)
2. **Не вызывать next.ServeHTTP** — цепочка прерывается
3. **Модифицировать response после next** — заголовки уже отправлены
4. **Хранить состояние в middleware** — race condition

**На интервью.**
Покажи порядок выполнения before/after. Объясни, почему Recovery должен быть первым. Расскажи, как передавать данные через context между middleware.

*Частые follow-up вопросы:*
- Как получить status code ответа в middleware?
- Как сделать middleware конфигурируемым?
- Чем отличается middleware от interceptor в gRPC?

---

### 3. Как правильно настраивать `http.Client`?

**Зачем спрашивают.**
Неправильная настройка клиента — частый источник проблем (утечки соединений, медленные ответы, отсутствие таймаутов). Интервьюер проверяет понимание connection pooling и resource management.

**Короткий ответ.**
Клиент создаётся один раз и переиспользуется. Всегда устанавливать таймауты. Всегда закрывать `resp.Body`. Настраивать `Transport` для connection pooling.

**Детальный разбор.**

**Структура http.Client:**
```
┌─────────────────────────────────────────────────────────────┐
│                      http.Client                             │
├─────────────────────────────────────────────────────────────┤
│  Timeout          — общий таймаут запроса                   │
│  Transport        — управление соединениями                 │
│  CheckRedirect    — политика редиректов                     │
│  Jar              — cookie storage                          │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    http.Transport                            │
├─────────────────────────────────────────────────────────────┤
│  MaxIdleConns          — макс. idle соединений всего        │
│  MaxIdleConnsPerHost   — макс. idle на хост                 │
│  MaxConnsPerHost       — макс. соединений на хост           │
│  IdleConnTimeout       — время жизни idle соединения        │
│  TLSHandshakeTimeout   — таймаут TLS handshake              │
│  ResponseHeaderTimeout — таймаут на заголовки ответа        │
│  DisableKeepAlives     — отключить keep-alive               │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**
```go
// Создаём клиент один раз (обычно на уровне пакета или DI)
var httpClient = &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        MaxConnsPerHost:     100,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
        DisableCompression:  false,

        // Настройка DNS и TCP
        DialContext: (&net.Dialer{
            Timeout:   30 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
    },
}

// Правильное использование
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    req.Header.Set("Accept", "application/json")

    resp, err := httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("do request: %w", err)
    }
    defer resp.Body.Close() // КРИТИЧНО: всегда закрывать!

    // Проверка статуса
    if resp.StatusCode != http.StatusOK {
        // Даже при ошибке читаем body для переиспользования соединения
        io.Copy(io.Discard, resp.Body)
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }

    // Лимит на размер ответа
    body, err := io.ReadAll(io.LimitReader(resp.Body, 10<<20)) // 10MB max
    if err != nil {
        return nil, fmt.Errorf("read body: %w", err)
    }

    return body, nil
}
```

**Retry с exponential backoff:**
```go
func fetchWithRetry(ctx context.Context, url string) ([]byte, error) {
    var lastErr error

    for attempt := 0; attempt < 3; attempt++ {
        if attempt > 0 {
            // Exponential backoff: 100ms, 200ms, 400ms
            backoff := time.Duration(100<<attempt) * time.Millisecond
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }

        data, err := fetchData(ctx, url)
        if err == nil {
            return data, nil
        }

        lastErr = err

        // Retry только для retriable ошибок
        if !isRetriable(err) {
            return nil, err
        }
    }

    return nil, fmt.Errorf("all retries failed: %w", lastErr)
}

func isRetriable(err error) bool {
    // Сетевые ошибки, таймауты, 5xx
    var netErr net.Error
    if errors.As(err, &netErr) && netErr.Timeout() {
        return true
    }
    // Добавить проверку HTTP статусов
    return false
}
```

**Типичные ошибки.**
1. **Создавать клиент на каждый запрос** — утечка соединений
2. **Не закрывать resp.Body** — соединение не возвращается в пул
3. **http.DefaultClient без таймаутов** — зависшие запросы
4. **Не читать body при ошибке** — соединение не переиспользуется

**На интервью.**
Объясни, почему важно закрывать body даже при ошибке. Покажи, как connection pool влияет на производительность. Расскажи про разницу между Client.Timeout и Transport таймаутами.

*Частые follow-up вопросы:*
- Что происходит при connection refused?
- Как реализовать circuit breaker?
- Когда использовать DisableKeepAlives?

---

### 4. Что такое сериализация и зачем она нужна?

**Зачем спрашивают.**
Сериализация — ключевой компонент сетевых сервисов. Интервьюер проверяет понимание trade-offs разных форматов и типичных проблем (версионирование, производительность).

**Короткий ответ.**
Сериализация — преобразование структур данных в байты для передачи/хранения. JSON — человекочитаемый, универсальный. Protobuf — компактный, быстрый, со схемой. Gob — Go-специфичный, для внутренних сервисов.

**Детальный разбор.**

**Сравнение форматов:**
```
┌─────────────┬────────────┬──────────────┬───────────────────┐
│ Формат      │ Размер     │ Скорость     │ Особенности       │
├─────────────┼────────────┼──────────────┼───────────────────┤
│ JSON        │ Большой    │ Медленно     │ Человекочитаемый  │
│ Protobuf    │ Малый      │ Быстро       │ Схема, версии     │
│ MessagePack │ Средний    │ Быстро       │ Как JSON, но bin  │
│ Gob         │ Средний    │ Быстро       │ Только Go         │
│ FlatBuffers │ Малый      │ Очень быстро │ Zero-copy         │
└─────────────┴────────────┴──────────────┴───────────────────┘
```

**Пример.**

**JSON — стандарт для REST API:**
```go
type User struct {
    ID        int64     `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"`
    CreatedAt time.Time `json:"created_at"`
}

// Сериализация
user := User{ID: 1, Name: "Ada", CreatedAt: time.Now()}
data, err := json.Marshal(user)
// {"id":1,"name":"Ada","created_at":"2024-01-15T10:30:00Z"}

// Десериализация
var decoded User
err = json.Unmarshal(data, &decoded)

// Streaming для больших данных
encoder := json.NewEncoder(w)
encoder.SetIndent("", "  ")
encoder.Encode(user)
```

**Protobuf — для gRPC и высоконагруженных систем:**
```protobuf
// user.proto
syntax = "proto3";

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    google.protobuf.Timestamp created_at = 4;
}
```

```go
// Сгенерированный код
user := &pb.User{
    Id:        1,
    Name:      "Ada",
    CreatedAt: timestamppb.Now(),
}

data, err := proto.Marshal(user)
// Бинарный формат: ~20 байт vs ~80 байт JSON

var decoded pb.User
err = proto.Unmarshal(data, &decoded)
```

**Gob — для внутренних Go сервисов:**
```go
type ComplexStruct struct {
    Data      map[string]any
    Functions []func() // НЕ сериализуется!
}

var buf bytes.Buffer
enc := gob.NewEncoder(&buf)
err := enc.Encode(user)

dec := gob.NewDecoder(&buf)
var decoded User
err = dec.Decode(&decoded)
```

**Типичные ошибки.**
1. **JSON для бинарных данных** — base64 увеличивает размер на 33%
2. **Нет валидации при десериализации** — принимаем невалидные данные
3. **time.Time в JSON** — неявный формат, часовые пояса
4. **Proto без google.protobuf.Timestamp** — свои форматы времени

**На интервью.**
Сравни JSON и Protobuf для конкретного use case. Объясни, как schema evolution работает в Protobuf (добавление/удаление полей). Расскажи про zero-copy десериализацию.

*Частые follow-up вопросы:*
- Как обрабатывать unknown fields в Protobuf?
- Когда MessagePack лучше JSON?
- Как версионировать JSON API?

---

### 5. Как устроен gRPC в Go?

**Зачем спрашивают.**
gRPC — стандарт для микросервисов. Интервьюер проверяет понимание протокола, типов RPC и паттернов (interceptors, streaming).

**Короткий ответ.**
gRPC использует HTTP/2 и Protobuf. Сервер регистрирует сервисы, клиент вызывает методы как локальные функции. Interceptors добавляют cross-cutting логику. Streaming позволяет передавать потоки данных.

**Детальный разбор.**

**Типы gRPC вызовов:**
```
┌─────────────────┬─────────────────────────────────────────┐
│ Тип             │ Описание                                │
├─────────────────┼─────────────────────────────────────────┤
│ Unary           │ Один запрос → один ответ                │
│ Server Stream   │ Один запрос → поток ответов             │
│ Client Stream   │ Поток запросов → один ответ             │
│ Bidirectional   │ Поток запросов ↔ поток ответов          │
└─────────────────┴─────────────────────────────────────────┘
```

**Архитектура:**
```
┌─────────────┐                      ┌─────────────┐
│   Client    │                      │   Server    │
├─────────────┤                      ├─────────────┤
│ Stub (gen)  │──────HTTP/2─────────▶│ Service     │
│ Interceptor │◀─────────────────────│ Interceptor │
│ Connection  │                      │ Listener    │
└─────────────┘                      └─────────────┘
```

**Пример.**

**Определение сервиса:**
```protobuf
syntax = "proto3";
package todo;

service TodoService {
    // Unary
    rpc CreateTodo(CreateTodoRequest) returns (Todo);

    // Server streaming
    rpc ListTodos(ListTodosRequest) returns (stream Todo);

    // Client streaming
    rpc BatchCreate(stream CreateTodoRequest) returns (BatchCreateResponse);

    // Bidirectional
    rpc Watch(stream WatchRequest) returns (stream Todo);
}

message Todo {
    string id = 1;
    string title = 2;
    bool completed = 3;
}
```

**Сервер:**
```go
type todoServer struct {
    pb.UnimplementedTodoServiceServer
    store *TodoStore
}

func (s *todoServer) CreateTodo(ctx context.Context, req *pb.CreateTodoRequest) (*pb.Todo, error) {
    todo, err := s.store.Create(ctx, req.Title)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "create failed: %v", err)
    }
    return todo, nil
}

func (s *todoServer) ListTodos(req *pb.ListTodosRequest, stream pb.TodoService_ListTodosServer) error {
    todos, err := s.store.List(stream.Context())
    if err != nil {
        return status.Errorf(codes.Internal, "list failed: %v", err)
    }

    for _, todo := range todos {
        if err := stream.Send(todo); err != nil {
            return err
        }
    }
    return nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    s := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            loggingInterceptor,
            authInterceptor,
        ),
    )

    pb.RegisterTodoServiceServer(s, &todoServer{store: NewStore()})

    // Graceful shutdown
    go func() {
        if err := s.Serve(lis); err != nil {
            log.Fatalf("failed to serve: %v", err)
        }
    }()

    <-ctx.Done()
    s.GracefulStop()
}
```

**Клиент (grpc.NewClient — рекомендуемый с gRPC 1.63+):**
```go
func main() {
    // Новый API: grpc.NewClient (рекомендуется)
    conn, err := grpc.NewClient(
        "localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithChainUnaryInterceptor(
            retryInterceptor,
            loggingInterceptor,
        ),
    )
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewTodoServiceClient(conn)

    // Unary call
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    todo, err := client.CreateTodo(ctx, &pb.CreateTodoRequest{Title: "Learn gRPC"})
    if err != nil {
        st, ok := status.FromError(err)
        if ok {
            log.Printf("gRPC error: code=%s message=%s", st.Code(), st.Message())
        }
        return
    }

    fmt.Printf("Created: %+v\n", todo)
}
```

**Interceptor:**
```go
func loggingInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (any, error) {
    start := time.Now()

    resp, err := handler(ctx, req)

    log.Printf("method=%s duration=%v error=%v",
        info.FullMethod,
        time.Since(start),
        err,
    )

    return resp, err
}
```

**Типичные ошибки.**
1. **grpc.Dial вместо grpc.NewClient** — Dial deprecated с gRPC 1.63
2. **Не проверять status.FromError** — теряется информация об ошибке
3. **Нет таймаутов на клиенте** — зависшие вызовы
4. **Игнорировать context в streaming** — не отменяются при ctx.Done()

**На интервью.**
Объясни разницу между Unary и Streaming. Покажи, как interceptors похожи на middleware. Расскажи про gRPC status codes и когда какой использовать.

*Частые follow-up вопросы:*
- Как реализовать retry в gRPC?
- Чем gRPC отличается от REST?
- Как работает load balancing в gRPC?

---

### 6. Как проектировать protobuf контракты и версии?

**Зачем спрашивают.**
Правильное проектирование API критично для backward/forward compatibility. Интервьюер проверяет понимание schema evolution и best practices.

**Короткий ответ.**
Номера полей уникальны и неизменны. Новые поля добавляются с новыми номерами. Удалённые поля помечаются `reserved`. Используй `oneof` для взаимоисключающих вариантов.

**Детальный разбор.**

**Правила совместимости:**
```
┌─────────────────────────────────────────────────────────────┐
│ Безопасно:                                                  │
│ • Добавить новое поле с новым номером                       │
│ • Удалить поле (старые клиенты проигнорируют)              │
│ • Переименовать поле (номер остаётся)                       │
├─────────────────────────────────────────────────────────────┤
│ Опасно / Запрещено:                                         │
│ • Изменить номер поля                                       │
│ • Изменить тип поля несовместимо                           │
│ • Переиспользовать номер удалённого поля                   │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Эволюция схемы:**
```protobuf
// Версия 1
message User {
    int64 id = 1;
    string name = 2;
}

// Версия 2: добавили email, удалили name
message User {
    int64 id = 1;
    reserved 2;           // name удалён, номер зарезервирован
    reserved "name";      // имя зарезервировано
    string email = 3;     // новое поле с новым номером
    string first_name = 4;
    string last_name = 5;
}
```

**Oneof для вариантов:**
```protobuf
message Notification {
    string id = 1;

    oneof content {
        EmailNotification email = 2;
        PushNotification push = 3;
        SMSNotification sms = 4;
    }

    google.protobuf.Timestamp created_at = 5;
}

message EmailNotification {
    string subject = 1;
    string body = 2;
}
```

**Wrapper types для nullable:**
```protobuf
import "google/protobuf/wrappers.proto";

message UpdateUserRequest {
    int64 id = 1;
    // Если не указан — не обновлять
    google.protobuf.StringValue name = 2;
    google.protobuf.Int32Value age = 3;
}
```

**Enum с UNSPECIFIED:**
```protobuf
enum Status {
    STATUS_UNSPECIFIED = 0;  // Всегда первый = 0
    STATUS_PENDING = 1;
    STATUS_ACTIVE = 2;
    STATUS_COMPLETED = 3;
}
```

**Field masks для partial updates:**
```protobuf
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
    User user = 1;
    google.protobuf.FieldMask update_mask = 2;
}

// Клиент указывает какие поля обновить:
// update_mask: { paths: ["name", "email"] }
```

**Типичные ошибки.**
1. **Переиспользование номеров** — несовместимость версий
2. **Enum без _UNSPECIFIED = 0** — 0 всегда default
3. **Nullable поля без wrappers** — не отличить "не указано" от значения по умолчанию
4. **Нет документации полей** — непонятно использование

**На интервью.**
Объясни, как добавить поле без breaking changes. Покажи разницу между optional и required (в proto3 всё optional). Расскажи про Google AIP guidelines.

*Частые follow-up вопросы:*
- Как версионировать gRPC сервисы?
- Когда использовать oneof vs отдельные поля?
- Как тестировать backward compatibility?

---

### 7. Есть ли для Go хороший ORM?

**Зачем спрашивают.**
Выбор инструмента работы с БД — важное архитектурное решение. Интервьюер оценивает знание опций и умение аргументировать выбор.

**Короткий ответ.**
Нет универсального "лучшего". `sqlc` — type-safe SQL через генерацию. `ent` — ORM с code generation. `gorm` — традиционный ORM. `pgx` + `sqlx` — низкоуровневый контроль.

**Детальный разбор.**

**Сравнение подходов:**
```
┌──────────────┬────────────────┬─────────────────────────────┐
│ Инструмент   │ Подход         │ Trade-offs                  │
├──────────────┼────────────────┼─────────────────────────────┤
│ sqlc         │ SQL → Go code  │ Type-safe, нет N+1, verbose │
│ ent          │ Schema → Code  │ Type-safe, migrations, ORM  │
│ gorm         │ Struct tags    │ Удобно, магия, N+1          │
│ sqlx         │ Raw SQL + scan │ Контроль, больше кода       │
│ pgx          │ PostgreSQL     │ Низкий уровень, performance │
└──────────────┴────────────────┴─────────────────────────────┘
```

**Пример.**

**sqlc — SQL первично:**
```sql
-- schema.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- queries.sql
-- name: GetUser :one
SELECT * FROM users WHERE id = $1;

-- name: ListUsers :many
SELECT * FROM users ORDER BY created_at DESC LIMIT $1;

-- name: CreateUser :one
INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *;
```

```go
// Сгенерированный код
func (q *Queries) CreateUser(ctx context.Context, arg CreateUserParams) (User, error)
func (q *Queries) GetUser(ctx context.Context, id int32) (User, error)
func (q *Queries) ListUsers(ctx context.Context, limit int32) ([]User, error)

// Использование
user, err := queries.CreateUser(ctx, db.CreateUserParams{
    Name:  "Ada",
    Email: "ada@go.dev",
})
```

**ent — Schema as Code:**
```go
// ent/schema/user.go
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name").NotEmpty(),
        field.String("email").Unique(),
        field.Time("created_at").Default(time.Now),
    }
}

func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("posts", Post.Type),
    }
}

// Использование
user, err := client.User.
    Create().
    SetName("Ada").
    SetEmail("ada@go.dev").
    Save(ctx)

users, err := client.User.
    Query().
    Where(user.EmailContains("@go.dev")).
    WithPosts().  // eager loading
    All(ctx)
```

**gorm — традиционный ORM:**
```go
type User struct {
    ID        uint   `gorm:"primaryKey"`
    Name      string `gorm:"not null"`
    Email     string `gorm:"uniqueIndex"`
    Posts     []Post
    CreatedAt time.Time
}

// Использование
var user User
db.First(&user, 1)  // SELECT * FROM users WHERE id = 1

db.Preload("Posts").Find(&users)  // eager loading

db.Create(&User{Name: "Ada", Email: "ada@go.dev"})
```

**Типичные ошибки.**
1. **N+1 запросы** — забывать про preload/eager loading
2. **Нет connection pooling** — каждый запрос новое соединение
3. **SQL injection с raw queries** — использовать placeholders
4. **Нет транзакций** — неатомарные операции

**На интервью.**
Объясни trade-off между type safety (sqlc) и удобством (gorm). Покажи, как избежать N+1 в выбранном инструменте. Расскажи про connection pooling.

*Частые follow-up вопросы:*
- Как тестировать код с БД?
- Как делать миграции?
- Когда raw SQL лучше ORM?

---

### 8. Как добавить наблюдаемость и управление нагрузкой?

**Зачем спрашивают.**
Observability (метрики, логи, трейсы) и управление нагрузкой (rate limiting, circuit breaker) критичны для production. Интервьюер проверяет практический опыт эксплуатации.

**Короткий ответ.**
Три столпа observability: метрики (Prometheus), логи (structured logging), трейсы (OpenTelemetry). Rate limiting защищает от перегрузки. Circuit breaker предотвращает cascade failures.

**Детальный разбор.**

**Три столпа observability:**
```
┌─────────────┬────────────────────────────────────────────┐
│ Metrics     │ Числовые данные во времени (latency, RPS)  │
│             │ → Prometheus, Grafana                       │
├─────────────┼────────────────────────────────────────────┤
│ Logs        │ Структурированные события                  │
│             │ → slog, zap → ELK, Loki                    │
├─────────────┼────────────────────────────────────────────┤
│ Traces      │ Распределённые запросы                     │
│             │ → OpenTelemetry → Jaeger, Tempo            │
└─────────────┴────────────────────────────────────────────┘
```

**Пример.**

**Prometheus метрики:**
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration)
}

// Middleware для метрик
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(wrapped, r)

        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            strconv.Itoa(wrapped.statusCode),
        ).Inc()

        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(time.Since(start).Seconds())
    })
}

// Endpoint для Prometheus
mux.Handle("/metrics", promhttp.Handler())
```

**Structured logging с slog:**
```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// Middleware для логирования
func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            requestID := uuid.New().String()

            // Добавляем request_id в context
            ctx := context.WithValue(r.Context(), requestIDKey, requestID)
            r = r.WithContext(ctx)

            wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
            next.ServeHTTP(wrapped, r)

            logger.Info("request completed",
                "request_id", requestID,
                "method", r.Method,
                "path", r.URL.Path,
                "status", wrapped.statusCode,
                "duration_ms", time.Since(start).Milliseconds(),
                "user_agent", r.UserAgent(),
            )
        })
    }
}
```

**OpenTelemetry трейсинг:**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("my-service")

func fetchUser(ctx context.Context, id string) (*User, error) {
    ctx, span := tracer.Start(ctx, "fetchUser",
        trace.WithAttributes(attribute.String("user.id", id)),
    )
    defer span.End()

    user, err := db.GetUser(ctx, id)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    return user, nil
}
```

**Health checks:**
```go
// Liveness — приложение запущено
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
})

// Readiness — готово принимать трафик
mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
    if err := db.Ping(r.Context()); err != nil {
        http.Error(w, "database unavailable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

**Типичные ошибки.**
1. **High cardinality labels** — path с ID создаёт миллионы серий
2. **Нет correlation** — request_id не передаётся между сервисами
3. **Логирование sensitive data** — пароли, токены в логах
4. **Нет rate limiting на /metrics** — DoS через scraping

**На интервью.**
Объясни разницу между metrics и traces. Покажи, как связать логи и трейсы через request_id. Расскажи про cardinality проблему в Prometheus.

*Частые follow-up вопросы:*
- Как выбрать buckets для histogram?
- Когда использовать Counter vs Gauge?
- Как сэмплировать трейсы в high-load системах?

---

### 9. Как работать с WebSocket в Go?

**Зачем спрашивают.**
WebSocket — основа real-time приложений (чаты, игры, live updates). Интервьюер проверяет знание протокола и паттернов двунаправленной коммуникации.

**Короткий ответ.**
WebSocket — полнодуплексный протокол поверх TCP. В Go используется `gorilla/websocket` или `nhooyr.io/websocket`. На каждое соединение — горутина для чтения и горутина для записи.

**Детальный разбор.**

**Архитектура WebSocket сервера:**
```
┌───────────────────────────────────────────────────────────────┐
│                       WebSocket Server                         │
├───────────────────────────────────────────────────────────────┤
│  Client 1 ◄─────────► goroutine (read/write)                  │
│                              │                                 │
│  Client 2 ◄─────────► goroutine (read/write)                  │
│                              │         broadcast              │
│  Client 3 ◄─────────► goroutine (read/write) ◄─────► Hub     │
│                              │                                 │
│  Client N ◄─────────► goroutine (read/write)                  │
└───────────────────────────────────────────────────────────────┘
```

**Жизненный цикл соединения:**
```
1. HTTP Upgrade → WebSocket handshake
2. Двунаправленный обмен сообщениями
3. Ping/Pong для keep-alive
4. Close handshake
```

**Пример.**
```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        // В production проверять Origin!
        return true
    },
}

type Client struct {
    conn *websocket.Conn
    send chan []byte
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()

        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()

        case message := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}

func handleWebSocket(hub *Hub, w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("upgrade error: %v", err)
        return
    }

    client := &Client{
        conn: conn,
        send: make(chan []byte, 256),
    }
    hub.register <- client

    // Запускаем горутины чтения и записи
    go client.writePump()
    go client.readPump(hub)
}

func (c *Client) readPump(hub *Hub) {
    defer func() {
        hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseAbnormalClosure) {
                log.Printf("error: %v", err)
            }
            break
        }
        hub.broadcast <- message
    }
}

func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            if err := c.conn.WriteMessage(websocket.TextMessage, message); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

**Типичные ошибки.**
1. **Не проверять Origin** — CSRF уязвимость
2. **Блокировка на записи** — используй буферизированный канал
3. **Нет ping/pong** — соединение зависает при обрыве сети
4. **Concurrent write** — websocket.Conn не thread-safe для записи

**На интервью.**
Объясни handshake (HTTP Upgrade). Покажи паттерн Hub для broadcast. Расскажи про ping/pong для detection dead connections.

*Частые follow-up вопросы:*
- Как масштабировать WebSocket на несколько серверов? (Redis Pub/Sub)
- Чем WebSocket отличается от SSE?
- Как аутентифицировать WebSocket соединения?

---

### 10. Как реализовать gRPC load balancing?

**Зачем спрашивают.**
gRPC использует HTTP/2 с persistent connections, что усложняет балансировку. Интервьюер проверяет понимание client-side и server-side балансировки.

**Короткий ответ.**
gRPC поддерживает client-side load balancing через resolver + balancer. Популярные стратегии: round-robin, weighted, pick-first. Для server-side используют Envoy, Linkerd или другие service mesh решения.

**Детальный разбор.**

**Проблема с HTTP/2:**
```
┌─────────────────────────────────────────────────────────────┐
│ HTTP/1.1: каждый запрос — новое соединение                  │
│   Client ─┬─ req1 ─▶ Backend1                               │
│           ├─ req2 ─▶ Backend2  ← LB распределяет запросы    │
│           └─ req3 ─▶ Backend3                               │
├─────────────────────────────────────────────────────────────┤
│ HTTP/2/gRPC: одно соединение — много запросов               │
│   Client ───────────▶ Backend1  ← все запросы идут сюда!    │
│                       Backend2  ← не используется           │
│                       Backend3  ← не используется           │
│                                                              │
│   Нужен client-side LB или L7 proxy (Envoy)                │
└─────────────────────────────────────────────────────────────┘
```

**Архитектура gRPC load balancing:**
```
┌─────────────────────────────────────────────────────────────┐
│                    gRPC Client                               │
├─────────────────────────────────────────────────────────────┤
│  Name Resolver (dns, consul, k8s)                           │
│       │                                                      │
│       ▼                                                      │
│  Balancer (round_robin, grpclb, xds)                        │
│       │                                                      │
│       ├───────────▶ SubConn 1 ──▶ Backend 1                 │
│       ├───────────▶ SubConn 2 ──▶ Backend 2                 │
│       └───────────▶ SubConn 3 ──▶ Backend 3                 │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**
```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    _ "google.golang.org/grpc/balancer/roundrobin"
)

// DNS-based client-side load balancing
func createClient() (*grpc.ClientConn, error) {
    // dns:/// включает DNS resolver
    // round_robin распределяет запросы
    conn, err := grpc.NewClient(
        "dns:///my-service.default.svc.cluster.local:50051",
        grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    return conn, err
}

// Kubernetes headless service
// Service должен быть headless (clusterIP: None)
// DNS возвращает все pod IPs

// Custom resolver для Consul/etcd
type consulResolver struct {
    client *consul.Client
}

func (r *consulResolver) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
    // Подписываемся на изменения в Consul
    go r.watch(target.Endpoint, cc)
    return r, nil
}

func (r *consulResolver) watch(service string, cc resolver.ClientConn) {
    for {
        entries, _, err := r.client.Health().Service(service, "", true, nil)
        if err != nil {
            continue
        }

        addrs := make([]resolver.Address, 0, len(entries))
        for _, entry := range entries {
            addr := fmt.Sprintf("%s:%d", entry.Service.Address, entry.Service.Port)
            addrs = append(addrs, resolver.Address{Addr: addr})
        }

        cc.UpdateState(resolver.State{Addresses: addrs})
        time.Sleep(10 * time.Second)
    }
}

// Регистрация resolver
func init() {
    resolver.Register(&consulResolverBuilder{})
}
```

**Service Mesh (Envoy/Linkerd):**
```yaml
# Envoy sidecar автоматически балансирует
# gRPC client подключается к localhost:50051
# Envoy проксирует к реальным backends

# Kubernetes Service + Envoy
apiVersion: v1
kind: Service
metadata:
  name: my-grpc-service
spec:
  ports:
    - port: 50051
      name: grpc  # важно для Istio/Envoy
```

**Типичные ошибки.**
1. **L4 балансировка для gRPC** — не работает из-за persistent connections
2. **Забыть `dns:///`** — используется pick-first вместо round-robin
3. **Не обновлять endpoints** — клиент не узнает о новых backends
4. **Нет health checks** — запросы идут на мёртвые backends

**На интервью.**
Объясни, почему L4 LB не работает с gRPC. Покажи настройку round-robin через service config. Расскажи про xDS для динамической конфигурации.

*Частые follow-up вопросы:*
- Как работает gRPC health checking protocol?
- Что такое xDS и как его использует gRPC?
- Когда использовать client-side vs server-side LB?

---

### 11. Когда использовать GraphQL vs REST vs gRPC?

**Зачем спрашивают.**
Выбор API протокола влияет на всю архитектуру. Интервьюер проверяет понимание trade-offs и умение принимать архитектурные решения.

**Короткий ответ.**
REST — простота, кэширование, широкая совместимость. gRPC — производительность, строгая типизация, streaming, внутренние сервисы. GraphQL — гибкие запросы, сложные связи данных, мобильные приложения.

**Детальный разбор.**

**Сравнительная таблица:**
```
┌──────────────────┬─────────────┬─────────────┬─────────────┐
│ Критерий         │ REST        │ gRPC        │ GraphQL     │
├──────────────────┼─────────────┼─────────────┼─────────────┤
│ Формат данных    │ JSON/XML    │ Protobuf    │ JSON        │
│ Типизация        │ Опционально │ Строгая     │ Строгая     │
│ HTTP версия      │ HTTP/1.1    │ HTTP/2      │ HTTP/1.1    │
│ Streaming        │ SSE/WS      │ Нативный    │ Subscriptions│
│ Кэширование      │ Отличное    │ Сложное     │ Сложное     │
│ Browser support  │ Нативный    │ grpc-web    │ Нативный    │
│ Code generation  │ OpenAPI     │ protoc      │ codegen     │
│ Производительность│ Средняя    │ Высокая     │ Средняя     │
│ Overfetching    │ Часто       │ Нет         │ Решает      │
│ Сложность       │ Низкая      │ Средняя     │ Высокая     │
└──────────────────┴─────────────┴─────────────┴─────────────┘
```

**Когда REST:**
- Публичные API (широкая совместимость)
- HTTP кэширование критично
- Простые CRUD операции
- Нужна поддержка браузеров без JS
- Команда не знакома с gRPC/GraphQL

**Когда gRPC:**
- Микросервисы (service-to-service)
- Высокие требования к latency
- Streaming (real-time, большие файлы)
- Строгие контракты между командами
- Полиглот окружение (много языков)

**Когда GraphQL:**
- Сложные связи данных (граф)
- Мобильные приложения (минимизация трафика)
- Разные клиенты нуждаются в разных данных
- BFF (Backend for Frontend)
- Rapid prototyping

**Пример.**

**REST:**
```go
// Простой CRUD, хорошее кэширование
// GET /users/123 → Cache-Control: max-age=300
// POST /users → создание
// PUT /users/123 → обновление
```

**gRPC:**
```protobuf
// Эффективный RPC между сервисами
service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

**GraphQL:**
```graphql
# Клиент запрашивает только нужные поля
query {
    user(id: 123) {
        name
        posts(first: 10) {
            title
            comments {
                text
            }
        }
    }
}
```

**Гибридный подход:**
```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile/Web Clients                       │
│                            │                                 │
│                   ┌────────┴────────┐                       │
│                   │  API Gateway    │                       │
│                   │ (REST/GraphQL)  │                       │
│                   └────────┬────────┘                       │
│              ┌─────────────┼─────────────┐                  │
│              │             │             │                  │
│         ┌────┴────┐  ┌────┴────┐  ┌────┴────┐             │
│         │ Service │  │ Service │  │ Service │             │
│         │    A    │◄─┼─►  B    │◄─┼─►  C    │             │
│         └─────────┘  └─────────┘  └─────────┘             │
│              gRPC между сервисами                          │
└─────────────────────────────────────────────────────────────┘
```

**Типичные ошибки.**
1. **gRPC для публичного API** — сложно для web клиентов
2. **REST для streaming** — неэффективно, лучше gRPC/WebSocket
3. **GraphQL для простых CRUD** — overkill, сложность без выгоды
4. **Игнорирование кэширования** — REST сильнее в HTTP cache

**На интервью.**
Покажи реальные сценарии выбора. Объясни trade-off: gRPC быстрее, но REST проще кэшировать. GraphQL гибкий, но сложнее оптимизировать на сервере.

*Частые follow-up вопросы:*
- Как версионировать API в каждом подходе?
- Можно ли комбинировать подходы?
- Как обрабатывать ошибки в каждом случае?

---

## Практика

1. **HTTP Server:** Реализуй сервер с middleware chain (logging, metrics, recovery, auth), graceful shutdown и health checks.

2. **HTTP Client:** Напиши клиент с connection pooling, retry с exponential backoff и circuit breaker.

3. **gRPC Service:** Создай сервис с unary и server streaming методами. Добавь interceptors для auth, logging и metrics.

4. **Serialization:** Сравни JSON, Protobuf и MessagePack для одного payload (размер, скорость encode/decode).

5. **Observability:** Настрой Prometheus метрики, structured logging и OpenTelemetry tracing для HTTP сервиса.

---

## Дополнительные материалы

- [net/http package](https://pkg.go.dev/net/http)
- [gRPC-Go documentation](https://grpc.io/docs/languages/go/)
- [Protocol Buffers Guide](https://protobuf.dev/programming-guides/proto3/)
- [Google API Improvement Proposals](https://google.aip.dev/)
- [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/)
- [Prometheus Go client](https://prometheus.io/docs/guides/go-application/)
- [sqlc documentation](https://sqlc.dev/)
- [ent documentation](https://entgo.io/)
