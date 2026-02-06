# Модели консистентности в распределённых системах

**Навигация:** [Трек System Design](./README.md) · Предыдущая тема: [17-distributed-transactions](./17-distributed-transactions.md) · Следующая тема: [19-failure-modes](./19-failure-modes.md)

---

## Ключевые концепции

Прежде чем переходить к вопросам, разберёмся с базовыми терминами, которые будут встречаться в этом модуле.

**Linearizability** — самая строгая модель консистентности, где все операции выполняются в одном глобальном порядке для всех клиентов, как будто выполняются последовательно на одной машине. Нужна для критичных операций вроде платежей, банковских транзакций и системы резервирования, где порядок и корректность абсолютно критичны.

**Sequential Consistency** — модель, где операции выглядят выполненными в каком-то последовательном порядке, но разные клиенты могут видеть разные порядки выполнения одних и тех же операций. Слабее линеаризуемости, но проще реализовать и она обеспечивает лучшую производительность.

**Causal Consistency** — модель, где причинно-связанные события видны всем в правильном причинном порядке. Хороший баланс между консистентностью и производительностью, подходит для социальных сетей, чатов и других приложений, где причинность важнее глобального порядка.

**Eventual Consistency** — самая слабая модель консистентности, где если не будет новых обновлений, все реплики рано или поздно станут одинаковыми. Позволяет максимальную доступность и масштабируемость системы, но допускает временную несогласованность между репликами.

**Strong Consistency** — общее название для моделей консистентности на левой стороне спектра (linearizable, sequential), которые гарантируют правильность и порядок. Гарантирует правильность операций, но жертвует производительностью и доступностью при сетевых разделениях.

**Weak Consistency** — общее название для моделей на правой стороне спектра (eventual, conflict-free), которые предпочитают производительность и доступность. Обеспечивает максимальную производительность и масштабируемость, но допускает временные несогласованности между репликами.

**Session Consistency** — модель, где в рамках вашей личной сессии вы видите свои изменения сразу же, но другие клиенты видят их с задержкой. Хороший компромисс для веб-приложений, где пользователь видит свои действия мгновенно, но может переоткрыть браузер и вернуть консистентность.

**Read-your-writes** — гарантия, что ваши собственные записи видны вам сразу же при чтении, даже если вы читаете с разных реплик. Минимальная гарантия консистентности для удобства пользователя и корректности приложения.

**Isolation Levels** — уровни изоляции ACID свойств в одной базе данных (Read Uncommitted, Read Committed, Repeatable Read, Serializable), которые определяют, какие грязные читания, lost updates и гонки данных возможны при одновременных транзакциях.

**Monotonic Reads** — гарантия, что если вы прочитали значение с версией V, все последующие чтения вернут это значение или более новую версию. Предотвращает путанный опыт, когда данные кажутся «откатываются назад» между чтениями.

**Replication Lag** — задержка между записью данных на основной сервер (primary) и её появлением на репликах. Объясняет, почему eventual consistency существует и требуется выбор подходящей модели консистентности для вашего приложения.

---

## 1. Какие модели консистентности существуют?

### Зачем спрашивают
Нужно понимать спектр возможностей от максимальной консистентности до максимальной доступности. Это фундамент для выбора архитектуры системы.

### Короткий ответ
Существует спектр моделей: **Strong Consistency** (linearizability, sequential) → **Weak Consistency** (causal, eventual). Выбор зависит от требований приложения к корректности и производительности.

### Детальный разбор

**Спектр моделей консистентности:**

```
┌─────────────────────────────────────────────────────────────┐
│           СПЕКТР МОДЕЛЕЙ КОНСИСТЕНТНОСТИ                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  СИЛЬНАЯ            СРЕДНЯЯ              СЛАБАЯ             │
│  КОНСИСТЕНТНОСТЬ    КОНСИСТЕНТНОСТЬ      КОНСИСТЕНТНОСТЬ    │
│                                                              │
│  Linearizability    Causal               Eventual            │
│       ↓             ↓                    ↓                   │
│   Sequential        Session              Conflict-free       │
│       ↓             ↓                    ↓                   │
│  Strict Serializability                  BASE                │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│  Гарантии:          Гарантии:            Гарантии:          │
│  • Один порядок     • Причинно-          • Отсутствие       │
│    для всех           связанные            конфликтов в     │
│  • Видны все          события в            вашей сессии     │
│    записи перед       правильном          • Рано или поздно  │
│    чтением           порядке               сходимость       │
│  • Высокая          • Персональная       • Высокая скорость  │
│    латентность        сессия видит       • Риск несоответ.  │
│                       изменения                             │
├─────────────────────────────────────────────────────────────┤
│  Примеры:           Примеры:             Примеры:           │
│  • Банки             • Соцсети            • DNS              │
│  • Платёжи           • Email              • Кэши             │
│  • Системы           • Чаты               • Социальные сети  │
│    резервирования    (часто)              (sometimes)        │
└─────────────────────────────────────────────────────────────┘
```

**Математическое описание:**
- **Linearizability**: Для каждой операции можно определить точку в времени, в которую она произошла. Все операции выглядят так, как будто выполнились атомарно в этой точке.
- **Sequential Consistency**: Результат выполнения совпадает с каким-то последовательным выполнением операций, но точка выполнения может различаться для разных клиентов.
- **Causal Consistency**: Причинно-связанные события видны в правильном порядке.
- **Eventual Consistency**: Если нет новых обновлений, все реплики рано или поздно станут идентичны.

### Пример

```python
# Модель: Банк с разными уровнями консистентности

class LinearizableBank:
    """Линеаризуемый банк - один порядок для всех"""
    def __init__(self):
        self.balance = 0
        self.lock = threading.Lock()

    def transfer(self, amount):
        with self.lock:
            current = self.balance
            self.balance = current + amount
            return self.balance

class EventuallyConsistentBank:
    """Самосогласованный банк - разные ордера возможны"""
    def __init__(self):
        self.local_balance = 0
        self.updates = []
        self.pending = []

    def transfer(self, amount):
        # Сразу применяем локально
        self.local_balance += amount
        # Помечаем на отправку другим репликам
        self.pending.append(amount)
        return self.local_balance

    def get_balance(self):
        # Может вернуть старое значение
        return self.local_balance

    def sync(self):
        # Синхронизируем с другими репликами
        self.updates.extend(self.pending)
        self.pending = []

# Использование
linear_bank = LinearizableBank()
eventual_bank = EventuallyConsistentBank()

# Linearizable: гарантия - все видят одинаковый порядок
# Eventual: быстрее, но временно разные значения
```

### Типичные ошибки
1. **Думать, что eventual consistency = никакой консистентности** - На самом деле это гарантия сходимости, просто нет гарантии на точное время
2. **Выбирать strong consistency везде** - убивает масштабируемость и доступность
3. **Не учитывать требования приложения** - Нужно анализировать, какие инварианты критичны
4. **Путать модели** - Linearizability строже Sequential Consistency

### На интервью
**Как отвечать:**
1. Сначала нарисовать спектр (5 сек)
2. Назвать основные модели (10 сек)
3. Объяснить базовые гарантии каждой (20 сек)
4. Привести примеры систем (15 сек)

**Возможные follow-up:**
- "Какую выбрать для чата?" → Causal or Sequential
- "Какую для платежей?" → Linearizable (или Serializable)
- "Почему eventual consistency popular?" → Масштабируемость и доступность

---

## 2. Что такое linearizability и зачем она нужна?

### Зачем спрашивают
Linearizability - это самое строгое и понятное определение корректности. Её нужно различать с другими моделями.

### Короткий ответ
**Linearizability** = все операции выполняются в одном глобальном порядке, который видят все клиенты. Как если бы была одна машина с глобальным часами. Нужна для операций, где порядок критичен: платежи, зарезервирование, синхронизированные операции.

### Детальный разбор

**Определение Linearizability:**
Операция линеаризуема, если существует точка в реальном времени (между её началом и концом), в которой она атомарно произошла, и все операции выглядят выполненными в порядке этих точек для всех наблюдателей.

```
LINEARIZABLE TIMELINE:

Client A:  Write(x=1)
           │
           ├─────────────┐
           │ Point of    │
           │ linearization
           ├─────────────┤
           │             ├──→ [x=1 для всех]
           │             │
Client B:  │      Read(x)→[всегда видит x=1]
           │
           └────────────┐
                        │
                    Global Order
                    для всех


НЕ-LINEARIZABLE:

Client A:  Write(x=1)
           │
           ├────────────┐
           │            │ [x=1]
           │            │
Client B:  │  Read(x)───┤ [видит x=0] ← Нарушение!
           │            │ потому что операции
           │            │ не в глобальном порядке
           │            │
           └────────────┘
```

**Ключевые свойства:**
1. **Atomicity**: Операция либо произойдёт полностью, либо вообще не произойдёт
2. **Total Order**: Все операции выстроены в один порядок
3. **Real-time Order**: Если операция A закончилась перед началом B, то A раньше B в порядке
4. **Single-writer**: Для простоты часто один писатель (или координатор)

**Примеры linearizable систем:**
- **Consensus** (Raft, Paxos) - гарантируют linear order
- **Centralized databases** - одна машина, естественно linear
- **Coordination services** (Zookeeper, etcd) - явно linearizable

**Примеры НЕ-linearizable:**
- **Асинхронная репликация** - между репликами задержки
- **Кэши** - часто содержат старые данные
- **DNS** - намеренно eventual consistency

### Пример

```python
import time
import threading

class LinearizableRegister:
    """Линеаризуемый регистр с помощью лидера"""
    def __init__(self):
        self.value = None
        self.version = 0
        self.lock = threading.Lock()

    def write(self, new_value):
        """Только лидер может писать"""
        with self.lock:
            self.version += 1
            v = self.version
            self.value = new_value
            return v

    def read(self):
        """Всегда читаем от лидера"""
        with self.lock:
            return self.value, self.version


class NonLinearizableCache:
    """НЕ-линеаризуемый кэш с репликацией"""
    def __init__(self):
        self.replicas = [
            {"value": None, "version": 0},
            {"value": None, "version": 0},
        ]
        self.primary = 0

    def write(self, new_value):
        """Асинхронная репликация"""
        version = self.replicas[self.primary]["version"] + 1
        self.replicas[self.primary]["value"] = new_value
        self.replicas[self.primary]["version"] = version

        # Асинхронно шлём в другую реплику (может потеряться)
        threading.Thread(
            target=self._async_replicate,
            args=(new_value, version),
            daemon=True
        ).start()

    def _async_replicate(self, value, version):
        time.sleep(0.1)  # Задержка в сети
        replica_idx = 1 - self.primary
        if version > self.replicas[replica_idx]["version"]:
            self.replicas[replica_idx]["value"] = value
            self.replicas[replica_idx]["version"] = version

    def read(self):
        """Может читать из replica с отставанием"""
        replica_idx = (self.primary + 1) % 2  # читаем из backup
        return self.replicas[replica_idx]["value"]


# Тест linearizability
def test_linearizability():
    reg = LinearizableRegister()
    results = []

    def writer():
        for i in range(10):
            reg.write(f"value_{i}")

    def reader():
        for _ in range(20):
            results.append(reg.read())

    t1 = threading.Thread(target=writer)
    t2 = threading.Thread(target=reader)

    t1.start()
    t2.start()
    t1.join()
    t2.join()

    # Все reads вернули один из write'ов (никогда не видели старое значение
    # после более нового)
    print(f"Linearizable results: {results}")


test_linearizability()
```

