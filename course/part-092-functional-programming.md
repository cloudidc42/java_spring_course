# Part 092: Functional Programming in Java

## Introduction

Functional programming treats computation as the evaluation of mathematical functions, avoiding mutable state and side effects. Java has supported functional programming since Java 8 with lambdas, streams, and the `java.util.function` package. This part covers advanced functional patterns with and without external libraries.

---

## Core Functional Interfaces

```java
// java.util.function package overview
import java.util.function.*;

// Function<T,R>: T -> R
Function<String, Integer> strlen = String::length;

// BiFunction<T,U,R>: (T,U) -> R
BiFunction<String, String, String> concat = String::concat;

// Predicate<T>: T -> boolean
Predicate<String> isBlank = String::isBlank;

// Consumer<T>: T -> void
Consumer<String> print = System.out::println;

// Supplier<T>: () -> T
Supplier<String> greeting = () -> "Hello, World!";

// UnaryOperator<T>: T -> T (specialization of Function)
UnaryOperator<String> trim = String::trim;

// BinaryOperator<T>: (T,T) -> T (specialization of BiFunction)
BinaryOperator<Integer> add = Integer::sum;
```

---

## Function Composition

```java
// src/main/java/com/example/functional/composition/FunctionComposition.java
package com.example.functional.composition;

import java.util.function.Function;
import java.util.function.Predicate;

public class FunctionComposition {

    /**
     * andThen: f.andThen(g) = g(f(x)) - apply f first, then g
     */
    public static void andThenExample() {
        Function<String, String> trim = String::trim;
        Function<String, String> toUpperCase = String::toUpperCase;
        Function<String, Integer> length = String::length;

        // Chain: trim → toUpperCase → length
        Function<String, Integer> pipeline = trim
            .andThen(toUpperCase)
            .andThen(length);

        System.out.println(pipeline.apply("  hello world  ")); // 11
    }

    /**
     * compose: f.compose(g) = f(g(x)) - apply g first, then f
     * Note: compose is the mathematical "before" operation
     */
    public static void composeExample() {
        Function<Integer, Integer> doubleIt = x -> x * 2;
        Function<Integer, Integer> addThree = x -> x + 3;

        // doubleIt.compose(addThree) = doubleIt(addThree(x)) = (x+3)*2
        Function<Integer, Integer> addThenDouble = doubleIt.compose(addThree);
        System.out.println(addThenDouble.apply(5)); // (5+3)*2 = 16

        // doubleIt.andThen(addThree) = addThree(doubleIt(x)) = (x*2)+3
        Function<Integer, Integer> doubleThenAdd = doubleIt.andThen(addThree);
        System.out.println(doubleThenAdd.apply(5)); // (5*2)+3 = 13
    }

    /**
     * Build a type-safe, composable transformation pipeline
     */
    public static <T, R, V> Function<T, V> compose(
            Function<T, R> first,
            Function<R, V> second) {
        return first.andThen(second);
    }

    // Higher-order function that returns a function
    public static Function<Integer, Integer> multiplier(int factor) {
        return x -> x * factor;
    }

    public static void main(String[] args) {
        andThenExample();
        composeExample();

        Function<Integer, Integer> triple = multiplier(3);
        Function<Integer, Integer> quadruple = multiplier(4);

        // Compose multipliers: 3 * 4 = 12x
        Function<Integer, Integer> twelveX = triple.andThen(quadruple);
        System.out.println(twelveX.apply(5)); // 60
    }
}
```

---

## Predicate Composition

