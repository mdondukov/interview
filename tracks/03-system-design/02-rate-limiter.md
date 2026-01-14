# 02. Rate Limiter

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Ограничить количество запросов per client/IP/API key
- Возвращать 429 Too Many Requests при превышении
- Разные лимиты для разных endpoints
- Опционально: burst allowance

### Нефункциональные
- Low latency (< 1ms overhead)
- High availability
- Accurate counting (minimal false positives/negatives)
- Distributed (работает с несколькими API servers)

---

## Где размещать Rate Limiter

```
Option 1: Client-side
+ Простота
- Легко обойти, не надёжно

Option 2: Server-side (middleware)
┌────────┐    ┌───────────────┐    ┌────────────┐
│ Client │───▶│ Rate Limiter  │───▶│ API Server │
└────────┘    └───────────────┘    └────────────┘

Option 3: API Gateway
┌────────┐    ┌─────────────┐    ┌────────────┐
│ Client │───▶│ API Gateway │───▶│ API Server │
└────────┘    │ (+ limiter) │    └────────────┘
              └─────────────┘

Option 4: Sidecar (Service Mesh)
```

**Рекомендация:** API Gateway или отдельный middleware сервис

---

## Алгоритмы Rate Limiting

### 1. Token Bucket

```
┌─────────────────────────────┐
│  Bucket (capacity = 4)      │
│  ● ● ● ●                    │
│  Tokens refill at rate R    │
└─────────────────────────────┘

1. Bucket has capacity N tokens
2. Tokens added at rate R per second
3. Each request consumes 1 token
4. If no tokens → reject
```

```python
import time
from dataclasses import dataclass

@dataclass
class TokenBucket:
    capacity: int
    tokens: float
    refill_rate: float  # tokens per second
    last_refill: float

    def allow_request(self) -> bool:
        now = time.time()

        # Refill tokens
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

**Pros:** Allows bursts up to capacity
**Cons:** Memory per user

### 2. Leaky Bucket

```
     Requests
        │
        ▼
┌───────────────┐
│   ● ● ● ●     │  Queue (fixed size)
└───────┬───────┘
        │ Fixed rate output
        ▼
    Processing
```

```python
from collections import deque
import time

class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate  # requests per second
        self.queue = deque()

    def allow_request(self) -> bool:
        now = time.time()

        # Leak old requests
        while self.queue and now - self.queue[0] > 1 / self.leak_rate:
            self.queue.popleft()

        if len(self.queue) < self.capacity:
            self.queue.append(now)
            return True
        return False
```

**Pros:** Smooth output rate
**Cons:** Doesn't handle bursts well

### 3. Fixed Window Counter

```
Window 1 (00:00-01:00)    Window 2 (01:00-02:00)
┌─────────────────────┐   ┌─────────────────────┐
│ Count: 100/100      │   │ Count: 50/100       │
└─────────────────────┘   └─────────────────────┘
                     ▲
               Boundary spike problem
```

```python
import time

class FixedWindowCounter:
    def __init__(self, limit: int, window_size: int):
        self.limit = limit
        self.window_size = window_size
        self.current_window = 0
        self.count = 0

    def allow_request(self) -> bool:
        now = int(time.time())
        window = now // self.window_size

        if window != self.current_window:
            self.current_window = window
            self.count = 0

        if self.count < self.limit:
            self.count += 1
            return True
        return False
```

**Pros:** Simple, memory efficient
**Cons:** Boundary spike (2x limit at window boundary)

### 4. Sliding Window Log

```
Keep timestamp of each request
┌─────────────────────────────────────────┐
│ [t1, t2, t3, t4, t5, t6, ...]           │
└─────────────────────────────────────────┘
Count requests in last N seconds
```

```python
import time
from collections import deque

class SlidingWindowLog:
    def __init__(self, limit: int, window_size: int):
        self.limit = limit
        self.window_size = window_size
        self.timestamps = deque()

    def allow_request(self) -> bool:
        now = time.time()

        # Remove old timestamps
        while self.timestamps and now - self.timestamps[0] > self.window_size:
            self.timestamps.popleft()

        if len(self.timestamps) < self.limit:
            self.timestamps.append(now)
            return True
        return False
```

**Pros:** Accurate, no boundary spikes
**Cons:** Memory intensive (store all timestamps)

### 5. Sliding Window Counter (Best)

```
Combines Fixed Window + weighted previous window

Current window: 70% through
Previous window count: 80
Current window count: 30

