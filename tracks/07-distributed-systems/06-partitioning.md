# 06. Partitioning

[← Назад к списку тем](README.md)

---

## Зачем партиционирование

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Why Partition?                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Scalability:                                                       │
│  - Данные не помещаются на один узел                                │
│  - Нагрузка слишком высока для одного узла                          │
│                                                                     │
│  Performance:                                                       │
│  - Параллельная обработка                                           │
│  - Локальность данных                                               │
│                                                                     │
│  Geography:                                                         │
│  - Данные ближе к пользователям                                     │
│  - Compliance (данные в определённом регионе)                       │
│                                                                     │
│                  ┌─────────────────────────────┐                    │
│                  │        Full Dataset         │                    │
│                  │       (too big/busy)        │                    │
│                  └─────────────┬───────────────┘                    │
│                                │                                    │
│              ┌─────────────────┼─────────────────┐                  │
│              │                 │                 │                  │
│              ▼                 ▼                 ▼                  │
│        ┌──────────┐      ┌──────────┐      ┌──────────┐             │
│        │Partition1│      │Partition2│      │Partition3│             │
│        │  A - F   │      │  G - M   │      │  N - Z   │             │
│        └──────────┘      └──────────┘      └──────────┘             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Стратегии партиционирования

### Range Partitioning

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Range Partitioning                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Key ranges mapped to partitions                                    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Partition 1  │  Partition 2  │  Partition 3  │  Partition 4  │  │
│  │   [A - F]     │   [G - M]     │   [N - S]     │   [T - Z]     │  │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ✅ Efficient range queries                                         │
│  ✅ Easy to understand                                               │
│  ❌ Hot spots (popular ranges)                                       │
│  ❌ Uneven distribution                                              │
│                                                                     │
│  Examples:                                                          │
│  - HBase: row key ranges                                            │
│  - Bigtable: tablet ranges                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type RangePartitioner struct {
    ranges []KeyRange  // sorted by start
}

type KeyRange struct {
    Start    string
    End      string
    Partition int
}

func (p *RangePartitioner) GetPartition(key string) int {
    // Binary search for the range containing key
    for _, r := range p.ranges {
        if key >= r.Start && key < r.End {
            return r.Partition
        }
    }
    return -1
}
```

### Hash Partitioning

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Hash Partitioning                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  partition = hash(key) mod N                                        │
│                                                                     │
│     Key: "user_123"                                                 │
│          │                                                          │
│          ▼                                                          │
│     hash("user_123") = 7823456                                      │
│          │                                                          │
│          ▼                                                          │
│     7823456 mod 4 = 0                                               │
│          │                                                          │
│          ▼                                                          │
│     Partition 0                                                     │
│                                                                     │
│  ✅ Even distribution                                                │
│  ✅ No hot spots                                                     │
│  ❌ Range queries require scatter-gather                            │
│  ❌ Adding/removing nodes = full reshuffle                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Consistent Hashing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Consistent Hashing                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Nodes and keys on a ring (0 to 2^32-1)                             │
│                                                                     │
│                    ┌───── Node A                                    │
│                   /│\                                               │
│                  / │ \                                              │
│                 /  │  \                                             │
│               ●    │    ●──── Node B                                │
│              /     │     \                                          │
│             /      │      \                                         │
│            │       │       │                                        │
│            │   Ring│       │                                        │
│            │       │       │                                        │
│             \      │      /                                         │
│              \     │     /                                          │
│               ●    │    ●──── Node C                                │
│                 \  │  /                                             │
│                  \ │ /                                              │
│                   \│/                                               │
│                    └───── Node D                                    │
│                                                                     │
│  Key → hash(key) → найти следующий node по часовой                  │
│                                                                     │
│  Adding node: only affects keys between new node and predecessor    │
│  Removing node: only affects keys that belonged to removed node     │
│                                                                     │
│  ~1/N keys moved when adding/removing node (vs all in hash mod N)   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Virtual Nodes

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Virtual Nodes                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Problem: uneven distribution with few physical nodes               │
│                                                                     │
│  Solution: each physical node = many virtual nodes                  │
│                                                                     │
│  Physical Node A → VN_A1, VN_A2, VN_A3, ...                         │
│  Physical Node B → VN_B1, VN_B2, VN_B3, ...                         │
│                                                                     │
│             VN_A1                                                   │
│               ●                                                     │
│              / \        VN_B1                                       │
│             /   \         ●                                         │
│            /     \       / \                                        │
│     VN_B2 ●───────●─────●───● VN_A2                                 │
│            \     /       \ /                                        │
│             \   /         ●                                         │
│              \ /        VN_A3                                       │
│               ●                                                     │
│             VN_B3                                                   │
│                                                                     │
│  Benefits:                                                          │
│  - More even distribution                                           │
│  - Proportional allocation (more VNs = more data)                   │
│  - Smooth rebalancing                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type ConsistentHash struct {
    ring       *treemap.Map  // sorted map: hash -> node
    vnodes     int           // virtual nodes per physical node
    hashFunc   func([]byte) uint32
}

func (ch *ConsistentHash) AddNode(node string) {
    for i := 0; i < ch.vnodes; i++ {
        vnode := fmt.Sprintf("%s#%d", node, i)
        hash := ch.hashFunc([]byte(vnode))
        ch.ring.Put(hash, node)
    }
}

func (ch *ConsistentHash) RemoveNode(node string) {
    for i := 0; i < ch.vnodes; i++ {
        vnode := fmt.Sprintf("%s#%d", node, i)
        hash := ch.hashFunc([]byte(vnode))
        ch.ring.Remove(hash)
    }
}

func (ch *ConsistentHash) GetNode(key string) string {
    hash := ch.hashFunc([]byte(key))

    // Find first node with hash >= key hash
    iter := ch.ring.Ceiling(hash)
    if iter == nil {
        // Wrap around to first node
        iter = ch.ring.Min()
    }

    return iter.Value().(string)
}
```

