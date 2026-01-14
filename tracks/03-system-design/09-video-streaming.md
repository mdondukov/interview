# 09. Video Streaming (YouTube/Netflix)

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Video upload & processing
- Video playback (on-demand)
- Adaptive bitrate streaming
- Search & recommendations
- Live streaming (опционально)
- Comments, likes, subscriptions

### Нефункциональные
- High availability (99.99%)
- Low buffering (< 200ms startup)
- Global scale (billions of views/day)
- Support multiple devices/resolutions
- Cost-efficient storage & delivery

---

## Capacity Estimation

```
Users & Views:
- 2B total users, 500M DAU
- 5 videos watched per user per day
- 2.5B video views/day
- Average video length: 5 minutes
- Watch time: 12.5B minutes/day

Storage:
- 500 hours of video uploaded per minute (YouTube-scale)
- 720,000 hours/day = 30,000 days of video/day
- Encoded versions: 5 resolutions × 3 codecs = 15 versions
- 1 hour original ~= 3GB, encoded ~= 500MB avg
- Daily new storage: 720,000 × 500MB = 360TB/day

Bandwidth:
- Peak concurrent viewers: 100M
- Average bitrate: 5 Mbps
- Peak bandwidth: 500 Pbps (!!)
- CDN is absolutely critical
```

---

## High-Level Design

```
                                    Upload Flow
┌──────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Creator  │────▶│   Upload    │────▶│  Encoding   │────▶│   Storage   │
│          │     │   Service   │     │   Pipeline  │     │   (S3/GCS)  │
└──────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                │
                                    Playback Flow               │
┌──────────┐     ┌─────────────┐     ┌─────────────┐           │
│  Viewer  │◀────│     CDN     │◀────│   Origin    │◀──────────┘
│          │     │             │     │   Servers   │
└──────────┘     └─────────────┘     └─────────────┘

                                    Metadata Flow
┌──────────┐     ┌─────────────┐     ┌─────────────┐
│  Client  │────▶│ API Gateway │────▶│  Video      │
│          │     │             │     │  Metadata   │
└──────────┘     └─────────────┘     │  Service    │
                                     └──────┬──────┘
                                            │
                      ┌─────────────────────┼─────────────────────┐
                      │                     │                     │
               ┌──────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
               │  Metadata   │       │   Search    │       │   Recs      │
               │     DB      │       │   (Elastic) │       │   Service   │
               └─────────────┘       └─────────────┘       └─────────────┘
```

---

## API Design

### Upload
```json
// Step 1: Create video record
POST /api/v1/videos
Request:
{
    "title": "My Video",
    "description": "...",
    "tags": ["tech", "tutorial"]
}

Response:
{
    "video_id": "vid_abc123",
    "upload_url": "https://upload.example.com/vid_abc123?sig=...",
    "status": "pending_upload"
}

// Step 2: Upload video file
PUT {upload_url}
Content-Type: video/mp4
Body: <video binary>

// Step 3: Check processing status
GET /api/v1/videos/{video_id}/status

Response:
{
    "video_id": "vid_abc123",
    "status": "processing",
    "progress": 45,
    "available_qualities": ["360p", "480p"]
}
```

### Playback
```json
GET /api/v1/videos/{video_id}/manifest

Response:
{
    "video_id": "vid_abc123",
    "title": "My Video",
    "duration": 300,
    "thumbnail": "https://cdn.../thumb.jpg",
    "streams": {
        "hls": "https://cdn.../vid_abc123/master.m3u8",
        "dash": "https://cdn.../vid_abc123/manifest.mpd"
    }
}
```

### HLS Manifest (m3u8)
```
#EXTM3U
#EXT-X-VERSION:3

#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=1400000,RESOLUTION=854x480
480p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
720p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
```

---

## Data Model

