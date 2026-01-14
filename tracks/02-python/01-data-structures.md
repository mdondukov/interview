# 01. Структуры данных Python

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как устроен list внутри?

### Зачем спрашивают
Проверяют понимание производительности операций и выбора правильной структуры данных.

### Короткий ответ
List в Python — динамический массив (не связный список). Хранит указатели на объекты в непрерывном блоке памяти. При переполнении выделяет новый блок большего размера и копирует ссылки. Амортизированная сложность append — O(1).

### Детальный разбор

**Внутренняя структура:**
```
PyListObject:
├── ob_refcnt     (reference count)
├── ob_type       (type pointer)
├── ob_size       (текущее количество элементов)
├── allocated     (выделенная ёмкость)
└── ob_item       (указатель на массив указателей)
        ↓
    [ptr0][ptr1][ptr2][...][пустые слоты]
```

**Сложность операций:**

| Операция | Средняя | Худшая |
|----------|---------|--------|
| `append(x)` | O(1)* | O(n) |
| `pop()` | O(1) | O(1) |
| `pop(0)` | O(n) | O(n) |
| `insert(i, x)` | O(n) | O(n) |
| `x in lst` | O(n) | O(n) |
| `lst[i]` | O(1) | O(1) |
| `del lst[i]` | O(n) | O(n) |
| `len(lst)` | O(1) | O(1) |

*амортизированная

**Стратегия роста:**
```
Формула: new_size = old_size + (old_size >> 3) + 6
Примерно: 0, 4, 8, 16, 25, 35, 46, 58, 72, 88...
```

### Пример

```python
import sys

# Демонстрация роста памяти
lst = []
prev_size = 0
for i in range(20):
    lst.append(i)
    size = sys.getsizeof(lst)
    if size != prev_size:
        print(f"len={len(lst):2}, size={size}, allocated≈{(size-56)//8}")
        prev_size = size

# len= 1, size=88,  allocated≈4
# len= 5, size=120, allocated≈8
# len= 9, size=184, allocated≈16
# len=17, size=248, allocated≈24

# Эффективное использование
# Плохо - много insert(0, x)
def build_reversed_bad(n):
    result = []
    for i in range(n):
        result.insert(0, i)  # O(n) каждый раз!
    return result

# Хорошо - append и reverse
def build_reversed_good(n):
    result = []
    for i in range(n):
        result.append(i)  # O(1) амортизированно
    result.reverse()      # O(n) один раз
    return result

# Или используйте deque для операций с двух сторон
from collections import deque
d = deque()
d.appendleft(1)  # O(1)
```

### Типичные ошибки
- Думать, что list — это связный список
- Использовать `insert(0, x)` в цикле (O(n²))
- Не знать про амортизированную сложность

### На интервью
**Как отвечать:** Объясните, что это динамический массив указателей, опишите стратегию роста и амортизированную сложность append. Покажите когда лучше использовать deque.

**Follow-up вопросы:**
- Почему append амортизированно O(1)?
- Когда использовать list vs deque?
- Как работает slice?

---

## Вопрос 2: Как устроен dict внутри?

### Зачем спрашивают
Dict — основная структура данных Python. Понимание хэш-таблиц критично для оптимизации.

### Короткий ответ
Dict — хэш-таблица с открытой адресацией. Хранит хэш, ключ и значение. При коллизиях использует пробирование. С Python 3.7 гарантирует порядок вставки. Средняя сложность операций — O(1).

### Детальный разбор

**Структура (Python 3.6+):**
```
Compact dict:
┌─────────────────────┐
│ Sparse indices      │  [None, 0, None, 1, 2, None, ...]
├─────────────────────┤
│ Dense entries       │  [(hash0, key0, val0),
│                     │   (hash1, key1, val1),
│                     │   (hash2, key2, val2)]
└─────────────────────┘
```

**Сложность операций:**

| Операция | Средняя | Худшая |
|----------|---------|--------|
| `d[key]` | O(1) | O(n) |
| `d[key] = val` | O(1) | O(n) |
| `del d[key]` | O(1) | O(n) |
| `key in d` | O(1) | O(n) |
| `len(d)` | O(1) | O(1) |

