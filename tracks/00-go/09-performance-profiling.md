# 09 — Performance & Profiling

Разбор вопросов о поиске узких мест в Go-приложениях: профили и трассировки, настройка GC, наблюдаемость, overhead инструментов.

**Навигация:** [← 08-networking-grpc](./08-networking-grpc.md) | [Трек Go](./README.md) | [10-distributed-systems →](./10-distributed-systems.md)

---

## Вопросы и разборы

### 1. Какие профили доступны в Go и что они показывают?

**Зачем спрашивают.**
Понимание профилей — ключ к системной диагностике. Интервьюер проверяет, знает ли кандидат, какой инструмент использовать для какой проблемы (CPU, память, блокировки).

**Короткий ответ.**
CPU profile — горячие функции. Heap profile — аллокации и использование памяти. Block/Mutex — ожидания на каналах и мьютексах. Goroutine — стеки всех горутин (для поиска утечек).

**Детальный разбор.**

**Типы профилей:**
```
┌──────────────────┬───────────────────────────────────────────┐
│ Профиль          │ Что показывает                            │
├──────────────────┼───────────────────────────────────────────┤
│ CPU              │ Где тратится процессорное время           │
│                  │ Сэмплирование каждые ~10мс                │
├──────────────────┼───────────────────────────────────────────┤
│ Heap (inuse)     │ Сколько памяти удерживается сейчас        │
│ Heap (alloc)     │ Сколько памяти было выделено всего        │
├──────────────────┼───────────────────────────────────────────┤
│ Allocs           │ Количество аллокаций (не размер)          │
├──────────────────┼───────────────────────────────────────────┤
│ Block            │ Где горутины ждут на каналах/select       │
├──────────────────┼───────────────────────────────────────────┤
│ Mutex            │ Contention на мьютексах (кто ждёт lock)   │
├──────────────────┼───────────────────────────────────────────┤
│ Goroutine        │ Стеки всех горутин (debug=1: счётчики,    │
│                  │ debug=2: полные стеки)                    │
├──────────────────┼───────────────────────────────────────────┤
│ Threadcreate     │ Создание OS-потоков                       │
└──────────────────┴───────────────────────────────────────────┘
```

**Когда какой профиль использовать:**
```
Симптом                          → Профиль
─────────────────────────────────────────────
Высокий CPU                      → cpu
Растёт память                    → heap (inuse)
Много аллокаций / частый GC      → heap (alloc), allocs
Низкий throughput при низком CPU → block, mutex
Горутины не завершаются          → goroutine
```

**Пример.**
```bash
# Сбор профилей через go test
$ go test -run=^$ -bench=BenchmarkProcess \
    -cpuprofile cpu.out \
    -memprofile mem.out \
    -blockprofile block.out \
    -mutexprofile mutex.out

# Анализ CPU профиля
$ go tool pprof cpu.out
(pprof) top10          # топ функций по времени
(pprof) top -cum       # топ по кумулятивному времени
(pprof) list processBatch  # исходный код функции
(pprof) web            # граф в браузере

# Web UI (рекомендуется)
$ go tool pprof -http=:8080 cpu.out
```

**Сбор профилей через HTTP:**
```bash
# CPU профиль (30 секунд)
$ curl -o cpu.prof "http://localhost:6060/debug/pprof/profile?seconds=30"

# Heap профиль
$ curl -o heap.prof "http://localhost:6060/debug/pprof/heap"

# Все горутины с полными стеками
$ curl "http://localhost:6060/debug/pprof/goroutine?debug=2"

# Block профиль (нужен runtime.SetBlockProfileRate)
$ curl -o block.prof "http://localhost:6060/debug/pprof/block"
```

**Типичные ошибки.**
1. **CPU профиль без нагрузки** — профиль будет пустым
2. **Слишком короткий CPU профиль** — нужно минимум 30 секунд
3. **Не включать block/mutex rate** — профиль не собирается по умолчанию
4. **Игнорировать alloc vs inuse** — inuse для текущих утечек, alloc для частых аллокаций

**На интервью.**
Объясни разницу между heap inuse и alloc. Покажи, как найти goroutine leak через профиль. Расскажи, почему CPU профиль сэмплирует, а не трассирует.

*Частые follow-up вопросы:*
- Как включить mutex профиль?
- Что такое cumulative time в pprof?
- Как сравнить два профиля?

---

### 2. Как строить процесс оптимизации?

**Зачем спрашивают.**
Оптимизация без методологии — гадание. Интервьюер проверяет системный подход: измерение → анализ → изменение → повторное измерение.

**Короткий ответ.**
Measure, don't guess. Зафиксировать baseline метрику, собрать профиль, внести изменение, повторно измерить и сравнить. Важно контролировать условия эксперимента.

**Детальный разбор.**

**Цикл оптимизации:**
```
┌─────────────────────────────────────────────────────────────┐
│  1. DEFINE     Определить метрику (latency, QPS, memory)    │
│       ↓                                                      │
│  2. MEASURE    Зафиксировать baseline                       │
│       ↓                                                      │
│  3. PROFILE    Собрать профиль, найти bottleneck            │
│       ↓                                                      │
│  4. OPTIMIZE   Внести одно изменение                        │
│       ↓                                                      │
│  5. VALIDATE   Измерить снова, сравнить                     │
│       ↓                                                      │
│  6. ITERATE    Если цель не достигнута → п.3                │
└─────────────────────────────────────────────────────────────┘
```

**Контроль условий:**
```
┌────────────────────┬────────────────────────────────────────┐
│ Параметр           │ Что контролировать                     │
├────────────────────┼────────────────────────────────────────┤
│ GOMAXPROCS         │ Число P (обычно = CPU cores)           │
│ GOGC               │ Цель роста heap (default 100)          │
│ Данные             │ Одинаковый input dataset               │
│ Warmup             │ Прогрев перед измерением               │
│ Итерации           │ Достаточное количество (-count=10)     │
│ Hardware           │ Тот же сервер, нет interference        │
└────────────────────┴────────────────────────────────────────┘
```

