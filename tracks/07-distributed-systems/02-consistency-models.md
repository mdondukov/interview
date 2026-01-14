# 02. Consistency Models

[← Назад к списку тем](README.md)

---

## Спектр консистентности

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Consistency Spectrum                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Strong ◄──────────────────────────────────────────────────► Weak   │
│                                                                     │
│  Linearizable                                                       │
│      │                                                              │
│      ▼                                                              │
│  Sequential Consistency                                             │
│      │                                                              │
│      ▼                                                              │
│  Causal Consistency                                                 │
│      │                                                              │
│      ▼                                                              │
│  Read-your-writes                                                   │
│      │                                                              │
│      ▼                                                              │
│  Monotonic Reads                                                    │
│      │                                                              │
│      ▼                                                              │
│  Eventual Consistency                                               │
│                                                                     │
│  ──────────────────────────────────────────────────────────────     │
│  Strong: проще для разработчика, дороже в производительности        │
│  Weak: сложнее для разработчика, дешевле в производительности       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Linearizability (Strong Consistency)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Linearizability                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Операции выглядят так, как будто выполняются атомарно              │
│  в некоторой последовательности, соответствующей real-time.         │
│                                                                     │
│  Time →                                                             │
│                                                                     │
│  Client A:  ────[Write x=1]────────────────────────────             │
│                      │                                              │
│                      ▼ (linearization point)                        │
│  Client B:  ──────────────[Read x]─────────────────────             │
│                              │                                      │
│                              └─▶ Must return 1                      │
│                                                                     │
│  Client C:  ───────────────────────[Read x]────────────             │
│                                        │                            │
│                                        └─▶ Must return 1            │
│                                                                     │
│  Если Write завершился до начала Read, Read видит результат.        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Примеры

```go
// Linearizable: single-leader replication with sync writes
// Все операции идут через leader

// ✅ Linearizable операции:
// - Compare-and-swap (CAS)
// - Test-and-set
// - Fetch-and-add

// Пример: distributed lock
func acquireLock(key string) bool {
    // CAS: set key = myID only if key doesn't exist
    success, _ := redis.SetNX(ctx, key, myID, ttl)
    return success
}
```

---

## Sequential Consistency

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Sequential Consistency                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Все процессы видят операции в одном порядке.                       │
│  НО порядок может отличаться от real-time порядка.                  │
│                                                                     │
│  Real-time order:                                                   │
│  Client A: Write(x=1) ────────────────────────                      │
│  Client B: ─────────────────── Write(x=2) ─────                     │
│                                                                     │
│  Sequential order (valid):                                          │
│  All clients see: Write(x=2), Write(x=1)                            │
│  Result: x = 1                                                      │
│                                                                     │
│  ИЛИ                                                                │
│                                                                     │
│  Sequential order (also valid):                                     │
│  All clients see: Write(x=1), Write(x=2)                            │
│  Result: x = 2                                                      │
│                                                                     │
│  Главное: все видят ОДИНАКОВЫЙ порядок                              │
│  (не обязательно real-time)                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Causal Consistency

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Causal Consistency                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Причинно связанные операции видны в правильном порядке.            │
│  Независимые операции могут быть в любом порядке.                   │
│                                                                     │
│  Causal relationship:                                               │
│  A → B означает "A happened before B"                               │
│                                                                     │
│  Example:                                                           │
│  ─────────                                                          │
│  Alice: posts "Hello"                                               │
│  Bob: sees Alice's post, replies "Hi!"                              │
│                                                                     │
│  Reply causally depends on original post.                           │
│  Everyone must see: Alice's post → Bob's reply                      │
│                                                                     │
│  Carol: posts "Good morning" (independent)                          │
│  Carol's post can appear before or after Alice+Bob                  │
│                                                                     │
│  ✅ Valid:   Alice → Bob → Carol                                    │
│  ✅ Valid:   Carol → Alice → Bob                                    │
│  ✅ Valid:   Alice → Carol → Bob                                    │
│  ❌ Invalid: Bob → Alice (causal order violated)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation

```go
// Vector clocks track causality
type VectorClock map[string]int64

func (vc VectorClock) Increment(nodeID string) {
    vc[nodeID]++
}

func (vc VectorClock) Merge(other VectorClock) {
    for k, v := range other {
        if v > vc[k] {
            vc[k] = v
        }
    }
}

func (vc VectorClock) HappensBefore(other VectorClock) bool {
    // vc < other if all components <= and at least one <
    less := false
    for k, v := range vc {
        if v > other[k] {
            return false
        }
        if v < other[k] {
            less = true
        }
    }
    return less
}
```

---