**Требования к ключам:**
1. Hashable (реализован `__hash__`)
2. Immutable (или корректно реализован hash)
3. Корректная реализация `__eq__`

**Правило:** `a == b` → `hash(a) == hash(b)`
(Обратное не обязательно — коллизии допустимы)

### Пример

```python
# Хэширование
print(hash("hello"))  # -1267296259540920282 (может отличаться)
print(hash((1, 2, 3)))  # Tuples hashable
# print(hash([1, 2, 3]))  # TypeError: unhashable type: 'list'

# Кастомный hashable объект
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __hash__(self):
        return hash((self.x, self.y))

    def __eq__(self, other):
        return isinstance(other, Point) and self.x == other.x and self.y == other.y

points = {Point(0, 0): "origin", Point(1, 1): "diagonal"}

# Порядок сохраняется (Python 3.7+)
d = {}
d['first'] = 1
d['second'] = 2
d['third'] = 3
print(list(d.keys()))  # ['first', 'second', 'third']

# Демонстрация коллизий
class BadHash:
    def __init__(self, value):
        self.value = value
    def __hash__(self):
        return 1  # Все объекты имеют одинаковый хэш!
    def __eq__(self, other):
        return self.value == other.value

# Работает, но очень медленно
bad_dict = {BadHash(i): i for i in range(1000)}  # O(n²)!

# Dict comprehension
squares = {x: x**2 for x in range(10)}

# Объединение словарей (Python 3.9+)
d1 = {"a": 1}
d2 = {"b": 2}
merged = d1 | d2  # {"a": 1, "b": 2}
d1 |= d2  # in-place update
```

### Типичные ошибки
- Использовать mutable объекты как ключи (list, dict)
- Не реализовать `__eq__` при переопределении `__hash__`
- Менять объект после использования как ключа

### На интервью
**Как отвечать:** Опишите хэш-таблицу с открытой адресацией, упомяните compact dict (3.6+), объясните требования к ключам и гарантию порядка (3.7+).

**Follow-up вопросы:**
- Что будет, если изменить mutable ключ?
- Как работает dict comprehension?
- Чем отличается dict от OrderedDict теперь?

---

## Вопрос 3: Как устроен set и чем отличается от dict?

### Зачем спрашивают
Set — важная структура для алгоритмических задач. Проверяют понимание когда использовать.

### Короткий ответ
Set — хэш-таблица без значений (только ключи). Использует тот же механизм, что и dict. Гарантирует уникальность элементов и O(1) проверку принадлежности. Элементы должны быть hashable.

### Детальный разбор

**Сложность операций:**

| Операция | Средняя | Худшая |
|----------|---------|--------|
| `x in s` | O(1) | O(n) |
| `s.add(x)` | O(1) | O(n) |
| `s.remove(x)` | O(1) | O(n) |
| `s \| t` (union) | O(len(s)+len(t)) | — |
| `s & t` (intersection) | O(min(len(s), len(t))) | — |
| `s - t` (difference) | O(len(s)) | — |

**Операции над множествами:**
```python
s1 | s2   # union (объединение)
s1 & s2   # intersection (пересечение)
s1 - s2   # difference (разность)
s1 ^ s2   # symmetric_difference (симметричная разность)
s1 <= s2  # issubset (подмножество)
s1 >= s2  # issuperset (надмножество)
```

### Пример

