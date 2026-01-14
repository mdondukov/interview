# 05. Concurrency Advanced

[← Назад к списку тем](README.md)

---

## ExecutorService

### Thread Pools

```java
import java.util.concurrent.*;

// Fixed thread pool - fixed number of threads
ExecutorService fixed = Executors.newFixedThreadPool(4);

// Cached thread pool - creates threads as needed, reuses idle
ExecutorService cached = Executors.newCachedThreadPool();

// Single thread executor - one thread, queue tasks
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled executor - delayed/periodic execution
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

// Work-stealing pool (Java 8+) - uses ForkJoinPool
ExecutorService workStealing = Executors.newWorkStealingPool();

// Custom ThreadPoolExecutor
ThreadPoolExecutor custom = new ThreadPoolExecutor(
    2,                      // corePoolSize
    4,                      // maxPoolSize
    60, TimeUnit.SECONDS,   // keepAliveTime for idle threads
    new ArrayBlockingQueue<>(100),  // work queue
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
);
```

### Submitting Tasks

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit Runnable - no result
executor.execute(() -> System.out.println("Running"));

// Submit Runnable - returns Future<?>
Future<?> future1 = executor.submit(() -> System.out.println("Running"));

// Submit Callable - returns Future<T>
Future<Integer> future2 = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});

// Get result (blocking)
try {
    Integer result = future2.get();  // Blocks until done
    Integer result2 = future2.get(5, TimeUnit.SECONDS);  // With timeout
} catch (InterruptedException | ExecutionException | TimeoutException e) {
    e.printStackTrace();
}

// Check status
future2.isDone();
future2.isCancelled();
future2.cancel(true);  // mayInterruptIfRunning

// Submit multiple tasks
List<Callable<Integer>> tasks = List.of(
    () -> 1,
    () -> 2,
    () -> 3
);

List<Future<Integer>> futures = executor.invokeAll(tasks);  // Wait for all
Integer firstResult = executor.invokeAny(tasks);  // Return first completed

// Shutdown
executor.shutdown();  // Stop accepting, finish existing
executor.shutdownNow();  // Stop accepting, interrupt running
executor.awaitTermination(60, TimeUnit.SECONDS);  // Wait for termination
```

### ThreadPoolExecutor Configuration

```
┌─────────────────────────────────────────────────────────────────────┐
│              ThreadPoolExecutor Behavior                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. If threads < corePoolSize → create new thread                   │
│  2. If threads >= corePoolSize → queue task                         │
│  3. If queue full and threads < maxPoolSize → create new thread     │
│  4. If queue full and threads >= maxPoolSize → reject               │
│                                                                     │
│  Rejection Policies:                                                │
│  - AbortPolicy (default): throws RejectedExecutionException         │
│  - CallerRunsPolicy: caller thread executes task                    │
│  - DiscardPolicy: silently discard task                             │
│  - DiscardOldestPolicy: discard oldest, try again                   │
│                                                                     │
│  Queue Types:                                                       │
│  - LinkedBlockingQueue: unbounded (FixedThreadPool uses this)       │
│  - ArrayBlockingQueue: bounded                                      │
│  - SynchronousQueue: no capacity (CachedThreadPool uses this)       │
│  - PriorityBlockingQueue: priority ordering                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CompletableFuture

### Basic Usage

```java
// Create completed future
CompletableFuture<String> cf = CompletableFuture.completedFuture("result");

// Async execution (uses ForkJoinPool.commonPool())
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // Long computation
    return "Hello";
});

// Async with custom executor
ExecutorService executor = Executors.newFixedThreadPool(4);
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "World";
}, executor);

// No return value
CompletableFuture<Void> runFuture = CompletableFuture.runAsync(() -> {
    System.out.println("Running async");
});

// Get result
String result = future.get();           // Blocking, throws checked exceptions
String result2 = future.join();         // Blocking, throws unchecked exceptions
String result3 = future.getNow("default");  // Non-blocking, return default if not done
```

### Chaining Operations

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello");

// Transform result (synchronous)
CompletableFuture<Integer> length = future.thenApply(s -> s.length());

// Transform result (asynchronous)
CompletableFuture<Integer> lengthAsync = future.thenApplyAsync(s -> s.length());

// Consume result (no return)
CompletableFuture<Void> consumed = future.thenAccept(s -> System.out.println(s));

// Run after completion (no input, no output)
CompletableFuture<Void> after = future.thenRun(() -> System.out.println("Done"));