**Пример.**
```go
func BenchmarkEncode(b *testing.B) {
    payload := generatePayload() // подготовка данных

    b.ReportAllocs()  // отчёт по аллокациям
    b.ResetTimer()    // не считать setup

    for i := 0; i < b.N; i++ {
        _ = Encode(payload)
    }
}

func BenchmarkEncode_Optimized(b *testing.B) {
    payload := generatePayload()
    buf := make([]byte, 0, 1024)  // prealloc

    b.ReportAllocs()
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        buf = EncodeWithBuffer(payload, buf[:0])
    }
}
```

**Сравнение результатов с benchstat:**
```bash
# До оптимизации
$ go test -bench=BenchmarkEncode -count=10 > old.txt

# После оптимизации
$ go test -bench=BenchmarkEncode -count=10 > new.txt

# Сравнение
$ benchstat old.txt new.txt
name          old time/op    new time/op    delta
Encode-8      1.20µs ± 2%    0.85µs ± 1%  -29.17%  (p=0.000 n=10+10)

name          old alloc/op   new alloc/op   delta
Encode-8        256B ± 0%        0B ± 0%  -100.00%  (p=0.000 n=10+10)
```

**Типичные ошибки.**
1. **Оптимизация без профиля** — гадание, что медленно
2. **Одно измерение** — нужно `-count=10` для статистики
3. **Несколько изменений сразу** — непонятно, что помогло
4. **Setup в измерении** — забыть `b.ResetTimer()`

**На интервью.**
Опиши реальный кейс оптимизации: что измерял, что нашёл в профиле, как исправил, какой результат. Покажи понимание, что premature optimization — зло.

*Частые follow-up вопросы:*
- Как избежать compiler optimizations в бенчмарке?
- Что такое p-value в benchstat?
- Когда оптимизация не нужна?

---

### 3. Как настроить `go test` для профилирования?

**Зачем спрашивают.**
`go test` — основной инструмент для бенчмарков и сбора профилей. Интервьюер проверяет знание флагов и практик.

**Короткий ответ.**
`go test -bench` запускает бенчмарки. Флаги `-cpuprofile`, `-memprofile`, `-trace` собирают профили. `b.ReportAllocs()` добавляет метрики аллокаций.

**Детальный разбор.**

**Основные флаги:**
```
┌─────────────────────┬──────────────────────────────────────────┐
│ Флаг                │ Назначение                               │
├─────────────────────┼──────────────────────────────────────────┤
│ -bench=.            │ Запустить все бенчмарки                  │
│ -bench=BenchmarkX   │ Запустить конкретный бенчмарк            │
│ -run=^$             │ Пропустить unit тесты                    │
│ -benchtime=5s       │ Время на каждый бенчмарк                 │
│ -benchtime=1000x    │ Количество итераций                      │
│ -count=10           │ Повторить N раз (для benchstat)          │
│ -benchmem           │ Показать аллокации                       │
├─────────────────────┼──────────────────────────────────────────┤
│ -cpuprofile=X.out   │ Собрать CPU профиль                      │
│ -memprofile=X.out   │ Собрать heap профиль                     │
│ -blockprofile=X.out │ Собрать block профиль                    │
│ -mutexprofile=X.out │ Собрать mutex профиль                    │
│ -trace=X.out        │ Собрать execution trace                  │
└─────────────────────┴──────────────────────────────────────────┘
```

**Пример.**
```bash
# Полный набор для анализа
$ go test ./pkg/encoding \
    -run=^$ \
    -bench=BenchmarkProcess \
    -benchtime=5s \
    -benchmem \
    -cpuprofile=cpu.out \
    -memprofile=mem.out \
    -trace=trace.out

# Результат:
# BenchmarkProcess-8   50000   25000 ns/op   1024 B/op   8 allocs/op
```

**Методы testing.B:**
```go
func BenchmarkComplex(b *testing.B) {
    // Setup — не измеряется
    data := loadTestData()

    b.ReportAllocs()     // включить отчёт по памяти
    b.SetBytes(1024)     // для расчёта MB/s
    b.ResetTimer()       // сбросить таймер после setup

    for i := 0; i < b.N; i++ {
        result := process(data)

        // Предотвращение оптимизации компилятором
        if result == nil {
            b.Fatal("unexpected nil")
        }
    }

    b.StopTimer()        // остановить для cleanup
    cleanup()
}

// Sub-benchmarks
func BenchmarkEncode(b *testing.B) {
    sizes := []int{64, 256, 1024, 4096}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := make([]byte, size)
            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                encode(data)
            }
        })
    }
}
```

**Типичные ошибки.**
1. **Setup в измерении** — забыть `b.ResetTimer()`
2. **Результат не используется** — компилятор может убрать код
3. **Мало итераций** — для сравнения нужен `-count=10+`
4. **Профили без бенчмарков** — профиль будет от тестов, не от целевого кода

**На интервью.**
Покажи типичную команду для бенчмарка с профилем. Объясни, зачем `b.ResetTimer()` и `b.ReportAllocs()`. Расскажи про sub-benchmarks.

*Частые follow-up вопросы:*
- Как профилировать конкретный бенчмарк?
- Что такое b.N и как он определяется?
- Как бенчмаркать с разными входными данными?

---

### 4. Как управлять GC и что означают метрики gctrace?

**Зачем спрашивают.**
GC — источник latency spikes. Интервьюер проверяет понимание триггеров GC, влияния GOGC и умение читать диагностику.

**Короткий ответ.**
`GOGC` контролирует частоту GC (100 = удвоение heap). `GODEBUG=gctrace=1` выводит детали каждого цикла. `runtime/metrics` даёт программный доступ к статистике GC.

**Детальный разбор.**

