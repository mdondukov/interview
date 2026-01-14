# 13 — Reflection & Code Generation

Разбор вопросов о рефлексии и кодогенерации в Go: reflect, struct tags, go generate, ast, wire.

**Навигация:** [← 12-security-crypto](./12-security-crypto.md) | [Трек Go](./README.md)

---

## Вопросы и разборы

### 1. Когда использовать reflect и когда избегать?

**Зачем спрашивают.**
Рефлексия — мощный, но опасный инструмент. Интервьюер проверяет понимание trade-offs и альтернатив.

**Короткий ответ.**
Reflect нужен для generic библиотек (JSON, ORM, validation). Избегай в application code — медленнее, не type-safe, сложнее отлаживать. Предпочитай generics, кодогенерацию, интерфейсы.

**Детальный разбор.**

**Когда использовать:**
```
┌─────────────────────────────────────────────────────────────┐
│ ДА (library code):                                           │
│ • Сериализация (encoding/json, encoding/xml)                │
│ • ORM (GORM, sqlx scanning)                                 │
│ • Validation (go-playground/validator)                      │
│ • Dependency injection (fx, dig)                            │
│ • Testing (deep equality, mocking)                          │
│ • Generic utilities (spew, pretty print)                    │
├─────────────────────────────────────────────────────────────┤
│ НЕТ (application code):                                      │
│ • Бизнес-логика                                             │
│ • Обычные функции                                           │
│ • Когда можно использовать generics                         │
│ • Когда можно использовать интерфейсы                       │
│ • Когда можно использовать кодогенерацию                    │
└─────────────────────────────────────────────────────────────┘
```

**Trade-offs:**
```
┌─────────────────┬──────────────────────────────────────────┐
│ Плюсы           │ Минусы                                   │
├─────────────────┼──────────────────────────────────────────┤
│ Generic code    │ Медленнее (10-100x)                      │
│ Runtime flex    │ Нет compile-time checks                  │
│ Меньше кода     │ Сложнее отлаживать                       │
│                 │ Паники вместо compile errors             │
│                 │ Хуже IDE support                         │
└─────────────────┴──────────────────────────────────────────┘
```

**Пример.**

**Плохое использование (можно без reflect):**
```go
// ПЛОХО: reflect для известных типов
func ProcessValue(v interface{}) {
    rv := reflect.ValueOf(v)
    switch rv.Kind() {
    case reflect.String:
        processString(rv.String())
    case reflect.Int:
        processInt(rv.Int())
    }
}

// ХОРОШО: type switch
func ProcessValue(v interface{}) {
    switch x := v.(type) {
    case string:
        processString(x)
    case int:
        processInt(x)
    }
}

// ЕЩЁ ЛУЧШЕ: generics (Go 1.18+)
func ProcessValue[T string | int](v T) {
    // type-safe обработка
}
```

**Хорошее использование (библиотека):**
```go
// Пример: generic struct validator
func Validate(v interface{}) error {
    rv := reflect.ValueOf(v)
    if rv.Kind() == reflect.Ptr {
        rv = rv.Elem()
    }

    if rv.Kind() != reflect.Struct {
        return errors.New("expected struct")
    }

    rt := rv.Type()
    for i := 0; i < rv.NumField(); i++ {
        field := rt.Field(i)
        value := rv.Field(i)

        // Читаем validation tags
        tag := field.Tag.Get("validate")
        if tag == "" {
            continue
        }

        if err := validateField(field.Name, value, tag); err != nil {
            return err
        }
    }

    return nil
}
```

**Альтернативы reflect:**

```go
// 1. Generics
func Map[T, R any](items []T, fn func(T) R) []R {
    result := make([]R, len(items))
    for i, item := range items {
        result[i] = fn(item)
    }
    return result
}

// 2. Интерфейсы
type Stringer interface {
    String() string
}

func PrintAll(items []Stringer) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

// 3. Кодогенерация
//go:generate stringer -type=Status
type Status int
const (
    StatusPending Status = iota
    StatusActive
    StatusDone
)
```

**Типичные ошибки.**
1. **Reflect в горячем пути** — performance bottleneck
2. **Паники вместо errors** — reflect паникует на invalid operations
3. **Reflect вместо interface{}** — type assertion быстрее
4. **Не кешировать Type** — повторные вызовы TypeOf дорогие

