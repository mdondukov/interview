# 05. Monitoring & Observability

[← Назад к списку тем](README.md)

---

## Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Three Pillars                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  METRICS                                                            │
│  ───────                                                            │
│  Numerical data over time                                           │
│  - CPU, memory, request rate, error rate                            │
│  - Aggregatable: sum, avg, percentiles                              │
│  - Low cardinality                                                  │
│  - Good for: dashboards, alerts, trends                             │
│                                                                     │
│  LOGS                                                               │
│  ────                                                               │
│  Discrete events with context                                       │
│  - Request details, errors, audit trail                             │
│  - High cardinality                                                 │
│  - Structured (JSON) preferred                                      │
│  - Good for: debugging, forensics                                   │
│                                                                     │
│  TRACES                                                             │
│  ──────                                                             │
│  Request flow across services                                       │
│  - Spans with timing, parent-child                                  │
│  - Distributed context propagation                                  │
│  - Good for: latency analysis, dependency mapping                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Metrics

### Metric Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Metric Types (Prometheus)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Counter                                                            │
│  - Only increases (or resets to 0)                                  │
│  - Examples: requests_total, errors_total                           │
│  - Use rate() to get per-second rate                                │
│                                                                     │
│  Gauge                                                              │
│  - Can go up and down                                               │
│  - Examples: temperature, queue_size, active_connections            │
│                                                                     │
│  Histogram                                                          │
│  - Observations in buckets                                          │
│  - Examples: request_duration_seconds                               │
│  - Pre-defined buckets: 0.01, 0.05, 0.1, 0.5, 1, 5 seconds         │
│  - Can calculate percentiles                                        │
│                                                                     │
│  Summary                                                            │
│  - Pre-calculated percentiles                                       │
│  - Less flexible than histogram                                     │
│  - Cannot aggregate across instances                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Four Golden Signals

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Four Golden Signals                               │
│                    (Google SRE Book)                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. LATENCY                                                         │
│     Time to service a request                                       │
│     - Separate successful vs failed requests                        │
│     - Track percentiles (p50, p90, p99)                             │
│                                                                     │
│  2. TRAFFIC                                                         │
│     Demand on the system                                            │
│     - HTTP requests/sec                                             │
│     - Transactions/sec                                              │
│                                                                     │
│  3. ERRORS                                                          │
│     Rate of failed requests                                         │
│     - HTTP 5xx                                                      │
│     - Failed transactions                                           │
│                                                                     │
│  4. SATURATION                                                      │
│     How "full" the system is                                        │
│     - CPU utilization                                               │
│     - Memory usage                                                  │
│     - Queue depth                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Prometheus Queries (PromQL)

```promql
# Request rate (per second)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# 99th percentile latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Memory usage percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
  /
node_memory_MemTotal_bytes * 100

# Top 5 endpoints by request rate
topk(5, sum by (endpoint) (rate(http_requests_total[5m])))

# Increase over time (absolute, not rate)
increase(http_requests_total[1h])
```

---

## SLOs, SLIs, SLAs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SLO / SLI / SLA                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SLI (Service Level Indicator)                                      │
│  ─────────────────────────────                                      │
│  Quantitative measure of service behavior                           │
│  - Request latency                                                  │
│  - Error rate                                                       │
│  - Throughput                                                       │
│  - Availability                                                     │
│                                                                     │
│  SLO (Service Level Objective)                                      │
│  ─────────────────────────────                                      │
│  Target value for an SLI                                            │
│  - 99.9% of requests < 200ms                                        │
│  - Error rate < 0.1%                                                │
│  - 99.95% availability                                              │
│                                                                     │
│  SLA (Service Level Agreement)                                      │
│  ─────────────────────────────                                      │
│  Contract with consequences                                         │
│  - SLO + business/legal terms                                       │
│  - Credits, refunds if breached                                     │
│  - Usually less strict than SLO                                     │
│                                                                     │
│  Error Budget:                                                      │
│  - 99.9% SLO = 0.1% error budget                                    │
│  - ~43 min/month of allowed downtime                                │
│  - Use to balance reliability vs velocity                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Error Budget Example

```
SLO: 99.9% availability (monthly)

30 days × 24 hours × 60 minutes = 43,200 minutes

Error budget = 43,200 × 0.1% = 43.2 minutes

If 30 minutes used → 13 minutes remaining
If 50 minutes used → Budget exceeded, freeze deployments
```

---

## Logging

### Structured Logging

```go
// Bad: Unstructured
log.Printf("User %s logged in from %s", userID, ip)

// Good: Structured (JSON)
logger.Info("user logged in",
    zap.String("user_id", userID),
    zap.String("ip", ip),
    zap.String("request_id", requestID),
)

// Output:
// {"level":"info","msg":"user logged in","user_id":"123","ip":"1.2.3.4","request_id":"abc"}
```

### Log Levels

```
┌─────────────────┬──────────────────────────────────────────────────┐
│ Level           │ When to use                                      │
├─────────────────┼──────────────────────────────────────────────────┤
│ DEBUG           │ Detailed troubleshooting (not in production)     │
│ INFO            │ Normal operations, events                        │
│ WARN            │ Unexpected but handled situations                │
│ ERROR           │ Errors that need attention                       │
│ FATAL           │ App cannot continue                              │
└─────────────────┴──────────────────────────────────────────────────┘
```

