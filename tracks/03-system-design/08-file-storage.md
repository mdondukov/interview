# 08 — File Storage / Cloud Storage

Развёрнутые вопросы и ответы про проектирование систем хранения файлов: загрузка, скачивание, чанкирование, дедупликация, метаданные, CDN, репликация, масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [07-distributed-cache](./07-distributed-cache.md) · Следующая тема: [09-video-streaming](./09-video-streaming.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Multipart Upload** — это стратегия загрузки больших файлов по частям (чанкам) параллельно вместо загрузки целиком за один запрос. Например, 1 ГБ файл может быть разделён на 100 чанков по 10 МБ, каждый из которых загружается отдельно и параллельно. Это значительно повышает скорость загрузки и позволяет возобновить загрузку с того места, где она была прервана, если соединение разорвалось. AWS S3, Google Cloud Storage и другие облачные сервисы поддерживают multipart upload.

**Chunking** — это разделение файла на равные куски или блоки (например, по 5 МБ). Чанкирование позволяет паралелизировать загрузку нескольких чанков одновременно, эффективнее управлять памятью (не нужно загружать весь файл в оперативную память) и обрабатывать файлы любого размера, даже больше доступной памяти сервера. Типичный размер чанка: 1-10 МБ.

**Presigned URL** — это временный URL с криптографической подписью, который даёт клиенту право загружать или скачивать файл напрямую в облачное хранилище (S3, GCS) без участия вашего API сервера. Клиент получает URL, говорит браузер загружает файл прямо в S3, и API сервер не потребляет пропускную способность. Это снижает нагрузку на ваши серверы и значительно ускоряет загрузку больших файлов.

**Block Storage** — это распределённое хранилище данных на уровне блоков (обычно 4-128 МБ), где каждый блок реплицирован на несколько узлов для надёжности. Block storage отличается от объектного хранилища тем, что оптимизирован для высокого throughput и reliability, обеспечивает очень высокую durability (11 "девяток", то есть 99.999999999% гарантия) и масштабируется на петабайты данных.

**Metadata Service** — это отдельный микросервис, который хранит информацию о файлах в базе данных (имя, размер, владелец, теги, дату создания, версию и т.д.). Metadata service позволяет быстро искать файлы по различным критериям, фильтровать, сортировать, управлять доступом — всё это без прямого обращения в медленное block storage, где хранятся фактические данные файлов.

**CDN (Content Delivery Network)** — это глобальная сеть серверов, географически распределённых по разным регионам и странам, которые кэшируют контент близко к конечным пользователям. Благодаря CDN, пользователь в России может скачать файл не с серверов в США за 100-200 мс, а с региональной CDN точки в России за 10-20 мс. CDN также снижает bandwidth на origin сервере.

**Replication Factor** — это количество полных копий (реплик) каждого блока данных на разных физических узлах. Например, replication factor 3 означает, что каждый блок хранится на трёх разных серверах. Если один или даже два узла выходят из строя, данные остаются доступны на третьем. Это критично для надёжности: потеря одного сервера не приводит к потере данных.

**Consistency Hashing** — это алгоритм для распределения блоков данных по узлам storage кластера, который минимизирует перемещение блоков при добавлении нового узла. Вместо пересчета всех блоков, новый узел получает только свою справедливую долю (примерно 1/N). Это позволяет облегчить масштабирование системы хранения: добавление нового сервера не вызывает массивного переброса данных.

**Durability** — это гарантия, что данные не будут потеряны даже при сбоях оборудования, отказе нескольких узлов или даже потере целого дата центра. AWS S3 гарантирует durability 11 "девяток" (99.999999999%), что означает примерно 1 потеря в 100 миллионов лет. Durability достигается через replication блоков в разные регионы и периодическое восстановление.

**Deduplication** — это обнаружение и удаление дублирующихся блоков данных на уровне хранилища. Если два файла содержат одинаковые 5 МБ блока, система хранит блок только один раз и создаёт ссылку на него из обоих файлов. Deduplication экономит огромное количество места, особенно для резервных копий и архивов, где много одинаковых данных, и может сэкономить 50-70% места.

**Sharding** — это горизонтальное разделение файлов по разным block storage кластерам по какому-то ключу (например, по хешу файла или по диапазону). Вместо хранения всех данных на одном кластере, данные распределяются по нескольким кластерам. Это позволяет масштабировать систему на петабайты и более, распределяя нагрузку равномерно.

**Lazy Loading** — это отложенная загрузка метаданных или данных по требованию, вместо предварительной загрузки всего сразу. Например, когда пользователь заходит в папку с 10,000 файлов, система сразу загружает только метаданные первых 100 файлов, остальные загружаются при скроллинге. Lazy loading оптимизирует память, улучшает скорость запуска и обеспечивает лучший пользовательский опыт.

**Write-Once Read-Many (WORM)** — это модель хранения данных, в которой файл может быть записан (загружен) один раз, а затем может быть прочитан и скопирован столько раз, сколько нужно, но не может быть изменён или удалён. Это упрощает consistency и обеспечивает immutability (неизменяемость), что критично для логов, архивов и документов для соответствия нормативным требованиям.

---

## Вопросы и разборы

### 1. Как спроектировать архитектуру системы хранения файлов?

**Зачем спрашивают.** Базовый вопрос про file storage требует понимания компонентов (upload/download сервисы, метаданные, block storage) и их взаимодействия. Интервьюер проверяет способность структурировать сложную систему.

**Короткий ответ.** Система состоит из нескольких слоёв: API gateway маршрутизирует запросы, upload сервис обрабатывает загрузку с чанкированием, download сервис отдаёт файлы через CDN, metadata сервис хранит информацию в БД, block storage кластер хранит фактические данные. Все компоненты независимо масштабируются.

**Детальный разбор.**

**Требования и оценка:**
```
Функциональные требования:
- Upload files (любой размер, до TB)
- Download files
- Delete files
- List files (с pagination)
- Metadata management (опционально: versioning, access control)

Нефункциональные требования:
- High durability: 99.999999999% (11 nines)
- High availability: 99.99%
- Low latency downloads (via CDN)
- Scale: петабайты хранилища
- Support для больших файлов (multi-GB)

Capacity estimation (пример):
- 500M пользователей
- 100 файлов на юзера в среднем
- 50B файлов всего
- Средний размер: 1MB
- Total storage: 50 PB

- 10M uploads/day → ~115 QPS
- 100M downloads/day → ~1,150 QPS (mostly via CDN)
- Metadata: 50B files × 500 bytes = 25TB
```

**Архитектура высокого уровня:**
```
                            ┌──────────────────┐
                            │   Load Balancer  │
                            └────────┬─────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
              ┌─────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
              │   Upload   │  │  Download   │  │  Metadata  │
              │   Service  │  │   Service   │  │   Service  │
              └─────┬──────┘  └──────┬──────┘  └─────┬──────┘
                    │                │              │
              ┌─────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
              │  Chunk     │  │    CDN      │  │ Metadata   │
              │  Service   │  │   Network   │  │ Database   │
              └─────┬──────┘  └──────┬──────┘  └─────┬──────┘
                    │                │              │
                    └────────┬───────┘──────────────┘
                             │
                      ┌──────▼──────┐
                      │   Block     │
                      │   Storage   │
                      │   Cluster   │
                      │ (replicated)│
                      └─────────────┘
```

**Компоненты системы:**

| Компонент | Ответственность | Примеры |
|-----------|-----------------|---------|
| **API Gateway** | Маршрутизация, auth, rate limiting | nginx, HAProxy |
| **Upload Service** | Инициация upload, управление чанками | Custom service |
| **Download Service** | Генерация presigned URLs, редирект на CDN | Custom service |
| **Metadata Service** | CRUD операции с метаданными файла | PostgreSQL + cache |
| **Block Storage** | Физическое хранилище данных | S3-compatible, custom |
| **CDN** | Edge caching для быстрого скачивания | Cloudflare, Akamai |

**Пример.**

```python
# Upload flow
POST /api/v1/files/upload/init
{
    "filename": "video.mp4",
    "size": 1073741824,  # 1GB
    "content_type": "video/mp4"
}

Response:
{
    "upload_id": "upload_xyz",
    "chunk_size": 5242880,  # 5MB chunks
    "presigned_urls": [
        {"part": 1, "url": "https://storage.../part1?sig=..."},
        {"part": 2, "url": "https://storage.../part2?sig=..."}
    ]
}

# Client uploads chunks directly to storage
PUT {presigned_url}
Body: <binary chunk data>

# Complete upload
POST /api/v1/files/upload/complete
{
    "upload_id": "upload_xyz",
    "parts": [
        {"part": 1, "etag": "etag1"},
        {"part": 2, "etag": "etag2"}
    ]
}

Response:
{
    "file_id": "file_123",
    "url": "https://cdn.example.com/file_123"
}

# Download redirects to CDN
GET /api/v1/files/file_123
Response: 302 Redirect to
https://cdn.example.com/file_123?sig=...&expires=...
```

**Типичные ошибки.**
- Забыть про metadata bottleneck — нужна кэширование и шардирование для быстрого доступа.
- Синхронная репликация для всех операций — убивает latency, нужна асинхронная для блоков.
- Не использовать presigned URLs — security risk, большая нагрузка на API.
- Одна точка отказа (один block storage кластер) — нужны replica в других datacenter.

**На интервью.**
- Расскажи про separation of concerns: upload/download/metadata сервисы независимы.
- Упомяни presigned URLs как критическую оптимизацию (client пишет напрямую в storage).
- Follow-up: «Как масштабировать систему на petabytes?» — sharding по user_id или hash(file_id), replication.

---

### 2. Как реализовать multipart upload с чанкированием?

**Зачем спрашивают.** Загрузка больших файлов — критическая операция. Multipart upload решает проблемы: resumability, параллелизм, robustness. Интервьюер проверяет понимание state management и failure handling.

**Короткий ответ.** Разбиваем файл на чанки (5-100MB), каждый загружается отдельно с помощью presigned URL. Upload сервис отслеживает статус каждого чанка в Redis. После загрузки всех чанков выполняем verify (checksums) и combine. State хранится временно и удаляется после complete или timeout.

**Детальный разбор.**

**Multipart upload flow:**
```
Client                Upload Service           Block Storage         Redis
  │                        │                        │                 │
  │──── init upload ──────→ │                        │                 │
  │                        │── allocate chunks ────→ │                 │
  │                        │── store state ────────────────────────→  │
  │                        │                        │           {      │
  │◄─── presigned URLs ────│                        │     upload: {    │
  │      (part 1,2,3)      │                        │       chunks: [] │
  │                        │                        │     }            │
  │────── chunk 1 ───────────────────────────────→  │                 │
  │  (parallel)            │                        │                 │
  │────── chunk 2 ───────────────────────────────→  │                 │
  │  (parallel)            │                        │                 │
  │────── chunk 3 ───────────────────────────────→  │                 │
  │                        │                        │                 │
  │                        │◄──── etag 1,2,3 ──────│                 │
  │                        │                        │                 │
  │── complete upload ────→│                        │                 │
  │   (with parts list)    │── verify checksums ────────────────────→│
  │                        │◄─── check state ──────────────────────→│
  │                        │── combine chunks ────→ │                 │
  │                        │── create metadata ────→ │                 │
  │                        │── cleanup state ──────────────────────→│
  │◄──── file_id ──────────│                        │                 │
```

**Upload state machine:**
```
┌──────────────┐
│   INITIATED  │  presigned URLs ready
│ upload_id    │
│ chunk_size   │
└──────┬───────┘
       │
       │ upload chunk 1
       ▼
┌──────────────────┐
│  CHUNK_RECEIVED  │  chunk 1 stored
│  parts_received: │  increment counter
│  1/3             │
└──────┬───────────┘
       │
       │ upload chunks 2, 3
       ▼
┌──────────────────┐
│   ALL_RECEIVED   │  all chunks ready
│  parts_received: │
│  3/3             │
└──────┬───────────┘
       │
       │ complete upload + verify
       ▼
┌──────────────────┐
│    VERIFIED      │
│  checksums OK    │
└──────┬───────────┘
       │
       │ combine + commit metadata
       ▼
┌──────────────────┐
│   COMPLETED      │ file_id assigned
│  file_id: 123    │
└──────────────────┘
```

**Пример реализации:**

```python
import asyncio
import hashlib
from enum import Enum
from dataclasses import dataclass

CHUNK_SIZE = 5 * 1024 * 1024  # 5MB

@dataclass
class UploadState:
    upload_id: str
    filename: str
    total_size: int
    num_chunks: int
    uploaded_chunks: set  # {0, 1, 2}
    status: str  # "initiated", "in_progress", "completed"
    created_at: datetime

class UploadService:
    async def initiate_upload(self, filename: str, file_size: int) -> dict:
        """Step 1: Client инициирует upload"""
        upload_id = generate_uuid()
        num_chunks = math.ceil(file_size / CHUNK_SIZE)

        # Store state in Redis
        state = {
            "filename": filename,
            "size": file_size,
            "num_chunks": num_chunks,
            "uploaded_chunks": [],
            "status": "initiated",
            "created_at": datetime.now().isoformat()
        }
        await redis.hset(f"upload:{upload_id}", mapping=state)
        await redis.expire(f"upload:{upload_id}", 86400)  # TTL 24 hours

        # Generate presigned URLs for each chunk
        presigned_urls = []
        for i in range(num_chunks):
            url = self.generate_presigned_url(
                upload_id=upload_id,
                part_number=i + 1,
                expires_in=3600  # 1 hour
            )
            presigned_urls.append({
                "part": i + 1,
                "url": url,
                "size_bytes": CHUNK_SIZE
            })

        return {
            "upload_id": upload_id,
            "chunk_size": CHUNK_SIZE,
            "num_chunks": num_chunks,
            "presigned_urls": presigned_urls
        }

    async def record_chunk(self, upload_id: str, part_num: int, etag: str) -> None:
        """Step 2: Block storage вызывает callback после загрузки чанка"""
        # Increment counter
        await redis.lpush(f"upload:{upload_id}:chunks", f"{part_num}:{etag}")
        await redis.hset(f"upload:{upload_id}", "last_chunk_time",
                        datetime.now().isoformat())

    async def complete_upload(self, upload_id: str, parts: list) -> dict:
        """Step 3: Client завершает upload"""
        # Get state
        state = await redis.hgetall(f"upload:{upload_id}")
        if not state:
            raise UploadNotFoundError(upload_id)

        num_chunks = int(state["num_chunks"])

        # Verify all parts received
        if len(parts) != num_chunks:
            raise IncompleteUploadError(
                f"Expected {num_chunks} parts, got {len(parts)}"
            )

        # Verify checksums
        for part in parts:
            stored_etag = await redis.get(
                f"upload:{upload_id}:part:{part['part']}:etag"
            )
            if stored_etag != part['etag']:
                raise ChecksumMismatchError(
                    f"Part {part['part']} checksum mismatch"
                )

        # Create metadata and commit
        file_id = generate_uuid()
        await self.create_file_metadata(
            file_id=file_id,
            filename=state["filename"],
            size=int(state["size"]),
            upload_id=upload_id
        )

        # Cleanup
        await self.cleanup_upload_state(upload_id)

        return {"file_id": file_id}

    async def cleanup_upload_state(self, upload_id: str) -> None:
        """Clean Redis keys after upload completes or timeout"""
        keys_pattern = f"upload:{upload_id}*"
        async for key in redis.scan_iter(match=keys_pattern):
            await redis.delete(key)

    def generate_presigned_url(self, upload_id: str, part_number: int,
                              expires_in: int) -> str:
        """Generate signed URL to block storage"""
        # Signature includes: upload_id, part_number, expires_at
        payload = {
            "upload_id": upload_id,
            "part": part_number,
            "expires": int(time.time()) + expires_in
        }
        signature = hmac.new(
            SECRET_KEY.encode(),
            json.dumps(payload).encode(),
            hashlib.sha256
        ).hexdigest()

        return (f"https://storage.example.com/upload"
                f"?{urlencode(payload)}"
                f"&sig={signature}")
```

**Типичные ошибки.**
- Не сохранять upload state — невозможно resume при сбое сети.
- Генерировать все presigned URLs с большим timeout — security risk, URL может быть перехвачена.
- Не устанавливать TTL на Redis state — orphaned uploads будут занимать память.
- Не verify checksums при complete — corrupted chunks попадут в production.

**На интервью.**
- Объясни, почему presigned URLs генерируются на сервере (не отправляются в client).
- Упомяни resumability — что произойдёт если disconnect в середине upload.
- Follow-up: «Как обеспечить atomicity?» — либо все чанки успешны, либо весь upload откатывается.

---

### 3. Как организовать хранение и получение файлов через CDN?

**Зачем спрашивают.** CDN критична для low-latency downloads на scale. Нужно понимать: edge caching, cache invalidation, presigned URLs, signed cookies. Интервьюер проверяет знание trade-offs между performance и cost.

**Короткий ответ.** Файлы хранятся в origin (block storage), запросы идут на CDN edge nodes, которые кэшируют на первый запрос. Для security используем presigned URLs с expiration. Cache invalidation делается через versioning в URL или через explicit purge API. Для большого трафика используем geo-distribution и load balancing между edge nodes.

**Детальный разбор.**

**CDN architecture:**
```
User A (USA)              User B (EU)              User C (APAC)
     │                         │                        │
     │ GET /file/123           │ GET /file/123          │ GET /file/123
     │                         │                        │
     ▼                         ▼                        ▼
  ┌─────────────┐          ┌─────────────┐        ┌─────────────┐
  │ CDN Edge NA │          │ CDN Edge EU │        │ CDN Edge SG │
  │  (cached)   │          │  (cached)   │        │  (cached)   │
  └──────┬──────┘          └──────┬──────┘        └──────┬──────┘
         │                        │                      │
         │  miss (or TTL expired) │ miss                 │ miss
         ▼                        ▼                      ▼
         └────────────┬───────────┴──────────────────────┘
                      │
              ┌───────▼────────┐
              │  Origin (S3)   │
              │  /storage/     │
              │  file_123.mp4  │
              │ (source)       │
              └────────────────┘
```

**Download flow с CDN:**
```
Client                     API                    CDN                    Origin
  │                        │                      │                       │
  │─ GET /file/id ────────→ │                      │                       │
  │                        │                      │                       │
  │                        │ generate presigned   │                       │
  │                        │ URL with exp time    │                       │
  │◄─ 302 redirect to CDN─ │                      │                       │
  │  /file/123?sig=...     │                      │                       │
  │  &exp=1704067200       │                      │                       │
  │                        │                      │                       │
  │─ GET /file/123?... ───────────────────────→  │                       │
  │                        │                      │ (HIT from cache)      │
  │◄─ 200 file content ─────────────────────────│                       │
  │                        │                      │                       │
  │                        │                      │ (next request)        │
  │                        │                      │                       │
  │─ GET /file/123?... ───────────────────────→  │                       │
  │                        │                      │ cache expired or      │
  │                        │                      │ explicit purge        │
  │                        │                      │                       │
  │                        │                      │─ GET /storage/... ───→ │
  │                        │                      │                      │ get
  │                        │                      │◄─ file content ──────│
  │◄─ 200 file content ────────────────────────│                       │
  │                        │                      │                       │
```

**Presigned URL с expiration:**
```python
class DownloadService:
    async def get_download_url(self, file_id: str, user_id: str) -> str:
        """Generate presigned URL to CDN"""
        # Check access control
        file = await metadata_db.get_file(file_id)
        if not self.has_access(file, user_id):
            raise AccessDeniedError()

        # Generate URL with signature
        expires_at = int(time.time()) + 3600  # 1 hour

        payload = {
            "file_id": file_id,
            "user_id": user_id,
            "expires": expires_at
        }

        # HMAC signature (secret on server, not exposed to client)
        signature = hmac.new(
            SECRET_KEY.encode(),
            json.dumps(payload, sort_keys=True).encode(),
            hashlib.sha256
        ).hexdigest()

        # Redirect to CDN
        cdn_url = (f"https://cdn.example.com/file/{file_id}"
                  f"?user_id={user_id}"
                  f"&expires={expires_at}"
                  f"&sig={signature}")

        return cdn_url

class CDNEdge:
    async def handle_request(self, file_id: str, user_id: str,
                            expires: int, sig: str) -> bytes:
        """CDN edge validates signature and serves file"""
        # Check expiration
        if int(time.time()) > expires:
            raise ExpiredURLError()

        # Verify signature
        payload = {
            "file_id": file_id,
            "user_id": user_id,
            "expires": expires
        }
        expected_sig = hmac.new(
            SECRET_KEY.encode(),
            json.dumps(payload, sort_keys=True).encode(),
            hashlib.sha256
        ).hexdigest()

        if not hmac.compare_digest(sig, expected_sig):
            raise InvalidSignatureError()

        # Check cache
        cache_key = f"file:{file_id}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        # Cache miss: fetch from origin
        origin_data = await self.fetch_from_origin(file_id)

        # Store in edge cache with TTL
        await self.cache.set(cache_key, origin_data,
                            ttl=86400)  # 24 hours

        return origin_data
```

**Cache control strategies:**
```
Strategy 1: URL versioning (immutable URLs)
- GET /file/id/v1/content
- GET /file/id/v2/content (новая версия = новый URL)
- TTL: very high (365 days)
- Pros: simple, no purge needed
- Cons: version explosion

Strategy 2: Content-hash in URL (content-addressable)
- GET /file/hash/abc123def456
- URL = hash(content), изменение content = новый URL
- TTL: very high (365 days)
- Pros: automatic dedup via cache
- Cons: requires hash computation

Strategy 3: Short TTL + explicit purge
- GET /file/id/current
- TTL: 1 hour or 24 hours
- Explicit purge when file updated
- Pros: simple, flexible
- Cons: invalidation overhead

Strategy 4: Signed cookies (for sessions)
- Cookie signature valid for N hours
- Works for multiple URLs in same domain
- TTL: session duration
- Pros: user-level control
- Cons: more complex setup
```

**Типичные ошибки.**
- Кэшировать без проверки expiration — старые файлы в CDN после update.
- Слишком короткий TTL — высокая нагрузка на origin, медленные downloads.
- Слишком длинный TTL — невозможно быстро обновить файл.
- Не использовать signature — anyone с URL может скачать файл, даже если access revoked.

**На интервью.**
- Объясни про presigned URLs: подпись содержит file_id, user_id, expires.
- Упомяни несколько стратегий cache invalidation и их trade-offs.
- Follow-up: «Как обрабатывать случай когда access отозван?» — проверить permissions в API перед redirect на CDN, или store access list в Redis.

---

### 4. Как реализовать дедупликацию файлов?

**Зачем спрашивают.** Дедупликация экономит петабайты storage. Нужно понимать: content-addressable storage, hash collisions, garbage collection для orphaned блоков. Интервьюер проверяет умение оптимизировать costs.

**Короткий ответ.** Вычисляем SHA-256 hash содержимого файла (или блока). Если hash уже существует — создаём reference на существующий block вместо дублирования. Для каждого блока храним reference count. При удалении файла уменьшаем count, и когда count == 0, удаляем физический блок. Garbage collection периодически убирает orphaned blocks.

**Детальный разбор.**

**Content-addressable storage (CAS):**
```
File A: ABCDEFGH
  hash = SHA256(ABCDEFGH) = abc123

File B: ABCDEFGH (identical content)
  hash = SHA256(ABCDEFGH) = abc123 (same!)

Storage:
Block "abc123" → ABCDEFGH (stored once)
FileA → reference to abc123
FileB → reference to abc123

Savings: 1 block size (no duplication)
```

**Block-level deduplication (chunking):**
```
File A: [Block1, Block2, Block3, Block4]
File B: [Block1, Block5, Block3, Block6]

Hashes:
Block1 = hash("data1") = h1
Block2 = hash("data2") = h2
Block3 = hash("data3") = h3
Block4 = hash("data4") = h4
Block5 = hash("data5") = h5
Block6 = hash("data6") = h6

Storage:
h1 → data1 (ref_count: 2, used by A and B)
h2 → data2 (ref_count: 1, used by A)
h3 → data3 (ref_count: 2, used by A and B)
h4 → data4 (ref_count: 1, used by A)
h5 → data5 (ref_count: 1, used by B)
h6 → data6 (ref_count: 1, used by B)

Savings: 2 blocks (h1, h3 shared)
Total stored: 4 blocks instead of 8
```

**Деуп со скользящим окном (для больших файлов):**
```
File content: ABCDEFGHIJKLMNOPQRSTUVWXYZ (26 bytes)

Chunk size: 4 bytes, use content-defined chunking
┌─────┐
│ABCD│ hash1
└────┬┘
     │
   ┌─┴──┐
   │BCDE│ hash2
   └────┬┘
        │
        ┌─────┐
        │CDEF│ hash3
        └────┬┘
             │
            ┌──────┐
            │DEFGH│ hash4
            └──────┤
                   ...

Advantage: если файл изменится (добавится символ),
то старые блоки остаются с теми же хешами,
новые блоки создаются с новыми хешами.
Это лучше, чем фиксированное chunking.
```

**Реализация с reference counting:**

```python
class DeduplicationService:
    async def store_file_with_dedup(self, file_id: str,
                                    file_content: bytes,
                                    user_id: str) -> None:
        """
        Store file using content-based deduplication.
        Multiple users can reference same content.
        """
        # Split file into chunks
        chunks = self.split_into_chunks(file_content)

        # Process each chunk
        block_ids = []
        for chunk in chunks:
            block_id = await self.store_chunk(chunk)
            block_ids.append(block_id)

        # Create file metadata referencing blocks
        await self.create_file_metadata(
            file_id=file_id,
            block_ids=block_ids,
            user_id=user_id
        )

    async def store_chunk(self, chunk_data: bytes) -> str:
        """Store chunk and update reference count"""
        # Compute content hash
        content_hash = hashlib.sha256(chunk_data).hexdigest()

        # Check if content already exists
        existing_block = await self.block_storage.get_block_by_hash(
            content_hash
        )

        if existing_block:
            # Content exists: just increment ref count
            await redis.incr(f"block:{existing_block['block_id']}:refcount")
            return existing_block['block_id']

        # New content: store block
        block_id = generate_uuid()
        await self.block_storage.store_block(
            block_id=block_id,
            data=chunk_data,
            content_hash=content_hash
        )

        # Initialize reference count
        await redis.set(f"block:{block_id}:refcount", 1)

        # Store hash-to-block mapping for lookup
        await redis.set(f"hash:{content_hash}:block_id", block_id)

        return block_id

    async def delete_file(self, file_id: str) -> None:
        """Delete file and decrement block references"""
        # Get file's blocks
        file = await self.metadata_db.get_file(file_id)

        for block_id in file['block_ids']:
            # Decrement reference count
            refcount = await redis.decr(f"block:{block_id}:refcount")

            if refcount <= 0:
                # No more references: schedule block for deletion
                await self.schedule_block_deletion(
                    block_id=block_id,
                    delay=timedelta(hours=24)  # grace period
                )

        # Delete file metadata
        await self.metadata_db.delete_file(file_id)

    def split_into_chunks(self, data: bytes) -> list:
        """
        Split file into chunks.
        Options:
        1. Fixed-size: 4MB chunks
        2. Content-defined: use rolling hash to find boundaries
        """
        # Fixed-size approach (simpler)
        chunk_size = 4 * 1024 * 1024
        chunks = [
            data[i:i+chunk_size]
            for i in range(0, len(data), chunk_size)
        ]
        return chunks

        # Content-defined approach (more efficient)
        # Uses rolling hash to find natural chunk boundaries
        # See LBFS (Low-Bandwidth File System) for details
```

**Garbage collection для orphaned blocks:**
```python
class GarbageCollector:
    async def run_gc(self):
        """
        Scan for blocks with refcount == 0
        and delete them after grace period.
        """
        # Get all blocks scheduled for deletion
        deletion_queue = await self.get_deletion_queue()

        now = datetime.now()
        for deletion_entry in deletion_queue:
            block_id = deletion_entry['block_id']
            scheduled_at = deletion_entry['scheduled_at']
            grace_period = timedelta(hours=24)

            if now - scheduled_at > grace_period:
                # Grace period passed: safe to delete
                await self.block_storage.delete_block(block_id)

                # Clean up metadata
                await redis.delete(f"block:{block_id}:refcount")

                # Log for monitoring
                self.metrics.increment("gc_blocks_deleted")
```

**Типичные ошибки.**
- Забыть про garbage collection — orphaned blocks займут storage.
- Не использовать grace period перед удалением — if bug, unrecoverable data loss.
- Hash collision не обрабатывается — при SHA-256 вероятность пренебрежима, но нужна verification.
- Не track reference count точно — потеря блоков или неправильное удаление.

**На интервью.**
- Объясни, как работает content-addressable storage: hash = id.
- Упомяни reference counting и garbage collection.
- Follow-up: «Что произойдёт при hash collision?» — вероятность в 2^-256, но можно добавить verification (store actual content hash в метаданные).

---

### 5. Как управлять метаданными файлов в больших масштабах?

**Зачем спрашивают.** Метаданные — горячая точка (hot path) при файловых операциях. Нужно обрабатывать миллионы запросов в секунду. Интервьюер проверяет знание: sharding, caching, indexing.

**Короткий ответ.** Метаданные хранятся в PostgreSQL с индексами по owner_id и file_id. Для scale используем горизонтальный шардинг (hash(user_id) % num_shards). Hot metadata кэшируется в Redis (TTL 1 hour). Каждый шард реплицируется для high availability. Index optimization критична для быстрого lookup.

**Детальный разбор.**

**Metadata schema:**
```sql
-- Primary metadata table
CREATE TABLE files (
    id              UUID PRIMARY KEY,
    owner_id        UUID NOT NULL,
    filename        VARCHAR(255) NOT NULL,
    size            BIGINT NOT NULL,
    content_type    VARCHAR(100),
    content_hash    VARCHAR(64),  -- SHA256 для dedup
    storage_path    VARCHAR(500),  -- путь в block storage
    status          VARCHAR(20),  -- uploading, active, deleted
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP,
    deleted_at      TIMESTAMP,  -- soft delete

    INDEX idx_owner (owner_id),
    INDEX idx_status (status),
    INDEX idx_created (created_at)
);

-- File versioning (optional)
CREATE TABLE file_versions (
    id              UUID PRIMARY KEY,
    file_id         UUID REFERENCES files(id),
    version_num     INT NOT NULL,
    size            BIGINT,
    content_hash    VARCHAR(64),
    storage_path    VARCHAR(500),
    created_at      TIMESTAMP,
    created_by      UUID,

    UNIQUE (file_id, version_num)
);

-- Access control (optional)
CREATE TABLE file_permissions (
    id              UUID PRIMARY KEY,
    file_id         UUID REFERENCES files(id),
    user_id         UUID,
    permission      VARCHAR(20),  -- read, write, admin
    granted_at      TIMESTAMP,
    granted_by      UUID
);
```

**Sharding strategy для metadata:**
```
Hash-based sharding по owner_id:

User A (owner_id = user123)
  hash(user123) = h % 16 = 3
  → Shard 3 (PostgreSQL database 3)

User B (owner_id = user456)
  hash(user456) = h % 16 = 7
  → Shard 7 (PostgreSQL database 7)

Advantages:
- Load распределена равномерно
- Все файлы пользователя на одном шарде
- Easy to compute: hash(owner_id) % num_shards

Disadvantages:
- Нельзя сделать global list of all files
- Rebalancing сложная, если изменить num_shards

Topology:
┌──────────────┐
│ Shard Router │ (routes query to correct shard)
└──────┬───────┘
       │
   ┌───┴────┬────────┬─────────────┬────┐
   │        │        │             │    │
┌──▼──┐ ┌──▼──┐ ┌───▼──┐  ... ┌──▼──┐
│ S0  │ │ S1  │ │ S2   │      │ S15 │
│ PG  │ │ PG  │ │ PG   │      │ PG  │
└─────┘ └─────┘ └──────┘      └─────┘
```

**Caching layer:**
```python
class MetadataService:
    async def get_file(self, file_id: str, owner_id: str) -> dict:
        """Get file metadata with caching"""
        # Check cache first
        cache_key = f"file:{file_id}:{owner_id}"
        cached = await redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # Cache miss: query database
        shard_id = hash(owner_id) % NUM_SHARDS
        db = self.get_shard_connection(shard_id)

        file = await db.query("""
            SELECT * FROM files
            WHERE id = %s AND owner_id = %s
        """, (file_id, owner_id))

        if not file:
            raise FileNotFoundError(file_id)

        # Store in cache (TTL 1 hour)
        await redis.setex(
            cache_key,
            3600,
            json.dumps(file)
        )

        return file

    async def create_file(self, owner_id: str, filename: str,
                         size: int) -> dict:
        """Create file metadata"""
        file_id = generate_uuid()

        # Shard by owner_id
        shard_id = hash(owner_id) % NUM_SHARDS
        db = self.get_shard_connection(shard_id)

        await db.execute("""
            INSERT INTO files
            (id, owner_id, filename, size, status, created_at)
            VALUES (%s, %s, %s, %s, 'active', NOW())
        """, (file_id, owner_id, filename, size))

        # Invalidate cache for owner's file list
        await redis.delete(f"files:{owner_id}:list")

        return {"file_id": file_id}

    async def update_file(self, file_id: str, owner_id: str,
                         updates: dict) -> None:
        """Update file metadata"""
        shard_id = hash(owner_id) % NUM_SHARDS
        db = self.get_shard_connection(shard_id)

        # Build update query
        set_clause = ", ".join([f"{k}=%s" for k in updates.keys()])
        values = list(updates.values()) + [file_id, owner_id]

        await db.execute(f"""
            UPDATE files
            SET {set_clause}, updated_at=NOW()
            WHERE id = %s AND owner_id = %s
        """, values)

        # Invalidate caches
        await redis.delete(f"file:{file_id}:{owner_id}")
        await redis.delete(f"files:{owner_id}:list")

class ListFilesService:
    async def list_files(self, owner_id: str, limit: int = 20,
                        offset: int = 0) -> dict:
        """List user's files with pagination"""
        # Query shard for this owner
        shard_id = hash(owner_id) % NUM_SHARDS
        db = self.get_shard_connection(shard_id)

        files = await db.query("""
            SELECT id, filename, size, created_at, updated_at
            FROM files
            WHERE owner_id = %s AND deleted_at IS NULL
            ORDER BY created_at DESC
            LIMIT %s OFFSET %s
        """, (owner_id, limit, offset))

        total_count = await db.query("""
            SELECT COUNT(*) as count
            FROM files
            WHERE owner_id = %s AND deleted_at IS NULL
        """, (owner_id,))

        return {
            "files": files,
            "total": total_count[0]['count'],
            "offset": offset,
            "limit": limit
        }
```

**Типичные ошибки.**
- Кэширование без инвалидации — стейл data после update.
- Индекс только по file_id, нет по owner_id — slow list queries.
- Шардирование по file_id — все файлы одного пользователя распределены, slow list.
- Не replicating shards — single shard failure = unavailable.

**На интервью.**
- Объясни sharding strategy: hash(owner_id) % num_shards.
- Упомяни о кэшировании с TTL и инвалидации.
- Follow-up: «Как обрабатывать глобальный поиск по всем файлам?» — отдельный Elasticsearch индекс для search.

---

### 6. Как реализовать репликацию и обеспечить высокую durability?

**Зачем спрашивают.** Durability критична для облачного хранилища (11 nines = 99.999999999%). Нужно понимать: replication vs erasure coding, sync vs async, cross-datacenter replication. Интервьюер проверяет trade-offs между reliability и performance.

**Короткий ответ.** Каждый блок реплицируется 3 раза в разные datacenters. Запись на primary синхронна, репликация на вторичные асинхронна. При сбое primary читаем с replicas. Альтернатива: erasure coding (более экономно по storage, но медленнее на recover). Для additional durability используем bit rot detection (periodic checksumming).

**Детальный разбор.**

**3-way replication архитектура:**
```
Block write:

Client                Primary DC            Replica DC1            Replica DC2
  │                      │                     │                       │
  │─ write block ────────→ │                     │                       │
  │  (sync)               │                     │                       │
  │                       │ store locally       │                       │
  │                       │ (success)           │                       │
  │                       │                     │                       │
  │◄─ ack (success) ──────│                     │                       │
  │                       │                     │                       │
  │                       │─ replicate async ──→ │                       │
  │                       │─ replicate async ──────────────────────────→ │
  │                       │                     │                       │
  │                       │                     │ (background task)     │
  │                       │                     │ store replica         │
  │                       │                     │ ack to primary        │
  │                       │                     │                       │


Block read:

Client                Primary DC            Replica DC1            Replica DC2
  │                      │                     │                       │
  │─ read block ─────────→ │                     │                       │
  │                       │ read locally        │                       │
  │                       │ (success)           │                       │
  │◄─ data ───────────────│                     │                       │
  │                       │                     │                       │

(If primary fails:)

Client                Primary DC            Replica DC1            Replica DC2
  │                   (DOWN)                  │                       │
  │─ read block ─────────X                    │                       │
  │ (timeout)           X                      │                       │
  │                       X                    │                       │
  │─ read block (retry) ──────────────────────→ │                       │
  │                       │                     │ read locally          │
  │◄─ data ────────────────────────────────────│ (success)             │
```

**Consistency при async replication:**
```
Timeline:

T0: Block stored on Primary
T1: Block replicated to DC1 (async)
T2: Block replicated to DC2 (async)
T3: Client confirms write (ACK sent at T0)

Window: T0 to T1-T2 → block exists only on Primary
        → if Primary fails, data might be lost!

Solution: Use quorum writes
- Require ack from 2/3 replicas before ACK to client
- Ensures data is replicated before client sees success
- Trade-off: higher latency
```

**Реализация:**

```python
class ReplicationService:
    def __init__(self, datacenters: list):
        self.primary_dc = datacenters[0]  # DC1
        self.replica_dcs = datacenters[1:]  # DC2, DC3
        self.replication_factor = 3

    async def write_block(self, block_id: str, data: bytes) -> None:
        """
        Write with async replication.
        Primary ACKs immediately, replicas update asynchronously.
        """
        # Write to primary synchronously
        try:
            await self.primary_dc.store(block_id, data)
        except Exception as e:
            raise WriteError(f"Primary write failed: {e}")

        # Schedule async replication
        for dc in self.replica_dcs:
            # Fire-and-forget, but log failures
            asyncio.create_task(self.replicate_to_dc(dc, block_id, data))

        # Metrics
        self.metrics.increment("blocks_written")

    async def replicate_to_dc(self, dc, block_id: str, data: bytes) -> None:
        """
        Replicate to single datacenter with retry logic.
        """
        max_retries = 3
        backoff = 1  # seconds

        for attempt in range(max_retries):
            try:
                await dc.store(block_id, data)
                self.metrics.increment(f"replication_success_{dc.name}")
                return

            except Exception as e:
                if attempt == max_retries - 1:
                    # Final attempt failed: queue for async retry
                    await self.queue_replication_retry(dc, block_id, data)
                    self.metrics.increment(f"replication_failed_{dc.name}")
                    return

                # Exponential backoff
                await asyncio.sleep(backoff)
                backoff *= 2

    async def read_block(self, block_id: str) -> bytes:
        """
        Read from available replica.
        Prefer primary, fallback to replicas.
        """
        # Try primary first
        try:
            data = await self.primary_dc.read(block_id)
            self.metrics.increment("read_primary_hit")
            return data
        except Exception:
            self.metrics.increment("read_primary_miss")

        # Fallback to replicas (any order)
        for dc in self.replica_dcs:
            try:
                data = await dc.read(block_id)
                self.metrics.increment(f"read_replica_hit_{dc.name}")
                return data
            except Exception:
                continue

        # All copies failed
        raise BlockNotFoundError(f"Block {block_id} not found in any replica")

    async def verify_replication(self, block_id: str) -> dict:
        """
        Periodic check: verify all replicas have the block.
        Used for monitoring and repair.
        """
        status = {"block_id": block_id, "replicas": {}}

        # Check primary
        try:
            data = await self.primary_dc.read(block_id)
            status["replicas"]["primary"] = "ok"
        except Exception as e:
            status["replicas"]["primary"] = f"missing: {e}"

        # Check secondaries
        for dc in self.replica_dcs:
            try:
                data = await dc.read(block_id)
                status["replicas"][dc.name] = "ok"
            except Exception as e:
                status["replicas"][dc.name] = f"missing: {e}"

        return status
```

**Erasure coding альтернатива:**
```
Reed-Solomon (6,4) encoding:

Original blocks:     [D1, D2, D3, D4]  (4 data blocks)

Encode:
P1, P2 = encode(D1, D2, D3, D4)  (2 parity blocks)

Store across 6 nodes:
Node 1: D1
Node 2: D2
Node 3: D3
Node 4: D4
Node 5: P1
Node 6: P2

Properties:
- Can recover from any 2 node failures
- Storage overhead: 50% (vs 200% for 3x replication)
- Decode cost: higher CPU (reconstruction needed)
- Read cost: need 4/6 blocks to reconstruct

Trade-off table:
                    Storage     CPU      Latency   Recovery
3-way replication   200%        Low      Fast      Fast
Erasure Coding      50%         High     Slower    Slower
  (6,4)
```

**Типичные ошибки.**
- Синхронная репликация везде — убивает latency.
- Async replication без quorum — data loss при primary failure.
- Не проверять replication status — orphaned replicas, inconsistency.
- Replication только в одном datacenter — не защита от datacenter failure.

**На интервью.**
- Объясни sync primary write + async replica updates.
- Упомяни data loss window и как его минимизировать (quorum writes).
- Follow-up: «Как сделать replication synchronous без убийства latency?» — parallel writes, local replication.

---

### 7. Как обрабатывать консистентность и reconciliation при распределённом хранении?

**Зачем спрашивают.** Распределённые системы требуют eventual consistency. Нужно обрабатывать: divergence между репликами, bit rot, missing blocks. Интервьюер проверяет понимание real-world issues (это сложнее, чем теория).

**Короткий ответ.** Используем версионирование (version number + timestamp) и checksums (CRC/SHA) на каждом блоке. Periodic scrubbing (background verification) проверяет целостность. Если replica не совпадает с primary — repair через re-replication. Для metadata используем distributed consensus (Paxos/Raft) или optimistic locking.

**Детальный разбор.**

**Consistency issues в distributed storage:**
```
Issue 1: Replica divergence
┌─────────────────────┐
│ Primary DC          │
│ Block: v1 = content │
└─────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
┌────────┐         ┌────────┐
│DC1     │         │DC2     │
│v1      │ (slow)  │v0      │  ← diverged!
└────────┘         └────────┘

Issue 2: Bit rot (corruption during storage)
Block stored: [1010101...]
Read from disk: [1010011...] ← bit flipped!
Silent corruption (no error, wrong data)

Issue 3: Missing replica
┌────────────┐
│Primary: OK │
└──────┬─────┘
       │
    ┌──┴────┬─────────┐
    │       │         │
    ▼       ▼         ▼
┌────┐ ┌────┐    (down)
│OK  │ │OK  │    1/3 replicas missing
└────┘ └────┘    → reduced durability!
```

**Versioning и checksums:**
```python
class BlockHeader:
    """
    Metadata stored with each block.
    Helps detect corruption and divergence.
    """
    version: int  # monotonically increasing
    timestamp: int  # when written (UTC seconds)
    size: int  # block size in bytes
    content_hash: str  # SHA-256 of content
    compression: str  # "none", "gzip", "zstd"

    # Binary layout (64 bytes):
    # 0-3: magic (0xDEADBEEF)
    # 4-7: version
    # 8-15: timestamp
    # 16-23: size
    # 24-55: content_hash (32 bytes SHA-256)
    # 56-63: header_crc32
```

**Scrubbing процесс:**
```python
class ScrubService:
    """
    Periodic background process that verifies block integrity.
    Detects corruption and divergence.
    """

    async def run_scrub_job(self):
        """
        Scan all blocks and verify against replicas.
        Run periodically (e.g., once per week).
        """
        batch_size = 1000
        offset = 0

        while True:
            # Get blocks to scrub
            blocks = await self.storage.scan_blocks(
                offset=offset,
                limit=batch_size
            )

            if not blocks:
                break

            # Verify each block
            for block in blocks:
                result = await self.verify_block(block['id'])

                if result['status'] == 'corruption_detected':
                    # Trigger repair
                    await self.repair_block(block['id'])
                    self.metrics.increment("scrub_corruptions_found")

                elif result['status'] == 'replica_mismatch':
                    # Replicate from primary
                    await self.sync_replica(block['id'])
                    self.metrics.increment("scrub_mismatches_found")

            offset += batch_size

    async def verify_block(self, block_id: str) -> dict:
        """
        Verify a single block across all replicas.
        Check: hash, size, version consistency.
        """
        # Read from primary
        primary = await self.primary_dc.read_block_with_header(block_id)

        # Read from replicas
        replicas = []
        for dc in self.replica_dcs:
            try:
                replica = await dc.read_block_with_header(block_id)
                replicas.append((dc.name, replica))
            except Exception as e:
                replicas.append((dc.name, None))

        # Verify primary integrity
        primary_hash = hashlib.sha256(primary['data']).hexdigest()
        if primary_hash != primary['header']['content_hash']:
            return {
                "status": "corruption_detected",
                "location": "primary",
                "block_id": block_id
            }

        # Verify replicas match primary
        for replica_dc, replica in replicas:
            if replica is None:
                # Replica missing or unreachable
                return {
                    "status": "replica_mismatch",
                    "location": replica_dc,
                    "issue": "missing",
                    "block_id": block_id
                }

            replica_hash = hashlib.sha256(replica['data']).hexdigest()
            if replica_hash != primary['header']['content_hash']:
                # Mismatch
                return {
                    "status": "replica_mismatch",
                    "location": replica_dc,
                    "issue": "diverged",
                    "block_id": block_id
                }

        return {
            "status": "ok",
            "block_id": block_id
        }

    async def repair_block(self, block_id: str) -> None:
        """
        Repair corrupted block by re-replicating from primary.
        """
        # Read primary
        data = await self.primary_dc.read(block_id)

        # Re-write to all replicas
        for dc in self.replica_dcs:
            try:
                await dc.store(block_id, data)
            except Exception as e:
                # Queue for retry
                await self.queue_repair_retry(dc, block_id, data)
```

**Metadata consistency (для распределённого metadata):**
```python
class DistributedMetadataService:
    """
    If metadata replicated across nodes, ensure consistency.
    Use either:
    1. Distributed consensus (Paxos/Raft)
    2. Optimistic locking + versioning
    """

    async def update_file_optimistic(self, file_id: str,
                                     expected_version: int,
                                     updates: dict) -> bool:
        """
        Optimistic locking: only update if version matches.
        Prevents lost updates.
        """
        query = f"""
        UPDATE files
        SET {', '.join(f"{k}=%s" for k in updates.keys())},
            version = version + 1,
            updated_at = NOW()
        WHERE id = %s AND version = %s
        """

        values = list(updates.values()) + [file_id, expected_version]

        result = await self.db.execute(query, values)

        if result.affected_rows == 0:
            # Version mismatch: conflict!
            raise VersionConflictError(
                f"Expected version {expected_version}, "
                f"but file has newer version"
            )

        return True

    async def read_file_with_version(self, file_id: str) -> dict:
        """
        Read file including current version.
        Client must include this version when updating.
        """
        file = await self.db.query(
            "SELECT * FROM files WHERE id = %s",
            (file_id,)
        )

        return {
            "file": file,
            "version": file['version']
        }
```

**Типичные ошибки.**
- Не проверять bit rot — corruption не detected, wrong data served.
- Scrubbing запускается слишком редко или вообще не запускается.
- Repair не автоматический —누누누 ручных операций.
- Metadata не версионировано — lost updates при concurrent modifications.

**На интервью.**
- Объясни bit rot detection через checksumming.
- Упомяни scrubbing как background process.
- Follow-up: «Как быстро detected corruption?» — зависит от scrubbing schedule (weekly = up to 7 days).

---

### 8. Как масштабировать систему на петабайты?

**Зачем спрашивают.** От миллиардов файлов к петабайтам — качественные изменения в архитектуре. Нужно понимать: sharding, federation, geo-distribution. Интервьюер проверяет ability to think at massive scale.

**Короткий ответ.** На петабайты масштабируемся через горизонтальный шардинг: разделяем файлы на множество block storage кластеров (каждый может быть несколько PB). Metadata шардируется по owner_id. CDN распределяется географически (каждый регион имеет edge nodes). Каждый шард реплицируется для durability. Глобальная балансировка трафика через DNS/anycast.

**Детальный разбор.**

**Sharding архитектура на петабайты:**
```
Namespace federation:

Users 0-100M → Shard 0
  - Primary: DC1
  - Replicas: DC2, DC3
  - Storage: 500TB block storage cluster

Users 100M-200M → Shard 1
  - Primary: DC4
  - Replicas: DC5, DC6
  - Storage: 500TB block storage cluster

Users 200M-300M → Shard 2
  ...

Users 400M-500M → Shard 4
  - Primary: DC13
  - Replicas: DC14, DC15
  - Storage: 500TB block storage cluster

Total: 5 shards × 500TB = 2.5 PB minimum
(With replication: 2.5 × 3 = 7.5 PB actual used)

Load balancing:
┌─────────────────┐
│   DNS / Anycast │ (global load balancer)
└────────┬────────┘
         │
   ┌─────┴────────┬─────────┬──────────┐
   │              │         │          │
┌──▼───┐      ┌──▼───┐ ┌──▼───┐  ┌──▼───┐
│Shard │      │Shard │ │Shard │  │Shard │
│  0   │      │  1   │ │  2   │  │  3   │
└──────┘      └──────┘ └──────┘  └──────┘
(DC1/2/3)     (DC4/5/6) ...       (DC...)
```

**Metadata routing:**
```python
class ShardRouter:
    """
    Route requests to correct metadata shard.
    """

    NUM_METADATA_SHARDS = 256

    def get_metadata_shard(self, owner_id: str) -> int:
        """
        Deterministic hashing: owner → shard number.
        """
        hash_value = int(hashlib.md5(owner_id.encode()).hexdigest(), 16)
        shard_id = hash_value % self.NUM_METADATA_SHARDS
        return shard_id

    async def get_file_metadata(self, file_id: str, owner_id: str) -> dict:
        """
        Route to correct shard and query.
        """
        shard_id = self.get_metadata_shard(owner_id)
        db = self.metadata_shards[shard_id]

        file = await db.query(
            "SELECT * FROM files WHERE id = %s AND owner_id = %s",
            (file_id, owner_id)
        )

        return file

class BlockStorageRouter:
    """
    Route file blocks to correct storage shard.
    """

    NUM_STORAGE_SHARDS = 32

    def get_storage_shard(self, owner_id: str) -> int:
        """
        Owner → storage shard (may differ from metadata shard).
        Allows independent scaling.
        """
        hash_value = int(hashlib.md5(owner_id.encode()).hexdigest(), 16)
        shard_id = hash_value % self.NUM_STORAGE_SHARDS
        return shard_id

    async def write_block(self, owner_id: str, block_id: str,
                         data: bytes) -> None:
        """
        Route block to storage shard.
        """
        shard_id = self.get_storage_shard(owner_id)
        storage_cluster = self.storage_shards[shard_id]

        await storage_cluster.store_block(block_id, data)

    async def read_block(self, owner_id: str, block_id: str) -> bytes:
        """
        Read from storage shard.
        """
        shard_id = self.get_storage_shard(owner_id)
        storage_cluster = self.storage_shards[shard_id]

        return await storage_cluster.read_block(block_id)
```

**Geo-distributed CDN:**
```
Global CDN topology:

┌─────────────────────┐
│  Global DNS (Anycast)│
└──────────┬──────────┘
           │
      ┌────┴─────────┬──────────┬─────────┐
      │              │          │         │
      ▼              ▼          ▼         ▼
┌──────────────┐ ┌──────────┐ ┌──────┐ ┌──────┐
│ Americas POP │ │ Europe   │ │APAC  │ │ MENA │
│(São Paulo,   │ │POP       │ │POP   │ │ POP  │
│NY, LA)       │ │(London,  │ │(SG,  │ │(DXB) │
│              │ │Frankfurt)│ │Tokyo)│ │      │
└──────┬───────┘ └────┬─────┘ └──┬───┘ └──┬───┘
       │              │          │        │
       └──────────────┼──────────┼────────┘
                      │
              ┌───────▼──────────┐
              │ Regional Origins │
              │ (edge-pull CDN)  │
              └────────┬─────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
      ┌──────────┐┌──────────┐┌──────────┐
      │Primary DC││Replica1  ││Replica2  │
      │  (US)    ││(Europe)  ││(APAC)    │
      └────┬─────┘└──────┬───┘└────┬─────┘
           │             │         │
           └─────┬───────┴────┬────┘
                 │            │
          ┌──────▼──┐  ┌──────▼──┐
          │Block    │  │Block    │
          │Storage  │  │Storage  │
          │S0-S15   │  │S0-S15   │
          └─────────┘  └─────────┘
```

**Capacity planning для петабайта:**
```
Assumptions:
- 500M users
- 100 files per user = 50B files
- Average file: 1MB
- Total: 50 PB raw

With 3x replication:
50 PB × 3 = 150 PB total capacity needed

Hardware planning (estimate):
- 1 storage node: 10TB disk
- 1 node redundancy factor: 3x
- Effective per node: 3TB usable
- Nodes needed: 150 PB / 3 TB = 50,000 nodes

Distributed across DCs:
- 16 datacenters
- ~3,000 nodes per DC
- ~30 racks per DC (100 nodes per rack)

But with deduplication (common in real systems):
- Actual might be 50 PB × 1.5 (50% unique) = 75 PB
- After 3x replication: 225 PB
- Nodes: 225 / 3 = 75,000 nodes

Bandwidth requirements:
- Upload: 10M files/day × 1MB = 10TB/day
- = 115 Mbps average (peaks 500 Mbps)

- Download: 100M files/day × 1MB = 100TB/day
- = 1.15 Gbps average (peaks 5 Gbps)
- Most via CDN (99%), so origin sees ~11 Mbps on average

Metadata requirements:
- 50B files × 500 bytes = 25TB
- With versioning, permissions: 50TB
- Distributed across 256 shards
- 50TB / 256 = 195GB per shard (manageable)
```

**Типичные ошибки.**
- Забыть про replication overhead при capacity planning (3x, не 1x).
- Все шарды на одном datacenters — not geographically distributed.
- Не планировать growth — capacity quickly exhausted.
- Uneven shard distribution — некоторые overloaded.

**На интервью.**
- Расскажи про sharding по owner_id и как масштабировать на петабайты.
- Упомяни geo-distributed replicas для low latency и high availability.
- Follow-up: «Как переехать с 4 шардов на 8 шардов?» — consistent hashing или migration job.

---

### 9. Как обеспечить надёжные загрузки большых файлов с resumability?

**Зачем спрашивают.** Resumable uploads критичны для UX (bad networks). Нужно обрабатывать: network interruptions, client crashes, timeout recovery. Интервьюер проверяет robustness thinking.

**Короткий ответ.** Разбиваем файл на чанки, каждый имеет ID. При disconnect сохраняем progress (какие чанки загружены). При retry клиент запрашивает статус upload и загружает только недостающие чанки. Timeout для state в Redis (обычно 7 дней). CRC проверка каждого чанка исключает corrupted uploads.

**Детальный разбор.**

**Resumable upload flow:**
```
Attempt 1:
┌─────────┐
│ Client  │
└────┬────┘
     │
     │ init upload_id=XYZ
     ├──────────────────→ Upload Service
     │                      ├─ allocate chunks
     │                      └─ return presigned URLs
     │◄─────────────────────
     │ presigned_urls (1-10)
     │
     │ upload chunks 1-3 (parallel)
     │─────────────────→ Block Storage
     │  (chunk 1: 5MB)
     │  (chunk 2: 5MB)
     │  (chunk 3: 5MB)
     │
     │ Network fails!
     │ (only chunks 1-3 uploaded)
     X

(Network down for 30 minutes)

Attempt 2 (resume):
┌─────────┐
│ Client  │
└────┬────┘
     │
     │ query status upload_id=XYZ
     ├──────────────────→ Upload Service
     │                      ├─ lookup state
     │                      └─ return progress
     │◄─────────────────────
     │ {chunks_received: [1,2,3],
     │  chunks_needed: [4,5,6,7,8,9,10]}
     │
     │ upload chunks 4-10 (only missing ones)
     │─────────────────→ Block Storage
     │
     │ all chunks received
     │
     │ complete upload
     ├──────────────────→ Upload Service
     │                      ├─ verify all chunks
     │                      └─ combine, create metadata
     │◄─────────────────────
     │ {file_id: ABC123}
```

**State management:**
```python
class ResumableUploadService:
    """
    Manages long-running uploads with resume capability.
    """

    UPLOAD_STATE_TTL = 7 * 86400  # 7 days
    CHUNK_SIZE = 5 * 1024 * 1024  # 5MB

    async def initiate_upload(self, user_id: str, filename: str,
                             file_size: int) -> dict:
        """
        Step 1: Initialize resumable upload.
        """
        upload_id = generate_uuid()
        num_chunks = math.ceil(file_size / self.CHUNK_SIZE)

        # Store upload metadata with TTL
        upload_state = {
            "user_id": user_id,
            "filename": filename,
            "file_size": file_size,
            "num_chunks": num_chunks,
            "uploaded_chunks": [],  # array of uploaded chunk numbers
            "chunk_etags": {},  # {chunk_num: etag}
            "status": "initiated",
            "created_at": datetime.utcnow().isoformat(),
            "last_activity": datetime.utcnow().isoformat()
        }

        # Store in Redis with expiration
        await redis.hset(
            f"upload:{upload_id}",
            mapping=upload_state
        )
        await redis.expire(f"upload:{upload_id}", self.UPLOAD_STATE_TTL)

        # Generate presigned URLs for all chunks
        presigned_urls = []
        for chunk_num in range(num_chunks):
            url = self.generate_presigned_upload_url(
                upload_id=upload_id,
                chunk_num=chunk_num,
                expires_in=3600  # 1 hour per chunk
            )
            presigned_urls.append({
                "chunk": chunk_num,
                "url": url
            })

        return {
            "upload_id": upload_id,
            "num_chunks": num_chunks,
            "chunk_size": self.CHUNK_SIZE,
            "presigned_urls": presigned_urls
        }

    async def get_upload_status(self, upload_id: str,
                               user_id: str) -> dict:
        """
        Step 1b: Client can query progress to resume.
        """
        # Retrieve state
        state = await redis.hgetall(f"upload:{upload_id}")

        if not state:
            raise UploadNotFoundError(f"Upload {upload_id} expired or invalid")

        # Verify user owns this upload
        if state['user_id'] != user_id:
            raise AccessDeniedError("Not your upload")

        # Extract progress
        uploaded_chunks = json.loads(state.get('uploaded_chunks', '[]'))
        num_chunks = int(state['num_chunks'])

        # Calculate which chunks still needed
        chunks_needed = [
            i for i in range(num_chunks)
            if i not in uploaded_chunks
        ]

        return {
            "upload_id": upload_id,
            "num_chunks": num_chunks,
            "uploaded_chunks": uploaded_chunks,
            "chunks_needed": chunks_needed,
            "status": state['status'],
            "created_at": state['created_at']
        }

    async def upload_chunk(self, upload_id: str, chunk_num: int,
                          chunk_data: bytes, etag: str) -> None:
        """
        Step 2: Upload single chunk.
        Client can call multiple times (same chunk = idempotent).
        """
        # Verify upload exists
        state = await redis.hgetall(f"upload:{upload_id}")
        if not state:
            raise UploadNotFoundError(upload_id)

        # Verify chunk number valid
        num_chunks = int(state['num_chunks'])
        if chunk_num >= num_chunks:
            raise InvalidChunkError(f"Chunk {chunk_num} > {num_chunks}")

        # Check if chunk already uploaded
        stored_etag = await redis.hget(
            f"upload:{upload_id}:chunk_etags",
            str(chunk_num)
        )

        if stored_etag:
            if stored_etag == etag:
                # Same chunk: idempotent
                return {"status": "already_uploaded", "chunk": chunk_num}
            else:
                # Different etag: conflict (corrupted? retried with different data?)
                raise ChecksumMismatchError(
                    f"Chunk {chunk_num} already exists with different etag"
                )

        # Verify chunk checksum before storing
        chunk_crc = hashlib.sha256(chunk_data).hexdigest()
        # (In practice, client sends CRC, we verify it matches)

        # Store chunk in block storage
        block_id = f"{upload_id}:chunk:{chunk_num}"
        await self.block_storage.store_chunk(block_id, chunk_data)

        # Record in upload state
        await redis.hset(
            f"upload:{upload_id}:chunk_etags",
            chunk_num,
            etag
        )

        # Update uploaded_chunks list
        uploaded = json.loads(await redis.hget(f"upload:{upload_id}",
                                               "uploaded_chunks") or "[]")
        if chunk_num not in uploaded:
            uploaded.append(chunk_num)
            await redis.hset(
                f"upload:{upload_id}",
                "uploaded_chunks",
                json.dumps(uploaded)
            )

        # Update last activity
        await redis.hset(
            f"upload:{upload_id}",
            "last_activity",
            datetime.utcnow().isoformat()
        )

        return {"status": "uploaded", "chunk": chunk_num}

    async def complete_upload(self, upload_id: str, user_id: str) -> dict:
        """
        Step 3: Finalize upload when all chunks received.
        """
        # Get state
        state = await redis.hgetall(f"upload:{upload_id}")
        if not state:
            raise UploadNotFoundError(upload_id)

        if state['user_id'] != user_id:
            raise AccessDeniedError()

        # Verify all chunks present
        uploaded_chunks = json.loads(state['uploaded_chunks'])
        num_chunks = int(state['num_chunks'])

        if len(uploaded_chunks) != num_chunks:
            missing = [i for i in range(num_chunks)
                      if i not in uploaded_chunks]
            raise IncompleteUploadError(
                f"Missing chunks: {missing[:10]}"  # show first 10
            )

        # Combine chunks
        file_id = generate_uuid()
        combined_data = b""

        for chunk_num in range(num_chunks):
            block_id = f"{upload_id}:chunk:{chunk_num}"
            chunk_data = await self.block_storage.read_chunk(block_id)
            combined_data += chunk_data

        # Verify final content
        expected_size = int(state['file_size'])
        if len(combined_data) != expected_size:
            raise SizeValidationError(
                f"Expected {expected_size}, got {len(combined_data)}"
            )

        # Store combined file
        await self.block_storage.store_file(file_id, combined_data)

        # Create metadata
        await self.create_file_metadata(
            file_id=file_id,
            owner_id=user_id,
            filename=state['filename'],
            size=expected_size
        )

        # Cleanup
        await self.cleanup_upload_state(upload_id)

        return {"file_id": file_id}

    async def cleanup_upload_state(self, upload_id: str) -> None:
        """
        Delete all temporary upload state.
        """
        keys = await redis.keys(f"upload:{upload_id}*")
        if keys:
            await redis.delete(*keys)
```

**Client-side resumability:**
```javascript
class ResumableUploader {
    constructor(file, options = {}) {
        this.file = file;
        this.chunkSize = options.chunkSize || 5 * 1024 * 1024;
        this.uploadId = options.uploadId;
        this.maxRetries = options.maxRetries || 3;
    }

    async start() {
        // Step 1: Initiate or resume
        if (!this.uploadId) {
            const init = await this.initiate();
            this.uploadId = init.upload_id;
            this.presignedUrls = init.presigned_urls;
        } else {
            const status = await this.getStatus();
            this.uploadedChunks = status.uploaded_chunks;
            this.chunksNeeded = status.chunks_needed;
        }

        // Step 2: Upload chunks in parallel
        const promises = this.chunksNeeded.map(
            chunkNum => this.uploadChunk(chunkNum)
        );

        await Promise.all(promises);

        // Step 3: Complete
        const result = await this.complete();
        return result;
    }

    async uploadChunk(chunkNum) {
        const start = chunkNum * this.chunkSize;
        const end = Math.min(start + this.chunkSize, this.file.size);
        const chunk = this.file.slice(start, end);

        // Calculate etag for this chunk
        const etag = await this.calculateEtag(chunk);

        // Get presigned URL
        const url = this.presignedUrls.find(
            p => p.chunk === chunkNum
        )?.url;

        if (!url) {
            throw new Error(`No presigned URL for chunk ${chunkNum}`);
        }

        // Retry logic
        for (let attempt = 0; attempt < this.maxRetries; attempt++) {
            try {
                await fetch(url, {
                    method: 'PUT',
                    body: chunk,
                    headers: {
                        'X-Chunk-Num': chunkNum.toString(),
                        'X-Chunk-Etag': etag
                    }
                });

                console.log(`Chunk ${chunkNum} uploaded`);
                return;
            } catch (error) {
                if (attempt === this.maxRetries - 1) {
                    throw error;
                }

                // Exponential backoff
                const delay = Math.pow(2, attempt) * 1000;
                await new Promise(r => setTimeout(r, delay));
            }
        }
    }

    async calculateEtag(chunk) {
        const buffer = await chunk.arrayBuffer();
        const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
        const hashArray = Array.from(new Uint8Array(hashBuffer));
        return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    }
}

// Usage:
const uploader = new ResumableUploader(file, {
    uploadId: localStorage.getItem('currentUploadId')
});

// Periodically save upload ID to localStorage
window.addEventListener('beforeunload', () => {
    localStorage.setItem('currentUploadId', uploader.uploadId);
});

await uploader.start();
localStorage.removeItem('currentUploadId');
```

**Типичные ошибки.**
- Слишком коротк TTL на upload state — resume невозможен после перезагрузки.
- Не check idempotency (same chunk uploaded twice) — duplicate data или corruption.
- Не verify checksum chunks — corrupted chunks попадут в файл.
- Генерировать все presigned URLs в initiate — долго для больших файлов.

**На интервью.**
- Объясни resumable upload: состояние в Redis, query status перед retry.
- Упомяни idempotency: повторная загрузка same chunk = безопасна.
- Follow-up: «Как обрабатывать случай где клиент загрузил chunk, но сервер crash?» — chunk stored в block storage, но не marked as received; при retry клиент перепошлёт, идемпотент.

---

### 10. Как оптимизировать систему по costs и performance?

**Зачем спрашивают.** File storage на петабайты — дорого. Нужно балансировать: deduplication (CPU), compression (latency), erasure coding (complexity), tiering (complexity). Интервьюер проверяет practical optimization mindset.

**Короткий ответ.** Используем многоуровневый подход: hot data (last 30 дней) хранится на быстром SSD с 3x replication, warm data (30-180 дней) на HDD с erasure coding, cold data (>180 дней) архивируется на glacier-like storage. Compression на cold tier. Деуп на горячие данные. Metrics-driven optimization — monitor access patterns, storage costs, bandwidth.

**Детальный разбор.**

**Cost optimization strategies:**
```
Strategy 1: Tiered storage
┌────────────────────────────────────────┐
│ Hot tier (SSD)                         │
│ - 30-day retention                     │
│ - 3x replication                       │
│ - Fast reads (< 100ms latency)         │
│ - Cost: $0.025/GB/month                │
│ - 2% of files (50B × 0.02 = 1B files) │
│ - Size: 1B × 1MB = 1 PB               │
│ - Total: 1 PB × 3x = 3 PB stored      │
│ - Cost: 3 PB × $0.025 = $75k/month    │
├────────────────────────────────────────┤
│ Warm tier (HDD)                        │
│ - 30-180 day retention                 │
│ - Erasure coding (6,4)                 │
│ - Medium latency (1-10s)               │
│ - Cost: $0.015/GB/month                │
│ - 18% of files (50B × 0.18 = 9B)      │
│ - Size: 9B × 1MB = 9 PB               │
│ - Total: 9 PB × 1.5x (EC) = 13.5 PB   │
│ - Cost: 13.5 PB × $0.015 = $202k/month│
├────────────────────────────────────────┤
│ Cold tier (Glacier)                    │
│ - >180 day retention                   │
│ - Erasure coding (20,10)               │
│ - High latency (hours)                 │
│ - Cost: $0.005/GB/month + retrieval    │
│ - 80% of files (50B × 0.80 = 40B)     │
│ - Size: 40B × 1MB = 40 PB             │
│ - Total: 40 PB × 1.2x (EC) = 48 PB    │
│ - Cost: 48 PB × $0.005 = $240k/month  │
└────────────────────────────────────────┘

Total cost: $75k + $202k + $240k = $517k/month
For 50 PB raw (vs. $1.25M if all on hot SSD)
Savings: 59% cost reduction!
```

**Compression for cold data:**
```python
class CompressionService:
    """
    Compress files before archiving to cold tier.
    """

    async def compress_for_archive(self, file_data: bytes) -> tuple:
        """
        Compress file, measure ratio.
        """
        import zstd

        # Use zstd (better compression ratio than gzip, faster)
        compressor = zstd.ZstdCompressor(level=10)
        compressed = compressor.compress(file_data)

        original_size = len(file_data)
        compressed_size = len(compressed)
        ratio = compressed_size / original_size

        # Only keep if compression ratio < 70%
        if ratio < 0.7:
            return compressed, "zstd"
        else:
            return file_data, "none"

    async def transition_to_cold(self, file_id: str):
        """
        Move file to cold tier with compression.
        """
        # Read from warm tier
        file_data = await self.warm_storage.read(file_id)

        # Compress
        compressed_data, codec = await self.compress_for_archive(file_data)

        # Store in cold (archive)
        await self.cold_storage.store(file_id, compressed_data, codec)

        # Delete from warm
        await self.warm_storage.delete(file_id)

        # Update metadata
        await self.metadata_db.update_file(
            file_id,
            {
                "storage_tier": "cold",
                "compression": codec,
                "compressed_size": len(compressed_data)
            }
        )
```

**Access pattern monitoring и optimization:**
```python
class AccessMonitor:
    """
    Track file access patterns to optimize tiering.
    """

    async def record_access(self, file_id: str):
        """
        Log file access for analysis.
        """
        now = datetime.utcnow()
        day = now.date().isoformat()

        # Increment access count for today
        await redis.incr(f"file_access:{file_id}:{day}")

    async def analyze_access_patterns(self):
        """
        Analyze access patterns and identify candidates for tiering.
        Run daily.
        """
        # Get all files not accessed in last 30 days
        cutoff_date = (datetime.utcnow() - timedelta(days=30)).date()

        warm_tier_candidates = []
        cold_tier_candidates = []

        async for file in self.metadata_db.iterate_all_files():
            file_id = file['id']
            created_at = file['created_at']
            last_accessed = await self.get_last_access(file_id)

            if last_accessed < cutoff_date:
                # Not accessed in 30 days → move to warm
                warm_tier_candidates.append({
                    "file_id": file_id,
                    "last_accessed": last_accessed,
                    "age_days": (now.date() - created_at.date()).days
                })

            if last_accessed < cutoff_date - timedelta(days=150):
                # Not accessed in 180 days → move to cold
                cold_tier_candidates.append({
                    "file_id": file_id,
                    "last_accessed": last_accessed,
                    "age_days": (now.date() - created_at.date()).days
                })

        # Transition files
        for candidate in warm_tier_candidates:
            await self.transition_to_warm(candidate['file_id'])

        for candidate in cold_tier_candidates:
            await self.transition_to_cold(candidate['file_id'])

        # Metrics
        self.metrics.gauge("warm_tier_transitions", len(warm_tier_candidates))
        self.metrics.gauge("cold_tier_transitions", len(cold_tier_candidates))
```

**Bandwidth optimization:**
```
Strategy: P2P transfers for large downloads

Normal flow:
User ─→ CDN edge ─→ Origin
(single path, bounded by edge-origin link)

P2P flow (for large files):
User A ─→ CDN edge → Download file
User B ─→ CDN edge → Ask "who has file X?"
         → redirect to User A's local copy
User B ← User A (P2P transfer, less CDN bandwidth)

Benefits:
- Reduce origin bandwidth
- Reduce CDN bandwidth
- Faster downloads for users with good connectivity
- Risk: user leaves before transfer completes

Trade-offs:
- Complexity (tracking who has what)
- Privacy (user sees other users' IPs)
- Security (spoofing, tampering)
- Only worth for very large files (> 100MB)
```

**Cost monitoring dashboard:**
```python
class CostDashboard:
    """
    Monitor costs in real-time.
    """

    async def compute_daily_costs(self):
        """
        Calculate storage costs by tier.
        """
        stats = {
            "hot_storage_gb": await redis.get("stats:hot_storage_gb"),
            "warm_storage_gb": await redis.get("stats:warm_storage_gb"),
            "cold_storage_gb": await redis.get("stats:cold_storage_gb"),
            "bandwidth_gb": await redis.get("stats:bandwidth_gb")
        }

        costs = {
            "hot": int(stats["hot_storage_gb"]) * 0.025 / 1024,  # per month
            "warm": int(stats["warm_storage_gb"]) * 0.015 / 1024,
            "cold": int(stats["cold_storage_gb"]) * 0.005 / 1024,
            "bandwidth": int(stats["bandwidth_gb"]) * 0.12 / 1024
        }

        costs["total"] = sum(costs.values())

        # Project monthly
        costs["monthly"] = costs["total"] * 30

        return costs

    async def optimization_recommendations(self):
        """
        Suggest cost optimizations.
        """
        recommendations = []

        stats = await self.compute_daily_costs()

        # Check if too much hot tier
        if stats["hot"] > stats["total"] * 0.3:
            recommendations.append({
                "issue": "High hot tier ratio",
                "suggestion": "Increase tiering cutoff (move to warm earlier)"
            })

        # Check dedup effectiveness
        dedup_ratio = await redis.get("stats:dedup_ratio")
        if float(dedup_ratio) < 0.2:
            recommendations.append({
                "issue": "Low deduplication",
                "suggestion": "Enable block-level dedup"
            })

        return recommendations
```

**Типичные ошибки.**
- Compression on hot tier — latency penalty outweighs storage savings.
- Не monitor access patterns — stale data stays in expensive tier.
- Tiering too complex — operational overhead eats savings.
- Erasure coding on hot tier — recovery latency unacceptable.

**На интервью.**
- Расскажи про tiered storage: hot/warm/cold с разными trade-offs.
- Упомяни compression для cold, и почему не для hot.
- Follow-up: «Как решить, какой файл переместить?» — access patterns + age.

---

## Итоговые рекомендации для интервью

### Что обязательно знать
1. **Architecture**: sharding, replication, caching
2. **Chunking & Upload**: multipart, resumability, checksums
3. **Deduplication**: hashing, reference counting, GC
4. **CDN & Download**: presigned URLs, edge caching
5. **Metadata**: indexing, sharding, consistency
6. **Consistency**: versioning, scrubbing, reconciliation
7. **Replication**: sync vs async, erasure coding
8. **Scale**: petabytes, shardsinged, geo-distribution
9. **Resumable uploads**: state tracking, idempotency
10. **Optimization**: tiering, compression, monitoring

### На собеседовании
- Начни с high-level design (API, components, data flow)
- Обсуди требования (functional & non-functional)
- Deep dive в интересующий интервьюера компонент
- Приводи примеры и диаграммы
- Обсуди trade-offs и альтернативы
- Упомяни monitoring и operations

### Типичные follow-up вопросы
- Как обрабатывать versioning?
- Как реализовать access control?
- Как обеспечить compliance (GDPR, data residency)?
- Как обновить файл конкурентно?
- Как мигрировать между хранилищами?
- Как обнаружить и исправить corruption?

---

## См. также

- [Шардирование баз данных](../01-databases/06-sharding.md) — распределение данных по узлам
- [Репликация](../02-distributed-systems/05-replication.md) — обеспечение отказоустойчивости
- [CDN и географическое кэширование](../03-protocols/03-cdn.md) — оптимизация доставки контента
- [Erasure coding](../02-distributed-systems/08-erasure-coding.md) — устойчивость к сбоям

---

[← Назад к списку тем](README.md)
