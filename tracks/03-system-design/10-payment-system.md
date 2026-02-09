# 10. Payment System

Развёрнутые вопросы и ответы про проектирование платёжных систем: идемпотентность, двойная запись, выверка, соответствие PCI, обработка отказов, возвраты и масштабирование. Материал построен в формате вопрос-ответ с единой структурой для подготовки к интервью.

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [09-video-streaming](./09-video-streaming.md) · Следующая тема: [11-booking-system](./11-booking-system.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Idempotency Key** — это уникальный идентификатор запроса, который клиент отправляет вместе с платежом (например, UUID). Сервер использует этот ключ для обнаружения дублирующихся запросов: если приходит второй запрос с тем же ключом, сервер знает, что это повтор, и возвращает результат первой обработки вместо создания нового платежа. Idempotency Key — главное оружие против двойных платежей.

**Double Payment** — это наихудший сценарий в платёжных системах, когда один платёж обработан дважды из-за сетевого сбоя, retry клиента или ошибки в коде. Например, клиент отправляет платёж, сервер обрабатывает его, но ответ теряется в сети, и клиент повторяет запрос. Если система не защищена, платёж будет снят дважды. Double payment ведёт к гневу пользователей, судам и репутационному ущербу, и предотвращается только через idempotency.

**Idempotent Operation** — это операция, которая дает одинаковый результат при повторном выполнении, как и при первом. Например, запись в базу данных с уникальным ключом идемпотентна: второй запрос с тем же ключом просто обновит существующую запись или вернёт старый результат. Для платежей идемпотентность критична: повтор запроса платежа не должен создавать новый платёж.

**Double-Entry Bookkeeping** — это классическая система финансовой учёта, в которой каждая операция записывается дважды: один раз как дебет (из) и один раз как кредит (в). Например, когда деньги переводятся от пользователя A к компании, записывается "-$100 для A" и "+$100 для компании". Это гарантирует консистентность: сумма всех дебетов всегда равна сумме всех кредитов, и любая ошибка немедленно обнаруживается.

**Ledger** — это реестр (журнал) всех финансовых транзакций системы, содержащий детали каждого платежа: ID, сумму, кто платит, кому платит, статус (pending/success/failed), время. Ledger — это "истина источник" для всех финансовых вопросов и используется для аудита, разрешения оспариваемых платежей, финансовых отчётов и соответствия нормативам.

**Settlement** — это процесс окончательного перемещения денег от платёжного провайдера (например, Stripe, PayPal) на банковский счёт вашей компании. Settlement может занимать 1-3 дня: в день 1 вы обрабатываете платежи, в день 2-3 деньги фактически поступают на счёт. Между платежом и settlement нужна ежедневная сверка, чтобы убедиться, что всё согласовано.

**Reconciliation** — это ежедневная сверка платежей, обработанных вашей системой, с данными, полученными от платёжного провайдера. Например, ваша система зафиксировала 1000 успешных платежей на $100,000, но провайдер отправил данные о 999 платежах на $99,500 (один платёж был откажен или возвращен). Reconciliation обнаруживает такие расхождения и помогает найти ошибки, chargebacks или мошенничество.

**PCI DSS (Payment Card Industry Data Security Standard)** — это набор правил и стандартов безопасности для компаний, обрабатывающих платежи кредитными/дебетовыми картами. PCI требует никогда не хранить полные номера карт, использовать шифрование, проводить регулярные аудиты безопасности и многое другое. Несоответствие PCI может привести к штрафам в сотни тысяч долларов и запрету обработки платежей.

**Tokenization** — это замена чувствительных данных карты (номер, CVV) на безопасный токен (строка случайных символов). Например, вместо хранения полного номера карты "4532-1234-5678-9010", система хранит токен "tok_3fe4a2c1". Tokenization позволяет компании обрабатывать платежи без хранения полного номера карты на своих серверах, что значительно облегчает соответствие PCI и снижает риск взлома.

**Webhook** — это асинхронный обратный вызов (callback) от платёжного провайдера к вашей системе, уведомляющий о результате платежа. Вместо того чтобы ваша система постоянно опрашивать провайдера ("Готов ли платёж?"), провайдер сам отправляет вам HTTP POST с новостью: "Платёж успешно обработан". Webhook критичен для асинхронной обработки и позволяет системе оставаться в синхронизации с провайдером.

**Chargeback** — это возврат платежа, инициированный самим клиентом через его банк, обычно из-за мошенничества ("Я этого не покупал"), несанкционированного платежа или просто недовольства покупкой. Chargeback — худший сценарий для продавца: деньги возвращаются, вы теряете товар или услугу, и платёжный провайдер берёт комиссию за chargeback ($15-100). Требуется логирование всех деталей платежа для доказательства при оспаривании.

**Exponential Backoff** — это стратегия повтора запроса с растущей задержкой при сетевых ошибках: первая попытка сразу, при ошибке ждёте 1 сек, вторая попытка, если ошибка ждёте 2 сек, третья попытка, если ошибка ждёте 4 сек, и т.д. (1s → 2s → 4s → 8s → 16s...). Exponential backoff предотвращает перегрузку системы при проблемах сети, позволяет удалённому серверу восстановиться и существенно повышает надёжность.

**Transaction** — это атомная операция в базе данных, которая либо полностью выполнится, либо полностью откатится при ошибке. Например, при платеже нужно одновременно: (1) списать деньги со счета пользователя, (2) пополнить счёт компании, (3) обновить статус платежа в ledger. Если что-то пойдёт не так на шаге 2, вся транзакция откатывается и ничего не меняется. Транзакции гарантируют консистентность данных в критических операциях.

---

## Вопросы и разборы

### 1. Как спроектировать платёжную систему, обеспечивающую никогда не повторить платёж (идемпотентность)?

**Зачем спрашивают.** Идемпотентность — основа надёжных платёжных систем. Двойной платёж — катастрофа. Интервьюер проверяет понимание того, как избежать гонок и переобработки при сетевых сбоях.

**Короткий ответ.** Используй уникальный `idempotency_key` от клиента. Проверяй его в Redis (быстро) и в БД (надёжно). Обрабатывай три случая: обработан ранее (вернуть результат), обрабатывается сейчас (ждать), новый (обработать). Результат кэшируй с TTL.

**Детальный разбор.**

**Сценарии, требующие идемпотентности:**
```
1. Сетевой таймаут после обработки:
   Client → Payment Service → обработана → response теряется → timeout
   Client повторяет → опасность двойного платежа!

2. Retry клиента:
   Client: "Обработай платёж, key=uuid1"
   Service: обрабатывает, но ответ теряется
   Client: "Обработай платёж, key=uuid1" (again)
   Service: должен вернуть тот же результат

3. Webhook от провайдера дважды:
   Stripe: "Payment succeeded, id=pay_123"
   Network blip → webhook отправляется дважды
   Должны быть идемпотентны
```

**Архитектура идемпотентности:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     Payment Request                             │
│              idempotency_key: "uuid-12345"                      │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │  Check Redis Cache (fast)    │
        │  key: idempotency:uuid-12345 │
        └───┬──────────────────────────┘
            │
      ┌─────┴──────────┬──────────────┐
      │                │              │
   Found          Not Found      Locked
   (cached)       (in-flight)    (wait)
      │                │              │
      │                ▼              │
      │         Try acquire lock      │
      │      idempotency:uuid:lock    │
      │                │              │
      │           ┌────┴────┐         │
      │       Acquired   Exists       │
      │           │         │         │
      │           ▼         ▼         │
      │    Check DB    Sleep & retry  │
      │                 go back       │
      │           │                   │
      │      ┌────┴────┐              │
      │    Found    Not found         │
      │      │         │              │
      │      │         ▼              │
      │      │    Process payment     │
      │      │    (external API)      │
      │      │         │              │
      │      │         ▼              │
      │      │    Save to DB          │
      │      │         │              │
      │      └────┬────┘              │
      │           │                   │
      └───────────┼───────────────────┘
                  │
                  ▼
         Cache result in Redis
             (TTL: 24h)
                  │
                  ▼
         Return response to client
```

**Пример кода (Python):**
```python
class IdempotentPaymentProcessor:
    def __init__(self, redis_client, db_client):
        self.redis = redis_client
        self.db = db_client
        self.stripe = stripe_client
        
    async def process_payment(self, idempotency_key: str, payment_request: dict):
        """
        Обработай платёж идемпотентно.
        """
        # 1. Проверяем кэш (быстро)
        cached_response = await self.redis.get(f"idempotency:{idempotency_key}")
        if cached_response:
            log.info(f"Cache hit: {idempotency_key}")
            return json.loads(cached_response)
        
        # 2. Пытаемся захватить блокировку (защита от одновременной обработки)
        lock_key = f"idempotency:{idempotency_key}:lock"
        lock_acquired = await self.redis.set(
            lock_key,
            str(uuid4()),
            nx=True,
            ex=30  # таймаут 30 сек (должна завершиться к тому времени)
        )
        
        if not lock_acquired:
            # Другой процесс обрабатывает → ждём и повторяем
            log.info(f"Lock not acquired, waiting: {idempotency_key}")
            await asyncio.sleep(1)
            return await self.process_payment(idempotency_key, payment_request)
        
        try:
            # 3. Проверяем БД (должна быть истина)
            existing_payment = await self.db.get_payment_by_idempotency_key(idempotency_key)
            if existing_payment:
                response = self._format_response(existing_payment)
                # Кэшируем найденный результат
                await self.redis.set(
                    f"idempotency:{idempotency_key}",
                    json.dumps(response),
                    ex=86400
                )
                return response
            
            # 4. Новый платёж → обрабатываем
            payment = await self._process_new_payment(idempotency_key, payment_request)
            
            # 5. Кэшируем результат
            response = self._format_response(payment)
            await self.redis.set(
                f"idempotency:{idempotency_key}",
                json.dumps(response),
                ex=86400  # кэшируем на 24 часа
            )
            
            return response
        
        finally:
            # 6. Освобождаем блокировку
            await self.redis.delete(lock_key)
    
    async def _process_new_payment(self, idempotency_key: str, request: dict):
        """
        Обработай платёж во внешнем провайдере и сохрани локально.
        """
        async with self.db.transaction():
            # Сохраняем платёж до обработки (статус: pending)
            payment = Payment(
                id=str(uuid4()),
                idempotency_key=idempotency_key,
                customer_id=request['customer_id'],
                amount=request['amount'],
                currency=request['currency'],
                status='pending',
                created_at=datetime.utcnow()
            )
            await self.db.insert_payment(payment)
            
            try:
                # Отправляем в Stripe, PayPal и т.д.
                external_response = await self.stripe.charge(
                    amount=request['amount'],
                    currency=request['currency'],
                    token=request['payment_method_token']
                )
                
                # Обновляем статус
                payment.status = 'succeeded'
                payment.external_id = external_response['id']
                payment.updated_at = datetime.utcnow()
                await self.db.update_payment(payment)
                
                return payment
            
            except Exception as e:
                # Даже при ошибке сохраняем попытку (для отладки)
                payment.status = 'failed'
                payment.error_message = str(e)
                payment.updated_at = datetime.utcnow()
                await self.db.update_payment(payment)
                raise
```

**SQL Schema (PostgreSQL):**
```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key VARCHAR(100) UNIQUE NOT NULL,
    customer_id UUID NOT NULL,
    amount BIGINT NOT NULL,  -- в центах
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,  -- pending, succeeded, failed
    external_id VARCHAR(255),  -- ID от Stripe/PayPal
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    
    INDEX idx_idempotency (idempotency_key),
    INDEX idx_customer_created (customer_id, created_at)
);
```

**Типичные ошибки.**
- Полагаться только на Redis кэш → при рестарте потеряется, БД должна быть источником истины.
- Коротким TTL на кэш → новые запросы с тем же ключом обработаны заново (потери данных).
- Не обработать race condition между `GET` и `SET` → используй `SET ... NX` (atomic).
- Не освободить блокировку в finally → горутина заблокирует другие попытки навеки.
- Игнорировать результаты повторной обработки → если всё было успешно, повторный вызов может вернуть разный статус.

**На интервью.**
- Объясни три фазы: кэш → блокировка → БД.
- Упомяни, почему Redis быстрее БД для проверки (in-memory).
- Уточняющий вопрос: «Как обработать просроченный idempotency key?» → Установить TTL на запись в БД (7 дней для соответствия PCI), старые ключи не обрабатываются повторно.
- Уточняющий вопрос: «Что если Redis недоступен?» → Fallback на БД только (медленнее, но надёжно).

---

### 2. Как спроектировать бухгалтерский реестр (ledger) с двойной записью для платёжной системы?

**Зачем спрашивают.** Двойная запись — фундамент финансовых систем. Баланс всегда должен сходиться. Это проверяет понимание финансовой логики и консистентности данных.

**Короткий ответ.** Каждый платёж создаёт записи в реестре: дебет на счёте отправителя, кредит на счёте получателя, кредит на счёте комиссий. Сумма дебетов всегда равна сумме кредитов. Использование транзакций БД гарантирует, что все записи созданы или ничего.

**Детальный разбор.**

**Основы двойной записи:**
```
Баланс = ∑(Кредиты) − ∑(Дебеты)

