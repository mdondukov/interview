# 08. Failure Detection

[← Назад к списку тем](README.md)

---

## Проблема

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Failure Detection Challenge                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Невозможно отличить:                                               │
│  - Crashed node                                                     │
│  - Slow node                                                        │
│  - Network partition                                                │
│                                                                     │
│      Node A                    Node B                               │
│         │                         │                                 │
│         │ ───── Request ──────▶   │                                 │
│         │                         │ Processing...                   │
│         │         ???             │ (or crashed?)                   │
│         │                         │ (or network lost?)              │
│         │ ◀──── Timeout ──────    │                                 │
│         │                         │                                 │
│                                                                     │
│  "In an asynchronous system, you cannot distinguish                 │
│   a slow process from a dead one."                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Heartbeats

### Basic Heartbeat

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Basic Heartbeat                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Node A sends periodic heartbeats to monitor                        │
│  If no heartbeat for T seconds → consider failed                    │
│                                                                     │
│  Node A                         Monitor                             │
│    │                               │                                │
│    │ ──── Heartbeat ────────────▶  │                                │
│    │                               │ Reset timer                    │
│    │                               │                                │
│    │ ──── Heartbeat ────────────▶  │                                │
│    │                               │ Reset timer                    │
│    │                               │                                │
│    │        (crash!)               │                                │
│    ✗                               │                                │
│                                    │ Timeout!                       │
│                                    │ Node A suspected               │
│                                                                     │
│  Trade-off:                                                         │
│  Short timeout → Fast detection, more false positives               │
│  Long timeout → Fewer false positives, slow detection               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type HeartbeatMonitor struct {
    nodes     map[string]time.Time  // node -> last heartbeat
    timeout   time.Duration
    mu        sync.RWMutex
    onFailure func(nodeID string)
}

func (m *HeartbeatMonitor) ReceiveHeartbeat(nodeID string) {
    m.mu.Lock()
    m.nodes[nodeID] = time.Now()
    m.mu.Unlock()
}

func (m *HeartbeatMonitor) CheckNodes() {
    m.mu.RLock()
    defer m.mu.RUnlock()

    for nodeID, lastSeen := range m.nodes {
        if time.Since(lastSeen) > m.timeout {
            m.onFailure(nodeID)
        }
    }
}

func (m *HeartbeatMonitor) Start() {
    ticker := time.NewTicker(m.timeout / 2)
    for range ticker.C {
        m.CheckNodes()
    }
}
```

---

## Phi Accrual Failure Detector

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Phi Accrual Failure Detector                        │
│                    (Hayashibara et al., 2004)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Instead of binary (alive/dead), outputs suspicion level (φ)        │
│                                                                     │
│  φ = -log₁₀(P(node is alive | time since last heartbeat))          │
│                                                                     │
│  φ = 1  → 10% chance of mistake                                     │
│  φ = 2  → 1% chance of mistake                                      │
│  φ = 3  → 0.1% chance of mistake                                    │
│  φ = 8  → 0.00000001% chance of mistake                             │
│                                                                     │
│  Application chooses threshold based on requirements:               │
│  - Low latency: lower φ threshold (faster, more false positives)    │
│  - High accuracy: higher φ threshold (slower, fewer mistakes)       │
│                                                                     │
│  Algorithm:                                                         │
│  1. Track inter-arrival times of heartbeats                         │
│  2. Compute mean (μ) and variance (σ²)                              │
│  3. Model as normal distribution                                    │
│  4. P(next heartbeat > now) from distribution                       │
│  5. φ = -log₁₀(P)                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type PhiAccrualDetector struct {
    // Sliding window of inter-arrival times
    intervals  *SlidingWindow
    lastBeat   time.Time
    threshold  float64
}

func (d *PhiAccrualDetector) Heartbeat() {
    now := time.Now()
    if !d.lastBeat.IsZero() {
        interval := now.Sub(d.lastBeat)
        d.intervals.Add(float64(interval.Milliseconds()))
    }
    d.lastBeat = now
}

func (d *PhiAccrualDetector) Phi() float64 {
    if d.intervals.Count() < 2 {
        return 0
    }

    timeSinceLast := time.Since(d.lastBeat).Milliseconds()
    mean := d.intervals.Mean()
    stdDev := d.intervals.StdDev()

    // P(interval > timeSinceLast) assuming normal distribution
    // Using complementary CDF
    y := float64(timeSinceLast) - mean
    e := math.Exp(-y / stdDev)
    p := 1.0 / (1.0 + e)

    return -math.Log10(p)
}

func (d *PhiAccrualDetector) IsAvailable() bool {
    return d.Phi() < d.threshold
}
```

---

