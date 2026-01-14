# 06. Инструменты и тестирование

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работает pytest и чем он лучше unittest?

### Зачем спрашивают
pytest — стандарт в индустрии. Проверяют практический опыт тестирования.

### Короткий ответ
pytest — мощный фреймворк с простым синтаксисом (обычные assert), автоматическим обнаружением тестов, fixtures для setup/teardown, параметризацией и богатой экосистемой плагинов. Unittest требует классы и специальные методы (self.assertEqual).

### Детальный разбор

**Сравнение:**
| Аспект | pytest | unittest |
|--------|--------|----------|
| Синтаксис | `assert x == y` | `self.assertEqual(x, y)` |
| Fixtures | `@pytest.fixture` | `setUp/tearDown` методы |
| Параметризация | Встроенная | Через subTest |
| Плагины | Богатая экосистема | Ограниченная |
| Вывод ошибок | Детальный | Базовый |

### Пример

```python
# pytest - простой синтаксис
def test_addition():
    assert 1 + 1 == 2

def test_string():
    assert "hello".upper() == "HELLO"

# unittest - классы и методы
import unittest

class TestAddition(unittest.TestCase):
    def test_addition(self):
        self.assertEqual(1 + 1, 2)

# pytest показывает детали при падении
def test_dict_comparison():
    expected = {"a": 1, "b": 2}
    actual = {"a": 1, "b": 3}
    assert actual == expected
    # AssertionError: assert {'a': 1, 'b': 3} == {'a': 1, 'b': 2}
    # Differing items: {'b': 3} != {'b': 2}

# Группировка в классы (опционально)
class TestUserService:
    def test_create_user(self):
        user = create_user("Alice")
        assert user.name == "Alice"

    def test_delete_user(self):
        ...

# Запуск
# pytest                    # Все тесты
# pytest test_file.py       # Конкретный файл
# pytest -k "test_user"     # По имени
# pytest -v                 # Verbose
# pytest -x                 # Остановиться на первом падении
# pytest --lf               # Только упавшие в прошлый раз
```

### Типичные ошибки
- Использовать unittest стиль в pytest
- Забывать про `-v` для детального вывода
- Не использовать fixtures

### На интервью
**Как отвечать:** Покажите простоту assert, упомяните fixtures и параметризацию, сравните с unittest.

**Follow-up вопросы:**
- Как pytest обнаруживает тесты?
- Что такое conftest.py?
- Какие плагины pytest используете?

---

## Вопрос 2: Что такое fixtures в pytest?

### Зачем спрашивают
Fixtures — мощная концепция pytest. Показывает понимание DRY в тестах.

### Короткий ответ
Fixtures — функции с `@pytest.fixture`, которые предоставляют данные или ресурсы для тестов. Автоматически вызываются по имени параметра. Поддерживают scope (function, class, module, session), yield для cleanup, и могут зависеть друг от друга.

### Детальный разбор

**Scopes:**
- `function` — для каждого теста (default)
- `class` — для класса тестов
- `module` — для модуля
- `session` — один раз за запуск

### Пример

```python
import pytest

# Простая fixture
@pytest.fixture
def user():
    return {"name": "Alice", "age": 30}

def test_user_name(user):
    assert user["name"] == "Alice"

# Fixture с setup и cleanup
@pytest.fixture
def database():
    db = connect_to_db()
    yield db  # Значение для теста
    db.close()  # Cleanup после теста

# Scope
@pytest.fixture(scope="session")
def app():
    """Создаётся один раз за сессию"""
    return create_app()

@pytest.fixture(scope="module")
def db_connection(app):
    """Зависит от app fixture"""
    return app.get_db()

# Parametrized fixture
@pytest.fixture(params=["mysql", "postgres", "sqlite"])
def database_type(request):
    return request.param

def test_database(database_type):
    # Тест запустится 3 раза
    assert database_type in ["mysql", "postgres", "sqlite"]

# conftest.py - shared fixtures
# conftest.py
@pytest.fixture
def client():
    return TestClient(app)

# test_api.py
def test_endpoint(client):  # Автоматически из conftest
    response = client.get("/")
    assert response.status_code == 200

# Autouse
@pytest.fixture(autouse=True)
def setup_logging():
    """Выполняется для каждого теста автоматически"""
    logging.basicConfig(level=logging.DEBUG)

# Factory fixture
@pytest.fixture
def make_user():
    created = []
    def _make_user(name, age=30):
        user = User(name=name, age=age)
        created.append(user)
        return user
    yield _make_user
    # Cleanup всех созданных
    for user in created:
        user.delete()

def test_multiple_users(make_user):
    alice = make_user("Alice")
    bob = make_user("Bob", age=25)
    assert alice.name != bob.name
```

