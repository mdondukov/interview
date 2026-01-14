# 04. Asyncio и конкурентность

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работает event loop в asyncio?

### Зачем спрашивают
Event loop — сердце asyncio. Понимание его работы критично для написания эффективного async кода.

### Короткий ответ
Event loop — бесконечный цикл, который выполняет корутины, обрабатывает I/O события и планирует callbacks. Работает в одном потоке, переключаясь между задачами на точках await. Использует селекторы ОС (epoll/kqueue) для эффективного ожидания I/O.

### Детальный разбор

**Упрощённая модель:**
```python
while running:
    # 1. Получить готовые I/O события
    events = selector.select(timeout)

    # 2. Обработать события
    for event in events:
        handle_event(event)

    # 3. Выполнить готовые callbacks
    for callback in ready_callbacks:
        callback()

    # 4. Возобновить готовые корутины
    for task in ready_tasks:
        task.step()
```

**Ключевые концепции:**
- **Корутина** — async функция, может быть приостановлена
- **Task** — обёртка для планирования корутины
- **Future** — результат асинхронной операции
- **await** — точка переключения контекста

### Пример

```python
import asyncio

# Получение event loop
async def main():
    loop = asyncio.get_running_loop()
    print(f"Loop running: {loop.is_running()}")

asyncio.run(main())

# Ручное управление loop (устаревший способ)
# loop = asyncio.get_event_loop()
# loop.run_until_complete(main())
# loop.close()

# Scheduling callbacks
async def demo_callbacks():
    loop = asyncio.get_running_loop()

    def callback():
        print("Callback executed!")

    # Немедленное выполнение (после текущей корутины)
    loop.call_soon(callback)

    # Отложенное выполнение
    loop.call_later(1.0, callback)

    # Выполнение в конкретное время
    loop.call_at(loop.time() + 2.0, callback)

    await asyncio.sleep(3)

# Как работает переключение
async def task_a():
    print("A: start")
    await asyncio.sleep(1)  # <-- Переключение!
    print("A: end")

async def task_b():
    print("B: start")
    await asyncio.sleep(0.5)  # <-- Переключение!
    print("B: end")

async def main():
    await asyncio.gather(task_a(), task_b())
    # Вывод:
    # A: start
    # B: start
    # B: end (через 0.5s)
    # A: end (через 1s)

asyncio.run(main())

# Debug mode
asyncio.run(main(), debug=True)

# Или через переменную окружения
# PYTHONASYNCIODEBUG=1 python script.py
```

### Типичные ошибки
- Блокирующий код в async функции (без await)
- Создание нескольких event loops
- Забывать про `asyncio.run()` для запуска

### На интервью
**Как отвечать:** Опишите цикл обработки событий, объясните роль await как точки переключения, упомяните что это однопоточная модель.

**Follow-up вопросы:**
- Что происходит при блокирующем вызове в async функции?
- Как запустить блокирующий код в async контексте?
- Чем uvloop отличается от стандартного loop?

---

## Вопрос 2: В чём разница между coroutine, task и future?

### Зачем спрашивают
Фундаментальные абстракции asyncio. Путаница между ними — частая причина багов.

### Короткий ответ
**Coroutine** — объект, возвращаемый async функцией, требует await для выполнения. **Task** — обёртка для планирования корутины в event loop, выполняется параллельно. **Future** — низкоуровневый объект для результата асинхронной операции. Task наследует Future.

### Детальный разбор

**Иерархия:**
```
Future (низкоуровневый результат)
   ↑
Task (запланированная корутина)
   ↑
Coroutine (async def function)
```

**Когда что использовать:**
- **Coroutine** — базовый блок, создаётся через `async def`
- **Task** — для параллельного выполнения, `asyncio.create_task()`
- **Future** — редко напрямую, для интеграции с callback API

### Пример

```python
import asyncio

# Coroutine - просто объект, не выполняется
async def my_coroutine():
    await asyncio.sleep(1)
    return "result"

# Это корутина, не результат!
coro = my_coroutine()
print(type(coro))  # <class 'coroutine'>

# Корутина выполняется только с await
async def main():
    result = await my_coroutine()
    print(result)

# Task - запланированная корутина
async def main():
    # Создание task запускает выполнение
    task = asyncio.create_task(my_coroutine())
    print(type(task))  # <class '_asyncio.Task'>

    # Task выполняется параллельно
    await asyncio.sleep(0)  # Даём task начать

    # Ждём результат
    result = await task
    print(result)

# Разница: await coroutine vs create_task
async def demo():
    # Последовательно (2 секунды)
    await asyncio.sleep(1)
    await asyncio.sleep(1)

    # Параллельно (1 секунда)
    t1 = asyncio.create_task(asyncio.sleep(1))
    t2 = asyncio.create_task(asyncio.sleep(1))
    await t1
    await t2

# Future - низкоуровневый API
async def future_example():
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    # В другом месте кода
    async def set_result():
        await asyncio.sleep(1)
        future.set_result("done!")

    asyncio.create_task(set_result())
    result = await future
    print(result)

# Интеграция с callback API
async def callback_to_async(callback_api):
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    def callback(result):
        loop.call_soon_threadsafe(future.set_result, result)

    callback_api(callback)
    return await future

# Task cancellation
async def cancellation_demo():
    async def long_task():
        try:
            await asyncio.sleep(10)
        except asyncio.CancelledError:
            print("Task was cancelled!")
            raise  # Важно пробросить!

    task = asyncio.create_task(long_task())
    await asyncio.sleep(0.1)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Caught cancellation")

# Task naming (Python 3.8+)
task = asyncio.create_task(my_coroutine(), name="my_task")
print(task.get_name())  # my_task

asyncio.run(main())
```

