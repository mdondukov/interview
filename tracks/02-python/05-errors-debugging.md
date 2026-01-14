# 05. Обработка ошибок и отладка

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работает exception handling в Python?

### Зачем спрашивают
Базовый навык, но часто делают ошибки. Проверяют понимание механизма и best practices.

### Короткий ответ
Python использует `try/except/else/finally` для обработки исключений. Исключения — объекты, наследующие от `BaseException`. `except` ловит указанный тип и его подклассы. `else` выполняется если исключений не было. `finally` — всегда, для cleanup.

### Детальный разбор

**Структура:**
```python
try:
    # Код, который может вызвать исключение
except SomeError as e:
    # Обработка конкретного исключения
except (TypeError, ValueError):
    # Несколько типов
except Exception:
    # Все "обычные" исключения
else:
    # Если исключений не было
finally:
    # Выполняется всегда (cleanup)
```

**Иерархия исключений:**
```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── StopIteration
    ├── ArithmeticError
    │   └── ZeroDivisionError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── OSError
    │   └── FileNotFoundError
    ├── ValueError
    ├── TypeError
    └── ...
```

### Пример

```python
# Базовая обработка
def divide(a, b):
    try:
        result = a / b
    except ZeroDivisionError:
        return None
    except TypeError as e:
        print(f"Type error: {e}")
        raise  # Пробрасываем дальше
    else:
        print("Success!")
        return result
    finally:
        print("Cleanup")

# Несколько except
def process(data):
    try:
        value = data['key']
        result = int(value)
        return result / 10
    except KeyError:
        return "Key not found"
    except ValueError:
        return "Invalid value"
    except ZeroDivisionError:
        return "Division by zero"

# Группировка исключений
try:
    ...
except (ValueError, TypeError, KeyError) as e:
    print(f"Error: {e}")

# Ловить все (плохая практика)
try:
    ...
except Exception:  # Не BaseException!
    ...

# else - выполняется если нет исключений
def safe_open(filename):
    try:
        f = open(filename)
    except FileNotFoundError:
        return None
    else:
        # Файл успешно открыт
        content = f.read()
        f.close()
        return content

# finally для cleanup
def process_file(filename):
    f = None
    try:
        f = open(filename)
        return f.read()
    except FileNotFoundError:
        return ""
    finally:
        if f:
            f.close()  # Всегда выполнится

# Лучше - context manager
def process_file_better(filename):
    try:
        with open(filename) as f:
            return f.read()
    except FileNotFoundError:
        return ""

# Exception info
import sys
import traceback

try:
    1 / 0
except ZeroDivisionError:
    exc_type, exc_value, exc_tb = sys.exc_info()
    print(f"Type: {exc_type}")
    print(f"Value: {exc_value}")
    print("Traceback:")
    traceback.print_tb(exc_tb)

# raise from - exception chaining
def load_config(path):
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError as e:
        raise ConfigError(f"Config not found: {path}") from e

# Exception groups (Python 3.11+)
try:
    raise ExceptionGroup("errors", [
        ValueError("value error"),
        TypeError("type error")
    ])
except* ValueError as eg:
    print(f"ValueError: {eg.exceptions}")
except* TypeError as eg:
    print(f"TypeError: {eg.exceptions}")
```

### Типичные ошибки
- Ловить `Exception` вместо конкретных исключений
- Забывать `raise` при логировании исключения
- Не использовать `finally` или context managers для cleanup
- Ловить `BaseException` (включает SystemExit, KeyboardInterrupt)

### На интервью
**Как отвечать:** Покажите полную структуру try/except/else/finally, объясните иерархию исключений, упомяните exception chaining.

**Follow-up вопросы:**
- Зачем нужен else в try?
- Чем Exception отличается от BaseException?
- Что такое exception chaining?

---

## Вопрос 2: В чём разница между Exception и BaseException?

### Зачем спрашивают
Важное различие для правильной обработки ошибок. Часто путают.

### Короткий ответ
`BaseException` — корень иерархии всех исключений. `Exception` — базовый класс для "обычных" исключений. `SystemExit`, `KeyboardInterrupt`, `GeneratorExit` наследуют напрямую от `BaseException`, не от `Exception`. Ловить нужно `Exception`, не `BaseException`.

### Детальный разбор

**Исключения вне Exception:**
- `SystemExit` — вызывается `sys.exit()`
- `KeyboardInterrupt` — Ctrl+C
- `GeneratorExit` — закрытие генератора