### Log Aggregation Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ELK / EFK Stack                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Application → Fluentd/Filebeat → Elasticsearch → Kibana            │
│                                                                     │
│  Or:                                                                │
│  Application → Loki → Grafana                                       │
│                                                                     │
│  Or (cloud):                                                        │
│  Application → CloudWatch Logs                                      │
│  Application → Datadog                                              │
│  Application → Splunk                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Distributed Tracing

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Distributed Trace                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Request: GET /api/orders/123                                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Trace ID: abc123                                             │   │
│  │                                                             │   │
│  │ Span: API Gateway (50ms)                                    │   │
│  │ ├── Span: Auth Service (10ms)                               │   │
│  │ └── Span: Order Service (35ms)                              │   │
│  │     ├── Span: Database Query (15ms)                         │   │
│  │     └── Span: Cache Lookup (5ms)                            │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Key concepts:                                                      │
│  - Trace: End-to-end request journey                                │
│  - Span: Single operation                                           │
│  - Context propagation: Pass trace ID between services              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### OpenTelemetry

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func handleRequest(ctx context.Context) {
    // Start span
    ctx, span := otel.Tracer("my-service").Start(ctx, "handleRequest")
    defer span.End()

    // Add attributes
    span.SetAttributes(
        attribute.String("user.id", userID),
        attribute.Int("order.count", len(orders)),
    )

    // Call downstream service (context propagated)
    result, err := orderService.GetOrder(ctx, orderID)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
    }
}
```

---

## Alerting

### Alert Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Good Alerting                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Actionable                                                         │
│  - Every page requires human action                                 │
│  - If nothing to do, it's not a page                                │
│                                                                     │
│  Symptom-based (not cause)                                          │
│  - Alert on: "Error rate > 1%"                                      │
│  - Not on: "CPU > 80%" (might not affect users)                     │
│                                                                     │
│  With good signal-to-noise                                          │
│  - Too many alerts = alert fatigue                                  │
│  - Better to miss than over-alert                                   │
│                                                                     │
│  Clear runbooks                                                     │
│  - Every alert has documentation                                    │
│  - What it means                                                    │
│  - How to investigate                                               │
│  - How to mitigate                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Prometheus Alerting Rules

```yaml
# prometheus alerting rules
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate"
          description: "Error rate is {{ $value | humanizePercentage }}"
          runbook: "https://wiki/runbooks/high-error-rate"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency"
          description: "p99 latency is {{ $value }}s"
```

---

## Dashboards

### Dashboard Design

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Dashboard Layout                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Top: Key metrics at a glance (SLIs)                                │
│  ┌─────────┬─────────┬─────────┬─────────┐                          │
│  │ Uptime  │ Error   │ p99     │ RPS     │                          │
│  │ 99.9%   │ 0.05%   │ 120ms   │ 5,230   │                          │
│  └─────────┴─────────┴─────────┴─────────┘                          │
│                                                                     │
│  Middle: Time series graphs                                         │
│  ┌────────────────────────────────────────┐                         │
│  │ Request Rate                           │                         │
│  │  ▄▄▆▆████▆▆▄▄▄▄▆▆████▆▆▄▄              │                         │
│  └────────────────────────────────────────┘                         │
│                                                                     │
│  Bottom: Details, logs, traces                                      │
│  ┌────────────────────────────────────────┐                         │
│  │ Recent Errors                          │                         │
│  │ - 500 /api/orders timeout              │                         │
│  │ - 500 /api/users db connection         │                         │
│  └────────────────────────────────────────┘                         │
│                                                                     │
│  Principles:                                                        │
│  - Most important at top                                            │
│  - Use consistent colors                                            │
│  - Include time selector                                            │
│  - Link to related dashboards                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Monitoring & Alerting](../03-system-design/14-monitoring-alerting.md) — проектирование систем мониторинга в распределённых системах
- [Incident Management](./06-incident-management.md) — использование мониторинга для обнаружения и реагирования на инциденты

---

## На интервью

### Типичные вопросы

1. **Three pillars of observability?**
   - Metrics: numerical data over time
   - Logs: discrete events with context
   - Traces: request flow across services

2. **Four golden signals?**
   - Latency, Traffic, Errors, Saturation
   - Focus on user-facing symptoms

3. **SLO vs SLA?**
   - SLO: internal target (99.9% availability)
   - SLA: contract with consequences
   - SLA usually less strict than SLO

4. **How to design good alerts?**
   - Actionable: requires human response
   - Symptom-based: not cause-based
   - Low noise: avoid alert fatigue
   - With runbooks

5. **Metrics vs Logs?**
   - Metrics: aggregatable, low cardinality, trends
   - Logs: high cardinality, forensics, debugging
   - Use metrics for dashboards/alerts, logs for investigation

6. **What is error budget?**
   - Allowed downtime based on SLO
   - 99.9% = 43 min/month budget
   - Balance reliability vs velocity

---

[← Назад к списку тем](README.md)
