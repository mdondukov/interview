# 13. Distributed ID Generator

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Generate globally unique IDs
- IDs must be sortable by time (roughly)
- Support multiple datacenters
- No single point of failure

### Нефункциональные
- High throughput (100K+ IDs/sec per node)
- Low latency (< 1ms)
- High availability (99.99%)
- No coordination required (or minimal)
- IDs should be compact (64-128 bits)

---

## Capacity Estimation

```
Requirements:
- 100K IDs/second per node
- 10 nodes per datacenter
- 5 datacenters
- Total: 5M IDs/second globally

ID Space:
- 64-bit ID = 2^64 = 18 quintillion unique IDs
- At 5M/sec: 116,000 years before exhaustion

Storage overhead:
- UUID (128-bit): 16 bytes
- Snowflake (64-bit): 8 bytes
- Per 1B records: 8GB vs 16GB difference
```

---

## Common Approaches

### 1. UUID (Universally Unique Identifier)

```
UUID v4 (Random):
550e8400-e29b-41d4-a716-446655440000
│        │    │    │    │
└────────┴────┴────┴────┴── 128 bits, random

UUID v1 (Time-based):
6ba7b810-9dad-11d1-80b4-00c04fd430c8
│        │    │    │    │
│        │    │    │    └── Node (MAC address)
│        │    │    └─────── Clock sequence
│        │    └──────────── Version + variant
└────────┴───────────────── Timestamp
```

```python
import uuid

# UUID v4 (random)
id = uuid.uuid4()  # 550e8400-e29b-41d4-a716-446655440000

# UUID v1 (time-based)
id = uuid.uuid1()  # 6ba7b810-9dad-11d1-80b4-00c04fd430c8
```

| Pros | Cons |
|------|------|
| No coordination | 128 bits (large) |
| Simple | Not sortable (v4) |
| Universal support | Poor index performance |

---

### 2. Snowflake ID (Twitter)

```
┌─────────────────────────────────────────────────────────────────┐
│                        64-bit Snowflake ID                       │
├─────────────────────────────────────────────────────────────────┤
│  1 bit  │      41 bits       │  10 bits  │      12 bits        │
│ (sign)  │    (timestamp)     │ (machine) │    (sequence)       │
│    0    │ ms since epoch     │   ID      │ counter per ms      │
├─────────────────────────────────────────────────────────────────┤
│    0    │ 1704067200000      │   42      │      1234           │
└─────────────────────────────────────────────────────────────────┘

Total: 64 bits
- 1 bit: Always 0 (positive number)
- 41 bits: Timestamp (69 years from custom epoch)
- 10 bits: Machine ID (1024 machines)
- 12 bits: Sequence (4096 per ms per machine)

Max throughput: 4096 × 1000 = 4M IDs/sec per machine
```

```python
import time
import threading

class Snowflake:
    # Custom epoch (2024-01-01 00:00:00 UTC)
    EPOCH = 1704067200000

    MACHINE_BITS = 10
    SEQUENCE_BITS = 12

    MAX_MACHINE_ID = (1 << MACHINE_BITS) - 1  # 1023
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1   # 4095

    def __init__(self, machine_id: int):
        if machine_id < 0 or machine_id > self.MAX_MACHINE_ID:
            raise ValueError(f"Machine ID must be 0-{self.MAX_MACHINE_ID}")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def generate(self) -> int:
        with self.lock:
            timestamp = self._current_time()

            if timestamp < self.last_timestamp:
                raise ClockMovedBackwardError()

            if timestamp == self.last_timestamp:
                # Same millisecond - increment sequence
                self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE
                if self.sequence == 0:
                    # Sequence overflow - wait for next ms
                    timestamp = self._wait_next_millis(timestamp)
            else:
                # New millisecond - reset sequence
                self.sequence = 0

            self.last_timestamp = timestamp

            # Compose ID
            id = ((timestamp - self.EPOCH) << (self.MACHINE_BITS + self.SEQUENCE_BITS)) | \
                 (self.machine_id << self.SEQUENCE_BITS) | \
                 self.sequence

            return id

    def _current_time(self) -> int:
        return int(time.time() * 1000)

    def _wait_next_millis(self, last_ts: int) -> int:
        ts = self._current_time()
        while ts <= last_ts:
            ts = self._current_time()
        return ts

    @staticmethod
    def parse(id: int) -> dict:
        """Extract components from Snowflake ID"""
        sequence = id & Snowflake.MAX_SEQUENCE
        machine_id = (id >> Snowflake.SEQUENCE_BITS) & Snowflake.MAX_MACHINE_ID
        timestamp = (id >> (Snowflake.MACHINE_BITS + Snowflake.SEQUENCE_BITS)) + Snowflake.EPOCH

        return {
            "timestamp": timestamp,
            "machine_id": machine_id,
            "sequence": sequence,
            "datetime": datetime.fromtimestamp(timestamp / 1000)
        }
```

