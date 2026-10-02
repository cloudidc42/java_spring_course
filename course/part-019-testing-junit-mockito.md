# Part 019: Testing with JUnit 5 & Mockito

## เนื้อหาในส่วนนี้
- JUnit 5 Architecture และ Annotations
- Assertions และ Assumptions
- Parameterized Tests
- Test Lifecycle และ Extensions
- Mockito: Mocking, Stubbing, Verification
- Test-Driven Development (TDD)
- Integration Testing Patterns
- Code Coverage และ Best Practices

---

## 1. JUnit 5 Overview

JUnit 5 ประกอบด้วย 3 sub-projects:
- **JUnit Platform**: foundation สำหรับ launch test frameworks
- **JUnit Jupiter**: new programming model (annotations, assertions)
- **JUnit Vintage**: backward compatibility กับ JUnit 3/4

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.7.0</version>
        <scope>test</scope>
    </dependency>
    
    <!-- AssertJ (optional but recommended) -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.1.2</version>
        </plugin>
    </plugins>
</build>
```

---

## 2. JUnit 5 Basic Annotations

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    
    private Calculator calculator;
    
    // ทำงานก่อนทุก test class (static)
    @BeforeAll
    static void initAll() {
        System.out.println("=== Starting Calculator Tests ===");
    }
    
    // ทำงานก่อนแต่ละ test method
    @BeforeEach
    void init() {
        calculator = new Calculator();
    }
    
    // ทำงานหลังแต่ละ test method
    @AfterEach
    void tearDown() {
        calculator = null;
    }
    
    // ทำงานหลังทุก test class (static)
    @AfterAll
    static void tearDownAll() {
        System.out.println("=== Calculator Tests Complete ===");
    }
    
    @Test
    @DisplayName("Addition of two positive numbers")
    void addTwoNumbers() {
        assertEquals(5, calculator.add(2, 3));
    }
    
    @Test
    @DisplayName("Division should throw on zero divisor")
    void divisionByZeroThrows() {
        assertThrows(ArithmeticException.class, () -> calculator.divide(10, 0));
    }
    
    @Test
    @Disabled("Feature not implemented yet")
    void notImplementedFeature() {
        // This test is skipped
    }
    
    @Test
    @Tag("slow")
    @Tag("integration")
    void heavyComputationTest() {
        // Tagged tests can be filtered in CI/CD
        int result = calculator.factorial(10);
        assertEquals(3628800, result);
    }
}

// Simple Calculator class
class Calculator {
    public int add(int a, int b) { return a + b; }
    public int subtract(int a, int b) { return a - b; }
    public int multiply(int a, int b) { return a * b; }
    
    public int divide(int a, int b) {
        if (b == 0) throw new ArithmeticException("Cannot divide by zero");
        return a / b;
    }
    
    public long factorial(int n) {
        if (n < 0) throw new IllegalArgumentException("Negative number");
        if (n == 0 || n == 1) return 1;
        return n * factorial(n - 1);
    }
}
```

---

## 3. JUnit 5 Assertions

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;
import java.time.Duration;
import java.util.List;

class AssertionsExampleTest {
    
    @Test
    void basicAssertions() {
        // assertEquals
        assertEquals(4, 2 + 2, "2 + 2 should equal 4");
        assertEquals(3.14, Math.PI, 0.01, "Pi approximation");
        
        // assertNotEquals
        assertNotEquals(5, 2 + 2);
        
        // assertTrue / assertFalse
        assertTrue("hello".startsWith("h"));
        assertFalse("hello".isEmpty());
        
        // assertNull / assertNotNull
        String name = null;
        assertNull(name);
        assertNotNull("value");
        
        // assertSame / assertNotSame (reference equality)
        String a = "hello";
        String b = a;
        assertSame(a, b);
        assertNotSame(a, new String("hello"));
        
        // assertArrayEquals
        assertArrayEquals(new int[]{1, 2, 3}, new int[]{1, 2, 3});
        
        // assertIterableEquals
        assertIterableEquals(List.of(1, 2, 3), List.of(1, 2, 3));
        
        // assertLinesMatch (for strings)
        assertLinesMatch(
            List.of("Line 1", "Line 2"),
            List.of("Line 1", "Line 2")
        );
    }
    
    @Test
    void groupedAssertions() {
        // assertAll - all assertions run even if one fails
        assertAll("person",
            () -> assertEquals("John", "John", "First name should match"),
            () -> assertEquals(30, 30, "Age should match"),
            () -> assertTrue(true, "Status should be active")
        );
    }
    
    @Test
    void exceptionAssertions() {
        // assertThrows - returns the exception for further assertions
        IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> { throw new IllegalArgumentException("Invalid input: -1"); }
        );
        assertEquals("Invalid input: -1", exception.getMessage());
        assertTrue(exception.getMessage().contains("Invalid"));
        
