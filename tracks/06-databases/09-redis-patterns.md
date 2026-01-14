# 09. Redis Patterns

[← Назад к списку тем](README.md)

---

## Основы Redis

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Redis Overview                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  In-memory data store                                               │
│  - Все данные в RAM                                                 │
│  - Persistence опционально (RDB, AOF)                               │
│  - Single-threaded event loop                                       │
│  - Sub-millisecond latency                                          │
│                                                                     │
│  Use cases:                                                         │
│  - Caching                                                          │
│  - Session storage                                                  │
│  - Rate limiting                                                    │
│  - Leaderboards                                                     │
│  - Pub/Sub messaging                                                │
│  - Distributed locks                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Structures

### Strings

```redis
# Basic operations
SET user:123:name "John"
GET user:123:name

# With expiration
SET session:abc123 "user_data" EX 3600  # 1 hour

# Atomic increment
INCR page:views:homepage
INCRBY user:123:points 100

# Only set if not exists (for locks)
SET lock:order:456 "process_1" NX EX 30

# Only set if exists (update)
SET user:123:name "Jane" XX
```

### Hashes

```redis
# User profile as hash
HSET user:123 name "John" email "john@example.com" age 30
HGET user:123 name
HGETALL user:123
HINCRBY user:123 age 1

# Efficient for objects with many fields
# Better than JSON string for partial updates
```

```go
// Go example
type User struct {
    Name  string `redis:"name"`
    Email string `redis:"email"`
    Age   int    `redis:"age"`
}

func getUser(ctx context.Context, rdb *redis.Client, id string) (*User, error) {
    var user User
    err := rdb.HGetAll(ctx, "user:"+id).Scan(&user)
    return &user, err
}
```

### Lists

```redis
# Queue (FIFO)
LPUSH queue:jobs '{"task": "send_email"}'
RPOP queue:jobs  # or BRPOP for blocking

# Stack (LIFO)
LPUSH stack:history "page1"
LPOP stack:history

# Recent items (capped list)
LPUSH user:123:recent_views "product:456"
LTRIM user:123:recent_views 0 99  # Keep only last 100

# Range
LRANGE user:123:recent_views 0 9  # Get first 10
```

### Sets

```redis
# Unique items
SADD user:123:tags "golang" "redis" "postgres"
SMEMBERS user:123:tags
SISMEMBER user:123:tags "redis"  # Check membership

# Set operations
SADD user:123:friends "456" "789"
SADD user:456:friends "123" "789" "999"

# Mutual friends
SINTER user:123:friends user:456:friends  # {"789"}

# Recommendations (friends of friends)
SDIFF user:456:friends user:123:friends  # {"999"}
```

### Sorted Sets

```redis
# Leaderboard
ZADD leaderboard 1000 "player:123"
ZADD leaderboard 950 "player:456"
ZADD leaderboard 1050 "player:789"

# Top 10
ZREVRANGE leaderboard 0 9 WITHSCORES

# Rank of player
ZREVRANK leaderboard "player:123"  # 1 (0-indexed)

# Score range
ZRANGEBYSCORE leaderboard 900 1000 WITHSCORES

# Increment score
ZINCRBY leaderboard 50 "player:123"

# Time-based expiration (score = timestamp)
ZADD events 1704067200 "event:1"
ZRANGEBYSCORE events 0 1704067200  # Events before timestamp
ZREMRANGEBYSCORE events 0 1704067200  # Remove old events
```

### Streams

```redis
# Add to stream
XADD orders * user_id 123 product_id 456 amount 100

# Read from stream
XREAD COUNT 10 STREAMS orders 0

# Consumer groups
XGROUP CREATE orders order-processors $ MKSTREAM
XREADGROUP GROUP order-processors consumer-1 COUNT 1 STREAMS orders >

# Acknowledge processed
XACK orders order-processors 1234567890-0

# Pending entries (not ACKed)
XPENDING orders order-processors
```

---

## Common Patterns

### Caching

```go
func getUser(ctx context.Context, id string) (*User, error) {
    // Try cache first
    cached, err := rdb.Get(ctx, "user:"+id).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    if err != redis.Nil {
        return nil, err
    }

    // Cache miss - fetch from DB
    user, err := db.GetUser(ctx, id)
    if err != nil {
        return nil, err
    }

    // Store in cache
    data, _ := json.Marshal(user)
    rdb.Set(ctx, "user:"+id, data, 1*time.Hour)

    return user, nil
}
```

