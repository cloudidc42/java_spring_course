# Part 042: Java 21 Modern Features

## Overview

Java 21 is a Long-Term Support (LTS) release packed with features that fundamentally change how we write Java. This part covers Records, Sealed Classes, Pattern Matching, Virtual Threads, Structured Concurrency, and practical integration with Spring Boot.

---

## 1. Records

Records are immutable data carriers with built-in equals, hashCode, and toString. They eliminate boilerplate for value objects.

### 1.1 Basic Record

```java
package com.example.modern.records;

// Basic record
public record Point(double x, double y) {
    // All fields are private, final, and immutable
    // Constructor, getters (x(), y()), equals(), hashCode(), toString() auto-generated
}
```

### 1.2 Compact Canonical Constructor

```java
package com.example.modern.records;

import java.util.Objects;

public record PersonRecord(String firstName, String lastName, int age) {

    // Compact canonical constructor - great for validation
    public PersonRecord {
        Objects.requireNonNull(firstName, "firstName must not be null");
        Objects.requireNonNull(lastName, "lastName must not be null");
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Age must be between 0 and 150, got: " + age);
        }
        // Can also normalize: firstName = firstName.trim()
        firstName = firstName.strip();
        lastName = lastName.strip();
    }

    // Custom instance methods
    public String fullName() {
        return firstName + " " + lastName;
    }

    public boolean isAdult() {
        return age >= 18;
    }

    // Static factory method
    public static PersonRecord of(String firstName, String lastName, int age) {
        return new PersonRecord(firstName, lastName, age);
    }
}
```

### 1.3 Records with Custom Overrides

```java
package com.example.modern.records;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public record Money(BigDecimal amount, String currency) {

    // Compact constructor with normalization
    public Money {
        Objects.requireNonNull(amount, "amount required");
        Objects.requireNonNull(currency, "currency required");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        // Normalize to 2 decimal places
        amount = amount.setScale(2, RoundingMode.HALF_UP);
        currency = currency.toUpperCase();
    }

    // Business methods
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(double factor) {
        return new Money(
            this.amount.multiply(BigDecimal.valueOf(factor)),
            this.currency
        );
    }

    public boolean isGreaterThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare different currencies");
        }
        return this.amount.compareTo(other.amount) > 0;
    }

    @Override
    public String toString() {
        return String.format("%s %s", amount.toPlainString(), currency);
    }

    // Static factory methods
    public static Money usd(double amount) {
        return new Money(BigDecimal.valueOf(amount), "USD");
    }

    public static Money eur(double amount) {
        return new Money(BigDecimal.valueOf(amount), "EUR");
    }
}
```

### 1.4 Records in Spring Boot

```java
package com.example.modern.dto;

import jakarta.validation.constraints.*;
import com.fasterxml.jackson.annotation.JsonFormat;
import java.time.LocalDate;

// DTO record with Bean Validation
public record CreateUserRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    String username,

    @Email(message = "Invalid email format")
    @NotBlank
    String email,

    @NotBlank
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password,

    @NotNull
    @Past(message = "Date of birth must be in the past")
    @JsonFormat(pattern = "yyyy-MM-dd")
    LocalDate dateOfBirth
) {}
```

```java
package com.example.modern.dto;

import java.time.Instant;

// Response record
public record UserResponse(
    Long id,
    String username,
    String email,
    Instant createdAt
) {
    // Hide sensitive data with factory method
    public static UserResponse from(com.example.modern.entity.User user) {
        return new UserResponse(
            user.getId(),
            user.getUsername(),
            user.getEmail(),
            user.getCreatedAt()
        );
    }
}
```

```java
package com.example.modern.controller;

import com.example.modern.dto.CreateUserRequest;
import com.example.modern.dto.UserResponse;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;
import java.util.List;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        // Request is automatically validated via @Valid
        System.out.println("Creating user: " + request.username());
        // ... service call
        return ResponseEntity.ok(new UserResponse(1L, request.username(), request.email(), null));
    }
}
```

---

## 2. Sealed Classes and Pattern Matching

### 2.1 Sealed Classes

