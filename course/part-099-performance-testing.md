# Part 099: Performance Testing and Optimization

## Introduction

Performance problems in production are expensive. This part teaches you to find bottlenecks systematically — from JVM-level profiling down to individual SQL queries — and shows a complete optimization case study taking a 5-second endpoint to under 50ms.

---

## Performance Investigation Methodology

```
1. MEASURE first — don't optimize blind
2. Identify the biggest bottleneck (Amdahl's Law)
3. Fix one thing at a time
4. Verify the fix actually helped
5. Repeat
```

---

## JVM Profiling with JFR (Java Flight Recorder)

```java
// src/main/java/com/example/perf/JfrConfig.java
package com.example.perf;

import jdk.jfr.Configuration;
import jdk.jfr.Recording;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;
import java.nio.file.Path;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicReference;

/**
 * HTTP endpoints to start/stop JFR recordings on demand.
 * In production, restrict these endpoints to admin users.
 */
@Slf4j
@RestController
@RequestMapping("/internal/profiling")
public class JfrController {

    private final AtomicReference<Recording> activeRecording = new AtomicReference<>();

    @PostMapping("/jfr/start")
    public String startRecording() throws Exception {
        if (activeRecording.get() != null) {
            return "Recording already in progress";
        }

        Configuration config = Configuration.getConfiguration("profile");
        Recording recording = new Recording(config);
        recording.enable("jdk.ObjectAllocationInNewTLAB").withThreshold(Duration.ZERO);
        recording.enable("jdk.CPULoad").withPeriod(Duration.ofSeconds(1));
        recording.enable("jdk.GarbageCollection").withThreshold(Duration.ZERO);
        recording.enable("jdk.JavaMonitorWait").withThreshold(Duration.ofMillis(10));
        recording.enable("jdk.SocketRead").withThreshold(Duration.ofMillis(10));
        recording.enable("jdk.FileWrite").withThreshold(Duration.ofMillis(10));
        recording.enable("jdk.Compilation").withThreshold(Duration.ZERO);
        recording.enable("jdk.MethodSample").withPeriod(Duration.ofMillis(20));

        recording.start();
        activeRecording.set(recording);
        log.info("JFR recording started");
        return "Recording started";
    }

    @PostMapping("/jfr/stop")
    public String stopRecording() throws IOException {
        Recording recording = activeRecording.getAndSet(null);
        if (recording == null) {
            return "No recording in progress";
        }

        recording.stop();
        Path outputPath = Path.of("/tmp/recording-" + System.currentTimeMillis() + ".jfr");
        recording.dump(outputPath);
        recording.close();

        log.info("JFR recording saved to {}", outputPath);
        return "Recording saved to " + outputPath;
    }
}
```

```bash
# Start recording with async-profiler (attach to running JVM)
# async-profiler can profile JVM without JFR, including native code
./profiler.sh -d 30 -e cpu -f /tmp/profile.html $(jps | grep Application | cut -d' ' -f1)

# Generate flame graph
./profiler.sh -d 30 -e cpu --format flamegraph -f /tmp/flamegraph.html $(pgrep -f "java.*Application")

# Profile memory allocations
./profiler.sh -d 30 -e alloc -f /tmp/alloc.html $(pgrep -f "java.*Application")

# Profile lock contention
./profiler.sh -d 30 -e lock -f /tmp/locks.html $(pgrep -f "java.*Application")
```

---

## Thread Dump Analysis