**Почему важно:**
```python
# ПЛОХО - ловит Ctrl+C!
try:
    while True:
        process()
except:  # или except BaseException
    pass  # Невозможно прервать программу!

# ХОРОШО
try:
    while True:
        process()
except Exception:
    pass  # Ctrl+C всё ещё работает
```

### Пример

```python
import sys

# BaseException hierarchy
print(BaseException.__subclasses__())
# [Exception, GeneratorExit, SystemExit, KeyboardInterrupt]

print(Exception.__subclasses__())
# [StopIteration, StopAsyncIteration, ArithmeticError, ...]

# Правильная обработка
def main():
    try:
        run_application()
    except KeyboardInterrupt:
        print("\nInterrupted by user")
        sys.exit(1)
    except SystemExit:
        raise  # Пропускаем sys.exit()
    except Exception as e:
        print(f"Error: {e}")
        sys.exit(1)

# GeneratorExit
def my_generator():
    try:
        while True:
            yield 1
    except GeneratorExit:
        print("Generator closed")
        # Cleanup

gen = my_generator()
next(gen)
gen.close()  # Вызывает GeneratorExit

# SystemExit
def cleanup():
    print("Cleaning up...")

import atexit
atexit.register(cleanup)

# sys.exit() выбрасывает SystemExit
# try:
#     sys.exit(0)
# except SystemExit as e:
#     print(f"Exit code: {e.code}")
#     raise  # Обычно пробрасываем

# Когда нужен BaseException
class Supervisor:
    """Перехватывает ВСЕ исключения для логирования"""
    def run(self, func):
        try:
            return func()
        except BaseException as e:
            self.log_exception(e)
            raise  # ВАЖНО: всегда пробрасываем

    def log_exception(self, e):
        print(f"Logged: {type(e).__name__}: {e}")

# Проверка типа исключения
def is_critical(exc):
    return isinstance(exc, (SystemExit, KeyboardInterrupt))

# Антипаттерн
try:
    data = fetch_data()
except:  # Ловит ВСЁ, включая Ctrl+C
    data = default_data  # ПЛОХО!

# Правильно
try:
    data = fetch_data()
except Exception:  # Только "обычные" ошибки
    data = default_data
```

### Типичные ошибки
- Использовать `except:` без типа (ловит BaseException)
- Ловить `BaseException` и не пробрасывать
- Путать с кастомными исключениями (должны наследовать Exception)

### На интервью
**Как отвечать:** Перечислите исключения вне Exception (SystemExit, KeyboardInterrupt, GeneratorExit), объясните почему нужно ловить Exception, а не BaseException.

**Follow-up вопросы:**
- Когда нужно ловить BaseException?
- Что делает sys.exit()?
- Как правильно обработать Ctrl+C?

---

## Вопрос 3: Как создавать custom exceptions правильно?

### Зачем спрашивают
Custom exceptions — часть API библиотеки/приложения. Показывает качество кода.

### Короткий ответ
Наследуйте от `Exception`, не от `BaseException`. Создайте базовый класс для своего модуля. Храните контекст в атрибутах. Переопределите `__str__` для читаемых сообщений. Используйте иерархию для разных типов ошибок.

### Детальный разбор

**Best practices:**
1. Один базовый класс на модуль/пакет
2. Наследование для подтипов ошибок
3. Информативные сообщения
4. Контекст в атрибутах
5. Docstrings

### Пример

