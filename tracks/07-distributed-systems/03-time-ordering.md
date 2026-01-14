# 03. Time and Ordering

[← Назад к списку тем](README.md)

---

## Проблема времени

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Time in Distributed Systems                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Проблема: нет глобального времени                                  │
│                                                                     │
│  Node A: 10:00:00.000                                               │
│  Node B: 10:00:00.050  ← 50ms ahead                                 │
│  Node C: 09:59:59.980  ← 20ms behind                                │
│                                                                     │
│  Clock skew reasons:                                                │
│  - Hardware imperfections                                           │
│  - Temperature variations                                           │
│  - NTP synchronization delays                                       │
│                                                                     │
│  Consequences:                                                      │
│  - "Later" write might have earlier timestamp                       │
│  - Last-Write-Wins might lose data                                  │
│  - Cannot rely on wall-clock ordering                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Happens-Before Relation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Happens-Before (→)                                │
│                    (Leslie Lamport, 1978)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  a → b означает "a happened before b"                               │
│                                                                     │
│  Rules:                                                             │
│  ──────                                                             │
│  1. Внутри процесса: если a перед b, то a → b                       │
│  2. Отправка → получение сообщения                                  │
│  3. Транзитивность: если a → b и b → c, то a → c                    │
│                                                                     │
│  Process 1     Process 2     Process 3                              │
│     │              │              │                                 │
│     a ─────────────────────────▶ c                                  │
│     │              │              │                                 │
│     │              b              │                                 │
│     │              │              │                                 │
│     │              └──────────▶  d                                  │
│     │              │              │                                 │
│                                                                     │
│  a → c (message)                                                    │
│  b → d (message)                                                    │
│  a ∥ b (concurrent, no relation)                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Lamport Clocks

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Lamport Clocks                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Logical clock: просто счётчик                                      │
│                                                                     │
│  Rules:                                                             │
│  1. Перед событием: counter++                                       │
│  2. При отправке: включить counter в сообщение                      │
│  3. При получении: counter = max(local, received) + 1               │
│                                                                     │
│  Process 1     Process 2     Process 3                              │
│  [1]              │              │                                  │
│     │              │              │                                 │
│  [2]─────────────────────────▶[3]                                   │
│     │              │              │                                 │
│     │           [1]               │                                 │
│     │              │              │                                 │
│     │           [2]───────────▶[4]                                  │
│     │              │              │                                 │
│  [3]◀─────────────────────────[5]                                   │
│     │              │              │                                 │
│  [6]              [3]           [6]                                  │
│                                                                     │
│  Property:                                                          │
│  if a → b then L(a) < L(b)                                          │
│  BUT: L(a) < L(b) does NOT imply a → b                              │
│       (события могут быть concurrent)                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type LamportClock struct {
    counter int64
    mu      sync.Mutex
}

func (lc *LamportClock) Tick() int64 {
    lc.mu.Lock()
    defer lc.mu.Unlock()
    lc.counter++
    return lc.counter
}

func (lc *LamportClock) Update(received int64) int64 {
    lc.mu.Lock()
    defer lc.mu.Unlock()
    if received > lc.counter {
        lc.counter = received
    }
    lc.counter++
    return lc.counter
}

// Usage
func sendMessage(msg *Message) {
    msg.Timestamp = clock.Tick()
    network.Send(msg)
}

func receiveMessage(msg *Message) {
    clock.Update(msg.Timestamp)
    process(msg)
}
```

---

## Vector Clocks

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Vector Clocks                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Отслеживают causality точно                                        │
│  Каждый процесс хранит вектор [N1, N2, N3, ...]                     │
│                                                                     │
│  Rules:                                                             │
│  1. Своя компонента: counter++                                      │
│  2. При отправке: включить весь вектор                              │
│  3. При получении: merge (поэлементный max), затем своя++           │
│                                                                     │
│  Process 1     Process 2     Process 3                              │
│  [1,0,0]           │              │                                 │
│     │              │              │                                 │
│  [2,0,0]─────▶ [2,1,0]            │                                 │
│     │              │              │                                 │
│     │          [2,2,0]────▶ [2,2,1]                                 │
│     │              │              │                                 │
│  [3,0,0]           │         [2,2,2]                                │
│     │              │              │                                 │
│                                                                     │
│  Comparison:                                                        │
│  V1 < V2  iff  all V1[i] <= V2[i] AND exists i: V1[i] < V2[i]       │
│  V1 ∥ V2  iff  neither V1 < V2 nor V2 < V1                          │
│                                                                     │
│  [2,0,0] < [2,2,1]  → happens-before                                │
│  [3,0,0] ∥ [2,2,1]  → concurrent                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type VectorClock map[string]int64

func NewVectorClock(nodeID string) VectorClock {
    return VectorClock{nodeID: 0}
}

func (vc VectorClock) Increment(nodeID string) VectorClock {
    result := vc.Clone()
    result[nodeID]++
    return result
}

func (vc VectorClock) Merge(other VectorClock) VectorClock {
    result := vc.Clone()
    for k, v := range other {
        if v > result[k] {
            result[k] = v
        }
    }
    return result
}

func (vc VectorClock) Compare(other VectorClock) int {
    less, greater := false, false

    // Check all keys in vc
    for k, v := range vc {
        ov := other[k]
        if v < ov {
            less = true
        } else if v > ov {
            greater = true
        }
    }

    // Check keys in other that are not in vc
    for k, v := range other {
        if _, ok := vc[k]; !ok && v > 0 {
            less = true
        }
    }

    if less && !greater {
        return -1  // vc < other (happens-before)
    }
    if greater && !less {
        return 1   // vc > other
    }
    if !less && !greater {
        return 0   // equal
    }
    return 2       // concurrent
}
```