```java
// src/main/java/com/example/functional/composition/PredicateComposition.java
package com.example.functional.composition;

import java.util.List;
import java.util.function.Predicate;
import java.util.stream.Collectors;

public class PredicateComposition {

    record Product(String name, double price, String category, int stock) {}

    public static void main(String[] args) {
        List<Product> products = List.of(
            new Product("Laptop", 999.99, "Electronics", 50),
            new Product("Phone", 499.99, "Electronics", 0),
            new Product("Desk", 299.99, "Furniture", 10),
            new Product("Chair", 149.99, "Furniture", 5),
            new Product("Monitor", 349.99, "Electronics", 25),
            new Product("Keyboard", 79.99, "Electronics", 100)
        );

        // Define reusable predicates
        Predicate<Product> isElectronics = p -> "Electronics".equals(p.category());
        Predicate<Product> inStock = p -> p.stock() > 0;
        Predicate<Product> underFiveHundred = p -> p.price() < 500.0;
        Predicate<Product> expensive = p -> p.price() >= 300.0;

        // Compose: in stock AND electronics AND under $500
        Predicate<Product> affordableElectronics = isElectronics
            .and(inStock)
            .and(underFiveHundred);

        List<Product> result = products.stream()
            .filter(affordableElectronics)
            .collect(Collectors.toList());

        result.forEach(p -> System.out.printf("%-12s $%.2f%n", p.name(), p.price()));
        // Keyboard    $79.99
        // Monitor     $349.99

        // OR composition
        Predicate<Product> wantedProducts = isElectronics.or(expensive);
        System.out.println("\nElectronics or expensive items:");
        products.stream()
            .filter(wantedProducts)
            .forEach(p -> System.out.printf("%-12s %-12s $%.2f%n",
                p.name(), p.category(), p.price()));

        // NOT composition
        Predicate<Product> notElectronics = Predicate.not(isElectronics);
        Predicate<Product> outOfStock = Predicate.not(inStock);

        System.out.println("\nOut of stock non-electronics:");
        products.stream()
            .filter(notElectronics.and(outOfStock))
            .forEach(p -> System.out.println(p.name()));

        // Dynamic predicate building
        Predicate<Product> dynamicFilter = buildFilter("Electronics", 100.0, 500.0);
        System.out.println("\nDynamic filter results:");
        products.stream()
            .filter(dynamicFilter)
            .forEach(p -> System.out.println(p.name()));
    }

    // Factory method for dynamic predicate composition
    static Predicate<Product> buildFilter(String category, double minPrice, double maxPrice) {
        Predicate<Product> filter = p -> true;  // start with "accept all"

        if (category != null) {
            filter = filter.and(p -> category.equals(p.category()));
        }
        if (minPrice > 0) {
            filter = filter.and(p -> p.price() >= minPrice);
        }
        if (maxPrice < Double.MAX_VALUE) {
            filter = filter.and(p -> p.price() <= maxPrice);
        }

        return filter;
    }
}
```

---

## Optional as Monad

```java
// src/main/java/com/example/functional/monad/OptionalMonad.java
package com.example.functional.monad;

import java.util.Optional;

public class OptionalMonad {

    record User(Long id, String email, Long addressId) {}
    record Address(Long id, String city, Long zipCodeId) {}
    record ZipCode(Long id, String code, String timezone) {}

    // Simulated repositories
    Optional<User> findUser(Long id) {
        return id == 1L
            ? Optional.of(new User(1L, "alice@example.com", 10L))
            : Optional.empty();
    }

    Optional<Address> findAddress(Long id) {
        return id == 10L
            ? Optional.of(new Address(10L, "New York", 100L))
            : Optional.empty();
    }

    Optional<ZipCode> findZipCode(Long id) {
        return id == 100L
            ? Optional.of(new ZipCode(100L, "10001", "America/New_York"))
            : Optional.empty();
    }

    /**
     * Deeply nested Optional chaining (the monadic flatMap pattern)
     * Avoids nested null checks entirely
     */
    public Optional<String> getUserTimezone(Long userId) {
        return findUser(userId)                         // Optional<User>
            .map(User::addressId)                       // Optional<Long>
            .flatMap(this::findAddress)                 // Optional<Address>
            .map(Address::zipCodeId)                    // Optional<Long>
            .flatMap(this::findZipCode)                 // Optional<ZipCode>
            .map(ZipCode::timezone);                    // Optional<String>
    }

    /**
     * Using Optional.or() for fallback (Java 9+)
     */
    public String getUserEmail(Long userId) {
        return findUser(userId)
            .map(User::email)
            .or(() -> Optional.of("default@example.com"))  // fallback Optional
            .orElse("unknown");
    }

    /**
     * ifPresentOrElse (Java 9+)
     */
    public void processUser(Long userId) {
        findUser(userId).ifPresentOrElse(
            user -> System.out.println("Processing user: " + user.email()),
            ()   -> System.out.println("User not found, using guest mode")
        );
    }

    /**
     * Filter inside Optional
     */
    public Optional<User> findVerifiedUser(Long userId) {
        return findUser(userId)
            .filter(user -> user.email() != null && user.email().contains("@"));
    }

    /**
     * Combining multiple Optionals
     */
    public Optional<String> getCityAndTimezone(Long userId) {
        Optional<String> city = findUser(userId)
            .map(User::addressId)
            .flatMap(this::findAddress)
            .map(Address::city);

        Optional<String> timezone = getUserTimezone(userId);

        // Both must be present
        return city.flatMap(c -> timezone.map(tz -> c + " (" + tz + ")"));
    }

    /**
     * Converting nullable values safely
     */
    public static Optional<Integer> parseInteger(String s) {
        try {
            return Optional.ofNullable(s)
                .filter(str -> !str.isBlank())
                .map(Integer::parseInt);
        } catch (NumberFormatException e) {
            return Optional.empty();
        }
    }

    public static void main(String[] args) {
        OptionalMonad demo = new OptionalMonad();

        demo.getUserTimezone(1L).ifPresent(tz -> System.out.println("Timezone: " + tz));
        demo.getUserTimezone(999L).ifPresentOrElse(
            tz -> System.out.println("Timezone: " + tz),
            () -> System.out.println("No timezone found")
        );

        System.out.println(parseInteger("42"));       // Optional[42]
        System.out.println(parseInteger("  "));       // Optional.empty
        System.out.println(parseInteger("not a num")); // Optional.empty
        System.out.println(parseInteger(null));        // Optional.empty
    }
}
```