### Типичные ошибки
1. **Путать linearizability с isolation levels** - Linearizability про видимость между сессиями, не про ACID
2. **Думать что lock = linearizability** - Lock помогает, но не достаточно в распределённых системах
3. **Предполагать что consensus = linearizability** - Consensus помогает, но нужно правильно его использовать
4. **Забывать про real-time order** - Критичная часть определения linearizability

### На интервью
**Как отвечать:**
1. "Linearizability - это когда все видят один глобальный порядок операций"
2. Нарисовать диаграмму с timeline
3. Показать пример: "Если я write(x=1), а потом ты читаешь, ты видишь x=1"
4. Назвать где используется: "Платежи, зарезервирование, лидер в consensus"

**Возможные follow-up:**
- "Linearizable или available?" → CAP theorem
- "Как реализовать?" → Consensus с лидером
- "Стоимость?" → Одна машина или координатор - bottleneck

---

## 3. Что такое sequential consistency?

### Зачем спрашивают
Sequential consistency слабее linearizability, но проще реализовать. Нужно различать эти две модели.

### Короткий ответ
**Sequential Consistency** = результат выполнения совпадает с каким-то последовательным выполнением, но разные клиенты могут видеть разные порядки операций (главное - без циклов). Слабее linearizability, так как не требует real-time order.

### Детальный разбор

**Сравнение Linearizability и Sequential:**

```
LINEARIZABILITY:                SEQUENTIAL CONSISTENCY:

Write(x=1)  ────┐              Write(x=1)  ────┐
               │                              │
               ├─→ Глобальная                 ├─→ Есть ЧТО-ТО
               │    точка в                   │    консистентное
               │    реальном                  │    последовательное
               │    времени                   │    выполнение
               │                              │
Read(x)     ────┘                Read(x)     ────┘
Видит: 1                         Видит: 1 (или 0, но не нарушает
                                 последовательность для других)


ПРИМЕР НАРУШЕНИЯ LINEARIZABILITY НО НЕ SEQUENTIAL:

Реалькое время:    t1  t2  t3  t4
                   │   │   │   │
Thread 1:  Write(x=1)
                ────┴──────┐
Thread 2:                Read(x)→0  ← После write, видит старое!
                           ────┴──────┐
Thread 3:                        Read(x)→1  ← Позже видит новое

Это нарушает linearizability (real-time order),
но может быть sequential consistency
если есть последовательность операций,
которая объясняет это (Write→Read1→Read2).
```

**Математическое определение:**
1. Все процессы "согласны" на порядок операций (но не обязательно глобальный)
2. Порядок соответствует program order в каждом процессе
3. Нет циклических зависимостей

**Где используется Sequential Consistency:**
- **Многопроцессорные системы** - кэши с protocol (e.g., MSI protocol)
- **Системы с несколькими версиями** - если версионирование правильное
- **Некоторые NoSQL БД** - когда требуется сбалансировать consistency и performance

### Пример

```python
# Sequential vs Linearizable visibility

class SequentialConsistencyModel:
    """Модель Sequential Consistency"""
    def __init__(self):
        self.memory = {"x": 0}
        self.write_log = []

    def write(self, key, value):
        """Запись в память"""
        self.memory[key] = value
        self.write_log.append((key, value))
        # Важно: не глобальная точка во времени!

    def read(self, key):
        """Чтение - видит последний write"""
        return self.memory[key]


class TwoReplicasSequential:
    """Две реплики с sequential consistency"""
    def __init__(self):
        self.replicas = [
            {"value": 0, "timestamp": 0},  # Replica A
            {"value": 0, "timestamp": 0},  # Replica B
        ]

    def write(self, new_value):
        """Пишем в обе реплики без синхронизации"""
        ts = time.time()
        self.replicas[0]["value"] = new_value
        self.replicas[0]["timestamp"] = ts

        # Асинхронно в другую реплику
        self.replicas[1]["value"] = new_value
        self.replicas[1]["timestamp"] = ts

    def read_from_replica_a(self):
        """Всегда читаем из одной реплики"""
        return self.replicas[0]["value"]

    def read_from_replica_b(self):
        """Или из другой"""
        return self.replicas[1]["value"]

    @property
    def is_sequential_consistent(self):
        """Проверяем: есть ли последовательное объяснение?"""
        # Если оба записали в одно время - есть объяснение
        return self.replicas[0]["timestamp"] == self.replicas[1]["timestamp"]


# Пример нарушения linearizability но не sequential

def demonstrate_sequential_not_linear():
    """
    Показываем случай, когда есть sequential consistency
    но нет linearizability
    """

    class MultiCopyConsistency:
        def __init__(self):
            self.copy_a = {"x": 0}
            self.copy_b = {"x": 0}
            self.operation_order = []

        def write(self, copy, key, value):
            """Пишем в одну из копий"""
            if copy == "A":
                self.copy_a[key] = value
            else:
                self.copy_b[key] = value
            self.operation_order.append(f"Write({key}={value})")

        def read(self, copy, key):
            """Читаем из одной из копий"""
            if copy == "A":
                return self.copy_a[key]
            else:
                return self.copy_b[key]

    m = MultiCopyConsistency()

    # Сценарий: Write in A, Read from B (видит старое)
    m.write("A", "x", 1)
    result = m.read("B", "x")  # Может вернуть 0 (старое значение)

    # Нарушение linearizability: write произошёл раньше read в реальном времени,
    # но read видит старое значение

    # Но это может быть sequential consistent:
    # если есть последовательный порядок, который объясняет это
    return result


demonstrate_sequential_not_linear()
```

### Типичные ошибки
1. **Думать что sequential = слабая консистентность** - Наоборот, это вполне сильно для многих приложений
2. **Путать с causal consistency** - Sequential требует согласованности всех, causal - только причинно-связанные
3. **Забывать про program order** - Каждый процесс должен видеть свои операции в порядке
4. **Не учитывать real-time order** - Вот чем sequential отличается от linearizable

### На интервью
**Как отвечать:**
1. "Sequential consistency - это когда есть КАКОЙ-ТО последовательный порядок для всех"
2. "Отличие от linearizable: не требуется real-time order"
3. "Пример: две реплики, читаем из любой - могут быть разновидности в видимости"
4. "Используется в системах памяти, многопроцессорных системах"

**Возможные follow-up:**
- "Разница с linearizable?" → real-time order
- "Разница с causal?" → Causal только для зависимых, sequential для всех
- "Как реализовать?" → Через версионирование или write-through кэши

---

## 4. Что такое causal consistency?

### Зачем спрашивают
Causal consistency - практичная модель между sequential и eventual. Часто встречается в real-world системах: соцсети, чаты, документы.

### Короткий ответ
**Causal Consistency** = если событие A причинно влияет на B, то все видят A перед B. Но независимые события могут быть видны в разном порядке. Достаточно сильна для большинства приложений, при этом масштабируемая.

### Детальный разбор

**Причинно-связанные события:**

```
ПРИМЕР 1: Соцсеть

User A пишет пост (Event Write_Post)
              │
              ├─→ Причинная зависимость
              │
User B видит пост и комментирует (Event Comment)
              │
              ├─→ Причинная зависимость
              │
User C видит комментарий (Event See_Comment)

Causal Consistency гарантирует:
Write_Post ──→ Comment ──→ See_Comment
  (всегда в этом порядке для всех)


ПРИМЕР 2: Чат

10:00 Alice: "Hello"  ──→ Причина
              │
10:01 Bob: "Hi there" ──→ Должен видеть Hello перед этим
              │
10:02 Alice: "How are you?"


Causal требует: Alice и Bob и все зрители видят эту последовательность
Но если Alice отправит второе сообщение в 10:02, а Bob - в 10:01,
Алиса может видеть сообщения в порядке:
  - "Hello" (своё)
  - "Hi there" (от Bob)
  - "How are you?" (своё)

Ботом может видеть:
  - "Hi there" (своё)
  - "Hello" (от Alice) <- Может быть позже, но перед второй Alice

Главное: если Hello причинно связана с Hi there,
то все видят Hello перед Hi there.
```

**Как реализовать Causal Consistency:**

```
ВАРИАНТ 1: Vector Clocks (классический подход)

struct Message {
    content: String,
    vector_clock: {
        "Alice": 2,
        "Bob": 1,
        "Charlie": 0,
    }
}

Правило: Сообщение M2 может быть доставлено только если
мой vector clock >= vector clock M1 для всех M1 в causal history M2


ВАРИАНТ 2: Lamport Timestamps + dependency tracking

struct Message {
    content: String,
    lamport_timestamp: 42,
    depends_on: [Message_id_1, Message_id_2, ...],
}

Правило: Перед доставкой M2 убедись, что все зависимости
из depends_on уже доставлены


ВАРИАНТ 3: Session tokens (для read-your-writes)

Client делает:
  Write(key, value)  ← Получает session_token = {version: 1}
  Read(key)  ← Шлёт session_token = {version: 1}
              ← Сервер читает с версии >= 1
```

**Гарантии causal consistency:**
1. **Respect causality** - Причинно-связанные события в правильном порядке
2. **Concurrent independence** - Независимые события могут быть в любом порядке
3. **FIFO consistency** - События от одного отправителя видны в порядке отправки
4. **Per-session consistency** - Твои операции видны в правильном порядке

### Пример

