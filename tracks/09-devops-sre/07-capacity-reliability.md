# 07. Capacity & Reliability

[← Назад к списку тем](README.md)

---

## Capacity Planning

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Capacity Planning Process                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. UNDERSTAND CURRENT STATE                                        │
│     - Current resource usage                                        │
│     - Peak vs average load                                          │
│     - Growth trends                                                 │
│                                                                     │
│  2. FORECAST DEMAND                                                 │
│     - Business growth projections                                   │
│     - Seasonal patterns                                             │
│     - Planned launches/campaigns                                    │
│                                                                     │
│  3. MODEL CAPACITY                                                  │
│     - Resources needed per unit of demand                           │
│     - Scaling characteristics                                       │
│     - Dependencies and bottlenecks                                  │
│                                                                     │
│  4. PLAN PROVISIONING                                               │
│     - Lead time for new capacity                                    │
│     - Cost considerations                                           │
│     - Buffer for unexpected growth                                  │
│                                                                     │
│  5. VALIDATE AND ITERATE                                            │
│     - Load testing                                                  │
│     - Actual vs predicted                                           │
│     - Adjust models                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Capacity Metrics

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Key Capacity Metrics                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  UTILIZATION                                                        │
│  - CPU: target 60-70% average, allow spikes to 80%                  │
│  - Memory: target 70-80%                                            │
│  - Disk: alert at 80%, critical at 90%                              │
│  - Network: depends on NIC capacity                                 │
│                                                                     │
│  SATURATION                                                         │
│  - Queue depth                                                      │
│  - Thread pool exhaustion                                           │
│  - Connection pool utilization                                      │
│                                                                     │
│  THROUGHPUT                                                         │
│  - Requests per second                                              │
│  - Transactions per second                                          │
│  - Messages processed per second                                    │
│                                                                     │
│  HEADROOM                                                           │
│  - Current capacity - current usage                                 │
│  - "How much more can we handle?"                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Load Testing

### Types of Load Tests

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Load Test Types                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SMOKE TEST                                                         │
│  - Minimal load                                                     │
│  - Verify system works                                              │
│  - Quick sanity check                                               │
│                                                                     │
│  LOAD TEST                                                          │
│  - Expected production load                                         │
│  - Verify SLOs met                                                  │
│  - Find baseline performance                                        │
│                                                                     │
│  STRESS TEST                                                        │
│  - Beyond expected load                                             │
│  - Find breaking points                                             │
│  - Understand degradation                                           │
│                                                                     │
│  SPIKE TEST                                                         │
│  - Sudden load increase                                             │
│  - Test autoscaling                                                 │
│  - Simulate traffic spikes                                          │
│                                                                     │
│  SOAK TEST                                                          │
│  - Sustained load over time                                         │
│  - Find memory leaks                                                │
│  - Test stability                                                   │
│                                                                     │
│            ▲                                                        │
│   Load    │     ╭──────╮    Stress                                  │
│           │    ╱        ╲                                           │
│           │  ╭╯          ╰────────────  Load                        │
│           │ ╱                                                       │
│           │╱                                                        │
│           └─────────────────────────▶ Time                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### K6 Example

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up
    { duration: '5m', target: 100 },  // Hold
    { duration: '2m', target: 200 },  // Spike
    { duration: '5m', target: 100 },  // Back to normal
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% < 500ms
    http_req_failed: ['rate<0.01'],    // <1% errors
  },
};

