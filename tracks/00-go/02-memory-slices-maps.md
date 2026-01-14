# 03 — Memory, Slices & Maps

Разбор вопросов о том, как Go управляет памятью: устройство слайсов и карт, escape analysis, аллокации, GC взаимодействие. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек Go](./README.md) · Предыдущая тема: [02-goroutines-channels](./02-goroutines-channels.md) · Следующая тема: [04-errors-context](./04-errors-context.md)

---

## Вопросы и разборы

### 1. Как устроен слайс на уровне структуры?

**Зачем спрашивают.** Слайсы — основная коллекция в Go. Непонимание их устройства приводит к багам с aliasing, неожиданным мутациям и утечкам памяти.

**Короткий ответ.** Слайс — структура из трёх полей: указатель на массив (`ptr`), длина (`len`), ёмкость (`cap`). Копирование слайса копирует заголовок, не данные.

**Детальный разбор.**

**Внутренняя структура:**
```go
// runtime/slice.go (упрощённо)
type slice struct {
    array unsafe.Pointer  // указатель на первый элемент
    len   int             // количество элементов
    cap   int             // размер underlying array от ptr до конца
}
```

**Визуализация:**
```
Slice header:          Underlying array:
┌─────────────┐        ┌───┬───┬───┬───┬───┐
│ ptr ────────┼───────►│ 1 │ 2 │ 3 │ 4 │ 5 │
│ len: 3      │        └───┴───┴───┴───┴───┘
│ cap: 5      │              ▲
└─────────────┘              │
                             │
Sub-slice [1:3]:             │
┌─────────────┐              │
│ ptr ────────┼──────────────┘
│ len: 2      │        (указывает на элемент [1])
│ cap: 4      │
└─────────────┘
```

**Ключевые свойства:**
- `len(s)` — количество доступных элементов
- `cap(s)` — максимум элементов без реаллокации
- `s[i]` — доступ к элементу (0 ≤ i < len)
- `s[lo:hi]` — создаёт новый заголовок, тот же массив
- `s[lo:hi:max]` — ограничивает cap до `max-lo`

**Пример.**
```go
func main() {
    // Создание слайса
    arr := [5]int{1, 2, 3, 4, 5}
    s := arr[1:4]  // слайс от массива

    fmt.Printf("s=%v len=%d cap=%d\n", s, len(s), cap(s))
    // s=[2 3 4] len=3 cap=4

    // Aliasing: изменение через слайс меняет массив
    s[0] = 42
    fmt.Println(arr)  // [1 42 3 4 5]

    // Sub-slice делит тот же массив
    sub := s[:2]
    sub[1] = 100
    fmt.Println(s)    // [42 100 4]
    fmt.Println(arr)  // [1 42 100 4 5]

    // Ограничение capacity с третьим индексом
    limited := s[0:2:2]  // len=2, cap=2
    fmt.Printf("limited cap=%d\n", cap(limited))  // 2
}
```

**Типичные ошибки.**
- Думать, что `s2 := s1` создаёт независимую копию — нет, оба указывают на один массив.
- Передавать слайс в функцию и ожидать, что изменения не видны снаружи.
- Забывать про aliasing при параллельной работе — race condition.

**На интервью.**
- Нарисуй структуру слайса на доске.
- Объясни, почему `append` может или не может изменить оригинальный слайс.
- Follow-up: «Как создать независимую копию слайса?» — `copy(dst, src)` или `append([]T(nil), src...)`.

---

### 2. Как работает `append` и почему меняется capacity?

**Зачем спрашивают.** `append` — самая частая операция со слайсами. Непонимание механики роста приводит к проблемам производительности и неожиданному aliasing.

**Короткий ответ.** Пока `len < cap`, append записывает в существующий массив. При `len == cap` создаётся новый массив с увеличенной ёмкостью (~2x до 1024, затем ~1.25x), данные копируются.

**Детальный разбор.**

**Алгоритм роста (Go 1.18+):**
```go
// runtime/slice.go (упрощённо)
func growslice(et *_type, old slice, cap int) slice {
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        const threshold = 256
        if old.cap < threshold {
            newcap = doublecap
        } else {
            for newcap < cap {
                newcap += (newcap + 3*threshold) / 4  // ~1.25x
            }
        }
    }
    // аллокация и копирование
}
```

