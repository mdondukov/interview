# 10. Heap & Priority Queue

[← Назад к списку тем](README.md)

---

## Теория: Heap

### Свойства
- **Min-heap:** parent ≤ children (корень — минимум)
- **Max-heap:** parent ≥ children (корень — максимум)
- **Complete binary tree:** заполняется слева направо

### Операции

| Операция | Сложность |
|----------|-----------|
| Insert (push) | O(log n) |
| Extract min/max (pop) | O(log n) |
| Get min/max (peek) | O(1) |
| Heapify array | O(n) |

### Python heapq

```python
import heapq

# Min-heap по умолчанию
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
print(heapq.heappop(heap))  # 1 (минимум)

# Max-heap: инвертируем значения
heapq.heappush(heap, -5)  # Храним -5 вместо 5

# Heapify
arr = [3, 1, 4, 1, 5]
heapq.heapify(arr)  # O(n)
```

---

## Задача 1: Kth Largest Element

### Паттерн
Min-heap размера k

### Условие
Найти k-й по величине элемент в массиве.

### Решение

```python
import heapq

def find_kth_largest(nums: list[int], k: int) -> int:
    # Min-heap размера k
    # Корень = k-й наибольший
    heap = []

    for num in nums:
        heapq.heappush(heap, num)
        if len(heap) > k:
            heapq.heappop(heap)

    return heap[0]

# Альтернатива: nlargest
def find_kth_largest_v2(nums: list[int], k: int) -> int:
    return heapq.nlargest(k, nums)[-1]

# QuickSelect — O(n) average
def find_kth_largest_quickselect(nums: list[int], k: int) -> int:
    k = len(nums) - k  # k-th largest = (n-k)-th smallest

    def quickselect(left, right):
        pivot = nums[right]
        p = left

        for i in range(left, right):
            if nums[i] <= pivot:
                nums[i], nums[p] = nums[p], nums[i]
                p += 1

        nums[p], nums[right] = nums[right], nums[p]

        if p < k:
            return quickselect(p + 1, right)
        elif p > k:
            return quickselect(left, p - 1)
        else:
            return nums[p]

    return quickselect(0, len(nums) - 1)

# Пример
nums = [3, 2, 1, 5, 6, 4]
k = 2
print(find_kth_largest(nums, k))  # 5
```

```go
import "container/heap"

type IntHeap []int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func findKthLargest(nums []int, k int) int {
    h := &IntHeap{}
    heap.Init(h)

    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    return (*h)[0]
}
```

### Сложность
- Heap: Time O(n log k), Space O(k)
- QuickSelect: Time O(n) average, O(n²) worst, Space O(1)

---

## Задача 2: Top K Frequent Elements

### Паттерн
Counting + Min-heap

### Условие
Найти k самых частых элементов.

### Решение

```python
from collections import Counter
import heapq

def top_k_frequent(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)

    # Min-heap по частоте, размер k
    heap = []

    for num, freq in count.items():
        heapq.heappush(heap, (freq, num))
        if len(heap) > k:
            heapq.heappop(heap)

    return [num for freq, num in heap]

# Bucket sort — O(n)
def top_k_frequent_bucket(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)

    # bucket[i] = elements with frequency i
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

    result := []int{}
    for i := len(buckets) - 1; i >= 0 && len(result) < k; i-- {
        result = append(result, buckets[i]...)
    }

    return result[:k]
}
```

### Сложность
- Heap: O(n log k)
- Bucket: O(n)

---

## Задача 3: Merge K Sorted Lists

### Паттерн
Min-heap для k-way merge

### Условие
Слить k отсортированных списков в один.

### Решение

```python
import heapq

class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def merge_k_lists(lists: list[ListNode]) -> ListNode:
    # Для сравнения в heap
    heap = []

    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))

    dummy = ListNode(0)
    current = dummy

    while heap:
        val, i, node = heapq.heappop(heap)
        current.next = node
        current = current.next

        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next
```

```go
func mergeKLists(lists []*ListNode) *ListNode {
    h := &ListNodeHeap{}
    heap.Init(h)

    for _, node := range lists {
        if node != nil {
            heap.Push(h, node)
        }
    }

    dummy := &ListNode{}
    current := dummy

    for h.Len() > 0 {
        node := heap.Pop(h).(*ListNode)
        current.Next = node
        current = current.Next

        if node.Next != nil {
            heap.Push(h, node.Next)
        }
    }

    return dummy.Next
}

type ListNodeHeap []*ListNode
func (h ListNodeHeap) Len() int           { return len(h) }
func (h ListNodeHeap) Less(i, j int) bool { return h[i].Val < h[j].Val }
func (h ListNodeHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *ListNodeHeap) Push(x any)        { *h = append(*h, x.(*ListNode)) }
func (h *ListNodeHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}
```

### Сложность
- Time: O(N log k), где N — общее количество элементов
- Space: O(k)

---

## Задача 4: Find Median from Data Stream

### Паттерн
Two heaps (max-heap + min-heap)

### Условие
Поддерживать медиану потока чисел.

### Интуиция
- Max-heap для меньшей половины (корень = макс меньшей части)
- Min-heap для большей половины (корень = мин большей части)
- Балансируем размеры

### Решение

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []  # Max-heap (храним отрицательные)
        self.large = []  # Min-heap

    def addNum(self, num: int) -> None:
        # Добавляем в small (max-heap)
        heapq.heappush(self.small, -num)

        # Перемещаем максимум small в large
        heapq.heappush(self.large, -heapq.heappop(self.small))

        # Балансируем размеры
        if len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self) -> float:
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2