**На интервью.**
Приведи примеры, когда reflect оправдан. Покажи альтернативу через generics. Объясни performance impact.

*Частые follow-up вопросы:*
- Насколько медленнее reflect?
- Как кешировать reflect metadata?
- Когда generics не заменяют reflect?

---

### 2. Как работает reflect.Type и reflect.Value?

**Зачем спрашивают.**
Понимание Type/Value — основа работы с reflect. Интервьюер проверяет знание API и особенностей.

**Короткий ответ.**
`reflect.Type` — описание типа (имя, поля, методы). `reflect.Value` — конкретное значение. TypeOf/ValueOf создают из interface{}. Для изменения нужен указатель и Elem().

**Детальный разбор.**

**Type vs Value:**
```
┌─────────────────────────────────────────────────────────────┐
│ reflect.Type                    reflect.Value                │
├─────────────────────────────────────────────────────────────┤
│ Описывает тип                   Хранит значение              │
│ Immutable                       Может быть settable          │
│ reflect.TypeOf(v)               reflect.ValueOf(v)           │
│                                                              │
│ .Name() string                  .Interface() interface{}     │
│ .Kind() Kind                    .Kind() Kind                 │
│ .NumField() int                 .NumField() int              │
│ .Field(i) StructField           .Field(i) Value              │
│ .NumMethod() int                .NumMethod() int             │
│ .Method(i) Method               .Method(i) Value             │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Базовые операции:**
```go
import "reflect"

type User struct {
    ID   int    `json:"id" db:"user_id"`
    Name string `json:"name" db:"user_name"`
    Age  int    `json:"age,omitempty"`
}

func ExploreType() {
    var u User

    t := reflect.TypeOf(u)
    fmt.Println("Type:", t.Name())       // "User"
    fmt.Println("Kind:", t.Kind())       // struct
    fmt.Println("NumField:", t.NumField()) // 3

    // Итерация по полям
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("Field: %s, Type: %s, Tag: %s\n",
            field.Name, field.Type, field.Tag)
    }
}

func ExploreValue() {
    u := User{ID: 1, Name: "Alice", Age: 30}

    v := reflect.ValueOf(u)
    fmt.Println("Kind:", v.Kind())       // struct
    fmt.Println("NumField:", v.NumField()) // 3

    // Получение значений полей
    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        fmt.Printf("Field %d: %v\n", i, field.Interface())
    }
}
```

**Изменение значений:**
```go
func ModifyValue() {
    u := User{ID: 1, Name: "Alice", Age: 30}

    // Для изменения нужен указатель
    v := reflect.ValueOf(&u).Elem()

    // Проверяем, можно ли изменять
    if v.CanSet() {
        nameField := v.FieldByName("Name")
        if nameField.CanSet() {
            nameField.SetString("Bob")
        }
    }

    fmt.Println(u.Name) // "Bob"
}

// Почему нужен указатель?
func WhyPointer() {
    x := 10

    // ValueOf создаёт копию
    v := reflect.ValueOf(x)
    // v.SetInt(20) // PANIC: cannot set

    // Через указатель
    v = reflect.ValueOf(&x).Elem()
    v.SetInt(20)
    fmt.Println(x) // 20
}
```

**Работа со слайсами и мапами:**
```go
func SliceReflection() {
    nums := []int{1, 2, 3}

    v := reflect.ValueOf(&nums).Elem()

    // Добавление элемента
    v.Set(reflect.Append(v, reflect.ValueOf(4)))

    // Изменение элемента
    v.Index(0).SetInt(10)

    fmt.Println(nums) // [10 2 3 4]
}

func MapReflection() {
    m := map[string]int{"a": 1, "b": 2}

    v := reflect.ValueOf(m)

    // Итерация
    for _, key := range v.MapKeys() {
        value := v.MapIndex(key)
        fmt.Printf("%s: %d\n", key.String(), value.Int())
    }

    // Добавление (map уже указатель)
    v.SetMapIndex(reflect.ValueOf("c"), reflect.ValueOf(3))
}
```

**Вызов методов:**
```go
func (u User) Greet() string {
    return fmt.Sprintf("Hello, %s!", u.Name)
}

func CallMethod() {
    u := User{Name: "Alice"}

    v := reflect.ValueOf(u)

    // Получаем метод
    method := v.MethodByName("Greet")
    if !method.IsValid() {
        fmt.Println("Method not found")
        return
    }

    // Вызываем
    results := method.Call(nil)
    fmt.Println(results[0].String()) // "Hello, Alice!"
}