---

## Either Pattern for Error Handling

The `Either<L, R>` type represents a value of two possible types — conventionally `Left` for errors and `Right` for success.

```java
// src/main/java/com/example/functional/either/Either.java
package com.example.functional.either;

import java.util.NoSuchElementException;
import java.util.Optional;
import java.util.function.Consumer;
import java.util.function.Function;

/**
 * A right-biased Either monad. Right = success, Left = error.
 */
public sealed interface Either<L, R> permits Either.Left, Either.Right {

    // Factory methods
    static <L, R> Either<L, R> left(L value) {
        return new Left<>(value);
    }

    static <L, R> Either<L, R> right(R value) {
        return new Right<>(value);
    }

    static <L, R> Either<L, R> fromNullable(R value, L errorIfNull) {
        return value != null ? right(value) : left(errorIfNull);
    }

    // Core operations
    boolean isRight();
    boolean isLeft();

    R getRight();
    L getLeft();

    // Functor: map over the right value
    <U> Either<L, U> map(Function<R, U> mapper);

    // Monad: flatMap (bind) over the right value
    <U> Either<L, U> flatMap(Function<R, Either<L, U>> mapper);

    // Map over the left value
    <U> Either<U, R> mapLeft(Function<L, U> mapper);

    // Fold: extract value from both sides
    <T> T fold(Function<L, T> leftMapper, Function<R, T> rightMapper);

    // Side effects
    Either<L, R> peek(Consumer<R> consumer);
    Either<L, R> peekLeft(Consumer<L> consumer);

    // Convert to Optional (loses the error)
    Optional<R> toOptional();

    // Get or default
    R getOrElse(R defaultValue);
    R getOrElse(Function<L, R> defaultFunction);

    // Record implementations
    record Left<L, R>(L value) implements Either<L, R> {
        @Override public boolean isRight() { return false; }
        @Override public boolean isLeft() { return true; }

        @Override
        public R getRight() {
            throw new NoSuchElementException("Either.Left has no Right value");
        }

        @Override
        public L getLeft() { return value; }

        @Override
        @SuppressWarnings("unchecked")
        public <U> Either<L, U> map(Function<R, U> mapper) {
            return (Either<L, U>) this;  // Left passes through unchanged
        }

        @Override
        @SuppressWarnings("unchecked")
        public <U> Either<L, U> flatMap(Function<R, Either<L, U>> mapper) {
            return (Either<L, U>) this;
        }

        @Override
        public <U> Either<U, R> mapLeft(Function<L, U> mapper) {
            return new Left<>(mapper.apply(value));
        }

        @Override
        public <T> T fold(Function<L, T> leftMapper, Function<R, T> rightMapper) {
            return leftMapper.apply(value);
        }

        @Override
        public Either<L, R> peek(Consumer<R> consumer) { return this; }

        @Override
        public Either<L, R> peekLeft(Consumer<L> consumer) {
            consumer.accept(value);
            return this;
        }

        @Override
        public Optional<R> toOptional() { return Optional.empty(); }

        @Override
        public R getOrElse(R defaultValue) { return defaultValue; }

        @Override
        public R getOrElse(Function<L, R> defaultFunction) {
            return defaultFunction.apply(value);
        }
    }

    record Right<L, R>(R value) implements Either<L, R> {
        @Override public boolean isRight() { return true; }
        @Override public boolean isLeft() { return false; }

        @Override
        public R getRight() { return value; }

        @Override
        public L getLeft() {
            throw new NoSuchElementException("Either.Right has no Left value");
        }

        @Override
        public <U> Either<L, U> map(Function<R, U> mapper) {
            return new Right<>(mapper.apply(value));
        }

        @Override
        public <U> Either<L, U> flatMap(Function<R, Either<L, U>> mapper) {
            return mapper.apply(value);
        }

        @Override
        @SuppressWarnings("unchecked")
        public <U> Either<U, R> mapLeft(Function<L, U> mapper) {
            return (Either<U, R>) this;
        }

        @Override
        public <T> T fold(Function<L, T> leftMapper, Function<R, T> rightMapper) {
            return rightMapper.apply(value);
        }

        @Override
        public Either<L, R> peek(Consumer<R> consumer) {
            consumer.accept(value);
            return this;
        }

        @Override
        public Either<L, R> peekLeft(Consumer<L> consumer) { return this; }

        @Override
        public Optional<R> toOptional() { return Optional.of(value); }

        @Override
        public R getOrElse(R defaultValue) { return value; }

        @Override
        public R getOrElse(Function<L, R> defaultFunction) { return value; }
    }
}
```