// Chain multiple operations
CompletableFuture<String> chain = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")
    .thenApply(String::toUpperCase);  // "HELLO WORLD"
```

### Combining Futures

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

// Combine two futures
CompletableFuture<String> combined = future1.thenCombine(future2,
    (s1, s2) -> s1 + " " + s2);  // "Hello World"

// Run after both complete
CompletableFuture<Void> both = future1.thenAcceptBoth(future2,
    (s1, s2) -> System.out.println(s1 + " " + s2));

// Run after first completes
CompletableFuture<String> first = future1.applyToEither(future2,
    s -> s.toUpperCase());

// Compose futures (flatMap)
CompletableFuture<String> composed = future1.thenCompose(s ->
    CompletableFuture.supplyAsync(() -> s + " World"));

// Wait for all
CompletableFuture<Void> allOf = CompletableFuture.allOf(future1, future2);

// Wait for any
CompletableFuture<Object> anyOf = CompletableFuture.anyOf(future1, future2);
```

### Error Handling

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Error!");
    return "Success";
});

// Handle exception - returns new result
CompletableFuture<String> handled = future.exceptionally(ex -> {
    System.out.println("Error: " + ex.getMessage());
    return "Default";
});

// Handle both success and failure
CompletableFuture<String> handledAll = future.handle((result, ex) -> {
    if (ex != null) {
        return "Error: " + ex.getMessage();
    }
    return result;
});

// Execute regardless of outcome
CompletableFuture<String> withFinally = future.whenComplete((result, ex) -> {
    if (ex != null) {
        System.out.println("Failed: " + ex.getMessage());
    } else {
        System.out.println("Succeeded: " + result);
    }
});
```

### Practical Example

```java
public CompletableFuture<OrderResult> processOrder(Order order) {
    return CompletableFuture
        // Validate order
        .supplyAsync(() -> validateOrder(order))

        // Check inventory in parallel with user validation
        .thenCombine(
            CompletableFuture.supplyAsync(() -> checkInventory(order)),
            (validated, inventory) -> {
                if (!validated || !inventory) {
                    throw new OrderException("Validation failed");
                }
                return order;
            })

        // Process payment
        .thenCompose(validOrder ->
            CompletableFuture.supplyAsync(() -> processPayment(validOrder)))

        // Send confirmation
        .thenApply(payment -> {
            sendConfirmation(order, payment);
            return new OrderResult(order, payment);
        })

        // Handle errors
        .exceptionally(ex -> {
            logError(ex);
            return new OrderResult(order, null, ex.getMessage());
        });
}
```

---

## Fork/Join Framework

### ForkJoinPool

```java
// ForkJoinPool - work-stealing pool for divide-and-conquer tasks
ForkJoinPool pool = new ForkJoinPool();  // Default parallelism = CPU cores
ForkJoinPool pool2 = new ForkJoinPool(4);  // Custom parallelism

// Common pool (shared, used by parallel streams)
ForkJoinPool common = ForkJoinPool.commonPool();

// Submit tasks
ForkJoinTask<Long> task = pool.submit(new SumTask(array, 0, array.length));
Long result = task.join();
```

### RecursiveTask (returns result)

```java
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000;
    private final long[] array;
    private final int start, end;

    public SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;

        // Small enough - compute directly
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        }

        // Split into subtasks
        int mid = start + length / 2;

        SumTask leftTask = new SumTask(array, start, mid);
        SumTask rightTask = new SumTask(array, mid, end);

        // Fork left task (async execution)
        leftTask.fork();

        // Compute right task in current thread
        Long rightResult = rightTask.compute();

        // Join left task result
        Long leftResult = leftTask.join();

        return leftResult + rightResult;
    }
}

// Usage
long[] array = new long[10_000_000];
ForkJoinPool pool = new ForkJoinPool();
Long sum = pool.invoke(new SumTask(array, 0, array.length));
```

### RecursiveAction (no result)

```java
public class SortTask extends RecursiveAction {
    private static final int THRESHOLD = 1000;
    private final int[] array;
    private final int start, end;

    public SortTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            Arrays.sort(array, start, end);
            return;
        }

        int mid = start + (end - start) / 2;

        SortTask left = new SortTask(array, start, mid);
        SortTask right = new SortTask(array, mid, end);

        invokeAll(left, right);  // Fork both, wait for both

        merge(array, start, mid, end);
    }

    private void merge(int[] array, int start, int mid, int end) {
        // Merge sorted halves
    }
}
```

---

## Synchronizers

### CountDownLatch

```java
// Wait for N events to occur
CountDownLatch latch = new CountDownLatch(3);

