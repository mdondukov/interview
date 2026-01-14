# 00. Анализ сложности

[← Назад к списку тем](README.md)

---

## Теория: Big O нотация

### Зачем это нужно
Анализ сложности — фундамент для оценки алгоритмов. На интервью ожидают, что вы можете оценить time/space complexity любого решения.

### Основные классы сложности

| Сложность | Название | Пример |
|-----------|----------|--------|
| O(1) | Константная | Доступ по индексу |
| O(log n) | Логарифмическая | Бинарный поиск |
| O(n) | Линейная | Один проход по массиву |
| O(n log n) | Линейно-логарифмическая | Сортировка (merge, quick) |
| O(n²) | Квадратичная | Два вложенных цикла |
| O(2ⁿ) | Экспоненциальная | Подмножества, рекурсия без мемоизации |
| O(n!) | Факториальная | Перестановки |

### Правила анализа

1. **Константы игнорируются:** O(2n) = O(n)
2. **Слагаемые низшего порядка игнорируются:** O(n² + n) = O(n²)
3. **Умножение при вложенности:** два вложенных цикла по n = O(n²)
4. **Сложение при последовательности:** два цикла подряд = O(n + n) = O(n)

---

## Задача 1: Определить сложность

### Условие
Определите time complexity для каждого фрагмента кода.

### Примеры

```python
# Пример 1: O(n)
def example1(arr):
    total = 0
    for x in arr:
        total += x
    return total

# Пример 2: O(n²)
def example2(arr):
    for i in range(len(arr)):
        for j in range(len(arr)):
            print(arr[i], arr[j])

# Пример 3: O(n) — не O(n²)!
def example3(arr):
    for i in range(len(arr)):
        for j in range(i, min(i + 5, len(arr))):  # Внутренний цикл ≤ 5 итераций
            print(arr[i], arr[j])

# Пример 4: O(log n)
def example4(n):
    while n > 1:
        n //= 2

# Пример 5: O(n log n)
def example5(arr):
    for i in range(len(arr)):  # O(n)
        j = len(arr)
        while j > 1:           # O(log n)
            j //= 2

# Пример 6: O(2ⁿ) — рекурсия без мемоизации
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)

# Пример 7: O(n) — с мемоизацией
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n <= 1:
        return n
    return fib_memo(n-1) + fib_memo(n-2)
```

---

## Задача 2: Амортизированный анализ

### Паттерн
Amortized Analysis

### Условие
Объясните, почему `append()` в Python list имеет амортизированную сложность O(1), хотя иногда требует O(n).

### Интуиция
Dynamic array удваивает размер при переполнении. Копирование O(n) происходит редко — каждые n операций. Распределяя стоимость копирования на все операции: O(n) / n = O(1) амортизированно.

### Пример

```python
# Внутренняя реализация dynamic array
class DynamicArray:
    def __init__(self):
        self.capacity = 1
        self.size = 0
        self.data = [None] * self.capacity

    def append(self, item):
        if self.size == self.capacity:
            self._resize(2 * self.capacity)  # O(n) — редко
        self.data[self.size] = item          # O(1) — часто
        self.size += 1

    def _resize(self, new_capacity):
        new_data = [None] * new_capacity
        for i in range(self.size):
            new_data[i] = self.data[i]
        self.data = new_data
        self.capacity = new_capacity

# Анализ n операций append:
# Копирования: 1 + 2 + 4 + 8 + ... + n = 2n - 1 = O(n)
# Всего операций: n
# Амортизированная сложность: O(n) / n = O(1) на операцию
```

```go
// В Go slice работает аналогично
func appendDemo() {
    var s []int
    for i := 0; i < 10; i++ {
        s = append(s, i) // Амортизированно O(1)
        fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
    }
}
```

### Сложность
- Worst case (единичная операция): O(n)
- Amortized (в среднем): O(1)

---

## Задача 3: Space Complexity

### Паттерн
Space Analysis

### Условие
Определите space complexity для рекурсивных и итеративных решений.

### Примеры

