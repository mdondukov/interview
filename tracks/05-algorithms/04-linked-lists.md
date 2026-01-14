# 04. Linked Lists

[← Назад к списку тем](README.md)

---

## Теория: Связные списки

### Структура узла

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

```go
type ListNode struct {
    Val  int
    Next *ListNode
}
```

### Ключевые техники

1. **Dummy node:** упрощает работу с головой списка
2. **Two pointers:** slow/fast для нахождения середины, цикла
3. **In-place reversal:** разворот без дополнительной памяти

---

## Задача 1: Reverse Linked List

### Паттерн
In-place Reversal

### Условие
Развернуть односвязный список.

### Решение

```python
def reverse_list(head: ListNode) -> ListNode:
    prev = None
    curr = head

    while curr:
        next_temp = curr.next  # Сохраняем следующий
        curr.next = prev       # Разворачиваем ссылку
        prev = curr            # Двигаем prev
        curr = next_temp       # Двигаем curr

    return prev

# Рекурсивное решение
def reverse_list_recursive(head: ListNode) -> ListNode:
    if not head or not head.next:
        return head

    new_head = reverse_list_recursive(head.next)
    head.next.next = head
    head.next = None

    return new_head
```

```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head

    for curr != nil {
        nextTemp := curr.Next
        curr.Next = prev
        prev = curr
        curr = nextTemp
    }

    return prev
}
```

### Сложность
- Time: O(n)
- Space: O(1) итеративно, O(n) рекурсивно

---

## Задача 2: Merge Two Sorted Lists

### Паттерн
Dummy Node + Two Pointers

### Условие
Слить два отсортированных списка в один.

### Решение

```python
def merge_two_lists(l1: ListNode, l2: ListNode) -> ListNode:
    dummy = ListNode(0)
    current = dummy

    while l1 and l2:
        if l1.val <= l2.val:
            current.next = l1
            l1 = l1.next
        else:
            current.next = l2
            l2 = l2.next
        current = current.next

    # Присоединяем оставшийся список
    current.next = l1 if l1 else l2

    return dummy.next
```

```go
func mergeTwoLists(l1, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    current := dummy

    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            current.Next = l1
            l1 = l1.Next
        } else {
            current.Next = l2
            l2 = l2.Next
        }
        current = current.Next
    }

    if l1 != nil {
        current.Next = l1
    } else {
        current.Next = l2
    }

    return dummy.Next
}
```

### Сложность
- Time: O(n + m)
- Space: O(1)

---

## Задача 3: Linked List Cycle