Weighted count = 80 × 0.3 + 30 = 54
```

```python
import time

class SlidingWindowCounter:
    def __init__(self, limit: int, window_size: int):
        self.limit = limit
        self.window_size = window_size
        self.prev_count = 0
        self.curr_count = 0
        self.curr_window = 0

    def allow_request(self) -> bool:
        now = time.time()
        window = int(now // self.window_size)
        window_position = (now % self.window_size) / self.window_size

        if window != self.curr_window:
            self.prev_count = self.curr_count if window == self.curr_window + 1 else 0
            self.curr_count = 0
            self.curr_window = window

        # Weighted count
        weighted_count = self.prev_count * (1 - window_position) + self.curr_count

        if weighted_count < self.limit:
            self.curr_count += 1
            return True
        return False
```

**Pros:** Memory efficient, accurate
**Cons:** Slightly more complex

---

## Distributed Rate Limiting

### Problem
Multiple API servers need consistent rate limiting.

### Solution 1: Centralized Redis

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Server1 │     │ Server2 │     │ Server3 │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
     └───────────────┼───────────────┘
                     │
              ┌──────▼──────┐
              │    Redis    │
              │  (counters) │
              └─────────────┘
```

```python
import redis
import time

class DistributedRateLimiter:
    def __init__(self, redis_client, limit: int, window: int):
        self.redis = redis_client
        self.limit = limit
        self.window = window

    def allow_request(self, client_id: str) -> bool:
        key = f"rate_limit:{client_id}"
        now = time.time()
        window_start = now - self.window

        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)  # Remove old
        pipe.zadd(key, {str(now): now})              # Add current
        pipe.zcard(key)                               # Count
        pipe.expire(key, self.window)                 # Set TTL
        _, _, count, _ = pipe.execute()

        return count <= self.limit
```

### Solution 2: Token Bucket in Redis

```lua
-- Lua script for atomic token bucket
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return 1
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    return 0
end
```

### Solution 3: Sticky Sessions

```
Route same client to same server (by IP hash)
Each server maintains local rate limit

Pros: No distributed state
Cons: Uneven distribution, failure handling
```

---

## High-Level Design

```
┌────────────┐
│   Client   │
└─────┬──────┘
      │
┌─────▼──────┐     ┌───────────────────┐
│    LB      │     │  Rules Engine     │
└─────┬──────┘     │  (per endpoint,   │
      │            │   per user tier)  │
┌─────▼──────┐     └─────────┬─────────┘
│ API Gateway│◄──────────────┘
│ + Limiter  │
└─────┬──────┘
      │
┌─────▼──────┐     ┌─────────────────┐
│   Redis    │◄────│  Config Service │
│  Cluster   │     │  (rate limits)  │
└────────────┘     └─────────────────┘
```

---

## API Design

### Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1623456789
```

### Rate Limited Response

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1623456789
Retry-After: 30

{
    "error": "Rate limit exceeded",
    "retry_after": 30
}
```

---

## Trade-offs

### Algorithm Comparison

| Algorithm | Accuracy | Memory | Burst | Complexity |
|-----------|----------|--------|-------|------------|
| Token Bucket | High | Medium | Yes | Low |
| Leaky Bucket | High | Medium | No | Low |
| Fixed Window | Low | Low | No | Low |
| Sliding Log | High | High | Yes | Medium |
| Sliding Counter | High | Low | Partial | Medium |

### Local vs Distributed

| Approach | Latency | Accuracy | Consistency |
|----------|---------|----------|-------------|
| Local only | <0.1ms | Low | Weak |
| Redis sync | 1-5ms | High | Strong |
| Redis async | <0.1ms | Medium | Eventual |

---

## На интервью

### Ключевые моменты
1. **Algorithm choice** — sliding window counter лучший баланс
2. **Distributed state** — Redis как single source of truth
3. **Graceful degradation** — что делать если Redis недоступен
4. **Different limits** — per user, per IP, per endpoint

### Типичные follow-up
- Как реализовать разные лимиты для разных пользователей (free/paid)?
- Что делать при отказе Redis?
- Как избежать race conditions?
- Как scale Redis cluster?

---

## См. также

- [Паттерны конкурентности в Go](../00-go/05-concurrency-patterns.md) — реализация rate limiter на уровне приложения
- [Паттерны использования Redis](../06-databases/09-redis-patterns.md) — распределённые счётчики и блокировки

---

[← Назад к списку тем](README.md)
