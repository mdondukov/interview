# 02. Управление памятью и GC

[← Назад к списку тем](README.md)

---

## Вопрос 1: Что такое GIL и почему он существует?

### Зачем спрашивают
GIL — самый обсуждаемый аспект CPython. Вопрос определяет понимание многопоточности в Python.

### Короткий ответ
GIL (Global Interpreter Lock) — мьютекс, который позволяет только одному потоку выполнять Python байткод в любой момент времени. Существует для защиты reference counting от race conditions. Ограничивает параллелизм CPU-bound задач, но не влияет на I/O-bound.

### Детальный разбор

**Почему GIL существует:**
1. Защита reference count от race conditions
2. Упрощение C-расширений
3. Исторические причины (легче было реализовать)

**Влияние GIL:**
```
CPU-bound:                    I/O-bound:
┌─────────────────────┐      ┌─────────────────────┐
│ Thread 1 ▓▓▓▓░░░░░  │      │ Thread 1 ▓▓░░░▓▓░░ │
│ Thread 2 ░░░░▓▓▓▓░  │      │ Thread 2 ░░▓▓░░░▓▓ │
│          ↑ GIL      │      │          ↑ Overlap  │
└─────────────────────┘      └─────────────────────┘
   Нет параллелизма          GIL освобождается при I/O
```

**Когда GIL НЕ проблема:**
- I/O-bound задачи (сеть, диск)
- Вызовы C-расширений (NumPy, Pandas)
- Multiprocessing (отдельные процессы)
- Async/await (кооперативная многозадачность)

### Пример

```python
import threading
import time

# CPU-bound - GIL мешает
def cpu_bound():
    count = 0
    for _ in range(10**7):
        count += 1
    return count

# Однопоточное выполнение
start = time.time()
cpu_bound()
cpu_bound()
print(f"Sequential: {time.time() - start:.2f}s")  # ~1.5s

# Многопоточное выполнение - НЕ быстрее из-за GIL!
start = time.time()
t1 = threading.Thread(target=cpu_bound)
t2 = threading.Thread(target=cpu_bound)
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Threaded: {time.time() - start:.2f}s")  # ~1.5s (не быстрее!)

# I/O-bound - GIL не мешает
import urllib.request

def io_bound(url):
    return urllib.request.urlopen(url).read()

# Многопоточное I/O - быстрее!
start = time.time()
threads = [
    threading.Thread(target=io_bound, args=('http://example.com',))
    for _ in range(5)
]
for t in threads: t.start()
for t in threads: t.join()
print(f"Threaded I/O: {time.time() - start:.2f}s")

# Решение для CPU-bound: multiprocessing
from multiprocessing import Pool

if __name__ == '__main__':
    start = time.time()
    with Pool(2) as p:
        p.map(cpu_bound, range(2))
    print(f"Multiprocessing: {time.time() - start:.2f}s")  # ~0.75s
```

### Типичные ошибки
- Думать, что threading ускоряет CPU-bound код
- Не знать, что GIL освобождается при I/O
- Путать GIL с общей thread-safety

### На интервью
**Как отвечать:** Объясните зачем GIL нужен (reference counting), покажите когда он мешает (CPU-bound) и когда нет (I/O), предложите альтернативы (multiprocessing, asyncio).

**Follow-up вопросы:**
- Как обойти GIL?
- Почему GIL не убрали?
- Что такое free-threading в Python 3.13?

---

## Вопрос 2: Как работает reference counting?

### Зачем спрашивают
Фундамент управления памятью в Python. Критично для понимания производительности.

### Короткий ответ
Reference counting — механизм подсчёта ссылок на объект. Каждый объект хранит счётчик (`ob_refcnt`). При создании ссылки счётчик увеличивается, при удалении — уменьшается. Когда счётчик становится 0, память освобождается немедленно.

### Детальный разбор

**Как работает:**
```
x = [1, 2, 3]    # refcount = 1
y = x            # refcount = 2
del x            # refcount = 1
del y            # refcount = 0 → память освобождена
```

**Когда счётчик увеличивается:**
- Присваивание переменной
- Добавление в контейнер
- Передача в функцию

**Когда уменьшается:**
- `del` переменной
- Переменная выходит из scope
- Переприсваивание переменной
- Удаление из контейнера

**Преимущества:**
- Детерминированное освобождение памяти
- Простота реализации
- Нет пауз GC

