# 04 — Errors & Context

Ответы на ключевые вопросы о работе с ошибками и контекстами в Go: интерфейс `error`, обёртки, классификация ошибок, отмена и дедлайны, интеграция с логированием.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [03-memory-slices-maps](./03-memory-slices-maps.md) · Следующая тема: [05-tooling-testing](./05-tooling-testing.md)

---

## Вопросы и разборы

### 1. Как устроен интерфейс error и зачем нужны обёртки?

**Зачем спрашивают.** Обработка ошибок — центральная тема в Go. Интервьюер проверяет, понимаешь ли ты философию «errors are values» и механизм error wrapping.

**Короткий ответ.** `error` — интерфейс с методом `Error() string`. Обёртки (`fmt.Errorf` с `%w`) сохраняют цепочку ошибок для анализа через `errors.Is` и `errors.As`.

**Детальный разбор.**

**Интерфейс error:**
```go
type error interface {
    Error() string
}
```

Любой тип с методом `Error() string` является ошибкой. Это позволяет создавать кастомные типы ошибок с дополнительным контекстом.

**Error wrapping (Go 1.13+):**
```
┌────────────────────────────────────────────────────┐
│ fmt.Errorf("save user: %w", err)                   │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │ fmt.Errorf("query db: %w", err)              │  │
│  │                                              │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │ sql.ErrNoRows (original error)         │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘

errors.Is(err, sql.ErrNoRows)  // true — проходит по цепочке
errors.Unwrap(err)             // возвращает следующий уровень
```

**API для работы с ошибками:**
- `errors.Is(err, target)` — проверяет, содержит ли цепочка target
- `errors.As(err, &target)` — извлекает ошибку конкретного типа
- `errors.Unwrap(err)` — возвращает обёрнутую ошибку
- `errors.Join(err1, err2)` — объединяет несколько ошибок (Go 1.20+)

**Пример.**
```go
// Создание ошибки с контекстом
func GetUser(id int) (*User, error) {
    user, err := db.QueryUser(id)
    if err != nil {
        // %w сохраняет оригинальную ошибку
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return user, nil
}

// Проверка конкретной ошибки
func handleError(err error) {
    if errors.Is(err, sql.ErrNoRows) {
        // обработка "not found"
        return
    }
    // другая ошибка
}

// Извлечение типизированной ошибки
var netErr *net.OpError
if errors.As(err, &netErr) {
    fmt.Printf("network error on %s: %v\n", netErr.Op, netErr.Err)
}

// Вывод полной цепочки
fmt.Printf("%+v\n", err)  // с pkg/errors или собственной реализацией
```

**Типичные ошибки.**
- Использовать `%v` вместо `%w` — теряется возможность unwrap.
- Сравнивать ошибки через `==` вместо `errors.Is`.
- Не добавлять контекст при пробросе — теряется информация о месте ошибки.
- Добавлять слишком много контекста — шум в логах.

**На интервью.**
- Объясни разницу между `%v` и `%w` в `fmt.Errorf`.
- Покажи, как `errors.Is` проходит по цепочке.
- Follow-up: «Как реализовать свой метод Is/As?» — добавить методы `Is(error) bool` или `As(any) bool` к типу.

---

### 2. Sentinel errors vs типизированные ошибки vs errors.Join?

**Зачем спрашивают.** Существует несколько подходов к структурированию ошибок. Интервьюер проверяет, знаешь ли ты когда какой применять.

**Короткий ответ.** Sentinel errors (`var ErrNotFound = errors.New(...)`) — для простых проверок. Типизированные ошибки (`type MyError struct`) — для ошибок с контекстом. `errors.Join` — для агрегации нескольких ошибок.

**Детальный разбор.**

**Sentinel errors:**
```go
// Определение
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)

// Использование
if errors.Is(err, ErrNotFound) {
    return http.StatusNotFound
}
```

Плюсы: простота, быстрая проверка.
Минусы: нет дополнительного контекста.

