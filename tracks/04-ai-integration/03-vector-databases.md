# 03. Vector Databases

[← Назад к списку тем](README.md)

---

## Vector DB Fundamentals

### Why Vector Databases?

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Vector Database Purpose                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Traditional DB:  SELECT * FROM docs WHERE content LIKE '%python%'  │
│  → Exact keyword match only                                         │
│                                                                     │
│  Vector DB: Find documents similar to query embedding               │
│  → Semantic similarity (meaning, not just keywords)                 │
│                                                                     │
│  Use cases:                                                         │
│  - RAG: retrieve relevant context for LLM                           │
│  - Semantic search: find similar documents                          │
│  - Recommendations: find similar items/users                        │
│  - Anomaly detection: find outliers                                 │
│  - Image search: find visually similar images                       │
│                                                                     │
│  Key operations:                                                    │
│  - Insert: store vector + metadata                                  │
│  - Search: find k nearest neighbors (KNN)                           │
│  - Filter: combine vector search with metadata filters              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Similarity Metrics

```python
import numpy as np

# Cosine Similarity (most common for text)
def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    """
    Range: [-1, 1], higher is more similar
    Measures angle between vectors, ignores magnitude
    Best for: text embeddings (normalized)
    """
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Euclidean Distance (L2)
def euclidean_distance(a: np.ndarray, b: np.ndarray) -> float:
    """
    Range: [0, inf), lower is more similar
    Measures straight-line distance
    Best for: when magnitude matters
    """
    return np.linalg.norm(a - b)

# Dot Product (Inner Product)
def dot_product(a: np.ndarray, b: np.ndarray) -> float:
    """
    Range: (-inf, inf), higher is more similar
    Fast, but requires normalized vectors for meaningful comparison
    Best for: pre-normalized embeddings, performance-critical
    """
    return np.dot(a, b)

# Manhattan Distance (L1)
def manhattan_distance(a: np.ndarray, b: np.ndarray) -> float:
    """
    Sum of absolute differences
    Best for: sparse vectors, grid-like spaces
    """
    return np.sum(np.abs(a - b))
```

### Indexing Algorithms

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Vector Index Types                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Flat (Brute Force):                                                │
│  - Exact search, O(n) per query                                     │
│  - Perfect recall, slowest                                          │
│  - Good for: < 10K vectors                                          │
│                                                                     │
│  IVF (Inverted File Index):                                         │
│  - Clusters vectors, searches relevant clusters                     │
│  - Trade-off: speed vs recall (nprobe parameter)                    │
│  - Good for: 10K - 1M vectors                                       │
│                                                                     │
│  HNSW (Hierarchical Navigable Small World):                         │
│  - Graph-based, very fast search                                    │
│  - High recall, more memory                                         │
│  - Good for: 10K - 100M vectors                                     │
│                                                                     │
│  PQ (Product Quantization):                                         │
│  - Compresses vectors, reduces memory                               │
│  - Lower recall, much smaller footprint                             │
│  - Good for: memory-constrained, 1M+ vectors                        │
│                                                                     │
│  Hybrid (IVF-PQ, HNSW-PQ):                                          │
│  - Combines approaches for scale                                    │
│  - Best for: billion-scale                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Popular Vector Databases

### Pinecone (Managed)

```python
from pinecone import Pinecone, ServerlessSpec

# Initialize
pc = Pinecone(api_key="your-api-key")

# Create index
pc.create_index(
    name="my-index",
    dimension=1536,  # Match your embedding model
    metric="cosine",
    spec=ServerlessSpec(cloud="aws", region="us-east-1")
)

index = pc.Index("my-index")

# Upsert vectors
index.upsert(
    vectors=[
        {
            "id": "doc1",
            "values": [0.1, 0.2, ...],  # embedding
            "metadata": {"source": "wiki", "category": "science"}
        },
        {
            "id": "doc2",
            "values": [0.3, 0.4, ...],
            "metadata": {"source": "blog", "category": "tech"}
        }
    ],
    namespace="documents"  # Optional namespace for isolation
)

# Query with filter
results = index.query(
    vector=[0.15, 0.25, ...],
    top_k=5,
    filter={"category": {"$eq": "science"}},
    include_metadata=True,
    namespace="documents"
)

for match in results.matches:
    print(f"{match.id}: {match.score} - {match.metadata}")
```

