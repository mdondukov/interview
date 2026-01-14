# 01 — Go Runtime & Scheduler

Конспект по внутреннему устройству Go runtime: горутины, модель G/M/P, планировщик, вытеснение, взаимодействие с GC. Материал построен в формате вопрос-ответ для подготовки к техническим интервью.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [00-language-overview](./00-language-overview.md) · Следующая тема: [02-goroutines-channels](./02-goroutines-channels.md)

---

## Архитектура планировщика (G/M/P)

```
┌─────────────────────────────────────────────────────────────┐
│                        Go Runtime                           │
├─────────────────────────────────────────────────────────────┤
│  Global Run Queue: [G] [G] [G] ...                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                 │
│  │   P0    │    │   P1    │    │   P2    │   (GOMAXPROCS)  │
│  │ Local Q │    │ Local Q │    │ Local Q │                 │
│  │ [G][G]  │    │ [G]     │    │ [G][G]  │                 │
│  └────┬────┘    └────┬────┘    └────┬────┘                 │
│       │              │              │                       │
│  ┌────▼────┐    ┌────▼────┐    ┌────▼────┐                 │
│  │   M0    │    │   M1    │    │   M2    │   (OS Threads)  │
│  │ (g0)    │    │ (g0)    │    │ (g0)    │                 │
│  └─────────┘    └─────────┘    └─────────┘                 │
│                                                             │
│  Network Poller: [blocked G] [blocked G] ...               │
│  Idle M list: [M] [M] ...                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Вопросы и разборы

### 1. Что такое горутина и как она устроена внутри?

**Зачем спрашивают.** Горутины — фундамент конкурентности в Go. Интервьюер проверяет, понимаешь ли ты разницу между горутиной и потоком ОС, и знаешь ли внутреннее устройство.

**Короткий ответ.** Горутина — это легковесная единица выполнения, управляемая Go runtime. Она представлена структурой `runtime.g` с собственным стеком (~2KB начально), указателем инструкции, состоянием и метаданными.

**Детальный разбор.**

Каждая горутина — это структура `g` в runtime:
```go
type g struct {
    stack       stack   // стек горутины [lo, hi)
    stackguard0 uintptr // для проверки переполнения стека
    m           *m      // текущий M, выполняющий эту G
    sched       gobuf   // контекст: sp, pc, g, bp
    atomicstatus uint32 // состояние: _Gidle, _Grunnable, _Grunning, _Gwaiting...
    goid        int64   // уникальный ID горутины
    waitsince   int64   // время начала ожидания
    // ... другие поля
}
```

**Состояния горутины:**
- `_Gidle` — только создана, не использовалась
- `_Grunnable` — готова к выполнению, в очереди
- `_Grunning` — выполняется на M
- `_Gwaiting` — заблокирована (канал, mutex, syscall)
- `_Gsyscall` — в системном вызове
- `_Gdead` — завершена, может быть переиспользована

**Стек горутины:**
- Начальный размер: ~2KB (было 8KB до Go 1.4)
- Растёт динамически при необходимости
- Максимум: 1GB (настраивается через `runtime/debug.SetMaxStack`)
- Механизм роста: копирование стека (stack copying, не segmented stacks)

**Пример.**
```go
func main() {
    // Выводим информацию о текущей горутине
    var buf [64]byte
    n := runtime.Stack(buf[:], false)
    fmt.Printf("Goroutine stack:\n%s\n", buf[:n])

    // Смотрим количество горутин
    fmt.Printf("Number of goroutines: %d\n", runtime.NumGoroutine())

    // Запускаем горутину
    go func() {
        fmt.Println("Hello from goroutine")
    }()

    time.Sleep(time.Millisecond)
    fmt.Printf("Number of goroutines: %d\n", runtime.NumGoroutine())
}