**Визуализация роста:**
```
append(s, 6):

До (len=5, cap=5):        После (len=6, cap=10):
┌─────────────┐           ┌─────────────┐
│ ptr ───┐    │           │ ptr ───┐    │
│ len: 5 │    │           │ len: 6 │    │
│ cap: 5 │    │           │ cap: 10│    │
└────────┼────┘           └────────┼────┘
         ▼                         ▼
┌─┬─┬─┬─┬─┐               ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│1│2│3│4│5│               │1│2│3│4│5│6│ │ │ │ │
└─┴─┴─┴─┴─┘               └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
(старый массив                (новый массив)
 будет собран GC)
```

**Пример.**
```go
func main() {
    s := make([]int, 0, 2)
    fmt.Printf("Initial: len=%d cap=%d ptr=%p\n", len(s), cap(s), s)

    for i := 0; i < 10; i++ {
        oldPtr := fmt.Sprintf("%p", s)
        s = append(s, i)
        newPtr := fmt.Sprintf("%p", s)

        if oldPtr != newPtr {
            fmt.Printf("Realloc at i=%d: len=%d cap=%d\n", i, len(s), cap(s))
        }
    }
}

// Output:
// Initial: len=0 cap=2 ptr=0xc0000b6020
// Realloc at i=2: len=3 cap=4
// Realloc at i=4: len=5 cap=8
// Realloc at i=8: len=9 cap=16
```

**Оптимизация: преаллокация:**
```go
// Плохо: много реаллокаций
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)
}

// Хорошо: одна аллокация
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)
}

// Или если размер известен точно:
s := make([]int, 10000)
for i := 0; i < 10000; i++ {
    s[i] = i
}
```

**Типичные ошибки.**
- Не возвращать результат append: `append(s, x)` без присваивания.
- Полагаться на то, что append не изменит оригинальный слайс — может, если cap позволяет.
- Преаллоцировать слишком много — тратишь память.

**На интервью.**
- Объясни, когда append создаёт новый массив.
- Покажи разницу в производительности с преаллокацией.
- Follow-up: «Почему коэффициент роста меняется?» — амортизация O(1) для append, но экономия памяти для больших слайсов.

---

### 3. Что такое nil-слайс и пустой слайс? В чём разница?

**Зачем спрашивают.** Путаница между nil и empty slice — частый источник багов, особенно при сериализации в JSON.

**Короткий ответ.** `var s []T` — nil-слайс (ptr=nil, len=0, cap=0). `s := []T{}` или `make([]T, 0)` — пустой слайс (ptr≠nil, len=0, cap=0). Обе версии функционально эквивалентны для большинства операций.

**Детальный разбор.**

**Сравнение:**
| Свойство | nil slice | empty slice |
|----------|-----------|-------------|
| `s == nil` | true | false |
| `len(s)` | 0 | 0 |
| `cap(s)` | 0 | 0 |
| `append(s, x)` | работает | работает |
| `for range s` | 0 итераций | 0 итераций |
| `json.Marshal` | `null` | `[]` |

**Пример.**
```go
func main() {
    var nilSlice []int
    emptySlice := []int{}
    madeSlice := make([]int, 0)

    fmt.Printf("nil:   %v, isNil=%t, len=%d\n", nilSlice, nilSlice == nil, len(nilSlice))
    fmt.Printf("empty: %v, isNil=%t, len=%d\n", emptySlice, emptySlice == nil, len(emptySlice))
    fmt.Printf("made:  %v, isNil=%t, len=%d\n", madeSlice, madeSlice == nil, len(madeSlice))

    // JSON разница
    j1, _ := json.Marshal(nilSlice)
    j2, _ := json.Marshal(emptySlice)
    fmt.Printf("nil JSON: %s\n", j1)    // null
    fmt.Printf("empty JSON: %s\n", j2)  // []

    // Обе работают с append
    nilSlice = append(nilSlice, 1)
    emptySlice = append(emptySlice, 1)
    fmt.Printf("after append: nil=%v, empty=%v\n", nilSlice, emptySlice)
}
```

