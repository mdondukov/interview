# 04. Consensus

[← Назад к списку тем](README.md)

---

## Что такое Consensus

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Consensus Problem                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Несколько узлов должны договориться о значении.                    │
│                                                                     │
│  Properties:                                                        │
│  ───────────                                                        │
│  1. Agreement: все узлы решают одно значение                        │
│  2. Validity: решённое значение было предложено кем-то              │
│  3. Termination: все корректные узлы eventually решают              │
│                                                                     │
│  Use cases:                                                         │
│  ───────────                                                        │
│  - Leader election                                                  │
│  - Distributed locking                                              │
│  - Atomic commit (2PC/3PC)                                          │
│  - Replicated state machines                                        │
│  - Blockchain                                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## FLP Impossibility

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FLP Impossibility Theorem                          │
│                 (Fischer, Lynch, Paterson, 1985)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  В асинхронной системе с возможностью хотя бы одного crash failure  │
│  невозможно гарантировать consensus с termination.                  │
│                                                                     │
│  Почему?                                                            │
│  - Нельзя отличить медленный узел от упавшего                       │
│  - Любой алгоритм может застрять в ожидании                         │
│                                                                     │
│  Практические обходы:                                               │
│  - Randomization (probabilistic termination)                        │
│  - Timeouts (partial synchrony assumption)                          │
│  - Failure detectors (орakle for crashes)                           │
│                                                                     │
│  Реальные системы используют timeouts:                              │
│  "Eventually synchronous" модель                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Paxos (Overview)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Paxos Roles                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Proposer: предлагает значения                                      │
│  Acceptor: голосует за значения (quorum needed)                     │
│  Learner: узнаёт выбранное значение                                 │
│                                                                     │
│  (Часто один узел играет все роли)                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Basic Paxos (Single-Value)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Basic Paxos Protocol                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Phase 1a: Prepare                                                  │
│  Proposer → Acceptors: "Prepare(n)"                                 │
│  n = unique proposal number (higher = newer)                        │
│                                                                     │
│  Phase 1b: Promise                                                  │
│  Acceptor → Proposer:                                               │
│  If n > highest seen:                                               │
│    Promise not to accept < n                                        │
│    Return any previously accepted (n', v')                          │
│                                                                     │
│  Phase 2a: Accept                                                   │
│  If Proposer got majority promises:                                 │
│    If any accepted value returned: use that value                   │
│    Else: use own proposed value                                     │
│    Proposer → Acceptors: "Accept(n, v)"                             │
│                                                                     │
│  Phase 2b: Accepted                                                 │
│  Acceptor → Learners:                                               │
│  If n >= highest promised: accept(n, v)                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```
Example execution:

Proposer P1           Acceptor A1    Acceptor A2    Acceptor A3
    │                      │              │              │
    │ Prepare(1)           │              │              │
    ├─────────────────────▶├──────────────┼──────────────┤
    │                      │              │              │
    │ Promise(1,null)      │              │              │
    │◀─────────────────────┤              │              │
    │◀────────────────────────────────────┤              │
    │                      │              │              │
    │ Accept(1,"hello")    │              │              │
    ├─────────────────────▶├──────────────┼──────────────┤
    │                      │              │              │
    │ Accepted(1,"hello")  │              │              │
    │◀─────────────────────┤              │              │
    │◀────────────────────────────────────┤              │
    │                      │              │              │

Value "hello" is chosen (majority accepted)
```

---

## Raft

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Raft Overview                                │
│                    (Ongaro, Ousterhout, 2014)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Designed for understandability (vs Paxos)                          │
│  Decomposed into sub-problems:                                      │
│                                                                     │
│  1. Leader Election                                                 │
│  2. Log Replication                                                 │
│  3. Safety                                                          │
│                                                                     │
│  Server states:                                                     │
│  - Follower: passive, responds to Leader/Candidate                  │
│  - Candidate: trying to become Leader                               │
│  - Leader: handles all client requests                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Leader Election

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Raft Leader Election                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Terms: logical time periods with at most 1 leader                  │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ Term 1         │ Term 2      │ Term 3      │ Term 4 ...    │     │
│  │ Leader: S1     │ (no leader) │ Leader: S3  │ Leader: S2    │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Election process:                                                  │
│  1. Follower times out (no heartbeat from Leader)                   │
│  2. Becomes Candidate, increments term, votes for self              │
│  3. Sends RequestVote to all servers                                │
│  4. If gets majority votes → becomes Leader                         │
│  5. If receives heartbeat from valid Leader → becomes Follower      │
│  6. If election timeout → start new election                        │
│                                                                     │
│  Randomized timeout: 150-300ms (prevents split votes)               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type RaftState int

const (
    Follower RaftState = iota
    Candidate
    Leader
)

type RaftNode struct {
    state       RaftState
    currentTerm int64
    votedFor    string
    log         []LogEntry

    // Volatile state
    commitIndex int64
    lastApplied int64

    // Leader state
    nextIndex  map[string]int64
    matchIndex map[string]int64
}