```java
// src/main/java/com/example/perf/ThreadDumpController.java
package com.example.perf;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.Arrays;
import java.util.Map;
import java.util.stream.Collectors;

@Slf4j
@RestController
@RequestMapping("/internal/threads")
public class ThreadDumpController {

    @GetMapping("/dump")
    public Map<String, Object> threadDump() {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] infos = bean.dumpAllThreads(true, true);

        Map<Thread.State, Long> stateDistribution = Arrays.stream(infos)
            .collect(Collectors.groupingBy(ThreadInfo::getThreadState, Collectors.counting()));

        long[] deadlocked = bean.findDeadlockedThreads();

        return Map.of(
            "total_threads", infos.length,
            "state_distribution", stateDistribution,
            "deadlocked_threads", deadlocked != null ? deadlocked.length : 0,
            "threads", Arrays.stream(infos)
                .filter(t -> t.getThreadState() != Thread.State.WAITING)
                .map(this::summarize)
                .collect(Collectors.toList())
        );
    }

    @GetMapping("/deadlocks")
    public Map<String, Object> checkDeadlocks() {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        long[] deadlocked = bean.findDeadlockedThreads();

        if (deadlocked == null) {
            return Map.of("deadlocked", false);
        }

        ThreadInfo[] infos = bean.getThreadInfo(deadlocked, 20);
        return Map.of(
            "deadlocked", true,
            "count", deadlocked.length,
            "threads", Arrays.stream(infos).map(this::summarize).toList()
        );
    }

    private Map<String, Object> summarize(ThreadInfo info) {
        StackTraceElement[] stack = info.getStackTrace();
        return Map.of(
            "name", info.getThreadName(),
            "state", info.getThreadState(),
            "cpu_time_ms", ManagementFactory.getThreadMXBean()
                .getThreadCpuTime(info.getThreadId()) / 1_000_000,
            "waiting_on", info.getLockName() != null ? info.getLockName() : "nothing",
            "top_frame", stack.length > 0 ? stack[0].toString() : "no frame"
        );
    }
}
```

---

## GC Tuning

```bash
# GC tuning flags for different collectors

# G1GC (default in Java 9+, recommended for most apps)
JAVA_OPTS="\
  -XX:+UseG1GC \
  -Xms512m -Xmx2g \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:+PrintGCDateStamps \
  -XX:+PrintGCDetails \
  -Xloggc:/var/log/gc.log \
  -XX:+UseGCLogFileRotation \
  -XX:NumberOfGCLogFiles=10 \
  -XX:GCLogFileSize=10m"

# ZGC (Java 15+, sub-millisecond pauses, good for low-latency services)
JAVA_OPTS="\
  -XX:+UseZGC \
  -Xms2g -Xmx8g \
  -XX:SoftMaxHeapSize=6g \
  -XX:+ZGenerational"  # Java 21+ generational ZGC

# Shenandoah (Red Hat, similar goals to ZGC)
JAVA_OPTS="\
  -XX:+UseShenandoahGC \
  -Xms1g -Xmx4g \
  -XX:ShenandoahGCMode=iu"  # Incremental-Update mode (default)

# Check GC overhead in production (non-intrusive)
jstat -gcutil $(pgrep -f "java.*Application") 5000 20
# Columns: S0 S1 E O M CCS YGC YGCT FGC FGCT GCT
# Watch for: high FGC (Full GC) count means memory pressure
```

```java
// src/main/java/com/example/perf/GcMetricsConfig.java
package com.example.perf;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.JvmGcMetrics;
import io.micrometer.core.instrument.binder.jvm.JvmHeapPressureMetrics;
import io.micrometer.core.instrument.binder.jvm.JvmMemoryMetrics;
import io.micrometer.core.instrument.binder.jvm.JvmThreadMetrics;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GcMetricsConfig {

    @Bean
    public JvmGcMetrics jvmGcMetrics() {
        return new JvmGcMetrics();
    }

    @Bean
    public JvmMemoryMetrics jvmMemoryMetrics() {
        return new JvmMemoryMetrics();
    }

    @Bean
    public JvmHeapPressureMetrics jvmHeapPressureMetrics() {
        return new JvmHeapPressureMetrics();
    }

    @Bean
    public JvmThreadMetrics jvmThreadMetrics() {
        return new JvmThreadMetrics();
    }
}
```

---

## JMH Benchmarking

```java
// src/jmh/java/com/example/benchmark/StringConcatBenchmark.java
package com.example.benchmark;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
public class StringConcatBenchmark {

    @Param({"10", "100", "1000"})
    private int iterations;

    @Benchmark
    public String concatenation_plus() {
        String result = "";
        for (int i = 0; i < iterations; i++) {
            result += "item" + i;
        }
        return result;
    }

    @Benchmark
    public String concatenation_stringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            sb.append("item").append(i);
        }
        return sb.toString();
    }

    @Benchmark
    public String concatenation_formatted() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < iterations; i++) {
            sb.append(String.format("item%d", i));
        }
        return sb.toString();
    }

    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
            .include(StringConcatBenchmark.class.getSimpleName())
            .forks(2)
            .build();
        new Runner(opt).run();
    }
}
```