```java
package com.example.modern.sealed;

// Sealed class - restricts which classes can extend it
public sealed interface Shape
    permits Circle, Rectangle, Triangle, CompositeShape {

    double area();
    double perimeter();
    String name();
}
```

```java
package com.example.modern.sealed;

public record Circle(double radius) implements Shape {

    public Circle {
        if (radius <= 0) throw new IllegalArgumentException("Radius must be positive");
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }

    @Override
    public String name() {
        return "Circle(r=" + radius + ")";
    }
}
```

```java
package com.example.modern.sealed;

public record Rectangle(double width, double height) implements Shape {

    public Rectangle {
        if (width <= 0 || height <= 0) {
            throw new IllegalArgumentException("Dimensions must be positive");
        }
    }

    @Override
    public double area() {
        return width * height;
    }

    @Override
    public double perimeter() {
        return 2 * (width + height);
    }

    @Override
    public String name() {
        return String.format("Rectangle(%sx%s)", width, height);
    }

    public boolean isSquare() {
        return width == height;
    }
}
```

```java
package com.example.modern.sealed;

public record Triangle(double a, double b, double c) implements Shape {

    public Triangle {
        if (a <= 0 || b <= 0 || c <= 0) {
            throw new IllegalArgumentException("Sides must be positive");
        }
        if (a + b <= c || a + c <= b || b + c <= a) {
            throw new IllegalArgumentException("Invalid triangle sides");
        }
    }

    @Override
    public double area() {
        // Heron's formula
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }

    @Override
    public double perimeter() {
        return a + b + c;
    }

    @Override
    public String name() {
        return String.format("Triangle(%s,%s,%s)", a, b, c);
    }
}
```

```java
package com.example.modern.sealed;

import java.util.List;

// Non-sealed allows further extension
public non-sealed class CompositeShape implements Shape {

    private final List<Shape> shapes;

    public CompositeShape(List<Shape> shapes) {
        this.shapes = List.copyOf(shapes);
    }

    @Override
    public double area() {
        return shapes.stream().mapToDouble(Shape::area).sum();
    }

    @Override
    public double perimeter() {
        return shapes.stream().mapToDouble(Shape::perimeter).sum();
    }

    @Override
    public String name() {
        return "Composite(" + shapes.size() + " shapes)";
    }
}
```

---

## 3. Switch Expressions and Pattern Matching for switch

### 3.1 Traditional Switch to Modern Switch Expression

```java
package com.example.modern.switch_expr;

public class SwitchExpressionExamples {

    // Old style (Java 8)
    public String getOldDayType(int day) {
        String type;
        switch (day) {
            case 1:
            case 7:
                type = "Weekend";
                break;
            case 2:
            case 3:
            case 4:
            case 5:
            case 6:
                type = "Weekday";
                break;
            default:
                type = "Invalid";
        }
        return type;
    }

    // New style (Java 14+): Switch expression with arrow
    public String getDayType(int day) {
        return switch (day) {
            case 1, 7 -> "Weekend";
            case 2, 3, 4, 5, 6 -> "Weekday";
            default -> "Invalid day: " + day;
        };
    }

    // Switch expression with yield
    public String getDayTypeWithComplex(int day) {
        return switch (day) {
            case 1, 7 -> {
                System.out.println("It's the weekend!");
                yield "Weekend";
            }
            case 2, 3, 4, 5, 6 -> "Weekday";
            default -> throw new IllegalArgumentException("Invalid day: " + day);
        };
    }
}
```

### 3.2 Pattern Matching for switch (Java 21)

