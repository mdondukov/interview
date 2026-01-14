# 08. File Storage / Cloud Storage

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Upload files (любой размер, до TB)
- Download files
- Delete files
- List files (with pagination)
- Metadata management
- Опционально: versioning, access control

### Нефункциональные
- High durability (99.999999999% — 11 nines)
- High availability (99.99%)
- Low latency for downloads (via CDN)
- Scale: petabytes of storage
- Support for large files (multi-GB)

---

## Capacity Estimation

```
Storage:
- 500M users
- 100 files per user average
- 50B total files
- Average file size: 1MB
- Total: 50 PB

Upload:
- 10M uploads/day
- Average file: 1MB
- Bandwidth: 10TB/day upload
- QPS: 10M / 86400 ≈ 115 QPS

Download:
- 100M downloads/day
- QPS: 1,150 QPS (mostly via CDN)
- Peak: 5,000 QPS

Metadata:
- 50B files × 500 bytes = 25TB metadata
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Client  │────▶│  Load Balancer  │────▶│   API Gateway    │
└──────────┘     └─────────────────┘     └────────┬─────────┘
                                                  │
                ┌─────────────────────────────────┼─────────────────────────────────┐
                │                                 │                                 │
         ┌──────▼──────┐                   ┌──────▼──────┐                   ┌──────▼──────┐
         │   Upload    │                   │  Download   │                   │  Metadata   │
         │   Service   │                   │   Service   │                   │   Service   │
         └──────┬──────┘                   └──────┬──────┘                   └──────┬──────┘
                │                                 │                                 │
         ┌──────▼──────┐                   ┌──────▼──────┐                   ┌──────▼──────┐
         │   Chunk     │                   │    CDN      │                   │ Metadata DB │
         │   Service   │                   │             │                   │ (PostgreSQL)│
         └──────┬──────┘                   └──────┬──────┘                   └─────────────┘
                │                                 │
                └─────────────────┬───────────────┘
                                  │
                           ┌──────▼──────┐
                           │   Block     │
                           │   Storage   │
                           │   Cluster   │
                           └─────────────┘
```

---

## API Design

### Upload
```json
// Step 1: Initiate upload
POST /api/v1/files/upload/init
Request:
{
    "filename": "video.mp4",
    "size": 1073741824,  // 1GB
    "content_type": "video/mp4",
    "checksum": "sha256:abc123..."
}

Response:
{
    "upload_id": "upload_xyz",
    "chunk_size": 5242880,  // 5MB
    "presigned_urls": [
        {"part": 1, "url": "https://storage.../part1?sig=..."},
        {"part": 2, "url": "https://storage.../part2?sig=..."}
    ]
}

// Step 2: Upload chunks (direct to storage)
PUT {presigned_url}
Body: <binary data>

// Step 3: Complete upload
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
    "url": "https://cdn.example.com/file_123",
    "size": 1073741824
}
```

### Download
```
GET /api/v1/files/{file_id}

Response: 302 Redirect to CDN URL
Location: https://cdn.example.com/file_123?sig=...&expires=...
```

### Metadata
```json
GET /api/v1/files/{file_id}/metadata

Response:
{
    "file_id": "file_123",
    "filename": "video.mp4",
    "size": 1073741824,
    "content_type": "video/mp4",
    "checksum": "sha256:abc123",
    "created_at": "2024-01-15T10:00:00Z",
    "owner_id": "user_456"
}
```

---

## Data Model

### File Metadata (PostgreSQL)
```sql
CREATE TABLE files (
    id              UUID PRIMARY KEY,
    owner_id        UUID NOT NULL,
    filename        VARCHAR(255) NOT NULL,
    size            BIGINT NOT NULL,
    content_type    VARCHAR(100),
    checksum        VARCHAR(100),
    storage_path    VARCHAR(500) NOT NULL,
    status          VARCHAR(20),  -- uploading, active, deleted
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP,
    deleted_at      TIMESTAMP,

    INDEX idx_owner (owner_id),
    INDEX idx_status (status)
);

CREATE TABLE file_versions (
    id              UUID PRIMARY KEY,
    file_id         UUID REFERENCES files(id),
    version_num     INT NOT NULL,
    size            BIGINT NOT NULL,
    checksum        VARCHAR(100),
    storage_path    VARCHAR(500) NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),

    UNIQUE (file_id, version_num)
);
```

### Block Storage Layout
```
/storage/
├── shard_001/
│   ├── block_00001
│   ├── block_00002
│   └── ...
├── shard_002/
│   └── ...
└── shard_N/

Block file format:
┌─────────────────────────────────────┐
│ Block Header (64 bytes)             │
│   - magic number                    │
│   - version                         │
│   - checksum                        │
│   - compression type                │
└─────────────────────────────────────┘
│ Data (variable)                     │
└─────────────────────────────────────┘
```

---

## Deep Dive