**Как работает GOGC:**
```
┌─────────────────────────────────────────────────────────────┐
│ GOGC=100 (default)                                          │
│                                                              │
│ GC триггерится когда heap = 2 × heap_after_last_gc          │
│                                                              │
│ Пример: после GC heap = 100MB                               │
│         следующий GC при heap = 200MB                       │
├─────────────────────────────────────────────────────────────┤
│ GOGC=50  → чаще GC, меньше памяти, больше CPU               │
│ GOGC=200 → реже GC, больше памяти, меньше CPU               │
│ GOGC=off → GC отключён (только для диагностики!)            │
└─────────────────────────────────────────────────────────────┘
```

**Формат gctrace:**
```
gc 1 @0.006s 2%: 0.015+0.52+0.013 ms clock, 0.12+0.21/0.36/0+0.10 ms cpu, 4->4->1 MB, 5 MB goal, 8 P

│  │     │    │       │                   │                      │          │       │
│  │     │    │       │                   │                      │          │       └─ GOMAXPROCS
│  │     │    │       │                   │                      │          └─ target heap
│  │     │    │       │                   │                      └─ heap before->after->live
│  │     │    │       │                   └─ CPU время: idle/mark/scan
│  │     │    │       └─ wall clock: STW₁ + concurrent + STW₂
│  │     │    └─ % CPU на GC с начала программы
│  │     └─ время с начала программы
│  └─ номер GC цикла
└─ тип (gc = полный, scvg = scavenger)
```

**Пример.**
```bash
# Запуск с gctrace
$ GODEBUG=gctrace=1 ./myapp

# Сравнение разных GOGC
$ GOGC=50 GODEBUG=gctrace=1 ./myapp   # чаще GC
$ GOGC=200 GODEBUG=gctrace=1 ./myapp  # реже GC
```

**Программный контроль GC:**
```go
import (
    "runtime"
    "runtime/debug"
)

func main() {
    // Установить GOGC программно
    debug.SetGCPercent(50)

    // Отключить GC временно (опасно!)
    debug.SetGCPercent(-1)
    defer debug.SetGCPercent(100)

    // Форсировать GC
    runtime.GC()

    // Освободить память OS
    debug.FreeOSMemory()
}
```

**runtime/metrics для GC статистики:**
```go
import "runtime/metrics"

func getGCStats() {
    samples := make([]metrics.Sample, 3)
    samples[0].Name = "/gc/cycles/total:gc-cycles"
    samples[1].Name = "/gc/heap/goal:bytes"
    samples[2].Name = "/gc/pauses:seconds"

    metrics.Read(samples)

    fmt.Printf("GC cycles: %d\n", samples[0].Value.Uint64())
    fmt.Printf("Heap goal: %d bytes\n", samples[1].Value.Uint64())

    // Паузы — histogram
    hist := samples[2].Value.Float64Histogram()
    fmt.Printf("GC pause buckets: %v\n", hist.Counts)
}
```

**Типичные ошибки.**
1. **GOGC=off в проде** — OOM гарантирован
2. **Игнорировать паузы** — latency spikes на высоких перцентилях
3. **GOMAXPROCS=1 с большим heap** — GC concurrent mark не параллелится
4. **Удержание больших буферов** — heap не уменьшается

**На интервью.**
Объясни, что означают числа в gctrace. Расскажи trade-off между GOGC=50 и GOGC=200. Покажи, как уменьшить GC паузы (меньше аллокаций, sync.Pool).

*Частые follow-up вопросы:*
- Как GC влияет на latency?
- Что такое memory ballast?
- Когда имеет смысл GOMEMLIMIT?

---

### 5. Что даёт `go tool trace`?

**Зачем спрашивают.**
Trace показывает события runtime во времени — незаменимо для диагностики latency, блокировок и проблем с планировщиком. Интервьюер проверяет знание этого мощного инструмента.

**Короткий ответ.**
Trace фиксирует события runtime: создание/остановку горутин, GC, syscalls, network. Web UI показывает timeline и позволяет понять, почему горутина ждёт.

**Детальный разбор.**

**Что фиксирует trace:**
```
┌──────────────────────┬──────────────────────────────────────┐
│ Категория            │ События                              │
├──────────────────────┼──────────────────────────────────────┤
│ Goroutines           │ Create, Start, Stop, Block, Unblock  │
│ Scheduler            │ GoSched, GoPreempt, GoSleep          │
│ GC                   │ GCStart, GCDone, GCMarkAssist        │
│ System               │ Syscall, SyscallExit                 │
│ Network              │ Network poll, I/O events             │
│ User                 │ Пользовательские события             │
└──────────────────────┴──────────────────────────────────────┘
```

**Пример.**
```bash
# Сбор trace
$ go test -trace=trace.out -run=^$ -bench=BenchmarkProcess

# Из HTTP endpoint
$ curl -o trace.out "http://localhost:6060/debug/pprof/trace?seconds=5"

# Анализ
$ go tool trace trace.out
# Открывается веб-интерфейс
```

**Вкладки в UI:**
```
┌────────────────────────┬────────────────────────────────────┐
│ View trace             │ Timeline всех событий              │
│ Goroutine analysis     │ Статистика по горутинам            │
│ Network blocking       │ Ожидание сети                      │
│ Sync blocking          │ Ожидание мьютексов/каналов         │
│ Syscall blocking       │ Ожидание syscalls                  │
│ Scheduler latency      │ Задержки планировщика              │
│ User-defined tasks     │ Пользовательские задачи            │
└────────────────────────┴────────────────────────────────────┘
```

**Пользовательские события в trace:**
```go
import "runtime/trace"

func processRequest(ctx context.Context, req Request) {
    ctx, task := trace.NewTask(ctx, "processRequest")
    defer task.End()

    // Region для части обработки
    trace.WithRegion(ctx, "validate", func() {
        validate(req)
    })

    trace.WithRegion(ctx, "process", func() {
        process(req)
    })

    // Log событие
    trace.Log(ctx, "requestID", req.ID)
}
```