```java
package com.example.modern.switch_expr;

import com.example.modern.sealed.Circle;
import com.example.modern.sealed.Rectangle;
import com.example.modern.sealed.Shape;
import com.example.modern.sealed.Triangle;

public class PatternMatchingSwitch {

    // Pattern matching for switch with sealed class
    public String describeShape(Shape shape) {
        return switch (shape) {
            case Circle c when c.radius() > 100 -> "Large circle with radius " + c.radius();
            case Circle c -> "Small circle with radius " + c.radius();
            case Rectangle r when r.isSquare() -> "Square with side " + r.width();
            case Rectangle r -> String.format("Rectangle %s x %s", r.width(), r.height());
            case Triangle t -> String.format("Triangle with sides %s, %s, %s", t.a(), t.b(), t.c());
            default -> "Unknown shape: " + shape.name();
        };
    }

    // Pattern matching for instanceof (Java 16+)
    public double calculateIfShape(Object obj) {
        if (obj instanceof Circle c) {
            return c.area();  // c is already cast!
        } else if (obj instanceof Rectangle r) {
            return r.area();
        }
        return 0;
    }

    // Type patterns in switch
    public String formatValue(Object value) {
        return switch (value) {
            case null -> "null";
            case Integer i -> "Integer: " + i;
            case Long l -> "Long: " + l;
            case Double d -> String.format("Double: %.2f", d);
            case String s when s.isEmpty() -> "Empty string";
            case String s -> "String: '" + s + "'";
            case int[] arr -> "int array of length " + arr.length;
            case List<?> list when list.isEmpty() -> "Empty list";
            case List<?> list -> "List with " + list.size() + " elements";
            default -> "Unknown type: " + value.getClass().getSimpleName();
        };
    }

    // Deconstruction patterns with records
    public String describePoint(Object obj) {
        return switch (obj) {
            case com.example.modern.records.Point(double x, double y) when x == 0 && y == 0 ->
                "Origin point";
            case com.example.modern.records.Point(double x, double y) when x == y ->
                "Point on diagonal: (" + x + ", " + y + ")";
            case com.example.modern.records.Point(double x, double y) ->
                "Point at (" + x + ", " + y + ")";
            default -> "Not a point";
        };
    }
}
```

---

## 4. Text Blocks

```java
package com.example.modern.textblocks;

public class TextBlockExamples {

    // Old way: String concatenation
    public String getOldJson() {
        return "{\n" +
               "  \"name\": \"John\",\n" +
               "  \"age\": 30,\n" +
               "  \"city\": \"New York\"\n" +
               "}";
    }

    // New way: Text blocks
    public String getJson() {
        return """
                {
                  "name": "John",
                  "age": 30,
                  "city": "New York"
                }
                """;
    }

    // SQL queries
    public String buildQuery(String status, int limit) {
        return """
                SELECT o.id, o.customer_name, COUNT(i.id) as item_count
                FROM orders o
                LEFT JOIN order_items i ON i.order_id = o.id
                WHERE o.status = '%s'
                GROUP BY o.id, o.customer_name
                ORDER BY o.created_at DESC
                LIMIT %d
                """.formatted(status, limit);
    }

    // HTML templates
    public String getHtmlTemplate(String title, String body) {
        return """
                <!DOCTYPE html>
                <html lang="en">
                <head>
                    <meta charset="UTF-8">
                    <title>%s</title>
                </head>
                <body>
                    <main>%s</main>
                </body>
                </html>
                """.formatted(title, body);
    }

    // Indentation control
    public String getIndented() {
        String text = """
                Line 1
                  Line 2 (indented)
                Line 3
                """;
        // stripIndent(), translateEscapes(), stripLeading(), stripTrailing()
        return text.stripIndent();
    }
}
```

---

## 5. Virtual Threads (Project Loom) - Practical Examples

### 5.1 Virtual Thread Fundamentals