```python
# Базовый класс для модуля
class MyAppError(Exception):
    """Базовое исключение для приложения"""
    pass

# Иерархия исключений
class ValidationError(MyAppError):
    """Ошибка валидации данных"""
    def __init__(self, field, message, value=None):
        self.field = field
        self.message = message
        self.value = value
        super().__init__(f"{field}: {message}")

class DatabaseError(MyAppError):
    """Ошибка работы с БД"""
    pass

class ConnectionError(DatabaseError):
    """Ошибка соединения с БД"""
    def __init__(self, host, port, cause=None):
        self.host = host
        self.port = port
        self.cause = cause
        super().__init__(f"Cannot connect to {host}:{port}")

class QueryError(DatabaseError):
    """Ошибка выполнения запроса"""
    def __init__(self, query, cause=None):
        self.query = query
        self.cause = cause
        super().__init__(f"Query failed: {query[:50]}...")

# Использование
def validate_user(data):
    if not data.get('email'):
        raise ValidationError('email', 'Email is required')
    if not '@' in data['email']:
        raise ValidationError('email', 'Invalid email format', data['email'])

def connect_to_db(host, port):
    try:
        # connection logic
        pass
    except socket.error as e:
        raise ConnectionError(host, port, cause=e) from e

# Обработка
def create_user(data):
    try:
        validate_user(data)
        user = save_to_db(data)
        return user
    except ValidationError as e:
        print(f"Validation failed for {e.field}: {e.message}")
        if e.value:
            print(f"  Value was: {e.value}")
    except DatabaseError as e:
        print(f"Database error: {e}")

# С кодами ошибок (API style)
class APIError(Exception):
    """Базовое исключение для API"""
    status_code = 500
    error_code = "INTERNAL_ERROR"

    def __init__(self, message=None, details=None):
        self.message = message or self.__class__.__doc__
        self.details = details or {}
        super().__init__(self.message)

    def to_dict(self):
        return {
            "error": self.error_code,
            "message": self.message,
            "details": self.details
        }

class NotFoundError(APIError):
    """Resource not found"""
    status_code = 404
    error_code = "NOT_FOUND"

class UnauthorizedError(APIError):
    """Authentication required"""
    status_code = 401
    error_code = "UNAUTHORIZED"

class BadRequestError(APIError):
    """Invalid request"""
    status_code = 400
    error_code = "BAD_REQUEST"

# Использование в API
@app.errorhandler(APIError)
def handle_api_error(error):
    response = jsonify(error.to_dict())
    response.status_code = error.status_code
    return response

def get_user(user_id):
    user = db.find_user(user_id)
    if not user:
        raise NotFoundError(f"User {user_id} not found")
    return user

# Enum для типов ошибок
from enum import Enum

class ErrorCode(Enum):
    INVALID_INPUT = "INVALID_INPUT"
    NOT_FOUND = "NOT_FOUND"
    PERMISSION_DENIED = "PERMISSION_DENIED"

class AppException(Exception):
    def __init__(self, code: ErrorCode, message: str, **context):
        self.code = code
        self.message = message
        self.context = context
        super().__init__(message)
```

### Типичные ошибки
- Наследовать от BaseException
- Не сохранять контекст (только message)
- Слишком много исключений без иерархии
- Не использовать `from e` для chaining

### На интервью
**Как отвечать:** Покажите иерархию с базовым классом, объясните хранение контекста в атрибутах, упомяните exception chaining.

**Follow-up вопросы:**
- Зачем нужен базовый класс на модуль?
- Как создать исключение для API?
- Когда использовать exception codes?

---

## Вопрос 4: Что такое exception chaining (from clause)?

### Зачем спрашивают
Exception chaining — важная фича для debugging. Показывает понимание error propagation.

### Короткий ответ
`raise NewError() from original` связывает исключения, сохраняя цепочку причин. `__cause__` хранит явно указанную причину, `__context__` — неявную (при raise внутри except). `raise ... from None` подавляет контекст.

### Детальный разбор

**Типы chaining:**
1. **Explicit** (`from e`) — явная причина, `__cause__`
2. **Implicit** — автоматический контекст, `__context__`
3. **Suppressed** (`from None`) — без контекста

### Пример

```python
# Implicit chaining (автоматический)
def process():
    try:
        return 1 / 0
    except ZeroDivisionError:
        raise ValueError("Cannot process")  # __context__ установлен

try:
    process()
except ValueError as e:
    print(e.__context__)  # ZeroDivisionError
    print(e.__cause__)    # None

# Traceback покажет:
# During handling of the above exception, another exception occurred:

# Explicit chaining (from)
def convert(value):
    try:
        return int(value)
    except ValueError as e:
        raise ConversionError(f"Cannot convert {value}") from e

try:
    convert("abc")
except ConversionError as e:
    print(e.__cause__)    # ValueError
    print(e.__context__)  # ValueError (то же)

# Traceback покажет:
# The above exception was the direct cause of the following exception:

# Suppressing context (from None)
def clean_convert(value):
    try:
        return int(value)
    except ValueError:
        raise ConversionError(f"Invalid value: {value}") from None

try:
    clean_convert("abc")
except ConversionError as e:
    print(e.__cause__)    # None
    print(e.__context__)  # None (подавлен)

# Traceback не покажет оригинальное исключение

# Практический пример
class ConfigError(Exception):
    pass

def load_config(path):
    try:
        with open(path) as f:
            data = json.load(f)
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in {path}: line {e.lineno}") from e

    return data

# Проход по цепочке
def get_root_cause(exc):
    """Находит изначальную причину исключения"""
    while exc.__cause__:
        exc = exc.__cause__
    return exc

try:
    load_config("missing.json")
except ConfigError as e:
    root = get_root_cause(e)
    print(f"Root cause: {type(root).__name__}: {root}")

# Логирование с цепочкой
import traceback

try:
    process()
except Exception as e:
    # Полный traceback с цепочкой
    traceback.print_exc()

    # Или в строку
    tb_str = traceback.format_exc()

# Re-raise с сохранением оригинала
def wrapper():
    try:
        risky_operation()
    except Exception as e:
        log_error(e)
        raise  # Сохраняет оригинальный traceback

# Добавление контекста без создания нового исключения
def add_context(e, context):
    """Добавляет контекст к существующему исключению"""
    e.args = (f"{context}: {e.args[0]}",) + e.args[1:]
    return e

try:
    process_item(item)
except Exception as e:
    raise add_context(e, f"Processing item {item.id}")
```

