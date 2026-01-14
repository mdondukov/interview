# 12. Метапрограммирование

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работают декораторы?

### Зачем спрашивают
Декораторы — фундамент Python. Используются повсеместно.

### Короткий ответ
Декоратор — функция, принимающая функцию и возвращающая новую функцию. `@decorator` — синтаксический сахар для `func = decorator(func)`. Сохраняют метаданные через `@functools.wraps`. Могут принимать аргументы.

### Пример

```python
import functools

# Простой декоратор
def my_decorator(func):
    @functools.wraps(func)  # Сохраняет __name__, __doc__
    def wrapper(*args, **kwargs):
        print("Before")
        result = func(*args, **kwargs)
        print("After")
        return result
    return wrapper

@my_decorator
def say_hello(name):
    """Says hello"""
    print(f"Hello, {name}")

# Эквивалентно:
# say_hello = my_decorator(say_hello)

# Декоратор с аргументами
def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet():
    print("Hello!")

# Декоратор для классов
def singleton(cls):
    instances = {}
    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Database:
    pass

# Практические примеры
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__}: {time.time() - start:.2f}s")
        return result
    return wrapper

def retry(times=3, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == times - 1:
                        raise
        return wrapper
    return decorator
```

### На интервью
**Как отвечать:** Покажите базовый декоратор, объясните @functools.wraps, покажите декоратор с аргументами.

---

## Вопрос 2: Как написать декоратор с параметрами?

### Зачем спрашивают
Более сложный паттерн. Показывает глубокое понимание.

### Короткий ответ
Декоратор с параметрами — это функция, возвращающая декоратор. Три уровня вложенности: внешняя функция (параметры) → декоратор (функция) → wrapper (вызов). Альтернатива — класс-декоратор.

### Пример

```python
import functools

# Функциональный подход (3 уровня)
def retry(times=3, delay=1, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            import time
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt < times - 1:
                        time.sleep(delay)
                    else:
                        raise
        return wrapper
    return decorator

@retry(times=5, delay=2)
def fetch_data():
    ...

# Класс-декоратор (более читаемый)
class Retry:
    def __init__(self, times=3, delay=1):
        self.times = times
        self.delay = delay

    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            import time
            for attempt in range(self.times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt < self.times - 1:
                        time.sleep(self.delay)
                    else:
                        raise
        return wrapper

@Retry(times=5, delay=2)
def fetch_data():
    ...

# Декоратор с опциональными параметрами
def decorator(func=None, *, option=True):
    def actual_decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            if option:
                print("Option enabled")
            return func(*args, **kwargs)
        return wrapper

    if func is not None:
        return actual_decorator(func)
    return actual_decorator

@decorator  # Без параметров
def func1(): pass

@decorator()  # С пустыми скобками
def func2(): pass

@decorator(option=False)  # С параметром
def func3(): pass
```

### На интервью
**Как отвечать:** Покажите 3 уровня вложенности, объясните когда использовать класс-декоратор.

---

## Вопрос 3: Что такое метаклассы и когда их использовать?

### Зачем спрашивают
Метаклассы — продвинутая тема. Показывает глубокое понимание Python.

### Короткий ответ
Метакласс — "класс класса". Контролирует создание классов. `type` — базовый метакласс. Используйте для: автоматической регистрации классов, валидации, изменения атрибутов класса. В большинстве случаев достаточно декораторов классов или `__init_subclass__`.

### Пример

```python
# type — метакласс по умолчанию
class MyClass:
    pass

# Эквивалентно:
MyClass = type('MyClass', (), {})

# Кастомный метакласс
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        print(f"Creating class: {name}")
        # Модификация класса
        namespace['class_id'] = id(name)
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=Meta):
    pass

print(MyClass.class_id)

# Практический пример: автоматическая регистрация
class PluginMeta(type):
    plugins = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if name != 'Plugin':  # Не регистрируем базовый класс
            mcs.plugins[name] = cls
        return cls

class Plugin(metaclass=PluginMeta):
    pass

class JSONPlugin(Plugin):
    pass

class XMLPlugin(Plugin):
    pass

print(PluginMeta.plugins)  # {'JSONPlugin': ..., 'XMLPlugin': ...}

# Альтернатива: __init_subclass__ (проще!)
class Plugin:
    plugins = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.plugins[cls.__name__] = cls

class JSONPlugin(Plugin):
    pass

# Когда использовать метаклассы:
# 1. ORM (SQLAlchemy, Django ORM)
# 2. API frameworks
# 3. Когда нужно контролировать создание класса
# 4. Singleton pattern

# Singleton через метакласс
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    pass
```

### На интервью
**Как отвечать:** Объясните что метакласс — класс класса, покажите практический пример (регистрация), упомяните `__init_subclass__` как альтернативу.

---

## Вопрос 4: Как работает `__new__` vs `__init__`?

### Зачем спрашивают
Понимание создания объектов. Важно для immutable типов и паттернов.

### Короткий ответ
`__new__` создаёт объект (allocates memory), `__init__` инициализирует его. `__new__` — classmethod, возвращает экземпляр. `__init__` — instance method, ничего не возвращает. Для immutable типов переопределяйте `__new__`.

### Пример

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        print(f"__new__ called")
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        print(f"__init__ called")
        self.value = value

obj = MyClass(42)
# __new__ called
# __init__ called

