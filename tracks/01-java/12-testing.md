# 12. Testing (JUnit 5 & Mockito)

[← Назад к списку тем](README.md)

---

## JUnit 5 Basics

### Test Structure

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private Calculator calculator;

    @BeforeAll
    static void setUpAll() {
        // Once before all tests (must be static)
        System.out.println("Starting tests...");
    }

    @AfterAll
    static void tearDownAll() {
        // Once after all tests (must be static)
        System.out.println("All tests completed");
    }

    @BeforeEach
    void setUp() {
        // Before each test
        calculator = new Calculator();
    }

    @AfterEach
    void tearDown() {
        // After each test
        calculator = null;
    }

    @Test
    void shouldAddTwoNumbers() {
        int result = calculator.add(2, 3);
        assertEquals(5, result);
    }

    @Test
    @DisplayName("Division by zero should throw exception")
    void shouldThrowOnDivisionByZero() {
        assertThrows(ArithmeticException.class,
            () -> calculator.divide(10, 0));
    }

    @Test
    @Disabled("Not implemented yet")
    void futureFeature() {
        // Skipped
    }
}
```

### Assertions

```java
import static org.junit.jupiter.api.Assertions.*;

// Basic assertions
assertEquals(expected, actual);
assertEquals(expected, actual, "Custom message");
assertNotEquals(unexpected, actual);

assertTrue(condition);
assertFalse(condition);

assertNull(object);
assertNotNull(object);

assertSame(expected, actual);      // Same reference
assertNotSame(unexpected, actual);

// Array assertions
assertArrayEquals(expectedArray, actualArray);

// Floating point (with delta)
assertEquals(3.14, actual, 0.001);

// Exception assertions
Exception exception = assertThrows(IllegalArgumentException.class,
    () -> service.process(null));
assertEquals("Value required", exception.getMessage());

assertDoesNotThrow(() -> service.process("valid"));

// Timeout
assertTimeout(Duration.ofSeconds(1), () -> {
    // Code that should complete within 1 second
});

assertTimeoutPreemptively(Duration.ofMillis(100), () -> {
    // Aborted if exceeds timeout
});

// Multiple assertions (all executed, failures collected)
assertAll("person",
    () -> assertEquals("John", person.getName()),
    () -> assertEquals(30, person.getAge()),
    () -> assertNotNull(person.getEmail())
);
```

### Parameterized Tests

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;

class ParameterizedTestExamples {

    // ValueSource - single parameter
    @ParameterizedTest
    @ValueSource(strings = {"hello", "world", "junit"})
    void testWithStrings(String word) {
        assertTrue(word.length() > 0);
    }

    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 4, 5})
    void testWithInts(int number) {
        assertTrue(number > 0);
    }

    // NullAndEmptySource
    @ParameterizedTest
    @NullAndEmptySource
    @ValueSource(strings = {"  ", "\t", "\n"})
    void testBlankStrings(String text) {
        assertTrue(text == null || text.isBlank());
    }

    // EnumSource
    @ParameterizedTest
    @EnumSource(Month.class)
    void testWithEnum(Month month) {
        assertNotNull(month);
    }

    @ParameterizedTest
    @EnumSource(value = Month.class, names = {"APRIL", "JUNE", "SEPTEMBER", "NOVEMBER"})
    void testThirtyDayMonths(Month month) {
        // Only specified months
    }

    // CsvSource - multiple parameters
    @ParameterizedTest
    @CsvSource({
        "1, 2, 3",
        "10, 20, 30",
        "0, 0, 0"
    })
    void testAddition(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }

    @ParameterizedTest
    @CsvSource({
        "apple, APPLE",
        "hello, HELLO",
        "'foo, bar', 'FOO, BAR'"  // Quoted strings
    })
    void testUpperCase(String input, String expected) {
        assertEquals(expected, input.toUpperCase());
    }

    // CsvFileSource - from file
    @ParameterizedTest
    @CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
    void testFromFile(String name, int age) {
        // ...
    }

    // MethodSource - custom method
    @ParameterizedTest
    @MethodSource("provideTestData")
    void testWithMethodSource(String input, int expected) {
        assertEquals(expected, input.length());
    }

    static Stream<Arguments> provideTestData() {
        return Stream.of(
            Arguments.of("hello", 5),
            Arguments.of("world", 5),
            Arguments.of("", 0)
        );
    }

    // ArgumentsSource - custom provider
    @ParameterizedTest
    @ArgumentsSource(MyArgumentsProvider.class)
    void testWithCustomProvider(String argument) {
        assertNotNull(argument);
    }
}

class MyArgumentsProvider implements ArgumentsProvider {
    @Override
    public Stream<? extends Arguments> provideArguments(ExtensionContext context) {
        return Stream.of("apple", "banana").map(Arguments::of);
    }
}
```

