# 07 — Tooling & Testing

Ответы на вопросы про экосистему инструментов Go: модули, тесты, бенчмарки, статический анализ, сборку, профилирование.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [06-databases-persistence](./06-databases-persistence.md) · Следующая тема: [08-networking-grpc](./08-networking-grpc.md)

---

## Вопросы и разборы

### 1. Как устроены Go Modules и рабочий цикл разработки?

**Зачем спрашивают.**
Модули — основа управления зависимостями в Go. Интервьюер хочет убедиться, что кандидат понимает версионирование, lock-файлы и умеет решать проблемы с зависимостями.

**Короткий ответ.**
`go.mod` описывает модуль и его зависимости, `go.sum` хранит контрольные суммы для обеспечения воспроизводимости сборки. Основные команды: `go mod init`, `go get`, `go mod tidy`.

**Детальный разбор.**

Структура go.mod:
```
module github.com/acme/service   // имя модуля

go 1.21                          // минимальная версия Go

require (
    github.com/google/uuid v1.3.0  // прямые зависимости
)

require (
    golang.org/x/sys v0.5.0 // indirect  // транзитивные
)
```

Ключевые команды:
```
┌─────────────────┬─────────────────────────────────────────┐
│ Команда         │ Назначение                              │
├─────────────────┼─────────────────────────────────────────┤
│ go mod init     │ Инициализация нового модуля             │
│ go get pkg@v1   │ Добавление/обновление зависимости       │
│ go mod tidy     │ Синхронизация go.mod с кодом            │
│ go mod vendor   │ Копирование зависимостей в vendor/      │
│ go mod graph    │ Вывод графа зависимостей                │
│ go mod why pkg  │ Объяснение, зачем нужна зависимость     │
└─────────────────┴─────────────────────────────────────────┘
```

**Семантическое версионирование (SemVer):**
- `v1.2.3` — major.minor.patch
- `v0.x.x` и `v1.x.x` — совместимый API
- `v2.0.0+` — требует `/v2` в пути импорта

**Пример.**
```bash
# Инициализация модуля
$ go mod init github.com/acme/service

# Добавление зависимости с конкретной версией
$ go get github.com/google/uuid@v1.3.0

# Обновление всех зависимостей до последних minor/patch версий
$ go get -u ./...

# Удаление неиспользуемых и добавление отсутствующих
$ go mod tidy
```

**Локальная разработка с replace:**
```go
// go.mod
replace github.com/acme/lib => ../lib

// или для форка
replace github.com/original/pkg => github.com/myfork/pkg v1.0.0
```

**Типичные ошибки.**
1. **Не запускать `go mod tidy`** — в go.mod остаются лишние зависимости или пропущены новые
2. **Игнорировать go.sum** — файл должен быть в git для воспроизводимости
3. **Использовать `replace` в библиотеках** — replace работает только в main модуле
4. **Забывать про `/v2` путь** — при мажорном апгрейде нужно менять импорты

**На интервью.**
Объясни разницу между go.mod и go.sum, расскажи про SemVer и проблему diamond dependency (когда A→B→D v1, A→C→D v2).

*Частые follow-up вопросы:*
- Как Go выбирает версию при конфликте? (MVS — Minimal Version Selection)
- Что делать, если нужна конкретная версия транзитивной зависимости?
- Как работает vendor директория?

---

### 2. Какие практики тестирования распространены в Go?

**Зачем спрашивают.**
Тестирование — неотъемлемая часть культуры Go. Интервьюер оценивает знание стандартного пакета testing, паттернов написания тестов и понимание философии "table-driven tests".

**Короткий ответ.**
Стандарт — table-driven tests с `t.Run()` для субтестов. Для изоляции используют `t.Parallel()`, для очистки ресурсов — `t.Cleanup()`. Популярные библиотеки: testify для assertions, gomock/mockery для моков.

**Детальный разбор.**

**Структура тестового файла:**
```
pkg/
├── handler.go          // код
├── handler_test.go     // юнит-тесты
├── handler_integration_test.go  // интеграционные (с build tag)
└── testdata/           // тестовые данные (игнорируется go build)
```

**Виды тестов:**
```
┌──────────────────┬─────────────────────────────────────────┐
│ Тип              │ Характеристики                          │
├──────────────────┼─────────────────────────────────────────┤
│ Unit             │ Изолированные, быстрые, без I/O         │
│ Integration      │ С БД/сетью, используют build tags       │
│ E2E              │ Полный стек, обычно отдельный пакет     │
│ Benchmark        │ Измерение производительности            │
│ Fuzz             │ Автоматическая генерация входных данных │
│ Example          │ Документация + проверка вывода          │
└──────────────────┴─────────────────────────────────────────┘
```

