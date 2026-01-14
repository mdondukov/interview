# 08. Resilience Patterns

[← Назад к списку тем](README.md)

---

## Circuit Breaker

Предотвращает каскадные сбои, прекращая запросы к failing сервису.

### State Machine

```
┌────────────────────────────────────────────────────────────────────┐
│                    Circuit Breaker States                           │
│                                                                    │
│                         failures >= threshold                       │
│     ┌────────────┐  ─────────────────────────▶  ┌────────────┐     │
│     │   CLOSED   │                              │    OPEN    │     │
│     │  (normal)  │  ◀─────────────────────────  │  (failing) │     │
│     └────────────┘      success in half-open    └──────┬─────┘     │
│           │                                            │            │
│           │                                            │            │
│           │                  timeout                   │            │
│           │              ┌────────────┐                │            │
│           └─────────────▶│ HALF-OPEN  │◀───────────────┘            │
│              probe       │  (testing) │                             │
│                          └────────────┘                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Implementation

```go
type CircuitBreaker struct {
    mu              sync.RWMutex
    state           State
    failures        int
    successes       int
    lastFailure     time.Time

    // Configuration
    failureThreshold int           // failures to open
    successThreshold int           // successes to close
    timeout          time.Duration // time in open before half-open
}

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

func NewCircuitBreaker(failureThreshold, successThreshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:            StateClosed,
        failureThreshold: failureThreshold,
        successThreshold: successThreshold,
        timeout:          timeout,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if !cb.canExecute() {
        return ErrCircuitOpen
    }

    err := fn()

    cb.recordResult(err)

    return err
}

func (cb *CircuitBreaker) canExecute() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateClosed:
        return true
    case StateOpen:
        // Check if timeout passed
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
            return true
        }
        return false
    case StateHalfOpen:
        return true
    }
    return false
}

func (cb *CircuitBreaker) recordResult(err error) {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()

        if cb.state == StateHalfOpen {
            cb.state = StateOpen
        } else if cb.failures >= cb.failureThreshold {
            cb.state = StateOpen
        }
    } else {
        if cb.state == StateHalfOpen {
            cb.successes++
            if cb.successes >= cb.successThreshold {
                cb.state = StateClosed
                cb.failures = 0
            }
        } else {
            cb.failures = 0
        }
    }
}

// Usage
func (c *OrderClient) GetOrder(ctx context.Context, id string) (*Order, error) {
    var order *Order

    err := c.circuitBreaker.Execute(func() error {
        var err error
        order, err = c.httpClient.Get(ctx, "/orders/"+id)
        return err
    })

    if errors.Is(err, ErrCircuitOpen) {
        // Return cached data or default
        return c.cache.Get(id), nil
    }

    return order, err
}
```

### Using sony/gobreaker

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "order-service",
    MaxRequests: 3,                // max requests in half-open
    Interval:    10 * time.Second, // reset interval in closed
    Timeout:     30 * time.Second, // time in open before half-open
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 3 && failureRatio >= 0.6
    },
    OnStateChange: func(name string, from, to gobreaker.State) {
        log.Printf("Circuit breaker %s: %s -> %s", name, from, to)
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return client.GetOrder(ctx, orderID)
})
```

---

## Retry with Backoff