```python
from dataclasses import dataclass
from typing import Dict, List, Set

@dataclass
class VectorClock:
    """Vector clock для causal consistency"""
    clock: Dict[str, int]

    def increment(self, process_id: str):
        """Увеличиваем наш счётчик"""
        self.clock[process_id] = self.clock.get(process_id, 0) + 1

    def merge(self, other: 'VectorClock'):
        """Мерджим два vector clock"""
        for process, ts in other.clock.items():
            self.clock[process] = max(self.clock.get(process, 0), ts)

    def happens_before(self, other: 'VectorClock') -> bool:
        """Проверяем: self < other (happens before)?"""
        less_or_equal = all(
            self.clock.get(p, 0) <= other.clock.get(p, 0)
            for p in set(list(self.clock.keys()) + list(other.clock.keys()))
        )
        strictly_less = any(
            self.clock.get(p, 0) < other.clock.get(p, 0)
            for p in set(list(self.clock.keys()) + list(other.clock.keys()))
        )
        return less_or_equal and strictly_less

    def concurrent(self, other: 'VectorClock') -> bool:
        """Проверяем: параллельны ли два события?"""
        return not self.happens_before(other) and not other.happens_before(self)


@dataclass
class Message:
    """Сообщение с vector clock"""
    sender: str
    content: str
    vector_clock: VectorClock
    id: str


class CausalConsistencyChat:
    """Чат с causal consistency гарантиями"""
    def __init__(self, participants: List[str]):
        self.participants = participants
        self.messages: List[Message] = []
        self.delivered: Set[str] = set()
        self.pending: List[Message] = []

        # Локальный vector clock каждого участника
        self.local_clocks = {p: VectorClock({pp: 0 for pp in participants})
                            for p in participants}

    def send_message(self, sender: str, content: str):
        """Отправляем сообщение"""
        # Увеличиваем свой счётчик
        vc = self.local_clocks[sender]
        vc.increment(sender)

        msg = Message(
            sender=sender,
            content=content,
            vector_clock=VectorClock(dict(vc.clock)),
            id=f"{sender}_{len(self.messages)}"
        )

        self.messages.append(msg)
        print(f"📤 {sender} отправил: {content}")
        print(f"   Vector clock: {vc.clock}")

        return msg

    def can_deliver_message(self, msg: Message, receiver: str) -> bool:
        """Можем ли доставить сообщение?

        Правило: сообщение M с vector clock VC может быть доставлено
        только если:
        - Для каждого другого процесса P:
          VC[P] <= местный_clock[receiver][P]
        """
        receiver_vc = self.local_clocks[receiver]

        for process in self.participants:
            msg_count = msg.vector_clock.clock.get(process, 0)
            my_count = receiver_vc.clock.get(process, 0)

            if msg_count > my_count + (1 if process == msg.sender else 0):
                return False

        return True

    def deliver_message(self, receiver: str):
        """Доставляем очередное сообщение, если можем"""
        for msg in self.pending:
            if self.can_deliver_message(msg, receiver):
                # Доставляем
                print(f"📥 {receiver} получил: '{msg.content}' от {msg.sender}")

                # Обновляем local clock
                self.local_clocks[receiver].merge(msg.vector_clock)
                self.local_clocks[receiver].increment(receiver)

                self.delivered.add(msg.id)
                self.pending.remove(msg)
                return True

        return False

    def propagate_to_all(self, msg: Message):
        """Пропагируем сообщение всем участникам"""
        for receiver in self.participants:
            if receiver != msg.sender and msg.id not in self.delivered:
                self.pending.append(msg)

        # Доставляем всем кто готов
        for receiver in self.participants:
            if receiver != msg.sender:
                while self.deliver_message(receiver):
                    pass


# Пример использования
def test_causal_chat():
    print("=== CAUSAL CONSISTENCY CHAT ===\n")
    chat = CausalConsistencyChat(["Alice", "Bob", "Charlie"])

    # Alice пишет
    msg1 = chat.send_message("Alice", "Hello everyone!")
    chat.propagate_to_all(msg1)
    print()

    # Bob отвечает (причинно зависит от Alice)
    msg2 = chat.send_message("Bob", "Hi Alice!")
    chat.propagate_to_all(msg2)
    print()

    # Charlie присоединяется (видит оба)
    msg3 = chat.send_message("Charlie", "Hi folks!")
    chat.propagate_to_all(msg3)
    print()

    print("✓ Все сообщения доставлены с соблюдением causality!")


test_causal_chat()
```

### Типичные ошибки
1. **Думать что causal = порядок всех сообщений** - Только причинно-связанных!
2. **Забывать про vector clocks** - Основной инструмент реализации
3. **Путать с session consistency** - Session consistency = read-your-writes, это часть causal
4. **Предполагать что causal дешевле eventual** - На самом деле дороже, но дешевле linear

### На интервью
**Как отвечать:**
1. "Causal consistency - это когда причинно-связанные события видны в правильном порядке"
2. "Пример: если пост вышел ПЕРЕД комментарием, все видят пост перед комментарием"
3. "Реализуется через vector clocks или dependency tracking"
4. "Достаточно для чатов, соцсетей, документов"

**Возможные follow-up:**
- "Как реализовать vector clocks?" → Рассказать алгоритм
- "Память?" → O(N) на сообщение для N участников
- "Чем отличается от sequential?" → Sequential для всех, causal только для зависимых

---

## 5. Что такое eventual consistency?

### Зачем спрашивают
Eventual consistency - основа масштабируемых распределённых систем. Нужно понимать гарантии, которые она даёт и не даёт.

### Короткий ответ
**Eventual Consistency** = если прекратить писать обновления, все реплики рано или поздно содержат одни и те же данные. Нет гарантий на точное время сходимости, но гарантия сходимости есть. Используется везде: DNS, CDN, кэши, социальные сети.

### Детальный разбор

**Модель eventual consistency:**

```
┌─────────────────────────────────────────────────────────────┐
│           EVENTUAL CONSISTENCY TIMELINE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Replica A    Replica B    Replica C                        │
│     x=0          x=0          x=0       ← Initial state     │
│      │            │            │                            │
│  Write(x=1)       │            │                            │
│      │            │            │                            │
│      x=1          x=0          x=0      ← Расходятся       │
│      │     ┌──────┘            │                            │
│      │     │ Асинхронная       │                            │
│      │     │ репликация        │                            │
│      │     ├─────────────────┐ │                            │
│      x=1    x=1    x=1  ← Сходятся со временем             │
│                                                              │
│  ════════════════════════════════════════════════════════   │
│  "Eventually Consistent State": x=1 на всех репликах        │
│  ════════════════════════════════════════════════════════   │
│                                                              │
└─────────────────────────────────────────────────────────────┘

ГАРАНТИЯ:
  Если Write(x=1) и нет новых Writes,
  то ∃ T такое что
  для всех t > T все Reads вернут x=1

  Но T не определено! Может быть 100ms или 10 минут.
```

**Гарантии eventual consistency:**
1. **Eventual agreement** - Все реплики сойдутся
2. **Availability** - Всегда можно читать/писать (нет блокировок)
3. **Partition tolerance** - Работает при сетевых разделениях
4. **NO strong consistency** - Может быть stale data между обновлениями

**Как обеспечить eventual consistency:**
1. **Replication** - Копирование на несколько узлов
2. **Write to any replica** - Пишем в ближайший узел
3. **Anti-entropy** - Асинхронное выравнивание (merkle trees, gossip)
4. **Conflict resolution** - Определяем что делать при конфликтах

**Разные "интерпретации" eventual consistency:**

```
Weak Eventual Consistency (WEC):
  Если нет новых обновлений, реплики сходятся
  Пример: простая async репликация

Strong Eventual Consistency (SEC):
  WEC + любые два узла с одинаковыми обновлениями
  заканчивают в одном состоянии
  Пример: CRDT (Conflict-free Replicated Data Types)

Тестируемая eventual consistency:
  После определённого времени (SLA), гарантируем сходимость
  Пример: "Синхронизируем каждый час"
```

### Пример

```python
import time
import threading
from dataclasses import dataclass
from typing import Dict, List

@dataclass
class UpdateLog:
    """Запись об обновлении"""
    timestamp: float
    key: str
    value: any
    replica_id: str


class EventuallyConsistentStore:
    """Хранилище с eventual consistency"""
    def __init__(self, replica_id: str):
        self.replica_id = replica_id
        self.data: Dict = {}
        self.update_log: List[UpdateLog] = []
        self.other_replicas: List['EventuallyConsistentStore'] = []
        self.sync_interval = 0.1  # 100ms для примера

    def write(self, key: str, value: any):
        """Пишем локально, потом асинхронно реплицируем"""
        ts = time.time()

        # Сразу пишем в себя
        self.data[key] = value
        self.update_log.append(UpdateLog(ts, key, value, self.replica_id))

        print(f"✍️  {self.replica_id}: write({key}={value})")

        # Асинхронно шлём в другие реплики
        for other in self.other_replicas:
            threading.Thread(
                target=self._replicate_to,
                args=(other, UpdateLog(ts, key, value, self.replica_id)),
                daemon=True
            ).start()

    def _replicate_to(self, other: 'EventuallyConsistentStore', update: UpdateLog):
        """Асинхронная репликация"""
        time.sleep(0.05)  # Задержка в сети
        other.apply_update(update)

    def apply_update(self, update: UpdateLog):
        """Применяем полученное обновление"""
        # Простая политика: последнее обновление по времени побеждает
        if update.key not in self.data or \
           self.get_timestamp(update.key) < update.timestamp:
            self.data[update.key] = update.value
            self.update_log.append(update)

    def get_timestamp(self, key: str) -> float:
        """Получаем timestamp последнего обновления ключа"""
        for log in reversed(self.update_log):
            if log.key == key:
                return log.timestamp
        return 0

    def read(self, key: str):
        """Читаем из локального хранилища (может быть stale)"""
        return self.data.get(key, None)

    def register_replica(self, other: 'EventuallyConsistentStore'):
        """Регистрируем другую реплику для репликации"""
        self.other_replicas.append(other)

    def sync_anti_entropy(self):
        """Anti-entropy: периодическое выравнивание логов"""
        for other in self.other_replicas:
            # Шлём свой лог для синхронизации
            for update in self.update_log:
                other.apply_update(update)


# Пример использования: DNS-like система

def test_dns_eventual_consistency():
    print("=== EVENTUAL CONSISTENCY (DNS-like) ===\n")

    # Три DNS сервера
    dns_us = EventuallyConsistentStore("DNS-US")
    dns_eu = EventuallyConsistentStore("DNS-EU")
    dns_asia = EventuallyConsistentStore("DNS-ASIA")

    # Регистрируем для репликации
    dns_us.register_replica(dns_eu)
    dns_us.register_replica(dns_asia)
    dns_eu.register_replica(dns_us)
    dns_eu.register_replica(dns_asia)
    dns_asia.register_replica(dns_us)
    dns_asia.register_replica(dns_eu)

    # Администратор обновляет DNS на одном сервере
    print("1️⃣  Admin updates example.com on US server:")
    dns_us.write("example.com", "1.2.3.4")

    # Сразу читаем - видим старое значение в других локациях
    time.sleep(0.02)
    print(f"\n2️⃣  Read example.com from EU server: {dns_eu.read('example.com')}")
    print(f"   (stale data - UPDATE NOT YET REPLICATED)")

    # Ждём репликации
    print(f"\n3️⃣  Waiting for replication...")
    time.sleep(0.15)

    # Теперь видим обновление везде
    print(f"\n4️⃣  After replication:")
    print(f"   US:   {dns_us.read('example.com')}")
    print(f"   EU:   {dns_eu.read('example.com')}")
    print(f"   ASIA: {dns_asia.read('example.com')}")
    print(f"\n✓ EVENTUALLY CONSISTENT STATE REACHED!")

    # Пример конфликта
    print(f"\n\n5️⃣  Demonstrating conflict resolution:")

    dns_us2 = EventuallyConsistentStore("DNS-US-2")
    dns_eu2 = EventuallyConsistentStore("DNS-EU-2")
    dns_us2.register_replica(dns_eu2)
    dns_eu2.register_replica(dns_us2)

    t1 = time.time()
    dns_us2.write("conflict.com", "1.1.1.1")  # Time: t1
    time.sleep(0.01)
    dns_eu2.write("conflict.com", "2.2.2.2")  # Time: t1 + 0.01

    # ждём репликации
    time.sleep(0.15)

    print(f"\nUS writes:   1.1.1.1 at t={0}")
    print(f"EU writes:   2.2.2.2 at t=+0.01")
    print(f"\nFinal state (Last-Write-Wins):")
    print(f"   US:   {dns_us2.read('conflict.com')}")
    print(f"   EU:   {dns_eu2.read('conflict.com')}")
    print(f"✓ Conflict resolved via timestamp comparison")


test_dns_eventual_consistency()
```

