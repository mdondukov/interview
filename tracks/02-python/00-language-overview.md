# 00. Обзор языка Python

[← Назад к списку тем](README.md)

---

## Вопрос 1: Чем Python отличается от других языков? Что такое Zen of Python?

### Зачем спрашивают
Проверяют понимание философии языка и умение писать "pythonic" код.

### Короткий ответ
Python — интерпретируемый, динамически типизированный язык с акцентом на читаемость. Zen of Python (`import this`) — 19 афоризмов о философии языка: "Explicit is better than implicit", "Simple is better than complex".

### Детальный разбор

**Ключевые характеристики Python:**
- **Интерпретируемый** — код выполняется построчно через интерпретатор (CPython, PyPy)
- **Динамическая типизация** — типы проверяются в runtime, не при компиляции
- **Сильная типизация** — нет неявных преобразований (`"1" + 1` вызовет TypeError)
- **Duck typing** — "если выглядит как утка и крякает как утка, это утка"
- **Garbage collection** — автоматическое управление памятью

**Zen of Python (ключевые принципы):**
```
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Errors should never pass silently.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
```

### Пример

```python
# НЕ pythonic
def get_first(lst):
    if len(lst) > 0:
        return lst[0]
    else:
        return None

# Pythonic
def get_first(lst):
    return lst[0] if lst else None

# Ещё лучше - EAFP (Easier to Ask Forgiveness than Permission)
def get_first(lst):
    try:
        return lst[0]
    except IndexError:
        return None
```

### Типичные ошибки
- Путать сильную и статическую типизацию
- Не знать, что Python сильно типизирован (нельзя сложить строку и число)
- Писать код в стиле Java/C++ вместо pythonic подхода

### На интервью
**Как отвечать:** Начните с главных характеристик (интерпретируемый, динамический, сильно типизированный), затем упомяните Zen of Python и приведите пример pythonic кода.

**Follow-up вопросы:**
- В чём разница между CPython и PyPy?
- Что такое EAFP vs LBYL?
- Почему Python медленнее C++?

---

## Вопрос 2: Как работает интерпретатор CPython?

### Зачем спрашивают
Проверяют понимание внутреннего устройства Python для отладки и оптимизации.

### Короткий ответ
CPython компилирует исходный код в байткод (.pyc), который затем выполняется виртуальной машиной Python (PVM). Байткод — промежуточное представление, не машинный код.

### Детальный разбор

**Этапы выполнения Python кода:**

```
Source Code (.py)
       ↓
    Lexer (токенизация)
       ↓
    Parser (AST)
       ↓
    Compiler (байткод)
       ↓
    Bytecode (.pyc)
       ↓
    PVM (интерпретация)
       ↓
    Результат
```

**Байткод:**
- Хранится в `__pycache__/` директории
- Формат файла: `module.cpython-311.pyc`
- Можно посмотреть через модуль `dis`

**Виртуальная машина (PVM):**
- Stack-based (использует стек для операций)
- Выполняет байткод инструкция за инструкцией
- GIL (Global Interpreter Lock) предотвращает параллельное выполнение байткода

### Пример

```python
import dis

def add(a, b):
    return a + b

# Посмотреть байткод
dis.dis(add)
"""
  2           0 LOAD_FAST                0 (a)
              2 LOAD_FAST                1 (b)
              4 BINARY_ADD
              6 RETURN_VALUE
"""

# Проверить байткод функции
print(add.__code__.co_code)      # bytes объект
print(add.__code__.co_consts)    # константы
print(add.__code__.co_varnames)  # локальные переменные
```

### Типичные ошибки
- Путать компиляцию в байткод с компиляцией в машинный код
- Думать, что .pyc файлы ускоряют выполнение (только загрузку)
- Не знать о существовании GIL

### На интервью
**Как отвечать:** Опишите pipeline: исходный код → AST → байткод → PVM. Упомяните GIL как важную особенность CPython.

**Follow-up вопросы:**
- Что делает GIL и почему он существует?
- Чем PyPy отличается от CPython?
- Как работает JIT в PyPy?

---

## Вопрос 3: Что такое duck typing и как он работает?

### Зачем спрашивают
Проверяют понимание философии Python и умение писать гибкий код.

