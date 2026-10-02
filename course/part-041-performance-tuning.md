# Part 041: Performance Tuning for Spring Boot

## Overview

Performance tuning is not a one-time task — it's a continuous practice of measuring, identifying bottlenecks, applying fixes, and measuring again. This part covers the full stack of Spring Boot performance optimization: from JVM profiling through database tuning to async processing.

---

## 1. JVM Profiling Tools

### 1.1 Java Flight Recorder (JFR)

JFR is a low-overhead profiling and event collection framework built into the JVM.

**Enable JFR in Spring Boot application:**

```java
// application.properties
# Enable JFR via JVM args in startup script
# java -XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

// Or programmatically:
package com.example.performance;

import jdk.jfr.Configuration;
import jdk.jfr.Recording;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.file.Path;
import java.time.Duration;

@Component
@Profile("profiling")
public class JfrRecordingRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        Configuration config = Configuration.getConfiguration("profile");
        try (Recording recording = new Recording(config)) {
            recording.setMaxSize(100 * 1024 * 1024); // 100 MB
            recording.setMaxAge(Duration.ofMinutes(5));
            recording.setToDisk(true);
            recording.setDestination(Path.of("app-profile.jfr"));
            recording.start();

            // Application runs here...
            System.out.println("JFR Recording started. Press Ctrl+C to stop.");
            Thread.currentThread().join();
        }
    }
}
```

**JFR event for custom application metrics:**

```java
package com.example.performance.jfr;

import jdk.jfr.Category;
import jdk.jfr.Description;
import jdk.jfr.Event;
import jdk.jfr.Label;
import jdk.jfr.Name;
import jdk.jfr.StackTrace;

@Name("com.example.DatabaseQuery")
@Label("Database Query")
@Category({"Application", "Database"})
@Description("Records database query execution details")
@StackTrace(false)
public class DatabaseQueryEvent extends Event {

    @Label("SQL Query")
    public String query;

    @Label("Execution Time (ms)")
    public long executionTimeMs;

    @Label("Row Count")
    public int rowCount;

    @Label("Table Name")
    public String tableName;
}
```

**Using the custom JFR event:**

```java
package com.example.performance;

import com.example.performance.jfr.DatabaseQueryEvent;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class QueryProfilingAspect {

    @Around("@annotation(com.example.performance.ProfileQuery)")
    public Object profileQuery(ProceedingJoinPoint joinPoint) throws Throwable {
        DatabaseQueryEvent event = new DatabaseQueryEvent();
        event.begin();

        long startTime = System.currentTimeMillis();
        Object result = null;
        try {
            result = joinPoint.proceed();
            return result;
        } finally {
            event.executionTimeMs = System.currentTimeMillis() - startTime;
            event.query = joinPoint.getSignature().getName();
            event.commit();
        }
    }
}
```

### 1.2 Async-profiler Integration

```java
// build.gradle
dependencies {
    implementation 'tools.profiler:async-profiler:2.9'
    // Or use the agent via JVM args:
    // -agentpath:/path/to/libasyncProfiler.so=start,event=cpu,file=profile.html
}
```

```java
package com.example.performance;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;

@RestController
@RequestMapping("/admin/profiler")
public class ProfilerController {

    @GetMapping("/start")
    public String startProfiling() throws IOException, InterruptedException {
        // Trigger async-profiler via shell command (requires async-profiler installed)
        ProcessBuilder pb = new ProcessBuilder(
            "java",
            "-jar", "async-profiler.jar",
            "-d", "60",
            "-f", "/tmp/flamegraph.html",
            String.valueOf(ProcessHandle.current().pid())
        );
        pb.start();
        return "Profiling started for 60 seconds";
    }
}
```

### 1.3 VisualVM / JConsole via JMX

```yaml
# application.yml
spring:
  jmx:
    enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,heapdump,threaddump
  endpoint:
    heapdump:
      enabled: true
    threaddump:
      enabled: true
```

```java
// JVM args for remote JMX profiling
// -Dcom.sun.management.jmxremote
// -Dcom.sun.management.jmxremote.port=9010
// -Dcom.sun.management.jmxremote.authenticate=false
// -Dcom.sun.management.jmxremote.ssl=false
```

---

## 2. Connection Pool Tuning (HikariCP)

HikariCP is the default connection pool in Spring Boot. Proper tuning is critical for performance under load.

### 2.1 HikariCP Configuration

