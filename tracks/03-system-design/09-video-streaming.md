# 09. Video Streaming (YouTube/Netflix)

Развёрнутые вопросы и ответы про систему потоковой передачи видео: архитектура загрузки, транскодирование, адаптивное битрейт-потоковое вещание (HLS/DASH), CDN, прямое вещание, шифрование, рекомендации, поиск и масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [08-file-storage](./08-file-storage.md) · Следующая тема: [10-payment-system](./10-payment-system.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Transcoding** — это преобразование видеофайла из одного формата в другой, включая изменение кодека (H.264, H.265, VP9), разрешения (480p, 720p, 1080p, 4K) и битрейта (1 Mbps, 5 Mbps, 20 Mbps). Transcoding позволяет системе хранить видео один раз в оригинальном качестве, а затем генерировать версии для разных устройств и сетевых условий. Это критично для видеосервисов, так как нельзя требовать от пользователей с медленным интернетом загружать видео в 4K.

**Bitrate** — это количество данных видео, передаваемых в секунду, измеряется в Mbps (мегабит в секунду), например 1 Mbps, 5 Mbps, 20 Mbps. Выше битрейт означает выше качество видео (чётче картинка, более детальное изображение), но больше данные и более высокие требования к полосе пропускания. Выбор подходящего битрейта для пользователя — ключ к хорошему пользовательскому опыту: никто не хочет буферизации.

**Adaptive Bitrate Streaming (ABR)** — это техника автоматического переключения качества видео на лету в зависимости от текущей пропускной способности сети и буфера плеера. Видеоплеер непрерывно мониторит скорость загрузки и переключается между доступными качествами: если сеть замедляется, плеер переходит на более низкое качество для предотвращения буферизации. Это обеспечивает бесперебойное воспроизведение без остановок.

**HLS (HTTP Live Streaming)** — это стандарт потоковой передачи видео через обычный HTTP, разработанный Apple и широко поддерживаемый. HLS разбивает видео на короткие сегменты (2-10 секунд каждый) и создаёт манифест (m3u8 файл), описывающий все сегменты и качества. Клиент скачивает сегменты последовательно по одному, адаптируя качество на основе скорости загрузки. HLS работает через стандартный HTTP, что упрощает доставку через CDN и прохождение файрволлов.

**DASH (Dynamic Adaptive Streaming over HTTP)** — это стандарт для адаптивной потоковой передачи, разработанный MPEG и похожий на HLS, но использует другой формат манифеста (MPD XML вместо M3U8). DASH более гибкий и расширяемый, чем HLS, и поддерживает больше кодеков. Современные браузеры и видеоплеры поддерживают оба стандарта.

**Manifest** — это файл (например, .m3u8 для HLS или .mpd для DASH), который описывает структуру видео для плеера: список всех доступных сегментов, их длительность, различные качества (разрешения и битрейты), порядок воспроизведения и другие метаданные. Плеер сначала загружает манифест, затем использует информацию из него, чтобы решить, какие сегменты загружать и в каком качестве.

**Segment** — это короткий кусок видео длительностью 2-10 секунд, который упакован и подготовлен для независимой доставки. Вместо передачи часовового видеофайла целиком, видео разбивается на сегменты, и плеер загружает их по мере надобности. Разбиение на сегменты позволяет плееру независимо для каждого сегмента адаптировать качество, что обеспечивает гибкость и плавность воспроизведения.

**CDN PoP (Point of Presence)** — это региональный узел сети доставки контента (CDN), обычно расположенный в крупном городе или регионе. CDN PoP кэширует популярные видеосегменты локально, ближе к конечным пользователям. Благодаря PoP, видеоконтент доставляется с малой задержкой (latency 10-20 мс вместо 100-200 мс при доставке с origin сервера на другом конце света).

**Media Manifest** — это подробное описание всех доступных качеств видео (разрешение 360p/480p/720p/1080p/4K, соответствующие битрейты, использованные кодеки) и путей к их сегментам. Media manifest помогает видеоплееру выбрать наиболее подходящее качество для текущего устройства: мобильный телефон может выбрать 480p, планшет 720p, а ПК 1080p.

**GPU Transcoding** — это использование графических процессоров (GPU/видеокарт) для кодирования (transcoding) видео вместо использования CPU. GPU специализирована на обработке видео и может транскодировать видео в 5-10 раз быстрее, чем CPU, при использовании намного меньше электроэнергии и процессорных ресурсов. Это позволяет хостировать видеосистемы с меньшим оборудованием и расходами.

**Chunked Transfer** — это передача видео в виде множества небольших чанков (chunks) поочередно, вместо загрузки одного большого файла. Благодаря chunked transfer, воспроизведение может начаться до того, как загружен весь файл (streaming вместо download): пользователь начинает смотреть видео, пока остаток всё ещё загружается. Это обеспечивает намного лучший пользовательский опыт, так как не нужно ждать загрузки всего видео.

**DRM (Digital Rights Management)** — это технология защиты видеоконтента от несанкционированного копирования, скачивания или распространения. DRM шифрует видеопоток, требует лицензию для воспроизведения и может отслеживать использование. DRM критична для лицензионного контента (фильмы, сериалы), где правообладатель хочет защитить свои авторские права, но может затруднить пользовательский опыт.

**Low-Latency HLS** — это модификация стандарта HLS, которая снижает задержку между трансляцией и воспроизведением с типичных 15-30 секунд до 2-5 секунд. Low-latency HLS достигается за счёт уменьшения размера сегментов и их более частого обновления. Это критично для live streaming событий, где низкая latency важна (спортивные трансляции, игры, интерактивные шоу).

---

## Вопросы и разборы

### 1. Как спроектировать общую архитектуру системы видеопотока?

**Зачем спрашивают.** Видеопотоковая система — один из самых сложных систем дизайна, охватывающих загрузку, обработку, кэширование, доставку и анализ. Интервьюер проверяет умение структурировать большую распределённую систему.

**Короткий ответ.** Система делится на три потока: (1) Upload Flow — загрузка оригинального видео в S3, (2) Processing Flow — транскодирование в несколько качеств и упаковка в HLS/DASH, (3) Playback Flow — доставка через CDN с адаптивным выбором качества.

**Детальный разбор.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Video Streaming Architecture                    │
└─────────────────────────────────────────────────────────────────────┘

                              UPLOAD FLOW
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ Creator  │────▶│ Upload   │────▶│ Message  │────▶│ S3/GCS   │
│ App      │     │ Service  │     │ Queue    │     │ Storage  │
└──────────┘     │ (chunked)│     │ (Kafka)  │     └──────┬───┘
                 └──────────┘     └──────────┘            │
                                                          │
                         PROCESSING FLOW                  │
                                                          │
        ┌────────────────────────────────────────────────▼─┐
        │                                                   │
  ┌─────▼─────┐    ┌──────────┐    ┌─────────┐    ┌──────▼──┐
  │ Transcode  │───▶│ HLS/DASH │───▶│ Upload  │───▶│ CDN     │
  │ Workers    │    │ Packager │    │ to CDN  │    │ Origin  │
  │ (GPU)      │    │          │    │         │    │         │
  └────────────┘    └──────────┘    └─────────┘    └────┬────┘
                                                         │
                                                         │
                          PLAYBACK FLOW                  │
                                                         │
  ┌──────────┐     ┌──────────┐     ┌──────────┐       │
  │ Viewer   │────▶│ Edge CDN │────▶│ Regional │◀──────┘
  │ App      │     │ (nearest)│     │ PoP      │
  └──────────┘     │ cache    │     │ Origin   │
                   └──────────┘     └──────────┘

                        METADATA FLOW

  ┌──────────┐     ┌──────────┐     ┌─────────────────┐
  │ Viewer   │────▶│ API      │────▶│ Video Metadata  │
  │ Request  │     │ Gateway  │     │ Service         │
  └──────────┘     └──────────┘     └────────┬────────┘
                                             │
                      ┌──────────────────────┼──────────────────────┐
                      │                      │                      │
                 ┌────▼────┐           ┌────▼────┐          ┌──────▼──┐
                 │ Metadata │           │ Search  │          │ Recs    │
                 │ DB (SQL) │           │ Service │          │ Service │
                 │          │           │(Elastic)│          │ (ML)    │
                 └──────────┘           └─────────┘          └─────────┘
```

**Ключевые компоненты:**
- **Upload Service** — управление загрузкой по частям, возобновление при разрыве соединения
- **Transcoding Pipeline** — масштабируемые воркеры с GPU для 5-15 версий видео
- **CDN** — распределённая сеть с региональными PoP (point of presence)
- **Metadata Service** — управление метаданными видео, статусом обработки
- **Search & Recommendations** — полнотекстовый поиск + ML для рекомендаций

**Пример.**
```go
type VideoSystem struct {
    uploadService    *UploadService
    processingQueue  *KafkaQueue
    transcoder       *TranscodingService
    cdnManager       *CDNManager
    metadataDB       *PostgreSQL
    searchService    *ElasticsearchService
    recommendService *RecommendationService
}

func (vs *VideoSystem) UploadVideo(ctx context.Context, video *VideoMetadata) error {
    videoID, err := vs.metadataDB.CreateVideo(video)
    if err != nil {
        return err
    }
    uploadURL := vs.uploadService.GenerateUploadURL(videoID)
    vs.metadataDB.UpdateStatus(videoID, "processing")
    return nil
}

func (vs *VideoSystem) ProcessVideo(ctx context.Context, videoID string) error {
    original, err := vs.uploadService.DownloadFromS3(videoID)
    if err != nil {
        return err
    }
    metadata := vs.transcoder.ExtractMetadata(original)
    thumb := vs.transcoder.GenerateThumbnail(original)
    vs.uploadService.UploadToS3(videoID, "thumbnail.jpg", thumb)

    qualities := []string{"360p", "480p", "720p", "1080p"}
    for _, quality := range qualities {
        go func(q string) {
            encoded := vs.transcoder.Transcode(original, q)
            vs.uploadService.UploadToS3(videoID, q, encoded)
            vs.metadataDB.UpdateQuality(videoID, q, "available")
        }(quality)
    }

    manifest := vs.transcoder.GenerateHLSManifest(videoID, qualities)
    vs.uploadService.UploadToS3(videoID, "master.m3u8", manifest)
    vs.cdnManager.Publish(videoID)
    vs.metadataDB.UpdateStatus(videoID, "published")
    return nil
}

func (vs *VideoSystem) GetPlaybackManifest(ctx context.Context, userID, videoID string) (*Manifest, error) {
    video, err := vs.metadataDB.GetVideo(videoID)
    if err != nil {
        return nil, err
    }
    if !vs.metadataDB.CanWatch(userID, videoID) {
        return nil, ErrAccessDenied
    }
    cdnURL := vs.cdnManager.GetNearestEdge(userID)
    manifest := &Manifest{
        VideoID:   videoID,
        Title:     video.Title,
        Duration:  video.Duration,
        Thumbnail: fmt.Sprintf("%s/%s/thumbnail.jpg", cdnURL, videoID),
        Streams: map[string]string{
            "hls":  fmt.Sprintf("%s/%s/master.m3u8", cdnURL, videoID),
            "dash": fmt.Sprintf("%s/%s/manifest.mpd", cdnURL, videoID),
        },
    }
    go vs.metadataDB.RecordView(userID, videoID)
    return manifest, nil
}
```

**Типичные ошибки.**
- Попытка транскодировать одновременно все видео без ограничения параллелизма — истощение GPU ресурсов.
- Кэширование всех видео на всех edge серверах — неэффективное использование дискового пространства.
- Отсутствие отслеживания прогресса транскодирования — пользователь не видит, идёт ли обработка.
- Не учитывать задержку распространения через CDN — манифесты могут быть несинхронизированы.

**На интервью.**
- Объясни, почему upload и processing отделены от playback.
- Упомяни асинхронность: upload не блокирует пользователя, обработка идёт в фоне.
- Follow-up: «Как обрабатывать видео, которое загружается 10 часов?» — resumable uploads.
- Follow-up: «Как оптимизировать стоимость транскодирования?» — spot instances, очередь приоритетов.

---

### 2. Как спроектировать конвейер транскодирования видео?

**Зачем спрашивают.** Транскодирование — самая дорогая и сложная операция. Нужно балансировать между качеством, стоимостью и скоростью обработки.

**Короткий ответ.** Конвейер состоит из: (1) разделение видео на чанки, (2) параллельное транскодирование на GPU воркерах, (3) прогрессивная загрузка результатов, (4) использование spot instances для снижения стоимости.

**Детальный разбор.**

```
┌────────────────────────────────────────────────────────────────┐
│                  Transcoding Pipeline                          │
└────────────────────────────────────────────────────────────────┘

INPUT VIDEO (original)
       │
       ▼
┌──────────────────────┐
│ Extract Metadata     │  (duration, resolution, frame rate, audio)
│ Generate Thumbnail   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Split into Chunks    │  (10-60 second segments for parallel)
└──────────┬───────────┘
           │
     ┌─────┴──────────────────┐
     │                        │
     ▼                        ▼
┌──────────────────┐   ┌──────────────────┐
│ Transcode 360p   │   │ Transcode 1080p  │
│ H.264 @600kbps   │   │ H.264 @5Mbps     │
│ (Worker GPU 1)   │   │ (Worker GPU N)   │
└────────┬─────────┘   └──────────┬───────┘
         │                        │
         └──────────┬─────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Merge Segments       │
         │ (reassemble chunks)  │
         └──────────┬───────────┘
                    │
         ┌──────────┴────────────┐
         │                       │
         ▼                       ▼
  ┌────────────────┐    ┌────────────────┐
  │ HLS Packaging  │    │ DASH Packaging │
  │ (m3u8 playlists)   │ (mpd manifest) │
  └────────┬───────┘    └────────┬───────┘
           │                     │
           └──────────┬──────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ Upload to CDN    │
              │ (S3/GCS origin)  │
              └──────────────────┘

Quality Settings (per resolution):
┌────────┬──────────┬────────┬──────────┬────────────┐
│ Quality│ Size     │ Bitrate│ Codec    │ Use Case   │
├────────┼──────────┼────────┼──────────┼────────────┤
│ 2160p  │ 6GB/hour │ 15Mbps │ H.265    │ 4K devices │
│ 1080p  │ 2GB/hour │ 5Mbps  │ H.264    │ Desktop    │
│ 720p   │ 1GB/hour │ 2.5Mbps│ H.264    │ Tablet     │
│ 480p   │ 500MB/h  │ 1Mbps  │ H.264    │ Mobile     │
│ 360p   │ 300MB/h  │ 600kbps│ H.264    │ Low speed  │
└────────┴──────────┴────────┴──────────┴────────────┘
```

**Пример.**
```go
type TranscodingService struct {
    workerPool    *WorkerPool
    jobQueue      chan *TranscodeJob
    s3Client      *S3Client
    progressDB    *ProgressDB
    costOptimizer *CostOptimizer
}

type TranscodeJob struct {
    VideoID        string
    Resolution     string
    Codec          string
    Bitrate        int
    InputPath      string
    OutputPath     string
    Priority       int
    RetryCount     int
}

func (ts *TranscodingService) SubmitTranscodeJob(job *TranscodeJob) error {
    estimatedCost := ts.costOptimizer.EstimateCost(job)
    if estimatedCost > 10.0 {
        job.UseSpotInstance = true
    }
    ts.jobQueue <- job
    ts.progressDB.CreateJob(job)
    return nil
}

func (ts *TranscodingService) transcodeVideo(job *TranscodeJob) error {
    cmd := exec.CommandContext(context.Background(),
        "ffmpeg",
        "-i", job.InputPath,
        "-vf", fmt.Sprintf("scale=-2:%s", job.Resolution[:len(job.Resolution)-1]),
        "-c:v", "libx264",
        "-preset", "medium",
        "-crf", "23",
        "-b:v", fmt.Sprintf("%dk", job.Bitrate),
        "-c:a", "aac",
        "-b:a", "128k",
        job.OutputPath,
    )

    if err := cmd.Run(); err != nil {
        return fmt.Errorf("ffmpeg failed: %w", err)
    }

    if err := ts.s3Client.UploadFile(job.OutputPath, job.VideoID, job.Resolution); err != nil {
        return fmt.Errorf("S3 upload failed: %w", err)
    }

    return nil
}
```

**Типичные ошибки.**
- Не параллелизировать работу — каждое видео транскодируется последовательно.
- Использовать неправильный `preset` в ffmpeg — `slow` даёт лучшее качество, но медленнее в 10 раз.
- Не отслеживать прогресс — пользователь видит "processing" часами.
- Не обрабатывать отказы воркеров — потеря видео в очереди.

**На интервью.**
- Объясни, почему параллелизм критичен для масштабирования.
- Упомяни GPU ускорение: NVIDIA NVENC быстрее чем CPU ffmpeg в 50-100x.
- Follow-up: «Как обрабатывать видео длиной 10 часов?» — разделение на чанки.
- Follow-up: «Как оптимизировать качество/скорость?» — two-pass encoding.

---

### 3. Как реализовать адаптивное потоковое вещание (HLS/DASH)?

**Зачем спрашивают.** HLS/DASH — стандарты для потоковой передачи. Нужно понимать, как клиент выбирает качество на основе пропускной способности.

**Короткий ответ.** HLS/DASH разбивают видео на сегменты (2-10 сек). Каждый сегмент кодируется в несколько качеств. Плеер скачивает манифест (список сегментов) и динамически переключает качество в зависимости от пропускной способности и уровня буфера.

**Детальный разбор.**

```
┌──────────────────────────────────────────────────────────────┐
│         HLS Manifest (Adaptive Bitrate Streaming)            │
└──────────────────────────────────────────────────────────────┘

Master Playlist (master.m3u8):
  ├─ #EXT-X-STREAM-INF:BANDWIDTH=600000,RESOLUTION=640x360
  │  └─ 360p/playlist.m3u8
  │
  ├─ #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
  │  └─ 720p/playlist.m3u8
  │
  └─ #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
     └─ 1080p/playlist.m3u8

Quality-Specific Playlist (720p/playlist.m3u8):
  ├─ #EXTINF:6.0,
  │  └─ segment0.ts
  │
  ├─ #EXTINF:6.0,
  │  └─ segment1.ts
  │
  └─ #EXTINF:6.0,
     └─ segment2.ts

┌────────────────────────────────────────────────────────────┐
│            Adaptive Bitrate Selection Algorithm             │
└────────────────────────────────────────────────────────────┘

BANDWIDTH ESTIMATION:
Each segment download time → measure actual bandwidth
Exponential Moving Average (EMA) to smooth estimates

Buffer-Based Algorithm (BBA):
┌─────────────────────────────────────────┐
│ Buffer Level (seconds)                  │
├─────────────────────────────────────────┤
│  30+ secs: can use 1080p (5Mbps)       │
│  15-30: use 720p (2.5Mbps)             │
│  5-15: switch to 480p (1Mbps)          │
│  <5: emergency - use 360p (600kbps)    │
└─────────────────────────────────────────┘

THROUGHPUT CONSTRAINT:
Selected bitrate < 80% of measured bandwidth
(20% safety margin to avoid rebuffering)
```

**Пример.**
```go
type ABRController struct {
    qualities         []Quality
    bandwidthHistory  []float64
    bufferHistory     []float64
    currentQuality    int
    segmentDuration   time.Duration
}

func (abr *ABRController) SelectQuality() string {
    bandwidth := abr.estimateBandwidth()
    buffer := abr.getCurrentBuffer()

    if buffer < 5 {
        abr.currentQuality = 0
        return abr.qualities[0].Name
    }

    if buffer > 30 {
        return abr.selectByBandwidth(bandwidth)
    }

    bufferRatio := (buffer - 5.0) / 25.0
    maxQuality := int(float64(len(abr.qualities)-1) * bufferRatio)

    selected := abr.selectByBandwidth(bandwidth)
    selectedIdx := abr.qualityIndex(selected)

    if selectedIdx > maxQuality {
        return abr.qualities[maxQuality].Name
    }

    return selected
}

func (abr *ABRController) selectByBandwidth(bandwidth float64) string {
    safeBandwidth := bandwidth * 0.8
    var selected int = 0
    for i, q := range abr.qualities {
        bitrateMbps := float64(q.Bitrate) / 1000.0
        if bitrateMbps <= safeBandwidth {
            selected = i
        }
    }
    return abr.qualities[selected].Name
}

func (abr *ABRController) estimateBandwidth() float64 {
    if len(abr.bandwidthHistory) == 0 {
        return 10.0
    }
    alpha := 0.3
    ema := abr.bandwidthHistory[0]
    for i := 1; i < len(abr.bandwidthHistory); i++ {
        ema = alpha*abr.bandwidthHistory[i] + (1-alpha)*ema
    }
    return ema
}
```

**Типичные ошибки.**
- Слишком частое переключение качества — вызывает рестарты буфера.
- Не сглаживать оценку пропускной способности — колебания приводят к скачкам.
- Использовать слишком длинные сегменты (>10 сек) — медленное переключение.

**На интервью.**
- Объясни разницу между HLS и DASH.
- Упомяни ABR как критичную для хорошего UX.
- Follow-up: «Как обрабатывать медленную сеть?» — более агрессивное понижение качества.
- Follow-up: «Как синхронизировать аудио и видео?» — timestamp synchronization.

---

### 4. Как спроектировать систему CDN для видеодоставки?

**Зачем спрашивают.** CDN — решающий компонент для масштабирования. Нужно понимать географическое распределение, кэширование и маршрутизацию.

**Короткий ответ.** CDN состоит из: (1) Origin servers — хранилище исходного контента, (2) Regional PoPs — промежуточные узлы, (3) Edge servers — узлы рядом с пользователями. Кэширование по уровням: горячие на edge, тёплые на regional, всё на origin.

**Детальный разбор.**

```
┌──────────────────────────────────────────────────────────────┐
│                    CDN Architecture                          │
└──────────────────────────────────────────────────────────────┘

                     ┌─────────────────┐
                     │  Origin Server  │
                     │  (authoritative)│
                     │  (all content)  │
                     └────────┬────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
        ┌──▼──┐           ┌──▼──┐           ┌──▼──┐
        │ US  │           │ EU  │           │APAC │
        │ PoP │           │ PoP │           │PoP  │
        └──┬──┘           └──┬──┘           └──┬──┘
           │                 │                 │
    ┌──────┼──────┐   ┌──────┼──────┐   ┌──────┼──────┐
    │      │      │   │      │      │   │      │      │
  ┌─▼─┐ ┌─▼─┐ ┌─▼─┐ ┌─▼─┐ ┌─▼─┐ ┌─▼─┐ ┌─▼─┐ ┌─▼─┐ ┌─▼─┐
  │NYC│ │LAX│ │CHI│ │LHR│ │CDG│ │FRA│ │NRT│ │SGP│ │SYD│
  │Cch│ │Cch│ │Cch│ │Cch│ │Cch│ │Cch│ │Cch│ │Cch│ │Cch│
  └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘ └─┬─┘
    │     │     │     │     │     │     │     │     │
   Users Users Users Users Users Users Users Users Users

CACHE HIERARCHY:
┌─────────────────────────────────────┐
│ Edge Cache (5-15 GB SSD)             │
│ - Last 24h popular segments          │
│ - Hit rate: 85-95%                  │
│ - Latency: <50ms                    │
└──────────────────┬──────────────────┘
                   │ Miss
                   ▼
┌─────────────────────────────────────┐
│ Regional PoP Cache (100-500 GB)     │
│ - Last week popular segments         │
│ - Hit rate: 60-80%                  │
│ - Latency: 100-300ms                │
└──────────────────┬──────────────────┘
                   │ Miss
                   ▼
┌─────────────────────────────────────┐
│ Origin Server (PB scale)             │
│ - All content                        │
│ - Hit rate: 100%                    │
│ - Latency: 500-1000ms               │
└─────────────────────────────────────┘
```

**Пример.**
```go
type CDNManager struct {
    originServers  []string
    regionalPops   map[string]string
    edgeServers    map[string]string
    geoRouter      *GeoIPRouter
    cacheManager   *CacheManager
}

func (cdn *CDNManager) GetPlaybackURL(userIP, videoID, quality string) (string, error) {
    userLoc := cdn.geoRouter.LookupLocation(userIP)
    nearestEdges := cdn.findNearestEdges(userLoc, 3)

    for _, edge := range nearestEdges {
        if cdn.cacheManager.IsCached(edge, videoID, quality) {
            return fmt.Sprintf("https://%s/%s/%s/segment.ts", edge, videoID, quality), nil
        }
    }

    return fmt.Sprintf("https://%s/%s/%s/segment.ts", nearestEdges[0], videoID, quality), nil
}

func (cdn *CDNManager) WarmCache(videoID string, expectedViews int) error {
    if expectedViews > 1_000_000 {
        return cdn.preCacheToEdges(videoID, cdn.getAllEdges())
    } else if expectedViews > 100_000 {
        return cdn.preCacheToEdges(videoID, cdn.getMajorEdges())
    }
    return nil
}
```

**Типичные ошибки.**
- Кэшировать всё на всех edge серверах — неэффективно и дорого.
- Не отслеживать популярность видео — кэшируют непопулярные, вытесняя популярные.
- Origin перегружен — нужна защита (rate limiting, circuit breaker).

**На интервью.**
- Объясни иерархию кэширования.
- Упомяни geo-routing для снижения latency.
- Follow-up: «Как обновить видео?» — cache invalidation.
- Follow-up: «Как обрабатывать traffic spike?» — burst capacity на edge.

---

### 5. Как спроектировать систему загрузки видео с возобновлением?

**Зачем спрашивают.** Большие видеофайлы часто загружаются нестабильно. Нужен механизм для возобновления с того же места.

**Короткий ответ.** Разделить видео на чанки (5-100 MB каждый). Клиент загружает каждый чанк с уникальным ID. Server отслеживает загруженные чанки. При разрыве клиент запрашивает статус и продолжает с пропущенных чанков.

**Детальный разбор.**

```
┌──────────────────────────────────────────────────────────────┐
│           Resumable Upload Architecture                      │
└──────────────────────────────────────────────────────────────┘

CLIENT:
┌──────────────────────────────────────┐
│ Local Video File (1-50 GB)           │
└────────────────┬─────────────────────┘
                 │
          Split into Chunks
                 │
    ┌────────────┼────────────┬────────────┐
    │            │            │            │
  Chunk1      Chunk2      Chunk3      ChunkN
   (50MB)      (50MB)      (50MB)    (variable)
    │            │            │            │
    ▼            ▼            ▼            ▼
┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐
│Upload│     │Upload│     │Upload│     │Upload│
│#1    │     │#2    │     │#3    │     │#N    │
└──┬───┘     └──┬───┘     └──┬───┘     └──┬───┘
   │            │            │            │
   ├────────────┼────────────┼────────────┤
   │            │            │            │
   ▼ (retry-able)            ▼ (retry-able)

   Network failure at chunk 3
                 │
        Resume Upload Status
                 │
   Server responds: chunks 1-2 OK, 3+ pending
                 │
   Resume from chunk 3 (skip 1-2)
```

**Пример.**
```go
type UploadService struct {
    s3Client       *s3.Client
    metadataDB     *PostgreSQL
    chunkSize      int64 // 50 MB
}

type UploadSession struct {
    UploadID     string
    VideoID      string
    Filename     string
    TotalSize    int64
    Status       string
    ChunksMap    map[int]ChunkStatus
}

func (us *UploadService) InitiateUpload(ctx context.Context, videoID, filename string) (*UploadSession, error) {
    session := &UploadSession{
        UploadID:   uuid.New().String(),
        VideoID:    videoID,
        Filename:   filename,
        Status:     "initiated",
        CreatedAt:  time.Now(),
        ExpiresAt:  time.Now().Add(24 * time.Hour),
        ChunksMap:  make(map[int]ChunkStatus),
    }

    err := us.metadataDB.CreateUploadSession(ctx, session)
    return session, err
}

func (us *UploadService) GetUploadStatus(ctx context.Context, uploadID string) (*UploadSession, error) {
    session, err := us.metadataDB.GetUploadSession(ctx, uploadID)
    if err != nil {
        return nil, err
    }

    if time.Now().After(session.ExpiresAt) {
        us.metadataDB.DeleteUploadSession(ctx, uploadID)
        return nil, ErrUploadExpired
    }

    return session, nil
}

func (us *UploadService) UploadChunk(ctx context.Context, req *UploadChunkRequest) error {
    session, err := us.GetUploadStatus(ctx, req.UploadID)
    if err != nil {
        return err
    }

    chunkIdx := req.ChunkIndex

    if status, exists := session.ChunksMap[chunkIdx]; exists && status.Status == "completed" {
        return nil
    }

    s3Key := fmt.Sprintf("uploads/%s/%s/chunk-%d", session.VideoID, req.UploadID, chunkIdx)

    actualChecksum := us.calculateChecksum(req.Data)
    if actualChecksum != req.Checksum {
        return ErrChecksumMismatch
    }

    _, err = us.s3Client.PutObject(ctx, &s3.PutObjectInput{
        Bucket: aws.String("video-uploads"),
        Key:    aws.String(s3Key),
        Body:   bytes.NewReader(req.Data),
    })
    if err != nil {
        return fmt.Errorf("S3 upload failed: %w", err)
    }

    session.ChunksMap[chunkIdx] = ChunkStatus{
        Index:      chunkIdx,
        Size:       int64(len(req.Data)),
        Status:     "completed",
        Checksum:   actualChecksum,
        UploadedAt: time.Now(),
    }

    return us.metadataDB.UpdateUploadSession(ctx, session)
}

func (us *UploadService) CompleteUpload(ctx context.Context, uploadID string) error {
    session, err := us.GetUploadStatus(ctx, uploadID)
    if err != nil {
        return err
    }

    numChunks := (session.TotalSize + us.chunkSize - 1) / us.chunkSize
    for i := 0; i < int(numChunks); i++ {
        if status, exists := session.ChunksMap[i]; !exists || status.Status != "completed" {
            return fmt.Errorf("chunk %d not uploaded", i)
        }
    }

    uploadKey := fmt.Sprintf("videos/%s/original.mp4", session.VideoID)

    // Assemble chunks (simplified)
    session.Status = "completed"
    us.metadataDB.UpdateUploadSession(ctx, session)

    us.triggerTranscoding(ctx, session.VideoID)

    return nil
}
```

**Типичные ошибки.**
- Не использовать checksums — файл может быть повреждён.
- Хранить весь файл в памяти — невозможно загружать большие видео.
- Не очищать expired sessions — диск заполняется orphaned чанками.

**На интервью.**
- Объясни, почему нужны чанки.
- Упомяни parallelism для ускорения.
- Follow-up: «Как обрабатывать network timeouts?» — exponential backoff.
- Follow-up: «Как избежать storage waste?» — cleanup, garbage collection.

---

### 6. Как реализовать прямое вещание (live streaming)?

**Зачем спрашивают.** Live имеет другие требования чем VOD: низкая latency, real-time transcoding, масштабирование audience.

**Короткий ответ.** Live использует RTMP для инжеста, real-time транскодирование, низко-latency HLS или WebRTC. Отличие: сегменты на лету, нет pre-recording, масштабирование для spike зрителей.

**Детальный разбор.**

```
┌──────────────────────────────────────────────────────────────┐
│              Live Streaming Architecture                     │
└──────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│Broadcaster   │───▶│ Ingest   │───▶│Transcode │───▶│ Packager │
│ (OBS)        │RTMP│ Service  │    │(realtime)│    │(HLS/LL)  │
└──────────────┘    └──────────┘    └──────────┘    └────┬─────┘
                                                          │
                                                    ┌─────▼─────┐
                                                    │    CDN    │
                                                    │  Viewers  │
                                                    └───────────┘

Latency targets:
- Traditional HLS: 15-30 seconds
- Low-Latency HLS: 2-5 seconds
- WebRTC: <1 second (interactive)
```

**Пример.**
```go
type LiveStreamService struct {
    ingestService  *IngestService
    transcoders    []*RealtimeTranscoder
    packager       *LivePackager
    cdnManager     *CDNManager
    viewerTracker  *ViewerTracker
}

func (ls *LiveStreamService) StartLiveStream(ctx context.Context, channelID, title string) (*LiveStream, error) {
    stream := &LiveStream{
        StreamID:   uuid.New().String(),
        ChannelID:  channelID,
        StreamKey:  generateStreamKey(),
        Status:     "offline",
        Title:      title,
        Bitrates: map[string]int{
            "1080p": 5000,
            "720p":  2500,
            "480p":  1000,
            "360p":  600,
        },
    }

    err := ls.storeStreamMetadata(ctx, stream)
    return stream, err
}

func (ls *LiveStreamService) IngestStream(ctx context.Context, streamID string, rtmpInput io.Reader) error {
    stream, err := ls.getStream(ctx, streamID)
    if err != nil {
        return err
    }

    stream.Status = "live"
    stream.StartedAt = time.Now()

    transcodeTasks := make([]chan []byte, 4)
    for i := range transcodeTasks {
        transcodeTasks[i] = make(chan []byte, 30)
    }

    go func() {
        frameBuffer := bufio.NewReader(rtmpInput)
        for {
            frame, err := readRTMPFrame(frameBuffer)
            if err != nil {
                return
            }

            for _, ch := range transcodeTasks {
                select {
                case ch <- frame:
                case <-ctx.Done():
                    return
                default:
                    // Buffer full, skip frame (acceptable for live)
                }
            }
        }
    }()

    qualities := []string{"1080p", "720p", "480p", "360p"}
    var wg sync.WaitGroup
    segmentChannels := make(map[string]chan []byte)

    for i, quality := range qualities {
        wg.Add(1)
        segmentChannels[quality] = make(chan []byte, 100)

        go func(q string, frameCh chan []byte) {
            defer wg.Done()
            transcoder := ls.transcoders[i%len(ls.transcoders)]
            for frame := range frameCh {
                encoded := transcoder.EncodeFrame(frame, q)
                select {
                case segmentChannels[q] <- encoded:
                case <-ctx.Done():
                    return
                }
            }
        }(quality, transcodeTasks[i])
    }

    go ls.packager.PackageSegments(ctx, stream.StreamID, segmentChannels)
    wg.Wait()
    return nil
}
```

**Типичные ошибки.**
- Не обрабатывать отказ трансокодера — поток прерывается.
- Кэшировать live как VOD — кэш должен быть краткосрочным.
- Игнорировать spike in viewers — CDN перегружена.

**На интервью.**
- Объясни разницу live vs VOD.
- Упомяни real-time constraints.
- Follow-up: «Как обрабатывать broadcaster disconnect?» — graceful shutdown.
- Follow-up: «Как масштабировать для миллиона зрителей?» — multi-CDN.

---

### 7. Как реализовать защиту видеоконтента (DRM)?

**Зачем спрашивают.** Premium контент нужно защищать. DRM требует шифрования и управления ключами.

**Короткий ответ.** DRM использует AES-128 шифрование видеосегментов + token-based key delivery. Ключи в secure key server, доставляются только авторизованным клиентам. Клиент проигрывает, но не может сохранить в открытом виде.

**Детальный разбор.**

```
┌──────────────────────────────────────────────────────────────┐
│              DRM (Digital Rights Management)                 │
└──────────────────────────────────────────────────────────────┘

ENCRYPTION AT ENCODING:
┌──────────────┐
│ Raw Video    │
└──────┬───────┘
       │
       ▼
┌────────────────────────────┐
│ Transcode + Encrypt        │
│ - AES-128-CBC encryption   │
│ - Random IV per segment    │
│ - Key ID in manifest       │
└──────────┬─────────────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
Encrypted    Encrypted
Seg1.ts      Seg2.ts

PLAYBACK FLOW:
┌──────────────┐
│ Player       │ Has user token
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│ Request master.m3u8  │ (contains key IDs)
└──────────┬───────────┘
           │
           ▼
┌────────────────────────────────────────┐
│ For each segment:                      │
│ 1. Get key ID from manifest            │
│ 2. Request license from key server     │
│    (include user token + key ID)       │
└──────────┬─────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────┐
│ Key Server validates:                  │
│ - User is subscribed                   │
│ - Token not expired                    │
│ - IP geolocation matches               │
└──────────┬─────────────────────────────┘
           │
           ├─ Approved: return AES key
           │
           └─ Denied: return error (403)
           │
           ▼
┌────────────────────────────────────────┐
│ Player decrypts segment in-memory      │
│ (key never stored on disk)             │
└────────────────────────────────────────┘
```

**Пример.**
```go
type DRMService struct {
    keyDB         *PostgreSQL
    tokenVerifier *TokenVerifier
    encryptor     *AESEncryptor
    licenseCache  *Cache
}

func (drm *DRMService) EncryptSegment(segment []byte, videoID string, segmentNum int) ([]byte, error) {
    iv := make([]byte, 16)
    rand.Read(iv)

    key := drm.getOrCreateKey(videoID)
    encrypted, err := drm.encryptor.Encrypt(key, iv, segment)
    if err != nil {
        return nil, err
    }

    drm.storeIV(videoID, segmentNum, iv)
    return encrypted, nil
}

func (drm *DRMService) getOrCreateKey(videoID string) []byte {
    existing, err := drm.keyDB.GetKey(videoID)
    if err == nil {
        return existing
    }

    key := make([]byte, 16)
    rand.Read(key)
    drm.keyDB.StoreKey(videoID, key)
    return key
}

type LicenseRequest struct {
    KeyID       string
    UserToken   string
    VideoID     string
}

type LicenseResponse struct {
    Key           string
    ExpiresAt     int64
    CacheDuration int
}

func (drm *DRMService) DeliverLicense(ctx context.Context, req *LicenseRequest) (*LicenseResponse, error) {
    claims, err := drm.tokenVerifier.Verify(req.UserToken)
    if err != nil {
        return nil, ErrUnauthorized
    }

    userID := claims.UserID

    access, err := drm.checkAccess(ctx, userID, req.VideoID)
    if err != nil || !access {
        return nil, ErrAccessDenied
    }

    cacheKey := fmt.Sprintf("%s:%s", userID, req.VideoID)
    if cached, ok := drm.licenseCache.Get(cacheKey); ok {
        return cached.(*LicenseResponse), nil
    }

    key := drm.getOrCreateKey(req.VideoID)

    license := &LicenseResponse{
        Key:           base64.StdEncoding.EncodeToString(key),
        ExpiresAt:     time.Now().Add(24 * time.Hour).Unix(),
        CacheDuration: 3600,
    }

    drm.licenseCache.Set(cacheKey, license, 5*time.Minute)
    drm.logLicenseDelivery(userID, req.VideoID)

    return license, nil
}

func (drm *DRMService) checkAccess(ctx context.Context, userID, videoID string) (bool, error) {
    subscription, err := drm.keyDB.GetSubscription(ctx, userID)
    if err != nil {
        return false, err
    }

    if !subscription.IsActive {
        return false, nil
    }

    video, err := drm.keyDB.GetVideo(ctx, videoID)
    if err != nil {
        return false, err
    }

    if !isVideoAvailableInRegion(video, subscription.Region) {
        return false, nil
    }

    return true, nil
}
```

**Типичные ошибки.**
- Хранить ключи в открытом виде в коде.
- Не ограничивать скачивание лицензии по IP/location.
- Не логировать доступ к контенту.

**На интервью.**
- Объясни, почему DRM нужен.
- Упомяни разницу между DRM и просто шифрованием.
- Follow-up: «Как справиться с circumvention?» — behavioral detection.
- Follow-up: «Как обновить ключи?» — key rotation на уровне манифеста.

---

### 8. Как спроектировать систему рекомендаций видео?

**Зачем спрашивают.** Рекомендации — ключ к engagement. Нужно работать с ML и масштабированием.

**Короткий ответ.** Двухстадийная система: (1) Candidate generation — тысячи потенциальных видео, (2) Ranking — ML модель ранжирует по engagement. Re-ranking добавляет диверсификацию.

**Детальный разбор.**

```
┌────────────────────────────────────────────────────────────┐
│        Recommendation System Architecture                  │
└────────────────────────────────────────────────────────────┘

STAGE 1: CANDIDATE GENERATION
┌──────────┬──────────┬──────────┬──────────┐
│ Similar  │Subscribed│ Trending │ Popular  │
│ Videos   │ Channels │ Videos   │ Videos   │
└──────────┴──────────┴──────────┴──────────┘
         → 5000 candidates

STAGE 2: RANKING
User Features + Video Features → ML Model → Score

STAGE 3: RE-RANKING
- Diversity: mix categories
- Freshness: boost new videos
- Personalization

         → Top 20 results
```

**Пример.**
```go
type RecommendationService struct {
    candidateGen    *CandidateGenerator
    rankingModel    *RankingModel
    userFeatureDB   *PostgreSQL
    videoFeatureDB  *PostgreSQL
    cache           *Redis
}

func (rs *RecommendationService) GetRecommendations(ctx context.Context, userID string, count int) ([]*Video, error) {
    cacheKey := fmt.Sprintf("recs:%s", userID)
    if cached, ok := rs.cache.Get(cacheKey); ok {
        return cached.([]*Video), nil
    }

    user, err := rs.userFeatureDB.GetUser(ctx, userID)
    if err != nil {
        return nil, err
    }

    candidates := rs.candidateGen.Generate(ctx, user)

    type scoredVideo struct {
        video *Video
        score float32
    }

    scored := make([]*scoredVideo, len(candidates))
    for i, video := range candidates {
        score := rs.rankingModel.Score(ctx, user, video)
        scored[i] = &scoredVideo{video, score}
    }

    sort.Slice(scored, func(i, j int) bool {
        return scored[i].score > scored[j].score
    })

    final := rs.applyReranking(scored, user, count)
    rs.cache.Set(cacheKey, final, 5*time.Minute)

    return final, nil
}

type CandidateGenerator struct {
    collaborativeFiltering *CFModel
    videoDB                *PostgreSQL
}

func (cg *CandidateGenerator) Generate(ctx context.Context, user *User) []*Video {
    var candidates []*Video

    recent := cg.videoDB.GetRecentlyWatched(ctx, user.ID, 20)
    for _, video := range recent {
        similar := cg.collaborativeFiltering.GetSimilar(ctx, video.ID, 50)
        candidates = append(candidates, similar...)
    }

    subscriptions := cg.videoDB.GetSubscriptions(ctx, user.ID)
    for _, channel := range subscriptions {
        new := cg.videoDB.GetLatestFromChannel(ctx, channel.ID, 30)
        candidates = append(candidates, new...)
    }

    trending := cg.videoDB.GetTrending(ctx, user.Region, user.Language, 100)
    candidates = append(candidates, trending...)

    seen := make(map[string]bool)
    var unique []*Video
    for _, v := range candidates {
        if !seen[v.ID] {
            seen[v.ID] = true
            unique = append(unique, v)
        }
    }

    return unique
}

func (rm *RankingModel) Score(ctx context.Context, user *User, video *Video) float32 {
    userFeatures := extractUserFeatures(user)
    videoFeatures := extractVideoFeatures(video)
    features := append(userFeatures, videoFeatures...)
    return rm.model.Predict(features)
}
```

**Типичные ошибки.**
- Cold start: новому пользователю нечего рекомендовать.
- Filter bubble: рекомендовать только похожее.
- Медленный ранкинг → ML prefetch для top users.

**На интервью.**
- Объясни двухстадийный подход.
- Упомяни cold start challenge.
- Follow-up: «Как обновлять модель?» — daily retrain.
- Follow-up: «Как A/B тестировать?» — online experimentation.

---

### 9. Как спроектировать поиск по видео?

**Зачем спрашивают.** Поиск должен быть быстрым и релевантным. Elasticsearch — стандарт.

**Короткий ответ.** Индексировать метаданные (название, описание, теги) в Elasticsearch. BM25 для релевантности + фильтры. Кэш популярных запросов, ML ranking.

**Детальный разбор.**

```
USER QUERY: "machine learning tutorial"
           │
           ▼
    ┌──────────────┐
    │ Query Parser │  tokenize, remove stop words
    └──────┬───────┘
           │
    machine, learning, tutorial (tokens)
           │
           ▼
┌────────────────────────────────┐
│ Elasticsearch Query (BM25)     │
│ - Full text search             │
│ - Filters: duration, date      │
│ - Boost: freshness, views      │
└────────┬────────────────────────┘
         │
    10000s candidates
         │
         ▼
┌────────────────────────────────┐
│ Re-ranking & ML Model          │
│ - Personalization              │
│ - CTR prediction               │
└────┬───────────────────────────┘
     │
  Top 20 results
```

**Пример.**
```go
type SearchService struct {
    elasticClient *elasticsearch.Client
    queryCache    *Redis
}

func (ss *SearchService) Search(ctx context.Context, req *SearchRequest) (*SearchResult, error) {
    cacheKey := hashQuery(req)
    if cached, ok := ss.queryCache.Get(cacheKey); ok {
        return cached.(*SearchResult), nil
    }

    esQuery := ss.buildESQuery(req)

    searchResp, err := ss.elasticClient.Search(
        ss.elasticClient.Search.WithContext(ctx),
        ss.elasticClient.Search.WithIndex("videos"),
        ss.elasticClient.Search.WithBody(esQuery),
    )
    if err != nil {
        return nil, err
    }
    defer searchResp.Body.Close()

    var hits struct {
        Hits struct {
            Total struct {
                Value int64
            }
            Hits []struct {
                Source *Video
                Score  float32
            }
        }
    }

    json.NewDecoder(searchResp.Body).Decode(&hits)

    videos := make([]*Video, len(hits.Hits.Hits))
    for i, hit := range hits.Hits.Hits {
        video := hit.Source
        video.Score = ss.personalizeScore(hit.Score, req.UserID, video)
        videos[i] = video
    }

    result := &SearchResult{
        Videos: videos,
        Total:  hits.Hits.Total.Value,
    }
    ss.queryCache.Set(cacheKey, result, 1*time.Hour)

    return result, nil
}
```

**Типичные ошибки.**
- Отсутствие кэширования популярных запросов.
- Не учитывать typos — используй fuzzy matching.
- Плохая релевантность — нужна перестройка модели на CTR.

**На интервью.**
- Объясни BM25.
- Упомяни кэширование.
- Follow-up: «Как обрабатывать опечатки?» — fuzzy matching.
- Follow-up: «Как оптимизировать автозавершение?» — prefix index.

---

### 10. Как масштабировать видеопотоковую систему?

**Зачем спрашивают.** YouTube обслуживает триллионы просмотров. Нужно понимать bottlenecks.

**Короткий ответ.** Масштабирование требует: (1) горизонтальное расширение (мульти-регион CDN), (2) асинхронная обработка (очереди), (3) оптимизация стоимости (tiering, spot instances), (4) надежность (replication, failover).

**Детальный разбор.**

```
BOTTLENECK 1: UPLOAD BANDWIDTH
30,000 hours/day → need 150K concurrent uploaders
Solution: CDN upload points, resumable, parallel chunks

BOTTLENECK 2: TRANSCODING COST
$50K-100K/day
Solution: GPU acceleration, spot instances, smart queue

BOTTLENECK 3: STORAGE COSTS
5PB+ data = $115K/month on S3
Solution: Tiering, archival (Glacier), deduplication

BOTTLENECK 4: CDN BANDWIDTH
500 Pbps peak × $0.01-0.05/GB = $5-50M/month
Solution: Own CDN, ISP partnerships, aggressive caching

BOTTLENECK 5: DATABASE SCALABILITY
Billions of videos
Solution: Sharding, read replicas, separate DBs

BOTTLENECK 6: RECOMMENDATION LATENCY
Generate top 20 in <100ms with billions of videos
Solution: Pre-compute, candidate selection, caching
```

**Пример.**
```go
type ScaledVideoSystem struct {
    regions              map[string]*RegionCluster
    globalLB             *GlobalLoadBalancer
    cdnManager           *MultiCDNManager
    transcodingCluster   *TranscodingCluster
    videoShards          map[int]*VideoShard
}

func (sys *ScaledVideoSystem) RouteRequest(req *Request) *RegionCluster {
    userLocation := geoip.Lookup(req.ClientIP)
    nearestRegion := findNearestRegion(userLocation)
    return sys.regions[nearestRegion]
}

func (sys *ScaledVideoSystem) GetVideoShard(videoID string) *VideoShard {
    hash := murmur3.Sum64([]byte(videoID))
    shardIdx := int(hash % uint64(len(sys.videoShards)))
    return sys.videoShards[shardIdx]
}

type TranscodingCluster struct {
    jobQueue      *PriorityQueue
    workerPools   map[string]*WorkerPool
    costOptimizer *CostOptimizer
}

func (tc *TranscodingCluster) SubmitJob(job *TranscodeJob) error {
    priority := tc.estimateViralScore(job.VideoID)
    if priority < 0.3 {
        job.UseSpotInstance = true
    }
    tc.jobQueue.Push(job, priority)
    return nil
}

type CacheManager struct {
    edgeCache      *LocalCache
    regionalCache  *Redis
}

func (cm *CacheManager) GetSegment(videoID, quality, segmentNum string) ([]byte, error) {
    key := fmt.Sprintf("%s:%s:%s", videoID, quality, segmentNum)

    if data, ok := cm.edgeCache.Get(key); ok {
        return data, nil
    }

    if data, ok := cm.regionalCache.Get(key); ok {
        cm.edgeCache.Set(key, data, 1*time.Hour)
        return data, nil
    }

    data, err := cm.getFromOrigin(videoID, quality, segmentNum)
    if err != nil {
        return nil, err
    }

    cm.regionalCache.Set(key, data, 1*time.Week)
    if cm.shouldCacheAtEdge(videoID) {
        cm.edgeCache.Set(key, data, 1*time.Hour)
    }

    return data, nil
}
```

**Типичные ошибки.**
- Не планировать мульти-регион с самого начала.
- Кэшировать всё везде — неправильное управление памятью.
- Не обрабатывать failure режимов.
- Переплатить за on-demand.

**На интервью.**
- Объясни главные bottlenecks в порядке.
- Упомяни мульти-CDN strategy.
- Follow-up: «Как обрабатывать viral spike?» — burst queue, spot scaling.
- Follow-up: «Какой бюджет?» — $100M+/year.

---

## Summary

| Компонент | Challenge | Solution |
|-----------|-----------|----------|
| Upload | Большие файлы | Resumable, parallel chunks |
| Transcoding | $100K+/day | GPU, spot instances, queue |
| CDN | $50M+/month | Own CDN, caching |
| Storage | 5PB+ | Tiering, archival |
| Database | Billion videos | Sharding, replicas |
| Search | Latency | Elasticsearch, caching |
| Recommendations | ML compute | Two-stage, pre-compute |
| Live | Low latency | LL-HLS, WebRTC |
| DRM | Copyright | AES encryption, key delivery |

---

[← Назад к списку тем](./README.md)