### Nested Tests

```java
@DisplayName("Calculator Tests")
class CalculatorTest {

    Calculator calculator = new Calculator();

    @Nested
    @DisplayName("Addition")
    class AdditionTests {

        @Test
        void shouldAddPositiveNumbers() {
            assertEquals(5, calculator.add(2, 3));
        }

        @Test
        void shouldAddNegativeNumbers() {
            assertEquals(-5, calculator.add(-2, -3));
        }
    }

    @Nested
    @DisplayName("Division")
    class DivisionTests {

        @Test
        void shouldDivide() {
            assertEquals(2, calculator.divide(10, 5));
        }

        @Test
        void shouldThrowOnZero() {
            assertThrows(ArithmeticException.class,
                () -> calculator.divide(10, 0));
        }
    }
}
```

### Test Lifecycle

```java
// Per-method (default) - new instance for each test
@TestInstance(TestInstance.Lifecycle.PER_METHOD)
class PerMethodTest {
    private int counter = 0;

    @Test
    void test1() { counter++; assertEquals(1, counter); }  // Pass

    @Test
    void test2() { counter++; assertEquals(1, counter); }  // Pass (new instance)
}

// Per-class - same instance for all tests
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class PerClassTest {
    private int counter = 0;

    @BeforeAll
    void setUp() {  // No static needed
        counter = 0;
    }

    @Test
    void test1() { counter++; }  // counter = 1

    @Test
    void test2() { counter++; }  // counter = 2 (same instance)
}
```

### Test Ordering

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderedTests {

    @Test
    @Order(1)
    void firstTest() { }

    @Test
    @Order(2)
    void secondTest() { }

    @Test
    @Order(3)
    void thirdTest() { }
}

// Other orderers:
// MethodOrderer.MethodName - alphabetical
// MethodOrderer.Random - random order
// MethodOrderer.DisplayName - by @DisplayName
```

---

## Mockito

### Basic Mocking

```java
import static org.mockito.Mockito.*;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    @Test
    void shouldFindUser() {
        // Given - setup mock behavior
        User mockUser = new User(1L, "John");
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

        // When - call method under test
        User result = userService.findById(1L);

        // Then - verify result
        assertEquals("John", result.getName());

        // Verify interaction
        verify(userRepository).findById(1L);
        verify(userRepository, times(1)).findById(1L);
        verify(userRepository, never()).delete(any());
    }
}

// Alternative: create mocks programmatically
@Test
void testWithoutAnnotations() {
    UserRepository repo = mock(UserRepository.class);
    when(repo.findById(1L)).thenReturn(Optional.of(new User(1L, "John")));
    // ...
}
```

### Stubbing Methods

```java
// Return value
when(mock.method()).thenReturn(value);
when(mock.method(arg)).thenReturn(value);

// Return different values on consecutive calls
when(mock.method())
    .thenReturn(value1)
    .thenReturn(value2)
    .thenReturn(value3);

// Or shorter
when(mock.method()).thenReturn(value1, value2, value3);

// Throw exception
when(mock.method()).thenThrow(new RuntimeException("Error"));
when(mock.method()).thenThrow(RuntimeException.class);

