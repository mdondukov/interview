# 09. Exceptions & Error Handling

[← Назад к списку тем](README.md)

---

## Exception Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Exception Hierarchy                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                         Throwable                                   │
│                        /         \                                  │
│                       /           \                                 │
│                    Error         Exception                          │
│                      │               │                              │
│                      │          ┌────┴────┐                         │
│                      │          │         │                         │
│              OutOfMemoryError   │    RuntimeException               │
│              StackOverflowError │         │                         │
│              NoClassDefFoundError│         │                        │
│                                 │    NullPointerException           │
│                        IOException  IllegalArgumentException        │
│                        SQLException ArrayIndexOutOfBoundsException  │
│                        ClassNotFoundException  ClassCastException   │
│                                                                     │
│  Checked Exceptions:                Unchecked Exceptions:           │
│  - Must be declared/caught          - RuntimeException + subclasses │
│  - IOException, SQLException        - No need to declare            │
│  - Recoverable situations           - Programming errors            │
│                                                                     │
│  Errors:                                                            │
│  - Serious problems                                                 │
│  - Should not be caught             (usually)                       │
│  - JVM issues                                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Checked vs Unchecked

```java
// Checked Exception - must handle or declare
public void readFile(String path) throws IOException {  // Must declare
    FileInputStream fis = new FileInputStream(path);  // Can throw IOException
}

// Calling code must handle
public void caller() {
    try {
        readFile("test.txt");
    } catch (IOException e) {
        // Handle exception
    }
}

// Or propagate
public void caller() throws IOException {
    readFile("test.txt");
}

// Unchecked Exception - no need to declare
public void validate(String value) {
    if (value == null) {
        throw new IllegalArgumentException("Value cannot be null");
    }
}

// Can call without try-catch
public void caller() {
    validate(null);  // May throw, but no need to handle at compile time
}
```

---

## Try-Catch-Finally

### Basic Syntax

```java
try {
    // Code that may throw exception
    riskyOperation();
} catch (IOException e) {
    // Handle IOException
    logger.error("IO error", e);
} catch (SQLException e) {
    // Handle SQLException
    logger.error("DB error", e);
} finally {
    // Always executed (even if exception thrown)
    cleanup();
}
```

### Multi-Catch (Java 7+)

```java
try {
    riskyOperation();
} catch (IOException | SQLException e) {
    // Handle both exceptions the same way
    logger.error("Error: " + e.getMessage());
}

// Note: variable 'e' is implicitly final in multi-catch
```

### Try-with-Resources (Java 7+)

```java
// Old way - verbose and error-prone
InputStream is = null;
try {
    is = new FileInputStream("file.txt");
    // use stream
} catch (IOException e) {
    // handle
} finally {
    if (is != null) {
        try {
            is.close();
        } catch (IOException e) {
            // ignore or log
        }
    }
}

// Try-with-resources - automatic close
try (InputStream is = new FileInputStream("file.txt")) {
    // use stream
} catch (IOException e) {
    // handle
}
// is.close() called automatically!

// Multiple resources
try (
    InputStream is = new FileInputStream("input.txt");
    OutputStream os = new FileOutputStream("output.txt")
) {
    // use streams
}
// Both closed in reverse order

// Works with any AutoCloseable
public interface AutoCloseable {
    void close() throws Exception;
}

// Java 9+ - effectively final variables
InputStream is = new FileInputStream("file.txt");
try (is) {  // Can use existing variable if effectively final
    // use stream
}
```

### Suppressed Exceptions

```java
// When both try block and close() throw exceptions
try (MyResource resource = new MyResource()) {
    throw new RuntimeException("From try");
} // close() also throws exception

// The exception from close() is suppressed
catch (RuntimeException e) {
    System.out.println(e.getMessage());  // "From try"

    Throwable[] suppressed = e.getSuppressed();
    // Contains exception from close()
}
```

---

## Creating Custom Exceptions

### Checked Exception

