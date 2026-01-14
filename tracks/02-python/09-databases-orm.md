# 09. Базы данных и ORM

[← Назад к списку тем](README.md)

---

## Вопрос 1: Как работать с SQLAlchemy? (Core vs ORM)

### Зачем спрашивают
SQLAlchemy — стандарт для работы с БД в Python. Важно понимать оба уровня.

### Короткий ответ
SQLAlchemy имеет два уровня: Core (SQL Expression Language) и ORM (Object-Relational Mapping). Core — низкоуровневый, близкий к SQL. ORM — высокоуровневый с маппингом объектов на таблицы. Для сложных запросов — Core, для CRUD — ORM.

### Пример

```python
from sqlalchemy import create_engine, Column, Integer, String, Table, MetaData, select
from sqlalchemy.orm import declarative_base, Session

# === Core ===
engine = create_engine("postgresql://user:pass@localhost/db")
metadata = MetaData()

users = Table('users', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String),
)

# Core запрос
with engine.connect() as conn:
    result = conn.execute(select(users).where(users.c.name == 'Alice'))
    for row in result:
        print(row)

# === ORM ===
Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)

# ORM запрос (2.0 style)
with Session(engine) as session:
    users = session.execute(select(User).where(User.name == 'Alice')).scalars().all()

# ORM CRUD
with Session(engine) as session:
    user = User(name='Bob')
    session.add(user)
    session.commit()
```

### На интервью
**Как отвечать:** Объясните два уровня, когда использовать каждый, покажите 2.0 style синтаксис.

---

## Вопрос 2: Как управлять сессиями и транзакциями?

### Зачем спрашивают
Правильное управление сессиями критично для надёжности и производительности.

### Короткий ответ
Session — контекст для операций с БД. Используйте context manager для автоматического закрытия. Транзакции управляются через `commit()` и `rollback()`. `sessionmaker` создаёт фабрику сессий. Для FastAPI — зависимость с yield.

### Пример

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

engine = create_engine("postgresql://...")
SessionLocal = sessionmaker(bind=engine)

# Context manager
with SessionLocal() as session:
    user = User(name='Alice')
    session.add(user)
    session.commit()

# Транзакции
with SessionLocal() as session:
    try:
        session.add(user1)
        session.add(user2)
        session.commit()
    except Exception:
        session.rollback()
        raise

# Nested transactions (savepoints)
with SessionLocal() as session:
    session.add(user1)
    with session.begin_nested():  # Savepoint
        session.add(user2)
        # Если ошибка - откатится только user2
    session.commit()

# FastAPI dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
def get_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

### На интервью
**Как отвечать:** Покажите context manager, объясните commit/rollback, упомяните dependency для FastAPI.

---

## Вопрос 3: Что такое Unit of Work паттерн в SQLAlchemy?

### Зачем спрашивают
UoW — основа работы Session. Понимание важно для отладки.

### Короткий ответ
Unit of Work отслеживает изменения объектов в Session и применяет их одной транзакцией при commit(). Session имеет identity map (кэш объектов) и отслеживает new, dirty, deleted объекты. Flush записывает изменения в БД, commit фиксирует транзакцию.

### Пример

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    user = User(name='Alice')
    session.add(user)

    print(session.new)     # {User(name='Alice')}
    print(session.dirty)   # set()

    session.flush()  # SQL выполнен, но транзакция открыта
    print(user.id)   # ID присвоен

    user.name = 'Bob'
    print(session.dirty)   # {User(name='Bob')}

    session.commit()  # Транзакция зафиксирована

# Identity Map
with Session(engine) as session:
    user1 = session.get(User, 1)
    user2 = session.get(User, 1)
    assert user1 is user2  # Один объект!

# Expunge - убрать из session
session.expunge(user)

# Refresh - перечитать из БД
session.refresh(user)
```

### На интервью
**Как отвечать:** Объясните tracking изменений, identity map, разницу flush vs commit.

---

## Вопрос 4: Как работать с миграциями? (Alembic)

### Зачем спрашивают
Миграции — обязательная часть работы с БД в production.

### Короткий ответ
Alembic — инструмент миграций для SQLAlchemy. Отслеживает изменения моделей и генерирует скрипты миграций. Команды: `revision` (создать), `upgrade` (применить), `downgrade` (откатить). Храните миграции в git.

### Пример

```bash
# Инициализация
alembic init alembic