// Output:
// Goroutine stack:
// goroutine 1 [running]:
// main.main()
//     /tmp/main.go:10 +0x...
// Number of goroutines: 1
// Hello from goroutine
// Number of goroutines: 1
```

**Типичные ошибки.**
- Думать, что горутина = поток ОС. Горутины мультиплексируются на потоки.
- Не понимать, что стек копируется при росте — указатели на локальные переменные остаются валидными (компилятор знает об этом).
- Создавать миллионы горутин с большим стеком — каждая занимает память.

**На интервью.**
- Упомяни, что горутина легче потока: переключение ~десятки наносекунд vs микросекунды для потоков ОС.
- Объясни stack growth: компилятор вставляет проверку `stackguard` в пролог функций.
- Follow-up: «Как увидеть все горутины?» — `runtime.Stack(buf, true)` или `pprof.Lookup("goroutine")`.

---

### 2. Что такое модель G/M/P и зачем она нужна?

**Зачем спрашивают.** G/M/P — сердце планировщика Go. Вопрос проверяет понимание архитектуры runtime.

**Короткий ответ.** G (goroutine), M (machine/OS thread), P (processor) — три сущности планировщика. P определяет параллелизм и содержит ресурсы для выполнения G на M.

**Детальный разбор.**

**G (Goroutine):**
- Представляет единицу работы
- Содержит стек, состояние, контекст выполнения
- Миллионы G — норма

**M (Machine):**
- Обёртка над потоком ОС
- Содержит служебную горутину `g0` для работы runtime
- Содержит `gsignal` для обработки сигналов
- Может быть больше, чем GOMAXPROCS (если G блокируются в syscall)

**P (Processor):**
- Логический процессор, контекст выполнения
- Количество = GOMAXPROCS (по умолчанию = число CPU)
- Содержит:
  - Локальную очередь runnable G (до 256)
  - mcache (per-P кеш аллокатора)
  - Таймеры (per-P heap таймеров)
  - Рандомизатор для work stealing

**Зачем P нужен:**
1. Ограничивает параллелизм — не больше GOMAXPROCS горутин выполняются одновременно
2. Локальность данных — кеши, очереди привязаны к P, не к M
3. Позволяет M отдать P при блокировке и продолжить работу на другом M

**Пример.**
```go
func main() {
    // Устанавливаем число P
    runtime.GOMAXPROCS(4)

    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0)) // 4
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())          // физические ядра

    // Запускаем CPU-bound работу
    var wg sync.WaitGroup
    for i := 0; i < 8; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // CPU-bound работа
            sum := 0
            for j := 0; j < 1e8; j++ {
                sum += j
            }
            fmt.Printf("Goroutine %d done\n", id)
        }(i)
    }
    wg.Wait()
}
```

**Визуализация работы:**
```
Время →
P0: [G1]────────[G5]────────
P1: [G2]────────[G6]────────
P2: [G3]────────[G7]────────
P3: [G4]────────[G8]────────

