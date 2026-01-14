# 10. Payment System

[← Назад к списку тем](README.md)

---

## Требования

### Функциональные
- Process payments (credit card, bank transfer)
- Refunds & chargebacks
- Multi-currency support
- Payment history & receipts
- Subscription/recurring payments
- Fraud detection

### Нефункциональные
- High consistency (no double charges)
- High availability (99.99%)
- PCI DSS compliance
- Idempotency (safe retries)
- Audit trail for all operations
- Low latency (< 3s for payment)

---

## Capacity Estimation

```
Transactions:
- 100M users
- 10M transactions/day average
- Peak: 50M/day (Black Friday, holidays)
- QPS: 10M / 86400 ≈ 115 QPS
- Peak QPS: 600 QPS

Storage:
- Transaction record: ~2KB
- 10M × 2KB × 365 = 7.3TB/year
- Retention: 7 years (regulatory)
- Total: ~50TB

Amount:
- Average transaction: $50
- Daily volume: $500M
- Annual volume: $180B
```

---

## High-Level Design

```
┌──────────┐     ┌─────────────┐     ┌──────────────────┐
│  Client  │────▶│ API Gateway │────▶│  Payment Service │
└──────────┘     └─────────────┘     └────────┬─────────┘
                                              │
          ┌───────────────────────────────────┼───────────────────────────────────┐
          │                                   │                                   │
   ┌──────▼──────┐                     ┌──────▼──────┐                     ┌──────▼──────┐
   │   Fraud     │                     │   Ledger    │                     │   Payment   │
   │   Service   │                     │   Service   │                     │   Router    │
   └─────────────┘                     └──────┬──────┘                     └──────┬──────┘
                                              │                                   │
                                       ┌──────▼──────┐              ┌─────────────┼─────────────┐
                                       │   Ledger    │              │             │             │
                                       │     DB      │        ┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐
                                       │ (PostgreSQL)│        │  Stripe  │  │ PayPal   │  │  Adyen   │
                                       └─────────────┘        └──────────┘  └──────────┘  └──────────┘
```

---

## API Design

### Create Payment
```json
POST /api/v1/payments
X-Idempotency-Key: {uuid}

Request:
{
    "amount": 9999,  // cents
    "currency": "USD",
    "payment_method": {
        "type": "card",
        "token": "tok_xxx"  // from client-side tokenization
    },
    "description": "Order #12345",
    "metadata": {
        "order_id": "order_12345",
        "customer_id": "cust_789"
    }
}

Response:
{
    "payment_id": "pay_abc123",
    "status": "succeeded",
    "amount": 9999,
    "currency": "USD",
    "created_at": "2024-01-15T10:00:00Z",
    "receipt_url": "https://receipts.example.com/pay_abc123"
}
```

### Refund
```json
POST /api/v1/payments/{payment_id}/refunds
X-Idempotency-Key: {uuid}

Request:
{
    "amount": 5000,  // partial refund
    "reason": "customer_request"
}

Response:
{
    "refund_id": "ref_xyz789",
    "payment_id": "pay_abc123",
    "amount": 5000,
    "status": "succeeded"
}
```

### Payment Status
```
GET /api/v1/payments/{payment_id}
GET /api/v1/payments?customer_id={id}&limit=20&cursor={cursor}
```

---

## Data Model

### Ledger (PostgreSQL - Double-Entry Bookkeeping)
```sql
-- Accounts
CREATE TABLE accounts (
    id              UUID PRIMARY KEY,
    account_type    VARCHAR(50),  -- customer, merchant, fee, reserve
    entity_id       UUID,         -- customer_id or merchant_id
    currency        VARCHAR(3),
    balance         BIGINT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Transactions (immutable ledger entries)
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY,
    transaction_id  UUID NOT NULL,
    account_id      UUID REFERENCES accounts(id),
    amount          BIGINT NOT NULL,  -- positive = credit, negative = debit
    balance_after   BIGINT NOT NULL,
    entry_type      VARCHAR(50),  -- payment, refund, fee, transfer
    created_at      TIMESTAMP DEFAULT NOW(),

    INDEX idx_transaction (transaction_id),
    INDEX idx_account_created (account_id, created_at)
);

-- Payments
CREATE TABLE payments (
    id                  UUID PRIMARY KEY,
    idempotency_key     VARCHAR(100) UNIQUE,
    customer_id         UUID NOT NULL,
    merchant_id         UUID NOT NULL,
    amount              BIGINT NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    status              VARCHAR(20),  -- pending, processing, succeeded, failed
    payment_method_type VARCHAR(20),
    external_id         VARCHAR(100),  -- ID from payment provider
    error_code          VARCHAR(50),
    error_message       TEXT,
    metadata            JSONB,
    created_at          TIMESTAMP DEFAULT NOW(),
    updated_at          TIMESTAMP,

    INDEX idx_idempotency (idempotency_key),
    INDEX idx_customer (customer_id),
    INDEX idx_merchant (merchant_id)
);
```

