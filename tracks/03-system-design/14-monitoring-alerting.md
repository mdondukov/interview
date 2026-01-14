# 14. Monitoring & Alerting System

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Collect metrics from services (CPU, memory, custom)
- Store time-series data
- Query and visualize metrics
- Define alerting rules
- Send notifications (email, Slack, PagerDuty)
- Dashboard creation

### Нефункциональные
- High write throughput (millions of data points/sec)
- Low query latency for dashboards (< 1s)
- High availability (monitoring must be more reliable than monitored systems)
- Data retention (days for high-res, years for aggregated)
- Minimal overhead on monitored services

---

## Capacity Estimation

```
Metrics:
- 10,000 services
- 100 metrics per service
- 1M total metric series
- Collection interval: 10 seconds
- Write QPS: 1M / 10 = 100K data points/sec

Storage:
- Data point: 16 bytes (timestamp + value)
- 100K × 16B × 86400 = 138GB/day (raw)
- With compression: ~20GB/day
- 30 days retention: 600GB
- Aggregated (1yr): ~100GB

Query:
- 1000 dashboards
- 10 panels per dashboard
- 5 queries per panel
- Peak: 50K queries/min
```

---

## High-Level Design

```
┌────────────────────────────────────────────────────────────────────────┐
│                         Data Collection                                 │
└────────────────────────────────────────────────────────────────────────┘

┌──────────┐     ┌──────────┐     ┌──────────┐
│ Service  │     │ Service  │     │ Service  │
│ + Agent  │     │ + Agent  │     │ + Agent  │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────────────┼────────────────┘
                      │
               ┌──────▼──────┐
               │  Collector  │
               │   (Push)    │
               └──────┬──────┘
                      │
┌─────────────────────┼─────────────────────┐
│                     │                     │
│              ┌──────▼──────┐              │
│              │   Kafka     │              │
│              │  (Buffer)   │              │
│              └──────┬──────┘              │
│                     │                     │
│         ┌───────────┼───────────┐         │
│         │           │           │         │
│   ┌─────▼────┐ ┌────▼─────┐ ┌───▼──────┐ │
│   │  TSDB    │ │ Alerting │ │ Stream   │ │
│   │  Writer  │ │ Engine   │ │ Processor│ │
│   └─────┬────┘ └────┬─────┘ └──────────┘ │
│         │           │                     │
│   ┌─────▼────┐ ┌────▼─────┐              │
│   │  Time    │ │  Alert   │              │
│   │  Series  │ │  Manager │              │
│   │    DB    │ └────┬─────┘              │
│   └─────┬────┘      │                     │
│         │      ┌────▼─────┐              │
│         │      │ Notifier │              │
│         │      └──────────┘              │
│         │                                 │
│   ┌─────▼────────────────────────────┐   │
│   │           Query Layer            │   │
│   └─────────────────┬────────────────┘   │
│                     │                     │
└─────────────────────┼─────────────────────┘
                      │
               ┌──────▼──────┐
               │  Dashboard  │
               │   (Grafana) │
               └─────────────┘
```

---

## API Design

### Metrics Ingestion
```json
// Push metrics
POST /api/v1/metrics/write

Request (Prometheus format):
http_requests_total{method="GET",path="/api",status="200"} 1234 1704067200000
http_requests_total{method="POST",path="/api",status="201"} 567 1704067200000
cpu_usage{host="server1",core="0"} 0.75 1704067200000

// Or JSON format
{
    "metrics": [
        {
            "name": "http_requests_total",
            "labels": {"method": "GET", "path": "/api", "status": "200"},
            "value": 1234,
            "timestamp": 1704067200000
        }
    ]
}
```

### Query API (PromQL-like)
```
// Instant query
GET /api/v1/query?query=http_requests_total{status="500"}&time=1704067200

// Range query
GET /api/v1/query_range?
    query=rate(http_requests_total[5m])&
    start=1704067200&
    end=1704153600&
    step=60

Response:
{
    "status": "success",
    "data": {
        "resultType": "matrix",
        "result": [
            {
                "metric": {"method": "GET", "path": "/api"},
                "values": [
                    [1704067200, "1.5"],
                    [1704067260, "1.8"],
                    [1704067320, "2.1"]
                ]
            }
        ]
    }
}
```

