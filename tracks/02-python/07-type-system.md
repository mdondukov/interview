# 07. Система типов

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работает typing module?

### Зачем спрашивают
Type hints — стандарт в современном Python. Проверяют знание типизации.

### Короткий ответ
`typing` модуль предоставляет типы для аннотаций: `List`, `Dict`, `Optional`, `Union`, `Callable` и др. Аннотации проверяются статически (mypy), не в runtime. С Python 3.9+ можно использовать встроенные типы (`list[int]` вместо `List[int]`).

### Пример

```python
from typing import List, Dict, Optional, Union, Callable, Any, Tuple

# Базовые аннотации
def greet(name: str) -> str:
    return f"Hello, {name}"

# Коллекции
def process(items: List[int]) -> Dict[str, int]:
    return {"sum": sum(items)}

# Python 3.9+ - встроенные типы
def process_modern(items: list[int]) -> dict[str, int]:
    return {"sum": sum(items)}

# Optional = Union[X, None]
def find_user(id: int) -> Optional[User]:
    return users.get(id)  # Может вернуть None

# Union - несколько типов
def parse(data: Union[str, bytes]) -> dict:
    if isinstance(data, bytes):
        data = data.decode()
    return json.loads(data)

# Python 3.10+ - | вместо Union
def parse_modern(data: str | bytes) -> dict:
    ...

# Callable
def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# Any - отключает проверку
def process_any(data: Any) -> Any:
    return data

# Type aliases
UserId = int
UserDict = Dict[str, Any]

def get_user(id: UserId) -> UserDict:
    ...

# TypeAlias (explicit, Python 3.10+)
from typing import TypeAlias
Vector: TypeAlias = list[float]
```

### На интервью
**Как отвечать:** Покажите базовые типы, объясните Optional vs Union, упомяните Python 3.9+ синтаксис.

---

## Вопрос 2: Что такое TypeVar и Generic?

### Зачем спрашивают
Generics — продвинутая типизация. Показывает глубокое понимание системы типов.

### Короткий ответ
`TypeVar` создаёт переменную типа для generic функций/классов. `Generic[T]` — базовый класс для generic классов. Позволяет писать код, работающий с разными типами, сохраняя type safety.

### Пример

```python
from typing import TypeVar, Generic, List, Sequence

# TypeVar для функций
T = TypeVar('T')

def first(items: List[T]) -> T:
    return items[0]

# Вызов сохраняет тип
x: int = first([1, 2, 3])       # T = int
y: str = first(["a", "b", "c"]) # T = str

# Bounded TypeVar
# Note: Comparable не существует в typing, это гипотетический пример
# Можно использовать Protocol для определения Comparable
T = TypeVar('T', bound='Comparable')  # Предполагается определённый Protocol

def max_item(items: List[T]) -> T:
    return max(items)

# Constrained TypeVar
T = TypeVar('T', int, float)  # Только int или float

def add(a: T, b: T) -> T:
    return a + b

# Generic класс
T = TypeVar('T')

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: List[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

stack: Stack[int] = Stack()
stack.push(1)

# Python 3.12+ - упрощённый синтаксис
def first[T](items: list[T]) -> T:
    return items[0]

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

# Multiple TypeVars
K = TypeVar('K')
V = TypeVar('V')

class Mapping(Generic[K, V]):
    def get(self, key: K) -> V: ...
```

### На интервью
**Как отвечать:** Объясните TypeVar как параметр типа, покажите Generic класс, упомяните Python 3.12 синтаксис.

---

## Вопрос 3: Как работает Protocol для structural subtyping?

### Зачем спрашивают
Protocol — формализация duck typing. Важно для typed Python.

### Короткий ответ
Protocol (PEP 544) определяет интерфейс через методы/атрибуты. Класс соответствует Protocol если имеет нужные члены, без явного наследования. Это structural subtyping (как Go interfaces) vs nominal (как ABC).

### Пример

```python
from typing import Protocol, runtime_checkable

# Определение Protocol
class Readable(Protocol):
    def read(self) -> str: ...

class Writeable(Protocol):
    def write(self, data: str) -> None: ...

# Реализация без наследования
class File:
    def read(self) -> str:
        return "content"
    def write(self, data: str) -> None:
        pass

def process(reader: Readable) -> str:
    return reader.read()

# File соответствует Readable без наследования!
process(File())

# runtime_checkable для isinstance
@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

print(isinstance(File(), Readable))  # Error без @runtime_checkable

# Protocol с атрибутами
class Named(Protocol):
    name: str

class User:
    def __init__(self, name: str):
        self.name = name

def greet(obj: Named) -> str:
    return f"Hello, {obj.name}"

# Callable Protocol
class Handler(Protocol):
    def __call__(self, request: dict) -> dict: ...
```

### На интервью
**Как отвечать:** Объясните structural vs nominal typing, покажите что наследование не нужно, упомяните @runtime_checkable.

---

## Вопрос 4: Что такое Literal, Final, TypedDict?

### Зачем спрашивают
Продвинутые типы для специфических случаев. Показывает глубину знаний.

### Короткий ответ
`Literal` — конкретные значения как тип. `Final` — константы, нельзя переприсвоить. `TypedDict` — dict с фиксированными ключами и типами значений. Все проверяются статически.

