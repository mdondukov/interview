# 00. JVM Fundamentals

[← Назад к списку тем](README.md)

---

## JVM Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      JVM Architecture                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  .java → javac → .class (bytecode) → JVM                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Class Loader Subsystem                    │   │
│  │  Bootstrap → Extension → Application → Custom                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Runtime Data Areas                        │   │
│  │  ┌──────────────┬──────────────┬──────────────────────────┐ │   │
│  │  │  Method Area │     Heap     │  Thread-specific:        │ │   │
│  │  │  (Metaspace) │              │  - PC Register           │ │   │
│  │  │              │              │  - JVM Stack             │ │   │
│  │  │  - Classes   │  - Objects   │  - Native Method Stack   │ │   │
│  │  │  - Methods   │  - Arrays    │                          │ │   │
│  │  │  - Constants │              │                          │ │   │
│  │  └──────────────┴──────────────┴──────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Execution Engine                          │   │
│  │  - Interpreter                                               │   │
│  │  - JIT Compiler (C1, C2)                                     │   │
│  │  - Garbage Collector                                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Runtime Data Areas

### Method Area (Metaspace)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Method Area (Metaspace)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Shared between all threads                                         │
│                                                                     │
│  Contains:                                                          │
│  - Class metadata (structure, methods, fields)                      │
│  - Constant Pool (literals, symbolic references)                    │
│  - Static variables                                                 │
│  - Method bytecode                                                  │
│  - JIT compiled code (Code Cache)                                   │
│                                                                     │
│  Java 8+: Metaspace (native memory, not heap)                       │
│  Before Java 8: PermGen (part of heap)                              │
│                                                                     │
│  Tuning:                                                            │
│  -XX:MetaspaceSize=256m                                             │
│  -XX:MaxMetaspaceSize=512m                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Heap

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Java Heap                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     Young Generation                          │  │
│  │  ┌─────────────┬─────────────┬─────────────┐                 │  │
│  │  │    Eden     │ Survivor 0  │ Survivor 1  │                 │  │
│  │  │   (new)     │    (S0)     │    (S1)     │                 │  │
│  │  └─────────────┴─────────────┴─────────────┘                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     Old Generation                            │  │
│  │                   (long-lived objects)                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Object lifecycle:                                                  │
│  1. New object → Eden                                               │
│  2. Survives Minor GC → S0 or S1                                    │
│  3. After N GCs (threshold) → Old Gen                               │
│  4. Large objects → directly to Old Gen                             │
│                                                                     │
│  Tuning:                                                            │
│  -Xms512m         (initial heap)                                    │
│  -Xmx2g           (max heap)                                        │
│  -Xmn256m         (young generation size)                           │
│  -XX:NewRatio=2   (Old/Young ratio)                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Stack (per thread)

```java
// Each method call creates a stack frame
public int calculate(int a, int b) {
    int result = a + b;  // Local variable in stack frame
    return result;
}

/*
Stack Frame contains:
┌─────────────────────────────────┐
│ Local Variables Array           │  ← a, b, result
├─────────────────────────────────┤
│ Operand Stack                   │  ← for computations
├─────────────────────────────────┤
│ Frame Data                      │  ← return address, exception table
│ (Constant Pool reference)       │
└─────────────────────────────────┘

Tuning:
-Xss512k  (stack size per thread)
*/
```

---

## Class Loading

### Class Loader Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Class Loader Hierarchy                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Bootstrap ClassLoader (native, loads rt.jar)                       │
│           │                                                         │
│           ▼                                                         │
│  Platform/Extension ClassLoader (jre/lib/ext)                       │
│           │                                                         │
│           ▼                                                         │
│  Application ClassLoader (classpath)                                │
│           │                                                         │
│           ▼                                                         │
│  Custom ClassLoaders (user-defined)                                 │
│                                                                     │
│  Delegation Model (Parent-First):                                   │
│  1. Check if class already loaded                                   │
│  2. Delegate to parent                                              │
│  3. If parent can't load, try self                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Class Loading Process

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Class Loading Phases                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. LOADING                                                         │
│     - Find .class file                                              │
│     - Read bytecode into memory                                     │
│     - Create Class object in Method Area                            │
│                                                                     │
│  2. LINKING                                                         │
│     a) Verification                                                 │
│        - Bytecode verification                                      │
│        - Check class file format                                    │
│     b) Preparation                                                  │
│        - Allocate memory for static fields                          │
│        - Initialize with default values                             │
│     c) Resolution                                                   │
│        - Resolve symbolic references                                │
│        - Can be lazy (at first use)                                 │
│                                                                     │
│  3. INITIALIZATION                                                  │
│     - Execute static initializers                                   │
│     - Execute static blocks                                         │
│     - Thread-safe (class is locked)                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Custom ClassLoader

