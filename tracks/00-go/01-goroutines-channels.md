# 01 — Goroutines & Channels

Развёрнутые вопросы и ответы про горутины и каналы: запуск и остановка, буферизация, `select`, закрытие, утечки, коммуникационные паттерны. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [00-language-overview](./00-language-overview.md) · Следующая тема: [02-memory-slices-maps](./02-memory-slices-maps.md)

---

## Вопросы и разборы

### 1. Как запустить горутину и контролировать её жизненный цикл?

**Зачем спрашивают.** Запустить `go func()` просто, но без синхронизации горутина может завершиться раньше или позже основного потока, оставив ресурсы висеть. Интервьюер проверяет умение управлять горутинами.

**Короткий ответ.** Комбинируй `sync.WaitGroup` для ожидания завершения и `context.Context` для сигнала отмены. `WaitGroup.Add` перед запуском, `defer Done()` внутри горутины, `Wait()` для блокировки.

**Детальный разбор.**

**Проблема «fire and forget»:**
```go
func main() {
    go doWork()  // может не успеть выполниться
}
// main завершился → процесс умер → горутина не отработала
```

**Паттерн: WaitGroup + Context:**
```
                    ┌─────────────────┐
                    │     main()      │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
    │ Worker1 │         │ Worker2 │         │ Worker3 │
    │ wg.Done │         │ wg.Done │         │ wg.Done │
    └────┬────┘         └────┬────┘         └────┬────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                    ┌────────▼────────┐
                    │    wg.Wait()    │
                    └─────────────────┘
```

**Компоненты:**
- `sync.WaitGroup` — счётчик активных горутин
- `context.Context` — сигнал отмены для graceful shutdown
- `defer wg.Done()` — гарантирует декремент даже при panic

**Пример.**
```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // важно: освобождает ресурсы context

    var wg sync.WaitGroup

    // Запуск воркеров
    for i := 0; i < 5; i++ {
        wg.Add(1) // ДО запуска горутины!
        go func(id int) {
            defer wg.Done()
            worker(ctx, id)
        }(i)
    }

    // Даём поработать
    time.Sleep(100 * time.Millisecond)

    // Сигнал остановки
    cancel()

    // Ждём завершения всех воркеров
    wg.Wait()
    fmt.Println("All workers stopped")
}

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: stopping\n", id)
            return
        default:
            fmt.Printf("Worker %d: working\n", id)
            time.Sleep(10 * time.Millisecond)
        }
    }
}
```

**Альтернативы:**
```go
// errgroup — WaitGroup + первая ошибка + автоматическая отмена
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error {
    return worker(ctx)
})
if err := g.Wait(); err != nil {
    log.Fatal(err)
}

// semaphore — ограничение параллелизма
sem := semaphore.NewWeighted(10)
sem.Acquire(ctx, 1)
go func() {
    defer sem.Release(1)
    doWork()
}()
```

**Типичные ошибки.**
- `wg.Add(1)` внутри горутины — race condition, main может вызвать `Wait()` до `Add()`.
- Забыть `defer wg.Done()` — `Wait()` заблокируется навсегда.
- Не передавать `ctx` в горутину — нет способа остановить.
- Захват переменной цикла без копирования: `go func() { use(i) }()` — все горутины увидят последнее значение `i`.

**На интервью.**
- Объясни, почему `Add` должен быть до `go`.
- Упомяни `errgroup` как более удобную альтернативу для обработки ошибок.
- Follow-up: «Как ограничить число одновременных горутин?» — буферизированный канал или `semaphore.Weighted`.

---

### 2. Что такое канал и зачем нужна буферизация?

**Зачем спрашивают.** Каналы — основной механизм коммуникации в Go. Понимание буферизации критично для производительности и избежания deadlock.

**Короткий ответ.** Канал — типизированная очередь для передачи данных между горутинами. Небуферизированный канал синхронизирует отправителя и получателя. Буферизированный позволяет отправителю продолжить работу, пока буфер не заполнен.

**Детальный разбор.**

