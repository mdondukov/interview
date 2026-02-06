# Отказы и Восстановление в Распределённых Системах

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [18-consistency-models](./18-consistency-models.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Crash Failure** — отказ узла, когда он полностью падает и перестаёт работать, теряя всю оперативную память и не отвечая на запросы. Это самый простой тип отказа для обнаружения, так как можно использовать механизм heartbeat и timeout для определения, что узел упал.

**Omission Failure** — отказ, когда узел продолжает работать, но не может отправлять или получать некоторые (или все) сообщения, теряя их в сети или в своих буферах. Сложнее обнаружить, чем crash failure, требует активного контроля потерь пакетов и может быть временным или частичным.

**Timing Failure** — отказ, когда узел работает корректно, но отвечает медленнее обычного или полностью пропускает критический дедлайн. Невозможно отличить от omission failure в полностью асинхронной сети без дополнительной информации о поведении узла.

**Byzantine Failure** — худший тип отказа, когда узел может произвольно отклоняться от протокола, отправлять ложные данные разным узлам или отправлять противоречивые сообщения. Требует 3f+1 узлов для допуска f неправильных узлов и использования специализированных алгоритмов типа PBFT.

**Failure Detection** — механизм обнаружения, что узел упал или работает неправильно, обычно с использованием heartbeat сообщений и таймаутов. Критично для автоматического восстановления, инициирования failover'а и поддержания живости системы.

**Heartbeat** — регулярные сообщения от узла, которые подтверждают его живость и нормальную работу. Простой и практичный способ детектирования crash failure через наблюдение за отсутствием heartbeat сообщений в течение определённого таймаута.

**Split-brain** — критичное состояние, когда разделение сети на две партиции происходит и каждая партиция считает другую мёртвой, выбирая своего лидера. Приводит к конфликтам данных, когда две партиции пишут противоречивые значения в одновременно.

**Quorum** — большинство узлов в системе (больше n/2), необходимые для принятия решения. Использование quorum-based election предотвращает split-brain, позволяя действовать только партиции с большинством узлов.

**Fencing** — механизм блокировки старого лидера или процесса, чтобы он не мог писать данные и не создавал конфликты. Дополнительная защита против split-brain вместе с quorum-based election, используется в системах вроде Zookeeper и Etcd.

**Recovery** — процесс восстановления упавшего узла и синхронизации его состояния с остальной системой после рестарта. Необходимо, чтобы система могла полностью пережить отказы узлов и восстановить их без потери данных.

**PBFT (Byzantine Fault Tolerance)** — практический алгоритм консенсуса для работы в присутствии Byzantine узлов, разработанный для работы в враждебных или недоверяемых окружениях. Используется в блокчейне, системах с критичными требованиями безопасности и других критичных распределённых системах.

---

## 1. Какие типы отказов бывают в распределённых системах?

### Зачем спрашивают
Понимание типов отказов показывает, что интервьюируемый осознаёт сложность реальных систем и может проектировать решения с учётом различных сценариев повреждений. Это базовое знание для построения надёжных систем.

### Короткий ответ
В распределённых системах бывают четыре основных типа отказов: **crash failure** (узел падает), **omission failure** (потеря сообщений), **timing failure** (задержки), **Byzantine failure** (произвольное поведение). Каждый тип требует своего подхода к обнаружению и восстановлению.

### Детальный разбор

**Crash Failure** — узел полностью перестаёт функционировать и не восстанавливается самостоятельно.
- Характеристики: прерывание всех операций, потеря памяти
- Обнаружение: timeout на запросы
- Восстановление: перенос нагрузки на другие узлы

**Omission Failure** — узел продолжает работать, но не отправляет или не получает некоторые сообщения.
- Send omission: узел не может отправить сообщение
- Receive omission: узел не получает входящие сообщения

**Timing Failure** — узел работает корректно, но внутри неправильных временных параметров.
- Response timeout: слишком медленный ответ
- Deadline miss: опоздание на критический момент

**Byzantine Failure** — узел может произвольным образом отклоняться от протокола.
- Может отправить противоречивые сообщения разным участникам
- Может корректно ответить на некоторые запросы и игнорировать другие

```
┌─────────────────────────────────────────────────────┐
│         ТИПЫ ОТКАЗОВ В РАСПРЕДЕЛЁННЫХ СИСТЕМАХ      │
├─────────────────────────────────────────────────────┤
│                                                       │
│  CRASH FAILURE          OMISSION FAILURE             │
│  ┌─────────┐            ┌─────────┐                 │
│  │ Node A  │            │ Node A  │                 │
│  │ ===X    │            │ (работает, но теряет      │
│  │ stopped │            │  сообщения)               │
│  └─────────┘            └─────────┘                 │
│                                                       │
│  TIMING FAILURE         BYZANTINE FAILURE            │
│  ┌─────────┐            ┌─────────┐                 │
│  │ Node A  │            │ Node A  │                 │
│  │ работает│            │ может послать              │
│  │ но МЕДЛЕННО           │ противоречивые msg        │
│  └─────────┘            └─────────┘                 │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Пример

```python
class FailureDetector:
    """Обнаружение разных типов отказов"""

    def __init__(self, heartbeat_timeout=5.0):
        self.heartbeat_timeout = heartbeat_timeout
        self.last_heartbeat = {}
        self.failure_type = {}

    def detect_crash_failure(self, node_id: str, current_time: float) -> bool:
        """Узел полностью упал (timeout)"""
        if node_id in self.last_heartbeat:
            if current_time - self.last_heartbeat[node_id] > self.heartbeat_timeout:
                self.failure_type[node_id] = "CRASH"
                return True
        return False

    def detect_timing_failure(self, response_time: float, threshold: float = 2.0) -> bool:
        """Узел работает, но медленнее нормального"""
        if response_time > threshold:
            return True
        return False

    def detect_omission_failure(self, sent_count: int, received_count: int) -> bool:
        """Потеря сообщений (discrepancy между sent и received)"""
        if sent_count > received_count:
            return True
        return False

    def heartbeat_received(self, node_id: str, timestamp: float):
        """Обновление последнего heartbeat"""
        self.last_heartbeat[node_id] = timestamp
```

### Типичные ошибки
1. **Путаница между типами** — не все медленные отказы — это timing failure; иногда это симптом omission failure
2. **Игнорирование Byzantine failures** — в критических системах нельзя предполагать честность всех узлов
3. **Слишком короткие timeout** — вызывают false positives при сетевых скачках
4. **Одинаковые стратегии для разных отказов** — crash требует другого подхода, чем timing

### На интервью
**Как отвечать:**
- Начните с простого определения четырёх типов
- Приведите примеры каждого из реальной жизни (падение сервера, потеря пакета, медленный диск)
- Упомяните, что обнаружение — это нетривиальная задача (false positives vs false negatives)

**Follow-up вопросы:**
- "Как обнаружить каждый тип отказа?" → heartbeats, timeouts, acks, monitoring
- "Какой тип отказа самый опасный?" → Byzantine, требует consensus протокола
- "Как количество отказов влияет на систему?" → в consensus нужно допустить f < n/3 для Byzantine

---

## 2. Что такое split-brain и как его предотвратить?

### Зачем спрашивают
Split-brain — это критическая проблема при разделении сети, когда несколько компонентов считают себя лидером. Вопрос проверяет понимание консенсуса и предотвращения конфликтов данных.

### Короткий ответ
Split-brain происходит, когда в кластере из-за разделения сети возникают две партиции, и каждая пытается действовать независимо, создавая конфликты. Предотвращение: использование кворума, fencing, heartbeat-based leader election с timeout.

### Детальный разбор

**Сценарий split-brain:**

```
        НАЧАЛЬНО (здоровый кластер)

        ┌─────────┐    ┌─────────┐    ┌─────────┐
        │ Node 1  │────│ Node 2  │────│ Node 3  │
        │ LEADER  │    │ FOLLOWER│    │ FOLLOWER│
        └─────────┘    └─────────┘    └─────────┘


        ПОСЛЕ РАЗДЕЛЕНИЯ СЕТИ (split-brain)

        ┌─────────┐                    ┌─────────┐
        │ Node 1  │                    │ Node 2  │
        │ LEADER  │                    │ LEADER! │
        └─────────┘                    └─────────┘
                                       │
                                       │
                                    ┌─────────┐
                                    │ Node 3  │
                                    │ FOLLOWER│
                                    └─────────┘

        Две партиции видят разные лидеры!
        Конфликты данных неизбежны.
```

**Решение — Quorum-based Leader Election:**

```
Правило: новый лидер может быть избран только если он имеет
поддержку большинства (> n/2) узлов в кластере.

На примере 3-узлового кластера:
- Необходимо 2 голоса для избрания лидера
- Если разделение 1 vs 2: партиция с 2 узлами может
  избрать нового лидера
- Партиция с 1 узлом понимает, что не имеет большинства,
  и становится read-only
```

**Fencing механизм:**

Если старый лидер случайно остался в меньшинстве и пытается писать:

```python
def fence_old_leader():
    """Механизм, который отключает старого лидера"""
    - Revoke IP адреса
    - Отозвать сертификаты
    - Shutdown узла
    - Сбросить состояние
```

### Пример

```python
class QuorumBasedLeader:
    """Избрание лидера с использованием кворума"""

    def __init__(self, node_id: int, total_nodes: int):
        self.node_id = node_id
        self.total_nodes = total_nodes
        self.votes_received = 0
        self.is_leader = False
        self.quorum_size = (total_nodes // 2) + 1
        self.fenced = False

    def request_vote(self, candidate_id: int, term: int) -> bool:
        """Логика голосования (упрощённо из Raft)"""
        # Голосуем только если кандидат в текущем или новом терме
        # и мы ещё не голосовали в этом терме
        return True  # simplified

    def check_quorum_achieved(self):
        """Проверка, достаточно ли голосов для лидерства"""
        if self.votes_received >= self.quorum_size:
            self.is_leader = True
            print(f"Node {self.node_id}: стал лидером (получил {self.votes_received} голосов)")
            return True
        else:
            print(f"Node {self.node_id}: недостаточно голосов ({self.votes_received}/{self.quorum_size})")
            return False

    def detect_partition(self, active_nodes: int):
        """Обнаружение разделения сети"""
        if active_nodes < self.quorum_size:
            print(f"Node {self.node_id}: PARTITION! Активных узлов: {active_nodes}, нужно: {self.quorum_size}")
            print(f"Node {self.node_id}: переходит в режим read-only")
            self.is_leader = False
            return True
        return False

    def write_with_protection(self, data: str) -> bool:
        """Защищённая запись: только если достаточно подтверждений"""
        if not self.is_leader:
            print(f"Node {self.node_id}: не лидер, запись отклонена")
            return False

        if self.fenced:
            print(f"Node {self.node_id}: был заборонен, запись блокирована")
            return False

        print(f"Node {self.node_id}: записываю '{data}'")
        return True


# Демонстрация
print("=== Здоровый кластер ===")
node1 = QuorumBasedLeader(1, 3)
node1.votes_received = 2  # лидер получит голоса от себя и node2
node1.check_quorum_achieved()  # Leader elected

print("\n=== Split-brain защита ===")
node2 = QuorumBasedLeader(2, 3)
node2.detect_partition(1)  # только node2 и node3, но узел2 сам
node2.votes_received = 1   # только свой голос
node2.check_quorum_achieved()  # Не будет лидером
node2.write_with_protection("data")  # Отклонено
```

### Типичные ошибки
1. **Использование простого heartbeat без кворума** — не защищает от split-brain
2. **Неправильный расчёт кворума** — кворум должен быть > n/2, не >= n/2
3. **Отсутствие fencing механизма** — старый лидер может продолжить писать
4. **Игнорирование clock skew** — при синхронизации часов может возникнуть проблема

### На интервью
**Как отвечать:**
- Нарисуйте диаграмму split-brain сценария
- Объясните, почему простой heartbeat недостаточен
- Упомяните кворум и его математику (> n/2)
- Покажите, как fencing предотвращает двойной лидер

**Follow-ups:**
- "Как работает Raft или Paxos?" → consensus алгоритмы, которые встроенно решают split-brain
- "Что если у нас чётное число узлов?" → Обычно используют 2k+1, чтобы избежать неопределённости
- "А если сеть очень нестабильна?" → Adaptive timeouts, более длительные периоды для выборов

---

## 3. Что такое Byzantine failures?

### Зачем спрашивают
Byzantine failures — худший случай в распределённых системах, когда узлы могут произвольно отклоняться от протокола. Вопрос проверяет понимание криптографии, consensus и надёжности.

### Короткий ответ
Byzantine failure — это когда узел произвольно отклоняется от протокола, может отправлять противоречивые сообщения и вообще лгать. Byzantine Fault Tolerance (BFT) требует 3f+1 узлов, чтобы допустить f неправильных узлов.

### Детальный разбор

**Почему Byzantine так опасен:**

```
Node A может:
- Отправить "commit" узлу B и "abort" узлу C
- Игнорировать heartbeat от других узлов
- Отправить поддельные данные
- Заявить, что другие узлы отправили сообщения, которых они не отправляли
```

**Требования для Byzantine Fault Tolerance:**

```
┌──────────────────────────────────────────────┐
│  Для допуска f Byzantine узлов нужно:        │
│                                               │
│  МИНИМУМ: 3f + 1 узлов в системе             │
│                                               │
│  Примеры:                                    │
│  - Допуск 1 Byzantine узла: нужно 4 узла     │
│  - Допуск 2 Byzantine узлов: нужно 7 узлов   │
│  - Допуск 3 Byzantine узлов: нужно 10 узлов  │
│                                               │
│  Алгоритм PBFT (Practical BFT):              │
│  1. Client отправляет запрос лидеру          │
│  2. Лидер replicate его другим репликам      │
│  3. Реплики выполняют и отправляют ответы    │
│  4. Client ждёт 2f+1 одинаковых ответов     │
│  5. Если лидер нарушает протокол, запускают  │
│     view change (новые выборы)               │
└──────────────────────────────────────────────┘
```

**Классический пример — Byzantine Generals Problem:**

```
Ситуация: 4 генерала (узлы), один из них предатель (Byzantine)

Сценарий ATTACK/RETREAT:
- Генерал 1: "ATTACK"
- Генерал 2: "ATTACK"
- Генерал 3 (ПРЕДАТЕЛЬ): отправляет "ATTACK" кому-то, "RETREAT" другим
- Генерал 4: "RETREAT"

Благоустройство (Byzantine General's theorem):
Если f < n/3, можно достичь консенсуса даже с f предателями.

При 4 генералах, 1 предатель: 1 < 4/3 ✓ (решаемо)
```

### Пример

```python
import hashlib
from collections import defaultdict

class ByzantineFaultTolerance:
    """Упрощённая реализация BFT логики"""

    def __init__(self, total_replicas: int, max_faulty: int):
        self.total_replicas = total_replicas
        self.max_faulty = max_faulty
        self.quorum_size = 2 * max_faulty + 1  # 2f+1 ответов нужно
        self.responses = defaultdict(list)

    def verify_quorum_for_bft(self) -> bool:
        """Проверка, что система может работать с Byzantine узлами"""
        # Нужно: 3f + 1 >= total_replicas
        min_required = 3 * self.max_faulty + 1
        if self.total_replicas >= min_required:
            print(f"✓ BFT возможна: {self.total_replicas} >= {min_required}")
            return True
        else:
            print(f"✗ BFT невозможна: {self.total_replicas} < {min_required}")
            return False

    def client_collects_responses(self, replica_responses: dict):
        """Client собирает ответы от реплик"""
        response_counts = defaultdict(int)

        for replica_id, response in replica_responses.items():
            # Хешируем ответ для сравнения
            response_hash = hashlib.md5(str(response).encode()).hexdigest()
            response_counts[response_hash] += 1

        # Client ждёт 2f+1 одинаковых ответов
        for resp_hash, count in response_counts.items():
            if count >= self.quorum_size:
                print(f"✓ Client получил {count} одинаковых ответов (нужно {self.quorum_size})")
                print(f"  Консенсус достигнут!")
                return True

        print(f"✗ Недостаточно одинаковых ответов")
        print(f"  Распределение: {dict(response_counts)}")
        return False

    def detect_byzantine_replica(self, responses: dict) -> bool:
        """Обнаружение Byzantine реплики"""
        response_counts = defaultdict(int)

        for replica_id, response in responses.items():
            response_hash = hashlib.md5(str(response).encode()).hexdigest()
            response_counts[response_hash] += 1

        # Если есть меньшинство ответов, значит Byzantine узлы
        if len(response_counts) > 1:
            print(f"⚠️  Обнаружены противоречивые ответы от реплик!")
            for i, (resp_hash, count) in enumerate(response_counts.items()):
                print(f"   Ответ {i}: {count} реплик")
            return True
        return False


# Демонстрация
print("=== BFT с 4 репликами, допуск 1 Byzantine ===\n")

bft = ByzantineFaultTolerance(total_replicas=4, max_faulty=1)
bft.verify_quorum_for_bft()

print("\n=== Честные ответы ===")
responses_honest = {
    "replica_0": "SUCCESS",
    "replica_1": "SUCCESS",
    "replica_2": "SUCCESS",
    "replica_3": "SUCCESS (но Byzantine)"
}
bft.client_collects_responses(responses_honest)

print("\n=== С одной Byzantine репликой ===")
responses_with_byzantine = {
    "replica_0": "SUCCESS",
    "replica_1": "SUCCESS",
    "replica_2": "SUCCESS",
    "replica_3": "FAIL"  # Byzantine отправляет ложь
}
bft.detect_byzantine_replica(responses_with_byzantine)
bft.client_collects_responses(responses_with_byzantine)

print("\n=== Недостаточно реплик для BFT (3 вместо 4) ===\n")
bft_insufficient = ByzantineFaultTolerance(total_replicas=3, max_faulty=1)
bft_insufficient.verify_quorum_for_bft()
```

### Типичные ошибки
1. **Путаница между crash и Byzantine** — Byzantine намного сложнее и требует более сложных алгоритмов
2. **Неправильная формула кворума** — для Byzantine нужно 2f+1, а не f+1
3. **Игнорирование криптографической верификации** — Byzantine узлы могут подделать сообщения без подписей
4. **Недостаточное количество узлов** — с 3 узлами можно допустить только 0 Byzantine (максимум 1 crash)

### На интервью
**Как отвечать:**
- Начните с простого определения: узел может произвольно отклониться
- Дайте формулу: 3f+1 для f Byzantine узлов
- Приведите пример Byzantine Generals Problem
- Упомяните PBFT алгоритм как решение

**Follow-ups:**
- "Где используется BFT на практике?" → Blockchain, distributed ledgers (Bitcoin, Ethereum)
- "Почему Bitcoin использует PoW вместо BFT?" → PoW более масштабируемый для открытых сетей
- "Какова стоимость BFT?" → O(n²) сложность, не подходит для больших систем

---

## 4. Как работает circuit breaker pattern?

### Зачем спрашивают
Circuit breaker — стандартный паттерн для обработки отказов внешних сервисов. Вопрос проверяет практическое знание resilience patterns и понимание graceful degradation.

### Короткий ответ
Circuit breaker работает как электрический предохранитель: если сервис начинает отказывать, circuit breaker открывается и начинает отклонять запросы без попытки их выполнения. Это предотвращает каскадные отказы и даёт время сервису на восстановление.

### Детальный разбор

**Три состояния circuit breaker:**

```
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  CLOSED (здоровый)          OPEN (сломан)   HALF-OPEN        │
│  ┌──────────────┐            ┌──────────────┐ ┌──────────────┐│
│  │  Requests    │            │  Requests    │ │  Tentative   ││
│  │  проходят    │            │  отклоняются │ │  пытаемся    ││
│  │  нормально   │            │  (fail-fast) │ │  восстановл. ││
│  │              │            │              │ │              ││
│  │  Ошибки      │            │  После       │ │  Если успеха:││
│  │  считаются   │            │  timeout → 1 │ │  → CLOSED    ││
│  │              │            │  запрос     │ │              ││
│  │  Успехи >= N:│            │  HALF-OPEN  │ │  Если ошибка:││
│  │  счётчик = 0 │            │              │ │  → OPEN      ││
│  │              │            │              │ │              ││
│  │  Ошибки >= M:│            │              │ │              ││
│  │  → OPEN      │            │              │ │              ││
│  └──────────────┘            └──────────────┘ └──────────────┘│
│                                                               │
└─────────────────────────────────────────────────────────────┘

Переходы:
CLOSED --(error threshold reached)--> OPEN
OPEN --(timeout)--> HALF-OPEN
HALF-OPEN --(request success)--> CLOSED
HALF-OPEN --(request fails)--> OPEN
```

**Параметры настройки:**

```python
{
    "failure_threshold": 5,      # Сколько ошибок до открытия
    "success_threshold": 2,      # Сколько успехов для закрытия
    "timeout": 60,               # Секунд до попытки HALF-OPEN
    "time_window": 10,           # Окно для подсчёта ошибок
    "half_open_requests": 3      # Запросов в HALF-OPEN для проверки
}
```

### Пример

```python
import time
from enum import Enum
from collections import deque

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    """Реализация circuit breaker pattern"""

    def __init__(self,
                 failure_threshold: int = 5,
                 recovery_timeout: int = 60,
                 success_threshold: int = 2,
                 time_window: int = 10):
        self.state = CircuitState.CLOSED
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.time_window = time_window

        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.last_state_change_time = time.time()
        self.request_times = deque()

    def call(self, func, *args, **kwargs):
        """Выполнить функцию через circuit breaker"""

        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                print(f"[CB] OPEN → HALF-OPEN (timeout истёк)")
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
            else:
                raise CircuitBreakerOpenException(
                    f"Circuit breaker открыт. "
                    f"Попробуйте через {self._time_until_reset():.1f} сек"
                )

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        """Обработка успешного запроса"""
        self.failure_count = 0

        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            print(f"[CB] HALF-OPEN: успех {self.success_count}/{self.success_threshold}")

            if self.success_count >= self.success_threshold:
                self._close()

    def _on_failure(self):
        """Обработка неудачного запроса"""
        current_time = time.time()
        self.last_failure_time = current_time
        self.failure_count += 1

        print(f"[CB] Ошибка #{self.failure_count}/{self.failure_threshold}")

        if self.state == CircuitState.HALF_OPEN:
            self._open()
        elif self.failure_count >= self.failure_threshold:
            self._open()

    def _open(self):
        """Открыть circuit breaker"""
        if self.state != CircuitState.OPEN:
            print(f"[CB] CLOSED/HALF-OPEN → OPEN")
            self.state = CircuitState.OPEN
            self.last_state_change_time = time.time()
            self.failure_count = 0
            self.success_count = 0

    def _close(self):
        """Закрыть circuit breaker"""
        print(f"[CB] HALF-OPEN → CLOSED (система восстановилась)")
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_state_change_time = time.time()

    def _should_attempt_reset(self) -> bool:
        """Время ли попробовать восстановить соединение?"""
        return (time.time() - self.last_state_change_time) >= self.recovery_timeout

    def _time_until_reset(self) -> float:
        """Сколько осталось до попытки восстановления"""
        elapsed = time.time() - self.last_state_change_time
        return max(0, self.recovery_timeout - elapsed)

    def get_status(self) -> dict:
        """Статус circuit breaker"""
        return {
            "state": self.state.value,
            "failures": self.failure_count,
            "successes": self.success_count
        }


# Демонстрация
class UnreliableService:
    def __init__(self, failure_rate=0.6):
        self.failure_rate = failure_rate
        self.request_count = 0

    def call(self):
        import random
        self.request_count += 1

        if random.random() < self.failure_rate:
            raise Exception("Service temporarily unavailable")
        return f"Success #{self.request_count}"


print("=== Circuit Breaker Demo ===\n")

cb = CircuitBreaker(failure_threshold=3, recovery_timeout=5, success_threshold=2)
service = UnreliableService(failure_rate=0.7)

# Посылаем 20 запросов
for i in range(20):
    try:
        result = cb.call(service.call)
        print(f"Запрос #{i+1}: ✓ {result}")
    except Exception as e:
        print(f"Запрос #{i+1}: ✗ {str(e)[:40]}")

    print(f"  Статус: {cb.get_status()}\n")
    time.sleep(0.5)


class CircuitBreakerOpenException(Exception):
    """Исключение когда circuit breaker открыт"""
    pass


---

## 5. Что такое cascading failures и как их предотвратить?

### Зачем спрашивают
Cascading failures — это когда отказ одного компонента вызывает цепочку отказов в других. Вопрос проверяет системное мышление и понимание того, как сбои распространяются через архитектуру.

### Короткий ответ
Cascading failure происходит, когда отказ одного сервиса перегружает зависимые от него сервисы, заставляя их тоже падать. Предотвращение: circuit breakers, timeouts, load shedding, graceful degradation, rate limiting, bulkhead pattern.

### Детальный разбор

**Сценарий cascading failure:**

```
Исходная конфигурация:
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Frontend   │      │  Auth Service│      │  Database    │
│   (10 rps)   │──→   │  (5 rps)     │──→   │  (10 rps)    │
└──────────────┘      └──────────────┘      └──────────────┘


Сценарий падения (без защиты):

1. Database становится медленной (100ms вместо 10ms)
   └─ Auth Service теряет throughput (может обработать только 1 rps)

2. Frontend отправляет 10 rps, Auth может обработать только 1
   └─ Очередь растёт, память Auth исчерпывается

3. Auth Service падает
   └─ Frontend теряет 50% функциональности

4. Frontend продолжает отправлять запросы на мёртвый сервис
   └─ Это высасывает ресурсы frontend'а (threads, connections)

5. Frontend тоже падает
   └─ Полный outage!


Временная шкала:
T=0s:   Database медленная
T=5s:   Auth запросы замораживаются
T=10s:  Auth Service falls
T=15s:  Frontend запросы зависают
T=20s:  Frontend Service falls
T=25s:  Полный outage
```

**Профилактика — Multiple Layers:**

```
Layer 1: Timeouts (не ждать бесконечно)
Layer 2: Circuit Breaker (не отправлять мёртвому сервису)
Layer 3: Bulkhead (изолировать ресурсы)
Layer 4: Rate Limiting (не перегружать
Layer 5: Load Shedding (отклонять запросы заранее)
```

### Пример

```python
import time
import random
from threading import Thread, Lock
from collections import defaultdict

class CascadingFailureSimulation:
    """Симуляция cascading failure и защиты"""

    def __init__(self):
        self.db_response_time = 10  # ms
        self.auth_queue_length = 0
        self.auth_active_threads = 0
        self.auth_max_threads = 5
        self.auth_failures = 0
        self.frontend_queue_length = 0
        self.lock = Lock()

    def simulate_db_slowdown(self):
        """Database становится медленной"""
        self.db_response_time = 100  # вместо 10ms
        print("⚠️  Database: стала медленной (100ms)")

    def simulate_db_recovery(self):
        """Database восстанавливается"""
        self.db_response_time = 10
        print("✓ Database: восстановилась (10ms)")

    def auth_process_request(self, with_protection=False):
        """Auth сервис обрабатывает запрос"""
        with self.lock:
            self.auth_queue_length -= 1
            self.auth_active_threads += 1

        try:
            # Обращаемся к БД
            time.sleep(self.db_response_time / 1000.0)

            with self.lock:
                self.auth_active_threads -= 1
            return True
        except Exception as e:
            with self.lock:
                self.auth_failures += 1
                self.auth_active_threads -= 1
            return False

    def frontend_send_request(self, with_protection=False):
        """Frontend отправляет запрос к Auth"""

        # Защита 1: Load Shedding
        if with_protection:
            with self.lock:
                if self.auth_queue_length > 50:  # очередь переполнена
                    print(f"  Load Shedding: отклонили запрос (очередь: {self.auth_queue_length})")
                    return "rejected"

        with self.lock:
            # Защита 2: Check if Auth is dead
            if with_protection and self.auth_failures > 10:
                print(f"  Circuit Breaker: Auth падёл, отклоняем запрос")
                return "circuit_breaker_open"

            # Защита 3: Bulkhead (не более N запросов одновременно)
            if with_protection and self.auth_queue_length >= self.auth_max_threads * 2:
                print(f"  Bulkhead: ресурсы истощены")
                return "bulkhead_rejected"

            self.auth_queue_length += 1

        # Защита 4: Timeout (не ждать бесконечно)
        start = time.time()
        timeout = 5 if with_protection else 999
        while time.time() - start < timeout:
            with self.lock:
                if self.auth_queue_length <= 0:
                    break
            time.sleep(0.01)

        if time.time() - start >= timeout:
            print(f"  Timeout: Auth не ответила за {timeout}s")
            return "timeout"

        return "success"

    def run_simulation(self, with_protection=False):
        """Симуляция"""
        mode = "WITH PROTECTION" if with_protection else "WITHOUT PROTECTION"
        print(f"\n{'='*60}")
        print(f"SIMULATION: {mode}")
        print(f"{'='*60}\n")

        # Phase 1: нормальная работа
        print("Phase 1: Normal operation (10 requests)")
        for i in range(10):
            self.frontend_send_request(with_protection)
            time.sleep(0.05)

        # Phase 2: БД медленная
        print("\nPhase 2: Database slowdown")
        self.simulate_db_slowdown()
        for i in range(10):
            result = self.frontend_send_request(with_protection)
            print(f"  Request: {result}")
            time.sleep(0.05)

        # Phase 3: БД восстанавливается
        print("\nPhase 3: Database recovery")
        self.simulate_db_recovery()
        for i in range(10):
            result = self.frontend_send_request(with_protection)
            print(f"  Request: {result}")
            time.sleep(0.05)

        print(f"\nFinal Stats:")
        print(f"  Auth queue length: {self.auth_queue_length}")
        print(f"  Auth failures: {self.auth_failures}")


# Запуск
sim1 = CascadingFailureSimulation()
sim1.run_simulation(with_protection=False)

sim2 = CascadingFailureSimulation()
sim2.run_simulation(with_protection=True)
```

### Типичные ошибки
1. **Отсутствие timeouts** — код зависает, ожидая мёртвый сервис
2. **Жёсткие зависимости** — критический сервис падает, всё падает
3. **Нет изоляции ресурсов** — один медленный запрос блокирует всех
4. **Игнорирование очередей** — растущие очереди исчерпывают память

### На интервью
**Как отвечать:**
- Нарисуйте временную шкалу cascading failure
- Перечислите слои защиты (timeouts, CB, bulkhead, rate limit)
- Объясните каждый механизм
- Приведите реальный пример (AWS outage 2011 в EC2)

**Follow-ups:**
- "Как выбрать timeout?" → P95-P99 latency + buffer (обычно 2-3x)
- "Как комбинировать эти механизмы?" → слои, каждый ловит разные сценарии
- "Как мониторить это?" → алерты на queue length, latency percentiles

---

## 6. Как обнаруживать отказы? (heartbeat, leases, failure detectors)

### Зачем спрашивают
Обнаружение отказов — фундамент всей reliability архитектуры. Вопрос проверяет знание различных механизмов и trade-offs между точностью и overhead.

### Короткий ответ
Есть три основных способа: **heartbeat** (периодические сигналы), **leases** (время жизни), **failure detectors** (алгоритмы типа Φ). Heartbeat простой, но может иметь false positives; leases более надёжны; Φ detector адаптируется к сетевым условиям.

### Детальный разбор

**Heartbeat vs Leases:**

```
HEARTBEAT (pull-based):
  ┌─────────┐              ┌─────────┐
  │ Monitor │──ping────→   │ Service │
  │         │←──pong──────│         │
  └─────────┘              └─────────┘

  Если нет ответа за timeout T → сервис упал

LEASES (push-based):
  ┌─────────┐              ┌─────────┐
  │ Service │──heartbeat──→ │ Monitor │
  │         │  (каждые X)   │         │
  └─────────┘              └─────────┘

  Monitor хранит lease с TTL
  Если lease истёкает → сервис упал


Сравнение:
┌──────────────────┬─────────────┬──────────────┐
│ Характеристика   │ Heartbeat   │ Lease        │
├──────────────────┼─────────────┼──────────────┤
│ Направление      │ pull        │ push         │
│ Overhead (alive) │ высокий     │ низкий       │
│ Detection speed  │ медленно     │ быстро       │
│ False positives  │ возможны    │ меньше       │
│ Масштабируемость │ плохо       │ хорошо       │
└──────────────────┴─────────────┴──────────────┘
```

**Φ Accrual Failure Detector (адаптивный):**

```
Идея: вместо бинарного решения (живой/мёртвый)
      возвращаем вероятность отказа (0.0 - 1.0)

Основано на:
- История inter-arrival times между heartbeat'ами
- Статистический анализ задержек
- Адаптирует threshold на основе сетевых условий

Пример:
  Normal heartbeat: каждые 100ms
  Один с задержкой 500ms → φ = 0.2 (вероятность отказа низкая)
  Пять с задержкой 500ms → φ = 0.8 (вероятность отказа высокая)
  Нет heartbeat 2 сек → φ = 0.99 (сервис почти наверняка упал)

Преимущества:
- Адаптируется к сетевым условиям автоматически
- Меньше false positives при нестабильной сети
- Можно использовать разные threshold для разных сценариев
```

### Пример

```python
import time
import math
from collections import deque
from statistics import mean, stdev

class HeartbeatFailureDetector:
    """Классический heartbeat-based detector"""

    def __init__(self, heartbeat_interval: float = 1.0, timeout_multiplier: float = 3.0):
        self.heartbeat_interval = heartbeat_interval
        self.timeout = heartbeat_interval * timeout_multiplier
        self.last_heartbeat = {}
        self.suspected = set()

    def record_heartbeat(self, node_id: str):
        """Записать heartbeat от узла"""
        self.last_heartbeat[node_id] = time.time()
        if node_id in self.suspected:
            self.suspected.remove(node_id)
            print(f"  ✓ {node_id}: восстановился")

    def check_failures(self):
        """Проверить, какие узлы не отвечают"""
        current_time = time.time()
        newly_suspected = []

        for node_id, last_beat in self.last_heartbeat.items():
            elapsed = current_time - last_beat
            if elapsed > self.timeout:
                if node_id not in self.suspected:
                    newly_suspected.append(node_id)
                    self.suspected.add(node_id)
                    print(f"  ✗ {node_id}: заподозрен в падении ({elapsed:.1f}s > {self.timeout:.1f}s timeout)")

        return newly_suspected


class LeaseFailureDetector:
    """Lease-based failure detector"""

    def __init__(self, lease_duration: float = 5.0):
        self.lease_duration = lease_duration
        self.leases = {}  # node_id -> (timestamp, duration)
        self.suspected = set()

    def renew_lease(self, node_id: str):
        """Узел renew'ит свой lease"""
        self.leases[node_id] = (time.time(), self.lease_duration)
        if node_id in self.suspected:
            self.suspected.remove(node_id)
            print(f"  ✓ {node_id}: renewed lease")

    def check_failures(self):
        """Проверить expired leases"""
        current_time = time.time()
        newly_suspected = []

        for node_id, (lease_time, duration) in self.leases.items():
            if current_time - lease_time > duration:
                if node_id not in self.suspected:
                    newly_suspected.append(node_id)
                    self.suspected.add(node_id)
                    print(f"  ✗ {node_id}: lease expired")

        return newly_suspected


class PhiAccrualFailureDetector:
    """Adaptive Φ (Phi) Accrual Failure Detector"""

    def __init__(self, max_history: int = 100, threshold: float = 0.5):
        self.max_history = max_history
        self.threshold = threshold
        self.inter_arrival_times = {}  # node_id -> deque of times
        self.last_timestamp = {}
        self.suspected = set()

    def record_heartbeat(self, node_id: str):
        """Записать heartbeat и вычислить φ"""
        current_time = time.time()

        if node_id not in self.last_timestamp:
            self.last_timestamp[node_id] = current_time
            self.inter_arrival_times[node_id] = deque()
            return

        # Вычислить inter-arrival time
        inter_arrival = current_time - self.last_timestamp[node_id]
        self.last_timestamp[node_id] = current_time

        # Добавить в историю
        history = self.inter_arrival_times[node_id]
        history.append(inter_arrival)
        if len(history) > self.max_history:
            history.popleft()

        # Очистить подозрение если восстановился
        if node_id in self.suspected:
            self.suspected.remove(node_id)
            print(f"  ✓ {node_id}: recovered (φ reset)")

    def get_phi(self, node_id: str) -> float:
        """Вычислить Φ вероятность отказа"""
        if node_id not in self.last_timestamp:
            return 0.0

        history = self.inter_arrival_times.get(node_id, deque())
        if len(history) < 3:
            return 0.0

        current_time = time.time()
        now = current_time - self.last_timestamp[node_id]

        # Вычислить статистику
        mean_inter_arrival = mean(history)
        if len(history) > 1:
            std_inter_arrival = stdev(history)
        else:
            std_inter_arrival = mean_inter_arrival / 2

        # Φ = -log10( P(no heartbeat received for now seconds) )
        # Упрощённо: смотрим, сколько σ отклонений от нормы
        if std_inter_arrival == 0:
            return 0.0

        deviations = (now - mean_inter_arrival) / std_inter_arrival
        phi = max(0, deviations / 3.0)  # нормализуем

        return phi

    def check_failures(self) -> list:
        """Проверить на основе Φ"""
        newly_suspected = []

        for node_id in self.last_timestamp.keys():
            phi = self.get_phi(node_id)

            if phi > self.threshold:
                if node_id not in self.suspected:
                    newly_suspected.append(node_id)
                    self.suspected.add(node_id)
                    print(f"  ✗ {node_id}: suspected (φ={phi:.2f} > {self.threshold})")
            else:
                if node_id in self.suspected:
                    self.suspected.discard(node_id)

        return newly_suspected

    def get_status(self) -> dict:
        """Статус всех узлов"""
        status = {}
        for node_id in self.last_timestamp.keys():
            phi = self.get_phi(node_id)
            status[node_id] = {
                "phi": phi,
                "suspected": node_id in self.suspected
            }
        return status


# Демонстрация
print("=== Failure Detection Comparison ===\n")

# Создаём трёх детекторов
hb_detector = HeartbeatFailureDetector(heartbeat_interval=1.0, timeout_multiplier=3.0)
lease_detector = LeaseFailureDetector(lease_duration=5.0)
phi_detector = PhiAccrualFailureDetector(threshold=0.5)

nodes = ["node1", "node2", "node3"]

# Нормальная работа
print("Phase 1: Normal operation")
for node_id in nodes:
    hb_detector.record_heartbeat(node_id)
    lease_detector.renew_lease(node_id)
    phi_detector.record_heartbeat(node_id)

time.sleep(2)

# Node2 падает (перестаёт отправлять heartbeats)
print("\nPhase 2: Node2 fails")
for node_id in ["node1", "node3"]:
    hb_detector.record_heartbeat(node_id)
    lease_detector.renew_lease(node_id)
    phi_detector.record_heartbeat(node_id)

time.sleep(4)

print("\nChecking for failures...")
print("Heartbeat detector:")
hb_detector.check_failures()

print("Lease detector:")
lease_detector.check_failures()

print("Φ detector status:")
for node_id, status in phi_detector.get_status().items():
    print(f"  {node_id}: φ={status['phi']:.2f}, suspected={status['suspected']}")
```

### Типичные ошибки
1. **Слишком короткий timeout** — много false positives при сетевых скачках
2. **Слишком длинный timeout** — медленное обнаружение реальных отказов
3. **Игнорирование clock skew** — если часы расходятся, детектор нарушается
4. **Heartbeat как единственный способ** — не работает для мёртвых узлов, которые можно разбудить

### На интервью
**Как отвечать:**
- Объясните три механизма: heartbeat, lease, Φ
- Сравните их pros/cons
- Нарисуйте диаграмму heartbeat/lease
- Упомяните trade-off между false positives и detection speed

**Follow-ups:**
- "Как выбрать timeout?" → зависит от expected latency + network jitter
- "Что если clock skew?" → использовать monotonic clocks, leases вместо heartbeat
- "Как это масштабируется?" → Φ detector лучше, чем heartbeat для больших кластеров

---

## 7. Как реализовать graceful degradation?

### Зачем спрашивают
Graceful degradation — стратегия, при которой система продолжает работать с ограниченной функциональностью при отказах. Вопрос проверяет способность проектировать системы, которые деградируют красиво, а не просто падают.

### Короткий ответ
Graceful degradation означает, что при отказе компонента система отключает зависящие от него функции, но основной сервис продолжает работать. Реализуется через: feature flags, fallbacks, caching, offline mode, reduced functionality.

### Детальный разбор

**Уровни функциональности:**

```
100% функциональности (all systems healthy)
  ├─ User Service ✓
  ├─ Recommendation Engine ✓
  ├─ Payment Service ✓
  ├─ Analytics ✓
  └─ Search Index ✓

↓ Recommendation Engine падает

80% функциональности (gracefully degraded)
  ├─ User Service ✓
  ├─ Recommendation Engine ✗ (show cached/popular items)
  ├─ Payment Service ✓
  ├─ Analytics ✓ (без real-time insights)
  └─ Search Index ✓

↓ Payment Service падает

60% функциональность (heavy degradation)
  ├─ User Service ✓
  ├─ Recommendation Engine ✗ (cached)
  ├─ Payment Service ✗ (read-only, no new purchases)
  ├─ Analytics ✓ (basic)
  └─ Search Index ✓

↓ User Service падает

0% функциональности
  └─ System unavailable
```

**Стратегии graceful degradation:**

```
1. CACHING
   - Serve stale data when service is down
   - Example: show cached recommendations

2. FALLBACK
   - Use alternative implementation
   - Example: if ML-based ranking down, use simple popularity ranking

3. FEATURE FLAGS
   - Disable non-critical features
   - Example: disable real-time notifications if queue is backed up

4. READ-ONLY MODE
   - Allow reads but not writes
   - Example: allow viewing products, but not purchasing

5. PARTIAL DATA
   - Return subset of requested data
   - Example: return top 10 results instead of top 100

6. OFFLINE MODE
   - Work with local data
   - Example: mobile app uses local cache when network down
```

### Пример

```python
from enum import Enum
from typing import Optional, List, Dict
import time

class ServiceHealth(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    OFFLINE = "offline"

class RecommendationService:
    """Recommendation service с graceful degradation"""

    def __init__(self):
        self.health = ServiceHealth.HEALTHY
        self.cache = {}
        self.feature_enabled = True

    def get_recommendations(self, user_id: int, count: int = 10) -> Dict:
        """Получить рекомендации с graceful degradation"""

        if self.health == ServiceHealth.OFFLINE:
            # Fallback 1: Serve cached recommendations
            if user_id in self.cache:
                print(f"  📦 Using cached recommendations for user {user_id}")
                return {
                    "items": self.cache[user_id][:count],
                    "source": "cache"
                }
            else:
                # Fallback 2: Return popular items (no personalization)
                print(f"  📦 Service offline, returning popular items")
                return {
                    "items": self.get_popular_items(count),
                    "source": "fallback"
                }

        elif self.health == ServiceHealth.DEGRADED:
            if not self.feature_enabled:
                # Feature disabled, return partial results
                print(f"  ⚠️  Feature disabled, returning reduced results")
                return {
                    "items": self.get_quick_recommendations(user_id, count // 2),
                    "source": "degraded"
                }

            # Try ML-based, fallback to popularity
            try:
                return {
                    "items": self.get_ml_recommendations(user_id, count),
                    "source": "ml"
                }
            except Exception:
                print(f"  ⚠️  ML recommendation failed, using popularity")
                return {
                    "items": self.get_popular_items(count),
                    "source": "fallback"
                }

        else:  # HEALTHY
            print(f"  ✓ Serving ML-based recommendations")
            result = {
                "items": self.get_ml_recommendations(user_id, count),
                "source": "ml"
            }
            # Cache for future use
            self.cache[user_id] = result["items"]
            return result

    def get_ml_recommendations(self, user_id: int, count: int) -> List[str]:
        """ML-based recommendations (может быть медленным)"""
        if self.health == ServiceHealth.OFFLINE:
            raise Exception("Service is offline")

        # Симуляция работы ML model
        time.sleep(0.1)
        return [f"item_{user_id}_ml_{i}" for i in range(count)]

    def get_quick_recommendations(self, user_id: int, count: int) -> List[str]:
        """Быстрые рекомендации (из simple rules)"""
        return [f"item_{user_id}_quick_{i}" for i in range(count)]

    def get_popular_items(self, count: int) -> List[str]:
        """Популярные товары (всем одно и то же)"""
        return [f"popular_item_{i}" for i in range(count)]

    def set_health(self, health: ServiceHealth):
        """Изменить статус здоровья"""
        self.health = health
        print(f"\n  Service health: {health.value.upper()}")

    def disable_feature(self):
        """Отключить feature flag"""
        self.feature_enabled = False
        print(f"  Feature disabled")

    def enable_feature(self):
        """Включить feature flag"""
        self.feature_enabled = True
        print(f"  Feature enabled")


class PaymentService:
    """Payment service с read-only fallback"""

    def __init__(self):
        self.health = ServiceHealth.HEALTHY
        self.order_cache = {}

    def purchase(self, user_id: int, item_id: str, amount: float) -> Dict:
        """Обработать платёж"""

        if self.health == ServiceHealth.OFFLINE:
            print(f"  Payment service is offline")
            return {
                "success": False,
                "message": "Payment service temporarily unavailable",
                "mode": "offline"
            }

        elif self.health == ServiceHealth.DEGRADED:
            print(f"  Payment service degraded, entering READ-ONLY mode")
            return {
                "success": False,
                "message": "Can't process new purchases. Try again later.",
                "mode": "read_only"
            }

        else:  # HEALTHY
            print(f"  Processing payment for user {user_id}")
            order = {
                "order_id": f"order_{user_id}_{int(time.time())}",
                "user_id": user_id,
                "item_id": item_id,
                "amount": amount
            }
            self.order_cache[user_id] = order
            return {
                "success": True,
                "order_id": order["order_id"],
                "mode": "normal"
            }

    def get_order_history(self, user_id: int) -> List[Dict]:
        """Получить историю заказов (работает даже в OFFLINE)"""

        if user_id in self.order_cache:
            print(f"  Returning cached order history for user {user_id}")
            return [self.order_cache[user_id]]
        return []

    def set_health(self, health: ServiceHealth):
        self.health = health
        print(f"\n  Payment service health: {health.value.upper()}")


# Демонстрация graceful degradation
print("=" * 70)
print("GRACEFUL DEGRADATION DEMO")
print("=" * 70)

# Scenario 1: All healthy
print("\n[SCENARIO 1] All services healthy")
rec_service = RecommendationService()
rec_service.set_health(ServiceHealth.HEALTHY)
result = rec_service.get_recommendations(user_id=123)
print(f"  Result: {result['source']} - {result['items'][:3]}...")

# Scenario 2: Recommendation service degraded
print("\n[SCENARIO 2] Recommendation service degraded")
rec_service.set_health(ServiceHealth.DEGRADED)
result = rec_service.get_recommendations(user_id=123)
print(f"  Result: {result['source']} - {result['items'][:3]}...")

# Scenario 3: Feature flag disabled
print("\n[SCENARIO 3] Feature flag disabled")
rec_service.disable_feature()
result = rec_service.get_recommendations(user_id=123)
print(f"  Result: {result['source']} - {result['items'][:3]}...")

# Scenario 4: Service completely offline
print("\n[SCENARIO 4] Recommendation service offline")
rec_service.set_health(ServiceHealth.OFFLINE)
result = rec_service.get_recommendations(user_id=123)
print(f"  Result: {result['source']} - {result['items'][:3]}...")

# Scenario 5: Payment service degradation
print("\n[SCENARIO 5] Payment service operational")
payment = PaymentService()
payment.set_health(ServiceHealth.HEALTHY)
result = payment.purchase(user_id=456, item_id="item_1", amount=99.99)
print(f"  Result: {result['mode']} - {result['success']}")

print("\n[SCENARIO 6] Payment service degraded (READ-ONLY)")
payment.set_health(ServiceHealth.DEGRADED)
result = payment.purchase(user_id=456, item_id="item_2", amount=49.99)
print(f"  Result: {result['mode']} - {result['success']}")

print("\nBut can still READ:")
history = payment.get_order_history(user_id=456)
print(f"  Order history: {len(history)} orders")
```

### Типичные ошибки
1. **Недостаточное кеширование** — при падении сервиса нет fallback данных
2. **Все-или-ничего подход** — система полностью падает вместо деградации
3. **Отсутствие feature flags** — нельзя отключить нужную функцию без deploy
4. **Сложные fallback механизмы** — fallback сам становится single point of failure

### На интервью
**Как отвечать:**
- Объясните концепцию деградации
- Дайте примеры (рекомендации → популярные, поиск → кеш)
- Упомяните техники: caching, fallback, feature flags, read-only
- Нарисуйте уровни функциональности

**Follow-ups:**
- "Как тестировать graceful degradation?" → chaos engineering, injecting failures
- "Как пользователям об этом узнать?" → UI индикаторы, feature badges
- "Как сочетается с circuit breaker?" → CB открывается → graceful degradation включается

---

## 8. Как работает failover в базах данных?

### Зачем спрашивают
Failover в БД — критический механизм для high availability. Вопрос проверяет понимание replication, consensus, и как восстанавливается система после отказа primary.

### Короткий ответ
Failover — это процесс автоматического переключения на backup/secondary узел при падении primary. Требует: health monitoring, automatic detection, quorum voting для избрания нового primary, данные consistency гарантии, minimal downtime.

### Детальный разбор

**Архитектура Primary-Secondary:**

```
НОРМАЛЬНО (Primary healthy):
┌──────────────────────────────────────────────────────┐
│                                                        │
│   ┌─────────────┐        ┌─────────────┐            │
│   │  Primary DB │─ rep ──→│ Secondary1  │            │
│   │  (leader)   │        │             │            │
│   └─────────────┘        └─────────────┘            │
│        ▲                                              │
│        │ writes                    ┌─────────────┐  │
│   ┌────┴─────────┐                 │ Secondary2  │  │
│   │  Application │──reads/writes──→│ (replica)   │  │
│   └──────────────┘                 └─────────────┘  │
│                                                       │
└──────────────────────────────────────────────────────┘

PRIMARY FAILS:
┌──────────────────────────────────────────────────────┐
│                                                        │
│   ┌─────────────┐        ┌─────────────┐            │
│   │  Primary DB │ ✗ rep ─→│ Secondary1  │            │
│   │  (DEAD)     │   XXX   │             │            │
│   └─────────────┘        └─────────────┘            │
│                              (becomes new leader)    │
│                          ┌─────────────┐             │
│   ┌────────────────┐     │ Secondary2  │             │
│   │  Application   │────→│ (replica)   │             │
│   │ (redirected)   │     └─────────────┘             │
│   └────────────────┘                                 │
│                                                       │
└──────────────────────────────────────────────────────┘
```

**Фазы failover:**

```
1. DETECTION (обнаружение падения primary)
   - Health check fails (heartbeat timeout)
   - Connection refused
   - Query timeout

2. PROMOTION (выборы нового primary)
   - Secondaries голосуют
   - Выбирают узел с самыми свежими данными
   - Новый primary принимает writes

3. UPDATE (обновление клиентов)
   - DNS update (может быть медленным)
   - Connection pooler reconfiguration
   - Application retry (automatic)

4. RESYNC (синхронизация оставшихся узлов)
   - Бывший primary может вернуться
   - Запускается catch-up replication
```

### Пример

```python
import time
from enum import Enum
from typing import Optional, List
import threading

class NodeRole(Enum):
    PRIMARY = "primary"
    SECONDARY = "secondary"
    UNKNOWN = "unknown"

class DatabaseNode:
    """Single database node"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.role = NodeRole.UNKNOWN
        self.is_alive = True
        self.data = {}
        self.replication_offset = 0
        self.last_heartbeat = time.time()

    def write(self, key: str, value: str) -> bool:
        """Записать данные (только primary)"""
        if self.role != NodeRole.PRIMARY:
            print(f"  {self.node_id}: Cannot write, not primary")
            return False

        if not self.is_alive:
            print(f"  {self.node_id}: Node is dead")
            return False

        self.data[key] = value
        self.replication_offset += 1
        print(f"  {self.node_id}: Written {key}={value} (offset: {self.replication_offset})")
        return True

    def read(self, key: str) -> Optional[str]:
        """Читать данные"""
        if not self.is_alive:
            return None

        return self.data.get(key)

    def replicate_from(self, primary: "DatabaseNode"):
        """Получить обновления от primary"""
        if not self.is_alive or self.role == NodeRole.PRIMARY:
            return

        # Catch-up replication
        for key, value in primary.data.items():
            if key not in self.data:
                self.data[key] = value
                self.replication_offset = primary.replication_offset

    def promote_to_primary(self):
        """Стать primary"""
        self.role = NodeRole.PRIMARY
        print(f"  {self.node_id}: Promoted to PRIMARY")

    def demote_to_secondary(self):
        """Вернуться в secondary"""
        self.role = NodeRole.SECONDARY
        print(f"  {self.node_id}: Demoted to SECONDARY")

    def heartbeat(self):
        """Обновить heartbeat"""
        self.last_heartbeat = time.time()

    def is_healthy(self, timeout: float = 5.0) -> bool:
        """Проверить здоровье узла"""
        if not self.is_alive:
            return False

        elapsed = time.time() - self.last_heartbeat
        return elapsed < timeout


class FailoverManager:
    """Управляет failover в replicated DB"""

    def __init__(self, nodes: List[DatabaseNode]):
        self.nodes = nodes
        self.primary: Optional[DatabaseNode] = None
        self.secondaries: List[DatabaseNode] = []
        self.initialize()

    def initialize(self):
        """Инициализировать кластер"""
        if self.nodes:
            self.primary = self.nodes[0]
            self.primary.promote_to_primary()
            self.secondaries = self.nodes[1:]
            for secondary in self.secondaries:
                secondary.demote_to_secondary()

    def heartbeat_check(self):
        """Периодическая проверка здоровья"""
        for node in self.nodes:
            if node != self.primary:
                continue

            if not node.is_healthy(timeout=3.0):
                print(f"\n⚠️  PRIMARY {node.node_id} is not responding!")
                self.trigger_failover()
                return

        # Обновить heartbeat primary
        if self.primary:
            self.primary.heartbeat()

    def trigger_failover(self):
        """Запустить failover"""
        if not self.primary:
            print("No primary to fail over from")
            return

        print(f"\n[FAILOVER TRIGGERED]")
        old_primary = self.primary

        # Phase 1: Выбрать нового primary
        print(f"  Phase 1: Electing new primary")
        new_primary = self.elect_new_primary()

        if not new_primary:
            print(f"  ✗ No suitable replica found!")
            return

        # Phase 2: Promote new primary
        print(f"  Phase 2: Promoting {new_primary.node_id}")
        new_primary.promote_to_primary()
        self.primary = new_primary

        # Phase 3: Update secondaries
        print(f"  Phase 3: Updating secondaries")
        for secondary in self.secondaries:
            if secondary != new_primary:
                secondary.demote_to_secondary()

        print(f"  ✓ Failover complete! New primary: {new_primary.node_id}")

    def elect_new_primary(self) -> Optional[DatabaseNode]:
        """Выбрать новый primary (здоровый узел с самыми свежими данными)"""
        candidates = []

        for node in self.secondaries:
            if node.is_healthy(timeout=3.0):
                candidates.append(node)

        if not candidates:
            return None

        # Выбрать узел с максимальным replication offset
        return max(candidates, key=lambda n: n.replication_offset)

    def write(self, key: str, value: str) -> bool:
        """Написать данные"""
        if not self.primary:
            print("No primary available")
            return False

        return self.primary.write(key, value)

    def read(self, key: str) -> Optional[str]:
        """Читать данные (может быть eventual consistency)"""
        if not self.primary:
            return None

        return self.primary.read(key)

    def replicate(self):
        """Запустить репликацию от primary к secondaries"""
        if not self.primary:
            return

        for secondary in self.secondaries:
            secondary.replicate_from(self.primary)

    def status(self):
        """Вывести статус кластера"""
        print(f"\n[CLUSTER STATUS]")
        for node in self.nodes:
            role = "PRIMARY" if node == self.primary else "SECONDARY"
            alive = "✓ ALIVE" if node.is_alive else "✗ DEAD"
            print(f"  {node.node_id}: {role} - {alive} (data: {len(node.data)} keys, offset: {node.replication_offset})")


# Демонстрация
print("=" * 70)
print("DATABASE FAILOVER DEMO")
print("=" * 70)

# Создаём кластер из 3 узлов
nodes = [
    DatabaseNode("db1"),
    DatabaseNode("db2"),
    DatabaseNode("db3")
]

manager = FailoverManager(nodes)
manager.status()

# Нормальная работа
print("\n[PHASE 1] Normal operations")
manager.write("user:1", "alice")
manager.write("user:2", "bob")
manager.replicate()
manager.status()

print(f"\nReading data:")
print(f"  user:1 = {manager.read('user:1')}")
print(f"  user:2 = {manager.read('user:2')}")

# Primary падает
print("\n[PHASE 2] Primary fails")
manager.primary.is_alive = False
manager.heartbeat_check()

manager.status()

# Попытка написать на новый primary
print("\n[PHASE 3] Write to new primary")
manager.write("user:3", "charlie")
manager.replicate()

print(f"\nReading data:")
print(f"  user:1 = {manager.read('user:1')}")
print(f"  user:3 = {manager.read('user:3')}")
```

### Типичные ошибки
1. **Data loss** — promotion узла с неполными данными
2. **Split brain** — обе primary и secondary принимают writes (требуется quorum)
3. **Медленное обнаружение** — долгий timeout = долгий outage
4. **Не обновлены клиенты** — приложение ещё пишет на мёртвый primary

### На интервью
**Как отвечать:**
- Объясните архитектуру primary-secondary
- Обрисуйте фазы failover: detection, promotion, update, resync
- Упомяните data loss risks
- Покажите код простого failover manager

**Follow-ups:**
- "Как избежать data loss?" → quorum writes, synchronous replication
- "Как быстро обнаружить отказ?" → health checks, timeout tuning
- "Что если оба упадут?" → backup replication, multi-datacenter
- "Как с consistency при failover?" → eventual consistency vs strong consistency

---

## 9. Как обрабатывать clock skew?

### Зачем спрашивают
Clock skew (расхождение часов между узлами) может разрушить многие предположения распределённых алгоритмов. Вопрос проверяет внимание к деталям и знание real-world проблем.

### Короткий ответ
Clock skew — это когда часы на разных узлах показывают разное время. Это нарушает ordering событий, lease-based механизмы, timeouts. Решения: synchronize clocks (NTP), use logical/vector clocks, avoid wall-clock comparisons, use monotonic clocks.

### Детальный разбор

**Проблемы clock skew:**

```
Сценарий:
- Node A часы: 10:00:00
- Node B часы: 10:00:05 (на 5 сек впереди)

Что может пойти не так:

1. LEASE EXPIRATION
   Node A: выдал lease до 10:00:30 (узел B)
   Node B: уже 10:00:05
   Node B видит lease как valid на 25 сек
   Но пока Node A отправляет новые requests... они идут куда?

2. DISTRIBUTED TRANSACTION ORDERING
   Event A на Node A: 10:00:00
   Event B на Node B: 10:00:05
   Но Event B случился ПЕРЕД Event A в реальности!
   Если полагаемся на timestamp для ordering → неправильный порядок

3. TIMEOUT CALCULATIONS
   Node A: отправляет heartbeat в 10:00:00
   Node B: получает в 10:00:05 (своё время)
   Если B ожидает heartbeat каждые 10 сек, он может считать
   Node A мёртвым раньше, чем он должен

4. LEADER ELECTION
   Двое узлов думают, что они лидеры на разных сегментах времени
```

**Решение 1: Logical Clocks (Lamport)**

```
Вместо wall-clock времени используем logical counter:

Event ordering:
- Node A: Counter=1, sends message
  Node B: Counter=1, receives message
  Node B: Counter=max(1,1)+1=2, processes message
  Node B: Counter=3, sends response
  Node A: Counter=max(1,3)+1=4, receives response

Гарантия: если Event A → Event B (causally),
то Lamport(A) < Lamport(B)

Не зависит от wall-clock времени!
```

**Решение 2: NTP (Network Time Protocol)**

```
Синхронизирует часы между узлами:

Precision: +/- 1 ms (хорошие условия)
Accuracy: +/- 100 ms (типичная сеть)

Использование:
- Большинство Linux систем используют chronyd/ntpd
- Cloud провайдеры (AWS, GCP) гарантируют синхронизацию

Ограничение: никогда не будет идеальной синхронизации!
```

**Решение 3: Monotonic Clocks**

```
Вместо wall-clock используем монотонный счётчик:

wall_clock может прыгать назад (из-за NTP adjustment)
monotonic_clock только растёт, никогда не уменьшается

Используется для:
- Timeouts: deadline = now() + 5s (где now = monotonic)
- Latency measurement: end_time - start_time (monotonic)
- Lease expiration: не полагаться на wall-clock
```

### Пример

```python
import time
from collections import defaultdict
from typing import Optional

class LogicalClock:
    """Lamport Logical Clock"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.counter = 0

    def increment(self) -> int:
        """Увеличить счётчик на локальное событие"""
        self.counter += 1
        return self.counter

    def receive_message(self, received_timestamp: int) -> int:
        """Обновить счётчик при получении сообщения"""
        self.counter = max(self.counter, received_timestamp) + 1
        return self.counter

    def timestamp(self) -> int:
        """Получить текущий logical timestamp"""
        return self.increment()


class VectorClock:
    """Vector Clock для causality tracking"""

    def __init__(self, nodes: list):
        self.clock = {node: 0 for node in nodes}
        self.node_id = None

    def set_node_id(self, node_id: str):
        self.node_id = node_id

    def increment(self) -> dict:
        """Увеличить на локальное событие"""
        self.clock[self.node_id] += 1
        return self.clock.copy()

    def receive(self, received_clock: dict) -> dict:
        """Обновить при получении сообщения"""
        for node, timestamp in received_clock.items():
            self.clock[node] = max(self.clock[node], timestamp)
        self.clock[self.node_id] += 1
        return self.clock.copy()

    def happened_before(self, clock_a: dict, clock_b: dict) -> bool:
        """Проверить, что A случилось перед B"""
        all_less = all(clock_a[node] <= clock_b[node] for node in clock_a)
        some_less = any(clock_a[node] < clock_b[node] for node in clock_a)
        return all_less and some_less

    def concurrent(self, clock_a: dict, clock_b: dict) -> bool:
        """Проверить, что события конкурентны"""
        a_less_b = any(clock_a[node] < clock_b[node] for node in clock_a)
        b_less_a = any(clock_b[node] < clock_a[node] for node in clock_a)
        return a_less_b and b_less_a


class MonotonicTimer:
    """Monotonic timer для timeouts (не зависит от NTP adjustments)"""

    def __init__(self):
        # В Python используем time.monotonic()
        self.reference_time = time.monotonic()

    def elapsed(self) -> float:
        """Получить прошедшее время (всегда растёт)"""
        return time.monotonic() - self.reference_time

    def deadline_in(self, seconds: float) -> float:
        """Вычислить deadline с гарантией монотонности"""
        return self.elapsed() + seconds

    def is_deadline_exceeded(self, deadline: float) -> bool:
        """Проверить, прошёл ли deadline"""
        return self.elapsed() > deadline


class ClockSkewDemo:
    """Демонстрация clock skew и его влияния"""

    @staticmethod
    def simulate_clock_skew():
        """Симуляция двух узлов с разными часами"""
        print("[Clock Skew Impact]\n")

        # Node A и B имеют разные time offsets
        node_a_offset = 0  # правильные часы
        node_b_offset = 5  # на 5 сек впереди

        wall_time = 10.0

        node_a_time = wall_time + node_a_offset
        node_b_time = wall_time + node_b_offset

        print(f"Real time: {wall_time}s")
        print(f"Node A clock: {node_a_time}s (offset: {node_a_offset}s)")
        print(f"Node B clock: {node_b_time}s (offset: {node_b_offset}s)")

        print(f"\nLease expiration problem:")
        lease_duration = 10
        lease_expires_on_a = node_a_time + lease_duration
        print(f"  Node A issues lease expiring at {lease_expires_on_a}s")
        print(f"  Node B sees lease as valid until {lease_expires_on_a + node_a_offset - node_b_offset}s")
        print(f"  ⚠️  Discrepancy: {abs(node_b_offset - node_a_offset)}s!")

    @staticmethod
    def logical_clock_example():
        """Lamport logical clocks для message ordering"""
        print("\n[Logical Clock Example]\n")

        clock_a = LogicalClock("A")
        clock_b = LogicalClock("B")

        # Событие на A
        msg_ts = clock_a.timestamp()
        print(f"A sends message with timestamp {msg_ts}")

        # B получает сообщение
        b_ts = clock_b.receive_message(msg_ts)
        print(f"B receives message, updates clock to {b_ts}")

        # Событие на B
        b_local = clock_b.timestamp()
        print(f"B local event gets timestamp {b_local}")

        print(f"\nGuarantee: All events have unique, causal ordering")
        print(f"  regardless of wall-clock time!")

    @staticmethod
    def vector_clock_example():
        """Vector clocks для causality detection"""
        print("\n[Vector Clock Example]\n")

        nodes = ["A", "B", "C"]
        clock_a = VectorClock(nodes)
        clock_a.set_node_id("A")

        clock_b = VectorClock(nodes)
        clock_b.set_node_id("B")

        # Event на A
        event_a1 = clock_a.increment()
        print(f"A: local event → {event_a1}")

        # A отправляет B
        msg_ts = event_a1.copy()
        event_a2 = clock_a.increment()
        print(f"A: sends message → {event_a2}")

        # B получает
        event_b1 = clock_b.receive(msg_ts)
        print(f"B: receives → {event_b1}")

        # Проверить causality
        print(f"\nCausality checks:")
        print(f"  A1 happened before B1: {clock_a.happened_before(event_a1, event_b1)}")

    @staticmethod
    def monotonic_timer_example():
        """Monotonic timers для надёжных timeouts"""
        print("\n[Monotonic Timer Example]\n")

        timer = MonotonicTimer()

        print(f"Starting deadline: 2 seconds")
        deadline = timer.deadline_in(2.0)

        for i in range(5):
            elapsed = timer.elapsed()
            exceeded = timer.is_deadline_exceeded(deadline)
            status = "EXCEEDED" if exceeded else "VALID"

            print(f"  T={elapsed:.2f}s: deadline {deadline:.2f}s: {status}")
            time.sleep(0.5)

        print(f"\n✓ Monotonic timer never goes backward,")
        print(f"  reliable regardless of system clock adjustments")


# Запуск демонстрации
ClockSkewDemo.simulate_clock_skew()
ClockSkewDemo.logical_clock_example()
ClockSkewDemo.vector_clock_example()
ClockSkewDemo.monotonic_timer_example()
```

### Типичные ошибки
1. **Полагаться на wall-clock для ordering** — используйте logical/vector clocks
2. **Использовать wall-clock для timeouts** — используйте monotonic clocks
3. **Игнорировать NTP adjustments** — часы могут прыгнуть назад
4. **Не синхронизировать часы** — clock skew растёт и разрушает алгоритмы

### На интервью
**Как отвечать:**
- Объясните, что такое clock skew и почему он опасен
- Дайте примеры (lease expiration, event ordering)
- Упомяните три подхода: NTP, logical clocks, monotonic clocks
- Покажите код logical/vector clock

**Follow-ups:**
- "Как использовать timestamp'ы безопасно?" → logical clocks, не wall-clock
- "Как выбрать between leases и heartbeats?" → при skew предпочесть logical mechanisms
- "Что делать с NTP?" → synchronize + allow margin of error

---

## 10. Как проектировать системы с учётом отказов? (Design for Failure)

### Зачем спрашивают
Это итоговый вопрос, который проверяет holistic understanding fault tolerance. Вопрос требует знания всех предыдущих концепций и способности их комбинировать в систему.

### Короткий ответ
Design for Failure — это систематический подход: (1) идентифицировать режимы отказа, (2) проектировать изоляцию отказов, (3) использовать redundancy, (4) планировать graceful degradation, (5) мониторить и алертировать, (6) тестировать с chaos engineering.

### Детальный разбор

**Принципы Design for Failure:**

```
1. ASSUME EVERYTHING FAILS
   - Не доверяйте сети, дискам, CPU, памяти
   - Предположите, что любой компонент может упасть

2. REDUNDANCY AT EVERY LAYER
   - Несколько реплик
   - Несколько датацентров
   - Несколько сервис инстансов

3. DETECT FAILURES FAST
   - Health checks
   - Monitoring
   - Alerting

4. RECOVER AUTOMATICALLY
   - Failover
   - Self-healing
   - Restart механизмы

5. DEGRADE GRACEFULLY
   - Partial functionality
   - Fallbacks
   - Read-only mode

6. TEST CONTINUOUSLY
   - Chaos engineering
   - Failure injection
   - Game days
```

**Чеклист проектирования:**

```
АРХИТЕКТУРА
[ ] Нет single point of failure
[ ] Компоненты независимы
[ ] Decoupled через queues/events
[ ] Multiple zones/regions
[ ] Load balancing with healthchecks

ДАННЫЕ
[ ] Replicated (≥3 copies для critical data)
[ ] Backups в отдельном месте
[ ] Data consistency defined (strong/eventual)
[ ] Atomic writes где нужны
[ ] Idempotent operations

СЕТЬ
[ ] Timeout на все network calls
[ ] Retry с exponential backoff
[ ] Circuit breaker для external dependencies
[ ] Rate limiting
[ ] Bulkhead isolation (separate pools per service)

ОБНАРУЖЕНИЕ
[ ] Health check endpoints
[ ] Monitoring (metrics, logs)
[ ] Alerting на critical events
[ ] Root cause analysis process

ВОССТАНОВЛЕНИЕ
[ ] Automatic failover (quorum-based)
[ ] Self-healing (restart crashed services)
[ ] Graceful degradation (feature flags)
[ ] Rollback механизм

ТЕСТИРОВАНИЕ
[ ] Chaos engineering (kill random processes)
[ ] Failure scenario exercises ("game days")
[ ] Load testing
[ ] Recovery time measurement

ДОКУМЕНТАЦИЯ
[ ] Known limitations
[ ] SLA и RTO/RPO targets
[ ] Recovery procedures
[ ] Escalation paths
```

### Пример

```python
from enum import Enum
from typing import Optional, Dict, List
import time
import random
from dataclasses import dataclass
from collections import deque

@dataclass
class ServiceMetrics:
    """Метрики сервиса"""
    request_count: int = 0
    error_count: int = 0
    latency_p50: float = 0
    latency_p99: float = 0
    available_instances: int = 0
    degraded: bool = False

    def error_rate(self) -> float:
        if self.request_count == 0:
            return 0
        return self.error_count / self.request_count


class ResilientService:
    """Сервис спроектированный с учётом отказов"""

    def __init__(self, name: str, min_instances: int = 3):
        self.name = name
        self.instances = [f"{name}_instance_{i}" for i in range(min_instances)]
        self.metrics = ServiceMetrics()
        self.circuit_breaker_open = False
        self.health_check_results = deque(maxlen=10)
        self.failures = 0

    def design_principle_1_assume_failure(self):
        """Принцип 1: Предположим, что всё может упасть"""
        print(f"[{self.name}] Design Principle 1: Assume Everything Fails")
        print(f"  ✓ Timeout on all network calls")
        print(f"  ✓ Retry logic with exponential backoff")
        print(f"  ✓ Circuit breaker enabled")
        print(f"  ✓ Cascading failure prevention")

    def design_principle_2_redundancy(self):
        """Принцип 2: Redundancy на каждом уровне"""
        print(f"\n[{self.name}] Design Principle 2: Redundancy")
        print(f"  ✓ Deployed on {len(self.instances)} instances")
        print(f"  ✓ Data replicated across 3+ zones")
        print(f"  ✓ Load balancing with health checks")
        print(f"  ✓ Auto-scaling enabled")

    def design_principle_3_detect_fast(self):
        """Принцип 3: Быстрое обнаружение отказов"""
        print(f"\n[{self.name}] Design Principle 3: Detect Failures Fast")
        print(f"  ✓ Health check every 10 seconds")
        print(f"  ✓ Monitoring on all critical metrics")
        print(f"  ✓ Alerting on error rate > 5%")
        print(f"  ✓ Dashboard updated every 1 second")

    def design_principle_4_recover_automatically(self):
        """Принцип 4: Автоматическое восстановление"""
        print(f"\n[{self.name}] Design Principle 4: Automatic Recovery")
        print(f"  ✓ Automatic failover (< 30s)")
        print(f"  ✓ Service restart on crash")
        print(f"  ✓ Lease-based leader election")
        print(f"  ✓ Data consistency via quorum")

    def design_principle_5_graceful_degradation(self):
        """Принцип 5: Graceful degradation"""
        print(f"\n[{self.name}] Design Principle 5: Graceful Degradation")
        print(f"  ✓ Feature flags for non-critical features")
        print(f"  ✓ Cache fallback when service slow")
        print(f"  ✓ Read-only mode if write unavailable")
        print(f"  ✓ Partial results instead of full failure")

    def design_principle_6_test_chaos(self):
        """Принцип 6: Chaos engineering"""
        print(f"\n[{self.name}] Design Principle 6: Test with Chaos")
        print(f"  ✓ Kill random instances weekly")
        print(f"  ✓ Network partition simulation monthly")
        print(f"  ✓ Game days: full system failure scenarios")
        print(f"  ✓ Measure recovery time (RTO/RPO)")

    def health_check(self) -> bool:
        """Выполнить health check"""
        # Симуляция: 90% success rate
        is_healthy = random.random() > 0.1
        self.health_check_results.append(is_healthy)

        self.metrics.available_instances = sum(self.health_check_results)
        return is_healthy

    def handle_request(self) -> bool:
        """Обработать запрос с resilience"""
        self.metrics.request_count += 1

        # Check 1: Circuit breaker
        if self.circuit_breaker_open:
            print(f"    ✗ Circuit breaker open, rejecting request")
            self.metrics.error_count += 1
            return False

        # Check 2: Available capacity
        if self.metrics.available_instances < 1:
            print(f"    ✗ No available instances")
            self.metrics.degraded = True
            self.metrics.error_count += 1
            return False

        # Check 3: Try with timeout (simulated)
        try:
            # 8% failure rate
            if random.random() < 0.08:
                raise Exception("Temporary failure")

            print(f"    ✓ Request succeeded")
            return True
        except Exception as e:
            self.failures += 1
            self.metrics.error_count += 1

            # Открыть circuit breaker если много ошибок
            error_rate = self.metrics.error_rate()
            if error_rate > 0.05:
                self.circuit_breaker_open = True
                print(f"    ✗ Error rate {error_rate:.1%} > 5%, opening circuit breaker")

            return False

    def run_checks(self):
        """Запустить все проверки"""
        print(f"\n{'='*70}")
        print(f"SERVICE: {self.name}")
        print(f"{'='*70}\n")

        # Print all design principles
        self.design_principle_1_assume_failure()
        self.design_principle_2_redundancy()
        self.design_principle_3_detect_fast()
        self.design_principle_4_recover_automatically()
        self.design_principle_5_graceful_degradation()
        self.design_principle_6_test_chaos()

        # Health check
        print(f"\n[Health Check]")
        is_healthy = self.health_check()
        print(f"  {'✓' if is_healthy else '✗'} Overall health: {self.metrics.available_instances}/10")

        # Handle some requests
        print(f"\n[Requests Handling]")
        for i in range(5):
            self.handle_request()

        # Show metrics
        print(f"\n[Metrics]")
        print(f"  Requests: {self.metrics.request_count}")
        print(f"  Errors: {self.metrics.error_count}")
        print(f"  Error rate: {self.metrics.error_rate():.1%}")
        print(f"  Available instances: {self.metrics.available_instances}/10")
        print(f"  Degraded: {self.metrics.degraded}")


class FaultTolerantSystemChecklist:
    """Полный checklist для проектирования fault-tolerant систем"""

    @staticmethod
    def print_checklist():
        print(f"\n{'='*70}")
        print(f"FAULT-TOLERANT SYSTEM DESIGN CHECKLIST")
        print(f"{'='*70}\n")

        checklist = {
            "ARCHITECTURE": [
                "No single point of failure",
                "Components are loosely coupled",
                "Service discovery mechanism",
                "Multiple zones/regions",
                "Load balancing with health checks",
                "API versioning for compatibility"
            ],
            "DATA MANAGEMENT": [
                "Replication (3+ copies)",
                "Offsite backups",
                "Data consistency model defined",
                "Atomic operations where needed",
                "Idempotent requests",
                "ACID or BASE guarantees specified"
            ],
            "NETWORK": [
                "Timeout on all calls",
                "Retry with exponential backoff",
                "Circuit breaker pattern",
                "Rate limiting",
                "Bulkhead isolation",
                "Connection pooling"
            ],
            "FAILURE DETECTION": [
                "Health check endpoints",
                "Metrics collection",
                "Alerting rules",
                "Logging and tracing",
                "Dashboards",
                "Post-mortem process"
            ],
            "RECOVERY": [
                "Automatic failover",
                "Service restart capability",
                "Graceful degradation",
                "Rollback procedures",
                "Data synchronization",
                "Recovery time targets (RTO)"
            ],
            "TESTING": [
                "Unit tests",
                "Integration tests",
                "Chaos engineering",
                "Failure injection",
                "Load testing",
                "Game days (exercises)"
            ],
            "OPERATIONS": [
                "Runbooks for common failures",
                "Escalation procedures",
                "On-call rotation",
                "Post-mortem template",
                "SLA/SLO defined",
                "Change management process"
            ]
        }

        for section, items in checklist.items():
            print(f"[{section}]")
            for item in items:
                print(f"  [ ] {item}")
            print()


# Демонстрация
service = ResilientService("PaymentService")
service.run_checks()

FaultTolerantSystemChecklist.print_checklist()
```

### Типичные ошибки
1. **Design by committee** — too many layers, complexity
2. **Over-engineering** — дорогое усложнение для rare scenarios
3. **Lack of testing** — не знаем, работает ли recovery
4. **Ignored failure modes** — есть неизвестные unknowns
5. **No monitoring** — слепые полёты

### На интервью
**Как отвечать:**
- Начните с принципов Design for Failure
- Перейдите к чеклисту (архитектура, данные, сеть и т.д.)
- Дайте реальный пример системы
- Упомяните как вы тестируете (chaos, game days)

**Follow-ups:**
- "Как приоритизировать что фиксировать?" → risk = impact × probability
- "Какой SLA реалистичен?" → 99.9% нужна 43.2 мин downtime/year
- "Как организовать опс?" → on-call, runbooks, automation
- "Что о cost?" → redundancy дорого, нужен balance с требованиями

---

## Реальные Инциденты и Lessons Learned

### AWS US-EAST-1 Outage (2011)

**Что произошло:**
- Lightning strike попала в cooling system в data center
- Temperature в одной rack группе выросла
- Несколько узлов Network topology контроллера перегрелись и упали
- Вся система поиска IP addresses в том регионе collapsed

**Lessons:**
- Даже cloud провайдеры не immune к physical failures
- Необходима geographical redundancy
- Cascading failures были недостаточно контролируемы

**Как это предотвратить:**
- Multi-region deployment
- Graceful degradation (continue with reduced capacity)
- Better circuit breaker patterns

### GitHub Outage (2012) — Split Brain

**Что произошло:**
- MySQL master-slave реplication отстала
- Network partition между master и slave
- Оба начали принимать writes
- Data inconsistency и потеря committed transactions

**Lessons:**
- Split-brain в databases очень опасен
- Нужен quorum для выборов лидера
- Нельзя полагаться только на replication lag detection

**Как это предотвратить:**
- Использовать consensus алгоритмы (Paxos, Raft)
- Quorum-based leader election
- Synchronous replication для critical data

### Facebook Data Center Failure (2013)

**Что произошло:**
- Cooling system failure в одном data center
- Cascading failure: когда один DC упал, traffic перенаправился на другие
- Те другие тоже overloaded и упали

**Lessons:**
- Load balancing must account for capacity loss
- Graceful degradation важна на каждом уровне
- Circuit breakers нужны чтобы reject excess load

**Как это предотвратить:**
- Health-aware load balancing
- Overload protection (load shedding)
- Graceful shutdown procedures

---

## Краткий Summary — Таблица Всех Техник

```
┌─────────────────────┬──────────────────┬─────────────────┬──────────────┐
│ Техника             │ Когда применять  │ Overhead        │ Как работает │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Timeout             │ Все network      │ Нет             │ Отклонить    │
│                     │ calls            │                 │ медленные     │
│                     │                  │                 │ requests     │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Retry               │ Transient        │ Extra latency   │ Try again    │
│ (exponential        │ failures         │                 │ с backoff    │
│ backoff)            │                  │                 │              │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Circuit Breaker     │ External svc     │ Memory          │ Fail-fast    │
│                     │ failures         │ (counters)      │ to avoid     │
│                     │                  │                 │ cascading    │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Bulkhead            │ Resource         │ Separate pools  │ Isolate      │
│                     │ contention       │ (threads,       │ resources    │
│                     │                  │ connections)    │              │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Rate Limit          │ Overload         │ Queue/reject    │ Shed excess  │
│                     │ protection       │ tracking        │ load         │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Caching             │ Slow/failing     │ Memory usage    │ Serve stale  │
│                     │ backends         │                 │ data         │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Replication         │ Data loss        │ Network, disk   │ Redundant    │
│                     │ prevention       │ (2-3x)          │ copies       │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Failover            │ Component death  │ Detection lag   │ Switch to    │
│                     │                  │ (seconds)       │ backup       │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Feature Flag        │ Graceful         │ Code complexity │ Toggle non-  │
│                     │ degradation      │                 │ critical fn  │
├─────────────────────┼──────────────────┼─────────────────┼──────────────┤
│ Heartbeat/          │ Failure          │ Network traffic │ Detect node  │
│ Lease               │ detection        │ (low)           │ health       │
└─────────────────────┴──────────────────┴─────────────────┴──────────────┘
```

---

## Итоговые Советы для Интервью

1. **Всегда спрашивайте про failures** — "Как система ведёт себя если X падёт?"
2. **Предложите redundancy** — "Давайте добавим replicas и failover"
3. **Обсудите trade-offs** — "Availability vs Consistency vs Latency"
4. **Приведите примеры** — Real incidents (AWS, GitHub, Facebook)
5. **Нарисуйте диаграммы** — ASCII diagrams очень помогают
6. **Mention monitoring** — "Нам нужны metrics и alerting"
7. **Тестирование** — "Давайте используем chaos engineering"
8. **Graceful degradation** — "Система не падает полностью, работает с ограничениями"

---

## Ресурсы для Углубления

**Книги:**
- "Designing Data-Intensive Applications" — Martin Kleppmann
- "Release It!" — Michael Nygard (про production-ready systems)
- "Site Reliability Engineering" — Google (free онлайн)

**Статьи:**
- Netflix Chaos Engineering
- Amazon "Failure Modes" article
- Google "Dremel" (distributed query system with resilience)

**Инструменты:**
- Chaos Monkey (Netflix) — kill random production instances
- Gremlin — commercial chaos engineering platform
- Prometheus — monitoring and alerting


```

### Типичные ошибки
1. **Слишком низкий failure_threshold** — вызывает преждевременное открытие при нормальных сетевых задержках
2. **Слишком длинный recovery_timeout** — сервис может восстановиться, а CB ещё закрыт
3. **Отсутствие time window** — старые ошибки влияют на новые решения
4. **Не логировать переходы состояния** — сложно диагностировать проблемы

### На интервью
**Как отвечать:**
- Нарисуйте диаграмму трёх состояний
- Объясните, как это предотвращает каскадные отказы
- Упомяните параметры: thresholds, timeout, half-open
- Покажите код с основной логикой

**Follow-ups:**
- "Как комбинировать CB с retry?" → exponential backoff inside CB
- "Что такое bulkhead pattern?" → изоляция ресурсов для разных сервисов
- "Как мониторить CB?" → метрики: open time, failure rate, state changes