### 1. Chunking & Multipart Upload

```python
CHUNK_SIZE = 5 * 1024 * 1024  # 5MB

class ChunkUploadService:
    async def initiate_upload(self, file_size: int, filename: str) -> dict:
        upload_id = generate_upload_id()
        num_chunks = math.ceil(file_size / CHUNK_SIZE)

        # Store upload state
        await redis.hset(f"upload:{upload_id}", {
            "filename": filename,
            "size": file_size,
            "num_chunks": num_chunks,
            "uploaded_chunks": 0,
            "status": "in_progress"
        })

        # Generate presigned URLs for each chunk
        presigned_urls = []
        for i in range(num_chunks):
            url = self.generate_presigned_url(upload_id, i)
            presigned_urls.append({"part": i + 1, "url": url})

        return {
            "upload_id": upload_id,
            "presigned_urls": presigned_urls
        }

    async def complete_upload(self, upload_id: str, parts: list) -> dict:
        # Verify all chunks received
        upload_state = await redis.hgetall(f"upload:{upload_id}")

        if len(parts) != int(upload_state['num_chunks']):
            raise IncompleteUploadError()

        # Verify checksums
        for part in parts:
            expected = await redis.get(f"chunk:{upload_id}:{part['part']}:etag")
            if expected != part['etag']:
                raise ChecksumMismatchError()

        # Combine chunks into final file
        file_id = await self.combine_chunks(upload_id, parts)

        # Create metadata record
        await self.create_file_record(file_id, upload_state)

        # Cleanup
        await self.cleanup_upload_state(upload_id)

        return {"file_id": file_id}
```

### 2. Deduplication

```python
class DeduplicationService:
    """
    Content-addressable storage using file/chunk hashes
    """

    async def store_with_dedup(self, data: bytes, user_id: str) -> str:
        # Calculate content hash
        content_hash = hashlib.sha256(data).hexdigest()

        # Check if content already exists
        existing = await self.get_block_by_hash(content_hash)

        if existing:
            # Content exists - just create reference
            file_id = generate_file_id()
            await self.create_reference(file_id, existing['block_id'], user_id)
            return file_id

        # New content - store and create reference
        block_id = await self.store_block(data, content_hash)
        file_id = generate_file_id()
        await self.create_reference(file_id, block_id, user_id)

        return file_id

    async def delete_with_dedup(self, file_id: str):
        # Remove reference
        block_id = await self.get_block_id(file_id)
        await self.remove_reference(file_id)

        # Check if block still has references
        ref_count = await self.get_reference_count(block_id)

        if ref_count == 0:
            # No references - safe to delete block
            await self.delete_block(block_id)
```

**Block-level deduplication:**
```
File A: [Block1, Block2, Block3, Block4]
File B: [Block1, Block5, Block3, Block6]

Storage:
Block1 → stored once, referenced by A and B
Block2 → stored once, referenced by A only
Block3 → stored once, referenced by A and B
Block4 → stored once, referenced by A only
Block5 → stored once, referenced by B only
Block6 → stored once, referenced by B only

Savings: 2 blocks (Block1, Block3 shared)
```

### 3. Replication & Durability

```
┌────────────────────────────────────────────────────────────┐
│              Replication Strategy                           │
├────────────────────────────────────────────────────────────┤
│ 3-way replication across data centers:                     │
│                                                            │
│   DC1 (Primary)    DC2 (Replica)    DC3 (Replica)         │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│   │ Block A │ ───▶ │ Block A │      │ Block A │           │
│   └─────────┘  ↘   └─────────┘   ↗  └─────────┘           │
│                 ↘               ↗                          │
│                   async replication                        │
│                                                            │
│ Durability: 11 nines (99.999999999%)                      │
│ - Probability of all 3 copies failing simultaneously      │
└────────────────────────────────────────────────────────────┘
```

```python
class ReplicationService:
    def __init__(self, datacenters: list):
        self.primary_dc = datacenters[0]
        self.replica_dcs = datacenters[1:]
        self.replication_factor = 3

    async def store_block(self, block_id: str, data: bytes):
        # Write to primary synchronously
        await self.primary_dc.store(block_id, data)

        # Replicate to other DCs asynchronously
        for dc in self.replica_dcs:
            asyncio.create_task(self.replicate_to_dc(dc, block_id, data))

    async def replicate_to_dc(self, dc, block_id: str, data: bytes):
        max_retries = 3
        for attempt in range(max_retries):
            try:
                await dc.store(block_id, data)
                return
            except Exception as e:
                if attempt == max_retries - 1:
                    await self.queue_for_retry(dc, block_id)
                await asyncio.sleep(2 ** attempt)

    async def read_block(self, block_id: str) -> bytes:
        # Try primary first
        try:
            return await self.primary_dc.read(block_id)
        except Exception:
            pass

        # Fallback to replicas
        for dc in self.replica_dcs:
            try:
                return await dc.read(block_id)
            except Exception:
                continue

        raise BlockNotFoundError(block_id)
```

