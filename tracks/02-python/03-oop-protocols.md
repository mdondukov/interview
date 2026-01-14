# 03. ООП и протоколы

[← Назад к списку тем](README.md)

---

## Вопрос 1: Что такое MRO и как работает множественное наследование?

### Зачем спрашивают
MRO (Method Resolution Order) — критично для понимания наследования в Python. Часто источник неожиданного поведения.

### Короткий ответ
MRO определяет порядок поиска методов при множественном наследовании. Python использует алгоритм C3 linearization, который гарантирует: потомок перед родителем, порядок родителей сохраняется. Посмотреть MRO можно через `ClassName.__mro__` или `ClassName.mro()`.

### Детальный разбор

**C3 Linearization правила:**
1. Класс всегда перед своими родителями
2. Родители сохраняют порядок объявления
3. Общие предки появляются только один раз (в конце)

**Diamond problem:**
```
      A
     / \
    B   C
     \ /
      D
```
MRO для D: `[D, B, C, A, object]`

**`super()` использует MRO:**
```python
super()  # Следующий класс в MRO, не обязательно родитель!
```

### Пример

```python
# Простое наследование
class A:
    def method(self):
        return 'A'

class B(A):
    def method(self):
        return 'B'

print(B.__mro__)  # (<class 'B'>, <class 'A'>, <class 'object'>)

# Diamond problem
class A:
    def method(self):
        print('A', end=' ')

class B(A):
    def method(self):
        print('B', end=' ')
        super().method()

class C(A):
    def method(self):
        print('C', end=' ')
        super().method()

class D(B, C):
    def method(self):
        print('D', end=' ')
        super().method()

d = D()
d.method()  # D B C A
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

# super() следует MRO, не родителю!
class X:
    def __init__(self):
        print('X init')

class Y:
    def __init__(self):
        print('Y init')

class Z(X, Y):
    def __init__(self):
        print('Z init')
        super().__init__()  # Вызывает X.__init__, не Y!

Z()  # Z init, X init (Y не вызван!)

# Правильный паттерн: cooperative inheritance
class X:
    def __init__(self, **kwargs):
        print('X init')
        super().__init__(**kwargs)

class Y:
    def __init__(self, **kwargs):
        print('Y init')
        super().__init__(**kwargs)

class Z(X, Y):
    def __init__(self, **kwargs):
        print('Z init')
        super().__init__(**kwargs)

Z()  # Z init, X init, Y init (все вызваны!)

# Проверка MRO
print(Z.mro())

# Невалидный MRO (C3 не может разрешить)
# class A(X, Y): pass
# class B(Y, X): pass
# class C(A, B): pass  # TypeError: Cannot create a consistent MRO
```

### Типичные ошибки
- Думать, что `super()` всегда вызывает родителя
- Не использовать `**kwargs` для cooperative inheritance
- Создавать конфликтующий MRO

### На интервью
**Как отвечать:** Объясните C3 linearization, покажите diamond problem, продемонстрируйте что `super()` следует MRO.

**Follow-up вопросы:**
- Когда MRO невозможен?
- Как работает super() без аргументов?
- Чем отличается от C++ множественного наследования?

---

## Вопрос 2: Что такое дескрипторы?

### Зачем спрашивают
Дескрипторы — фундамент Python OOP. На них построены property, classmethod, staticmethod.

### Короткий ответ
Дескриптор — объект, который определяет методы `__get__`, `__set__`, и/или `__delete__`. Контролирует доступ к атрибуту класса. Data descriptor (с `__set__` или `__delete__`) имеет приоритет над `__dict__` инстанса. Non-data descriptor (только `__get__`) — нет.

### Детальный разбор

**Типы дескрипторов:**
- **Data descriptor:** `__get__` + (`__set__` или `__delete__`)
- **Non-data descriptor:** только `__get__`

**Порядок поиска атрибута:**
1. Data descriptor в классе
2. Instance `__dict__`
3. Non-data descriptor в классе
4. Class `__dict__`
5. `__getattr__` (если определён)

**Сигнатуры методов:**
```python
def __get__(self, obj, objtype=None): ...
def __set__(self, obj, value): ...
def __delete__(self, obj): ...
```

