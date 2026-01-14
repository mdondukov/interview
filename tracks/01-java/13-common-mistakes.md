# 13. Common Mistakes & Best Practices

[← Назад к списку тем](README.md)

---

## String Handling

### String Concatenation in Loops

```java
// BAD - O(n²) complexity, creates many String objects
String result = "";
for (String item : items) {
    result += item + ", ";  // Creates new String each iteration!
}

// GOOD - O(n) complexity
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item).append(", ");
}
String result = sb.toString();

// BEST - use joining
String result = String.join(", ", items);
// or
String result = items.stream().collect(Collectors.joining(", "));
```

### String Comparison

```java
// BAD - can throw NullPointerException
if (userInput.equals("expected")) { }

// GOOD - null-safe
if ("expected".equals(userInput)) { }

// Or use Objects.equals
if (Objects.equals(userInput, "expected")) { }

// For case-insensitive
if ("expected".equalsIgnoreCase(userInput)) { }

// NEVER use == for String comparison (except interned strings)
String s1 = new String("hello");
String s2 = new String("hello");
s1 == s2;      // false (different objects)
s1.equals(s2); // true (same content)
```

### Empty String Check

```java
// Verbose
if (s != null && !s.isEmpty()) { }

// Better (Java 11+)
if (s != null && !s.isBlank()) { }  // Also handles whitespace

// Using utility
if (StringUtils.isNotBlank(s)) { }  // Apache Commons

// Modern Optional approach
Optional.ofNullable(s)
    .filter(str -> !str.isBlank())
    .ifPresent(this::process);
```

---

## Null Handling

### NullPointerException Patterns

```java
// BAD - NPE waiting to happen
public void process(User user) {
    String name = user.getName().toUpperCase();
    // What if user is null? What if getName() returns null?
}

// GOOD - defensive
public void process(User user) {
    Objects.requireNonNull(user, "User cannot be null");

    String name = user.getName();
    if (name != null) {
        String upper = name.toUpperCase();
    }
}

// BETTER - use Optional
public void process(User user) {
    Optional.ofNullable(user)
        .map(User::getName)
        .map(String::toUpperCase)
        .ifPresent(this::handleName);
}

// Return empty collections instead of null
// BAD
public List<User> findUsers() {
    if (noResults) return null;  // Caller must check for null!
}

// GOOD
public List<User> findUsers() {
    if (noResults) return Collections.emptyList();
}
```

### Optional Misuse

```java
// BAD - Optional as field
public class User {
    private Optional<String> middleName;  // Don't do this!
}

// GOOD - return Optional from method
public class User {
    private String middleName;  // Can be null

    public Optional<String> getMiddleName() {
        return Optional.ofNullable(middleName);
    }
}

// BAD - Optional.get() without check
Optional<User> user = findUser(id);
String name = user.get().getName();  // NoSuchElementException if empty!

// GOOD - proper Optional usage
String name = findUser(id)
    .map(User::getName)
    .orElse("Unknown");

// BAD - isPresent + get
if (optional.isPresent()) {
    process(optional.get());
}

// GOOD - ifPresent
optional.ifPresent(this::process);
```

---

## Collections

### ConcurrentModificationException

```java
// BAD - modifying while iterating
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String item : list) {
    if (item.equals("b")) {
        list.remove(item);  // ConcurrentModificationException!
    }
}

// GOOD - use Iterator.remove()
Iterator<String> iter = list.iterator();
while (iter.hasNext()) {
    if (iter.next().equals("b")) {
        iter.remove();
    }
}

// BETTER - use removeIf (Java 8+)
list.removeIf(item -> item.equals("b"));

// For concurrent access - use concurrent collections
List<String> concurrentList = new CopyOnWriteArrayList<>();
```

### Mutable Keys in HashMap

```java
// BAD - mutable object as key
public class Person {
    private String name;  // Mutable!

    public void setName(String name) { this.name = name; }

    @Override
    public int hashCode() { return name.hashCode(); }
    // ...
}

Map<Person, String> map = new HashMap<>();
Person key = new Person("John");
map.put(key, "value");

key.setName("Jane");  // Changed!
map.get(key);         // null! (different bucket now)

// GOOD - use immutable objects as keys
// Or make Person immutable (record in Java 16+)
public record Person(String name) { }
```