Когда клиент платит 100 USD за заказ:

┌─────────────────────────────────────────────────────┐
│ Платёж: pay_12345                                   │
├─────────────────────────────────────────────────────┤
│ Счёт                      Дебет    Кредит   Баланс  │
├─────────────────────────────────────────────────────┤
│ Клиент (Liability)                  100    −100     │
│ Продавец (Asset)          97                +97     │
│ Комиссия (Revenue)        3                 +3      │
├─────────────────────────────────────────────────────┤
│ Итого                     100      100       ✓      │
└─────────────────────────────────────────────────────┘

Баланс клиента: -100 (должен компании)
Баланс продавца: +97 (компания должна ему)
Баланс комиссии: +3 (доход компании)

Проверка: дебеты (100) = кредиты (97 + 3) ✓
```

**Проектирование схемы:**
```sql
-- Счета (accounts): каждая учётная запись в системе
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_type VARCHAR(50) NOT NULL,  -- customer, merchant, fee_reserve, settlement
    entity_id UUID NOT NULL,  -- customer_id или merchant_id
    currency VARCHAR(3) NOT NULL,
    balance BIGINT DEFAULT 0,  -- в центах
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(account_type, entity_id, currency)
);

-- Реестр (ledger_entries): неизменяемые записи всех операций
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL,  -- связь с платежом или возвратом
    account_id UUID NOT NULL REFERENCES accounts(id),
    amount BIGINT NOT NULL,  -- положительно = кредит, отрицательно = дебет
    balance_after BIGINT NOT NULL,  -- баланс счета после этой записи
    entry_type VARCHAR(50) NOT NULL,  -- payment, refund, fee, transfer, chargeback
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    INDEX idx_transaction (transaction_id),
    INDEX idx_account_created (account_id, created_at)
);

-- Платежи (для справки)
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    merchant_id UUID NOT NULL,
    amount BIGINT NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_customer (customer_id),
    INDEX idx_merchant (merchant_id)
);
```

**Пример обработки платежа (Python с PostgreSQL):**
```python
class LedgerService:
    def __init__(self, db_connection):
        self.db = db_connection
    
    async def record_payment(self, payment: Payment):
        """
        Запиши платёж в реестр с двойной записью.
        Гарантия: либо все записи созданы, либо ничего.
        """
        async with self.db.transaction() as txn:
            # Вычислим комиссию (2.9% + $0.30)
            fee_cents = int(payment.amount * 0.029 + 30)
            merchant_amount = payment.amount - fee_cents
            
            # Получим ID счетов
            customer_account = await self.db.get_account(
                account_type='customer',
                entity_id=payment.customer_id,
                currency=payment.currency
            )
            
            merchant_account = await self.db.get_account(
                account_type='merchant',
                entity_id=payment.merchant_id,
                currency=payment.currency
            )
            
            fee_account = await self.db.get_account(
                account_type='fee_reserve',
                entity_id=None,  # общий счет комиссий
                currency=payment.currency
            )
            
            # Создаём записи в реестре
            entries = [
                # Дебет со счета клиента (он платит)
                LedgerEntry(
                    transaction_id=payment.id,
                    account_id=customer_account.id,
                    amount=-payment.amount,  # минус = дебет
                    entry_type='payment',
                    description=f'Payment to merchant {payment.merchant_id}'
                ),
                # Кредит на счет продавца (он получает минус комиссия)
                LedgerEntry(
                    transaction_id=payment.id,
                    account_id=merchant_account.id,
                    amount=merchant_amount,  # плюс = кредит
                    entry_type='payment',
                    description=f'Payment from customer {payment.customer_id}'
                ),
                # Кредит на счет комиссий (доход компании)
                LedgerEntry(
                    transaction_id=payment.id,
                    account_id=fee_account.id,
                    amount=fee_cents,  # плюс = кредит
                    entry_type='fee',
                    description=f'Fee from payment {payment.id}'
                )
            ]
            
            # Проверяем баланс: дебеты = кредиты
            total_debits = sum(e.amount for e in entries if e.amount < 0)
            total_credits = sum(e.amount for e in entries if e.amount > 0)
            
            if abs(total_debits) != total_credits:
                raise ValueError(
                    f"Unbalanced transaction: debits={total_debits}, credits={total_credits}"
                )
            
            # Вставляем записи и обновляем балансы атомарно
            for entry in entries:
                # Вставляем запись в реестр
                new_balance = await self._insert_ledger_entry(entry)
                
                # Обновляем баланс счета
                await self.db.execute("""
                    UPDATE accounts 
                    SET balance = balance + %s
                    WHERE id = %s
                """, (entry.amount, entry.account_id))
            
            # Обновляем платёж
            await self.db.execute("""
                UPDATE payments 
                SET status = 'succeeded', updated_at = NOW()
                WHERE id = %s
            """, (payment.id,))
            
            return payment
    
    async def record_refund(self, payment_id: str, refund_amount: int):
        """
        Запиши возврат (обратный платёж).
        """
        async with self.db.transaction() as txn:
            payment = await self.db.get_payment(payment_id)
            
            # Возврат: инвертируем направления платежа
            # Комиссия за возврат обычно меньше или нулевая
            refund_fee_cents = 0
            customer_refund = refund_amount
            
            customer_account = await self.db.get_account(
                account_type='customer',
                entity_id=payment.customer_id,
                currency=payment.currency
            )
            
            merchant_account = await self.db.get_account(
                account_type='merchant',
                entity_id=payment.merchant_id,
                currency=payment.currency
            )
            
            entries = [
                # Кредит клиенту (ему возвращаем)
                LedgerEntry(
                    transaction_id=payment_id,
                    account_id=customer_account.id,
                    amount=customer_refund,
                    entry_type='refund'
                ),
                # Дебет со счета продавца (у него забираем)
                LedgerEntry(
                    transaction_id=payment_id,
                    account_id=merchant_account.id,
                    amount=-refund_amount,
                    entry_type='refund'
                )
            ]
            
            # Проверяем баланс
            total_debits = sum(e.amount for e in entries if e.amount < 0)
            total_credits = sum(e.amount for e in entries if e.amount > 0)
            assert abs(total_debits) == total_credits
            
            # Вставляем записи
            for entry in entries:
                await self._insert_ledger_entry(entry)
                await self.db.execute("""
                    UPDATE accounts 
                    SET balance = balance + %s
                    WHERE id = %s
                """, (entry.amount, entry.account_id))
            
            return True
```

**Верификация баланса:**
```python
async def verify_ledger_balance(self, account_id: str):
    """
    Проверь, что баланс счета согласуется с записями в реестре.
    """
    # Текущий баланс в таблице accounts
    account = await self.db.get_account(account_id)
    current_balance = account.balance
    
    # Пересчитанный баланс из реестра
    entries = await self.db.query("""
        SELECT amount FROM ledger_entries
        WHERE account_id = %s
        ORDER BY created_at
    """, (account_id,))
    
    calculated_balance = sum(e['amount'] for e in entries)
    
    if current_balance != calculated_balance:
        raise BalanceMismatchError(
            f"Account {account_id}: recorded={current_balance}, "
            f"calculated={calculated_balance}"
        )
    
    return True
```

**Типичные ошибки.**
- Не обрабатывать комиссии → баланс не сходится.
- Обновлять баланс после вставки записи, а не транзакционно → race condition.
- Не проверять баланс перед вставкой → ошибки остаются незамеченными.
- Использовать float для денег → ошибки округления (всегда целые центы).
- Не иметь истории реестра → невозможно провести аудит.

**На интервью.**
- Объясни, почему дебеты всегда должны равняться кредитам.
- Покажи, как одна ошибка в кодировании может разбалансировать систему.
- Уточняющий вопрос: «Как обнаружить discrepancy между реестром и платежами?» → ежедневная сверка (reconciliation).
- Уточняющий вопрос: «Как обрабатывать несколько валют?» → отдельные счета и реестры для каждой валюты.

---

### 3. Как реализовать возвраты (refunds) с частичным возмещением и обработкой ошибок?

**Зачем спрашивают.** Возвраты сложнее, чем может показаться. Частичные возвраты, несколько возвратов, обработка ошибок после частичного возврата. Это проверяет практическое мышление о граничных случаях.

**Короткий ответ.** Отслеживай состояние возврата: pending, succeeded, failed. Проверяй, что сумма возврата ≤ оставшегося возвратимого платежа. Используй идемпотентность для возвратов (как и для платежей). Храни историю всех попыток возврата для аудита.

**Детальный разбор.**

**Модель состояний для платежа и возвратов:**
```
Платёж: 100 USD

┌─────────────┐
│  Succeeded  │
└──────┬──────┘
       │
   Refund 1: 30 USD ──► pending → succeeded ✓
       │ (refundable remaining: 70)
       │
   Refund 2: 50 USD ──► pending → succeeded ✓
       │ (refundable remaining: 20)
       │
   Refund 3: 25 USD ──► pending → FAILED (exceeds 20 available)
       │
       └─► status: partially_refunded
           refunded_amount: 80
           remaining: 20
```

**Архитектура обработки возврата:**
```
POST /api/v1/payments/{id}/refunds
{
  "amount": 5000,
  "reason": "customer_request",
  "idempotency_key": "uuid-xyz"
}
       │
       ▼
┌──────────────────────────────┐
│  Check idempotency key       │
│  (same as with payments)     │
└───┬──────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│  Get payment + lock          │
│  Verify status = succeeded   │
└───┬──────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│  Calculate refundable        │
│  = amount − refunded         │
└───┬──────────────────────────┘
    │
    ┌─────────────────┬──────────────┐
    │                 │              │
  Valid           Amount too     Already
  refund          large          refunded
    │                 │              │
    ▼                 ▼              ▼
Process         Return error    Return previous
to provider                     result
    │
    ▼
┌──────────────────────────────┐
│  Update ledger               │
│  (refund as reverse payment) │
└──────────────────────────────┘
    │
    ▼