### Типичные ошибки
- Не использовать `from e` при перебрасывании
- Забывать про `from None` когда контекст не нужен
- Не логировать полный traceback

### На интервью
**Как отвечать:** Объясните разницу между `__cause__` и `__context__`, покажите когда использовать `from e` и `from None`.

**Follow-up вопросы:**
- Как найти root cause?
- Когда использовать from None?
- Как логировать цепочку исключений?

---

## Вопрос 5: Как работает context manager и with statement?

### Зачем спрашивают
Context managers — идиоматический Python для управления ресурсами. Критично для написания надёжного кода.

### Короткий ответ
Context manager — объект с методами `__enter__` и `__exit__`. `with` вызывает `__enter__` при входе и `__exit__` при выходе (даже при исключении). Используется для cleanup: файлы, соединения, блокировки. `contextlib` предоставляет утилиты для создания context managers.

### Детальный разбор

**Протокол:**
```python
class ContextManager:
    def __enter__(self):
        # Setup, возвращает значение для as
        return resource

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Cleanup
        # Если вернуть True - исключение подавлено
        return False
```

**Эквивалентный код:**
```python
# with cm as value:
#     body

manager = cm
value = manager.__enter__()
try:
    body
except:
    if not manager.__exit__(*sys.exc_info()):
        raise
else:
    manager.__exit__(None, None, None)
```

### Пример

```python
# Класс-based context manager
class ManagedFile:
    def __init__(self, filename, mode='r'):
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.file:
            self.file.close()
        return False  # Не подавляем исключения

with ManagedFile('test.txt', 'w') as f:
    f.write('Hello')

# contextlib.contextmanager - генератор
from contextlib import contextmanager

@contextmanager
def managed_file(filename, mode='r'):
    f = open(filename, mode)
    try:
        yield f
    finally:
        f.close()

with managed_file('test.txt') as f:
    content = f.read()

# Подавление исключений
class SuppressErrors:
    def __init__(self, *exceptions):
        self.exceptions = exceptions

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type and issubclass(exc_type, self.exceptions):
            return True  # Подавить
        return False

with SuppressErrors(FileNotFoundError):
    os.remove('nonexistent.txt')  # Не падает

# contextlib.suppress (встроенный)
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove('nonexistent.txt')

# Множественные context managers
with open('in.txt') as f_in, open('out.txt', 'w') as f_out:
    f_out.write(f_in.read())

# ExitStack для динамического количества
from contextlib import ExitStack

def process_files(filenames):
    with ExitStack() as stack:
        files = [stack.enter_context(open(f)) for f in filenames]
        # Все файлы закроются при выходе
        return [f.read() for f in files]

# Async context manager
class AsyncResource:
    async def __aenter__(self):
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.disconnect()
        return False

async def use_resource():
    async with AsyncResource() as r:
        await r.do_something()

# contextlib.asynccontextmanager
from contextlib import asynccontextmanager

@asynccontextmanager
async def async_db():
    conn = await connect_db()
    try:
        yield conn
    finally:
        await conn.close()

# Практические примеры
@contextmanager
def timer(name):
    import time
    start = time.time()
    try:
        yield
    finally:
        print(f"{name}: {time.time() - start:.2f}s")

with timer("operation"):
    heavy_computation()

# Транзакция
@contextmanager
def transaction(conn):
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise

with transaction(db.connection()) as conn:
    conn.execute("INSERT ...")

# Временное изменение
@contextmanager
def temporary_change(obj, attr, value):
    original = getattr(obj, attr)
    setattr(obj, attr, value)
    try:
        yield
    finally:
        setattr(obj, attr, original)

with temporary_change(config, 'debug', True):
    run_debug_code()
```