**Небуферизированный канал (`make(chan T)`):**
```
Sender                              Receiver
   │                                    │
   │──────── ch <- value ───────────────│ (блокируется)
   │                                    │
   │ (ждёт получателя)                  │ <-ch (блокируется)
   │                                    │
   │◄─────── handoff ──────────────────►│
   │                                    │
   │ (разблокирован)                    │ (разблокирован)
```

**Буферизированный канал (`make(chan T, n)`):**
```
Sender                 Buffer [cap=3]              Receiver
   │                   ┌───┬───┬───┐                  │
   │── ch <- 1 ───────►│ 1 │   │   │                  │
   │── ch <- 2 ───────►│ 1 │ 2 │   │                  │
   │── ch <- 3 ───────►│ 1 │ 2 │ 3 │                  │
   │── ch <- 4 ─────── │ (блокируется, буфер полон)   │
   │                   │ 1 │ 2 │ 3 │◄──────── <-ch ───│ → получил 1
   │ (разблокирован) ──│ 4 │ 2 │ 3 │                  │
```

**Семантика:**
| Операция | Unbuffered | Buffered |
|----------|------------|----------|
| `ch <- v` | Блокирует до `<-ch` | Блокирует если `len(ch) == cap(ch)` |
| `<-ch` | Блокирует до `ch <-` | Блокирует если `len(ch) == 0` |
| `close(ch)` | Разблокирует все `<-ch` | Разблокирует после опустошения буфера |

**Когда использовать буфер:**
- **Burst handling** — сглаживание пиков нагрузки
- **Decoupling** — отправитель не ждёт получателя
- **Bounded work queue** — ограничение backlog

**Пример.**
```go
func main() {
    // Unbuffered — синхронная передача
    sync := make(chan int)
    go func() {
        fmt.Println("Sender: sending")
        sync <- 42  // блокируется до получения
        fmt.Println("Sender: sent")
    }()
    time.Sleep(100 * time.Millisecond)
    fmt.Println("Receiver: receiving")
    v := <-sync
    fmt.Printf("Receiver: got %d\n", v)

    // Buffered — асинхронная передача
    async := make(chan int, 3)
    async <- 1  // не блокируется
    async <- 2  // не блокируется
    async <- 3  // не блокируется
    // async <- 4  // заблокировалось бы!

    fmt.Println(len(async), cap(async))  // 3, 3

    // Чтение
    for v := range async {
        fmt.Println(v)
        if len(async) == 0 {
            close(async)  // Ошибка! Нельзя закрывать в получателе
        }
    }
}
```

**Типичные ошибки.**
- Буфер = 1 для «быстроты» без понимания — может скрыть race condition.
- Огромный буфер «на всякий случай» — потребляет память, скрывает проблемы производительности.
- Закрывать канал в получателе — паника при следующей отправке.
- Отправлять в закрытый канал — panic.

**На интервью.**
- Объясни разницу unbuffered/buffered на примере producer-consumer.
- Упомяни, что unbuffered канал — точка синхронизации (happens-before).
- Follow-up: «Какой размер буфера выбрать?» — зависит от профиля нагрузки, часто 0 или 1 достаточно.

---

### 3. Как работает `select` и какие паттерны распространены?

**Зачем спрашивают.** `select` — главный инструмент мультиплексирования каналов. Без него невозможны таймауты, graceful shutdown, fan-in.

**Короткий ответ.** `select` выбирает готовую ветку случайно. Если готовых нет — блокируется (или выполняет `default`). Основные паттерны: таймаут, неблокирующая операция, fan-in, graceful shutdown.

**Детальный разбор.**

**Семантика select:**
1. Вычисляются все channel expressions
2. Если несколько готовы — выбирается случайно (fairness)
3. Если ни одна не готова:
   - Есть `default` → выполняется `default`
   - Нет `default` → блокируется до готовности

**Паттерн: Таймаут**
```go
select {
case result := <-work:
    return result, nil
case <-time.After(5 * time.Second):
    return nil, errors.New("timeout")
}
```

**Паттерн: Неблокирующая операция**
```go
select {
case ch <- value:
    // отправлено
default:
    // канал заполнен, делаем что-то другое
}

select {
case v := <-ch:
    // получено
default:
    // канал пуст
}
```