**Недостатки:**
- Накладные расходы на каждую операцию
- Не справляется с циклическими ссылками
- Thread-safety требует GIL

### Пример

```python
import sys

# Проверка refcount
x = [1, 2, 3]
print(sys.getrefcount(x))  # 2 (x + аргумент getrefcount)

y = x
print(sys.getrefcount(x))  # 3

del y
print(sys.getrefcount(x))  # 2

# В контейнере
container = [x]
print(sys.getrefcount(x))  # 3

# Функция увеличивает refcount временно
def process(obj):
    print(sys.getrefcount(obj))  # 4 (x + container + obj + getrefcount)

process(x)
print(sys.getrefcount(x))  # 3 (после возврата из функции)

# Немедленное освобождение (детерминированное)
class Resource:
    def __init__(self, name):
        self.name = name
        print(f"Created: {name}")

    def __del__(self):
        print(f"Destroyed: {name}")

r = Resource("test")  # Created: test
r = None              # Destroyed: test (сразу!)

# Циклическая ссылка - НЕ освобождается refcount'ом
class Node:
    def __init__(self):
        self.ref = None

a = Node()
b = Node()
a.ref = b
b.ref = a  # Цикл!
del a, b   # Память НЕ освобождена (refcount = 1 у обоих)
# Нужен GC для очистки!
```

### Типичные ошибки
- Думать, что `del` гарантированно освобождает память
- Не знать про циклические ссылки
- Полагаться на `__del__` для cleanup (лучше context manager)

### На интервью
**Как отвечать:** Объясните механизм счётчика, покажите через `sys.getrefcount()`, упомяните проблему циклических ссылок.

**Follow-up вопросы:**
- Почему getrefcount() показывает на 1 больше?
- Как Python справляется с циклическими ссылками?
- Когда вызывается `__del__`?

---

## Вопрос 3: Как работает garbage collector в Python?

### Зачем спрашивают
Дополнение к reference counting. Важно для оптимизации памяти.

### Короткий ответ
GC в Python использует generational сборку мусора для обнаружения циклических ссылок. Объекты делятся на 3 поколения. Молодые объекты проверяются чаще. GC запускается автоматически по порогам или вручную через `gc.collect()`.

### Детальный разбор

**Поколения (generations):**
```
Generation 0: Новые объекты     │ Проверяется часто
Generation 1: Пережили 1 сборку │ Проверяется реже
Generation 2: Пережили 2+ сборки│ Проверяется редко
```

**Пороги запуска:**
```python
import gc
print(gc.get_threshold())  # (700, 10, 10)
# Gen 0: после 700 аллокаций
# Gen 1: после 10 сборок Gen 0
# Gen 2: после 10 сборок Gen 1
```

**Алгоритм:**
1. Найти все контейнерные объекты (могут содержать ссылки)
2. Для каждого объекта вычислить внутренние ссылки
3. Если refcount - internal_refs = 0, объект недостижим

### Пример

```python
import gc

# Статистика GC
gc.get_count()      # (123, 5, 0) - объекты в каждом поколении
gc.get_threshold()  # (700, 10, 10) - пороги

# Принудительная сборка
gc.collect()        # Собрать все поколения
gc.collect(0)       # Только поколение 0

# Отключение GC (для performance-critical кода)
gc.disable()
# ... критический код ...
gc.enable()

# Отслеживание циклических ссылок
gc.set_debug(gc.DEBUG_SAVEALL)

class Node:
    def __init__(self, name):
        self.name = name
        self.ref = None

# Создаём цикл
a = Node('a')
b = Node('b')
a.ref = b
b.ref = a
del a, b

# Проверяем мусор
gc.collect()
print(gc.garbage)  # Список объектов, которые не удалось очистить

# Weak references для избежания циклов
import weakref

class Cache:
    def __init__(self):
        self._cache = {}

    def get(self, key):
        ref = self._cache.get(key)
        if ref is not None:
            obj = ref()
            if obj is not None:
                return obj
        return None

    def set(self, key, value):
        self._cache[key] = weakref.ref(value)

# Callbacks при сборке
def callback(phase, info):
    if phase == 'start':
        print(f"GC starting: gen {info['generation']}")
    else:
        print(f"GC done: collected {info['collected']}")

gc.callbacks.append(callback)

# Проверка на утечки памяти
import tracemalloc

tracemalloc.start()
# ... код ...
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
for stat in top_stats[:10]:
    print(stat)
```