```go
type RetryConfig struct {
    MaxAttempts     int
    InitialInterval time.Duration
    MaxInterval     time.Duration
    Multiplier      float64
    Jitter          float64
}

func DefaultRetryConfig() RetryConfig {
    return RetryConfig{
        MaxAttempts:     3,
        InitialInterval: 100 * time.Millisecond,
        MaxInterval:     10 * time.Second,
        Multiplier:      2.0,
        Jitter:          0.1,
    }
}

func Retry(ctx context.Context, cfg RetryConfig, fn func() error) error {
    var lastErr error
    interval := cfg.InitialInterval

    for attempt := 0; attempt < cfg.MaxAttempts; attempt++ {
        if err := fn(); err != nil {
            lastErr = err

            // Check if retryable
            if !isRetryable(err) {
                return err
            }

            // Check context
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
            }

            // Calculate backoff with jitter
            jitter := interval.Seconds() * cfg.Jitter * (rand.Float64()*2 - 1)
            sleep := interval + time.Duration(jitter*float64(time.Second))

            time.Sleep(sleep)

            // Increase interval for next attempt
            interval = time.Duration(float64(interval) * cfg.Multiplier)
            if interval > cfg.MaxInterval {
                interval = cfg.MaxInterval
            }
        } else {
            return nil
        }
    }

    return fmt.Errorf("max retries exceeded: %w", lastErr)
}

func isRetryable(err error) bool {
    // Network errors, 5xx, timeout - retryable
    // 4xx (except 429), business errors - not retryable
    var netErr net.Error
    if errors.As(err, &netErr) {
        // Примечание: netErr.Temporary() deprecated, используем только Timeout()
        return netErr.Timeout()
    }

    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode >= 500 || httpErr.StatusCode == 429
    }

    return false
}
```

---

## Timeout

```go
func (c *Client) GetWithTimeout(ctx context.Context, url string) (*Response, error) {
    // Request-level timeout
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    resp, err := c.httpClient.Do(req)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, ErrTimeout
        }
        return nil, err
    }

    return resp, nil
}

// HTTP Client with timeouts
client := &http.Client{
    Timeout: 30 * time.Second, // Total request timeout
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,  // Connection timeout
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   5 * time.Second,
        ResponseHeaderTimeout: 10 * time.Second,
        IdleConnTimeout:       90 * time.Second,
        MaxIdleConns:          100,
        MaxIdleConnsPerHost:   10,
    },
}
```

---

## Bulkhead

Изоляция ресурсов для предотвращения cascade failures.

### Semaphore Bulkhead

```go
type Bulkhead struct {
    sem chan struct{}
}

func NewBulkhead(maxConcurrent int) *Bulkhead {
    return &Bulkhead{
        sem: make(chan struct{}, maxConcurrent),
    }
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()
    case <-ctx.Done():
        return ctx.Err()
    default:
        return ErrBulkheadFull
    }
}

// Usage: separate bulkheads per dependency
type OrderService struct {
    paymentBulkhead   *Bulkhead
    inventoryBulkhead *Bulkhead
}

func NewOrderService() *OrderService {
    return &OrderService{
        paymentBulkhead:   NewBulkhead(10), // max 10 concurrent payment calls
        inventoryBulkhead: NewBulkhead(20), // max 20 concurrent inventory calls
    }
}

func (s *OrderService) ProcessOrder(ctx context.Context, order *Order) error {
    // Payment call is isolated
    err := s.paymentBulkhead.Execute(ctx, func() error {
        return s.paymentClient.Charge(ctx, order)
    })
    if errors.Is(err, ErrBulkheadFull) {
        return ErrServiceOverloaded
    }

    // ...
}
```

### Thread Pool Bulkhead

```go
type WorkerPool struct {
    tasks   chan func()
    workers int
}

func NewWorkerPool(workers, queueSize int) *WorkerPool {
    pool := &WorkerPool{
        tasks:   make(chan func(), queueSize),
        workers: workers,
    }

    for i := 0; i < workers; i++ {
        go pool.worker()
    }

    return pool
}

func (p *WorkerPool) worker() {
    for task := range p.tasks {
        task()
    }
}

func (p *WorkerPool) Submit(task func()) error {
    select {
    case p.tasks <- task:
        return nil
    default:
        return ErrQueueFull
    }
}
```

---

## Fallback

```go
type FallbackClient struct {
    primary   OrderClient
    fallback  OrderClient
    cache     Cache
}

func (c *FallbackClient) GetOrder(ctx context.Context, id string) (*Order, error) {
    // Try primary
    order, err := c.primary.GetOrder(ctx, id)
    if err == nil {
        c.cache.Set(id, order)
        return order, nil
    }

    log.Printf("Primary failed: %v, trying fallback", err)

    // Try fallback service
    order, err = c.fallback.GetOrder(ctx, id)
    if err == nil {
        return order, nil
    }

    log.Printf("Fallback failed: %v, trying cache", err)

    // Return cached data
    if cached, ok := c.cache.Get(id); ok {
        return cached, nil
    }

    // Return default/degraded response
    return &Order{
        ID:     id,
        Status: "unknown",
    }, ErrDegradedResponse
}
```

