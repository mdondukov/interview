# 05. News Feed / Timeline

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Публикация постов (текст, изображения, видео)
- Просмотр персонализированной ленты
- Follow/Unfollow пользователей
- Like, comment, share
- Пагинация (infinite scroll)

### Нефункциональные
- Low latency feed generation (< 200ms)
- High availability (99.99%)
- Eventual consistency acceptable
- Scale: 500M DAU, 10K followers avg

---

## Capacity Estimation

```
Users & Connections:
- 500M DAU
- Average 200 followers per user
- 10% users post daily = 50M posts/day

Feed Requests:
- Each user refreshes feed 10x/day
- 500M × 10 = 5B feed requests/day
- QPS = 5B / 86400 ≈ 58,000 QPS
- Peak: 150,000 QPS

Storage:
- Post metadata: ~1KB
- 50M posts × 1KB = 50GB/day
- 5 years: ~90TB

Fanout:
- Celebrity post: 10M followers
- Fanout time constraint: < 5 min
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Client  │────▶│  Load Balancer  │────▶│    API Gateway   │
└──────────┘     └─────────────────┘     └────────┬─────────┘
                                                  │
          ┌───────────────────┬───────────────────┼───────────────────┐
          │                   │                   │                   │
    ┌─────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
    │   Post    │      │    Feed     │     │   Graph     │     │   Media     │
    │  Service  │      │   Service   │     │   Service   │     │   Service   │
    └─────┬─────┘      └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
          │                   │                   │                   │
          │            ┌──────▼──────┐            │                   │
          │            │   Fanout    │            │                   │
          │            │   Service   │            │                   │
          │            └──────┬──────┘            │                   │
          │                   │                   │                   │
    ┌─────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
    │   Posts   │      │ Feed Cache  │     │    Graph    │     │     S3      │
    │    DB     │      │   (Redis)   │     │     DB      │     │     CDN     │
    └───────────┘      └─────────────┘     └─────────────┘     └─────────────┘
```

---

## API Design

### Post Management
```json
POST /api/v1/posts
Request:
{
    "content": "Hello world!",
    "media_ids": ["media_123"],
    "visibility": "public"
}

Response:
{
    "post_id": "post_456",
    "created_at": "2024-01-15T10:00:00Z"
}
```

### Feed Retrieval
```json
GET /api/v1/feed?cursor={cursor}&limit=20

Response:
{
    "posts": [
        {
            "post_id": "post_123",
            "author": {
                "user_id": "user_456",
                "name": "John Doe",
                "avatar_url": "..."
            },
            "content": "Hello!",
            "media": [...],
            "likes_count": 150,
            "comments_count": 23,
            "created_at": "2024-01-15T10:00:00Z"
        }
    ],
    "next_cursor": "cursor_abc123"
}
```

### Social Actions
```
POST /api/v1/posts/{id}/like
DELETE /api/v1/posts/{id}/like
POST /api/v1/posts/{id}/comments
POST /api/v1/users/{id}/follow
DELETE /api/v1/users/{id}/follow
```

---

## Data Model

### Posts (PostgreSQL + Cassandra)
```sql
-- PostgreSQL: Post metadata
CREATE TABLE posts (
    id              UUID PRIMARY KEY,
    author_id       UUID NOT NULL,
    content         TEXT,
    media_urls      TEXT[],
    visibility      VARCHAR(20),
    likes_count     INT DEFAULT 0,
    comments_count  INT DEFAULT 0,
    shares_count    INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_author_created (author_id, created_at DESC)
);

-- Cassandra: User timeline (fanout on write result)
CREATE TABLE user_feed (
    user_id     UUID,
    post_id     TIMEUUID,
    author_id   UUID,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);
```

### Social Graph (Neo4j or PostgreSQL)
```sql
CREATE TABLE follows (
    follower_id     UUID,
    followee_id     UUID,
    created_at      TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);

CREATE INDEX idx_followee ON follows(followee_id);
```

### Feed Cache (Redis)
```
# User's feed (sorted set)
ZADD feed:{user_id} {timestamp} {post_id}
ZREVRANGE feed:{user_id} 0 19  # Get latest 20

# Post details cache
HSET post:{post_id} content "..." author_id "..." ...

# User follower count
SET user:{user_id}:followers_count 1000000
```