```java
package com.example.performance.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {

    @Value("${spring.datasource.url}")
    private String jdbcUrl;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();

        // Core settings
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setDriverClassName("org.postgresql.Driver");

        // Pool sizing
        // Formula: connections = ((core_count * 2) + effective_spindle_count)
        // For 4 CPU cores: 4 * 2 + 1 = 9, rounded to 10
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(5);

        // Timeouts
        config.setConnectionTimeout(30_000);  // 30s - max wait for connection
        config.setIdleTimeout(600_000);       // 10min - idle connection lifetime
        config.setMaxLifetime(1_800_000);     // 30min - max connection lifetime
        config.setKeepaliveTime(60_000);      // 1min - keepalive ping interval

        // Performance settings
        config.setAutoCommit(false);
        config.setTransactionIsolation("TRANSACTION_READ_COMMITTED");

        // Connection validation
        config.setConnectionTestQuery("SELECT 1");
        config.setValidationTimeout(5_000);

        // Pool name for monitoring
        config.setPoolName("MainConnectionPool");

        // Register to JMX for monitoring
        config.setRegisterMbeans(true);

        // PostgreSQL-specific optimizations
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");

        return new HikariDataSource(config);
    }
}
```

### 2.2 Pool Metrics Endpoint

```java
package com.example.performance.metrics;

import com.zaxxer.hikari.HikariDataSource;
import com.zaxxer.hikari.HikariPoolMXBean;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

@Component
@Endpoint(id = "hikari")
public class HikariMetricsEndpoint {

    private final HikariPoolMXBean hikariPool;

    @Autowired
    public HikariMetricsEndpoint(DataSource dataSource) {
        this.hikariPool = ((HikariDataSource) dataSource).getHikariPoolMXBean();
    }

    @ReadOperation
    public Map<String, Object> hikariStats() {
        Map<String, Object> stats = new HashMap<>();
        stats.put("activeConnections", hikariPool.getActiveConnections());
        stats.put("idleConnections", hikariPool.getIdleConnections());
        stats.put("totalConnections", hikariPool.getTotalConnections());
        stats.put("threadsAwaitingConnection", hikariPool.getThreadsAwaitingConnection());
        return stats;
    }
}
```

---

## 3. JPA/Hibernate N+1 Problem Detection and Fix

The N+1 problem is one of the most common performance killers in JPA applications.

### 3.1 Reproducing the N+1 Problem

```java
package com.example.performance.entity;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerName;
    private String status;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();

    // getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }
}
```

```java
package com.example.performance.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
public class OrderItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    private String productName;
    private int quantity;
    private BigDecimal price;

    // getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
    public String getProductName() { return productName; }
    public void setProductName(String productName) { this.productName = productName; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
}
```

### 3.2 N+1 Bad Example (Don't do this)

```java
// BAD: This triggers N+1 queries
// 1 query for all orders + N queries (one per order) for items
@Service
public class OrderServiceBad {

    @Autowired
    private OrderRepository orderRepository;

    public List<String> getAllOrderSummaries() {
        List<Order> orders = orderRepository.findAll(); // 1 query

        return orders.stream()
            .map(order -> {
                // This triggers a new query for each order's items!
                int itemCount = order.getItems().size(); // N queries
                return order.getCustomerName() + " - " + itemCount + " items";
            })
            .toList();
    }
}
```

### 3.3 Fix 1: JOIN FETCH

```java
package com.example.performance.repository;

import com.example.performance.entity.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // GOOD: Single query with JOIN FETCH
    @Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items WHERE o.status = :status")
    List<Order> findByStatusWithItems(String status);

    // GOOD: JPQL with JOIN FETCH for all orders
    @Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items")
    List<Order> findAllWithItems();

    // Native SQL alternative
    @Query(
        value = """
            SELECT DISTINCT o.* FROM orders o
            LEFT JOIN order_items oi ON oi.order_id = o.id
            WHERE o.status = :status
            """,
        nativeQuery = true
    )
    List<Order> findByStatusNative(String status);
}
```

### 3.4 Fix 2: Entity Graph

```java
package com.example.performance.repository;

import com.example.performance.entity.Order;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface OrderWithGraphRepository extends JpaRepository<Order, Long> {

    // Eager load items using EntityGraph
    @EntityGraph(attributePaths = {"items"})
    List<Order> findByStatus(String status);

    @EntityGraph(attributePaths = {"items"})
    Optional<Order> findById(Long id);
}
```