### Пример

```python
# Простой дескриптор
class Verbose:
    def __get__(self, obj, objtype=None):
        print(f"Getting from {obj}")
        return 42

    def __set__(self, obj, value):
        print(f"Setting {value} on {obj}")

class MyClass:
    attr = Verbose()

m = MyClass()
print(m.attr)   # Getting from <MyClass>, 42
m.attr = 100    # Setting 100 on <MyClass>

# Дескриптор с хранением в instance
class TypedAttribute:
    def __init__(self, name, expected_type):
        self.name = name
        self.expected_type = expected_type

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(f"Expected {self.expected_type}")
        obj.__dict__[self.name] = value

class Person:
    name = TypedAttribute('name', str)
    age = TypedAttribute('age', int)

p = Person()
p.name = "Alice"  # OK
p.age = 30        # OK
# p.age = "thirty"  # TypeError!

# Non-data vs Data descriptor
class NonData:
    def __get__(self, obj, objtype=None):
        return "non-data"

class Data:
    def __get__(self, obj, objtype=None):
        return "data"
    def __set__(self, obj, value):
        pass

class Test:
    nd = NonData()
    d = Data()

t = Test()
print(t.nd)  # "non-data"
print(t.d)   # "data"

t.__dict__['nd'] = 'instance'
t.__dict__['d'] = 'instance'

print(t.nd)  # "instance" (instance dict wins for non-data)
print(t.d)   # "data" (data descriptor wins!)

# Реализация property через дескриптор
class Property:
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("Can't get")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("Can't set")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("Can't delete")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)
```

### Типичные ошибки
- Путать data и non-data descriptors
- Забывать хранить данные в instance `__dict__`
- Не обрабатывать `obj is None` (доступ через класс)

### На интервью
**Как отвечать:** Объясните разницу между data и non-data дескрипторами, покажите порядок поиска атрибутов, приведите пример с типизацией.

**Follow-up вопросы:**
- Как property работает через дескрипторы?
- Почему функции — non-data descriptors?
- Как реализовать cached property?

---

## Вопрос 3: Что такое Protocol и чем он отличается от ABC?

### Зачем спрашивают
Protocol — современный подход к duck typing с поддержкой статической проверки. Важно для typed Python.

### Короткий ответ
Protocol (PEP 544) — structural subtyping для Python. Класс соответствует Protocol если имеет нужные методы, без явного наследования. ABC (Abstract Base Class) — nominal subtyping, требует явного наследования. Protocol проверяется статически (mypy), ABC — в runtime.

### Детальный разбор

**Сравнение:**

| Аспект | Protocol | ABC |
|--------|----------|-----|
| Наследование | Не требуется | Обязательно |
| Проверка | Статическая (mypy) | Runtime |
| Duck typing | Да | Нет |
| Регистрация | Нет | `register()` |

**Когда использовать:**
- **Protocol** — duck typing с типами, внешние библиотеки
- **ABC** — контракт с обязательной реализацией, свой код

### Пример