```java
// src/jmh/java/com/example/benchmark/CollectionBenchmark.java
package com.example.benchmark;

import org.openjdk.jmh.annotations.*;
import java.util.*;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Thread)
@Fork(1)
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class CollectionBenchmark {

    @Param({"100", "10000", "1000000"})
    private int size;

    private List<Integer> data;

    @Setup
    public void setup() {
        data = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            data.add(i);
        }
    }

    @Benchmark
    public long streamSum() {
        return data.stream().mapToLong(Integer::longValue).sum();
    }

    @Benchmark
    public long parallelStreamSum() {
        return data.parallelStream().mapToLong(Integer::longValue).sum();
    }

    @Benchmark
    public long forLoopSum() {
        long sum = 0;
        for (int val : data) {
            sum += val;
        }
        return sum;
    }

    @Benchmark
    public List<Integer> streamFilter() {
        return data.stream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
    }
}
```

---

## k6 Load Testing Script

```javascript
// scripts/k6/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Counter, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const slowRequests = new Counter('slow_requests');
const apiDuration = new Trend('api_duration');

export const options = {
    scenarios: {
        // Ramp up test
        ramp_up: {
            executor: 'ramping-vus',
            startVUs: 0,
            stages: [
                { duration: '1m', target: 10 },   // Warm up
                { duration: '3m', target: 50 },   // Ramp up to 50 users
                { duration: '2m', target: 100 },  // Peak load
                { duration: '1m', target: 0 },    // Cool down
            ],
        },
        // Spike test (run after ramp_up)
        spike: {
            executor: 'ramping-vus',
            startTime: '7m',
            startVUs: 0,
            stages: [
                { duration: '30s', target: 0 },
                { duration: '10s', target: 200 }, // Sudden spike
                { duration: '1m', target: 200 },  // Hold
                { duration: '30s', target: 0 },   // Drop
            ],
        },
    },
    thresholds: {
        http_req_duration: ['p(95)<500', 'p(99)<2000'],
        http_req_failed: ['rate<0.01'],  // Less than 1% errors
        errors: ['rate<0.01'],
    },
};

const BASE_URL = 'http://localhost:8080';

// Pre-create authentication token
const AUTH_TOKEN = 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';

export function setup() {
    // One-time setup: create test data
    const response = http.post(`${BASE_URL}/api/test-setup`, null, {
        headers: { Authorization: AUTH_TOKEN }
    });
    return { testOrderId: JSON.parse(response.body).orderId };
}

export default function (data) {
    const headers = {
        'Content-Type': 'application/json',
        'Authorization': AUTH_TOKEN,
    };

    // Scenario 1: Browse products (60% of traffic)
    if (Math.random() < 0.6) {
        const res = http.get(`${BASE_URL}/api/products?page=0&size=20`, { headers });
        const ok = check(res, {
            'product list status 200': (r) => r.status === 200,
            'product list has data': (r) => JSON.parse(r.body)._embedded !== undefined,
        });

        errorRate.add(!ok);
        apiDuration.add(res.timings.duration, { endpoint: 'product_list' });

        if (res.timings.duration > 500) {
            slowRequests.add(1, { endpoint: 'product_list' });
        }
    }

    // Scenario 2: View single product (30% of traffic)
    else if (Math.random() < 0.9) {
        const productId = Math.floor(Math.random() * 100) + 1;
        const res = http.get(`${BASE_URL}/api/products/${productId}`, { headers });

        const ok = check(res, {
            'product detail status 200': (r) => r.status === 200,
        });
        errorRate.add(!ok);
        apiDuration.add(res.timings.duration, { endpoint: 'product_detail' });
    }

    // Scenario 3: Create order (10% of traffic)
    else {
        const payload = JSON.stringify({
            customerId: `customer-${__VU}`,
            items: [{ productId: 1, quantity: 2, unitPrice: 49.99 }]
        });

        const res = http.post(`${BASE_URL}/api/orders`, payload, { headers });
        const ok = check(res, {
            'order created status 201': (r) => r.status === 201,
            'order has id': (r) => JSON.parse(r.body).id !== undefined,
        });

        errorRate.add(!ok);
        apiDuration.add(res.timings.duration, { endpoint: 'create_order' });
    }

    sleep(Math.random() * 2); // Think time: 0-2 seconds between requests
}

export function teardown(data) {
    // Cleanup test data
    http.del(`${BASE_URL}/api/test-cleanup/${data.testOrderId}`, null, {
        headers: { Authorization: AUTH_TOKEN }
    });
}
```