**Идиоматичный Go:**
```go
// Предпочтительно: nil slice как zero value
func GetUsers() []User {
    var users []User  // nil slice
    // ... заполнение
    return users  // вернёт nil если пусто
}

// Когда нужен именно пустой слайс (для JSON):
func GetUsersForAPI() []User {
    users := []User{}  // или make([]User, 0)
    // ... заполнение
    return users  // вернёт [] в JSON, не null
}
```

**Типичные ошибки.**
- Проверять `len(s) == 0` И `s == nil` отдельно — обычно достаточно `len(s) == 0`.
- Возвращать `[]T{}` везде «для безопасности» — лишние аллокации.
- Не учитывать разницу в JSON сериализации.

**На интервью.**
- Объясни разницу в представлении в памяти.
- Покажи, когда разница важна (JSON, reflect).
- Follow-up: «Какой вариант предпочтительнее?» — nil slice как zero value, empty slice для JSON API.

---

### 4. Как избежать aliasing и утечек памяти со слайсами?

**Зачем спрашивают.** Aliasing и memory leaks — типичные проблемы при работе со слайсами. Интервьюер проверяет понимание ownership и GC.

**Короткий ответ.** Срез большого массива удерживает весь массив в памяти. Решение: копировать нужные данные в новый слайс. Для избежания aliasing: использовать `copy` или full slice expression `s[lo:hi:hi]`.

**Детальный разбор.**

**Проблема удержания памяти:**
```
Большой массив (1MB):
┌──────────────────────────────────────────────────┐
│ data data data data data data data data data ... │
└──────────────────────────────────────────────────┘
        ▲
        │
Маленький слайс (10 bytes):
┌─────────────┐
│ ptr ────────┘
│ len: 10     │
│ cap: 1MB    │  ← GC не освободит 1MB пока слайс жив!
└─────────────┘
```

**Решение — копирование:**
```go
func extractHeader(data []byte) []byte {
    // Плохо: удерживает весь data в памяти
    // return data[:10]

    // Хорошо: независимая копия
    header := make([]byte, 10)
    copy(header, data[:10])
    return header
}
```

**Проблема aliasing:**
```go
func dangerous(s []int) {
    s[0] = 999  // изменяет оригинальный слайс!
}

func safe(s []int) {
    local := make([]int, len(s))
    copy(local, s)
    local[0] = 999  // не влияет на оригинал
}
```

**Full slice expression для ограничения cap:**
```go
func main() {
    original := []int{1, 2, 3, 4, 5}

    // Обычный срез: cap позволяет append затронуть original
    sub1 := original[1:3]
    sub1 = append(sub1, 100)
    fmt.Println(original)  // [1 2 3 100 5] — затронут!

    original = []int{1, 2, 3, 4, 5}  // reset

    // Full slice expression: cap ограничен
    sub2 := original[1:3:3]  // len=2, cap=2
    sub2 = append(sub2, 200)
    fmt.Println(original)  // [1 2 3 4 5] — не затронут
    fmt.Println(sub2)      // [2 3 200] — новый массив
}
```

**Пример.**
```go
func processLargeFile(filename string) ([]byte, error) {
    data, err := os.ReadFile(filename)  // 100MB файл
    if err != nil {
        return nil, err
    }

    // Нужны только первые 100 байт
    // Плохо:
    // return data[:100], nil  // удерживает 100MB

    // Хорошо:
    result := make([]byte, 100)
    copy(result, data[:100])
    return result, nil  // data будет собран GC
}

// Паттерн: "clone" функция
func clone[T any](s []T) []T {
    if s == nil {
        return nil
    }
    c := make([]T, len(s))
    copy(c, s)
    return c
}
```

**Типичные ошибки.**
- Возвращать срез большого буфера из функции.
- Хранить срез в долгоживущей структуре без копирования.
- Не понимать, что `append` может изменить оригинальный массив.

**На интервью.**
- Объясни, почему срез удерживает весь underlying array.
- Покажи full slice expression `s[lo:hi:max]`.
- Follow-up: «Как slices.Clone() решает эту проблему?» — создаёт независимую копию (Go 1.21+).

