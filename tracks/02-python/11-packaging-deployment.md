# 11. Пакетирование и деплой

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как создать Python пакет?

### Зачем спрашивают
Создание пакетов — навык для библиотек и переиспользуемого кода.

### Короткий ответ
Современный способ — `pyproject.toml` (PEP 517/518). Определяет метаданные, зависимости, build system. Poetry или setuptools для сборки. Структура: `src/` layout или flat layout.

### Пример

```
mypackage/
├── pyproject.toml
├── README.md
├── src/
│   └── mypackage/
│       ├── __init__.py
│       └── core.py
└── tests/
    └── test_core.py
```

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "mypackage"
version = "0.1.0"
description = "My awesome package"
readme = "README.md"
requires-python = ">=3.9"
license = {text = "MIT"}
authors = [{name = "Author", email = "author@example.com"}]
dependencies = [
    "requests>=2.28",
]

[project.optional-dependencies]
dev = ["pytest", "black", "mypy"]

[project.scripts]
mycli = "mypackage.cli:main"

[project.urls]
Homepage = "https://github.com/user/mypackage"

# Poetry формат
[tool.poetry]
name = "mypackage"
version = "0.1.0"
description = ""
authors = ["Author <author@example.com>"]

[tool.poetry.dependencies]
python = "^3.9"
requests = "^2.28"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
```

### На интервью
**Как отвечать:** Покажите pyproject.toml, объясните src layout, упомяните poetry vs setuptools.

---

## Вопрос 2: Как работает setup.py vs pyproject.toml?

### Зачем спрашивают
Эволюция packaging. Важно знать современный подход.

### Короткий ответ
`setup.py` — старый способ (setuptools). `pyproject.toml` — современный стандарт (PEP 517/518/621). pyproject.toml декларативный, поддерживает разные build backends. setup.py ещё нужен для сложных кастомизаций.

### Пример

```python
# setup.py (старый способ)
from setuptools import setup, find_packages

setup(
    name="mypackage",
    version="0.1.0",
    packages=find_packages(where="src"),
    package_dir={"": "src"},
    install_requires=["requests>=2.28"],
    python_requires=">=3.9",
)

# Теперь лучше pyproject.toml
```

```bash
# Сборка
pip install build
python -m build  # Создаёт dist/

# Установка в dev режиме
pip install -e .
pip install -e ".[dev]"  # С optional dependencies
```

### На интервью
**Как отвечать:** Рекомендуйте pyproject.toml, объясните когда setup.py ещё нужен.

---

## Вопрос 3: Как опубликовать пакет в PyPI?

### Зачем спрашивают
Публикация — финальный шаг создания библиотеки.

### Короткий ответ
1) Создайте аккаунт на PyPI. 2) Соберите пакет (`python -m build`). 3) Загрузите через `twine`. Используйте TestPyPI для тестирования. Автоматизируйте через GitHub Actions.

### Пример

```bash
# Установка инструментов
pip install build twine

# Сборка
python -m build

# Загрузка на TestPyPI
twine upload --repository testpypi dist/*

# Загрузка на PyPI
twine upload dist/*

# С токеном (рекомендуется)
twine upload -u __token__ -p pypi-xxx dist/*
```

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI
on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install build twine
      - run: python -m build
      - run: twine upload dist/*
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_TOKEN }}
```

### На интервью
**Как отвечать:** Покажите build + twine workflow, упомяните TestPyPI и CI/CD.

---

## Вопрос 4: Как работать с Docker для Python приложений?

### Зачем спрашивают
Docker — стандарт для деплоя. Практический навык.

### Короткий ответ
Используйте official Python образы. Multi-stage builds для уменьшения размера. Копируйте requirements.txt отдельно для кэширования. Не запускайте от root. Используйте `.dockerignore`.

### Пример

```dockerfile
# Simple
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

CMD ["python", "main.py"]

# Multi-stage (оптимизированный)
FROM python:3.11-slim as builder

WORKDIR /app
RUN pip install poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt -o requirements.txt

FROM python:3.11-slim

WORKDIR /app
COPY --from=builder /app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

CMD ["python", "-m", "src.main"]
```

```
# .dockerignore
__pycache__
*.pyc
.git
.env
.venv
tests/
```

### На интервью
**Как отвечать:** Покажите multi-stage build, объясните кэширование слоёв, упомяните security (non-root user).

---

## Вопрос 5: Как управлять конфигурацией?

### Зачем спрашивают
Конфигурация — важная часть production приложений.

### Короткий ответ
`pydantic-settings` для типизированной конфигурации из env vars. Поддерживает .env файлы, валидацию, вложенные модели. Следуйте 12-factor app — конфигурация через окружение.

### Пример

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file='.env',
        env_file_encoding='utf-8',
        case_sensitive=False,
    )

    # Database
    database_url: str = Field(..., alias='DATABASE_URL')
    db_pool_size: int = 5

    # API
    api_key: str
    debug: bool = False

    # Nested
    class Redis(BaseSettings):
        host: str = 'localhost'
        port: int = 6379

    redis: Redis = Redis()

settings = Settings()
print(settings.database_url)
print(settings.redis.host)
```

```
# .env
DATABASE_URL=postgresql://user:pass@localhost/db
API_KEY=secret123
DEBUG=true
REDIS__HOST=redis.example.com
```

### На интервью
**Как отвечать:** Покажите pydantic-settings, объясните 12-factor app принципы.

---

## Вопрос 6: Как работать с переменными окружения безопасно?

### Зачем спрашивают
Безопасность конфигурации — критично для production.

### Короткий ответ
Никогда не коммитьте секреты в git. Используйте .env файлы для development (добавьте в .gitignore). Для production — переменные окружения или secrets managers (Vault, AWS Secrets Manager). Валидируйте при старте.

### Пример

```python
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Required — падает при отсутствии
    secret_key: str
    database_url: str

    # Optional с default
    debug: bool = False

    class Config:
        env_file = '.env' if os.getenv('ENV') != 'production' else None

# Валидация при старте
try:
    settings = Settings()
except ValidationError as e:
    print(f"Configuration error: {e}")
    sys.exit(1)

# Для production secrets
# AWS Secrets Manager
import boto3

def get_secret(name: str) -> str:
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=name)
    return response['SecretString']

# HashiCorp Vault
import hvac

client = hvac.Client(url='http://vault:8200')
secret = client.secrets.kv.read_secret_version(path='myapp/config')
```

```
# .gitignore
.env
.env.local
*.pem
secrets/
```

### На интервью
**Как отвечать:** Объясните разделение dev/prod конфигурации, упомяните secrets managers для production.

---

[← Назад к списку тем](README.md)
