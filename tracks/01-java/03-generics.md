# 03. Generics

[← Назад к списку тем](README.md)

---

## Основы Generics

```java
// Без generics (до Java 5)
List list = new ArrayList();
list.add("hello");
list.add(123);  // Компилируется!
String s = (String) list.get(0);  // Нужен cast
String s2 = (String) list.get(1); // ClassCastException в runtime!

// С generics - type safety
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(123);  // Compile error!
String s = list.get(0);  // Без cast
```

### Generic Class

```java
public class Box<T> {
    private T content;

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }
}

// Usage
Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String s = stringBox.get();

Box<Integer> intBox = new Box<>();
intBox.set(42);
Integer i = intBox.get();
```

### Generic Method

```java
public class Util {
    // Generic method - type parameter before return type
    public static <T> T getFirst(List<T> list) {
        return list.isEmpty() ? null : list.get(0);
    }

    // Multiple type parameters
    public static <K, V> V getValue(Map<K, V> map, K key) {
        return map.get(key);
    }

    // Type inference
    public static <T> List<T> arrayToList(T[] array) {
        return Arrays.asList(array);
    }
}

// Usage - type inferred
String first = Util.getFirst(List.of("a", "b", "c"));
Integer num = Util.getFirst(List.of(1, 2, 3));

// Explicit type (rarely needed)
String s = Util.<String>getFirst(list);
```

### Generic Interface

```java
public interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(T entity);
}

public class UserRepository implements Repository<User, Long> {
    @Override
    public User findById(Long id) { ... }

    @Override
    public List<User> findAll() { ... }

    @Override
    public void save(User entity) { ... }

    @Override
    public void delete(User entity) { ... }
}
```

---

## Type Erasure

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Type Erasure                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Generics существуют только в compile time!                         │
│  После компиляции все type parameters стираются                     │
│                                                                     │
│  Source code:           After erasure:                              │
│  ──────────────         ──────────────                              │
│  List<String>           List                                        │
│  Box<Integer>           Box                                         │
│  T extends Number       Number                                      │
│  T (unbounded)          Object                                      │
│                                                                     │
│  Последствия:                                                       │
│  - Нельзя создать new T()                                           │
│  - Нельзя создать new T[]                                           │
│  - Нельзя использовать instanceof с generic type                    │
│  - Нельзя создать generic static field                              │
│  - Нельзя catch/throw generic type                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Ограничения Type Erasure

```java
public class Box<T> {
    private T content;

    // COMPILE ERROR: Cannot instantiate type parameter
    public void create() {
        // T obj = new T();  // Error!
    }

    // COMPILE ERROR: Cannot create generic array
    public void createArray() {
        // T[] arr = new T[10];  // Error!
    }

    // COMPILE ERROR: instanceof with generic type
    public void check(Object obj) {
        // if (obj instanceof T) { }  // Error!
    }

    // COMPILE ERROR: static field with type parameter
    // private static T staticContent;  // Error!
}

// Workaround: Class<T> token
public class Factory<T> {
    private Class<T> type;

    public Factory(Class<T> type) {
        this.type = type;
    }

    public T create() throws Exception {
        return type.getDeclaredConstructor().newInstance();
    }
}

// Usage
Factory<String> factory = new Factory<>(String.class);
String s = factory.create();
```

### Runtime Type Information

```java
// В runtime все List<X> это просто List
List<String> strings = new ArrayList<>();
List<Integer> integers = new ArrayList<>();

// В runtime это одинаковые классы!
System.out.println(strings.getClass() == integers.getClass());  // true

// Нельзя определить generic type в runtime
public static <T> void process(List<T> list) {
    // Какой тип T? Невозможно узнать!
    // list.getClass() возвращает ArrayList, не ArrayList<String>
}
```

---

## Bounds (Ограничения типов)

### Upper Bound (extends)

```java
// T должен быть Number или его подкласс
public class NumericBox<T extends Number> {
    private T value;

    public double doubleValue() {
        return value.doubleValue();  // Можем вызывать методы Number
    }
}

NumericBox<Integer> intBox = new NumericBox<>();  // OK
NumericBox<Double> doubleBox = new NumericBox<>(); // OK
// NumericBox<String> stringBox = new NumericBox<>(); // Error!

// Multiple bounds
public class Container<T extends Comparable<T> & Serializable> {
    // T должен implements Comparable И Serializable
    // Если есть class (не interface), он должен быть первым
}
```

### Bounded Generic Methods

```java
// Только для Comparable типов
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

// Usage
String maxStr = max("apple", "banana");  // "banana"
Integer maxInt = max(10, 20);            // 20
```

---

## Wildcards

### Unbounded Wildcard (?)

```java
// Любой тип
public void printList(List<?> list) {
    for (Object o : list) {
        System.out.println(o);
    }
}

// Можно передать любой List
printList(List.of("a", "b"));
printList(List.of(1, 2, 3));
printList(List.of(new Object()));

// Но нельзя добавлять (кроме null)
public void addToList(List<?> list) {
    // list.add("hello");  // Error! Не знаем тип
    list.add(null);        // OK, null подходит любому типу
}
```

### Upper Bounded Wildcard (? extends T)

```java
// Принимает List<Number>, List<Integer>, List<Double>, etc.
public double sum(List<? extends Number> numbers) {
    double sum = 0;
    for (Number n : numbers) {
        sum += n.doubleValue();
    }
    return sum;
}

sum(List.of(1, 2, 3));           // List<Integer>
sum(List.of(1.0, 2.0, 3.0));     // List<Double>

// PRODUCER EXTENDS - можно читать как T
// Нельзя добавлять (кроме null)
List<? extends Number> nums = new ArrayList<Integer>();
Number n = nums.get(0);  // OK - читаем как Number
// nums.add(1);          // Error! Может быть List<Double>
```

