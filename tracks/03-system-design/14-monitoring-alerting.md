# 14 — Monitoring & Alerting System

Развёрнутые вопросы и ответы про проектирование систем мониторинга и алертинга: сбор метрик, временные ряды, логирование, аномалии, правила алертов, дашборды, распределённая трассировка, SLI/SLO/SLA, масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [13-distributed-id](./13-distributed-id.md) · Следующая тема: [15-ai-systems](./15-ai-systems.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Metrics** — измеримые величины состояния системы, которые характеризуют её производительность и здоровье. Существует три основных типа метрик: counters (счётчики, которые только растут, например количество обработанных запросов), gauges (калибры, которые могут расти и уменьшаться, например текущее использование памяти), и histograms (гистограммы, которые показывают распределение значений, например времена отклика). Правильно выбранные метрики позволяют понять что происходит в системе. Это основа для всего мониторинга.

**Time Series Database** — специализированная база данных, оптимизированная для хранения и аналитики временных рядов (последовательности данных с временными метками). Примеры включают Prometheus, InfluxDB и TimescaleDB. Эти БД очень эффективно сжимают данные (так как последовательные значения часто близки друг к другу) и могут быстро агрегировать данные по времени (например, найти среднее значение за час). Специализированные временные БД намного эффективнее обычных БД для этой задачи.

**Pull vs Push Model** — два противоположных подхода к сбору метрик. В pull модели (Prometheus-style) центральный сервер сам инициирует запросы к приложениям и собирает метрики через HTTP endpoint. В push модели приложения сами отправляют метрики на центральный коллектор. Pull проще управлять (единая конфигурация на сервере), но может не работать через firewall. Push работает везде и может быть быстрее, но требает правильной обработки на приложении стороне.

**SLI (Service Level Indicator)** — конкретная метрика, которая реально отражает опыт пользователя с сервисом. Примеры включают: процент успешных запросов (успешность), средний результат времени отклика (latency), или процент запросов обработанных за < 1 секунду. SLI должна быть измеримой и отражать то, что действительно важно для пользователей. Это основа для установления целей надёжности и отслеживания compliance с обещаниями.

**SLO (Service Level Objective)** — конкретная цель по SLI, которую организация обещает достичь. Примеры включают: "99.9% успешность запросов", "средняя latency < 200ms", или "95th percentile latency < 500ms". SLO привязан к SLI и управляет скоростью разработки: разработчики могут делать быстрые развёртывания пока остаётся error budget, но должны замедлиться перед крайним сроком.

**SLA (Service Level Agreement)** — контрактное обязательство между сервис-провайдером и клиентом, которое гарантирует достижение определённого уровня сервиса (часто основанного на SLO). Если провайдер не достигает SLA, он может быть обязан выплатить штраф или предоставить компенсацию клиентам. SLA более строга чем SLO, так как имеет финансовые последствия и часто включает дополнительные требования (например, техническую поддержку).

**Error Budget** — допустимое количество времени downtime (или неудачных операций) в месяц, которое может быть позволено при сохранении SLO. Например, если SLO составляет 99.9% доступность, то error budget = 43 минуты downtime в месяц. Это позволяет более быстрые развёртывания и экспериментирование пока бюджет не исчерпан. Когда бюджет на исходе, нужно замедлиться и сосредоточиться на надёжности.

**Alerting Rule** — условие, которое определяет когда нужно отправить уведомление на-duty инженеру. Примеры включают: "latency > 1 секунда", "error rate > 5%", или "CPU использование > 80%". Alerting rules должны быть тщательно настроены, чтобы ловить реальные проблемы без слишком большого количества false alarms. Критично для быстрого обнаружения и реагирования на проблемы.

**Anomaly Detection** — техника автоматического обнаружения нетипичного поведения системы, используя статистические методы или machine learning. Вместо установки фиксированных порогов (которые часто неправильны), система может учиться нормальному поведению и уведомлять когда поведение отличается. Это может ловить проблемы, которые простые пороги могут пропустить, например медленный деграда производительности, который не превышает порог за раз.

**Dashboard** — визуализация метрик и состояния системы в реальном времени, обычно на веб-странице или приложении. Хороший dashboard позволяет на-duty инженеру быстро увидеть проблему и её масштаб, например: какой сервис падает, какова его latency, и есть ли корреляция с изменениями в коде. Dashboards часто используются как первый шаг в отладке проблемы на production.

---

## Вопросы и разборы

### 1. Как спроектировать систему сбора метрик: pull vs push?

**Зачем спрашивают.** Выбор модели сбора метрик — критическое решение, влияющее на надёжность, задержку и сложность системы. Интервьюер проверяет понимание trade-offs и умение выбирать архитектуру под требования.

**Короткий ответ.** Pull-модель (Prometheus) позволяет централизованное управление и контроль, но требует доступности сервисов. Push-модель (StatsD, OpenTelemetry) работает везде, но рискует перегружать коллектор и требует буферизации. Выбирай по требованиям: pull для управляемой среды, push для микросервисов и облака.

**Детальный разбор.**

**Pull-модель (Prometheus-style):**
```
                    ┌──────────────────┐
                    │  Metrics Server  │
                    │   (Scraper)      │
                    └────────┬─────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
        ┌───▼──┐         ┌───▼──┐         ┌──▼───┐
        │ Svc1 │         │ Svc2 │         │ Svc3 │
        │ :8080│         │ :8080│         │:8080 │
        └──────┘         └──────┘         └──────┘
        /metrics         /metrics         /metrics

Сбор: каждые 15 секунд, параллельные запросы
```

**Преимущества pull:**
- Единая точка управления (что, когда, как часто собирать)
- Service discovery встроенный
- Защита от flooding (коллектор сам контролирует нагрузку)
- Легко добавить новый сервис
- Статусы scrape failures видны централизованно

**Недостатки pull:**
- Требует сетевой доступности от коллектора
- Firewall/NAT затрудняют (нужны reverse proxy, tunnels)
- Невозможна отправка метрик из offline систем
- Задержка до следующего scrape cycle (до 15 сек)

**Push-модель (StatsD/OpenTelemetry-style):**
```
┌──────┐    ┌──────┐    ┌──────┐
│ Svc1 │    │ Svc2 │    │ Svc3 │
└───┬──┘    └───┬──┘    └───┬──┘
    │           │           │
    └───────────┼───────────┘
                │
        ┌───────▼────────┐
        │  UDP/TCP Port  │
        │  :8125 (push)  │
        └───────┬────────┘
                │
        ┌───────▼────────┐
        │  Metrics       │
        │  Collector     │
        │  (Statsd, etc) │
        └───────┬────────┘
                │
        ┌───────▼────────┐
        │ Time Series DB │
        └────────────────┘
```

**Преимущества push:**
- Работает везде (нет firewall проблем)
- Низкая задержка (сразу после события)
- Декаплинг (сервис не ждёт ответа)
- Работает с offline/batch системами

**Недостатки push:**
- Flooding risk (клиент может спамить метриками)
- Нужна буферизация и flow control
- Требует конфигурирования у каждого клиента
- Потеря метрик при сбое коллектора (UDP)
- Сложнее мониторить сам collector

**Гибридный подход:**
```
┌──────┐          ┌──────────────┐
│ Svc1 │─ push ──►│  Receiver     │
└──────┘          │  (Buffer)     │
                  └───────┬───────┘
                          │
                  ┌───────▼───────┐
                  │    Kafka      │ ◄── decoupling
                  │    (Buffer)   │
                  └───────┬───────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
    ┌───▼────┐        ┌───▼────┐        ┌──▼───┐
    │ TSDB   │        │ Stream  │        │Alert │
    │ Writer │        │ Proc    │        │Engine│
    └────────┘        └────────┘        └──────┘
```

**Пример.**

```python
# Pull-модель (Prometheus client)
from prometheus_client import Counter, Histogram, start_http_server
import time

# Регистрируем метрики
http_requests = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'path', 'status']
)

request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    buckets=[0.1, 0.5, 1.0, 5.0]
)

# Сервис отвечает на /metrics
start_http_server(8000)

# В обработчике запроса
@app.route('/api/data')
def handle_request():
    with request_duration.time():
        result = process_data()
        http_requests.labels(
            method='GET',
            path='/api/data',
            status=200
        ).inc()
    return result

# Prometheus делает:
# GET http://service:8000/metrics каждые 15 сек
# Парсит текстовый формат:
# http_requests_total{method="GET",path="/api/data",status="200"} 42
```

```python
# Push-модель (StatsD client)
from statsd import StatsClient

statsd = StatsClient('localhost', 8125, prefix='myapp')

@app.route('/api/data')
def handle_request():
    start = time.time()

    try:
        result = process_data()
        duration = time.time() - start

        # Push метрики
        statsd.timing('http_request_duration_ms', duration * 1000)
        statsd.incr('http_requests_total')
        statsd.gauge('active_requests', active_count)

        return result
    except Exception as e:
        statsd.incr('errors_total')
        raise
```

```python
# Hybrid: Push с буферизацией и batch отправкой
import asyncio
from collections import defaultdict

class BufferedMetricsClient:
    def __init__(self, collector_url: str, batch_size=100, flush_interval=5):
        self.collector_url = collector_url
        self.batch_size = batch_size
        self.flush_interval = flush_interval
        self.buffer = []
        self.lock = asyncio.Lock()

    async def record_metric(self, name: str, value: float, labels: dict):
        metric = {
            'name': name,
            'value': value,
            'labels': labels,
            'timestamp': int(time.time() * 1000)
        }

        async with self.lock:
            self.buffer.append(metric)

            if len(self.buffer) >= self.batch_size:
                await self.flush()

    async def flush(self):
        if not self.buffer:
            return

        batch = self.buffer[:]
        self.buffer.clear()

        try:
            async with aiohttp.ClientSession() as session:
                await session.post(
                    f"{self.collector_url}/api/v1/metrics/write",
                    json={'metrics': batch},
                    timeout=5
                )
        except Exception as e:
            # Retry или log failure
            print(f"Failed to flush metrics: {e}")

    async def periodic_flush(self):
        while True:
            await asyncio.sleep(self.flush_interval)
            async with self.lock:
                await self.flush()
```

**Сравнительная таблица:**

| Критерий | Pull | Push |
|----------|------|------|
| Firewall friendly | Нет | Да |
| Latency | 15+ сек | <1 сек |
| Load control | Есть | Нужно реализовать |
| Discovery | Встроено | Нужна конфигурация |
| Потери при сбое | Нет (retry) | Да (UDP) |
| Сложность | Средняя | Высокая |

**Типичные ошибки.**
- Забыть про буферизацию в push-модели → flooding коллектора
- Не учесть firewall правила при pull → scrape failures
- Установить слишком малый interval → high cardinality explosion
- Не добавить retry logic в push → потеря метрик
- Мониторить metrics server только pull-моделью → не видно когда сам server падает

**На интервью.**
- Объясни, почему Prometheus выбрал pull, а не push
- Упомяни Remote Write API (Prometheus может быть source data для других TSDB)
- Уточняющий вопрос: «Как обработать случай, когда сервис временно недоступен?» → retry с backoff, staleness marking
- Уточняющий вопрос: «Как избежать high cardinality?» → ограничения на labels, cardinality limit alerts

---

### 2. Как хранить метрики в Time Series Database (TSDB)?

**Зачем спрашивают.** Метрики — это не обычные данные: много точек, специфические паттерны доступа, требования к компрессии. Понимание TSDB архитектуры критично для оптимизации.

**Короткий ответ.** TSDB организует данные по временным рядам (последовательность значений одной метрики с одним набором labels). Использует специализированную компрессию (delta-of-delta для timestamps, XOR для значений), columnar storage, и индексы по labels. Данные группируются в блоки (chunks), блоки в бэки́, старые данные downsampling'уются.

**Детальный разбор.**

**Структура временного ряда:**
```
Metric: http_requests_total{method="GET", service="api"}

Series ID = hash(metric_name + sorted(labels))
         = hash("http_requests_total" + "method=GET" + "service=api")
         = 0x7F3A2B1C

Точки данных:
┌─────────────────────────────────────────────────────┐
│ Timestamp      │ Value  │ Timestamp (delta) │        │
├─────────────────────────────────────────────────────┤
│ 1704067200     │ 1000   │                   │        │
│ 1704067260     │ 1050   │ +60               │ +50    │
│ 1704067320     │ 1120   │ +60 (stable!)     │ +70    │
│ 1704067380     │ 1200   │ +60               │ +80    │
│ 1704067440     │ 1265   │ +60               │ +65    │
└─────────────────────────────────────────────────────┘

Паттерн: timestamps часто имеют одинаковые дельты
         values изменяются более хаотично
```

**Компрессия (Gorilla algorithm):**

```
Timestamps: Delta-of-delta encoding
─────────────────────────────────────
Raw:     [t0, t0+60, t0+120, t0+180, t0+240]
Delta:   [60, 60, 60, 60]
D-o-D:   [60, 0, 0, 0]  ← очень сжимаемо!

Хранение:
- Если delta == previous_delta: 1 бит (0)
- Иначе: 2 бита (10) + новая дельта

Values: XOR encoding (для float64)
──────────────────────────────────
Идея: float64 часто отличаются только в нескольких битах

v1 = 3.14159265 = 0x400921FA00000000
v2 = 3.14159470 = 0x400921FB00000000
              XOR = 0x00000001 00000000  ← только 1 бит отличается!

Хранение:
- Если leading_zeros == prev: 1 бит (0) + trailing_zeros bits
- Иначе: 2-3 бита + new_leading_zeros + data
```

**Структура хранения (Prometheus/VictoriaMetrics style):**

```
┌─────────────────────────────────────────────────────┐
│            Memory (Ingestion Buffer)                 │
├─────────────────────────────────────────────────────┤
│ Series 1 → [sample, sample, ...]                     │
│ Series 2 → [sample, sample, ...]                     │
│ ...                                                   │
└─────────────────────────────────────────────────────┘
         │ Flush каждые 1-2 часа
         ▼
┌─────────────────────────────────────────────────────┐
│              Disk (TSDB Blocks)                      │
├─────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────┐   │
│ │ Block 1: 2024-01-01 00:00 - 02:00 (2h)       │   │
│ ├──────────────────────────────────────────────┤   │
│ │ meta.json:                                    │   │
│ │ {                                             │   │
│ │   "min_time": 1704067200000,                 │   │
│ │   "max_time": 1704074400000,                 │   │
│ │   "series_count": 125000                      │   │
│ │ }                                             │   │
│ │                                               │   │
│ │ index (indices/index):                        │   │
│ │ ┌──────────────────────────────────────────┐ │   │
│ │ │ Series ID → symbol offsets               │ │   │
│ │ │ Label indices:                           │ │   │
│ │ │   "method" → {"GET": [s1, s2, s5, ...]} │ │   │
│ │ │   "status" → {"200": [s1, s3, ...]}     │ │   │
│ │ └──────────────────────────────────────────┘ │   │
│ │                                               │   │
│ │ chunks (data/chunks):                         │   │
│ │ ┌──────────────────────────────────────────┐ │   │
│ │ │ Series 1: [compressed_chunk1] [chunk2]  │ │   │
│ │ │ Series 2: [compressed_chunk1] [chunk2]  │ │   │
│ │ │ ...                                       │ │   │
│ │ └──────────────────────────────────────────┘ │   │
│ └──────────────────────────────────────────────┘   │
│                                                      │
│ ┌──────────────────────────────────────────────┐   │
│ │ Block 2: 2024-01-01 02:00 - 04:00 (2h)       │   │
│ └──────────────────────────────────────────────┘   │
│                                                      │
│ ┌──────────────────────────────────────────────┐   │
│ │ Block 3 (compacted): 2024-01-01 00:00 - 8h  │   │
│ └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

Compaction: мёржит 4-8 блоков в 1 для лучшей компрессии
```

**Индексирование для queries:**

```
Индекс labels:
┌─────────────────────────────────────────┐
│ method="GET"  → [Series 1, 3, 7, ...]   │
│ method="POST" → [Series 2, 5, 9, ...]   │
│ status="200"  → [Series 1, 2, 3, ...]   │
│ status="500"  → [Series 4, 8, ...]      │
└─────────────────────────────────────────┘

Когда query: http_requests{method="GET", status="200"}
1. Найди series с method="GET"
2. Найди series с status="200"
3. Intersection → [Series 1, 3]
4. Fetch data для Series 1 и 3
```

**Пример.**

```python
# Time Series Storage Implementation
from dataclasses import dataclass
from typing import List
import struct
import zlib

@dataclass
class Sample:
    timestamp: int  # milliseconds
    value: float

class TSDBChunk:
    """Сжатый блок данных для одного временного ряда"""

    def __init__(self):
        self.samples: List[Sample] = []
        self.compressed_data: bytes = b''

    def add_sample(self, sample: Sample):
        self.samples.append(sample)

    def compress(self) -> bytes:
        """Gorilla compression"""
        if not self.samples:
            return b''

        # Sort by timestamp
        self.samples.sort(key=lambda s: s.timestamp)

        # Compress timestamps (delta-of-delta)
        timestamps = [s.timestamp for s in self.samples]
        delta_timestamps = self._compress_timestamps(timestamps)

        # Compress values (XOR)
        values = [s.value for s in self.samples]
        compressed_values = self._compress_values(values)

        # Pack together
        header = struct.pack('<I', len(self.samples))
        self.compressed_data = header + delta_timestamps + compressed_values
        return self.compressed_data

    def _compress_timestamps(self, timestamps: List[int]) -> bytes:
        """Delta-of-delta encoding"""
        data = []

        # First timestamp (raw)
        data.append(struct.pack('<Q', timestamps[0]))

        if len(timestamps) < 2:
            return b''.join(data)

        # First delta
        prev_delta = timestamps[1] - timestamps[0]
        data.append(struct.pack('<I', prev_delta))

        # Delta-of-delta for the rest
        bits = []
        for i in range(2, len(timestamps)):
            delta = timestamps[i] - timestamps[i-1]
            delta_of_delta = delta - prev_delta

            if delta_of_delta == 0:
                bits.append('0')  # Same as before
            else:
                bits.append('1')
                bits.append(bin(delta_of_delta)[2:])

            prev_delta = delta

        # Pack bits (simplified)
        return b''.join(data)

    def _compress_values(self, values: List[float]) -> bytes:
        """XOR compression для float64"""
        data = []

        # First value (raw)
        data.append(struct.pack('<d', values[0]))

        if len(values) < 2:
            return b''.join(data)

        # XOR для остальных
        prev_bits = struct.unpack('<Q', struct.pack('<d', values[0]))[0]

        for value in values[1:]:
            curr_bits = struct.unpack('<Q', struct.pack('<d', value))[0]
            xor_bits = prev_bits ^ curr_bits

            # Простое хранение (обычно с контролем leading/trailing zeros)
            if xor_bits == 0:
                data.append(b'\x00')  # No change
            else:
                data.append(struct.pack('<Q', xor_bits))

            prev_bits = curr_bits

        return b''.join(data)

class TSDBWriter:
    """Буферизирует и пишет метрики в блоки"""

    def __init__(self, storage_path: str, flush_interval_seconds: int = 3600):
        self.storage_path = storage_path
        self.flush_interval = flush_interval_seconds
        self.buffers = {}  # series_id → TSDBChunk
        self.label_index = {}  # label → set(series_ids)

    def write(self, metric_name: str, labels: dict, value: float, timestamp: int):
        """Записать одну метрику"""
        series_id = self._get_series_id(metric_name, labels)

        if series_id not in self.buffers:
            self.buffers[series_id] = TSDBChunk()
            self._update_label_index(series_id, labels)

        self.buffers[series_id].add_sample(Sample(timestamp, value))

    def flush(self):
        """Сбросить всё буферизированное в диск"""
        import os
        from datetime import datetime

        block_dir = os.path.join(
            self.storage_path,
            datetime.utcnow().strftime('%Y%m%d_%H%M%S')
        )
        os.makedirs(block_dir, exist_ok=True)

        # Сжать и написать chunks
        for series_id, chunk in self.buffers.items():
            compressed = chunk.compress()
            chunk_file = os.path.join(block_dir, f'{series_id}.chunk')
            with open(chunk_file, 'wb') as f:
                f.write(compressed)

        # Написать индекс
        self._write_index(block_dir)
        self._write_metadata(block_dir)

        # Очистить буфферы
        self.buffers.clear()

    def _get_series_id(self, metric_name: str, labels: dict) -> str:
        """Создать уникальный ID серии"""
        import hashlib

        sorted_labels = ''.join(
            f'{k}={v}' for k, v in sorted(labels.items())
        )
        key = f'{metric_name}|{sorted_labels}'
        return hashlib.md5(key.encode()).hexdigest()

    def _update_label_index(self, series_id: str, labels: dict):
        """Обновить индекс для быстрого поиска по labels"""
        for k, v in labels.items():
            label_key = f'{k}={v}'
            if label_key not in self.label_index:
                self.label_index[label_key] = set()
            self.label_index[label_key].add(series_id)

    def _write_index(self, block_dir: str):
        """Написать индекс в JSON"""
        import json
        index_file = os.path.join(block_dir, 'index.json')
        with open(index_file, 'w') as f:
            json.dump(
                {k: list(v) for k, v in self.label_index.items()},
                f
            )

    def _write_metadata(self, block_dir: str):
        """Написать metadata (время, кол-во серий)"""
        import json
        from datetime import datetime

        meta_file = os.path.join(block_dir, 'meta.json')
        with open(meta_file, 'w') as f:
            json.dump({
                'created_at': datetime.utcnow().isoformat(),
                'series_count': len(self.buffers),
                'samples_count': sum(
                    len(chunk.samples) for chunk in self.buffers.values()
                )
            }, f)
```

**Сравнительная таблица TSDB:**

| TSDB | Запись | Чтение | Компрессия | Масштабирование |
|------|--------|--------|------------|-----------------|
| Prometheus | Fast | Good | Gorilla | Single node |
| InfluxDB | Fast | Good | Delta + compression | Clustering (paid) |
| TimescaleDB | Medium | Best | PostgreSQL | Horizontal |
| VictoriaMetrics | Very fast | Good | Gorilla + | Excellent |
| ClickHouse | Medium | Excellent | Columnar | Excellent |

**Типичные ошибки.**
- Недостаточная компрессия → дорогое хранилище
- Неэффективные индексы → slow queries
- Забыть про high cardinality → explosion в памяти и storage
- Не downsampling старые данные → retained forever
- Мешать разные resolution в одном блоке → плохая компрессия

**На интервью.**
- Объясни, почему XOR compression лучше для float, чем delta
- Упомяни high cardinality problem (логи с user_id)
- Уточняющий вопрос: «Как обрабатывать queries который требуют данных из разных блоков?» → merging, buffering
- Уточняющий вопрос: «Как оптимизировать query на 10 миллионов серий?» → pre-aggregation, materialized views

---

### 3. Как реализовать систему логирования и агрегации логов?

**Зачем спрашивают.** Логи — критическая часть observability, но их объём огромный. Нужно понимать архитектуру сбора, буферизации, индексирования и поиска.

**Короткий ответ.** Логирование работает в 3 этапа: 1) сбор логов с сервисов (через agents как Filebeat/Fluent Bit), 2) буферизация в Kafka для decoupling, 3) индексирование и хранение (Elasticsearch/ELK stack). На каждом этапе нужна обработка (парсинг, обогащение), фильтрация и маршрутизация.