### Типичные ошибки
- Не await'ить корутину (предупреждение "coroutine was never awaited")
- Путать `await coro()` и `create_task(coro())`
- Не обрабатывать CancelledError

### На интервью
**Как отвечать:** Объясните что корутина — это объект, который нужно await'ить, Task — для параллельного выполнения, Future — для callback интеграции.

**Follow-up вопросы:**
- Когда использовать create_task vs await?
- Как отменить task?
- Что делает ensure_future?

---

## Вопрос 3: Как использовать async/await правильно?

### Зачем спрашивают
Проверяют практический опыт работы с asyncio и понимание best practices.

### Короткий ответ
`async def` создаёт корутину, `await` приостанавливает до завершения awaitable объекта. Используйте `asyncio.gather()` для параллельного выполнения, `asyncio.create_task()` для background задач. Никогда не блокируйте event loop синхронным кодом.

### Детальный разбор

**Правила:**
1. `async def` — всегда корутина
2. `await` — только внутри async функции
3. Блокирующий код → `run_in_executor()`
4. Параллельное выполнение → `gather()` или `TaskGroup`

**Awaitable объекты:**
- Coroutines
- Tasks
- Futures
- Объекты с `__await__()`

### Пример

```python
import asyncio
import aiohttp

# Базовый паттерн
async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    async with aiohttp.ClientSession() as session:
        html = await fetch_url(session, 'http://example.com')
        print(len(html))

# Параллельное выполнение
async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        return results

# С обработкой ошибок
async def fetch_safe(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        for url, result in zip(urls, results):
            if isinstance(result, Exception):
                print(f"Error fetching {url}: {result}")
            else:
                print(f"Success: {url}")
        return results

# TaskGroup (Python 3.11+) - более безопасный подход
async def fetch_with_taskgroup(urls):
    results = []
    async with asyncio.TaskGroup() as tg:
        async with aiohttp.ClientSession() as session:
            for url in urls:
                tg.create_task(fetch_url(session, url))
    # Если любая task падает - все отменяются

# Async context managers
class AsyncResource:
    async def __aenter__(self):
        print("Acquiring resource")
        await asyncio.sleep(0.1)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Releasing resource")
        await asyncio.sleep(0.1)

async def use_resource():
    async with AsyncResource() as r:
        print("Using resource")

# Async iterators
class AsyncCounter:
    def __init__(self, stop):
        self.stop = stop
        self.count = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.count >= self.stop:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        self.count += 1
        return self.count

async def iterate():
    async for i in AsyncCounter(5):
        print(i)

# Async generator
async def async_gen(n):
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def consume():
    async for value in async_gen(5):
        print(value)

# Блокирующий код в async
import time

async def with_blocking():
    loop = asyncio.get_running_loop()
    # Запуск в thread pool
    result = await loop.run_in_executor(None, time.sleep, 1)

# Timeout
async def with_timeout():
    try:
        async with asyncio.timeout(1.0):  # Python 3.11+
            await asyncio.sleep(10)
    except asyncio.TimeoutError:
        print("Timed out!")

    # Или старый способ
    try:
        await asyncio.wait_for(asyncio.sleep(10), timeout=1.0)
    except asyncio.TimeoutError:
        print("Timed out!")

asyncio.run(main())
```

### Типичные ошибки
- Блокирующий код без run_in_executor
- Забывать await (RuntimeWarning)
- Не использовать async with для ресурсов

### На интервью
**Как отвечать:** Покажите правильную структуру async кода, объясните gather vs TaskGroup, упомяните run_in_executor для блокирующего кода.

**Follow-up вопросы:**
- Как обработать timeout?
- Что такое async context manager?
- Как тестировать async код?

---

## Вопрос 4: Что такое asyncio.gather vs asyncio.wait vs TaskGroup?

### Зачем спрашивают
Разные инструменты для параллельного выполнения. Важно понимать когда какой использовать.

### Короткий ответ
`gather()` — собирает результаты в порядке передачи, простой API. `wait()` — возвращает sets (done, pending), больше контроля. `TaskGroup` (Python 3.11+) — структурированная конкурентность, автоматическая отмена при ошибке.

### Детальный разбор

**Сравнение:**

| Аспект | gather | wait | TaskGroup |
|--------|--------|------|-----------|
| Результат | List | Sets (done, pending) | Implicit |
| Порядок | Сохраняется | Нет | N/A |
| Отмена при ошибке | Нет | Опционально | Да |
| Синтаксис | `await gather(...)` | `await wait(...)` | `async with` |
| Python | 3.4+ | 3.4+ | 3.11+ |

### Пример