**Стратегии разрешения конфликтов при eventual consistency:**

```python
# 1. Last-Write-Wins (LWW)
def lww_resolve(value1, ts1, value2, ts2):
    """Побеждает последнее по времени обновление"""
    return value1 if ts1 > ts2 else value2

# 2. Vector Clock based
def vc_resolve(value1, vc1, value2, vc2):
    """Если VC'ы параллельны - конфликт, нужна app logic"""
    if vc1.happens_before(vc2):
        return value2
    elif vc2.happens_before(vc1):
        return value1
    else:
        return merge(value1, value2)  # App-specific merge

# 3. Merge (для CRDT)
def crdt_resolve(value1, value2):
    """Merge-friendly структуры (счётчики, sets)"""
    return value1 + value2  # Контекст-зависимо

# 4. Application callback
def app_resolve(value1, value2):
    """Приложение решает"""
    return max(value1, value2)  # Или min, или custom logic
```

### Типичные ошибки
1. **Думать что eventual = никакой консистентности** - На самом деле гарантия сходимости
2. **Забывать про conflict resolution** - Конфликты БУДУТ, нужно их разрешать
3. **Не учитывать stale reads** - Между write и full replication данные могут быть старыми
4. **Полагаться на временные метки без vector clocks** - Для причинных зависимостей нужны VC

### На интервью
**Как отвечать:**
1. "Eventual consistency - это гарантия что реплики РАНО ИЛИ ПОЗДНО сойдутся"
2. "Нет гарантий на время сходимости - может быть 1ms или 1 час"
3. "Основа масштабируемых систем: DNS, CDN, кэши"
4. "Требует стратегии разрешения конфликтов"

**Возможные follow-up:**
- "Как разрешать конфликты?" → LWW, VectorClocks, CRDT, application logic
- "Stale reads?" → Да, между write и replication
- "Гарантия на время?" → Обычно нет, но можно SLA

---

## 6. Что такое read-your-writes и session consistency?

### Зачем спрашивают
Практичные модели консистентности для real-world приложений. Баланс между consistency и performance.

### Короткий ответ
**Read-Your-Writes** = если ты писал value, то все последующие твои reads видят это value. **Session Consistency** = расширение: в пределах сессии видны все твои write'ы перед read'ами. Используется в web приложениях, базах данных.

### Детальный разбор

```
READ-YOUR-WRITES CONSISTENCY:

Client A:
  Write(user_data, new_profile)
        ├─→ [отправляется на сервер]
        │
  Read(user_data)
        └─→ [видит new_profile, хотя может быть в другой реплике]


SESSION CONSISTENCY (расширение read-your-writes):

Client A, Сессия #123:
  Write(settings, dark_mode=true)
        ├─→ Puts session_token = {version: 1}
        │
  Write(theme, dark)
        ├─→ Updates session_token = {version: 2}
        │
  Read(settings)
        └─→ Читает с версии >= 2 [видит все свои writes]


ПРИМЕР: Социальная сеть

Alice постит фото:

  alice.post_photo() ─┐
                     ├─→ Version_A = 1
                     │
  alice.like_photo() ─┐
                     ├─→ Version_A = 2
                     │
  alice.view_feed() ─┐
                    └─→ Видит Version_A >= 2
                        (видит свой лайк)

Bob видит ленту:

  bob.view_feed() ─┐
                  └─→ Может видеть фото БЕЗ лайка
                      (видит версию < 2)

                      ИЛИ с лайком (версию >= 2)

                      Но это нормально, потому что Bob
                      не писал лайк, это не его сессия
```

**Гарантии session consistency:**
1. **Read-Your-Writes** - Видишь свои write'ы
2. **Monotonic Reads** - Версия не уменьшается
3. **Writes Follow Reads** - Если читал версию V, то пишешь >= V
4. **Monotonic Writes** - Твои write'ы применяются в порядке

**Как реализовать:**
1. **Session ID** - Каждый клиент получает уникальный ID
2. **Version Tokens** - Отслеживаем версию в рамках сессии
3. **Session-aware read** - При read'е клиент шлёт token версии

### Пример

```python
import hashlib
import time
from dataclasses import dataclass
from typing import Dict, Any

@dataclass
class SessionToken:
    """Token для отслеживания версии в сессии"""
    session_id: str
    version: int
    timestamp: float


class SessionConsistencyStore:
    """Хранилище с session consistency гарантиями"""
    def __init__(self):
        self.data: Dict[str, Any] = {}
        self.versions: Dict[str, int] = {}  # key -> version
        self.sessions: Dict[str, SessionToken] = {}

    def create_session(self) -> SessionToken:
        """Создаём новую сессию"""
        session_id = hashlib.md5(str(time.time()).encode()).hexdigest()
        token = SessionToken(
            session_id=session_id,
            version=0,
            timestamp=time.time()
        )
        self.sessions[session_id] = token
        return token

    def write(self, session_token: SessionToken, key: str, value: Any) -> SessionToken:
        """Пишем значение в рамках сессии"""
        # Увеличиваем версию
        current_version = self.versions.get(key, 0)
        new_version = current_version + 1

        self.data[key] = value
        self.versions[key] = new_version

        # Обновляем session token
        session_token.version = max(session_token.version, new_version)
        session_token.timestamp = time.time()

        print(f"✍️  Session {session_token.session_id[:8]}: "
              f"write({key}={value}, version={new_version})")

        return session_token

    def read(self, session_token: SessionToken, key: str) -> Any:
        """Читаем значение, гарантируя видимость write'ов из этой сессии"""
        # Важно: читаем только если версия >= версия из token'а
        current_version = self.versions.get(key, 0)

        # Если current_version < session_token.version, ждём репликации
        # Или читаем из replicas с нужной версией
        if current_version < session_token.version:
            print(f"⏳  Session {session_token.session_id[:8]}: "
                  f"waiting for replication (have v{current_version}, "
                  f"need v{session_token.version})")
            # В реальности: retry или quorum read
            time.sleep(0.01)

        value = self.data.get(key, None)

        print(f"📖  Session {session_token.session_id[:8]}: "
              f"read({key})={value} (version={self.versions.get(key, 0)})")

        return value


class MultiReplicaSessionStore:
    """Несколько реплик с session consistency"""
    def __init__(self, num_replicas: int = 3):
        self.replicas = [SessionConsistencyStore() for _ in range(num_replicas)]
        self.primary = 0  # Первая реплика - primary

    def create_session(self) -> SessionToken:
        """Создаём сессию на primary"""
        return self.replicas[self.primary].create_session()

    def write(self, session_token: SessionToken, key: str, value: Any) -> SessionToken:
        """Пишем на primary, затем асинхронно реплицируем"""
        new_token = self.replicas[self.primary].write(session_token, key, value)

        # Асинхронно реплицируем на другие реплики
        import threading
        for i in range(len(self.replicas)):
            if i != self.primary:
                threading.Thread(
                    target=self._async_replicate,
                    args=(key, value, i, new_token.version),
                    daemon=True
                ).start()

        return new_token

    def _async_replicate(self, key: str, value: Any, replica_idx: int, version: int):
        """Асинхронная репликация"""
        time.sleep(0.02)  # Имитируем задержку в сети

        # Обновляем версию на replica только если это новая версия
        if self.replicas[replica_idx].versions.get(key, 0) < version:
            self.replicas[replica_idx].data[key] = value
            self.replicas[replica_idx].versions[key] = version

    def read(self, session_token: SessionToken, key: str) -> Any:
        """Читаем с session consistency гарантией"""
        # Читаем с primary (где писали)
        # Или с другой реплики, если она достаточно new

        # Стратегия 1: Всегда читаем с primary
        return self.replicas[self.primary].read(session_token, key)

        # Стратегия 2: Читаем с ближайшей реплики, которая имеет версию
        # for replica in self.replicas:
        #     if replica.versions.get(key, 0) >= session_token.version:
        #         return replica.read(session_token, key)


# Пример использования

def test_session_consistency():
    print("=== SESSION CONSISTENCY ===\n")

    store = MultiReplicaSessionStore(num_replicas=3)

    # Создаём сессию
    session = store.create_session()
    print(f"📱 Created session: {session.session_id[:8]}\n")

    # Alice в своей сессии пишет и читает
    print("1️⃣  Alice's operations:")

    session = store.write(session, "profile", {"name": "Alice", "age": 30})
    print()

    session = store.write(session, "settings", {"theme": "dark"})
    print()

    # Read-Your-Writes гарантия: видим свои write'ы
    profile = store.read(session, "profile")
    print(f"   Alice reads her profile: {profile}\n")

    settings = store.read(session, "settings")
    print(f"   Alice reads her settings: {settings}\n")

    # Monotonic reads: версия не уменьшается
    print("2️⃣  Monotonic reads guarantee:")
    v1 = store.replicas[0].versions.get("profile", 0)
    time.sleep(0.01)
    v2 = store.replicas[0].versions.get("profile", 0)
    print(f"   Version 1: {v1}")
    print(f"   Version 2: {v2}")
    print(f"   ✓ Version >= previous version\n")

    print("✓ Session consistency maintained!")


test_session_consistency()
```

### Типичные ошибки
1. **Путать session consistency с causal consistency** - Session про одного клиента, causal про причинные зависимости
2. **Забывать про version tokens** - Нужно отслеживать версию
3. **Не учитывать стоимость** - Read-your-writes может требовать read с primary
4. **Предполагать что это самое сильное** - На самом деле слабее linearizable

### На интервью
**Как отвечать:**
1. "Session consistency - это гарантия что в пределах твоей сессии видны все твои write'ы"
2. "Используем session tokens/version numbers"
3. "Достаточно для web приложений"
4. "Примеры: Facebook, Twitter, web apps в целом"

**Возможные follow-up:**
- "Как реализовать?" → Primary replica, version tokens, read от primary
- "Стоимость?" → Read может быть на primary = bottleneck
- "Альтернативы?" → Causal consistency, eventual consistency

---

## 7. Как реализовать quorum reads/writes?

### Зачем спрашивают
Quorum - фундаментальный инструмент для достижения консистентности без единого лидера. Нужно понимать как это работает.

### Короткий ответ
**Quorum** = читаем/пишем на большинстве реплик (> N/2). Если write на W реплик, а read на R реплик, и W + R > N, то читаем гарантированно последнее написанное. Основа систем без лидера: Dynamo, Cassandra.

### Детальный разбор

**Quorum math:**