```java
package com.example.modern.virtual;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class VirtualThreadExamples {

    // Create a single virtual thread
    public static void createVirtualThread() throws InterruptedException {
        Thread vThread = Thread.ofVirtual()
            .name("my-virtual-thread")
            .start(() -> {
                System.out.println("Running on: " + Thread.currentThread());
                System.out.println("Is virtual: " + Thread.currentThread().isVirtual());
            });

        vThread.join();
    }

    // Virtual thread executor
    public static void virtualThreadExecutor() throws Exception {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = new ArrayList<>();

            for (int i = 0; i < 10_000; i++) {
                final int taskId = i;
                futures.add(executor.submit(() -> {
                    Thread.sleep(Duration.ofMillis(100)); // I/O simulation - thread yields
                    return "Task " + taskId + " completed by " + Thread.currentThread().getName();
                }));
            }

            // Wait for all tasks
            int completed = 0;
            for (Future<String> f : futures) {
                f.get();
                completed++;
            }
            System.out.println("All " + completed + " tasks completed");
        }
    }

    // Performance comparison
    public static void compareThreadTypes(int taskCount) throws Exception {
        Runnable task = () -> {
            try {
                Thread.sleep(Duration.ofMillis(50)); // Simulate blocking I/O
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        // Platform threads (fixed pool)
        Instant start1 = Instant.now();
        try (ExecutorService platform = Executors.newFixedThreadPool(200)) {
            List<Future<?>> futures = new ArrayList<>();
            for (int i = 0; i < taskCount; i++) {
                futures.add(platform.submit(task));
            }
            futures.forEach(f -> {
                try { f.get(); } catch (Exception e) { /* ignore */ }
            });
        }
        long platformMs = Duration.between(start1, Instant.now()).toMillis();

        // Virtual threads (unbounded)
        Instant start2 = Instant.now();
        try (ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<?>> futures = new ArrayList<>();
            for (int i = 0; i < taskCount; i++) {
                futures.add(virtual.submit(task));
            }
            futures.forEach(f -> {
                try { f.get(); } catch (Exception e) { /* ignore */ }
            });
        }
        long virtualMs = Duration.between(start2, Instant.now()).toMillis();

        System.out.printf("Tasks: %d | Platform threads: %d ms | Virtual threads: %d ms%n",
            taskCount, platformMs, virtualMs);
        System.out.printf("Virtual threads are %.1fx faster%n",
            (double) platformMs / virtualMs);
    }
}
```

### 5.2 Virtual Threads in Spring Boot

```java
package com.example.modern.virtual;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.web.client.RestClient;

import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

@Configuration
public class VirtualThreadConfig {

    // Named executor with virtual threads
    @Bean("virtualExecutor")
    public Executor virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    // RestClient with virtual threads
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .baseUrl("https://api.example.com")
            .build();
    }
}
```

```java
package com.example.modern.virtual;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Executor;

@Service
public class VirtualThreadService {

    @Autowired
    @Qualifier("virtualExecutor")
    private Executor virtualExecutor;

    // I/O-bound operations run on virtual threads
    @Async("virtualExecutor")
    public CompletableFuture<String> fetchFromExternalService(String id) {
        // This blocks the virtual thread (which is cheap)
        // The carrier thread is freed while waiting
        try {
            Thread.sleep(100); // Simulate HTTP call
            return CompletableFuture.completedFuture("Result for " + id);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }

    public List<String> fetchAll(List<String> ids) {
        List<CompletableFuture<String>> futures = ids.stream()
            .map(id -> CompletableFuture.supplyAsync(
                () -> {
                    try {
                        Thread.sleep(100);
                        return "Result for " + id;
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                },
                virtualExecutor
            ))
            .toList();

        return futures.stream()
            .map(CompletableFuture::join)
            .toList();
    }
}
```

---

## 6. Structured Concurrency

### 6.1 Basic Structured Concurrency

```java
package com.example.modern.structured;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.StructuredTaskScope;

public class StructuredConcurrencyExamples {

    record UserProfile(String name, String email) {}
    record UserOrders(int count, double totalSpent) {}
    record UserDashboard(UserProfile profile, UserOrders orders) {}

    // Structured concurrency: both tasks succeed or both fail
    public UserDashboard fetchUserDashboard(Long userId)
            throws InterruptedException, ExecutionException {

        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            // Fork subtasks
            StructuredTaskScope.Subtask<UserProfile> profileTask =
                scope.fork(() -> fetchUserProfile(userId));

            StructuredTaskScope.Subtask<UserOrders> ordersTask =
                scope.fork(() -> fetchUserOrders(userId));

            // Wait for both tasks
            scope.join();

            // If either failed, this throws
            scope.throwIfFailed();

            // Both succeeded - combine results
            return new UserDashboard(profileTask.get(), ordersTask.get());
        }
    }

    // If ANY task completes successfully, cancel the rest
    public String fetchFirstAvailableData(String id) throws InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
            scope.fork(() -> fetchFromPrimarySource(id));
            scope.fork(() -> fetchFromSecondarySource(id));
            scope.fork(() -> fetchFromTertiarySource(id));

            scope.join(); // Blocks until first success or all fail
            return scope.result(); // Returns the first successful result
        }
    }

    // Simulated service calls
    private UserProfile fetchUserProfile(Long userId) throws InterruptedException {
        Thread.sleep(50);
        return new UserProfile("John Doe", "john@example.com");
    }

    private UserOrders fetchUserOrders(Long userId) throws InterruptedException {
        Thread.sleep(80);
        return new UserOrders(15, 1250.50);
    }

    private String fetchFromPrimarySource(String id) throws InterruptedException {
        Thread.sleep(200);
        return "Primary data for " + id;
    }

    private String fetchFromSecondarySource(String id) throws InterruptedException {
        Thread.sleep(50);
        return "Secondary data for " + id;
    }

    private String fetchFromTertiarySource(String id) throws InterruptedException {
        Thread.sleep(300);
        return "Tertiary data for " + id;
    }
}
```