```python
import asyncio

async def task(name, delay):
    await asyncio.sleep(delay)
    if name == "fail":
        raise ValueError("Task failed!")
    return f"{name} done"

# gather - простой параллельный запуск
async def demo_gather():
    # Результаты в порядке передачи
    results = await asyncio.gather(
        task("A", 0.3),
        task("B", 0.1),
        task("C", 0.2),
    )
    print(results)  # ['A done', 'B done', 'C done']

    # С обработкой исключений
    results = await asyncio.gather(
        task("A", 0.1),
        task("fail", 0.2),
        task("C", 0.3),
        return_exceptions=True  # Не падать, вернуть исключение
    )
    print(results)  # ['A done', ValueError('Task failed!'), 'C done']

# wait - больше контроля
async def demo_wait():
    tasks = [
        asyncio.create_task(task("A", 0.3), name="A"),
        asyncio.create_task(task("B", 0.1), name="B"),
        asyncio.create_task(task("C", 0.2), name="C"),
    ]

    # Ждать все
    done, pending = await asyncio.wait(tasks)
    for t in done:
        print(f"{t.get_name()}: {t.result()}")

    # Ждать первый завершённый
    tasks = [asyncio.create_task(task(n, d)) for n, d in [("A", 0.3), ("B", 0.1)]]
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    print(f"First done: {done.pop().result()}")
    # Отменяем остальные
    for t in pending:
        t.cancel()

    # С timeout
    tasks = [asyncio.create_task(task("slow", 10))]
    done, pending = await asyncio.wait(tasks, timeout=1.0)
    print(f"Done: {len(done)}, Pending: {len(pending)}")

# TaskGroup (Python 3.11+) - структурированная конкурентность
async def demo_taskgroup():
    try:
        async with asyncio.TaskGroup() as tg:
            t1 = tg.create_task(task("A", 0.1))
            t2 = tg.create_task(task("fail", 0.2))  # Падает!
            t3 = tg.create_task(task("C", 0.3))
        # Сюда не дойдём - fail вызовет отмену всех
    except* ValueError as eg:
        print(f"Caught: {eg.exceptions}")
        # t1 и t3 отменены автоматически!

    # Успешный сценарий
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(task("A", 0.1))
        t2 = tg.create_task(task("B", 0.2))

    print(t1.result(), t2.result())  # Доступны после выхода из with

# as_completed - результаты по мере готовности
async def demo_as_completed():
    tasks = [
        asyncio.create_task(task("A", 0.3)),
        asyncio.create_task(task("B", 0.1)),
        asyncio.create_task(task("C", 0.2)),
    ]

    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(result)  # B done, C done, A done (по мере готовности)

# Практический пример: запросы с ограничением
async def fetch_with_limit(urls, max_concurrent=10):
    semaphore = asyncio.Semaphore(max_concurrent)

    async def fetch_one(url):
        async with semaphore:
            async with aiohttp.ClientSession() as session:
                async with session.get(url) as response:
                    return await response.text()

    return await asyncio.gather(*[fetch_one(url) for url in urls])

asyncio.run(demo_gather())
```

### Типичные ошибки
- Не обрабатывать pending tasks в wait()
- Забывать return_exceptions в gather()
- Не использовать TaskGroup когда нужна отмена при ошибке

### На интервью
**Как отвечать:** Сравните API и use cases: gather для простых случаев, wait для контроля, TaskGroup для надёжности. Упомяните as_completed для обработки по мере готовности.

**Follow-up вопросы:**
- Что такое structured concurrency?
- Как обработать частичные результаты в gather?
- Чем TaskGroup лучше gather?

---

## Вопрос 5: Как обрабатывать ошибки в async коде?

### Зачем спрашивают
Error handling в async — сложнее чем в sync коде. Проверяют production-ready навыки.

### Короткий ответ
Используйте try/except как обычно, но помните: CancelledError нужно пробрасывать, в gather — `return_exceptions=True`, в TaskGroup — ExceptionGroup (Python 3.11+). Всегда cleanup через finally или async context managers.

### Детальный разбор

**Типы ошибок:**
- `asyncio.CancelledError` — задача отменена
- `asyncio.TimeoutError` — превышен timeout
- `ExceptionGroup` — множественные ошибки (Python 3.11+)
- Обычные исключения

**Best practices:**
1. Не глотать CancelledError
2. Использовать `return_exceptions=True` или TaskGroup
3. Cleanup в finally или context managers
4. Логировать unhandled exceptions

### Пример

