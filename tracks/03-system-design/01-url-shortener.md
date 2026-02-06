# 01 — URL Shortener

Развёрнутые вопросы и ответы про проектирование сервиса сокращения ссылок: генерация кодов, кэширование, редиректы, масштабирование, аналитика, высокая доступность. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [00-design-fundamentals](./00-design-fundamentals.md) · Следующая тема: [02-rate-limiter](./02-rate-limiter.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Hash функция** — математическая функция, превращающая входные данные любого размера в строку фиксированной длины. В контексте URL shortener хеш применяется для детерминированного преобразования длинного URL в короткий код. Это гарантирует, что один и тот же URL всегда преобразуется в один и тот же код, что критично для консистентности системы.

**Base62** — система счисления с 62 символами: цифры 0-9, строчные буквы a-z и прописные A-Z. Используется для представления коротких ссылок в максимально компактной и читаемой форме. Base62 кодирование URL-safe и избегает специальных символов, которые требуют экранирования в URL.

**Collision** — ситуация, когда два разных входа обрабатываются одинаково и производят идентичный output. В URL shortener коллизии хешей означают, что два разных длинных URL могут отобразиться на один и тот же короткий код, что приводит к конфликтам. Требуется специальная обработка: либо детектирование и rehashing, либо используется collision-resistant алгоритм.

**QPS (Queries Per Second)** — метрика, отражающая количество запросов, обрабатываемых системой в течение одной секунды. QPS используется для расчёта пропускной способности и определения необходимого количества серверов. Для URL shortener обычно требуется оценка QPS в обе стороны: создание новых ссылок и редирект по существующим.

**Cache (Кэш)** — быстрое временное хранилище в оперативной памяти, которое хранит копии часто используемых данных. В URL shortener кэш ускоряет редирект для популярных ссылок в сотни или тысячи раз, так как не требует обращения к диску. Обычно реализуется с Redis или Memcached.

**TTL (Time To Live)** — время жизни записи в кэше, после которого данные автоматически удаляются. TTL критичен для экономии памяти, так как предотвращает накопление старых данных. Для часто используемых ссылок TTL может быть часов или дней, для редких — минут.

**Sharding** — метод горизонтального масштабирования, при котором данные разделяются по нескольким серверам на основе ключа (например, по хешу). Sharding позволяет системе хранить и обрабатывать объёмы данных, превышающие возможности одного сервера. Каждый shard отвечает за определённое подмножество ключей.

**Redirect** — механизм перенаправления браузера с одного URL на другой с использованием HTTP кодов 301 (постоянный) или 302 (временный). В URL shortener редирект преобразует запрос к короткому URL в запрос к исходному длинному URL. Это основная функция сервиса.

**Idempotency** — свойство операции, при котором повторное выполнение с теми же параметрами производит идентичный результат, не создавая побочных эффектов. Для URL shortener идемпотентность означает, что если создать одну и ту же ссылку дважды, получится один и тот же код, а не два разных. Это требует дедупликации.

**Distributed ID** — уникальный идентификатор, генерируемый в распределённой системе без центрального координатора. В высоконагруженных URL shortener используются алгоритмы вроде Snowflake ID вместо обычных incrementing IDs, чтобы избежать bottleneck на одном сервере и гарантировать уникальность глобально.

---

## Вопросы и разборы

### 1. Как спроектировать URL shortener на интервью?

**Зачем спрашивают.** URL shortener — классический вопрос на system design интервью. Это средней сложности задача, которая требует понимания основных компонентов: API дизайн, выбор БД, масштабирование, кэширование. Интервьюер проверяет структурированность мышления и умение делать trade-off.

**Короткий ответ.** URL shortener преобразует длинные ссылки в короткие коды, перенаправляет обратно. Основные компоненты: stateless API серверы, кэш для горячих данных, база данных для маппинга short→long URL. Паттерн: read-heavy (соотношение ~100:1), требует высокой доступности и низкой задержки.

**Детальный разбор.**

**Требования (обычно уточняются на интервью):**

Функциональные:
- Преобразование длинной ссылки в короткую
- Редирект по короткой ссылке на длинную
- Кастомные алиасы (опционально)
- Экспирация ссылок (опционально)
- Аналитика кликов (опционально)

Нефункциональные:
- High availability (99.9%+)
- Low latency (< 100 мс на редирект)
- Масштабируемость: 100M DAU
- Непредсказуемые коды (нельзя угадать следующий)

**Capacity estimation:**
```
Write нагрузка:
- 100M DAU × 1 URL per user per day = 100M URLs/day
- Average QPS = 100M / 86400 ≈ 1,155 QPS
- Peak QPS (3x) ≈ 3,500 QPS

Read нагрузка (обычно 100:1 к write):
- Average QPS = 115,500 QPS
- Peak QPS ≈ 346,500 QPS

Storage (5 лет):
- URLs per year = 100M × 365 = 36.5B URLs
- 5 years = 182.5B URLs
- Размер одного URL record ≈ 500 bytes (с метаданными)
- Total: 182.5B × 500B = 91.25 TB

Bandwidth (на чтение при peak):
- 346,500 QPS × 500 bytes ≈ 173 MB/s
```

**High-level архитектура:**
```
                              ┌──────────────┐
                              │     CDN      │
                              └──────┬───────┘
                                     │
┌──────────┐    ┌──────────────┐    ┌▼─────────┐    ┌─────────────┐
│  Client  │───▶│Load Balancer │───▶│API Server│───▶│    Cache    │
└──────────┘    └──────────────┘    └┬────────┘    └──────┬──────┘
                                     │                    │
                                     │            ┌───────▼────────┐
                                     └───────────▶│    Database    │
                                                  └────────────────┘

User Flow:
1. Client отправляет длинный URL на API Server
2. API Server генерирует уникальный код (6-8 символов)
3. Записывает маппинг в БД
4. Возвращает short URL клиенту

Redirect Flow:
1. Client запрашивает /{short_code}
2. API Server проверяет Cache (Redis)
3. Если miss — запрашивает БД
4. Возвращает 301/302 редирект на long URL
5. Обновляет аналитику асинхронно
```

**Компоненты:**

1. **API Gateway / Load Balancer** — распределение нагрузки, может быть round-robin или consistent hash
2. **API Servers** — stateless, горизонтально масштабируются, обрабатывают write и read запросы
3. **Cache (Redis)** — хранит горячие ссылки (top 20% URLs = 80% traffic)
4. **Database (SQL или NoSQL)** — persistent storage маппинга short→long URL
5. **Message Queue** — асинхронная обработка аналитики

**Пример.**
```python
# Архитектура на высоком уровне

class URLShortenerAPI:
    def __init__(self):
        self.cache = RedisCache()
        self.db = PostgreSQL()
        self.id_gen = IDGenerator()

    def create_short_url(self, long_url: str, custom_alias: str = None) -> str:
        """
        Создание короткой ссылки.
        Вызовы должны быть идемпотентными: один long_url = один short_code
        """
        # Проверка дубликата
        existing = self.db.find_by_long_url(long_url)
        if existing:
            return existing.short_code

        # Выбор кода: кастомный или сгенерированный
        if custom_alias:
            short_code = custom_alias
            if self.db.exists(short_code):
                raise ValueError("Alias already taken")
        else:
            # Генерируем уникальный код (о методах - далее)
            short_code = self.generate_unique_code()

        # Сохраняем в БД
        url_record = URLRecord(
            short_code=short_code,
            long_url=long_url,
            created_at=now(),
            click_count=0
        )
        self.db.insert(url_record)

        return short_code

    def redirect(self, short_code: str) -> str:
        """
        Редирект по короткому коду.
        Read-heavy операция, требует кэширования.
        """
        # 1. Проверяем кэш (большинство запросов попадают сюда)
        cached_url = self.cache.get(f"url:{short_code}")
        if cached_url:
            self.analytics_queue.send({
                "short_code": short_code,
                "timestamp": now()
            })
            return cached_url

        # 2. Проверяем БД
        url_record = self.db.get_by_short_code(short_code)
        if not url_record:
            return None  # 404

        # 3. Проверяем экспирацию
        if url_record.expires_at and url_record.expires_at < now():
            return None  # 410 Gone

        # 4. Кэшируем результат
        self.cache.set(f"url:{short_code}", url_record.long_url, ttl=3600)

        # 5. Асинхронно обновляем аналитику
        self.analytics_queue.send({
            "short_code": short_code,
            "timestamp": now(),
            "ip": request.remote_addr,
            "user_agent": request.user_agent
        })

        return url_record.long_url

    def generate_unique_code(self) -> str:
        """Генерация уникального кода (методы описаны далее)"""
        pass
```

**Типичные ошибки.**
- Не уточнить требования (масштаб, read/write ratio,需要ли аналитика)
- Выбрать неправильную БД (SQL хорош для аналитики, NoSQL лучше масштабируется)
- Забыть про кэширование — система не выдержит read-heavy нагрузку
- Синхронная аналитика — замедлит редиректы, нужна queue
- Не учитывать идемпотентность — повторный запрос с одним URL может создать разные коды

**На интервью.**
- Начни с требований и capacity estimation
- Нарисуй high-level архитектуру (5-10 компонентов)
- Обсуди trade-off: SQL vs NoSQL, кэширование, асинхронность
- Покажи знание классических паттернов: sharding, replication, circuit breaker
- Follow-up: «Как обработать 10x traffic spike?» — добавить инстанс, scale cache, optimize queries

---

### 2. Какие подходы к генерации коротких кодов существуют?

**Зачем спрашивают.** Генерация кода — критическое решение. Выбор подхода влияет на всю архитектуру: нужна ли координация, как обрабатывать коллизии, как масштабировать. Интервьюер проверяет понимание trade-off между простотой и производительностью.

**Короткий ответ.** Три основных подхода: (1) Хеш-based — детерминированно, но коллизии; (2) Счётчик-based (distributed ID) — без коллизий, нужна координация; (3) Pre-generated keys — самый быстрый, но требует отдельный сервис генерации.

**Детальный разбор.**

**Подход 1: Hash-based**
```
┌─────────────┐
│ "Long URL"  │
└──────┬──────┘
       │
       ▼
┌──────────────────────┐
│ MD5/SHA1 hash        │ ─► 128/160 bits
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Take first 6 bytes   │ ─► 48 bits
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Base62 encode        │ ─► 6-8 символов (62^6 = 56.8B combinations)
└──────────────────────┘
```

**Преимущества:**
- Stateless, детерминированный
- Один и тот же URL всегда генерирует одинаковый код (идемпотентность)
- Не требует координации между серверами

**Недостатки:**
- Коллизии возможны (разные URL хешируются в одно)
- Нужно обработать: retry с suffix или счётчик

```python
import hashlib
import base64

def hash_based_code_gen(long_url: str, retry_count: int = 0) -> str:
    """
    Hash-based подход с обработкой коллизий
    """
    # Кешируем значение retry_count в hash
    suffix = long_url + str(retry_count)

    # SHA-256 → first 6 bytes → base62
    hash_bytes = hashlib.sha256(suffix.encode()).digest()[:6]
    code_num = int.from_bytes(hash_bytes, 'big')

    return base62_encode(code_num)

def base62_encode(num: int) -> str:
    """Кодирует число в base62 строку"""
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    if num == 0:
        return chars[0]

    result = []
    while num > 0:
        result.append(chars[num % 62])
        num //= 62

    return ''.join(reversed(result))

# Usage:
long_url = "https://example.com/very/long/path/with/many/parameters"
short_code = hash_based_code_gen(long_url)  # "a1b2c3"

# При коллизии:
# Если код уже существует в БД, пытаемся с retry_count=1
short_code = hash_based_code_gen(long_url, retry_count=1)  # "d4e5f6"
```

**Подход 2: Distributed ID + Base62**
```
┌──────────────────────────────────────────────────────────────────┐
│                       Snowflake ID (64-bit)                      │
├─────────────┬──────────────────┬─────────────┬──────────────────┤
│ Timestamp   │  Machine ID      │  Sequence   │  Random bits     │
│ (41 bits)   │  (10 bits)       │  (12 bits)  │  (1 bit)         │
│ ~69 years   │  ~1000 machines  │  4096/ms    │                  │
└─────────────┴──────────────────┴─────────────┴──────────────────┘
       │
       ▼
┌──────────────────────┐
│ Base62 encode        │ ─► ~11 символов
└──────────────────────┘
```

**Преимущества:**
- Гарантированно уникален (no collisions)
- Монотонно возрастающий (можно использовать как primary key)
- Распределённая генерация (не требует центрального сервиса)

**Недостатки:**
- Не детерминирован (одинаковые URL → разные коды)
- Требует синхронизации времени между машинами
- Коды длиннее (11 символов vs 6-7)

```python
import time
import random

class SnowflakeIDGenerator:
    """Распределённый генератор ID (упрощённая версия)"""

    EPOCH = 1577836800000  # 2020-01-01 in milliseconds
    MACHINE_ID_BITS = 10
    SEQUENCE_BITS = 12
    MAX_MACHINE_ID = (1 << MACHINE_ID_BITS) - 1
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1

    def __init__(self, machine_id: int):
        if machine_id > self.MAX_MACHINE_ID:
            raise ValueError(f"Machine ID must be <= {self.MAX_MACHINE_ID}")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1

    def generate(self) -> int:
        """Генерирует уникальный 64-bit ID"""
        timestamp = int(time.time() * 1000)

        if timestamp < self.last_timestamp:
            raise Exception("Clock moved backwards")

        if timestamp == self.last_timestamp:
            # Одна и та же миллисекунда, инкрементируем sequence
            self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE
            if self.sequence == 0:
                # Переполнение sequence, ждём следующую миллисекунду
                timestamp = self._wait_next_millis(self.last_timestamp)
        else:
            # Новая миллисекунда, сбрасываем sequence
            self.sequence = 0

        self.last_timestamp = timestamp

        # Собираем ID
        id_val = ((timestamp - self.EPOCH) << (self.MACHINE_ID_BITS + self.SEQUENCE_BITS))
        id_val |= (self.machine_id << self.SEQUENCE_BITS)
        id_val |= self.sequence

        return id_val

    def _wait_next_millis(self, last_timestamp: int) -> int:
        """Ждёт следующую миллисекунду"""
        timestamp = int(time.time() * 1000)
        while timestamp <= last_timestamp:
            timestamp = int(time.time() * 1000)
        return timestamp

# Usage:
generator = SnowflakeIDGenerator(machine_id=1)
snowflake_id = generator.generate()  # e.g., 1234567890123456789
short_code = base62_encode(snowflake_id)  # "a1b2c3d4e5f"
```

**Подход 3: Pre-generated Keys Service**
```
┌─────────────────────────────────────────────────────────────────┐
│              Key Generation Service (отдельный сервис)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Генерирует батчи уникальных кодов в фоне (например, по    │
│     1M кодов за раз)                                            │
│                                                                 │
│  2. Хранит коды в отдельной таблице с статусом:                 │
│     - AVAILABLE — готов к использованию                        │
│     - USED — был выделен API server                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
       │
       │
┌──────▼──────────────────┐
│  API Server 1           │ ─► Запросит batch (e.g., 1000 ключей)
└─────────────────────────┘
       │
┌──────▼──────────────────┐
│  API Server 2           │ ─► Запросит batch
└─────────────────────────┘

Поток:
1. API Server запрашивает batch доступных ключей
2. KGS отмечает их как USED и отправляет серверу
3. API Server локально кэширует batch
4. При создании URL — берёт ключ из локального кэша
5. Когда кэш исчерпан, запрашивает новый batch
```

**Преимущества:**
- Очень быстро (не нужно генерировать при каждом запросе)
- Гарантия уникальности
- КГС может интегрироваться с другими системами

**Недостатки:**
- Требует отдельный сервис (дополнительная complexity)
- Может быть узким местом (все серверы зависят от КГС)
- Потеря неиспользованных ключей при сбое API server

```python
class KeyGenerationService:
    """Key Generation Service — генерирует и распределяет коды"""

    def __init__(self, db):
        self.db = db
        self.batch_size = 100000

    def generate_batch(self, batch_size: int = None) -> List[str]:
        """
        Генерирует батч уникальных кодов и помечает как используемые
        """
        if batch_size is None:
            batch_size = self.batch_size

        codes = []
        while len(codes) < batch_size:
            # Генерируем случайный код (или используем Snowflake)
            code = self._generate_random_code()

            # Проверяем уникальность
            if not self.db.code_exists(code):
                codes.append(code)

        # Помечаем все как AVAILABLE
        self.db.batch_insert_codes(
            [(code, 'AVAILABLE') for code in codes]
        )

        return codes

    def request_batch(self, server_id: str, batch_size: int = 1000) -> List[str]:
        """
        API Server запрашивает батч кодов для локального использования
        """
        # Получаем доступные коды
        available_codes = self.db.fetch_available_codes(batch_size)

        if len(available_codes) < batch_size * 0.9:
            # Генерируем ещё, если мало осталось
            self._generate_batch()

        # Помечаем как используемые сервером
        self.db.mark_codes_used(available_codes, server_id)

        return available_codes

    def _generate_random_code(self, length: int = 6) -> str:
        """Генерирует случайный код"""
        import random
        chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
        return ''.join(random.choice(chars) for _ in range(length))

# Usage в API Server:
class URLShortenerAPI:
    def __init__(self, kgs_client):
        self.kgs = kgs_client
        self.local_key_cache = []

    def create_short_url(self, long_url: str) -> str:
        # Пополняем локальный кэш, если нужно
        if not self.local_key_cache:
            self.local_key_cache = self.kgs.request_batch()

        # Берём ключ из локального кэша (очень быстро)
        short_code = self.local_key_cache.pop()

        # Сохраняем маппинг в БД
        self.db.insert(short_code, long_url)

        return short_code
```

**Сравнение подходов:**
```
┌──────────────┬────────────────┬──────────────┬─────────────────┐
│ Характеристика │ Hash-based    │ Snowflake    │ Pre-generated   │
├──────────────┼────────────────┼──────────────┼─────────────────┤
│ Коллизии     │ Возможны       │ Нет          │ Нет             │
│ Скорость     │ Медленно        │ Быстро       │ Очень быстро    │
│ Complexity   │ Низкая         │ Средняя      │ Высокая         │
│ Идемпотентность│ Да (один URL  │ Нет (разные  │ Нет             │
│              │  = один код)   │  коды)       │                 │
│ Координация  │ Не нужна       │ Нужна (часы) │ Нужна (КГС)     │
│ Масштабируемость│ Хорошо       │ Отлично      │ Отлично         │
└──────────────┴────────────────┴──────────────┴─────────────────┘
```

**Типичные ошибки.**
- Hash без обработки коллизий — может не найти код при INSERT
- Snowflake с неправильной синхронизацией часов — дублирование ID
- Pre-generated коды без резерва — KGS может не успеть сгенерировать
- Не учитывать идемпотентность — разные коды для одного URL

**На интервью.**
- Объясни каждый подход с диаграммой
- Обсуди trade-off: простота vs производительность
- Рекомендуй Snowflake для большинства случаев (good balance)
- Follow-up: «Какой подход выбрать при 100K QPS?» — Pre-generated для минимальной latency

---

### 3. Как работает base62 encoding?

**Зачем спрашивают.** Base62 — стандарт для URL shortener и distributed ID. Интервьюер проверяет базовое понимание систем счисления и кодирования данных.

**Короткий ответ.** Base62 использует алфавит из 62 символов [0-9a-zA-Z], чтобы закодировать число в URL-safe строку. Преобразуем число в base-62, повторно деля на 62 и собирая остатки. Декодирование — обратный процесс (Horner's method).

**Детальный разбор.**

**Система счисления:**
```
Base-10 (decimal):    0-9 (10 символов)
Base-16 (hex):        0-9A-F (16 символов)
Base-62:              0-9a-zA-Z (62 символа)

Почему 62? Потому что:
- URL-safe (в отличие от Base64, где есть +/=)
- Достаточно для URL shortener
- Case-sensitive (различает a и A)
```

**Как кодировать (число → строка):**
```
Число: 12345

Шаг 1: 12345 % 62 = 57      (индекс символа в алфавите)
       12345 / 62 = 198

Шаг 2: 198 % 62 = 10
       198 / 62 = 3

Шаг 3: 3 % 62 = 3
       3 / 62 = 0 (конец)

Остатки (в обратном порядке): [3, 10, 57]
Алфавит:  0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
Индексы:  3 → 'd', 10 → 'a', 57 → 'v'

Результат: "dav"
```

**Как декодировать (строка → число):**
```
Строка: "dav"

Алфавит индексы: d=3, a=10, v=57

Horner's method:
result = 0
result = result * 62 + 3 = 3
result = result * 62 + 10 = 3*62 + 10 = 196
result = result * 62 + 57 = 196*62 + 57 = 12152 + 57 = 12209

Хм, не совпадает (ожидаем 12345). Давайте пересчитаем:
Правильно: 3*62^2 + 10*62 + 57 = 3*3844 + 620 + 57 = 11532 + 620 + 57 = 12209

Ошибка в расчётах выше. Фактически:
12345 % 62 = 57
12345 // 62 = 198

198 % 62 = 10
198 // 62 = 3

3 % 62 = 3
3 // 62 = 0

Обратно: 3*62^2 + 10*62 + 57 = 11532 + 620 + 57 = 12209 ≠ 12345

Давайте пересчитаем 12345 в base 62 корректно:
12345 = q1 * 62 + r0
12345 = 199 * 62 + 7
199 = q2 * 62 + r1
199 = 3 * 62 + 13
3 = q3 * 62 + r2
3 = 0 * 62 + 3

Остатки (snacktop): [3, 13, 7]
Символы: 3→'3', 13→'d', 7→'7'
Результат: "3d7"

Проверка: 3*62^2 + 13*62 + 7 = 3*3844 + 806 + 7 = 11532 + 806 + 7 = 12345 ✓
```

**Пример кода:**

```python
def base62_encode(num: int) -> str:
    """
    Кодирует число в base62 строку

    62^6 = 56,800,235,584 (56 млрд)
    62^7 = 3,521,614,606,208 (3.5 трлн)

    Для URL shortener обычно достаточно 6-7 символов
    """
    if num == 0:
        return "0"

    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    result = []

    while num > 0:
        result.append(chars[num % 62])
        num //= 62

    # Развернули в обратном порядке
    return ''.join(reversed(result))

def base62_decode(s: str) -> int:
    """
    Декодирует base62 строку в число
    """
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    char_to_index = {c: i for i, c in enumerate(chars)}

    result = 0
    for char in s:
        result = result * 62 + char_to_index[char]

    return result

# Testing
num = 12345
encoded = base62_encode(num)
decoded = base62_decode(encoded)
assert decoded == num, f"Mismatch: {num} != {decoded}"
print(f"{num} → {encoded} → {decoded}")  # 12345 → 3d7 → 12345
```

**Почему base62, а не base64?**
```
Base64: 0-9A-Za-z+/=  (65 символов вместе с паддингом)
- Проблема: +, / и = нужно URL-encoding (%2B, %2F, %3D)
- Длиннее на ~10%

Base62: 0-9a-zA-Z  (ровно 62 символа)
- Все символы URL-safe (не требуют экранирования)
- Компактнее

Пример:
Одно число:
- Base64: 6 символов + padding
- Base62: 6 символов, no padding
```

**Масштабирование кодов:**
```
Какой длины код выбрать?

62^5 ≈ 916 миллионов
62^6 ≈ 56.8 миллиардов  ← обычно используется (совпадение с twitter shortlinks!)
62^7 ≈ 3.5 триллионов
62^8 ≈ 218 триллионов

Для 100M DAU × 365 дней × 5 лет = 182.5B URLs
→ нужно минимум 62^7 (3.5T > 182.5B)

На практике часто используют:
- 6 символов для личных projects (до 56B URLs)
- 7 символов для production (до 3.5T URLs)
```

**Пример.**
```python
# URL Shortener с base62

import hashlib

def hash_based_short_code(long_url: str) -> str:
    """Генерирует код на основе хеша"""
    # Хешируем URL
    hash_bytes = hashlib.md5(long_url.encode()).digest()[:6]
    # Конвертируем в число
    num = int.from_bytes(hash_bytes, 'big')
    # Кодируем в base62
    return base62_encode(num)

# Тест
url1 = "https://example.com/very/long/path"
url2 = "https://github.com/user/repo/issues/123"

code1 = hash_based_short_code(url1)
code2 = hash_based_short_code(url2)

print(f"URL1: {code1}")  # e.g., "aBcDeFg"
print(f"URL2: {code2}")  # e.g., "xYzAbCd"

# Гарантия: одинаковые URL дают одинаковые коды (идемпотентность)
assert hash_based_short_code(url1) == code1
```

**Типичные ошибки.**
- Забыть развернуть результат при кодировании
- Неправильная арифметика base conversion
- Не учитывать edge case: число 0
- Использовать неправильный алфавит (с символами, требующими экранирования)

**На интервью.**
- Объясни алгоритм кодирования и декодирования
- Покажи, как считаются combinations (62^n)
- Упомяни, почему base62, а не base64
- Follow-up: «Как выбрать оптимальную длину кода?» — балансировать между коллизиями и длиной

---

### 4. Как обрабатывать коллизии при хешировании?

**Зачем спрашивают.** Hash-based кодирование неизбежно приводит к коллизиям. Интервьюер проверяет понимание вероятности коллизий, методов обработки и trade-off между ними.

**Короткий ответ.** Коллизия — когда два разных URL хешируются в одинаковый код. Способы обработки: (1) Retry с добавлением счётчика к исходной строке; (2) Увеличить хеш (использовать больше байтов); (3) Проверить в БД и обновить при конфликте.

**Детальный разбор.**

**Probability Paradox (Birthday Problem):**
```
Base62^6 = 56.8 млрд возможных кодов

Для достижения 50% вероятности коллизии:
sqrt(56.8B) ≈ 238,000 URLs нужно

Т.е. при 238K URLs вероятность коллизии = 50% !

Для 99% коллизии: sqrt(56.8B * ln(1/0.01)) ≈ 812K URLs

Поэтому base62^6 опасна для большого масштаба.
```

**Метод 1: Retry с счётчиком**
```
┌─────────────────┐
│  Long URL       │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────┐
│ hash(URL + counter) for counter  │
│ in range(0, max_retries)         │
└────────┬────────────────────────┘
         │
         ├─► counter=0: hash("URL") → "aBcDeFg" → Check DB
         │   → Already exists! Retry...
         │
         ├─► counter=1: hash("URL1") → "xYzAbCd" → Check DB
         │   → Not exists! Use this code.
         │
         └─► If all retries fail → error or random fallback
```

**Преимущества:**
- Простая реализация
- Deterministic (один URL всегда генерирует одинаковый код при одном counter)

**Недостатки:**
- Множественные запросы к БД при коллизиях
- Не масштабируется при высоком traffic (много коллизий)

```python
class HashBasedGenerator:
    MAX_RETRIES = 3

    def generate(self, long_url: str) -> str:
        for counter in range(self.MAX_RETRIES):
            suffix = long_url + str(counter)
            code = self._hash_to_code(suffix)

            # Проверяем уникальность в БД
            if not self.db.code_exists(code):
                return code

        # Не удалось после retries
        raise Exception("Could not generate unique code")

    def _hash_to_code(self, s: str) -> str:
        hash_bytes = hashlib.md5(s.encode()).digest()[:6]
        num = int.from_bytes(hash_bytes, 'big')
        return base62_encode(num)
```

**Метод 2: Проверка и обновление**
```
┌─────────────────┐
│  Long URL       │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────┐
│ hash(URL) → code                 │
└────────┬────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│ INSERT code INTO urls...         │
│ ON CONFLICT (code) UPDATE        │
│ long_url = EXCLUDED.long_url     │
└────────┬────────────────────────┘
         │
         ├─► Успех: code уникален
         │
         └─► Конфликт: обновляем существующий запись
             (если long_urls совпадают, это OK)
```

Работает, если требуется идемпотентность: один long_url всегда маппится на один short_code, даже при повторных запросах.

```python
def upsert_url(self, long_url: str) -> str:
    """
    Deterministic: один long_url → один short_code
    Идемпотентен: повторные запросы вернут тот же code
    """
    code = self._hash_to_code(long_url)

    # Пытаемся вставить; если конфликт — обновляем
    result = self.db.execute("""
        INSERT INTO urls (short_code, long_url, created_at)
        VALUES (%s, %s, NOW())
        ON CONFLICT (short_code) DO UPDATE
        SET long_url = EXCLUDED.long_url
        RETURNING short_code
    """, (code, long_url))

    return result.short_code
```

**Метод 3: Увеличить размер хеша**
```
┌─────────────────┐
│  Long URL       │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────┐
│ hash(URL) — использовать     │
│ больше байтов (e.g., 8      │
│ вместо 6)                    │
└────────┬────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Base62 encode                │
│ Результат: 9-10 символов    │
│ вместо 6-7                   │
└────────────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Вероятность коллизии:        │
│ 62^9 = 13.8 триллионов       │
│ → практически невозможна     │
└────────────────────────────┘
```

**Trade-off:** Коды длиннее (9-10 символов), но гарантия уникальности без retries.

```python
def generate_large_hash(self, long_url: str) -> str:
    """Используем 8 байтов хеша (64 bits) вместо 6 (48 bits)"""
    hash_bytes = hashlib.sha256(long_url.encode()).digest()[:8]
    num = int.from_bytes(hash_bytes, 'big')
    return base62_encode(num)  # ~10 символов

# 62^8 = 218 триллионов → вероятность коллизии ≈ 0
```

**Статистика коллизий:**

```
┌────────────┬──────────────────┬──────────────────────────────────────┐
│ Длина кода │ Всего комбинаций │ URLs до 50% коллизии (Birthday)       │
├────────────┼──────────────────┼──────────────────────────────────────┤
│ 5          │ 916M             │ 38K                                  │
│ 6          │ 56.8B            │ 238K                                 │
│ 7          │ 3.5T             │ 1.4M                                 │
│ 8          │ 218T             │ 9.3M                                 │
└────────────┴──────────────────┴──────────────────────────────────────┘

Вывод: длина 7 достаточна для 100M URLs (даже с 50% safety margin)
```

**Пример полной обработки коллизий:**

```python
class CollisionHandler:
    """Полная стратегия обработки коллизий"""

    STRATEGIES = {
        'retry': RetryStrategy(),
        'upsert': UpsertStrategy(),
        'larger_hash': LargerHashStrategy(),
    }

    def __init__(self, strategy: str = 'retry'):
        self.strategy = self.STRATEGIES[strategy]

    def generate_short_code(self, long_url: str) -> str:
        """
        Генерирует код с обработкой коллизий согласно стратегии
        """
        return self.strategy.generate(long_url)

class RetryStrategy:
    """Retry с увеличивающимся счётчиком"""
    def generate(self, long_url: str) -> str:
        for attempt in range(10):
            code = self._hash_with_attempt(long_url, attempt)
            if not self._exists_in_db(code):
                return code
        raise Exception("Max retries exceeded")

class UpsertStrategy:
    """Upsert для идемпотентности"""
    def generate(self, long_url: str) -> str:
        code = self._simple_hash(long_url)
        self._upsert_into_db(code, long_url)
        return code

class LargerHashStrategy:
    """Использовать больший хеш"""
    def generate(self, long_url: str) -> str:
        # Используем 8 байтов SHA256 вместо 6 MD5
        return self._large_hash_to_code(long_url)
```

**Типичные ошибки.**
- Игнорировать вероятность коллизий для малых кодов
- Retry в синхронном пути критикал-сервиса (замедляет)
- Не логировать коллизии (сложно отладить)
- Использовать слишком малый хеш (< 48 bits)

**На интервью.**
- Объясни Birthday Paradox и вероятность коллизий
- Обсуди каждый метод с pros/cons
- Рекомендуй комбинацию: большой хеш (8 байтов) + retry для safety
- Follow-up: «Как мониторить коллизии?» — логировать в каждом retry, аналитика

---

### 5. Какую БД выбрать для URL shortener?

**Зачем спрашивают.** Выбор базы данных определяет масштабируемость, нужна ли шардизация, как обрабатывать replication. Интервьюер проверяет понимание trade-off между SQL и NoSQL.

**Короткий ответ.** Для URL shortener подходят оба: SQL (PostgreSQL) если нужна аналитика и ACID, NoSQL (DynamoDB, Cassandra) если нужна максимальная масштабируемость. Критерии: access pattern (key-value lookup), read/write ratio (100:1), consistency требования.

**Детальный разбор.**

**SQL подход (PostgreSQL):**
```
Таблица: urls
┌────┬─────────────┬───────────────────┬─────────┬──────────────┬──────────────┐
│ id │ short_code  │ long_url          │ user_id │ created_at   │ click_count  │
├────┼─────────────┼───────────────────┼─────────┼──────────────┼──────────────┤
│ 1  │ "aBcDeFg"   │ "https://exa..."  │ 100     │ 2024-01-01   │ 1234         │
│ 2  │ "xYzAbCd"   │ "https://git..."  │ 101     │ 2024-01-02   │ 5678         │
│ 3  │ "pQrStUvW"  │ "https://hub..."  │ 102     │ 2024-01-03   │ 0            │
└────┴─────────────┴───────────────────┴─────────┴──────────────┴──────────────┘

Индексы:
- PRIMARY KEY (id) — auto-increment
- UNIQUE INDEX (short_code) — быстрый lookup по коду
- INDEX (user_id) — поиск всех ссылок пользователя
```

**Преимущества SQL:**
- ACID транзакции (гарантия целостности)
- JOIN для analytics (покупка пользователя + его ссылки)
- Простые запросы
- Хорошо изучена и поддерживается

**Недостатки SQL:**
- Vertical scaling до лимита
- Horizontal scaling требует sharding (complexity)
- Write bottleneck при высоком traffic

**Пример SQL schema:**

```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    custom_alias BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0,
    is_deleted BOOLEAN DEFAULT false,

    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

CREATE TABLE clicks (
    id BIGSERIAL PRIMARY KEY,
    url_id BIGINT NOT NULL REFERENCES urls(id),
    timestamp TIMESTAMP DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT,
    referer TEXT,
    country VARCHAR(2),

    INDEX idx_url_id (url_id),
    INDEX idx_timestamp (timestamp)
);

-- Sharding для масштабирования (добавлено позже)
-- Вместо одной таблицы urls → urls_0, urls_1, ..., urls_N
-- Разделяем по hash(short_code) % N
```

**NoSQL подход (DynamoDB / Cassandra):**
```
Table: urls
Partition Key: short_code
Sort Key: none (не нужен)

Атрибуты:
- short_code (PK)
- long_url
- user_id
- created_at
- click_count
- expires_at

Доступ:
- GetItem("short_code") → O(1) в любом масштабе
```

**Преимущества NoSQL:**
- Горизонтальное масштабирование встроено
- High availability (multiple replicas)
- Отличный для read-heavy нагрузок
- Простой sharding (по partition key)

**Недостатки NoSQL:**
- Нет JOIN (нельзя получить все ссылки пользователя легко)
- Eventual consistency (может быть задержка)
- Queries ограничены (не SQL flexibility)
- Дороже на ops

**Пример DynamoDB:**

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('urls')

# Создание ссылки
table.put_item(Item={
    'short_code': 'aBcDeFg',
    'long_url': 'https://example.com/very/long/path',
    'user_id': '100',
    'created_at': int(time.time()),
    'click_count': 0,
    'expires_at': None,
})

# Получение ссылки (очень быстро)
response = table.get_item(Key={'short_code': 'aBcDeFg'})
item = response['Item']
long_url = item['long_url']

# Update click count
table.update_item(
    Key={'short_code': 'aBcDeFg'},
    UpdateExpression='SET click_count = click_count + :inc',
    ExpressionAttributeValues={':inc': 1}
)

# Поиск по user_id (требует Global Secondary Index)
# Таблица должна иметь GSI: user_id as PK, created_at as SK
response = table.query(
    IndexName='user_id_index',
    KeyConditionExpression='user_id = :uid',
    ExpressionAttributeValues={':uid': '100'}
)
```

**Сравнение:**

```
┌─────────────────────────┬──────────────────────┬──────────────────────┐
│ Характеристика          │ SQL (PostgreSQL)     │ NoSQL (DynamoDB)     │
├─────────────────────────┼──────────────────────┼──────────────────────┤
│ Consistency             │ Strong               │ Eventual             │
│ Transactions            │ ACID                 │ Limited (item level) │
│ Scalability             │ Vertical first       │ Horizontal (встроено)│
│ Complex queries         │ Да (JOIN, GROUP BY)  │ Нет                  │
│ Sharding                │ Нужен (сложный)      │ Встроен (просто)     │
│ Cost (small scale)      │ Дешево               │ Дороговато           │
│ Cost (large scale)      │ Дорого               │ Может быть дешевле   │
│ Operations complexity   │ Средняя              │ Низкая (managed)     │
│ Analytics               │ Легко (JOIN)         │ Сложно (ETL needed)  │
└─────────────────────────┴──────────────────────┴──────────────────────┘
```

**Гибридный подход (рекомендуется):**

```
┌──────────────────────────────────────┐
│         API Request                  │
└───────────────┬──────────────────────┘
                │
                ▼
    ┌───────────────────────┐
    │   Short Code Gen      │
    │   & Validation        │
    └───────────┬───────────┘
                │
    ┌───────────┴───────────┐
    │                       │
    ▼                       ▼
┌─────────────┐       ┌──────────────┐
│  DynamoDB   │       │  PostgreSQL  │
│  (primary)  │       │  (secondary) │
│             │       │              │
│ - Fast GET  │       │ - Analytics  │
│ - Scale     │       │ - User stuff │
│ - Cheap     │       │ - Reporting  │
└─────────────┘       └──────────────┘
    │                       │
    └───────────┬───────────┘
                │
       ┌────────▼─────────┐
       │ Redis Cache      │
       │ (Hot 20% URLs)   │
       └──────────────────┘
```

**Типичные ошибки.**
- Выбрать SQL для масштаба 10B URLs без планирования sharding
- Выбрать NoSQL и потом понять, что нужны сложные аналитические queries
- Забыть про cache между NoSQL и приложением
- Не индексировать (для SQL) или забыть про GSI (для NoSQL)

**На интервью.**
- Начни с requirements: scale, consistency, queries
- Рекомендуй SQL для среднего масштаба с аналитикой, NoSQL для huge scale
- Упомяни гибридный подход (DynamoDB + PostgreSQL)
- Follow-up: «Как шардировать PostgreSQL?» — по hash(short_code) % N, нужен шардирующий слой

---

### 6. Как реализовать кэширование для read-heavy нагрузки?

**Зачем спрашивают.** URL shortener — read-heavy (100:1 ratio). Без кэша система не выдержит нагрузку. Интервьюер проверяет понимание cache strategy, invalidation и cache warming.

**Короткий ответ.** Используем Redis для кэширования популярных ссылок. Парето: 20% ссылок = 80% трафика. При read: проверяем cache → miss → DB → update cache. TTL~1 часа для автоматической инвалидации. Cache warming при startup или на основе popularity.

**Детальный разбор.**

**Кэш-фреймворк:**
```
┌──────────────────┐
│  Client Request  │
│  GET /aBcDeFg    │
└────────┬─────────┘
         │
         ▼
    ┌─────────────┐
    │ Cache Hit?  │
    └──┬──────┬──┘
       │      │
     YES    NO
       │      │
       │      ▼
       │   ┌──────────┐
       │   │ DB Lookup│
       │   └────┬─────┘
       │        │
       │        ▼
       │   ┌──────────────────┐
       │   │ Set Cache        │
       │   │ TTL = 3600s      │
       │   └────┬─────────────┘
       │        │
       └────┬───┘
            │
            ▼
    ┌──────────────────┐
    │ Return long_url  │
    │ 301/302 redirect │
    └──────────────────┘
```

**Кэш-стратегии:**

1. **Lazy loading (Demand-filled)** — кэшируем по запросу
```python
def get_url(short_code: str) -> str:
    # 1. Check cache
    cached = cache.get(short_code)
    if cached:
        return cached

    # 2. Miss → fetch from DB
    url = db.get(short_code)
    if not url:
        return None

    # 3. Populate cache
    cache.set(short_code, url, ttl=3600)
    return url
```

Плюсы: просто, нет пустых кэш-записей
Минусы: холодный старт (первый запрос медленный)

2. **Cache warming** — предварительно загружаем популярные
```python
def warm_cache(limit: int = 1000):
    """
    При startup или периодически загружаем top URLs в кэш
    """
    # Получаем top URLs по click_count
    top_urls = db.query("""
        SELECT short_code, long_url
        FROM urls
        ORDER BY click_count DESC
        LIMIT %s
    """, (limit,))

    # Загружаем в кэш
    for code, url in top_urls:
        cache.set(code, url, ttl=86400)  # 24 hours

# Вызов при startup
# warm_cache()
```

Плюсы: улучшает latency для популярных ссылок
Минусы: дополнительная задача на startup

3. **Write-through** — обновляем кэш сразу при создании
```python
def create_short_url(long_url: str, short_code: str) -> str:
    # 1. Сохраняем в БД
    db.insert(short_code, long_url)

    # 2. Обновляем кэш
    cache.set(short_code, long_url, ttl=3600)

    return short_code
```

**Estimation кэш-размера:**

```
По Парето (20/80):
- 100M URLs в системе
- Top 20% = 20M URLs
- 80% трафика идёт в эти 20M URLs

Размер одной записи в Redis:
- Key: "url:aBcDeFg" = ~12 bytes
- Value: "https://example.com/very/long/path" = ~100 bytes (avg)
- Redis overhead = ~50 bytes
- Total per entry ≈ ~160 bytes

Размер кэша для top 20M:
20M × 160 bytes = 3.2 GB

На практике:
- Берём 10-20% от top (2-4M URLs)
- Размер кэша = 500 MB - 1 GB (вполне управляемо)
```

**Пример Redis кэширования:**

```python
import redis
import json
from typing import Optional

class URLCache:
    def __init__(self, redis_host='localhost', redis_port=6379):
        self.redis = redis.Redis(host=redis_host, port=redis_port)
        self.prefix = 'url:'
        self.default_ttl = 3600  # 1 час

    def get(self, short_code: str) -> Optional[str]:
        """Получает URL из кэша"""
        key = f"{self.prefix}{short_code}"
        value = self.redis.get(key)
        return value.decode('utf-8') if value else None

    def set(self, short_code: str, long_url: str, ttl: int = None) -> bool:
        """Устанавливает URL в кэш"""
        if ttl is None:
            ttl = self.default_ttl

        key = f"{self.prefix}{short_code}"
        return self.redis.setex(key, ttl, long_url)

    def delete(self, short_code: str) -> bool:
        """Удаляет из кэша"""
        key = f"{self.prefix}{short_code}"
        return self.redis.delete(key) > 0

    def warm_popular(self, top_urls: list) -> int:
        """Загружает популярные URLs в кэш"""
        pipe = self.redis.pipeline()

        for short_code, long_url in top_urls:
            key = f"{self.prefix}{short_code}"
            # Используем longer TTL для popular
            pipe.setex(key, 86400, long_url)  # 24 hours

        pipe.execute()
        return len(top_urls)

# Usage в приложении:

class URLShortenerService:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def redirect(self, short_code: str) -> Optional[str]:
        """Redirect с кэшированием"""
        # 1. Try cache
        cached_url = self.cache.get(short_code)
        if cached_url:
            self._record_click_async(short_code)  # async analytics
            return cached_url

        # 2. Try DB
        db_result = self.db.get_url(short_code)
        if not db_result:
            return None  # 404

        long_url = db_result['long_url']
        expires_at = db_result.get('expires_at')

        # 3. Check expiration
        if expires_at and expires_at < datetime.now():
            return None  # 410 Gone

        # 4. Update cache
        self.cache.set(short_code, long_url)

        # 5. Record analytics
        self._record_click_async(short_code)

        return long_url

    def _record_click_async(self, short_code: str):
        """Отправляем в queue для асинхронной обработки"""
        self.analytics_queue.push({
            'short_code': short_code,
            'timestamp': time.time(),
            'user_agent': request.user_agent,
            'ip': request.remote_addr
        })
```

**Cache invalidation strategies:**

```
1. TTL (Time-To-Live) — автоматическое удаление через N секунд
   - Плюсы: просто, не требует логики инвалидации
   - Минусы: может быть outdated, может быть потрачено место на expired

2. LRU (Least Recently Used) — при переполнении удаляем давно не используемые
   - Плюсы: Redis встроенная поддержка (maxmemory-policy=allkeys-lru)
   - Минусы: потеря данных при pressure

3. Event-based — удаляем при обновлении или удалении URL
   - Плюсы: always fresh
   - Минусы: нужна логика обновления

Рекомендуемый подход: TTL (1 час) + LRU policy
```

**Типичные ошибки.**
- Кэш без TTL — outdated data
- Очень большой кэш — тратим память зря
- Синхронная запись в кэш на critical path — замедляет
- Забыть invalidate при обновлении URL

**На интервью.**
- Объясни Парето (20/80)
- Покажи расчёт размера кэша
- Обсуди TTL vs LRU vs Event-based инвалидацию
- Follow-up: «Как обновить кэш при изменении URL?» — publish-subscribe pattern или direct invalidation

---

### 7. В чём разница между 301 и 302 редиректом?

**Зачем спрашивают.** Выбор между 301 и 302 влияет на кэширование (браузер vs сервер), SEO и возможность аналитики. Это часто задаваемый вопрос о trade-off.

**Короткий ответ.** 301 (Moved Permanently) — браузер кэширует редирект, не запрашивает сервер второй раз. 302 (Found/Temporary Redirect) — каждый раз идёт на сервер. Для аналитики нужен 302, для минимальной нагрузки — 301.

**Детальный разбор.**

**HTTP редиректы:**
```
301 Moved Permanently
├─ Семантика: URL переместился навсегда
├─ Кэширование: Браузер кэширует на длительное время
├─ Second request: Браузер напрямую идёт на target URL, не запрашивая короткую ссылку
└─ Use: максимальная производительность, но теряем аналитику на repeated

302 Found (Temporary Redirect)
├─ Семантика: URL временно переместился
├─ Кэширование: Браузер не кэширует (по умолчанию)
├─ Second request: Браузер запрашивает короткую ссылку → сервер редиректит
└─ Use: отслеживание кликов, гибкость в смене target URL

Поток запросов с 301:
┌─────────────┐
│  Client 1   │
│  GET /abc   │
└──────┬──────┘
       │
       ▼
┌──────────────────────────────┐
│ Server returns 301           │
│ Location: https://target...  │
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Browser CACHES this redirect │
│ (обычно на годы или forever) │
└──────────────────────────────┘

Повторный запрос (Client 1):
┌─────────────┐
│  Client 1   │
│  GET /abc   │
└──────┬──────┘
       │
    ┌──▼──┐
    │ HIT │ ← Браузер имеет кэш!
    │ CACHE
    └──┬──┘
       │
       ▼
┌──────────────────────────────┐
│ Browser directly redirects   │
│ to https://target...         │
│ (НЕ отправляет запрос на     │
│  короткую ссылку!)           │
└──────────────────────────────┘

Поток запросов с 302:
┌─────────────┐
│  Client 1   │
│  GET /abc   │
└──────┬──────┘
       │
       ▼
┌──────────────────────────────┐
│ Server returns 302           │
│ Location: https://target...  │
└──────────────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Browser does NOT cache       │
└──────────────────────────────┘

Повторный запрос (Client 1):
┌─────────────┐
│  Client 1   │
│  GET /abc   │
└──────┬──────┘
       │
       ▼ (повторный запрос!)
┌──────────────────────────────┐
│ Server returns 302 again     │
│ → Analytics event recorded   │
│ → Может обновить target      │
└──────────────────────────────┘
```

**Сравнение:**

```
┌─────────────────────────┬──────────────────────┬──────────────────────┐
│ Характеристика          │ 301 Permanent        │ 302 Temporary         │
├─────────────────────────┼──────────────────────┼──────────────────────┤
│ Server Load             │ Ниже (браузер кэш)   │ Выше (всегда идёт)   │
│ Analytics               │ Теряем после кэша    │ Есть всегда           │
│ Target URL flexibility  │ Нельзя менять        │ Можно менять          │
│ Bandwidth consumption   │ Ниже (меньше req)    │ Выше                  │
│ SEO impact              │ ✓ Передаёт authority │ ✗ Не передаёт         │
│ Browser caching         │ Обычно forever       │ Обычно нет            │
└─────────────────────────┴──────────────────────┴──────────────────────┘
```

**Реальные цифры:**

```
Пример: bit.ly (популярный shortener)
- Использует 301 для большинства ссылок
- Причина: снизить нагрузку на сервер и улучшить latency

Пример: analytics-heavy сервис
- Используют 302 для всех ссылок
- Причина: каждый клик → запись в БД

Пример: e-commerce
- Могут использовать 307 (Temporary Redirect, сохраняет метод)
  чтобы гарантировать, что POST запрос не станет GET
```

**Пример кода:**

```python
from flask import Flask, redirect, request
import time

app = Flask(__name__)

@app.route('/<short_code>', methods=['GET'])
def redirect_to_long_url(short_code):
    # Получаем URL из БД/кэша
    long_url = get_long_url(short_code)

    if not long_url:
        return 404

    # Опция 1: 301 (Permanent) — минимальная нагрузка
    if should_use_permanent_redirect():
        return redirect(long_url, code=301)

    # Опция 2: 302 (Temporary) — аналитика, гибкость
    else:
        # Асинхронно записываем клик
        analytics_queue.put({
            'short_code': short_code,
            'timestamp': time.time(),
            'user_agent': request.headers.get('User-Agent'),
            'ip': request.remote_addr,
            'referer': request.referrer,
        })
        return redirect(long_url, code=302)

def should_use_permanent_redirect() -> bool:
    """
    Логика выбора между 301 и 302:
    - 301: для личных/stable ссылок (минимизировать нагрузку)
    - 302: для внутренних/trackable ссылок (полная аналитика)
    """
    # В реальности это может быть параметр в БД
    return False  # Используем 302 для большинства

# Оптимизация: кэширование на Redis
@app.route('/<short_code>')
def redirect_optimized(short_code):
    # Быстрая дорожка: cache hit → 302 redirect
    long_url = cache.get(short_code)

    if long_url:
        # Even faster: inline analytics to queue (non-blocking)
        asyncio.create_task(record_click(short_code))
        return redirect(long_url, code=302)

    # Медленная дорожка: DB hit
    long_url = db.get(short_code)
    if long_url:
        cache.set(short_code, long_url, ttl=3600)
        asyncio.create_task(record_click(short_code))
        return redirect(long_url, code=302)

    return 404
```

**Типичные ошибки.**
- Использовать 301 и потом удивляться, почему аналитика не работает
- Использовать 302 везде и перегруживать сервер
- Забыть, что браузер кэширует 301 (может переустановить)
- Не учитывать SEO implications (301 передаёт authority)

**На интервью.**
- Объясни разницу и когда использовать каждый
- Обсуди trade-off: нагрузка vs аналитика
- Рекомендуй гибридный подход: 302 для большинства, 301 для popular/permanent
- Follow-up: «Как минимизировать overhead 302?» — кэширование, асинхронная аналитика, load balancing

---

### 8. Как масштабировать URL shortener?

**Зачем спрашивают.** Масштабирование — ключевая часть system design. Интервьюер проверяет, как вы думаете о bottleneck и как их устранять.

**Короткий ответ.** Основные узкие места: (1) Write bottleneck — используем pre-generated keys или distributed ID; (2) Read bottleneck — кэш и CDN; (3) DB bottleneck — sharding. Каждый компонент может масштабироваться горизонтально (добавляем инстансы).

**Детальный разбор.**

**Bottleneck анализ:**

```
┌──────────────────────────────────────────────────────────────────┐
│                     URL Shortener at Scale                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Current: 100M DAU → 115K avg QPS (read), 1.2K (write)          │
│  Target: 1B DAU  → 1.15M avg QPS (read), 12K (write)            │
│                                                                  │
│  Bottlenecks:                                                    │
│  1. API Servers — scale horizontally (add instances)            │
│  2. Cache — scale Redis cluster                                  │
│  3. Database — scale via sharding                                │
│  4. Network — CDN for popular URLs                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Масштабирование по компонентам:**

**1. API Server масштабирование**
```
Текущая архитектура:
┌──────────────┐
│ Load Balancer│
└───────┬──────┘
        │
    ┌───┴───┐
    │       │
┌───▼──┐ ┌──▼───┐
│API 1 │ │API 2 │ ← 2 инстанса
└──────┘ └──────┘

Масштабирование (10x traffic):
┌──────────────┐
│ Load Balancer│
└────────┬─────┘
         │
    ┌────┴─────┐
    │           │
┌───▼──┐ ┌──▼───┐ ... ┌──────┐
│API 1 │ │API 2 │     │API 20│ ← 20 инстансов
└──────┘ └──────┘     └──────┘

Load Balancer: Round-robin, least connections, consistent hashing
```

**Стратегия:** Stateless API servers → добавляем инстансы по мере роста трафика

**2. Cache масштабирование (Redis)**
```
Текущая архитектура:
┌─────────────┐
│  Redis      │ ← Single instance
│  (4-8GB)    │ (bottleneck!)
└─────────────┘

Масштабирование:
Redis Cluster (Sharded):
┌──────────────┬──────────────┬──────────────┐
│Redis Shard 1 │Redis Shard 2 │Redis Shard 3 │
│ (slot 0-5460)│(slot 5461-   │(slot 10923-  │
│              │ 10922)       │ 16383)       │
└──────────────┴──────────────┴──────────────┘
        │              │              │
┌───────┴──────────────┴──────────────┴────────┐
│  Client (Redis Client with cluster aware)    │
│  Хеширует key → определяет shard → запрос    │
└────────────────────────────────────────────────┘

Распределение: hash(short_code) % num_shards
```

**Пример:**
```python
import redis.cluster

# Cluster mode Redis
redis_cluster = redis.cluster.StrictRedisCluster(
    startup_nodes=[
        {"host": "shard1.example.com", "port": 6379},
        {"host": "shard2.example.com", "port": 6379},
        {"host": "shard3.example.com", "port": 6379},
    ],
    decode_responses=True
)

# Кэш hit/miss автоматически роутируется в правильный shard
redis_cluster.set("url:abc123", "https://example.com/long/url")
long_url = redis_cluster.get("url:abc123")
```

**3. Database масштабирование (Sharding)**
```
Текущая БД (monolithic):
┌────────────────────────┐
│   PostgreSQL DB        │
│   182.5B URLs          │ ← Не влезает в один сервер
│   ~91 TB storage       │
└────────────────────────┘

Sharding по hash(short_code) % N:
┌──────────────────┬──────────────────┬──────────────────┐
│ DB Shard 0       │ DB Shard 1       │ DB Shard 2       │
│ hash % 3 == 0    │ hash % 3 == 1    │ hash % 3 == 2    │
│ (~30 TB)         │ (~30 TB)         │ (~30 TB)         │
└──────────────────┴──────────────────┴──────────────────┘

Клиент:
1. Вычисляет shard_id = hash(short_code) % 3
2. Маршрутизирует запрос в правильный shard
```

**Пример:**
```python
class ShardedDatabase:
    def __init__(self, num_shards=3, shard_hosts=None):
        self.num_shards = num_shards
        self.shards = [
            psycopg2.connect(host)
            for host in (shard_hosts or get_default_hosts())
        ]

    def get_shard_id(self, short_code: str) -> int:
        """Определяет shard на основе key"""
        return hash(short_code) % self.num_shards

    def get_shard_connection(self, short_code: str):
        """Возвращает connection к нужному shard"""
        shard_id = self.get_shard_id(short_code)
        return self.shards[shard_id]

    def get_url(self, short_code: str) -> Optional[str]:
        """Fetches from correct shard"""
        shard = self.get_shard_connection(short_code)
        cursor = shard.cursor()
        cursor.execute(
            "SELECT long_url FROM urls WHERE short_code = %s",
            (short_code,)
        )
        result = cursor.fetchone()
        return result[0] if result else None

    def insert_url(self, short_code: str, long_url: str):
        """Inserts into correct shard"""
        shard = self.get_shard_connection(short_code)
        cursor = shard.cursor()
        cursor.execute(
            "INSERT INTO urls (short_code, long_url) VALUES (%s, %s)",
            (short_code, long_url)
        )
        shard.commit()
```

**4. CDN для популярных URLs**
```
Текущая архитектура:
┌─────────────────┐
│  Client (Asia)  │ ──────────────────┐
└─────────────────┘                   │
                                      │
┌─────────────────┐                   │
│  Client (US)    │ ──────────────────┤────► API Server (US)
└─────────────────┘                   │
                                      │
┌─────────────────┐                   │
│  Client (EU)    │ ──────────────────┘
└─────────────────┘

Проблемы:
- Latency для Asia (300+ ms)
- Всегда идёт в один data center

Масштабирование с CDN:
┌─────────────────┐         ┌──────────────┐
│  Client (Asia)  │────────▶│ CDN POP Asia │────┐
└─────────────────┘         └──────────────┘    │
                                                │
┌─────────────────┐         ┌──────────────┐    │
│  Client (US)    │────────▶│ CDN POP US   │────┼──► API Server
└─────────────────┘         └──────────────┘    │    (Master)
                                                │
┌─────────────────┐         ┌──────────────┐    │
│  Client (EU)    │────────▶│ CDN POP EU   │────┘
└─────────────────┘         └──────────────┘

CDN кэширует:
- 301 редиректы (браузер кэширует на длительное время)
- 302 редиректы (по умолчанию TTL зависит от Cache-Control header)

Для 302: Set Cache-Control: public, max-age=300 (5 mins)
```

**5. Geographical Distribution**
```
Текущая архитектура (Single region):
┌────────────────┐
│  All traffic   │
│  → US-DC only  │
└────────────────┘

Масштабирование (Multiple regions):
┌────────────────────────────────────────────────────────────┐
│                    Global DNS / GeoDNS                     │
└─────┬──────────────────────────────────────────────────────┘
      │
   ┌──┴──────────────┬──────────────┬──────────────┐
   │                 │              │              │
   ▼                 ▼              ▼              ▼
┌─────────┐    ┌──────────┐   ┌──────────┐   ┌──────────┐
│ US-DC   │    │ EU-DC    │   │ APAC-DC  │   │ LATAM-DC │
│ (master)│    │(replica) │   │(replica) │   │(replica) │
└─────────┘    └──────────┘   └──────────┘   └──────────┘
     │              │             │             │
     └──────────────┼─────────────┼─────────────┘
                    │
            ┌───────▼────────┐
            │ Replication    │
            │ Pipeline       │
            │ (MySQL/Postgres│
            │  bin logs)     │
            └────────────────┘

Write: всегда в Master (US-DC)
Read: из ближайшего региона (replica или cache)
```

**Полная архитектура (Scaled to 1B DAU):**

```
                        ┌──────────────────┐
                        │   Global DNS     │
                        └────────┬─────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
    ┌─────────┐            ┌──────────┐          ┌──────────┐
    │ CDN NA  │            │ CDN EU   │          │ CDN APAC │
    │(Edge)   │            │(Edge)    │          │(Edge)    │
    └────┬────┘            └────┬─────┘          └────┬─────┘
         │                      │                     │
         │   ┌──────────────────┴─────────────────┐   │
         │   │                                    │   │
         ▼   ▼                                    ▼   ▼
    ┌─────────────────┐                   ┌─────────────────┐
    │  API Tier       │                   │  API Tier       │
    │  (US Region)    │◄──────────────────│  (EU Region)    │
    │  - 50 instances │                   │  - 30 instances │
    └────────┬────────┘                   └────────┬────────┘
             │                                     │
        ┌────┴──────────────────┬──────────────────┘
        │                       │
        ▼                       ▼
    ┌─────────────────┐    ┌──────────────┐
    │ Redis Cluster   │    │ Redis Cluster│
    │ (3 shards)      │    │ (3 shards)   │
    └────────┬────────┘    └──────┬───────┘
             │                    │
             │            ┌───────▼──────┐
             │            │ Replication  │
             │            │ (Read-only)  │
             └────┬───────┘              │
                  │                      │
                  ▼                      ▼
             ┌──────────────────────────────────┐
             │ Primary DB (Master Write)        │
             │ - 4 Shards × PostgreSQL 16       │
             │ - ~22 TB per shard               │
             │ - SSD storage, replication       │
             └──────────────────────────────────┘
                    ▲                  │
                    │                  │
        ┌───────────┼──────────────────┐
        │           │                  │
        │    ┌──────▼──────┐   ┌───────▼────┐
        │    │ Read       │   │ Read       │
        │    │ Replicas   │   │ Replicas   │
        │    │ (EU Region)│   │ (APAC)     │
        │    └────────────┘   └────────────┘
        │
    ┌───▼────────────┐
    │ Message Queue  │
    │ (Kafka)        │
    │ - Analytics    │
    │ - Indexing     │
    └────────────────┘
```

**Типичные ошибки.**
- Попытаться масштабировать монолит без sharding (hits ceiling)
- Добавить слишком много слоёв кэширования (complex cache invalidation)
- Забыть про consistent hashing (переhashing при добавлении шардов)
- Не мониторить bottleneck (масштабируем не то что нужно)

**На интервью.**
- Начни с текущего состояния (100M DAU)
- Определи bottleneck (DB, cache, network, CPU)
- Рекомендуй решение для каждого bottleneck
- Упомяни consistent hashing для добавления шардов без rehashing
- Follow-up: «Как добавить shard без downtime?» — consistent hashing, shadow sharding

---

### 9. Как реализовать аналитику кликов?

**Зачем спрашивают.** Аналитика — часто требуется на интервью. Требует асинхронной обработки, чтобы не замедлить редиректы. Интервьюер проверяет понимание async patterns и event-driven архитектуры.

**Короткий ответ.** При редиректе не блокируем на аналитике. Отправляем click event в message queue (Kafka, RabbitMQ). Consumer обрабатывает асинхронно: агрегирует clicks (Redis), периодически пишет в DB. Даёт истинно real-time аналитику без замедления редиректов.

**Детальный разбор.**

**Synchronous approach (НЕПРАВИЛЬНО):**
```python
def redirect(short_code: str) -> str:
    # Получаем URL
    long_url = cache.get(short_code) or db.get(short_code)

    # ❌ БЛОКИРУЕМ на аналитике!
    db.increment_click_count(short_code)  # 10-100ms latency!
    db.insert_click_event({
        'short_code': short_code,
        'timestamp': now(),
        'ip': request.ip,
        'user_agent': request.user_agent,
    })  # Ещё 10-100ms!

    # После 200ms задержки возвращаем редирект
    return redirect(long_url, code=302)

# Проблемы:
# - Latency 200ms+ вместо <10ms
# - DB перегружена аналитикой
# - User experience страдает
```

**Asynchronous approach (ПРАВИЛЬНО):**
```
┌──────────────────────────────────────────────────────────────┐
│                    Click Event Pipeline                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Redirect Request (Fast path):                            │
│     Client → API Server (check cache/db) → Redirect          │
│     Latency: <10ms                                           │
│                                                              │
│  2. Click Event (Async):                                     │
│     API → Kafka (fast, non-blocking)                         │
│     Latency: <1ms                                            │
│                                                              │
│  3. Analytics Consumer (Background):                         │
│     Kafka → Process events → Aggregate in Redis →            │
│     Periodically flush to DB                                 │
│                                                              │
│  4. Analytics Dashboard:                                     │
│     Redis (real-time) + DB (historical)                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Архитектура:**

```python
from kafka import KafkaProducer, KafkaConsumer
import json

class URLShortenerAPI:
    def __init__(self, kafka_bootstrap_servers):
        self.kafka_producer = KafkaProducer(
            bootstrap_servers=kafka_bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
        self.db = get_db()
        self.cache = get_redis()

    def redirect(self, short_code: str):
        """
        Fast path: redirect быстро, аналитика асинхронно
        """
        # 1. Get URL (cache или DB)
        long_url = self._get_long_url(short_code)
        if not long_url:
            return 404

        # 2. FAST: Send click event to Kafka (non-blocking)
        self.kafka_producer.send_async(
            topic='url_clicks',
            value={
                'short_code': short_code,
                'timestamp': int(time.time() * 1000),
                'ip': request.remote_addr,
                'user_agent': request.headers.get('User-Agent'),
                'referer': request.referrer,
                'country': get_country_from_ip(request.remote_addr),
            },
            callback=self._on_send_success,
            errback=self._on_send_error,
        )

        # 3. Redirect (клиент получит редирект почти мгновенно)
        return redirect(long_url, code=302)

    def _on_send_success(self, metadata):
        """Callback при успешной отправке в Kafka"""
        pass  # Logging опционально

    def _on_send_error(self, exc):
        """Callback при ошибке отправки в Kafka"""
        logger.error(f"Failed to send click event: {exc}")


class AnalyticsConsumer:
    """
    Слушает Kafka, обрабатывает click events,
    агрегирует в Redis, флешит в DB
    """

    def __init__(self, kafka_bootstrap_servers, db, cache):
        self.kafka_consumer = KafkaConsumer(
            'url_clicks',
            bootstrap_servers=kafka_bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            group_id='analytics_consumer_group',
            auto_offset_reset='earliest',
        )
        self.db = db
        self.cache = cache
        self.batch_size = 1000
        self.flush_interval_secs = 60
        self.events_batch = []
        self.last_flush = time.time()

    def run(self):
        """Main consumer loop"""
        for message in self.kafka_consumer:
            event = message.value

            # 1. Update Redis counters (real-time)
            self._update_redis_counters(event)

            # 2. Batch events for DB insert
            self.events_batch.append(event)

            # 3. Flush to DB periodically
            if (len(self.events_batch) >= self.batch_size or
                time.time() - self.last_flush > self.flush_interval_secs):
                self._flush_to_db()

    def _update_redis_counters(self, event: dict):
        """
        Обновляем счётчики в Redis (real-time аналитика)
        """
        short_code = event['short_code']
        timestamp = event['timestamp']
        country = event.get('country', 'UNKNOWN')

        # Increment click count
        self.cache.incr(f"clicks:{short_code}")

        # Track by country (for geographic analytics)
        self.cache.incr(f"clicks:{short_code}:country:{country}")

        # Update last accessed timestamp
        self.cache.set(f"last_click:{short_code}", timestamp)

        # Daily buckets (для трендов)
        day = datetime.fromtimestamp(timestamp/1000).strftime("%Y-%m-%d")
        self.cache.incr(f"clicks:{short_code}:date:{day}")

    def _flush_to_db(self):
        """
        Пишем батч событий в БД для исторических данных
        """
        if not self.events_batch:
            return

        # Insert batch of events
        self.db.execute_many("""
            INSERT INTO clicks (url_id, timestamp, ip_address, user_agent, country)
            SELECT urls.id, %(timestamp)s, %(ip)s, %(user_agent)s, %(country)s
            FROM urls WHERE urls.short_code = %(short_code)s
        """, self.events_batch)

        self.db.commit()
        self.events_batch = []
        self.last_flush = time.time()

        logger.info(f"Flushed analytics to DB")


class AnalyticsAPI:
    """Endpoint для получения аналитики"""

    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def get_analytics(self, short_code: str) -> dict:
        """
        Возвращает real-time аналитику из Redis + исторические из DB
        """
        # Real-time counts (from Redis)
        total_clicks = self.cache.get(f"clicks:{short_code}") or 0
        last_click = self.cache.get(f"last_click:{short_code}") or None

        # Geographic breakdown (from Redis)
        countries = {}
        for key in self.cache.scan_iter(f"clicks:{short_code}:country:*"):
            country = key.split(':')[-1]
            countries[country] = int(self.cache.get(key) or 0)

        # Historical data (from DB, if needed)
        # Обычно не нужно если Redis достаточно

        return {
            'short_code': short_code,
            'total_clicks': int(total_clicks),
            'last_click': last_click,
            'countries': countries,
        }

    def get_daily_trend(self, short_code: str, days: int = 30) -> list:
        """Дневные тренды из Redis"""
        trend = []
        today = datetime.now().date()

        for i in range(days, -1, -1):
            date = today - timedelta(days=i)
            date_str = date.strftime("%Y-%m-%d")
            clicks = int(self.cache.get(f"clicks:{short_code}:date:{date_str}") or 0)
            trend.append({
                'date': date_str,
                'clicks': clicks,
            })

        return trend
```

**Масштабирование аналитики:**

```
Kafka Partitioning:
┌─────────────────────────────────────────────────────────────┐
│                  Topic: url_clicks                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Partition 0: short_codes hashing to 0 % num_partitions   │
│  Partition 1: short_codes hashing to 1 % num_partitions   │
│  ...                                                        │
│  Partition N-1: short_codes hashing to N-1 % num_partitions│
│                                                             │
│  Consumer Group: multiple consumers, each reads partition   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Пример:
- 10 partitions
- 5 consumers
- Каждый consumer читает 2 партиции
- При 100K events/sec → 10K events/sec per consumer (manageable)
```

**Типичные ошибки.**
- Синхронная запись в БД для каждого клика
- Забыть про backpressure (Kafka queue переполняется)
- Не батчить writes в БД (много round-trips)
- Потерять события при падении Consumer

**На интервью.**
- Объясни разницу sync vs async
- Покажи architecture с Kafka/queue
- Упомяни Redis для real-time counts, DB для historical
- Follow-up: «Как гарантировать at-least-once delivery?» — Kafka offset management + idempotent DB writes

---

### 10. Как обеспечить high availability?

**Зачем спрашивают.** HA — критично для production. Интервьюер проверяет знание replication, failover, disaster recovery.

**Короткий ответ.** Репликация на уровне каждого компонента: DB (master-slave), Cache (cluster), API (stateless). Multi-AZ deployment, load balancing, health checks, circuit breakers. Цель: 99.99% uptime (5.26 минут downtime в год).

**Детальный разбор.**

**Database HA:**

```
Master-Slave Replication:
┌──────────────┐
│ Master (RW)  │ (US-DC primary)
│              │
│ bin log ──┐  │
└──────────┼──┘
           │
        ┌──▼────────────────────────────┐
        │   Replication Pipeline         │
        │   (Network, ordered, durable)  │
        └──────────┬─────────────────────┘
                   │
        ┌──────────▼──────────┐
        │ Slave (R) — Replica │ (EU-DC)
        │                     │
        │ Apply bin log       │
        │ (eventually        │
        │  consistent)       │
        └─────────────────────┘

Failover (when Master dies):
1. Detect master failure (heartbeat timeout)
2. Elect new master from replicas
3. Update DNS to point to new master
4. Replay remaining bin logs on replicas
```

**Пример master-slave PostgreSQL:**

```python
import psycopg2

class HA_Database:
    def __init__(self):
        self.master = psycopg2.connect(
            host='master.example.com',
            dbname='urls',
            user='admin'
        )
        self.replicas = [
            psycopg2.connect(
                host=f'replica{i}.example.com',
                dbname='urls',
                user='reader'
            )
            for i in range(3)  # 3 read replicas
        ]
        self.replica_index = 0

    def write(self, query, params):
        """Write always goes to master"""
        cursor = self.master.cursor()
        cursor.execute(query, params)
        self.master.commit()

    def read(self, query, params):
        """Read from replica (round-robin)"""
        replica = self.replicas[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replicas)

        cursor = replica.cursor()
        cursor.execute(query, params)
        return cursor.fetchall()
```

**Cache HA:**

```
Redis Cluster (Sharded + Replicated):
┌──────────────┬──────────────┬──────────────┐
│Shard 1 (3.3K)│Shard 2 (3.3K)│Shard 3 (3.3K)│
├──────────────┼──────────────┼──────────────┤
│  Master      │  Master      │  Master      │
│      ↓       │      ↓       │      ↓       │
│  Replica     │  Replica     │  Replica     │
│  Replica     │  Replica     │  Replica     │
└──────────────┴──────────────┴──────────────┘

Каждый shard имеет:
- 1 master (writes)
- 2+ replicas (reads, failover)
- Replication (synchronous or async)
```

**API Server HA:**

```
┌──────────────────────────────────────┐
│       Load Balancer (DNS + LB)       │
│  - Health checks (heartbeat)         │
│  - Auto-remove dead servers          │
│  - Auto-add new servers              │
└───────────────┬──────────────────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ API 1   │ │ API 2   │ │ API 3   │
│ ✓ alive │ │ ✓ alive │ │ ✗ dead  │
└─────────┘ └─────────┘ └─────────┘

Failover:
1. Health check не пройден для API 3
2. LB исключает API 3 из rotation
3. Traffic идёт только на API 1, 2
4. Auto-scaling спинит новый инстанс
```

**Circuit Breaker Pattern (для API dependency):**

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"       # Нормально, запросы идут
    OPEN = "open"          # Ошибки, запросы отклоняются
    HALF_OPEN = "half_open" # Пытаемся восстановиться

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED

    def call(self, func, *args, **kwargs):
        """Запускаем функцию, контролируя состояние"""

        if self.state == CircuitState.OPEN:
            # Проверяем, прошло ли время timeout
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
                self.failure_count = 0
            else:
                raise Exception("Circuit is OPEN, failing fast")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage:
breaker = CircuitBreaker(failure_threshold=5, timeout=30)

def call_external_api():
    return breaker.call(fetch_from_external_service)

try:
    result = call_external_api()
except Exception as e:
    logger.error(f"API call failed: {e}")
    # Fallback: return cached value or default
    return get_cached_fallback()
```

**Multi-AZ Deployment:**

```
┌────────────────────────────────────────────────────────────────┐
│                      Global Architecture                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  US Region (Primary)              EU Region (Secondary)        │
│  ┌────────────────────┐           ┌────────────────────┐      │
│  │ US-AZ-A (active)   │           │ EU-AZ-A (replica)  │      │
│  │ - API (10 instances)           │ - API (5 instances)│      │
│  │ - DB Master        │◄──────────│ - DB Replica       │      │
│  │ - Cache (3 shards) │           │ - Cache (3 shards) │      │
│  └────────────────────┘           └────────────────────┘      │
│           │                                │                   │
│           │  Replication Pipeline          │                   │
│           └────────────┬────────────────────┘                   │
│                        │                                       │
│  US-AZ-B (standby)     │     EU-AZ-B (standby)                │
│  ┌────────────────────┐│     ┌────────────────────┐           │
│  │ - API (3 instances)││     │ - API (2 instances)│           │
│  │ - DB Replica       │└────▶│ - DB Replica       │           │
│  │ - Cache (local)    │      │ - Cache (local)    │           │
│  └────────────────────┘      └────────────────────┘           │
│                                                                │
└────────────────────────────────────────────────────────────────┘

Failover scenarios:
1. API instance fails → LB removes, auto-scale spins new
2. Cache cluster fails → failover to secondary cache
3. Master DB fails → promote replica to master
4. Entire US region fails → DNS switches to EU region
```

**Monitoring & Alerting:**

```python
class HealthChecker:
    """Continuously monitors system health"""

    def health_check(self):
        """Health check endpoint"""
        checks = {
            'database': self._check_db(),
            'cache': self._check_cache(),
            'api': self._check_api(),
            'replication_lag': self._check_replication_lag(),
        }

        all_ok = all(checks.values())
        return {'status': 'healthy' if all_ok else 'unhealthy', 'checks': checks}

    def _check_db(self):
        """Check master and replicas"""
        try:
            self.db.execute("SELECT 1")
            return True
        except:
            return False

    def _check_cache(self):
        """Check Redis cluster"""
        try:
            self.cache.ping()
            return True
        except:
            return False

    def _check_replication_lag(self):
        """Check master-replica lag"""
        master_pos = self.db.get_binlog_position()
        replica_pos = self.replica_db.get_binlog_position()
        lag_bytes = master_pos - replica_pos

        # Alert if lag > 1 MB
        return lag_bytes < 1_000_000

    def _check_api(self):
        """Check API responsiveness"""
        try:
            response = requests.get('http://localhost:8080/health', timeout=1)
            return response.status_code == 200
        except:
            return False
```

**Типичные ошибки.**
- Реплика отстаёт, клиент читает outdated data
- Не настроить здоровые check intervals (слишком редко = долгий failover)
- Забыть про split-brain (оба master и replica думают что они master)
- Не тестировать failover сценарии

**На интервью.**
- Объясни master-slave replication
- Упомяни circuit breaker для graceful degradation
- Рекомендуй multi-AZ deployment для geographic HA
- Follow-up: «Как избежать split-brain?» — consensus algorithm (Raft) или external coordinator (Zookeeper)

---

### 11. Какие дополнительные требования часто спрашивают?

**Зачем спрашивают.** После основной архитектуры интервьюер обычно задаёт follow-up для углубления знаний.

**Короткий ответ.** Частые требования: custom aliases (validation, uniqueness), URL expiration (TTL, cleanup), abuse prevention (rate limiting, blacklist), SEO consideration (301 vs 302), API versioning, monitoring/logging, cost optimization.

**Детальный разбор.**

**Custom Aliases:**
```python
def create_short_url_with_alias(long_url: str, custom_alias: str = None) -> str:
    """
    Позволяет пользователю выбрать alias вместо случайного кода
    """
    if custom_alias:
        # Validation
        if not is_valid_alias(custom_alias):
            raise ValueError("Invalid alias format")

        # Check uniqueness
        if db.code_exists(custom_alias):
            raise ValueError("Alias already taken")

        short_code = custom_alias
    else:
        # Generate random code
        short_code = generate_unique_code()

    db.insert(short_code, long_url)
    return short_code

def is_valid_alias(alias: str) -> bool:
    """Validate alias: length, characters, reserved words"""
    if not (3 <= len(alias) <= 20):
        return False
    if not alias.isalnum():  # Only alphanumeric
        return False

    reserved = {'api', 'admin', 'health', 'metrics'}
    if alias in reserved:
        return False

    return True
```

**URL Expiration:**
```python
def create_short_url_with_expiration(
    long_url: str,
    expires_in_days: int = None
) -> str:
    """
    URLs могут истекать (обычно для temporary links)
    """
    short_code = generate_unique_code()

    expires_at = None
    if expires_in_days:
        expires_at = datetime.now() + timedelta(days=expires_in_days)

    db.insert({
        'short_code': short_code,
        'long_url': long_url,
        'expires_at': expires_at,
    })

    # Cleanup task: периодически удаляем истёкшие
    # (можно использовать TTL на БД, Redis, или batch cleanup)
    return short_code

def redirect_with_expiration_check(short_code: str) -> Optional[str]:
    """Проверяем expiration перед редиректом"""
    url_record = db.get(short_code)

    if not url_record:
        return None  # 404

    if url_record.expires_at and url_record.expires_at < datetime.now():
        # Optional: мягкое удаление (mark as expired)
        db.mark_expired(short_code)
        return None  # 410 Gone

    return url_record.long_url
```

**Abuse Prevention (Rate Limiting + Blacklist):**
```python
from ratelimit import RateLimiter

class AbusePreventionService:
    def __init__(self):
        self.rate_limiter = RateLimiter(
            max_calls=100,  # 100 requests
            time_period=3600  # per hour per IP
        )
        self.blacklist = RedisSet('url_blacklist')

    def check_before_create(self, long_url: str, ip: str) -> bool:
        """
        Перед созданием shortener проверяем:
        1. Rate limit (не более 100 в час)
        2. Blacklist (known spam/malware)
        """
        # 1. Rate limiting
        if not self.rate_limiter.is_allowed(ip):
            raise RateLimitExceeded(f"Too many requests from {ip}")

        # 2. Blacklist check
        if long_url in self.blacklist:
            raise ValueError("URL is blacklisted (malware/spam)")

        # 3. URL validation
        if not self._is_safe_url(long_url):
            raise ValueError("URL failed safety checks")

        return True

    def _is_safe_url(self, url: str) -> bool:
        """Check for phishing, malware domains"""
        domain = extract_domain(url)

        # Check against known bad domains
        if domain in get_malware_database():
            return False

        # Check domain reputation score
        reputation = get_domain_reputation(domain)
        if reputation.score < 0.3:  # Low reputation
            return False

        return True
```

**SEO Considerations:**
```python
def create_short_url(long_url: str, use_permanent_redirect: bool = False) -> str:
    """
    301 vs 302:
    - 301: лучше для SEO (передаёт link juice),
            но нельзя менять target
    - 302: гибче, но не передаёт authority
    """
    short_code = generate_unique_code()

    db.insert({
        'short_code': short_code,
        'long_url': long_url,
        'redirect_type': 301 if use_permanent_redirect else 302,
    })

    return short_code

def redirect(short_code: str):
    url_record = db.get(short_code)
    redirect_code = url_record.get('redirect_type', 302)

    return redirect(url_record.long_url, code=redirect_code)
```

**Типичные ошибки.**
- Custom alias без validation (коллизии, injection)
- Expiration без cleanup (таблица растёт бесконечно)
- Отсутствие rate limiting (DDoS/abuse)
- Игнорировать SEO требования (301 vs 302)

**На интервью.**
- Обсуди customer requirements (хотят ли custom aliases?)
- Объясни rate limiting strategy (per IP, per user, global)
- Рекомендуй monitoring и logging для abuse detection
- Follow-up: «Как обнаружить phishing links?» — malware database, domain reputation scoring, user reports

---

## Резюме подходов

```
┌─────────────────────────────────────────────────────────────┐
│           URL Shortener Design Decisions                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. Code Generation:                                         │
│    → Snowflake ID (good balance of perf & simplicity)      │
│    → Pre-generated keys (if extreme perf needed)            │
│                                                             │
│ 2. Database:                                                │
│    → SQL + Sharding (if need analytics & complex queries)  │
│    → NoSQL (if pure key-value, extreme scale)              │
│    → Hybrid (NoSQL for URLs, SQL for analytics)            │
│                                                             │
│ 3. Caching:                                                 │
│    → Redis cluster (multi-AZ, replication)                 │
│    → Cache hot 20% (80/20 rule)                            │
│    → TTL ~1-24 hours                                       │
│                                                             │
│ 4. Redirect:                                                │
│    → 301 for permanent, predictable target                 │
│    → 302 for analytics & flexibility                       │
│                                                             │
│ 5. Analytics:                                               │
│    → Async (Kafka/queue) not blocking                       │
│    → Redis for real-time, DB for historical               │
│                                                             │
│ 6. HA/Failover:                                            │
│    → Master-slave replication                              │
│    → Multi-AZ deployment                                   │
│    → Circuit breakers & health checks                      │
│                                                             │
│ 7. Scaling:                                                │
│    → Horizontal scaling (stateless API)                    │
│    → Sharding (database)                                   │
│    → CDN (for popular URLs)                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Практика

1. **Implement a simple URL shortener** — с hash-based генерацией и SQLite, добавить 302 редиректы

2. **Add Redis caching** — интегрировать Redis для кэширования часто используемых URL

3. **Implement analytics** — добавить счётчик кликов (асинхронно через queue или sync простой версии)

4. **Horizontal scaling simulation** — реализовать sharding логику для распределения данных

5. **HA testing** — симулировать failure сценарии (DB down, cache fail) и проверить failover

6. **Load testing** — generate traffic, мониторить bottleneck (DB, cache, network)

---

## Дополнительные материалы

- [System Design Interview](https://www.systemdesigninterview.com) — подробный разбор URL shortener с диаграммами
- [Designing Data-Intensive Applications](https://dataintensive.net/) — классическая книга про scaling и replication
- [High Performance MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/) — sharding, replication, optimization
- [Kafka Design](https://kafka.apache.org/documentation/) — асинхронная обработка events
- [Redis Best Practices](https://redis.io/docs/management/optimization/) — кэширование и масштабирование

---

← [00-design-fundamentals](./00-design-fundamentals.md) | [Трек System Design](./README.md) | [02-rate-limiter](./02-rate-limiter.md) →