**Паттерн: Graceful shutdown**
```go
for {
    select {
    case job := <-jobs:
        process(job)
    case <-ctx.Done():
        // cleanup
        return ctx.Err()
    }
}
```

**Паттерн: Fan-in**
```go
func fanIn(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for {
                select {
                case v, ok := <-c:
                    if !ok {
                        return
                    }
                    select {
                    case out <- v:
                    case <-ctx.Done():
                        return
                    }
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
```

**Паттерн: Heartbeat**
```go
func worker(ctx context.Context) <-chan struct{} {
    heartbeat := make(chan struct{})

    go func() {
        defer close(heartbeat)
        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                select {
                case heartbeat <- struct{}{}:
                default: // не блокируемся если никто не слушает
                }
            case <-ctx.Done():
                return
            }
        }
    }()

    return heartbeat
}
```

**Пример.**
```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "from ch1"
    }()

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "from ch2"
    }()

    // Получаем из первого готового
    for i := 0; i < 2; i++ {
        select {
        case msg := <-ch1:
            fmt.Println(msg)
        case msg := <-ch2:
            fmt.Println(msg)
        case <-time.After(time.Second):
            fmt.Println("timeout")
        }
    }
}

// Output:
// from ch1
// from ch2
```

**Типичные ошибки.**
- `time.After` в цикле — утечка таймеров, используй `time.NewTimer` + `Reset`.
- Забыть `ctx.Done()` в `select` — горутина не остановится.
- Пустой `select{}` — блокируется навсегда (иногда полезно, но часто баг).
- Полагаться на порядок веток — выбор случайный.

**На интервью.**
- Покажи несколько паттернов: таймаут, shutdown, fan-in.
- Объясни, почему выбор случайный (предотвращает starvation).
- Follow-up: «Как избежать утечки таймеров в цикле?» — `time.NewTimer`, `timer.Reset()`, `timer.Stop()`.

---

### 4. Как правильно закрывать канал?

**Зачем спрашивают.** Неправильное закрытие — частый источник panic. Интервьюер проверяет понимание ownership и протокола завершения.

**Короткий ответ.** Закрывает только отправитель (владелец). Получатели проверяют закрытие через `v, ok := <-ch`. После закрытия чтение возвращает zero value, запись вызывает panic.

**Детальный разбор.**

**Правило: закрывает тот, кто отправляет.**
- Получатель не знает, будут ли ещё данные
- Отправитель знает, когда закончил
- Множественные отправители → нужна координация

**Поведение после close:**
| Операция | Результат |
|----------|-----------|
| `<-ch` | zero value, `ok=false` |
| `ch <- v` | panic |
| `close(ch)` | panic (double close) |

**Паттерн: Один producer**
```go
func producer() <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)  // producer закрывает
        for i := 0; i < 5; i++ {
            ch <- i
        }
    }()
    return ch
}

// Consumer использует range
for v := range producer() {
    fmt.Println(v)
}
```

**Паттерн: Множественные producers**
```go
func manyProducers(n int) <-chan int {
    ch := make(chan int)
    var wg sync.WaitGroup

    for i := 0; i < n; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            ch <- id
        }(i)
    }

    // Отдельная горутина закрывает после всех
    go func() {
        wg.Wait()
        close(ch)
    }()

    return ch
}
```

**Паттерн: Сигнальный канал (close as broadcast)**
```go
func main() {
    done := make(chan struct{})

    // Много получателей
    for i := 0; i < 5; i++ {
        go func(id int) {
            <-done  // блокируется до close
            fmt.Printf("Worker %d: received signal\n", id)
        }(i)
    }

    time.Sleep(100 * time.Millisecond)
    close(done)  // разблокирует всех получателей одновременно
    time.Sleep(100 * time.Millisecond)
}
```