**Named Entity Graph on entity:**

```java
@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "Order.withItems",
    attributeNodes = {
        @NamedAttributeNode("items")
    }
)
public class Order {
    // ... same as above
}
```

```java
@EntityGraph("Order.withItems")
List<Order> findAll();
```

### 3.5 Fix 3: Batch Loading

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 25
        # This tells Hibernate to load items in batches of 25
        # Instead of N queries, you get N/25 queries
```

```java
@Entity
@Table(name = "orders")
public class Order {

    // ... fields ...

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    @BatchSize(size = 25)  // Hibernate-specific annotation
    private List<OrderItem> items = new ArrayList<>();
}
```

### 3.6 Detect N+1 with Hibernate Statistics

```yaml
# application.yml for development
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 25
logging:
  level:
    org.hibernate.stat: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

```java
package com.example.performance.service;

import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import jakarta.persistence.EntityManagerFactory;

@Service
public class HibernateStatisticsService {

    private final Statistics statistics;

    @Autowired
    public HibernateStatisticsService(EntityManagerFactory emf) {
        SessionFactory sessionFactory = emf.unwrap(SessionFactory.class);
        this.statistics = sessionFactory.getStatistics();
        this.statistics.setStatisticsEnabled(true);
    }

    public void printStats() {
        System.out.println("=== Hibernate Statistics ===");
        System.out.println("Query count: " + statistics.getQueryExecutionCount());
        System.out.println("Entity load count: " + statistics.getEntityLoadCount());
        System.out.println("Collection load count: " + statistics.getCollectionLoadCount());
        System.out.println("Second level cache hit: " + statistics.getSecondLevelCacheHitCount());
        System.out.println("Second level cache miss: " + statistics.getSecondLevelCacheMissCount());
        System.out.printf("Slowest query: %s ms%n", statistics.getQueryExecutionMaxTime());
        System.out.println("Slowest query SQL: " + statistics.getQueryExecutionMaxTimeQueryString());
    }

    public void resetStats() {
        statistics.clear();
    }
}
```

---

## 4. Query Optimization (Indexes, EXPLAIN ANALYZE)

### 4.1 Strategic Index Creation

```sql
-- Single column indexes
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_customer_name ON orders(customer_name);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Composite index (order matters! Put most selective column first)
CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);

-- Partial index (for frequently queried subsets)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'PENDING';

-- Index on expression
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM created_at));

-- Covering index (includes columns to avoid table lookup)
CREATE INDEX idx_orders_covering ON orders(status, created_at)
INCLUDE (customer_name, total_amount);
```

### 4.2 EXPLAIN ANALYZE Examples

```sql
-- Check if index is being used
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'PENDING'
  AND created_at > NOW() - INTERVAL '7 days';

-- Example output analysis:
-- Index Scan using idx_orders_status_created on orders (cost=0.43..45.23 rows=100 width=150)
--   Index Cond: ((status = 'PENDING') AND (created_at > (now() - '7 days'::interval)))
-- Planning Time: 0.5 ms
-- Execution Time: 1.2 ms

-- Without index:
-- Seq Scan on orders (cost=0.00..25000.00 rows=100000 width=150)
-- Planning Time: 0.1 ms
-- Execution Time: 850.0 ms  <-- Much slower!
```

### 4.3 Spring Integration for Query Analysis

```java
package com.example.performance.service;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class QueryAnalysisService {

    @PersistenceContext
    private EntityManager em;

    @Transactional(readOnly = true)
    public List<Object[]> explainAnalyze(String sql) {
        return em.createNativeQuery("EXPLAIN ANALYZE " + sql)
                 .getResultList();
    }

    @Transactional(readOnly = true)
    public void printQueryPlan(String sql) {
        List<Object[]> plan = explainAnalyze(sql);
        System.out.println("=== Query Execution Plan ===");
        plan.forEach(row -> System.out.println(row[0]));
    }
}
```

---

## 5. Spring Cache Strategy for Performance

### 5.1 Cache Configuration