### Alert Rules
```yaml
# Alert rule definition
groups:
  - name: service_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
          > 0.5
        for: 10m
        labels:
          severity: warning
```

---

## Data Model

### Time Series Storage

```
┌────────────────────────────────────────────────────────────────┐
│                    Time Series Data Model                       │
├────────────────────────────────────────────────────────────────┤
│ Metric: http_requests_total{method="GET", status="200"}        │
│                                                                │
│ Series ID: hash(metric_name + sorted(labels))                  │
│                                                                │
│ Storage Layout (columnar):                                     │
│                                                                │
│ Block 1 (2h window):                                           │
│ ┌──────────────────────────────────────────────────┐          │
│ │ Timestamps: [t1, t2, t3, t4, ...]  (delta-encoded)│          │
│ │ Values:     [v1, v2, v3, v4, ...]  (XOR-compressed)│         │
│ └──────────────────────────────────────────────────┘          │
│                                                                │
│ Index:                                                         │
│ ┌──────────────────────────────────────────────────┐          │
│ │ Series ID → [Block1_offset, Block2_offset, ...]  │          │
│ │ Label Index: method=GET → [series1, series2]     │          │
│ └──────────────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

### Schema (If using SQL-based TSDB)
```sql
-- Metric metadata
CREATE TABLE metrics (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    labels_hash     BIGINT NOT NULL,
    labels          JSONB NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),

    UNIQUE (name, labels_hash)
);

