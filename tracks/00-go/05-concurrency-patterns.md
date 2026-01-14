# 05 — Concurrency Patterns

Подборка вопросов и ответов о типичных паттернах конкурентного программирования в Go: worker pools, rate limiting, errgroup, буферы и lock-free техники.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [04-go-runtime](./04-go-runtime.md) · Следующая тема: [06-databases-persistence](./06-databases-persistence.md)

---

## Вопросы и разборы

### 1. Как спроектировать worker pool?

**Зачем спрашивают.**
Worker pool — фундаментальный паттерн для параллельной обработки задач с контролем ресурсов. Интервьюер проверяет понимание координации горутин, graceful shutdown и правильного закрытия каналов.

**Короткий ответ.**
Worker pool — это фиксированное число горутин, читающих из общего канала задач. `WaitGroup` отслеживает завершение воркеров. Канал результатов закрывается после завершения всех воркеров.

**Детальный разбор.**

Архитектура worker pool:
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Producer   │────▶│  jobs chan  │────▶│   Workers   │
│ (отправка)  │     │  (буфер N)  │     │ (M штук)    │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌─────────────┐            │
                    │ results chan│◀───────────┘
                    │  (буфер N)  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Consumer   │
                    │ (получение) │
                    └─────────────┘
```

**Ключевые компоненты:**
1. **Канал задач** — входной буфер для работы
2. **Воркеры** — горутины, обрабатывающие задачи
3. **WaitGroup** — ожидание завершения всех воркеров
4. **Канал результатов** — выходной буфер
5. **Context** — для отмены и graceful shutdown

**Пример.**
```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    JobID int
    Value string
    Err   error
}

func NewPool(ctx context.Context, numWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result, numWorkers) // буфер предотвращает блокировку воркеров

    var wg sync.WaitGroup
    wg.Add(numWorkers)

    // Запуск воркеров
    for i := 0; i < numWorkers; i++ {
        go func(workerID int) {
            defer wg.Done()

            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return // канал закрыт, выход
                    }
                    // Обработка задачи
                    result := processJob(job)

                    select {
                    case results <- result:
                    case <-ctx.Done():
                        return
                    }

                case <-ctx.Done():
                    return // отмена контекста
                }
            }
        }(i)
    }

    // Закрытие канала результатов после завершения всех воркеров
    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}

func processJob(job Job) Result {
    // ... обработка
    return Result{JobID: job.ID, Value: "processed: " + job.Data}
}
```

**Использование с graceful shutdown:**
```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Создаём канал задач
    jobs := make(chan Job, 100)

    // Запускаем пул
    results := NewPool(ctx, runtime.NumCPU(), jobs)

    // Producer — отправляет задачи
    go func() {
        defer close(jobs) // важно: закрыть после отправки всех задач
        for i := 0; i < 1000; i++ {
            select {
            case jobs <- Job{ID: i, Data: fmt.Sprintf("task-%d", i)}:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Consumer — собирает результаты
    for result := range results {
        fmt.Printf("Result: %+v\n", result)
    }
}
```

**Типичные ошибки.**
1. **Не закрывать канал задач** — воркеры зависают в ожидании
2. **Закрывать канал результатов до завершения воркеров** — panic при записи
3. **Не обрабатывать отмену контекста** — горутины не завершаются при shutdown
4. **Буфер = 0 без учёта порядка** — deadlock если consumer медленнее producer

**На интервью.**
Нарисуй диаграмму потока данных. Объясни, почему WaitGroup нужен между воркерами и закрытием результатов. Покажи, как graceful shutdown корректно завершает in-flight задачи.

*Частые follow-up вопросы:*
- Как масштабировать пул динамически?
- Как приоритизировать задачи?
- Что если обработка задачи может зависнуть?

---

### 2. Как реализовать rate limiting и backpressure?

**Зачем спрашивают.**
Rate limiting защищает системы от перегрузки. Backpressure — механизм обратной связи, замедляющий producer при медленном consumer. Это критично для production систем.

**Короткий ответ.**
Token bucket — канал с токенами, пополняемый тикером. Consumer берёт токен перед операцией. Backpressure реализуется буферизованным каналом — producer блокируется при заполнении.

**Детальный разбор.**

**Алгоритмы rate limiting:**
```
┌────────────────┬─────────────────────────────────────────┐
│ Алгоритм       │ Характеристика                          │
├────────────────┼─────────────────────────────────────────┤
│ Token Bucket   │ Burst разрешён, средняя скорость        │
│ Leaky Bucket   │ Строго равномерный выход                │
│ Fixed Window   │ Простой, но допускает 2x burst на стыке │
│ Sliding Window │ Точнее fixed, но сложнее реализация     │
└────────────────┴─────────────────────────────────────────┘
```

**Token Bucket визуализация:**
```
      пополнение (rate)
           │
           ▼
    ┌─────────────┐
    │ ○ ○ ○ ○ ○ ○ │  ← bucket (capacity = burst)
    │             │
    └──────┬──────┘
           │
           ▼
      потребление
    (1 токен = 1 запрос)
```

**Пример.**
```go
// Token Bucket с graceful shutdown
func NewRateLimiter(ctx context.Context, rate int, burst int) <-chan struct{} {
    bucket := make(chan struct{}, burst)

    // Начальное заполнение
    for i := 0; i < burst; i++ {
        bucket <- struct{}{}
    }

    go func() {
        ticker := time.NewTicker(time.Second / time.Duration(rate))
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                select {
                case bucket <- struct{}{}:
                    // добавили токен
                default:
                    // bucket полон, пропускаем
                }
            case <-ctx.Done():
                close(bucket)  // graceful shutdown
                return
            }
        }
    }()

    return bucket
}

