# 07. Distributed Cache

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Key-value storage (GET, SET, DELETE)
- TTL (Time To Live)
- Поддержка различных data types
- Atomic operations (INCR, DECR)
- Опционально: pub/sub, Lua scripts

### Нефункциональные
- Ultra-low latency (< 1ms for reads)
- High throughput (100K+ ops/sec per node)
- High availability (99.99%)
- Horizontal scalability
- Consistency (configurable)

---

## Capacity Estimation

```
Operations:
- 10M DAU application
- 1000 cache ops per user session
- 10B cache operations/day
- QPS = 10B / 86400 ≈ 115,000 QPS
- Peak: 300,000 QPS

Storage:
- 100M cached objects
- Average object size: 1KB
- Total: 100GB (fits in RAM cluster)

Memory per node:
- 64GB RAM nodes
- ~60GB usable for cache
- 2 nodes minimum for 100GB with replication
```

---

## High-Level Design

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Application Layer                            │
└────────────────────────────────────┬─────────────────────────────────┘
                                     │
┌────────────────────────────────────▼─────────────────────────────────┐
│                          Cache Client Library                         │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐  │
│  │ Connection Pool│  │ Consistent Hash│  │ Serialization          │  │
│  └────────────────┘  └────────────────┘  └────────────────────────┘  │
└────────────────────────────────────┬─────────────────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
   ┌─────▼─────┐               ┌─────▼─────┐               ┌─────▼─────┐
   │  Cache    │               │  Cache    │               │  Cache    │
   │  Node 1   │               │  Node 2   │               │  Node 3   │
   │  (Primary)│               │  (Primary)│               │  (Primary)│
   └─────┬─────┘               └─────┬─────┘               └─────┬─────┘
         │                           │                           │
   ┌─────▼─────┐               ┌─────▼─────┐               ┌─────▼─────┐
   │  Cache    │               │  Cache    │               │  Cache    │
   │  Node 1'  │               │  Node 2'  │               │  Node 3'  │
   │  (Replica)│               │  (Replica)│               │  (Replica)│
   └───────────┘               └───────────┘               └───────────┘
```

---

## API Design

### Basic Operations
```python
# SET with TTL
cache.set("user:123", user_data, ttl=3600)

# GET
user = cache.get("user:123")

# DELETE
cache.delete("user:123")

# MGET (batch)
users = cache.mget(["user:1", "user:2", "user:3"])

# MSET (batch)
cache.mset({"user:1": data1, "user:2": data2})
```

### Atomic Operations
```python
# Increment counter
cache.incr("pageviews:homepage")

# Compare-and-swap
cache.cas("inventory:item123", old_value, new_value)

# Distributed lock
lock = cache.lock("resource:xyz", timeout=10)
try:
    # Critical section
finally:
    lock.release()
```

### Redis Protocol
```
SET key value [EX seconds] [PX milliseconds] [NX|XX]
GET key
DEL key [key ...]
MGET key [key ...]
EXPIRE key seconds
TTL key
INCR key
SETNX key value
```

---

## Data Model

### Key Design
```
# Pattern: {entity}:{id}:{attribute}
user:123:profile
user:123:settings
session:abc123
cache:api:/users/123
rate_limit:user:123:endpoint:/api/posts

# Avoid
too_long_keys_waste_memory_and_bandwidth
```

### Value Types (Redis)
```
STRING: "hello"
  - Simple values, JSON, serialized objects

LIST: [a, b, c]
  - Message queues, recent items

SET: {a, b, c}
  - Unique items, tags

SORTED SET: {(score, member), ...}
  - Leaderboards, time-series

HASH: {field: value, ...}
  - Objects with multiple fields
```

---

## Deep Dive

### 1. Consistent Hashing

```
Problem: Adding/removing nodes shouldn't invalidate entire cache

