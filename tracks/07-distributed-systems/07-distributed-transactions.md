# 07. Distributed Transactions

[← Назад к списку тем](README.md)

---

## Проблема

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Distributed Transaction Problem                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Transfer $100 from Account A to Account B                          │
│  A and B on different databases/services                            │
│                                                                     │
│  ┌──────────────┐         ┌──────────────┐                          │
│  │  Database A  │         │  Database B  │                          │
│  │              │         │              │                          │
│  │  A: $500     │         │  B: $300     │                          │
│  │  A = A - 100 │   ???   │  B = B + 100 │                          │
│  └──────────────┘         └──────────────┘                          │
│                                                                     │
│  What if:                                                           │
│  - A debited but B credit fails?                                    │
│  - Network partition between operations?                            │
│  - One database crashes mid-transaction?                            │
│                                                                     │
│  Need: Atomic Commit across multiple nodes                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Two-Phase Commit (2PC)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Two-Phase Commit                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Coordinator              Participant A       Participant B         │
│       │                        │                    │               │
│       │     PREPARE            │                    │               │
│       ├───────────────────────▶│                    │               │
│       ├────────────────────────┼───────────────────▶│               │
│       │                        │                    │               │
│       │     VOTE YES           │                    │               │
│       │◀───────────────────────┤                    │               │
│       │◀────────────────────────────────────────────┤               │
│       │                        │                    │               │
│       │     COMMIT             │                    │               │
│       ├───────────────────────▶│                    │               │
│       ├────────────────────────┼───────────────────▶│               │
│       │                        │                    │               │
│       │     ACK                │                    │               │
│       │◀───────────────────────┤                    │               │
│       │◀────────────────────────────────────────────┤               │
│                                                                     │
│  Phase 1: Prepare                                                   │
│  - Coordinator asks all participants to prepare                     │
│  - Participants write to log, lock resources                        │
│  - Vote YES (can commit) or NO (abort)                              │
│                                                                     │
│  Phase 2: Commit/Abort                                              │
│  - If all YES: Coordinator sends COMMIT                             │
│  - If any NO: Coordinator sends ABORT                               │
│  - Participants commit/abort and release locks                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2PC Problems

```
┌─────────────────────────────────────────────────────────────────────┐
│                      2PC Problems                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Blocking Protocol:                                                 │
│  - If coordinator fails after PREPARE, participants stuck           │
│  - They voted YES, waiting for COMMIT/ABORT                         │
│  - Locks held indefinitely                                          │
│                                                                     │
│  Coordinator fails:                                                 │
│  ┌─────────────┐       ┌─────────────┐                              │
│  │ Participant │       │ Participant │                              │
│  │   "I voted  │       │   "I voted  │                              │
│  │    YES..."  │       │    YES..."  │                              │
│  │   WAITING   │       │   WAITING   │                              │
│  └─────────────┘       └─────────────┘                              │
│                                                                     │
│  Participant fails after voting YES:                                │
│  - Must recover and finish transaction                              │
│  - Log must survive crash                                           │
│                                                                     │
│  Performance:                                                       │
│  - 2 round trips                                                    │
│  - Locks held during entire protocol                                │
│  - Latency = 2 * network RTT + disk writes                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// Coordinator implementation
type TwoPhaseCommit struct {
    participants []Participant
    log          TransactionLog
}

func (tpc *TwoPhaseCommit) Execute(tx Transaction) error {
    txID := tx.ID

    // Phase 1: Prepare
    tpc.log.Write(txID, "PREPARE")

    votes := make(chan bool, len(tpc.participants))
    for _, p := range tpc.participants {
        go func(p Participant) {
            ok := p.Prepare(txID, tx)
            votes <- ok
        }(p)
    }

    allYes := true
    for i := 0; i < len(tpc.participants); i++ {
        if !<-votes {
            allYes = false
            break
        }
    }

    // Phase 2: Commit or Abort
    if allYes {
        tpc.log.Write(txID, "COMMIT")
        for _, p := range tpc.participants {
            p.Commit(txID)
        }
    } else {
        tpc.log.Write(txID, "ABORT")
        for _, p := range tpc.participants {
            p.Abort(txID)
        }
    }

    return nil
}
```

---

## Three-Phase Commit (3PC)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Three-Phase Commit                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Adds PRE-COMMIT phase to avoid blocking                            │
│                                                                     │
│  Phase 1: Can Commit?                                               │
│  Coordinator → Participants: "Can you commit?"                      │
│  Participants → Coordinator: YES/NO                                 │
│                                                                     │
│  Phase 2: Pre-Commit                                                │
│  If all YES:                                                        │
│    Coordinator → Participants: "Prepare to commit"                  │
│    Participants: Write to log, ACK                                  │
│                                                                     │
│  Phase 3: Do Commit                                                 │
│  Coordinator → Participants: "Commit!"                              │
│  Participants: Commit and ACK                                       │
│                                                                     │
│  Recovery:                                                          │
│  - If participant in PRE-COMMIT and timeout → can COMMIT            │
│  - If participant in CAN-COMMIT and timeout → ABORT                 │
│  - Non-blocking in async model (with assumptions)                   │
│                                                                     │
│  Still has issues:                                                  │
│  - Network partitions can cause inconsistency                       │
│  - Rarely used in practice                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Saga Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Saga Pattern                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Long-lived transaction as sequence of local transactions           │
│  Each has a compensating transaction for rollback                   │
│                                                                     │
│  T1 ─────▶ T2 ─────▶ T3 ─────▶ ... ─────▶ Tn                        │
│   │         │         │                    │                        │
│   ▼         ▼         ▼                    ▼                        │
│  C1        C2        C3                   Cn  (compensations)       │
│                                                                     │
│  On failure at T3:                                                  │
│  T1 → T2 → T3(fail) → C2 → C1                                       │
│                                                                     │
│  Example: Book Trip Saga                                            │
│  T1: Reserve flight                                                 │
│  T2: Reserve hotel                                                  │
│  T3: Reserve car                                                    │
│  T4: Charge payment                                                 │
│                                                                     │
│  If T3 fails:                                                       │
│  C2: Cancel hotel                                                   │
│  C1: Cancel flight                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Choreography vs Orchestration

