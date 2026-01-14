# 07. Streams & Lambdas

[← Назад к списку тем](README.md)

---

## Lambda Expressions

### Syntax

```java
// Lambda = anonymous function
// (parameters) -> expression
// (parameters) -> { statements; }

// No parameters
Runnable r = () -> System.out.println("Hello");

// One parameter (parentheses optional)
Consumer<String> c = s -> System.out.println(s);
Consumer<String> c2 = (s) -> System.out.println(s);

// Multiple parameters
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// With types (usually inferred)
BiFunction<Integer, Integer, Integer> add2 = (Integer a, Integer b) -> a + b;

// Multiple statements
BiFunction<Integer, Integer, Integer> calc = (a, b) -> {
    int result = a + b;
    return result * 2;
};
```

### Functional Interfaces

```java
// Interface with single abstract method (SAM)
@FunctionalInterface
public interface MyFunction {
    int apply(int x);

    // Can have default methods
    default int applyTwice(int x) {
        return apply(apply(x));
    }

    // Can have static methods
    static MyFunction identity() {
        return x -> x;
    }
}

// Built-in functional interfaces (java.util.function)

// Supplier<T> - no input, returns T
Supplier<String> supplier = () -> "Hello";
String s = supplier.get();

// Consumer<T> - takes T, returns void
Consumer<String> consumer = s -> System.out.println(s);
consumer.accept("Hello");

// Function<T, R> - takes T, returns R
Function<String, Integer> length = s -> s.length();
Integer len = length.apply("Hello");

// Predicate<T> - takes T, returns boolean
Predicate<String> isEmpty = s -> s.isEmpty();
boolean result = isEmpty.test("");

// BiFunction<T, U, R> - takes T and U, returns R
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// UnaryOperator<T> - takes T, returns T
UnaryOperator<Integer> square = x -> x * x;

// BinaryOperator<T> - takes T and T, returns T
BinaryOperator<Integer> sum = (a, b) -> a + b;
```

### Method References

```java
// Reference to static method
Function<String, Integer> parseInt = Integer::parseInt;
// Same as: s -> Integer.parseInt(s)

// Reference to instance method of object
String str = "Hello";
Supplier<Integer> length = str::length;
// Same as: () -> str.length()

// Reference to instance method of type
Function<String, Integer> len = String::length;
// Same as: s -> s.length()

// Reference to constructor
Supplier<ArrayList<String>> newList = ArrayList::new;
// Same as: () -> new ArrayList<>()

Function<Integer, ArrayList<String>> listWithSize = ArrayList::new;
// Same as: size -> new ArrayList<>(size)

// Examples
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
names.forEach(System.out::println);
names.sort(String::compareToIgnoreCase);
```

---

## Stream API

### Creating Streams

```java
// From collection
List<String> list = List.of("a", "b", "c");
Stream<String> stream1 = list.stream();
Stream<String> parallelStream = list.parallelStream();

// From array
String[] array = {"a", "b", "c"};
Stream<String> stream2 = Arrays.stream(array);

// Using Stream.of
Stream<String> stream3 = Stream.of("a", "b", "c");

// Empty stream
Stream<String> empty = Stream.empty();

// Generate infinite stream
Stream<Double> randoms = Stream.generate(Math::random);
Stream<Integer> integers = Stream.iterate(0, n -> n + 2);  // 0, 2, 4, 6...

// With limit (Java 9+)
Stream<Integer> limited = Stream.iterate(0, n -> n < 100, n -> n + 10);

// From other sources
IntStream chars = "hello".chars();
Stream<String> lines = Files.lines(Path.of("file.txt"));
Stream<String> split = Pattern.compile(",").splitAsStream("a,b,c");
```

### Intermediate Operations

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "David");

// filter - keep elements matching predicate
names.stream()
    .filter(s -> s.length() > 3)  // Alice, Charlie, David

// map - transform elements
names.stream()
    .map(String::toUpperCase)  // ALICE, BOB, CHARLIE, DAVID

// flatMap - flatten nested structures
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3, 4));
nested.stream()
    .flatMap(List::stream)  // 1, 2, 3, 4