**Детальный разбор.**

**Log Aggregation Pipeline:**

```
┌──────┐  ┌──────┐  ┌──────┐
│ Svc1 │  │ Svc2 │  │ Svc3 │
│ logs │  │ logs │  │ logs │
└───┬──┘  └───┬──┘  └───┬──┘
    │         │         │
    │ stdout/ │         │
    │ files   │         │
    │         │         │
    └─────────┼─────────┘
              │
      ┌───────▼────────┐
      │  Log Collector │
      │ (Filebeat,     │
      │  Fluent Bit)   │
      └───────┬────────┘
              │
    ┌─────────▼──────────┐
    │  Buffer/Queue      │
    │  (Kafka)           │
    │  - Decouple write  │
    │  - Replay          │
    │  - Distribution    │
    └─────────┬──────────┘
              │
    ┌─────────┴──────────┐
    │                    │
┌───▼────┐         ┌────▼──┐
│ Stream  │         │ Batch │
│Processing         │ Import│
│ (Real-time)       │       │
└───┬────┘         └────┬───┘
    │                   │
    └───────┬───────────┘
            │
    ┌───────▼────────┐
    │ Indexing &     │
    │ Enrichment     │
    │ (Logstash)     │
    └───────┬────────┘
            │
    ┌───────▼────────┐
    │  Search Index  │
    │ (Elasticsearch)│
    └───────┬────────┘
            │
    ┌───────▼────────┐
    │  Visualization │
    │  (Kibana)      │
    └────────────────┘
```

**Форматы логов:**

```
Сырой текст:
2024-01-27 10:15:42 ERROR [api-service] Request failed: timeout after 5s

Структурированный JSON:
{
  "timestamp": "2024-01-27T10:15:42.123Z",
  "level": "ERROR",
  "service": "api-service",
  "message": "Request failed: timeout after 5s",
  "trace_id": "abc123def456",
  "span_id": "xyz789",
  "duration_ms": 5000,
  "http_status": 408,
  "request_path": "/api/users",
  "user_id": "user_123",
  "tags": {"region": "us-east", "env": "prod"}
}

Преимущества JSON:
- Поля доступны для запросов
- Сортировка, фильтрация, агрегация
- Correlation ID для трейсинга
- Контекст (user, request, etc)
```

**Log Collection & Processing:**

```python
# 1. Collector (на каждом хосте)
# Filebeat/Fluent Bit конфиг:
"""
input {
  file {
    path => "/var/log/app/*.log"
    start_position => "beginning"
    codec => json
  }
}

filter {
  # Parse timestamp
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }

  # Добавить метаданные
  mutate {
    add_field => {
      "[@metadata][index_name]" => "logs-%{+YYYY.MM.dd}"
      "host" => "${HOSTNAME}"
      "environment" => "production"
    }
  }

  # Обогащение (enrichment)
  translate {
    field => "error_code"
    destination => "error_description"
    dictionary => {
      "001" => "Timeout"
      "002" => "Connection refused"
    }
  }

  # Филтрация (drop spammy logs)
  if [message] =~ /health_check/ {
    drop { }
  }
}

output {
  kafka {
    codec => json
    topic_id => "logs"
    bootstrap_servers => "kafka:9092"
    compression_type => "snappy"
  }
}
"""

# 2. Stream Processing (Kafka Consumer)
from kafka import KafkaConsumer
import json

class LogProcessor:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'logs',
            bootstrap_servers=['kafka:9092'],
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            group_id='log-processor'
        )

    async def process(self):
        batch = []

        for message in self.consumer:
            log = message.value

            # Обогащение
            log['@timestamp'] = self._parse_timestamp(log.get('timestamp'))
            log['service'] = self._extract_service(log)
            log['severity_level'] = self._normalize_level(log.get('level'))

            # Фильтрация по severity
            if log['severity_level'] < 2:  # DEBUG/INFO
                if not self._should_keep(log):
                    continue

            batch.append(log)

            # Batch write to Elasticsearch
            if len(batch) >= 1000:
                await self.write_to_es(batch)
                batch.clear()

        # Flush remaining
        if batch:
            await self.write_to_es(batch)

    async def write_to_es(self, logs):
        """Write batch to Elasticsearch"""
        actions = []
        for log in logs:
            action = {
                "index": {
                    "_index": f"logs-{log['service']}-{self._date_str()}",
                    "_type": "_doc"
                }
            }
            actions.append(json.dumps(action))
            actions.append(json.dumps(log))

        # Bulk insert
        body = '\n'.join(actions) + '\n'
        await self.es_client.bulk(body=body)
```

