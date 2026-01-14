# 04. Concurrency Basics

[← Назад к списку тем](README.md)

---

## Thread Fundamentals

### Creating Threads

```java
// Way 1: Extend Thread
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

MyThread t = new MyThread();
t.start();  // Запускает новый поток

// Way 2: Implement Runnable (preferred)
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

Thread t = new Thread(new MyRunnable());
t.start();

// Way 3: Lambda (Java 8+)
Thread t = new Thread(() -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
});
t.start();

// Way 4: Callable (returns result)
Callable<Integer> task = () -> {
    return 42;
};
// Used with ExecutorService
```

### Thread Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Thread States                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│    NEW ──start()──▶ RUNNABLE ◀──────────────────────────┐           │
│                        │                                │           │
│                        │                                │           │
│           ┌────────────┼────────────┐                   │           │
│           │            │            │                   │           │
│           ▼            ▼            ▼                   │           │
│      BLOCKED      WAITING    TIMED_WAITING             │           │
│    (lock wait)  (wait/join)  (sleep/wait+timeout)      │           │
│           │            │            │                   │           │
│           └────────────┼────────────┘                   │           │
│                        │                                │           │
│                        └────────────────────────────────┘           │
│                        │                                            │
│                        ▼                                            │
│                   TERMINATED                                        │
│                                                                     │
│  Methods:                                                           │
│  - start(): NEW → RUNNABLE                                          │
│  - sleep(ms): RUNNABLE → TIMED_WAITING                              │
│  - wait(): RUNNABLE → WAITING                                       │
│  - join(): waits for thread to finish                               │
│  - yield(): hint to scheduler                                       │
│  - interrupt(): sets interrupt flag                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Basic Thread Operations

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(1000);  // Sleep 1 second
        System.out.println("Done");
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();  // Restore flag
        System.out.println("Interrupted");
    }
});

t.start();
t.join();        // Wait for t to finish
t.join(5000);    // Wait max 5 seconds

// Thread properties
t.setName("Worker-1");
t.setPriority(Thread.MAX_PRIORITY);  // 1-10, default 5
t.setDaemon(true);  // Daemon thread - JVM exits when only daemons left

// Current thread
Thread current = Thread.currentThread();
```

---

## Synchronization

### synchronized Keyword

```java
public class Counter {
    private int count = 0;

    // Synchronized method - locks on 'this'
    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }

    // Synchronized block - more granular
    public void incrementWithBlock() {
        synchronized (this) {
            count++;
        }
    }

    // Static synchronized - locks on Class object
    private static int staticCount = 0;

    public static synchronized void incrementStatic() {
        staticCount++;
    }

    // Or with block
    public static void incrementStaticBlock() {
        synchronized (Counter.class) {
            staticCount++;
        }
    }
}
```

### Monitor Concept

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Java Monitor                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Каждый объект в Java имеет:                                        │
│  1. Lock (mutex) - для synchronized                                 │
│  2. Wait set - для wait/notify                                      │
│                                                                     │
│  synchronized(obj) {         // Acquire lock                        │
│      // Critical section     // Only one thread at a time           │
│      obj.wait();             // Release lock, enter wait set        │
│      // ...                  // Re-acquire lock after notify        │
│  }                           // Release lock                        │
│                                                                     │
│  Thread states:                                                     │
│  - Entry set: waiting to acquire lock (BLOCKED)                     │
│  - Wait set: called wait(), waiting for notify (WAITING)            │
│  - Owner: has the lock, executing                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### wait/notify/notifyAll

```java
public class ProducerConsumer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 10;

    public synchronized void produce(int item) throws InterruptedException {
        // Wait while queue is full
        while (queue.size() == capacity) {
            wait();  // Release lock, wait for space
        }

        queue.add(item);
        System.out.println("Produced: " + item);

        notifyAll();  // Wake up consumers
    }

    public synchronized int consume() throws InterruptedException {
        // Wait while queue is empty
        while (queue.isEmpty()) {
            wait();  // Release lock, wait for items
        }

        int item = queue.poll();
        System.out.println("Consumed: " + item);

        notifyAll();  // Wake up producers
        return item;
    }
}

/*
IMPORTANT:
- Always call wait() in a while loop (spurious wakeups!)
- wait/notify must be called within synchronized block
- wait() releases the lock, notify() does not
- notifyAll() preferred over notify() (safer)
*/
```

---

## volatile Keyword

```java
public class VolatileExample {
    // Without volatile - other threads may see stale value
    private boolean running = true;

    // With volatile - visibility guaranteed
    private volatile boolean stopped = false;

    public void runTask() {
        while (!stopped) {
            // Do work
        }
    }

    public void stop() {
        stopped = true;  // Change visible to all threads immediately
    }
}
```

### volatile vs synchronized

```
┌─────────────────────────────────────────────────────────────────────┐
│              volatile vs synchronized                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  volatile:                       synchronized:                      │
│  ─────────                       ────────────                       │
│  Visibility guarantee            Visibility + atomicity             │
│  No blocking                     Blocking (lock)                    │
│  Single read/write atomic        Compound operations atomic         │
│  No reordering guarantee*        Happens-before guarantee           │
│                                                                     │
│  Use volatile for:               Use synchronized for:              │
│  - Flags (boolean)               - Multiple operations              │
│  - Status fields                 - Read-modify-write                │
│  - One writer, many readers      - Check-then-act                   │
│                                                                     │
│  * volatile does prevent reordering for that specific variable      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// volatile НЕ достаточно для:
private volatile int count = 0;