---

## Deep Dive

### 1. Fanout Strategies

**Fanout on Write (Push Model)**
```
При публикации → записать в feed всех followers

Pros:
- Быстрое чтение feed (pre-computed)
- Простая логика чтения

Cons:
- Медленная публикация для celebrities
- Wasted storage (inactive users)
- Hotspot при большом числе followers
```

```python
async def publish_post(author_id: str, post: dict):
    # 1. Save post
    post_id = await posts_db.insert(post)

    # 2. Get followers
    followers = await get_followers(author_id)

    # 3. Fanout to followers' feeds
    for batch in chunks(followers, 1000):
        await fanout_queue.send({
            "post_id": post_id,
            "user_ids": batch,
            "timestamp": time.time()
        })
```

**Fanout on Read (Pull Model)**
```
При чтении → собрать посты от всех followees

Pros:
- Быстрая публикация
- Нет wasted storage

Cons:
- Медленное чтение (N queries)
- Сложная агрегация
```

```python
async def get_feed_pull(user_id: str, limit: int = 20):
    # Get all followees
    followees = await get_followees(user_id)

    # Fetch recent posts from each (parallel)
    posts = await asyncio.gather(*[
        get_user_posts(followee_id, limit=5)
        for followee_id in followees
    ])

    # Merge and sort
    all_posts = sorted(
        itertools.chain(*posts),
        key=lambda p: p['created_at'],
        reverse=True
    )[:limit]

    return all_posts
```

**Hybrid Approach (Best)**
```
┌──────────────────────────────────────────────┐
│          User Classification                  │
├──────────────────────────────────────────────┤
│ Regular users (<10K followers):              │
│   → Fanout on Write                          │
│                                              │
│ Celebrities (>10K followers):                │
│   → Fanout on Read (at feed generation time) │
└──────────────────────────────────────────────┘
```

```python
CELEBRITY_THRESHOLD = 10000

async def publish_post(author_id: str, post: dict):
    post_id = await save_post(post)

    follower_count = await get_follower_count(author_id)

    if follower_count < CELEBRITY_THRESHOLD:
        # Fanout on write
        await fanout_to_followers(author_id, post_id)
    else:
        # Mark as celebrity post (pull at read time)
        await mark_celebrity_post(author_id, post_id)

async def get_feed(user_id: str):
    # 1. Get pre-computed feed (fanout on write results)
    feed_posts = await redis.zrevrange(f"feed:{user_id}", 0, 100)

    # 2. Get celebrity followees
    celebrity_followees = await get_celebrity_followees(user_id)

    # 3. Fetch celebrity posts (fanout on read)
    celebrity_posts = await get_recent_celebrity_posts(celebrity_followees)

    # 4. Merge and rank
    return merge_and_rank(feed_posts, celebrity_posts)
```

### 2. Feed Ranking

```python
def calculate_feed_score(post: dict, user: dict) -> float:
    """
    EdgeRank-like scoring formula
    """
    # Time decay
    age_hours = (time.time() - post['created_at']) / 3600
    time_decay = 1 / (1 + age_hours * 0.1)

    # Affinity score (user interaction history with author)
    affinity = get_affinity_score(user['id'], post['author_id'])

    # Engagement signals
    engagement = (
        post['likes_count'] * 1.0 +
        post['comments_count'] * 2.0 +
        post['shares_count'] * 3.0
    )

    # Content type weight
    content_weight = {
        'video': 1.5,
        'image': 1.2,
        'text': 1.0
    }.get(post['type'], 1.0)

    return time_decay * affinity * (1 + math.log(1 + engagement)) * content_weight
```

### 3. Caching Strategy

```
┌───────────────────────────────────────────────────────────┐
│                    Cache Layers                            │
├───────────────────────────────────────────────────────────┤
│ L1: Feed Cache (Redis)                                    │
│     - Pre-computed feeds for active users                 │
│     - TTL: 5 minutes                                      │
│     - Size: top 100 posts per user                        │
├───────────────────────────────────────────────────────────┤
│ L2: Post Cache (Redis)                                    │
│     - Individual post details                              │
│     - TTL: 1 hour                                         │
│     - Hot posts: extended TTL                             │
├───────────────────────────────────────────────────────────┤
│ L3: Social Graph Cache                                    │
│     - Follower/followee lists                             │
│     - Celebrity flags                                     │
└───────────────────────────────────────────────────────────┘
```