**Elasticsearch Query & Analysis:**

```json
// Find errors in last 1 hour
GET logs-api-2024.01.27/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "ERROR" } },
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "errors_by_service": {
      "terms": {
        "field": "service.keyword",
        "size": 10
      },
      "aggs": {
        "errors_by_type": {
          "terms": {
            "field": "error_type.keyword",
            "size": 5
          }
        }
      }
    }
  }
}

// Find slow requests
GET logs-*/_search
{
  "query": {
    "range": {
      "duration_ms": { "gte": 1000 }
    }
  },
  "sort": [
    { "duration_ms": { "order": "desc" } }
  ],
  "size": 100
}

// Correlate by trace_id
GET logs-*/_search
{
  "query": {
    "match": {
      "trace_id": "abc123def456"
    }
  },
  "sort": [
    { "@timestamp": { "order": "asc" } }
  ]
}
```

**Log Retention & Archival:**

```python
class LogRetentionManager:
    """Управляет сохранением и архивацией логов"""

    RETENTION_POLICIES = {
        'ERROR': 30,      # 30 дней
        'WARNING': 14,    # 14 дней
        'INFO': 7,        # 7 дней
        'DEBUG': 1        # 1 день
    }

    async def apply_retention(self):
        """Удалить старые логи"""
        for level, days in self.RETENTION_POLICIES.items():
            cutoff_date = (datetime.utcnow() - timedelta(days=days))

            # Найти и удалить индексы
            indices = await self.es_client.indices.get(
                index=f"logs-*-{cutoff_date.strftime('%Y.%m.%d')}"
            )

            for index in indices:
                await self.es_client.indices.delete(index=index)

    async def archive_to_s3(self):
        """Архивировать логи в S3 для долгосрочного хранения"""
        week_ago = datetime.utcnow() - timedelta(days=7)
        index_pattern = f"logs-*-{week_ago.strftime('%Y.%m.%d')}"

        # Export index to file
        body = await self.es_client.search(
            index=index_pattern,
            size=10000  # Iterate with scroll for large datasets
        )

        # Compress and upload to S3
        import gzip
        compressed = gzip.compress(json.dumps(body).encode())

        await self.s3_client.put_object(
            Bucket='logs-archive',
            Key=f"logs/{week_ago.isoformat()}.json.gz",
            Body=compressed
        )
```

**Типичные ошибки.**
- Структурировать всё в JSON сразу → production log storm при дебаге
- Не фильтровать спамные логи (health checks) → дорогое хранилище
- Забыть про correlation IDs (trace_id) → невозможно найти root cause
- Недостаточный буфффер в Kafka → потеря логов при peak нагрузке
- Низкая retention для нужных логов → нельзя debug старые инциденты

**На интервью.**
- Объясни, почему буферизация в Kafka критична
- Упомяни structured logging и importance of trace_id
- Уточняющий вопрос: «Как обработать 1TB логов в день?» → filtering, sampling, archival
- Уточняющий вопрос: «Как избежать PII в логах?» → data masking, redaction rules

---

### 4. Как обнаруживать аномалии в метриках?

**Зачем спрашивают.** Аномалия детекшн отличается от простого alerting. Нужно понимать statistical methods, baseline computation, и handling false positives.

**Короткий ответ.** Anomaly detection сравнивает текущие значения с историческим baseline. Простой метод: threshold based (if value > mean + 3*sigma). Лучше: seasonal decomposition (отделить trend, seasonal и anomaly компоненты), или machine learning (ARIMA, Isolation Forest). Главное — избежать false positives на legit spikes.

**Детальный разбор.**

**Методы детекции аномалий:**

```
┌─────────────────────────────────────────┐
│  1. Static Threshold (простейший)        │
├─────────────────────────────────────────┤
│ alert if value > 100                    │
│                                         │
│ ┌─────────────────────────────────────┐│
│ │  Normal          │ Anomaly           ││
│ │  50, 60, 70      │ 150, 160, 200     ││
│ └─────────────────────────────────────┘│
│                                         │
│ ✓ Простая реализация                   │
│ ✗ Не учитывает patterns                │
│ ✗ Много false positives при спайках    │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  2. Statistical: σ (Sigma) Based         │
├─────────────────────────────────────────┤
│ mean = average(last 7 days)             │
│ std = stddev(last 7 days)               │
│ if |value - mean| > 3*std → ANOMALY    │
│                                         │
│     3σ above                            │
│        ↑                                │
│        │   Anomaly zone                 │
│ ──────┼──────────── mean + 3σ          │
│       │   Normal    │                   │
│ ──────┼─────────────┼─────── mean       │
│       │   Normal    │                   │
│ ──────┼──────────── mean - 3σ          │
│        │   Anomaly zone                 │
│        ↓                                │
│    3σ below                             │
│                                         │
│ ✓ Учитывает volatility                 │
│ ✓ Авто-адаптирующиеся пороги          │
│ ✗ Не учитывает daily/weekly паттерны   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  3. Seasonal Decomposition (STL)         │
├─────────────────────────────────────────┤
│ Разложить: T = Trend + Seasonal + Rest  │
│                                         │
│ Original:                               │
│ ┌─────────────────────────────────────┐│
│ │  /╲  /╲  /╲  /╲  /╲  /╲ Spikes!   ││
│ │ /  ╲/  ╲/  ╲/  ╲/  ╲/  \           ││
│ └─────────────────────────────────────┘│
│ Trend (long-term):                      │
│ ┌─────────────────────────────────────┐│
│ │  ─── gradually increasing ───────   ││
│ └─────────────────────────────────────┘│
│ Seasonal (daily pattern):               │
│ ┌─────────────────────────────────────┐│
│ │  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲        ││
│ │╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱  \       ││
│ └─────────────────────────────────────┘│
│ Residuals (anomalies):                  │
│ ┌─────────────────────────────────────┐│
│ │   ↓   ↓   ↓               ↑         ││
│ │ ─────────────────────────────────   ││
│ └─────────────────────────────────────┘│
│                                         │
│ ✓ Учитывает все паттерны               │
│ ✓ Точный детект реальных аномалий     │
│ ✗ Нужна история (1+ месяца данных)    │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  4. Machine Learning (ARIMA, Prophet)   │
├─────────────────────────────────────────┤
│ Предсказать значение → сравнить        │
│                                         │
│ ┌─────────────────────────────────────┐│
│ │Forecast ═══════ Actual ↑ (anomaly)  ││
│ │       ════════ ────────             ││
│ │  Confidence       Normal             ││
│ │  interval         region             ││
│ └─────────────────────────────────────┘│
│                                         │
│ ✓ Мощные методы                        │
│ ✓ Обрабатывает тренды и сезоны        │
│ ✗ Требует ML expertise                │
│ ✗ Медленнее других методов             │
└─────────────────────────────────────────┘
```

**Реализация:**

```python
import numpy as np
from scipy import stats
from sklearn.ensemble import IsolationForest

class AnomalyDetector:
    """Детектор аномалий с несколькими методами"""

    def __init__(self, method='sigma', lookback_days=7):
        self.method = method
        self.lookback_days = lookback_days
        self.history = {}

    def add_metric(self, series_id: str, timestamp: int, value: float):
        """Добавить метрику в историю"""
        if series_id not in self.history:
            self.history[series_id] = []
        self.history[series_id].append({
            'timestamp': timestamp,
            'value': value
        })

    def detect(self, series_id: str, value: float) -> dict:
        """Обнаружить аномалию"""
        if self.method == 'sigma':
            return self._detect_sigma(series_id, value)
        elif self.method == 'seasonal':
            return self._detect_seasonal(series_id, value)
        elif self.method == 'ml':
            return self._detect_ml(series_id, value)

    def _detect_sigma(self, series_id: str, value: float) -> dict:
        """Метод сигма (3-sigma rule)"""
        data = self.history.get(series_id, [])
        if len(data) < 10:
            return {'is_anomaly': False, 'reason': 'insufficient_data'}

        # Последние 7 дней
        cutoff = time.time() - (self.lookback_days * 86400)
        recent = [d['value'] for d in data if d['timestamp'] > cutoff]

        mean = np.mean(recent)
        std = np.std(recent)

        deviation = abs(value - mean) / (std + 1e-6)  # Избежать деления на 0

        is_anomaly = deviation > 3  # 3-sigma

        return {
            'is_anomaly': is_anomaly,
            'mean': mean,
            'std': std,
            'deviation_sigma': deviation,
            'confidence': min(1.0, deviation / 3)
        }

    def _detect_seasonal(self, series_id: str, value: float) -> dict:
        """Seasonal decomposition (STL)"""
        from statsmodels.tsa.seasonal import seasonal_decompose

        data = self.history.get(series_id, [])
        if len(data) < 100:  # Need enough history
            return self._detect_sigma(series_id, value)

        # Prepare time series
        values = [d['value'] for d in data[-1000:]]  # Last 1000 points

        try:
            # Decompose: T = Trend + Seasonal + Residual
            result = seasonal_decompose(
                values,
                model='additive',
                period=24  # 24 часа (hourly data)
            )

            # Ожидаемое значение = trend + seasonal
            expected = result.trend[-1] + result.seasonal[-1]
            residual_std = np.std(result.resid.dropna())

            error = abs(value - expected)
            is_anomaly = error > (3 * residual_std)

            return {
                'is_anomaly': is_anomaly,
                'expected': expected,
                'actual': value,
                'error': error,
                'residual_std': residual_std
            }
        except Exception as e:
            # Fallback на sigma
            return self._detect_sigma(series_id, value)

    def _detect_ml(self, series_id: str, value: float) -> dict:
        """Machine Learning (Isolation Forest)"""
        data = self.history.get(series_id, [])
        if len(data) < 50:
            return self._detect_sigma(series_id, value)

        values = np.array([d['value'] for d in data]).reshape(-1, 1)

        # Train Isolation Forest
        clf = IsolationForest(contamination=0.05, random_state=42)
        clf.fit(values)

        # Predict (-1 = anomaly, 1 = normal)
        prediction = clf.predict(np.array([[value]]))[0]

        # Get anomaly score (lower = more anomalous)
        score = clf.score_samples(np.array([[value]]))[0]

        return {
            'is_anomaly': prediction == -1,
            'score': score,
            'method': 'isolation_forest'
        }

class AdaptiveThreshold:
    """Адаптивные пороги, учитывающие time-of-day"""

    def __init__(self):
        self.hourly_stats = {}  # hour → {mean, std}

    def learn(self, timestamp: int, value: float):
        """Изучить паттерны по часам дня"""
        hour = (timestamp // 3600) % 24

        if hour not in self.hourly_stats:
            self.hourly_stats[hour] = {'values': []}

        self.hourly_stats[hour]['values'].append(value)

        # Пересчитать stats
        vals = self.hourly_stats[hour]['values']
        self.hourly_stats[hour]['mean'] = np.mean(vals)
        self.hourly_stats[hour]['std'] = np.std(vals)

    def is_anomaly(self, timestamp: int, value: float) -> bool:
        """Проверить с adaptive threshold"""
        hour = (timestamp // 3600) % 24

        if hour not in self.hourly_stats:
            return False  # No data for this hour

        stats = self.hourly_stats[hour]
        mean = stats.get('mean', 0)
        std = stats.get('std', 1)

        deviation = abs(value - mean) / (std + 1e-6)
        return deviation > 3
```

**Пример интеграции с alerting:**

```python
class AdaptiveAlertingEngine:
    def __init__(self, anomaly_detector):
        self.detector = anomaly_detector
        self.alert_rules = []

    async def evaluate(self, metric):
        # Detect anomaly
        result = self.detector.detect(
            metric['series_id'],
            metric['value']
        )

        if not result['is_anomaly']:
            return None

        # Check alert rules
        for rule in self.alert_rules:
            if self._matches(metric, rule):
                # Get baseline for context
                baseline = self.detector.get_baseline(metric['series_id'])

                # Create alert with context
                alert = {
                    'rule': rule.name,
                    'severity': self._calculate_severity(result),
                    'message': f"Anomaly detected: {metric['value']} (baseline: {baseline})",
                    'context': result,
                    'timestamp': metric['timestamp']
                }

                return alert

        return None

    def _calculate_severity(self, detection_result):
        """Severity зависит от magnitude аномалии"""
        if detection_result.get('deviation_sigma', 0) > 6:
            return 'critical'
        elif detection_result.get('deviation_sigma', 0) > 4:
            return 'warning'
        else:
            return 'info'
```

**Типичные ошибки.**
- Не учитывать seasonality → false positives в бизнес-часы
- Использовать static thresholds → много false positives при легитных спайках
- Недостаточно истории → неточные baseline'ы
- Не валидировать аномалии вручную → обучение на ошибках
- Забыть про outliers в baseline → смещённые пороги