### 4. Erasure Coding (Alternative to Replication)

```
┌────────────────────────────────────────────────────────────┐
│                   Erasure Coding (Reed-Solomon)            │
├────────────────────────────────────────────────────────────┤
│ Original data: [D1, D2, D3, D4]                           │
│                                                            │
│ Encode with (6, 4) scheme:                                │
│   Data chunks:   [D1, D2, D3, D4]                         │
│   Parity chunks: [P1, P2]                                 │
│                                                            │
│ Storage: 6 chunks across 6 nodes                          │
│   Node 1: D1                                              │
│   Node 2: D2                                              │
│   Node 3: D3                                              │
│   Node 4: D4                                              │
│   Node 5: P1                                              │
│   Node 6: P2                                              │
│                                                            │
│ Can recover from any 2 node failures                      │
│ Storage overhead: 50% (vs 200% for 3x replication)        │
└────────────────────────────────────────────────────────────┘
```

```python
import reedsolo

class ErasureCodingService:
    def __init__(self, data_chunks: int = 4, parity_chunks: int = 2):
        self.data_chunks = data_chunks
        self.parity_chunks = parity_chunks
        self.rs = reedsolo.RSCodec(parity_chunks)

    def encode(self, data: bytes) -> list:
        # Split data into chunks
        chunk_size = len(data) // self.data_chunks
        data_chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]

        # Generate parity
        encoded = self.rs.encode(data)
        parity_data = encoded[len(data):]

        # Split parity
        parity_chunk_size = len(parity_data) // self.parity_chunks
        parity_chunks = [parity_data[i:i+parity_chunk_size]
                        for i in range(0, len(parity_data), parity_chunk_size)]

        return data_chunks + parity_chunks

    def decode(self, chunks: list, missing_indices: list) -> bytes:
        # Reconstruct from available chunks
        # Reed-Solomon can recover if missing <= parity_chunks
        # Note: missing_indices indicates which chunks are unavailable;
        # the RS library uses this internally to reconstruct the data
        return self.rs.decode(chunks, missing_indices)
```

### 5. Garbage Collection

```python
class GarbageCollector:
    """
    Mark-and-sweep GC for orphaned blocks
    """

    async def run_gc(self):
        # Phase 1: Mark - collect all referenced blocks
        referenced_blocks = set()

        async for file in self.iterate_all_files():
            for block_id in file.block_ids:
                referenced_blocks.add(block_id)

        # Phase 2: Sweep - find and delete unreferenced blocks
        async for block in self.iterate_all_blocks():
            if block.id not in referenced_blocks:
                # Check if block is old enough (avoid deleting in-progress uploads)
                if block.created_at < datetime.now() - timedelta(hours=24):
                    await self.delete_block(block.id)
                    self.metrics.increment("gc_blocks_deleted")

    async def run_gc_incremental(self):
        """
        Incremental GC using reference counting
        """
        # Process deletion queue
        async for deletion in self.deletion_queue.consume():
            block_id = deletion['block_id']

            # Decrement reference count
            new_count = await redis.decr(f"block:{block_id}:refs")

            if new_count <= 0:
                # Schedule for deletion after grace period
                await self.schedule_deletion(block_id, delay=timedelta(hours=24))
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Large file uploads | Multipart upload, resumable |
| Storage costs | Deduplication, erasure coding, tiering |
| Download latency | CDN, geographic distribution |
| Metadata bottleneck | Sharding, caching |
| Data integrity | Checksums, periodic scrubbing |

---

## Trade-offs

### Replication vs Erasure Coding

| Approach | Storage Overhead | CPU Overhead | Recovery Speed |
|----------|------------------|--------------|----------------|
| 3x Replication | 200% | Low | Fast |
| Erasure Coding (6,4) | 50% | High | Slower |

### Consistency Models

| Model | Durability | Latency | Complexity |
|-------|------------|---------|------------|
| Sync replication | Highest | Higher | Medium |
| Async replication | High | Lower | Medium |
| Erasure coding | High | Medium | High |

---

## На интервью

### Ключевые моменты
1. **Chunking** — multipart upload для больших файлов
2. **Deduplication** — content-addressable storage
3. **Durability** — replication vs erasure coding
4. **CDN integration** — для низкой latency downloads

### Типичные follow-up
- Как реализовать resumable uploads?
- Как обрабатывать concurrent updates одного файла?
- Как реализовать versioning?
- Как оптимизировать storage costs для cold data?
- Как обеспечить compliance (GDPR, data residency)?

---

## См. также

- [Шардирование баз данных](../06-databases/06-sharding.md) — распределение данных по узлам хранения
- [Репликация](../07-distributed-systems/05-replication-theory.md) — обеспечение отказоустойчивости и durability

---

[← Назад к списку тем](README.md)
