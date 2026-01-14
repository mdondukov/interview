# 15. AI Systems Design

[← Назад к списку тем](README.md)

---

## Обзор

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AI System Design Challenges                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Уникальные аспекты AI-систем:                                      │
│                                                                     │
│  1. Latency                                                         │
│     - LLM inference: 500ms-30s vs traditional APIs: <100ms          │
│     - Streaming responses, async processing                         │
│                                                                     │
│  2. Cost                                                            │
│     - Pay per token (input/output)                                  │
│     - GPT-4: ~$30/1M tokens vs DB query: ~$0.0001                   │
│     - Caching critical for cost control                             │
│                                                                     │
│  3. Non-determinism                                                 │
│     - Same input → different outputs                                │
│     - Testing, debugging complexity                                 │
│     - Need for guardrails, validation                               │
│                                                                     │
│  4. Rate Limits                                                     │
│     - Provider quotas (RPM, TPM)                                    │
│     - Queuing, backpressure required                                │
│                                                                     │
│  5. Evaluation                                                      │
│     - No ground truth for many tasks                                │
│     - Human evaluation, automated metrics                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Система 1: AI Chatbot Platform

### Требования

**Функциональные:**
- Multi-turn conversational AI
- Context-aware responses
- Tool/function calling (search, calculations, APIs)
- File upload and analysis
- Conversation history

**Нефункциональные:**
- Latency: < 200ms time-to-first-token (streaming)
- Availability: 99.9%
- Scale: 100K DAU, 1M messages/day
- Security: PII handling, content moderation

### Capacity Estimation

```
Users: 100K DAU
Messages: 10 msgs/user/day = 1M messages/day
Average conversation: 5 turns
Avg tokens per message: 500 input + 200 output

Daily tokens: 1M × (500 + 200) = 700M tokens
Cost (GPT-4o): 700M × $0.005/1K = ~$3,500/day

Storage:
- Message: ~2KB (text + metadata)
- 1M × 2KB = 2GB/day
- 1 year = 730GB

Peak RPS: 1M / 86400 × 3 (peak factor) = ~35 RPS
```

### High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌──────────┐    ┌──────────────┐    ┌─────────────┐              │
│   │  Client  │───▶│  API Gateway │───▶│   Chat      │              │
│   │  (Web)   │◀───│  (Auth, RL)  │◀───│   Service   │              │
│   └──────────┘    └──────────────┘    └─────────────┘              │
│        │                                     │                      │
│        │ WebSocket/SSE                       │                      │
│        ▼                                     ▼                      │
│   ┌──────────┐                        ┌─────────────┐              │
│   │ Streaming│                        │   Context   │              │
│   │  Server  │                        │   Manager   │              │
│   └──────────┘                        └─────────────┘              │
│                                              │                      │
│                                              ▼                      │
│   ┌──────────┐    ┌──────────────┐    ┌─────────────┐              │
│   │  Cache   │◀──▶│  LLM Router  │───▶│  Provider   │              │
│   │ (Redis)  │    │              │    │   APIs      │              │
│   └──────────┘    └──────────────┘    └─────────────┘              │
│        │                 │                                          │
│        │                 ▼                                          │
│        │          ┌─────────────┐    ┌─────────────┐              │
│        │          │   Tool      │───▶│  External   │              │
│        │          │   Executor  │    │   APIs      │              │
│        │          └─────────────┘    └─────────────┘              │
│        │                                                            │
│        ▼                                                            │
│   ┌──────────┐    ┌──────────────┐                                 │
│   │  History │    │  PostgreSQL  │                                 │
│   │   DB     │    │  (metadata)  │                                 │
│   └──────────┘    └──────────────┘                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### API Design

```yaml
# Chat API
POST /v1/chat/completions
{
  "conversation_id": "uuid",
  "message": "string",
  "stream": true,
  "tools": ["search", "calculator"]
}

Response (streaming):
data: {"type": "start", "message_id": "uuid"}
data: {"type": "content", "delta": "Hello"}
data: {"type": "content", "delta": " world"}
data: {"type": "tool_call", "name": "search", "args": {...}}
data: {"type": "tool_result", "result": {...}}
data: {"type": "done", "usage": {"prompt_tokens": 100, "completion_tokens": 50}}

# Conversation Management
GET /v1/conversations
GET /v1/conversations/{id}
DELETE /v1/conversations/{id}

# File Upload
POST /v1/files
{
  "file": "base64...",
  "purpose": "chat"
}
```

### Data Model