**На интервью.**
- Объясни разницу между static и adaptive thresholds
- Упомяни importance of history и cold-start problem
- Уточняющий вопрос: «Как избежать alert fatigue?» → grouping, deduplication, на практике только critical alerting
- Уточняющий вопрос: «Как обнаруживать медленные аномалии (gradual degradation)?» → trend analysis, rate of change

---

### 5. Как спроектировать систему алертов и правил?

**Зачем спрашивают.** Alerting — это не просто condition check. Нужно управление состоянием (pending/firing/resolved), дедупликация, маршрутизация, silencing, и integration с разными channels.

**Короткий ответ.** Alert rule состоит из expression (query + threshold), evaluation interval, duration before firing (для избежания flapping). Alerts имеют состояние: pending (условие верно < duration), firing (условие верно >= duration), resolved (условие false). Есть deduplication по label'ам, routing rules, silencing, и integration с PagerDuty/Slack/Email.

**Детальный разбор.**

**Alert State Machine:**

```
                    ┌──────────────────┐
                    │     Initial      │
                    │    (no alert)    │
                    └────────┬─────────┘
                             │
                      Condition becomes true
                             │
                    ┌────────▼─────────┐
                    │     Pending      │
                    │   (waiting for   │
                    │   'for' duration)│
                    └────────┬─────────┘
                             │
                    Duration elapsed OR
                    Condition stays true
                             │
                    ┌────────▼─────────┐
                    │     Firing       │  ◄── Send notification
                    │   (alert sent)   │
                    └────────┬─────────┘
                             │
                      Condition becomes false
                             │
                    ┌────────▼──────────┐
                    │    Resolved       │  ◄── Send resolved notification
                    │ (clean up state)  │
                    └────────┬──────────┘
                             │
                    ┌────────▼─────────┐
                    │     Initial      │
                    │    (no alert)    │
                    └──────────────────┘

Важно:
- Pending → Firing: только если duration прошла
- Firing → Pending: если условие false, но alert ещё существует
- Firing → Resolved: чистая history
```

**Alert Rules Definition:**

```yaml
# Alert rule file (alert-rules.yml)
groups:
  - name: service_alerts
    interval: 30s  # Evaluation interval

    rules:
      # Rule 1: High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 5m                    # Duration before firing
        labels:
          severity: critical
          team: platform
          service: api
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

      # Rule 2: High memory usage
      - alert: HighMemoryUsage
        expr: |
          process_resident_memory_bytes /
          (1024 * 1024 * 1024) > 4  # 4GB
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory: {{ $value }}GB"

      # Rule 3: Service unavailable
      - alert: ServiceDown
        expr: up{job="api"} == 0
        for: 1m
        labels:
          severity: critical
          team: oncall
        annotations:
          summary: "Service {{ $labels.instance }} is down"

  - name: business_alerts
    interval: 1m

    rules:
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 0.5  # 500ms
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High P99 latency"

# Alert routing & silencing
rule_files:
  - 'alert-rules.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# AlertManager конфиг (alertmanager.yml)
global:
  resolve_timeout: 5m

route:
  # Default routing
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s     # Wait before first notification
  group_interval: 10m # Wait before next notification in group
  repeat_interval: 4h # Repeat if not resolved

  # Routing rules
  routes:
    # Critical alerts → PagerDuty immediately
    - match:
        severity: critical
      receiver: 'pagerduty'
      group_wait: 0s      # Send immediately
      repeat_interval: 5m # Repeat often

    # Warning → Slack
    - match:
        severity: warning
      receiver: 'slack'

    # Platform team → specific channel
    - match:
        team: platform
      receiver: 'platform-team'
      group_by: ['alertname']

receivers:
  - name: 'default'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#alerts'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_SERVICE_KEY}'
        description: '{{ .GroupLabels.alertname }}'

  - name: 'platform-team'
    slack_configs:
      - api_url: '${SLACK_PLATFORM_WEBHOOK}'
        channel: '#platform-alerts'

# Silencing & Inhibition
inhibit_rules:
  # Don't alert on pod errors if node is down
  - source_match:
      severity: 'critical'
      alertname: 'NodeDown'
    target_match_re:
      alertname: 'Pod.*'
    equal: ['node']

# Silences (manually added)
silences:
  - matchers:
      - name: alertname
        value: HighMemoryUsage
        isRegex: false
    start_time: 2024-01-27T14:00:00Z
    end_time: 2024-01-27T15:00:00Z
    comment: 'Maintenance window'
```

**Реализация Alert Engine:**

```python
from datetime import datetime, timedelta
import asyncio

class AlertState:
    def __init__(self, rule_name: str, labels: dict):
        self.rule_name = rule_name
        self.labels = labels
        self.status = 'pending'  # pending, firing, resolved
        self.started_at = None
        self.last_fired_at = None
        self.last_sent_at = None
        self.fire_duration = None

class AlertingEngine:
    def __init__(self, rules_file: str):
        self.rules = self._load_rules(rules_file)
        self.states = {}  # {alert_key} → AlertState
        self.notification_queue = asyncio.Queue()

    async def evaluation_loop(self):
        """Main evaluation loop"""
        while True:
            for rule_group in self.rules:
                # Evaluate each rule
                for rule in rule_group['rules']:
                    await self.evaluate_rule(rule)

            # Sleep until next evaluation
            await asyncio.sleep(10)  # Typical: 15-30 seconds

    async def evaluate_rule(self, rule: dict):
        """Evaluate single rule"""
        try:
            # Execute PromQL expression
            results = await self.tsdb.query(rule['expr'])

            # Check each result
            for result in results:
                alert_key = self._get_alert_key(rule, result)
                state = self.states.get(alert_key)

                # Condition is true
                if not state:
                    # First time this alert fired
                    state = AlertState(rule['alert'], result.labels)
                    state.started_at = datetime.utcnow()
                    self.states[alert_key] = state

                # Check if firing duration elapsed
                pending_duration = datetime.utcnow() - state.started_at
                for_duration = self._parse_duration(rule.get('for', '0s'))

                if pending_duration >= for_duration:
                    if state.status == 'pending':
                        # Transition to firing
                        state.status = 'firing'
                        state.last_fired_at = datetime.utcnow()

                        await self.send_notification(
                            state, rule, result, 'firing'
                        )
                else:
                    # Still pending
                    state.status = 'pending'

            # Condition is false - resolve alerts
            for alert_key in list(self.states.keys()):
                if self._key_in_results(alert_key, results):
                    continue

                state = self.states[alert_key]
                if state.status == 'firing':
                    state.status = 'resolved'
                    rule = self._find_rule_for_alert(alert_key)

                    await self.send_notification(
                        state, rule, None, 'resolved'
                    )

                    # Clean up resolved alert
                    del self.states[alert_key]

        except Exception as e:
            # Log evaluation error
            print(f"Error evaluating rule {rule.get('alert')}: {e}")

    async def send_notification(self, state: AlertState, rule: dict,
                               result, event_type: str):
        """Send notification"""

        # Check for duplicate/repeat
        if event_type == 'firing':
            # Group wait: don't send if another same type just sent
            if state.last_sent_at:
                time_since_sent = datetime.utcnow() - state.last_sent_at
                if time_since_sent < timedelta(minutes=10):
                    return  # Wait longer before repeat

        # Prepare notification
        notification = {
            'alertname': rule['alert'],
            'status': event_type,
            'labels': {**state.labels, **rule.get('labels', {})},
            'annotations': rule.get('annotations', {}),
            'timestamp': datetime.utcnow().isoformat(),
            'group_labels': self._extract_group_labels(rule, state),
        }

        # Route based on rules
        receivers = self._route_alert(notification)

        # Queue for sending
        for receiver in receivers:
            await self.notification_queue.put({
                'receiver': receiver,
                'notification': notification
            })

        state.last_sent_at = datetime.utcnow()

    def _route_alert(self, alert: dict) -> list:
        """Route alert to receivers based on routing rules"""
        receivers = []

        # Extract routing key labels
        severity = alert['labels'].get('severity', 'warning')
        team = alert['labels'].get('team', 'default')

        # Check routing rules
        if severity == 'critical':
            receivers.append({
                'type': 'pagerduty',
                'service_key': '${PAGERDUTY_KEY}'
            })

        if team == 'platform':
            receivers.append({
                'type': 'slack',
                'channel': '#platform-alerts'
            })

        # Default
        if not receivers:
            receivers.append({
                'type': 'slack',
                'channel': '#alerts'
            })

        return receivers

    def _get_alert_key(self, rule: dict, result) -> str:
        """Unique key for alert instance"""
        # Group by: alertname + specified labels
        group_labels = rule.get('group_by', ['alertname'])
        key_parts = [rule['alert']]

        for label in group_labels:
            value = result.labels.get(label, '')
            key_parts.append(f"{label}={value}")

        return '|'.join(key_parts)

    def _parse_duration(self, duration_str: str) -> timedelta:
        """Parse '5m', '10s', '1h', etc."""
        value = int(duration_str[:-1])
        unit = duration_str[-1]

        if unit == 's':
            return timedelta(seconds=value)
        elif unit == 'm':
            return timedelta(minutes=value)
        elif unit == 'h':
            return timedelta(hours=value)
        else:
            raise ValueError(f"Unknown duration unit: {unit}")

    def _load_rules(self, rules_file: str) -> list:
        import yaml
        with open(rules_file, 'r') as f:
            return yaml.safe_load(f).get('groups', [])
```

**Типичные ошибки.**
- Слишком короткий `for` duration → flapping alerts (constantly firing/resolving)
- Слишком длинный → задержка в обнаружении проблемы
- Забыть про `group_wait` → слишком много уведомлений
- Неправильная маршрутизация → alert идёт не туда
- Не добавить silencing → спам во время maintenance

**На интервью.**
- Объясни state machine и почему pending нужен
- Упомяни grouping, deduplication, repeat_interval
- Уточняющий вопрос: «Как избежать alert fatigue?» → intelligent routing, silencing, runbooks
- Уточняющий вопрос: «Как обрабатывать alert storms?» → inhibition, deduplication, rate limiting

---

### 6. Как спроектировать информативные дашборды для мониторинга?

**Зачем спрашивают.** Dashboard — интерфейс для observability. Нужно понимать информационную архитектуру, выбор метрик, компоновку, и производительность визуализации.

**Короткий ответ.** Dashboard должен отвечать на вопрос: что происходит с системой? Используй RED method (Rate, Errors, Duration) для user-facing сервисов, USE method (Utilization, Saturation, Errors) для ресурсов. Организуй панели иерархически: обзорный dashboard (status, key metrics), детальные dashboards (per-service, per-component). Кэшируй результаты запросов, используй downsampled данные для больших временных диапазонов.

**Детальный разбор.**

**Dashboard Design Principles:**

```
┌──────────────────────────────────────────────────────┐
│           Service Status Dashboard                   │
├──────────────────────────────────────────────────────┤
│ ┌───────────────┬─────────────┬──────────────────┐  │
│ │   Health      │   Uptime    │  Error Rate      │  │
│ │  Status       │   99.95%    │  0.02%           │  │
│ └───────────────┴─────────────┴──────────────────┘  │
│                                                      │
│ Key Metrics (Last 1h):                              │
│ ┌────────────────────────────────────────────────┐  │
│ │ QPS        │ Latency (p99)  │ Memory Usage    │  │
│ │ 10K/s      │ 250ms          │ 2.3GB (58%)     │  │
│ └────────────────────────────────────────────────┘  │
│                                                      │
│ Alerts:                                             │
│ ┌────────────────────────────────────────────────┐  │
│ │ ⚠ HighLatencyP99 (15 mins)                      │  │
│ │ 🔴 HighMemoryUsage (2 mins)                     │  │
│ └────────────────────────────────────────────────┘  │
│                                                      │
│ Request Rate (24h):                                 │
│ ┌────────────────────────────────────────────────┐  │
│ │     /╲  /╲      /╲  /╲      /╲  /╲    ↑       │  │
│ │    /  ╲/  ╲    /  ╲/  ╲    /  ╲/  ╲   │ 10K  │  │
│ │────────────────────────────────────────────    │  │
│ └────────────────────────────────────────────────┘  │
│                                                      │
│ Error Rate (24h):                                   │
│ ┌────────────────────────────────────────────────┐  │
│ │  ─────────────┬──────────────────────────      │  │
│ │              ▲ (incident)                      │  │
│ │             ╱ ╲                                │  │
│ │            ╱   ╲───────────────────────        │  │
│ └────────────────────────────────────────────────┘  │
│                                                      │
│ [API Details] [Database] [Cache] [Queue]           │
└──────────────────────────────────────────────────────┘
```

**RED Method для User-Facing Services:**

```
┌────────────────────────────────────────┐
│ R - Rate: Requests per second           │
│ ├─ Total QPS (все endpoints)            │
│ ├─ QPS per endpoint (/api/users, etc)   │
│ └─ QPS per status (2xx, 4xx, 5xx)       │
├────────────────────────────────────────┤
│ E - Errors: Error rate                  │
│ ├─ Total error rate (5xx / total)       │
│ ├─ Errors per endpoint                  │
│ └─ Error types (timeout, 500, etc)      │
├────────────────────────────────────────┤
│ D - Duration: Latency                   │
│ ├─ p50, p95, p99 latency                │
│ ├─ Per endpoint                         │
│ └─ Histogram of response times          │
└────────────────────────────────────────┘
```