        // assertDoesNotThrow
        assertDoesNotThrow(() -> {
            int result = 10 / 2;
        });
    }
    
    @Test
    void timeoutAssertions() {
        // assertTimeout - fails if execution takes too long
        String result = assertTimeout(Duration.ofSeconds(1), () -> {
            Thread.sleep(100);
            return "quick operation";
        });
        assertEquals("quick operation", result);
        
        // assertTimeoutPreemptively - aborts after timeout
        assertTimeoutPreemptively(Duration.ofSeconds(2), () -> {
            // This must complete within 2 seconds
            long sum = 0;
            for (int i = 0; i < 1_000_000; i++) sum += i;
        });
    }
    
    @Test
    void assumptionsExample() {
        // Assumptions - test is skipped if assumption fails (not a failure)
        String os = System.getProperty("os.name");
        Assumptions.assumeTrue(os != null, "OS name not available");
        
        // assumingThat - conditional execution
        Assumptions.assumingThat(
            "ci".equals(System.getProperty("environment")),
            () -> assertEquals(2, 2) // only runs in CI environment
        );
        
        // This assertion always runs
        assertTrue(true, "Always executed");
    }
    
    @Test
    void failManually() {
        try {
            // Some risky operation
            riskyOperation();
        } catch (RuntimeException e) {
            // Expected - good
        } catch (Exception e) {
            fail("Unexpected exception type: " + e.getClass().getName());
        }
    }
    
    private void riskyOperation() throws RuntimeException {
        throw new RuntimeException("Expected");
    }
}
```

---

## 4. Parameterized Tests

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import static org.junit.jupiter.api.Assertions.*;
import java.util.stream.Stream;

class ParameterizedTestsExample {
    
    // ValueSource - single parameter
    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 4, 5})
    void isPositive(int number) {
        assertTrue(number > 0);
    }
    
    @ParameterizedTest
    @ValueSource(strings = {"hello", "world", "java"})
    void isNotEmpty(String str) {
        assertFalse(str.isEmpty());
    }
    
    // NullAndEmptySource
    @ParameterizedTest
    @NullAndEmptySource
    @ValueSource(strings = {"  ", "\t", "\n"})
    void isNullOrBlank(String str) {
        assertTrue(str == null || str.isBlank());
    }
    
    // CsvSource - multiple parameters
    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource({
        "1, 2, 3",
        "5, 3, 8",
        "10, -2, 8",
        "0, 0, 0"
    })
    void additionTest(int a, int b, int expected) {
        assertEquals(expected, a + b);
    }
    
    // CsvFileSource - from CSV file
    @ParameterizedTest
    @CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
    void csvFileTest(String input, int expected) {
        assertEquals(expected, input.length());
    }
    
    // MethodSource - from method
    @ParameterizedTest
    @MethodSource("provideStringsForIsBlank")
    void isBlank(String input, boolean expected) {
        assertEquals(expected, input == null || input.isBlank());
    }
    
    static Stream<Arguments> provideStringsForIsBlank() {
        return Stream.of(
            Arguments.of(null, true),
            Arguments.of("", true),
            Arguments.of("  ", true),
            Arguments.of("not blank", false),
            Arguments.of("  not blank  ", false)
        );
    }
    
    // EnumSource
    enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }
    
    @ParameterizedTest
    @EnumSource(value = Day.class, names = {"SAT", "SUN"})
    void weekendDays(Day day) {
        assertTrue(day == Day.SAT || day == Day.SUN);
    }
    
    @ParameterizedTest
    @EnumSource(value = Day.class, names = {"SAT", "SUN"}, mode = EnumSource.Mode.EXCLUDE)
    void weekdays(Day day) {
        assertTrue(day != Day.SAT && day != Day.SUN);
    }
    
    // Custom argument converter
    @ParameterizedTest
    @ValueSource(strings = {"GOLD", "SILVER", "BRONZE"})
    void membershipTiers(String tier) {
        assertNotNull(MembershipTier.valueOf(tier));
    }
    
    enum MembershipTier { GOLD, SILVER, BRONZE }
}
```

---

## 5. Test Lifecycle and Nested Tests

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;
import java.util.ArrayList;
import java.util.List;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)  // One instance per class
class ShoppingCartTest {
    
    private ShoppingCart cart;
    
    @BeforeAll
    void initAll() {
        System.out.println("Initializing test suite");
    }
    
    @BeforeEach
    void createCart() {
        cart = new ShoppingCart();
    }
    
    @Test
    @DisplayName("New cart should be empty")
    void newCartIsEmpty() {
        assertTrue(cart.isEmpty());
        assertEquals(0, cart.getItemCount());
    }
    
    @Nested
    @DisplayName("When items are added")
    class WhenItemsAreAdded {
        
        @BeforeEach
        void addItem() {
            cart.addItem("Apple", 1.50, 3);
        }
        
        @Test
        @DisplayName("Cart should not be empty")
        void cartShouldNotBeEmpty() {
            assertFalse(cart.isEmpty());
        }
        
        @Test
        @DisplayName("Item count should be correct")
        void itemCountShouldBeCorrect() {
            assertEquals(3, cart.getItemCount());
        }
        
        @Test
        @DisplayName("Total should be calculated correctly")
        void totalShouldBeCalculatedCorrectly() {
            assertEquals(4.50, cart.getTotal(), 0.001);
        }
        
        @Nested
        @DisplayName("And then removed")
        class WhenItemIsRemoved {
            
            @BeforeEach
            void removeItem() {
                cart.removeItem("Apple");
            }
            
            @Test
            @DisplayName("Cart should be empty again")
            void cartShouldBeEmptyAgain() {
                assertTrue(cart.isEmpty());
            }
            
            @Test
            @DisplayName("Total should be zero")
            void totalShouldBeZero() {
                assertEquals(0.0, cart.getTotal(), 0.001);
            }
        }
    }
    
    @Nested
    @DisplayName("Discount tests")
    class DiscountTests {
        
        @Test
        @DisplayName("10% discount on orders over 100")
        void discountOnLargeOrders() {
            cart.addItem("Laptop", 50.0, 3); // 150.0
            cart.applyDiscount(0.10);
            assertEquals(135.0, cart.getTotal(), 0.001);
        }
        
        @Test
        @DisplayName("No discount on small orders")
        void noDiscountOnSmallOrders() {
            cart.addItem("Book", 20.0, 1);
            cart.applyDiscount(0.10); // Should not apply
            assertEquals(20.0, cart.getTotal(), 0.001);
        }
    }
}

// ShoppingCart implementation
class ShoppingCart {
    record CartItem(String name, double price, int quantity) {}
    
    private final List<CartItem> items = new ArrayList<>();
    private double discountRate = 0;
    
    public void addItem(String name, double price, int quantity) {
        items.add(new CartItem(name, price, quantity));
    }
    
    public void removeItem(String name) {
        items.removeIf(item -> item.name().equals(name));
    }
    
    public boolean isEmpty() {
        return items.isEmpty();
    }
    
    public int getItemCount() {
        return items.stream().mapToInt(CartItem::quantity).sum();
    }
    
    public double getTotal() {
        double total = items.stream()
            .mapToDouble(item -> item.price() * item.quantity())
            .sum();
        if (total > 100 && discountRate > 0) {
            total *= (1 - discountRate);
        }
        return total;
    }
    