```sql
-- Conversations
CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    title VARCHAR(255),
    model VARCHAR(50) DEFAULT 'gpt-4o',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Messages
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    conversation_id UUID REFERENCES conversations(id),
    role VARCHAR(20) NOT NULL, -- user, assistant, system, tool
    content TEXT,
    tool_calls JSONB,
    tool_call_id VARCHAR(100),
    tokens_used INTEGER,
    latency_ms INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for conversation history
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at);
```

### Deep Dive: Context Management

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Context Window Management                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Problem: LLM context limits (128K tokens)                          │
│  Long conversations → context overflow                              │
│                                                                     │
│  Strategy 1: Sliding Window                                         │
│  ┌─────────────────────────────────────────────┐                   │
│  │ [System] [Last N messages] [Current message] │                   │
│  └─────────────────────────────────────────────┘                   │
│  + Simple, predictable                                              │
│  - Loses early context                                              │
│                                                                     │
│  Strategy 2: Summarization                                          │
│  ┌─────────────────────────────────────────────┐                   │
│  │ [System] [Summary] [Recent] [Current]        │                   │
│  └─────────────────────────────────────────────┘                   │
│  + Preserves key info                                               │
│  - Extra LLM call for summarization                                 │
│                                                                     │
│  Strategy 3: RAG over History                                       │
│  ┌─────────────────────────────────────────────┐                   │
│  │ [System] [Retrieved relevant] [Recent]       │                   │
│  └─────────────────────────────────────────────┘                   │
│  + Semantic relevance                                               │
│  - Embedding/retrieval latency                                      │
│                                                                     │
│  Hybrid approach:                                                   │
│  - Always include: system prompt, last 5 messages                   │
│  - Summarize if > 50 messages                                       │
│  - RAG for specific facts/data                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| LLM latency (seconds) | Streaming, speculative decoding, smaller models for simple queries |
| Rate limits | Queue with backpressure, multi-provider fallback, priority queues |
| Cost explosion | Response caching, model routing by complexity, token budgets |
| Context overflow | Sliding window + summarization, RAG for history |
| Provider outages | Multi-provider with automatic failover |
| Non-determinism | Temperature=0 for consistency, output validation |

### Trade-offs

**Streaming vs Batch**
- Streaming: Better UX (immediate feedback), harder to cache
- Batch: Cacheable, simpler error handling, worse UX

**Single vs Multi-provider**
- Single: Simpler, consistent behavior
- Multi: Resilience, cost optimization, complexity

---

## Система 2: RAG-based Knowledge Base

### Требования

**Функциональные:**
- Document upload (PDF, DOCX, HTML)
- Semantic search over documents
- Q&A with source citations
- Multi-tenant (company knowledge bases)

**Нефункциональные:**
- Query latency: < 3s
- Indexing: < 5 min for 100-page PDF
- Accuracy: > 90% relevant retrieval
- Scale: 1M documents, 10K queries/day

### Capacity Estimation

```
Documents: 1M total, 100 new/day
Avg document: 10 pages × 500 words = 5000 words
Chunks: 500 words/chunk → 10 chunks/doc → 10M chunks

Embeddings (1536 dim, float32):
10M × 1536 × 4 bytes = 60GB

Queries: 10K/day → 0.1 RPS average, 1 RPS peak
Each query: 1 embedding + 1 LLM call
```

### High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Document Ingestion Pipeline                                       │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│   │  Upload  │───▶│  Parser  │───▶│ Chunker  │───▶│ Embedder │    │
│   │   API    │    │ (Tika)   │    │          │    │          │    │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘    │
│                                                           │         │
│                                                           ▼         │
│   Query Pipeline                                   ┌──────────┐    │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    │  Vector  │    │
│   │  Query   │───▶│  Query   │───▶│ Retriever│◀──▶│    DB    │    │
│   │   API    │    │ Embedder │    │          │    │ (Qdrant) │    │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘    │
│        │                               │                            │
│        │                               ▼                            │
│        │                        ┌──────────┐    ┌──────────┐       │
│        │                        │ Reranker │───▶│    LLM   │       │
│        │                        │ (Cohere) │    │ Generator│       │
│        │                        └──────────┘    └──────────┘       │
│        │                                              │             │
│        │◀─────────────────────────────────────────────┘             │
│                                                                     │
│   Storage                                                           │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│   │   S3     │    │PostgreSQL│    │  Redis   │                     │
│   │ (files)  │    │(metadata)│    │ (cache)  │                     │
│   └──────────┘    └──────────┘    └──────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Model