**Пример.**
```go
func TestParseUserID(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {
            name:  "valid positive",
            input: "123",
            want:  123,
        },
        {
            name:  "valid zero",
            input: "0",
            want:  0,
        },
        {
            name:    "invalid negative",
            input:   "-1",
            wantErr: true,
        },
        {
            name:    "invalid string",
            input:   "abc",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // тесты выполняются параллельно

            got, err := ParseUserID(tt.input)

            if tt.wantErr {
                if err == nil {
                    t.Fatal("expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

**t.Cleanup для автоматической очистки:**
```go
func TestWithDatabase(t *testing.T) {
    db := setupTestDB(t)
    t.Cleanup(func() {
        db.Close()  // вызовется автоматически после теста
    })

    // ... тест
}
```

**Использование testify:**
```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestWithTestify(t *testing.T) {
    result, err := DoSomething()

    require.NoError(t, err)        // при ошибке — сразу остановка теста
    assert.Equal(t, expected, result)  // при ошибке — продолжение
    assert.Contains(t, result.Items, item)
}
```

**Типичные ошибки.**
1. **Забывать `tc := tc` в loop (до Go 1.22)** — замыкание захватывает переменную цикла
2. **t.Error вместо t.Fatal** — тест продолжается после ошибки, вызывая панику
3. **Тестирование приватных функций** — лучше тестировать публичный API
4. **Хрупкие тесты** — зависимость от времени, порядка, внешних сервисов

**На интервью.**
Объясни преимущества table-driven подхода: легко добавлять кейсы, субтесты запускаются отдельно (`go test -run TestParse/valid`), параллелизм ускоряет выполнение.

*Частые follow-up вопросы:*
- Как тестировать код с внешними зависимостями? (интерфейсы + моки)
- Разница между t.Error и t.Fatal?
- Как организовать интеграционные тесты?

---

### 3. Как использовать race detector, benchmarks и fuzzing?

**Зачем спрашивают.**
Это продвинутые инструменты для поиска data races, оптимизации производительности и обнаружения edge cases. Их знание показывает глубину понимания тестирования в Go.

**Короткий ответ.**
Race detector (`-race`) находит data races в рантайме. Benchmarks измеряют производительность с `b.N` итераций. Fuzzing автоматически генерирует входные данные для поиска crashes и паник.

**Детальный разбор.**

**Race Detector:**
```
┌─────────────────────────────────────────────────────────────┐
│ Race detector использует ThreadSanitizer для отслеживания  │
│ всех обращений к памяти и синхронизации                    │
├─────────────────────────────────────────────────────────────┤
│ • Замедляет выполнение в 2-10x                             │
│ • Увеличивает потребление памяти в 5-10x                   │
│ • Находит races только в исполняемом коде                  │
│ • Не находит 100% races (нужен конкретный interleaving)    │
└─────────────────────────────────────────────────────────────┘
```

**Benchmarks:**
```go
func BenchmarkEncode(b *testing.B) {
    // Подготовка данных вне измерений
    data := makeTestData()

    b.ReportAllocs()  // отчёт по аллокациям
    b.ResetTimer()    // сброс таймера после setup

    for i := 0; i < b.N; i++ {
        _ = Encode(data)
    }
}

