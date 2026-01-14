# 09. Coordination

[← Назад к списку тем](README.md)

---

## Coordination Services

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Coordination Services                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Provide primitives for distributed coordination:                   │
│                                                                     │
│  - Configuration management                                         │
│  - Service discovery                                                │
│  - Leader election                                                  │
│  - Distributed locking                                              │
│  - Barrier synchronization                                          │
│                                                                     │
│  Examples:                                                          │
│  ───────────                                                        │
│  - ZooKeeper (Apache)                                               │
│  - etcd (CoreOS/CNCF)                                               │
│  - Consul (HashiCorp)                                               │
│                                                                     │
│  All use consensus (Paxos/Raft/ZAB) internally                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ZooKeeper

### Data Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ZooKeeper Data Model                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Hierarchical namespace (like file system)                          │
│                                                                     │
│  /                                                                  │
│  ├── /app                                                           │
│  │   ├── /app/config                                                │
│  │   │   └── data: {"db_host": "..."}                               │
│  │   ├── /app/leader                                                │
│  │   │   └── data: "node-1"                                         │
│  │   └── /app/workers                                               │
│  │       ├── /app/workers/node-1 (ephemeral)                        │
│  │       └── /app/workers/node-2 (ephemeral)                        │
│  └── /locks                                                         │
│      └── /locks/resource-1                                          │
│          └── /locks/resource-1/lock-0000000001 (ephemeral, seq)     │
│                                                                     │
│  Node types:                                                        │
│  - Persistent: survives client disconnect                           │
│  - Ephemeral: deleted when client session ends                      │
│  - Sequential: auto-incrementing suffix                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Watches

```go
// ZooKeeper watches - one-time notifications
func watchConfig(zk *zk.Conn) {
    for {
        data, _, eventCh, err := zk.GetW("/app/config")
        if err != nil {
            log.Fatal(err)
        }

        // Use the config
        applyConfig(data)

        // Wait for change
        event := <-eventCh
        log.Printf("Config changed: %v", event.Type)
        // Loop to re-read and set new watch
    }
}
```

---

## etcd

### Key-Value API

```go
import clientv3 "go.etcd.io/etcd/client/v3"

func main() {
    cli, _ := clientv3.New(clientv3.Config{
        Endpoints:   []string{"localhost:2379"},
        DialTimeout: 5 * time.Second,
    })
    defer cli.Close()

    ctx := context.Background()

    // Put
    cli.Put(ctx, "/app/config", `{"db_host": "localhost"}`)

    // Get
    resp, _ := cli.Get(ctx, "/app/config")
    for _, kv := range resp.Kvs {
        fmt.Printf("%s: %s\n", kv.Key, kv.Value)
    }

    // Get with prefix
    resp, _ = cli.Get(ctx, "/app/", clientv3.WithPrefix())

    // Watch
    watchChan := cli.Watch(ctx, "/app/config")
    for watchResp := range watchChan {
        for _, event := range watchResp.Events {
            fmt.Printf("Type: %s, Key: %s\n", event.Type, event.Kv.Key)
        }
    }
}
```

### Lease (TTL)

```go
// Create lease with TTL
lease, _ := cli.Grant(ctx, 10)  // 10 seconds

// Put with lease
cli.Put(ctx, "/app/workers/node-1", "alive", clientv3.WithLease(lease.ID))

// Keep alive (heartbeat)
keepAliveCh, _ := cli.KeepAlive(ctx, lease.ID)
go func() {
    for range keepAliveCh {
        // Lease renewed
    }
}()

// When client disconnects or stops heartbeat, key expires
```

---

## Distributed Lock

### Using etcd

```go
import "go.etcd.io/etcd/client/v3/concurrency"

func distributedLock(cli *clientv3.Client) error {
    // Create session (tied to lease)
    session, err := concurrency.NewSession(cli, concurrency.WithTTL(10))
    if err != nil {
        return err
    }
    defer session.Close()

    // Create mutex
    mutex := concurrency.NewMutex(session, "/locks/my-resource")

    // Acquire lock
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := mutex.Lock(ctx); err != nil {
        return err
    }

    // Critical section
    doWork()

    // Release lock
    return mutex.Unlock(context.Background())
}
```

