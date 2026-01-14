# 03. Binary Search

[← Назад к списку тем](README.md)

---

## Теория: Binary Search

### Когда использовать
- Отсортированный массив
- Поиск границы (первый/последний элемент с условием)
- Поиск в пространстве решений (search space)
- "Минимизировать максимум" / "Максимизировать минимум"

### Шаблон

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2  # Избегаем overflow

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1  # Не найден
```

### Варианты границ

```python
# left <= right: ищем конкретный элемент
# left < right: ищем границу (left == right в конце)
# left + 1 < right: когда нужно сравнить соседей
```

---

## Задача 1: Classic Binary Search

### Паттерн
Basic Binary Search

### Условие
Найти индекс target в отсортированном массиве.

### Решение

```python
def binary_search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

```go
func binarySearch(nums []int, target int) int {
    left, right := 0, len(nums)-1

    for left <= right {
        mid := left + (right-left)/2

        if nums[mid] == target {
            return mid
        } else if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}
```

### Сложность
- Time: O(log n)
- Space: O(1)

---

## Задача 2: Search Insert Position

### Паттерн
Lower Bound

### Условие
Найти позицию для вставки target в отсортированный массив.

### Интуиция
Ищем первый элемент >= target (lower bound).

### Решение

```python
def search_insert(nums: list[int], target: int) -> int:
    left, right = 0, len(nums)

    while left < right:
        mid = left + (right - left) // 2

        if nums[mid] < target:
            left = mid + 1
        else:
            right = mid

    return left

# Используя bisect
import bisect

def search_insert_bisect(nums: list[int], target: int) -> int:
    return bisect.bisect_left(nums, target)
```

```go
func searchInsert(nums []int, target int) int {
    left, right := 0, len(nums)

    for left < right {
        mid := left + (right-left)/2

        if nums[mid] < target {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return left
}

// Используя sort.SearchInts
import "sort"

func searchInsertStd(nums []int, target int) int {
    return sort.SearchInts(nums, target)
}
```

### Сложность
- Time: O(log n)
- Space: O(1)

---

## Задача 3: Find First and Last Position

### Паттерн
Lower/Upper Bound

### Условие
Найти первую и последнюю позицию target в отсортированном массиве.

### Решение

```python
def search_range(nums: list[int], target: int) -> list[int]:
    def find_left():
        left, right = 0, len(nums)
        while left < right:
            mid = left + (right - left) // 2
            if nums[mid] < target:
                left = mid + 1
            else:
                right = mid
        return left

    def find_right():
        left, right = 0, len(nums)
        while left < right:
            mid = left + (right - left) // 2
            if nums[mid] <= target:  # <= для upper bound
                left = mid + 1
            else:
                right = mid
        return left - 1

    left_idx = find_left()

    if left_idx >= len(nums) or nums[left_idx] != target:
        return [-1, -1]

    return [left_idx, find_right()]

# Используя bisect
import bisect

def search_range_bisect(nums: list[int], target: int) -> list[int]:
    left = bisect.bisect_left(nums, target)
    if left >= len(nums) or nums[left] != target:
        return [-1, -1]
    right = bisect.bisect_right(nums, target) - 1
    return [left, right]

# Пример
nums = [5, 7, 7, 8, 8, 10]
target = 8
print(search_range(nums, target))  # [3, 4]
```

```go
func searchRange(nums []int, target int) []int {
    findLeft := func() int {
        left, right := 0, len(nums)
        for left < right {
            mid := left + (right-left)/2
            if nums[mid] < target {
                left = mid + 1
            } else {
                right = mid
            }
        }
        return left
    }

    findRight := func() int {
        left, right := 0, len(nums)
        for left < right {
            mid := left + (right-left)/2
            if nums[mid] <= target {
                left = mid + 1
            } else {
                right = mid
            }
        }
        return left - 1
    }

    leftIdx := findLeft()
    if leftIdx >= len(nums) || nums[leftIdx] != target {
        return []int{-1, -1}
    }
    return []int{leftIdx, findRight()}
}
```

### Сложность
- Time: O(log n)
- Space: O(1)

---

## Задача 4: Search in Rotated Sorted Array

### Паттерн
Modified Binary Search

### Условие
Найти target в отсортированном массиве, который был повёрнут (rotated).

### Интуиция
Одна из половин всегда отсортирована. Определяем какая и ищем в ней.

### Решение

```python
def search_rotated(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid

        # Левая часть отсортирована
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Правая часть отсортирована
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1

# Пример
nums = [4, 5, 6, 7, 0, 1, 2]
target = 0
print(search_rotated(nums, target))  # 4
```

```go
func searchRotated(nums []int, target int) int {
    left, right := 0, len(nums)-1

    for left <= right {
        mid := left + (right-left)/2

        if nums[mid] == target {
            return mid
        }

        // Левая часть отсортирована
        if nums[left] <= nums[mid] {
            if nums[left] <= target && target < nums[mid] {
                right = mid - 1
            } else {
                left = mid + 1
            }
        } else {
            // Правая часть отсортирована
            if nums[mid] < target && target <= nums[right] {
                left = mid + 1
            } else {
                right = mid - 1
            }
        }
    }
    return -1
}
```

### Сложность
- Time: O(log n)
- Space: O(1)

### Вариации
- **С дубликатами:** worst case O(n) когда nums[left] == nums[mid] == nums[right]
- **Найти минимум:** искать точку разрыва