## Gossip Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Gossip Protocol                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Epidemic-style information dissemination                           │
│                                                                     │
│  Round 1:          Round 2:          Round 3:                       │
│    ●───●             ●───●             ●───●                        │
│    │   │             │\ /│             │\X/│                        │
│    │   │   →         │ X │   →         │/X\│                        │
│    │   │             │/ \│             │\ /│                        │
│    ●   ●             ●   ●             ●───●                        │
│                                                                     │
│  Each node periodically:                                            │
│  1. Select random peer                                              │
│  2. Exchange state information                                      │
│  3. Merge states                                                    │
│                                                                     │
│  Properties:                                                        │
│  - Eventually consistent                                            │
│  - Scalable (O(log N) rounds to spread)                             │
│  - Fault tolerant (no single point of failure)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### SWIM Protocol

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SWIM Protocol                                      │
│          (Scalable Weakly-consistent Infection-style                │
│           Process Group Membership Protocol)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Combines:                                                          │
│  - Failure detection (ping/ping-req)                                │
│  - Membership dissemination (gossip)                                │
│                                                                     │
│  Failure Detection:                                                 │
│                                                                     │
│  Node A                  Node B                  Node C             │
│     │                       │                       │               │
│     │ ──── ping ─────────▶  │                       │               │
│     │                       │                       │               │
│     │ ◀──── ack ──────────  │                       │               │
│     │                       │                       │               │
│                                                                     │
│  If no ack:                                                         │
│     │                       │                       │               │
│     │ ── ping-req(B) ───────────────────────────▶   │               │
│     │                       │                       │               │
│     │                       │ ◀──── ping ────────   │               │
│     │                       │                       │               │
│     │                       │ ───── ack ─────────▶  │               │
│     │                       │                       │               │
│     │ ◀──────────────────── ack ────────────────    │               │
│     │                       │                       │               │
│                                                                     │
│  If still no response: B is suspected/failed                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type SWIMNode struct {
    members    map[string]*MemberState
    suspected  map[string]time.Time
    self       string
    pingPeriod time.Duration
}

type MemberState struct {
    Status      string  // alive, suspected, dead
    Incarnation int64
}

func (n *SWIMNode) ProbeRandom() {
    target := n.randomMember()
    if target == "" {
        return
    }

    // Direct ping
    if n.ping(target) {
        return
    }

    // Indirect ping through k random members
    helpers := n.randomMembers(3)
    responses := make(chan bool, len(helpers))

    for _, helper := range helpers {
        go func(h string) {
            responses <- n.pingReq(h, target)
        }(helper)
    }

    // Wait for any success or timeout
    timeout := time.After(n.pingPeriod)
    for i := 0; i < len(helpers); i++ {
        select {
        case success := <-responses:
            if success {
                return
            }
        case <-timeout:
            break
        }
    }

    // No response - suspect the node
    n.suspect(target)
}

func (n *SWIMNode) suspect(nodeID string) {
    n.suspected[nodeID] = time.Now()
    n.members[nodeID].Status = "suspected"

    // Disseminate via gossip
    n.gossip(SuspectMessage{NodeID: nodeID})
}
```

---

## Failure Detection Properties

```
┌─────────────────────────────────────────────────────────────────────┐
│                Failure Detector Properties                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Completeness:                                                      │
│  - Strong: Every failed process is eventually detected by all       │
│  - Weak: Every failed process is eventually detected by some        │
│                                                                     │
│  Accuracy:                                                          │
│  - Strong: No process is suspected before it fails                  │
│  - Weak: Some correct process is never suspected                    │
│                                                                     │
│  Perfect Failure Detector (P):                                      │
│  - Strong completeness + strong accuracy                            │
│  - Impossible in asynchronous systems!                              │
│                                                                     │
│  Eventually Perfect (◇P):                                           │
│  - Eventually stops making mistakes                                 │
│  - After some unknown time T, accurate                              │
│  - Sufficient for consensus (Chandra-Toueg)                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Реальные системы

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Real-World Systems                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cassandra:                                                         │
│  - Phi Accrual Failure Detector                                     │
│  - Default threshold: φ = 8                                         │
│  - Gossip for membership                                            │
│                                                                     │
│  Consul / Serf:                                                     │
│  - SWIM-based (Lifeguard enhancements)                              │
│  - Suspicion with refutation                                        │
│                                                                     │
│  Kubernetes:                                                        │
│  - Node heartbeats to API server                                    │
│  - Configurable timeouts                                            │
│  - Pod liveness/readiness probes                                    │
│                                                                     │
│  ZooKeeper:                                                         │
│  - Session timeouts                                                 │
│  - Ephemeral nodes disappear on failure                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## См. также

- [Coordination](./09-coordination.md) — сервисы координации и service discovery
- [Incident Management](../09-devops-sre/06-incident-management.md) — управление инцидентами и реагирование на сбои

---

## На интервью

### Типичные вопросы

1. **Почему failure detection сложен?**
   - Нельзя отличить slow от failed
   - Асинхронная система
   - Trade-off: speed vs accuracy

2. **Heartbeat — как выбрать timeout?**
   - Network latency
   - Expected load
   - False positive tolerance

3. **Phi Accrual — преимущества?**
   - Adaptive threshold
   - Based on statistics
   - Application chooses accuracy

4. **SWIM protocol — как работает?**
   - Direct + indirect ping
   - Gossip dissemination
   - Suspicion before declaration

5. **Gossip — почему O(log N)?**
   - Each round doubles informed nodes
   - 2^k nodes after k rounds

---

[← Назад к списку тем](README.md)