```python
from typing import Protocol, runtime_checkable
from abc import ABC, abstractmethod

# ABC подход (nominal subtyping)
class Drawable(ABC):
    @abstractmethod
    def draw(self) -> None:
        pass

class Circle(Drawable):  # Явное наследование обязательно!
    def draw(self) -> None:
        print("Drawing circle")

# Это НЕ Drawable, хотя имеет draw()
class Square:
    def draw(self) -> None:
        print("Drawing square")

# isinstance работает только с наследованием
print(isinstance(Circle(), Drawable))  # True
print(isinstance(Square(), Drawable))  # False

# Protocol подход (structural subtyping)
class Renderable(Protocol):
    def render(self) -> str: ...

class HTMLRenderer:
    def render(self) -> str:
        return "<html>...</html>"

class JSONRenderer:
    def render(self) -> str:
        return '{"data": ...}'

# Оба соответствуют Renderable без наследования!
def process(r: Renderable) -> None:
    print(r.render())

process(HTMLRenderer())  # OK
process(JSONRenderer())  # OK

# runtime_checkable для isinstance()
@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

class File:
    def close(self) -> None:
        print("Closing file")

print(isinstance(File(), Closeable))  # True

# Сложный Protocol
class Repository(Protocol):
    def get(self, id: int) -> dict: ...
    def save(self, data: dict) -> int: ...
    def delete(self, id: int) -> bool: ...

class InMemoryRepo:
    def __init__(self):
        self._data = {}
        self._counter = 0

    def get(self, id: int) -> dict:
        return self._data.get(id, {})

    def save(self, data: dict) -> int:
        self._counter += 1
        self._data[self._counter] = data
        return self._counter

    def delete(self, id: int) -> bool:
        return self._data.pop(id, None) is not None

def use_repo(repo: Repository) -> None:
    id = repo.save({"name": "test"})
    print(repo.get(id))

use_repo(InMemoryRepo())  # Работает!

# Protocol с атрибутами
class Named(Protocol):
    name: str

class Person:
    def __init__(self, name: str):
        self.name = name

def greet(obj: Named) -> str:
    return f"Hello, {obj.name}"

greet(Person("Alice"))  # OK

# Callable Protocol
from typing import Callable

class Handler(Protocol):
    def __call__(self, request: dict) -> dict: ...

def process_request(handler: Handler, request: dict) -> dict:
    return handler(request)
```

### Типичные ошибки
- Использовать ABC где достаточно Protocol
- Забывать `@runtime_checkable` для isinstance()
- Не указывать `...` в методах Protocol

### На интервью
**Как отвечать:** Объясните structural vs nominal typing, покажите что Protocol не требует наследования, упомяните `@runtime_checkable`.

**Follow-up вопросы:**
- Когда Protocol лучше ABC?
- Как Protocol работает с generics?
- Что делает @runtime_checkable?

---

## Вопрос 4: Как работают dataclass и какие есть параметры?

### Зачем спрашивают
Dataclass — современный способ создания data-классов. Проверяют знание Python 3.7+ фич.

### Короткий ответ
`@dataclass` автоматически генерирует `__init__`, `__repr__`, `__eq__` и другие методы на основе аннотаций типов. Параметры контролируют генерацию: `frozen=True` для immutability, `order=True` для сравнения, `slots=True` для оптимизации памяти (Python 3.10+).

### Детальный разбор

**Параметры @dataclass:**
```python
@dataclass(
    init=True,       # Генерировать __init__
    repr=True,       # Генерировать __repr__
    eq=True,         # Генерировать __eq__
    order=False,     # Генерировать __lt__, __le__, __gt__, __ge__
    unsafe_hash=False,  # Генерировать __hash__
    frozen=False,    # Immutable (как tuple)
    match_args=True, # Для pattern matching (3.10+)
    kw_only=False,   # Все поля keyword-only (3.10+)
    slots=False,     # Использовать __slots__ (3.10+)
)
```

**field() параметры:**
```python
field(
    default=MISSING,      # Значение по умолчанию
    default_factory=...,  # Фабрика для mutable defaults
    repr=True,            # Включать в __repr__
    hash=None,            # Включать в __hash__
    compare=True,         # Использовать в сравнении
    metadata=None,        # Дополнительные данные
    kw_only=False,        # Keyword-only (3.10+)
)
```

### Пример