### Video Metadata (PostgreSQL)
```sql
CREATE TABLE videos (
    id              UUID PRIMARY KEY,
    channel_id      UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    duration        INT,  -- seconds
    status          VARCHAR(20),  -- uploading, processing, published, removed
    visibility      VARCHAR(20),  -- public, private, unlisted
    view_count      BIGINT DEFAULT 0,
    like_count      BIGINT DEFAULT 0,
    published_at    TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_channel (channel_id),
    INDEX idx_status (status),
    INDEX idx_published (published_at)
);

CREATE TABLE video_encodings (
    id              UUID PRIMARY KEY,
    video_id        UUID REFERENCES videos(id),
    resolution      VARCHAR(10),  -- 360p, 720p, 1080p, 4k
    codec           VARCHAR(20),  -- h264, h265, vp9, av1
    bitrate         INT,
    storage_path    VARCHAR(500),
    file_size       BIGINT,
    status          VARCHAR(20),
    created_at      TIMESTAMP DEFAULT NOW(),

    UNIQUE (video_id, resolution, codec)
);
```

### View Analytics (Cassandra)
```sql
CREATE TABLE video_views (
    video_id    UUID,
    date        DATE,
    hour        INT,
    view_count  COUNTER,
    PRIMARY KEY ((video_id, date), hour)
);

CREATE TABLE user_watch_history (
    user_id         UUID,
    watched_at      TIMESTAMP,
    video_id        UUID,
    watch_duration  INT,
    PRIMARY KEY (user_id, watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);
```

---

## Deep Dive

### 1. Video Processing Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Video Processing Pipeline                         │
└──────────────────────────────────────────────────────────────────────┘

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│ Upload  │───▶│  Queue  │───▶│ Transcode│───▶│ Package │───▶│ Publish │
│         │    │ (Kafka) │    │ Workers  │    │ (HLS/   │    │ to CDN  │
└─────────┘    └─────────┘    └─────────┘    │  DASH)  │    └─────────┘
                                             └─────────┘

Transcoding outputs:
┌─────────────────────────────────────────────────┐
│ Resolution  │ Bitrate   │ Codec  │ Use Case    │
├─────────────────────────────────────────────────┤
│ 2160p (4K)  │ 15 Mbps   │ H.265  │ TV, Desktop │
│ 1080p       │ 5 Mbps    │ H.264  │ Desktop     │
│ 720p        │ 2.5 Mbps  │ H.264  │ Tablet      │
│ 480p        │ 1 Mbps    │ H.264  │ Mobile      │
│ 360p        │ 600 Kbps  │ H.264  │ Poor network│
│ 240p        │ 300 Kbps  │ H.264  │ Very poor   │
└─────────────────────────────────────────────────┘
```

```python
class VideoProcessor:
    async def process_video(self, video_id: str, source_path: str):
        # 1. Download original
        local_path = await self.download(source_path)

        # 2. Extract metadata
        metadata = await self.extract_metadata(local_path)

        # 3. Generate thumbnail
        thumbnail = await self.generate_thumbnail(local_path)
        await self.upload_thumbnail(video_id, thumbnail)

        # 4. Transcode to multiple resolutions
        resolutions = self.get_target_resolutions(metadata)

        for resolution in resolutions:
            encoded = await self.transcode(local_path, resolution)
            await self.upload_encoded(video_id, resolution, encoded)

            # Update status progressively
            await self.update_status(video_id, resolution, "available")

        # 5. Generate HLS/DASH manifests
        await self.generate_manifests(video_id, resolutions)

        # 6. Publish to CDN
        await self.publish_to_cdn(video_id)

    async def transcode(self, input_path: str, resolution: str) -> str:
        # FFmpeg command
        output_path = f"/tmp/{resolution}.mp4"

        cmd = [
            'ffmpeg', '-i', input_path,
            '-vf', f'scale=-2:{resolution[:-1]}',  # e.g., 720p -> 720
            '-c:v', 'libx264',
            '-preset', 'medium',
            '-crf', '23',
            '-c:a', 'aac',
            '-b:a', '128k',
            output_path
        ]

        await asyncio.create_subprocess_exec(*cmd)
        return output_path