```python
# Создание
s1 = {1, 2, 3}
s2 = set([1, 2, 3])  # Из list
s3 = set()  # Пустой set (не {}, это dict!)

# Проверка принадлежности - O(1) vs O(n) для list
large_list = list(range(1000000))
large_set = set(range(1000000))

import timeit
print(timeit.timeit('999999 in large_list', globals=globals(), number=100))  # ~1.5s
print(timeit.timeit('999999 in large_set', globals=globals(), number=100))   # ~0.00001s

# Операции над множествами
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

print(a | b)   # {1, 2, 3, 4, 5, 6}
print(a & b)   # {3, 4}
print(a - b)   # {1, 2}
print(a ^ b)   # {1, 2, 5, 6}

# Практические применения
# 1. Удаление дубликатов
items = [1, 2, 2, 3, 3, 3]
unique = list(set(items))  # [1, 2, 3] (порядок не гарантирован!)

# С сохранением порядка (Python 3.7+)
unique_ordered = list(dict.fromkeys(items))  # [1, 2, 2, 3]

# 2. Быстрая проверка принадлежности
valid_users = {"alice", "bob", "charlie"}
if username in valid_users:
    allow_access()

# 3. Поиск общих элементов
def common_elements(list1, list2):
    return set(list1) & set(list2)

# Set comprehension
squares = {x**2 for x in range(-5, 6)}  # {0, 1, 4, 9, 16, 25}
```

### Типичные ошибки
- Использовать `{}` для пустого set (это dict!)
- Забывать, что set не сохраняет порядок
- Использовать list вместо set для проверки принадлежности

### На интервью
**Как отвечать:** Объясните, что set — это dict без значений, покажите O(1) проверку принадлежности, перечислите операции над множествами.

**Follow-up вопросы:**
- Как удалить дубликаты с сохранением порядка?
- Что такое frozenset?
- Когда set медленнее list?

---

## Вопрос 4: Что такое tuple и в чём его преимущества?

### Зачем спрашивают
Tuple — фундаментальная структура. Проверяют понимание immutability и оптимизаций.

### Короткий ответ
Tuple — неизменяемая последовательность. Быстрее list, занимает меньше памяти, может быть ключом dict/элементом set. Python кэширует маленькие tuples. Используется для heterogeneous данных (разнотипные элементы).

### Детальный разбор

**Преимущества tuple над list:**
1. **Меньше памяти** — нет over-allocation
2. **Быстрее создание** — Python кэширует маленькие tuples
3. **Hashable** — можно использовать как ключ dict
4. **Безопасность** — случайно не изменишь
5. **Семантика** — "запись" vs "коллекция"

**Сравнение памяти:**
```python
import sys
lst = [1, 2, 3]
tpl = (1, 2, 3)
print(sys.getsizeof(lst))  # 88 bytes
print(sys.getsizeof(tpl))  # 64 bytes
```

**Named tuples:**
```python
from collections import namedtuple
Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)
print(p.x, p.y)  # 1 2
```

### Пример

```python
# Создание
t1 = (1, 2, 3)
t2 = 1, 2, 3      # Без скобок тоже работает
t3 = (1,)         # Один элемент - нужна запятая!
t4 = tuple([1, 2, 3])  # Из list

# Распаковка
x, y, z = t1
first, *rest = (1, 2, 3, 4)  # first=1, rest=[2, 3, 4]

# Как ключ dict
points = {(0, 0): "origin", (1, 1): "diagonal"}

# Named tuple
from collections import namedtuple
User = namedtuple('User', ['name', 'age', 'email'])
user = User('Alice', 30, 'alice@example.com')
print(user.name)  # Alice
print(user[0])    # Alice (индекс тоже работает)

# typing.NamedTuple (более современный способ)
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    label: str = ""  # default value

p = Point(1.0, 2.0, "A")

# Tuple immutability (но не deep!)
t = ([1, 2], [3, 4])
# t[0] = [5, 6]  # TypeError!
t[0].append(3)   # Работает! t = ([1, 2, 3], [3, 4])

# Конкатенация создаёт новый tuple
t1 = (1, 2)
t2 = t1 + (3, 4)  # Новый tuple (1, 2, 3, 4)

# Сравнение производительности
import timeit
print(timeit.timeit('(1, 2, 3, 4, 5)', number=10000000))  # ~0.15s
print(timeit.timeit('[1, 2, 3, 4, 5]', number=10000000))  # ~0.35s
```

