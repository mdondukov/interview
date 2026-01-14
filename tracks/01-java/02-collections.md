# 02. Collections Framework

[← Назад к списку тем](README.md)

---

## Collections Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Collections Hierarchy                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Iterable                                                           │
│      │                                                              │
│  Collection                                                         │
│      ├── List (ordered, duplicates allowed)                         │
│      │    ├── ArrayList                                             │
│      │    ├── LinkedList                                            │
│      │    ├── Vector (synchronized)                                 │
│      │    └── Stack                                                 │
│      │                                                              │
│      ├── Set (no duplicates)                                        │
│      │    ├── HashSet                                               │
│      │    ├── LinkedHashSet (ordered)                               │
│      │    ├── TreeSet (sorted)                                      │
│      │    └── EnumSet                                               │
│      │                                                              │
│      └── Queue                                                      │
│           ├── PriorityQueue                                         │
│           ├── ArrayDeque                                            │
│           └── LinkedList                                            │
│                                                                     │
│  Map (not Collection!)                                              │
│      ├── HashMap                                                    │
│      ├── LinkedHashMap (ordered)                                    │
│      ├── TreeMap (sorted)                                           │
│      ├── Hashtable (synchronized)                                   │
│      ├── WeakHashMap                                                │
│      └── EnumMap                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## List Implementations

### ArrayList

```java
// Dynamic array, fast random access
List<String> list = new ArrayList<>();

list.add("a");           // O(1) amortized
list.add(0, "b");        // O(n) - shift elements
list.get(0);             // O(1)
list.set(0, "c");        // O(1)
list.remove(0);          // O(n) - shift elements
list.remove("a");        // O(n) - search + shift
list.contains("a");      // O(n)
list.size();             // O(1)

// Initial capacity to avoid resizing
List<String> list = new ArrayList<>(1000);

// Internal: Object[] elementData
// Grows by 50% when full: newCapacity = oldCapacity + (oldCapacity >> 1)
```

### LinkedList

```java
// Doubly-linked list, fast insert/delete at ends
List<String> list = new LinkedList<>();
Deque<String> deque = new LinkedList<>();  // Also implements Deque

list.add("a");           // O(1) - add at end
list.add(0, "b");        // O(1) for first/last, O(n) for middle
list.get(0);             // O(n) - traverse from head/tail
list.set(0, "c");        // O(n)
list.remove(0);          // O(1) for first/last
list.contains("a");      // O(n)

// Deque operations
deque.addFirst("x");     // O(1)
deque.addLast("y");      // O(1)
deque.removeFirst();     // O(1)
deque.peekFirst();       // O(1)
```

### ArrayList vs LinkedList

```
┌─────────────────────────────────────────────────────────────────────┐
│              ArrayList vs LinkedList                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Operation          ArrayList        LinkedList                     │
│  ─────────          ─────────        ──────────                     │
│  get(index)         O(1)             O(n)                           │
│  add(end)           O(1)*            O(1)                           │
│  add(index)         O(n)             O(n)**                         │
│  remove(index)      O(n)             O(n)**                         │
│  contains           O(n)             O(n)                           │
│  Memory             Less             More (node overhead)           │
│  Cache              Good             Poor (scattered)               │
│                                                                     │
│  * amortized, can be O(n) when resizing                             │
│  ** O(1) if already at position                                     │
│                                                                     │
│  Use ArrayList in most cases (better cache locality)                │
│  Use LinkedList for queue/deque operations                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Set Implementations

### HashSet

```java
// Based on HashMap, no order guarantee
Set<String> set = new HashSet<>();

set.add("a");            // O(1)
set.remove("a");         // O(1)
set.contains("a");       // O(1)
set.size();              // O(1)

// Initial capacity and load factor
Set<String> set = new HashSet<>(16, 0.75f);

// Collision handling: linked list → tree (when > 8 elements in bucket)
```

### LinkedHashSet

```java
// Maintains insertion order
Set<String> set = new LinkedHashSet<>();

set.add("b");
set.add("a");
set.add("c");

for (String s : set) {
    System.out.print(s + " ");  // b a c (insertion order)
}

// Uses doubly-linked list to maintain order
// Slightly more memory than HashSet
```

### TreeSet

```java
// Sorted set, based on TreeMap (Red-Black tree)
Set<String> set = new TreeSet<>();

set.add("b");
set.add("a");
set.add("c");

for (String s : set) {
    System.out.print(s + " ");  // a b c (sorted)
}

