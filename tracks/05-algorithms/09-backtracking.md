# 09. Backtracking

[← Назад к списку тем](README.md)

---

## Теория: Backtracking

### Когда использовать
- Перебор всех возможных комбинаций/перестановок
- Поиск всех решений (не только оптимального)
- Задачи с ограничениями (constraint satisfaction)

### Шаблон

```python
def backtrack(state, choices):
    if is_solution(state):
        result.append(state.copy())
        return

    for choice in choices:
        if is_valid(choice, state):
            state.add(choice)        # Make choice
            backtrack(state, ...)    # Recurse
            state.remove(choice)     # Undo choice (backtrack)
```

### Отличие от DFS
- DFS обходит все узлы
- Backtracking отсекает ветви (pruning), которые не могут привести к решению

---

## Задача 1: Subsets

### Паттерн
Backtracking / Bit manipulation

### Условие
Найти все подмножества массива.

### Решение

```python
def subsets(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(start, current):
        result.append(current.copy())

        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()

    backtrack(0, [])
    return result

# Итеративный подход
def subsets_iterative(nums: list[int]) -> list[list[int]]:
    result = [[]]

    for num in nums:
        result += [subset + [num] for subset in result]

    return result

# Bit manipulation
def subsets_bits(nums: list[int]) -> list[list[int]]:
    n = len(nums)
    result = []

    for mask in range(1 << n):  # 2^n combinations
        subset = []
        for i in range(n):
            if mask & (1 << i):
                subset.append(nums[i])
        result.append(subset)

    return result

# Пример
nums = [1, 2, 3]
print(subsets(nums))
# [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
```

```go
func subsets(nums []int) [][]int {
    result := [][]int{}

    var backtrack func(start int, current []int)
    backtrack = func(start int, current []int) {
        // Make a copy to store
        subset := make([]int, len(current))
        copy(subset, current)
        result = append(result, subset)

        for i := start; i < len(nums); i++ {
            current = append(current, nums[i])
            backtrack(i+1, current)
            current = current[:len(current)-1]
        }
    }

    backtrack(0, []int{})
    return result
}
```

### Сложность
- Time: O(n × 2ⁿ)
- Space: O(n) для рекурсии

---

## Задача 2: Permutations

### Паттерн
Backtracking с used array

### Условие
Найти все перестановки массива.

### Решение

```python
def permute(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(current):
        if len(current) == len(nums):
            result.append(current.copy())
            return

        for num in nums:
            if num not in current:  # O(n) — можно оптимизировать с set
                current.append(num)
                backtrack(current)
                current.pop()

    backtrack([])
    return result

# Оптимизация с used array
def permute_optimized(nums: list[int]) -> list[list[int]]:
    result = []
    used = [False] * len(nums)

    def backtrack(current):
        if len(current) == len(nums):
            result.append(current.copy())
            return

        for i in range(len(nums)):
            if not used[i]:
                used[i] = True
                current.append(nums[i])
                backtrack(current)
                current.pop()
                used[i] = False

    backtrack([])
    return result

# Swap-based (in-place)
def permute_swap(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(start):
        if start == len(nums):
            result.append(nums.copy())
            return

        for i in range(start, len(nums)):
            nums[start], nums[i] = nums[i], nums[start]
            backtrack(start + 1)
            nums[start], nums[i] = nums[i], nums[start]

    backtrack(0)
    return result
```

```go
func permute(nums []int) [][]int {
    result := [][]int{}
    used := make([]bool, len(nums))

    var backtrack func(current []int)
    backtrack = func(current []int) {
        if len(current) == len(nums) {
            perm := make([]int, len(current))
            copy(perm, current)
            result = append(result, perm)
            return
        }

        for i := 0; i < len(nums); i++ {
            if !used[i] {
                used[i] = true
                current = append(current, nums[i])
                backtrack(current)
                current = current[:len(current)-1]
                used[i] = false
            }
        }
    }

    backtrack([]int{})
    return result
}
```

### Сложность
- Time: O(n × n!)
- Space: O(n)

---

## Задача 3: Combination Sum

### Паттерн
Backtracking с повторениями

### Условие
Найти все комбинации чисел, сумма которых равна target. Каждое число можно использовать неограниченно.

