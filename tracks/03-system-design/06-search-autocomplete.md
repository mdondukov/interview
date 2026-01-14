# 06. Search Autocomplete / Typeahead

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Предложения по мере ввода (typeahead)
- Top suggestions based on popularity
- Персонализация (история поиска)
- Multi-language support
- Spell correction (опционально)

### Нефункциональные
- Ultra-low latency (< 100ms)
- High availability (99.99%)
- Scale: 10B queries/day
- Fresh suggestions (hourly updates)

---

## Capacity Estimation

```
Queries:
- 10B search queries/day
- ~5 characters typed per query on average
- 10B × 5 = 50B autocomplete requests/day
- QPS = 50B / 86400 ≈ 580,000 QPS
- Peak: 1.5M QPS

Storage:
- 5M unique search terms
- Average term: 20 characters = 20 bytes
- With metadata: ~100 bytes per term
- Total: 5M × 100B = 500MB (fits in memory!)

Bandwidth:
- Response: ~1KB (10 suggestions × 100 bytes)
- 580K × 1KB = 580MB/s
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Client  │────▶│  Load Balancer  │────▶│ Autocomplete     │
│          │     │                 │     │ Service          │
└──────────┘     └─────────────────┘     └────────┬─────────┘
                                                  │
                          ┌───────────────────────┴───────────────────────┐
                          │                                               │
                   ┌──────▼──────┐                                 ┌──────▼──────┐
                   │    Trie     │                                 │   Search    │
                   │   Service   │                                 │   Ranking   │
                   └──────┬──────┘                                 └──────┬──────┘
                          │                                               │
                   ┌──────▼──────┐                                 ┌──────▼──────┐
                   │  Trie DB    │                                 │  Analytics  │
                   │  (Redis)    │                                 │    DB       │
                   └─────────────┘                                 └─────────────┘
                          │
                   ┌──────▼──────┐
                   │  Data       │
                   │  Pipeline   │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │   Query     │
                   │   Logs      │
                   └─────────────┘
```

---

## API Design

### Autocomplete Request
```json
GET /api/v1/autocomplete?q=how+to&limit=10&lang=en

Response:
{
    "suggestions": [
        {"text": "how to learn python", "score": 95000},
        {"text": "how to tie a tie", "score": 87000},
        {"text": "how to lose weight", "score": 82000},
        {"text": "how to cook rice", "score": 75000}
    ],
    "query_time_ms": 12
}
```

### Search Query Logging
```json
POST /api/v1/search
{
    "query": "how to learn python",
    "user_id": "user_123",
    "timestamp": "2024-01-15T10:00:00Z"
}
```

---

## Data Model

### Trie Node Structure
```python
class TrieNode:
    def __init__(self):
        self.children = {}  # char -> TrieNode
        self.is_end = False
        self.top_suggestions = []  # [(score, term), ...] - precomputed top K

# In-memory representation
# "cat", "car", "card", "care"
#
#     (root)
#       │
#       c
#       │
#       a ─── top: ["cat", "car", "card"]
#      / \
#     t   r ─── top: ["car", "card", "care"]
#         |
#        (d, e)
```

### Redis Storage
```
# Sorted set for prefix → suggestions
ZADD autocomplete:prefix:how 95000 "how to learn python"
ZADD autocomplete:prefix:how 87000 "how to tie a tie"
ZADD autocomplete:prefix:how+ 95000 "how to learn python"
ZADD autocomplete:prefix:how+t 87000 "how to tie a tie"

# Query frequency
ZINCRBY query:frequency 1 "how to learn python"

# User search history
ZADD user:123:history {timestamp} "how to learn python"
```

### Search Analytics (Cassandra)
```sql
CREATE TABLE search_queries (
    date        DATE,
    hour        INT,
    query       TEXT,
    count       COUNTER,
    PRIMARY KEY ((date, hour), query)
);

CREATE TABLE query_daily_stats (
    query       TEXT,
    date        DATE,
    count       BIGINT,
    PRIMARY KEY (query, date)
) WITH CLUSTERING ORDER BY (date DESC);
```

---

## Deep Dive

### 1. Trie Data Structure