// distinct - remove duplicates
Stream.of(1, 2, 2, 3, 3, 3)
    .distinct()  // 1, 2, 3

// sorted - sort elements
names.stream()
    .sorted()  // Alice, Bob, Charlie, David
    .sorted(Comparator.reverseOrder())  // David, Charlie, Bob, Alice
    .sorted(Comparator.comparingInt(String::length))  // Bob, Alice, David, Charlie

// peek - debug/side effects
names.stream()
    .peek(s -> System.out.println("Processing: " + s))
    .filter(s -> s.length() > 3)

// limit/skip
Stream.iterate(1, n -> n + 1)
    .skip(5)    // skip first 5
    .limit(10)  // take next 10

// takeWhile/dropWhile (Java 9+)
Stream.of(1, 2, 3, 4, 5, 1, 2)
    .takeWhile(n -> n < 4)  // 1, 2, 3
    .dropWhile(n -> n < 3)  // 3
```

### Terminal Operations

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

// forEach - perform action on each element
names.stream().forEach(System.out::println);

// collect - gather into collection
List<String> list = names.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());

Set<String> set = names.stream().collect(Collectors.toSet());

// toArray
String[] array = names.stream().toArray(String[]::new);

// reduce - combine elements
Optional<String> concat = names.stream().reduce((a, b) -> a + b);
String concat2 = names.stream().reduce("", (a, b) -> a + b);
int sum = Stream.of(1, 2, 3).reduce(0, Integer::sum);

// count
long count = names.stream().filter(s -> s.length() > 3).count();

// min/max
Optional<String> shortest = names.stream()
    .min(Comparator.comparingInt(String::length));

// findFirst/findAny
Optional<String> first = names.stream()
    .filter(s -> s.startsWith("A"))
    .findFirst();

// anyMatch/allMatch/noneMatch
boolean anyStartsWithA = names.stream().anyMatch(s -> s.startsWith("A"));
boolean allLong = names.stream().allMatch(s -> s.length() > 2);
boolean noneEmpty = names.stream().noneMatch(String::isEmpty);
```

### Collectors

```java
List<Person> people = List.of(
    new Person("Alice", 30),
    new Person("Bob", 25),
    new Person("Charlie", 30)
);

// toList, toSet, toCollection
List<String> names = people.stream()
    .map(Person::getName)
    .collect(Collectors.toList());

Set<String> nameSet = people.stream()
    .map(Person::getName)
    .collect(Collectors.toSet());

TreeSet<String> treeSet = people.stream()
    .map(Person::getName)
    .collect(Collectors.toCollection(TreeSet::new));

// toMap
Map<String, Integer> nameToAge = people.stream()
    .collect(Collectors.toMap(Person::getName, Person::getAge));

// With merge function (for duplicates)
Map<Integer, String> ageToName = people.stream()
    .collect(Collectors.toMap(
        Person::getAge,
        Person::getName,
        (name1, name2) -> name1 + ", " + name2  // merge duplicates
    ));

// groupingBy
Map<Integer, List<Person>> byAge = people.stream()
    .collect(Collectors.groupingBy(Person::getAge));

// groupingBy with downstream collector
Map<Integer, Long> countByAge = people.stream()
    .collect(Collectors.groupingBy(Person::getAge, Collectors.counting()));

Map<Integer, String> namesByAge = people.stream()
    .collect(Collectors.groupingBy(
        Person::getAge,
        Collectors.mapping(Person::getName, Collectors.joining(", "))
    ));

// partitioningBy (boolean grouping)
Map<Boolean, List<Person>> partitioned = people.stream()
    .collect(Collectors.partitioningBy(p -> p.getAge() > 25));

// joining
String allNames = people.stream()
    .map(Person::getName)
    .collect(Collectors.joining(", "));  // "Alice, Bob, Charlie"

// summarizing
IntSummaryStatistics stats = people.stream()
    .collect(Collectors.summarizingInt(Person::getAge));
// count, sum, min, average, max
```

---

## Optional