```python
import asyncio
import logging

logging.basicConfig(level=logging.INFO)

# Базовая обработка
async def may_fail():
    await asyncio.sleep(0.1)
    raise ValueError("Something went wrong")

async def basic_handling():
    try:
        await may_fail()
    except ValueError as e:
        print(f"Caught: {e}")
    finally:
        print("Cleanup")

# CancelledError - ВАЖНО пробрасывать!
async def cancellable_task():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("Task cancelled, cleaning up...")
        # Cleanup код
        raise  # ВАЖНО! Иначе отмена не сработает

async def cancel_demo():
    task = asyncio.create_task(cancellable_task())
    await asyncio.sleep(0.1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Task was cancelled")

# Timeout handling
async def with_timeout():
    try:
        async with asyncio.timeout(1.0):  # Python 3.11+
            await asyncio.sleep(10)
    except TimeoutError:
        print("Operation timed out")

    # Или wait_for
    try:
        await asyncio.wait_for(asyncio.sleep(10), timeout=1.0)
    except asyncio.TimeoutError:
        print("Operation timed out")

# gather с return_exceptions
async def gather_errors():
    async def task1():
        return "success"

    async def task2():
        raise ValueError("error 2")

    async def task3():
        raise TypeError("error 3")

    results = await asyncio.gather(
        task1(), task2(), task3(),
        return_exceptions=True
    )

    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Task {i} failed: {result}")
        else:
            print(f"Task {i} succeeded: {result}")

# TaskGroup с ExceptionGroup (Python 3.11+)
async def taskgroup_errors():
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(asyncio.sleep(0.1))
            tg.create_task(may_fail())
            tg.create_task(asyncio.sleep(0.2))
    except* ValueError as eg:
        for exc in eg.exceptions:
            print(f"ValueError: {exc}")
    except* TypeError as eg:
        for exc in eg.exceptions:
            print(f"TypeError: {exc}")

# Exception handler для loop
def exception_handler(loop, context):
    exception = context.get('exception')
    message = context.get('message')
    logging.error(f"Unhandled exception: {message}, {exception}")

async def setup_handler():
    loop = asyncio.get_running_loop()
    loop.set_exception_handler(exception_handler)

# Retry pattern
async def retry_async(coro_func, retries=3, delay=1.0):
    last_exception = None
    for attempt in range(retries):
        try:
            return await coro_func()
        except Exception as e:
            last_exception = e
            logging.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt < retries - 1:
                await asyncio.sleep(delay * (attempt + 1))
    raise last_exception

async def unreliable():
    import random
    if random.random() < 0.7:
        raise ConnectionError("Network error")
    return "Success!"

async def main():
    result = await retry_async(unreliable, retries=5)
    print(result)

# Shield от отмены
async def important_cleanup():
    await asyncio.sleep(1)
    print("Cleanup done")

async def shielded():
    try:
        await asyncio.sleep(10)
    finally:
        # Защищаем cleanup от отмены
        await asyncio.shield(important_cleanup())

asyncio.run(main())
```

### Типичные ошибки
- Глотать CancelledError (не пробрасывать)
- Не использовать return_exceptions в gather
- Забывать про cleanup в finally

### На интервью
**Как отвечать:** Объясните особенности CancelledError, покажите gather с return_exceptions, упомяните TaskGroup и ExceptionGroup для Python 3.11+.

**Follow-up вопросы:**
- Почему нужно пробрасывать CancelledError?
- Как реализовать retry в async?
- Что делает asyncio.shield?

---

## Вопрос 6: Когда использовать threading vs multiprocessing vs asyncio?

### Зачем спрашивают
Ключевой архитектурный вопрос. Показывает понимание конкурентности в Python.

### Короткий ответ
**asyncio** — для I/O-bound задач с множеством соединений. **threading** — для I/O-bound с блокирующими библиотеками. **multiprocessing** — для CPU-bound задач, обходит GIL. Выбор зависит от характера задачи и используемых библиотек.

### Детальный разбор

**Сравнение:**

| Аспект | asyncio | threading | multiprocessing |
|--------|---------|-----------|-----------------|
| GIL | Не проблема | Проблема для CPU | Обходит GIL |
| Память | Низкая | Средняя | Высокая |
| I/O-bound | Отлично | Хорошо | Overhead |
| CPU-bound | Плохо | Плохо (GIL) | Отлично |
| Масштабируемость | 10K+ соединений | 100s потоков | 10s процессов |
| Сложность | Средняя | Высокая (race conditions) | Средняя |

**Правило выбора:**
```
I/O-bound + async библиотеки → asyncio
I/O-bound + sync библиотеки → threading
CPU-bound → multiprocessing
```

### Пример

```python
import asyncio
import threading
import multiprocessing
import time
import aiohttp
import requests

# === I/O-bound: сетевые запросы ===

# Синхронно (плохо)
def sync_fetch(urls):
    results = []
    for url in urls:
        response = requests.get(url)
        results.append(len(response.text))
    return results
# 10 URLs × 0.5s = 5s

# Threading (хорошо для sync библиотек)
def threaded_fetch(urls):
    results = [None] * len(urls)

    def fetch(i, url):
        response = requests.get(url)
        results[i] = len(response.text)

    threads = [threading.Thread(target=fetch, args=(i, url))
               for i, url in enumerate(urls)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return results
# 10 URLs параллельно ≈ 0.5s

# Asyncio (лучше для async библиотек)
async def async_fetch(urls):
    async with aiohttp.ClientSession() as session:
        async def fetch_one(url):
            async with session.get(url) as response:
                return len(await response.text())
        return await asyncio.gather(*[fetch_one(url) for url in urls])
# 10 URLs параллельно ≈ 0.5s, меньше памяти

# === CPU-bound: вычисления ===

def cpu_task(n):
    """Тяжёлая CPU операция"""
    return sum(i * i for i in range(n))

# Синхронно
def sync_compute(tasks):
    return [cpu_task(n) for n in tasks]
# 4 tasks × 1s = 4s

# Threading (НЕ помогает из-за GIL!)
def threaded_compute(tasks):
    results = [None] * len(tasks)

    def compute(i, n):
        results[i] = cpu_task(n)

    threads = [threading.Thread(target=compute, args=(i, n))
               for i, n in enumerate(tasks)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return results
# Всё ещё ~4s из-за GIL!

# Multiprocessing (решает проблему)
def parallel_compute(tasks):
    with multiprocessing.Pool() as pool:
        return pool.map(cpu_task, tasks)
# 4 tasks на 4 ядрах ≈ 1s

# === Гибридный подход ===

# Asyncio + run_in_executor для блокирующего кода
async def hybrid_approach():
    loop = asyncio.get_running_loop()

    # CPU-bound в процессе
    with multiprocessing.Pool() as pool:
        cpu_result = await loop.run_in_executor(
            None,  # Default executor
            lambda: pool.map(cpu_task, [10**7] * 4)
        )

    # I/O-bound асинхронно
    async with aiohttp.ClientSession() as session:
        io_result = await asyncio.gather(*[
            session.get('http://example.com')
            for _ in range(10)
        ])

    return cpu_result, io_result

# ThreadPoolExecutor для блокирующего I/O
from concurrent.futures import ThreadPoolExecutor

async def with_blocking_library():
    loop = asyncio.get_running_loop()

    with ThreadPoolExecutor(max_workers=10) as executor:
        # requests блокирующий, запускаем в thread pool
        results = await asyncio.gather(*[
            loop.run_in_executor(executor, requests.get, url)
            for url in urls
        ])

    return results

# === Сравнение производительности ===

def benchmark():
    urls = ['http://example.com'] * 10
    n_tasks = [10**6] * 4

    # I/O-bound
    start = time.time()
    asyncio.run(async_fetch(urls))
    print(f"Asyncio I/O: {time.time() - start:.2f}s")

    start = time.time()
    threaded_fetch(urls)
    print(f"Threading I/O: {time.time() - start:.2f}s")

    # CPU-bound
    start = time.time()
    sync_compute(n_tasks)
    print(f"Sync CPU: {time.time() - start:.2f}s")

    start = time.time()
    parallel_compute(n_tasks)
    print(f"Multiprocess CPU: {time.time() - start:.2f}s")
```