---

## Gatling Load Test (Scala DSL)

```scala
// src/gatling/scala/com/example/simulations/OrderSimulation.scala
package com.example.simulations

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderSimulation extends Simulation {

  val httpConf = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .header("Authorization", "Bearer test-token")

  // Feeder for test data
  val productFeeder = csv("data/products.csv").circular

  val browseProducts = scenario("Browse Products")
    .exec(
      http("Get products page 1")
        .get("/api/products?page=0&size=20")
        .check(status.is(200))
        .check(jsonPath("$.page.totalElements").saveAs("totalProducts"))
    )
    .pause(1, 3)
    .feed(productFeeder)
    .exec(
      http("Get product detail")
        .get("/api/products/${productId}")
        .check(status.is(200))
        .check(jsonPath("$.name").exists)
    )

  val placeOrder = scenario("Place Order")
    .feed(productFeeder)
    .exec(
      http("Create order")
        .post("/api/orders")
        .body(StringBody(
          """{"customerId": "perf-test-${productId}", "items": [{"productId": ${productId}, "quantity": 1}]}"""
        ))
        .check(status.is(201))
        .check(jsonPath("$.id").saveAs("orderId"))
    )
    .pause(500.milliseconds)
    .exec(
      http("Get order status")
        .get("/api/orders/${orderId}")
        .check(status.is(200))
        .check(jsonPath("$.status").in("PENDING", "CONFIRMED"))
    )

  setUp(
    browseProducts.inject(
      nothingFor(5.seconds),
      atOnceUsers(10),
      rampUsers(50) during (2.minutes),
      constantUsersPerSec(20) during (3.minutes)
    ),
    placeOrder.inject(
      nothingFor(10.seconds),
      rampUsers(20) during (2.minutes),
      constantUsersPerSec(5) during (3.minutes)
    )
  ).protocols(httpConf)
   .assertions(
     global.responseTime.percentile3.lt(1000),  // 99th percentile < 1000ms
     global.responseTime.percentile2.lt(500),   // 95th percentile < 500ms
     global.successfulRequests.percent.gt(99),   // 99%+ success rate
     forAll.responseTime.max.lt(5000)            // No request > 5s
   )
}
```

---

## Case Study: Optimizing a 5-Second Endpoint to 50ms

### The Slow Endpoint

```java
// BEFORE: GET /api/orders/summary - takes 5000ms
@GetMapping("/orders/summary")
public OrderSummaryResponse getOrderSummary(@RequestParam String customerId) {
    // PROBLEM 1: Loads ALL orders including all items (N+1)
    List<Order> orders = orderRepository.findByCustomerId(customerId);

    // PROBLEM 2: For each order, separately loads products (N+1 within N+1)
    BigDecimal totalRevenue = BigDecimal.ZERO;
    Map<String, Integer> productCounts = new HashMap<>();

    for (Order order : orders) {
        for (OrderItem item : order.getItems()) {
            // PROBLEM 3: Lazy load triggers here - separate SQL per item
            Product product = productRepository.findById(item.getProductId()).orElseThrow();

            // PROBLEM 4: Redundant computation in loop
            totalRevenue = totalRevenue.add(
                item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity()))
            );
            productCounts.merge(product.getName(), item.getQuantity(), Integer::sum);
        }
    }

    // PROBLEM 5: Another query to count orders
    long orderCount = orderRepository.countByCustomerId(customerId);

    return new OrderSummaryResponse(orderCount, totalRevenue, productCounts);
}
```

### Investigation

```bash
# 1. Add timing logs around the method
# 2. Enable Hibernate SQL logging temporarily
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.type.descriptor.sql=TRACE

# Observation from logs:
# - 1 query: SELECT * FROM orders WHERE customer_id = ?  → returns 50 orders
# - 50 queries: SELECT * FROM order_items WHERE order_id = ?
# - 500 queries: SELECT * FROM products WHERE id = ?
# - 1 query: SELECT count(*) FROM orders WHERE customer_id = ?
# Total: 552 SQL queries!
```

### Fix 1: Eliminate N+1 with Single SQL