```sql
-- Documents
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    filename VARCHAR(255),
    file_type VARCHAR(50),
    file_size BIGINT,
    s3_key VARCHAR(500),
    status VARCHAR(20), -- pending, processing, indexed, failed
    page_count INTEGER,
    word_count INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Chunks
CREATE TABLE chunks (
    id UUID PRIMARY KEY,
    document_id UUID REFERENCES documents(id),
    content TEXT,
    page_number INTEGER,
    chunk_index INTEGER,
    token_count INTEGER,
    embedding_id VARCHAR(100), -- ID in vector DB
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Queries (for analytics)
CREATE TABLE queries (
    id UUID PRIMARY KEY,
    tenant_id UUID,
    query_text TEXT,
    retrieved_chunk_ids UUID[],
    response TEXT,
    latency_ms INTEGER,
    feedback INTEGER, -- 1=good, -1=bad
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Deep Dive: Chunking Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Chunking Strategies                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Fixed Size                                                      │
│  ┌────────┬────────┬────────┬────────┐                             │
│  │ 500    │ 500    │ 500    │ 500    │ tokens                      │
│  └────────┴────────┴────────┴────────┘                             │
│  + Simple, predictable                                              │
│  - Breaks mid-sentence, loses context                               │
│                                                                     │
│  2. Recursive Character Split                                       │
│  Split by: \n\n → \n → sentence → word                              │
│  + Respects document structure                                      │
│  + Variable size within bounds                                      │
│                                                                     │
│  3. Semantic Chunking                                               │
│  ┌──────────────┐ ┌─────────────────┐ ┌───────────┐                │
│  │  Paragraph 1 │ │   Paragraph 2   │ │ Table     │                │
│  │  (topic A)   │ │   (topic A)     │ │ (data)    │                │
│  └──────────────┘ └─────────────────┘ └───────────┘                │
│  Merge similar → Split different                                    │
│  + Best semantic coherence                                          │
│  - Requires embedding similarity check                              │
│                                                                     │
│  4. Document-aware                                                  │
│  - Headers → metadata                                               │
│  - Tables → structured extraction                                   │
│  - Code blocks → preserve formatting                                │
│                                                                     │
│  Best practice: Recursive + overlap (10-20%)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Deep Dive: Retrieval Optimization

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Retrieval Pipeline                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Query: "What is the refund policy for premium users?"              │
│                                                                     │
│  Step 1: Query Expansion                                            │
│  ┌─────────────────────────────────────────────┐                   │
│  │ Original: "refund policy premium users"      │                   │
│  │ Expanded: ["refund", "return", "cancel",     │                   │
│  │            "premium", "subscription", ...]   │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  Step 2: Hybrid Search                                              │
│  ┌──────────────┐    ┌──────────────┐                              │
│  │   Vector     │    │   Keyword    │                              │
│  │   Search     │    │   (BM25)     │                              │
│  │   k=20       │    │   k=20       │                              │
│  └──────────────┘    └──────────────┘                              │
│         │                   │                                       │
│         └───────┬───────────┘                                       │
│                 ▼                                                    │
│  Step 3: Reciprocal Rank Fusion                                     │
│  score = Σ 1/(k + rank_i) for each retriever                        │
│                                                                     │
│  Step 4: Reranking (Cross-encoder)                                  │
│  ┌─────────────────────────────────────────────┐                   │
│  │ Cohere Rerank / BGE Reranker                 │                   │
│  │ Input: (query, candidate) pairs              │                   │
│  │ Output: relevance scores                     │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  Step 5: Final top-K selection                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Slow document indexing | Async processing, parallel chunking/embedding |
| Irrelevant retrieval | Hybrid search, reranking, metadata filtering |
| Hallucination | Source citations, confidence scores, fact-checking |
| Large documents | Hierarchical indexing (summary + chunks) |
| Multi-tenant isolation | Tenant-prefixed collections, row-level security |
| Embedding cost | Batch processing, caching, smaller models |

---

## Система 3: AI Agent Platform

### Требования

**Функциональные:**
- Multi-step task execution
- Tool integration (APIs, databases, code execution)
- Human-in-the-loop approval
- Audit trail of all actions

**Нефункциональные:**
- Task completion: < 5 min for typical tasks
- Reliability: Retry, rollback, checkpointing
- Security: Sandboxed execution, permission control

### High-Level Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌──────────┐    ┌──────────────┐    ┌─────────────┐              │
│   │  Client  │───▶│  Task API    │───▶│   Task      │              │
│   │          │◀───│              │◀───│   Manager   │              │
│   └──────────┘    └──────────────┘    └─────────────┘              │
│                                               │                     │
│                           ┌───────────────────┼──────────────┐      │
│                           ▼                   ▼              ▼      │
│                    ┌───────────┐       ┌───────────┐  ┌───────────┐│
│                    │  Planner  │       │  Executor │  │  Monitor  ││
│                    │   (LLM)   │       │           │  │           ││
│                    └───────────┘       └───────────┘  └───────────┘│
│                           │                   │                     │
│                           ▼                   ▼                     │
│                    ┌───────────────────────────────┐               │
│                    │         Tool Registry          │               │
│                    ├───────────┬───────────┬───────┤               │
│                    │   APIs    │   DBs     │ Code  │               │
│                    └───────────┴───────────┴───────┘               │
│                                                                     │
│   State Management                                                  │
│   ┌───────────┐    ┌───────────┐    ┌───────────┐                  │
│   │   Redis   │    │PostgreSQL │    │    S3     │                  │
│   │  (state)  │    │  (audit)  │    │ (results) │                  │
│   └───────────┘    └───────────┘    └───────────┘                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Agent Loop Pattern

```python
# ReAct (Reasoning + Acting) Pattern
class AgentLoop:
    def __init__(self, llm, tools, max_iterations=10):
        self.llm = llm
        self.tools = tools
        self.max_iterations = max_iterations

    async def run(self, task: str) -> str:
        messages = [{"role": "user", "content": task}]

        for i in range(self.max_iterations):
            # 1. Think: Get next action from LLM
            response = await self.llm.complete(
                messages=messages,
                tools=self.tools.schemas()
            )

            messages.append(response.message)

            # 2. Check if done
            if response.stop_reason == "end_turn":
                return response.content

            # 3. Act: Execute tool calls
            if response.tool_calls:
                for tool_call in response.tool_calls:
                    # Security check
                    if not self.tools.is_allowed(tool_call):
                        result = {"error": "Action not permitted"}
                    else:
                        result = await self.tools.execute(tool_call)

                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": json.dumps(result)
                    })

            # 4. Checkpoint state
            await self.checkpoint(messages)

        return "Max iterations reached"
