# 08. Modern Java (9-21)

[← Назад к списку тем](README.md)

---

## Java 9

### Modules (Project Jigsaw)

```java
// module-info.java
module com.myapp {
    requires java.sql;              // Dependency on module
    requires transitive java.logging;  // Transitive dependency

    exports com.myapp.api;          // Export package
    exports com.myapp.internal to com.myapp.impl;  // Qualified export

    opens com.myapp.model to com.fasterxml.jackson.databind;  // For reflection

    provides com.myapp.spi.Plugin   // Service provider
        with com.myapp.impl.PluginImpl;

    uses com.myapp.spi.Plugin;      // Service consumer
}

// Benefits:
// - Strong encapsulation
// - Reliable configuration
// - Smaller runtime (jlink)
```

### Collection Factory Methods

```java
// Immutable collections
List<String> list = List.of("a", "b", "c");
Set<String> set = Set.of("a", "b", "c");
Map<String, Integer> map = Map.of("a", 1, "b", 2);

// Map with more than 10 entries
Map<String, Integer> bigMap = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2),
    // ...
);

// Immutable - throws UnsupportedOperationException on modification
// list.add("d");  // UnsupportedOperationException!

// Null not allowed
// List.of("a", null);  // NullPointerException!
```

### Stream Enhancements

```java
// takeWhile - take elements while predicate is true
Stream.of(1, 2, 3, 4, 5, 1, 2)
    .takeWhile(n -> n < 4)  // 1, 2, 3

// dropWhile - drop elements while predicate is true
Stream.of(1, 2, 3, 4, 5, 1, 2)
    .dropWhile(n -> n < 4)  // 4, 5, 1, 2

// iterate with predicate
Stream.iterate(1, n -> n < 100, n -> n * 2)  // 1, 2, 4, 8, 16, 32, 64

// ofNullable
Stream.ofNullable(getNullableValue())  // Empty stream if null
```

### Optional Enhancements

```java
Optional<String> opt = Optional.of("value");

// ifPresentOrElse
opt.ifPresentOrElse(
    value -> System.out.println(value),
    () -> System.out.println("Empty")
);

// or - lazy alternative
Optional<String> result = opt.or(() -> Optional.of("default"));

// stream
opt.stream().forEach(System.out::println);
```

### Private Interface Methods

```java
public interface MyInterface {
    default void method1() {
        commonLogic();
    }

    default void method2() {
        commonLogic();
    }

    // Private method to share code between default methods
    private void commonLogic() {
        System.out.println("Common logic");
    }
}
```

---

## Java 10

### Local Variable Type Inference (var)

```java
// var for local variables
var list = new ArrayList<String>();  // ArrayList<String>
var map = new HashMap<String, Integer>();  // HashMap<String, Integer>
var stream = list.stream();  // Stream<String>

// Works with:
var x = 10;                    // int
var s = "hello";               // String
var obj = new MyClass();       // MyClass

// Does NOT work with:
// var x;                      // Error: cannot infer type
// var x = null;               // Error: cannot infer type
// var x = () -> {};           // Error: lambda needs target type
// var x = {1, 2, 3};          // Error: array initializer needs type

// Cannot be used for:
// - Fields
// - Method parameters
// - Return types

// Best practices:
// Good - type is obvious from RHS
var users = new ArrayList<User>();
var response = httpClient.send(request);

// Avoid - type is not clear
var result = process();  // What is result type?
var x = getValue();      // Unclear
```

### Unmodifiable Collection Copies

```java
List<String> original = new ArrayList<>();
original.add("a");
original.add("b");

// Create unmodifiable copy
List<String> copy = List.copyOf(original);

// Also for Set and Map
Set<String> setCopy = Set.copyOf(Set.of("a", "b"));
Map<String, Integer> mapCopy = Map.copyOf(Map.of("a", 1));

// Collectors.toUnmodifiableList/Set/Map
List<String> immutable = original.stream()
    .collect(Collectors.toUnmodifiableList());
```

---

## Java 11

### String Enhancements

```java
String s = "  hello  ";

// New methods
s.isBlank();          // false
"   ".isBlank();      // true
s.strip();            // "hello" (Unicode-aware trim)
s.stripLeading();     // "hello  "
s.stripTrailing();    // "  hello"

"hello".repeat(3);    // "hellohellohello"

"line1\nline2\nline3".lines()  // Stream<String>
    .forEach(System.out::println);
```

### Files Enhancements

```java
// Read/write string directly
String content = Files.readString(Path.of("file.txt"));
Files.writeString(Path.of("file.txt"), "content");

// With charset
String content = Files.readString(path, StandardCharsets.UTF_8);
```

### HttpClient (Standard)

