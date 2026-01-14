# 07. AI Backend Patterns

[← Назад к списку тем](README.md)

---

## AI Service Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AI Service Architecture                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────┐  │
│  │  API    │───▶│  AI Gateway │───▶│  LLM        │───▶│ Provider│  │
│  │ Gateway │    │  Service    │    │  Processor  │    │  APIs   │  │
│  └─────────┘    └─────────────┘    └─────────────┘    └─────────┘  │
│       │               │                   │                        │
│       │               ▼                   ▼                        │
│       │        ┌─────────────┐    ┌─────────────┐                  │
│       │        │   Cache     │    │   Queue     │                  │
│       │        │   (Redis)   │    │ (RabbitMQ)  │                  │
│       │        └─────────────┘    └─────────────┘                  │
│       │                                  │                         │
│       ▼                                  ▼                         │
│  ┌─────────────┐              ┌─────────────────┐                  │
│  │  Rate       │              │  Worker Pool    │                  │
│  │  Limiter    │              │  (async proc)   │                  │
│  └─────────────┘              └─────────────────┘                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### AI Gateway Service

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
from typing import Optional
import asyncio
from openai import AsyncOpenAI

app = FastAPI()
client = AsyncOpenAI()

class CompletionRequest(BaseModel):
    prompt: str
    model: str = "gpt-4o"
    max_tokens: int = 1000
    temperature: float = 0.7
    stream: bool = False

class CompletionResponse(BaseModel):
    id: str
    content: str
    model: str
    usage: dict

@app.post("/v1/completions", response_model=CompletionResponse)
async def create_completion(request: CompletionRequest):
    response = await client.chat.completions.create(
        model=request.model,
        messages=[{"role": "user", "content": request.prompt}],
        max_tokens=request.max_tokens,
        temperature=request.temperature
    )

    return CompletionResponse(
        id=response.id,
        content=response.choices[0].message.content,
        model=response.model,
        usage={
            "prompt_tokens": response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
            "total_tokens": response.usage.total_tokens
        }
    )
```

---

## Async Processing Patterns

### Background Task Processing

```python
from fastapi import FastAPI, BackgroundTasks
from celery import Celery
import uuid
import redis

app = FastAPI()
celery_app = Celery("tasks", broker="redis://localhost:6379")
redis_client = redis.Redis()

class TaskStatus:
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

@celery_app.task(bind=True)
def process_ai_request(self, task_id: str, prompt: str):
    """Long-running AI task."""
    from openai import OpenAI
    client = OpenAI()

    redis_client.hset(f"task:{task_id}", "status", TaskStatus.PROCESSING)

    try:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}]
        )

        result = response.choices[0].message.content
        redis_client.hset(f"task:{task_id}", mapping={
            "status": TaskStatus.COMPLETED,
            "result": result
        })
        redis_client.expire(f"task:{task_id}", 3600)  # 1 hour TTL

    except Exception as e:
        redis_client.hset(f"task:{task_id}", mapping={
            "status": TaskStatus.FAILED,
            "error": str(e)
        })

@app.post("/v1/async/completions")
async def create_async_completion(prompt: str):
    task_id = str(uuid.uuid4())
    redis_client.hset(f"task:{task_id}", "status", TaskStatus.PENDING)

    process_ai_request.delay(task_id, prompt)

    return {"task_id": task_id, "status_url": f"/v1/tasks/{task_id}"}

@app.get("/v1/tasks/{task_id}")
async def get_task_status(task_id: str):
    task_data = redis_client.hgetall(f"task:{task_id}")
    if not task_data:
        raise HTTPException(status_code=404, detail="Task not found")

    return {
        "task_id": task_id,
        "status": task_data.get(b"status", b"").decode(),
        "result": task_data.get(b"result", b"").decode() or None,
        "error": task_data.get(b"error", b"").decode() or None
    }