---

### 5. Как работает map и как Go хранит данные?

**Зачем спрашивают.** Maps — вторая по частоте коллекция. Понимание устройства помогает писать эффективный код и избегать багов.

**Короткий ответ.** Map — хеш-таблица с bucket'ами по 8 элементов. Каждый bucket хранит tophash (первые 8 бит хеша) и пары ключ-значение. При заполнении происходит рехеширование.

**Детальный разбор.**

**Внутренняя структура:**
```go
// runtime/map.go (упрощённо)
type hmap struct {
    count     int            // количество элементов
    B         uint8          // log2(количество buckets)
    buckets   unsafe.Pointer // массив из 2^B bucket'ов
    oldbuckets unsafe.Pointer // для incremental rehashing
    // ...
}

type bmap struct {  // bucket
    tophash [8]uint8           // первые 8 бит хеша каждого ключа
    // за tophash следуют:
    // keys   [8]keytype
    // values [8]valuetype
    // overflow *bmap
}
```

**Визуализация:**
```
hmap:
┌──────────────┐
│ count: 5     │
│ B: 2         │     buckets (2^2 = 4):
│ buckets ─────┼───► ┌────────────────────────────┐
└──────────────┘     │ bucket 0                   │
                     │ tophash: [h1,h2,0,0,0,0,0,0]│
                     │ keys:    [k1,k2,_,_,_,_,_,_]│
                     │ values:  [v1,v2,_,_,_,_,_,_]│
                     ├────────────────────────────┤
                     │ bucket 1                   │
                     │ ...                        │
                     ├────────────────────────────┤
                     │ bucket 2                   │
                     │ ...                        │
                     ├────────────────────────────┤
                     │ bucket 3                   │
                     │ ...                        │
                     └────────────────────────────┘
```

**Load factor и рехеширование:**
- Load factor ~6.5 (в среднем 6.5 элементов на bucket)
- Рехеширование происходит инкрементально (не STW)
- Новый размер = 2x текущего

**Пример.**
```go
func main() {
    m := make(map[string]int)

    // Вставка
    m["one"] = 1
    m["two"] = 2

    // Доступ
    v, ok := m["one"]
    fmt.Printf("one=%d, exists=%t\n", v, ok)

    // Несуществующий ключ
    v, ok = m["three"]
    fmt.Printf("three=%d, exists=%t\n", v, ok)  // 0, false

    // Удаление
    delete(m, "one")

    // Итерация (порядок случаен!)
    for k, v := range m {
        fmt.Printf("%s: %d\n", k, v)
    }

    // Преаллокация для известного размера
    big := make(map[string]int, 10000)
    _ = big
}
```

**Типичные ошибки.**
- Итерировать и модифицировать одновременно — undefined behavior.
- Полагаться на порядок ключей — он рандомизирован.
- Использовать map без инициализации — panic при записи.
- Брать адрес значения в map: `&m[key]` — не компилируется (значение может переехать при рехеше).

**На интервью.**
- Объясни bucket structure и tophash оптимизацию.
- Расскажи про incremental rehashing.
- Follow-up: «Почему порядок итерации случаен?» — специально рандомизирован, чтобы не полагались на порядок.

---

### 6. Как сделать map потокобезопасным?

**Зачем спрашивают.** Concurrent map access — частая причина data race. Интервьюер проверяет знание способов синхронизации.

**Короткий ответ.** Maps не потокобезопасны. Варианты: `sync.Mutex`, `sync.RWMutex`, `sync.Map`, шардирование.

**Детальный разбор.**

**Проблема:**
```go
func bad() {
    m := make(map[int]int)
    go func() { m[1] = 1 }()  // write
    go func() { _ = m[1] }()  // read
    // fatal error: concurrent map read and map write
}
```

**Решение 1: sync.RWMutex**
```go
type SafeMap[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func (sm *SafeMap[K, V]) Get(key K) (V, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    v, ok := sm.m[key]
    return v, ok
}

func (sm *SafeMap[K, V]) Set(key K, value V) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SafeMap[K, V]) Delete(key K) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    delete(sm.m, key)
}
```