```java
public class BusinessException extends Exception {

    private final String errorCode;

    public BusinessException(String message) {
        super(message);
        this.errorCode = "UNKNOWN";
    }

    public BusinessException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public BusinessException(String message, Throwable cause) {
        super(message, cause);
        this.errorCode = "UNKNOWN";
    }

    public BusinessException(String message, String errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}

// Usage
public void process(Order order) throws BusinessException {
    if (order.getAmount() <= 0) {
        throw new BusinessException("Invalid amount", "ORDER_001");
    }
}
```

### Unchecked Exception

```java
public class ValidationException extends RuntimeException {

    private final List<String> errors;

    public ValidationException(String message) {
        super(message);
        this.errors = List.of(message);
    }

    public ValidationException(List<String> errors) {
        super(String.join(", ", errors));
        this.errors = new ArrayList<>(errors);
    }

    public List<String> getErrors() {
        return Collections.unmodifiableList(errors);
    }
}

// Usage - no need to declare
public void validate(User user) {
    List<String> errors = new ArrayList<>();

    if (user.getName() == null) {
        errors.add("Name is required");
    }
    if (user.getEmail() == null) {
        errors.add("Email is required");
    }

    if (!errors.isEmpty()) {
        throw new ValidationException(errors);
    }
}
```

---

## Exception Best Practices

### Do's

```java
// 1. Use specific exceptions
public void processFile(String path) throws FileNotFoundException {
    // Not: throws Exception
}

// 2. Include context in message
throw new IllegalArgumentException(
    "Invalid age: " + age + ". Must be between 0 and 150"
);

// 3. Chain exceptions (preserve root cause)
try {
    parseData(data);
} catch (ParseException e) {
    throw new BusinessException("Failed to parse data", e);  // Chain!
}

// 4. Log at appropriate level
try {
    riskyOperation();
} catch (Exception e) {
    logger.error("Operation failed for user {}: {}", userId, e.getMessage(), e);
    throw e;  // Re-throw if can't handle
}

// 5. Use try-with-resources for AutoCloseable
try (var connection = dataSource.getConnection();
     var statement = connection.prepareStatement(sql)) {
    // ...
}

// 6. Catch most specific exception first
try {
    // ...
} catch (FileNotFoundException e) {
    // Handle file not found specifically
} catch (IOException e) {
    // Handle other IO errors
}
```

### Don'ts

```java
// 1. DON'T catch and ignore
try {
    riskyOperation();
} catch (Exception e) {
    // Empty! BAD!
}

// 2. DON'T catch generic Exception (usually)
try {
    // ...
} catch (Exception e) {  // Too broad!
    // ...
}

// 3. DON'T use exceptions for flow control
// BAD:
try {
    while (true) {
        array[i++] = getValue();
    }
} catch (ArrayIndexOutOfBoundsException e) {
    // Expected exit
}

// GOOD:
for (int i = 0; i < array.length; i++) {
    array[i] = getValue();
}

// 4. DON'T throw in finally
try {
    // ...
} finally {
    throw new RuntimeException();  // BAD! Hides original exception
}

// 5. DON'T log and throw
try {
    riskyOperation();
} catch (Exception e) {
    logger.error("Failed", e);
    throw e;  // Leads to double logging
}
// Either log OR throw, not both

// 6. DON'T catch Throwable/Error (usually)
try {
    // ...
} catch (Throwable t) {  // Catches Error too - BAD!
    // ...
}
```

### Exception Translation

```java
// Convert low-level exceptions to domain exceptions
public User findUser(long id) throws UserNotFoundException {
    try {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    } catch (SQLException e) {
        // Translate to domain exception
        throw new DataAccessException("Failed to find user: " + id, e);
    }
}
```

---

## Common Runtime Exceptions