public void increment() {
    count++;  // NOT atomic! Read + modify + write
}

// Нужно synchronized или AtomicInteger:
public synchronized void incrementSafe() {
    count++;
}

// Или:
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();  // Atomic
```

---

## Locks (java.util.concurrent.locks)

### ReentrantLock

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private int count = 0;
    private final Lock lock = new ReentrantLock();

    public void increment() {
        lock.lock();  // Acquire lock
        try {
            count++;
        } finally {
            lock.unlock();  // Always release in finally!
        }
    }

    // tryLock - non-blocking
    public boolean tryIncrement() {
        if (lock.tryLock()) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }

    // tryLock with timeout
    public boolean tryIncrementWithTimeout() throws InterruptedException {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }

    // Interruptible lock
    public void incrementInterruptibly() throws InterruptedException {
        lock.lockInterruptibly();  // Can be interrupted while waiting
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}
```

### ReadWriteLock

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    private final Map<String, String> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple readers can read simultaneously
    public String get(String key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    // Only one writer, blocks all readers
    public void put(String key, String value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
}
```

### Condition

```java
public class BoundedBuffer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity;
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public void put(int item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait until not full
            }
            queue.add(item);
            notEmpty.signal();  // Signal that not empty
        } finally {
            lock.unlock();
        }
    }

    public int take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait until not empty
            }
            int item = queue.poll();
            notFull.signal();  // Signal that not full
            return item;
        } finally {
            lock.unlock();
        }
    }
}

// Condition advantages over wait/notify:
// - Multiple conditions per lock
// - More expressive API
// - Interruptible, timed waits
```

---

## Thread Safety Issues

### Race Condition

```java
// Race condition - result depends on timing
public class Counter {
    private int count = 0;

    public void increment() {
        count++;  // NOT atomic!
        // Read count → Increment → Write count
        // Thread 1: Read 0
        // Thread 2: Read 0
        // Thread 1: Write 1
        // Thread 2: Write 1
        // Expected: 2, Actual: 1
    }
}
```

### Deadlock

```java
// Deadlock - threads waiting for each other
public class DeadlockExample {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    public void method1() {
        synchronized (lock1) {
            sleep(100);  // Increase chance of deadlock
            synchronized (lock2) {
                // ...
            }
        }
    }

    public void method2() {
        synchronized (lock2) {  // Opposite order!
            sleep(100);
            synchronized (lock1) {
                // ...
            }
        }
    }
}

// Prevention: Always acquire locks in same order
public void method1Safe() {
    synchronized (lock1) {
        synchronized (lock2) {
            // ...
        }
    }
}

public void method2Safe() {
    synchronized (lock1) {  // Same order as method1
        synchronized (lock2) {
            // ...
        }
    }
}
```

### Livelock & Starvation

```java
// Livelock - threads keep changing state without progress
// Example: Two people in corridor trying to pass each other

// Starvation - thread never gets CPU/lock
// Can happen with unfair locks or priority issues

// ReentrantLock can be fair
Lock fairLock = new ReentrantLock(true);  // FIFO ordering
```

---

## Atomic Variables

```java
import java.util.concurrent.atomic.*;

// AtomicInteger - thread-safe integer
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();     // ++counter
counter.getAndIncrement();     // counter++
counter.addAndGet(5);          // counter += 5
counter.compareAndSet(5, 10);  // CAS operation

// AtomicLong
AtomicLong longCounter = new AtomicLong(0);

// AtomicReference - for objects
AtomicReference<String> ref = new AtomicReference<>("initial");
ref.set("new value");
ref.compareAndSet("new value", "updated");

// AtomicBoolean
AtomicBoolean flag = new AtomicBoolean(false);
flag.set(true);
flag.compareAndSet(true, false);

// Array versions
AtomicIntegerArray array = new AtomicIntegerArray(10);
array.incrementAndGet(0);  // Increment element at index 0

// LongAdder - better for high contention
LongAdder adder = new LongAdder();
adder.increment();
adder.add(5);
long sum = adder.sum();  // Get total

// Use LongAdder for counters with high write contention
// Use AtomicLong for sequences or when exact value needed
```

---

## См. также

- [Concurrency Advanced](./05-concurrency-advanced.md) — продвинутые механизмы многопоточности в Java
- [Goroutines & Channels](../00-go/01-goroutines-channels.md) — модель конкурентности в Go

---

## На интервью

### Типичные вопросы

1. **synchronized vs Lock?**
   - synchronized: simpler, auto-release, reentrant
   - Lock: tryLock, timeout, interruptible, multiple conditions
   - Lock: must manually release in finally

2. **volatile vs synchronized?**
   - volatile: visibility only, no atomicity
   - synchronized: visibility + atomicity + mutual exclusion
   - volatile for flags, synchronized for compound operations

3. **What is deadlock? How to prevent?**
   - Threads waiting for each other's locks
   - Prevention: lock ordering, timeout, avoid nested locks

4. **wait() vs sleep()?**
   - wait(): releases lock, must be in synchronized
   - sleep(): doesn't release lock, can be anywhere
   - Both throw InterruptedException

5. **Why use while loop with wait()?**
   - Spurious wakeups possible
   - Condition may change between notify and wake
   - Always recheck condition

6. **Thread states?**
   - NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED

---

[← Назад к списку тем](README.md)