    public void applyDiscount(double rate) {
        this.discountRate = rate;
    }
}
```

---

## 6. Custom Extensions

```java
import org.junit.jupiter.api.extension.*;
import java.lang.annotation.*;
import java.lang.reflect.Method;
import java.util.logging.Logger;

// Custom annotation
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@ExtendWith(TimingExtension.class)
@interface Timed {}

// Custom Extension - measures test execution time
class TimingExtension implements BeforeTestExecutionCallback, AfterTestExecutionCallback {
    
    private static final Logger logger = Logger.getLogger(TimingExtension.class.getName());
    private static final String START_TIME = "start time";
    
    @Override
    public void beforeTestExecution(ExtensionContext context) {
        getStore(context).put(START_TIME, System.currentTimeMillis());
    }
    
    @Override
    public void afterTestExecution(ExtensionContext context) {
        Method testMethod = context.getRequiredTestMethod();
        boolean slow = testMethod.isAnnotationPresent(Timed.class);
        
        long startTime = getStore(context).remove(START_TIME, long.class);
        long duration = System.currentTimeMillis() - startTime;
        
        logger.info(() -> String.format("Method [%s] took %d ms", testMethod.getName(), duration));
        
        if (slow && duration > 100) {
            logger.warning(() -> String.format("Test took too long: %d ms", duration));
        }
    }
    
    private ExtensionContext.Store getStore(ExtensionContext context) {
        return context.getStore(
            ExtensionContext.Namespace.create(getClass(), context.getRequiredTestMethod())
        );
    }
}

// Custom Extension - provides test data
class DatabaseExtension implements BeforeEachCallback, AfterEachCallback {
    
    private TestDatabase database;
    
    @Override
    public void beforeEach(ExtensionContext context) {
        database = new TestDatabase();
        database.connect();
        database.seedTestData();
        
        // Store in context for test access
        context.getStore(ExtensionContext.Namespace.GLOBAL)
            .put("database", database);
    }
    
    @Override
    public void afterEach(ExtensionContext context) {
        database.cleanup();
        database.disconnect();
    }
}

// Usage
@ExtendWith({TimingExtension.class, DatabaseExtension.class})
class ExtensionUsageTest {
    
    @Test
    @Timed
    void testWithTiming() throws Exception {
        Thread.sleep(50);
        assertTrue(true);
    }
}