**Пример.**
```go
func main() {
    ch := make(chan int, 3)

    // Producer
    go func() {
        defer close(ch)
        for i := 1; i <= 3; i++ {
            ch <- i
        }
    }()

    // Consumer с проверкой закрытия
    for {
        v, ok := <-ch
        if !ok {
            fmt.Println("Channel closed")
            break
        }
        fmt.Printf("Received: %d\n", v)
    }

    // Эквивалентно:
    // for v := range ch {
    //     fmt.Printf("Received: %d\n", v)
    // }
}

// Output:
// Received: 1
// Received: 2
// Received: 3
// Channel closed
```

**Типичные ошибки.**
- Закрывать канал в получателе — паника у отправителей.
- Double close — паника.
- Не закрывать канал — получатели блокируются навсегда (goroutine leak).
- Проверять `ok` без comma-ok syntax: `v := <-ch` вернёт zero value для закрытого канала.

**На интервью.**
- Сформулируй правило: «владелец (sender) закрывает».
- Покажи паттерн с WaitGroup для множественных producers.
- Follow-up: «Как узнать, закрыт ли канал без чтения?» — нельзя, только через `v, ok := <-ch` или `select` с `default`.

---

### 5. Как избежать утечек горутин?

**Зачем спрашивают.** Goroutine leak — частая проблема. Горутины накапливаются, потребляют память, и программа деградирует.

**Короткий ответ.** Всегда предусматривай путь выхода: `context.Context`, done channel, закрытие входного канала. Мониторь `runtime.NumGoroutine()` в продакшене.

**Детальный разбор.**

**Причины утечек:**
1. **Blocked channel operation** — никто не читает/пишет
2. **Missing cancellation** — нет ctx.Done()
3. **Abandoned goroutine** — создали и забыли
4. **Deadlock in select** — все ветки заблокированы

**Детектирование:**
```go
// В тестах
func TestNoLeak(t *testing.T) {
    before := runtime.NumGoroutine()
    doSomething()
    time.Sleep(100 * time.Millisecond) // дать время завершиться
    after := runtime.NumGoroutine()
    if after > before {
        t.Errorf("goroutine leak: before=%d, after=%d", before, after)
    }
}

// В продакшене
go func() {
    for {
        log.Printf("goroutines: %d", runtime.NumGoroutine())
        time.Sleep(time.Minute)
    }
}()
```

**Паттерн: Context cancellation**
```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case job, ok := <-jobs:
            if !ok {
                return nil  // канал закрыт
            }
            if err := job.Process(ctx); err != nil {
                return err
            }
        case <-ctx.Done():
            return ctx.Err()  // отмена
        }
    }
}
```

**Паттерн: Done channel**
```go
func producer(done <-chan struct{}) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; ; i++ {
            select {
            case out <- i:
            case <-done:
                return
            }
        }
    }()
    return out
}

// Использование
done := make(chan struct{})
ch := producer(done)
fmt.Println(<-ch)  // 0
fmt.Println(<-ch)  // 1
close(done)        // остановить producer
```

**Паттерн: Draining**
```go
// Если producer отправляет, но receiver ушёл — producer заблокируется
// Решение: drain канал

func drain[T any](ch <-chan T) {
    for range ch {
        // отбрасываем значения
    }
}

// Или неблокирующая отправка
select {
case ch <- value:
default:
    // receiver не готов, отбрасываем или логируем
}
```

