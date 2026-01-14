# 08. NoSQL Patterns

[← Назад к списку тем](README.md)

---

## Типы NoSQL

```
┌─────────────────────────────────────────────────────────────────────┐
│                        NoSQL Categories                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Document Store                Key-Value Store                      │
│  ──────────────                ───────────────                      │
│  MongoDB, CouchDB              Redis, DynamoDB                      │
│  JSON-like documents           Simple key → value                   │
│  Flexible schema               Fastest reads/writes                 │
│  Good for: content, catalog    Good for: cache, sessions            │
│                                                                     │
│  Wide-Column Store             Graph Database                       │
│  ─────────────────             ──────────────                       │
│  Cassandra, HBase              Neo4j, Neptune                       │
│  Column families               Nodes + Relationships                │
│  Time-series, IoT              Good for: social networks,           │
│  Good for: analytics           recommendations, fraud               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Document Store (MongoDB)

### Data Model

```javascript
// Embedded documents (denormalized)
{
  "_id": ObjectId("..."),
  "name": "John Doe",
  "email": "john@example.com",
  "address": {                    // Embedded
    "street": "123 Main St",
    "city": "NYC",
    "zip": "10001"
  },
  "orders": [                     // Embedded array
    {
      "order_id": "ord_123",
      "items": ["product_1", "product_2"],
      "total": 150.00,
      "status": "delivered"
    }
  ]
}

// Referenced documents (normalized)
// users collection
{
  "_id": ObjectId("user_123"),
  "name": "John Doe",
  "email": "john@example.com"
}

// orders collection
{
  "_id": ObjectId("order_456"),
  "user_id": ObjectId("user_123"),  // Reference
  "items": [...],
  "total": 150.00
}
```

### Когда embed, когда reference

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Embed vs Reference                                │
├──────────────────────────────────┬──────────────────────────────────┤
│           Embed                  │          Reference               │
├──────────────────────────────────┼──────────────────────────────────┤
│ One-to-few relationships         │ One-to-many (unbounded)          │
│ Data read together               │ Data accessed independently      │
│ Atomic updates needed            │ Many-to-many relationships       │
│ Document < 16MB                  │ Frequently updated subdocuments  │
│                                  │ Need to query subdocuments alone │
└──────────────────────────────────┴──────────────────────────────────┘

// ✅ Embed: адрес пользователя (1:1, читается вместе)
// ✅ Embed: комментарии к посту (1:few, до ~100)
// ❌ Reference: заказы пользователя (1:many, могут быть тысячи)
// ❌ Reference: теги (many:many, переиспользуются)
```

### Indexes in MongoDB

```javascript
// Single field index
db.users.createIndex({ email: 1 })  // 1 = ascending, -1 = descending

// Compound index
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Multikey index (arrays)
db.posts.createIndex({ tags: 1 })

// Text index
db.articles.createIndex({ title: "text", body: "text" })
db.articles.find({ $text: { $search: "mongodb tutorial" }})

// TTL index (auto-delete)
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 })
```

### Aggregation Pipeline

```javascript
db.orders.aggregate([
  // Stage 1: Match
  { $match: { status: "completed" }},

  // Stage 2: Group
  { $group: {
      _id: "$user_id",
      total_orders: { $sum: 1 },
      total_amount: { $sum: "$amount" }
  }},

  // Stage 3: Sort
  { $sort: { total_amount: -1 }},

  // Stage 4: Limit
  { $limit: 10 },

  // Stage 5: Lookup (JOIN)
  { $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user"
  }},

  // Stage 6: Unwind
  { $unwind: "$user" },

  // Stage 7: Project
  { $project: {
      user_name: "$user.name",
      total_orders: 1,
      total_amount: 1
  }}
])
```

---

## Key-Value Store (DynamoDB)