// Simple TestDatabase stub
class TestDatabase {
    void connect() { System.out.println("DB Connected"); }
    void disconnect() { System.out.println("DB Disconnected"); }
    void seedTestData() { System.out.println("Test data seeded"); }
    void cleanup() { System.out.println("Test data cleaned"); }
}
```

---

## 7. Mockito Basics

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import java.util.List;
import java.util.Optional;

@ExtendWith(MockitoExtension.class)
class MockitoBasicsTest {
    
    // @Mock - creates a mock object
    @Mock
    private UserRepository userRepository;
    
    // @InjectMocks - creates instance with mocks injected
    @InjectMocks
    private UserService userService;
    
    @Test
    void testFindUser() {
        // Arrange - stubbing
        User expectedUser = new User(1L, "John", "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Optional.of(expectedUser));
        
        // Act
        Optional<User> result = userService.findUser(1L);
        
        // Assert
        assertTrue(result.isPresent());
        assertEquals("John", result.get().getName());
        
        // Verify interaction
        verify(userRepository, times(1)).findById(1L);
    }
    
    @Test
    void testCreateUser() {
        // Arrange
        User newUser = new User(null, "Jane", "jane@example.com");
        User savedUser = new User(2L, "Jane", "jane@example.com");
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        
        // Act
        User result = userService.createUser("Jane", "jane@example.com");
        
        // Assert
        assertNotNull(result.getId());
        assertEquals("Jane", result.getName());
        
        // Verify with argument captor
        ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
        verify(userRepository).save(userCaptor.capture());
        assertEquals("jane@example.com", userCaptor.getValue().getEmail());
    }
    
    @Test
    void testUserNotFound() {
        when(userRepository.findById(anyLong())).thenReturn(Optional.empty());
        
        assertThrows(UserNotFoundException.class, () -> userService.getUser(999L));
        
        verify(userRepository).findById(999L);
        verifyNoMoreInteractions(userRepository);
    }
    
    @Test
    void testDeleteUser() {
        // doNothing for void methods
        doNothing().when(userRepository).deleteById(anyLong());
        
        userService.deleteUser(1L);
        
        verify(userRepository).deleteById(1L);
    }
    
    @Test
    void testExceptionStubbing() {
        when(userRepository.findById(anyLong()))
            .thenThrow(new DatabaseException("Connection failed"));
        
        assertThrows(DatabaseException.class, () -> userService.findUser(1L));
    }
    
    @Test
    void testReturnDifferentValues() {
        // Return different values on successive calls
        when(userRepository.count())
            .thenReturn(0L)
            .thenReturn(1L)
            .thenReturn(2L);
        
        assertEquals(0L, userRepository.count());
        assertEquals(1L, userRepository.count());
        assertEquals(2L, userRepository.count());
        assertEquals(2L, userRepository.count()); // stays at last value
    }
    
    @Test
    void testSpy() {
        // Spy - partial mocking (real methods unless stubbed)
        List<String> spyList = spy(new java.util.ArrayList<>());
        
        spyList.add("one");
        spyList.add("two");
        
        assertEquals(2, spyList.size()); // Real method
        
        // Stub specific method
        doReturn(100).when(spyList).size();
        assertEquals(100, spyList.size()); // Stubbed
        
        verify(spyList, times(2)).add(anyString());
    }
    
    @Test
    void testArgumentMatchers() {
        when(userRepository.findByName(anyString())).thenReturn(Optional.empty());
        when(userRepository.findByEmail(eq("exact@example.com"))).thenReturn(Optional.empty());
        when(userRepository.findByAge(intThat(age -> age >= 18))).thenReturn(List.of());
        
        userRepository.findByName("anything");
        userRepository.findByEmail("exact@example.com");
        userRepository.findByAge(25);
        
        verify(userRepository).findByName(anyString());
        verify(userRepository).findByEmail("exact@example.com");
        verify(userRepository).findByAge(intThat(age -> age >= 18));
    }
}

// Supporting classes
record User(Long id, String name, String email) {}

class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) { super(message); }
}

class DatabaseException extends RuntimeException {
    public DatabaseException(String message) { super(message); }
}

interface UserRepository {
    Optional<User> findById(Long id);
    User save(User user);
    void deleteById(Long id);
    long count();
    Optional<User> findByName(String name);
    Optional<User> findByEmail(String email);
    List<User> findByAge(int age);
}

class UserService {
    private final UserRepository userRepository;
    
    UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    Optional<User> findUser(Long id) {
        return userRepository.findById(id);
    }
    
    User getUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }
    
    User createUser(String name, String email) {
        return userRepository.save(new User(null, name, email));
    }
    
    void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

---

## 8. Advanced Mockito

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import java.util.function.Consumer;

@ExtendWith(MockitoExtension.class)
class AdvancedMockitoTest {
    
    @Mock
    private PaymentGateway paymentGateway;
    
    @Mock
    private EmailService emailService;
    
    @Mock
    private OrderRepository orderRepository;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void testCallbackMocking() {
        // Callback/Consumer mocking
        doAnswer(invocation -> {
            Consumer<String> callback = invocation.getArgument(1);
            callback.accept("TXN-12345");
            return null;
        }).when(paymentGateway).processAsync(any(), any());
        
        String[] capturedTxnId = {null};
        orderService.placeOrderAsync(new Order("ORD-001", 100.0), txnId -> {
            capturedTxnId[0] = txnId;
        });
        
        assertEquals("TXN-12345", capturedTxnId[0]);
    }
    
    @Test
    void testOrderOfInteractions() {
        // InOrder - verify interaction order
        Order order = new Order("ORD-002", 200.0);
        when(paymentGateway.charge(anyDouble())).thenReturn("TXN-001");
        
        orderService.placeOrder(order);
        
        InOrder inOrder = inOrder(paymentGateway, emailService, orderRepository);
        inOrder.verify(paymentGateway).charge(200.0);
        inOrder.verify(orderRepository).save(any(Order.class));
        inOrder.verify(emailService).sendConfirmation(anyString());
    }
    
    @Test
    void testVerifyNoInteractions() {
        when(paymentGateway.charge(anyDouble()))
            .thenThrow(new PaymentException("Card declined"));
        
        assertThrows(PaymentException.class, 
            () -> orderService.placeOrder(new Order("ORD-003", 50.0)));
        
        // Email should NOT be sent if payment fails
        verifyNoInteractions(emailService);
    }
    
    @Test
    void testMockStaticMethod() {
        // Mockito 3.4+ can mock static methods
        try (MockedStatic<OrderValidator> mocked = mockStatic(OrderValidator.class)) {
            mocked.when(() -> OrderValidator.validate(any())).thenReturn(true);
            
            Order order = new Order("ORD-004", 75.0);
            when(paymentGateway.charge(anyDouble())).thenReturn("TXN-004");
            
            orderService.placeOrder(order);
            
            mocked.verify(() -> OrderValidator.validate(order));
        }
    }
    
    @Test
    void testMockConstructor() {
        // Mock object construction
        try (MockedConstruction<PaymentGateway> mocked = 
                mockConstruction(PaymentGateway.class, (mock, context) -> {
                    when(mock.charge(anyDouble())).thenReturn("TXN-NEW");
                })) {
            
            PaymentGateway gateway = new PaymentGateway();
            assertEquals("TXN-NEW", gateway.charge(100.0));
        }
    }
    
    @Test
    void testCapturingMultipleInvocations() {
        // Capture multiple calls
        Order order1 = new Order("ORD-005", 100.0);
        Order order2 = new Order("ORD-006", 200.0);
        
        when(paymentGateway.charge(anyDouble())).thenReturn("TXN-A", "TXN-B");
        
        orderService.placeOrder(order1);
        orderService.placeOrder(order2);
        
        ArgumentCaptor<Double> amountCaptor = ArgumentCaptor.forClass(Double.class);
        verify(paymentGateway, times(2)).charge(amountCaptor.capture());
        
        assertEquals(2, amountCaptor.getAllValues().size());
        assertEquals(100.0, amountCaptor.getAllValues().get(0));
        assertEquals(200.0, amountCaptor.getAllValues().get(1));
    }
    
    @Test
    void testCustomAnswers() {
        // thenAnswer for complex behavior
        when(paymentGateway.charge(anyDouble())).thenAnswer(invocation -> {
            double amount = invocation.getArgument(0);
            if (amount > 1000) {
                throw new PaymentException("Amount exceeds limit");
            }
            return "TXN-" + (int)(amount * 100);
        });
        
        assertEquals("TXN-10000", paymentGateway.charge(100.0));
        assertEquals("TXN-25000", paymentGateway.charge(250.0));
        assertThrows(PaymentException.class, () -> paymentGateway.charge(1500.0));
    }
}

// Supporting classes for order tests
record Order(String id, double amount) {}

interface PaymentGateway {
    String charge(double amount);
    void processAsync(Order order, Consumer<String> callback);
}

interface EmailService {
    void sendConfirmation(String email);
}

interface OrderRepository {
    void save(Order order);
}

class PaymentException extends RuntimeException {
    public PaymentException(String message) { super(message); }
}

class OrderValidator {
    public static boolean validate(Order order) {
        return order != null && order.amount() > 0;
    }
}

class OrderService {
    private final PaymentGateway paymentGateway;
    private final EmailService emailService;
    private final OrderRepository orderRepository;
    
    OrderService(PaymentGateway pg, EmailService es, OrderRepository or) {
        this.paymentGateway = pg;
        this.emailService = es;
        this.orderRepository = or;
    }
    
    void placeOrder(Order order) {
        String txnId = paymentGateway.charge(order.amount());
        orderRepository.save(order);
        emailService.sendConfirmation("customer@example.com");
    }
    
    void placeOrderAsync(Order order, Consumer<String> callback) {
        paymentGateway.processAsync(order, callback);
    }
}
```

---

## 9. Test-Driven Development (TDD)

TDD cycle: **Red → Green → Refactor**

