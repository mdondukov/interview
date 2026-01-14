# 06. AI API Integration

[← Назад к списку тем](README.md)

---

## Обзор API провайдеров

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AI API Landscape                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  OpenAI:                                                            │
│  - GPT-4, GPT-4o, GPT-4o-mini                                       │
│  - API: chat completions, embeddings, images, audio                 │
│  - Pricing: per token, vision per image                             │
│                                                                     │
│  Anthropic:                                                         │
│  - Claude 3.5 Sonnet, Claude 3 Opus/Haiku                           │
│  - API: messages, tool use, vision                                  │
│  - Focus: safety, long context (200K tokens)                        │
│                                                                     │
│  Google:                                                            │
│  - Gemini Pro, Gemini Flash                                         │
│  - API: generateContent, embeddings                                 │
│  - Integration: Vertex AI for enterprise                            │
│                                                                     │
│  Open Source (self-hosted):                                         │
│  - Llama 3, Mistral, Mixtral                                        │
│  - Serving: vLLM, TGI, Ollama                                       │
│  - OpenAI-compatible endpoints                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## OpenAI API

### Базовый клиент

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# Простой запрос
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain microservices in 2 sentences."}
    ],
    temperature=0.7,
    max_tokens=150
)

print(response.choices[0].message.content)
```

### Streaming

```python
from openai import OpenAI

client = OpenAI()

# Streaming response
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Write a short poem"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### Structured Output (JSON Mode)

```python
from openai import OpenAI
from pydantic import BaseModel
import json

client = OpenAI()

# Способ 1: JSON mode
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Output JSON with fields: name, age, skills (array)"},
        {"role": "user", "content": "Extract info: John is 30, knows Python and Go"}
    ],
    response_format={"type": "json_object"}
)

data = json.loads(response.choices[0].message.content)

# Способ 2: Structured Outputs (schema enforcement)
class UserInfo(BaseModel):
    name: str
    age: int
    skills: list[str]

response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "user", "content": "Extract: John is 30, knows Python and Go"}
    ],
    response_format=UserInfo
)

user = response.choices[0].message.parsed  # UserInfo object
```

### Function Calling (Tools)

```python
from openai import OpenAI
import json

client = OpenAI()

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {"type": "string", "description": "City name"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                },
                "required": ["location"]
            }
        }
    }
]

def get_weather(location: str, unit: str = "celsius") -> dict:
    # Реальный API call
    return {"temp": 22, "unit": unit, "condition": "sunny"}

# Запрос с tools
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What's the weather in Moscow?"}],
    tools=tools,
    tool_choice="auto"
)

message = response.choices[0].message

# Обработка tool calls
if message.tool_calls:
    tool_call = message.tool_calls[0]
    args = json.loads(tool_call.function.arguments)

    # Выполняем функцию
    result = get_weather(**args)

    # Отправляем результат обратно
    final_response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "user", "content": "What's the weather in Moscow?"},
            message,
            {
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result)
            }
        ]
    )
    print(final_response.choices[0].message.content)
```

### Embeddings

```python
from openai import OpenAI

client = OpenAI()

# Single embedding
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Machine learning is fascinating"
)
embedding = response.data[0].embedding  # List[float], dim=1536

# Batch embeddings
texts = ["First document", "Second document", "Third document"]
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=texts
)
embeddings = [d.embedding for d in response.data]
```

---

## Anthropic API

### Базовый клиент

```python
from anthropic import Anthropic

client = Anthropic()  # Uses ANTHROPIC_API_KEY env var

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explain event sourcing briefly."}
    ]
)

print(message.content[0].text)
```

### System Prompt и Multi-turn

```python
from anthropic import Anthropic

client = Anthropic()

conversation = []

def chat(user_message: str) -> str:
    conversation.append({"role": "user", "content": user_message})

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system="You are a senior software architect. Be concise.",
        messages=conversation
    )

    assistant_message = response.content[0].text
    conversation.append({"role": "assistant", "content": assistant_message})

    return assistant_message

# Multi-turn conversation
print(chat("What's CQRS?"))
print(chat("How does it relate to event sourcing?"))
```

