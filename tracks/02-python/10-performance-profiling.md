# 10. Производительность и профилирование

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как профилировать CPU?

### Зачем спрашивают
Профилирование — навык senior разработчика. Важно для оптимизации.

### Короткий ответ
`cProfile` — встроенный профайлер, показывает время в функциях. `py-spy` — sampling profiler, работает без изменения кода. `line_profiler` — построчное профилирование. Визуализация через `snakeviz` или flame graphs.

### Пример

```python
# cProfile
import cProfile
import pstats

cProfile.run('my_function()', 'output.prof')

# Анализ
stats = pstats.Stats('output.prof')
stats.sort_stats('cumulative')
stats.print_stats(10)

# Как декоратор
def profile(func):
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        result = profiler.runcall(func, *args, **kwargs)
        profiler.print_stats()
        return result
    return wrapper

# py-spy (без изменения кода)
# pip install py-spy
# py-spy top --pid 12345
# py-spy record -o profile.svg --pid 12345

# line_profiler
# pip install line_profiler
# @profile  # Декоратор от line_profiler
# def slow_function():
#     ...
# kernprof -l -v script.py

# snakeviz визуализация
# pip install snakeviz
# snakeviz output.prof
```

```bash
# Командная строка
python -m cProfile -o output.prof script.py
python -m cProfile -s cumulative script.py

# py-spy
py-spy record -o profile.svg -- python script.py
```

### На интервью
**Как отвечать:** Покажите cProfile для базового профилирования, py-spy для production, упомяните визуализацию.

---

## Вопрос 2: Как оптимизировать Python код?

### Зачем спрашивают
Практические знания оптимизации. Важно для senior.

### Короткий ответ
1) Используйте правильные структуры данных. 2) Избегайте лишних операций в циклах. 3) Используйте built-in функции и comprehensions. 4) Кэшируйте результаты. 5) Используйте генераторы для больших данных. 6) Для CPU-bound — multiprocessing или Cython.

### Пример

```python
# Структуры данных
# Плохо: O(n) поиск в списке
if item in large_list:
    ...

# Хорошо: O(1) поиск в set
if item in large_set:
    ...

# Выносите из цикла
# Плохо
for item in items:
    result = len(items)  # Вычисляется каждый раз
    ...

# Хорошо
length = len(items)
for item in items:
    ...

# Built-in функции (реализованы на C)
# Плохо
total = 0
for x in numbers:
    total += x

# Хорошо
total = sum(numbers)

# Comprehensions быстрее циклов
# Плохо
squares = []
for x in range(1000):
    squares.append(x ** 2)

# Хорошо
squares = [x ** 2 for x in range(1000)]

# Кэширование
from functools import lru_cache

@lru_cache(maxsize=1000)
def expensive_function(n):
    ...

# Генераторы для больших данных
# Плохо: всё в памяти
def get_all_lines(files):
    result = []
    for f in files:
        result.extend(open(f).readlines())
    return result

# Хорошо: ленивая генерация
def get_all_lines(files):
    for f in files:
        yield from open(f)

# String concatenation
# Плохо
s = ""
for item in items:
    s += str(item)

# Хорошо
s = "".join(str(item) for item in items)
```

### На интервью
**Как отвечать:** Перечислите основные техники, покажите примеры с Big O, упомяните профилирование перед оптимизацией.

---

## Вопрос 3: Когда использовать Cython или PyPy?

### Зачем спрашивают
Альтернативы для CPU-bound задач. Показывает глубину знаний.

### Короткий ответ
**PyPy** — альтернативный интерпретатор с JIT, ускоряет чистый Python в 5-10x без изменений кода. **Cython** — компилирует Python в C, требует типизации для максимального ускорения. PyPy для целого приложения, Cython для hot spots.

### Пример

```python
# PyPy - просто запустите код
# pypy script.py

# Cython
# setup.py
from setuptools import setup
from Cython.Build import cythonize

setup(ext_modules=cythonize("module.pyx"))

# module.pyx
def slow_function(int n):
    cdef int i, total = 0
    for i in range(n):
        total += i
    return total

# Компиляция
# python setup.py build_ext --inplace

# Или inline с cython.compile
import cython

@cython.compile
def fast_function(n: cython.int) -> cython.int:
    total: cython.int = 0
    i: cython.int
    for i in range(n):
        total += i
    return total
```

### На интервью
**Как отвечать:** PyPy для общего ускорения без изменений, Cython для критических участков с типизацией.

---

## Вопрос 4: Как работает JIT в Python? (numba)

### Зачем спрашивают
JIT — альтернатива Cython для числовых вычислений.