```

### 2. Adaptive Bitrate Streaming (ABR)

```
┌────────────────────────────────────────────────────────────────────┐
│                    Adaptive Bitrate Streaming                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Video is split into segments (2-10 seconds each)                 │
│  Each segment encoded at multiple bitrates                        │
│                                                                    │
│  Timeline: |--seg1--|--seg2--|--seg3--|--seg4--|--seg5--|         │
│                                                                    │
│  1080p:    [=======][=======][=======][=======][=======]          │
│  720p:     [=======][=======][=======][=======][=======]          │
│  480p:     [=======][=======][=======][=======][=======]          │
│  360p:     [=======][=======][=======][=======][=======]          │
│                                                                    │
│  Player can switch quality between segments based on:             │
│  - Available bandwidth                                             │
│  - Buffer level                                                    │
│  - Device capability                                               │
└────────────────────────────────────────────────────────────────────┘
```

```python
class ABRController:
    """
    Client-side adaptive bitrate algorithm
    """

    def __init__(self, available_bitrates: list):
        self.bitrates = sorted(available_bitrates)
        self.buffer_level = 0
        self.bandwidth_estimate = 0
        self.history = []

    def select_quality(self) -> int:
        # Buffer-based algorithm (BBA)

        # Low buffer - be conservative
        if self.buffer_level < 5:  # seconds
            return self.bitrates[0]

        # High buffer - can be aggressive
        if self.buffer_level > 30:
            return self.select_by_bandwidth()

        # Medium buffer - interpolate
        buffer_ratio = (self.buffer_level - 5) / 25
        max_idx = int(buffer_ratio * (len(self.bitrates) - 1))

        return min(self.select_by_bandwidth(), self.bitrates[max_idx])

    def select_by_bandwidth(self) -> int:
        # Select highest bitrate that fits in bandwidth
        # Use 80% of estimated bandwidth for safety margin
        safe_bandwidth = self.bandwidth_estimate * 0.8

        for bitrate in reversed(self.bitrates):
            if bitrate <= safe_bandwidth:
                return bitrate

        return self.bitrates[0]

    def update_bandwidth_estimate(self, segment_size: int, download_time: float):
        measured = segment_size * 8 / download_time  # bits per second

        # Exponential moving average
        alpha = 0.3
        self.bandwidth_estimate = alpha * measured + (1 - alpha) * self.bandwidth_estimate
```

### 3. CDN Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                      CDN Architecture                               │
└────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │  Origin Server  │
                         │   (cold data)   │
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
       ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
       │  Regional   │     │  Regional   │     │  Regional   │
       │   PoP US    │     │   PoP EU    │     │   PoP Asia  │
       └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
              │                   │                   │
       ┌──────┼──────┐     ┌──────┼──────┐     ┌──────┼──────┐
       │      │      │     │      │      │     │      │      │
    ┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐
    │Edge ││Edge ││Edge ││Edge ││Edge ││Edge ││Edge ││Edge ││Edge │
    │ NYC ││ LA  ││ CHI ││ LON ││ FRA ││ PAR ││ TOK ││ SIN ││ SYD │
    └──┬──┘└──┬──┘└──┬──┘└──┬──┘└──┬──┘└──┬──┘└──┬──┘└──┬──┘└──┬──┘
       │      │      │      │      │      │      │      │      │
    Users  Users  Users  Users  Users  Users  Users  Users  Users

Cache strategy:
- Edge: Hot segments (last 24h popular videos)
- Regional: Warm content (last week)
- Origin: Everything
```

```python
class CDNRouting:
    def get_best_edge(self, user_ip: str, video_id: str) -> str:
        # 1. Get user location
        user_location = geoip.lookup(user_ip)

        # 2. Find nearest edge servers
        nearby_edges = self.get_nearby_edges(user_location, limit=3)

        # 3. Check which edges have the content cached
        for edge in nearby_edges:
            if await edge.has_content(video_id):
                return edge.url

        # 4. Route to nearest edge (will pull from origin)
        return nearby_edges[0].url

    async def cache_warmup(self, video_id: str, predicted_views: int):
        """Pre-cache popular videos to edge servers"""
        if predicted_views > 100000:
            # Cache to all major edges
            for edge in self.major_edges:
                await edge.prefetch(video_id)
        elif predicted_views > 10000:
            # Cache to regional PoPs
            for pop in self.regional_pops:
                await pop.prefetch(video_id)
```

### 4. Live Streaming

```
┌────────────────────────────────────────────────────────────────────┐
│                    Live Streaming Architecture                      │
└────────────────────────────────────────────────────────────────────┘

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│Broadcaster│───▶│  Ingest  │───▶│ Transcode│───▶│ Packager │───▶│   CDN    │
│  (OBS)   │RTMP│  Server  │    │ (realtime)│    │ (HLS/LL) │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └────┬─────┘
                                                                     │
                                                              ┌──────▼──────┐
                                                              │   Viewers   │
                                                              └─────────────┘

Latency targets:
- Traditional HLS: 15-30 seconds
- Low-Latency HLS (LL-HLS): 2-5 seconds
- WebRTC: < 1 second (for interactive)
```