```java
// Create client
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .build();

// Synchronous request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .header("Content-Type", "application/json")
    .GET()
    .build();

HttpResponse<String> response = client.send(
    request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());

// Asynchronous request
CompletableFuture<HttpResponse<String>> future = client.sendAsync(
    request, HttpResponse.BodyHandlers.ofString());

future.thenApply(HttpResponse::body)
    .thenAccept(System.out::println);

// POST with body
HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/data"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"test\"}"))
    .build();
```

### Running Single-File Programs

```bash
# No need to compile first
java HelloWorld.java
```

---

## Java 14-16

### Switch Expressions (Java 14)

```java
// Old switch
String result;
switch (day) {
    case MONDAY:
    case FRIDAY:
        result = "Work hard";
        break;
    case SATURDAY:
    case SUNDAY:
        result = "Rest";
        break;
    default:
        result = "Normal";
}

// New switch expression
String result = switch (day) {
    case MONDAY, FRIDAY -> "Work hard";
    case SATURDAY, SUNDAY -> "Rest";
    default -> "Normal";
};

// With blocks and yield
String result = switch (day) {
    case MONDAY -> {
        System.out.println("Monday!");
        yield "Start of week";
    }
    case FRIDAY -> "Almost weekend";
    default -> "Normal day";
};

// Exhaustiveness - must cover all cases for enums
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }
String type = switch (day) {
    case MON, TUE, WED, THU, FRI -> "Weekday";
    case SAT, SUN -> "Weekend";
    // No default needed - all cases covered
};
```

### Text Blocks (Java 15)

```java
// Old way
String json = "{\n" +
    "  \"name\": \"John\",\n" +
    "  \"age\": 30\n" +
    "}";

// Text blocks
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;

// HTML
String html = """
    <html>
        <body>
            <h1>Hello</h1>
        </body>
    </html>
    """;

// SQL
String sql = """
    SELECT id, name, email
    FROM users
    WHERE status = 'ACTIVE'
    ORDER BY name
    """;

// Trailing whitespace: use \s
String text = """
    line with trailing space   \s
    another line
    """;

// No newline at end: use \
String single = """
    This is \
    one line\
    """;
```

### Records (Java 16)

```java
// Old way - boilerplate
public final class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int x() { return x; }
    public int y() { return y; }

    @Override
    public boolean equals(Object o) { ... }
    @Override
    public int hashCode() { ... }
    @Override
    public String toString() { ... }
}

// Record - one line!
public record Point(int x, int y) { }

// Usage
Point p = new Point(10, 20);
int x = p.x();  // Accessor method (not getX!)
int y = p.y();

// Custom constructor (validation)
public record Person(String name, int age) {
    // Compact constructor
    public Person {
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
        name = name.trim();  // Can modify parameters
    }
}

// Additional methods
public record Rectangle(int width, int height) {
    public int area() {
        return width * height;
    }

    // Static factory
    public static Rectangle square(int size) {
        return new Rectangle(size, size);
    }
}

// Records are:
// - Implicitly final
// - Cannot extend other classes (but can implement interfaces)
// - All fields are final
// - Automatically get: constructor, accessors, equals, hashCode, toString
```

### Pattern Matching for instanceof (Java 16)

```java
// Old way
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Pattern matching
if (obj instanceof String s) {
    System.out.println(s.length());  // s is already String
}

// With conditions
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s);
}

// In else branch - variable NOT in scope
if (!(obj instanceof String s)) {
    return;
}
// s is in scope here (flow scoping)
System.out.println(s.length());
```

---

## Java 17 (LTS)

### Sealed Classes

```java
// Sealed class - restricts which classes can extend it
public sealed class Shape
    permits Circle, Rectangle, Triangle {
}

// Permitted subclasses must be: final, sealed, or non-sealed

public final class Circle extends Shape {
    private final double radius;
    // ...
}

public final class Rectangle extends Shape {
    private final double width, height;
    // ...
}

public non-sealed class Triangle extends Shape {
    // Can be extended by anyone
}

// Sealed interfaces
public sealed interface Vehicle
    permits Car, Truck, Motorcycle {
}

public final class Car implements Vehicle { }
public final class Truck implements Vehicle { }
public final class Motorcycle implements Vehicle { }

// Benefits:
// - Exhaustive pattern matching
// - Clear hierarchy
// - Better encapsulation
```

### Pattern Matching for switch (Preview → Java 21)