**Пример.**
```go
// Утечка
func leakyWorker(jobs <-chan int) {
    go func() {
        for job := range jobs {  // блокируется если jobs не закрыт
            process(job)
        }
    }()
}

// Исправлено
func safeWorker(ctx context.Context, jobs <-chan int) {
    go func() {
        for {
            select {
            case job, ok := <-jobs:
                if !ok {
                    return
                }
                process(job)
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

**Типичные ошибки.**
- Создавать горутину без способа остановки.
- Игнорировать возвращаемые каналы — producer заблокируется.
- Не закрывать каналы — consumers заблокируются.
- Использовать глобальные channels без lifecycle management.

**На интервью.**
- Перечисли причины утечек и способы предотвращения.
- Покажи, как детектировать утечки в тестах.
- Follow-up: «Как найти, где именно утечка?» — `pprof.Lookup("goroutine")` покажет stack traces.

---

### 6. Какие коммуникационные паттерны нужно знать?

**Зачем спрашивают.** Каналы — примитив, паттерны — реальные решения. Интервьюер проверяет опыт применения конкурентности.

**Короткий ответ.** Pipeline, Fan-in/Fan-out, Worker Pool, Semaphore, Or-channel, Tee. Каждый решает конкретную задачу координации.

**Детальный разбор.**

**Pipeline:**
```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Stage 1 │────►│ Stage 2 │────►│ Stage 3 │
│ (gen)   │     │ (sq)    │     │ (print) │
└─────────┘     └─────────┘     └─────────┘
```

```go
func gen(ctx context.Context, nums ...int) <-chan int {
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

func sq(ctx context.Context, in <-chan int) <-chan int {
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

// Использование: sq(ctx, sq(ctx, gen(ctx, 1, 2, 3)))
```

**Fan-out / Fan-in:**
```
         ┌──────────┐
         │ Worker 1 │───┐
         └──────────┘   │
┌────┐   ┌──────────┐   │   ┌────────┐
│ In │──►│ Worker 2 │───┼──►│ Merge  │──► Out
└────┘   └──────────┘   │   └────────┘
         ┌──────────┐   │
         │ Worker 3 │───┘
         └──────────┘
```

```go
func fanOut(ctx context.Context, in <-chan int, n int) []<-chan int {
    outs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        outs[i] = worker(ctx, in)
    }
    return outs
}

func fanIn(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
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
```

**Or-channel (first to close):**
```go
func or(ctx context.Context, channels ...<-chan struct{}) <-chan struct{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan struct{})
    go func() {
        defer close(orDone)

        // Создаём select динамически через reflect или рекурсию
        // Упрощённая версия:
        select {
        case <-channels[0]:
        case <-channels[1]:
        case <-or(ctx, append(channels[2:], orDone)...):
        case <-ctx.Done():
        }
    }()

    return orDone
}

// Использование: сигнал, когда любой из каналов закрыт
```

**Semaphore (ограничение параллелизма):**
```go
func semaphore(ctx context.Context, maxConcurrent int, tasks []func()) error {
    sem := make(chan struct{}, maxConcurrent)
    var wg sync.WaitGroup
    errCh := make(chan error, len(tasks))

    for _, task := range tasks {
        wg.Add(1)
        go func(t func()) {
            defer wg.Done()

            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
                t()
            case <-ctx.Done():
                errCh <- ctx.Err()
            }
        }(task)
    }

    wg.Wait()
    close(errCh)

    for err := range errCh {
        if err != nil {
            return err
        }
    }
    return nil
}
```

**Пример.**
```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Pipeline: генерация → квадрат → фильтр
    nums := gen(ctx, 1, 2, 3, 4, 5)
    squared := sq(ctx, nums)

    for n := range squared {
        fmt.Println(n)
    }
}

// Output:
// 1
// 4
// 9
// 16
// 25
```

**Типичные ошибки.**
- Не закрывать каналы между стадиями pipeline.
- Fan-out без ограничения — взрыв горутин.
- Or-channel без context — невозможно отменить.
- Забыть WaitGroup в fan-in — преждевременное закрытие output.

**На интервью.**
- Нарисуй схему pipeline или fan-out/fan-in.
- Объясни, когда какой паттерн применим.
- Follow-up: «Как обрабатывать ошибки в pipeline?» — errgroup, отдельный error channel, или возврат `struct{value, err}`.

---

### 7. Как работают примитивы синхронизации в связке с каналами?

**Зачем спрашивают.** Каналы и sync-примитивы дополняют друг друга. Интервьюер проверяет, знаешь ли ты когда что использовать.

**Короткий ответ.** `sync.Mutex` для защиты shared state, `sync.WaitGroup` для ожидания, `sync.Once` для однократной инициализации, `sync.Cond` для условных переменных. Каналы — для передачи ownership.

**Детальный разбор.**

**Выбор между каналами и mutex:**
| Сценарий | Инструмент |
|----------|------------|
| Передача данных между горутинами | channel |
| Защита shared mutable state | mutex |
| Координация нескольких горутин | WaitGroup, channel |
| Broadcast сигнал | close(channel), Cond |
| Однократная инициализация | sync.Once |

**sync.Mutex + channel:**
```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
    done  chan struct{}
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func (c *SafeCounter) Wait() {
    <-c.done
}
```

**sync.Once для lazy init:**
```go
type Connection struct {
    once sync.Once
    conn net.Conn
    err  error
}

