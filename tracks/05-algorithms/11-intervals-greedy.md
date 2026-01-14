# 11. Intervals & Greedy

[← Назад к списку тем](README.md)

---

## Теория: Интервалы

### Типичные операции
1. Сортировка по началу или концу
2. Проверка пересечения: `a.start < b.end and b.start < a.end`
3. Слияние: `[min(a.start, b.start), max(a.end, b.end)]`

### Greedy подход
Делаем локально оптимальный выбор на каждом шаге, надеясь на глобально оптимальное решение.

**Когда работает:**
- Оптимальная подструктура
- Жадный выбор не отменяет будущие возможности

---

## Задача 1: Merge Intervals

### Паттерн
Sort + Linear merge

### Условие
Слить пересекающиеся интервалы.

### Решение

```python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    if not intervals:
        return []

    # Сортируем по началу
    intervals.sort(key=lambda x: x[0])

    result = [intervals[0]]

    for start, end in intervals[1:]:
        # Если пересекается с последним
        if start <= result[-1][1]:
            result[-1][1] = max(result[-1][1], end)
        else:
            result.append([start, end])

    return result

# Пример
intervals = [[1,3], [2,6], [8,10], [15,18]]
print(merge(intervals))  # [[1,6], [8,10], [15,18]]
```

```go
func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 {
        return nil
    }

    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    result := [][]int{intervals[0]}

    for _, interval := range intervals[1:] {
        last := result[len(result)-1]
        if interval[0] <= last[1] {
            if interval[1] > last[1] {
                last[1] = interval[1]
            }
        } else {
            result = append(result, interval)
        }
    }

    return result
}
```

### Сложность
- Time: O(n log n)
- Space: O(n)

---

## Задача 2: Insert Interval

### Паттерн
Find position + Merge

### Условие
Вставить новый интервал в отсортированный список и слить при необходимости.

### Решение

```python
def insert(intervals: list[list[int]], newInterval: list[int]) -> list[list[int]]:
    result = []
    i = 0
    n = len(intervals)

    # Добавляем интервалы до newInterval
    while i < n and intervals[i][1] < newInterval[0]:
        result.append(intervals[i])
        i += 1

    # Сливаем пересекающиеся
    while i < n and intervals[i][0] <= newInterval[1]:
        newInterval[0] = min(newInterval[0], intervals[i][0])
        newInterval[1] = max(newInterval[1], intervals[i][1])
        i += 1

    result.append(newInterval)

    # Добавляем оставшиеся
    while i < n:
        result.append(intervals[i])
        i += 1

    return result

# Пример
intervals = [[1,3], [6,9]]
newInterval = [2,5]
print(insert(intervals, newInterval))  # [[1,5], [6,9]]
```

```go
func insert(intervals [][]int, newInterval []int) [][]int {
    result := [][]int{}
    i := 0
    n := len(intervals)

    // Before newInterval
    for i < n && intervals[i][1] < newInterval[0] {
        result = append(result, intervals[i])
        i++
    }

    // Merge overlapping
    for i < n && intervals[i][0] <= newInterval[1] {
        newInterval[0] = min(newInterval[0], intervals[i][0])
        newInterval[1] = max(newInterval[1], intervals[i][1])
        i++
    }
    result = append(result, newInterval)

    // After newInterval
    for i < n {
        result = append(result, intervals[i])
        i++
    }

    return result
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 3: Non-overlapping Intervals

### Паттерн
Greedy — минимизация удалений

### Условие
Найти минимальное количество интервалов для удаления, чтобы остальные не пересекались.

### Интуиция
Сортируем по концу. Жадно выбираем интервалы с наименьшим концом — они оставляют больше места для следующих.

### Решение

```python
def erase_overlap_intervals(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0

    # Сортируем по концу
    intervals.sort(key=lambda x: x[1])

    count = 0
    prev_end = float('-inf')

    for start, end in intervals:
        if start >= prev_end:
            # Не пересекается — берём
            prev_end = end
        else:
            # Пересекается — удаляем (увеличиваем count)
            count += 1

    return count

# Альтернатива: считаем сколько оставить
def erase_overlap_intervals_v2(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0

    intervals.sort(key=lambda x: x[1])

    kept = 1
    prev_end = intervals[0][1]

    for start, end in intervals[1:]:
        if start >= prev_end:
            kept += 1
            prev_end = end

    return len(intervals) - kept

# Пример
intervals = [[1,2], [2,3], [3,4], [1,3]]
print(erase_overlap_intervals(intervals))  # 1 (удаляем [1,3])
```

```go
func eraseOverlapIntervals(intervals [][]int) int {
    if len(intervals) == 0 {
        return 0
    }

    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][1] < intervals[j][1]
    })

    count := 0
    prevEnd := intervals[0][1]

    for i := 1; i < len(intervals); i++ {
        if intervals[i][0] >= prevEnd {
            prevEnd = intervals[i][1]
        } else {
            count++
        }
    }

    return count
}
```

### Сложность
- Time: O(n log n)
- Space: O(1)

---

## Задача 4: Meeting Rooms II

### Паттерн
Min-heap / Sweep line

### Условие
Найти минимальное количество переговорных для всех встреч.

### Решение

```python
import heapq