func (n *RaftNode) startElection() {
    n.state = Candidate
    n.currentTerm++
    n.votedFor = n.id

    votes := 1  // vote for self

    for _, peer := range n.peers {
        go func(peer string) {
            resp := n.sendRequestVote(peer, RequestVoteArgs{
                Term:         n.currentTerm,
                CandidateID:  n.id,
                LastLogIndex: n.lastLogIndex(),
                LastLogTerm:  n.lastLogTerm(),
            })

            if resp.VoteGranted {
                votes++
                if votes > len(n.peers)/2 {
                    n.becomeLeader()
                }
            }
        }(peer)
    }
}
```

### Log Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Raft Log Replication                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Leader receives client request → appends to log → replicates       │
│                                                                     │
│  Leader Log:                                                        │
│  ┌─────┬─────┬─────┬─────┬─────┐                                    │
│  │ 1:x │ 1:y │ 2:z │ 3:a │ 3:b │  (term:command)                    │
│  └─────┴─────┴─────┴─────┴─────┘                                    │
│    1     2     3     4     5     (index)                            │
│                      ▲                                              │
│                 commitIndex                                         │
│                                                                     │
│  Follower receives AppendEntries:                                   │
│  - Check prevLogIndex/prevLogTerm match                             │
│  - If conflict: delete conflicting entries                          │
│  - Append new entries                                               │
│  - Update commitIndex                                               │
│                                                                     │
│  Entry committed when replicated on majority                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```go
type AppendEntriesArgs struct {
    Term         int64
    LeaderID     string
    PrevLogIndex int64
    PrevLogTerm  int64
    Entries      []LogEntry
    LeaderCommit int64
}

func (n *RaftNode) AppendEntries(args AppendEntriesArgs) AppendEntriesReply {
    if args.Term < n.currentTerm {
        return AppendEntriesReply{Term: n.currentTerm, Success: false}
    }

    // Check log consistency
    if args.PrevLogIndex > 0 {
        if n.log[args.PrevLogIndex].Term != args.PrevLogTerm {
            // Log inconsistency - need to go back
            return AppendEntriesReply{Success: false}
        }
    }

    // Append new entries (overwrite conflicting)
    for i, entry := range args.Entries {
        index := args.PrevLogIndex + 1 + int64(i)
        if index < int64(len(n.log)) && n.log[index].Term != entry.Term {
            n.log = n.log[:index]  // Delete conflicting
        }
        if index >= int64(len(n.log)) {
            n.log = append(n.log, entry)
        }
    }

    // Update commit index
    if args.LeaderCommit > n.commitIndex {
        n.commitIndex = min(args.LeaderCommit, int64(len(n.log)-1))
    }

    return AppendEntriesReply{Term: n.currentTerm, Success: true}
}
```

### Safety

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Raft Safety                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Election Restriction:                                              │
│  Candidate's log must be at least as up-to-date as majority         │
│                                                                     │
│  Voter grants vote only if:                                         │
│  - Candidate's last log term > voter's last log term                │
│  - OR terms equal AND candidate's log >= voter's log length         │
│                                                                     │
│  This ensures:                                                      │
│  - Leader has all committed entries                                 │
│  - No need to transfer entries to new leader                        │
│                                                                     │
│  Leader Completeness:                                               │
│  If entry committed in term T, present in all leaders of terms > T  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Paxos vs Raft

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│                      │        Paxos         │        Raft          │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Understandability    │ Complex              │ Simpler              │
│ Leader               │ Optional             │ Strong leader        │
│ Log replication      │ Multi-Paxos          │ Built-in             │
│ Membership changes   │ Complex              │ Joint consensus      │
│ Implementations      │ Many variations      │ More standardized    │
│ Used by              │ Chubby, Spanner      │ etcd, Consul, Kafka  │
└──────────────────────┴──────────────────────┴──────────────────────┘
```

---

## Реальные системы

```
etcd:
- Raft implementation
- Kubernetes uses for cluster state

ZooKeeper:
- ZAB (Zookeeper Atomic Broadcast)
- Similar to Paxos

Consul:
- Raft for consensus
- Service discovery, KV store

CockroachDB:
- Raft for each range
- Distributed SQL
```

---

## На интервью

### Типичные вопросы

1. **Что такое consensus и зачем?**
   - Договориться о значении
   - Leader election, locks, replicated state

2. **FLP impossibility?**
   - Нельзя гарантировать termination в async системе
   - Обходят через timeouts

3. **Paxos vs Raft?**
   - Raft проще понять
   - Strong leader, понятный log replication
   - Paxos более гибкий

4. **Raft leader election?**
   - Terms, randomized timeouts
   - Majority votes
   - Log up-to-date restriction

5. **Как Raft гарантирует safety?**
   - Election restriction (log completeness)
   - Leader has all committed entries
   - Commit only when majority replicated

---

[← Назад к списку тем](README.md)
