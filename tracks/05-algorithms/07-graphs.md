# 07. Graphs

[← Назад к списку тем](README.md)

---

## Теория: Графы

### Представление графа

```python
# Adjacency List (самый частый)
graph = {
    'A': ['B', 'C'],
    'B': ['A', 'D'],
    'C': ['A', 'D'],
    'D': ['B', 'C']
}

# Adjacency Matrix
matrix = [
    [0, 1, 1, 0],
    [1, 0, 0, 1],
    [1, 0, 0, 1],
    [0, 1, 1, 0]
]

# Edge List
edges = [('A', 'B'), ('A', 'C'), ('B', 'D'), ('C', 'D')]
```

### BFS vs DFS

| | BFS | DFS |
|---|-----|-----|
| Структура | Queue | Stack/Recursion |
| Порядок | По уровням | В глубину |
| Кратчайший путь | Да (невзвешенный) | Нет |
| Память | O(width) | O(height) |

---

## Задача 1: Number of Islands

### Паттерн
DFS/BFS на сетке

### Условие
Посчитать количество островов (групп связанных '1') в матрице.

### Решение

```python
def num_islands(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return

        grid[r][c] = '0'  # Mark visited

        dfs(r + 1, c)
        dfs(r - 1, c)
        dfs(r, c + 1)
        dfs(r, c - 1)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                dfs(r, c)

    return count

# BFS версия
from collections import deque

def num_islands_bfs(grid: list[list[str]]) -> int:
    if not grid:
        return 0

    rows, cols = len(grid), len(grid[0])
    count = 0
    directions = [(1, 0), (-1, 0), (0, 1), (0, -1)]

    def bfs(r, c):
        queue = deque([(r, c)])
        grid[r][c] = '0'

        while queue:
            row, col = queue.popleft()
            for dr, dc in directions:
                nr, nc = row + dr, col + dc
                if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == '1':
                    grid[nr][nc] = '0'
                    queue.append((nr, nc))

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                bfs(r, c)

    return count
```

```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }

    rows, cols := len(grid), len(grid[0])
    count := 0

    var dfs func(r, c int)
    dfs = func(r, c int) {
        if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1' {
            return
        }

        grid[r][c] = '0'

        dfs(r+1, c)
        dfs(r-1, c)
        dfs(r, c+1)
        dfs(r, c-1)
    }

    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if grid[r][c] == '1' {
                count++
                dfs(r, c)
            }
        }
    }

    return count
}
```

### Сложность
- Time: O(rows × cols)
- Space: O(rows × cols) worst case для рекурсии

---

## Задача 2: Clone Graph

### Паттерн
DFS/BFS с hash map для mapping

### Условие
Глубокое копирование графа.

### Решение

```python
class Node:
    def __init__(self, val=0, neighbors=None):
        self.val = val
        self.neighbors = neighbors if neighbors else []

def clone_graph(node: Node) -> Node:
    if not node:
        return None

    cloned = {}  # original -> clone

    def dfs(n):
        if n in cloned:
            return cloned[n]

        clone = Node(n.val)
        cloned[n] = clone

        for neighbor in n.neighbors:
            clone.neighbors.append(dfs(neighbor))

        return clone

    return dfs(node)

# BFS версия
def clone_graph_bfs(node: Node) -> Node:
    if not node:
        return None

    cloned = {node: Node(node.val)}
    queue = deque([node])

    while queue:
        n = queue.popleft()
        for neighbor in n.neighbors:
            if neighbor not in cloned:
                cloned[neighbor] = Node(neighbor.val)
                queue.append(neighbor)
            cloned[n].neighbors.append(cloned[neighbor])

    return cloned[node]
```

