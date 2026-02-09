# Консенсус: Алгоритм Raft

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [15-ai-systems](./15-ai-systems.md) · Следующая тема: [17-distributed-transactions](./17-distributed-transactions.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Consensus** — алгоритм, при котором несколько узлов договариваются об одном значении несмотря на отказы и сетевые задержки. Это фундамент надёжных распределённых систем, поскольку все узлы должны иметь одинаковое состояние, чтобы система работала корректно. Консенсус критичен для выборов лидера, реплицирования данных и других операций в системе.

**Raft** — алгоритм консенсуса, который делит время на временные периоды (term'ы), выбирает одного лидера и реплицирует логи операций. Raft был создан как более понятная альтернатива Paxos, он проще реализовать и отладить. Широко используется в системах вроде etcd, Consul и других распределённых решений.

**Leader Election** — процесс выбора одного узла, который станет координатором и будет руководить системой. Без лидера система не может принимать решения, консенсус не достижим и система теряет способность прогрессировать. Это критичный механизм в любом консенсус-алгоритме, основанном на лидере.

**Log Replication** — процесс копирования операций со временем от лидера на все узлы-последователи (фоллоуеры). Гарантирует, что все узлы выполнят операции в одинаковом порядке и придут к одному состоянию. Это обеспечивает согласованность данных во всей системе.

**Term** — логический период времени в алгоритме Raft, в котором может быть избран только один лидер. Терм помогает узлам обнаружить устаревшую информацию и синхронизировать состояние, предотвращая одновременное существование нескольких лидеров.

**Quorum** — большинство узлов в системе (больше половины от общего количества). Использование кворума гарантирует, что даже при разделении сети на две части, только одна партиция (с большинством узлов) может принимать решения и считаться живой.

**Split-brain** — ситуация, когда разделение сети создаёт две независимые партиции, и каждая из них считает себя единственной живой и выбирает своего лидера. Это опасное состояние, которое ведёт к конфликтам данных, нарушению консистентности и расхождению между партициями.

**Paxos** — более сложный и ранний алгоритм консенсуса, предшественник Raft. Он теоретически обоснован и доказано корректен, но очень сложен в реализации и понимании, что делает его менее популярным в практических системах.

**Follower** — узел, который следует за лидером и не инициирует изменения. Большинство узлов в системе находятся в состоянии follower и пассивно принимают команды и логи от лидера.

**Candidate** — узел, который пытается стать лидером и активно запрашивает голоса у других узлов. Это временное состояние во время процесса выборов лидера, из которого узел либо становится лидером, либо возвращается в состояние follower.

**FLP Impossibility** — теорема Фишера-Линча-Патерсона, утверждающая, что в полностью асинхронной системе невозможно построить детерминированный алгоритм консенсуса, который одновременно гарантирует safety и liveness. Это объясняет, почему все реальные системы используют randomization или предполагают частичную синхронизацию.

**Safety & Liveness** — два противоположных требования консенсуса. Safety означает, что все узлы согласны на одно и то же значение и будут согласны на него навечно. Liveness означает, что система в итоге примет и вернёт некоторое значение за конечное время. Балансирование между этими требованиями лежит в основе любого практического алгоритма консенсуса.

---

## 1. Что такое консенсус в распределённых системах и зачем он нужен?

### Зачем спрашивают
Консенсус — фундаментальная проблема распределённых систем. Интервьюеры проверяют, понимаете ли вы, почему согласованность состояния критична и какие вызовы её обеспечивают.

### Короткий ответ
Консенсус — это алгоритм, позволяющий группе узлов достичь соглашения о единственном значении несмотря на отказы и сетевые задержки. Он необходим для построения надёжных распределённых систем, где все узлы должны иметь одинаковое состояние.

### Детальный разбор

**Проблема консенсуса:**
- В распределённой системе нужно, чтобы несколько узлов договорились о значении (например, кто лидер, какой следующий элемент очереди)
- Узлы могут отказать, сеть может задерживать/терять сообщения
- Нельзя полагаться на синхронизацию часов или точные таймауты

**Требования к консенсусу:**
1. **Safety** (безопасность): Все узлы, которые приняли значение, приняли одно и то же значение
2. **Liveness** (живость): Система в итоге примет некоторое значение (не зависает)
3. **Fault-tolerance** (устойчивость к отказам): Работает даже если отказали f узлов из N (обычно f < N/2)

**Классические решения:**

```
┌─────────────────────────────────────────┐
│     Алгоритмы консенсуса               │
├─────────────────────────────────────────┤
│ Paxos (сложный, асимметричный)        │
│ Raft (понятный, лидер-следователь)    │
│ PBFT (Byzantine, для враждебной среды) │
│ Viewstamped Replication (ранний)      │
└─────────────────────────────────────────┘
```

**FLP impossibility** (Fischer-Lynch-Paterson):
- В асинхронной системе нельзя построить детерминированный консенсус, гарантирующий одновременно safety и liveness
- На практике: используют randomization или частичную синхронизацию

**Применение консенсуса:**
- **Consensus log**: Все узлы поддерживают одинаковый лог операций
- **Elected leader**: Выбор единого координатора
- **Distributed transactions**: Двухфазный коммит использует консенсус
- **Distributed locks**: Захват ресурса в системе

### Пример

```python
# Абстрактный консенсус: все узлы должны договориться о значении
class ConsensusNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.committed_value = None
        self.state = "follower"  # follower, candidate, leader

    def propose(self, value):
        """Предложить значение для консенсуса"""
        # Нужно согласовать это значение с большинством
        votes = 0
        for peer in self.peers:
            if peer.vote_for(value):
                votes += 1

        if votes > len(self.peers) // 2:
            # Большинство согласилось
            self.committed_value = value
            return True
        return False

    def get_committed_value(self):
        return self.committed_value
```

### Типичные ошибки
1. **Путаница между consensus и replication**: Консенсус — это согласованность, репликация — это копирование
2. **Забывание о split-brain**: Если разделена сеть, консенсус должен работать только у большинства
3. **Игнорирование сетевых задержек**: Консенсус работает в асинхронной среде, не на локальных часах

### На интервью
- Объясните, почему консенсус сложен (асинхронность, отказы)
- Назовите примеры систем: etcd, Consul, Zookeeper (ZAB), CockroachDB
- Упомяните trade-offs: безопасность vs живость, сложность vs надёжность
- Будьте готовы к follow-up: "Почему нельзя просто скопировать данные на все узлы?" (Ответ: нужно согласованное время применения изменений)

---

## 2. Как работает алгоритм Raft? (Leader election, log replication, safety)

### Зачем спрашивают
Это core вопрос. Интервьюеры хотят знать, понимаете ли вы основные компоненты Raft: выбор лидера, репликацию логов и гарантии безопасности.

### Короткий ответ
Raft делит время на term'ы. В каждом term'е сначала выбирается лидер через quorum голосов, затем лидер реплицирует лог операций на followers. Safety обеспечивается через ограничения на состояние, которое может быть выбрано лидером.

### Детальный разбор

**Три основных состояния узла:**

```
    ┌──────────────────────────────┐
    │                              │
    v                              │
┌────────┐   timeout       ┌────────────┐
│Follower├──────────────-> │ Candidate  │
└────────┘                 └──────┬─────┘
    ^                             │
    │                        wins election
    │                             │
    │                             v
    │                      ┌─────────────┐
    └──────────────────────┤   Leader    │
                           └─────────────┘
```

**Структура лога:**

```
Узел A (Leader):
┌─────┬─────┬─────┬─────┬─────┐
│ #1  │ #2  │ #3  │ #4  │ #5  │ commitIndex=3
│"SET"│"SET"│"DEL"│"ADD"│"SET"│ lastApplied=2
└─────┴─────┴─────┴─────┴─────┘
 term  term  term  term  term
  1     2     3     3     3

Узел B (Follower):
┌─────┬─────┬─────┐
│ #1  │ #2  │ #3  │ commitIndex=2
│"SET"│"SET"│"DEL"│ lastApplied=1
└─────┴─────┴─────┘
 term  term  term
  1     2     3
```

**Ключевые параметры каждого узла:**

```python
class RaftState:
    # Persistent (на диске)
    current_term = 0              # Последний известный term
    voted_for = None              # За кого голосовали в current_term
    log = []                       # Лог записей [term, value]

    # Volatile (в памяти)
    commit_index = 0              # Индекс, до которого безопасно применить
    last_applied = 0              # Индекс последней применённой записи

    # На leader'е
    next_index = {}               # Для каждого follower: индекс следующей записи
    match_index = {}              # Для каждого follower: индекс последней совпадающей записи
```

**Жизненный цикл записи:**

```
1. Client отправляет команду на Leader
2. Leader добавляет в свой лог (NOT committed)
3. Leader отправляет AppendEntries на Followers
4. Followers добавляют в свои логи, отвечают ОК
5. Leader когда большинство ответило: commitIndex += 1
6. Leader применяет команду в state machine
7. Leader отправляет результат Client'у
8. Followers применяют команду когда узнают о новом commitIndex
```

### Пример

```python
from enum import Enum
from typing import List, Dict, Tuple
import time
import threading
from collections import defaultdict

class NodeState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

class RaftNode:
    def __init__(self, node_id: str, peers: List[str]):
        self.node_id = node_id
        self.peers = peers

        # Persistent state
        self.current_term = 0
        self.voted_for = None
        self.log = []  # List[Tuple[term, command]]

        # Volatile state
        self.state = NodeState.FOLLOWER
        self.commit_index = 0
        self.last_applied = 0

        # Leader state
        self.next_index = defaultdict(lambda: len(self.log))
        self.match_index = defaultdict(int)

        # Timers
        self.election_timeout = time.time() + self._random_timeout()
        self.heartbeat_interval = 0.15
        self.last_heartbeat = time.time()

        self.state_machine = {}  # KV store
        self.lock = threading.Lock()

    def _random_timeout(self):
        """Random timeout между 1.5 и 3 секундами"""
        import random
        return random.uniform(1.5, 3.0)

    def append_entry(self, term: int, leader_id: str, prev_log_index: int,
                     prev_log_term: int, entries: List, leader_commit: int) -> bool:
        """
        RPC: Append Entries (от Leader)
        Используется для репликации логов И как heartbeat
        """
        with self.lock:
            # Отклонить если term старый
            if term < self.current_term:
                return False

            # Обновить term если новый
            if term > self.current_term:
                self.current_term = term
                self.state = NodeState.FOLLOWER
                self.voted_for = None

            self.election_timeout = time.time() + self._random_timeout()

            # Проверить log matching property
            if prev_log_index > 0:
                if prev_log_index > len(self.log) - 1:
                    return False  # Нет записи на prev_log_index
                if self.log[prev_log_index][0] != prev_log_term:
                    return False  # Term не совпадает

            # Добавить новые записи
            if entries:
                # Удалить конфликтующие записи
                while len(self.log) > prev_log_index + 1:
                    self.log.pop()

                # Добавить новые
                for entry in entries:
                    self.log.append(entry)

            # Обновить commit_index
            if leader_commit > self.commit_index:
                self.commit_index = min(leader_commit, len(self.log) - 1)

            return True

    def request_vote(self, term: int, candidate_id: str,
                     last_log_index: int, last_log_term: int) -> bool:
        """
        RPC: Request Vote (от Candidate)
        """
        with self.lock:
            # Отклонить если term старый
            if term < self.current_term:
                return False

            # Обновить term
            if term > self.current_term:
                self.current_term = term
                self.voted_for = None
                self.state = NodeState.FOLLOWER

            # Проверить условия голоса
            if self.voted_for is not None and self.voted_for != candidate_id:
                return False  # Уже голосовали в этом term'е

            # Проверить log "completeness"
            my_last_log_index = len(self.log) - 1
            my_last_log_term = self.log[my_last_log_index][0] if self.log else 0

            if last_log_term < my_last_log_term:
                return False  # Кандидат отстаёт по term'ам

            if last_log_term == my_last_log_term and last_log_index < my_last_log_index:
                return False  # Кандидат отстаёт по длине логов

            # Голосуем за кандидата
            self.voted_for = candidate_id
            self.election_timeout = time.time() + self._random_timeout()
            return True

    def apply_committed_entries(self):
        """Применить доставленные записи в state machine"""
        with self.lock:
            while self.last_applied < self.commit_index:
                self.last_applied += 1
                term, command = self.log[self.last_applied]
                # Применить команду в state machine
                # Например: {"op": "SET", "key": "x", "val": 5}
                self._apply_command(command)

    def _apply_command(self, command):
        """Применить команду в состояние"""
        if command.get("op") == "SET":
            self.state_machine[command["key"]] = command["val"]
        elif command.get("op") == "GET":
            return self.state_machine.get(command["key"])

    def become_leader(self):
        """Инициализировать leader'а после победы в выборах"""
        with self.lock:
            self.state = NodeState.LEADER
            # Инициализировать next_index и match_index
            for peer in self.peers:
                self.next_index[peer] = len(self.log)
                self.match_index[peer] = 0
            # Отправить начальный heartbeat
            self._send_heartbeats()

    def _send_heartbeats(self):
        """Отправить AppendEntries на все followers (как heartbeat)"""
        for peer in self.peers:
            prev_log_index = self.next_index[peer] - 1
            prev_log_term = self.log[prev_log_index][0] if prev_log_index >= 0 else 0

            entries = self.log[self.next_index[peer]:]

            # Отправить RPC (в реальности: асинхронно)
            # peer.append_entry(self.current_term, self.node_id,
            #                   prev_log_index, prev_log_term,
            #                   entries, self.commit_index)
```

### Типичные ошибки
1. **Забыть про term'ы**: Term — это логические часы, гарантирующие, что старые leader'ы не конфликтуют с новыми
2. **Неправильное обновление commit_index**: Leader может commit'ить только записи из ТЕКУЩЕГО term'а
3. **Игнорирование log matching**: Followers должны проверить, что новые записи совпадают с существующим логом

### На интервью
- Нарисуйте диаграмму state transitions между Follower/Candidate/Leader
- Объясните, почему term увеличивается монотонно
- Дайте пример: как обновляется лог, когда leader отказывает
- Уточняющий вопрос: "Что если leader падает после commit'а но до отправки ответа client'у?"

---

## 3. Как происходит выбор лидера в Raft?

### Зачем спрашивают
Выбор лидера — критический компонент. Интервьюеры проверяют, понимаете ли вы quorum voting и как избежать split-brain.

### Короткий ответ
Когда follower не получает heartbeat от leader'а, он становится candidate и запускает выборы. Candidate увеличивает term, голосует за себя и просит других узлов голосовать. Первый кандидат, набравший большинство голосов (quorum), становится leader.

### Детальный разбор

**Процесс выборов в деталях:**

```
┌─────────────────────────────────────────────────────┐
│ Follower не получает heartbeat за election_timeout │
└────────────────┬────────────────────────────────────┘
                 │
                 v
         ┌──────────────┐
         │   Candidate  │
         │ +current_term│
         └──────┬───────┘
                │
                v
    ┌───────────────────────────┐
    │ Отправить RequestVote RPC │
    │ со своим term и лог info  │
    └────────────┬──────────────┘
                 │
        ┌────────┴────────┐
        │                 │
        v                 v
   ┌────────┐        ┌─────────┐
   │ Выиграл│        │ Проиграл│
   │большин │        │(отсутств│
   │ство   │        │ий term) │
   │голосов │        └────┬────┘
   └───┬────┘             │
       │             ┌────v─────┐
       │             │ Fallback │
       │             │ к Follower
       v             └──────────┘
   ┌──────┐
   │Leader│ <- может получить heartbeat от другого leader'а
   └──────┘    и вернуться в follower (если его term старше)
```

**Правила голосования:**

```python
def can_vote_for(candidate_id, candidate_term, candidate_last_log_index,
                 candidate_last_log_term, my_term, my_voted_for, my_log):
    """Правила, по которым follower голосует за candidate'а"""

    # Правило 1: Только голосуем в one vote per term
    if my_voted_for is not None and my_voted_for != candidate_id:
        return False

    # Правило 2: Candidate должен быть "up-to-date" (не отстава от нас по логу)
    my_last_log_term = my_log[-1][0] if my_log else 0
    my_last_log_index = len(my_log) - 1

    # Сравнение: сначала term, потом index (lexicographic order)
    if candidate_last_log_term != my_last_log_term:
        return candidate_last_log_term > my_last_log_term
    else:
        return candidate_last_log_index >= my_last_log_index
```

**Пример выборов на 5 узлах:**

```
Ситуация: Leader (A) упал, timeout истекает в B

Шаг 1: B становится candidate, увеличивает term на 3
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│ A   │  │ B*  │  │ C   │  │ D   │  │ E   │
│ T=2 │  │ T=3 │  │ T=2 │  │ T=2 │  │ T=2 │
└─────┘  └─────┘  └─────┘  └─────┘  └─────┘
            |
            ├─ RequestVote T=3 -> C, D, E
            │

Шаг 2: C, D, E получают RequestVote
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│ A   │  │ B*  │  │ C   │  │ D   │  │ E   │
│ T=2 │  │ T=3 │  │ T=3 │  │ T=3 │  │ T=3 │
└─────┘  └─────┘  │voted│  │voted│  │voted│
                  │ for │  │ for │  │ for │
                  │  B  │  │  B  │  │  B  │
                  └─────┘  └─────┘  └─────┘

Шаг 3: B получает 3 голоса (большинство из 5)
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│ A   │  │ B L │  │ C   │  │ D   │  │ E   │
│ T=2 │  │ T=3 │  │ T=3 │  │ T=3 │  │ T=3 │
└─────┘  │LEAD │  └─────┘  └─────┘  └─────┘
         │ER  │
         └─────┘
            │
            └─> AppendEntries heartbeat на C, D, E
```

**Почему нужно большинство (quorum)?**

```
Пример 5 узлов (нужно 3 голоса для большинства):
- Один лидер может быть выбран за раз
- Если разделится сеть: 3-2 split
  - Группа из 3 продолжает работать (есть большинство)
  - Группа из 2 не может выбрать лидера (нет большинства)
  - Избегаем split-brain (несколько лидеров)

Пример 4 узлов (нужно 3 голоса):
- Максимум один отказ: 3 узла работают, 1 отказал
- Два отказа: не могут выбрать лидера (нет большинства)
```

**Timeout и backoff:**

```python
class ElectionTimer:
    def __init__(self):
        self.base_timeout = 1.5  # секунд
        self.max_timeout = 3.0   # секунд
        self.last_reset = time.time()

    def should_start_election(self):
        elapsed = time.time() - self.last_reset
        timeout = random.uniform(self.base_timeout, self.max_timeout)
        return elapsed > timeout

    def reset(self):
        self.last_reset = time.time()

# Randomization предотвращает election storms
# Если все узлы используют одинаковый timeout,
# может быть много simultaneous candidates
```

### Пример

```python
class RaftElection:
    def __init__(self, node_id: str, peers: List[str], current_term: int):
        self.node_id = node_id
        self.peers = peers
        self.current_term = current_term
        self.votes_received = set()
        self.election_timeout = None

    def start_election(self):
        """Инициировать выборы"""
        self.current_term += 1
        self.votes_received = {self.node_id}  # Голосуем за себя
        self.election_timeout = time.time() + random.uniform(1.5, 3.0)

        print(f"Node {self.node_id}: Starting election for term {self.current_term}")

        # Отправить RequestVote на все peers
        for peer in self.peers:
            self._request_vote_from(peer)

    def _request_vote_from(self, peer):
        """Отправить RequestVote RPC"""
        # В реальности: async RPC
        # vote = peer.request_vote(self.current_term, self.node_id, ...)
        pass

    def receive_vote(self, voter_id: str):
        """Получить голос от узла"""
        self.votes_received.add(voter_id)

        quorum = len(self.peers) // 2 + 1
        if len(self.votes_received) >= quorum:
            print(f"Node {self.node_id}: Won election with {len(self.votes_received)} votes")
            return "LEADER"

        return "CANDIDATE"

    def receive_heartbeat_from_leader(self, leader_term: int):
        """Получить heartbeat (AppendEntries) от leader'а"""
        if leader_term >= self.current_term:
            print(f"Node {self.node_id}: Accepting leader in term {leader_term}")
            return "FOLLOWER"
        return "CANDIDATE"
```

### Типичные ошибки
1. **Не увеличивать term при старте выборов**: Это критично для различения старых и новых candidate'ов
2. **Забывать про voted_for**: Persistence этого поля предотвращает vote splitting
3. **Неправильный quorum расчёт**: Для N узлов нужно (N/2 + 1) голосов, не N

### На интервью
- Объясните, почему randomization election timeout важна
- Нарисуйте диаграмму: что происходит когда есть multiple candidates?
- Дайте пример: как избежать split-brain
- Уточняющий вопрос: "Сколько времени может занять выбор лидера?" (Ответ: несколько election timeout'ов в худшем случае)

---

## 4. Как работает репликация лога в Raft?

### Зачем спрашивают
Репликация — это как данные распространяются. Интервьюеры хотят знать механику AppendEntries и как обрабатываются конфликты.

### Короткий ответ
Leader хранит next_index и match_index для каждого follower. Периодически leader отправляет AppendEntries RPC с новыми логами. Followers проверяют log matching (совпадение с предыдущей записью) и добавляют новые записи. Leader обновляет commit_index когда большинство followers приняло запись.

### Детальный разбор

**Механика AppendEntries:**

```
Leader отправляет:
┌────────────────────────────────────┐
│ term: 3                            │
│ leaderID: "A"                      │
│ prevLogIndex: 5                    │ <- Последняя репликированная запись
│ prevLogTerm: 2                     │ <- Её term
│ entries: [                         │ <- Новые записи
│   {term: 3, cmd: "SET x=10"},      │
│   {term: 3, cmd: "SET y=20"}       │
│ ]                                  │
│ leaderCommit: 4                    │ <- Commit index на leader'е
└────────────────────────────────────┘

Follower проверяет:
1. prevLogIndex существует в логе?
2. log[prevLogIndex].term == prevLogTerm?
3. Если нет: REJECT (log mismatch)
4. Если да: добавить entries, обновить commitIndex
```

**Процесс обновления next_index:**

```
Leader: А я думаю, у Follower C есть записи до индекса 7

Попытка 1:
  next_index[C] = 7
  leader_log: [1,2,3,4,5,6,7]
  follower_log: [1,2,3]
  leader отправляет: prevLogIndex=7 -> REJECT (log too short)

Попытка 2 (backoff):
  next_index[C] = 6
  leader отправляет: prevLogIndex=6 -> REJECT

Попытка 3:
  next_index[C] = 3
  leader отправляет: prevLogIndex=3
  Follower проверяет: есть ли log[3]? Есть! Term совпадает? Да!
  Follower добавляет: log[4], log[5], log[6], log[7]
  -> SUCCESS
```

**Диаграмма: Replication process**

```
Момент времени t=0 (Leader падает):
Leader L:  ┌─────┬─────┬─────┬─────┬─────┐
           │  1  │  2  │  3  │  4  │  5  │  (uncommitted)
           └─────┴─────┴─────┴─────┴─────┘
           commit_index = 3

Follower A: ┌─────┬─────┬─────┐
            │  1  │  2  │  3  │
            └─────┴─────┴─────┘

Follower B: ┌─────┬─────┐
            │  1  │  2  │
            └─────┴─────┘

Follower C: ┌─────┬─────┬─────┬─────┐
            │  1  │  2  │  3  │  4  │
            └─────┴─────┴─────┴─────┘
            (diverged log!)

Момент времени t=1 (Leader восстанавливается и посылает heartbeat):
AppendEntries на B:
  prevLogIndex=3, prevLogTerm=3 <- Leader хочет проверить совпадение
  Но у B только 2 логи!
  B ответит: false (log mismatch)

Момент времени t=2 (Leader уменьшает next_index[B]):
next_index[B] = 2
AppendEntries на B:
  prevLogIndex=2, prevLogTerm=2
  B находит log[2], term совпадает
  B отвечает: true
  B получает новые entries [3, 4, 5]

Момент времени t=3 (Leader обновил большинство):
commitIndex на leader = 4 (большинство принимает ≥ 4)
Leader отправляет commitIndex=4
Followers применяют log[4]
```

**Гарантии репликации:**

```python
# Leader может commit запись ТОЛЬКО если:
# 1. Запись уже в логе лидера
# 2. Большинство followers ответили OK на AppendEntries с этой записью
# 3. Запись из ТЕКУЩЕГО term'а лидера (важно!)

def update_commit_index(self):
    """Найти максимальный N такой что:
    - log[N].term == currentTerm
    - большинство servers имеют log[N]
    """
    for n in range(len(self.log) - 1, self.commit_index, -1):
        if self.log[n].term == self.current_term:
            # Проверить, есть ли у большинства эта запись
            count = 1  # Leader сам имеет
            for peer in self.peers:
                if self.match_index[peer] >= n:
                    count += 1

            if count > len(self.peers) // 2:
                self.commit_index = n
                break
```

### Пример

```python
class RaftLogReplication:
    def __init__(self, node_id: str, peers: List[str]):
        self.node_id = node_id
        self.peers = peers
        self.log = []
        self.current_term = 0

        # Leader state
        self.next_index = {}      # Для каждого peer: next индекс для отправки
        self.match_index = {}     # Для каждого peer: последний совпадающий индекс

        for peer in peers:
            self.next_index[peer] = len(self.log)
            self.match_index[peer] = 0

    def replicate_to_followers(self):
        """Отправить AppendEntries на все followers"""
        for peer in self.peers:
            self._send_append_entries(peer)

    def _send_append_entries(self, peer: str):
        """Отправить AppendEntries RPC на один peer"""
        prev_log_index = self.next_index[peer] - 1
        prev_log_term = 0

        if prev_log_index >= 0 and prev_log_index < len(self.log):
            prev_log_term = self.log[prev_log_index][0]

        entries = self.log[self.next_index[peer]:]

        rpc_args = {
            "term": self.current_term,
            "leader_id": self.node_id,
            "prev_log_index": prev_log_index,
            "prev_log_term": prev_log_term,
            "entries": entries,
            "leader_commit": 0,  # Normally: self.commit_index
        }

        # В реальности: async RPC с timeout
        # success = peer.append_entries(rpc_args)
        # if success:
        #     self.match_index[peer] = len(self.log) - 1
        #     self.next_index[peer] = len(self.log)
        # else:
        #     self.next_index[peer] -= 1

    def handle_append_entries_success(self, peer: str, new_match_index: int):
        """Обработать успешный ответ на AppendEntries"""
        self.match_index[peer] = new_match_index
        self.next_index[peer] = new_match_index + 1

    def handle_append_entries_failure(self, peer: str):
        """Обработать неудачный ответ на AppendEntries"""
        # Уменьшить next_index и повторить попытку
        if self.next_index[peer] > 0:
            self.next_index[peer] -= 1

    def update_commit_index(self) -> int:
        """Обновить commit_index когда большинство согласилось"""
        # Найти максимальный индекс N такой что большинство имеют log[N]
        for n in range(len(self.log) - 1, -1, -1):
            if n == 0:
                continue

            # Пропустить, если это не из текущего term'а
            if self.log[n][0] != self.current_term:
                continue

            # Сосчитать, сколько имеют эту запись
            count = 1  # Leader сам
            for peer in self.peers:
                if self.match_index.get(peer, 0) >= n:
                    count += 1

            # Если большинство согласилось
            if count > (len(self.peers) + 1) // 2:
                return n

        return 0
```

### Типичные ошибки
1. **Забыть про log matching property**: Нельзя просто добавить записи в конец логов
2. **Неправильно обработать конфликты**: Нужно удалить diverged записи перед добавлением новых
3. **Commit записи из старых term'ов**: Может привести к нарушению safety

### На интервью
- Нарисуйте процесс: как leader разруливает diverged logs?
- Объясните, почему нужно проверять prevLogIndex и prevLogTerm
- Дайте пример: что если leader отправляет 1000 записей на slow follower? (Ответ: может использовать snapshot'ы)
- Уточняющий вопрос: "Как долго может занять репликация?" (Зависит от сети и размера лога)

---

## 5. Что такое term и как он используется?

### Зачем спрашивают
Term — это логические часы в Raft. Интервьюеры хотят знать, почему это необходимо и как это гарантирует безопасность.

### Короткий ответ
Term — это монотонно возрастающий счётчик, увеличивающийся при каждых выборах. Каждая запись в логе помечена term'ом того лидера, который её добавил. Term'ы обеспечивают полный порядок на событиях и предотвращают старых leader'ов от нарушения безопасности.

### Детальный разбор

**Зачем нужны term'ы:**

```
Проблема без term'ов:
┌──────────────┐         ┌──────────────┐
│  Leader L1   │  CRASH  │  Leader L2   │
│ записал: x=5 │-------> │ записал: x=10│
└──────────────┘         └──────────────┘
           │                     │
           └─> Может быть two leaders одновременно!
                (если сеть разделена)

Решение с term'ами:
┌──────────────────────────────────────┐
│ Узел A (term=5): я лидер в term 5    │
│ Узел B (term=6): я лидер в term 6    │
└──────────────────────────────────────┘

Узел B имеет более высокий term, поэтому A признаёт B
```

**Правила работы с term'ами:**

```python
class TermHandling:
    def __init__(self):
        self.current_term = 0
        self.voted_for = None

    def on_receive_rpc(self, rpc_term: int) -> bool:
        """
        Правило RPC'44:
        Если rpc_term > currentTerm:
          - Обновить currentTerm на rpc_term
          - Сохранить на диск (persistent state!)
          - Стать FOLLOWER'ом
          - Очистить votedFor
        Если rpc_term < currentTerm:
          - Отклонить RPC
        """
        if rpc_term > self.current_term:
            print(f"Term {self.current_term} -> {rpc_term}")
            self.current_term = rpc_term
            self.voted_for = None
            # PERSIST TO DISK!
            return True
        elif rpc_term < self.current_term:
            return False
        return True

    def on_start_election(self):
        """
        Правило election'22:
        Если timeout истёк:
          - Увеличить currentTerm
          - Установить state = CANDIDATE
          - Голосовать за себя
          - Отправить RequestVote на других
        """
        self.current_term += 1
        self.voted_for = self
```

**Временная шкала term'ов:**

```
Timeline:

Election для term 1: Лидер A выбран
┌────────────────────────────────────┐
│ Term 1: A is leader, 10 сек        │
└────────────────────────────────────┘

A падает, начинаются выборы
┌────────────┐  ┌────────────┐  ┌────────────┐
│ Term 2: B  │  │ Term 3: C  │  │ Term 4: D  │
│ candidate  │->│ candidate  │->│ LEADER     │
│ 1 sec      │  │ 1 sec      │  │ (стабилен)│
└────────────┘  └────────────┘  └────────────┘
    (B потерял) (C потерял)
    выборы      выборы

Term'ы обеспечивают:
1. Порядок: term 4 > term 3 > term 2 > term 1
2. Уникальность: в каждом term'е один лидер (из-за quorum)
3. Невозвратимость: old leader в term 1 никогда не сможет перезаписать term 4
```

**Важность Persistence (долговечности состояния):**

```python
class PersistentState:
    """
    Эти значения ОБЯЗАТЕЛЬНО сохранять на диск!
    Если падение -> потеря этих данных -> нарушение safety!
    """

    def __init__(self, filename="persistent.json"):
        self.filename = filename
        self.current_term = self._load_or_default("current_term", 0)
        self.voted_for = self._load_or_default("voted_for", None)
        self.log = self._load_or_default("log", [])

    def update_term(self, new_term: int):
        """ДОЛЖНО быть синхронно на диск!"""
        if new_term > self.current_term:
            self.current_term = new_term
            self._persist_to_disk()

    def vote(self, candidate_id: str):
        """ДОЛЖНО быть синхронно на диск!"""
        self.voted_for = candidate_id
        self._persist_to_disk()

    def append_log(self, entry):
        """ДОЛЖНО быть синхронно на диск!"""
        self.log.append(entry)
        self._persist_to_disk()

    def _persist_to_disk(self):
        """Синхронная запись на диск"""
        with open(self.filename, 'w') as f:
            json.dump({
                'current_term': self.current_term,
                'voted_for': self.voted_for,
                'log': self.log
            }, f)
```

**Term в контексте выборов:**

```
Сценарий: Разделение сети

┌─────────────────────────────────────────┐
│ Кластер: [A, B, C, D, E]                │
│ Лидер: A (term=5)                       │
└─────────────────────────────────────────┘
           │
           ├─[РАЗДЕЛЕНИЕ]─┤
           │              │
      Группа 1       Группа 2
      (A, B, C)      (D, E)

Время t=0: Разделение происходит
  Группа 1: A продолжает как лидер, term=5
  Группа 2: Нет большинства, выборы начинаются

Время t=1: Выборы Группы 2
  D становится candidate, начинает выборы term 6
  Но D, E < 3 голосов нужно (D только D, E)
  Не может выиграть!

Время t=2: Группа 1 обновляется
  Лидер A в term 5 обновляет всех A, B, C

Время t=3: Разделение заживает
  D, E получают AppendEntries от A с term=5
  D, E видят term 5 > их term (всё ещё 5)
  На самом деле D может проголосовать в term 6, поэтому:
  D обновляет term=5 когда видит A, становится follower

Ключевая идея:
  - Группа 1 (большинство) продолжает нормально
  - Группа 2 (меньшинство) не может committing ничего
  - При заживлении: Группа 2 догонит Группу 1
```

### Пример

```python
class TermManagement:
    def __init__(self):
        self.current_term = 0
        self.voted_for = None
        self.log = []
        self.state = "follower"
        self.term_lock = threading.Lock()

    def request_vote_rpc(self, candidate_term: int, candidate_id: str,
                         candidate_last_log_index: int,
                         candidate_last_log_term: int) -> Tuple[int, bool]:
        """
        Process RequestVote RPC
        Returns: (current_term, vote_granted)
        """
        with self.term_lock:
            # Rule 1: Reply false if term < currentTerm
            if candidate_term < self.current_term:
                return (self.current_term, False)

            # Rule 2: If term > currentTerm, update and become follower
            if candidate_term > self.current_term:
                self.current_term = candidate_term
                self.voted_for = None
                self.state = "follower"
                self._persist_state()

            # Rule 3: If votedFor is null or candidateId, vote
            if self.voted_for is None or self.voted_for == candidate_id:
                # Check if candidate's log is up-to-date
                my_last_term = self.log[-1][0] if self.log else 0
                my_last_index = len(self.log) - 1

                # Candidate is up-to-date if term is greater,
                # or term is same and index >= mine
                if (candidate_last_log_term > my_last_term or
                    (candidate_last_log_term == my_last_term and
                     candidate_last_log_index >= my_last_index)):

                    self.voted_for = candidate_id
                    self._persist_state()
                    return (self.current_term, True)

            return (self.current_term, False)

    def append_entries_rpc(self, leader_term: int, leader_id: str,
                          prev_log_index: int, prev_log_term: int,
                          entries: List, leader_commit: int) -> Tuple[int, bool]:
        """
        Process AppendEntries RPC
        Returns: (current_term, success)
        """
        with self.term_lock:
            # Rule 1: Reply false if term < currentTerm
            if leader_term < self.current_term:
                return (self.current_term, False)

            # Rule 2: If term > currentTerm, update
            if leader_term > self.current_term:
                self.current_term = leader_term
                self.voted_for = None
                self.state = "follower"
                self._persist_state()

            # If we're in an older term and just got new term from leader,
            # reset election timeout

            return (self.current_term, True)

    def _persist_state(self):
        """Save persistent state to disk"""
        # In production: WAL (Write-Ahead Log)
        state = {
            'current_term': self.current_term,
            'voted_for': self.voted_for,
            'log': self.log
        }
        # Save to disk...
        pass
```

### Типичные ошибки
1. **Забыть про persistence**: Потеря current_term или voted_for = нарушение safety
2. **Неправильно обновить term**: Не все RPC обновляют term одинаково
3. **Забыть о term в логе**: Каждая запись должна знать, в каком term'е она была добавлена

### На интервью
- Объясните, почему term монотонно возрастает
- Дайте пример: как term предотвращает split-brain
- Объясните, почему term'ы нужно сохранять на диск
- Уточняющий вопрос: "Что если узел упал и потерял свой current_term?" (Ответ: может нарушить safety, поэтому обязана persistence)

---

## 6. Как Raft обеспечивает safety guarantees?

### Зачем спрашивают
Safety — это главное свойство консенсуса. Интервьюеры хотят знать, какие гарантии дает Raft и почему они работают.

### Короткий ответ
Raft обеспечивает three safety properties: (1) Election Safety — в каждом term'е максимум один лидер (quorum voting), (2) Leader Append-Only — лидер не перезаписывает свой лог, только добавляет, (3) Log Matching — если два узла имеют одинаковую запись на одинаковом индексе, все предыдущие записи идентичны. Это гарантирует, что committed записи никогда не потеряются или не перезапишутся.

### Детальный разбор

**Свойства безопасности в деталях:**

```
Property 1: Election Safety
┌─────────────────────────────────────────────┐
│ В каждом term максимум один лидер выбран    │
├─────────────────────────────────────────────┤
│ Механизм: Quorum voting                     │
│ - Каждый узел голосует за максимум одного   │
│   candidate в term'е (voted_for на диск)   │
│ - Leader выбирается только если получает    │
│   большинство голосов (quorum)             │
│                                             │
│ Следствие: Два узла не могут получить      │
│ большинство голосов в одном term'е        │
│ (потому что большинства всегда < half+1)  │
└─────────────────────────────────────────────┘

Property 2: Leader Append-Only
┌─────────────────────────────────────────────┐
│ Leader никогда не удаляет или не            │
│ перезаписывает записи из своего лога        │
├─────────────────────────────────────────────┤
│ Механизм: AppendEntries - append-only      │
│ - Leader может только добавлять новые      │
│ - Если есть конфликт (follower имеет      │
│   другую запись на этом индексе):        │
│   - Удалить старую запись у follower      │
│   - Добавить новую                         │
│                                             │
│ Не существует механизма удаления записей  │
│ из log'а leader'а в Raft core             │
│ (Снимаются только через Snapshotting)     │
└─────────────────────────────────────────────┘

Property 3: Log Matching
┌─────────────────────────────────────────────┐
│ Если log[a].term == log[b].term и          │
│ log[a].index == log[b].index,              │
│ то log[a] == log[b] и все записи до них   │
│ идентичны                                   │
├─────────────────────────────────────────────┤
│ Механизм: prevLogIndex и prevLogTerm       │
│ - Leader проверяет, что новые записи      │
│   идут после последней совпадающей       │
│ - Followers проверяют log matching         │
│ - Индукция обеспечивает целостность       │
└─────────────────────────────────────────────┘
```

**Свойство полноты лидера:**

```
Ключевая идея:
If a log entry is committed, it is present in
the log of every leader elected in later terms

┌──────────────────────────────────────────────┐
│ Term 1: Leader A commits entry 5              │
│ Term 2: Who can be elected?                   │
│                                               │
│ Только узлы, которые либо:                   │
│ - Имеют entry 5 (получили от A)             │
│ - Или имеют log >= length и term >= term A  │
│                                               │
│ A реплицировал entry 5 на большинство       │
│ (иначе бы не смог commit).                   │
│ Поэтому любое большинство в term 2           │
│ будет содержать минимум одного узла с entry 5│
│ Этот узел сможет быть выбран leader'ом.     │
└──────────────────────────────────────────────┘
```

**Визуализация гарантий:**

```
Сценарий: Entry 5 committed в term 2

Шаг 1: Entry 5 добавлен в большинство
┌──┐  ┌──┐  ┌──┐
│A │  │B │  │C │
│ 5│  │ 5│  │ 5│ <- Большинство (3/5) имеет
└──┘  └──┘  └──┘

Шаг 2: A падает, выборы начинаются (term 3)
Кандидаты: B, C, D, E

B попросит голоса с lastLogIndex=?, lastLogTerm=2
C попросит голоса с lastLogIndex=?, lastLogTerm=2

D попросит голоса с lastLogIndex=?, lastLogTerm=1
E попросит голоса с lastLogIndex=?, lastLogTerm=1

D, E имеют более старые логи (term 1)
B, C имеют entry 5 (term 2)

Voters (старые followers):
- Если их последний entry из term 2: голосуют за B или C
- Если их последний entry из term 1: голосуют только за B или C
  (потому что B/C имеют term 2, D/E имеют term 1)

Следствие: B или C будет выбран leader (у кого будет большинство)
Entry 5 будет сохранён!

Key point: Raft использует log-based leader election,
а не случайный выбор. Это обеспечивает, что новый leader'
всегда имеет все committed записи.
```

**Правило "no overlap":**

```python
class SafetyRules:
    """
    Raft Safety Rules:

    State Machine Safety (R5'45):
    If a server applies a log entry at a given index
    to its state machine, no other server will apply
    a different log entry for the same index.
    """

    def on_apply_entry(self, index: int, entry):
        """
        Гарантия: этот entry никогда не будет
        перезаписан другим entry'ем на этом индексе
        """
        # Это гарантировано благодаря:
        # 1. Quorum'у при выборе лидера
        # 2. Leader Completeness
        # 3. Log Matching
        pass

    def can_apply_entry(self, index: int) -> bool:
        """
        Entry может быть applied только если:
        1. Он в логе лидера
        2. Большинство followers получило его
        3. Лидер из текущего term'а это знает
        """
        # commitIndex = максимальный индекс такой что:
        # - entry[commitIndex].term == currentTerm
        # - большинство имеет entry[commitIndex]
        return index <= self.commit_index
```

### Пример

```python
class SafetyDemo:
    """
    Демонстрация safety guarantees в Raft
    """

    def demo_election_safety(self):
        """
        Property 1: No two leaders in same term
        """
        print("=== Election Safety ===")

        # Cluster: 5 узлов
        nodes = ['A', 'B', 'C', 'D', 'E']

        # Term 1: A gets 3 votes -> leader
        print("Term 1:")
        print(f"  A wins with votes from: A, B, C (quorum=3)")
        print(f"  Leader A starts")

        # B doesn't get leader because:
        print(f"  B can get at most: B, D, E (2 votes)")
        print(f"  Not enough for quorum (need >=3)")

        print("\nKey mechanism: voted_for persistence")
        print("  A, B, C save voted_for='A' on disk")
        print("  D, E save voted_for=None")
        print("  Even after crash, this persists")

    def demo_leader_append_only(self):
        """
        Property 2: Leader only appends, never deletes
        """
        print("\n=== Leader Append-Only ===")

        print("Leader A's log operations:")
        log = []

        # Operation 1
        log.append(("SET", "x", 5))
        print(f"1. Append: {log}")

        # Operation 2
        log.append(("SET", "y", 10))
        print(f"2. Append: {log}")

        # If conflict with follower:
        print("\n3. Conflict with follower C:")
        print("   C has: [SET x=5, SET y=100] (different!)")
        print("   A doesn't delete own entry!")
        print("   Instead: sends prevLogIndex=1, prevLogTerm=1")
        print("   C removes its entry and adds A's entry")
        print("   Result: All have [SET x=5, SET y=10]")

    def demo_log_matching(self):
        """
        Property 3: Log Matching -> State Machine Safety
        """
        print("\n=== Log Matching Property ===")

        print("Key insight: If node X and Y have identical entry at index N:")
        print("  log[X][N] == log[Y][N]")
        print("Then: log[X][0..N] == log[Y][0..N]")
        print("This holds for all entries they've applied")

        print("\nProof by induction:")
        print("Base: First entry (index 0) comes from same leader, same term")
        print("      -> Same entry if both have it")
        print("Step: If entries 0..N-1 same, and entry N same")
        print("      -> Entry N+1 from same source (if they have it)")

    def demo_no_rollback(self):
        """
        Демонстрация: Committed записи никогда не теряются
        """
        print("\n=== No Rollback of Committed Entries ===")

        print("Scenario: Entry committed in term T")
        print("  1. Leader has entry in index N, term T")
        print("  2. Majority has entry")
        print("  3. Leader commits it")

        print("\nLater: New term T+1, different leader elected")
        print("  Can new leader have different entry at index N? NO!")
        print("  Why:")
        print("    - Old majority contained >= 1 node with entry N, term T")
        print("    - Any majority for new leader overlaps with old majority")
        print("    - So new leader must have seen entry N, term T")
        print("    - New leader won't overwrite it (append-only)")
        print("    - New leader will replicate it forward")
```

### Типичные ошибки
1. **Думать, что log matching достаточно**: Нужна также quorum voting
2. **Забыть о term'е entries**: Старые entries из других leader'ов не должны быть переписаны
3. **Неправильно считать majority**: При 6 узлах нужно 4, не 3

### На интервью
- Объясните три safety properties: Election Safety, Leader Append-Only, Log Matching
- Дайте пример: почему quorum необходимо
- Объясните Leader Completeness (hardest part)
- Уточняющий вопрос: "Может ли committed запись быть потеряна?" (Ответ: Нет, потому что большинство имеет её и любое новое большинство пересекается)

---

## 7. Как обрабатывается split-brain в Raft?

### Зачем спрашивают
Split-brain — это демон распределённых систем. Интервьюеры хотят знать, как Raft его предотвращает.

### Короткий ответ
Raft предотвращает split-brain через quorum voting. Если сеть разделится на две группы, только группа с большинством узлов сможет избрать нового лидера и продолжить работу. Группа с меньшинством узлов не сможет достичь quorum и останется неработающей, предотвращая конфликтующие обновления.

### Детальный разбор

**Проблема Split-brain:**

```
Классическая проблема без консенсуса:

Система 3 узла: [A, B, C]
Leader: A

NETWORK PARTITION:
┌─────┐           ┌─────┐
│ A   │   (crash) │ B, C│
└─────┘           └─────┘

Без Raft:
  A think it's still leader -> commits writes
  B, C think A is dead -> elect B as leader -> commits writes

  Result: Two leaders! SPLIT-BRAIN!
  When network heals: Conflicting data!
```

**Raft решает это через quorum:**

```
Система 3 узла: [A, B, C]
Leader: A (term=5)

NETWORK PARTITION:
┌─────┐           ┌─────────┐
│ A   │   (crash) │ B   C   │
└─────┘           └─────────┘

Raft with quorum (need 2/3 = 2 votes):

Group 1 (A alone):
  A's term timeout fires
  A becomes candidate in term 6
  A votes for itself: 1 vote
  B, C are partitioned -> no response
  A gets 1 vote < quorum (2)
  -> A remains candidate, can't commit anything

Group 2 (B, C):
  B's election timeout fires
  B becomes candidate in term 6
  B requests vote from C
  C votes for B: quorum achieved (2/3 = 2 votes)
  B becomes leader
  B can commit entries!

Result:
  Group 1 (minority): Can't commit
  Group 2 (majority): Can commit
  -> NO SPLIT-BRAIN!
  -> Data consistency maintained!
```

**Ключевое наблюдение: Пересечение Quorum'ов:**

```
Доказательство что не может быть двух лидеров:

Система N узлов
Quorum size Q = (N/2 + 1)

Если есть два quorum'а:
  Q1 = N/2 + 1
  Q2 = N/2 + 1

Их минимальное пересечение:
  Q1 ∩ Q2 >= (N/2 + 1) + (N/2 + 1) - N
           = N + 2 - N
           = 2

То есть пересечение >=1 узел

Следствие:
  Если A выбран лидером в term T с quorum Q1
  То Q1 содержит A и большинство узлов

  Когда выборы в term T+1:
  Candidate B попросит голос
  Любой узел из Q1 не будет голосовать B (проголосовал в term T)
  B не может получить большинство (есть overlap с Q1)

  -> Только уже выбранный лидер может быть лидером в term T
```

**Split-brain Timeline:**

```
Time    Group 1 (A)          Group 2 (B,C)
────────────────────────────────────────────
 0      A is leader(T=5)     Connected to A

 0.5    PARTITION HAPPENS
        ────────────────────
 1      A: Election timeout  B: Election timeout
        A -> Candidate T=6   B -> Candidate T=6
        A votes for self     B requests vote from C

 2      A: 1 vote (self)     C votes for B
        < quorum (need 2)    B: 2 votes = QUORUM
        A can't commit       B becomes LEADER

 3      Client tries write   Client write succeeds!
        on A:                on B's group:
        A appends to log     B replicates to C
        A can't commit       Both C get entry
        (not quorum)         (majority has it)
        -> Request fails     -> Committed!

 4      PARTITION HEALS
        A -------connected------ B, C

        A sees B has term 6 (higher than its 5)
        A becomes follower, adopts B's term
        A discards its uncommitted entries
        A fetches B's committed entries

Result:
  A's uncommitted writes: LOST (ok, never committed)
  B's committed writes: PRESERVED
  No data loss for committed data!
```

**Детали реализации:**

```python
class SplitBrainPrevention:
    def __init__(self, node_id: str, cluster_size: int):
        self.node_id = node_id
        self.cluster_size = cluster_size
        self.quorum_size = cluster_size // 2 + 1  # Большинство

        self.log = []
        self.current_term = 0
        self.state = "follower"
        self.voted_for = None

    def can_commit(self, ack_count: int) -> bool:
        """
        Проверить может ли быть commit
        ack_count = количество узлов, которые ответили OK
        (включая leader'а самого)
        """
        return ack_count >= self.quorum_size

    def can_be_elected(self, votes: int) -> bool:
        """
        Проверить может ли быть elected лидером
        """
        return votes >= self.quorum_size

    def start_election(self):
        """
        Попытка стать лидером
        """
        self.current_term += 1
        self.voted_for = self.node_id
        votes = 1  # Голосуем за себя

        # Отправить RequestVote RPC на других
        # ... (в реальности асинхронно)

        # Если получили quorum голосов:
        if votes >= self.quorum_size:
            print(f"{self.node_id}: Became leader in term {self.current_term}")
            self.state = "leader"
        else:
            print(f"{self.node_id}: Failed to get quorum in term {self.current_term}")
            self.state = "follower"

    def apply_state_machine(self, entry):
        """
        Применить entry к состоянию
        ТОЛЬКО если он commitment (большинство имеет)
        """
        # Leader только commit'ит когда большинство ответило
        # Followers только применяют когда leader сказал (via commitIndex)
        pass
```

### Пример

```python
class SplitBrainDemo:
    """
    Симуляция split-brain scenario
    """

    def simulate(self):
        print("=== Raft Split-Brain Prevention Demo ===\n")

        # 5-узловой кластер
        nodes = {
            'A': {'term': 5, 'state': 'leader', 'votes': 0},
            'B': {'term': 5, 'state': 'follower', 'votes': 0},
            'C': {'term': 5, 'state': 'follower', 'votes': 0},
            'D': {'term': 5, 'state': 'follower', 'votes': 0},
            'E': {'term': 5, 'state': 'follower', 'votes': 0},
        }
        quorum = 3

        print("Initial state:")
        print(f"  Leader: A (term 5)")
        print(f"  Followers: B, C, D, E (term 5)")
        print(f"  Quorum size: {quorum}")

        print("\n--- PARTITION HAPPENS ---")
        print("Group 1: [A] (minority)")
        print("Group 2: [B, C, D, E] (majority)")

        print("\n--- TIME: Elections Start ---")
        # Group 1
        print("\nGroup 1 (A):")
        print(f"  A timeout -> become candidate (term 6)")
        print(f"  A: votes for self: 1")
        print(f"  A: waiting for responses from B, C, D, E... (no network)")
        print(f"  A: votes < quorum (1 < 3)")
        print(f"  A: election FAILED, remains candidate")
        print(f"  A: CANNOT COMMIT NEW WRITES")

        # Group 2
        print("\nGroup 2 (B, C, D, E):")
        print(f"  B timeout -> become candidate (term 6)")
        print(f"  B: votes for self: 1")
        print(f"  B: RequestVote(term=6) -> C")
        print(f"  B: RequestVote(term=6) -> D")
        print(f"  B: RequestVote(term=6) -> E")
        print(f"  B: got vote from C: 2 votes")
        print(f"  B: got vote from D: 3 votes = QUORUM!")
        print(f"  B: became LEADER (term 6)")
        print(f"  B: CAN COMMIT NEW WRITES")

        print("\n--- TIME: Write Requests ---")
        print("\nClient tries: SET x=10")
        print("\n  On A (minority):")
        print(f"    A: append to log, try to replicate")
        print(f"    A: send AppendEntries to B, C, D, E... (no network)")
        print(f"    A: acks < quorum (only self: 1 < 3)")
        print(f"    A: cannot commit")
        print(f"    A: reply to client: FAIL")

        print("\n  On B (majority):")
        print(f"    B: append to log")
        print(f"    B: send AppendEntries to C, D, E")
        print(f"    B: got OK from C, D: acks = 3 (self+C+D)")
        print(f"    B: acks >= quorum (3 >= 3): COMMIT")
        print(f"    B: reply to client: SUCCESS")

        print("\n--- TIME: Partition Heals ---")
        print("\nA is now connected to B, C, D, E!")
        print("\nA receives AppendEntries(term=6) from B:")
        print(f"  A: term 6 > my term 5")
        print(f"  A: update term to 6, become follower")
        print(f"  A: discard uncommitted entries")
        print(f"  A: fetch committed entries from B")
        print(f"\nResult:")
        print(f"  A's unfulfilled write: LOST (was never committed)")
        print(f"  B's committed write: PRESERVED everywhere")
        print(f"  NO SPLIT-BRAIN!")
        print(f"  DATA CONSISTENCY MAINTAINED!")
```

### Типичные ошибки
1. **Думать, что leader'а достаточно**: Нужен quorum для commit'а
2. **Забыть про network partition**: Меньшинство должно остановиться
3. **Неправильный quorum расчёт**: 3 узла нужно 2, 4 узла нужно 3, 5 узлов нужно 3

### На интервью
- Нарисуйте диаграмму: группы после partition
- Объясните, почему меньшинство не может commit'ить
- Дайте пример: что происходит с данными во время partition
- Уточняющий вопрос: "Что если partition длится 10 минут?" (Ответ: группа с меньшинством просто не может ничего делать, но система safe)

---

## 8. Чем Raft отличается от Paxos?

### Зачем спрашивают
Сравнение алгоритмов показывает глубокое понимание. Интервьюеры хотят знать, почему выбрать Raft vs Paxos.

### Короткий ответ
Raft и Paxos оба решают консенсус, но по-разному: (1) Raft использует лидера для упорядочивания (simpler), Paxos более distributed; (2) Raft разделяет problem на части (elections, replication, safety), Paxos более abstract; (3) Paxos tolerates Byzantine фaults, Raft нет; (4) Raft: проще понимать и реализовать, Paxos: более "оптимальный" теоретически.

### Детальный разбор

**Taблица сравнения:**

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│ Аспект               │ Raft                 │ Paxos                │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Leader               │ Один в каждом term   │ Нет single leader     │
│                      │ (simplifies things)  │ (distributed voting)  │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Сложность            │ Проще (разделена на  │ Сложно (abstract)    │
│                      │ 3 части)             │                       │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Fault tolerance      │ f < N/2              │ f < N/3 (Byzantine)  │
│                      │ (crash faults)       │                       │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Log replication      │ Leader-driven        │ Proposer-Acceptor    │
│                      │ (straightforward)    │ (more complex)        │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Safety               │ Strong (TLA+ proven) │ Strong (mathemat.)   │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Практическое        │ etcd, Consul,        │少用 (less popular)   │
│ применение          │ CockroachDB          │                       │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Implementation       │ 500-700 строк        │ 1000+ строк          │
│ difficulty          │ (относительно)       │ (более сложно)        │
└──────────────────────┴──────────────────────┴──────────────────────┘
```

**Обзор Paxos:**

```
Paxos делит участников на три роли:
1. Proposer  - предлагает значения
2. Acceptor  - голосует за значения
3. Learner   - узнает consensual value

Один node может быть все три роли одновременно.

Два этапа (phases):
1. Prepare phase:
   - Proposer отправляет prepare request с номером N
   - Acceptors отвечают, если N выше их highest seen N
   - Acceptors обещают не принимать ничего с N ниже

2. Accept phase:
   - Proposer отправляет accept request
   - Acceptors принимают если выполнили promise из phase 1
   - Learners узнают консенсус

Problem: Livelock возможен, если несколько proposers одновременно!
```

**Диаграмма: Paxos vs Raft:**

```
PAXOS:
┌───────────────────────────────────────────────────┐
│ Proposer1 ── Prepare(N=1)  ────┐                  │
│                               │                   │
│ Proposer2 ── Prepare(N=2) ──┐ │                   │
│                             │ v                   │
│                        ┌──────────────┐            │
│                        │  Acceptors   │            │
│                        │ [A, B, C]    │            │
│                        └──────────────┘            │
│                             ^ ^                    │
│                             │ │                    │
│ Proposer1 ── Accept(N=1) ──┐ │                    │
│ Proposer2 ── Accept(N=2) ──┘─┘                    │
│                                                    │
│ Result: Two proposals competing, complex         │
│         Must use higher-numbered proposer         │
└───────────────────────────────────────────────────┘

RAFT:
┌───────────────────────────────────────────────────┐
│ Candidate1 ── RequestVote ────┐                   │
│                               v                   │
│                        ┌──────────────┐            │
│                        │  Voters      │            │
│                        │ [A, B, C]    │            │
│                        └──────────────┘            │
│                               ^                    │
│                               │                    │
│ Candidate2 ── RequestVote ────┘                   │
│                                                    │
│ Result: Winner takes all (quorum votes)          │
│         Clear leader, simpler process             │
└───────────────────────────────────────────────────┘
```

**Raft: Подход на основе лидера:**

```
Term: Leader election фаза
  ┌─────────────────────────────────────────┐
  │ 1. Follower timeout                     │
  │ 2. Become candidate, request votes      │
  │ 3. Quorum grants votes                  │
  │ 4. Become leader (clear winner!)        │
  └─────────────────────────────────────────┘

Entry replication phase
  ┌─────────────────────────────────────────┐
  │ Leader gets write request               │
  │ Leader appends to own log               │
  │ Leader sends to followers               │
  │ Followers append                        │
  │ Leader waits for quorum acks            │
  │ Leader commits, tells followers         │
  │ All apply to state machine              │
  └─────────────────────────────────────────┘

Paxos: Proposer-Acceptor approach

Prepare phase
  ┌─────────────────────────────────────────┐
  │ Proposer sends prepare(N)               │
  │ Acceptors promise not to accept < N     │
  │ Return highest (ballot, value) accepted │
  └─────────────────────────────────────────┘

Accept phase
  ┌─────────────────────────────────────────┐
  │ Proposer sends accept(N, value)         │
  │ Acceptors accept if no higher N seen    │
  │ Send confirmation to learners           │
  └─────────────────────────────────────────┘

Problem: If multiple proposers, can conflict!
  P1 sends prepare(1)
  P2 sends prepare(2)
  P1 sends accept(1) -> rejected (P2 has 2)
  P2 sends accept(2)
  -> Now P1 must retry with higher number
  -> Can cause livelock!
```

**Устойчивость к отказам:**

```
Raft (crash failures):
  - f < N/2 (can tolerate minority failures)
  - If f = 1: need 3 nodes
  - If f = 2: need 5 nodes

  Example: 5 nodes, can tolerate 2 failures
  ┌─ Leader (healthy)
  ├─ Follower (healthy)
  ├─ Follower (healthy)
  ├─ Crashed
  ├─ Crashed

  Majority (3) still healthy, work continues

Paxos (Byzantine failures):
  - f < N/3 (can tolerate < 1/3 failures, even malicious)
  - If f = 1: need 4 nodes (instead of 3 for Raft)
  - If f = 2: need 7 nodes (instead of 5 for Raft)

  More expensive, but handles Byzantine!
```

### Пример

```python
class RaftVsPaxos:
    """
    Сравнение реализации
    """

    def raft_implementation(self):
        """
        Raft: простая и понятная
        """
        print("=== RAFT Implementation ===\n")

        code = '''
class RaftNode:
    def __init__(self, id, peers):
        self.state = "follower"
        self.term = 0
        self.log = []

    # 1. Leader election (clear mechanism)
    def election(self):
        self.term += 1
        self.state = "candidate"
        votes = self.request_votes()
        if votes > len(self.peers) // 2:
            self.state = "leader"

    # 2. Log replication (straightforward)
    def append_entry(self, value):
        self.log.append(value)
        for peer in self.peers:
            peer.replicate(self.log)

    # 3. Safety (quorum-based)
    def commit(self):
        count = sum(1 for p in self.peers if p.has_entry)
        if count > len(self.peers) // 2:
            self.state_machine.apply(self.log)
        '''
        print(code)
        print("Lines of code: ~50-100 for core logic")
        print("Understandability: HIGH")

    def paxos_implementation(self):
        """
        Paxos: более сложная абстракция
        """
        print("\n=== PAXOS Implementation ===\n")

        code = '''
class PaxosProposer:
    def propose(self, value):
        ballot_number = self.get_next_ballot()
        # Phase 1: Prepare
        promises = self.send_prepare(ballot_number)
        if len(promises) > len(self.acceptors) // 2:
            highest_value = max(p.value for p in promises)
            # Phase 2: Accept
            accepts = self.send_accept(ballot_number, highest_value)
            if len(accepts) > len(self.acceptors) // 2:
                return highest_value
        else:
            # Conflict! Must retry with higher ballot
            return self.propose(value)  # Retry!

class PaxosAcceptor:
    def __init__(self):
        self.promised_ballot = None
        self.accepted_ballot = None
        self.accepted_value = None

    def prepare(self, ballot):
        if ballot > self.promised_ballot:
            self.promised_ballot = ballot
            return (self.accepted_ballot, self.accepted_value)
        else:
            raise REJECT

    def accept(self, ballot, value):
        if ballot >= self.promised_ballot:
            self.accepted_ballot = ballot
            self.accepted_value = value
        else:
            raise REJECT
        '''
        print(code)
        print("\nLines of code: ~150-200+ for core logic")
        print("Understandability: LOWER (abstract phases)")
        print("Risk: Livelock if multiple proposers conflict")
```

### Типичные ошибки
1. **Путаница ролей**: Raft = leader + followers, Paxos = proposer + acceptor + learner
2. **Думать, что Paxos проще**: На практике наоборот
3. **Забывать про Byzantine**: Paxos нужен if you need Byzantine tolerance

### На интервью
- Объясните: Raft использует leader, Paxos нет
- Дайте таблицу: сложность, performance, fault tolerance
- Объясните: почему Raft проще в реальности
- Уточняющий вопрос: "Когда использовать Paxos?" (Ответ: если нужна Byzantine tolerance или очень decentralized system)

---

## 9. Где применяется Raft на практике?

### Зачем спрашивают
Real-world применение показывает практическое понимание. Интервьюеры хотят знать, где и как Raft используется.

### Короткий ответ
Raft используется в modern distributed systems: etcd (key-value store для Kubernetes, coordination), Consul (service mesh и distributed configuration), CockroachDB (distributed SQL database), HashiCorp Vault (secrets management). Все эти системы требуют strong consistency и высокой availability.

### Детальный разбор

**etcd — Kubernetes Config Store:**

```
┌──────────────────────────────────────────────────────┐
│ etcd: Distributed key-value store                    │
├──────────────────────────────────────────────────────┤
│                                                      │
│ Kubernetes использует etcd для:                      │
│ - Хранить конфигурацию кластера                     │
│ - Pod, Service, Deployment definitions              │
│ - Leader election для control plane                  │
│ - Watch API (notifications на изменения)            │
│                                                      │
│ Архитектура:                                        │
│ ┌────────────────┐  ┌────────────────┐              │
│ │ etcd node 1    │  │ etcd node 3    │              │
│ │ (Leader)       │  │ (Follower)     │              │
│ │ Raft consensus │  │ Replicates     │              │
│ └────────────┬───┘  └────────┬───────┘              │
│              │                │                      │
│              └────────┬───────┘                      │
│                       │                              │
│              ┌────────v───────┐                     │
│              │ etcd node 2    │                     │
│              │ (Follower)     │                     │
│              └────────────────┘                     │
│                                                      │
│ Clients (API Server, kubelet, etc) read/write      │
│ через Raft consensus layer                          │
└──────────────────────────────────────────────────────┘
```

**Consul — Service Discovery:**

```
Consul uses Raft for:
1. Catalog of services
2. Leader election
3. ACL tokens storage
4. Configuration (key-value)

Example:
  ┌─────────────────────────────────┐
  │ Consul Cluster (Raft)           │
  │ ┌──────────────────────────┐    │
  │ │ Leader                   │    │
  │ │ - Accepts writes         │    │
  │ │ - Replicates via Raft    │    │
  │ └───────────┬──────────────┘    │
  │             │ append_entries    │
  │      ┌──────v─────────┐         │
  │      │ Followers      │         │
  │      │ - Read-only    │         │
  │      │ - Replicate    │         │
  │      └────────────────┘         │
  └─────────────────────────────────┘
         │
         │ DNS/HTTP API
         v
    ┌─────────────────────┐
    │ Applications query  │
    │ "Where is my DB?"   │
    │ Gets response from  │
    │ Raft consensus      │
    └─────────────────────┘
```

**CockroachDB — Distributed SQL:**

```
CockroachDB: SQL database с распределённостью

┌─────────────────────────────────────────────┐
│ CockroachDB Cluster                         │
├─────────────────────────────────────────────┤
│                                             │
│ Raft используется для:                      │
│ 1. Consensus on data blocks (ranges)        │
│ 2. Ensure strong consistency                │
│ 3. Replication across nodes                 │
│                                             │
│ Architecture:                               │
│ ┌───────────────┐  ┌───────────────┐       │
│ │ Node 1        │  │ Node 2        │       │
│ │ Range [a-m]   │  │ Range [a-m]   │       │
│ │ Raft group 1  │  │ (replica)     │       │
│ └──────┬────────┘  └──────┬────────┘       │
│        │                  │                 │
│        └────────┬─────────┘                 │
│               (Raft consensus)              │
│                                             │
│ ┌───────────────┐  ┌───────────────┐       │
│ │ Node 1        │  │ Node 3        │       │
│ │ Range [n-z]   │  │ Range [n-z]   │       │
│ │ Raft group 2  │  │ (replica)     │       │
│ └──────┬────────┘  └──────┬────────┘       │
│        │                  │                 │
│        └────────┬─────────┘                 │
│               (Raft consensus)              │
│                                             │
│ SQL transactions use Raft for consistency  │
└─────────────────────────────────────────────┘
```

**Сравнение систем:**

```
┌───────────────┬──────────┬───────────────┬────────────┐
│ System        │ Use case │ Raft role     │ Nodes      │
├───────────────┼──────────┼───────────────┼────────────┤
│ etcd          │ Config   │ Consensus     │ 3-5 usually│
│               │ store    │ for all data  │            │
├───────────────┼──────────┼───────────────┼────────────┤
│ Consul        │ Service  │ Consensus for │ 3-5 servers│
│               │ discovery│ state         │ per DC     │
├───────────────┼──────────┼───────────────┼────────────┤
│ CockroachDB   │ SQL DB   │ Per-range     │ Any number │
│               │          │ consensus     │            │
├───────────────┼──────────┼───────────────┼────────────┤
│ Vault         │ Secrets  │ Consensus for │ 3-5 seals  │
│               │          │ encrypted key │            │
└───────────────┴──────────┴───────────────┴────────────┘
```

**Характеристики производительности:**

```
etcd:
  - Write latency: ~10-50ms
  - Throughput: ~10,000 ops/sec on single instance
  - Consistency: Strong (all reads from leader)
  - Use case: Kubernetes API, control plane

Consul:
  - Write latency: ~20-100ms
  - Throughput: Variable (depends on consistency settings)
  - Consistency: Tunable (can read from followers with stale data)
  - Use case: Service mesh, configuration management

CockroachDB:
  - Write latency: ~50-200ms (across network)
  - Throughput: Depends on cluster size and schema
  - Consistency: Strong (SERIALIZABLE isolation)
  - Use case: Full SQL database, OLTP
```

### Пример

```python
class PracticalRaftExamples:
    """
    Примеры использования Raft в реальных системах
    """

    def etcd_example(self):
        """
        etcd example: Kubernetes configuration
        """
        print("=== etcd in Kubernetes ===\n")

        print("Scenario: Deploy new application")
        print("1. kubectl apply -f deployment.yaml")
        print("2. API Server writes Deployment to etcd")
        print("3. etcd's Raft ensures consistency:")
        print("   - Leader appends to log")
        print("   - Replicates to followers")
        print("   - Waits for majority quorum")
        print("4. Commit happens, all replicas apply")
        print("5. Controllers watch etcd for changes")
        print("6. Reconciliation loop starts")

        print("\nKey guarantee:")
        print("  - Never two API servers with conflicting deployments")
        print("  - No split-brain (quorum prevents it)")
        print("  - Ordered log of operations")

    def consul_example(self):
        """
        Consul example: Service discovery
        """
        print("\n=== Consul Service Discovery ===\n")

        print("Scenario: Register service and discover it")
        print("1. Application registers itself: /v1/agent/service/register")
        print("2. Consul leader commits to Raft")
        print("3. Replicates to followers")
        print("4. Service catalog in consensus state")
        print("5. Other apps query: /v1/catalog/service/myapp")
        print("6. Get list of healthy instances")

        print("\nRaft ensures:")
        print("  - Catalog consistent across all nodes")
        print("  - Deregistration is atomic")
        print("  - No inconsistent views of services")

    def cockroachdb_example(self):
        """
        CockroachDB example: Distributed transactions
        """
        print("\n=== CockroachDB Distributed SQL ===\n")

        print("Scenario: Write to distributed table")
        print("Table distributed by key range:")
        print("  Range 1: Keys 'a' to 'm'")
        print("  Range 2: Keys 'n' to 'z'")

        print("\nWrite: INSERT INTO users VALUES ('alice', 25)")
        print("1. Determine range: key 'alice' -> Range 1")
        print("2. Send write to Range 1 leader")
        print("3. Range 1 leader appends to Raft log")
        print("4. Raft consensus: replicate to Range 1 followers")
        print("5. When quorum acks, commit")
        print("6. Return success to client")

        print("\nRaft ensures:")
        print("  - Strong consistency within range")
        print("  - No data loss (replicated)")
        print("  - ACID guarantees")

    def failure_scenario(self):
        """
        Что происходит при failure
        """
        print("\n=== Failure Scenario ===\n")

        print("etcd cluster: [A(leader), B, C]")
        print("Write: PUT /kubernetes.io/config/apiserver")

        print("\nNormal path:")
        print("  A leader appends to log")
        print("  A sends to B, C")
        print("  B, C ack: majority!")
        print("  A commits, tells B, C")
        print("  Return success")

        print("\nFailure: Network partition [A] vs [B, C]")
        print("  A: append to log, try to send to B, C")
        print("  B, C: no message, timeout, elect new leader")
        print("  B becomes leader (term 2)")
        print("  A still thinks it's leader (term 1)")
        print("  A tries to append: no quorum, fails")
        print("  B accepts writes, gets quorum, commits")
        print("  A's write: REJECTED")
        print("  B's write: COMMITTED")
        print("")
        print("Partition heals:")
        print("  A sees B has term 2 > 1")
        print("  A becomes follower")
        print("  A gets B's committed data")
        print("  NO CONFLICT (quorum prevented split-brain)")
```

### Типичные ошибки
1. **Думать, что Raft только для tiny clusters**: CockroachDB использует на сотнях узлов
2. **Забывать про quorum read overhead**: etcd может быть slow для reads
3. **Неправильные expectations по performance**: Raft → disk I/O → latency

### На интервью
- Объясните: как etcd используется в Kubernetes
- Дайте примеры трёх разных систем и их use cases
- Объясните: почему consensus нужен каждой из систем
- Уточняющий вопрос: "Может ли Raft быть bottleneck?" (Ответ: да, если много writes; solutions: batching, pipelining)

---

## 10. Как реализовать distributed lock с помощью Raft?

### Зачем спрашивают
Практический вопрос о использовании Raft. Интервьюеры хотят знать, как построить higher-level primitives на Raft.

### Короткий ответ
Distributed lock с помощью Raft работает через consensus state machine. Все операции с lock'ом (acquire, release) реплицируются через Raft. Lock'ом может овладеть только один клиент за раз (гарантировано Raft consensus). Освобождение тоже через consensus, обеспечивая linearizability и preventing deadlocks.

### Детальный разбор

**Базовый механизм:**

```
Distributed Lock = State Machine + Consensus

┌──────────────────────────────┐
│ Raft Consensus Layer         │
│ (replicates all operations)  │
└──────────────┬───────────────┘
               │
               v
┌──────────────────────────────────┐
│ Lock State Machine              │
│ locks = {                        │
│   "/database": {                 │
│     holder: "server1",           │
│     acquired_time: 1000,         │
│     lease_timeout: 5000          │
│   }                              │
│ }                                │
└──────────────────────────────────┘

Operations (all through Raft):
1. ACQUIRE "/database" by "server1"
   -> Raft log: [term, ACQUIRE, "/database", "server1"]
   -> Apply: locks["/database"] = {holder: "server1", ...}

2. RELEASE "/database" by "server1"
   -> Raft log: [term, RELEASE, "/database"]
   -> Apply: delete locks["/database"]

All replicas have SAME lock state (consistency!)
```

**Архитектура:**

```
┌─────────────────────────────────────────────────────────┐
│ Lock Server Cluster                                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│ │ Server A     │  │ Server B     │  │ Server C     │   │
│ │ (Leader)     │  │ (Follower)   │  │ (Follower)   │   │
│ ├──────────────┤  ├──────────────┤  ├──────────────┤   │
│ │ Raft         │  │ Raft         │  │ Raft         │   │
│ │ consensus    │  │ replication  │  │ replication  │   │
│ ├──────────────┤  ├──────────────┤  ├──────────────┤   │
│ │ Lock SM:     │  │ Lock SM:     │  │ Lock SM:     │   │
│ │ "/db"  ->A   │  │ "/db"  ->A   │  │ "/db"  ->A   │   │
│ │ "/tbl" ->B   │  │ "/tbl" ->B   │  │ "/tbl" ->B   │   │
│ └──────────────┘  └──────────────┘  └──────────────┘   │
│         │                │                │             │
│         │ RPC: ACQUIRE   │ append_entries  │             │
│         │    "/db"       │                 │             │
│         └────────────────┼─────────────────┘             │
│                          │                               │
│                    Log entry added                       │
│              When committed (quorum acks):               │
│              All apply ACQUIRE to SM                     │
│              All agree: "/db" locked by X                │
└─────────────────────────────────────────────────────────┘
```

**Гарантии и граничные случаи:**

```
1. Mutual Exclusion (mutex):
   - Только один holder за раз
   - Гарантировано consensus state machine
   - Если двое пытаются ACQUIRE одновременно:
     -> Попадают в очередь Raft log'а в порядке
     -> Первый в log'е (consensus order) получит

2. Deadlock prevention (lease timeout):
   - Lock имеет TTL (time to live)
   - Если holder не REFRESH (пролонгирует):
     -> Lock автоматически освобождается
   - Prevents: holder crashed -> lock held forever

3. Linearizability:
   - Все операции видны в порядке Raft log'а
   - ACQUIRE всегда happens-before RELEASE
   - No race conditions (consensus обеспечивает порядок)

4. Network partition:
   - Client в minority group: ACQUIRE fails (no quorum)
   - Client в majority group: ACQUIRE succeeds
   - No split-brain locks
```

**Пример: Distributed Lock реализация:**

```python
class DistributedLockRaft:
    """
    Distributed lock using Raft consensus
    """

    def __init__(self, raft_node):
        self.raft = raft_node  # Underlying Raft consensus
        self.locks = {}  # {lock_name: holder, timeout}

    def acquire_lock(self, lock_name: str, holder_id: str,
                    lease_duration_ms: int) -> bool:
        """
        Acquire lock through Raft consensus
        """
        # Create command for Raft
        command = {
            "op": "ACQUIRE",
            "lock": lock_name,
            "holder": holder_id,
            "lease": lease_duration_ms,
            "timestamp": time.time_ms()
        }

        # Submit to Raft (only leader accepts writes)
        if not self.raft.is_leader():
            raise NotLeaderError()

        # Append to log and replicate
        index = self.raft.append_log_entry(command)

        # Wait for commit
        # (in real impl: use index, wait for apply notification)
        self.raft.wait_for_apply(index, timeout=5000)

        # Check result
        if lock_name in self.locks:
            if self.locks[lock_name]["holder"] == holder_id:
                return True  # Successfully acquired

        return False  # Someone else got it

    def release_lock(self, lock_name: str, holder_id: str) -> bool:
        """
        Release lock through Raft
        """
        command = {
            "op": "RELEASE",
            "lock": lock_name,
            "holder": holder_id
        }

        if not self.raft.is_leader():
            raise NotLeaderError()

        index = self.raft.append_log_entry(command)
        self.raft.wait_for_apply(index, timeout=5000)

        return lock_name not in self.locks

    def apply_command(self, command):
        """
        Apply command to state machine (called when committed)
        This is called on ALL nodes with same order
        """
        if command["op"] == "ACQUIRE":
            lock_name = command["lock"]
            holder = command["holder"]
            lease = command["lease"]
            acquired_time = command["timestamp"]

            # Check if lock is free (or expired)
            if lock_name in self.locks:
                expired = (time.time_ms() -
                          self.locks[lock_name]["acquired_time"] >
                          self.locks[lock_name]["lease"])

                if not expired:
                    # Lock held, operation fails
                    # But still logged (idempotent)
                    return False

            # Acquire lock
            self.locks[lock_name] = {
                "holder": holder,
                "acquired_time": acquired_time,
                "lease": lease
            }
            return True

        elif command["op"] == "RELEASE":
            lock_name = command["lock"]
            holder = command["holder"]

            if lock_name in self.locks:
                if self.locks[lock_name]["holder"] == holder:
                    del self.locks[lock_name]
                    return True

            return False

    def refresh_lock(self, lock_name: str, holder_id: str,
                    new_lease_ms: int) -> bool:
        """
        Refresh lock lease (extend timeout)
        """
        command = {
            "op": "REFRESH",
            "lock": lock_name,
            "holder": holder_id,
            "lease": new_lease_ms,
            "timestamp": time.time_ms()
        }

        index = self.raft.append_log_entry(command)
        self.raft.wait_for_apply(index, timeout=5000)

        return lock_name in self.locks

    def check_lock(self, lock_name: str) -> Optional[str]:
        """
        Check who holds lock (can read from any node)
        """
        if lock_name in self.locks:
            holder_info = self.locks[lock_name]

            # Check if expired
            elapsed = time.time_ms() - holder_info["acquired_time"]
            if elapsed > holder_info["lease"]:
                # Expired, but not yet cleaned up
                return None

            return holder_info["holder"]

        return None
```

**Сложные моменты:**

```
1. Отказ лидера во время ACQUIRE:
   Проблема: Клиент отправляет ACQUIRE, лидер падает перед репликацией
   Решение:
     - Клиент не знает, был ли захвачен lock
     - Необходимо использовать timeout и retry при отсутствии ответа
     - Гарантия Raft: либо полностью применено, либо вообще не применено

2. Расхождение часов в lease timeout:
   Проблема: Разные серверы показывают разное время
   Решение:
     - Использовать логические timestamps (Raft term + index)
     - Или синхронизировать часы (NTP)
     - Raft не зависит от точности часов

3. Thundering herd при освобождении lock:
   Проблема: 1000 клиентов ждут lock, все пытаются захватить
   Решение:
     - Упорядоченная очередь в Raft log
     - Следующий ожидающий детерминирован
     - Не нужна отдельная очередь

4. Отзыв lock для безопасности:
   Проблема: Владелец упал, но lock ещё не истёк → зависание
   Решение:
     - Админ может отозвать lock через команду REVOKE
     - Или использовать более короткие lease
     - Или реализовать heartbeat-based renewal
```

### Пример

```python
class DistributedLockDemo:
    """
    Demo of distributed lock with Raft
    """

    def demo_basic_lock(self):
        print("=== Distributed Lock Demo ===\n")

        print("Scenario: Multiple servers competing for resource")
        print("Servers: [A, B, C] (Raft cluster)")
        print("Resource: Database migration lock\n")

        print("Time t=0:")
        print("  Server A: ACQUIRE /migration for 10 seconds")
        print("  -> Raft: Append log entry [ACQUIRE, /migration, A]")
        print("  -> Replicate to B, C")
        print("  -> B, C ack")
        print("  -> Majority! Commit")
        print("  -> A, B, C all apply: locks[/migration] = {holder: A}")
        print("  Result: A LOCKED ✓")

        print("\nTime t=1:")
        print("  Server B: ACQUIRE /migration for 10 seconds")
        print("  -> Raft: Append log entry [ACQUIRE, /migration, B]")
        print("  -> Check state machine: lock already held by A")
        print("  -> Apply returns false")
        print("  Result: B FAILED (already locked)")

        print("\nTime t=5:")
        print("  Server A: REFRESH /migration for 10 more seconds")
        print("  -> Raft: Append [REFRESH, /migration, A]")
        print("  -> Update lease timeout")
        print("  Result: A's lock extended")

        print("\nTime t=11 (no refresh from A):")
        print("  Server B: ACQUIRE /migration")
        print("  -> Check: Lock expired (11 - 5 > 10)")
        print("  -> Raft: Append [ACQUIRE, /migration, B]")
        print("  -> No holder, so grant!")
        print("  Result: B LOCKED ✓")

        print("\nKey guarantees:")
        print("  - Never two servers hold lock simultaneously")
        print("  - Order is strictly by Raft log")
        print("  - No split-brain (quorum prevents it)")
        print("  - Automatic timeout prevents deadlock")

    def demo_network_partition(self):
        print("\n=== Network Partition Scenario ===\n")

        print("Setup: 5-server cluster [A, B, C, D, E]")
        print("Lock held by A\n")

        print("PARTITION: [A] vs [B, C, D, E]")

        print("\nMinority [A]:")
        print("  A tries RELEASE /lock")
        print("  -> Append to log")
        print("  -> Try to replicate... (network down)")
        print("  -> No acks from B, C, D, E")
        print("  -> Operation fails (no quorum)")
        print("  Result: RELEASE REJECTED")

        print("\nMajority [B, C, D, E]:")
        print("  B is elected new leader (B, C, D, E vote)")
        print("  C tries ACQUIRE /lock (lock expired or forced)")
        print("  -> Leader B appends [ACQUIRE, /lock, C]")
        print("  -> Replicates to D, E")
        print("  -> Quorum! Commit")
        print("  Result: C LOCKED ✓")

        print("\nPartition heals:")
        print("  A sees new term from B/C/D/E")
        print("  A becomes follower")
        print("  A fetches committed log from B")
        print("  A learns C has lock")
        print("  NO CONFLICT (quorum prevented split-brain)")
```

### Типичные ошибки
1. **Забыть про lease timeout**: Lock может быть held forever
2. **Неправильно обработать leader failure**: Client не знает если ACQUIRE был применён
3. **Race condition с clock skew**: Разные серверы думают lock expired в разное время

### На интервью
- Объясните: State Machine + Raft = distributed lock
- Нарисуйте: как ACQUIRE проходит через Raft
- Дайте пример: что если leader падает во время ACQUIRE?
- Уточняющий вопрос: "Как избежать thundering herd?" (Ответ: Raft log гарантирует порядок, следующий в очереди берёт lock)

---

## Дополнение: Trade-offs и когда НЕ использовать Raft

### Когда Raft хорош:
- Strong consistency требуется
- Crash failures (не Byzantine)
- Reasonable cluster size (3-9 узлов)
- Acceptable latency (10-100ms)

### Когда Raft может быть не оптимален:
- **Extreme performance needed**: Raft → log replication → disk I/O
- **Byzantine adversaries**: Используйте PBFT или BFT алгоритмы
- **Very large clusters**: Quorum для 1000 узлов → половине нужно ack → slow
- **High availability > consistency**: Eventual consistency может быть проще
- **Real-time требования**: Raft latency может быть неприемлемым

### Оптимизации Raft:
1. **Log compaction**: Снимать snapshots для уменьшения лога
2. **Batching**: Группировать несколько команд в одну Raft entry
3. **Pipelining**: Отправлять несколько entries одновременно
4. **Async replication**: Не ждать acks от всех (но lose safety гарантий)

---

## Заключение

Raft — это elegant алгоритм consensu**, разработанный для **understandability** и **practicality**. Его используют в production системах (etcd, Consul, CockroachDB), и понимание Raft essential для system design interviews.

**Ключевые takeaways:**
- Terms + Quorum voting = Election Safety
- Append-only log + Log Matching = Leader Completeness
- These three guarantee State Machine Safety
- Practical, not just theoretical
- Компромиссы: consistency vs performance vs complexity

На интервью будьте готовы:
1. Объяснить complete жизненный цикл запроса
2. Нарисовать диаграммы: elections, replication, partitions
3. Дать примеры из реальных систем
4. Обсудить trade-offs и limitations