**Типизированные ошибки:**
```go
// Определение
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

// Кастомный Is для совместимости
func (e *ValidationError) Is(target error) bool {
    _, ok := target.(*ValidationError)
    return ok
}

// Использование
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("Field %s: %s\n", valErr.Field, valErr.Message)
}
```

Плюсы: структурированный контекст, типобезопасность.
Минусы: больше кода.

**errors.Join (Go 1.20+):**
```go
func validateUser(u User) error {
    var errs []error

    if u.Name == "" {
        errs = append(errs, errors.New("name required"))
    }
    if u.Email == "" {
        errs = append(errs, errors.New("email required"))
    }
    if len(u.Password) < 8 {
        errs = append(errs, errors.New("password too short"))
    }

    return errors.Join(errs...)  // nil если errs пуст
}

// Проверка: errors.Is работает для всех ошибок в Join
```

**Когда что использовать:**
| Сценарий | Подход |
|----------|--------|
| Простые условия (not found, unauthorized) | Sentinel errors |
| Ошибки с данными (validation, network) | Типизированные ошибки |
| Параллельные операции, несколько ошибок | errors.Join |
| Логирование с контекстом | fmt.Errorf + %w |

**Пример.**
```go
// Комбинированный подход
var ErrValidation = errors.New("validation error")

type FieldError struct {
    Field   string
    Message string
}

func (e *FieldError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

func (e *FieldError) Is(target error) bool {
    return target == ErrValidation
}

func validateEmail(email string) error {
    if email == "" {
        return &FieldError{Field: "email", Message: "required"}
    }
    if !strings.Contains(email, "@") {
        return &FieldError{Field: "email", Message: "invalid format"}
    }
    return nil
}

// Использование
err := validateEmail("")
if errors.Is(err, ErrValidation) {
    var fe *FieldError
    if errors.As(err, &fe) {
        fmt.Printf("Field: %s, Error: %s\n", fe.Field, fe.Message)
    }
}
```

**Типичные ошибки.**
- Экспортировать sentinel errors без необходимости — загрязняет API.
- Создавать новую ошибку каждый раз вместо sentinel — `errors.Is` не сработает.
- Игнорировать errors.Join и терять ошибки в параллельных операциях.

**На интервью.**
- Приведи примеры из stdlib: `io.EOF`, `sql.ErrNoRows`, `context.Canceled`.
- Объясни, когда кастомный тип лучше sentinel.
- Follow-up: «Как errors.Join работает с errors.Is?» — проверяет каждую ошибку в списке.

---

### 3. Как проектировать многоуровневую обработку ошибок?

**Зачем спрашивают.** В реальных приложениях ошибки проходят через несколько слоёв. Интервьюер проверяет понимание архитектуры обработки ошибок.

**Короткий ответ.** Нижние слои добавляют контекст через wrapping. Верхние слои принимают решения (retry, fallback, HTTP status). Логирование — на границе системы, не на каждом уровне.

**Детальный разбор.**

**Архитектура обработки:**
```
┌─────────────────────────────────────────────────────┐
│ HTTP Handler (presentation layer)                   │
│ - Преобразует ошибку в HTTP статус                 │
│ - Логирует для observability                       │
│ - Возвращает user-friendly сообщение               │
├─────────────────────────────────────────────────────┤
│ Service Layer (business logic)                      │
│ - Добавляет бизнес-контекст                        │
│ - Принимает решения о retry/fallback               │
│ - НЕ логирует (пробрасывает наверх)                │
├─────────────────────────────────────────────────────┤
│ Repository Layer (data access)                      │
│ - Добавляет технический контекст (query, params)   │
│ - Оборачивает низкоуровневые ошибки                │
├─────────────────────────────────────────────────────┤
│ Infrastructure (database, external APIs)            │
│ - Оригинальные ошибки (sql.ErrNoRows, net.Error)   │
└─────────────────────────────────────────────────────┘
```