# Создание миграции
alembic revision --autogenerate -m "add users table"

# Применение
alembic upgrade head
alembic upgrade +1    # На одну вперёд
alembic downgrade -1  # На одну назад

# Текущая версия
alembic current
alembic history
```

```python
# alembic/env.py
from app.models import Base
target_metadata = Base.metadata

# migrations/versions/xxx_add_users.py
def upgrade():
    op.create_table('users',
        sa.Column('id', sa.Integer, primary_key=True),
        sa.Column('name', sa.String(50)),
    )

def downgrade():
    op.drop_table('users')
```

### На интервью
**Как отвечать:** Покажите базовые команды, объясните autogenerate, упомяните хранение в git.

---

## Вопрос 5: Как работать с async SQLAlchemy?

### Зачем спрашивают
Async — для высоконагруженных приложений. Важно для FastAPI.

### Короткий ответ
SQLAlchemy 1.4+ поддерживает async через `asyncpg` или `aiosqlite`. Используйте `create_async_engine` и `AsyncSession`. Для FastAPI — async dependency с yield.

### Пример

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy import select

# Async engine
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
async_session = async_sessionmaker(engine, class_=AsyncSession)

# Async запросы
async def get_users():
    async with async_session() as session:
        result = await session.execute(select(User))
        return result.scalars().all()

# FastAPI dependency
async def get_db():
    async with async_session() as session:
        yield session

@app.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()

# Создание таблиц
async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

### На интервью
**Как отвечать:** Покажите async engine и session, объясните run_sync для sync операций.

---

## Вопрос 6: Как тестировать код с базой данных?

### Зачем спрашивают
Тестирование с БД — сложная тема. Показывает production опыт.

### Короткий ответ
Варианты: in-memory SQLite, тестовая БД с rollback, testcontainers. Используйте fixtures для setup/teardown. Изолируйте тесты через транзакции с rollback. Для интеграционных — testcontainers.

### Пример

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Fixture с SQLite
@pytest.fixture
def db():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.close()

# Fixture с rollback
@pytest.fixture
def db_with_rollback(db):
    db.begin_nested()
    yield db
    db.rollback()

# Testcontainers
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:15") as pg:
        yield pg.get_connection_url()

@pytest.fixture
def db(postgres):
    engine = create_engine(postgres)
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    with Session() as session:
        yield session

# Тест
def test_create_user(db):
    user = User(name="Alice")
    db.add(user)
    db.commit()
    assert db.query(User).count() == 1
```

### На интервью
**Как отвечать:** Покажите SQLite in-memory для unit тестов, testcontainers для интеграционных.

---

## Вопрос 7: ORM vs Query Builder vs Raw SQL - когда что использовать?

### Зачем спрашивают
Trade-offs между подходами. Показывает архитектурное мышление.

### Короткий ответ
**ORM** — для CRUD, domain models, когда важна абстракция. **Query Builder** (Core) — для сложных запросов, отчётов, когда нужен контроль. **Raw SQL** — для максимальной производительности, специфичных фич БД. Комбинируйте по необходимости.

### Пример

```python
# ORM - простой CRUD
user = User(name="Alice")
session.add(user)
session.commit()

users = session.query(User).filter(User.age > 18).all()

# Query Builder (Core) - сложные запросы
from sqlalchemy import select, func, join

stmt = (
    select(User.name, func.count(Order.id).label('order_count'))
    .select_from(join(User, Order))
    .group_by(User.name)
    .having(func.count(Order.id) > 5)
)
result = session.execute(stmt)

# Raw SQL - специфичные фичи
from sqlalchemy import text

result = session.execute(text("""
    SELECT * FROM users
    WHERE name @@ to_tsquery('alice & bob')
"""))

# Комбинация
from sqlalchemy import literal_column

stmt = select(User).where(
    literal_column("name").op("~")("^A.*")  # PostgreSQL regex
)
```

### На интервью
**Как отвечать:** Объясните trade-offs (абстракция vs контроль vs производительность), покажите когда комбинировать.

---

## См. также

- [Основы SQL](../06-databases/00-sql-fundamentals.md) — фундаментальные концепции SQL
- [Базы данных и persistence в Go](../00-go/06-databases-persistence.md) — работа с БД в Go

---

[← Назад к списку тем](README.md)