Return refund_id + status
```

**SQL Schema:**
```sql
CREATE TABLE refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key VARCHAR(100) UNIQUE NOT NULL,
    payment_id UUID NOT NULL REFERENCES payments(id),
    amount BIGINT NOT NULL,  -- в центах
    reason VARCHAR(50) NOT NULL,  -- customer_request, duplicate, fraud, etc.
    status VARCHAR(20) NOT NULL,  -- pending, succeeded, failed
    external_id VARCHAR(255),  -- ID от Stripe если они обрабатывают
    error_code VARCHAR(50),
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    
    FOREIGN KEY (payment_id) REFERENCES payments(id),
    INDEX idx_idempotency (idempotency_key),
    INDEX idx_payment (payment_id)
);
```

**Обработка возврата (Python):**
```python
class RefundService:
    def __init__(self, ledger_service, stripe_client, db):
        self.ledger = ledger_service
        self.stripe = stripe_client
        self.db = db
    
    async def create_refund(
        self, 
        payment_id: str, 
        amount: int,
        reason: str,
        idempotency_key: str
    ) -> Refund:
        """
        Создай возврат с полной идемпотентностью и проверками.
        """
        # 1. Проверка идемпотентности (как для платежей)
        cached = await self.redis.get(f"refund_idempotency:{idempotency_key}")
        if cached:
            return json.loads(cached)
        
        # 2. Захват блокировки
        lock_key = f"refund_idempotency:{idempotency_key}:lock"
        if not await self.redis.set(lock_key, "1", nx=True, ex=30):
            await asyncio.sleep(1)
            return await self.create_refund(payment_id, amount, reason, idempotency_key)
        
        try:
            async with self.db.transaction():
                # 3. Получаем платёж (с блокировкой для записи)
                payment = await self.db.get_payment_for_update(payment_id)
                
                if payment.status != 'succeeded':
                    raise InvalidPaymentStatusError(
                        f"Cannot refund {payment.status} payment"
                    )
                
                # 4. Проверяем, была ли попытка возврата с этим ключом
                existing_refund = await self.db.query("""
                    SELECT * FROM refunds 
                    WHERE idempotency_key = %s
                """, (idempotency_key,))
                
                if existing_refund:
                    response = self._format_refund_response(existing_refund[0])
                    await self.redis.set(
                        f"refund_idempotency:{idempotency_key}",
                        json.dumps(response),
                        ex=86400
                    )
                    return response
                
                # 5. Вычисляем возвратимую сумму
                total_refunded = await self.db.query("""
                    SELECT SUM(amount) as total FROM refunds
                    WHERE payment_id = %s AND status = 'succeeded'
                """, (payment_id,))
                
                total_refunded_so_far = total_refunded[0]['total'] or 0
                refundable_amount = payment.amount - total_refunded_so_far
                
                # 6. Проверяем, что возврат не превышает доступную сумму
                if amount > refundable_amount:
                    raise InsufficientRefundableAmountError(
                        f"Requested: {amount}, available: {refundable_amount}"
                    )
                
                # 7. Создаём запись возврата в БД
                refund = Refund(
                    id=str(uuid4()),
                    idempotency_key=idempotency_key,
                    payment_id=payment_id,
                    amount=amount,
                    reason=reason,
                    status='pending',
                    created_at=datetime.utcnow()
                )
                
                await self.db.insert_refund(refund)
                
                # 8. Отправляем в Stripe
                try:
                    external_refund = await self.stripe.refund(
                        payment_id=payment.external_id,
                        amount=amount,
                        reason=reason,
                        idempotency_key=idempotency_key  # у Stripe тоже есть
                    )
                    
                    refund.status = 'succeeded'
                    refund.external_id = external_refund['id']
                    refund.updated_at = datetime.utcnow()
                    
                except Exception as e:
                    refund.status = 'failed'
                    refund.error_code = str(e.__class__.__name__)
                    refund.error_message = str(e)
                    refund.updated_at = datetime.utcnow()
                    raise
                
                finally:
                    await self.db.update_refund(refund)
                
                # 9. Обновляем реестр (только если succeeded)
                if refund.status == 'succeeded':
                    await self.ledger.record_refund(payment_id, amount)
                
                # 10. Обновляем статус платежа
                if total_refunded_so_far + amount == payment.amount:
                    # Полный возврат
                    await self.db.execute("""
                        UPDATE payments SET status = 'refunded' WHERE id = %s
                    """, (payment_id,))
                else:
                    # Частичный возврат
                    await self.db.execute("""
                        UPDATE payments SET status = 'partially_refunded' WHERE id = %s
                    """, (payment_id,))
                
                # 11. Кэшируем результат
                response = self._format_refund_response(refund)
                await self.redis.set(
                    f"refund_idempotency:{idempotency_key}",
                    json.dumps(response),
                    ex=86400
                )
                
                return response
        
        finally:
            await self.redis.delete(lock_key)
    
    async def get_refund(self, refund_id: str) -> Refund:
        """Получи статус возврата."""
        return await self.db.get_refund(refund_id)
    
    async def list_refunds(self, payment_id: str) -> List[Refund]:
        """Список всех возвратов для платежа."""
        return await self.db.query("""
            SELECT * FROM refunds 
            WHERE payment_id = %s 
            ORDER BY created_at DESC
        """, (payment_id,))
```

**Обработка ошибок при возврате:**
```python
class RefundErrorHandler:
    async def handle_stripe_error(self, error: Exception) -> dict:
        """
        Обработай ошибку от Stripe при возврате.
        """
        if isinstance(error, stripe.error.InvalidRequestError):
            if "already refunded" in str(error):
                # Stripe уже обработал возврат
                return {
                    "status": "succeeded",
                    "reason": "already_refunded_by_provider"
                }
            elif "no pending charge" in str(error):
                # Платёж уже отменён
                return {
                    "status": "failed",
                    "reason": "charge_not_found"
                }
        
        if isinstance(error, stripe.error.RateLimitError):
            # Retry later
            return {
                "status": "pending",
                "reason": "rate_limited"
            }
        
        # Неизвестная ошибка
        return {
            "status": "failed",
            "reason": "unknown_error",
            "error": str(error)
        }
```

**Типичные ошибки.**
- Не проверить, что платёж в статусе succeeded → попытка возврата failed платежа.
- Позволить возврат суммы > первоначального платежа → финансовое несоответствие.
- Не использовать идемпотентность для возвратов → дублировать возвраты.
- Обновить БД, но не отправить в Stripe → несоответствие между системой и провайдером.
- Не обновить статус платежа на partially_refunded → невозможно проверить возвратимость.

**На интервью.**
- Объясни, как трёхфазный коммит помогает в возвратах (запись в БД → внешний провайдер → реестр).
- Упомяни partial refund и его сложность (несколько возвратов одного платежа).
- Уточняющий вопрос: «Как обработать, если Stripe подтвердил возврат, но наша БД упала?» → Webhook от Stripe придёт позже, будет идемпотентным.
- Уточняющий вопрос: «Почему частичные возвраты требуют особой логики?» → Нужно отслеживать сумму, которую ещё можно вернуть.

---

### 4. Как спроектировать reconciliation (ежедневная сверка с провайдерами платежей)?

**Зачем спрашивают.** Reconciliation — критически важна для обнаружения ошибок, мошенничества, потерь данных. Это проверяет, понимаешь ли ты, что даже проверенные системы должны регулярно верифицироваться.

**Короткий ответ.** Каждый день выгружай свои платежи и платежи от Stripe/PayPal/Adyen. Сопоставь по ID и сумме. Найди discrepancies (недопоставленные, разные суммы, разные статусы). Залогируй, создай тикеты в очередь для ручной проверки.

**Детальный разбор.**

**Виды несоответствий:**
```
1. Missing External:
   Наше запись: payment_123, $100, succeeded
   Stripe: (ничего)
   → Платёж не дошёл до провайдера, но мы сказали клиенту "успешно"

2. Missing Internal:
   Stripe: payment_stripe_123, $100
   Наша запись: (ничего)
   → Stripe обработал, а мы не записали

3. Amount Mismatch:
   Наше: $100
   Stripe: $95
   → Мы пересчитали комиссию иначе

4. Status Mismatch:
   Наше: succeeded
   Stripe: failed
   → Статус не синхронизировался

5. Timing Issue:
   Платёж в нашей БД создан в 23:59
   Stripe отчёт на 00:00 (другой день)
   → Скрещивание дней, нормально с задержкой
```

**Архитектура reconciliation:**
```
┌────────────────────────────────────────────────────────────────┐
│                  Daily Reconciliation Job                       │
│                   (runs at 02:00 UTC)                          │
└─────────────────┬──────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┬─────────────┬──────────────┐
        │                   │             │              │
        ▼                   ▼             ▼              ▼
   Our DB          Stripe API        PayPal API     Adyen API
  (Transactions)   (Settlement)    (Settlement)    (Settlement)
        │                   │             │              │
        └─────────┬─────────┴─────────────┴──────────────┘
                  │
                  ▼
         ┌──────────────────────┐
         │  Match Transactions  │
         │  by external_id      │
         └─────────┬────────────┘
                   │
        ┌──────────┴──────────┬──────────────┬──────────┐
        │                     │              │          │
    Matched             Unmatched         Amount      Status
  (validate)            (external)      Mismatch    Mismatch
        │                     │              │          │
        ▼                     ▼              ▼          ▼
    Compare        Flag for         Investigate  Update our
    amounts &      investigation    reason       status
    statuses
        │                     │              │          │
        └─────────┬───────────┴──────────────┴──────────┘
                  │
                  ▼
         ┌──────────────────────┐
         │  Generate Report     │
         │ & Alert Operations   │
         └──────────────────────┘
```

**SQL для вытягивания данных:**
```sql
-- Наши платежи за день
SELECT 
    p.id,
    p.external_id,
    p.amount,
    p.currency,
    p.status,
    p.created_at
FROM payments p
WHERE DATE(p.created_at) = CURRENT_DATE - INTERVAL '1 day'
  AND p.status IN ('succeeded', 'failed')
ORDER BY p.external_id;

-- Данные от Stripe (после выгрузки)
SELECT 
    id AS external_id,
    amount,
    currency,
    status,
    created_at
FROM stripe_settlement
WHERE DATE(created_at) = CURRENT_DATE - INTERVAL '1 day'
ORDER BY id;
```

**Код reconciliation (Python):**
```python
class ReconciliationService:
    def __init__(self, db, stripe_api, paypal_api, adyen_api, logger):
        self.db = db
        self.stripe = stripe_api
        self.paypal = paypal_api
        self.adyen = adyen_api
        self.logger = logger
    
    async def reconcile_daily(self, date: date = None):
        """
        Выполни ежедневную сверку.
        """
        if date is None:
            date = date.today() - timedelta(days=1)  # вчера
        
        self.logger.info(f"Starting reconciliation for {date}")
        
        # 1. Получаем наши платежи
        internal_payments = await self.db.query("""
            SELECT id, external_id, amount, currency, status, created_at
            FROM payments
            WHERE DATE(created_at) = %s
              AND status IN ('succeeded', 'failed')
            ORDER BY external_id
        """, (date,))
        
        # 2. Получаем платежи от провайдеров
        provider_payments = {}
        try:
            provider_payments['stripe'] = await self._fetch_stripe_settlement(date)
            provider_payments['paypal'] = await self._fetch_paypal_settlement(date)
            provider_payments['adyen'] = await self._fetch_adyen_settlement(date)
        except Exception as e:
            self.logger.error(f"Failed to fetch provider data: {e}")
            raise
        
        # 3. Сопоставляем платежи
        matched, unmatched_internal, unmatched_external = self._match_payments(
            internal_payments,
            provider_payments
        )
        
        # 4. Находим discrepancies
        discrepancies = []
        for match in matched:
            issues = self._check_discrepancies(match)
            discrepancies.extend(issues)
        
        # 5. Создаём отчёт
        report = ReconciliationReport(
            date=date,
            total_internal=len(internal_payments),
            total_external=sum(len(p) for p in provider_payments.values()),
            matched=len(matched),
            unmatched_internal=len(unmatched_internal),
            unmatched_external=len(unmatched_external),
            discrepancies=discrepancies,
            created_at=datetime.utcnow()
        )
        
        # 6. Сохраняем отчёт
        await self.db.insert_reconciliation_report(report)
        
        # 7. Отправляем алерты при проблемах
        if unmatched_internal or unmatched_external or discrepancies:
            await self._alert_operations(report)
        
        self.logger.info(f"Reconciliation complete: {report.summary()}")
        return report
    
    async def _fetch_stripe_settlement(self, date: date):
        """
        Получи список платежей от Stripe API.
        """
        # Stripe обычно выплачивает на следующий день
        # Но данные по платежам доступны сразу
        payments = []
        cursor = None
        
        while True:
            response = await self.stripe.payout_transactions(
                payout=None,  # все платежи (не только выплаченные)
                created={
                    'gte': int(date.timestamp()),
                    'lt': int((date + timedelta(days=1)).timestamp())
                },
                limit=100,
                starting_after=cursor
            )
            
            for txn in response['data']:
                if txn['type'] == 'charge':
                    payments.append({
                        'external_id': txn['source']['id'],
                        'amount': txn['amount'],
                        'currency': txn['currency'],
                        'status': self._map_stripe_status(txn['source']),
                        'created_at': datetime.fromtimestamp(txn['created'])
                    })
            
            if response['has_more']:
                cursor = response['data'][-1]['id']
            else:
                break
        
        return payments
    
    async def _fetch_paypal_settlement(self, date: date):
        """Выгрузи от PayPal."""
        # PayPal предоставляет отчёты для скачивания
        transactions = []
        
        report = await self.paypal.get_daily_report(date)
        for row in report:
            if row['status'] == 'Completed':
                transactions.append({
                    'external_id': row['transaction_id'],
                    'amount': int(float(row['amount']) * 100),  # в центы
                    'currency': row['currency'],
                    'status': 'succeeded',
                    'created_at': datetime.fromisoformat(row['date'])
                })
        
        return transactions
    
    def _match_payments(self, internal, provider_dict):
        """
        Сопоставь наши платежи с данными провайдеров.
        """
        matched = []
        unmatched_internal = list(internal)
        unmatched_external = []
        
        for provider_name, external_payments in provider_dict.items():
            for ext_payment in external_payments:
                # Ищем во внутренних по external_id
                for i, int_payment in enumerate(unmatched_internal):
                    if int_payment['external_id'] == ext_payment['external_id']:
                        matched.append({
                            'internal': int_payment,
                            'external': ext_payment,
                            'provider': provider_name
                        })
                        unmatched_internal.pop(i)
                        break
                else:
                    # Не нашли
                    unmatched_external.append({
                        **ext_payment,
                        'provider': provider_name
                    })
        
        return matched, unmatched_internal, unmatched_external
    
    def _check_discrepancies(self, match):
        """
        Проверь соответствие между внутренним и внешним платежом.
        """
        issues = []
        internal = match['internal']
        external = match['external']
        
        # Проверяем сумму
        if internal['amount'] != external['amount']:
            issues.append({
                'type': 'amount_mismatch',
                'payment_id': internal['id'],
                'external_id': external['external_id'],
                'internal_amount': internal['amount'],
                'external_amount': external['amount'],
                'provider': match['provider']
            })
        
        # Проверяем статус
        if internal['status'] != external['status']:
            issues.append({
                'type': 'status_mismatch',
                'payment_id': internal['id'],
                'external_id': external['external_id'],
                'internal_status': internal['status'],
                'external_status': external['status'],
                'provider': match['provider']
            })
        
        # Проверяем валюту
        if internal['currency'] != external['currency']:
            issues.append({
                'type': 'currency_mismatch',
                'payment_id': internal['id'],
                'external_id': external['external_id'],
                'internal_currency': internal['currency'],
                'external_currency': external['currency'],
                'provider': match['provider']
            })
        
        return issues
    
    async def _alert_operations(self, report: ReconciliationReport):
        """
        Отправь алерт операционной команде.
        """
        if report.unmatched_internal:
            self.logger.warning(
                f"Unmatched internal: {len(report.unmatched_internal)} payments"
            )
            # Создаём тикеты для ручной проверки
            for payment in report.unmatched_internal:
                await self._create_support_ticket(
                    title=f"Payment {payment['id']} not found in provider settlement",
                    priority='high',
                    payment_id=payment['id']
                )
        
        if report.unmatched_external:
            self.logger.warning(
                f"Unmatched external: {len(report.unmatched_external)} payments"
            )
        
        if report.discrepancies:
            self.logger.warning(
                f"Discrepancies found: {len(report.discrepancies)}"
            )
            for disc in report.discrepancies:
                if disc['type'] == 'amount_mismatch':
                    self.logger.error(
                        f"Amount mismatch: {disc['payment_id']} "
                        f"({disc['internal_amount']} vs {disc['external_amount']})"
                    )