```java
// src/main/java/com/example/functional/either/EitherDemo.java
package com.example.functional.either;

import java.math.BigDecimal;

public class EitherDemo {

    // Error types
    sealed interface OrderError permits OrderError.NotFound, OrderError.PaymentFailed,
            OrderError.InsufficientStock, OrderError.ValidationError {
        record NotFound(Long id) implements OrderError {}
        record PaymentFailed(String reason) implements OrderError {}
        record InsufficientStock(String productId, int requested, int available) implements OrderError {}
        record ValidationError(String field, String message) implements OrderError {}
    }

    record Order(Long id, String customerId, BigDecimal amount) {}
    record PaymentResult(String transactionId, BigDecimal amount) {}
    record Confirmation(Order order, PaymentResult payment, String trackingId) {}

    // Each step returns Either<Error, Success>
    Either<OrderError, Order> findOrder(Long id) {
        if (id == null || id <= 0) {
            return Either.left(new OrderError.ValidationError("id", "ID must be positive"));
        }
        if (id == 999L) {
            return Either.left(new OrderError.NotFound(id));
        }
        return Either.right(new Order(id, "customer_1", new BigDecimal("49.99")));
    }

    Either<OrderError, Order> validateStock(Order order) {
        // Simulate out-of-stock for certain orders
        if (order.id() % 5 == 0) {
            return Either.left(new OrderError.InsufficientStock("prod_A", 2, 0));
        }
        return Either.right(order);
    }

    Either<OrderError, PaymentResult> processPayment(Order order) {
        if (order.amount().compareTo(new BigDecimal("10000")) > 0) {
            return Either.left(new OrderError.PaymentFailed("Amount exceeds limit"));
        }
        return Either.right(new PaymentResult("txn_" + order.id(), order.amount()));
    }

    Either<OrderError, Confirmation> createShipment(Order order, PaymentResult payment) {
        String trackingId = "TRACK_" + order.id() + "_" + System.currentTimeMillis();
        return Either.right(new Confirmation(order, payment, trackingId));
    }

    /**
     * Railway-oriented programming: compose steps with flatMap.
     * Errors short-circuit the chain.
     */
    Either<OrderError, Confirmation> processOrder(Long orderId) {
        return findOrder(orderId)
            .flatMap(this::validateStock)
            .flatMap(order ->
                processPayment(order)
                    .flatMap(payment -> createShipment(order, payment))
            );
    }

    public static void main(String[] args) {
        EitherDemo demo = new EitherDemo();

        // Successful order
        Either<OrderError, Confirmation> result = demo.processOrder(1L);
        String message = result.fold(
            error -> "Order failed: " + switch (error) {
                case OrderError.NotFound nf -> "Order " + nf.id() + " not found";
                case OrderError.PaymentFailed pf -> "Payment failed: " + pf.reason();
                case OrderError.InsufficientStock is ->
                    "Insufficient stock for " + is.productId();
                case OrderError.ValidationError ve ->
                    "Validation error on " + ve.field() + ": " + ve.message();
            },
            conf -> "Order confirmed! Tracking: " + conf.trackingId()
        );
        System.out.println(message);

        // Not found
        demo.processOrder(999L).peekLeft(err ->
            System.out.println("Error: " + err)
        );

        // Out of stock
        demo.processOrder(5L).fold(
            err -> { System.out.println("Order 5 failed: " + err); return null; },
            ok  -> { System.out.println("Order 5 succeeded: " + ok); return null; }
        );
    }
}
```

---

## Currying and Partial Application

```java
// src/main/java/com/example/functional/currying/CurryingDemo.java
package com.example.functional.currying;

import java.util.function.BiFunction;
import java.util.function.Function;

public class CurryingDemo {

    /**
     * Currying: transform f(a, b) -> (a -> (b -> result))
     */
    static <A, B, R> Function<A, Function<B, R>> curry(BiFunction<A, B, R> f) {
        return a -> b -> f.apply(a, b);
    }

    /**
     * Uncurrying: reverse of curry
     */
    static <A, B, R> BiFunction<A, B, R> uncurry(Function<A, Function<B, R>> f) {
        return (a, b) -> f.apply(a).apply(b);
    }

    /**
     * Partial application: fix first argument, return function for remaining
     */
    static <A, B, R> Function<B, R> partial(BiFunction<A, B, R> f, A a) {
        return b -> f.apply(a, b);
    }

    public static void main(String[] args) {
        // Standard BiFunction
        BiFunction<Integer, Integer, Integer> add = Integer::sum;

        // Curried version
        Function<Integer, Function<Integer, Integer>> curriedAdd = curry(add);

        // Create adders via partial application
        Function<Integer, Integer> add5 = curriedAdd.apply(5);
        Function<Integer, Integer> add10 = curriedAdd.apply(10);

        System.out.println(add5.apply(3));   // 8
        System.out.println(add10.apply(3));  // 13

        // Practical: tax rate calculator
        BiFunction<Double, Double, Double> taxCalc = (rate, amount) -> amount * (1 + rate);
        Function<Double, Double> withUSTax = partial(taxCalc, 0.0875);
        Function<Double, Double> withEUVAT = partial(taxCalc, 0.20);

        double price = 100.0;
        System.out.printf("US price: $%.2f%n", withUSTax.apply(price));  // $108.75
        System.out.printf("EU price: $%.2f%n", withEUVAT.apply(price));  // $120.00

        // Curried URL builder
        Function<String, Function<String, Function<String, String>>> urlBuilder =
            protocol -> host -> path ->
                protocol + "://" + host + "/" + path;

        Function<String, Function<String, String>> httpsBuilder =
            urlBuilder.apply("https");
        Function<String, String> apiBuilder =
            httpsBuilder.apply("api.example.com");

        System.out.println(apiBuilder.apply("users"));     // https://api.example.com/users
        System.out.println(apiBuilder.apply("products"));  // https://api.example.com/products
    }
}
```

