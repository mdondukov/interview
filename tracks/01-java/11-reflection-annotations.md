# 11. Reflection & Annotations

[← Назад к списку тем](README.md)

---

## Reflection API

### Getting Class Information

```java
// Getting Class object
Class<String> cls1 = String.class;                     // From type
Class<?> cls2 = "hello".getClass();                    // From instance
Class<?> cls3 = Class.forName("java.lang.String");     // By name (can throw)

// Class info
String name = cls1.getName();           // java.lang.String
String simpleName = cls1.getSimpleName(); // String
Package pkg = cls1.getPackage();
int modifiers = cls1.getModifiers();

// Check type
Modifier.isPublic(modifiers);
Modifier.isAbstract(modifiers);
Modifier.isFinal(modifiers);

cls1.isInterface();
cls1.isEnum();
cls1.isArray();
cls1.isPrimitive();
cls1.isAnnotation();

// Hierarchy
Class<?> superclass = cls1.getSuperclass();
Class<?>[] interfaces = cls1.getInterfaces();
boolean isAssignable = Number.class.isAssignableFrom(Integer.class); // true
```

### Working with Fields

```java
public class Person {
    public String name;
    private int age;
    private static final String TYPE = "HUMAN";
}

Class<Person> cls = Person.class;

// Get fields
Field[] publicFields = cls.getFields();           // Only public (including inherited)
Field[] allFields = cls.getDeclaredFields();      // All declared (not inherited)
Field nameField = cls.getField("name");           // By name (public only)
Field ageField = cls.getDeclaredField("age");     // By name (any access)

// Field info
String fieldName = ageField.getName();
Class<?> fieldType = ageField.getType();
int modifiers = ageField.getModifiers();

// Access private field
ageField.setAccessible(true);  // Bypass access check

// Get/set value
Person person = new Person();
ageField.set(person, 25);
int age = (int) ageField.get(person);

// Static field
Field typeField = cls.getDeclaredField("TYPE");
typeField.setAccessible(true);
String type = (String) typeField.get(null);  // null for static
```

### Working with Methods

```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    private int multiply(int a, int b) { return a * b; }
    public static int square(int x) { return x * x; }
}

Class<Calculator> cls = Calculator.class;

// Get methods
Method[] publicMethods = cls.getMethods();          // Public (including inherited)
Method[] declaredMethods = cls.getDeclaredMethods(); // All declared

// Get specific method
Method addMethod = cls.getMethod("add", int.class, int.class);
Method multiplyMethod = cls.getDeclaredMethod("multiply", int.class, int.class);

// Method info
String methodName = addMethod.getName();
Class<?> returnType = addMethod.getReturnType();
Class<?>[] paramTypes = addMethod.getParameterTypes();
Class<?>[] exceptions = addMethod.getExceptionTypes();

// Invoke method
Calculator calc = new Calculator();
Object result = addMethod.invoke(calc, 5, 3);  // returns 8

// Invoke private method
multiplyMethod.setAccessible(true);
Object product = multiplyMethod.invoke(calc, 4, 5);  // returns 20

// Invoke static method
Method squareMethod = cls.getMethod("square", int.class);
Object squared = squareMethod.invoke(null, 5);  // null for static, returns 25
```

### Working with Constructors

```java
public class Person {
    private String name;
    private int age;

    public Person() { }
    public Person(String name) { this.name = name; }
    private Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

Class<Person> cls = Person.class;

// Get constructors
Constructor<?>[] publicCtors = cls.getConstructors();
Constructor<?>[] allCtors = cls.getDeclaredConstructors();

// Get specific constructor
Constructor<Person> defaultCtor = cls.getConstructor();
Constructor<Person> nameCtor = cls.getConstructor(String.class);
Constructor<Person> privateCtor = cls.getDeclaredConstructor(String.class, int.class);

// Create instance
Person p1 = defaultCtor.newInstance();
Person p2 = nameCtor.newInstance("John");

// Using private constructor
privateCtor.setAccessible(true);
Person p3 = privateCtor.newInstance("Jane", 25);

// Simple instance creation (default constructor)
Person p4 = cls.getDeclaredConstructor().newInstance();
```

### Working with Arrays

```java
// Create array via reflection
int[] intArray = (int[]) Array.newInstance(int.class, 5);

// Get/set elements
Array.set(intArray, 0, 42);
int value = (int) Array.get(intArray, 0);

// Get array info
Class<?> componentType = intArray.getClass().getComponentType(); // int
int length = Array.getLength(intArray);

// Multi-dimensional
int[][] matrix = (int[][]) Array.newInstance(int.class, 3, 4);
```

### Generic Type Information