### Типичные ошибки
- Забывать cleanup в finally генератора
- Возвращать True в `__exit__` без необходимости (подавляет исключения!)
- Не использовать context manager для ресурсов

### На интервью
**Как отвечать:** Объясните протокол `__enter__`/`__exit__`, покажите @contextmanager, упомяните ExitStack и async context managers.

**Follow-up вопросы:**
- Что делает return True в `__exit__`?
- Как создать async context manager?
- Когда использовать ExitStack?

---

## Вопрос 6: Как использовать logging правильно?

### Зачем спрашивают
Logging — необходимый навык для production. Показывает зрелость разработчика.

### Короткий ответ
Используйте модуль `logging`, не `print()`. Создавайте логгеры через `logging.getLogger(__name__)`. Настраивайте уровни: DEBUG, INFO, WARNING, ERROR, CRITICAL. Конфигурируйте handlers и formatters. Структурированное логирование для production.

### Детальный разбор

**Уровни:**
| Level | Значение | Использование |
|-------|----------|---------------|
| DEBUG | 10 | Детальная информация для отладки |
| INFO | 20 | Подтверждение работы |
| WARNING | 30 | Что-то неожиданное, но работает |
| ERROR | 40 | Ошибка, часть функционала не работает |
| CRITICAL | 50 | Серьёзная ошибка, программа может упасть |

**Компоненты:**
- **Logger** — создаёт записи
- **Handler** — отправляет записи (файл, консоль, сеть)
- **Formatter** — форматирует записи
- **Filter** — фильтрует записи

### Пример

```python
import logging

# Базовая настройка
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Получение логгера (используйте __name__)
logger = logging.getLogger(__name__)

# Логирование
logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")
logger.critical("Critical message")

# С параметрами (lazy evaluation)
logger.info("Processing item %s", item_id)  # Правильно
# logger.info(f"Processing item {item_id}")  # Работает, но менее эффективно

# Логирование исключений
try:
    risky_operation()
except Exception:
    logger.exception("Operation failed")  # Включает traceback

# Продвинутая конфигурация
logger = logging.getLogger('myapp')
logger.setLevel(logging.DEBUG)

# Console handler
console = logging.StreamHandler()
console.setLevel(logging.INFO)
console_format = logging.Formatter('%(levelname)s - %(message)s')
console.setFormatter(console_format)
logger.addHandler(console)

# File handler
file_handler = logging.FileHandler('app.log')
file_handler.setLevel(logging.DEBUG)
file_format = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
file_handler.setFormatter(file_format)
logger.addHandler(file_handler)

# Конфигурация через dict
import logging.config

LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'standard': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        },
        'json': {
            'class': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard',
            'stream': 'ext://sys.stdout'
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'level': 'DEBUG',
            'formatter': 'json',
            'filename': 'app.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5
        }
    },
    'loggers': {
        '': {  # Root logger
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': True
        },
        'myapp': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': False
        }
    }
}

logging.config.dictConfig(LOGGING_CONFIG)

# Структурированное логирование
import structlog

structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

log = structlog.get_logger()
log.info("user_login", user_id=123, ip="192.168.1.1")
# {"user_id": 123, "ip": "192.168.1.1", "event": "user_login", ...}

# Контекст в логах
class RequestContextFilter(logging.Filter):
    def filter(self, record):
        record.request_id = getattr(request_context, 'id', 'N/A')
        return True

# Extra данные
logger.info("Event", extra={'user_id': 123, 'action': 'login'})

# Не делайте так:
# print("Debug: something happened")  # Нет контроля
# logger.info("User " + str(user_id))  # Конкатенация
# logger.exception("Error")  # Вне except блока
```

### Типичные ошибки
- Использовать print() вместо logging
- Конкатенация строк вместо параметров
- Логировать пароли и персональные данные
- Не использовать `logger.exception()` в except блоках

### На интервью
**Как отвечать:** Покажите getLogger(__name__), объясните уровни, упомяните структурированное логирование для production.

**Follow-up вопросы:**
- Почему не использовать print?
- Как настроить rotation логов?
- Что такое структурированное логирование?

---

## Вопрос 7: Как дебажить Python код?

### Зачем спрашивают
Debugging — ежедневный навык. Проверяют практический опыт.

### Короткий ответ
Используйте `breakpoint()` (Python 3.7+) или `pdb.set_trace()` для интерактивной отладки. `pdb` — встроенный debugger с командами n(ext), s(tep), c(ontinue), p(rint). IDE debuggers (PyCharm, VS Code) удобнее. Для post-mortem: `pdb.post_mortem()`.