### Типичные ошибки
- Забывать запятую для single-element tuple: `(1)` vs `(1,)`
- Думать, что tuple полностью immutable (содержимое mutable объектов можно менять)
- Не использовать named tuples для читаемости

### На интервью
**Как отвечать:** Перечислите преимущества (память, скорость, hashable), объясните shallow immutability, упомяните named tuples.

**Follow-up вопросы:**
- Чем NamedTuple отличается от dataclass?
- Почему tuple быстрее создаётся?
- Когда лучше использовать dataclass вместо NamedTuple?

---

## Вопрос 5: Что такое frozenset и когда его использовать?

### Зачем спрашивают
Проверяют понимание immutable структур и их применения.

### Короткий ответ
Frozenset — неизменяемая версия set. Hashable, поэтому может быть ключом dict или элементом другого set. Используется когда нужно неизменяемое множество или ключ из множества элементов.

### Детальный разбор

**Отличия от set:**
| Аспект | set | frozenset |
|--------|-----|-----------|
| Mutable | Да | Нет |
| Hashable | Нет | Да |
| Методы add/remove | Да | Нет |
| Операции (&, \|, -) | Да | Да |
| Как ключ dict | Нет | Да |

**Когда использовать:**
1. Ключ словаря из множества элементов
2. Элемент другого set (set of sets)
3. Кэширование множеств
4. Immutable конфигурация

### Пример

```python
# Создание
fs1 = frozenset([1, 2, 3])
fs2 = frozenset({1, 2, 3})

# Как ключ dict
permissions = {
    frozenset({'read', 'write'}): 'editor',
    frozenset({'read'}): 'viewer',
    frozenset({'read', 'write', 'delete'}): 'admin',
}

user_perms = frozenset({'read', 'write'})
role = permissions[user_perms]  # 'editor'

# Set of sets (невозможно с обычным set)
# {{'a', 'b'}, {'c', 'd'}}  # TypeError!
{frozenset({'a', 'b'}), frozenset({'c', 'd'})}  # OK

# Операции работают так же
fs1 = frozenset({1, 2, 3})
fs2 = frozenset({2, 3, 4})
print(fs1 | fs2)  # frozenset({1, 2, 3, 4})
print(fs1 & fs2)  # frozenset({2, 3})

# Кэширование с frozenset
from functools import lru_cache

@lru_cache(maxsize=100)
def process_items(items: frozenset) -> int:
    # items должен быть hashable для кэширования
    return sum(items)

# Вызов с frozenset
result = process_items(frozenset({1, 2, 3}))

# Конвертация
s = {1, 2, 3}
fs = frozenset(s)  # set → frozenset
s2 = set(fs)       # frozenset → set
```

### Типичные ошибки
- Пытаться модифицировать frozenset
- Не использовать когда нужен hashable set
- Забывать про frozenset при работе с кэшированием

### На интервью
**Как отвечать:** Объясните как immutable версию set, покажите использование как ключа dict, упомяните применение с lru_cache.

**Follow-up вопросы:**
- Как создать set of sets?
- Когда frozenset лучше tuple?
- Как frozenset влияет на производительность?

---

## Вопрос 6: Как работает collections.deque?

### Зачем спрашивают
Deque — важная структура для алгоритмов и реальных задач (очереди, sliding window).

### Короткий ответ
Deque (double-ended queue) — двусторонняя очередь с O(1) операциями на обоих концах. Реализована как двусвязный список блоков. Поддерживает maxlen для fixed-size буфера. Идеальна для очередей и стеков.

### Детальный разбор

**Сложность операций:**

| Операция | list | deque |
|----------|------|-------|
| `append(x)` | O(1)* | O(1) |
| `appendleft(x)` | O(n) | O(1) |
| `pop()` | O(1) | O(1) |
| `popleft()` | O(n) | O(1) |
| `x[i]` | O(1) | O(n) |

**Внутренняя структура:**
```
Двусвязный список блоков (по 64 элемента):
[Block] ↔ [Block] ↔ [Block]
   ↑                   ↑
  head                tail
```

