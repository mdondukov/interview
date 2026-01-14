# 05. Replication

[← Назад к списку тем](README.md)

---

## Зачем нужна репликация

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Replication Goals                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. High Availability (HA)                                          │
│     - Failover при падении primary                                  │
│     - Минимальный downtime                                          │
│                                                                     │
│  2. Read Scaling                                                    │
│     - Распределение read нагрузки                                   │
│     - Реплики обрабатывают SELECT                                   │
│                                                                     │
│  3. Geographic Distribution                                         │
│     - Реплика ближе к пользователям                                 │
│     - Меньше latency для reads                                      │
│                                                                     │
│  4. Disaster Recovery                                               │
│     - Копия в другом датацентре                                     │
│     - Защита от catastrophic failure                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Leader-Follower (Primary-Replica)

### Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Leader-Follower Replication                       │
│                                                                     │
│                         ┌────────────┐                              │
│      Write ───────────▶ │  Primary   │                              │
│                         │  (Leader)  │                              │
│                         └─────┬──────┘                              │
│                               │                                     │
│                               │ WAL / Binlog                        │
│               ┌───────────────┼───────────────┐                     │
│               │               │               │                     │
│               ▼               ▼               ▼                     │
│        ┌──────────┐    ┌──────────┐    ┌──────────┐                 │
│        │ Replica 1│    │ Replica 2│    │ Replica 3│                 │
│        │(Follower)│    │(Follower)│    │(Follower)│                 │
│        └────┬─────┘    └────┬─────┘    └────┬─────┘                 │
│             │               │               │                       │
│    Read ◀───┴───────────────┴───────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Synchronous vs Asynchronous

```
┌─────────────────────────────────────────────────────────────────────┐
│              Synchronous vs Asynchronous Replication                 │
├────────────────────────────────┬────────────────────────────────────┤
│          Synchronous           │           Asynchronous             │
├────────────────────────────────┼────────────────────────────────────┤
│ Primary ждёт ACK от реплики    │ Primary НЕ ждёт                    │
│ Гарантия: данные на реплике    │ Возможна потеря данных             │
│ Выше latency write             │ Ниже latency write                 │
│ Если реплика down — блокировка │ Реплика может отстать              │
│                                │ Replication lag                    │
└────────────────────────────────┴────────────────────────────────────┘

Типичная конфигурация:
- 1 synchronous replica (для durability)
- N asynchronous replicas (для read scaling)
```

### PostgreSQL Streaming Replication

```sql
-- На Primary: postgresql.conf
wal_level = replica
max_wal_senders = 10
synchronous_commit = on  -- или 'remote_apply'

-- На Primary: pg_hba.conf
host replication replicator 10.0.0.0/8 md5

-- На Replica
pg_basebackup -h primary -D /var/lib/postgresql/data -U replicator -P

-- postgresql.conf на Replica
primary_conninfo = 'host=primary user=replicator password=...'
```

### Проверка состояния

```sql
-- На Primary: статус репликации
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) as lag_bytes
FROM pg_stat_replication;

-- На Replica: отставание
SELECT
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp(),
    NOW() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

## Multi-Leader (Master-Master)

### Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Multi-Leader Replication                          │
│                                                                     │
│        Datacenter A              Datacenter B                       │
│    ┌───────────────────┐    ┌───────────────────┐                   │
│    │                   │    │                   │                   │
│    │   ┌──────────┐    │    │   ┌──────────┐    │                   │
│    │   │ Leader A │◀───┼────┼──▶│ Leader B │    │                   │
│    │   └────┬─────┘    │    │   └────┬─────┘    │                   │
│    │        │          │    │        │          │                   │
│    │   ┌────▼─────┐    │    │   ┌────▼─────┐    │                   │
│    │   │ Follower │    │    │   │ Follower │    │                   │
│    │   └──────────┘    │    │   └──────────┘    │                   │
│    │                   │    │                   │                   │
│    └───────────────────┘    └───────────────────┘                   │
│                                                                     │
│    Writes могут идти на любой Leader                                │
│    ⚠️ Возможны конфликты!                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Conflict Resolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Conflict Resolution Strategies                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Last Write Wins (LWW)                                           │
│     - Используется timestamp                                        │
│     - Проблема: может потерять данные                               │
│                                                                     │
│  2. Merge                                                           │
│     - Автоматическое слияние (для некоторых типов данных)           │
│     - Пример: CRDT (Conflict-free Replicated Data Types)            │
│                                                                     │
│  3. Custom Resolution                                               │
│     - Приложение решает конфликт                                    │
│     - Сохраняем обе версии, показываем пользователю                 │
│                                                                     │
│  4. Avoid Conflicts                                                 │
│     - Шардирование по user/region                                   │
│     - Каждый user пишет в свой leader                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Пример конфликта

```sql
-- Leader A (timestamp T1):
UPDATE users SET name = 'Alice' WHERE id = 1;

-- Leader B (timestamp T2, T2 > T1):
UPDATE users SET name = 'Bob' WHERE id = 1;

-- LWW: выиграет 'Bob' (более поздний timestamp)
-- Но что если T1 и T2 разошлись из-за clock skew?

