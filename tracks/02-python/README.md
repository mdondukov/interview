# Трек: Python Core

Фокус: глубокое понимание языка, управления памятью, асинхронности и практических паттернов для Python инженера.

## Структура

- Каждая тема — файл `NN-topic-slug.md`
- Модули отсортированы по частоте вопросов на интервью

## Темы и прогресс

### Основы (чаще всего спрашивают)
- [ ] [00-language-overview](00-language-overview.md) — философия, CPython, generators, comprehensions
- [ ] [01-data-structures](01-data-structures.md) — list, dict, set, collections, Big O
- [ ] [02-memory-gc](02-memory-gc.md) — GIL, reference counting, GC, interning
- [ ] [03-oop-protocols](03-oop-protocols.md) — MRO, Protocol, dataclass, descriptors, slots

### Конкурентность и практика
- [ ] [04-asyncio-concurrency](04-asyncio-concurrency.md) — event loop, async/await, threading, multiprocessing
- [ ] [05-errors-debugging](05-errors-debugging.md) — exceptions, context managers, logging, pdb
- [ ] [06-tooling-testing](06-tooling-testing.md) — pytest, fixtures, mock, poetry, ruff

### Типизация и веб
- [ ] [07-type-system](07-type-system.md) — typing, generics, Protocol, TypeVar, mypy
- [ ] [08-fastapi-web](08-fastapi-web.md) — FastAPI, DI, middleware, Pydantic, testing
- [ ] [09-databases-orm](09-databases-orm.md) — SQLAlchemy, Alembic, transactions, async

### Senior/Architect level
- [ ] [10-performance-profiling](10-performance-profiling.md) — cProfile, py-spy, optimization, Cython
- [ ] [11-packaging-deployment](11-packaging-deployment.md) — pyproject.toml, Docker, config management
- [ ] [12-metaprogramming](12-metaprogramming.md) — decorators, metaclasses, descriptors, AST

## Структура каждого вопроса

Каждый вопрос содержит 6 секций:
1. **Зачем спрашивают** — контекст вопроса
2. **Короткий ответ** — 2-3 предложения для быстрого ответа
3. **Детальный разбор** — глубокое объяснение с подзаголовками
4. **Пример** — рабочий код
5. **Типичные ошибки** — что проверяет интервьюер
6. **На интервью** — как отвечать + follow-up вопросы

## Статистика

- **Модулей:** 13
- **Вопросов:** ~102
- **Уровень:** Middle → Senior → Architect