**Пример.**
```go
// Repository layer
func (r *UserRepo) FindByID(ctx context.Context, id int64) (*User, error) {
    user := &User{}
    err := r.db.QueryRowContext(ctx, "SELECT * FROM users WHERE id = $1", id).Scan(&user.ID, &user.Name)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrUserNotFound  // конвертация в доменную ошибку
        }
        return nil, fmt.Errorf("query user %d: %w", id, err)
    }
    return user, nil
}

// Service layer
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        // Добавляем бизнес-контекст, но не логируем
        return nil, fmt.Errorf("get user: %w", err)
    }
    return user, nil
}

// HTTP Handler (presentation layer)
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)

    user, err := h.svc.GetUser(r.Context(), id)
    if err != nil {
        // Логируем на границе
        h.logger.Error("get user failed",
            "error", err,
            "user_id", id,
            "request_id", middleware.GetReqID(r.Context()),
        )

        // Преобразуем в HTTP статус
        switch {
        case errors.Is(err, ErrUserNotFound):
            http.Error(w, "User not found", http.StatusNotFound)
        case errors.Is(err, context.DeadlineExceeded):
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
        default:
            http.Error(w, "Internal error", http.StatusInternalServerError)
        }
        return
    }

    json.NewEncoder(w).Encode(user)
}
```

**Типичные ошибки.**
- Логировать на каждом уровне — дублирование в логах.
- Не добавлять контекст — непонятно, где произошла ошибка.
- Добавлять слишком много контекста — трудночитаемые сообщения.
- Возвращать внутренние ошибки наружу — утечка информации.

**На интервью.**
- Нарисуй архитектуру с потоком ошибок.
- Объясни, почему логирование должно быть на границе.
- Follow-up: «Как тестировать обработку ошибок?» — моки, table-driven tests с разными ошибками.

---

### 4. Как различать Recoverable и Fatal ошибки?

**Зачем спрашивают.** Не все ошибки одинаковы: некоторые можно retry, другие — fatal. Интервьюер проверяет понимание классификации.

**Короткий ответ.** Используй интерфейсы с методами типа `Temporary()`, `Retryable()`, или кастомные типы. Пример из stdlib: `net.Error` с `Timeout()` и `Temporary()`.

**Детальный разбор.**

**Классификация ошибок:**
| Тип | Действие | Примеры |
|-----|----------|---------|
| Recoverable/Transient | Retry с backoff | timeout, connection reset, 503 |
| Permanent/Fatal | Fail fast | 404, validation error, auth failure |
| Partial | Fallback/degrade | cache miss → query DB |

**Паттерн: интерфейс Retryable**
```go
type Retryable interface {
    error
    Retryable() bool
}

type retryableError struct {
    err       error
    retryable bool
}

func (e *retryableError) Error() string { return e.err.Error() }
func (e *retryableError) Unwrap() error { return e.err }
func (e *retryableError) Retryable() bool { return e.retryable }

func NewRetryable(err error) error {
    return &retryableError{err: err, retryable: true}
}

func NewPermanent(err error) error {
    return &retryableError{err: err, retryable: false}
}

// Использование
func callAPI() error {
    resp, err := http.Get(url)
    if err != nil {
        var netErr net.Error
        if errors.As(err, &netErr) && netErr.Timeout() {
            return NewRetryable(err)
        }
        return NewPermanent(err)
    }

    switch resp.StatusCode {
    case 503, 429:
        return NewRetryable(fmt.Errorf("status %d", resp.StatusCode))
    case 400, 401, 404:
        return NewPermanent(fmt.Errorf("status %d", resp.StatusCode))
    }
    return nil
}

// Retry logic
func withRetry(fn func() error, maxAttempts int) error {
    var lastErr error
    for i := 0; i < maxAttempts; i++ {
        err := fn()
        if err == nil {
            return nil
        }

        var r Retryable
        if errors.As(err, &r) && !r.Retryable() {
            return err  // permanent error, don't retry
        }

        lastErr = err
        time.Sleep(time.Duration(i+1) * 100 * time.Millisecond)
    }
    return fmt.Errorf("max attempts exceeded: %w", lastErr)
}
```