```java
// Creating Optional
Optional<String> opt1 = Optional.of("value");       // Must be non-null
Optional<String> opt2 = Optional.ofNullable(null);  // May be null
Optional<String> opt3 = Optional.empty();

// Checking presence
if (opt1.isPresent()) {
    String value = opt1.get();
}

opt1.ifPresent(System.out::println);

opt1.ifPresentOrElse(
    System.out::println,
    () -> System.out.println("Empty")
);

// Getting value
String value1 = opt1.get();                      // Throws if empty!
String value2 = opt1.orElse("default");
String value3 = opt1.orElseGet(() -> "computed");
String value4 = opt1.orElseThrow();              // NoSuchElementException
String value5 = opt1.orElseThrow(() -> new RuntimeException("Empty!"));

// Transforming
Optional<Integer> length = opt1.map(String::length);
Optional<String> upper = opt1.map(String::toUpperCase);

// flatMap for nested Optional
Optional<Optional<String>> nested = Optional.of(Optional.of("value"));
Optional<String> flat = nested.flatMap(Function.identity());

// filter
Optional<String> filtered = opt1.filter(s -> s.length() > 3);

// or (Java 9+)
Optional<String> result = opt2.or(() -> Optional.of("alternative"));

// stream (Java 9+)
Stream<String> stream = opt1.stream();
```

### Optional Best Practices

```java
// DON'T: use Optional as field
class Bad {
    private Optional<String> name;  // Bad!
}

// DO: return Optional from methods
class Good {
    private String name;

    public Optional<String> getName() {
        return Optional.ofNullable(name);
    }
}

// DON'T: Optional.of(null) or get() without check
Optional.of(null);           // NullPointerException!
optional.get();              // NoSuchElementException if empty!

// DO: use ofNullable and proper unwrapping
Optional.ofNullable(value);
optional.orElse("default");

// DON'T: use Optional for primitives
Optional<Integer> opt = Optional.of(42);  // Boxing overhead

// DO: use specialized OptionalInt, OptionalLong, OptionalDouble
OptionalInt optInt = OptionalInt.of(42);
```

---

## Primitive Streams

```java
// IntStream, LongStream, DoubleStream - avoid boxing

// Creating
IntStream ints = IntStream.of(1, 2, 3);
IntStream range = IntStream.range(0, 10);        // 0-9
IntStream rangeClosed = IntStream.rangeClosed(0, 10);  // 0-10

// From array
int[] array = {1, 2, 3};
IntStream fromArray = Arrays.stream(array);

// From Stream
IntStream intStream = Stream.of("a", "bb", "ccc")
    .mapToInt(String::length);

// Special operations
int sum = IntStream.of(1, 2, 3).sum();
OptionalDouble avg = IntStream.of(1, 2, 3).average();
OptionalInt max = IntStream.of(1, 2, 3).max();
IntSummaryStatistics stats = IntStream.of(1, 2, 3).summaryStatistics();

// Boxing back to Stream<Integer>
Stream<Integer> boxed = IntStream.of(1, 2, 3).boxed();
```

---

## См. также

- [Python Data Structures](../02-python/01-data-structures.md) — функциональные подходы в Python
- [Arrays & Hashing](../05-algorithms/01-arrays-hashing.md) — алгоритмы обработки коллекций

---

## На интервью

### Типичные вопросы

1. **Lambda vs Anonymous class?**
   - Lambda: shorter syntax, no `this` reference to enclosing class
   - Anonymous: creates new class, has `this`, can extend classes

2. **Intermediate vs Terminal operations?**
   - Intermediate: return Stream, lazy (filter, map, sorted)
   - Terminal: produce result, trigger execution (collect, forEach, reduce)

3. **Stream vs Collection?**
   - Stream: one-time use, lazy, functional
   - Collection: reusable, eager, mutable

4. **When to use Optional?**
   - Return type for methods that may return nothing
   - Not for fields, parameters, or collections

5. **flatMap vs map?**
   - map: one-to-one transformation
   - flatMap: one-to-many, flattens nested structure

6. **Parallel streams?**
   - Use for CPU-intensive, independent operations on large data
   - Avoid for I/O, small data, ordered operations

---

[← Назад к списку тем](README.md)