---

## Memoization

```java
// src/main/java/com/example/functional/memoization/Memoizer.java
package com.example.functional.memoization;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.BiFunction;
import java.util.function.Function;

public class Memoizer {

    /**
     * Memoize a function: cache results to avoid recomputation
     */
    public static <T, R> Function<T, R> memoize(Function<T, R> fn) {
        Map<T, R> cache = new ConcurrentHashMap<>();
        return input -> cache.computeIfAbsent(input, fn);
    }

    /**
     * Memoize a BiFunction
     */
    public static <T, U, R> BiFunction<T, U, R> memoize(BiFunction<T, U, R> fn) {
        record Pair<A, B>(A first, B second) {}
        Map<Pair<T, U>, R> cache = new ConcurrentHashMap<>();
        return (t, u) -> cache.computeIfAbsent(new Pair<>(t, u), k -> fn.apply(k.first(), k.second()));
    }

    /**
     * Recursive memoization via self-reference
     */
    public static <T, R> Function<T, R> memoizeRecursive(
            Function<Function<T, R>, Function<T, R>> fnFactory) {
        Map<T, R> cache = new ConcurrentHashMap<>();
        // We need a mutable reference for the recursive call
        Function<T, R>[] memoizedRef = new Function[1];
        memoizedRef[0] = input -> cache.computeIfAbsent(input, k ->
            fnFactory.apply(memoizedRef[0]).apply(k));
        return memoizedRef[0];
    }

    public static void main(String[] args) {
        // Expensive computation simulation
        Function<Integer, Long> slowFibonacci = n -> {
            if (n <= 1) return (long) n;
            // Deliberately slow iterative
            long a = 0, b = 1;
            for (int i = 2; i <= n; i++) {
                long temp = a + b;
                a = b;
                b = temp;
            }
            return b;
        };

        Function<Integer, Long> memoizedFib = memoize(slowFibonacci);

        long start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            memoizedFib.apply(i % 100);
        }
        long cached = System.nanoTime() - start;

        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            slowFibonacci.apply(i % 100);
        }
        long uncached = System.nanoTime() - start;

        System.out.printf("Cached: %,d ns | Uncached: %,d ns | Speedup: %.1fx%n",
            cached, uncached, (double) uncached / cached);

        // Memoized recursive fibonacci
        Function<Integer, Long> memoFib = memoizeRecursive(
            self -> n -> n <= 1 ? (long) n : self.apply(n - 1) + self.apply(n - 2)
        );
        System.out.println("fib(40) = " + memoFib.apply(40)); // 102334155
    }
}
```

---

## Immutable Collections

```java
// src/main/java/com/example/functional/immutable/ImmutableDemo.java
package com.example.functional.immutable;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class ImmutableDemo {

    public static void main(String[] args) {
        // Java 9+ factory methods produce unmodifiable collections
        List<String> names = List.of("Alice", "Bob", "Charlie");
        Map<String, Integer> scores = Map.of("Alice", 100, "Bob", 90);
        Set<String> roles = Set.of("ADMIN", "USER", "MODERATOR");

        // Cannot add/remove - throws UnsupportedOperationException
        // names.add("Dave"); // throws

        // "Modification" creates new collections
        List<String> moreNames = Stream.concat(names.stream(), Stream.of("Dave"))
            .collect(Collectors.toUnmodifiableList());

        List<String> withoutBob = names.stream()
            .filter(n -> !n.equals("Bob"))
            .collect(Collectors.toUnmodifiableList());

        System.out.println(names);     // [Alice, Bob, Charlie]
        System.out.println(moreNames); // [Alice, Bob, Charlie, Dave]
        System.out.println(withoutBob);// [Alice, Charlie]

        // Copying with modifications
        Map<String, Integer> updatedScores = Stream.concat(
            scores.entrySet().stream(),
            Map.of("Charlie", 95).entrySet().stream()
        ).collect(Collectors.toUnmodifiableMap(
            Map.Entry::getKey, Map.Entry::getValue
        ));
        System.out.println(updatedScores);

        // Java 10+ copyOf
        List<String> copy = List.copyOf(names);
        System.out.println(copy == names); // may be true if already unmodifiable
    }
}
```

