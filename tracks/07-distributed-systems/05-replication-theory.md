# 05. Replication Theory

[← Назад к списку тем](README.md)

---

## State Machine Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                  State Machine Replication (SMR)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Идея: если все реплики выполняют одни и те же команды              │
│        в одном порядке, они будут в одинаковом состоянии.           │
│                                                                     │
│  Requirements:                                                      │
│  - Deterministic state machine                                      │
│  - Total order of commands                                          │
│  - All replicas start from same state                               │
│                                                                     │
│      Clients                                                        │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   Consensus Layer                            │    │
│  │                (Raft, Paxos, ZAB)                            │    │
│  │         Ensures total order of commands                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │              │              │                             │
│         ▼              ▼              ▼                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                        │
│  │ Replica 1 │  │ Replica 2 │  │ Replica 3 │                        │
│  │   State   │  │   State   │  │   State   │                        │
│  │  Machine  │  │  Machine  │  │  Machine  │                        │
│  └───────────┘  └───────────┘  └───────────┘                        │
│       ║              ║              ║                               │
│   Same state     Same state     Same state                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation

```go
// State Machine interface
type StateMachine interface {
    Apply(command []byte) (result []byte, err error)
    Snapshot() ([]byte, error)
    Restore(snapshot []byte) error
}

// Key-Value Store as State Machine
type KVStore struct {
    data map[string]string
    mu   sync.RWMutex
}

func (kv *KVStore) Apply(command []byte) ([]byte, error) {
    var cmd Command
    json.Unmarshal(command, &cmd)

    kv.mu.Lock()
    defer kv.mu.Unlock()

    switch cmd.Op {
    case "set":
        kv.data[cmd.Key] = cmd.Value
        return []byte("OK"), nil
    case "get":
        return []byte(kv.data[cmd.Key]), nil
    case "delete":
        delete(kv.data, cmd.Key)
        return []byte("OK"), nil
    }
    return nil, errors.New("unknown command")
}

// Replicated KV Store
type ReplicatedKVStore struct {
    raft *raft.Raft
    sm   *KVStore
}

func (r *ReplicatedKVStore) Set(key, value string) error {
    cmd, _ := json.Marshal(Command{Op: "set", Key: key, Value: value})

    // Replicate through consensus
    future := r.raft.Apply(cmd, 5*time.Second)
    if err := future.Error(); err != nil {
        return err
    }

    return nil
}
```

---

## Primary-Backup Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Primary-Backup Replication                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Simpler than SMR: primary executes, sends state to backups         │
│                                                                     │
│      Client                                                         │
│         │                                                           │
│         │ Request                                                   │
│         ▼                                                           │
│  ┌───────────┐                                                      │
│  │  Primary  │ ←── All writes go here                               │
│  │           │                                                      │
│  └─────┬─────┘                                                      │
│        │                                                            │
│        │ State updates (sync or async)                              │
│        │                                                            │
│        ├───────────────────┐                                        │
│        ▼                   ▼                                        │
│  ┌───────────┐      ┌───────────┐                                   │
│  │  Backup 1 │      │  Backup 2 │                                   │
│  └───────────┘      └───────────┘                                   │
│                                                                     │
│  Differences from SMR:                                              │
│  - Primary executes, sends result (not command)                     │
│  - Simpler: no need for determinism                                 │
│  - Can handle non-deterministic operations                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### SMR vs Primary-Backup

```
┌─────────────────────────┬────────────────────────────────────────────┐
│     State Machine       │           Primary-Backup                   │
├─────────────────────────┼────────────────────────────────────────────┤
│ Replicates commands     │ Replicates state/results                   │
│ All execute commands    │ Only primary executes                      │
│ Must be deterministic   │ Can be non-deterministic                   │
│ Less bandwidth (cmds)   │ More bandwidth (state)                     │
│ Complex failure handling│ Simpler failure handling                   │
└─────────────────────────┴────────────────────────────────────────────┘
```

---

