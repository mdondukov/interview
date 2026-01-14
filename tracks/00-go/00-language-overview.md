# 00 — Язык Go: позиционирование и система типов

Развёрнутые ответы на «общие» вопросы о Go: философия языка, система типов, generics, преимущества и ограничения. Материал построен в формате вопрос-ответ с единой структурой для быстрого повторения перед интервью.

**Навигация:** [Трек Go](./README.md) · Следующая тема: [01-goroutines-channels](./01-goroutines-channels.md)

---

## Вопросы и разборы

### 1. Go — императивный или декларативный язык?

**Зачем спрашивают.** Интервьюер проверяет, понимаешь ли ты место Go среди других языков и осознаёшь ли его философию. Ответ показывает, как ты будешь подходить к проектированию: писать явный код или искать абстракции.

**Короткий ответ.** Go — императивный язык с процедурным ядром. Программист явно описывает последовательность действий, а не декларирует желаемый результат.

**Детальный разбор.**

*Императивность* означает, что ты управляешь состоянием через присваивания, циклы, условия. В Go нет встроенных конструкций для ленивых вычислений, pattern matching на уровне синтаксиса или монадических цепочек. Это сознательный выбор дизайнеров языка: простота и предсказуемость важнее выразительности.

При этом Go отлично интегрируется с декларативными DSL:
- SQL-запросы через `database/sql` или ORM
- Kubernetes manifests через client-go
- Конфигурации в YAML/JSON через `encoding/json`, `gopkg.in/yaml.v3`

Декларативность появляется на уровне библиотек, а не языка. Это позволяет сохранить простоту ядра и при этом решать сложные задачи.

**Пример.**
```go
// Императивная state machine — явный контроль переходов
type State int

const (
    StateIdle State = iota
    StateRunning
    StateStopped
)

func (s State) Transition(event string) State {
    switch s {
    case StateIdle:
        if event == "start" {
            return StateRunning
        }
    case StateRunning:
        if event == "stop" {
            return StateStopped
        }
    }
    return s // остаёмся в текущем состоянии
}

// Использование
func main() {
    state := StateIdle
    state = state.Transition("start")  // StateRunning
    state = state.Transition("stop")   // StateStopped
}
```

**Типичные ошибки.**
- Пытаться писать на Go как на Haskell/Scala — строить сложные абстракции, монады, функциональные цепочки. Код становится нечитаемым для команды.
- Игнорировать простоту языка и усложнять там, где достаточно `for` и `if`.

**На интервью.**
- Подчеркни, что императивность Go — это фича, а не баг. Это даёт предсказуемость и простоту отладки.
- Упомяни, что декларативные подходы возможны через библиотеки (sqlc, ent, wire).
- Follow-up вопрос: «А как бы ты реализовал декларативный API на Go?» — ответ: через fluent builders или functional options.

---

### 2. Что такое type switch и когда его использовать?

**Зачем спрашивают.** В Go нет union types или sealed classes. Type switch — основной инструмент для работы с гетерогенными данными. Интервьюер проверяет, умеешь ли ты безопасно обрабатывать `interface{}` / `any`.

**Короткий ответ.** Type switch (`switch v := x.(type)`) позволяет ветвиться по конкретному типу значения, хранящегося в интерфейсе. Переменная `v` в каждой ветке уже приведена к соответствующему типу.

**Детальный разбор.**

Type switch работает только с интерфейсными значениями. Под капотом Go хранит в интерфейсе пару `(type, value)` — type switch извлекает информацию о типе и сравнивает её.

Основные сценарии применения:
1. **Обработка JSON** — `json.Unmarshal` в `map[string]any` возвращает примитивы как `float64`, `string`, `bool`, `nil`, `[]any`, `map[string]any`.
2. **Работа с ошибками** — определение конкретного типа ошибки для специфичной обработки (хотя `errors.As` предпочтительнее).
3. **Visitor pattern** — обход AST или других иерархических структур.
4. **Generic-код до Go 1.18** — обработка разных типов в одной функции.

