# Распределённые транзакции и Saga паттерн

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [16-consensus-raft](./16-consensus-raft.md) · Следующая тема: [18-consistency-models](./18-consistency-models.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Saga** — паттерн асинхронного выполнения распределённой транзакции через цепочку локальных транзакций, каждая из которых выполняется в отдельном микросервисе. Позволяет обновлять несколько микросервисов и поддерживать консистентность при невозможности использовать традиционные ACID транзакции, которые требуют единого центрального координатора.

**2PC (Two-Phase Commit)** — протокол координации для гарантирования атомарности между несколькими участниками или базами данных. Фаза 1 (prepare) подготавливает и блокирует ресурсы, фаза 2 (commit/abort) окончательно фиксирует или откатывает изменения. Это классический подход, но он имеет серьёзные ограничения в масштабируемости.

**Compensation** — операция отката или обратная операция, которая отменяет эффект предыдущей операции в Saga паттерне. Если какой-то шаг Saga не удался, система запускает compensation шаги в обратном порядке, чтобы отменить уже выполненные действия.

**Orchestration** — стиль координации, где один центральный сервис (оркестратор) координирует все шаги распределённой транзакции и принимает решения. Проще отследить логику выполнения, но центральный сервис становится bottleneck'ом и точкой отказа.

**Choreography** — децентрализованный стиль координации, где каждый микросервис слушает события и сам решает, какой шаг выполнить дальше, основываясь на событиях от других сервисов. Децентрализованно и независимо, но намного сложнее отследить логику и отладить система.

**Idempotency** — свойство операции, при котором выполнение её многократно даёт точно такой же результат, как и выполнение один раз. Это критично в распределённых системах, чтобы retry операций не создавал дубликаты и не нарушал консистентность данных.

**Eventual Consistency** — модель консистентности, гарантирующая, что если не будет новых обновлений, все реплики рано или поздно придут к одному и тому же состоянию. Позволяет системе быть доступной и масштабируемой даже при сетевых разделениях или задержках репликации.

**CAP Theorem** — фундаментальная теорема, утверждающая, что невозможно одновременно гарантировать все три свойства: Consistency (консистентность), Availability (доступность) и Partition Tolerance (устойчивость к разделениям сети). Показывает, что в распределённых системах нужно выбирать, какие требования приносить в жертву.

**Atomicity** — ACID свойство "всё или ничего", означающее, что либо вся транзакция выполнится полностью, либо откатится полностью без частичных изменений. В распределённых системах это очень сложно гарантировать без центрального координатора и синхронного взаимодействия.

**ACID** — аббревиатура четырёх свойств надёжной локальной транзакции: Atomicity, Consistency, Isolation, Durability. В распределённых системах приходится жертвовать частью этих гарантий в пользу масштабируемости и доступности, что привело к появлению BASE моделей.

**Blocking** — ситуация, когда ресурсы заблокированы для других операций, что предотвращает их использование. В протоколе 2PC блокирование может привести к deadlock'ам между транзакциями и значительному снижению throughput'а системы.

---

## 1. Почему ACID транзакции не работают в распределённых системах?

### Зачем спрашивают
Это фундаментальный вопрос, который показывает понимание различий между монолитными БД и микросервисной архитектурой. Интервьюер проверяет, знаете ли вы ограничения ACID в сетевых системах.

### Короткий ответ
ACID транзакции требуют синхронного доступа к общему состоянию, что невозможно в распределённых системах из-за сетевых задержек и вероятности отказов. CAP-теорема показывает, что нельзя гарантировать одновременно Consistency, Availability и Partition tolerance.

### Детальный разбор

**Почему ACID не масштабируется:**

```
Монолитная БД (ACID работает):
┌─────────────────────────────────┐
│  Единая точка доступа           │
│  - Синхронные блокировки        │
│  - Общее состояние              │
│  - Атомарность гарантирована    │
└─────────────────────────────────┘

Распределённая система (ACID нарушается):
┌──────────────────┐     ┌──────────────────┐
│   Сервис 1       │     │   Сервис 2       │
│  (Бухгалтерия)   │     │   (Платежи)      │
└────────┬─────────┘     └────────┬─────────┘
         │ Сетевая задержка       │
         │◄────────────────────►│
    Два независимых      Невозможно
    состояния            синхронизировать
```

**Проблемы ACID в распределённых системах:**

| Проблема | ACID решение | Распределённая система |
|---------|-------------|----------------------|
| **Atomicity** | Всё или ничего в одной БД | Откаты на разных сервисах не синхронны |
| **Consistency** | Блокировки обеспечивают | CAP: выбираем Availability |
| **Isolation** | Уровни изоляции (MVCC) | Сложно с разными СУБД |
| **Durability** | WAL (Write-Ahead Log) | Разные механизмы в разных БД |

**CAP-теорема применительно к транзакциям:**

```
      ┌─────────────────────┐
      │  CAP Theorem        │
      └────────┬────────────┘
              ╱ ╲ ╲
             ╱   ╲ ╲
    ┌─────C─────┐ ┌─────P─────┐
    │Consistency│ │ Partition │
    │           │╱│ Tolerance │
    └───────────╱ └───────────┘
         │         │
         └────┬────┘
              │
           ┌──A──┐
           │Avail│
           └─────┘

В распределённых системах: выбираем AP, жертвуем C
Транзакции → выбираем CP или CA, теряем масштабируемость
```

### Пример

```python
# ❌ Невозможное: попытка ACID транзакции в микросервисах

def transfer_money(account_id_from, account_id_to, amount):
    """Попытка ACID транзакции между двумя сервисами"""
    try:
        # Проблема: нет единой транзакции!
        response1 = requests.post(
            'http://billing-service/debit',
            json={'account': account_id_from, 'amount': amount}
        )
        # Если здесь сеть порвётся, что дальше?

        response2 = requests.post(
            'http://payment-service/credit',
            json={'account': account_id_to, 'amount': amount}
        )
        return 'success'
    except Exception as e:
        # Как откатить первый платёж?
        # Могли бы зафиксировать только первый
        return 'failed'

# ✅ Правильно: асинхронная согласованность с Saga

class DistributedTransaction:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.state = {}

    def transfer_money_saga(self, from_acc, to_acc, amount):
        """Saga для перевода денег"""
        # Начало: попытаемся дебетировать
        self.state['transaction_id'] = uuid.uuid4()

        # Шаг 1: дебет
        event = {
            'type': 'MoneyDebitRequested',
            'account': from_acc,
            'amount': amount,
            'transaction_id': self.state['transaction_id']
        }
        self.event_bus.publish(event)

        # Ожидаем подтверждения (асинхронно)
        # Если ошибка - опубликуем компенсирующее событие

# Обработчик события (на микросервисе)
def handle_debit_requested(event):
    try:
        result = billing_service.debit(
            event['account'],
            event['amount']
        )
        # Публикуем успех
        event_bus.publish({
            'type': 'MoneyDebited',
            'transaction_id': event['transaction_id'],
            'status': 'success'
        })
    except InsufficientFundsError:
        # Публикуем ошибку
        event_bus.publish({
            'type': 'MoneyDebitFailed',
            'transaction_id': event['transaction_id'],
            'reason': 'insufficient_funds'
        })
```

### Типичные ошибки

1. **Попытка синхронных ACID между микросервисами** — приводит к deadlock и cascade failures
2. **Игнорирование сетевых отказов** — забывают обработать timeout
3. **Смешивание локальных и распределённых транзакций** — путают уровни абстракции

### На интервью

- Объясните, почему вы не можете использовать традиционные транзакции в облаке
- Упомяните, что ACID возможна в пределах одного сервиса (локальная БД)
- Говорите о "eventual consistency" — это норма в распределённых системах
- **Уточняющий вопрос:** "А как вы гарантируете целостность данных?" → Saga, Event Sourcing

---

## 2. Что такое Two-Phase Commit (2PC)?

### Зачем спрашивают
Классический паттерн координации в распределённых системах. Нужно знать как теорию, так и практические ограничения.

### Короткий ответ
2PC — это протокол координации между несколькими участниками для выполнения атомарной транзакции. Координатор спрашивает всех участников "готовы ли вы?" (Phase 1), затем командует "коммитьте!" или "откатывайте!" (Phase 2).

### Детальный разбор

**Архитектура 2PC:**

```
Фаза 1: PREPARE (голосование)

  Координатор              Участник 1         Участник 2
      │                       │                   │
      │─────prepare───────────►│─────prepare────►│
      │                       │                   │
      │                    Блокирует        Блокирует
      │                    ресурсы          ресурсы
      │                       │                   │
      │◄────yes/no────────────│◄────yes/no────────│
      │                       │                   │

Фаза 2: COMMIT/ROLLBACK (исполнение)

  Координатор              Участник 1         Участник 2
      │                       │                   │
      │─────commit────────────►│─────commit────►│
      │                       │                   │
      │                    Коммитим         Коммитим
      │                       │                   │
      │◄────ack────────────────│◄────ack─────────│
```

**Состояния участников в 2PC:**

```
Состояние жизненного цикла транзакции:

┌─────────────────┐
│    INITIAL      │  Ничего не делали
└────────┬────────┘
         │ Начало транзакции
         ▼
┌─────────────────┐
│  PREPARE_SENT   │  Отправили prepare, ждём ответов
└────────┬────────┘
         │
    ┌────┴────────────────┐
    │                     │
    ▼                     ▼
┌──────────┐         ┌──────────┐
│COMMIT    │         │ROLLBACK  │  Если кто-то сказал "нет"
│DECIDED   │         │DECIDED   │
└────┬─────┘         └────┬─────┘
     │ все согласны       │ откатываем
     ▼                    ▼
┌─────────────────┐ ┌─────────────────┐
│  COMMITTING     │ │  ABORTING       │
└────────┬────────┘ └────────┬────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐ ┌─────────────────┐
│  COMMITTED      │ │  ABORTED        │
└─────────────────┘ └─────────────────┘
```

**Проблемы 2PC:**

```
Проблема 1: BLOCKING (блокировка ресурсов)
┌─────────────────────────────────────┐
│ Prepare Phase (ресурсы заблокированы)│
│ ├─ Участник 1: блокирует счёт       │
│ ├─ Участник 2: ждёт ответа         │
│ ├─ Координатор: сеть порвалась!     │
│ └─ DEADLOCK на Участнике 1          │
└─────────────────────────────────────┘

Проблема 2: SYNCHRONOUS (синхронность)
Если Участник 2 медленный:
  → Координатор ждёт
  → Участник 1 держит блокировку
  → Все остальные транзакции ждут
  → Throughput падает

Проблема 3: SPLIT BRAIN (если координатор упал)
  Фаза 1 завершена: все готовы, ресурсы заблокированы
  Координатор ПАДАЕТ
  Участники: что делать? Коммитить или откатить?
  → Нужен новый координатор или timeout
```

### Пример

```python
import logging
from enum import Enum
from typing import List, Dict

class TransactionState(Enum):
    INITIAL = "initial"
    PREPARE_SENT = "prepare_sent"
    COMMIT_DECIDED = "commit_decided"
    ROLLBACK_DECIDED = "rollback_decided"
    COMMITTED = "committed"
    ABORTED = "aborted"

class Participant:
    """Участник в 2PC"""
    def __init__(self, name: str):
        self.name = name
        self.prepared = False
        self.committed = False

    def prepare(self, transaction_data) -> bool:
        """Фаза 1: подготовка"""
        logging.info(f"[{self.name}] Preparing transaction...")
        try:
            # Проверяем возможность выполнить
            # Блокируем ресурсы
            self.prepared = True
            logging.info(f"[{self.name}] Ready to commit")
            return True
        except Exception as e:
            logging.error(f"[{self.name}] Cannot prepare: {e}")
            return False

    def commit(self) -> bool:
        """Фаза 2: коммит"""
        if not self.prepared:
            return False
        try:
            logging.info(f"[{self.name}] Committing...")
            # Выполняем изменения
            self.committed = True
            logging.info(f"[{self.name}] Committed")
            return True
        except Exception as e:
            logging.error(f"[{self.name}] Commit failed: {e}")
            return False

    def rollback(self) -> bool:
        """Откат"""
        logging.info(f"[{self.name}] Rolling back...")
        self.prepared = False
        self.committed = False
        # Освобождаем ресурсы
        return True

class TwoPhaseCommitCoordinator:
    """Координатор 2PC"""
    def __init__(self):
        self.participants: List[Participant] = []
        self.state = TransactionState.INITIAL
        self.tx_id = None

    def add_participant(self, participant: Participant):
        self.participants.append(participant)

    def execute_transaction(self, tx_id: str, tx_data) -> bool:
        """Выполнить распределённую транзакцию"""
        self.tx_id = tx_id
        self.state = TransactionState.PREPARE_SENT

        # ФАЗА 1: Prepare
        logging.info(f"[Coordinator] Phase 1: Prepare (TX: {tx_id})")
        votes = []
        for participant in self.participants:
            vote = participant.prepare(tx_data)
            votes.append(vote)
            logging.info(f"[Coordinator] {participant.name} voted: {vote}")

        # Проверяем все ли готовы
        if all(votes):
            # ФАЗА 2: Commit
            logging.info(f"[Coordinator] Phase 2: Commit")
            self.state = TransactionState.COMMIT_DECIDED

            results = []
            for participant in self.participants:
                result = participant.commit()
                results.append(result)

            success = all(results)
            self.state = TransactionState.COMMITTED if success else TransactionState.INITIAL
            return success
        else:
            # ФАЗА 2: Rollback
            logging.info(f"[Coordinator] Phase 2: Rollback (not all ready)")
            self.state = TransactionState.ROLLBACK_DECIDED

            for participant in self.participants:
                participant.rollback()

            self.state = TransactionState.ABORTED
            return False

# Пример использования
if __name__ == "__main__":
    coordinator = TwoPhaseCommitCoordinator()

    bank1 = Participant("Bank-1")
    bank2 = Participant("Bank-2")

    coordinator.add_participant(bank1)
    coordinator.add_participant(bank2)

    # Успешная транзакция
    print("=== Successful Transaction ===")
    result = coordinator.execute_transaction("TX-001", {"amount": 100})
    print(f"Result: {result}\n")

    # Неудачная транзакция (один участник отказал)
    print("=== Failed Transaction ===")
    coordinator2 = TwoPhaseCommitCoordinator()
    bank3 = Participant("Bank-3")
    bank4 = Participant("Bank-4")

    # Подделаем отказ bank4
    coordinator2.add_participant(bank3)
    coordinator2.add_participant(bank4)

    # Вручную отказываем
    bank4.prepare = lambda x: False

    result2 = coordinator2.execute_transaction("TX-002", {"amount": 50})
    print(f"Result: {result2}")
```

### Типичные ошибки

1. **Не учитывают timeout в Prepare фазе** — координатор может зависнуть
2. **Забывают про logging** — нельзя восстановиться после краша
3. **Используют для высоконагруженных систем** — 2PC нарушает масштабируемость

### На интервью

- Объясните две фазы: **Prepare** (вопрос) и **Commit/Rollback** (решение)
- Упомяните **блокировку ресурсов** — это главная проблема
- Скажите, что это используется в финансовых системах, но редко в облаке
- **Уточняющий вопрос:** "Почему не использовать 2PC везде?" → Масштабируемость

---

## 3. Что такое Saga паттерн?

### Зачем спрашивают
Альтернатива 2PC для масштабируемых систем. Это паттерн, который интервьюер ожидает услышать от разработчика микросервисной архитектуры.

### Короткий ответ
Saga — это последовательность локальных транзакций, координируемых через события или оркестратор. Каждый микросервис выполняет свою часть работы и публикует события. Если что-то не удаётся, выполняются компенсирующие действия в обратном порядке.

### Детальный разбор

**Saga против 2PC:**

```
2PC: Синхронный, блокирующий
┌──────────────────────────────────────┐
│ Фаза 1: Все готовы?                 │
│ ├─ Сервис 1: готов (блокирует)      │
│ ├─ Сервис 2: готов (блокирует)      │
│ └─ Сервис 3: готов (блокирует)      │
│ Фаза 2: Коммитим все                │
└──────────────────────────────────────┘

SAGA: Асинхронный, без блокировки
┌──────────────────────────────────────┐
│ Шаг 1: Сервис 1 выполнил → событие   │
│ Шаг 2: Сервис 2 услышал → выполнил   │
│ Шаг 3: Сервис 3 услышал → выполнил   │
│ Или: Сервис N упал → компенсация!   │
└──────────────────────────────────────┘
```

**Типы Saga:**

```
1. ORCHESTRATION (оркестратор управляет)

         ┌───────────────────┐
         │   ORCHESTRATOR    │
         │ (центральный мозг)│
         └────────┬──────────┘
         ┌────────┼──────────┐
         ▼        ▼          ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │ Service│ │ Service│ │ Service│
    │   1    │ │   2    │ │   3    │
    └────────┘ └────────┘ └────────┘

Оркестратор:
  1. Вызовет Сервис 1
  2. Когда ответит → вызовет Сервис 2
  3. Когда ответит → вызовет Сервис 3
  4. Если ошибка на шаге N → откатывает шаги N-1, N-2, ...

2. CHOREOGRAPHY (события управляют)

    ┌────────┐              ┌────────┐
    │Service1│─event1───────│Service2│
    └────────┘              └────┬───┘
                                 │event2
                            ┌────▼───┐
                            │Service3 │
                            └─────────┘

Каждый сервис:
  1. Слушает события
  2. Выполняет свою часть
  3. Публикует новое событие
  4. Другие слышат → делают свою часть
```

### Пример

```python
from enum import Enum
from typing import List, Dict
import uuid
from datetime import datetime

class SagaState(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"

class SagaStep:
    """Один шаг Saga"""
    def __init__(self, name: str, action, compensation):
        self.name = name
        self.action = action
        self.compensation = compensation
        self.executed = False
        self.result = None

    def execute(self) -> bool:
        try:
            self.result = self.action()
            self.executed = True
            return True
        except Exception as e:
            print(f"Step {self.name} failed: {e}")
            return False

    def compensate(self) -> bool:
        if not self.executed:
            return True
        try:
            self.compensation(self.result)
            self.executed = False
            return True
        except Exception as e:
            print(f"Compensation for {self.name} failed: {e}")
            return False

class OrchestratorSaga:
    """Оркестратор Saga (управляет последовательностью)"""
    def __init__(self, saga_id: str = None):
        self.saga_id = saga_id or str(uuid.uuid4())
        self.steps: List[SagaStep] = []
        self.state = SagaState.PENDING
        self.current_step = -1
        self.events: List[Dict] = []

    def add_step(self, step: SagaStep):
        self.steps.append(step)

    def execute(self) -> bool:
        """Выполнить Saga"""
        self.state = SagaState.IN_PROGRESS
        print(f"\n[Saga {self.saga_id}] Starting orchestrated saga...")

        for i, step in enumerate(self.steps):
            self.current_step = i
            print(f"\n  Step {i+1}/{len(self.steps)}: {step.name}")

            if not step.execute():
                print(f"  ✗ FAILED at step {i+1}")
                self._compensate_backward(i)
                self.state = SagaState.FAILED
                return False

            print(f"  ✓ Completed")
            self.events.append({
                'step': step.name,
                'status': 'completed',
                'timestamp': datetime.now().isoformat()
            })

        self.state = SagaState.COMPLETED
        print(f"\n[Saga {self.saga_id}] ✓ COMPLETED")
        return True

    def _compensate_backward(self, failed_index: int):
        """Откатить выполненные шаги в обратном порядке"""
        print(f"\n  Compensating {failed_index} steps...")
        self.state = SagaState.COMPENSATING

        for i in range(failed_index - 1, -1, -1):
            step = self.steps[i]
            print(f"    Compensating step {i+1}: {step.name}")
            if step.compensate():
                print(f"    ✓ Compensation done")
                self.events.append({
                    'step': step.name,
                    'status': 'compensated',
                    'timestamp': datetime.now().isoformat()
                })
            else:
                print(f"    ✗ Compensation failed")

class ChoreographySaga:
    """Хореография Saga (события управляют)"""
    def __init__(self):
        self.subscribers: Dict[str, List] = {}
        self.saga_instances: Dict[str, Dict] = {}

    def subscribe(self, event_type: str, handler):
        """Подписаться на событие"""
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(handler)

    def publish(self, event_type: str, data: Dict):
        """Опубликовать событие"""
        print(f"\n[Event] {event_type}")
        if event_type in self.subscribers:
            for handler in self.subscribers[event_type]:
                try:
                    handler(data)
                except Exception as e:
                    print(f"  Error in handler: {e}")
                    # Публикуем компенсирующее событие
                    self.publish("compensation_requested", data)

# Пример использования: Заказ с оплатой и доставкой

# === ORCHESTRATION SAGA ===
print("=" * 50)
print("ORCHESTRATION SAGA: Order Processing")
print("=" * 50)

def order_service_action():
    print("    → Order service: Creating order")
    return {"order_id": "ORD-001", "total": 100}

def order_service_compensation(result):
    print("    → Order service: Cancelling order")

def payment_service_action():
    print("    → Payment service: Processing payment")
    return {"payment_id": "PAY-001", "amount": 100}

def payment_service_compensation(result):
    print("    → Payment service: Refunding payment")

def shipping_service_action():
    print("    → Shipping service: Creating shipment")
    return {"shipment_id": "SHIP-001"}

def shipping_service_compensation(result):
    print("    → Shipping service: Cancelling shipment")

saga = OrchestratorSaga("ORDER-001")
saga.add_step(SagaStep("CreateOrder", order_service_action, order_service_compensation))
saga.add_step(SagaStep("ProcessPayment", payment_service_action, payment_service_compensation))
saga.add_step(SagaStep("CreateShipment", shipping_service_action, shipping_service_compensation))

# Успешный сценарий
saga.execute()

# === CHOREOGRAPHY SAGA ===
print("\n" + "=" * 50)
print("CHOREOGRAPHY SAGA: Event-Driven Order Processing")
print("=" * 50)

event_bus = ChoreographySaga()

def handle_order_created(event):
    print("    [OrderService] Order created: Processing payment")
    event_bus.publish("payment_requested", event)

def handle_payment_completed(event):
    print("    [PaymentService] Payment completed: Shipping order")
    event_bus.publish("shipment_requested", event)

def handle_shipment_created(event):
    print("    [ShippingService] Shipment created: Order complete!")

def handle_compensation(event):
    print(f"    [CompensationHandler] Compensation triggered for {event}")

event_bus.subscribe("order_created", handle_order_created)
event_bus.subscribe("payment_requested", handle_payment_completed)
event_bus.subscribe("shipment_requested", handle_shipment_created)
event_bus.subscribe("compensation_requested", handle_compensation)

# Запускаем событие
event_bus.publish("order_created", {"order_id": "ORD-002", "total": 100})
```

### Типичные ошибки

1. **Не учитывают идемпотентность** — событие может прийти дважды
2. **Забывают про мониторинг** — компенсация может висеть в памяти
3. **Смешивают Orchestration и Choreography** — нужна единая стратегия

### На интервью

- Скажите "Saga — это последовательность локальных транзакций"
- Упомяните два типа: Orchestration и Choreography
- Объясните компенсирующие действия (undo операции)
- **Уточняющий вопрос:** "Что лучше: Orchestration или Choreography?" → Зависит от сложности

---

## 4. Orchestration vs Choreography

### Зачем спрашивают
Это ключевой выбор архитектуры. Интервьюер хочет видеть, что вы понимаете trade-offs между централизованным управлением и распределённой автономией.

### Короткий ответ
**Orchestration** — один сервис управляет (как дирижёр). **Choreography** — сервисы сами реагируют на события (как танцоры). Orchestration проще для простых потоков, Choreography лучше для масштабируемости.

### Детальный разбор

**Сравнение:**

| Аспект | Orchestration | Choreography |
|--------|---------------|-------------|
| **Управление** | Центральный оркестратор | Распределённое |
| **Сложность** | Проще (видна вся логика) | Сложнее (логика разбросана) |
| **Масштабируемость** | Ограничена (bottleneck) | Лучше (децентрализованно) |
| **Отладка** | Легко (одно место) | Сложно (логика везде) |
| **Тестирование** | Проще | Сложнее (нужны все сервисы) |
| **Связанность** | Оркестратор зависит от всех | Слабая связанность |
| **Высокая нагрузка** | Оркестратор становится узким местом | Масштабируется лучше |

**Архитектурные различия:**

```
ORCHESTRATION (дирижёр управляет)

          Orchestrator Service
          (центр управления)
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
    Order   Payment  Shipping
    Service Service  Service

Оркестратор знает о всех сервисах и управляет их согласованностью


CHOREOGRAPHY (события управляют)

    Order Service
         │
      Order Created
         │
         ▼
    Payment Service ──Payment Processed──┐
                                          ▼
                                   Shipping Service

Каждый сервис реагирует на события других сервисов
```

**Масштабируемость:**

```
ORCHESTRATION: Узкое место на оркестраторе

При 1000 заказов/сек:
  ┌─────────────┐
  │Orchestrator │ ← 1000 запросов/сек (CPU 90%)
  └─────────────┘
       │
  ┌────┴────┐
  ▼         ▼
 Service1  Service2 (легко масштабировать)

CHOREOGRAPHY: Распределённая нагрузка

При 1000 заказов/сек:
  ┌─────────┐
  │Service 1│ ← 200 события/сек (CPU 30%)
  └────┬────┘
       │
       ▼ Order.Created
  ┌──────────┐
  │Service 2 │ ← 200 события/сек (CPU 30%)
  └────┬─────┘
       │
       ▼ Payment.Done
  ┌──────────┐
  │Service 3 │ ← 200 события/сек (CPU 30%)
  └──────────┘
```

### Пример

```python
# === ORCHESTRATION ===
class OrderOrchestrator:
    """Оркестратор управляет потоком заказа"""
    def __init__(self, order_service, payment_service, shipping_service):
        self.order_service = order_service
        self.payment_service = payment_service
        self.shipping_service = shipping_service

    def process_order(self, order_data) -> bool:
        print(f"[Orchestrator] Processing order: {order_data['id']}")

        # Шаг 1: Создаём заказ
        order = self.order_service.create(order_data)
        if not order:
            print("  ✗ Failed to create order")
            return False
        print(f"  ✓ Order created: {order['id']}")

        # Шаг 2: Обрабатываем платёж
        payment = self.payment_service.process_payment(order['id'], order['total'])
        if not payment:
            print("  ✗ Failed to process payment - rolling back order")
            self.order_service.cancel(order['id'])
            return False
        print(f"  ✓ Payment processed: {payment['id']}")

        # Шаг 3: Создаём доставку
        shipment = self.shipping_service.create_shipment(order['id'])
        if not shipment:
            print("  ✗ Failed to create shipment - refunding and cancelling")
            self.payment_service.refund(payment['id'])
            self.order_service.cancel(order['id'])
            return False
        print(f"  ✓ Shipment created: {shipment['id']}")

        return True

# === CHOREOGRAPHY ===
class EventBus:
    """Шина событий (publisher-subscriber)"""
    def __init__(self):
        self.subscribers = {}

    def subscribe(self, event_type, handler):
        if event_type not in self.subscribers:
            self.subscribers[event_type] = []
        self.subscribers[event_type].append(handler)

    def publish(self, event_type, data):
        print(f"\n  [Event] {event_type}: {data.get('id', '')}")
        if event_type in self.subscribers:
            for handler in self.subscribers[event_type]:
                handler(data)

class OrderService:
    def __init__(self, event_bus):
        self.event_bus = event_bus

    def create_order(self, order_data):
        print(f"  [OrderService] Creating order: {order_data['id']}")
        # Создаём заказ в БД
        event_data = {"id": order_data['id'], "total": order_data['total']}
        # Публикуем событие - остальные слышат
        self.event_bus.publish("OrderCreated", event_data)
        return event_data

class PaymentService:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        # Подписываемся на события заказа
        event_bus.subscribe("OrderCreated", self.handle_order_created)

    def handle_order_created(self, order_data):
        print(f"  [PaymentService] Processing payment for: {order_data['id']}")
        # Обрабатываем платёж
        payment_data = {"id": f"PAY-{order_data['id']}", "amount": order_data['total']}
        # Публикуем событие платежа - другие слышат
        self.event_bus.publish("PaymentProcessed", payment_data)

class ShippingService:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        # Подписываемся на события платежа
        event_bus.subscribe("PaymentProcessed", self.handle_payment_processed)

    def handle_payment_processed(self, payment_data):
        print(f"  [ShippingService] Creating shipment for: {payment_data['id']}")
        # Создаём доставку
        shipment_data = {"id": f"SHIP-{payment_data['id']}"}
        # Публикуем событие доставки
        self.event_bus.publish("ShipmentCreated", shipment_data)

# Демонстрация
print("\n" + "=" * 50)
print("ORCHESTRATION: Управляемый поток")
print("=" * 50)

class MockService:
    def create(self, data): return {"id": data['id']}
    def cancel(self, id): print(f"    Cancelled {id}")
    def process_payment(self, order_id, amount): return {"id": f"PAY-{order_id}"}
    def refund(self, payment_id): print(f"    Refunded {payment_id}")
    def create_shipment(self, order_id): return {"id": f"SHIP-{order_id}"}

orchestrator = OrderOrchestrator(MockService(), MockService(), MockService())
orchestrator.process_order({"id": "ORD-001", "total": 100})

print("\n" + "=" * 50)
print("CHOREOGRAPHY: Событийно-управляемый поток")
print("=" * 50)

event_bus = EventBus()
order_service = OrderService(event_bus)
payment_service = PaymentService(event_bus)
shipping_service = ShippingService(event_bus)

order_service.create_order({"id": "ORD-002", "total": 100})
```

### Типичные ошибки

1. **Orchestration везде** — становится bottleneck в масштабировании
2. **Choreography везде** — сложная отладка, нужно знать весь граф событий
3. **Смешивание** — непредсказуемое поведение

### На интервью

- Используйте таблицу сравнения выше
- Скажите о trade-offs: простота vs масштабируемость
- **На большой системе:** Choreography
- **На простых потоках:** Orchestration (легче понять)
- **Уточняющий вопрос:** "Как вы выбирали бы между ними?" → Начните с Orchestration, переходите на Choreography по мере роста

---

## 5. Компенсирующие транзакции

### Зачем спрашивают
Ключевой механизм откатов в Saga. Показывает, понимаете ли вы, как обеспечить данные-в-консистентность в распределённых системах.

### Короткий ответ
Компенсирующие транзакции (undo операции) выполняются в обратном порядке, если какой-то шаг Saga упал. Например, если платёж прошёл, но доставка не создалась, нужно вернуть деньги.

### Детальный разбор

**Структура компенсации:**

```
Нормальный поток:
  Шаг 1: Списаны деньги
  Шаг 2: Зарезервирован товар
  Шаг 3: Создана доставка

Если Шаг 3 упал:
  Компенсация 2: Отпущен товар
  Компенсация 1: Возврачены деньги
  Состояние = как и было

     Forward →
Step 1   Step 2   Step 3
│        │        │
│✓       │✓       │✗ FAIL
└──► Compensation starts
     ← Backward
    Comp 2  Comp 1
      │        │
      ✓        ✓
```

**Вызовы в парах:**

```
Transaction ↔ Compensation

1. Debit Account ↔ Credit Account (вернуть деньги)
2. Reserve Item ↔ Release Item (отпустить товар)
3. Create Shipment ↔ Cancel Shipment (отменить доставку)
4. Update Inventory ↔ Restore Inventory (восстановить инвентарь)
5. Send Email ↔ Send Cancel Email (отправить отмену)
```

### Пример

```python
from datetime import datetime
from typing import Callable, Any, List

class Transaction:
    """Транзакция с компенсацией"""
    def __init__(self, name: str, action: Callable, compensation: Callable):
        self.name = name
        self.action = action
        self.compensation = compensation
        self.executed = False
        self.result = None
        self.error = None

    def execute(self) -> bool:
        try:
            self.result = self.action()
            self.executed = True
            print(f"  ✓ {self.name}: SUCCESS")
            return True
        except Exception as e:
            self.error = str(e)
            print(f"  ✗ {self.name}: FAILED - {e}")
            return False

    def compensate(self) -> bool:
        if not self.executed:
            print(f"  - {self.name}: Not executed, skipping compensation")
            return True
        try:
            self.compensation(self.result)
            self.executed = False
            print(f"  ↶ {self.name}: COMPENSATED")
            return True
        except Exception as e:
            print(f"  ✗ {self.name}: Compensation FAILED - {e}")
            return False

class CompensatingSaga:
    """Saga с компенсирующими транзакциями"""
    def __init__(self, saga_id: str):
        self.saga_id = saga_id
        self.transactions: List[Transaction] = []
        self.completed_steps = 0
        self.failed_step = -1

    def add_transaction(self, transaction: Transaction):
        self.transactions.append(transaction)

    def execute(self) -> bool:
        print(f"\n[Saga {self.saga_id}] Starting with {len(self.transactions)} transactions")

        # Выполняем вперёд
        for i, tx in enumerate(self.transactions):
            print(f"\n  Step {i+1}/{len(self.transactions)}: {tx.name}")
            if not tx.execute():
                self.failed_step = i
                print(f"\n  ✗ FAILED at step {i+1}")

                # Откатываем назад
                print(f"\n  Starting compensation (rolling back {i} steps)...")
                self._compensate_backward(i)

                return False
            self.completed_steps = i + 1

        print(f"\n✓ [Saga {self.saga_id}] COMPLETED successfully")
        return True

    def _compensate_backward(self, failed_index: int):
        """Откатить транзакции в обратном порядке"""
        for i in range(failed_index - 1, -1, -1):
            tx = self.transactions[i]
            print(f"\n  Step {failed_index - i}: Compensate {tx.name}")
            tx.compensate()

# === ПРИМЕР: Покупка в интернет-магазине ===

# Моделируем микросервисы
class BillingService:
    def __init__(self):
        self.debits = {}

    def debit(self, account: str, amount: float) -> dict:
        """Списать деньги"""
        if amount <= 0:
            raise ValueError("Negative amount")
        tx_id = f"TX-{datetime.now().timestamp()}"
        self.debits[tx_id] = {"account": account, "amount": amount}
        print(f"      [BillingService] Debited ${amount} from {account}")
        return {"tx_id": tx_id, "amount": amount}

    def refund(self, tx_id: str):
        """Вернуть деньги"""
        if tx_id not in self.debits:
            raise ValueError(f"Transaction {tx_id} not found")
        tx = self.debits.pop(tx_id)
        print(f"      [BillingService] Refunded ${tx['amount']} to {tx['account']}")

class InventoryService:
    def __init__(self):
        self.reserved = {}

    def reserve_item(self, item_id: str, quantity: int) -> dict:
        """Зарезервировать товар"""
        if quantity <= 0:
            raise ValueError("Invalid quantity")
        reservation_id = f"RES-{datetime.now().timestamp()}"
        self.reserved[reservation_id] = {"item": item_id, "qty": quantity}
        print(f"      [InventoryService] Reserved {quantity}x {item_id}")
        return {"reservation_id": reservation_id}

    def release_item(self, reservation_id: str):
        """Отпустить товар"""
        if reservation_id not in self.reserved:
            raise ValueError(f"Reservation {reservation_id} not found")
        res = self.reserved.pop(reservation_id)
        print(f"      [InventoryService] Released {res['qty']}x {res['item']}")

class ShippingService:
    def __init__(self):
        self.shipments = {}

    def create_shipment(self, items: dict) -> dict:
        """Создать доставку"""
        if not items:
            raise ValueError("No items to ship")
        shipment_id = f"SHIP-{datetime.now().timestamp()}"
        self.shipments[shipment_id] = items
        print(f"      [ShippingService] Created shipment {shipment_id}")
        return {"shipment_id": shipment_id}

    def cancel_shipment(self, shipment_id: str):
        """Отменить доставку"""
        if shipment_id not in self.shipments:
            raise ValueError(f"Shipment {shipment_id} not found")
        self.shipments.pop(shipment_id)
        print(f"      [ShippingService] Cancelled shipment {shipment_id}")

# Инициализируем сервисы
billing = BillingService()
inventory = InventoryService()
shipping = ShippingService()

# === УСПЕШНЫЙ СЦЕНАРИЙ ===
print("=" * 60)
print("SUCCESS SCENARIO: All transactions succeed")
print("=" * 60)

saga_success = CompensatingSaga("ORDER-SUCCESS-001")

# Транзакция 1: Дебет счёта
saga_success.add_transaction(Transaction(
    "Debit Payment",
    lambda: billing.debit("customer@example.com", 100),
    lambda tx_id: billing.refund(tx_id["tx_id"])
))

# Транзакция 2: Резерв товара
saga_success.add_transaction(Transaction(
    "Reserve Inventory",
    lambda: inventory.reserve_item("ITEM-123", 1),
    lambda res_id: inventory.release_item(res_id["reservation_id"])
))

# Транзакция 3: Создание доставки
saga_success.add_transaction(Transaction(
    "Create Shipment",
    lambda: shipping.create_shipment({"ITEM-123": 1}),
    lambda shipment: shipping.cancel_shipment(shipment["shipment_id"])
))

saga_success.execute()

# === СЦЕНАРИЙ С ОШИБКОЙ ===
print("\n" + "=" * 60)
print("FAILURE SCENARIO: Shipment service fails")
print("=" * 60)

saga_failure = CompensatingSaga("ORDER-FAILURE-001")

# Транзакция 1: Дебет счёта (успешно)
saga_failure.add_transaction(Transaction(
    "Debit Payment",
    lambda: billing.debit("customer2@example.com", 50),
    lambda tx_id: billing.refund(tx_id["tx_id"])
))

# Транзакция 2: Резерв товара (успешно)
saga_failure.add_transaction(Transaction(
    "Reserve Inventory",
    lambda: inventory.reserve_item("ITEM-456", 2),
    lambda res_id: inventory.release_item(res_id["reservation_id"])
))

# Транзакция 3: Создание доставки (ОШИБКА)
saga_failure.add_transaction(Transaction(
    "Create Shipment",
    lambda: shipping.create_shipment(None),  # Ошибка: None items
    lambda shipment: shipping.cancel_shipment(shipment["shipment_id"])
))

result = saga_failure.execute()

print("\n" + "=" * 60)
print(f"Final result: {'SUCCESS' if result else 'FAILED with compensation'}")
print("=" * 60)
```

### Типичные ошибки

1. **Отсутствие идемпотентности** — компенсация может быть вызвана дважды
2. **Забывают про логирование** — сложно отследить что произошло
3. **Неправильный порядок отката** — откатывают не в обратном порядке

### На интервью

- Объясните: "Компенсирующая транзакция — это undo операция"
- Используйте пример: "Если платёж прошёл, но доставка упала → вернуть деньги"
- Упомяните **порядок отката: обратный порядок выполнения**
- **Уточняющий вопрос:** "Что если компенсация тоже упадёт?" → Нужен retry или manual intervention

---

## 6. Идемпотентность в Saga

### Зачем спрашивают
Критический аспект надёжности. Показывает, что вы понимаете проблемы сетей (duplicate messages).

### Короткий ответ
Идемпотентность означает, что одна и та же операция может быть выполнена несколько раз с одинаковым результатом. В Saga это критично, так как события могут доставляться дважды из-за сетевых ошибок.

### Детальный разбор

**Проблема дублирования:**

```
Нормальный поток:
  Payment Service отправляет "PaymentProcessed"
  Shipping Service получает и создаёт доставку
  ✓ Доставка создана 1 раз

С сетевой ошибкой (повторная доставка события):
  Payment Service отправляет "PaymentProcessed"
  Shipping Service получает и создаёт доставку #1
  Timeout: Payment Service не получил ack → повторно отправляет
  Shipping Service получает (дубликат!) и создаёт доставку #2
  ✗ Теперь 2 доставки для одного платежа!
```

**Как обеспечить идемпотентность:**

```
Подход 1: Дедупликация по ID события
  ┌─────────────────────────────────┐
  │ Событие:                        │
  │ {                               │
  │   "id": "EVT-12345",            │
  │   "type": "PaymentProcessed",   │
  │   "timestamp": 123456789        │
  │ }                               │
  └─────────────────────────────────┘
         ↓
  При получении события:
  1. Проверяем: есть ли EVT-12345 в БД?
  2. Если ДА → не обрабатываем (уже обработали)
  3. Если НЕТ → обрабатываем и записываем ID

Подход 2: Версионирование состояния
  ┌─────────────────────────────────┐
  │ Состояние:                      │
  │ {                               │
  │   "id": "ORDER-1",              │
  │   "status": "shipped",          │
  │   "version": 3                  │
  │ }                               │
  └─────────────────────────────────┘
         ↓
  При получении события:
  1. Переход: draft → pending (v1 → v2) ✓
  2. Повторный переход: draft → pending
     Проверяем: текущая версия = 2, не 1 → пропускаем
```

### Пример

```python
from typing import Dict, Set
from enum import Enum
import json

class OrderStatus(Enum):
    DRAFT = 1
    PAYMENT_PENDING = 2
    PAID = 3
    SHIPPED = 4
    DELIVERED = 5

class IdempotentSagaService:
    """Сервис с идемпотентностью через дедупликацию"""
    def __init__(self):
        self.processed_events: Set[str] = set()
        self.orders: Dict[str, Dict] = {}

    def handle_payment_processed(self, event: Dict) -> bool:
        """Обработать событие платежа (с дедупликацией по ID)"""
        event_id = event['id']

        # Проверка 1: Уже ли обработали это событие?
        if event_id in self.processed_events:
            print(f"  [DUPLICATE] Event {event_id} already processed, skipping")
            return True  # Возвращаем true, так как это "успешно"

        # Проверка 2: Выполняем действие
        order_id = event['order_id']
        if order_id not in self.orders:
            print(f"  [ERROR] Order {order_id} not found")
            return False

        order = self.orders[order_id]
        order['status'] = OrderStatus.PAID
        order['payment_id'] = event['payment_id']

        # Проверка 3: Записываем в "обработанные"
        self.processed_events.add(event_id)

        print(f"  ✓ Event {event_id} processed: {order_id} marked as PAID")
        return True

class VersionedSagaService:
    """Сервис с идемпотентностью через версионирование"""
    def __init__(self):
        self.orders: Dict[str, Dict] = {}

    def create_order(self, order_id: str) -> Dict:
        """Создать заказ (версия 0)"""
        order = {
            'id': order_id,
            'status': 'draft',
            'version': 0
        }
        self.orders[order_id] = order
        return order

    def handle_payment_completed(self, event: Dict) -> bool:
        """Обработать платёж с проверкой версии"""
        order_id = event['order_id']
        expected_version = event['expected_version']  # Какую версию ждём

        if order_id not in self.orders:
            print(f"  [ERROR] Order {order_id} not found")
            return False

        order = self.orders[order_id]
        current_version = order['version']

        # Проверка версии
        if current_version != expected_version:
            print(f"  [DUPLICATE/OUT_OF_ORDER] Event for version {expected_version}, "
                  f"but order is at version {current_version}, skipping")
            return True  # Идемпотентно: просто пропускаем

        # Выполняем переход состояния и увеличиваем версию
        order['status'] = 'paid'
        order['payment_id'] = event['payment_id']
        order['version'] += 1

        print(f"  ✓ {order_id}: v{current_version} → v{order['version']} (paid)")
        return True

    def handle_shipment_created(self, event: Dict) -> bool:
        """Обработать создание доставки с проверкой версии"""
        order_id = event['order_id']
        expected_version = event['expected_version']

        if order_id not in self.orders:
            return False

        order = self.orders[order_id]

        if order['version'] != expected_version:
            print(f"  [DUPLICATE] Event for version {expected_version}, "
                  f"but order is at version {order['version']}, skipping")
            return True

        order['status'] = 'shipped'
        order['shipment_id'] = event['shipment_id']
        order['version'] += 1

        print(f"  ✓ {order_id}: v{order['version']-1} → v{order['version']} (shipped)")
        return True

# === ДЕМОНСТРАЦИЯ ===

print("=" * 60)
print("IDEMPOTENCY WITH DEDUPLICATION")
print("=" * 60)

service = IdempotentSagaService()
service.orders['ORD-001'] = {'id': 'ORD-001', 'status': 'draft'}

# Событие платежа
event1 = {'id': 'EVT-100', 'order_id': 'ORD-001', 'payment_id': 'PAY-001'}

print("\nПервый раз (обработка):")
service.handle_payment_processed(event1)

print("\nДубликат (тот же event_id):")
service.handle_payment_processed(event1)

print("\nНовое событие (другой event_id):")
event2 = {'id': 'EVT-101', 'order_id': 'ORD-001', 'payment_id': 'PAY-002'}
service.handle_payment_processed(event2)

print("\n" + "=" * 60)
print("IDEMPOTENCY WITH VERSIONING")
print("=" * 60)

versionedService = VersionedSagaService()
versionedService.create_order('ORD-002')

print("\nПоследовательные события (в правильном порядке):")
event_pay = {'order_id': 'ORD-002', 'expected_version': 0, 'payment_id': 'PAY-001'}
versionedService.handle_payment_completed(event_pay)

event_ship = {'order_id': 'ORD-002', 'expected_version': 1, 'shipment_id': 'SHIP-001'}
versionedService.handle_shipment_created(event_ship)

print("\nПовторно отправили evento_pay (дубликат):")
versionedService.handle_payment_completed(event_pay)

print("\nПоследовательность нарушена (old event_ship):")
event_ship_old = {'order_id': 'ORD-002', 'expected_version': 1, 'shipment_id': 'SHIP-002'}
versionedService.handle_shipment_created(event_ship_old)

print(f"\nFinal order state: {versionedService.orders['ORD-002']}")
```

### Типичные ошибки

1. **Не учитывают дубликаты** — создают две доставки
2. **Неправильное хранение дедупликационных ID** — теряют при перезагрузке
3. **Забывают про TTL** — дедупликационный набор растёт бесконечно

### На интервью

- "Идемпотентность критична из-за дублирования сообщений"
- Два подхода: **дедупликация по ID** или **версионирование**
- Приведите пример: "Событие приходит дважды → создаём одну доставку, не две"
- **Уточняющий вопрос:** "Как очищать дедупликационный набор?" → TTL, например 24 часа

---

## 7. Обработка частичных отказов

### Зачем спрашивают
Реальная проблема в распределённых системах. Интервьюер проверяет, знаете ли вы, как система может зависнуть в "плохом" состоянии.

### Короткий ответ
Частичный отказ — когда часть операции выполнилась, а часть нет. Например, платёж прошёл, но доставку не смогли создать. Нужна компенсация для откатов.

### Детальный разбор

**Типы частичных отказов:**

```
1. Отказ в СЕРЕДИНЕ Saga (самый плохой)
   Выполнено: Step 1, Step 2 ✓
   Упало: Step 3 ✗
   Нужна компенсация: Comp 2, Comp 1

2. Отказ компенсации
   Выполнено: Step 1 ✓
   Упало: Step 2 ✗
   Компенсация: Comp 1 ✗ (тоже упала!)
   → Нужна ручная интервенция

3. Timeout (неизвестное состояние)
   Выполнено: Step 1 ✓
   Step 2: Timeout (не знаем, выполнилось или нет)
   Решение: Retry с идемпотентностью

4. Cascade failure (волновой отказ)
   Service 1 упал → Service 2 ждёт → Service 3 ждёт
   Весь поток стоит
```

**Стратегии обработки:**

```
┌──────────────────────────────────────────────┐
│ Partial Failure Recovery Strategies          │
├──────────────────────────────────────────────┤
│ 1. Automatic Compensation                    │
│    ├─ Откатываем все успешные шаги           │
│    ├─ Надежнее всего                         │
│    └─ Но требует идемпотентных компенсаций  │
│                                               │
│ 2. Retry with Exponential Backoff            │
│    ├─ Повторяем упавший шаг                  │
│    ├─ 1s, 2s, 4s, 8s, 16s...                │
│    └─ Хорош для временных сбоев             │
│                                               │
│ 3. Circuit Breaker                           │
│    ├─ Если сервис падает → дальше не ломим │
│    ├─ Даём ему время восстановиться          │
│    └─ Failfast вместо дикого retry           │
│                                               │
│ 4. Manual Intervention / Dead Letter Queue   │
│    ├─ Отправляем в очередь для человека      │
│    ├─ "Помоги разобраться"                  │
│    └─ Для сложных случаев                    │
└──────────────────────────────────────────────┘
```

### Пример

```python
import time
import random
from enum import Enum
from typing import Callable, Optional

class PartialFailureRecoveryStrategy(Enum):
    COMPENSATE = "compensate"
    RETRY = "retry"
    CIRCUIT_BREAKER = "circuit_breaker"
    DEAD_LETTER = "dead_letter"

class RetryableOperation:
    """Операция с повторами"""
    def __init__(self, name: str, max_retries: int = 3):
        self.name = name
        self.max_retries = max_retries
        self.attempt = 0
        self.last_error = None

    def execute_with_retry(self, action: Callable) -> bool:
        """Выполнить с повторами и exponential backoff"""
        for attempt in range(1, self.max_retries + 1):
            self.attempt = attempt
            try:
                print(f"    [{self.name}] Attempt {attempt}/{self.max_retries}")
                action()
                print(f"    ✓ Success")
                return True
            except Exception as e:
                self.last_error = str(e)
                if attempt < self.max_retries:
                    backoff = 2 ** (attempt - 1)  # 1s, 2s, 4s, 8s
                    print(f"    ✗ Failed: {e}")
                    print(f"    Retrying in {backoff}s...")
                    time.sleep(backoff)
                else:
                    print(f"    ✗ All {self.max_retries} attempts failed")
                    return False
        return False

class CircuitBreaker:
    """Circuit Breaker паттерн"""
    class State(Enum):
        CLOSED = "closed"      # Нормально, пропускаем запросы
        OPEN = "open"          # Сервис упал, блокируем запросы
        HALF_OPEN = "half_open"  # Пробуем восстановиться

    def __init__(self, failure_threshold: int = 3, timeout: int = 5):
        self.state = self.State.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None

    def call(self, action: Callable) -> bool:
        """Обезопасённый вызов через circuit breaker"""
        if self.state == self.State.OPEN:
            # Проверяем, прошло ли время для восстановления
            if time.time() - self.last_failure_time > self.timeout:
                print(f"    [CB] Transitioning to HALF_OPEN (trying recovery)")
                self.state = self.State.HALF_OPEN
            else:
                print(f"    [CB] OPEN - Failing fast, not calling service")
                return False

        try:
            action()
            if self.state == self.State.HALF_OPEN:
                print(f"    [CB] HALF_OPEN → CLOSED (service recovered!)")
                self.state = self.State.CLOSED
                self.failure_count = 0
            return True
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            print(f"    [CB] Failure {self.failure_count}/{self.failure_threshold}")

            if self.failure_count >= self.failure_threshold:
                self.state = self.State.OPEN
                print(f"    [CB] CLOSED → OPEN (too many failures)")
            return False

class ResilientSagaCoordinator:
    """Координатор Saga с устойчивостью к частичным отказам"""
    def __init__(self):
        self.completed_steps = []
        self.failed_steps = []
        self.dead_letter_queue = []
        self.circuit_breakers = {}

    def execute_with_strategies(self, steps: list) -> bool:
        """Выполнить Saga с различными стратегиями восстановления"""
        print(f"\n[Saga Coordinator] Starting with {len(steps)} steps")

        for i, step in enumerate(steps):
            print(f"\nStep {i+1}: {step['name']}")

            # Выбираем стратегию
            strategy = step.get('strategy', PartialFailureRecoveryStrategy.COMPENSATE)
            success = False

            if strategy == PartialFailureRecoveryStrategy.RETRY:
                success = self._execute_with_retry(step)
            elif strategy == PartialFailureRecoveryStrategy.CIRCUIT_BREAKER:
                success = self._execute_with_circuit_breaker(step)
            elif strategy == PartialFailureRecoveryStrategy.DEAD_LETTER:
                success = self._execute_with_dead_letter(step)
            else:  # COMPENSATE
                success = self._execute_with_compensation(step)

            if success:
                self.completed_steps.append(step['name'])
            else:
                self.failed_steps.append(step['name'])
                print(f"\n✗ FAILURE at step: {step['name']}")
                print(f"Executing compensation for {len(self.completed_steps)} steps...")
                self._compensate_all_backward()
                return False

        print(f"\n✓ [Saga] COMPLETED successfully")
        return True

    def _execute_with_retry(self, step: dict) -> bool:
        """Стратегия: Retry"""
        op = RetryableOperation(step['name'], max_retries=3)
        return op.execute_with_retry(step['action'])

    def _execute_with_circuit_breaker(self, step: dict) -> bool:
        """Стратегия: Circuit Breaker"""
        service_name = step.get('service', 'unknown')

        if service_name not in self.circuit_breakers:
            self.circuit_breakers[service_name] = CircuitBreaker()

        cb = self.circuit_breakers[service_name]
        return cb.call(step['action'])

    def _execute_with_dead_letter(self, step: dict) -> bool:
        """Стратегия: Dead Letter Queue"""
        try:
            step['action']()
            return True
        except Exception as e:
            print(f"    ✗ Error: {e}")
            print(f"    → Sending to Dead Letter Queue")
            self.dead_letter_queue.append({
                'step': step['name'],
                'error': str(e),
                'timestamp': time.time()
            })
            return False  # Не продолжаем, хотя попытались

    def _execute_with_compensation(self, step: dict) -> bool:
        """Стратегия: Compensation"""
        try:
            step['action']()
            return True
        except Exception as e:
            print(f"    ✗ Error: {e}")
            return False

    def _compensate_all_backward(self):
        """Откатить все успешные шаги"""
        for step_name in reversed(self.completed_steps):
            print(f"  ↶ Compensating: {step_name}")

# === ДЕМОНСТРАЦИЯ ===

print("=" * 60)
print("PARTIAL FAILURE RECOVERY")
print("=" * 60)

# Имитируем нестабильный сервис
failure_rate = 0.4  # 40% вероятность падения

def unstable_action(name: str) -> bool:
    if random.random() < failure_rate:
        raise Exception(f"{name} service temporarily unavailable")
    print(f"      {name} executed successfully")
    return True

# Сценарий 1: С Retry
print("\n### Scenario 1: RETRY STRATEGY")
coordinator1 = ResilientSagaCoordinator()

steps1 = [
    {'name': 'Payment', 'action': lambda: unstable_action("Payment"),
     'strategy': PartialFailureRecoveryStrategy.RETRY},
    {'name': 'Inventory', 'action': lambda: unstable_action("Inventory"),
     'strategy': PartialFailureRecoveryStrategy.RETRY},
]

coordinator1.execute_with_strategies(steps1)

# Сценарий 2: С Circuit Breaker
print("\n### Scenario 2: CIRCUIT BREAKER STRATEGY")
coordinator2 = ResilientSagaCoordinator()

# Имитируем сломанный сервис
def always_fail():
    raise Exception("Service is down")

steps2 = [
    {'name': 'Request1', 'action': always_fail,
     'service': 'payment-service',
     'strategy': PartialFailureRecoveryStrategy.CIRCUIT_BREAKER},
    {'name': 'Request2', 'action': always_fail,
     'service': 'payment-service',
     'strategy': PartialFailureRecoveryStrategy.CIRCUIT_BREAKER},
    {'name': 'Request3', 'action': always_fail,
     'service': 'payment-service',
     'strategy': PartialFailureRecoveryStrategy.CIRCUIT_BREAKER},
]

coordinator2.execute_with_strategies(steps2)

# Сценарий 3: С Dead Letter Queue
print("\n### Scenario 3: DEAD LETTER QUEUE STRATEGY")
coordinator3 = ResilientSagaCoordinator()

steps3 = [
    {'name': 'Step1', 'action': lambda: unstable_action("Step1"),
     'strategy': PartialFailureRecoveryStrategy.DEAD_LETTER},
    {'name': 'Step2', 'action': lambda: unstable_action("Step2"),
     'strategy': PartialFailureRecoveryStrategy.DEAD_LETTER},
]

coordinator3.execute_with_strategies(steps3)
print(f"\nDead Letter Queue: {len(coordinator3.dead_letter_queue)} items")
for item in coordinator3.dead_letter_queue:
    print(f"  - {item['step']}: {item['error']}")
```

### Типичные ошибки

1. **Игнорирование timeout** — ждут вечно
2. **Неправильный retry** — повторяют без backoff
3. **Забывают про логирование** — невозможно отследить историю

### На интервью

- "Частичный отказ — это когда часть транзакции выполнилась"
- Основные стратегии: **Retry, Circuit Breaker, Compensation**
- Приведите пример: "Платёж прошёл, доставка упала → откатываем платёж"
- **Уточняющий вопрос:** "Что делать, если компенсация тоже упала?" → Manual intervention

---

## 8. Saga для платёжной системы

### Зачем спрашивают
Конкретный пример использования. Показывает, как теория применяется в реальной системе.

### Короткий ответ
Платёжная Saga состоит из: валидация платежа → резервирование средств → обработка платежа → подтверждение. Каждый шаг может упасть, нужны компенсирующие откаты.

### Детальный разбор

**Архитектура платёжной системы:**

```
                    Payment Request
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Validate │  │ Reserve  │  │ Process  │
    │ Payment  │  │  Funds   │  │ Payment  │
    └────┬─────┘  └────┬─────┘  └────┬─────┘
         │             │             │
         ▼             ▼             ▼
    Result Event  Result Event  Result Event
         │             │             │
         └──────────────┼──────────────┘
                        ▼
                   Saga State Machine

States:
  INITIAL → VALIDATING → RESERVED → PROCESSING → COMPLETED
            ↓             ↓           ↓            ↓
            INVALID       FAILED      FAILED       SUCCESS
            (abort)       (refund)    (refund)     (confirm)
```

### Пример

```python
from enum import Enum
from dataclasses import dataclass
from typing import Dict, Optional
from datetime import datetime

class PaymentStatus(Enum):
    INITIAL = "initial"
    VALIDATING = "validating"
    FUNDS_RESERVED = "funds_reserved"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    ROLLED_BACK = "rolled_back"

@dataclass
class PaymentRequest:
    transaction_id: str
    user_id: str
    amount: float
    merchant_id: str
    card_token: str

@dataclass
class PaymentState:
    transaction_id: str
    status: PaymentStatus
    user_id: str
    amount: float
    merchant_id: str
    reserved_amount: float = 0.0
    payment_processor_id: Optional[str] = None
    created_at: str = ""
    updated_at: str = ""

class FraudDetectionService:
    """Сервис проверки мошенничества"""
    def validate(self, request: PaymentRequest) -> bool:
        print(f"    [FraudDetection] Validating {request.transaction_id}...")
        # Имитируем проверку (в реальности - ML модель)
        if request.amount > 10000:
            print(f"    ✗ Amount too high: ${request.amount}")
            return False
        print(f"    ✓ Fraud check passed")
        return True

class AccountService:
    """Сервис счётов (резервирование средств)"""
    def __init__(self):
        self.accounts = {}
        self.reservations = {}

    def add_account(self, user_id: str, balance: float):
        self.accounts[user_id] = balance

    def reserve_funds(self, user_id: str, amount: float, transaction_id: str) -> bool:
        print(f"    [Account] Reserving ${amount} for {user_id}...")

        if user_id not in self.accounts:
            print(f"    ✗ Account not found")
            return False

        available = self.accounts[user_id]
        if available < amount:
            print(f"    ✗ Insufficient funds: ${available} < ${amount}")
            return False

        # Блокируем средства
        self.accounts[user_id] -= amount
        self.reservations[transaction_id] = {
            'user_id': user_id,
            'amount': amount,
            'reserved_at': datetime.now()
        }
        print(f"    ✓ Funds reserved, remaining: ${self.accounts[user_id]}")
        return True

    def release_funds(self, transaction_id: str) -> bool:
        print(f"    [Account] Releasing funds for {transaction_id}...")

        if transaction_id not in self.reservations:
            print(f"    ✗ Reservation not found")
            return False

        res = self.reservations.pop(transaction_id)
        self.accounts[res['user_id']] += res['amount']
        print(f"    ✓ Funds released back to account")
        return True

class PaymentProcessorService:
    """Сервис обработки платежей (PSP - Payment Service Provider)"""
    def process(self, request: PaymentRequest) -> Optional[str]:
        print(f"    [PaymentProcessor] Processing payment {request.transaction_id}...")

        # Имитируем обработку платежа
        if request.card_token.startswith("INVALID"):
            print(f"    ✗ Invalid card")
            return None

        processor_id = f"PSP-{request.transaction_id}-{int(datetime.now().timestamp())}"
        print(f"    ✓ Payment processed: {processor_id}")
        return processor_id

    def refund(self, processor_id: str) -> bool:
        print(f"    [PaymentProcessor] Refunding payment {processor_id}...")
        # Имитируем возврат
        print(f"    ✓ Payment refunded")
        return True

class MerchantNotificationService:
    """Сервис уведомления мерчанта"""
    def notify_success(self, merchant_id: str, amount: float, transaction_id: str):
        print(f"    [Merchant] Notifying merchant {merchant_id}: Payment success ${amount}")

    def notify_failure(self, merchant_id: str, amount: float, transaction_id: str):
        print(f"    [Merchant] Notifying merchant {merchant_id}: Payment failed ${amount}")

class PaymentSagaOrchestrator:
    """Оркестратор платёжной Saga"""
    def __init__(self):
        self.fraud_service = FraudDetectionService()
        self.account_service = AccountService()
        self.processor_service = PaymentProcessorService()
        self.merchant_service = MerchantNotificationService()
        self.payment_states: Dict[str, PaymentState] = {}

    def setup_demo_account(self, user_id: str, balance: float):
        """Добавить демо-счёт"""
        self.account_service.add_account(user_id, balance)

    def process_payment(self, request: PaymentRequest) -> PaymentState:
        """Выполнить платёжную Saga"""

        # Инициализируем состояние
        state = PaymentState(
            transaction_id=request.transaction_id,
            status=PaymentStatus.INITIAL,
            user_id=request.user_id,
            amount=request.amount,
            merchant_id=request.merchant_id,
            created_at=datetime.now().isoformat()
        )
        self.payment_states[request.transaction_id] = state

        print(f"\n[Payment Saga] Starting: {request.transaction_id}")
        print(f"  Amount: ${request.amount}, User: {request.user_id}")

        # STEP 1: Validate
        print(f"\n  Step 1: Fraud Detection")
        state.status = PaymentStatus.VALIDATING
        if not self.fraud_service.validate(request):
            state.status = PaymentStatus.FAILED
            self.merchant_service.notify_failure(
                request.merchant_id, request.amount, request.transaction_id
            )
            print(f"  ✗ FAILED at fraud check")
            return state

        # STEP 2: Reserve Funds
        print(f"\n  Step 2: Reserve Funds")
        state.status = PaymentStatus.FUNDS_RESERVED
        if not self.account_service.reserve_funds(
            request.user_id, request.amount, request.transaction_id
        ):
            state.status = PaymentStatus.FAILED
            self.merchant_service.notify_failure(
                request.merchant_id, request.amount, request.transaction_id
            )
            print(f"  ✗ FAILED at reserve funds")
            return state

        state.reserved_amount = request.amount

        # STEP 3: Process Payment
        print(f"\n  Step 3: Process Payment with Processor")
        state.status = PaymentStatus.PROCESSING
        processor_id = self.processor_service.process(request)
        if not processor_id:
            state.status = PaymentStatus.FAILED
            print(f"  ✗ FAILED at processor - COMPENSATING")
            self._compensate(state, request)
            return state

        state.payment_processor_id = processor_id

        # STEP 4: Confirm (SUCCESS)
        print(f"\n  Step 4: Confirmation")
        state.status = PaymentStatus.COMPLETED
        state.updated_at = datetime.now().isoformat()
        self.merchant_service.notify_success(
            request.merchant_id, request.amount, request.transaction_id
        )
        print(f"  ✓ COMPLETED successfully")

        return state

    def _compensate(self, state: PaymentState, request: PaymentRequest):
        """Откатить платёж"""
        print(f"\n  Compensation steps (rolling back):")

        # Откатываем в обратном порядке (LIFO)
        if state.status == PaymentStatus.PROCESSING and state.payment_processor_id:
            print(f"\n    1. Refund from processor")
            self.processor_service.refund(state.payment_processor_id)

        if state.reserved_amount > 0:
            print(f"\n    2. Release reserved funds")
            self.account_service.release_funds(request.transaction_id)

        state.status = PaymentStatus.ROLLED_BACK
        print(f"\n  ✓ Compensation completed")

# === ДЕМОНСТРАЦИЯ ===

print("=" * 70)
print("PAYMENT SAGA SYSTEM")
print("=" * 70)

orchestrator = PaymentSagaOrchestrator()

# Добавляем демо-счёт
orchestrator.setup_demo_account("user123", balance=1000.0)

# Сценарий 1: Успешный платёж
print("\n" + "=" * 70)
print("SCENARIO 1: Successful Payment")
print("=" * 70)

request1 = PaymentRequest(
    transaction_id="TX-001",
    user_id="user123",
    amount=50.0,
    merchant_id="shop1",
    card_token="4111-1111-1111-1111"
)
state1 = orchestrator.process_payment(request1)
print(f"\nFinal state: {state1.status.value}")

# Сценарий 2: Мошенничество (высокая сумма)
print("\n" + "=" * 70)
print("SCENARIO 2: Fraud Detected (High Amount)")
print("=" * 70)

request2 = PaymentRequest(
    transaction_id="TX-002",
    user_id="user123",
    amount=15000.0,
    merchant_id="shop2",
    card_token="4111-1111-1111-1111"
)
state2 = orchestrator.process_payment(request2)
print(f"\nFinal state: {state2.status.value}")

# Сценарий 3: Недостаточно средств
print("\n" + "=" * 70)
print("SCENARIO 3: Insufficient Funds")
print("=" * 70)

request3 = PaymentRequest(
    transaction_id="TX-003",
    user_id="user123",
    amount=2000.0,  # В счёте осталось меньше (из-за TX-001)
    merchant_id="shop3",
    card_token="4111-1111-1111-1111"
)
state3 = orchestrator.process_payment(request3)
print(f"\nFinal state: {state3.status.value}")

# Сценарий 4: Отказ процессора
print("\n" + "=" * 70)
print("SCENARIO 4: Payment Processor Failure")
print("=" * 70)

orchestrator.setup_demo_account("user456", balance=500.0)

request4 = PaymentRequest(
    transaction_id="TX-004",
    user_id="user456",
    amount=100.0,
    merchant_id="shop4",
    card_token="INVALID-CARD-TOKEN"  # Вызовет отказ процессора
)
state4 = orchestrator.process_payment(request4)
print(f"\nFinal state: {state4.status.value}")

print("\n" + "=" * 70)
print("DEMO COMPLETED")
print("=" * 70)
```

### Типичные ошибки

1. **Забывают про случай отказа процессора** — теряют деньги
2. **Не хранят состояние Saga** — невозможно восстановиться после краша
3. **Не обрабатывают timeout** — платёж висит в неизвестном состоянии

### На интервью

- "Платёжная Saga: валидация → резервирование → обработка → подтверждение"
- Упомяните критические точки отказа
- Говорите про необходимость хранить состояние
- **Уточняющий вопрос:** "Что если процессор упал после обработки, но до подтверждения?" → Нужен status check

---

## 9. Saga для системы бронирования

### Зачем спрашивают
Другой сценарий использования Saga. Показывает гибкость паттерна.

### Короткий ответ
Booking Saga: проверка доступности → резервирование → оплата → подтверждение. Сложность в том, что нужно откатить бронирование, если оплата упадёт.

### Детальный разбор

**Процесс бронирования:**

```
Пользователь хочет забронировать отель:

1. CHECK AVAILABILITY
   ├─ Проверяем: есть ли свободные номера на даты?
   ├─ Если нет → abort, return error
   └─ Если да → переходим к Step 2

2. RESERVE ROOM
   ├─ Зарезервируем номер на имя пользователя
   ├─ Статус номера: AVAILABLE → RESERVED
   ├─ Если ошибка → компенсируем (отпускаем номер)
   └─ Если успех → переходим к Step 3

3. PROCESS PAYMENT
   ├─ Списываем деньги со счёта
   ├─ Если ошибка → компенсируем (отпускаем номер)
   └─ Если успех → переходим к Step 4

4. SEND CONFIRMATION
   ├─ Отправляем письмо пользователю
   └─ Бронирование завершено

Если на каком-то шаге ошибка:
  → Откатываем предыдущие шаги в обратном порядке
  → Пользователь получит письмо об отмене
```

### Пример

```python
from enum import Enum
from dataclasses import dataclass
from typing import Dict, List, Optional
from datetime import datetime, timedelta

class BookingStatus(Enum):
    PENDING = "pending"
    ROOM_RESERVED = "room_reserved"
    PAYMENT_PROCESSED = "payment_processed"
    CONFIRMED = "confirmed"
    CANCELLED = "cancelled"
    FAILED = "failed"

@dataclass
class BookingRequest:
    booking_id: str
    user_id: str
    hotel_id: str
    room_type: str
    check_in: str  # "2024-01-15"
    check_out: str  # "2024-01-17"
    price_per_night: float
    guest_count: int

@dataclass
class BookingState:
    booking_id: str
    user_id: str
    hotel_id: str
    room_id: Optional[str] = None
    status: BookingStatus = BookingStatus.PENDING
    total_price: float = 0.0
    nights: int = 0
    payment_id: Optional[str] = None
    created_at: str = ""

class AvailabilityService:
    """Сервис доступности номеров"""
    def __init__(self):
        self.rooms = {}

    def add_room(self, hotel_id: str, room_id: str, room_type: str, price: float):
        key = f"{hotel_id}:{room_id}"
        self.rooms[key] = {
            'room_id': room_id,
            'hotel_id': hotel_id,
            'type': room_type,
            'price': price,
            'status': 'available'  # available, reserved, booked
        }

    def check_availability(self, hotel_id: str, room_type: str,
                          check_in: str, check_out: str) -> Optional[str]:
        """Проверить доступность номера"""
        print(f"    [Availability] Checking {room_type} at {hotel_id}...")
        print(f"      Dates: {check_in} to {check_out}")

        for key, room in self.rooms.items():
            if (room['hotel_id'] == hotel_id and
                room['type'] == room_type and
                room['status'] == 'available'):
                print(f"      ✓ Found room: {room['room_id']}")
                return room['room_id']

        print(f"      ✗ No available rooms")
        return None

class ReservationService:
    """Сервис резервирования номеров"""
    def __init__(self, availability_service: AvailabilityService):
        self.availability = availability_service
        self.reservations = {}

    def reserve_room(self, booking_id: str, room_id: str,
                    hotel_id: str) -> bool:
        """Зарезервировать номер"""
        print(f"    [Reservation] Reserving room {room_id}...")

        key = f"{hotel_id}:{room_id}"
        if key not in self.availability.rooms:
            print(f"      ✗ Room not found")
            return False

        room = self.availability.rooms[key]
        if room['status'] != 'available':
            print(f"      ✗ Room not available (status: {room['status']})")
            return False

        room['status'] = 'reserved'
        self.reservations[booking_id] = room_id
        print(f"      ✓ Room reserved")
        return True

    def release_room(self, booking_id: str, room_id: str, hotel_id: str) -> bool:
        """Отпустить зарезервированный номер"""
        print(f"    [Reservation] Releasing room {room_id}...")

        key = f"{hotel_id}:{room_id}"
        if key not in self.availability.rooms:
            return False

        room = self.availability.rooms[key]
        if room['status'] != 'reserved':
            print(f"      ✗ Room not in reserved state")
            return False

        room['status'] = 'available'
        self.reservations.pop(booking_id, None)
        print(f"      ✓ Room released")
        return True

class PaymentService:
    """Сервис платежей"""
    def __init__(self):
        self.payments = {}
        self.accounts = {}

    def add_account(self, user_id: str, balance: float):
        self.accounts[user_id] = balance

    def process_payment(self, booking_id: str, user_id: str, amount: float) -> Optional[str]:
        """Обработать платёж"""
        print(f"    [Payment] Processing ${amount} for {user_id}...")

        if user_id not in self.accounts:
            print(f"      ✗ Account not found")
            return None

        if self.accounts[user_id] < amount:
            print(f"      ✗ Insufficient funds: ${self.accounts[user_id]} < ${amount}")
            return None

        self.accounts[user_id] -= amount
        payment_id = f"PAY-{booking_id}"
        self.payments[payment_id] = {'amount': amount, 'user_id': user_id}
        print(f"      ✓ Payment processed: {payment_id}")
        return payment_id

    def refund(self, payment_id: str) -> bool:
        """Вернуть платёж"""
        print(f"    [Payment] Refunding {payment_id}...")

        if payment_id not in self.payments:
            print(f"      ✗ Payment not found")
            return False

        payment = self.payments.pop(payment_id)
        self.accounts[payment['user_id']] += payment['amount']
        print(f"      ✓ Refunded ${payment['amount']}")
        return True

class NotificationService:
    """Сервис уведомлений"""
    def send_confirmation(self, user_id: str, booking_id: str, details: str):
        print(f"    [Notification] Sending confirmation to {user_id}")
        print(f"      Booking: {booking_id}")
        print(f"      Details: {details}")

    def send_cancellation(self, user_id: str, booking_id: str):
        print(f"    [Notification] Sending cancellation to {user_id}")
        print(f"      Booking cancelled: {booking_id}")

class BookingSagaOrchestrator:
    """Оркестратор бронирования"""
    def __init__(self):
        self.availability = AvailabilityService()
        self.reservation = ReservationService(self.availability)
        self.payment = PaymentService()
        self.notification = NotificationService()
        self.bookings: Dict[str, BookingState] = {}

    def setup_demo(self):
        """Инициализация демо-данных"""
        # Добавляем номера
        self.availability.add_room("hotel1", "room101", "standard", 100)
        self.availability.add_room("hotel1", "room102", "standard", 100)
        self.availability.add_room("hotel1", "room201", "deluxe", 150)

        # Добавляем счёты пользователей
        self.payment.add_account("user1", 500)
        self.payment.add_account("user2", 150)
        self.payment.add_account("user3", 50)

    def process_booking(self, request: BookingRequest) -> BookingState:
        """Выполнить Saga бронирования"""

        # Инициализируем состояние
        check_in = datetime.fromisoformat(request.check_in)
        check_out = datetime.fromisoformat(request.check_out)
        nights = (check_out - check_in).days
        total_price = nights * request.price_per_night

        state = BookingState(
            booking_id=request.booking_id,
            user_id=request.user_id,
            hotel_id=request.hotel_id,
            total_price=total_price,
            nights=nights,
            created_at=datetime.now().isoformat()
        )
        self.bookings[request.booking_id] = state

        print(f"\n[Booking Saga] {request.booking_id}")
        print(f"  User: {request.user_id}")
        print(f"  Hotel: {request.hotel_id}, {request.room_type}")
        print(f"  Dates: {request.check_in} to {request.check_out} ({nights} nights)")
        print(f"  Total: ${total_price}")

        # STEP 1: Check Availability
        print(f"\n  Step 1: Check Availability")
        room_id = self.availability.check_availability(
            request.hotel_id, request.room_type,
            request.check_in, request.check_out
        )
        if not room_id:
            state.status = BookingStatus.FAILED
            print(f"  ✗ FAILED: No rooms available")
            return state

        # STEP 2: Reserve Room
        print(f"\n  Step 2: Reserve Room")
        if not self.reservation.reserve_room(request.booking_id, room_id, request.hotel_id):
            state.status = BookingStatus.FAILED
            print(f"  ✗ FAILED: Cannot reserve room")
            return state

        state.room_id = room_id
        state.status = BookingStatus.ROOM_RESERVED

        # STEP 3: Process Payment
        print(f"\n  Step 3: Process Payment")
        payment_id = self.payment.process_payment(
            request.booking_id, request.user_id, total_price
        )
        if not payment_id:
            state.status = BookingStatus.FAILED
            print(f"  ✗ FAILED: Payment failed")
            print(f"\n  Compensation: Releasing room")
            self.reservation.release_room(request.booking_id, room_id, request.hotel_id)
            self.notification.send_cancellation(request.user_id, request.booking_id)
            return state

        state.payment_id = payment_id
        state.status = BookingStatus.PAYMENT_PROCESSED

        # STEP 4: Send Confirmation
        print(f"\n  Step 4: Send Confirmation")
        state.status = BookingStatus.CONFIRMED
        details = f"Room {room_id}, {request.check_in} to {request.check_out}"
        self.notification.send_confirmation(request.user_id, request.booking_id, details)
        print(f"  ✓ CONFIRMED")

        return state

# === ДЕМОНСТРАЦИЯ ===

print("=" * 70)
print("BOOKING SAGA SYSTEM")
print("=" * 70)

orchestrator = BookingSagaOrchestrator()
orchestrator.setup_demo()

# Сценарий 1: Успешное бронирование
print("\n" + "=" * 70)
print("SCENARIO 1: Successful Booking")
print("=" * 70)

request1 = BookingRequest(
    booking_id="BK-001",
    user_id="user1",
    hotel_id="hotel1",
    room_type="standard",
    check_in="2024-01-15",
    check_out="2024-01-17",
    price_per_night=100,
    guest_count=1
)
state1 = orchestrator.process_booking(request1)
print(f"\nFinal status: {state1.status.value}")

# Сценарий 2: Недостаточно средств (компенсация)
print("\n" + "=" * 70)
print("SCENARIO 2: Insufficient Funds (Compensation)")
print("=" * 70)

request2 = BookingRequest(
    booking_id="BK-002",
    user_id="user3",  # У юзера только $50
    hotel_id="hotel1",
    room_type="deluxe",  # $150 за ночь
    check_in="2024-02-01",
    check_out="2024-02-02",
    price_per_night=150,
    guest_count=2
)
state2 = orchestrator.process_booking(request2)
print(f"\nFinal status: {state2.status.value}")

# Сценарий 3: Нет доступных номеров
print("\n" + "=" * 70)
print("SCENARIO 3: No Available Rooms")
print("=" * 70)

request3 = BookingRequest(
    booking_id="BK-003",
    user_id="user2",
    hotel_id="hotel1",
    room_type="luxury",  # Такого типа нет
    check_in="2024-03-01",
    check_out="2024-03-03",
    price_per_night=200,
    guest_count=4
)
state3 = orchestrator.process_booking(request3)
print(f"\nFinal status: {state3.status.value}")

print("\n" + "=" * 70)
print("DEMO COMPLETED")
print("=" * 70)
```

### Типичные ошибки

1. **Не откатывают резервирование при ошибке платежа** → номер остаётся занят
2. **Забывают про overlapping bookings** → два пользователя на один номер
3. **Отправляют подтверждение до платежа** → деньги не прошли, но клиент думает что всё ОК

### На интервью

- "Booking Saga: проверка → резервирование → оплата → подтверждение"
- Упомяните компенсацию (отпуск номера при ошибке платежа)
- **Уточняющий вопрос:** "Что если два пользователя одновременно забронировали последний номер?" → Нужна оптимистичная блокировка или версионирование

---

## 10. Temporal/Cadence инструменты

### Зачем спрашивают
Практические инструменты для Saga. Показывает, что вы не только теоретически знаете, но и использовали реальные фреймворки.

### Короткий ответ
Temporal и Cadence — это фреймворки для управления долгоживущими процессами. Они автоматически обрабатывают retry, compensation, state persistence. Позволяют писать Saga как обычный последовательный код.

### Детальный разбор

**Сравнение подходов:**

```
Manual Saga (без инструментов):
├─ Пишем Event Bus
├─ Пишем State Machine
├─ Пишем Retry логику
├─ Пишем Compensation логику
├─ Сложно тестировать
├─ Легко ошибиться
└─ Много boilerplate кода

Temporal/Cadence (с инструментами):
├─ Определяем Workflow (как обычный код)
├─ Определяем Activities (отдельные операции)
├─ Framework сам обрабатывает:
│  ├─ State persistence
│  ├─ Retry + exponential backoff
│  ├─ Timeout handling
│  ├─ Compensation (через try-catch)
│  └─ Distributed tracing
├─ Легко тестировать
├─ Безопаснее
└─ Чистый код
```

**Архитектура Temporal:**

```
┌─────────────────────────────────────┐
│         Temporal Server             │
│ (Управляет состоянием Workflows)    │
└──────────────┬──────────────────────┘
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
 Worker 1  Worker 2  Worker 3
 (Runtime) (Runtime) (Runtime)
     │         │         │
     ├─────────┼─────────┤
     │   Workflow        │
     │   Execution       │
     │   History         │
     └────────────────────┘
```

### Пример

```python
"""
Пример использования Temporal-подобного фреймворка
(симулируем основной функционал)
"""

from typing import Callable, Any, Dict, List
from enum import Enum
import json
from dataclasses import dataclass

class ActivityStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    RETRYING = "retrying"

@dataclass
class ActivityExecution:
    activity_name: str
    status: ActivityStatus
    attempt: int
    result: Any = None
    error: str = None

class TemporalActivity:
    """Симуляция Activity в Temporal"""
    def __init__(self, name: str, max_retries: int = 3):
        self.name = name
        self.max_retries = max_retries

    def __call__(self, func: Callable):
        def wrapper(*args, **kwargs):
            # Это была бы обёртка вокруг Activity
            return func(*args, **kwargs)
        wrapper._activity_name = self.name
        wrapper._max_retries = self.max_retries
        return wrapper

class TemporalWorkflow:
    """Симуляция Workflow в Temporal"""
    def __init__(self, name: str):
        self.name = name
        self.activities: Dict[str, Callable] = {}
        self.history: List[ActivityExecution] = []

    def register_activity(self, activity_func: Callable):
        """Зарегистрировать Activity"""
        activity_name = getattr(activity_func, '_activity_name', activity_func.__name__)
        self.activities[activity_name] = activity_func

    def execute_activity_with_retry(self, activity_name: str, *args, **kwargs):
        """Выполнить Activity с retry и exponential backoff"""
        max_retries = 3
        attempt = 0

        while attempt < max_retries:
            attempt += 1
            try:
                print(f"    [Temporal] Executing {activity_name} (attempt {attempt}/{max_retries})")

                activity = self.activities[activity_name]
                result = activity(*args, **kwargs)

                execution = ActivityExecution(
                    activity_name=activity_name,
                    status=ActivityStatus.COMPLETED,
                    attempt=attempt,
                    result=result
                )
                self.history.append(execution)
                return result

            except Exception as e:
                execution = ActivityExecution(
                    activity_name=activity_name,
                    status=ActivityStatus.FAILED if attempt >= max_retries else ActivityStatus.RETRYING,
                    attempt=attempt,
                    error=str(e)
                )
                self.history.append(execution)

                if attempt >= max_retries:
                    print(f"    ✗ Activity failed after {max_retries} attempts")
                    raise

                backoff = 2 ** attempt
                print(f"    ✗ Failed: {e}")
                print(f"    Retrying in {backoff}s...")
                import time
                time.sleep(backoff / 1000)  # Имитируем delay

# === ПРИМЕР: Платёжный Workflow ===

# Регистрируем Activities
@TemporalActivity("ValidatePayment", max_retries=3)
def validate_payment(amount: float) -> bool:
    """Activity: Проверка платежа"""
    print("      → Validating payment")
    if amount < 0:
        raise ValueError("Invalid amount")
    return True

@TemporalActivity("ProcessPayment", max_retries=5)
def process_payment(user_id: str, amount: float) -> str:
    """Activity: Обработка платежа"""
    print(f"      → Processing payment: ${amount}")
    # Имитируем вызов платёжной системы
    payment_id = f"PAY-{user_id}-{int(amount)}"
    return payment_id

@TemporalActivity("SendNotification", max_retries=3)
def send_notification(user_id: str, message: str) -> bool:
    """Activity: Отправить уведомление"""
    print(f"      → Sending notification to {user_id}: {message}")
    return True

@TemporalActivity("RefundPayment", max_retries=5)
def refund_payment(payment_id: str) -> bool:
    """Activity: Возврат платежа (compensation)"""
    print(f"      → Refunding payment: {payment_id}")
    return True

# Определяем Workflow (это просто обычный Python код!)
def payment_workflow(user_id: str, amount: float):
    """
    Workflow: Обработка платежа с компенсацией

    В Temporal это выглядит как обычный последовательный код.
    Temporal автоматически:
    - Сохраняет состояние
    - Обрабатывает retry
    - Обрабатывает timeout
    - Восстанавливается после краша
    """

    try:
        # Шаг 1: Валидация
        validate_payment(amount)

        # Шаг 2: Обработка платежа
        payment_id = process_payment(user_id, amount)

        # Шаг 3: Уведомление
        send_notification(user_id, f"Payment successful: ${amount}")

        return {"status": "success", "payment_id": payment_id}

    except Exception as e:
        # Автоматическая компенсация
        try:
            refund_payment(payment_id)
            send_notification(user_id, f"Payment failed and refunded: {str(e)}")
        except Exception as comp_error:
            print(f"Compensation failed: {comp_error}")

        return {"status": "failed", "error": str(e)}

# === ДЕМОНСТРАЦИЯ ===

print("=" * 70)
print("TEMPORAL-LIKE WORKFLOW EXECUTION")
print("=" * 70)

# Создаём Workflow
workflow = TemporalWorkflow("PaymentWorkflow")

# Регистрируем Activities
workflow.register_activity(validate_payment)
workflow.register_activity(process_payment)
workflow.register_activity(send_notification)
workflow.register_activity(refund_payment)

# Сценарий 1: Успешный платёж
print("\n" + "=" * 70)
print("SCENARIO 1: Successful Payment (with automatic retry)")
print("=" * 70)

print("\n[Workflow] Starting payment workflow")

try:
    # Выполняем Activities через Temporal
    workflow.execute_activity_with_retry("ValidatePayment", 100)
    payment_id = workflow.execute_activity_with_retry("ProcessPayment", "user1", 100)
    workflow.execute_activity_with_retry("SendNotification", "user1", "Payment successful!")

    print(f"\n✓ Workflow COMPLETED successfully")
    print(f"Payment ID: {payment_id}")

except Exception as e:
    print(f"\n✗ Workflow FAILED: {e}")

# Сценарий 2: Сценарий с kompенсацией
print("\n" + "=" * 70)
print("SCENARIO 2: Payment with Compensation")
print("=" * 70)

print("\n[Workflow] Starting payment workflow with error handling")

workflow2 = TemporalWorkflow("PaymentWorkflow2")
workflow2.register_activity(validate_payment)
workflow2.register_activity(process_payment)
workflow2.register_activity(send_notification)
workflow2.register_activity(refund_payment)

payment_id_for_refund = None
try:
    workflow2.execute_activity_with_retry("ValidatePayment", 100)
    payment_id_for_refund = workflow2.execute_activity_with_retry("ProcessPayment", "user2", 100)

    # Имитируем ошибку в Send Notification
    print("    [Workflow] Simulating error in SendNotification...")
    raise Exception("Notification service temporary unavailable")

except Exception as e:
    print(f"\n✗ Error in workflow: {e}")
    print(f"Initiating compensation...")

    try:
        if payment_id_for_refund:
            workflow2.execute_activity_with_retry("RefundPayment", payment_id_for_refund)
            workflow2.execute_activity_with_retry("SendNotification", "user2", "Payment refunded due to error")
        print(f"✓ Compensation completed")
    except Exception as comp_error:
        print(f"✗ Compensation failed: {comp_error}")

# === ВЫВОД ИСТОРИИ ВЫПОЛНЕНИЯ ===

print("\n" + "=" * 70)
print("WORKFLOW EXECUTION HISTORY")
print("=" * 70)

def print_history(workflow: TemporalWorkflow):
    print(f"\nWorkflow: {workflow.name}")
    print("Activities executed:")
    for i, execution in enumerate(workflow.history, 1):
        status_symbol = "✓" if execution.status == ActivityStatus.COMPLETED else "✗"
        print(f"  {i}. {status_symbol} {execution.activity_name} "
              f"(attempt {execution.attempt}, status: {execution.status.value})")
        if execution.result:
            print(f"     Result: {execution.result}")
        if execution.error:
            print(f"     Error: {execution.error}")

print_history(workflow)
print_history(workflow2)

# === СРАВНЕНИЕ С MANUAL SAGA ===

print("\n" + "=" * 70)
print("MANUAL SAGA vs TEMPORAL")
print("=" * 70)

comparison = """
MANUAL SAGA (без Temporal):
├─ 200 строк кода для простого Workflow
├─ Нужно писать State Machine
├─ Нужно писать Retry логику
├─ Нужно писать Event Bus
├─ Сложно тестировать (mock Event Bus)
├─ Легко упустить edge cases
└─ Сложно отследить историю

TEMPORAL (с инструментом):
├─ 20-30 строк кода (код выглядит как обычный Python)
├─ Temporal сам управляет State Machine
├─ Встроенные Retry + exponential backoff
├─ Встроенные Compensation (через try-catch)
├─ Легко тестировать (встроенный TestWorkflowEnvironment)
├─ Безопаснее (less error-prone)
├─ Встроенная история выполнения
├─ Встроенный Distributed Tracing
└─ Безопасное восстановление после краша
"""

print(comparison)
```

### Типичные ошибки

1. **Забывают про determinism** — Workflow должен быть детерминирован
2. **Используют случайные числа в Workflow** — приводит к различным результатам при replay
3. **Не возвращаются к Temporal после обучения** — думают, что сложно

### На интервью

- "Temporal — это фреймворк для долгоживущих процессов"
- Преимущества: автоматический retry, state persistence, compensation
- Покажите, как Workflow выглядит как обычный код
- **Уточняющий вопрос:** "Почему Workflow должен быть детерминирован?" → Потому что Temporal воспроизводит его из истории

---

## Заключение

| Тема | Ключевой вывод |
|------|---|
| ACID в распределённых системах | Невозможно гарантировать синхронно, нужна eventual consistency |
| 2PC | Работает, но блокирует ресурсы и не масштабируется |
| Saga паттерн | Лучший выбор для микросервисов: локальные транзакции + компенсация |
| Orchestration vs Choreography | Orchestration проще, Choreography масштабируется лучше |
| Компенсирующие транзакции | Undo операции в обратном порядке выполнения |
| Идемпотентность | Критична для обработки дублирующихся событий |
| Частичные отказы | Retry, Circuit Breaker, Compensation, Dead Letter Queue |
| Платёжная система | Валидация → резервирование → обработка → подтверждение |
| Система бронирования | Проверка → резервирование → оплата → подтверждение |
| Temporal/Cadence | Фреймворки для управления Saga без boilerplate кода |

**Финальный совет на интервью:** Начните с базового понимания, что ACID не работает в распределённых системах, затем переходите к Saga как решению. Покажите, что вы знаете trade-offs между Orchestration и Choreography, и упомяните инструменты вроде Temporal для практической реализации.