```java
// Type erasure limits what we can get at runtime
// But we can get generic info from class/method declarations

public class Box<T> {
    private T value;
    private List<String> strings;
}

// Get generic field type
Field field = Box.class.getDeclaredField("strings");
Type genericType = field.getGenericType();

if (genericType instanceof ParameterizedType) {
    ParameterizedType pt = (ParameterizedType) genericType;
    Type rawType = pt.getRawType();  // List
    Type[] typeArgs = pt.getActualTypeArguments();  // [String]
}

// Get generic method return type
public <T> List<T> getItems() { return null; }

Method method = MyClass.class.getMethod("getItems");
Type returnType = method.getGenericReturnType();

// Get type parameters of class
TypeVariable<?>[] typeParams = Box.class.getTypeParameters();
```

---

## Annotations

### Built-in Annotations

```java
// @Override - compiler checks method overrides
@Override
public String toString() { return "..."; }

// @Deprecated - marks as deprecated
@Deprecated(since = "2.0", forRemoval = true)
public void oldMethod() { }

// @SuppressWarnings - suppress compiler warnings
@SuppressWarnings("unchecked")
public void method() { }

@SuppressWarnings({"unchecked", "deprecation"})
public void method2() { }

// @FunctionalInterface - compiler checks single abstract method
@FunctionalInterface
public interface MyFunction {
    void apply();
}

// @SafeVarargs - suppress varargs warnings
@SafeVarargs
public final <T> void process(T... items) { }
```

### Creating Custom Annotations

```java
// Simple annotation
public @interface Author {
    String name();
    String date() default "unknown";
}

// Usage
@Author(name = "John Doe", date = "2024-01-01")
public class MyClass { }

// Marker annotation (no elements)
public @interface Immutable { }

@Immutable
public class Point { }

// Single-element annotation (use 'value')
public @interface Version {
    int value();
}

@Version(1)  // Can omit 'value ='
public class MyClass { }

// With array
public @interface Tags {
    String[] value();
}

@Tags({"important", "reviewed"})
public class MyClass { }

@Tags("single")  // Single element - no braces needed
public class MyClass2 { }
```

### Meta-Annotations

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)  // When available
@Target(ElementType.METHOD)           // Where can be used
@Documented                           // Include in Javadoc
@Inherited                            // Inherited by subclasses
public @interface MyAnnotation {
    String value();
}

/*
@Retention values:
- SOURCE: compile-time only, not in bytecode
- CLASS: in bytecode, not at runtime (default)
- RUNTIME: available at runtime via reflection

@Target values:
- TYPE: class, interface, enum
- FIELD: fields
- METHOD: methods
- PARAMETER: method parameters
- CONSTRUCTOR: constructors
- LOCAL_VARIABLE: local variables
- ANNOTATION_TYPE: other annotations
- PACKAGE: package declaration
- TYPE_PARAMETER: type parameters (Java 8+)
- TYPE_USE: any type use (Java 8+)
*/
```

### Repeatable Annotations (Java 8+)

```java
// Container annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Schedules {
    Schedule[] value();
}

// Repeatable annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Repeatable(Schedules.class)
public @interface Schedule {
    String dayOfWeek();
    String time();
}

// Usage - can repeat
@Schedule(dayOfWeek = "Monday", time = "09:00")
@Schedule(dayOfWeek = "Wednesday", time = "14:00")
public class Task { }
```

### Processing Annotations at Runtime

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Test {
    String description() default "";
    boolean enabled() default true;
}

public class TestRunner {

    public static void runTests(Class<?> testClass) throws Exception {
        // Check class annotation
        if (testClass.isAnnotationPresent(Test.class)) {
            Test classAnnotation = testClass.getAnnotation(Test.class);
            System.out.println("Test class: " + classAnnotation.description());
        }

        Object instance = testClass.getDeclaredConstructor().newInstance();

        // Find and run test methods
        for (Method method : testClass.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Test.class)) {
                Test annotation = method.getAnnotation(Test.class);

                if (annotation.enabled()) {
                    System.out.println("Running: " + method.getName());
                    try {
                        method.invoke(instance);
                        System.out.println("PASSED");
                    } catch (Exception e) {
                        System.out.println("FAILED: " + e.getCause());
                    }
                }
            }
        }

        // Get all annotations
        Annotation[] annotations = testClass.getAnnotations();
        for (Annotation a : annotations) {
            System.out.println(a.annotationType().getName());
        }
    }
}

// Test class
@Test(description = "Calculator tests")
public class CalculatorTest {

    @Test(description = "Addition test")
    public void testAdd() {
        assert 2 + 2 == 4;
    }

    @Test(enabled = false)
    public void testSkipped() {
        // Not executed
    }
}
```