**Типичные ошибки.**
1. **Trace в проде** — overhead до 10x, только для staging
2. **Слишком длинный trace** — файл огромный, UI тормозит
3. **Нет нагрузки** — trace пустой
4. **Игнорировать Scheduler latency** — показывает проблемы с GOMAXPROCS

**На интервью.**
Объясни, когда trace лучше pprof (для понимания timing и блокировок). Покажи, как найти причину высокого latency через trace. Расскажи про user tasks и regions.

*Частые follow-up вопросы:*
- Какой overhead у trace?
- Как найти goroutine leak через trace?
- Что такое GCMarkAssist и почему он важен?

---

### 6. Как искать проблемы производительности на продакшене?

**Зачем спрашивают.**
Production profiling требует баланса между диагностикой и overhead. Интервьюер проверяет практический опыт и понимание рисков.

**Короткий ответ.**
pprof endpoint на отдельном порту (защищённый). Короткие профили (30 сек) под реальной нагрузкой. Prometheus метрики для мониторинга. Structured logging для корреляции.

**Детальный разбор.**

**Архитектура для production profiling:**
```
┌─────────────────────────────────────────────────────────────┐
│                        Production                           │
├─────────────────────────────────────────────────────────────┤
│  :8080 (public)          :6060 (internal only)              │
│  ┌─────────────────┐     ┌─────────────────┐               │
│  │  Application    │     │  /debug/pprof   │               │
│  │  /api/*         │     │  /metrics       │               │
│  │  /health        │     │  /debug/vars    │               │
│  └─────────────────┘     └─────────────────┘               │
│                                │                            │
│                          ┌─────┴─────┐                      │
│                          │ Auth/mTLS │                      │
│                          └───────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Безопасный pprof endpoint:**
```go
import (
    "net/http"
    "net/http/pprof"
)

func startDebugServer() {
    mux := http.NewServeMux()

    // Регистрируем pprof handlers вручную
    mux.HandleFunc("/debug/pprof/", pprof.Index)
    mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    mux.HandleFunc("/debug/pprof/trace", pprof.Trace)

    // Только на localhost!
    srv := &http.Server{
        Addr:    "127.0.0.1:6060",
        Handler: mux,
    }

    go srv.ListenAndServe()
}

// Или с basic auth
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        user, pass, ok := r.BasicAuth()
        if !ok || !checkCredentials(user, pass) {
            w.Header().Set("WWW-Authenticate", `Basic realm="pprof"`)
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

**Снятие профиля под нагрузкой:**
```bash
# Терминал 1: генерируем нагрузку
$ wrk -t4 -c32 -d60s http://localhost:8080/api/process

# Терминал 2: снимаем профиль
$ go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
(pprof) top10
(pprof) web
```

**Автоматический профиль при аномалии:**
```go
import (
    "runtime/pprof"
    "os"
    "time"
)

func watchLatency(latencyChan <-chan time.Duration) {
    for latency := range latencyChan {
        if latency > 100*time.Millisecond {
            // Высокая latency — снять профиль
            saveProfile()
        }
    }
}

func saveProfile() {
    f, _ := os.Create(fmt.Sprintf("heap-%d.prof", time.Now().Unix()))
    defer f.Close()
    pprof.WriteHeapProfile(f)
}
```

**Типичные ошибки.**
1. **pprof на публичном порту** — security risk
2. **Длинный CPU профиль** — overhead, замедление
3. **Нет нагрузки при профилировании** — профиль бесполезен
4. **Игнорирование метрик** — проблема уже ушла когда снял профиль

**На интервью.**
Расскажи, как настроить безопасный pprof в production. Объясни процесс диагностики: metrics → alert → profile. Покажи, как корелировать slow requests с профилями.

*Частые follow-up вопросы:*
- Как защитить pprof endpoint?
- Какой overhead у heap профиля?
- Как автоматизировать сбор профилей?

---

### 7. Какие стандартные метрики стоит экспортировать?

**Зачем спрашивают.**
Метрики — основа мониторинга. Интервьюер проверяет понимание RED method, runtime метрик и best practices Prometheus.

**Короткий ответ.**
RED: Rate, Errors, Duration для каждого endpoint. Runtime: горутины, heap, GC. Business: domain-specific метрики. `prometheus/client_golang` предоставляет готовые collectors.

**Детальный разбор.**

**RED method:**
```
┌─────────────────┬────────────────────────────────────────┐
│ Rate            │ Requests per second                    │
│ Errors          │ Error rate (4xx, 5xx)                  │
│ Duration        │ Request latency (histogram)            │
└─────────────────┴────────────────────────────────────────┘
```

**USE method (для ресурсов):**
```
┌─────────────────┬────────────────────────────────────────┐
│ Utilization     │ % времени ресурс занят                 │
│ Saturation      │ Очередь к ресурсу                      │
│ Errors          │ Ошибки ресурса                         │
└─────────────────┴────────────────────────────────────────┘
```

**Пример.**
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/collectors"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // RED metrics
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Namespace: "myservice",
            Name:      "http_requests_total",
            Help:      "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Namespace: "myservice",
            Name:      "http_request_duration_seconds",
            Help:      "HTTP request duration",
            Buckets:   []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )

    // Business metrics
    ordersProcessed = prometheus.NewCounter(
        prometheus.CounterOpts{
            Namespace: "myservice",
            Name:      "orders_processed_total",
            Help:      "Total orders processed",
        },
    )

    queueSize = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Namespace: "myservice",
            Name:      "queue_size",
            Help:      "Current queue size",
        },
    )
)

func init() {
    // Custom registry (рекомендуется)
    registry := prometheus.NewRegistry()

    // Go runtime metrics
    registry.MustRegister(collectors.NewGoCollector())
    registry.MustRegister(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}))

    // Application metrics
    registry.MustRegister(httpRequestsTotal)
    registry.MustRegister(httpRequestDuration)
    registry.MustRegister(ordersProcessed)
    registry.MustRegister(queueSize)

    // Handler
    http.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{}))
}
```

**runtime/metrics для детальной статистики:**
```go
import "runtime/metrics"

