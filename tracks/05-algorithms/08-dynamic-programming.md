# 08. Dynamic Programming

[← Назад к списку тем](README.md)

---

## Теория: Динамическое программирование

### Когда использовать DP
1. **Оптимальная подструктура:** оптимальное решение строится из оптимальных подзадач
2. **Перекрывающиеся подзадачи:** одни и те же подзадачи решаются многократно

### Подходы
- **Top-down (memoization):** рекурсия + кэширование
- **Bottom-up (tabulation):** итеративное заполнение таблицы

### Шаги решения
1. Определить состояние (что храним в dp[i])
2. Найти переходы (как dp[i] зависит от предыдущих)
3. Определить базовые случаи
4. Определить порядок вычисления
5. Определить ответ

---

## Задача 1: Climbing Stairs

### Паттерн
1D DP (Fibonacci)

### Условие
n ступенек, можно шагать на 1 или 2. Сколько способов подняться?

### Решение

```python
# Top-down с мемоизацией
from functools import lru_cache

def climb_stairs_memo(n: int) -> int:
    @lru_cache(maxsize=None)
    def dp(i):
        if i <= 2:
            return i
        return dp(i - 1) + dp(i - 2)

    return dp(n)

# Bottom-up
def climb_stairs(n: int) -> int:
    if n <= 2:
        return n

    dp = [0] * (n + 1)
    dp[1], dp[2] = 1, 2

    for i in range(3, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]

    return dp[n]

# Оптимизация памяти O(1)
def climb_stairs_optimized(n: int) -> int:
    if n <= 2:
        return n

    prev2, prev1 = 1, 2
    for _ in range(3, n + 1):
        prev2, prev1 = prev1, prev2 + prev1

    return prev1
```

```go
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }

    prev2, prev1 := 1, 2
    for i := 3; i <= n; i++ {
        prev2, prev1 = prev1, prev2+prev1
    }
    return prev1
}
```

### Сложность
- Time: O(n)
- Space: O(1) оптимизированно

---

## Задача 2: House Robber

### Паттерн
1D DP (выбор включать/не включать)

### Условие
Нельзя грабить соседние дома. Максимизировать добычу.

### Решение

```python
def rob(nums: list[int]) -> int:
    if not nums:
        return 0
    if len(nums) == 1:
        return nums[0]

    # dp[i] = max money up to house i
    # dp[i] = max(dp[i-1], dp[i-2] + nums[i])

    prev2, prev1 = 0, 0

    for num in nums:
        prev2, prev1 = prev1, max(prev1, prev2 + num)

    return prev1

# House Robber II (circular)
def rob_circular(nums: list[int]) -> int:
    if len(nums) == 1:
        return nums[0]

    def rob_linear(houses):
        prev2, prev1 = 0, 0
        for num in houses:
            prev2, prev1 = prev1, max(prev1, prev2 + num)
        return prev1

    # Либо не берём первый, либо не берём последний
    return max(rob_linear(nums[1:]), rob_linear(nums[:-1]))
```

```go
func rob(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    if len(nums) == 1 {
        return nums[0]
    }

    prev2, prev1 := 0, 0
    for _, num := range nums {
        prev2, prev1 = prev1, max(prev1, prev2+num)
    }
    return prev1
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 3: Coin Change

### Паттерн
Unbounded Knapsack

### Условие
Минимальное количество монет для суммы amount.

### Решение

```python
def coin_change(coins: list[int], amount: int) -> int:
    # dp[i] = min coins to make amount i
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0

    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i and dp[i - coin] != float('inf'):
                dp[i] = min(dp[i], dp[i - coin] + 1)

    return dp[amount] if dp[amount] != float('inf') else -1

# Количество способов набрать сумму
def coin_change_ways(coins: list[int], amount: int) -> int:
    dp = [0] * (amount + 1)
    dp[0] = 1

    for coin in coins:  # Порядок важен!
        for i in range(coin, amount + 1):
            dp[i] += dp[i - coin]

    return dp[amount]
```

```go
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := range dp {
        dp[i] = amount + 1 // "infinity"
    }
    dp[0] = 0

    for i := 1; i <= amount; i++ {
        for _, coin := range coins {
            if coin <= i && dp[i-coin] < dp[i]-1 {
                dp[i] = dp[i-coin] + 1
            }
        }
    }

    if dp[amount] > amount {
        return -1
    }
    return dp[amount]
}
```

### Сложность
- Time: O(amount × len(coins))
- Space: O(amount)

---

## Задача 4: Longest Increasing Subsequence (LIS)

### Паттерн
1D DP / Binary Search

### Условие
Найти длину наибольшей возрастающей подпоследовательности.

### Решение

```python
# O(n²) DP
def length_of_lis(nums: list[int]) -> int:
    if not nums:
        return 0

    n = len(nums)
    dp = [1] * n  # dp[i] = LIS ending at i

    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)

# O(n log n) с binary search
import bisect

def length_of_lis_optimized(nums: list[int]) -> int:
    # tails[i] = smallest tail element for LIS of length i+1
    tails = []

    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num

    return len(tails)