// Benchmark с разными размерами
func BenchmarkSort(b *testing.B) {
    sizes := []int{100, 1000, 10000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := makeData(size)
            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}
```

**Fuzzing (Go 1.18+):**
```go
func FuzzParseJSON(f *testing.F) {
    // Seed corpus — начальные данные
    f.Add([]byte(`{"name": "test"}`))
    f.Add([]byte(`{}`))
    f.Add([]byte(`null`))

    f.Fuzz(func(t *testing.T, data []byte) {
        var result map[string]any

        // Fuzzer будет мутировать data и искать паники
        _ = json.Unmarshal(data, &result)

        // Или проверять инварианты
        if err := Validate(data); err == nil {
            // Если валидация прошла, парсинг не должен падать
            if _, err := Parse(data); err != nil {
                t.Errorf("valid input failed parse: %v", err)
            }
        }
    })
}
```

**Пример.**
```bash
# Race detector
$ go test -race ./...
$ go run -race main.go

# Benchmarks
$ go test -bench=. -benchmem ./pkg/encoding
$ go test -bench=BenchmarkEncode -count=10 -benchmem

# Сравнение результатов (с benchstat)
$ go test -bench=. -count=10 > old.txt
# ... внести изменения ...
$ go test -bench=. -count=10 > new.txt
$ benchstat old.txt new.txt

# Fuzzing
$ go test -fuzz=FuzzParse -fuzztime=30s
$ go test -fuzz=FuzzParse -fuzztime=1000x  # 1000 итераций
```

**Интерпретация результатов benchmark:**
```
BenchmarkEncode-8   1000000   1052 ns/op   256 B/op   4 allocs/op
     │               │          │           │          │
     │               │          │           │          └─ аллокаций на операцию
     │               │          │           └─ байт на операцию
     │               │          └─ наносекунд на операцию
     │               └─ количество итераций
     └─ число GOMAXPROCS
```

**Типичные ошибки.**
1. **Включать setup в измерение** — используй `b.ResetTimer()` после подготовки
2. **Компилятор оптимизирует код** — результат должен использоваться (записать в `sink`)
3. **Мало итераций при сравнении** — нужен `-count=10+` и benchstat
4. **Запускать race detector в проде** — только для тестов/staging

**На интервью.**
Приведи пример найденного race detector'ом бага. Объясни, как benchmark помог оптимизировать код. Покажи понимание, что fuzzing находит edge cases, которые люди не придумают.

*Частые follow-up вопросы:*
- Какие ограничения у race detector?
- Как избежать оптимизаций компилятора в бенчмарках?
- Где хранится corpus для fuzzing?

---

### 4. Какие статические анализаторы стоит подключить?

**Зачем спрашивают.**
Статический анализ предотвращает баги до запуска кода. Интервьюер хочет понять, как кандидат настраивает линтеры и какие проблемы они ловят.

**Короткий ответ.**
`go vet` — встроенный анализатор для базовых ошибок. `staticcheck` — продвинутый анализатор. `golangci-lint` — мета-линтер, объединяющий 50+ анализаторов с единой конфигурацией.

**Детальный разбор.**

**Иерархия инструментов:**
```
┌─────────────────────────────────────────────────────────────┐
│ gofmt / goimports       — форматирование                   │
├─────────────────────────────────────────────────────────────┤
│ go vet                  — базовый анализ (подозрительный   │
│                           код, printf аргументы, etc.)     │
├─────────────────────────────────────────────────────────────┤
│ staticcheck             — продвинутый анализ               │
│                           (deprecated API, ошибки логики)  │
├─────────────────────────────────────────────────────────────┤
│ golangci-lint           — агрегатор 50+ линтеров           │
│                           (govet, staticcheck, gosec, etc.)│
└─────────────────────────────────────────────────────────────┘
```

**Популярные линтеры в golangci-lint:**
```
┌──────────────┬─────────────────────────────────────────────┐
│ Линтер       │ Что проверяет                               │
├──────────────┼─────────────────────────────────────────────┤
│ errcheck     │ Проверка обработки возвращаемых ошибок      │
│ gosec        │ Безопасность (SQL injection, hardcoded pwd) │
│ gocritic     │ Стиль кода и performance hints              │
│ ineffassign  │ Неиспользуемые присваивания                 │
│ prealloc     │ Предложения по preallocate slices           │
│ unparam      │ Неиспользуемые параметры функций            │
│ gocyclo      │ Цикломатическая сложность                   │
│ misspell     │ Опечатки в комментариях и строках           │
│ bodyclose    │ Незакрытые http.Response.Body               │
└──────────────┴─────────────────────────────────────────────┘
```

**Пример.**

Конфигурация `.golangci.yml`:
```yaml
run:
  timeout: 5m
  tests: true

linters:
  enable:
    - govet
    - staticcheck
    - errcheck
    - gosec
    - gocritic
    - ineffassign
    - unused
    - bodyclose

  disable:
    - wsl          # слишком строгий по пробелам
    - gofumpt      # если уже используете gofmt

linters-settings:
  govet:
    enable-all: true

  errcheck:
    check-type-assertions: true
    check-blank: true

  gocritic:
    enabled-tags:
      - diagnostic
      - performance
    disabled-checks:
      - hugeParam  # иногда ложные срабатывания

issues:
  exclude-rules:
    # Не проверять ошибки в тестах для fmt.Println
    - path: _test\.go
      linters:
        - errcheck
      text: "fmt.Print"

    # Игнорировать deprecated в сгенерированном коде
    - path: "generated"
      linters:
        - staticcheck
      text: "SA1019"
```

**Команды:**
```bash
# Базовые проверки
$ gofmt -d ./...           # показать diff форматирования
$ go vet ./...             # встроенный анализ
$ staticcheck ./...        # продвинутый анализ

# golangci-lint
$ golangci-lint run ./...
$ golangci-lint run --fix  # автоисправление где возможно
$ golangci-lint linters    # список доступных линтеров
```

**Типичные ошибки.**
1. **Включать все линтеры сразу** — много шума, сложно внедрить в существующий проект
2. **Игнорировать предупреждения** — `//nolint` должен быть с объяснением
3. **Не кэшировать в CI** — golangci-lint может кэшировать результаты
4. **Разные версии локально и в CI** — фиксируй версию линтера

**На интервью.**
Расскажи о реальных багах, которые поймал линтер. Объясни, как постепенно внедрять линтеры в legacy проект (начать с критичных, добавлять по одному).

*Частые follow-up вопросы:*
- Как подавить ложные срабатывания?
- Какие линтеры включаете в первую очередь?
- Разница между `go vet` и `staticcheck`?

---

### 5. Как собирать и публиковать приложения?

**Зачем спрашивают.**
Сборка и деплой — важная часть доставки софта. Интервьюер проверяет понимание кросс-компиляции, оптимизации бинарников и инфраструктуры публикации.

**Короткий ответ.**
`go build` создаёт исполняемый файл. `GOOS`/`GOARCH` задают целевую платформу. Build tags позволяют условную компиляцию. `go install` публикует в `$GOBIN`.

**Детальный разбор.**

**Флаги сборки:**
```
┌─────────────────────┬───────────────────────────────────────┐
│ Флаг                │ Назначение                            │
├─────────────────────┼───────────────────────────────────────┤
│ -o output           │ Имя выходного файла                   │
│ -v                  │ Вывод компилируемых пакетов           │
│ -race               │ Включить race detector                │
│ -ldflags            │ Флаги линкера                         │
│ -trimpath           │ Убрать локальные пути из бинарника    │
│ -tags               │ Build tags для условной компиляции    │
└─────────────────────┴───────────────────────────────────────┘
```

**ldflags для инжекции данных:**
```go
// version.go
package main

var (
    Version   = "dev"
    Commit    = "unknown"
    BuildTime = "unknown"
)
```

```bash
$ go build -ldflags "-X main.Version=v1.0.0 \
                     -X main.Commit=$(git rev-parse HEAD) \
                     -X main.BuildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

**Кросс-компиляция:**
```bash
# Собрать для Linux ARM64
$ GOOS=linux GOARCH=arm64 go build -o bin/app-linux-arm64 ./cmd/app

# Собрать для Windows
$ GOOS=windows GOARCH=amd64 go build -o bin/app.exe ./cmd/app

# Посмотреть доступные платформы
$ go tool dist list
```

**Пример.**

**Build tags для условной компиляции:**
```go
//go:build linux && amd64
// +build linux,amd64

package main

// Этот файл компилируется только для linux/amd64
```

```go
//go:build integration

package mypackage

// Тесты здесь требуют: go test -tags=integration
```

**Makefile для автоматизации:**
```makefile
VERSION := $(shell git describe --tags --always)
COMMIT := $(shell git rev-parse HEAD)
BUILD_TIME := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
LDFLAGS := -ldflags "-X main.Version=$(VERSION) \
                     -X main.Commit=$(COMMIT) \
                     -X main.BuildTime=$(BUILD_TIME) \
                     -s -w"

.PHONY: build
build:
	go build $(LDFLAGS) -o bin/app ./cmd/app

.PHONY: build-all
build-all:
	GOOS=linux GOARCH=amd64 go build $(LDFLAGS) -o bin/app-linux-amd64 ./cmd/app
	GOOS=darwin GOARCH=arm64 go build $(LDFLAGS) -o bin/app-darwin-arm64 ./cmd/app
	GOOS=windows GOARCH=amd64 go build $(LDFLAGS) -o bin/app-windows-amd64.exe ./cmd/app

.PHONY: install
install:
	go install $(LDFLAGS) ./cmd/app
```

**Минимизация размера бинарника:**
```bash
# -s — убрать symbol table
# -w — убрать DWARF debug info
$ go build -ldflags "-s -w" -o app ./cmd/app

# С UPX (осторожно: увеличивает время старта)
$ upx --best app
```

**Типичные ошибки.**
1. **Забывать -trimpath** — в бинарнике остаются локальные пути
2. **Собирать с -race для прода** — замедляет выполнение в 10x
3. **CGO_ENABLED=1 по умолчанию** — может сломать кросс-компиляцию
4. **Не указывать версию** — сложно отлаживать в проде без версии

**На интервью.**
Покажи понимание процесса: от `go build` до Docker образа. Объясни зачем `-trimpath` и `-ldflags "-s -w"` для production builds.

*Частые follow-up вопросы:*
- Как уменьшить размер Docker образа с Go приложением?
- Что такое CGO и когда оно нужно?
- Как работают build tags?

---

### 6. Как интегрировать профилирование и трассировки?

**Зачем спрашивают.**
Профилирование необходимо для оптимизации production приложений. Интервьюер проверяет, знает ли кандидат инструменты и методологию поиска bottlenecks.

**Короткий ответ.**
`net/http/pprof` экспортирует профили через HTTP. `runtime/pprof` — программный API. `go tool pprof` анализирует профили. `go tool trace` визуализирует события планировщика.

**Детальный разбор.**

**Виды профилей:**
```
┌────────────────┬───────────────────────────────────────────┐
│ Профиль        │ Что показывает                            │
├────────────────┼───────────────────────────────────────────┤
│ CPU            │ Где тратится процессорное время           │
│ Heap           │ Аллокации в куче (inuse/alloc)            │
│ Allocs         │ Количество аллокаций (не размер)          │
│ Goroutine      │ Stack traces всех горутин                 │
│ Block          │ Где горутины блокируются                  │
│ Mutex          │ Contention на мьютексах                   │
│ Threadcreate   │ Создание OS-потоков                       │
└────────────────┴───────────────────────────────────────────┘
```

**Два способа сбора:**

1. **HTTP endpoint (для сервисов):**
```go
import (
    "net/http"
    _ "net/http/pprof"  // регистрирует handlers
)

func main() {
    // Отдельный порт для профилирования (безопасность!)
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // ... основной сервер
}
```

2. **Программный API (для CLI/тестов):**
```go
import "runtime/pprof"

func main() {
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // ... код
}
```

**Пример.**

**Сбор профилей:**
```bash
# CPU профиль (30 секунд)
$ curl -o cpu.prof "http://localhost:6060/debug/pprof/profile?seconds=30"

# Heap профиль
$ curl -o heap.prof "http://localhost:6060/debug/pprof/heap"

# Все горутины
$ curl "http://localhost:6060/debug/pprof/goroutine?debug=2"

# Из тестов
$ go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof
```

**Анализ с pprof:**
```bash
# Интерактивный режим
$ go tool pprof cpu.prof
(pprof) top10
(pprof) list FunctionName
(pprof) web  # открыть в браузере (требует graphviz)

# Web UI (рекомендуется)
$ go tool pprof -http=:8080 cpu.prof

# Сравнение профилей
$ go tool pprof -base=old.prof new.prof
```

**Трассировка:**
```bash
# Сбор trace
$ curl -o trace.out "http://localhost:6060/debug/pprof/trace?seconds=5"

# Или в тестах
$ go test -trace=trace.out

# Анализ
$ go tool trace trace.out
```

**runtime/metrics (Go 1.16+):**
```go
import "runtime/metrics"

func collectMetrics() {
    // Получить все доступные метрики
    descs := metrics.All()

    // Читать конкретные метрики
    samples := make([]metrics.Sample, 2)
    samples[0].Name = "/gc/heap/allocs:bytes"
    samples[1].Name = "/sched/goroutines:goroutines"

    metrics.Read(samples)

    fmt.Printf("Heap allocs: %d bytes\n", samples[0].Value.Uint64())
    fmt.Printf("Goroutines: %d\n", samples[1].Value.Uint64())
}
```

**Типичные ошибки.**
1. **pprof endpoint доступен публично** — только на localhost или с авторизацией
2. **Слишком короткий CPU профиль** — нужно минимум 30 секунд под нагрузкой
3. **Профилирование без нагрузки** — профиль будет пустым
4. **Игнорировать alloc профиль** — CPU может быть нормальным, но GC убивает latency

**На интервью.**
Опиши реальный кейс оптимизации с pprof. Покажи понимание разницы между inuse и alloc в heap профиле. Объясни, когда смотреть на trace vs pprof.

*Частые follow-up вопросы:*
- Как найти memory leak в Go?
- Что такое flame graph и как его читать?
- Как профилировать production без overhead?

---

### 7. Как использовать golden files для тестирования?

**Зачем спрашивают.**
Golden files упрощают тестирование вывода: шаблонов, сериализации, API responses. Интервьюер проверяет знание этого паттерна для сложных структурированных данных.

**Короткий ответ.**
Golden file — эталонный файл с ожидаемым выводом. Тест сравнивает фактический вывод с содержимым файла. Флаг `-update` перезаписывает golden files при легитимных изменениях.

**Детальный разбор.**

**Когда использовать:**
- Генерация кода, шаблонов, HTML
- JSON/YAML/XML сериализация
- CLI output
- Snapshot testing сложных структур

**Структура проекта:**
```
pkg/
├── template.go
├── template_test.go
└── testdata/
    ├── input/
    │   ├── case1.json
    │   └── case2.json
    └── golden/
        ├── case1.golden
        └── case2.golden
```

**Пример.**
```go
var updateGolden = flag.Bool("update", false, "update golden files")

func TestRenderTemplate(t *testing.T) {
    tests := []struct {
        name  string
        input string
    }{
        {"simple", "testdata/input/simple.json"},
        {"complex", "testdata/input/complex.json"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Читаем входные данные
            inputData, err := os.ReadFile(tt.input)
            require.NoError(t, err)

            // Выполняем функцию
            got, err := RenderTemplate(inputData)
            require.NoError(t, err)

            // Путь к golden file
            goldenPath := filepath.Join("testdata", "golden", tt.name+".golden")

            // Обновляем golden file если флаг установлен
            if *updateGolden {
                err := os.WriteFile(goldenPath, got, 0644)
                require.NoError(t, err)
                return
            }

            // Сравниваем с golden
            want, err := os.ReadFile(goldenPath)
            require.NoError(t, err)

            if diff := cmp.Diff(string(want), string(got)); diff != "" {
                t.Errorf("mismatch (-want +got):\n%s", diff)
            }
        })
    }
}
```

**Хелпер функция:**
```go
func assertGolden(t *testing.T, name string, got []byte) {
    t.Helper()

    goldenPath := filepath.Join("testdata", "golden", name+".golden")

    if *updateGolden {
        if err := os.MkdirAll(filepath.Dir(goldenPath), 0755); err != nil {
            t.Fatal(err)
        }
        if err := os.WriteFile(goldenPath, got, 0644); err != nil {
            t.Fatal(err)
        }
        return
    }

    want, err := os.ReadFile(goldenPath)
    if err != nil {
        t.Fatalf("failed to read golden file: %v", err)
    }

    if !bytes.Equal(want, got) {
        t.Errorf("output mismatch\nwant:\n%s\ngot:\n%s", want, got)
    }
}