-- Time series data (partitioned by time)
CREATE TABLE metric_data (
    metric_id       INT NOT NULL,
    timestamp       TIMESTAMP NOT NULL,
    value           DOUBLE PRECISION NOT NULL,

    PRIMARY KEY (metric_id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Create partitions
CREATE TABLE metric_data_2024_01 PARTITION OF metric_data
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### Alert State
```sql
CREATE TABLE alerts (
    id              UUID PRIMARY KEY,
    rule_name       VARCHAR(255) NOT NULL,
    state           VARCHAR(20),  -- pending, firing, resolved
    labels          JSONB,
    annotations     JSONB,
    started_at      TIMESTAMP,
    resolved_at     TIMESTAMP,
    last_eval_at    TIMESTAMP
);

CREATE TABLE alert_notifications (
    id              UUID PRIMARY KEY,
    alert_id        UUID REFERENCES alerts(id),
    channel         VARCHAR(50),  -- email, slack, pagerduty
    status          VARCHAR(20),  -- sent, failed, acknowledged
    sent_at         TIMESTAMP,
    error           TEXT
);
```

---

## Deep Dive

### 1. Metrics Collection

**Pull Model (Prometheus-style)**
```python
class MetricsScraper:
    def __init__(self, targets: list, interval: int = 15):
        self.targets = targets
        self.interval = interval

    async def scrape_loop(self):
        while True:
            start = time.time()

            # Scrape all targets in parallel
            results = await asyncio.gather(*[
                self.scrape_target(target)
                for target in self.targets
            ], return_exceptions=True)

            # Process results
            for target, result in zip(self.targets, results):
                if isinstance(result, Exception):
                    self.record_scrape_failure(target, result)
                else:
                    await self.store_metrics(target, result)

            # Wait for next interval
            elapsed = time.time() - start
            await asyncio.sleep(max(0, self.interval - elapsed))

    async def scrape_target(self, target: dict) -> list:
        url = f"http://{target['host']}:{target['port']}/metrics"
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=10) as response:
                text = await response.text()
                return self.parse_prometheus_format(text)
```

**Push Model (StatsD-style)**
```python
class MetricsReceiver:
    async def handle_metrics(self, request):
        metrics = await request.json()

        # Validate and normalize
        normalized = []
        for metric in metrics['metrics']:
            normalized.append({
                'name': metric['name'],
                'labels': self.normalize_labels(metric.get('labels', {})),
                'value': float(metric['value']),
                'timestamp': metric.get('timestamp', int(time.time() * 1000))
            })

        # Write to buffer (Kafka)
        await self.kafka.send('metrics', normalized)

        return {'status': 'ok', 'count': len(normalized)}
```

### 2. Time Series Database

**Write Path**
```python
class TSDBWriter:
    def __init__(self, storage):
        self.storage = storage
        self.buffer = {}  # series_id -> samples
        self.buffer_size = 1000

    async def write(self, metric: dict):
        series_id = self.get_series_id(metric['name'], metric['labels'])

        # Buffer writes
        if series_id not in self.buffer:
            self.buffer[series_id] = []

        self.buffer[series_id].append({
            'timestamp': metric['timestamp'],
            'value': metric['value']
        })

        # Flush if buffer full
        if len(self.buffer[series_id]) >= self.buffer_size:
            await self.flush_series(series_id)

    async def flush_series(self, series_id: str):
        samples = self.buffer.pop(series_id, [])
        if not samples:
            return

        # Compress samples
        compressed = self.compress_samples(samples)

        # Write to storage
        await self.storage.write_block(series_id, compressed)

    def compress_samples(self, samples: list) -> bytes:
        """
        Gorilla compression:
        - Timestamps: Delta-of-delta encoding
        - Values: XOR with previous value
        """
        timestamps = [s['timestamp'] for s in samples]
        values = [s['value'] for s in samples]

        # Delta-of-delta for timestamps
        ts_compressed = self.delta_encode(timestamps)

        # XOR for values (floating point)
        val_compressed = self.xor_encode(values)

        return self.pack(ts_compressed, val_compressed)
```

**Query Path**
```python
class TSDBQuerier:
    async def query_range(self, query: str, start: int, end: int, step: int):
        # 1. Parse query
        ast = self.parse_promql(query)

        # 2. Find matching series
        matchers = self.extract_matchers(ast)
        series_ids = await self.find_series(matchers)

        # 3. Fetch data for time range
        data = await asyncio.gather(*[
            self.fetch_series_data(sid, start, end)
            for sid in series_ids
        ])

        # 4. Execute query operations (rate, sum, avg, etc.)
        result = self.execute_query(ast, data, step)

        return result

    async def fetch_series_data(self, series_id: str, start: int, end: int):
        # Find relevant blocks
        blocks = await self.storage.find_blocks(series_id, start, end)

        # Read and decompress
        samples = []
        for block in blocks:
            block_data = await self.storage.read_block(block)
            samples.extend(self.decompress(block_data))

        # Filter to time range
        return [s for s in samples if start <= s['timestamp'] <= end]
```

### 3. Alerting Engine

```python
class AlertingEngine:
    def __init__(self, rules: list, tsdb: TSDBQuerier):
        self.rules = rules
        self.tsdb = tsdb
        self.alert_states = {}  # rule_id -> AlertState

    async def evaluation_loop(self):
        while True:
            for rule in self.rules:
                await self.evaluate_rule(rule)

            await asyncio.sleep(rule.evaluation_interval)

    async def evaluate_rule(self, rule: AlertRule):
        # Execute query
        result = await self.tsdb.query(rule.expr)

        # Check threshold
        for series in result:
            if series.value > rule.threshold:
                await self.fire_alert(rule, series)
            else:
                await self.resolve_alert(rule, series)

    async def fire_alert(self, rule: AlertRule, series):
        alert_key = self.get_alert_key(rule, series)
        state = self.alert_states.get(alert_key)

        if state is None:
            # New alert - start pending
            self.alert_states[alert_key] = AlertState(
                status='pending',
                started_at=datetime.utcnow()
            )

        elif state.status == 'pending':
            # Check if pending long enough
            pending_duration = datetime.utcnow() - state.started_at
            if pending_duration >= rule.for_duration:
                state.status = 'firing'
                await self.send_notification(rule, series, 'firing')

    async def resolve_alert(self, rule: AlertRule, series):
        alert_key = self.get_alert_key(rule, series)
        state = self.alert_states.get(alert_key)

        if state and state.status == 'firing':
            state.status = 'resolved'
            state.resolved_at = datetime.utcnow()
            await self.send_notification(rule, series, 'resolved')
```

### 4. Notification System

```python
class NotificationManager:
    def __init__(self):
        self.channels = {
            'email': EmailChannel(),
            'slack': SlackChannel(),
            'pagerduty': PagerDutyChannel(),
            'webhook': WebhookChannel()
        }
        self.routing_rules = []  # label-based routing

    async def send_notification(self, alert: Alert, event_type: str):
        # Find receivers based on routing rules
        receivers = self.route_alert(alert)

        # Send to all matched receivers
        for receiver in receivers:
            channel = self.channels.get(receiver.channel)
            if channel:
                try:
                    await channel.send(
                        receiver.config,
                        self.format_message(alert, event_type)
                    )
                    await self.record_notification(alert, receiver, 'sent')
                except Exception as e:
                    await self.record_notification(alert, receiver, 'failed', str(e))

    def route_alert(self, alert: Alert) -> list:
        """Route alert based on labels"""
        receivers = []
        for rule in self.routing_rules:
            if self.matches(alert.labels, rule.matchers):
                receivers.extend(rule.receivers)
                if not rule.continue_matching:
                    break
        return receivers

class SlackChannel:
    async def send(self, config: dict, message: dict):
        webhook_url = config['webhook_url']
        payload = {
            'channel': config.get('channel', '#alerts'),
            'username': 'AlertBot',
            'attachments': [{
                'color': 'danger' if message['status'] == 'firing' else 'good',
                'title': message['alert_name'],
                'text': message['description'],
                'fields': [
                    {'title': k, 'value': v, 'short': True}
                    for k, v in message['labels'].items()
                ]
            }]
        }

        async with aiohttp.ClientSession() as session:
            await session.post(webhook_url, json=payload)
```

### 5. Data Retention & Downsampling

```python
class RetentionManager:
    RETENTION_POLICIES = [
        {'resolution': '10s', 'retention': '7d'},
        {'resolution': '1m', 'retention': '30d'},
        {'resolution': '5m', 'retention': '90d'},
        {'resolution': '1h', 'retention': '365d'},
    ]

    async def downsample_job(self):
        """Run periodically to downsample old data"""
        for i, policy in enumerate(self.RETENTION_POLICIES[:-1]):
            next_policy = self.RETENTION_POLICIES[i + 1]

            # Find data older than retention but not yet downsampled
            cutoff = datetime.utcnow() - parse_duration(policy['retention'])

            # Aggregate data
            await self.aggregate_data(
                source_resolution=policy['resolution'],
                target_resolution=next_policy['resolution'],
                cutoff=cutoff
            )

    async def aggregate_data(self, source_resolution, target_resolution, cutoff):
        # For each series
        async for series in self.iter_series():
            # Read old high-res data
            data = await self.read_data(series.id, end=cutoff, resolution=source_resolution)

            # Aggregate (avg, min, max, count)
            aggregated = self.aggregate(data, target_resolution)

            # Write aggregated data
            await self.write_aggregated(series.id, aggregated, target_resolution)

            # Delete old high-res data
            await self.delete_data(series.id, end=cutoff, resolution=source_resolution)
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| High cardinality | Label limits, cardinality analysis |
| Write throughput | Buffering, batching, sharding |
| Query performance | Pre-aggregation, caching, indexes |
| Storage growth | Compression, downsampling, retention |
| Alert storms | Grouping, inhibition, silencing |

---

## Trade-offs

### Push vs Pull

| Model | Pros | Cons |
|-------|------|------|
| Pull | Central control, service discovery | Firewall issues, scale limits |
| Push | Works anywhere, lower latency | Requires client config, flooding risk |

### Storage

| Engine | Write | Query | Scale |
|--------|-------|-------|-------|
| Prometheus | Fast | Good | Single node |
| InfluxDB | Fast | Good | Clustering |
| TimescaleDB | Medium | Best | PostgreSQL ecosystem |
| VictoriaMetrics | Very fast | Good | Excellent |

---

## На интервью

### Ключевые моменты
1. **Data model** — time series, labels, cardinality
2. **Storage** — compression, downsampling, retention
3. **Query language** — aggregations, rate calculations
4. **Alerting** — evaluation, state machine, notification routing

### Типичные follow-up
- Как обработать high cardinality metrics?
- Как масштабировать на миллионы серий?
- Как реализовать cross-datacenter monitoring?
- Как избежать alert fatigue?
- Как мониторить саму систему мониторинга?

---

[← Назад к списку тем](README.md)