**stdlib пример: net.Error**
```go
type Error interface {
    error
    Timeout() bool   // был ли таймаут
    Temporary() bool // временная ли ошибка (deprecated в Go 1.18)
}

// Использование
var netErr net.Error
if errors.As(err, &netErr) {
    if netErr.Timeout() {
        // retry
    }
}
```

**Пример.**
```go
// Полный пример с классификацией
type ErrorKind int

const (
    KindUnknown ErrorKind = iota
    KindNotFound
    KindValidation
    KindPermission
    KindTransient
    KindInternal
)

type AppError struct {
    Kind    ErrorKind
    Message string
    Err     error
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Message, e.Err)
    }
    return e.Message
}

func (e *AppError) Unwrap() error { return e.Err }

func (e *AppError) Is(target error) bool {
    t, ok := target.(*AppError)
    return ok && t.Kind == e.Kind
}

// Хелперы
func NotFound(msg string) error {
    return &AppError{Kind: KindNotFound, Message: msg}
}

func Transient(err error) error {
    return &AppError{Kind: KindTransient, Message: "transient error", Err: err}
}

// HTTP mapping
func HTTPStatus(err error) int {
    var appErr *AppError
    if errors.As(err, &appErr) {
        switch appErr.Kind {
        case KindNotFound:
            return http.StatusNotFound
        case KindValidation:
            return http.StatusBadRequest
        case KindPermission:
            return http.StatusForbidden
        case KindTransient:
            return http.StatusServiceUnavailable
        default:
            return http.StatusInternalServerError
        }
    }
    return http.StatusInternalServerError
}
```

**Типичные ошибки.**
- Retry всё подряд — бесполезно для validation errors.
- Не ограничивать число retry — бесконечный цикл.
- Игнорировать Retry-After header — нагружать сервер.

**На интервью.**
- Приведи примеры recoverable (timeout) и fatal (404) ошибок.
- Покажи паттерн с интерфейсом Retryable.
- Follow-up: «Как реализовать exponential backoff?» — `time.Sleep(baseDelay * 2^attempt)` с jitter.

---

### 5. Как работает context.Context?

**Зачем спрашивают.** Context — основа для отмены операций, таймаутов и передачи request-scoped данных. Без него невозможен production Go код.

**Короткий ответ.** Context передаёт deadline, cancellation signal и key-value данные между функциями. Создаётся через `context.Background()`, производные — через `WithCancel`, `WithTimeout`, `WithValue`.

**Детальный разбор.**

**Интерфейс Context:**
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error  // nil, Canceled, или DeadlineExceeded
    Value(key any) any
}
```

**Иерархия контекстов:**
```
context.Background()
        │
        ├── WithCancel(parent)
        │       │
        │       └── WithTimeout(parent, 5s)
        │               │
        │               └── WithValue(parent, "user_id", 123)
        │
        └── WithDeadline(parent, time.Now().Add(10s))

Отмена родителя отменяет всех детей!
```

**Создание контекстов:**
```go
// Background — корневой, никогда не отменяется
ctx := context.Background()

// TODO — для рефакторинга, когда context ещё не реализован
ctx := context.TODO()

// WithCancel — ручная отмена
ctx, cancel := context.WithCancel(parent)
defer cancel()  // ВАЖНО: всегда вызывать для освобождения ресурсов

// WithTimeout — автоматическая отмена через время
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

// WithDeadline — отмена в конкретный момент
ctx, cancel := context.WithDeadline(parent, time.Now().Add(time.Hour))
defer cancel()