```

### Webhook Pattern

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, HttpUrl
from typing import Optional
import httpx
import asyncio

app = FastAPI()

class WebhookRequest(BaseModel):
    prompt: str
    webhook_url: HttpUrl
    webhook_secret: Optional[str] = None

async def send_webhook(url: str, payload: dict, secret: Optional[str] = None):
    headers = {}
    if secret:
        import hmac
        import hashlib
        import json
        signature = hmac.new(
            secret.encode(),
            json.dumps(payload).encode(),
            hashlib.sha256
        ).hexdigest()
        headers["X-Webhook-Signature"] = signature

    async with httpx.AsyncClient() as client:
        try:
            await client.post(url, json=payload, headers=headers, timeout=30)
        except Exception as e:
            # Log and retry logic here
            pass

async def process_with_webhook(request: WebhookRequest):
    from openai import AsyncOpenAI
    client = AsyncOpenAI()

    try:
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": request.prompt}]
        )

        await send_webhook(
            str(request.webhook_url),
            {
                "status": "completed",
                "result": response.choices[0].message.content
            },
            request.webhook_secret
        )
    except Exception as e:
        await send_webhook(
            str(request.webhook_url),
            {"status": "failed", "error": str(e)},
            request.webhook_secret
        )

@app.post("/v1/webhook/completions")
async def create_webhook_completion(request: WebhookRequest, background_tasks: BackgroundTasks):
    background_tasks.add_task(process_with_webhook, request)
    return {"status": "accepted"}
```

---

## Streaming to Clients

### Server-Sent Events (SSE)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI
import asyncio
import json

app = FastAPI()
client = AsyncOpenAI()

async def generate_stream(prompt: str):
    """Generate SSE stream from LLM."""
    stream = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    async for chunk in stream:
        if chunk.choices[0].delta.content:
            data = {
                "content": chunk.choices[0].delta.content,
                "done": False
            }
            yield f"data: {json.dumps(data)}\n\n"

    yield f"data: {json.dumps({'done': True})}\n\n"

@app.get("/v1/stream")
async def stream_completion(prompt: str):
    return StreamingResponse(
        generate_stream(prompt),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )
```

### WebSocket Streaming

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from openai import AsyncOpenAI
import json

app = FastAPI()
client = AsyncOpenAI()

@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await websocket.accept()

    try:
        while True:
            # Receive message
            data = await websocket.receive_json()
            prompt = data.get("prompt")

            if not prompt:
                await websocket.send_json({"error": "No prompt provided"})
                continue

            # Stream response
            stream = await client.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": prompt}],
                stream=True
            )

            async for chunk in stream:
                if chunk.choices[0].delta.content:
                    await websocket.send_json({
                        "type": "chunk",
                        "content": chunk.choices[0].delta.content
                    })

            await websocket.send_json({"type": "done"})

    except WebSocketDisconnect:
        pass
```

---

## Batch Processing

### Batch API Pattern

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
from typing import List, Optional
import asyncio
from openai import AsyncOpenAI

app = FastAPI()
client = AsyncOpenAI()

class BatchRequest(BaseModel):
    prompts: List[str]
    model: str = "gpt-4o"

class BatchResult(BaseModel):
    index: int
    prompt: str
    result: Optional[str] = None
    error: Optional[str] = None

async def process_single(index: int, prompt: str, model: str, semaphore: asyncio.Semaphore) -> BatchResult:
    async with semaphore:
        try:
            response = await client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}]
            )
            return BatchResult(
                index=index,
                prompt=prompt,
                result=response.choices[0].message.content
            )
        except Exception as e:
            return BatchResult(
                index=index,
                prompt=prompt,
                error=str(e)
            )

@app.post("/v1/batch/completions")
async def batch_completions(request: BatchRequest) -> List[BatchResult]:
    # Limit concurrent requests
    semaphore = asyncio.Semaphore(10)

    tasks = [
        process_single(i, prompt, request.model, semaphore)
        for i, prompt in enumerate(request.prompts)
    ]

    results = await asyncio.gather(*tasks)
    return sorted(results, key=lambda r: r.index)
```

### Queue-based Batch Processing

```python
from celery import Celery, group
from typing import List
import json

celery_app = Celery("tasks", broker="redis://localhost:6379", backend="redis://localhost:6379")

@celery_app.task
def process_prompt(prompt: str, model: str = "gpt-4o") -> dict:
    from openai import OpenAI
    client = OpenAI()

    try:
        response = client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}]
        )
        return {
            "success": True,
            "prompt": prompt,
            "result": response.choices[0].message.content
        }
    except Exception as e:
        return {
            "success": False,
            "prompt": prompt,
            "error": str(e)
        }