// Worker threads
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        latch.countDown();  // Signal completion
    }).start();
}

// Main thread waits
latch.await();  // Blocks until count reaches 0
System.out.println("All workers done");

// One-shot - cannot be reset
```

### CyclicBarrier

```java
// N threads wait for each other
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All parties arrived!");  // Barrier action
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        doWork();
        barrier.await();  // Wait for others
        // All threads proceed together
    }).start();
}

// Can be reused (cyclic)
```

### Semaphore

```java
// Limit concurrent access
Semaphore semaphore = new Semaphore(3);  // 3 permits

// Worker thread
semaphore.acquire();  // Get permit (blocks if none available)
try {
    accessLimitedResource();
} finally {
    semaphore.release();  // Return permit
}

// Try without blocking
if (semaphore.tryAcquire()) {
    try {
        accessLimitedResource();
    } finally {
        semaphore.release();
    }
}

// Multiple permits
semaphore.acquire(2);
semaphore.release(2);
```

### Phaser

```java
// Flexible barrier for dynamic number of parties
Phaser phaser = new Phaser(3);  // 3 initial parties

// Worker
new Thread(() -> {
    for (int phase = 0; phase < 3; phase++) {
        doWork(phase);
        phaser.arriveAndAwaitAdvance();  // Wait for others
    }
    phaser.arriveAndDeregister();  // Leave the phaser
}).start();

// Dynamic registration
phaser.register();  // Add party
phaser.arriveAndDeregister();  // Remove party
```

### Exchanger

```java
// Exchange data between two threads
Exchanger<String> exchanger = new Exchanger<>();

// Thread 1
String data1 = "Data from Thread 1";
String received1 = exchanger.exchange(data1);  // Blocks until thread 2 exchanges
System.out.println("Thread 1 received: " + received1);

// Thread 2
String data2 = "Data from Thread 2";
String received2 = exchanger.exchange(data2);  // Blocks until thread 1 exchanges
System.out.println("Thread 2 received: " + received2);
```

---

## Parallel Streams

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Sequential
int sum1 = numbers.stream()
    .mapToInt(Integer::intValue)
    .sum();

// Parallel
int sum2 = numbers.parallelStream()
    .mapToInt(Integer::intValue)
    .sum();

// Or convert to parallel
int sum3 = numbers.stream()
    .parallel()
    .mapToInt(Integer::intValue)
    .sum();

// Uses ForkJoinPool.commonPool() by default

// Custom pool
ForkJoinPool customPool = new ForkJoinPool(4);
int sum4 = customPool.submit(() ->
    numbers.parallelStream()
        .mapToInt(Integer::intValue)
        .sum()
).join();
```

### When to Use Parallel Streams

```
┌─────────────────────────────────────────────────────────────────────┐
│            Parallel Streams - When to Use                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ GOOD for:                                                        │
│  - Large datasets (10K+ elements)                                   │
│  - CPU-intensive operations                                         │
│  - Independent elements                                             │
│  - ArrayList, arrays (good split)                                   │
│                                                                     │
│  ✗ BAD for:                                                         │
│  - Small datasets (overhead > benefit)                              │
│  - I/O operations (blocking)                                        │
│  - Ordered operations (forEach ordering lost)                       │
│  - LinkedList (bad split)                                           │
│  - Shared mutable state                                             │
│                                                                     │
│  Rule of thumb: Measure! Don't assume parallel is faster            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **Thread pool types?**
   - Fixed: fixed threads, unbounded queue
   - Cached: creates as needed, 60s keep-alive
   - Single: one thread, sequential tasks
   - Scheduled: delayed/periodic execution
   - WorkStealing: ForkJoinPool

2. **Future vs CompletableFuture?**
   - Future: blocking get(), limited API
   - CompletableFuture: non-blocking, chaining, combining, error handling

3. **thenApply vs thenCompose?**
   - thenApply: sync transform, like map
   - thenCompose: async transform, like flatMap

4. **Fork/Join framework?**
   - Divide-and-conquer parallelism
   - Work-stealing for load balancing
   - RecursiveTask (with result), RecursiveAction (no result)

5. **CountDownLatch vs CyclicBarrier?**
   - CountDownLatch: one-shot, wait for N events
   - CyclicBarrier: reusable, N threads wait for each other

6. **When to use parallel streams?**
   - Large data, CPU-intensive, independent elements
   - Avoid for I/O, small data, shared state

---

[← Назад к списку тем](README.md)