// Call real method
when(mock.method()).thenCallRealMethod();

// Answer - dynamic response
when(mock.greet(anyString())).thenAnswer(invocation -> {
    String name = invocation.getArgument(0);
    return "Hello, " + name;
});

// doReturn/doThrow for void methods
doNothing().when(mock).voidMethod();
doThrow(new RuntimeException()).when(mock).voidMethod();

// For void methods that need stubbing
doAnswer(invocation -> {
    System.out.println("Called");
    return null;
}).when(mock).voidMethod();
```

### Argument Matchers

```java
import static org.mockito.ArgumentMatchers.*;

// Any value
when(mock.method(any())).thenReturn(value);
when(mock.method(anyInt())).thenReturn(value);
when(mock.method(anyString())).thenReturn(value);
when(mock.method(anyList())).thenReturn(value);

// Specific type
when(mock.method(any(User.class))).thenReturn(value);

// Null / not null
when(mock.method(isNull())).thenReturn(value);
when(mock.method(notNull())).thenReturn(value);

// Equals
when(mock.method(eq("specific"))).thenReturn(value);

// Custom matcher
when(mock.method(argThat(arg -> arg.length() > 5))).thenReturn(value);

// Combining matchers (must use matchers for ALL args if using any matcher)
when(mock.method(eq("name"), anyInt())).thenReturn(value);
// when(mock.method("name", anyInt())).thenReturn(value);  // ERROR!

// Verify with matchers
verify(mock).method(eq("value"));
verify(mock).method(anyString());
verify(mock).method(argThat(s -> s.startsWith("prefix")));
```

### Verification

```java
// Simple verification
verify(mock).method();
verify(mock).method(arg);

// Number of invocations
verify(mock, times(3)).method();
verify(mock, never()).method();
verify(mock, atLeast(1)).method();
verify(mock, atLeastOnce()).method();
verify(mock, atMost(5)).method();

// Order of invocations
InOrder inOrder = inOrder(mock1, mock2);
inOrder.verify(mock1).firstMethod();
inOrder.verify(mock2).secondMethod();

// No more interactions
verifyNoMoreInteractions(mock);
verifyNoInteractions(mock);

// Argument captor
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(mock).save(captor.capture());
User capturedUser = captor.getValue();
assertEquals("John", capturedUser.getName());

// Multiple captures
verify(mock, times(2)).save(captor.capture());
List<User> allCaptured = captor.getAllValues();
```

### Spy

```java
// Spy - partial mock, real object with some methods stubbed
List<String> realList = new ArrayList<>();
List<String> spyList = spy(realList);

// Real methods are called by default
spyList.add("one");
assertEquals(1, spyList.size());  // Real size

// But can stub specific methods
when(spyList.size()).thenReturn(100);
assertEquals(100, spyList.size());  // Stubbed

// Use doReturn for spy (to avoid calling real method during stubbing)
doReturn("mocked").when(spyList).get(0);

// With annotation
@Spy
List<String> spyList = new ArrayList<>();
```

### BDD Style (Given-When-Then)

```java
import static org.mockito.BDDMockito.*;

@Test
void shouldFindUser_BDDStyle() {
    // Given
    given(userRepository.findById(1L))
        .willReturn(Optional.of(new User(1L, "John")));

    // When
    User result = userService.findById(1L);

    // Then
    then(userRepository).should().findById(1L);
    then(userRepository).should(never()).delete(any());
    assertThat(result.getName()).isEqualTo("John");  // AssertJ
}
```

---

## AssertJ (Fluent Assertions)

```java
import static org.assertj.core.api.Assertions.*;

// String assertions
assertThat(name)
    .isNotNull()
    .isNotEmpty()
    .startsWith("J")
    .endsWith("n")
    .contains("oh")
    .hasSize(4);

// Number assertions
assertThat(age)
    .isPositive()
    .isGreaterThan(18)
    .isLessThanOrEqualTo(100)
    .isBetween(18, 100);