### Idempotency Store (Redis)
```
# Idempotency key → response cache
SET idempotency:{key} {serialized_response} EX 86400

# In-flight request lock
SET idempotency:{key}:lock {request_id} NX EX 30
```

---

## Deep Dive

### 1. Idempotency

```python
class IdempotentPaymentProcessor:
    async def process_payment(self, idempotency_key: str, request: dict):
        # 1. Check if already processed
        cached = await redis.get(f"idempotency:{idempotency_key}")
        if cached:
            return deserialize(cached)

        # 2. Try to acquire lock for in-flight request
        lock_acquired = await redis.set(
            f"idempotency:{idempotency_key}:lock",
            request_id,
            nx=True,
            ex=30
        )

        if not lock_acquired:
            # Another request in progress - wait and retry
            await asyncio.sleep(1)
            return await self.process_payment(idempotency_key, request)

        try:
            # 3. Check database for existing payment
            existing = await db.get_payment_by_idempotency_key(idempotency_key)
            if existing:
                response = self.format_response(existing)
                await redis.set(
                    f"idempotency:{idempotency_key}",
                    serialize(response),
                    ex=86400
                )
                return response

            # 4. Process new payment
            payment = await self._process_new_payment(idempotency_key, request)

            # 5. Cache response
            response = self.format_response(payment)
            await redis.set(
                f"idempotency:{idempotency_key}",
                serialize(response),
                ex=86400
            )

            return response

        finally:
            # Release lock
            await redis.delete(f"idempotency:{idempotency_key}:lock")
```

### 2. Double-Entry Bookkeeping

```
Every financial transaction creates balanced entries:
Debits = Credits

Example: $100 payment from Customer to Merchant (with 2.9% fee)

┌────────────────────────────────────────────────────────────┐
│ Transaction: pay_abc123                                    │
├────────────────────────────────────────────────────────────┤
│ Account              │ Debit   │ Credit  │ Balance After  │
├──────────────────────┼─────────┼─────────┼────────────────┤
│ Customer (liability) │         │ $100.00 │ -$100.00       │
│ Merchant (asset)     │ $97.10  │         │ +$97.10        │
│ Fee (revenue)        │ $2.90   │         │ +$2.90         │
├──────────────────────┼─────────┼─────────┼────────────────┤
│ Total                │ $100.00 │ $100.00 │ Balanced ✓     │
└────────────────────────────────────────────────────────────┘
```

```python
class LedgerService:
    async def record_payment(self, payment: Payment):
        async with db.transaction():
            # Calculate fee
            fee_amount = int(payment.amount * 0.029)  # 2.9%
            merchant_amount = payment.amount - fee_amount

            # Create ledger entries (must balance)
            entries = [
                # Debit customer (they owe us)
                LedgerEntry(
                    account_id=payment.customer_account_id,
                    amount=-payment.amount,  # negative = debit
                    entry_type="payment"
                ),
                # Credit merchant (we owe them)
                LedgerEntry(
                    account_id=payment.merchant_account_id,
                    amount=merchant_amount,
                    entry_type="payment"
                ),
                # Credit fee account
                LedgerEntry(
                    account_id=self.fee_account_id,
                    amount=fee_amount,
                    entry_type="fee"
                )
            ]

            # Verify balance
            total = sum(e.amount for e in entries)
            if total != 0:
                raise UnbalancedTransactionError()

            # Insert all entries atomically
            for entry in entries:
                await self.insert_entry(entry)

            # Update account balances
            for entry in entries:
                await self.update_account_balance(entry.account_id, entry.amount)
```

### 3. Payment State Machine

```
┌─────────────────────────────────────────────────────────────────┐
│                    Payment State Machine                         │
└─────────────────────────────────────────────────────────────────┘

                         ┌───────────┐
                         │  Created  │
                         └─────┬─────┘
                               │
                    ┌──────────▼──────────┐
                    │     Processing      │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
       ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
       │  Succeeded  │  │   Failed    │  │  Requires   │
       │             │  │             │  │   Action    │
       └──────┬──────┘  └─────────────┘  └──────┬──────┘
              │                                 │
              │         (3D Secure)             │
              │                ┌────────────────┘
              │                │
       ┌──────▼──────┐  ┌──────▼──────┐
       │ Refund      │  │  Succeeded  │
       │ (partial/   │  │             │
       │  full)      │  └─────────────┘
       └─────────────┘
```