// Использование
func makeRequest(ctx context.Context, limiter <-chan struct{}) error {
    select {
    case <-limiter:
        // получили токен, выполняем запрос
        return doRequest()
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**Backpressure через буферизованный канал:**
```go
func Producer(ctx context.Context, out chan<- Item) {
    for {
        item := generateItem()

        select {
        case out <- item:
            // успешно отправили
        case <-ctx.Done():
            return
        }
        // Если буфер полон, producer блокируется здесь
        // Это и есть backpressure!
    }
}

func Consumer(ctx context.Context, in <-chan Item) {
    for item := range in {
        process(item)  // медленная обработка
    }
}
```

**Использование golang.org/x/time/rate:**
```go
import "golang.org/x/time/rate"

func main() {
    // 10 запросов в секунду, burst до 5
    limiter := rate.NewLimiter(10, 5)

    for i := 0; i < 100; i++ {
        // Блокируется пока не получит токен
        if err := limiter.Wait(ctx); err != nil {
            return
        }
        makeRequest()
    }
}
```

**Типичные ошибки.**
1. **Не учитывать context в rate limiter** — горутина не завершается
2. **Burst = 0** — первый запрос всегда ждёт
3. **Игнорировать backpressure** — OOM при быстром producer
4. **Rate limit только на клиенте** — сервер всё равно может перегрузиться

**На интервью.**
Объясни разницу между token bucket и leaky bucket. Покажи, как backpressure предотвращает memory exhaustion. Расскажи про distributed rate limiting (Redis + Lua).

*Частые follow-up вопросы:*
- Как реализовать distributed rate limiting?
- Чем отличается client-side от server-side rate limiting?
- Как обрабатывать rate limit exceeded?

---

### 3. Какие инструменты есть для управления группами горутин?

**Зачем спрашивают.**
Координация нескольких горутин — частая задача. Интервьюер проверяет знание sync-примитивов и пакета errgroup для graceful error handling.

**Короткий ответ.**
`sync.WaitGroup` для ожидания завершения горутин. `errgroup.Group` добавляет обработку ошибок и автоматическую отмену при первой ошибке через контекст.

**Детальный разбор.**

**Сравнение подходов:**
```
┌──────────────────┬────────────┬────────────┬───────────────┐
│ Инструмент       │ Ожидание   │ Ошибки     │ Отмена        │
├──────────────────┼────────────┼────────────┼───────────────┤
│ sync.WaitGroup   │ ✓          │ ✗          │ ✗ (вручную)   │
│ errgroup.Group   │ ✓          │ ✓ (первая) │ ✓ (контекст)  │
│ errgroup + Limit │ ✓          │ ✓          │ ✓ + семафор   │
└──────────────────┴────────────┴────────────┴───────────────┘
```

**Пример.**

**sync.WaitGroup — базовый случай:**
```go
func fetchAll(urls []string) {
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            fetch(u)
        }(url)
    }

    wg.Wait()
    // Все горутины завершились
}
```

**errgroup — с обработкой ошибок:**
```go
import "golang.org/x/sync/errgroup"

func fetchAllWithErrors(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)

    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url  // захват переменных (до Go 1.22)

        g.Go(func() error {
            result, err := fetch(ctx, url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            results[i] = result
            return nil
        })
    }

    // Wait блокируется до завершения всех горутин
    // Возвращает первую ошибку (остальные отменяются через ctx)
    if err := g.Wait(); err != nil {
        return err
    }

    process(results)
    return nil
}
```

**errgroup с ограничением параллелизма:**
```go
func fetchWithLimit(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)

    // Ограничение: максимум 10 одновременных запросов
    g.SetLimit(10)

    for _, url := range urls {
        url := url

        g.Go(func() error {
            return fetch(ctx, url)
        })
    }

    return g.Wait()
}
```

**Собственная реализация с семафором:**
```go
import "golang.org/x/sync/semaphore"

func fetchWithSemaphore(ctx context.Context, urls []string, maxConcurrent int64) error {
    sem := semaphore.NewWeighted(maxConcurrent)
    var mu sync.Mutex
    var firstErr error

    var wg sync.WaitGroup

    for _, url := range urls {
        // Захват слота семафора
        if err := sem.Acquire(ctx, 1); err != nil {
            return err
        }

        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            defer sem.Release(1)

            if err := fetch(ctx, u); err != nil {
                mu.Lock()
                if firstErr == nil {
                    firstErr = err
                }
                mu.Unlock()
            }
        }(url)
    }

    wg.Wait()
    return firstErr
}
```

**Типичные ошибки.**
1. **wg.Add внутри горутины** — race condition, Wait может вернуться раньше
2. **Не передавать ctx в горутины** — они не отменяются при ошибке
3. **Доступ к общим данным без синхронизации** — в примере выше results[i] безопасен, т.к. разные индексы
4. **Забывать про SetLimit** — 10k URL = 10k горутин

**На интервью.**
Покажи разницу между WaitGroup и errgroup на диаграмме. Объясни, как errgroup.WithContext отменяет оставшиеся горутины. Когда нужен SetLimit?

*Частые follow-up вопросы:*
- Как собрать все ошибки, а не только первую?
- Что если горутина игнорирует ctx.Done()?
- Как добавить timeout на всю группу?

---

### 4. Как обнаруживать и предотвращать deadlock?

**Зачем спрашивают.**
Deadlock — серьёзная проблема в concurrent коде. Интервьюер оценивает понимание причин deadlock и методов их предотвращения.

**Короткий ответ.**
Deadlock возникает при циклическом ожидании ресурсов. Предотвращение: единый порядок захвата locks, таймауты на операции, избегание вложенных блокировок.

**Детальный разбор.**

**Условия возникновения deadlock (все 4 одновременно):**
```
1. Mutual Exclusion   — ресурс эксклюзивен
2. Hold and Wait      — держим один ресурс, ждём другой
3. No Preemption      — ресурс нельзя отобрать
4. Circular Wait      — A ждёт B, B ждёт A
```

**Классический пример deadlock:**
```go
// ПЛОХО: deadlock при одновременном вызове
var muA, muB sync.Mutex

func funcA() {
    muA.Lock()
    defer muA.Unlock()

    time.Sleep(time.Millisecond) // имитация работы

    muB.Lock()       // ← ждёт muB
    defer muB.Unlock()
}

func funcB() {
    muB.Lock()
    defer muB.Unlock()

    time.Sleep(time.Millisecond)

    muA.Lock()       // ← ждёт muA → DEADLOCK!
    defer muA.Unlock()
}
```

**Пример.**

**Решение 1: Единый порядок захвата:**
```go
// ХОРОШО: всегда сначала muA, потом muB
func funcA() {
    muA.Lock()
    defer muA.Unlock()

    muB.Lock()
    defer muB.Unlock()

    // работа
}

func funcB() {
    muA.Lock()       // тот же порядок!
    defer muA.Unlock()

    muB.Lock()
    defer muB.Unlock()

    // работа
}
```

**Решение 2: Try-lock с таймаутом:**
```go
func safeSend(ctx context.Context, ch chan<- int, v int) error {
    select {
    case ch <- v:
        return nil
    case <-time.After(time.Second):
        return errors.New("timeout: channel blocked")
    case <-ctx.Done():
        return ctx.Err()
    }
}

func safeReceive(ctx context.Context, ch <-chan int) (int, error) {
    select {
    case v := <-ch:
        return v, nil
    case <-time.After(time.Second):
        return 0, errors.New("timeout: channel empty")
    case <-ctx.Done():
        return 0, ctx.Err()
    }
}
```

**Решение 3: Избегание вложенных locks:**
```go
// ПЛОХО: вложенный lock
func (c *Cache) GetOrCreate(key string) *Item {
    c.mu.Lock()
    defer c.mu.Unlock()

    if item, ok := c.items[key]; ok {
        return item
    }

    // create() может захватить другой lock → potential deadlock
    item := c.create(key)
    c.items[key] = item
    return item
}

// ХОРОШО: минимизация области блокировки
func (c *Cache) GetOrCreate(key string) *Item {
    c.mu.RLock()
    if item, ok := c.items[key]; ok {
        c.mu.RUnlock()
        return item
    }
    c.mu.RUnlock()

    // create() вне lock
    item := c.create(key)

    c.mu.Lock()
    defer c.mu.Unlock()

    // Double-check после получения write lock
    if existing, ok := c.items[key]; ok {
        return existing
    }
    c.items[key] = item
    return item
}
```

**Обнаружение deadlock:**
```go
// Go runtime обнаруживает простые deadlocks
func main() {
    ch := make(chan int)
    <-ch  // fatal error: all goroutines are asleep - deadlock!
}

// Для сложных случаев — профилирование
// curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

**Типичные ошибки.**
1. **Вызов внешнего кода под lock** — неизвестно, что он захватывает
2. **Lock внутри callback** — порядок вызовов непредсказуем
3. **defer Unlock без проверки** — паника до Lock = Unlock без Lock
4. **Игнорирование таймаутов** — вечное ожидание

**На интервью.**
Нарисуй циклическую зависимость, вызывающую deadlock. Объясни, как единый порядок разрывает цикл. Покажи пример обнаружения через pprof goroutine dump.

*Частые follow-up вопросы:*
- Как найти deadlock в production?
- Что такое livelock?
- Как избежать deadlock при работе с БД?

---

### 5. Что такое pipeline и fan-in/fan-out?

**Зачем спрашивают.**
Pipeline — основа потоковой обработки данных. Fan-in/fan-out позволяют параллелизовать узкие места. Это ключевые паттерны для высоконагруженных систем.

**Короткий ответ.**
Pipeline — цепочка стадий, связанных каналами. Fan-out — распараллеливание одной стадии на несколько воркеров. Fan-in — объединение нескольких каналов в один.

**Детальный разбор.**

**Архитектура pipeline:**
```
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│ Source │───▶│ Stage1 │───▶│ Stage2 │───▶│  Sink  │
│        │    │ parse  │    │ process│    │        │
└────────┘    └────────┘    └────────┘    └────────┘

     chan         chan         chan
    ─────▶      ─────▶       ─────▶
```

**Fan-out / Fan-in:**
```
                    ┌─────────┐
                ┌──▶│ worker1 │──┐
                │   └─────────┘  │
┌────────┐      │   ┌─────────┐  │    ┌────────┐
│ Source │──────┼──▶│ worker2 │──┼───▶│ Fan-in │
└────────┘      │   └─────────┘  │    └────────┘
   fan-out      │   ┌─────────┐  │
                └──▶│ worker3 │──┘
                    └─────────┘
```

**Пример.**

**Простой pipeline:**
```go
// Генератор — источник данных
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Стадия — трансформация
func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Использование
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Pipeline: generate -> square -> square
    nums := generate(ctx, 1, 2, 3, 4, 5)
    squared := square(ctx, nums)
    result := square(ctx, squared)

    for v := range result {
        fmt.Println(v)  // 1, 16, 81, 256, 625
    }
}
```

**Fan-out / Fan-in:**
```go
// Fan-out: запуск N воркеров на одном input канале
func fanOut(ctx context.Context, in <-chan int, n int) []<-chan int {
    outs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        outs[i] = worker(ctx, in)
    }
    return outs
}