```
N = количество реплик
W = количество реплик для write
R = количество реплик для read

ГАРАНТИЯ КОНСИСТЕНТНОСТИ:
  Если W + R > N, то: Read гарантированно видит Latest Write


ПРИМЕРЫ:
┌──────────────────────────────────────┐
│  N  │  W  │  R  │  W+R>N? │ Strong? │
├──────────────────────────────────────┤
│  3  │  2  │  2  │  4 > 3  │  ✓ YES  │
│  3  │  1  │  3  │  4 > 3  │  ✓ YES  │
│  3  │  3  │  1  │  4 > 3  │  ✓ YES  │
│  3  │  1  │  1  │  2 > 3  │  ✗ NO   │
│  5  │  3  │  3  │  6 > 5  │  ✓ YES  │
│  5  │  2  │  3  │  5 > 5  │  ✗ NO   │
│  5  │  3  │  2  │  5 > 5  │  ✗ NO   │
└──────────────────────────────────────┘


TRADE-OFFS:

W=N, R=1:
  ┌─ Mediums latency на write
  ├─ Fast read
  └─ Tolerate: 0 failures (N-W=0)

W=1, R=N:
  ├─ Fast write
  ├─ Slow read
  └─ Tolerate: 0 failures (N-R=0)

W=(N+1)/2, R=(N+1)/2:
  ├─ Moderate write latency
  ├─ Moderate read latency
  ├─ Strong consistency
  └─ Tolerate: (N-1)/2 failures
```

**Когда использовать:**

```
HIGH WRITE, LOW READ:
  N=3, W=1, R=3
  ├─ Быстро пишем
  ├─ Читаем со всех
  └─ Сильная консистентность

HIGH READ, LOW WRITE:
  N=3, W=3, R=1
  ├─ Медленно пишем
  ├─ Быстро читаем
  └─ Сильная консистентность

BALANCED:
  N=5, W=3, R=3
  ├─ Сбалансировано
  └─ Терпим 1 failure
```

**Read Repair и Hinted Handoff:**

```
READ REPAIR (исправление при чтении):

Client читает:
  Replica A: value=v1, version=5
  Replica B: value=v0, version=3
  Replica C: value=v1, version=5

Берём большинство (v1), затем апдейтим отстающие реплики:
  Replica B: update to value=v1, version=5

Результат: консистентность без anti-entropy работы


HINTED HANDOFF (помощь offline реплике):

Replica A down, пишем:
  Replica B: принимает write и помечает: "это for A"
  Replica C: принимает write и помечает: "это for A"

Позже:
  Replica A goes online
  Replica B & C: шлют помеченные write'ы на A
  Result: A синхронизируется
```

### Пример

```python
import time
import random
from dataclasses import dataclass
from typing import Dict, List, Any, Optional
from collections import defaultdict

@dataclass
class VersionedValue:
    """Значение с версией для resolve конфликтов"""
    value: Any
    version: int
    timestamp: float


class QuorumReplica:
    """Одна реплика в quorum системе"""
    def __init__(self, replica_id: str):
        self.replica_id = replica_id
        self.data: Dict[str, VersionedValue] = {}
        self.is_online = True

    def write(self, key: str, value: Any, version: int) -> bool:
        """Пишем значение"""
        if not self.is_online:
            return False

        self.data[key] = VersionedValue(
            value=value,
            version=version,
            timestamp=time.time()
        )
        return True

    def read(self, key: str) -> Optional[VersionedValue]:
        """Читаем значение"""
        if not self.is_online:
            return None

        return self.data.get(key)

    def repair(self, key: str, value: VersionedValue):
        """Read repair: исправляем значение"""
        if self.is_online:
            current = self.data.get(key)
            if current is None or current.version < value.version:
                self.data[key] = value


class QuorumStore:
    """Распределённое хранилище с quorum reads/writes"""
    def __init__(self, n_replicas: int, w: int, r: int):
        self.n = n_replicas
        self.w = w  # Write quorum
        self.r = r  # Read quorum
        self.version_counter = defaultdict(int)

        self.replicas = [QuorumReplica(f"Replica-{i}") for i in range(n_replicas)]

        # Проверяем что можем гарантировать консистентность
        if w + r <= n:
            print(f"⚠️  WARNING: W+R={w}+{r}={w+r} <= N={n}")
            print(f"   Strong consistency NOT guaranteed!")

    def write(self, key: str, value: Any) -> bool:
        """Пишем с quorum"""
        self.version_counter[key] += 1
        version = self.version_counter[key]

        # Пишем на W реплик
        successful_writes = 0
        for replica in self.replicas:
            if replica.write(key, value, version):
                successful_writes += 1

        success = successful_writes >= self.w

        status = "✓" if success else "✗"
        print(f"{status} Write({key}={value}): {successful_writes}/{self.w} replicas")

        return success

    def read(self, key: str) -> tuple[Optional[Any], bool]:
        """Читаем с quorum и read repair"""
        # Читаем с R реплик
        values: List[VersionedValue] = []
        read_from = []

        for i, replica in enumerate(self.replicas):
            val = replica.read(key)
            if val is not None:
                values.append(val)
                read_from.append(i)

        success = len(values) >= self.r

        if not success:
            print(f"✗ Read({key}): only {len(values)}/{self.r} replicas available")
            return None, False

        # Берём значение с максимальной версией
        best_value = max(values, key=lambda v: v.version)

        # Read repair: обновляем реплики которые вернули старую версию
        for i, replica in enumerate(self.replicas):
            if i in read_from:
                replica.repair(key, best_value)

        print(f"✓ Read({key})={best_value.value}: {len(values)}/{self.r} replicas, "
              f"version={best_value.version}")

        return best_value.value, True

    def get_consistency_level(self) -> str:
        """Возвращаем уровень консистентности"""
        if self.w + self.r > self.n:
            return "STRONG (W+R > N)"
        elif self.w + self.r == self.n:
            return "EVENTUAL (W+R = N)"
        else:
            return "WEAK (W+R < N)"

    def failover_test(self, failures: int = 1):
        """Тест: сколько failures можем выдержать?"""
        print(f"\n📊 Failover test with {failures} failures:")

        # Отключаем replicas
        for i in range(failures):
            self.replicas[i].is_online = False
            print(f"  Replica-{i} DOWN")

        # Пытаемся писать
        success = self.write("test_key", "test_value")

        # Пытаемся читать
        value, read_success = self.read("test_key")

        # Восстанавливаем
        for i in range(failures):
            self.replicas[i].is_online = True

        print(f"  Result: Write={'OK' if success else 'FAIL'}, "
              f"Read={'OK' if read_success else 'FAIL'}")

        return success and read_success


# Примеры использования

def test_quorum_strong_consistency():
    print("=== QUORUM WITH STRONG CONSISTENCY ===\n")

    # N=3, W=2, R=2 => W+R=4 > 3 => Strong!
    store = QuorumStore(n_replicas=3, w=2, r=2)
    print(f"Config: N=3, W=2, R=2")
    print(f"Consistency: {store.get_consistency_level()}\n")

    # Пишем
    store.write("key1", "value1")
    print()

    # Читаем - видим что писали
    value, success = store.read("key1")
    print()

    # Еще write
    store.write("key1", "value2")
    print()

    # Read видит latest
    value, success = store.read("key1")
    print()

    store.failover_test(failures=1)


def test_quorum_weak_consistency():
    print("\n\n=== QUORUM WITH WEAK CONSISTENCY ===\n")

    # N=5, W=2, R=2 => W+R=4 < 5 => Weak!
    store = QuorumStore(n_replicas=5, w=2, r=2)
    print(f"Config: N=5, W=2, R=2")
    print(f"Consistency: {store.get_consistency_level()}\n")

    # Пишем (быстро, только 2 реплики)
    store.write("user:123", "Alice")
    print()

    # Читаем (быстро, только 2 реплики, может быть stale)
    value, success = store.read("user:123")
    print(f"  (May see stale data)\n")

    store.failover_test(failures=2)


def test_quorum_high_availability():
    print("\n\n=== QUORUM FOR HIGH AVAILABILITY ===\n")

    # N=5, W=3, R=3 => W+R=6 > 5 => Strong
    # Но можем выдержать 2 failures на write, 2 на read
    store = QuorumStore(n_replicas=5, w=3, r=3)
    print(f"Config: N=5, W=3, R=3")
    print(f"Consistency: {store.get_consistency_level()}")
    print(f"Write tolerance: {5-3} failures")
    print(f"Read tolerance: {5-3} failures\n")

    store.write("important", "data")
    print()

    store.read("important")
    print()

    store.failover_test(failures=2)


test_quorum_strong_consistency()
test_quorum_weak_consistency()
test_quorum_high_availability()
```

### Типичные ошибки
1. **Неправильный расчёт quorum** - W + R должно быть > N для strong consistency
2. **Забывать про read repair** - Без этого медленно сходимся
3. **Не учитывать latency** - Quorum требует ждать большинства = медленнее
4. **Путать с consensus** - Quorum про консистентность, consensus про согласование

### На интервью
**Как отвечать:**
1. "Пишем на W из N реплик, читаем с R из N реплик"
2. "Если W + R > N, гарантируем strong consistency"
3. "Примеры: Dynamo, Cassandra, Riak"
4. "Trade-off: latency vs consistency vs availability"

**Возможные follow-up:**
- "Как выбрать W и R?" → Зависит от требований
- "Read repair?" → Исправляем при чтении
- "Failures?" → Можем выдержать min(N-W, N-R) failures

---

## 8. Что такое vector clocks и как они работают?

### Зачем спрашивают
Vector clocks - основной инструмент для отслеживания причинных зависимостей. Нужно понимать как они работают и где их применять.

### Короткий ответ
**Vector Clock** = каждый процесс имеет счётчик для каждого другого процесса. Когда происходит событие, увеличиваем свой счётчик. Когда отправляем сообщение, шлём весь vector clock. Позволяет определить был ли порядок событий причинной зависимостью или они параллельны.

### Детальный разбор

**Как работают vector clocks:**

```
ПРОЦЕСС A              ПРОЦЕСС B              ПРОЦЕСС C

VC=[1,0,0]             VC=[0,1,0]             VC=[0,0,1]

Event A1 ──┐
│          │
│ VC=[2,0,0]
│          │
├──────────┼──→ Шлёт сообщение с VC=[2,0,0]
           │           ├─ B получает, мерджит
           │           │  VC_B = max(VC_B, [2,0,0])
           │           │  VC_B = [2,1,0]
           │           │
           │    Event B1 ─┐
           │           │
           │           │ VC=[2,2,0]
           │           │
           │           ├────────────┼──→ Шлёт C с VC=[2,2,0]
           │           │
           │           │       ├─ C получает, мерджит
           │           │       │  VC_C = max([0,0,1], [2,2,0])
           │           │       │  VC_C = [2,2,1]
           │           │       │
           │           │    Event C1 ─┐
           │           │       │      │
           │           │       │ VC=[2,2,2]
           │           │
Event A2 ──┐
           │ VC=[3,0,0]


СРАВНЕНИЕ VECTOR CLOCKS:

[2,0,0] happens-before [2,1,0]?
  [2,0,0] <= [2,1,0]? YES (2<=2, 0<=1, 0<=0)
  [2,0,0] < [2,1,0]? YES (и есть хотя бы одна >)
  RESULT: YES, A1 -> B1

[2,2,0] happens-before [0,0,1]?
  [2,2,0] <= [0,0,1]? NO (2 > 0)
  RESULT: NO, они параллельны!

[3,0,0] vs [2,2,0]?
  [3,0,0] <= [2,2,0]? NO (3 > 2)
  [2,2,0] <= [3,0,0]? NO (2 > 0)
  RESULT: параллельны! (neither is before)
```

**Правила для vector clocks:**

