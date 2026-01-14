# 06. Trees & Traversal

[← Назад к списку тем](README.md)

---

## Теория: Деревья

### Структура узла

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}
```

### Типы обходов

| Обход | Порядок | Применение |
|-------|---------|------------|
| Preorder | Root → Left → Right | Копирование дерева |
| Inorder | Left → Root → Right | BST: отсортированный порядок |
| Postorder | Left → Right → Root | Удаление дерева |
| Level order | По уровням (BFS) | Поиск по ширине |

---

## Задача 1: Binary Tree Traversals

### Паттерн
DFS (рекурсия / стек)

### Решение

```python
# Рекурсивные версии
def preorder(root: TreeNode) -> list[int]:
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

def inorder(root: TreeNode) -> list[int]:
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

def postorder(root: TreeNode) -> list[int]:
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]

# Итеративные версии
def preorder_iterative(root: TreeNode) -> list[int]:
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)
        # Right first, then left (stack is LIFO)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result

def inorder_iterative(root: TreeNode) -> list[int]:
    result = []
    stack = []
    current = root

    while current or stack:
        # Go to the leftmost node
        while current:
            stack.append(current)
            current = current.left

        current = stack.pop()
        result.append(current.val)
        current = current.right

    return result

def postorder_iterative(root: TreeNode) -> list[int]:
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)

    return result[::-1]  # Reverse
```

```go
func inorderIterative(root *TreeNode) []int {
    result := []int{}
    stack := []*TreeNode{}
    current := root

    for current != nil || len(stack) > 0 {
        for current != nil {
            stack = append(stack, current)
            current = current.Left
        }

        current = stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        result = append(result, current.Val)
        current = current.Right
    }

    return result
}
```

### Сложность
- Time: O(n)
- Space: O(h), где h — высота дерева

---

## Задача 2: Level Order Traversal

### Паттерн
BFS с очередью

### Условие
Обойти дерево по уровням, вернуть список уровней.

### Решение

```python
from collections import deque

def level_order(root: TreeNode) -> list[list[int]]:
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result

# Пример
#     3
#    / \
#   9  20
#     /  \
#    15   7
# Результат: [[3], [9, 20], [15, 7]]
```

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }

    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        levelSize := len(queue)
        level := make([]int, 0, levelSize)

        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }

        result = append(result, level)
    }

    return result
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 3: Maximum Depth of Binary Tree

### Паттерн
DFS / BFS

### Решение

```python
# DFS рекурсивно
def max_depth(root: TreeNode) -> int:
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))

# BFS
def max_depth_bfs(root: TreeNode) -> int:
    if not root:
        return 0

    depth = 0
    queue = deque([root])

    while queue:
        depth += 1
        for _ in range(len(queue)):
            node = queue.popleft()
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return depth
```

```go
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }

    left := maxDepth(root.Left)
    right := maxDepth(root.Right)

    if left > right {
        return 1 + left
    }
    return 1 + right
}
```

### Сложность
- Time: O(n)
- Space: O(h)

---

## Задача 4: Validate Binary Search Tree

### Паттерн
DFS с границами

### Условие
Проверить, что дерево является валидным BST.

### Интуиция
Для каждого узла проверяем, что значение в допустимом диапазоне (min, max).

### Решение

```python
def is_valid_bst(root: TreeNode) -> bool:
    def validate(node, min_val, max_val):
        if not node:
            return True

        if not (min_val < node.val < max_val):
            return False

        return (validate(node.left, min_val, node.val) and
                validate(node.right, node.val, max_val))

    return validate(root, float('-inf'), float('inf'))

# Альтернатива: inorder должен быть отсортирован
def is_valid_bst_inorder(root: TreeNode) -> bool:
    prev = float('-inf')

    def inorder(node):
        nonlocal prev
        if not node:
            return True

        if not inorder(node.left):
            return False

        if node.val <= prev:
            return False
        prev = node.val

        return inorder(node.right)

    return inorder(root)
```

```go
func isValidBST(root *TreeNode) bool {
    return validate(root, nil, nil)
}