```

**Типичные ошибки.**
- Игнорировать временные зоны → сравнивать платежи из разных дней.
- Не учитывать задержку в отчётах провайдера → считать платёж потерянным слишком рано.
- Не обновлять статус платежа при обнаружении discrepancy → система остаётся в неправильном состоянии.
- Не создавать тикеты для ручной проверки → проблемы остаются незамеченными.
- Запускать reconciliation во время пиковых нагрузок → медленный процесс блокирует операции.

**На интервью.**
- Объясни, почему reconciliation необходимо даже если всё кажется корректным.
- Упомяни, как часто её запускать (ежедневно, иногда в реальном времени).
- Уточняющий вопрос: «Как обработать unmatched external платёж?» → Это может быть мошеничество или ошибка в коде; требует ручного расследования.
- Уточняющий вопрос: «Зачем проверять статус, если платёж был успешен локально?» → Провайдер может отозвать платёж (chargeback, fraud); статус может измениться.

---

### 5. Как убедиться в соответствии PCI DSS при проектировании платёжной системы?

**Зачем спрашивают.** PCI DSS — обязательное требование при работе с картами. Это не просто техника, это юридическое обязательство. Интервьюер проверяет, понимаешь ли ты компромисс между функциональностью и безопасностью.

**Короткий ответ.** Никогда не храни полный номер карты, CVV, PIN в своей БД. Используй tokenization от Stripe/PayPal (они PCI compliant). Если требуется хранение — используй HSM (Hardware Security Module) и шифрование. Регулярные security audits и penetration testing обязательны.

**Детальный разбор.**

**Требования PCI DSS (кратко):**
```
1. Firewall configuration
   → Изолировать payment infrastructure

2. No default credentials
   → Менять default пароли на всех устройствах

3. Protect stored data
   → Шифровать данные карт в покое и в движении

4. Monitor and test networks
   → Регулярные penetration tests

5. Maintain security policy
   → Документировать и обновлять политики

6. Restrict access to cardholder data
   → Only PCI compliant employees

7. Unique ID for each user
   → Audit trail всех доступов

8. Restrict access to data by business need
   → Least privilege principle

9. Encrypt transmission
   → TLS 1.2+ для всех платежей

10. Maintain security testing & procedures
    → Regular updates и patches

11. Regular monitoring & testing
    → WAF, IDS, log monitoring

12. Maintain information security policy
    → Training for all staff
```

**Архитектура PCI-compliant системы:**
```
┌──────────────────────────────────────────────────────────────────┐
│                    INTERNET (untrusted)                          │
└─────────────────────┬──────────────────────────────────────────┘
                      │
         ┌────────────▼──────────────┐
         │   Stripe Hosted Form      │
         │   (tokenization)          │  ← Клиент вводит карту
         │ PCI compliance: Stripe    │
         └────────────┬──────────────┘
                      │
          Card token: tok_visa_4242
                      │
         ┌────────────▼──────────────┐
         │   API Gateway             │
         │   (TLS 1.2, WAF)          │
         └────────────┬──────────────┘
                      │
         ┌────────────▼──────────────┐
         │   Payment Service         │
         │   (inside VPC)            │  ← НИКОГДА не получаем
         │                           │    полный номер карты
         └────────────┬──────────────┘
                      │
         ┌────────────▼──────────────┐
         │  Never store tokens       │
         │  Pass immediately to      │
         │  Stripe/PayPal            │
         │  (tokenization provider)  │
         └────────────┬──────────────┘
                      │
         ┌────────────▼──────────────┐
         │ Stripe API                │
         │ (PCI Level 1)             │
         │ - Stores tokens           │
         │ - Compliant               │
         └───────────────────────────┘
```

**Что НЕЛЬЗЯ делать:**
```
❌ НИКОГДА храни эту информацию в БД:
   - Полный номер карты (PAN)
   - Полный магнитный тек (full track data)
   - CVV / CVC (трёхзначный код)
   - PIN код
   - 3D Secure пароль

❌ НИКОГДА не логируй эту информацию:
   - Даже в production logs
   - Даже частично
   - Это требование PCI DSS v3.4

❌ НИКОГДА не передавай по незащищённому каналу:
   - Всегда TLS 1.2+
   - Никогда HTTP
   - Никогда unencrypted email

❌ НИКОГДА не реализуй собственное шифрование карт:
   - Сложно сделать правильно
   - Легко внедрить уязвимость
   - Используй tokenization (Stripe и т.д.)
```

**Что МОЖНО и НУЖНО хранить:**
```
✅ Tokenized card info:
   - tok_visa_1234 (Stripe token)
   
✅ Masked PAN (для отображения):
   - last_4: "4242"
   - brand: "visa"
   - exp_month: 12
   - exp_year: 2025
   - (это NOT cardholder data по PCI)

✅ Сами платежи:
   - Сумма
   - Дата
   - Статус
   - Customer ID
   - (без чувствительных данных карты)

✅ Logs (без PAN):
   - Payment created at 2024-01-15T10:00:00Z
   - Customer: cust_123
   - Token: tok_visa_****
   - Amount: $100
```

**Пример безопасной обработки платежа (Python):**
```python
class SecurePaymentProcessor:
    """
    PCI-compliant обработка платежей.
    Никогда не касаемся полного номера карты.
    """
    
    def __init__(self, stripe_api_key):
        self.stripe = stripe.StripeClient(api_key=stripe_api_key)
    
    async def process_payment_from_client(
        self,
        customer_id: str,
        token: str,  # ← tok_visa_xxx от клиента (не полный номер!)
        amount: int,
        currency: str,
        idempotency_key: str
    ) -> PaymentResult:
        """
        Обработай платёж. Требуй токен, никогда полный номер.
        """
        # Проверяем, что это токен, а не номер карты
        if token.startswith('4') or token.startswith('5'):  # выглядит как номер карты
            raise SecurityError("Received card number instead of token! PCI violation.")
        
        if not token.startswith('tok_'):  # Stripe токен
            raise SecurityError("Invalid token format")
        
        # Отправляем в Stripe (они PCI compliant)
        try:
            charge = await self.stripe.charges.create(
                amount=amount,
                currency=currency,
                source=token,  # используем токен
                idempotency_key=idempotency_key,
                metadata={
                    'customer_id': customer_id,
                    'order_id': idempotency_key
                }
            )
        except Exception as e:
            # Никогда не логируем полную информацию карты
            log.error(f"Payment failed for customer {customer_id}: {e}")
            raise
        
        # Сохраняем только маскированную информацию
        payment = Payment(
            id=charge.id,
            customer_id=customer_id,
            amount=amount,
            currency=currency,
            card_last_4=charge.source.last4,
            card_brand=charge.source.brand,
            card_exp_month=charge.source.exp_month,
            card_exp_year=charge.source.exp_year,
            status='succeeded' if charge.paid else 'failed',
            # НИКОГДА не храним:
            # - charge.source.number (полный номер)
            # - charge.source.cvc (CVV)
            # - charge.source.track_data
            created_at=datetime.utcnow()
        )
        
        await self.db.insert_payment(payment)
        
        return PaymentResult(
            payment_id=payment.id,
            status=payment.status,
            card_last_4=payment.card_last_4,  # ← безопасно отправить клиенту
            # НИКОГДА не включаем в ответ полный номер карты
        )
    
    async def handle_webhook_from_stripe(self, payload: dict):
        """
        Обработай webhook от Stripe (также PCI-compliant).
        Stripe никогда не отправляет полный номер карты в webhook.
        """
        event = payload['data']['object']
        
        if payload['type'] == 'charge.succeeded':
            payment_id = event['id']
            
            # Обновляем платёж
            await self.db.update_payment(
                id=payment_id,
                status='succeeded',
                # Stripe предоставляет маскированную информацию
                card_last_4=event['source']['last4'],
                card_brand=event['source']['brand']
            )
        
        elif payload['type'] == 'charge.failed':
            payment_id = event['id']
            await self.db.update_payment(
                id=payment_id,
                status='failed',
                error_message=event.get('failure_message')
            )
```

**Логирование без нарушения PCI:**
```python
class SecureLogger:
    """
    Логирование, соответствующее PCI DSS.
    """
    
    @staticmethod
    def mask_card_number(card: str) -> str:
        """Оставить только последние 4 цифры."""
        if len(card) >= 4:
            return f"****{card[-4:]}"
        return "****"
    
    @staticmethod
    def log_payment(payment_id, customer_id, amount, card_token):
        """Безопасное логирование платежа."""
        # ✅ ХОРОШО
        log.info(f"Payment {payment_id} for customer {customer_id}: ${amount} with token {card_token[:8]}...")
        
        # ❌ ПЛОХО (нарушение PCI)
        # log.info(f"Payment {payment_id} with card 4111111111111111")
        # log.info(f"CVV verification: 123")
```

**Требования к инфраструктуре:**
```
1. Network Segmentation:
   - Payment servers inside VPC
   - Restricted firewall rules
   - No direct internet access to payment DB

2. TLS/SSL:
   - All payment communications: TLS 1.2+
   - Certificate pinning for APIs
   - HSTS headers

3. Authentication:
   - Multi-factor authentication for admins
   - API keys with rotation
   - No hardcoded secrets (use Vault/SecretsManager)