def submit_batch(prompts: List[str], model: str = "gpt-4o") -> str:
    """Submit batch and return group ID."""
    job = group(process_prompt.s(p, model) for p in prompts)
    result = job.apply_async()
    return result.id

def get_batch_results(group_id: str) -> dict:
    """Get results of batch processing."""
    from celery.result import GroupResult
    result = GroupResult.restore(group_id)

    if result.ready():
        return {
            "status": "completed",
            "results": result.get()
        }
    else:
        return {
            "status": "processing",
            "completed": result.completed_count(),
            "total": len(result)
        }
```

---

## Prompt Management

### Template System

```python
from dataclasses import dataclass
from typing import Dict, Any, Optional
from jinja2 import Template
import yaml

@dataclass
class PromptTemplate:
    name: str
    template: str
    model: str
    temperature: float
    max_tokens: int
    system_prompt: Optional[str] = None

class PromptManager:
    def __init__(self, config_path: str = "prompts.yaml"):
        self.templates: Dict[str, PromptTemplate] = {}
        self._load_templates(config_path)

    def _load_templates(self, path: str):
        with open(path) as f:
            config = yaml.safe_load(f)

        for name, data in config.get("prompts", {}).items():
            self.templates[name] = PromptTemplate(
                name=name,
                template=data["template"],
                model=data.get("model", "gpt-4o"),
                temperature=data.get("temperature", 0.7),
                max_tokens=data.get("max_tokens", 1000),
                system_prompt=data.get("system_prompt")
            )

    def render(self, name: str, **variables) -> dict:
        template = self.templates.get(name)
        if not template:
            raise ValueError(f"Template '{name}' not found")

        jinja_template = Template(template.template)
        rendered = jinja_template.render(**variables)

        messages = []
        if template.system_prompt:
            messages.append({"role": "system", "content": template.system_prompt})
        messages.append({"role": "user", "content": rendered})

        return {
            "messages": messages,
            "model": template.model,
            "temperature": template.temperature,
            "max_tokens": template.max_tokens
        }

# prompts.yaml example:
# prompts:
#   summarize:
#     template: "Summarize the following text:\n\n{{ text }}"
#     model: gpt-4o-mini
#     temperature: 0.3
#     max_tokens: 500
#     system_prompt: "You are a concise summarizer."
```

### Version Control for Prompts

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, List, Optional
import hashlib
import json

@dataclass
class PromptVersion:
    id: str
    version: int
    template: str
    created_at: datetime
    metadata: Dict[str, Any]

class VersionedPromptStore:
    def __init__(self, storage):  # Redis, DB, etc.
        self.storage = storage

    def _generate_id(self, name: str, template: str) -> str:
        content = f"{name}:{template}"
        return hashlib.sha256(content.encode()).hexdigest()[:12]

    def save(self, name: str, template: str, metadata: Optional[Dict] = None) -> PromptVersion:
        # Get current version
        current = self.get_latest(name)
        new_version = (current.version + 1) if current else 1

        version = PromptVersion(
            id=self._generate_id(name, template),
            version=new_version,
            template=template,
            created_at=datetime.utcnow(),
            metadata=metadata or {}
        )

        # Store version
        key = f"prompt:{name}:v{new_version}"
        self.storage.set(key, json.dumps({
            "id": version.id,
            "version": version.version,
            "template": version.template,
            "created_at": version.created_at.isoformat(),
            "metadata": version.metadata
        }))

        # Update latest pointer
        self.storage.set(f"prompt:{name}:latest", new_version)

        return version

    def get_latest(self, name: str) -> Optional[PromptVersion]:
        latest_version = self.storage.get(f"prompt:{name}:latest")
        if not latest_version:
            return None
        return self.get_version(name, int(latest_version))

    def get_version(self, name: str, version: int) -> Optional[PromptVersion]:
        data = self.storage.get(f"prompt:{name}:v{version}")
        if not data:
            return None

        parsed = json.loads(data)
        return PromptVersion(
            id=parsed["id"],
            version=parsed["version"],
            template=parsed["template"],
            created_at=datetime.fromisoformat(parsed["created_at"]),
            metadata=parsed["metadata"]
        )
```

---

## Monitoring & Observability

### Request Logging

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional, Dict, Any
import json
import time
from contextlib import contextmanager