┌────────────────────────────────────────────────┐
│                  Hash Ring                      │
│                                                │
│            Node A (0°)                         │
│              ┌──┐                              │
│         ─────│  │─────                         │
│      ──      └──┘      ──                      │
│    ──    Key1 ●          ──    Node B (90°)   │
│   ──       ↓               ──  ┌──┐           │
│  │    → routes to A         │──│  │           │
│   ──                       ──  └──┘           │
│    ──   Key2 ●           ──                   │
│      ──    ↓          ──                      │
│         ─────│  │─────    → routes to B       │
│              └──┘                              │
│            Node C (180°)                       │
└────────────────────────────────────────────────┘
```

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, nodes: list, virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}  # hash -> node
        self.sorted_hashes = []

        for node in nodes:
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_val = self._hash(virtual_key)
            self.ring[hash_val] = node
            self.sorted_hashes.append(hash_val)

        self.sorted_hashes.sort()

    def remove_node(self, node: str):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_val = self._hash(virtual_key)
            del self.ring[hash_val]
            self.sorted_hashes.remove(hash_val)

    def get_node(self, key: str) -> str:
        if not self.ring:
            return None

        hash_val = self._hash(key)
        idx = bisect_right(self.sorted_hashes, hash_val)

        if idx == len(self.sorted_hashes):
            idx = 0

        return self.ring[self.sorted_hashes[idx]]
```

### 2. Cache Eviction Policies

```
┌─────────────────────────────────────────────────────────────┐
│                    Eviction Policies                         │
├─────────────────────────────────────────────────────────────┤
│ LRU (Least Recently Used)                                   │
│   - Evict keys not accessed recently                        │
│   - Good for general workloads                              │
│   - Redis: maxmemory-policy allkeys-lru                     │
├─────────────────────────────────────────────────────────────┤
│ LFU (Least Frequently Used)                                 │
│   - Evict keys accessed less frequently                     │
│   - Better for skewed access patterns                       │
│   - Redis: maxmemory-policy allkeys-lfu                     │
├─────────────────────────────────────────────────────────────┤
│ TTL-based                                                   │
│   - Evict keys with nearest expiration                      │
│   - Redis: maxmemory-policy volatile-ttl                    │
├─────────────────────────────────────────────────────────────┤
│ Random                                                      │
│   - Random eviction                                         │
│   - Lowest overhead                                         │
│   - Redis: maxmemory-policy allkeys-random                  │
└─────────────────────────────────────────────────────────────┘
```

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: str):
        if key not in self.cache:
            return None

        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def set(self, key: str, value, ttl: int = None):
        if key in self.cache:
            self.cache.move_to_end(key)
        else:
            if len(self.cache) >= self.capacity:
                # Evict least recently used (first item)
                self.cache.popitem(last=False)

        self.cache[key] = value
```

### 3. Cache Invalidation Patterns

**Pattern 1: Cache-Aside (Lazy Loading)**
```python
def get_user(user_id: str):
    # 1. Check cache
    user = cache.get(f"user:{user_id}")
    if user:
        return deserialize(user)

    # 2. Cache miss - load from DB
    user = db.get_user(user_id)
    if user:
        cache.set(f"user:{user_id}", serialize(user), ttl=3600)

    return user

def update_user(user_id: str, data: dict):
    # 1. Update DB
    db.update_user(user_id, data)

    # 2. Invalidate cache
    cache.delete(f"user:{user_id}")
```

**Pattern 2: Write-Through**
```python
def update_user(user_id: str, data: dict):
    # 1. Update cache AND DB together
    user = {**get_user(user_id), **data}

    # Write to cache first
    cache.set(f"user:{user_id}", serialize(user))

    # Write to DB
    db.update_user(user_id, user)
```

**Pattern 3: Write-Behind (Write-Back)**
```python
def update_user(user_id: str, data: dict):
    # 1. Update cache immediately
    cache.set(f"user:{user_id}", serialize(data))

    # 2. Queue async write to DB
    write_queue.send({
        "operation": "update",
        "table": "users",
        "id": user_id,
        "data": data
    })

# Background worker
async def process_writes():
    while True:
        batch = await write_queue.get_batch(100)
        await db.bulk_write(batch)
```

### 4. Cache Stampede Prevention

```
Problem: When cache expires, many requests hit DB simultaneously

Solution 1: Locking
```python
def get_with_lock(key: str, fetch_func):
    value = cache.get(key)
    if value:
        return value

    lock_key = f"lock:{key}"

    # Try to acquire lock
    if cache.setnx(lock_key, "1", ex=10):
        try:
            # Won the lock - fetch data
            value = fetch_func()
            cache.set(key, value, ttl=3600)
            return value
        finally:
            cache.delete(lock_key)
    else:
        # Lost the lock - wait and retry
        time.sleep(0.1)
        return get_with_lock(key, fetch_func)
```

```
Solution 2: Probabilistic Early Expiration
```python
import random
import time