---

## Dynamic Proxy

```java
import java.lang.reflect.*;

// Interface to proxy
public interface UserService {
    User findById(long id);
    void save(User user);
}

// Implementation
public class UserServiceImpl implements UserService {
    public User findById(long id) { return new User(id); }
    public void save(User user) { System.out.println("Saved: " + user); }
}

// InvocationHandler - intercepts all method calls
public class LoggingHandler implements InvocationHandler {
    private final Object target;

    public LoggingHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before: " + method.getName());
        long start = System.currentTimeMillis();

        try {
            Object result = method.invoke(target, args);
            return result;
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            System.out.println("After: " + method.getName() + " (" + elapsed + "ms)");
        }
    }
}

// Create proxy
UserService realService = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{UserService.class},
    new LoggingHandler(realService)
);

// Use proxy
proxy.findById(1);
// Output:
// Before: findById
// After: findById (5ms)
```

### Practical Proxy Examples

```java
// Transaction proxy
public class TransactionHandler implements InvocationHandler {
    private final Object target;
    private final TransactionManager txManager;

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        txManager.begin();
        try {
            Object result = method.invoke(target, args);
            txManager.commit();
            return result;
        } catch (Exception e) {
            txManager.rollback();
            throw e;
        }
    }
}

// Caching proxy
public class CachingHandler implements InvocationHandler {
    private final Object target;
    private final Map<String, Object> cache = new ConcurrentHashMap<>();

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.isAnnotationPresent(Cacheable.class)) {
            String key = method.getName() + Arrays.toString(args);
            return cache.computeIfAbsent(key, k -> {
                try {
                    return method.invoke(target, args);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
        }
        return method.invoke(target, args);
    }
}
```

---

## Reflection Performance

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Reflection Performance                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Reflection is SLOWER than direct access:                           │
│  - Method lookup: cache Method objects                              │
│  - setAccessible(true): call once, reuse                            │
│  - Method invocation: ~5-10x slower                                 │
│                                                                     │
│  Optimization tips:                                                 │
│                                                                     │
│  // BAD - lookup every time                                         │
│  for (...) {                                                        │
│      Method m = cls.getMethod("process", String.class);             │
│      m.invoke(obj, value);                                          │
│  }                                                                  │
│                                                                     │
│  // GOOD - cache Method object                                      │
│  Method m = cls.getMethod("process", String.class);                 │
│  m.setAccessible(true);                                             │
│  for (...) {                                                        │
│      m.invoke(obj, value);                                          │
│  }                                                                  │
│                                                                     │
│  Modern alternatives:                                               │
│  - MethodHandle (java.lang.invoke) - faster than reflection         │
│  - VarHandle - for field access                                     │
│  - LambdaMetafactory - generate lambda at runtime                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### MethodHandle Example

```java
import java.lang.invoke.*;

// Faster than reflection for repeated calls
MethodHandles.Lookup lookup = MethodHandles.lookup();

// Get method handle
MethodType type = MethodType.methodType(int.class, int.class, int.class);
MethodHandle addHandle = lookup.findVirtual(Calculator.class, "add", type);

// Invoke
Calculator calc = new Calculator();
int result = (int) addHandle.invoke(calc, 5, 3);  // or invokeExact

// For static methods
MethodHandle staticHandle = lookup.findStatic(Math.class, "max",
    MethodType.methodType(int.class, int.class, int.class));
int max = (int) staticHandle.invoke(10, 20);
```

---

## На интервью

### Типичные вопросы

1. **What is Reflection?**
   - Inspect/modify classes at runtime
   - Get class info, fields, methods, constructors
   - Create instances, invoke methods dynamically

2. **When to use Reflection?**
   - Frameworks (Spring, Hibernate, JUnit)
   - Serialization/deserialization
   - Dynamic proxy
   - Plugin systems
   - Avoid for regular application code

3. **Reflection drawbacks?**
   - Performance overhead
   - Breaks encapsulation
   - No compile-time checking
   - Security restrictions (modules)

4. **What is Dynamic Proxy?**
   - Create interface implementation at runtime
   - Proxy.newProxyInstance()
   - InvocationHandler intercepts calls
   - Used for AOP, transactions, caching

5. **Annotation retention policies?**
   - SOURCE: compile-time only
   - CLASS: in bytecode, not runtime
   - RUNTIME: available via reflection

6. **How Spring uses annotations?**
   - Scans classpath for annotated classes
   - Processes @Component, @Service, etc.
   - Creates beans, injects dependencies
   - Uses reflection and proxies

---

[← Назад к списку тем](README.md)