### Короткий ответ
Duck typing — подход, при котором тип объекта определяется не наследованием, а наличием нужных методов. "Если объект крякает как утка и плавает как утка — это утка". Python проверяет наличие методов в runtime, а не при компиляции.

### Детальный разбор

**Принцип:**
```python
# Нам не важен тип, важно наличие метода
def process(obj):
    return obj.read()  # Любой объект с методом read()
```

**Duck typing vs Nominal typing:**
- **Nominal typing** (Java, C#): тип определяется явным наследованием
- **Duck typing** (Python, Ruby): тип определяется поведением
- **Structural typing** (Go interfaces, TypeScript): компромисс — проверка структуры при компиляции

**Протоколы в Python (формализация duck typing):**
```python
from typing import Protocol

class Readable(Protocol):
    def read(self) -> str: ...

# Любой класс с методом read() соответствует протоколу
# Без явного наследования!
```

### Пример

```python
# Duck typing в действии
class Duck:
    def quack(self):
        return "Quack!"

class Person:
    def quack(self):
        return "I'm quacking like a duck!"

class Dog:
    def bark(self):
        return "Woof!"

def make_it_quack(thing):
    # Не проверяем тип, просто вызываем метод
    return thing.quack()

# Работает с любым объектом, у которого есть quack()
print(make_it_quack(Duck()))    # Quack!
print(make_it_quack(Person()))  # I'm quacking like a duck!
print(make_it_quack(Dog()))     # AttributeError: 'Dog' object has no attribute 'quack'

# Проверка наличия метода (LBYL)
def safe_quack(thing):
    if hasattr(thing, 'quack'):
        return thing.quack()
    return "Can't quack"

# Или через try/except (EAFP - более pythonic)
def safe_quack_eafp(thing):
    try:
        return thing.quack()
    except AttributeError:
        return "Can't quack"
```

### Типичные ошибки
- Проверять типы через `isinstance()` там, где достаточно duck typing
- Не обрабатывать AttributeError при использовании duck typing
- Путать duck typing и динамическую типизацию

### На интервью
**Как отвечать:** Объясните концепцию через пример с "уткой". Покажите, как это связано с Protocol из typing модуля для статической проверки.

**Follow-up вопросы:**
- Когда лучше использовать isinstance() вместо duck typing?
- Что такое Protocol и как он связан с duck typing?
- Чем отличается EAFP от LBYL?

---

## Вопрос 4: В чём разница между Python 2 и Python 3?

### Зачем спрашивают
Проверяют осведомлённость об эволюции языка (хотя Python 2 устарел, вопрос показывает глубину знаний).

### Короткий ответ
Python 3 — несовместимая версия с улучшенной обработкой Unicode (строки по умолчанию Unicode), новым синтаксисом (print функция, / для true division), и удалением устаревших конструкций. Python 2 EOL был в январе 2020.

### Детальный разбор

**Ключевые отличия:**

| Аспект | Python 2 | Python 3 |
|--------|----------|----------|
| print | `print "hello"` | `print("hello")` |
| Деление | `5/2 = 2` | `5/2 = 2.5`, `5//2 = 2` |
| Строки | ASCII по умолчанию | Unicode по умолчанию |
| range() | Возвращает list | Возвращает iterator |
| input() | Выполняет eval() | Возвращает строку |
| dict.keys() | Возвращает list | Возвращает view |

**Новые возможности Python 3:**

```python
# f-strings (3.6+)
name = "World"
print(f"Hello, {name}!")

# Walrus operator (3.8+)
if (n := len(data)) > 10:
    print(f"Too long: {n}")

# Pattern matching (3.10+)
match command:
    case ["quit"]:
        quit()
    case ["load", filename]:
        load(filename)

# Union types (3.10+)
def process(x: int | str) -> None: ...

# Type parameter syntax (3.12+)
def first[T](lst: list[T]) -> T: ...
```

### Пример

```python
# Python 2 код (НЕ работает в Python 3)
# print "Hello"
# xrange(10)
# u"unicode string"
# dict.iteritems()

# Python 3 эквивалент
print("Hello")
range(10)  # lazy iterator
"unicode string"  # по умолчанию unicode
dict.items()  # возвращает view

# Миграция кода
# Используйте: python -m lib2to3 -w script.py
# Или: pip install future; futurize script.py
```

### Типичные ошибки
- Не знать про EOL Python 2 (январь 2020)
- Путать поведение деления (/ vs //)
- Забывать про различия в обработке Unicode

### На интервью
**Как отвечать:** Упомяните главные отличия (print, деление, Unicode), затем перечислите новые возможности Python 3.x (f-strings, walrus, pattern matching).

**Follow-up вопросы:**
- Что нового в Python 3.10/3.11/3.12?
- Как мигрировать код с Python 2 на Python 3?
- Почему Python 3 изначально был непопулярен?

---

## Вопрос 5: Что такое LEGB и как работает scope resolution?

### Зачем спрашивают
Проверяют понимание области видимости переменных — критично для отладки и написания корректного кода.

### Короткий ответ
LEGB — порядок поиска переменных: Local → Enclosing → Global → Built-in. Python ищет имя в локальной области, затем во вложенных функциях, глобальной области модуля, и наконец во встроенных именах.

### Детальный разбор

**LEGB правило:**

```
Built-in (builtins)
    ↑
Global (module level)
    ↑
Enclosing (outer functions)
    ↑
Local (current function)
```

**Ключевые слова для управления scope:**
- `global` — объявить переменную глобальной
- `nonlocal` — ссылаться на переменную из enclosing scope

**Важные правила:**
1. Присваивание создаёт локальную переменную
2. Чтение без присваивания ищет по LEGB
3. `global`/`nonlocal` должны быть до использования переменной

### Пример

```python
# LEGB демонстрация
x = "global"  # Global

def outer():
    x = "enclosing"  # Enclosing

    def inner():
        x = "local"  # Local
        print(x)  # local

    inner()
    print(x)  # enclosing

outer()
print(x)  # global

# Использование global
counter = 0

def increment():
    global counter
    counter += 1

increment()
print(counter)  # 1

# Использование nonlocal
def make_counter():
    count = 0

    def counter():
        nonlocal count
        count += 1
        return count

    return counter

c = make_counter()
print(c())  # 1
print(c())  # 2

# Частая ошибка - UnboundLocalError
x = 10

def broken():
    print(x)  # UnboundLocalError!
    x = 20    # Это делает x локальной во всей функции

def fixed():
    global x
    print(x)  # 10
    x = 20
```

### Типичные ошибки
- UnboundLocalError из-за присваивания после чтения
- Забывать про `nonlocal` в замыканиях
- Злоупотреблять `global` (плохая практика)

### На интервью
**Как отвечать:** Расшифруйте LEGB, покажите пример с вложенными функциями, объясните разницу между `global` и `nonlocal`.

**Follow-up вопросы:**
- Почему `global` считается плохой практикой?
- Что такое замыкание (closure)?
- Как работает `__slots__` с точки зрения scope?

---

## Вопрос 6: Как работают генераторы и итераторы?

### Зачем спрашивают
Генераторы — один из самых мощных инструментов Python для работы с большими данными и ленивыми вычислениями.

### Короткий ответ
Итератор — объект с методами `__iter__()` и `__next__()`. Генератор — функция с `yield`, которая возвращает итератор и выполняется лениво (по требованию). Генераторы экономят память, так как не хранят все значения сразу.

### Детальный разбор

**Итератор (Iterator Protocol):**
```python
class MyIterator:
    def __iter__(self):
        return self

    def __next__(self):
        # Возвращает следующий элемент или raise StopIteration
        pass
```

**Генератор (Generator):**
- Функция с `yield` вместо `return`
- Сохраняет состояние между вызовами
- Автоматически реализует iterator protocol

**Generator expression:**
```python
# List comprehension - создаёт список в памяти
squares_list = [x**2 for x in range(1000000)]

# Generator expression - ленивое вычисление
squares_gen = (x**2 for x in range(1000000))
```

**Методы генератора:**
- `send(value)` — отправить значение в генератор
- `throw(exception)` — выбросить исключение внутри
- `close()` — завершить генератор

### Пример

```python
# Простой генератор
def countdown(n):
    while n > 0:
        yield n
        n -= 1

for i in countdown(5):
    print(i)  # 5, 4, 3, 2, 1

# Генератор vs обычная функция
def get_squares_list(n):
    result = []
    for i in range(n):
        result.append(i ** 2)
    return result  # Всё в памяти сразу

def get_squares_gen(n):
    for i in range(n):
        yield i ** 2  # По одному значению

# Бесконечный генератор
def infinite_counter():
    n = 0
    while True:
        yield n
        n += 1

# yield from для делегирования
def chain(*iterables):
    for iterable in iterables:
        yield from iterable

list(chain([1, 2], [3, 4]))  # [1, 2, 3, 4]

# Корутина с send()
def accumulator():
    total = 0
    while True:
        value = yield total
        if value is not None:
            total += value

acc = accumulator()
next(acc)       # Инициализация, возвращает 0
acc.send(10)    # 10
acc.send(20)    # 30
```

### Типичные ошибки
- Забывать, что генератор можно итерировать только один раз
- Использовать list comprehension для больших данных вместо генератора
- Не вызывать `next()` перед `send()` для инициализации

### На интервью
**Как отвечать:** Объясните разницу между итератором и генератором, покажите экономию памяти на примере, упомяните `yield from` и `send()`.

**Follow-up вопросы:**
- Что такое `yield from`?
- Как генераторы используются в asyncio?
- Как реализовать бесконечный итератор?

---

## Вопрос 7: Что такое comprehensions и когда их использовать?

### Зачем спрашивают
Comprehensions — идиоматический Python. Проверяют умение писать краткий и читаемый код.

### Короткий ответ
Comprehensions — синтаксический сахар для создания коллекций: list `[x for x in ...]`, dict `{k: v for ...}`, set `{x for x in ...}`. Они быстрее циклов и более читаемы для простых преобразований.

### Детальный разбор

**Типы comprehensions:**

```python
# List comprehension
[expression for item in iterable if condition]

# Dict comprehension
{key: value for item in iterable if condition}

# Set comprehension
{expression for item in iterable if condition}

# Generator expression (не comprehension, но похоже)
(expression for item in iterable if condition)
```

**Вложенные comprehensions:**
```python
# Эквивалент вложенных циклов
# for i in range(3):
#     for j in range(3):
#         result.append((i, j))

[(i, j) for i in range(3) for j in range(3)]
```

**Когда НЕ использовать:**
- Сложная логика (больше одного условия)
- Side effects (print, append к внешнему списку)
- Когда нужно несколько операций

### Пример

```python
# List comprehension
numbers = [1, 2, 3, 4, 5]
squares = [n ** 2 for n in numbers]  # [1, 4, 9, 16, 25]

# С условием
evens = [n for n in numbers if n % 2 == 0]  # [2, 4]

# Dict comprehension
word_lengths = {word: len(word) for word in ["hello", "world"]}
# {'hello': 5, 'world': 5}

# Set comprehension
unique_lengths = {len(word) for word in ["hi", "hello", "by"]}
# {2, 5}

# Вложенный comprehension (матрица)
matrix = [[j for j in range(3)] for i in range(3)]
# [[0, 1, 2], [0, 1, 2], [0, 1, 2]]

# Flatten (развернуть матрицу)
flat = [x for row in matrix for x in row]
# [0, 1, 2, 0, 1, 2, 0, 1, 2]

# Walrus operator в comprehension (3.8+)
results = [y for x in data if (y := process(x)) is not None]

# Плохо - слишком сложно
# [x.strip().lower() for x in lines if x.strip() and not x.strip().startswith('#')]

# Лучше - вынести в функцию
def is_valid_line(line):
    stripped = line.strip()
    return stripped and not stripped.startswith('#')

[x.strip().lower() for x in lines if is_valid_line(x)]
```

### Типичные ошибки
- Слишком сложные comprehensions (более одной строки)
- Использовать для side effects
- Путать порядок циклов в вложенных comprehensions

### На интервью
**Как отвечать:** Покажите все типы comprehensions, объясните когда использовать, когда нет. Упомяните walrus operator для Python 3.8+.

**Follow-up вопросы:**
- Что быстрее: comprehension или map/filter?
- Как работает walrus operator в comprehensions?
- Когда лучше использовать обычный цикл?

---

## Вопрос 8: Что такое pattern matching (match/case) в Python 3.10+?

### Зачем спрашивают
Pattern matching — крупнейшее синтаксическое добавление в Python за последние годы. Проверяют знание современного Python.

### Короткий ответ
Pattern matching (`match`/`case`) — структурное сопоставление с образцом, добавленное в Python 3.10. Позволяет деконструировать данные и проверять структуру объектов. Мощнее switch/case из других языков.

### Детальный разбор

**Типы паттернов:**

1. **Literal patterns** — точное совпадение значения
2. **Capture patterns** — захват значения в переменную
3. **Wildcard pattern** (`_`) — любое значение
4. **Sequence patterns** — списки, кортежи
5. **Mapping patterns** — словари
6. **Class patterns** — объекты классов
7. **OR patterns** (`|`) — альтернативы
8. **Guard clauses** (`if`) — дополнительные условия

### Пример

```python
# Literal pattern
def http_status(status):
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500:
            return "Server Error"
        case _:
            return "Unknown"

# Sequence pattern с capture
def process_point(point):
    match point:
        case (0, 0):
            return "Origin"
        case (x, 0):
            return f"On X axis at {x}"
        case (0, y):
            return f"On Y axis at {y}"
        case (x, y):
            return f"Point at ({x}, {y})"

# Mapping pattern
def process_command(command):
    match command:
        case {"action": "quit"}:
            return "Quitting"
        case {"action": "move", "direction": d}:
            return f"Moving {d}"
        case {"action": "attack", "target": t, **rest}:
            return f"Attacking {t} with {rest}"

# Class pattern
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

def describe(obj):
    match obj:
        case Point(x=0, y=0):
            return "Origin"
        case Point(x=x, y=y) if x == y:
            return f"On diagonal at {x}"
        case Point():
            return "Some point"

# OR pattern и guard
def categorize(value):
    match value:
        case 0 | 1 | 2:
            return "Small"
        case n if n < 0:
            return "Negative"
        case n if n > 100:
            return "Large"
        case _:
            return "Medium"

# Вложенные паттерны
def process_data(data):
    match data:
        case {"users": [{"name": name, "role": "admin"}, *rest]}:
            return f"Admin: {name}, others: {len(rest)}"
        case {"users": []}:
            return "No users"
```

### Типичные ошибки
- Путать с switch/case (pattern matching мощнее)
- Забывать `_` для default case
- Неправильный порядок паттернов (более специфичные должны быть первыми)

### На интервью
**Как отвечать:** Объясните, что это структурное сопоставление (не просто switch). Покажите деконструкцию данных на примере sequence или mapping pattern.

**Follow-up вопросы:**
- Чем отличается от switch в других языках?
- Как работает guard clause?
- Как использовать с dataclass?

---

## Вопрос 9: Какие есть способы форматирования строк?

### Зачем спрашивают
Базовый вопрос, но показывает эволюцию языка и знание best practices.

### Короткий ответ
В Python есть 4 способа: `%` formatting (старый), `.format()` (Python 2.6+), f-strings (Python 3.6+, рекомендуемый), Template strings (для пользовательского ввода). F-strings — самый быстрый и читаемый способ.

### Детальный разбор

**Сравнение методов:**

| Метод | Синтаксис | Производительность | Безопасность |
|-------|-----------|-------------------|--------------|
| % | `"Hello %s" % name` | Средняя | Низкая |
| .format() | `"Hello {}".format(name)` | Средняя | Низкая |
| f-string | `f"Hello {name}"` | Высокая | Низкая |
| Template | `Template("Hello $name")` | Низкая | Высокая |

**Когда что использовать:**
- **f-strings** — по умолчанию для всего
- **Template** — для пользовательского ввода (защита от инъекций)
- **%** — logging (по историческим причинам)
- **.format()** — когда строка определяется заранее

### Пример

```python
name = "Alice"
age = 30
price = 19.99

# %-formatting (устаревший)
print("Name: %s, Age: %d" % (name, age))
print("Price: %.2f" % price)

# .format()
print("Name: {}, Age: {}".format(name, age))
print("Name: {n}, Age: {a}".format(n=name, a=age))
print("{0} is {1} years old. {0} likes Python.".format(name, age))

# f-strings (рекомендуется)
print(f"Name: {name}, Age: {age}")
print(f"Price: {price:.2f}")
print(f"Next year: {age + 1}")

# Выражения внутри f-string
print(f"Upper: {name.upper()}")
print(f"List: {[x**2 for x in range(5)]}")

# Форматирование в f-string
number = 1234567.89
print(f"{number:,.2f}")      # 1,234,567.89
print(f"{number:>15,.2f}")   # "  1,234,567.89"
print(f"{42:08d}")           # 00000042
print(f"{0.5:.0%}")          # 50%

# Debug mode (Python 3.8+)
x = 10
print(f"{x=}")       # x=10
print(f"{x=:.2f}")   # x=10.00

# Template strings (безопасно для пользовательского ввода)
from string import Template
t = Template("Hello, $name!")
print(t.substitute(name=name))

# ОПАСНО с f-strings!
user_input = "__import__('os').system('ls')"
# eval(f"print({user_input})")  # НЕ ДЕЛАЙТЕ ТАК!
```

### Типичные ошибки
- Использовать % formatting в новом коде
- Не знать про `=` для debug в f-strings
- Использовать f-strings с пользовательским вводом без санитизации

### На интервью
**Как отвечать:** Начните с f-strings как рекомендуемого способа, упомяните производительность и фичи (выражения, форматирование, debug mode). Объясните когда использовать Template.

**Follow-up вопросы:**
- Почему f-strings быстрее .format()?
- Как форматировать даты в f-strings?
- Какие есть проблемы безопасности с f-strings?

---

## Вопрос 10: Что такое `__name__ == "__main__"` и зачем это нужно?

### Зачем спрашивают
Базовый паттерн Python, показывает понимание модульной системы.

### Короткий ответ
`__name__` — специальная переменная, равная `"__main__"` при прямом запуске файла и имени модуля при импорте. Конструкция `if __name__ == "__main__":` позволяет выполнять код только при прямом запуске, но не при импорте.

### Детальный разбор

**Как работает:**

```python
# module.py
print(f"__name__ = {__name__}")

def main():
    print("Running main()")

if __name__ == "__main__":
    main()
```

```bash
# Прямой запуск
$ python module.py
__name__ = __main__
Running main()

# Импорт
$ python -c "import module"
__name__ = module
# main() не вызывается!
```

**Зачем нужно:**
1. **Тестирование** — код в `if __name__` не мешает импорту
2. **Библиотеки vs скрипты** — один файл может быть и тем, и другим
3. **Multiprocessing** — на Windows требуется для spawn

### Пример

```python
# calculator.py
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def main():
    """Entry point для CLI"""
    import sys
    if len(sys.argv) == 4:
        op, a, b = sys.argv[1], int(sys.argv[2]), int(sys.argv[3])
        if op == "add":
            print(add(a, b))
        elif op == "sub":
            print(subtract(a, b))
    else:
        print("Usage: python calculator.py <add|sub> <a> <b>")

if __name__ == "__main__":
    main()

# Использование:
# python calculator.py add 2 3  → 5
# from calculator import add    → Импорт без вывода

# Паттерн с argparse
import argparse

def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--verbose", "-v", action="store_true")
    return parser.parse_args()

def run(args):
    if args.verbose:
        print("Verbose mode")

if __name__ == "__main__":
    args = parse_args()
    run(args)

# Для multiprocessing (важно на Windows!)
from multiprocessing import Pool

def worker(x):
    return x ** 2

if __name__ == "__main__":
    # Без этого на Windows будет бесконечный импорт
    with Pool(4) as p:
        print(p.map(worker, range(10)))
```

### Типичные ошибки
- Забывать про `if __name__` при использовании multiprocessing
- Выполнять тяжёлую инициализацию на уровне модуля
- Путать `__name__` с `__file__`

### На интервью
**Как отвечать:** Объясните механизм (значение `__name__` при запуске vs импорте), приведите практический пример, упомяните multiprocessing на Windows.

**Follow-up вопросы:**
- Что такое `__file__`?
- Как работает `-m` флаг при запуске Python?
- Что происходит при `python -m module`?

---

[← Назад к списку тем](README.md)
