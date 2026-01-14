# 05. Stacks & Queues

[← Назад к списку тем](README.md)

---

## Теория

### Stack (LIFO)
- Push: O(1)
- Pop: O(1)
- Peek: O(1)

### Queue (FIFO)
- Enqueue: O(1)
- Dequeue: O(1)

### Monotonic Stack
Стек, в котором элементы упорядочены (возрастают или убывают). Используется для задач "следующий больший/меньший элемент".

---

## Задача 1: Valid Parentheses

### Паттерн
Stack для matching

### Условие
Проверить, что скобки правильно сбалансированы.

### Решение

```python
def is_valid(s: str) -> bool:
    stack = []
    mapping = {')': '(', '}': '{', ']': '['}

    for char in s:
        if char in mapping:
            if not stack or stack[-1] != mapping[char]:
                return False
            stack.pop()
        else:
            stack.append(char)

    return len(stack) == 0

# Пример
print(is_valid("()[]{}"))  # True
print(is_valid("([)]"))    # False
```

```go
func isValid(s string) bool {
    stack := []rune{}
    mapping := map[rune]rune{')': '(', '}': '{', ']': '['}

    for _, char := range s {
        if open, ok := mapping[char]; ok {
            if len(stack) == 0 || stack[len(stack)-1] != open {
                return false
            }
            stack = stack[:len(stack)-1]
        } else {
            stack = append(stack, char)
        }
    }

    return len(stack) == 0
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 2: Min Stack

### Паттерн
Stack с дополнительной информацией

### Условие
Реализовать стек с операцией getMin() за O(1).

### Решение

```python
class MinStack:
    def __init__(self):
        self.stack = []      # (value, current_min)

    def push(self, val: int) -> None:
        current_min = min(val, self.stack[-1][1]) if self.stack else val
        self.stack.append((val, current_min))

    def pop(self) -> None:
        self.stack.pop()

    def top(self) -> int:
        return self.stack[-1][0]

    def getMin(self) -> int:
        return self.stack[-1][1]

# Альтернатива: два стека
class MinStack2:
    def __init__(self):
        self.stack = []
        self.min_stack = []

    def push(self, val: int) -> None:
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)

    def pop(self) -> None:
        if self.stack.pop() == self.min_stack[-1]:
            self.min_stack.pop()

    def top(self) -> int:
        return self.stack[-1]

    def getMin(self) -> int:
        return self.min_stack[-1]
```

```go
type MinStack struct {
    stack []int
    minStack []int
}

func Constructor() MinStack {
    return MinStack{}
}

func (s *MinStack) Push(val int) {
    s.stack = append(s.stack, val)
    if len(s.minStack) == 0 || val <= s.minStack[len(s.minStack)-1] {
        s.minStack = append(s.minStack, val)
    }
}

func (s *MinStack) Pop() {
    if s.stack[len(s.stack)-1] == s.minStack[len(s.minStack)-1] {
        s.minStack = s.minStack[:len(s.minStack)-1]
    }
    s.stack = s.stack[:len(s.stack)-1]
}

func (s *MinStack) Top() int {
    return s.stack[len(s.stack)-1]
}

func (s *MinStack) GetMin() int {
    return s.minStack[len(s.minStack)-1]
}
```

### Сложность
- Time: O(1) все операции
- Space: O(n)

---

## Задача 3: Evaluate Reverse Polish Notation

### Паттерн
Stack для вычислений

### Условие
Вычислить выражение в обратной польской нотации.

### Решение

```python
def eval_rpn(tokens: list[str]) -> int:
    stack = []
    ops = {
        '+': lambda a, b: a + b,
        '-': lambda a, b: a - b,
        '*': lambda a, b: a * b,
        '/': lambda a, b: int(a / b),  # Truncate toward zero
    }

    for token in tokens:
        if token in ops:
            b, a = stack.pop(), stack.pop()
            stack.append(ops[token](a, b))
        else:
            stack.append(int(token))

    return stack[0]