```java
package com.example.performance.config;

import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();

        // Default cache settings
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats()
        );

        return cacheManager;
    }

    // Custom cache for different TTLs
    @Bean
    public CacheManager multiCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager() {
            @Override
            protected com.github.benmanes.caffeine.cache.Cache<Object, Object>
                    createNativeCaffeineCache(String name) {
                return switch (name) {
                    case "products" -> Caffeine.newBuilder()
                        .maximumSize(500)
                        .expireAfterWrite(30, TimeUnit.MINUTES)
                        .recordStats()
                        .build();
                    case "users" -> Caffeine.newBuilder()
                        .maximumSize(1000)
                        .expireAfterWrite(5, TimeUnit.MINUTES)
                        .recordStats()
                        .build();
                    case "configs" -> Caffeine.newBuilder()
                        .maximumSize(100)
                        .expireAfterWrite(24, TimeUnit.HOURS)
                        .recordStats()
                        .build();
                    default -> Caffeine.newBuilder()
                        .maximumSize(200)
                        .expireAfterWrite(10, TimeUnit.MINUTES)
                        .recordStats()
                        .build();
                };
            }
        };
        return cacheManager;
    }
}
```

### 5.2 Service Layer Caching

```java
package com.example.performance.service;

import com.example.performance.entity.Order;
import com.example.performance.repository.OrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.Caching;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional
public class OrderCachingService {

    @Autowired
    private OrderRepository orderRepository;

    // Cache by ID - only fills cache on first call
    @Cacheable(value = "orders", key = "#id")
    @Transactional(readOnly = true)
    public Order findById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Order not found: " + id));
    }

    // Cache by status
    @Cacheable(value = "orders", key = "'status:' + #status",
               unless = "#result.size() == 0")
    @Transactional(readOnly = true)
    public List<Order> findByStatus(String status) {
        return orderRepository.findByStatus(status);
    }

    // Update cache after save
    @CachePut(value = "orders", key = "#result.id")
    public Order save(Order order) {
        return orderRepository.save(order);
    }

    // Evict specific entry on update
    @Caching(evict = {
        @CacheEvict(value = "orders", key = "#order.id"),
        @CacheEvict(value = "orders", allEntries = true) // Clear all status caches
    })
    public Order update(Order order) {
        return orderRepository.save(order);
    }

    // Evict on delete
    @CacheEvict(value = "orders", key = "#id")
    public void delete(Long id) {
        orderRepository.deleteById(id);
    }
}
```

### 5.3 Redis Cache for Distributed Environments

```yaml
# application.yml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes in ms
      cache-null-values: false
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
```

```java
package com.example.performance.config;

import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableCaching
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer())
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())
            )
            .disableCachingNullValues();

        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        cacheConfigs.put("products", defaultConfig.entryTtl(Duration.ofMinutes(30)));
        cacheConfigs.put("users", defaultConfig.entryTtl(Duration.ofMinutes(5)));
        cacheConfigs.put("configs", defaultConfig.entryTtl(Duration.ofHours(24)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

---

## 6. HTTP/2 and Compression in Spring Boot

### 6.1 Enable HTTP/2

```yaml
# application.yml
server:
  http2:
    enabled: true
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: changeit
    key-store-type: PKCS12
    key-alias: myapp
  port: 8443

  # For development without SSL (HTTP/2 over cleartext - h2c)
  # Note: h2c requires Undertow, not Tomcat
```

```java
package com.example.performance.config;

import org.apache.catalina.connector.Connector;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ServerConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tomcatCustomizer() {
        return factory -> {
            factory.addConnectorCustomizers(connector -> {
                connector.setProperty("maxThreads", "200");
                connector.setProperty("minSpareThreads", "10");
                connector.setProperty("connectionTimeout", "20000");
                connector.setProperty("keepAliveTimeout", "20000");
                connector.setProperty("maxKeepAliveRequests", "100");
            });
        };
    }
}
```

### 6.2 Response Compression

```yaml
# application.yml
server:
  compression:
    enabled: true
    mime-types:
      - application/json
      - application/xml
      - text/html
      - text/xml
      - text/plain
      - text/css
      - application/javascript
    min-response-size: 1024  # Only compress responses > 1KB
```

---

## 7. Async Processing with @Async and CompletableFuture

### 7.1 Async Configuration

```java
package com.example.performance.config;

import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurer;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.Arrays;
import java.util.concurrent.Executor;
import java.lang.reflect.Method;

@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    @Bean(name = "taskExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (Throwable ex, Method method, Object... params) -> {
            System.err.printf("Async method %s threw: %s%nParams: %s%n",
                method.getName(), ex.getMessage(), Arrays.toString(params));
        };
    }

    // Separate executor for I/O-heavy tasks
    @Bean(name = "ioExecutor")
    public Executor ioExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("io-async-");
        executor.initialize();
        return executor;
    }
}
```

### 7.2 Async Service Methods

```java
package com.example.performance.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CompletableFuture;