---

## 7. Sequenced Collections

### 7.1 New Interfaces

```java
package com.example.modern.collections;

import java.util.*;

public class SequencedCollectionExamples {

    public void demonstrate() {
        // SequencedCollection - ordered collections
        List<String> list = new ArrayList<>(List.of("a", "b", "c", "d"));

        // New methods
        System.out.println(list.getFirst()); // "a" (no more list.get(0))
        System.out.println(list.getLast());  // "d" (no more list.get(list.size()-1))

        list.addFirst("x"); // Add to front
        list.addLast("z");  // Add to end

        list.removeFirst(); // Remove from front
        list.removeLast();  // Remove from end

        // Get reversed view (lightweight, no copy!)
        SequencedCollection<String> reversed = list.reversed();
        System.out.println(reversed.getFirst()); // Was last

        // SequencedSet - ordered Set
        LinkedHashSet<String> set = new LinkedHashSet<>(Set.of("b", "c", "a"));
        System.out.println(set.getFirst());
        System.out.println(set.getLast());

        // SequencedMap - ordered Map
        LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
        map.put("one", 1);
        map.put("two", 2);
        map.put("three", 3);

        System.out.println(map.firstEntry());  // Map.Entry("one", 1)
        System.out.println(map.lastEntry());   // Map.Entry("three", 3)

        map.putFirst("zero", 0);  // Inserts at front
        map.putLast("four", 4);   // Inserts at end

        // Reversed view of map
        SequencedMap<String, Integer> reversedMap = map.reversed();

        // Useful pattern: take N elements from end
        List<String> items = new ArrayList<>(List.of("item1", "item2", "item3", "item4"));
        // Get last 2 items
        List<String> last2 = new ArrayList<>();
        last2.addFirst(items.removeLast());
        last2.addFirst(items.removeLast());
        System.out.println(last2);
    }
}
```

---

## 8. String Templates (Preview in Java 21)

```java
package com.example.modern.templates;

// Note: String templates are a preview feature in Java 21
// Requires --enable-preview flag to compile

public class StringTemplateExamples {

    // Traditional approaches (available without preview)
    public String traditionalFormat(String name, int age) {
        // String.format
        return String.format("Hello, %s! You are %d years old.", name, age);
    }

    public String formattedMethod(String name, int age) {
        // Formatted (Java 15+)
        return "Hello, %s! You are %d years old.".formatted(name, age);
    }

    public String textBlockFormatted(String name, int age) {
        return """
                Hello, %s!
                You are %d years old.
                """.formatted(name, age);
    }

    // Common patterns in Spring Boot applications
    public String buildLogMessage(String operation, long durationMs, boolean success) {
        return "[%s] %s in %d ms".formatted(
            success ? "SUCCESS" : "FAILURE",
            operation,
            durationMs
        );
    }

    public String buildApiPath(String base, String resource, Long id) {
        return "%s/%s/%d".formatted(base, resource, id);
    }

    // SQL query building with text blocks (safer than concatenation)
    public String buildDynamicQuery(String table, String column, Object value) {
        // Never use string concatenation for user input (SQL injection!)
        // Use parameterized queries instead - this is just for building template
        return """
                SELECT * FROM %s
                WHERE %s = ?
                ORDER BY id
                """.formatted(table, column);
    }
}
```