---

## Functional Pipeline Pattern

```java
// src/main/java/com/example/functional/pipeline/DataPipeline.java
package com.example.functional.pipeline;

import java.util.List;
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.stream.Collectors;

/**
 * A composable, lazy data pipeline
 */
public class DataPipeline<T> {

    private final List<T> data;

    private DataPipeline(List<T> data) {
        this.data = List.copyOf(data);
    }

    public static <T> DataPipeline<T> of(List<T> data) {
        return new DataPipeline<>(data);
    }

    public DataPipeline<T> filter(Predicate<T> predicate) {
        return new DataPipeline<>(
            data.stream().filter(predicate).collect(Collectors.toList())
        );
    }

    public <R> DataPipeline<R> map(Function<T, R> mapper) {
        return new DataPipeline<>(
            data.stream().map(mapper).collect(Collectors.toList())
        );
    }

    public DataPipeline<T> limit(int n) {
        return new DataPipeline<>(data.stream().limit(n).collect(Collectors.toList()));
    }

    public DataPipeline<T> skip(int n) {
        return new DataPipeline<>(data.stream().skip(n).collect(Collectors.toList()));
    }

    public List<T> toList() {
        return List.copyOf(data);
    }

    public long count() {
        return data.size();
    }
}
```

---

## Real Example: Data Processing Pipeline

```java
// src/main/java/com/example/functional/pipeline/SalesReportPipeline.java
package com.example.functional.pipeline;

import com.example.functional.either.Either;
import lombok.extern.slf4j.Slf4j;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.util.*;
import java.util.function.Function;
import java.util.function.Predicate;
import java.util.stream.Collectors;

@Slf4j
public class SalesReportPipeline {

    record SaleRecord(
        String orderId,
        String customerId,
        String region,
        String product,
        int quantity,
        BigDecimal unitPrice,
        LocalDate saleDate
    ) {
        BigDecimal total() { return unitPrice.multiply(BigDecimal.valueOf(quantity)); }
    }

    record RegionSummary(
        String region,
        long orderCount,
        BigDecimal totalRevenue,
        BigDecimal averageOrderValue,
        String topProduct
    ) {}

    sealed interface ParseError permits ParseError.MissingField, ParseError.InvalidFormat {
        record MissingField(String field) implements ParseError {}
        record InvalidFormat(String field, String value) implements ParseError {}
    }

    // --- Pure parsing functions ---

    static Either<ParseError, SaleRecord> parseCsvRow(String[] cols) {
        if (cols.length < 7) {
            return Either.left(new ParseError.MissingField("Expected 7 columns, got " + cols.length));
        }

        try {
            return Either.right(new SaleRecord(
                cols[0].trim(),
                cols[1].trim(),
                cols[2].trim(),
                cols[3].trim(),
                Integer.parseInt(cols[4].trim()),
                new BigDecimal(cols[5].trim()),
                LocalDate.parse(cols[6].trim())
            ));
        } catch (NumberFormatException e) {
            return Either.left(new ParseError.InvalidFormat("quantity or price", e.getMessage()));
        } catch (Exception e) {
            return Either.left(new ParseError.InvalidFormat("date", cols[6]));
        }
    }

    // --- Pure transformation functions ---

    static Function<List<SaleRecord>, Map<String, List<SaleRecord>>> groupByRegion() {
        return records -> records.stream()
            .collect(Collectors.groupingBy(SaleRecord::region));
    }

    static Function<Map.Entry<String, List<SaleRecord>>, RegionSummary> summarizeRegion() {
        return entry -> {
            String region = entry.getKey();
            List<SaleRecord> records = entry.getValue();

            BigDecimal totalRevenue = records.stream()
                .map(SaleRecord::total)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

            BigDecimal avgOrderValue = records.isEmpty() ? BigDecimal.ZERO :
                totalRevenue.divide(BigDecimal.valueOf(records.size()), 2, RoundingMode.HALF_UP);

            String topProduct = records.stream()
                .collect(Collectors.groupingBy(SaleRecord::product,
                    Collectors.summingLong(r -> (long) r.quantity())))
                .entrySet().stream()
                .max(Map.Entry.comparingByValue())
                .map(Map.Entry::getKey)
                .orElse("N/A");

            return new RegionSummary(region, records.size(), totalRevenue, avgOrderValue, topProduct);
        };
    }

    // --- Composable filter predicates ---

    static Predicate<SaleRecord> inDateRange(LocalDate from, LocalDate to) {
        return record -> !record.saleDate().isBefore(from) && !record.saleDate().isAfter(to);
    }

    static Predicate<SaleRecord> minRevenue(BigDecimal minimum) {
        return record -> record.total().compareTo(minimum) >= 0;
    }

    static Predicate<SaleRecord> inRegions(String... regions) {
        Set<String> regionSet = Set.of(regions);
        return record -> regionSet.contains(record.region());
    }

    /**
     * Full pipeline: parse CSV → validate → filter → group → summarize → sort
     */
    public static List<RegionSummary> runReport(
            List<String[]> rawData,
            LocalDate from,
            LocalDate to,
            BigDecimal minOrderRevenue) {

        // Parse and partition into valid/invalid
        List<Either<ParseError, SaleRecord>> parsed = rawData.stream()
            .map(SalesReportPipeline::parseCsvRow)
            .toList();

        long errorCount = parsed.stream().filter(Either::isLeft).count();
        if (errorCount > 0) {
            log.warn("Skipping {} rows with parse errors", errorCount);
            parsed.stream()
                .filter(Either::isLeft)
                .map(Either::getLeft)
                .forEach(err -> log.debug("Parse error: {}", err));
        }

        // Process valid records through the pipeline
        return parsed.stream()
            .filter(Either::isRight)
            .map(Either::getRight)
            // Apply filters
            .filter(inDateRange(from, to))
            .filter(minRevenue(minOrderRevenue))
            // Group by region
            .collect(Collectors.groupingBy(SaleRecord::region))
            .entrySet().stream()
            // Summarize each region
            .map(summarizeRegion())
            // Sort by total revenue descending
            .sorted(Comparator.comparing(RegionSummary::totalRevenue).reversed())
            .toList();
    }

    public static void main(String[] args) {
        List<String[]> csvData = List.of(
            new String[]{"ORD001", "CUST1", "North", "Laptop",    "2", "999.99", "2024-01-15"},
            new String[]{"ORD002", "CUST2", "South", "Phone",     "3", "499.99", "2024-01-20"},
            new String[]{"ORD003", "CUST3", "North", "Keyboard",  "5",  "79.99", "2024-02-01"},
            new String[]{"ORD004", "CUST4", "East",  "Monitor",   "1", "349.99", "2024-02-10"},
            new String[]{"ORD005", "CUST5", "North", "Laptop",    "1", "999.99", "2024-02-15"},
            new String[]{"INVALID_ROW_TOO_SHORT"},                          // parse error
            new String[]{"ORD006", "CUST6", "West",  "Chair",     "2", "149.99", "2024-03-01"},
            new String[]{"ORD007", "CUST7", "South", "Desk",      "1", "299.99", "2024-03-05"}
        );

        List<RegionSummary> report = runReport(
            csvData,
            LocalDate.of(2024, 1, 1),
            LocalDate.of(2024, 12, 31),
            new BigDecimal("100.00")
        );

        System.out.println("=== Regional Sales Report ===");
        report.forEach(r -> System.out.printf(
            "%-8s | Orders: %3d | Revenue: $%,10.2f | Avg: $%,8.2f | Top: %s%n",
            r.region(), r.orderCount(), r.totalRevenue(), r.averageOrderValue(), r.topProduct()
        ));
    }
}
```