```go
func cloneGraph(node *Node) *Node {
    if node == nil {
        return nil
    }

    cloned := make(map[*Node]*Node)

    var dfs func(*Node) *Node
    dfs = func(n *Node) *Node {
        if clone, ok := cloned[n]; ok {
            return clone
        }

        clone := &Node{Val: n.Val}
        cloned[n] = clone

        for _, neighbor := range n.Neighbors {
            clone.Neighbors = append(clone.Neighbors, dfs(neighbor))
        }

        return clone
    }

    return dfs(node)
}
```

### Сложность
- Time: O(V + E)
- Space: O(V)

---

## Задача 3: Course Schedule (Cycle Detection)

### Паттерн
Topological Sort / Cycle Detection

### Условие
Проверить, можно ли завершить все курсы (нет циклических зависимостей).

### Решение

```python
def can_finish(num_courses: int, prerequisites: list[list[int]]) -> bool:
    # Build adjacency list
    graph = [[] for _ in range(num_courses)]
    for course, prereq in prerequisites:
        graph[prereq].append(course)

    # 0 = unvisited, 1 = visiting, 2 = visited
    state = [0] * num_courses

    def has_cycle(course):
        if state[course] == 1:  # Cycle detected
            return True
        if state[course] == 2:  # Already processed
            return False

        state[course] = 1  # Mark as visiting

        for next_course in graph[course]:
            if has_cycle(next_course):
                return True

        state[course] = 2  # Mark as visited
        return False

    for course in range(num_courses):
        if has_cycle(course):
            return False

    return True

# BFS (Kahn's Algorithm)
from collections import deque

def can_finish_bfs(num_courses: int, prerequisites: list[list[int]]) -> bool:
    graph = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1

    # Start with courses having no prerequisites
    queue = deque([i for i in range(num_courses) if in_degree[i] == 0])
    completed = 0

    while queue:
        course = queue.popleft()
        completed += 1

        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    return completed == num_courses
```

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    graph := make([][]int, numCourses)
    inDegree := make([]int, numCourses)

    for _, prereq := range prerequisites {
        course, pre := prereq[0], prereq[1]
        graph[pre] = append(graph[pre], course)
        inDegree[course]++
    }

    queue := []int{}
    for i := 0; i < numCourses; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    completed := 0
    for len(queue) > 0 {
        course := queue[0]
        queue = queue[1:]
        completed++

        for _, next := range graph[course] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    return completed == numCourses
}
```

### Сложность
- Time: O(V + E)
- Space: O(V + E)

---

## Задача 4: Course Schedule II (Topological Sort)

### Паттерн
Topological Sort

### Условие
Вернуть порядок прохождения курсов.

### Решение

```python
def find_order(num_courses: int, prerequisites: list[list[int]]) -> list[int]:
    graph = [[] for _ in range(num_courses)]
    in_degree = [0] * num_courses

    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1

    queue = deque([i for i in range(num_courses) if in_degree[i] == 0])
    order = []

    while queue:
        course = queue.popleft()
        order.append(course)

        for next_course in graph[course]:
            in_degree[next_course] -= 1
            if in_degree[next_course] == 0:
                queue.append(next_course)

    return order if len(order) == num_courses else []
