# AI Integration

Подготовка к интервью для бэкенд-инженеров, интегрирующих AI в продакшн-системы. Фокус на практических паттернах: агенты, MCP серверы, RAG, работа с API провайдеров.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по практической значимости

## Темы и прогресс

### LLM Fundamentals
- [ ] [00-llm-basics](00-llm-basics.md) — как работают LLM, токенизация, attention, inference
- [ ] [01-prompt-engineering](01-prompt-engineering.md) — техники промптинга, CoT, few-shot, structured output

### RAG & Retrieval
- [ ] [02-rag-patterns](02-rag-patterns.md) — RAG архитектура, chunking, embedding, retrieval strategies
- [ ] [03-vector-databases](03-vector-databases.md) — Pinecone, Qdrant, Chroma, pgvector

### Agents & Tools
- [ ] [04-ai-agents](04-ai-agents.md) — архитектуры агентов, ReAct, planning, multi-agent (Google ADK)
- [ ] [05-mcp-servers](05-mcp-servers.md) — MCP protocol, fastmcp, tool integration

### Backend Integration
- [ ] [06-ai-api-integration](06-ai-api-integration.md) — OpenAI/Anthropic/Google API, async, error handling, caching
- [ ] [07-ai-backend-patterns](07-ai-backend-patterns.md) — сервисы, очереди, streaming, мониторинг, тестирование

## Формат вопроса

```markdown
## Тема: [Название]

### Теория
[Объяснение концепции]

### Код (Python)
[Примеры кода]

### На интервью
[Типичные вопросы и ответы]
```

## Статистика

- **Модулей:** 8
- **Вопросов:** ~50
- **Уровень:** Middle → Senior Backend Engineer
