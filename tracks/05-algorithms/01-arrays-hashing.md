# 01. Массивы и хеширование

[← Назад к списку тем](README.md)

---

## Задача 1: Two Sum

### Паттерн
Hash Map для O(1) lookup

### Условие
Дан массив чисел и target. Найти индексы двух чисел, сумма которых равна target.

### Интуиция
Для каждого числа x ищем complement = target - x. Hash map позволяет найти complement за O(1).

### Решение

```python
def two_sum(nums: list[int], target: int) -> list[int]:
    seen = {}  # value -> index
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []

# Пример
nums = [2, 7, 11, 15]
target = 9
print(two_sum(nums, target))  # [0, 1]
```

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int) // value -> index
    for i, num := range nums {
        complement := target - num
        if j, ok := seen[complement]; ok {
            return []int{j, i}
        }
        seen[num] = i
    }
    return nil
}
```

### Сложность
- Time: O(n)
- Space: O(n)

### Вариации
- **Two Sum II (sorted array):** используйте two pointers — O(1) space
- **Three Sum:** сортировка + two pointers — O(n²)
- **Two Sum — множество решений:** собрать все пары

---

## Задача 2: Contains Duplicate

### Паттерн
Hash Set для уникальности

### Условие
Определить, есть ли в массиве дубликаты.

### Решение

```python
def contains_duplicate(nums: list[int]) -> bool:
    return len(nums) != len(set(nums))

# Альтернатива с ранним выходом
def contains_duplicate_early(nums: list[int]) -> bool:
    seen = set()
    for num in nums:
        if num in seen:
            return True
        seen.add(num)
    return False