```
1. ИНИЦИАЛИЗАЦИЯ:
   VC[P] = [0, 0, ..., 0] для каждого процесса P

2. ЛОКАЛЬНОЕ СОБЫТИЕ:
   VC[self] += 1

3. ОТПРАВКА СООБЩЕНИЯ:
   VC[self] += 1
   send(VC)

4. ПОЛУЧЕНИЕ СООБЩЕНИЯ:
   received_vc = message.vc
   for each process p:
       VC[p] = max(VC[p], received_vc[p])
   VC[self] += 1

5. СРАВНЕНИЕ:
   A happens-before B если:
   A.VC[i] <= B.VC[i] для всех i
   И A.VC[j] < B.VC[j] для какого-то j
```

**Примеры систем с vector clocks:**
- **Dynamo** - для conflict detection
- **Cassandra** - для causal consistency
- **Riak** - для conflict-free replication
- **Git** - для merge tracking

**Плюсы:**
- Точный порядок причинных зависимостей
- Детектируем конфликты

**Минусы:**
- O(N) память на событие (где N = число процессов)
- Может расти когда процессы приходят/уходят

### Пример

```python
from dataclasses import dataclass, field
from typing import Dict, List, Tuple

@dataclass
class VectorClock:
    """Vector clock для отслеживания причинных зависимостей"""
    clock: Dict[str, int] = field(default_factory=dict)

    def increment(self, process_id: str):
        """Увеличиваем счётчик процесса"""
        self.clock[process_id] = self.clock.get(process_id, 0) + 1

    def merge(self, other: 'VectorClock'):
        """Мерджим с другим vector clock'ом"""
        for process, ts in other.clock.items():
            self.clock[process] = max(self.clock.get(process, 0), ts)

    def happens_before(self, other: 'VectorClock') -> bool:
        """Проверяем: self < other (self happens before other)?

        A happens-before B если:
        - A.VC[i] <= B.VC[i] для всех i
        - A.VC[j] < B.VC[j] для какого-то j
        """
        all_processes = set(list(self.clock.keys()) + list(other.clock.keys()))

        less_or_equal = all(
            self.clock.get(p, 0) <= other.clock.get(p, 0)
            for p in all_processes
        )

        strictly_less = any(
            self.clock.get(p, 0) < other.clock.get(p, 0)
            for p in all_processes
        )

        return less_or_equal and strictly_less

    def concurrent(self, other: 'VectorClock') -> bool:
        """Проверяем: параллельны ли события?"""
        return (not self.happens_before(other) and
                not other.happens_before(self))

    def __repr__(self) -> str:
        items = sorted(self.clock.items())
        return "{" + ", ".join(f"{k}:{v}" for k, v in items) + "}"


@dataclass
class Event:
    """Событие с vector clock"""
    process_id: str
    name: str
    vector_clock: VectorClock
    dependencies: List['Event'] = field(default_factory=list)

    def __repr__(self) -> str:
        return f"{self.process_id}:{self.name}({self.vector_clock})"


class DistributedSystem:
    """Распределённая система с vector clock'ами"""
    def __init__(self, process_ids: List[str]):
        self.process_ids = process_ids
        self.vector_clocks = {p: VectorClock({proc: 0 for proc in process_ids})
                             for p in process_ids}
        self.events: List[Event] = []

    def local_event(self, process_id: str, event_name: str) -> Event:
        """Локальное событие в процессе"""
        vc = self.vector_clocks[process_id]
        vc.increment(process_id)

        event = Event(
            process_id=process_id,
            name=event_name,
            vector_clock=VectorClock(dict(vc.clock))
        )

        self.events.append(event)
        print(f"📍 {process_id}: {event_name} {event.vector_clock}")

        return event

    def send_message(self, sender: str, receiver: str, message: str) -> Tuple[Event, Event]:
        """Отправка сообщения между процессами"""
        # Отправитель: увеличивает свой счётчик
        sender_vc = self.vector_clocks[sender]
        sender_vc.increment(sender)

        send_event = Event(
            process_id=sender,
            name=f"send({message}) to {receiver}",
            vector_clock=VectorClock(dict(sender_vc.clock))
        )
        self.events.append(send_event)
        print(f"📤 {sender} → {receiver}: {message} {send_event.vector_clock}")

        # Получатель: мерджит vector clock и увеличивает свой
        receiver_vc = self.vector_clocks[receiver]
        receiver_vc.merge(sender_vc)
        receiver_vc.increment(receiver)

        recv_event = Event(
            process_id=receiver,
            name=f"recv({message}) from {sender}",
            vector_clock=VectorClock(dict(receiver_vc.clock)),
            dependencies=[send_event]
        )
        self.events.append(recv_event)
        print(f"📥 {receiver} ← {sender}: {message} {recv_event.vector_clock}")

        return send_event, recv_event

    def detect_order(self, event1: Event, event2: Event) -> str:
        """Определяем порядок двух событий"""
        if event1.vector_clock.happens_before(event2.vector_clock):
            return f"✓ {event1.process_id}:{event1.name} → {event2.process_id}:{event2.name}"
        elif event2.vector_clock.happens_before(event1.vector_clock):
            return f"✓ {event2.process_id}:{event2.name} → {event1.process_id}:{event1.name}"
        else:
            return f"↔ {event1.process_id}:{event1.name} ∥ {event2.process_id}:{event2.name} (concurrent)"


# Пример использования

def test_vector_clocks():
    print("=== VECTOR CLOCKS DEMONSTRATION ===\n")

    system = DistributedSystem(["Alice", "Bob", "Charlie"])

    # Alice делает локальное событие
    e_a1 = system.local_event("Alice", "E1")
    print()

    # Bob делает локальное событие (параллельно)
    e_b1 = system.local_event("Bob", "E1")
    print()

    # Alice отправляет Bob сообщение
    e_a2, e_b2 = system.send_message("Alice", "Bob", "Hello")
    print()

    # Bob отправляет Charlie
    e_b3, e_c1 = system.send_message("Bob", "Charlie", "Hi!")
    print()

    # Alice отправляет Charlie (независимо)
    e_a3, e_c2 = system.send_message("Alice", "Charlie", "Hey!")
    print()

    # Анализируем порядок
    print("\n=== CAUSALITY ANALYSIS ===\n")

    print(system.detect_order(e_a1, e_b2))
    print(system.detect_order(e_b1, e_a1))
    print(system.detect_order(e_b2, e_c1))
    print(system.detect_order(e_a3, e_c2))
    print(system.detect_order(e_b1, e_a3))
    print()

    # Пример использования для conflict detection
    print("=== CONFLICT DETECTION ===\n")

    system2 = DistributedSystem(["Replica_A", "Replica_B"])

    # Обе реплики одновременно обновляют один ключ
    print("Simultaneous writes (conflict):\n")

    write_a = system2.local_event("Replica_A", "write(key=value_a)")
    print()

    write_b = system2.local_event("Replica_B", "write(key=value_b)")
    print()

    # Обнаруживаем конфликт
    if write_a.vector_clock.concurrent(write_b.vector_clock):
        print("⚠️  CONFLICT DETECTED: Concurrent writes!")
        print(f"   Need to resolve: {write_a.vector_clock} vs {write_b.vector_clock}")

    print()
    print("Resolution strategies:")
    print("  1. Last-Write-Wins (compare timestamps)")
    print("  2. Vector clock based (application logic)")
    print("  3. CRDT (Conflict-free data structure)")


test_vector_clocks()
```

### Типичные ошибки
1. **Забывать про merge** - При получении сообщения нужно merge, не просто копировать
2. **Не учитывать что O(N) память** - Для больших систем может быть bottleneck
3. **Путать с Lamport timestamps** - LT проще но менее информативны
4. **Не отслеживать зависимости** - Vector clock - это только инструмент
5. **Думать что это решает все конфликты** - Помогает детектировать, но не разрешает

### На интервью
**Как отвечать:**
1. "Vector clock - счётчик каждого процесса для каждого другого процесса"
2. Нарисовать пример с двумя-тремя процессами
3. "Увеличиваем свой счётчик на каждое событие"
4. "Мерджим при получении сообщения"
5. "Используется в Dynamo, Cassandra для конфликтов"

**Возможные follow-up:**
- "Память O(N)?" → Да, может быть проблема для больших систем
- "Как с версионированием?" → Вместе с timestamps
- "Альтернативы?" → Lamport clocks (проще но менее информативны)

---

## 9. Как выбрать модель консистентности для системы?

### Зачем спрашивают
Это практический вопрос - нужно уметь обосновывать выбор для конкретной системы.

### Короткий ответ
Выбираем на основе требований: нужна ли **корректность** (платежи → linearizable), **скорость** (кэш → eventual), или **баланс** (чат → causal/session). Используем **decision tree** для каждого типа данных отдельно.

### Детальный разбор

**Decision Tree для выбора консистентности:**

```
┌─────────────────────────────────────────┐
│  Критично ли быть НЕПРАВИЛЬНЫМИ?        │
│  (платежи, банк, резервирование)        │
└────────────────┬────────────────────────┘
           ДА / НЕТ
          /        \
         /          \
        ДА           НЕТ
        │            │
        ▼            ├─ Потерпеть временное
    LINEARIZABLE      │  несоответствие?
    (1 лидер,         │
     consensus)       ДА / НЕТ
                     /        \
                    ДА        НЕТ
                    │          │
                    ▼          ▼
             SEQUENTIAL      EVENTUAL
             (или CAUSAL     CONSISTENCY
              для причин)    (кэши, DNS)
             (чаты)


Дополнительные вопросы:

1. Какой % операций - reads vs writes?
   HIGH READ (>90%):
     - Кэширование
     - Read-only replicas

   HIGH WRITE (>50% writes):
     - Quorum writes
     - Batch оптимизация

2. Сетевые разделения часто?
   HIGH:
     - Eventual consistency
     - Partition tolerance (BASE)

   LOW:
     - Strong consistency OK (ACID)

3. Нужна ли cause-effect?
   YES (чат, соцсеть):
     - Causal consistency
     - Vector clocks

   NO (кэш):
     - Eventual OK

4. Есть ли пользовательская сессия?
   YES (web app):
     - Session consistency
     - Read-your-writes

   NO (batch system):
     - Eventual OK
```

**Примеры разных систем и их выбора:**

```
FACEBOOK / SOCIAL NETWORK:
├─ Posts: Causal consistency (пост перед комментариями)
├─ Likes: Eventual consistency (счётчик может быть stale)
├─ Messages: Causal consistency (порядок важен)
└─ Feed: Session consistency (я вижу свои actions)

BANK / PAYMENT SYSTEM:
├─ Transfers: Linearizable (один лидер, consensus)
├─ Balances: Linearizable (must be accurate)
└─ History: Sequential or Linearizable

E-COMMERCE:
├─ Inventory: Linearizable (quorum or locking)
├─ Shopping Cart: Session consistency
├─ Recommendations: Eventual (stale OK)
└─ Reviews: Causal (older reviews before newer)

VIDEO STREAMING:
├─ Watch history: Eventual (fast write)
├─ Recommendations: Eventual (batch updated)
├─ Views count: Eventual (can be stale)
└─ Subtitles: Strong (must be consistent)

CHAT APPLICATION:
├─ Messages: Causal consistency (порядок критичен)
├─ Typing indicators: Eventual (не критично)
├─ Read receipts: Causal (связаны с message)
└─ User presence: Eventual (can be stale)
```

