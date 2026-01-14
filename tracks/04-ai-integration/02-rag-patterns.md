# 02. RAG Fundamentals

[← Назад к списку тем](README.md)

---

## RAG Architecture

### Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RAG Pipeline                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  INDEXING (offline):                                                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │ Documents│ → │  Chunk   │ → │  Embed   │ → │  Store   │      │
│  │          │    │          │    │          │    │ (VectorDB)│     │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│                                                                     │
│  RETRIEVAL (online):                                                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  Query   │ → │  Embed   │ → │  Search  │ → │ Top-K    │      │
│  │          │    │  Query   │    │ VectorDB │    │ Results  │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│                                                                     │
│  GENERATION:                                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                      │
│  │  Query + │ → │   LLM    │ → │ Response │                      │
│  │ Context  │    │          │    │          │                      │
│  └──────────┘    └──────────┘    └──────────┘                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Basic Implementation

```python
from openai import OpenAI
import chromadb

client = OpenAI()
db = chromadb.Client()
collection = db.create_collection("documents")

# 1. Index documents
def index_documents(documents: list[str]):
    for i, doc in enumerate(documents):
        # Get embedding
        response = client.embeddings.create(
            model="text-embedding-3-small",
            input=doc
        )
        embedding = response.data[0].embedding

        # Store in vector DB
        collection.add(
            embeddings=[embedding],
            documents=[doc],
            ids=[f"doc_{i}"]
        )

# 2. Retrieve relevant context
def retrieve(query: str, top_k: int = 3) -> list[str]:
    # Embed query
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    query_embedding = response.data[0].embedding

    # Search
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=top_k
    )
    return results["documents"][0]

# 3. Generate answer
def rag_query(query: str) -> str:
    # Retrieve context
    context_docs = retrieve(query)
    context = "\n\n".join(context_docs)

    # Generate with context
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": f"""Answer based on the provided context.
If the answer is not in the context, say "I don't know."

Context:
{context}"""},
            {"role": "user", "content": query}
        ]
    )
    return response.choices[0].message.content
```

---

## Chunking Strategies

### Fixed-Size Chunking

```python
def fixed_size_chunk(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    """Split text into fixed-size chunks with overlap."""
    chunks = []
    start = 0

    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap  # Overlap for context continuity

    return chunks

# Problems:
# - Can split mid-sentence
# - Ignores document structure
```

### Semantic Chunking

```python
import re

def semantic_chunk(text: str, max_chunk_size: int = 1000) -> list[str]:
    """Split by semantic boundaries (paragraphs, sections)."""

    # Split by paragraphs
    paragraphs = re.split(r'\n\n+', text)

    chunks = []
    current_chunk = ""

    for para in paragraphs:
        if len(current_chunk) + len(para) < max_chunk_size:
            current_chunk += para + "\n\n"
        else:
            if current_chunk:
                chunks.append(current_chunk.strip())
            current_chunk = para + "\n\n"

    if current_chunk:
        chunks.append(current_chunk.strip())

    return chunks
```

### Recursive Character Splitter

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""]  # Try each in order
)

chunks = splitter.split_text(document)

# Benefits:
# - Respects natural boundaries
# - Falls back to smaller separators if needed
```

### Document-Aware Chunking

```python
def chunk_by_structure(document: dict) -> list[dict]:
    """Chunk based on document structure (headers, sections)."""
    chunks = []

    for section in document["sections"]:
        chunk = {
            "content": section["content"],
            "metadata": {
                "title": document["title"],
                "section": section["header"],
                "page": section.get("page"),
                "source": document["source"]
            }
        }

        # If section too large, split further
        if len(chunk["content"]) > 1000:
            sub_chunks = semantic_chunk(chunk["content"])
            for i, sub in enumerate(sub_chunks):
                chunks.append({
                    "content": sub,
                    "metadata": {**chunk["metadata"], "sub_section": i}
                })
        else:
            chunks.append(chunk)

    return chunks
```

### Chunking Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Chunking Guidelines                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Chunk Size:                                                        │
│  - Too small: loses context, more retrieval noise                   │
│  - Too large: dilutes relevance, wastes context window              │
│  - Sweet spot: 200-1000 tokens depending on use case                │
│                                                                     │
│  Overlap:                                                           │
│  - 10-20% overlap helps maintain context                            │
│  - Important for semantic continuity                                │
│                                                                     │
│  Metadata:                                                          │
│  - Always store: source, page, section, date                        │
│  - Enables filtering and citation                                   │
│                                                                     │
│  Content Type:                                                      │
│  - Code: chunk by function/class                                    │
│  - Legal: chunk by clause/section                                   │
│  - Chat logs: chunk by conversation turn                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Embeddings

### Popular Embedding Models

```python
# OpenAI
from openai import OpenAI
client = OpenAI()