8 горутин выполняются по 4 одновременно (GOMAXPROCS=4)
```

**Типичные ошибки.**
- Ставить GOMAXPROCS > числа CPU для CPU-bound задач — увеличивает contention.
- Путать P и CPU — P логический, их число можно менять в runtime.
- Думать, что GOMAXPROCS=1 запрещает параллелизм syscalls — нет, M может быть больше.

**На интервью.**
- Нарисуй диаграмму G/M/P на доске.
- Объясни, почему P введён в Go 1.1 (до этого был G/M и глобальный lock).
- Follow-up: «Что происходит, когда G блокируется в syscall?» — переход к вопросу 5.

---

### 3. Как работает планировщик и что такое work stealing?

**Зачем спрашивают.** Понимание планировщика критично для написания эффективного конкурентного кода. Work stealing — ключевой механизм балансировки нагрузки.

**Короткий ответ.** Планировщик распределяет G по P. Каждый P имеет локальную очередь. Когда очередь пуста, P «ворует» работу у других P или берёт из глобальной очереди.

**Детальный разбор.**

**Очереди горутин:**
1. **Локальная очередь P** — до 256 G, lock-free (LIFO push, FIFO steal)
2. **Глобальная очередь** — неограниченная, защищена mutex
3. **runnext слот** — одна G с приоритетом (для producer-consumer локальности)

**Алгоритм планирования (упрощённо):**
```
func schedule() {
    // 1. Проверить runnext
    if g := p.runnext; g != nil {
        return g
    }

    // 2. Взять из локальной очереди
    if g := p.runq.get(); g != nil {
        return g
    }

    // 3. Каждый 61-й раз проверить глобальную очередь
    if schedtick%61 == 0 {
        if g := globrunq.get(); g != nil {
            return g
        }
    }

    // 4. Проверить netpoll (готовые I/O горутины)
    if g := netpoll(); g != nil {
        return g
    }

    // 5. Work stealing: украсть половину очереди случайного P
    for i := 0; i < 4; i++ {
        victim := randomP()
        if g := victim.runq.steal(); g != nil {
            return g
        }
    }

    // 6. Нет работы — припарковать M
    stopm()
}
```

**Work stealing детали:**
- Вор выбирает жертву случайно (снижает contention)
- Ворует половину очереди с хвоста (FIFO для вора)
- Хвост содержит «старые» задачи — лучше для кеша

**runnext оптимизация:**
Когда G разблокирует другую G (например, send в канал → recv), разблокированная попадает в `runnext`. Это даёт:
- Локальность данных (данные в кеше)
- Быстрое переключение (producer-consumer паттерн)

**Пример.**
```go
func main() {
    // Включаем трассировку планировщика
    // Запуск: GODEBUG=schedtrace=1000 go run main.go

    for i := 0; i < 10; i++ {
        go func(id int) {
            time.Sleep(100 * time.Millisecond)
            fmt.Printf("G%d done\n", id)
        }(i)
    }

    time.Sleep(time.Second)
}

// Output (schedtrace):
// SCHED 0ms: gomaxprocs=8 idleprocs=7 threads=2 spinningthreads=0
//            idlethreads=0 runqueue=0 [0 0 0 0 0 0 0 0]
// SCHED 1000ms: gomaxprocs=8 idleprocs=8 threads=3 spinningthreads=0
//               idlethreads=1 runqueue=0 [0 0 0 0 0 0 0 0]
```

**Интерпретация schedtrace:**
- `gomaxprocs` — число P
- `idleprocs` — свободные P
- `threads` — общее число M
- `runqueue` — размер глобальной очереди
- `[0 0 0 ...]` — размеры локальных очередей каждого P

**Типичные ошибки.**
- Думать, что горутины выполняются в порядке создания — нет гарантий.
- Полагаться на `runtime.Gosched()` для синхронизации — это hint, не гарантия.
- Создавать тысячи горутин, которые сразу блокируются — забивают очереди.

**На интервью.**
- Объясни, почему work stealing лучше центрального планировщика: меньше contention, лучше cache locality.
- Упомяни `schedtrace` для диагностики.
- Follow-up: «Что будет, если все P заняты, а новые G создаются?» — они идут в локальную очередь, при переполнении — в глобальную.

---

### 4. Как работает вытеснение (preemption)?

**Зачем спрашивают.** Без вытеснения одна горутина могла бы заблокировать весь процессор. Понимание preemption важно для отладки «зависших» программ.

**Короткий ответ.** Go использует кооперативное вытеснение (на function calls) + асинхронное (Go 1.14+, через сигналы). Горутина может быть прервана на safe points или асинхронно.

**Детальный разбор.**

**Кооперативное вытеснение (до Go 1.14):**
- Происходит только в safe points: вызовы функций, операции с каналами, syscalls
- Компилятор вставляет проверку `stackguard` в пролог функций
- Если `stackguard0 == stackPreempt`, горутина вызывает `runtime.morestack` → планировщик

**Проблема:** бесконечный цикл без вызовов функций блокирует P:
```go
for {
    i++  // никогда не отдаст управление (до Go 1.14)
}
```

**Асинхронное вытеснение (Go 1.14+):**
- Sysmon горутина периодически проверяет, работает ли G слишком долго (>10ms)
- Если да, отправляет сигнал SIGURG потоку
- Обработчик сигнала устанавливает флаг preempt
- При следующей проверке стека G переключается

**Safe points для GC и preemption:**
- Вызовы функций
- Возврат из функций
- Операции с каналами/select
- Вход в syscall
- Асинхронно через сигналы (Go 1.14+)

**Пример.**
```go
func main() {
    runtime.GOMAXPROCS(1)

    // До Go 1.14 эта горутина заблокировала бы единственный P
    go func() {
        for {
            // tight loop без вызовов функций
        }
    }()

    // Эта горутина может не получить управление (до Go 1.14)
    go func() {
        fmt.Println("This may never print")
    }()

    time.Sleep(time.Second)
}