```java
// src/main/java/com/example/repository/OrderSummaryRepository.java
@Repository
public interface OrderSummaryRepository extends JpaRepository<Order, Long> {

    @Query(value = """
        SELECT
            COUNT(DISTINCT o.id) as orderCount,
            SUM(oi.quantity * oi.unit_price) as totalRevenue,
            p.name as productName,
            SUM(oi.quantity) as productQuantity
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.id
        JOIN products p ON p.id = oi.product_id
        WHERE o.customer_id = :customerId
          AND o.status NOT IN ('CANCELLED')
        GROUP BY p.name
        ORDER BY productQuantity DESC
        """, nativeQuery = true)
    List<ProductCountResult> getSummaryForCustomer(@Param("customerId") String customerId);

    interface ProductCountResult {
        Long getOrderCount();
        Double getTotalRevenue();
        String getProductName();
        Integer getProductQuantity();
    }
}
```

### Fix 2: Add Index

```sql
-- Migration: V7__add_order_summary_index.sql
-- The query filters on customer_id and status - create a composite index
CREATE INDEX idx_orders_customer_status
    ON orders (customer_id, status)
    WHERE status NOT IN ('CANCELLED');

-- Partial index for active orders only (smaller, faster)
CREATE INDEX idx_order_items_order_id
    ON order_items (order_id)
    INCLUDE (product_id, quantity, unit_price);
```

### Fix 3: Cache the Result

```java
// src/main/java/com/example/service/OrderSummaryService.java
@Service
@RequiredArgsConstructor
public class OrderSummaryService {

    private final OrderSummaryRepository repository;

    @Cacheable(value = "orderSummary", key = "#customerId",
               condition = "#customerId != null",
               unless = "#result == null")
    public OrderSummaryResponse getOrderSummary(String customerId) {
        List<OrderSummaryRepository.ProductCountResult> rows =
            repository.getSummaryForCustomer(customerId);

        if (rows.isEmpty()) {
            return new OrderSummaryResponse(0L, BigDecimal.ZERO, Map.of());
        }

        long orderCount = rows.get(0).getOrderCount();
        BigDecimal totalRevenue = BigDecimal.valueOf(rows.get(0).getTotalRevenue());

        Map<String, Integer> productCounts = rows.stream()
            .collect(Collectors.toMap(
                OrderSummaryRepository.ProductCountResult::getProductName,
                OrderSummaryRepository.ProductCountResult::getProductQuantity
            ));

        return new OrderSummaryResponse(orderCount, totalRevenue, productCounts);
    }

    @CacheEvict(value = "orderSummary", key = "#customerId")
    public void invalidateSummaryCache(String customerId) {
        // Called when an order is created or updated
    }
}
```

### Fix 4: Async Computation for Non-Critical Data

```java
@RestController
@RequiredArgsConstructor
public class OptimizedOrderController {

    private final OrderSummaryService summaryService;
    private final RecommendationService recommendations;

    @GetMapping("/orders/summary")
    public DeferredResult<OrderDashboard> getOrderDashboard(
            @RequestParam String customerId) {

        DeferredResult<OrderDashboard> result = new DeferredResult<>(5000L);

        // Core summary - fast (single query + cache)
        CompletableFuture<OrderSummaryResponse> summary =
            CompletableFuture.supplyAsync(() -> summaryService.getOrderSummary(customerId));

        // Recommendations - can be slower, load in parallel
        CompletableFuture<List<String>> recs =
            CompletableFuture.supplyAsync(() -> recommendations.get(customerId))
                .exceptionally(e -> List.of());  // Don't fail the whole request

        CompletableFuture.allOf(summary, recs).whenComplete((v, ex) -> {
            if (ex != null) {
                result.setErrorResult(ex);
            } else {
                result.setResult(new OrderDashboard(
                    summary.join(), recs.join()
                ));
            }
        });

        return result;
    }
}
```

### Performance Results

```
BEFORE (552 SQL queries, no index, no cache):
  Avg: 5023ms | p95: 6800ms | p99: 8200ms
  DB time: 4900ms (97% of request)

AFTER (1 SQL query, indexed, cached):
  Cold cache: 42ms | p95: 65ms | p99: 90ms
  Warm cache: 3ms  | p95: 5ms  | p99: 8ms
  
Improvement: 100x faster (cold) / 1674x faster (warm)
```

---

## Memory Leak Detection