def get_with_early_refresh(key: str, fetch_func, ttl: int = 3600):
    value, expiry = cache.get_with_expiry(key)

    if value is None:
        # Cache miss
        value = fetch_func()
        cache.set(key, value, ttl=ttl)
        return value

    # Probabilistic early refresh
    remaining_ttl = expiry - time.time()
    refresh_probability = 1 - (remaining_ttl / ttl)

    if random.random() < refresh_probability * 0.1:  # 10% max probability
        # Refresh in background
        asyncio.create_task(refresh_cache(key, fetch_func, ttl))

    return value
```

### 5. Redis Cluster Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Redis Cluster                               │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Hash Slots (0 - 16383)                      │  │
│  │  ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐            │  │
│  │  │0-5460 │  │5461-  │  │10923- │  │ ...   │            │  │
│  │  │       │  │ 10922 │  │ 16383 │  │       │            │  │
│  │  └───┬───┘  └───┬───┘  └───┬───┘  └───────┘            │  │
│  │      │          │          │                            │  │
│  │  ┌───▼───┐  ┌───▼───┐  ┌───▼───┐                       │  │
│  │  │Node 1 │  │Node 2 │  │Node 3 │  (Masters)            │  │
│  │  └───┬───┘  └───┬───┘  └───┬───┘                       │  │
│  │      │          │          │                            │  │
│  │  ┌───▼───┐  ┌───▼───┐  ┌───▼───┐                       │  │
│  │  │Node 1'│  │Node 2'│  │Node 3'│  (Replicas)           │  │
│  │  └───────┘  └───────┘  └───────┘                       │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

Key → CRC16(key) % 16384 → Slot → Node
```

```python
class RedisClusterClient:
    def __init__(self, nodes: list):
        self.nodes = nodes
        self.slot_map = {}  # slot -> node
        self.refresh_slots()

    def refresh_slots(self):
        # Get cluster slots from any node
        slots_info = self.nodes[0].cluster_slots()
        for start_slot, end_slot, master, *replicas in slots_info:
            for slot in range(start_slot, end_slot + 1):
                self.slot_map[slot] = master

    def get_slot(self, key: str) -> int:
        # Handle hash tags: {user}:profile → slot based on "user"
        if '{' in key and '}' in key:
            start = key.index('{') + 1
            end = key.index('}')
            key = key[start:end]

        return crc16(key.encode()) % 16384

    def get_node(self, key: str):
        slot = self.get_slot(key)
        return self.slot_map[slot]

    def get(self, key: str):
        node = self.get_node(key)
        try:
            return node.get(key)
        except MovedError as e:
            # Cluster resharding - refresh slot map
            self.refresh_slots()
            return self.get(key)
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Hot keys | Local caching, key replication |
| Large values | Compression, chunking |
| Network latency | Connection pooling, pipelining |
| Memory fragmentation | Regular restarts, jemalloc |
| Node failure | Replication, automatic failover |

---

## Trade-offs

### Consistency Models

| Model | Latency | Consistency | Use Case |
|-------|---------|-------------|----------|
| No replication | Lowest | N/A | Dev/test |
| Async replication | Low | Eventual | Most cases |
| Sync replication | Higher | Strong | Financial |

### Persistence Options

| Option | Durability | Performance | Recovery |
|--------|------------|-------------|----------|
| No persistence | None | Highest | N/A |
| RDB snapshots | Point-in-time | High | Minutes |
| AOF | Per operation | Medium | Seconds |
| RDB + AOF | Best | Lower | Best |

---

## На интервью

### Ключевые моменты
1. **Consistent hashing** — для распределения и масштабирования
2. **Eviction policies** — LRU vs LFU vs TTL
3. **Cache invalidation** — cache-aside, write-through, write-behind
4. **Stampede prevention** — locking, probabilistic refresh

### Типичные follow-up
- Как обработать hot keys (celebrity problem)?
- Как обеспечить consistency между cache и DB?
- Как мониторить hit rate и latency?
- Как мигрировать на новую версию без downtime?
- Как реализовать distributed lock?

---

## См. также

- [Паттерны использования Redis](../06-databases/09-redis-patterns.md) — практические паттерны работы с Redis
- [CAP-теорема](../07-distributed-systems/01-cap-theorem.md) — компромиссы между консистентностью и доступностью

---

[← Назад к списку тем](README.md)