---

## Rebalancing

### Fixed Partitions

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Fixed Partitions                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Create many more partitions than nodes                             │
│  Assign partitions to nodes                                         │
│                                                                     │
│  Example: 1000 partitions, 4 nodes                                  │
│                                                                     │
│  Initial:                                                           │
│  Node A: P1-P250                                                    │
│  Node B: P251-P500                                                  │
│  Node C: P501-P750                                                  │
│  Node D: P751-P1000                                                 │
│                                                                     │
│  Add Node E:                                                        │
│  Move some partitions from each node to E                           │
│  Node A: P1-P200                                                    │
│  Node B: P251-P450                                                  │
│  Node C: P501-P700                                                  │
│  Node D: P751-P950                                                  │
│  Node E: P201-P250, P451-P500, P701-P750, P951-P1000               │
│                                                                     │
│  ✅ Minimal data movement                                            │
│  ✅ Predictable                                                      │
│  ❌ Fixed number upfront                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Dynamic Partitions

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Dynamic Partitions                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Split partition when too large                                     │
│  Merge partitions when too small                                    │
│                                                                     │
│  Initial:                                                           │
│  ┌─────────────────────────────┐                                    │
│  │       Partition 1           │  (size: 10GB)                      │
│  │         [A - Z]             │                                    │
│  └─────────────────────────────┘                                    │
│                                                                     │
│  After growth (split at 10GB):                                      │
│  ┌──────────────┐  ┌──────────────┐                                 │
│  │ Partition 1a │  │ Partition 1b │                                 │
│  │   [A - M]    │  │   [N - Z]    │                                 │
│  └──────────────┘  └──────────────┘                                 │
│                                                                     │
│  Used by: HBase, RethinkDB, MongoDB                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Secondary Indexes

### Local Index (Document-partitioned)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Local Index                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Each partition has its own index                                   │
│                                                                     │
│  Partition 1              Partition 2                               │
│  ┌─────────────────┐      ┌─────────────────┐                       │
│  │ Data: users 1-50│      │ Data: users 51+ │                       │
│  │ Index: by city  │      │ Index: by city  │                       │
│  │  NYC → [1,5,12] │      │  NYC → [55,78]  │                       │
│  │  LA → [3,8,15]  │      │  LA → [60,92]   │                       │
│  └─────────────────┘      └─────────────────┘                       │
│                                                                     │
│  Query "all users in NYC":                                          │
│  ✅ Writes: update local index only                                 │
│  ❌ Reads: scatter-gather (query all partitions)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Global Index (Term-partitioned)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Global Index                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Index partitioned by term                                          │
│                                                                     │
│  Index Partition 1        Index Partition 2                         │
│  (cities A-M)             (cities N-Z)                              │
│  ┌─────────────────┐      ┌─────────────────┐                       │
│  │ LA → [3,8,60,92]│      │ NYC → [1,5,55,78]│                      │
│  │ Miami → [...]   │      │ Seattle → [...] │                       │
│  └─────────────────┘      └─────────────────┘                       │
│                                                                     │
│  Query "all users in NYC":                                          │
│  ✅ Reads: single partition                                         │
│  ❌ Writes: may need to update multiple partitions                  │
│                                                                     │
│  Index updates often async → eventually consistent                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Routing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Request Routing                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Option 1: Random node + redirect                                   │
│  ┌────────┐                                                         │
│  │ Client │──▶ Any Node ──▶ Correct Node                            │
│  └────────┘    (redirect)                                           │
│                                                                     │
│  Option 2: Routing tier                                             │
│  ┌────────┐    ┌─────────────┐                                      │
│  │ Client │──▶│ Router Layer │──▶ Correct Node                      │
│  └────────┘    └─────────────┘                                      │
│                  (knows mapping)                                    │
│                                                                     │
│  Option 3: Client-aware                                             │
│  ┌────────┐                                                         │
│  │ Client │──────────────────▶ Correct Node                         │
│  └────────┘                                                         │
│  (client knows mapping)                                             │
│                                                                     │
│  Mapping storage:                                                   │
│  - ZooKeeper / etcd                                                 │
│  - Gossip protocol                                                  │
│  - Centralized coordinator                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Sharding](../06-databases/06-sharding.md) — шардирование баз данных и стратегии распределения данных
- [Ride Sharing System Design](../03-system-design/12-ride-sharing.md) — пример геопартиционирования в системе такси

---

## На интервью

### Типичные вопросы

1. **Range vs Hash partitioning?**
   - Range: range queries, hot spots
   - Hash: even distribution, scatter-gather

2. **Consistent hashing — как работает?**
   - Ring 0 to 2^32
   - Key → hash → next node clockwise
   - Add/remove → only affects neighbors

3. **Virtual nodes — зачем?**
   - Even distribution
   - Proportional allocation
   - Smooth rebalancing

4. **Local vs Global indexes?**
   - Local: fast writes, scatter reads
   - Global: fast reads, slow writes

5. **Как происходит rebalancing?**
   - Fixed partitions: reassign
   - Dynamic: split/merge

---

[← Назад к списку тем](README.md)