// С аргументами
func (u User) SetAge(age int) {
    // ...
}

func CallMethodWithArgs() {
    u := User{Name: "Alice"}
    v := reflect.ValueOf(&u)

    method := v.MethodByName("SetAge")
    args := []reflect.Value{reflect.ValueOf(30)}
    method.Call(args)
}
```

**Создание новых значений:**
```go
func CreateNew() {
    // Создание struct
    t := reflect.TypeOf(User{})
    v := reflect.New(t) // возвращает *User
    user := v.Interface().(*User)
    user.Name = "Bob"

    // Создание slice
    sliceType := reflect.SliceOf(reflect.TypeOf(0))
    slice := reflect.MakeSlice(sliceType, 0, 10)
    slice = reflect.Append(slice, reflect.ValueOf(1))

    // Создание map
    mapType := reflect.MapOf(reflect.TypeOf(""), reflect.TypeOf(0))
    m := reflect.MakeMap(mapType)
    m.SetMapIndex(reflect.ValueOf("key"), reflect.ValueOf(42))
}
```

**Типичные ошибки.**
1. **SetXxx без CanSet** — паника
2. **Не использовать Elem()** — работа с указателем, не значением
3. **Interface() на unexported** — паника
4. **Нет проверки Kind** — паника на неверных операциях

**На интервью.**
Покажи разницу Type vs Value. Объясни, зачем Elem(). Продемонстрируй изменение значения через reflect.

*Частые follow-up вопросы:*
- Почему нужен указатель для Set?
- Как получить unexported поля?
- Как вызвать метод через reflect?

---

### 3. Что такое go generate и как писать генераторы?

**Зачем спрашивают.**
go generate — альтернатива runtime reflect. Интервьюер проверяет знание инструмента и практику кодогенерации.

**Короткий ответ.**
`go generate` запускает команды из комментариев `//go:generate`. Генераторы создают Go код перед компиляцией. Примеры: stringer, mockgen, protoc.

**Детальный разбор.**

**Как работает:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Разработчик пишет: //go:generate stringer -type=Status   │
│                                                              │
│ 2. Запуск: go generate ./...                                │
│                                                              │
│ 3. go находит комментарии //go:generate                     │
│                                                              │
│ 4. Выполняет команды (stringer, mockgen, etc.)             │
│                                                              │
│ 5. Генераторы создают файлы (status_string.go)             │
│                                                              │
│ 6. Сгенерированный код компилируется вместе с остальным     │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Использование stringer:**
```go
// status.go

//go:generate stringer -type=Status
type Status int

const (
    StatusPending Status = iota
    StatusActive
    StatusCompleted
    StatusFailed
)

// После go generate создаётся status_string.go:
// func (s Status) String() string { ... }

func main() {
    fmt.Println(StatusActive) // "StatusActive"
}
```

**mockgen для тестов:**
```go
// repository.go

//go:generate mockgen -source=repository.go -destination=mocks/repository_mock.go -package=mocks

type UserRepository interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

// После go generate: mocks/repository_mock.go с MockUserRepository
```

**Написание своего генератора:**
```go
// cmd/enumgen/main.go

package main

import (
    "bytes"
    "go/ast"
    "go/parser"
    "go/token"
    "os"
    "strings"
    "text/template"
)

const tmpl = `// Code generated by enumgen. DO NOT EDIT.
package {{.Package}}

var _{{.TypeName}}Names = map[{{.TypeName}}]string{
{{range .Values}}    {{$.TypeName}}{{.}}: "{{.}}",
{{end}}}

func (e {{.TypeName}}) String() string {
    if name, ok := _{{.TypeName}}Names[e]; ok {
        return name
    }
    return "Unknown"
}