---

## Functional Spring Beans (Functional Bean Registration)

```java
// src/main/java/com/example/functional/spring/FunctionalBeanConfig.java
package com.example.functional.spring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.WebApplicationType;
import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.support.GenericApplicationContext;
import org.springframework.web.servlet.function.RouterFunction;
import org.springframework.web.servlet.function.RouterFunctions;
import org.springframework.web.servlet.function.ServerResponse;

import java.util.function.Supplier;

/**
 * Functional bean registration using the ApplicationContextInitializer API.
 * Avoids class-path scanning for faster startup (useful in GraalVM native images).
 */
public class FunctionalBeanConfig
        implements ApplicationContextInitializer<GenericApplicationContext> {

    @Override
    public void initialize(GenericApplicationContext context) {
        // Register beans programmatically without @Component or @Bean
        context.registerBean(UserRepository.class, UserRepository::new);
        context.registerBean(UserService.class, () ->
            new UserService(context.getBean(UserRepository.class)));

        // Functional router (replaces @RestController for simple endpoints)
        context.registerBean("userRouter", RouterFunction.class, () ->
            buildUserRouter(context.getBean(UserService.class)));
    }

    private RouterFunction<ServerResponse> buildUserRouter(UserService userService) {
        return RouterFunctions.route()
            .GET("/api/users/{id}", request -> {
                String id = request.pathVariable("id");
                return userService.findById(Long.parseLong(id))
                    .map(user -> ServerResponse.ok().body(user))
                    .orElse(ServerResponse.notFound().build());
            })
            .POST("/api/users", request -> {
                // handle user creation
                return ServerResponse.ok().body("created");
            })
            .build();
    }
}
```

---

## Vavr Integration

Vavr is a functional library that brings Haskell-like types to Java.