**Пример.**
```go
// Функция складывает число с int64, принимая разные типы
func addToInt64(a any, b int64) (int64, error) {
    switch v := a.(type) {
    case int:
        return int64(v) + b, nil
    case int64:
        return v + b, nil
    case float64:
        return int64(v) + b, nil  // потеря точности — документируй!
    case json.Number:
        val, err := v.Int64()
        if err != nil {
            return 0, fmt.Errorf("parse json.Number: %w", err)
        }
        return val + b, nil
    case string:
        val, err := strconv.ParseInt(v, 10, 64)
        if err != nil {
            return 0, fmt.Errorf("parse string %q: %w", v, err)
        }
        return val + b, nil
    default:
        return 0, fmt.Errorf("unsupported type %T", a)
    }
}

// Использование
result, _ := addToInt64(json.Number("42"), 10)  // 52
result, _ = addToInt64(3.14, 10)                 // 13
```

**Типичные ошибки.**
- Забыть `default` — если придёт неожиданный тип, функция вернёт zero value без ошибки.
- Использовать type switch вместо `errors.As` для ошибок — теряется unwrap chain.
- Порядок веток важен: `case int, int64` — переменная `v` будет типа `any`, а не конкретного типа.

**На интервью.**
- Покажи, что понимаешь разницу между type switch и type assertion (`x.(T)`).
- Упомяни, что с generics (Go 1.18+) многие use cases type switch заменяются параметрическим полиморфизмом.
- Follow-up: «Как работает type switch под капотом?» — сравнение `runtime._type` указателей.

---

### 3. Как явно указать, что тип реализует интерфейс?

**Зачем спрашивают.** Go использует структурную типизацию — реализация интерфейса неявная. Интервьюер проверяет, знаешь ли ты идиому защиты от регрессий.

**Короткий ответ.** Используй compile-time assertion: `var _ Interface = (*Type)(nil)`. Если тип перестанет удовлетворять интерфейсу, компиляция упадёт на этой строке.

**Детальный разбор.**

В отличие от Java/C#, где пишут `class Foo implements Bar`, в Go тип реализует интерфейс автоматически, если у него есть все нужные методы. Это удобно для расширяемости, но создаёт риск: рефакторинг метода (переименование, изменение сигнатуры) может незаметно сломать реализацию интерфейса.

Идиома `var _ I = (*T)(nil)` создаёт переменную типа интерфейса и присваивает ей nil-указатель на тип. Компилятор проверяет совместимость типов. Переменная не занимает память в рантайме (оптимизируется).

Варианты записи:
```go
var _ io.Reader = (*myReader)(nil)     // проверка pointer receiver
var _ io.Reader = myReader{}           // проверка value receiver
var _ io.Reader = (*myReader)(nil)     // pointer receiver покрывает оба случая
```

**Пример.**
```go
// domain/repository.go
type UserRepository interface {
    FindByID(ctx context.Context, id int64) (*User, error)
    Save(ctx context.Context, user *User) error
}

// infra/postgres/user_repo.go
type pgUserRepository struct {
    db *sql.DB
}

// Compile-time check — если забудем метод, увидим ошибку здесь
var _ domain.UserRepository = (*pgUserRepository)(nil)

func (r *pgUserRepository) FindByID(ctx context.Context, id int64) (*domain.User, error) {
    // реализация
}

func (r *pgUserRepository) Save(ctx context.Context, user *domain.User) error {
    // реализация
}
```

**Типичные ошибки.**
- Забыть про pointer vs value receiver: `var _ I = T{}` не проверит методы с pointer receiver.
- Размещать проверку далеко от типа — лучше сразу после объявления структуры.

**На интервью.**
- Объясни, почему Go выбрал структурную типизацию: меньше связанность, легче тестировать, можно реализовать интерфейс для чужих типов.
- Упомяни, что проверка работает в compile-time и не влияет на производительность.
- Follow-up: «А как проверить, реализует ли тип интерфейс в runtime?» — type assertion `_, ok := x.(Interface)`.

---

### 4. Где определять интерфейс: в пакете реализации или использования?

**Зачем спрашивают.** Вопрос о проектировании зависимостей. Неправильное размещение интерфейсов приводит к циклическим импортам и tight coupling.

**Короткий ответ.** Интерфейс определяет потребитель (consumer), а не провайдер. Это позволяет зависеть от абстракций, а не от конкретных реализаций.

**Детальный разбор.**