def min_meeting_rooms(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0

    # Сортируем по началу
    intervals.sort(key=lambda x: x[0])

    # Min-heap — времена окончания занятых комнат
    heap = []

    for start, end in intervals:
        # Освобождаем комнату если она освободилась
        if heap and heap[0] <= start:
            heapq.heappop(heap)

        # Занимаем комнату
        heapq.heappush(heap, end)

    return len(heap)

# Sweep line approach
def min_meeting_rooms_sweep(intervals: list[list[int]]) -> int:
    events = []
    for start, end in intervals:
        events.append((start, 1))   # Meeting starts
        events.append((end, -1))    # Meeting ends

    events.sort()

    rooms = 0
    max_rooms = 0

    for _, delta in events:
        rooms += delta
        max_rooms = max(max_rooms, rooms)

    return max_rooms

# Пример
intervals = [[0,30], [5,10], [15,20]]
print(min_meeting_rooms(intervals))  # 2
```

```go
func minMeetingRooms(intervals [][]int) int {
    if len(intervals) == 0 {
        return 0
    }

    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    h := &IntMinHeap{}
    heap.Init(h)

    for _, interval := range intervals {
        if h.Len() > 0 && (*h)[0] <= interval[0] {
            heap.Pop(h)
        }
        heap.Push(h, interval[1])
    }

    return h.Len()
}
```

### Сложность
- Heap: O(n log n)
- Sweep line: O(n log n)
- Space: O(n)

---

## Задача 5: Minimum Number of Arrows

### Паттерн
Greedy — максимальное покрытие

### Условие
Минимальное количество стрел, чтобы лопнуть все шарики (интервалы).

### Решение

```python
def find_min_arrow_shots(points: list[list[int]]) -> int:
    if not points:
        return 0

    # Сортируем по концу
    points.sort(key=lambda x: x[1])

    arrows = 1
    arrow_pos = points[0][1]

    for start, end in points[1:]:
        # Если шарик не лопнут текущей стрелой
        if start > arrow_pos:
            arrows += 1
            arrow_pos = end

    return arrows

# Пример
points = [[10,16], [2,8], [1,6], [7,12]]
print(find_min_arrow_shots(points))  # 2
```

```go
func findMinArrowShots(points [][]int) int {
    if len(points) == 0 {
        return 0
    }

    sort.Slice(points, func(i, j int) bool {
        return points[i][1] < points[j][1]
    })

    arrows := 1
    arrowPos := points[0][1]

    for i := 1; i < len(points); i++ {
        if points[i][0] > arrowPos {
            arrows++
            arrowPos = points[i][1]
        }
    }

    return arrows
}
```

### Сложность
- Time: O(n log n)
- Space: O(1)

---

## Задача 6: Jump Game

### Паттерн
Greedy — максимальный reach

### Условие
Можно ли допрыгнуть до конца массива?

### Решение

```python
def can_jump(nums: list[int]) -> bool:
    max_reach = 0

    for i, jump in enumerate(nums):
        if i > max_reach:
            return False
        max_reach = max(max_reach, i + jump)

    return True

# Jump Game II — минимальное количество прыжков
def jump(nums: list[int]) -> int:
    jumps = 0
    current_end = 0
    farthest = 0

    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])

        if i == current_end:
            jumps += 1
            current_end = farthest

    return jumps

# Пример
nums = [2, 3, 1, 1, 4]
print(can_jump(nums))  # True
print(jump(nums))      # 2
```

```go
func canJump(nums []int) bool {
    maxReach := 0

    for i, jump := range nums {
        if i > maxReach {
            return false
        }
        if i+jump > maxReach {
            maxReach = i + jump
        }
    }

    return true
}

func jump(nums []int) int {
    jumps := 0
    currentEnd := 0
    farthest := 0

    for i := 0; i < len(nums)-1; i++ {
        if i+nums[i] > farthest {
            farthest = i + nums[i]
        }

        if i == currentEnd {
            jumps++
            currentEnd = farthest
        }
    }

    return jumps
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 7: Gas Station

### Паттерн
Greedy — выбор стартовой точки

### Условие
Найти стартовую заправку для кругового маршрута.

### Решение

```python
def can_complete_circuit(gas: list[int], cost: list[int]) -> int:
    total_tank = 0
    current_tank = 0
    start = 0

    for i in range(len(gas)):
        diff = gas[i] - cost[i]
        total_tank += diff
        current_tank += diff

        if current_tank < 0:
            # Не можем доехать из start до i+1
            # Начинаем с i+1
            start = i + 1
            current_tank = 0

    return start if total_tank >= 0 else -1

# Пример
gas = [1, 2, 3, 4, 5]
cost = [3, 4, 5, 1, 2]
print(can_complete_circuit(gas, cost))  # 3
```

```go
func canCompleteCircuit(gas []int, cost []int) int {
    totalTank := 0
    currentTank := 0
    start := 0

    for i := 0; i < len(gas); i++ {
        diff := gas[i] - cost[i]
        totalTank += diff
        currentTank += diff

        if currentTank < 0 {
            start = i + 1
            currentTank = 0
        }
    }

    if totalTank >= 0 {
        return start
    }
    return -1
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## На интервью

### Паттерны интервалов

| Задача | Сортировка | Подход |
|--------|------------|--------|
| Merge | По началу | Linear scan |
| Max non-overlapping | По концу | Greedy |
| Meeting rooms | По началу | Min-heap |
| Insert | Уже отсортировано | 3 фазы |

### Greedy checklist

1. **Можно ли применить greedy?**
   - Локальный оптимум → глобальный оптимум?
   - Нет "отката" решений?

2. **По чему сортировать?**
   - По началу: когда важен порядок событий
   - По концу: когда важно освободить место раньше

3. **Что отслеживать?**
   - Последний конец
   - Текущий максимум
   - Количество активных

### Типичные ошибки

1. Сортировка по неправильному критерию
2. Неправильная обработка границ (включительно/исключительно)
3. Забывают про пустой вход
4. Не проверяют доказуемость greedy подхода

---

[← Назад к списку тем](README.md)