### Cache Invalidation Strategies

```go
// 1. Write-through: update cache on write
func updateUser(ctx context.Context, user *User) error {
    if err := db.UpdateUser(ctx, user); err != nil {
        return err
    }
    data, _ := json.Marshal(user)
    return rdb.Set(ctx, "user:"+user.ID, data, 1*time.Hour).Err()
}

// 2. Write-behind: invalidate, lazy reload
func updateUser(ctx context.Context, user *User) error {
    if err := db.UpdateUser(ctx, user); err != nil {
        return err
    }
    return rdb.Del(ctx, "user:"+user.ID).Err()
}

// 3. TTL-based: accept staleness
func cacheWithTTL(ctx context.Context, key string, ttl time.Duration, fetch func() (interface{}, error)) (interface{}, error) {
    // Short TTL for frequently changing data
    // Long TTL for stable data
}
```

### Rate Limiting

```go
// Sliding window rate limiter
func isRateLimited(ctx context.Context, key string, limit int, window time.Duration) (bool, error) {
    now := time.Now().UnixNano()
    windowStart := now - int64(window)

    pipe := rdb.Pipeline()

    // Remove old entries
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart, 10))

    // Add current request
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})

    // Count requests in window
    countCmd := pipe.ZCard(ctx, key)

    // Set expiry
    pipe.Expire(ctx, key, window)

    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }

    return countCmd.Val() > int64(limit), nil
}

// Token bucket
func tokenBucket(ctx context.Context, key string, rate float64, capacity int64) (bool, error) {
    script := redis.NewScript(`
        local key = KEYS[1]
        local rate = tonumber(ARGV[1])
        local capacity = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local bucket = redis.call('HMGET', key, 'tokens', 'last_update')
        local tokens = tonumber(bucket[1]) or capacity
        local last_update = tonumber(bucket[2]) or now

        local elapsed = now - last_update
        tokens = math.min(capacity, tokens + elapsed * rate)

        if tokens >= 1 then
            tokens = tokens - 1
            redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
            redis.call('EXPIRE', key, 3600)
            return 1
        end
        return 0
    `)

    result, err := script.Run(ctx, rdb, []string{key},
        rate, capacity, time.Now().Unix()).Int()

    return result == 1, err
}
```

### Distributed Lock

```go
func acquireLock(ctx context.Context, key string, ttl time.Duration) (string, error) {
    token := uuid.New().String()

    ok, err := rdb.SetNX(ctx, "lock:"+key, token, ttl).Result()
    if err != nil {
        return "", err
    }
    if !ok {
        return "", ErrLockNotAcquired
    }

    return token, nil
}

func releaseLock(ctx context.Context, key, token string) error {
    // Lua script for atomic check-and-delete
    script := redis.NewScript(`
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `)

    result, err := script.Run(ctx, rdb, []string{"lock:" + key}, token).Int()
    if err != nil {
        return err
    }
    if result == 0 {
        return ErrLockNotOwned
    }
    return nil
}

// Usage
func processOrder(ctx context.Context, orderID string) error {
    token, err := acquireLock(ctx, "order:"+orderID, 30*time.Second)
    if err != nil {
        return err
    }
    defer releaseLock(ctx, "order:"+orderID, token)

    // Process order...
    return nil
}
```

### Session Storage

```go
type Session struct {
    UserID    string            `json:"user_id"`
    Data      map[string]string `json:"data"`
    ExpiresAt time.Time         `json:"expires_at"`
}

func saveSession(ctx context.Context, sessionID string, session *Session) error {
    data, _ := json.Marshal(session)
    ttl := time.Until(session.ExpiresAt)
    return rdb.Set(ctx, "session:"+sessionID, data, ttl).Err()
}

func getSession(ctx context.Context, sessionID string) (*Session, error) {
    data, err := rdb.Get(ctx, "session:"+sessionID).Bytes()
    if err == redis.Nil {
        return nil, ErrSessionNotFound
    }
    if err != nil {
        return nil, err
    }

    var session Session
    json.Unmarshal(data, &session)
    return &session, nil
}
```

