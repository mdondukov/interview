# 08. FastAPI и веб-разработка

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работает FastAPI под капотом?

### Зачем спрашивают
Понимание архитектуры важно для отладки и оптимизации.

### Короткий ответ
FastAPI построен на Starlette (ASGI фреймворк) и Pydantic (валидация данных). ASGI позволяет async обработку. Pydantic валидирует request/response автоматически. OpenAPI документация генерируется из type hints.

### Пример

```python
from fastapi import FastAPI, Query, Path, Body
from pydantic import BaseModel

app = FastAPI()

# Pydantic модель для валидации
class User(BaseModel):
    name: str
    age: int
    email: str | None = None

# Автоматическая валидация и документация
@app.post("/users")
async def create_user(user: User) -> User:
    return user

# Path и Query параметры
@app.get("/users/{user_id}")
async def get_user(
    user_id: int = Path(..., ge=1),
    include_email: bool = Query(False)
) -> dict:
    return {"id": user_id}

# Запуск
# uvicorn main:app --reload
```

### На интервью
**Как отвечать:** Объясните stack (Starlette + Pydantic), покажите автоматическую валидацию из type hints.

---

## Вопрос 2: Как работает dependency injection в FastAPI?

### Зачем спрашивают
DI — ключевая фича FastAPI. Позволяет переиспользовать логику и тестировать.

### Короткий ответ
`Depends()` инжектирует зависимости в эндпоинты. Зависимости — функции или классы, возвращающие значения. Поддерживают вложенность и async. Используются для auth, DB connections, общей логики.

### Пример

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

# Простая зависимость
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
async def get_users(db: Session = Depends(get_db)):
    return db.query(User).all()

# Auth зависимость
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    user = decode_token(token)
    if not user:
        raise HTTPException(status_code=401)
    return user

@app.get("/me")
async def read_me(user: User = Depends(get_current_user)):
    return user

# Классы как зависимости
class CommonParams:
    def __init__(self, skip: int = 0, limit: int = 100):
        self.skip = skip
        self.limit = limit

@app.get("/items")
async def get_items(params: CommonParams = Depends()):
    return {"skip": params.skip, "limit": params.limit}

# Вложенные зависимости
async def get_current_active_user(
    user: User = Depends(get_current_user)
):
    if not user.is_active:
        raise HTTPException(status_code=400)
    return user
```

### На интервью
**Как отвечать:** Покажите Depends(), yield для cleanup, вложенные зависимости для auth.

---

## Вопрос 3: Как правильно организовать структуру FastAPI проекта?

### Зачем спрашивают
Архитектура важна для масштабируемости. Показывает production опыт.

### Короткий ответ
Разделяйте на роутеры (blueprints), модели, схемы, сервисы. Используйте `APIRouter` для группировки эндпоинтов. Держите бизнес-логику в сервисах, не в эндпоинтах. Конфигурация через pydantic-settings.

### Пример

```
project/
├── app/
│   ├── __init__.py
│   ├── main.py           # FastAPI app
│   ├── config.py         # Settings
│   ├── dependencies.py   # Shared dependencies
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── items.py
│   ├── models/           # SQLAlchemy models
│   │   └── user.py
│   ├── schemas/          # Pydantic schemas
│   │   └── user.py
│   ├── services/         # Business logic
│   │   └── user.py
│   └── db/
│       └── database.py
├── tests/
└── pyproject.toml
```

```python
# app/main.py
from fastapi import FastAPI
from app.routers import users, items

app = FastAPI()
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])

# app/routers/users.py
from fastapi import APIRouter, Depends
router = APIRouter()

@router.get("/")
async def list_users():
    ...

# app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    secret_key: str

    class Config:
        env_file = ".env"
```

### На интервью
**Как отвечать:** Покажите структуру с роутерами, объясните разделение на schemas/models/services.

---

## Вопрос 4: Как работать с middleware?

### Зачем спрашивают
Middleware — для cross-cutting concerns (логирование, auth, CORS).

### Короткий ответ
Middleware обрабатывает каждый request/response. Добавляется через `@app.middleware` или `app.add_middleware()`. Используется для логирования, CORS, аутентификации, метрик.

### Пример

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
import time

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom middleware
@app.middleware("http")
async def add_process_time(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    process_time = time.time() - start
    response.headers["X-Process-Time"] = str(process_time)
    return response

# Logging middleware
@app.middleware("http")
async def log_requests(request: Request, call_next):
    logger.info(f"{request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Status: {response.status_code}")
    return response
```