```java
// src/main/java/com/example/perf/HeapAnalysis.java
package com.example.perf;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.lang.management.MemoryUsage;

@Slf4j
@Component
public class HeapAnalysis {

    private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();

    @Scheduled(fixedDelay = 60_000)  // Every minute
    public void logMemoryUsage() {
        MemoryUsage heap = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeap = memoryBean.getNonHeapMemoryUsage();

        double heapUsedPct = (double) heap.getUsed() / heap.getMax() * 100;

        log.info("Heap: {}MB / {}MB ({}%) | NonHeap: {}MB",
            heap.getUsed() / 1_048_576,
            heap.getMax() / 1_048_576,
            String.format("%.1f", heapUsedPct),
            nonHeap.getUsed() / 1_048_576);

        if (heapUsedPct > 85) {
            log.warn("HIGH HEAP USAGE: {}%", String.format("%.1f", heapUsedPct));
        }
    }
}
```

```bash
# Trigger heap dump for offline analysis
jmap -dump:format=b,live,file=/tmp/heap.hprof $(pgrep -f "java.*Application")

# Or via JMX in production (safer - doesn't pause the JVM as long)
jcmd $(pgrep -f "java.*Application") GC.heap_dump /tmp/heap.hprof

# Analyze with Eclipse MAT
# 1. Open heap.hprof
# 2. Run "Leak Suspects Report"
# 3. Look for objects with high "Retained Heap"
# 4. Use "OQL" to query for specific classes:
#    SELECT * FROM java.util.HashMap WHERE size > 10000
```

---

## Performance Regression Test in CI

```java
// src/test/java/com/example/perf/PerformanceRegressionTest.java
package com.example.perf;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.LongSummaryStatistics;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class PerformanceRegressionTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void orderSummaryEndpointShouldBeUnder200ms() throws Exception {
        String url = "http://localhost:" + port + "/api/orders/summary?customerId=perf-test-1";

        // Warm up
        for (int i = 0; i < 5; i++) {
            restTemplate.getForEntity(url, String.class);
        }

        // Measure 20 calls
        List<Long> durations = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            Instant start = Instant.now();
            restTemplate.getForEntity(url, String.class);
            durations.add(Duration.between(start, Instant.now()).toMillis());
        }

        LongSummaryStatistics stats = durations.stream()
            .mapToLong(Long::longValue)
            .summaryStatistics();

        System.out.printf("Order Summary Performance: avg=%.0fms p95=%.0fms max=%dms%n",
            stats.getAverage(),
            percentile(durations, 95),
            stats.getMax());

        assertThat(stats.getAverage())
            .as("Average response time should be under 200ms")
            .isLessThan(200);

        assertThat(percentile(durations, 95))
            .as("95th percentile should be under 500ms")
            .isLessThan(500);
    }

    private double percentile(List<Long> durations, int p) {
        List<Long> sorted = durations.stream().sorted().toList();
        int index = (int) Math.ceil(p / 100.0 * sorted.size()) - 1;
        return sorted.get(Math.max(0, index));
    }
}
```

---

## Summary

| Tool | Purpose | When to Use |
|---|---|---|
| JFR (Java Flight Recorder) | CPU, memory, I/O profiling | First step in performance investigation |
| async-profiler | Flame graphs, allocation profiling | Finding CPU hotspots, GC pressure |
| Thread dump | Find blocked/deadlocked threads | Throughput problems, high response times |
| Eclipse MAT | Heap dump analysis | Memory leaks, OOM errors |
| `jstat -gcutil` | GC statistics | Detecting GC overhead |
| JMH | Micro-benchmarks | Comparing algorithms/implementations |
| k6 | HTTP load testing | API endpoint performance at scale |
| Gatling | Complex load scenarios | Multi-scenario simulations |
| `EXPLAIN ANALYZE` | SQL query plan | Database query performance |
| Micrometer + Prometheus | Production metrics | Ongoing performance monitoring |

### Key Takeaways
- The #1 cause of slow Java web endpoints is N+1 SQL queries
- Always profile before optimizing — intuition is often wrong
- Caching a slow result is cheap insurance against a slow query
- Indexes on foreign keys and filter columns have the biggest impact
- GC pauses above 200ms are noticeable to users; target < 50ms
- Parallel streams are NOT always faster — test with JMH for each use case

---

## Next Part Preview

**Part 100: Capstone - Enterprise E-Commerce Platform** brings together every concept from this course into a complete reference architecture: microservices, hexagonal architecture, security, observability, deployment, and a team roadmap.