4. Data Encryption:
   - AES-256 for stored data (if needed)
   - Tokenization is preferred over encryption
   - Key rotation: annually

5. Access Control:
   - Least privilege: only PCI-trained staff
   - Role-based access control (RBAC)
   - Audit logs for all access

6. Monitoring:
   - Intrusion detection system (IDS)
   - Web Application Firewall (WAF)
   - Real-time alerting
   - Log aggregation (ELK, Splunk)

7. Vulnerability Management:
   - Quarterly penetration testing
   - Vulnerability scanning
   - Patch management: within 30 days
   - Code reviews with security focus
```

**Типичные ошибки.**
- Логировать полный номер карты в Slack или email → автоматическое нарушение PCI.
- Хранить CVV → абсолютно запрещено, используй only для проверки в момент платежа.
- Передавать номер карты через API вместо токена → ты стал PCI responsible (дорого).
- Забыть про HTTPS → перехват номера карты в пути.
- Использовать старые версии TLS → уязвимости (SSL 3.0, TLS 1.0).

**На интервью.**
- Объясни, почему tokenization лучше шифрования.
- Упомяни, что даже одна ошибка может потребовать нарушения и штрафа (PCI fine: $5k-$100k+).
- Уточняющий вопрос: «Как обрабатывать сохранённые карты клиента (для подписки)?» → Stripe сохраняет токен, ты запрашиваешь по токену, не по номеру.
- Уточняющий вопрос: «Что делать при penetration testing?» → Разрешить безопасный тестинг, исправить найденные уязвимости.

---

(Продолжение в следующей части - questions 6-10)

### 6. Как интегрировать несколько платёжных провайдеров (Stripe, PayPal, Adyen) с автоматическим failover?

**Зачем спрашивают.** Один провайдер может упасть. Архитектура должна автоматически переключаться. Это проверяет умение строить отказоустойчивые системы.

**Короткий ответ.** Используй abstraction layer (PaymentGateway interface). Попробуй основной провайдер, при ошибке переключись на резервный. Кэшируй здоровье провайдеров для быстрого переключения. Логируй все попытки для анализа.

**Детальный разбор.**

**Архитектура failover:**
```
Payment Request
       │
       ▼
┌──────────────────────┐
│ PaymentGateway       │
│ (abstraction layer)  │
└──────────────┬───────┘
               │
    ┌──────────┼──────────┐
    │          │          │
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│ Stripe │ │ PayPal │ │ Adyen  │
│ (main) │ │ (back) │ │(backup)│
└────┬───┘ └───┬────┘ └───┬────┘
     │         │          │
     ▼         ▼          ▼
[Health Check] [Circuit Breaker]
     │         │          │
     └────┬────┴────┬─────┘
          │         │
      Healthy?  Try next
        YES       provider
         │
    Process
    with
   selected
   provider
```

**Пример кода:**
```python
from enum import Enum
from typing import List
import asyncio

class ProviderName(Enum):
    STRIPE = "stripe"
    PAYPAL = "paypal"
    ADYEN = "adyen"

class PaymentGateway:
    """
    Abstraction layer для платёжных провайдеров.
    Поддерживает failover и load balancing.
    """
    
    def __init__(self):
        self.providers = {
            ProviderName.STRIPE: StripeProvider(),
            ProviderName.PAYPAL: PayPalProvider(),
            ProviderName.ADYEN: AdyenProvider()
        }
        
        # Порядок попыток (основной → резервные)
        self.provider_order = [
            ProviderName.STRIPE,
            ProviderName.PAYPAL,
            ProviderName.ADYEN
        ]
        
        # Health check кэш (provider → is_healthy)
        self.health_cache = {p: True for p in ProviderName}
        self.health_check_interval = 30  # сек
    
    async def process_payment(
        self,
        amount: int,
        currency: str,
        token: str,
        idempotency_key: str
    ) -> PaymentResult:
        """
        Обработай платёж с failover.
        """
        errors = []
        
        for provider_name in self.provider_order:
            # Пропускаем нездоровых провайдеров
            if not await self._is_provider_healthy(provider_name):
                log.warning(f"Skipping unhealthy provider: {provider_name.value}")
                continue
            
            provider = self.providers[provider_name]
            
            try:
                log.info(f"Attempting payment with {provider_name.value}")
                result = await provider.charge(
                    amount=amount,
                    currency=currency,
                    token=token,
                    idempotency_key=idempotency_key
                )
                
                log.info(f"Payment succeeded with {provider_name.value}")
                
                # Сохраняем в БД какой провайдер обработал
                result.provider = provider_name.value
                return result
            
            except ProviderTemporaryError as e:
                # Временная ошибка (timeout, rate limit)
                log.warning(f"Temporary error from {provider_name.value}: {e}")
                errors.append((provider_name.value, "temporary", str(e)))
                
                # Пробуем следующего
                continue
            
            except ProviderPermanentError as e:
                # Постоянная ошибка (некорректные данные карты)
                log.error(f"Permanent error from {provider_name.value}: {e}")
                errors.append((provider_name.value, "permanent", str(e)))
                
                # Не переключаемся дальше, возвращаем ошибку клиенту
                raise
            
            except Exception as e:
                # Неизвестная ошибка
                log.error(f"Unknown error from {provider_name.value}: {e}")
                errors.append((provider_name.value, "unknown", str(e)))
                
                # Отмечаем провайдера как нездорового
                await self._mark_provider_unhealthy(provider_name)
                
                # Пробуем следующего
                continue
        
        # Все провайдеры не удались
        log.error(f"All payment providers failed: {errors}")
        raise PaymentProcessingError(
            f"All payment providers unavailable. Errors: {errors}"
        )
    
    async def _is_provider_healthy(self, provider_name: ProviderName) -> bool:
        """
        Проверь здоровье провайдера (из кэша, без реального вызова).
        """
        # Если в кэше помечен как нездоровый, не пробуем
        if not self.health_cache.get(provider_name, True):
            # Периодически переопробуем даже нездоровых
            if time.time() > self.health_check_last[provider_name] + self.health_check_interval:
                asyncio.create_task(self._health_check_provider(provider_name))
            
            return False
        
        return True
    
    async def _mark_provider_unhealthy(self, provider_name: ProviderName):
        """Отметь провайдера как нездорового."""
        self.health_cache[provider_name] = False
        self.health_check_last[provider_name] = time.time()
        
        log.warning(f"Provider {provider_name.value} marked as unhealthy")
    
    async def _health_check_provider(self, provider_name: ProviderName):
        """
        Периодическая проверка здоровья провайдера (background).
        """
        try:
            provider = self.providers[provider_name]
            await provider.health_check()
            
            self.health_cache[provider_name] = True
            log.info(f"Provider {provider_name.value} is healthy again")
        
        except Exception as e:
            log.error(f"Health check failed for {provider_name.value}: {e}")
```

**Circuit Breaker паттерн:**
```python
class CircuitBreaker:
    """
    Защита от cascade failure (когда падающий сервис заваливает систему).
    """
    
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        
        self.failure_count = 0
        self.state = "CLOSED"  # CLOSED (работает) → OPEN (упал) → HALF_OPEN (восстанавливается)
        self.opened_at = None
    
    async def call(self, provider_func, *args, **kwargs):
        """
        Выполни функцию с protection от cascade failure.
        """
        if self.state == "OPEN":
            # Провайдер падает, не пробуем
            if time.time() - self.opened_at > self.timeout:
                # Прошло 60 сек, пробуем восстановиться
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError(f"Circuit is open for {self.timeout}s")
        
        try:
            result = await provider_func(*args, **kwargs)
            
            # Успех → сбрасываем счётчик ошибок
            self.failure_count = 0
            
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                log.info("Circuit breaker closed (provider recovered)")
            
            return result
        
        except Exception as e:
            self.failure_count += 1
            
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
                self.opened_at = time.time()
                log.error(f"Circuit breaker opened (threshold: {self.failure_threshold})")
            
            raise
```

**Распределение нагрузки (load balancing):**
```python
class LoadBalancedGateway(PaymentGateway):
    """
    Распределяй платежи между провайдерами для оптимизации.
    """
    
    async def process_payment(self, *args, **kwargs):
        """Используй основного провайдера только если это целесообразно."""
        
        # Выбираем провайдера на основе:
        # 1. Здоровья
        # 2. Текущей нагрузки
        # 3. Цены комиссии
        # 4. Поддержки валюты
        
        selected_providers = []
        
        for provider_name in self.provider_order:
            if not await self._is_provider_healthy(provider_name):
                continue
            
            provider = self.providers[provider_name]
            
            # Проверяем, поддерживает ли валюту
            if not provider.supports_currency(kwargs['currency']):
                continue
            
            # Получаем метрики
            current_load = await provider.get_current_load()
            fee_percent = provider.get_fee_percent()
            
            selected_providers.append({
                'name': provider_name,
                'provider': provider,
                'load': current_load,
                'fee': fee_percent,
                'priority': self.provider_order.index(provider_name)
            })
        
        # Сортируем: основной (priority 0), потом по нагрузке
        selected_providers.sort(
            key=lambda x: (x['priority'], x['load'])
        )
        
        for entry in selected_providers:
            try:
                result = await entry['provider'].charge(*args, **kwargs)
                result.provider = entry['name'].value
                return result
            except Exception as e:
                log.warning(f"Failed with {entry['name'].value}: {e}")
                continue
        
        raise PaymentProcessingError("All providers failed")
```

**Типичные ошибки.**
- Не кэшировать здоровье провайдеров → каждый платёж проверяет все провайдеры (медленно).
- Не обрабатывать разные типы ошибок → переключаться и при постоянных ошибках (бессмыслено).
- Не логировать какой провайдер использовался → невозможно дебажить проблемы.
- Забыть про idempotency_key при failover → risk повторной обработки.

**На интервью.**
- Объясни разницу между temporary и permanent errors.
- Упомяни Circuit Breaker для защиты от cascade.
- Уточняющий вопрос: «Как тестировать failover без падения реального провайдера?» → Mock провайдер, inject ошибки.

---

### 7. Как обработать retry и exponential backoff при ошибках платежей?

**Зачем спрашивают.** Сетевые ошибки случаются. Слепой retry вызовет двойные платежи. Нужна стратегия retry, которая уважает идемпотентность и не перегружает систему.

**Короткий ответ.** Используй exponential backoff с jitter: 1s, 2s, 4s, 8s... Максимум 3-5 попыток. Используй idempotency_key для всех повторов. Distinguish transient errors (retry) от permanent (fail immediately).

**Детальный разбор.**

**Стратегии retry:**
```
Blind Retry (ПЛОХО):
  Попытка 1: TIMEOUT
  Попытка 2: Сразу же
  Попытка 3: Сразу же
  → Перегружаем систему, не даём ей восстановиться

Exponential Backoff (ХОРОШО):
  Попытка 1: TIMEOUT
  Попытка 2: ждём 1s
  Попытка 3: ждём 2s
  Попытка 4: ждём 4s
  → Даём системе время восстановиться

Exponential Backoff + Jitter (ЕЩЁ ЛУЧШЕ):
  Попытка 1: TIMEOUT
  Попытка 2: ждём 1s + random(0-1s) = 0.3s
  Попытка 3: ждём 2s + random(0-2s) = 1.8s
  Попытка 4: ждём 4s + random(0-4s) = 2.1s
  → Распределяем нагрузку (избегаем thundering herd)
```

**Код с retry:**
```python
import asyncio
import random
from typing import Callable, Optional

class RetryConfig:
    """Конфигурация retry стратегии."""
    
    def __init__(
        self,
        max_attempts: int = 3,
        initial_backoff_ms: int = 1000,
        max_backoff_ms: int = 30000,
        jitter: bool = True
    ):
        self.max_attempts = max_attempts
        self.initial_backoff_ms = initial_backoff_ms
        self.max_backoff_ms = max_backoff_ms
        self.jitter = jitter
    
    def calculate_backoff(self, attempt: int) -> float:
        """
        Вычисли время ожидания для попытки (в секундах).
        """
        backoff = min(
            self.initial_backoff_ms * (2 ** attempt),
            self.max_backoff_ms
        )
        
        if self.jitter:
            # Добавляем случайную величину до backoff
            jitter_ms = random.randint(0, backoff)
            backoff = jitter_ms
        
        return backoff / 1000.0  # в секунды