// Go 1.14+: обе горутины получат CPU time благодаря async preemption
```

**Управление preemption:**
```go
// Добровольная отдача процессора
runtime.Gosched()

// Закрепить горутину на OS thread (отключает миграцию между M)
runtime.LockOSThread()
defer runtime.UnlockOSThread()

// Отключить async preemption (для отладки)
// GODEBUG=asyncpreemptoff=1 go run main.go
```

**Типичные ошибки.**
- Писать tight loops и ожидать preemption в старых версиях Go.
- Использовать `runtime.Gosched()` вместо правильной синхронизации.
- Забывать `runtime.UnlockOSThread()` — поток остаётся заблокированным.

**На интервью.**
- Объясни разницу между кооперативным и preemptive scheduling.
- Упомяни, что async preemption добавило сложности (сигналы, safe points), но решило реальную проблему.
- Follow-up: «Как async preemption влияет на performance?» — минимально, сигналы редки.

---

### 5. Что происходит при системных вызовах?

**Зачем спрашивают.** Syscalls — точка взаимодействия с ОС. Понимание того, как Go обрабатывает блокирующие вызовы, критично для I/O-bound приложений.

**Короткий ответ.** При blocking syscall M отдаёт P другому M и блокируется. При возврате M ищет свободный P или становится idle. Non-blocking I/O использует netpoller.

**Детальный разбор.**

**Blocking syscall (файловый I/O, CGO):**
```
1. G вызывает syscall
2. M вызывает entersyscall():
   - P открепляется от M
   - P ищет другой M (или создаёт новый)
   - G переходит в состояние _Gsyscall
3. M блокируется в syscall ОС
4. После возврата M вызывает exitsyscall():
   - Пытается получить свой P обратно
   - Если P занят — ищет свободный P
   - Если нет свободных P — G идёт в глобальную очередь, M паркуется
```

**Non-blocking I/O (сеть):**
```
1. G вызывает сетевую операцию (Read, Write, Accept)
2. Если операция блокирующая:
   - G регистрируется в netpoller (epoll/kqueue/IOCP)
   - G переходит в _Gwaiting
   - P продолжает выполнять другие G
3. Когда дескриптор готов:
   - Netpoller будит G
   - G попадает в очередь P
```

**Netpoller:**
- Работает в отдельном потоке без P
- Использует epoll (Linux), kqueue (macOS/BSD), IOCP (Windows)
- Горутины не занимают M при ожидании I/O

**Пример.**
```go
func main() {
    // Файловый I/O — блокирует M
    go func() {
        data, _ := os.ReadFile("/dev/urandom") // blocking syscall
        _ = data
    }()

    // Сетевой I/O — использует netpoller
    go func() {
        conn, _ := net.Dial("tcp", "example.com:80")
        conn.Read(make([]byte, 1024)) // non-blocking через netpoller
    }()

    // Смотрим число потоков
    time.Sleep(100 * time.Millisecond)

    // GOMAXPROCS=4, но threads может быть больше из-за syscalls
    // Проверить: ps -T -p <pid> или runtime метрики
}
```

**Sysmon горутина:**
- Работает в отдельном потоке без P
- Проверяет:
  - Горутины в syscall > 10ms — отбирает P
  - Горутины без preemption > 10ms — посылает сигнал
  - Сеть — опрашивает netpoller
  - GC — запускает если нужно

**Типичные ошибки.**
- Думать, что файловый I/O не блокирует — блокирует M.
- Открывать тысячи файлов параллельно — создаёт тысячи M.
- Не понимать разницу между сетевым и файловым I/O в Go.

**На интервью.**
- Объясни, почему сетевой I/O эффективен (netpoller), а файловый — нет.
- Упомяни io_uring (Linux 5.1+) — Go экспериментирует с ним для файлового I/O.
- Follow-up: «Как CGO влияет на планировщик?» — CGO вызовы = blocking syscalls, M блокируется.

---

### 6. Как GC взаимодействует с планировщиком?

**Зачем спрашивают.** GC паузы влияют на latency. Понимание взаимодействия GC и планировщика помогает оптимизировать приложения.

**Короткий ответ.** Go использует concurrent mark-and-sweep GC. Краткие STW паузы (~1ms) для старта/финиша цикла, основная работа concurrent. Горутины помогают GC при аллокациях (mutator assist).

**Детальный разбор.**

**Фазы GC:**
```
1. Sweep Termination (STW ~100μs)
   - Завершить предыдущий sweep
   - Включить write barrier