@dataclass
class AIRequestLog:
    request_id: str
    timestamp: datetime
    model: str
    prompt_tokens: int
    completion_tokens: int
    latency_ms: float
    status: str
    error: Optional[str] = None
    metadata: Dict[str, Any] = field(default_factory=dict)

class AILogger:
    def __init__(self, logger):
        self.logger = logger

    @contextmanager
    def track_request(self, request_id: str, model: str, metadata: Optional[Dict] = None):
        start_time = time.time()
        log_entry = {
            "request_id": request_id,
            "model": model,
            "timestamp": datetime.utcnow().isoformat(),
            "metadata": metadata or {}
        }

        try:
            yield log_entry
            log_entry["status"] = "success"
        except Exception as e:
            log_entry["status"] = "error"
            log_entry["error"] = str(e)
            raise
        finally:
            log_entry["latency_ms"] = (time.time() - start_time) * 1000
            self.logger.info(json.dumps(log_entry))

# Usage
import logging
logger = logging.getLogger("ai_requests")
ai_logger = AILogger(logger)

async def tracked_completion(client, prompt: str, request_id: str):
    with ai_logger.track_request(request_id, "gpt-4o") as log:
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}]
        )
        log["prompt_tokens"] = response.usage.prompt_tokens
        log["completion_tokens"] = response.usage.completion_tokens
        return response
```

### Metrics Collection

```python
from prometheus_client import Counter, Histogram, Gauge
import time

# Define metrics
ai_requests_total = Counter(
    "ai_requests_total",
    "Total AI API requests",
    ["model", "provider", "status"]
)

ai_request_duration = Histogram(
    "ai_request_duration_seconds",
    "AI request duration in seconds",
    ["model", "provider"],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0]
)

ai_tokens_total = Counter(
    "ai_tokens_total",
    "Total tokens used",
    ["model", "provider", "type"]  # type: prompt/completion
)

ai_cost_total = Counter(
    "ai_cost_dollars_total",
    "Total API cost in dollars",
    ["model", "provider"]
)

# Token pricing (per 1M tokens)
PRICING = {
    "gpt-4o": {"input": 2.50, "output": 10.00},
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
}

def record_ai_request(
    model: str,
    provider: str,
    duration: float,
    prompt_tokens: int,
    completion_tokens: int,
    success: bool
):
    status = "success" if success else "error"

    ai_requests_total.labels(model=model, provider=provider, status=status).inc()
    ai_request_duration.labels(model=model, provider=provider).observe(duration)

    ai_tokens_total.labels(model=model, provider=provider, type="prompt").inc(prompt_tokens)
    ai_tokens_total.labels(model=model, provider=provider, type="completion").inc(completion_tokens)

    # Calculate cost
    if model in PRICING:
        cost = (
            (prompt_tokens / 1_000_000) * PRICING[model]["input"] +
            (completion_tokens / 1_000_000) * PRICING[model]["output"]
        )
        ai_cost_total.labels(model=model, provider=provider).inc(cost)
```

### Tracing with OpenTelemetry

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from functools import wraps

# Setup
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter())
)

def trace_ai_call(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        with tracer.start_as_current_span("ai_completion") as span:
            span.set_attribute("ai.model", kwargs.get("model", "unknown"))

            try:
                result = await func(*args, **kwargs)

                span.set_attribute("ai.prompt_tokens", result.usage.prompt_tokens)
                span.set_attribute("ai.completion_tokens", result.usage.completion_tokens)
                span.set_attribute("ai.status", "success")

                return result
            except Exception as e:
                span.set_attribute("ai.status", "error")
                span.set_attribute("ai.error", str(e))
                span.record_exception(e)
                raise

    return wrapper

@trace_ai_call
async def traced_completion(client, model: str, messages: list):
    return await client.chat.completions.create(
        model=model,
        messages=messages
    )
```

---

## Testing AI Integrations

### Mocking LLM Responses

