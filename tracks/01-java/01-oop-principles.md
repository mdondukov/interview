# 01. OOP Principles

[← Назад к списку тем](README.md)

---

## Four Pillars of OOP

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Four Pillars of OOP                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ENCAPSULATION                                                      │
│  - Bundle data and methods together                                 │
│  - Hide internal state                                              │
│  - Control access via modifiers                                     │
│                                                                     │
│  INHERITANCE                                                        │
│  - Reuse code from parent class                                     │
│  - "is-a" relationship                                              │
│  - Single inheritance in Java (extends)                             │
│                                                                     │
│  POLYMORPHISM                                                       │
│  - Same interface, different implementations                        │
│  - Runtime (overriding) and compile-time (overloading)              │
│  - Enables abstraction                                              │
│                                                                     │
│  ABSTRACTION                                                        │
│  - Hide complexity, show only essentials                            │
│  - Abstract classes and interfaces                                  │
│  - Focus on "what" not "how"                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Encapsulation

```java
// Bad: public fields, no control
public class User {
    public String name;
    public int age;
}

// Good: private fields with controlled access
public class User {
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be empty");
        }
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age");
        }
        this.age = age;
    }
}
```

### Access Modifiers

```
┌─────────────────┬─────────┬─────────┬───────────┬───────────┐
│ Modifier        │ Class   │ Package │ Subclass  │ World     │
├─────────────────┼─────────┼─────────┼───────────┼───────────┤
│ public          │   ✓     │    ✓    │     ✓     │     ✓     │
│ protected       │   ✓     │    ✓    │     ✓     │     ✗     │
│ (default)       │   ✓     │    ✓    │     ✗     │     ✗     │
│ private         │   ✓     │    ✗    │     ✗     │     ✗     │
└─────────────────┴─────────┴─────────┴───────────┴───────────┘
```

---

## Inheritance

```java
public class Animal {
    protected String name;

    public Animal(String name) {
        this.name = name;
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

public class Dog extends Animal {
    private String breed;

    public Dog(String name, String breed) {
        super(name);  // Call parent constructor
        this.breed = breed;
    }

    // Additional method
    public void bark() {
        System.out.println(name + " is barking");
    }

    // Override parent method
    @Override
    public void eat() {
        System.out.println(name + " is eating dog food");
    }
}
```

### Inheritance Rules

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Inheritance Rules                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Single inheritance: class extends only one class                │
│  2. Multiple interface implementation allowed                       │
│  3. final class cannot be extended                                  │
│  4. final method cannot be overridden                               │
│  5. Constructor is not inherited (but can call super())             │
│  6. private members are not inherited (but exist)                   │
│  7. static methods are not overridden (hidden)                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Polymorphism

### Runtime Polymorphism (Method Overriding)

```java
public class Shape {
    public double area() {
        return 0;
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    private double width, height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() {
        return width * height;
    }
}

// Usage - polymorphic behavior
Shape shape1 = new Circle(5);
Shape shape2 = new Rectangle(4, 6);

System.out.println(shape1.area());  // Circle's area: 78.54
System.out.println(shape2.area());  // Rectangle's area: 24.0
```

### Compile-time Polymorphism (Method Overloading)

```java
public class Calculator {
    // Same method name, different parameters
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    // NOT overloading - different return type only is compile error
    // public double add(int a, int b) { ... }
}
```

### Overriding Rules

```java
public class Parent {
    protected Number getValue() throws IOException {
        return 1;
    }
}

public class Child extends Parent {
    // Valid: covariant return type (Integer extends Number)
    // Valid: narrower access (public vs protected)
    // Valid: narrower exception or no exception
    @Override
    public Integer getValue() {  // No exception declared
        return 42;
    }
}

/*
Overriding rules:
1. Same method signature (name + parameters)
2. Return type: same or covariant (subtype)
3. Access: same or wider (private → default → protected → public)
4. Exceptions: same, narrower, or none (unchecked always OK)
5. Cannot override static methods (they are hidden)
6. Cannot override final methods
*/
```