```

```go
func containsDuplicate(nums []int) bool {
    seen := make(map[int]bool)
    for _, num := range nums {
        if seen[num] {
            return true
        }
        seen[num] = true
    }
    return false
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 3: Valid Anagram

### Паттерн
Character counting с hash map

### Условие
Определить, является ли строка t анаграммой строки s.

### Решение

```python
from collections import Counter

def is_anagram(s: str, t: str) -> bool:
    return Counter(s) == Counter(t)

# Без Counter
def is_anagram_manual(s: str, t: str) -> bool:
    if len(s) != len(t):
        return False

    count = {}
    for c in s:
        count[c] = count.get(c, 0) + 1

    for c in t:
        if c not in count:
            return False
        count[c] -= 1
        if count[c] < 0:
            return False

    return True
```

```go
func isAnagram(s, t string) bool {
    if len(s) != len(t) {
        return false
    }

    count := make(map[rune]int)
    for _, c := range s {
        count[c]++
    }

    for _, c := range t {
        count[c]--
        if count[c] < 0 {
            return false
        }
    }
    return true
}
```

### Сложность
- Time: O(n)
- Space: O(k), где k — размер алфавита

### Вариации
- **Group Anagrams:** группировка слов-анаграмм по отсортированному ключу

---

## Задача 4: Group Anagrams

### Паттерн
Hash Map с ключом-сигнатурой

### Условие
Сгруппировать слова, которые являются анаграммами друг друга.

### Решение

```python
from collections import defaultdict

def group_anagrams(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for s in strs:
        # Ключ — отсортированная строка
        key = tuple(sorted(s))
        groups[key].append(s)
    return list(groups.values())

# Альтернатива: ключ — подсчёт символов (быстрее для длинных строк)
def group_anagrams_count(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for s in strs:
        count = [0] * 26
        for c in s:
            count[ord(c) - ord('a')] += 1
        groups[tuple(count)].append(s)
    return list(groups.values())

# Пример
strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
print(group_anagrams(strs))
# [["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]
```

```go
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, s := range strs {
        // Сортируем строку как ключ
        runes := []rune(s)
        sort.Slice(runes, func(i, j int) bool {
            return runes[i] < runes[j]
        })
        key := string(runes)
        groups[key] = append(groups[key], s)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}
```

### Сложность
- Time: O(n * k log k), где k — максимальная длина строки
- Space: O(n * k)

---

## Задача 5: Top K Frequent Elements

### Паттерн
Hash Map + частотный анализ

### Условие
Найти k самых частых элементов в массиве.

### Решение

```python
from collections import Counter
import heapq

def top_k_frequent(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)
    # nlargest использует heap — O(n log k)
    return [x for x, _ in count.most_common(k)]

# Bucket sort approach — O(n)
def top_k_frequent_bucket(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)

    # Bucket: индекс = частота, значение = список элементов
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, freq in count.items():
        buckets[freq].append(num)

    result = []
    for i in range(len(buckets) - 1, -1, -1):
        for num in buckets[i]:
            result.append(num)
            if len(result) == k:
                return result
    return result
```

```go
func topKFrequent(nums []int, k int) []int {
    count := make(map[int]int)
    for _, num := range nums {
        count[num]++
    }

    // Bucket sort
    buckets := make([][]int, len(nums)+1)
    for num, freq := range count {
        buckets[freq] = append(buckets[freq], num)
    }

    result := make([]int, 0, k)
    for i := len(buckets) - 1; i >= 0 && len(result) < k; i-- {
        result = append(result, buckets[i]...)
    }
    return result[:k]
}
```

### Сложность
- Heap approach: Time O(n log k), Space O(n)
- Bucket sort: Time O(n), Space O(n)

---

## Задача 6: Product of Array Except Self

### Паттерн
Prefix/Suffix products

### Условие
Для каждого элемента вычислить произведение всех остальных элементов (без деления).

### Интуиция
result[i] = (произведение всех слева) × (произведение всех справа)

### Решение

```python
def product_except_self(nums: list[int]) -> list[int]:
    n = len(nums)
    result = [1] * n

    # Prefix products (слева направо)
    prefix = 1
    for i in range(n):
        result[i] = prefix
        prefix *= nums[i]

    # Suffix products (справа налево)
    suffix = 1
    for i in range(n - 1, -1, -1):
        result[i] *= suffix
        suffix *= nums[i]

    return result

# Пример
nums = [1, 2, 3, 4]
print(product_except_self(nums))  # [24, 12, 8, 6]
```

```go
func productExceptSelf(nums []int) []int {
    n := len(nums)
    result := make([]int, n)

    // Prefix
    prefix := 1
    for i := 0; i < n; i++ {
        result[i] = prefix
        prefix *= nums[i]
    }

    // Suffix
    suffix := 1
    for i := n - 1; i >= 0; i-- {
        result[i] *= suffix
        suffix *= nums[i]
    }

    return result
}
```

### Сложность
- Time: O(n)
- Space: O(1) — не считая выходной массив

---

## Задача 7: Longest Consecutive Sequence

### Паттерн
Hash Set для O(1) проверки

### Условие
Найти длину самой длинной последовательности подряд идущих чисел. Решение должно быть O(n).

### Интуиция
Для каждого числа проверяем, является ли оно началом последовательности (num-1 не в set). Если да — считаем длину.

### Решение

```python
def longest_consecutive(nums: list[int]) -> int:
    num_set = set(nums)
    longest = 0

    for num in num_set:
        # Проверяем, что это начало последовательности
        if num - 1 not in num_set:
            current = num
            length = 1

            while current + 1 in num_set:
                current += 1
                length += 1

            longest = max(longest, length)

    return longest

# Пример
nums = [100, 4, 200, 1, 3, 2]
print(longest_consecutive(nums))  # 4 (последовательность 1, 2, 3, 4)
```

```go
func longestConsecutive(nums []int) int {
    numSet := make(map[int]bool)
    for _, num := range nums {
        numSet[num] = true
    }

    longest := 0
    for num := range numSet {
        // Начало последовательности
        if !numSet[num-1] {
            current := num
            length := 1

            for numSet[current+1] {
                current++
                length++
            }

            if length > longest {
                longest = length
            }
        }
    }
    return longest
}
```

### Сложность
- Time: O(n) — каждое число посещается максимум 2 раза
- Space: O(n)

---

## Задача 8: Subarray Sum Equals K

### Паттерн
Prefix Sum + Hash Map

### Условие
Найти количество подмассивов с суммой равной k.

### Интуиция
Если prefix[j] - prefix[i] = k, то сумма элементов от i+1 до j равна k. Храним количество каждой prefix sum в hash map.

### Решение

```python
def subarray_sum(nums: list[int], k: int) -> int:
    count = 0
    prefix_sum = 0
    prefix_count = {0: 1}  # sum -> количество раз

    for num in nums:
        prefix_sum += num

        # Сколько раз встречалась сумма (prefix_sum - k)?
        if prefix_sum - k in prefix_count:
            count += prefix_count[prefix_sum - k]

        prefix_count[prefix_sum] = prefix_count.get(prefix_sum, 0) + 1

    return count

# Пример
nums = [1, 1, 1]
k = 2
print(subarray_sum(nums, k))  # 2 ([1,1] дважды)
```

```go
func subarraySum(nums []int, k int) int {
    count := 0
    prefixSum := 0
    prefixCount := map[int]int{0: 1}

    for _, num := range nums {
        prefixSum += num

        if c, ok := prefixCount[prefixSum-k]; ok {
            count += c
        }

        prefixCount[prefixSum]++
    }

    return count
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## См. также

- [Collections Framework в Java](../01-java/02-collections.md) — реализация hash map и array list в Java
- [Структуры данных Python](../02-python/01-data-structures.md) — dict, set и list в Python

---

## На интервью

### Как распознать паттерн

| Сигнал | Паттерн |
|--------|---------|
| "Найти пару/тройку с суммой X" | Hash Map / Two Pointers |
| "Проверить дубликаты/уникальность" | Hash Set |
| "Подсчитать частоты" | Counter / Hash Map |
| "Сумма подмассива" | Prefix Sum + Hash Map |
| "Группировка по признаку" | Hash Map с ключом-сигнатурой |

### Типичные ошибки

1. Забывают про edge cases: пустой массив, один элемент
2. Не учитывают отрицательные числа в prefix sum
3. Путают index и value при работе с hash map
4. Забывают инициализировать prefix_count[0] = 1

---

[← Назад к списку тем](README.md)