### Типичные ошибки
- Отключать GC без понимания последствий
- Не знать про `__del__` и его влияние на GC
- Создавать циклические ссылки без weak references

### На интервью
**Как отвечать:** Объясните generational подход, покажите пороги и методы gc модуля, упомяните weak references для предотвращения циклов.

**Follow-up вопросы:**
- Когда отключать GC?
- Что такое gc.garbage?
- Как `__del__` влияет на GC?

---

## Вопрос 4: Что такое циклические ссылки и как с ними бороться?

### Зачем спрашивают
Распространённая причина утечек памяти. Проверяют понимание управления памятью.

### Короткий ответ
Циклическая ссылка — когда объекты ссылаются друг на друга (прямо или транзитивно), образуя цикл. Reference counting не может их освободить, нужен GC. Для предотвращения используют weak references или явное разрывание циклов.

### Детальный разбор

**Типы циклов:**
```python
# Прямой цикл
a.ref = b
b.ref = a

# Self-reference
a.ref = a

# Транзитивный цикл
a.ref = b
b.ref = c
c.ref = a
```

**Решения:**
1. **Weak references** — не увеличивают refcount
2. **Явное разрывание** — `obj.ref = None` перед удалением
3. **Context managers** — автоматический cleanup
4. **`__slots__`** — предотвращает `__dict__` и некоторые циклы

### Пример

```python
import weakref
import gc

# Проблема: циклическая ссылка
class Parent:
    def __init__(self):
        self.children = []

class Child:
    def __init__(self, parent):
        self.parent = parent  # Сильная ссылка на parent
        parent.children.append(self)

p = Parent()
c = Child(p)  # p -> c -> p (цикл!)
del p, c
gc.collect()  # GC очистит, но лучше избежать

# Решение 1: Weak reference
class ChildWeak:
    def __init__(self, parent):
        self.parent = weakref.ref(parent)  # Слабая ссылка
        parent.children.append(self)

    def get_parent(self):
        return self.parent()  # Может вернуть None если parent удалён

p = Parent()
c = ChildWeak(p)
print(c.get_parent())  # <Parent object>
del p
print(c.get_parent())  # None (parent удалён)

# Решение 2: WeakValueDictionary
cache = weakref.WeakValueDictionary()

class ExpensiveObject:
    def __init__(self, key):
        self.key = key
        cache[key] = self  # Weak reference в кэше

obj = ExpensiveObject('data')
print(cache.get('data'))  # <ExpensiveObject>
del obj
print(cache.get('data'))  # None (автоматически удалено из кэша)

# Решение 3: Явное разрывание (Context Manager)
class Connection:
    def __init__(self):
        self.callbacks = []

    def add_callback(self, callback):
        self.callbacks.append(callback)
        # callback может ссылаться на Connection -> цикл

    def close(self):
        self.callbacks.clear()  # Явно разрываем

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.close()

# Использование
with Connection() as conn:
    conn.add_callback(lambda: print(conn))  # Цикл внутри
# После выхода callbacks очищены

# Обнаружение циклов
gc.set_debug(gc.DEBUG_SAVEALL)
# После gc.collect() проверяем gc.garbage

# Finalization с weakref.finalize
def cleanup(name):
    print(f"Cleaning up {name}")

class Resource:
    def __init__(self, name):
        self.name = name
        self._finalizer = weakref.finalize(self, cleanup, name)

r = Resource("test")
del r  # "Cleaning up test" - гарантированный cleanup
```

### Типичные ошибки
- Игнорировать циклические ссылки
- Не использовать weak references для кэшей
- Полагаться на `__del__` для cleanup

### На интервью
**Как отвечать:** Объясните проблему с refcount, покажите weak references как решение, упомяните WeakValueDictionary для кэшей.

**Follow-up вопросы:**
- Когда нужен WeakKeyDictionary vs WeakValueDictionary?
- Как работает weakref.finalize?
- Можно ли создать weak reference на int/str?

---

## Вопрос 5: В чём разница между `is` и `==`?

### Зачем спрашивают
Базовый вопрос, но часто путают. Проверяют понимание identity vs equality.