Принцип «интерфейс у потребителя» следует из Dependency Inversion Principle (SOLID). Преимущества:
1. **Минимальный контракт** — потребитель определяет только те методы, которые ему нужны.
2. **Тестируемость** — легко создать mock, реализующий маленький интерфейс.
3. **Слабая связанность** — реализацию можно заменить без изменения потребителя.

Исключения:
- **Стандартные интерфейсы** (`io.Reader`, `http.Handler`, `fmt.Stringer`) — определены в stdlib, используются везде.
- **Plugin-архитектура** — интерфейс определяет контракт для расширений, живёт в ядре системы.

**Пример.**
```go
// === Плохо: интерфейс в пакете реализации ===
// storage/storage.go
package storage

type Storage interface {  // 10 методов, большинство не нужны потребителям
    Save(ctx context.Context, data []byte) error
    Load(ctx context.Context, key string) ([]byte, error)
    Delete(ctx context.Context, key string) error
    List(ctx context.Context, prefix string) ([]string, error)
    // ... ещё 6 методов
}

type S3Storage struct { /* ... */ }

// === Хорошо: интерфейс у потребителя ===
// service/user_service.go
package service

// Только то, что нужно этому сервису
type UserStore interface {
    Save(ctx context.Context, data []byte) error
    Load(ctx context.Context, key string) ([]byte, error)
}

type UserService struct {
    store UserStore  // зависим от абстракции
}

func NewUserService(store UserStore) *UserService {
    return &UserService{store: store}
}

// В тестах легко создать mock
type mockStore struct{}
func (m *mockStore) Save(ctx context.Context, data []byte) error { return nil }
func (m *mockStore) Load(ctx context.Context, key string) ([]byte, error) { return nil, nil }
```

**Типичные ошибки.**
- Создавать «god interface» с десятком методов в пакете реализации.
- Дублировать интерфейсы — если несколько потребителей используют одинаковый контракт, вынеси в общий пакет.
- Определять интерфейс заранее «на всякий случай» — в Go принято «accept interfaces, return structs».

**На интервью.**
- Приведи пример из реальной практики: `database/sql` определяет `driver.Driver`, но потребители (`sql.DB`) работают с конкретными методами.
- Упомяни Go proverb: «The bigger the interface, the weaker the abstraction».
- Follow-up: «Как избежать циклических импортов?» — интерфейсы в пакете потребителя или в отдельном `domain`/`ports` пакете.

---

### 5. Почему embedding — это не наследование?

**Зачем спрашивают.** Многие приходят из ООП-языков и пытаются использовать embedding как наследование. Это приводит к багам и неидиоматичному коду.

**Короткий ответ.** Embedding — это композиция с синтаксическим сахаром. Методы встроенного типа «поднимаются» на уровень внешнего, но нет виртуальных вызовов, переопределения или `super`.

**Детальный разбор.**

При embedding Go делает следующее:
1. Встроенный тип становится анонимным полем структуры.
2. Методы встроенного типа доступны напрямую через внешний тип.
3. Если внешний тип определяет метод с тем же именем — он «затеняет» (shadows) встроенный.

Ключевые отличия от наследования:
| Аспект | Наследование (Java/C++) | Embedding (Go) |
|--------|------------------------|----------------|
| Иерархия | `class Dog extends Animal` | Нет иерархии типов |
| Полиморфизм | Виртуальные методы, `override` | Нет, методы не переопределяются |
| `super` / `base` | Доступ к родителю | Нет, но можно обратиться к полю явно |
| Память | Объект содержит данные родителя | Поле занимает память явно |
| Type identity | `Dog` is-a `Animal` | `Handler` has-a `Logger` |

**Пример.**
```go
// Embedding для переиспользования поведения
type Logger struct {
    prefix string
}

func (l *Logger) Log(msg string) {
    fmt.Printf("[%s] %s\n", l.prefix, msg)
}

func (l *Logger) SetPrefix(p string) {
    l.prefix = p
}

type Handler struct {
    *Logger  // embedding — Handler получает методы Logger
    svc Service
}

func NewHandler(svc Service) *Handler {
    return &Handler{
        Logger: &Logger{prefix: "HTTP"},
        svc:    svc,
    }
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    h.Log(r.URL.Path)  // вызов «унаследованного» метода
    // ...
}

// Можно «переопределить» метод (shadowing)
func (h *Handler) Log(msg string) {
    h.Logger.Log("OVERRIDE: " + msg)  // явный вызов embedded
}
```