### Типичные ошибки
- Использовать threading для CPU-bound (GIL!)
- Использовать multiprocessing для I/O-bound (overhead)
- Блокирующий код в asyncio без run_in_executor

### На интервью
**Как отвечать:** Начните с характера задачи (I/O vs CPU), объясните влияние GIL, покажите гибридный подход.

**Follow-up вопросы:**
- Как определить является ли задача I/O или CPU-bound?
- Что такое GIL и как его обойти?
- Когда использовать ProcessPoolExecutor?

---

## Вопрос 7: Как работает concurrent.futures?

### Зачем спрашивают
Высокоуровневый API для параллелизма. Унифицирует threading и multiprocessing.

### Короткий ответ
`concurrent.futures` предоставляет `ThreadPoolExecutor` и `ProcessPoolExecutor` с единым API. Возвращает `Future` объекты для результатов. Поддерживает `map()`, `submit()`, context managers. Интегрируется с asyncio через `run_in_executor`.

### Детальный разбор

**Основные компоненты:**
- `Executor` — базовый класс для пулов
- `ThreadPoolExecutor` — пул потоков
- `ProcessPoolExecutor` — пул процессов
- `Future` — результат асинхронной операции

**Методы:**
- `submit(fn, *args)` — запуск одной задачи
- `map(fn, *iterables)` — запуск для каждого элемента
- `shutdown(wait=True)` — завершение executor

### Пример

```python
from concurrent.futures import (
    ThreadPoolExecutor,
    ProcessPoolExecutor,
    as_completed,
    wait,
    FIRST_COMPLETED
)
import time

def task(n):
    time.sleep(n)
    return n * 2

# ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=4) as executor:
    # submit - одна задача
    future = executor.submit(task, 1)
    print(future.result())  # 2

    # map - несколько задач
    results = list(executor.map(task, [0.1, 0.2, 0.3]))
    print(results)  # [0.2, 0.4, 0.6]

# ProcessPoolExecutor (тот же API)
def cpu_task(n):
    return sum(i * i for i in range(n))

with ProcessPoolExecutor() as executor:
    results = list(executor.map(cpu_task, [10**6] * 4))

# as_completed - результаты по мере готовности
with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(task, n): n for n in [0.3, 0.1, 0.2]}

    for future in as_completed(futures):
        n = futures[future]
        try:
            result = future.result()
            print(f"Task {n} returned {result}")
        except Exception as e:
            print(f"Task {n} raised {e}")

# wait - ждать с условием
with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(task, n) for n in [0.3, 0.1, 0.2]]

    # Ждать первый
    done, pending = wait(futures, return_when=FIRST_COMPLETED)
    print(f"First done: {done.pop().result()}")

    # С timeout
    done, pending = wait(futures, timeout=0.15)
    print(f"Done in 0.15s: {len(done)}")

# Future API
with ThreadPoolExecutor(max_workers=1) as executor:
    future = executor.submit(task, 1)

    print(future.done())       # False
    print(future.running())    # True/False
    print(future.cancelled())  # False

    # Отмена (если ещё не начался)
    future.cancel()

    # Callback
    future.add_done_callback(lambda f: print(f"Done: {f.result()}"))

    # Блокирующее получение результата
    result = future.result(timeout=5)

    # Исключение
    try:
        result = future.result()
    except Exception as e:
        print(f"Task failed: {e}")

# Интеграция с asyncio
import asyncio

async def run_blocking():
    loop = asyncio.get_running_loop()

    # В thread pool (I/O-bound блокирующий код)
    with ThreadPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, task, 1)

    # В process pool (CPU-bound)
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_task, 10**6)

    return result

# Практический пример: параллельная загрузка файлов
import requests

def download(url):
    response = requests.get(url)
    return len(response.content)

def download_all(urls, max_workers=10):
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {executor.submit(download, url): url for url in urls}
        results = {}
        for future in as_completed(futures):
            url = futures[future]
            try:
                results[url] = future.result()
            except Exception as e:
                results[url] = f"Error: {e}"
        return results

# Обработка исключений в map
def may_fail(x):
    if x == 2:
        raise ValueError("Can't process 2")
    return x * 2

with ThreadPoolExecutor() as executor:
    # map падает на первом исключении
    try:
        results = list(executor.map(may_fail, [1, 2, 3]))
    except ValueError as e:
        print(f"Caught: {e}")

    # submit позволяет обработать каждое отдельно
    futures = [executor.submit(may_fail, x) for x in [1, 2, 3]]
    for future in as_completed(futures):
        try:
            print(future.result())
        except ValueError as e:
            print(f"Failed: {e}")
```