### Qdrant (Open Source / Managed)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

# Connect (local or cloud)
client = QdrantClient(":memory:")  # or url="https://..."

# Create collection
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE)
)

# Insert points
client.upsert(
    collection_name="documents",
    points=[
        PointStruct(
            id=1,
            vector=[0.1, 0.2, ...],
            payload={"text": "Document content", "source": "wiki"}
        ),
        PointStruct(
            id=2,
            vector=[0.3, 0.4, ...],
            payload={"text": "Another doc", "source": "blog"}
        )
    ]
)

# Search with filters
results = client.search(
    collection_name="documents",
    query_vector=[0.15, 0.25, ...],
    limit=5,
    query_filter={
        "must": [{"key": "source", "match": {"value": "wiki"}}]
    }
)

for result in results:
    print(f"{result.id}: {result.score} - {result.payload}")
```

### Chroma (Lightweight, Local-First)

```python
import chromadb
from chromadb.utils import embedding_functions

# Create client
client = chromadb.Client()  # or PersistentClient("./chroma_db")

# With OpenAI embeddings
openai_ef = embedding_functions.OpenAIEmbeddingFunction(
    api_key="your-key",
    model_name="text-embedding-3-small"
)

# Create collection
collection = client.create_collection(
    name="documents",
    embedding_function=openai_ef
)

# Add documents (auto-embeds)
collection.add(
    documents=["Document one content", "Document two content"],
    metadatas=[{"source": "wiki"}, {"source": "blog"}],
    ids=["doc1", "doc2"]
)

# Query (auto-embeds query)
results = collection.query(
    query_texts=["search query"],
    n_results=5,
    where={"source": "wiki"}  # Optional filter
)

print(results["documents"])
print(results["distances"])
```

### pgvector (PostgreSQL Extension)

```python
import psycopg2
from pgvector.psycopg2 import register_vector

# Connect
conn = psycopg2.connect("postgresql://localhost/mydb")
register_vector(conn)

# Create table with vector column
cur = conn.cursor()
cur.execute("""
    CREATE EXTENSION IF NOT EXISTS vector;

    CREATE TABLE documents (
        id SERIAL PRIMARY KEY,
        content TEXT,
        embedding vector(1536),
        metadata JSONB
    );

    CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
""")

# Insert
cur.execute("""
    INSERT INTO documents (content, embedding, metadata)
    VALUES (%s, %s, %s)
""", ("Document content", [0.1, 0.2, ...], {"source": "wiki"}))

# Search
cur.execute("""
    SELECT id, content, 1 - (embedding <=> %s) as similarity
    FROM documents
    WHERE metadata->>'source' = 'wiki'
    ORDER BY embedding <=> %s
    LIMIT 5
""", ([0.15, 0.25, ...], [0.15, 0.25, ...]))

results = cur.fetchall()
```

---

## Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                Vector Database Comparison                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Database    │ Type      │ Best For           │ Scale    │ Cost    │
│  ────────────┼───────────┼────────────────────┼──────────┼─────────│
│  Pinecone    │ Managed   │ Production, scale  │ Billions │ $$$     │
│  Qdrant      │ OSS/Cloud │ Flexibility, perf  │ Billions │ $/$$    │
│  Chroma      │ OSS       │ Dev, prototyping   │ Millions │ Free    │
│  pgvector    │ Extension │ Existing Postgres  │ Millions │ Free    │
│  Weaviate    │ OSS/Cloud │ Hybrid search      │ Billions │ $/$$    │
│  Milvus      │ OSS       │ Large scale OSS    │ Billions │ Free    │
│                                                                     │
│  Selection criteria:                                                │
│  - Existing infra: pgvector if you have Postgres                    │
│  - Quick prototype: Chroma                                          │
│  - Production managed: Pinecone or Qdrant Cloud                     │
│  - Self-hosted scale: Milvus or Qdrant                              │
│  - Hybrid search: Weaviate                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### Metadata Design

```python
# Good metadata structure
document = {
    "id": "doc_12345",
    "embedding": [...],
    "metadata": {
        # Filterable fields (indexed)
        "source": "confluence",
        "category": "engineering",
        "date": "2024-01-15",
        "author_id": 42,

        # Non-filterable (stored only)
        "title": "System Design Guide",
        "url": "https://...",
        "chunk_index": 3,
        "parent_id": "doc_12340"
    }
}