### Пример

```python
from dataclasses import dataclass
from enum import Enum

class ConsistencyModel(Enum):
    """Модели консистентности"""
    LINEARIZABLE = "Linearizable"
    SEQUENTIAL = "Sequential"
    CAUSAL = "Causal"
    SESSION = "Session"
    EVENTUAL = "Eventual"


@dataclass
class SystemRequirement:
    """Требование к системе"""
    name: str
    importance: str  # "critical" | "high" | "medium" | "low"
    description: str


@dataclass
class ConsistencyChoice:
    """Выбор консистентности с обоснованием"""
    model: ConsistencyModel
    reasoning: str
    tradeoffs: str
    implementation_difficulty: str  # "easy" | "medium" | "hard"


class ConsistencyChooser:
    """Помощник для выбора консистентности"""

    def choose_for_banking_system(self) -> ConsistencyChoice:
        """Банковская система"""
        return ConsistencyChoice(
            model=ConsistencyModel.LINEARIZABLE,
            reasoning=(
                "Платежи - критичны. Два платежа не должны видеть "
                "друг друга до завершения. Нужна атомарность с глобальным порядком."
            ),
            tradeoffs=(
                "- Медленнее (один лидер или consensus)"
                "- Может быть down при разделении сети"
                "- Но гарантирует корректность"
            ),
            implementation_difficulty="hard"
        )

    def choose_for_social_network(self) -> dict[str, ConsistencyChoice]:
        """Социальная сеть"""
        return {
            "posts": ConsistencyChoice(
                model=ConsistencyModel.CAUSAL,
                reasoning="Посты должны быть перед комментариями",
                tradeoffs="- Vector clocks память\n- Но гарантирует причинный порядок",
                implementation_difficulty="medium"
            ),
            "likes": ConsistencyChoice(
                model=ConsistencyModel.EVENTUAL,
                reasoning="Счётчик лайков может быть stale",
                tradeoffs="- Быстро\n- Но иногда видим старое число",
                implementation_difficulty="easy"
            ),
            "feed": ConsistencyChoice(
                model=ConsistencyModel.SESSION,
                reasoning="Я должен видеть свои actions",
                tradeoffs="- Session token\n- Я вижу свои, но другие - eventual",
                implementation_difficulty="medium"
            )
        }

    def choose_for_ecommerce(self) -> dict[str, ConsistencyChoice]:
        """E-commerce"""
        return {
            "inventory": ConsistencyChoice(
                model=ConsistencyModel.LINEARIZABLE,
                reasoning="Два покупателя не должны купить одно товар",
                tradeoffs="- Slower\n- Но no double-sell",
                implementation_difficulty="hard"
            ),
            "shopping_cart": ConsistencyChoice(
                model=ConsistencyModel.SESSION,
                reasoning="Я вижу мою корзину",
                tradeoffs="- Read-your-writes\n- Простая реализация",
                implementation_difficulty="easy"
            ),
            "product_reviews": ConsistencyChoice(
                model=ConsistencyModel.CAUSAL,
                reasoning="Ответ на review должен быть после самого review",
                tradeoffs="- Гарантируем порядок\n- Но может быть stale",
                implementation_difficulty="medium"
            ),
            "recommendations": ConsistencyChoice(
                model=ConsistencyModel.EVENTUAL,
                reasoning="Могут быть старыми, обновляем раз в час",
                tradeoffs="- Очень быстро\n- Batch update OK",
                implementation_difficulty="easy"
            )
        }

    def choose_for_chat(self) -> ConsistencyChoice:
        """Чат"""
        return ConsistencyChoice(
            model=ConsistencyModel.CAUSAL,
            reasoning=(
                "Сообщения должны быть в причинном порядке. "
                "Если я отвечаю на сообщение, все должны видеть "
                "исходное сообщение перед моим ответом."
            ),
            tradeoffs=(
                "- Vector clocks память O(N)\n"
                "- Немного медленнее\n"
                "- Но гарантируем логический порядок"
            ),
            implementation_difficulty="medium"
        )


# Пример использования

def demonstrate_choice():
    print("=== CONSISTENCY CHOICES FOR DIFFERENT SYSTEMS ===\n")

    chooser = ConsistencyChooser()

    # Banking
    print("1️⃣  BANKING SYSTEM:")
    choice = chooser.choose_for_banking_system()
    print(f"   Model: {choice.model.value}")
    print(f"   Reasoning: {choice.reasoning}")
    print(f"   Trade-offs:\n{choice.tradeoffs}")
    print(f"   Difficulty: {choice.implementation_difficulty}\n")

    # Social Network
    print("2️⃣  SOCIAL NETWORK:")
    for data_type, choice in chooser.choose_for_social_network().items():
        print(f"\n   {data_type.upper()}:")
        print(f"   - Model: {choice.model.value}")
        print(f"   - Reasoning: {choice.reasoning}")

    print("\n\n3️⃣  E-COMMERCE:")
    for data_type, choice in chooser.choose_for_ecommerce().items():
        print(f"\n   {data_type.upper()}:")
        print(f"   - Model: {choice.model.value}")

    print("\n\n4️⃣  CHAT APPLICATION:")
    choice = chooser.choose_for_chat()
    print(f"   Model: {choice.model.value}")
    print(f"   Reasoning: {choice.reasoning}")


demonstrate_choice()
```

**Таблица сравнения:**

```
┌──────────────────┬───────────┬──────────────┬────────────┬──────────────┐
│ Model            │ Latency   │ Availability │ Tolerance  │ Complexity   │
├──────────────────┼───────────┼──────────────┼────────────┼──────────────┤
│ Linearizable     │ HIGH (ms) │ LOW (leader) │ LOW        │ HARD         │
│ Sequential       │ MEDIUM    │ MEDIUM       │ MEDIUM     │ MEDIUM       │
│ Causal           │ MEDIUM    │ HIGH         │ MEDIUM     │ MEDIUM       │
│ Session          │ LOW       │ HIGH         │ MEDIUM     │ EASY-MEDIUM  │
│ Eventual         │ VERY LOW  │ VERY HIGH    │ VERY HIGH  │ EASY         │
└──────────────────┴───────────┴──────────────┴────────────┴──────────────┘
```

### Типичные ошибки
1. **Выбрать одну для всей системы** - Разные данные нужны разные уровни
2. **Переинвестировать в консистентность** - Eventual часто достаточно
3. **Недооценивать стоимость strong consistency** - Это дорого!
4. **Не проверять требования** - Нужно спросить у product

### На интервью
**Как отвечать:**
1. Первый вопрос: "Какие требования к корректности?"
2. Второй: "Как часто конфликты?"
3. Третий: "Сетевые разделения?"
4. Потом: "Для каждого типа данных выбираем отдельно"
5. Примеры:
   - Платежи → Linearizable
   - Чаты → Causal
   - Кэши → Eventual

**Возможные follow-up:**
- "Почему не linearizable везде?" → Стоимость, масштабируемость
- "Как тестировать?" → Chaos engineering, consistency checker
- "Как migrate?" → Постепенно, с абстракциями

---

## 10. Как гарантировать порядок сообщений в чате?

### Зачем спрашивают
Практический пример, где нужно применить causal consistency и другие техники.

### Короткий ответ
**Порядок в чате:** каждое сообщение имеет **sequence number** (монотонно растущий) или **vector clock** (для причинных зависимостей). Если сообщение пришло не по порядку, буферируем до прихода предыдущих. Используем **read repair** для выравнивания реплик.

### Детальный разбор

**Архитектура чата с гарантией порядка:**

```
┌────────────────────────────────────────────────────┐
│           ORDERED CHAT SYSTEM                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  CLIENT A    CLIENT B    CLIENT C                 │
│      │           │           │                    │
│      └───────────┼───────────┘                    │
│              │ Send                               │
│              ▼                                     │
│         ┌─────────────────────────────────┐      │
│         │  PRIMARY REPLICA (Sequencer)    │      │
│         │                                 │      │
│         │  Message Queue + Seq Numbers   │      │
│         │  ┌───────────────┐             │      │
│         │  │ M1 (seq=1)    │             │      │
│         │  │ M2 (seq=2)    │             │      │
│         │  │ M3 (seq=3)    │             │      │
│         │  └───────────────┘             │      │
│         └──────────────┬──────────────────┘      │
│                        │ Async replicate         │
│              ┌─────────┼─────────┐               │
│              ▼         ▼         ▼               │
│          REPLICA1  REPLICA2  REPLICA3           │
│                                                  │
│  Delivery to clients (in order):                │
│  1. Wait for seq=1                              │
│  2. Deliver M1 to all clients                   │
│  3. Wait for seq=2                              │
│  4. Deliver M2 to all clients                   │
│  5. Etc.                                        │
│                                                  │
└────────────────────────────────────────────────────┘


TWO APPROACHES:

APPROACH 1: Central Sequencer (strong order)
┌──────────────────┐
│  Primary Replica │ ← All writes go here
│  (sequencer)     │   seq_num++
└────┬─────────────┘
     │ async replicate
     ├─────────────────┬──────────────┐
     ▼                 ▼              ▼
  Replica1         Replica2      Replica3
  (read-only)      (read-only)    (read-only)

Pros: Simple, strong order
Cons: Primary is bottleneck, single point of failure


APPROACH 2: Distributed Sequencing (with quorum)
Client A          Client B
    │                 │
    ├─────────┬───────┤
    │         │       │
    ▼         ▼       ▼
Replica1   Replica2  Replica3
    │         │       │
    └────┬────┴───┬───┘
         │        │
    Quorum write:
    Write to 2/3 replicas with seq_num

    Quorum read:
    Read from 2/3 replicas
    Deliver in order of seq_num

Pros: No single bottleneck
Cons: More complex, possible gaps
```

**Гарантии для чата:**

```
1. ORDERED DELIVERY:
   Если Alice отправила "Hello" (M1) потом "World" (M2),
   ВСЕ видят M1 перед M2

2. CAUSAL DELIVERY:
   Если Bob отвечает на "Hello", все видят "Hello" перед ответом

3. RELIABILITY:
   Ни одно сообщение не потеряется

4. FAIRNESS:
   Ни один автор не будет "глушить" другого
   (не может быть что 1000 сообщений от A перед одним от B)
```

### Пример