### Tool Use

```python
from anthropic import Anthropic
import json

client = Anthropic()

tools = [
    {
        "name": "search_database",
        "description": "Search for users in the database",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"},
                "limit": {"type": "integer", "description": "Max results"}
            },
            "required": ["query"]
        }
    }
]

def search_database(query: str, limit: int = 10) -> list:
    # Реальный DB query
    return [{"id": 1, "name": "John"}, {"id": 2, "name": "Jane"}]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Find users named John"}]
)

# Обработка tool use
if response.stop_reason == "tool_use":
    tool_use = next(block for block in response.content if block.type == "tool_use")

    result = search_database(**tool_use.input)

    # Продолжаем с результатом
    final_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=tools,
        messages=[
            {"role": "user", "content": "Find users named John"},
            {"role": "assistant", "content": response.content},
            {
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": tool_use.id,
                    "content": json.dumps(result)
                }]
            }
        ]
    )
    print(final_response.content[0].text)
```

### Vision

```python
from anthropic import Anthropic
import base64
import httpx

client = Anthropic()

# From URL
image_url = "https://example.com/image.png"
image_data = base64.standard_b64encode(httpx.get(image_url).content).decode("utf-8")

# From file
with open("image.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": image_data
                    }
                },
                {
                    "type": "text",
                    "text": "Describe this architecture diagram."
                }
            ]
        }
    ]
)
```

---

## Google Gemini API

### Базовый клиент

```python
import google.generativeai as genai
import os

genai.configure(api_key=os.getenv("GOOGLE_API_KEY"))

model = genai.GenerativeModel("gemini-1.5-flash")

response = model.generate_content("Explain Kubernetes in simple terms")
print(response.text)
```

### Multi-turn Chat

```python
import google.generativeai as genai

model = genai.GenerativeModel("gemini-1.5-pro")
chat = model.start_chat(history=[])

response1 = chat.send_message("What is Docker?")
print(response1.text)

response2 = chat.send_message("How does it differ from VMs?")
print(response2.text)

# Access history
for message in chat.history:
    print(f"{message.role}: {message.parts[0].text[:50]}...")
```

### Function Calling

```python
import google.generativeai as genai

def get_stock_price(symbol: str) -> dict:
    # Real API call
    return {"symbol": symbol, "price": 150.25, "currency": "USD"}

model = genai.GenerativeModel(
    "gemini-1.5-pro",
    tools=[get_stock_price]
)

chat = model.start_chat(enable_automatic_function_calling=True)
response = chat.send_message("What's the price of AAPL stock?")
print(response.text)  # Model automatically called get_stock_price
```

---

## Async клиенты

### OpenAI Async

```python
from openai import AsyncOpenAI
import asyncio

client = AsyncOpenAI()

async def generate_response(prompt: str) -> str:
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

async def process_batch(prompts: list[str]) -> list[str]:
    tasks = [generate_response(p) for p in prompts]
    return await asyncio.gather(*tasks)

# Usage
prompts = ["Explain REST", "Explain GraphQL", "Explain gRPC"]
results = asyncio.run(process_batch(prompts))
```

### Anthropic Async

```python
from anthropic import AsyncAnthropic
import asyncio

client = AsyncAnthropic()

async def generate(prompt: str) -> str:
    message = await client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text

async def stream_response(prompt: str):
    async with client.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)
```

---

## Error Handling

### Retry Logic

```python
from openai import OpenAI, RateLimitError, APIError, APITimeoutError
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import logging

logger = logging.getLogger(__name__)

client = OpenAI()

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60),
    retry=retry_if_exception_type((RateLimitError, APITimeoutError)),
    before_sleep=lambda retry_state: logger.warning(
        f"Retrying after {retry_state.outcome.exception()}"
    )
)
def call_with_retry(messages: list) -> str:
    try:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            timeout=30.0
        )
        return response.choices[0].message.content
    except APIError as e:
        logger.error(f"API error: {e}")
        raise
```

