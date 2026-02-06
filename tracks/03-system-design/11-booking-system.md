# 11 — Booking System (Hotel/Flights/Events)

Развёрнутые вопросы и ответы про системы бронирования: управление инвентарем, предотвращение двойного бронирования, распределённые блокировки, оверубинг, управление очередями, масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [10-payment-system](./10-payment-system.md) · Следующая тема: [12-ride-sharing](./12-ride-sharing.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Double Booking** — ситуация, когда две системы или два пользователя одновременно бронируют одно и то же место, что приводит к перепроданию. Это одна из самых критичных проблем в системах бронирования, так как нарушает доверие пользователей и может привести к финансовым потерям. Предотвращение двойного бронирования требует сильной консистентности данных и координации между всеми компонентами системы. В реальной жизни это означает, что при попытке забронировать место система должна гарантировать, что никто другой не забронировал его в этот же момент.

**Pessimistic Locking** — стратегия контроля параллелизма, при которой ресурс блокируется перед началом его изменения (например, с помощью SQL команды FOR UPDATE). Этот подход гарантирует безопасность данных в условиях высокой контентеции, когда много пользователей одновременно пытаются получить доступ к одному ресурсу. Недостаток этого подхода в том, что блокировка может привести к снижению производительности, если много запросов будут ждать друг друга. Pessimistic Locking лучше всего подходит для систем с частыми конфликтами доступа.

**Optimistic Locking** — более гибкая стратегия, при которой система проверяет версию объекта при обновлении и отклоняет обновление, если версия изменилась с момента чтения. Этот подход быстрее при низкой контентеции, так как не требует блокирования ресурсов. Однако он требует логики повторных попыток (retry logic) на клиентской стороне в случае конфликта версий. Optimistic Locking хорошо работает в системах с редкими конфликтами доступа.

**Inventory Management** — процесс отслеживания доступного количества комнат, мест или билетов по датам в системе бронирования. Это основа системы, требующая как быстрого чтения данных о доступности для поиска, так и безопасного обновления инвентаря при создании новых бронирований. Инвентарь должен быть согласованным между всеми источниками (веб-сайт, мобильное приложение, партнёрские системы). Эффективное управление инвентарём критично для предотвращения перепродаж.

**Overbooking** — намеренное бронирование большего количества мест, чем физически доступно, основанное на историческом анализе процента отмен. Авиакомпании и отели используют эту стратегию для компенсации no-shows (когда забронировавшие не приходят). Нужна специальная обработка при check-in: если приходит больше людей чем мест, может потребоваться bumping (перебронирование на другой рейс с компенсацией).

**Reservation Hold** — временное резервирование места на период 5-10 минут, во время которого пользователь может завершить платёж. Это предотвращает потерю места по причине медленного процесса оплаты и улучшает пользовательский опыт. После истечения времени холда место освобождается для других пользователей. Система должна отслеживать время холда и автоматически отменять устаревшие холды.

**Distributed Lock** — механизм синхронизации, реализованный через внешнее хранилище (например, Redis), вместо традиционных блокировок базы данных. Это необходимо в распределённых системах, где несколько процессов или сервисов работают одновременно и нуждаются в экклюзивном доступе к ресурсам. Распределённые блокировки помогают избежать race conditions в микросервисной архитектуре. Примером является Redis SETNX или более сложные системы вроде Redlock.

**Idempotency Key** — уникальный идентификатор запроса, который позволяет системе определить, был ли этот запрос обработан ранее. Это предотвращает двойные платежи и другие побочные эффекты при повторных попытках клиента (retries). Если клиент отправит один и тот же запрос дважды с одним idempotency key, сервер вернёт результат первой обработки вместо повторной обработки.

**Rate Limiting** — техника ограничения количества запросов, которые клиент может отправить в единицу времени. Это защищает систему от флеш-распродаж (когда все пользователи пытаются забронировать одно место одновременно) и от DDoS атак. Rate Limiting может быть реализовано на уровне пользователя, IP-адреса или API ключа. Правильно настроенный rate limiting обеспечивает справедливое распределение нагрузки.

**Cache Invalidation** — процесс обновления кеша при изменении данных инвентаря в основной базе данных. Это критично для системы бронирования, так как если кеш содержит устаревшую информацию о доступности, пользователи будут видеть неправильные данные и пытаться забронировать недоступные места. Правильная кеш-инвалидация требует тщательного проектирования и может быть одним из самых сложных аспектов системы.

---

## Вопросы и разборы

### 1. Как спроектировать систему бронирования: требования и архитектура?

**Зачем спрашивают.** Базовый вопрос для понимания масштаба, требований консистентности и основных компонентов. Проверяет умение выделить критичные свойства: нет двойного бронирования, низкая latency поиска, высокая доступность.

**Короткий ответ.** Система бронирования требует сильной консистентности при создании бронирования (без двойных заказов) и высокой доступности при поиске. Архитектура: поиск через Elasticsearch, бронирование через SQL с блокировками, очередь для пиков нагрузки.

**Детальный разбор.**

**Требования системы:**

Функциональные:
- Поиск доступного инвентаря (отели, рейсы, события)
- Просмотр деталей и цен
- Создание/отмена бронирований
- История бронирований пользователя
- Уведомления (подтверждение, напоминания)
- Интеграция с платежами

Нефункциональные:
- No double booking (strong consistency)
- Высокая доступность поиска (99.5%+)
- Low latency поиска: <200ms
- Поддержка флеш-распродаж (1000+ одновременных запросов)
- Оверубинг (компенсация за no-shows)

**Estimation для отельного сервиса:**

```
- 500K отелей × 100 номеров = 50M room-nights
- 10M поисков/день = 115 QPS avg, 600 QPS peak
- 500K бронирований/день = 6 QPS avg, 30 QPS peak
- Пиковая нагрузка: 5x в праздники

Storage:
- Метаданные отелей: 500K × 10KB = 5GB
- Инвентарь (room-nights): 50M × 365 × 100B = 1.8TB
- Бронирования/год: 500K × 2KB × 365 = 365GB
```

**Архитектура:**

```
┌──────────────────────────────────────────────────────┐
│                   Client/Frontend                     │
└───────┬──────────────────────────────────┬────────────┘
        │                                  │
        │ (Search)                         │ (Booking)
        │                                  │
    ┌───▼─────────────┐          ┌────────▼──────────┐
    │  API Gateway    │          │  Booking Service  │
    │  + Rate Limit   │          │  + Reservation    │
    └───┬─────────────┘          └────────┬──────────┘
        │                                 │
    ┌───▼──────────────┐         ┌────────▼────────┐
    │ Search Service   │         │ Inventory       │
    │ (Stateless)      │         │ Service         │
    └───┬──────────────┘         └────────┬────────┘
        │                                 │
    ┌───▼──────────────┐         ┌────────▼────────┐
    │ Elasticsearch    │         │ PostgreSQL      │
    │ (50ms read)      │         │ (Consistency)   │
    └──────────────────┘         └────────┬────────┘
                                         │
                                  ┌──────▼────────┐
                                  │ Redis (Locks) │
                                  │ (30ms)        │
                                  └───────────────┘
```

**API Design:**

```json
GET /api/v1/hotels/search?location=NYC&checkin=2024-06-01&checkout=2024-06-03

Response:
{
  "hotels": [
    {
      "hotel_id": "htl_123",
      "name": "Grand Hotel",
      "rating": 4.5,
      "rooms": [
        {
          "room_type_id": "rt_456",
          "name": "Deluxe King",
          "price_per_night": 19900,
          "available_count": 5
        }
      ]
    }
  ]
}

POST /api/v1/bookings
X-Idempotency-Key: uuid
{
  "hotel_id": "htl_123",
  "room_type_id": "rt_456",
  "checkin": "2024-06-01",
  "checkout": "2024-06-03",
  "guest": {...},
  "payment_method_id": "pm_xxx"
}

Response:
{
  "booking_id": "bk_abc123",
  "status": "confirmed",
  "confirmation_code": "ABCD1234",
  "total_amount": 39800
}
```

**Типичные ошибки.**

1. Проверить availability и забронировать в разные транзакции → double booking
2. Кешировать availability без синхронизации → overbooking
3. Блокировать весь инвентарь вместо date range → deadlock
4. Одна база данных для search и booking → контention
5. Забыть про idempotency → duplicate charges

**На интервью.**

Начните с требований:
- "Нет двойного бронирования — strong consistency при создании"
- "Поиск может быть eventual consistent, но быстрый"
- "Нужно выдержать флеш-распродажи"

Обсудите разделение забот:
- Search: Elasticsearch + cache (eventual consistency)
- Booking: PostgreSQL + locks (strong consistency)
- High concurrency: Redis locks, queue, reservation timeout

Типичные follow-up:
- Как обработать групповое бронирование (10+ комнат)?
- Как реализовать waitlist при sold out?
- Как синхронизировать с GDS (Global Distribution Systems)?
- Как обрабатывать timezone?

---

### 2. Как предотвратить двойное бронирование одного места?

**Зачем спрашивают.** Главная проблема систем бронирования. Проверяет знание блокировок, транзакций, уровней изоляции, оптимистичного vs пессимистичного locking. Это отличает junior разработчика от senior.

**Короткий ответ.** Три подхода: пессимистичное блокирование (FOR UPDATE в одной транзакции), оптимистичное (version check), или распределённая блокировка (Redis). Пессимистичное лучше для высокой контентеции, оптимистичное — для низкой.

**Детальный разбор.**

**Подход 1: Пессимистичное блокирование (FOR UPDATE)**

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Заблокировать все room-nights для date range
SELECT * FROM room_inventory
WHERE room_type_id = $1
  AND date >= $2 AND date < $3
FOR UPDATE;

-- Проверить доступность
-- (в цикле для каждого дня)

-- Обновить счётчик
UPDATE room_inventory
SET booked_rooms = booked_rooms + 1
WHERE id = $1;

-- Создать бронирование
INSERT INTO bookings (...)
VALUES (...);

COMMIT;
```

Плюсы:
- Гарантирует consistency в одной транзакции
- Простая логика (нет retry)
- Хорошо для высокой контентеции

Минусы:
- Может быть медленно при большом date range
- Deadlock risk если несколько транзакций пересекаются
- Lock hold время долгое (платёж внутри?)

```
Временная шкала:
T0:  User1 locks room for 6/1-6/3
T1:  User2 tries lock → BLOCKED (ждёт 50ms)
T2:  User1 commits
T3:  User2 acquires lock, checks, books
```

**Подход 2: Оптимистичное блокирование (Version)**

```sql
-- Прочитать инвентарь
SELECT id, booked_rooms, version FROM room_inventory
WHERE room_type_id = $1 AND date >= $2 AND date < $3;

-- Проверить доступность в памяти
// if any day has no room, fail

-- Обновить с версией
BEGIN;
  UPDATE room_inventory
  SET booked_rooms = booked_rooms + 1,
      version = version + 1
  WHERE id = $1 AND version = $2
    AND (total_rooms - booked_rooms - blocked_rooms) >= 1;

  IF rows_affected == 0:
    ROLLBACK
    retry with exponential backoff
  ELSE:
    INSERT INTO bookings (...)
    COMMIT
```

Плюсы:
- Нет долгих блокировок
- Нет deadlock'ов
- Лучше для low contention

Минусы:
- Нужно retry логику
- Медленнее при высокой контентеции (много conflicts)
- Сложнее для debug

```
Временная шкала:
T0:  User1 reads room_inventory (version=5)
T1:  User2 reads room_inventory (version=5)
T2:  User1 commits (version→6)
T3:  User2 tries update WHERE version=5 → 0 rows
T4:  User2 retries, reads (version=6), updates
```

**Подход 3: Распределённая блокировка (Redis)**

```python
async def create_booking(request):
    lock_key = f"lock:hotel:{room_type_id}:{checkin}:{checkout}"

    async with redis_distributed_lock(lock_key, timeout=30):
        # Только один запрос в этом моменте
        availability = await check_availability(...)

        if not available:
            raise NoAvailability()

        booking = await create_booking_record(request)
        return booking
```

Реализация Redis lock:

```python
async def acquire_lock(redis, key, ttl=30):
    token = uuid.uuid4()
    acquired = await redis.set(
        key,
        token,
        nx=True,  # Only if not exists
        ex=ttl    # Expiry
    )

    if not acquired:
        raise LockBusy()

    return token

async def release_lock(redis, key, token):
    # Lua script to ensure atomic check-and-delete
    script = """
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
    """

    await redis.eval(script, 1, key, token)
```

Плюсы:
- Простая логика
- Масштабируется на несколько сервисов
- Нет database contention

Минусы:
- Redis отказ → потеря консистентности
- Нужно handle lock expiry (what if payment takes 40s?)
- Доп. сервис для операции

**Сравнение подходов:**

```
                   Latency   Contention   Deadlock  Simplicity
Pessimistic        High      Low          Yes       High
Optimistic         Low       High         No        Medium
Redis Lock         Medium    Medium       No        High
```

**Рекомендация по выбору:**

- **Высокая контентеция** (флеш-распродажи): Pessimistic + queue
- **Нормальная нагрузка**: Optimistic для search, pessimistic для booking
- **Распределённая система**: Redis lock с backup DB lock
- **Очень критичный**: Dual locks (DB + Redis)

**Типичные ошибки.**

1. FOR UPDATE без SERIALIZABLE isolation → phantom read
2. Забыть lock на release inventory при отмене
3. Check и update в разных транзакциях
4. Redis lock без fallback если Redis down
5. Lock timeout слишком короткий (не хватает на платёж)

**На интервью.**

Спросят: "Что если высокая контентеция?"
→ Pessimistic с небольшим date range, или очередь

Спросят: "Как избежать deadlock?"
→ Consistent lock order (всегда сортировать room_ids)

Спросят: "Что если платёж медленный?"
→ Зарезервировать сначала (Redis 5 min), потом платёж, потом подтвердить

Код для показа: pessimistic FOR UPDATE — самый очевидный и правильный.

---

### 3. Как спроектировать эффективное управление инвентарем?

**Зачем спрашивают.** Инвентарь — центр системы. Проверяет понимание data model, индексов, batch updates, кеширования. Нужно обсудить date-based структуру, оверубинг, блокировку.

**Короткий ответ.** Инвентарь должен быть per-date, не per-room. Структура: `room_inventory(room_type_id, date, booked, blocked, total)`. Индексы по room_type_id и date для быстрых range queries. Кеш для поиска, прямое обновление в БД для бронирования.

**Детальный разбор.**

**Плохой дизайн:**

```sql
CREATE TABLE rooms (
    id UUID PRIMARY KEY,
    room_number VARCHAR(10),
    hotel_id UUID,
    room_type_id UUID,
    status VARCHAR(20)  -- 'available', 'booked', 'maintenance'
);

-- Проблема: для 50M номеров, каждый день новый статус
-- Нужна таблица per-room per-night = огромная
-- Индексы неэффективны
```

**Хороший дизайн:**

```sql
CREATE TABLE room_inventory (
    id UUID PRIMARY KEY,
    hotel_id UUID NOT NULL,
    room_type_id UUID NOT NULL,
    date DATE NOT NULL,

    total_rooms INT NOT NULL,          -- Capacity
    booked_rooms INT DEFAULT 0,        -- Confirmed bookings
    blocked_rooms INT DEFAULT 0,       -- Maintenance, staff blocks
    reserved_rooms INT DEFAULT 0,      -- Temporary holds (flash sale)

    base_price INT NOT NULL,
    dynamic_price INT,                 -- Surge pricing

    version INT DEFAULT 1,
    updated_at TIMESTAMP,

    UNIQUE(room_type_id, date),
    INDEX idx_room_date (room_type_id, date),
    INDEX idx_hotel_date (hotel_id, date),

    CONSTRAINT available_check CHECK (
        booked_rooms + blocked_rooms + reserved_rooms <= total_rooms
    )
);
```

**Определение доступности:**

```
available = total_rooms - booked_rooms - blocked_rooms - reserved_rooms

Статусы комнат:
┌─────────────────┐
│  total_rooms=10 │
├─────────────────┤
│ booked=5        │  (confirmed bookings)
│ blocked=1       │  (maintenance, staff)
│ reserved=2      │  (flash sale timeout)
├─────────────────┤
│ available=2     │  (can book now)
└─────────────────┘
```

**Оперативная эффективность:**

```python
class InventoryManager:
    async def check_availability(self, room_type_id, checkin, checkout):
        """
        Эффективная проверка: один SQL запрос
        """
        rows = await db.fetch("""
            SELECT id, date,
                   total_rooms - booked_rooms - blocked_rooms - reserved_rooms as available
            FROM room_inventory
            WHERE room_type_id = $1
              AND date >= $2 AND date < $3
            ORDER BY date
        """, room_type_id, checkin, checkout)

        # Найти узкое место (день с минимальной доступностью)
        min_available = min(r['available'] for r in rows)

        return min_available > 0, rows

    async def reserve_rooms(self, room_type_id, checkin, checkout, count, duration_sec):
        """
        Temporary hold using reserved_rooms
        """
        await db.execute("""
            UPDATE room_inventory
            SET reserved_rooms = reserved_rooms + $1
            WHERE room_type_id = $2
              AND date >= $3 AND date < $4
              AND (total_rooms - booked_rooms - blocked_rooms - reserved_rooms) >= $1
        """, count, room_type_id, checkin, checkout)

        # Запланировать release через duration_sec
        await queue.schedule_release(
            {room_type_id, checkin, checkout, count},
            delay=duration_sec
        )

    async def confirm_booking(self, room_type_id, checkin, checkout, count):
        """
        Convert reserved → booked
        """
        await db.execute("""
            UPDATE room_inventory
            SET reserved_rooms = reserved_rooms - $1,
                booked_rooms = booked_rooms + $1
            WHERE room_type_id = $2
              AND date >= $3 AND date < $4
        """, count, room_type_id, checkin, checkout)
```

**Batch Updates для обновления цен:**

```python
async def update_dynamic_pricing(self, room_type_id, date_range, pricing_fn):
    """
    Обновить цены для всего date range за раз (не per-night)
    """
    rows = await db.fetch("""
        SELECT id, date, booked_rooms FROM room_inventory
        WHERE room_type_id = $1 AND date >= $2 AND date < $3
    """, room_type_id, date_range['start'], date_range['end'])

    updates = []
    for row in rows:
        occupancy = row['booked_rooms'] / row['total_rooms']
        new_price = pricing_fn(occupancy)
        updates.append((row['id'], new_price))

    # Batch update
    await db.execute("""
        UPDATE room_inventory SET dynamic_price = v.price
        FROM (SELECT * FROM UNNEST($1::uuid[], $2::int[])) as v(id, price)
        WHERE room_inventory.id = v.id
    """, [u[0] for u in updates], [u[1] for u in updates])
```

**Кеширование для поиска:**

```python
class CachedInventory:
    """
    Кеш availability для поиска (не critical accuracy)
    """

    async def get_availability_cached(self, room_type_id, checkin, checkout, ttl_sec=300):
        cache_key = f"avail:{room_type_id}:{checkin}:{checkout}"

        # Попробовать кеш
        cached = await redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # Иначе — из БД
        avail = await self.check_availability(room_type_id, checkin, checkout)

        # Кешировать на 5 минут
        await redis.setex(cache_key, ttl_sec, json.dumps(avail))

        return avail

    async def invalidate_on_booking(self, room_type_id, checkin, checkout):
        """
        Инвалидировать кеш при бронировании
        """
        # Все пересекающиеся date ranges
        pattern = f"avail:{room_type_id}:*"
        keys = await redis.keys(pattern)

        # Удалить пересекающиеся
        for key in keys:
            parts = key.split(':')
            cached_start = parts[2]
            cached_end = parts[3]

            if overlaps(cached_start, cached_end, checkin, checkout):
                await redis.delete(key)
```

**Масштабирование инвентаря:**

```
Размер:
- 1B отелей × 100 room-types × 365 дней × 100 bytes
= 3.65TB

Разбиение по партициям:
- По hotel_id (каждый отель в своей партиции)
- Или по date (каждый месяц в своей партиции)
- Или по room_type_id (каждый тип комнаты)

Рекомендация: по hotel_id для locality
```

**Типичные ошибки.**

1. Хранить inventory per-room вместо per-room-type
2. Обновлять inventory асинхронно → overbooking
3. Не проверять constraint при UPDATE
4. Забыть индекс на date
5. Не инвалидировать кеш при бронировании

**На интервью.**

Вопрос: "Как обновить цены для 50K номеров?"
→ Batch update, не per-room, группировать по room_type

Вопрос: "Как хранить историю цен?"
→ Отдельная таблица price_history с snapshot per-date

Вопрос: "Что если нужны разные цены per-bed-count?"
→ Иерархия: hotel → room_type → variant (1bed, 2beds)

---

### 4. Оптимистичное vs пессимистичное блокирование: когда что использовать?

**Зачем спрашивают.** Фундаментальное решение в распределённых системах. Проверяет понимание trade-offs, конкурентности, перформанса. Часто идёт как follow-up на предыдущий вопрос.

**Короткий ответ.** Пессимистичное: lock сразу, медленнее, но безопаснее при высокой контентеции. Оптимистичное: lock только при update, быстрее, но retry при conflict. Выбор зависит от rate conflict'ов: если <10% — optimistic, если >50% — pessimistic.

**Детальный разбор.**

**Оптимистичное блокирование (Optimistic Locking):**

Идея: читать данные, затем при update проверить, что они не изменились.

```sql
-- Schema
CREATE TABLE bookings (
    id UUID PRIMARY KEY,
    status VARCHAR(20),
    total_amount INT,
    version INT DEFAULT 1,  -- Incrementing version
    updated_at TIMESTAMP
);

-- Read
SELECT * FROM bookings WHERE id = $1;  -- Получим version=5

-- Бизнес логика (вне БД)
booking.total_amount = calculate_refund();

-- Write с проверкой версии
UPDATE bookings
SET status = 'refunded',
    total_amount = $1,
    version = version + 1,
    updated_at = NOW()
WHERE id = $2 AND version = $3;  -- $3 = старая версия (5)

IF rows_affected == 0:
    -- Конфликт! Кто-то изменил между read и write
    -- Retry: прочитать заново, пересчитать, попробовать снова
    RETURN CONFLICT
```

**Реализация с retry:**

```python
async def update_booking_with_retry(booking_id, update_fn, max_retries=3):
    for attempt in range(max_retries):
        try:
            # 1. Read
            booking = await db.get_booking(booking_id)
            old_version = booking.version

            # 2. Compute
            new_state = update_fn(booking)

            # 3. Write with version check
            result = await db.execute("""
                UPDATE bookings
                SET status = $1, total_amount = $2, version = version + 1
                WHERE id = $3 AND version = $4
            """, new_state.status, new_state.amount, booking_id, old_version)

            if result.rowcount == 0:
                raise VersionConflict()

            return new_state

        except VersionConflict:
            if attempt == max_retries - 1:
                raise

            # Exponential backoff
            await asyncio.sleep(0.01 * (2 ** attempt))
```

**Пессимистичное блокирование (Pessimistic Locking):**

Идея: lock на read, никто не может изменить, затем safely update.

```sql
-- Read with lock
SELECT * FROM bookings WHERE id = $1 FOR UPDATE;  -- Exclusive lock

-- Бизнес логика
booking.total_amount = calculate_refund();

-- Safe to update (никто не изменит между read и write)
UPDATE bookings
SET status = 'refunded', total_amount = $1
WHERE id = $2;

-- Автоматически release lock при COMMIT
```

**Параллелизм: Optimistic vs Pessimistic**

```
Сценарий: 10 юзеров пытаются отредактировать бронирование

PESSIMISTIC (FOR UPDATE):
T0:  User1 locks
T1:  User2 tries lock → BLOCKED (waits)
T2:  User3 tries lock → BLOCKED (waits)
...
T10: User1 commits, releases lock
T11: User2 acquires lock → BLOCKED → commits
T12: User3 acquires lock → BLOCKED → commits
...
Timeline: T0 → T11 → T22 → T33  (serial)

OPTIMISTIC (version):
T0:  User1 reads (version=1)
T1:  User2 reads (version=1)
T2:  User3 reads (version=1)
...
T10: User1 commits (version→2)
T11: User2 tries update WHERE version=1 → conflict, retry
T12: User3 tries update WHERE version=1 → conflict, retry
T13: User2 reads (version=2), retries
...
Timeline: T0 → T13 → T24  (faster when retries are rare)
```

**Выбор по контентеции:**

```python
class AdaptiveLocking:
    def choose_locking_strategy(self, resource_id):
        # Измерить conflict rate за последний час
        recent_conflicts = await metrics.get_conflicts(resource_id)

        conflict_rate = recent_conflicts / total_operations

        if conflict_rate > 0.50:
            return "PESSIMISTIC"  # Много конфликтов → lock сразу
        elif conflict_rate > 0.10:
            return "HYBRID"  # Среднее → оптимистичное, но с quick fallback
        else:
            return "OPTIMISTIC"  # Мало конфликтов → оптимистичное
```

**Hybrid подход: Optimistic с быстрым fallback**

```python
async def update_with_hybrid(resource_id, update_fn):
    # Попробовать оптимистичное
    try:
        return await update_optimistic(resource_id, update_fn)

    except VersionConflict:
        # Fallback на пессимистичное
        return await update_pessimistic(resource_id, update_fn)

async def update_optimistic(resource_id, update_fn):
    resource = await db.get_resource(resource_id)
    new_state = update_fn(resource)

    result = await db.execute("""
        UPDATE resources
        SET ... version = version + 1
        WHERE id = $1 AND version = $2
    """, resource_id, resource.version)

    if result.rowcount == 0:
        raise VersionConflict()

    return new_state

async def update_pessimistic(resource_id, update_fn):
    async with db.transaction():
        resource = await db.get_resource_locked(resource_id)  # FOR UPDATE
        new_state = update_fn(resource)

        await db.execute("""
            UPDATE resources SET ... WHERE id = $1
        """, resource_id)

        return new_state
```

**Сравнение таблица:**

```
                    Optimistic          Pessimistic
Lock Hold Time      Short (only update) Long (read → update)
Concurrency         High (parallel)     Low (serial)
Conflict Cost       Retry overhead      Lock wait
Best For            Low conflict        High conflict
Deadlock Risk       No                  Yes
Complexity          Medium              Low

Scenario: Booking confirmation
- Optimistic: 100ms (read) + 1ms (update) = 101ms, no wait
- Pessimistic: 100ms (locked read) → 1ms (update), others wait

If 10 concurrent users:
- Optimistic: 10 in parallel → 101ms total
- Pessimistic: 10 serialized → 1010ms total

But if 50% retry (high conflict):
- Optimistic: 10 → conflict → retry → 202ms
- Pessimistic: still 1010ms

Break-even: when retry_rate * retry_cost > lock_wait_cost
```

**Типичные ошибки.**

1. Optimistic на высоко-контентиве ресурсе → высокий retry rate
2. Pessimistic на low-contention → неё нужное wait
3. Забыть increment version при update
4. Retry без backoff → thundering herd
5. Lock timeout слишком короткий

**На интервью.**

Спросят: "Flash sale с 1000 одновременных запросов?"
→ Pessimistic при очень высокой контентеции, или queue

Спросят: "Как избежать retry при optimistic?"
→ Client-side locking (получить version перед update)

Спросят: "Что если transaction очень долгий?"
→ Pessimistic + timeout, или split на несколько коротких транзакций

---

### 5. Как реализовать распределённую блокировку (Distributed Lock)?

**Зачем спрашивают.** Необходимо для координации в микросервисной архитектуре. Проверяет знание Redis, Zookeeper, или других механизмов, а также понимание edge cases (lock expiry, network partition).

**Короткий ответ.** Redis distributed lock через SET NX EX (set if not exists с expiry). Реальное имплементация требует Lua скрипта для atomic release (проверить token перед удалением). Fallback: DB lock если Redis недоступен.

**Детальный разбор.**

**Базовая реализация (неправильная):**

```python
# ❌ Проблемы в этом коде!

async def acquire_lock(redis, key, ttl=30):
    acquired = await redis.set(key, "locked", nx=True, ex=ttl)
    return acquired

async def release_lock(redis, key):
    await redis.delete(key)  # ❌ DANGER: может удалить чужую блокировку!

# Сценарий:
# T0:  Client1 acquires lock with ttl=30
# T15: Network delay, Client1 thinks it has lock
# T29: Lock expires (ttl=30 пройдёт)
# T30: Client2 acquires lock
# T31: Client1 resumes, calls release_lock → удаляет Client2's lock!
```

**Правильная реализация с Lua:**

```python
import uuid
import redis

class DistributedLock:
    def __init__(self, redis_client, key, ttl=30):
        self.redis = redis_client
        self.key = key
        self.ttl = ttl
        self.token = str(uuid.uuid4())  # Unique token

    async def acquire(self, timeout=None):
        """
        Попробовать获取 lock с таймаутом
        """
        acquired = await self.redis.set(
            self.key,
            self.token,
            nx=True,      # Only if key not exists
            ex=self.ttl   # Expiry time
        )

        if not acquired:
            if timeout:
                # Poll with backoff
                deadline = time.time() + timeout
                while time.time() < deadline:
                    await asyncio.sleep(0.01)
                    acquired = await self.redis.set(
                        self.key, self.token, nx=True, ex=self.ttl
                    )
                    if acquired:
                        return True
                return False
            else:
                raise LockBusyError()

        return True

    async def release(self):
        """
        Безопасно освободить lock только если это наш token
        """
        # Lua script for atomic check-and-delete
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0  -- Lock already expired or stolen
        end
        """

        result = await self.redis.eval(
            script,
            1,           # Number of keys
            self.key,    # KEYS[1]
            self.token   # ARGV[1]
        )

        return result == 1

    async def extend(self):
        """
        Продлить TTL если ещё держим lock
        """
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("EXPIRE", KEYS[1], ARGV[2])
        else
            return 0
        end
        """

        result = await self.redis.eval(
            script, 1, self.key, self.token, self.ttl
        )

        return result == 1

    async def __aenter__(self):
        await self.acquire()
        return self

    async def __aexit__(self, exc_type, exc, tb):
        await self.release()
```

**Использование:**

```python
async def process_booking(booking_id):
    lock_key = f"booking:{booking_id}"

    async with DistributedLock(redis, lock_key, ttl=30) as lock:
        # Critical section
        booking = await db.get_booking(booking_id)

        # Долгая операция может потребовать extend
        if await payment_takes_long():
            await lock.extend()

        await process_payment(booking)
        await confirm_booking(booking)

        # Auto-release на exit
```

**Обработка долгих операций:**

```python
class LockWithHeartbeat:
    """
    Автоматически продлевать TTL пока работаем
    """

    def __init__(self, redis, key, ttl=30):
        self.redis = redis
        self.key = key
        self.ttl = ttl
        self.token = str(uuid.uuid4())
        self.heartbeat_task = None

    async def acquire(self):
        acquired = await self.redis.set(
            self.key, self.token, nx=True, ex=self.ttl
        )

        if not acquired:
            raise LockBusy()

        # Запустить heartbeat (продлевать каждые ttl/2)
        self.heartbeat_task = asyncio.create_task(self._heartbeat())
        return True

    async def _heartbeat(self):
        while True:
            try:
                await asyncio.sleep(self.ttl / 2)
                await self.extend()
            except Exception as e:
                logger.error(f"Heartbeat failed: {e}")
                # Если heartbeat failed, лучше bail out
                raise

    async def extend(self):
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("EXPIRE", KEYS[1], ARGV[2])
        else
            return 0
        end
        """

        result = await self.redis.eval(
            script, 1, self.key, self.token, self.ttl
        )

        if result == 0:
            raise LockStolen()

        return True

    async def release(self):
        if self.heartbeat_task:
            self.heartbeat_task.cancel()

        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """

        await self.redis.eval(
            script, 1, self.key, self.token
        )
```

**Распределённый lock с множественными ключами:**

```python
async def acquire_multi_lock(redis, keys, ttl=30):
    """
    Заблокировать несколько ресурсов атомарно
    """
    token = str(uuid.uuid4())

    # Сортировать для избежания deadlock (consistent order)
    sorted_keys = sorted(keys)

    pipe = redis.pipeline()

    for key in sorted_keys:
        pipe.set(f"{key}:lock", token, nx=True, ex=ttl)

    results = await pipe.execute()

    # Все или ничего
    if not all(results):
        # Rollback: удалить уже acquired locks
        for i, acquired in enumerate(results):
            if acquired:
                await release_lock(redis, sorted_keys[i])

        raise LockPartialError()

    return token

async def release_multi_lock(redis, keys, token):
    """
    Освободить множество locks
    """
    pipe = redis.pipeline()

    for key in sorted(keys):
        script = """
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
        """
        pipe.eval(script, 1, f"{key}:lock", token)

    results = await pipe.execute()

    return all(results)
```

**Fallback на DB lock если Redis down:**

```python
class ResilientLock:
    """
    Использовать Redis если доступен, иначе fallback на DB
    """

    async def acquire(self, resource_id, ttl=30):
        try:
            # Попробовать Redis
            lock = DistributedLock(redis, f"lock:{resource_id}", ttl=ttl)
            await lock.acquire(timeout=1)
            self.is_redis_lock = True
            self.lock = lock
            return
        except (redis.ConnectionError, asyncio.TimeoutError):
            logger.warning("Redis unavailable, using DB lock")

        # Fallback на DB
        async with db.transaction():
            # SELECT FOR UPDATE эмулирует распределённый lock
            result = await db.execute("""
                INSERT INTO distributed_locks (resource_id, owner_id, expires_at)
                VALUES ($1, $2, NOW() + INTERVAL '1 second' * $3)
                ON CONFLICT (resource_id) DO NOTHING
            """, resource_id, self.owner_id, ttl)

            if result.rowcount == 0:
                raise LockBusy()

            self.is_redis_lock = False
            self.resource_id = resource_id

    async def release(self):
        if self.is_redis_lock:
            await self.lock.release()
        else:
            await db.execute("""
                DELETE FROM distributed_locks
                WHERE resource_id = $1 AND owner_id = $2
            """, self.resource_id, self.owner_id)
```

**Типичные ошибки.**

1. Забыть token → могу удалить чужую блокировку
2. TTL слишком короткий → lock expires раньше завершения операции
3. Нет heartbeat для долгих операций
4. Забыть Lua script для atomic release
5. Deadlock при multiple locks → не сортировать ключи

**На интервью.**

Вопрос: "Что если Redis упадёт?"
→ Fallback на DB lock или Zookeeper

Вопрос: "Как избежать deadlock при multiple locks?"
→ Сортировать ключи в consistent order перед acquire

Вопрос: "Как обработать long-running operation?"
→ Heartbeat / extend TTL, или split на smaller chunks

---

### 6. Как предотвратить overbooking и управлять резервированиями?

**Зачем спрашивают.** Реальная проблема авиалиний и отелей. Проверяет понимание business logic, компромиссов между доходом и customer satisfaction, технических решений.

**Короткий ответ.** Overbooking допускается намеренно (% на no-shows). Резервирование: hold за счёт Redis (5 минут TTL) или DB флага. Prevent actual overbooking: constraints в БД, pessimistic lock на create_booking.

**Детальный разбор.**

**Overbooking Strategy:**

Авиалинии и отели пересчитывают на то, что часть клиентов не приедут.

```
Нормальный сценарий (без overbooking):
- 100 мест в самолёте
- Продавать только 100 билетов
- Если 5 не придут → 5 пустых мест → потеря $1000

С overbooking (10%):
- 100 мест в самолёте
- Продавать 110 билетов
- Ожидание: 5 no-shows → 105 приехали, 5 bumped
- Дать каждому bumped $400 voucher
- Net: +$5000 (110*50) - 5*400 = $2000 дополнительного дохода
```

**Расчёт overbooking лимита:**

```python
class OverbookingCalculator:
    """
    Основан на историческом no-show rate
    """

    # Historical data
    NO_SHOW_RATES = {
        'economy': 0.15,      # 15% economy passengers don't show up
        'business': 0.08,
        'first_class': 0.05
    }

    def calculate_overbooking_limit(self, cabin: str, base_capacity: int) -> int:
        """
        Сколько extra бронирований принять
        """

        # Base rate
        rate = self.NO_SHOW_RATES.get(cabin, 0.10)

        # Adjustments
        adjustments = {
            'friday_evening': 1.2,  # Business travelers show up more
            'holiday_period': 0.8,  # Families more likely to cancel
            'long_haul': 1.1,       # Higher show-up rate
            'short_haul': 1.3,      # Higher no-show rate
        }

        adjusted_rate = rate * adjustments.get(self.get_segment(), 1.0)

        # Expected no-shows
        expected_no_shows = int(base_capacity * adjusted_rate)

        # Add safety margin (don't oversell too much)
        max_allowed_overbooking = int(base_capacity * 0.10)  # Never > 10%

        return min(expected_no_shows, max_allowed_overbooking)

    def get_effective_capacity(self, flight_id: str) -> int:
        """
        Сколько билетов можно продать
        """
        flight = db.get_flight(flight_id)

        for cabin in ['first_class', 'business', 'economy']:
            base = flight.capacity[cabin]
            overbook = self.calculate_overbooking_limit(cabin, base)

            effective = base + overbook

            logger.info(
                f"Flight {flight_id} cabin {cabin}: "
                f"{base} + {overbook} = {effective} available"
            )

        return effective
```

**Управление резервированиями (Reservations):**

Сценарий: пользователь выбрал место, но ещё не оплатил.

```
Временная шкала при бронировании:
T0:  SELECT room_inventory WHERE date >= checkin AND date < checkout FOR UPDATE
T1:  Check availability
T2:  CREATE reservation (temporary hold, TTL = 5 min)
T3:  SHOW payment form
T10: User completes payment
T11: UPDATE reservation SET status='confirmed'
T12: COMMIT (release lock)

Что если user не платит?
T5:  (5 minutes pass)
T305: Background job: DELETE reservation WHERE expires_at < NOW()
T305: UPDATE room_inventory SET reserved = reserved - 1 (release hold)
```

**Реализация резервирования:**

```python
class ReservationService:
    RESERVATION_TTL = 300  # 5 minutes

    async def create_reservation(
        self,
        room_type_id: str,
        checkin: date,
        checkout: date,
        user_id: str
    ):
        """
        Hold room for 5 minutes (pessimistic lock)
        """

        async with db.transaction(isolation='SERIALIZABLE'):
            # 1. Acquire lock на date range
            inventory_rows = await db.fetch("""
                SELECT id, date, total_rooms, booked_rooms, blocked_rooms, reserved_rooms
                FROM room_inventory
                WHERE room_type_id = $1
                  AND date >= $2 AND date < $3
                FOR UPDATE
            """, room_type_id, checkin, checkout)

            # 2. Check availability
            for row in inventory_rows:
                available = (
                    row['total_rooms'] -
                    row['booked_rooms'] -
                    row['blocked_rooms'] -
                    row['reserved_rooms']
                )

                if available < 1:
                    raise NoAvailabilityError(row['date'])

            # 3. Reserve (increment reserved_rooms)
            await db.execute("""
                UPDATE room_inventory
                SET reserved_rooms = reserved_rooms + 1
                WHERE room_type_id = $1
                  AND date >= $2 AND date < $3
            """, room_type_id, checkin, checkout)

            # 4. Create reservation record
            reservation = await db.execute("""
                INSERT INTO reservations (
                    user_id, room_type_id, checkin_date, checkout_date,
                    status, expires_at
                )
                VALUES ($1, $2, $3, $4, 'pending', NOW() + INTERVAL '5 minutes')
                RETURNING *
            """, user_id, room_type_id, checkin, checkout)

            # 5. Schedule expiry
            await queue.schedule(
                'release_reservation',
                reservation['id'],
                delay=self.RESERVATION_TTL
            )

            return reservation

    async def confirm_reservation(self, reservation_id: str, payment_id: str):
        """
        Confirm reservation → convert to booking
        """

        async with db.transaction():
            # 1. Get reservation with lock
            reservation = await db.execute("""
                SELECT * FROM reservations
                WHERE id = $1 AND status = 'pending'
                FOR UPDATE
            """, reservation_id)

            if not reservation:
                raise ReservationExpiredError()

            # 2. Update inventory: reserved → booked
            await db.execute("""
                UPDATE room_inventory
                SET reserved_rooms = reserved_rooms - 1,
                    booked_rooms = booked_rooms + 1
                WHERE room_type_id = $1
                  AND date >= $2 AND date < $3
            """, reservation['room_type_id'],
                reservation['checkin_date'],
                reservation['checkout_date'])

            # 3. Create booking
            booking = await db.execute("""
                INSERT INTO bookings (
                    reservation_id, user_id, room_type_id,
                    checkin_date, checkout_date, payment_id, status
                )
                VALUES ($1, $2, $3, $4, $5, $6, 'confirmed')
                RETURNING *
            """, reservation_id, reservation['user_id'],
                reservation['room_type_id'],
                reservation['checkin_date'],
                reservation['checkout_date'],
                payment_id)

            # 4. Cancel expiry task
            await queue.cancel('release_reservation', reservation_id)

            return booking

    async def release_reservation(self, reservation_id: str):
        """
        Release hold when TTL expires or user cancels
        """

        async with db.transaction():
            reservation = await db.get_reservation(reservation_id)

            if reservation['status'] != 'pending':
                return  # Already confirmed or released

            # Release reserved_rooms
            await db.execute("""
                UPDATE room_inventory
                SET reserved_rooms = reserved_rooms - 1
                WHERE room_type_id = $1
                  AND date >= $2 AND date < $3
            """, reservation['room_type_id'],
                reservation['checkin_date'],
                reservation['checkout_date'])

            # Mark as released
            await db.execute("""
                UPDATE reservations
                SET status = 'released', released_at = NOW()
                WHERE id = $1
            """, reservation_id)
```

**Capacity states диаграмма:**

```
┌─────────────────────────────────────┐
│  Total Capacity = 100 rooms         │
├─────────────────────────────────────┤
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ Booked: 50                      │ │  (confirmed bookings)
│ │ [#############]                 │ │
│ ├─────────────────────────────────┤ │
│ │ Reserved: 10                    │ │  (pending, TTL 5 min)
│ │ [##]                            │ │
│ ├─────────────────────────────────┤ │
│ │ Blocked: 5                      │ │  (maintenance)
│ │ [#]                             │ │
│ ├─────────────────────────────────┤ │
│ │ Available: 35                   │ │  (can book now)
│ │ [ ]                             │ │
│ ├─────────────────────────────────┤ │
│ │ Overbooking Buffer: 10          │ │  (expected no-shows)
│ │ [#]                             │ │
│ └─────────────────────────────────┘ │
│                                     │
└─────────────────────────────────────┘

Formula:
Available = Total - Booked - Reserved - Blocked

Can sell until = Total + Overbooking - Booked - Reserved - Blocked
                = 100 + 10 - 50 - 10 - 5
                = 45
```

**Типичные ошибки.**

1. Забыть про reserved rooms при проверке availability
2. Not release reservation на expiry → phantom inventory
3. Race condition: check availability и reserve в разных транзакциях
4. TTL слишком длинный (30 минут) → блокирует много мест
5. TTL слишком короткий (30 секунд) → frustrated users при slow payment

**На интервью.**

Вопрос: "Как избежать overbooking в реальности?"
→ Constraints в БД: CHECK (booked + reserved + blocked <= total + overbook_limit)

Вопрос: "Что если customer платит медленно?"
→ Extend reservation TTL если близко к expiry, или make payment async

Вопрос: "Как обрабатывать no-shows при check-in?"
→ Mark as no-show, release to available inventory

---

### 7. Как реализовать выбор мест/кресел в динамичной системе?

**Зачем спрашивают.** Специфичный, но важный case для airline/cinema. Проверяет понимание real-time sync, WebSocket, conflict resolution, кеширования.

**Короткий ответ.** Сиденье может быть в одном из 5 состояний: available, reserved, booked, blocked, maintenance. Использовать WebSocket для real-time updates, Redis для кеша состояния, DB для persistence. При выборе мест: резервировать через optimistic lock или Redis.

**Детальный разбор.**

**Модель данных для мест:**

```sql
CREATE TABLE seats (
    id UUID PRIMARY KEY,
    flight_id UUID REFERENCES flights(id),
    seat_number VARCHAR(10),  -- 1A, 2B, etc
    seat_type VARCHAR(20),    -- window, middle, aisle
    price INT,

    UNIQUE(flight_id, seat_number)
);

-- Состояние мест как материализованное представление
CREATE TABLE seat_state (
    id UUID PRIMARY KEY,
    flight_id UUID REFERENCES flights(id),
    seat_id UUID REFERENCES seats(id),

    status VARCHAR(20),  -- available, reserved, booked, blocked
    reserved_by UUID,    -- user_id who reserved
    reserved_until TIMESTAMP,
    booked_by UUID,      -- booking_id

    version INT DEFAULT 1,
    updated_at TIMESTAMP,

    UNIQUE(flight_id, seat_id),
    INDEX idx_flight_status (flight_id, status)
);
```

**Кеш состояния в Redis:**

```python
class SeatCacheManager:
    """
    Redis как single source of truth для real-time состояния
    DB для persistence
    """

    async def get_seat_state(self, flight_id: str, seat_id: str):
        """
        Получить состояние из Redis
        """
        key = f"seat:{flight_id}:{seat_id}"

        cached = await redis.hgetall(key)
        if cached:
            return {
                'status': cached['status'],
                'reserved_by': cached.get('reserved_by'),
                'version': int(cached['version'])
            }

        # Cache miss → get from DB
        seat_state = await db.get_seat_state(flight_id, seat_id)

        # Cache for 1 hour
        await redis.hset(key, mapping={
            'status': seat_state['status'],
            'reserved_by': seat_state['reserved_by'] or '',
            'version': seat_state['version']
        }, ex=3600)

        return seat_state

    async def get_all_seats_for_flight(self, flight_id: str):
        """
        Получить все мест для рейса в один запрос
        """

        # Pattern: seat:flight_id:*
        pattern = f"seat:{flight_id}:*"

        # Scan + get all
        seats = {}
        async for key in redis.scan_iter(pattern):
            seat_data = await redis.hgetall(key)
            seats[key] = seat_data

        return seats

    async def reserve_seat(
        self,
        flight_id: str,
        seat_id: str,
        user_id: str,
        ttl: int = 300
    ):
        """
        Попытка зарезервировать место
        """
        key = f"seat:{flight_id}:{seat_id}"

        # Check-and-set using Lua
        script = """
        local current = redis.call('HGET', KEYS[1], 'status')

        if current == 'available' or current == false then
            redis.call('HSET', KEYS[1], 'status', 'reserved')
            redis.call('HSET', KEYS[1], 'reserved_by', ARGV[1])
            redis.call('HSET', KEYS[1], 'reserved_until', ARGV[2])
            redis.call('EXPIRE', KEYS[1], ARGV[3])
            return 'OK'
        else
            return 'UNAVAILABLE'
        end
        """

        result = await redis.eval(
            script,
            1,
            key,
            user_id,
            time.time() + ttl,
            ttl
        )

        if result != 'OK':
            raise SeatUnavailableError()

        # Sync to DB asynchronously
        asyncio.create_task(self._sync_to_db(flight_id, seat_id, 'reserved', user_id))

    async def release_seat(self, flight_id: str, seat_id: str):
        """
        Освободить место
        """
        key = f"seat:{flight_id}:{seat_id}"

        await redis.hset(key, 'status', 'available')
        await redis.hdel(key, 'reserved_by', 'reserved_until')

        await self._sync_to_db(flight_id, seat_id, 'available', None)

    async def confirm_seat(
        self,
        flight_id: str,
        seat_id: str,
        booking_id: str
    ):
        """
        Convert reserved → booked
        """
        key = f"seat:{flight_id}:{seat_id}"

        await redis.hset(key, mapping={
            'status': 'booked',
            'booked_by': booking_id
        })

        await redis.hdel(key, 'reserved_by', 'reserved_until')

        await self._sync_to_db(flight_id, seat_id, 'booked', booking_id)

    async def _sync_to_db(self, flight_id, seat_id, status, owned_by):
        """
        Async sync to DB (doesn't block client)
        """
        try:
            await db.execute("""
                UPDATE seat_state
                SET status = $1, reserved_by = $2, updated_at = NOW()
                WHERE flight_id = $3 AND seat_id = $4
            """, status, owned_by, flight_id, seat_id)
        except Exception as e:
            logger.error(f"DB sync failed: {e}")
            # Ideally: retry with backoff
```

**WebSocket для real-time updates:**

```python
class SeatUpdateBroadcaster:
    """
    Broadcast seat state changes к всем подключённым клиентам
    """

    def __init__(self):
        self.connections: dict[str, set] = {}  # flight_id → set of websockets

    async def subscribe(self, flight_id: str, websocket):
        """
        User подключается к real-time обновлениям для рейса
        """
        if flight_id not in self.connections:
            self.connections[flight_id] = set()

        self.connections[flight_id].add(websocket)

        # Send initial state
        seats = await self.seat_cache.get_all_seats_for_flight(flight_id)
        await websocket.send_json({
            'type': 'INITIAL_STATE',
            'seats': seats
        })

    async def broadcast_seat_update(self, flight_id: str, seat_id: str, new_state: dict):
        """
        Send update к всем subscribers для рейса
        """

        if flight_id not in self.connections:
            return

        message = {
            'type': 'SEAT_UPDATE',
            'flight_id': flight_id,
            'seat_id': seat_id,
            'status': new_state['status'],
            'reserved_by': new_state.get('reserved_by')
        }

        # Send to all connected clients
        disconnected = set()

        for websocket in self.connections[flight_id]:
            try:
                await websocket.send_json(message)
            except Exception as e:
                logger.error(f"Failed to send: {e}")
                disconnected.add(websocket)

        # Remove disconnected
        self.connections[flight_id] -= disconnected

    async def on_seat_reserved(self, flight_id: str, seat_id: str, user_id: str):
        """
        Event handler when seat is reserved
        """
        await self.broadcast_seat_update(flight_id, seat_id, {
            'status': 'reserved',
            'reserved_by': user_id
        })

    async def on_seat_released(self, flight_id: str, seat_id: str):
        await self.broadcast_seat_update(flight_id, seat_id, {
            'status': 'available'
        })
```

**Frontend: Real-time seat map**

```javascript
// Simplified example
class SeatMap {
    constructor(flightId) {
        this.flightId = flightId;
        this.seats = new Map();
        this.selectedSeats = new Set();
        this.websocket = null;
        this.init();
    }

    async init() {
        // Connect to WebSocket
        this.websocket = new WebSocket(
            `wss://api.airline.com/flights/${this.flightId}/seats`
        );

        this.websocket.onmessage = (event) => {
            const message = JSON.parse(event.data);

            if (message.type === 'INITIAL_STATE') {
                // Load initial seat states
                for (const [seatId, state] of Object.entries(message.seats)) {
                    this.seats.set(seatId, state.status);
                }
                this.render();
            }

            if (message.type === 'SEAT_UPDATE') {
                // Update single seat
                this.seats.set(message.seat_id, message.status);
                this.renderSeat(message.seat_id);
            }
        };
    }

    async selectSeat(seatId) {
        if (this.seats.get(seatId) !== 'available') {
            alert('Seat unavailable!');
            return;
        }

        try {
            // Optimistic update
            this.selectedSeats.add(seatId);
            this.renderSeat(seatId);

            // Server-side reserve
            const response = await fetch(
                `/api/flights/${this.flightId}/seats/${seatId}/reserve`,
                { method: 'POST' }
            );

            if (!response.ok) {
                // Revert optimistic update
                this.selectedSeats.delete(seatId);
                this.renderSeat(seatId);
                throw new Error('Reserve failed');
            }
        } catch (e) {
            alert('Failed to reserve seat');
        }
    }

    renderSeat(seatId) {
        const element = document.getElementById(seatId);
        const status = this.seats.get(seatId);
        const isSelected = this.selectedSeats.has(seatId);

        element.className = `seat seat-${status}`;
        if (isSelected) {
            element.classList.add('selected');
        }
    }

    render() {
        // Initial render all seats
        for (const [seatId] of this.seats) {
            this.renderSeat(seatId);
        }
    }
}
```

**Типичные ошибки.**

1. Cache-aside без invalidation → stale seat state
2. WebSocket broadcast без deduplication → duplicate updates
3. Select seat и reserve в разных запросах → race condition
4. Не handle WebSocket disconnect → missed updates
5. Load initial state асинхронно → client books unavailable seat

**На интервью.**

Вопрос: "Что если two users выберут одно место одновременно?"
→ Redis Lua script для atomic check-and-set

Вопрос: "Как обновить 1000 мест когда рейс отменяется?"
→ Batch update в Redis, cascade delete через queue

Вопрос: "Масштабирование для 10M мест?"
→ Partition по flight_id, ненужное broadcast только подписчикам

---

### 8. Как реализовать waitlist для sold-out бронирования?

**Зачем спрашивают.** Комплекс систем: очередь, нотификации, автоматическое обновление, handling cancellations. Проверяет умение работать с полнофункциональным feature.

**Короткий ответ.** Waitlist как FIFO очередь в БД. Когда место освобождается (cancellation), автоматически offer следующему в очереди (5 мин timeout). Нотификация через email/SMS. Если не ответит, перейти к следующему.

**Детальный разбор.**

**Схема БД для waitlist:**

```sql
CREATE TABLE waitlist (
    id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(id),
    room_type_id UUID REFERENCES room_types(id),
    checkin_date DATE,
    checkout_date DATE,
    user_id UUID REFERENCES users(id),

    position INT NOT NULL,  -- 1, 2, 3... (order in queue)
    status VARCHAR(20),     -- pending, offered, accepted, expired, removed

    created_at TIMESTAMP DEFAULT NOW(),
    offered_at TIMESTAMP,
    offer_expires_at TIMESTAMP,
    accepted_at TIMESTAMP,

    INDEX idx_room_date_pos (room_type_id, checkin_date, checkout_date, position)
);

-- Table to track offers sent
CREATE TABLE waitlist_offers (
    id UUID PRIMARY KEY,
    waitlist_id UUID REFERENCES waitlist(id),
    offer_price INT,
    offered_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    status VARCHAR(20),  -- pending, accepted, expired, declined
    accepted_at TIMESTAMP
);
```

**Добавление в waitlist:**

```python
class WaitlistService:

    async def add_to_waitlist(
        self,
        user_id: str,
        room_type_id: str,
        checkin: date,
        checkout: date
    ):
        """
        Добавить user в waitlist
        """

        async with db.transaction():
            # 1. Get next position
            max_position = await db.query_scalar("""
                SELECT MAX(position) FROM waitlist
                WHERE room_type_id = $1
                  AND checkin_date = $2
                  AND checkout_date = $3
                  AND status IN ('pending', 'offered')
            """, room_type_id, checkin, checkout)

            next_position = (max_position or 0) + 1

            # 2. Check if user already in waitlist
            existing = await db.query_one("""
                SELECT id FROM waitlist
                WHERE user_id = $1
                  AND room_type_id = $2
                  AND checkin_date = $3
                  AND checkout_date = $4
                  AND status IN ('pending', 'offered')
            """, user_id, room_type_id, checkin, checkout)

            if existing:
                raise AlreadyInWaitlistError()

            # 3. Insert
            waitlist_entry = await db.execute("""
                INSERT INTO waitlist (
                    user_id, room_type_id, checkin_date, checkout_date, position
                )
                VALUES ($1, $2, $3, $4, $5)
                RETURNING *
            """, user_id, room_type_id, checkin, checkout, next_position)

            # 4. Send confirmation
            user = await db.get_user(user_id)
            await notification_service.send_email(
                user.email,
                subject=f"Added to waitlist (position {next_position})",
                body=f"You are #{next_position} in queue"
            )

            return waitlist_entry
```

**Обработка отмены бронирования (cancellation trigger):**

```python
class WaitlistNotificationService:

    async def on_booking_cancelled(self, booking_id: str):
        """
        Когда бронирование отменено, offer next in waitlist
        """

        booking = await db.get_booking(booking_id)

        # 1. Find all people in waitlist for this room/dates
        queue = await db.fetch("""
            SELECT * FROM waitlist
            WHERE room_type_id = $1
              AND checkin_date = $2
              AND checkout_date = $3
              AND status = 'pending'
            ORDER BY position ASC
            LIMIT 5  -- Offer to top 5 in case some decline/timeout
        """, booking.room_type_id, booking.checkin_date, booking.checkout_date)

        # 2. For each, send offer
        for entry in queue:
            await self.send_offer_to_user(entry)

    async def send_offer_to_user(self, waitlist_entry: dict):
        """
        Send time-limited offer к user в waitlist
        """

        user = await db.get_user(waitlist_entry['user_id'])

        # Calculate new price (might be different)
        price = await self._get_current_price(
            waitlist_entry['room_type_id'],
            waitlist_entry['checkin_date'],
            waitlist_entry['checkout_date']
        )

        # OFFER_TTL: 5 minutes for user to decide
        offer_expires = datetime.utcnow() + timedelta(minutes=5)

        # Create offer
        offer = await db.execute("""
            INSERT INTO waitlist_offers (
                waitlist_id, offer_price, offered_at, expires_at
            )
            VALUES ($1, $2, NOW(), $3)
            RETURNING *
        """, waitlist_entry['id'], price, offer_expires)

        # Update waitlist entry
        await db.execute("""
            UPDATE waitlist
            SET status = 'offered', offered_at = NOW(), offer_expires_at = $2
            WHERE id = $1
        """, waitlist_entry['id'], offer_expires)

        # Send email with offer link
        confirmation_token = self._create_token(offer['id'])

        accept_link = f"https://hotel.com/waitlist-offer/{confirmation_token}/accept"
        decline_link = f"https://hotel.com/waitlist-offer/{confirmation_token}/decline"

        await notification_service.send_email(
            user.email,
            subject=f"GREAT NEWS! Room available for {waitlist_entry['checkin_date']}",
            body=f"""
            A room is now available!

            Room: {waitlist_entry['room_type']}
            Check-in: {waitlist_entry['checkin_date']}
            Price: ${price / 100}

            ACCEPT (expires in 5 minutes): {accept_link}
            DECLINE: {decline_link}
            """
        )

        # Schedule expiry task
        await queue.schedule(
            'expire_waitlist_offer',
            offer['id'],
            delay=300  # 5 minutes
        )
```

**Обработка Accept/Decline:**

```python
class WaitlistOfferHandler:

    async def accept_offer(self, offer_token: str):
        """
        User accepts offer → try to book immediately
        """

        offer_id = self._validate_token(offer_token)
        offer = await db.get_offer(offer_id)

        # Check if still valid
        if offer['status'] != 'pending':
            raise OfferExpiredError()

        if offer['expires_at'] < datetime.utcnow():
            raise OfferExpiredError()

        waitlist_entry = await db.get_waitlist(offer['waitlist_id'])

        async with db.transaction():
            # 1. Verify still available (double-check)
            availability = await check_availability(
                waitlist_entry['room_type_id'],
                waitlist_entry['checkin_date'],
                waitlist_entry['checkout_date']
            )

            if not availability:
                # Someone else grabbed it
                await self.send_next_offer(waitlist_entry)
                raise StillUnavailableError()

            # 2. Create booking
            booking = await create_booking(
                user_id=waitlist_entry['user_id'],
                room_type_id=waitlist_entry['room_type_id'],
                checkin=waitlist_entry['checkin_date'],
                checkout=waitlist_entry['checkout_date'],
                price=offer['offer_price']
            )

            # 3. Mark offer as accepted
            await db.execute("""
                UPDATE waitlist_offers
                SET status = 'accepted', accepted_at = NOW()
                WHERE id = $1
            """, offer_id)

            # 4. Mark waitlist entry as accepted
            await db.execute("""
                UPDATE waitlist
                SET status = 'accepted', accepted_at = NOW()
                WHERE id = $1
            """, offer['waitlist_id'])

            # 5. Cancel other offers for same inventory
            await db.execute("""
                UPDATE waitlist_offers
                SET status = 'cancelled'
                WHERE id != $1
                  AND waitlist_id IN (
                    SELECT id FROM waitlist
                    WHERE room_type_id = $2
                      AND checkin_date = $3
                      AND checkout_date = $4
                      AND status = 'offered'
                  )
            """, offer_id, waitlist_entry['room_type_id'],
                waitlist_entry['checkin_date'],
                waitlist_entry['checkout_date'])

            # 6. Send confirmation
            user = await db.get_user(waitlist_entry['user_id'])
            await notification_service.send_confirmation(user.email, booking)

            return booking

    async def decline_offer(self, offer_token: str):
        """
        User declines offer → send to next in waitlist
        """

        offer_id = self._validate_token(offer_token)
        offer = await db.get_offer(offer_id)
        waitlist_entry = await db.get_waitlist(offer['waitlist_id'])

        async with db.transaction():
            # Mark as declined
            await db.execute("""
                UPDATE waitlist_offers
                SET status = 'declined'
                WHERE id = $1
            """, offer_id)

            # Revert waitlist entry to pending
            await db.execute("""
                UPDATE waitlist
                SET status = 'pending', offered_at = NULL
                WHERE id = $1
            """, offer['waitlist_id'])

            # Send offer to next person
            await self.send_next_offer(waitlist_entry)

    async def send_next_offer(self, current_entry: dict):
        """
        Find next person in waitlist and send offer
        """

        next_entry = await db.query_one("""
            SELECT * FROM waitlist
            WHERE room_type_id = $1
              AND checkin_date = $2
              AND checkout_date = $3
              AND status = 'pending'
              AND position > (SELECT position FROM waitlist WHERE id = $4)
            ORDER BY position ASC
            LIMIT 1
        """, current_entry['room_type_id'],
            current_entry['checkin_date'],
            current_entry['checkout_date'],
            current_entry['id'])

        if not next_entry:
            return  # No more people in waitlist

        await self.send_offer_to_user(next_entry)
```

**Background job для cleanup:**

```python
class WaitlistMaintenanceJob:

    async def cleanup_expired_offers(self):
        """
        Ежеминутно: удалить expired offers, отправить следующим
        """

        expired_offers = await db.fetch("""
            SELECT * FROM waitlist_offers
            WHERE status = 'pending'
              AND expires_at < NOW()
        """)

        for offer in expired_offers:
            waitlist_entry = await db.get_waitlist(offer['waitlist_id'])

            # Mark as expired
            await db.execute("""
                UPDATE waitlist_offers SET status = 'expired' WHERE id = $1
            """, offer['id'])

            # Revert to pending
            await db.execute("""
                UPDATE waitlist SET status = 'pending'
                WHERE id = $1
            """, offer['waitlist_id'])

            # Send to next
            await send_next_offer(waitlist_entry)

    async def remove_old_expired_entries(self):
        """
        Удалить entries после 30 дней (не нужны в памяти)
        """

        await db.execute("""
            DELETE FROM waitlist
            WHERE status IN ('expired', 'removed')
              AND created_at < NOW() - INTERVAL '30 days'
        """)
```

**Типичные ошибки.**

1. Забыть invalidate offer при accept другого user'а
2. Position не обновляется при удалении в середине очереди
3. Race condition: two offers sent одновременно на same room
4. Не cancel old offers если inventory becomes available through other channel
5. TTL на offer слишком длинный (30 min) → user не в спешке ответить

**На интервью.**

Вопрос: "Как обработать peak когда много offers одновременно?"
→ Queue with rate limiting, или randomize slight delays

Вопрос: "Как давать приоритет VIP users?"
→ Separate waitlist с более высоким position, shorter TTL

Вопрос: "Масштабирование для миллионов в waitlist?"
→ Shard по room_type_id + date, lazy load positions

---

### 9. Как интегрировать календарь и отправлять уведомления?

**Зачем спрашивают.** Integration с external systems, notification patterns, scheduling. Проверяет практичность дизайна.

**Короткий ответ.** Calendar как view на inventory table (группировать по дням). Уведомления: event-driven (Kafka), асинхронные через queue. Pre-send напоминания за N дней до check-in, отправлять при state changes (confirmation, cancellation).

**Детальный разбор.**

**Calendar data model:**

```sql
CREATE TABLE calendar_view (
    id UUID PRIMARY KEY,
    hotel_id UUID REFERENCES hotels(id),
    room_type_id UUID REFERENCES room_types(id),
    date DATE,

    total_capacity INT,
    booked_count INT,
    reserved_count INT,
    available_count INT,  -- computed

    min_price INT,
    max_price INT,
    avg_price INT,

    occupancy_rate DECIMAL(3, 2),  -- booked / total

    INDEX idx_hotel_dates (hotel_id, date)
);
```

**Populate calendar view (materialized view or periodic refresh):**

```python
async def refresh_calendar_view(hotel_id: str, room_type_id: str, date_from: date, date_to: date):
    """
    Refresh calendar for date range
    """

    # Aggregate from room_inventory
    rows = await db.fetch("""
        SELECT
            ri.date,
            SUM(ri.total_rooms) as total_capacity,
            SUM(ri.booked_rooms) as booked_count,
            SUM(ri.reserved_rooms) as reserved_count,
            MIN(ri.dynamic_price OR ri.base_price) as min_price,
            MAX(ri.dynamic_price OR ri.base_price) as max_price,
            AVG(ri.dynamic_price OR ri.base_price) as avg_price
        FROM room_inventory ri
        WHERE ri.hotel_id = $1
          AND ri.room_type_id = $2
          AND ri.date >= $3
          AND ri.date < $4
        GROUP BY ri.date
    """, hotel_id, room_type_id, date_from, date_to)

    # Upsert into calendar_view
    for row in rows:
        available = row['total_capacity'] - row['booked_count'] - row['reserved_count']
        occupancy = row['booked_count'] / row['total_capacity'] if row['total_capacity'] > 0 else 0

        await db.execute("""
            INSERT INTO calendar_view (
                hotel_id, room_type_id, date,
                total_capacity, booked_count, reserved_count,
                available_count, min_price, max_price, avg_price,
                occupancy_rate
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            ON CONFLICT (hotel_id, room_type_id, date) DO UPDATE SET
                booked_count = $5,
                reserved_count = $6,
                available_count = $7,
                min_price = $8,
                max_price = $9,
                avg_price = $10,
                occupancy_rate = $11
        """, hotel_id, room_type_id, date_from, ... occupancy)
```

**Event-driven notification система:**

```python
class NotificationService:
    """
    Асинхронная отправка уведомлений через Kafka
    """

    async def send_booking_confirmation(self, booking_id: str):
        """
        Publish event на Kafka
        """

        booking = await db.get_booking(booking_id)

        event = {
            'event_type': 'BOOKING_CONFIRMED',
            'booking_id': booking_id,
            'user_id': booking.user_id,
            'email': booking.guest_email,
            'phone': booking.guest_phone,
            'hotel_name': booking.hotel_name,
            'checkin_date': booking.checkin_date.isoformat(),
            'confirmation_code': booking.confirmation_code,
            'total_amount': booking.total_amount,
            'timestamp': datetime.utcnow().isoformat()
        }

        # Publish to Kafka
        await kafka_producer.send('booking-notifications', event)

    async def send_cancellation_notification(self, booking_id: str):
        """
        Notify on cancellation
        """

        booking = await db.get_booking(booking_id)

        event = {
            'event_type': 'BOOKING_CANCELLED',
            'booking_id': booking_id,
            'user_id': booking.user_id,
            'email': booking.guest_email,
            'refund_amount': booking.refund_amount,
            'timestamp': datetime.utcnow().isoformat()
        }

        await kafka_producer.send('booking-notifications', event)


class NotificationConsumer:
    """
    Kafka consumer для обработки events
    """

    async def process_notification_event(self, event: dict):
        """
        Consume event, отправить фактическое письмо/SMS
        """

        event_type = event['event_type']

        if event_type == 'BOOKING_CONFIRMED':
            await self._send_confirmation_email(event)
            await self._send_confirmation_sms(event)
            await self._schedule_reminder_notifications(event)

        elif event_type == 'BOOKING_CANCELLED':
            await self._send_cancellation_email(event)

        elif event_type == 'REMINDER_CHECKIN':
            await self._send_checkin_reminder(event)

    async def _send_confirmation_email(self, event: dict):
        """
        Отправить email confirmation через SES/Sendgrid
        """

        template = """
        <h1>Booking Confirmed!</h1>
        <p>Confirmation Code: {confirmation_code}</p>
        <p>Hotel: {hotel_name}</p>
        <p>Check-in: {checkin_date}</p>
        <p>Amount Paid: ${amount}</p>
        """

        html = template.format(**event)

        await email_service.send(
            to=event['email'],
            subject='Booking Confirmation',
            html=html
        )

    async def _send_confirmation_sms(self, event: dict):
        """
        Отправить SMS
        """

        message = f"""
        Booking confirmed!
        Confirmation: {event['confirmation_code']}
        Hotel: {event['hotel_name']}
        Check-in: {event['checkin_date']}
        """

        await sms_service.send(event['phone'], message)

    async def _schedule_reminder_notifications(self, event: dict):
        """
        Schedule reminder emails/SMS для выполнения позже
        """

        checkin_date = datetime.fromisoformat(event['checkin_date'])

        # Send reminders at:
        # - 7 days before check-in
        # - 2 days before check-in
        # - 1 day before check-in
        # - 2 hours before check-in (if we know time)

        reminders = [
            (checkin_date - timedelta(days=7), '7 days until check-in'),
            (checkin_date - timedelta(days=2), '2 days until check-in'),
            (checkin_date - timedelta(days=1), 'Tomorrow is check-in'),
        ]

        for reminder_time, message_type in reminders:
            await task_queue.schedule(
                'send_reminder_email',
                {
                    'booking_id': event['booking_id'],
                    'email': event['email'],
                    'message_type': message_type
                },
                delay=(reminder_time - datetime.utcnow()).total_seconds()
            )
```

**Scheduled job для периодических напоминаний:**

```python
class ReminderScheduler:
    """
    Cron job: each hour, check если need to send reminders
    """

    async def check_and_send_reminders(self):
        """
        Run every hour
        """

        now = datetime.utcnow()

        # Find bookings that need reminder
        bookings_needing_7day_reminder = await db.fetch("""
            SELECT * FROM bookings
            WHERE status = 'confirmed'
              AND checkin_date = DATE($1) + INTERVAL '7 days'
              AND reminder_7day_sent = false
        """, now)

        for booking in bookings_needing_7day_reminder:
            await self.send_reminder_email(
                booking,
                'Your booking in 7 days'
            )

            await db.execute("""
                UPDATE bookings SET reminder_7day_sent = true WHERE id = $1
            """, booking['id'])

        # Similar for 2-day, 1-day reminders
        ...

    async def send_reminder_email(self, booking: dict, subject: str):

        template = """
        <h2>{subject}</h2>
        <p>Check-in date: {checkin_date}</p>
        <p>Hotel: {hotel_name}</p>
        <p>Confirmation: {confirmation_code}</p>
        """

        await email_service.send(
            to=booking['guest_email'],
            subject=subject,
            html=template.format(**booking)
        )
```

**SMS gateway integration:**

```python
class SMSService:
    """
    Integration с Twilio/AWS SNS
    """

    async def send(self, phone_number: str, message: str):
        """
        Send SMS
        """

        try:
            response = await twilio_client.messages.create(
                body=message,
                from_=TWILIO_PHONE_NUMBER,
                to=phone_number
            )

            # Log for debugging
            await db.execute("""
                INSERT INTO notification_logs (
                    user_id, channel, message, external_id, status
                )
                VALUES ($1, $2, $3, $4, 'sent')
            """, phone_number, 'sms', message, response.sid)

            return response.sid

        except Exception as e:
            logger.error(f"SMS send failed: {e}")

            # Log failure
            await db.execute("""
                INSERT INTO notification_logs (
                    user_id, channel, message, status, error
                )
                VALUES ($1, $2, $3, 'failed', $4)
            """, phone_number, 'sms', message, str(e))

            # Retry through queue
            await task_queue.schedule(
                'send_sms_retry',
                {'phone': phone_number, 'message': message},
                delay=300  # Retry in 5 minutes
            )
```

**Типичные ошибки.**

1. Sending notification в same transaction как booking → fails atomically
2. Notification без idempotency key → duplicate sends
3. Email template hardcoded → трудно изменить
4. No retry logic для failed sends
5. Storing sensitive данные в notification logs

**На интервью.**

Вопрос: "Что если email service down?"
→ Queue all notifications, retry with exponential backoff

Вопрос: "Как обработать unsubscribe?"
→ Check user preferences перед sending

Вопрос: "Как отправлять на разные timezones?"
→ Store user timezone, schedule в их local midnight

---

### 10. Как масштабировать систему при 100K одновременных бронирований?

**Зачем спрашивают.** Финальный, comprehensive вопрос. Проверяет ability собрать все pieces вместе, identify bottlenecks, трейд-офф.

**Короткий ответ.** На 100K/sec: sharding по hotel_id, read replicas для search, Redis cluster для locks, async job queue для нотификаций, API rate limiting. Database переходит в bottleneck → нужны batch updates, eventual consistency где возможно, кеширование.

**Детальный разбор.**

**Архитектура при масштабировании:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Layer                              │
│              (Web, Mobile, Partner APIs)                     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Load Balancer                               │
│              (Round-robin, least-conn)                       │
└──┬──────────────────────────────────────┬─────────────────┬──┘
   │                                      │                 │
   ▼                                      ▼                 ▼
┌────────────────┐              ┌────────────────┐   ┌──────────┐
│ API Gateway #1 │              │ API Gateway #2 │...│ Gateway N│
│ + Rate Limit   │              │ + Rate Limit   │   │          │
│ (1K req/sec)   │              │ (1K req/sec)   │   │          │
└────────┬───────┘              └────────┬───────┘   └────┬─────┘
         │                               │                │
         └───────────────┬───────────────┴────────────────┘
                         │
         ┌───────────────┴────────────────┐
         │                                │
    ┌────▼──────┐                    ┌────▼──────┐
    │ Search    │                    │ Booking   │
    │ Service   │                    │ Service   │
    │ (Read-RI) │                    │ (Txn)     │
    └────┬──────┘                    └────┬──────┘
         │                                │
    ┌────▼──────────────┐            ┌────▼──────────────┐
    │  Elasticsearch    │            │  PostgreSQL       │
    │  Cluster          │            │  Cluster          │
    │  (Replicas)       │            │  (Primary + Rep)  │
    └───────────────────┘            └────┬──────────────┘
                                           │
                                  ┌────────▼────────┐
                                  │  Redis Cluster  │
                                  │  (Locks)        │
                                  └─────────────────┘

Task Queue:
┌──────────────────────────────────────┐
│    RabbitMQ / Kafka                  │
│  (Notifications, Reminders, Reports) │
└──────────────────────────────────────┘
```

**Bottleneck Analysis при 100K req/sec:**

```
┌─────────────┬────────┬────────────┐
│ Component   │ Type   │ Bottleneck │
├─────────────┼────────┼────────────┤
│ API Gateway │ I/O    │ ~100K req  │
│ Search      │ CPU    │ ElasticS   │
│ Booking DB  │ I/O    │ Write IOPS │
│ Redis Locks │ I/O    │ Throughput │
│ Memory      │ Memory │ Cache hit  │
└─────────────┴────────┴────────────┘

Typical bottleneck at 100K:
1. Database write throughput (300MB/s limit on standard SSD)
2. Redis lock contention (unless cluster)
3. Network bandwidth (10Gbps limit)
```

**Database sharding strategy:**

```python
class ShardingStrategy:
    """
    Shard по hotel_id для равномерного распределения нагрузки
    """

    SHARD_COUNT = 16  # Power of 2
    REPLICATION_FACTOR = 2

    def get_shard_id(self, hotel_id: str) -> int:
        """
        Consistent hash hotel_id к shard
        """
        hash_val = int(hashlib.md5(hotel_id.encode()).hexdigest(), 16)
        return hash_val % self.SHARD_COUNT

    def get_db_config(self, hotel_id: str):
        """
        Resolve к konkrétní database instance
        """
        shard_id = self.get_shard_id(hotel_id)

        # Shard 0 → db0.primary, db0.replica1, db0.replica2
        return {
            'primary': f"db{shard_id}.primary.internal:5432",
            'replicas': [
                f"db{shard_id}.replica1.internal:5432",
                f"db{shard_id}.replica2.internal:5432"
            ]
        }

    async def execute_read(self, hotel_id: str, query: str, params: list):
        """
        Read from replica для load distribution
        """
        db_config = self.get_db_config(hotel_id)

        # Round-robin через replicas
        replica = random.choice(db_config['replicas'])

        connection = await get_connection(replica)
        return await connection.fetch(query, *params)

    async def execute_write(self, hotel_id: str, query: str, params: list):
        """
        Write to primary (replicated to replicas)
        """
        db_config = self.get_db_config(hotel_id)

        connection = await get_connection(db_config['primary'])
        return await connection.execute(query, *params)


class ShardingRouter:
    """
    Route запросов к korektnímu shard'у
    """

    def __init__(self):
        self.sharding = ShardingStrategy()

    async def create_booking(self, request):
        """
        Route по hotel_id
        """
        hotel_id = request.hotel_id

        result = await self.sharding.execute_write(
            hotel_id,
            """INSERT INTO bookings (...) VALUES (...)""",
            [...]
        )

        return result

    async def get_hotel_bookings(self, hotel_id: str):
        """
        Route read к replica
        """
        results = await self.sharding.execute_read(
            hotel_id,
            """SELECT * FROM bookings WHERE hotel_id = $1""",
            [hotel_id]
        )

        return results
```

**Connection pooling и timeout:**

```python
class ConnectionPool:
    """
    Reuse connections, manage timeouts
    """

    def __init__(self, min_size=10, max_size=100):
        self.min_size = min_size
        self.max_size = max_size
        self.pools = {}  # db_host → pool

    async def init(self):
        # Pre-create connections на startup
        for db_host in self.get_all_db_hosts():
            self.pools[db_host] = await asyncpg.create_pool(
                db_host,
                min_size=self.min_size,
                max_size=self.max_size,
                timeout=5,  # Connection timeout
                command_timeout=30,  # Query timeout
                max_cached_statement_lifetime=3600,
                max_cacheable_statement_size=15000
            )

    async def execute(self, db_host: str, query: str, *args):
        pool = self.pools[db_host]

        async with pool.acquire() as connection:
            # Auto-timeout query
            try:
                result = await asyncio.wait_for(
                    connection.fetch(query, *args),
                    timeout=30
                )
                return result
            except asyncio.TimeoutError:
                logger.error(f"Query timeout: {query}")
                raise QueryTimeoutError()
```

**Batch updates для массовых операций:**

```python
class BatchBookingProcessor:
    """
    При 100K req/sec, нельзя update per-request
    Group updates и писать в batch
    """

    BATCH_SIZE = 1000
    BATCH_TIMEOUT = 0.1  # 100ms

    def __init__(self):
        self.batch_queue = asyncio.Queue()
        self.pending_updates = defaultdict(list)

    async def submit_booking(self, booking_data: dict):
        """
        Add к batch queue
        """
        await self.batch_queue.put(booking_data)

        return await self.wait_for_result(booking_data['id'])

    async def batch_processor(self):
        """
        Background task: periodically flush batch
        """
        while True:
            try:
                # Collect batch
                batch = []

                deadline = asyncio.get_event_loop().time() + self.BATCH_TIMEOUT

                while len(batch) < self.BATCH_SIZE:
                    remaining = deadline - asyncio.get_event_loop().time()

                    if remaining <= 0:
                        break

                    try:
                        item = await asyncio.wait_for(
                            self.batch_queue.get(),
                            timeout=remaining
                        )
                        batch.append(item)
                    except asyncio.TimeoutError:
                        break

                if not batch:
                    continue

                # Write batch to DB
                await self.flush_batch(batch)

            except Exception as e:
                logger.error(f"Batch processing failed: {e}")
                await asyncio.sleep(1)

    async def flush_batch(self, batch: list):
        """
        Single INSERT with many rows
        """

        values = []
        params = []

        for item in batch:
            placeholders = f"($1, ${len(params)+1}, ...)"
            values.append(placeholders)
            params.extend([item['hotel_id'], item['room_type_id'], ...])

        query = f"""
            INSERT INTO bookings (...) VALUES
            {', '.join(values)}
            RETURNING id
        """

        results = await db.fetch(query, *params)

        # Notify waiters
        for item, result in zip(batch, results):
            self.pending_results[item['id']].set_result(result)
```

**Circuit breaker для failure handling:**

```python
class CircuitBreaker:
    """
    Prevent cascading failures
    """

    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
        self.last_failure_time = None

    async def call(self, fn, *args, **kwargs):
        """
        Execute fn with circuit breaker
        """

        if self.state == 'OPEN':
            # Check if timeout passed
            if (datetime.utcnow() - self.last_failure_time).seconds > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise CircuitOpenError()

        try:
            result = await fn(*args, **kwargs)

            if self.state == 'HALF_OPEN':
                self.state = 'CLOSED'
                self.failure_count = 0

            return result

        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = datetime.utcnow()

            if self.failure_count >= self.failure_threshold:
                self.state = 'OPEN'
                logger.error(f"Circuit breaker OPEN: {e}")

            raise


class ResilientBookingService:
    def __init__(self):
        self.db_breaker = CircuitBreaker(failure_threshold=5, timeout=30)
        self.payment_breaker = CircuitBreaker(failure_threshold=3, timeout=60)

    async def create_booking(self, request):
        try:
            # Protected call to DB
            booking = await self.db_breaker.call(
                self.db.create_booking,
                request
            )

            # Protected call to payment
            payment = await self.payment_breaker.call(
                self.payment_service.charge,
                request.payment_method_id,
                booking.total_amount
            )

            return booking

        except CircuitOpenError:
            # Fallback: queue for later processing
            await self.fallback_queue.put({
                'request': request,
                'retry_at': datetime.utcnow() + timedelta(seconds=30)
            })

            return QueuedResponse()
```

**Caching strategy при масштабировании:**

```python
class CachingStrategy:
    """
    Multi-level caching при 100K req/sec
    """

    # L1: In-memory (per-process)
    local_cache = TTLCache(maxsize=10000, ttl=60)

    # L2: Distributed (Redis)
    redis_cache = RedisCache(ttl=300)

    # L3: Database

    async def get_hotel_info(self, hotel_id: str):
        """
        3-tier lookup
        """

        # L1: Check local cache
        if hotel_id in self.local_cache:
            return self.local_cache[hotel_id]

        # L2: Check Redis
        cached = await self.redis_cache.get(f"hotel:{hotel_id}")
        if cached:
            self.local_cache[hotel_id] = cached
            return cached

        # L3: Load from DB
        hotel = await db.get_hotel(hotel_id)

        # Cache at all levels
        await self.redis_cache.set(f"hotel:{hotel_id}", hotel, ttl=300)
        self.local_cache[hotel_id] = hotel

        return hotel
```

**Monitoring при 100K req/sec:**

```python
class MonitoringService:
    """
    Track system health at scale
    """

    async def setup_metrics(self):
        # Prometheus metrics
        self.booking_latency = Histogram(
            'booking_latency_ms',
            buckets=[10, 50, 100, 500, 1000, 5000]
        )

        self.booking_rate = Counter('bookings_total')

        self.db_pool_utilization = Gauge('db_pool_utilization')

        self.cache_hit_rate = Counter(
            'cache_hits_total'
        )

        self.queue_depth = Gauge('task_queue_depth')

    async def health_check(self):
        """
        Monitor critical metrics
        """

        metrics = {
            'db_latency_p99': await self.get_db_latency_percentile(99),
            'redis_latency_p99': await self.get_redis_latency_percentile(99),
            'cache_hit_rate': self.get_cache_hit_rate(),
            'queue_backlog': await self.get_queue_depth(),
            'error_rate': self.get_error_rate()
        }

        # Alert thresholds
        if metrics['db_latency_p99'] > 500:
            alert('Database latency HIGH')

        if metrics['queue_backlog'] > 100000:
            alert('Queue backlog HIGH')

        if metrics['error_rate'] > 0.01:  # 1%
            alert('Error rate HIGH')

        return metrics
```

**Типичные ошибки при масштабировании.**

1. Не shard'ить → single database bottleneck
2. Synchronous booking + payment → timeouts
3. Кешировать без invalidation → stale data
4. Забыть connection pooling → connection exhaustion
5. No circuit breaker → cascading failures

**На интервью.**

Вопрос: "Как обработать spike до 200K/sec?"
→ Auto-scaling (add more API gateways), graceful degradation (cache-only mode)

Вопрос: "Как обнаружить при какой нагрузке система ломается?"
→ Load testing, gradual increase, monitor key metrics

Вопрос: "Как сохранить данные при database failure?"
→ Replication, WAL, backup to S3, multi-region setup

---

## Сравнительные таблицы

**Все решения для предотвращения double booking:**

```
                Pessimistic   Optimistic   Redis Lock
Latency         High          Low          Medium
Contention      Good          Poor         Good
Deadlock        Yes           No           No
Complexity      Low           Medium       Medium
Cost            Low           Low          Medium
Best For        High conflict Low conflict  Distributed
```

**Inventory management approaches:**

```
                    Per-Room    Per-Room-Type   Per-Date
Scalability         Poor        Good            Excellent
Query Efficiency    Slow        Fast            Very Fast
Update Frequency    High        Medium          Low
Storage             Huge        Medium          Small
```

---

## На интервью: Final Strategy

**Как структурировать ответ:**

1. **Clarify requirements** (2 минуты)
   - Daily volume, peak QPS
   - Consistency requirements (strong vs eventual)
   - Geographic distribution

2. **High-level architecture** (2 минуты)
   - Components: search, booking, inventory
   - Data flow diagram
   - Key trade-offs

3. **Deep dive** (5-7 минут)
   - Choose 1-2 critical problems
   - Show locking strategy, database design
   - Include code snippets

4. **Scale it up** (2 минуты)
   - How to handle 10x traffic
   - Sharding strategy, caching
   - Bottlenecks and solutions

5. **Trade-offs & alternatives** (1 минута)
   - Consistency vs availability
   - Consistency vs latency
   - Why not other approaches

**Типичные follow-ups:**

- "Как обработать flash sale?"
- "Как реализовать waitlist?"
- "Что если payment gateway медленный?"
- "Как интегрироваться с GDS?"
- "Как обрабатывать timezone?"
- "Как реализовать dynamic pricing?"

**Pro tips:**

- Не говорить про все сразу — depth лучше breadth
- Код показывает понимание лучше чем words
- Признавать trade-offs — показывает maturity
- Спрашивать clarifying questions — shows communication

---

## Дополнительные ресурсы

### Связанные топики:
- [Транзакции в базах данных](../06-databases/03-transactions.md) — ACID, isolation levels, locking
- [Rate Limiter](./02-rate-limiter.md) — API protection при high load
- [Distributed Cache](./07-distributed-cache.md) — Redis patterns
- [Notification System](./03-notification-system.md) — Event-driven architecture

### Практика:
- Реализовать simple booking system с pessimistic locking
- Добавить optimistic locking и compare performance
- Implement Redis distributed lock
- Design calendar view с materialized view
- Implement notification queue with retry

---

[← Назад к списку тем](README.md)