// Использование
func TestGenerator(t *testing.T) {
    output := Generate(input)
    assertGolden(t, "generator_output", output)
}
```

**Типичные ошибки.**
1. **Нестабильный вывод** — timestamps, random IDs ломают сравнение
2. **Golden files не в git** — теряются при клонировании
3. **Слепой `-update`** — всегда проверяй diff перед коммитом
4. **Бинарные файлы** — сложнее diffable, рассмотри base64 или hex dump

**На интервью.**
Объясни workflow: создание теста, первый запуск с `-update`, ревью diff'а, коммит golden файлов. Покажи, как избежать проблем с нестабильным выводом (замена timestamps на плейсхолдеры).

*Частые follow-up вопросы:*
- Как обрабатывать нестабильные данные (timestamps, UUIDs)?
- Как организовать golden files в большом проекте?
- Когда golden files лучше обычных assertions?

---

### 8. Что такое property-based testing?

**Зачем спрашивают.**
Property-based testing (PBT) генерирует множество случайных входов и проверяет инварианты. Интервьюер оценивает знание продвинутых техник тестирования для нахождения edge cases.

**Короткий ответ.**
Вместо конкретных примеров описываем свойства (properties), которые должны выполняться для всех входов. Генератор создаёт случайные данные, shrinking находит минимальный провальный пример. В Go: `rapid`, `gopter`, встроенный `testing/quick`.

**Детальный разбор.**

**Сравнение подходов:**
```
┌─────────────────────────────────────────────────────────────┐
│ Example-based (table-driven):                               │
│   Input: [1, 2, 3]                                          │
│   Expected: 6                                               │
│   → Проверяет один конкретный случай                        │
├─────────────────────────────────────────────────────────────┤
│ Property-based:                                             │
│   Property: Sum(list) == Sum(Reverse(list))                 │
│   → Проверяет для тысяч случайных списков                   │
│   → Находит edge cases, которые человек не придумает        │
└─────────────────────────────────────────────────────────────┘
```

**Типичные свойства:**
- **Идемпотентность:** `f(f(x)) == f(x)` (например, Sort)
- **Roundtrip:** `Decode(Encode(x)) == x` (сериализация)
- **Инварианты:** `len(Filter(list)) <= len(list)`
- **Коммутативность:** `f(a, b) == f(b, a)`
- **Монотонность:** `if a > b then f(a) >= f(b)`

**Пример.**
```go
import "pgregory.net/rapid"

