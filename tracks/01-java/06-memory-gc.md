# 06. Memory & Garbage Collection

[← Назад к списку тем](README.md)

---

## Java Memory Model (JMM)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Java Memory Model                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Main Memory                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Shared variables (heap, static fields)                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           ▲                     ▲                     ▲             │
│           │ read/write          │                     │             │
│           │                     │                     │             │
│  ┌────────┴───────┐   ┌────────┴───────┐   ┌────────┴───────┐     │
│  │ Thread 1       │   │ Thread 2       │   │ Thread 3       │     │
│  │ Local Memory   │   │ Local Memory   │   │ Local Memory   │     │
│  │ (CPU cache)    │   │ (CPU cache)    │   │ (CPU cache)    │     │
│  └────────────────┘   └────────────────┘   └────────────────┘     │
│                                                                     │
│  Problems:                                                          │
│  - Visibility: changes may not be visible to other threads          │
│  - Reordering: compiler/CPU may reorder instructions                │
│                                                                     │
│  Solutions:                                                         │
│  - synchronized: visibility + atomicity                             │
│  - volatile: visibility + ordering                                  │
│  - final: safe publication                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Happens-Before

```java
/*
Happens-before guarantees:

1. Program order: statement A before B in same thread
2. Monitor lock: unlock happens-before subsequent lock
3. Volatile: write happens-before subsequent read
4. Thread start: start() happens-before any action in thread
5. Thread join: all actions in thread happen-before join() returns
6. Transitivity: if A h-b B and B h-b C, then A h-b C
*/

// Example: happens-before with volatile
class Example {
    int x = 0;
    volatile boolean ready = false;

    // Thread 1
    void writer() {
        x = 42;           // 1
        ready = true;     // 2 (volatile write)
    }

    // Thread 2
    void reader() {
        if (ready) {      // 3 (volatile read)
            assert x == 42; // 4 - guaranteed!
        }
    }
}
// 1 happens-before 2 (program order)
// 2 happens-before 3 (volatile)
// 3 happens-before 4 (program order)
// Therefore: 1 happens-before 4 (transitivity)
```

---

## Garbage Collection

### Heap Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Heap Structure                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Young Generation                           │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │     Eden                                                 │ │  │
│  │  │   (new objects)                                          │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────┐  ┌────────────────────┐             │  │
│  │  │    Survivor 0       │  │    Survivor 1       │             │  │
│  │  └────────────────────┘  └────────────────────┘             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Old Generation                             │  │
│  │                  (long-lived objects)                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Object Flow:                                                       │
│  1. New object → Eden                                               │
│  2. Minor GC: live objects Eden → S0 or S1                          │
│  3. Subsequent minor GCs: S0 ↔ S1, age incremented                  │
│  4. Age threshold reached → Old Generation                          │
│  5. Large objects → directly to Old Gen                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### GC Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GC Event Types                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Minor GC (Young Generation):                                       │
│  - Triggered: Eden full                                             │
│  - Collects: Young Gen only                                         │
│  - Speed: Fast (usually < 100ms)                                    │
│  - STW: Yes, but short                                              │
│                                                                     │
│  Major GC (Old Generation):                                         │
│  - Triggered: Old Gen full                                          │
│  - Collects: Old Gen (sometimes Young too)                          │
│  - Speed: Slow                                                      │
│  - STW: Yes, longer pause                                           │
│                                                                     │
│  Full GC:                                                           │
│  - Triggered: System.gc(), OOM prevention                           │
│  - Collects: Entire heap + Metaspace                                │
│  - Speed: Slowest                                                   │
│  - STW: Yes, longest pause                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Garbage Collectors

### Serial GC

```bash
# Single-threaded, STW
# Good for: small heaps, single CPU
-XX:+UseSerialGC
```

### Parallel GC

```bash
# Multi-threaded, STW
# Good for: throughput, batch jobs
-XX:+UseParallelGC
-XX:ParallelGCThreads=4
```

### G1 GC (default since Java 9)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    G1 Garbage Collector                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Region-based heap:                                                 │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                         │
│  │ E │ E │ S │ O │ O │ O │ H │ O │ E │ S │                         │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                         │
│                                                                     │
│  E = Eden, S = Survivor, O = Old, H = Humongous                     │
│                                                                     │
│  Features:                                                          │
│  - Regions instead of fixed generations                             │
│  - Predictable pause times (target: -XX:MaxGCPauseMillis=200)       │
│  - Concurrent marking                                               │
│  - Evacuates regions with most garbage first ("Garbage-First")      │
│                                                                     │
│  Phases:                                                            │
│  1. Young GC: evacuate young regions (STW)                          │
│  2. Concurrent marking: find live objects (mostly concurrent)       │
│  3. Mixed GC: young + selected old regions (STW)                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
-XX:+UseG1GC                    # Enable G1 (default Java 9+)
-XX:MaxGCPauseMillis=200        # Target max pause time
-XX:G1HeapRegionSize=4m         # Region size (1-32MB)
-XX:G1NewSizePercent=5          # Min young gen
-XX:G1MaxNewSizePercent=60      # Max young gen
```

### ZGC (Java 11+)

```bash
# Ultra-low latency (<10ms pauses)
# Concurrent, scalable to TB heaps
-XX:+UseZGC
-XX:ZCollectionInterval=60      # Force GC every N seconds
```

### Shenandoah (Java 12+)

```bash
# Low latency, concurrent compaction
-XX:+UseShenandoahGC
```

### GC Comparison

```
┌─────────────────┬──────────┬───────────┬────────────┬──────────────┐
│ GC              │ Latency  │ Throughput│ Heap Size  │ Use Case     │
├─────────────────┼──────────┼───────────┼────────────┼──────────────┤
│ Serial          │ High     │ Medium    │ < 100MB    │ Single CPU   │
│ Parallel        │ Medium   │ High      │ Medium     │ Batch jobs   │
│ G1              │ Low      │ Good      │ > 4GB      │ General      │
│ ZGC             │ Very Low │ Medium    │ > 8GB      │ Low latency  │
│ Shenandoah      │ Very Low │ Medium    │ > 8GB      │ Low latency  │
└─────────────────┴──────────┴───────────┴────────────┴──────────────┘
```

---

## Memory Leaks

### Common Causes

```java
// 1. Static collections
public class Cache {
    private static Map<String, Object> cache = new HashMap<>();