```python
from dataclasses import dataclass, field, asdict, astuple
from typing import List

# Базовый dataclass
@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
print(p)           # Point(x=1.0, y=2.0)
print(p == Point(1.0, 2.0))  # True

# С default values
@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    debug: bool = False

c = Config()
print(c)  # Config(host='localhost', port=8080, debug=False)

# Mutable default - ОШИБКА!
# @dataclass
# class Bad:
#     items: list = []  # ValueError!

# Правильно: default_factory
@dataclass
class Container:
    items: List[int] = field(default_factory=list)

# frozen для immutability
@dataclass(frozen=True)
class FrozenPoint:
    x: float
    y: float

fp = FrozenPoint(1.0, 2.0)
# fp.x = 3.0  # FrozenDataclassError!
print(hash(fp))  # Можно использовать как ключ dict

# order для сравнения
@dataclass(order=True)
class Version:
    major: int
    minor: int
    patch: int

versions = [Version(1, 0, 0), Version(2, 0, 0), Version(1, 5, 0)]
print(sorted(versions))  # [Version(1, 0, 0), Version(1, 5, 0), Version(2, 0, 0)]

# slots для оптимизации (Python 3.10+)
@dataclass(slots=True)
class OptimizedPoint:
    x: float
    y: float
# Меньше памяти, быстрее доступ

# field() для тонкой настройки
@dataclass
class User:
    name: str
    email: str
    password: str = field(repr=False)  # Не показывать в repr
    id: int = field(default_factory=lambda: id(object()))
    created_at: float = field(default_factory=time.time, compare=False)  # import time

# Post-init обработка
@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)  # Не в __init__

    def __post_init__(self):
        self.area = self.width * self.height

r = Rectangle(3, 4)
print(r.area)  # 12

# Наследование
@dataclass
class Point3D(Point):
    z: float = 0.0

p3 = Point3D(1.0, 2.0, 3.0)

# Конвертация
@dataclass
class Person:
    name: str
    age: int

p = Person("Alice", 30)
print(asdict(p))   # {'name': 'Alice', 'age': 30}
print(astuple(p))  # ('Alice', 30)

# kw_only (Python 3.10+)
@dataclass(kw_only=True)
class Settings:
    debug: bool
    verbose: bool

s = Settings(debug=True, verbose=False)  # Только kwargs
```

### Типичные ошибки
- Использовать mutable default без `default_factory`
- Забывать про `frozen=True` для hashable объектов
- Не знать про `__post_init__`

### На интервью
**Как отвечать:** Покажите базовый пример, объясните ключевые параметры (frozen, order, slots), упомяните `field()` и `__post_init__`.

**Follow-up вопросы:**
- Чем dataclass отличается от NamedTuple?
- Как работает наследование dataclass?
- Когда использовать Pydantic вместо dataclass?

---

## Вопрос 5: Что такое `__slots__` и когда их использовать?

### Зачем спрашивают
`__slots__` — оптимизация памяти и скорости. Важно для высоконагруженных систем.

### Короткий ответ
`__slots__` — объявление фиксированного набора атрибутов класса. Отключает `__dict__` и `__weakref__`, экономит ~40-50% памяти на инстанс. Ускоряет доступ к атрибутам. Ограничивает добавление новых атрибутов.

### Детальный разбор

**Как работает:**
```python
# Обычный класс: каждый инстанс имеет __dict__
class Normal:
    def __init__(self, x, y):
        self.x = x
        self.y = y
# Каждый объект: 56 bytes (__dict__) + атрибуты

# Со slots: фиксированная структура
class Slotted:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y
# Каждый объект: только атрибуты (меньше памяти)
```

**Ограничения:**
- Нельзя добавлять новые атрибуты
- Нет `__dict__` (если явно не добавить в slots)
- Нет weak references (если `__weakref__` нет в slots)
- Наследование требует осторожности

### Пример

```python
import sys

# Сравнение памяти
class Normal:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class Slotted:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

n = Normal(1, 2)
s = Slotted(1, 2)

print(sys.getsizeof(n))  # 48
print(sys.getsizeof(s))  # 48
print(sys.getsizeof(n.__dict__))  # 104 (дополнительно!)

# Проверка наличия __dict__
print(hasattr(n, '__dict__'))  # True
print(hasattr(s, '__dict__'))  # False

# Нельзя добавить атрибут
# s.z = 3  # AttributeError!

# Массовое создание объектов
import tracemalloc
tracemalloc.start()

normal_objects = [Normal(i, i) for i in range(100000)]
print(f"Normal: {tracemalloc.get_traced_memory()[0] / 1024 / 1024:.2f} MB")

tracemalloc.reset_peak()
slotted_objects = [Slotted(i, i) for i in range(100000)]
print(f"Slotted: {tracemalloc.get_traced_memory()[0] / 1024 / 1024:.2f} MB")

# Наследование со slots
class Base:
    __slots__ = ('x',)

class Derived(Base):
    __slots__ = ('y',)  # Добавляем только новые атрибуты!

d = Derived()
d.x = 1
d.y = 2

# Разрешить __dict__ явно
class Flexible:
    __slots__ = ('x', 'y', '__dict__')

f = Flexible()
f.x = 1
f.z = 3  # Теперь можно!

# С dataclass (Python 3.10+)
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: float
    y: float

# Slots и property
class Circle:
    __slots__ = ('_radius',)

    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be positive")
        self._radius = value

# Slots и дескрипторы работают вместе
```