### Решение

```python
def combination_sum(candidates: list[int], target: int) -> list[list[int]]:
    result = []

    def backtrack(start, current, remaining):
        if remaining == 0:
            result.append(current.copy())
            return

        if remaining < 0:
            return

        for i in range(start, len(candidates)):
            current.append(candidates[i])
            backtrack(i, current, remaining - candidates[i])  # i, не i+1 — можно повторять
            current.pop()

    backtrack(0, [], target)
    return result

# Combination Sum II (каждый элемент только раз, есть дубликаты)
def combination_sum2(candidates: list[int], target: int) -> list[list[int]]:
    result = []
    candidates.sort()  # Важно для пропуска дубликатов

    def backtrack(start, current, remaining):
        if remaining == 0:
            result.append(current.copy())
            return

        if remaining < 0:
            return

        for i in range(start, len(candidates)):
            # Пропускаем дубликаты на том же уровне
            if i > start and candidates[i] == candidates[i - 1]:
                continue

            current.append(candidates[i])
            backtrack(i + 1, current, remaining - candidates[i])
            current.pop()

    backtrack(0, [], target)
    return result

# Пример
candidates = [2, 3, 6, 7]
target = 7
print(combination_sum(candidates, target))  # [[2,2,3], [7]]
```

```go
func combinationSum(candidates []int, target int) [][]int {
    result := [][]int{}

    var backtrack func(start int, current []int, remaining int)
    backtrack = func(start int, current []int, remaining int) {
        if remaining == 0 {
            comb := make([]int, len(current))
            copy(comb, current)
            result = append(result, comb)
            return
        }

        if remaining < 0 {
            return
        }

        for i := start; i < len(candidates); i++ {
            current = append(current, candidates[i])
            backtrack(i, current, remaining-candidates[i])
            current = current[:len(current)-1]
        }
    }

    backtrack(0, []int{}, target)
    return result
}
```

### Сложность
- Time: O(n^(target/min))
- Space: O(target/min)

---

## Задача 4: N-Queens

### Паттерн
Constraint-based backtracking

### Условие
Разместить n ферзей на доске n×n так, чтобы они не били друг друга.

### Решение

```python
def solve_n_queens(n: int) -> list[list[str]]:
    result = []
    board = [['.'] * n for _ in range(n)]

    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col

    def backtrack(row):
        if row == n:
            result.append([''.join(r) for r in board])
            return

        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue

            # Place queen
            board[row][col] = 'Q'
            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)

            backtrack(row + 1)

            # Remove queen
            board[row][col] = '.'
            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)

    backtrack(0)
    return result

# Только количество решений
def total_n_queens(n: int) -> int:
    count = 0
    cols = set()
    diag1 = set()
    diag2 = set()

    def backtrack(row):
        nonlocal count
        if row == n:
            count += 1
            return

        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue

            cols.add(col)
            diag1.add(row - col)
            diag2.add(row + col)

            backtrack(row + 1)

            cols.remove(col)
            diag1.remove(row - col)
            diag2.remove(row + col)

    backtrack(0)
    return count
```

```go
func solveNQueens(n int) [][]string {
    result := [][]string{}
    board := make([][]byte, n)
    for i := range board {
        board[i] = make([]byte, n)
        for j := range board[i] {
            board[i][j] = '.'
        }
    }

    cols := make(map[int]bool)
    diag1 := make(map[int]bool)
    diag2 := make(map[int]bool)

    var backtrack func(row int)
    backtrack = func(row int) {
        if row == n {
            solution := make([]string, n)
            for i := range board {
                solution[i] = string(board[i])
            }
            result = append(result, solution)
            return
        }

        for col := 0; col < n; col++ {
            if cols[col] || diag1[row-col] || diag2[row+col] {
                continue
            }

            board[row][col] = 'Q'
            cols[col] = true
            diag1[row-col] = true
            diag2[row+col] = true

            backtrack(row + 1)

            board[row][col] = '.'
            delete(cols, col)
            delete(diag1, row-col)
            delete(diag2, row+col)
        }
    }

    backtrack(0)
    return result
}
```

### Сложность
- Time: O(n!)
- Space: O(n)