# Пример
mf = MedianFinder()
mf.addNum(1)
mf.addNum(2)
print(mf.findMedian())  # 1.5
mf.addNum(3)
print(mf.findMedian())  # 2
```

```go
type MedianFinder struct {
    small *IntMaxHeap // max-heap for smaller half
    large *IntMinHeap // min-heap for larger half
}

func Constructor() MedianFinder {
    return MedianFinder{
        small: &IntMaxHeap{},
        large: &IntMinHeap{},
    }
}

func (mf *MedianFinder) AddNum(num int) {
    heap.Push(mf.small, num)
    heap.Push(mf.large, heap.Pop(mf.small).(int))

    if mf.large.Len() > mf.small.Len() {
        heap.Push(mf.small, heap.Pop(mf.large).(int))
    }
}

func (mf *MedianFinder) FindMedian() float64 {
    if mf.small.Len() > mf.large.Len() {
        return float64((*mf.small)[0])
    }
    return float64((*mf.small)[0]+(*mf.large)[0]) / 2.0
}
```

### Сложность
- addNum: O(log n)
- findMedian: O(1)
- Space: O(n)

---

## Задача 5: Task Scheduler

### Паттерн
Greedy + Max-heap

### Условие
Выполнить задачи с cooldown n между одинаковыми задачами. Найти минимальное время.

### Решение

```python
from collections import Counter
import heapq

def least_interval(tasks: list[str], n: int) -> int:
    count = Counter(tasks)
    max_heap = [-c for c in count.values()]
    heapq.heapify(max_heap)

    time = 0
    queue = []  # (count, available_time)

    while max_heap or queue:
        time += 1

        if max_heap:
            cnt = heapq.heappop(max_heap) + 1  # Execute task
            if cnt < 0:
                queue.append((cnt, time + n))

        if queue and queue[0][1] == time:
            heapq.heappush(max_heap, queue.pop(0)[0])

    return time

# Математическое решение O(n)
def least_interval_math(tasks: list[str], n: int) -> int:
    count = Counter(tasks)
    max_freq = max(count.values())
    max_count = sum(1 for freq in count.values() if freq == max_freq)

    # (max_freq - 1) полных циклов по (n + 1) + max_count последних задач
    return max(len(tasks), (max_freq - 1) * (n + 1) + max_count)
```

```go
func leastInterval(tasks []byte, n int) int {
    count := make(map[byte]int)
    for _, task := range tasks {
        count[task]++
    }

    maxFreq := 0
    maxCount := 0
    for _, freq := range count {
        if freq > maxFreq {
            maxFreq = freq
            maxCount = 1
        } else if freq == maxFreq {
            maxCount++
        }
    }

    result := (maxFreq-1)*(n+1) + maxCount
    if len(tasks) > result {
        return len(tasks)
    }
    return result
}
```

### Сложность
- Heap: O(n log 26) = O(n)
- Math: O(n)

---

## Задача 6: K Closest Points to Origin

### Паттерн
Max-heap размера k

### Условие
Найти k ближайших точек к началу координат.

### Решение

```python
import heapq

def k_closest(points: list[list[int]], k: int) -> list[list[int]]:
    # Max-heap по расстоянию (отрицательное)
    heap = []

    for x, y in points:
        dist = -(x * x + y * y)
        heapq.heappush(heap, (dist, x, y))
        if len(heap) > k:
            heapq.heappop(heap)

    return [[x, y] for _, x, y in heap]

# Альтернатива: nsmallest
def k_closest_v2(points: list[list[int]], k: int) -> list[list[int]]:
    return heapq.nsmallest(k, points, key=lambda p: p[0]**2 + p[1]**2)
```

```go
func kClosest(points [][]int, k int) [][]int {
    h := &MaxDistHeap{}
    heap.Init(h)

    for _, p := range points {
        dist := p[0]*p[0] + p[1]*p[1]
        heap.Push(h, []int{dist, p[0], p[1]})
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    result := make([][]int, k)
    for i := 0; i < k; i++ {
        item := (*h)[i]
        result[i] = []int{item[1], item[2]}
    }
    return result
}
```

### Сложность
- Time: O(n log k)
- Space: O(k)

---

## См. также

- [Графы](./07-graphs.md) — алгоритм Dijkstra использует min-heap
- [Мониторинг и алертинг](../03-system-design/14-monitoring-alerting.md) — приоритетные очереди для обработки событий

---

## На интервью

### Когда использовать heap

| Сигнал | Паттерн |
|--------|---------|
| "K наибольших/наименьших" | Heap размера k |
| "Медиана потока" | Two heaps |
| "Merge K sorted" | Min-heap |
| "Scheduling с приоритетами" | Priority queue |

### Типичные ошибки

1. Путают min-heap и max-heap в Python (heapq — min-heap)
2. Забывают про (priority, tie_breaker) для стабильности
3. Используют heap когда достаточно сортировки
4. Не оптимизируют до O(k) space когда возможно

### Python heapq tips

```python
# Max-heap через отрицательные значения
heapq.heappush(heap, -value)
max_val = -heapq.heappop(heap)

# Heap с объектами
heapq.heappush(heap, (priority, index, object))  # index для tie-breaking

# Top K
heapq.nlargest(k, iterable)
heapq.nsmallest(k, iterable)
```

---

[← Назад к списку тем](README.md)