**Типичные ошибки.**
- Ожидать, что `Handler` будет типом `Logger` — это не так, type assertion `h.(Logger)` не сработает.
- Встраивать мьютекс `sync.Mutex` без осознания, что его методы станут публичными.
- Конфликты имён: если встроены два типа с одинаковым методом, вызов без квалификации не скомпилируется.

**На интервью.**
- Объясни, что embedding — это «has-a», а не «is-a».
- Приведи практический пример: встраивание `sync.Mutex` для защиты структуры, встраивание `http.Handler` для middleware.
- Follow-up: «Как embedding помогает реализовать интерфейс?» — встроенный тип может уже реализовывать интерфейс, и внешний тип автоматически его получает.

---

### 6. Какие средства обобщённого программирования есть в Go?

**Зачем спрашивают.** Generics появились в Go 1.18 — интервьюер проверяет, знаешь ли ты современный Go и понимаешь ли trade-offs.

**Короткий ответ.** Go 1.18+ поддерживает параметрический полиморфизм через type parameters. До этого использовали интерфейсы, `interface{}`/`any`, `reflect` и кодогенерацию.

**Детальный разбор.**

**До Go 1.18:**
- `interface{}` + type switch — runtime проверки, нет type safety.
- `reflect` — мощно, но медленно и хрупко.
- Кодогенерация (`go generate`, `text/template`) — безопасно, но громоздко.
- Копипаст — просто, но не DRY.

**Go 1.18+:**
- Type parameters: `func Foo[T any](x T) T`
- Type constraints: `[T constraints.Ordered]`, `[T io.Reader]`
- Type inference: `Foo(42)` вместо `Foo[int](42)`
- Моноинстанциация: компилятор генерирует отдельный код для каждого типа (как C++ templates).

**Ограничения generics в Go:**
- Нет специализации (нельзя написать особую версию для конкретного типа).
- Нет variadic type parameters (`[T1, T2, T3...]`).
- Методы не могут иметь собственных type parameters.
- Нет covariance/contravariance.

**Пример.**
```go
import "golang.org/x/exp/constraints"  // или cmp.Ordered в Go 1.21+

// Generic функция с constraint
func Max[T constraints.Ordered](items []T) (T, error) {
    if len(items) == 0 {
        var zero T
        return zero, errors.New("empty slice")
    }

    max := items[0]
    for _, v := range items[1:] {
        if v > max {
            max = v
        }
    }
    return max, nil
}

// Generic тип
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Использование
func main() {
    // Type inference — компилятор выводит [int]
    maxInt, _ := Max([]int{1, 5, 3})      // 5
    maxStr, _ := Max([]string{"a", "c", "b"})  // "c"

    stack := &Stack[string]{}
    stack.Push("hello")
    stack.Push("world")
    val, _ := stack.Pop()  // "world"
}
```

**Типичные ошибки.**
- Использовать generics везде — если достаточно `[]byte` или `io.Reader`, не усложняй.
- Забыть про zero value: `var zero T` нужен для возврата при ошибке.
- Путать `any` и `interface{}` — это синонимы, но `any` читается лучше в generic контексте.

**На интервью.**
- Покажи, что понимаешь, когда generics нужны (коллекции, алгоритмы), а когда нет (доменная логика).
- Упомяни пакеты `slices`, `maps` из stdlib (Go 1.21+) как примеры идиоматичного использования.
- Follow-up: «Как работает моноинстанциация?» — компилятор создаёт копию функции для каждого набора типов.

---

### 7. Какие технологические преимущества у Go?

**Зачем спрашивают.** Нужно уметь аргументировать выбор Go для проекта. Интервьюер оценивает, понимаешь ли ты сильные стороны языка.

**Короткий ответ.** Быстрая компиляция, встроенная конкурентность (горутины/каналы), богатая stdlib, статическая линковка, простой деплой, отличные инструменты (go fmt/vet/test).

**Детальный разбор.**

