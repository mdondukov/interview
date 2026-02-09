# 15. AI Systems Design

Развёрнутые вопросы и ответы про проектирование систем с AI: архитектура чатбот-платформ, RAG, embeddings, AI agents, LLM serving, оптимизация затрат, масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [14-monitoring-alerting](./14-monitoring-alerting.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**LLM (Large Language Model)** — нейросетевая модель, обученная на огромном количестве текста из интернета, книг и других источников. LLM способна выполнять разные задачи с текстом через естественный язык: переводить, резюмировать, отвечать на вопросы, писать код, создавать креативный контент. Примеры включают OpenAI GPT серию, Google Gemini, Meta Llama и др. LLM работают путём предсказания следующего слова на основе предыдущих слов, используя архитектуру трансформер.

**Inference** — процесс получения ответа от LLM для нового пользовательского запроса. Это самая дорогая (в плане вычислительных ресурсов и денег) и медленная часть работы AI системы. Inference требует передачи запроса через нейросеть, что может занять от 500 миллисекунд до десятков секунд, в зависимости от длины ответа и модели. Оптимизация inference (кеширование, батчинг, модель-routing) критична для экономики AI сервиса.

**Token** — единица текста, которую обрабатывает LLM, обычно это слово или часть слова (подслово). Для системы важно понимать токены, так как стоимость API обычно считается в токенах (например, $0.5 за 1M входных токенов). Длина в токенах также влияет на latency: больше токенов = дольше время inference. Примерно 1 токен ≈ 4 символа или 0.75 слова на английском языке.

**Prompt** — текст запроса к LLM, который содержит инструкцию и контекст. Хороший prompt специфичен, содержит примеры (few-shot learning), и объясняет что именно нужно сделать. Качество prompt'а критично влияет на качество ответа: два очень похожих prompt'а могут привести к совершенно разным результатам. Техника prompt-engineering занимается оптимизацией prompt'ов для достижения лучших результатов.

**RAG (Retrieval Augmented Generation)** — техника поиска релевантных документов в приватной базе знаний перед отправкой запроса к LLM. Это позволяет LLM ответить на вопрос о приватных данных компании (например, документы, коды, внутренние системы), которых нет в его обучающих данных. RAG работает путём преобразования вопроса в вектор, поиска похожих документов в векторной БД, и добавления найденных документов в контекст для LLM.

**Embedding** — преобразование текста в вектор чисел фиксированной размерности, которое кодирует семантическое значение текста. Embeddings позволяют сравнивать похожесть текстов используя операции с векторами (например, косинусное расстояние). Embedding модели обучены так, что похожий текст имеет похожие embeddings. Это основа RAG: вопрос кодируется в embedding, и затем поиск находит документы с похожими embeddings.

**Temperature** — параметр LLM, который контролирует "случайность" ответа (0 = детерминированный, 1 = случайный, обычно выше чем 1 запрещено). Низкая temperature (например, 0.1) делает модель более консервативной и сосредоточенной на самых вероятных ответах, подходящей для точных задач (вопрос-ответ, кодирование). Высокая temperature (например, 0.9) делает модель более творческой и разнообразной, подходящей для креативных задач (написание рассказа, генерирование идей).

**Hallucination** — явление, когда LLM выдумывает информацию (факты, ссылки, цифры) вместо того чтобы признать что она не знает ответ. Это опасна для критичных систем (например, в медицине или праве), так как пользователь может поверить неправильной информации. Решение включает: использование RAG для предоставления точных источников, промpting модель к ответу "я не знаю", и валидацию выходов системы против известных источников.

**Context Window** — максимальное количество токенов, которое LLM может обработать в одном запросе. Это технический лимит модели: запрос + контекст + ответ должны уместиться в context window. Например, GPT-4 имеет context window 8K или 128K токенов. Context window ограничивает длину документов, которые можно передать в RAG, поэтому большие документы нужно разбивать (chunking) на меньшие куски.

**Rate Limiting** — ограничение количества запросов (и токенов) к LLM API, которое накладывает провайдер. Это может быть лимит в секунду, в минуту или в день. Провайдеры ограничивают нагрузку для контроля стоимости и стабильности. Приложение должно обрабатывать rate limiting путём буферизации запросов и реализации retry logic с exponential backoff.

---

## Вопросы и разборы

### 1. Какие уникальные вызовы возникают при проектировании AI-систем?

**Зачем спрашивают.** AI-системы отличаются от traditional backend резко: нестабильность, высокая стоимость, непредсказуемая латенция. Интервьюер проверяет осознание этих различий перед тем, как углубляться в архитектуру.

**Короткий ответ.** AI-системы имеют пять ключевых отличий: экстремальная латенция (500ms-30s), высокая стоимость (платить за токены), недетерминизм (один input → разные outputs), rate limits провайдера, сложная оценка качества без ground truth.

**Детальный разбор.**

```
┌─────────────────────────────────────────────────────────────────────┐
│              Уникальные вызовы AI-систем vs Traditional             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. LATENCY                                                         │
│     Traditional API: < 100ms                                        │
│     LLM inference:   500ms - 30s (зависит от длины ответа)          │
│     └─ Streaming необходим для UX                                   │
│     └─ Speculative UI (показывать skeleton раньше)                  │
│                                                                     │
│  2. COST                                                            │
│     Traditional DB query: ~$0.0001 на операцию                      │
│     LLM call: $0.005 за 1M входных токенов                          │
│     1M сообщений/день: ~$3500/день без оптимизации                  │
│     └─ Caching critical                                             │
│     └─ Model routing by complexity                                  │
│     └─ Token budgets                                                │
│                                                                     │
│  3. NON-DETERMINISM                                                 │
│     Один вход → N разных выходов                                    │
│     └─ Невозможно тестировать как traditional API                   │
│     └─ Нужны guardrails и validation                                │
│     └─ Temperature для контроля: 0 = детерминизм                    │
│                                                                     │
│  4. RATE LIMITS                                                     │
│     OpenAI GPT-4: 10K RPM (requests/min), 2M TPM (tokens/min)        │
│     └─ Простое обчисление volume может быть > лимитов               │
│     └─ Queuing, backpressure, multi-provider failover               │
│                                                                     │
│  5. QUALITY EVALUATION                                              │
│     Нет ground truth для многих задач                                │
│     └─ Human evaluation + automated metrics                         │
│     └─ A/B testing на реальных пользователях                        │
│     └─ Continuous monitoring                                        │
│                                                                     │
│  6. SECURITY & SAFETY                                               │
│     PII в контексте + outputs                                       │
│     Prompt injection attacks                                        │
│     Hallucination может привести к неправильным действиям           │
│     └─ Content moderation                                           │
│     └─ Source validation                                            │
│     └─ Confidence scores                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Пример.**

Расчёт для сервиса с 100K DAU, 10 сообщений/день:

```
Daily volume:
- Users: 100K DAU
- Messages: 1M/day
- Avg tokens/message: 500 input + 200 output = 700
- Total tokens: 700M/day

Стоимость (GPT-4o: $3/$15 per 1M tokens):
- Input: 500M × $3/1M = $1500
- Output: 200M × $15/1M = $3000
- Total: $4500/day = $1.64M/year

Влияние на latency:
- Inference: 2-5 seconds per message
- Peak RPS: 1M / 86400 × 3 = ~35 RPS
- Must support streaming for UX (TTFT < 200ms)

Лимиты провайдера:
- 700M tokens/day = 486K tokens/min average
- OpenAI limit: 2M TPM (plenty room)
- But at peak: 1.5M tokens/min possible → near limit

Отслеживание качества:
- Can't test deterministically
- Need user feedback (thumbs up/down)
- Monitor: hallucination rate, accuracy, cost per successful query
```

**Типичные ошибки.**

- Думать, что AI system дизайн = как-то запустить LLM API. Нет управления стоимостью, нет обработки латенции, нет graceful degradation.
- Игнорировать rate limits до production → production fail.
- Использовать temperature=default (0.7) для consistency → недетерминизм ломает тесты и кэширование.
- Не планировать fallback на более дешёвые модели для simple queries.

**На интервью.**

- Объясни, почему AI system expensive и how это отличается от traditional backend.
- Упомяни streaming и speculative UI как necessary для хорошего UX.
- Говори о cost optimization раньше, чем о архитектуре — это показывает приоритизацию.
- Уточняющий вопрос: «Какие метрики ты бы трекил?» — не только latency/errors, но и cost/token, hallucination rate.

---

### 2. Как спроектировать AI Chatbot Platform для 100K DAU?

**Зачем спрашивают.** Chatbot — common use case для AI. Требует управления context, streaming, tool calling, cost optimization, conversation history. Real system design problem.

**Короткий ответ.** Chatbot Platform = API Gateway (auth, RL) → Chat Service (context mgmt, LLM routing) → LLM Provider. Нужно streaming для UX, caching для cost, sliding window/summarization для context, multi-provider for resilience.

**Детальный разбор.**

Архитектура:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Requests                            │
└────────────────┬─────────────────────────────────────────────────────┘
                 │
                 ▼
    ┌────────────────────────────┐
    │    API Gateway             │
    │  - Authentication          │
    │  - Rate Limiting (100 req/min per user)  │
    │  - Request validation      │
    └────────────┬───────────────┘
                 │ WebSocket/SSE for streaming
                 ▼
    ┌────────────────────────────┐
    │    Chat Service            │
    │  - Conversation mgmt       │
    │  - Context window mgmt     │
    │  - Message persistence     │
    └────────┬───────────────────┘
             │
    ┌────────┴──────────────┐
    │                       │
    ▼                       ▼
┌──────────────┐   ┌─────────────────┐
│ Context      │   │ LLM Router      │
│ Manager      │   │ - Model select  │
│              │   │ - Load balance  │
│ Strategies:  │   │ - Failover      │
│ 1. Sliding   │   │ - Cost routing  │
│    window    │   │                 │
│ 2. Summary   │   ▼                 │
│ 3. RAG       │ ┌────────────────────────────┐
│              │ │  Provider APIs             │
│              │ │  - OpenAI (GPT-4, GPT-4o)  │
│              │ │  - Anthropic (Claude)      │
│              │ │  - Azure OpenAI            │
│              │ │  - Self-hosted (LLaMA)     │
│              │ └────────┬───────────────────┘
│              │          │
│              │          ▼
│              │ ┌────────────────────┐
│              │ │  Tool Executor     │
│              │ │  - Web search      │
│              │ │  - Calculator      │
│              │ │  - External APIs   │
│              │ │  - Code execution  │
│              │ └────────────────────┘
│              │
└──────────────┘
    │
    ├──────────────────────┐
    │                      │
    ▼                      ▼
┌──────────┐      ┌─────────────┐
│  Cache   │      │  History    │
│ (Redis)  │      │  Database   │
│ TTL:1h   │      │ (PostgreSQL)│
│          │      │             │
│ Store:   │      │ - Convos    │
│ - Full   │      │ - Messages  │
│   responses    │ - Tokens    │
│ - Context      │             │
│   summaries    │             │
└──────────┘      └─────────────┘
```

Оценка ёмкости:

```
Input: 100K DAU, 10 messages/day, 5 turn-conversations
- Daily messages: 1M
- Peak RPS: 1M / 86400 × 3 = ~35 RPS
- Avg latency to first token (streaming): 200ms
- Full latency (complete response): 2-5 seconds

Хранилище:
- Message: ~2KB (text + metadata)
- 1M × 2KB = 2GB/day
- 1 year: 730GB
- Assume 30-day retention: 60GB hot storage

Стоимость:
- Input tokens: 1M × 500 = 500M/day × $3/1M = $1500/day
- Output tokens: 1M × 200 = 200M/day × $15/1M = $3000/day
- Total: $4500/day ≈ $1.6M/year
```

Дизайн API:

```yaml
# Создать/продолжить беседу
POST /v1/chat/completions
{
  "conversation_id": "uuid or null for new",
  "message": "user message",
  "stream": true,
  "model": "gpt-4o",  # optional, auto-select if omitted
  "tools": ["search", "calculator"],  # optional
  "temperature": 0.7,  # for consistency: use 0
  "max_tokens": 2000
}

# Формат потокового ответа
data: {"type": "start", "message_id": "uuid", "conversation_id": "uuid"}
data: {"type": "content", "delta": "Привет"}
data: {"type": "content", "delta": " мир"}
data: {"type": "tool_call", "name": "search", "args": {"query": "..."}}
data: {"type": "tool_result", "result": {...}}
data: {"type": "done", "usage": {"prompt_tokens": 100, "completion_tokens": 50}}

# Получить беседу
GET /v1/conversations/{id}

# Список бесед
GET /v1/conversations?limit=20&offset=0

# Удалить беседу
DELETE /v1/conversations/{id}

# Загрузка файла для анализа
POST /v1/files
{
  "file": "base64...",
  "purpose": "chat"
}
```

Модель данных:

```sql
-- Conversations (indexed by user_id for listing)
CREATE TABLE conversations (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    title VARCHAR(255),
    model VARCHAR(50) DEFAULT 'gpt-4o',
    system_prompt TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_conversations_user ON conversations(user_id, updated_at DESC);

-- Messages (partitioned by conversation for fast retrieval)
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    role VARCHAR(20) NOT NULL,  -- user, assistant, system, tool
    content TEXT,
    tool_calls JSONB,  -- { "name": "...", "args": {...}, "id": "..." }
    tool_call_id VARCHAR(100),
    tokens_used INTEGER,
    latency_ms INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_messages_conversation ON messages(
    conversation_id,
    created_at DESC
);

-- Token tracking for cost optimization
CREATE TABLE token_usage (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    conversation_id UUID,
    input_tokens INTEGER,
    output_tokens INTEGER,
    cost_cents INTEGER,
    model VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_token_usage_user_date ON token_usage(
    user_id,
    DATE(created_at)
);
```

Context window management:

```
Проблема: контекстное окно LLM ограничено (128K токенов для GPT-4o)
Длинные разговоры → переполнение → ошибки или обрезание контекста

Сравнение стратегий:

1. СКОЛЬЗЯЩЕЕ ОКНО (самое простое)
   ┌─────────────────────────────────┐
   │ [Система] [Последних 10 сообщ] [Текущ]│
   └─────────────────────────────────┘
   ✓ Предсказуемо, простая реализация
   ✗ Теряет ранний контекст, тема может запутаться

2. СУММАРИЗАЦИЯ (качество)
   ┌──────────────────────────────────────┐
   │ [Система] [Краткое] [Последних 5] [Текущ]│
   └──────────────────────────────────────┘
   ✓ Сохраняет ключевую информацию в разговоре
   ✗ Дополнительный вызов LLM (стоимость + latency)
   └─ Вызвать LLM для суммаризации каждые 50 сообщений

3. HYBRID (recommended)
   Always include:
   - System prompt (tokens: ~100)
   - Last 3 messages (recent context)
   - Current message

   If > 20 messages total:
   - Summarize older messages into 1 summary

   If user asks about old message:
   - BM25 search in conversation history
   - Retrieve relevant chunks, include them

4. RAG OVER HISTORY (most sophisticated)
   ┌──────────────────────────────────────────────┐
   │ [System] [Retrieved relevant] [Recent] [New] │
   └──────────────────────────────────────────────┘
   ✓ Semantic search for relevant context
   ✗ Embedding call adds latency (~100ms)
   └─ Use vector DB (Qdrant) with embeddings
```

Tool calling flow:

```
┌─────────────────┐
│  User message   │
│ "search for..." │
└────────┬────────┘
         │
         ▼
    LLM decides: need search tool

    Tool schema: {
      "name": "search",
      "description": "Search the web",
      "parameters": {
        "query": "string",
        "num_results": "int"
      }
    }

    LLM outputs:
    {
      "type": "tool_call",
      "id": "call_123",
      "name": "search",
      "arguments": "{\"query\": \"...\"}"
    }

    System executes tool → result

    Append to messages:
    {
      "role": "tool",
      "tool_call_id": "call_123",
      "content": "{\"results\": [...]}"
    }

    LLM continues with tool result

    Outputs final response to user
```

**Типичные ошибки.**

- Не streaming → UX ломается, 2-5 segundos of blank screen.
- Context не managed → overflow errors when conversations are long.
- Single provider → outages cause total system failure.
- No cost tracking → bill shock when popular.
- Temperature=default → non-determinism breaks response caching.
- No tool execution timeout → runaway calls to external APIs.

**На интервью.**

- Объясни, почему streaming необходим для chatbot UX.
- Speak about context management strategies and tradeoffs.
- Упомяни multi-provider failover для resilience.
- Уточняющий вопрос: «How would you cost-optimize this?» → model routing, caching, smaller models for simple queries.
- Уточняющий вопрос: «How would you handle 10M DAU?» → horizontal scaling of chat service, vector DB for context retrieval, more aggressive caching.

---

### 3. Что такое RAG и как его спроектировать для знаниевой базы на 1M документов?

**Зачем спрашивают.** RAG (Retrieval Augmented Generation) — essential pattern для integrating external knowledge с LLM. Требует понимания embedding, retrieval, reranking, и как всё это work together для accuracy.

**Короткий ответ.** RAG = берём документ, chunking, embeddings в vector DB, потом при запросе — retrieve top-K relevant chunks by semantic similarity, pass их в LLM context, LLM generates answer with citations. Accuracy depends на chunking quality, retrieval strategy (hybrid search + reranking), и LLM generation.

**Детальный разбор.**

RAG Pipeline:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DOCUMENT INGESTION PIPELINE                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   PDF/DOCX/HTML              Tika Parser              Chunks         │
│   ┌─────────────┐            ┌─────────────┐         ┌──────────┐   │
│   │ Document 1  │───text────▶│ Extract     │────────▶│ Chunk 1  │   │
│   │ (100 pages) │            │ text+meta   │         │ (500     │   │
│   │             │            │             │         │  tokens) │   │
│   └─────────────┘            └─────────────┘         │          │   │
│                                                       │ Chunk 2  │   │
│                                                       │ (500     │   │
│                                                       │  tokens) │   │
│                                                       └────┬─────┘   │
│                                                            │          │
│                                                            ▼          │
│                                                    ┌──────────────┐  │
│                                                    │ Embeddings   │  │
│                                                    │ (1536 dim)   │  │
│                                                    │ via OpenAI   │  │
│                                                    │ text-embed-  │  │
│                                                    │ ding-3-large│  │
│                                                    └──────┬───────┘  │
│                                                           │          │
│                                                           ▼          │
│                                                    ┌──────────────┐  │
│                                                    │ Vector DB    │  │
│                                                    │ (Qdrant)     │  │
│                                                    │ Store:       │  │
│                                                    │ - Vector     │  │
│                                                    │ - Metadata   │  │
│                                                    │ - Text       │  │
│                                                    └──────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                      QUERY PIPELINE                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User Query              Query Encoding            Vector Search     │
│  ┌──────────────┐        ┌──────────────┐        ┌─────────────┐   │
│  │ "What is the │───────▶│ Embed query  │───────▶│ k-NN search │   │
│  │  refund      │        │ (same model) │        │ in vector DB│   │
│  │  policy?"    │        └──────────────┘        │             │   │
│  └──────────────┘                                │ Top-20      │   │
│                                                  │ candidates  │   │
│                                                  └────┬────────┘   │
│                                                       │             │
│                                                       ▼             │
│                                                  ┌──────────────┐  │
│                                                  │ Hybrid Search│  │
│                                                  │              │  │
│                                                  │ Combine:     │  │
│                                                  │ - Vector     │  │
│                                                  │ - BM25       │  │
│                                                  │              │  │
│                                                  │ Reciprocal   │  │
│                                                  │ Rank Fusion  │  │
│                                                  └────┬────────┘   │
│                                                       │             │
│                                                       ▼             │
│                                                  ┌──────────────┐  │
│                                                  │ Reranker     │  │
│                                                  │ (Cohere /    │  │
│                                                  │  BGE)        │  │
│                                                  │              │  │
│                                                  │ (query,      │  │
│                                                  │  doc_i) →    │  │
│                                                  │  score_i     │  │
│                                                  └────┬────────┘   │
│                                                       │             │
│                                                       ▼             │
│                                                  ┌──────────────┐  │
│                                                  │ Top-5        │  │
│                                                  │ Relevant     │  │
│                                                  │ Chunks       │  │
│                                                  └────┬────────┘   │
│                                                       │             │
│                                                       ▼             │
│                                                  ┌──────────────┐  │
│                                                  │ Prompt       │  │
│                                                  │ Assembly     │  │
│                                                  │              │  │
│                                                  │ [System]     │  │
│                                                  │ [Retrieved]  │  │
│                                                  │ [Query]      │  │
│                                                  └────┬────────┘   │
│                                                       │             │
│                                                       ▼             │
│                                                  ┌──────────────┐  │
│                                                  │ LLM Answer   │  │
│                                                  │ Generation   │  │
│                                                  │ + Citations  │  │
│                                                  └──────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

Оценка пропускной способности:

```
Документы: 1M всего, 100 новых/день
Средний документ: 10 страниц × 500 слов = 5K слов

Разбиение на чанки:
- Стратегия: рекурсивное разбиение с перекрытием 10%
- Размер чанка: ~500 токенов
- Чанков на документ: ~10
- Всего чанков: 10M

Embeddings (OpenAI text-embedding-3-large):
- Размер: 1536 (float32)
- Хранилище: 10M × 1536 × 4 байта = 60GB
- Ingestion: 100 docs/день × 10 chunks × embedding call
  = 1000 embeddings/день
  = ~$0.02/день на embeddings

Трафик запросов:
- 10K запросов/день = 0.1 RPS в среднем
- Пик: 1 RPS
- Каждый запрос: 1 embedding + search + rerank + LLM
  Total latency: 100ms (embed) + 50ms (search) + 50ms (rerank)
                + 2000ms (LLM) = ~2.2 сек

Стоимость:
- Embeddings (ingestion): ~$6/month
- Embeddings (queries): ~$1.5/month
- LLM calls: 10K × $0.02 (avg) = $200/month
- Total: ~$210/month (scales with usage)
```

Chunking strategies:

```
Strategy 1: FIXED SIZE
┌─────────┬─────────┬─────────┬─────────┐
│  500t   │  500t   │  500t   │  500t   │
└─────────┴─────────┴─────────┴─────────┘
✓ Simple, predictable
✗ Breaks mid-sentence, loses semantic coherence

Strategy 2: RECURSIVE CHARACTER SPLIT
Split hierarchy: \n\n → \n → sentence → word (if needed)
Example:
  Section heading
  Paragraph 1 (800 tokens)
  Paragraph 2 (300 tokens) + Paragraph 3 (200 tokens)
✓ Respects document structure
✓ Variable size, semantic coherence
✗ Slightly more complex

Strategy 3: SEMANTIC CHUNKING
Algorithm:
  1. Split into small pieces
  2. Embed each piece
  3. Merge consecutive pieces if similarity > threshold (0.95)
  4. Split if similarity < threshold

Result:
┌────────────────┐ ┌──────────────────┐ ┌──────────┐
│ Intro section  │ │ Methods section  │ │ Data tbl │
│ (400 tokens)   │ │ (600 tokens)     │ │ (200 t.) │
└────────────────┘ └──────────────────┘ └──────────┘
✓ Best semantic coherence
✓ Chunks naturally form ideas
✗ Requires embedding all pieces (expensive)

Strategy 4: DOCUMENT-AWARE
- Headers → metadata tags
- Tables → structured JSON extraction
- Code blocks → preserve formatting + syntax highlight
- Lists → keep hierarchy

RECOMMENDATION:
- Use Strategy 2 (Recursive) for most documents
- Add 10-20% overlap between chunks (repeat last 50 tokens)
- Include document metadata (filename, page, section)
- For specialized docs (code, tables): use Strategy 4

Metadata to store:
{
  "document_id": "...",
  "chunk_index": 5,
  "page_number": 12,
  "section": "Installation Guide",
  "filename": "manual.pdf"
}
```

Retrieval optimization:

```
Step 1: QUERY EXPANSION (optional, for recall)
Original: "refund policy premium"
Expanded versions generated by LLM:
- "premium subscription cancellation refund"
- "money back guarantee premium tier"
- "return policy subscription"

Use BM25 for each expanded variant, combine results.

Step 2: HYBRID SEARCH
Vector Search (semantic):
  - k=20 nearest neighbors in embedding space
  - Good for: synonyms, paraphrases, conceptual matches

Keyword Search (BM25):
  - k=20 exact term matches + IDF scoring
  - Good for: specific entities, technical terms, dates

Reciprocal Rank Fusion:
  rrf_score = Σ 1/(k + rank_i)

  Example:
  Vector: rank1=A, rank2=B, rank3=C
  BM25:   rank1=C, rank2=A, rank3=D

  A: 1/(60+1) + 1/(60+2) = 0.0164 + 0.0161 = 0.0325
  B: 1/(60+2) = 0.0161
  C: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.0323
  D: 1/(60+3) = 0.0159

  Final ranking: A > C > B > D

Step 3: RERANKING (cross-encoder)
Input: (query, candidate_chunk) pairs
Model: Cohere Rerank or BGE Reranker
Output: relevance score 0-1

Cohere example:
  query: "What is refund policy for premium?"
  doc1: "Premium users can request refund..." → 0.95
  doc2: "Free tier does not have..." → 0.2
  doc3: "Refunds processed within..." → 0.85

After reranking: [doc1, doc3, doc2]

Benefits:
- Better ranking than vector similarity alone
- Cross-encoder (sees both query AND doc)
- vs Bi-encoder (independent representations)

Cost: ~50ms per query, ~$0.001 per 1000 queries

Step 4: CONTEXT ASSEMBLY

System Prompt (tokens: ~200):
"You are a helpful assistant answering questions about our product.
Always cite the sources of information you use.
If you don't know, say so."

Retrieved Context (tokens: ~2000, top-5 chunks):
"Source: UserManual.pdf (p. 42)
[chunk 1 text]
---
Source: FAQ.pdf (p. 8)
[chunk 2 text]
---
..."

Query (tokens: ~50):
"What is refund policy for premium users?"

Total: ~2250 tokens → $0.11 LLM cost

Step 5: GENERATION + CITATIONS

LLM generates:
"Premium users can request a refund within 30 days of purchase.
[1] The refund will be processed to the original payment method
within 5-7 business days. [2]

[1] Source: UserManual.pdf, page 42
[2] Source: FAQ.pdf, page 8"

Benefits:
- User can verify facts
- Reduces hallucination
- Transparent system
```

Avoiding hallucination:

```
Problem: LLM hallucinates facts not in retrieved documents

Solutions:

1. CONFIDENCE THRESHOLD
   - If no retrieved chunks are similar (similarity < 0.5)
   - Tell user: "I don't have enough info to answer this"

2. CONTRADICTION CHECK
   - LLM analyzes: does generated answer match retrieved docs?
   - If contradiction detected: "I'm not confident about this"

3. FACT VERIFICATION
   - For factual claims: check if mentioned in sources
   - Require explicit citation

4. TEMPERATURE CONTROL
   - Use temperature=0 for consistency
   - Reduces creative generation (sometimes too creative!)

5. PROMPT INSTRUCTIONS
   System prompt:
   "If the answer is not in the provided context,
    say 'I don't know' rather than guessing."

Metric tracking:
- Hallucination rate: % of answers contradicting sources
- Citation accuracy: % of citations pointing to relevant text
- User feedback: thumbs up/down
```

**Типичные ошибки.**

- Fixed-size chunking без overlap → loses context between chunks.
- Vector search alone без reranking → lower accuracy.
- No metadata in chunks → can't cite sources, hard to debug.
- Ingesting documents without parsing → garbage embeddings.
- Not handling multi-tenant isolation → tenant A sees B's docs.
- Embedding cost explosion → embed everything, no batching.

**На интервью.**

- Explain RAG pipeline from document ingestion to answer generation.
- Discuss chunking strategies and tradeoffs.
- Explain why hybrid search (vector + keyword) beats pure vector search.
- Упомяни reranking как critical для accuracy.
- Уточняющий вопрос: «How would you improve accuracy?» → better chunking, query expansion, fact verification.
- Уточняющий вопрос: «How would you scale to 100M documents?» → sharding vector DB, batch embeddings, multi-region.

---

### 4. Какова роль embeddings и как выбрать vector DB?

**Зачем спрашивают.** Embeddings — foundation of modern AI retrieval. Нужно понимать dimensionality, similarity metrics, performance tradeoffs между vector DBs. Critical для RAG, semantic search, clustering.

**Короткий ответ.** Embeddings — vector representation (1536-3072 dim) текста, capturing semantic meaning. Similar texts have similar embeddings (cosine similarity > 0.9). Vector DB stores billions of embeddings with fast k-NN search. Choose based on: scale, latency requirements, cost, single-machine vs cloud.

**Детальный разбор.**

Embedding models:

```
MODEL COMPARISON (2024)
┌──────────────────────────────────────────────────────────┐
│ Model            │ Dims │ Cost      │ Latency │ Quality  │
├──────────────────────────────────────────────────────────┤
│ text-embed-3-     │ 256- │ $0.02/     │ 100ms   │ Best     │
│ large (OpenAI)   │ 3072 │ 1M tokens  │ (batch) │ (MTEB)   │
│                  │      │            │         │          │
│ text-embed-3-     │ 512  │ $0.02/     │ 100ms   │ Good     │
│ small (OpenAI)   │      │ 1M tokens  │ (batch) │ (fast)   │
│                  │      │            │         │          │
│ BGE-large-en-    │ 768  │ Free       │ 20ms    │ Good     │
│ v1.5 (local)     │      │ (self-host)│ (local) │ (MTEB)   │
│                  │      │            │         │          │
│ MiniLM-L6-v2     │ 384  │ Free       │ 5ms     │ OK       │
│ (local, fast)    │      │ (self-host)│ (local) │ (fast)   │
│                  │      │            │         │          │
│ Cohere Embed-v3  │ 1024 │ $0.10/     │ 50ms    │ Excellent│
│                  │      │ 1M tokens  │ (batch) │ (multilingual)
│                  │      │            │         │          │
└──────────────────────────────────────────────────────────┘

Компромиссы:
- Dimensionality: higher = better quality but more storage/memory
- Cost: API models (OpenAI, Cohere) vs free local (BGE, MiniLM)
- Latency: local models fast, API models have network latency
- Quality: newer models better (text-embed-3-large best)

Recommendation:
- For general retrieval: text-embed-3-large (best quality/cost)
- For real-time, cost-sensitive: text-embed-3-small or MiniLM
- For on-premise/privacy: BGE-large-en-v1.5 (local, good quality)
- For multilingual: Cohere Embed-v3
```

Similarity metrics:

```
┌────────────────────────────────────────────────────────┐
│               SIMILARITY METRICS                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│ COSINE SIMILARITY (most common)                        │
│ Formula: cos(θ) = (A·B) / (|A||B|)                     │
│ Range: [-1, 1] (usually [0, 1] for normalized)         │
│ Interpretation:                                        │
│   1.0 = identical direction                            │
│   0.9 = very similar (semantic match likely)           │
│   0.5 = somewhat similar                               │
│   0.0 = orthogonal (no relation)                       │
│                                                        │
│ Use case: Standard for text embeddings                 │
│ Speed: O(n*m) with HNSW indexing = fast                │
│                                                        │
│ L2 DISTANCE (Euclidean)                                │
│ Formula: sqrt((a₁-b₁)² + (a₂-b₂)² + ...)              │
│ Range: [0, ∞]                                          │
│ Interpretation:                                        │
│   0 = identical                                        │
│   < 1 = very similar                                   │
│   > 10 = dissimilar                                    │
│                                                        │
│ Use case: OK for embeddings, more common in traditional ML
│ Speed: Similar to cosine with HNSW                     │
│                                                        │
│ DOT PRODUCT                                            │
│ Formula: A·B = Σ aᵢ*bᵢ                                  │
│ Range: [-1, 1] (for normalized vectors)                │
│ Interpretation:                                        │
│   > 0.5 = semantically similar                         │
│   < 0 = opposite meaning                               │
│                                                        │
│ Use case: Fast on GPUs (matrix mult), some VDBs        │
│ Speed: Fastest on GPU                                  │
│                                                        │
│ HAMMING DISTANCE (binary embeddings)                   │
│ Formula: count(bitwise XOR)                            │
│ Range: [0, bits]                                       │
│ Interpretation: number of differing bits               │
│                                                        │
│ Use case: Binary embeddings (ultra-fast, low memory)   │
│ Speed: Fastest (single CPU operation)                  │
│                                                        │
└────────────────────────────────────────────────────────┘

Practice:
- Embed document:      "Premium users can get 30-day refund"
  → vec_doc = [0.2, -0.5, 0.8, ...]

- Embed query:         "refund policy premium"
  → vec_query = [0.1, -0.4, 0.85, ...]

- Cosine similarity = 0.94 → high match!

- Embed unrelated:     "How to install Python?"
  → vec_other = [0.9, 0.1, -0.3, ...]

- Cosine similarity = 0.15 → low match
```

Vector Database comparison:

```
┌─────────────────────────────────────────────────────────────────┐
│ VDB       │ Scale  │ Latency │ Cost  │ Managed │ Notes           │
├─────────────────────────────────────────────────────────────────┤
│ Qdrant    │ 100M+  │ <50ms   │ Free  │ Cloud   │ Easy, fast      │
│           │        │         │       │ option  │ HNSW default    │
│           │        │         │       │         │                 │
│ Pinecone  │ 100M+  │ <100ms  │ $$$   │ Yes     │ Managed, easy   │
│           │        │         │       │         │ Good for        │
│           │        │         │       │         │ enterprise      │
│           │        │         │       │         │                 │
│ Weaviate  │ 100M+  │ <100ms  │ Free/ │ Cloud   │ GraphQL API     │
│           │        │         │ $$$   │ option  │ Multi-modal     │
│           │        │         │       │         │                 │
│ Milvus    │ 1B+    │ <50ms   │ Free  │ No      │ High-perf       │
│           │        │         │       │         │ self-hosted     │
│           │        │         │       │         │                 │
│ pgvector  │ 10M    │ <100ms  │ Free  │ No      │ PostgreSQL ext  │
│ (Postgres)│        │         │       │         │ For small-scale │
│           │        │         │       │         │                 │
│ Redis     │ 1-10M  │ <10ms   │ Free  │ Cloud   │ In-memory, fast │
│ Search    │        │         │       │ option  │ Cached results  │
│           │        │         │       │         │                 │
│ Vespa     │ 1B+    │ <100ms  │ Free  │ Cloud   │ Advanced ranking│
│           │        │         │       │ option  │ Re-ranking      │
│           │        │         │       │         │                 │
└─────────────────────────────────────────────────────────────────┘

Selection criteria:

For RAG MVP (< 10M documents):
→ Start with: pgvector (if using PostgreSQL already)
   or Qdrant (simplest, good for scaling)

For production (10M - 100M documents):
→ Qdrant self-hosted (open-source, high performance)
   or Pinecone (managed, less ops burden)

For massive scale (> 100M documents):
→ Milvus (self-hosted, distributed)
   or Vespa (advanced ranking)

For cost optimization:
→ pgvector: cheapest, tied to Postgres
→ Redis Search: fast caching layer in front

For ease of use:
→ Pinecone: fully managed, minimal setup
→ Weaviate: good API, multi-modal support
```

Indexing strategies:

```
HNSW (Hierarchical Navigable Small World)
- Most vector DBs default
- O(log n) search complexity
- Memory overhead: ~2x vector size
- Good for: general purpose, balanced speed/memory

Building:
1. Insert vectors in random order
2. Build probabilistic graph structure
3. Higher layers: sparse (long-range connections)
4. Lower layers: dense (local connections)

Query:
1. Start from top layer (few vectors)
2. Move down, narrowing search radius
3. Bottom layer: return k nearest

Example (simplified):
Layer 3: [v1] ─── [v5]           (sparse, finding direction)
Layer 2: [v1]─[v2]─[v5]─[v7]     (medium)
Layer 1: [v1]─[v2]─[v3]─[v5]─[v6]─[v7] (dense, exact results)

Benefits:
✓ Fast search: O(log n)
✓ Balanced memory usage
✓ Good for streaming ingestion
✗ Not optimal for very high dimensions (>2000)

HNSW vs LSH vs IVF:

IVF (Inverted File Index):
- Partition space into clusters
- Search: scan nearby clusters
- Fast but approximate
- Good for: very large scale (billions)

HNSW vs IVF:
- HNSW: better quality, more memory
- IVF: faster for massive scale, less memory
- Hybrid: use both for ultra-high performance
```

Memory and cost calculations:

```
Scenario: 10M documents, 1536-dim embeddings

Storage:
- Per embedding: 1536 dimensions × 4 bytes = 6.144 KB
- 10M embeddings: 10M × 6.144 KB = 61.44 GB
- Metadata (doc_id, source): +5 KB per = 50 GB
- HNSW index overhead (2x): 122.88 GB
- Total: ~235 GB

Memory (in-memory VDB like Redis):
- Would need 235 GB RAM
- Not practical for single machine
- Solution: Use disk-backed (Qdrant, Milvus)

Query latency breakdown:
1. Encode query (text → embedding): 100ms
2. Search vector DB (k-NN): 20ms
3. Retrieve metadata: 5ms
4. Network + serialization: 10ms
---
Total: ~135ms for retrieval
+ 2000ms for LLM generation
= 2135ms total

Cost (monthly):
- Vector storage (S3): 235 GB × $0.023/GB = $5.4
- Query compute (Qdrant): 1M queries/day × $0.0001 = $3/day = $90
- LLM calls: 1M queries × $0.02/query = $20k
- Embeddings (batch ingestion): 100 docs/day × 10 chunks × $0.00002 = $0.2/day = $6
---
Total: ~$20k/month (dominated by LLM cost)
```

**Типичные ошибки.**

- Using high-dim embeddings (3072) when 384 sufficient → wastes memory.
- Storing embeddings in regular SQL DB → queries scan all rows (slow).
- Not batching embeddings API calls → 1000x more expensive.
- Choosing wrong similarity metric → precision misses.
- Not monitoring embedding quality → hallucinations go undetected.
- Embedding update failures → stale vectors in DB.

**На интервью.**

- Explain embeddings as semantic vectors and cosine similarity.
- Discuss embedding model selection based on quality/cost/latency.
- Explain why vector DB needed vs regular SQL for k-NN search.
- Discuss tradeoffs: Qdrant (open-source) vs Pinecone (managed).
- Уточняющий вопрос: «How to optimize for cost?» → batch embeddings, reuse embeddings, smaller models.
- Уточняющий вопрос: «How to handle private data?» → self-hosted embeddings model + local VDB.

---

### 5. Как спроектировать LLM Serving Infrastructure для 1000 RPS?

**Зачем спрашивают.** LLM serving — critical infrastructure bottleneck. Requires understanding batching, quantization, caching, model parallelism. Real production challenge.

**Короткий ответ.** LLM serving = load balancer → model servers (vLLM/TensorRT-LLM) with request batching & paged attention, KV cache optimization, quantization (int8/int4), multi-replica deployment, monitoring (throughput/latency). Target: maximize throughput (batching) while respecting latency SLAs.

**Детальный разбор.**

LLM Serving Architecture:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Load Balancer                                │
│              (distribute by latency SLA, model)                      │
└────────┬──────────────────────┬──────────────────────┬──────────────┘
         │                      │                      │
         ▼                      ▼                      ▼
    ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
    │  vLLM       │       │  vLLM       │       │  vLLM       │
    │  Server 1   │       │  Server 2   │       │  Server 3   │
    │             │       │             │       │             │
    │ Model:      │       │ Model:      │       │ Model:      │
    │ GPT-4o      │       │ GPT-4o-mini │       │ Llama-70b   │
    │             │       │             │       │             │
    │ ┌─────────┐ │       │ ┌─────────┐ │       │ ┌─────────┐ │
    │ │ KV      │ │       │ │ KV      │ │       │ │ KV      │ │
    │ │ Cache   │ │       │ │ Cache   │ │       │ │ Cache   │ │
    │ │ (100GB) │ │       │ │ (50GB)  │ │       │ │ (200GB) │ │
    │ └─────────┘ │       │ └─────────┘ │       │ └─────────┘ │
    │             │       │             │       │             │
    │ Batching:   │       │ Batching:   │       │ Batching:   │
    │ - Batch: 64 │       │ - Batch: 128│       │ - Batch: 32 │
    │ - Token:    │       │ - Token:    │       │ - Token:    │
    │   20K/sec   │       │   30K/sec   │       │   10K/sec   │
    └─────────────┘       └─────────────┘       └─────────────┘
         │                      │                      │
         └──────────────────────┼──────────────────────┘
                                │
                                ▼
                    ┌────────────────────────┐
                    │ Monitoring/Metrics     │
                    │ - Requests/sec         │
                    │ - Avg latency p50/p99  │
                    │ - Cache hit ratio      │
                    │ - GPU utilization      │
                    └────────────────────────┘
```

Механика батчинга запросов:

```
Без батчинга (последовательно):
Запрос 1: [tokens...] → infer → 500ms
Запрос 2: [tokens...] → infer → 500ms
Запрос 3: [tokens...] → infer → 500ms
Общее время: 1500ms, Пропускная способность: 2 запроса/сек

С батчингом (параллельно):
                    ┌──────────────────────────────┐
Запрос 1: [tokens]─┤                              │
Запрос 2: [tokens]─┤ Batch inference (GPU)       │── 600ms всего
Запрос 3: [tokens]─┤ Обработать все одновременно│
                    └──────────────────────────────┘
Общее время: 600ms, Пропускная способность: 5 запросов/сек

Paged Attention в vLLM:
- Проблема фрагментации памяти: KV cache → 60% потерь
- Решение: виртуальная память (paging) как в ОС
- Результат: 4-10x улучшение пропускной способности

Схема KV cache:
Стандартная внимание:
  Токены: [t1, t2, t3, t4, ..., t50]
  K/V:    [allocated block 1] [allocated block 2] [allocated block 3]
  Потери:  [gap] [gap] [gap]

Paged Attention:
  Страницы: [page1: t1-t10] [page2: t11-t20] [page3: t21-t30]
  Переиспользование: страницы только по необходимости, без потерь
  = 60% экономия памяти
```

Quantization benefits:

```
Model Storage & Latency:
- FP32 (full precision): 70B params × 4 bytes = 280GB
- FP16 (half precision): 70B params × 2 bytes = 140GB
- INT8 (8-bit):         70B params × 1 byte = 70GB
- INT4 (4-bit):         70B params × 0.5 bytes = 35GB

INT4 GPTQ quantization tradeoff:
✓ 8x smaller model (35GB vs 280GB)
✓ Fits single GPU (80GB A100)
✓ Inference 2-3x faster
✗ ~1-2% accuracy loss (negligible for most tasks)

Quality degradation (INT4 vs FP32):
- Factual tasks: ~0.5% loss
- Creative writing: ~1-2% loss
- Code generation: ~0.8% loss
- Generally imperceptible to users

When to use quantization:
- Throughput critical: use INT4/INT8
- Accuracy critical: use FP16
- Cost critical: use INT4 + monitor metrics
```

Deployment strategy:

```
Multi-tier model serving:

Tier 1: Fast, cheap (for simple queries)
- Model: GPT-3.5 or gpt-4o-mini
- Latency: < 100ms (TTFT)
- Cost: $0.001 per query
- 70% of queries → answer here

Router logic:
if input_tokens < 200 and !tools_needed:
    → use GPT-4o-mini
else:
    → use GPT-4

Tier 2: High quality (for complex queries)
- Model: GPT-4, Claude-3-Opus, Llama-70b
- Latency: 200-500ms (TTFT)
- Cost: $0.01-0.02 per query
- 25% of queries → answer here

Tier 3: Specialized (for code, math, etc.)
- Models: Specialized fine-tuned models
- Latency: 300-1000ms
- Cost: $0.02-0.05
- 5% запросов → ответ здесь

Оптимизация стоимости:
Общая стоимость = (70% × $0.001) + (25% × $0.01) + (5% × $0.02)
           = $0.0007 + $0.0025 + $0.001
           = $0.0042 за запрос в среднем

vs всегда использовать GPT-4:
Общая стоимость = 100% × $0.02 = $0.02 за запрос

Экономия: ~79% снижение стоимости при той же среднего качестве
```

Масштабирование до 1000 RPS:

```
Расчёт:
- 1000 RPS = 1000 запросов/сек
- Средние токены на запрос: 100 входных + 150 выходных = 250 токенов
- Средняя длительность: 5 секунд (для 150 токенов при 30 токенов/сек)
- Одновременные запросы = 1000 × 5 = 5000 параллельных

Развёртывание:
- vLLM пропускная способность на GPU: 20-30 запросов/сек
- Нужно реплик: 5000 / (20 × batch_size)
- С batch_size=64: 5000 / (20×64) ≈ 4 GPU
- С распределением нагрузки

Оборудование (40 x A100 80GB):
- 4 inference серверов × 8 GPU = 32 GPU
- 1 router/lb + backup: 2 GPU
- 2 cache слоя: 6 GPU
- Всего: 40 GPU

Стоимость: 40 × $3.06/час (AWS A100) = $122.4/час
Годовая: ~$1M только на инфраструктуру

Но обслуживает 86.4M запросов/день = $0.00001 за запрос стоимость вычислений
(Плюс стоимость LLM API если использовать OpenAI)
```

**Типичные ошибки.**

- Using FP32 for all models → OOM, expensive hardware.
- No batching → 10x lower throughput.
- Single GPU server → single point of failure, can't scale.
- Not monitoring KV cache hit rate → hidden performance issues.
- Serving all queries with largest model → unnecessary cost.
- Not rate limiting clients → overload during spikes.

**На интервью.**

- Explain why batching is critical for LLM serving throughput.
- Discuss quantization tradeoffs: memory vs accuracy.
- Explain load balancing: round-robin vs least loaded.
- Mention vLLM/TensorRT-LLM as key open-source tools.
- Уточняющий вопрос: «How would you scale to 10K RPS?» → multi-region, multi-tier models, caching.
- Уточняющий вопрос: «How to meet strict latency SLA (< 100ms)?» → smaller models, more aggressive quantization, speculative decoding.

---

### 6. Как работает prompt engineering и когда лучше использовать fine-tuning вместо RAG?

**Зачем спрашивают.** Prompt engineering часто enough to solve problems без модель training. Но sometimes fine-tuning нужен. Интервьюер проверяет умение выбрать right tool.

**Короткий ответ.** Prompt engineering = instructions in context (few-shot, chain-of-thought, role-playing). RAG = retrieval of external knowledge. Fine-tuning = training model on specific examples. Use prompting first (cheap, fast), RAG for knowledge, fine-tuning only if prompting fails after 100+ iterations.

**Детальный разбор.**

Prompt engineering techniques:

```
1. SYSTEM PROMPT (role definition)
   "You are an expert in customer support."
   ✓ Sets tone, expertise level
   ✓ 5-10% quality improvement typical

2. FEW-SHOT EXAMPLES (demonstration)
   User: "How do I refund?"
   Assistant: "Request via Settings > Billing."

   User: "Can I get my money back?"
   Assistant: "Yes, refunds available within 30 days. Go to Settings..."
   ✓ Teaches format and style
   ✓ 10-20% quality improvement

3. CHAIN-OF-THOUGHT (reasoning steps)
   Bad: "What is 2+3?"
   Good: "Let me think step by step:
          - Start with 2
          - Add 3
          - Result: 5"
   ✓ Improves reasoning accuracy 30-50%
   ✓ Better for math, logic tasks

4. EXPLICIT CONSTRAINTS
   "Always cite sources. If unsure, say so. Keep under 100 words."
   ✓ Format control
   ✓ Reduces hallucinations 20%

5. ROLE-PLAYING
   "You are a CEO responding to a press question..."
   ✓ Tone/style control
   ✓ 5-15% quality improvement

6. ITERATIVE REFINEMENT
   v1: "Summarize this document"
       → Output too long

   v2: "Summarize in 3 bullet points, max 10 words each"
       → Perfect

   Typical: 5-20 iterations to dial in quality
```

When to use each approach:

```
SCENARIO 1: Customer Support Chatbot
Problem: Generic LLM says wrong policy

Solution progression:
1. Start with system prompt (15 min setup)
   - Include company policies in system prompt
   - Cost: $0 (just context)
   - Quality: +10%
   - Try 10 prompts

2. Add few-shot examples (1 hour setup)
   - 5-10 real good examples
   - Cost: +10 tokens per query
   - Quality: +15% (total +25%)
   - Try 50 variations

3. Use RAG (1-2 days engineering)
   - Index all policies
   - Cost: +100-200 tokens, +50ms latency
   - Quality: +30% (total +40%)
   - Try 100 queries, measure

4. Fine-tune only if RAG+prompt still fails (weeks)
   - 500+ labeled examples
   - Cost: $1000-5000 training + new API cost
   - Quality: +5% (total +45%)

SCENARIO 2: Highly Specific Task (e.g., medical coding)
Problem: LLM needs to assign ICD-10 codes to diagnoses

Solution:
1. Prompt + few examples: 60-70% accuracy
2. Prompt + RAG (reference manual): 75-85% accuracy
3. Fine-tune on 1000+ codes: 90%+ accuracy
   → Consider fine-tuning if domain-specific terminology

SCENARIO 3: Real-time Chat (latency critical)
Problem: Can't afford RAG latency (100-200ms embedding + search)

Solution:
1. Aggressive prompt + caching: 40ms
2. More specific fine-tuned model: 30ms
   → Fine-tune worth considering

Decision tree:

Is quality > 80% with good prompting?
  YES → Stop. Prompting sufficient
  NO → Try RAG

Does RAG + prompting > 85%?
  YES → Ship RAG solution
  NO → Consider fine-tuning

Do you have 100+ high-quality examples?
  YES → Fine-tune, measure improvement
  NO → Focus on better RAG or prompting
```

Fine-tuning economics:

```
Cost comparison (per year, 1M queries):

1. Prompting only
   LLM calls: 1M × $0.01 = $10k
   Engineering: 100 hours × $150 = $15k
   Total: $25k

2. Prompting + RAG
   LLM calls: 1M × $0.01 = $10k
   Embeddings: 1M × $0.0001 = $100
   Vector DB: $200/month = $2.4k
   Engineering: 200 hours = $30k
   Total: $42.5k
   (Better quality, but more complex)

3. Prompting + Fine-tuning
   LLM calls: 1M × $0.008 = $8k (smaller fine-tuned model)
   Fine-tuning cost: $3k (one-time)
   Engineering: 300 hours = $45k
   Total: $56k
   (Best quality, highest total cost but pays off at scale)

Break-even:
- RAG vs Prompting: 5M queries/year (RAG amortizes)
- Fine-tuning vs RAG: 10M queries/year (fine-tuning amortizes)

At 100M queries/year:
- Prompting: $100k → $1.5M
- RAG: $100k + $20k/month = $340k
- Fine-tuning: $3k + $100k + $8k/month = $200k
→ Fine-tuning wins
```

**Типичные ошибки.**

- Spending weeks on prompting when RAG would solve in days.
- Fine-tuning without baseline prompting/RAG metrics.
- Using too many few-shot examples (> 5-10) → context bloat, slower inference.
- Not measuring quality improvement → "better" prompts that aren't actually better.
- Fine-tuning without proper train/test split → overfitting.

**На интервью.**

- Explain prompt engineering progression: system prompt → few-shot → chain-of-thought.
- Discuss RAG vs fine-tuning tradeoff: retrieval vs training.
- Mention cost: RAG cheaper for knowledge, fine-tuning cheaper at huge scale.
- Provide decision framework: try prompting first, RAG if needed, fine-tune as last resort.
- Уточняющий вопрос: «How would you measure quality?» → human eval, automatic metrics (BLEU, exact match).
- Уточняющий вопрос: «What if fine-tuning model had lower latency?» → might be worth it for real-time systems.

---

### 7. Как оптимизировать стоимость AI-систем без потери качества?

**Зачем спрашивают.** Cost control = critical for production AI systems. From $0 to billion dollars depending on architecture. Интервьюер проверяет practical optimization mindset.

**Короткий ответ.** Cost optimization = tiered models by complexity, response caching, token budgets, embedding batching, quantization. Typical 50-80% cost savings with 1-5% quality loss. Combine multiple strategies.

**Детальный разбор.**

Cost breakdown for AI system (1M queries/day):

```
Component               Default    Optimized   Savings
─────────────────────────────────────────────────────
LLM calls              $20,000    $5,000      75%
  (GPT-4 for all)      (→ smart routing)

Embeddings             $500       $50         90%
  (embed everything)   (→ batch, cache)

Vector DB compute      $3,000     $500        83%
  (unoptimized queries)(→ fewer queries)

Storage                $500       $100        80%
  (all messages)       (→ archival)

Total                  $24,000    $5,650      76% savings
```

Strategy 1: Model routing by complexity

```
Classify each query by complexity:

Simple queries (50%):
- Lookup facts, read docs
- Model: GPT-4o-mini or Claude-3-Haiku
- Cost: $0.002 per query
- Quality: 95% match to GPT-4

Medium queries (35%):
- Multi-step reasoning, tool use
- Model: GPT-4o or Claude-3-Sonnet
- Cost: $0.01 per query
- Quality: 99% match

Complex queries (15%):
- Novel reasoning, long context
- Model: GPT-4 Turbo or Claude-3-Opus
- Cost: $0.03 per query
- Quality: 99.5%

Routing logic:
```python
def select_model(query, context_length):
    # Оценить сложность запроса 0-10
    complexity = 0

    if len(query) < 50:
        complexity += 1  # короткий = простой

    if "how" not in query.lower():
        complexity += 2  # фактический = простой

    if context_length > 5000:
        complexity += 3  # большой контекст = сложный

    if "not" in query or "but" in query:
        complexity += 2  # исключения = сложный

    if complexity < 3:
        return "gpt-4o-mini"  # $0.002
    elif complexity < 7:
        return "gpt-4o"  # $0.01
    else:
        return "gpt-4"  # $0.03
```

Cost impact:
50% × $0.002 + 35% × $0.01 + 15% × $0.03 = $0.0079/query
vs $0.02 (GPT-4 always) = 60% savings

Strategy 2: Response caching

```
Cache taxonomy:

1. EXACT MATCH CACHE (1 hour TTL)
   Key: hash(model, user_id, query)
   Hit rate: 5-15% (user repeats)
   Latency: <10ms (cache hit)
   Storage: 1GB per 1M queries

2. SEMANTIC SIMILARITY CACHE (24 hour TTL)
   Key: embedding(query)
   Match if similarity > 0.95
   Hit rate: 10-20% (paraphrases)
   Latency: 50ms (embedding lookup)
   Storage: 10GB per 1M queries (with embeddings)

3. PROVIDER-LEVEL CACHE (if using OpenAI)
   Built-in: caching last 2 calls
   Hit rate: < 1% (varies by user patterns)
   Cost saving: 50% if cache hits

Practical implementation:
```python
def get_response(query, user_id):
    # Проверить точный кэш
    cache_key = f"{user_id}:{hash(query)}"
    if cache.exists(cache_key):
        return cache.get(cache_key)  # 5-10ms

    # Встроить для semantic cache
    emb = embed(query)
    similar = semantic_cache.search(emb, threshold=0.95)
    if similar:
        return similar[0].response  # 20-30ms

    # Кэш miss, вызвать LLM
    response = llm.complete(query)

    # Сохранить в оба кэша
    cache.set(cache_key, response, ttl=3600)
    semantic_cache.add(emb, response)

    return response
```

Caching impact:
- Exact cache hit: 10% of queries → 90% cost save
- Semantic cache hit: 10% of queries → 90% cost save
- Total: 20% cache hit rate → 18% average cost savings

Стоимость: $0.02 за запрос × 80% (cache miss) = $0.016 (vs $0.02 без кэша)

Стратегия 3: Контроль бюджета токенов

```
Бюджет токенов пользователя (ежемесячно):

Бесплатный уровень: 100K токенов
  → 200 запросов при 500 средних токенов
  → $0.50/месяц стоимость

Pro уровень: 1M токенов
  → 2000 запросов
  → $5/месяц стоимость (пользователь платит $10 → прибыль)

Enterprise: Безлимитно

Исполнение:
```python
def check_token_budget(user_id, tokens_needed):
    used = get_user_usage(user_id, month_start())
    limit = get_user_limit(user_id)

    if used + tokens_needed > limit:
        return False, f"Monthly limit reached: {limit}"

    return True, None
```

Влияние: снижает запросы на злоупотребление, поощряет эффективность

Стратегия 4: Пакетная обработка embeddings

```
Наивный подход:
100K документов для embedding

for doc in documents:
    emb = embed_api(doc)  # 100K × $0.0001 = $10

Проблема: 100K API вызовов, неэффективно

Подход с батчингом:
for batch in chunks(documents, 500):
    embs = embed_batch_api(batch)  # 200 вызовов

Стоимость: та же $10, но в 500x быстрее
Latency: 500ms для всех vs 50 секунд для наивного

Стратегия размера батча:
- Small docs: batch_size=1000
- Large docs: batch_size=100
- Network cost: negligible for batching

Implementation:
```python
def embed_documents(docs, batch_size=500):
    results = []
    for i in range(0, len(docs), batch_size):
        batch = docs[i:i+batch_size]
        # API expects List[str], returns List[Vector]
        batch_embeddings = embedding_model.embed_batch(batch)
        results.extend(batch_embeddings)
    return results

# Before: 10,000 API calls
# After: 20 API calls (for 10K docs at batch=500)
# Speedup: 500x
```

Strategy 5: Aggressive data retention policies

```
Data storage cost: $0.023/GB/month (S3 Standard)

Lifecycle policy:
- Hot tier (< 7 days): Full resolution, fast access
  - 100GB (recent messages)
  - Cost: $2.30/month

- Warm tier (7-30 days): Compressed, slower access
  - 500GB (archive in Glacier)
  - Cost: $2.50/month

- Cold tier (> 30 days): Rare access
  - 5TB (deep archive)
  - Cost: $1/month

Total: $5.80/month vs $115/month (all hot)
= 95% storage savings
```

Strategy 6: Open-source models for specific tasks

```
Примеры использования: Генерация кода

Коммерческое: GPT-4 Turbo
- Стоимость: $0.03 за запрос
- Качество: 95% на популярных бенчмарках
- Лицензия: Proprietary

Open-source: Code-Llama-70b
- Стоимость: $0 (self-hosted) или $0.002 (on-demand)
- Качество: 92% на бенчмарках
- Лицензия: MIT

Разница в качестве: 3%, но разница в стоимости: 93%

Для большого масштаба (1M запросов генерации кода/день):
Коммерческое: $30k/день = $11M/год
Open-source: $60/день = $22k/год
Экономия: 99.8% ($11M - $22k)

Компромисс: нужна инфраструктура для open-source
Фиксированная стоимость: $1M/год (4 A100 GPU)
Но всё ещё 5x дешевле в этом масштабе
```

Контрольный список оптимизации стоимости:

```
Неделя 1 (быстрые победы, 20% экономии):
- ☐ Реализовать caching (exact + semantic)
- ☐ Установить бюджеты токенов по уровню пользователя
- ☐ Пакетизировать запросы embedding

Неделя 2 (среднее усилие, +30% экономии):
- ☐ Реализовать routing модели по сложности
- ☐ Переместить холодные данные в архивное хранилище
- ☐ Мониторить стоимость запроса по пользователю/функции

Месяц 1 (значительное усилие, +20% экономии):
- ☐ Оценить open-source модели для задач
- ☐ Реализовать quantization для self-hosted
- ☐ Set up rate limiting per user tier

Ongoing:
- Monitor cost per user/query
- A/B test model changes
- Regular audit of unused features
```

**Типичные ошибки.**

- Optimize cost before optimizing quality → ship broken system.
- Use biggest model for all queries → unnecessary cost.
- Cache everything without TTL → stale data.
- No per-user billing → can't incentivize efficiency.
- Switching to open-source too early → infrastructure complexity outweighs savings.

**На интервью.**

- Explain model routing strategy: score complexity, choose appropriate model.
- Discuss caching: exact vs semantic, TTL, hit rate expectations.
- Mention token budgets as fairness + cost mechanism.
- Provide numbers: 50-80% cost savings achievable without much quality loss.
- Уточняющий вопрос: «How would you measure cost effectiveness?» → cost per successful query, not just raw cost.
- Уточняющий вопрос: «When to move to open-source?» → at scale (millions of queries), when infrastructure cost amortizes.

---

### 8. Как построить AI Agent систему, которая надежна и безопасна?

**Зачем спрашивают.** AI agents = most powerful but dangerous applications. Can execute arbitrary code, make API calls, write to databases. Интервьюер проверяет security mindset и understanding of reliability.

**Короткий ответ.** Reliable & safe agents = permission scoping (what agent can do), human-in-the-loop for risky actions, sandboxed execution (Docker containers), comprehensive audit trails, graceful degradation (retry, rollback). Never trust agent output directly.

**Детальный разбор.**

Agent architecture with safety:

```
┌────────────────────────────────────────────────────────┐
│                   User Request                          │
│              "Create report and send email"              │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
         ┌─────────────────────────┐
         │ Permission Check        │
         │ - User allowed?         │
         │ - Request scope valid?  │
         │ - Rate limit exceeded?  │
         └────────┬────────────────┘
                  │
                  ▼
         ┌─────────────────────────┐
         │ Task Decomposition      │
         │ (LLM plans steps)       │
         │ 1. Read data            │
         │ 2. Process              │
         │ 3. Create report        │
         │ 4. Send email           │
         └────────┬────────────────┘
                  │
     ┌────────────┴────────────┐
     │                         │
     ▼                         ▼
┌──────────────┐        ┌────────────────┐
│ Step 1: Read │        │ Permission     │
│ (allowed)    │        │ Check:         │
│              │        │ read_data OK?  │
│ Execute      │        │ YES            │
└──────┬───────┘        └────────────────┘
       │
       ▼
   ┌──────────────┐        ┌─────────────────┐
   │ Step 2: Proc │        │ Step 4: Send    │
   │ (allowed)    │  ...   │ email (blocked) │
   │              │        │                 │
   │ Execute      │        │ Needs approval  │
   └──────┬───────┘        │ (send to human) │
          │                └────────┬────────┘
          │                         │
          └────────────┬────────────┘
                       │
                       ▼
         ┌──────────────────────────┐
         │ Human Review/Approval    │
         │ Email attachment logged  │
         │ User confirms            │
         └────────┬─────────────────┘
                  │
                  ▼
         ┌──────────────────────────┐
         │ Final Execution          │
         │ (with audit trail)       │
         └────────┬─────────────────┘
                  │
                  ▼
         ┌──────────────────────────┐
         │ Report + Audit Log       │
         └──────────────────────────┘
```

Permission model:

```
Define capability matrix:

Action              Risk  Approval Needed   Rate Limit
─────────────────────────────────────────────────────
read_db             LOW   No                100/min
call_public_api     LOW   No                50/min
write_db            HIGH  Yes (>$100)       10/min
delete_db           HIGH  Yes (all)         1/min
send_email          MED   Yes (external)    5/min
execute_code        HIGH  Yes (all)         1/min
payment             CRIT  Yes (all)         1/day

Implementation:
```python
class Agent:
    def __init__(self, user_id, role):
        self.user_id = user_id
        self.permissions = load_permissions(role)
        self.rate_limits = load_rate_limits(role)

    async def execute_action(self, action, args):
        # 1. Permission check
        if action not in self.permissions:
            raise PermissionError(f"{action} not allowed")

        # 2. Rate limit check
        if self.rate_limits.exceeded(action):
            raise RateLimitError(f"Rate limit for {action}")

        # 3. Оценка рисков
        risk = self.permissions[action]["risk"]
        if risk == "HIGH":
            # Нужно одобрение
            approval_id = await self.request_approval(action, args)
            if not await self.wait_approval(approval_id, timeout=300):
                raise ApprovalDenied()

        # 4. Выполнение в песочнице
        result = await self.sandbox_execute(action, args)

        # 5. Лог аудита
        self.audit_log(action, args, result, status="success")

        return result
```

Sandboxing strategies:

```
Strategy 1: Docker Container (most secure)
┌─────────────────────────────────────┐
│ Host Process                        │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ Docker Container                    │
│ - Limited CPU: 2 cores              │
│ - Limited memory: 512MB             │
│ - No network (by default)           │
│ - Read-only root filesystem         │
│ - Timeout: 30 seconds               │
│ - Kill if exceeds resources         │
└─────────────────────────────────────┘

Usage:
```python
async def sandbox_execute(code):
    container = docker.create_container(
        image="python:3.11-slim",
        cpu_quota=200000,  # 2 cores max
        memswap_limit=512 * 1024 * 1024,  # 512MB
        network_disabled=True,
        timeout=30,  # seconds
    )
    result = container.run(code)
    container.remove()
    return result
```

Strategy 2: Process isolation (faster, less secure)
Strategy 3: Restricted interpreter (slowest, hardest to escape)

Recommendation: Use Docker for risky code, process isolation for APIs

Checkpointing and recovery:

```
Problem: Agent crashes mid-execution
Solution: Checkpoint after each step

State machine:
```
PENDING → RUNNING → CHECKPOINTING → (RUNNING | COMPLETED | FAILED)
```

Checkpoint structure:
```python
class Checkpoint:
    task_id: str
    step_number: int
    step_result: Any
    messages: List[str]  # full history
    next_action: Optional[str]
    timestamp: datetime
    status: str  # running, complete, error

# Сохранено в базу данных
# При отказе можно возобновить с последней контрольной точки
```

Recovery logic:
```python
async def run_with_recovery(task_id):
    checkpoint = load_checkpoint(task_id)

    if checkpoint.status == "complete":
        return checkpoint.step_result

    if checkpoint.status == "error":
        # Возобновить с последнего успешного шага
        messages = checkpoint.messages
        next_action = plan_next_step(messages)
    else:
        # Начать с нуля
        messages = []
        next_action = initial_plan(task)

    # Continue execution
    for step in range(checkpoint.step_number + 1, max_steps):
        result = await execute_step(next_action)
        save_checkpoint(task_id, step, result, messages)

        messages.append(result)
        next_action = plan_next_step(messages)

    return result
```

Reliability patterns:

```
1. RETRY WITH BACKOFF
   ```
   for attempt in range(1, max_retries + 1):
       try:
           result = await action()
           return result
       except TemporaryError:
           wait = 2 ** attempt  # 2, 4, 8, 16... seconds
           await asyncio.sleep(wait)

   raise FinalFailure()
   ```

2. CIRCUIT BREAKER (fail fast when system down)
   ```
   if failures_in_last_min > threshold:
       return fallback_response()
   ```

3. TIMEOUT (prevent hanging)
   ```
   try:
       result = await asyncio.wait_for(action(), timeout=30)
   except asyncio.TimeoutError:
       log_timeout_error()
       return fallback_response()
   ```

4. GRACEFUL DEGRADATION (partial success)
   ```
   try:
       result = await high_quality_action()
   except Exception:
       result = await fallback_action()
   ```
```

Audit logging:

```
Every action must be logged:

log_entry = {
    "timestamp": "2024-01-15T10:30:45Z",
    "agent_id": "agent_abc123",
    "user_id": "user_xyz",
    "action": "write_db",
    "resource": "users.email",
    "input": {"email": "new@example.com"},
    "result": "success",
    "output": {"rows_updated": 1},
    "duration_ms": 145,
    "approval_id": "apr_123"  # if required
}

Storage: Immutable (append-only) database
Access: Limited to compliance team
Retention: 7 years (regulatory)
Encryption: At rest + in transit

Query examples:
- "All actions by user X in past week"
- "All write operations to users table"
- "Actions requiring approval"
- "Failed actions in past hour"
```

**Типичные ошибки.**

- Trusting agent output → agent can lie or hallucinate.
- No rate limiting → agent can spam actions.
- Executing untrusted code directly → agent can escape sandbox.
- No audit trails → can't investigate issues.
- Human-in-the-loop only for financial → missing other risky actions.

**На интервью.**

- Explain permission model: what can agent do, what needs approval.
- Discuss sandboxing: Docker for code execution, network isolation.
- Mention checkpointing: recover from failures, resumable execution.
- Emphasize audit logging: every action logged, immutable records.
- Уточняющий вопрос: «How would you prevent prompt injection?» → validate inputs, constrain outputs.
- Уточняющий вопрос: «How to handle agent hallucinations?» → fact verification, source citations.

---

### 9. Какие метрики отслеживать для AI-систем и как выявить проблемы?

**Зачем спрашивают.** "If you can't measure it, you can't improve it." Metrics reveal problems early. Интервьюер проверяет monitoring mindset.

**Короткий ответ.** Track: latency (p50/p99), cost/query, error rate, hallucination rate, user satisfaction, cache hit rate, model usage distribution. Set alerts for anomalies. Correlate metrics to find root causes.

**Детальный разбор.**

Metric categories:

```
┌─────────────────────────────────────────────────────────┐
│                    AI System Metrics                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 1. PERFORMANCE METRICS                                  │
│    - TTFT (Time To First Token): 100-500ms             │
│    - Full latency: 1-10 seconds                         │
│    - p50, p99, p999 (percentiles)                       │
│    - Throughput: requests/sec                           │
│    - Cache hit rate: % responses from cache             │
│                                                         │
│ 2. COST METRICS                                         │
│    - Cost per query: $0.001-0.05                        │
│    - Cost per successful query (failures don't count)   │
│    - Token usage distribution (p50, p95)                │
│    - Cost by user/feature/model                         │
│    - Embedding cost (separate tracking)                 │
│                                                         │
│ 3. QUALITY METRICS                                      │
│    - Accuracy: % correct answers                        │
│    - Precision/Recall: for retrieval                    │
│    - Hallucination rate: % false info                   │
│    - User satisfaction: thumbs up/down                  │
│    - Citation accuracy: % accurate sources              │
│                                                         │
│ 4. RELIABILITY METRICS                                  │
│    - Error rate: % failed requests                      │
│    - Timeout rate: % exceeded latency SLA               │
│    - Rate limit exceeded: % rejected requests           │
│    - Provider availability: uptime %                    │
│    - Fallback usage: % queries used fallback model      │
│                                                         │
│ 5. SYSTEM HEALTH METRICS                                │
│    - GPU utilization: % used                            │
│    - Memory usage: MB/GB                                │
│    - Queue depth: pending requests                      │
│    - Batch efficiency: avg batch size vs max            │
│    - KV cache hit rate: % cached                        │
│                                                         │
│ 6. BUSINESS METRICS                                     │
│    - DAU/MAU: active users                              │
│    - Queries per user: engagement                       │
│    - Revenue: $ per user                                │
│    - Churn: % lost users                                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Dashboarding example:

```
Real-time Dashboard:

┌─ OVERVIEW ────────────────────────────┐
│ Requests: 250/sec (green)             │
│ Errors: 0.02% (green)                 │
│ Avg cost: $0.0087 (yellow - trending) │
│ Cache hit: 18% (green)                │
└────────────────────────────────────────┘

┌─ LATENCY PERCENTILES ─────────────────┐
│ p50: 120ms ▁ (green)                  │
│ p99: 850ms ▃ (yellow)                 │
│ p999: 2100ms ▅ (red - SLA = 2000ms)   │
└────────────────────────────────────────┘

┌─ COST BREAKDOWN ──────────────────────┐
│ LLM calls:    $0.0072 (80%)           │
│ Embeddings:   $0.0010 (12%)           │
│ Infra:        $0.0005 (6%)            │
│ Total/query:  $0.0087                 │
└────────────────────────────────────────┘

┌─ QUALITY METRICS ─────────────────────┐
│ Accuracy (human eval): 87% (daily)    │
│ Hallucination rate: 2.1% (up from 1%) │
│ User satisfaction: 4.2/5 ★ (daily)    │
│ Cache accuracy: 98%                   │
└────────────────────────────────────────┘

Alerts:
🔴 p99 latency > 2000ms for 5 min
🟡 Cost per query > $0.012 (trending up)
🟡 Hallucination rate > 2.5% (up from 2%)
```

Implementing key metrics:

```python
from prometheus_client import Counter, Histogram, Gauge

# Отслеживание latency
latency_histogram = Histogram(
    'llm_inference_latency_ms',
    'Время от запроса до ответа',
    buckets=[10, 50, 100, 200, 500, 1000, 2000, 5000]
)

# Отслеживание стоимости
cost_counter = Counter(
    'llm_cost_cents',
    'Общая стоимость в центах',
    labelnames=['model', 'user_tier']
)

# Отслеживание качества
accuracy_gauge = Gauge(
    'llm_accuracy',
    'Процент точности',
    labelnames=['feature']
)

# Метрики кэша
cache_hits = Counter('cache_hits', 'Количество попаданий в кэш')
cache_misses = Counter('cache_misses', 'Количество промахов кэша')

# Usage
async def handle_query(query, user_id):
    start = time.time()

    try:
        # ... execute query ...
        response = await llm.complete(query)

        latency_ms = (time.time() - start) * 1000
        latency_histogram.observe(latency_ms)

        cost_cents = estimate_cost(response.tokens)
        cost_counter.labels(
            model=response.model,
            user_tier=get_user_tier(user_id)
        ).inc(cost_cents)

    except Exception as e:
        log_error(e)
```

Правила алертинга:

```
Правило 1: Высокий процент ошибок
IF error_rate > 1% FOR 5 минут
THEN send_alert("Высокий процент ошибок", severity="critical")

Правило 2: Нарушение SLA latency
IF latency_p99 > 2000ms FOR 10 минут
THEN send_alert("Нарушение SLA", severity="warning")

Правило 3: Аномалия стоимости
IF cost_per_query > (avg_cost × 1.5) FOR 30 минут
THEN send_alert("Увеличение стоимости", severity="info")

Правило 4: Деградация качества
IF hallucination_rate > 3% FOR 1 часа
THEN send_alert("Деградация качества", severity="critical")

Правило 5: Эффективность кэша упала
IF cache_hit_rate < 10% (было 20%) FOR 1 часа
THEN send_alert("Проблема с кэшем", severity="warning")
```

Корреляционный анализ для поиска root cause:

```
Алерт: "Latency p99 > 2000ms"

Исследовать:
1. Высокая ли глубина очереди?
   - ДА → больше параллельных запросов → добавить мощность

2. Изменилось ли использование GPU?
   - ДА (было 60%, теперь 95%) → регрессия модели или увеличение нагрузки

3. Изменился ли размер батча?
   - ДА (было 32, теперь 16) → проблема с батчингом модели

4. Медленнее ли отвечает провайдер?
   - ДА → проблема провайдера, переключиться на failover

5. Сместился ли трафик на более сложные запросы?
   - ДА (средние токены увеличились) → ожидаемо, мониторить тренд

Root cause: Размер батча упал с 32 на 16
Действие: Исследовать логи vLLM сервера, перезагрузить если нужно
```

Еженедельный контрольный список:

```
Понедельник утро:
□ Проверить выходные проблемы → были ли инциденты?
□ Рассмотреть тренды стоимости → приемлемый диапазон?
□ Quality metrics → any regressions?
□ User satisfaction → up or down?

Wednesday:
□ Deploy metrics dashboard → team visibility
□ Share cost breakdown → per-feature analysis
□ Quality metrics by model → which performs best?
□ Top errors → any patterns?

Friday:
□ Write weekly report → metrics, incidents, trends
□ Plan improvements → based on metrics
□ Review alerts → noise level OK?
□ Capacity planning → growth rate sustainable?
```

**Типичные ошибки.**

- Tracking too many metrics → noise, missing signal.
- Not correlating metrics → can't find root cause.
- Alerting on absolute values instead of trends → false positives.
- No human evaluation of quality → automated metrics miss issues.
- Not breaking down by dimension (model, user, feature) → can't diagnose.

**На интервью.**

- Explain key metrics: latency, cost, accuracy, error rate.
- Discuss dashboarding: real-time alerts for critical issues.
- Mention correlation analysis: latency spike → check queue depth, GPU usage.
- Provide examples: which metrics indicate which problems.
- Уточняющий вопрос: «How would you detect data drift?» → track input distribution, compare to baseline.
- Уточняющий вопрос: «How to set alert thresholds?» → baseline + 2σ deviation, or SLA-based.

---

### 10. Как масштабировать AI-систему с 100K до 1M DAU?

**Зачем спрашивают.** Scaling = hitting bottlenecks. Requires rethinking architecture at each scale. Интервьюер проверяет systems thinking.

**Короткий ответ.** Scale phases: 100K DAU (single region, basic caching), 500K (multi-region, smarter routing), 1M (sharding everything, multi-model). Each phase requires different optimizations: database sharding, cache hierarchy, model parallelism, global load balancing.

**Детальный разбор.**

Scaling roadmap:

```
Phase 1: 100K DAU (Current)
- Single region (us-east-1)
- Single chat service (2-4 replicas)
- Single database (PostgreSQL)
- Basic caching (Redis, 1 replica)
- Load: 35 RPS average, 150 RPS peak

Bottleneck: Database writes during peak

Phase 2: 500K DAU (5x growth)
- Multi-region (us-east, eu-west, ap-se)
- 10 chat service replicas per region
- Database read replicas (per region)
- Distributed caching layer
- Vector DB sharding
- Load: 175 RPS average, 700 RPS peak

Bottleneck: Inter-region latency, data consistency

Phase 3: 1M DAU (10x from start)
- Global load balancing (CDN edge)
- Dedicated model serving (vLLM fleet)
- Database sharding (by user region)
- Multi-tier cache (CDN → regional → local)
- Vector DB distributed across regions
- Load: 350 RPS average, 1400 RPS peak

Bottleneck: Cost, operational complexity
```

Database scaling:

```
Phase 1 (100K DAU): Single database
┌─────────────────┐
│  PostgreSQL     │
│  - User data    │
│  - Messages     │
│  - Metadata     │
│  Size: 60GB     │
└─────────────────┘

Phase 2 (500K DAU): Read replicas
┌──────────────────┐
│ Primary          │
│ (us-east)        │
│ - Writes         │
└────┬─────┬───────┘
     │     │
   ┌─┘     └─┐
   │         │
   ▼         ▼
┌──────┐  ┌──────┐
│Replica│ │Replica│
│ (eu) │  │ (ap) │
│Read  │  │Read  │
└──────┘  └──────┘

Issues: 300ms latency to replicas, write contention
```

Phase 3 (1M DAU): Sharding

```
Shard by user region:

┌─────────────────────────────────────────┐
│ Global Router                           │
│ ("which shard for user X?")             │
└────────┬────────────────┬───────────────┘
         │                │
         ▼                ▼
    ┌─────────┐      ┌──────────┐
    │ Shard 1 │      │ Shard 2  │
    │ (NAM+EU)│      │ (APAC)   │
    │ US-EAST │      │ AP-SE    │
    │ 30K DAU │      │ 70K DAU  │
    └─────────┘      └──────────┘

Shard assignment:
- By user country (geolocation)
- By user ID hash (if multi-region)
- By customer tier (enterprise separate)

Benefits:
✓ Each shard smaller → faster
✓ Parallel scaling
✓ Fault isolation

Challenges:
✗ Cross-shard queries (e.g., admin reports)
✗ Rebalancing when growth skewed
✗ Operational complexity
```

Vector DB scaling:

```
Phase 1 (100K DAU): Single Qdrant instance
- 10M documents
- 60GB embeddings
- Single machine

Phase 2 (500K DAU): Qdrant Cluster
- 50M documents
- 300GB embeddings
- 3-node cluster (replication)

Phase 3 (1M DAU): Distributed Vector DB
- 100M+ documents
- Shard by namespace or collection
- Regional instances:
  - US Shard (Qdrant, 500GB)
  - EU Shard (Qdrant, 400GB)
  - APAC Shard (Qdrant, 300GB)

Query routing:
```python
def get_vector_db(user_id, query):
    user_region = get_user_region(user_id)
    if user_region == "US":
        return qdrant_us
    elif user_region == "EU":
        return qdrant_eu
    else:
        return qdrant_apac
```
```

LLM serving scaling:

```
Phase 1 (100K DAU): Single region serving
- 4 x vLLM servers (A100 GPU)
- 1000 RPS throughput
- Load balanced round-robin

Phase 2 (500K DAU): Multi-region serving
- Each region: 4 x vLLM servers
- Total: 12 servers, 3000 RPS
- Regional failover

Phase 3 (1M DAU): Specialized model fleet
┌────────────────────────────────────────┐
│ Global LLM Router                      │
│ - Route by query type/model             │
│ - Cost optimization                     │
└────┬────────┬─────────┬────────────────┘
     │        │         │
     ▼        ▼         ▼
┌────────┐ ┌──────┐ ┌────────┐
│GPT-4o  │ │Fast  │ │Special-│
│pool    │ │pool  │ │ized    │
│(8 GPU) │ │(16   │ │pool    │
│Quality │ │GPU)  │ │ (4     │
│queries │ │Light │ │GPU)    │
│ 200RPS │ │light │ │Code/   │
│        │ │ 800  │ │Math    │
│        │ │ RPS  │ │100 RPS │
└────────┘ └──────┘ └────────┘

Routing logic:
- Complex query (> 500 tokens) → GPT-4o pool
- Simple query (< 100 tokens) → Fast pool
- Code/math query → Specialized pool
```

Caching hierarchy:

```
Phase 1: Single Redis
┌─────────────────────────┐
│ Redis Cache (100GB)     │
│ - User messages         │
│ - LLM responses         │
│ - Embeddings            │
└─────────────────────────┘

Phase 2: Multi-tier
                CDN (CloudFlare)
                ↓
        ┌──────────────────┐
        │ Redis Cluster    │
        │ (3 nodes, 500GB) │
        └────┬─────────────┘
             ↓
        ┌──────────────────┐
        │ Local Cache      │
        │ (in-process)     │
        └──────────────────┘

Phase 3: Geographically distributed
┌─────────────────────────────────────┐
│ CDN POP (CloudFlare edge)           │
│ - Static responses                  │
│ - High-traffic queries              │
│ TTL: 24h                            │
└────┬────────────────┬───────────────┘
     │                │
     ▼                ▼
┌──────────┐    ┌──────────┐
│Regional  │    │Regional  │
│Redis     │    │Redis     │
│(us-east) │    │(eu-west) │
└──────────┘    └──────────┘
     │                │
     └────┬───────────┘
          ▼
    ┌──────────────┐
    │Central       │
    │PostgreSQL    │
    │(source of    │
    │truth)        │
    └──────────────┘
```

Data consistency model:

```
Transactional consistency (strong):
- Every read sees latest write
- Expensive (locks, replication)
- Used for: critical data (billing, auth)

Eventual consistency (weak):
- Reads might be slightly stale
- Fast, scalable
- Used for: messages, cache

Pattern: CQRS (Command Query Responsibility Segregation)

Commands (writes):
┌──────────────────────┐
│ Authoritative DB     │
│ - Single primary     │
│ - Strong consistency │
│ - Slower writes      │
└──────────────────────┘

Queries (reads):
┌──────────────────────┐
│ Read replicas        │
│ - Multiple regions   │
│ - Eventual cons.     │
│ - Fast reads         │
└──────────────────────┘

Sync: Primary → Replicas (async, ~100ms delay)

User experience:
- Write message → success (acknowledged)
- Read message → might see old version briefly
- Acceptable: messages update within 1 second

Cost impact:
- 1M DAU with CQRS: $100k/month
- Without CQRS (single DB): $500k/month
- Saves 80% but adds complexity
```

Operational complexity at scale:

```
Monitoring stack:
- Prometheus (metrics)
- Grafana (dashboards)
- Loki (logs)
- Jaeger (distributed tracing)

Deployment:
- Kubernetes (orchestration)
- Terraform (infrastructure as code)
- ArgoCD (GitOps)

Cost at each phase:
Phase 1 (100K DAU): $50k/month
  - 4 GPU servers: $30k
  - 1 database: $5k
  - CDN, bandwidth, misc: $15k

Phase 2 (500K DAU): $150k/month
  - 12 GPU servers: $70k
  - 3 regional databases: $20k
  - Caching, CDN: $30k
  - DevOps tooling: $30k

Phase 3 (1M DAU): $350k/month
  - 30 GPU servers: $150k
  - Distributed databases: $50k
  - Vector DBs: $40k
  - Caching hierarchy: $40k
  - DevOps, monitoring: $70k
```

**Типичные ошибки.**

- Scaling database before caching (cache solves most issues).
- Sharding database too early (adds complexity, rarely needed < 1M queries/day).
- Single point of failure at scale (needs redundancy, failover).
- Not monitoring costs during scaling (costs explode 5-10x).
- Scaling compute before optimizing queries (premature horizontal scaling).

**На интервью.**

- Explain scaling phases: what breaks at each stage, how to fix it.
- Discuss database scaling: replicas → sharding progression.
- Mention caching hierarchy: local → regional → global.
- Explain data consistency tradeoffs: strong vs eventual.
- Уточняющий вопрос: «How would you scale to 10M DAU?» → cross-regional sharding, edge computing.
- Уточняющий вопрос: «How to keep costs down while scaling?» → optimize first, then scale; tiered models.

---

## Интервью Типичные Вопросы

### Быстрые Вопросы (Screen Call)

1. **What's the bottleneck in AI systems vs traditional backends?**
   → Latency (seconds, not ms), cost (per-token pricing), non-determinism

2. **How would you reduce LLM API costs by 50%?**
   → Model routing by complexity, caching, smaller models for simple queries

3. **Why is batching critical for LLM serving?**
   → 5-10x throughput improvement, small latency increase

4. **What's the difference between RAG and fine-tuning?**
   → RAG for knowledge (fast, cheap), fine-tuning for style/format (slow, expensive)

5. **How would you debug high hallucination rate?**
   → Check retrieval quality, reduce temperature, add constraints to prompt

### Design Questions (System Design)

1. **Design a RAG system for 1M documents**
   → Chunking strategy, vector DB choice, hybrid search, reranking

2. **Design an AI chatbot for 100K DAU**
   → Streaming, context management, caching, multi-provider

3. **How to serve LLM at 1000 RPS?**
   → Batching, quantization, multi-replica, load balancing

4. **Design an AI agent that's safe and reliable**
   → Permissions, human-in-the-loop, sandboxing, audit logs

### Deep Dives (Experienced Hires)

1. **How would you scale to 1M DAU while keeping costs flat?**
   → Aggressive optimization, open-source models, edge inference

2. **Design a multi-tenant RAG system**
   → Data isolation, shared infrastructure, cost attribution

3. **How would you do real-time personalization with 1M users?**
   → Embeddings-based retrieval, edge caching, batch processing

---

## Ресурсы

- OpenAI API documentation: https://platform.openai.com/docs
- vLLM (LLM serving): https://github.com/vllm-project/vllm
- Qdrant (vector DB): https://qdrant.tech
- LangChain (AI framework): https://python.langchain.com
- Anthropic Claude API: https://www.anthropic.com/api

---

**Навигация:** [Трек System Design](./README.md) · Следующая тема: [Индекс](./README.md)