### Using Redis (Redlock)

```go
func acquireLock(rdb *redis.Client, key string, ttl time.Duration) (string, bool) {
    token := uuid.New().String()

    // SET key token NX PX ttl
    ok, err := rdb.SetNX(ctx, key, token, ttl).Result()
    if err != nil || !ok {
        return "", false
    }

    return token, true
}

func releaseLock(rdb *redis.Client, key, token string) bool {
    // Lua script: delete only if token matches
    script := `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `
    result, _ := rdb.Eval(ctx, script, []string{key}, token).Int()
    return result == 1
}

// Redlock: acquire lock on majority of N independent Redis instances
// For fault tolerance
```

---

## Leader Election

### Using etcd

```go
func leaderElection(cli *clientv3.Client, nodeID string) {
    session, _ := concurrency.NewSession(cli, concurrency.WithTTL(10))
    defer session.Close()

    election := concurrency.NewElection(session, "/app/leader")

    // Campaign to become leader
    ctx := context.Background()
    if err := election.Campaign(ctx, nodeID); err != nil {
        log.Fatal(err)
    }

    // We are the leader!
    log.Printf("I am the leader: %s", nodeID)

    // Do leader work
    doLeaderWork()

    // Resign (optional, or session expires)
    election.Resign(ctx)
}

func watchLeader(cli *clientv3.Client) {
    election := concurrency.NewElection(cli.Session, "/app/leader")

    // Watch for leader changes
    for resp := range election.Observe(context.Background()) {
        leader := string(resp.Kvs[0].Value)
        log.Printf("Current leader: %s", leader)
    }
}
```

### Using ZooKeeper (Sequential Ephemeral)

```
┌─────────────────────────────────────────────────────────────────────┐
│              Leader Election with ZooKeeper                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Each node creates ephemeral sequential node under /election     │
│     /election/node-0000000001  (Node A)                             │
│     /election/node-0000000002  (Node B)                             │
│     /election/node-0000000003  (Node C)                             │
│                                                                     │
│  2. Node with smallest sequence number is leader                    │
│     Node A is leader (0000000001)                                   │
│                                                                     │
│  3. Others watch the node with next smaller number                  │
│     Node B watches 0000000001                                       │
│     Node C watches 0000000002                                       │
│                                                                     │
│  4. When leader fails:                                              │
│     - Ephemeral node 0000000001 deleted                             │
│     - Node B notified, checks if now smallest                       │
│     - Node B becomes leader                                         │
│                                                                     │
│  Avoids herd effect: only one node notified per failure             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Service Discovery

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Service Discovery                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Service instances register themselves                              │
│  Clients discover available instances                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │              Service Registry (etcd/Consul)              │        │
│  │  /services/user-service/instances/                       │        │
│  │    node-1: {"host": "10.0.0.1", "port": 8080}           │        │
│  │    node-2: {"host": "10.0.0.2", "port": 8080}           │        │
│  └─────────────────────────────────────────────────────────┘        │
│       ▲              ▲              │                               │
│       │ Register     │ Register     │ Discover                      │
│       │              │              ▼                               │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                          │
│  │ User    │    │ User    │    │ API     │                          │
│  │ Svc 1   │    │ Svc 2   │    │ Gateway │                          │
│  └─────────┘    └─────────┘    └─────────┘                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
// Service registration with etcd
func registerService(cli *clientv3.Client, serviceName, instanceID string, addr string) error {
    // Create lease
    lease, _ := cli.Grant(ctx, 10)

    key := fmt.Sprintf("/services/%s/instances/%s", serviceName, instanceID)
    value, _ := json.Marshal(ServiceInstance{
        Host: addr,
        Port: 8080,
    })

    // Register with lease
    cli.Put(ctx, key, string(value), clientv3.WithLease(lease.ID))

    // Keep alive
    cli.KeepAlive(ctx, lease.ID)

    return nil
}

// Service discovery
func discoverService(cli *clientv3.Client, serviceName string) ([]ServiceInstance, error) {
    prefix := fmt.Sprintf("/services/%s/instances/", serviceName)
    resp, err := cli.Get(ctx, prefix, clientv3.WithPrefix())
    if err != nil {
        return nil, err
    }

    var instances []ServiceInstance
    for _, kv := range resp.Kvs {
        var instance ServiceInstance
        json.Unmarshal(kv.Value, &instance)
        instances = append(instances, instance)
    }

    return instances, nil
}
```