### Паттерн
Fast & Slow Pointers (Floyd's Cycle Detection)

### Условие
Определить, есть ли цикл в списке.

### Интуиция
Быстрый указатель догонит медленный если есть цикл (как бегуны на круговой дорожке).

### Решение

```python
def has_cycle(head: ListNode) -> bool:
    if not head or not head.next:
        return False

    slow = head
    fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

        if slow == fast:
            return True

    return False

# Найти начало цикла
def detect_cycle(head: ListNode) -> ListNode:
    if not head or not head.next:
        return None

    slow = fast = head

    # Находим точку встречи
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

        if slow == fast:
            break
    else:
        return None  # Нет цикла

    # Находим начало цикла
    slow = head
    while slow != fast:
        slow = slow.next
        fast = fast.next

    return slow
```

```go
func hasCycle(head *ListNode) bool {
    if head == nil || head.Next == nil {
        return false
    }

    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next

        if slow == fast {
            return true
        }
    }
    return false
}

func detectCycle(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return nil
    }

    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next

        if slow == fast {
            slow = head
            for slow != fast {
                slow = slow.Next
                fast = fast.Next
            }
            return slow
        }
    }
    return nil
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 4: Find Middle of Linked List

### Паттерн
Fast & Slow Pointers

### Условие
Найти середину списка. При чётной длине — второй из двух средних.

### Решение

```python
def middle_node(head: ListNode) -> ListNode:
    slow = fast = head

    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    return slow

# Для первого среднего при чётной длине
def middle_node_first(head: ListNode) -> ListNode:
    slow = fast = head

    while fast.next and fast.next.next:
        slow = slow.next
        fast = fast.next.next

    return slow
```

```go
func middleNode(head *ListNode) *ListNode {
    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    return slow
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 5: Remove Nth Node From End

### Паттерн
Two Pointers с отступом

### Условие
Удалить n-й узел с конца списка за один проход.

### Интуиция
Первый указатель уходит на n шагов вперёд. Затем оба двигаются вместе. Когда первый дойдёт до конца, второй будет на n-м с конца.

### Решение

```python
def remove_nth_from_end(head: ListNode, n: int) -> ListNode:
    dummy = ListNode(0, head)
    first = second = dummy

    # Первый уходит на n+1 шагов
    for _ in range(n + 1):
        first = first.next

    # Оба двигаются до конца
    while first:
        first = first.next
        second = second.next

    # Удаляем узел
    second.next = second.next.next

    return dummy.next
```

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    dummy := &ListNode{Next: head}
    first, second := dummy, dummy

    for i := 0; i <= n; i++ {
        first = first.Next
    }

    for first != nil {
        first = first.Next
        second = second.Next
    }

    second.Next = second.Next.Next

    return dummy.Next
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 6: Reorder List

### Паттерн
Find Middle + Reverse + Merge

### Условие
Переупорядочить список: L0 → Ln → L1 → Ln-1 → L2 → Ln-2 → ...

### Решение

```python
def reorder_list(head: ListNode) -> None:
    if not head or not head.next:
        return

    # 1. Найти середину
    slow = fast = head
    while fast.next and fast.next.next:
        slow = slow.next
        fast = fast.next.next

    # 2. Развернуть вторую половину
    second = slow.next
    slow.next = None  # Разрываем список

    prev = None
    while second:
        next_temp = second.next
        second.next = prev
        prev = second
        second = next_temp
    second = prev

    # 3. Слить две половины
    first = head
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first, second = tmp1, tmp2
```

```go
func reorderList(head *ListNode) {
    if head == nil || head.Next == nil {
        return
    }

    // Find middle
    slow, fast := head, head
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    // Reverse second half
    second := slow.Next
    slow.Next = nil

    var prev *ListNode
    for second != nil {
        nextTemp := second.Next
        second.Next = prev
        prev = second
        second = nextTemp
    }
    second = prev

    // Merge
    first := head
    for second != nil {
        tmp1, tmp2 := first.Next, second.Next
        first.Next = second
        second.Next = tmp1
        first, second = tmp1, tmp2
    }
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 7: LRU Cache

### Паттерн
Hash Map + Doubly Linked List

### Условие
Реализовать LRU (Least Recently Used) кэш с операциями get и put за O(1).

### Интуиция
- Hash map для O(1) доступа по ключу
- Doubly linked list для O(1) удаления/перемещения
- Самый новый элемент — в начале, старый — в конце

### Решение

```python
class DLinkedNode:
    def __init__(self, key=0, val=0):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> node

        # Dummy head and tail
        self.head = DLinkedNode()
        self.tail = DLinkedNode()
        self.head.next = self.tail
        self.tail.prev = self.head

    def _add_to_head(self, node):
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove_node(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev

    def _move_to_head(self, node):
        self._remove_node(node)
        self._add_to_head(node)

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        node = self.cache[key]
        self._move_to_head(node)
        return node.val

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.val = value
            self._move_to_head(node)
        else:
            node = DLinkedNode(key, value)
            self.cache[key] = node
            self._add_to_head(node)

            if len(self.cache) > self.capacity:
                # Удаляем LRU (перед tail)
                lru = self.tail.prev
                self._remove_node(lru)
                del self.cache[lru.key]
```

```go
type LRUCache struct {
    capacity int
    cache    map[int]*DLinkedNode
    head     *DLinkedNode
    tail     *DLinkedNode
}

type DLinkedNode struct {
    key, val   int
    prev, next *DLinkedNode
}

func Constructor(capacity int) LRUCache {
    head := &DLinkedNode{}
    tail := &DLinkedNode{}
    head.next = tail
    tail.prev = head

    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*DLinkedNode),
        head:     head,
        tail:     tail,
    }
}

func (c *LRUCache) addToHead(node *DLinkedNode) {
    node.prev = c.head
    node.next = c.head.next
    c.head.next.prev = node
    c.head.next = node
}

func (c *LRUCache) removeNode(node *DLinkedNode) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (c *LRUCache) Get(key int) int {
    if node, ok := c.cache[key]; ok {
        c.removeNode(node)
        c.addToHead(node)
        return node.val
    }
    return -1
}

func (c *LRUCache) Put(key int, value int) {
    if node, ok := c.cache[key]; ok {
        node.val = value
        c.removeNode(node)
        c.addToHead(node)
    } else {
        node := &DLinkedNode{key: key, val: value}
        c.cache[key] = node
        c.addToHead(node)

        if len(c.cache) > c.capacity {
            lru := c.tail.prev
            c.removeNode(lru)
            delete(c.cache, lru.key)
        }
    }
}
```

### Сложность
- Time: O(1) для get и put
- Space: O(capacity)

---

## На интервью

### Ключевые техники

| Задача | Техника |
|--------|---------|
| Разворот | Три указателя: prev, curr, next |
| Цикл | Fast & Slow pointers |
| Середина | Fast & Slow pointers |
| N-й с конца | Two pointers с отступом |
| Слияние | Dummy node |

### Типичные ошибки

1. Забывают про edge cases: пустой список, один элемент
2. Теряют указатель на следующий узел при развороте
3. Не используют dummy node и усложняют код
4. Не проверяют fast.next перед fast.next.next

### Совет

Всегда рисуйте список на бумаге и отслеживайте указатели пошагово.

---

[← Назад к списку тем](README.md)