### Короткий ответ
`is` проверяет identity (один и тот же объект в памяти, `id(a) == id(b)`). `==` проверяет equality (равенство значений, вызывает `__eq__`). Для `None` всегда используйте `is None`.

### Детальный разбор

**Правило:**
```python
a is b  # True если id(a) == id(b)
a == b  # True если a.__eq__(b) возвращает True
```

**Когда использовать `is`:**
- Сравнение с `None`
- Сравнение синглтонов
- Проверка идентичности объекта

**Когда использовать `==`:**
- Сравнение значений
- Почти всегда для данных

**Особенности CPython:**
- Интернирование маленьких целых чисел (-5 до 256)
- Интернирование коротких строк
- Эти объекты кэшируются и переиспользуются

### Пример

```python
# Базовое различие
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True (равные значения)
print(a is b)  # False (разные объекты)
print(a is c)  # True (один объект)
print(id(a), id(b), id(c))  # id(a) == id(c) != id(b)

# None - всегда используйте is
x = None
if x is None:      # Правильно
    pass
if x == None:      # Работает, но не pythonic
    pass

# Integer interning
a = 256
b = 256
print(a is b)  # True (интернированы)

a = 257
b = 257
print(a is b)  # False (не интернированы)*
# *Может быть True в интерактивном режиме!

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True (интернированы)

s1 = "hello world!"
s2 = "hello world!"
print(s1 is s2)  # False (пробел и ! блокируют interning)

# Принудительное интернирование
import sys
s1 = sys.intern("hello world!")
s2 = sys.intern("hello world!")
print(s1 is s2)  # True

# Кастомный __eq__
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1 == p2)  # True (равные координаты)
print(p1 is p2)  # False (разные объекты)

# Singleton pattern
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True (один и тот же объект)
```

### Типичные ошибки
- Использовать `== None` вместо `is None`
- Полагаться на integer interning
- Путать identity и equality для коллекций

### На интервью
**Как отвечать:** Объясните identity vs equality, покажите `id()`, упомяните interning и правило `is None`.

**Follow-up вопросы:**
- Почему `is` быстрее `==`?
- Что такое string interning?
- Когда `a is b` может дать неожиданный результат?

---

## Вопрос 6: Что такое interning и string interning?

### Зачем спрашивают
Оптимизация CPython. Важно для понимания поведения `is` и производительности.

### Короткий ответ
Interning — техника переиспользования immutable объектов. CPython автоматически интернирует маленькие числа (-5 до 256) и некоторые строки. Экономит память и ускоряет сравнение через `is`. Можно принудительно интернировать через `sys.intern()`.

### Детальный разбор

**Что интернируется автоматически:**
- Integers от -5 до 256
- Строки, похожие на идентификаторы (буквы, цифры, _)
- Строки из байткода (константы)
- Имена переменных и атрибутов

**Почему это полезно:**
1. **Экономия памяти** — один объект на все ссылки
2. **Быстрое сравнение** — `is` вместо `==`
3. **Использование как ключи dict** — быстрее хэширование

### Пример

```python
import sys

# Integer interning
a = 100
b = 100
print(a is b)  # True

a = 1000
b = 1000
print(a is b)  # False (вне диапазона)

# Демонстрация диапазона
for i in range(-10, 260):
    a = i
    b = i
    if a is not b:
        print(f"First non-interned: {i}")  # 257
        break

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True (автоматически интернирована)

s1 = "hello world"
s2 = "hello world"
print(s1 is s2)  # Может быть True или False!

# Принудительное интернирование
s1 = sys.intern("hello world")
s2 = sys.intern("hello world")
print(s1 is s2)  # True (гарантированно)

# Когда интернировать строки
# 1. Часто сравниваемые строки
STATUS_ACTIVE = sys.intern("active")
STATUS_INACTIVE = sys.intern("inactive")

def check_status(status):
    status = sys.intern(status)  # Интернируем входные данные
    if status is STATUS_ACTIVE:  # Быстрое сравнение
        return True
    return False

# 2. Ключи словаря из пользовательского ввода
def process_json(data):
    # Интернируем ключи для экономии памяти
    return {sys.intern(k): v for k, v in data.items()}

# Демонстрация экономии памяти
strings = ["common_key"] * 1000000

# Без interning
import sys
total_size = sum(sys.getsizeof(s) for s in strings)
print(f"Without interning: {total_size:,} bytes")

# С interning (все ссылаются на один объект)
interned = [sys.intern("common_key") for _ in range(1000000)]
unique_objects = len(set(id(s) for s in interned))
print(f"Unique objects: {unique_objects}")  # 1

# Tuple interning (Python 3.10+)
t1 = (1, 2, 3)
t2 = (1, 2, 3)
print(t1 is t2)  # True (короткие tuples интернируются)
```