    public static void add(String key, Object value) {
        cache.put(key, value);  // Never removed!
    }
}

// Solution: Use weak references or explicit cleanup
private static Map<String, WeakReference<Object>> cache = new HashMap<>();

// 2. Unclosed resources
public void readFile(String path) {
    InputStream is = new FileInputStream(path);  // Never closed!
    // ...
}

// Solution: try-with-resources
public void readFile(String path) {
    try (InputStream is = new FileInputStream(path)) {
        // ...
    }
}

// 3. Inner class holding outer reference
public class Outer {
    private byte[] largeArray = new byte[1_000_000];

    public Runnable getTask() {
        return new Runnable() {  // Holds reference to Outer
            public void run() {
                System.out.println("Running");
            }
        };
    }
}

// Solution: Use static inner class or lambda
public Runnable getTask() {
    return () -> System.out.println("Running");  // No reference to Outer
}

// 4. Listeners not removed
button.addActionListener(listener);
// ... listener never removed
// button = null;  // Listener still holds reference

// Solution: Remove listeners explicitly
button.removeActionListener(listener);

// 5. ThreadLocal not removed
private static ThreadLocal<Connection> conn = new ThreadLocal<>();

public void process() {
    conn.set(getConnection());
    try {
        // ... use connection
    } finally {
        conn.remove();  // IMPORTANT: clear after use
    }
}
```

### Detection Tools

```bash
# Heap dump
jmap -dump:format=b,file=heap.hprof <pid>

# Analyze with tools:
# - Eclipse MAT (Memory Analyzer Tool)
# - VisualVM
# - JProfiler

# GC logging
-Xlog:gc*:file=gc.log
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

---

## Reference Types

```java
// Strong Reference - prevents GC
Object obj = new Object();  // Strong reference

// Soft Reference - GC'd when memory low
SoftReference<Object> soft = new SoftReference<>(new Object());
Object value = soft.get();  // may be null if GC'd
// Use for: caches

// Weak Reference - GC'd at next GC
WeakReference<Object> weak = new WeakReference<>(new Object());
Object value = weak.get();  // may be null after GC
// Use for: caches that shouldn't prevent GC

// Phantom Reference - for cleanup notification
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), queue);
// get() always returns null
// Use for: cleanup actions before object reclaimed

// WeakHashMap - keys are weak references
Map<Key, Value> cache = new WeakHashMap<>();
// Entry removed when key is GC'd
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Reference Strength                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Strong > Soft > Weak > Phantom                                     │
│                                                                     │
│  Strong:  Never GC'd while reachable                                │
│  Soft:    GC'd when heap is low (before OOM)                        │
│  Weak:    GC'd at any GC cycle                                      │
│  Phantom: For finalization replacement                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## JVM Tuning

### Common Parameters

```bash
# Heap size
-Xms512m              # Initial heap
-Xmx2g                # Maximum heap
-Xmn256m              # Young generation size

# Metaspace
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# Stack
-Xss512k              # Thread stack size

# GC selection
-XX:+UseG1GC
-XX:+UseZGC
-XX:+UseParallelGC

# G1 tuning
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=4m

# GC logging (Java 9+)
-Xlog:gc*:file=gc.log:time,level,tags

# Heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof

# JIT
-XX:+TieredCompilation
-XX:CompileThreshold=10000
```

### Monitoring Commands

```bash
# JVM statistics
jstat -gc <pid> 1000     # GC stats every 1s
jstat -gcutil <pid>      # GC utilization %

# Thread dump
jstack <pid>

# Heap histogram
jmap -histo <pid>

# JVM info
jinfo <pid>

# GUI tools
jconsole
jvisualvm
```

---

## На интервью

### Типичные вопросы

1. **Heap structure?**
   - Young Gen: Eden + 2 Survivors
   - Old Gen: long-lived objects
   - Minor GC in Young, Major GC in Old

2. **What is GC?**
   - Automatic memory reclamation
   - Finds unreachable objects
   - Different algorithms: Serial, Parallel, G1, ZGC

3. **G1 vs ZGC?**
   - G1: balanced latency/throughput, default
   - ZGC: ultra-low latency (<10ms), larger heaps

4. **Memory leak causes?**
   - Static collections, unclosed resources
   - Inner class references, listeners
   - ThreadLocal not cleaned

5. **Reference types?**
   - Strong (normal), Soft (cache), Weak (WeakHashMap)
   - Phantom (cleanup notification)

6. **happens-before?**
   - Memory visibility guarantee
   - synchronized, volatile, thread start/join

---

[← Назад к списку тем](README.md)