---

## Abstraction

### Abstract Classes

```java
public abstract class Vehicle {
    protected String brand;

    // Constructor
    public Vehicle(String brand) {
        this.brand = brand;
    }

    // Concrete method
    public void start() {
        System.out.println("Vehicle starting...");
    }

    // Abstract method - must be implemented by subclass
    public abstract void move();

    // Abstract method
    public abstract int getWheels();
}

public class Car extends Vehicle {
    public Car(String brand) {
        super(brand);
    }

    @Override
    public void move() {
        System.out.println(brand + " car is driving");
    }

    @Override
    public int getWheels() {
        return 4;
    }
}

public class Motorcycle extends Vehicle {
    public Motorcycle(String brand) {
        super(brand);
    }

    @Override
    public void move() {
        System.out.println(brand + " motorcycle is riding");
    }

    @Override
    public int getWheels() {
        return 2;
    }
}
```

### Interfaces

```java
public interface Drawable {
    // Constants (public static final by default)
    int MAX_SIZE = 100;

    // Abstract method (public abstract by default)
    void draw();

    // Default method (Java 8+)
    default void setColor(String color) {
        System.out.println("Setting color to " + color);
    }

    // Static method (Java 8+)
    static void printInfo() {
        System.out.println("Drawable interface");
    }

    // Private method (Java 9+) - for code reuse in defaults
    private void validateColor(String color) {
        if (color == null) throw new IllegalArgumentException();
    }
}

public interface Resizable {
    void resize(int width, int height);
}

// Multiple interface implementation
public class Rectangle implements Drawable, Resizable {
    @Override
    public void draw() {
        System.out.println("Drawing rectangle");
    }

    @Override
    public void resize(int width, int height) {
        System.out.println("Resizing to " + width + "x" + height);
    }
}
```

### Abstract Class vs Interface

```
┌─────────────────────────────────────────────────────────────────────┐
│              Abstract Class vs Interface                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Abstract Class:                   Interface:                       │
│  ────────────────                  ──────────                       │
│  - Can have state (fields)         - No state (only constants)      │
│  - Constructor allowed             - No constructor                 │
│  - Single inheritance              - Multiple implementation        │
│  - Any access modifiers            - public by default              │
│  - "is-a" relationship             - "can-do" relationship          │
│                                                                     │
│  Use Abstract Class when:          Use Interface when:              │
│  - Shared state needed             - Defining contract              │
│  - Common base implementation      - Multiple inheritance needed    │
│  - Related classes                 - Unrelated classes              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## SOLID Principles

### S — Single Responsibility Principle

```java
// BAD: Multiple responsibilities
public class User {
    public void saveToDatabase() { ... }
    public void sendEmail() { ... }
    public void generateReport() { ... }
}

// GOOD: Single responsibility
public class User {
    private String name;
    private String email;
    // Only user data and behavior
}

public class UserRepository {
    public void save(User user) { ... }
}

public class EmailService {
    public void sendWelcomeEmail(User user) { ... }
}

public class ReportGenerator {
    public Report generateUserReport(User user) { ... }
}
```

### O — Open/Closed Principle

```java
// BAD: Must modify class to add new shape
public class AreaCalculator {
    public double calculate(Object shape) {
        if (shape instanceof Circle) {
            Circle c = (Circle) shape;
            return Math.PI * c.radius * c.radius;
        } else if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle) shape;
            return r.width * r.height;
        }
        // Must add new if-else for each new shape!
        return 0;
    }
}

// GOOD: Open for extension, closed for modification
public interface Shape {
    double area();
}