### Типичные ошибки
- Добавлять родительские атрибуты в дочерние `__slots__`
- Забывать про `__weakref__` если нужны weak references
- Использовать slots без реальной необходимости

### На интервью
**Как отвечать:** Объясните экономию памяти, покажите ограничения, упомяните `@dataclass(slots=True)` для Python 3.10+.

**Follow-up вопросы:**
- Когда НЕ использовать slots?
- Как работает наследование с slots?
- Можно ли добавить __dict__ в slots?

---

## Вопрос 6: Что такое property и как его использовать?

### Зачем спрашивают
Property — основной способ инкапсуляции в Python. Проверяют понимание дескрипторов.

### Короткий ответ
`@property` превращает метод в "виртуальный" атрибут с контролируемым доступом. Позволяет добавить валидацию, вычисление или логирование без изменения интерфейса класса. Под капотом — data descriptor.

### Детальный разбор

**Синтаксис:**
```python
class C:
    @property
    def x(self):
        """Getter"""
        return self._x

    @x.setter
    def x(self, value):
        """Setter"""
        self._x = value

    @x.deleter
    def x(self):
        """Deleter"""
        del self._x
```

**Когда использовать:**
- Валидация при установке значения
- Вычисляемые атрибуты
- Ленивая инициализация
- Логирование доступа
- Обратная совместимость (атрибут → property)

### Пример

```python
# Базовый property
class Circle:
    def __init__(self, radius):
        self._radius = radius  # "приватный" атрибут

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value

    @property
    def area(self):
        """Read-only вычисляемый атрибут"""
        return 3.14159 * self._radius ** 2

c = Circle(5)
print(c.radius)  # 5
print(c.area)    # 78.53975
c.radius = 10    # OK
# c.area = 100   # AttributeError (no setter)
# c.radius = -1  # ValueError

# Cached property (ленивое вычисление)
class ExpensiveComputation:
    def __init__(self, data):
        self.data = data
        self._result = None

    @property
    def result(self):
        if self._result is None:
            print("Computing...")
            self._result = sum(self.data)  # Дорогая операция
        return self._result

e = ExpensiveComputation([1, 2, 3, 4, 5])
print(e.result)  # Computing... 15
print(e.result)  # 15 (без вычисления)

# functools.cached_property (Python 3.8+)
from functools import cached_property

class Data:
    def __init__(self, items):
        self.items = items

    @cached_property
    def processed(self):
        print("Processing...")
        return [x * 2 for x in self.items]

d = Data([1, 2, 3])
print(d.processed)  # Processing... [2, 4, 6]
print(d.processed)  # [2, 4, 6] (кэшировано)
del d.processed     # Сбросить кэш
print(d.processed)  # Processing... (пересчитано)

# Property для обратной совместимости
class User:
    def __init__(self, first_name, last_name):
        self.first_name = first_name
        self.last_name = last_name
        # Раньше был self.name = first_name + " " + last_name

    @property
    def name(self):
        """Обратная совместимость"""
        return f"{self.first_name} {self.last_name}"

    @name.setter
    def name(self, value):
        parts = value.split(maxsplit=1)
        self.first_name = parts[0]
        self.last_name = parts[1] if len(parts) > 1 else ""

u = User("John", "Doe")
print(u.name)       # John Doe
u.name = "Jane Smith"
print(u.first_name) # Jane

# Property без декоратора
class Temperature:
    def __init__(self):
        self._celsius = 0

    def get_fahrenheit(self):
        return self._celsius * 9/5 + 32

    def set_fahrenheit(self, value):
        self._celsius = (value - 32) * 5/9

    fahrenheit = property(get_fahrenheit, set_fahrenheit)

t = Temperature()
t.fahrenheit = 100
print(t._celsius)  # 37.78
```

### Типичные ошибки
- Использовать property для тяжёлых вычислений без кэширования
- Забывать про @setter (создаёт read-only property)
- Не использовать underscore для backing attribute