```python
async def get_feed_cached(user_id: str, cursor: str = None):
    cache_key = f"feed:{user_id}"

    # Try cache
    cached_feed = await redis.zrevrange(cache_key, 0, 100, withscores=True)

    if cached_feed:
        # Hydrate with post details
        post_ids = [post_id for post_id, _ in cached_feed]
        posts = await get_posts_batch(post_ids)
        return posts

    # Cache miss - generate feed
    feed = await generate_feed(user_id)

    # Cache the feed
    await cache_feed(user_id, feed)

    return feed
```

### 4. Pagination with Cursor

```python
import base64
import json

def encode_cursor(post_id: str, timestamp: float) -> str:
    data = {"post_id": post_id, "ts": timestamp}
    return base64.b64encode(json.dumps(data).encode()).decode()

def decode_cursor(cursor: str) -> dict:
    return json.loads(base64.b64decode(cursor).decode())

async def get_feed_paginated(user_id: str, cursor: str = None, limit: int = 20):
    if cursor:
        cursor_data = decode_cursor(cursor)
        # Fetch posts older than cursor
        posts = await redis.zrevrangebyscore(
            f"feed:{user_id}",
            max=cursor_data['ts'],
            min=0,
            start=1,  # Skip the cursor post
            num=limit
        )
    else:
        posts = await redis.zrevrange(f"feed:{user_id}", 0, limit - 1)

    if posts:
        last_post = posts[-1]
        next_cursor = encode_cursor(last_post['id'], last_post['timestamp'])
    else:
        next_cursor = None

    return {"posts": posts, "next_cursor": next_cursor}
```

### 5. Real-time Updates

```python
# WebSocket for live feed updates
class FeedWebSocket:
    async def on_connect(self, user_id: str, websocket):
        # Subscribe to user's feed channel
        await redis.subscribe(f"feed_updates:{user_id}")

    async def on_new_post(self, user_id: str, post: dict):
        # Push to connected clients
        await websocket.send_json({
            "type": "new_post",
            "post": post
        })

# Notification on new posts from close friends
async def notify_close_friends(author_id: str, post_id: str):
    close_friends = await get_close_friends(author_id)
    for friend_id in close_friends:
        await redis.publish(f"feed_updates:{friend_id}", {
            "type": "new_post",
            "post_id": post_id
        })
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Celebrity post fanout | Hybrid approach (pull for celebrities) |
| Hot posts (viral) | Dedicated cache, CDN for media |
| Feed generation latency | Pre-computed feeds, async ranking |
| Storage for inactive users | TTL on feed cache, lazy generation |
| Consistency (new followers) | Async backfill, eventual consistency |

---

## Trade-offs

### Push vs Pull vs Hybrid

| Approach | Write Latency | Read Latency | Storage | Best For |
|----------|---------------|--------------|---------|----------|
| Push (Fanout on Write) | High | Low | High | Small-medium following |
| Pull (Fanout on Read) | Low | High | Low | Large following |
| Hybrid | Medium | Medium | Medium | Mixed user base |

### Consistency vs Availability

| Choice | Guarantee | Trade-off |
|--------|-----------|-----------|
| Strong | All see same feed | Higher latency |
| Eventual | Fast reads | Temporary inconsistency |

**Рекомендация:** Eventual consistency — пользователи не заметят небольшую задержку в появлении постов.

---

## На интервью

### Ключевые моменты
1. **Fanout strategy** — hybrid approach для разных типов пользователей
2. **Caching** — multi-layer, pre-computed feeds
3. **Ranking** — personalization vs chronological
4. **Pagination** — cursor-based для infinite scroll

### Типичные follow-up
- Как обработать viral post с миллионами likes?
- Как реализовать "Seen by" для постов?
- Как добавить stories (24h expiration)?
- Как бороться со spam и fake news?
- Как реализовать группы/communities?

---

[← Назад к списку тем](README.md)