func Parse{{.TypeName}}(s string) ({{.TypeName}}, bool) {
    for val, name := range _{{.TypeName}}Names {
        if name == s {
            return val, true
        }
    }
    return 0, false
}
`

type TemplateData struct {
    Package  string
    TypeName string
    Values   []string
}

func main() {
    typeName := os.Args[1]
    filename := os.Args[2]

    fset := token.NewFileSet()
    file, err := parser.ParseFile(fset, filename, nil, parser.ParseComments)
    if err != nil {
        panic(err)
    }

    var values []string

    ast.Inspect(file, func(n ast.Node) bool {
        vs, ok := n.(*ast.ValueSpec)
        if !ok {
            return true
        }

        for _, name := range vs.Names {
            values = append(values, name.Name)
        }
        return true
    })

    data := TemplateData{
        Package:  file.Name.Name,
        TypeName: typeName,
        Values:   values,
    }

    t := template.Must(template.New("enum").Parse(tmpl))

    var buf bytes.Buffer
    if err := t.Execute(&buf, data); err != nil {
        panic(err)
    }

    outputFile := strings.ToLower(typeName) + "_enum.go"
    os.WriteFile(outputFile, buf.Bytes(), 0644)
}

// Использование:
//go:generate go run ./cmd/enumgen Status status.go
```

**Генерация с go/ast:**
```go
// Более правильный подход - использовать go/ast для генерации

import (
    "go/ast"
    "go/format"
    "go/token"
)

func generateCode() {
    fset := token.NewFileSet()

    // Создаём AST программно
    file := &ast.File{
        Name: ast.NewIdent("main"),
        Decls: []ast.Decl{
            &ast.FuncDecl{
                Name: ast.NewIdent("Hello"),
                Type: &ast.FuncType{
                    Params:  &ast.FieldList{},
                    Results: nil,
                },
                Body: &ast.BlockStmt{
                    List: []ast.Stmt{
                        // ...
                    },
                },
            },
        },
    }

    // Форматируем и выводим
    format.Node(os.Stdout, fset, file)
}
```

**Best practices для go:generate:**
```go
// 1. Указывать в начале файла
//go:generate ...

// 2. Генерировать в отдельные файлы с суффиксом
// user_generated.go, user_mock.go

// 3. Добавлять header "DO NOT EDIT"
// Code generated by XXX. DO NOT EDIT.

// 4. Коммитить сгенерированный код
// (чтобы go install работал без generate)

// 5. Makefile target
// generate:
//     go generate ./...

// 6. CI проверка
// go generate ./... && git diff --exit-code
```

**Типичные ошибки.**
1. **Не коммитить generated** — сломается go install
2. **Редактировать generated** — перезапишется
3. **Зависимость от PATH** — использовать go run
4. **Нет проверки в CI** — generated устаревает

**На интервью.**
Покажи пример go:generate директивы. Объясни, когда генерация лучше reflect. Расскажи про stringer, mockgen.

*Частые follow-up вопросы:*
- Как отлаживать генераторы?
- Когда коммитить generated код?
- Как написать свой генератор?

---

### 4. Как работают struct tags и как их парсить?

**Зачем спрашивают.**
Struct tags используются везде: JSON, DB, validation. Интервьюер проверяет понимание формата и парсинга.

**Короткий ответ.**
Struct tags — строковые метаданные полей. Формат: `key:"value" key2:"value2"`. Парсинг через `reflect.StructTag.Get()`. Используются для сериализации, ORM, validation.

**Детальный разбор.**

**Формат тегов:**
```
┌─────────────────────────────────────────────────────────────┐
│ type User struct {                                           │
│     ID   int    `json:"id" db:"user_id" validate:"required"`│
│     Name string `json:"name,omitempty" db:"name"`           │
│ }                                                            │
│                                                              │
│ Формат: `key:"value" key2:"value2,option"`                  │
│                                                              │
│ Правила:                                                     │
│ • key:value без пробелов вокруг :                           │
│ • Значение в двойных кавычках                               │
│ • Пробел между key:value парами                             │
│ • Опции через запятую внутри value                          │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Парсинг тегов:**
```go
import "reflect"