```python
# O(1) extra space — итеративный
def reverse_array(arr):
    left, right = 0, len(arr) - 1
    while left < right:
        arr[left], arr[right] = arr[right], arr[left]
        left += 1
        right -= 1

# O(n) extra space — новый массив
def reverse_array_copy(arr):
    return arr[::-1]  # Создаёт копию

# O(n) space — стек вызовов
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)  # n кадров в стеке

# O(log n) space — сбалансированная рекурсия
def binary_search(arr, target, left, right):
    if left > right:
        return -1
    mid = (left + right) // 2
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search(arr, target, mid + 1, right)
    else:
        return binary_search(arr, target, left, mid - 1)
```

```go
// O(1) space — in-place
func reverseArray(arr []int) {
    for i, j := 0, len(arr)-1; i < j; i, j = i+1, j-1 {
        arr[i], arr[j] = arr[j], arr[i]
    }
}

// O(n) space — tail recursion не оптимизируется в Go
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}
```

---

## Задача 4: Сравнение алгоритмов сортировки

### Условие
Сравните сложности разных алгоритмов сортировки.

### Таблица сравнения

| Алгоритм | Best | Average | Worst | Space | Stable |
|----------|------|---------|-------|-------|--------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes |

### Когда какой использовать

```python
# Небольшие массивы (n < 50) — Insertion Sort
# Python's Timsort использует это внутри

# Общий случай — встроенная сортировка (Timsort)
arr.sort()  # O(n log n), stable

# Нужна стабильность — Merge Sort
from functools import cmp_to_key

# Известный диапазон значений — Counting Sort
def counting_sort(arr, max_val):
    count = [0] * (max_val + 1)
    for x in arr:
        count[x] += 1

    result = []
    for i, c in enumerate(count):
        result.extend([i] * c)
    return result
```

---

## Задача 5: Master Theorem

### Паттерн
Recurrence Relations

### Условие
Используйте Master Theorem для анализа рекурсивных алгоритмов.

### Формула
Для рекуррентности T(n) = aT(n/b) + O(nᵈ):

- Если d < log_b(a): T(n) = O(n^(log_b(a)))
- Если d = log_b(a): T(n) = O(nᵈ log n)
- Если d > log_b(a): T(n) = O(nᵈ)

### Примеры

```
# Binary Search: T(n) = T(n/2) + O(1)
# a=1, b=2, d=0
# log_2(1) = 0 = d → T(n) = O(log n)

# Merge Sort: T(n) = 2T(n/2) + O(n)
# a=2, b=2, d=1
# log_2(2) = 1 = d → T(n) = O(n log n)

# Karatsuba (умножение): T(n) = 3T(n/2) + O(n)
# a=3, b=2, d=1
# log_2(3) ≈ 1.58 > d=1 → T(n) = O(n^1.58)
```

---

## См. также

- [Массивы и хеширование](./01-arrays-hashing.md) — применение анализа сложности к структурам данных
- [Оптимизация SQL-запросов](../06-databases/02-query-optimization.md) — анализ сложности запросов к базам данных

---

## На интервью

### Как отвечать на вопросы о сложности

1. **Сначала определите операции:** что делает каждый цикл/рекурсия
2. **Считайте вложенность:** умножайте сложности вложенных структур
3. **Учитывайте скрытые операции:**
   - `in list` — O(n)
   - `in set/dict` — O(1)
   - `list.append()` — амортизированно O(1)
   - `list.insert(0, x)` — O(n)
4. **Различайте average и worst case**

### Типичные ошибки

- Забывают про space complexity стека рекурсии
- Не учитывают амортизированную сложность
- Путают O(log n) и O(n) для операций со словарём
- Забывают про сложность встроенных функций (sort, reverse)

### Полезные факты для интервью

```python
# Сложности операций в Python

# list
# - index access: O(1)
# - append: O(1) amortized
# - insert(i, x): O(n)
# - pop(): O(1)
# - pop(i): O(n)
# - in: O(n)
# - sort: O(n log n)

# dict / set
# - access/insert/delete: O(1) average, O(n) worst
# - in: O(1) average

# string
# - concatenation: O(n) — создаёт новую строку!
# - join: O(n) — эффективнее множественной конкатенации
# - slice: O(k) где k — размер slice

# collections.deque
# - append/appendleft: O(1)
# - pop/popleft: O(1)
# - access by index: O(n)

# heapq
# - heappush: O(log n)
# - heappop: O(log n)
# - heapify: O(n)
```

---

[← Назад к списку тем](README.md)