func (c *Connection) Get() (net.Conn, error) {
    c.once.Do(func() {
        c.conn, c.err = net.Dial("tcp", "localhost:8080")
    })
    return c.conn, c.err
}
```

**sync.Cond (редко нужен, но полезно знать):**
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
    q.items = append(q.items, item)
    q.cond.Signal()  // разбудить одного ожидающего
    q.mu.Unlock()
}

func (q *Queue) Pop() int {
    q.mu.Lock()
    for len(q.items) == 0 {
        q.cond.Wait()  // освобождает lock и ждёт Signal
    }
    item := q.items[0]
    q.items = q.items[1:]
    q.mu.Unlock()
    return item
}
```

**Пример.**
```go
func main() {
    var (
        wg      sync.WaitGroup
        mu      sync.Mutex
        results []int
    )

    ch := make(chan int, 10)

    // Producers
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 5; j++ {
                ch <- id*10 + j
            }
        }(i)
    }

    // Consumer with mutex-protected aggregation
    go func() {
        for v := range ch {
            mu.Lock()
            results = append(results, v)
            mu.Unlock()
        }
    }()

    wg.Wait()
    close(ch)

    time.Sleep(100 * time.Millisecond)
    fmt.Printf("Results: %v\n", results)
}
```

**Типичные ошибки.**
- Использовать mutex для передачи данных — это задача каналов.
- Использовать каналы для защиты state — mutex проще и быстрее.
- Забывать Unlock — deadlock.
- Lock в defer при длительных операциях — держишь lock дольше нужного.

**На интервью.**
- Сформулируй правило: «Share memory by communicating, don't communicate by sharing memory» — но знай исключения.
- Покажи sync.Once для singleton/connection pool.
- Follow-up: «Когда sync.RWMutex лучше Mutex?» — когда читателей много больше писателей.

---

### 8. Что такое singleflight и когда его использовать?

**Зачем спрашивают.** Singleflight — важный паттерн для дедупликации запросов. Интервьюер проверяет знание продвинутых инструментов конкурентности из `golang.org/x/sync`.

**Короткий ответ.** `singleflight.Group` гарантирует, что для одного ключа в момент времени выполняется только один вызов функции. Остальные вызовы с тем же ключом ждут результата первого. Используется для защиты от thundering herd при cache miss.

**Детальный разбор.**

**Проблема thundering herd:**
```
Cache miss → 1000 одновременных запросов к DB
            → DB перегружена
            → cascade failure
```

**Решение с singleflight:**
```
Cache miss → 1 запрос к DB, 999 ждут результат
            → DB не перегружена
            → все получают один результат
```

**Как работает:**
```
┌─────────────────────────────────────────────────────────────┐
│                    singleflight.Group                        │
├─────────────────────────────────────────────────────────────┤
│  key: "user:123"                                            │
│  ┌─────────────┐                                            │
│  │ In-flight   │ ◄── Goroutine 1 (owner)                    │
│  │   call      │ ◄── Goroutine 2 (waiter)                   │
│  │             │ ◄── Goroutine 3 (waiter)                   │
│  └─────────────┘                                            │
│       │                                                      │
│       ▼                                                      │
│  DB query: SELECT * FROM users WHERE id = 123               │
│       │                                                      │
│       ▼                                                      │
│  Result shared with all goroutines                          │
└─────────────────────────────────────────────────────────────┘
```

**API:**
```go
type Group struct { ... }

// Do выполняет fn только один раз для ключа key
// shared = true если результат получен от другого вызова
func (g *Group) Do(key string, fn func() (any, error)) (v any, err error, shared bool)

// DoChan возвращает канал с результатом (async версия)
func (g *Group) DoChan(key string, fn func() (any, error)) <-chan Result

// Forget удаляет ключ, следующий Do создаст новый вызов
func (g *Group) Forget(key string)
```