### Comprehensive Error Handling

```python
from openai import OpenAI, APIError, RateLimitError, APIConnectionError, AuthenticationError
from anthropic import Anthropic, APIStatusError, RateLimitError as AnthropicRateLimitError

def safe_openai_call(client: OpenAI, messages: list) -> dict:
    """Handle OpenAI API errors gracefully."""
    try:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages
        )
        return {
            "success": True,
            "content": response.choices[0].message.content,
            "usage": {
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens
            }
        }
    except AuthenticationError:
        return {"success": False, "error": "Invalid API key"}
    except RateLimitError as e:
        return {"success": False, "error": "Rate limit exceeded", "retry_after": e.response.headers.get("retry-after")}
    except APIConnectionError:
        return {"success": False, "error": "Connection failed"}
    except APIError as e:
        return {"success": False, "error": f"API error: {e.message}"}
```

---

## Rate Limiting

### Token Bucket

```python
import time
import asyncio
from dataclasses import dataclass

@dataclass
class RateLimiter:
    requests_per_minute: int
    tokens_per_minute: int
    _request_tokens: float = 0
    _token_tokens: float = 0
    _last_update: float = 0

    def __post_init__(self):
        self._request_tokens = self.requests_per_minute
        self._token_tokens = self.tokens_per_minute
        self._last_update = time.time()

    def _refill(self):
        now = time.time()
        elapsed = now - self._last_update

        self._request_tokens = min(
            self.requests_per_minute,
            self._request_tokens + elapsed * (self.requests_per_minute / 60)
        )
        self._token_tokens = min(
            self.tokens_per_minute,
            self._token_tokens + elapsed * (self.tokens_per_minute / 60)
        )
        self._last_update = now

    async def acquire(self, estimated_tokens: int):
        while True:
            self._refill()

            if self._request_tokens >= 1 and self._token_tokens >= estimated_tokens:
                self._request_tokens -= 1
                self._token_tokens -= estimated_tokens
                return

            await asyncio.sleep(0.1)

# Usage
limiter = RateLimiter(requests_per_minute=60, tokens_per_minute=90000)

async def limited_call(client, messages, estimated_tokens=500):
    await limiter.acquire(estimated_tokens)
    return await client.chat.completions.create(
        model="gpt-4o",
        messages=messages
    )
```

---

## Caching

### Response Cache

```python
import hashlib
import json
from functools import lru_cache
from typing import Optional
import redis

class LLMCache:
    def __init__(self, redis_client: redis.Redis, ttl: int = 3600):
        self.redis = redis_client
        self.ttl = ttl

    def _cache_key(self, model: str, messages: list, **kwargs) -> str:
        content = json.dumps({
            "model": model,
            "messages": messages,
            **kwargs
        }, sort_keys=True)
        return f"llm:{hashlib.sha256(content.encode()).hexdigest()}"

    def get(self, model: str, messages: list, **kwargs) -> Optional[str]:
        key = self._cache_key(model, messages, **kwargs)
        cached = self.redis.get(key)
        return cached.decode() if cached else None

    def set(self, model: str, messages: list, response: str, **kwargs):
        key = self._cache_key(model, messages, **kwargs)
        self.redis.setex(key, self.ttl, response)

# Usage
cache = LLMCache(redis.Redis())

def cached_completion(client, model: str, messages: list) -> str:
    # Try cache first
    cached = cache.get(model, messages)
    if cached:
        return cached

    # Call API
    response = client.chat.completions.create(
        model=model,
        messages=messages
    )
    content = response.choices[0].message.content

    # Cache response
    cache.set(model, messages, content)

    return content
```

### Semantic Cache

