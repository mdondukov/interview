# 05 — News Feed / Timeline

Развёрнутые вопросы и ответы про проектирование систем новостных лент: стратегии раздачи постов, ранжирование, кэширование, пагинация, обработка знаменитостей. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [04-chat-messenger](./04-chat-messenger.md) · Следующая тема: [06-search-autocomplete](./06-search-autocomplete.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Fanout-on-write (Push)** — стратегия, при которой при публикации нового поста система сразу же записывает его в feed каждого follower'а автора. Это обеспечивает быстрое чтение ленты (O(1) операция), но медленную публикацию если у автора много followers. Push используется для обычных пользователей, так как чтение ленты выполняется чаще, чем её обновление.

**Fanout-on-read (Pull)** — альтернативная стратегия, при которой feed ленты собирается динамически при чтении пользователем. При чтении система запрашивает посты от всех follow'ов пользователя и объединяет их. Это обеспечивает быструю публикацию, но медленное чтение (может потребоваться много запросов). Pull используется для знаменитостей с миллионами followers.

**Timeline** — упорядоченный (обычно хронологический) список постов для конкретного пользователя, сформированный из постов авторов, за которыми он следит. Timeline — основное представление news feed в БД. Обычно хранится в отдельной таблице/коллекции и индексируется по user_id и timestamp для быстрого получения последних постов.

**Ranking** — алгоритм или набор правил, которые переупорядочивают посты в feed на основе релевантности для пользователя. Ranking определяет, в каком порядке пользователь видит посты: просто хронологически или приоритизируя более интересные. Хороший ranking алгоритм критичен для вовлечённости, но требует ML модели или эвристик.

**EdgeRank** — классическая формула ранжирования, разработанная Facebook в начале 2010х годов: Score = Affinity × Weight × Time Decay. Affinity измеряет близость пользователя к автору, Weight указывает на тип действия (коммент важнее, чем лайк), Time Decay экспоненциально уменьшает вес старых постов. EdgeRank показывает, как сбалансировать различные факторы.

**Affinity** — числовая мера взаимодействия пользователя с конкретным автором. Affinity увеличивается при лайках, комментах, шарах постов данного автора. Более высокая affinity означает, что посты этого автора должны показываться чаще. Affinity может быть рассчитана как экспоненциальное взвешивание последних действий (more recent = более важно).

**Time decay** — компонент ранжирования, который экспоненциально снижает релевантность старых постов. Например, пост от часа назад может иметь в 2 раза выше score, чем пост от недели назад. Time decay гарантирует, что новые, актуальные события выходят на первый план, даже если они менее популярны, чем старые вирусные посты.

**Engagement** — совокупность взаимодействий пользователей с постом: лайки, комментарии, репосты/шары. Engagement является ключевым сигналом для ранжирования, так как показывает, насколько интересен пост для аудитории. Посты с высоким engagement должны быть показаны большему числу людей благодаря алгоритму рекомендации.

**Celebrity threshold** — количество followers (обычно 10K-100K), выше которого система переключается с push на pull модель для fanout. Для знаменитостей push становится неэффективным (слишком много writes при публикации), поэтому лучше использовать pull (читатели собирают пост при запросе). Threshold зависит от нагрузки и архитектуры.

**Graph service** — отдельный микросервис, управляющий социальной сетью: хранит информацию о follows, followers, friendships между пользователями. Graph service отвечает на вопросы вроде "кого следит пользователь X?" и "кто следит за пользователем Y?". Обычно использует граф БД или кэш Redis для быстрого доступа, так как граф очень часто запрашивается.

---

## Вопросы и разборы

### 1. Как спроектировать систему news feed с нуля?

**Зачем спрашивают.** News feed — комплексная система, требующая баланса между скоростью чтения, скоростью публикации, масштабируемостью и консистентностью. Интервьюер проверяет умение работать с большими системами и делать компромиссы.

**Короткий ответ.** Раздели на сервисы: Post Service (публикация), Feed Service (генерация), Graph Service (социальная граница), Media Service (медиа). Используй гибридный подход fanout: для обычных пользователей — push на write, для знаменитостей — pull на read. Кэшируй feeds в Redis, посты в памяти, используй Cassandra для timeline.

**Детальный разбор.**

**Архитектура на высоком уровне:**
```
┌──────────────┐
│   Clients    │
│ (Web, Mobile)│
└──────┬───────┘
       │
┌──────▼────────────────────────────┐
│      API Gateway + LB             │
└──────┬───────┬──────┬──────┬──────┘
       │       │      │      │
    ┌──▼─┐  ┌─▼──┐ ┌─▼──┐ ┌─▼──┐
    │Post│  │Feed│ │Grph│ │Med │
    │Svc │  │Svc │ │Svc │ │Svc │
    └──┬─┘  └─┬──┘ └─┬──┘ └─┬──┘
       │      │      │      │
    ┌──▼──┐ ┌─▼──┐ ┌─▼──┐ ┌─▼──┐
    │Post │ │Feed│ │Grph│ │CDN │
    │ DB  │ │Cache│ │ DB │ │S3  │
    └─────┘ └────┘ └────┘ └────┘
```

**Компоненты системы:**

1. **Post Service**
   - Создание, обновление, удаление постов
   - Хранилище: PostgreSQL (metadata) + Cassandra (timeline)
   - Индексы: по author_id, created_at

2. **Feed Service**
   - Генерация персонализированной ленты
   - Применяет ранжирование, фильтрацию
   - Интегрирует с Cache (Redis) и Graph Service

3. **Graph Service**
   - Управление социальной сетью (follows)
   - Кэширует followers/followees
   - Выявляет знаменитостей (>10K followers)

4. **Fanout Service**
   - Асинхронная раздача постов на write (для обычных)
   - Message queue (Kafka): батчинг, retries
   - Может работать несколько часов для celebrities

**Capacity Planning:**
```
Входные данные:
- 500M DAU
- 200 followers avg per user
- 10% publish rate = 50M posts/day
- Feed refresh: 10x per user/day = 5B requests/day
- QPS peak: 150,000

Storage:
- Post metadata: 1KB
- 50M posts/day × 1KB = 50GB/day
- 5 лет: ~90TB (с компрессией ~20TB)
```

**Ключевые решения на архитектурном уровне:**
- **Fan-out strategy:** Гибридный подход (push для обычных, pull для celebrities)
- **Cache layers:** Feed cache (5 min TTL), Post cache (1 hour), Graph cache (realtime)
- **Storage:** Hot (Redis), Warm (PostgreSQL), Cold (S3 for media)
- **Consistency:** Eventual consistency (задержка в секунды приемлема)

**Пример.**

```python
# News Feed System Architecture (Python-like pseudocode)

class NewsFeedException(Exception):
    pass

class FeedSystem:
    def __init__(self):
        self.post_db = PostgreSQL()      # Основные посты
        self.timeline_db = Cassandra()   # Timeline для каждого
        self.cache = Redis()             # Feed cache
        self.graph_db = Neo4j()          # Social graph
        self.fanout_queue = Kafka()      # Async fanout
        self.media_cdn = S3()            # Media storage

    # Constants
    CELEBRITY_THRESHOLD = 10000
    FEED_CACHE_TTL = 300        # 5 minutes
    POST_CACHE_TTL = 3600       # 1 hour
    FEED_SIZE = 100             # Posts per page

    async def publish_post(self, author_id: str, content: str,
                          media_ids: list) -> str:
        """Публикация поста с фанаутом"""

        # 1. Сохранить пост в основную БД
        post_id = await self.post_db.insert({
            'author_id': author_id,
            'content': content,
            'media_ids': media_ids,
            'created_at': time.time(),
            'likes_count': 0
        })

        # 2. Получить количество followers
        follower_count = await self.graph_db.get_follower_count(author_id)

        # 3. Решить: push или pull
        if follower_count < self.CELEBRITY_THRESHOLD:
            # Обычный пользователь: фанаут на write (push)
            await self._fanout_to_followers(author_id, post_id)
        else:
            # Знаменитость: пометить как celebrity post
            await self.post_db.update(post_id, {'is_celebrity': True})

        return post_id

    async def _fanout_to_followers(self, author_id: str, post_id: str):
        """Push-модель: запись в feeds всех followers"""

        # Получить followers батчами
        followers = await self.graph_db.get_followers(author_id)

        for batch in chunks(followers, 1000):
            await self.fanout_queue.send({
                'type': 'fanout_post',
                'post_id': post_id,
                'author_id': author_id,
                'user_ids': batch,
                'timestamp': time.time()
            })

    async def get_feed(self, user_id: str, cursor: str = None,
                      limit: int = 20) -> dict:
        """Получить персонализированную ленту"""

        # 1. Попытаться получить из кэша
        cache_key = f"feed:{user_id}"
        cached_posts = await self.cache.zrevrange(
            cache_key, 0, limit - 1, withscores=True
        )

        if cached_posts:
            posts = await self._hydrate_posts(cached_posts)
            return self._format_feed_response(posts, cursor)

        # 2. Cache miss: генерировать ленту
        feed_posts = await self._generate_feed(user_id)

        # 3. Кэшировать результат
        await self._cache_feed(user_id, feed_posts)

        return self._format_feed_response(feed_posts, cursor)

    async def _generate_feed(self, user_id: str) -> list:
        """Генерация ленты: pre-computed + celebrity posts"""

        # 1. Get following list
        followees = await self.graph_db.get_followees(user_id)

        # 2. Разделить на обычных и знаменитостей
        regular = [f for f in followees
                   if not await self._is_celebrity(f)]
        celebrities = [f for f in followees
                       if await self._is_celebrity(f)]

        # 3. Получить посты из timeline для обычных
        timeline_posts = await self.timeline_db.query(
            user_id, limit=self.FEED_SIZE
        )

        # 4. Получить recent posts от знаменитостей
        celebrity_posts = await asyncio.gather(*[
            self.post_db.get_recent_posts(f, limit=5)
            for f in celebrities
        ])

        # 5. Объединить и ранжировать
        all_posts = timeline_posts + flatten(celebrity_posts)
        ranked = sorted(all_posts,
                       key=lambda p: self._calculate_score(p, user_id),
                       reverse=True)

        return ranked[:self.FEED_SIZE]

    async def _cache_feed(self, user_id: str, posts: list):
        """Кэшировать ленту в Redis"""
        cache_key = f"feed:{user_id}"

        for score, post in enumerate(posts):
            await self.cache.zadd(
                cache_key,
                {post['id']: score},
                ex=self.FEED_CACHE_TTL
            )

    def _calculate_score(self, post: dict, user_id: str) -> float:
        """Ranking formula (EdgeRank-like)"""

        # Time decay: новые посты выше
        age_hours = (time.time() - post['created_at']) / 3600
        time_score = 1 / (1 + age_hours * 0.1)

        # Affinity: как часто пользователь взаимодействует с автором
        affinity = self._get_affinity_score(user_id, post['author_id'])

        # Engagement: likes, comments, shares
        engagement = (
            post.get('likes_count', 0) * 1.0 +
            post.get('comments_count', 0) * 2.0 +
            post.get('shares_count', 0) * 3.0
        )

        # Content type weight
        content_type = post.get('type', 'text')
        content_weight = {
            'video': 1.5,
            'image': 1.2,
            'text': 1.0
        }.get(content_type, 1.0)

        final_score = (time_score * affinity *
                      (1 + math.log(1 + engagement)) * content_weight)

        return final_score

    async def _is_celebrity(self, user_id: str) -> bool:
        """Проверить, знаменитость ли пользователь"""
        count = await self.graph_db.get_follower_count(user_id)
        return count >= self.CELEBRITY_THRESHOLD
```

**Типичные ошибки.**
- Пытаться применить push для celebrities — затор в fanout queue
- Не кэшировать feeds — каждый read = N database queries
- Забыть про eventual consistency — требовать strong consistency будет узкое место
- Не отделить hot path (feed read) от slow path (fanout write)

**На интервью.**
- Нарисуй архитектуру сверху вниз: API Gateway → Services → Databases
- Объясни гибридный подход: почему celebrities требуют pull
- Уточняющий вопрос: «Как обработать, если пользователь становится celebrity?» — перемиграция feed старых постов в timeline
- Уточняющий вопрос: «Как реализовать soft delete постов?» — mark as deleted, скрывать из feeds

---

### 2. В чём разница между fanout-on-write и fanout-on-read?

**Зачем спрашивают.** Две противоположные стратегии с разными компромиссами. Интервьюер проверяет понимание trade-offs и когда что применимо.

**Короткий ответ.** Fanout-on-write (push): при публикации записать пост в feed каждого follower. Быстрое чтение, медленная публикация. Fanout-on-read (pull): при чтении собрать посты от всех followees. Быстрая публикация, медленное чтение. Гибридный подход: push для обычных, pull для celebrities.

**Детальный разбор.**

**Fanout-on-Write (Push Model):**
```
Post публикуется → Fanout Service читает followers →
Записывает в feed каждого → Redis/Cassandra обновлены

Timeline:
User A posts → get followers (1M) →
  batch 1000 → queue message (1)
  batch 1000 → queue message (2)
  ...
  batch 1000 → queue message (1000)

Worker processes messages asynchronously
User B reads feed → Redis (instant hit)
```

**Плюсы:**
- ✅ Чтение feed очень быстро: O(1) в Redis, pre-computed
- ✅ Простая логика чтения
- ✅ Consistent ranking (вычисляется один раз при публикации)

**Минусы:**
- ❌ Публикация медленная для celebrities (10M followers = 10K messages)
- ❌ Хранилище: неактивные юзеры занимают место в timeline
- ❌ Wasted writes: если пост удален, нужно чистить все timelines

**Fanout-on-Read (Pull Model):**
```
User publishes post → Save to Posts DB (done in 100ms)

User reads feed → Get followees (50) →
  Fetch recent posts from each (50 parallel queries) →
  Merge & rank → Return to client

Timeline:
User A posts → PostgreSQL (1 write)
User B reads feed →
  get_followees() → 50 queries
  get_posts(followee_1) → parallel
  get_posts(followee_2) → parallel
  ...
  Merge + rank in memory
```

**Плюсы:**
- ✅ Публикация instant (один write в DB)
- ✅ Нет wasted storage
- ✅ Automatic staleness: deleted posts исчезают сразу
- ✅ Гибкость: ранжирование может быть персональным

**Минусы:**
- ❌ Чтение медленное: N queries (N = number of followees)
- ❌ High load on posts DB при viral posts
- ❌ Latency: 50 followees × 50ms per query = 2.5s timeout

**Сравнение:**

| Аспект | Push | Pull | Hybrid |
|--------|------|------|--------|
| **Write latency** | High (celebrities) | Low | Medium |
| **Read latency** | Low | High | Low |
| **Storage** | High | Low | Medium |
| **Consistency** | Eventually consistent | Always fresh | Both |
| **Best for** | Regular users | Small followers | Mixed |

**Пример.**

```python
# Push Model
async def publish_post_push(author_id: str, content: str):
    post_id = await posts_db.insert({
        'author_id': author_id,
        'content': content,
        'created_at': time.time()
    })

    # Get followers in batches
    followers = await graph_db.get_followers(author_id)

    # Queue fanout jobs
    for batch in chunks(followers, 1000):
        await fanout_queue.send({
            'post_id': post_id,
            'follower_ids': batch
        })

    # Fanout worker (async, can take hours for celebrities)
    # async def fanout_worker(job):
    #     for user_id in job['follower_ids']:
    #         await cassandra.add_to_timeline(
    #             user_id,
    #             job['post_id'],
    #             timestamp=time.time()
    #         )

async def get_feed_push(user_id: str, limit: int = 20):
    # Cache hit: return immediately
    cached = await redis.zrevrange(f"feed:{user_id}", 0, limit-1)
    if cached:
        return await _hydrate_posts(cached)

    # Cache miss: re-generate (slow path)
    posts = await _generate_feed_push(user_id)
    return posts[:limit]


# Pull Model
async def publish_post_pull(author_id: str, content: str):
    post_id = await posts_db.insert({
        'author_id': author_id,
        'content': content,
        'created_at': time.time()
    })

    # Done! No fanout needed
    return post_id

async def get_feed_pull(user_id: str, limit: int = 20):
    # Get all followees
    followees = await graph_db.get_followees(user_id)

    # Fetch recent posts from each (parallel)
    posts_by_followee = await asyncio.gather(*[
        posts_db.get_recent_by_author(f, limit=10)
        for f in followees
    ])

    # Flatten and merge
    all_posts = []
    for posts in posts_by_followee:
        all_posts.extend(posts)

    # Rank by score
    ranked = sorted(
        all_posts,
        key=lambda p: calculate_score(p, user_id),
        reverse=True
    )

    return ranked[:limit]


# Hybrid Model (Best of both)
CELEBRITY_THRESHOLD = 10000

async def publish_post_hybrid(author_id: str, content: str):
    post_id = await posts_db.insert({
        'author_id': author_id,
        'content': content,
        'created_at': time.time()
    })

    follower_count = await graph_db.get_follower_count(author_id)

    if follower_count < CELEBRITY_THRESHOLD:
        # Push for regular users
        await fanout_to_followers(author_id, post_id)
    else:
        # Mark as celebrity post for pull at read time
        await posts_db.update(post_id, {'is_celebrity': True})

    return post_id

async def get_feed_hybrid(user_id: str, limit: int = 20):
    # 1. Get pre-computed feed (push results)
    cache_key = f"feed:{user_id}"
    regular_posts = await redis.zrevrange(cache_key, 0, 100)

    # 2. Get celebrity followees
    followees = await graph_db.get_followees(user_id)
    celebrities = await asyncio.gather(*[
        graph_db.is_celebrity(f) for f in followees
    ])
    celebrity_followees = [f for f, is_celeb in zip(followees, celebrities)
                          if is_celeb]

    # 3. Pull celebrity posts
    celebrity_posts = await asyncio.gather(*[
        posts_db.get_recent_by_author(f, limit=5)
        for f in celebrity_followees
    ])

    # 4. Merge
    all_posts = regular_posts + flatten(celebrity_posts)

    # 5. Rank and return
    ranked = sorted(
        all_posts,
        key=lambda p: calculate_score(p, user_id),
        reverse=True
    )

    return ranked[:limit]
```

**Асинхронная обработка в push-модели:**
```
Publisher          Fanout Queue       Worker Pool        Timeline DB
    │                   │                  │                  │
    ├──publish──────────┤                  │                  │
    │                   ├──message 1───────┤                  │
    │ (return 100ms)    │                  ├──add_timeline──→ │
    │                   ├──message 2───────┤                  │
    │                   │                  ├──add_timeline──→ │
    │                   ├──message 3───────┤                  │
    │                   │                  ├──add_timeline──→ │
    │                   ├──message 4───────┤                  │
    (возможно часы      │                  ├──add_timeline──→ │
     для 10M followers) │                  │                  │
```

**Типичные ошибки.**
- Использовать push для celebrities — сотни тысяч фанаут-задач одновременно
- Использовать pull для всех — 1M followers = 1M DB queries при каждом feed read
- Не батчить fanout messages — перегруженная очередь
- Забыть про timeout в pull-модели — пользователь ждёт, пока загрузятся посты от всех

**На интервью.**
- Нарисуй timeline обеих моделей: что происходит при publish, что при read
- Объясни, почему hybrid лучше: push для большинства, pull для редких случаев
- Уточняющий вопрос: «Как определить threshold для celebrity?» — эмпирически: 10K—100K в зависимости от нагрузки
- Уточняющий вопрос: «Как переделать обычного пользователя в celebrity?» — переписать старые посты из feed в posts DB, включить pull-режим

---

### 3. Как реализовать ranking алгоритм для ленты?

**Зачем спрашивают.** Ranking отличает качественную ленту от случайной последовательности. Интервьюер проверяет понимание машинного обучения, сигналов важности, балансирования факторов.

**Короткий ответ.** Используй многофакторную формулу (как EdgeRank у Facebook): time decay (новые посты выше), affinity score (история взаимодействия), engagement signals (likes, comments, shares), content type weight (видео > картинки > текст). Комбинируй факторы: `score = time_decay × affinity × (1 + log(engagement)) × content_weight`.

**Детальный разбор.**

**Компоненты ranking formula:**

```
Score(post, user) = TimeDecay(post)
                   × Affinity(user, author)
                   × (1 + log(1 + Engagement(post)))
                   × ContentTypeWeight(post)

Каждый компонент [0..1] или [0..∞]
```

**1. Time Decay (Freshness):**
```
Новые посты должны быть выше, но не слишком агрессивно.
Формула: 1 / (1 + age_hours × decay_rate)

age_hours = 0 (новый) → score = 1.0
age_hours = 1 → score = 0.909 (decay_rate=0.1)
age_hours = 10 → score = 0.5
age_hours = 100 → score = 0.09

Graph:
  1.0  │●
       │ ●
  0.8  │  ●
       │   ●
  0.6  │    ●
       │     ●
  0.4  │      ●
       │       ●●
  0.2  │         ●●●
       │            ●●●●
  0.0  └────────────────────► age (hours)
```

**2. Affinity Score (User-Author Interaction):**
```
Как часто пользователь взаимодействует с автором.
Измеряется через:
- Количество лайков на постах этого автора
- Количество комментариев
- Количество шеров
- Время проведённое на постах

Формула: (likes + 2×comments + 3×shares) / total_user_actions

Affinity ≈ 0.1: автор не знаком
Affinity ≈ 0.5: знакомый автор
Affinity ≈ 0.9: близкий друг
```

**3. Engagement Signal:**
```
Общий engagement: сколько людей взаимодействовало с постом.
Формула: likes + 2×comments + 3×shares

Логарифмический scale: log(1 + engagement)
Причина: 1000 лайков только в 2× "лучше" чем 100 лайков

engagement = 0 → log(1) = 0
engagement = 100 → log(101) = 4.6
engagement = 1000 → log(1001) = 6.9
engagement = 10000 → log(10001) = 9.2
```

**4. Content Type Weight:**
```
Разные типы контента имеют разную ценность.

видео (1.5) > картинка (1.2) > ссылка (1.1) > текст (1.0)

Причины:
- Видео требует больше engagement для просмотра
- Картинки легче расшариваются
- Текстовые посты самые простые
```

**Полный пример формулы:**

```python
def calculate_ranking_score(post: dict, user_id: str,
                           user_affinity_cache: dict) -> float:
    """
    EdgeRank-like ranking formula for news feed
    """

    # 1. TIME DECAY
    age_hours = (time.time() - post['created_at']) / 3600
    time_decay = 1.0 / (1.0 + age_hours * 0.1)

    # 2. AFFINITY (from cache or compute)
    author_id = post['author_id']
    if author_id in user_affinity_cache:
        affinity = user_affinity_cache[author_id]
    else:
        # Query user's interaction history with author
        interactions = get_user_interactions(user_id, author_id)
        affinity = (
            interactions['likes'] * 1.0 +
            interactions['comments'] * 2.0 +
            interactions['shares'] * 3.0
        ) / max(1, get_user_total_interactions(user_id))
        affinity = min(1.0, affinity)  # Cap at 1.0

    # 3. ENGAGEMENT SIGNAL
    engagement_raw = (
        post.get('likes_count', 0) * 1.0 +
        post.get('comments_count', 0) * 2.0 +
        post.get('shares_count', 0) * 3.0
    )
    engagement_score = math.log(1.0 + engagement_raw)

    # 4. CONTENT TYPE WEIGHT
    content_type = post.get('type', 'text')
    content_weights = {
        'video': 1.5,
        'image': 1.2,
        'link': 1.1,
        'text': 1.0,
    }
    content_weight = content_weights.get(content_type, 1.0)

    # 5. FINAL SCORE
    final_score = (
        time_decay *
        affinity *
        (1.0 + engagement_score) *
        content_weight
    )

    return final_score

# Пример вычислений:
post_example = {
    'post_id': 'post_123',
    'author_id': 'user_456',
    'content': 'Check out my video!',
    'type': 'video',
    'created_at': time.time() - 2*3600,  # 2 hours ago
    'likes_count': 150,
    'comments_count': 30,
    'shares_count': 15
}

user_id = 'user_789'
user_affinity = {'user_456': 0.6}  # Close friend

# Calculation:
# time_decay = 1 / (1 + 2 * 0.1) = 1 / 1.2 = 0.833
# affinity = 0.6
# engagement = 150*1 + 30*2 + 15*3 = 150 + 60 + 45 = 255
# engagement_score = log(1 + 255) = log(256) = 5.545
# content_weight = 1.5 (video)
# final = 0.833 * 0.6 * (1 + 5.545) * 1.5 = 0.833 * 0.6 * 6.545 * 1.5 ≈ 4.89

score = calculate_ranking_score(post_example, user_id, user_affinity)
print(f"Ranking score: {score:.2f}")
```

**Оптимизация вычислений:**

```python
# Проблема: вычислять score для 100 постов × 500M пользователей = слишком дорого

# Решение 1: Кэшировать affinity scores
class AffiniityCache:
    def __init__(self, ttl_seconds=3600):
        self.cache = {}  # {user_id: {author_id: score}}
        self.ttl = ttl_seconds

    def get(self, user_id: str, author_id: str) -> float:
        key = f"{user_id}:{author_id}"
        if key in self.cache:
            return self.cache[key]
        # Fallback to DB query (and cache result)
        return 0.1  # default low affinity

# Решение 2: Вычислять score один раз, кэшировать
# При push-fanout: вычислить score для каждого пользователя сразу
# Сохранить score в Redis вместе с постом
# При pull: на лету, но с кэшированием affinity

# Решение 3: Используй ML модель
# Собери данные: (features, clicked?)
# Обучи GBDT или Neural Network
# Deploy как inference сервис (ms-latency)

class MLRanker:
    def __init__(self, model_path: str):
        self.model = load_model(model_path)  # TFLite or ONNX

    def score(self, post: dict, user_context: dict) -> float:
        features = self._extract_features(post, user_context)
        return self.model.predict(features)[0]

    def _extract_features(self, post: dict, user_context: dict) -> list:
        return [
            post.get('likes_count', 0),
            post.get('comments_count', 0),
            post.get('shares_count', 0),
            post.get('created_at', 0),
            user_context.get('affinity', 0),
            user_context.get('is_close_friend', 0),
            post.get('type_embedding', [0]*256),
        ]
```

**Типичные ошибки.**
- Использовать только recency (newest first) — скучно, теряются хорошие посты
- Использовать только engagement — горячие посты затмевают остальное
- Не нормализовать компоненты — один компонент может доминировать
- Не кэшировать affinity — пересчитывать его для каждого read

**На интервью.**
- Объясни каждый компонент формулы и почему он нужен
- Упомяни time decay логику: старые посты не должны исчезать мгновенно
- Уточняющий вопрос: «Как обнаружить и изолировать spam?» — отдельный classifier, низкий affinity score
- Уточняющий вопрос: «Как персонализировать ranking?» — ML модель, обучена на click data пользователя

---

### 4. Как кэшировать news feed?

**Зачем спрашивают.** Кэширование — основной инструмент масштабирования. Интервьюер проверяет понимание cache invalidation, TTL, multi-layer caching.

**Короткий ответ.** Используй 3-уровневый кэш: L1 — pre-computed feed в Redis (5 min TTL), L2 — post details в Redis/memcached (1 hour), L3 — social graph в Redis (realtime update). Инвалидируй при новом посте: удалить старую ленту, добавить новый пост в начало. Для celebrities используй immutable кэш без инвалидации.

**Детальный разбор.**

**Cache Layers для News Feed:**

```
┌─────────────────────────────────────────────────┐
│  Layer 1: Feed Cache (Redis)                    │
│  Key: feed:{user_id}                            │
│  Value: Sorted Set [{score}→{post_id}...]       │
│  TTL: 5 minutes                                 │
│  Size: 100-200 posts per user                   │
│  Hit Rate: ~80% (active users)                  │
└─────────────────────────────────────────────────┘
         │
         │ cache miss
         ▼
┌─────────────────────────────────────────────────┐
│  Layer 2: Post Cache (Redis/Memcached)          │
│  Key: post:{post_id}                            │
│  Value: {author, content, media, counts...}     │
│  TTL: 1 hour                                    │
│  Hit Rate: ~90%                                 │
└─────────────────────────────────────────────────┘
         │
         │ cache miss
         ▼
┌─────────────────────────────────────────────────┐
│  Layer 3: Post DB (PostgreSQL)                  │
│  Query: SELECT * FROM posts WHERE id = ?        │
│  Latency: 10-50ms                               │
└─────────────────────────────────────────────────┘

Отдельно:
┌─────────────────────────────────────────────────┐
│  Graph Cache (Redis)                            │
│  Key: user:{user_id}:followers                  │
│  Value: List of follower IDs                    │
│  TTL: Real-time updates via events              │
└─────────────────────────────────────────────────┘
```

**Layer 1: Feed Cache в Redis:**

```python
import redis
import json

class FeedCache:
    def __init__(self, redis_client, ttl=300):
        self.redis = redis_client
        self.ttl = ttl

    def cache_key(self, user_id: str) -> str:
        return f"feed:{user_id}"

    def get_feed(self, user_id: str, limit: int = 20) -> list:
        """Получить закэшированную ленту"""
        key = self.cache_key(user_id)

        # ZREVRANGE: sorted set в обратном порядке (новые первыми)
        # limit=-1: до конца (или до первых 100)
        post_ids = self.redis.zrevrange(key, 0, limit - 1)

        return post_ids  # List of post_ids

    def add_post_to_feed(self, user_id: str, post_id: str,
                        score: float, timestamp: float):
        """Добавить пост в ленту пользователя"""
        key = self.cache_key(user_id)

        # ZADD: добавить в sorted set с score (timestamp)
        self.redis.zadd(key, {post_id: score})

        # Установить TTL
        self.redis.expire(key, self.ttl)

        # Обрезать старые посты (keep top 200)
        self.redis.zremrangebyrank(key, 0, -201)

    def invalidate_feed(self, user_id: str):
        """Инвалидировать ленту (force refresh)"""
        key = self.cache_key(user_id)
        self.redis.delete(key)

# Использование:
redis_client = redis.Redis(host='localhost', port=6379)
feed_cache = FeedCache(redis_client, ttl=300)

# При публикации поста:
feed_cache.add_post_to_feed('user_789', 'post_123', score=4.89,
                            timestamp=time.time())

# При чтении ленты:
cached_posts = feed_cache.get_feed('user_789', limit=20)
```

**Layer 2: Post Details Cache:**

```python
class PostCache:
    def __init__(self, redis_client, ttl=3600):
        self.redis = redis_client
        self.ttl = ttl

    def cache_key(self, post_id: str) -> str:
        return f"post:{post_id}"

    def get_post(self, post_id: str) -> dict:
        """Получить пост из кэша или DB"""
        key = self.cache_key(post_id)

        # HGETALL: get all fields of hash
        cached = self.redis.hgetall(key)

        if cached:
            # Deserialize from bytes
            return {k.decode(): v.decode() for k, v in cached.items()}

        return None  # Cache miss

    def cache_post(self, post_id: str, post_data: dict):
        """Закэшировать пост"""
        key = self.cache_key(post_id)

        # HSET: store post as hash
        self.redis.hset(key, mapping=post_data)
        self.redis.expire(key, self.ttl)

    def invalidate_post(self, post_id: str):
        """Инвалидировать пост при изменении"""
        key = self.cache_key(post_id)
        self.redis.delete(key)

# Использование:
post_cache = PostCache(redis_client, ttl=3600)

# При публикации:
post_data = {
    'post_id': 'post_123',
    'author_id': 'user_456',
    'content': 'Hello world',
    'likes_count': '0',
    'comments_count': '0',
    'created_at': str(int(time.time()))
}
post_cache.cache_post('post_123', post_data)

# При чтении:
post = post_cache.get_post('post_123')
if not post:
    post = posts_db.query_one('post_123')
    post_cache.cache_post('post_123', post)
```

**Layer 3: Social Graph Cache:**

```python
class GraphCache:
    def __init__(self, redis_client):
        self.redis = redis_client

    def get_followers(self, user_id: str) -> list:
        """Получить followers из кэша"""
        key = f"followers:{user_id}"
        followers = self.redis.smembers(key)  # Set
        return [f.decode() for f in followers]

    def add_follower(self, user_id: str, follower_id: str):
        """Добавить follower (при follow)"""
        key = f"followers:{user_id}"
        self.redis.sadd(key, follower_id)
        # No TTL: обновляется через events

    def remove_follower(self, user_id: str, follower_id: str):
        """Удалить follower (при unfollow)"""
        key = f"followers:{user_id}"
        self.redis.srem(key, follower_id)

    def get_follower_count(self, user_id: str) -> int:
        """Получить количество followers (cached)"""
        key = f"user:{user_id}:followers_count"
        count = self.redis.get(key)
        return int(count) if count else 0

    def set_follower_count(self, user_id: str, count: int):
        """Обновить количество followers"""
        key = f"user:{user_id}:followers_count"
        self.redis.set(key, count, ex=3600)  # 1 hour TTL

# Использование:
graph_cache = GraphCache(redis_client)

# Event: user A follows user B
graph_cache.add_follower('user_B', 'user_A')

# Read: get feed for user A
followers = graph_cache.get_followers('user_B')
```

**Cache Invalidation Strategy:**

```
Проблема: Cache invalidation — один из двух самых сложных вещей.

Стратегии:
1. TTL: expire after N seconds (simplest, eventual consistency)
2. Event-based: publish event when data changes
3. Write-through: update cache immediately
4. Write-behind: queue updates, batch them
```

```python
class CacheInvalidationStrategy:
    """Cache invalidation при публикации нового поста"""

    async def on_post_published(self, author_id: str, post_id: str,
                               followers: list):
        """Вызывается когда пост опубликован"""

        # 1. Добавить пост в ленты всех followers (push model)
        for batch in chunks(followers, 1000):
            for user_id in batch:
                # Добавить в feed cache
                feed_cache.add_post_to_feed(
                    user_id,
                    post_id,
                    score=time.time()
                )

        # 2. Кэшировать сам пост
        post = await posts_db.query_one(post_id)
        post_cache.cache_post(post_id, post)

        # 3. Publish event в message queue
        await event_bus.publish({
            'event': 'post_published',
            'post_id': post_id,
            'author_id': author_id,
            'timestamp': time.time()
        })

    async def on_post_liked(self, post_id: str, user_id: str):
        """Вызывается когда пост лайкнули"""

        # 1. Обновить лайки в DB
        await posts_db.increment_likes(post_id)

        # 2. Инвалидировать кэш поста (так как лайки изменились)
        post_cache.invalidate_post(post_id)

        # 3. Publish event
        await event_bus.publish({
            'event': 'post_liked',
            'post_id': post_id,
            'user_id': user_id
        })

    async def on_user_followed(self, follower_id: str, followee_id: str):
        """Вызывается когда пользователь подписался"""

        # 1. Обновить followers в DB
        await graph_db.add_follow(follower_id, followee_id)

        # 2. Обновить followers cache
        graph_cache.add_follower(followee_id, follower_id)

        # 3. Инвалидировать ленту follower'а (добавились новые посты)
        feed_cache.invalidate_feed(follower_id)

        # 4. Publish event
        await event_bus.publish({
            'event': 'user_followed',
            'follower_id': follower_id,
            'followee_id': followee_id
        })
```

**Cache Warming:**

```python
class CacheWarmer:
    """Pre-fill кэш для активных пользователей"""

    async def warm_cache_for_active_users(self):
        """Запускается периодически (каждые 5 минут)"""

        # 1. Get top 1000 active users (by daily logins)
        active_users = await analytics.get_active_users(limit=1000)

        for user_id in active_users:
            # 2. Generate feed for each
            feed = await feed_service.generate_feed(user_id)

            # 3. Populate cache
            feed_cache.invalidate_feed(user_id)  # Clear old
            for post in feed:
                feed_cache.add_post_to_feed(
                    user_id,
                    post['id'],
                    score=post['ranking_score'],
                    timestamp=post['created_at']
                )

    async def warm_post_cache_for_hot_posts(self):
        """Кэшировать горячие посты (trending)"""

        # 1. Get trending posts (by recent engagement)
        trending = await posts_db.get_trending_posts(limit=100)

        for post_id in trending:
            # 2. Get full post data
            post = await posts_db.query_one(post_id)

            # 3. Extend TTL for hot posts
            post_cache.cache_post(post_id, post)
            redis_client.expire(f"post:{post_id}", 3600 * 6)  # 6 hours
```

**Типичные ошибки.**
- Кэшировать без TTL — проблемы когда пост обновляется
- Слишком короткий TTL (< 5 мин) — часто пересчитываешь, нет выгоды
- Слишком длинный TTL (> 1 часа) — старая информация, плохая UX
- Не инвалидировать при write — кэш не совпадает с DB
- Кэшировать для неактивных пользователей — трата памяти

**На интервью.**
- Объясни 3-layer caching: feed, posts, graph
- Почему 5 мин TTL для feed cache? — баланс между свежестью и производительностью
- Уточняющий вопрос: «Что если feed cache теряется?» — graceful degradation: пересчитать на лету, вернуть в 500ms
- Уточняющий вопрос: «Как кэшировать для celebrities?» — не кэшировать их посты, pull на read

---

### 5. Как реализовать пагинацию ленты?

**Зачем спрашивают.** Пагинация нужна для infinite scroll и эффективного использования памяти. Интервьюер проверяет понимание cursor-based pagination, вместо offset.

**Короткий ответ.** Используй cursor-based pagination вместо offset. Cursor — это base64-кодированный JSON с post_id и timestamp. При следующем запросе fetch posts с timestamp меньше, чем в cursor. Это O(1) в базе (по индексу), в отличие от offset который O(N).

**Детальный разбор.**

**Offset-based pagination (неправильный способ):**

```
Request 1: GET /feed?limit=20&offset=0
→ SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 0
→ Skip 0, return 20

Request 2: GET /feed?limit=20&offset=20
→ SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 20
→ Skip 20, return 20

Request 3: GET /feed?limit=20&offset=40
→ SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 40
→ Skip 40, return 20

Проблемы:
1. OFFSET N slow при большом N (O(N) scan)
2. Race condition: если пост добавлен в начало, он может дублироваться
3. Может пропустить элементы если они удаляются
```

**Cursor-based pagination (правильный способ):**

```
Request 1: GET /feed?limit=20
→ SELECT * FROM posts ORDER BY created_at DESC LIMIT 20
→ return 20 posts, next_cursor = encode(last_post_id, last_timestamp)

Request 2: GET /feed?limit=20&cursor=next_cursor
→ Decode cursor: {post_id: "post_123", timestamp: 1234567890}
→ SELECT * FROM posts
  WHERE created_at < 1234567890
  ORDER BY created_at DESC
  LIMIT 20
→ return 20 posts, next_cursor = ...

Плюсы:
1. O(1) lookup (по индексу created_at, post_id)
2. Stable even if posts added/removed
3. No duplicates or gaps
```

**Cursor Implementation:**

```python
import base64
import json

class PaginationCursor:
    """Encode/decode cursor для pagination"""

    @staticmethod
    def encode(post_id: str, timestamp: float) -> str:
        """Encode cursor to base64"""
        data = {
            'post_id': post_id,
            'timestamp': timestamp
        }
        json_str = json.dumps(data)
        return base64.b64encode(json_str.encode()).decode()

    @staticmethod
    def decode(cursor: str) -> dict:
        """Decode cursor from base64"""
        json_str = base64.b64decode(cursor).decode()
        return json.loads(json_str)

# Example
cursor = PaginationCursor.encode('post_123', 1234567890.5)
# cursor = "eyJwb3N0X2lkIjogInBvc3RfMTIzIiwgInRpbWVzdGFtcCI6IDEyMzQ1Njc4OTAuNX0="

decoded = PaginationCursor.decode(cursor)
# decoded = {'post_id': 'post_123', 'timestamp': 1234567890.5}
```

**Feed Pagination Service:**

```python
class FeedPaginationService:
    def __init__(self, feed_cache, posts_db, limit=20):
        self.feed_cache = feed_cache
        self.posts_db = posts_db
        self.limit = limit

    async def get_feed_page(self, user_id: str, cursor: str = None) -> dict:
        """Get feed page with pagination"""

        # 1. Get post IDs from cache (sorted by score)
        if cursor:
            cursor_data = PaginationCursor.decode(cursor)
            post_ids = self.feed_cache.get_feed_after(
                user_id,
                cursor_data['timestamp']
            )
        else:
            post_ids = self.feed_cache.get_feed(user_id, limit=self.limit)

        # 2. Hydrate with full post data
        posts = []
        for post_id in post_ids:
            post = await self.posts_db.get_post(post_id)
            if post:
                posts.append(post)

        # 3. Generate next cursor
        next_cursor = None
        if len(posts) == self.limit:
            last_post = posts[-1]
            next_cursor = PaginationCursor.encode(
                last_post['id'],
                last_post['created_at']
            )

        return {
            'posts': posts,
            'next_cursor': next_cursor,
            'has_more': next_cursor is not None
        }

    async def prefetch_next_page(self, user_id: str, cursor: str):
        """Prefetch next page (mobile optimization)"""

        if not cursor:
            return

        # Get next page in background, cache it
        next_page = await self.get_feed_page(user_id, cursor)

        # Store in temporary cache with short TTL (1 minute)
        await self._cache_prefetch(user_id, cursor, next_page)
```

**Redis optimizations для pagination:**

```python
class FeedCacheOptimized:
    def __init__(self, redis_client):
        self.redis = redis_client

    def get_feed(self, user_id: str, limit: int = 20) -> list:
        """Get first page from sorted set"""
        key = f"feed:{user_id}"
        # ZREVRANGE: descending order (highest score first)
        return self.redis.zrevrange(key, 0, limit - 1)

    def get_feed_after(self, user_id: str, max_timestamp: float) -> list:
        """Get posts older than cursor"""
        key = f"feed:{user_id}"
        limit = 20

        # ZREVRANGEBYSCORE: range query in sorted set
        # max=max_timestamp: scores less than this (older posts)
        # min=0: from 0 upwards
        # Returns newest first
        posts = self.redis.zrevrangebyscore(
            key,
            max=max_timestamp,
            min=0,
            start=0,
            num=limit
        )
        return posts

# API usage
GET /api/v1/feed
Response:
{
    "posts": [
        {
            "id": "post_123",
            "author": {...},
            "content": "...",
            "created_at": 1234567890,
            "likes_count": 150
        },
        ...
    ],
    "next_cursor": "eyJwb3N0X2lkIjogInBvc3RfNDU2IiwgInRpbWVzdGFtcCI6IDEyMzQ1Njc4NTB9",
    "has_more": true
}

GET /api/v1/feed?cursor=eyJwb3N0X2lkIjogInBvc3RfNDU2IiwgInRpbWVzdGFtcCI6IDEyMzQ1Njc4NTB9
Response: (next page of posts, next_cursor...)
```

**Boundary Conditions:**

```python
async def handle_pagination_edge_cases(self):
    """Обработка граничных случаев"""

    # 1. User has no posts
    posts = await get_feed_page(user_id='no_posts_user')
    assert posts['posts'] == []
    assert posts['next_cursor'] is None
    assert posts['has_more'] is False

    # 2. Cursor points to deleted post
    # Solution: Continue from next available post
    async def get_feed_safe(self, user_id, cursor):
        while cursor:
            posts = await get_feed_page(user_id, cursor)
            if posts['posts']:
                return posts  # Found valid posts
            cursor = posts['next_cursor']  # Try next page
        return {'posts': [], 'next_cursor': None}

    # 3. User unfollowed someone, timeline stale
    # Solution: Soft invalidate (short TTL, not hard delete)
    feed_cache.expire(f"feed:{user_id}", 60)  # Re-generate in 1 min

    # 4. Rapid pagination clicks
    # Solution: Prefetch, debounce requests, cache second page
```

**Performance Optimization:**

```python
# Проблема: fetch и hydrate 20 постов = 20 DB queries или 1 Redis batch

# Решение: Pipeline запросы
async def hydrate_posts_batched(self, post_ids: list) -> list:
    """Batch fetch post details"""

    # Get from post cache
    posts_map = {}
    missing_ids = []

    for post_id in post_ids:
        cached = await post_cache.get(post_id)
        if cached:
            posts_map[post_id] = cached
        else:
            missing_ids.append(post_id)

    # Batch query missing
    if missing_ids:
        missing_posts = await posts_db.query_batch(missing_ids)
        for post in missing_posts:
            posts_map[post['id']] = post
            await post_cache.cache(post['id'], post)

    # Return in original order
    return [posts_map[pid] for pid in post_ids if pid in posts_map]

# Оптимизация 2: Store partial data in cache
# Вместо полного поста, хранить только ID + title + score
# Hydrate full data (body, comments) only when user clicks
```

**Типичные ошибки.**
- Использовать OFFSET вместо cursor — O(N) queries
- Кодировать слишком много в cursor — размер растет
- Не обрабатывать deleted posts — пропуски в ленте
- Не limit cursor lifetime — старые cursors invalid

**На интервью.**
- Объясни почему offset плохо: O(N) скан, race conditions
- Покажи как encode/decode cursor
- Уточняющий вопрос: «Что если пост удалён?» — handle gracefully, skip to next
- Уточняющий вопрос: «Как prefetch следующую страницу?» — background fetch, store with short TTL

---

### 6. Как обрабатывать celebrity problem?

**Зачем спрашивают.** Celebrities создают hotspots: миллионы followers, миллионы лайков. Простые решения не масштабируются. Интервьюер проверяет понимание sharding, caching, и hybrid approaches.

**Короткий ответ.** Celebrities (>10K followers) требуют отдельной обработки. Используй pull-модель: пост сохраняется один раз в posts DB, при чтении ленты followers fetch его. Для likes/comments используй sharding по post_id. Кэшируй агрегированные метрики (likes count) с короткой консистентностью.

**Детальный разбор.**

**Проблема с push-моделью для celebrities:**

```
Celebrity A публикует пост
→ 10M followers
→ Fanout queue: 10,000 messages (батчи по 1000)
→ Fanout workers обрабатывают несколько часов
→ Проблемы:
  1. Queue bottleneck
  2. Cassandra hotspot (все пишут в одну таблицу)
  3. Cache thrashing (10M новых записей в Redis)
  4. Followers видят пост с задержкой

Решение: Pull-модель для celebrities
```

**Pull-модель для celebrities:**

```
Celebrity A публикует пост
→ 1 запись в posts DB (100ms)
→ Mark as celebrity post
→ Done!

Followers читают ленту
→ Get followees
→ Identify celebrities (cached)
→ Fetch recent posts from celebrities (parallel)
→ Merge с regular posts
→ Rank и return
```

**Hybrid Strategy Implementation:**

```python
CELEBRITY_THRESHOLD = 10000
WARM_THRESHOLD = 5000  # Category between regular and celebrity

async def classify_user(self, user_id: str) -> str:
    """Classify user based on followers"""
    count = await graph_db.get_follower_count(user_id)

    if count < WARM_THRESHOLD:
        return 'regular'
    elif count < CELEBRITY_THRESHOLD:
        return 'warm'
    else:
        return 'celebrity'

async def publish_post(self, author_id: str, content: str):
    """Publish with appropriate fanout strategy"""

    # Save post
    post_id = await posts_db.insert({
        'author_id': author_id,
        'content': content,
        'created_at': time.time()
    })

    # Classify author
    category = await classify_user(author_id)

    if category == 'regular':
        # Push-model: fanout immediately
        await self._fanout_to_followers(author_id, post_id)

    elif category == 'warm':
        # Semi-hybrid: fanout to top followers only (cache)
        top_followers = await graph_db.get_top_followers(author_id, limit=100K)
        for batch in chunks(top_followers, 1000):
            await fanout_queue.send({
                'post_id': post_id,
                'user_ids': batch
            })

    else:  # celebrity
        # Pull-model: mark as celebrity post
        await posts_db.update(post_id, {'is_celebrity': True})

    return post_id

async def get_feed(self, user_id: str) -> list:
    """Get feed with hybrid approach"""

    # 1. Get pre-computed feed from cache (push results)
    feed_posts = await feed_cache.get_feed(user_id, limit=80)

    # 2. Get celebrity followees
    followees = await graph_db.get_followees(user_id)
    celebrity_followees = [
        f for f in followees
        if await classify_user(f) == 'celebrity'
    ]

    # 3. Pull recent posts from celebrities
    if celebrity_followees:
        celebrity_posts = await asyncio.gather(*[
            posts_db.get_recent_by_author(f, limit=5)
            for f in celebrity_followees
        ], timeout=1.0)  # Timeout if takes too long

        celebrity_posts = flatten(celebrity_posts)
    else:
        celebrity_posts = []

    # 4. Merge
    all_posts = feed_posts + celebrity_posts

    # 5. Rank
    ranked = sorted(
        all_posts,
        key=lambda p: self._calculate_score(p, user_id),
        reverse=True
    )

    return ranked[:20]
```

**Likes/Comments Handling for Hot Posts:**

```
Проблема: Celebrity post gets 10M likes
→ Single row in likes table
→ Update counter = hotspot in DB
→ 100K QPS writes to same row = bottleneck

Решение: Sharding
```

```python
class HotPostCounter:
    """Shard likes counter across multiple rows"""

    SHARD_COUNT = 16  # Распределить на 16 шардов

    def get_shard_id(self, post_id: str, shard_count: int) -> int:
        """Deterministic sharding"""
        return hash(post_id) % shard_count

    async def increment_likes(self, post_id: str):
        """Increment likes counter (sharded)"""

        shard_id = self.get_shard_id(post_id, self.SHARD_COUNT)

        # Write to shard (spreads load)
        await likes_db.increment(
            f"post:{post_id}:likes:shard_{shard_id}"
        )

        # Also update aggregate in cache
        await cache.increment(f"post:{post_id}:likes_count")

    async def get_likes_count(self, post_id: str) -> int:
        """Get total likes (read from cache first)"""

        # Try cache
        cached = await cache.get(f"post:{post_id}:likes_count")
        if cached:
            return int(cached)

        # Cache miss: sum all shards
        total = 0
        for shard_id in range(self.SHARD_COUNT):
            count = await likes_db.get(f"post:{post_id}:likes:shard_{shard_id}")
            total += count

        # Cache result (short TTL, will be updated frequently)
        await cache.set(f"post:{post_id}:likes_count", total, ex=30)

        return total

# Usage
async def like_post(self, post_id: str, user_id: str):
    # Check if user already liked
    if await user_likes_post(post_id, user_id):
        return {'error': 'already liked'}

    # Add like (idempotent)
    await likes_db.add({
        'post_id': post_id,
        'user_id': user_id,
        'created_at': time.time()
    })

    # Increment counter (sharded)
    await like_counter.increment_likes(post_id)

    return {'status': 'ok'}
```

**Timeline for Celebrity Post:**

```
Timeline for regular user's post:

Follower 1          Follower 2          Follower 3
     │                   │                   │
     ▼                   ▼                   ▼
timeline(User1)   timeline(User2)   timeline(User3)
[post_123]        [post_123]        [post_123]

Storage: 1M posts × 1 byte = ~1MB per million followers


Timeline for celebrity post:

User1 follows Celebrity
User2 follows Celebrity
...
User10M follows Celebrity

posts (single record):
[celebrity_post_123]

User1 feed:         User2 feed:         User3 feed:
[own posts...]      [own posts...]      [own posts...]
[celebrity post]    [celebrity post]    [celebrity post]
(fetched at read)   (fetched at read)   (fetched at read)

Storage: 1 record, no fan-out
```

**Monitoring Celebrity Hotspots:**

```python
class HotspotMonitor:
    """Detect and alert on hotspots"""

    async def monitor_queue_depth(self):
        """Monitor fanout queue"""
        while True:
            depth = await fanout_queue.get_depth()

            if depth > 100000:
                await alert_on_slack(
                    f"Fanout queue depth: {depth}"
                )

            await asyncio.sleep(10)

    async def monitor_db_write_latency(self):
        """Monitor database write latency"""
        while True:
            p99_latency = await metrics.get_p99('db.write.latency')

            if p99_latency > 100:  # 100ms
                await alert_on_slack(
                    f"DB write latency high: {p99_latency}ms"
                )

            await asyncio.sleep(30)

    async def detect_new_celebrity(self):
        """Detect when user becomes celebrity"""
        while True:
            # Check for users that crossed threshold
            new_celebrities = await graph_db.query(
                "SELECT user_id FROM users "
                "WHERE followers > ? AND was_celebrity = false",
                CELEBRITY_THRESHOLD
            )

            for user_id in new_celebrities:
                await alert_on_slack(
                    f"New celebrity detected: {user_id}"
                )

                # Migrate their posts to pull-model
                await migrate_to_pull_model(user_id)
```

**Типичные ошибки.**
- Не отделять celebrities — push-модель для всех = bottleneck
- Шардировать только likes, забыть про comments — тот же hotspot для comments
- Не мониторить — узнаешь о проблеме когда система упала
- Hardcode threshold — нужно динамическое определение на основе CPU/QPS

**На интервью.**
- Объясни почему push-модель плохо для celebrities
- Покажи как спроектировать hybrid
- Уточняющий вопрос: «Как обрабатывать когда popular user becomes celebrity?» — async migration job
- Уточняющий вопрос: «Как sharding работает для комментариев?» — аналогично likes

---

### 7. Как хранить посты и связи между пользователями?

**Зачем спрашивают.** Storage design — основа системы. Интервьюер проверяет понимание выбора БД, индексирования, денормализации.

**Короткий ответ.** Используй 3 разные БД для разных целей: PostgreSQL для posts metadata (с индексами), Cassandra для timeline (append-only, распределённая), Neo4j или PostgreSQL с JSON для social graph (follow relations). Денормализуй с кэшем: store likes_count в posts table, updatable as cache, не read from likes table.

**Детальный разбор.**

**Database Selection:**

```
Posts Metadata:
├─ What: post_id, author_id, content, media_urls, created_at
├─ Access pattern: read by post_id, query by author_id + time
├─ Scale: 50M posts/day, 10K QPS reads
├─ Choice: PostgreSQL
│  Reasons: ACID, complex queries, secondary indexes
└─

Timeline (append-only):
├─ What: user_id, post_id, timestamp (for each user's feed)
├─ Access pattern: scan most recent (reverse ordered)
├─ Scale: 500M users × 100 posts = 50B entries
├─ Choice: Cassandra
│  Reasons: time-series, distributed, append-only
└─

Social Graph (relationships):
├─ What: follower_id, followee_id, relationship_type
├─ Access pattern: followers of X, followees of X, shortest path
├─ Scale: 500M users × 200 connections = 100B edges
├─ Choice: Neo4j or PostgreSQL with denormalization
│  Neo4j reasons: graph queries, traversal optimization
│  PostgreSQL reasons: simpler, fewer deps
└─
```

**PostgreSQL Schema:**

```sql
-- Posts table
CREATE TABLE posts (
    id              UUID PRIMARY KEY,
    author_id       UUID NOT NULL,
    content         TEXT,
    media_urls      TEXT[],          -- JSON array
    visibility      VARCHAR(20),     -- 'public', 'friends', 'private'
    likes_count     INT DEFAULT 0,   -- denormalized, updated via cache
    comments_count  INT DEFAULT 0,
    shares_count    INT DEFAULT 0,
    is_celebrity    BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    deleted_at      TIMESTAMP,       -- soft delete

    CONSTRAINT fk_author FOREIGN KEY (author_id) REFERENCES users(id)
);

-- Indexes for common queries
CREATE INDEX idx_posts_author_created
ON posts(author_id, created_at DESC)
WHERE deleted_at IS NULL;

CREATE INDEX idx_posts_created
ON posts(created_at DESC)
WHERE deleted_at IS NULL;

CREATE INDEX idx_posts_celebrity
ON posts(created_at DESC)
WHERE is_celebrity = TRUE AND deleted_at IS NULL;


-- Likes table (denormalized in posts.likes_count)
-- But for detailed like list:
CREATE TABLE likes (
    post_id         UUID NOT NULL,
    user_id         UUID NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (post_id, user_id)
);

CREATE INDEX idx_likes_post_created
ON likes(post_id, created_at DESC);

CREATE INDEX idx_likes_user_created
ON likes(user_id, created_at DESC);


-- Comments table
CREATE TABLE comments (
    id              UUID PRIMARY KEY,
    post_id         UUID NOT NULL,
    author_id       UUID NOT NULL,
    content         TEXT,
    likes_count     INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW(),
    deleted_at      TIMESTAMP,

    CONSTRAINT fk_post FOREIGN KEY (post_id) REFERENCES posts(id),
    CONSTRAINT fk_author FOREIGN KEY (author_id) REFERENCES users(id)
);

CREATE INDEX idx_comments_post_created
ON comments(post_id, created_at DESC)
WHERE deleted_at IS NULL;


-- Social graph
CREATE TABLE follows (
    follower_id     UUID NOT NULL,
    followee_id     UUID NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (follower_id, followee_id),
    CONSTRAINT fk_follower FOREIGN KEY (follower_id) REFERENCES users(id),
    CONSTRAINT fk_followee FOREIGN KEY (followee_id) REFERENCES users(id),
    CONSTRAINT cannot_follow_self CHECK (follower_id != followee_id)
);

CREATE INDEX idx_follows_followee
ON follows(followee_id);


-- User stats (denormalized for fast access)
CREATE TABLE user_stats (
    user_id             UUID PRIMARY KEY,
    followers_count     INT DEFAULT 0,
    following_count     INT DEFAULT 0,
    posts_count         INT DEFAULT 0,
    updated_at          TIMESTAMP DEFAULT NOW(),

    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**Cassandra Schema:**

```cql
-- Timeline table (wide column store)
CREATE TABLE user_timelines (
    user_id         UUID,
    post_timestamp  BIGINT,
    post_id         UUID,
    author_id       UUID,
    created_at      TIMESTAMP,

    PRIMARY KEY (user_id, post_timestamp, post_id)
) WITH CLUSTERING ORDER BY (post_timestamp DESC, post_id DESC)
AND default_time_to_live = 2592000;  -- 30 days retention

-- Partition key: user_id (one row per user)
-- Clustering key: post_timestamp (sorted, allows range queries)
-- Good for: zrevrange-like queries (get last 100 posts)

-- User's post timeline (posts they created)
CREATE TABLE author_timelines (
    author_id       UUID,
    post_timestamp  BIGINT,
    post_id         UUID,

    PRIMARY KEY (author_id, post_timestamp)
) WITH CLUSTERING ORDER BY (post_timestamp DESC);

-- Hot posts (for denormalization)
CREATE TABLE hot_posts (
    date_hour       TEXT,  -- '2024-01-15-10'
    post_id         UUID,
    likes_count     INT,

    PRIMARY KEY (date_hour, post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);
```

**Denormalization Strategy:**

```python
class PostStorage:
    """Store posts with denormalization"""

    def __init__(self, postgres_db, redis_cache):
        self.db = postgres_db
        self.cache = redis_cache

    async def create_post(self, author_id: str, content: str) -> str:
        """Create post with denormalized stats"""

        post = {
            'id': generate_uuid(),
            'author_id': author_id,
            'content': content,
            'created_at': time.time(),
            'likes_count': 0,        # Denormalized (will update via cache)
            'comments_count': 0,
            'shares_count': 0
        }

        # Save to DB
        await self.db.insert('posts', post)

        # Cache the post
        await self.cache.hset(f"post:{post['id']}", mapping=post)

        return post['id']

    async def increment_likes(self, post_id: str):
        """Increment likes counter"""

        # Update cache immediately (fast)
        await self.cache.hincr(f"post:{post_id}", 'likes_count', 1)

        # Batch update to DB (eventually)
        await self._async_queue.add({
            'op': 'increment_likes',
            'post_id': post_id
        })

    async def get_post(self, post_id: str) -> dict:
        """Get post with caching"""

        # Try cache
        cached = await self.cache.hgetall(f"post:{post_id}")
        if cached:
            return cached

        # DB query
        post = await self.db.query_one('posts', {'id': post_id})

        # Cache it
        await self.cache.hset(f"post:{post_id}", mapping=post)

        return post

    async def flush_likes_to_db(self):
        """Periodic job: sync cache likes to DB"""

        # Run every 1 minute
        while True:
            posts_to_update = await self._async_queue.get_batch(limit=10000)

            if not posts_to_update:
                await asyncio.sleep(60)
                continue

            # Batch update
            updates = defaultdict(int)
            for job in posts_to_update:
                updates[job['post_id']] += 1

            for post_id, increment in updates.items():
                await self.db.update(
                    'posts',
                    {'id': post_id},
                    {'likes_count': self.db.sql.literal(f'likes_count + {increment}')}
                )

            await asyncio.sleep(60)
```

**Indexing Strategy:**

```sql
-- Index for feed queries (most common)
-- Pattern: SELECT * FROM posts WHERE author_id = ? ORDER BY created_at DESC
CREATE INDEX idx_posts_feed
ON posts(author_id, created_at DESC)
WHERE deleted_at IS NULL;

-- Index for search/explore
-- Pattern: SELECT * FROM posts ORDER BY created_at DESC (with limit)
CREATE INDEX idx_posts_explore
ON posts(created_at DESC)
WHERE deleted_at IS NULL AND visibility = 'public';

-- Index for celebrity posts
-- Pattern: SELECT * FROM posts WHERE is_celebrity = true ORDER BY likes DESC
CREATE INDEX idx_posts_hot
ON posts(is_celebrity, likes_count DESC, created_at DESC)
WHERE deleted_at IS NULL;

-- Analyze plan to verify indexes work
EXPLAIN ANALYZE
SELECT * FROM posts
WHERE author_id = 'uuid'
ORDER BY created_at DESC
LIMIT 20;

-- Should see: Index Scan using idx_posts_feed
```

**Sharding Pattern:**

```python
class ShardedStorage:
    """Shard data across multiple DB instances"""

    def __init__(self, shards: list):
        self.shards = shards  # List of DB connections
        self.shard_count = len(shards)

    def get_shard_id(self, key: str) -> int:
        """Consistent hashing"""
        return hash(key) % self.shard_count

    async def insert_post(self, post: dict) -> str:
        """Insert post into appropriate shard"""

        shard_id = self.get_shard_id(post['author_id'])
        await self.shards[shard_id].insert('posts', post)

        return post['id']

    async def get_posts_by_author(self, author_id: str) -> list:
        """Query posts from author's shard"""

        shard_id = self.get_shard_id(author_id)
        posts = await self.shards[shard_id].query(
            'posts',
            {'author_id': author_id},
            order_by=[('created_at', 'DESC')]
        )

        return posts
```

**Типичные ошибки.**
- Хранить likes_count в отдельной таблице, query при каждом read — медленно
- Не индексировать по author_id + created_at — full table scan для author's posts
- Использовать текст вместо UUID — много места, медленнее
- Не мониторить index fragmentation — со временем scan становится медленнее

**На интервью.**
- Объясни выбор PostgreSQL vs Cassandra vs Neo4j
- Покажи индексы для common queries
- Уточняющий вопрос: «Как денормализовать likes_count?» — cache layer, async batch updates
- Уточняющий вопрос: «Как шардировать?» — consistent hash по author_id или post_id

---

### 8. Как реализовать real-time updates в ленте?

**Зачем спрашивают.** Real-time updates требуют другого подхода (WebSocket, Server-Sent Events). Интервьюер проверяет понимание push-технологий, message queues, и масштабирования WebSocket.

**Короткий ответ.** Используй WebSocket для persistent connections, Redis Pub/Sub для распределённых сообщений. При публикации поста отправить event в Redis (постоянно живые followers), subscribe на `feed_updates:{user_id}`, получить новый пост через WebSocket. Масштабируй с помощью sticky sessions или redis streams.

**Детальный разбор.**

**Real-time Architecture:**

```
User publishes post
       │
       ▼
Post Service
       │
       ├─→ Save to DB
       │
       ├─→ Publish to Redis Pub/Sub
       │
       └─→ Fanout Service (async)
              │
              └─→ Add to feeds (for push-model)

Redis Pub/Sub:
channel: "feed_updates:{user_id}"
         │
         ├─→ Follower1 (WebSocket connected)
         ├─→ Follower2 (WebSocket connected)
         └─→ Follower3 (WebSocket connected)

WebSocket Server receives message
       │
       ▼
Send to all connected clients
       │
       ├─→ {type: 'new_post', post_id: '...'}
       ├─→ {type: 'like', post_id: '...', count: 150}
       └─→ {type: 'comment', post_id: '...', count: 5}
```

**WebSocket Implementation:**

```python
import asyncio
import websockets
import json
from redis import Redis
from typing import Set

class FeedWebSocketServer:
    def __init__(self, redis_url: str):
        self.redis = Redis.from_url(redis_url)
        self.connections: dict[str, Set[websockets.WebSocketServerProtocol]] = {}

    async def handle_client(self, websocket, path: str):
        """Handle WebSocket connection"""

        user_id = None
        try:
            # Wait for auth message
            auth_msg = await asyncio.wait_for(websocket.recv(), timeout=5.0)
            auth_data = json.loads(auth_msg)
            user_id = auth_data.get('user_id')

            if not user_id:
                await websocket.close(1008, 'No user_id provided')
                return

            # Register connection
            if user_id not in self.connections:
                self.connections[user_id] = set()
            self.connections[user_id].add(websocket)

            print(f"User {user_id} connected. Total: {len(self.connections[user_id])}")

            # Subscribe to Redis
            channel = f"feed_updates:{user_id}"
            subscriber = self.redis.pubsub()
            subscriber.subscribe(channel)

            # Listen for messages
            async def listen_redis():
                for message in subscriber.listen():
                    if message['type'] == 'message':
                        await self._broadcast_to_user(
                            user_id,
                            message['data'].decode()
                        )

            # Listen for client messages
            async def listen_client():
                try:
                    async for message in websocket:
                        # Handle client actions (like, comment, etc)
                        await self._handle_client_action(user_id, message)
                except websockets.exceptions.ConnectionClosed:
                    pass

            # Run both concurrently
            await asyncio.gather(
                listen_redis(),
                listen_client(),
                return_exceptions=True
            )

        finally:
            if user_id and user_id in self.connections:
                self.connections[user_id].discard(websocket)
                if not self.connections[user_id]:
                    del self.connections[user_id]
                print(f"User {user_id} disconnected")

            subscriber.close()

    async def _broadcast_to_user(self, user_id: str, message: str):
        """Broadcast message to all connections of a user"""

        if user_id not in self.connections:
            return

        # Send to all connected clients
        dead_connections = set()
        for ws in self.connections[user_id]:
            try:
                await ws.send(message)
            except:
                dead_connections.add(ws)

        # Clean up dead connections
        for ws in dead_connections:
            self.connections[user_id].discard(ws)

    async def _handle_client_action(self, user_id: str, message: str):
        """Handle actions from client (like, comment)"""

        try:
            data = json.loads(message)
            action = data.get('action')

            if action == 'like':
                post_id = data.get('post_id')
                await like_post(user_id, post_id)

            elif action == 'comment':
                post_id = data.get('post_id')
                content = data.get('content')
                await comment_post(user_id, post_id, content)

        except json.JSONDecodeError:
            pass

# Start server
async def main():
    async with websockets.serve(
        FeedWebSocketServer('redis://localhost').handle_client,
        'localhost',
        8765
    ):
        await asyncio.Future()  # run forever

# asyncio.run(main())
```

**Client-side JavaScript:**

```javascript
class FeedRealtimeClient {
    constructor(userId) {
        this.userId = userId;
        this.ws = null;
        this.messageHandlers = {};
    }

    connect(wsUrl) {
        this.ws = new WebSocket(wsUrl);

        this.ws.onopen = () => {
            // Send auth
            this.ws.send(JSON.stringify({
                user_id: this.userId
            }));

            console.log('Connected to feed server');
        };

        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            this._handleMessage(message);
        };

        this.ws.onerror = (error) => {
            console.error('WebSocket error:', error);
        };

        this.ws.onclose = () => {
            console.log('Disconnected from feed server');
            // Reconnect after 3 seconds
            setTimeout(() => this.connect(wsUrl), 3000);
        };
    }

    _handleMessage(message) {
        const {type, data} = message;

        if (type === 'new_post') {
            // Add to top of feed
            this._insertPostAtTop(data);
        } else if (type === 'like') {
            // Update like count
            this._updateLikeCount(data.post_id, data.count);
        } else if (type === 'comment') {
            // Update comment count
            this._updateCommentCount(data.post_id, data.count);
        }
    }

    sendAction(action, data) {
        this.ws.send(JSON.stringify({
            action,
            ...data
        }));
    }

    _insertPostAtTop(post) {
        const feedElement = document.getElementById('feed');
        const postElement = createPostElement(post);
        feedElement.insertBefore(postElement, feedElement.firstChild);

        // Animate
        postElement.style.opacity = '0';
        postElement.animate([
            {opacity: 0, transform: 'translateY(-20px)'},
            {opacity: 1, transform: 'translateY(0)'}
        ], {duration: 300});
    }

    _updateLikeCount(postId, count) {
        const element = document.querySelector(`[data-post-id="${postId}"] .like-count`);
        if (element) {
            element.textContent = count;
        }
    }
}

// Usage
const client = new FeedRealtimeClient('user_123');
client.connect('ws://localhost:8765');
```

**Publishing Real-time Updates:**

```python
class PostPublisher:
    def __init__(self, redis_client):
        self.redis = redis_client

    async def publish_new_post(self, author_id: str, post_id: str,
                              followers: list):
        """Publish new post to followers' feeds"""

        post_data = await posts_db.get_post(post_id)

        message = {
            'type': 'new_post',
            'data': {
                'post_id': post_id,
                'author_id': author_id,
                'content': post_data['content'],
                'media': post_data.get('media', []),
                'created_at': post_data['created_at']
            }
        }

        # Publish to each follower
        for batch in chunks(followers, 1000):
            # Batch publish to Redis (more efficient)
            pipe = self.redis.pipeline()
            for user_id in batch:
                channel = f"feed_updates:{user_id}"
                pipe.publish(channel, json.dumps(message))
            pipe.execute()

    async def publish_like(self, post_id: str, likes_count: int):
        """Publish like update"""

        post = await posts_db.get_post(post_id)
        author_id = post['author_id']

        message = {
            'type': 'like',
            'data': {
                'post_id': post_id,
                'count': likes_count
            }
        }

        # Publish to post author (to see likes on their posts)
        self.redis.publish(
            f"feed_updates:{author_id}",
            json.dumps(message)
        )
```

**Scaling WebSocket Servers:**

```
Проблема: 1 WebSocket сервер может держать ~50K connections
         500M DAU → нужно 10K серверов (очень дорого)

Решение 1: Redis Pub/Sub (масштабируется)
- Все серверы subscribe на Redis channels
- Сообщение через Redis идёт на все серверы
- Сервер отправляет только своим connected клиентам

Решение 2: Sticky Sessions
- Load Balancer направляет клиента на тот же сервер
- Сервер имеет in-memory список connections
- Меньше Redis traffic

Решение 3: Redis Streams
- Вместо Pub/Sub, использовать Streams
- Позволяет persistent storage, replay, consumer groups
```

```python
class ScalableWebSocketServer:
    """WebSocket server with Redis backend"""

    def __init__(self, redis_url: str, server_id: str):
        self.redis = Redis.from_url(redis_url)
        self.server_id = server_id
        self.local_connections = {}  # user_id -> {websockets}

    async def handle_client(self, websocket, path):
        user_id = await self._authenticate(websocket)

        # Register in local map
        if user_id not in self.local_connections:
            self.local_connections[user_id] = set()
        self.local_connections[user_id].add(websocket)

        # Register in Redis (for message routing)
        await self.redis.sadd(f"user:{user_id}:servers", self.server_id)

        try:
            async for message in websocket:
                await self._handle_message(user_id, message)
        finally:
            self.local_connections[user_id].discard(websocket)
            if not self.local_connections[user_id]:
                del self.local_connections[user_id]
                await self.redis.srem(f"user:{user_id}:servers", self.server_id)

    async def broadcast_to_user(self, user_id: str, message: str):
        """Broadcast message to user (may be on multiple servers)"""

        # Send to local connections
        if user_id in self.local_connections:
            for ws in self.local_connections[user_id]:
                try:
                    await ws.send(message)
                except:
                    pass

        # Publish to Redis (reaches other servers)
        await self.redis.publish(f"feed_updates:{user_id}", message)
```

**Типичные ошибки.**
- Отправлять все обновления через WebSocket — traffic overload
- Не батчить Redis operations — too many round trips
- Не reconnect при disconnection — UX деградирует
- Держать стейт на WebSocket сервере — не восстанавливается при перезагрузке

**На интервью.**
- Объясни архитектуру: WebSocket + Redis Pub/Sub
- Как масштабировать 500M DAU? — Redis, multiple servers, sticky sessions
- Уточняющий вопрос: «Как обеспечить delivery guarantee?» — Redis Streams + consumer groups
- Уточняющий вопрос: «Как retry при connection loss?» — exponential backoff, queue messages locally

---

### 9. Как фильтровать и персонализировать контент?

**Зачем спрашивают.** Персонализация улучшает engagement. Интервьюер проверяет понимание ML, filtering, и user preferences.

**Короткий ответ.** Используй 3 уровня фильтрации: 1) Hard filters (удалить blocked users, explicit flags), 2) Soft ranking (affinity, time decay, engagement), 3) ML ranking (trained on user behavior). Кэшируй user preferences (interests, blacklists), apply в feed generation. Для A/B testing используй bucketing по user_id.

**Детальный разбор.**

**Filtering Pipeline:**

```
Raw Posts
    │
    ├─→ [1] Hard Filters
    │       ├─ Remove posts from blocked users
    │       ├─ Remove posts user explicitly reported
    │       ├─ Remove posts with adult content (if user != adult)
    │       └─ Remove spam (ML classifier)
    │
    ├─→ [2] Personalization
    │       ├─ Language filter (user's preferred languages)
    │       ├─ Content type filter (no videos if on mobile)
    │       ├─ Interest filter (topics user follows)
    │       └─ Recency filter (don't show very old posts)
    │
    ├─→ [3] Ranking
    │       ├─ Affinity score (history with author)
    │       ├─ Engagement score (likes, comments)
    │       ├─ Novelty score (how fresh)
    │       └─ Diversity score (don't cluster same authors)
    │
    └─→ [4] ML Model
            ├─ Feature extraction
            ├─ GBDT / Neural Network
            └─ Final ranking score

Result: Personalized Feed
```

**Implementation:**

```python
class PersonalizationEngine:
    def __init__(self, ml_model, redis_cache):
        self.model = ml_model
        self.cache = redis_cache

    async def personalize_feed(self, user_id: str,
                               raw_posts: list) -> list:
        """Complete personalization pipeline"""

        # 1. Get user preferences
        user_prefs = await self._get_user_preferences(user_id)

        # 2. Hard filters
        filtered_posts = await self._apply_hard_filters(
            raw_posts, user_id, user_prefs
        )

        # 3. Soft filters
        filtered_posts = await self._apply_soft_filters(
            filtered_posts, user_id, user_prefs
        )

        # 4. ML ranking
        ranked_posts = await self._ml_rank(
            filtered_posts, user_id, user_prefs
        )

        return ranked_posts

    async def _get_user_preferences(self, user_id: str) -> dict:
        """Get user preferences from cache/DB"""

        cache_key = f"user_prefs:{user_id}"

        # Try cache
        cached = await self.cache.get(cache_key)
        if cached:
            return json.loads(cached)

        # Query DB
        prefs = {
            'language': await self._get_user_language(user_id),
            'interests': await self._get_user_interests(user_id),
            'blocked_users': await self._get_blocked_users(user_id),
            'device_type': await self._get_device_type(user_id),
            'is_adult': await self._is_user_adult(user_id),
        }

        # Cache (TTL: 1 hour)
        await self.cache.set(cache_key, json.dumps(prefs), ex=3600)

        return prefs

    async def _apply_hard_filters(self, posts: list, user_id: str,
                                  prefs: dict) -> list:
        """Remove posts user should not see"""

        filtered = []

        for post in posts:
            # Check blacklist
            if post['author_id'] in prefs['blocked_users']:
                continue

            # Check explicit report
            if await self._is_post_reported_by_user(user_id, post['id']):
                continue

            # Check adult content
            if post.get('is_adult') and not prefs['is_adult']:
                continue

            # Check spam
            if await self._is_likely_spam(post):
                continue

            filtered.append(post)

        return filtered

    async def _apply_soft_filters(self, posts: list, user_id: str,
                                  prefs: dict) -> list:
        """Filter based on preferences"""

        filtered = []

        for post in posts:
            # Language match
            post_lang = post.get('language', 'en')
            if post_lang != prefs['language']:
                continue

            # Content type match (device-aware)
            if prefs['device_type'] == 'mobile' and post['type'] == 'video':
                # Expensive to load on mobile (could skip or reduce)
                continue

            # Interest match
            post_topics = post.get('topics', [])
            user_interests = prefs['interests']
            if post_topics and not any(t in user_interests for t in post_topics):
                # Post has specific topics, none match user interests
                # (you may soft-filter, not hard)
                if len(post_topics) > 2 and len(user_interests) > 0:
                    continue

            filtered.append(post)

        return filtered

    async def _ml_rank(self, posts: list, user_id: str,
                      prefs: dict) -> list:
        """Rank using ML model"""

        # Extract features for each post
        features_list = []
        for post in posts:
            features = self._extract_features(post, user_id, prefs)
            features_list.append(features)

        # Batch predict
        scores = self.model.predict(features_list)

        # Add scores to posts
        ranked = [
            {**post, 'ml_score': score}
            for post, score in zip(posts, scores)
        ]

        # Sort by score
        ranked.sort(key=lambda p: p['ml_score'], reverse=True)

        return ranked

    def _extract_features(self, post: dict, user_id: str,
                         prefs: dict) -> list:
        """Extract ML features from post"""

        # Time decay
        age_hours = (time.time() - post['created_at']) / 3600
        time_decay = 1.0 / (1.0 + age_hours * 0.1)

        # Affinity with author
        affinity = get_affinity(user_id, post['author_id'])

        # Engagement signals
        engagement = (
            post.get('likes_count', 0) +
            2 * post.get('comments_count', 0) +
            3 * post.get('shares_count', 0)
        )

        # User interaction history
        user_interactions = get_user_interactions(user_id)

        # Post characteristics
        content_type_embedding = get_content_embedding(post['type'])

        # Topic matching
        topic_match = sum(1 for t in post.get('topics', [])
                         if t in prefs['interests'])

        features = [
            time_decay,
            affinity,
            math.log(1 + engagement),
            len(user_interactions),
            topic_match,
            post.get('length', 0),  # Text length
            # ... more features (256-dimensional vector typically)
        ]

        return features
```

**A/B Testing Framework:**

```python
class ABTestingFramework:
    """Control experiments with user bucketing"""

    BUCKET_SIZE = 10000  # 0.1% granularity

    def get_bucket(self, user_id: str, experiment_id: str) -> str:
        """Deterministic bucketing"""

        # Hash user_id + experiment_id
        hash_value = hash(f"{user_id}_{experiment_id}") % 100

        # Split: 50% control, 50% variant
        if hash_value < 50:
            return 'control'
        else:
            return 'variant'

    async def get_personalization_config(self, user_id: str) -> dict:
        """Get ML model config based on experiments"""

        bucket = self.get_bucket(user_id, 'ranking_model_v2')

        if bucket == 'control':
            # Old model
            return {
                'model': 'ranking_v1',
                'ml_weight': 0.3,
                'affinity_weight': 0.4,
                'engagement_weight': 0.3
            }
        else:  # variant
            # New model
            return {
                'model': 'ranking_v2',
                'ml_weight': 0.7,
                'affinity_weight': 0.2,
                'engagement_weight': 0.1
            }

    async def log_experiment_result(self, user_id: str,
                                   experiment_id: str,
                                   metric: str,
                                   value: float):
        """Log metrics for analysis"""

        bucket = self.get_bucket(user_id, experiment_id)

        # Send to analytics
        await analytics.log({
            'user_id': user_id,
            'experiment_id': experiment_id,
            'bucket': bucket,
            'metric': metric,
            'value': value,
            'timestamp': time.time()
        })
```

**Interest-based Filtering:**

```python
class InterestFiltering:
    """Filter based on user interests"""

    async def get_user_interests(self, user_id: str) -> set:
        """Get interests from user's explicit follows"""

        # Users can follow topics (not just people)
        # SELECT topic_id FROM user_follows_topic WHERE user_id = ?

        interests = await db.query(
            'SELECT topic_id FROM user_follows_topic WHERE user_id = ?',
            user_id
        )

        return {row['topic_id'] for row in interests}

    async def get_post_topics(self, post_id: str) -> list:
        """Extract topics from post"""

        # Could be:
        # 1. Explicit tags from author
        # 2. ML NLP classifier
        # 3. Hashtag parsing

        post = await posts_db.get(post_id)
        topics = []

        # Parse hashtags
        for hashtag in re.findall(r'#(\w+)', post['content']):
            topics.append(f'topic_{hashtag.lower()}')

        # NLP classification (batch)
        if not topics:
            nlp_topics = await nlp_service.classify(post['content'])
            topics.extend(nlp_topics)

        return topics

    async def matches_interests(self, post_id: str, user_id: str) -> float:
        """Compute interest match score [0..1]"""

        post_topics = await self.get_post_topics(post_id)
        user_interests = await self.get_user_interests(user_id)

        if not post_topics or not user_interests:
            return 0.5  # Neutral

        matches = len(set(post_topics) & user_interests)
        total = len(set(post_topics) | user_interests)

        return matches / total if total > 0 else 0.5
```

**Типичные ошибки.**
- Не кэшировать preferences — каждый read пересчитывает
- ML ranking без fallback — если модель fails, нет feed
- Не монитор фильтрацию — может отсечь слишком много
- Слишком агрессивная фильтрация — user видит только знакомое, скучно

**На интервью.**
- Объясни 4-уровневый pipeline: hard filters → soft filters → ranking → ML
- Как A/B test новый ranking model? — bucketing, logging metrics
- Уточняющий вопрос: «Как обнаружить bias в ML модели?» — fairness metrics, cross-demographic analysis
- Уточняющий вопрос: «Как позволить user'у контролировать фильтрацию?» — preferences UI, explicit interests

---

### 10. Как масштабировать news feed?

**Зачем спрашивают.** Масштабирование требует понимания bottlenecks и их решения. Интервьюер проверяет системное мышление.

**Короткий ответ.** Масштабируй по слоям: 1) Database sharding (по user_id или post_id), 2) Cache layering (Redis + CDN), 3) Compute (stateless, load balance), 4) Async processing (Kafka fanout queue). Мониторь metrics (QPS, latency, cache hit rate), оптимизируй bottleneck. Для 500M DAU: 10 DB shards, 100 Redis instances, 50 feed servers, 1000 fanout workers.

**Детальный разбор.**

**Scaling Checklist:**

```
Компонент          Baseline         500M DAU         Стратегия
────────────────────────────────────────────────────────────────
Database (Posts)   1 instance       10 shards       Shard by author_id
Timeline (Cass)    1 cluster        5 clusters      Geographic distribution
Cache (Redis)      1 instance       100 instances   Redis Cluster
Feed Service       1 server         50 servers      Stateless load balance
Fanout Workers     10 workers       1000 workers    Kafka consumer group
WebSocket Servers  -                10K servers     Sticky sessions + Redis
────────────────────────────────────────────────────────────────
```

**Database Scaling:**

```
Single PostgreSQL instance: max 10K QPS

Шардинг по author_id:
Shard 0: authors 0,16,32,... → queries from followers
Shard 1: authors 1,17,33,... → queries from followers
...
Shard N-1

Advantage:
- 10 shards × 10K QPS = 100K QPS total
- Followers' queries load balanced

Disadvantage:
- Cross-shard query hard (get posts from multiple authors)
- Schema migration complex

Code:
def get_shard_id(author_id):
    return hash(author_id) % 10

def insert_post(author_id, post):
    shard_id = get_shard_id(author_id)
    db_shards[shard_id].insert('posts', post)

def get_user_posts(author_id, limit):
    shard_id = get_shard_id(author_id)
    return db_shards[shard_id].query(
        'SELECT * FROM posts WHERE author_id = ? ORDER BY created_at DESC LIMIT ?',
        author_id, limit
    )
```

**Redis Cluster Scaling:**

```
Single Redis: max 100K QPS, 32GB memory

Redis Cluster: 100 nodes
- Each node: ~1K QPS, ~320MB memory
- 100 nodes × 1K = 100K QPS
- Consistent hashing distributes keys

Slot mapping:
16384 slots total
slot = crc16(key) % 16384
node = slot_to_node[slot]

Client example:
redis_cluster = redis.RedisCluster(['localhost:7000', ...])
redis_cluster.set(f'feed:{user_id}', ...)
```

**Compute Scaling (Feed Service):**

```
Stateless design:
- No local state
- Can kill/restart any instance
- Easy horizontal scaling

Load Balancing:
Client → Load Balancer (round-robin / least connections)
         ├→ Feed Service 1
         ├→ Feed Service 2
         ...
         └→ Feed Service 50

Configuration:
- 50 Feed Service instances
- Each: 4 CPU, 16GB RAM
- Handles 1K feed requests/sec
- Total: 50K feed QPS
```

**Fanout Queue Scaling:**

```
Single worker: 1K fanouts/sec

For celebrity post (10M followers):
- 10M / 1K = 10K seconds = 2.8 hours wait

Solution: Kafka + Consumer Group
- Partition posts into 1000 topics
- 1000 workers, each consuming 1 partition
- 1000 workers × 1K = 1M fanouts/sec
- 10M / 1M = 10 seconds fanout time

Kafka config:
topic: 'fanout_jobs'
partitions: 1000
replication_factor: 3
retention: 1 day

Consumer Group:
group_id: 'fanout_workers'
instances: 1000
```

**Complete Scaling Architecture:**

```
┌────────────────────────────────────────────────┐
│           Clients (500M DAU)                    │
│     Web / Mobile / Desktop / API                │
└────────────┬─────────────────────────────────┘
             │
┌────────────▼──────────────────────────────────┐
│    Load Balancer (Geographic LB)               │
│    Health checks, rate limiting                │
└────────────┬──────────────────────────────────┘
             │
    ┌────────┼────────┬────────┬──────────┐
    │        │        │        │          │
┌───▼─┐ ┌───▼─┐ ┌───▼─┐ ┌───▼─┐ ┌──────▼┐
│ DC1 │ │ DC2 │ │ DC3 │ │ DC4 │ │ ...  │
└──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬───┘
   │       │       │       │       │
   └───────┴───┬───┴───────┴───────┘
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼────────┐   ┌─────────▼────┐
│ Feed Svcs  │   │ Post Svcs    │
│ (50x)      │   │ (30x)        │
└───┬────────┘   └──────┬───────┘
    │                   │
    │       ┌───────────┴──────────┐
    │       │                      │
┌───▼─────────────────┐  ┌────────▼──────────┐
│ Redis Cluster       │  │ PostgreSQL Shards │
│ (100 nodes)         │  │ (10 shards)       │
│ - Feed cache        │  │ - Posts metadata  │
│ - Post details      │  │ - Likes/Comments  │
│ - Graph cache       │  │ - User stats      │
└─────────────────────┘  └───────────────────┘

                         ┌──────────────────┐
                         │ Cassandra        │
                         │ (5 clusters)     │
                         │ - Timelines      │
                         └──────────────────┘

    ┌─────────────────┐
    │ Message Queues  │
    │ Kafka (10K      │
    │ topics)         │
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │ Worker Pools    │
    │ (5000 workers)  │
    │ - Fanout jobs   │
    │ - Indexing      │
    │ - Analytics     │
    └─────────────────┘
```

**Monitoring Scaling Health:**

```python
class ScalingMetrics:
    async def monitor_all(self):
        while True:
            # Database
            db_qps = await get_db_qps()
            if db_qps > 8000:  # Approaching limit (10K)
                alert("DB QPS high, consider adding shard")

            # Cache
            cache_hit_rate = await get_cache_hit_rate()
            if cache_hit_rate < 0.7:  # Below 70%
                alert("Cache hit rate low, increase cache size")

            # Fanout queue
            queue_depth = await get_queue_depth()
            if queue_depth > 100000:  # 100K messages
                alert("Queue backing up, add workers")

            # Feed latency
            p99_latency = await get_p99_feed_latency()
            if p99_latency > 500:  # 500ms
                alert("Feed latency high, check database/cache")

            await asyncio.sleep(60)
```

**Performance Optimization Tips:**

```python
# 1. Database query optimization
# Instead of:
posts = db.query('SELECT * FROM posts WHERE author_id = ? ORDER BY created_at DESC', author_id)
posts = [p for p in posts if p.created_at > time.time() - 30*86400]

# Do:
posts = db.query(
    'SELECT * FROM posts WHERE author_id = ? AND created_at > ? ORDER BY created_at DESC LIMIT 100',
    author_id,
    time.time() - 30*86400
)

# 2. Batch operations
# Instead of:
for post_id in post_ids:
    post = redis.get(f'post:{post_id}')

# Do:
posts = redis.mget([f'post:{pid}' for pid in post_ids])

# 3. Connection pooling
# Instead of:
conn = redis.Redis()  # New connection per request

# Do:
pool = redis.ConnectionPool(max_connections=100)
conn = redis.Redis(connection_pool=pool)

# 4. Compression
# Instead of:
self.redis.hset(f'post:{post_id}', mapping=post)  # 5KB per post

# Do:
compressed = gzip.compress(json.dumps(post).encode())
self.redis.set(f'post:{post_id}', compressed)  # 0.5KB per post
```

**Typical Error Budget:**

```
System availability target: 99.99%
= 52.6 minutes downtime per year
= 4.38 minutes per month

Error budget allocation:
- Database outages: 2 min/month (50%)
- Deployment incidents: 1 min/month (20%)
- Cache failures: 0.5 min/month (10%)
- Network issues: 0.5 min/month (10%)
- Emergency fixes: 0.38 min/month (10%)

Monitoring alerts:
- DB CPU > 80%: page on-call
- Cache hit < 65%: page on-call
- Feed latency p99 > 1s: escalate to eng team
- Queue depth > 1M: pause publishing
```

**Типичные ошибки.**
- Масштабировать все одновременно — сложно определить bottleneck
- Добавлять кэш без измерения hit rate — может быть напрасно
- Не мониторить скрытые bottlenecks (network, CPU context switching)
- Преждевременная оптимизация — оптимизируй то, что медленно

**На интервью.**
- Нарисуй scaling путь: 1K DAU → 1M DAU → 1B DAU
- Объясни как масштабировать каждый слой
- Уточняющий вопрос: «Как обнаружить bottleneck?» — metrics, profiling, load testing
- Уточняющий вопрос: «Какой максимум DAU за одним шардом?» — зависит от write pattern, обычно 50-100M

---

## См. также

- [Распределённый кэш](./07-distributed-cache.md) — техники кэширования для высоконагруженных систем
- [Шардирование баз данных](../06-databases/06-sharding.md) — партиционирование данных для масштабирования
- [Message Queues](./08-message-queues.md) — асинхронная обработка и fanout
- [Система рекомендаций](./09-recommendation-system.md) — персонализация и ML ranking

---

## Практика

1. **Базовая архитектура** — спроектируй feed систему на бумаге для 1M DAU. Определи компоненты, databases, cache layers.

2. **Fanout optimization** — реализуй гибридный fanout (push для обычных, pull для celebrities). Напиши unit tests.

3. **Ranking formula** — реализуй ranking formula с time decay, affinity, engagement. Протестируй на synthetic data.

4. **Pagination** — реализуй cursor-based pagination. Обработай edge case когда пост удалён.

5. **Redis sharding** — разверни Redis Cluster (или эмулируй) и реализуй consistent hashing для feed cache.

6. **Load test** — создай load test с градуальным увеличением DAU. Определи bottlenecks при 10K, 50K, 100K QPS.

7. **A/B testing** — реализуй bucketing и логирование для 2 версий ranking алгоритма.

8. **Real-time mock** — напиши простой WebSocket сервер, который реализует push новых постов.

---

## Дополнительные материалы

- [Facebook's Paper on Feed Ranking](https://research.facebook.com/blog//) — EdgeRank algorithm
- [Twitter's Architecture](https://blog.twitter.com/engineering) — timeline at scale
- [Instagram's Scaling](https://instagram-engineering.com/) — handling billions of stories
- [System Design Interview by Alex Xu](https://www.amazon.com/) — detailed feed design chapter
- [Designing Data-Intensive Applications](https://dataintensive.systems/) — Chapter on batch processing for feeds
- [Cassandra Documentation](https://cassandra.apache.org/doc/) — time-series databases
- [Redis Cluster](https://redis.io/topics/cluster-tutorial) — distributed caching

---

← [04-chat-messenger](./04-chat-messenger.md) | [Трек System Design](./README.md) | [06-search-autocomplete](./06-search-autocomplete.md) →