// Свойство: Encode → Decode = identity
func TestEncodeDecodeRoundtrip(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        // Генерируем случайную структуру
        original := User{
            ID:    rapid.Int64().Draw(t, "id"),
            Name:  rapid.String().Draw(t, "name"),
            Email: rapid.StringMatching(`[a-z]+@[a-z]+\.[a-z]+`).Draw(t, "email"),
        }

        // Encode → Decode
        encoded, err := json.Marshal(original)
        if err != nil {
            t.Fatal(err)
        }

        var decoded User
        err = json.Unmarshal(encoded, &decoded)
        if err != nil {
            t.Fatal(err)
        }

        // Проверяем roundtrip
        if original != decoded {
            t.Fatalf("roundtrip failed: %+v != %+v", original, decoded)
        }
    })
}

// Свойство: сортированный список остаётся сортированным
func TestSortIsIdempotent(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        list := rapid.SliceOf(rapid.Int()).Draw(t, "list")

        // Сортируем дважды
        sort.Ints(list)
        firstSort := append([]int{}, list...)

        sort.Ints(list)
        secondSort := list

        // Идемпотентность
        if !reflect.DeepEqual(firstSort, secondSort) {
            t.Fatal("sort is not idempotent")
        }
    })
}

// Свойство: Filter не увеличивает длину
func TestFilterPreservesLength(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        list := rapid.SliceOf(rapid.Int()).Draw(t, "list")

        filtered := Filter(list, func(x int) bool {
            return x > 0
        })

        if len(filtered) > len(list) {
            t.Fatal("filtered list is longer than original")
        }
    })
}
```

**Использование testing/quick (stdlib):**
```go
import "testing/quick"