### На интервью
**Как отвечать:** Покажите CORS и custom middleware, объясните порядок выполнения.

---

## Вопрос 5: Как реализовать аутентификацию?

### Зачем спрашивают
Auth — обязательная часть API. Проверяют понимание безопасности.

### Короткий ответ
FastAPI поддерживает OAuth2, JWT, API keys через `fastapi.security`. OAuth2PasswordBearer для token-based auth. JWT для stateless аутентификации. Используйте зависимости для проверки.

### Пример

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import jwt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"])
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_token(data: dict) -> str:
    return jwt.encode(data, SECRET_KEY, algorithm="HS256")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user = get_user(payload["sub"])
        if not user:
            raise HTTPException(status_code=401)
        return user
    except jwt.JWTError:
        raise HTTPException(status_code=401)

@app.post("/token")
async def login(form: OAuth2PasswordRequestForm = Depends()):
    user = authenticate(form.username, form.password)
    if not user:
        raise HTTPException(status_code=401)
    token = create_token({"sub": user.username})
    return {"access_token": token, "token_type": "bearer"}

@app.get("/protected")
async def protected(user: User = Depends(get_current_user)):
    return {"user": user.username}
```

### На интервью
**Как отвечать:** Покажите OAuth2PasswordBearer + JWT, объясните flow аутентификации.

---

## Вопрос 6: Как работать с WebSocket в FastAPI?

### Зачем спрашивают
WebSocket — для real-time функциональности. Показывает знание async.

### Короткий ответ
FastAPI поддерживает WebSocket нативно. Используйте `@app.websocket()` декоратор. Обрабатывайте connect/disconnect. Для broadcast нужен connection manager.

### Пример

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.connections.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Message: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

### На интервью
**Как отвечать:** Покажите базовый WebSocket endpoint, объясните connection manager для broadcast.

---

## Вопрос 7: Как документировать API?

### Зачем спрашивают
Документация критична для API. FastAPI генерирует автоматически.

### Короткий ответ
FastAPI автоматически генерирует OpenAPI (Swagger) из type hints и Pydantic моделей. Доступно на `/docs` (Swagger UI) и `/redoc`. Добавляйте описания через docstrings и параметры.

### Пример

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field

app = FastAPI(
    title="My API",
    description="API for managing users",
    version="1.0.0"
)

class User(BaseModel):
    """User model"""
    name: str = Field(..., description="User's full name")
    age: int = Field(..., ge=0, description="User's age")

    # Pydantic v1 syntax. Для v2: model_config = ConfigDict(json_schema_extra={...})
    class Config:
        schema_extra = {
            "example": {"name": "Alice", "age": 30}
        }

@app.post("/users", response_model=User, summary="Create user")
async def create_user(user: User):
    """
    Create a new user with the following data:
    - **name**: required, user's name
    - **age**: required, must be >= 0
    """
    return user
```

### На интервью
**Как отвечать:** Покажите автогенерацию из type hints, объясните кастомизацию через Field и docstrings.

---

## Вопрос 8: Как тестировать FastAPI приложения?

### Зачем спрашивают
Тестирование API — обязательный навык.

### Короткий ответ
Используйте `TestClient` из Starlette (синхронный) или `httpx.AsyncClient` (async). Переопределяйте зависимости через `app.dependency_overrides`. Fixtures для setup/cleanup.

### Пример

```python
from fastapi.testclient import TestClient
from app.main import app
from app.dependencies import get_db

# Sync тестирование
client = TestClient(app)

def test_read_main():
    response = client.get("/")
    assert response.status_code == 200

def test_create_user():
    response = client.post("/users", json={"name": "Alice", "age": 30})
    assert response.status_code == 201
    assert response.json()["name"] == "Alice"

# Переопределение зависимостей
def override_get_db():
    return TestDatabase()

app.dependency_overrides[get_db] = override_get_db

# Async тестирование
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_async():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/")
        assert response.status_code == 200
```

### На интервью
**Как отвечать:** Покажите TestClient, объясните dependency_overrides для мокирования.

---

## См. также

- [Проектирование API](../08-architecture/07-api-design.md) — общие принципы дизайна API
- [Сетевое программирование и gRPC](../00-go/08-networking-grpc.md) — веб-разработка на Go

---

[← Назад к списку тем](README.md)