func exportRuntimeMetrics(reg prometheus.Registerer) {
    descs := metrics.All()

    for _, desc := range descs {
        // Пример: /gc/heap/allocs:bytes
        name := strings.ReplaceAll(desc.Name, "/", "_")
        name = strings.ReplaceAll(name, ":", "_")

        // Создать Prometheus метрику на основе типа
        switch desc.Kind {
        case metrics.KindUint64:
            // Counter или Gauge
        case metrics.KindFloat64Histogram:
            // Histogram
        }
    }
}
```

**Типичные ошибки.**
1. **High cardinality** — path с ID создаёт миллионы серий
2. **Нет buckets для latency** — histogram бесполезен
3. **Забыть runtime metrics** — не видно горутины, GC
4. **Counter вместо Gauge** — для queue size нужен Gauge

**На интервью.**
Объясни RED vs USE. Покажи, как избежать cardinality explosion. Расскажи, как выбрать buckets для histogram.

*Частые follow-up вопросы:*
- Когда Counter, когда Gauge, когда Histogram?
- Как агрегировать метрики с разных инстансов?
- Что такое exemplars?

---

### 8. Как встроить стандартный профайлер в приложение?

**Зачем спрашивают.**
pprof endpoint — стандартный способ профилирования. Интервьюер проверяет знание setup и security considerations.

**Короткий ответ.**
Import `_ "net/http/pprof"` + HTTP сервер на отдельном порту. Защита через localhost, mTLS или basic auth. Не экспонировать публично.

**Детальный разбор.**

**Способы интеграции:**
```
┌───────────────────────────────────────────────────────────────┐
│ 1. Простой (DefaultServeMux)                                 │
│    import _ "net/http/pprof"                                 │
│    http.ListenAndServe(":6060", nil)                         │
├───────────────────────────────────────────────────────────────┤
│ 2. Кастомный mux                                             │
│    mux.HandleFunc("/debug/pprof/", pprof.Index)              │
│    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)     │
├───────────────────────────────────────────────────────────────┤
│ 3. С middleware (auth)                                       │
│    mux.Handle("/debug/pprof/", authMiddleware(pprof.Index))  │
└───────────────────────────────────────────────────────────────┘
```

**Пример.**

**Минимальный setup:**
```go
import (
    "log"
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // pprof на отдельном порту
    go func() {
        log.Println("pprof server on :6060")
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // основной сервер
    http.ListenAndServe(":8080", appHandler)
}
```

**Production-ready с защитой:**
```go
func startProfileServer(cfg Config) {
    mux := http.NewServeMux()

    // Регистрируем handlers вручную
    mux.HandleFunc("/debug/pprof/", pprof.Index)
    mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
    mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
    mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
    mux.HandleFunc("/debug/pprof/trace", pprof.Trace)

    // Добавляем /debug/vars для expvar
    mux.Handle("/debug/vars", expvar.Handler())

    // Обёртка с auth
    handler := basicAuth(mux, cfg.ProfileUser, cfg.ProfilePass)

    srv := &http.Server{
        Addr:         cfg.ProfileAddr, // "127.0.0.1:6060"
        Handler:      handler,
        ReadTimeout:  60 * time.Second,
        WriteTimeout: 60 * time.Second, // для длинных профилей
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil {
            log.Printf("profile server: %v", err)
        }
    }()
}

func basicAuth(next http.Handler, user, pass string) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        u, p, ok := r.BasicAuth()
        if !ok || subtle.ConstantTimeCompare([]byte(u), []byte(user)) != 1 ||
           subtle.ConstantTimeCompare([]byte(p), []byte(pass)) != 1 {
            w.Header().Set("WWW-Authenticate", `Basic realm="pprof"`)
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

**Включение block/mutex профилей:**
```go
import "runtime"

func init() {
    // Block профиль: записывать блокировки > 1ms
    runtime.SetBlockProfileRate(1_000_000) // наносекунды

    // Mutex профиль: записывать 1 из 5 событий
    runtime.SetMutexProfileFraction(5)
}
```

**Типичные ошибки.**
1. **DefaultServeMux в проде** — конфликт с основным сервером
2. **Публичный pprof** — security risk
3. **WriteTimeout < профиля** — обрезается ответ
4. **Не включать block/mutex rate** — профили пустые

**На интервью.**
Покажи production-ready настройку с auth. Объясни, почему отдельный порт лучше. Расскажи про SetBlockProfileRate/SetMutexProfileFraction.

*Частые follow-up вопросы:*
- Как защитить pprof в Kubernetes?
- Можно ли оставить pprof включённым постоянно?
- Как получить профиль удалённо?

---

### 9. Каков overhead стандартного профайлера?

**Зачем спрашивают.**
Понимание overhead критично для принятия решения о профилировании в production. Интервьюер проверяет знание компромиссов.

**Короткий ответ.**
CPU профиль: 5-10% overhead во время сбора. Heap профиль: минимальный (snapshot). Trace: до 10x замедление. pprof endpoint без запросов: 0% overhead.

**Детальный разбор.**

**Overhead по типам:**
```
┌─────────────────┬────────────────┬──────────────────────────┐
│ Инструмент      │ Overhead       │ Когда использовать       │
├─────────────────┼────────────────┼──────────────────────────┤
│ pprof endpoint  │ 0% (idle)      │ Всегда можно держать     │
│ CPU profile     │ 5-10%          │ Короткие сессии (30s)    │
│ Heap profile    │ <1%            │ Можно чаще               │
│ Block profile   │ 1-5%           │ При проблемах с блоками  │
│ Mutex profile   │ 1-5%           │ При contention           │
│ Trace           │ 10-100x        │ Только staging/dev       │
│ Race detector   │ 2-10x          │ Только тесты             │
└─────────────────┴────────────────┴──────────────────────────┘
```

**Механизм сэмплирования CPU:**
```
┌─────────────────────────────────────────────────────────────┐
│ CPU профиль не трассирует каждую инструкцию                 │
│                                                              │
│ Вместо этого: SIGPROF каждые ~10ms → snapshot стека         │
│                                                              │
│ 30 сек профиль = ~3000 samples                              │
│ Статистически значимо для горячих функций                   │
│ Может пропустить редкие события                             │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**
```bash
# Измерение влияния CPU профиля на latency

# Терминал 1: baseline
$ wrk -t4 -c32 -d30s http://localhost:8080
# Requests/sec: 50000
# Latency p99: 5ms

# Терминал 2: с профилем
$ curl http://localhost:6060/debug/pprof/profile?seconds=30 > /dev/null &

# Терминал 1: повторно
$ wrk -t4 -c32 -d30s http://localhost:8080
# Requests/sec: 47000 (-6%)
# Latency p99: 5.3ms (+6%)
```

**Стратегии минимизации overhead:**
```go
// Continuous profiling с низким overhead
// Используют pyroscope, parca, Google Cloud Profiler

// Пример: periodic heap snapshots
func periodicHeapProfile(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for range ticker.C {
        f, err := os.CreateTemp("", "heap-*.prof")
        if err != nil {
            continue
        }

        pprof.WriteHeapProfile(f)
        f.Close()

        // Ротация старых профилей
        cleanupOldProfiles()
    }
}
```

**Типичные ошибки.**
1. **Trace в production** — недопустимый overhead
2. **Длинные CPU профили** — накопленный overhead
3. **Race detector в проде** — 10x замедление
4. **SetBlockProfileRate(1)** — каждое событие = overhead

**На интервью.**
Объясни, как CPU профиль сэмплирует. Сравни overhead разных профилей. Расскажи про continuous profiling и когда он оправдан.

*Частые follow-up вопросы:*
- Можно ли профилировать production постоянно?
- Как выбрать sample rate для block профиля?
- Что такое continuous profiling?

---

### 10. Какие типичные узкие места стоит искать?

**Зачем спрашивают.**
Опыт показывает паттерны проблем. Интервьюер проверяет знание типичных bottlenecks в Go и умение их избегать.

**Короткий ответ.**
Лишние аллокации (string↔[]byte), lock contention (глобальные map), блокировки каналов, fmt.Sprintf в циклах, неограниченный параллелизм (goroutine explosion), большие heap → частый GC.

**Детальный разбор.**

**Категории bottlenecks:**
```
┌─────────────────────────────────────────────────────────────┐
│ CPU-bound                                                    │
│ • Горячие функции (видно в CPU profile)                     │
│ • Избыточные вычисления                                     │
│ • Reflection, regexp.MustCompile в цикле                    │
├─────────────────────────────────────────────────────────────┤
│ Memory-bound                                                 │
│ • Лишние аллокации                                          │
│ • string↔[]byte конверсии                                   │
│ • Большие буферы без переиспользования                      │
├─────────────────────────────────────────────────────────────┤
│ Contention                                                   │
│ • Глобальные мьютексы                                       │
│ • sync.Map при частых записях                               │
│ • Каналы без буфера                                         │
├─────────────────────────────────────────────────────────────┤
│ I/O-bound                                                    │
│ • Сетевые запросы без таймаутов                             │
│ • Синхронные операции с диском                              │
│ • Нет connection pooling                                    │
├─────────────────────────────────────────────────────────────┤
│ GC-bound                                                     │
│ • Частые аллокации → частый GC                              │
│ • Большой heap → длинные паузы                              │
│ • Удержание ненужных объектов                               │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Аллокации:**
```go
// ПЛОХО: аллокация на каждой итерации
for _, s := range inputs {
    out := fmt.Sprintf("%s-%s", prefix, s)  // аллокация
    process(out)
}

// ХОРОШО: strings.Builder
var sb strings.Builder
for _, s := range inputs {
    sb.Reset()
    sb.WriteString(prefix)
    sb.WriteByte('-')
    sb.WriteString(s)
    process(sb.String())  // одна аллокация
}

// ЕЩЁ ЛУЧШЕ: работа с []byte
buf := make([]byte, 0, 256)
for _, s := range inputs {
    buf = buf[:0]
    buf = append(buf, prefix...)
    buf = append(buf, '-')
    buf = append(buf, s...)
    processBytes(buf)  // zero allocations
}
```

**Lock contention:**
```go
// ПЛОХО: глобальный mutex
var (
    mu    sync.Mutex
    cache = make(map[string]Value)
)

func Get(key string) Value {
    mu.Lock()
    defer mu.Unlock()
    return cache[key]
}

// ХОРОШО: sharded cache
type ShardedCache struct {
    shards [256]struct {
        sync.RWMutex
        data map[string]Value
    }
}

func (c *ShardedCache) Get(key string) Value {
    shard := &c.shards[hash(key)%256]
    shard.RLock()
    defer shard.RUnlock()
    return shard.data[key]
}
```

**Goroutine explosion:**
```go
// ПЛОХО: неограниченный параллелизм
for _, item := range items {
    go process(item)  // 1M items = 1M goroutines
}

// ХОРОШО: worker pool с ограничением
sem := make(chan struct{}, 100)  // max 100 goroutines
var wg sync.WaitGroup

for _, item := range items {
    sem <- struct{}{}
    wg.Add(1)

    go func(it Item) {
        defer wg.Done()
        defer func() { <-sem }()
        process(it)
    }(item)
}

wg.Wait()
```

**Типичные ошибки.**
1. **fmt.Sprintf в горячем пути** — strconv.Itoa быстрее
2. **regexp.MustCompile в цикле** — компилировать один раз
3. **append без prealloc** — многократное копирование
4. **defer в цикле** — накопление, выполнение в конце функции

**На интервью.**
Приведи конкретные примеры оптимизации из опыта. Покажи, как профиль помог найти проблему. Объясни trade-off между читаемостью и производительностью.

*Частые follow-up вопросы:*
- Как найти goroutine leak?
- Когда sync.Pool оправдан?
- Как оптимизировать JSON parsing?

---

### 11. Как работает GOMEMLIMIT и когда его использовать?

**Зачем спрашивают.**
GOMEMLIMIT (Go 1.19+) — новый инструмент управления памятью, заменяющий memory ballast. Интервьюер проверяет знание современных практик memory management.

**Короткий ответ.**
`GOMEMLIMIT` устанавливает soft limit на общее использование памяти Go runtime. GC станет агрессивнее при приближении к лимиту, избегая OOM. Заменяет хак с memory ballast.

**Детальный разбор.**

**Как работает:**
```
┌─────────────────────────────────────────────────────────────┐
│ Без GOMEMLIMIT (только GOGC=100)                            │
│                                                              │
│ heap: 100MB → GC → 50MB live → target 100MB                 │
│       200MB → GC → 100MB live → target 200MB                │
│       ...продолжает расти до OOM                            │
├─────────────────────────────────────────────────────────────┤
│ С GOMEMLIMIT=500MiB                                          │
│                                                              │
│ heap: 100MB → 200MB → 400MB                                 │
│       Приближаемся к 500MB → GC становится агрессивнее      │
│       GC запускается чаще, чтобы не превысить limit         │
│       Soft limit: может временно превысить при необходимости│
└─────────────────────────────────────────────────────────────┘
```

**GOMEMLIMIT vs GOGC:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ Параметр        │ Что контролирует                         │
├─────────────────┼──────────────────────────────────────────┤
│ GOGC            │ Цель роста heap (% от live heap)         │
│                 │ GOGC=100: GC при 2× live data            │
├─────────────────┼──────────────────────────────────────────┤
│ GOMEMLIMIT      │ Абсолютный лимит памяти (bytes)          │
│                 │ GC агрессивнее при приближении           │
├─────────────────┼──────────────────────────────────────────┤
│ Вместе          │ GC балансирует между GOGC и GOMEMLIMIT   │
│                 │ GOGC=off + GOMEMLIMIT — максимум памяти  │
└─────────────────┴──────────────────────────────────────────┘
```

**Пример.**
```bash
# Через переменную окружения
$ GOMEMLIMIT=512MiB ./myapp

# Суффиксы: B, KiB, MiB, GiB, TiB (IEC units)
$ GOMEMLIMIT=2GiB ./myapp

# В Kubernetes (80-90% от memory limit)
# resources.limits.memory: 1Gi
$ GOMEMLIMIT=900MiB ./myapp
```

**Программная установка:**
```go
import "runtime/debug"

func main() {
    // Установить лимит программно
    debug.SetMemoryLimit(512 * 1024 * 1024) // 512 MiB

    // Читать текущий лимит
    limit := debug.SetMemoryLimit(-1) // -1 = только чтение
    fmt.Printf("Memory limit: %d bytes\n", limit)
}
```

**Автоматическое определение в контейнере:**
```go
import (
    "os"
    "runtime/debug"
    "strconv"
)

func setMemoryLimitFromCgroup() {
    // Читаем лимит из cgroup v2
    data, err := os.ReadFile("/sys/fs/cgroup/memory.max")
    if err != nil {
        return
    }

    limit, err := strconv.ParseInt(strings.TrimSpace(string(data)), 10, 64)
    if err != nil {
        return
    }

    // Устанавливаем 90% от лимита контейнера
    debug.SetMemoryLimit(limit * 90 / 100)
}

// Или использовать automaxprocs/automemory:
// import _ "go.uber.org/automaxprocs"
// import _ "github.com/KimMachineGun/automemlimit"
```

**Memory ballast (устаревший подход):**
```go
// УСТАРЕВШИЙ СПОСОБ (до Go 1.19)
// Аллокация большого куска для "обмана" GC
var ballast = make([]byte, 1<<30) // 1GB ballast

// СОВРЕМЕННЫЙ СПОСОБ
// GOGC=off + GOMEMLIMIT
// или GOGC=100 + GOMEMLIMIT для баланса
```

**Типичные ошибки.**
1. **Слишком низкий лимит** — постоянный GC, CPU overhead
2. **Лимит = memory limit контейнера** — нет запаса для стеков, cgo
3. **GOGC=off без GOMEMLIMIT** — OOM гарантирован
4. **Игнорирование в Kubernetes** — OOMKilled при пиках

**На интервью.**
Объясни, почему GOMEMLIMIT лучше memory ballast. Покажи, как рассчитать лимит для Kubernetes (80-90% от memory.limits). Расскажи про взаимодействие с GOGC.

*Частые follow-up вопросы:*
- Что происходит при превышении GOMEMLIMIT?
- Как мониторить приближение к лимиту?
- Когда использовать GOGC=off с GOMEMLIMIT?

---

### 12. Что такое continuous profiling и как его использовать?

**Зачем спрашивают.**
Continuous profiling — практика постоянного сбора профилей в production. Интервьюер проверяет знание современных инструментов observability и способность находить проблемы ретроспективно.

**Короткий ответ.**
Continuous profiling — постоянный сбор low-overhead профилей в production. Позволяет анализировать историю и сравнивать версии. Инструменты: Pyroscope, Parca, Google Cloud Profiler, Datadog.

**Детальный разбор.**

**Зачем нужен continuous profiling:**
```
┌─────────────────────────────────────────────────────────────┐
│ Традиционный подход:                                        │
│                                                              │
│ 1. Получить alert о высоком latency                         │
│ 2. Подключиться к серверу                                   │
│ 3. Снять профиль вручную                                    │
│ 4. Проблема уже ушла... профиль бесполезен                 │
├─────────────────────────────────────────────────────────────┤
│ Continuous profiling:                                        │
│                                                              │
│ 1. Профили собираются постоянно (каждые 10-60 сек)         │
│ 2. Хранятся с метаданными (version, instance, time)        │
│ 3. Можно анализировать историю и сравнивать                │
│ 4. Находить регрессии при деплое новых версий              │
└─────────────────────────────────────────────────────────────┘
```

**Архитектура:**
```
┌─────────────────────────────────────────────────────────────┐
│                       Production                             │
│                                                              │
│ ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│ │ App #1  │  │ App #2  │  │ App #3  │  ← Go приложения      │
│ │ +agent  │  │ +agent  │  │ +agent  │  ← встроенный агент   │
│ └────┬────┘  └────┬────┘  └────┬────┘                       │
│      │            │            │                             │
│      └────────────┼────────────┘                             │
│                   │                                          │
│                   ▼                                          │
│          ┌──────────────────┐                                │
│          │ Profiling Server │  ← Pyroscope/Parca/Cloud      │
│          │ (storage + UI)   │                                │
│          └──────────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Pyroscope (self-hosted):**
```go
import "github.com/grafana/pyroscope-go"

func main() {
    pyroscope.Start(pyroscope.Config{
        ApplicationName: "my-service",
        ServerAddress:   "http://pyroscope:4040",

        // Какие профили собирать
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
            pyroscope.ProfileInuseObjects,
            pyroscope.ProfileInuseSpace,
            pyroscope.ProfileGoroutines,
            pyroscope.ProfileMutexCount,
            pyroscope.ProfileMutexDuration,
            pyroscope.ProfileBlockCount,
            pyroscope.ProfileBlockDuration,
        },

        // Лейблы для фильтрации
        Tags: map[string]string{
            "version":     "1.2.3",
            "environment": "production",
            "region":      "us-east-1",
        },
    })
    defer pyroscope.Stop()

    // ... application code
}
```

**Parca (Kubernetes-native):**
```yaml
# parca-agent DaemonSet автоматически профилирует все Go процессы
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: parca-agent
spec:
  template:
    spec:
      containers:
      - name: parca-agent
        image: ghcr.io/parca-dev/parca-agent
        args:
        - --node=$(NODE_NAME)
        - --store-address=parca:7070
        - --insecure
        securityContext:
          privileged: true  # для eBPF
```

**Google Cloud Profiler:**
```go
import "cloud.google.com/go/profiler"

func main() {
    if err := profiler.Start(profiler.Config{
        Service:        "my-service",
        ServiceVersion: "1.2.3",
        ProjectID:      "my-gcp-project",

        // CPU и heap профили включены по умолчанию
        MutexProfiling: true,  // опционально
    }); err != nil {
        log.Printf("profiler.Start: %v", err)
    }

    // ... application code
}
```

**Сравнение инструментов:**
```
┌─────────────────┬──────────────┬──────────────┬──────────────┐
│ Инструмент      │ Overhead     │ Особенности  │ Цена         │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ Pyroscope       │ 1-5%         │ Self-hosted  │ Free/OSS     │
│                 │              │ Grafana UI   │              │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ Parca           │ 1-5%         │ eBPF-based   │ Free/OSS     │
│                 │              │ K8s native   │              │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ Google Cloud    │ <1%          │ Managed      │ Включён в GCP│
│ Profiler        │              │ Auto-scaling │              │
├─────────────────┼──────────────┼──────────────┼──────────────┤
│ Datadog         │ 1-2%         │ Managed      │ Платный      │
│ Continuous Prof │              │ Full APM     │              │
└─────────────────┴──────────────┴──────────────┴──────────────┘
```

**Анализ с лейблами:**
```go
// Добавление контекстных лейблов
func handleRequest(w http.ResponseWriter, r *http.Request) {
    pyroscope.TagWrapper(context.Background(), pyroscope.Labels(
        "endpoint", r.URL.Path,
        "user_tier", getUserTier(r),
    ), func(ctx context.Context) {
        // ... обработка запроса
        // CPU время будет атрибутировано этим лейблам
    })
}
```

**Типичные ошибки.**
1. **Не включать в staging** — проблемы обнаруживаются только в prod
2. **Слишком редкая выборка** — теряются кратковременные пики
3. **Нет version tag** — сложно сравнивать релизы
4. **Игнорирование diff view** — главная фича для поиска регрессий

**На интервью.**
Объясни преимущества continuous profiling над ad-hoc профилированием. Покажи, как использовать лейблы для фильтрации. Расскажи про сравнение версий для поиска регрессий.

*Частые follow-up вопросы:*
- Как выбрать между Pyroscope и Parca?
- Какой overhead у continuous profiling?
- Как интегрировать с existing observability stack?

---

## Практика

1. **Профили:** Напиши бенчмарк для двух версий алгоритма. Собери CPU/heap профили, сравни с benchstat.

2. **GC Tuning:** Запусти приложение с `GODEBUG=gctrace=1` и разными GOGC (50, 100, 200). Сравни паузы и использование памяти.

3. **Contention:** Включи `runtime.SetMutexProfileFraction(5)`, найди горячий mutex в своей программе, оптимизируй.

4. **Production Setup:** Настрой безопасный pprof endpoint с basic auth. Сними профиль под нагрузкой с wrk/vegeta.

5. **Metrics:** Добавь Prometheus Go collector + RED метрики. Настрой dashboard в Grafana.

6. **Trace:** Собери execution trace, проанализируй scheduler latency и GC events. Добавь custom tasks/regions.

---

## Дополнительные материалы

- [Profiling Go Programs](https://go.dev/blog/pprof)
- [runtime/pprof package](https://pkg.go.dev/runtime/pprof)
- [runtime/trace package](https://pkg.go.dev/runtime/trace)
- [runtime/metrics package](https://pkg.go.dev/runtime/metrics)
- [A Guide to the Go Garbage Collector](https://tip.golang.org/doc/gc-guide)
- [google/pprof](https://github.com/google/pprof)
- [Prometheus Go client](https://prometheus.io/docs/guides/go-application/)
- Talks: Dave Cheney "High Performance Go Workshop"