```java
// TDD Example: Building a Password Validator

// STEP 1: RED - Write failing test first
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class PasswordValidatorTDDTest {
    
    private PasswordValidator validator;
    
    @BeforeEach
    void setUp() {
        validator = new PasswordValidator();
    }
    
    // Red: Test 1 - Minimum length
    @Test
    void passwordMustBeAtLeast8Characters() {
        assertFalse(validator.isValid("Short1!"));
        assertTrue(validator.isValid("LongPass1!"));
    }
    
    // Red: Test 2 - Must contain uppercase
    @Test
    void passwordMustContainUppercase() {
        assertFalse(validator.isValid("lowercase1!"));
        assertTrue(validator.isValid("Uppercase1!"));
    }
    
    // Red: Test 3 - Must contain lowercase
    @Test
    void passwordMustContainLowercase() {
        assertFalse(validator.isValid("UPPERCASE1!"));
        assertTrue(validator.isValid("Uppercase1!"));
    }
    
    // Red: Test 4 - Must contain digit
    @Test
    void passwordMustContainDigit() {
        assertFalse(validator.isValid("NoDigits!!"));
        assertTrue(validator.isValid("WithDigit1!"));
    }
    
    // Red: Test 5 - Must contain special character
    @Test
    void passwordMustContainSpecialCharacter() {
        assertFalse(validator.isValid("NoSpecial1"));
        assertTrue(validator.isValid("Special1!"));
    }
    
    // Red: Test 6 - null/empty handling
    @Test
    void nullOrEmptyPasswordIsInvalid() {
        assertFalse(validator.isValid(null));
        assertFalse(validator.isValid(""));
    }
    
    // Red: Test 7 - Error messages
    @Test
    void validationErrorMessages() {
        ValidationResult result = validator.validate("short");
        assertFalse(result.isValid());
        assertTrue(result.getErrors().contains("Password must be at least 8 characters"));
        assertTrue(result.getErrors().contains("Password must contain at least one uppercase letter"));
        assertTrue(result.getErrors().contains("Password must contain at least one digit"));
        assertTrue(result.getErrors().contains("Password must contain at least one special character"));
    }
    
    @Test
    void strongPasswordPassesAll() {
        ValidationResult result = validator.validate("StrongP@ss1");
        assertTrue(result.isValid());
        assertTrue(result.getErrors().isEmpty());
    }
}

// STEP 2: GREEN - Minimal implementation to pass tests
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

class ValidationResult {
    private final boolean valid;
    private final List<String> errors;
    
    ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = Collections.unmodifiableList(errors);
    }
    
    boolean isValid() { return valid; }
    List<String> getErrors() { return errors; }
}

class PasswordValidator {
    
    boolean isValid(String password) {
        return validate(password).isValid();
    }
    
    ValidationResult validate(String password) {
        List<String> errors = new ArrayList<>();
        
        if (password == null || password.isEmpty()) {
            errors.add("Password must not be null or empty");
            return new ValidationResult(false, errors);
        }
        
        if (password.length() < 8) {
            errors.add("Password must be at least 8 characters");
        }
        
        if (!password.chars().anyMatch(Character::isUpperCase)) {
            errors.add("Password must contain at least one uppercase letter");
        }
        
        if (!password.chars().anyMatch(Character::isLowerCase)) {
            errors.add("Password must contain at least one lowercase letter");
        }
        
        if (!password.chars().anyMatch(Character::isDigit)) {
            errors.add("Password must contain at least one digit");
        }
        
        if (!password.chars().anyMatch(c -> "!@#$%^&*()_+-=[]{}|;:,.<>?".indexOf(c) >= 0)) {
            errors.add("Password must contain at least one special character");
        }
        
        return new ValidationResult(errors.isEmpty(), errors);
    }
}

// STEP 3: REFACTOR - Improve code quality while keeping tests green
// (Using Strategy pattern for rules)
import java.util.function.Predicate;

class RefactoredPasswordValidator {
    
    record ValidationRule(Predicate<String> check, String errorMessage) {}
    
    private static final List<ValidationRule> RULES = List.of(
        new ValidationRule(
            p -> p.length() >= 8,
            "Password must be at least 8 characters"
        ),
        new ValidationRule(
            p -> p.chars().anyMatch(Character::isUpperCase),
            "Password must contain at least one uppercase letter"
        ),
        new ValidationRule(
            p -> p.chars().anyMatch(Character::isLowerCase),
            "Password must contain at least one lowercase letter"
        ),
        new ValidationRule(
            p -> p.chars().anyMatch(Character::isDigit),
            "Password must contain at least one digit"
        ),
        new ValidationRule(
            p -> p.chars().anyMatch(c -> "!@#$%^&*()_+-=[]{}|;:,.<>?".indexOf(c) >= 0),
            "Password must contain at least one special character"
        )
    );
    
    public ValidationResult validate(String password) {
        if (password == null || password.isEmpty()) {
            return new ValidationResult(false, 
                List.of("Password must not be null or empty"));
        }
        
        List<String> errors = RULES.stream()
            .filter(rule -> !rule.check().test(password))
            .map(ValidationRule::errorMessage)
            .toList();
        
        return new ValidationResult(errors.isEmpty(), errors);
    }
    
    public boolean isValid(String password) {
        return validate(password).isValid();
    }
}
```

---

## 10. Integration Testing

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;
import java.sql.*;
import java.util.List;
import java.util.ArrayList;