### Conflict Detection

```go
// Используется в Dynamo-style systems
type VersionedValue struct {
    Value   interface{}
    Version VectorClock
}

func write(key string, value interface{}, clientVersion VectorClock) error {
    currentVersions := store.Get(key)

    for _, v := range currentVersions {
        cmp := clientVersion.Compare(v.Version)
        if cmp == 2 {
            // Concurrent! Conflict detected
            // Option 1: Store both, let client resolve
            // Option 2: Use conflict resolution (LWW, merge)
            return ErrConflict{Current: v.Version, New: clientVersion}
        }
    }

    // No conflict, write new version
    newVersion := clientVersion.Increment(nodeID)
    store.Put(key, VersionedValue{Value: value, Version: newVersion})
    return nil
}
```

---

## Hybrid Logical Clocks (HLC)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Hybrid Logical Clocks                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Комбинация physical time + logical counter                         │
│  Лучшее из обоих миров                                              │
│                                                                     │
│  HLC = (physical_time, logical_counter)                             │
│                                                                     │
│  Rules:                                                             │
│  1. На событие:                                                     │
│     l' = max(l, physical_time)                                      │
│     if l' == l: c++ else c = 0                                      │
│                                                                     │
│  2. На получение (l', c'):                                          │
│     l'' = max(l, l', physical_time)                                 │
│     if l'' == l == l': c = max(c, c') + 1                           │
│     else if l'' == l: c++                                           │
│     else if l'' == l': c = c' + 1                                   │
│     else: c = 0                                                     │
│                                                                     │
│  Properties:                                                        │
│  - Близко к wall-clock (bounded drift)                              │
│  - Сохраняет happens-before ordering                                │
│  - O(1) space (не вектор)                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type HLC struct {
    mu   sync.Mutex
    l    int64  // logical (wall time estimate)
    c    int64  // counter
}

func (h *HLC) Now() (int64, int64) {
    h.mu.Lock()
    defer h.mu.Unlock()

    pt := time.Now().UnixNano()

    if pt > h.l {
        h.l = pt
        h.c = 0
    } else {
        h.c++
    }

    return h.l, h.c
}

func (h *HLC) Update(remoteL, remoteC int64) (int64, int64) {
    h.mu.Lock()
    defer h.mu.Unlock()

    pt := time.Now().UnixNano()

    // Find max of local l, remote l, and physical time
    maxL := max(h.l, max(remoteL, pt))

    if maxL == h.l && maxL == remoteL {
        h.c = max(h.c, remoteC) + 1
    } else if maxL == h.l {
        h.c++
    } else if maxL == remoteL {
        h.c = remoteC + 1
    } else {
        h.c = 0
    }

    h.l = maxL
    return h.l, h.c
}
```

---

## Сравнение

```
┌─────────────────┬────────────────┬────────────────┬─────────────────┐
│ Clock Type      │ Space          │ Causality      │ Wall-clock      │
├─────────────────┼────────────────┼────────────────┼─────────────────┤
│ Physical        │ O(1)           │ ❌ (clock skew)│ ✅              │
│ Lamport         │ O(1)           │ Partial        │ ❌              │
│ Vector          │ O(N) nodes     │ ✅ Complete    │ ❌              │
│ HLC             │ O(1)           │ ✅ Complete    │ ≈ (bounded)     │
└─────────────────┴────────────────┴────────────────┴─────────────────┘

N = number of nodes in the system
```

---

## Реальные применения

```
Lamport Clocks:
- Totally ordered logs (Raft)
- Transaction ordering

Vector Clocks:
- Amazon DynamoDB (конфликт detection)
- Riak (conflict resolution)

HLC:
- CockroachDB (transaction timestamps)
- Spanner (TrueTime uses GPS/atomic clocks)
- YugabyteDB
```

---

## См. также

- [Consensus](./04-consensus.md) — алгоритмы консенсуса Paxos и Raft для согласования между узлами
- [Distributed ID Generation](../03-system-design/13-distributed-id.md) — генерация уникальных идентификаторов в распределённых системах

---

## На интервью

### Типичные вопросы

1. **Почему нельзя использовать wall clock?**
   - Clock skew между узлами
   - NTP не точен (ms-level)
   - "Later" event может иметь earlier timestamp

2. **Lamport vs Vector clocks?**
   - Lamport: O(1) space, partial causality
   - Vector: O(N) space, complete causality
   - Vector нужен для conflict detection

3. **Что такое happens-before?**
   - Partial order на событиях
   - Определяется: внутри процесса, сообщениями, транзитивно
   - Concurrent события: нет отношения

4. **Hybrid Logical Clocks — зачем?**
   - Bounded drift от wall-clock
   - Полная causality
   - O(1) space

---

[← Назад к списку тем](README.md)