# Query with filters
results = index.query(
    vector=query_embedding,
    top_k=10,
    filter={
        "source": {"$eq": "confluence"},
        "date": {"$gte": "2024-01-01"},
        "category": {"$in": ["engineering", "product"]}
    }
)
```

### Batch Operations

```python
def batch_upsert(documents: list, batch_size: int = 100):
    """Efficient batch insertion."""
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]

        # Embed batch
        texts = [d["text"] for d in batch]
        embeddings = embed_batch(texts)

        # Prepare for upsert
        vectors = [
            {
                "id": d["id"],
                "values": emb,
                "metadata": d["metadata"]
            }
            for d, emb in zip(batch, embeddings)
        ]

        # Upsert batch
        index.upsert(vectors=vectors)

        # Rate limiting if needed
        time.sleep(0.1)
```

### Hybrid Search Implementation

```python
class HybridVectorStore:
    """Combine vector search with full-text search."""

    def __init__(self):
        self.vector_db = QdrantClient()
        self.bm25_index = {}  # Or Elasticsearch

    def add(self, doc_id: str, text: str, embedding: list):
        # Add to vector DB
        self.vector_db.upsert(...)

        # Add to full-text index
        self.bm25_index[doc_id] = text

    def search(
        self,
        query: str,
        query_embedding: list,
        top_k: int = 10,
        alpha: float = 0.5
    ) -> list:
        # Vector search
        vector_results = self.vector_db.search(
            query_vector=query_embedding,
            limit=top_k * 2
        )

        # Full-text search (bm25_search method needs to be implemented)
        text_results = self.bm25_search(query, top_k * 2)  # TODO: implement bm25_search

        # Reciprocal Rank Fusion
        return self.rrf_merge(vector_results, text_results, top_k)

    def rrf_merge(self, results1, results2, top_k, k=60):
        """Merge results using Reciprocal Rank Fusion."""
        scores = {}

        for rank, r in enumerate(results1):
            scores[r.id] = scores.get(r.id, 0) + 1 / (k + rank + 1)

        for rank, r in enumerate(results2):
            scores[r.id] = scores.get(r.id, 0) + 1 / (k + rank + 1)

        sorted_ids = sorted(scores, key=scores.get, reverse=True)
        return sorted_ids[:top_k]
```

---

## См. также

- [RAG Patterns](./02-rag-patterns.md) — паттерны Retrieval-Augmented Generation с использованием векторного поиска
- [Indexing](../06-databases/01-indexing.md) — индексирование данных и оптимизация поиска

---

## На интервью

### Типичные вопросы

1. **How does vector search work?**
   - Embed query to vector
   - Find nearest neighbors by similarity metric
   - Return top-k most similar

2. **Cosine vs Euclidean similarity?**
   - Cosine: angle between vectors, [-1, 1]
   - Euclidean: distance, [0, inf)
   - Cosine preferred for text (magnitude-invariant)

3. **HNSW vs IVF?**
   - HNSW: graph-based, faster search, more memory
   - IVF: clustering-based, tunable recall/speed
   - HNSW typically preferred for quality

4. **How to handle updates?**
   - Most DBs support upsert (insert or update)
   - Delete + insert for full reindex
   - Some support incremental updates

5. **Scaling strategies?**
   - Sharding by namespace/tenant
   - Replicas for read scaling
   - Approximate search (HNSW, IVF)

6. **When pgvector vs dedicated vector DB?**
   - pgvector: small scale, existing Postgres, joins needed
   - Dedicated: large scale, performance-critical, specialized features

---

[← Назад к списку тем](README.md)