### Data Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DynamoDB Concepts                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Table: users                                                       │
│  ┌──────────────┬─────────────┬───────────────────────────────┐     │
│  │ Partition Key│  Sort Key   │  Attributes                   │     │
│  │ (PK)         │  (SK)       │                               │     │
│  ├──────────────┼─────────────┼───────────────────────────────┤     │
│  │ USER#123     │ PROFILE     │ name: "John", email: "..."    │     │
│  │ USER#123     │ ORDER#001   │ total: 100, status: "shipped" │     │
│  │ USER#123     │ ORDER#002   │ total: 250, status: "pending" │     │
│  │ USER#456     │ PROFILE     │ name: "Jane", email: "..."    │     │
│  └──────────────┴─────────────┴───────────────────────────────┘     │
│                                                                     │
│  Single Table Design:                                               │
│  - Все сущности в одной таблице                                     │
│  - PK + SK определяют тип записи                                    │
│  - Позволяет получить связанные данные одним запросом               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Access Patterns

```python
# Get user profile
table.get_item(
    Key={
        'PK': 'USER#123',
        'SK': 'PROFILE'
    }
)

# Get all orders for user
table.query(
    KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues={
        ':pk': 'USER#123',
        ':sk': 'ORDER#'
    }
)

# Get user profile + recent orders (one query!)
table.query(
    KeyConditionExpression='PK = :pk',
    ExpressionAttributeValues={
        ':pk': 'USER#123'
    }
)
```

### Global Secondary Index (GSI)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GSI Example                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Main Table: orders                                                 │
│  PK: order_id                                                       │
│                                                                     │
│  Access pattern: "Get all orders by status"                         │
│                                                                     │
│  GSI: status-created-index                                          │
│  PK: status                                                         │
│  SK: created_at                                                     │
│                                                                     │
│  ┌────────────┬──────────────┬─────────────┐                        │
│  │ status(PK) │ created_at(SK)│ order_id    │                       │
│  ├────────────┼──────────────┼─────────────┤                        │
│  │ pending    │ 2024-01-15   │ order_001   │                        │
│  │ pending    │ 2024-01-16   │ order_005   │                        │
│  │ shipped    │ 2024-01-14   │ order_002   │                        │
│  └────────────┴──────────────┴─────────────┘                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Wide-Column Store (Cassandra)

### Data Model

```cql
-- Keyspace (like database)
CREATE KEYSPACE ecommerce
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,
    'dc2': 3
};

-- Table
CREATE TABLE orders (
    user_id UUID,
    order_id TIMEUUID,
    status TEXT,
    total DECIMAL,
    created_at TIMESTAMP,
    PRIMARY KEY ((user_id), order_id)
) WITH CLUSTERING ORDER BY (order_id DESC);

-- Partition key: user_id (determines node)
-- Clustering key: order_id (sorts within partition)
```

### Query Patterns

```cql
-- ✅ Queries that work (follow primary key)
SELECT * FROM orders WHERE user_id = ?;
SELECT * FROM orders WHERE user_id = ? AND order_id = ?;
SELECT * FROM orders WHERE user_id = ? AND order_id > ? LIMIT 10;

-- ❌ Queries that DON'T work (need secondary index or ALLOW FILTERING)
SELECT * FROM orders WHERE status = 'pending';  -- No partition key!
SELECT * FROM orders WHERE total > 100;

-- Secondary index (use sparingly)
CREATE INDEX ON orders (status);
SELECT * FROM orders WHERE user_id = ? AND status = 'pending';

-- Materialized View
CREATE MATERIALIZED VIEW orders_by_status AS
    SELECT * FROM orders
    WHERE status IS NOT NULL AND user_id IS NOT NULL AND order_id IS NOT NULL
    PRIMARY KEY ((status), user_id, order_id);

SELECT * FROM orders_by_status WHERE status = 'pending';
```