```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from dataclasses import dataclass

@dataclass
class MockUsage:
    prompt_tokens: int = 10
    completion_tokens: int = 20
    total_tokens: int = 30

@dataclass
class MockMessage:
    content: str
    role: str = "assistant"

@dataclass
class MockChoice:
    message: MockMessage
    index: int = 0

@dataclass
class MockCompletion:
    id: str = "mock-id"
    choices: list = None
    usage: MockUsage = None
    model: str = "gpt-4o"

    def __post_init__(self):
        if self.choices is None:
            self.choices = [MockChoice(message=MockMessage(content="Mock response"))]
        if self.usage is None:
            self.usage = MockUsage()

@pytest.fixture
def mock_openai_client():
    client = MagicMock()
    client.chat.completions.create = AsyncMock(return_value=MockCompletion())
    return client

@pytest.mark.asyncio
async def test_completion_service(mock_openai_client):
    # Configure mock response
    mock_openai_client.chat.completions.create.return_value = MockCompletion(
        choices=[MockChoice(message=MockMessage(content="Test response"))]
    )

    # Test your service
    from your_service import CompletionService
    service = CompletionService(client=mock_openai_client)

    result = await service.complete("Test prompt")

    assert result == "Test response"
    mock_openai_client.chat.completions.create.assert_called_once()
```

### Integration Tests with Fixtures

```python
import pytest
import os

@pytest.fixture
def real_openai_client():
    """Use real API for integration tests."""
    from openai import AsyncOpenAI

    if not os.getenv("OPENAI_API_KEY"):
        pytest.skip("OPENAI_API_KEY not set")

    return AsyncOpenAI()

@pytest.fixture
def test_prompts():
    return {
        "simple": "Say 'hello'",
        "json": "Return a JSON object with key 'test' and value 'value'",
        "long": "Write a 3 paragraph essay about testing."
    }

@pytest.mark.integration
@pytest.mark.asyncio
async def test_real_completion(real_openai_client, test_prompts):
    response = await real_openai_client.chat.completions.create(
        model="gpt-4o-mini",  # Use cheaper model for tests
        messages=[{"role": "user", "content": test_prompts["simple"]}],
        max_tokens=10
    )

    assert response.choices[0].message.content is not None
    assert "hello" in response.choices[0].message.content.lower()
```

### Snapshot Testing for Prompts

```python
import pytest
import json
import hashlib
from pathlib import Path

class PromptSnapshotTester:
    def __init__(self, snapshot_dir: str = "tests/snapshots"):
        self.snapshot_dir = Path(snapshot_dir)
        self.snapshot_dir.mkdir(parents=True, exist_ok=True)

    def _get_snapshot_path(self, name: str) -> Path:
        return self.snapshot_dir / f"{name}.json"

    def _hash_prompt(self, prompt: str) -> str:
        return hashlib.sha256(prompt.encode()).hexdigest()[:12]

    def assert_prompt_unchanged(self, name: str, prompt: str):
        """Assert prompt hasn't changed since last snapshot."""
        snapshot_path = self._get_snapshot_path(name)

        current = {
            "prompt": prompt,
            "hash": self._hash_prompt(prompt)
        }

        if snapshot_path.exists():
            with open(snapshot_path) as f:
                saved = json.load(f)

            assert saved["hash"] == current["hash"], (
                f"Prompt '{name}' has changed!\n"
                f"Saved: {saved['prompt'][:100]}...\n"
                f"Current: {prompt[:100]}..."
            )
        else:
            # Create new snapshot
            with open(snapshot_path, "w") as f:
                json.dump(current, f, indent=2)

@pytest.fixture
def prompt_snapshot():
    return PromptSnapshotTester()

def test_summarize_prompt_unchanged(prompt_snapshot):
    prompt = """Summarize the following text in 3 bullet points:

{text}

Requirements:
- Be concise
- Focus on key points
- Use plain language"""

    prompt_snapshot.assert_prompt_unchanged("summarize", prompt)
```

---

## Cost Management

### Budget Tracking