// Collection assertions
assertThat(list)
    .hasSize(3)
    .contains("apple", "banana")
    .containsExactly("apple", "banana", "cherry")
    .containsExactlyInAnyOrder("cherry", "apple", "banana")
    .doesNotContain("grape")
    .allMatch(s -> s.length() > 3);

// Object assertions
assertThat(user)
    .isNotNull()
    .hasFieldOrPropertyWithValue("name", "John")
    .extracting(User::getName, User::getAge)
    .containsExactly("John", 30);

// Exception assertions
assertThatThrownBy(() -> service.process(null))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("Value required")
    .hasMessageContaining("required");

assertThatCode(() -> service.process("valid"))
    .doesNotThrowAnyException();

// Soft assertions (collect all failures)
SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(user.getName()).isEqualTo("John");
    softly.assertThat(user.getAge()).isEqualTo(30);
    softly.assertThat(user.getEmail()).contains("@");
});
```

---

## Testing Best Practices

### Test Naming

```java
// Pattern: should[ExpectedBehavior]_when[Condition]
@Test
void shouldReturnEmpty_whenUserNotFound() { }

@Test
void shouldThrowException_whenAmountIsNegative() { }

// Or with @DisplayName
@Test
@DisplayName("Should return empty when user not found")
void userNotFoundTest() { }
```

### AAA Pattern

```java
@Test
void shouldCalculateTotal() {
    // Arrange - setup
    ShoppingCart cart = new ShoppingCart();
    cart.addItem(new Item("Apple", 10.0));
    cart.addItem(new Item("Banana", 5.0));

    // Act - execute
    double total = cart.calculateTotal();

    // Assert - verify
    assertEquals(15.0, total);
}
```

### Test Independence

```java
// BAD - tests depend on each other
class BadTests {
    static User user;

    @Test @Order(1)
    void createUser() {
        user = userService.create("John");
    }

    @Test @Order(2)
    void updateUser() {
        userService.update(user.getId(), "Jane");  // Depends on test1!
    }
}

// GOOD - each test is independent
class GoodTests {
    @Test
    void shouldCreateUser() {
        User user = userService.create("John");
        assertNotNull(user.getId());
    }

    @Test
    void shouldUpdateUser() {
        User user = userService.create("John");  // Own setup
        userService.update(user.getId(), "Jane");
        assertEquals("Jane", userService.findById(user.getId()).getName());
    }
}
```

### Test Pyramid

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Test Pyramid                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                          /\                                         │
│                         /  \       E2E Tests                        │
│                        /    \      (few, slow, brittle)             │
│                       /──────\                                      │
│                      /        \    Integration Tests                │
│                     /          \   (some, medium speed)             │
│                    /────────────\                                   │
│                   /              \  Unit Tests                      │
│                  /                \ (many, fast, isolated)          │
│                 /──────────────────\                                │
│                                                                     │
│  Unit Tests:        70-80%  - Test single class/method              │
│  Integration Tests: 15-20%  - Test component interaction            │
│  E2E Tests:         5-10%   - Test full user scenarios              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **JUnit 4 vs JUnit 5?**
   - JUnit 5: modular (Jupiter, Platform, Vintage)
   - New annotations: @BeforeEach, @DisplayName, @Nested
   - Parameterized tests, extensions

2. **Mock vs Stub vs Spy?**
   - Mock: fake object, no behavior by default
   - Stub: mock with predefined responses
   - Spy: real object with some methods stubbed

3. **When to use mocking?**
   - External dependencies (DB, APIs)
   - Slow operations
   - Non-deterministic behavior
   - Don't mock value objects

4. **What is the Test Pyramid?**
   - Many unit tests (fast, isolated)
   - Some integration tests
   - Few E2E tests (slow, brittle)

5. **@InjectMocks behavior?**
   - Creates instance of class
   - Injects @Mock fields
   - Constructor, setter, or field injection

6. **Test naming conventions?**
   - Describe expected behavior
   - should[Expected]_when[Condition]
   - Use @DisplayName for readability

---

[← Назад к списку тем](README.md)