// NavigableSet operations
TreeSet<Integer> nums = new TreeSet<>(List.of(1, 5, 10, 15, 20));
nums.first();             // 1
nums.last();              // 20
nums.floor(12);           // 10 (≤ 12)
nums.ceiling(12);         // 15 (≥ 12)
nums.lower(10);           // 5  (< 10)
nums.higher(10);          // 15 (> 10)
nums.headSet(10);         // [1, 5] (< 10)
nums.tailSet(10);         // [10, 15, 20] (≥ 10)
nums.subSet(5, 15);       // [5, 10] (≥ 5, < 15)

// All operations O(log n)
```

### Set Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Set Comparison                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Set Type         Order              add/remove/contains            │
│  ────────         ─────              ──────────────────             │
│  HashSet          No order           O(1)                           │
│  LinkedHashSet    Insertion order    O(1)                           │
│  TreeSet          Sorted             O(log n)                       │
│  EnumSet          Enum declaration   O(1) - bit vector              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Map Implementations

### HashMap

```java
// Hash table implementation
Map<String, Integer> map = new HashMap<>();

map.put("a", 1);         // O(1)
map.get("a");            // O(1)
map.remove("a");         // O(1)
map.containsKey("a");    // O(1)
map.containsValue(1);    // O(n)
map.size();              // O(1)

// Useful methods
map.getOrDefault("x", 0);
map.putIfAbsent("b", 2);
map.computeIfAbsent("c", k -> k.length());
map.computeIfPresent("a", (k, v) -> v + 1);
map.merge("a", 1, Integer::sum);

// Iteration
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
}
map.forEach((k, v) -> System.out.println(k + "=" + v));
```

### HashMap Internals

```
┌─────────────────────────────────────────────────────────────────────┐
│                  HashMap Internals                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Structure: array of buckets (Node<K,V>[])                          │
│                                                                     │
│  index = hash(key) & (capacity - 1)                                 │
│                                                                     │
│  Collision handling:                                                │
│  - Linked list (≤8 elements in bucket)                              │
│  - Red-Black tree (>8 elements) - Java 8+                           │
│                                                                     │
│  Buckets:                                                           │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┐                                  │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                  │
│  └─┬─┴───┴─┬─┴───┴───┴───┴─┬─┴───┘                                  │
│    │       │               │                                        │
│    ▼       ▼               ▼                                        │
│  [A→B]   [C→D→E]         [F→G]        (linked lists)                │
│                                                                     │
│  Load factor: 0.75 (default)                                        │
│  Resize: when size > capacity × loadFactor                          │
│  New capacity = 2 × old capacity                                    │
│                                                                     │
│  Key requirements:                                                  │
│  - Implement hashCode() and equals()                                │
│  - Immutable keys recommended                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### LinkedHashMap

```java
// Maintains insertion order or access order
Map<String, Integer> map = new LinkedHashMap<>();
map.put("c", 3);
map.put("a", 1);
map.put("b", 2);

// Insertion order
for (String key : map.keySet()) {
    System.out.print(key + " ");  // c a b
}

// Access-order mode (for LRU cache)
Map<String, Integer> lru = new LinkedHashMap<>(16, 0.75f, true);
lru.put("a", 1);
lru.put("b", 2);
lru.get("a");  // Move "a" to end
// Order now: b, a

// LRU Cache implementation
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true);  // accessOrder = true
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

### TreeMap

```java
// Sorted map, Red-Black tree
Map<String, Integer> map = new TreeMap<>();
map.put("b", 2);
map.put("a", 1);
map.put("c", 3);

// Iteration in sorted order
for (String key : map.keySet()) {
    System.out.print(key + " ");  // a b c
}

// NavigableMap operations
TreeMap<Integer, String> nums = new TreeMap<>();
nums.put(1, "one");
nums.put(5, "five");
nums.put(10, "ten");

nums.firstKey();          // 1
nums.lastKey();           // 10
nums.floorKey(7);         // 5 (≤ 7)
nums.ceilingKey(7);       // 10 (≥ 7)
nums.headMap(5);          // {1=one}
nums.tailMap(5);          // {5=five, 10=ten}
nums.subMap(1, 10);       // {1=one, 5=five}

// Custom comparator
Map<String, Integer> map = new TreeMap<>(Comparator.reverseOrder());
```

---

## Queue and Deque

### Queue Implementations

```java
// FIFO queue
Queue<String> queue = new LinkedList<>();
// or
Queue<String> queue = new ArrayDeque<>();  // Preferred!

queue.offer("a");        // Add to tail, returns false if full
queue.add("b");          // Add to tail, throws if full
queue.poll();            // Remove from head, returns null if empty
queue.remove();          // Remove from head, throws if empty
queue.peek();            // View head, returns null if empty
queue.element();         // View head, throws if empty