**Решение 2: sync.Map**
```go
func useSyncMap() {
    var m sync.Map

    // Store
    m.Store("key", "value")

    // Load
    if v, ok := m.Load("key"); ok {
        fmt.Println(v.(string))
    }

    // LoadOrStore — атомарный get-or-set
    actual, loaded := m.LoadOrStore("key2", "default")
    fmt.Printf("value=%v, wasLoaded=%t\n", actual, loaded)

    // Range — итерация
    m.Range(func(k, v any) bool {
        fmt.Printf("%v: %v\n", k, v)
        return true  // continue
    })

    // Delete
    m.Delete("key")
}
```

**Решение 3: Шардирование**
```go
type ShardedMap[K comparable, V any] struct {
    shards []*shard[K, V]
    hash   func(K) uint64
}

type shard[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func (sm *ShardedMap[K, V]) getShard(key K) *shard[K, V] {
    h := sm.hash(key)
    return sm.shards[h%uint64(len(sm.shards))]
}

func (sm *ShardedMap[K, V]) Get(key K) (V, bool) {
    shard := sm.getShard(key)
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    v, ok := shard.m[key]
    return v, ok
}
```

**Когда что использовать:**
| Сценарий | Решение |
|----------|---------|
| Простой случай, мало contention | `sync.RWMutex` |
| Write-once, read-many (кеш) | `sync.Map` |
| Много ключей, set рандомный | `sync.Map` |
| Высокий contention | Шардирование |

**Пример.**
```go
func main() {
    // RWMutex для типизированного доступа
    cache := struct {
        sync.RWMutex
        data map[string]int
    }{data: make(map[string]int)}

    // Write
    cache.Lock()
    cache.data["key"] = 42
    cache.Unlock()

    // Read
    cache.RLock()
    v := cache.data["key"]
    cache.RUnlock()

    fmt.Println(v)
}
```

**Типичные ошибки.**
- Использовать sync.Map везде — для типизированных данных RWMutex проще и часто быстрее.
- Забывать Unlock — deadlock.
- Lock на всю операцию вместо минимального участка.

**На интервью.**
- Объясни trade-offs между RWMutex, sync.Map и шардированием.
- Покажи, когда sync.Map предпочтительнее.
- Follow-up: «Как sync.Map работает внутри?» — read-only map + dirty map с promotion.

---

### 7. Что делает escape analysis и как увидеть результат?

**Зачем спрашивают.** Escape analysis определяет, где выделяется память: стек или heap. Понимание помогает писать код с меньшим количеством аллокаций.

**Короткий ответ.** Компилятор анализирует, «убегает» ли переменная за пределы функции. Если да — аллоцируется в heap, иначе — на стеке. Флаг `-gcflags=-m` показывает решения компилятора.

**Детальный разбор.**

**Когда переменная escapes to heap:**
1. Указатель возвращается из функции
2. Указатель присваивается глобальной переменной
3. Указатель сохраняется в структуре, которая escapes
4. Размер слишком велик для стека
5. Размер неизвестен в compile time
6. Closure захватывает переменную

**Stack vs Heap:**
| Stack | Heap |
|-------|------|
| Автоматическое освобождение | GC |
| Быстрая аллокация (bump pointer) | Сложная аллокация |
| Локальность данных | Фрагментация |
| Ограниченный размер | Неограниченный |

**Пример.**
```go
//go:noinline
func stackAlloc() int {
    x := 42  // не escapes — стек
    return x
}

//go:noinline
func heapAlloc() *int {
    x := 42  // escapes — heap
    return &x
}

//go:noinline
func sliceEscape() []int {
    s := make([]int, 10)  // escapes — heap (возвращается)
    return s
}

//go:noinline
func sliceNoEscape() {
    s := make([]int, 10)  // не escapes — стек
    _ = s
}

//go:noinline
func bigSlice() {
    s := make([]int, 1<<20)  // escapes — слишком большой для стека
    _ = s
}
```