type User struct {
    ID        int       `json:"id" db:"user_id" validate:"required,gt=0"`
    Name      string    `json:"name" db:"user_name" validate:"required,min=2,max=100"`
    Email     string    `json:"email,omitempty" db:"email" validate:"required,email"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

func ParseTags() {
    t := reflect.TypeOf(User{})

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)

        // Получаем конкретный тег
        jsonTag := field.Tag.Get("json")
        dbTag := field.Tag.Get("db")
        validateTag := field.Tag.Get("validate")

        fmt.Printf("Field: %s\n", field.Name)
        fmt.Printf("  json: %s\n", jsonTag)
        fmt.Printf("  db: %s\n", dbTag)
        fmt.Printf("  validate: %s\n", validateTag)
    }
}

// Парсинг с опциями
func ParseJSONTag(tag string) (name string, omitempty bool) {
    if tag == "" {
        return "", false
    }

    parts := strings.Split(tag, ",")
    name = parts[0]

    for _, opt := range parts[1:] {
        if opt == "omitempty" {
            omitempty = true
        }
    }

    return name, omitempty
}
```

**Custom tag parser:**
```go
type ValidationRule struct {
    Name  string
    Param string
}

func ParseValidateTag(tag string) []ValidationRule {
    if tag == "" {
        return nil
    }

    var rules []ValidationRule
    parts := strings.Split(tag, ",")

    for _, part := range parts {
        if idx := strings.Index(part, "="); idx != -1 {
            rules = append(rules, ValidationRule{
                Name:  part[:idx],
                Param: part[idx+1:],
            })
        } else {
            rules = append(rules, ValidationRule{Name: part})
        }
    }

    return rules
}

// Использование
// validate:"required,min=2,max=100,email"
// → [{Name: "required"}, {Name: "min", Param: "2"}, ...]
```

**Валидатор на основе тегов:**
```go
type Validator struct {
    validators map[string]ValidatorFunc
}

type ValidatorFunc func(value reflect.Value, param string) error

func NewValidator() *Validator {
    v := &Validator{
        validators: make(map[string]ValidatorFunc),
    }

    v.validators["required"] = func(value reflect.Value, _ string) error {
        if value.IsZero() {
            return errors.New("field is required")
        }
        return nil
    }

    v.validators["min"] = func(value reflect.Value, param string) error {
        min, _ := strconv.Atoi(param)
        switch value.Kind() {
        case reflect.String:
            if len(value.String()) < min {
                return fmt.Errorf("must be at least %d characters", min)
            }
        case reflect.Int, reflect.Int64:
            if value.Int() < int64(min) {
                return fmt.Errorf("must be at least %d", min)
            }
        }
        return nil
    }

    v.validators["email"] = func(value reflect.Value, _ string) error {
        email := value.String()
        if !strings.Contains(email, "@") {
            return errors.New("invalid email format")
        }
        return nil
    }

    return v
}

func (v *Validator) Validate(obj interface{}) error {
    rv := reflect.ValueOf(obj)
    if rv.Kind() == reflect.Ptr {
        rv = rv.Elem()
    }

    rt := rv.Type()

    for i := 0; i < rv.NumField(); i++ {
        field := rt.Field(i)
        value := rv.Field(i)

        tag := field.Tag.Get("validate")
        if tag == "" {
            continue
        }

        rules := ParseValidateTag(tag)
        for _, rule := range rules {
            validator, ok := v.validators[rule.Name]
            if !ok {
                continue
            }

            if err := validator(value, rule.Param); err != nil {
                return fmt.Errorf("field %s: %w", field.Name, err)
            }
        }
    }

    return nil
}
```

**Lookup vs Get:**
```go
field := t.Field(0)

// Get — возвращает значение или пустую строку
json := field.Tag.Get("json")

// Lookup — возвращает значение и флаг наличия
json, ok := field.Tag.Lookup("json")
if !ok {
    // тег отсутствует
}
```

**Типичные ошибки.**
1. **Пробел после :** — `json: "name"` не парсится
2. **Одинарные кавычки** — `json:'name'` невалидно
3. **Get на несуществующий** — возвращает "", не ошибку
4. **Неэкспортируемые поля** — теги игнорируются

**На интервью.**
Покажи формат struct tag. Напиши парсер для validation tags. Объясни разницу Get vs Lookup.

*Частые follow-up вопросы:*
- Как валидировать формат тегов?
- Как сделать custom marshaler на основе тегов?
- Почему теги — строки, а не typed?

---

### 5. Как реализовать Dependency Injection без рефлексии?

**Зачем спрашивают.**
DI — важный паттерн для тестируемости. Интервьюер проверяет знание подходов: manual, wire, fx.

**Короткий ответ.**
Wire — compile-time DI через кодогенерацию. Определяешь providers (конструкторы), wire генерирует wiring code. Альтернатива: manual DI (конструкторы с зависимостями), fx (runtime).

**Детальный разбор.**

**Подходы к DI:**
```
┌─────────────────────────────────────────────────────────────┐
│ 1. Manual DI (конструкторы)                                  │
│    Плюс: простота, нет зависимостей                         │
│    Минус: boilerplate при большом графе                     │
├─────────────────────────────────────────────────────────────┤
│ 2. Wire (compile-time codegen)                               │
│    Плюс: type-safe, compile-time ошибки                     │
│    Минус: learning curve, go:generate                       │
├─────────────────────────────────────────────────────────────┤
│ 3. Fx/Dig (runtime reflection)                               │
│    Плюс: flexible, lifecycle management                     │
│    Минус: runtime errors, медленнее                         │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Manual DI:**
```go
// Зависимости
type Config struct {
    DBHost string
    DBPort int
}

type DB struct {
    conn *sql.DB
}

func NewDB(cfg *Config) (*DB, error) {
    dsn := fmt.Sprintf("host=%s port=%d", cfg.DBHost, cfg.DBPort)
    conn, err := sql.Open("postgres", dsn)
    return &DB{conn: conn}, err
}

type UserRepository struct {
    db *DB
}

func NewUserRepository(db *DB) *UserRepository {
    return &UserRepository{db: db}
}

type UserService struct {
    repo *UserRepository
}

func NewUserService(repo *UserRepository) *UserService {
    return &UserService{repo: repo}
}

// Manual wiring
func main() {
    cfg := &Config{DBHost: "localhost", DBPort: 5432}

    db, err := NewDB(cfg)
    if err != nil {
        log.Fatal(err)
    }

    repo := NewUserRepository(db)
    service := NewUserService(repo)

    // Использование service
}
```

**Wire (compile-time):**
```go
// wire.go
//go:build wireinject

package main

import "github.com/google/wire"

// Providers
var ConfigSet = wire.NewSet(
    NewConfig,
)

var DBSet = wire.NewSet(
    NewDB,
)

var RepositorySet = wire.NewSet(
    NewUserRepository,
)

var ServiceSet = wire.NewSet(
    NewUserService,
)

// Injector
func InitializeApp() (*App, error) {
    wire.Build(
        ConfigSet,
        DBSet,
        RepositorySet,
        ServiceSet,
        NewApp,
    )
    return nil, nil  // wire заменит
}

// После go generate wire создаст wire_gen.go:
func InitializeApp() (*App, error) {
    config := NewConfig()
    db, err := NewDB(config)
    if err != nil {
        return nil, err
    }
    userRepository := NewUserRepository(db)
    userService := NewUserService(userRepository)
    app := NewApp(userService)
    return app, nil
}
```

**Wire с интерфейсами:**
```go
// Определяем интерфейсы
type UserRepository interface {
    GetByID(ctx context.Context, id string) (*User, error)
}

type userRepository struct {
    db *DB
}

func NewUserRepository(db *DB) UserRepository {
    return &userRepository{db: db}
}

// Wire binding
var RepositorySet = wire.NewSet(
    NewUserRepository,
    wire.Bind(new(UserRepository), new(*userRepository)),
)
```

**Wire с cleanup:**
```go
func NewDB(cfg *Config) (*DB, func(), error) {
    conn, err := sql.Open("postgres", cfg.DSN)
    if err != nil {
        return nil, nil, err
    }

    cleanup := func() {
        conn.Close()
    }

    return &DB{conn: conn}, cleanup, nil
}

// Wire автоматически агрегирует cleanup функции
func InitializeApp() (*App, func(), error) {
    wire.Build(...)
    return nil, nil, nil
}

// Использование
func main() {
    app, cleanup, err := InitializeApp()
    if err != nil {
        log.Fatal(err)
    }
    defer cleanup()

    app.Run()
}
```

**Uber Fx (runtime):**
```go
import "go.uber.org/fx"

func main() {
    fx.New(
        fx.Provide(
            NewConfig,
            NewDB,
            NewUserRepository,
            NewUserService,
        ),
        fx.Invoke(func(service *UserService) {
            // Использование
        }),
    ).Run()
}

// С lifecycle
func NewDB(lc fx.Lifecycle, cfg *Config) *DB {
    db := &DB{}

    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            return db.Connect(cfg)
        },
        OnStop: func(ctx context.Context) error {
            return db.Close()
        },
    })

    return db
}
```

**Типичные ошибки.**
1. **Circular dependencies** — wire/fx не смогут resolve
2. **Нет интерфейсов** — сложно мокать
3. **Слишком много в одном Set** — сложно понять граф
4. **Fx в библиотеках** — только в main

**На интервью.**
Покажи manual DI vs Wire. Объясни, когда wire лучше fx. Расскажи про cleanup и lifecycle.

*Частые follow-up вопросы:*
- Как тестировать с DI?
- Wire vs Fx — когда что?
- Как избежать circular dependencies?

---

### 6. Как использовать пакет ast для анализа кода?

**Зачем спрашивают.**
go/ast — основа для статического анализа и кодогенерации. Интервьюер проверяет понимание AST и практические применения.

**Короткий ответ.**
`go/ast` представляет Go код как дерево. Парсинг: `go/parser`. Обход: `ast.Inspect`. Применения: линтеры, генераторы, рефакторинг.

**Детальный разбор.**

**Структура AST:**
```
┌─────────────────────────────────────────────────────────────┐
│ package main                                                 │
│                                                              │
│ func Add(a, b int) int {                                    │
│     return a + b                                            │
│ }                                                            │
│                                                              │
│ AST:                                                         │
│ File                                                         │
│ └── Name: main                                              │
│ └── Decls:                                                  │
│     └── FuncDecl                                            │
│         └── Name: Add                                       │
│         └── Type: FuncType                                  │
│             └── Params: [a int, b int]                      │
│             └── Results: [int]                              │
│         └── Body: BlockStmt                                 │
│             └── List:                                       │
│                 └── ReturnStmt                              │
│                     └── Results:                            │
│                         └── BinaryExpr (a + b)              │
└─────────────────────────────────────────────────────────────┘
```

**Пример.**

**Парсинг и обход:**
```go
import (
    "go/ast"
    "go/parser"
    "go/token"
)