```

```go
func findOrder(numCourses int, prerequisites [][]int) []int {
    graph := make([][]int, numCourses)
    inDegree := make([]int, numCourses)

    for _, prereq := range prerequisites {
        course, pre := prereq[0], prereq[1]
        graph[pre] = append(graph[pre], course)
        inDegree[course]++
    }

    queue := []int{}
    for i := 0; i < numCourses; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    order := []int{}
    for len(queue) > 0 {
        course := queue[0]
        queue = queue[1:]
        order = append(order, course)

        for _, next := range graph[course] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    if len(order) == numCourses {
        return order
    }
    return []int{}
}
```

### Сложность
- Time: O(V + E)
- Space: O(V + E)

---

## Задача 5: Union-Find (Disjoint Set)

### Паттерн
Union-Find с path compression и rank

### Условие
Эффективно объединять множества и проверять принадлежность.

### Решение

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # Path compression
        return self.parent[x]

    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False  # Already connected

        # Union by rank
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1

        return True

    def connected(self, x, y):
        return self.find(x) == self.find(y)

# Пример: Number of Connected Components
def count_components(n: int, edges: list[list[int]]) -> int:
    uf = UnionFind(n)
    components = n

    for a, b in edges:
        if uf.union(a, b):
            components -= 1

    return components
```

```go
type UnionFind struct {
    parent []int
    rank   []int
}

func NewUnionFind(n int) *UnionFind {
    parent := make([]int, n)
    rank := make([]int, n)
    for i := range parent {
        parent[i] = i
    }
    return &UnionFind{parent, rank}
}

func (uf *UnionFind) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x])
    }
    return uf.parent[x]
}

func (uf *UnionFind) Union(x, y int) bool {
    px, py := uf.Find(x), uf.Find(y)
    if px == py {
        return false
    }

    if uf.rank[px] < uf.rank[py] {
        px, py = py, px
    }
    uf.parent[py] = px
    if uf.rank[px] == uf.rank[py] {
        uf.rank[px]++
    }
    return true
}
```

### Сложность
- Find/Union: O(α(n)) ≈ O(1) амортизированно
- Space: O(n)

---

## Задача 6: Shortest Path (Dijkstra)

### Паттерн
Dijkstra с heap

### Условие
Найти кратчайшие пути от источника до всех вершин во взвешенном графе.

### Решение

```python
import heapq

def dijkstra(graph: dict, start: str) -> dict:
    # graph = {'A': [('B', 1), ('C', 4)], ...}
    distances = {node: float('inf') for node in graph}
    distances[start] = 0

    heap = [(0, start)]  # (distance, node)

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

# Пример: Network Delay Time
def network_delay_time(times: list[list[int]], n: int, k: int) -> int:
    graph = {i: [] for i in range(1, n + 1)}
    for u, v, w in times:
        graph[u].append((v, w))

    distances = dijkstra(graph, k)

    max_dist = max(distances.values())
    return max_dist if max_dist < float('inf') else -1
```

```go
func dijkstra(graph map[int][][]int, start int) map[int]int {
    distances := make(map[int]int)
    for node := range graph {
        distances[node] = math.MaxInt32
    }
    distances[start] = 0

    // Min-heap: (distance, node)
    h := &IntHeap{{0, start}}
    heap.Init(h)

    for h.Len() > 0 {
        item := heap.Pop(h).([]int)
        dist, node := item[0], item[1]

        if dist > distances[node] {
            continue
        }

        for _, edge := range graph[node] {
            neighbor, weight := edge[0], edge[1]
            newDist := dist + weight
            if newDist < distances[neighbor] {
                distances[neighbor] = newDist
                heap.Push(h, []int{newDist, neighbor})
            }
        }
    }

    return distances
}

// IntHeap implementation for min-heap
type IntHeap [][]int
func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i][0] < h[j][0] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.([]int)) }
func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}
```

### Сложность
- Time: O((V + E) log V)
- Space: O(V)

---

## См. также

- [Ride Sharing (Uber/Lyft)](../03-system-design/12-ride-sharing.md) — применение графов для поиска маршрутов
- [Динамическое программирование](./08-dynamic-programming.md) — задачи на графах с оптимизацией

---

## На интервью

### Выбор алгоритма

| Задача | Алгоритм |
|--------|----------|
| Обход всех вершин | DFS/BFS |
| Кратчайший путь (невзвешенный) | BFS |
| Кратчайший путь (взвешенный) | Dijkstra |
| Цикл в направленном графе | DFS с состояниями |
| Порядок зависимостей | Topological Sort |
| Связные компоненты | Union-Find / DFS |

### Типичные ошибки

1. Забывают отмечать посещённые вершины
2. Путают направленный и ненаправленный граф
3. Не обрабатывают несвязный граф
4. Используют DFS вместо BFS для кратчайшего пути

---

[← Назад к списку тем](README.md)