### На интервью
**Как отвечать:** Покажите базовый пример с валидацией, объясните read-only property, упомяните `cached_property`.

**Follow-up вопросы:**
- Как property работает под капотом?
- Чем отличается cached_property от обычного property?
- Можно ли наследовать property?

---

## Вопрос 7: Как работают classmethod и staticmethod?

### Зачем спрашивают
Базовые декораторы, но часто путают. Проверяют понимание методов класса.

### Короткий ответ
`@staticmethod` — функция в пространстве имён класса, не получает ни `self`, ни `cls`. `@classmethod` — метод, получающий класс как первый аргумент (`cls`), работает с классом, а не инстансом. Оба могут вызываться через класс или инстанс.

### Детальный разбор

**Сравнение:**

| Тип | Первый аргумент | Доступ к классу | Доступ к инстансу |
|-----|-----------------|-----------------|-------------------|
| Instance method | `self` | Через `type(self)` | Да |
| Class method | `cls` | Да | Нет |
| Static method | Нет | Через явное имя | Нет |

**Когда использовать:**
- **staticmethod** — утилитарная функция, связанная с классом концептуально
- **classmethod** — альтернативные конструкторы, работа с class-level данными

### Пример

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    # Instance method
    def __str__(self):
        return f"{self.year}-{self.month:02d}-{self.day:02d}"

    # Class method - альтернативный конструктор
    @classmethod
    def from_string(cls, date_string):
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)  # cls, не Date!

    @classmethod
    def today(cls):
        import datetime
        d = datetime.date.today()
        return cls(d.year, d.month, d.day)

    # Static method - утилитарная функция
    @staticmethod
    def is_valid_date(year, month, day):
        return 1 <= month <= 12 and 1 <= day <= 31

# Использование
d1 = Date(2024, 1, 15)
d2 = Date.from_string("2024-06-20")
d3 = Date.today()
print(Date.is_valid_date(2024, 13, 1))  # False

# Наследование - classmethod работает правильно
class DateTime(Date):
    def __init__(self, year, month, day, hour=0, minute=0):
        super().__init__(year, month, day)
        self.hour = hour
        self.minute = minute

dt = DateTime.from_string("2024-06-20")  # Возвращает DateTime!
print(type(dt))  # <class 'DateTime'>

# Паттерн: регистрация подклассов
class Handler:
    _handlers = {}

    @classmethod
    def register(cls, name):
        def decorator(handler_cls):
            cls._handlers[name] = handler_cls
            return handler_cls
        return decorator

    @classmethod
    def get_handler(cls, name):
        return cls._handlers.get(name)

@Handler.register('json')
class JSONHandler(Handler):
    pass

@Handler.register('xml')
class XMLHandler(Handler):
    pass

handler = Handler.get_handler('json')

# staticmethod vs обычная функция
class MathUtils:
    @staticmethod
    def add(a, b):
        return a + b

# Почти то же самое, но:
# 1. Логически связано с классом
# 2. Можно переопределить в подклассе
# 3. Доступно через self.add(1, 2)

# classmethod как factory с валидацией
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    @classmethod
    def from_dict(cls, data):
        if 'name' not in data or 'age' not in data:
            raise ValueError("Missing required fields")
        return cls(data['name'], data['age'])

    @classmethod
    def from_json(cls, json_string):
        import json
        return cls.from_dict(json.loads(json_string))