```python
class LiveStreamProcessor:
    async def process_live_stream(self, stream_key: str, rtmp_input):
        # Real-time transcoding
        encoders = [
            self.create_encoder('1080p', bitrate=5000),
            self.create_encoder('720p', bitrate=2500),
            self.create_encoder('480p', bitrate=1000),
        ]

        async for frame in rtmp_input:
            # Encode in parallel
            segments = await asyncio.gather(*[
                encoder.encode_frame(frame)
                for encoder in encoders
            ])

            # Package to HLS segments (2-second chunks for low latency)
            for quality, segment in zip(['1080p', '720p', '480p'], segments):
                await self.packager.add_segment(stream_key, quality, segment)

            # Push to edge servers
            await self.push_to_cdn(stream_key)
```

### 5. Recommendations Engine

```python
class VideoRecommender:
    def get_recommendations(self, user_id: str, context: dict) -> list:
        # Multi-stage recommendation pipeline

        # 1. Candidate Generation (fast, broad)
        candidates = []

        # Similar to recently watched
        recent = self.get_recent_videos(user_id, limit=10)
        for video in recent:
            similar = self.get_similar_videos(video.id, limit=50)
            candidates.extend(similar)

        # From subscribed channels
        subscriptions = self.get_subscriptions(user_id)
        for channel in subscriptions:
            new_videos = self.get_channel_videos(channel.id, limit=20)
            candidates.extend(new_videos)

        # Trending in user's region
        trending = self.get_trending(context['country'], limit=100)
        candidates.extend(trending)

        # 2. Ranking (slower, precise)
        scored = []
        for video in set(candidates):
            score = self.rank_video(user_id, video, context)
            scored.append((score, video))

        # 3. Re-ranking for diversity
        final = self.diversify(sorted(scored, reverse=True)[:100])

        return final[:20]

    def rank_video(self, user_id: str, video, context: dict) -> float:
        # Features
        user_features = self.get_user_features(user_id)
        video_features = self.get_video_features(video.id)

        # ML model prediction (e.g., engagement probability)
        score = self.model.predict([
            user_features,
            video_features,
            context
        ])

        # Business rules adjustments
        if video.is_premium and not user.is_premium:
            score *= 0.5

        if video.age_hours < 24:
            score *= 1.2  # Boost fresh content

        return score
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Upload bandwidth | Direct upload to storage, resumable |
| Transcoding costs | GPU acceleration, spot instances |
| CDN costs | Tiered caching, popularity-based |
| Live latency | LL-HLS, WebRTC for interactive |
| Cold start | Pre-warming popular content |

---

## Trade-offs

### Codec Selection

| Codec | Compression | CPU Cost | Compatibility |
|-------|-------------|----------|---------------|
| H.264 | Good | Low | Universal |
| H.265 | Better (50%) | High | Modern devices |
| VP9 | Better (30%) | High | Browsers |
| AV1 | Best (50%+) | Very High | Limited |

### Segment Duration

| Duration | Latency | Overhead | Adaptability |
|----------|---------|----------|--------------|
| 2s | Low | High | Fast |
| 6s | Medium | Medium | Medium |
| 10s | High | Low | Slow |

---

## На интервью

### Ключевые моменты
1. **Video processing pipeline** — upload, transcode, package
2. **ABR streaming** — HLS/DASH, segment-based quality switching
3. **CDN architecture** — edge caching, geographic distribution
4. **Scale challenges** — storage, bandwidth, processing costs

### Типичные follow-up
- Как оптимизировать время до первого кадра?
- Как реализовать live streaming с низкой latency?
- Как обработать copyright detection?
- Как масштабировать transcoding для viral videos?
- Как реализовать watch progress sync между устройствами?

---

## См. также

- [Распределённый кэш](./07-distributed-cache.md) — кэширование сегментов видео на edge серверах
- [Kubernetes](../09-devops-sre/02-kubernetes.md) — оркестрация сервисов транскодирования

---

[← Назад к списку тем](README.md)