```

### Security Considerations

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Agent Security Model                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Permission Scoping                                              │
│  ┌─────────────────────────────────────────────┐                   │
│  │ Agent A: [read_db, call_api_x]               │                   │
│  │ Agent B: [read_db, write_db, execute_code]   │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  2. Human-in-the-Loop                                               │
│  ┌─────────────────────────────────────────────┐                   │
│  │ Actions requiring approval:                  │                   │
│  │ - write_db, delete, send_email, payments     │                   │
│  │ - Anything above $X or affecting >N records  │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  3. Sandboxed Execution                                             │
│  ┌─────────────────────────────────────────────┐                   │
│  │ Code execution in:                           │                   │
│  │ - Docker containers (resource limits)        │                   │
│  │ - No network by default                      │
│  │ - Timeout enforcement                        │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
│  4. Audit Trail                                                     │
│  ┌─────────────────────────────────────────────┐                   │
│  │ Log: who, what, when, input, output, result  │                   │
│  │ Immutable storage (append-only)              │                   │
│  └─────────────────────────────────────────────┘                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **How do you handle LLM latency in user-facing apps?**
   - Streaming responses
   - Speculative UI (show loading state)
   - Async processing for long tasks
   - Smaller models for real-time, larger for quality

2. **How do you ensure RAG accuracy?**
   - Quality chunking (semantic, overlap)
   - Hybrid retrieval (vector + keyword)
   - Reranking
   - Source citations for verification

3. **How do you control AI costs?**
   - Response caching
   - Model routing by complexity
   - Token budgets per user/request
   - Batch processing where possible

4. **How do you make AI agents reliable?**
   - Checkpointing for resume
   - Retry with backoff
   - Human approval for critical actions
   - Comprehensive audit logging

5. **How do you test AI systems?**
   - Mock LLM responses for unit tests
   - Golden set evaluation
   - A/B testing prompts
   - Continuous monitoring of quality metrics

6. **What metrics would you track?**
   - Latency (p50, p99)
   - Token usage and cost
   - Error rates
   - User satisfaction (thumbs up/down)
   - Retrieval relevance scores

---

## См. также

- [Основы LLM](../04-ai-integration/00-llm-basics.md) — базовые концепции работы с большими языковыми моделями
- [Распределённый кэш](./07-distributed-cache.md) — кэширование ответов LLM для оптимизации стоимости

---

[← Назад к списку тем](README.md)