**USE Method для Resource-Heavy Services:**

```
┌────────────────────────────────────────┐
│ U - Utilization: % time busy            │
│ ├─ CPU: 45%                             │
│ ├─ Memory: 60%                          │
│ ├─ Disk I/O: 30%                        │
│ └─ Network: 20%                         │
├────────────────────────────────────────┤
│ S - Saturation: queue depth             │
│ ├─ CPU runqueue length: 2.1             │
│ ├─ Disk I/O queue: 5                    │
│ └─ Network buffer: 10MB                 │
├────────────────────────────────────────┤
│ E - Errors: error rate                  │
│ ├─ I/O errors: 0.01%                    │
│ ├─ Dropped packets: 0                   │
│ └─ Memory errors: 0                     │
└────────────────────────────────────────┘
```

**Dashboard Hierarchy:**

```
                    ┌─────────────────┐
                    │ Company Health   │
                    │ Dashboard       │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐        ┌────▼────┐
    │ Platform │         │ Backend  │        │ Mobile  │
    │ Team     │         │ Services │        │ App     │
    │ Board    │         │ Board    │        │ Board   │
    └────┬────┘         └────┬────┘        └────┬────┘
         │                   │                   │
    ┌────▼──────────┐   ┌────▼──────────┐  ┌────▼──────────┐
    │ API Gateway   │   │ API Service   │  │ iOS App       │
    │ Dashboard     │   │ Dashboard     │  │ Analytics     │
    │ (network,etc) │   │ (code metrics)│  │ Dashboard     │
    └───────────────┘   └───────────────┘  └───────────────┘
```

**Пример.**

```python
# Dashboard Definition (as code using Grafana API)
from grafana_client.client import GrafanaClient
from grafana_client.model import Dashboard, Row, Panel

class DashboardBuilder:
    def __init__(self, grafana_url: str, api_key: str):
        self.client = GrafanaClient(
            (grafana_url, api_key),
            host=grafana_url,
            version=9
        )

    def build_service_dashboard(self, service_name: str) -> Dashboard:
        """Build dashboard for a microservice"""

        dashboard = Dashboard(
            title=f"{service_name} Monitoring",
            tags=[service_name, "microservice"],
            timezone="browser",
            panels=[
                # Row 1: Health Status
                self._build_status_row(service_name),

                # Row 2: RED Metrics
                self._build_red_metrics_row(service_name),

                # Row 3: Resource Usage
                self._build_resource_row(service_name),

                # Row 4: Dependencies
                self._build_dependency_row(service_name),

                # Row 5: Recent Errors
                self._build_errors_row(service_name),
            ],
            refresh="30s",
            time={'from': 'now-6h', 'to': 'now'},
        )

        return dashboard

    def _build_status_row(self, service: str) -> Row:
        """Status: Healthy/Warning/Critical"""
        return Row(
            panels=[
                Panel(
                    title="Service Status",
                    targets=[
                        {
                            'expr': f'up{{service="{service}"}}',
                            'format': 'table',
                        }
                    ],
                    graphPanel=False,  # Stat panel
                    thresholds='0,1',
                    colorBackground=True,
                ),
                Panel(
                    title="Uptime (30 days)",
                    targets=[
                        {
                            'expr': f'(1 - increase(service_downtime_seconds{{service="{service}"}}[30d]) / 2592000) * 100'
                        }
                    ],
                ),
            ]
        )

    def _build_red_metrics_row(self, service: str) -> Row:
        """Rate, Errors, Duration metrics"""
        return Row(
            panels=[
                Panel(
                    title="Request Rate",
                    targets=[
                        {
                            'expr': f'rate(http_requests_total{{service="{service}"}}[5m])',
                            'legendFormat': '{{method}} {{path}}',
                        }
                    ],
                    yaxes=[
                        {'format': 'reqps', 'label': 'Requests/sec'},
                    ],
                ),
                Panel(
                    title="Error Rate",
                    targets=[
                        {
                            'expr': f'rate(http_requests_total{{service="{service}",status=~"5.."}}[5m]) / rate(http_requests_total{{service="{service}"}}[5m])',
                            'legendFormat': 'Error Rate',
                        }
                    ],
                    yaxes=[
                        {'format': 'percent', 'min': 0, 'max': 1},
                    ],
                    alert={
                        'name': f'{service} High Error Rate',
                        'conditions': [
                            {
                                'evaluator': {'type': 'gt', 'params': [0.05]},
                            }
                        ],
                    }
                ),
                Panel(
                    title="Latency (p99)",
                    targets=[
                        {
                            'expr': f'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{{service="{service}"}}[5m]))',
                            'legendFormat': 'p99',
                        }
                    ],
                    yaxes=[
                        {'format': 's', 'label': 'Seconds'},
                    ],
                ),
            ]
        )

    def _build_resource_row(self, service: str) -> Row:
        """CPU, Memory, Disk"""
        return Row(
            panels=[
                Panel(
                    title="CPU Usage",
                    targets=[
                        {
                            'expr': f'rate(process_cpu_seconds_total{{service="{service}"}}[5m]) * 100',
                            'legendFormat': '{{instance}}',
                        }
                    ],
                    yaxes=[
                        {'format': 'percent', 'min': 0, 'max': 100},
                    ],
                ),
                Panel(
                    title="Memory Usage",
                    targets=[
                        {
                            'expr': f'process_resident_memory_bytes{{service="{service}"}} / (1024^3)',
                            'legendFormat': '{{instance}}',
                        }
                    ],
                    yaxes=[
                        {'format': 'bytes', 'label': 'GB'},
                    ],
                ),
                Panel(
                    title="Goroutines",
                    targets=[
                        {
                            'expr': f'go_goroutines{{service="{service}"}}',
                            'legendFormat': '{{instance}}',
                        }
                    ],
                ),
            ]
        )

    def _build_dependency_row(self, service: str) -> Row:
        """Database, Cache, Queue latencies"""
        return Row(
            panels=[
                Panel(
                    title="Database Latency",
                    targets=[
                        {
                            'expr': f'histogram_quantile(0.99, rate(db_query_duration_seconds_bucket{{service="{service}"}}[5m]))',
                            'legendFormat': '{{query_type}}',
                        }
                    ],
                ),
                Panel(
                    title="Cache Hit Rate",
                    targets=[
                        {
                            'expr': f'rate(cache_hits_total{{service="{service}"}}[5m]) / (rate(cache_hits_total{{service="{service}"}}[5m]) + rate(cache_misses_total{{service="{service}"}}[5m]))',
                        }
                    ],
                    yaxes=[
                        {'format': 'percent'},
                    ],
                ),
                Panel(
                    title="Queue Depth",
                    targets=[
                        {
                            'expr': f'queue_length{{service="{service}"}}',
                            'legendFormat': '{{queue_name}}',
                        }
                    ],
                ),
            ]
        )

    def _build_errors_row(self, service: str) -> Row:
        """Log errors, exceptions"""
        return Row(
            panels=[
                Panel(
                    title="Recent Errors (logs)",
                    targets=[
                        {
                            'expr': f'service="{service}" AND level=ERROR',
                            'datasource': 'Elasticsearch',
                            'format': 'table',
                        }
                    ],
                    tableOptions={
                        'showHeader': True,
                        'sortBy': {'displayName': '@timestamp', 'desc': True},
                    }
                ),
            ]
        )

    def publish_dashboard(self, dashboard: Dashboard):
        """Publish to Grafana"""
        self.client.dashboards.db.create(dashboard)
```

**Dashboard Query Optimization:**

```python
class DashboardQueryCache:
    """Cache dashboard queries for performance"""

    def __init__(self, tsdb, cache_ttl_seconds=60):
        self.tsdb = tsdb
        self.cache_ttl = cache_ttl_seconds
        self.cache = {}
        self.cache_timestamps = {}

    async def query_cached(self, query_hash: str, query: str, start: int, end: int):
        """Cache dashboard queries"""
        cache_key = query_hash

        # Check cache validity
        if cache_key in self.cache_timestamps:
            age = time.time() - self.cache_timestamps[cache_key]
            if age < self.cache_ttl:
                return self.cache[cache_key]

        # Cache miss - query TSDB
        result = await self.tsdb.query_range(query, start, end)

        # Store in cache
        self.cache[cache_key] = result
        self.cache_timestamps[cache_key] = time.time()

        return result

    async def query_downsampled(self, query: str, start: int, end: int):
        """Use downsampled data for long time ranges"""
        duration_hours = (end - start) / 3600

        if duration_hours > 24:
            # Use 5m resolution instead of 10s
            query = query.replace('[5m]', '[5m]').replace('[10s]', '[5m]')

        if duration_hours > 168:  # 1 week
            # Use 1h resolution
            query = query.replace('[5m]', '[1h]')

        if duration_hours > 2592000:  # 30 days
            # Use 1d resolution
            query = query.replace('[1h]', '[1d]')

        return await self.query_cached(hash(query), query, start, end)
```

**Типичные ошибки.**
- Слишком много информации на одном дашборде → информационная перегрузка
- Неправильный выбор агрегации (mean вместо p99) → скрытие реальных проблем
- Отсутствие alerts на панелях → alerts отделены от metrics
- Нет drill-down mechanism → нельзя понять root cause
- Querying raw 10s data для 30-дневного view → slow queries

**На интервью.**
- Объясни RED vs USE methods и когда каждый применяется
- Упомяни importance of alerting thresholds на дашборде
- Уточняющий вопрос: «Как оптимизировать queries для больших time ranges?» → downsampling, caching, pre-aggregation
- Уточняющий вопрос: «Как организовать dashboards для 100 микросервисов?» → hierarchy, templating, auto-generation

---

### 7. Как реализовать распределённую трассировку (Distributed Tracing)?

**Зачем спрашивают.** Трассировка критична для понимания flow в микросервисной архитектуре. Нужно понимать span collection, trace propagation, sampling, и storage.

**Короткий ответ.** Distributed trace отслеживает request через несколько сервисов. Каждый сервис создаёт spans (atomic unit of work) и отправляет их в collector (Jaeger, Zipkin). Trace ID распространяется через HTTP headers (Trace-Context), и каждый сервис создаёт child spans. Sampling необходима для масштабируемости (sample 1 из 100 traces). Данные хранятся в специализированных TSDB для трассировки.

**Детальный разбор.**

**Anatomy of Trace:**

```
Request: GET /api/users/123

Trace ID: abc123def456

┌─────────────────────────────────────────────────────┐
│ api-gateway (root span)                             │
│ Time: 0ms - 250ms (total 250ms)                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Span: receive_request (0-10ms)                  │ │
│ │ Span: validate_token (10-30ms)                  │ │
│ │ ┌────────────────────────────────────────────┐ │ │
│ │ │ Span: call_auth_service (30-60ms)         │ │ │
│ │ │                                            │ │ │
│ │ │ ┌──────────────────────────────────────┐  │ │ │
│ │ │ │ auth-service (child of call_auth)   │  │ │ │
│ │ │ │ Time: 35ms - 55ms                   │  │ │ │
│ │ │ │ Span: validate_token (35-40ms)      │  │ │ │
│ │ │ │ Span: query_db (40-55ms)            │  │ │ │
│ │ │ │                                     │  │ │ │
│ │ │ │ ┌──────────────────────────────┐   │  │ │ │
│ │ │ │ │ postgres (child)              │   │  │ │ │
│ │ │ │ │ Time: 42ms - 54ms            │   │  │ │ │
│ │ │ │ │ Span: execute_query (42-54)  │   │  │ │ │
│ │ │ │ └──────────────────────────────┘   │  │ │ │
│ │ │ └──────────────────────────────────┘  │ │ │
│ │ └────────────────────────────────────────┘ │ │
│ │ Span: call_user_service (60-150ms)       │ │
│ │ ┌────────────────────────────────────────┐ │ │
│ │ │ user-service                           │ │ │
│ │ │ Span: get_user (65-145ms)              │ │ │
│ │ │   - query_db (70-85ms)                 │ │ │
│ │ │   - format_response (85-145ms)         │ │ │
│ │ └────────────────────────────────────────┘ │ │
│ │ Span: send_response (150-250ms)          │ │
│ └─────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────┘

Результат анализа:
- Total latency: 250ms
- Slowest service: user-service (80ms)
- Database queries: 38ms total
- Hotspot: format_response in user-service
```

**Trace Propagation (W3C Trace Context):**

```
┌──────────────────────────────────┐
│ Client                           │
│ Creates Trace ID                 │
└──────────────┬───────────────────┘
               │ GET /api/users/123
               │ Traceparent: 00-abc123def456-1234567890ab-01
               │
         ┌─────▼──────────────────────────────┐
         │ api-gateway                        │
         │ Receives trace context             │
         │ Creates span: api-gateway          │
         │ Parent-ID: root                    │
         │ Span-ID: aaaa                      │
         │                                    │
         │ Forwards to auth-service           │
         │ Traceparent: 00-abc123...-aaaa-01 │
         └─────┬──────────────────────────────┘
               │
         ┌─────▼──────────────────────────────┐
         │ auth-service                       │
         │ Receives trace context             │
         │ Parent-ID: aaaa (from gateway)     │
         │ Creates span: auth-service         │
         │ Span-ID: bbbb                      │
         │                                    │
         │ Calls postgres                     │
         │ Traceparent: 00-abc123...-bbbb-01 │
         └─────┬──────────────────────────────┘
               │
         ┌─────▼──────────────────────────────┐
         │ PostgreSQL Driver                  │
         │ Parent-ID: bbbb                    │
         │ Creates span: execute_query        │
         │ Span-ID: cccc                      │
         └──────────────────────────────────────┘
```