2. Mark Phase (Concurrent)
   - Сканировать стеки, глобальные переменные
   - Трассировать достижимые объекты
   - Write barrier отслеживает изменения

3. Mark Termination (STW ~100μs)
   - Пересканировать изменённые объекты
   - Отключить write barrier

4. Sweep Phase (Concurrent)
   - Освобождать память по мере аллокаций
```

**Mutator assist:**
Если приложение аллоцирует быстрее, чем GC успевает mark:
```go
func mallocgc(size uintptr, ...) {
    // Проверить, нужна ли помощь GC
    if assistG.gcAssistBytes < 0 {
        // Выполнить часть mark работы
        gcAssistAlloc(assistG)
    }
    // ... аллокация
}
```

Это добавляет latency аллокациям в «горячие» моменты.

**Триггер GC:**
- По умолчанию: heap вырос на 100% с последнего GC (GOGC=100)
- `GOGC=50` — GC чаще, меньше памяти
- `GOGC=200` — GC реже, больше памяти
- `debug.SetGCPercent(-1)` — отключить автоматический GC

**Пример.**
```go
func main() {
    // Включить трассировку GC
    // GODEBUG=gctrace=1 go run main.go

    var stats runtime.MemStats

    // Много аллокаций
    for i := 0; i < 1000; i++ {
        _ = make([]byte, 1<<20) // 1MB
    }

    runtime.ReadMemStats(&stats)
    fmt.Printf("NumGC: %d\n", stats.NumGC)
    fmt.Printf("PauseTotalNs: %d ms\n", stats.PauseTotalNs/1e6)
    fmt.Printf("HeapAlloc: %d MB\n", stats.HeapAlloc/1e6)
}

// gctrace output:
// gc 1 @0.001s 2%: 0.010+0.20+0.010 ms clock, 0.080+0/0.15/0.20+0.080 ms cpu, 4->4->0 MB, 5 MB goal, 8 P
//
// Интерпретация:
// gc 1           — номер GC цикла
// @0.001s        — время с начала программы
// 2%             — % CPU на GC
// 0.010+0.20+0.010 ms clock — STW1 + concurrent + STW2
// 4->4->0 MB     — heap перед mark → после mark → живых объектов
// 5 MB goal      — цель heap size
// 8 P            — число P
```

**Оптимизация GC:**
```go
// Уменьшить частоту GC (больше памяти, меньше CPU)
debug.SetGCPercent(200)

// Установить soft memory limit (Go 1.19+)
debug.SetMemoryLimit(1 << 30) // 1GB

// Принудительный GC
runtime.GC()

