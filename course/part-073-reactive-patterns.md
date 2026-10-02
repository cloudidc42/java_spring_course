# Part 073: Advanced Reactive Patterns

This part covers advanced Project Reactor patterns for building robust, backpressure-aware reactive applications with Spring WebFlux.

---

## Table of Contents

1. [Reactive Streams Specification](#reactive-streams)
2. [Hot vs Cold Publishers](#hot-cold)
3. [Advanced Operators: groupBy, window, buffer](#advanced-operators)
4. [flatMapMany and flatMap](#flatmap)
5. [Backpressure Strategies](#backpressure)
6. [Reactive Error Handling](#error-handling)
7. [Context Propagation](#context)
8. [Combining Publishers](#combining)
9. [Reactive Caching with Mono.cache()](#caching)
10. [Reactive Transactions](#transactions)
11. [Reactive Security](#security)
12. [Real Example: Real-Time Data Pipeline](#pipeline)

---

## 1. Reactive Streams Specification {#reactive-streams}

The Reactive Streams specification defines 4 interfaces:

```
Publisher  → produces data
Subscriber → consumes data
Subscription → connects Publisher and Subscriber, controls demand
Processor  → both Publisher and Subscriber
```

Project Reactor implements this with `Mono<T>` (0 or 1 item) and `Flux<T>` (0 to N items).

```java
// pom.xml dependencies
/*
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
*/
```

---

## 2. Hot vs Cold Publishers {#hot-cold}

```java
// src/main/java/com/example/reactive/HotColdDemo.java
package com.example.reactive;

import reactor.core.publisher.*;
import java.time.Duration;

public class HotColdDemo {

    /**
     * COLD publisher: each subscriber gets its own stream from the beginning.
     * Like a Netflix movie — each viewer starts at the beginning.
     */
    public static Flux<Integer> coldPublisher() {
        return Flux.range(1, 5)
            .doOnSubscribe(s -> System.out.println("New subscriber started from beginning"))
            .delayElements(Duration.ofMillis(100));
    }

    /**
     * HOT publisher: subscribers share the same stream.
     * Like live TV — you only see what's broadcast while you're watching.
     */
    public static Flux<Long> hotPublisher() {
        // publish() makes a ConnectableFlux (hot)
        return Flux.interval(Duration.ofMillis(100))
            .publish()
            .autoConnect(1); // Start when first subscriber subscribes
    }

    /**
     * Sinks.many() — manually push items to a hot publisher
     */
    public static Sinks.Many<String> createEventSink() {
        return Sinks.many().multicast().onBackpressureBuffer();
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== COLD Publisher ===");
        Flux<Integer> cold = coldPublisher();

        cold.subscribe(v -> System.out.println("Subscriber 1: " + v));
        Thread.sleep(200);
        cold.subscribe(v -> System.out.println("Subscriber 2: " + v)); // starts from 1

        System.out.println("\n=== HOT Publisher ===");
        Flux<Long> hot = hotPublisher();

        hot.subscribe(v -> System.out.println("Early subscriber: " + v));
        Thread.sleep(300);
        hot.subscribe(v -> System.out.println("Late subscriber: " + v)); // misses first items

        Thread.sleep(500);

        System.out.println("\n=== Sink as Hot Publisher ===");
        Sinks.Many<String> sink = createEventSink();
        Flux<String> eventStream = sink.asFlux();

        eventStream.subscribe(v -> System.out.println("Sub 1: " + v));
        eventStream.subscribe(v -> System.out.println("Sub 2: " + v));

        sink.tryEmitNext("Event A");
        sink.tryEmitNext("Event B");
        sink.tryEmitComplete();
    }
}
```

---

## 3. Advanced Operators {#advanced-operators}

### groupBy

```java
// src/main/java/com/example/reactive/GroupByExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import reactor.core.publisher.GroupedFlux;
import java.util.List;

public class GroupByExample {

    record Order(String customerId, String product, double amount) {}

    public static void groupOrdersByCustomer() {
        Flux<Order> orders = Flux.just(
            new Order("customer-1", "Laptop", 999.99),
            new Order("customer-2", "Mouse", 29.99),
            new Order("customer-1", "Keyboard", 79.99),
            new Order("customer-3", "Monitor", 399.99),
            new Order("customer-2", "Webcam", 89.99),
            new Order("customer-1", "Headset", 149.99)
        );

        orders
            .groupBy(Order::customerId)
            .flatMap(group -> group
                .collectList()
                .map(orderList -> {
                    double total = orderList.stream()
                        .mapToDouble(Order::amount)
                        .sum();
                    return "Customer " + group.key() + ": " + orderList.size() +
                           " orders, total $" + String.format("%.2f", total);
                })
            )
            .subscribe(System.out::println);
    }

    // Group and process with concatMap to maintain order within each group
    public static Flux<String> groupAndProcessOrdered(Flux<Order> orders) {
        return orders
            .groupBy(Order::customerId)
            .concatMap(group ->
                group
                    .reduce(0.0, (acc, order) -> acc + order.amount())
                    .map(total -> group.key() + "=" + String.format("%.2f", total))
            );
    }
}
```

### window

```java
// src/main/java/com/example/reactive/WindowExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import java.time.Duration;

public class WindowExample {

    /**
     * window(size) — splits stream into fixed-size windows
     */
    public static void fixedSizeWindow() {
        Flux.range(1, 12)
            .window(3) // Windows of 3
            .flatMap(window -> window.collectList())
            .subscribe(batch -> System.out.println("Batch: " + batch));
        // Output: [1,2,3], [4,5,6], [7,8,9], [10,11,12]
    }

    /**
     * windowTimeout — batch by count OR time, whichever comes first
     */
    public static Flux<List<Integer>> windowByCountOrTime(Flux<Integer> source) {
        return source
            .windowTimeout(10, Duration.ofSeconds(1))
            .flatMap(window -> window.collectList());
    }

    /**
     * Sliding window with windowUntil
     */
    public static void slidingWindow() {
        Flux.range(1, 10)
            .windowUntil(n -> n % 3 == 0, true) // window ends when divisible by 3, include boundary
            .flatMap(window -> window.collectList())
            .subscribe(w -> System.out.println("Window: " + w));
    }

    /**
     * windowWhile — window as long as predicate is true
     */
    public static void windowWhileExample() {
        Flux.just(1, 2, 3, 10, 11, 12, 1, 2)
            .windowWhile(n -> n < 10) // start new window when predicate fails
            .flatMap(window -> window.collectList())
            .subscribe(w -> System.out.println("Window: " + w));
    }
}
```

### buffer

```java
// src/main/java/com/example/reactive/BufferExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import java.time.Duration;
import java.util.List;

public class BufferExample {

    /**
     * buffer(size) — collects into List of fixed size
     */
    public static void fixedBuffer() {
        Flux.range(1, 10)
            .buffer(3)
            .subscribe(batch -> System.out.println("Buffer: " + batch));
    }

    /**
     * bufferTimeout — collect up to N items or until timeout
     * Great for batching database writes
     */
    public static Flux<List<String>> batchForDatabase(Flux<String> events) {
        return events
            .bufferTimeout(100, Duration.ofMillis(500))
            .filter(batch -> !batch.isEmpty());
    }

    /**
     * Sliding buffer with overlap
     * buffer(size, skip) where skip < size creates overlapping windows
     */
    public static void slidingBuffer() {
        Flux.range(1, 6)
            .buffer(3, 1) // window size 3, advance by 1 each time
            .subscribe(w -> System.out.println("Sliding: " + w));
        // [1,2,3], [2,3,4], [3,4,5], [4,5,6]
    }

    /**
     * bufferUntil — accumulate until predicate is true
     */
    public static void bufferUntilExample() {
        Flux.just("a", "b", "END", "c", "d", "END", "e")
            .bufferUntil(s -> s.equals("END"))
            .subscribe(batch -> System.out.println("Chunk: " + batch));
    }
}
```

---

## 4. flatMap and flatMapMany {#flatmap}

```java
// src/main/java/com/example/reactive/FlatMapExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;
import java.util.List;
import java.util.Random;

public class FlatMapExample {

    private static final Random random = new Random();

    /**
     * flatMap — concurrent, interleaved results
     * Use when order doesn't matter and you want max throughput
     */
    public static Flux<String> flatMapConcurrent(List<String> userIds) {
        return Flux.fromIterable(userIds)
            .flatMap(userId ->
                fetchUserData(userId).subscribeOn(Schedulers.boundedElastic()),
                4 // max concurrency = 4 parallel operations
            );
    }

    /**
     * concatMap — sequential, preserves order
     * Use when order matters
     */
    public static Flux<String> concatMapSequential(List<String> userIds) {
        return Flux.fromIterable(userIds)
            .concatMap(userId -> fetchUserData(userId));
    }

    /**
     * flatMapSequential — concurrent but results emitted in order
     * Best of both worlds: parallel execution, ordered results
     */
    public static Flux<String> flatMapSequentialOrdered(List<String> userIds) {
        return Flux.fromIterable(userIds)
            .flatMapSequential(userId ->
                fetchUserData(userId).subscribeOn(Schedulers.boundedElastic())
            );
    }

    /**
     * flatMapMany — expand one Mono into a Flux
     */
    public static Flux<String> expandToMultiple(String orderId) {
        return getOrder(orderId)
            .flatMapMany(order -> Flux.fromIterable(order.itemIds()))
            .flatMap(itemId -> getItemDetails(itemId));
    }

    private static Mono<String> fetchUserData(String userId) {
        int delay = 50 + random.nextInt(150);
        return Mono.delay(Duration.ofMillis(delay))
            .map(d -> "UserData[" + userId + "] after " + delay + "ms");
    }

    record Order(String id, List<String> itemIds) {}

    private static Mono<Order> getOrder(String orderId) {
        return Mono.just(new Order(orderId, List.of("item-1", "item-2", "item-3")));
    }

    private static Mono<String> getItemDetails(String itemId) {
        return Mono.just("Details for " + itemId);
    }
}
```

---

## 5. Backpressure Strategies {#backpressure}

```java
// src/main/java/com/example/reactive/BackpressureExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;

public class BackpressureExample {

    /**
     * onBackpressureBuffer — buffer excess items (risky: can OOM)
     */
    public static void bufferStrategy() {
        Flux.interval(Duration.ofMillis(1)) // fast producer: 1000 items/sec
            .onBackpressureBuffer(1000,     // buffer up to 1000
                item -> System.out.println("Buffer overflow, dropped: " + item))
            .publishOn(Schedulers.boundedElastic())
            .subscribe(item -> {
                try { Thread.sleep(10); } catch (InterruptedException e) {}
                // Slow consumer: 100 items/sec
            });
    }

    /**
     * onBackpressureDrop — drop newest items when overwhelmed
     */
    public static void dropStrategy() {
        Flux.interval(Duration.ofMillis(1))
            .onBackpressureDrop(dropped ->
                System.out.println("Dropped item: " + dropped))
            .publishOn(Schedulers.boundedElastic())
            .subscribe(item -> {
                try { Thread.sleep(10); } catch (InterruptedException e) {}
            });
    }

    /**
     * onBackpressureLatest — keep only the latest item
     * Useful for streaming sensor data where stale readings don't matter
     */
    public static void latestStrategy() {
        Flux.interval(Duration.ofMillis(1))
            .onBackpressureLatest()
            .publishOn(Schedulers.boundedElastic())
            .subscribe(item -> {
                System.out.println("Processing latest: " + item);
                try { Thread.sleep(100); } catch (InterruptedException e) {}
            });
    }

    /**
     * onBackpressureError — throw error when buffer full
     */
    public static void errorStrategy() {
        Flux.interval(Duration.ofMillis(1))
            .onBackpressureError()
            .publishOn(Schedulers.boundedElastic())
            .subscribe(
                item -> {
                    try { Thread.sleep(10); } catch (InterruptedException e) {}
                },
                error -> System.err.println("Backpressure error: " + error.getMessage())
            );
    }

    /**
     * limitRate — request items in batches to control demand
     * Ideal for database cursor-based reads
     */
    public static Flux<String> limitedRateExample(Flux<String> dataSource) {
        return dataSource
            .limitRate(50)  // request 50 at a time
            .publishOn(Schedulers.boundedElastic())
            .map(item -> item.toUpperCase());
    }
}
```

---

## 6. Reactive Error Handling {#error-handling}

```java
// src/main/java/com/example/reactive/ErrorHandlingExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

public class ErrorHandlingExample {

    /**
     * onErrorResume — switch to fallback publisher on error
     */
    public static Mono<String> withFallback(String userId) {
        return fetchFromPrimaryDb(userId)
            .onErrorResume(ex -> {
                System.out.println("Primary DB failed, trying replica: " + ex.getMessage());
                return fetchFromReplicaDb(userId);
            })
            .onErrorResume(ex -> {
                System.out.println("Replica also failed, returning cached: " + ex.getMessage());
                return getCachedValue(userId);
            });
    }

    /**
     * onErrorReturn — return default value on error
     */
    public static Mono<String> withDefault(String userId) {
        return fetchFromPrimaryDb(userId)
            .onErrorReturn("default-user-data");
    }

    /**
     * onErrorMap — transform error type
     */
    public static Mono<String> withMappedError(String userId) {
        return fetchFromPrimaryDb(userId)
            .onErrorMap(DatabaseException.class,
                ex -> new ServiceUnavailableException("DB unavailable: " + ex.getMessage()))
            .onErrorMap(ex -> !(ex instanceof ServiceUnavailableException),
                ex -> new InternalException("Unexpected error", ex));
    }

    /**
     * retry — simple retry on any error
     */
    public static Mono<String> withSimpleRetry(String userId) {
        return fetchFromPrimaryDb(userId)
            .retry(3);
    }

    /**
     * retryWhen with exponential backoff — the right way to retry
     */
    public static Mono<String> withExponentialBackoff(String userId) {
        return fetchFromPrimaryDb(userId)
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                .maxBackoff(Duration.ofSeconds(10))
                .jitter(0.5) // add jitter to avoid thundering herd
                .filter(ex -> ex instanceof DatabaseException) // only retry DB errors
                .doBeforeRetry(retrySignal ->
                    System.out.println("Retry attempt " + retrySignal.totalRetries() +
                        " after error: " + retrySignal.failure().getMessage()))
                .onRetryExhaustedThrow((spec, signal) ->
                    new MaxRetriesExceededException("Failed after 3 retries", signal.failure()))
            );
    }

    /**
     * doOnError — side effect on error (logging, metrics)
     * Does NOT handle the error — just observes it
     */
    public static Flux<String> withErrorLogging(Flux<String> source) {
        return source
            .doOnError(ex -> System.err.println("Error in stream: " + ex.getMessage()))
            .onErrorResume(ex -> Flux.empty()); // then handle
    }

    /**
     * materialize/dematerialize — wrap signals for error-tolerant processing
     */
    public static Flux<String> processWithErrorTolerance(Flux<String> items) {
        return items
            .flatMap(item ->
                processItem(item)
                    .onErrorResume(ex -> {
                        System.err.println("Failed to process " + item + ": " + ex.getMessage());
                        return Mono.empty(); // skip failed items
                    })
            );
    }

    /**
     * retryWhen with custom condition and state
     */
    public static Mono<String> withConditionalRetry() {
        AtomicInteger attempts = new AtomicInteger();
        return Mono.fromSupplier(() -> {
                int attempt = attempts.incrementAndGet();
                System.out.println("Attempt: " + attempt);
                if (attempt < 3) throw new RuntimeException("Transient error");
                return "Success on attempt " + attempt;
            })
            .retryWhen(Retry.max(5)
                .filter(ex -> ex instanceof RuntimeException)
            );
    }

    // Helper methods (simulated)
    private static Mono<String> fetchFromPrimaryDb(String userId) {
        return Mono.error(new DatabaseException("Primary DB unreachable"));
    }
    private static Mono<String> fetchFromReplicaDb(String userId) {
        return Mono.just("data-from-replica-" + userId);
    }
    private static Mono<String> getCachedValue(String userId) {
        return Mono.just("cached-" + userId);
    }
    private static Mono<String> processItem(String item) {
        return Mono.just(item.toUpperCase());
    }

    static class DatabaseException extends RuntimeException {
        DatabaseException(String msg) { super(msg); }
    }
    static class ServiceUnavailableException extends RuntimeException {
        ServiceUnavailableException(String msg) { super(msg); }
    }
    static class InternalException extends RuntimeException {
        InternalException(String msg, Throwable cause) { super(msg, cause); }
    }
    static class MaxRetriesExceededException extends RuntimeException {
        MaxRetriesExceededException(String msg, Throwable cause) { super(msg, cause); }
    }
}
```

---

## 7. Context Propagation {#context}

```java
// src/main/java/com/example/reactive/ContextExample.java
package com.example.reactive;

import reactor.core.publisher.Mono;
import reactor.util.context.Context;

public class ContextExample {

    static final String TRACE_ID_KEY = "traceId";
    static final String USER_ID_KEY = "userId";

    /**
     * Reactor Context flows upstream — set it where you subscribe,
     * read it anywhere in the chain.
     */
    public static Mono<String> processRequest() {
        return enrichWithTraceId()
            .flatMap(data -> logWithContext(data))
            .flatMap(data -> callDownstreamService(data));
    }

    private static Mono<String> enrichWithTraceId() {
        return Mono.deferContextual(ctx -> {
            String traceId = ctx.getOrDefault(TRACE_ID_KEY, "no-trace");
            String userId = ctx.getOrDefault(USER_ID_KEY, "anonymous");
            System.out.println("[" + traceId + "] User: " + userId);
            return Mono.just("enriched-data");
        });
    }

    private static Mono<String> logWithContext(String data) {
        return Mono.deferContextual(ctx -> {
            String traceId = ctx.getOrDefault(TRACE_ID_KEY, "no-trace");
            System.out.println("[" + traceId + "] Processing: " + data);
            return Mono.just(data);
        });
    }

    private static Mono<String> callDownstreamService(String data) {
        return Mono.deferContextual(ctx -> {
            String traceId = ctx.getOrDefault(TRACE_ID_KEY, "no-trace");
            // Pass trace ID to downstream HTTP calls, etc.
            return Mono.just("result-for-" + data);
        });
    }

    public static void main(String[] args) {
        processRequest()
            .contextWrite(Context.of(
                TRACE_ID_KEY, "trace-abc-123",
                USER_ID_KEY, "user-456"
            ))
            .subscribe(result -> System.out.println("Result: " + result));
    }
}
```

```java
// src/main/java/com/example/reactive/ReactiveSecurityContext.java
package com.example.reactive;

import reactor.core.publisher.Mono;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.core.Authentication;

public class ReactiveSecurityContext {

    /**
     * Access Spring Security context in reactive chain
     */
    public static Mono<String> getAuthenticatedUsername() {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication())
            .map(Authentication::getName)
            .defaultIfEmpty("anonymous");
    }

    /**
     * Build audit log entries with user context
     */
    public static Mono<String> createAuditEntry(String action, String resourceId) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .defaultIfEmpty("system")
            .map(username ->
                String.format("AUDIT: user=%s, action=%s, resource=%s",
                    username, action, resourceId)
            );
    }
}
```

---

## 8. Combining Publishers {#combining}

```java
// src/main/java/com/example/reactive/CombiningExample.java
package com.example.reactive;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.time.Duration;
import java.util.List;

public class CombiningExample {

    /**
     * zip — combine items one-by-one from multiple publishers
     * Waits for ALL publishers to emit before combining
     */
    public static Mono<UserProfile> buildUserProfile(String userId) {
        Mono<String> username = getUsername(userId);
        Mono<String> email = getEmail(userId);
        Mono<List<String>> roles = getRoles(userId);

        return Mono.zip(username, email, roles)
            .map(tuple -> new UserProfile(tuple.getT1(), tuple.getT2(), tuple.getT3()));
    }

    /**
     * zipWith — zip with one other publisher
     */
    public static Flux<String> zipWithExample() {
        Flux<String> names = Flux.just("Alice", "Bob", "Charlie");
        Flux<Integer> scores = Flux.just(90, 85, 95);

        return names.zipWith(scores, (name, score) -> name + ": " + score);
    }

    /**
     * combineLatest — re-combine whenever ANY publisher emits
     * Great for real-time dashboards
     */
    public static Flux<String> dashboard() {
        Flux<Integer> cpuUsage = Flux.interval(Duration.ofSeconds(1))
            .map(i -> (int)(Math.random() * 100));

        Flux<Integer> memoryUsage = Flux.interval(Duration.ofSeconds(2))
            .map(i -> (int)(Math.random() * 100));

        return Flux.combineLatest(cpuUsage, memoryUsage,
            (cpu, mem) -> "CPU: " + cpu + "%, Memory: " + mem + "%");
    }

    /**
     * merge — interleave items from multiple publishers concurrently
     */
    public static Flux<String> mergeStreams() {
        Flux<String> stream1 = Flux.interval(Duration.ofMillis(100))
            .map(i -> "Stream1: " + i).take(5);
        Flux<String> stream2 = Flux.interval(Duration.ofMillis(150))
            .map(i -> "Stream2: " + i).take(5);

        return Flux.merge(stream1, stream2); // Items interleaved by arrival time
    }

    /**
     * concat — subscribe to publishers sequentially
     */
    public static Flux<String> concatStreams() {
        Flux<String> first = Flux.just("A", "B", "C");
        Flux<String> second = Flux.just("D", "E", "F");

        return Flux.concat(first, second); // A,B,C then D,E,F
    }

    /**
     * mergeWith — combine this flux with another
     */
    public static Flux<Integer> mergeWith() {
        return Flux.just(1, 3, 5)
            .mergeWith(Flux.just(2, 4, 6)); // Interleaved
    }

    /**
     * switchMap — cancel previous inner publisher when new item arrives
     * Great for search-as-you-type (cancel previous search)
     */
    public static Flux<String> searchAsYouType(Flux<String> searchTerms) {
        return searchTerms
            .switchMap(term ->
                searchDatabase(term)
                    .delayElements(Duration.ofMillis(100)) // Simulate DB query
            );
    }

    // Simulated methods
    record UserProfile(String username, String email, List<String> roles) {}
    private static Mono<String> getUsername(String id) { return Mono.just("user-" + id); }
    private static Mono<String> getEmail(String id) { return Mono.just(id + "@example.com"); }
    private static Mono<List<String>> getRoles(String id) { return Mono.just(List.of("USER")); }
    private static Flux<String> searchDatabase(String term) {
        return Flux.just("Result 1 for " + term, "Result 2 for " + term);
    }
}
```

---

## 9. Reactive Caching with Mono.cache() {#caching}

```java
// src/main/java/com/example/reactive/ReactiveCaching.java
package com.example.reactive;

import reactor.core.publisher.Mono;
import java.time.Duration;
import java.util.concurrent.ConcurrentHashMap;

public class ReactiveCaching {

    private final ConcurrentHashMap<String, Mono<String>> cache = new ConcurrentHashMap<>();

    /**
     * cache() — memoize a Mono so it executes only once,
     * even with multiple subscribers
     */
    public Mono<String> getCachedConfig(String key) {
        return cache.computeIfAbsent(key, k ->
            loadExpensiveConfig(k)
                .cache() // All subscribers share the same result
                .doOnSubscribe(s -> System.out.println("Loading config for: " + k))
        );
    }

    /**
     * cache(Duration) — expire after TTL
     */
    public Mono<String> getCachedWithTtl(String key) {
        return cache.computeIfAbsent(key, k ->
            loadExpensiveConfig(k)
                .cache(Duration.ofMinutes(5)) // Cache for 5 minutes
        );
    }

    /**
     * CacheMono from reactor-extra for per-key caching with eviction
     */
    public static Mono<String> withCacheMono(String key) {
        // Using Caffeine cache as backend (optional dependency)
        // In practice use: reactor.extra.cache.CacheMono from reactor-extra
        // For demonstration, showing the pattern manually
        return Mono.defer(() -> {
            System.out.println("Cache miss for: " + key);
            return loadExpensiveConfig(key);
        }).cache(Duration.ofMinutes(5));
    }

    /**
     * share() — share a Flux among multiple subscribers
     * Like cache() but for ongoing streams
     */
    public static Flux<String> sharedStream() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(i -> "Event-" + i)
            .share(); // Multiple subscribers share the same source
    }

    private Mono<String> loadExpensiveConfig(String key) {
        return Mono.delay(Duration.ofMillis(500))
            .map(d -> "config-value-for-" + key)
            .doOnNext(v -> System.out.println("Loaded: " + v));
    }
}
```

---

## 10. Reactive Transactions {#transactions}

```java
// src/main/java/com/example/reactive/ReactiveTransactionExample.java
package com.example.reactive;

import org.springframework.r2dbc.connection.R2dbcTransactionManager;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.reactive.TransactionalOperator;
import reactor.core.publisher.Mono;

@Service
public class ReactiveTransactionExample {

    private final TransactionalOperator transactionalOperator;
    private final ReactiveOrderRepository orderRepository;
    private final ReactiveInventoryRepository inventoryRepository;

    public ReactiveTransactionExample(
            TransactionalOperator transactionalOperator,
            ReactiveOrderRepository orderRepository,
            ReactiveInventoryRepository inventoryRepository) {
        this.transactionalOperator = transactionalOperator;
        this.orderRepository = orderRepository;
        this.inventoryRepository = inventoryRepository;
    }

    /**
     * @Transactional works with reactive return types (Mono/Flux)
     */
    @Transactional
    public Mono<String> placeOrderDeclarative(String orderId, String productId, int quantity) {
        return inventoryRepository.findByProductId(productId)
            .flatMap(inventory -> {
                if (inventory.quantity() < quantity) {
                    return Mono.error(new InsufficientStockException("Not enough stock"));
                }
                return inventoryRepository.decrementStock(productId, quantity)
                    .then(orderRepository.create(orderId, productId, quantity));
            });
    }

    /**
     * Programmatic transaction control with TransactionalOperator
     */
    public Mono<String> placeOrderProgrammatic(String orderId, String productId, int quantity) {
        Mono<String> operation = inventoryRepository.findByProductId(productId)
            .flatMap(inventory -> {
                if (inventory.quantity() < quantity) {
                    return Mono.error(new InsufficientStockException("Not enough stock"));
                }
                return inventoryRepository.decrementStock(productId, quantity)
                    .then(orderRepository.create(orderId, productId, quantity));
            });

        return transactionalOperator.transactional(operation);
    }

    /**
     * Manual transaction with as() operator
     */
    public Mono<Void> transferFunds(String fromAccount, String toAccount, long amount) {
        return Mono.just(amount)
            .flatMap(amt -> debitAccount(fromAccount, amt))
            .flatMap(amt -> creditAccount(toAccount, amt))
            .as(transactionalOperator::transactional)
            .then();
    }

    private Mono<Long> debitAccount(String accountId, long amount) {
        return Mono.just(amount); // Simplified
    }

    private Mono<Long> creditAccount(String accountId, long amount) {
        return Mono.just(amount); // Simplified
    }

    record InventoryItem(String productId, int quantity) {}

    interface ReactiveOrderRepository {
        Mono<String> create(String orderId, String productId, int quantity);
    }

    interface ReactiveInventoryRepository {
        Mono<InventoryItem> findByProductId(String productId);
        Mono<Void> decrementStock(String productId, int quantity);
    }

    static class InsufficientStockException extends RuntimeException {
        InsufficientStockException(String msg) { super(msg); }
    }
}
```

---

## 11. Reactive Security with Spring WebFlux {#security}

```java
// src/main/java/com/example/reactive/security/ReactiveSecurityConfig.java
package com.example.reactive.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.security.web.server.context.WebSessionServerSecurityContextRepository;
import reactor.core.publisher.Mono;

@Configuration
@EnableWebFluxSecurity
public class ReactiveSecurityConfig {

    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/api/public/**").permitAll()
                .pathMatchers("/api/admin/**").hasRole("ADMIN")
                .anyExchange().authenticated()
            )
            .httpBasic(httpBasic -> httpBasic
                .authenticationEntryPoint(new CustomAuthenticationEntryPoint())
            )
            .build();
    }
}
```

```java
// src/main/java/com/example/reactive/security/ReactiveJwtFilter.java
package com.example.reactive.security;

import org.springframework.http.HttpHeaders;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.web.server.WebFilter;
import org.springframework.web.server.WebFilterChain;
import reactor.core.publisher.Mono;
import java.util.List;

@Component
public class ReactiveJwtFilter implements WebFilter {

    private final JwtTokenValidator tokenValidator;

    public ReactiveJwtFilter(JwtTokenValidator tokenValidator) {
        this.tokenValidator = tokenValidator;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String authHeader = exchange.getRequest()
            .getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return chain.filter(exchange);
        }

        String token = authHeader.substring(7);

        return tokenValidator.validate(token)
            .flatMap(claims -> {
                String username = claims.getSubject();
                List<SimpleGrantedAuthority> authorities = claims.getRoles().stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                    .toList();

                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(username, null, authorities);

                return chain.filter(exchange)
                    .contextWrite(ReactiveSecurityContextHolder.withAuthentication(auth));
            })
            .onErrorResume(ex -> chain.filter(exchange)); // Skip auth on invalid token
    }

    interface JwtTokenValidator {
        Mono<JwtClaims> validate(String token);
    }

    record JwtClaims(String subject, List<String> roles) {
        String getSubject() { return subject; }
        List<String> getRoles() { return roles; }
    }
}
```

---

## 12. Real Example: Real-Time Data Pipeline with Backpressure {#pipeline}

```java
// src/main/java/com/example/reactive/pipeline/SensorDataPipeline.java
package com.example.reactive.pipeline;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;
import java.time.Instant;
import java.util.*;

public class SensorDataPipeline {

    record SensorReading(String sensorId, double value, String unit, Instant timestamp) {}
    record ProcessedReading(String sensorId, double rawValue, double smoothedValue,
                            boolean isAnomaly, Instant timestamp) {}
    record AggregatedStats(String sensorId, double avg, double min, double max,
                           long count, Instant windowEnd) {}

    private final Sinks.Many<SensorReading> sensorSink;
    private final Flux<SensorReading> sensorStream;

    public SensorDataPipeline() {
        this.sensorSink = Sinks.many().multicast().onBackpressureBuffer(10000);
        this.sensorStream = sensorSink.asFlux().share();
    }

    /**
     * Ingest a new reading (called by sensor adapters)
     */
    public void ingest(SensorReading reading) {
        Sinks.EmitResult result = sensorSink.tryEmitNext(reading);
        if (result.isFailure()) {
            System.err.println("Failed to ingest reading from sensor: " + reading.sensorId() +
                ", result: " + result);
        }
    }

    /**
     * Pipeline Stage 1: Filter, validate, and deduplicate readings
     */
    public Flux<SensorReading> validatedStream() {
        Set<String> recentReadingIds = Collections.synchronizedSet(
            Collections.newSetFromMap(new LinkedHashMap<>() {
                @Override protected boolean removeEldestEntry(Map.Entry eldest) {
                    return size() > 1000;
                }
            })
        );

        return sensorStream
            .filter(r -> r.value() != Double.NaN)                    // remove NaN
            .filter(r -> r.value() >= -273.15)                       // above absolute zero
            .filter(r -> r.sensorId() != null && !r.sensorId().isBlank())
            .filter(r -> {                                            // deduplicate
                String key = r.sensorId() + "_" + r.timestamp().toEpochMilli();
                return recentReadingIds.add(key);
            })
            .doOnNext(r -> System.out.println("Valid reading: " + r.sensorId()));
    }

    /**
     * Pipeline Stage 2: Smooth values using sliding window average
     */
    public Flux<ProcessedReading> smoothedStream() {
        return validatedStream()
            .groupBy(SensorReading::sensorId)
            .flatMap(group ->
                group
                    .window(5, 1)  // sliding window of size 5, advance by 1
                    .flatMap(window ->
                        window.collectList()
                            .filter(readings -> !readings.isEmpty())
                            .map(readings -> {
                                double raw = readings.get(readings.size() - 1).value();
                                double smoothed = readings.stream()
                                    .mapToDouble(SensorReading::value)
                                    .average()
                                    .orElse(raw);
                                boolean isAnomaly = Math.abs(raw - smoothed) > 3 * computeStdDev(readings, smoothed);
                                return new ProcessedReading(
                                    group.key(), raw, smoothed, isAnomaly,
                                    readings.get(readings.size() - 1).timestamp()
                                );
                            })
                    )
            )
            .subscribeOn(Schedulers.parallel());
    }

    /**
     * Pipeline Stage 3: Compute statistics per 1-minute tumbling window
     */
    public Flux<AggregatedStats> aggregatedStream() {
        return smoothedStream()
            .groupBy(ProcessedReading::sensorId)
            .flatMap(group ->
                group
                    .windowTimeout(1000, Duration.ofMinutes(1)) // 1000 items or 1 minute
                    .flatMap(window ->
                        window.collectList()
                            .filter(readings -> !readings.isEmpty())
                            .map(readings -> {
                                double avg = readings.stream()
                                    .mapToDouble(ProcessedReading::smoothedValue)
                                    .average().orElse(0);
                                double min = readings.stream()
                                    .mapToDouble(ProcessedReading::smoothedValue)
                                    .min().orElse(0);
                                double max = readings.stream()
                                    .mapToDouble(ProcessedReading::smoothedValue)
                                    .max().orElse(0);
                                return new AggregatedStats(
                                    group.key(), avg, min, max,
                                    readings.size(), Instant.now()
                                );
                            })
                    )
            );
    }

    /**
     * Pipeline Stage 4: Anomaly alerting stream
     */
    public Flux<String> anomalyAlerts() {
        return smoothedStream()
            .filter(ProcessedReading::isAnomaly)
            .buffer(Duration.ofSeconds(10)) // batch alerts, send every 10 seconds
            .filter(batch -> !batch.isEmpty())
            .map(batch -> {
                StringBuilder alert = new StringBuilder("ANOMALY ALERT:\n");
                batch.forEach(r -> alert.append(String.format(
                    "  Sensor %s: raw=%.2f, smoothed=%.2f at %s\n",
                    r.sensorId(), r.rawValue(), r.smoothedValue(), r.timestamp()
                )));
                return alert.toString();
            });
    }

    private double computeStdDev(List<SensorReading> readings, double mean) {
        if (readings.size() < 2) return 0;
        double variance = readings.stream()
            .mapToDouble(r -> Math.pow(r.value() - mean, 2))
            .average()
            .orElse(0);
        return Math.sqrt(variance);
    }
}
```

```java
// src/main/java/com/example/reactive/pipeline/SensorDataController.java
package com.example.reactive.pipeline;

import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/sensors")
public class SensorDataController {

    private final SensorDataPipeline pipeline;

    public SensorDataController(SensorDataPipeline pipeline) {
        this.pipeline = pipeline;
    }

    /**
     * Server-Sent Events stream — browser can subscribe to live data
     */
    @GetMapping(value = "/stream/processed", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<SensorDataPipeline.ProcessedReading> streamProcessed() {
        return pipeline.smoothedStream()
            .onBackpressureLatest(); // Drop old readings if client is slow
    }

    @GetMapping(value = "/stream/stats", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<SensorDataPipeline.AggregatedStats> streamStats() {
        return pipeline.aggregatedStream();
    }

    @GetMapping(value = "/stream/alerts", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamAlerts() {
        return pipeline.anomalyAlerts();
    }

    @PostMapping("/ingest")
    public void ingestReading(@RequestBody SensorDataPipeline.SensorReading reading) {
        pipeline.ingest(reading);
    }
}
```

```java
// src/test/java/com/example/reactive/pipeline/SensorDataPipelineTest.java
package com.example.reactive.pipeline;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import reactor.test.StepVerifier;

import java.time.Instant;

class SensorDataPipelineTest {

    private SensorDataPipeline pipeline;

    @BeforeEach
    void setUp() {
        pipeline = new SensorDataPipeline();
    }

    @Test
    void shouldFilterInvalidReadings() {
        StepVerifier.create(pipeline.validatedStream().take(1))
            .then(() -> {
                // Send invalid reading (NaN)
                pipeline.ingest(new SensorDataPipeline.SensorReading(
                    "sensor-1", Double.NaN, "celsius", Instant.now()));
                // Send valid reading
                pipeline.ingest(new SensorDataPipeline.SensorReading(
                    "sensor-1", 25.5, "celsius", Instant.now()));
            })
            .expectNextMatches(r -> r.value() == 25.5)
            .thenCancel()
            .verify();
    }

    @Test
    void shouldBatchReadings() {
        StepVerifier.create(
            pipeline.validatedStream()
                .buffer(3)
                .take(1)
        )
        .then(() -> {
            for (int i = 0; i < 3; i++) {
                pipeline.ingest(new SensorDataPipeline.SensorReading(
                    "sensor-" + i, 20.0 + i, "celsius", Instant.now()));
            }
        })
        .expectNextMatches(batch -> batch.size() == 3)
        .thenCancel()
        .verify();
    }
}
```

---

## Summary

| Operator | Use Case | Notes |
|---|---|---|
| `groupBy` | Partition stream by key | Each group is its own Flux |
| `window` | Split stream into time/count windows | Returns Flux of Flux |
| `buffer` | Collect into List chunks | Returns Flux of List |
| `flatMap` | Concurrent async mapping | May interleave results |
| `concatMap` | Sequential async mapping | Preserves order |
| `flatMapSequential` | Parallel but ordered | Best of both worlds |
| `onBackpressureBuffer` | Buffer overflow | Risk of OOM |
| `onBackpressureDrop` | Drop on overflow | Loses data |
| `onBackpressureLatest` | Keep newest only | Good for live data |
| `retryWhen` | Retry with policy | Use exponential backoff |
| `onErrorResume` | Fallback publisher | Most flexible error handling |
| `zip` | Combine multiple publishers | All must emit |
| `combineLatest` | React to any publisher | Emits on each update |
| `switchMap` | Cancel previous, subscribe to new | Search-as-you-type |
| `Mono.cache()` | Memoize expensive calls | Thread-safe |
| `share()` | Hot broadcast | Multiple subscribers |

---

## Next Part Preview

**Part 074: GraalVM Native Image with Spring Boot** — compile your Spring Boot application to a native executable with 10x faster startup and drastically reduced memory usage using GraalVM AOT compilation.