**Полезные методы:**
- `rotate(n)` — циклический сдвиг
- `maxlen` — фиксированный размер (для буферов)

### Пример

```python
from collections import deque

# Создание
d = deque([1, 2, 3])
d = deque(maxlen=3)  # Fixed-size buffer

# Операции с обоих концов
d = deque([2, 3])
d.appendleft(1)  # deque([1, 2, 3])
d.append(4)      # deque([1, 2, 3, 4])
d.popleft()      # 1, deque([2, 3, 4])
d.pop()          # 4, deque([2, 3])

# Как очередь (FIFO)
queue = deque()
queue.append("task1")    # enqueue
queue.append("task2")
task = queue.popleft()   # dequeue - "task1"

# Как стек (LIFO) - но list тоже подходит
stack = deque()
stack.append("item1")    # push
stack.append("item2")
item = stack.pop()       # pop - "item2"

# Fixed-size buffer (circular buffer)
history = deque(maxlen=5)
for i in range(10):
    history.append(i)
print(list(history))  # [5, 6, 7, 8, 9]

# Rotate
d = deque([1, 2, 3, 4, 5])
d.rotate(2)   # deque([4, 5, 1, 2, 3])
d.rotate(-2)  # deque([1, 2, 3, 4, 5])

# Sliding window
def sliding_window_max(nums, k):
    """Находит максимум в каждом окне размера k"""
    result = []
    window = deque()  # Хранит индексы

    for i, num in enumerate(nums):
        # Удаляем элементы вне окна
        while window and window[0] <= i - k:
            window.popleft()

        # Удаляем меньшие элементы
        while window and nums[window[-1]] < num:
            window.pop()

        window.append(i)

        if i >= k - 1:
            result.append(nums[window[0]])

    return result

print(sliding_window_max([1, 3, -1, -3, 5, 3, 6, 7], 3))
# [3, 3, 5, 5, 6, 7]
```

### Типичные ошибки
- Использовать list для операций с начала
- Забывать про maxlen для буферов
- Индексация deque в цикле (O(n) каждый раз)

### На интервью
**Как отвечать:** Объясните O(1) операции с обоих концов, покажите использование для очереди и sliding window, упомяните maxlen.

**Follow-up вопросы:**
- Почему индексация O(n)?
- Как реализовать circular buffer?
- Когда использовать deque vs list?

---

## Вопрос 7: Что такое defaultdict, Counter, OrderedDict?

### Зачем спрашивают
Collections — важный модуль для повседневной работы. Проверяют практические знания.

### Короткий ответ
`defaultdict` — dict с default factory для отсутствующих ключей. `Counter` — dict для подсчёта элементов с полезными методами. `OrderedDict` — dict с гарантированным порядком (теперь менее актуален после Python 3.7).

### Детальный разбор

**defaultdict:**
```python
from collections import defaultdict

# Вместо проверки if key in dict
d = defaultdict(list)
d['key'].append(1)  # Автоматически создаёт list

# Популярные фабрики:
defaultdict(list)    # Группировка
defaultdict(int)     # Счётчики (0 по умолчанию)
defaultdict(set)     # Уникальные значения
defaultdict(lambda: 'default')  # Кастомное значение
```

**Counter:**
```python
from collections import Counter

# Подсчёт элементов
c = Counter(['a', 'b', 'a', 'c', 'a', 'b'])
# Counter({'a': 3, 'b': 2, 'c': 1})

# Методы
c.most_common(2)  # [('a', 3), ('b', 2)]
c.total()         # 6 (Python 3.10+)
```

**OrderedDict:**
```python
from collections import OrderedDict

# До Python 3.7 - единственный способ сохранить порядок
od = OrderedDict()
od['a'] = 1
od['b'] = 2

# Уникальные методы
od.move_to_end('a')  # Переместить в конец
od.move_to_end('b', last=False)  # В начало
```

### Пример