---

## Задача 5: Word Search

### Паттерн
Backtracking на сетке

### Условие
Найти слово в матрице букв (можно двигаться в 4 направлениях).

### Решение

```python
def exist(board: list[list[str]], word: str) -> bool:
    rows, cols = len(board), len(board[0])

    def backtrack(r, c, i):
        if i == len(word):
            return True

        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != word[i]:
            return False

        # Mark visited
        temp, board[r][c] = board[r][c], '#'

        # Try all directions
        found = (backtrack(r + 1, c, i + 1) or
                 backtrack(r - 1, c, i + 1) or
                 backtrack(r, c + 1, i + 1) or
                 backtrack(r, c - 1, i + 1))

        # Restore
        board[r][c] = temp

        return found

    for r in range(rows):
        for c in range(cols):
            if backtrack(r, c, 0):
                return True

    return False
```

```go
func exist(board [][]byte, word string) bool {
    rows, cols := len(board), len(board[0])

    var backtrack func(r, c, i int) bool
    backtrack = func(r, c, i int) bool {
        if i == len(word) {
            return true
        }

        if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] != word[i] {
            return false
        }

        temp := board[r][c]
        board[r][c] = '#'

        found := backtrack(r+1, c, i+1) ||
                 backtrack(r-1, c, i+1) ||
                 backtrack(r, c+1, i+1) ||
                 backtrack(r, c-1, i+1)

        board[r][c] = temp

        return found
    }

    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if backtrack(r, c, 0) {
                return true
            }
        }
    }

    return false
}
```

### Сложность
- Time: O(m × n × 4^L), где L — длина слова
- Space: O(L)

---

## Задача 6: Palindrome Partitioning

### Паттерн
Backtracking + проверка палиндрома

### Условие
Разбить строку на подстроки-палиндромы.

### Решение

```python
def partition(s: str) -> list[list[str]]:
    result = []

    def is_palindrome(string):
        return string == string[::-1]

    def backtrack(start, current):
        if start == len(s):
            result.append(current.copy())
            return

        for end in range(start + 1, len(s) + 1):
            substring = s[start:end]
            if is_palindrome(substring):
                current.append(substring)
                backtrack(end, current)
                current.pop()

    backtrack(0, [])
    return result

# Пример
s = "aab"
print(partition(s))  # [["a","a","b"], ["aa","b"]]
```

```go
func partition(s string) [][]string {
    result := [][]string{}

    isPalindrome := func(str string) bool {
        for i, j := 0, len(str)-1; i < j; i, j = i+1, j-1 {
            if str[i] != str[j] {
                return false
            }
        }
        return true
    }

    var backtrack func(start int, current []string)
    backtrack = func(start int, current []string) {
        if start == len(s) {
            partition := make([]string, len(current))
            copy(partition, current)
            result = append(result, partition)
            return
        }

        for end := start + 1; end <= len(s); end++ {
            substring := s[start:end]
            if isPalindrome(substring) {
                current = append(current, substring)
                backtrack(end, current)
                current = current[:len(current)-1]
            }
        }
    }

    backtrack(0, []string{})
    return result
}
```

### Сложность
- Time: O(n × 2ⁿ)
- Space: O(n)

---

## См. также

- [Динамическое программирование](./08-dynamic-programming.md) — оптимизация рекурсивных решений через мемоизацию
- [Графы](./07-graphs.md) — DFS и обход графов связаны с backtracking

---

## На интервью

### Шаблон решения

1. Определите что является "выбором" на каждом шаге
2. Определите условие завершения
3. Определите ограничения (pruning)
4. Реализуйте make choice → recurse → undo choice

### Типичные ошибки

1. Забывают делать copy() при добавлении результата
2. Не восстанавливают состояние после рекурсии
3. Неправильные границы циклов
4. Забывают про pruning (пропуск дубликатов)

### Pruning техники

```python
# Пропуск дубликатов (после сортировки)
if i > start and nums[i] == nums[i-1]:
    continue

# Ранний выход
if remaining < 0:
    return

# Предварительная фильтрация
candidates.sort()  # Для раннего выхода
```

---

[← Назад к списку тем](README.md)