def openai_embed(texts: list[str]) -> list[list[float]]:
    response = client.embeddings.create(
        model="text-embedding-3-small",  # or text-embedding-3-large
        input=texts
    )
    return [e.embedding for e in response.data]

# text-embedding-3-small: 1536 dims, cheap, fast
# text-embedding-3-large: 3072 dims, better quality

# Open source - Sentence Transformers
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')  # 384 dims
embeddings = model.encode(texts)

# Other popular models:
# - all-mpnet-base-v2: better quality, slower
# - instructor-large: instruction-tuned
# - bge-large-en: BGE models from BAAI
```

### Embedding Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Embedding Models Comparison                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Model                    │ Dims  │ Speed  │ Quality │ Cost        │
│  ─────────────────────────┼───────┼────────┼─────────┼────────     │
│  text-embedding-3-small   │ 1536  │ Fast   │ Good    │ $0.02/1M    │
│  text-embedding-3-large   │ 3072  │ Medium │ Best    │ $0.13/1M    │
│  all-MiniLM-L6-v2         │ 384   │ Fast   │ OK      │ Free        │
│  all-mpnet-base-v2        │ 768   │ Medium │ Good    │ Free        │
│  bge-large-en-v1.5        │ 1024  │ Medium │ Great   │ Free        │
│  Cohere embed-v3          │ 1024  │ Fast   │ Great   │ $0.10/1M    │
│                                                                     │
│  Selection criteria:                                                │
│  - Quality needs: benchmark on your data                            │
│  - Latency: batch vs real-time                                      │
│  - Cost: volume of embeddings                                       │
│  - Privacy: self-hosted vs API                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Embedding Best Practices

```python
# 1. Batch embeddings for efficiency
def batch_embed(texts: list[str], batch_size: int = 100) -> list:
    embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        batch_emb = openai_embed(batch)
        embeddings.extend(batch_emb)
    return embeddings

# 2. Normalize embeddings for cosine similarity
import numpy as np

def normalize(embeddings: np.ndarray) -> np.ndarray:
    norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
    return embeddings / norms

# 3. Consider query vs document embeddings
# Some models (e.g., instructor) use different prompts
def embed_query(text: str) -> list[float]:
    return model.encode(f"query: {text}")

def embed_document(text: str) -> list[float]:
    return model.encode(f"passage: {text}")
```

---

## Retrieval Strategies

### Hybrid Search

```python
from rank_bm25 import BM25Okapi

class HybridRetriever:
    """Combine semantic (vector) and keyword (BM25) search."""

    def __init__(self, documents: list[str]):
        self.documents = documents

        # BM25 for keyword search
        tokenized = [doc.split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)

        # Vector store for semantic search
        self.embeddings = embed_documents(documents)

    def search(
        self,
        query: str,
        top_k: int = 5,
        alpha: float = 0.5  # Weight for semantic vs keyword
    ) -> list[tuple[str, float]]:
        # Semantic search
        query_emb = embed_query(query)
        semantic_scores = cosine_similarity([query_emb], self.embeddings)[0]

        # Keyword search
        keyword_scores = self.bm25.get_scores(query.split())

        # Normalize scores to [0, 1]
        semantic_norm = (semantic_scores - semantic_scores.min()) / (semantic_scores.max() - semantic_scores.min() + 1e-6)
        keyword_norm = (keyword_scores - keyword_scores.min()) / (keyword_scores.max() - keyword_scores.min() + 1e-6)

        # Combine scores
        combined = alpha * semantic_norm + (1 - alpha) * keyword_norm

        # Return top-k
        top_indices = combined.argsort()[-top_k:][::-1]
        return [(self.documents[i], combined[i]) for i in top_indices]
```

### Reranking

```python
from sentence_transformers import CrossEncoder

class Reranker:
    """Rerank retrieved documents for better relevance."""

    def __init__(self):
        self.model = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

    def rerank(
        self,
        query: str,
        documents: list[str],
        top_k: int = 3
    ) -> list[tuple[str, float]]:
        # Cross-encoder scores query-document pairs
        pairs = [[query, doc] for doc in documents]
        scores = self.model.predict(pairs)

        # Sort by score
        ranked = sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
        return ranked[:top_k]

