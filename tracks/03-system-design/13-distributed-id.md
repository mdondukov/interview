# 13 — Distributed ID Generator

Развёрнутые вопросы и ответы про проектирование сервиса генерации распределённых ID: UUID, Snowflake, ULID, синхронизация часов, монотонность, предотвращение коллизий, кодирование ID. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [12-ride-sharing](./12-ride-sharing.md) · Следующая тема: [14-monitoring-alerting](./14-monitoring-alerting.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**UUID** — универсально уникальный идентификатор размером 128 бит, который можно генерировать локально на каждом клиенте без координации с другими узлами. Есть несколько версий UUID (v1, v4, v5), которые используют разные методы генерации (случайные числа, MAC адрес, timestamp и т.д.). UUID гарантирует отсутствие коллизий и очень просто реализуется, но они намного больше (32 символа) чем Snowflake ID. UUID идеальны для случаев, когда простота важнее компактности.

**Snowflake ID** — 64-битный распределённый ID, разработанный Twitter, который комбинирует timestamp (41 бит), идентификатор машины (10 бит) и последовательность (12 бит). Это гораздо компактнее UUID, сортируется по времени (что полезно для индексирования), и может масштабироваться на тысячи машин. Snowflake требует синхронизации часов между серверами и небольшой обработки на стороне генератора ID. Это стандарт для распределённых систем больших масштабов.

**ULID** — sortable ID размером 128 бит, который комбинирует timestamp (48 бит) с наносекундной точностью и случайное число (80 бит). Это компромисс между UUID и Snowflake: сохраняет сортируемость Snowflake с меньше требованиями к синхронизации часов. ULID имеет хорошую производительность и растущую популярность в новых приложениях. Однако ULID менее распространён чем Snowflake и не все базы данных имеют встроенную поддержку.

**Monotonicity** — свойство ID увеличиваться (или оставаться неизменным) с течением времени. Это важно для индексирования, так как монотонные ID позволяют базе данных добавлять новые записи в конец индекса вместо вставки в середину. Монотонность также критична для аналитики и упорядочения событий по времени. Однако монотонность требует синхронизации часов и может быть сложна в распределённых системах.

**Clock Skew** — расхождение системных часов между разными серверами в распределённой системе. Это может быть проблемой при генерации ID, основанных на timestamp: если часы на разных серверах показывают разное время, то ID может потерять монотонность или может произойти коллизия. Решение включает использование NTP для синхронизации часов или использование других методов (например, логических часов).

**Collision-Free** — гарантия отсутствия дублей ID при генерации очень большого количества идентификаторов. Эта гарантия зависит от размера ID и алгоритма генерации: UUID с 128 битами практически невозможно столкнуться, а Snowflake с 64 битами может гарантировать отсутствие коллизий при условии что machine_id уникален. Правильный выбор размера ID критичен для избежания коллизий.

**Distributed ID Generator** — сервис (или набор сервисов), который генерирует уникальные ID на множестве узлов в распределённой системе. Это сервис избегает Single Point of Failure (SPOF), который был бы, если бы ID генерировались только одним сервером, и масштабируется горизонтально путём добавления новых узлов. Обычно реализуется как REST сервис или встроено в базу данных.

**Centralized Counter** — простой подход генерации ID через единую базу данных, которая выдаёт последовательные ID по порядку. Это очень простой в реализации подход, который гарантирует отсутствие коллизий и монотонность. Однако он становится bottleneck при масштабировании, так как все запросы на ID проходят через одну БД. Этот подход может работать для небольших систем, но не подходит для высоконагруженных приложений.

**Bit Packing** — техника кодирования информации в разные части ID, где каждый отрезок бит кодирует разную информацию. Например, в Snowflake ID первые 41 бит кодируют timestamp, следующие 10 бит кодируют machine_id, и последние 12 бит кодируют sequence. Bit Packing позволяет сжать несколько логических значений в компактное 64-битное целое число, что улучшает эффективность хранилища и передачи данных.

**Batching** — получение пакета (например, 1000) ID за один RPC запрос вместо одного ID за запрос. Это снижает задержку и улучшает пропускную способность (throughput), так как обеспечивает лучшую утилизацию сетевого соединения и уменьшает количество контекстных переходов. Например, приложение может запросить 1000 ID один раз и использовать их локально, вместо того чтобы делать 1000 отдельных запросов.

---

## Вопросы и разборы

### 1. Зачем нужны распределённые ID и какие требования к ним?

**Зачем спрашивают.** Это первый вопрос на интервью: надо понять, какую проблему решает ID generator и почему простые подходы (автоинкремент БД) не подходят для распределённых систем. Интервьюер проверяет понимание масштабируемости и требований.

**Короткий ответ.** Распределённые ID нужны для уникальной идентификации сущностей (записи, события, задачи) в системах с множеством узлов. Простой БД-счётчик не масштабируется: один узел-координатор становится bottleneck. Нужна система, которая генерирует уникальные ID без центрального координатора, с низкой задержкой (< 1ms), высокой пропускной способностью (100K+ IDs/sec per node) и поддержкой временной сортировки.

**Детальный разбор.**

**Проблема с единым счётчиком:**
```
┌─────────────────┐
│   Application   │
│    Server 1     │
└────────┬────────┘
         │
         │ SELECT nextval('seq') ─┐
         │                        │
┌────────▼─────────────────────────▼──────┐
│   Central Database / Coordinator         │
│         Single Point of Failure          │
│   Bottleneck: ~10K IDs/sec max          │
└──────────────────────────────────────────┘

Issues:
- Network latency на каждый ID запрос
- SPOF (Single Point of Failure) — если БД падает, нет новых ID
- Сложно масштабировать на несколько датацентров
- Не поддерживает offline generation
```

**Требования к распределённому ID generator:**

Функциональные:
- Генерировать глобально уникальные ID (collision-free)
- Поддерживать несколько датацентров и регионов
- Быть устойчивым к сбоям узлов (no SPOF)
- Быть детерминированным (если нужна временная сортировка)

Нефункциональные:
- Throughput: ≥100K IDs/sec per node, ≥5M IDs/sec globally
- Latency: < 1ms per ID (без network roundtrip)
- Availability: 99.99%+ (no SPOF)
- ID size: ≤128 bits (обычно 64 bits)
- Monotonicity (optional): ID увеличивается со временем

**Capacity estimation:**
```
Scale:
- 100K IDs/second per node
- 10 nodes per datacenter
- 5 datacenters worldwide
- Total: 5M IDs/second globally

64-bit ID space:
- Total IDs available: 2^64 = 18.4 × 10^18
- At 5M IDs/sec: 116,000 years before exhaustion
- Even at 1B IDs/sec: 584 years

128-bit ID space (UUID):
- Total IDs: 2^128 = 3.4 × 10^38
- Practically infinite for any realistic scale

Storage implications:
- UUID (128-bit): 16 bytes per ID
- Snowflake (64-bit): 8 bytes per ID
- Per 1B records: 8GB vs 16GB difference
- For 1T records: 8TB vs 16TB
```

**Пример.**
```python
# Проблема: centralized approach
class CentralizedIDGenerator:
    def __init__(self, db_connection):
        self.db = db_connection

    async def generate(self) -> int:
        # Требует network roundtrip к БД на каждый ID
        result = await self.db.execute("SELECT nextval('id_sequence')")
        return result[0]

# Масштабирование: децентрализованный подход
class DistributedIDGenerator:
    def __init__(self, machine_id: int):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1

    def generate(self) -> int:
        # Локальное вычисление без network roundtrip
        timestamp = self._current_time_ms()

        if timestamp == self.last_timestamp:
            self.sequence += 1
        else:
            self.sequence = 0

        self.last_timestamp = timestamp

        # Комбинируем: timestamp (41 bits) + machine_id (10 bits) + sequence (12 bits)
        id = (timestamp << 22) | (self.machine_id << 12) | self.sequence
        return id

    def _current_time_ms(self) -> int:
        return int(time.time() * 1000)
```

**Типичные ошибки.**
- Выбрать слишком простое решение (UUID v4) без понимания требований к сортировке
- Использовать БД-счётчик как единственный источник (SPOF, не масштабируется)
- Забыть о clock skew / clock drift между серверами
- Не учесть требования к throughput (нужна ≥4K IDs/ms на узле для 100K/sec)
- Выбрать слишком большие ID (128 bits вместо 64 bits) без необходимости

**На интервью.**
- Начни с clarification: «Какие требования к ID? Нужна ли сортировка? Какой масштаб?»
- Объясни trade-off между размером ID и информацией, которую он кодирует
- Покажи понимание проблем распределённых систем: clock skew, SPOF, network delays
- Упомяни основные подходы (UUID, Snowflake, ULID) и когда их использовать
- Будь готов к follow-up про clock sync, обработку сбоев, масштабирование

---

### 2. Какие версии UUID существуют и в чём их различия?

**Зачем спрашивают.** UUID — самый простой и часто используемый стандарт для уникальных ID. Нужно знать различия между версиями (v1, v4, v5), их strengths и weaknesses. Интервьюер проверяет глубину понимания.

**Короткий ответ.** UUID имеет несколько версий: v1 (time-based, использует MAC адрес), v4 (случайный, простой), v5 (детерминированный хеш). v1 sortable но раскрывает MAC адрес. v4 простой и приватный, но не sortable и занимает 128 bits. v5 — хеш namespace и имени. Для распределённых ID лучше v1 (если нужна сортировка) или использовать вместо UUID другой подход.

**Детальный разбор.**

**UUID структура:**
```
UUID = 128 bits = 36 символов (с дефисами) = 16 байт

Стандартный формат:
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
8       4    4    4    12
(каждый x — один hex digit)

Пример: 550e8400-e29b-41d4-a716-446655440000
```

**UUID v1 (Time-based with MAC address):**
```
┌────────────────────────────────────────────────┐
│            UUID v1 (128 bits)                   │
├────────────────────────────────────────────────┤
│ 32 bits    │ 16 bits  │ 16 bits  │ 48 bits    │
│ Timestamp  │ Timestamp│ Timestamp│ MAC/Random │
│ (seconds)  │ (microsec)│(version) │ (node ID)  │
├────────────────────────────────────────────────┤
│ 6ba7b810   │ 9dad     │ 11d1     │ 80b4 00c04fd430c8
│            │          │          │
│ Time of    │ Microsec │ Version1 │ MAC address of node
│ generation │ + variant│ + clock  │ or random
└────────────────────────────────────────────────┘

Преимущества:
- Sortable по времени (по первым 60 bits)
- Можно извлечь timestamp из ID
- Deterministic для того же узла в то же время

Недостатки:
- Раскрывает MAC адрес узла (privacy issue)
- Требует часов на каждом узле (clock skew проблемы)
- 128 bits очень большой
- Коллизии если на узле > 10K IDs в одну микросекунду
```

**UUID v4 (Random):**
```
┌────────────────────────────────────────────────┐
│            UUID v4 (128 bits)                   │
├────────────────────────────────────────────────┤
│         122 bits random          │ 6 bits (version/variant)
│ Completely pseudorandom values   │ Fixed to 0100 and 10xx
├────────────────────────────────────────────────┤
│ 550e8400   │ e29b     │ 41d4     │ a716 446655440000
│ 32 random  │ 12 rnd   │ 16 rnd   │ 48 random
│ bits       │ bits     │ bits     │ bits (with fixed marker)
└────────────────────────────────────────────────┘

Преимущества:
- Простой и не требует координации
- Приватный — не раскрывает информацию
- Стандартный, поддерживается везде
- Очень низкая вероятность коллизии (birthday paradox: ~2.71 × 10^18 IDs для 50% коллизии)

Недостатки:
- Не sortable (совершенно случайный)
- 128 bits — большой размер
- Плохая локальность для индексов
- Случайность может быть качеством зависит от RNG
```

**UUID v5 (Deterministic hash-based):**
```
UUID v5 = SHA-1(namespace + name)

Структура:
- Namespace ID (128 bits) — какому "пространству" принадлежит имя
- Name (variable) — значение, для которого генерируем ID
- SHA-1 hash первых 128 bits из SHA-1(namespace + name)

Примеры namespaces:
- DNS namespace: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
- URL namespace: 6ba7b811-9dad-11d1-80b4-00c04fd430c8
- ISO OID namespace: 6ba7b812-9dad-11d1-80b4-00c04fd430c8
- X.500 DN namespace: 6ba7b814-9dad-11d1-80b4-00c04fd430c8

Преимущества:
- Deterministic: один namespace+name → один ID всегда
- Не требует хранить маппинг
- Reproducible (можно пересчитать ID без БД)

Недостатки:
- Не sortable по времени
- SHA-1 вычисление дороже чем простой random
- Нужно знать namespace (не подходит для ad-hoc ID)
```

**Сравнение:**
```
┌─────────────┬──────────┬────────────┬─────────┬──────────────┐
│ Версия      │ Размер   │ Сортируемо │ Privacy │ Use Case     │
├─────────────┼──────────┼────────────┼─────────┼──────────────┤
│ v1 (time)   │ 128 bits │ Да*        │ Нет     │ Not for dist │
│ v4 (random) │ 128 bits │ Нет        │ Да      │ Simple cases │
│ v5 (hash)   │ 128 bits │ Нет        │ Да      │ Determinism  │
└─────────────┴──────────┴────────────┴─────────┴──────────────┘
```

**Пример.**
```python
import uuid
import time

# UUID v1 - time-based (раскрывает MAC)
id_v1 = uuid.uuid1()
# Output: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
# Можно извлечь timestamp:
print(id_v1.time)  # 137644046748375168 (100-nanosecond intervals since 1582-10-15)

# UUID v4 - random (приватный, не сортируется)
id_v4 = uuid.uuid4()
# Output: 550e8400-e29b-41d4-a716-446655440000
# Всегда разный, даже для одного источника в один момент

# UUID v5 - hash-based DNS namespace
namespace_dns = uuid.NAMESPACE_DNS
name = "example.com"
id_v5 = uuid.uuid5(namespace_dns, name)
# Output: e6f9e0ba-f6b6-5a44-a3e7-f4b7c1f0d5e2
# Всегда одинаковый для одного имени

# Проблема с UUID для distributed ID:
class UUIDBasedIDGen:
    def generate_id(self) -> str:
        return str(uuid.uuid4())

# Результат:
for i in range(3):
    id = UUIDBasedIDGen().generate_id()
    print(f"ID {i+1}: {id}")
    # Полностью несортируемые ID — плохо для индексов!
    # Требует 128 bits — дорого для storage

# Лучше использовать UUID v1 только если:
# - Нужна сортировка
# - OK раскрывать MAC адрес
# - Синхронизированы часы на всех узлах
# Иначе — лучше Snowflake или ULID
```

**Типичные ошибки.**
- Использовать UUID v1 в распределённой системе без синхронизации часов (clock skew → коллизии)
- Выбрать UUID v4 когда нужна сортировка (индексы будут неэффективны)
- Забыть что UUID раскрывает информацию (v1 → MAC адрес, v5 → dependency на namespace)
- Использовать слишком большой размер (128 bits) когда можно 64 bits
- Предположить что UUID v4 не может иметь коллизии (birthday paradox — возможны!)

**На интервью.**
- Объясни различия между v1, v4, v5 и когда их использовать
- Упомяни privacy и security implications (особенно v1)
- Покажи понимание trade-off: простота vs размер vs сортировка
- Объясни почему UUID часто не лучший выбор для distributed ID (вместо этого Snowflake/ULID)
- Будь готов к follow-up про как парсить timestamp из UUID v1

---

### 3. Как работает Snowflake ID (Twitter) и почему это хороший выбор?

**Зачем спрашивают.** Snowflake — де-факто стандарт для distributed ID generation в крупных компаниях (Twitter, Uber, Discord). Это лучший balance между простотой, размером, throughput и информативностью. Интервьюер проверяет глубокое понимание деталей реализации.

**Короткий ответ.** Snowflake кодирует 64-bit ID как: 1 бит (знак) + 41 бит (timestamp в мс) + 10 бит (machine ID) + 12 бит (sequence counter). Это даёт 4K IDs per millisecond per machine, sortable по времени, без центрального координатора. Требует machine ID assignment и синхронизации часов, но масштабируется на миллионы IDs/sec.

**Детальный разбор.**

**Snowflake структура (64 bits):**
```
┌─────────────────────────────────────────────────────────────────┐
│                        64-bit Snowflake ID                       │
├─────────────────────────────────────────────────────────────────┤
│  1 bit  │      41 bits       │  10 bits  │      12 bits        │
│ (sign)  │    (timestamp)     │ (machine) │    (sequence)       │
│    0    │ milliseconds       │   ID      │ counter per ms      │
│        │ since custom epoch │           │                      │
├─────────────────────────────────────────────────────────────────┤
│    0    │  1704067200000     │   42      │      1234           │
│        │ (2024-01-01)       │ (node 42) │                      │
└─────────────────────────────────────────────────────────────────┘

Bit allocation explanation:

1 bit (sign):
  - Always 0 (makes ID positive in two's complement)
  - Allows database sorting by ID as integer
  - Unused: could use for flag/shard ID

41 bits (timestamp):
  - Milliseconds since custom epoch (e.g., 2024-01-01)
  - 2^41 = 2,199,023,255,552 milliseconds ≈ 69.7 years
  - Sortable by time (IDs with earlier timestamp < later timestamp)

10 bits (machine ID):
  - Identifies which node generated the ID
  - 2^10 = 1,024 possible machine IDs (0-1023)
  - Requires coordination (ZK/etcd) to assign unique IDs per node
  - Can encode datacenter + node in this field

12 bits (sequence):
  - Counter that increments within same millisecond
  - 2^12 = 4,096 possible sequences per millisecond
  - Resets when moving to next millisecond
  - Prevents duplicates from same machine in same ms

Maximum throughput per machine:
  4,096 sequences/ms × 1,000 ms/sec = 4,096,000 IDs/sec
  ≈ 4M IDs per machine per second
```

**Визуализация генерации:**
```
Time 1704067200000 ms:
┌─────┬──────────────────────┬──────────┬────────────┐
│  0  │    1704067200000     │   42     │     0      │ ID = 0
├─────┼──────────────────────┼──────────┼────────────┤
│  0  │    1704067200000     │   42     │     1      │ ID = 1
├─────┼──────────────────────┼──────────┼────────────┤
│  0  │    1704067200000     │   42     │     2      │ ID = 2
├─────┼──────────────────────┼──────────┼────────────┤
│  0  │    1704067200000     │   42     │  4095      │ ID = 4095 (max)
└─────┴──────────────────────┴──────────┴────────────┘
                    ↓
            Достигнут максимум последовательности
                    ↓
Time 1704067200001 ms:
┌─────┬──────────────────────┬──────────┬────────────┐
│  0  │    1704067200001     │   42     │     0      │ ID reset to 0
└─────┴──────────────────────┴──────────┴────────────┘

Machine 43 в то же время:
┌─────┬──────────────────────┬──────────┬────────────┐
│  0  │    1704067200000     │   43     │     0      │ ID = 0
├─────┼──────────────────────┼──────────┼────────────┤
│  0  │    1704067200000     │   43     │     1      │ ID = 1
└─────┴──────────────────────┴──────────┴────────────┘
(Same timestamp, different machine_id → different IDs!)
```

**Пример.**
```python
import time
import threading
from datetime import datetime

class SnowflakeIDGenerator:
    # Epoch: 2024-01-01 00:00:00 UTC
    # Используем custom epoch чтобы увеличить жизненный цикл ID
    EPOCH = 1704067200000  # January 1, 2024 in milliseconds

    MACHINE_BITS = 10
    SEQUENCE_BITS = 12
    TIMESTAMP_BITS = 41

    # Максимальные значения
    MAX_MACHINE_ID = (1 << MACHINE_BITS) - 1      # 1023
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1       # 4095
    MAX_TIMESTAMP = (1 << TIMESTAMP_BITS) - 1     # 2,199,023,255,551

    def __init__(self, machine_id: int):
        if machine_id < 0 or machine_id > self.MAX_MACHINE_ID:
            raise ValueError(f"Machine ID must be 0-{self.MAX_MACHINE_ID}")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()  # Для thread-safe генерации

    def generate(self) -> int:
        """Generate next Snowflake ID"""
        with self.lock:
            timestamp = self._current_time_ms()

            # Проверка clock backwards
            if timestamp < self.last_timestamp:
                raise ClockMovedBackwardError(
                    f"Clock moved backwards. Last: {self.last_timestamp}, Current: {timestamp}"
                )

            if timestamp == self.last_timestamp:
                # Та же миллисекунда — инкрементируем sequence
                self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE

                if self.sequence == 0:
                    # Переполнение sequence — ждём следующую миллисекунду
                    timestamp = self._wait_next_millis(timestamp)
            else:
                # Новая миллисекунда — сбрасываем sequence
                self.sequence = 0

            self.last_timestamp = timestamp

            # Собираем ID: (timestamp << 22) | (machine_id << 12) | sequence
            # Shift:
            # - timestamp на 22 bits (MACHINE_BITS + SEQUENCE_BITS)
            # - machine_id на 12 bits (SEQUENCE_BITS)
            # - sequence на 0 bits (уже в позиции)

            id = (
                (timestamp - self.EPOCH) << (self.MACHINE_BITS + self.SEQUENCE_BITS) |
                (self.machine_id << self.SEQUENCE_BITS) |
                self.sequence
            )

            return id

    def _current_time_ms(self) -> int:
        """Текущее время в миллисекундах"""
        return int(time.time() * 1000)

    def _wait_next_millis(self, last_timestamp: int) -> int:
        """Дождитесь следующей миллисекунды"""
        ts = self._current_time_ms()
        while ts <= last_timestamp:
            ts = self._current_time_ms()
        return ts

    @staticmethod
    def parse(snowflake_id: int) -> dict:
        """Разбор Snowflake ID на компоненты"""
        sequence = snowflake_id & SnowflakeIDGenerator.MAX_SEQUENCE
        machine_id = (snowflake_id >> SnowflakeIDGenerator.SEQUENCE_BITS) & SnowflakeIDGenerator.MAX_MACHINE_ID
        timestamp = (snowflake_id >> (SnowflakeIDGenerator.MACHINE_BITS + SnowflakeIDGenerator.SEQUENCE_BITS))
        timestamp_ms = timestamp + SnowflakeIDGenerator.EPOCH

        return {
            "id": snowflake_id,
            "timestamp_ms": timestamp_ms,
            "machine_id": machine_id,
            "sequence": sequence,
            "datetime": datetime.fromtimestamp(timestamp_ms / 1000),
        }

# Использование:
generator = SnowflakeIDGenerator(machine_id=42)

# Генерируем несколько ID:
ids = [generator.generate() for _ in range(5)]
print(f"Generated IDs: {ids}")
# Output: [123456789, 123456790, 123456791, 123456792, 123456793]

# Разбираем ID:
for id in ids:
    parsed = SnowflakeIDGenerator.parse(id)
    print(f"ID {id}: machine={parsed['machine_id']}, seq={parsed['sequence']}, time={parsed['datetime']}")

# Тестируем высокую пропускную способность:
import time
start = time.time()
count = 1_000_000
for _ in range(count):
    generator.generate()
elapsed = time.time() - start
print(f"Generated {count:,} IDs in {elapsed:.3f}s = {count/elapsed:,.0f} IDs/sec")
# Output: Generated 1,000,000 IDs in 0.083s = 12,000,000 IDs/sec
```

**Преимущества Snowflake:**
```
✓ 64 bits — compact (в 2 раза меньше чем UUID)
✓ Sortable по времени — IDs монотонно растут (хорошо для индексов)
✓ Декодируемый — можно извлечь timestamp, machine_id, sequence
✓ Высокая пропускная способность — 4M IDs/sec per machine
✓ Масштабируемый — поддерживает 1024 машин
✓ Нет централизации — каждый узел может генерировать независимо
✓ Distribuable — работает в разных датацентрах (если машины получают разные IDs)
```

**Недостатки Snowflake:**
```
✗ Requires machine ID coordination — нужен ZK/etcd для assignment
✗ Clock dependency — если часы прыгают назад, может быть error
✗ Finite lifetime — 69 лет от custom epoch (но в production переходят на больший epoch)
✗ Non-monotonic clock — IDs могут быть "out of order" если machine_id не в хронологическом порядке
✗ Complex to implement — больше кода чем UUID
```

**Типичные ошибки.**
- Забыть про machine ID assignment (collision если два узла имеют одинаковый ID)
- Не синхронизировать часы (clock drift → out-of-order IDs, ошибки)
- Использовать слишком маленький/большой custom epoch
- Не обработать sequence overflow
- Предположить monotonic guarantees когда их нет (разные машины → не всегда монотонные)

**На интервью.**
- Объясни bit layout и почему именно такие размеры (41 bits timestamp, 10 bits machine, 12 bits sequence)
- Покажи как вычислить maximum throughput (4K sequences × 1K ms = 4M IDs/sec)
- Обсуди требования к machine ID assignment и координации
- Упомяни clock synchronization проблемы и как их обработать
- Покажи как парсить ID и извлечь timestamp
- Уточняющий вопрос: как масштабировать на несколько датацентров? (answer: encode DC в machine_id)

---

### 4. Что такое ULID и когда его использовать вместо Snowflake?

**Зачем спрашивают.** ULID — более новый стандарт, который решает некоторые проблемы Snowflake (notamment не требует machine ID assignment). Интервьюер проверяет знание альтернатив и когда их выбирать.

**Короткий ответ.** ULID (Universally Unique Lexicographically Sortable ID) кодирует 128 bits как: 48 bits timestamp (ms) + 80 bits cryptographic random. Это даёт 64-character URL-safe, case-insensitive ID, sortable по времени, без требования координации machine IDs. Используй ULID когда не нужна machine ID complexity, нужна higher collision resistance, или когда ID часто используется как string.

**Детальный разбор.**

**ULID структура (128 bits, 26 символов):**
```
┌─────────────────────────────────────────────────┐
│         ULID (128 bits, 26 characters)           │
├─────────────────────────────────────────────────┤
│      48 bits        │        80 bits            │
│    (timestamp)      │      (randomness)         │
│  milliseconds since │  cryptographic random    │
│   Unix epoch (1970) │  / monotonic counter     │
├─────────────────────────────────────────────────┤
│ 01ARZ3NDEK          │   TSV4RRFFQ69G5FAV       │
│ (10 chars)          │   (16 chars)             │
│ Timestamp part      │   Random part            │
└─────────────────────────────────────────────────┘

Кодирование: Crockford's Base32
  Алфавит: 0123456789ABCDEFGHJKMNPQRSTVWXYZ
  (32 символа, без I, L, O, U для избежания путаницы)
  26 символов ULID = 10 символов timestamp + 16 символов random

Часть timestamp (48 бит = 10 символов Crockford Base32):
  - 48 бит = 281,474,976,710,656 миллисекунд
  - ≈ 8,921 лет (с 1970 по 10889)

Часть random (80 бит = 16 символов Crockford Base32):
  - 2^80 = 1,208,925,819,614,629,174,706,176 комбинаций
  - Вероятность коллизии экстремально низкая
```

**ULID vs Snowflake:**
```
┌──────────────────┬─────────────────┬──────────────────┐
│ Aspect           │ Snowflake       │ ULID             │
├──────────────────┼─────────────────┼──────────────────┤
│ Size             │ 64 bits (8B)    │ 128 bits (16B)   │
│ Encoding         │ Binary          │ Base32 (26 char) │
│ URL-safe         │ No              │ Yes              │
│ Sortable         │ Yes (numeric)   │ Yes (lexical)    │
│ Machine ID req   │ Yes             │ No               │
│ Coordination     │ Yes (ZK/etcd)   │ No               │
│ Collision risk   │ Birthday paradox│ Negligible       │
│ Monotonicity     │ Partial*        │ Monotonic opt    │
│ Storage          │ 8 bytes         │ 16 bytes         │
│ Read perf        │ Better (compact)│ Worse (string)   │
│ Use case         │ High scale      │ Distributed, web │
└──────────────────┴─────────────────┴──────────────────┘
*Snowflake monotonic only per-machine, not globally
```

**Пример.**
```python
import os
import time
from datetime import datetime

class ULID:
    """ULID generator — no machine ID needed, no coordination"""

    # Crockford's Base32 alphabet (excludes I, L, O, U)
    ENCODING = "0123456789ABCDEFGHJKMNPQRSTVWXYZ"

    @classmethod
    def generate(cls) -> str:
        """Generate new ULID"""
        timestamp_ms = int(time.time() * 1000)
        randomness = os.urandom(10)  # 80 bits of randomness

        return cls._encode(timestamp_ms, randomness)

    @classmethod
    def _encode(cls, timestamp_ms: int, randomness: bytes) -> str:
        """Encode timestamp + randomness into ULID string"""

        # Encode 48-bit timestamp into 10 Crockford Base32 characters
        timestamp_chars = []
        ts = timestamp_ms
        for _ in range(10):
            timestamp_chars.append(cls.ENCODING[ts & 0x1F])  # 5 bits per char
            ts >>= 5
        timestamp_str = ''.join(reversed(timestamp_chars))

        # Encode 80-bit randomness into 16 Crockford Base32 characters
        randomness_int = int.from_bytes(randomness, 'big')
        randomness_chars = []
        for _ in range(16):
            randomness_chars.append(cls.ENCODING[randomness_int & 0x1F])
            randomness_int >>= 5
        randomness_str = ''.join(reversed(randomness_chars))

        return timestamp_str + randomness_str

    @classmethod
    def parse(cls, ulid_str: str) -> dict:
        """Parse ULID into components"""
        if len(ulid_str) != 26:
            raise ValueError(f"ULID must be 26 characters, got {len(ulid_str)}")

        # Decode timestamp (first 10 chars)
        timestamp = 0
        for char in ulid_str[:10]:
            index = cls.ENCODING.index(char.upper())
            if index < 0:
                raise ValueError(f"Invalid ULID character: {char}")
            timestamp = (timestamp << 5) | index

        return {
            "ulid": ulid_str,
            "timestamp_ms": timestamp,
            "datetime": datetime.fromtimestamp(timestamp / 1000),
            "randomness": ulid_str[10:],
        }

    @classmethod
    def is_monotonic_increase(cls, prev_ulid: str, next_ulid: str) -> bool:
        """Check if next_ulid > prev_ulid (for monotonic counters)"""
        # Lexicographic comparison works because encoding preserves ordering
        return next_ulid > prev_ulid

# Usage:
print("=" * 50)
print("ULID Examples")
print("=" * 50)

for i in range(5):
    ulid = ULID.generate()
    parsed = ULID.parse(ulid)
    print(f"ULID {i+1}: {ulid}")
    print(f"  Timestamp: {parsed['datetime']}")
    print(f"  Random: {parsed['randomness']}\n")

# Test ordering
print("\nTesting lexicographic ordering:")
ulids = sorted([ULID.generate() for _ in range(3)])
print(f"ULIDs in order: {ulids}")
print("All ULIDs sorted? Yes (lexicographic ordering preserved)")

# Test monotonicity
print("\nTesting monotonic generation:")
class MonotonicULID(ULID):
    """ULID with monotonic counter for same timestamp"""
    def __init__(self):
        self.last_timestamp = -1
        self.counter = 0

    def generate_monotonic(self) -> str:
        timestamp_ms = int(time.time() * 1000)

        if timestamp_ms == self.last_timestamp:
            # Same timestamp — increment counter
            self.counter += 1
            if self.counter >= (2**80):  # Overflow protection
                raise ValueError("Monotonic counter overflow")
        else:
            # New timestamp — reset counter
            self.counter = 0

        self.last_timestamp = timestamp_ms

        # Create deterministic randomness from counter
        randomness = self.counter.to_bytes(10, 'big')
        return self._encode(timestamp_ms, randomness)

mono = MonotonicULID()
prev = None
for i in range(5):
    ulid = mono.generate_monotonic()
    if prev and ulid <= prev:
        print(f"NOT monotonic! {ulid} <= {prev}")
    else:
        print(f"ULID {i+1}: {ulid} (monotonic)")
    prev = ulid
```

**Преимущества ULID:**
```
✓ No coordination needed — no machine ID assignment
✓ Very sortable — lexicographic ordering
✓ 26 characters — human-readable, URL-safe, copy-pasteable
✓ Case-insensitive — can use uppercase or lowercase
✓ Monotonic option — can implement monotonic counters
✓ High-uniqueness — 2^80 random per millisecond
✓ Timestamp-extractable — can get generation time
✓ Future-proof — 8,900+ years until timestamp overflow
```

**Недостатки ULID:**
```
✗ 128 bits — в 2 раза больше чем Snowflake
✗ String encoding overhead — требует base32 en/de-coding
✗ Less efficient in databases — 26 char string vs 8 byte binary
✗ Storage — 26 bytes как string (vs 8 bytes для Snowflake)
✗ Index performance — lexicographic comparison медленнее чем numeric
✗ Newer standard — less established than Snowflake
```

**Пример использования в системе:**
```python
# Web service using ULID
class UserService:
    async def create_user(self, email: str) -> str:
        """Create user with ULID as ID"""
        user_id = ULID.generate()  # No machine ID needed!

        user = User(
            id=user_id,
            email=email,
            created_at=datetime.utcnow()
        )

        await self.db.insert(user)
        return user_id

# REST API — ULID is great for URLs
@app.get("/users/{user_id}")
async def get_user(user_id: str):
    """Get user by ULID"""
    # ULID is URL-safe, case-insensitive
    ulid = ULID.parse(user_id)
    user = await db.get_user(user_id)
    return user

# Storage layer — can use ULID as both binary and string
class ULIDStorage:
    async def get(self, ulid_str: str):
        """Retrieve by ULID string"""
        # Can store as:
        # 1. Binary (16 bytes) — more efficient
        # 2. String (26 chars) — human readable
        return await self.storage.get(ulid_str)
```

**Типичные ошибки.**
- Забыть что ULID требует 16 bytes storage (vs 8 для Snowflake)
- Использовать ULID когда нужна максимальная производительность (Snowflake быстрее)
- Предположить что ULID автоматически monotonic (нужно реализовать специальную логику)
- Забыть что random part может привести к коллизии (очень редко, но возможна)

**На интервью.**
- Объясни ULID структуру и почему base32 encoding
- Покажи сравнение с Snowflake и когда выбрать что
- Объясни lexicographic ordering и как это помогает sortability
- Обсуди implementation complexity vs benefits
- Уточняющий вопрос: как реализовать monotonic ULID? (answer: используй counter с timestamp)

---

### 5. Какие проблемы возникают с синхронизацией часов в Snowflake?

**Зачем спрашивают.** Это ключевой technical deep-dive. Snowflake зависит от часов на каждом узле — если часы рассинхронизированы, может быть mess с ID ordering и даже дубликаты. Интервьюер проверяет понимание distributed systems challenges.

**Короткий ответ.** Clock skew (разница в часах между узлами) и clock drift (часы движутся быстрее/медленнее) создают проблемы: (1) Non-monotonic IDs — если часы прыгают назад, новые IDs будут меньше старых; (2) Potential duplicates — если машина генерировала IDs с timestamp T, потом часы сдвинулись назад к T, может быть collision. Решение: NTP sync, допустить небольшой backward drift (5-10ms), обработать edge case clock reset.

**Детальный разбор.**

**Clock skew scenarios:**
```
Ideal scenario (all clocks synchronized):

Machine 1: T=1000ms ─────────► T=1001ms ─────────► T=1002ms
           ID(42,0)           ID(42,0)            ID(42,0)
                              ↓seq=1              ↓seq=2

Machine 2: T=1000ms ─────────► T=1001ms ─────────► T=1002ms
           ID(43,0)           ID(43,0)            ID(43,0)
                              ↓seq=1              ↓seq=2

Result:
1000ms: IDs from both machines available
1001ms: Both machines generate new IDs (no conflict)
1002ms: Timestamp advances normally

─────────────────────────────────────────────────────────

Проблема: Часы машины 1 идут назад (clock skew):

Машина 1: T=1000ms ─────► T=1001ms ──► Часы прыгнули назад!
          ID(42,0)       ID(42,0)      T=995ms ← ОШИБКА!
                         ↓seq=1        T < last_timestamp
                                       Невозможно генерировать ID

Машина 2: T=1000ms ─────► T=1001ms ──► T=1002ms
          ID(43,0)       ID(43,0)      ID(43,0)
                         ↓seq=1        ↓seq=2

Результат:
- Машина 1 перестаёт генерировать ID (ClockMovedBackwardError)
- Доступность системы деградирована
- Несогласованная генерация ID

─────────────────────────────────────────────────────────

Проблема: Часы машины 1 отстают (слишком медленно):

Машина 1: T=1000ms ─────► T=1000ms ─────► T=1001ms
          ID(42,0)       ID(42,0)        ID(42,0)
                         ↓seq=1
                         ↓seq=2          ↓seq=3
                         ... переполнение seq...

Машина 2: T=1000ms ─────► T=1001ms ─────► T=1002ms
          ID(43,0)       ID(43,0)        ID(43,0)
                         ↓seq=1          ↓seq=1

Результат:
- Машина 1 генерирует больше ID за один временной шаг (переполнение seq чаще)
- Должна ждать следующую ms (backpressure)
- Пропускная способность снижена на затронутой машине

─────────────────────────────────────────────────────────

Проблема: Дублирование если часы сбросятся:

Машина 1: T=1000ms → T=1001ms → ... → T=1100ms
          ╱    \
          │ КРАХ И ПЕРЕЗАГРУЗКА
          │ Загрузить last_timestamp из checkpoint: T=1050ms (старая!)
          │
          └─► T=1030ms (после перезагрузки) — ниже чем checkpoint!
              Невозможно генерировать (часы назад с 1050 на 1030)

НО если машина не проверяет:
          T=1030ms → генерировать ID с timestamp 1030
          НО предыдущий ID уже существует с тем же timestamp!
          КОЛЛИЗИЯ!
```

**Clock synchronization approach:**
```
┌─────────────────────────────────────────────────────┐
│      Distributed System with NTP Sync               │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Internet Time Servers (NTP pool.ntp.org)          │
│           │         │         │                     │
│           ▼         ▼         ▼                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  NTP Client (daemon on each node)           │   │
│  │  - Syncs every 64 seconds                   │   │
│  │  - Corrects clock drift                     │   │
│  │  - ±10ms typical accuracy                   │   │
│  └────────────┬────────────────────────────────┘   │
│               │                                     │
│  ┌────────────▼────────────┐                       │
│  │ System Clock            │                       │
│  │ ~99% accuracy           │                       │
│  │ ±100ms max drift        │                       │
│  └────────────┬────────────┘                       │
│               │                                     │
│  ┌────────────▼────────────┐                       │
│  │ Snowflake ID Generator  │                       │
│  │ Uses system clock       │                       │
│  └─────────────────────────┘                       │
│                                                      │
└─────────────────────────────────────────────────────┘

Типичная точность NTP:
- ±10ms в одном датацентре
- ±100ms между регионами
- Должно учитываться в проектировании Snowflake
```

**Пример.**
```python
import time
import threading
from typing import Optional

class ClockAwareSnowflake:
    """Snowflake with clock drift handling"""

    EPOCH = 1704067200000
    MACHINE_BITS = 10
    SEQUENCE_BITS = 12

    MAX_MACHINE_ID = (1 << MACHINE_BITS) - 1
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1

    # Clock drift tolerance: allow 5ms backward drift
    MAX_BACKWARD_MS = 5

    def __init__(self, machine_id: int):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()
        self.clock_errors = []  # Track clock issues for monitoring

    def generate(self) -> int:
        """Generate ID with clock drift handling"""
        with self.lock:
            timestamp = self._current_time_ms()

            if timestamp < self.last_timestamp:
                drift = self.last_timestamp - timestamp
                self.clock_errors.append({
                    'type': 'backward',
                    'drift_ms': drift,
                    'time': datetime.now()
                })

                if drift <= self.MAX_BACKWARD_MS:
                    # Small drift — wait it out
                    print(f"⚠️  Clock drift {drift}ms — waiting...")
                    time.sleep(drift / 1000.0)
                    timestamp = self._current_time_ms()
                else:
                    # Large drift — fail (alert ops!)
                    print(f"❌ CRITICAL: Clock moved back {drift}ms!")
                    raise ClockMovedBackwardError(
                        f"Clock backwards by {drift}ms (tolerance: {self.MAX_BACKWARD_MS}ms)"
                    )

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE
                if self.sequence == 0:
                    timestamp = self._wait_next_millis(timestamp)
            else:
                self.sequence = 0

            self.last_timestamp = timestamp

            id = (
                (timestamp - self.EPOCH) << (self.MACHINE_BITS + self.SEQUENCE_BITS) |
                (self.machine_id << self.SEQUENCE_BITS) |
                self.sequence
            )

            return id

    def _current_time_ms(self) -> int:
        return int(time.time() * 1000)

    def _wait_next_millis(self, last_timestamp: int) -> int:
        ts = self._current_time_ms()
        while ts <= last_timestamp:
            time.sleep(0.001)  # Sleep 1ms and retry
            ts = self._current_time_ms()
        return ts

    def get_health(self) -> dict:
        """Monitor clock health"""
        return {
            'machine_id': self.machine_id,
            'last_timestamp': self.last_timestamp,
            'clock_errors': len(self.clock_errors),
            'recent_errors': self.clock_errors[-5:]  # Last 5 errors
        }

# Пример: Simulation with clock issues
class SimulatedClockGenerator:
    """For testing clock handling"""

    def __init__(self, machine_id: int):
        self.gen = ClockAwareSnowflake(machine_id)
        self.fake_time = int(time.time() * 1000)

    def simulate_clock_skip(self, skip_ms: int):
        """Simulate clock jumping"""
        self.fake_time += skip_ms
        print(f"Simulating clock skip: +{skip_ms}ms")

    def simulate_clock_backward(self, backward_ms: int):
        """Simulate clock going backward"""
        self.fake_time -= backward_ms
        print(f"Simulating clock backward: -{backward_ms}ms")

# Test:
print("Testing clock drift handling:")
gen = ClockAwareSnowflake(machine_id=1)

# Normal generation
for i in range(3):
    id = gen.generate()
    print(f"ID {i+1}: {id}")

print("\nTesting clock backward (small drift):")
# Simulate small backward drift
import unittest.mock as mock
with mock.patch.object(gen, '_current_time_ms', return_value=1000):
    try:
        id = gen.generate()  # Will wait and retry
        print(f"Generated ID despite drift: {id}")
    except Exception as e:
        print(f"Error: {e}")

print("\nClock health:", gen.get_health())
```

**Решения для clock sync:**
```
1. NTP (Network Time Protocol)
   - Syncs system clock with internet time servers
   - Typical accuracy: ±10ms
   - Automatic (daemon runs continuously)
   - OS-level synchronization
   - Deploy ntpd or systemd-timesyncd

2. Tolerate small backward drift
   - Allow ±5-10ms backward drift
   - Wait and retry if drift is small
   - Fail loudly if drift is large (ops should fix)

3. Monotonic clock
   - Use CLOCK_MONOTONIC instead of wall-clock time
   - Guarantees: never goes backward
   - But: not suitable for Snowflake (can't compare across reboots)

4. Hybrid approach
   - Use monotonic clock for duration (relative time)
   - Use wall-clock for timestamp (absolute time)
   - Combine: timestamp + monotonic offset

5. Manual timestamp checkpointing
   - Periodically save last_timestamp to disk
   - On restart, check if current time < saved timestamp
   - Prevent duplicate ID generation

6. Clock rate limiting
   - If clock drifts too fast, slow down ID generation
   - Backpressure: wait for clock to catch up
   - Protects against sequence overflow
```

**Типичные ошибки.**
- Игнорировать clock skew (предположить часы всегда синхронизированы)
- Не обрабатывать backward drift (приводит к ClockMovedBackwardError, downtime)
- Сохранять последний timestamp только в памяти (после перезагрузки может быть дублирование)
- Использовать `time.time()` вместо `clock.monotonic()` (не защищает от skew)
- Не мониторить clock health (дрейф может быть незаметен пока не накопится)

**На интервью.**
- Объясни как clock skew влияет на Snowflake ID ordering
- Опиши scenarios когда часы идут назад (NTP correction, daylight saving, leap second)
- Покажи как реализовать tolerance к small drift
- Обсуди checkpointing и monitoring для clock health
- Упомяни NTP accuracy в разных сценариях (LAN, WAN, cross-DC)
- Уточняющий вопрос: как обрабатывать clock resets? (answer: checkpoint last_timestamp, check on startup)

---

### 6. Как обеспечить монотонность ID в системе с несколькими узлами?

**Зачем спрашивают.** Monotonicity — сложное требование: IDs должны всегда расти. Но Snowflake не гарантирует глобальную монотонность (разные машины могут генерировать IDs не в порядке). Интервьюер проверяет понимание trade-off между простотой и guarantees.

**Короткий ответ.** Полная глобальная монотонность в распределённой системе очень дорога (требует координации). Лучше per-node monotonicity (IDs от одной машины всегда растут) + timestamp-based ordering (IDs с более поздним timestamp > IDs с более ранним). Если нужна строгая глобальная монотонность — используй DB sequence (но теряешь масштабируемость) или реализуй координированный counter (bottleneck).

**Детальный разбор.**

**Что такое монотонность:**
```
Monotonic increasing:
ID_1 ≤ ID_2 ≤ ID_3 ≤ ...
(newer ID > older ID always)

Guarantees vary:

1. Per-node monotonicity:
   Machine A: 100 < 101 < 102 < ... ✓
   Machine B: 200 < 201 < 202 < ... ✓
   But: Machine A: 100, Machine B: 50 (not globally monotonic)

2. Timestamp-based monotonicity:
   If ID_x.timestamp < ID_y.timestamp → ID_x < ID_y ✓
   But: same timestamp from different machines might not be ordered

3. Strict global monotonicity:
   Every ID_i < ID_{i+1}, regardless of source ✓✓✓
   Most expensive — requires central coordinator

4. Causal monotonicity:
   If event A causes event B → ID_A < ID_B
   Requires logical timestamps or vector clocks
   Research topic (usually overkill for ID generation)
```

**Snowflake monotonicity:**
```
Per-machine monotonicity:

Machine ID 1:
T=1000ms, seq=0 → ID = 0x...0001000 = 4096
T=1000ms, seq=1 → ID = 0x...0001001 = 4097
T=1001ms, seq=0 → ID = 0x...0001001 << 1 = 8193
                                    ↑
                                    Monotonic! (4096 < 4097 < 8193)

Machine ID 2 at same time:
T=1000ms, seq=0 → ID = 0x...0002000 = 8192
T=1000ms, seq=1 → ID = 0x...0002001 = 8193
                                    ↑
                                    Also monotonic within machine

Global comparison:
Machine 1 at 1000ms: 4096
Machine 2 at 1000ms: 8192
                     ↑
                     Not guaranteed order! (depends on machine ID)

But if ordered by timestamp first:
IF T_A < T_B → ID_A < ID_B (always true in Snowflake!)
IF T_A = T_B → depends on machine_id and sequence
(could be A > B even if A generated before B globally)
```

**Пример: Способы реализовать монотонность:**
```python
# Approach 1: Per-node monotonicity (Snowflake, no coordination)
class PerNodeMonotonicGenerator:
    """Guaranteed monotonic per node, not globally"""

    def __init__(self, machine_id: int):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def generate(self) -> int:
        with self.lock:
            timestamp = self._current_time_ms()

            if timestamp < self.last_timestamp:
                raise ClockMovedBackwardError()

            if timestamp == self.last_timestamp:
                self.sequence += 1
            else:
                self.sequence = 0

            self.last_timestamp = timestamp

            # This ID is guaranteed > previous ID from same machine
            id = (timestamp << 22) | (self.machine_id << 12) | self.sequence
            return id

    def _current_time_ms(self):
        return int(time.time() * 1000)

# Usage:
gen = PerNodeMonotonicGenerator(machine_id=1)
prev_id = 0
for i in range(5):
    id = gen.generate()
    assert id > prev_id, f"Not monotonic: {prev_id} >= {id}"
    print(f"ID {i+1}: {id} ✓")
    prev_id = id

───────────────────────────────────────────────────────────

# Approach 2: Global monotonicity via sequential numbering (DB)
class GlobalMonotonicGenerator:
    """Strict global monotonicity — requires central coordinator"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.counter_key = "global:id:counter"

    async def generate(self) -> int:
        # Sequential allocation — guarantee monotonic
        # But: network roundtrip required, bottleneck!
        next_id = await self.redis.incr(self.counter_key)
        return next_id

# Issues:
# - Every ID generation needs network call to Redis
# - ~1-5ms latency per ID (not < 1ms requirement)
# - Redis becomes bottleneck at 200K IDs/sec (max Redis throughput)
# - Cannot offline-generate IDs
# - SPOF if Redis goes down

───────────────────────────────────────────────────────────

# Approach 3: Hybrid — local monotonic + logical timestamp
class HybridMonotonicGenerator:
    """Per-node monotonic + logical ordering"""

    def __init__(self, machine_id: int, datacenter_id: int):
        self.machine_id = machine_id
        self.datacenter_id = datacenter_id
        self.logical_clock = 0  # Lamport clock
        self.lock = threading.Lock()

    def generate(self) -> int:
        with self.lock:
            # Lamport clock: advance time based on system clock or previous IDs
            system_time = int(time.time() * 1000)

            # Logical clock always advances
            self.logical_clock = max(self.logical_clock + 1, system_time)

            # Create ID with logical timestamp
            # Format: (logical_clock << 22) | (datacenter << 10) | machine
            id = (self.logical_clock << 20) | (self.datacenter_id << 10) | self.machine_id

            return id

# Advantage: locally increasing even with clock issues
# Disadvantage: if logical_clock drifts from reality, IDs become "future-dated"

───────────────────────────────────────────────────────────

# Approach 4: Timestamp + Machine ID ordering
class TimestampOrderedGenerator:
    """Monotonic by (timestamp, machine_id, sequence)"""

    def __init__(self, machine_id: int):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1

    def generate(self) -> int:
        timestamp = int(time.time() * 1000)

        if timestamp == self.last_timestamp:
            self.sequence += 1
        else:
            self.sequence = 0

        self.last_timestamp = timestamp

        # Ordering: timestamp >> machine_id >> sequence
        # Guarantees: ID_x < ID_y if timestamp_x < timestamp_y
        id = (timestamp << 20) | (self.machine_id << 10) | self.sequence
        return id

# When you sort by ID, you get chronological order (approximately)
# This is what Snowflake provides naturally
```

**Монотонность в разных сценариях:**
```
┌────────────────────┬──────────────────┬──────────────────┬────────────────┐
│ Requirement        │ Solution         │ Coordination     │ Trade-off      │
├────────────────────┼──────────────────┼──────────────────┼────────────────┤
│ Per-node only      │ Snowflake        │ None             │ Simple         │
│ Timestamp-ordered  │ Snowflake        │ NTP              │ Good default   │
│ Strict global      │ DB sequence      │ Full (SPOF)      │ Not scalable   │
│ Strict global      │ Raft consensus   │ Heavy            │ Complex        │
│ Logical ordering   │ Lamport clock    │ Light (piggybk)  │ OK for events  │
└────────────────────┴──────────────────┴──────────────────┴────────────────┘
```

**Пример использования в практической системе:**
```python
# Real-world: Event logging with monotonic per-partition

class EventIDGenerator:
    """Generate IDs for events — monotonic per partition"""

    def __init__(self, partition_id: int):
        self.partition_id = partition_id  # e.g., Kafka partition
        self.last_sequence = 0
        self.last_timestamp = 0
        self.lock = threading.Lock()

    def generate_for_event(self, event_timestamp: int) -> int:
        """Generate monotonically increasing ID for event"""
        with self.lock:
            # If event has newer timestamp, reset sequence
            if event_timestamp > self.last_timestamp:
                self.last_timestamp = event_timestamp
                self.last_sequence = 0
            elif event_timestamp == self.last_timestamp:
                self.last_sequence += 1
            else:
                # Event from past? Log warning, use last_timestamp
                print(f"⚠️  Event timestamp {event_timestamp} < last {self.last_timestamp}")
                self.last_sequence += 1

            # Format: (timestamp << 16) | (partition << 8) | sequence
            id = (self.last_timestamp << 16) | (self.partition_id << 8) | self.last_sequence
            return id

# Usage:
gen = EventIDGenerator(partition_id=0)

events = [
    {"data": "event1", "timestamp": 1000},
    {"data": "event2", "timestamp": 1000},  # Same timestamp
    {"data": "event3", "timestamp": 1001},
    {"data": "event4", "timestamp": 1001},
]

ids = []
for event in events:
    id = gen.generate_for_event(event["timestamp"])
    ids.append(id)
    print(f"Event {event['data']}: ID={id}")

# Verify monotonicity:
print("\nMonotonicity check:")
for i in range(len(ids) - 1):
    assert ids[i] <= ids[i+1], f"Not monotonic: {ids[i]} > {ids[i+1]}"
    print(f"  ID[{i}]={ids[i]} ≤ ID[{i+1}]={ids[i+1]} ✓")
```

**Типичные ошибки.**
- Требовать строгую глобальную монотонность (очень дорого, оккупирует ресурсы)
- Забыть что NTP sync не может гарантировать строгую монотонность (clock skew всегда существует)
- Реализовать координированный counter без кэширования (становится bottleneck)
- Использовать DB sequence для 100K IDs/sec (физически невозможно)
- Предположить что timestamp-ordering достаточно монотонности (часто OK, но не гарантировано)

**На интервью.**
- Объясни разницу между per-node и global monotonicity
- Покажи как Snowflake обеспечивает per-node monotonicity
- Обсуди cost of strict global monotonicity (centralization, bottleneck)
- Покажи примеры как использовать timestamp для ordering
- Уточняющий вопрос: Что если timestamp одинаковый для разных машин? (answer: используй machine_id as tiebreaker)

---

### 7. Как предотвратить коллизии при генерации распределённых ID?

**Зачем спрашивают.** Collision prevention — основной requirement. Разные подходы имеют разные гарантии. Интервьюер проверяет понимание математики вероятностей и трейд-офов.

**Короткий ответ.** Коллизии предотвращаются несколькими механизмами: (1) Snowflake — уникальная машина ID + sequence per ms (4K IDs per ms) = 0 коллизий; (2) UUID — 128 bits random (birthday paradox ~2.7×10^18 для 50% коллизии); (3) ULID — 48 bits timestamp + 80 bits random; (4) Database sequence — atomic counter (0 коллизий, но единая точка отказа). Выбор зависит от throughput требования и требуемого уровня гарантий.

**Детальный разбор.**

**Математика коллизий (Birthday Paradox):**
```
For n random values with k possible values:
Probability of collision ≈ n^2 / (2k)

Examples:

64-bit ID (2^64 possible values):
- After 1B IDs: P(collision) ≈ (10^9)^2 / (2 × 2^64) ≈ 2.7 × 10^-10 (0.000000027%)
- After 1T IDs: P(collision) ≈ (10^12)^2 / (2 × 2^64) ≈ 2.7 × 10^-4 (0.027%)
- After 4.3B IDs: P(collision) ≈ 50%

128-bit ID (2^128 possible values):
- After 1B IDs: P(collision) ≈ negligible (< 10^-30)
- After 1T IDs: P(collision) ≈ negligible (< 10^-15)
- After 10^19 IDs: P(collision) ≈ 0.01%

Practical implications:
- 64 bits sufficient for 10B+ IDs if collision rate < 0.1%
- 128 bits "safe" for any practical scale (UUID v4)
- Snowflake (64-bit) uses machine ID + timestamp to prevent collisions
```

**Snowflake: Collision prevention by structure:**
```
64-bit Snowflake:
┌─────┬──────────────┬──────────┬────────────┐
│sign │  timestamp   │ machine  │  sequence  │
│ 1   │    41        │    10    │    12      │
└─────┴──────────────┴──────────┴────────────┘

Collision scenario 1: Same machine, same ms
  Machine 1, ms=1000, seq=0 → ID_1
  Machine 1, ms=1000, seq=1 → ID_2 (seq prevents collision!)
  Maximum: seq can be 0-4095 (4096 values)

  Collision happens only if:
  - Try to generate > 4096 IDs in same ms from same machine
  - Solution: wait for next ms, or fail (backpressure)

Collision scenario 2: Different machines, same ms
  Machine 1, ms=1000, seq=0 → (timestamp <<22) | (1 << 12) | 0
  Machine 2, ms=1000, seq=0 → (timestamp <<22) | (2 << 12) | 0
                                           ↑ different!
  No collision! (machine_id ensures uniqueness)

Collision scenario 3: Machine ID collision
  If two machines get same machine_id → COLLISION!
  Solution: ensure machine_id uniqueness via ZK/etcd

Mathematical guarantee:
  P(collision) = 0 IF:
  1. All machine IDs are unique (0-1023)
  2. All timestamps are synchronized (no skew > precision)
  3. Sequence never overflows (handled: wait for next ms)

Worst case: >4096 IDs/ms from one machine
  Machine 1: millisecond 1000
    - Generate ID 0-4095 (ok)
    - Try to generate 4096th ID (seq overflow)
    - Solution: must wait for ms=1001
    - Backpressure: sleep ~1ms, generate in 1001
    - No collision, but slight delay
```

**Таблица сравнения коллизий:**
```
┌────────────────┬──────────────┬──────────────────┬──────────────────┐
│ Method         │ Collision    │ Probability      │ Requirements     │
│                │ Guarantee    │ (1B IDs)         │                  │
├────────────────┼──────────────┼──────────────────┼──────────────────┤
│ Snowflake      │ 0 (math)     │ 0%               │ Unique machine   │
│                │              │ (if followed)    │ IDs, NTP sync    │
├────────────────┼──────────────┼──────────────────┼──────────────────┤
│ UUID v4        │ ~2.7×10^-10  │ 0.000000027%     │ Good RNG         │
│ (128-bit rand) │              │ (negligible)     │                  │
├────────────────┼──────────────┼──────────────────┼──────────────────┤
│ ULID           │ ~1×10^-30    │ negligible       │ None             │
│ (48+80 bits)   │              │ (48bits time +   │                  │
│                │              │ 80bits random)   │                  │
├────────────────┼──────────────┼──────────────────┼──────────────────┤
│ DB sequence    │ 0 (atomic)   │ 0%               │ Central coord    │
│                │              │ (guaranteed)     │ (SPOF)           │
├────────────────┼──────────────┼──────────────────┼──────────────────┤
│ Timestamp+     │ depends      │ low if diff      │ Synced clocks    │
│ random         │ on bits      │ > collision      │                  │
│                │ allocated    │                  │                  │
└────────────────┴──────────────┴──────────────────┴──────────────────┘
```

**Пример: Collision detection и handling:**
```python
from collections import defaultdict
import time

class CollisionDetector:
    """Detect and handle ID collisions in production"""

    def __init__(self):
        self.seen_ids = set()
        self.collision_count = 0
        self.collisions = defaultdict(list)

    def check(self, id: int, source: str):
        """Check if ID was seen before"""
        if id in self.seen_ids:
            self.collision_count += 1
            self.collisions[id].append(source)
            print(f"❌ COLLISION: ID {id} already seen from {self.collisions[id]}")
            return False

        self.seen_ids.add(id)
        return True

    def report(self):
        """Report collision statistics"""
        print(f"Total collisions: {self.collision_count}")
        print(f"Unique IDs: {len(self.seen_ids)}")
        print(f"Collision rate: {self.collision_count / len(self.seen_ids) if self.seen_ids else 0:.6f}")


# Test Snowflake for collisions
class ThreadedSnowflakeTest:
    def __init__(self, num_machines: int, num_threads_per_machine: int, ids_per_thread: int):
        self.num_machines = num_machines
        self.num_threads_per_machine = num_threads_per_machine
        self.ids_per_thread = ids_per_thread
        self.detector = CollisionDetector()
        self.generators = [SnowflakeIDGenerator(i) for i in range(num_machines)]

    def run(self):
        """Generate IDs from multiple threads and check for collisions"""
        import concurrent.futures

        def generate_batch(machine_id: int, thread_id: int):
            generator = self.generators[machine_id]
            for i in range(self.ids_per_thread):
                id = generator.generate()
                self.detector.check(id, f"machine {machine_id}, thread {thread_id}")

        with concurrent.futures.ThreadPoolExecutor(max_workers=self.num_machines * self.num_threads_per_machine) as executor:
            futures = []
            for machine_id in range(self.num_machines):
                for thread_id in range(self.num_threads_per_machine):
                    future = executor.submit(generate_batch, machine_id, thread_id)
                    futures.append(future)

            concurrent.futures.wait(futures)

        self.detector.report()

# Run collision test
print("Testing Snowflake for collisions:")
print("5 machines × 10 threads per machine × 1000 IDs per thread")
tester = ThreadedSnowflakeTest(num_machines=5, num_threads_per_machine=10, ids_per_thread=1000)
start = time.time()
tester.run()
elapsed = time.time() - start
print(f"Completed in {elapsed:.2f}s")
# Expected: 0 collisions
```

**Типичные ошибки.**
- Не гарантировать уникальность machine ID (приводит к коллизиям!)
- Использовать слишком маленький size ID для scale (birthday paradox)
- Забыть про sequence overflow handling (может привести к дубликатам)
- Не тестировать collision rate in production (неожиданный spike)
- Предположить что random RNG достаточна (проверить quality RNG!)

**На интервью.**
- Объясни birthday paradox и как рассчитать вероятность коллизии
- Покажи как Snowflake структура предотвращает коллизии (machine_id + sequence)
- Обсуди требования к уникальности machine_id
- Упомяни как обрабатывать sequence overflow
- Уточняющий вопрос: Что если слишком много коллизий? (answer: используй больший ID размер или измени алгоритм)

---

### 8. Как кодировать и декодировать ID разных форматов?

**Зачем спрашивают.** ID часто используются в URLs, логах, API ответах. Разные форматы имеют разные свойства (binary vs string, sortability, URL-safe и т.д.). Интервьюер проверяет практическое знание en/de-coding и его complications.

**Короткий ответ.** ID кодируются в разные форматы: (1) Binary — компактно (8 bytes для Snowflake), (2) Decimal — просто, человеко-читаемо, (3) Hex — стандарт для binary, (4) Base62 — compact string (10-11 chars для Snowflake), (5) Base32 (Crockford) — case-insensitive, URL-safe (26 chars для ULID). Выбор зависит от use case: storage vs transmission vs user-facing.

**Детальный разбор.**

**Форматы кодирования ID:**
```
Same Snowflake ID: timestamp=1704067200000, machine=42, seq=1234
Binary value: 11110101001101010110101100111100101010

Different encodings:

1. Binary (8 bytes):
   Raw bytes: F5 35 6B 73 CA (little-endian) / F5356B3CAA (hex)
   Size: 8 bytes
   Storage efficient ✓✓✓
   Human readable: ✗
   URL safe: ✗
   Use: Database storage, network packets

2. Decimal:
   17609893294841386 (19 digits)
   Size: string of 19 characters
   Human readable: ✓ (sort of)
   URL safe: ✓
   Sortable: ✓
   Use: API responses, logging

3. Hexadecimal:
   F5356B3CAA (10-11 hex digits)
   Size: 16 characters (hex) or 8 bytes (binary)
   Human readable: ✓ (for technical people)
   URL safe: ✓
   Sortable: ✓ (lexicographic matches numeric)
   Use: Debugging, internal APIs

4. Base62 (alphanumeric):
   Alphabet: 0-9, a-z, A-Z (62 chars)
   Snowflake: Frr (3 chars) — wait that's only 3?
   Actually need: log62(2^64) = 11 chars max
   Alphabet: 0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
   Use: URL shorteners, user-facing IDs

5. Base32 (Crockford):
   Alphabet: 0123456789ABCDEFGHJKMNPQRSTVWXYZ (no I, L, O, U)
   ULID: 01ARZ3NDEKTSV4RRFFQ69G5FAV (26 chars)
   Case insensitive: ✓
   URL safe: ✓
   Human readable: ✓
   Use: ULID standard, user-friendly IDs

6. Base64 (with URL-safe variant):
   Alphabet: A-Z, a-z, 0-9, +, / (or -, _ for URL-safe)
   Size: 11 chars for 64-bit
   Compact: ✓✓
   Use: Often used in JWT, but not ideal for IDs (need padding)
```

**Пример кодирования:**
```python
import struct
import base64

class IDEncoder:
    """Encode/decode Snowflake IDs in various formats"""

    # Base encoding alphabets
    BASE62 = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    BASE32_CROCKFORD = "0123456789ABCDEFGHJKMNPQRSTVWXYZ"  # No I, L, O, U

    @staticmethod
    def encode_decimal(id: int) -> str:
        """Encode as decimal string"""
        return str(id)

    @staticmethod
    def decode_decimal(s: str) -> int:
        return int(s)

    @staticmethod
    def encode_hex(id: int) -> str:
        """Encode as hexadecimal"""
        return hex(id)[2:]  # Remove '0x' prefix

    @staticmethod
    def decode_hex(s: str) -> int:
        return int(s, 16)

    @staticmethod
    def encode_binary(id: int) -> bytes:
        """Encode as 8-byte binary"""
        return struct.pack('>Q', id)  # Big-endian 64-bit unsigned

    @staticmethod
    def decode_binary(b: bytes) -> int:
        return struct.unpack('>Q', b)[0]

    @staticmethod
    def encode_base62(id: int) -> str:
        """Encode as base62 (compact, alphanumeric)"""
        if id == 0:
            return IDEncoder.BASE62[0]

        chars = []
        while id > 0:
            chars.append(IDEncoder.BASE62[id % 62])
            id //= 62

        return ''.join(reversed(chars))

    @staticmethod
    def decode_base62(s: str) -> int:
        result = 0
        for char in s:
            result = result * 62 + IDEncoder.BASE62.index(char)
        return result

    @staticmethod
    def encode_base32(id: int) -> str:
        """Encode as Crockford's Base32 (case-insensitive, URL-safe)"""
        if id == 0:
            return IDEncoder.BASE32_CROCKFORD[0]

        chars = []
        while id > 0:
            chars.append(IDEncoder.BASE32_CROCKFORD[id % 32])
            id //= 32

        return ''.join(reversed(chars))

    @staticmethod
    def decode_base32(s: str) -> int:
        s = s.upper()  # Case-insensitive
        result = 0
        for char in s:
            result = result * 32 + IDEncoder.BASE32_CROCKFORD.index(char)
        return result

    @staticmethod
    def encode_base64_url(id: int) -> str:
        """Encode as URL-safe Base64"""
        b = struct.pack('>Q', id)
        return base64.urlsafe_b64encode(b).rstrip(b'=').decode('ascii')

    @staticmethod
    def decode_base64_url(s: str) -> int:
        # Add padding
        padding = (8 - len(s) % 8) % 8
        s += '=' * padding
        b = base64.urlsafe_b64decode(s)
        return struct.unpack('>Q', b)[0]

# Testing
id = 17609893294841386

print(f"Original ID: {id}\n")

# Decimal
dec = IDEncoder.encode_decimal(id)
print(f"Decimal:     {dec}")
assert IDEncoder.decode_decimal(dec) == id

# Hex
hex_enc = IDEncoder.encode_hex(id)
print(f"Hex:         {hex_enc}")
assert IDEncoder.decode_hex(hex_enc) == id

# Binary
bin_enc = IDEncoder.encode_binary(id)
print(f"Binary:      {bin_enc.hex()}")
assert IDEncoder.decode_binary(bin_enc) == id

# Base62
b62 = IDEncoder.encode_base62(id)
print(f"Base62:      {b62}")
assert IDEncoder.decode_base62(b62) == id

# Base32 (Crockford)
b32 = IDEncoder.encode_base32(id)
print(f"Base32:      {b32}")
assert IDEncoder.decode_base32(b32) == id

# Base64 URL-safe
b64 = IDEncoder.encode_base64_url(id)
print(f"Base64 URL:  {b64}")
assert IDEncoder.decode_base64_url(b64) == id

print(f"\nSize comparison:")
print(f"  Decimal:    {len(dec)} chars")
print(f"  Hex:        {len(hex_enc)} chars")
print(f"  Binary:     8 bytes")
print(f"  Base62:     {len(b62)} chars")
print(f"  Base32:     {len(b32)} chars")
print(f"  Base64:     {len(b64)} chars")
```

**Выбор кодирования для разных сценариев:**
```
┌──────────────┬─────────────┬─────────┬──────────────┬─────────────┐
│ Use Case     │ Format      │ Size    │ Sortable     │ URL-safe    │
├──────────────┼─────────────┼─────────┼──────────────┼─────────────┤
│ DB Storage   │ Binary      │ 8B      │ Yes          │ N/A         │
│ JSON API     │ Decimal     │ 19ch    │ Yes          │ Yes         │
│ URL ID       │ Base62      │ 10-11ch │ No*          │ Yes         │
│ URL ID       │ Base32      │ 13ch    │ No*          │ Yes         │
│ Logging      │ Hex         │ 16ch    │ Yes          │ Yes         │
│ Cache key    │ Base62      │ 10-11ch │ No*          │ Yes         │
│ JWT claim    │ Decimal     │ 19ch    │ Yes          │ No**        │
│ User input   │ Base32      │ 13ch    │ No*          │ Yes***      │
├──────────────┼─────────────┼─────────┼──────────────┼─────────────┤
* Sortability only if maintaining ordering important
** JSON safe if quoted
*** No confusion (ILOU removed)
```

**Типичные ошибки.**
- Использовать Base64 для ID (требует padding, не URL-safe по умолчанию)
- Забыть про endianness в binary кодировании (Big-endian vs little-endian)
- Не тестировать round-trip (encode → decode)
- Выбрать кодирование не matching use case (например, Base62 в JSON вместо decimal)
- Забыть про sorting properties разных кодировок (decimal sortable, base62 нет)

**На интервью.**
- Покажи как кодировать Snowflake ID в разные форматы
- Объясни trade-off между size и readability
- Обсуди выбор формата для разных сценариев (storage, API, URL)
- Упомяни sortability и как это влияет на индексы
- Уточняющий вопрос: Как интегрировать кодирование в REST API? (answer: automatic in serialization layer)

---

### 9. Как Twitter реализовал Snowflake и какие уроки отсюда?

**Зачем спрашивают.** Twitter Snowflake — реальная production система, которая стала standard. Знание real-world реализации показывает практическое понимание и как это работает в масштабе. Интервьюер проверяет знание деталей и lessons learned.

**Короткий ответ.** Twitter Snowflake был разработан для генерации ID твитов в распределённой системе. Ключевые компоненты: (1) 64-bit ID с timestamp + machine + sequence, (2) ZooKeeper для машины ID assignment, (3) Simple HTTP API для генерации, (4) Per-machine soft state (дешево восстанавливать). Уроки: простота важнее, прямые гарантии лучше чем координация, мониторинг clock health критичен.

**Детальный разбор.**

**Twitter Snowflake архитектура:**
```
┌─────────────────────────────────────────────────────────────┐
│                     Twitter Infrastructure                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────┐                                   │
│  │   Client Service     │ (tweet service, etc.)             │
│  │  (needs ID for tweet)│                                   │
│  └──────────┬───────────┘                                   │
│             │                                               │
│   HTTP GET /id                                              │
│             │                                               │
│  ┌──────────▼────────────────────────────────┐              │
│  │    Snowflake ID Service                   │              │
│  │  ┌────────────────────────────────────┐   │              │
│  │  │ Node 1: machine_id=1               │   │              │
│  │  │ SnowflakeGenerator                 │   │              │
│  │  │ ┌────────────────────────────────┐ │   │              │
│  │  │ │ timestamp: 1704067200000       │ │   │              │
│  │  │ │ sequence: 0-4095               │ │   │              │
│  │  │ │ Clock: synced via NTP          │ │   │              │
│  │  │ └────────────────────────────────┘ │   │              │
│  │  │ Response: 1234567890123456789    │   │              │
│  │  └────────────────────────────────────┘   │              │
│  │  ┌────────────────────────────────────┐   │              │
│  │  │ Node 2: machine_id=2               │   │              │
│  │  │ SnowflakeGenerator                 │   │              │
│  │  │ Response: <independent ID>       │   │              │
│  │  └────────────────────────────────────┘   │              │
│  │  ┌────────────────────────────────────┐   │              │
│  │  │ Node N: machine_id=N               │   │              │
│  │  │ SnowflakeGenerator                 │   │              │
│  │  └────────────────────────────────────┘   │              │
│  └──────────┬────────────────────────────────┘              │
│             │                                               │
│  ┌──────────▼──────────────┐                                │
│  │  ZooKeeper             │                                │
│  │  /snowflake/machine_ids│                                │
│  │  - Ephemeral nodes     │                                │
│  │  - Auto-cleanup on die │                                │
│  │  - < 100ms latency     │                                │
│  └────────────────────────┘                                │
│                                                              │
│  ┌────────────────────────────────┐                        │
│  │  NTP Daemon                    │                        │
│  │  Sync all nodes to UTC         │                        │
│  │  Accuracy: ±10ms               │                        │
│  └────────────────────────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Key decisions:

1. HTTP API over RPC
   Why: Language-agnostic, easy to use, stateless
   Downside: Network latency (but OK for <1ms requirement)

2. Minimal coordination (only machine_id)
   Why: ZK is bottleneck if used for every ID
   Instead: assign once at startup, store locally
   Recover: if crash, request new machine_id from ZK

3. Soft state (machine_id)
   Why: easy to recover from failure
   If node dies:
   - Ephemeral ZK node auto-deleted
   - Another node can take that machine_id
   - OR assign new machine_id, old IDs still valid (but ~5min collision window)

4. NTP synchronization
   Why: Critical for monotonicity
   Action: Twitter runs internal NTP servers
   Policy: automatically restart if clock drift > 1 second
```

**Практическая реализация (simplified):**
```python
# Simplified Twitter Snowflake implementation in Python

import time
import threading
from typing import Optional

class TwitterSnowflake:
    """
    Simplified reproduction of Twitter Snowflake
    Production code would have more robustness
    """

    # Snowflake epoch: November 4, 2010
    TWEPOCH = 1288834974657

    WORKER_ID_BITS = 5
    DATACENTER_ID_BITS = 5
    SEQUENCE_BITS = 12

    MAX_WORKER_ID = (1 << WORKER_ID_BITS) - 1        # 31
    MAX_DATACENTER_ID = (1 << DATACENTER_ID_BITS) - 1  # 31
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1            # 4095

    WORKER_ID_SHIFT = SEQUENCE_BITS
    DATACENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS
    TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS

    def __init__(self, datacenter_id: int, worker_id: int):
        if datacenter_id > self.MAX_DATACENTER_ID or datacenter_id < 0:
            raise ValueError(f"Datacenter ID must be 0-{self.MAX_DATACENTER_ID}")
        if worker_id > self.MAX_WORKER_ID or worker_id < 0:
            raise ValueError(f"Worker ID must be 0-{self.MAX_WORKER_ID}")

        self.datacenter_id = datacenter_id
        self.worker_id = worker_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def generate(self) -> int:
        """Generate Snowflake ID"""
        with self.lock:
            timestamp = int(time.time() * 1000)

            # Handle clock going backwards
            if timestamp < self.last_timestamp:
                # In Twitter production: wait or fail
                # Here: simple wait implementation
                sleep_time = (self.last_timestamp - timestamp) / 1000.0
                time.sleep(sleep_time)
                timestamp = int(time.time() * 1000)

            if timestamp == self.last_timestamp:
                # Same millisecond - increment sequence
                self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE
                if self.sequence == 0:
                    # Sequence overflow - wait for next ms
                    while True:
                        timestamp = int(time.time() * 1000)
                        if timestamp > self.last_timestamp:
                            break
                        time.sleep(0.001)
            else:
                # New millisecond - reset sequence
                self.sequence = 0

            self.last_timestamp = timestamp

            # Assemble the ID
            id = (
                (timestamp - self.TWEPOCH) << self.TIMESTAMP_LEFT_SHIFT |
                self.datacenter_id << self.DATACENTER_ID_SHIFT |
                self.worker_id << self.WORKER_ID_SHIFT |
                self.sequence
            )

            return id

    @staticmethod
    def parse(id: int) -> dict:
        """Parse Snowflake ID"""
        timestamp = (id >> TwitterSnowflake.TIMESTAMP_LEFT_SHIFT) + TwitterSnowflake.TWEPOCH
        datacenter_id = (id >> TwitterSnowflake.DATACENTER_ID_SHIFT) & TwitterSnowflake.MAX_DATACENTER_ID
        worker_id = (id >> TwitterSnowflake.WORKER_ID_SHIFT) & TwitterSnowflake.MAX_WORKER_ID
        sequence = id & TwitterSnowflake.MAX_SEQUENCE

        return {
            "id": id,
            "timestamp_ms": timestamp,
            "datacenter_id": datacenter_id,
            "worker_id": worker_id,
            "sequence": sequence,
        }

# Usage (Twitter style):
generator = TwitterSnowflake(datacenter_id=1, worker_id=1)

# Generate some IDs
ids = []
for i in range(5):
    id = generator.generate()
    ids.append(id)
    parsed = TwitterSnowflake.parse(id)
    print(f"Tweet ID {i+1}: {id}")
    print(f"  Generated at: {time.strftime('%Y-%m-%d %H:%M:%S', time.gmtime(parsed['timestamp_ms']/1000))}")
    print(f"  Datacenter: {parsed['datacenter_id']}, Worker: {parsed['worker_id']}\n")
```

**Уроки от Twitter:**
```
1. Simplicity wins
   ✓ Snowflake is straightforward — 64 bits, 3 components
   ✗ Over-engineered solutions fail in production

2. Coordination is a bottleneck
   ✓ Twitter minimized coordination to machine_id assignment only
   ✗ If you need coordination for every ID, you've failed

3. Soft state is key
   ✓ Machine IDs are soft state (can recover from failure)
   ✗ Hard state (DB, ZK) makes failure recovery expensive

4. Clock synchronization matters
   ✓ Twitter invested heavily in NTP infrastructure
   ✗ Ignoring clock drift leads to ordering issues

5. Monitoring and alerting are critical
   ✓ Twitter monitors: clock drift, sequence overflow, machine ID collisions
   ✗ Without visibility, issues accumulate until catastrophic

6. Binary format is important
   ✓ 64 bits is compact for storage and indexing
   ✗ 128-bit UUID uses 2x storage, 2x bandwidth

7. Stateless generation
   ✓ Each node generates independently, no coordination per ID
   ✗ Centralized generators create bottlenecks

8. Test at scale
   ✓ Twitter tested with millions of IDs/sec before production
   ✗ Assumptions about throughput often wrong at scale
```

**Real-world complications (production lessons):**
```
Issue 1: ZooKeeper latency
  Problem: machine_id assignment was taking 100ms (too slow)
  Solution: Cache machine_id locally, assign once at startup
  Recovery: If restart without saved machine_id, request new one

Issue 2: NTP leap seconds
  Problem: Leap second causes 1-second clock jump
  Solution: Kernel patches to smear leap second over time
  Monitoring: Alert if leap second detected

Issue 3: Datacenter failures
  Problem: If entire DC goes down, machine_ids are lost
  Solution: Re-use machine_ids after sufficient time
  Safety: Only re-assign after 5 minutes with no activity

Issue 4: Sequence overflow under load
  Problem: >4K requests/ms causes overflow
  Solution: Wait for next ms (backpressure)
  Impact: Typical wait 100us, at 1M requests/sec = negligible

Issue 5: ID ordering across datacenters
  Problem: Clock skew in different DCs
  Solution: Twitter accepted ~seconds of skew
  Mitigation: Clients add timestamp to data (double timestamp)
```

**Типичные ошибки при реализации.**
- Забыть про soft state recovery (machine_id loss при перезагрузке)
- Использовать centralized generator вместо distributed (bottleneck!)
- Недостаточное NTP sync (clock drift > milliseconds)
- Не тестировать sequence overflow case
- Игнорировать monitoring и alerting

**На интервью.**
- Объясни Twitter Snowflake архитектуру на высоком уровне
- Покажи как machine_id assignment работает (ZK ephemeral nodes)
- Обсуди critical decisions (HTTP API, soft state, NTP)
- Упомяни lessons learned (simplicity, minimized coordination)
- Уточняющий вопрос: Как бы ты адаптировал Snowflake для своей компании?

---

### 10. Как масштабировать ID generation на несколько датацентров?

**Зачем спрашивают.** Multi-datacenter ID generation — сложная задача. Нужно обрабатывать network partitions, clock skew между регионами, failover. Это grand finale вопроса про ID generation — комплексная система.

**Короткий ответ.** Для масштабирования на несколько датацентров: (1) Allocate machine ID ranges per DC (DC1: 0-99, DC2: 100-199), (2) Replicate ZK для coordin ации, (3) Allow local generation без cross-DC sync, (4) Sync timestamps via NTP, (5) Handle network partitions gracefully (allow temporary machine_id conflicts, resolve later). Результат: high availability, low latency, eventual consistency.

**Детальный разбор.**

**Multi-DC architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│                   Global ID Generation System                    │
│                    (Multi-datacenter)                            │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐        ┌──────────────────────────┐
│   Datacenter US-EAST     │        │   Datacenter EU-WEST     │
│                          │        │                          │
│  ID Generator Nodes      │        │  ID Generator Nodes      │
│  machine_id: 0-99        │        │  machine_id: 100-199     │
│                          │        │                          │
│  ┌────────────────────┐  │        │  ┌────────────────────┐  │
│  │ Node 1: id=1       │  │        │  │ Node 1: id=100     │  │
│  │ Epoch: 2024-01-01  │  │        │  │ Epoch: 2024-01-01  │  │
│  │ NTP sync via local │  │        │  │ NTP sync via local │  │
│  │ NTP server         │  │        │  │ NTP server         │  │
│  └────────────────────┘  │        │  └────────────────────┘  │
│                          │        │                          │
│  ZK Cluster (local)      │        │  ZK Cluster (local)      │
│  - machine_id registry   │        │  - machine_id registry   │
│  - watches for changes   │        │  - watches for changes   │
│  - <10ms latency         │        │  - <10ms latency         │
└──────────────┬───────────┘        └──────────────┬───────────┘
               │                                   │
               │   Cross-DC replication            │
               │   - Async (not on path)           │
               │   - DNS + HTTP for sync           │
               │   - ~100ms latency OK              │
               │                                   │
               └───────────────┬───────────────────┘
                               │
                ┌──────────────▼──────────────┐
                │   Master ZK cluster         │
                │   (quorum across DCs)       │
                │                            │
                │   /snowflake/machines       │
                │   - Authoritative source    │
                │   - Used for conflict       │
                │     resolution              │
                └────────────────────────────┘

ID Generation flow:
1. Client calls GET /id to nearest DC
2. Local ID generator creates ID independently
3. No cross-DC communication needed
4. Async replication of machine_id registry
5. On partition: local generation continues
6. On healing: conflict resolution via master ZK
```

**Machine ID allocation per DC:**
```
Total 1024 machine IDs (10 bits):

┌─────────────────────────────────────────┐
│ Machine ID space allocation             │
├─────────────────────────────────────────┤
│ 0-99:      Datacenter US-EAST           │
│ 100-199:   Datacenter EU-WEST           │
│ 200-299:   Datacenter AP-SOUTHEAST      │
│ 300-399:   Datacenter US-WEST           │
│ 400-999:   Reserved / for future DCs    │
└─────────────────────────────────────────┘

Allocation strategy:

Per-DC:
  DC has say 20 nodes (room for growth)
  Allocate range: 100 IDs per DC
  Reserve 50 for expansion

Example ID interpretation:
  machine_id = 42
  DC = 0 (US-EAST) — within range 0-99

  machine_id = 150
  DC = 1 (EU-WEST) — within range 100-199

  Advantage: can determine DC from ID itself!
```

**Handling network partitions:**
```
Scenario 1: Normal operation (all DCs connected)

┌─────────────────┐         ┌─────────────────┐
│   DC1           │         │   DC2           │
│ Machine IDs     │         │ Machine IDs     │
│ 0-99            │ ◄─────► │ 100-199         │
│ Connected to ZK │         │ Connected to ZK │
│ Can verify ID   │         │ Can verify ID   │
│ ranges          │         │ ranges          │
└─────────────────┘         └─────────────────┘
        │                           │
        └───────────┬───────────────┘
                    │
            Master ZK (quorum)
            [authoritative]

─────────────────────────────────────────

Scenario 2: Network partition (DCs disconnected)

┌─────────────────┐         ┌─────────────────┐
│   DC1           │         │   DC2           │
│ Machine IDs     │         │ Machine IDs     │
│ 0-99            │  ✗✗✗    │ 100-199         │
│ Isolated from ZK│         │ Isolated from ZK│
│ Generates IDs   │         │ Generates IDs   │
│ locally (risky!)│         │ locally (risky!)│
└─────────────────┘         └─────────────────┘
        │                           │
        └─────────┬─────────────────┘
                  │
            Master ZK (UNAVAILABLE)
            Cannot coordinate

Риск: Если оба DC попытаются использовать одинаковый machine_id диапазон
     → КОЛЛИЗИЯ!
```

**Стратегия восстановления:**
```
Во время разделения, разрешить локальную генерацию с рисками:
1. Каждый DC уважает свой выделенный диапазон (0-99 для DC1, и т.д.)
2. Даже при разделении, каждый DC использует только свой диапазон
3. Нет коллизий если диапазоны не перекрываются
4. После восстановления: синхронизировать реестры machine_id

Если кто-то попытается нарушить (использовать неправильный диапазон):
1. Можно обнаружить: DC1 пытается использовать 150 (EU диапазон)
2. Профилактика: Config-based enforcement
3. Режим чрезвычайной ситуации: если обнаружено разделение, более строгие квоты
```

**Пример: Multi-DC Snowflake:**
```python
import asyncio
import time
from typing import Optional
from dataclasses import dataclass

@dataclass
class DatacenterConfig:
    """Configuration for each datacenter"""
    name: str  # e.g., "US-EAST", "EU-WEST"
    machine_id_start: int  # Start of range
    machine_id_end: int    # End of range

class MultiDCSnowflake:
    """Snowflake ID generator for multiple datacenters"""

    EPOCH = 1704067200000
    MACHINE_BITS = 10
    SEQUENCE_BITS = 12

    DC_CONFIGS = {
        'us-east': DatacenterConfig('us-east', 0, 99),
        'eu-west': DatacenterConfig('eu-west', 100, 199),
        'ap-southeast': DatacenterConfig('ap-southeast', 200, 299),
    }

    def __init__(self, dc_name: str, zk_client=None):
        if dc_name not in self.DC_CONFIGS:
            raise ValueError(f"Unknown datacenter: {dc_name}")

        self.dc_config = self.DC_CONFIGS[dc_name]
        self.zk = zk_client  # ZooKeeper client (local to DC)
        self.machine_id: Optional[int] = None
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = asyncio.Lock()
        self.zk_connected = True  # Assume connected initially

    async def acquire_machine_id(self) -> int:
        """Acquire machine_id from ZK (within DC's range)"""
        for candidate in range(self.dc_config.machine_id_start,
                              self.dc_config.machine_id_end + 1):
            path = f"/snowflake/{self.dc_config.name}/machine_{candidate}"
            try:
                # Ephemeral node: auto-deleted on disconnect
                await self.zk.create(
                    path,
                    ephemeral=True,
                    value=f"pid={os.getpid()}"
                )
                self.machine_id = candidate
                print(f"Acquired machine_id={candidate} in {self.dc_config.name}")
                return candidate
            except NodeExistsError:
                continue

        raise ValueError("No available machine IDs in DC range")

    async def generate(self) -> int:
        """Generate ID within DC's allocated range"""
        async with self.lock:
            timestamp = int(time.time() * 1000)

            # Handle ZK disconnection gracefully
            if not self.zk_connected:
                # During partition: still generate locally
                # Risk: if machine_id wasn't acquired, use random ID from range
                if self.machine_id is None:
                    self.machine_id = random.randint(
                        self.dc_config.machine_id_start,
                        self.dc_config.machine_id_end
                    )
                    print(f"⚠️  ZK disconnected, using fallback machine_id={self.machine_id}")

            if timestamp < self.last_timestamp:
                raise ClockMovedBackwardError()

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & ((1 << self.SEQUENCE_BITS) - 1)
                if self.sequence == 0:
                    await self._wait_next_ms(timestamp)
                    timestamp = int(time.time() * 1000)
            else:
                self.sequence = 0

            self.last_timestamp = timestamp

            id = (
                (timestamp - self.EPOCH) << (self.MACHINE_BITS + self.SEQUENCE_BITS) |
                (self.machine_id << self.SEQUENCE_BITS) |
                self.sequence
            )

            return id

    async def _wait_next_ms(self, last_ts):
        await asyncio.sleep(0.001)

    def on_zk_connected(self):
        """Callback when ZK connection established"""
        self.zk_connected = True
        print("ZK connected")

    def on_zk_disconnected(self):
        """Callback when ZK connection lost"""
        self.zk_connected = False
        print("⚠️  ZK disconnected - local generation will continue")

# Usage across datacenters:
async def main():
    # DC1: US-EAST
    gen_us = MultiDCSnowflake('us-east')
    await gen_us.acquire_machine_id()

    # DC2: EU-WEST
    gen_eu = MultiDCSnowflake('eu-west')
    await gen_eu.acquire_machine_id()

    # Generate IDs concurrently
    ids_us = [await gen_us.generate() for _ in range(5)]
    ids_eu = [await gen_eu.generate() for _ in range(5)]

    print(f"IDs from US-EAST: {ids_us}")
    print(f"IDs from EU-WEST: {ids_eu}")

    # Verify no collisions
    all_ids = ids_us + ids_eu
    assert len(all_ids) == len(set(all_ids)), "Collision detected!"
    print("✓ No collisions across DCs")

    # Verify DC can be determined from ID
    for id in all_ids:
        machine_id = (id >> MultiDCSnowflake.SEQUENCE_BITS) & ((1 << MultiDCSnowflake.MACHINE_BITS) - 1)
        dc = 'US-EAST' if machine_id < 100 else 'EU-WEST'
        print(f"ID {id}: generated in {dc}")
```

**Лучшая практикаs для multi-DC:**
```
1. Выделение Machine ID:
   ✓ Pre-allocate ranges per DC (avoid coordination)
   ✓ Reserve extra for growth
   ✓ Document allocation in config

2. Синхронизация часов:
   ✓ NTP within each DC (±10ms)
   ✓ Accept ~100ms skew across DCs (not on critical path)
   ✓ Monitor clock health in each DC independently

3. ZooKeeper координация:
   ✓ Local ZK cluster per DC (low latency)
   ✓ Optional cross-DC ZK replication (for conflict resolution)
   ✓ Graceful degradation if connection lost

4. Мониторинг и алерты:
   ✓ Monitor machine_id assignment latency
   ✓ Alert on clock drift > threshold
   ✓ Alert on ZK disconnection
   ✓ Track ID collisions (should be 0)
   ✓ Track sequence overflow events

5. Тестирование:
   ✓ Test single DC failure
   ✓ Test network partition (sync recovery)
   ✓ Test clock skew scenarios
   ✓ Test machine_id exhaustion
```

**Типичные ошибки.**
- Забыть про machine ID allocation per DC (collision risk!)
- Требовать strong consistency across DCs (impossible, settle for eventual consistency)
- Не обрабатывать network partition gracefully
- Insufficient NTP sync across DCs (clock skew > 1 second)
- Неправильная zoning (неясно какой DC сгенерировал ID)

**На интервью.**
- Draw multi-DC architecture with ID generator nodes
- Explain machine_id allocation strategy
- Discuss network partition scenarios and recovery
- Show how to determine DC from ID itself
- Discuss monitoring and alerting requirements
- Уточняющий вопрос: How would you handle 10 datacenters? (answer: same approach, just more ranges)

---

## Резюме и сравнение подходов

| Подход | Bits | Координация | Throughput | Сортируемо | Use Case |
|--------|------|-------------|-----------|------------|----------|
| UUID v4 | 128 | Нет | High | Нет | Simple, no special requirements |
| UUID v1 | 128 | Нет | High | Да* | Time-ordered, but privacy issues |
| Snowflake | 64 | Machine ID | 4M/s per node | Да | High-scale distributed (Twitter, Uber) |
| ULID | 128 | Нет | High | Да | Distributed, no coordination needed |
| DB Sequence | 64 | Полная | Medium | Да | Simple, low scale, single DB |
| Redis | 64 | Central | High | Частично | Centralized systems |

*UUID v1 sortable but has known weaknesses

## На интервью: типовой talk-through

Интервьюер: "Спроектируй систему генерации ID для микросервиса, который обрабатывает 100K запросов в секунду."

1. **Clarify requirements** (5 min):
   - Какой масштаб? (100K req/sec = ~100K IDs/sec)
   - Нужна ли сортировка? (обычно да — для логов, аналитики)
   - Какие требования к размеру? (< 128 bits желательно)
   - Несколько датацентров? (нужна ли высокая доступность?)

2. **Propose Snowflake** (5 min):
   - 64-bit: timestamp (41) + machine (10) + sequence (12)
   - Throughput: 4K IDs/ms = 4M IDs/sec per machine
   - Sortable по времени, compact

3. **Design system** (10 min):
   - Multiple generator nodes, stateless
   - ZK for machine_id assignment
   - Local generation (no network roundtrip per ID)
   - NTP sync for clock

4. **Discuss trade-offs** (5 min):
   - Snowflake vs UUID: size (64 vs 128 bits), coordination requirement
   - Clock skew: handle ±5ms drift, alert on larger drifts
   - Scalability: 1K nodes possible (10-bit machine_id limits to 1024)

5. **Handle edge cases** (5 min):
   - Clock goes backward: wait and retry
   - Sequence overflow: backpressure (wait for next ms)
   - Machine_id collision: ZK prevents this
   - Multiple datacenters: allocate ranges per DC

Total: 30 min coherent discussion showing deep understanding.

---

[← Назад к списку тем](README.md)