| Pros | Cons |
|------|------|
| 64 bits (compact) | Needs machine ID coordination |
| Sortable by time | Clock sync issues |
| High throughput | 69-year limit from epoch |
| No central coordination | Machine ID must be unique |

---

### 3. ULID (Universally Unique Lexicographically Sortable ID)

```
┌─────────────────────────────────────────────────────────────────┐
│                        128-bit ULID                              │
├─────────────────────────────────────────────────────────────────┤
│           48 bits              │           80 bits              │
│         (timestamp)            │          (random)              │
│     ms since Unix epoch        │    cryptographic random        │
├─────────────────────────────────────────────────────────────────┤
│ 01ARZ3NDEKTSV4RRFFQ69G5FAV                                      │
│ └──────────┘└──────────────────┘                                │
│   timestamp     randomness                                      │
└─────────────────────────────────────────────────────────────────┘

Encoding: Crockford's Base32 (26 characters)
- Case insensitive
- URL safe
- Lexicographically sortable
```

```python
import os
import time

class ULID:
    ENCODING = "0123456789ABCDEFGHJKMNPQRSTVWXYZ"  # Crockford's Base32

    @classmethod
    def generate(cls) -> str:
        timestamp = int(time.time() * 1000)
        randomness = os.urandom(10)

        # Encode timestamp (48 bits = 10 characters)
        ts_chars = []
        for _ in range(10):
            ts_chars.append(cls.ENCODING[timestamp & 0x1F])
            timestamp >>= 5
        timestamp_str = ''.join(reversed(ts_chars))

        # Encode randomness (80 bits = 16 characters)
        rand_int = int.from_bytes(randomness, 'big')
        rand_chars = []
        for _ in range(16):
            rand_chars.append(cls.ENCODING[rand_int & 0x1F])
            rand_int >>= 5
        random_str = ''.join(reversed(rand_chars))

        return timestamp_str + random_str

    @classmethod
    def parse(cls, ulid_str: str) -> dict:
        # Decode timestamp
        timestamp = 0
        for char in ulid_str[:10]:
            timestamp = (timestamp << 5) | cls.ENCODING.index(char.upper())

        return {
            "timestamp": timestamp,
            "datetime": datetime.fromtimestamp(timestamp / 1000),
            "randomness": ulid_str[10:]
        }
```

| Pros | Cons |
|------|------|
| Sortable | 128 bits (larger than Snowflake) |
| No coordination | Random collision possible |
| URL safe | Newer, less adoption |
| Case insensitive | |

---

### 4. Database Sequences

```sql
-- PostgreSQL
CREATE SEQUENCE global_id_seq
    START WITH 1
    INCREMENT BY 1
    NO MAXVALUE
    CACHE 1000;  -- Pre-allocate 1000 IDs

SELECT nextval('global_id_seq');

-- MySQL Auto-increment
CREATE TABLE id_generator (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    stub CHAR(1) NOT NULL DEFAULT ''
) ENGINE=InnoDB;

REPLACE INTO id_generator (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

**Distributed approach (Flickr-style):**
```sql
-- Server 1: Odd IDs
SET @@auto_increment_increment = 2;
SET @@auto_increment_offset = 1;
-- Generates: 1, 3, 5, 7, ...

-- Server 2: Even IDs
SET @@auto_increment_increment = 2;
SET @@auto_increment_offset = 2;
-- Generates: 2, 4, 6, 8, ...
```

| Pros | Cons |
|------|------|
| Simple | Single point of failure |
| Sequential | Network roundtrip |
| DB-native | Scalability limited |

---

### 5. Redis-based

```python
class RedisIDGenerator:
    def __init__(self, redis_client, key_prefix: str = "id"):
        self.redis = redis_client
        self.key = f"{key_prefix}:counter"

    async def generate(self) -> int:
        return await self.redis.incr(self.key)

    async def generate_batch(self, count: int) -> list:
        """Generate multiple IDs efficiently"""
        end = await self.redis.incrby(self.key, count)
        start = end - count + 1
        return list(range(start, end + 1))
```

**Lua script for Snowflake-like IDs:**
```lua
-- Redis Lua script for atomic ID generation
local timestamp = redis.call('TIME')
local ms = tonumber(timestamp[1]) * 1000 + math.floor(tonumber(timestamp[2]) / 1000)

local machine_id = tonumber(KEYS[1])
local sequence_key = 'seq:' .. machine_id .. ':' .. ms

local sequence = redis.call('INCR', sequence_key)
redis.call('EXPIRE', sequence_key, 1)