# Пример
nums = [10, 9, 2, 5, 3, 7, 101, 18]
print(length_of_lis(nums))  # 4 ([2, 3, 7, 101])
```

```go
func lengthOfLIS(nums []int) int {
    tails := []int{}

    for _, num := range nums {
        pos := sort.SearchInts(tails, num)
        if pos == len(tails) {
            tails = append(tails, num)
        } else {
            tails[pos] = num
        }
    }

    return len(tails)
}
```

### Сложность
- O(n²) DP версия
- O(n log n) с binary search
- Space: O(n)

---

## Задача 5: Longest Common Subsequence (LCS)

### Паттерн
2D DP

### Условие
Найти длину наибольшей общей подпоследовательности двух строк.

### Решение

```python
def longest_common_subsequence(text1: str, text2: str) -> int:
    m, n = len(text1), len(text2)

    # dp[i][j] = LCS of text1[:i] and text2[:j]
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[m][n]

# Оптимизация памяти до O(n)
def longest_common_subsequence_optimized(text1: str, text2: str) -> int:
    if len(text1) < len(text2):
        text1, text2 = text2, text1

    m, n = len(text1), len(text2)
    prev = [0] * (n + 1)

    for i in range(1, m + 1):
        curr = [0] * (n + 1)
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                curr[j] = prev[j - 1] + 1
            else:
                curr[j] = max(prev[j], curr[j - 1])
        prev = curr

    return prev[n]
```

```go
func longestCommonSubsequence(text1 string, text2 string) int {
    m, n := len(text1), len(text2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }

    return dp[m][n]
}
```

### Сложность
- Time: O(m × n)
- Space: O(m × n), оптимизированно O(min(m, n))

---

## Задача 6: 0/1 Knapsack

### Паттерн
2D DP → 1D оптимизация

### Условие
Выбрать предметы с максимальной ценностью, не превышая вместимость.

### Решение

```python
def knapsack(weights: list[int], values: list[int], capacity: int) -> int:
    n = len(weights)

    # 2D DP
    # dp = [[0] * (capacity + 1) for _ in range(n + 1)]
    # for i in range(1, n + 1):
    #     for w in range(capacity + 1):
    #         dp[i][w] = dp[i-1][w]
    #         if weights[i-1] <= w:
    #             dp[i][w] = max(dp[i][w], dp[i-1][w-weights[i-1]] + values[i-1])
    # return dp[n][capacity]

    # 1D оптимизация (обход справа налево!)
    dp = [0] * (capacity + 1)

    for i in range(n):
        for w in range(capacity, weights[i] - 1, -1):  # Обратный порядок!
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]

# Пример
weights = [1, 2, 3]
values = [6, 10, 12]
capacity = 5
print(knapsack(weights, values, capacity))  # 22 (items 1 and 2)
```

```go
func knapsack(weights, values []int, capacity int) int {
    dp := make([]int, capacity+1)

    for i := 0; i < len(weights); i++ {
        for w := capacity; w >= weights[i]; w-- {
            dp[w] = max(dp[w], dp[w-weights[i]]+values[i])
        }
    }

    return dp[capacity]
}
```

### Сложность
- Time: O(n × capacity)
- Space: O(capacity)

---

## Задача 7: Edit Distance

### Паттерн
2D DP (строковые операции)

### Условие
Минимальное количество операций (insert, delete, replace) для превращения word1 в word2.

### Решение

```python
def min_distance(word1: str, word2: str) -> int:
    m, n = len(word1), len(word2)

    # dp[i][j] = min operations for word1[:i] -> word2[:j]
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Base cases
    for i in range(m + 1):
        dp[i][0] = i  # Delete all
    for j in range(n + 1):
        dp[0][j] = j  # Insert all

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # Delete
                    dp[i][j - 1],      # Insert
                    dp[i - 1][j - 1]   # Replace
                )

    return dp[m][n]
```

```go
func minDistance(word1 string, word2 string) int {
    m, n := len(word1), len(word2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
        dp[i][0] = i
    }
    for j := 0; j <= n; j++ {
        dp[0][j] = j
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = 1 + min(dp[i-1][j], min(dp[i][j-1], dp[i-1][j-1]))
            }
        }
    }

    return dp[m][n]
}
```

### Сложность
- Time: O(m × n)
- Space: O(m × n)

---

## См. также

- [Backtracking](./09-backtracking.md) — альтернативный подход к задачам перебора
- [Графы](./07-graphs.md) — DP на графах (кратчайшие пути, топологическая сортировка)

---

## На интервью

### Паттерны DP

| Тип | Примеры |
|-----|---------|
| 1D linear | Climbing Stairs, House Robber |
| 1D с выбором | Coin Change, Knapsack |
| 2D строки | LCS, Edit Distance |
| 2D сетка | Unique Paths, Min Path Sum |
| Interval | Matrix Chain, Palindrome |

### Типичные ошибки

1. Неправильные базовые случаи
2. Неправильный порядок обхода
3. Off-by-one в индексах
4. Забывают про оптимизацию памяти

### Советы

1. Начните с рекурсии + мемоизации
2. Определите размерность DP таблицы
3. Чётко определите что означает dp[i][j]
4. Проверьте переходы на примере

---

[← Назад к списку тем](README.md)
