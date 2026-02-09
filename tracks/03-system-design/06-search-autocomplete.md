# 06 — Search Autocomplete / Typeahead

Развёрнутые вопросы и ответы про проектирование систем автодополнения поиска: Trie структуры данных, ранжирование, сбор статистики, персонализация, оптимизация latency, шардирование, обработка опечаток и масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [05-news-feed](./05-news-feed.md) · Следующая тема: [07-distributed-cache](./07-distributed-cache.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Trie** — это структура данных в виде дерева префиксов, где каждый узел представляет один символ. Trie позволяет выполнять поиск по префиксу за O(p) операций, где p — длина префикса, что даёт предсказуемо быстрый результат независимо от количества данных. Она идеально подходит для систем автодополнения, так как позволяет мгновенно найти все слова, начинающиеся с введённого префикса. Благодаря компактному хранению и быстрому поиску, Trie стала стандартом в поисковых системах.

**Precomputed top-K** — это техника предварительного вычисления топ-10 или топ-100 популярных подсказок на каждом узле Trie. Вместо того чтобы при каждом запросе проходить по всему поддереву и считать популярность, система хранит готовый список на каждом узле. Это снижает время ответа до O(1) и позволяет вернуть результаты почти мгновенно. Техника критична для систем с миллиардами запросов в день, где даже миллисекунды имеют значение.

**Consistent Hashing** — это алгоритм распределения данных по узлам на виртуальном кольце, который минимизирует переассоциацию ключей при добавлении или удалении серверов. Когда в кольцо добавляется новый узел, только примерно 1/N ключей переместятся на него, в отличие от простого хеша, где пришлось бы перехешировать почти все ключи. Это особенно важно для систем, работающих с горячими данными и кэшем, так как переассоциация большого количества ключей вызывает всплеск нагрузки.

**Cache Hit Rate** — это процент запросов, которые успешно найдены в кэше (например, 95 из 100 запросов попадают в кэш). Высокий cache hit rate (выше 80%) означает, что система эффективна и большинство обращений удовлетворяются из памяти. Это метрика здоровья системы: рост cache hit rate прямо ведёт к снижению latency и уменьшению нагрузки на основное хранилище.

**TTL (Time To Live)** — это время, в течение которого данные остаются актуальными в кэше перед автоматическим удалением. Например, TTL 5 минут означает, что через 5 минут запись исчезнет и следующий запрос будет загружать свежие данные из источника. TTL защищает от бесконечного хранения устаревших данных и помогает поддерживать баланс между свежестью и производительностью.

**Data Pipeline** — это асинхронный конвейер сбора и обработки данных, часто построенный как Kafka → Spark → Trie Builder. Входящие события (поисковые запросы) буферизируются в Kafka, затем обрабатываются Spark для подсчёта статистики и агрегации за часы и дни, и наконец результаты используются для периодического пересчёта Trie. Этот подход позволяет обрабатывать огромные объёмы данных без блокировки основного API.

**Levenshtein Distance** — это минимальное количество операций (вставка, удаление, замена символа) для преобразования одной строки в другую. Например, расстояние между "cat" и "car" равно 1 (одна замена). Эта метрика помогает системам обнаруживать опечатки и предлагать похожие слова: если пользователь напечатал "serch" вместо "search", система может найти опечатку и предложить исправленный вариант.

**Double Buffering** — это техника атомного обновления Trie через две версии структуры, хранящиеся одновременно в памяти. Когда нужно обновить Trie новыми данными, система строит новую версию в фоновом режиме, а затем переключается на неё одним переводом указателя. Это позволяет обновлять данные без downtime: клиенты плавно переходят с старой версии на новую без перерывов в обслуживании.

**CDN Caching** — это кэширование данных на географически распределённых узлах на краю сети, ближе к конечным пользователям. Для автодополнения CDN кэширует популярные префиксы с коротким TTL (1-5 минут), обеспечивая близко к нулевой latency для самых частых запросов. CDN может быть "первой линией защиты", вернув результат за 10-20 мс, прежде чем запрос достигнет основного сервера.

**Sharding** — это разделение данных по нескольким серверам по какому-то ключу (например, по первому символу или по хешу). В контексте автодополнения, все Trie для запросов, начинающихся с буквы "А", могут храниться на одном сервере, буквы "B" на другом и т.д. Это позволяет масштабировать систему горизонтально: каждый шард обрабатывает свою долю трафика, и при росте нагрузки можно добавить новые серверы.

**Collaborative Filtering** — это техника рекомендаций на основе анализа поведения похожих пользователей. Если пользователь A и пользователь B часто ищут одинаковые вещи, то рекомендации, которые нравятся A, могут быть предложены B. В контексте автодополнения, это позволяет персонализировать предложения: каждый пользователь видит не только топ-популярные, но и релевантные для его интересов варианты.

**Debouncing** — это техника задержки обработки частых событий от клиента. Например, вместо отправления запроса на сервер при вводе каждого символа (10+ запросов при наборе "example"), система ждёт 300 миллисекунд после последнего символа, а затем отправляет один запрос. Это значительно снижает нагрузку на сервер и улучшает пользовательский опыт, так как клиент не получает множество промежуточных результатов.

---

## Вопросы и разборы

### 1. Как спроектировать search autocomplete систему на высоком уровне?

**Зачем спрашивают.** Автодополнение — одна из самых часто используемых фич в поисковых системах. Интервьюер проверяет умение спроектировать систему под высокую нагрузку, учитывая требования по latency, freshness и масштабируемости.

**Короткий ответ.** Система состоит из трёх компонентов: (1) сбор статистики поисковых запросов через Kafka в Data Pipeline, (2) построение Trie структуры данных с precomputed top-K suggestions в памяти, (3) fast retrieval через in-memory Trie или Redis с кэшированием на уровне CDN и клиента.

**Детальный разбор.**

**Требования:**
- Функциональные: typeahead suggestions по мере ввода, top suggestions по популярности, персонализация, поддержка нескольких языков
- Нефункциональные: ultra-low latency (<100ms), high availability (99.99%), масштабирование на 10B queries/day

**Архитектура:**
```
Запрос от пользователя
        │
        ▼
┌───────────────────┐
│ Load Balancer     │
│ (distributes)     │
└────────┬──────────┘
         │
    ┌────┴────────────────────────────┐
    │                                 │
┌───▼──────────────────┐    ┌────────▼───────┐
│ Autocomplete Service │    │ Search Service  │
│ (fast lookup)        │    │ (query logging) │
└───┬──────────────────┘    └────────┬───────┘
    │                                │
    │        ┌────────────────────────┘
    │        │
    ▼        ▼
┌─────────────────────┐    ┌──────────────────┐
│ Redis/Trie Shards   │    │ Kafka (logs)     │
│ (in-memory)         │    │                  │
└─────────────────────┘    └──────────┬───────┘
                                      │
                           ┌──────────▼───────┐
                           │ Data Pipeline    │
                           │ (Spark/Flink)    │
                           └──────────┬───────┘
                                      │
                           ┌──────────▼───────┐
                           │ Trie Builder     │
                           │ (batch job)      │
                           └──────────┬───────┘
                                      │
                           ┌──────────▼───────┐
                           │ Trie DB (update) │
                           │ (Redis/Memcached)│
                           └──────────────────┘
```

**Оценка пропускной способности:**
```
Входные данные: 10B запросов в день
- ~5 символов на запрос в среднем
- 10B × 5 = 50B запросов autocomplete в день
- QPS = 50B / 86400 ≈ 580,000 QPS (среднее)
- Пик: 1.5M QPS (в 3 раза выше среднего)

Хранилище:
- 5M уникальных поисковых терминов
- Средний терм: 20 символов = 20 байт
- С метаданными (score, frequency): ~100 байт на терм
- Всего: 5M × 100B = 500MB (влезает в памяти одного сервера!)
- С репликацией (3x): 1.5GB на shard

Пропускная способность:
- Ответ: ~1KB (10 suggestions × 100 байт каждое)
- 580K QPS × 1KB = 580MB/s исходящих данных
- Нужна пропускная способность 5-10 Gbps (учитывая пики и резервирование)
```

**Ключевые компоненты:**
1. **Query Logger** — асинхронно логирует поисковые запросы в Kafka
2. **Data Pipeline** — Spark/Flink job агрегирует query frequency по часам/дням
3. **Trie Builder** — batch job строит Trie с precomputed top-K suggestions
4. **Autocomplete Service** — fast lookup в памяти, возвращает suggestions за <10ms
5. **Personalization Engine** — boost suggestions based on user history, trending topics
6. **CDN & Client Cache** — кэширование popular prefixes, debouncing на клиенте

**Пример.**
```python
# API высокого уровня
GET /api/v1/autocomplete?q=python&limit=10&user_id=user123

Ответ:
{
    "suggestions": [
        {"text": "python tutorial", "score": 95000},
        {"text": "python programming", "score": 87000},
        {"text": "python for beginners", "score": 82000}
    ],
    "query_time_ms": 8,
    "is_personalized": true
}

# Batch pipeline (каждый час)
1. Читаем логи из Kafka
2. Группируем по запросу
3. Считаем frequency и количество уникальных пользователей
4. Score = frequency * 0.7 + unique_users * 0.3
5. Строим Trie с топ-10 для каждого префикса
6. Отправляем в Redis (atomic update)
```

**Типичные ошибки.**
- Использовать обычную базу данных вместо in-memory Trie — не будет достаточно быстро при 580K QPS.
- Пытаться обновлять suggestions в real-time для каждого запроса — слишком дорого. Batch update (hourly) работает лучше.
- Не шардировать data — single node не выдержит 1.5M peak QPS.
- Забыть про client-side debouncing — генерирует ненужные запросы на сервер.
- Игнорировать freshness требования — старые suggestions не релевантны.

**На интервью.**
- Объясни capacity estimation и почему data fits in memory.
- Упомяни, что Trie с precomputed top-K даёт O(1) lookup для каждого prefix.
- Уточняющий вопрос: «Как обновить suggestions без downtime?» — double buffering или consistent hashing.
- Уточняющий вопрос: «Как обработать распределённую систему?» — sharding by first character.

---

### 2. Как работает Trie структура данных для autocomplete?

**Зачем спрашивают.** Trie (prefix tree) — оптимальная структура для prefix-based search. Интервьюер проверяет понимание классических структур данных и их применения.

**Короткий ответ.** Trie — дерево, где каждый узел представляет символ. На каждом узле хранятся precomputed top-K suggestions, что даёт O(K) ответ вместо O(n log n) сортировки. Построение: O(n × m), где n — количество слов, m — средняя длина. Поиск: O(p), где p — длина prefix.

**Детальный разбор.**

**Структура Trie для autocomplete:**
```
        (root)
       /  |  \
      c   h   p
      |   |   |
      a   o   y
     / \  |  / \
    t   r w  t   s
    |   |  |  |
   *   e  n  h
      / \    |  \
     d   e   *   o
     |    |      |
    *     *      n
                 |
                 *

Примечание: * означает конец слова

Слова: "cat", "car", "card", "care", "how", "hown", "python", "python" (дубликат)

На каждом узле (node):
- node.children: dict[char] -> TrieNode
- node.is_end: bool (это конец слова?)
- node.top_suggestions: list[(score, word)] (топ-K suggestions)
```

**Подход предварительно вычисленного top-K:**
```
При insertion "python" с score=95000:
root.children['p'].top_suggestions = [("python", 95000)]
root.children['p'].children['y'].top_suggestions = [("python", 95000)]
...
root.children['p'].children['y'].children['t'].children['h'].children['o'].top_suggestions = [("python", 95000)]
root.children['p'].children['y'].children['t'].children['h'].children['o'].children['n'].top_suggestions = [("python", 95000), ...]

При поиске prefix "pyt":
1. Пройти по дереву: p -> y -> t
2. Вернуть node['t'].top_suggestions[:10]
3. Время: O(|prefix| + K) = O(p + 10) ≈ O(1) для small K
```

**Память:**
```
Для 5M слов, средняя длина 20 символов:
- Узлов в дереве: ~5M × 20 = 100M узлов (хуже всего)
- На самом деле: ~10-20M узлов (много слов шарят prefixes)

На узел:
- children dict: ~256 entries (max) × 8 bytes (pointer) = 2KB в worst case
- Но средний узел: ~3-5 children
- top_suggestions list: 10 entries × (string ref + score) ≈ 200 bytes
- Итого на узел: ~400-500 bytes

Память: 20M nodes × 500 bytes = 10GB (это всё равно acceptable для distributed system)
```

**Коллизии и оптимизации:**
```
1. Часто используемые prefixes vs редкие:
   - "the" — часто встречается, много suggestions
   - "xyz..." — редко встречается, мало suggestions

2. Обрезание редких слов:
   - Удаляем слова с frequency < threshold
   - Освобождаем память, не теряя качества

3. Compressed Trie (PATRICIA tree):
   - Узлы с одним child объединяются в edges с labels
   - Экономит память ~30%

4. Lazy loading:
   - Не загружаем всё в памяти
   - Используем Redis с lazy hydration
```

**Пример.**
```python
class TrieNode:
    def __init__(self):
        self.children = {}  # char -> TrieNode
        self.is_end = False
        self.top_suggestions = []  # [(score, word), ...] sorted descending

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str, score: int):
        """Insert word with score, update top suggestions at each node"""
        node = self.root

        for char in word.lower():
            if char not in node.children:
                node.children[char] = TrieNode()

            node = node.children[char]
            self._update_top_suggestions(node, word, score)

        node.is_end = True

    def _update_top_suggestions(self, node: TrieNode, word: str, score: int, k: int = 10):
        """Maintain top K suggestions sorted by score"""
        # Remove if exists (update case)
        suggestions = [(s, w) for s, w in node.top_suggestions if w != word]

        # Add new
        suggestions.append((score, word))

        # Sort by score descending
        suggestions.sort(key=lambda x: x[0], reverse=True)

        # Keep top K
        node.top_suggestions = suggestions[:k]

    def search(self, prefix: str, limit: int = 10) -> list:
        """Search for prefix, return top suggestions"""
        node = self.root

        # Navigate to prefix node
        for char in prefix.lower():
            if char not in node.children:
                return []
            node = node.children[char]

        # Return precomputed top suggestions
        results = []
        for score, word in node.top_suggestions[:limit]:
            results.append({
                "text": word,
                "score": score
            })

        return results

# Usage
trie = Trie()
trie.insert("python tutorial", score=95000)
trie.insert("python programming", score=87000)
trie.insert("python for beginners", score=82000)

suggestions = trie.search("python", limit=10)
# [{"text": "python tutorial", "score": 95000}, ...]
```

**Типичные ошибки.**
- Не precomputing top-K на каждом узле — требует O(n) сортировки при каждом поиске.
- Хранить ссылку на весь список suggestions вместо top-K — тратит память впустую.
- Не обновлять parent nodes при update — inconsistent suggestions.
- Забыть про дедупликацию — если word уже есть с higher score, удалить старый.

**На интервью.**
- Нарисуй структуру для нескольких слов и покажи, как работает поиск.
- Объясни, почему O(p + K) лучше, чем O(n log n) для каждого запроса.
- Уточняющий вопрос: «Как обновить Trie без перестроения всего?» — incremental updates с merge операциями.
- Уточняющий вопрос: «Что если Trie не влезает в памяти?» — sharding по prefixes, lazy loading, compression.

---

### 3. Как реализовать ranking для suggestions?

**Зачем спрашивают.** Просто возвращать популярные suggestions недостаточно. Интервьюер проверяет умение комбинировать множество сигналов для качественного ранжирования.

**Короткий ответ.** Используй multi-factor scoring: base score (frequency × unique users), recency boost (trending topics), personalization boost (user history), context boost (location, device, time). Финальный score: base × (1 + boost1 + boost2 + ...).

**Детальный разбор.**

**Факторы ранжирования:**
```
Базовый скор (60% вес):
  score_base = frequency × 0.7 + unique_users × 0.3

Recency (15% вес):
  boost_recency = log(1 + (today - last_seen) ^ -0.5)
  Trending queries из последних 24 часов получают boost

Персонализация (15% вес):
  boost_personal = 0.5 / (1 + user_history_rank)
  Недавние поиски пользователя в boost

Context (10% вес):
  boost_context = location_score + device_score + time_score
  Локация: ищет "chinese restaurants" в Бангкоке
  Устройство: mobile vs desktop
  Время: "flu symptoms" популярнее зимой

Финальный: score_final = score_base × (1 + boost_recency + boost_personal + boost_context)
```

**Структура данных для ranking:**
```
Redis:
├── query:frequency (sorted set)
│   └── score = frequency count
│
├── query:unique_users (sorted set)
│   └── score = unique user count
│
├── user:123:history (sorted set)
│   └── score = timestamp
│
├── trending:queries:24h (sorted set)
│   └── score = trending score
│
└── location:US:queries (sorted set)
    └── score = query frequency in location
```

**Pipeline обновления scores:**
```
Batch job каждый час:
┌──────────────────────────┐
│ Исходные логи запросов   │
└────────┬─────────────────┘
         │
┌────────▼──────────────────────┐
│ Слой агрегации                │
│ - Считаем frequency           │
│ - Уникальные пользователи     │
│ - Decay во времени            │
└────────┬──────────────────────┘
         │
┌────────▼──────────────────────┐
│ Вычисление scores             │
│ - Применяем веса              │
│ - Boost trending              │
│ - Кэшируем результаты         │
└────────┬──────────────────────┘
         │
┌────────▼──────────────┐
│ Redis update          │
│ - Atomic operations   │
│ - Maintain consistency│
└──────────────────────┘
```

**Пример.**
```python
from datetime import datetime, timedelta
import redis
import math

class SuggestionRanker:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.base_weight_freq = 0.7
        self.base_weight_users = 0.3

    def rank_suggestions(self, suggestions: list, user_id: str, context: dict) -> list:
        """
        Rank suggestions considering multiple factors.

        Args:
            suggestions: list of {"text": ..., "base_score": ...}
            user_id: for personalization
            context: {"location": "US", "device": "mobile", "time": datetime}

        Returns:
            sorted list with final_score
        """
        scored = []

        for suggestion in suggestions:
            query = suggestion["text"]
            base_score = suggestion["base_score"]

            # Calculate boosts
            recency_boost = self._get_recency_boost(query)
            personal_boost = self._get_personal_boost(user_id, query)
            context_boost = self._get_context_boost(query, context)

            # Combine scores (multiplicative with boosts)
            # score = base * (1 + boost1 + boost2 + ...)
            total_boost = recency_boost + personal_boost + context_boost
            final_score = base_score * (1.0 + total_boost)

            scored.append({
                **suggestion,
                "final_score": final_score,
                "boosts": {
                    "recency": recency_boost,
                    "personal": personal_boost,
                    "context": context_boost
                }
            })

        # Sort by final score
        scored.sort(key=lambda x: x["final_score"], reverse=True)
        return scored

    def _get_recency_boost(self, query: str) -> float:
        """Boost trending queries from last 24 hours"""
        trending_score = self.redis.zscore("trending:queries:24h", query)

        if trending_score is None:
            return 0.0

        # Normalize to [0, 0.3]
        return min(trending_score / 10000.0, 0.3)

    def _get_personal_boost(self, user_id: str, query: str) -> float:
        """Boost queries in user's recent search history"""
        history_key = f"user:{user_id}:history"

        # zrevrank returns rank (0 = most recent)
        rank = self.redis.zrevrank(history_key, query)

        if rank is None:
            return 0.0

        # Recent searches get higher boost: 0.5 / (1 + rank)
        return 0.5 / (1.0 + rank)

    def _get_context_boost(self, query: str, context: dict) -> float:
        """Boost based on location, device, time"""
        boost = 0.0

        # Location boost
        location = context.get("location")
        if location:
            location_key = f"location:{location}:queries"
            location_rank = self.redis.zrevrank(location_key, query)

            if location_rank is not None:
                boost += min(0.15, 0.15 / (1.0 + location_rank))

        # Device boost (simplified)
        device = context.get("device", "desktop")
        if device == "mobile":
            # Mobile users might prefer shorter queries
            if len(query) <= 30:
                boost += 0.05

        # Time of day boost
        current_hour = context.get("hour", datetime.now().hour)
        if 20 <= current_hour or current_hour < 6:
            # Late night/early morning: more entertainment searches
            if any(term in query.lower() for term in ["movie", "song", "game"]):
                boost += 0.05

        return min(boost, 0.25)  # Cap at 25%

# Usage in autocomplete request
def get_personalized_suggestions(prefix: str, user_id: str, context: dict):
    ranker = SuggestionRanker(redis_client)

    # Get base suggestions from Trie
    base_suggestions = trie.search(prefix, limit=50)  # Get more than needed

    # Rank with personalization
    ranked = ranker.rank_suggestions(base_suggestions, user_id, context)

    # Return top 10
    return ranked[:10]
```

**Типичные ошибки.**
- Неправильные веса — некоторые факторы доминируют, давя на остальные.
- Забыть про temporal decay — старые популярные queries доминируют forever.
- Не нормализовать scores перед комбинированием — один фактор может быть в [0, 1M].
- Слишком много факторов — сложно отладить и обслуживать.
- Игнорировать контекст пользователя — suggestions less relevant.

**На интервью.**
- Объясни, почему multiplicative model лучше, чем additive.
- Упомяни temporal decay и как его реализовать.
- Уточняющий вопрос: «Как A/B тестировать разные weights?» — shadow ranking с метриками CTR/conversion.
- Уточняющий вопрос: «Как быстро адаптировать к новым трендам?» — real-time scoring для trending queries.

---

### 4. Как собирать и агрегировать query statistics?

**Зачем спрашивают.** Без quality data источника, все suggestions плохие. Интервьюер проверяет умение проектировать data pipeline для сбора и обработки больших объёмов данных.

**Короткий ответ.** Используй sampling + async logging в Kafka для минимизации latency. Spark/Flink job агрегирует по часам, считает frequency и unique users. Результаты пишутся в sorted set Redis или Cassandra. Batch job (hourly) перестраивает Trie и pushает в production.

**Детальный разбор.**

**Архитектура data pipeline:**
```
┌─────────────────┐
│ User searches   │
│ (10B/day)       │
└────────┬────────┘
         │
         │ async logging (sampling)
         │
    ┌────▼─────────┐
    │ Kafka topics │
    │              │
    │ search_logs  │
    │ (high volume)│
    └────┬─────────┘
         │
         │ Spark streaming / Flink
         │
    ┌────▼──────────────────┐
    │ Aggregation layer     │
    │ - Window: 1 hour      │
    │ - Group by: query     │
    │ - Metrics:            │
    │   * frequency count   │
    │   * unique users      │
    │   * top locations     │
    │   * timestamps        │
    └────┬──────────────────┘
         │
         │ Cassandra / Redis
         │
    ┌────▼──────────────────┐
    │ Time-series storage   │
    │ query_daily_stats     │
    │ query_hourly_stats    │
    └────┬──────────────────┘
         │
         │ Batch job (hourly)
         │
    ┌────▼──────────────────┐
    │ Trie builder          │
    │ - Read from storage   │
    │ - Calculate scores    │
    │ - Build Trie          │
    └────┬──────────────────┘
         │
    ┌────▼──────────────────┐
    │ Redis push (atomic)   │
    │ - No downtime         │
    │ - Consistent hashing  │
    └──────────────────────┘
```

**Sampling strategy:**
```
Не логируем все 10B запросов в день — слишком дорого (50B автодополнений).
Вместо этого используем adaptive sampling:

1. High-frequency queries (>1000/day):
   - Log 10% (random sampling)
   - Still get accurate frequency estimates

2. Medium-frequency queries (10-1000/day):
   - Log 50% (increase precision)

3. Low-frequency / new queries:
   - Log 100% (capture all rare queries)
   - Important for trending detection

4. Deduplication:
   - Only log once per user per day
   - Prevents duplicate entries

Итог: ~5B logs / day (50% reduction) вместо 50B
```

**Агрегирование:**
```
Spark job (hourly):

input: kafka topic "search_logs"
  schema: {user_id, query, timestamp, ...}

transform:
  1. Parse and normalize query (lowercase, trim)
  2. Window by hour: [timestamp - (timestamp % 3600), ...)
  3. Group by: (hour, query)
  4. Aggregate:
     - COUNT(*) as frequency
     - COUNT(DISTINCT user_id) as unique_users
     - COLLECT_SET(location) as locations

output: parquet to S3
  schema: {hour, query, frequency, unique_users, locations}

Пример:
hour: 2024-01-15 14:00:00
query: "python tutorial"
frequency: 15000
unique_users: 8500
locations: ["US", "GB", "CA"]
```

**Вычисление итоговых scores:**
```python
class QueryAggregator:
    def __init__(self, spark):
        self.spark = spark

    def aggregate_queries(self, date_range: str):
        """
        Aggregate search queries from logs.

        Args:
            date_range: "2024-01-15" or "2024-01-15 to 2024-01-16"
        """
        # Read from Kafka/S3 logs
        logs = self.spark.read.parquet(f"s3://logs/search/{date_range}")

        # Normalize and aggregate
        agg = logs \
            .withColumn("query_normalized",
                       F.lower(F.trim(F.col("query")))) \
            .groupBy("query_normalized") \
            .agg(
                F.count("*").alias("frequency"),
                F.countDistinct("user_id").alias("unique_users"),
                F.collect_set("location").alias("top_locations"),
                F.min("timestamp").alias("first_seen"),
                F.max("timestamp").alias("last_seen")
            )

        # Calculate score
        scored = agg \
            .withColumn("base_score",
                       F.col("frequency") * 0.7 +
                       F.col("unique_users") * 0.3) \
            .withColumn("days_old",
                       F.datediff(F.current_date(), F.col("last_seen"))) \
            .withColumn("recency_decay",
                       F.when(F.col("days_old") <= 1, 1.0)
                        .when(F.col("days_old") <= 7, 0.9)
                        .when(F.col("days_old") <= 30, 0.7)
                        .otherwise(0.5)) \
            .withColumn("final_score",
                       F.col("base_score") * F.col("recency_decay"))

        return scored

    def build_trie_from_scores(self, scored_queries):
        """Build Trie from scored queries"""
        trie = Trie()

        # Collect to driver (data fits in memory)
        results = scored_queries.collect()

        for row in results:
            query = row.query_normalized
            score = int(row.final_score)
            trie.insert(query, score)

        return trie

    def update_redis_trie(self, trie):
        """Push Trie to Redis shards"""
        # Get all suggestions from Trie
        for prefix, suggestions in trie.traverse_with_prefix():
            shard = self.get_shard_for_prefix(prefix)

            # Use pipelined commands for atomicity
            pipe = shard.pipeline()
            key = f"autocomplete:prefix:{prefix}"

            # Clear old
            pipe.delete(key)

            # Add new (use zadd for sorted set)
            for score, word in suggestions:
                pipe.zadd(key, {word: score})

            # Set expiry (24 hours)
            pipe.expire(key, 86400)

            # Execute all at once
            pipe.execute()

# Usage in batch job
aggregator = QueryAggregator(spark_context)
scored = aggregator.aggregate_queries("2024-01-15")
trie = aggregator.build_trie_from_scores(scored)
aggregator.update_redis_trie(trie)
```

**Обработка горячих data:**
```
Проблема: данные обновляются каждый час, но какие prefixes обновляются?

Решение: cache invalidation

1. Отслеживаем какие prefixes изменились:
   - Помещаем старую/новую top-10 списки
   - Если changed, инвалидируем cache

2. Multi-level caching:
   - L1: CDN (most popular prefixes, 1 min TTL)
   - L2: Redis (all prefixes, 5 min TTL)
   - L3: Trie in-memory (full Trie, updated hourly)

3. Cache stampede prevention:
   - Использовать probabilistic early expiration (XFetch)
   - Или background refresh 1 min до expiry
```

**Типичные ошибки.**
- Логировать все запросы без sampling — слишком дорого на pipeline.
- Не дедупицировать по пользователю — одни и те же люди искажают frequency.
- Неправильный window (1 день вместо 1 часа) — теряется recency информация.
- Не обрабатывать spelling variations — "python" vs "pyhton" как разные queries.
- Игнорировать outliers — один спам может поднять query на top.

**На интервью.**
- Объясни, почему sampling важен и как выбрать rates.
- Упомяни, как обработать trending queries в real-time (separate pipeline).
- Уточняющий вопрос: «Как обнаружить spam/abuse в queries?» — anomaly detection на frequency spike.
- Уточняющий вопрос: «Как обновлять Trie без downtime?» — consistent hashing + double buffering.

---

### 5. Как обновлять Trie в реальном времени?

**Зачем спрашивают.** Batch updates (hourly) удовлетворяют стандартным требованиям, но для trending topics нужны updates в real-time. Интервьюер проверяет умение балансировать свежесть данных и производительность.

**Короткий ответ.** Используй two-tier approach: (1) hourly batch updates для основных suggestions (99% трафика), (2) real-time trending pipeline для top-N emerging queries. Trending queries хранятся в отдельном sorted set с TTL 24 часа и мёрджатся на query time.

**Детальный разбор.**

**Two-tier update strategy:**
```
Tier 1: Batch (Hourly) — 99% traffic
├── Aggregates all queries from past hour
├── Calculates frequency and scores
├── Rebuilds Trie with precomputed top-K
├── Pushes to Redis atomically
└── TTL: 24 hours

Tier 2: Trending (Real-time) — 1% traffic (emerging queries)
├── Kafka stream detects spikes
├── Anomaly detection: frequency > 3σ from mean
├── Stores in separate "trending:24h" sorted set
├── Мергируется при запросе (наивысший приоритет)
└── TTL: 24 часа (автоматическое истечение)

Пример:
Обычный запрос: "python tutorial" (score 95000)
Trending запрос: "why is stock market down" (обнаружен spike, score 50000 + trending_boost)
Финальный рейтинг: trending запрос может быть выше, если boost достаточен
```

**Обнаружение trending в реальном времени:**
```
Kafka stream processing (Flink):

Окно: tumbling 1-минутные окна

Для каждого запроса в окне:
  1. Считаем текущую frequency
  2. Сравниваем с baseline (среднее за прошлые 7 дней)
  3. Если frequency > baseline * 3 (3σ): аномалия!
  4. Вычисляем trend_score = (current - baseline) / baseline
  5. Если trend_score > threshold (например, 2.0): emitим как trending

Trending запросы хранятся отдельно:
  ZADD trending:queries:24h {trend_score} "why is stock market down"
  EXPIRE trending:queries:24h 86400
```

**Архитектура для real-time обновлений:**
```
┌──────────────────────────────────┐
│ Kafka поисковый stream           │
│ (все запросы в реальном времени)  │
└────────────┬─────────────────────┘
             │
      ┌──────┴──────────────────────┐
      │                             │
   ┌──▼─────────────────┐  ┌────────▼────────────┐
   │ Агрегация         │  │ Обнаружение        │
   │ (каждый час)      │  │ аномалий           │
   │ - Обычные         │  │ (реальное время)   │
   │   updates      │     │ - Trending      │
   │               │     │   queries       │
   └──┬────────────┘     └────────┬────────┘
      │                          │
      │                    ┌─────▼─────────┐
      │                    │ Redis         │
      │                    │ trending:24h  │
      │                    └───────────────┘
      │
   ┌──▼──────────────────────┐
   │ Redis (Shards)          │
   │ autocomplete:prefix:*    │
   │ (Tier 1 data)           │
   └─────────────────────────┘
```

**Merger algorithm при query:**
```
GET /autocomplete?q=stock

1. Get Tier 1 suggestions (batch) from Redis
   Redis key: autocomplete:prefix:s -> [...]
   Redis key: autocomplete:prefix:st -> [...]
   Redis key: autocomplete:prefix:sto -> [...]
   Redis key: autocomplete:prefix:stoc -> [...]
   Redis key: autocomplete:prefix:stock -> ["stock market", ...]

2. Get Tier 2 suggestions (trending) from Redis
   Redis key: trending:queries:24h
   Extract matches: ["stock market crash", "stock buy", ...]

3. Merge results
   - Combine both lists
   - Sort by score (Tier 1 base + Tier 2 trending boost)
   - Return top 10

4. Priority:
   - If query is in trending list: apply boost
   - boost = trending_score * multiplier (e.g., 1.5x)
   - Re-rank against Tier 1
```

**Пример реализации:**
```python
class RealtimeAutocompleteService:
    def __init__(self, redis_client, trending_multiplier=1.5):
        self.redis = redis_client
        self.trending_multiplier = trending_multiplier
        self.trending_ttl = 86400  # 24 hours

    def get_suggestions(self, prefix: str, limit: int = 10) -> list:
        """Get suggestions with real-time trending merged"""

        # Tier 1: Get batch suggestions
        batch_suggestions = self._get_batch_suggestions(prefix, limit * 2)

        # Tier 2: Get trending suggestions
        trending_suggestions = self._get_trending_suggestions(prefix, limit * 2)

        # Merge and re-rank
        merged = self._merge_suggestions(
            batch_suggestions,
            trending_suggestions,
            limit
        )

        return merged

    def _get_batch_suggestions(self, prefix: str, limit: int) -> list:
        """Get suggestions from hourly batch Trie"""
        key = f"autocomplete:prefix:{prefix.lower()}"

        # Redis sorted set: zrevrange gets top by score
        results = self.redis.zrevrange(key, 0, limit - 1, withscores=True)

        suggestions = []
        for text, score in results:
            suggestions.append({
                "text": text,
                "score": float(score),
                "source": "batch"
            })

        return suggestions

    def _get_trending_suggestions(self, prefix: str, limit: int) -> list:
        """Get trending suggestions (real-time)"""

        # Get all trending queries
        trending = self.redis.zrevrange("trending:queries:24h", 0, -1, withscores=True)

        # Filter by prefix
        prefix_lower = prefix.lower()
        filtered = []

        for text, score in trending:
            if text.lower().startswith(prefix_lower):
                filtered.append({
                    "text": text,
                    "score": float(score),
                    "source": "trending"
                })

            if len(filtered) >= limit:
                break

        return filtered

    def _merge_suggestions(self, batch: list, trending: list, limit: int) -> list:
        """Merge batch and trending suggestions with boosting"""

        # Create lookup for batch suggestions
        batch_lookup = {s["text"]: s for s in batch}

        merged = []
        seen = set()

        # First, add trending suggestions with boost
        for suggestion in trending:
            text = suggestion["text"]

            if text in seen:
                continue
            seen.add(text)

            # Apply trending boost
            boosted_score = suggestion["score"] * self.trending_multiplier

            merged.append({
                "text": text,
                "score": boosted_score,
                "source": "trending",
                "is_trending": True
            })

        # Then, add batch suggestions not already in merged
        for suggestion in batch:
            text = suggestion["text"]

            if text in seen:
                continue
            seen.add(text)

            merged.append({
                "text": text,
                "score": suggestion["score"],
                "source": "batch",
                "is_trending": False
            })

        # Sort by score descending
        merged.sort(key=lambda x: x["score"], reverse=True)

        # Return top limit
        return merged[:limit]

    def detect_trending(self, query: str, current_frequency: int, baseline_frequency: float):
        """Called by real-time pipeline to detect and store trending queries"""

        if current_frequency <= baseline_frequency:
            return  # Not trending

        # Calculate trend score
        trend_score = (current_frequency - baseline_frequency) / (baseline_frequency + 1)

        # Threshold for trending: 2x baseline or 100+ absolute increase
        if trend_score < 2.0 and (current_frequency - baseline_frequency) < 100:
            return

        # Store in trending set with TTL
        key = "trending:queries:24h"
        score = current_frequency * (1.0 + trend_score)  # Boost score

        self.redis.zadd(key, {query: score})
        self.redis.expire(key, self.trending_ttl)

# Kafka stream processing for anomaly detection
from kafka import KafkaConsumer
from collections import deque
import time

class TrendingDetector:
    def __init__(self, redis_client, baseline_window_days=7):
        self.redis = redis_client
        self.baseline_window = baseline_window_days * 86400
        self.query_counts = {}  # query -> count in current window

    def process_search(self, query: str):
        """Process each search query in real-time"""

        query_lower = query.lower()

        # Increment count in current window
        if query_lower not in self.query_counts:
            self.query_counts[query_lower] = 0
        self.query_counts[query_lower] += 1

        # Check for trending (every 60 seconds)
        if int(time.time()) % 60 == 0:
            self._check_all_queries()

    def _check_all_queries(self):
        """Check all queries for trending (background task)"""

        for query, count in self.query_counts.items():
            # Get baseline from historical data
            baseline_key = f"query:baseline:{query}"
            baseline = float(self.redis.get(baseline_key) or 100)

            # Calculate trend
            trend_score = (count - baseline) / (baseline + 1)

            # If trending: store
            if trend_score > 2.0:
                self.redis.zadd("trending:queries:24h", {query: count})
                self.redis.expire("trending:queries:24h", 86400)

        # Reset counts for next window
        self.query_counts.clear()
```

**Типичные ошибки.**
- Не использовать TTL на trending queries — старые trending остаются навсегда.
- Применять слишком высокий boost — trending queries доминируют, давя на relевантные.
- Не детектировать anomalies правильно — spam может выглядеть как trending.
- Обновлять batch Trie слишком часто (более 1 часа) — waste compute resources.
- Игнорировать cache invalidation — старые suggestions из cache.

**На интервью.**
- Объясни, почему two-tier approach лучше, чем real-time for all.
- Упомяни, как дефектировать trending с confidence (3σ rule).
- Уточняющий вопрос: «Как обработать spam в trending?» — rate limiting per query, manual review, ML filtering.
- Уточняющий вопрос: «Как быстро реагировать на очень актуальные события?» — separate "breaking news" pipeline с даже shorter TTL.

---

### 6. Как реализовать персонализацию suggestions?

**Зачем спрашивают.** Персонализованные suggestions намного более relevant, чем generic. Интервьюер проверяет умение комбинировать user data с ranking signals.

**Короткий ответ.** Храни user search history в Redis sorted set (timestamp as score). При query time: boost suggestions, которые пользователь ищил раньше. Используй collaborative filtering для cross-user recommendations. Кэшируй personalization данные, чтобы избежать latency.

**Детальный разбор.**

**Источники персональных данных:**
```
1. Search history (в памяти)
   - Redis: user:123:history {timestamp} -> query
   - Последние 100 поисков пользователя
   - Boost коэффициент: recent = 0.5 / (1 + rank)

2. Click-through data (in Cassandra)
   - user:123:clicked_queries {count} -> query
   - Queries с higher CTR получают boost

3. Dwell time (time spent on result)
   - Если пользователь провел 5+ минут на результате
   - Suggestion более relevant

4. Demographic data (opt-in)
   - Age, gender, location, interests
   - For content-based filtering

5. Collaborative filtering
   - Похожие пользователи искали X
   - Recommend X если ты похож на них

6. Device & context
   - Mobile vs Desktop queries разные
   - Time of day patterns
```

**Архитектура personalization:**
```
User search request
        │
        ▼
┌───────────────────────────┐
│ Get base suggestions      │
│ (from Trie/batch)         │
└───────────┬───────────────┘
            │
    ┌───────┴────────────────────────┐
    │                                │
┌───▼──────────────┐    ┌───────────▼─────────┐
│ User history     │    │ Collaborative       │
│ (Redis)          │    │ filtering (Cassandra)│
│ - Boost recent   │    │ - Similar users     │
└───┬──────────────┘    └────────┬────────────┘
    │                           │
    │        ┌──────────────────┘
    │        │
┌───▼────────▼────────┐
│ Re-rank suggestions │
│ - Combine signals   │
│ - Final score       │
└───┬─────────────────┘
    │
┌───▼──────────────────┐
│ Cache & return       │
│ (Client cache: 1 min)│
└──────────────────────┘
```

**User history tracking:**
```
Redis data structure:

user:123:history (sorted set)
  score = timestamp (Unix)
  member = текст запроса

Пример:
ZADD user:123:history 1705339200 "python tutorial"
ZADD user:123:history 1705339300 "python django"
ZADD user:123:history 1705339400 "machine learning"

Обратный ранг (самое свежее первым):
ZREVRANGE user:123:history 0 -1
  1. "machine learning" (самое свежее)
  2. "python django"
  3. "python tutorial"

Обслуживание:
- EXPIRE user:123:history 7776000 (90 дней TTL)
- Хранить только последние 100 запросов (ZREMRANGEBYRANK ... -101 -1)
```

**Персонализированное ранжирование:**
```python
class PersonalizedRanker:
    def __init__(self, redis_client):
        self.redis = redis_client

    def personalize_suggestions(self, suggestions: list, user_id: str) -> list:
        """
        Boost suggestions based on user history and behavior.
        """

        # Load user data
        user_history = self._get_user_history(user_id)
        user_clicks = self._get_user_clicks(user_id)
        similar_users = self._get_similar_users(user_id, top_k=10)

        scored = []

        for suggestion in suggestions:
            query = suggestion["text"]
            base_score = suggestion["score"]

            # Personal boost: recent searches
            personal_boost = 0.0
            if query in user_history:
                rank = user_history[query]["rank"]
                personal_boost = 0.5 / (1.0 + rank)

            # Click-through boost
            click_boost = 0.0
            if query in user_clicks:
                clicks = user_clicks[query]
                click_boost = min(0.3, 0.1 * clicks)  # Cap at 30%

            # Collaborative boost: what similar users search
            collab_boost = 0.0
            similar_user_count = 0
            for sim_user_id in similar_users:
                sim_history = self._get_user_history(sim_user_id)
                if query in sim_history:
                    similar_user_count += 1

            if similar_user_count > 0:
                collab_boost = min(0.2, 0.02 * similar_user_count)

            # Final score
            total_boost = personal_boost + click_boost + collab_boost
            final_score = base_score * (1.0 + total_boost)

            scored.append({
                **suggestion,
                "final_score": final_score,
                "boosts": {
                    "personal": personal_boost,
                    "clicks": click_boost,
                    "collaborative": collab_boost
                }
            })

        # Sort by final score
        scored.sort(key=lambda x: x["final_score"], reverse=True)
        return scored

    def _get_user_history(self, user_id: str) -> dict:
        """Get user's search history with recency ranks"""
        key = f"user:{user_id}:history"

        # ZREVRANGE: most recent first
        history = self.redis.zrevrange(key, 0, -1, withscores=True)

        result = {}
        for rank, (query, timestamp) in enumerate(history):
            result[query] = {
                "rank": rank,
                "timestamp": timestamp
            }

        return result

    def _get_user_clicks(self, user_id: str) -> dict:
        """Get click-through data for user"""
        # This would come from Cassandra or another data store
        # For now, simplified

        key = f"user:{user_id}:clicks"
        clicks = self.redis.hgetall(key)

        return {query: int(count) for query, count in clicks.items()}

    def _get_similar_users(self, user_id: str, top_k: int) -> list:
        """Find users similar to this user"""
        # Simplified: use k-NN from precomputed similarity matrix
        # In reality: use approximate nearest neighbor search

        key = f"user:{user_id}:similar"
        similar = self.redis.zrevrange(key, 0, top_k - 1)

        return [user for user in similar]

    def record_search(self, user_id: str, query: str):
        """Record a search in user history"""
        key = f"user:{user_id}:history"
        timestamp = int(time.time())

        # Add to sorted set (newest has highest score)
        self.redis.zadd(key, {query: timestamp})

        # Keep only last 100
        self.redis.zremrangebyrank(key, 0, -101)

        # Set 90-day expiry
        self.redis.expire(key, 7776000)

    def record_click(self, user_id: str, query: str):
        """Record a click on search result"""
        key = f"user:{user_id}:clicks"

        # Increment click count
        self.redis.hincrby(key, query, 1)

        # Set expiry
        self.redis.expire(key, 2592000)  # 30 days
```

**Collaborative filtering:**
```python
class CollaborativeFiltering:
    def __init__(self, cassandra_client):
        self.cassandra = cassandra_client

    def get_recommendations_for_user(self, user_id: str) -> list:
        """
        Find queries that similar users like but this user hasn't searched yet.
        """

        # Find k nearest neighbors
        similar_users = self._get_knn_users(user_id, k=10)

        # Aggregate queries from similar users
        query_scores = {}

        for sim_user_id in similar_users:
            sim_history = self._get_user_history(sim_user_id)

            for query, count in sim_history.items():
                if query not in query_scores:
                    query_scores[query] = 0

                query_scores[query] += count

        # Filter out queries user has already searched
        user_history = set(self._get_user_history(user_id).keys())

        recommendations = [
            {"query": q, "score": s}
            for q, s in query_scores.items()
            if q not in user_history
        ]

        # Sort by score
        recommendations.sort(key=lambda x: x["score"], reverse=True)
        return recommendations[:10]

    def _get_knn_users(self, user_id: str, k: int) -> list:
        """Get k nearest neighbor users based on query history similarity"""
        # Use precomputed similarity scores from batch job
        # Cassandra: user_similarity (from_user, to_user, similarity_score)

        query = """
            SELECT to_user, similarity_score
            FROM user_similarity
            WHERE from_user = %s
            ORDER BY similarity_score DESC
            LIMIT %s
        """

        results = self.cassandra.execute(query, [user_id, k])
        return [row.to_user for row in results]

    def _get_user_history(self, user_id: str) -> dict:
        """Get user's complete search history from Cassandra"""
        query = """
            SELECT query, count
            FROM user_query_history
            WHERE user_id = %s
        """

        results = self.cassandra.execute(query, [user_id])
        return {row.query: row.count for row in results}
```

**Caching personalization:**
```
Проблема: Вычисление personalization для каждого пользователя дорого.
          Если 1M одновременных пользователей, запрашивать Redis 1M раз = медленно.

Решение: Client-side cache + TTL

1. Клиент кэширует suggestions для префикса на 1 минуту
2. Даже если пользователь ищет тот же префикс несколько раз
3. Personalization не меняется сильно за 1 минуту

Пример:
Пользователь вводит "python"
  - Запрос отправлен, personalization применена
  - Результаты закэшированы локально на 60 секунд
Пользователь вводит "python " (с пробелом)
  - Проверяем, есть ли "python" в кэше и <60 сек старость
  - Если да, используем закэшированные suggestions + применяем новый boost
  - Если нет, делаем новый запрос

В браузере:
localStorage: {
  "cache:python": {
    "suggestions": [...],
    "timestamp": 1705339200
  }
}

```

**Типичные ошибки.**
- Загружать весь user history для каждого запроса — N+1 query problem.
- Кэшировать personalized suggestions на сервере — разные пользователи нужны разные results.
- Игнорировать privacy — персональные данные чувствительны.
- Применять слишком сильный personal boost — generic suggestions важнее.
- Не обновлять similar users — collaborative filtering становится stale.

**На интервью.**
- Объясни, почему client-side cache лучше для personalization.
- Упомяни privacy concerns и как их адресовать (anonymization, opt-in).
- Уточняющий вопрос: «Как найти similar users на масштабе?» — LSH (Locality-Sensitive Hashing) или approximate nearest neighbors.
- Уточняющий вопрос: «Как обработать cold start (новый пользователь)?» — use demographic или content-based filtering.

---

### 7. Как оптимизировать latency для type-ahead?

**Зачем спрашивают.** Ultra-low latency (<100ms, ideally <50ms) — critical для user experience. Интервьюер проверяет знание техник оптимизации и трейд-оффов.

**Короткий ответ.** Multi-level caching (CDN → Redis → in-memory), параллельные запросы, batch processing на backend, client-side debouncing, префетчинг, и асинхронные операции без блокировок.

**Детальный разбор.**

**Разбор latency бюджета:**
```
Цель: <100ms end-to-end

Сетевая latency (US):
  - Клиент к ближайшему PoP: ~10-20ms (зависит от локации)
  - PoP к origin: ~20-50ms (зависит от расстояния)

Обработка на сервере:
  - Load balancing: ~1-2ms
  - Trie lookup: <1ms
  - Personalization: ~5-10ms
  - Ranking: ~2-3ms
  - Redis roundtrips: ~1-2ms
  - Serialization: <1ms

Сеть (возврат):
  - Ответ клиенту: ~10-20ms

Всего: ~50-90ms (если хорошо оптимизировано)

Если любой компонент >20ms, мы превышаем budget.
```

**Архитектура low-latency:**
```
                    Client (Browser)
                           │
                           │ HTTP/2
                           │
┌──────────────────────────┼──────────────────────────┐
│                          │                          │
│                    Cloudflare CDN                   │
│              (cache top 1000 prefixes)              │
│                          │                          │
│                    (HIT: return <5ms)               │
│                          │                          │
└──────────────────────────┼──────────────────────────┘
                          │
                (MISS or stale)
                          │
                          ▼
          ┌──────────────────────────────┐
          │ Load Balancer (GeoDNS)       │
          │ Routes to nearest region     │
          └──────────────┬───────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
    │ Region1 │     │ Region2 │     │ Region3 │
    │ US-East │     │ EU-West │     │ AP      │
    └────┬────┘     └────┬────┘     └────┬────┘
         │               │               │
    ┌────▼─────────────────────────────────────┐
    │ Autocomplete Service (multiple instances)│
    │                                         │
    │ In-memory Trie (300MB per instance)     │
    │ Redis local cache (hot prefixes)        │
    │                                         │
    │ Single threaded server:                  │
    │ - Go / Rust (fast)                      │
    │ - Minimal allocations                    │
    │ - Zero-copy serialization (protobuf)     │
    └─────────────────────────────────────────┘
```

**Multi-level caching:**
```
Level 1: Browser cache
  - Prefix -> suggestions
  - TTL: 1 minute
  - Hit rate: ~70% (users type similar prefixes)
  - Cost: Client-side storage

Level 2: CDN cache
  - Top 1000 prefixes
  - TTL: 1 minute
  - Hit rate: ~80% (top prefixes serve ~20% of queries)
  - Cost: CDN bandwidth (cheap)
  - Return: <10ms

Level 3: Redis cache (hot prefixes)
  - Top 10K prefixes
  - TTL: 5 minutes
  - Hit rate: ~95%
  - Cost: Redis memory
  - Return: ~2-5ms

Level 4: In-memory Trie (complete)
  - All prefixes
  - TTL: 1 hour (batch updates)
  - Hit rate: 100%
  - Cost: Server memory
  - Return: <1ms
```

**Параллелизм:**
```
Последовательный подход (медленно):
  1. Проверка кэша: 5ms
  2. Персонализация: 8ms
  3. Ранжирование: 3ms
  4. Сериализация: 1ms
  Всего: 17ms

Параллельный подход (быстрее):
  1. Проверка кэша: 5ms
  2. [Параллель] Персонализация: 8ms, Ранжирование: 3ms, Другое
  3. Слияние результатов: 1ms
  Всего: ~10ms (ограничено самой медленной задачей)

Реализация:
  - Запускаем goroutines для независимых задач
  - Используем channels для fan-in
  - Timeout если любая задача медленная
```

**Client-side optimizations:**
```
1. Debouncing (100ms)
   - User types: "p-y-t-h-o-n" (5 chars, 500ms)
   - Without debounce: 5 requests
   - With debounce: 1 request
   - Reduces server load 5x

2. Prefetching
   - User types "pyt"
   - Prefetch "python" and "pyt"
   - When user adds next character, we have cached result

3. Request cancellation
   - User types quickly: "pyth-o"
   - Cancel pending request for "pyth"
   - Only care about "pytho"

4. Progressive rendering
   - Return results as soon as first 5 available
   - Send "Loading..." state
   - Lazy load next 5

Example JavaScript:
```

**Пример оптимизированного backend:**
```go
// Ultra-fast autocomplete server in Go

type AutocompleteServer struct {
    trie *Trie                    // In-memory, precomputed
    redis *redis.Client            // Hot prefix cache
    ranker *PersonalizationRanker  // Fast scoring
}

func (s *AutocompleteServer) GetSuggestions(
    ctx context.Context,
    req *pb.AutocompleteRequest,
) (*pb.AutocompleteResponse, error) {

    prefix := req.GetQuery()
    userId := req.GetUserId()
    limit := int(req.GetLimit())

    // Timeout: must complete in 80ms max
    ctx, cancel := context.WithTimeout(ctx, 80*time.Millisecond)
    defer cancel()

    // Step 1: Lookup in Trie (in-memory, <1ms)
    baseSuggestions := s.trie.Search(prefix, limit*2)

    if len(baseSuggestions) == 0 {
        return &pb.AutocompleteResponse{}, nil
    }

    // Step 2-4: Parallel personalization + ranking
    // Use errgroup for goroutine management
    g, gctx := errgroup.WithContext(ctx)

    var (
        personalBoosts map[string]float32
        trendingBoosts map[string]float32
    )

    // Fetch personal boosts in parallel
    g.Go(func() error {
        var err error
        personalBoosts, err = s.ranker.GetPersonalBoosts(gctx, userId, baseSuggestions)
        return err
    })

    // Fetch trending boosts in parallel
    g.Go(func() error {
        var err error
        trendingBoosts, err = s.ranker.GetTrendingBoosts(gctx, baseSuggestions)
        return err
    })

    // Wait for both (up to 80ms timeout)
    if err := g.Wait(); err != nil {
        // Timeout or error: return base suggestions
        return s.buildResponse(baseSuggestions, nil, nil, limit)
    }

    // Step 5: Re-rank with boosts (single-threaded)
    rankedSuggestions := s.ranker.Rank(
        baseSuggestions,
        personalBoosts,
        trendingBoosts,
        limit,
    )

    // Step 6: Serialize to protobuf (zero-copy)
    response := s.buildResponse(rankedSuggestions, personalBoosts, trendingBoosts, limit)

    return response, nil
}

func (s *AutocompleteServer) buildResponse(
    suggestions []string,
    personalBoosts, trendingBoosts map[string]float32,
    limit int,
) *pb.AutocompleteResponse {

    resp := &pb.AutocompleteResponse{
        Suggestions: make([]*pb.Suggestion, 0, limit),
    }

    for i, text := range suggestions {
        if i >= limit {
            break
        }

        resp.Suggestions = append(resp.Suggestions, &pb.Suggestion{
            Text: text,
            IsTrending: trendingBoosts[text] > 0,
            IsPersonal: personalBoosts[text] > 0,
        })
    }

    return resp
}
```

**Типичные ошибки.**
- Синхронные вызовы для независимых операций — упускаешь parallelism.
- Кэшировать на сервере по user_id — каждый user разные results, плохая cache hit rate.
- Игнорировать timeout — slow query blocks other users.
- Не использовать zero-copy serialization — лишние аллокации.
- Слишком агрессивный debounce (>500ms) — user thinks system is slow.

**На интервью.**
- Нарисуй latency бюджет и покажи, где находятся bottlenecks.
- Упомяни, что параллелизм (Go goroutines) критичен.
- Уточняющий вопрос: «Как мониторить latency в production?» — P50/P99 latencies, SLI/SLO.
- Уточняющий вопрос: «Как справиться с tail latency (P99)?» — request hedging, response caching, fast fail.

---

### 8. Как хранить и шардировать данные autocomplete?

**Зачем спрашивают.** Sharding — неизбежен при масштабировании на 580K QPS. Интервьюер проверяет понимание trade-offs и практических соображений при распределении данных.

**Короткий ответ.** Используй consistent hashing по первому символу prefix (a-f, g-l, m-r, s-z, 0-9). Каждый shard хранит полный Trie для своего диапазона в памяти + Redis backup. Replication: 3x для HA. Rebalancing: seamless с consistent hashing.

**Детальный разбор.**

**Стратегии sharding:**
```
1. Hash-based sharding
   shard_id = hash(first_char(prefix)) % num_shards

   Pros:均匀распределение
   Cons: Rebalancing сложно при изменении num_shards

2. Range-based sharding (MEJOR)
   a-f   -> Shard 0
   g-l   -> Shard 1
   m-r   -> Shard 2
   s-z   -> Shard 3
   0-9   -> Shard 4

   Pros: Простое переразбиение, нет хеширования
   Cons: Может быть unbalanced (e.g., "the", "that" в Shard 3)

3. Prefix-based sharding
   1-3 chars по Trie path

   Pros: Баланс между диапазоном и хешем
   Cons: Сложнее реализовать

Рекомендация: Range-based по первому символу (5 shards)
- 5 shards = 5 Redis instances ~3GB each = 15GB total
- Каждый shard обрабатывает ~116K QPS
- Реplication: 3x = 45GB SSD
```

**Архитектура шардирования:**
```
                    Client request
                    GET /autocomplete?q=python
                           │
                    ┌───────▼───────┐
                    │ Load Balancer │
                    └───────┬───────┘
                            │
            ┌───────────────┴───────────────┐
            │                               │
   ┌────────▼────────┐          ┌──────────▼─────────┐
   │ Route by prefix │          │ Route: p -> Shard2 │
   │ first char (p)  │          │ (m-r range)        │
   └────────┬────────┘          └──────────┬─────────┘
            │                              │
     ┌──────┴──────────────────────────────┴──────┐
     │                                            │
┌────▼──────┐ ┌──────────┐ ┌────────────┐        │
│ Shard0    │ │ Shard1   │ │  Shard2    │        │
│ (a-f)     │ │ (g-l)    │ │  (m-r)     │        │
│           │ │          │ │            │◄───────┘
│ In-memory │ │ In-memory│ │ In-memory  │
│ Trie      │ │ Trie     │ │ Trie       │
│           │ │          │ │            │
│ Redis:    │ │ Redis:   │ │ Redis:     │
│ Replica   │ │ Replica  │ │ Replica    │
│ (backup)  │ │ (backup) │ │ (backup)   │
└───────────┘ └──────────┘ └────────────┘

               ┌──────────┐ ┌────────────┐
               │ Shard3   │ │  Shard4    │
               │ (s-z)    │ │  (0-9)     │
               │          │ │            │
               │ In-memory│ │ In-memory  │
               │ Trie     │ │ Trie       │
               │          │ │            │
               │ Redis:   │ │ Redis:     │
               │ Replica  │ │ Replica    │
               └──────────┘ └────────────┘
```

**Consistent hashing для load balancing:**
```
Проблема: Если добавить новый Shard, нужно перехешировать всё.

Решение: Consistent hashing (ring)

                    0 (a)
                   /     \
              f (1500)   b (500)
              /             \
          s (1400)         c (600)
          /                    \
       z (1300)              m (700)
         |                       |
         |                       |
       . (1200)              r (800)

Запрос "python":
  hash("python") = позиция на ring
  Найти следующий shard по часовой стрелке
  -> маршрутизация на этот shard

Добавление нового shard:
  - Только ключи между старым и новым shard движутся
  - Не все ключи перехешируются!
```

**Replication и failover:**
```
Primary-Replica setup для каждого shard:

         Load Balancer
         (предпочитаем Primary)
              │
      ┌───────┼───────┐
      │               │
   Primary      Replica 1
   (Leader)     (Standby)

   ┌─────────────────┐
   │  Shard 2        │
   │  (m-r prefix)   │
   │                 │
   │  Redis:         │
   │  - Host A (3GB) │
   │  - Host B (3GB) │
   │  - Host C (3GB) │
   │                 │
   │ Replication:    │
   │  - A -> B, C    │
   │  - Async writes │
   │  - TTL: 24h     │
   └─────────────────┘

Failover:
  1. Primary A падает
  2. Heartbeat падает, обнаружено за 5 сек
  3. Повышаем B до primary
  4. Маршрутизируем новые запросы на B
  5. Клиенты повторяют failed запросы
```

**Пример реализации:**
```python
class ShardedAutocompleteService:
    def __init__(self, num_shards: int = 5):
        self.num_shards = num_shards
        self.shards = {}  # shard_id -> (primary, replicas)
        self.range_map = self._create_range_map()

    def _create_range_map(self) -> dict:
        """Map first character to shard"""
        ranges = [
            ('a', 'f', 0),
            ('g', 'l', 1),
            ('m', 'r', 2),
            ('s', 'z', 3),
            ('0', '9', 4),
        ]

        range_map = {}
        for start, end, shard_id in ranges:
            for c in range(ord(start), ord(end) + 1):
                range_map[chr(c)] = shard_id

        return range_map

    def get_shard_id(self, prefix: str) -> int:
        """Get shard ID for prefix"""
        if not prefix:
            return 0

        first_char = prefix[0].lower()
        return self.range_map.get(first_char, 0)

    def get_suggestions(self, prefix: str, limit: int = 10) -> list:
        """Get suggestions from appropriate shard"""
        shard_id = self.get_shard_id(prefix)
        shard = self.shards[shard_id]

        # Try primary first
        primary = shard['primary']
        try:
            return primary.search(prefix, limit)
        except TimeoutError:
            # Fallback to replica
            for replica in shard['replicas']:
                try:
                    return replica.search(prefix, limit)
                except TimeoutError:
                    continue

            # All shards down
            raise Exception(f"Shard {shard_id} unavailable")

    def health_check(self):
        """Periodically check shard health"""
        for shard_id, shard in self.shards.items():
            primary = shard['primary']

            try:
                # Ping primary
                primary.ping()
                shard['primary_healthy'] = True
            except:
                shard['primary_healthy'] = False

                # Promote replica if primary down
                for i, replica in enumerate(shard['replicas']):
                    try:
                        replica.ping()
                        shard['primary'] = replica
                        shard['replicas'][i] = primary  # Old primary as replica
                        break
                    except:
                        continue

# Usage
service = ShardedAutocompleteService(num_shards=5)

# Request 1: "python"
shard_id = service.get_shard_id("python")  # -> 2 (m-r)
suggestions = service.get_suggestions("python")

# Request 2: "apple"
shard_id = service.get_shard_id("apple")  # -> 0 (a-f)
suggestions = service.get_suggestions("apple")
```

**Data migration при добавлении shard:**
```
Текущие: 5 shards
Добавляем: Shard 5 (для перераспределения)

План миграции:
1. Строим новый Trie на Shard 5 (с нуля или копируем)
2. Обновляем routing: некоторые ranges движутся на Shard 5
   Было: m-r -> Shard 2
   Стало: m-p -> Shard 2, q-r -> Shard 5

3. Движение данных (offline safe):
   - Читаем из Shard 2
   - Пишем в Shard 5
   - Проверяем консистентность
   - Cut over (atomic)

4. Мониторинг:
   - Проверяем latency не растёт
   - Мониторим success rates
   - Rollback если нужно
```

**Типичные ошибки.**
- Неправильное распределение (одна shard 100x больше других).
- Не дублировать для HA (single point of failure).
- Синхронная репликация — слишком медленно, используй async.
- Забыть про health checks — dead shard остаётся в rotation.
- Миграция без backpressure — overload новой shard.

**На интервью.**
- Объясни выбор range-based sharding vs hash-based.
- Упомяни replication и failover стратегию.
- Уточняющий вопрос: «Как перебалансировать при росте?» — consistent hashing или двойная запись во время migration.
- Уточняющий вопрос: «Как обработать очень популярный shard?» — read replicas, caching, splitting диапазона.

---

### 9. Как обрабатывать typos и fuzzy matching?

**Зачем спрашивают.** Users делают опечатки. Система должна исправлять их или находить похожие queries. Интервьюер проверяет знание NLP техник и их application в scale.

**Короткий ответ.** Используй Levenshtein distance для меры similarity. Для fast lookup: generate variations (add/delete/replace character). Хранить на отдельном Trie с prefix. Для production: используй вероятностные data structures (Bloom filter) для быстрой отраженной.

**Детальный разбор.**

**Типы опечаток:**
```
1. Substitution (замена символа)
   python -> pyton (y заменили на o)
   Levenshtein distance: 1

2. Deletion (удаление символа)
   python -> pyhon (удалили t)
   Levenshtein distance: 1

3. Insertion (вставка символа)
   python -> pythoon (вставили o)
   Levenshtein distance: 1

4. Transposition (перестановка)
   python -> ptnhoy (сложнее, Levenshtein не ловит)
   Damerau-Levenshtein distance: 2-3

5. Phonetic similarity (звучат похоже)
   python -> pythun (local accent)
   Metaphone/Soundex для detection
```

**Стратегия обработки:**
```
1. Client-side suggestions (instant)
   - While typing "pyton", show "Did you mean: python?"
   - Use local Trie (can be compressed, 10MB on phone)

2. Server-side correction (if not found)
   - User typed "pyton", no exact match in Trie
   - Search for similar: distance <= 1 or 2
   - Return corrected + suggestions

3. Ranking corrected results
   - Exact matches: score 100
   - Distance 1: score 80
   - Distance 2: score 50
   - Don't show if very dissimilar
```

**Levenshtein distance:**
```
Dynamic programming approach:

s1 = "python"
s2 = "pyton"

        ""  p  y  t  o  n
    ""   0  1  2  3  4  5
    p    1  0  1  2  3  4
    y    2  1  0  1  2  3
    t    3  2  1  0  1  2
    h    4  3  2  1  1  2
    o    5  4  3  2  1  2
    n    6  5  4  3  2  1

Distance = 1 (delete 'h')

Time: O(m × n), Space: O(min(m, n))
For "python" vs typo: ~40 operations
```

**Быстрая генерация вариаций (для Trie lookup):**
```
Word: "python"

Deletions (5):
  "ython", "pthon", "pyhon", "pyton", "pytho"

Substitutions (5 × 26 = 130):
  "python" -> "ayth on", "byth on", ..., "zyth on"
  "python" -> "payth on", "pbyth on", ...
  ...

Insertions (6 × 26 = 156):
  "python" -> "apython", "bpython", ..., "zpython"
  ...

Total: ~300 variations

Check each in Trie:
  For each variation, check if word exists
  If yes, and in top suggestions, return it

Проблема: Слишком много вариаций при большом edit distance!

Решение: Делаем distance 1 для fast path
         Distance 2 только если distance 1 ничего не нашёл
```

**Пример реализации:**
```python
class SpellCorrector:
    def __init__(self, trie: Trie, max_distance: int = 2):
        self.trie = trie
        self.max_distance = max_distance

    def correct(self, prefix: str, limit: int = 10) -> list:
        """
        Найти suggestions для потенциально неправильно написанного префикса.
        Возвращает: [(suggestion, is_corrected), ...]
        """

        # Сначала пытаемся точное совпадение
        suggestions = self.trie.search(prefix, limit)
        if suggestions:
            return [(s, False) for s in suggestions]

        # Нет точного совпадения, пытаемся corrections
        corrections = self.find_corrections(prefix, limit, max_dist=1)

        if corrections:
            return [(s, True) for s in corrections]

        # Try distance 2 if distance 1 found nothing
        corrections = self.find_corrections(prefix, limit, max_dist=2)

        return [(s, True) for s in corrections]

    def find_corrections(self, prefix: str, limit: int, max_dist: int = 1) -> list:
        """Find words within edit distance"""

        candidates = set()

        # Generate variations
        if max_dist >= 1:
            variations = self.generate_variations_distance_1(prefix)

            for var in variations:
                # Check if variation prefix exists in Trie
                found = self.trie.search(var, 1)
                if found:
                    candidates.update(found)

        if max_dist >= 2 and len(candidates) < limit:
            # Generate distance 2 only if needed
            variations = self.generate_variations_distance_2(prefix)

            for var in variations:
                found = self.trie.search(var, 1)
                if found:
                    candidates.update(found)

        # Score by Levenshtein distance
        scored = []
        for candidate in candidates:
            dist = self.levenshtein_distance(prefix, candidate)
            # Score: higher is better
            score = 100 - (dist * 20)
            scored.append((score, candidate))

        # Sort by score descending
        scored.sort(reverse=True)

        return [word for _, word in scored[:limit]]

    def generate_variations_distance_1(self, s: str) -> set:
        """Generate all variations at distance 1"""
        variations = set()

        # Deletions
        for i in range(len(s)):
            variations.add(s[:i] + s[i+1:])

        # Substitutions
        for i in range(len(s)):
            for c in 'abcdefghijklmnopqrstuvwxyz':
                if c != s[i]:
                    variations.add(s[:i] + c + s[i+1:])

        # Insertions
        for i in range(len(s) + 1):
            for c in 'abcdefghijklmnopqrstuvwxyz':
                variations.add(s[:i] + c + s[i:])

        return variations

    def generate_variations_distance_2(self, s: str) -> set:
        """Generate variations at distance 2 (expensive!)"""

        variations_d1 = self.generate_variations_distance_1(s)
        variations_d2 = set()

        for var in variations_d1:
            variations_d2.update(self.generate_variations_distance_1(var))

        return variations_d2

    @staticmethod
    def levenshtein_distance(s1: str, s2: str) -> int:
        """Calculate Levenshtein distance"""

        m, n = len(s1), len(s2)

        # Optimize space
        if m > n:
            s1, s2 = s2, s1
            m, n = n, m

        prev = list(range(n + 1))

        for i in range(1, m + 1):
            curr = [i] + [0] * n

            for j in range(1, n + 1):
                if s1[i - 1] == s2[j - 1]:
                    curr[j] = prev[j - 1]
                else:
                    curr[j] = 1 + min(
                        prev[j],      # deletion
                        curr[j - 1],  # insertion
                        prev[j - 1]   # substitution
                    )

            prev = curr

        return prev[n]
```

**Phonetic matching (для accent variations):**
```python
# Metaphone or Soundex for accent-insensitive matching

def metaphone(word: str) -> str:
    """Simple metaphone encoding"""
    # Rules for English phonetics
    # python -> PYTHN
    # pythan -> PYTHN (same!)

    # Simplified version
    word = word.upper()

    # Drop vowels (except first)
    result = word[0]
    for i in range(1, len(word)):
        if word[i] not in 'AEIOU':
            result += word[i]

    return result

def find_phonetic_matches(prefix: str, trie: Trie) -> list:
    """Find words that sound similar"""
    prefix_phonetic = metaphone(prefix)

    matches = []
    for suggestion in trie.all_suggestions():
        if metaphone(suggestion) == prefix_phonetic:
            matches.append(suggestion)

    return matches[:10]
```

**Bloom filter для быстрого rejection:**
```python
from pybloom import BloomFilter

class FastSpellChecker:
    def __init__(self, words: list, expected_elements: int = 5000000):
        # Create Bloom filter of common misspellings
        self.bloom = BloomFilter(capacity=expected_elements, error_rate=0.001)

        for word in words:
            self.bloom.add(word)

    def possibly_misspelled(self, word: str) -> bool:
        """Quick check: if not in Bloom filter, definitely misspelled"""
        return word not in self.bloom  # Bloom filter negative

    def correct(self, word: str) -> list:
        """Only do expensive Levenshtein if might be misspelled"""

        if word in self.bloom:
            return self.trie.search(word)

        # Definitely misspelled or very rare
        return self.find_corrections(word)
```

**Типичные ошибки.**
- Гепеневать все variations без фильтрации — explosion в количестве candidates.
- Максимальное расстояние >2 — results становятся неправильными.
- Использовать только Levenshtein без phonetic — не ловит accent variations.
- Не ранжировать corrected results — user видит random suggestions.
- Блокировать на spell checking при every request — latency spike.

**На интервью.**
- Объясни, почему distance 1-2 достаточно для most typos.
- Упомяни, как использовать Bloom filter для быстрого rejection.
- Уточняющий вопрос: «Как обрабатывать frequent misspellings?» — precomputed corrections в Trie.
- Уточняющий вопрос: «Как адаптироваться к user's typing patterns?» — ML model на top of corrections.

---

### 10. Как масштабировать autocomplete систему?

**Зачем спрашивают.** От 100K до 1M QPS требует разных approachов. Интервьюер проверяет практическое знание bottlenecks и solutions на разных scale.

**Короткий ответ.** Start: single Trie instance + Redis. 100K: multiple regions + CDN. 500K: sharding по prefix. 1M+: edge computing (FaaS на CDN PoPы). Monitoring: continuous profiling, load testing, capacity planning.

**Детальный разбор.**

**Масштабирование по stages:**
```
Stage 1: 10K QPS (MVP)
┌──────────┐
│ Single   │
│ Instance │
│ + Redis  │
└──────────┘
Ограничения: CPU/память одной машины
Действие: Обновить hardware если нужно

Stage 2: 50K-100K QPS
┌──────────────────────────┐
│ Множество инстансов      │
│ за LB                    │
│ + Redis cluster          │
│ + CDN для популярных      │
│   префиксов              │
└──────────────────────────┘
Ограничения: Network bandwidth, Redis memory
Действие: Добавить инстансы, sharding

Stage 3: 500K-1M QPS
┌───────────────────────────────────┐
│ Regional deployment               │
│ - North America                   │
│ - Europe                          │
│ - Asia                            │
│                                   │
│ Каждый region:                    │
│ - 5 shards                        │
│ - 3x replication                  │
│ - CDN cache                       │
│ - Hot prefix caching              │
└───────────────────────────────────┘
Ограничения: Cost, operational complexity
Действие: Оптимизировать per-region, edge caching

Stage 4: 10M+ QPS (Global)
┌────────────────────────────────────┐
│ Edge computing (CDN nodes)         │
│ - Compute at edge PoP              │
│ - Local Trie (compressed)          │
│ - 1-минутная sync обновлений       │
│ - Sub-5ms latency                  │
│                                    │
│ Origin:                            │
│ - Coordination                     │
│ - Analytics                        │
│ - Trie updates (каждый час)        │
└────────────────────────────────────┘
Ограничения: Consistency, operational overhead
```

**Bottleneck analysis и solutions:**
```
Bottleneck 1: CPU (Ranking, Personalization)
┌─────────────────────────────────────┐
│ Problem                             │
│ - Personalization too expensive     │
│ - 10% of QPS times out              │
└──────────────────┬──────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
    Solution A:        Solution B:
    Cache personal     Approximate
    boosts for 1min    personalization
    ↓                  ↓
  Hit rate: 70%       Fast path: <5ms
  Fallback: simple    Slow path: <50ms

Bottleneck 2: Memory (Trie size)
┌──────────────────────────────────────┐
│ Problem                              │
│ - Single Trie: 10GB                  │
│ - Need 3x replication                │
│ - 30GB RAM per region                │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
   Solution: Sharding   Solution: Compression
   - 5 shards           - Trie -> PATRICIA
   - 6GB each           - 30% memory savings
   - Total: 18GB        - 7GB total

   Both solutions:
   - Cost optimization
   - Better cache locality
```

**Архитектура для 500K+ QPS:**
```
┌──────────────────────────────────────────────────┐
│ Global Load Balancer (GeoDNS)                    │
│ - Route by geographic region                    │
│ - Health check all endpoints                    │
└────────────┬─────────────────────────────────────┘
             │
      ┌──────┴──────────────────────┐
      │                             │
   US East                       EU West
   (250K QPS)                    (250K QPS)
      │                             │
┌─────▼────────────────┐     ┌─────▼────────────────┐
│ CloudFlare PoPs      │     │ CloudFlare PoPs      │
│ (global)             │     │ (global)             │
│ - Cache top 1K       │     │ - Cache top 1K       │
│   prefixes           │     │   prefixes           │
│ - TTL: 1 minute      │     │ - TTL: 1 minute      │
└─────┬────────────────┘     └─────┬────────────────┘
      │ MISS (20%)               │ MISS (20%)
      │                          │
   ┌──▼──────────────────┐  ┌──▼──────────────────┐
   │ Origin: us-east-1   │  │ Origin: eu-west-1   │
   │                     │  │                     │
   │ LB + 10 instances   │  │ LB + 10 instances   │
   │                     │  │                     │
   │ Sharding:           │  │ Sharding:           │
   │ - 5 shards          │  │ - 5 shards          │
   │ - 3 replicas each   │  │ - 3 replicas each   │
   │ - Hot cache (Redis) │  │ - Hot cache (Redis) │
   │                     │  │                     │
   │ Analytics:          │  │ Analytics:          │
   │ - Kafka (logs)      │  │ - Kafka (logs)      │
   │ - Trie rebuilds     │  │ - Trie rebuilds     │
   │   (hourly)          │  │   (hourly)          │
   └─────────────────────┘  └─────────────────────┘
```

**Monitoring и observability:**
```
Metrics to track:

1. Request metrics:
   - QPS (queries per second)
   - Latency: p50, p99, p999
   - Error rate (4xx, 5xx)
   - Cache hit rate

2. Resource metrics:
   - CPU usage
   - Memory usage
   - Network bandwidth
   - Disk I/O (for analytics)

3. Application metrics:
   - Trie build time (hourly)
   - Trending detection latency
   - Personalization cache hit rate
   - Shard imbalance

4. Business metrics:
   - Autocomplete engagement rate
   - CTR (click through rate)
   - User satisfaction (NPS)

SLOs (Service Level Objectives):
   - p99 latency: < 100ms
   - Availability: 99.99%
   - Cache hit rate: > 80%

Alert triggers:
   - p99 latency > 150ms
   - Error rate > 0.1%
   - Cache hit rate < 70%
   - Trie build failure
```

**Пример мониторинга:**
```python
from prometheus_client import Counter, Histogram, Gauge

class MetricsCollector:
    def __init__(self):
        # Counters
        self.requests = Counter('autocomplete_requests_total',
                               'Total requests')
        self.cache_hits = Counter('cache_hits_total',
                                 'Cache hits by level',
                                 ['level'])
        self.errors = Counter('autocomplete_errors_total',
                             'Errors by type',
                             ['error_type'])

        # Histograms
        self.latency = Histogram('autocomplete_latency_ms',
                                'Request latency',
                                buckets=[5, 10, 25, 50, 100])

        # Gauges
        self.trie_size = Gauge('trie_size_bytes',
                              'Trie memory usage')
        self.personalization_cache_size = Gauge('personalization_cache_entries',
                                               'User cache entries')

    def record_request(self, start_time, cache_level, success):
        latency = (time.time() - start_time) * 1000  # ms

        self.requests.inc()
        self.latency.observe(latency)

        if success:
            self.cache_hits.labels(level=cache_level).inc()
        else:
            self.errors.labels(error_type='timeout').inc()

# Usage
metrics = MetricsCollector()

@app.get("/autocomplete")
def get_suggestions(q: str, user_id: str):
    start = time.time()

    try:
        # Check CDN cache
        if q in cdn_cache:
            results = cdn_cache[q]
            metrics.record_request(start, cache_level='cdn', success=True)
            return results

        # Check Redis cache
        if q in redis_cache:
            results = redis_cache[q]
            metrics.record_request(start, cache_level='redis', success=True)
            return results

        # Get from Trie
        results = autocomplete_service.get_suggestions(q, user_id)
        metrics.record_request(start, cache_level='trie', success=True)

        return results

    except TimeoutError:
        metrics.record_request(start, cache_level='none', success=False)
        return {"error": "timeout"}
```

**Capacity planning:**
```
Цель: Поддержать 1M QPS

Расчёт:
- 1M QPS = 1,000,000 запросов/сек
- Средний размер запроса: 20 байт
- Входящая пропускная способность: 1M × 20 байт = 20 MB/s

- Средний ответ: 1 KB (10 suggestions)
- Исходящая пропускная способность: 1M × 1 KB = 1 GB/s
- Требуется: 10 Gbps network links (с margin безопасности)

Пропускная способность сервера:
- 1 сервер обрабатывает ~100K QPS
- Нужно: 1M / 100K = 10 серверов на shard
- 5 shards × 10 серверов = 50 серверов
- С репликацией (3x): 150 серверов всего

Память:
- Trie: 10 GB
- Redis cache: 10 GB
- Per server: 20 GB RAM
- Всего: 150 × 20 GB = 3 TB RAM

Оценка стоимости (AWS):
- EC2 instances: ~$100/месяц каждый × 150 = $15K/месяц
- Networking: ~$50K/месяц (10 Gbps inter-region)
- RDS/Cassandra: ~$30K/месяц (analytics)
- Всего: ~$100K/месяц для инфраструктуры
```

**Типичные ошибки.**
- Не планировать заранее — suddenly overwhelmed при spike.
- Недооценивать networking — bandwidth bottleneck раньше чем вы ожидаете.
- Неправильная шардинг стратегия — одна shard overloaded.
- Отсутствие monitoring — проблемы обнаруживают пользователи.
- Игнорировать DR и failover — единая точка отказа.

**На интервью.**
- Покажи, как масштабировать от 10K до 1M QPS step-by-step.
- Упомяни key bottlenecks на каждом stage.
- Уточняющий вопрос: «Как обнаружить и зафиксить bottleneck в production?» — load testing, profiling, monitoring.
- Уточняющий вопрос: «Как справиться с uneven load distribution?» — consistent hashing, traffic shaping, request hedging.

---

### 11. Какие оптимизации и trade-offs существуют?

**Зачем спрашивают.** System design не существует в вакууме — всегда есть компромиссы. Интервьюер проверяет реалистичность и pragmatism подхода.

**Короткий ответ.** Латентность vs свежесть (batch updates compromis), память vs точность (pruning редких terms), точность vs скорость (approximate matching), стоимость vs производительность (hardware scaling vs algorithmic optimization).

**Детальный разбор.**

**Trade-off 1: Freshness vs Performance**
```
Batch updates (hourly):
+ Cheap: 1 Spark job/hour
+ Fast: precomputed top-K at each node
+ Reliable: batch processing well-understood
- Stale: suggestions lag 1 hour behind reality
- Missed trends: trending topics delayed

Real-time updates:
+ Fresh: instant trending detection
+ Adaptive: responds to user behavior
- Expensive: continuous processing
- Complex: harder to operate
- Risky: bugs affect live suggestions

RECOMMENDATION:
Use hybrid:
- Hourly batch for 99% of traffic (stable queries)
- Real-time for 1% (trending queries)
- Cost: 1 batch job + 1 streaming pipeline
- Freshness: base suggestions lag 1h, trending is real-time
```

**Trade-off 2: Precision vs Recall**
```
Сценарий: Пользователь ищет "python"
          У нас 10,000 запросов начинающихся с "python"

Опция 1: Вернуть только топ-10
+ Быстро: O(K) lookup где K=10
+ Память efficient: хранить только 10 на узле
- Может пропустить relевантные suggestions (recall = 10/10000)

Опция 2: Вернуть топ-100
+ Лучше recall: 100/10000
- Медленнее: нужно ранжировать и сортировать
- Память: в 10 раз больше

Опция 3: Вернуть все 10,000
+ Perfect recall
- Невозможно медленно для клиента
- Слишком много данных

РЕКОМЕНДАЦИЯ:
Вернуть топ-100 с сервера, клиент показывает топ-10
- Recall: 100/10000 = 1%
- Precision: если пользователь видит топ-10, в основном хорошо
- Trade: небольшое увеличение latency, лучший UX
```

**Trade-off 3: Accuracy vs Latency**
```
Сценарий: Ранжирование suggestions с personalization

Точное ранжирование (50ms):
- Multiple boosts: personal + trending + context
- Complex calculations
- Хорошее качество
- Risk: может timeout на медленной сети

Быстрое ранжирование (5ms):
- Simple heuristics: frequency only
- Нет personalization
- Быстро но менее relевантно
- Всегда успешно

РЕКОМЕНДАЦИЯ:
Tiered подход:
- Пытаемся точное ранжирование с 50ms timeout
- Если timeout, fallback к быстрому ранжированию
- Log медленные случаи для оптимизации
- Best of both worlds
```

**Trade-off 4: Consistency vs Availability**
```
Сценарий: Обновление suggestions в distributed shards

Strong consistency:
- Synchronous replication на все 3 replicas
- Ждём всех acknowledge
- Гарантии: все readers видят один и тот же data
- Risk: если 1 replica медленная, все blocked

Eventual consistency:
- Async replication
- Return immediately
- Risk: stale reads на несколько секунд
- Benefit: всегда доступна, быстрая

РЕКОМЕНДАЦИЯ:
Используем eventual consistency + batch versioning:
- Новый Trie строится каждый час
- Все shards получают новую версию в течение 1 минуты
- Worst case: пользователь видит suggestions на 1 час старее
- Acceptable: suggestions не меняются сильно час от часа
```

**Trade-off 5: Cost vs Quality**
```
Сценарий: Стратегия caching

CDN кэшировать все префиксы:
+ Perfect hit rate
+ Lowest latency
- Cost: $100K+/месяц (дорого)
- Overkill: много префиксов редко запрашивается

CDN кэшировать только топ-1000:
- Hit rate: ~80%
- Cost: $20K/месяц
- Trade: 20% запросов идут к origin

Только client-side cache:
- Hit rate: ~70% (много пользователей)
- Cost: $0
- Trade: more origin requests

RECOMMENDATION:
Layered caching:
- Client: 1-min cache (~70% hit)
- CDN: top-1000 prefixes (~20% hit of misses)
- Origin: all data
- Total cost: $20K/month
- Total hit rate: ~86%
- Compromise: balance cost and performance
```

**Comparative analysis таблица:**
```
┌────────────────────┬──────────┬─────────┬─────────┬──────────┐
│ Approach           │ Latency  │ Cost    │ Freshness│ Complexity
├────────────────────┼──────────┼─────────┼─────────┼──────────┤
│ Trie (in-memory)   │ <1ms     │ $$      │ 1h lag  │ Low      │
│ ElasticSearch      │ 50-100ms │ $$$     │ Fresh   │ Medium   │
│ Database (SQL)     │ 100-500ms│ $$      │ Fresh   │ Low      │
│ Hybrid             │ 10-50ms  │ $$$     │ 1h lag  │ High     │
└────────────────────┴──────────┴─────────┴─────────┴──────────┘

CHOOSE Trie IF:
- Need <50ms latency (critical)
- Have small dataset that fits memory
- Can tolerate 1-hour stale data
- Want simplicity

CHOOSE ElasticSearch IF:
- Need fuzzy matching, typo tolerance
- Large dataset, complex queries
- Real-time fresh data required
- Team familiar with ES

CHOOSE Hybrid IF:
- Best latency + freshness
- Have resources for complexity
- Large-scale system
```

**Типичные ошибки.**
- Преследовать "perfect" когда "good enough" работает.
- Не документировать trade-offs — team не понимает решений.
- Оптимизировать не того bottleneck — потрачена энергия на wrong problem.
- Забыть про operational cost — perfect архитектура которую нельзя запустить.

**На интервью.**
- Упомяни trade-offs явно: "We chose X over Y because...".
- Покажи, как измерить impact каждого trade-off.
- Уточняющий вопрос: «Как бы ты изменил дизайн если...» — freshness requirement doubled, users in 10 countries, budget tripled?
- Уточняющий вопрос: «Как A/B тестировать разные strategies?» — segment traffic, measure metrics, decide.

---

### 12. На интервью: как правильно отвечать и типичные вопросы-ловушки?

**Зачем спрашивают.** Последний вопрос нужен, чтобы интервьюер понял, как ты коммуникируешь и справляешься с неожиданными follow-ups.

**Короткий ответ.** Начни с high-level обзора, затем уходи в детали по запросу интервьюера. Упомяни trade-offs, explain reasoning, ask clarifying questions. Не паникуй при follow-up — покажи flexibility и thinking process.

**Детальный разбор.**

**Структура ответа (SNAKE):**
```
S — Scope: Уточни требования
N — Numbers: Capacity estimation
A — Architecture: High-level design
K — Key components: Глубокий анализ критических частей
E — Edge cases: Как обрабатывать failures

Пример:
Интервьюер: "Спроектируй autocomplete для Google search."

1. SCOPE (clarify)
   "10B запросов/день, 100ms latency, 99.99% uptime?
    Multi-language? Typo correction? Personalization?
    Я буду предполагать English, с trending topics, без typo handling."

2. NUMBERS (estimate)
   "10B запросов/день = 580K QPS, Peak 1.5M QPS
    Data: 5M unique terms, ~500MB в памяти
    Bandwidth: 1GB/s исходящих данных"

3. ARCHITECTURE (diagram)
   "Client -> CDN -> LB -> Autocomplete Service -> Trie/Redis"

4. KEY COMPONENTS
   "Trie для O(K) lookup, Redis для caching, Kafka для логирования"

5. EDGE CASES
   "Failures: replica failover, cache invalidation, backpressure"
```

**Типичные follow-up questions и как ответить:**

```
Follow-up 1: "Как бы ты обработал spike трафика в 10x?"

Плохой ответ:
"Um... добавить больше серверов?"

Хороший ответ:
"Сначала, проверю откуда spike:
1. Geographic (одна region перегружена)?
   → Route traffic к другим regions через GeoDNS
2. Prefix-specific (например, trending topic)?
   → Increase trending detection sensitivity
3. Bot traffic (spam)?
   → Rate limit by IP, CAPTCHAs

Short-term: scale horizontally (add instances)
Long-term: optimize bottleneck (caching, compression)

SLA: Keep p99 latency <150ms, error rate <0.5%"

Follow-up 2: "How would you test this system?"

Плохой ответ:
"Unit tests and integration tests?"

Хороший ответ:
"Three levels of testing:

1. Unit tests (fast)
   - Trie insertion/search
   - Levenshtein distance
   - Ranking algorithm

2. Integration tests (slow)
   - Full pipeline: Kafka -> Spark -> Trie -> Redis
   - Verify Trie consistency
   - Failover scenarios

3. Load testing (production-like)
   - Simulate 1.5M QPS
   - Measure p50/p99 latencies
   - Identify bottlenecks
   - Stress test: 3x load for 1 hour

4. Chaos engineering (failure modes)
   - Kill random servers
   - Network partitions
   - Slow Redis
   - Verify graceful degradation"

Follow-up 3: "What if Trie doesn't fit in memory?"

Плохой ответ:
"Just use a database?"

Хороший ответ:
"Если Trie превышает доступную память:

Опция 1: Compression
- PATRICIA tree (30% экономия)
- Прорежать редкие термины (<100/день)
- Результат: 70% -> 50% памяти

Опция 2: Sharding
- По первому символу (5 shards)
- Каждый shard: 10GB / 5 = 2GB
- Управляемо на современном оборудовании

Опция 3: Tiered storage
- Hot prefixes (топ 10K) в RAM
- Cold prefixes в Redis
- Very cold в Cassandra
- Lazy loading с LRU cache

Опция 4: External storage
- Elasticsearch для full search
- Trade: 50-100ms vs <1ms latency
- Gain: unlimited size, fuzzy matching

Я бы попробовал Опции 1 + 2 первыми (compression + sharding).
Если всё ещё не помещается, добавляю Опцию 3 (tiered storage)."

Follow-up 4: "How would you optimize for mobile users?"

Плохой ответ:
"Same as desktop?"

Хороший ответ:
"Mobile is different:
- Network latency higher (3G/4G)
- Bandwidth lower
- Battery sensitive
- Screen smaller (fewer suggestions visible)

Оптимизации:
1. Network
   - Выше debounce (200ms vs 100ms) чтобы уменьшить запросы
   - Compression (gzip responses)
   - Меньше suggestions (5 vs 10)
   - Prefetch next likely queries

2. Battery
   - Batch requests (coalesce с другими API calls)
   - Reduce frequency updates
   - Cache aggressively (5min vs 1min)

3. Storage
   - Compressed local cache (1MB vs 10MB)
   - SQLite вместо in-memory map
   - Lazy initialization

4. Metrics
   - Monitor mobile p99 latency отдельно
   - Set выше SLO (150ms vs 100ms) для mobile
   - A/B test разные settings"

Follow-up 5: "Проблемы безопасности?"

Плохой ответ:
"Никогда не думал об этом."

Хороший ответ:
"Несколько security vectors:

1. Abuse
   - Rate limit by IP (100 requests/second)
   - Rate limit by user (1000 requests/hour)
   - Pattern detection (unusual query sequences)

2. Privacy
   - Encrypt logs (PII в queries)
   - Anonymize user IDs в analytics
   - GDPR compliance (right to deletion)
   - Don't correlate with identity

3. Injection attacks
   - Sanitize queries (remove HTML/SQL)
   - Validate prefix length (max 100 chars)
   - Use prepared statements in analytics DB

4. DDoS
   - Anycast DDoS mitigation (Cloudflare)
   - Rate limiting at edge
   - Health checks, automatic failover

5. Data leakage
   - Sensitive queries shouldn't be logged
   - Medical, financial queries: filter at source
   - Whisper mode: don't save to history"
```

**Как справиться если не знаешь ответ:**

```
Сценарий: Interviewer спрашивает про Bloom filters

Плохой ответ:
"I don't know."

Хороший ответ:
"I'm not familiar with Bloom filters specifically, but
let me think out loud:

The name suggests it filters something?
Probably a probabilistic data structure for membership testing?
Benefits might be: memory-efficient, fast lookup, maybe false positives?

If used in autocomplete... maybe for fast rejection?
Like: 'is this prefix in the dictionary?'

Could you tell me more? I'm interested in learning."

→ Shows curiosity, problem-solving, not defensive
→ Interviewer often explains and continues discussion
→ Better than pretending to know
```

**Red flags (things NOT to do):**

```
❌ Don't over-engineer
   "I'll use machine learning for ranking"
   → Too complex, overkill

❌ Don't ignore the numbers
   "We'll just cache everything"
   → 10B queries/day, need math

❌ Don't flip-flop
   "Let's use Trie... actually Elasticsearch... no wait, Trie again"
   → Shows uncertainty, makes decision and explain it

❌ Don't ignore failure modes
   "Everything works perfectly, no downtime ever"
   → Naive, unrealistic

❌ Don't be overconfident
   "This will definitely work, 100% guaranteed"
   → Nothing is guaranteed, show you've thought of risks

❌ Don't ignore cost
   "Let's use a petabyte of memory"
   → Not realistic, show you think about constraints
```

**Green flags (things TO do):**

```
✅ Ask clarifying questions
   "Should I optimize for latency or cost?"
   "How fresh should suggestions be?"

✅ Do capacity estimation
   "10B queries = 580K QPS, data fits in ~500MB"

✅ Trade-offs discussion
   "I chose batch updates over real-time because..."

✅ Mention monitoring
   "We'll track p99 latency, cache hit rate, error rate"

✅ Discuss edge cases
   "What if one shard fails? Answer: failover to replica"

✅ Show flexibility
   "If requirements change, we could..."

✅ Mention operational aspects
   "This requires monitoring, on-call rotation, runbooks"
```

**Time management в интервью:**

```
45-minute system design interview:

0-5 min: Clarify requirements and scope
5-10 min: Capacity estimation (be quick!)
10-25 min: Architecture and design (deep dive)
25-40 min: Follow-ups and trade-offs (interactive)
40-45 min: Wrap up, ask questions

If you finish architecture early:
- Dig deeper into one component
- Don't rush, show depth
- Interviewer will follow up anyway

If interviewer asks follow-up you can't handle:
- Don't panic, think out loud
- "Good question, let me think..."
- Propose solution, explain reasoning
```

**Типичные ошибки.**
- Не уточнять требования — design не подходит для actual use case.
- Скакать между идеями — выглядит неподготовленно.
- Говорить без пауз — интервьюер не может вставить вопрос.
- Забыть про non-functional requirements — latency, availability, cost.
- Не слушать hints от интервьюера — он пытается помочь!

**На интервью.**
- Сначала speak with confidence, потом listen to feedback.
- Don't memorize answer — be ready to adapt.
- Show your thinking process, not just final answer.
- Ask clarifying questions (this shows maturity).
- Discuss trade-offs explicitly.

---

## См. также

- [System Design Overview](./README.md) — общий гайд по system design
- [Rate Limiter](./02-rate-limiter.md) — применение в API protection
- [URL Shortener](./04-url-shortener.md) — другая high-scale система
- [News Feed](./05-news-feed.md) — similar ranking & personalization challenges
- [Distributed Cache](./07-distributed-cache.md) — Redis & cache patterns
- [Trie Data Structure](../05-algorithms/06-trees-traversal.md) — algorithmic foundation

---

## Практика

1. **Базовая Trie** — реализуй Trie с insertion и search. Добавь precomputed top-K на каждом узле.

2. **Персонализация** — реализуй простую систему ranking с personal boost. Используй mock data.

3. **Distributed sharding** — спроектируй шардинг strategy для 5 shards. Напиши routing logic.

4. **Load testing** — используй Apache JMeter или wrk для load testing собственного сервера. Найди bottleneck.

5. **Latency optimization** — профилируй свой autocomplete сервер (CPU flame graphs, allocation profiling). Оптимизируй top 3 bottleneck'а.

6. **Failure handling** — спроектируй failover для replica loss. Implement health checks и automatic rerouting.

---

## Дополнительные материалы

- [Designing Data-Intensive Applications](https://dataintensive.info/) — классический подход к system design
- [System Design Interview](https://systemdesigninterview.com/) — структурированный гайд
- [Google Autocomplete Patents](https://patents.google.com/?q=autocomplete) — как Google это делает
- [Trie based search](https://en.wikipedia.org/wiki/Trie) — основная структура данных
- [Levenshtein Distance](https://en.wikipedia.org/wiki/Levenshtein_distance) — для typo handling
- [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing) — для sharding

---

← [05-news-feed](./05-news-feed.md) | [Трек System Design](./README.md) | [07-distributed-cache](./07-distributed-cache.md) →