// WithValue — передача данных
ctx := context.WithValue(parent, "request_id", "abc123")
```

**Пример.**
```go
func main() {
    // HTTP server автоматически создаёт context для каждого запроса
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()  // уже с cancel при закрытии connection

        // Добавляем timeout для операции
        ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
        defer cancel()

        result, err := fetchData(ctx)
        if err != nil {
            if errors.Is(err, context.DeadlineExceeded) {
                http.Error(w, "Timeout", http.StatusGatewayTimeout)
                return
            }
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }

        json.NewEncoder(w).Encode(result)
    })
}

func fetchData(ctx context.Context) (*Data, error) {
    // Проверка отмены перед долгой операцией
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    // Передаём context дальше
    resp, err := httpClient.Do(req.WithContext(ctx))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    // ...
}
```

**Типичные ошибки.**
- Забывать вызывать `cancel()` — утечка ресурсов.
- Хранить context в структуре — должен передаваться первым аргументом.
- Использовать context.Value для бизнес-данных — только для request-scoped metadata.
- Передавать nil context — используй context.TODO() если нужен placeholder.

**На интервью.**
- Объясни разницу между cancel и timeout.
- Покажи паттерн defer cancel().
- Follow-up: «Что хранить в context.Value?» — request ID, auth token, logger, trace ID. НЕ бизнес-параметры.

---

### 6. Как правильно использовать таймауты и дедлайны?

**Зачем спрашивают.** Таймауты критичны для надёжности. Интервьюер проверяет понимание разницы между timeout и deadline, и правильного их применения.

**Короткий ответ.** Timeout — длительность операции. Deadline — абсолютное время завершения. В сетевых вызовах всегда устанавливай таймауты. Проверяй ctx.Done() в долгих операциях.

**Детальный разбор.**

**Timeout vs Deadline:**
```go
// Timeout — относительное время
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
// deadline = now + 5s

// Deadline — абсолютное время
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
```

**Таймауты HTTP клиента:**
```go
client := &http.Client{
    Timeout: 10 * time.Second,  // общий таймаут на весь запрос

    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,  // таймаут на установку соединения
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   5 * time.Second,
        ResponseHeaderTimeout: 5 * time.Second,
        IdleConnTimeout:       90 * time.Second,
    },
}
```

**Таймауты в разных слоях:**
```
HTTP Request Timeout (10s)
├── Connect (2s)
├── TLS Handshake (2s)
├── Send Request
├── Wait for Headers (3s)
└── Read Body (remaining time)

Каждый слой должен уложиться в общий таймаут!
```

**Пример.**
```go
func callWithTimeout(ctx context.Context) (*Response, error) {
    // Создаём контекст с таймаутом
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // Используем select для timeout-aware операций
    resultCh := make(chan *Response, 1)
    errCh := make(chan error, 1)

    go func() {
        resp, err := doSlowOperation(ctx)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- resp
    }()

    select {
    case resp := <-resultCh:
        return resp, nil
    case err := <-errCh:
        return nil, err
    case <-ctx.Done():
        return nil, ctx.Err()  // DeadlineExceeded или Canceled
    }
}

// Проверка remaining time
func processWithDeadline(ctx context.Context) error {
    deadline, ok := ctx.Deadline()
    if !ok {
        return errors.New("no deadline set")
    }

    remaining := time.Until(deadline)
    if remaining < 100*time.Millisecond {
        return errors.New("not enough time")
    }

    // Продолжаем выполнение...
    return nil
}
```

**Типичные ошибки.**
- Не устанавливать таймауты — операции висят бесконечно.
- Слишком короткие таймауты — ложные ошибки.
- Не проверять ctx.Done() в циклах — операция не прервётся.
- Игнорировать оставшееся время при передаче context дочерним операциям.

**На интервью.**
- Покажи настройку таймаутов HTTP клиента.
- Объясни, как проверить оставшееся время через Deadline().
- Follow-up: «Как обработать частичный результат при таймауте?» — select с resultCh и ctx.Done().

---

### 7. Как интегрировать ошибки с логированием?

**Зачем спрашивают.** Хорошее логирование — ключ к отладке. Интервьюер проверяет понимание structured logging и интеграции с ошибками.

**Короткий ответ.** Используй structured logging (slog, zap, zerolog). Логируй на границе системы, не на каждом уровне. Включай request ID, user ID, и другой контекст.

**Детальный разбор.**

**Проблемы стандартного log:**
- Нет уровней (info, error, debug)
- Нет structured output (JSON)
- Нет контекста (request ID, user ID)

**slog (Go 1.21+):**
```go
import "log/slog"