### Типичные ошибки
- Полагаться на interning для сравнения (использовать `==`)
- Не знать про диапазон integer interning
- Забывать про `sys.intern()` для оптимизации

### На интервью
**Как отвечать:** Объясните что интернируется (числа -5..256, строки-идентификаторы), покажите `sys.intern()`, упомяните use cases (ключи словарей, статусы).

**Follow-up вопросы:**
- Почему диапазон именно -5 до 256?
- Когда использовать sys.intern()?
- Можно ли интернировать свои объекты?

---

## Вопрос 7: Как работает копирование объектов? (copy vs deepcopy)

### Зачем спрашивают
Распространённый источник багов. Проверяют понимание mutable/immutable и ссылок.

### Короткий ответ
`copy.copy()` создаёт shallow copy — новый контейнер со ссылками на те же объекты. `copy.deepcopy()` создаёт deep copy — рекурсивно копирует все вложенные объекты. Для immutable объектов оба возвращают тот же объект.

### Детальный разбор

**Типы копирования:**
```
Original: [1, [2, 3]]
           ↓    ↓
         int  list

Shallow:  [1, [2, 3]]   # Новый list, но [2,3] - та же ссылка
           ↓    ↓
         int  list (тот же!)

Deep:     [1, [2, 3]]   # Всё новое
           ↓    ↓
         int  list (новый!)
```

**Когда использовать:**
- **Assignment** (`b = a`) — только новая ссылка
- **Shallow copy** — независимый контейнер, зависимые элементы
- **Deep copy** — полностью независимая копия

### Пример

```python
import copy

# Assignment vs copy
original = [[1, 2], [3, 4]]

# Assignment - просто ссылка
reference = original
reference[0][0] = 999
print(original)  # [[999, 2], [3, 4]] - изменился!

# Shallow copy
original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
# или: shallow = original[:]
# или: shallow = list(original)

shallow.append([5, 6])
print(original)  # [[1, 2], [3, 4]] - не изменился

shallow[0][0] = 999
print(original)  # [[999, 2], [3, 4]] - изменился! (общие элементы)

# Deep copy
original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)

deep[0][0] = 999
print(original)  # [[1, 2], [3, 4]] - не изменился!

# Кастомное копирование
class Node:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []

    def __copy__(self):
        # Shallow: новый Node, но те же children
        return Node(self.value, self.children)

    def __deepcopy__(self, memo):
        # Deep: рекурсивно копируем всё
        return Node(
            copy.deepcopy(self.value, memo),
            copy.deepcopy(self.children, memo)
        )

# Memo словарь предотвращает бесконечную рекурсию
# при циклических ссылках

# Оптимизация для immutable
original = (1, 2, 3)
shallow = copy.copy(original)
deep = copy.deepcopy(original)

print(original is shallow)  # True (tuple immutable)
print(original is deep)     # True (нет смысла копировать)

# Частичное копирование
class Config:
    def __init__(self, settings, cache):
        self.settings = settings  # Копировать
        self.cache = cache        # Не копировать (shared)

    def __deepcopy__(self, memo):
        return Config(
            copy.deepcopy(self.settings, memo),
            self.cache  # Намеренно shared
        )

# Dict и list методы копирования
d = {'a': [1, 2]}
d_copy = d.copy()  # Shallow copy
d_copy = dict(d)   # Shallow copy
d_copy = {**d}     # Shallow copy (Python 3.5+)

lst = [1, [2, 3]]
lst_copy = lst.copy()  # Shallow
lst_copy = lst[:]      # Shallow
lst_copy = list(lst)   # Shallow
```

### Типичные ошибки
- Думать, что `list.copy()` делает deep copy
- Использовать `=` вместо `copy()` для контейнеров
- Забывать про memo в `__deepcopy__`

### На интервью
**Как отвечать:** Нарисуйте диаграмму с ссылками, покажите разницу на примере вложенных списков, упомяните кастомизацию через `__copy__`/`__deepcopy__`.