### Wrong equals/hashCode

```java
// BAD - only equals, no hashCode
public class User {
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(id, user.id);
    }

    // Missing hashCode! HashMap/HashSet won't work correctly!
}

// GOOD - both equals and hashCode
@Override
public int hashCode() {
    return Objects.hash(id);
}

// Contract:
// 1. If a.equals(b), then a.hashCode() == b.hashCode()
// 2. If hashCodes differ, objects are not equal
// 3. hashCode should be consistent
```

### Arrays.asList Pitfalls

```java
// Arrays.asList returns fixed-size list
List<String> list = Arrays.asList("a", "b", "c");
list.set(0, "x");  // OK - modification allowed
list.add("d");     // UnsupportedOperationException!
list.remove(0);    // UnsupportedOperationException!

// Backed by original array
String[] array = {"a", "b", "c"};
List<String> list = Arrays.asList(array);
array[0] = "x";
System.out.println(list.get(0));  // "x" - list changed!

// Use for truly mutable list
List<String> mutableList = new ArrayList<>(Arrays.asList("a", "b", "c"));

// Or List.of for immutable (Java 9+)
List<String> immutable = List.of("a", "b", "c");
```

---

## Concurrency

### Thread Safety Issues

```java
// BAD - not thread-safe
public class Counter {
    private int count = 0;

    public void increment() {
        count++;  // Not atomic! Race condition!
    }
}

// GOOD - use AtomicInteger
public class Counter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }
}

// Or synchronized
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }
}
```

### Double-Checked Locking (Wrong)

```java
// BAD - broken without volatile
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  // Can see partially constructed!
                }
            }
        }
        return instance;
    }
}

// GOOD - with volatile
public class Singleton {
    private static volatile Singleton instance;
    // ... same code
}

// BETTER - enum singleton
public enum Singleton {
    INSTANCE;
}

// Or static holder (lazy, thread-safe)
public class Singleton {
    private Singleton() { }

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

### Not Closing ExecutorService

```java
// BAD - executor never shut down
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(task);
// JVM may not exit! Threads keep running

// GOOD - proper shutdown
ExecutorService executor = Executors.newFixedThreadPool(10);
try {
    executor.submit(task);
} finally {
    executor.shutdown();
    try {
        if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
            executor.shutdownNow();
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
        Thread.currentThread().interrupt();
    }
}

// Java 19+ - try-with-resources
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(task);
}
```

---

## Resource Management

### Not Closing Resources

```java
// BAD - resource leak
public void readFile(String path) throws IOException {
    InputStream is = new FileInputStream(path);
    // Exception here = stream never closed!
    byte[] data = is.readAllBytes();
    is.close();  // May not be reached
}

// GOOD - try-with-resources
public void readFile(String path) throws IOException {
    try (InputStream is = new FileInputStream(path)) {
        byte[] data = is.readAllBytes();
    }  // Automatically closed
}

// Multiple resources
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {
    // ...
}  // All closed in reverse order
```

### Connection Pool Leaks

```java
// BAD - connection not returned to pool
public User findUser(long id) {
    Connection conn = dataSource.getConnection();
    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    stmt.setLong(1, id);
    ResultSet rs = stmt.executeQuery();
    // Exception here = connection leaked!
    if (rs.next()) {
        return mapUser(rs);
    }
    conn.close();  // May not be reached
    return null;
}

// GOOD - always close in finally or try-with-resources
public User findUser(long id) throws SQLException {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
        stmt.setLong(1, id);
        try (ResultSet rs = stmt.executeQuery()) {
            if (rs.next()) {
                return mapUser(rs);
            }
        }
    }
    return null;
}
```

---

## Exception Handling

### Catching Generic Exception

```java
// BAD - catches everything including programming errors
try {
    process(data);
} catch (Exception e) {
    log.error("Failed", e);
}