**Конкурентность:**
- Горутины стартуют с ~2KB стека, переключение — наносекунды.
- Каналы и `select` — first-class citizens в языке.
- `sync` пакет для низкоуровневой синхронизации.
- Миллион горутин — норма для Go-приложения.

**Инструментарий:**
- `go build` — компиляция без Makefile/CMake.
- `go test` — тесты, бенчмарки, fuzzing, coverage из коробки.
- `go fmt` — единый стиль кода, нет споров о форматировании.
- `go vet`, `staticcheck` — статический анализ.
- `go mod` — управление зависимостями.

**Деплой:**
- Статически слинкованный бинарник (если CGO_ENABLED=0).
- Минимальный Docker-образ (`FROM scratch` или `FROM alpine`).
- Кросс-компиляция одной командой: `GOOS=linux GOARCH=arm64 go build`.

**Stdlib:**
- HTTP/2 сервер и клиент.
- Криптография (TLS, x509, bcrypt).
- JSON, XML, CSV парсеры.
- Шаблонизатор (`text/template`, `html/template`).
- Профилирование (`net/http/pprof`).

**Пример.**
```go
// Конкурентный HTTP-сервер в несколько строк
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
    })
    http.ListenAndServe(":8080", nil)  // каждый запрос — горутина
}
```

**Типичные ошибки.**
- Думать, что Go быстрее всех — это не Rust/C++, GC добавляет overhead.
- Игнорировать stdlib и тянуть зависимости для базовых задач.

**На интервью.**
- Приведи конкретные примеры из опыта: «На прошлом проекте мы выбрали Go, потому что...»
- Сравни с альтернативами: Java (JVM overhead, медленный старт), Python (GIL, скорость), Rust (сложность, время компиляции).
- Follow-up: «Для каких задач Go не подходит?» — переход к следующему вопросу.

---

### 8. Какие технологические недостатки и ограничения у Go?

**Зачем спрашивают.** Честная оценка инструмента — признак зрелости. Интервьюер проверяет, не фанат ли ты, слепо защищающий любимый язык.

**Короткий ответ.** Скромная система типов (нет sum types, pattern matching), многословная обработка ошибок, GC не подходит для hard real-time, ограниченные функциональные абстракции.

**Детальный разбор.**

**Система типов:**
- Нет sum types / discriminated unions — эмулируем через интерфейсы + type switch.
- Нет pattern matching — только `switch` без деструктуризации.
- Нет наследования — хорошо для composition, плохо для некоторых паттернов.
- Generics появились недавно и пока ограничены.

**Обработка ошибок:**
- `if err != nil` на каждой строке — много boilerplate.
- Нет Result/Either типов — легко забыть проверить ошибку.
- Нет stack traces из коробки — нужны библиотеки или `%+v` с `pkg/errors`.

**Runtime:**
- GC добавляет latency — не подходит для sub-millisecond требований.
- Stop-the-world паузы минимизированы, но не исключены.
- Нет manual memory management (как в Rust/C++).

**Экосистема:**
- GUI — слабо развито.
- ML/Data Science — Python доминирует.
- Мобильная разработка — ограниченная поддержка (gomobile).

**Пример.**
```go
// Многословная обработка ошибок
func processOrder(ctx context.Context, orderID string) error {
    order, err := repo.FindOrder(ctx, orderID)
    if err != nil {
        return fmt.Errorf("find order: %w", err)
    }

    user, err := repo.FindUser(ctx, order.UserID)
    if err != nil {
        return fmt.Errorf("find user: %w", err)
    }

    payment, err := paymentSvc.Charge(ctx, user, order.Amount)
    if err != nil {
        return fmt.Errorf("charge payment: %w", err)
    }

    err = repo.UpdateOrder(ctx, order.ID, payment.ID)
    if err != nil {
        return fmt.Errorf("update order: %w", err)
    }

    return nil
}

// Сравни с Rust:
// let order = repo.find_order(order_id)?;
// let user = repo.find_user(order.user_id)?;
// let payment = payment_svc.charge(&user, order.amount)?;
// repo.update_order(order.id, payment.id)?;
```

**Типичные ошибки.**
- Бороться с языком, пытаясь писать на нём как на Rust/Haskell.
- Выбирать Go для задач, где он не силён (ML, real-time games, GUI).