func AnalyzeFile(filename string) error {
    fset := token.NewFileSet()

    file, err := parser.ParseFile(fset, filename, nil, parser.ParseComments)
    if err != nil {
        return err
    }

    // Вывод имени пакета
    fmt.Println("Package:", file.Name.Name)

    // Обход всех узлов
    ast.Inspect(file, func(n ast.Node) bool {
        switch node := n.(type) {
        case *ast.FuncDecl:
            fmt.Printf("Function: %s\n", node.Name.Name)
            if node.Recv != nil {
                // Метод
                for _, recv := range node.Recv.List {
                    fmt.Printf("  Receiver: %s\n", exprToString(recv.Type))
                }
            }

        case *ast.TypeSpec:
            fmt.Printf("Type: %s\n", node.Name.Name)

        case *ast.ImportSpec:
            fmt.Printf("Import: %s\n", node.Path.Value)
        }

        return true  // продолжить обход
    })

    return nil
}
```

**Поиск определённых паттернов:**
```go
// Найти все fmt.Println вызовы
func FindPrintln(file *ast.File) []token.Pos {
    var positions []token.Pos

    ast.Inspect(file, func(n ast.Node) bool {
        call, ok := n.(*ast.CallExpr)
        if !ok {
            return true
        }

        sel, ok := call.Fun.(*ast.SelectorExpr)
        if !ok {
            return true
        }

        ident, ok := sel.X.(*ast.Ident)
        if !ok {
            return true
        }

        if ident.Name == "fmt" && sel.Sel.Name == "Println" {
            positions = append(positions, call.Pos())
        }

        return true
    })

    return positions
}