@Service
public class AsyncNotificationService {

    private static final Logger log = LoggerFactory.getLogger(AsyncNotificationService.class);

    @Autowired(required = false)
    private JavaMailSender mailSender;

    // Fire-and-forget async task
    @Async("taskExecutor")
    public void sendEmailAsync(String to, String subject, String body) {
        log.info("Sending email async to: {} on thread: {}", to, Thread.currentThread().getName());
        try {
            if (mailSender != null) {
                SimpleMailMessage message = new SimpleMailMessage();
                message.setTo(to);
                message.setSubject(subject);
                message.setText(body);
                mailSender.send(message);
            }
            Thread.sleep(100); // Simulate email sending
            log.info("Email sent to: {}", to);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    // Async with return value
    @Async("ioExecutor")
    public CompletableFuture<String> fetchExternalDataAsync(String url) {
        log.info("Fetching from: {} on thread: {}", url, Thread.currentThread().getName());
        try {
            Thread.sleep(200); // Simulate HTTP call
            return CompletableFuture.completedFuture("Data from " + url);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

### 7.3 CompletableFuture Composition

```java
package com.example.performance.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

@Service
public class OrderProcessingService {

    @Autowired
    private AsyncNotificationService notificationService;

    public record OrderResult(String inventoryCheck, String paymentResult, String shippingLabel) {}

    public OrderResult processOrderParallel(Long orderId) throws ExecutionException, InterruptedException {
        // Execute three operations in parallel
        CompletableFuture<String> inventoryFuture =
            notificationService.fetchExternalDataAsync("inventory-service/check/" + orderId);

        CompletableFuture<String> paymentFuture =
            notificationService.fetchExternalDataAsync("payment-service/charge/" + orderId);

        CompletableFuture<String> shippingFuture =
            notificationService.fetchExternalDataAsync("shipping-service/label/" + orderId);

        // Wait for all to complete
        CompletableFuture<Void> allOf = CompletableFuture.allOf(
            inventoryFuture, paymentFuture, shippingFuture
        );

        allOf.join(); // Block until all complete

        return new OrderResult(
            inventoryFuture.get(),
            paymentFuture.get(),
            shippingFuture.get()
        );
    }

    public CompletableFuture<List<String>> fetchMultipleUrls(List<String> urls) {
        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> notificationService.fetchExternalDataAsync(url))
            .toList();

        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .toList()
            );
    }
}
```

---

## 8. Virtual Threads (Java 21) for I/O-Bound Work

### 8.1 Enabling Virtual Threads in Spring Boot

```yaml
# application.yml (Spring Boot 3.2+)
spring:
  threads:
    virtual:
      enabled: true
```

```java
package com.example.performance.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

@Configuration
public class VirtualThreadConfig {

    // Spring Boot 3.2+ automatically uses virtual threads when enabled
    // For explicit configuration:

    @Bean(name = "virtualThreadExecutor")
    public Executor virtualThreadExecutor() {
        // Unlimited virtual threads - each task gets its own virtual thread
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

### 8.2 Virtual Threads for Async Operations

```java
package com.example.performance.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@Service
public class VirtualThreadService {

    private static final Logger log = LoggerFactory.getLogger(VirtualThreadService.class);

    // With virtual threads, you can have thousands of concurrent I/O operations
    public List<String> fetchAllConcurrently(List<String> urls) throws InterruptedException {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<CompletableFuture<String>> futures = urls.stream()
                .map(url -> CompletableFuture.supplyAsync(
                    () -> simulateFetch(url), executor
                ))
                .toList();

            return futures.stream()
                .map(CompletableFuture::join)
                .toList();
        }
    }

    private String simulateFetch(String url) {
        log.info("Fetching {} on thread: {} (virtual: {})",
            url, Thread.currentThread().getName(), Thread.currentThread().isVirtual());
        try {
            Thread.sleep(100); // Simulate I/O - virtual thread yields during sleep
            return "Result from: " + url;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    // Benchmark: Virtual threads vs platform threads for I/O
    public void benchmark() throws Exception {
        int taskCount = 10_000;
        List<String> urls = List.of("url1", "url2", "url3").stream()
            .flatMap(url -> List.of(url).stream().limit(taskCount / 3))
            .toList();

        // Platform threads
        long platformStart = System.currentTimeMillis();
        try (ExecutorService platformPool = Executors.newFixedThreadPool(200)) {
            List<CompletableFuture<String>> futures = urls.stream()
                .map(url -> CompletableFuture.supplyAsync(() -> simulateFetch(url), platformPool))
                .toList();
            futures.forEach(CompletableFuture::join);
        }
        long platformTime = System.currentTimeMillis() - platformStart;

        // Virtual threads
        long virtualStart = System.currentTimeMillis();
        try (ExecutorService virtualPool = Executors.newVirtualThreadPerTaskExecutor()) {
            List<CompletableFuture<String>> futures = urls.stream()
                .map(url -> CompletableFuture.supplyAsync(() -> simulateFetch(url), virtualPool))
                .toList();
            futures.forEach(CompletableFuture::join);
        }
        long virtualTime = System.currentTimeMillis() - virtualStart;

        System.out.printf("Platform threads (200 pool, %d tasks): %d ms%n", taskCount, platformTime);
        System.out.printf("Virtual threads (%d tasks): %d ms%n", taskCount, virtualTime);
    }
}
```

---

## 9. Pagination Best Practices

### 9.1 Offset vs. Keyset Pagination

```java
package com.example.performance.repository;

import com.example.performance.entity.Order;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Slice;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.Instant;
import java.util.List;

@Repository
public interface PaginationOrderRepository extends JpaRepository<Order, Long> {

    // Offset pagination (simple but slow for large offsets)
    Page<Order> findByStatus(String status, Pageable pageable);

    // Slice (more efficient: no count query)
    Slice<Order> findByStatusOrderByCreatedAtDesc(String status, Pageable pageable);

    // Keyset pagination (constant time regardless of page size)
    // Uses the last seen ID/cursor instead of OFFSET
    @Query("""
        SELECT o FROM Order o
        WHERE o.status = :status
          AND (o.createdAt < :lastSeenCreatedAt
               OR (o.createdAt = :lastSeenCreatedAt AND o.id < :lastSeenId))
        ORDER BY o.createdAt DESC, o.id DESC
        """)
    List<Order> findByStatusKeysetPagination(
        @Param("status") String status,
        @Param("lastSeenCreatedAt") Instant lastSeenCreatedAt,
        @Param("lastSeenId") Long lastSeenId,
        Pageable pageable
    );

    // Count query optimization (use approximate count for large tables)
    @Query(value = "SELECT reltuples::bigint FROM pg_class WHERE relname = 'orders'",
           nativeQuery = true)
    Long approximateCount();
}
```

### 9.2 Pagination Controller

```java
package com.example.performance.controller;

import com.example.performance.entity.Order;
import com.example.performance.repository.PaginationOrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Slice;
import org.springframework.data.domain.Sort;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/orders")
public class OrderPaginationController {

    @Autowired
    private PaginationOrderRepository orderRepository;

    // Offset pagination
    @GetMapping
    public ResponseEntity<?> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "PENDING") String status
    ) {
        PageRequest pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        Slice<Order> result = orderRepository.findByStatusOrderByCreatedAtDesc(status, pageable);

        return ResponseEntity.ok(Map.of(
            "orders", result.getContent(),
            "hasNext", result.hasNext(),
            "hasPrevious", result.hasPrevious(),
            "pageNumber", result.getNumber()
        ));
    }

    // Keyset pagination (cursor-based, recommended for large datasets)
    @GetMapping("/cursor")
    public ResponseEntity<?> getOrdersCursor(
        @RequestParam(defaultValue = "PENDING") String status,
        @RequestParam(required = false) Instant lastCreatedAt,
        @RequestParam(required = false) Long lastId,
        @RequestParam(defaultValue = "20") int size
    ) {
        List<Order> orders;

        if (lastCreatedAt == null || lastId == null) {
            // First page
            orders = orderRepository.findByStatusKeysetPagination(
                status, Instant.now(), Long.MAX_VALUE,
                PageRequest.of(0, size)
            );
        } else {
            orders = orderRepository.findByStatusKeysetPagination(
                status, lastCreatedAt, lastId,
                PageRequest.of(0, size)
            );
        }

        Order lastOrder = orders.isEmpty() ? null : orders.getLast();
        return ResponseEntity.ok(Map.of(
            "orders", orders,
            "nextCursor", lastOrder != null ? Map.of(
                "createdAt", lastOrder.getCreatedAt(),
                "id", lastOrder.getId()
            ) : null,
            "hasMore", orders.size() == size
        ));
    }
}
```

---

## 10. Slow Query Detection with Hibernate Statistics

### 10.1 Slow Query Log Configuration

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        format_sql: true
        use_sql_comments: true
        session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE
    org.hibernate.stat: INFO
    org.hibernate.engine.internal.StatisticalLoggingSessionEventListener: INFO
```

### 10.2 Custom Slow Query Interceptor

```java
package com.example.performance.hibernate;

import org.hibernate.resource.jdbc.spi.StatementInspector;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class SlowQueryInspector implements StatementInspector {

    private static final Logger log = LoggerFactory.getLogger(SlowQueryInspector.class);
    private static final long SLOW_QUERY_THRESHOLD_MS = 100;

    private static final ConcurrentHashMap<String, AtomicLong> queryCounts = new ConcurrentHashMap<>();
    private static final ConcurrentHashMap<String, AtomicLong> queryTimes = new ConcurrentHashMap<>();

    @Override
    public String inspect(String sql) {
        // Track query execution (in a real scenario, measure actual time)
        queryCounts.computeIfAbsent(sql, k -> new AtomicLong()).incrementAndGet();
        return sql;
    }

    public static void reportTopQueries(int topN) {
        log.info("=== Top {} Most Executed Queries ===", topN);
        queryCounts.entrySet().stream()
            .sorted((a, b) -> Long.compare(b.getValue().get(), a.getValue().get()))
            .limit(topN)
            .forEach(e -> log.info("Count: {} | SQL: {}",
                e.getValue().get(),
                e.getKey().substring(0, Math.min(100, e.getKey().length()))
            ));
    }
}
```

```java
package com.example.performance.config;

import com.example.performance.hibernate.SlowQueryInspector;
import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;

@Configuration
public class HibernateConfig {

    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return hibernateProperties -> {
            hibernateProperties.put(
                "hibernate.session_factory.statement_inspector",
                new SlowQueryInspector()
            );
        };
    }
}
```

---

## 11. Real Example: Before/After Performance Comparison

### 11.1 The Problem Scenario

```java
// BEFORE: Poorly optimized e-commerce order listing
// Benchmark: 10,000 orders, each with 5-15 items
// Result: ~3500ms for loading 100 orders

package com.example.performance.before;

import com.example.performance.entity.Order;
import com.example.performance.entity.OrderItem;
import com.example.performance.repository.OrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class OrderServiceBefore {

    @Autowired
    private OrderRepository orderRepository;

    // BEFORE: Triggers N+1, no cache, blocking I/O
    @Transactional(readOnly = true)
    public List<OrderSummaryDto> getOrderSummaries(String status) {
        // 1 query for orders
        List<Order> orders = orderRepository.findByStatus(status);

        return orders.stream()
            .map(order -> {
                // N queries for items (N+1 problem)
                List<OrderItem> items = order.getItems();

                BigDecimal total = items.stream()
                    .map(item -> item.getPrice()
                        .multiply(BigDecimal.valueOf(item.getQuantity())))
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

                return new OrderSummaryDto(
                    order.getId(),
                    order.getCustomerName(),
                    items.size(),
                    total
                );
            })
            .collect(Collectors.toList());
    }

    public record OrderSummaryDto(Long id, String customer, int itemCount, BigDecimal total) {}
}
```

### 11.2 The Optimized Version

```java
package com.example.performance.after;

import com.example.performance.repository.OrderRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Slice;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.math.BigDecimal;
import java.util.List;
import java.util.concurrent.CompletableFuture;

@Repository
interface OptimizedOrderRepository extends JpaRepository<com.example.performance.entity.Order, Long> {

    // Fix 1: JOIN FETCH eliminates N+1
    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.items
        WHERE o.status = :status
        ORDER BY o.createdAt DESC
        """)
    List<com.example.performance.entity.Order> findByStatusWithItems(@Param("status") String status);

    // Fix 2: Use projection to avoid loading unnecessary fields
    @Query("""
        SELECT new com.example.performance.after.OrderSummaryProjection(
            o.id, o.customerName, COUNT(i), SUM(i.price * i.quantity)
        )
        FROM Order o
        LEFT JOIN o.items i
        WHERE o.status = :status
        GROUP BY o.id, o.customerName
        """)
    List<OrderSummaryProjection> findSummaryByStatus(@Param("status") String status);
}

record OrderSummaryProjection(Long id, String customer, Long itemCount, BigDecimal total) {}

@Service
class OptimizedOrderService {

    @Autowired
    private OptimizedOrderRepository orderRepository;

    // AFTER: Single query, cached, non-blocking
    @Cacheable(value = "orderSummaries", key = "#status")
    @Transactional(readOnly = true)
    public List<OrderSummaryProjection> getOrderSummaries(String status) {
        // Single query with aggregation - returns only needed data
        return orderRepository.findSummaryByStatus(status);
    }
}
```

### 11.3 Benchmark Results

```java
package com.example.performance.benchmark;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Component
public class PerformanceBenchmark {

    @Autowired
    private com.example.performance.before.OrderServiceBefore beforeService;

    @Autowired
    private com.example.performance.after.OptimizedOrderService afterService;

    public void runBenchmark() {
        int warmupRounds = 3;
        int benchmarkRounds = 10;

        System.out.println("=== Performance Benchmark ===");
        System.out.println("Dataset: 10,000 orders, avg 10 items each");
        System.out.println();

        // Warmup
        for (int i = 0; i < warmupRounds; i++) {
            beforeService.getOrderSummaries("PENDING");
            afterService.getOrderSummaries("PENDING");
        }

        // Benchmark BEFORE
        List<Long> beforeTimes = new ArrayList<>();
        for (int i = 0; i < benchmarkRounds; i++) {
            Instant start = Instant.now();
            beforeService.getOrderSummaries("PENDING");
            beforeTimes.add(Duration.between(start, Instant.now()).toMillis());
        }
        double beforeAvg = beforeTimes.stream().mapToLong(Long::longValue).average().orElse(0);

        // Benchmark AFTER
        List<Long> afterTimes = new ArrayList<>();
        for (int i = 0; i < benchmarkRounds; i++) {
            Instant start = Instant.now();
            afterService.getOrderSummaries("PENDING");
            afterTimes.add(Duration.between(start, Instant.now()).toMillis());
        }
        double afterAvg = afterTimes.stream().mapToLong(Long::longValue).average().orElse(0);

        System.out.printf("BEFORE (N+1 queries): %.1f ms average%n", beforeAvg);
        System.out.printf("AFTER (JOIN FETCH + cache): %.1f ms average%n", afterAvg);
        System.out.printf("Improvement: %.1fx faster%n", beforeAvg / afterAvg);

        /*
         * Expected results with 10,000 orders (each with ~10 items):
         *
         * Queries executed:
         *   BEFORE: 10,001 queries (1 + N for each order's items)
         *   AFTER:  1 query (single JOIN with aggregation)
         *
         * Timing:
         *   BEFORE: ~3,500 ms (10,000 extra roundtrips at ~0.3ms each)
         *   AFTER first call: ~45 ms (single optimized query)
         *   AFTER cached: ~2 ms (in-memory cache hit)
         *
         * Improvement: ~77x faster (uncached), ~1750x faster (cached)
         *
         * Memory usage:
         *   BEFORE: ~150 MB (full entity graph loaded)
         *   AFTER:  ~8 MB (only projected data)
         */
    }
}
```

---

## Summary

| Technique | Problem Solved | Improvement |
|-----------|---------------|-------------|
| JFR / Async-profiler | CPU hotspots, memory leaks | Identify root cause |
| HikariCP tuning | Connection exhaustion, pool saturation | -50% connection wait time |
| JOIN FETCH | N+1 query problem | -99% query count |
| EntityGraph | Selective eager loading | -90% query count |
| Batch loading | Lazy collection loading | -95% query count |
| Projection queries | Loading unnecessary fields | -80% memory usage |
| Spring Cache | Repeated expensive queries | -99% DB load |
| Redis Cache | Distributed cache | Same across instances |
| HTTP/2 | Multiple request overhead | -40% latency (bundled) |
| Response compression | Network bandwidth | -70% payload size |
| @Async | Blocking main thread | -N x latency (parallel I/O) |
| Virtual Threads | Platform thread overhead | -80% memory at high concurrency |
| Keyset pagination | OFFSET slowness on large data | O(1) vs O(n) |

---

## Next Part Preview

**Part 042: Java 21 Modern Features** — We'll explore Records, Sealed Classes, Pattern Matching for switch, Virtual Threads, Structured Concurrency, and how to use all these modern Java features in Spring Boot applications.