## Eventual Consistency

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Eventual Consistency                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Если нет новых обновлений, все реплики                             │
│  в конечном итоге вернут одинаковое значение.                       │
│                                                                     │
│  Timeline:                                                          │
│                                                                     │
│  Write(x=1)     Propagating...           All replicas have x=1      │
│      │               │                            │                 │
│      ▼               ▼                            ▼                 │
│  ────●───────────────────────────────────────────●──────────────    │
│      │               │                            │                 │
│      │               │                            │                 │
│  Node A: x=1    Node A: x=1                  Node A: x=1            │
│  Node B: x=0    Node B: x=0    ← stale      Node B: x=1             │
│  Node C: x=0    Node C: x=1                  Node C: x=1            │
│                                                                     │
│  "Eventually" может быть:                                           │
│  - Миллисекунды (обычно)                                            │
│  - Секунды (под нагрузкой)                                          │
│  - Минуты (при проблемах)                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Strong Eventual Consistency

```
┌─────────────────────────────────────────────────────────────────────┐
│              Strong Eventual Consistency                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Eventually Consistent +                                            │
│  Детерминированный conflict resolution                              │
│                                                                     │
│  CRDTs (Conflict-free Replicated Data Types):                       │
│  - G-Counter: grow-only counter                                     │
│  - PN-Counter: positive-negative counter                            │
│  - G-Set: grow-only set                                             │
│  - LWW-Register: last-writer-wins                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// G-Counter: каждый узел инкрементит свой счётчик
type GCounter map[string]int64

func (gc GCounter) Increment(nodeID string) {
    gc[nodeID]++
}

func (gc GCounter) Value() int64 {
    var sum int64
    for _, v := range gc {
        sum += v
    }
    return sum
}

func (gc GCounter) Merge(other GCounter) {
    for k, v := range other {
        if v > gc[k] {
            gc[k] = v
        }
    }
}

// Нет конфликтов! Merge всегда корректен.
```

---

## Session Guarantees

### Read Your Writes

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Read Your Writes                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Клиент всегда видит свои собственные записи.                       │
│                                                                     │
│  Client:                                                            │
│    Write(x=1) ───────────▶ Success                                  │
│    Read(x)    ───────────▶ Must return 1                            │
│                                                                     │
│  Implementation:                                                    │
│  ─────────────                                                      │
│  - Sticky sessions (route to same replica)                          │
│  - Write to leader, read from leader                                │
│  - Version tokens (wait for version)                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Monotonic Reads

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Monotonic Reads                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Время никогда не идёт назад.                                       │
│  Если видели x=2, никогда не увидим x=1.                            │
│                                                                     │
│  ❌ Without monotonic reads:                                        │
│  Read(x) → 2                                                        │
│  Read(x) → 1  ← Different replica, older value!                     │
│                                                                     │
│  ✅ With monotonic reads:                                           │
│  Read(x) → 2                                                        │
│  Read(x) → 2 or higher                                              │
│                                                                     │
│  Implementation:                                                    │
│  - Track last-read version                                          │
│  - Read from replica with >= version                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Monotonic Writes

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Monotonic Writes                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Записи клиента применяются в порядке их выполнения.                │
│                                                                     │
│  Client:                                                            │
│    Write(x=1) at T1                                                 │
│    Write(x=2) at T2 (T2 > T1)                                       │
│                                                                     │
│  All replicas must apply: x=1 before x=2                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Сравнение

```
┌────────────────────┬────────────────┬────────────────┬──────────────┐
│ Model              │ Ordering       │ Availability   │ Use Case     │
├────────────────────┼────────────────┼────────────────┼──────────────┤
│ Linearizable       │ Real-time      │ Low (blocks)   │ Locks, CAS   │
│ Sequential         │ Total order    │ Medium         │ Replicated   │
│                    │ (not real-time)│                │ state        │
│ Causal             │ Partial order  │ High           │ Messaging,   │
│                    │ (causality)    │                │ social       │
│ Eventual           │ None guaranteed│ Highest        │ Caching,     │
│                    │                │                │ metrics      │
└────────────────────┴────────────────┴────────────────┴──────────────┘
```

---

## На интервью

### Типичные вопросы

1. **Linearizability vs Sequential Consistency?**
   - Linearizable: real-time ordering
   - Sequential: total order, but not real-time

2. **Что такое Causal Consistency?**
   - Причинно связанные операции в порядке
   - Независимые могут быть в любом порядке

3. **Eventual Consistency — что гарантирует?**
   - Все реплики сойдутся
   - Но не говорит когда
   - Не гарантирует порядок

4. **Session guarantees?**
   - Read-your-writes
   - Monotonic reads
   - Monotonic writes

5. **Когда нужна Strong Consistency?**
   - Финансы, inventory
   - Distributed locks
   - Leader election

---

[← Назад к списку тем](README.md)