```
┌─────────────────────────────────────────────────────────────────────┐
│                Choreography-based Saga                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Services communicate via events                                    │
│  No central coordinator                                             │
│                                                                     │
│  Order      Payment       Inventory      Shipping                   │
│  Service    Service       Service        Service                    │
│     │           │             │              │                      │
│     │ OrderCreated           │              │                       │
│     ├──────────▶│             │              │                      │
│     │           │ PaymentCompleted          │                       │
│     │           ├────────────▶│              │                      │
│     │           │             │ StockReserved│                      │
│     │           │             ├─────────────▶│                      │
│     │           │             │              │ ShipmentCreated      │
│     │◀──────────┼─────────────┼──────────────┤                      │
│                                                                     │
│  ✅ Loose coupling                                                  │
│  ❌ Hard to track flow                                              │
│  ❌ Cyclic dependencies risk                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                 Orchestration-based Saga                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Central orchestrator coordinates steps                             │
│                                                                     │
│                    ┌─────────────┐                                  │
│                    │ Orchestrator│                                  │
│                    └──────┬──────┘                                  │
│              ┌────────────┼────────────┐                            │
│              │            │            │                            │
│              ▼            ▼            ▼                            │
│         ┌────────┐   ┌────────┐   ┌────────┐                        │
│         │Payment │   │Inventory│  │Shipping│                        │
│         └────────┘   └────────┘   └────────┘                        │
│                                                                     │
│  ✅ Clear flow, easy to track                                       │
│  ✅ Easier compensation                                             │
│  ❌ Single point of coordination                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// Saga Orchestrator
type BookTripSaga struct {
    steps []SagaStep
}

type SagaStep struct {
    Execute    func(ctx context.Context) error
    Compensate func(ctx context.Context) error
}

func (s *BookTripSaga) Run(ctx context.Context) error {
    completedSteps := []int{}

    for i, step := range s.steps {
        if err := step.Execute(ctx); err != nil {
            // Compensate in reverse order
            for j := len(completedSteps) - 1; j >= 0; j-- {
                s.steps[completedSteps[j]].Compensate(ctx)
            }
            return err
        }
        completedSteps = append(completedSteps, i)
    }

    return nil
}

// Usage
saga := &BookTripSaga{
    steps: []SagaStep{
        {
            Execute:    reserveFlight,
            Compensate: cancelFlight,
        },
        {
            Execute:    reserveHotel,
            Compensate: cancelHotel,
        },
        {
            Execute:    chargePayment,
            Compensate: refundPayment,
        },
    },
}

saga.Run(ctx)
```

---

## Comparison

```
┌──────────────────┬─────────────────┬─────────────────┬───────────────┐
│                  │      2PC        │      Saga       │   TCC         │
├──────────────────┼─────────────────┼─────────────────┼───────────────┤
│ Atomicity        │ Strong          │ Eventual        │ Eventual      │
│ Isolation        │ Yes (locks)     │ No              │ No            │
│ Consistency      │ Strong          │ Eventual        │ Eventual      │
│ Performance      │ Slow (blocking) │ Fast            │ Medium        │
│ Availability     │ Low             │ High            │ High          │
│ Complexity       │ Medium          │ Medium          │ High          │
│ Rollback         │ Automatic       │ Compensations   │ Cancel phase  │
└──────────────────┴─────────────────┴─────────────────┴───────────────┘

TCC = Try-Confirm-Cancel
Similar to Saga but with explicit reserve/confirm phases
```

---

## Best Practices

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Best Practices                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Avoid distributed transactions if possible                      │
│     - Design services to be self-contained                          │
│     - Use eventual consistency                                      │
│                                                                     │
│  2. Make operations idempotent                                      │
│     - Safe to retry                                                 │
│     - Use idempotency keys                                          │
│                                                                     │
│  3. Design compensations carefully                                  │
│     - Must handle partial states                                    │
│     - May need to be retried                                        │
│                                                                     │
│  4. Use outbox pattern for reliability                              │
│     - Store event with local transaction                            │
│     - Publish asynchronously                                        │
│                                                                     │
│  5. Monitor and alert on stuck transactions                         │
│     - Timeouts                                                      │
│     - Dead letter queues                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **2PC — как работает и проблемы?**
   - Prepare → Commit/Abort
   - Blocking if coordinator fails
   - Locks held during protocol

2. **Saga vs 2PC?**
   - Saga: eventual consistency, compensations
   - 2PC: strong consistency, blocking
   - Saga for microservices, 2PC for databases

3. **Choreography vs Orchestration?**
   - Choreography: event-driven, loose coupling
   - Orchestration: central control, easier to track

4. **Как обрабатывать failures в Saga?**
   - Compensating transactions
   - Idempotent operations
   - Retry with exponential backoff

---

[← Назад к списку тем](README.md)