```java
// NullPointerException
String s = null;
s.length();  // NPE!

// IllegalArgumentException
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative: " + age);
    }
}

// IllegalStateException
public void start() {
    if (running) {
        throw new IllegalStateException("Already running");
    }
}

// IndexOutOfBoundsException
List<String> list = List.of("a", "b");
list.get(5);  // IndexOutOfBoundsException

// ClassCastException
Object obj = "string";
Integer num = (Integer) obj;  // ClassCastException

// NumberFormatException
Integer.parseInt("not a number");  // NumberFormatException

// UnsupportedOperationException
List<String> immutable = List.of("a", "b");
immutable.add("c");  // UnsupportedOperationException

// ConcurrentModificationException
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    list.remove(s);  // ConcurrentModificationException!
}
```

---

## Null Handling

### Defensive Programming

```java
// Check parameters at method entry
public void process(String value) {
    Objects.requireNonNull(value, "Value cannot be null");
    // ...
}

// With message supplier (lazy)
Objects.requireNonNull(value, () -> "Value for " + key + " cannot be null");

// Validation
public void setName(String name) {
    if (name == null || name.isBlank()) {
        throw new IllegalArgumentException("Name is required");
    }
    this.name = name;
}
```

### Optional Instead of Null

```java
// Return Optional instead of null
public Optional<User> findUser(long id) {
    User user = userMap.get(id);
    return Optional.ofNullable(user);
}

// Usage
findUser(id)
    .map(User::getName)
    .orElse("Unknown");

// Don't use Optional for:
// - Fields (use null)
// - Parameters (use overloads)
// - Collections (return empty collection)
```

---

## Exception in Lambdas

```java
// Problem: checked exceptions in lambdas
list.stream()
    .map(item -> parseItem(item))  // Compile error if parseItem throws checked exception
    .collect(toList());

// Solution 1: Wrap in runtime exception
list.stream()
    .map(item -> {
        try {
            return parseItem(item);
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    })
    .collect(toList());

// Solution 2: Utility method
@FunctionalInterface
public interface ThrowingFunction<T, R, E extends Exception> {
    R apply(T t) throws E;

    static <T, R, E extends Exception> Function<T, R> unchecked(
            ThrowingFunction<T, R, E> f) {
        return t -> {
            try {
                return f.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

// Usage
list.stream()
    .map(ThrowingFunction.unchecked(this::parseItem))
    .collect(toList());

// Solution 3: Sneaky throws (not recommended)
@SuppressWarnings("unchecked")
static <E extends Throwable> void sneakyThrow(Throwable e) throws E {
    throw (E) e;
}
```

---

## Assert

```java
// Assertions - disabled by default, enable with -ea flag
public void process(List<String> items) {
    assert items != null : "Items cannot be null";
    assert !items.isEmpty() : "Items cannot be empty";

    // ... process items
}

// DON'T use for:
// - Input validation (use exceptions)
// - Production checks

// DO use for:
// - Internal invariants
// - Impossible conditions
// - Post-conditions during development

// Run with assertions enabled:
// java -ea MyApp
// java -enableassertions MyApp
```

---

## На интервью

### Типичные вопросы

1. **Checked vs Unchecked exceptions?**
   - Checked: must declare/catch, recoverable (IOException)
   - Unchecked: RuntimeException, programming errors
   - Error: serious JVM problems, don't catch

2. **When to use checked vs unchecked?**
   - Checked: caller can reasonably recover
   - Unchecked: programming error, nothing caller can do

3. **finally block guarantees?**
   - Always executes (except System.exit())
   - Executes before return in try/catch
   - If both throw, finally exception "wins"

4. **try-with-resources benefits?**
   - Automatic close()
   - Handles suppressed exceptions
   - Cleaner code, fewer bugs

5. **How to handle exceptions in lambdas?**
   - Wrap in try-catch
   - Convert to RuntimeException
   - Use wrapper functional interfaces

6. **Exception best practices?**
   - Don't catch and ignore
   - Chain exceptions (preserve cause)
   - Use specific exception types
   - Include context in messages

---

[← Назад к списку тем](README.md)