class RetryablePaymentProcessor:
    """
    Обработка платежей с retry и exponential backoff.
    """
    
    RETRYABLE_ERRORS = (
        ProviderTimeoutError,
        ConnectionError,
        TimeoutError,
        ProviderRateLimitError,  # 429 Too Many Requests
    )
    
    NON_RETRYABLE_ERRORS = (
        InvalidCardError,
        InsufficientFundsError,
        CardDeclinedError,
        InvalidAmountError,
    )
    
    def __init__(self, gateway: PaymentGateway):
        self.gateway = gateway
        self.retry_config = RetryConfig(max_attempts=3)
    
    async def process_payment_with_retry(
        self,
        customer_id: str,
        amount: int,
        currency: str,
        token: str,
        idempotency_key: str
    ) -> PaymentResult:
        """
        Обработай платёж с retry и exponential backoff.
        """
        last_error = None
        
        for attempt in range(self.retry_config.max_attempts):
            try:
                log.info(
                    f"Payment attempt {attempt + 1}/{self.retry_config.max_attempts} "
                    f"for {customer_id} (key: {idempotency_key})"
                )
                
                result = await self.gateway.process_payment(
                    amount=amount,
                    currency=currency,
                    token=token,
                    idempotency_key=idempotency_key  # ← КРИТИЧНО: одинаковый для всех попыток
                )
                
                log.info(f"Payment succeeded on attempt {attempt + 1}")
                return result
            
            except Exception as e:
                last_error = e
                
                # Проверяем тип ошибки
                if isinstance(e, self.NON_RETRYABLE_ERRORS):
                    # Постоянная ошибка → не пробуем дальше
                    log.error(f"Non-retryable error: {e}")
                    raise
                
                if not isinstance(e, self.RETRYABLE_ERRORS):
                    # Неизвестная ошибка → не пробуем дальше
                    log.error(f"Unknown error type, not retrying: {e}")
                    raise
                
                # Это transient error
                if attempt < self.retry_config.max_attempts - 1:
                    # Ещё есть попытки
                    backoff_seconds = self.retry_config.calculate_backoff(attempt)
                    log.warning(
                        f"Payment failed on attempt {attempt + 1}: {e}. "
                        f"Retrying in {backoff_seconds:.1f}s..."
                    )
                    
                    await asyncio.sleep(backoff_seconds)
                    # Цикл продолжается
                else:
                    # Исчерпали все попытки
                    log.error(
                        f"Payment failed after {self.retry_config.max_attempts} attempts: {e}"
                    )
        
        raise PaymentRetryExhaustedError(
            f"Failed after {self.retry_config.max_attempts} attempts",
            last_error=last_error
        )
```

**Dead Letter Queue для failed платежей:**
```python
class PaymentRetryQueue:
    """
    Очередь для повторной обработки failed платежей.
    """
    
    def __init__(self, db, message_queue):
        self.db = db
        self.queue = message_queue
    
    async def handle_failed_payment(
        self,
        payment_id: str,
        customer_id: str,
        error: Exception,
        attempt: int
    ):
        """
        Сохрани failed платёж для позже обработки.
        """
        # Сохраняем в DLQ (Dead Letter Queue)
        await self.db.insert_payment_retry(
            payment_id=payment_id,
            customer_id=customer_id,
            error_message=str(error),
            attempt=attempt,
            next_retry_at=self._calculate_next_retry_time(attempt),
            created_at=datetime.utcnow()
        )
        
        # Также кладём в очередь для background worker'а
        await self.queue.enqueue(
            task='retry_failed_payment',
            kwargs={'payment_id': payment_id},
            scheduled_at=self._calculate_next_retry_time(attempt)
        )
    
    def _calculate_next_retry_time(self, attempt: int) -> datetime:
        """
        Следующая попытка через exponential backoff.
        """
        backoff_minutes = min(2 ** attempt, 1440)  # макс 1 день
        return datetime.utcnow() + timedelta(minutes=backoff_minutes)
    
    async def process_retry_queue(self):
        """
        Background job, обрабатывающий очередь retry.
        Запускается периодически (напр., каждую минуту).
        """
        # Получаем платежи, пора которых пришла
        pending = await self.db.query("""
            SELECT * FROM payment_retries
            WHERE next_retry_at <= NOW()
              AND attempt < 5
            LIMIT 100
        """)
        
        for retry in pending:
            try:
                payment = await self.db.get_payment(retry['payment_id'])
                
                # Пробуем обработать с тем же idempotency key
                result = await self.process_payment_with_retry(
                    customer_id=payment.customer_id,
                    amount=payment.amount,
                    currency=payment.currency,
                    token=payment.token,
                    idempotency_key=payment.idempotency_key  # ← идемпотентно!
                )
                
                # Успех → удаляем из DLQ
                await self.db.delete_payment_retry(retry['id'])
                log.info(f"Retried payment {retry['payment_id']} succeeded")
            
            except Exception as e:
                # Ещё упал → обновляем DLQ
                await self.db.update_payment_retry(
                    id=retry['id'],
                    attempt=retry['attempt'] + 1,
                    error_message=str(e),
                    next_retry_at=self._calculate_next_retry_time(retry['attempt'] + 1)
                )
                
                log.warning(f"Retry for {retry['payment_id']} failed again: {e}")
                
                # После 5 попыток отправляем в real DLQ для manual review
                if retry['attempt'] >= 4:
                    await self.alert_operations(
                        f"Payment {retry['payment_id']} failed after 5 retries",
                        retry=retry
                    )
```

**Типичные ошибки.**
- Retry без idempotency key → двойные платежи.
- Retry сразу без backoff → перегрузить систему.
- Retry всех ошибок одинаково → повторять постоянные ошибки.
- Забыть про maximal backoff → retry может происходить слишком долго.
- Не логировать retry attempts → невозможно дебажить.

**На интервью.**
- Объясни exponential backoff + jitter.
- Покажи, как различать transient vs permanent errors.
- Уточняющий вопрос: «Как тестировать retry без reальных ошибок?» → Inject ошибки в тестах, используй mocking.

---

### 8. Как масштабировать платёжную систему при пиковых нагрузках (Black Friday, праздники)?

**Зачем спрашивают.** 100 QPS → 1000 QPS (10x spike). База данных падает, API timeout'ы. Нужна архитектура, которая справляется с пиками.

**Короткий ответ.** Используй асинхронность (очереди), кэширование, database sharding, load balancing, auto-scaling. Платежи обрабатывай в фоне (статус: pending) и нотифицируй по webhook'е.

**Детальный разбор.**

**Архитектура при скейлировании:**
```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer                            │
│              (sticky sessions)                              │
└────────────┬──────────────┬──────────────┬──────────────────┘
             │              │              │
       ┌─────▼────┐    ┌─────▼────┐   ┌─────▼────┐
       │ API Pod  │    │ API Pod  │   │ API Pod  │  (auto-scale)
       │ Instance │    │ Instance │   │ Instance │
       └─────┬────┘    └─────┬────┘   └─────┬────┘
             │              │              │
             └──────────────┼──────────────┘
                            │
                     ┌──────▼──────┐
                     │   Redis     │  (session store, cache)
                     │   Cluster   │
                     └──────┬──────┘
                            │
             ┌──────────────┼──────────────┐
             │              │              │
        ┌────▼────┐    ┌────▼────┐   ┌────▼────┐
        │ Queue   │    │ Queue   │   │ Queue   │
        │ (RabbitMQ)   │          │   │ Redis   │
        └────┬────┘    └────┬────┘   └────┬────┘
             │              │              │
       ┌─────▼──────────────▼──────────────▼──────┐
       │    Payment Workers (background)         │
       │  (consume from queue, process payments) │
       │  (multiple instances)                   │
       └──────────────┬──────────────────────────┘
                      │
             ┌────────▼────────┐
             │    Database     │
             │  (partitioned)  │
             └─────────────────┘
```

**Асинхронная обработка платежей:**
```python
class AsyncPaymentProcessor:
    """
    Асинхронная обработка платежей для скейлирования.
    Клиент получает ответ быстро (статус: pending).
    Фактическая обработка в фоне (queue).
    """
    
    def __init__(self, db, queue, payment_gateway):
        self.db = db
        self.queue = queue
        self.gateway = payment_gateway
    
    async def initiate_payment(
        self,
        customer_id: str,
        amount: int,
        currency: str,
        token: str,
        idempotency_key: str
    ) -> PaymentResponse:
        """
        Инициируй платёж (быстро, асинхронно).
        """
        async with self.db.transaction():
            # Сразу создаём запись платежа
            payment = Payment(
                id=str(uuid4()),
                idempotency_key=idempotency_key,
                customer_id=customer_id,
                amount=amount,
                currency=currency,
                status='pending',  # ← КЛЮЧ: сразу pending, обработка в фоне
                created_at=datetime.utcnow()
            )
            
            await self.db.insert_payment(payment)
            
            # Кладём в очередь для обработки
            await self.queue.enqueue(
                task='process_payment',
                kwargs={
                    'payment_id': payment.id,
                    'token': token,
                    'idempotency_key': idempotency_key
                },
                priority='high'  # платежи в приоритете
            )
            
            # Возвращаем клиенту быстро
            return PaymentResponse(
                payment_id=payment.id,
                status='pending',  # не "succeeded" пока!
                message='Payment is being processed. You will be notified via webhook.'
            )
    
    async def process_payment_background(
        self,
        payment_id: str,
        token: str,
        idempotency_key: str
    ):
        """
        Фоновая обработка платежа (запускается из очереди).
        """
        payment = await self.db.get_payment(payment_id)
        
        try:
            # Обрабатываем в провайдере
            result = await self.gateway.process_payment(
                amount=payment.amount,
                currency=payment.currency,
                token=token,
                idempotency_key=idempotency_key
            )
            
            # Обновляем статус
            payment.status = 'succeeded'
            payment.external_id = result.external_id
            payment.updated_at = datetime.utcnow()
            
        except Exception as e:
            payment.status = 'failed'
            payment.error_message = str(e)
            payment.updated_at = datetime.utcnow()
        
        finally:
            await self.db.update_payment(payment)
            
            # Отправляем webhook клиенту (асинхронный callback)
            await self._send_webhook(payment)
    
    async def _send_webhook(self, payment: Payment):
        """Уведомь клиента про статус платежа."""
        # Получаем webhook URL клиента
        customer = await self.db.get_customer(payment.customer_id)
        
        if not customer.webhook_url:
            return
        
        payload = {
            'payment_id': payment.id,
            'status': payment.status,
            'amount': payment.amount,
            'currency': payment.currency,
            'timestamp': payment.updated_at.isoformat()
        }
        
        # Отправляем с retry
        for attempt in range(3):
            try:
                async with aiohttp.ClientSession() as session:
                    async with session.post(
                        customer.webhook_url,
                        json=payload,
                        timeout=aiohttp.ClientTimeout(total=10)
                    ) as response:
                        if response.status == 200:
                            log.info(f"Webhook sent for payment {payment.id}")
                            return
            except Exception as e:
                if attempt < 2:
                    await asyncio.sleep(2 ** attempt)  # exponential backoff
                else:
                    log.error(f"Webhook failed for payment {payment.id}: {e}")
```

**Database Sharding для масштабирования:**
```python
class ShardedPaymentDB:
    """
    Разбей платежи на шарды для масштабирования БД.
    """
    
    SHARD_COUNT = 256  # 256 шардов
    
    def __init__(self, shard_configs: dict):
        """
        shard_configs: {
            0: ("host1", "port1", "db1"),
            1: ("host2", "port2", "db2"),
            ...
        }
        """
        self.shards = {}
        for shard_id, config in shard_configs.items():
            self.shards[shard_id] = PostgresConnection(*config)
    
    def _get_shard_id(self, customer_id: str) -> int:
        """
        Определи шард на основе customer_id (consistent hashing).
        """
        hash_value = hash(customer_id)
        return hash_value % self.SHARD_COUNT
    
    async def insert_payment(self, payment: Payment):
        """Вставь платёж в правильный шард."""
        shard_id = self._get_shard_id(payment.customer_id)
        shard = self.shards[shard_id]
        
        await shard.execute("""
            INSERT INTO payments (...) VALUES (...)
        """, payment)
    
    async def get_customer_payments(
        self,
        customer_id: str,
        limit: int = 10
    ) -> List[Payment]:
        """Получи платежи клиента (все из одного шарда)."""
        shard_id = self._get_shard_id(customer_id)
        shard = self.shards[shard_id]
        
        return await shard.query("""
            SELECT * FROM payments
            WHERE customer_id = %s
            ORDER BY created_at DESC
            LIMIT %s
        """, (customer_id, limit))