func TestAddCommutative(t *testing.T) {
    f := func(a, b int) bool {
        return Add(a, b) == Add(b, a)
    }

    if err := quick.Check(f, nil); err != nil {
        t.Error(err)
    }
}
```

**Shrinking:**
```
┌─────────────────────────────────────────────────────────────┐
│ rapid нашёл провальный вход:                                │
│   list = [1000, -500, 234, 0, -1, 999]                      │
│                                                              │
│ После shrinking:                                            │
│   list = [-1]                                               │
│   → Минимальный пример, воспроизводящий баг                 │
└─────────────────────────────────────────────────────────────┘
```

**Типичные ошибки.**
1. **Тестирование implementation, не specification** — свойство не должно дублировать код
2. **Слишком слабые свойства** — `len(result) >= 0` всегда true
3. **Игнорирование shrinking** — важно для понимания бага
4. **Неограниченные генераторы** — могут создать слишком большие входы

**На интервью.**
Приведи пример свойства для конкретной функции из твоего опыта. Объясни shrinking и почему он важен. Покажи понимание ограничений PBT (не заменяет example-based тесты полностью).

*Частые follow-up вопросы:*
- Как выбрать хорошие свойства для тестирования?
- Чем PBT отличается от fuzzing?
- Когда PBT не подходит?

---

### 9. Как писать contract tests для API?

**Зачем спрашивают.**
Contract tests проверяют, что API соответствует контракту между сервисами. Интервьюер оценивает понимание тестирования в микросервисной архитектуре.

**Короткий ответ.**
Contract test проверяет взаимодействие между consumer и provider без запуска обоих. Consumer определяет ожидания (contract), provider проверяет, что удовлетворяет контракту. Инструменты: Pact, OpenAPI validators, custom mocks.

**Детальный разбор.**

**Проблема интеграционных тестов:**
```
┌─────────────────────────────────────────────────────────────┐
│ E2E тесты:                                                  │
│   Service A ────────────────▶ Service B                     │
│                                                              │
│   Проблемы:                                                 │
│   • Медленные (нужно поднять всё)                          │
│   • Хрупкие (зависят от окружения)                         │
│   • Сложно понять, кто сломал                              │
├─────────────────────────────────────────────────────────────┤
│ Contract tests:                                             │
│                                                              │
│   Consumer test:        Provider test:                      │
│   ┌─────────┐           ┌─────────┐                        │
│   │Service A│           │Service B│                        │
│   │         │           │         │                        │
│   │ mock B ◄┼───────────┼─► contract                       │
│   └─────────┘           └─────────┘                        │
│                                                              │
│   • Быстрые (изолированные)                                │
│   • Понятно, кто нарушил контракт                          │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Consumer-driven contract (с Pact):**
```go
// Consumer side: определяем ожидания
func TestUserServiceConsumer(t *testing.T) {
    // Создаём mock provider
    mockProvider, err := consumer.NewV2Pact(consumer.MockHTTPProviderConfig{
        Consumer: "OrderService",
        Provider: "UserService",
    })
    require.NoError(t, err)

    // Определяем ожидаемое взаимодействие
    mockProvider.
        AddInteraction().
        Given("user 123 exists").
        UponReceiving("a request for user 123").
        WithRequest(http.MethodGet, "/users/123").
        WillRespondWith(http.StatusOK, func(b *consumer.V2ResponseBuilder) {
            b.Header("Content-Type", term("application/json", `application\/json`))
            b.JSONBody(map[string]any{
                "id":    123,
                "name":  like("John Doe"),
                "email": like("john@example.com"),
            })
        })

    // Запускаем тест с mock
    err = mockProvider.ExecuteTest(t, func(config consumer.MockServerConfig) error {
        client := NewUserClient(config.Host, config.Port)

        user, err := client.GetUser(context.Background(), 123)
        if err != nil {
            return err
        }

        assert.Equal(t, int64(123), user.ID)
        return nil
    })

    require.NoError(t, err)
}

// Provider side: верифицируем контракт
func TestUserServiceProvider(t *testing.T) {
    // Поднимаем реальный сервис
    server := startTestServer()
    defer server.Close()

    // Верифицируем против контрактов
    verifier := provider.NewVerifier()

    err := verifier.VerifyProvider(t, provider.VerifyRequest{
        Provider:        "UserService",
        ProviderBaseURL: server.URL,
        PactFiles:       []string{"pacts/orderservice-userservice.json"},
        StateHandlers: map[string]provider.StateHandler{
            "user 123 exists": func(setup bool, state provider.ProviderState) error {
                if setup {
                    // Setup test data
                    createTestUser(123, "John Doe", "john@example.com")
                }
                return nil
            },
        },
    })

    require.NoError(t, err)
}
```