func worker(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- heavyProcess(v):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Fan-in: объединение нескольких каналов в один
func fanIn(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    wg.Add(len(channels))
    for _, ch := range channels {
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                select {
                case out <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

// Использование
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    in := generate(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    // Fan-out на 3 воркера
    workers := fanOut(ctx, in, 3)

    // Fan-in обратно в один канал
    results := fanIn(ctx, workers...)

    for v := range results {
        fmt.Println(v)
    }
}
```

**Типичные ошибки.**
1. **Не закрывать каналы** — downstream стадии зависают
2. **Не обрабатывать ctx.Done()** — pipeline не останавливается
3. **Fan-in без WaitGroup** — out закрывается преждевременно
4. **Unbuffered каналы везде** — низкая пропускная способность

**На интервью.**
Нарисуй pipeline с fan-out на CPU-intensive стадии. Объясни, как пропагируется отмена через контекст. Покажи, где добавить буферизацию для throughput.

*Частые follow-up вопросы:*
- Как обрабатывать ошибки в pipeline?
- Как добавить backpressure?
- Когда fan-out не помогает?

---

### 6. Как соединить каналы с `select` и `default` для неблокирующих операций?

**Зачем спрашивают.**
`select` — основа мультиплексирования в Go. Интервьюер проверяет понимание приоритетов, неблокирующих операций и типичных паттернов.

**Короткий ответ.**
`select` ждёт готовности любого из case. `default` делает операцию неблокирующей. При нескольких готовых case выбор случайный (для fairness).

**Детальный разбор.**

**Поведение select:**
```
┌──────────────────────────────────────────────────────────┐
│ select {                                                 │
│     case v := <-ch1:  // готов?                         │
│     case ch2 <- x:    // готов?                         │
│     case <-ctx.Done(): // готов?                        │
│     default:          // выполнить сразу, если никто    │
│ }                                                        │
├──────────────────────────────────────────────────────────┤
│ • Если ничего не готово и нет default — блокировка      │
│ • Если несколько готовы — случайный выбор               │
│ • nil канал в case — этот case игнорируется             │
└──────────────────────────────────────────────────────────┘
```

**Пример.**

**Неблокирующее чтение/запись:**
```go
// Неблокирующее чтение
select {
case v := <-ch:
    process(v)
default:
    // канал пуст, делаем что-то другое
}

// Неблокирующая запись
select {
case ch <- value:
    // успешно отправили
default:
    // буфер полон, отбрасываем или логируем
    log.Println("channel full, dropping message")
}
```

**Heartbeat паттерн:**
```go
func worker(ctx context.Context, tasks <-chan Task, heartbeat chan<- struct{}) {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case task := <-tasks:
            process(task)

        case <-ticker.C:
            // Отправляем heartbeat, если получатель готов
            select {
            case heartbeat <- struct{}{}:
            default:
                // получатель занят, пропускаем
            }

        case <-ctx.Done():
            return
        }
    }
}
```

**Приоритетный select (псевдо-приоритет):**
```go
func prioritySelect(high, low <-chan int) {
    for {
        // Сначала проверяем высокий приоритет
        select {
        case v := <-high:
            handleHighPriority(v)
            continue
        default:
        }

        // Затем ждём любой
        select {
        case v := <-high:
            handleHighPriority(v)
        case v := <-low:
            handleLowPriority(v)
        }
    }
}
```

**Timeout паттерн:**
```go
func fetchWithTimeout(ctx context.Context, url string) ([]byte, error) {
    result := make(chan []byte, 1)
    errCh := make(chan error, 1)

    go func() {
        data, err := fetch(url)
        if err != nil {
            errCh <- err
            return
        }
        result <- data
    }()

    select {
    case data := <-result:
        return data, nil
    case err := <-errCh:
        return nil, err
    case <-time.After(5 * time.Second):
        return nil, errors.New("timeout")
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

**Динамическое отключение case через nil:**
```go
func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for ch1 != nil || ch2 != nil {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil  // отключаем этот case
                    continue
                }
                out <- v

            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                    continue
                }
                out <- v
            }
        }
    }()

    return out
}
```

**Типичные ошибки.**
1. **default без необходимости** — busy loop, 100% CPU
2. **Игнорирование ok при чтении** — не отличаем закрытый канал от zero value
3. **Ожидание приоритетов** — select выбирает случайно среди готовых
4. **time.After в цикле** — memory leak (таймер не освобождается)

**На интервью.**
Объясни, почему select не гарантирует порядок. Покажи, как избежать memory leak с time.After. Продемонстрируй паттерн "try send".

*Частые follow-up вопросы:*
- Как реализовать приоритетную очередь с каналами?
- Почему nil канал блокирует select case?
- Как избежать busy loop с default?

---

### 7. Можно ли использовать один и тот же буфер `[]byte` в нескольких горутинах?

**Зачем спрашивают.**
Совместное использование памяти — источник data races. Интервьюер проверяет понимание модели памяти Go и безопасных паттернов работы с буферами.

**Короткий ответ.**
Только для чтения. Для записи необходима либо копия буфера, либо синхронизация через mutex. Для переиспользования буферов — `sync.Pool`.

**Детальный разбор.**

**Модель памяти Go:**
```
┌─────────────────────────────────────────────────────────┐
│ Безопасно без синхронизации:                            │
│ • Чтение неизменяемых данных                            │
│ • Запись в разные индексы слайса                        │
│ • Atomic операции                                       │
├─────────────────────────────────────────────────────────┤
│ Требует синхронизации:                                  │
│ • Чтение и запись одних данных                          │
│ • Изменение длины/capacity слайса                       │
│ • Изменение указателя в интерфейсе                      │
└─────────────────────────────────────────────────────────┘
```

**Пример.**

**Небезопасно — data race:**
```go
func bad() {
    buf := []byte("data")

    go func() {
        buf[0] = 'D'  // запись
    }()

    go func() {
        fmt.Println(buf)  // чтение — RACE!
    }()
}
```

**Безопасно — копия:**
```go
func good() {
    buf := []byte("data")

    go func(b []byte) {
        b[0] = 'D'  // своя копия
    }(append([]byte(nil), buf...))

    go func(b []byte) {
        fmt.Println(b)  // своя копия
    }(append([]byte(nil), buf...))
}
```

**Безопасно — sync.Pool для переиспользования:**
```go
var bufPool = sync.Pool{
    New: func() any {
        buf := make([]byte, 0, 4096)
        return &buf
    },
}

func process(data []byte) {
    // Получаем буфер из пула
    bufPtr := bufPool.Get().(*[]byte)
    buf := (*bufPtr)[:0]  // сброс длины, сохранение capacity

    // Используем буфер
    buf = append(buf, data...)
    buf = transform(buf)

    // Возвращаем в пул
    *bufPtr = buf
    bufPool.Put(bufPtr)
}

func transform(buf []byte) []byte {
    // ... трансформация
    return buf
}
```

**Безопасно — mutex:**
```go
type SafeBuffer struct {
    mu  sync.RWMutex
    buf []byte
}

func (b *SafeBuffer) Write(data []byte) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.buf = append(b.buf, data...)
}

func (b *SafeBuffer) Read() []byte {
    b.mu.RLock()
    defer b.mu.RUnlock()
    // Возвращаем копию!
    return append([]byte(nil), b.buf...)
}
```

**Типичные ошибки.**
1. **Возврат slice без копии** — получатель может модифицировать оригинал
2. **sync.Pool с указателем на slice** — slice header копируется, но underlying array общий
3. **bytes.Buffer без сброса** — данные накапливаются
4. **Передача slice в горутину без копии** — race condition

**На интервью.**
Объясни разницу между slice header и underlying array. Покажи, как `append([]byte(nil), buf...)` создаёт копию. Расскажи про escape analysis и когда буфер попадает в heap.

*Частые follow-up вопросы:*
- Почему sync.Pool может "терять" объекты?
- Как избежать аллокаций при работе с буферами?
- Когда использовать bytes.Buffer vs []byte?

---

### 8. Какие типы мьютексов предоставляет stdlib?

**Зачем спрашивают.**
Правильный выбор синхронизации критичен для производительности. Интервьюер проверяет знание примитивов и понимание, когда какой использовать.

**Короткий ответ.**
`sync.Mutex` — эксклюзивная блокировка. `sync.RWMutex` — разделение читателей/писателей. `sync.Once` — однократная инициализация. `sync.Cond` — условные переменные. `sync/atomic` — lock-free операции.

**Детальный разбор.**

**Сравнение примитивов:**
```
┌─────────────────┬────────────────────────────────────────┐
│ Примитив        │ Применение                             │
├─────────────────┼────────────────────────────────────────┤
│ sync.Mutex      │ Эксклюзивный доступ, короткие секции   │
│ sync.RWMutex    │ Много читателей, редкие записи         │
│ sync.Once       │ Ленивая инициализация singleton        │
│ sync.Cond       │ Ожидание условия (producer/consumer)   │
│ sync.Map        │ Конкурентный map (read-heavy)          │
│ sync/atomic     │ Простые операции без блокировок        │
│ sync.Pool       │ Переиспользование объектов             │
└─────────────────┴────────────────────────────────────────┘
```

**Пример.**

**sync.Mutex — базовая блокировка:**
```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

**sync.RWMutex — оптимизация для read-heavy:**
```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

func (c *Cache) Get(key string) (Item, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[key]
    return item, ok
}

func (c *Cache) Set(key string, item Item) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = item
}
```

**sync.Once — безопасная инициализация:**
```go
var (
    instance *Database
    once     sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        instance = &Database{}
        instance.Connect()
    })
    return instance
}
```

**sync.Cond — ожидание условия:**
```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []int
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Push(item int) {
    q.mu.Lock()
    defer q.mu.Unlock()
    q.items = append(q.items, item)
    q.cond.Signal()  // разбудить одного ожидающего
}

func (q *Queue) Pop() int {
    q.mu.Lock()
    defer q.mu.Unlock()

    for len(q.items) == 0 {
        q.cond.Wait()  // освобождает lock и ждёт Signal
    }

    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

**sync/atomic — без блокировок:**
```go
type AtomicCounter struct {
    value atomic.Int64
}

func (c *AtomicCounter) Inc() {
    c.value.Add(1)
}

func (c *AtomicCounter) Value() int64 {
    return c.value.Load()
}

// atomic.Pointer для lock-free структур
type Config struct {
    ptr atomic.Pointer[Settings]
}

func (c *Config) Update(s *Settings) {
    c.ptr.Store(s)
}

func (c *Config) Get() *Settings {
    return c.ptr.Load()
}
```

**Типичные ошибки.**
1. **RWMutex при частых записях** — хуже обычного Mutex из-за overhead
2. **Копирование Mutex** — UB, используй указатель или embed
3. **Lock без Unlock** — забыть defer в функции с early return
4. **Cond.Wait без цикла** — spurious wakeup

**На интервью.**
Объясни, когда RWMutex выигрывает у Mutex (много читателей). Покажи, как Once гарантирует однократное выполнение. Расскажи про memory ordering в atomic.

*Частые follow-up вопросы:*
- Почему нельзя копировать Mutex?
- Что такое spurious wakeup?
- Когда sync.Map лучше map + RWMutex?

---

### 9. Что такое lock-free структуры данных и есть ли они в Go?

**Зачем спрашивают.**
Lock-free алгоритмы — advanced тема, показывающая глубокое понимание concurrency. Интервьюер оценивает знание атомарных операций и их ограничений.

**Короткий ответ.**
Lock-free структуры используют атомарные операции (CAS) вместо блокировок. В Go — `sync/atomic`. Планировщик Go использует lock-free очереди (work-stealing). Писать свои lock-free структуры сложно и редко нужно.

**Детальный разбор.**

**Гарантии прогресса:**
```
┌────────────────┬─────────────────────────────────────────┐
│ Тип            │ Гарантия                                │
├────────────────┼─────────────────────────────────────────┤
│ Blocking       │ Один поток может заблокировать все      │
│ Lock-free      │ Хотя бы один поток прогрессирует        │
│ Wait-free      │ Каждый поток прогрессирует              │
└────────────────┴─────────────────────────────────────────┘
```

**CAS (Compare-And-Swap) — основа lock-free:**
```
┌──────────────────────────────────────────────────────────┐
│ CompareAndSwap(addr, old, new):                          │
│   if *addr == old:                                       │
│       *addr = new                                        │
│       return true                                        │
│   return false                                           │
│                                                          │
│ Выполняется атомарно на уровне CPU                      │
└──────────────────────────────────────────────────────────┘
```

**Пример.**

**Lock-free счётчик:**
```go
type LockFreeCounter struct {
    value atomic.Int64
}

func (c *LockFreeCounter) Inc() int64 {
    return c.value.Add(1)
}

func (c *LockFreeCounter) Value() int64 {
    return c.value.Load()
}
```

**Lock-free stack (LIFO):**
```go
type node[T any] struct {
    value T
    next  *node[T]
}

type LockFreeStack[T any] struct {
    head atomic.Pointer[node[T]]
}

func (s *LockFreeStack[T]) Push(value T) {
    newNode := &node[T]{value: value}

    for {
        oldHead := s.head.Load()
        newNode.next = oldHead

        // CAS: если head не изменился, подменяем
        if s.head.CompareAndSwap(oldHead, newNode) {
            return
        }
        // Иначе retry — кто-то успел раньше
    }
}

func (s *LockFreeStack[T]) Pop() (T, bool) {
    for {
        oldHead := s.head.Load()
        if oldHead == nil {
            var zero T
            return zero, false
        }

        newHead := oldHead.next
        if s.head.CompareAndSwap(oldHead, newHead) {
            return oldHead.value, true
        }
        // Retry
    }
}
```

**Atomic pointer для конфигурации:**
```go
type Config struct {
    Settings atomic.Pointer[Settings]
}

type Settings struct {
    MaxConnections int
    Timeout        time.Duration
}

func (c *Config) Update(s *Settings) {
    c.Settings.Store(s)  // атомарная замена
}

func (c *Config) Get() *Settings {
    return c.Settings.Load()
}

// Использование — без блокировок
func handler(c *Config) {
    settings := c.Get()  // всегда консистентный snapshot
    // используем settings
}
```

**Lock-free в runtime Go:**
```go
// Планировщик использует lock-free work-stealing deque
// для распределения горутин между P

// sync.Pool внутри использует lock-free списки
// для хранения объектов per-P
```

**Типичные ошибки.**
1. **ABA проблема** — значение изменилось и вернулось, CAS не заметил
2. **Memory ordering** — нужны правильные барьеры памяти
3. **Преждевременная оптимизация** — mutex часто достаточно
4. **Сложность отладки** — race conditions труднее найти

**На интервью.**
Объясни CAS и почему он атомарный. Покажи ABA проблему на примере. Расскажи, почему lock-free не всегда быстрее (contention, cache invalidation).

*Частые follow-up вопросы:*
- Что такое ABA проблема и как её решить?
- Почему lock-free структуры сложно писать правильно?
- Когда mutex лучше atomic?

---

### 10. Как реализовать Circuit Breaker?

**Зачем спрашивают.**
Circuit Breaker — критически важный паттерн для устойчивости микросервисов. Интервьюер проверяет понимание failure handling и graceful degradation.

**Короткий ответ.**
Circuit Breaker отслеживает ошибки зависимости. При превышении порога переходит в состояние Open и сразу возвращает ошибку без вызова. После таймаута переходит в Half-Open для пробного запроса. При успехе — Closed, при ошибке — снова Open.

**Детальный разбор.**

**Состояния Circuit Breaker:**
```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│    ┌────────┐   errors > threshold   ┌────────┐             │
│    │ CLOSED │ ─────────────────────▶ │  OPEN  │             │
│    │        │                        │        │             │
│    └───▲────┘                        └───┬────┘             │
│        │                                 │                   │
│        │ success                         │ timeout           │
│        │                                 ▼                   │
│        │                           ┌──────────┐             │
│        └─────────────── success ── │HALF-OPEN │             │
│                                    │          │             │
│           ┌────────────────────────│ (1 req)  │             │
│           │ error                  └──────────┘             │
│           ▼                                                  │
│        ┌────────┐                                           │
│        │  OPEN  │                                           │
│        └────────┘                                           │
└──────────────────────────────────────────────────────────────┘
```

**Параметры:**
- `threshold` — количество ошибок для открытия
- `timeout` — время в Open до перехода в Half-Open
- `successThreshold` — количество успехов в Half-Open для закрытия

**Пример.**
```go
type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    mu sync.RWMutex

    state           State
    failures        int
    successes       int
    lastFailureTime time.Time

    // Настройки
    maxFailures     int
    timeout         time.Duration
    successThreshold int
}

func NewCircuitBreaker(maxFailures int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:           StateClosed,
        maxFailures:     maxFailures,
        timeout:         timeout,
        successThreshold: 1,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if !cb.AllowRequest() {
        return errors.New("circuit breaker is open")
    }

    err := fn()

    cb.RecordResult(err == nil)
    return err
}

func (cb *CircuitBreaker) AllowRequest() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateClosed:
        return true

    case StateOpen:
        // Проверяем, прошёл ли timeout
        if time.Since(cb.lastFailureTime) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            return true
        }
        return false

    case StateHalfOpen:
        return true // разрешаем пробный запрос
    }

    return false
}

func (cb *CircuitBreaker) RecordResult(success bool) {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateClosed:
        if success {
            cb.failures = 0
        } else {
            cb.failures++
            cb.lastFailureTime = time.Now()
            if cb.failures >= cb.maxFailures {
                cb.state = StateOpen
            }
        }

    case StateHalfOpen:
        if success {
            cb.successes++
            if cb.successes >= cb.successThreshold {
                cb.state = StateClosed
                cb.failures = 0
            }
        } else {
            cb.state = StateOpen
            cb.lastFailureTime = time.Now()
        }
    }
}

// Использование
func main() {
    cb := NewCircuitBreaker(5, 30*time.Second)

    err := cb.Execute(func() error {
        return callExternalService()
    })

    if err != nil {
        // Circuit open или ошибка сервиса
        return fallbackResponse()
    }
}
```

**Использование библиотеки (sony/gobreaker):**
```go
import "github.com/sony/gobreaker"

var cb *gobreaker.CircuitBreaker

func init() {
    settings := gobreaker.Settings{
        Name:        "external-service",
        MaxRequests: 1,                    // в Half-Open
        Interval:    10 * time.Second,     // сброс счётчика
        Timeout:     30 * time.Second,     // Open → Half-Open
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            log.Printf("Circuit %s: %s → %s", name, from, to)
        },
    }
    cb = gobreaker.NewCircuitBreaker(settings)
}

func CallService(ctx context.Context) (string, error) {
    result, err := cb.Execute(func() (any, error) {
        return externalService.Call(ctx)
    })

    if err != nil {
        return "", err
    }
    return result.(string), nil
}
```

**Типичные ошибки.**
1. **Слишком низкий threshold** — circuit открывается от единичных ошибок
2. **Слишком короткий timeout** — не даёт сервису восстановиться
3. **Нет fallback** — circuit breaker бесполезен без альтернативы
4. **Один breaker на все endpoints** — должен быть per-dependency

**На интервью.**
Нарисуй state machine. Объясни, зачем Half-Open состояние (проверка восстановления без шторма запросов). Расскажи про метрики: error rate, response time.

*Частые follow-up вопросы:*
- Как настроить пороги для конкретного сервиса?
- Чем Circuit Breaker отличается от retry?
- Как тестировать Circuit Breaker?

---

### 11. Что такое Bulkhead pattern?

**Зачем спрашивают.**
Bulkhead изолирует компоненты системы, предотвращая каскадные сбои. Интервьюер проверяет понимание resilience patterns и resource management.

**Короткий ответ.**
Bulkhead (переборка) разделяет ресурсы между компонентами. Если один компонент перегружен или сломан, он не истощает общие ресурсы. В Go реализуется через отдельные worker pools, semaphores или connection pools для каждой зависимости.

**Детальный разбор.**

**Аналогия с кораблём:**
```
┌──────────────────────────────────────────────────────────────┐
│ Корабль без переборок:                                       │
│ ┌────────────────────────────────────────────────────────┐  │
│ │                    одна полость                         │  │
│ │              пробоина → весь корабль тонет             │  │
│ └────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│ Корабль с переборками (bulkheads):                           │
│ ┌─────────┬─────────┬─────────┬─────────┬─────────┐        │
│ │ отсек 1 │ отсек 2 │ отсек 3 │ отсек 4 │ отсек 5 │        │
│ │         │ ЗАТОП-  │         │         │         │        │
│ │         │  ЛЕН    │         │         │         │        │
│ └─────────┴─────────┴─────────┴─────────┴─────────┘        │
│           пробоина → только один отсек                      │
└──────────────────────────────────────────────────────────────┘
```

**Применение в микросервисах:**
```
┌────────────────────────────────────────────────────────────┐
│                       Service A                             │
│  ┌─────────────────┐  ┌─────────────────┐                 │
│  │ Pool: Service B │  │ Pool: Service C │                 │
│  │ max: 20 conns   │  │ max: 10 conns   │                 │
│  │ ████████░░░░░░  │  │ ██████████      │ ← saturated    │
│  └─────────────────┘  └─────────────────┘                 │
│           │                    │                           │
│  Service B медленный,  Service C не затронут              │
│  но не блокирует C                                        │
└────────────────────────────────────────────────────────────┘
```

**Пример.**

**Semaphore-based Bulkhead:**
```go
type Bulkhead struct {
    name string
    sem  chan struct{}
}

func NewBulkhead(name string, maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        name: name,
        sem:  make(chan struct{}, maxConcurrent),
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()

    case <-ctx.Done():
        return fmt.Errorf("bulkhead %s: %w", b.name, ctx.Err())

    default:
        return fmt.Errorf("bulkhead %s: rejected (at capacity)", b.name)
    }
}

func (b *Bulkhead) ExecuteWithWait(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()

    case <-ctx.Done():
        return fmt.Errorf("bulkhead %s: %w", b.name, ctx.Err())
    }
}
```

**Изолированные worker pools:**
```go
type ServiceClient struct {
    bulkheadPayments  *Bulkhead
    bulkheadInventory *Bulkhead
    bulkheadNotify    *Bulkhead
}

func NewServiceClient() *ServiceClient {
    return &ServiceClient{
        // Разные лимиты для разных зависимостей
        bulkheadPayments:  NewBulkhead("payments", 20),   // критичный
        bulkheadInventory: NewBulkhead("inventory", 50),  // высокий throughput
        bulkheadNotify:    NewBulkhead("notify", 10),     // некритичный
    }
}

func (c *ServiceClient) ProcessPayment(ctx context.Context, order Order) error {
    return c.bulkheadPayments.Execute(ctx, func() error {
        return paymentService.Charge(ctx, order)
    })
}

func (c *ServiceClient) CheckInventory(ctx context.Context, items []Item) error {
    return c.bulkheadInventory.Execute(ctx, func() error {
        return inventoryService.Check(ctx, items)
    })
}

func (c *ServiceClient) SendNotification(ctx context.Context, msg Message) error {
    // Уведомления некритичны — используем ExecuteWithWait
    return c.bulkheadNotify.ExecuteWithWait(ctx, func() error {
        return notifyService.Send(ctx, msg)
    })
}
```

**Bulkhead + Circuit Breaker:**
```go
type ResilientClient struct {
    bulkhead *Bulkhead
    breaker  *CircuitBreaker
}

func (c *ResilientClient) Call(ctx context.Context, req Request) (Response, error) {
    var resp Response

    err := c.bulkhead.Execute(ctx, func() error {
        return c.breaker.Execute(func() error {
            var err error
            resp, err = c.doCall(ctx, req)
            return err
        })
    })

    return resp, err
}
```

**Метрики для мониторинга:**
```go
type BulkheadMetrics struct {
    name           string
    maxConcurrency int
    sem            chan struct{}

    // Метрики
    activeCount    atomic.Int64
    rejectedCount  atomic.Int64
    completedCount atomic.Int64
}

func (b *BulkheadMetrics) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        b.activeCount.Add(1)
        defer func() {
            <-b.sem
            b.activeCount.Add(-1)
            b.completedCount.Add(1)
        }()
        return fn()

    default:
        b.rejectedCount.Add(1)
        return ErrBulkheadFull
    }
}

func (b *BulkheadMetrics) Stats() map[string]int64 {
    return map[string]int64{
        "active":    b.activeCount.Load(),
        "rejected":  b.rejectedCount.Load(),
        "completed": b.completedCount.Load(),
        "available": int64(b.maxConcurrency) - b.activeCount.Load(),
    }
}
```

**Типичные ошибки.**
1. **Общий pool для всех зависимостей** — медленная зависимость блокирует все
2. **Слишком маленький bulkhead** — законные запросы отклоняются
3. **Нет fallback при rejection** — клиент получает ошибку без альтернативы
4. **Игнорирование метрик** — не видно, когда bulkhead переполнен

**На интервью.**
Объясни аналогию с корабельными переборками. Покажи, как semaphore изолирует ресурсы. Расскажи, как выбрать размер bulkhead (Little's Law: L = λW).

*Частые follow-up вопросы:*
- Как определить оптимальный размер bulkhead?
- Чем Bulkhead отличается от Rate Limiter?
- Как комбинировать Bulkhead с Circuit Breaker?

---

## См. также

- [Goroutines & Channels](./01-goroutines-channels.md) — базовые примитивы конкурентности
- [Errors & Context](./03-errors-context.md) — отмена и таймауты
- [Rate Limiter](../03-system-design/02-rate-limiter.md) — применение в System Design
- [Resilience Patterns](../08-architecture/08-resilience-patterns.md) — паттерны отказоустойчивости
- [Distributed Cache](../03-system-design/07-distributed-cache.md) — кеширование с конкурентным доступом

---

## Практика

1. **Worker Pool:** Реализуй worker pool с ограничением одновременных задач, graceful shutdown через context и метриками (обработано, в очереди).

2. **Rate Limiter:** Напиши token bucket с distributed state (Redis) и сравни с локальной версией в бенчмарках.

3. **Pipeline:** Построй pipeline для обработки файлов: чтение → парсинг → трансформация → запись. Добавь errgroup для отмены при ошибке.

4. **Deadlock Detection:** Смоделируй deadlock на трёх mutex и покажи, как pprof goroutine dump помогает найти проблему.

5. **Lock-free Stack:** Реализуй lock-free stack на atomic.Pointer, напиши тесты с -race и бенчмарки vs mutex.

6. **sync.Pool Manager:** Создай пул буферов с метриками (hits/misses), сравни производительность с новыми аллокациями.

---

## Дополнительные материалы

- [Go Concurrency Patterns](https://go.dev/blog/pipelines)
- [Advanced Go Concurrency Patterns](https://go.dev/blog/io2013-talk-concurrency)
- [golang.org/x/sync](https://pkg.go.dev/golang.org/x/sync) — errgroup, semaphore, singleflight
- [The Go Memory Model](https://go.dev/ref/mem)
- Kavya Joshi — "Understanding Channels" (GopherCon 2017)
- Bryan Mills — "Rethinking Classical Concurrency Patterns" (GopherCon 2018)