```

**Кэширование для скейлирования:**
```python
class CachedPaymentService:
    """Кэшируй frequently accessed data."""
    
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache
    
    async def get_payment(self, payment_id: str) -> Payment:
        """
        Получи платёж с кэшированием.
        """
        # Проверяем кэш (Redis)
        cached = await self.cache.get(f"payment:{payment_id}")
        if cached:
            return json.loads(cached)
        
        # Получаем из БД
        payment = await self.db.get_payment(payment_id)
        
        # Кэшируем (TTL: 1 час)
        await self.cache.set(
            f"payment:{payment_id}",
            json.dumps(payment.to_dict()),
            ex=3600
        )
        
        return payment
    
    async def invalidate_payment_cache(self, payment_id: str):
        """Инвалидируй кэш при обновлении платежа."""
        await self.cache.delete(f"payment:{payment_id}")
```

**Auto-scaling конфигурация:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3  # базовое количество pod'ов
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2  # максимум +2 pod'а за раз
      maxUnavailable: 0  # 0 downtime
  
  template:
    spec:
      containers:
      - name: payment-api
        image: payment-api:v1
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        
        livenessProbe:  # pod жив?
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        
        readinessProbe:  # pod готов к трафику?
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 2

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-api
  
  minReplicas: 3
  maxReplicas: 50  # максимум 50 pod'ов при spike
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # scale при 70% CPU
  
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # scale при 80% Memory
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # жди 5 мин перед scale down
      policies:
      - type: Percent
        value: 50  # удаляй максимум 50% pod'ов
    
    scaleUp:
      stabilizationWindowSeconds: 0  # scale up немедленно
      policies:
      - type: Percent
        value: 100  # удваивай
```

**Типичные ошибки.**
- Синхронная обработка при высокой нагрузке → timeout'ы.
- Без кэширования → перегрузка БД при повторных запросах.
- Без sharding → одна БД не выдержит.
- Без auto-scaling → ручное управление pod'ами.

**На интервью.**
- Объясни асинхронную обработку платежей.
- Упомяни database sharding и key (customer_id).
- Уточняющий вопрос: «Как обработать webhook если URL недоступен?» → Retry с exponential backoff, потом DLQ.

---

### 9. Как обрабатывать chargebacks и dispute'ы от клиентов?

**Зачем спрашивают.** Chargeback — когда клиент оспаривает платёж у банка. Это не просто возврат, это штраф. Система должна обнаруживать и обрабатывать.

**Короткий ответ.** Слушай webhook'и от провайдера (Stripe sends chargeback notifications). Обновляй статус платежа на "chargeback". Регистрируй в реестре как отрицательную запись. Анализируй паттерны для выявления мошенников.

**Детальный разбор.**

**Жизненный цикл chargeback:**
```
1. Платёж успешен (succeeded)
   ↓
2. Клиент оспаривает платёж в банке (через несколько дней)
   ↓
3. Stripe получает уведомление от банка
   ↓
4. Stripe отправляет webhook нам: "charge.dispute.created"
   ↓
5. Мы обновляем платёж на статус "chargeback"
   ↓
6. Клиент может представить доказательства (invoice, email)
   ↓
7. Stripe отправляет "charge.dispute.closed" (won/lost)
   ↓
8. Мы обновляем финальный статус + штраф
```

**Обработка webhook'а от Stripe:**
```python
class ChargebackHandler:
    """
    Обработка chargebacks и disputes от платёжного провайдера.
    """
    
    def __init__(self, db, ledger_service):
        self.db = db
        self.ledger = ledger_service
    
    async def handle_stripe_webhook(self, event: dict):
        """
        Обработай webhook от Stripe.
        """
        event_type = event['type']
        data = event['data']['object']
        
        if event_type == 'charge.dispute.created':
            await self._handle_dispute_created(data)
        
        elif event_type == 'charge.dispute.evidence.submitted':
            await self._handle_evidence_submitted(data)
        
        elif event_type == 'charge.dispute.closed':
            await self._handle_dispute_closed(data)
    
    async def _handle_dispute_created(self, dispute: dict):
        """
        Клиент оспорил платёж.
        """
        charge_id = dispute['charge']
        
        # Находим платёж по external_id
        payment = await self.db.get_payment_by_external_id(charge_id)
        
        if not payment:
            log.error(f"Dispute for unknown charge: {charge_id}")
            return
        
        # Обновляем статус на chargeback
        payment.status = 'chargeback'
        payment.dispute_id = dispute['id']
        payment.dispute_reason = dispute['reason']
        payment.updated_at = datetime.utcnow()
        
        await self.db.update_payment(payment)
        
        # Записываем в реестр (денег вернулось клиенту, минус штраф)
        chargeback_amount = dispute['amount'] + 1500  # $1500 штраф Stripe
        
        await self.ledger.record_chargeback(
            payment_id=payment.id,
            chargeback_amount=chargeback_amount,
            reason=dispute['reason']
        )
        
        # Отправляем алерт на операции
        await self._alert_chargeback(payment, dispute)
        
        log.warning(f"Chargeback created: {charge_id}, amount: {chargeback_amount}")
    
    async def _handle_dispute_closed(self, dispute: dict):
        """
        Спор разрешён (в пользу клиента или продавца).
        """
        charge_id = dispute['charge']
        payment = await self.db.get_payment_by_external_id(charge_id)
        
        if not payment:
            return
        
        # Обновляем статус в зависимости от исхода
        if dispute['status'] == 'won':
            # Мы выиграли → платёж остаётся успешным, штраф не платим
            payment.status = 'succeeded'
            payment.chargeback_status = 'won'
            
            log.info(f"Chargeback won: {charge_id}")
        
        elif dispute['status'] == 'lost':
            # Мы проиграли → платёж теряется
            payment.status = 'chargeback_lost'
            payment.chargeback_status = 'lost'
            
            log.warning(f"Chargeback lost: {charge_id}")
        
        payment.updated_at = datetime.utcnow()
        await self.db.update_payment(payment)
    
    async def _alert_chargeback(self, payment: Payment, dispute: dict):
        """
        Отправь алерт operationс при chargeback.
        """
        message = f"""
        ⚠️ CHARGEBACK RECEIVED
        
        Payment ID: {payment.id}
        Customer: {payment.customer_id}
        Amount: ${payment.amount / 100:.2f}
        Reason: {dispute['reason']}
        
        Action Required: Review evidence by {dispute['evidence_due_by']}
        """
        
        await self._send_slack_alert(message)
        await self._create_support_ticket(message, priority='critical')


class ChargebackAnalytics:
    """
    Анализ chargebacks для выявления мошенников.
    """
    
    async def detect_fraud_patterns(self, db):
        """
        Найди pattern'ы, указывающие на мошенничество.
        """
        # 1. Клиент с высоким rate chargeback'ов
        high_chargeback_customers = await db.query("""
            SELECT 
                customer_id,
                COUNT(*) as chargeback_count,
                COUNT(*)::FLOAT / 
                (SELECT COUNT(*) FROM payments WHERE customer_id = p.customer_id) 
                    as chargeback_rate
            FROM payments p
            WHERE status IN ('chargeback', 'chargeback_lost')
              AND created_at > NOW() - INTERVAL '30 days'
            GROUP BY customer_id
            HAVING COUNT(*) >= 3 OR (COUNT(*)::FLOAT / ...) > 0.5
        """)
        
        for row in high_chargeback_customers:
            log.warning(
                f"High chargeback customer: {row['customer_id']}, "
                f"rate: {row['chargeback_rate']:.1%}"
            )
            
            # Action: flag для manual review, может быть blacklist
            await self._flag_customer_for_review(row['customer_id'])
        
        # 2. Same card used with multiple customer accounts (account takeover)
        suspicious_cards = await db.query("""
            SELECT 
                card_token,
                COUNT(DISTINCT customer_id) as customer_count,
                COUNT(*) as transaction_count,
                COUNT(CASE WHEN status = 'chargeback' THEN 1 END) as chargeback_count
            FROM payments
            WHERE created_at > NOW() - INTERVAL '7 days'
            GROUP BY card_token
            HAVING COUNT(DISTINCT customer_id) > 5
        """)
        
        for row in suspicious_cards:
            if row['chargeback_count'] > 0:
                log.error(
                    f"Suspicious card with chargebacks: {row['card_token'][:8]}..., "
                    f"customers: {row['customer_count']}, chargebacks: {row['chargeback_count']}"
                )
                
                await self._block_card(row['card_token'])
        
        # 3. Geographic anomaly (payment from USA, chargeback from EU)
        # Требует tracking location, но вы поняли идею...
    
    async def _flag_customer_for_review(self, customer_id: str):
        """Отметь клиента для ручной проверки."""
        await self.db.execute("""
            UPDATE customers
            SET fraud_risk_level = 'high'
            WHERE id = %s
        """, (customer_id,))
    
    async def _block_card(self, card_token: str):
        """Заблокируй карту."""
        await self.db.execute("""
            UPDATE blocked_cards
            SET status = 'blocked'
            WHERE card_token = %s
        """, (card_token,))
```

**Запись chargeback в реестр:**
```python
async def record_chargeback(
    self,
    payment_id: str,
    chargeback_amount: int,
    reason: str
):
    """
    Запиши chargeback в реестр (отрицательная запись).
    """
    payment = await self.db.get_payment(payment_id)
    
    async with self.db.transaction():
        # Три записи (как при платеже, но в обратном направлении):
        
        # 1. Кредит клиенту (деньги вернулись)
        await self.db.insert_ledger_entry(
            transaction_id=payment_id,
            account_id=customer_account_id,
            amount=payment.amount,  # клиент получает обратно
            entry_type='chargeback'
        )
        
        # 2. Дебет с продавца (он теряет деньги)
        await self.db.insert_ledger_entry(
            transaction_id=payment_id,
            account_id=merchant_account_id,
            amount=-payment.amount,  # продавец теряет
            entry_type='chargeback'
        )
        
        # 3. Дебет с платформы (штраф от Stripe)
        await self.db.insert_ledger_entry(
            transaction_id=payment_id,
            account_id=stripe_fee_account_id,
            amount=-1500,  # штраф $15
            entry_type='chargeback_fee'
        )
        
        # Проверяем баланс
        # дебеты (-100, -1500) = кредиты (+100) ✓
```

**Типичные ошибки.**
- Игнорировать webhook'и от провайдера → не заметить chargeback.
- Не учитывать штраф в реестре → баланс не сходится.
- Не анализировать паттерны → пропустить мошенников.
- Не предоставить механизм для подачи доказательств → неправильно проиграть.

**На интервью.**
- Объясни жизненный цикл chargeback.
- Упомяни webhook и асинхронную обработку.
- Уточняющий вопрос: «Как уменьшить chargebacks?» → Fraud detection, customer verification, clear refund policy.

---

### 10. Как протестировать платёжную систему без реальных платежей?

**Зачем спрашивают.** Тестирование платежей сложно. Используешь реальные карты → дорого. Полагаешься на provider's test mode. Это проверяет практическое мышление о тестировании critical systems.

**Короткий ответ.** Используй test mode от Stripe/PayPal (test API keys, test cards). Mock провайдер для unit/integration tests. Staging окружение с test БД. E2E тесты на staging перед production.

**Детальный разбор.**

**Уровни тестирования платежей:**
```
┌──────────────────────────────────────────────────┐
│  Unit Tests (fast, local)                        │
│  - Test business logic (fee calculation, etc)    │
│  - Mock external providers                       │
│  - Run in milliseconds                           │
└──────────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│  Integration Tests (slower, test container)      │
│  - Test with real Redis, PostgreSQL             │
│  - Test idempotency, retry logic                │
│  - Test interactions between components         │
│  - Run in seconds                               │
└──────────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│  Staging Tests (Stripe test mode API keys)       │
│  - Test with actual Stripe sandbox               │
│  - Test webhooks, refunds, chargebacks          │
│  - Test payment routing, failover               │
│  - Run against staging DB replica               │
└──────────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────┐
│  Canary Deployment (small % of real traffic)     │
│  - 5% of prod traffic to new version            │
│  - Monitor error rates, latency                 │
│  - If good: ramp up to 100%                     │
└──────────────────────────────────────────────────┘
```