---

## Pub/Sub

```go
// Publisher
func publishEvent(ctx context.Context, channel string, event interface{}) error {
    data, _ := json.Marshal(event)
    return rdb.Publish(ctx, channel, data).Err()
}

// Subscriber
func subscribe(ctx context.Context, channel string, handler func([]byte)) {
    pubsub := rdb.Subscribe(ctx, channel)
    defer pubsub.Close()

    ch := pubsub.Channel()
    for msg := range ch {
        handler([]byte(msg.Payload))
    }
}

// Pattern subscription
pubsub := rdb.PSubscribe(ctx, "orders:*")
// Receives: orders:created, orders:updated, orders:deleted
```

---

## Lua Scripts

```go
// Atomic operations that need multiple commands
var transferScript = redis.NewScript(`
    local from = KEYS[1]
    local to = KEYS[2]
    local amount = tonumber(ARGV[1])

    local from_balance = tonumber(redis.call('GET', from) or 0)
    if from_balance < amount then
        return {err = "insufficient funds"}
    end

    redis.call('DECRBY', from, amount)
    redis.call('INCRBY', to, amount)

    return {ok = "transferred"}
`)

func transfer(ctx context.Context, from, to string, amount int) error {
    result, err := transferScript.Run(ctx, rdb,
        []string{"balance:" + from, "balance:" + to},
        amount).Result()

    if err != nil {
        return err
    }

    // Check result...
    return nil
}
```

---

## Redis Cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Redis Cluster                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  16384 hash slots distributed across nodes                          │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │               │
│  │ Slots 0-5460 │  │ Slots 5461-  │  │ Slots 10923- │               │
│  │              │  │ 10922        │  │ 16383        │               │
│  │  + Replica   │  │  + Replica   │  │  + Replica   │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                     │
│  Key → CRC16(key) mod 16384 → Slot → Node                           │
│                                                                     │
│  Hash tags for co-location:                                         │
│  {user:123}:profile and {user:123}:orders → same slot               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// Cluster client
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{
        "node1:6379",
        "node2:6379",
        "node3:6379",
    },
})

// Hash tags for multi-key operations
rdb.Set(ctx, "{user:123}:profile", "...", 0)
rdb.Set(ctx, "{user:123}:settings", "...", 0)

// MGET works because same hash tag
rdb.MGet(ctx, "{user:123}:profile", "{user:123}:settings")
```

---

## Persistence

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Redis Persistence                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  RDB (Redis Database Backup)                                        │
│  ──────────────────────────                                         │
│  - Point-in-time snapshots                                          │
│  - Compact binary format                                            │
│  - Fast restart                                                     │
│  - Can lose data since last snapshot                                │
│                                                                     │
│  save 900 1     # Snapshot if 1+ keys changed in 900 sec            │
│  save 300 10    # Snapshot if 10+ keys changed in 300 sec           │
│                                                                     │
│  AOF (Append Only File)                                             │
│  ─────────────────────                                              │
│  - Logs every write operation                                       │
│  - More durable (configurable fsync)                                │
│  - Larger file, slower restart                                      │
│  - Can rewrite to compact                                           │
│                                                                     │
│  appendonly yes                                                     │
│  appendfsync everysec  # or always, no                              │
│                                                                     │
│  Recommendation: Use both RDB + AOF                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **Зачем Redis если есть memcached?**
   - Data structures (не только strings)
   - Persistence
   - Pub/Sub
   - Lua scripts
   - Cluster mode

2. **Как реализовать rate limiter?**
   - Sliding window с sorted sets
   - Token bucket с Lua script
   - Fixed window с INCR + EXPIRE

3. **Distributed lock в Redis?**
   - SETNX + TTL
   - Check token before release
   - Redlock для multi-node

4. **Cache invalidation strategies?**
   - Write-through: обновляем кеш при записи
   - Write-behind: инвалидируем, lazy reload
   - TTL-based: принимаем staleness

5. **Redis Cluster vs Sentinel?**
   - Sentinel: HA для single-node, автоматический failover
   - Cluster: шардинг + HA, horizontal scaling

---

[← Назад к списку тем](README.md)