```python
class PaymentStateMachine:
    TRANSITIONS = {
        'created': ['processing'],
        'processing': ['succeeded', 'failed', 'requires_action'],
        'requires_action': ['processing', 'failed'],
        'succeeded': ['partially_refunded', 'refunded'],
        'partially_refunded': ['refunded'],
        'failed': [],
        'refunded': []
    }

    async def transition(self, payment_id: str, new_status: str, context: dict):
        async with db.transaction():
            payment = await db.get_payment_for_update(payment_id)

            if new_status not in self.TRANSITIONS[payment.status]:
                raise InvalidTransitionError(payment.status, new_status)

            # Record state change
            await db.insert_payment_event(PaymentEvent(
                payment_id=payment_id,
                from_status=payment.status,
                to_status=new_status,
                context=context,
                created_at=datetime.utcnow()
            ))

            # Update payment
            payment.status = new_status
            payment.updated_at = datetime.utcnow()
            await db.update_payment(payment)

            # Trigger side effects
            await self.handle_transition(payment, new_status, context)
```

### 4. Fraud Detection

```python
class FraudDetector:
    async def assess_risk(self, payment: dict) -> RiskAssessment:
        signals = await asyncio.gather(
            self.check_velocity(payment),
            self.check_device_fingerprint(payment),
            self.check_ip_reputation(payment),
            self.check_card_bin(payment),
            self.check_behavior_anomaly(payment),
        )

        risk_score = self.calculate_score(signals)

        return RiskAssessment(
            score=risk_score,
            signals=signals,
            action=self.determine_action(risk_score)
        )

    async def check_velocity(self, payment: dict) -> Signal:
        customer_id = payment['customer_id']
        card_hash = payment['card_fingerprint']

        # Check transaction velocity
        rules = [
            # More than 5 transactions in 1 hour
            await self.count_recent_txns(customer_id, hours=1) > 5,
            # More than $1000 in 24 hours
            await self.sum_recent_amount(customer_id, hours=24) > 100000,
            # Same card used by different customers
            await self.card_used_by_multiple(card_hash, hours=24),
            # First transaction on new card > $500
            await self.is_first_txn(card_hash) and payment['amount'] > 50000,
        ]

        return Signal(
            name='velocity',
            score=sum(rules) * 25,  # Each rule adds 25 points
            details={'rules_triggered': rules}
        )

    def determine_action(self, risk_score: int) -> str:
        if risk_score < 30:
            return 'allow'
        elif risk_score < 70:
            return 'review'  # Manual review
        else:
            return 'block'
```

### 5. Reconciliation

```python
class ReconciliationService:
    """
    Daily reconciliation between internal ledger and payment providers
    """

    async def reconcile_daily(self, date: date):
        # 1. Get our records
        internal_txns = await self.get_internal_transactions(date)

        # 2. Get provider records
        provider_txns = {}
        for provider in ['stripe', 'paypal', 'adyen']:
            provider_txns[provider] = await self.fetch_provider_report(provider, date)

        # 3. Match transactions
        matched, unmatched_internal, unmatched_external = self.match_transactions(
            internal_txns, provider_txns
        )

        # 4. Identify discrepancies
        discrepancies = []
        for match in matched:
            if match.internal.amount != match.external.amount:
                discrepancies.append(AmountMismatch(match))
            if match.internal.status != self.map_status(match.external.status):
                discrepancies.append(StatusMismatch(match))

        # 5. Generate report
        report = ReconciliationReport(
            date=date,
            total_internal=len(internal_txns),
            total_matched=len(matched),
            unmatched_internal=unmatched_internal,
            unmatched_external=unmatched_external,
            discrepancies=discrepancies
        )

        # 6. Alert on issues
        if unmatched_internal or unmatched_external or discrepancies:
            await self.alert_operations(report)

        return report
```

---

## Bottlenecks & Solutions

| Проблема | Решение |
|----------|---------|
| Double charging | Idempotency keys, database transactions |
| Provider downtime | Multiple providers, automatic failover |
| High latency | Async processing, webhook notifications |
| Fraud | ML models, velocity checks, 3D Secure |
| Reconciliation gaps | Daily automated reconciliation |

---

## Trade-offs

### Consistency vs Availability

| Approach | Consistency | Availability | Use Case |
|----------|-------------|--------------|----------|
| Sync processing | Strong | Medium | Critical payments |
| Async + webhooks | Eventual | High | High volume |
| Two-phase | Strong | Lower | Distributed txns |

### Build vs Buy

| Aspect | Build | Buy (Stripe/Adyen) |
|--------|-------|-------------------|
| Time to market | Slow | Fast |
| PCI compliance | Complex | Handled |
| Customization | Full | Limited |
| Cost at scale | Lower | Higher |

---

## На интервью

### Ключевые моменты
1. **Idempotency** — критично для финансовых операций
2. **Double-entry bookkeeping** — баланс всегда сходится
3. **State machine** — чёткие переходы между состояниями
4. **Reconciliation** — ежедневная сверка с провайдерами

### Типичные follow-up
- Как обработать partial failures при распределённых транзакциях?
- Как реализовать escrow (удержание средств)?
- Как масштабировать при пиковых нагрузках (Black Friday)?
- Как обеспечить PCI DSS compliance?
- Как обрабатывать chargebacks?

---

[← Назад к списку тем](README.md)