**OpenAPI validation:**
```go
import "github.com/getkin/kin-openapi/openapi3filter"

func TestAPIAgainstOpenAPISpec(t *testing.T) {
    // Загружаем спецификацию
    doc, err := openapi3.NewLoader().LoadFromFile("api/openapi.yaml")
    require.NoError(t, err)

    router, err := gorillamux.NewRouter(doc)
    require.NoError(t, err)

    // Валидируем реальные запросы/ответы
    tests := []struct {
        method string
        path   string
        body   string
    }{
        {"GET", "/users/123", ""},
        {"POST", "/users", `{"name":"John","email":"john@example.com"}`},
    }

    for _, tt := range tests {
        t.Run(tt.method+" "+tt.path, func(t *testing.T) {
            // Создаём запрос
            req := httptest.NewRequest(tt.method, tt.path, strings.NewReader(tt.body))
            req.Header.Set("Content-Type", "application/json")

            // Находим route
            route, params, err := router.FindRoute(req)
            require.NoError(t, err)

            // Валидируем запрос
            reqInput := &openapi3filter.RequestValidationInput{
                Request:    req,
                PathParams: params,
                Route:      route,
            }
            err = openapi3filter.ValidateRequest(context.Background(), reqInput)
            require.NoError(t, err)

            // Выполняем запрос к реальному handler
            rec := httptest.NewRecorder()
            handler.ServeHTTP(rec, req)

            // Валидируем ответ
            respInput := &openapi3filter.ResponseValidationInput{
                RequestValidationInput: reqInput,
                Status:                 rec.Code,
                Header:                 rec.Header(),
                Body:                   io.NopCloser(rec.Body),
            }
            err = openapi3filter.ValidateResponse(context.Background(), respInput)
            require.NoError(t, err)
        })
    }
}
```

