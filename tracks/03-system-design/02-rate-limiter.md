# 02 — Rate Limiter

Развёрнутые вопросы и ответы про rate limiting: алгоритмы, distributed implementation, HTTP headers, масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [01-url-shortener](./01-url-shortener.md) · Следующая тема: [03-notification-system](./03-notification-system.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Rate limiting** — механизм ограничения количества запросов, которые клиент может отправить в течение определённого промежутка времени. Rate limiting защищает серверы от перегрузки, DDoS атак и обеспечивает справедливое распределение ресурсов между пользователями. Это критически важный компонент для масштабируемых систем, работающих под высокой нагрузкой.

**Token bucket** — популярный алгоритм rate limiting, в котором токены накапливаются в "ведре" с фиксированной скоростью. Каждый входящий запрос требует один токен; если токенов нет, запрос отклоняется или ставится в очередь. Token bucket позволяет поддерживать burst трафика (временные пики), но в долгосрочной перспективе ограничивает среднюю скорость.

**Leaky bucket** — альтернативный алгоритм, где все входящие запросы встают в очередь и обрабатываются с фиксированной, гладкой скоростью. Это гарантирует предсказуемый output без резких пиков нагрузки. Leaky bucket особенно полезен для предотвращения наводнения downstream сервисов.

**API Gateway** — единая точка входа для всех входящих API запросов к системе. API Gateway часто содержит rate limiter на самом верхнем уровне, что позволяет эффективно отбросить плохие запросы до того, как они достигнут backend сервисов. Это более дешево, чем обрабатывать violations на каждом отдельном сервисе.

**Sliding window** — алгоритм, отслеживающий количество запросов в скользящем окне времени (например, за последние 60 секунд). Sliding window более точен, чем fixed window (поминутный счётчик), так как не имеет проблемы с граничными эффектами. Однако требует больше памяти для хранения меток времени запросов.

**Capacity** — максимальное количество токенов, которые могут одновременно находиться в token bucket. Capacity определяет допустимый размер burst трафика; если пришёл burst, система может обработать до Capacity запросов без задержки. Больший capacity позволяет более крупные bursts, но требует больше памяти.

**Refill rate** — скорость, с которой токены добавляются обратно в bucket (обычно измеряется в токенах в секунду). Refill rate определяет долгосрочную пропускную способность системы. Например, refill rate 100 токенов/сек означает, что в длительной перспективе система может обработать 100 запросов в секунду.

**429 Too Many Requests** — стандартный HTTP статус код, который сервер возвращает при превышении rate limit. Этот код сигнализирует клиенту, что запрос был отклонён из-за ограничений лимита, а не из-за ошибки в самом запросе. Клиент должен распознавать этот код и повторить попытку позже.

**Retry-After** — HTTP заголовок, который сервер отправляет вместе с 429 ответом, указывая клиенту, когда он может безопасно повторить запрос. Retry-After может быть выражен в секундах (Retry-After: 120) или как дата. Это помогает клиентам правильно распределять повторные попытки вместо случайной отправки.

**Circuit breaker** — механизм отказа в режиме fail-fast, который предотвращает каскадные сбои в системе. Когда сервис перегружен или недоступен, circuit breaker быстро отклоняет новые запросы вместо того, чтобы ждать timeout. Это позволяет защитить backend и позволить ему восстановиться.

---

## Вопросы и разборы

### 1. Зачем нужен rate limiter и где его размещать?

**Зачем спрашивают.** Rate limiting — основной механизм защиты API. Интервьюер проверяет понимание целей, компромиссов между разными позициями в архитектуре, и влияния на пользовательский опыт.

**Короткий ответ.** Rate limiter ограничивает количество запросов от клиента, защищая backend от перегрузки. Размещать лучше на API Gateway или отдельном middleware сервисе перед основными серверами — это дешевле, чем обрабатывать bad requests внутри приложения.

**Детальный разбор.**

**Зачем rate limiting:**
1. **DoS/DDoS protection** — не дать одному клиенту забить весь сервис
2. **Fair resource allocation** — каждый пользователь получает справедливую долю
3. **Database protection** — не перегружать БД дорогостоящими запросами
4. **Cost control** — ограничить использование expensive external APIs
5. **SLA compliance** — гарантировать service level для платных клиентов

**Где размещать:**

```
┌─────────────┐
│   Клиент    │
│   (можно    │
│  игнориро-  │
│   вать)     │
└──────┬──────┘
       │
   ┌───▼──────────────────────────────────┐
   │ API Gateway + Rate Limiter (лучше!)  │
   │ - Перехватывает запрос до backend    │
   │ - Экономит ресурсы приложения        │
   │ - Единая точка контроля              │
   │ - Легко масштабируется               │
   └───┬───────┬────────┬────────┬────────┘
       │       │        │        │
   ┌───▼──┐ ┌──▼───┐ ┌──▼───┐ ┌──▼───┐
   │App1  │ │App2  │ │App3  │ │App4  │
   └──────┘ └──────┘ └──────┘ └──────┘
       │        │        │        │
       └────────┼────────┼────────┘
                │
           ┌────▼───┐
           │ Redis  │
           │(state) │
           └────────┘

Альтернатива: Middleware в приложении
┌──────────┐
│ Client   │
└────┬─────┘
     │
┌────▼──────────────────┐
│ API Server #1         │
│ ├─ Rate Limiter       │
│ └─ Request Handler    │
└───────────────────────┘

❌ Проблемы:
  - Каждый сервер независимо считает
  - Легко обойти load balancer
  - Глобальные лимиты невозможны
  - Дорого обрабатывать rejected реквесты
```

**Сравнение позиций:**

| Позиция | Плюсы | Минусы | Когда использовать |
|---------|-------|--------|-------------------|
| **Client-side** | Экономит bandwidth | Легко обойти, ненадёжно | Только доп. уровень |
| **API Gateway** | Единая точка, экономит CPU | SPOF, нужна HA | Production-ready |
| **Middleware** | Гибко, flexible logic | Распределённое состояние | Локальные лимиты |
| **Sidecar (Istio)** | Per-pod контроль, прозрачно | Сложность операций | Kubernetes только |
| **External service** | Максимальная гибкость | Дополнительная latency | Очень strict requirements |

**Рекомендация:** API Gateway в production для глобальных лимитов + локальный middleware для per-endpoint fine-tuning.

**Пример.**

```python
# 1. Client-side (informational only, easily bypassed)
import requests
import time

class ClientRateLimiter:
    def __init__(self, max_requests=10, window=60):
        self.max_requests = max_requests
        self.window = window
        self.requests = []

    def can_request(self):
        now = time.time()
        # Remove old requests outside window
        self.requests = [t for t in self.requests if now - t < self.window]

        if len(self.requests) < self.max_requests:
            self.requests.append(now)
            return True
        return False

    def fetch(self, url):
        if not self.can_request():
            raise Exception("Rate limit exceeded")
        return requests.get(url)

# 2. API Gateway (recommended)
from flask import Flask, request, jsonify
from redis import Redis
import time

app = Flask(__name__)
redis = Redis(host='localhost')

def rate_limit_check(client_id, limit=100, window=60):
    """
    Sliding window counter in Redis.
    Accurate, memory-efficient, distributed.
    """
    key = f"rate_limit:{client_id}"
    now = time.time()
    window_start = now - window

    pipe = redis.pipeline()
    # Remove old timestamps
    pipe.zremrangebyscore(key, 0, window_start)
    # Add current timestamp
    pipe.zadd(key, {str(now): now})
    # Count requests in window
    pipe.zcard(key)
    # Set TTL
    pipe.expire(key, window + 1)

    results = pipe.execute()
    count = results[2]

    return count <= limit

@app.before_request
def check_rate_limit():
    client_id = request.remote_addr
    if not rate_limit_check(client_id, limit=100):
        return jsonify({"error": "Rate limit exceeded"}), 429

# 3. Graceful degradation (if Redis is down)
def rate_limit_with_fallback(client_id, limit=100):
    try:
        return rate_limit_check(client_id, limit)
    except Exception as e:
        print(f"Redis error: {e}, allowing request (degraded mode)")
        return True  # Fail open to avoid cascading failures

@app.errorhandler(429)
def rate_limit_handler(e):
    retry_after = redis.ttl(f"rate_limit:{request.remote_addr}")
    return jsonify({
        "error": "Rate limit exceeded",
        "retry_after": max(1, retry_after)
    }), 429, {
        "Retry-After": str(max(1, retry_after)),
        "X-RateLimit-Limit": "100",
        "X-RateLimit-Remaining": "0"
    }
```

**Типичные ошибки.**
- Полагаться на client-side rate limiting как на основное средство защиты.
- Размещать limiter внутри бизнес-логики приложения вместо gateway.
- Не предусматривать graceful degradation если Redis/external service недоступны.
- Забывать про burst traffic — пики легитимного использования (скажем, пользователь refreshит страницу несколько раз).
- Не логировать rate limit violations для анализа атак.

**На интервью.**
- Объясни, почему API Gateway лучше, чем middleware.
- Упомяни о graceful degradation.
- Уточняющий вопрос: «Как обрабатывать burst traffic?» — token bucket с большей capacity.
- «Как интегрировать разные лимиты для разных user tiers?» — rules engine с конфигом в Redis.

---

### 2. Как работает Token Bucket алгоритм?

**Зачем спрашивают.** Token Bucket — самый популярный алгоритм в production. Интервьюер проверяет понимание механизма, параметров (capacity, refill rate), и способности обрабатывать bursts.

**Короткий ответ.** Bucket содержит токены. Каждый запрос требует 1 токен. Токены пополняются с фиксированной скоростью R tokens/sec. Если токенов нет — запрос отклоняется. Позволяет bursts (до capacity) и smooth traffic на долгосроке.

**Детальный разбор.**

**Механизм работы:**

```
Capacity = 4 токена
Refill rate = 2 tokens/sec

t=0s:     ● ● ● ●    (4/4, полный)
          │ Request  (consume 1)
          ▼
t=0s:     ● ● ●      (3/4)
          │ Request  (consume 1)
          ▼
t=0s:     ● ●        (2/4)
          │ Wait...

t=0.5s:   ● ● ●      (3/4, refilled 1 token)
          │ Request  (consume 1)
          ▼
t=0.5s:   ● ●        (2/4)

t=1.0s:   ● ● ● ●    (4/4, refilled 2 tokens, capped at capacity)
          │ Request  (consume 1)
          ▼
t=1.0s:   ● ● ●      (3/4)
```

**Параметры:**
- **Capacity (C)**: максимум токенов в bucket, обычно = `limit` на окно
- **Refill rate (R)**: tokens/sec, обычно = `limit / window_size`
- **Burst allowance**: up to `C` запросов подряд

**Формула пополнения:**

```
current_tokens = min(capacity, current_tokens + (now - last_refill) * refill_rate)
```

**Паттерн: Ограничение burst**

```
Без token bucket: 1000 requests в первую мсек (burst)
                  → DB crash

С token bucket (capacity=10):
Первые 10 requests: accepted
Requests 11+: rejected до пополнения токенов
```

**Пример.**

```python
import time
from dataclasses import dataclass
from threading import Lock

@dataclass
class TokenBucket:
    """
    Token bucket rate limiter.

    Args:
        capacity: max tokens (burst allowance)
        refill_rate: tokens per second

    Example:
        limiter = TokenBucket(capacity=100, refill_rate=10)
        # Allows bursts up to 100 requests
        # Sustained rate: 10 req/sec = 600 req/min
    """
    capacity: float
    refill_rate: float

    def __post_init__(self):
        self.tokens = self.capacity
        self.last_refill = time.time()
        self.lock = Lock()

    def allow_request(self, tokens_required=1) -> bool:
        """
        Check if request is allowed.
        Returns True if tokens available, False otherwise.
        """
        with self.lock:
            now = time.time()
            elapsed = now - self.last_refill

            # Refill tokens
            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.refill_rate
            )
            self.last_refill = now

            # Check if enough tokens
            if self.tokens >= tokens_required:
                self.tokens -= tokens_required
                return True
            return False

    def get_remaining(self) -> float:
        """Get current token count without consuming."""
        with self.lock:
            now = time.time()
            elapsed = now - self.last_refill

            tokens = min(
                self.capacity,
                self.tokens + elapsed * self.refill_rate
            )
            return tokens

    def reset(self):
        """Reset bucket to full capacity."""
        with self.lock:
            self.tokens = self.capacity
            self.last_refill = time.time()

# Usage example
if __name__ == "__main__":
    # 100 requests per minute = 1.67 req/sec
    # Burst allowance: 10 requests
    limiter = TokenBucket(capacity=10, refill_rate=100/60)

    print("Simulation: 20 consecutive requests")
    for i in range(20):
        allowed = limiter.allow_request()
        remaining = limiter.get_remaining()
        print(f"Request {i+1}: {'✓' if allowed else '✗'} (remaining: {remaining:.2f})")
        time.sleep(0.05)  # 50ms apart

    print("\n--- Output ---")
    # Requests 1-10: ✓ (consume all 10 tokens)
    # Requests 11-20: ✗✗✗✗ then ✓ (refill kicks in)
```

**Distributed version в Redis:**

```lua
-- rate_limit.lua: Token bucket in Redis (atomic)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local tokens_requested = tonumber(ARGV[4]) or 1

-- Get current bucket state
local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

-- Check if enough tokens
if tokens >= tokens_requested then
    tokens = tokens - tokens_requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)  -- TTL: 1 hour
    return {1, tokens}  -- [allowed, remaining]
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    return {0, tokens}  -- [denied, remaining]
end
```

```python
# Python wrapper for Lua script
import redis
import time

class DistributedTokenBucket:
    """Token bucket using Redis (distributed)."""

    def __init__(self, redis_client, capacity, refill_rate):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate

        # Load Lua script
        self.script = redis_client.register_script(open('rate_limit.lua').read())

    def allow_request(self, client_id, tokens_required=1) -> (bool, float):
        """
        Check rate limit for client.
        Returns (allowed, remaining_tokens)
        """
        key = f"rate_limit:{client_id}"
        now = time.time()

        result = self.script(
            keys=[key],
            args=[self.capacity, self.refill_rate, now, tokens_required]
        )

        allowed, remaining = result
        return bool(allowed), float(remaining)

# Usage
redis_client = redis.Redis(host='localhost', port=6379)
limiter = DistributedTokenBucket(
    redis_client=redis_client,
    capacity=100,
    refill_rate=10  # 10 tokens/sec
)

allowed, remaining = limiter.allow_request(client_id="user:123")
if allowed:
    print(f"Request allowed. {remaining:.2f} tokens remaining")
else:
    print(f"Rate limited. {remaining:.2f} tokens available")
```

**Типичные ошибки.**
- Забыть обновлять `last_refill` после пополнения — токены будут накапливаться бесконечно.
- Неправильно выбрать capacity — слишком большой вызывает burst, слишком малый блокирует легитимный трафик.
- Использовать float для времени без достаточной точности — может привести к неточностям при масштабировании.
- Не атомизировать операции в distributed системе — race condition между проверкой и обновлением.

**На интервью.**
- Объясни, как capacity и refill_rate влияют на поведение.
- Покажи Lua скрипт для атомизации в Redis.
- Уточняющий вопрос: «Как обрабатывать variable token costs?» — разные операции требуют разное количество токенов.

---

### 3. Как работает Leaky Bucket алгоритм?

**Зачем спрашивают.** Leaky Bucket — классический алгоритм для гарантии smooth output rate. Интервьюер проверяет отличие от Token Bucket и когда его применять.

**Короткий ответ.** Запросы попадают в очередь (bucket). Очередь обрабатывается с фиксированной скоростью (leak rate). Если очередь полна — новые запросы отклоняются. Гарантирует smooth output, но плохо с bursts.

**Детальный разбор.**

**Механизм:**

```
       Requests (variable rate)
            │
            ▼
      ┌──────────────┐
      │ ● ● ● ●      │ Queue (fixed size = 4)
      │              │
      │ Processing   │
      │ at fixed rate│
      │ (leak_rate = │
      │  2 req/sec)  │
      └──────┬───────┘
             │
             ▼
        Responses (smooth)

Пример: 3 быстрых запроса
t=0.0s:    ┌──────────────┐       → обработан
           │ ● ● ●        │
           └──────┬───────┘
t=0.0s:           ✓ (processed)
t=0.5s:    ┌──────────────┐       → обработан
           │ ● ●          │
           └──────┬───────┘
t=0.5s:           ✓ (processed)
t=1.0s:    ┌──────────────┐       → обработан
           │ ●            │
           └──────┬───────┘
t=1.0s:           ✓ (processed)
```

**Параметры:**
- **Queue capacity (C)**: макс элементов в очереди
- **Leak rate (L)**: элементов/сек для обработки

**Сравнение с Token Bucket:**

```
Token Bucket:
- Запрос обработан сразу если есть токены
- Burst = capacity
- Долгосроковая скорость = refill_rate
  → Гибкий, позволяет bursts

Leaky Bucket:
- Запрос обработан с фиксированной скоростью
- Burst = queue capacity, но тянется долго
- Долгосроковая скорость = leak_rate
  → Жёсткий, гладкий output
```

**Пример.**

```python
import time
from collections import deque
from threading import Lock
import heapq

class LeakyBucket:
    """
    Leaky bucket rate limiter.

    Requests queue up, processed at fixed leak_rate.

    Args:
        capacity: max queue size
        leak_rate: requests per second to process
    """

    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.time()
        self.lock = Lock()

    def allow_request(self) -> bool:
        """Check if request can be queued."""
        with self.lock:
            now = time.time()

            # Leak old requests
            self._leak(now)

            # Check if queue has space
            if len(self.queue) < self.capacity:
                self.queue.append(now)
                return True
            return False

    def _leak(self, now):
        """Remove processed requests from queue."""
        # Calculate how many to leak since last_leak
        elapsed = now - self.last_leak
        to_leak = int(elapsed * self.leak_rate)

        for _ in range(to_leak):
            if self.queue:
                self.queue.popleft()

        self.last_leak = now

    def queue_size(self) -> int:
        """Current queue length."""
        with self.lock:
            self._leak(time.time())
            return len(self.queue)

# Simulation
if __name__ == "__main__":
    bucket = LeakyBucket(capacity=5, leak_rate=2)  # 2 req/sec

    print("Scenario: 8 requests, queue capacity = 5")
    print("Leak rate = 2 req/sec\n")

    for i in range(8):
        allowed = bucket.allow_request()
        queue_len = bucket.queue_size()
        status = "✓ queued" if allowed else "✗ rejected (queue full)"
        print(f"Request {i+1}: {status} (queue: {queue_len}/5)")
        time.sleep(0.3)

    print("\nAfter waiting 2 seconds:")
    time.sleep(2)
    print(f"Queue length: {bucket.queue_size()} (leaked 4 requests)")
```

**Distributed version в Redis:**

```python
import redis
import time

class DistributedLeakyBucket:
    """Leaky bucket using Redis."""

    def __init__(self, redis_client, capacity, leak_rate):
        self.redis = redis_client
        self.capacity = capacity
        self.leak_rate = leak_rate

        # Lua script for atomic leaking + queueing
        self.script = redis_client.register_script("""
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local leak_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        -- Get queue length and last leak time
        local queue = redis.call('LLEN', key)
        local last_leak = tonumber(redis.call('HGET', key .. ':meta', 'last_leak') or now)

        -- Calculate leaks
        local elapsed = now - last_leak
        local to_leak = math.floor(elapsed * leak_rate)

        -- Remove processed items
        for i = 1, math.min(to_leak, queue) do
            redis.call('LPOP', key)
        end

        -- Update queue length
        queue = redis.call('LLEN', key)

        -- Try to add new request
        if queue < capacity then
            redis.call('RPUSH', key, now)
            redis.call('HSET', key .. ':meta', 'last_leak', now)
            redis.call('EXPIRE', key, 3600)
            return {1, queue + 1}  -- [allowed, new_queue_length]
        else
            redis.call('HSET', key .. ':meta', 'last_leak', now)
            return {0, queue}  -- [denied, queue_length]
        end
        """)

    def allow_request(self, client_id) -> (bool, int):
        """Returns (allowed, current_queue_length)."""
        key = f"bucket:{client_id}"
        now = time.time()

        result = self.script(
            keys=[key],
            args=[self.capacity, self.leak_rate, now]
        )

        allowed, queue_len = result
        return bool(allowed), int(queue_len)
```

**Когда использовать Leaky Bucket:**
1. **Smooth API responses** — важна предсказуемая скорость обработки
2. **Database load smoothing** — не хотим pikes в БД
3. **Payment processing** — никаких bursts, всё в очереди
4. **Message queues** — processed at consumer speed

**Типичные ошибки.**
- Неправильно рассчитать leak_rate — может быть слишком низко и запросы скапливаются.
- Забыть обновлять last_leak — приведёт к неправильному подсчёту.
- Смешивать semantics Token Bucket и Leaky Bucket — разные паттерны для разных случаев.

**На интервью.**
- Объясни отличие от Token Bucket: фиксированный output rate вместо burst.
- Упомяни случаи использования: платежи, smooth processing.
- Уточняющий вопрос: «Что если leak_rate >= arrival rate?» — очередь не растёт, обработка в реальном времени.

---

### 4. Как работает Fixed Window Counter?

**Зачем спрашивают.** Fixed Window Counter — самый простой алгоритм, часто первое решение. Интервьюер проверяет понимание его проблем (boundary spike) и когда его всё же можно использовать.

**Короткий ответ.** Делим время на фиксированные окна (например, по минутам). Считаем запросы в текущем окне. При переходе на новое окно — счётчик сбрасывается. Просто, но имеет проблему: два окна подряд позволяют 2x лимит на границе.

**Детальный разбор.**

**Проблема boundary spike:**

```
Window 1: 00:00-01:00      Window 2: 01:00-02:00
┌──────────────────────┐   ┌───────────────────────┐
│ 100 requests (full)  │   │ 100 requests (full)   │
│ Last 10 sec: /////// │   │ First 10 sec: /////// │
└──────────────────────┘   └───────────────────────┘
                  ▲
                  │
         Boundary spike:
         20 req in 20 sec = 2x limit!
```

**Механизм:**

```
Current time: 12:34:56

Window size = 60 sec
Current window = 12:34:00-12:35:00
Window index = int(12:34:56 / 60) = 754

Reset logic:
if window_index != prev_window_index:
    counter = 0  # Новое окно, сбросить счётчик

Проверка:
if counter < limit:
    counter += 1
    allowed = true
else:
    allowed = false
```

**Пример.**

```python
import time

class FixedWindowCounter:
    """
    Fixed window rate limiter.

    Simplest algorithm, but has boundary spike problem.

    Args:
        limit: max requests per window
        window_size: window duration in seconds

    Example:
        100 requests per minute:
        limiter = FixedWindowCounter(limit=100, window_size=60)
    """

    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.current_window = None
        self.counter = 0

    def allow_request(self) -> bool:
        """Check if request is allowed."""
        now = time.time()
        window_index = int(now // self.window_size)

        # Check if we're in a new window
        if window_index != self.current_window:
            self.current_window = window_index
            self.counter = 0

        # Check limit
        if self.counter < self.limit:
            self.counter += 1
            return True
        return False

    def get_remaining(self) -> int:
        """Requests remaining in current window."""
        return max(0, self.limit - self.counter)

    def get_reset_time(self) -> float:
        """Unix timestamp when window resets."""
        now = time.time()
        window_index = int(now // self.window_size)
        return (window_index + 1) * self.window_size

# Simulation showing boundary spike
if __name__ == "__main__":
    limiter = FixedWindowCounter(limit=10, window_size=5)

    print("Simulation: Fixed Window with boundary spike")
    print("Limit: 10 req/5sec\n")

    # Simulate requests near boundary
    test_times = [
        4.5,   # Window 0 (0-5s)
        4.7,
        4.9,
        5.0,   # BOUNDARY - Window 1 (5-10s) starts
        5.1,
        5.3,
    ]

    for test_time in test_times:
        # Mock time
        time_str = f"t={test_time:.1f}s"
        window = int(test_time // 5)

        # Simulate
        allowed = limiter.allow_request()
        status = "✓" if allowed else "✗"
        print(f"{time_str} (window {window}): {status} (count: {limiter.counter})")

    print("\nBoundary spike: 3 requests at 4.5-4.9s + 3 at 5.0-5.3s = 6 in 0.8s")
    print("Expected: 10 req/5s = 2 req/sec")
    print("Actual at boundary: 6 req/0.8s = 7.5 req/sec = 3.75x limit!")
```

**Distributed version в Redis:**

```python
import redis
import time

class DistributedFixedWindowCounter:
    """Fixed window counter using Redis."""

    def __init__(self, redis_client, limit, window_size):
        self.redis = redis_client
        self.limit = limit
        self.window_size = window_size

    def allow_request(self, client_id) -> bool:
        """Check rate limit."""
        now = time.time()
        window_index = int(now // self.window_size)

        key = f"window:{client_id}:{window_index}"

        # Atomic increment
        count = self.redis.incr(key)

        # Set expiration (cleanup old windows)
        self.redis.expire(key, int(self.window_size) + 1)

        return count <= self.limit

# Why it's fast but inaccurate
print("Pros:")
print("- O(1) operation")
print("- Minimal memory per client")
print("- Easy to implement")
print()
print("Cons:")
print("- Boundary spike: 2x limit possible")
print("- Not suitable for strict SLA")
print("- False rejects near boundary")
print()
print("Use case: Loose rate limiting, high-traffic APIs")
```

**Проблема визуально:**

```
Perfect scenario:
00:00 ──────────────────── 01:00 ──────────────────── 02:00
      │ 10 requests/min │        │ 10 requests/min │
      ✓ Balanced

Boundary spike scenario:
00:00 ──────────────────── 01:00 ──────────────────── 02:00
      │ 9 req      1 req│        │9 req      1 req │
      └─────── 2 requests in 2 seconds ────────────┘
      = 60 req/min equivalent at boundary!
```

**Типичные ошибки.**
- Использовать для critical SLA (payments, API quotas).
- Не знать про boundary spike problem.
- Неправильно рассчитать window_size — слишком большое окно скрывает spikes, слишком маленькое — много overhead.

**На интервью.**
- Объясни, как работает и в чём проблема.
- Нарисуй диаграмму boundary spike.
- Уточняющий вопрос: «Как исправить boundary spike?» → Sliding Window Log или Counter.

---

### 5. Как работает Sliding Window Log?

**Зачем спрашивают.** Sliding Window Log — точный алгоритм без boundary spike. Интервьюер проверяет понимание trade-off между accuracy и memory.

**Короткий ответ.** Храним timestamp каждого запроса в log. Для новой заявки подсчитываем запросы за последние N секунд. Если < лимит — добавляем timestamp. Очень точно, но требует memory O(rate * window_size) per client.

**Детальный разбор.**

**Механизм:**

```
Keep log of all request timestamps within window

t=0.1s:  [0.1]                          → 1 request
t=0.3s:  [0.1, 0.3]                    → 2 requests
t=0.5s:  [0.1, 0.3, 0.5]               → 3 requests
t=4.8s:  [0.1*, 0.3*, 0.5*, 4.8]
         (0.1, 0.3 outside window)      → 2 in-window
         [0.5, 4.8]                     → count valid requests

Window = 5 sec
Current time = 4.8s
Window start = 4.8 - 5 = -0.2s

Requests in window: [0.5, 4.8] = 2
(0.1 and 0.3 are before window start)
```

**Точность:**

```
Window: 5 seconds, Limit: 10 requests

Fixed Window:
- Boundary allows 20 requests in 2 seconds ❌

Sliding Window Log:
- Always accurate: max 10 in any 5 second window ✓
```

**Пример.**

```python
import time
from collections import deque
from threading import Lock

class SlidingWindowLog:
    """
    Sliding window log rate limiter.

    Exact algorithm, no boundary spike.
    Компромисс: uses O(rate * window) memory per client.

    Args:
        limit: max requests per window
        window_size: window duration in seconds
    """

    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.timestamps = deque()
        self.lock = Lock()

    def allow_request(self) -> bool:
        """Check if request is allowed."""
        with self.lock:
            now = time.time()
            window_start = now - self.window_size

            # Remove old timestamps outside window
            while self.timestamps and self.timestamps[0] < window_start:
                self.timestamps.popleft()

            # Check limit
            if len(self.timestamps) < self.limit:
                self.timestamps.append(now)
                return True
            return False

    def get_remaining(self) -> int:
        """Requests remaining in current window."""
        with self.lock:
            now = time.time()
            window_start = now - self.window_size

            # Count timestamps in window
            count = sum(1 for t in self.timestamps if t >= window_start)
            return max(0, self.limit - count)

    def get_oldest_request_time(self) -> float:
        """When will oldest request leave the window?"""
        with self.lock:
            if not self.timestamps:
                return time.time()

            now = time.time()
            window_start = now - self.window_size

            # Find oldest in-window timestamp
            for t in self.timestamps:
                if t >= window_start:
                    return t + self.window_size

# Simulation: no boundary spike
if __name__ == "__main__":
    limiter = SlidingWindowLog(limit=5, window_size=1)

    print("Simulation: Sliding Window Log")
    print("Limit: 5 req/1sec\n")

    # Mock requests across boundary
    requests_at = [
        0.7, 0.8, 0.9,  # Window 1
        1.0, 1.1,        # Boundary/Window 2
        1.2, 1.3,        # Window 2
    ]

    current_time = 0
    for req_time in requests_at:
        # Simulate time passing
        current_time = req_time

        # Manually check (in real code, we'd mock time)
        allowed = limiter.allow_request()
        remaining = limiter.get_remaining()

        status = "✓" if allowed else "✗"
        window_start = current_time - limiter.window_size
        in_window = sum(1 for t in limiter.timestamps if t >= window_start)

        print(f"t={req_time:.1f}s: {status} (in window: {in_window}/{limiter.limit})")

    print("\nNo spike: max requests in any 1s window = 5 (exact limit)")
```

**Distributed version в Redis (Sorted Set):**

```python
import redis
import time

class DistributedSlidingWindowLog:
    """Sliding window log using Redis Sorted Set."""

    def __init__(self, redis_client, limit, window_size):
        self.redis = redis_client
        self.limit = limit
        self.window_size = window_size

    def allow_request(self, client_id) -> bool:
        """
        Check rate limit using sorted set.
        Score = timestamp, Member = unique id.
        """
        key = f"log:{client_id}"
        now = time.time()
        window_start = now - self.window_size

        pipe = self.redis.pipeline()

        # Remove old timestamps
        pipe.zremrangebyscore(key, 0, window_start)

        # Add current timestamp
        pipe.zadd(key, {str(now): now})

        # Count in-window requests
        pipe.zcard(key)

        # Set expiration
        pipe.expire(key, int(self.window_size) + 1)

        results = pipe.execute()
        count = results[2]

        return count <= self.limit

    def get_remaining(self, client_id) -> int:
        """Requests remaining in current window."""
        key = f"log:{client_id}"
        now = time.time()
        window_start = now - self.window_size

        count = self.redis.zcount(key, window_start, now)
        return max(0, self.limit - count)

# Memory analysis
print("Memory usage: O(rate * window_size) per client")
print()
print("Example: 100 req/sec limit, 60 sec window")
print("Worst case: 100 * 60 = 6000 entries per client")
print("At 1 million clients: 6GB RAM (expensive!)")
print()
print("Use when:")
print("- Accuracy > memory")
print("- Small number of users/strict SLA")
print()
print("Don't use:")
print("- High-traffic APIs (1M+ requests/sec)")
print("- Tight memory constraints")
```

**Типичные ошибки.**
- Использовать для миллионов клиентов — memory explosion.
- Забыть удалять старые timestamps — утечка памяти.
- Не устанавливать TTL в Redis — ключи заполняют память.

**На интервью.**
- Объясни trade-off: perfect accuracy за memory cost.
- Упомяни Sorted Set implementation в Redis.
- Уточняющий вопрос: «Как оптимизировать память?» → Sliding Window Counter.

---

### 6. Как работает Sliding Window Counter?

**Зачем спрашивают.** Sliding Window Counter — best-of-both-worlds: точность без boundary spike, memory-efficient. Это favourite алгоритм в production. Интервьюер проверяет математику и способность реализовать.

**Короткий ответ.** Комбинируем Fixed Window для быстроты с весовым коэффициентом предыдущего окна для точности. Вычисляем: `weighted_count = prev_window_count * (1 - position) + current_window_count`. Если < лимит — разрешаем. Memory O(1), точнее чем Fixed, менее точно чем Log.

**Детальный разбор.**

**Механизм:**

```
Window size = 60 seconds

t = 34 sec (56.7% через окно)

┌────────────────────┐    ┌─────────────────────┐
│  Previous Window   │    │  Current Window     │
│  (00:00 - 01:00)   │    │  (01:00 - 02:00)    │
│  count = 80        │    │  count = 30         │
│  weight = 0.433    │    │  weight = 1.0       │
└────────────────────┘    └─────────────────────┘
                    ▲
                Boundary

Weighted count = 80 * (1 - 0.567) + 30
               = 80 * 0.433 + 30
               = 34.64 + 30
               = 64.64

Limit = 100
64.64 < 100 → Request allowed
```

**Математика:**

```
window_position = (current_time % window_size) / window_size
                = Percentage of current window elapsed

weighted_count = prev_count * (1 - window_position) + curr_count
               = Estimated requests in the rolling window
```

**Пример.**

```python
import time
from threading import Lock

class SlidingWindowCounter:
    """
    Sliding window counter rate limiter.

    Best balance: accuracy + memory efficiency.

    Args:
        limit: max requests per window
        window_size: window duration in seconds

    Memory: O(1) per client
    Accuracy: 99.9% (slight undercounting at boundaries)
    """

    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.prev_window_count = 0
        self.curr_window_count = 0
        self.curr_window_index = None
        self.lock = Lock()

    def allow_request(self) -> bool:
        """Check if request is allowed."""
        with self.lock:
            now = time.time()
            window_index = int(now // self.window_size)

            # Check if window changed
            if window_index != self.curr_window_index:
                # Move current to previous
                self.prev_window_count = self.curr_window_count
                self.curr_window_count = 0
                self.curr_window_index = window_index

            # Calculate weighted count
            window_position = (now % self.window_size) / self.window_size
            weighted_count = (
                self.prev_window_count * (1 - window_position) +
                self.curr_window_count
            )

            # Check limit
            if weighted_count < self.limit:
                self.curr_window_count += 1
                return True
            return False

    def get_remaining(self) -> float:
        """Estimated requests remaining."""
        with self.lock:
            now = time.time()
            window_index = int(now // self.window_size)

            # Synchronize window if changed
            prev_count = self.prev_window_count
            curr_count = self.curr_window_count
            curr_index = self.curr_window_index

            if window_index != curr_index:
                prev_count = curr_count
                curr_count = 0

            window_position = (now % self.window_size) / self.window_size
            weighted_count = prev_count * (1 - window_position) + curr_count

            return max(0, self.limit - weighted_count)

# Simulation: accurate, memory-efficient
if __name__ == "__main__":
    limiter = SlidingWindowCounter(limit=10, window_size=5)

    print("Simulation: Sliding Window Counter")
    print("Limit: 10 req/5sec (2 req/sec)\n")

    # Simulate 40 seconds with varying load
    test_scenarios = [
        (0.0, 3),    # t=0-1s: 3 requests (ok)
        (1.0, 2),    # t=1-2s: 2 requests (ok)
        (2.0, 4),    # t=2-3s: 4 requests (ok, total 9)
        (3.0, 2),    # t=3-4s: 2 requests (ok, total 11 but in sliding window)
        (4.0, 5),    # t=4-5s: near boundary
        (5.5, 3),    # t=5.5s: new window
    ]

    for time_offset, count in test_scenarios:
        allowed_in_burst = 0
        for _ in range(count):
            result = limiter.allow_request()
            if result:
                allowed_in_burst += 1

        remaining = limiter.get_remaining()
        print(f"t={time_offset:.1f}s: {allowed_in_burst}/{count} allowed (remaining: {remaining:.1f})")

# Distributed version in Redis
import redis

def distributed_sliding_window_counter(
    redis_client, client_id, limit, window_size
):
    """
    Sliding window counter in Redis.
    Uses two counters per client.
    """
    key_prefix = f"swc:{client_id}"

    now = time.time()
    window_index = int(now // window_size)
    curr_window_key = f"{key_prefix}:w{window_index}"
    prev_window_key = f"{key_prefix}:w{window_index-1}"

    pipe = redis_client.pipeline()

    # Increment current window
    pipe.incr(curr_window_key)
    pipe.expire(curr_window_key, int(window_size) + 1)

    # Get previous window count
    pipe.get(prev_window_key)

    results = pipe.execute()
    curr_count = results[0]
    prev_count = int(results[2] or 0)

    # Calculate weighted count
    window_position = (now % window_size) / window_size
    weighted_count = prev_count * (1 - window_position) + curr_count

    return weighted_count <= limit
```

**Сравнение всех алгоритмов:**

| Алгоритм | Accuracy | Memory | Boundary Spike | Complexity | Production Ready |
|----------|----------|--------|----------------|------------|------------------|
| Fixed Window | 60% | O(1) | Да (2x) | Очень простой | No |
| Leaky Bucket | 90% | O(queue) | Нет | Простой | Yes (smooth only) |
| Token Bucket | 95% | O(1) | Нет | Простой | Yes |
| Sliding Log | 99%+ | O(rate*window) | Нет | Средний | No (memory) |
| Sliding Counter | 99% | O(1) | Почти нет | Средний | Yes (best) |

**Типичные ошибки.**
- Неправильная формула взвешивания — легко ошибиться с (1 - position).
- Забыть обновлять prev_count при переходе на новое окно.
- Использовать целое число вместо float для window_position — потеря точности.
- Race condition в distributed версии — нужны lua скрипты.

**На интервью.**
- Выведи формулу на доске с диаграммой.
- Объясни, почему это лучше Fixed Window.
- Уточняющий вопрос: «Как реализовать в Redis?» → Lua скрипт для атомизации.

---

### 7. Как реализовать distributed rate limiting?

**Зачем спрашивают.** Это ключевое отличие между single-machine и production системами. Интервьюер проверяет понимание проблем (race conditions, consistency) и решений.

**Короткий ответ.** Нужна shared state (обычно Redis). Основные подходы: (1) Centralized Redis — все servers обращаются к single Redis; (2) Redis Cluster — sharded по client_id; (3) Eventual consistency — local state с периодической синхронизацией.

**Детальный разбор.**

**Проблема без distributed rate limiting:**

```
100 req/min limit

Server 1              Server 2
┌─────────────┐       ┌─────────────┐
│ count = 50  │       │ count = 50  │
└─────────────┘       └─────────────┘
       │                    │
       ▼                    ▼
User sends 51 requests to Server 1 → ALLOWED (50+1)
User sends 51 requests to Server 2 → ALLOWED (50+1)

Total: 102 requests allowed when limit = 100! ✗
```

**Решение 1: Centralized Redis**

```
┌─────────┐     ┌──────────┐     ┌─────────┐
│Server 1 │─┐   │ Server 2 │─┐   │Server 3 │─┐
└─────────┘ │   └──────────┘ │   └─────────┘ │
            │                │               │
            └────────┬───────┴───────┬───────┘
                     │               │
                ┌────▼───────────────▼────┐
                │  Redis                  │
                │  rate_limit:user123: 75 │
                └─────────────────────────┘

Advantages:
✓ Single source of truth
✓ Exact rate limiting
✓ Simple to implement

Disadvantages:
✗ Redis becomes SPOF (single point of failure)
✗ Network latency (1-5ms per request)
✗ Redis throughput limit (100K-300K ops/sec)
```

**Решение 2: Redis Cluster (sharded)**

```
client_id determines shard:
shard = hash(client_id) % num_nodes

Client A → hash = 0   → Redis Node 1
Client B → hash = 1   → Redis Node 2
Client C → hash = 1   → Redis Node 2  (same node as B)

Advantages:
✓ Horizontal scalability
✓ No single point of failure
✓ Higher throughput

Disadvantages:
✗ More complex setup
✗ Cross-node failures impact clients on that node
✗ Rebalancing complexity
```

**Решение 3: Sticky Sessions (local state)**

```
Load Balancer (hash by IP/user)

Client A ──┐
Client B ──┼──► Server 1 (count = 50)
           │
Client C ──┼──► Server 2 (count = 60)
Client D ──┘

Advantages:
✓ No network latency
✓ Simplest to implement
✓ Highest throughput

Disadvantages:
✗ Weak consistency (user can bypass by changing IP)
✗ Uneven distribution
✗ Server failure = lost counters
✗ Not suitable for strict SLA
```

**Решение 4: Eventual Consistency (async sync)**

```
Local counter + async sync to Redis

┌─────────────┐
│ Server 1    │
│ local: 45   │
│ redis: 45   │
└─────────────┘
       │
       ├─ Every 10 sec: sync to Redis
       │
┌──────▼───────────────────┐
│ Redis (global state)     │
│ rate_limit:user123: 120  │
│ (sum from all servers)   │
└──────────────────────────┘

Advantages:
✓ Low latency (no network per request)
✓ Scalable

Disadvantages:
✗ Eventual consistency (may overcount)
✗ Complex to implement correctly
```

**Пример.**

```python
import redis
import time
from typing import Tuple

class DistributedRateLimiter:
    """
    Distributed rate limiter using Redis.

    Supports multiple strategies:
    - Sliding window counter (recommended)
    - Token bucket
    - Leaky bucket
    """

    def __init__(self, redis_client, strategy='sliding_window'):
        self.redis = redis_client
        self.strategy = strategy

        # Load Lua scripts for atomic operations
        self.sliding_window_script = self._load_lua('sliding_window.lua')
        self.token_bucket_script = self._load_lua('token_bucket.lua')

    def _load_lua(self, filename):
        """Load and register Lua script."""
        with open(filename) as f:
            script_code = f.read()
        return self.redis.register_script(script_code)

    def allow_request(
        self, client_id: str, limit: int, window: int
    ) -> Tuple[bool, dict]:
        """
        Check rate limit.

        Returns:
            (allowed: bool, info: dict with remaining, reset_time, etc)
        """
        if self.strategy == 'sliding_window':
            return self._sliding_window(client_id, limit, window)
        elif self.strategy == 'token_bucket':
            return self._token_bucket(client_id, limit, window)
        else:
            raise ValueError(f"Unknown strategy: {self.strategy}")

    def _sliding_window(
        self, client_id: str, limit: int, window: int
    ) -> Tuple[bool, dict]:
        """Sliding window counter implementation."""
        key = f"rate_limit:{client_id}"
        now = time.time()
        window_start = now - window

        pipe = self.redis.pipeline()

        # Remove old timestamps
        pipe.zremrangebyscore(key, 0, window_start)

        # Count current + add new
        pipe.zcard(key)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)

        # Set expiration
        pipe.expire(key, window + 1)

        results = pipe.execute()
        before_count = results[1]
        after_count = results[3]

        allowed = after_count <= limit

        reset_time = now + window

        return allowed, {
            'limit': limit,
            'remaining': max(0, limit - after_count),
            'reset_in': int(reset_time - now),
            'reset_timestamp': int(reset_time),
        }

    def _token_bucket(
        self, client_id: str, limit: int, window: int
    ) -> Tuple[bool, dict]:
        """Token bucket implementation."""
        capacity = limit
        refill_rate = limit / window  # tokens per second

        key = f"bucket:{client_id}"
        now = time.time()

        # Lua script handles atomic refill + check
        result = self.token_bucket_script(
            keys=[key],
            args=[capacity, refill_rate, now]
        )

        allowed, remaining = result

        reset_time = now + window

        return bool(allowed), {
            'limit': limit,
            'remaining': float(remaining),
            'reset_in': int(reset_time - now),
            'reset_timestamp': int(reset_time),
        }

# Usage example
if __name__ == "__main__":
    redis_client = redis.Redis(host='localhost', port=6379)

    limiter = DistributedRateLimiter(
        redis_client=redis_client,
        strategy='sliding_window'
    )

    # Simulate multiple requests from same client
    client_id = "user:123"
    limit = 10
    window = 60

    print("Testing distributed rate limiter\n")

    for i in range(15):
        allowed, info = limiter.allow_request(client_id, limit, window)

        status = "✓ ALLOW" if allowed else "✗ BLOCK"
        print(f"Request {i+1:2d}: {status} | "
              f"Remaining: {info['remaining']} | "
              f"Reset in: {info['reset_in']}s")

        time.sleep(0.1)

# ============= Lua Scripts =============

# file: sliding_window.lua
lua_sliding_window = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local window_start = now - window

-- Remove old entries
redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

-- Count current
local count = redis.call('ZCARD', key)

-- Add new request
redis.call('ZADD', key, now, tostring(now))

-- Set TTL
redis.call('EXPIRE', key, window + 1)

-- Check limit
if count < limit then
    return {1, count + 1}  -- allowed, new_count
else
    return {0, count}      -- blocked, count
end
"""

# file: token_bucket.lua
lua_token_bucket = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Get bucket state
local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

-- Check limit
if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return {1, tokens}
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    return {0, tokens}
end
"""
```

**High availability setup:**

```
┌─────────────────────────────────────────────┐
│  Redis Cluster (3+ nodes)                   │
│  - Automatic failover                       │
│  - Data replication                         │
│  - Sharding by client_id                    │
└──┬──────────────────────────────────────────┘
   │
┌──┴──────────────────────────────────────────┐
│  API Gateway Load Balancer                  │
│  (Kong, AWS ALB, etc)                       │
├──┬──────┬──────┬────────────────────────────┤
│  │      │      │                            │
│  ▼      ▼      ▼                            ▼
│ Svc1   Svc2   Svc3            ... (N servers)
│
│ All query same Redis Cluster
│ Consistent rate limiting
└──────────────────────────────────────────────
```

**Типичные ошибки.**
- Использовать synchronous Redis calls без batching — latency spike.
- Забыть про Redis connection pooling — истощение connections.
- Не обрабатывать Redis failures — cascading failures.
- Неправильная sharding strategy — hot keys.

**На интервью.**
- Объясни trade-offs между сentralized, sharded, local.
- Упомяни Lua scripts для атомизации.
- Уточняющий вопрос: «Что если Redis недоступен?» → Graceful degradation (fail open или local fallback).

---

### 8. Как использовать Redis для rate limiting?

**Зачем спрашивают.** Redis — de-facto standard для distributed rate limiting. Интервьюер проверяет practical knowledge: data structures, Lua scripts, optimization.

**Короткий ответ.** Redis предоставляет быстрый key-value store с atomic operations. Для rate limiting используем: (1) Sorted Set (Zset) для sliding window log; (2) Hash для token bucket state; (3) String counters для fixed window. Lua scripts обеспечивают atomicity.

**Детальный разбор.**

**Redis data structures для rate limiting:**

```
1. String (INCR) — простой счётчик
   Key: "window:user123:hour5"
   Value: 45 (requests in this hour)
   Operation: INCR (O(1))

2. Sorted Set (ZADD, ZCARD, ZREMRANGEBYSCORE) — sliding window
   Key: "log:user123"
   Members: {timestamp1, timestamp2, ...}
   Scores: {timestamp1, timestamp2, ...} (same for sorting)
   Operation: Add score, remove old, count in range

3. Hash (HMGET, HMSET) — token bucket state
   Key: "bucket:user123"
   Fields: {tokens: 45.3, last_refill: 1234567890}
   Operation: Update both atomically
```

**Performance characteristics:**

```
Operation               Time Complexity   Notes
────────────────────────────────────────────────
INCR                   O(1)              Fastest
ZADD                   O(log N)          N = entries in window
ZCARD                  O(1)              Fast count
ZREMRANGEBYSCORE       O(log N + K)      K = removed count
ZCOUNT                 O(log N)          Count in range
HMGET/HMSET            O(N)              N = fields
```

**Лучшая практикаs:**

```
1. Use Lua scripts for atomic multi-step operations
   ✓ Prevent race conditions
   ✓ Reduce network roundtrips

2. Set appropriate TTL (EXPIRE)
   ✓ Cleanup old data automatically
   ✓ Prevent memory bloat

3. Choose right data structure
   ✓ Zset for accuracy (sliding window)
   ✓ String for speed (fixed window)
   ✓ Hash for state (token bucket)

4. Implement connection pooling
   ✓ Reuse connections
   ✓ Better throughput

5. Handle Redis unavailability
   ✓ Graceful degradation
   ✓ Local fallback counter
```

**Пример.**

```python
import redis
import time
from redis.lock import Lock
from typing import Optional

class RedisRateLimiter:
    """
    Production-ready Redis rate limiter.

    Features:
    - Multiple algorithms (sliding window, token bucket, fixed window)
    - Atomic Lua operations
    - Automatic cleanup with TTL
    - Connection pooling
    - Error handling & graceful degradation
    """

    def __init__(
        self,
        redis_host='localhost',
        redis_port=6379,
        redis_db=0,
        default_limit=100,
        default_window=60,
        fallback_mode=True
    ):
        self.redis = redis.Redis(
            host=redis_host,
            port=redis_port,
            db=redis_db,
            decode_responses=True,
            socket_connect_timeout=2,
            socket_keepalive=True,
            socket_keepalive_options={
                1: (1, 3),  # TCP_KEEPIDLE, TCP_KEEPINTVL
            },
        )

        # Connection pool
        self.redis.connection_pool.connection_class.socket_connect_timeout = 2

        self.default_limit = default_limit
        self.default_window = default_window
        self.fallback_mode = fallback_mode
        self.fallback_counters = {}  # In-memory fallback

        # Register Lua scripts
        self.sliding_window_script = self._register_script('sliding_window.lua')
        self.token_bucket_script = self._register_script('token_bucket.lua')

    def _register_script(self, script_name):
        """Register Lua script with Redis."""
        script_content = self._get_lua_script(script_name)
        return self.redis.register_script(script_content)

    def _get_lua_script(self, name):
        """Get Lua script content."""
        scripts = {
            'sliding_window.lua': """
                local key = KEYS[1]
                local limit = tonumber(ARGV[1])
                local window = tonumber(ARGV[2])
                local now = tonumber(ARGV[3])

                local window_start = now - window
                redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

                local count = redis.call('ZCARD', key)

                if count < limit then
                    redis.call('ZADD', key, now, tostring(now))
                    redis.call('EXPIRE', key, window + 1)
                    return {1, limit - count - 1}
                else
                    return {0, 0}
                end
            """,
            'token_bucket.lua': """
                local key = KEYS[1]
                local capacity = tonumber(ARGV[1])
                local refill_rate = tonumber(ARGV[2])
                local now = tonumber(ARGV[3])

                local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
                local tokens = tonumber(bucket[1]) or capacity
                local last_refill = tonumber(bucket[2]) or now

                local elapsed = now - last_refill
                tokens = math.min(capacity, tokens + elapsed * refill_rate)

                if tokens >= 1 then
                    tokens = tokens - 1
                    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
                    redis.call('EXPIRE', key, 3600)
                    return {1, tokens}
                else
                    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
                    return {0, tokens}
                end
            """
        }
        return scripts.get(name, "")

    def allow_request(
        self,
        client_id: str,
        limit: Optional[int] = None,
        window: Optional[int] = None,
        algorithm: str = 'sliding_window'
    ) -> dict:
        """
        Check if request is allowed.

        Returns:
            {
                'allowed': bool,
                'remaining': int,
                'reset_in': int (seconds),
                'limit': int,
                'window': int,
                'degraded': bool (if using fallback)
            }
        """
        limit = limit or self.default_limit
        window = window or self.default_window

        try:
            if algorithm == 'sliding_window':
                return self._sliding_window(client_id, limit, window)
            elif algorithm == 'token_bucket':
                return self._token_bucket(client_id, limit, window)
            elif algorithm == 'fixed_window':
                return self._fixed_window(client_id, limit, window)
        except Exception as e:
            print(f"Redis error: {e}")
            if self.fallback_mode:
                return self._fallback_allow(client_id, limit, window)
            else:
                # Fail open: allow request if Redis is down
                return {
                    'allowed': True,
                    'remaining': limit,
                    'reset_in': window,
                    'limit': limit,
                    'window': window,
                    'degraded': True
                }

    def _sliding_window(self, client_id, limit, window):
        """Sliding window counter."""
        key = f"rl:sw:{client_id}"
        now = time.time()

        allowed, remaining = self.sliding_window_script(
            keys=[key],
            args=[limit, window, now]
        )

        return {
            'allowed': bool(allowed),
            'remaining': int(remaining),
            'reset_in': window,
            'limit': limit,
            'window': window,
            'degraded': False
        }

    def _token_bucket(self, client_id, limit, window):
        """Token bucket."""
        key = f"rl:tb:{client_id}"
        now = time.time()
        refill_rate = limit / window

        allowed, remaining = self.token_bucket_script(
            keys=[key],
            args=[limit, refill_rate, now]
        )

        return {
            'allowed': bool(allowed),
            'remaining': float(remaining),
            'reset_in': window,
            'limit': limit,
            'window': window,
            'degraded': False
        }

    def _fixed_window(self, client_id, limit, window):
        """Fixed window counter."""
        now = int(time.time())
        window_key = now // window
        key = f"rl:fw:{client_id}:{window_key}"

        count = self.redis.incr(key)
        self.redis.expire(key, window + 1)

        allowed = count <= limit

        return {
            'allowed': allowed,
            'remaining': max(0, limit - count),
            'reset_in': window - (now % window),
            'limit': limit,
            'window': window,
            'degraded': False
        }

    def _fallback_allow(self, client_id, limit, window):
        """Fallback: in-memory counter when Redis is unavailable."""
        now = int(time.time())
        window_key = f"{client_id}:{now // window}"

        self.fallback_counters[window_key] = self.fallback_counters.get(window_key, 0) + 1
        count = self.fallback_counters[window_key]

        # Cleanup old windows
        if len(self.fallback_counters) > 10000:
            old_keys = [k for k in self.fallback_counters.keys()
                       if k < window_key]
            for k in old_keys[:1000]:
                del self.fallback_counters[k]

        allowed = count <= limit

        return {
            'allowed': allowed,
            'remaining': max(0, limit - count),
            'reset_in': window - (now % window),
            'limit': limit,
            'window': window,
            'degraded': True
        }

    def reset(self, client_id: str):
        """Reset rate limit for client."""
        keys = self.redis.keys(f"rl:*:{client_id}*")
        if keys:
            self.redis.delete(*keys)

# Usage
if __name__ == "__main__":
    limiter = RedisRateLimiter(
        default_limit=10,
        default_window=60,
        fallback_mode=True
    )

    print("Testing Redis rate limiter\n")

    for i in range(15):
        result = limiter.allow_request("user:123", algorithm='sliding_window')

        status = "✓ ALLOW" if result['allowed'] else "✗ BLOCK"
        degraded = " [DEGRADED]" if result['degraded'] else ""

        print(f"Request {i+1:2d}: {status} | "
              f"Remaining: {result['remaining']:>3} | "
              f"Reset in: {result['reset_in']}s{degraded}")

        time.sleep(0.1)
```

**Optimization tips:**

```python
# 1. Batch operations with pipeline
pipe = limiter.redis.pipeline()
for client_id in client_ids:
    pipe.zcard(f"log:{client_id}")
results = pipe.execute()

# 2. Use connection pooling
redis_client = redis.Redis(
    connection_pool=redis.ConnectionPool(
        max_connections=50
    )
)

# 3. Implement caching for frequently accessed limits
from functools import lru_cache

@lru_cache(maxsize=10000)
def get_limit_config(client_id):
    return redis_client.hget(f"config:{client_id}", 'limit')

# 4. Monitor Redis performance
info = redis_client.info('stats')
print(f"Commands/sec: {info['instantaneous_ops_per_sec']}")
print(f"Memory usage: {info['used_memory_human']}")
```

**Типичные ошибки.**
- Забыть про TTL — memory leak.
- Использовать string concatenation вместо formatting — injection risk.
- Не обрабатывать redis.ConnectionError — cascade failures.
- Неправильная sharding — hot keys in cluster.

**На интервью.**
- Объясни, почему Lua scripts нужны.
- Покажи разные data structures для разных алгоритмов.
- Уточняющий вопрос: «Как масштабировать Redis?» → Clustering, replication.

---

### 9. Какие HTTP заголовки использовать для rate limiting?

**Зачем спрашивают.** HTTP headers — стандартный способ сообщить клиенту о rate limiting status. Интервьюер проверяет знание RFC 6585, RateLimit-Info draft и лучшая практикаs.

**Короткий ответ.** Основные headers: `X-RateLimit-Limit` (лимит), `X-RateLimit-Remaining` (осталось), `X-RateLimit-Reset` (timestamp сброса). При превышении: `429 Too Many Requests` с `Retry-After` header. Поддержка этих headers позволяет клиентам делать smart decisions.

**Детальный разбор.**

**RFC 6585 и RateLimit-Info draft:**

```
RFC 6585 — standardized 429 status code
RateLimit-Info draft — standardized headers

Существует много вариантов:
1. GitHub style: X-RateLimit-*
2. Stripe style: RateLimit-* (без X-)
3. IETF draft: RateLimit-* с дополнительными полями

Рекомендация: следовать IETF draft или GitHub (де-факто стандарт)
```

**Header set для successful request:**

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1623456789

Значения:
- X-RateLimit-Limit: максимум запросов в окне
- X-RateLimit-Remaining: сколько запросов осталось
- X-RateLimit-Reset: unix timestamp, когда сбросится счётчик
```

**Header set для rate-limited response:**

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1623456789
Retry-After: 30

Content-Type: application/json

{
    "error": "rate_limit_exceeded",
    "message": "API rate limit exceeded",
    "retry_after": 30
}

Значения:
- Retry-After: либо seconds (int), либо HTTP-date
  можно быть в разных форматах, клиент должен поддерживать оба
```

**Variant: RateLimit-Info header (draft-polli-ratelimit-headers):**

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 87
RateLimit-Reset: 1623456789

Или более современный формат:
RateLimit-Limit: 100, 100;window=60
RateLimit-Remaining: 87
RateLimit-Reset: 1623456789

Поддержка:
- Более чистый формат
- Не использует X- prefix (который deprecated)
- Но ещё не полностью стандартизирован
```

**Пример.**

```python
from flask import Flask, request, jsonify
from functools import wraps
import time

app = Flask(__name__)

def rate_limit_headers(f):
    """Decorator to add rate limit headers to response."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        client_id = request.remote_addr

        # Check rate limit (use actual limiter here)
        allowed, info = check_rate_limit(client_id)

        # Generate response
        response = make_response(f(*args, **kwargs))

        # Add rate limit headers (always, even on success)
        response.headers['X-RateLimit-Limit'] = str(info['limit'])
        response.headers['X-RateLimit-Remaining'] = str(info['remaining'])
        response.headers['X-RateLimit-Reset'] = str(info['reset_timestamp'])

        # If limited, also add Retry-After
        if not allowed:
            response.status_code = 429

            # Retry-After can be int (seconds) or HTTP-date
            retry_after_seconds = info['reset_in']
            response.headers['Retry-After'] = str(retry_after_seconds)

            # Add JSON error body
            response.data = jsonify({
                'error': 'rate_limit_exceeded',
                'message': f'API rate limit exceeded. Try again in {retry_after_seconds}s',
                'retry_after': retry_after_seconds
            }).data

        return response

    return decorated_function

@app.route('/api/users/<int:user_id>')
@rate_limit_headers
def get_user(user_id):
    return jsonify({'id': user_id, 'name': 'John Doe'})

# ============ More complete example ============

from flask import make_response
import json

class RateLimitMiddleware:
    """
    Rate limiting middleware with proper headers.

    Supports:
    - Different limits per endpoint
    - Per-user/IP rate limiting
    - Proper HTTP headers
    - Graceful degradation
    """

    def __init__(self, app, limiter_config):
        self.app = app
        self.config = limiter_config  # Maps endpoint → limit config
        self.limiter = RedisRateLimiter()  # Actual limiter implementation

        app.before_request(self.check_rate_limit)
        app.after_request(self.add_rate_limit_headers)

    def check_rate_limit(self):
        """Check rate limit before processing request."""
        endpoint = request.endpoint
        client_id = self._get_client_id(request)

        # Get config for this endpoint
        config = self.config.get(endpoint, {})
        limit = config.get('limit', 100)
        window = config.get('window', 60)

        # Check rate limit
        allowed, info = self.limiter.allow_request(
            client_id=client_id,
            limit=limit,
            window=window
        )

        # Store in request context for response
        request.rate_limit_info = info
        request.rate_limit_allowed = allowed

        if not allowed:
            return self._make_rate_limit_response(info)

    def add_rate_limit_headers(self, response):
        """Add rate limit headers to every response."""
        if not hasattr(request, 'rate_limit_info'):
            return response

        info = request.rate_limit_info

        # Standard headers
        response.headers['X-RateLimit-Limit'] = str(info['limit'])
        response.headers['X-RateLimit-Remaining'] = str(info['remaining'])
        response.headers['X-RateLimit-Reset'] = str(info['reset_timestamp'])

        # Additional headers for better UX
        response.headers['X-RateLimit-Window'] = str(info['window'])

        if info.get('degraded'):
            response.headers['X-RateLimit-Status'] = 'degraded'

        if response.status_code == 429:
            response.headers['Retry-After'] = str(info['reset_in'])

        return response

    def _get_client_id(self, request):
        """Determine client identifier."""
        # Prefer explicit API key
        if 'X-API-Key' in request.headers:
            return f"key:{request.headers['X-API-Key']}"

        # Then user ID from auth
        if hasattr(request, 'user_id'):
            return f"user:{request.user_id}"

        # Fall back to IP
        return f"ip:{request.remote_addr}"

    def _make_rate_limit_response(self, info):
        """Create 429 response."""
        response = {
            'error': 'rate_limit_exceeded',
            'message': 'API rate limit exceeded',
            'retry_after': info['reset_in'],
            'limit': info['limit'],
            'window_seconds': info['window'],
            'reset_timestamp': info['reset_timestamp']
        }

        return jsonify(response), 429, {
            'X-RateLimit-Limit': str(info['limit']),
            'X-RateLimit-Remaining': '0',
            'X-RateLimit-Reset': str(info['reset_timestamp']),
            'Retry-After': str(info['reset_in']),
        }

# ============ Client-side handling ============

import requests
import time

class APIClientWithRateLimit:
    """Client that respects rate limit headers."""

    def __init__(self, base_url, max_retries=3):
        self.base_url = base_url
        self.max_retries = max_retries
        self.session = requests.Session()

    def request(self, method, path, **kwargs):
        """Make request with automatic retry on rate limit."""
        url = f"{self.base_url}{path}"

        for attempt in range(self.max_retries + 1):
            response = self.session.request(method, url, **kwargs)

            # Log rate limit info
            if 'X-RateLimit-Limit' in response.headers:
                limit = response.headers['X-RateLimit-Limit']
                remaining = response.headers['X-RateLimit-Remaining']
                reset = response.headers['X-RateLimit-Reset']

                print(f"Rate limit: {remaining}/{limit} (reset: {reset})")

            # Handle rate limit
            if response.status_code == 429:
                if attempt < self.max_retries:
                    retry_after = int(response.headers.get('Retry-After', 60))
                    print(f"Rate limited. Retrying in {retry_after}s...")
                    time.sleep(retry_after)
                    continue
                else:
                    raise Exception("Rate limited after max retries")

            return response

        return response

    def get(self, path, **kwargs):
        return self.request('GET', path, **kwargs)

    def post(self, path, **kwargs):
        return self.request('POST', path, **kwargs)

# Usage
client = APIClientWithRateLimit('https://api.example.com')
response = client.get('/api/users/123')
print(response.json())
```

**Header variations by popular APIs:**

```
GitHub API:
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 59
X-RateLimit-Reset: 1377013266

Stripe API:
X-Stripe-Should-Retry: false
Retry-After: 2

AWS API (v4):
Не использует стандартные headers, вместо этого:
x-amzn-ErrorType: ThrottlingException

Google APIs:
Не использует в response, вместо этого quotas в API настройках

Recommendation: Follow GitHub style (де-факто стандарт)
или IETF draft (будущее)
```

**Лучшая практикаs:**

```python
# 1. Всегда отправляй rate limit headers, даже если не лимитирован
response.headers['X-RateLimit-Limit'] = '100'
response.headers['X-RateLimit-Remaining'] = '42'
response.headers['X-RateLimit-Reset'] = '1623456789'

# 2. Используй unix timestamp для Reset, не relative time
reset_time = int(time.time()) + window_seconds
response.headers['X-RateLimit-Reset'] = str(reset_time)

# 3. Включай информацию в JSON body тоже
{
    "error": "rate_limit_exceeded",
    "rate_limit": {
        "limit": 100,
        "remaining": 0,
        "reset": 1623456789
    }
}

# 4. Обработай разные значения Retry-After
# Может быть: int (секунды) или RFC7231 date
if retry_after.isdigit():
    wait_seconds = int(retry_after)
else:
    # Parse RFC7231 date
    retry_time = email.utils.parsedate_to_datetime(retry_after)
    wait_seconds = int((retry_time - datetime.now()).total_seconds())

# 5. Логируй rate limit events для мониторинга
logger.warning(f"Client {client_id} rate limited", extra={
    'client_id': client_id,
    'limit': limit,
    'remaining': remaining,
    'reset_in': reset_in
})
```

**Типичные ошибки.**
- Отправлять headers только на 429 response — клиент не знает, близко ли к лимиту.
- Использовать relative time вместо unix timestamp — сложнее для клиента.
- Не включать Retry-After на 429 — клиент не знает, когда повторить.
- Inconsistent header names — микросервисы используют разные.

**На интервью.**
- Объясни, какие headers отправлять и в каких случаях.
- Покажи, как клиент может использовать headers для backoff.
- Уточняющий вопрос: «Почему unix timestamp лучше, чем relative time?» → Не зависит от clock skew.

---

### 10. Как масштабировать rate limiter?

**Зачем спрашивают.** Масштабирование от одного сервера к миллионам запросов/сек — ключевой skill. Интервьюер проверяет понимание bottlenecks и solutions на каждом уровне.

**Короткий ответ.** Масштабирование имеет несколько уровней: (1) Redis single → Redis Cluster; (2) Latency optimization (Lua scripts, pipelining); (3) Architecture scaling (sharding, caching, local counters); (4) Load distribution (sticky sessions, hash-based routing). На каждом уровне different trade-offs.

**Детальный разбор.**

**Bottlenecks на разных масштабах:**

```
Scale 1: ~1K req/sec
┌─────────────┐
│  Single     │
│  Redis      │
│  instance   │
└────┬────────┘
     │
  Bottleneck: Redis throughput (~100K ops/sec)
  → Single Redis can handle 100x load

Scale 2: ~10K req/sec
  Bottleneck: Network latency (1-5ms)
  → Need to reduce roundtrips

Scale 3: ~100K req/sec
  Bottleneck: Redis memory, bandwidth
  → Need clustering/sharding

Scale 4: ~1M req/sec
  Bottleneck: Everything
  → Need distributed architecture
  with caching, local state, eventual consistency
```

**Level 1: Single Redis Optimization**

```
Current: 1 request → 1 Redis call

Optimizations:
1. Lua scripts (atomic, no roundtrips)
2. Connection pooling (reuse connections)
3. Pipelining (batch multiple requests)
4. Async Redis client (non-blocking I/O)

Result: 10-50x throughput improvement
```

```python
# Bad: Multiple roundtrips
def check_limit_slow(client_id):
    count = redis.incr(f"window:{client_id}")  # Roundtrip 1
    window = redis.get(f"window_key:{client_id}")  # Roundtrip 2
    ttl = redis.ttl(f"window:{client_id}")  # Roundtrip 3
    return count <= limit

# Good: Lua script (single roundtrip)
def check_limit_fast(client_id):
    result = lua_script(
        keys=[f"window:{client_id}"],
        args=[limit, window_size, now]
    )  # Single roundtrip to Redis
    return result[0] == 1

# Better: Pipelining
def check_limits_batch(client_ids):
    pipe = redis.pipeline()
    for client_id in client_ids:
        pipe.incr(f"window:{client_id}")
    results = pipe.execute()  # Single roundtrip for all
    return results
```

**Level 2: Redis Cluster**

```
Before (single point of failure):
┌──────────────────────────────────┐
│  Redis Master                    │
│  rl:user123, rl:user456, ...     │
│  1000 ops/sec → BOTTLENECK       │
└──────────────────────────────────┘

After (sharded cluster):
┌──────────────┬──────────────┬──────────────┐
│ Redis Node 1 │ Redis Node 2 │ Redis Node 3 │
│ rl:user123   │ rl:user456   │ rl:user789   │
│ rl:user012   │ rl:user345   │ rl:user678   │
│ ~300 ops/sec │ ~300 ops/sec │ ~300 ops/sec │
└──────────────┴──────────────┴──────────────┘
Total: ~900 ops/sec (3x scaling)

Shard by: hash(client_id) % num_nodes
```

```python
import redis.sentinel
from consistent_hash import ConsistentHash

class ShardedRateLimiter:
    """Rate limiter with Redis cluster (automatic sharding)."""

    def __init__(self, nodes):
        # Create connection to each node
        self.nodes = {}
        for i, (host, port) in enumerate(nodes):
            self.nodes[i] = redis.Redis(host=host, port=port)

        self.num_nodes = len(nodes)

    def _get_node(self, client_id):
        """Determine which node for this client."""
        shard = hash(client_id) % self.num_nodes
        return self.nodes[shard]

    def allow_request(self, client_id, limit, window):
        """Check rate limit on appropriate shard."""
        node = self._get_node(client_id)

        result = node.eval(
            LUA_SCRIPT,
            1,  # num_keys
            f"rl:{client_id}",  # KEYS[1]
            limit, window, time.time()  # ARGV
        )

        return bool(result[0])

# Automatic failover with Sentinel
def get_redis_sentinel_client():
    sentinel = redis.sentinel.Sentinel([
        ('sentinel1', 26379),
        ('sentinel2', 26379),
        ('sentinel3', 26379),
    ])
    return sentinel.master_for('mymaster')
```

**Level 3: Architecture Scaling**

```
Problem: Even with sharding, Redis is bottleneck

Solution: Multi-level caching + eventual consistency

┌─────────────┐
│ API Server  │
│ (local)     │
│ count: 50   │
└──────┬──────┘
       │ Every 10s sync
┌──────▼──────────────────────┐
│ Redis (global state)        │
│ rl:user123:synced: 500      │
└──────┬──────────────────────┘
       │ (periodic aggregation)
       ▼
```

```python
class AsyncRateLimiter:
    """
    Local counter + async sync to Redis.

    Low latency (<0.1ms) with eventual consistency.
    Use for very high throughput (>100K req/sec).
    """

    def __init__(self, redis_client, sync_interval=10):
        self.redis = redis_client
        self.sync_interval = sync_interval
        self.local_counters = {}
        self.last_sync = time.time()

    def allow_request(self, client_id, limit, window):
        """Check using local counter (no Redis call)."""
        now = int(time.time())
        window_key = f"{client_id}:{now // window}"

        # Increment local counter
        self.local_counters[window_key] = self.local_counters.get(window_key, 0) + 1
        count = self.local_counters[window_key]

        # Periodically sync to Redis
        if now - self.last_sync >= self.sync_interval:
            self._sync_to_redis()

        return count <= limit

    def _sync_to_redis(self):
        """Async sync local counters to Redis."""
        import threading

        # Non-blocking sync
        def sync():
            pipe = self.redis.pipeline()
            for key, count in self.local_counters.items():
                # Get current value in Redis
                current = int(self.redis.get(key) or 0)
                # Update (may cause overcounting, but eventually consistent)
                pipe.set(key, current + count)
            pipe.execute()

        thread = threading.Thread(target=sync, daemon=True)
        thread.start()

        # Reset local counters
        self.local_counters.clear()
        self.last_sync = time.time()
```

**Level 4: Sticky Sessions**

```
Avoid Redis altogether for simple use cases.

Load Balancer (consistent hash by IP)

Client A ──┐                ┌─────────────┐
Client B ──┼──→ Server 1 ───┤ local count │
Client C ──┘                │ user A: 50  │
                            │ user B: 30  │
                            │ user C: 20  │
Client D ──┐                └─────────────┘
Client E ──┼──→ Server 2 ───┬─────────────┐
Client F ──┘                │ local count │
                            │ user D: 45  │
                            │ user E: 15  │
                            │ user F: 60  │
                            └─────────────┘

Pros: Zero Redis latency
Cons: Not suitable for strict limits (can bypass)
```

```python
class LocalStickyRateLimiter:
    """Rate limiter with sticky sessions (no Redis needed)."""

    def __init__(self, limit, window):
        self.limit = limit
        self.window = window
        self.counters = {}  # client_id → count
        self.timestamps = {}  # client_id → last_request_time

    def allow_request(self, client_id):
        """Check using local state only."""
        now = time.time()

        # Reset if window expired
        if client_id in self.timestamps:
            last_time = self.timestamps[client_id]
            if now - last_time > self.window:
                self.counters[client_id] = 0

        # Check limit
        count = self.counters.get(client_id, 0)
        if count < self.limit:
            self.counters[client_id] = count + 1
            self.timestamps[client_id] = now
            return True

        return False
```

**Scaling strategy for different scales:**

```
1-10K req/sec:
✓ Single Redis + Lua scripts
✓ Connection pooling
✗ No sharding needed yet

10-100K req/sec:
✓ Redis Cluster with sharding
✓ Pipelining for batch operations
✓ Local Lua scripts

100K - 1M req/sec:
✓ Async local counters + periodic sync
✓ Multi-layer caching
✓ Eventually consistent model
✓ Possible: sticky sessions for non-critical limits

1M+ req/sec:
✓ Hybrid approach:
  - Strict limits: async + sync
  - Soft limits: local only
  - Different algorithms per tier

✗ Avoid: Simple Redis calls per request
✗ Avoid: Heavy coordination between servers
```

**Пример.**

```python
class ScalableRateLimiter:
    """
    Production rate limiter that scales from 1K to 1M req/sec.

    Automatically selects strategy based on:
    - Number of clients
    - Throughput
    - Required accuracy
    - Redis availability
    """

    def __init__(self, redis_cluster, strategy='adaptive'):
        self.redis = redis_cluster
        self.strategy = strategy
        self.local_counters = {}

        # Metrics
        self.requests_per_sec = 0
        self.errors_count = 0
        self.redis_latency_ms = 0

    def allow_request(self, client_id, limit, window):
        """
        Check rate limit with adaptive strategy.
        """
        try:
            if self.strategy == 'adaptive':
                # Choose strategy based on current load
                if self._is_redis_available() and self._latency_acceptable():
                    return self._check_redis(client_id, limit, window)
                else:
                    return self._check_local(client_id, limit, window)
            elif self.strategy == 'strict':
                return self._check_redis(client_id, limit, window)
            elif self.strategy == 'lenient':
                return self._check_local(client_id, limit, window)
        except Exception as e:
            # Fail open on error
            self.errors_count += 1
            return True

    def _is_redis_available(self):
        """Check if Redis is healthy."""
        try:
            self.redis.ping()
            return True
        except:
            return False

    def _latency_acceptable(self):
        """Check if Redis latency is acceptable."""
        return self.redis_latency_ms < 5  # 5ms threshold

    def _check_redis(self, client_id, limit, window):
        """Strict check via Redis."""
        # Actual Redis implementation
        pass

    def _check_local(self, client_id, limit, window):
        """Local in-memory check."""
        # Local implementation
        pass
```

**Типичные ошибки.**
- Масштабировать Redis раньше времени — добавляет complexity без нужды.
- Забыть про network latency при оптимизации — может быть >RTT.
- Использовать eventual consistency для платежей/sensitive операций.
- Не мониторить Redis load и latency — проблемы обнаружатся слишком поздно.

**На интервью.**
- Нарисуй scalability graph: req/sec vs latency vs cost.
- Объясни, когда переходить с одной стратегии на другую.
- Уточняющий вопрос: «Как мониторить rate limiter?» → Metrics, latency, error rate.

---

## Заключение

Rate limiting — это critical component любого scalable API. Key takeaways:

1. **Choose the right algorithm**: Sliding Window Counter для большинства случаев
2. **Use Redis correctly**: Lua scripts, connection pooling, clustering
3. **Communicate with clients**: Proper HTTP headers
4. **Scale incrementally**: Optimize single Redis → cluster → hybrid architecture
5. **Handle failures gracefully**: Fallback strategies, monitoring, alerting

На интервью покажи:
- Понимание различных алгоритмов и их trade-offs
- Практические knowledge Redis и distributed systems
- Ability to think about scale и performance
- Graceful degradation и error handling

---

## См. также

- [URL Shortener](./01-url-shortener.md) — полный системный дизайн с кешированием
- [Notification System](./03-notification-system.md) — асинхронная обработка
- [Concurrency Patterns](../00-go/05-concurrency-patterns.md) — реализация rate limiter на уровне приложения
- [Redis Patterns](../06-databases/09-redis-patterns.md) — распределённые счётчики и блокировки
- [Load Balancing](./04-load-balancing.md) — распределение трафика

---

## Практика

1. **Реализуй Token Bucket** — с параметрами capacity и refill_rate
2. **Distributed rate limiter** — используя Redis и Lua scripts
3. **Graceful degradation** — fallback к local counter если Redis down
4. **HTTP headers** — правильно формировать 429 responses
5. **Масштабирование** — перейти с Single Redis → Cluster → Hybrid
6. **Мониторинг** — отслеживать latency, errors, rate limit violations

---

← [01-url-shortener](./01-url-shortener.md) | [Трек System Design](./README.md) | [03-notification-system](./03-notification-system.md) →