# Immutable тип (как tuple)
class Point(tuple):
    def __new__(cls, x, y):
        return super().__new__(cls, (x, y))

    @property
    def x(self):
        return self[0]

    @property
    def y(self):
        return self[1]

p = Point(1, 2)
print(p.x, p.y)  # 1, 2

# Singleton через __new__
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Factory через __new__
class Shape:
    def __new__(cls, shape_type):
        if shape_type == 'circle':
            return object.__new__(Circle)
        elif shape_type == 'square':
            return object.__new__(Square)
        return object.__new__(cls)

class Circle(Shape):
    pass

class Square(Shape):
    pass

shape = Shape('circle')
print(type(shape))  # <class 'Circle'>

# Важно: если __new__ не возвращает экземпляр cls,
# __init__ не вызывается!
```

### На интервью
**Как отвечать:** `__new__` создаёт, `__init__` инициализирует. Покажите Singleton и immutable примеры.

---

## Вопрос 5: Что такое `__getattr__` vs `__getattribute__`?

### Зачем спрашивают
Понимание доступа к атрибутам. Важно для proxy/decorator паттернов.

### Короткий ответ
`__getattribute__` вызывается для КАЖДОГО доступа к атрибуту. `__getattr__` вызывается только когда атрибут НЕ НАЙДЕН обычным способом. `__getattribute__` опасно переопределять (легко создать бесконечную рекурсию).

### Пример

```python
class MyClass:
    def __init__(self):
        self.existing = "I exist"

    def __getattr__(self, name):
        # Вызывается только если атрибут не найден
        return f"'{name}' not found"

    def __getattribute__(self, name):
        # Вызывается для КАЖДОГО доступа
        print(f"Accessing: {name}")
        return super().__getattribute__(name)

obj = MyClass()
print(obj.existing)    # Accessing: existing → I exist
print(obj.nonexistent) # Accessing: nonexistent → 'nonexistent' not found

# Proxy паттерн
class Proxy:
    def __init__(self, obj):
        # Используем object.__setattr__ чтобы избежать рекурсии
        object.__setattr__(self, '_obj', obj)

    def __getattr__(self, name):
        return getattr(self._obj, name)

    def __setattr__(self, name, value):
        setattr(self._obj, name, value)

# Lazy loading
class LazyLoader:
    def __init__(self, loader_func):
        self._loader = loader_func
        self._data = None

    def __getattr__(self, name):
        if self._data is None:
            self._data = self._loader()
        return getattr(self._data, name)

# Динамические атрибуты
class DynamicConfig:
    def __init__(self, config_dict):
        self._config = config_dict

    def __getattr__(self, name):
        if name in self._config:
            return self._config[name]
        raise AttributeError(f"No config '{name}'")

config = DynamicConfig({'debug': True, 'port': 8080})
print(config.debug)  # True
```

### На интервью
**Как отвечать:** `__getattribute__` — всегда, `__getattr__` — только при отсутствии. Покажите proxy паттерн.

---

## Вопрос 6: Как использовать inspect и ast модули?

### Зачем спрашивают
Продвинутая интроспекция и анализ кода. Senior-level тема.

### Короткий ответ
`inspect` — интроспекция объектов (получение сигнатуры, исходного кода, стека). `ast` — парсинг и модификация Python кода как дерева. Используются для: code generation, linting tools, documentation generators.

### Пример

```python
import inspect
import ast

# === inspect ===

def my_function(a: int, b: str = "default") -> str:
    """My docstring"""
    return f"{a}: {b}"

# Сигнатура
sig = inspect.signature(my_function)
print(sig)  # (a: int, b: str = 'default') -> str

for name, param in sig.parameters.items():
    print(f"{name}: {param.annotation}, default={param.default}")

# Исходный код
print(inspect.getsource(my_function))

# Информация о функции
print(inspect.isfunction(my_function))  # True
print(inspect.getdoc(my_function))      # My docstring

# Stack inspection
def outer():
    inner()

def inner():
    frame = inspect.currentframe()
    print(inspect.getouterframes(frame))

# === ast ===

code = """
def hello(name):
    print(f"Hello, {name}")
"""

# Парсинг
tree = ast.parse(code)
print(ast.dump(tree, indent=2))

# Обход дерева
class FunctionVisitor(ast.NodeVisitor):
    def visit_FunctionDef(self, node):
        print(f"Found function: {node.name}")
        self.generic_visit(node)

visitor = FunctionVisitor()
visitor.visit(tree)

# Модификация кода
class AddDocstring(ast.NodeTransformer):
    def visit_FunctionDef(self, node):
        if not ast.get_docstring(node):
            docstring = ast.Expr(value=ast.Constant(value="Auto-generated docstring"))
            node.body.insert(0, docstring)
        return node

modified_tree = AddDocstring().visit(tree)
print(ast.unparse(modified_tree))  # Python 3.9+

# Практический пример: извлечение всех строковых литералов
class StringCollector(ast.NodeVisitor):
    def __init__(self):
        self.strings = []

    def visit_Constant(self, node):
        if isinstance(node.value, str):
            self.strings.append(node.value)

collector = StringCollector()
collector.visit(tree)
print(collector.strings)
```

### На интервью
**Как отвечать:** inspect для runtime интроспекции, ast для статического анализа кода. Покажите практические примеры.

---

[← Назад к списку тем](README.md)