```java
// Type patterns in switch
String format(Object obj) {
    return switch (obj) {
        case Integer i -> "Integer: " + i;
        case Long l -> "Long: " + l;
        case Double d -> "Double: " + d;
        case String s -> "String: " + s;
        case null -> "null";
        default -> "Unknown: " + obj;
    };
}

// With sealed classes - exhaustive
String describe(Shape shape) {
    return switch (shape) {
        case Circle c -> "Circle with radius " + c.radius();
        case Rectangle r -> "Rectangle " + r.width() + "x" + r.height();
        case Triangle t -> "Triangle";
        // No default needed - all cases covered!
    };
}

// Guarded patterns
String describe(Object obj) {
    return switch (obj) {
        case String s when s.isEmpty() -> "Empty string";
        case String s -> "String: " + s;
        case Integer i when i > 0 -> "Positive: " + i;
        case Integer i when i < 0 -> "Negative: " + i;
        case Integer i -> "Zero";
        default -> "Other";
    };
}
```

---

## Java 21 (LTS)

### Record Patterns

```java
record Point(int x, int y) { }
record Circle(Point center, int radius) { }

// Nested record pattern
void process(Object obj) {
    if (obj instanceof Circle(Point(var x, var y), var r)) {
        System.out.println("Circle at (" + x + "," + y + ") radius " + r);
    }
}

// In switch
String describe(Object obj) {
    return switch (obj) {
        case Circle(Point(var x, var y), var r)
            when r > 10 -> "Large circle at " + x + "," + y;
        case Circle(Point(var x, var y), var r) ->
            "Circle at " + x + "," + y;
        default -> "Not a circle";
    };
}
```

### Virtual Threads (Project Loom)

```java
// Old way - platform threads (expensive)
Thread platformThread = new Thread(() -> {
    // task
});
platformThread.start();

// Virtual threads - lightweight
Thread virtualThread = Thread.ofVirtual().start(() -> {
    // task
});

// Virtual thread factory
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // Each task gets its own virtual thread
        return fetchData();
    });
}

// Can create millions of virtual threads!
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 1_000_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}

// Benefits:
// - Lightweight (~KB vs ~MB for platform threads)
// - Cheap to create and block
// - Same Thread API
// - Great for I/O-bound tasks

// Platform threads still better for:
// - CPU-intensive tasks
// - When you need thread-local pinning
```

### Sequenced Collections

```java
// New interfaces for ordered collections
interface SequencedCollection<E> extends Collection<E> {
    SequencedCollection<E> reversed();
    void addFirst(E e);
    void addLast(E e);
    E getFirst();
    E getLast();
    E removeFirst();
    E removeLast();
}

// Usage
List<String> list = new ArrayList<>();
list.addFirst("first");
list.addLast("last");
String first = list.getFirst();
String last = list.getLast();

// Reversed view
List<String> reversed = list.reversed();

// Also SequencedSet, SequencedMap
LinkedHashSet<String> set = new LinkedHashSet<>();
set.addFirst("a");
String last = set.getLast();

LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
var firstEntry = map.firstEntry();
var lastEntry = map.lastEntry();
```

### String Templates (Preview - Removed)

```java
// ВНИМАНИЕ: String Templates (STR, FMT) были удалены в Java 23!
// Эта функция была preview в Java 21-22, но была удалена из-за проблем с дизайном.
// Ниже приведён пример того, как это работало в preview-версиях:

// String interpolation (preview in Java 21-22, REMOVED in Java 23)
String name = "World";
// String greeting = STR."Hello, \{name}!";  // Не работает в Java 23+

// Вместо этого используйте String.format() или MessageFormat:
String greeting = String.format("Hello, %s!", name);

// Или StringBuilder / конкатенацию:
String greeting2 = "Hello, " + name + "!";
```

---

## См. также

- [Streams & Lambdas](./07-streams-lambdas.md) — функциональное программирование в Java 8+
- [JVM Fundamentals](./00-jvm-fundamentals.md) — основы платформы JVM

---

## На интервью

### Типичные вопросы

1. **What are Records?**
   - Immutable data carriers
   - Auto-generated: constructor, accessors, equals, hashCode, toString
   - Cannot extend classes, can implement interfaces
   - All fields are final

2. **What are Sealed Classes?**
   - Restrict which classes can extend
   - Subclasses must be: final, sealed, or non-sealed
   - Enables exhaustive pattern matching

3. **var keyword rules?**
   - Local variables only
   - Must have initializer
   - Cannot be null, lambda, array initializer
   - Type is inferred at compile time

4. **Virtual threads vs Platform threads?**
   - Virtual: lightweight, cheap blocking, for I/O
   - Platform: heavyweight, for CPU-intensive
   - Virtual threads managed by JVM, not OS

5. **Text blocks features?**
   - Triple quotes """
   - Automatic indentation stripping
   - \s for trailing space, \ for line continuation

6. **Switch expressions vs statements?**
   - Expression returns value
   - Arrow syntax, no fall-through
   - yield for multi-statement blocks
   - Must be exhaustive for return

---

[← Назад к списку тем](README.md)