-- Решение: используй vector clocks или Lamport timestamps
```

---

## Leaderless Replication

### Dynamo-style

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Leaderless Replication                           │
│                                                                     │
│             Write to N nodes, succeed with W acks                   │
│             Read from N nodes, succeed with R responses             │
│                                                                     │
│    Quorum: W + R > N                                                │
│                                                                     │
│    Пример: N=3, W=2, R=2                                            │
│                                                                     │
│      Client                                                         │
│        │                                                            │
│        │ Write(x=5)                                                 │
│        ├────────────────▶ Node 1 ✓                                  │
│        ├────────────────▶ Node 2 ✓                                  │
│        └────────────────▶ Node 3 ✗ (временно недоступен)            │
│                                                                     │
│        W=2 достигнут, write успешен                                 │
│                                                                     │
│      Client                                                         │
│        │ Read(x)                                                    │
│        ├────────────────▶ Node 1 → x=5                              │
│        ├────────────────▶ Node 2 → x=5                              │
│        └────────────────▶ Node 3 → x=старое значение                │
│                                                                     │
│        R=2 достигнут, возвращаем x=5 (последняя версия)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Quorum Configurations

```
N=3 (total nodes)

Strong consistency: W=2, R=2 (W+R > N)
- Всегда читаем последнюю запись

High availability writes: W=1, R=3
- Быстрые записи, но нужно читать со всех

High availability reads: W=3, R=1
- Быстрые чтения, но медленные записи

Sloppy quorum:
- При недоступности некоторых узлов
- Пишем на "заместителей"
- Hinted handoff: передаём данные когда узел вернётся
```

### Read Repair и Anti-Entropy

```
Read Repair:
- При чтении обнаруживаем устаревшие данные
- Исправляем их на лету

Anti-Entropy (Merkle Trees):
- Фоновый процесс
- Сравниваем hash деревья между узлами
- Синхронизируем расхождения
```

---

## Failover

### Automatic Failover

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Automatic Failover                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Detection                                                       │
│     - Heartbeat timeout                                             │
│     - Health checks                                                 │
│                                                                     │
│  2. Election                                                        │
│     - Выбор новой primary из реплик                                 │
│     - Самая "свежая" (наименьший lag)                               │
│                                                                     │
│  3. Promotion                                                       │
│     - Реплика становится primary                                    │
│     - Начинает принимать writes                                     │
│                                                                     │
│  4. Reconfiguration                                                 │
│     - Другие реплики переключаются на нового primary                │
│     - DNS/Load balancer обновляется                                 │
│                                                                     │
│  Проблемы:                                                          │
│  - Split brain (два primary)                                        │
│  - Потеря данных (async replication lag)                            │
│  - Thundering herd при восстановлении                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### PostgreSQL с Patroni

```yaml
# Patroni config
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    maximum_lag_on_failover: 1048576  # 1MB
    synchronous_mode: true

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /var/lib/postgresql/data
```

---

## Replication Lag

### Проблемы

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Replication Lag Issues                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Read-after-write inconsistency:                                    │
│  ──────────────────────────────                                     │
│  User writes → Primary                                              │
│  User reads ← Replica (ещё не получила update)                      │
│  User видит старые данные!                                          │
│                                                                     │
│  Monotonic reads violation:                                         │
│  ─────────────────────────                                          │
│  User читает с Replica A → видит новые данные                       │
│  User читает с Replica B → видит старые данные                      │
│  "Время пошло назад!"                                               │
│                                                                     │
│  Causality violation:                                               │
│  ───────────────────                                                │
│  User A пишет пост                                                  │
│  User B видит пост, пишет комментарий                               │
│  User C видит комментарий, но не видит пост!                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Решения

```go
// 1. Read-your-writes: читай со своей реплики или с primary
func getUser(ctx context.Context, userID string) (*User, error) {
    session := getSession(ctx)

    if session.LastWriteTime.After(time.Now().Add(-5 * time.Second)) {
        // Недавно писали — читаем с primary
        return primaryDB.GetUser(ctx, userID)
    }
    // Давно не писали — можно с реплики
    return replicaDB.GetUser(ctx, userID)
}

// 2. Sticky sessions: всегда читай с одной реплики
func getReplicaForUser(userID string) *sql.DB {
    hash := fnv.New32()
    hash.Write([]byte(userID))
    replicaIndex := hash.Sum32() % uint32(len(replicas))
    return replicas[replicaIndex]
}

// 3. Causal consistency: передавай version/timestamp
type Response struct {
    Data    interface{} `json:"data"`
    Version int64       `json:"version"`
}

func handler(w http.ResponseWriter, r *http.Request) {
    minVersion, _ := strconv.ParseInt(r.Header.Get("X-Min-Version"), 10, 64)

    // Жди пока реплика догонит
    waitForVersion(minVersion)

    // ...
}
```

---

## На интервью

### Типичные вопросы

1. **Sync vs Async replication?**
   - Sync: гарантия данных, выше latency
   - Async: ниже latency, возможна потеря

2. **Что такое replication lag и как с ним бороться?**
   - Задержка между primary и replica
   - Read-your-writes, sticky sessions, causal tokens

3. **Multi-leader — когда использовать?**
   - Географически распределённые системы
   - Offline-first приложения
   - Нужна стратегия conflict resolution

4. **Quorum в leaderless?**
   - W + R > N для strong consistency
   - Trade-off между доступностью и консистентностью

5. **Как происходит failover?**
   - Detection → Election → Promotion → Reconfiguration
   - Проблемы: split brain, data loss

---

[← Назад к списку тем](README.md)