**Пример.**

```python
# Span Collection & Propagation
import time
from opentelemetry import trace, baggage
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Setup Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name='localhost',
    agent_port=6831,
)

trace_provider = TracerProvider()
trace_provider.add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)
trace.set_tracer_provider(trace_provider)

tracer = trace.get_tracer(__name__)

# Instrument Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route('/api/users/<int:user_id>')
def get_user(user_id):
    """Endpoint с automatic tracing"""
    # Flask auto-creates root span

    # Create manual span for business logic
    with tracer.start_as_current_span("get_user_logic") as span:
        span.set_attribute("user_id", user_id)

        # Validate token
        with tracer.start_as_current_span("validate_token"):
            token = request.headers.get('Authorization')
            if not token:
                return {'error': 'No token'}, 401

        # Call auth service
        with tracer.start_as_current_span("call_auth_service") as auth_span:
            auth_span.set_attribute("service", "auth")

            # requests.get auto-creates child span for HTTP
            auth_resp = requests.get(
                'http://auth-service/validate',
                headers={'Authorization': token}
            )

        # Call user service
        with tracer.start_as_current_span("call_user_service") as user_span:
            user_span.set_attribute("service", "user")

            # Add baggage (context data passed to all downstream services)
            baggage.set_baggage("user_id", str(user_id))

            user_resp = requests.get(
                f'http://user-service/users/{user_id}'
            )

        return user_resp.json()

# Custom instrumentation for database
class DatabaseSpan:
    def __init__(self, tracer, query: str):
        self.tracer = tracer
        self.query = query

    def __enter__(self):
        self.span = self.tracer.start_span("db_query")
        self.span.set_attribute("db.statement", self.query[:100])  # Truncate
        self.span.set_attribute("db.system", "postgresql")
        self.start_time = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        duration = (time.time() - self.start_time) * 1000
        self.span.set_attribute("db.duration_ms", duration)

        if exc_type:
            self.span.record_exception(exc_val)

        self.span.end()

# Usage in database layer
def query_user(user_id: int):
    with DatabaseSpan(tracer, f"SELECT * FROM users WHERE id = {user_id}") as db:
        # Actual query
        user = db_session.query(User).filter(User.id == user_id).first()
        return user

# Span Sampling
class ProbabilisticSampler:
    """Sample 1 out of N traces"""

    def __init__(self, sample_rate=0.1):
        self.sample_rate = sample_rate

    def should_sample(self, trace_id: str) -> bool:
        # Deterministic sampling based on trace ID
        trace_hash = int(trace_id[:8], 16)
        max_value = 0xFFFFFFFF

        sample_threshold = int(self.sample_rate * max_value)
        return trace_hash < sample_threshold

# Async Span Processor (non-blocking export)
class AsyncSpanProcessor:
    def __init__(self, exporter, queue_size=2048):
        self.exporter = exporter
        self.queue = asyncio.Queue(maxsize=queue_size)
        self.worker_task = None

    def on_span_end(self, span):
        """Called when span ends"""
        try:
            self.queue.put_nowait(span)
        except asyncio.QueueFull:
            # Drop span if queue is full
            print(f"Dropped span {span.name} due to full queue")

    async def export_worker(self):
        """Background worker to export spans"""
        batch = []

        while True:
            try:
                span = await asyncio.wait_for(
                    self.queue.get(),
                    timeout=5  # Flush every 5 seconds
                )
                batch.append(span)

                if len(batch) >= 100:
                    # Export batch
                    self.exporter.export(batch)
                    batch.clear()

            except asyncio.TimeoutError:
                # Timeout - flush whatever we have
                if batch:
                    self.exporter.export(batch)
                    batch.clear()
```

**Trace Storage & Query (Elasticsearch-based Jaeger):**

```python
class TraceStorage:
    """Query traces from Jaeger/Elasticsearch"""

    def __init__(self, es_client):
        self.es = es_client

    async def find_traces_by_service(self, service: str, min_duration_ms: int = None):
        """Find traces that touched a service"""
        query = {
            "bool": {
                "must": [
                    {
                        "nested": {
                            "path": "spans",
                            "query": {
                                "match": {"spans.process.serviceName": service}
                            }
                        }
                    }
                ]
            }
        }

        if min_duration_ms:
            query["bool"]["must"].append({
                "range": {
                    "duration": {"gte": min_duration_ms}
                }
            })

        result = await self.es.search(
            index="jaeger-*",
            body={"query": query},
            sort=[{"startTime": {"order": "desc"}}],
            size=100
        )

        return result['hits']['hits']

    async def find_error_traces(self, service: str, lookback_minutes: int = 60):
        """Find traces with errors"""
        now = int(time.time() * 1_000_000)
        lookback_us = lookback_minutes * 60 * 1_000_000

        query = {
            "bool": {
                "must": [
                    {"range": {"startTime": {"gte": now - lookback_us}}},
                    {
                        "nested": {
                            "path": "spans",
                            "query": {
                                "bool": {
                                    "must": [
                                        {"match": {"spans.process.serviceName": service}},
                                        {
                                            "exists": {
                                                "field": "spans.tags.error"
                                            }
                                        }
                                    ]
                                }
                            }
                        }
                    }
                ]
            }
        }

        result = await self.es.search(
            index="jaeger-*",
            body={"query": query},
            size=50
        )

        return result['hits']['hits']

    async def get_latency_percentiles(self, service: str, lookback_minutes: int = 60):
        """Calculate latency distribution"""
        now = int(time.time() * 1_000_000)
        lookback_us = lookback_minutes * 60 * 1_000_000

        query = {
            "aggs": {
                "latency_percentiles": {
                    "percentiles": {
                        "field": "duration",
                        "percents": [50, 90, 95, 99],
                        "keyed": True
                    }
                }
            },
            "query": {
                "bool": {
                    "must": [
                        {"range": {"startTime": {"gte": now - lookback_us}}},
                        {
                            "nested": {
                                "path": "spans",
                                "query": {
                                    "match": {"spans.process.serviceName": service}
                                }
                            }
                        }
                    ]
                }
            }
        }

        result = await self.es.search(
            index="jaeger-*",
            body=query
        )

        return result['aggregations']['latency_percentiles']['values']
```

**Типичные ошибки.**
- Забыть про sampling → explosion в объёме данных
- Не распространять trace context → потеря трассировки между сервисами
- Слишком детальные spans → performance overhead
- Не инструментировать external calls → gap в understanding latency
- Хранить PII в span tags → privacy violation

**На интервью.**
- Объясни difference между trace, span, и parent-child relationships
- Упомяни trace context propagation через headers
- Уточняющий вопрос: «Как выбирать sampling rate?» → risk of missing errors vs cost
- Уточняющий вопрос: «Как найти problematic service в сложной trace?» → latency critical path analysis

---

### 8. Как определить и отслеживать SLI/SLO/SLA?

**Зачем спрашивают.** SLI/SLO/SLA — foundation of reliability engineering. Нужно понимать что измерять, как устанавливать targets, и как track compliance.

**Короткий ответ.** SLI (Service Level Indicator) — метрика, которую ты измеряешь (error rate, latency). SLO (Service Level Objective) — target для SLI (error rate < 0.1%). SLA (Service Level Agreement) — contractual promise с penalties. Определи SLI через RED/USE methods, установи SLO основываясь на user expectations и business requirements. Вводи Error Budget для управления изменениями vs reliability.

**Детальный разбор.**

**SLI Selection (Indicator Selection):**

```
Для User-Facing Service:
┌─────────────────────────────────────┐
│ 1. Availability / Error Rate         │
│    └─ HTTP 5xx rate                  │
│    └─ Service up/down                │
├─────────────────────────────────────┤
│ 2. Latency                           │
│    └─ p99 response time < 500ms      │
│    └─ p95 < 300ms                    │
├─────────────────────────────────────┤
│ 3. Throughput                        │
│    └─ QPS sustained                  │
├─────────────────────────────────────┤
│ 4. Data Freshness (if applicable)    │
│    └─ Max age of data in cache       │
├─────────────────────────────────────┤
│ 5. Correctness                       │
│    └─ Rate of data inconsistencies   │
└─────────────────────────────────────┘

Для Infrastructure/Resource Service:
┌─────────────────────────────────────┐
│ 1. Request Success Rate              │
│    └─ (Total - Errors) / Total       │
├─────────────────────────────────────┤
│ 2. Latency                           │
│    └─ p99, p95, mean                 │
├─────────────────────────────────────┤
│ 3. System Availability               │
│    └─ Time service is reachable      │
└─────────────────────────────────────┘
```

**Установка SLO (определение целей):**

```
Примеры SLO доступности:
─────────────────────────
99.0% (two nines)
 └─ 43.2 minutes downtime/month
 └─ Good for: non-critical services, dev environments
 └─ Error budget: 21.6 minutes/month

99.9% (three nines)
 └─ 4.32 minutes downtime/month
 └─ Good for: typical production services
 └─ Error budget: 2.16 minutes/month

99.95%
 └─ 2.16 minutes downtime/month
 └─ good for: high-value services
 └─ Error budget: 1.08 minutes/month

99.99% (four nines)
 └─ 26 seconds downtime/month
 └─ good for: critical financial systems
 └─ Error budget: 13 seconds/month

99.999% (five nines)
 └─ 2.6 seconds downtime/month
 └─ almost impossible in practice

Примеры SLO latency:
────────────────────
p99 < 100ms   (fast API)
p99 < 500ms   (web API)
p99 < 2s      (batch operations)
```

**Концепция Error Budget:**

```
SLO: 99.9% availability (0.1% error rate)

Расчёт ежемесячного бюджета:
─────────────────────────
Total requests/month: 100M
Allowed errors: 100K (0.1%)

Отслеживание выгорания бюджета:
┌──────────────────────────────────┐
│ Errors Used vs Budget Over Time  │
├──────────────────────────────────┤
│ │                                │
│ │                         ╱       │
│ │                    ╱╱╱╱╱        │
│ │           ╱╱╱╱╱╱╱╱            │
│ │      ╱╱╱╱╱╱                    │
│ │ ───────────────────────────── │
│ │ 100K (budget) ─────────────── │
│ │                                │
│ │ Actual      Budget OK         │
│ └──────────────────────────────┘
   Day 5  Day 10  Day 15  Day 30

Интерпретация:
- Если потребляешь бюджет быстрее → нужно заморозить релизы
- Если медленнее → можно быть более агрессивным в деплоях
- Конец месяца: если осталось много → "use it or lose it"
```

**Пример.**

```python
# SLI/SLO Calculation & Monitoring
from datetime import datetime, timedelta
import asyncio

class SLICalculator:
    """Calculate Service Level Indicators"""

    def __init__(self, tsdb):
        self.tsdb = tsdb

    async def calculate_availability(self, service: str, time_window: str = '30d') -> float:
        """Calculate availability SLI"""
        # Query: successful requests / total requests
        query = f'''
        sum(rate(http_requests_total{{service="{service}",status=~"[2345].."}}[5m]))
        /
        sum(rate(http_requests_total{{service="{service}"}}[5m]))
        '''

        result = await self.tsdb.query_range(query, time_window)

        # Average availability over the period
        values = [v['value'] for v in result['values']]
        avg_availability = sum(values) / len(values) if values else 0

        return avg_availability * 100  # Return as percentage

    async def calculate_latency_sli(self, service: str, percentile: int = 99, threshold_ms: int = 500):
        """Calculate latency SLI (% of requests under threshold)"""
        # Query: histogram_quantile gives us the latency
        query = f'''
        histogram_quantile({percentile/100}, rate(http_request_duration_seconds_bucket{{service="{service}"}}[5m]))
        '''

        result = await self.tsdb.query_range(query, '30d')

        # Calculate % of time p99 was under threshold
        threshold_seconds = threshold_ms / 1000
        values_under_threshold = sum(1 for v in result['values'] if float(v['value']) < threshold_seconds)

        sli = (values_under_threshold / len(result['values'])) * 100 if result['values'] else 0

        return sli

    async def calculate_error_sli(self, service: str, threshold_rate: float = 0.001):
        """Calculate error rate SLI (% of time under threshold)"""
        query = f'''
        sum(rate(http_requests_total{{service="{service}",status=~"5.."}}[5m]))
        /
        sum(rate(http_requests_total{{service="{service}"}}[5m]))
        '''

        result = await self.tsdb.query_range(query, '30d')

        values_under_threshold = sum(
            1 for v in result['values']
            if float(v['value']) < threshold_rate
        )

        sli = (values_under_threshold / len(result['values'])) * 100 if result['values'] else 0

        return sli

class ErrorBudgetTracker:
    """Track error budget consumption"""

    def __init__(self, slo_target: float, lookback_days: int = 30):
        self.slo_target = slo_target  # e.g., 0.999 for 99.9%
        self.lookback_days = lookback_days
        self.allowed_error_rate = 1 - slo_target

    def calculate_budget_status(self, actual_error_rate: float) -> dict:
        """How much error budget is left?"""
        error_budget_consumed = actual_error_rate / self.allowed_error_rate

        if error_budget_consumed > 1.0:
            # Exceeded budget
            status = "red"
            budget_remaining = 0
            days_until_depletion = 0
        else:
            budget_remaining = 1.0 - error_budget_consumed
            status = "green" if budget_remaining > 0.5 else "yellow"

            if actual_error_rate > 0:
                # Project when budget depletes
                days_until_depletion = self.lookback_days * budget_remaining / error_budget_consumed
            else:
                days_until_depletion = self.lookback_days

        return {
            'slo_target': self.slo_target,
            'allowed_error_rate': self.allowed_error_rate,
            'actual_error_rate': actual_error_rate,
            'budget_consumed_percent': min(100, error_budget_consumed * 100),
            'budget_remaining_percent': max(0, budget_remaining * 100),
            'status': status,
            'days_until_depletion': max(0, days_until_depletion),
            'safe_to_deploy': status != "red"
        }

    def calculate_monthly_budget(self, total_requests: int) -> dict:
        """Calculate monthly error budget"""
        allowed_errors = total_requests * self.allowed_error_rate

        return {
            'total_requests': total_requests,
            'allowed_errors': int(allowed_errors),
            'minutes_downtime_allowed': (allowed_errors / total_requests) * 24 * 60,
        }

# SLO Dashboard Integration
class SLODashboard:
    def __init__(self, tsdb):
        self.tsdb = tsdb
        self.sli_calc = SLICalculator(tsdb)

    async def generate_slo_report(self, service: str):
        """Generate SLO compliance report"""

        # Calculate SLIs
        availability = await self.sli_calc.calculate_availability(service)
        latency_sli = await self.sli_calc.calculate_latency_sli(service, percentile=99, threshold_ms=500)
        error_sli = await self.sli_calc.calculate_error_sli(service, threshold_rate=0.001)

        # Define SLOs
        slo_targets = {
            'availability': 0.999,      # 99.9%
            'latency_p99': 0.95,        # p99 < 500ms 95% of the time
            'error_rate': 0.995,        # Error rate < 0.1% 99.5% of the time
        }

        # Check compliance
        report = {
            'service': service,
            'generated_at': datetime.utcnow().isoformat(),
            'slis': {
                'availability': {
                    'actual': availability,
                    'target': slo_targets['availability'] * 100,
                    'status': 'pass' if availability >= slo_targets['availability'] * 100 else 'fail'
                },
                'latency_p99': {
                    'actual': latency_sli,
                    'target': slo_targets['latency_p99'] * 100,
                    'status': 'pass' if latency_sli >= slo_targets['latency_p99'] * 100 else 'fail'
                },
                'error_rate': {
                    'actual': error_sli,
                    'target': slo_targets['error_rate'] * 100,
                    'status': 'pass' if error_sli >= slo_targets['error_rate'] * 100 else 'fail'
                }
            },
            'overall_status': 'pass' if all(
                v['status'] == 'pass' for v in report['slis'].values()
            ) else 'fail'
        }

        # Error budget tracking
        tracker = ErrorBudgetTracker(slo_targets['availability'])
        actual_error_rate = (100 - availability) / 100
        budget = tracker.calculate_budget_status(actual_error_rate)

        report['error_budget'] = budget

        return report
```