**На интервью.**
- Покажи, что знаешь workarounds: `errgroup` для параллельных ошибок, structured logging с контекстом.
- Упомяни, что Go команда работает над улучшениями: generics, итераторы (Go 1.23), возможно error handling.
- Follow-up: «Как бы ты улучшил обработку ошибок в Go?» — обсуди proposals, покажи осведомлённость.

---

### 9. Как работает type inference в generics?

**Зачем спрашивают.** Проверяют понимание механики работы компилятора с generic-кодом. Важно знать, когда типы выводятся автоматически, а когда нужно указывать явно.

**Короткий ответ.** Компилятор Go выводит type arguments из аргументов функции. Если тип не может быть выведен из контекста — нужно указать явно. Возвращаемый тип не участвует в inference.

**Детальный разбор.**

Go использует **unification-based type inference**:
1. Компилятор смотрит на типы аргументов функции.
2. Сопоставляет их с параметрами типа в сигнатуре.
3. Если однозначно определяется — подставляет.

**Когда inference работает:**
```go
func Map[T, R any](items []T, fn func(T) R) []R { ... }

// T = int, R = string — выводятся из аргументов
result := Map([]int{1, 2, 3}, func(i int) string {
    return strconv.Itoa(i)
})
```

**Когда inference НЕ работает:**
1. **Тип только в возвращаемом значении:**
   ```go
   func Zero[T any]() T { var z T; return z }

   // Ошибка: cannot infer T
   // x := Zero()

   // Правильно:
   x := Zero[int]()
   ```

2. **Тип в констрейнте, но не в аргументах:**
   ```go
   func New[T Validator]() *T { return new(T) }
   v := New[MyValidator]()  // нужно указать явно
   ```

3. **Неоднозначность:**
   ```go
   func Convert[T, R any](v T) R { ... }
   // r := Convert(42)  // R неизвестен
   r := Convert[int, string](42)  // OK
   ```

**Пример.**
```go
// Type inference в действии
func Filter[T any](items []T, pred func(T) bool) []T {
    result := make([]T, 0, len(items))
    for _, item := range items {
        if pred(item) {
            result = append(result, item)
        }
    }
    return result
}

func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func main() {
    // T = string — выводится из []string
    words := Filter([]string{"go", "rust", "python"}, func(s string) bool {
        return len(s) <= 4
    })  // ["go", "rust"]

    // K = string, V = int — выводится из map[string]int
    m := map[string]int{"a": 1, "b": 2}
    k := Keys(m)  // ["a", "b"]
}
```

**Типичные ошибки.**
- Ожидать, что компилятор выведет тип из возвращаемого значения — Go так не делает.
- Забывать явно указывать типы при вызове конструкторов типа `New[T]()`.
- Путать inference с duck typing — это compile-time механизм.

**На интервью.**
- Объясни, что inference в Go проще, чем в Rust/Haskell — только по аргументам.
- Упомяни, что это design choice для простоты компилятора и читаемости кода.
- Follow-up: «Почему не выводят тип из возвращаемого значения?» — избегание сложности, однозначность.

---

### 10. Что такое build tags и как их использовать?

**Зачем спрашивают.** Build tags — механизм условной компиляции. Важно для кросс-платформенной разработки, feature flags, и разделения production/test кода.

**Короткий ответ.** Build tags — это специальные комментарии `//go:build` (или устаревший `// +build`), которые указывают компилятору, в каких условиях включать файл в сборку. Используются для OS/arch специфичного кода, feature toggles, и отделения интеграционных тестов.

**Детальный разбор.**

**Синтаксис (Go 1.17+):**
```go
//go:build linux && amd64

package mypackage
```

**Старый синтаксис (до Go 1.17):**
```go
// +build linux,amd64

package mypackage
```

**Встроенные теги:**
- OS: `linux`, `darwin`, `windows`, `freebsd`
- Arch: `amd64`, `arm64`, `386`
- Компилятор: `gc`, `gccgo`
- Версия Go: `go1.21`, `go1.22`
- CGO: `cgo`
- Race detector: `race`

**Логические операции:**
```go
//go:build linux || darwin           // OR
//go:build linux && amd64            // AND
//go:build !windows                  // NOT
//go:build (linux || darwin) && cgo  // комбинация
```