```python
import numpy as np
from openai import OpenAI

class SemanticCache:
    def __init__(self, client: OpenAI, threshold: float = 0.95):
        self.client = client
        self.threshold = threshold
        self.cache: list[tuple[np.ndarray, str, str]] = []  # (embedding, query, response)

    def _get_embedding(self, text: str) -> np.ndarray:
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return np.array(response.data[0].embedding)

    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

    def get(self, query: str) -> Optional[str]:
        query_embedding = self._get_embedding(query)

        for cached_embedding, cached_query, cached_response in self.cache:
            similarity = self._cosine_similarity(query_embedding, cached_embedding)
            if similarity >= self.threshold:
                return cached_response

        return None

    def set(self, query: str, response: str):
        embedding = self._get_embedding(query)
        self.cache.append((embedding, query, response))
```

---

## Конфигурация и секреты

### Environment Variables

```python
import os
from dataclasses import dataclass
from typing import Optional

@dataclass
class AIConfig:
    openai_api_key: Optional[str] = None
    anthropic_api_key: Optional[str] = None
    google_api_key: Optional[str] = None
    default_model: str = "gpt-4o"
    max_retries: int = 3
    timeout: float = 30.0

    @classmethod
    def from_env(cls) -> "AIConfig":
        return cls(
            openai_api_key=os.getenv("OPENAI_API_KEY"),
            anthropic_api_key=os.getenv("ANTHROPIC_API_KEY"),
            google_api_key=os.getenv("GOOGLE_API_KEY"),
            default_model=os.getenv("AI_DEFAULT_MODEL", "gpt-4o"),
            max_retries=int(os.getenv("AI_MAX_RETRIES", "3")),
            timeout=float(os.getenv("AI_TIMEOUT", "30.0"))
        )

# Usage
config = AIConfig.from_env()
```

### Multi-provider Fallback

```python
from openai import OpenAI
from anthropic import Anthropic
from dataclasses import dataclass
from typing import Optional

@dataclass
class AIProvider:
    name: str
    client: any
    model: str
    priority: int

class MultiProviderClient:
    def __init__(self):
        self.providers: list[AIProvider] = []

    def add_provider(self, name: str, client, model: str, priority: int = 0):
        self.providers.append(AIProvider(name, client, model, priority))
        self.providers.sort(key=lambda p: p.priority, reverse=True)

    def complete(self, messages: list) -> dict:
        last_error = None

        for provider in self.providers:
            try:
                if provider.name == "openai":
                    response = provider.client.chat.completions.create(
                        model=provider.model,
                        messages=messages
                    )
                    return {
                        "provider": "openai",
                        "content": response.choices[0].message.content
                    }
                elif provider.name == "anthropic":
                    response = provider.client.messages.create(
                        model=provider.model,
                        max_tokens=1024,
                        messages=messages
                    )
                    return {
                        "provider": "anthropic",
                        "content": response.content[0].text
                    }
            except Exception as e:
                last_error = e
                continue

        raise last_error or Exception("No providers available")

# Usage
ai = MultiProviderClient()
ai.add_provider("openai", OpenAI(), "gpt-4o", priority=10)
ai.add_provider("anthropic", Anthropic(), "claude-sonnet-4-20250514", priority=5)

result = ai.complete([{"role": "user", "content": "Hello"}])
```

---

## На интервью

### Типичные вопросы

1. **How do you handle API rate limits?**
   - Token bucket algorithm
   - Exponential backoff with jitter
   - Request queuing
   - Multiple API keys rotation

2. **How do you reduce API costs?**
   - Response caching (exact + semantic)
   - Smaller models for simple tasks
   - Prompt optimization (fewer tokens)
   - Batch requests when possible

3. **How do you ensure reliability?**
   - Multi-provider fallback
   - Circuit breaker pattern
   - Health checks and monitoring
   - Graceful degradation

4. **Streaming vs regular responses?**
   - Streaming: better UX, incremental processing
   - Regular: simpler, easier to cache
   - SSE for web, WebSocket for bidirectional

5. **How do you handle long-running requests?**
   - Async processing with webhooks
   - Background jobs (Celery, RQ)
   - Status polling endpoints

6. **Security considerations?**
   - API keys in secrets manager
   - Input sanitization
   - Output validation
   - Rate limiting per user

---

[← Назад к списку тем](README.md)