### Типичные ошибки
- Забывать про context manager (утечка ресурсов)
- Не обрабатывать исключения из futures
- Использовать ProcessPoolExecutor без `if __name__ == '__main__'`

### На интервью
**Как отвечать:** Покажите унифицированный API, объясните разницу между Thread и Process executor, упомяните as_completed и интеграцию с asyncio.

**Follow-up вопросы:**
- Чем submit отличается от map?
- Как обработать исключения?
- Когда Future отменяется?

---

## Вопрос 8: Как сочетать sync и async код?

### Зачем спрашивают
Реальные проекты часто имеют и sync, и async код. Важный практический навык.

### Короткий ответ
Sync → async: используйте `asyncio.run()` или `loop.run_until_complete()`. Async → sync (блокирующий код): используйте `run_in_executor()`. Для библиотек: создавайте sync wrappers или используйте `asyncio.to_thread()` (Python 3.9+).

### Детальный разбор

**Направления:**
1. **Sync вызывает async** — запуск event loop
2. **Async вызывает sync** — executor для блокирующего кода
3. **Смешанная кодовая база** — sync wrappers для async API

### Пример

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import time

# === Sync вызывает Async ===

async def async_operation():
    await asyncio.sleep(1)
    return "async result"

# Из sync кода
def sync_function():
    # Способ 1: asyncio.run() (создаёт новый loop)
    result = asyncio.run(async_operation())
    return result

# Способ 2: существующий loop (deprecated в Python 3.10+)
# loop = asyncio.get_event_loop()
# result = loop.run_until_complete(async_operation())

# Способ 3: для вложенных async (nest_asyncio)
# import nest_asyncio
# nest_asyncio.apply()
# asyncio.run(async_operation())

# === Async вызывает Sync ===

def blocking_io():
    """Блокирующая I/O операция"""
    time.sleep(1)
    return "io result"

def cpu_bound(n):
    """CPU-bound операция"""
    return sum(i * i for i in range(n))

async def run_blocking():
    loop = asyncio.get_running_loop()

    # Способ 1: run_in_executor с thread pool (I/O)
    result = await loop.run_in_executor(None, blocking_io)

    # Способ 2: asyncio.to_thread (Python 3.9+, проще)
    result = await asyncio.to_thread(blocking_io)

    # С process pool (CPU-bound)
    from concurrent.futures import ProcessPoolExecutor
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_bound, 10**6)

    return result

# === Sync wrappers для async библиотек ===

class AsyncClient:
    """Async HTTP клиент"""
    async def fetch(self, url):
        import aiohttp
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                return await response.text()

class SyncClient:
    """Sync wrapper для AsyncClient"""
    def __init__(self):
        self._async_client = AsyncClient()
        self._loop = None

    def _get_loop(self):
        if self._loop is None or self._loop.is_closed():
            self._loop = asyncio.new_event_loop()
        return self._loop

    def fetch(self, url):
        loop = self._get_loop()
        return loop.run_until_complete(
            self._async_client.fetch(url)
        )

    def close(self):
        if self._loop and not self._loop.is_closed():
            self._loop.close()

# Использование
sync_client = SyncClient()
result = sync_client.fetch('http://example.com')

# === Универсальный декоратор ===

def async_to_sync(func):
    """Конвертирует async функцию в sync"""
    import functools

    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return asyncio.run(func(*args, **kwargs))

    return wrapper

@async_to_sync
async def my_async_func():
    await asyncio.sleep(1)
    return "done"

# Теперь можно вызывать синхронно
result = my_async_func()

# === Параллельные sync вызовы из async ===

async def parallel_blocking():
    """Запуск множества блокирующих операций параллельно"""
    loop = asyncio.get_running_loop()

    with ThreadPoolExecutor(max_workers=10) as executor:
        tasks = [
            loop.run_in_executor(executor, blocking_io)
            for _ in range(10)
        ]
        results = await asyncio.gather(*tasks)

    return results

# === Контекст между sync и async ===

import contextvars

request_id = contextvars.ContextVar('request_id')

async def async_handler(rid):
    request_id.set(rid)
    await asyncio.to_thread(sync_logging)

def sync_logging():
    # ContextVar работает между sync/async
    print(f"Request ID: {request_id.get()}")

# === Тестирование async кода из sync тестов ===

import pytest

@pytest.fixture
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

def test_async_code(event_loop):
    async def my_async_test():
        result = await async_operation()
        assert result == "async result"

    event_loop.run_until_complete(my_async_test())