### Типичные ошибки
- Не использовать yield для cleanup
- Неправильный scope (утечка состояния)
- Дублирование fixtures вместо conftest.py

### На интервью
**Как отвечать:** Объясните injection по имени параметра, покажите yield для cleanup, упомяните scopes.

**Follow-up вопросы:**
- Как передать параметры в fixture?
- Что такое factory fixture?
- Когда использовать autouse?

---

## Вопрос 3: Как писать параметризованные тесты?

### Зачем спрашивают
Параметризация — способ тестировать множество случаев без дублирования кода.

### Короткий ответ
`@pytest.mark.parametrize` запускает тест с разными входными данными. Указывается имя параметра и список значений. Можно комбинировать несколько параметров. Для сложных случаев — parametrized fixtures.

### Пример

```python
import pytest

# Базовая параметризация
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_double(input, expected):
    assert input * 2 == expected

# С ID для читаемого вывода
@pytest.mark.parametrize("input,expected", [
    pytest.param(1, 2, id="one"),
    pytest.param(2, 4, id="two"),
    pytest.param(0, 0, id="zero"),
])
def test_double_with_ids(input, expected):
    assert input * 2 == expected

# Множественные параметры (декартово произведение)
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    # Запустится 4 раза: (1,10), (1,20), (2,10), (2,20)
    assert x * y > 0

# Ожидаемые падения
@pytest.mark.parametrize("input,expected", [
    ("1", 1),
    ("2", 2),
    pytest.param("abc", None, marks=pytest.mark.xfail),
])
def test_int_conversion(input, expected):
    assert int(input) == expected

# Skip определённых параметров
@pytest.mark.parametrize("db_type", [
    "postgres",
    pytest.param("mysql", marks=pytest.mark.skip(reason="MySQL not installed")),
])
def test_database(db_type):
    ...

# Параметризация fixture
@pytest.fixture(params=["json", "xml", "csv"])
def file_format(request):
    return request.param

def test_export(file_format):
    result = export_data(format=file_format)
    assert result is not None

# Indirect параметризация
@pytest.fixture
def user(request):
    name = request.param
    return User(name=name)

@pytest.mark.parametrize("user", ["Alice", "Bob"], indirect=True)
def test_user(user):
    assert user.name in ["Alice", "Bob"]

# Комплексный пример
@pytest.mark.parametrize("method,path,status", [
    ("GET", "/", 200),
    ("GET", "/users", 200),
    ("POST", "/users", 201),
    ("GET", "/nonexistent", 404),
    ("DELETE", "/users/1", 204),
])
def test_api_endpoints(client, method, path, status):
    response = getattr(client, method.lower())(path)
    assert response.status_code == status
```

### На интервью
**Как отвечать:** Покажите базовый синтаксис, упомяните IDs для читаемости, объясните indirect для fixtures.

---

## Вопрос 4: Как использовать mock и patch?

### Зачем спрашивают
Mocking — необходим для изоляции тестов от внешних зависимостей.

### Короткий ответ
`unittest.mock.Mock` создаёт объект-заглушку с настраиваемым поведением. `patch` временно заменяет объект на mock. Важно патчить там, где объект используется, не где определён. `MagicMock` поддерживает magic methods.

### Пример

```python
from unittest.mock import Mock, MagicMock, patch, AsyncMock

# Mock объект
mock = Mock()
mock.return_value = 42
assert mock() == 42

mock.method.return_value = "hello"
assert mock.method() == "hello"

# Проверка вызовов
mock = Mock()
mock(1, 2, key="value")
mock.assert_called_once()
mock.assert_called_with(1, 2, key="value")

# Side effect
mock = Mock(side_effect=ValueError("error"))
# mock()  # Raises ValueError

mock = Mock(side_effect=[1, 2, 3])
assert mock() == 1
assert mock() == 2

# patch как декоратор
@patch('module.external_api')
def test_with_mock(mock_api):
    mock_api.return_value = {"data": "test"}
    result = my_function()  # Вызывает external_api
    assert result == {"data": "test"}

# patch как context manager
def test_with_patch():
    with patch('module.datetime') as mock_datetime:
        mock_datetime.now.return_value = datetime(2024, 1, 1)
        assert get_current_year() == 2024

# Patch object
class MyClass:
    def method(self):
        return "original"

def test_patch_object():
    obj = MyClass()
    with patch.object(obj, 'method', return_value="mocked"):
        assert obj.method() == "mocked"

# Patch где используется, не где определён!
# module.py: from requests import get
# Патчим 'module.get', не 'requests.get'

# MagicMock для magic methods
mock = MagicMock()
mock.__str__.return_value = "mock string"
assert str(mock) == "mock string"

mock.__len__.return_value = 5
assert len(mock) == 5

# AsyncMock
async def test_async():
    mock = AsyncMock(return_value=42)
    result = await mock()
    assert result == 42

# Spec для валидации
mock = Mock(spec=RealClass)
# mock.nonexistent_method()  # AttributeError

# Property mock
class Config:
    @property
    def value(self):
        return "real"

with patch.object(Config, 'value', new_callable=PropertyMock) as mock:
    mock.return_value = "mocked"
```