**Пример.**
```go
import "golang.org/x/sync/singleflight"

type UserCache struct {
    cache map[int64]*User
    mu    sync.RWMutex
    db    *sql.DB
    sf    singleflight.Group
}

func (c *UserCache) Get(ctx context.Context, userID int64) (*User, error) {
    // Проверяем кэш
    c.mu.RLock()
    if user, ok := c.cache[userID]; ok {
        c.mu.RUnlock()
        return user, nil
    }
    c.mu.RUnlock()

    // Singleflight: дедупликация запросов к DB
    key := fmt.Sprintf("user:%d", userID)
    result, err, shared := c.sf.Do(key, func() (any, error) {
        // Этот код выполнится только один раз для всех concurrent запросов
        user, err := c.loadFromDB(ctx, userID)
        if err != nil {
            return nil, err
        }

        // Сохраняем в кэш
        c.mu.Lock()
        c.cache[userID] = user
        c.mu.Unlock()

        return user, nil
    })

    if shared {
        log.Printf("Request for user %d was deduplicated", userID)
    }

    if err != nil {
        return nil, err
    }
    return result.(*User), nil
}

func (c *UserCache) loadFromDB(ctx context.Context, userID int64) (*User, error) {
    row := c.db.QueryRowContext(ctx, "SELECT id, name FROM users WHERE id = $1", userID)
    var user User
    if err := row.Scan(&user.ID, &user.Name); err != nil {
        return nil, err
    }
    return &user, nil
}
```

**Когда использовать:**
1. **Cache miss protection** — при промахе кэша не штурмовать DB
2. **External API calls** — дедупликация одинаковых запросов
3. **Expensive computations** — не пересчитывать одно и то же
4. **Resource loading** — загрузка конфигов, шаблонов

**Когда НЕ использовать:**
- Результат зависит от контекста вызывающего
- Нужна изоляция между вызовами
- Функция имеет side effects, которые должны повторяться

**Типичные ошибки.**
- Использовать для функций с side effects — только первый вызов выполнится.
- Забыть про context cancellation — singleflight не передаёт ctx в fn автоматически.
- Не обрабатывать ошибки — ошибка тоже "расшаривается" между ожидающими.
- Огромные ключи — создают memory overhead.

**На интервью.**
- Объясни проблему thundering herd и как singleflight её решает.
- Упомяни, что `shared` флаг полезен для метрик/логирования.
- Follow-up: «Как добавить TTL к singleflight?» — комбинировать с `time.AfterFunc` и `Forget()`.

---

### 9. Как использовать errgroup.SetLimit?

**Зачем спрашивают.** `errgroup` — стандартный инструмент для параллельных задач с обработкой ошибок. `SetLimit` добавляет bounded concurrency. Интервьюер проверяет знание современных практик.

**Короткий ответ.** `errgroup.Group.SetLimit(n)` ограничивает количество одновременно выполняемых горутин до `n`. Вызов `Go()` блокируется, если лимит достигнут. Это встроенный semaphore для errgroup.

**Детальный разбор.**

**Проблема без лимита:**
```go
g, ctx := errgroup.WithContext(ctx)
for _, url := range urls {  // 10000 URLs
    url := url
    g.Go(func() error {
        return fetch(ctx, url)  // 10000 горутин одновременно!
    })
}
```

**Решение с SetLimit:**
```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10)  // максимум 10 горутин

for _, url := range urls {
    url := url
    g.Go(func() error {  // блокируется если уже 10 горутин
        return fetch(ctx, url)
    })
}
```

**Как работает:**
```
SetLimit(3)

g.Go(task1) → [slot 1: task1] ───────►
g.Go(task2) → [slot 2: task2] ───────►
g.Go(task3) → [slot 3: task3] ───────►
g.Go(task4) → (blocked, waiting...)

task1 completes → [slot 1: free]
g.Go(task4) → [slot 1: task4] ───────►
```