---

## 9. Record Patterns

```java
package com.example.modern.patterns;

import com.example.modern.records.Point;
import com.example.modern.records.Money;

public class RecordPatternExamples {

    // Nested record patterns
    record Line(Point start, Point end) {}
    record ColoredShape(String color, Object shape) {}

    public String describeLocation(Object obj) {
        return switch (obj) {
            // Deconstruct record in pattern
            case Point(double x, double y) when x == 0 && y == 0 ->
                "Origin";
            case Point(double x, double y) when x == 0 ->
                "Y-axis at " + y;
            case Point(double x, double y) when y == 0 ->
                "X-axis at " + x;
            case Point(double x, double y) ->
                "Point (" + x + ", " + y + ")";
            default -> "Not a point";
        };
    }

    public String describeColoredShape(ColoredShape shape) {
        return switch (shape) {
            // Nested deconstruction
            case ColoredShape(String color,
                             com.example.modern.sealed.Circle(double radius)) ->
                "A " + color + " circle with radius " + radius;

            case ColoredShape(String color,
                             com.example.modern.sealed.Rectangle(double w, double h)) ->
                "A " + color + " rectangle " + w + "x" + h;

            default -> "A " + shape.color() + " " + shape.shape();
        };
    }

    // Pattern matching in if statements
    public double extractPrice(Object obj) {
        if (obj instanceof Money(var amount, var currency) && currency.equals("USD")) {
            return amount.doubleValue();
        }
        return 0.0;
    }
}
```

---

## 10. Using Modern Java Features in Spring Boot

### 10.1 Modern Spring Boot Application Structure

```java
package com.example.modern;

import com.example.modern.dto.CreateUserRequest;
import com.example.modern.dto.UserResponse;
import jakarta.validation.Valid;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import org.springframework.data.repository.CrudRepository;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

@SpringBootApplication
public class ModernJavaApp {
    public static void main(String[] args) {
        SpringApplication.run(ModernJavaApp.class, args);
    }
}

// Entity using record-like style (Spring Data JDBC)
@Table("users")
class User {

    @Id
    private Long id;
    private String username;
    private String email;
    private Instant createdAt = Instant.now();

    // Java records can't be JPA entities (mutable state needed)
    // but Spring Data JDBC works well with compact classes

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }
}

@Repository
interface UserRepository extends CrudRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    List<User> findAll();
}

@Service
@Transactional
class UserService {

    private final UserRepository userRepository;

    UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserResponse createUser(CreateUserRequest request) {
        // Use sealed result type for explicit error handling
        if (userRepository.findByEmail(request.email()).isPresent()) {
            throw new IllegalArgumentException("Email already exists: " + request.email());
        }

        User user = new User();
        user.setUsername(request.username());
        user.setEmail(request.email());

        User saved = userRepository.save(user);
        return UserResponse.from(saved);
    }

    @Transactional(readOnly = true)
    public List<UserResponse> findAll() {
        return userRepository.findAll().stream()
            .map(UserResponse::from)
            .toList();
    }
}
```

### 10.2 Result Type Pattern with Sealed Classes

```java
package com.example.modern.result;

// Sealed Result type - no exceptions for expected failures
public sealed interface Result<T>
    permits Result.Success, Result.Failure {

    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String message, Throwable cause) implements Result<T> {
        public Failure(String message) {
            this(message, null);
        }
    }

    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Result<T> failure(String message) {
        return new Failure<>(message);
    }

    static <T> Result<T> failure(String message, Throwable cause) {
        return new Failure<>(message, cause);
    }

    default boolean isSuccess() {
        return this instanceof Success;
    }

    default T getOrThrow() {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> throw new RuntimeException(f.message(), f.cause());
        };
    }

    default T getOrDefault(T defaultValue) {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> defaultValue;
        };
    }
}
```