### На интервью
**Как отвечать:** Покажите patch, объясните где патчить (где используется), упомяните spec для безопасности.

---

## Вопрос 5: Что такое coverage и как измерять покрытие?

### Зачем спрашивают
Покрытие — метрика качества тестов. Важно для CI/CD.

### Короткий ответ
coverage.py измеряет какие строки кода выполняются тестами. `pytest-cov` интегрирует с pytest. Метрики: line coverage, branch coverage. 100% покрытие не гарантирует качество, но низкое покрытие — красный флаг.

### Пример

```bash
# Установка
pip install pytest-cov

# Запуск
pytest --cov=mypackage tests/
pytest --cov=mypackage --cov-report=html tests/

# Минимальный порог
pytest --cov=mypackage --cov-fail-under=80 tests/
```

```python
# .coveragerc
[run]
source = mypackage
omit = */tests/*

[report]
exclude_lines =
    pragma: no cover
    raise NotImplementedError
    if __name__ == .__main__.:

# pyproject.toml
[tool.coverage.run]
source = ["mypackage"]

[tool.coverage.report]
fail_under = 80
```

### На интервью
**Как отвечать:** Объясните line vs branch coverage, упомяните что 100% не означает качество, покажите настройку в CI.

---

## Вопрос 6: Как работать с виртуальными окружениями?

### Зачем спрашивают
Изоляция зависимостей — базовый навык Python разработчика.

### Короткий ответ
Виртуальное окружение изолирует зависимости проекта. `venv` — встроенный модуль. Создаёт копию Python интерпретатора с отдельным site-packages. Активация меняет PATH, deactivate возвращает.

### Пример

```bash
# venv (встроенный)
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
.venv\Scripts\activate     # Windows
deactivate

# Без активации
.venv/bin/python script.py
.venv/bin/pip install package

# virtualenv (более функциональный)
pip install virtualenv
virtualenv .venv

# pyenv для управления версиями Python
pyenv install 3.11.0
pyenv local 3.11.0

# Где хранятся пакеты
python -c "import site; print(site.getsitepackages())"
```

### На интервью
**Как отвечать:** Объясните изоляцию зависимостей, покажите команды venv, упомяните pyenv для версий.

---

## Вопрос 7: В чём разница между pip, poetry, pipenv?

### Зачем спрашивают
Управление зависимостями эволюционирует. Проверяют знание современных инструментов.

### Короткий ответ
**pip** — базовый установщик, требует requirements.txt. **poetry** — современный инструмент с lock-файлом, управлением версиями, публикацией. **pipenv** — pip + virtualenv с Pipfile, менее популярен. Poetry — рекомендуемый выбор для новых проектов.

### Пример

```bash
# pip
pip install requests
pip freeze > requirements.txt
pip install -r requirements.txt

# poetry
poetry new myproject
poetry add requests
poetry add --dev pytest
poetry install
poetry run python script.py
poetry build
poetry publish

# pipenv
pipenv install requests
pipenv install --dev pytest
pipenv shell
```

```toml
# pyproject.toml (poetry)
[tool.poetry]
name = "myproject"
version = "0.1.0"

[tool.poetry.dependencies]
python = "^3.9"
requests = "^2.28"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
```

### На интервью
**Как отвечать:** Сравните возможности, рекомендуйте poetry для новых проектов, объясните lock-файл.

---

## Вопрос 8: Какие есть линтеры и форматтеры?

### Зачем спрашивают
Качество кода важно. Проверяют знание экосистемы инструментов.

### Короткий ответ
**Форматтеры:** Black (автоформатирование), isort (сортировка импортов). **Линтеры:** Ruff (быстрый, заменяет flake8/isort), flake8 (стиль), pylint (глубокий анализ), mypy (типы). Ruff — современный выбор, заменяет множество инструментов.

### Пример

```bash
# Black - форматирование
pip install black
black .
black --check .  # Только проверка

# Ruff - линтер (замена flake8, isort, и др.)
pip install ruff
ruff check .
ruff check --fix .

# mypy - проверка типов
pip install mypy
mypy mypackage/
```

```toml
# pyproject.toml
[tool.black]
line-length = 88

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
strict = true
```

```yaml
# pre-commit config
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.0
    hooks:
      - id: ruff
```

### На интервью
**Как отвечать:** Рекомендуйте Black + Ruff + mypy, объясните pre-commit hooks для автоматизации.

---

[← Назад к списку тем](README.md)