// Integration test with real database (H2 in-memory)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class UserRepositoryIntegrationTest {
    
    private Connection connection;
    private UserJdbcRepository repository;
    
    @BeforeAll
    void setUpDatabase() throws SQLException {
        // H2 in-memory database
        connection = DriverManager.getConnection("jdbc:h2:mem:testdb", "sa", "");
        
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("""
                CREATE TABLE users (
                    id BIGINT AUTO_INCREMENT PRIMARY KEY,
                    name VARCHAR(100) NOT NULL,
                    email VARCHAR(255) UNIQUE NOT NULL,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """);
        }
        
        repository = new UserJdbcRepository(connection);
    }
    
    @BeforeEach
    void clearData() throws SQLException {
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("DELETE FROM users");
        }
    }
    
    @AfterAll
    void tearDownDatabase() throws SQLException {
        if (connection != null) {
            connection.close();
        }
    }
    
    @Test
    void saveAndFindUser() throws Exception {
        UserEntity user = new UserEntity(null, "Alice", "alice@example.com");
        UserEntity saved = repository.save(user);
        
        assertNotNull(saved.id());
        
        UserEntity found = repository.findById(saved.id()).orElseThrow();
        assertEquals("Alice", found.name());
        assertEquals("alice@example.com", found.email());
    }
    
    @Test
    void findAllUsers() throws Exception {
        repository.save(new UserEntity(null, "Bob", "bob@example.com"));
        repository.save(new UserEntity(null, "Charlie", "charlie@example.com"));
        
        List<UserEntity> users = repository.findAll();
        assertEquals(2, users.size());
    }
    
    @Test
    void updateUser() throws Exception {
        UserEntity saved = repository.save(new UserEntity(null, "Dave", "dave@example.com"));
        repository.update(new UserEntity(saved.id(), "David", "david@example.com"));
        
        UserEntity updated = repository.findById(saved.id()).orElseThrow();
        assertEquals("David", updated.name());
    }
    
    @Test
    void deleteUser() throws Exception {
        UserEntity saved = repository.save(new UserEntity(null, "Eve", "eve@example.com"));
        repository.deleteById(saved.id());
        
        assertTrue(repository.findById(saved.id()).isEmpty());
    }
    
    @Test
    void duplicateEmailThrowsException() {
        assertDoesNotThrow(() -> repository.save(
            new UserEntity(null, "Frank", "frank@example.com")));
        
        assertThrows(SQLException.class, () -> repository.save(
            new UserEntity(null, "Frank2", "frank@example.com")));
    }
}

record UserEntity(Long id, String name, String email) {}

class UserJdbcRepository {
    private final Connection connection;
    
    UserJdbcRepository(Connection connection) {
        this.connection = connection;
    }
    