```python
from collections import defaultdict, Counter, OrderedDict

# defaultdict - группировка
students = [
    ('Alice', 'Math'),
    ('Bob', 'Physics'),
    ('Alice', 'Physics'),
    ('Bob', 'Math'),
]

by_student = defaultdict(list)
for name, subject in students:
    by_student[name].append(subject)
# defaultdict(<class 'list'>, {'Alice': ['Math', 'Physics'], 'Bob': ['Physics', 'Math']})

# defaultdict - вложенные структуры
tree = lambda: defaultdict(tree)
taxonomy = tree()
taxonomy['Animal']['Mammal']['Dog'] = 'Canis lupus'
taxonomy['Animal']['Bird']['Eagle'] = 'Aquila chrysaetos'

# Counter - подсчёт слов
text = "to be or not to be"
word_count = Counter(text.split())
# Counter({'to': 2, 'be': 2, 'or': 1, 'not': 1})

# Counter - арифметика
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
print(c1 + c2)  # Counter({'a': 4, 'b': 3})
print(c1 - c2)  # Counter({'a': 2}) - отрицательные убираются

# Counter - топ N элементов
words = Counter(open('text.txt').read().split())
top_10 = words.most_common(10)

# OrderedDict - LRU cache (упрощённый)
class LRU(OrderedDict):
    def __init__(self, maxsize=128):
        super().__init__()
        self.maxsize = maxsize

    def __getitem__(self, key):
        value = super().__getitem__(key)
        self.move_to_end(key)
        return value

    def __setitem__(self, key, value):
        if key in self:
            self.move_to_end(key)
        super().__setitem__(key, value)
        if len(self) > self.maxsize:
            self.popitem(last=False)
```

### Типичные ошибки
- Использовать обычный dict с проверками вместо defaultdict
- Не знать про most_common() в Counter
- Использовать OrderedDict в Python 3.7+ без необходимости

### На интервью
**Как отвечать:** Покажите практические примеры: группировка с defaultdict, подсчёт с Counter. Объясните, что OrderedDict теперь нужен редко.

**Follow-up вопросы:**
- Как реализовать LRU cache с OrderedDict?
- Чем Counter отличается от defaultdict(int)?
- Что делает Counter.elements()?

---

## Вопрос 8: Как реализовать LRU cache?

### Зачем спрашивают
Классическая задача на структуры данных. Проверяют понимание dict + двусвязный список.

### Короткий ответ
LRU (Least Recently Used) cache можно реализовать через `functools.lru_cache` декоратор или вручную через OrderedDict/dict + двусвязный список для O(1) операций get/put.

### Детальный разбор

**Требования:**
- `get(key)` — O(1)
- `put(key, value)` — O(1)
- При переполнении удаляем наименее недавно использованный элемент

**Варианты реализации:**
1. `functools.lru_cache` — для мемоизации функций
2. `OrderedDict` — простая реализация
3. `dict + doubly linked list` — классическая LeetCode реализация

### Пример

```python
from functools import lru_cache

# 1. Декоратор lru_cache
@lru_cache(maxsize=100)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Статистика
print(fibonacci.cache_info())
# CacheInfo(hits=28, misses=31, maxsize=100, currsize=31)
fibonacci.cache_clear()  # Очистить кэш

# 2. OrderedDict реализация
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)

# 3. Dict + Doubly Linked List (классика)
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCacheManual:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        # Dummy head и tail для упрощения
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_end(self, node):
        node.prev = self.tail.prev
        node.next = self.tail
        self.tail.prev.next = node
        self.tail.prev = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_end(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._remove(node)
            self._add_to_end(node)
        else:
            if len(self.cache) >= self.capacity:
                # Удаляем LRU (после head)
                lru = self.head.next
                self._remove(lru)
                del self.cache[lru.key]

            node = Node(key, value)
            self.cache[key] = node
            self._add_to_end(node)

# Использование
cache = LRUCache(2)
cache.put(1, 1)
cache.put(2, 2)
print(cache.get(1))    # 1
cache.put(3, 3)        # Удаляет ключ 2
print(cache.get(2))    # -1
```

### Типичные ошибки
- Забывать обновлять позицию при get
- Неправильная работа с dummy nodes
- O(n) операции вместо O(1)