**Анализ:**
```bash
$ go build -gcflags='-m -m' main.go 2>&1 | grep -E '(escape|moved)'

./main.go:10:2: x escapes to heap:
./main.go:10:2:   flow: ~r0 = &x:
./main.go:10:2:     from &x (address-of) at ./main.go:11:9
./main.go:10:2:     from return &x (return) at ./main.go:11:2
./main.go:10:2: moved to heap: x
./main.go:15:11: make([]int, 10) escapes to heap
./main.go:20:11: make([]int, 10) does not escape
./main.go:25:11: make([]int, 1048576) escapes to heap
```

**Оптимизация: избегание escape:**
```go
// Плохо: slice escapes
func processData(data []byte) *Result {
    result := &Result{}  // escapes
    // ...
    return result
}

// Лучше: caller предоставляет память
func processDataInto(data []byte, result *Result) {
    // result не escapes (передан снаружи)
    // ...
}

// Или: возврат по значению (для небольших структур)
func processDataValue(data []byte) Result {
    var result Result  // не escapes
    // ...
    return result  // копируется, но для небольших struct это OK
}
```

**Типичные ошибки.**
- Оптимизировать escape до профилирования — premature optimization.
- Думать, что `new(T)` всегда heap — зависит от escape analysis.
- Возвращать указатели на большие структуры «для эффективности» — может быть медленнее из-за heap + GC.

**На интервью.**
- Покажи `-gcflags=-m` для анализа escape.
- Объясни, почему stack быстрее heap.
- Follow-up: «Как уменьшить heap allocations?» — sync.Pool, преаллокация, возврат по значению для маленьких struct.

---

### 8. Как работает sync.Pool для переиспользования памяти?

**Зачем спрашивают.** sync.Pool — стандартный способ уменьшить давление на GC. Интервьюер проверяет знание оптимизации аллокаций.

**Короткий ответ.** sync.Pool — кеш объектов, которые могут быть переиспользованы. Get возвращает объект или создаёт новый через New. Put возвращает объект в pool. Pool очищается при GC.

**Детальный разбор.**

**Как работает:**
```
┌─────────────────────────────────────────────┐
│                sync.Pool                     │
├─────────────────────────────────────────────┤
│  per-P private cache (lock-free)            │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐           │
│  │ P0  │ │ P1  │ │ P2  │ │ P3  │           │
│  │[obj]│ │[obj]│ │[obj]│ │[obj]│           │
│  └─────┘ └─────┘ └─────┘ └─────┘           │
├─────────────────────────────────────────────┤
│  shared pool (needs lock)                   │
│  [obj] [obj] [obj] ...                      │
└─────────────────────────────────────────────┘

Get():
1. Проверить private cache текущего P
2. Если пусто — проверить shared pool
3. Если пусто — вызвать New()

Put():
1. Положить в private cache текущего P
2. Если переполнен — в shared pool
```

**Пример.**
```go
var bufferPool = sync.Pool{
    New: func() any {
        return make([]byte, 0, 4096)
    },
}

func processRequest(data []byte) []byte {
    // Получить буфер из pool
    buf := bufferPool.Get().([]byte)
    buf = buf[:0]  // сбросить длину, сохранить capacity

    // Использовать буфер
    buf = append(buf, data...)
    buf = append(buf, '\n')

    // Важно: скопировать результат перед возвратом в pool!
    result := make([]byte, len(buf))
    copy(result, buf)

    // Вернуть буфер в pool
    bufferPool.Put(buf)

    return result
}

// Бенчмарк
func BenchmarkWithPool(b *testing.B) {
    data := []byte("hello world")
    for i := 0; i < b.N; i++ {
        _ = processRequest(data)
    }
}

func BenchmarkWithoutPool(b *testing.B) {
    data := []byte("hello world")
    for i := 0; i < b.N; i++ {
        buf := make([]byte, 0, 4096)
        buf = append(buf, data...)
        buf = append(buf, '\n')
        _ = buf
    }
}
```

**Типичные use cases:**
- Буферы для encoding/decoding
- Временные структуры для parsing
- Объекты HTTP request/response
- bytes.Buffer