---

## Rate Limiting (Client-side)

```go
import "golang.org/x/time/rate"

type RateLimitedClient struct {
    client  *http.Client
    limiter *rate.Limiter
}

func NewRateLimitedClient(rps int) *RateLimitedClient {
    return &RateLimitedClient{
        client:  http.DefaultClient,
        limiter: rate.NewLimiter(rate.Limit(rps), rps), // rps requests per second, burst = rps
    }
}

func (c *RateLimitedClient) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    // Wait for rate limiter
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }

    return c.client.Do(req)
}
```

---

## Health Checks

```go
type HealthChecker struct {
    checks map[string]HealthCheck
}

type HealthCheck func(ctx context.Context) error

type HealthStatus struct {
    Status  string                   `json:"status"` // "healthy", "degraded", "unhealthy"
    Checks  map[string]CheckResult   `json:"checks"`
}

type CheckResult struct {
    Status  string `json:"status"`
    Message string `json:"message,omitempty"`
}

func (h *HealthChecker) Check(ctx context.Context) HealthStatus {
    results := make(map[string]CheckResult)
    allHealthy := true
    anyHealthy := false

    for name, check := range h.checks {
        err := check(ctx)
        if err != nil {
            results[name] = CheckResult{Status: "unhealthy", Message: err.Error()}
            allHealthy = false
        } else {
            results[name] = CheckResult{Status: "healthy"}
            anyHealthy = true
        }
    }

    status := "healthy"
    if !allHealthy && anyHealthy {
        status = "degraded"
    } else if !anyHealthy {
        status = "unhealthy"
    }

    return HealthStatus{Status: status, Checks: results}
}

// Usage
checker := &HealthChecker{
    checks: map[string]HealthCheck{
        "database": func(ctx context.Context) error {
            return db.PingContext(ctx)
        },
        "redis": func(ctx context.Context) error {
            return redis.Ping(ctx).Err()
        },
        "payment-service": func(ctx context.Context) error {
            _, err := paymentClient.Health(ctx)
            return err
        },
    },
}
```

---

## Combined: Resilience4j-style

```go
type ResilientClient struct {
    circuitBreaker *CircuitBreaker
    bulkhead       *Bulkhead
    rateLimiter    *rate.Limiter
    retryConfig    RetryConfig
    timeout        time.Duration
}

func (c *ResilientClient) Execute(ctx context.Context, fn func() error) error {
    // 1. Rate limiting
    if err := c.rateLimiter.Wait(ctx); err != nil {
        return err
    }

    // 2. Bulkhead
    return c.bulkhead.Execute(ctx, func() error {
        // 3. Circuit breaker
        return c.circuitBreaker.Execute(func() error {
            // 4. Timeout
            ctx, cancel := context.WithTimeout(ctx, c.timeout)
            defer cancel()

            // 5. Retry
            return Retry(ctx, c.retryConfig, func() error {
                return fn()
            })
        })
    })
}
```

---

## См. также

- [Incident Management](../09-devops-sre/06-incident-management.md) — управление инцидентами и реагирование на сбои
- [Failure Detection](../07-distributed-systems/08-failure-detection.md) — обнаружение сбоев в распределённых системах

---

## На интервью

### Типичные вопросы

1. **Circuit Breaker — зачем?**
   - Предотвращение cascade failures
   - Fast fail вместо ожидания timeout
   - Три состояния: closed, open, half-open

2. **Retry strategy?**
   - Exponential backoff с jitter
   - Какие ошибки retryable
   - Max attempts, timeouts

3. **Bulkhead pattern?**
   - Изоляция ресурсов
   - Semaphore vs thread pool
   - Отдельный bulkhead per dependency

4. **Как комбинировать patterns?**
   - Rate limit → Bulkhead → Circuit Breaker → Timeout → Retry
   - Fallback на любом уровне

---

[← Назад к списку тем](README.md)