```

### Типичные ошибки
- Использовать staticmethod где нужен classmethod (потеря полиморфизма)
- Использовать classmethod для простых утилит
- Забывать про `cls` вместо конкретного имени класса

### На интервью
**Как отвечать:** Объясните разницу через первый аргумент, покажите альтернативные конструкторы для classmethod, упомяните наследование.

**Follow-up вопросы:**
- Когда staticmethod лучше обычной функции?
- Почему from_string использует cls, а не Date?
- Как работает classmethod под капотом?

---

## Вопрос 8: Что такое mixins и когда их использовать?

### Зачем спрашивают
Mixins — паттерн для переиспользования кода. Показывает понимание композиции vs наследования.

### Короткий ответ
Mixin — класс, предоставляющий дополнительную функциональность без полноценного наследования. Не предназначен для самостоятельного использования. Добавляет методы к классу через множественное наследование. Соглашение: имя заканчивается на `Mixin`.

### Детальный разбор

**Правила для mixins:**
1. Не иметь собственного состояния (или минимальное)
2. Не переопределять методы родителя
3. Предоставлять только дополнительную функциональность
4. Не вызывать `super().__init__()` (или делать это осторожно)

**Когда использовать:**
- Общая функциональность для несвязанных классов
- Добавление возможностей без глубокой иерархии
- Альтернатива множественному наследованию от "настоящих" классов

### Пример

```python
# Logging mixin
class LoggingMixin:
    def log(self, message):
        print(f"[{self.__class__.__name__}] {message}")

class Service(LoggingMixin):
    def process(self):
        self.log("Processing started")
        # ... логика ...
        self.log("Processing completed")

s = Service()
s.process()
# [Service] Processing started
# [Service] Processing completed

# Serialization mixin
import json

class JSONSerializableMixin:
    def to_json(self):
        return json.dumps(self.__dict__)

    @classmethod
    def from_json(cls, json_str):
        data = json.loads(json_str)
        obj = cls.__new__(cls)
        obj.__dict__.update(data)
        return obj

class User(JSONSerializableMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age

u = User("Alice", 30)
json_str = u.to_json()  # '{"name": "Alice", "age": 30}'
u2 = User.from_json(json_str)

# Comparison mixin
class ComparableMixin:
    """Требует реализации __lt__ в классе"""
    def __le__(self, other):
        return self == other or self < other

    def __gt__(self, other):
        return not self <= other

    def __ge__(self, other):
        return not self < other

class Version(ComparableMixin):
    def __init__(self, major, minor, patch):
        self.major = major
        self.minor = minor
        self.patch = patch

    def __eq__(self, other):
        return (self.major, self.minor, self.patch) == \
               (other.major, other.minor, other.patch)

    def __lt__(self, other):
        return (self.major, self.minor, self.patch) < \
               (other.major, other.minor, other.patch)

v1 = Version(1, 0, 0)
v2 = Version(2, 0, 0)
print(v1 < v2)   # True
print(v1 <= v2)  # True (из mixin)
print(v1 >= v2)  # False (из mixin)

# Множественные mixins
class TimestampMixin:
    def touch(self):
        import time
        self.updated_at = time.time()

class ValidationMixin:
    def validate(self):
        for field, rules in getattr(self, '_validation_rules', {}).items():
            value = getattr(self, field, None)
            for rule in rules:
                if not rule(value):
                    raise ValueError(f"Validation failed for {field}")
        return True

class Document(JSONSerializableMixin, TimestampMixin, ValidationMixin):
    _validation_rules = {
        'title': [lambda x: x and len(x) > 0],
    }

    def __init__(self, title, content):
        self.title = title
        self.content = content

doc = Document("Hello", "World")
doc.validate()
doc.touch()
print(doc.to_json())

# Django-style mixins
class CreateMixin:
    def create(self, data):
        return self.model(**data)

class UpdateMixin:
    def update(self, instance, data):
        for key, value in data.items():
            setattr(instance, key, value)
        return instance

class DeleteMixin:
    def delete(self, instance):
        # delete logic
        pass

class CRUDService(CreateMixin, UpdateMixin, DeleteMixin):
    model = User
```

### Типичные ошибки
- Mixin с собственным `__init__` и состоянием
- Порядок mixins в наследовании (MRO)
- Конфликты имён методов

### На интервью
**Как отвечать:** Объясните концепцию "добавления возможностей", покажите примеры (сериализация, логирование), упомяните правила хороших mixins.

**Follow-up вопросы:**
- Чем mixin отличается от абстрактного класса?
- Как избежать конфликтов имён?
- Когда лучше использовать композицию?

---

## См. также

- [Принципы ООП в Java](../01-java/01-oop-principles.md) — сравнение ООП подходов в Python и Java
- [Чистая архитектура](../08-architecture/01-clean-architecture.md) — применение ООП принципов в архитектуре

---

[← Назад к списку тем](README.md)