**Unit тесты с mock'ами:**
```python
import pytest
from unittest.mock import AsyncMock, patch

class TestPaymentProcessor:
    
    @pytest.fixture
    def processor(self):
        """Create processor with mocked gateway."""
        mock_gateway = AsyncMock()
        return PaymentProcessor(gateway=mock_gateway)
    
    @pytest.mark.asyncio
    async def test_process_payment_success(self, processor):
        """Test successful payment."""
        # Arrange
        processor.gateway.process_payment = AsyncMock(
            return_value=PaymentResult(
                id="pay_123",
                status="succeeded",
                external_id="ch_123"
            )
        )
        
        # Act
        result = await processor.process_payment(
            customer_id="cust_123",
            amount=10000,  # $100
            currency="USD",
            token="tok_visa_4242",
            idempotency_key="key_123"
        )
        
        # Assert
        assert result.status == "succeeded"
        assert result.id == "pay_123"
        processor.gateway.process_payment.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_process_payment_with_idempotency(self, processor):
        """Test idempotency (same key returns same result)."""
        # Arrange
        result1 = PaymentResult(id="pay_123", status="succeeded")
        processor.gateway.process_payment = AsyncMock(return_value=result1)
        
        # Act
        r1 = await processor.process_payment(
            customer_id="cust_123",
            amount=10000,
            currency="USD",
            token="tok_visa_4242",
            idempotency_key="key_abc"
        )
        
        r2 = await processor.process_payment(
            customer_id="cust_123",
            amount=10000,
            currency="USD",
            token="tok_visa_4242",
            idempotency_key="key_abc"  # same key
        )
        
        # Assert
        assert r1.id == r2.id
        # Gateway should be called only once (second call cached)
        assert processor.gateway.process_payment.call_count == 1
    
    @pytest.mark.asyncio
    async def test_payment_retry_on_transient_error(self, processor):
        """Test retry on transient errors."""
        # Arrange
        processor.gateway.process_payment = AsyncMock(
            side_effect=[
                ConnectionError("timeout"),  # первый вызов
                ConnectionError("timeout"),  # второй вызов
                PaymentResult(id="pay_123", status="succeeded")  # третий успех
            ]
        )
        
        # Act
        with patch('asyncio.sleep'):  # skip actual sleep
            result = await processor.process_payment_with_retry(
                customer_id="cust_123",
                amount=10000,
                currency="USD",
                token="tok_visa_4242",
                idempotency_key="key_123"
            )
        
        # Assert
        assert result.status == "succeeded"
        assert processor.gateway.process_payment.call_count == 3
    
    @pytest.mark.asyncio
    async def test_payment_fails_on_permanent_error(self, processor):
        """Test that permanent errors don't retry."""
        # Arrange
        processor.gateway.process_payment = AsyncMock(
            side_effect=InvalidCardError("card declined")
        )
        
        # Act & Assert
        with pytest.raises(InvalidCardError):
            await processor.process_payment(...)
        
        # Should fail immediately (no retries)
        assert processor.gateway.process_payment.call_count == 1
    
    @pytest.mark.asyncio
    async def test_fee_calculation(self):
        """Test correct fee calculation (2.9% + $0.30)."""
        amount = 10000  # $100
        
        fee_amount = int(amount * 0.029 + 30)
        
        assert fee_amount == 3030  # $30.30
```

**Integration тесты с real Redis/PostgreSQL:**
```python
import pytest
import testcontainers.postgres as postgres
import testcontainers.redis as redis

class TestPaymentIntegration:
    
    @pytest.fixture(scope="session")
    def postgres_container(self):
        """Spin up test PostgreSQL."""
        with postgres.PostgresContainer(
            image="postgres:14",
            username="test",
            password="test",
            dbname="testdb"
        ) as container:
            yield container
    
    @pytest.fixture(scope="session")
    def redis_container(self):
        """Spin up test Redis."""
        with redis.RedisContainer() as container:
            yield container
    
    @pytest.fixture
    async def payment_service(self, postgres_container, redis_container):
        """Create payment service with real deps."""
        db = await create_db_connection(
            host=postgres_container.get_container_host_ip(),
            port=postgres_container.get_exposed_port(5432),
            database="testdb"
        )
        
        cache = await create_redis_connection(
            host=redis_container.get_container_host_ip(),
            port=redis_container.get_exposed_port(6379)
        )
        
        gateway = AsyncMock()  # still mock external provider
        
        return PaymentService(db=db, cache=cache, gateway=gateway)
    
    @pytest.mark.asyncio
    async def test_payment_idempotency_with_real_db(self, payment_service):
        """Test idempotency with real PostgreSQL."""
        # Arrange
        payment_service.gateway.process_payment = AsyncMock(
            return_value=PaymentResult(id="pay_123", status="succeeded")
        )
        
        # Act
        r1 = await payment_service.process_payment(
            customer_id="cust_123",
            amount=10000,
            currency="USD",
            token="tok_visa_4242",
            idempotency_key="key_abc"
        )
        
        r2 = await payment_service.process_payment(
            customer_id="cust_123",
            amount=10000,
            currency="USD",
            token="tok_visa_4242",
            idempotency_key="key_abc"
        )
        
        # Assert
        assert r1.id == r2.id
        # Проверяем в БД: только одна запись платежа
        payments = await payment_service.db.query(
            "SELECT * FROM payments WHERE idempotency_key = %s",
            ("key_abc",)
        )
        assert len(payments) == 1
    
    @pytest.mark.asyncio
    async def test_ledger_balance_consistency(self, payment_service):
        """Test that ledger always balances."""
        # Process several payments
        for i in range(5):
            await payment_service.process_payment(
                customer_id=f"cust_{i}",
                amount=10000,
                currency="USD",
                token=f"tok_{i}",
                idempotency_key=f"key_{i}"
            )
        
        # Check ledger balance
        result = await payment_service.db.query("""
            SELECT SUM(CASE WHEN amount > 0 THEN amount ELSE 0 END) as credits,
                   SUM(CASE WHEN amount < 0 THEN -amount ELSE 0 END) as debits
            FROM ledger_entries
        """)
        
        assert result[0]['credits'] == result[0]['debits'], \
            "Ledger is unbalanced!"
```

**Staging тесты с Stripe test mode:**
```python
import pytest
import stripe

# Use test API key
stripe.api_key = os.getenv("STRIPE_TEST_SECRET_KEY")

class TestPaymentStaging:
    """
    Тесты с реальным Stripe API (в test mode).
    """
    
    TEST_CARDS = {
        'visa_success': '4242424242424242',
        'visa_decline': '4000000000000002',
        'amex_success': '378282246310005',
        'requires_3d_secure': '4000002500003155',
    }
    
    @pytest.mark.asyncio
    async def test_successful_charge(self):
        """Test successful charge with test card."""
        token = stripe.Token.create(
            card={
                "number": self.TEST_CARDS['visa_success'],
                "exp_month": 12,
                "exp_year": 2025,
                "cvc": "123"
            }
        )
        
        charge = stripe.Charge.create(
            amount=10000,
            currency="usd",
            source=token.id,
            idempotency_key="test_key_success"
        )
        
        assert charge.status == "succeeded"
        assert charge.amount == 10000
    
    @pytest.mark.asyncio
    async def test_declined_card(self):
        """Test that declined card is handled."""
        token = stripe.Token.create(
            card={
                "number": self.TEST_CARDS['visa_decline'],
                "exp_month": 12,
                "exp_year": 2025,
                "cvc": "123"
            }
        )
        
        with pytest.raises(stripe.error.CardError) as exc_info:
            stripe.Charge.create(
                amount=10000,
                currency="usd",
                source=token.id
            )
        
        assert exc_info.value.http_status == 402
    
    @pytest.mark.asyncio
    async def test_refund(self):
        """Test refund processing."""
        # Create charge
        charge = stripe.Charge.create(
            amount=10000,
            currency="usd",
            source="tok_visa",
            idempotency_key="test_refund_charge"
        )
        
        # Refund
        refund = stripe.Refund.create(
            charge=charge.id,
            idempotency_key="test_refund_1"
        )
        
        assert refund.status == "succeeded"
        assert refund.amount == 10000
    
    @pytest.mark.asyncio
    async def test_webhook_signature_verification(self):
        """Test webhook signature validation."""
        webhook_secret = os.getenv("STRIPE_TEST_WEBHOOK_SECRET")
        
        payload = json.dumps({
            "type": "charge.succeeded",
            "data": {"object": {"id": "ch_123"}},
            "created": int(time.time())
        })
        
        signature = stripe.WebhookEndpoint.create_signature(
            payload=payload,
            secret=webhook_secret
        )
        
        # Verify signature
        event = stripe.Webhook.construct_event(
            payload=payload,
            sig_header=signature,
            secret=webhook_secret
        )
        
        assert event['type'] == "charge.succeeded"
```

**Chaos testing (проверка отказоустойчивости):**
```python
class TestPaymentChaos:
    """
    Chaos testing: что происходит при сбоях?
    """
    
    @pytest.mark.asyncio
    async def test_payment_with_db_timeout(self, payment_service):
        """Test behavior when DB times out."""
        with patch.object(
            payment_service.db,
            'execute',
            side_effect=asyncio.TimeoutError()
        ):
            with pytest.raises(asyncio.TimeoutError):
                await payment_service.process_payment(...)
    
    @pytest.mark.asyncio
    async def test_payment_with_provider_down(self, payment_service):
        """Test failover when Stripe is down."""
        payment_service.gateway.process_payment = AsyncMock(
            side_effect=ConnectionError("Stripe unavailable")
        )
        
        # Should try failover providers
        # (depends on your architecture)
        result = await payment_service.process_payment(...)
        assert result is not None  # failover succeeded
    
    @pytest.mark.asyncio
    async def test_concurrent_payments(self, payment_service):
        """Test race conditions with concurrent payments."""
        tasks = [
            payment_service.process_payment(
                customer_id="cust_123",
                amount=10000,
                currency="USD",
                token="tok_visa",
                idempotency_key=f"key_{i}"
            )
            for i in range(50)  # 50 concurrent
        ]
        
        results = await asyncio.gather(*tasks)
        
        # All should succeed
        assert all(r.status == "succeeded" for r in results)
        
        # All should be unique payments
        assert len(set(r.id for r in results)) == 50
```

**Типичные ошибки.**
- Тестировать на реальных картах → дорого, медленно, опасно.
- Забыть про тестирование идемпотентности → miss race conditions.
- Не тестировать edge cases (partial refunds, chargebacks) → bugs в production.
- Не использовать test containers → непостоянные тесты.

**На интервью.**
- Объясни разные уровни тестирования (unit → integration → staging).
- Упомяни test mode от Stripe и как его использовать.
- Уточняющий вопрос: «Как тестировать webhook'и локально?» → Stripe CLI для forwarding webhooks в localhost.

---

## Резюме / Ключевые моменты

### Финансовые системы требуют:
1. **Идемпотентность** — никогда не повторять платёж
2. **Консистентность** — баланс всегда сходится (двойная запись)
3. **Надёжность** — failover, retry, reconciliation
4. **Безопасность** — PCI DSS, никогда не хранить полный номер карты
5. **Масштабируемость** — асинхронность, шардинг, кэширование
6. **Аудит** — логировать все операции для compliance

### Архитектурные паттерны:
- Tokenization вместо хранения карт
- Двойная запись в реестре для консистентности
- Асинхронная обработка для скейлирования
- Circuit Breaker для failover
- Exponential backoff + jitter для retry
- Webhook'и для асинхронных уведомлений

---

## См. также

- [Транзакции в базах данных](../06-databases/03-transactions.md) — ACID-гарантии
- [Распределённые транзакции](../07-distributed-systems/07-distributed-transactions.md) — Saga и Two-Phase Commit
- [Message Queues](../08-architecture/04-message-queues.md) — асинхронная обработка
- [Кэширование](../08-architecture/05-caching.md) — Redis, TTL

---

← [09-video-streaming](./09-video-streaming.md) | [Трек System Design](./README.md) | [11-booking-system](./11-booking-system.md) →