# Пример
tokens = ["2", "1", "+", "3", "*"]
print(eval_rpn(tokens))  # 9 = ((2 + 1) * 3)
```

```go
func evalRPN(tokens []string) int {
    stack := []int{}

    for _, token := range tokens {
        switch token {
        case "+", "-", "*", "/":
            b, a := stack[len(stack)-1], stack[len(stack)-2]
            stack = stack[:len(stack)-2]

            var result int
            switch token {
            case "+":
                result = a + b
            case "-":
                result = a - b
            case "*":
                result = a * b
            case "/":
                result = a / b
            }
            stack = append(stack, result)
        default:
            num, _ := strconv.Atoi(token)
            stack = append(stack, num)
        }
    }
    return stack[0]
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 4: Daily Temperatures

### Паттерн
Monotonic Stack (decreasing)

### Условие
Для каждого дня найти, через сколько дней будет теплее.

### Интуиция
Храним в стеке индексы дней с убывающими температурами. Когда встречаем более тёплый день — вычисляем ответы для всех дней в стеке.

### Решение

```python
def daily_temperatures(temperatures: list[int]) -> list[int]:
    n = len(temperatures)
    result = [0] * n
    stack = []  # Индексы

    for i in range(n):
        # Пока текущая температура выше тех, что в стеке
        while stack and temperatures[i] > temperatures[stack[-1]]:
            prev_idx = stack.pop()
            result[prev_idx] = i - prev_idx

        stack.append(i)

    return result

# Пример
temps = [73, 74, 75, 71, 69, 72, 76, 73]
print(daily_temperatures(temps))  # [1, 1, 4, 2, 1, 1, 0, 0]
```

```go
func dailyTemperatures(temperatures []int) []int {
    n := len(temperatures)
    result := make([]int, n)
    stack := []int{} // Индексы

    for i := 0; i < n; i++ {
        for len(stack) > 0 && temperatures[i] > temperatures[stack[len(stack)-1]] {
            prevIdx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[prevIdx] = i - prevIdx
        }
        stack = append(stack, i)
    }

    return result
}
```

### Сложность
- Time: O(n) — каждый элемент добавляется и удаляется из стека один раз
- Space: O(n)

---

## Задача 5: Next Greater Element

### Паттерн
Monotonic Stack

### Условие
Для каждого элемента найти следующий больший элемент справа.

### Решение

```python
def next_greater_element(nums: list[int]) -> list[int]:
    n = len(nums)
    result = [-1] * n
    stack = []

    # Обход справа налево
    for i in range(n - 1, -1, -1):
        # Удаляем элементы меньше или равные текущему
        while stack and stack[-1] <= nums[i]:
            stack.pop()

        if stack:
            result[i] = stack[-1]

        stack.append(nums[i])

    return result

# Альтернатива: обход слева направо (как в Daily Temperatures)
def next_greater_element_v2(nums: list[int]) -> list[int]:
    n = len(nums)
    result = [-1] * n
    stack = []  # Индексы

    for i in range(n):
        while stack and nums[i] > nums[stack[-1]]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result

# Пример
nums = [2, 1, 2, 4, 3]
print(next_greater_element(nums))  # [4, 2, 4, -1, -1]
```

```go
func nextGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }

    stack := []int{} // Индексы

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }

    return result
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 6: Largest Rectangle in Histogram

### Паттерн
Monotonic Stack (increasing)

### Условие
Найти площадь наибольшего прямоугольника в гистограмме.

### Интуиция
Для каждого столбца находим, как далеко он может расшириться влево и вправо. Используем стек для хранения возрастающих высот.

### Решение

```python
def largest_rectangle_area(heights: list[int]) -> int:
    stack = []  # Индексы
    max_area = 0
    heights.append(0)  # Sentinel для обработки оставшихся

    for i, h in enumerate(heights):
        start = i
        while stack and stack[-1][1] > h:
            idx, height = stack.pop()
            max_area = max(max_area, height * (i - idx))
            start = idx

        stack.append((start, h))

    return max_area

# Альтернатива без изменения входа
def largest_rectangle_area_v2(heights: list[int]) -> int:
    stack = [-1]  # Sentinel index
    max_area = 0

    for i, h in enumerate(heights):
        while stack[-1] != -1 and heights[stack[-1]] >= h:
            height = heights[stack.pop()]
            width = i - stack[-1] - 1
            max_area = max(max_area, height * width)
        stack.append(i)

    while stack[-1] != -1:
        height = heights[stack.pop()]
        width = len(heights) - stack[-1] - 1
        max_area = max(max_area, height * width)

    return max_area

# Пример
heights = [2, 1, 5, 6, 2, 3]
print(largest_rectangle_area(heights))  # 10
```

```go
func largestRectangleArea(heights []int) int {
    stack := []int{-1}
    maxArea := 0

    for i, h := range heights {
        for stack[len(stack)-1] != -1 && heights[stack[len(stack)-1]] >= h {
            height := heights[stack[len(stack)-1]]
            stack = stack[:len(stack)-1]
            width := i - stack[len(stack)-1] - 1
            if height*width > maxArea {
                maxArea = height * width
            }
        }
        stack = append(stack, i)
    }

    for stack[len(stack)-1] != -1 {
        height := heights[stack[len(stack)-1]]
        stack = stack[:len(stack)-1]
        width := len(heights) - stack[len(stack)-1] - 1
        if height*width > maxArea {
            maxArea = height * width
        }
    }

    return maxArea
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 7: Implement Queue using Stacks

### Паттерн
Two Stacks

### Условие
Реализовать очередь с помощью двух стеков.

### Решение

```python
class MyQueue:
    def __init__(self):
        self.input_stack = []   # Для push
        self.output_stack = []  # Для pop/peek

    def push(self, x: int) -> None:
        self.input_stack.append(x)

    def pop(self) -> int:
        self._transfer()
        return self.output_stack.pop()

    def peek(self) -> int:
        self._transfer()
        return self.output_stack[-1]

    def empty(self) -> bool:
        return not self.input_stack and not self.output_stack

    def _transfer(self):
        if not self.output_stack:
            while self.input_stack:
                self.output_stack.append(self.input_stack.pop())
```

```go
type MyQueue struct {
    inputStack  []int
    outputStack []int
}

func Constructor() MyQueue {
    return MyQueue{}
}

func (q *MyQueue) Push(x int) {
    q.inputStack = append(q.inputStack, x)
}

func (q *MyQueue) transfer() {
    if len(q.outputStack) == 0 {
        for len(q.inputStack) > 0 {
            q.outputStack = append(q.outputStack, q.inputStack[len(q.inputStack)-1])
            q.inputStack = q.inputStack[:len(q.inputStack)-1]
        }
    }
}

func (q *MyQueue) Pop() int {
    q.transfer()
    val := q.outputStack[len(q.outputStack)-1]
    q.outputStack = q.outputStack[:len(q.outputStack)-1]
    return val
}

func (q *MyQueue) Peek() int {
    q.transfer()
    return q.outputStack[len(q.outputStack)-1]
}

func (q *MyQueue) Empty() bool {
    return len(q.inputStack) == 0 && len(q.outputStack) == 0
}
```

### Сложность
- Push: O(1)
- Pop: O(1) амортизированно
- Space: O(n)

---

## На интервью

### Когда использовать

| Сигнал | Структура |
|--------|-----------|
| Matching (скобки, теги) | Stack |
| Следующий больший/меньший | Monotonic Stack |
| Вычисление выражений | Stack |
| FIFO обработка | Queue |
| BFS | Queue |

### Monotonic Stack шаблон

```python
# Следующий больший элемент
def next_greater(nums):
    result = [-1] * len(nums)
    stack = []  # Индексы

    for i in range(len(nums)):
        while stack and nums[i] > nums[stack[-1]]:
            idx = stack.pop()
            result[idx] = nums[i]
        stack.append(i)

    return result
```

### Типичные ошибки

1. Путают когда класть в стек значение vs индекс
2. Забывают обработать оставшиеся элементы в стеке
3. Неправильно определяют порядок операций (a - b vs b - a)

---

[← Назад к списку тем](README.md)