// Найти неиспользуемые параметры
func FindUnusedParams(funcDecl *ast.FuncDecl) []string {
    if funcDecl.Body == nil {
        return nil
    }

    // Собираем все параметры
    params := make(map[string]bool)
    for _, field := range funcDecl.Type.Params.List {
        for _, name := range field.Names {
            if name.Name != "_" {
                params[name.Name] = false
            }
        }
    }

    // Ищем использования в теле
    ast.Inspect(funcDecl.Body, func(n ast.Node) bool {
        ident, ok := n.(*ast.Ident)
        if ok {
            if _, exists := params[ident.Name]; exists {
                params[ident.Name] = true
            }
        }
        return true
    })

    // Неиспользованные
    var unused []string
    for name, used := range params {
        if !used {
            unused = append(unused, name)
        }
    }

    return unused
}
```

**Модификация AST:**
```go
// Добавить комментарий к функции
func AddDocComment(file *ast.File, funcName, comment string) {
    for _, decl := range file.Decls {
        funcDecl, ok := decl.(*ast.FuncDecl)
        if !ok || funcDecl.Name.Name != funcName {
            continue
        }

        funcDecl.Doc = &ast.CommentGroup{
            List: []*ast.Comment{
                {Text: "// " + comment},
            },
        }
        break
    }
}