export default function () {
  const res = http.get('https://api.example.com/users');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

```bash
# Run test
k6 run load-test.js

# Run with more output
k6 run --out influxdb=http://localhost:8086/k6 load-test.js
```

---

## Chaos Engineering

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Chaos Engineering                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRINCIPLES                                                         │
│  ──────────                                                         │
│  1. Define "steady state" (normal behavior)                         │
│  2. Hypothesize: "System tolerates failure X"                       │
│  3. Inject real-world failure                                       │
│  4. Observe if steady state maintained                              │
│  5. Fix weaknesses found                                            │
│                                                                     │
│  FAILURE TYPES TO INJECT                                            │
│  ─────────────────────────                                          │
│  - Kill instances/pods                                              │
│  - Network latency/partition                                        │
│  - CPU/memory stress                                                │
│  - Disk failure                                                     │
│  - Dependency unavailable                                           │
│  - Clock skew                                                       │
│                                                                     │
│  SAFETY                                                             │
│  ──────                                                             │
│  - Start small (staging first)                                      │
│  - Have abort button                                                │
│  - Limit blast radius                                               │
│  - Run during business hours                                        │
│  - Communicate with stakeholders                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Chaos Monkey / Litmus Example

```yaml
# litmus-chaos experiment
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-kill-chaos
spec:
  appinfo:
    appns: 'default'
    applabel: 'app=nginx'
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '30'
            - name: CHAOS_INTERVAL
              value: '10'
            - name: FORCE
              value: 'false'
```

### Game Days

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Game Day Planning                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PREPARATION                                                        │
│  - Define scenarios                                                 │
│  - Notify stakeholders                                              │
│  - Have rollback plan                                               │
│  - Prepare monitoring                                               │
│                                                                     │
│  EXECUTION                                                          │
│  - Run scenario                                                     │
│  - Observe system behavior                                          │
│  - Test response procedures                                         │
│  - Document findings                                                │
│                                                                     │
│  FOLLOW-UP                                                          │
│  - Review findings                                                  │
│  - Create action items                                              │
│  - Update runbooks                                                  │
│  - Schedule next game day                                           │
│                                                                     │
│  EXAMPLE SCENARIOS                                                  │
│  - "What if database primary fails?"                                │
│  - "What if region goes down?"                                      │
│  - "What if third-party API is slow?"                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Reliability Patterns

### Redundancy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Redundancy Patterns                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  HORIZONTAL REDUNDANCY                                              │
│  Multiple instances behind load balancer                            │
│  ┌───────┐    ┌───────┐    ┌───────┐                               │
│  │ App 1 │    │ App 2 │    │ App 3 │                               │
│  └───────┘    └───────┘    └───────┘                               │
│       ▲            ▲            ▲                                   │
│       └────────────┼────────────┘                                   │
│                    │                                                │
│            ┌──────────────┐                                         │
│            │ Load Balancer│                                         │
│            └──────────────┘                                         │
│                                                                     │
│  GEOGRAPHIC REDUNDANCY                                              │
│  Multiple regions/AZs                                               │
│                                                                     │
│  DATA REDUNDANCY                                                    │
│  - Replication (sync/async)                                         │
│  - Backups                                                          │
│  - Multi-region databases                                           │
│                                                                     │
│  N+1 / N+2 REDUNDANCY                                               │
│  - N+1: One extra instance for failures                             │
│  - N+2: Two extra for failures + maintenance                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Graceful Degradation

```go
// Graceful degradation example
func getRecommendations(ctx context.Context, userID string) ([]Recommendation, error) {
    // Try personalized recommendations
    recs, err := personalizedService.Get(ctx, userID)
    if err == nil {
        return recs, nil
    }

    // Fall back to cached recommendations
    log.Warn("personalized service failed, using cache", "error", err)
    recs, err = cache.Get(ctx, "popular_recs")
    if err == nil {
        return recs, nil
    }

    // Fall back to static popular items
    log.Warn("cache failed, using static fallback", "error", err)
    return staticPopularItems, nil
}
```

### Circuit Breaker

```go
// Circuit breaker pattern
type CircuitBreaker struct {
    failures    int
    threshold   int
    timeout     time.Duration
    lastFailure time.Time
    state       string // closed, open, half-open
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if cb.state == "open" {
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = "half-open"
        } else {
            return ErrCircuitOpen
        }
    }

    err := fn()
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.threshold {
            cb.state = "open"
        }
        return err
    }

    cb.failures = 0
    cb.state = "closed"
    return nil
}
```

---

## Availability Math

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Availability Calculations                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  AVAILABILITY = Uptime / Total Time                                 │
│                                                                     │
│  Nines    Availability    Downtime/Year    Downtime/Month           │
│  ─────    ────────────    ─────────────    ──────────────           │
│  2 nines     99%          3.65 days        7.3 hours                │
│  3 nines     99.9%        8.76 hours       43.8 minutes             │
│  4 nines     99.99%       52.6 minutes     4.38 minutes             │
│  5 nines     99.999%      5.26 minutes     26 seconds               │
│                                                                     │
│  SERIAL DEPENDENCIES                                                │
│  A → B → C                                                          │
│  Total = A × B × C                                                  │
│  99.9% × 99.9% × 99.9% = 99.7%                                      │
│                                                                     │
│  PARALLEL REDUNDANCY                                                │
│  A or B (either can serve)                                          │
│  Total = 1 - (1-A) × (1-B)                                          │
│  1 - (0.001 × 0.001) = 99.9999%                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Incident Management](./06-incident-management.md) — реагирование на инциденты при исчерпании ёмкости
- [Design Fundamentals](../03-system-design/00-design-fundamentals.md) — основы проектирования надёжных систем

---

## На интервью

### Типичные вопросы

1. **How do you do capacity planning?**
   - Understand current usage
   - Forecast growth
   - Model resource needs
   - Plan provisioning with buffer
   - Validate with load tests

2. **Types of load testing?**
   - Smoke: sanity check
   - Load: expected traffic
   - Stress: beyond capacity
   - Spike: sudden increase
   - Soak: sustained over time

3. **What is chaos engineering?**
   - Proactively inject failures
   - Test system resilience
   - Find weaknesses before production
   - Examples: kill pods, add latency, simulate outages

4. **How do you achieve high availability?**
   - Redundancy (N+1, N+2)
   - Multiple AZs/regions
   - Load balancing
   - Graceful degradation
   - Circuit breakers

5. **How do dependencies affect availability?**
   - Serial: multiply availabilities (gets worse)
   - Parallel: improve with redundancy
   - Minimize critical path dependencies

6. **What's graceful degradation?**
   - Provide partial service when components fail
   - Fallback to cached data, static content
   - Better UX than complete failure

---

[← Назад к списку тем](README.md)