# Или с pytest-asyncio
# @pytest.mark.asyncio
# async def test_async():
#     result = await async_operation()
#     assert result == "async result"
```

### Типичные ошибки
- Вызывать asyncio.run() внутри async функции
- Блокирующий код без run_in_executor
- Не закрывать event loop

### На интервью
**Как отвечать:** Покажите оба направления (sync→async и async→sync), объясните run_in_executor и to_thread, упомяните sync wrappers.

**Follow-up вопросы:**
- Почему нельзя вложенный asyncio.run()?
- Как работает contextvars с async?
- Что такое nest_asyncio?

---

## Вопрос 9: Что такое semaphore и как ограничить конкурентность?

### Зачем спрашивают
Rate limiting и ограничение ресурсов — практическая задача. Показывает production опыт.

### Короткий ответ
`asyncio.Semaphore` ограничивает количество одновременно выполняющихся корутин. Используется для rate limiting, ограничения соединений к БД/API, контроля ресурсов. `BoundedSemaphore` защищает от лишних release().

### Детальный разбор

**Примитивы синхронизации в asyncio:**
- `Lock` — эксклюзивный доступ
- `Semaphore` — ограничение числа одновременных доступов
- `BoundedSemaphore` — Semaphore с проверкой
- `Event` — сигнализация между задачами
- `Condition` — сложная синхронизация

### Пример

```python
import asyncio
import aiohttp
import time

# Базовый Semaphore
async def limited_task(sem, n):
    async with sem:
        print(f"Task {n} started")
        await asyncio.sleep(1)
        print(f"Task {n} done")

async def demo_semaphore():
    sem = asyncio.Semaphore(3)  # Максимум 3 одновременно
    tasks = [limited_task(sem, i) for i in range(10)]
    await asyncio.gather(*tasks)
    # Выполняются по 3 одновременно

# Rate limiting для API
class RateLimiter:
    def __init__(self, rate_per_second):
        self.rate = rate_per_second
        self.semaphore = asyncio.Semaphore(rate_per_second)
        self._reset_task = None

    async def acquire(self):
        await self.semaphore.acquire()
        # Освободить через 1 секунду
        asyncio.create_task(self._release_after(1.0))

    async def _release_after(self, delay):
        await asyncio.sleep(delay)
        self.semaphore.release()

async def rate_limited_fetch(limiter, session, url):
    await limiter.acquire()
    async with session.get(url) as response:
        return await response.text()

async def demo_rate_limiting():
    limiter = RateLimiter(rate_per_second=5)
    async with aiohttp.ClientSession() as session:
        tasks = [
            rate_limited_fetch(limiter, session, 'http://example.com')
            for _ in range(20)
        ]
        await asyncio.gather(*tasks)

# Ограничение соединений к БД
class ConnectionPool:
    def __init__(self, max_connections=10):
        self.semaphore = asyncio.Semaphore(max_connections)
        self.connections = []

    async def get_connection(self):
        async with self.semaphore:
            conn = await self._create_connection()
            try:
                yield conn
            finally:
                await self._release_connection(conn)

    async def _create_connection(self):
        # Создание соединения
        return object()

    async def _release_connection(self, conn):
        # Возврат в пул
        pass

# BoundedSemaphore - защита от лишних release
async def demo_bounded():
    sem = asyncio.BoundedSemaphore(3)

    async with sem:
        pass

    # sem.release()  # ValueError: BoundedSemaphore released too many times

# Lock - эксклюзивный доступ
async def demo_lock():
    lock = asyncio.Lock()
    shared_data = []

    async def worker(n):
        async with lock:
            # Критическая секция
            shared_data.append(n)
            await asyncio.sleep(0.1)

    await asyncio.gather(*[worker(i) for i in range(5)])

# Event - сигнализация
async def demo_event():
    event = asyncio.Event()

    async def waiter():
        print("Waiting for event...")
        await event.wait()
        print("Event received!")

    async def setter():
        await asyncio.sleep(1)
        print("Setting event")
        event.set()

    await asyncio.gather(waiter(), setter())

# Практический пример: параллельная загрузка с ограничением
async def download_with_limit(urls, max_concurrent=10):
    semaphore = asyncio.Semaphore(max_concurrent)
    results = {}

    async def fetch_one(url):
        async with semaphore:
            async with aiohttp.ClientSession() as session:
                try:
                    async with session.get(url, timeout=30) as response:
                        content = await response.text()
                        results[url] = len(content)
                except Exception as e:
                    results[url] = f"Error: {e}"

    await asyncio.gather(*[fetch_one(url) for url in urls])
    return results

# Barrier (Python 3.11+)
async def demo_barrier():
    barrier = asyncio.Barrier(3)

    async def worker(n):
        print(f"Worker {n} waiting")
        await barrier.wait()  # Ждёт пока 3 worker'а не дойдут сюда
        print(f"Worker {n} proceeding")

    await asyncio.gather(*[worker(i) for i in range(3)])

asyncio.run(demo_semaphore())
```

### Типичные ошибки
- Не использовать async with (забывать release)
- Слишком большой лимит Semaphore
- Deadlock при неправильном порядке acquire

### На интервью
**Как отвечать:** Покажите rate limiting с Semaphore, объясните разницу с Lock, упомяните BoundedSemaphore для безопасности.

**Follow-up вопросы:**
- Чем Semaphore отличается от Lock?
- Как реализовать rate limiter?
- Что такое Barrier?

---

## Вопрос 10: Как тестировать асинхронный код?

### Зачем спрашивают
Тестирование async — отдельный навык. Показывает production-ready подход.

### Короткий ответ
Используйте `pytest-asyncio` для async тестов, `asyncio.run()` для unittest. Мокайте async функции через `AsyncMock` (Python 3.8+) или `unittest.mock`. Для fixtures используйте `@pytest.fixture` с async. Изолируйте event loop для каждого теста.

### Детальный разбор

**Инструменты:**
- `pytest-asyncio` — pytest плагин
- `asynctest` — расширения для unittest (устаревает)
- `aioresponses` — мокирование aiohttp
- `AsyncMock` — мок для async функций

### Пример

```python
import pytest
import asyncio
from unittest.mock import AsyncMock, patch, MagicMock