**Простой contract test без библиотек:**
```go
// Контракт определён как Go структуры
type UserContract struct {
    ID    int64  `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func TestUserAPIContract(t *testing.T) {
    server := httptest.NewServer(userHandler)
    defer server.Close()

    resp, err := http.Get(server.URL + "/users/123")
    require.NoError(t, err)
    defer resp.Body.Close()

    // Проверяем, что ответ соответствует контракту
    var user UserContract
    err = json.NewDecoder(resp.Body).Decode(&user)
    require.NoError(t, err)

    // Валидация обязательных полей
    assert.NotZero(t, user.ID)
    assert.NotEmpty(t, user.Name)
    assert.Contains(t, user.Email, "@")
}
```

**Типичные ошибки.**
1. **Тестирование implementation details** — контракт должен быть стабильным
2. **Слишком строгие контракты** — `exact("John")` вместо `like("John")`
3. **Нет версионирования контрактов** — breaking changes ломают всех consumers
4. **Забывать про state setup** — provider должен подготовить test data

**На интервью.**
Объясни разницу между E2E и contract tests. Покажи workflow: consumer пишет контракт, provider верифицирует, CI блокирует breaking changes.

*Частые follow-up вопросы:*
- Как версионировать контракты?
- Что такое consumer-driven contracts?
- Как интегрировать contract tests в CI/CD?

---

## Практика

1. **Modules:** Настрой `go.mod` с `replace` для локальной библиотеки. Добавь зависимость, обнови версию, запусти `go mod graph`.

2. **Testing:** Напиши table-driven тесты с `t.Parallel()`, `t.Cleanup()` и субтестами. Добавь тесты с testify assertions.

3. **Race/Bench/Fuzz:** Добавь `Benchmark` с `b.ReportAllocs()`. Напиши `Fuzz` тест для парсера. Запусти тесты с `-race`.

4. **Linting:** Настрой `.golangci.yml` в проекте. Исправь все предупреждения. Добавь в CI.

5. **Building:** Создай Makefile с кросс-компиляцией и инжекцией версии через `-ldflags`.

6. **Profiling:** Встрой `net/http/pprof`, сними CPU и heap профили под нагрузкой, найди hotspot в `go tool pprof`.

---

## Дополнительные материалы

- [Go Modules Reference](https://go.dev/ref/mod)
- [Testing package documentation](https://pkg.go.dev/testing)
- [Fuzzing tutorial](https://go.dev/doc/tutorial/fuzz)
- [golangci-lint configuration](https://golangci-lint.run/usage/configuration/)
- [Profiling Go Programs](https://go.dev/blog/pprof)
- [runtime/metrics package](https://pkg.go.dev/runtime/metrics)
- [testify library](https://github.com/stretchr/testify)