### Короткий ответ
`numba` компилирует Python функции в машинный код через LLVM. Декоратор `@jit` или `@njit`. Работает с NumPy arrays. Огромное ускорение для числовых вычислений без переписывания на C.

### Пример

```python
from numba import jit, njit
import numpy as np

# Базовое использование
@jit
def sum_array(arr):
    total = 0
    for x in arr:
        total += x
    return total

# njit = nopython mode (строгий, быстрее)
@njit
def fast_sum(arr):
    total = 0.0
    for i in range(len(arr)):
        total += arr[i]
    return total

# Параллельное выполнение
from numba import prange

@njit(parallel=True)
def parallel_sum(arr):
    total = 0.0
    for i in prange(len(arr)):
        total += arr[i]
    return total

# Бенчмарк
arr = np.random.rand(10_000_000)
%timeit sum(arr)           # ~500ms
%timeit np.sum(arr)        # ~5ms
%timeit fast_sum(arr)      # ~5ms (первый вызов медленнее - компиляция)
```

### На интервью
**Как отвечать:** Покажите @njit декоратор, объясните ограничения (numpy arrays), упомяните parallel.

---

## Вопрос 5: Что такое `__slots__` и как они экономят память?

### Зачем спрашивают
Практическая оптимизация памяти. Часто спрашивают.

### Короткий ответ
`__slots__` отключает `__dict__` у экземпляров, храня атрибуты в фиксированной структуре. Экономит ~40-50% памяти на объект. Ускоряет доступ к атрибутам. Ограничение: нельзя добавлять новые атрибуты.

### Пример

```python
import sys

class Normal:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class Slotted:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

# Сравнение памяти
n = Normal(1, 2)
s = Slotted(1, 2)

print(sys.getsizeof(n) + sys.getsizeof(n.__dict__))  # ~152 bytes
print(sys.getsizeof(s))  # ~56 bytes

# Для dataclass (Python 3.10+)
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: float
    y: float
```

### На интервью
**Как отвечать:** Объясните отключение `__dict__`, покажите экономию памяти, упомяните ограничения.

---

## Вопрос 6: Как использовать `__slots__` с dataclass?

### Зачем спрашивают
Современный способ оптимизации. Python 3.10+ фича.

### Короткий ответ
С Python 3.10 dataclass поддерживает `slots=True` параметр. Автоматически создаёт `__slots__` из полей. Комбинирует удобство dataclass с оптимизацией памяти.

### Пример

```python
from dataclasses import dataclass

# Python 3.10+
@dataclass(slots=True)
class Point:
    x: float
    y: float
    z: float = 0.0

# Эквивалент вручную
@dataclass
class PointManual:
    __slots__ = ('x', 'y', 'z')
    x: float
    y: float
    z: float = 0.0

# Комбинация с frozen
@dataclass(slots=True, frozen=True)
class ImmutablePoint:
    x: float
    y: float
# Hashable и оптимизированный по памяти!

# Проверка
p = Point(1.0, 2.0)
print(hasattr(p, '__dict__'))  # False
# p.w = 3.0  # AttributeError
```

### На интервью
**Как отвечать:** Покажите slots=True в dataclass, объясните преимущества комбинации.

---

## Вопрос 7: Как работать с большими данными эффективно?

### Зачем спрашивают
Обработка больших данных — практический навык.

### Короткий ответ
Используйте генераторы вместо списков. `itertools` для ленивых операций. Chunk processing для файлов. `pandas` с `chunksize`. `dask` или `polars` для очень больших данных.

### Пример

```python
# Генераторы вместо списков
def read_large_file(path):
    with open(path) as f:
        for line in f:
            yield line.strip()

# itertools для ленивых операций
from itertools import islice, chain, groupby

# Первые N элементов
first_100 = list(islice(generator, 100))

# Chunk processing
def process_in_chunks(items, chunk_size=1000):
    chunk = []
    for item in items:
        chunk.append(item)
        if len(chunk) >= chunk_size:
            yield chunk
            chunk = []
    if chunk:
        yield chunk

# pandas с chunksize
import pandas as pd

for chunk in pd.read_csv('large.csv', chunksize=10000):
    process(chunk)

# polars - быстрее pandas для больших данных
import polars as pl

df = pl.scan_csv('large.csv')  # Ленивое чтение
result = df.filter(pl.col('value') > 100).collect()

# Memory mapping для очень больших файлов
import mmap

with open('large.bin', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    # Чтение без загрузки всего файла в память
```

### На интервью
**Как отвечать:** Генераторы для ленивой обработки, chunk processing, упомяните polars/dask для big data.

---

[← Назад к списку тем](README.md)