### На интервью
**Как отвечать:** Начните с `lru_cache` для практических задач. Для алгоритмической — OrderedDict или dict + linked list.

**Follow-up вопросы:**
- Как сделать thread-safe LRU cache?
- Что такое LFU cache?
- Как lru_cache работает внутри?

---

## Вопрос 9: Что такое heapq и когда его использовать?

### Зачем спрашивают
Heap — важная структура для задач с приоритетами. Проверяют знание алгоритмов.

### Короткий ответ
`heapq` — реализация min-heap на основе list. Используется для приоритетных очередей, Top-K задач, merge sorted lists. Операции push/pop — O(log n), peek — O(1). Это min-heap, для max-heap нужно инвертировать значения.

### Детальный разбор

**Сложность операций:**

| Операция | Сложность |
|----------|-----------|
| `heappush` | O(log n) |
| `heappop` | O(log n) |
| `heap[0]` (peek) | O(1) |
| `heapify` | O(n) |
| `nlargest/nsmallest` | O(n log k) |

**Ключевые функции:**
```python
heapq.heappush(heap, item)      # Добавить
heapq.heappop(heap)             # Извлечь минимум
heapq.heappushpop(heap, item)   # Push + pop (эффективнее)
heapq.heapreplace(heap, item)   # Pop + push
heapq.heapify(list)             # Преобразовать list в heap
heapq.nlargest(n, iterable)     # Top-N largest
heapq.nsmallest(n, iterable)    # Top-N smallest
heapq.merge(*iterables)         # Merge sorted iterables
```

### Пример

```python
import heapq

# Базовое использование
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
print(heap[0])        # 1 (минимум)
print(heapq.heappop(heap))  # 1

# Преобразование list в heap
nums = [5, 3, 8, 1, 9]
heapq.heapify(nums)   # In-place O(n)
print(nums)           # [1, 3, 8, 5, 9]

# Max-heap через инверсию
max_heap = []
for num in [1, 3, 2]:
    heapq.heappush(max_heap, -num)
print(-heapq.heappop(max_heap))  # 3 (максимум)

# Приоритетная очередь с кортежами
tasks = []
heapq.heappush(tasks, (1, 'high priority'))
heapq.heappush(tasks, (3, 'low priority'))
heapq.heappush(tasks, (2, 'medium priority'))
print(heapq.heappop(tasks))  # (1, 'high priority')

# Для равных приоритетов - добавляем счётчик
import itertools
counter = itertools.count()

class PriorityQueue:
    def __init__(self):
        self.heap = []
        self.counter = itertools.count()

    def push(self, priority, item):
        # counter гарантирует FIFO для равных приоритетов
        heapq.heappush(self.heap, (priority, next(self.counter), item))

    def pop(self):
        return heapq.heappop(self.heap)[2]

# Top-K элементов
nums = [3, 1, 4, 1, 5, 9, 2, 6]
print(heapq.nlargest(3, nums))   # [9, 6, 5]
print(heapq.nsmallest(3, nums))  # [1, 1, 2]

# С ключом
users = [{'name': 'Alice', 'age': 30}, {'name': 'Bob', 'age': 25}]
oldest = heapq.nlargest(1, users, key=lambda x: x['age'])

# Merge sorted lists
list1 = [1, 3, 5]
list2 = [2, 4, 6]
merged = list(heapq.merge(list1, list2))  # [1, 2, 3, 4, 5, 6]

# Dijkstra's algorithm (пример)
def dijkstra(graph, start):
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    heap = [(0, start)]

    while heap:
        dist, node = heapq.heappop(heap)
        if dist > distances[node]:
            continue
        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                heapq.heappush(heap, (new_dist, neighbor))

    return distances
```

### Типичные ошибки
- Забывать, что это min-heap (не max-heap)
- Не использовать counter для стабильной сортировки
- Использовать nlargest/nsmallest вместо sorted для маленьких k

### На интервью
**Как отвечать:** Объясните min-heap на основе list, покажите как сделать max-heap, приведите пример приоритетной очереди.