// Освободить память ОС
debug.FreeOSMemory()
```

**Типичные ошибки.**
- Игнорировать mutator assist при профилировании — latency может быть в allocations, не в GC pause.
- Ставить GOGC=-1 и забыть вызывать runtime.GC() — OOM.
- Не понимать, что «живые» объекты определяют работу GC, не аллокации.

**На интервью.**
- Объясни три фазы: STW → concurrent mark → STW → concurrent sweep.
- Упомяни write barrier и зачем он нужен (отслеживать изменения указателей).
- Follow-up: «Как уменьшить GC паузы?» — меньше аллокаций, sync.Pool, преаллокация, увеличить GOGC.

---

### 7. Чем управляет GOMAXPROCS и когда его менять?

**Зачем спрашивают.** GOMAXPROCS — единственная «ручка» для настройки параллелизма. Интервьюер проверяет, понимаешь ли ты его влияние.

**Короткий ответ.** GOMAXPROCS задаёт число P — максимум одновременно выполняющихся горутин. По умолчанию равен числу CPU. Менять стоит редко.

**Детальный разбор.**

**Что определяет GOMAXPROCS:**
- Число P (логических процессоров)
- Максимум горутин в состоянии _Grunning
- НЕ ограничивает число M (потоков ОС)
- НЕ ограничивает число G (горутин)

**Когда менять:**
| Сценарий | GOMAXPROCS | Почему |
|----------|------------|--------|
| CPU-bound, bare metal | = NumCPU (default) | Оптимально |
| CPU-bound, контейнер | = CPU limit | Избежать throttling |
| I/O-bound | = NumCPU или меньше | I/O не требует много P |
| Тесты с -race | = 1-4 | Детерминизм, меньше races |
| Debug/profile | = 1 | Упрощает анализ |

**Контейнеры и GOMAXPROCS:**
```go
// Проблема: Go видит все CPU хоста, не лимит контейнера
// Решение: использовать uber-go/automaxprocs

import _ "go.uber.org/automaxprocs"

func main() {
    // automaxprocs автоматически читает cgroup limits
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
}

// Или вручную:
// docker run --cpus=2 myapp
// В контейнере: runtime.GOMAXPROCS(2)
```

**Пример.**
```go
func main() {
    // Получить текущее значение (не меняя)
    current := runtime.GOMAXPROCS(0)
    fmt.Printf("Current GOMAXPROCS: %d\n", current)

    // Установить новое значение
    old := runtime.GOMAXPROCS(2)
    fmt.Printf("Old: %d, New: %d\n", old, runtime.GOMAXPROCS(0))

    // Бенчмарк с разными GOMAXPROCS
    for _, n := range []int{1, 2, 4, 8} {
        runtime.GOMAXPROCS(n)
        start := time.Now()
        parallelWork()
        fmt.Printf("GOMAXPROCS=%d: %v\n", n, time.Since(start))
    }
}

func parallelWork() {
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            sum := 0
            for j := 0; j < 1e7; j++ {
                sum += j
            }
        }()
    }
    wg.Wait()
}
```

**Типичные ошибки.**
- Ставить GOMAXPROCS > NumCPU для CPU-bound — увеличивает context switches.
- Ставить GOMAXPROCS=1 для продакшена — теряешь параллелизм.
- Не учитывать контейнерные лимиты — Go может использовать больше CPU, чем разрешено.

**На интервью.**
- Объясни, что GOMAXPROCS влияет на P, а M может быть больше (syscalls).
- Упомяни automaxprocs для контейнеров.
- Follow-up: «GOMAXPROCS=1 отключает параллелизм?» — нет, syscalls и netpoller работают параллельно.

---

### 8. Какими инструментами наблюдать за планировщиком?

**Зачем спрашивают.** Диагностика проблем с горутинами требует инструментов. Интервьюер проверяет практический опыт отладки.

**Короткий ответ.** `GODEBUG=schedtrace`, `go tool trace`, `pprof` goroutine profile, `runtime.Stack()`.

**Детальный разбор.**

**1. GODEBUG=schedtrace:**
```bash
GODEBUG=schedtrace=1000 ./myapp
# Вывод каждые 1000ms:
# SCHED 1000ms: gomaxprocs=8 idleprocs=5 threads=10
#               spinningthreads=1 idlethreads=3
#               runqueue=2 [1 0 0 5 0 0 0 0]