**Типичные ошибки.**
- Использовать Pool для всего — overhead может превысить выгоду.
- Не сбрасывать состояние объекта перед Put — грязные данные.
- Хранить указатели на pooled объекты — могут быть собраны GC.
- Pool для маленьких объектов — overhead больше выгоды.

**На интервью.**
- Объясни lifecycle объектов в Pool.
- Покажи, когда Pool полезен (high-throughput, большие объекты).
- Follow-up: «Почему Pool очищается при GC?» — предотвращает неограниченный рост кеша.

---

### 9. Как сортировать слайсы в Go?

**Зачем спрашивают.** Базовый вопрос, но показывает знание stdlib и generics.

**Короткий ответ.** `sort.Slice` для пользовательской сортировки, `slices.Sort` (Go 1.21+) для типов с `cmp.Ordered`, `slices.SortFunc` для пользовательского компаратора.

**Детальный разбор.**

**Варианты API:**
```go
// sort package (до Go 1.21)
sort.Ints([]int{3, 1, 2})
sort.Strings([]string{"c", "a", "b"})
sort.Slice(s, func(i, j int) bool { return s[i] < s[j] })
sort.SliceStable(s, less)  // сохраняет относительный порядок равных

// slices package (Go 1.21+)
slices.Sort([]int{3, 1, 2})
slices.SortFunc(users, func(a, b User) int {
    return cmp.Compare(a.Name, b.Name)
})
slices.SortStableFunc(users, cmp)
```

**Пример.**
```go
type User struct {
    Name string
    Age  int
}

func main() {
    users := []User{
        {"Charlie", 30},
        {"Alice", 25},
        {"Bob", 25},
    }

    // По имени
    sort.Slice(users, func(i, j int) bool {
        return users[i].Name < users[j].Name
    })
    fmt.Println(users)  // Alice, Bob, Charlie

    // По возрасту, затем по имени (stable)
    sort.SliceStable(users, func(i, j int) bool {
        if users[i].Age != users[j].Age {
            return users[i].Age < users[j].Age
        }
        return users[i].Name < users[j].Name
    })

    // Go 1.21+ с slices
    slices.SortFunc(users, func(a, b User) int {
        if n := cmp.Compare(a.Age, b.Age); n != 0 {
            return n
        }
        return cmp.Compare(a.Name, b.Name)
    })
}
```

**Типичные ошибки.**
- Путать `sort.Slice` и `sort.SliceStable`.
- Забывать, что sort изменяет слайс in-place.
- Неконсистентный компаратор (не транзитивный) — undefined behavior.

**На интервью.**
- Покажи оба варианта: sort.Slice и slices.SortFunc.
- Объясни разницу Stable/non-Stable.
- Follow-up: «Какой алгоритм использует sort?» — pattern-defeating quicksort (pdqsort) в Go 1.19+.

---

## Практика

1. **Slice internals** — напиши функцию, которая выводит ptr, len, cap слайса через unsafe. Покажи, как они меняются при append и срезах.

2. **Memory leak demo** — создай программу с утечкой памяти через срез большого массива. Исправь копированием.

3. **Escape analysis** — напиши несколько функций и проанализируй escape через `-gcflags=-m`. Оптимизируй, чтобы уменьшить heap allocations.

4. **sync.Pool benchmark** — реализуй HTTP handler с bufferPool. Сравни производительность с/без Pool.

5. **Concurrent map** — реализуй шардированную map и сравни с sync.Map и RWMutex на разных нагрузках.

6. **Custom sort** — реализуй сортировку структур по нескольким полям с поддержкой направления (asc/desc).

---

## Дополнительные материалы

- [Go blog: Slices usage and internals](https://go.dev/blog/slices-intro) — внутренее устройство слайсов
- [Go blog: Maps in action](https://go.dev/blog/maps) — работа с картами
- [Go wiki: Memory Model](https://go.dev/ref/mem) — гарантии видимости
- [slices package](https://pkg.go.dev/slices) — generic функции для слайсов (Go 1.21+)
- [maps package](https://pkg.go.dev/maps) — generic функции для карт (Go 1.21+)
- [sync.Pool design](https://github.com/golang/go/blob/master/src/sync/pool.go) — исходники Pool