// Создание logger
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// Использование
logger.Info("user created",
    "user_id", user.ID,
    "email", user.Email,
)

logger.Error("failed to create user",
    "error", err,
    "email", email,
)

// Output (JSON):
// {"time":"2024-01-15T10:30:00Z","level":"INFO","msg":"user created","user_id":123,"email":"user@example.com"}
// {"time":"2024-01-15T10:30:01Z","level":"ERROR","msg":"failed to create user","error":"validation failed","email":"bad@"}
```

**Logger в context:**
```go
type ctxKey string

const loggerKey ctxKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFrom(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}

// Middleware для HTTP
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()

        logger := slog.Default().With(
            "request_id", requestID,
            "method", r.Method,
            "path", r.URL.Path,
        )

        ctx := WithLogger(r.Context(), logger)
        r = r.WithContext(ctx)

        next.ServeHTTP(w, r)
    })
}
```

**Пример.**
```go
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    logger := LoggerFrom(ctx)

    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        logger.Warn("invalid request body", "error", err)
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    user, err := h.svc.CreateUser(ctx, req)
    if err != nil {
        // Логируем с полным контекстом
        logger.Error("failed to create user",
            "error", err,
            "email", req.Email,
        )

        // Маппинг на HTTP статус
        switch {
        case errors.Is(err, ErrEmailExists):
            http.Error(w, "Email already exists", http.StatusConflict)
        default:
            http.Error(w, "Internal error", http.StatusInternalServerError)
        }
        return
    }

    logger.Info("user created", "user_id", user.ID)
    json.NewEncoder(w).Encode(user)
}
```

**Типичные ошибки.**
- Логировать на каждом уровне — дублирование.
- Логировать sensitive data (пароли, токены).
- Не включать request ID — невозможно трассировать запрос.
- Использовать string concatenation вместо structured fields.

**На интервью.**
- Покажи пример structured logging с slog.
- Объясни, почему логировать нужно на границе.
- Follow-up: «Как интегрировать с tracing?» — передавать trace ID через context.

---

### 8. Best practices по использованию context и ошибок

**Зачем спрашивают.** Проверка практического опыта и знания идиом.

**Короткий ответ.** Context — первым аргументом, не хранить в struct. Ошибки — последним возвращаемым значением, всегда проверять, добавлять контекст при пробросе.

**Детальный разбор.**

**Context best practices:**
```go
// ✅ Context первым аргументом
func GetUser(ctx context.Context, id int64) (*User, error)

// ❌ Context в struct
type Service struct {
    ctx context.Context  // НЕ делать так
}

// ✅ Всегда вызывать cancel
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()

// ✅ Проверять Done() в циклах
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // работа
    }
}

// ❌ Использовать Value для бизнес-данных
ctx = context.WithValue(ctx, "user", user)  // плохо

// ✅ Только для request-scoped metadata
ctx = context.WithValue(ctx, requestIDKey, "abc123")
```

**Error best practices:**
```go
// ✅ Проверять ошибки
result, err := doSomething()
if err != nil {
    return err
}

// ❌ Игнорировать ошибки
result, _ := doSomething()

// ✅ Добавлять контекст при пробросе
if err != nil {
    return fmt.Errorf("save user %d: %w", id, err)
}

// ❌ Терять оригинальную ошибку
if err != nil {
    return fmt.Errorf("save user failed")  // теряем причину
}

// ✅ Использовать errors.Is/As
if errors.Is(err, sql.ErrNoRows) {
    return ErrNotFound
}

// ❌ Сравнивать через ==
if err == sql.ErrNoRows {  // работает, но не с wrapped errors
}