**Follow-up вопросы:**
- Как реализовать max-heap?
- Когда heapq лучше sorted?
- Как работает heapify за O(n)?

---

## Вопрос 10: Как выбрать правильную структуру данных?

### Зачем спрашивают
Финальный вопрос на понимание trade-offs. Проверяют практический опыт.

### Короткий ответ
Выбор зависит от операций: list для индексации и append, deque для двусторонних операций, set/dict для быстрого поиска, heap для приоритетов. Ключевые факторы: частота операций, требования к памяти, порядок элементов.

### Детальный разбор

**Таблица выбора:**

| Задача | Структура | Почему |
|--------|-----------|--------|
| Последовательность с индексацией | `list` | O(1) индексация |
| FIFO очередь | `deque` | O(1) popleft |
| LIFO стек | `list` или `deque` | O(1) append/pop |
| Быстрый поиск | `set` или `dict` | O(1) lookup |
| Подсчёт элементов | `Counter` | Встроенные методы |
| Группировка | `defaultdict(list)` | Без проверок |
| Приоритетная очередь | `heapq` | O(log n) операции |
| Отсортированные данные | `sortedcontainers` | O(log n) операции |
| Неизменяемая последовательность | `tuple` | Hashable, меньше памяти |

### Пример

```python
# Задача: частые проверки принадлежности
# Плохо:
items = [1, 2, 3, 4, 5]  # O(n) поиск
if x in items:  # Медленно при большом списке
    pass

# Хорошо:
items = {1, 2, 3, 4, 5}  # O(1) поиск
if x in items:  # Быстро
    pass

# Задача: очередь с приоритетами
# Плохо:
tasks = []
tasks.append((priority, task))
tasks.sort()  # O(n log n) каждый раз!
next_task = tasks.pop(0)

# Хорошо:
import heapq
tasks = []
heapq.heappush(tasks, (priority, task))  # O(log n)
next_task = heapq.heappop(tasks)  # O(log n)

# Задача: подсчёт слов
# Плохо:
word_count = {}
for word in words:
    if word in word_count:
        word_count[word] += 1
    else:
        word_count[word] = 1

# Хорошо:
from collections import Counter
word_count = Counter(words)

# Задача: история команд с ограничением
# Плохо:
history = []
history.append(command)
if len(history) > 100:
    history.pop(0)  # O(n)!

# Хорошо:
from collections import deque
history = deque(maxlen=100)
history.append(command)  # Автоматически удаляет старые

# Алгоритм выбора
def choose_structure(
    need_order: bool,
    need_fast_lookup: bool,
    need_fast_insert_remove: bool,
    data_mutable: bool,
    need_duplicates: bool
) -> str:
    if need_fast_lookup and not need_order:
        return "set" if not need_duplicates else "Counter"

    if need_fast_lookup and need_order:
        return "dict"  # Python 3.7+ сохраняет порядок

    if need_fast_insert_remove:
        return "deque" if need_order else "set"

    if not data_mutable:
        return "tuple" if need_order else "frozenset"

    return "list"  # По умолчанию

# Big O сравнение
"""
           | list | dict/set | deque |
-----------+------+----------+-------+
index      | O(1) |   N/A    | O(n)  |
search     | O(n) |   O(1)   | O(n)  |
insert end | O(1) |   O(1)   | O(1)  |
insert beg | O(n) |   N/A    | O(1)  |
delete     | O(n) |   O(1)   | O(n)* |

* O(1) для концов
"""
```

### Типичные ошибки
- Использовать list для частых проверок `in`
- Не знать про `deque` для двусторонних операций
- Не использовать `Counter` и `defaultdict`

### На интервью
**Как отвечать:** Спросите о требованиях (какие операции чаще всего?), затем обоснуйте выбор через Big O. Упомяните альтернативы.

**Follow-up вопросы:**
- Когда list лучше set?
- Что использовать для отсортированных данных?
- Как оптимизировать память?

---

[← Назад к списку тем](README.md)