// Переименовать функцию
func RenameFunc(file *ast.File, oldName, newName string) {
    ast.Inspect(file, func(n ast.Node) bool {
        switch node := n.(type) {
        case *ast.FuncDecl:
            if node.Name.Name == oldName {
                node.Name.Name = newName
            }
        case *ast.CallExpr:
            if ident, ok := node.Fun.(*ast.Ident); ok {
                if ident.Name == oldName {
                    ident.Name = newName
                }
            }
        }
        return true
    })
}
```

**Генерация кода из AST:**
```go
import (
    "go/ast"
    "go/format"
    "go/token"
)

func GenerateGetter(structName, fieldName, fieldType string) string {
    // Создаём AST программно
    funcDecl := &ast.FuncDecl{
        Recv: &ast.FieldList{
            List: []*ast.Field{
                {
                    Names: []*ast.Ident{ast.NewIdent("s")},
                    Type:  &ast.StarExpr{X: ast.NewIdent(structName)},
                },
            },
        },
        Name: ast.NewIdent("Get" + fieldName),
        Type: &ast.FuncType{
            Params: &ast.FieldList{},
            Results: &ast.FieldList{
                List: []*ast.Field{
                    {Type: ast.NewIdent(fieldType)},
                },
            },
        },
        Body: &ast.BlockStmt{
            List: []ast.Stmt{
                &ast.ReturnStmt{
                    Results: []ast.Expr{
                        &ast.SelectorExpr{
                            X:   ast.NewIdent("s"),
                            Sel: ast.NewIdent(fieldName),
                        },
                    },
                },
            },
        },
    }

    var buf bytes.Buffer
    fset := token.NewFileSet()
    format.Node(&buf, fset, funcDecl)

    return buf.String()
}

// Результат:
// func (s *User) GetName() string {
//     return s.Name
// }
```

**Типичные ошибки.**
1. **Не сохранять FileSet** — теряются позиции
2. **Мутировать во время Inspect** — undefined behavior
3. **Игнорировать comments** — теряются при генерации
4. **Не форматировать вывод** — нечитаемый код

**На интервью.**
Покажи парсинг и обход AST. Напиши поиск паттерна (fmt.Println). Объясни, как генерировать код из AST.

*Частые follow-up вопросы:*
- Как работает go/types для type checking?
- Как написать кастомный линтер?
- Когда text/template лучше AST?

---

## См. также

- [Java Reflection & Annotations](../01-java/11-reflection-annotations.md) — рефлексия в Java
- [Python Metaprogramming](../02-python/12-metaprogramming.md) — метапрограммирование в Python
- [Tooling & Testing](./07-tooling-testing.md) — go generate и инструменты сборки
- [Language Overview](./00-language-overview.md) — система типов Go
- [Clean Architecture](../08-architecture/01-clean-architecture.md) — DI и генерация кода

---

## Практика

1. **Reflect:** Напиши generic deep copy функцию. Сравни производительность с manual copy.

2. **Struct Tags:** Реализуй simple validator на основе тегов. Поддержи required, min, max.

3. **go:generate:** Напиши генератор для создания constructor функций из struct полей.

4. **Wire:** Настрой DI для небольшого приложения. Добавь lifecycle hooks.

5. **AST:** Напиши линтер, который находит функции без документации.

6. **Code Gen:** Создай генератор моков из интерфейсов (упрощённый mockgen).

---

## Дополнительные материалы

- [reflect package](https://pkg.go.dev/reflect)
- [go/ast package](https://pkg.go.dev/go/ast)
- [go/parser package](https://pkg.go.dev/go/parser)
- [Wire Tutorial](https://github.com/google/wire/blob/main/docs/guide.md)
- [Uber Fx](https://uber-go.github.io/fx/)
- [Writing Go Analyzers](https://github.com/golang/tools/blob/master/go/analysis/doc.go)
- [The Laws of Reflection](https://go.dev/blog/laws-of-reflection)
- Talks: "Advanced Testing with Go" — Mitchell Hashimoto