public class Circle implements Shape {
    private double radius;
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle implements Shape {
    private double width, height;
    @Override
    public double area() {
        return width * height;
    }
}

// New shapes don't require modifying AreaCalculator
public class AreaCalculator {
    public double calculate(Shape shape) {
        return shape.area();
    }
}
```

### L — Liskov Substitution Principle

```java
// BAD: Square violates LSP - unexpected behavior
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // Violates expectations!
    }
    @Override
    public void setHeight(int height) {
        this.width = height;  // Violates expectations!
        this.height = height;
    }
}

// Client code breaks:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
assert r.area() == 50;  // FAILS! area is 100

// GOOD: Separate hierarchies or use interface
public interface Shape {
    int area();
}

public class Rectangle implements Shape {
    private final int width;
    private final int height;
    // Immutable, no setters
}

public class Square implements Shape {
    private final int side;
    // Different class, no inheritance issues
}
```

### I — Interface Segregation Principle

```java
// BAD: Fat interface
public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class Robot implements Worker {
    @Override public void work() { ... }
    @Override public void eat() { /* Robots don't eat! */ }
    @Override public void sleep() { /* Robots don't sleep! */ }
}

// GOOD: Segregated interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class Human implements Workable, Eatable, Sleepable {
    @Override public void work() { ... }
    @Override public void eat() { ... }
    @Override public void sleep() { ... }
}

public class Robot implements Workable {
    @Override public void work() { ... }
    // Only implements what it needs
}
```

### D — Dependency Inversion Principle

```java
// BAD: High-level depends on low-level
public class MySQLDatabase {
    public void save(String data) { ... }
}

public class UserService {
    private MySQLDatabase database = new MySQLDatabase();  // Tight coupling!

    public void saveUser(User user) {
        database.save(user.toString());
    }
}

// GOOD: Both depend on abstraction
public interface Database {
    void save(String data);
}

public class MySQLDatabase implements Database {
    @Override
    public void save(String data) { ... }
}

public class MongoDatabase implements Database {
    @Override
    public void save(String data) { ... }
}

public class UserService {
    private final Database database;

    // Dependency Injection
    public UserService(Database database) {
        this.database = database;
    }

    public void saveUser(User user) {
        database.save(user.toString());
    }
}

// Usage - easy to switch implementations
UserService service = new UserService(new MySQLDatabase());
UserService service2 = new UserService(new MongoDatabase());
```

---

## Composition vs Inheritance

```java
// Inheritance: "is-a" (tight coupling)
public class ArrayList<E> extends AbstractList<E> {
    // Inherits all behavior, hard to change
}

// Composition: "has-a" (loose coupling) - PREFERRED
public class Stack<E> {
    private final List<E> list = new ArrayList<>();

    public void push(E item) {
        list.add(item);
    }

    public E pop() {
        return list.remove(list.size() - 1);
    }
}

/*
Prefer composition because:
1. More flexible
2. No fragile base class problem
3. Can change implementation at runtime
4. Better encapsulation
*/
```

---

## См. также

- [Python OOP & Protocols](../02-python/03-oop-protocols.md) — ООП и протоколы в Python
- [Clean Architecture](../08-architecture/01-clean-architecture.md) — применение SOLID в архитектуре

---

## На интервью

### Типичные вопросы

1. **Four pillars of OOP?**
   - Encapsulation, Inheritance, Polymorphism, Abstraction

2. **Abstract class vs Interface?**
   - Abstract: state, constructor, single inheritance, partial impl
   - Interface: contract, multiple implementation, default methods (Java 8+)

3. **What is SOLID?**
   - Single Responsibility, Open/Closed, Liskov Substitution,
     Interface Segregation, Dependency Inversion

4. **Method overloading vs overriding?**
   - Overloading: same name, different params, compile-time
   - Overriding: same signature in subclass, runtime

5. **Why prefer composition over inheritance?**
   - Flexibility, loose coupling, no fragile base class
   - "has-a" vs "is-a"

6. **What is Liskov Substitution?**
   - Subtypes must be substitutable for base types
   - Square/Rectangle is classic violation example

---

[← Назад к списку тем](README.md)