func validate(node *TreeNode, min, max *int) bool {
    if node == nil {
        return true
    }

    if min != nil && node.Val <= *min {
        return false
    }
    if max != nil && node.Val >= *max {
        return false
    }

    return validate(node.Left, min, &node.Val) &&
           validate(node.Right, &node.Val, max)
}
```

### Сложность
- Time: O(n)
- Space: O(h)

---

## Задача 5: Lowest Common Ancestor

### Паттерн
DFS с возвратом

### Условие
Найти наименьшего общего предка двух узлов.

### Интуиция
Если находим p или q — возвращаем его. Если оба потомка вернули не-null — текущий узел LCA.

### Решение

```python
def lowest_common_ancestor(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    if not root or root == p or root == q:
        return root

    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)

    if left and right:
        return root  # p и q в разных поддеревьях

    return left if left else right

# Для BST — проще
def lca_bst(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root
    return None
```

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }

    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)

    if left != nil && right != nil {
        return root
    }

    if left != nil {
        return left
    }
    return right
}
```

### Сложность
- Time: O(n)
- Space: O(h)

---

## Задача 6: Serialize and Deserialize Binary Tree

### Паттерн
Preorder + null markers

### Условие
Сериализовать и десериализовать бинарное дерево.

### Решение

```python
class Codec:
    def serialize(self, root: TreeNode) -> str:
        def preorder(node):
            if not node:
                return ["null"]
            return [str(node.val)] + preorder(node.left) + preorder(node.right)

        return ",".join(preorder(root))

    def deserialize(self, data: str) -> TreeNode:
        values = iter(data.split(","))

        def build():
            val = next(values)
            if val == "null":
                return None

            node = TreeNode(int(val))
            node.left = build()
            node.right = build()
            return node

        return build()

# Пример
#     1
#    / \
#   2   3
#      / \
#     4   5
# Сериализация: "1,2,null,null,3,4,null,null,5,null,null"
```

```go
type Codec struct{}

func (c *Codec) serialize(root *TreeNode) string {
    var result []string
    var preorder func(*TreeNode)
    preorder = func(node *TreeNode) {
        if node == nil {
            result = append(result, "null")
            return
        }
        result = append(result, strconv.Itoa(node.Val))
        preorder(node.Left)
        preorder(node.Right)
    }
    preorder(root)
    return strings.Join(result, ",")
}

func (c *Codec) deserialize(data string) *TreeNode {
    values := strings.Split(data, ",")
    idx := 0

    var build func() *TreeNode
    build = func() *TreeNode {
        if values[idx] == "null" {
            idx++
            return nil
        }
        val, _ := strconv.Atoi(values[idx])
        idx++
        node := &TreeNode{Val: val}
        node.Left = build()
        node.Right = build()
        return node
    }

    return build()
}
```

### Сложность
- Time: O(n)
- Space: O(n)

---

## Задача 7: Invert Binary Tree

### Паттерн
DFS / BFS

### Условие
Инвертировать (отзеркалить) бинарное дерево.

### Решение

```python
def invert_tree(root: TreeNode) -> TreeNode:
    if not root:
        return None

    root.left, root.right = root.right, root.left

    invert_tree(root.left)
    invert_tree(root.right)

    return root

# BFS версия
def invert_tree_bfs(root: TreeNode) -> TreeNode:
    if not root:
        return None

    queue = deque([root])

    while queue:
        node = queue.popleft()
        node.left, node.right = node.right, node.left

        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)

    return root
```

```go
func invertTree(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }

    root.Left, root.Right = root.Right, root.Left

    invertTree(root.Left)
    invertTree(root.Right)

    return root
}
```

### Сложность
- Time: O(n)
- Space: O(h)

---

## На интервью

### Выбор подхода

| Задача | Подход |
|--------|--------|
| Обход дерева | DFS (рекурсия/стек) или BFS |
| Поиск по уровням | BFS |
| Глубина/высота | DFS |
| Путь от корня | DFS с путём |
| BST поиск | Итеративно O(h) |

### Типичные ошибки

1. Забывают обработать null корень
2. Путают порядок в preorder/inorder/postorder
3. Не учитывают, что в BST все значения слева меньше, справа больше (не только непосредственные дети)
4. Забывают про stack overflow при глубокой рекурсии

### Шаблон DFS

```python
def dfs(node, ...):
    if not node:
        return base_case

    # Preorder: обработка до рекурсии

    left_result = dfs(node.left, ...)
    # Inorder: обработка между рекурсиями
    right_result = dfs(node.right, ...)

    # Postorder: обработка после рекурсии

    return combine(left_result, right_result)
```

---

[← Назад к списку тем](README.md)