```python
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Dict, Optional
import redis

@dataclass
class BudgetConfig:
    daily_limit: float
    monthly_limit: float
    alert_threshold: float = 0.8  # Alert at 80%

class BudgetTracker:
    def __init__(self, redis_client: redis.Redis, config: BudgetConfig):
        self.redis = redis_client
        self.config = config

    def _get_daily_key(self) -> str:
        return f"budget:daily:{datetime.utcnow().strftime('%Y-%m-%d')}"

    def _get_monthly_key(self) -> str:
        return f"budget:monthly:{datetime.utcnow().strftime('%Y-%m')}"

    def record_cost(self, cost: float, metadata: Optional[Dict] = None):
        """Record API cost."""
        self.redis.incrbyfloat(self._get_daily_key(), cost)
        self.redis.incrbyfloat(self._get_monthly_key(), cost)

        # Set expiry
        self.redis.expire(self._get_daily_key(), 86400 * 2)  # 2 days
        self.redis.expire(self._get_monthly_key(), 86400 * 35)  # 35 days

    def get_usage(self) -> Dict[str, float]:
        return {
            "daily": float(self.redis.get(self._get_daily_key()) or 0),
            "monthly": float(self.redis.get(self._get_monthly_key()) or 0)
        }

    def check_budget(self) -> Dict[str, bool]:
        usage = self.get_usage()
        return {
            "daily_exceeded": usage["daily"] >= self.config.daily_limit,
            "monthly_exceeded": usage["monthly"] >= self.config.monthly_limit,
            "daily_alert": usage["daily"] >= self.config.daily_limit * self.config.alert_threshold,
            "monthly_alert": usage["monthly"] >= self.config.monthly_limit * self.config.alert_threshold
        }

    def can_proceed(self, estimated_cost: float = 0.01) -> bool:
        usage = self.get_usage()
        return (
            usage["daily"] + estimated_cost <= self.config.daily_limit and
            usage["monthly"] + estimated_cost <= self.config.monthly_limit
        )
```

### Model Selection by Cost

```python
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum

class TaskComplexity(Enum):
    SIMPLE = "simple"  # Classification, extraction
    MEDIUM = "medium"  # Summarization, QA
    COMPLEX = "complex"  # Reasoning, creative

@dataclass
class ModelOption:
    name: str
    provider: str
    cost_per_1k_tokens: float
    max_tokens: int
    complexity_threshold: TaskComplexity

class ModelSelector:
    def __init__(self):
        self.models = [
            ModelOption("gpt-4o-mini", "openai", 0.00015, 128000, TaskComplexity.SIMPLE),
            ModelOption("claude-3-haiku", "anthropic", 0.00025, 200000, TaskComplexity.SIMPLE),
            ModelOption("gpt-4o", "openai", 0.005, 128000, TaskComplexity.MEDIUM),
            ModelOption("claude-3-5-sonnet", "anthropic", 0.003, 200000, TaskComplexity.COMPLEX),
        ]

    def select(
        self,
        complexity: TaskComplexity,
        prefer_provider: Optional[str] = None,
        max_cost: Optional[float] = None
    ) -> ModelOption:
        """Select cheapest model that meets requirements."""
        candidates = [
            m for m in self.models
            if self._complexity_rank(m.complexity_threshold) >= self._complexity_rank(complexity)
        ]

        if prefer_provider:
            provider_candidates = [m for m in candidates if m.provider == prefer_provider]
            if provider_candidates:
                candidates = provider_candidates

        if max_cost:
            candidates = [m for m in candidates if m.cost_per_1k_tokens <= max_cost]

        if not candidates:
            raise ValueError("No suitable model found")

        # Return cheapest
        return min(candidates, key=lambda m: m.cost_per_1k_tokens)

    def _complexity_rank(self, c: TaskComplexity) -> int:
        return {"simple": 1, "medium": 2, "complex": 3}[c.value]
```

---

## См. также

- [AI API Integration](./06-ai-api-integration.md) — работа с API провайдеров LLM
- [AI-системы в архитектуре](../03-system-design/15-ai-systems.md) — проектирование систем с использованием LLM

---

## На интервью

### Типичные вопросы

1. **How do you handle long-running AI requests?**
   - Async processing with task queues
   - Webhook callbacks
   - Polling endpoints with status

2. **How do you scale AI services?**
   - Horizontal scaling of gateway
   - Queue-based worker pools
   - Rate limiting per user/tenant
   - Caching at multiple levels

3. **How do you monitor AI costs?**
   - Track tokens per request
   - Budget alerts and limits
   - Model selection by complexity
   - Usage dashboards

4. **How do you test AI integrations?**
   - Mock LLM responses for unit tests
   - Snapshot testing for prompts
   - Integration tests with real API
   - Evaluation pipelines

5. **How do you handle AI service outages?**
   - Multi-provider fallback
   - Circuit breakers
   - Graceful degradation
   - Cached responses as fallback

6. **How do you version prompts?**
   - Template system with variables
   - Version control (git or DB)
   - A/B testing infrastructure
   - Rollback capability

---

[← Назад к списку тем](README.md)