```xml
<dependency>
    <groupId>io.vavr</groupId>
    <artifactId>vavr</artifactId>
    <version>0.10.4</version>
</dependency>
```

```java
// src/main/java/com/example/functional/vavr/VavrDemo.java
package com.example.functional.vavr;

import io.vavr.Tuple;
import io.vavr.Tuple2;
import io.vavr.collection.List;
import io.vavr.collection.Map;
import io.vavr.control.Either;
import io.vavr.control.Option;
import io.vavr.control.Try;
import io.vavr.control.Validation;

public class VavrDemo {

    // Vavr's Try monad for exception handling
    static Try<Integer> parseAndDivide(String a, String b) {
        return Try.of(() -> Integer.parseInt(a))
            .flatMap(x -> Try.of(() -> Integer.parseInt(b))
                .flatMap(y -> Try.of(() -> {
                    if (y == 0) throw new ArithmeticException("Division by zero");
                    return x / y;
                })));
    }

    // Vavr's Validation for accumulating errors
    record PersonRequest(String name, int age, String email) {}
    record Person(String name, int age, String email) {}

    static Validation<java.util.List<String>, String> validateName(String name) {
        if (name == null || name.isBlank()) {
            return Validation.invalid(java.util.List.of("Name cannot be blank"));
        }
        if (name.length() < 2) {
            return Validation.invalid(java.util.List.of("Name must be at least 2 characters"));
        }
        return Validation.valid(name.trim());
    }

    static Validation<java.util.List<String>, Integer> validateAge(int age) {
        if (age < 0 || age > 150) {
            return Validation.invalid(java.util.List.of("Age must be between 0 and 150"));
        }
        return Validation.valid(age);
    }

    static Validation<java.util.List<String>, String> validateEmail(String email) {
        if (email == null || !email.contains("@")) {
            return Validation.invalid(java.util.List.of("Invalid email address"));
        }
        return Validation.valid(email);
    }

    public static void main(String[] args) {
        // Try monad
        System.out.println(parseAndDivide("10", "2"));      // Success(5)
        System.out.println(parseAndDivide("10", "0"));      // Failure(ArithmeticException)
        System.out.println(parseAndDivide("abc", "2"));     // Failure(NumberFormatException)

        // Recover from failure
        int result = parseAndDivide("10", "0")
            .recover(ArithmeticException.class, e -> 0)
            .getOrElse(-1);
        System.out.println("Recovered: " + result); // 0

        // Immutable List (structural sharing)
        List<Integer> list1 = List.of(1, 2, 3);
        List<Integer> list2 = list1.prepend(0); // [0, 1, 2, 3] - new list
        System.out.println(list1); // List(1, 2, 3) - unchanged
        System.out.println(list2); // List(0, 1, 2, 3)

        // Persistent Map
        Map<String, Integer> map1 = Map.of("a", 1, "b", 2);
        Map<String, Integer> map2 = map1.put("c", 3);
        System.out.println(map1); // LinkedHashMap((a, 1), (b, 2))
        System.out.println(map2); // LinkedHashMap((a, 1), (b, 2), (c, 3))

        // Tuples
        Tuple2<String, Integer> tuple = Tuple.of("hello", 42);
        System.out.println(tuple._1); // hello
        System.out.println(tuple._2); // 42

        // Either (similar to our custom implementation)
        Either<String, Integer> success = Either.right(42);
        Either<String, Integer> failure = Either.left("Something went wrong");

        System.out.println(success.map(n -> n * 2)); // Right(84)
        System.out.println(failure.map(n -> n * 2)); // Left(Something went wrong)
    }
}
```

---

## Summary

| Concept | Java API | Library |
|---|---|---|
| Function composition | `Function.andThen`, `.compose` | Built-in |
| Predicate composition | `Predicate.and`, `.or`, `.not` | Built-in |
| Optional monad | `Optional.flatMap`, `.map` | Built-in |
| Either monad | Custom / Vavr | Custom / Vavr |
| Currying | Manual / Vavr `Function.curried()` | Vavr |
| Memoization | `ConcurrentHashMap.computeIfAbsent` | Built-in |
| Immutable collections | `List.of`, `Map.copyOf` | Built-in |
| Try monad | `Try.of()` | Vavr |
| Validation accumulation | `Validation` | Vavr |
| Persistent data structures | `List`, `Map`, `Set` | Vavr |

### Key Takeaways
- `andThen` = "then do this"; `compose` = "first do this"
- `flatMap` is the key to monadic chaining (Optional, Either, Stream)
- Either enables railway-oriented programming without exceptions
- Currying and partial application enable configuration-time binding
- Memoization with `computeIfAbsent` is thread-safe for concurrent access

---

## Next Part Preview

**Part 093: Protocol Buffers and gRPC Deep Dive** covers protobuf 3 syntax, all four gRPC streaming patterns, error handling with Status codes, interceptors for auth and logging, and a complete bidirectional streaming chat service implementation.