```java
package com.example.modern.service;

import com.example.modern.result.Result;
import org.springframework.stereotype.Service;

@Service
public class PaymentService {

    public Result<String> processPayment(String cardNumber, double amount) {
        // Validate input
        if (cardNumber == null || cardNumber.length() < 16) {
            return Result.failure("Invalid card number");
        }
        if (amount <= 0) {
            return Result.failure("Amount must be positive");
        }

        try {
            // Process payment...
            String transactionId = "TXN-" + System.currentTimeMillis();
            return Result.success(transactionId);
        } catch (Exception e) {
            return Result.failure("Payment failed: " + e.getMessage(), e);
        }
    }
}
```

---

## 11. Real Example: Refactoring Legacy Code with Modern Features

### 11.1 Legacy Code (Before)

```java
package com.example.modern.legacy;

// BEFORE: Legacy-style Java code
import java.util.*;

public class OrderProcessorLegacy {

    // Constants as magic strings
    private static final String STATUS_PENDING = "PENDING";
    private static final String STATUS_PAID = "PAID";
    private static final String STATUS_SHIPPED = "SHIPPED";
    private static final String STATUS_DELIVERED = "DELIVERED";

    // Mutable data class
    public static class OrderDto {
        public Long id;
        public String customerName;
        public String status;
        public List<String> items;
        public Double total;

        // No-arg constructor required by frameworks
        public OrderDto() {}

        public OrderDto(Long id, String customerName, String status,
                        List<String> items, Double total) {
            this.id = id;
            this.customerName = customerName;
            this.status = status;
            this.items = items;
            this.total = total;
        }
    }

    // Complex switch with string comparison
    public String getStatusMessage(OrderDto order) {
        if (order.status == null) {
            return "Unknown status";
        }
        switch (order.status) {
            case "PENDING":
                return "Order " + order.id + " is pending for customer " + order.customerName;
            case "PAID":
                return "Order " + order.id + " has been paid by " + order.customerName;
            case "SHIPPED":
                return "Order " + order.id + " has been shipped to " + order.customerName;
            case "DELIVERED":
                return "Order " + order.id + " was delivered to " + order.customerName;
            default:
                return "Order " + order.id + " has unknown status: " + order.status;
        }
    }

    // Verbose null checks and type casting
    public double calculateDiscount(Object orderObj) {
        if (!(orderObj instanceof OrderDto)) {
            return 0.0;
        }
        OrderDto order = (OrderDto) orderObj;
        if (order.total == null || order.items == null) {
            return 0.0;
        }
        if (order.total > 1000.0) {
            return 0.15;
        } else if (order.total > 500.0) {
            return 0.10;
        } else if (order.items.size() > 5) {
            return 0.05;
        } else {
            return 0.0;
        }
    }
}
```

### 11.2 Modern Code (After)

```java
package com.example.modern.refactored;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

// AFTER: Modern Java 21 style

// Sealed status hierarchy - compile-time exhaustiveness check
public sealed interface OrderStatus
    permits OrderStatus.Pending, OrderStatus.Paid, OrderStatus.Shipped, OrderStatus.Delivered, OrderStatus.Cancelled {

    record Pending(Instant createdAt) implements OrderStatus {}
    record Paid(Instant paidAt, String transactionId) implements OrderStatus {}
    record Shipped(Instant shippedAt, String trackingNumber) implements OrderStatus {}
    record Delivered(Instant deliveredAt) implements OrderStatus {}
    record Cancelled(Instant cancelledAt, String reason) implements OrderStatus {}
}
```

```java
package com.example.modern.refactored;

import java.math.BigDecimal;
import java.util.List;

// Immutable record for order data
public record Order(
    Long id,
    String customerName,
    String customerEmail,
    List<OrderLine> items,
    BigDecimal total,
    OrderStatus status
) {
    public Order {
        if (id == null) throw new IllegalArgumentException("id required");
        if (customerName == null || customerName.isBlank())
            throw new IllegalArgumentException("customerName required");
        items = List.copyOf(items); // Defensive copy, ensures immutability
    }
}
```

```java
package com.example.modern.refactored;

import java.math.BigDecimal;

public record OrderLine(
    String productName,
    int quantity,
    BigDecimal unitPrice
) {
    public OrderLine {
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
        if (unitPrice.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Unit price cannot be negative");
    }

    public BigDecimal lineTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

```java
package com.example.modern.refactored;

import java.math.BigDecimal;

public class ModernOrderProcessor {