if sequence > 4095 then
    return nil  -- Sequence overflow
end

local id = bit.bor(
    bit.lshift(ms - 1704067200000, 22),
    bit.lshift(machine_id, 12),
    sequence
)

return id
```

---

## System Design

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      ID Generation Service                        │
└──────────────────────────────────────────────────────────────────┘

┌──────────┐     ┌─────────────┐     ┌──────────────────────────────┐
│  Client  │────▶│     LB      │────▶│      ID Generator Nodes      │
└──────────┘     └─────────────┘     │  ┌──────┐ ┌──────┐ ┌──────┐ │
                                     │  │Node 1│ │Node 2│ │Node N│ │
                                     │  │ID=001│ │ID=002│ │ID=N  │ │
                                     │  └──────┘ └──────┘ └──────┘ │
                                     └──────────────────────────────┘
                                                    │
                                     ┌──────────────┼──────────────┐
                                     │              │              │
                                ┌────▼────┐   ┌────▼────┐    ┌────▼────┐
                                │ ZK/etcd │   │  Redis  │    │   DB    │
                                │(coord)  │   │(counter)│    │(backup) │
                                └─────────┘   └─────────┘    └─────────┘
```

### Machine ID Assignment

```python
class MachineIDManager:
    """
    Coordinate machine ID assignment using ZooKeeper/etcd
    """

    def __init__(self, zk_client, max_machines: int = 1024):
        self.zk = zk_client
        self.max_machines = max_machines
        self.machine_id = None

    async def acquire_machine_id(self) -> int:
        # Try to acquire an available machine ID
        for candidate_id in range(self.max_machines):
            path = f"/machine_ids/{candidate_id}"

            try:
                # Ephemeral node - auto-deleted when connection lost
                await self.zk.create(
                    path,
                    value=self.get_node_info(),
                    ephemeral=True
                )
                self.machine_id = candidate_id
                return candidate_id

            except NodeExistsError:
                continue

        raise NoAvailableMachineIDError()

    async def release_machine_id(self):
        if self.machine_id is not None:
            path = f"/machine_ids/{self.machine_id}"
            await self.zk.delete(path)
            self.machine_id = None
```

### Handling Clock Drift

```python
class ClockAwareSnowflake(Snowflake):
    MAX_BACKWARD_MS = 5  # Allow 5ms backward drift

    def generate(self) -> int:
        with self.lock:
            timestamp = self._current_time()

            if timestamp < self.last_timestamp:
                drift = self.last_timestamp - timestamp

                if drift <= self.MAX_BACKWARD_MS:
                    # Small drift - wait it out
                    time.sleep(drift / 1000)
                    timestamp = self._current_time()
                else:
                    # Large drift - reject
                    raise ClockMovedBackwardError(drift)

            # Continue with normal generation
            return super()._generate_with_timestamp(timestamp)
```

---

## Comparison

| Approach | Bits | Sortable | Coordination | Throughput | Use Case |
|----------|------|----------|--------------|------------|----------|
| UUID v4 | 128 | No | None | High | Simple, no order needed |
| UUID v1 | 128 | Yes | None | High | Time-ordered, MAC exposure |
| Snowflake | 64 | Yes | Machine ID | 4M/s/node | High-scale, time-ordered |
| ULID | 128 | Yes | None | High | Distributed, sortable |
| DB Sequence | 64 | Yes | Central | Medium | Simple, low scale |
| Redis | 64 | Partial | Central | High | Centralized, simple |

---

## Trade-offs

### Coordination vs Simplicity

| Approach | Coordination | Complexity | Failure Mode |
|----------|--------------|------------|--------------|
| UUID | None | Simple | N/A |
| Snowflake | Machine ID | Medium | Machine ID collision |
| DB Sequence | Full | Simple | SPOF |
| Raft-based | Consensus | High | Split-brain |

### Size vs Information

| Format | Size | Information Encoded |
|--------|------|---------------------|
| UUID v4 | 128 bits | None (random) |
| UUID v1 | 128 bits | Time, Node |
| Snowflake | 64 bits | Time, Machine, Sequence |
| ULID | 128 bits | Time, Random |

---

## На интервью

### Ключевые моменты
1. **Requirements clarification** — sortable? size constraints? throughput?
2. **Trade-offs** — coordination vs simplicity, size vs information
3. **Clock handling** — NTP sync, backward drift tolerance
4. **Machine ID** — static config vs dynamic assignment

### Типичные follow-up
- Как обработать clock drift между серверами?
- Как масштабировать на несколько датацентров?
- Как восстановить после failure (потеря machine ID)?
- Как гарантировать uniqueness при split-brain?
- Почему не использовать просто timestamp + random?

---

[← Назад к списку тем](README.md)