### Lower Bounded Wildcard (? super T)

```java
// Принимает List<Integer>, List<Number>, List<Object>
public void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    list.add(3);
}

List<Number> numbers = new ArrayList<>();
addNumbers(numbers);  // OK

List<Object> objects = new ArrayList<>();
addNumbers(objects);  // OK

// CONSUMER SUPER - можно добавлять T
// Читаем только как Object
List<? super Integer> list = new ArrayList<Number>();
list.add(1);          // OK - добавляем Integer
Object o = list.get(0); // Только Object, не Integer
```

### PECS (Producer Extends, Consumer Super)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PECS Rule                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PRODUCER EXTENDS:                                                  │
│  Если нужно ЧИТАТЬ из коллекции → ? extends T                       │
│  List<? extends Number> → можно get() как Number                    │
│                                                                     │
│  CONSUMER SUPER:                                                    │
│  Если нужно ПИСАТЬ в коллекцию → ? super T                          │
│  List<? super Integer> → можно add(Integer)                         │
│                                                                     │
│  Example from Collections:                                          │
│  public static <T> void copy(                                       │
│      List<? super T> dest,    // consumer - записываем              │
│      List<? extends T> src    // producer - читаем                  │
│  )                                                                  │
│                                                                     │
│  Мнемоника:                                                         │
│  GET = extends (Producer)                                           │
│  PUT = super (Consumer)                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Wildcard Examples

```java
// Копирование из src в dest
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (T item : src) {
        dest.add(item);
    }
}

// Работает с разными типами
List<Number> numbers = new ArrayList<>();
List<Integer> integers = List.of(1, 2, 3);
copy(numbers, integers);  // OK: Integer extends Number

// Comparator обычно использует super
interface Comparator<T> {
    int compare(T o1, T o2);
}

// Можно сортировать List<Integer> компаратором для Number
List<Integer> ints = new ArrayList<>(List.of(3, 1, 2));
Comparator<Number> comp = (a, b) -> Double.compare(a.doubleValue(), b.doubleValue());
ints.sort(comp);  // Работает благодаря ? super T в sort()
```

---

## Generic Type Invariance

```java
// Arrays are covariant (небезопасно!)
Number[] numbers = new Integer[10];
numbers[0] = 3.14;  // ArrayStoreException в runtime!

// Generics are invariant (безопасно)
// List<Number> numbers = new ArrayList<Integer>();  // Compile error!

// Почему это правильно:
List<Integer> ints = new ArrayList<>();
// Если бы это работало:
// List<Number> nums = ints;
// nums.add(3.14);  // Добавили Double в List<Integer>!
// Integer i = ints.get(0);  // ClassCastException!

// Wildcards позволяют гибкость с сохранением безопасности
List<? extends Number> nums = new ArrayList<Integer>();  // OK для чтения
List<? super Integer> dest = new ArrayList<Number>();    // OK для записи
```

---

## Generic Inheritance

```java
// Generic class может extends/implements
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable {
}

// Можно фиксировать type parameter
public class StringList extends ArrayList<String> {
    // Все методы уже типизированы для String
}

// Или добавить свой
public class Pair<K, V> { }
public class OrderedPair<K, V> extends Pair<K, V> { }
public class StringPair extends Pair<String, String> { }

// Generic interface hierarchies
interface Collection<E> { }
interface List<E> extends Collection<E> { }
interface Set<E> extends Collection<E> { }
```

---

## Рекурсивные ограничения типов

```java
// Self-referential type bounds
public interface Comparable<T> {
    int compareTo(T o);
}

// Enum использует рекурсивный bound
public abstract class Enum<E extends Enum<E>> {
    public final int compareTo(E o) { ... }
}

// Builder pattern с generics
public abstract class Builder<T extends Builder<T>> {
    protected String name;

    public T withName(String name) {
        this.name = name;
        return self();
    }

    protected abstract T self();
    public abstract Product build();
}

public class ConcreteBuilder extends Builder<ConcreteBuilder> {
    @Override
    protected ConcreteBuilder self() {
        return this;
    }

    @Override
    public Product build() {
        return new Product(name);
    }
}

// Fluent API работает без cast
ConcreteBuilder builder = new ConcreteBuilder()
    .withName("test")  // Возвращает ConcreteBuilder, не Builder
    .withName("test2");
```

---

## На интервью

### Типичные вопросы

1. **What is type erasure?**
   - Generic types exist only at compile time
   - Erased to bounds (or Object) at runtime
   - Cannot use instanceof, new T(), T.class

2. **List<?> vs List\<Object\>?**
   - `List<?>`: any type, read-only (can't add)
   - `List<Object>`: only `List<Object>`, can add Object

3. **Explain PECS?**
   - Producer Extends: read from collection
   - Consumer Super: write to collection
   - `copy(List<? super T> dest, List<? extends T> src)`

4. **Why are arrays covariant but generics invariant?**
   - Arrays: runtime type checking (ArrayStoreException)
   - Generics: compile-time safety, type erasure
   - Wildcards provide controlled covariance/contravariance

5. **Can you create generic array?**
   - No: `T[] arr = new T[10]` is compile error
   - Workaround: `@SuppressWarnings("unchecked") T[] arr = (T[]) new Object[10]`
   - Or use `Array.newInstance(clazz, size)`

6. **What is raw type?**
   - Generic type used without type parameter
   - `List list = new ArrayList()` - loses type safety
   - Exists for backward compatibility

---

[← Назад к списку тем](README.md)