    // Pattern matching + switch expression = clean, exhaustive handling
    public String getStatusMessage(Order order) {
        return switch (order.status()) {
            case OrderStatus.Pending(var createdAt) ->
                "Order %d is pending since %s for customer %s"
                    .formatted(order.id(), createdAt, order.customerName());

            case OrderStatus.Paid(var paidAt, var txId) ->
                "Order %d paid at %s (txn: %s)".formatted(order.id(), paidAt, txId);

            case OrderStatus.Shipped(var shippedAt, var tracking) ->
                "Order %d shipped at %s, tracking: %s"
                    .formatted(order.id(), shippedAt, tracking);

            case OrderStatus.Delivered(var deliveredAt) ->
                "Order %d delivered to %s at %s"
                    .formatted(order.id(), order.customerName(), deliveredAt);

            case OrderStatus.Cancelled(var cancelledAt, var reason) ->
                "Order %d cancelled at %s: %s".formatted(order.id(), cancelledAt, reason);
        };
    }

    // Clean pattern matching, no instanceof + cast
    public double calculateDiscount(Object obj) {
        return switch (obj) {
            case Order o when o.total().compareTo(BigDecimal.valueOf(1000)) > 0 -> 0.15;
            case Order o when o.total().compareTo(BigDecimal.valueOf(500)) > 0 -> 0.10;
            case Order o when o.items().size() > 5 -> 0.05;
            case Order o -> 0.0;
            default -> 0.0;
        };
    }

    // Status transition using pattern matching
    public Order shipOrder(Order order, String trackingNumber) {
        OrderStatus newStatus = switch (order.status()) {
            case OrderStatus.Paid(var paidAt, var txId) ->
                new OrderStatus.Shipped(java.time.Instant.now(), trackingNumber);
            default ->
                throw new IllegalStateException(
                    "Cannot ship order in status: " + order.status()
                );
        };

        return new Order(
            order.id(),
            order.customerName(),
            order.customerEmail(),
            order.items(),
            order.total(),
            newStatus
        );
    }

    // Builder for complex construction
    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        // Builder pattern still useful for optional fields
        private Long id;
        private String customerName;
        private String customerEmail;
        private java.util.List<OrderLine> items = new java.util.ArrayList<>();
        private java.math.BigDecimal total = BigDecimal.ZERO;
        private OrderStatus status = new OrderStatus.Pending(java.time.Instant.now());

        public Builder id(Long id) { this.id = id; return this; }
        public Builder customerName(String name) { this.customerName = name; return this; }
        public Builder customerEmail(String email) { this.customerEmail = email; return this; }
        public Builder addItem(OrderLine item) {
            this.items.add(item);
            this.total = this.total.add(item.lineTotal());
            return this;
        }
        public Builder status(OrderStatus status) { this.status = status; return this; }

        public Order build() {
            return new Order(id, customerName, customerEmail, items, total, status);
        }
    }
}
```

---

## Summary

| Feature | Java Version | Benefit |
|---------|-------------|---------|
| Records | 16 (stable) | Eliminates DTO boilerplate, immutable value objects |
| Sealed Classes | 17 (stable) | Exhaustive type hierarchies, safer domain models |
| Pattern Matching `instanceof` | 16 (stable) | Eliminates redundant cast, cleaner conditionals |
| Switch Expressions | 14 (stable) | Exhaustive switch, expression form, arrow syntax |
| Pattern Matching for `switch` | 21 (stable) | Type-safe, exhaustive, deconstruction patterns |
| Text Blocks | 15 (stable) | Multi-line strings, SQL/JSON/HTML clarity |
| Virtual Threads | 21 (stable) | Millions of cheap threads, great for I/O workloads |
| Structured Concurrency | 21 (preview) | Safe parallel tasks with automatic cleanup |
| Sequenced Collections | 21 (stable) | Consistent first/last access across all ordered collections |
| Record Patterns | 21 (stable) | Deconstruct records inline in patterns |

---

## Next Part Preview

**Part 043: Spring Batch for Bulk Processing** — We'll explore Spring Batch architecture, chunk-oriented processing, parallel partitioned steps, and build a complete ETL pipeline that processes 1 million CSV records.