```python
class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str, score: int):
        node = self.root
        prefix = ""

        for char in word.lower():
            prefix += char
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

            # Update top suggestions for this prefix
            self._update_top_suggestions(node, word, score)

        node.is_end = True

    def _update_top_suggestions(self, node: TrieNode, word: str, score: int, k: int = 10):
        # Maintain top K suggestions sorted by score
        suggestions = node.top_suggestions

        # Remove if exists (for update)
        suggestions = [(s, w) for s, w in suggestions if w != word]

        # Add new
        suggestions.append((score, word))

        # Keep top K
        suggestions.sort(reverse=True)
        node.top_suggestions = suggestions[:k]

    def search(self, prefix: str, limit: int = 10) -> list:
        node = self.root

        for char in prefix.lower():
            if char not in node.children:
                return []
            node = node.children[char]

        # Return precomputed top suggestions
        return [word for score, word in node.top_suggestions[:limit]]
```

### 2. Distributed Trie with Sharding

```
┌─────────────────────────────────────────────────────────┐
│                    Sharding Strategy                     │
├─────────────────────────────────────────────────────────┤
│ Shard by first character:                               │
│   Shard 0: a-f                                          │
│   Shard 1: g-l                                          │
│   Shard 2: m-r                                          │
│   Shard 3: s-z                                          │
│   Shard 4: 0-9, special chars                           │
└─────────────────────────────────────────────────────────┘
```

```python
class DistributedAutocomplete:
    def __init__(self, shards: list):
        self.shards = shards  # Redis connections

    def get_shard(self, prefix: str) -> Redis:
        if not prefix:
            return self.shards[0]

        first_char = prefix[0].lower()
        if 'a' <= first_char <= 'f':
            return self.shards[0]
        elif 'g' <= first_char <= 'l':
            return self.shards[1]
        elif 'm' <= first_char <= 'r':
            return self.shards[2]
        elif 's' <= first_char <= 'z':
            return self.shards[3]
        else:
            return self.shards[4]

    async def search(self, prefix: str, limit: int = 10) -> list:
        shard = self.get_shard(prefix)
        key = f"autocomplete:prefix:{prefix.lower()}"

        # Get top suggestions from sorted set
        results = await shard.zrevrange(key, 0, limit - 1, withscores=True)

        return [{"text": text, "score": int(score)} for text, score in results]
```

### 3. Data Collection Pipeline

```
┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│   Search   │────▶│   Kafka    │────▶│   Spark    │────▶│   Trie     │
│   Logs     │     │            │     │  Aggregator│     │  Builder   │
└────────────┘     └────────────┘     └────────────┘     └────────────┘
                                                                │
                                            ┌───────────────────┘
                                            │
                                     ┌──────▼──────┐
                                     │   Trie DB   │
                                     │  (Update)   │
                                     └─────────────┘
```

```python
# Spark job for aggregating search queries
def aggregate_queries(date_range):
    queries = spark.read.parquet(f"s3://logs/search/{date_range}")

    # Count query frequency
    query_counts = queries \
        .groupBy("query") \
        .agg(
            count("*").alias("frequency"),
            countDistinct("user_id").alias("unique_users")
        )

    # Calculate score
    scored_queries = query_counts \
        .withColumn("score",
            col("frequency") * 0.7 + col("unique_users") * 0.3
        )

    return scored_queries.collect()

def build_trie(scored_queries):
    trie = Trie()
    for row in scored_queries:
        trie.insert(row.query, int(row.score))
    return trie

def update_redis_trie(trie):
    for node, prefix in trie.traverse_with_prefix():
        key = f"autocomplete:prefix:{prefix}"
        for score, word in node.top_suggestions:
            redis.zadd(key, {word: score})
        redis.expire(key, 86400)  # 24h TTL
```

### 4. Ranking & Personalization