```python
import time
import threading
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from collections import defaultdict

@dataclass
class ChatMessage:
    """Сообщение в чате"""
    message_id: str
    sender: str
    content: str
    sequence_number: int  # Глобальный порядок
    timestamp: float
    vector_clock: Dict[str, int] = field(default_factory=dict)
    # vector_clock отслеживает причинные зависимости


class ChatServer:
    """Сервер чата с гарантией порядка"""
    def __init__(self, server_id: str):
        self.server_id = server_id
        self.messages: Dict[int, ChatMessage] = {}  # seq_num -> message
        self.next_sequence = 1
        self.lock = threading.Lock()
        self.vector_clocks: Dict[str, Dict[str, int]] = defaultdict(
            lambda: defaultdict(int)
        )

    def send_message(self, sender: str, content: str,
                    client_vc: Dict[str, int]) -> ChatMessage:
        """Отправляем сообщение с гарантией порядка"""
        with self.lock:
            seq_num = self.next_sequence
            self.next_sequence += 1

        # Обновляем vector clock отправителя
        self.vector_clocks[sender][sender] += 1
        for other in client_vc:
            self.vector_clocks[sender][other] = max(
                self.vector_clocks[sender].get(other, 0),
                client_vc.get(other, 0)
            )

        msg = ChatMessage(
            message_id=f"{sender}_{seq_num}",
            sender=sender,
            content=content,
            sequence_number=seq_num,
            timestamp=time.time(),
            vector_clock=dict(self.vector_clocks[sender])
        )

        self.messages[seq_num] = msg

        print(f"📤 {self.server_id}: Sequence #{seq_num} from {sender}: {content}")

        return msg

    def get_messages_in_order(self, up_to_seq: int) -> List[ChatMessage]:
        """Возвращаем сообщения в порядке"""
        result = []
        for seq in sorted(self.messages.keys()):
            if seq <= up_to_seq:
                result.append(self.messages[seq])
        return result


class ChatReplica:
    """Реплика сервера чата"""
    def __init__(self, replica_id: str, is_primary: bool = False):
        self.replica_id = replica_id
        self.is_primary = is_primary
        self.messages: Dict[int, ChatMessage] = {}
        self.next_expected_seq = 1
        self.pending_delivery: List[ChatMessage] = []
        self.delivered: set = set()
        self.lock = threading.Lock()

    def receive_message(self, msg: ChatMessage) -> bool:
        """Получаем сообщение"""
        with self.lock:
            self.messages[msg.sequence_number] = msg
            return True

    def deliver_messages(self) -> List[ChatMessage]:
        """Доставляем сообщения в порядке"""
        delivered = []

        with self.lock:
            while self.next_expected_seq in self.messages:
                msg = self.messages[self.next_expected_seq]

                # Проверяем что все зависимости доставлены
                if self._can_deliver(msg):
                    delivered.append(msg)
                    self.delivered.add(self.next_expected_seq)
                    self.next_expected_seq += 1
                else:
                    break

        return delivered

    def _can_deliver(self, msg: ChatMessage) -> bool:
        """Можем ли доставить сообщение?

        Для простого случая: просто если seq_num = next_expected
        Для causal: проверяем vector clock зависимости
        """
        # Простой случай: just sequence order
        return msg.sequence_number == self.next_expected_seq


class DistributedChat:
    """Распределённый чат с гарантией порядка"""
    def __init__(self):
        self.primary = ChatReplica("Primary", is_primary=True)
        self.replicas = [
            ChatReplica("Replica-1"),
            ChatReplica("Replica-2"),
        ]
        self.all_replicas = [self.primary] + self.replicas

        self.client_vector_clocks: Dict[str, Dict[str, int]] = defaultdict(
            lambda: defaultdict(int)
        )

    def send_message(self, sender: str, content: str):
        """Клиент отправляет сообщение"""
        # Обновляем vector clock отправителя
        self.client_vector_clocks[sender][sender] += 1

        # Отправляем на primary
        msg = ChatMessage(
            message_id=f"{sender}_{self.primary.next_expected_seq}",
            sender=sender,
            content=content,
            sequence_number=self.primary.next_expected_seq,
            timestamp=time.time(),
            vector_clock=dict(self.client_vector_clocks[sender])
        )

        self.primary.next_expected_seq += 1
        self.primary.receive_message(msg)

        print(f"✍️  {sender}: {content}")

        # Асинхронно реплицируем на другие
        for replica in self.replicas:
            threading.Thread(
                target=self._async_replicate,
                args=(msg, replica),
                daemon=True
            ).start()

        return msg

    def _async_replicate(self, msg: ChatMessage, replica: ChatReplica):
        """Асинхронная репликация"""
        time.sleep(0.01)  # Имитируем сетевую задержку
        replica.receive_message(msg)

    def deliver_to_clients(self):
        """Доставляем сообщения клиентам в порядке"""
        delivered_messages = self.primary.deliver_messages()

        for msg in delivered_messages:
            print(f"📨 DELIVERED: #{msg.sequence_number} from {msg.sender}: {msg.content}")

            # Обновляем vector clock клиентов
            for client_id in [msg.sender]:
                self.client_vector_clocks[client_id][msg.sender] += 1

    def check_replicas_consistent(self):
        """Проверяем что все реплики имеют одинаковые сообщения"""
        primary_messages = sorted(self.primary.messages.keys())

        all_consistent = True
        for replica in self.replicas:
            replica_messages = sorted(replica.messages.keys())

            if primary_messages != replica_messages:
                print(f"⚠️  {replica.replica_id} out of sync")
                all_consistent = False
            else:
                print(f"✓ {replica.replica_id} in sync")

        return all_consistent


# Пример использования

def test_ordered_chat():
    print("=== ORDERED CHAT WITH SEQUENCE NUMBERS ===\n")

    chat = DistributedChat()

    # Alice отправляет сообщение
    print("1️⃣  Alice sends message:")
    chat.send_message("Alice", "Hello everyone!")
    print()

    # Ждём репликации и доставки
    time.sleep(0.05)
    chat.deliver_to_clients()
    print()

    # Bob отправляет сообщение
    print("2️⃣  Bob sends message:")
    chat.send_message("Bob", "Hi Alice!")
    print()

    # Ждём репликации и доставки
    time.sleep(0.05)
    chat.deliver_to_clients()
    print()

    # Alice отправляет ещё
    print("3️⃣  Alice sends second message:")
    chat.send_message("Alice", "How are you?")
    print()

    # Ждём репликации и доставки
    time.sleep(0.05)
    chat.deliver_to_clients()
    print()

    # Проверяем консистентность реплик
    print("\n=== REPLICA CONSISTENCY CHECK ===\n")
    chat.check_replicas_consistent()

    print(f"\n✓ All messages delivered in order!")


def test_concurrent_messages():
    """Тест: что происходит когда несколько клиентов отправляют одновременно?"""
    print("\n\n=== CONCURRENT MESSAGE SCENARIO ===\n")

    chat = DistributedChat()

    # Симулируем concurrent sends
    threads = []
    for i in range(3):
        sender = ["Alice", "Bob", "Charlie"][i]
        msg = f"Message {i+1}"

        t = threading.Thread(
            target=chat.send_message,
            args=(sender, msg),
            daemon=True
        )
        threads.append(t)
        t.start()

    # Ждём всех
    for t in threads:
        t.join()

    print("\n✍️  All messages received, replicating...")
    time.sleep(0.1)

    print("\n📨 Delivering in order...")
    chat.deliver_to_clients()

    # Messages будут доставлены в порядке sequence numbers,
    # не в порядке отправки!


test_ordered_chat()
test_concurrent_messages()
```

### Типичные ошибки
1. **Забыть про sequencing** - Без уникальных номеров невозможно гарантировать порядок
2. **Потеря сообщений при репликации** - Нужен acknowledgement или quorum
3. **Deadlock на primary** - Primary может стать bottleneck
4. **Не учитывать latency** - Может быть заметная задержка перед доставкой
5. **Конфликты при distributed sequencing** - Сложнее чем с primary

### На интервью
**Как отвечать:**
1. "Каждому сообщению даём уникальный sequence number"
2. "Primary replica выдаёт номера (или quorum)"
3. "Доставляем сообщения в порядке seq_num"
4. "Для причинных зависимостей - vector clocks"
5. "Асинхронная репликация, read repair для выравнивания"

**Возможные follow-up:**
- "Что если primary down?" → Failover, новый primary продолжает с max_seq+1
- "Duplicate messages?" → Deduplication by message_id
- "Out of order receives?" → Buffer until we have seq_num-1

---

## Таблица сравнения моделей консистентности

| Модель | Гарантии | Латентность | Доступность | Примеры | Сложность |
|--------|----------|-------------|------------|---------|-----------|
| **Linearizable** | Один глобальный порядок | ВЫСОКАЯ | НИЗКАЯ | Платежи, консенсус | ВЫСОКАЯ |
| **Sequential** | Есть какой-то последовательный порядок | СРЕДНЯЯ | СРЕДНЯЯ | Многопроцессорные системы | СРЕДНЯЯ |
| **Causal** | Причинно-связанные в порядке | СРЕДНЯЯ | ВЫСОКАЯ | Чаты, соцсети | СРЕДНЯЯ |
| **Session** | Видишь свои write'ы | НИЗКАЯ | ВЫСОКАЯ | Web приложения | НИЗКАЯ |
| **Eventual** | Рано или поздно сойдёмся | ОЧЕНЬ НИЗКАЯ | ОЧЕНЬ ВЫСОКАЯ | DNS, кэши, CDN | НИЗКАЯ |

---

## Decision Tree: Какую консистентность выбрать?

```
Начните здесь:
│
├─ Критична ТОЧНОСТЬ данных? (финансы, резервирование, платежи)
│  ├─ ДА → LINEARIZABLE
│  │       Инструменты: лидер, consensus (Raft/Paxos)
│  │       Цена: медленнее, меньше доступность
│  │
│  └─ НЕТ → Переходим к следующему вопросу
│
├─ Важен ПОРЯДОК событий? (чаты, соцсети, логи)
│  ├─ ДА → Нужен ПРИЧИННЫЙ порядок?
│  │      ├─ ДА (ответы на комментарии, threads) → CAUSAL
│  │      │   Инструменты: vector clocks, dependency tracking
│  │      │
│  │      └─ НЕТ (просто хронологический) → SEQUENTIAL или LINEARIZABLE
│  │
│  └─ НЕТ → Переходим к следующему вопросу
│
├─ Клиент в СЕССИИ? (веб приложение, мобильный app)
│  ├─ ДА → SESSION CONSISTENCY
│  │       Гарантируем: читаешь свои write'ы
│  │       Инструменты: session tokens, primary replica for session
│  │
│  └─ НЕТ → Переходим к следующему вопросу
│
└─ По умолчанию → EVENTUAL CONSISTENCY
   Когда используется: кэши, DNS, рекомендации, счётчики
   Инструменты: async replication, anti-entropy, conflict resolution
   Плюсы: быстро, масштабируемо, доступно
   Минусы: stale data, конфликты (но разрешаемы)
```

---

## Ключевые выводы

1. **Нет одной правильной модели** - выбираем на основе требований
2. **Trade-off между скоростью и корректностью** - ACID vs BASE
3. **Разные данные - разные модели** - не одна для всей системы
4. **Инструменты реализации:**
   - Лидер/консенсус → Linearizable
   - Vector clocks → Causal
   - Session tokens → Session
   - Async replication → Eventual
5. **Тестирование важно** - используйте chaos engineering

---

## Дополнительные ресурсы

**Классические бумаги:**
- Lamport "Time, Clocks, and the Ordering of Events"
- Vogels "Eventually Consistent"
- Bailis "Highly Available Transactions"

**Системы для изучения:**
- **Linearizable:** Raft, Zookeeper, etcd
- **Sequential:** Cache coherency protocols
- **Causal:** Dynamo, Cassandra (with vector clocks)
- **Session:** Most web frameworks
- **Eventual:** DNS, CDN, S3