    UserEntity save(UserEntity user) throws SQLException {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
        try (PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, user.name());
            ps.setString(2, user.email());
            ps.executeUpdate();
            
            try (ResultSet rs = ps.getGeneratedKeys()) {
                rs.next();
                return new UserEntity(rs.getLong(1), user.name(), user.email());
            }
        }
    }
    
    java.util.Optional<UserEntity> findById(Long id) throws SQLException {
        String sql = "SELECT * FROM users WHERE id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setLong(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return java.util.Optional.of(mapRow(rs));
                }
                return java.util.Optional.empty();
            }
        }
    }
    
    List<UserEntity> findAll() throws SQLException {
        List<UserEntity> users = new ArrayList<>();
        try (Statement stmt = connection.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
            while (rs.next()) users.add(mapRow(rs));
        }
        return users;
    }
    
    void update(UserEntity user) throws SQLException {
        String sql = "UPDATE users SET name = ?, email = ? WHERE id = ?";
        try (PreparedStatement ps = connection.prepareStatement(sql)) {
            ps.setString(1, user.name());
            ps.setString(2, user.email());
            ps.setLong(3, user.id());
            ps.executeUpdate();
        }
    }
    
    void deleteById(Long id) throws SQLException {
        try (PreparedStatement ps = 
                connection.prepareStatement("DELETE FROM users WHERE id = ?")) {
            ps.setLong(1, id);
            ps.executeUpdate();
        }
    }
    
    private UserEntity mapRow(ResultSet rs) throws SQLException {
        return new UserEntity(rs.getLong("id"), rs.getString("name"), rs.getString("email"));
    }
}
```

---

## 11. Complete Test Suite - E-Commerce Example

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;
import java.util.*;
import java.math.BigDecimal;
import java.util.stream.Stream;

// Domain objects
record Product(String id, String name, BigDecimal price, int stock) {}
record OrderItem(Product product, int quantity) {
    BigDecimal subtotal() {
        return product.price().multiply(BigDecimal.valueOf(quantity));
    }
}

class ShoppingCart2 {
    private final Map<String, OrderItem> items = new LinkedHashMap<>();
    
    void addProduct(Product product, int quantity) {
        if (product == null) throw new IllegalArgumentException("Product cannot be null");
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
        if (quantity > product.stock()) throw new InsufficientStockException(product.name());
        
        items.merge(product.id(), 
            new OrderItem(product, quantity),
            (existing, newItem) -> new OrderItem(product, existing.quantity() + newItem.quantity())
        );
    }
    
    void removeProduct(String productId) {
        items.remove(productId);
    }
    
    BigDecimal getTotal() {
        return items.values().stream()
            .map(OrderItem::subtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    List<OrderItem> getItems() {
        return List.copyOf(items.values());
    }
    
    boolean isEmpty() { return items.isEmpty(); }
    int getItemCount() { return items.size(); }
}

class InsufficientStockException extends RuntimeException {
    InsufficientStockException(String productName) {
        super("Insufficient stock for: " + productName);
    }
}

// Interfaces
interface ProductRepository {
    Optional<Product> findById(String id);
    List<Product> findAll();
    Product save(Product product);
}

interface PricingService {
    BigDecimal calculatePrice(Product product, int quantity);
    BigDecimal applyDiscount(BigDecimal total, String couponCode);
}

// Service
class CartService {
    private final ProductRepository productRepository;
    private final PricingService pricingService;
    private final ShoppingCart2 cart = new ShoppingCart2();
    
    CartService(ProductRepository productRepository, PricingService pricingService) {
        this.productRepository = productRepository;
        this.pricingService = pricingService;
    }
    
    void addToCart(String productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException("Product not found: " + productId));
        cart.addProduct(product, quantity);
    }
    
    BigDecimal checkout(String couponCode) {
        if (cart.isEmpty()) throw new EmptyCartException("Cart is empty");
        BigDecimal total = cart.getTotal();
        if (couponCode != null) {
            total = pricingService.applyDiscount(total, couponCode);
        }
        return total;
    }
    
    ShoppingCart2 getCart() { return cart; }
}

class ProductNotFoundException extends RuntimeException {
    ProductNotFoundException(String message) { super(message); }
}
class EmptyCartException extends RuntimeException {
    EmptyCartException(String message) { super(message); }
}

// === Complete Test Suite ===
@ExtendWith(MockitoExtension.class)
class CartServiceTest {
    
    @Mock private ProductRepository productRepository;
    @Mock private PricingService pricingService;
    @InjectMocks private CartService cartService;
    
    private static final Product LAPTOP = new Product("LAPTOP-001", "Laptop Pro", 
        new BigDecimal("999.99"), 10);
    private static final Product MOUSE = new Product("MOUSE-001", "Wireless Mouse", 
        new BigDecimal("29.99"), 50);
    
    @Nested
    @DisplayName("Adding products to cart")
    class AddToCartTests {
        
        @Test
        @DisplayName("Successfully add product to cart")
        void addProductSuccessfully() {
            when(productRepository.findById("LAPTOP-001")).thenReturn(Optional.of(LAPTOP));
            
            cartService.addToCart("LAPTOP-001", 1);
            
            assertFalse(cartService.getCart().isEmpty());
            assertEquals(1, cartService.getCart().getItemCount());
        }
        
        @Test
        @DisplayName("Throw exception for non-existent product")
        void throwExceptionForNonExistentProduct() {
            when(productRepository.findById("INVALID")).thenReturn(Optional.empty());
            
            assertThrows(ProductNotFoundException.class, 
                () -> cartService.addToCart("INVALID", 1));
        }
        
        @Test
        @DisplayName("Throw exception for insufficient stock")
        void throwExceptionForInsufficientStock() {
            Product lowStockProduct = new Product("LOW-001", "Limited Item", 
                new BigDecimal("50.00"), 2);
            when(productRepository.findById("LOW-001")).thenReturn(Optional.of(lowStockProduct));
            
            assertThrows(InsufficientStockException.class, 
                () -> cartService.addToCart("LOW-001", 5));
        }
        
        @ParameterizedTest(name = "Add {1} units of product {0}")
        @CsvSource({
            "LAPTOP-001, 1, 999.99",
            "MOUSE-001, 2, 59.98"
        })
        @DisplayName("Cart total should match product price * quantity")
        void cartTotalMatchesProductPriceTimesQuantity(String id, int qty, String expected) {
            Product product = id.equals("LAPTOP-001") ? LAPTOP : MOUSE;
            when(productRepository.findById(id)).thenReturn(Optional.of(product));
            
            cartService.addToCart(id, qty);
            
            assertEquals(new BigDecimal(expected), cartService.getCart().getTotal());
        }
    }
    
    @Nested
    @DisplayName("Checkout process")
    class CheckoutTests {
        
        @BeforeEach
        void addItemsToCart() {
            when(productRepository.findById("LAPTOP-001")).thenReturn(Optional.of(LAPTOP));
            when(productRepository.findById("MOUSE-001")).thenReturn(Optional.of(MOUSE));
            cartService.addToCart("LAPTOP-001", 1);
            cartService.addToCart("MOUSE-001", 2);
        }
        
        @Test
        @DisplayName("Checkout without coupon returns full total")
        void checkoutWithoutCoupon() {
            BigDecimal expected = new BigDecimal("1059.97"); // 999.99 + 2*29.99
            assertEquals(expected, cartService.checkout(null));
        }
        
        @Test
        @DisplayName("Checkout with valid coupon applies discount")
        void checkoutWithValidCoupon() {
            BigDecimal originalTotal = new BigDecimal("1059.97");
            BigDecimal discountedTotal = new BigDecimal("953.97");
            
            when(pricingService.applyDiscount(originalTotal, "SAVE10"))
                .thenReturn(discountedTotal);
            
            assertEquals(discountedTotal, cartService.checkout("SAVE10"));
            verify(pricingService).applyDiscount(originalTotal, "SAVE10");
        }
        
        @Test
        @DisplayName("Checkout with empty cart throws exception")
        void checkoutWithEmptyCartThrows() {
            CartService emptyCartService = new CartService(productRepository, pricingService);
            assertThrows(EmptyCartException.class, () -> emptyCartService.checkout(null));
        }
    }
}

// Unit tests for domain objects
class ShoppingCartTest2 {
    
    @Test
    void newCartIsEmpty() {
        ShoppingCart2 cart = new ShoppingCart2();
        assertTrue(cart.isEmpty());
        assertEquals(BigDecimal.ZERO, cart.getTotal());
    }
    
    @Test
    void addProductIncreasesItemCount() {
        ShoppingCart2 cart = new ShoppingCart2();
        Product product = new Product("P001", "Test", new BigDecimal("10.00"), 100);
        
        cart.addProduct(product, 3);
        
        assertEquals(1, cart.getItemCount());
        assertEquals(new BigDecimal("30.00"), cart.getTotal());
    }
    
    @Test
    void addingSameProductTwiceAccumulatesQuantity() {
        ShoppingCart2 cart = new ShoppingCart2();
        Product product = new Product("P001", "Test", new BigDecimal("10.00"), 100);
        
        cart.addProduct(product, 2);
        cart.addProduct(product, 3);
        
        assertEquals(1, cart.getItemCount());
        assertEquals(new BigDecimal("50.00"), cart.getTotal());
    }
    
    @Test
    void removeProductRemovesFromCart() {
        ShoppingCart2 cart = new ShoppingCart2();
        Product product = new Product("P001", "Test", new BigDecimal("10.00"), 100);
        cart.addProduct(product, 1);
        
        cart.removeProduct("P001");
        
        assertTrue(cart.isEmpty());
    }
    
    @Test
    void addNullProductThrowsException() {
        ShoppingCart2 cart = new ShoppingCart2();
        assertThrows(IllegalArgumentException.class, 
            () -> cart.addProduct(null, 1));
    }
    
    @Test
    void addNegativeQuantityThrowsException() {
        ShoppingCart2 cart = new ShoppingCart2();
        Product product = new Product("P001", "Test", new BigDecimal("10.00"), 100);
        assertThrows(IllegalArgumentException.class, 
            () -> cart.addProduct(product, -1));
    }
    
    @ParameterizedTest
    @MethodSource("exceedingStockParameters")
    void addProductExceedingStockThrows(int stock, int quantity) {
        ShoppingCart2 cart = new ShoppingCart2();
        Product product = new Product("P001", "Test", new BigDecimal("10.00"), stock);
        assertThrows(InsufficientStockException.class, 
            () -> cart.addProduct(product, quantity));
    }
    
    static Stream<Arguments> exceedingStockParameters() {
        return Stream.of(
            Arguments.of(5, 6),
            Arguments.of(1, 10),
            Arguments.of(0, 1)
        );
    }
}
```