### Пример

```python
from typing import Literal, Final, TypedDict

# Literal - конкретные значения
def set_mode(mode: Literal["read", "write"]) -> None:
    ...

set_mode("read")   # OK
set_mode("delete") # Error

# Literal с числами
def roll_dice() -> Literal[1, 2, 3, 4, 5, 6]:
    import random
    return random.randint(1, 6)

# Final - константы
MAX_SIZE: Final = 100
MAX_SIZE = 200  # Error: cannot assign to final

class Config:
    DEBUG: Final[bool] = False

# TypedDict
class UserDict(TypedDict):
    name: str
    age: int
    email: str

user: UserDict = {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com"
}

# Optional fields
class UserDict(TypedDict, total=False):
    name: str      # Required
    nickname: str  # Optional

# Required + NotRequired (Python 3.11+)
from typing import Required, NotRequired

class UserDict(TypedDict):
    name: Required[str]
    nickname: NotRequired[str]
```

### На интервью
**Как отвечать:** Покажите use cases для каждого: Literal для enum-like значений, Final для констант, TypedDict для JSON.

---

## Вопрос 5: Как работает overload декоратор?

### Зачем спрашивают
Overload — для функций с разными сигнатурами. Продвинутая типизация.

### Короткий ответ
`@overload` объявляет разные сигнатуры функции для статической проверки. Сами overload функции не выполняются — нужна реализация без @overload. Позволяет типизировать функции, чей return type зависит от аргументов.

### Пример

```python
from typing import overload, Literal, Union

# Overload декларации
@overload
def process(data: str) -> str: ...
@overload
def process(data: bytes) -> bytes: ...
@overload
def process(data: int) -> int: ...

# Реальная реализация
def process(data: Union[str, bytes, int]) -> Union[str, bytes, int]:
    if isinstance(data, str):
        return data.upper()
    elif isinstance(data, bytes):
        return data.upper()
    else:
        return data * 2

# Теперь mypy знает точные типы
s: str = process("hello")    # OK
b: bytes = process(b"hello") # OK
n: int = process(42)         # OK

# Overload с Literal
@overload
def fetch(url: str, format: Literal["json"]) -> dict: ...
@overload
def fetch(url: str, format: Literal["text"]) -> str: ...
@overload
def fetch(url: str, format: Literal["bytes"]) -> bytes: ...

def fetch(url: str, format: str) -> Union[dict, str, bytes]:
    response = requests.get(url)
    if format == "json":
        return response.json()
    elif format == "text":
        return response.text
    else:
        return response.content

# Return type зависит от аргумента
result = fetch("http://api.com", "json")  # type: dict
```

### На интервью
**Как отвечать:** Объясните что overload — только для статической проверки, покажите паттерн с Literal.

---

## Вопрос 6: В чём разница между Union и Optional?

### Зачем спрашивают
Частый источник путаницы. Базовый вопрос по типизации.

### Короткий ответ
`Optional[X]` — это `Union[X, None]`. Используйте Optional когда значение может быть None. Union — для нескольких разных типов. С Python 3.10+ используйте `X | None` и `X | Y`.

### Пример

```python
from typing import Optional, Union

# Optional = Union[X, None]
def find(id: int) -> Optional[User]:
    return users.get(id)  # User или None

# Эквивалентно
def find(id: int) -> Union[User, None]:
    ...

# Python 3.10+
def find(id: int) -> User | None:
    ...

# Union для разных типов
def process(data: Union[str, int, list]) -> str:
    ...

# Python 3.10+
def process(data: str | int | list) -> str:
    ...

# Когда что использовать
def get_user(id: int) -> Optional[User]:  # Может быть None
    ...

def parse(data: str | bytes) -> dict:  # Разные типы, не None
    ...
```

### На интервью
**Как отвечать:** Объясните что Optional — это Union с None, рекомендуйте Python 3.10+ синтаксис.

---

## Вопрос 7: Как использовать mypy для проверки типов?

### Зачем спрашивают
mypy — основной инструмент статической проверки. Практический навык.

### Короткий ответ
mypy проверяет типы без запуска кода. Запуск: `mypy package/`. Настройка через pyproject.toml или mypy.ini. Strict mode для максимальной проверки. Интегрируется с CI/CD и IDE.

### Пример

```bash
# Установка и запуск
pip install mypy
mypy mypackage/
mypy --strict mypackage/

# Игнорирование ошибок
mypy --ignore-missing-imports mypackage/
```

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_ignores = true

[[tool.mypy.overrides]]
module = "third_party.*"
ignore_missing_imports = true
```

```python
# Inline игнорирование
x: int = "string"  # type: ignore

# Reveal type для отладки
from typing import reveal_type
reveal_type(variable)  # mypy покажет тип

# Cast для подсказки
from typing import cast
x = cast(int, some_value)

# TYPE_CHECKING для import только при проверке
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from expensive_module import Type
```

### На интервью
**Как отвечать:** Покажите базовую настройку, объясните strict mode, упомяните reveal_type для отладки.

---

## См. также

- [Generics в Java](../01-java/03-generics.md) — система дженериков в Java
- [Обзор языка Go](../00-go/00-language-overview.md) — статическая типизация в Go

---

[← Назад к списку тем](README.md)