// GOOD - catch specific exceptions
try {
    process(data);
} catch (IOException e) {
    log.error("IO error", e);
    // Handle IO specifically
} catch (ValidationException e) {
    log.warn("Validation failed: {}", e.getMessage());
    // Handle validation specifically
}
```

### Swallowing Exceptions

```java
// BAD - exception silently ignored
try {
    riskyOperation();
} catch (Exception e) {
    // Nothing! Silent failure
}

// BAD - only logging, not handling
try {
    riskyOperation();
} catch (Exception e) {
    log.error("Error", e);  // Then what? Program continues in bad state
}

// GOOD - handle or propagate
try {
    riskyOperation();
} catch (IOException e) {
    throw new ServiceException("Operation failed", e);  // Wrap and throw
}
```

### Losing Exception Information

```java
// BAD - losing original exception
try {
    parseData(input);
} catch (ParseException e) {
    throw new RuntimeException("Parse failed");  // Original exception lost!
}

// GOOD - chain exceptions
try {
    parseData(input);
} catch (ParseException e) {
    throw new RuntimeException("Parse failed", e);  // Original preserved
}
```

---

## Performance

### Autoboxing in Loops

```java
// BAD - autoboxing creates objects
Long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i;  // Autoboxing every iteration!
}

// GOOD - use primitives
long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i;
}
```

### Inefficient Contains Check

```java
// BAD - O(n) for each check
List<String> list = new ArrayList<>(hugeList);
for (String item : items) {
    if (list.contains(item)) {  // O(n)!
        // ...
    }
}

// GOOD - O(1) lookup
Set<String> set = new HashSet<>(hugeList);
for (String item : items) {
    if (set.contains(item)) {  // O(1)
        // ...
    }
}
```

### Creating Objects in Loops

```java
// BAD - new SimpleDateFormat every iteration
for (Date date : dates) {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");  // Expensive!
    strings.add(sdf.format(date));
}

// GOOD - reuse formatter
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");  // Once
for (Date date : dates) {
    strings.add(sdf.format(date));
}

// BEST - use DateTimeFormatter (thread-safe)
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
// Can be static final
```

---

## API Design

### Returning Null from Collections

```java
// BAD - caller must check for null
public List<User> findUsers() {
    if (noResults) {
        return null;
    }
    return results;
}

// GOOD - return empty collection
public List<User> findUsers() {
    if (noResults) {
        return Collections.emptyList();
    }
    return results;
}
```

### Not Using Defensive Copies

```java
// BAD - exposes internal state
public class Order {
    private List<Item> items;

    public List<Item> getItems() {
        return items;  // Caller can modify!
    }
}

// GOOD - return copy or unmodifiable view
public class Order {
    private List<Item> items;

    public List<Item> getItems() {
        return Collections.unmodifiableList(items);
        // or: return new ArrayList<>(items);
    }
}
```

### Boolean Parameters

```java
// BAD - unclear at call site
process(data, true, false);  // What do true/false mean?

// GOOD - use enums or builder
process(data, ProcessMode.STRICT, LogLevel.QUIET);

// Or builder pattern
processor.forData(data)
    .strict()
    .quiet()
    .process();
```

---

## На интервью

### Типичные вопросы

1. **Common memory leaks in Java?**
   - Static collections
   - Unclosed resources
   - Inner class references
   - ThreadLocal not removed
   - Listeners not unregistered

2. **Why is string concatenation in loop bad?**
   - Creates new String each iteration
   - O(n²) complexity
   - Use StringBuilder instead

3. **How to make class thread-safe?**
   - Immutable state
   - synchronized methods/blocks
   - volatile for visibility
   - Atomic classes
   - Concurrent collections

4. **equals/hashCode contract?**
   - If a.equals(b), hashCodes must be equal
   - Both must use same fields
   - Must be consistent

5. **Why use Optional?**
   - Explicit "may be empty" in API
   - Forces handling of absence
   - Don't use for fields/parameters

6. **Common exception handling mistakes?**
   - Catching generic Exception
   - Swallowing exceptions
   - Losing original exception
   - Not using try-with-resources

---

[← Назад к списку тем](README.md)