### Детальный разбор

**Инструменты:**
- `pdb` — встроенный debugger
- `breakpoint()` — точка останова (Python 3.7+)
- `ipdb` — улучшенный pdb с IPython
- `pudb` — консольный GUI debugger
- IDE debuggers — PyCharm, VS Code

**Команды pdb:**
| Команда | Действие |
|---------|----------|
| `n(ext)` | Следующая строка |
| `s(tep)` | Войти в функцию |
| `c(ontinue)` | Продолжить до следующего breakpoint |
| `p expr` | Напечатать выражение |
| `pp expr` | Pretty print |
| `l(ist)` | Показать код |
| `w(here)` | Показать stack trace |
| `b line` | Установить breakpoint |
| `q(uit)` | Выйти |

### Пример

```python
# breakpoint() - рекомендуемый способ
def process_data(data):
    result = []
    for item in data:
        breakpoint()  # Остановится здесь
        processed = transform(item)
        result.append(processed)
    return result

# Старый способ
import pdb

def old_style():
    pdb.set_trace()  # Эквивалент breakpoint()
    ...

# Условный breakpoint
for i, item in enumerate(items):
    if item.is_suspicious:
        breakpoint()
    process(item)

# Post-mortem debugging
try:
    buggy_function()
except Exception:
    import pdb
    pdb.post_mortem()  # Debugger на месте падения

# Из командной строки
# python -m pdb script.py

# Использование pdb
"""
(Pdb) n                    # Следующая строка
(Pdb) s                    # Войти в функцию
(Pdb) p variable           # Напечатать переменную
(Pdb) pp complex_object    # Pretty print
(Pdb) l                    # Показать код вокруг
(Pdb) l 10, 20             # Показать строки 10-20
(Pdb) w                    # Stack trace
(Pdb) u                    # Вверх по стеку
(Pdb) d                    # Вниз по стеку
(Pdb) b 42                 # Breakpoint на строке 42
(Pdb) b func               # Breakpoint на функции
(Pdb) b file.py:42         # Breakpoint в другом файле
(Pdb) cl                   # Очистить breakpoints
(Pdb) c                    # Продолжить
(Pdb) q                    # Выйти
"""

# ipdb - улучшенный pdb
# pip install ipdb
import ipdb
ipdb.set_trace()

# Или через переменную окружения
# PYTHONBREAKPOINT=ipdb.set_trace python script.py

# Отключить все breakpoints
# PYTHONBREAKPOINT=0 python script.py

# Remote debugging (для серверов)
import pdb
import socket

class RemotePdb(pdb.Pdb):
    def __init__(self, host='0.0.0.0', port=4444):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.sock.bind((host, port))
        self.sock.listen(1)
        conn, addr = self.sock.accept()
        self.handle = conn.makefile('rw')
        pdb.Pdb.__init__(self, stdin=self.handle, stdout=self.handle)

# Или используйте remote-pdb
# pip install remote-pdb
# from remote_pdb import set_trace
# set_trace(host='0.0.0.0', port=4444)
# nc localhost 4444

# Debugging в production (осторожно!)
import sys
import traceback

def debug_on_error(type, value, tb):
    if hasattr(sys, 'ps1') or not sys.stderr.isatty():
        # Non-interactive
        sys.__excepthook__(type, value, tb)
    else:
        # Interactive
        traceback.print_exception(type, value, tb)
        pdb.post_mortem(tb)

sys.excepthook = debug_on_error

# Print debugging (иногда достаточно)
def debug_print(*args, **kwargs):
    import inspect
    frame = inspect.currentframe().f_back
    print(f"[{frame.f_code.co_filename}:{frame.f_lineno}]", *args, **kwargs)

# icecream для удобного print debugging
# pip install icecream
from icecream import ic

x = 42
ic(x)  # ic| x: 42

def foo(x):
    return x + 1
ic(foo(10))  # ic| foo(10): 11
```

### Типичные ошибки
- Оставлять breakpoint() в production коде
- Не знать команды pdb
- Использовать только print() для debugging

### На интервью
**Как отвечать:** Покажите breakpoint() и основные команды pdb (n, s, c, p), упомяните post-mortem debugging.

**Follow-up вопросы:**
- Как дебажить удалённый сервер?
- Чем pdb отличается от ipdb?
- Как отключить все breakpoints?

---

[← Назад к списку тем](README.md)