# Тестируемый код
async def fetch_data(client, url):
    response = await client.get(url)
    return await response.json()

async def process_items(items):
    results = []
    for item in items:
        await asyncio.sleep(0.1)  # Simulate async work
        results.append(item * 2)
    return results

class AsyncService:
    def __init__(self, client):
        self.client = client

    async def get_user(self, user_id):
        data = await self.client.fetch(f"/users/{user_id}")
        return data

# === pytest-asyncio ===

# conftest.py
@pytest.fixture
def event_loop():
    """Создаёт новый event loop для каждого теста"""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

# Тесты
@pytest.mark.asyncio
async def test_process_items():
    result = await process_items([1, 2, 3])
    assert result == [2, 4, 6]

@pytest.mark.asyncio
async def test_with_mock():
    mock_client = AsyncMock()
    mock_client.get.return_value.json = AsyncMock(
        return_value={"data": "test"}
    )

    result = await fetch_data(mock_client, "http://example.com")
    assert result == {"data": "test"}
    mock_client.get.assert_called_once_with("http://example.com")

# Async fixtures
@pytest.fixture
async def async_client():
    client = await create_client()
    yield client
    await client.close()

@pytest.mark.asyncio
async def test_with_async_fixture(async_client):
    result = await async_client.fetch("/test")
    assert result is not None

# === AsyncMock детали ===

@pytest.mark.asyncio
async def test_async_mock():
    # Базовый AsyncMock
    mock = AsyncMock(return_value=42)
    result = await mock()
    assert result == 42

    # Side effect
    mock = AsyncMock(side_effect=ValueError("error"))
    with pytest.raises(ValueError):
        await mock()

    # Sequence of returns
    mock = AsyncMock(side_effect=[1, 2, 3])
    assert await mock() == 1
    assert await mock() == 2
    assert await mock() == 3

# === Мокирование aiohttp ===

import aiohttp
from aioresponses import aioresponses

@pytest.mark.asyncio
async def test_aiohttp_mock():
    with aioresponses() as mocked:
        mocked.get(
            'http://example.com/api',
            payload={'data': 'test'}
        )

        async with aiohttp.ClientSession() as session:
            async with session.get('http://example.com/api') as response:
                data = await response.json()
                assert data == {'data': 'test'}

# === Тестирование timeout ===

@pytest.mark.asyncio
async def test_timeout():
    async def slow_operation():
        await asyncio.sleep(10)
        return "done"

    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(slow_operation(), timeout=0.1)

# === Тестирование cancellation ===

@pytest.mark.asyncio
async def test_cancellation():
    cancelled = False

    async def cancellable():
        nonlocal cancelled
        try:
            await asyncio.sleep(10)
        except asyncio.CancelledError:
            cancelled = True
            raise

    task = asyncio.create_task(cancellable())
    await asyncio.sleep(0.1)
    task.cancel()

    with pytest.raises(asyncio.CancelledError):
        await task

    assert cancelled

# === Patch async methods ===

@pytest.mark.asyncio
async def test_patch_async():
    service = AsyncService(MagicMock())

    with patch.object(
        service.client, 'fetch',
        new_callable=AsyncMock,
        return_value={'name': 'John'}
    ):
        result = await service.get_user(1)
        assert result == {'name': 'John'}

# === unittest style ===

import unittest

class TestAsync(unittest.IsolatedAsyncioTestCase):
    async def test_process(self):
        result = await process_items([1, 2])
        self.assertEqual(result, [2, 4])

    async def asyncSetUp(self):
        self.client = await create_client()

    async def asyncTearDown(self):
        await self.client.close()

# === Тестирование конкурентности ===

@pytest.mark.asyncio
async def test_concurrent_access():
    counter = 0
    lock = asyncio.Lock()

    async def increment():
        nonlocal counter
        async with lock:
            temp = counter
            await asyncio.sleep(0.01)
            counter = temp + 1

    await asyncio.gather(*[increment() for _ in range(100)])
    assert counter == 100  # Проверяем thread-safety

# === Производительность async ===

@pytest.mark.asyncio
async def test_performance():
    import time

    start = time.time()
    await asyncio.gather(*[asyncio.sleep(0.1) for _ in range(100)])
    elapsed = time.time() - start

    # 100 sleep по 0.1s должны выполниться за ~0.1s, не за 10s
    assert elapsed < 0.5
```

### Типичные ошибки
- Забывать `@pytest.mark.asyncio`
- Не использовать AsyncMock для async функций
- Не изолировать event loop между тестами

### На интервью
**Как отвечать:** Покажите pytest-asyncio и AsyncMock, объясните изоляцию event loop, упомяните aioresponses для HTTP.

**Follow-up вопросы:**
- Как тестировать timeout?
- Как мокировать aiohttp?
- Чем AsyncMock отличается от MagicMock?

---

[← Назад к списку тем](README.md)