## Chain Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Chain Replication                               │
│                  (van Renesse, Schneider, 2004)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│        Writes                                                       │
│           │                                                         │
│           ▼                                                         │
│     ┌─────────┐     ┌─────────┐     ┌─────────┐                     │
│     │  Head   │────▶│ Middle  │────▶│  Tail   │                     │
│     └─────────┘     └─────────┘     └─────────┘                     │
│                                           │                         │
│                                           ▼                         │
│                                    Reply to client                  │
│                                                                     │
│        Reads                                                        │
│           │                                                         │
│           └──────────────────────────────▶│                         │
│                                   ┌───────▼───────┐                 │
│                                   │     Tail      │                 │
│                                   └───────────────┘                 │
│                                                                     │
│  Properties:                                                        │
│  - Writes: Head → ... → Tail (sequential)                           │
│  - Reads: only from Tail (strongly consistent)                      │
│  - Simple: no voting, just propagation                              │
│  - High write throughput (pipelining)                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### CRAQ (Chain Replication with Apportioned Queries)

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CRAQ                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Extension: allow reads from any node                               │
│                                                                     │
│  Each node stores:                                                  │
│  - Clean version: committed                                         │
│  - Dirty version: pending (not yet at tail)                         │
│                                                                     │
│  Read logic:                                                        │
│  - If only clean version: return it                                 │
│  - If dirty version: ask tail for latest committed                  │
│                                                                     │
│  Benefits:                                                          │
│  - Read scaling: any node can serve reads                           │
│  - Still strongly consistent                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quorum Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Quorum Replication                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  N = total nodes                                                    │
│  W = write quorum (nodes that must ack write)                       │
│  R = read quorum (nodes to read from)                               │
│                                                                     │
│  Condition: W + R > N                                               │
│  (ensures read sees at least one node with latest write)            │
│                                                                     │
│  Example: N=5, W=3, R=3                                             │
│                                                                     │
│  Write to 3:          Read from 3:                                  │
│  ┌───┐ ┌───┐ ┌───┐    ┌───┐ ┌───┐ ┌───┐                             │
│  │ W │ │ W │ │ W │    │ R │ │ R │ │ R │                             │
│  └───┘ └───┘ └───┘    └───┘ └───┘ └───┘                             │
│  ┌───┐ ┌───┐          ┌───┐ ┌───┐                                   │
│  │   │ │   │          │   │ │   │                                   │
│  └───┘ └───┘          └───┘ └───┘                                   │
│                                                                     │
│  At least one node is in both sets → sees latest                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Quorum Configurations

```
Strong Consistency: W + R > N

Read-heavy: W=N, R=1
- Slow writes (all nodes)
- Fast reads (any node)

Write-heavy: W=1, R=N
- Fast writes (one node)
- Slow reads (all nodes)

Balanced: W=R=⌊N/2⌋+1
- Example: N=3, W=2, R=2
- Trade-off between read/write

Sloppy Quorum:
- Accept writes even if some nodes down
- Write to available nodes
- Hinted handoff when original node recovers
```

---

## Conflict Resolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Conflict Resolution                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  When concurrent writes to same key:                                │
│                                                                     │
│  Last-Write-Wins (LWW):                                             │
│  - Use timestamps                                                   │
│  - Higher timestamp wins                                            │
│  - Simple but can lose data                                         │
│                                                                     │
│  Vector Clocks:                                                     │
│  - Track causality                                                  │
│  - Detect concurrent writes                                         │
│  - Return all versions, client resolves                             │
│                                                                     │
│  CRDTs:                                                             │
│  - Mathematically guaranteed merge                                  │
│  - No conflicts by design                                           │
│  - Limited data types                                               │
│                                                                     │
│  Application-specific:                                              │
│  - Custom merge logic                                               │
│  - Domain knowledge                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// LWW Register
type LWWRegister struct {
    Value     interface{}
    Timestamp int64
}

func (r *LWWRegister) Set(value interface{}, ts int64) {
    if ts > r.Timestamp {
        r.Value = value
        r.Timestamp = ts
    }
}

func (r *LWWRegister) Merge(other LWWRegister) {
    if other.Timestamp > r.Timestamp {
        r.Value = other.Value
        r.Timestamp = other.Timestamp
    }
}

// Multi-Value Register (detect conflicts)
type MVRegister struct {
    Values []VersionedValue
}

func (r *MVRegister) Set(value interface{}, clock VectorClock) {
    // Remove values that happen-before new value
    var kept []VersionedValue
    for _, v := range r.Values {
        if !v.Clock.HappensBefore(clock) {
            kept = append(kept, v)
        }
    }
    r.Values = append(kept, VersionedValue{Value: value, Clock: clock})
}

func (r *MVRegister) Get() []interface{} {
    // Return all concurrent values
    var values []interface{}
    for _, v := range r.Values {
        values = append(values, v.Value)
    }
    return values
}
```

---

## Comparison

```
┌─────────────────────┬──────────────┬──────────────┬─────────────────┐
│ Approach            │ Consistency  │ Availability │ Complexity      │
├─────────────────────┼──────────────┼──────────────┼─────────────────┤
│ SMR (Raft/Paxos)    │ Strong       │ Medium       │ High            │
│ Primary-Backup      │ Strong       │ Medium       │ Medium          │
│ Chain Replication   │ Strong       │ Medium       │ Low             │
│ Quorum              │ Configurable │ High         │ Medium          │
│ Leaderless (Dynamo) │ Eventual     │ Very High    │ Medium          │
└─────────────────────┴──────────────┴──────────────┴─────────────────┘
```

---

## См. также

- [Replication](../06-databases/05-replication.md) — практические паттерны репликации в базах данных
- [Consensus](./04-consensus.md) — алгоритмы консенсуса для координации реплик

---

## На интервью

### Типичные вопросы

1. **State Machine Replication — как работает?**
   - Детерминированные state machines
   - Consensus для total order
   - Все выполняют одни команды

2. **Chain replication — плюсы/минусы?**
   - Простота, высокий throughput
   - Latency пропорционален длине цепи
   - Failure handling сложнее

3. **Quorum — как выбрать W и R?**
   - W + R > N для consistency
   - Зависит от read/write ratio

4. **Conflict resolution strategies?**
   - LWW: простой, потеря данных
   - Vector clocks: detect conflicts
   - CRDTs: no conflicts

---

[← Назад к списку тем](README.md)