**API (Go 1.20+):**
```go
type Group struct { ... }

// SetLimit устанавливает лимит горутин
// n < 0 — без лимита (default)
// n == 0 — panic
func (g *Group) SetLimit(n int)

// TryGo пытается запустить f без блокировки
// Возвращает false если лимит достигнут
func (g *Group) TryGo(f func() error) bool
```

**Пример.**
```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10)  // максимум 10 concurrent requests

    responses := make([]Response, len(urls))

    for i, url := range urls {
        i, url := i, url  // capture
        g.Go(func() error {
            resp, err := fetchWithRetry(ctx, url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            responses[i] = resp
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return responses, nil
}

// Использование TryGo для graceful degradation
func processWithBackpressure(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(5)

    var skipped int64
    for _, item := range items {
        item := item
        // TryGo не блокирует, возвращает false если лимит достигнут
        if !g.TryGo(func() error {
            return process(ctx, item)
        }) {
            atomic.AddInt64(&skipped, 1)
            // Элемент пропущен из-за backpressure
        }
    }

    err := g.Wait()
    log.Printf("Skipped %d items due to backpressure", skipped)
    return err
}
```

**Сравнение с альтернативами:**

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| `errgroup.SetLimit` | Встроено, просто | Только в Go 1.20+ |
| `semaphore.Weighted` | Гибкость, взвешенные операции | Больше кода |
| Buffered channel | Простота | Нет интеграции с errgroup |
| Worker pool | Контроль над воркерами | Сложнее реализовать |

**Паттерн: комбинация с context timeout:**
```go
func processWithTimeout(items []Item) error {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(20)

    for _, item := range items {
        item := item
        g.Go(func() error {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
                return processItem(ctx, item)
            }
        })
    }

    return g.Wait()
}
```

**Типичные ошибки.**
- Вызывать `SetLimit` после `Go` — undefined behavior.
- `SetLimit(0)` — panic.
- Забыть capture переменную цикла — классическая ошибка с горутинами.
- Не проверять результат `TryGo` — элементы молча теряются.

**На интервью.**
- Покажи разницу между `Go` (блокирующий) и `TryGo` (неблокирующий).
- Объясни, когда использовать каждый: `Go` для обязательных задач, `TryGo` для best-effort.
- Follow-up: «Как реализовать SetLimit до Go 1.20?» — `semaphore.Weighted` + errgroup вручную.

---

## См. также

- [Concurrency Patterns](./05-concurrency-patterns.md) — паттерны построения конкурентных систем
- [Errors & Context](./03-errors-context.md) — отмена и таймауты через context
- [Java Concurrency Basics](../01-java/04-concurrency-basics.md) — сравнение подходов к многопоточности
- [Python Asyncio](../02-python/04-asyncio-concurrency.md) — асинхронность в Python
- [Rate Limiter](../03-system-design/02-rate-limiter.md) — применение конкурентности в System Design

---

## Практика

1. **Lifecycle management** — реализуй worker pool с graceful shutdown через context. Добавь таймаут на завершение.

2. **Pipeline с отменой** — построй pipeline из 3 стадий с контекстом. Убедись, что все горутины завершаются при отмене.

3. **Fan-in с таймаутом** — объедини результаты от нескольких HTTP-запросов. Первый таймаут — для отдельного запроса, второй — для всей операции.

4. **Rate limiter** — реализуй token bucket через канал и ticker. Добавь burst capability.

5. **Or-channel** — реализуй и протестируй функцию, возвращающую канал, закрывающийся при закрытии любого входного.

6. **Goroutine leak detector** — напиши тест, проверяющий отсутствие утечек в твоём concurrent коде.

---

## Дополнительные материалы

- [Go blog: Go Concurrency Patterns](https://go.dev/blog/pipelines) — официальный гайд по pipeline
- [Go blog: Advanced Concurrency Patterns](https://go.dev/blog/io2013-talk-concurrency) — продвинутые паттерны
- [Rob Pike: Concurrency Is Not Parallelism](https://www.youtube.com/watch?v=oV9rvDllKEg) — классический доклад
- [golang.org/x/sync](https://pkg.go.dev/golang.org/x/sync) — errgroup, semaphore, singleflight
- [context package](https://pkg.go.dev/context) — документация context