// PriorityQueue - heap-based, elements ordered by priority
Queue<Integer> pq = new PriorityQueue<>();  // Min-heap
pq.offer(3);
pq.offer(1);
pq.offer(2);
pq.poll();  // 1 (smallest)
pq.poll();  // 2
pq.poll();  // 3

// Max-heap
Queue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder());
```

### Deque (Double-ended Queue)

```java
Deque<String> deque = new ArrayDeque<>();

// Stack operations (LIFO)
deque.push("a");         // addFirst
deque.push("b");
deque.pop();             // removeFirst - "b"
deque.peek();            // peekFirst - "a"

// Queue operations (FIFO)
deque.offer("x");        // addLast
deque.poll();            // removeFirst

// Both ends
deque.addFirst("front");
deque.addLast("back");
deque.peekFirst();
deque.peekLast();
deque.removeFirst();
deque.removeLast();
```

---

## Concurrent Collections

```java
// Thread-safe alternatives

// ConcurrentHashMap - segmented locking (Java 8: CAS + sync)
Map<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);                          // Thread-safe
map.computeIfAbsent("b", k -> 2);         // Atomic
map.merge("a", 1, Integer::sum);          // Atomic

// CopyOnWriteArrayList - copy on write, good for read-heavy
List<String> list = new CopyOnWriteArrayList<>();
// Iterators see snapshot, no ConcurrentModificationException

// CopyOnWriteArraySet
Set<String> set = new CopyOnWriteArraySet<>();

// BlockingQueue - for producer-consumer
BlockingQueue<String> queue = new ArrayBlockingQueue<>(100);
queue.put("item");       // Blocks if full
queue.take();            // Blocks if empty

// ConcurrentLinkedQueue - non-blocking
Queue<String> queue = new ConcurrentLinkedQueue<>();

// ConcurrentSkipListMap/Set - sorted, concurrent
ConcurrentNavigableMap<String, Integer> map = new ConcurrentSkipListMap<>();
```

---

## equals() and hashCode()

```java
public class Person {
    private String name;
    private int age;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}

/*
Contract:
1. If a.equals(b), then a.hashCode() == b.hashCode()
2. If hashCode differs, objects must be different
3. Same hashCode doesn't mean equals (collisions allowed)
4. equals must be:
   - Reflexive: a.equals(a)
   - Symmetric: a.equals(b) == b.equals(a)
   - Transitive: if a.equals(b) && b.equals(c), then a.equals(c)
   - Consistent: same result for same objects
   - a.equals(null) = false

IMPORTANT: If you override equals, you MUST override hashCode
*/
```

---

## Comparable vs Comparator

```java
// Comparable - natural ordering, implemented by the class
public class Person implements Comparable<Person> {
    private String name;
    private int age;

    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);
    }
}

List<Person> people = new ArrayList<>();
Collections.sort(people);  // Uses compareTo

// Comparator - external comparison, multiple orderings
Comparator<Person> byName = Comparator.comparing(Person::getName);
Comparator<Person> byAge = Comparator.comparingInt(Person::getAge);
Comparator<Person> byNameThenAge = byName.thenComparing(byAge);
Comparator<Person> byAgeDesc = byAge.reversed();

Collections.sort(people, byName);
people.sort(byAge);

// Null handling
Comparator<Person> nullsFirst = Comparator.nullsFirst(byName);
Comparator<Person> nullsLast = Comparator.nullsLast(byName);
```

---

## На интервью

### Типичные вопросы

1. **ArrayList vs LinkedList?**
   - ArrayList: O(1) random access, O(n) insert/delete
   - LinkedList: O(n) random access, O(1) at ends
   - ArrayList preferred (cache locality)

2. **HashMap vs TreeMap?**
   - HashMap: O(1) operations, no order
   - TreeMap: O(log n) operations, sorted keys

3. **How does HashMap work?**
   - Array of buckets, hash → index
   - Collisions: linked list → tree (Java 8+)
   - Load factor 0.75, resize at threshold

4. **equals() and hashCode() contract?**
   - Equal objects must have same hashCode
   - Override both together
   - Used by HashMap, HashSet

5. **What is ConcurrentModificationException?**
   - Thrown when collection modified during iteration
   - Use Iterator.remove() or concurrent collections

6. **Fail-fast vs Fail-safe?**
   - Fail-fast: throws CME (ArrayList, HashMap)
   - Fail-safe: works on copy (CopyOnWriteArrayList, ConcurrentHashMap)

---

[← Назад к списку тем](README.md)