---

## Consul

```go
import "github.com/hashicorp/consul/api"

// Service registration
func registerWithConsul() {
    client, _ := api.NewClient(api.DefaultConfig())

    registration := &api.AgentServiceRegistration{
        ID:      "user-service-1",
        Name:    "user-service",
        Port:    8080,
        Address: "10.0.0.1",
        Check: &api.AgentServiceCheck{
            HTTP:     "http://10.0.0.1:8080/health",
            Interval: "10s",
            Timeout:  "3s",
        },
    }

    client.Agent().ServiceRegister(registration)
}

// Service discovery
func discoverFromConsul(serviceName string) ([]*api.ServiceEntry, error) {
    client, _ := api.NewClient(api.DefaultConfig())

    services, _, err := client.Health().Service(serviceName, "", true, nil)
    return services, err
}

// KV Store
func consulKV() {
    client, _ := api.NewClient(api.DefaultConfig())
    kv := client.KV()

    // Put
    kv.Put(&api.KVPair{Key: "config/db_host", Value: []byte("localhost")}, nil)

    // Get
    pair, _, _ := kv.Get("config/db_host", nil)
    fmt.Println(string(pair.Value))
}
```

---

## Comparison

```
┌─────────────────┬──────────────────┬─────────────────┬────────────────┐
│                 │    ZooKeeper     │      etcd       │     Consul     │
├─────────────────┼──────────────────┼─────────────────┼────────────────┤
│ Consensus       │ ZAB              │ Raft            │ Raft           │
│ Data Model      │ Hierarchical     │ Flat KV         │ Flat KV        │
│ Language        │ Java             │ Go              │ Go             │
│ Watches         │ One-time         │ Streaming       │ Blocking query │
│ Service Disc.   │ Manual           │ Manual          │ Built-in       │
│ Health Checks   │ Sessions         │ Leases          │ Built-in       │
│ ACL             │ Yes              │ Yes             │ Yes            │
└─────────────────┴──────────────────┴─────────────────┴────────────────┘
```

---

## См. также

- [Failure Detection](./08-failure-detection.md) — обнаружение сбоев и протоколы heartbeat
- [Consensus](./04-consensus.md) — алгоритмы консенсуса, лежащие в основе сервисов координации

---

## На интервью

### Типичные вопросы

1. **ZooKeeper vs etcd vs Consul?**
   - ZK: mature, Java, hierarchical
   - etcd: simpler, Go, Kubernetes native
   - Consul: service discovery built-in

2. **Как реализовать distributed lock?**
   - etcd: lease + concurrency package
   - Redis: SET NX PX + Lua for release
   - ZK: ephemeral sequential nodes

3. **Leader election — как работает?**
   - Create ephemeral nodes
   - Smallest sequence is leader
   - Watch predecessor (avoids herd)

4. **Service discovery patterns?**
   - Client-side: client queries registry
   - Server-side: load balancer queries
   - DNS-based: Consul DNS

5. **Ephemeral nodes — зачем?**
   - Auto-cleanup on disconnect
   - Failure detection
   - Leader election, service registration

---

[← Назад к списку тем](README.md)
