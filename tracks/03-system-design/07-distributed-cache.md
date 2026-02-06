# 07 — Distributed Cache

Развёрнутые вопросы и ответы про распределённый cache: consistent hashing, eviction policies, invalidation patterns, Redis cluster, cache stampede. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [06-search-autocomplete](./06-search-autocomplete.md) · Следующая тема: [08-file-storage](./08-file-storage.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Distributed Cache** — это хранилище данных в оперативной памяти (RAM), распределённое по нескольким сетевым узлам (серверам). Благодаря хранению в RAM, доступ к данным происходит за микросекунды, что в 100+ раз быстрее обращения к базе данных (50-100 мс). Distributed cache защищает базу данных от перегрузки, перехватывая горячие данные, и позволяет системам выдерживать миллионы запросов в секунду. Redis, Memcached и другие системы построены именно на этом принципе.

**Cache Hit** — это успешное нахождение запрошенного данного в кэше. При cache hit система экономит дорогостоящий запрос в базу данных и возвращает результат почти мгновенно (примерно 0.5-1 мс для Redis). Это лучший сценарий для производительности: чем больше процент hits, тем быстрее и более отзывчива система. Высокий cache hit rate (выше 80%) — признак здоровой и хорошо оптимизированной архитектуры.

**Cache Miss** — это ситуация, когда запрошенные данные отсутствуют в кэше и нужно загружать их из более медленного источника (база данных или disk). Cache miss медленнее, чем hit: если hit занимает 1 мс, то miss может занять 50-100 мс (время на обращение в БД). Серия cache misses называется "холодным запуском" и происходит, например, при перезапуске сервера, когда кэш полностью пуст.

**TTL (Time To Live)** — это время, в течение которого запись остаётся в кэше перед автоматическим удалением. Например, TTL 1 час означает, что запись будет доступна 60 минут, а затем исчезнет. TTL гарантирует свежесть данных: со временем устаревшие данные удаляются и заменяются свежими. Выбор TTL — компромисс: короткий TTL гарантирует свежесть, но повышает долю cache misses; длинный TTL экономит запросы в БД, но рискует хранить старые данные.

**Consistent Hashing** — это алгоритм распределения ключей по узлам кэша на виртуальном кольце, который минимизирует переассоциацию при добавлении или удалении узла. В обычной модели (простой хеш по количеству узлов), добавление одного нового узла требует перехеширования почти всех ключей. В consistent hashing каждый новый узел получает лишь 1/N ключей, остальные остаются на месте. Это критично для production систем, где переассоциация больших объёмов данных вызывает всплески нагрузки.

**Eviction Policy** — это правило, определяющее какие данные удалять из кэша, когда память переполнена. Существуют разные политики: LRU (удаляет давно не используемые), FIFO (удаляет самые старые), LFU (удаляет редко используемые), Random (удаляет случайно). Выбор политики влияет на cache hit rate: LRU часто работает лучше для реальных нагрузок, так как "горячие" данные используются повторно и остаются в памяти.

**LRU (Least Recently Used)** — это политика вытеснения, которая удаляет запись, которая дольше всего не была использована. Если у вас 100 МБ кэша, и вы добавляете новую запись, когда память полна, LRU удалит самую давно обращённую запись. Эта политика хорошо работает на практике, так как предполагает, что данные, которые часто используются, снова будут использованы (принцип локальности). Redis и Memcached позволяют выбирать разные eviction policies.

**Write-Through** — это паттерн записи, при котором обновление происходит одновременно в кэше и в базе данных. Когда клиент обновляет данные, система пишет их сначала в кэш, затем в БД, и только потом отвечает клиенту. Это гарантирует консистентность: если произойдёт сбой, данные в обоих хранилищах будут совпадать. Однако write-through медленнее, так как нужно ждать обновления БД.

**Write-Back (Write-Behind)** — это паттерн записи, при котором обновление происходит в кэше, а в БД отправляется позже (асинхронно). Клиент получает ответ сразу после обновления кэша, что быстрее, но рискует потерей данных: если сервер упадёт до того, как изменения достигнут БД, они будут потеряны. Write-back используется когда нужна высокая скорость, но нельзя гарантировать 100% надёжность (например, счётчики, рекомендации).

**Cache Stampede** (иногда называют "thundering herd") — это явление, когда много запросов одновременно попадают в cache miss после истечения TTL. Представьте, что кэш содержит популярный ключ с TTL 1 час. Когда ключ истекает, первые 10,000 пользователей, обращающихся одновременно, все видят miss и отправляют запросы в БД, вызывая огромный всплеск нагрузки. Решение: использовать блокировку (первый запрос обновляет кэш, остальные ждут) или probabilistic early expiration (обновлять данные немного раньше, чем они совсем устаревают).

**Replication** — это создание нескольких копий (репликаций) данных на разных узлах кэша. Например, если у вас 3-узловой Redis кластер с replication factor 2, каждый ключ хранится минимум на двух узлах. Если один узел упадёт, данные остаются доступны на других, повышая reliability. Replication также позволяет распределять нагрузку на чтение: клиенты могут читать с разных реплик, увеличивая throughput.

**Cache Invalidation** — это активное удаление устаревших данных из кэша, когда обновляется исходный источник. Например, если вы обновили пользовательский профиль в БД, нужно инвалидировать кэш этого профиля, чтобы следующие запросы получили свежие данные. Это сложная задача: можно использовать active invalidation (уведомления при обновлении) или passive invalidation (полагаться на TTL). Phil Karlton famously said "There are only two hard things in Computer Science: cache invalidation and naming things."

**Sharding** — это горизонтальное разделение данных кэша по нескольким узлам по какому-то ключу (обычно по хешу ключа). Если у вас есть 1000 ключей и 10 узлов, sharding распределяет примерно 100 ключей на каждый узел. Это масштабирует общую память и throughput: вместо одного сервера с 16 ГБ RAM, вы имеете 10 серверов с 1.6 ГБ каждый, что в сумме дает 16 ГБ, но обслуживает в 10 раз больше запросов.

---

## Вопросы и разборы

### 1. Зачем нужен distributed cache и как он улучшает архитектуру?

**Зачем спрашивают.** Cache — критический компонент масштабируемых систем. Интервьюер проверяет понимание роли cache, когда его использовать и какие проблемы он решает.

**Короткий ответ.** Distributed cache хранит горячие данные в памяти (RAM) ближе к приложению. Это снижает latency с 10-100ms (DB) до <1ms, уменьшает нагрузку на базу данных и повышает throughput. Без cache БД сразу падает под высокой нагрузкой.

**Детальный разбор.**

**Почему cache необходим:**

Современное приложение требует обработки 100K+ одновременных пользователей. База данных может выполнить 10K-50K запросов/сек при latency 5-100ms. Cache в памяти выполняет 1M+ операций/сек с latency <1ms.

```
Архитектура без cache:
┌─────────────┐
│     App     │
└──────┬──────┘
       │ 100K QPS
       │ (100ms latency)
       ▼
   ┌──────┐
   │  DB  │  ← перегружена
   └──────┘
   Вывод: падение, timeout, потеря данных
```

Архитектура с cache:

```
┌──────────────────┐
│      App         │
└────────┬─────────┘
         │ 100K QPS
    ┌────▼────┐
    │  Cache  │  ← 95% хитов
    │ (RAM)   │  ← <1ms latency
    └────┬────┘
         │ 5K QPS к DB
         │ (только промахи)
         ▼
    ┌────────┐
    │   DB   │  ← легко справляется
    └────────┘
```

**Основные метрики:**
- Cache hit rate: 90-95% (процент успешных обращений)
- Latency: <1ms vs 50-100ms от DB
- Throughput: 1M+ ops/sec vs 10-50K от DB
- Cost: В памяти дешевле, чем масштабировать DB

**Примеры использования:**
1. **Session store** — профиль пользователя, JWT токены
2. **Hot data** — топ товаров, популярные посты, рейтинги
3. **Counters** — views, likes, followers
4. **Temporal data** — последние комментарии, feed
5. **Computed results** — результаты сложных запросов
6. **Rate limit counters** — для защиты от DDoS
7. **Leaderboards** — для конкурсов, игр

**Когда НЕ нужен cache:**
- Данные изменяются часто (>10% writes), cache непрерывно инвалидируется
- Очень большие данные (>100GB), не влезает в память
- Требуется строгая консистентность (транзакции, финансы)
- Low-frequency access, hit rate будет низким

**Пример.**

```python
# БЕЗ cache
@app.route("/api/user/<user_id>")
def get_user(user_id: str):
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    # Каждый запрос идёт в БД (~50ms)
    # 100K QPS × 50ms = 5000 одновременных запросов к БД
    return user

# С cache
from redis import Redis

cache = Redis()

@app.route("/api/user/<user_id>")
def get_user(user_id: str):
    # 1. Попытка получить из cache (~0.5ms)
    cached = cache.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # Hit! Return immediately

    # 2. Cache miss - загрузить из БД
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)

    # 3. Сохранить в cache на 1 час
    cache.setex(f"user:{user_id}", 3600, json.dumps(user))

    return user

# Результат:
# - 95% запросов обслужены из cache за 0.5ms
# - Только 5% идут в БД за 50ms
# - Average latency: 95% × 0.5ms + 5% × 50ms ≈ 3ms
# - QPS к БД: 100K × 0.05 = 5K (вместо 100K)
```

**Типичные ошибки.**
- Использовать cache для часто изменяемых данных — ложные попадания, потеря актуальности
- Не установить TTL (Time To Live) — старые данные останутся в cache
- Cache на горячее хранилище без репликации — потеря данных при сбое
- Не монитории hit rate — cache может быть неэффективным

**На интервью.**
- Объясни, почему cache улучшает latency и throughput, с цифрами
- Назови 3-5 типичных использования cache в социальной сети
- Follow-up: «Какие проблемы cache создаёт?» — inconsistency, invalidation complexity, memory cost

---

### 2. Как работает consistent hashing и почему он необходим?

**Зачем спрашивают.** Consistent hashing — ключевой алгоритм распределения данных. Интервьюер проверяет понимание масштабирования cache кластера без потери большей части данных при добавлении нового узла.

**Короткий ответ.** Consistent hashing распределяет ключи по узлам так, чтобы при добавлении/удалении узла перераспределялась только часть ключей (~1/n). Стандартный модульный hash (key % nodes) при добавлении узла переходит на новый узел 100% ключей.

**Детальный разбор.**

**Проблема стандартного hash:**

```
3 узла: node0, node1, node2

Hash распределение:
key1 → hash(key1) % 3 = 0 → Node 0
key2 → hash(key2) % 3 = 1 → Node 1
key3 → hash(key3) % 3 = 2 → Node 2

Добавили Node 3:
key1 → hash(key1) % 4 = 0 → Node 0 (OK)
key2 → hash(key2) % 4 = 2 → Node 2 (было Node 1, CACHE MISS!)
key3 → hash(key3) % 4 = 1 → Node 1 (было Node 2, CACHE MISS!)

Результат: 66% ключей перехешировалось, потеря cache
```

**Решение: Consistent hashing**

```
Кольцо хешей 0 - 2^32:

                    Node A (hash=45°)
                           ◄──────┐
                       ┌───────────┐
                      /             \
                   ●      ●  Key1   │ Данные назначены узлу
               Key3       │          │ ближайшему по часовой
              /           │           \ стрелке на кольце
        Node C          ●  │             Node B
    (hash=200°)      Key2  └──(hash=120°)
         ▲                   │
         │                   │
         └───────────────────┘

Маршрутизация ключей по часовой стрелке:
Key1 (hash=50°)  → ближайший узел по CW = Node B (120°)
Key2 (hash=100°) → ближайший узел по CW = Node B (120°)
Key3 (hash=170°) → ближайший узел по CW = Node C (200°)
```

**Добавление нового узла:**

```
Было 3 узла, добавили Node D (hash=160°)

                    Node A
                  ┌──────────┐
                 /            \
              ●              ● ●  Key1
           Key3           Key2   │
          /                │    Node D (NEW)
       Node C          ┌───┘      │ ← только эти ключи
   (hash=200°)         │          │    перехешировались
       ▲               │          │
       │          Node B          │
       │         (120°)           │
       └─────────────────────────┘

Перехешировалось:
Key2 (100°) с Node B (120°) → Node D (160°)

Остаток остался на месте:
Key1 (50°) → Node B (120°)
Key3 (170°) → Node C (200°)

Cache loss: только ~33% (1/3 узла вместо 100%)
```

**Virtual nodes (виртуальные узлы):**

Если узлов мало, кольцо неравномерно. Virtual nodes решают это:

```
3 реальных узла, каждый имеет 150 виртуальных:

┌─────────────────────────────────────────┐
│           Hash Ring                     │
│  A0, A1, ..., A149                      │
│  B0, B1, ..., B149  (чередуются)        │
│  C0, C1, ..., C149                      │
│                                         │
│ Результат: равномерное распределение    │
└─────────────────────────────────────────┘

Без virtual nodes (3 точки):
┌─────────────────────────────────────────┐
│      A        B        C                │
│      │        │        │                │
│ Большие горячие зоны, неравномерно      │
└─────────────────────────────────────────┘
```

**Пример.**

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, nodes: list, virtual_nodes: int = 150):
        """
        nodes: список имён узлов ["node0", "node1", "node2"]
        virtual_nodes: количество виртуальных узлов на реальный
        """
        self.virtual_nodes = virtual_nodes
        self.ring = {}  # hash_value -> node_name
        self.sorted_hashes = []

        for node in nodes:
            self.add_node(node)

    def _hash(self, key: str) -> int:
        """Хешируем ключ в число 0 - 2^32"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Добавить узел на кольцо"""
        for i in range(self.virtual_nodes):
            # Виртуальный ключ: "node1:0", "node1:1", ..., "node1:149"
            virtual_key = f"{node}:{i}"
            hash_val = self._hash(virtual_key)
            self.ring[hash_val] = node
            self.sorted_hashes.append(hash_val)

        self.sorted_hashes.sort()
        print(f"Added node {node}, ring size now {len(self.ring)}")

    def remove_node(self, node: str):
        """Удалить узел с кольца"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_val = self._hash(virtual_key)
            del self.ring[hash_val]
            self.sorted_hashes.remove(hash_val)

        print(f"Removed node {node}, ring size now {len(self.ring)}")

    def get_node(self, key: str) -> str:
        """Получить узел для ключа"""
        if not self.ring:
            return None

        hash_val = self._hash(key)

        # Найти первый узел >= hash_val (по часовой стрелке)
        idx = bisect_right(self.sorted_hashes, hash_val)

        # Если переполнили, вернуться в начало
        if idx == len(self.sorted_hashes):
            idx = 0

        node = self.ring[self.sorted_hashes[idx]]
        return node

    def get_nodes(self, key: str, replicas: int = 3) -> list:
        """Получить несколько узлов для репликации"""
        if not self.ring:
            return []

        nodes = []
        seen = set()
        hash_val = self._hash(key)
        idx = bisect_right(self.sorted_hashes, hash_val)

        while len(nodes) < replicas and len(seen) < len(set(self.ring.values())):
            if idx >= len(self.sorted_hashes):
                idx = 0

            node = self.ring[self.sorted_hashes[idx]]
            if node not in seen:
                nodes.append(node)
                seen.add(node)

            idx += 1

        return nodes


# Тест
ch = ConsistentHash(["node0", "node1", "node2"])

# Распределение 10 ключей
for i in range(10):
    key = f"user:{i}"
    node = ch.get_node(key)
    print(f"{key:10} → {node}")

print("\n--- Добавили node3 ---\n")
ch.add_node("node3")

# Проверим, какие ключи переместились
print("\nПосле добавления node3:")
for i in range(10):
    key = f"user:{i}"
    node = ch.get_node(key)
    print(f"{key:10} → {node}")
```

**Типичные ошибки.**
- Забыть virtual nodes — кольцо будет сильно неравномерным, узлы станут bottleneck
- Неправильно считать ближайший узел — использовать < вместо >= по часовой стрелке
- Не обрабатывать удаление узла — старые данные останутся "зависшими"
- Использовать слабый хеш (простой CRC) — коллизии разрушат распределение

**На интервью.**
- Нарисуй кольцо с точками узлов и ключей
- Объясни, что при добавлении узла перехешировать только ~1/n данных
- Упомяни virtual nodes как решение неравномерности
- Follow-up: «Как обрабатывать failed узел?» — перехеширование ключей на replicas

---

### 3. Какие cache eviction policies существуют и когда их использовать?

**Зачем спрашивают.** Cache имеет ограниченную память. Нужно уметь выбирать политику вытеснения (какие ключи удалять) в зависимости от workload.

**Короткий ответ.** LRU (Least Recently Used) — самая популярная. LFU (Least Frequently Used) — для skewed доступа. TTL-based — для временных данных. FIFO — самая быстрая. Выбор зависит от паттерна доступа.

**Детальный разбор.**

**LRU (Least Recently Used):**

```
Идея: удалить самый давно не обращённый ключ

Временная шкала (время возрастает вправо):
┌───────────────────────────────────┐
│ Обращения к ключам               │
├───────────────────────────────────┤
│ key1 key3 key2 key1 key4 key3     │
│  ▲    ▲    ▲    ▲    ▲    ▲       │
│  1    2    3    4    5    6       │
└───────────────────────────────────┘

Last access time:
key1: время 4
key2: время 3 ← самый давно (будет вытеснен)
key3: время 6
key4: время 5
```

Хороший для: стандартная workload, web-приложения

```
Плюсы: простая логика, O(1) при правильной реализации
Минусы: неправильно для нечастого доступа (может быть важный, но старый)
```

**LFU (Least Frequently Used):**

```
Идея: удалить самый редко обращённый ключ

Счётчик обращений:
key1: 5 раз
key2: 1 раз ← самый редкий (будет вытеснен)
key3: 10 раз
key4: 3 раза
```

Хороший для: skewed доступ (few keys горячие, много холодных)

```
Плюсы: лучше для hot/cold данных, важные сохраняются
Минусы: O(log n) вставка/удаление, сложнее реализовать
```

**TTL-based (Time To Live):**

```
Идея: удалить самый скоро истекающий ключ

Время жизни:
key1: 10 минут
key2: 1 минуту ← скоро истечёт (удалим первым)
key3: 100 минут
key4: 5 минут
```

Хороший для: сессии, временные данные, оттоки (refresh tokens)

```
Плюсы: данные автоматически обновляются
Минусы: может вытеснить важный ключ с коротким TTL
```

**Random (случайное):**

```
Идея: выбрать случайный ключ и удалить

Плюсы: O(1), очень быстро, просто реализовать
Минусы: непредсказуемо, может удалить важный ключ
```

**Сравнение на примере:**

```
Cache capacity: 3 ключа
Workload: 10 обращений

Access pattern: A A A B C C D A B B

┌─────────────────────────────────────────┐
│ LRU                                     │
├─────────────────────────────────────────┤
│ A | A | A | B | A,B | A,B,C | C,D | A,D,B │ B |
│ ├─ Hit ─┤ Miss  Evict A  Evict B  Hit Hit │
│ Cache misses: 2                        │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ LFU                                     │
├─────────────────────────────────────────┤
│ Freq: A:3 B:2 C:1 D:1                   │
│ Evict least frequent: C, D              │
│ Cache misses: 3                         │
└─────────────────────────────────────────┘
```

**Реализация LRU:**

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: str):
        """Get value and mark as recently used"""
        if key not in self.cache:
            return None

        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def set(self, key: str, value):
        """Set value, evict if necessary"""
        if key in self.cache:
            # Already exists, just update and mark recent
            self.cache.move_to_end(key)
            self.cache[key] = value
        else:
            # New key
            if len(self.cache) >= self.capacity:
                # Evict least recently used (first item in OrderedDict)
                evicted_key, _ = self.cache.popitem(last=False)
                print(f"Evicted {evicted_key}")

            self.cache[key] = value

    def __repr__(self):
        return f"LRU({list(self.cache.items())})"


# Тест
lru = LRUCache(3)
lru.set("a", 1)
lru.set("b", 2)
lru.set("c", 3)
print(lru)  # LRU([('a', 1), ('b', 2), ('c', 3)])

lru.get("a")  # Mark 'a' as recently used
print(lru)  # LRU([('b', 2), ('c', 3), ('a', 1)])

lru.set("d", 4)  # Need space, evict 'b' (least recently used)
print(lru)  # LRU([('c', 3), ('a', 1), ('d', 4)])
```

**Redis eviction policies:**

```
maxmemory-policy:
- noeviction         ← Don't evict, return error (dangerous!)
- allkeys-lru        ← LRU, all keys
- volatile-lru       ← LRU, only keys with TTL
- allkeys-lfu        ← LFU, all keys
- volatile-lfu       ← LFU, only keys with TTL
- allkeys-random     ← Random, all keys
- volatile-random    ← Random, only keys with TTL
- volatile-ttl       ← Delete by TTL

Рекомендация для большинства: volatile-lru или allkeys-lru
```

**Пример.**

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Установить максимальную память и политику
# r.config_set('maxmemory', '100mb')
# r.config_set('maxmemory-policy', 'allkeys-lru')

# Заполнить cache
for i in range(1000):
    r.set(f"key:{i}", f"value{i}", ex=3600)  # ex=3600: TTL 1 час

# Получить информацию о cache
info = r.info('memory')
print(f"Used memory: {info['used_memory_human']}")
print(f"Peak memory: {info['used_memory_peak_human']}")

# При переполнении Redis автоматически удалит ключи
# согласно maxmemory-policy
```

**Типичные ошибки.**
- Использовать noeviction на production — cache переполнится и упадёт
- Выбрать LRU для热门-холодного распределения — LFU работает лучше
- Не установить TTL — старые данные останутся вечно
- Предположить, что eviction будет мягкой — может быть внезапным spike задержки

**На интервью.**
- Объясни, как LRU выбирает ключ для вытеснения
- Назови случаи, когда LFU лучше LRU
- Follow-up: «Как реализовать LFU эффективно?» — heap + frequency tracking
- Follow-up: «Что если нужен гибридный подход?» — комбинировать TTL + LRU

---

### 4. Какие write policies существуют (write-through, write-back, write-around)?

**Зачем спрашивают.** Cache не только читается, но и пишется. Политика записи определяет консистентность, durability и performance. Интервьюер проверяет понимание trade-offs.

**Короткий ответ.** Write-through (одновременно cache и DB) — консистентный, медленный. Write-back (сначала cache, потом async DB) — быстрый, рискованный. Write-around (bypass cache, писать сразу в DB) — для больших объёмов. Выбор зависит от требований.

**Детальный разбор.**

**Write-Through:**

```
Операция WRITE:
┌──────┐
│ App  │
└──┬───┘
   │
   ├──────────► Cache ─────┐
   │            (write)    │
   │                       ▼
   └──────────────────────► DB
                          (write)

Возврат: только после успеха обоих
```

Гарантии: Cache и DB всегда синхронны

```
Плюсы:
- Полная консистентность
- Сбой cache не теряет данные
- Читаем из cache всегда актуально

Минусы:
- Медленно (ждём DB latency)
- Два RTT: cache + DB
- Высокая нагрузка на DB при пиках

Latency: ~50-100ms (DB latency)
```

**Write-Back (Write-Behind):**

```
Операция WRITE:
┌──────┐
│ App  │
└──┬───┘
   │
   ├──────────► Cache ─────┐
   │            (write)    │ → Немедленно возвращаем
   │                       │
   └──────────────────┐    │
                      ▼    ▼
                   Queue   ┌──────────────┐
                           │ Background   │
                           │ Worker (async)
                           └──────┬───────┘
                                  ▼
                                Cache → DB (batch write)
```

Гарантии: Cache персистируется асинхронно

```
Плюсы:
- Быстро (return immediately)
- Batch write к DB (экономит network)
- Cache absorbs spike нагрузки

Минусы:
- Eventual consistency
- Риск потери при сбое cache (можно добавить WAL)
- Сложнее отладка (async логика)

Latency: ~1-10ms (только cache)
Throughput: очень высокий
```

**Write-Around:**

```
Операция WRITE:
┌──────┐
│ App  │
└──┬───┘
   │
   │ (bypass cache, write directly)
   │
   └──────────────────────────────► DB
                                  (write)

Cache остаётся как читаемая copy, не обновляется на write
```

Гарантии: DB истинный источник, cache optional

```
Плюсы:
- Простая консистентность (write идёт в DB)
- Не заполняет cache мусором
- Хорошо для write-heavy workloads

Минусы:
- Write не получает benefits cache
- Возможен cache stale data
- Нужна инвалидация при write

Latency: ~50-100ms (DB latency)
Throughput: ограничен DB
```

**Сравнение:**

```
┌────────────┬──────────┬────────────┬────────────┐
│ Policy     │ Speed    │ Consistency│ Best For   │
├────────────┼──────────┼────────────┼────────────┤
│ Through    │ Slow     │ Strong     │ Financial  │
│            │ ~100ms   │            │ Inventory  │
├────────────┼──────────┼────────────┼────────────┤
│ Back       │ Fast     │ Eventual   │ Feeds      │
│            │ ~5ms     │ ~100ms lag │ Analytics  │
├────────────┼──────────┼────────────┼────────────┤
│ Around     │ Medium   │ Weak       │ Large      │
│            │ ~50ms    │            │ Bulk write │
└────────────┴──────────┴────────────┴────────────┘
```

**Пример.**

```python
import redis
import json
from datetime import datetime
from threading import Thread
from queue import Queue
import time

r = redis.Redis()

# ===== WRITE-THROUGH =====
def write_through(user_id: str, data: dict):
    """Write to cache and DB synchronously"""
    # 1. Write to cache
    cache_key = f"user:{user_id}"
    r.set(cache_key, json.dumps(data), ex=3600)

    # 2. Write to DB (blocking)
    db_write(user_id, data)  # This is slow!

    # Only return after both succeed
    print(f"[THROUGH] Wrote user {user_id} at {datetime.now()}")


# ===== WRITE-BACK =====
write_queue = Queue()

def write_back(user_id: str, data: dict):
    """Write to cache immediately, queue DB write"""
    # 1. Write to cache immediately
    cache_key = f"user:{user_id}"
    r.set(cache_key, json.dumps(data), ex=3600)

    # 2. Queue for async DB write
    write_queue.put((user_id, data))

    # Return immediately!
    print(f"[BACK] Cached user {user_id} at {datetime.now()}")


def background_writer():
    """Worker that batches writes to DB"""
    batch = []

    while True:
        try:
            item = write_queue.get(timeout=5)
            batch.append(item)

            if len(batch) >= 100 or write_queue.empty():
                # Batch write to DB
                print(f"[BATCH] Writing {len(batch)} items to DB")
                for user_id, data in batch:
                    db_write(user_id, data)
                batch = []
        except:
            if batch:
                db_write_batch(batch)
                batch = []


# Start background writer thread
Thread(target=background_writer, daemon=True).start()


# ===== WRITE-AROUND =====
def write_around(user_id: str, data: dict):
    """Write directly to DB, bypass cache"""
    # 1. Write to DB
    db_write(user_id, data)

    # 2. Invalidate cache (optional)
    cache_key = f"user:{user_id}"
    r.delete(cache_key)

    print(f"[AROUND] Wrote user {user_id} directly to DB")


def db_write(user_id: str, data: dict):
    """Simulate DB write"""
    time.sleep(0.05)  # 50ms latency


# Тест
print("=== WRITE-THROUGH ===")
start = time.time()
write_through("user1", {"name": "Alice"})
print(f"Time: {time.time() - start:.3f}s\n")

print("=== WRITE-BACK ===")
start = time.time()
write_back("user2", {"name": "Bob"})
print(f"Time: {time.time() - start:.3f}s\n")

print("=== WRITE-AROUND ===")
start = time.time()
write_around("user3", {"name": "Charlie"})
print(f"Time: {time.time() - start:.3f}s\n")

# Wait for background writes
time.sleep(6)
```

**Гибридный подход (Write-Back + Durability):**

Для protection от потерь при сбое:

```
write_back_with_durability:
1. Приложение отправляет WRITE
2. Cache получает write
3. WAL (Write-Ahead Log) сохраняет на persistent storage
4. Вернуть OK приложению
5. Async: отправить в DB, затем удалить из WAL

Если cache упадёт до DB write:
- WAL восстанавливает данные
- Retry напишет в DB

Как Redis AOF (Append-Only File)
```

**Типичные ошибки.**
- Использовать write-through везде — очень медленно для high-traffic
- Write-back без WAL — потеря данных при сбое
- Забыть инвалидировать cache при write-around — stale reads
- Не батчировать writes в write-back — слишком много DB операций

**На интервью.**
- Объясни разницу write-through vs write-back с latency цифрами
- Когда use write-around? — для bulk writes, когда cache не помогает
- Follow-up: «Как добавить durability к write-back?» — WAL + batch commits
- Follow-up: «Как обрабатывать spikey traffic?» — write-back + queue limits

---

### 5. Как решить проблему cache invalidation?

**Зачем спрашивают.** Cache invalidation — одна из двух сложных задач в CS (другая — naming). Неправильная инвалидация → stale data и bugs. Интервьюер проверяет глубину понимания.

**Короткий ответ.** Основные подходы: 1) Явная инвалидация при update (delete ключ из cache), 2) TTL (автоматический expire), 3) Event-based (инвалидация по событиям), 4) Smart refresh (probabilistic или conditional). Идеально комбинировать несколько.

**Детальный разбор.**

**Проблема:**

```
Сценарий: пользователь обновляет профиль

1. Пользователь редактирует профиль
   App обновляет БД: UPDATE users SET name='Bob' WHERE id=1

2. Cache всё ещё вернёт старое имя: "Alice"

3. Читающий пользователь видит устаревшие данные

4. Когда обновится? Никогда, пока не истечёт TTL или cache не перезапустится
```

**Подход 1: Явная инвалидация**

```
┌─────────────┐
│ App Request │
└──────┬──────┘
       │
       ├─ Update in DB ─────────────────────────► DB
       │                                         (success)
       │
       └─ Delete from cache ────────────────────► Cache
                                                (invalidate)

Следующий read → cache miss → load fresh data

Плюсы: гарантированно свежие данные
Минусы: нужно помнить инвалидировать каждый update
```

**Подход 2: TTL (Time To Live)**

```
set("user:1", data, ttl=3600)  # Истечёт через час

Минусы:
- Есть окно неконсистентности (до 1 часа)
- Может быть слишком частое истечение или слишком редкое

Плюсы:
- Простая реализация
- Не нужно помнить инвалидировать
```

**Подход 3: Event-Based Invalidation**

```
Событие: пользователь обновил профиль

System:
┌─────────────────────────────────┐
│ Database Update                 │
└────────────┬────────────────────┘
             │
             ├─► Event Queue (publish)
             │
             ├─► Event Processor (subscribe)
             │   ├─► Invalidate cache
             │   ├─► Update search index
             │   ├─► Send notifications
             │   └─► Log audit

Плюсы: реактивная, масштабируемая
Минусы: сложнее реализовать, eventual consistency
```

**Подход 4: Probabilistic Early Refresh**

```
Идея: заменить ключ до истечения TTL

remaining_ttl = expiry - current_time
refresh_prob = 1 - (remaining_ttl / original_ttl)

Если вероятность < случайное число:
    → запустить background refresh асинхронно

┌──────────────────────────────────────────────┐
│ TTL = 3600s, current_time = 3550s             │
│ remaining = 50s                               │
│ refresh_prob = 1 - (50/3600) = 0.986          │
│ random.random() < 0.986 → ДА, refresh!       │
│                                              │
│ Но не заблокируемся, вернём старые данные    │
│ Refresh происходит в фоне                    │
└──────────────────────────────────────────────┘

Плюсы: улучшает cache hit rate, нет cache stampede
Минусы: data может быть старой до refresh
```

**Подход 5: Cache Versioning**

```
Вместо:
set("user:1", data)

Используем:
set("user:1:v2", data)  // version in key

При инвалидации:
delete("user:1:v1")  // старая версия исчезнет

Плюсы:
- Может одновременно существовать несколько версий
- Graceful degradation

Минусы:
- Увеличивает memory usage
- Нужна логика выбора версии
```

**Пример.**

```python
import redis
import json
import time
import random
from threading import Thread

r = redis.Redis()

# ===== EXPLICIT INVALIDATION =====
def update_user_explicit(user_id: str, new_data: dict):
    """Update DB and explicitly invalidate cache"""
    # 1. Update database
    db.update_user(user_id, new_data)

    # 2. Delete from cache (explicit invalidation)
    cache_key = f"user:{user_id}"
    r.delete(cache_key)

    # 3. Invalidate dependent caches (cascade!)
    r.delete(f"user_friends:{user_id}")
    r.delete(f"user_profile_html:{user_id}")

    print(f"[EXPLICIT] Invalidated user {user_id}")


def get_user_explicit(user_id: str):
    """Get user with cache"""
    cache_key = f"user:{user_id}"

    # Check cache
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    # Cache miss, load from DB
    user = db.get_user(user_id)
    r.set(cache_key, json.dumps(user), ex=3600)
    return user


# ===== TTL-BASED =====
def update_user_ttl(user_id: str, new_data: dict):
    """Update DB, cache will auto-expire via TTL"""
    db.update_user(user_id, new_data)
    # No explicit delete, TTL handles it


def get_user_ttl(user_id: str):
    """Get with TTL-based expiration"""
    cache_key = f"user:{user_id}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    user = db.get_user(user_id)
    r.set(cache_key, json.dumps(user), ex=3600)  # 1 hour TTL
    return user


# ===== PROBABILISTIC EARLY REFRESH =====
def get_user_probabilistic(user_id: str):
    """Get with probabilistic early refresh"""
    cache_key = f"user:{user_id}"

    # Try to get with TTL info
    cached = r.get(cache_key)
    if cached:
        ttl = r.ttl(cache_key)  # remaining TTL in seconds

        if ttl > 0:
            original_ttl = 3600
            # Probability increases as TTL decreases
            refresh_prob = 1.0 - (ttl / original_ttl)

            # Early refresh with low probability
            if random.random() < refresh_prob * 0.1:
                # Refresh in background (non-blocking)
                Thread(
                    target=refresh_cache_background,
                    args=(user_id,),
                    daemon=True
                ).start()

        return json.loads(cached)

    # Cache miss
    user = db.get_user(user_id)
    r.set(cache_key, json.dumps(user), ex=3600)
    return user


def refresh_cache_background(user_id: str):
    """Background refresh of cache"""
    user = db.get_user(user_id)
    cache_key = f"user:{user_id}"
    r.set(cache_key, json.dumps(user), ex=3600)
    print(f"[REFRESH] Refreshed user {user_id} in background")


# ===== EVENT-BASED INVALIDATION =====
def publish_user_updated(user_id: str):
    """Publish event when user is updated"""
    r.publish("user.updated", json.dumps({"user_id": user_id}))


def subscribe_to_updates():
    """Subscribe and invalidate cache"""
    pubsub = r.pubsub()
    pubsub.subscribe("user.updated")

    for message in pubsub.listen():
        if message['type'] == 'message':
            data = json.loads(message['data'])
            user_id = data['user_id']

            # Invalidate
            r.delete(f"user:{user_id}")
            print(f"[EVENT] Invalidated user {user_id}")


# Start event listener
Thread(target=subscribe_to_updates, daemon=True).start()


def update_user_event(user_id: str, new_data: dict):
    """Update and publish event"""
    db.update_user(user_id, new_data)
    publish_user_updated(user_id)


# Тест
print("=== EXPLICIT INVALIDATION ===")
update_user_explicit("user1", {"name": "Bob"})
print(get_user_explicit("user1"))

print("\n=== TTL-BASED ===")
get_user_ttl("user2")  # Cache for 1 hour
# After 1 hour, automatically expires

print("\n=== PROBABILISTIC EARLY REFRESH ===")
for i in range(5):
    print(get_user_probabilistic("user3"))
    time.sleep(0.5)
```

**Типичные ошибки.**
- Забыть инвалидировать dependent caches (cascade invalidation)
- TTL слишком короткий — часто промахи, слишком длинный — старые данные
- Race condition: DB update и cache invalidation не atomic
- Event-based без fallback — если событие потеряется, стать stale навсегда

**На интервью.**
- Объясни, почему cache invalidation сложная
- Назови 3 подхода и trade-offs
- Follow-up: «Как обрабатывать cascade invalidation?» — помечать dependencies
- Follow-up: «Что если DB update и cache invalidation не atomic?» — используй distributed lock или message queue

---

### 6. Как работает Redis Cluster и как он масштабируется?

**Зачем спрашивают.** Redis Cluster — стандартный способ масштабирования Redis. Интервьюер проверяет понимание distributed systems, sharding, replication.

**Короткий ответ.** Redis Cluster делит данные на 16384 hash slots, распределяя их по узлам. Ключ → CRC16(key) % 16384 → slot → узел. Каждый узел отвечает за диапазон slots. Есть master и replica для высокой доступности.

**Детальный разбор.**

**Архитектура:**

```
┌────────────────────────────────────────────────────────┐
│               Redis Cluster (6+ узлов)                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Hash Slot Range 0-16383 разделены между узлами      │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │ Master Node 1 (slots 0-5460)                   │   │
│  │ ├─ Replica 1                                   │   │
│  │ └─ Key distribution: CRC16(key) % 16384        │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │ Master Node 2 (slots 5461-10922)               │   │
│  │ ├─ Replica 2                                   │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │ Master Node 3 (slots 10923-16383)              │   │
│  │ ├─ Replica 3                                   │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
└────────────────────────────────────────────────────────┘

Ключ распределение:
"user:123" → CRC16("user:123") = 12345 → slot 12345 → Node 3
```

**Hash Tags (для группировки ключей):**

```
Проблема:
"user:123:name"  → slot X
"user:123:age"   → slot Y
Разные слоты → разные узлы → требует distributed transaction

Решение: hash tags
"{user:123}:name" → слот определяется только "user:123"
"{user:123}:age"  → тот же слот → один узел → atomic

CRC16 вычисляется для содержимого между {}
```

**Отказоустойчивость:**

```
Normal:
┌──────────┐        ┌──────────┐        ┌──────────┐
│ Node 1   │        │ Node 2   │        │ Node 3   │
│ (Master) │        │ (Master) │        │ (Master) │
└────┬─────┘        └────┬─────┘        └────┬─────┘
     │                   │                   │
┌────▼─────┐        ┌────▼─────┐        ┌────▼─────┐
│ Replica 1│        │ Replica 2│        │ Replica 3│
└──────────┘        └──────────┘        └──────────┘

Node 2 падает:
┌──────────┐        ┌─────X────┐        ┌──────────┐
│ Node 1   │        │ Node 2 X  │        │ Node 3   │
│ (Master) │        │ (down!)   │        │ (Master) │
└────┬─────┘        └────┬─────┘        └────┬─────┘
     │                   │                   │
┌────▼─────┐        ┌────▼─────┐        ┌────▼─────┐
│ Replica 1│        │ Replica 2 │◄───promoted to Master
└──────────┘        └──────────┘        └──────────┘
                    Automatic failover!
```

**Масштабирование (добавление узла):**

```
Было 3 узла:
Slots 0-5460   → Node A
Slots 5461-10922 → Node B
Slots 10923-16383 → Node C

Добавляем Node D:
1. Новый узел D присоединяется (empty)
2. Reshard: переместить часть slots с других узлов

Новая конфигурация:
Slots 0-4095   → Node A
Slots 4096-8191  → Node B
Slots 8192-12287 → Node C
Slots 12288-16383 → Node D

Процесс миграции: постепенно, без downtime
```

**Пример.**

```python
import redis
from redis.cluster import RedisCluster

# Создать кластер (обычно в prod)
# redis-cli --cluster create node1:6379 node2:6379 ... --cluster-replicas 1

# Подключиться к кластеру
rc = RedisCluster(
    startup_nodes=[
        {"host": "node1", "port": 6379},
        {"host": "node2", "port": 6379},
        {"host": "node3", "port": 6379},
    ],
    decode_responses=True
)

# Базовые операции (автоматически маршрутизируются)
rc.set("user:1:name", "Alice")
rc.set("user:2:name", "Bob")

# Получить с конкретного узла
node = rc.get_node("user:1:name")
print(f"Key 'user:1:name' on node {node}")

# Hash tags для группировки
rc.set("{user:1}:name", "Alice")
rc.set("{user:1}:age", 30)

# Batch операции требуют одного узла (разные keys)
# Для distributed batch нужна app-level логика
def batch_get_users(user_ids: list):
    """Получить несколько пользователей"""
    results = {}
    for user_id in user_ids:
        results[user_id] = rc.get(f"user:{user_id}:name")
    return results

# Мониторинг кластера
def cluster_info():
    """Получить информацию о кластере"""
    info = rc.execute_command("CLUSTER", "INFO")
    nodes = rc.execute_command("CLUSTER", "NODES")

    print("=== Cluster Info ===")
    print(info)
    print("\n=== Cluster Nodes ===")
    print(nodes)

# Обработка миграции узла
def migrate_key(key: str, from_node, to_node):
    """Мигрировать ключ с одного узла на другой"""
    try:
        value = rc.get(key)
        if value:
            # Redis cluster автоматически обрабатывает миграцию
            # Можно использовать MIGRATE команду вручную
            rc.execute_command(
                "MIGRATE",
                to_node.host, to_node.port,
                key, 0, 5000
            )
    except Exception as e:
        print(f"Migration failed: {e}")

# Обработка MOVED ошибок
def resilient_get(key: str, retries: int = 3):
    """Get с retry при MOVED errors"""
    for attempt in range(retries):
        try:
            return rc.get(key)
        except redis.ResponseError as e:
            if "MOVED" in str(e):
                # Cluster перестраивается, обновить topology
                rc.cluster_slots()
                if attempt < retries - 1:
                    continue
            raise
    raise Exception(f"Failed to get {key} after {retries} attempts")
```

**Операции кластера:**

```python
# Информация о кластере
CLUSTER INFO       # Состояние кластера
CLUSTER NODES      # Информация об узлах
CLUSTER SLOTS      # Назначение слотов узлам

# Управление
CLUSTER ADDSLOTS slot [slot ...]     # Добавить slots узлу
CLUSTER DELSLOTS slot [slot ...]     # Удалить slots
CLUSTER SETSLOT slot NODE node-id    # Перенести slot
CLUSTER MEET ip port                 # Добавить узел в кластер

# Миграция
CLUSTER SAVECONFIG               # Сохранить конфиг
MIGRATE host port key db timeout # Мигрировать ключ
```

**Типичные ошибки.**
- Использовать Redis Cluster для single commands, требующих consistency
- Забыть про hash tags при grouping related keys
- Не настроить min-replicas-to-write — потеря данных при partition
- Недостаточно nodes (< 6) — сложнее mastered failover

**На интервью.**
- Объясни, как ключ маршрутизируется на узел (CRC16 % 16384)
- Как обрабатывается failover при сбое узла?
- Follow-up: «Как добавить новый узел без downtime?» — reshard slots постепенно
- Follow-up: «Когда использовать hash tags?» — когда нужна группировка ключей на один узел

---

### 7. Как обработать cache stampede и thundering herd?

**Зачем спрашивают.** Cache stampede — критическая production проблема. Когда популярный ключ истекает, тысячи запросов штурмуют базу данных. Интервьюер проверяет знание защиты.

**Короткий ответ.** Cache stampede: много одновременных запросов при cache miss → перегруз DB. Решения: 1) Locking (первый запрос refreshes, остальные ждят), 2) Probabilistic early expiration (refresh до истечения), 3) Singleflight (дедупликация запросов), 4) Large TTL + versioning.

**Детальный разбор.**

**Проблема:**

```
t=0: Ключ "user:1:profile" в cache, popular, 1000 requests/sec

t=3600: TTL истекает (установлен 1 hour ago)

        Одновременно 1000+ запросов от разных clients:
        ┌──────────────────────┐
        │ GET user:1:profile   │ ← cache miss!
        │ GET user:1:profile   │ ← cache miss!
        │ GET user:1:profile   │ ← cache miss!
        │ ... 997 more         │
        └──────────────────────┘

        Все идут в БД:
        ┌────────────────────────────┐
        │ SELECT * FROM users WHERE  │ ← 1000 одновременных
        │ ... (database overload)    │
        └────────────────────────────┘

Результат: DB падает, timeout, cascade failure
```

**Решение 1: Locking (Lock-based refresh)**

```
Сценарий: 1000 requests на expired ключ

Request 1: Cache miss, acquire lock, load from DB, update cache
Request 2-1000: Cache miss, try to acquire lock → failed → wait → retry

┌──────────────────────────────────┐
│ Request 1                        │
├──────────────────────────────────┤
│ GET cache → miss                 │
│ SET lock:"user:1" (setnx)  ✓     │
│ DB query (all 1000 wait)         │
│ SET cache value, DEL lock  ✓     │
└──────────────────────────────────┘

┌──────────────────────────────────┐
│ Requests 2-1000                  │
├──────────────────────────────────┤
│ GET cache → miss                 │
│ SET lock:"user:1" (setnx) ✗      │
│ sleep(10ms)                      │
│ retry GET cache → hit!  ✓        │
└──────────────────────────────────┘

Результат: только 1 DB query вместо 1000
```

**Решение 2: Probabilistic Early Expiration**

```
Идея: refresh ключ ДО истечения TTL, чтобы старые данные
      никогда не истекали

current_time = 3550
TTL = 3600
remaining_ttl = 50
refresh_prob = 1 - (50 / 3600) = 0.986

if random() < 0.986:
    background_refresh(key)
    return old_value

Когда остаётся мало времени, refresh вероятность растёт
Но редко точно при истечении → cache всегда есть value

Плюсы: избегаем stampede, всегда есть value
Минусы: может быть stale до refresh
```

**Решение 3: Singleflight (Дедупликация запросов)**

```
import singleflight (или эквивалент)

1000 requests к "user:1:profile":

┌──────────────────────────────────┐
│ Singleflight.Do("user:1:profile")│
├──────────────────────────────────┤
│ Request 1: Execute fn()          │
│           (load from DB)         │
│ Requests 2-1000: Wait for Request 1
│                                  │
│ All get same result              │
└──────────────────────────────────┘

Результат: 1 DB query, 1000 clients получают результат
```

**Пример.**

```python
import redis
import time
import random
import threading
from concurrent.futures import ThreadPoolExecutor

r = redis.Redis()

# ===== LOCKING APPROACH =====
def get_with_lock(key: str, fetch_fn, lock_timeout: int = 10):
    """Get value with lock to prevent stampede"""
    # 1. Try to get from cache
    cached = r.get(key)
    if cached:
        return cached.decode()

    # 2. Cache miss - try to acquire lock
    lock_key = f"lock:{key}"

    try:
        # setnx: SET if Not eXists (atomic)
        acquired = r.set(lock_key, "1", nx=True, ex=lock_timeout)

        if acquired:
            # Won the lock - fetch data
            print(f"[LOCK] {threading.current_thread().name}: locked, fetching")
            value = fetch_fn()  # Expensive operation
            r.set(key, value, ex=3600)
            return value
        else:
            # Lost lock - wait and retry
            print(f"[LOCK] {threading.current_thread().name}: waiting for lock")
            for _ in range(50):  # Wait up to 5 seconds
                time.sleep(0.1)
                cached = r.get(key)
                if cached:
                    print(f"[LOCK] {threading.current_thread().name}: got value from cache")
                    return cached.decode()

            # Still no value, timeout waiting
            raise TimeoutError(f"Lock timeout for {key}")

    except Exception as e:
        print(f"[LOCK] Error: {e}")
        raise


# ===== PROBABILISTIC EARLY REFRESH =====
def get_with_early_refresh(key: str, fetch_fn, ttl: int = 3600):
    """Get with probabilistic early refresh to prevent stampede"""
    cached = r.get(key)

    if cached:
        # Check TTL
        remaining_ttl = r.ttl(key)

        if remaining_ttl > 0:
            # Probability increases as TTL decreases
            original_ttl = ttl
            refresh_probability = 1.0 - (remaining_ttl / original_ttl)

            # Early refresh with capped probability (max 10%)
            if random.random() < min(refresh_probability * 0.1, 0.1):
                print(f"[EARLY] Refreshing {key} in background")
                # Background refresh (non-blocking)
                threading.Thread(
                    target=lambda: r.set(key, fetch_fn(), ex=ttl),
                    daemon=True
                ).start()

        return cached.decode()

    # Cache miss
    value = fetch_fn()
    r.set(key, value, ex=ttl)
    return value


# ===== SINGLEFLIGHT APPROACH (manual implementation) =====
class SingleFlight:
    """Deduplicate concurrent requests for same key"""
    def __init__(self):
        self.inflight = {}  # key -> (lock, result_event)

    def do(self, key: str, fn):
        """Execute fn once for key, other requests wait"""
        # Check if already inflight
        if key in self.inflight:
            lock, result_event = self.inflight[key]
            print(f"[SF] {threading.current_thread().name}: waiting for inflight {key}")
            result_event.wait()  # Wait for result
            return self.inflight[key][2]  # Return cached result

        # Create new inflight request
        import threading as th
        result_event = th.Event()
        lock = th.Lock()

        self.inflight[key] = (lock, result_event, None)

        try:
            print(f"[SF] {threading.current_thread().name}: executing {key}")
            result = fn()  # Execute once
            self.inflight[key] = (lock, result_event, result)
            result_event.set()  # Signal completion
            return result
        finally:
            del self.inflight[key]


sf = SingleFlight()

def get_with_singleflight(key: str, fetch_fn):
    """Get with singleflight deduplication"""
    cached = r.get(key)
    if cached:
        return cached.decode()

    # Deduplicate concurrent cache misses
    result = sf.do(key, fetch_fn)
    r.set(key, result, ex=3600)
    return result


# ===== SIMULATION =====
def expensive_fetch():
    """Simulate expensive DB query"""
    time.sleep(0.5)  # 500ms latency
    return f"data_{time.time()}"


def test_stampede_without_protection():
    """Test stampede without protection"""
    r.delete("test_key")

    print("\n=== WITHOUT PROTECTION ===")
    start = time.time()

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = []
        for i in range(10):
            futures.append(
                executor.submit(
                    lambda: r.get("test_key") or
                            (time.sleep(0.5), r.set("test_key", expensive_fetch()))[1]
                )
            )
        for f in futures:
            f.result()

    elapsed = time.time() - start
    print(f"Time: {elapsed:.2f}s (serial, no dedup)")


def test_stampede_with_lock():
    """Test stampede with lock protection"""
    r.delete("test_key")
    r.delete("lock:test_key")

    print("\n=== WITH LOCK ===")
    start = time.time()

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = []
        for i in range(10):
            futures.append(
                executor.submit(get_with_lock, "test_key", expensive_fetch)
            )
        for f in futures:
            f.result()

    elapsed = time.time() - start
    print(f"Time: {elapsed:.2f}s (one fetch, others wait)")


def test_stampede_with_singleflight():
    """Test stampede with singleflight"""
    r.delete("test_key")

    print("\n=== WITH SINGLEFLIGHT ===")
    start = time.time()

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = []
        for i in range(10):
            futures.append(
                executor.submit(get_with_singleflight, "test_key", expensive_fetch)
            )
        for f in futures:
            f.result()

    elapsed = time.time() - start
    print(f"Time: {elapsed:.2f}s (one fetch, others share result)")


# Run tests
test_stampede_without_protection()
test_stampede_with_lock()
test_stampede_with_singleflight()
```

**Типичные ошибки.**
- Не установить timeout на lock — forever блокировка
- Probabilistic refresh без fallback — data может быть очень старой
- Singleflight без error handling — ошибка распространяется на всех
- Не мониторить cache hit rate — не заметить проблему

**На интервью.**
- Объясни сценарий cache stampede с цифрами
- Назови 3 решения и их trade-offs
- Follow-up: «Как комбинировать несколько решений?» — lock для безопасности + early refresh для улучшения hit rate
- Follow-up: «Как выбрать TTL чтобы минимизировать stampede?» — balance между freshness и stability

---

### 8. Как реализовать cache-aside pattern правильно?

**Зачем спрашивают.** Cache-aside (lazy loading) — самый популярный pattern. Интервьюер проверяет понимание деталей: consistency, invalidation, error handling.

**Короткий ответ.** Cache-aside: на read проверить cache, miss → load from DB + update cache; на write обновить DB и инвалидировать cache. Просто в теории, но есть race conditions и edge cases.

**Детальный разбор.**

**Базовый паттерн:**

```
READ:
1. Check cache
   ├─ HIT  → return value
   └─ MISS → load from DB, update cache, return

WRITE:
1. Update DB
2. Delete from cache (invalidate)
3. Next read will load fresh value
```

**Код:**

```python
def get(key: str):
    # 1. Check cache
    value = cache.get(key)
    if value is not None:
        return value

    # 2. Cache miss - load from DB
    value = db.get(key)

    # 3. Update cache
    if value is not None:
        cache.set(key, value, ttl=3600)

    return value

def set(key: str, value):
    # 1. Update DB
    db.set(key, value)

    # 2. Invalidate cache
    cache.delete(key)
```

**Проблемы и решения:**

**Проблема 1: Write-Read Race Condition**

```
Timeline:
t1: Client A: set(key, valueA) → DB updated
t2: Client B: get(key) → cache miss
t3: Client A: cache.delete(key) → invalidated
t4: Client B: load from DB → get valueA ✓
    Client B: cache.set(key, valueA) ✓

OK, работает правильно

Но если порядок операций:
t1: Client A: set(key, valueA)
t2: Client A: cache.delete(key)
t3: Client B: get(key) → cache miss
t4: Client B: cache.set(key, valueOLD) ← STALE! DB has valueA, cache has OLD

Решение: invert order
t1: Client A: cache.delete(key) → первым инвалидировать!
t2: Client A: set(key, valueA) → потом обновить DB
```

**Проблема 2: Thundering Herd**

Уже обсудили выше (Section 7). Решение: locking или early refresh.

**Проблема 3: Error Handling**

```
Что если DB down?

get(key):
    value = cache.get(key)  ✓
    if value:
        return value

    try:
        value = db.get(key)  ✗ timeout/error
    except Exception:
        # Option 1: вернуть stale cache value
        # Option 2: вернуть error
        # Option 3: retry с exponential backoff
        pass
```

**Проблема 4: Cascade Invalidation**

```
Обновляем пользователя:
update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")           # ✓
    cache.delete(f"user:{user_id}:friends")   # ✓
    cache.delete(f"user:{user_id}:profile")   # ✓
    cache.delete(f"feed:{user_id}")           # ✓
    # ... может быть 10+ related caches!

Забыли инвалидировать:
cache.delete(f"leaderboard:friends_count")  ← друзья посчитаны

Результат: stale leaderboard, пользователь видит неправильный ранг
```

**Пример правильной реализации:**

```python
from datetime import datetime, timedelta
from enum import Enum
import logging

logger = logging.getLogger(__name__)

class CacheAsideClient:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db

    def get(self, key: str, default=None):
        """Cache-aside READ pattern"""
        try:
            # 1. Try cache first
            cached = self.cache.get(key)
            if cached is not None:
                return cached
        except Exception as e:
            logger.warning(f"Cache error on get {key}: {e}")
            # Continue to DB on cache error

        try:
            # 2. Cache miss or error - load from DB
            value = self.db.get(key)

            # 3. Update cache (async is fine)
            if value is not None:
                try:
                    self.cache.set(key, value, ttl=3600)
                except Exception as e:
                    logger.warning(f"Failed to cache {key}: {e}")
                    # Continue anyway, cache is optional

            return value

        except Exception as e:
            logger.error(f"DB error on get {key}: {e}")
            return default

    def set(self, key: str, value):
        """Cache-aside WRITE pattern"""
        # Option 1: Invalidate first (safer for consistency)
        try:
            self.cache.delete(key)
        except Exception as e:
            logger.warning(f"Failed to invalidate {key}: {e}")

        # Option 2: Update DB
        self.db.set(key, value)

        # Option 3: Update cache (optional, for write-through feel)
        try:
            self.cache.set(key, value, ttl=3600)
        except Exception as e:
            logger.warning(f"Failed to cache after write {key}: {e}")

    def invalidate(self, key: str, related_keys: list = None):
        """Invalidate key and related keys"""
        keys_to_delete = [key]
        if related_keys:
            keys_to_delete.extend(related_keys)

        for k in keys_to_delete:
            try:
                self.cache.delete(k)
            except Exception as e:
                logger.warning(f"Failed to delete {k}: {e}")

    def get_or_refresh(self, key: str, fetch_fn, ttl: int = 3600):
        """
        Get from cache, optionally refresh if old
        Useful for expensive computations
        """
        cached = self.cache.get(key)
        if cached:
            return cached

        # Cache miss - fetch and update
        value = fetch_fn()
        self.cache.set(key, value, ttl=ttl)
        return value

    def batch_get(self, keys: list):
        """Batch get with cache"""
        result = {}
        uncached_keys = []

        # Try cache first
        for key in keys:
            try:
                cached = self.cache.get(key)
                if cached is not None:
                    result[key] = cached
                else:
                    uncached_keys.append(key)
            except Exception as e:
                logger.warning(f"Cache error: {e}")
                uncached_keys.append(key)

        # Load uncached from DB
        if uncached_keys:
            db_values = self.db.mget(uncached_keys)

            # Update cache for loaded values
            for key, value in db_values.items():
                result[key] = value
                try:
                    self.cache.set(key, value, ttl=3600)
                except Exception as e:
                    logger.warning(f"Failed to cache {key}: {e}")

        return result


# Usage example
from redis import Redis

cache = Redis()
db = MockDatabase()
client = CacheAsideClient(cache, db)

# Simple get
user = client.get("user:123")

# Set with invalidation
client.set("user:123", {"name": "Bob"})

# Invalidate related keys
client.invalidate(
    "user:123",
    related_keys=[
        "user:123:friends",
        "user:123:profile",
        f"feed:{user_id}",
    ]
)

# Batch operations
users = client.batch_get(["user:1", "user:2", "user:3"])
```

**Типичные ошибки.**
- Забыть инвалидировать на write → stale data
- Неправильный порядок delete/update → race condition
- Не обрабатывать ошибки cache → приложение падает
- Не мониторить cache hit rate → не замечают проблем

**На интервью.**
- Нарисуй timeline для race condition scenario
- Объясни, почему "invalidate first" безопаснее
- Follow-up: «Как обрабатывать cascade invalidation?» — пометить dependencies или use event-based invalidation
- Follow-up: «Что если DB и cache асинхронны?» — eventual consistency, нужна TTL backup

---

### 9. Как мониторить cache hit rate и другие метрики?

**Зачем спрашивают.** Без мониторинга не знаешь, работает ли cache. Интервьюер проверяет понимание операционных аспектов.

**Короткий ответ.** Основные метрики: hit rate (успешные reads), miss rate, eviction rate, memory usage, latency. Hit rate > 90% хорошо. Мониторим через stats (Redis INFO) и application-level metrics.

**Детальный разбор.**

**Ключевые метрики:**

```
1. Cache Hit Rate
   ├─ Definition: (hits) / (hits + misses) × 100%
   ├─ Good: > 90%
   ├─ Warning: < 80%
   └─ Why: если <80%, cache не помогает, деньги потрачены впустую

2. Cache Miss Rate
   ├─ Definition: (misses) / (hits + misses) × 100%
   ├─ Related to hit rate
   └─ Should be < 10% for good cache

3. Eviction Rate
   ├─ Definition: keys evicted per second
   ├─ High eviction → memory pressure
   ├─ Action: increase cache size or reduce TTL
   └─ Redis: evicted_keys

4. Memory Usage
   ├─ Used memory
   ├─ Peak memory
   ├─ Fragmentation ratio
   └─ Action: if >80%, increase capacity or reduce TTL

5. Latency
   ├─ Read latency (cache hit)
   ├─ Read latency (cache miss)
   ├─ Write latency
   └─ Threshold: hit < 1ms, miss < 50ms

6. Throughput
   ├─ Ops per second
   ├─ Should be >> DB throughput
   └─ If limited, check CPU/network
```

**Redis INFO для мониторинга:**

```
INFO stats:
total_connections_received: 1234567
total_commands_processed: 9876543
instantaneous_ops_per_sec: 45000  ← throughput
total_net_input_bytes: 456789
total_net_output_bytes: 789012

expired_keys: 5432  ← keys that expired
evicted_keys: 234   ← keys that were evicted (memory pressure)

keyspace_hits: 987654     ← successful reads
keyspace_misses: 12345    ← cache misses

hit_rate = 987654 / (987654 + 12345) = 98.8% ✓

INFO memory:
used_memory: 1073741824 (1 GB)
used_memory_human: 1.00G
used_memory_rss: 1207959552
used_memory_peak: 1207959552
mem_fragmentation_ratio: 1.12  ← >1.1 is bad, means fragmentation
```

**Пример.**

```python
import redis
import time
from datetime import datetime, timedelta

class CacheMonitor:
    def __init__(self, redis_client: redis.Redis):
        self.r = redis_client
        self.prev_hits = 0
        self.prev_misses = 0
        self.prev_time = time.time()

    def get_metrics(self):
        """Get current cache metrics"""
        info = self.r.info('stats')
        memory_info = self.r.info('memory')

        current_hits = info.get('keyspace_hits', 0)
        current_misses = info.get('keyspace_misses', 0)
        current_time = time.time()

        # Calculate hit rate
        total = current_hits + current_misses
        hit_rate = (current_hits / total * 100) if total > 0 else 0

        # Calculate deltas
        delta_hits = current_hits - self.prev_hits
        delta_misses = current_misses - self.prev_misses
        delta_time = current_time - self.prev_time

        # Calculate per-second rates
        hits_per_sec = delta_hits / delta_time if delta_time > 0 else 0
        misses_per_sec = delta_misses / delta_time if delta_time > 0 else 0

        # Memory metrics
        used_memory = memory_info.get('used_memory', 0)
        used_memory_peak = memory_info.get('used_memory_peak', 0)
        mem_fragmentation = memory_info.get('mem_fragmentation_ratio', 0)

        metrics = {
            'timestamp': datetime.now(),
            'hit_rate': hit_rate,
            'total_hits': current_hits,
            'total_misses': current_misses,
            'hits_per_sec': hits_per_sec,
            'misses_per_sec': misses_per_sec,
            'evicted_keys': info.get('evicted_keys', 0),
            'expired_keys': info.get('expired_keys', 0),
            'used_memory_mb': used_memory / 1024 / 1024,
            'peak_memory_mb': used_memory_peak / 1024 / 1024,
            'mem_fragmentation': mem_fragmentation,
            'connected_clients': info.get('connected_clients', 0),
            'instantaneous_ops_per_sec': info.get('instantaneous_ops_per_sec', 0),
        }

        # Update state
        self.prev_hits = current_hits
        self.prev_misses = current_misses
        self.prev_time = current_time

        return metrics

    def print_metrics(self):
        """Print human-readable metrics"""
        metrics = self.get_metrics()

        print(f"\n=== Cache Metrics ({metrics['timestamp']} ===")
        print(f"Hit Rate:            {metrics['hit_rate']:.1f}%")
        print(f"Hits/Sec:            {metrics['hits_per_sec']:.0f}")
        print(f"Misses/Sec:          {metrics['misses_per_sec']:.0f}")
        print(f"Ops/Sec:             {metrics['instantaneous_ops_per_sec']:.0f}")
        print(f"Memory Used:         {metrics['used_memory_mb']:.1f} MB")
        print(f"Memory Peak:         {metrics['peak_memory_mb']:.1f} MB")
        print(f"Memory Fragmentation:{metrics['mem_fragmentation']:.2f}")
        print(f"Evicted Keys:        {metrics['evicted_keys']}")
        print(f"Connected Clients:   {metrics['connected_clients']}")

        # Health check
        if metrics['hit_rate'] < 80:
            print("⚠️  WARNING: Low hit rate!")
        if metrics['mem_fragmentation'] > 1.5:
            print("⚠️  WARNING: High memory fragmentation!")
        if metrics['evicted_keys'] > 1000:
            print("⚠️  WARNING: High eviction rate!")

    def monitor_continuous(self, interval: int = 10):
        """Continuous monitoring"""
        try:
            while True:
                self.print_metrics()
                time.sleep(interval)
        except KeyboardInterrupt:
            print("\nMonitoring stopped")


# Application-level metrics (supplement Redis INFO)
class AppCacheMetrics:
    def __init__(self):
        self.hits = 0
        self.misses = 0
        self.errors = 0
        self.latencies = []

    def record_hit(self, latency_ms: float):
        self.hits += 1
        self.latencies.append(latency_ms)

    def record_miss(self, latency_ms: float):
        self.misses += 1
        self.latencies.append(latency_ms)

    def record_error(self):
        self.errors += 1

    def get_stats(self):
        total = self.hits + self.misses
        hit_rate = (self.hits / total * 100) if total > 0 else 0
        avg_latency = sum(self.latencies) / len(self.latencies) if self.latencies else 0

        return {
            'hit_rate': hit_rate,
            'total_hits': self.hits,
            'total_misses': self.misses,
            'errors': self.errors,
            'avg_latency_ms': avg_latency,
        }


# Usage
r = redis.Redis()
monitor = CacheMonitor(r)

# Continuous monitoring
# monitor.monitor_continuous(interval=10)

# Single snapshot
monitor.print_metrics()

# Application metrics
app_metrics = AppCacheMetrics()

def get_with_metrics(key: str):
    start = time.time()
    try:
        value = r.get(key)
        latency = (time.time() - start) * 1000  # ms

        if value:
            app_metrics.record_hit(latency)
        else:
            app_metrics.record_miss(latency)

        return value
    except Exception:
        app_metrics.record_error()
        raise

# Simulate some cache operations
for i in range(100):
    get_with_metrics(f"key:{i % 10}")

print("\n=== Application Metrics ===")
stats = app_metrics.get_stats()
for key, value in stats.items():
    print(f"{key}: {value}")
```

**Типичные ошибки.**
- Не мониторить cache hit rate — blind to problems
- Игнорировать memory fragmentation — waste of memory
- High eviction rate без action — cache переполнен
- Не корреллировать cache metrics с DB latency — не видно impact

**На интервью.**
- Назови 5+ ключевых метрик cache
- Что считаешь нормальным hit rate? — > 90%
- Follow-up: «Что если hit rate 50%?» — либо неправильный TTL, либо cache не подходит для этого workload
- Follow-up: «Как отладить low hit rate?» — посмотреть access pattern, есть ли hot keys

---

### 10. Как масштабировать distributed cache при росте данных?

**Зачем спрашивают.** Cache растёт вместе с системой. Интервьюер проверяет понимание масштабирования без downtime.

**Короткий ответ.** Основные стратегии: 1) Вертикальное масштабирование (больше RAM на узел), 2) Горизонтальное масштабирование (больше узлов), 3) Tiered cache (L1 local + L2 distributed). При горизонтальном нужна миграция ключей через consistent hashing.

**Детальный разбор.**

**Вертикальное масштабирование (Scale-Up):**

```
Было: 1 узел × 64GB
Стало: 1 узел × 256GB

Плюсы:
- Простая реализация (buy bigger hardware)
- Нет сложности распределения

Минусы:
- Есть limit (AWS max 768GB per instance)
- Downtime при upgrade
- Одна точка отказа (нет HA)

Когда использовать: small-medium scale
```

**Горизонтальное масштабирование (Scale-Out):**

```
Было:   3 узла × 64GB = 192GB total
Стало:  6 узлов × 64GB = 384GB total

Плюсы:
- Нет limits (можно бесконечно)
- HA через репликацию
- Parallel processing
- No downtime (gradual migration)

Минусы:
- Сложность (consistent hashing, rebalancing)
- Network overhead (inter-node communication)
- Operational complexity

Когда использовать: large scale, high availability needed
```

**Миграция ключей при добавлении узла:**

```
Было 3 узла, добавляем 4-й:

Before resharding:
Node 1: slots 0-4095
Node 2: slots 4096-8191
Node 3: slots 8192-12287
Node 4: slots 12288-16383

After resharding:
Node 1: slots 0-3071
Node 2: slots 3072-6143
Node 3: slots 6144-9215
Node 4: slots 9216-12287
... (равномерно распределены)

Миграция (no downtime):
1. Новый узел присоединяется с пустыми слотами
2. Постепенная миграция slots: старый узел → новый узел
3. Ключи в слотах перемещаются на фоне
4. Приложение видит перенаправления (MOVED)

Процесс:
redis-cli --cluster reshard <host>:<port>
  --cluster-from <source-node-id>
  --cluster-to <target-node-id>
  --cluster-slots 1024
  --cluster-yes
```

**Tiered Cache (многоуровневый cache):**

```
Архитектура:

┌──────────────────────────────┐
│    Application Layer         │
└────────────┬─────────────────┘
             │
    ┌────────▼──────────┐
    │  L1: Local Cache  │  (в памяти процесса)
    │  (fast, small)    │  - in-memory map
    │  <100MB           │  - <1μs latency
    │  Hit rate: 20%    │
    └────────┬──────────┘
             │ (L1 miss)
    ┌────────▼──────────────────┐
    │  L2: Distributed Cache    │  (Redis cluster)
    │  (bigger, shared)         │  - network latency
    │  100GB+                   │  - ~1ms latency
    │  Hit rate: 80%            │  - shared across services
    └────────┬──────────────────┘
             │ (L2 miss)
    ┌────────▼──────────────────┐
    │  L3: Database             │
    │  (source of truth)        │
    │  (slow, big)              │
    │  - ~50ms latency          │
    └───────────────────────────┘

Combined hit rate:
20% from L1 + 80% × 80% from L2 = 20% + 64% = 84%

Combined latency:
84% × 1μs (L1) + 16% × 1ms (L2) ≈ 0.16ms
```

**Пример.**

```python
import redis
from threading import Lock
from collections import OrderedDict
import time

# ===== L1: LOCAL CACHE (in-process) =====
class LocalLRUCache:
    def __init__(self, max_size: int = 10000):
        self.cache = OrderedDict()
        self.max_size = max_size
        self.lock = Lock()

    def get(self, key: str):
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
                return self.cache[key]
        return None

    def set(self, key: str, value):
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
            self.cache[key] = value

            if len(self.cache) > self.max_size:
                self.cache.popitem(last=False)  # Remove oldest

    def delete(self, key: str):
        with self.lock:
            self.cache.pop(key, None)


# ===== L2: DISTRIBUTED CACHE (Redis) =====
class TieredCache:
    """Two-level cache: L1 (local) + L2 (distributed)"""

    def __init__(self, redis_client: redis.Redis, max_local_size: int = 10000):
        self.l1 = LocalLRUCache(max_local_size)
        self.l2 = redis_client

        self.l1_hits = 0
        self.l2_hits = 0
        self.l2_misses = 0

    def get(self, key: str, fetch_fn=None):
        # L1: Check local cache
        value = self.l1.get(key)
        if value is not None:
            self.l1_hits += 1
            return value

        # L2: Check distributed cache
        value = self.l2.get(key)
        if value is not None:
            self.l2_hits += 1
            self.l1.set(key, value)  # Populate L1
            return value

        # L3: Fetch from source
        if fetch_fn:
            value = fetch_fn()
            if value is not None:
                self.set(key, value)
            self.l2_misses += 1

        return value

    def set(self, key: str, value):
        self.l1.set(key, value)
        self.l2.set(key, value, ex=3600)

    def delete(self, key: str):
        self.l1.delete(key)
        self.l2.delete(key)

    def get_stats(self):
        total = self.l1_hits + self.l2_hits + self.l2_misses
        if total == 0:
            return {}

        return {
            'l1_hit_rate': self.l1_hits / total * 100,
            'l2_hit_rate': self.l2_hits / total * 100,
            'miss_rate': self.l2_misses / total * 100,
            'total_requests': total,
        }


# ===== HORIZONTAL SCALING =====
class ScalableRedisCluster:
    """Manage Redis cluster scaling"""

    def __init__(self, nodes: list):
        self.rc = redis.cluster.RedisCluster(
            startup_nodes=nodes,
            decode_responses=True
        )

    def add_node(self, host: str, port: int):
        """Add new node to cluster"""
        print(f"Adding node {host}:{port}")

        # 1. Create new redis instance
        new_node = redis.Redis(host=host, port=port)
        new_node.cluster_meet(nodes[0]['host'], nodes[0]['port'])

        # 2. Reshard slots (gradual migration)
        # redis-cli --cluster reshard command
        print("Resharding slots...")
        # This would be done via redis-cli in production

    def remove_node(self, node_id: str):
        """Remove node from cluster"""
        print(f"Removing node {node_id}")

        # 1. Reshard slots away from node
        # 2. Wait for migration
        # 3. Delete node

    def get_cluster_info(self):
        """Get cluster status"""
        info = self.rc.execute_command("CLUSTER", "INFO")
        nodes = self.rc.execute_command("CLUSTER", "NODES")

        return {
            'info': info,
            'nodes': nodes,
        }


# ===== MIGRATION STRATEGY =====
def migrate_cache_data(source_rc: redis.cluster.RedisCluster,
                       dest_rc: redis.cluster.RedisCluster,
                       batch_size: int = 1000):
    """Migrate cache data from old cluster to new"""

    cursor = 0
    migrated = 0

    while True:
        # Scan keys in batches
        cursor, keys = source_rc.scan(cursor, count=batch_size)

        if not keys:
            break

        # Get values from source
        for key in keys:
            value = source_rc.get(key)
            ttl = source_rc.ttl(key)

            # Write to destination
            if ttl > 0:
                dest_rc.set(key, value, ex=ttl)
            else:
                dest_rc.set(key, value)

            migrated += 1

        if cursor == 0:
            break

    print(f"Migrated {migrated} keys")


# Example usage
cache = TieredCache(redis.Redis(host='localhost'))

# Populate with some data
for i in range(100):
    cache.set(f"key:{i}", f"value{i}")

# Access pattern (some cache hits)
for i in range(200):
    cache.get(f"key:{i % 100}")

# Stats
stats = cache.get_stats()
print("\n=== Tiered Cache Stats ===")
for key, value in stats.items():
    print(f"{key}: {value:.1f}%")
```

**Стратегии эвикции при нехватке памяти:**

```
Проблема: Cache переполнен, нужно освободить место

1. Aggressive TTL reduction
   maxmemory-policy: volatile-ttl
   → удалить keys с коротким TTL

2. LRU eviction
   maxmemory-policy: volatile-lru или allkeys-lru
   → удалить unused keys

3. Tiered approach
   L1: небольшая, only hot data
   L2: большая, distributed
   L3: database

   При нехватке памяти L2, данные:
   - Остаются в L1 (local cache)
   - Могут быть пересчитаны

4. Compression (сжатие)
   Сжимать values если > 1KB
   Trade-off: CPU vs memory
```

**Типичные ошибки.**
- Масштабировать без плана → chaos, downtime
- Забыть про миграцию ключей — они остаются на старых узлах
- Слишком маленький L1 cache — low efficiency
- Не мониторить hit rate после масштабирования — не видно impact

**На интервью.**
- Когда use vertical vs horizontal scaling?
- Объясни процесс добавления узла в Redis Cluster
- Follow-up: «Как минимизировать latency при миграции?» — batch migration, use pipeline
- Follow-up: «Зачем нужен tiered cache?» — улучшает hit rate и latency

---

## Практика

1. **Consistent Hashing** — реализовать с virtual nodes, добавить/удалить узел, проверить распределение ключей.

2. **LRU Cache** — реализовать с O(1) get/set, протестировать eviction.

3. **Cache Stampede Protection** — реализовать 3 способа (locking, early refresh, singleflight), сравнить.

4. **Cache-Aside Pattern** — реализовать с error handling, cascade invalidation.

5. **Tiered Cache** — L1 local + L2 distributed, мониторить hit rates на обоих уровнях.

6. **Redis Cluster Simulation** — эмулировать resharding, миграцию slots.

---

## Дополнительные материалы

- [Redis Official](https://redis.io/docs/) — документация Redis
- [Consistent Hashing Explained](https://www.toptal.com/big-data/consistent-hashing) — детальный гайд
- [Cache Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/cache-aside) — Microsoft Azure patterns
- [Scaling Memcached](https://www.nginx.com/blog/caching-strategies/) — NGINX caching strategies

---

← [06-search-autocomplete](./06-search-autocomplete.md) | [Трек System Design](./README.md) | [08-file-storage](./08-file-storage.md) →