**Кастомные теги:**
```go
//go:build integration

package mypackage_test
```
Запуск: `go test -tags=integration ./...`

**Пример.**
```go
// === file: db_prod.go ===
//go:build !mock

package storage

func NewDB(dsn string) (*DB, error) {
    return sql.Open("postgres", dsn)
}

// === file: db_mock.go ===
//go:build mock

package storage

func NewDB(dsn string) (*DB, error) {
    return &DB{mock: true}, nil
}

// === file: syscall_linux.go ===
//go:build linux

package platform

import "syscall"

func GetMemoryInfo() uint64 {
    var info syscall.Sysinfo_t
    syscall.Sysinfo(&info)
    return info.Totalram
}

// === file: syscall_darwin.go ===
//go:build darwin

package platform

func GetMemoryInfo() uint64 {
    // macOS-specific implementation
    return darwinGetPhysMem()
}

// === file: integration_test.go ===
//go:build integration

package api_test

func TestAgainstRealDB(t *testing.T) {
    // Этот тест запускается только с -tags=integration
    db := connectToRealDB()
    defer db.Close()
    // ...
}
```

**Практические применения:**
1. **Платформо-зависимый код** — разные реализации для Linux/macOS/Windows.
2. **Интеграционные тесты** — отделение от unit tests.
3. **Feature flags** — включение экспериментальных функций.
4. **Mock implementations** — подмена для тестов.
5. **Отладочный код** — включение debug-логирования.

**Типичные ошибки.**
- Забыть пустую строку между `//go:build` и `package` — тег не распознается.
- Смешивать старый и новый синтаксис — используй только `//go:build`.
- Не тестировать все комбинации тегов — код может не компилироваться с некоторыми.
- Использовать теги вместо интерфейсов — если можно абстрагировать через интерфейс, это лучше.

**На интервью.**
- Объясни разницу между `//go:build` и `// +build` — новый синтаксис более читаемый и использует стандартные логические операторы.
- Приведи пример использования для интеграционных тестов — очень частый use case.
- Follow-up: «Как посмотреть, какие файлы включены в сборку?» — `go list -f '{{.GoFiles}}' ./...`

---

## См. также

- [Goroutines & Channels](./01-goroutines-channels.md) — конкурентность как ключевая особенность Go
- [Go Runtime](./04-go-runtime.md) — как работает планировщик и GC
- [JVM Fundamentals](../01-java/00-jvm-fundamentals.md) — сравнение runtime моделей Go и Java
- [Python Language Overview](../02-python/00-language-overview.md) — сравнение философий языков

---

## Практика

1. **Императивный vs декларативный** — напиши два варианта фильтрации слайса: императивный цикл и «функциональный» с generic `Filter[T]`. Сравни читаемость.

2. **Type switch** — реализуй функцию `stringify(any) string`, которая обрабатывает `int`, `float64`, `string`, `bool`, `[]any`, `map[string]any` и возвращает JSON-подобную строку.

3. **Interface compliance** — добавь `var _ Interface = (*Type)(nil)` проверки в существующий проект и убедись, что рефакторинг ломает компиляцию в нужном месте.

4. **Embedding vs inheritance** — возьми пример с наследованием из Java/Python и переделай на Go с композицией. Сравни гибкость.

5. **Generics** — реализуй generic `Set[T comparable]` с методами `Add`, `Remove`, `Contains`, `Union`, `Intersection`.

6. **Trade-offs** — составь таблицу «Go vs X» для конкретного типа проекта из своего опыта (микросервис, CLI, data pipeline).

---

## Дополнительные материалы

- [Go Proverbs](https://go-proverbs.github.io/) — философия языка от Rob Pike
- [Go blog: Generics](https://go.dev/blog/intro-generics) — официальное введение
- [Effective Go](https://go.dev/doc/effective_go) — идиоматичный код
- [Why Go?](https://talks.golang.org/2012/splash.article) — Rob Pike о мотивации создания языка
- [golang.org/x/exp/constraints](https://pkg.go.dev/golang.org/x/exp/constraints) — стандартные constraints (до Go 1.21)
- [cmp package](https://pkg.go.dev/cmp) — `Ordered` constraint в Go 1.21+