### Write Path

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Cassandra Write Path                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client Write                                                       │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────┐                                                    │
│  │ Coordinator │  (любая нода может быть координатором)              │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼ Replicate to N nodes                                      │
│  ┌──────────────────────────────────────────────────────┐           │
│  │  Each Node:                                          │           │
│  │  1. Write to Commit Log (durability)                 │           │
│  │  2. Write to Memtable (memory)                       │           │
│  │  3. Flush to SSTable when full (disk)                │           │
│  └──────────────────────────────────────────────────────┘           │
│                                                                     │
│  Consistency Level:                                                 │
│  - ONE: write ack from 1 node                                       │
│  - QUORUM: write ack from majority (N/2 + 1)                        │
│  - ALL: write ack from all replicas                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Graph Database (Neo4j)

### Data Model

```cypher
// Nodes (entities)
CREATE (john:Person {name: 'John', age: 30})
CREATE (jane:Person {name: 'Jane', age: 28})
CREATE (company:Company {name: 'TechCorp'})

// Relationships
CREATE (john)-[:WORKS_AT {since: 2020}]->(company)
CREATE (jane)-[:WORKS_AT {since: 2019}]->(company)
CREATE (john)-[:FRIENDS_WITH]->(jane)
```

### Queries

```cypher
// Find friends of friends
MATCH (person:Person {name: 'John'})-[:FRIENDS_WITH*2]-(fof:Person)
WHERE person <> fof
RETURN DISTINCT fof.name

// Shortest path
MATCH path = shortestPath(
    (a:Person {name: 'John'})-[:KNOWS*]-(b:Person {name: 'Alice'})
)
RETURN path

// Recommendations (people who work at same company and have mutual friends)
MATCH (me:Person {name: 'John'})-[:WORKS_AT]->(company)<-[:WORKS_AT]-(colleague)
WHERE (me)-[:FRIENDS_WITH]-(:Person)-[:FRIENDS_WITH]-(colleague)
  AND NOT (me)-[:FRIENDS_WITH]-(colleague)
RETURN DISTINCT colleague.name as recommendation

// PageRank-like: most connected people
MATCH (p:Person)-[r:FRIENDS_WITH]-()
RETURN p.name, count(r) as connections
ORDER BY connections DESC
LIMIT 10
```

### When to use Graph DB

```
✅ Good for:
- Social networks (friends, followers)
- Recommendation engines
- Fraud detection (connected entities)
- Knowledge graphs
- Network/IT infrastructure
- Access control (role hierarchies)

❌ Not good for:
- Simple CRUD operations
- High-volume writes
- Large result sets
- Aggregations over entire dataset
```

---

## Выбор NoSQL

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NoSQL Decision Tree                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Need relationships traversal?                                      │
│  └─ Yes → Graph DB (Neo4j)                                          │
│  └─ No ↓                                                            │
│                                                                     │
│  Simple key-value access?                                           │
│  └─ Yes → Key-Value (Redis, DynamoDB)                               │
│  └─ No ↓                                                            │
│                                                                     │
│  Time-series / wide data?                                           │
│  └─ Yes → Wide-Column (Cassandra, HBase)                            │
│  └─ No ↓                                                            │
│                                                                     │
│  Flexible schema / nested data?                                     │
│  └─ Yes → Document (MongoDB)                                        │
│  └─ No → Maybe stick with SQL?                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Redis Patterns](./09-redis-patterns.md) — паттерны использования Redis
- [Vector Databases](../04-ai-integration/03-vector-databases.md) — векторные базы данных для AI

---

## На интервью

### Типичные вопросы

1. **Когда использовать NoSQL вместо SQL?**
   - Flexible schema
   - Horizontal scaling
   - Specific access patterns
   - High write throughput

2. **MongoDB: embed vs reference?**
   - Embed: 1:few, read together, atomic updates
   - Reference: 1:many, independent access, many:many

3. **DynamoDB: как моделировать данные?**
   - Start with access patterns
   - Single table design
   - PK + SK для разных сущностей
   - GSI для дополнительных patterns

4. **Cassandra: partition key vs clustering key?**
   - Partition key: determines node
   - Clustering key: sorts within partition
   - Query должен включать partition key

5. **Graph DB: когда использовать?**
   - Relationship traversal
   - Variable depth queries
   - Connected data patterns

---

[← Назад к списку тем](README.md)