**SLA Example (Contract):**

```
Service Level Agreement: API Service
───────────────────────────────────

1. Availability:
   - Target: 99.9% monthly availability
   - Measured: Successful HTTP responses / Total HTTP requests
   - Exclusions: Planned maintenance (4 hours/month)

2. Latency:
   - Target: p99 latency < 500ms for 95% of requests
   - Measured: HTTP request duration percentiles
   - Exclusions: Requests with invalid syntax

3. Support Response Time:
   - Critical issues: 1 hour
   - Major issues: 4 hours
   - Minor issues: 24 hours

4. Credits for SLA Violations:
   - 99.0% - 99.9%  → 10% monthly fee credit
   - 95.0% - 99.0%  → 25% monthly fee credit
   - < 95.0%        → 100% monthly fee credit + investigation

5. Reporting:
   - Monthly SLO report by 5th of month
   - Detailed metrics available in customer dashboard
   - Incident post-mortems for major outages
```

**Типичные ошибки.**
- Выбрать SLO слишком высокий → невозможно достичь, demoralizes team
- Слишком низкий → не защищает пользователей
- Не учитывать user pain points → SLI не соответствуют реальным проблемам
- Забыть про exclusions → SLO calculation становится спорным
- Не использовать error budget → continuous deployment без управления reliability

**На интервью.**
- Объясни разницу между SLI, SLO, и SLA
- Упомяни error budget и как он управляет deployment velocity
- Уточняющий вопрос: «Как выбирать SLI для нового сервиса?» → начни с RED metrics, iteratively refine
- Уточняющий вопрос: «Как обрабатывать SLO miss?» → incident review, error budget planning

---

### 9. Как масштабировать систему мониторинга на миллионы метрик?

**Зачем спрашивают.** Масштабирование мониторинга — not linear problem. Нужно понимать bottlenecks, sharding strategies, и distributed architecture.

**Короткий ответ.** На миллионы метрик требуется horizontal scaling на уровне ingestion, storage, и query. Используй sharding по series ID (Prometheus remote-write, VictoriaMetrics clustering). Вводи downsampling/aggregation для дешёвого хранения. Pre-aggregate популярные queries. Используй query-layer кэш. Разделяй metrics по service/team для isolation.

**Детальный разбор.**

**Scalability Bottlenecks:**

```
┌────────────────────────────────────────────────────┐
│ Problem                  │ Solution              │
├────────────────────────────────────────────────────┤
│ 1. Write throughput      │ Sharding, batching    │
│ High cardinality:        │ Cardinality limits    │
│   - 1M series × 100 pts  │ Label value limits    │
│   - per second           │                       │
├────────────────────────────────────────────────────┤
│ 2. Storage growth        │ Compression           │
│ 100GB/day raw            │ Downsampling          │
│ → 20GB/day compressed    │ Retention policies    │
│ → 600GB/month            │                       │
├────────────────────────────────────────────────────┤
│ 3. Query latency         │ Pre-aggregation       │
│ Scan 1M series in 1s     │ Caching               │
│ → not all queries fast   │ Sampling              │
├────────────────────────────────────────────────────┤
│ 4. Operational burden    │ Automation            │
│ Manage 10+ TSDB nodes    │ Self-healing          │
│                          │ Replication           │
└────────────────────────────────────────────────────┘
```

**Sharding Architecture:**

```
                    ┌─────────────────────┐
                    │  Metrics Collector  │
                    │  (Load Balancer)    │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
         ┌────▼────┐      ┌────▼────┐      ┌──▼────┐
         │ Shard 1 │      │ Shard 2 │      │Shard N│
         │ (hash   │      │ (hash   │      │ (hash │
         │ 0-33%)  │      │ 33-66%) │      │66-100)│
         │         │      │         │      │       │
         │ ┌─────┐ │      │ ┌─────┐ │      │┌─────┐│
         │ │TSDB │ │      │ │TSDB │ │      ││TSDB││
         │ │node1│ │      │ │node3│ │      ││node5││
         │ └─────┘ │      │ └─────┘ │      │└─────┘│
         │ ┌─────┐ │      │ ┌─────┐ │      │┌─────┐│
         │ │TSDB │ │      │ │TSDB │ │      ││TSDB││
         │ │node2│ │      │ │node4│ │      ││node6││
         │ └─────┘ │      │ └─────┘ │      │└─────┘│
         └────┬────┘      └────┬────┘      └──┬────┘
              │                │               │
              └────────────────┼───────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Query Aggregator   │
                    │  (Merge results)    │
                    └─────────────────────┘
```

**Cardinality Management:**

```
Problem: 1M unique series

┌──────────────────────────────────────┐
│ Example: request_duration_seconds    │
│ {method, path, status, user_id}      │
├──────────────────────────────────────┤
│                                      │
│ method: [GET, POST, PUT, DELETE]     │ 4
│    × path: [/api/users, /api/posts]  │ × 2
│    × status: [200, 400, 500]         │ × 3
│    × user_id: [1, 2, ..., 10000]     │ × 10000
│                                      │
│ = 240,000 combinations               │
│ (with just 10K users!)               │
│                                      │
│ If N users and M paths:              │
│ = methods × paths × statuses × users │
│ = 4 × 2 × 3 × M                      │
│ = 24M series (explosive!)            │
└──────────────────────────────────────┘

Solutions:
─────────
1. Remove high-cardinality labels:
   - Don't include user_id, session_id, request_id
   - Instead: use aggregate metrics or logging

2. Set cardinality limits:
   - Max 100 unique values per label
   - Drop series if exceeded

3. Use label enforcement:
   - Pre-define allowed label combinations
   - Reject invalid combinations

4. Sample high-cardinality:
   - Only sample 1 in 100 users
   - Accept data loss for granularity tradeoff
```

**Пример.**

```python
# Large-scale TSDB Architecture
from abc import ABC, abstractmethod
import hashlib

class ShardRouter:
    """Route metrics to correct shard"""

    def __init__(self, num_shards: int):
        self.num_shards = num_shards

    def get_shard(self, metric_name: str, labels: dict) -> int:
        """Consistent hashing for sharding"""
        # Create series ID
        sorted_labels = ''.join(
            f'{k}={v}' for k, v in sorted(labels.items())
        )
        series_id = f'{metric_name}|{sorted_labels}'

        # Hash to shard
        hash_value = int(hashlib.md5(series_id.encode()).hexdigest(), 16)
        return hash_value % self.num_shards

class ShardedTSDB:
    """Distributed TSDB with sharding"""

    def __init__(self, num_shards: int, shard_backends: list):
        self.router = ShardRouter(num_shards)
        self.shards = {i: backend for i, backend in enumerate(shard_backends)}

    async def write(self, metric_name: str, labels: dict, value: float, timestamp: int):
        """Write to appropriate shard"""
        shard_id = self.router.get_shard(metric_name, labels)
        shard = self.shards[shard_id]

        await shard.write(metric_name, labels, value, timestamp)

    async def query_range(self, query: str, start: int, end: int, step: int):
        """Query across all shards and merge"""
        import asyncio

        # Send query to all shards in parallel
        tasks = [
            shard.query_range(query, start, end, step)
            for shard in self.shards.values()
        ]

        shard_results = await asyncio.gather(*tasks, return_exceptions=True)

        # Merge results from all shards
        merged = {}
        for result in shard_results:
            if isinstance(result, Exception):
                continue  # Handle errors

            for series_key, values in result.items():
                if series_key not in merged:
                    merged[series_key] = []
                merged[series_key].extend(values)

        return merged

class CardinalityLimiter:
    """Enforce per-label cardinality limits"""

    def __init__(self, max_cardinality: int = 100):
        self.max_cardinality = max_cardinality
        self.label_values = {}  # label_name → set(values)
        self.lock = asyncio.Lock()

    async def check_cardinality(self, label_name: str, value: str) -> bool:
        """Check if adding this label value would exceed limit"""
        async with self.lock:
            if label_name not in self.label_values:
                self.label_values[label_name] = set()

            values = self.label_values[label_name]

            if value in values:
                return True  # Existing value, OK

            if len(values) < self.max_cardinality:
                values.add(value)
                return True
            else:
                # Would exceed limit
                return False

    async def drop_high_cardinality_metric(self, metric_name: str, labels: dict):
        """Drop metric if it would create too many series"""
        for label_name, value in labels.items():
            allowed = await self.check_cardinality(label_name, value)
            if not allowed:
                return True  # Drop this metric

        return False

class QueryCache:
    """Cache query results for repeated queries"""

    def __init__(self, ttl_seconds: int = 60):
        self.cache = {}
        self.ttl = ttl_seconds

    def get_cache_key(self, query: str, start: int, end: int, step: int) -> str:
        """Generate cache key"""
        import hashlib
        key = f"{query}|{start}|{end}|{step}"
        return hashlib.md5(key.encode()).hexdigest()

    async def query_with_cache(self, query_func, query: str, start: int, end: int, step: int):
        """Query with caching"""
        cache_key = self.get_cache_key(query, start, end, step)

        # Check cache
        if cache_key in self.cache:
            cached_result, cached_time = self.cache[cache_key]
            if time.time() - cached_time < self.ttl:
                return cached_result

        # Cache miss - execute query
        result = await query_func(query, start, end, step)

        # Store in cache
        self.cache[cache_key] = (result, time.time())

        return result

class PreAggregationEngine:
    """Pre-aggregate popular metrics"""

    def __init__(self, tsdb):
        self.tsdb = tsdb
        self.pre_aggregations = {
            # Define popular aggregations
            'http_requests_rate_by_service': {
                'base_query': 'rate(http_requests_total[5m])',
                'group_by': ['service'],
                'interval': '5m',
            },
            'error_rate_by_service': {
                'base_query': 'rate(http_requests_total{status=~"5.."}[5m])',
                'group_by': ['service'],
                'interval': '5m',
            },
        }

    async def update_aggregations(self):
        """Periodically compute pre-aggregations"""
        for agg_name, config in self.pre_aggregations.items():
            # Query base metric
            result = await self.tsdb.query(config['base_query'])

            # Group and aggregate
            grouped = {}
            for series in result:
                key = tuple(series.labels[label] for label in config['group_by'])
                if key not in grouped:
                    grouped[key] = 0
                grouped[key] += series.value

            # Store aggregated metrics
            for group_key, value in grouped.items():
                labels = dict(zip(config['group_by'], group_key))
                await self.tsdb.write(
                    f"pre_aggregated_{agg_name}",
                    labels,
                    value,
                    int(time.time() * 1000)
                )

class MetricsDownsampler:
    """Downsample old data for cost savings"""

    RETENTION_POLICIES = [
        {'resolution': '10s', 'retention_days': 7},    # High-res, short
        {'resolution': '1m', 'retention_days': 30},    # Medium-res
        {'resolution': '5m', 'retention_days': 90},    # Low-res
        {'resolution': '1h', 'retention_days': 365},   # Very low-res
    ]

    async def downsample_metrics(self, tsdb):
        """Run downsampling job"""
        for i, current_policy in enumerate(self.RETENTION_POLICIES[:-1]):
            next_policy = self.RETENTION_POLICIES[i + 1]

            # Find data that's older than retention but not yet downsampled
            cutoff_time = time.time() - (current_policy['retention_days'] * 86400)

            # Downsample by aggregating points
            # e.g., 10s data → 1m data (6 points → 1)
            await self._downsample_range(
                tsdb,
                cutoff_time,
                current_policy['resolution'],
                next_policy['resolution']
            )

    async def _downsample_range(self, tsdb, cutoff_time, src_res: str, dst_res: str):
        """Actually downsample a range of data"""
        # Simplified: read raw data, compute aggregates, write downsampled
        pass
```