// ✅ Документировать возвращаемые ошибки
// GetUser returns ErrNotFound if user doesn't exist.
func GetUser(ctx context.Context, id int64) (*User, error)
```

**Пример: полная картина**
```go
func (s *Service) ProcessOrder(ctx context.Context, orderID string) error {
    // Создаём таймаут для всей операции
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // Получаем заказ
    order, err := s.repo.GetOrder(ctx, orderID)
    if err != nil {
        return fmt.Errorf("get order: %w", err)
    }

    // Проверяем отмену перед долгой операцией
    if err := ctx.Err(); err != nil {
        return fmt.Errorf("cancelled before payment: %w", err)
    }

    // Обрабатываем платёж
    payment, err := s.payments.Charge(ctx, order)
    if err != nil {
        // Классифицируем ошибку
        var payErr *PaymentError
        if errors.As(err, &payErr) && payErr.Retryable() {
            return NewRetryable(fmt.Errorf("charge payment: %w", err))
        }
        return fmt.Errorf("charge payment: %w", err)
    }

    // Обновляем заказ
    if err := s.repo.UpdateOrderPayment(ctx, orderID, payment.ID); err != nil {
        // Платёж прошёл, но не сохранили — нужно компенсировать
        if refundErr := s.payments.Refund(ctx, payment.ID); refundErr != nil {
            return errors.Join(
                fmt.Errorf("update order: %w", err),
                fmt.Errorf("refund failed: %w", refundErr),
            )
        }
        return fmt.Errorf("update order (refunded): %w", err)
    }

    return nil
}
```

**Типичные ошибки.**
- Не добавлять контекст — непонятно, где ошибка.
- Добавлять контекст на каждом уровне — слишком длинные сообщения.
- Не вызывать defer cancel() — утечка горутин.
- Передавать nil context — panic в некоторых библиотеках.

**На интервью.**
- Сформулируй 3-5 главных правил по context и errors.
- Покажи пример правильной обработки цепочки операций.
- Follow-up: «Как тестировать timeout/cancel?» — context.WithCancel в тесте, вызвать cancel() перед операцией.

---

## См. также

- [Goroutines & Channels](./01-goroutines-channels.md) — использование context для отмены горутин
- [Java Exceptions](../01-java/09-exceptions.md) — сравнение подходов к обработке ошибок
- [Python Errors & Debugging](../02-python/05-errors-debugging.md) — исключения в Python
- [Resilience Patterns](../08-architecture/08-resilience-patterns.md) — паттерны отказоустойчивости
- [Incident Management](../09-devops-sre/06-incident-management.md) — обработка инцидентов в продакшене

---

## Практика

1. **Custom error type** — реализуй тип ошибки с полями `Op`, `Kind`, `Err`, поддержкой `Unwrap`, `Is`, `As`. Покрой тестами.

2. **Retry with classification** — реализуй retry logic, который различает retryable и permanent ошибки. Добавь exponential backoff.

3. **Context propagation** — реализуй HTTP middleware, который добавляет request ID в context и logger.

4. **Timeout handling** — реализуй функцию, которая выполняет несколько параллельных запросов с общим таймаутом и возвращает первый успешный результат.

5. **Error aggregation** — реализуй валидацию с `errors.Join`, возвращающую все ошибки, а не первую.

6. **Structured logging** — интегрируй slog в HTTP сервер с request ID, latency, и error logging.

---

## Дополнительные материалы

- [Go blog: Error handling and Go](https://go.dev/blog/error-handling-and-go) — основы
- [Go blog: Working with Errors in Go 1.13](https://go.dev/blog/go1.13-errors) — wrapping
- [Go blog: Errors are values](https://go.dev/blog/errors-are-values) — философия
- [context package](https://pkg.go.dev/context) — документация
- [slog package](https://pkg.go.dev/log/slog) — structured logging (Go 1.21+)
- [Dave Cheney: Don't just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)