---

## Задача 5: Find Minimum in Rotated Sorted Array

### Паттерн
Binary Search для точки разрыва

### Условие
Найти минимальный элемент в rotated sorted array.

### Решение

```python
def find_min(nums: list[int]) -> int:
    left, right = 0, len(nums) - 1

    while left < right:
        mid = left + (right - left) // 2

        if nums[mid] > nums[right]:
            # Минимум справа
            left = mid + 1
        else:
            # Минимум слева или это mid
            right = mid

    return nums[left]

# Пример
nums = [3, 4, 5, 1, 2]
print(find_min(nums))  # 1
```

```go
func findMin(nums []int) int {
    left, right := 0, len(nums)-1

    for left < right {
        mid := left + (right-left)/2

        if nums[mid] > nums[right] {
            left = mid + 1
        } else {
            right = mid
        }
    }
    return nums[left]
}
```

### Сложность
- Time: O(log n)
- Space: O(1)

---

## Задача 6: Search Space — Koko Eating Bananas

### Паттерн
Binary Search on Answer

### Условие
Коко ест бананы со скоростью k бананов/час. Есть h часов. Найти минимальное k, чтобы съесть все кучи бананов.

### Интуиция
Бинарный поиск по ответу: ищем минимальное k в диапазоне [1, max(piles)].

### Решение

```python
import math

def min_eating_speed(piles: list[int], h: int) -> int:
    def can_finish(k: int) -> bool:
        hours = sum(math.ceil(pile / k) for pile in piles)
        return hours <= h

    left, right = 1, max(piles)

    while left < right:
        mid = left + (right - left) // 2

        if can_finish(mid):
            right = mid  # Пробуем меньше
        else:
            left = mid + 1

    return left

# Пример
piles = [3, 6, 7, 11]
h = 8
print(min_eating_speed(piles, h))  # 4
```

```go
func minEatingSpeed(piles []int, h int) int {
    canFinish := func(k int) bool {
        hours := 0
        for _, pile := range piles {
            hours += (pile + k - 1) / k // ceil division
        }
        return hours <= h
    }

    left, right := 1, 0
    for _, pile := range piles {
        if pile > right {
            right = pile
        }
    }

    for left < right {
        mid := left + (right-left)/2

        if canFinish(mid) {
            right = mid
        } else {
            left = mid + 1
        }
    }
    return left
}
```

### Сложность
- Time: O(n * log(max(piles)))
- Space: O(1)

---

## Задача 7: Capacity To Ship Packages

### Паттерн
Binary Search on Answer (Minimize Maximum)

### Условие
Отправить все посылки за D дней. Найти минимальную вместимость корабля.

### Решение

```python
def ship_within_days(weights: list[int], days: int) -> int:
    def can_ship(capacity: int) -> bool:
        current_day = 1
        current_weight = 0

        for w in weights:
            if current_weight + w > capacity:
                current_day += 1
                current_weight = 0
            current_weight += w

        return current_day <= days

    left = max(weights)  # Минимум — самый тяжёлый груз
    right = sum(weights)  # Максимум — всё за один день

    while left < right:
        mid = left + (right - left) // 2

        if can_ship(mid):
            right = mid
        else:
            left = mid + 1

    return left

# Пример
weights = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
days = 5
print(ship_within_days(weights, days))  # 15
```

```go
func shipWithinDays(weights []int, days int) int {
    canShip := func(capacity int) bool {
        currentDay := 1
        currentWeight := 0

        for _, w := range weights {
            if currentWeight+w > capacity {
                currentDay++
                currentWeight = 0
            }
            currentWeight += w
        }
        return currentDay <= days
    }

    left, right := 0, 0
    for _, w := range weights {
        if w > left {
            left = w
        }
        right += w
    }

    for left < right {
        mid := left + (right-left)/2

        if canShip(mid) {
            right = mid
        } else {
            left = mid + 1
        }
    }
    return left
}
```

### Сложность
- Time: O(n * log(sum - max))
- Space: O(1)

---

## См. также

- [Деревья и обход](./06-trees-traversal.md) — бинарные деревья поиска (BST)
- [Индексы в базах данных](../06-databases/01-indexing.md) — B-tree индексы используют бинарный поиск

---

## На интервью

### Как распознать Binary Search

| Сигнал | Тип |
|--------|-----|
| Отсортированный массив | Classic |
| "Найти первый/последний с условием" | Lower/Upper bound |
| "Минимизировать максимум" | Search on answer |
| "Максимизировать минимум" | Search on answer |
| Монотонная функция | Search on answer |

### Типичные ошибки

1. **Overflow:** используйте `mid = left + (right - left) // 2`
2. **Бесконечный цикл:** проверьте условия `left <= right` vs `left < right`
3. **Off-by-one:** осторожно с `mid + 1` и `mid - 1`
4. **Неправильные границы:** начальные left/right

### Шаблон для Search on Answer

```python
def binary_search_answer(arr, constraint):
    left, right = min_possible, max_possible

    while left < right:
        mid = left + (right - left) // 2

        if is_feasible(mid, arr, constraint):
            right = mid  # Ищем меньше (minimize)
            # или left = mid + 1 (maximize)
        else:
            left = mid + 1  # Ищем больше (minimize)
            # или right = mid (maximize)

    return left
```

---

[← Назад к списку тем](README.md)