# Pipeline: retrieve 20 candidates, rerank to top 5
candidates = vector_search(query, top_k=20)
final = reranker.rerank(query, candidates, top_k=5)
```

### Query Transformation

```python
def expand_query(query: str) -> list[str]:
    """Generate multiple query variations for better recall."""
    prompt = f"""Generate 3 alternative ways to ask this question:
Original: {query}

Output as JSON array of strings."""

    response = llm.generate(prompt)
    variations = json.loads(response)
    return [query] + variations

def decompose_query(query: str) -> list[str]:
    """Break complex query into sub-queries."""
    prompt = f"""Break this complex question into simpler sub-questions:
Question: {query}

Output as JSON array of strings."""

    response = llm.generate(prompt)
    return json.loads(response)

# Use multiple queries for retrieval
def multi_query_retrieve(query: str, top_k: int = 5) -> list[str]:
    queries = expand_query(query)
    all_results = []

    for q in queries:
        results = vector_search(q, top_k=top_k)
        all_results.extend(results)

    # Deduplicate and rank
    return deduplicate_and_rank(all_results)
```

---

## Advanced RAG Patterns

### Parent Document Retriever

```python
class ParentDocumentRetriever:
    """
    Store small chunks for retrieval,
    return larger parent chunks for context.
    """

    def __init__(self):
        self.chunk_store = {}  # small chunks
        self.parent_store = {}  # larger context
        self.chunk_to_parent = {}  # mapping

    def add_document(self, doc: str, doc_id: str):
        # Create large parent chunks
        parents = chunk(doc, size=2000)
        for i, parent in enumerate(parents):
            parent_id = f"{doc_id}_parent_{i}"
            self.parent_store[parent_id] = parent

            # Create smaller child chunks
            children = chunk(parent, size=400)
            for j, child in enumerate(children):
                child_id = f"{doc_id}_child_{i}_{j}"
                self.chunk_store[child_id] = child
                self.chunk_to_parent[child_id] = parent_id

                # Index child chunk
                vector_store.add(child, child_id)

    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        # Search small chunks
        child_ids = vector_store.search(query, top_k=top_k * 2)

        # Return parent chunks (larger context)
        parent_ids = set(self.chunk_to_parent[cid] for cid in child_ids)
        return [self.parent_store[pid] for pid in list(parent_ids)[:top_k]]
```

### Self-RAG (Self-Reflective RAG)

```python
def self_rag(query: str) -> str:
    """RAG with self-reflection on retrieval quality."""

    # Step 1: Decide if retrieval is needed
    need_retrieval = llm.generate(f"""
Query: {query}
Do you need external information to answer this? (yes/no)
""").strip().lower() == "yes"

    if not need_retrieval:
        return llm.generate(query)

    # Step 2: Retrieve and evaluate relevance
    documents = retrieve(query, top_k=5)

    relevant_docs = []
    for doc in documents:
        is_relevant = llm.generate(f"""
Query: {query}
Document: {doc}
Is this document relevant to the query? (yes/no)
""").strip().lower() == "yes"

        if is_relevant:
            relevant_docs.append(doc)

    # Step 3: Generate with relevant context
    context = "\n\n".join(relevant_docs)
    response = llm.generate(f"""
Context: {context}
Query: {query}
Answer based on the context:
""")

    # Step 4: Verify answer is supported
    is_supported = llm.generate(f"""
Context: {context}
Answer: {response}
Is this answer fully supported by the context? (yes/partially/no)
""")

    if "no" in is_supported.lower():
        response += "\n\nNote: This answer may not be fully supported by the available information."

    return response
```

---

## На интервью

### Типичные вопросы

1. **What is RAG?**
   - Retrieval-Augmented Generation
   - Retrieve relevant docs, add to prompt context
   - Grounds LLM responses in facts

2. **How to choose chunk size?**
   - Too small: loses context
   - Too large: dilutes relevance
   - 200-1000 tokens typical
   - Experiment with your data

3. **Semantic vs keyword search?**
   - Semantic: meaning-based, handles synonyms
   - Keyword: exact match, good for names/codes
   - Hybrid: combine both for best results

4. **What is reranking?**
   - Second-stage ranking of retrieved docs
   - Cross-encoder more accurate than bi-encoder
   - Retrieve many, rerank to few

5. **How to evaluate RAG?**
   - Retrieval: precision, recall, MRR
   - Generation: faithfulness, relevance
   - End-to-end: answer correctness

6. **When RAG vs fine-tuning?**
   - RAG: dynamic knowledge, citations needed
   - Fine-tuning: behavior change, style
   - Often combine both

---

[← Назад к списку тем](README.md)