**Follow-up вопросы:**
- Как копировать объекты с циклическими ссылками?
- Для чего нужен memo в deepcopy?
- Когда deepcopy не работает?

---

## Вопрос 8: Как профилировать использование памяти?

### Зачем спрашивают
Практический навык для оптимизации и поиска утечек. Показывает senior-level опыт.

### Короткий ответ
Для профилирования памяти используют: `sys.getsizeof()` для размера объекта, `tracemalloc` для трассировки аллокаций, `memory_profiler` для построчного анализа, `objgraph` для поиска утечек. `gc` модуль показывает информацию о сборке мусора.

### Детальный разбор

**Инструменты:**

| Инструмент | Назначение |
|------------|------------|
| `sys.getsizeof()` | Размер одного объекта |
| `tracemalloc` | Трассировка аллокаций (stdlib) |
| `memory_profiler` | Построчное профилирование |
| `objgraph` | Визуализация объектов |
| `pympler` | Детальный анализ памяти |

### Пример

```python
import sys

# sys.getsizeof - размер объекта (без вложенных!)
print(sys.getsizeof([]))        # 56 bytes (пустой list)
print(sys.getsizeof([1, 2, 3])) # 88 bytes
print(sys.getsizeof({}))        # 64 bytes
print(sys.getsizeof("hello"))   # 54 bytes

# Реальный размер со вложенными объектами
def deep_getsizeof(obj, seen=None):
    if seen is None:
        seen = set()
    obj_id = id(obj)
    if obj_id in seen:
        return 0
    seen.add(obj_id)
    size = sys.getsizeof(obj)
    if isinstance(obj, dict):
        size += sum(deep_getsizeof(k, seen) + deep_getsizeof(v, seen)
                    for k, v in obj.items())
    elif isinstance(obj, (list, tuple, set, frozenset)):
        size += sum(deep_getsizeof(i, seen) for i in obj)
    return size

# tracemalloc - трассировка аллокаций
import tracemalloc

tracemalloc.start()

# Ваш код
data = [list(range(1000)) for _ in range(1000)]

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

print("Top 10 allocations:")
for stat in top_stats[:10]:
    print(stat)

# Сравнение снимков
tracemalloc.start()
snapshot1 = tracemalloc.take_snapshot()

# ... код, который может утекать ...
leaky_data = []
for i in range(10000):
    leaky_data.append(str(i) * 100)

snapshot2 = tracemalloc.take_snapshot()
top_stats = snapshot2.compare_to(snapshot1, 'lineno')

print("\nMemory diff:")
for stat in top_stats[:10]:
    print(stat)

# memory_profiler (pip install memory-profiler)
# Использование:
# @profile
# def my_function():
#     ...
# Запуск: python -m memory_profiler script.py

# Пример с декоратором
from memory_profiler import profile

@profile
def memory_test():
    a = [1] * 1000000
    b = [2] * 2000000
    del b
    return a

# objgraph (pip install objgraph)
# import objgraph
# objgraph.show_most_common_types(limit=10)
# objgraph.show_growth()
# objgraph.show_backrefs([obj], filename='refs.png')

# gc статистика
import gc

gc.collect()
print(f"Objects tracked: {len(gc.get_objects())}")
print(f"GC stats: {gc.get_stats()}")

# Пример поиска утечки
class Cache:
    _instances = []  # Утечка: все инстансы хранятся!

    def __init__(self, data):
        self.data = data
        Cache._instances.append(self)

# После создания 1000 объектов Cache - все в памяти
# Решение: WeakSet или явная очистка

# pympler для детального анализа
# from pympler import asizeof, tracker
# print(asizeof.asizeof(my_object))  # Реальный размер с вложенными

# tr = tracker.SummaryTracker()
# ... код ...
# tr.print_diff()  # Показывает изменения
```

### Типичные ошибки
- Использовать только `getsizeof()` (не видит вложенные)
- Не использовать `tracemalloc` для поиска утечек
- Игнорировать GC статистику

### На интервью
**Как отвечать:** Начните с `tracemalloc` как stdlib инструмента, покажите сравнение снимков, упомяните `memory_profiler` и `objgraph` для сложных случаев.

**Follow-up вопросы:**
- Как найти утечку памяти в production?
- Чем отличается RSS от heap size?
- Как ограничить использование памяти?

---

[← Назад к списку тем](README.md)