GODEBUG=schedtrace=1000,scheddetail=1 ./myapp
# Подробный вывод включая состояние каждого P и M
```

**2. go tool trace:**
```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    trace.Start(f)
    defer trace.Stop()

    // ... код программы
}
```
```bash
go run main.go
go tool trace trace.out
# Открывается веб-интерфейс с визуализацией
```

**3. pprof goroutine:**
```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil)
    // ...
}
```
```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
(pprof) top
(pprof) traces
```

**4. runtime.Stack():**
```go
func dumpGoroutines() {
    buf := make([]byte, 1<<20)
    n := runtime.Stack(buf, true) // true = все горутины
    fmt.Printf("%s\n", buf[:n])
}

// Или через pprof
pprof.Lookup("goroutine").WriteTo(os.Stdout, 1)
```

**Пример диагностики утечки горутин:**
```go
func main() {
    // Мониторинг числа горутин
    go func() {
        for {
            fmt.Printf("Goroutines: %d\n", runtime.NumGoroutine())
            time.Sleep(time.Second)
        }
    }()

    // Утечка: горутины не завершаются
    for i := 0; i < 100; i++ {
        go func() {
            ch := make(chan struct{})
            <-ch // навсегда заблокирована
        }()
    }

    time.Sleep(5 * time.Second)

    // Дамп для анализа
    pprof.Lookup("goroutine").WriteTo(os.Stdout, 1)
}
```

**Типичные ошибки.**
- Не знать про `schedtrace` — простейший способ увидеть состояние планировщика.
- Игнорировать `go tool trace` — мощнейший инструмент для понимания поведения.
- Не использовать `runtime.NumGoroutine()` для мониторинга в продакшене.

**На интервью.**
- Покажи, что знаешь несколько инструментов для разных задач.
- `schedtrace` — быстрая диагностика, `trace` — глубокий анализ, `pprof` — профилирование.
- Follow-up: «Как найти причину высокого числа горутин?» — `pprof.Lookup("goroutine")` покажет stack traces.

---

## См. также

- [Goroutines & Channels](./01-goroutines-channels.md) — использование горутин на практике
- [Performance & Profiling](./09-performance-profiling.md) — профилирование и оптимизация
- [Memory, Slices & Maps](./02-memory-slices-maps.md) — управление памятью
- [JVM Fundamentals](../01-java/00-jvm-fundamentals.md) — сравнение с Java runtime
- [Python Memory & GC](../02-python/02-memory-gc.md) — сравнение с Python GC

---

## Практика

1. **Scheduler Trace Explorer** — запусти программу с `GODEBUG=schedtrace=1000,scheddetail=1` и проанализируй вывод. Создай нагрузку разных типов (CPU, I/O, sleep) и наблюдай изменения.

2. **Stack Growth Experiment** — напиши рекурсивную функцию, измерь потребление стека через `runtime.MemStats.StackInuse`. Сравни с итеративной версией.

3. **Work Stealing Simulation** — реализуй простой планировщик с локальными очередями и work stealing. Сравни throughput с единой очередью.

4. **Syscall Impact** — создай программу с файловым и сетевым I/O. Наблюдай число M через `runtime` метрики или `ps`. Объясни разницу.

5. **GC Tuning** — запусти приложение с разными `GOGC` (50, 100, 200, off). Измерь latency и memory. Построй график trade-off.

6. **Goroutine Leak Detection** — создай утечку горутин, найди её через pprof, исправь добавив context/done channel.

---

## Дополнительные материалы

- [Go Memory Model](https://go.dev/ref/mem) — гарантии видимости памяти
- [Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw) — оригинальный дизайн от Dmitry Vyukov
- [Dmitry Vyukov: Go Scheduler](https://www.youtube.com/watch?v=-K11rY57K7k) — доклад автора планировщика
- [runtime package](https://pkg.go.dev/runtime) — документация runtime функций
- [GC Guide](https://tip.golang.org/doc/gc-guide) — официальный гайд по GC
- [go tool trace](https://pkg.go.dev/cmd/trace) — документация trace tool