**Типичные ошибки.**
- Shard на random labels → poor load distribution
- Не учитывать growth → shards become unbalanced
- Quering cross-shard без limits → scan billions of points
- Cardinality explosion → run out of memory/storage
- Downsampling агрессивно → loose visibility into problems

**На интервью.**
- Объясни consistent hashing для sharding
- Упомяни cardinality problem и solutions
- Уточняющий вопрос: «Как переши́фтировать данные при добавлении нового шарда?» → resharding, online migration
- Уточняющий вопрос: «Как оптимизировать для time-range queries?» → time-based partitioning

---

### 10. Как мониторить саму систему мониторинга?

**Зачем спрашивают.** Meta-monitoring — часто забывается, но critical для reliability. Нужно понимать что monitore, как обнаружить сбои в самом monitoring system.

**Короткий ответ.** Мониторь: write latency (lag), cardinality, storage usage, query latency, scrape failures, alert delivery. Используй synthetic monitoring (send fake metrics, check they appear). Настрой alerting на alert delivery failures (если alerts не приходят по часу). Versioning метрик кликает consistency check. Дублируй критические данные в другую TSDB.

**Детальный разбор.**

**What to Monitor in Monitoring Stack:**

```
┌──────────────────────────────────────────────────┐
│ 1. Ingestion / Write Path                        │
├──────────────────────────────────────────────────┤
│ Metric                 │ Alert Threshold        │
│────────────────────────┼────────────────────────│
│ Write latency (p99)    │ > 1s (vs 100ms normal) │
│ Scrape failures        │ > 1% of targets        │
│ Cardinality growth     │ > 10% daily            │
│ Buffer queue depth     │ > 80% full             │
│ CPU usage (collector)  │ > 80%                  │
│ Memory (collector)     │ > 85%                  │
│ Storage growth rate    │ > 150GB/day (vs 100)   │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 2. Storage / Query Path                          │
├──────────────────────────────────────────────────┤
│ Query latency (p99)    │ > 5s (vs 500ms normal) │
│ Query errors           │ > 0.1%                 │
│ Disk space usage       │ > 85% full             │
│ Compaction lag         │ > 1h behind            │
│ Index corruption       │ Any failures           │
│ Replication lag        │ > 5min (for HA)        │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 3. Alerting System                               │
├──────────────────────────────────────────────────┤
│ Alert evaluation time  │ > 10s                  │
│ Alert delivery time    │ > 1min                 │
│ Failed notifications   │ > 1%                   │
│ Firing alerts w/o data │ > 0 (data stale)       │
│ AlertManager uptime    │ < 99.9%                │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 4. Data Quality                                  │
├──────────────────────────────────────────────────┤
│ Missing data points    │ > 1% (gaps)            │
│ Duplicate data         │ > 0.1%                 │
│ Out-of-order points    │ > 0.01%                │
│ Stale metrics (no      │ > 5% of series         │
│   updates for 1h)      │                        │
└──────────────────────────────────────────────────┘
```

**Synthetic Monitoring (Canaries):**

```
┌─────────────────────────────────────┐
│  Synthetic Metric Generator         │
│  (Generates known metrics)          │
├─────────────────────────────────────┤
│ Every 10 seconds:                   │
│  - Generate metric:                 │
│    synthetic_heartbeat_timestamp    │
│    = current_unix_timestamp         │
│                                     │
│  - Generate metric:                 │
│    synthetic_random_value           │
│    = random(0, 100)                 │
│                                     │
│  - Generate metric:                 │
│    synthetic_latency_ms             │
│    = 100 (fixed)                    │
└─────────────────────────────────────┘
             │
             ▼
     [Metrics Collector]
             │
             ▼
   [TSDB, Alerting, etc]
             │
             ▼
   ┌─────────────────────────────────┐
   │  Synthetic Metric Validator      │
   │  (Check they arrived correctly)  │
   ├─────────────────────────────────┤
   │ Every minute:                   │
   │  1. Query synthetic metrics      │
   │  2. Check values are correct     │
   │  3. Check freshness (< 2min old) │
   │  4. Alert if anything wrong      │
   └─────────────────────────────────┘
```

**Пример.**

```python
# Meta-Monitoring Implementation
import time
import random

class SyntheticMetricsGenerator:
    """Generate known metrics for canary testing"""

    def __init__(self, tsdb):
        self.tsdb = tsdb
        self.generation_counter = 0

    async def generate_heartbeat(self):
        """Generate synthetic heartbeat metric"""
        timestamp = int(time.time() * 1000)

        await self.tsdb.write(
            metric_name='synthetic_heartbeat_seconds',
            labels={'source': 'monitoring-canary'},
            value=float(timestamp // 1000),
            timestamp=timestamp
        )

    async def generate_random_metric(self):
        """Generate synthetic metric with random value"""
        timestamp = int(time.time() * 1000)

        await self.tsdb.write(
            metric_name='synthetic_random_value',
            labels={'source': 'monitoring-canary'},
            value=random.random() * 100,
            timestamp=timestamp
        )

    async def generate_latency_metric(self):
        """Generate synthetic metric simulating latency"""
        start = time.time()

        # Simulate some work
        await asyncio.sleep(0.01)

        latency_ms = (time.time() - start) * 1000
        timestamp = int(time.time() * 1000)

        await self.tsdb.write(
            metric_name='synthetic_latency_ms',
            labels={'source': 'monitoring-canary'},
            value=latency_ms,
            timestamp=timestamp
        )

    async def run(self):
        """Run continuous generation"""
        while True:
            try:
                await self.generate_heartbeat()
                await self.generate_random_metric()
                await self.generate_latency_metric()

                self.generation_counter += 1

            except Exception as e:
                print(f"Error generating synthetic metrics: {e}")

            await asyncio.sleep(10)  # Generate every 10 seconds

class SyntheticMetricsValidator:
    """Validate that synthetic metrics arrive correctly"""

    def __init__(self, tsdb):
        self.tsdb = tsdb

    async def validate(self):
        """Check synthetic metrics are fresh and correct"""
        now = int(time.time() * 1000)
        one_minute_ago = now - 60000

        # Query synthetic heartbeat
        query = f'''
        synthetic_heartbeat_seconds{{source="monitoring-canary"}}
        '''

        try:
            result = await self.tsdb.query(query)

            if not result:
                return {
                    'status': 'failed',
                    'reason': 'No synthetic metrics found',
                    'severity': 'critical'
                }

            latest_metric = result[0]  # Get latest
            timestamp = latest_metric['timestamp']
            value = latest_metric['value']

            # Check freshness
            age_ms = now - timestamp
            if age_ms > 120000:  # > 2 minutes old
                return {
                    'status': 'failed',
                    'reason': f'Synthetic metrics stale: {age_ms}ms old',
                    'severity': 'critical'
                }

            # Check value correctness
            # Value should be unix timestamp (roughly)
            expected_timestamp = float(int(now / 1000))
            if abs(value - expected_timestamp) > 5:  # Allow 5s skew
                return {
                    'status': 'failed',
                    'reason': f'Synthetic metric value incorrect: {value} vs {expected_timestamp}',
                    'severity': 'critical'
                }

            return {
                'status': 'ok',
                'latency_ms': age_ms,
                'last_metric': timestamp
            }

        except Exception as e:
            return {
                'status': 'failed',
                'reason': f'Query failed: {e}',
                'severity': 'critical'
            }

class MonitoringSystemHealthChecker:
    """Monitor health of entire monitoring stack"""

    def __init__(self, tsdb, alertmanager_url: str):
        self.tsdb = tsdb
        self.alertmanager_url = alertmanager_url
        self.validators = {
            'synthetic_metrics': SyntheticMetricsValidator(tsdb),
        }

    async def check_write_latency(self):
        """Measure write latency"""
        start = time.time()

        await self.tsdb.write(
            metric_name='monitoring_write_latency_test',
            labels={'test': 'true'},
            value=1.0,
            timestamp=int(time.time() * 1000)
        )

        latency_ms = (time.time() - start) * 1000

        return {
            'metric': 'write_latency_ms',
            'value': latency_ms,
            'status': 'ok' if latency_ms < 100 else 'warning' if latency_ms < 1000 else 'critical'
        }

    async def check_query_latency(self):
        """Measure query latency"""
        query = 'http_requests_total'

        start = time.time()

        try:
            result = await self.tsdb.query(query)
            latency_ms = (time.time() - start) * 1000

            return {
                'metric': 'query_latency_ms',
                'value': latency_ms,
                'status': 'ok' if latency_ms < 500 else 'warning' if latency_ms < 5000 else 'critical'
            }
        except Exception as e:
            return {
                'metric': 'query_latency_ms',
                'error': str(e),
                'status': 'critical'
            }

    async def check_alert_delivery(self):
        """Check if alerts are being delivered"""
        # Query AlertManager API
        import aiohttp

        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(f"{self.alertmanager_url}/api/v1/alerts") as resp:
                    if resp.status != 200:
                        return {
                            'metric': 'alertmanager_reachable',
                            'status': 'critical',
                            'error': f'HTTP {resp.status}'
                        }

                    data = await resp.json()
                    alerts = data.get('data', [])

                    # Check for stale alerts (not delivered)
                    stale_count = sum(
                        1 for alert in alerts
                        if alert['state'] == 'firing' and
                        (time.time() - float(alert['startsAt'].timestamp())) > 3600
                    )

                    return {
                        'metric': 'alertmanager_reachable',
                        'status': 'ok' if stale_count == 0 else 'warning',
                        'stale_alerts': stale_count
                    }

        except Exception as e:
            return {
                'metric': 'alertmanager_reachable',
                'status': 'critical',
                'error': str(e)
            }

    async def run_health_check(self):
        """Run complete health check"""
        results = {
            'timestamp': datetime.utcnow().isoformat(),
            'checks': {}
        }

        # Run all checks
        results['checks']['write_latency'] = await self.check_write_latency()
        results['checks']['query_latency'] = await self.check_query_latency()
        results['checks']['synthetic_metrics'] = await self.validators['synthetic_metrics'].validate()
        results['checks']['alert_delivery'] = await self.check_alert_delivery()

        # Overall status
        critical_checks = [
            v for v in results['checks'].values()
            if v.get('status') == 'critical'
        ]

        results['overall_status'] = 'critical' if critical_checks else 'ok'

        return results

# Meta-alerts (alerts about the monitoring system)
META_ALERT_RULES = '''
groups:
  - name: monitoring_system_health
    interval: 1m

    rules:
      # Alert if synthetic metrics are stale
      - alert: SyntheticMetricsStale
        expr: |
          time() - synthetic_heartbeat_seconds{source="monitoring-canary"} > 120
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Monitoring system not ingesting metrics"

      # Alert if query latency is high
      - alert: TSDBQueryLatencyHigh
        expr: |
          monitoring_query_latency_ms > 5000
        for: 5m
        labels:
          severity: warning

      # Alert if write latency is high
      - alert: TSDBWriteLatencyHigh
        expr: |
          monitoring_write_latency_ms > 1000
        for: 5m
        labels:
          severity: warning

      # Alert if cardinality growing too fast
      - alert: CardinalityExplosion
        expr: |
          rate(tsdb_series_created_total[1h]) > 10000
        for: 10m
        labels:
          severity: critical

      # Alert if storage usage growing too fast
      - alert: StorageGrowthTooFast
        expr: |
          rate(tsdb_storage_bytes_total[1d]) > 200e9  # 200GB/day
        for: 1d
        labels:
          severity: warning
```

**Типичные ошибки.**
- Забыть про synthetic monitoring → не знаешь когда monitoring сломан
- Не мониторить сам alertmanager → alerts пропадают undetected
- Недостаточный redundancy → single point of failure
- Не версионировать metrics → compatibility issues с queries
- Игнорировать cardinality explosion → crash при growth

**На интервью.**
- Объясни importance of meta-monitoring
- Упомяни synthetic metrics как canary testing
- Уточняющий вопрос: «Как обнаружить когда monitoring скрывает問題?» → check data consistency, correlation analysis
- Уточняющий вопрос: «Как дизайнить для high availability monitoring?» → active-active setup, data replication

---

## Заключение

Мониторинг и алертинг — это основа observability в production системах. Ключевые принципы:

1. **Выбрать правильную модель сбора** (pull vs push) в зависимости от архитектуры
2. **Оптимизировать хранение** через компрессию и downsampling
3. **Structured logging** с trace IDs для корреляции
4. **Intelligent alerting** с состоянием, дедупликацией, и маршрутизацией
5. **Информативные dashboards** по RED/USE методам
6. **Distributed tracing** для микросервисов
7. **SLI/SLO/SLA** для управления reliability и deployment velocity
8. **Масштабирование** через sharding и aggregation
9. **Meta-monitoring** чтобы мониторить сам monitoring

---

[← Назад к списку тем](README.md)