---

## 12. Test Best Practices

```java
// ✅ GOOD: Descriptive test names using BDD style
class BDDStyleTest {
    
    // GIVEN_WHEN_THEN naming convention
    @Test
    void givenEmptyCart_whenAddingProduct_thenCartShouldContainOneItem() {}
    
    @Test
    void givenInvalidCoupon_whenCheckingOut_thenShouldThrowException() {}
    
    // Or use @DisplayName with natural language
    @Test
    @DisplayName("Cart becomes empty after all items are removed")
    void cartBecomesEmptyAfterRemoval() {}
}

// ✅ GOOD: One assertion per concept (logical unit)
class SingleConceptTest {
    @Test
    void userProfileContainsCorrectData() {
        // Multiple assertions are fine if they test one concept
        UserProfile profile = createTestProfile();
        assertAll("profile",
            () -> assertEquals("John Doe", profile.getFullName()),
            () -> assertEquals("john@example.com", profile.getEmail()),
            () -> assertTrue(profile.isActive())
        );
    }
    
    private UserProfile createTestProfile() {
        return new UserProfile("John", "Doe", "john@example.com", true);
    }
}

record UserProfile(String firstName, String lastName, String email, boolean active) {
    String getFullName() { return firstName + " " + lastName; }
    String getEmail() { return email; }
    boolean isActive() { return active; }
}

// ✅ GOOD: Use test fixtures / builders
class TestFixtures {
    
    static class UserBuilder {
        private Long id = 1L;
        private String name = "Test User";
        private String email = "test@example.com";
        private boolean active = true;
        
        UserBuilder withId(Long id) { this.id = id; return this; }
        UserBuilder withName(String name) { this.name = name; return this; }
        UserBuilder withEmail(String email) { this.email = email; return this; }
        UserBuilder inactive() { this.active = false; return this; }
        
        UserProfile build() {
            return new UserProfile(name.split(" ")[0], 
                name.contains(" ") ? name.split(" ")[1] : "",
                email, active);
        }
    }
    
    @Test
    void withBuilderPattern() {
        UserProfile activeUser = new UserBuilder().build();
        UserProfile inactiveUser = new UserBuilder().withName("Inactive User").inactive().build();
        
        assertTrue(activeUser.isActive());
        assertFalse(inactiveUser.isActive());
    }
}

// ✅ GOOD: Fast isolated tests (no external dependencies)
// ❌ BAD: Tests that depend on database, network, file system (without proper setup)
// ❌ BAD: Tests that share mutable state between test methods
// ❌ BAD: Tests with random data without seeding
// ❌ BAD: Tests that test implementation details instead of behavior
```

---

## 13. Code Coverage

```java
// Add JaCoCo plugin in pom.xml for code coverage:
/*
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>prepare-package</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
*/

// Example: Class designed for 100% coverage
class EmailValidator2 {
    
    private static final String EMAIL_REGEX = 
        "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$";
    
    public ValidationResult2 validate(String email) {
        if (email == null) {
            return ValidationResult2.invalid("Email cannot be null");
        }
        if (email.isBlank()) {
            return ValidationResult2.invalid("Email cannot be blank");
        }
        if (!email.matches(EMAIL_REGEX)) {
            return ValidationResult2.invalid("Invalid email format: " + email);
        }
        return ValidationResult2.valid();
    }
}

record ValidationResult2(boolean valid, String error) {
    static ValidationResult2 valid() { return new ValidationResult2(true, null); }
    static ValidationResult2 invalid(String error) { return new ValidationResult2(false, error); }
    boolean isValid() { return valid; }
}

// Tests achieving 100% coverage
class EmailValidator2Test {
    private final EmailValidator2 validator = new EmailValidator2();
    
    @Test void nullEmailIsInvalid() {
        assertFalse(validator.validate(null).isValid());
    }
    
    @Test void emptyEmailIsInvalid() {
        assertFalse(validator.validate("").isValid());
    }
    
    @Test void blankEmailIsInvalid() {
        assertFalse(validator.validate("   ").isValid());
    }
    
    @ParameterizedTest
    @ValueSource(strings = {"notanemail", "missing@", "@domain.com", "a@b", "a@b.c"})
    void invalidFormatIsInvalid(String email) {
        assertFalse(validator.validate(email).isValid());
    }
    
    @ParameterizedTest
    @ValueSource(strings = {"user@example.com", "test.user@company.org", "hello+tag@mail.co"})
    void validEmailIsValid(String email) {
        assertTrue(validator.validate(email).isValid());
    }
}
```

---

## สรุป Part 019

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|---------------|
| JUnit 5 Annotations | @Test, @BeforeEach, @AfterEach, @BeforeAll, @AfterAll, @Disabled, @Tag |
| Assertions | assertEquals, assertTrue, assertThrows, assertAll, assertTimeout |
| Parameterized Tests | @ValueSource, @CsvSource, @MethodSource, @EnumSource |
| Nested Tests | @Nested for organized test structure |
| Extensions | Custom BeforeTestExecutionCallback, AfterTestExecutionCallback |
| Mockito Basics | @Mock, @InjectMocks, when/thenReturn, verify |
| Advanced Mockito | ArgumentCaptor, InOrder, doAnswer, MockedStatic |
| TDD | Red → Green → Refactor cycle |
| Integration Tests | Real database with H2 |
| Code Coverage | JaCoCo integration |

---

**Part 020:** Maven & Gradle - Build Tools สำหรับ Java Projects
- Maven lifecycle, POM structure, dependencies, plugins
- Gradle build scripts, tasks, dependency management
- Multi-module projects
- Publishing artifacts