```python
class SuggestionRanker:
    def rank(self, suggestions: list, user_id: str, context: dict) -> list:
        scored = []

        for suggestion in suggestions:
            base_score = suggestion['score']

            # Personalization boost
            personal_boost = self.get_personal_boost(user_id, suggestion['text'])

            # Recency boost
            recency_boost = self.get_recency_boost(suggestion['text'])

            # Context boost (location, time, device)
            context_boost = self.get_context_boost(suggestion['text'], context)

            final_score = base_score * (1 + personal_boost + recency_boost + context_boost)
            scored.append({**suggestion, 'final_score': final_score})

        return sorted(scored, key=lambda x: x['final_score'], reverse=True)

    def get_personal_boost(self, user_id: str, query: str) -> float:
        # Check user's search history
        history_rank = redis.zrevrank(f"user:{user_id}:history", query)
        if history_rank is not None:
            return 0.5 / (1 + history_rank)  # Higher boost for recent searches
        return 0

    def get_recency_boost(self, query: str) -> float:
        # Trending queries get boost
        trending_score = redis.zscore("trending:queries:today", query)
        if trending_score:
            return min(trending_score / 10000, 0.3)
        return 0
```

### 5. Optimizations

**Sampling for Data Collection**
```python
# Don't log every query - sample
def should_log_query(user_id: str, query: str) -> bool:
    # Log 10% of queries
    if hash(f"{user_id}:{query}") % 10 == 0:
        return True

    # Always log unique/rare queries
    if redis.zscore("query:frequency", query) is None:
        return True

    return False
```

**Caching at Edge (CDN)**
```python
# Popular prefixes cached at CDN
CACHEABLE_PREFIXES = set()

def update_cacheable_prefixes():
    # Top 1000 prefixes
    top_prefixes = redis.zrevrange("prefix:frequency", 0, 999)
    CACHEABLE_PREFIXES.update(top_prefixes)

def get_suggestions(prefix: str) -> dict:
    response = get_suggestions_internal(prefix)

    # Add cache headers for popular prefixes
    if prefix in CACHEABLE_PREFIXES:
        response.headers['Cache-Control'] = 'public, max-age=60'
    else:
        response.headers['Cache-Control'] = 'private, max-age=0'

    return response
```

**Browser/Client Caching**
```javascript
// Client-side cache for suggestions
const suggestionCache = new Map();

async function getSuggestions(prefix) {
    // Check local cache first
    if (suggestionCache.has(prefix)) {
        const cached = suggestionCache.get(prefix);
        if (Date.now() - cached.timestamp < 60000) {  // 1 min TTL
            return cached.data;
        }
    }

    const response = await fetch(`/api/autocomplete?q=${prefix}`);
    const data = await response.json();

    suggestionCache.set(prefix, {data, timestamp: Date.now()});

    return data;
}

// Debounce requests
function debounce(fn, delay) {
    let timer;
    return function(...args) {
        clearTimeout(timer);
        timer = setTimeout(() => fn(...args), delay);
    };
}

const debouncedGetSuggestions = debounce(getSuggestions, 100);
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| High QPS | CDN caching, client debounce |
| Trie size | Sharding by prefix, pruning rare queries |
| Data freshness | Incremental updates, streaming pipeline |
| Personalization at scale | Approximate algorithms, sampling |
| Multi-language | Separate tries per language |

---

## Trade-offs

### Trie vs Elasticsearch

| Approach | Latency | Flexibility | Complexity |
|----------|---------|-------------|------------|
| Trie (in-memory) | <10ms | Limited | Low |
| Elasticsearch | 50-100ms | High (fuzzy, typo tolerance) | Medium |
| Hybrid | 10-50ms | Medium | High |

### Freshness vs Performance

| Update Frequency | Freshness | Resource Cost |
|------------------|-----------|---------------|
| Real-time | High | Very High |
| Hourly | Medium | Medium |
| Daily | Low | Low |

**Рекомендация:** Hourly batch updates + real-time for trending topics

---

## На интервью

### Ключевые моменты
1. **Trie structure** — precomputed top K at each node
2. **Low latency** — in-memory, CDN caching, client debounce
3. **Data pipeline** — collection → aggregation → trie build
4. **Personalization** — user history, trending boost

### Типичные follow-up
- Как обработать typos и fuzzy matching?
- Как добавить поддержку нескольких языков?
- Как фильтровать inappropriate suggestions?
- Как обрабатывать trending topics в real-time?
- Как приоритизировать location-based suggestions?

---

## См. также

- [Деревья и обходы](../05-algorithms/06-trees-traversal.md) — структура данных Trie для автодополнения
- [Индексирование в базах данных](../06-databases/01-indexing.md) — оптимизация поиска и индексов

---

[← Назад к списку тем](README.md)