```java
public class CustomClassLoader extends ClassLoader {

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = loadClassBytes(name);
        if (bytes == null) {
            throw new ClassNotFoundException(name);
        }
        return defineClass(name, bytes, 0, bytes.length);
    }

    private byte[] loadClassBytes(String name) {
        // Load from custom source (file, network, encrypted, etc.)
        String fileName = name.replace('.', '/') + ".class";
        try (InputStream is = getResourceAsStream(fileName)) {
            return is.readAllBytes();
        } catch (IOException e) {
            return null;
        }
    }
}

// Usage
ClassLoader loader = new CustomClassLoader();
Class<?> clazz = loader.loadClass("com.example.MyClass");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

---

## JIT Compilation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    JIT Compilation                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Execution modes:                                                   │
│  - Interpreter: bytecode → execute (slow, startup fast)             │
│  - JIT Compiler: bytecode → native code (fast, startup slow)        │
│                                                                     │
│  HotSpot JVM uses tiered compilation:                               │
│                                                                     │
│  Level 0: Interpreter                                               │
│      ↓ (after invocation threshold)                                 │
│  Level 1-3: C1 Compiler (Client)                                    │
│      ↓ (after more invocations)                                     │
│  Level 4: C2 Compiler (Server) - aggressive optimizations           │
│                                                                     │
│  C1 Optimizations:                                                  │
│  - Inlining (small methods)                                         │
│  - Simple escape analysis                                           │
│                                                                     │
│  C2 Optimizations:                                                  │
│  - Advanced inlining                                                │
│  - Escape analysis (stack allocation)                               │
│  - Lock elision                                                     │
│  - Loop unrolling                                                   │
│  - Dead code elimination                                            │
│  - Speculative optimizations (based on profiling)                   │
│                                                                     │
│  Flags:                                                             │
│  -XX:+TieredCompilation (default on)                                │
│  -XX:CompileThreshold=10000                                         │
│  -XX:+PrintCompilation                                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### JIT Optimizations Example

```java
// Escape Analysis - object doesn't escape, can allocate on stack
public int sumPoints() {
    Point p = new Point(1, 2);  // May be stack-allocated
    return p.x + p.y;
}

// Lock Elision - lock removed if no contention possible
public void process() {
    Object lock = new Object();
    synchronized (lock) {       // Lock can be eliminated
        // ... work ...         // (lock doesn't escape)
    }
}

// Method Inlining
public int calculate(int x) {
    return square(x) + 1;      // square() inlined
}
private int square(int n) {
    return n * n;
}
// Becomes effectively:
// return x * x + 1;
```

---

## Bytecode

### Viewing Bytecode

```bash
# Compile
javac MyClass.java

# View bytecode
javap -c MyClass.class

# Verbose (with constant pool)
javap -v MyClass.class
```

### Bytecode Example

```java
public int add(int a, int b) {
    return a + b;
}
```

```
// Bytecode:
public int add(int, int);
  Code:
     0: iload_1        // Load int from local var 1 (a) onto stack
     1: iload_2        // Load int from local var 2 (b) onto stack
     2: iadd           // Add two ints on stack
     3: ireturn        // Return int from stack
```

### Common Bytecode Instructions

```
┌─────────────────┬──────────────────────────────────────────────────┐
│ Instruction     │ Description                                      │
├─────────────────┼──────────────────────────────────────────────────┤
│ iload, lload    │ Load int/long from local variable                │
│ istore, lstore  │ Store int/long to local variable                 │
│ iconst, ldc     │ Push constant onto stack                         │
│ iadd, isub      │ Integer arithmetic                               │
│ if_icmplt       │ Compare and branch                               │
│ goto            │ Unconditional jump                               │
│ invokevirtual   │ Call instance method                             │
│ invokestatic    │ Call static method                               │
│ invokespecial   │ Call constructor, super, private                 │
│ invokeinterface │ Call interface method                            │
│ invokedynamic   │ Dynamic dispatch (lambdas)                       │
│ new             │ Create object                                    │
│ getfield        │ Get instance field                               │
│ getstatic       │ Get static field                                 │
│ checkcast       │ Type cast                                        │
│ athrow          │ Throw exception                                  │
└─────────────────┴──────────────────────────────────────────────────┘
```

---

## String Pool

```
┌─────────────────────────────────────────────────────────────────────┐
│                      String Pool                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  String Pool (String Intern Pool):                                  │
│  - Special memory region for String literals                        │
│  - Stored in Heap (since Java 7)                                    │
│  - Enables string reuse (memory optimization)                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// Literals go to String Pool
String s1 = "hello";           // → Pool
String s2 = "hello";           // → Same object from Pool
System.out.println(s1 == s2);  // true (same reference)

// new creates object in Heap
String s3 = new String("hello");  // → Heap (new object)
System.out.println(s1 == s3);     // false (different references)
System.out.println(s1.equals(s3)); // true (same content)

// intern() returns pooled version
String s4 = s3.intern();       // → Pool
System.out.println(s1 == s4);  // true

// Concatenation at compile time → Pool
String s5 = "hel" + "lo";      // Optimized to "hello" → Pool
System.out.println(s1 == s5);  // true

// Concatenation at runtime → Heap
String part = "lo";
String s6 = "hel" + part;      // → Heap (new object)
System.out.println(s1 == s6);  // false
```

---

## На интервью

### Типичные вопросы

1. **JVM memory areas?**
   - Heap: objects (shared)
   - Method Area/Metaspace: class metadata (shared)
   - Stack: method frames, local vars (per thread)
   - PC Register: current instruction (per thread)
   - Native Method Stack (per thread)

2. **Class loading process?**
   - Loading → Linking (Verify, Prepare, Resolve) → Initialization
   - Parent-first delegation
   - Static initializers run at initialization

3. **What is JIT?**
   - Just-In-Time compilation
   - Bytecode → native code at runtime
   - Tiered: Interpreter → C1 → C2
   - Optimizations: inlining, escape analysis, lock elision

4. **PermGen vs Metaspace?**
   - PermGen: fixed size, part of heap, OutOfMemoryError
   - Metaspace (Java 8+): native memory, auto-grows, no PermGen errors

5. **String Pool?**
   - Pool of unique String literals
   - Stored in Heap (since Java 7)
   - Literals reused, new String() creates separate object
   - intern() returns pooled version

6. **What can cause OutOfMemoryError?**
   - Heap: too many objects
   - Metaspace: too many classes loaded
   - Stack: deep recursion (StackOverflowError)
   - Native memory: direct buffers, native libs

---

[← Назад к списку тем](README.md)
