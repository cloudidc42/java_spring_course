# Part 090: Production Readiness Checklist

## Overview

Shipping to production is not just deploying code. This part covers every configuration decision that separates a stable production service from one that degrades under load: connection pools, thread pools, JVM sizing, graceful shutdown, health checks, secret management, logging, metrics, and a complete incident response runbook.

---

## 1. Production Application Configuration

```yaml
# application-production.yml

spring:
  application:
    name: order-service

  # ===== Database Connection Pool (HikariCP) =====
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      pool-name: OrderServicePool
      # Formula: num_cpus * 2 + num_disk_spindles (for SSD: num_cpus * 2)
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000       # 30s: fail fast if pool exhausted
      idle-timeout: 600000            # 10m: reclaim idle connections
      max-lifetime: 1800000           # 30m: recycle connections (< DB timeout)
      keepalive-time: 60000           # 1m: prevent firewall timeout drops
      validation-timeout: 5000        # 5s: health check timeout
      connection-test-query: SELECT 1 # explicit test query
      leak-detection-threshold: 60000 # 1m: warn on leaked connections

  # ===== JPA / Hibernate =====
  jpa:
    open-in-view: false          # critical: prevents connection leaks
    show-sql: false
    properties:
      hibernate:
        format_sql: false
        generate_statistics: false  # enable only for debugging
        jdbc:
          batch_size: 50
          fetch_size: 100
          order_inserts: true
          order_updates: true
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory

  # ===== Redis =====
  data:
    redis:
      url: ${REDIS_URL}
      timeout: 2000ms
      jedis:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5
          max-wait: 2000ms
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 5
          max-wait: -1ms  # lettuce: -1 = no timeout (non-blocking by default)

  # ===== Kafka (if used) =====
  kafka:
    producer:
      acks: all                    # wait for all replicas
      retries: 3
      properties:
        enable.idempotence: true   # exactly-once producer
        max.in.flight.requests.per.connection: 5
    consumer:
      auto-offset-reset: earliest
      enable-auto-commit: false    # manual commit for reliability

# ===== Server =====
server:
  port: 8080
  tomcat:
    threads:
      max: 200             # max concurrent HTTP requests
      min-spare: 20        # always-warm threads
    max-connections: 10000
    accept-count: 100      # queue when all threads busy
    connection-timeout: 20000
  compression:
    enabled: true
    min-response-size: 1024
    mime-types: application/json,application/xml,text/plain,text/html
  http2:
    enabled: true
  shutdown: graceful       # wait for active requests to complete

# ===== Graceful shutdown =====
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

# ===== Actuator =====
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
      probes:
        enabled: true        # /actuator/health/liveness, /actuator/health/readiness
    shutdown:
      enabled: false         # never expose in production
  health:
    circuitbreakers:
      enabled: true
    diskspace:
      enabled: true
      threshold: 10GB
    redis:
      enabled: true
    db:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5,0.9,0.95,0.99
      slo:
        http.server.requests: 10ms,100ms,500ms,1000ms
    tags:
      application: ${spring.application.name}
      environment: production
      region: ${APP_REGION:us-east-1}

# ===== Logging =====
logging:
  level:
    root: WARN
    com.example: INFO
    org.springframework.security: WARN
    org.hibernate.SQL: WARN
  file:
    path: /var/log/order-service
    name: ${logging.file.path}/application.log
  logback:
    rollingpolicy:
      max-file-size: 100MB
      max-history: 30
      total-size-cap: 3GB
      clean-history-on-start: false
```

---

## 2. Connection Pool Sizing

```java
// Connection pool sizing recommendations:
// HikariCP formula: pool_size = Tn * (Cm - 1) + 1
// where Tn = max concurrent threads, Cm = max time for one transaction

// For a service with max 200 threads and avg 2 DB calls per request:
// max-pool-size = 200 * (2 - 1) + 1 = 201  (too high!)
// In practice: profile your actual DB concurrency, start with 20-30

@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        return new HikariConfig();
    }

    @Bean
    public DataSource dataSource(HikariConfig config) {
        HikariDataSource ds = new HikariDataSource(config);
        // Register with Micrometer
        new HikariDataSourceMXBeanAdapter(ds, "primary").bindTo(Metrics.globalRegistry);
        return ds;
    }
}

// Read-only replica pool for read-heavy workloads
@Configuration
public class ReadReplicaConfig {

    @Bean("readOnlyDataSource")
    public DataSource readOnlyDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_READ_REPLICA_URL"));
        config.setUsername(System.getenv("DB_USERNAME"));
        config.setPassword(System.getenv("DB_PASSWORD"));
        config.setMaximumPoolSize(30);  // read replicas can handle more
        config.setReadOnly(true);
        config.setPoolName("ReadReplicaPool");
        return new HikariDataSource(config);
    }
}

// HikariCP metrics monitoring
@Slf4j
@Component
public class ConnectionPoolMonitor {

    private final DataSource dataSource;
    private final MeterRegistry meterRegistry;

    @Scheduled(fixedRate = 60000)
    public void checkPoolHealth() {
        if (dataSource instanceof HikariDataSource hikari) {
            HikariPoolMXBean pool = hikari.getHikariPoolMXBean();
            int active = pool.getActiveConnections();
            int total = pool.getTotalConnections();
            int waiting = pool.getThreadsAwaitingConnection();

            if (waiting > 0) {
                log.warn("Connection pool pressure: {} threads waiting, {}/{} connections active",
                        waiting, active, total);
            }

            if ((double) active / total > 0.9) {
                log.warn("Connection pool near capacity: {}/{} active", active, total);
            }
        }
    }
}
```

---

## 3. Thread Pool Sizing

```java
// Tomcat thread pool sizing
// Rule of thumb: num_threads = cpu_cores * (1 + wait_time / service_time)
// For I/O-bound services: 4 cores * (1 + 10ms / 1ms) = 44 threads
// For mixed workloads: start with 100-200, tune based on load testing

@Configuration
public class ThreadPoolConfiguration {

    // General async executor
    @Bean("generalExecutor")
    public Executor generalExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors() * 2);
        executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 4);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("general-");
        executor.setKeepAliveSeconds(60);
        executor.setAllowCoreThreadTimeOut(true);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    // I/O-bound executor (email, notifications)
    @Bean("ioExecutor")
    public Executor ioExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("io-");
        executor.initialize();
        return executor;
    }

    // CPU-bound executor (report generation, data processing)
    @Bean("cpuExecutor")
    public Executor cpuExecutor() {
        int cpus = Runtime.getRuntime().availableProcessors();
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(cpus);
        executor.setMaxPoolSize(cpus + 1);  // no benefit going higher for CPU-bound
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("cpu-");
        executor.initialize();
        return executor;
    }

    // Virtual threads for Java 21+ (best for I/O-bound)
    @Bean("virtualExecutor")
    public ExecutorService virtualExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}

// Scheduled task pool
@Configuration
@EnableScheduling
public class SchedulingConfiguration implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.setErrorHandler(throwable ->
                log.error("Scheduled task error: {}", throwable.getMessage(), throwable));
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```

---

## 4. JVM Heap Sizing for Containers

```bash
# Docker/Kubernetes JVM settings
# Never use: -Xmx512m -Xms512m (static sizes break in containers)
# Use: container-aware flags (Java 8u191+ / Java 10+)

# In Dockerfile or entrypoint
JAVA_OPTS="-XX:+UseContainerSupport \
           -XX:MaxRAMPercentage=75.0 \
           -XX:InitialRAMPercentage=50.0 \
           -XX:+UseG1GC \
           -XX:MaxGCPauseMillis=200 \
           -XX:+HeapDumpOnOutOfMemoryError \
           -XX:HeapDumpPath=/var/log/heap-dump.hprof \
           -XX:+ExitOnOutOfMemoryError \
           -XX:NativeMemoryTracking=summary \
           -Dfile.encoding=UTF-8 \
           -Djava.security.egd=file:/dev/./urandom"

# For Kubernetes:
# resources:
#   requests:
#     memory: "512Mi"
#     cpu: "500m"
#   limits:
#     memory: "1Gi"
#     cpu: "2000m"
# JVM uses 75% of 1Gi = ~768Mi for heap
```

```java
// JVM tuning in Spring Boot
@Configuration
public class JvmConfiguration {

    @Bean
    public ApplicationRunner jvmInfoLogger() {
        return args -> {
            Runtime runtime = Runtime.getRuntime();
            long maxMemory = runtime.maxMemory() / 1024 / 1024;
            int cpus = runtime.availableProcessors();

            log.info("JVM initialized: max heap={}MB, CPUs={}", maxMemory, cpus);
            log.info("Java version: {}, VM: {}",
                    System.getProperty("java.version"),
                    System.getProperty("java.vm.name"));
        };
    }
}

// GC tuning recommendations
// G1GC (default for Java 9+): -XX:+UseG1GC -XX:MaxGCPauseMillis=200
// ZGC (Java 15+, low latency): -XX:+UseZGC -XX:SoftMaxHeapSize=2g
// Shenandoah (low latency alternative): -XX:+UseShenandoahGC

// JVM flags for production
// -XX:+PrintFlagsFinal → log all JVM flags at startup
// -XX:+PrintGCDetails → verbose GC logging (use -Xlog:gc* in Java 9+)
// -Xlog:gc*:file=/var/log/gc.log:time,tags:filecount=5,filesize=20m
```

---

## 5. Graceful Shutdown Configuration

```java
// application.yml
// server.shutdown: graceful
// spring.lifecycle.timeout-per-shutdown-phase: 30s

// Custom shutdown hook
@Component
@Slf4j
public class GracefulShutdownHook {

    private final ApplicationContext context;
    private volatile boolean shuttingDown = false;

    @EventListener(ContextClosedEvent.class)
    public void onShutdown(ContextClosedEvent event) {
        log.info("Application shutdown initiated");
        shuttingDown = true;
    }

    public boolean isShuttingDown() {
        return shuttingDown;
    }
}

// Kubernetes health probes during shutdown
@RestController
@RequestMapping("/actuator")
public class HealthProbeController {

    private final GracefulShutdownHook shutdownHook;

    @GetMapping("/health/readiness")
    public ResponseEntity<Map<String, String>> readiness() {
        if (shutdownHook.isShuttingDown()) {
            // Tell Kubernetes to stop sending traffic BEFORE the pod exits
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                    .body(Map.of("status", "OUT_OF_SERVICE"));
        }
        return ResponseEntity.ok(Map.of("status", "UP"));
    }
}

// Graceful shutdown for Kafka consumers
@Component
public class KafkaConsumerShutdown {

    private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    @EventListener(ContextClosedEvent.class)
    @Order(Ordered.HIGHEST_PRECEDENCE)  // run before other shutdown hooks
    public void onShutdown() {
        log.info("Stopping Kafka listeners");
        kafkaListenerEndpointRegistry.stop(() ->
                log.info("All Kafka listeners stopped"));
    }
}

// Drain in-flight requests with SmartLifecycle
@Component
public class RequestDrainLifecycle implements SmartLifecycle {

    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private volatile boolean running = false;

    @Override
    public void start() {
        running = true;
    }

    @Override
    public void stop(Runnable callback) {
        running = false;
        log.info("Waiting for {} active requests to complete", activeRequests.get());

        // Wait up to 30 seconds for active requests
        Instant deadline = Instant.now().plusSeconds(30);
        while (activeRequests.get() > 0 && Instant.now().isBefore(deadline)) {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }

        log.info("Shutdown complete, {} requests still active", activeRequests.get());
        callback.run();
    }

    @Override
    public boolean isRunning() { return running; }

    @Override
    public int getPhase() { return Integer.MAX_VALUE - 1; }

    public void trackRequest() { activeRequests.incrementAndGet(); }
    public void releaseRequest() { activeRequests.decrementAndGet(); }
}
```

---

## 6. Health Check Implementation

```java
// Custom health indicators
@Component
public class ExternalPaymentServiceHealth implements HealthIndicator {

    private final PaymentClient paymentClient;
    private final CircuitBreaker circuitBreaker;

    @Override
    public Health health() {
        if (circuitBreaker.getState() == CircuitBreaker.State.OPEN) {
            return Health.down()
                    .withDetail("reason", "Circuit breaker is OPEN")
                    .withDetail("failureRate", circuitBreaker.getMetrics().getFailureRate())
                    .build();
        }

        try {
            paymentClient.ping();  // lightweight health endpoint
            return Health.up()
                    .withDetail("provider", "Stripe")
                    .withDetail("circuitBreaker", circuitBreaker.getState().name())
                    .build();
        } catch (Exception e) {
            return Health.down(e)
                    .withDetail("provider", "Stripe")
                    .withDetail("reason", e.getMessage())
                    .build();
        }
    }
}

@Component
public class DiskSpaceHealth implements HealthIndicator {

    @Override
    public Health health() {
        File logDir = new File("/var/log");
        long freeBytes = logDir.getFreeSpace();
        long totalBytes = logDir.getTotalSpace();
        double usedPercent = (double)(totalBytes - freeBytes) / totalBytes * 100;

        if (usedPercent > 90) {
            return Health.down()
                    .withDetail("path", "/var/log")
                    .withDetail("usedPercent", String.format("%.1f%%", usedPercent))
                    .withDetail("freeGB", freeBytes / 1024 / 1024 / 1024)
                    .build();
        }

        return Health.up()
                .withDetail("path", "/var/log")
                .withDetail("usedPercent", String.format("%.1f%%", usedPercent))
                .build();
    }
}

@Component
public class DatabaseConnectionHealth implements HealthIndicator {

    private final DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {

            ResultSet rs = stmt.executeQuery("SELECT 1");
            rs.next();

            if (dataSource instanceof HikariDataSource hikari) {
                HikariPoolMXBean pool = hikari.getHikariPoolMXBean();
                return Health.up()
                        .withDetail("active", pool.getActiveConnections())
                        .withDetail("idle", pool.getIdleConnections())
                        .withDetail("total", pool.getTotalConnections())
                        .withDetail("awaiting", pool.getThreadsAwaitingConnection())
                        .build();
            }

            return Health.up().build();
        } catch (SQLException e) {
            return Health.down(e).withDetail("reason", e.getMessage()).build();
        }
    }
}

// Kubernetes liveness and readiness probes
@Configuration
public class ProbeConfiguration {

    @Bean
    public LivenessStateHealthIndicator livenessIndicator() {
        return new LivenessStateHealthIndicator(applicationAvailability);
    }

    @Bean
    public ReadinessStateHealthIndicator readinessIndicator() {
        return new ReadinessStateHealthIndicator(applicationAvailability);
    }
}

// Mark app as not ready during warm-up
@Component
public class ApplicationWarmUp implements ApplicationListener<ApplicationReadyEvent> {

    private final ApplicationAvailability availability;
    private final CacheManager cacheManager;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("Application started, beginning warm-up");

        // Pre-warm caches
        warmCaches();

        // Signal readiness
        ((AvailabilityChangeEvent<ReadinessState>) AvailabilityChangeEvent.publish(
                event.getApplicationContext(), ReadinessState.ACCEPTING_TRAFFIC))
                .getState();

        log.info("Warm-up complete, accepting traffic");
    }

    private void warmCaches() {
        // Pre-load frequently accessed data into cache
    }
}
```

---

## 7. Circuit Breaker for Production

```java
// application.yml - Resilience4j for production
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 100
        failureRateThreshold: 50        # open at 50% failures
        slowCallRateThreshold: 80       # slow call threshold
        slowCallDurationThreshold: 3s
        waitDurationInOpenState: 60s    # wait before half-open
        permittedNumberOfCallsInHalfOpenState: 10
        minimumNumberOfCalls: 20        # min calls before circuit can open
        automaticTransitionFromOpenToHalfOpenEnabled: true
      inventoryService:
        registerHealthIndicator: true
        slidingWindowSize: 50
        failureRateThreshold: 40
        waitDurationInOpenState: 30s
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - java.net.ConnectException
        ignoreExceptions:
          - com.example.PaymentDeclinedException
  timelimiter:
    instances:
      paymentService:
        timeoutDuration: 5s
        cancelRunningFuture: true
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 25
        maxWaitDuration: 2s

// Usage with annotations
@Service
public class PaymentService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    @Bulkhead(name = "paymentService")
    public CompletableFuture<PaymentResult> charge(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> stripeClient.charge(request));
    }

    private CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest request, Exception ex) {
        log.error("Payment circuit breaker fallback triggered: {}", ex.getMessage());
        return CompletableFuture.completedFuture(
                PaymentResult.failed("Payment service unavailable. Please try again."));
    }
}
```

---

## 8. Logging Configuration

```xml
<!-- logback-spring.xml -->
<configuration scan="true" scanPeriod="30 seconds">

    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="environment" source="app.environment" defaultValue="local"/>

    <!-- Console (local dev) -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- File with rolling policy -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/${appName}/application.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"${appName}","env":"${environment}"}</customFields>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>/var/log/${appName}/application-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- Async appender wraps file appender for performance -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE"/>
        <queueSize>10000</queueSize>
        <discardingThreshold>20</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <neverBlock>false</neverBlock>
    </appender>

    <!-- Separate error log for alerting -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/${appName}/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/${appName}/error-%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>90</maxHistory>
        </rollingPolicy>
    </appender>

    <!-- Profile-specific configuration -->
    <springProfile name="production">
        <root level="WARN">
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
        </root>
        <logger name="com.example" level="INFO"/>
    </springProfile>

    <springProfile name="local,development">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

</configuration>
```

```java
// Structured logging with MDC
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String requestId = Optional.ofNullable(request.getHeader("X-Request-ID"))
                .orElse(UUID.randomUUID().toString());
        String userId = extractUserId(request);

        MDC.put("requestId", requestId);
        MDC.put("userId", userId != null ? userId : "anonymous");
        MDC.put("method", request.getMethod());
        MDC.put("path", request.getRequestURI());

        response.setHeader("X-Request-ID", requestId);

        long start = System.currentTimeMillis();
        try {
            chain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - start;
            MDC.put("statusCode", String.valueOf(response.getStatus()));
            MDC.put("duration", String.valueOf(duration));

            log.info("HTTP {} {} → {} ({}ms)", request.getMethod(),
                    request.getRequestURI(), response.getStatus(), duration);
            MDC.clear();
        }
    }
}
```

---

## 9. Secret Management

```java
// ===== AWS Parameter Store =====
// pom.xml: spring-cloud-starter-aws-parameter-store-config

// bootstrap.yml
/*
aws:
  paramstore:
    enabled: true
    prefix: /config
    profile-separator: _
    fail-fast: true
    name: order-service
*/
// Loads secrets from /config/order-service/db.password etc.

// ===== HashiCorp Vault =====
// pom.xml: spring-cloud-starter-vault-config

// bootstrap.yml
/*
spring:
  cloud:
    vault:
      uri: https://vault.company.com:8200
      authentication: KUBERNETES
      kubernetes:
        role: order-service
      generic:
        enabled: true
        default-context: order-service
      database:
        enabled: true
        role: order-service-db
*/

// ===== Kubernetes Secrets =====
// Mount as environment variables or volume
/*
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-service-secrets
        key: db-password
  - name: STRIPE_API_KEY
    valueFrom:
      secretKeyRef:
        name: payment-secrets
        key: stripe-api-key
*/

// ===== Never do this =====
// @Value("${db.password:hardcoded-password}")  // NO!

// ===== Do this =====
@Service
public class SecretAwareService {

    @Value("${db.password}")  // loaded from environment, Vault, SSM, or K8s Secret
    private String dbPassword;

    // Or use a dedicated secrets bean
    @Autowired
    private ApplicationSecrets secrets;
}

@Configuration
@ConfigurationProperties(prefix = "app.secrets")
@Validated
public class ApplicationSecrets {

    @NotBlank
    private String stripeApiKey;

    @NotBlank
    private String jwtSecret;

    @NotBlank
    private String encryptionKey;

    // getters/setters
}

// Secret rotation support
@Component
@Slf4j
public class SecretRotationListener {

    private final ApplicationSecrets secrets;
    private final EncryptionService encryptionService;

    @EventListener
    public void onEnvironmentChanged(EnvironmentChangeEvent event) {
        if (event.getKeys().contains("app.secrets.encryption-key")) {
            log.info("Encryption key rotated, updating encryption service");
            encryptionService.reloadKey(secrets.getEncryptionKey());
        }
    }
}
```

---

## 10. Application Metrics and Alerting

```java
// Custom business metrics
@Service
@Slf4j
public class OrderMetricsService {

    private final MeterRegistry registry;
    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderAmountSummary;
    private final AtomicInteger pendingOrders;

    public OrderMetricsService(MeterRegistry registry) {
        this.registry = registry;

        this.orderCreatedCounter = Counter.builder("orders.created")
                .description("Total orders created")
                .tags("env", "production")
                .register(registry);

        this.orderFailedCounter = Counter.builder("orders.failed")
                .description("Total orders that failed")
                .register(registry);

        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
                .description("Order processing time")
                .publishPercentiles(0.5, 0.9, 0.95, 0.99)
                .publishPercentileHistogram()
                .register(registry);

        this.orderAmountSummary = DistributionSummary.builder("orders.amount")
                .description("Distribution of order amounts in USD")
                .baseUnit("USD")
                .publishPercentiles(0.5, 0.9, 0.95, 0.99)
                .register(registry);

        this.pendingOrders = registry.gauge("orders.pending.count",
                new AtomicInteger(0));
    }

    public void recordOrderCreated(Order order) {
        orderCreatedCounter.increment(
                Tags.of("payment_method", order.getPaymentMethod()));
        orderAmountSummary.record(order.getTotalAmount().doubleValue());
    }

    public void recordOrderFailed(String reason) {
        orderFailedCounter.increment(Tags.of("reason", reason));
    }

    public <T> T timeOrderProcessing(Callable<T> operation) {
        return orderProcessingTimer.record(operation);
    }

    public void updatePendingCount(int count) {
        pendingOrders.set(count);
    }
}

// SLO-based alerts (Prometheus AlertManager rules)
/*
groups:
  - name: order-service-slos
    rules:
      # Error rate
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{
            application="order-service",
            status=~"5.."
          }[5m]) /
          rate(http_server_requests_seconds_count{
            application="order-service"
          }[5m]) > 0.01
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Order Service error rate > 1%"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # Latency SLO
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket{
              application="order-service",
              uri="/api/orders"
            }[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 1s on /api/orders"

      # Connection pool exhaustion
      - alert: ConnectionPoolNearCapacity
        expr: |
          hikaricp_connections_active{pool="OrderServicePool"} /
          hikaricp_connections_max{pool="OrderServicePool"} > 0.9
        for: 1m
        labels:
          severity: warning

      # Circuit breaker open
      - alert: CircuitBreakerOpen
        expr: resilience4j_circuitbreaker_state{state="open"} == 1
        for: 1m
        labels:
          severity: critical
*/
```

---

## 11. Incident Response Runbook

```java
// Embed runbook info in health endpoint
@Component
public class RunbookHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        return Health.unknown()
                .withDetail("runbook", "https://wiki.company.com/runbooks/order-service")
                .withDetail("oncall", "platform-team@company.com")
                .withDetail("slack", "#platform-alerts")
                .build();
    }
}
```

```markdown
# Order Service Runbook

## High Error Rate (5xx)

**Symptoms**: Alert: HighErrorRate fires

**Immediate Actions**:
1. Check logs: `kubectl logs -l app=order-service --since=10m | grep ERROR`
2. Check recent deployments: `kubectl rollout history deployment/order-service`
3. Check DB connection pool: `GET /actuator/health/db`
4. Check circuit breakers: `GET /actuator/health/circuitbreakers`

**Resolution**:
- Recent deploy → Rollback: `kubectl rollout undo deployment/order-service`
- DB issue → Check DB dashboard, failover if needed
- Payment service down → Circuit breaker will open automatically, monitor

---

## OOM / Memory Issues

**Symptoms**: Pod restarts, OutOfMemoryError in logs

**Immediate Actions**:
1. Get heap dump: `kubectl exec <pod> -- jcmd 1 VM.native_memory`
2. Check heap usage: `GET /actuator/metrics/jvm.memory.used?tag=area:heap`
3. Trigger manual GC: `jcmd <pid> GC.run` (temporary relief only)

**Resolution**:
- Increase memory limit in deployment (notify capacity team)
- Analyze heap dump for memory leaks (Eclipse MAT or VisualVM)
- Check for ThreadLocal leaks, static Map growth, cache misses

---

## Slow Response Times

**Symptoms**: Alert: HighP99Latency fires

**Immediate Actions**:
1. Check DB slow query log
2. Check HikariCP pool: waiting threads at `/actuator/health/db`
3. Check external dependencies: `GET /actuator/health`
4. Enable SQL logging temporarily: POST /actuator/loggers/org.hibernate.SQL level=DEBUG

**Resolution**:
- Slow DB query → Add index, optimize query
- Pool exhaustion → Increase `maximum-pool-size` (check DB connection limit first)
- External service slow → Circuit breaker, increase timeout, or add fallback
```

---

## 12. Real Example: Production-Ready Application

```java
// ===== Complete production-ready main class =====

@SpringBootApplication
@EnableAsync
@EnableCaching
@EnableScheduling
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(OrderServiceApplication.class);
        app.setBannerMode(Banner.Mode.LOG);  // no banner in production
        app.run(args);
    }
}

// ===== Production validation at startup =====

@Component
@Slf4j
public class StartupValidator implements ApplicationRunner {

    @Value("${app.secrets.jwt-secret}")
    private String jwtSecret;

    @Value("${app.secrets.encryption-key}")
    private String encryptionKey;

    @Autowired
    private DataSource dataSource;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        log.info("Running startup validation...");
        validateSecrets();
        validateDatabaseConnectivity();
        log.info("Startup validation complete");
    }

    private void validateSecrets() {
        if (jwtSecret == null || jwtSecret.length() < 32) {
            throw new IllegalStateException("JWT secret is missing or too short (min 32 chars)");
        }
        if (encryptionKey == null || encryptionKey.isEmpty()) {
            throw new IllegalStateException("Encryption key is required");
        }
        log.info("Secrets validated");
    }

    private void validateDatabaseConnectivity() throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            conn.prepareStatement("SELECT 1").executeQuery();
            log.info("Database connectivity confirmed");
        } catch (SQLException e) {
            throw new IllegalStateException("Cannot connect to database at startup", e);
        }
    }
}

// ===== Production-ready service with all patterns =====

@Service
@Slf4j
@Validated
public class ProductionOrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final AuditLogService auditLogService;
    private final OrderMetricsService metrics;
    private final TransactionTemplate transactionTemplate;

    @Transactional
    public OrderDto createOrder(@Valid CreateOrderRequest request) {
        return metrics.timeOrderProcessing(() -> {
            try {
                // 1. Validate + reserve inventory (idempotent with request ID)
                inventoryService.reserve(request.getItems(), request.getIdempotencyKey());

                // 2. Create order in DB
                Order order = Order.from(request);
                order = orderRepository.save(order);

                // 3. Process payment (outside main transaction to avoid long lock)
                Long orderId = order.getId();
                transactionTemplate.executeWithoutResult(status -> {
                    PaymentResult payment = paymentService.charge(
                            new PaymentRequest(request.getPaymentToken(), order.getTotalAmount()));

                    if (!payment.isSuccess()) {
                        inventoryService.release(request.getItems(), request.getIdempotencyKey());
                        status.setRollbackOnly();
                        metrics.recordOrderFailed("payment_declined");
                        throw new PaymentFailedException(payment.getErrorMessage());
                    }

                    orderRepository.markConfirmed(orderId, payment.getTransactionId());
                });

                // 4. Record metrics
                metrics.recordOrderCreated(order);

                // 5. Async post-processing
                orderEventPublisher.publishOrderConfirmed(order);

                return orderMapper.toDto(order);
            } catch (Exception e) {
                metrics.recordOrderFailed(e.getClass().getSimpleName());
                throw e;
            }
        });
    }
}
```

---

## Production Readiness Checklist

| Category | Item | Status |
|---|---|---|
| **Database** | `spring.jpa.open-in-view=false` | Critical |
| **Database** | HikariCP pool sized correctly | Required |
| **Database** | `max-lifetime` < DB timeout | Required |
| **Database** | Connection validation enabled | Required |
| **Server** | `server.shutdown=graceful` | Required |
| **Server** | Tomcat thread pool sized | Required |
| **Server** | HTTP/2 enabled | Recommended |
| **Server** | Response compression enabled | Recommended |
| **JVM** | Container-aware memory flags | Required |
| **JVM** | `-XX:+ExitOnOutOfMemoryError` | Required |
| **JVM** | Heap dump on OOM configured | Required |
| **Security** | All secrets from env/vault/SSM | Critical |
| **Security** | No hardcoded credentials | Critical |
| **Logging** | JSON structured logging | Required |
| **Logging** | Log rotation configured | Required |
| **Logging** | PII masking in logs | Required |
| **Logging** | MDC with request ID | Required |
| **Metrics** | Prometheus endpoint exposed | Required |
| **Metrics** | Business metrics defined | Required |
| **Metrics** | SLO alerts configured | Required |
| **Health** | Liveness probe implemented | Required |
| **Health** | Readiness probe implemented | Required |
| **Health** | Circuit breakers with health indicator | Required |
| **Resilience** | Circuit breakers on all external calls | Required |
| **Resilience** | Retry with exponential backoff | Required |
| **Resilience** | Timeouts on all I/O | Required |
| **Resilience** | Bulkhead on high-traffic paths | Recommended |
| **Auditing** | `@CreatedDate`/`@LastModifiedDate` | Required |
| **Auditing** | Audit log for sensitive operations | Required |
| **GDPR** | PII encryption at rest | Required |
| **GDPR** | Data erasure endpoint | Required |
| **GDPR** | Data export endpoint | Required |
| **Operations** | Runbook documented | Required |
| **Operations** | On-call contacts configured | Required |
| **Startup** | Secret presence validated | Required |
| **Startup** | DB connectivity validated | Required |

---

## Summary

| Area | Key Config |
|---|---|
| Connection pool | `maximum-pool-size=20`, `max-lifetime=1800000`, `connection-test-query` |
| Thread pool | `server.tomcat.threads.max=200`, custom `Executor` beans |
| JVM | `UseContainerSupport`, `MaxRAMPercentage=75`, `ExitOnOutOfMemoryError` |
| Shutdown | `server.shutdown=graceful`, 30s drain timeout |
| Health | `/actuator/health/liveness`, `/actuator/health/readiness` |
| Secrets | AWS SSM / Vault / K8s Secrets — never hardcode |
| Logging | JSON + async appender + rotation + PII masking |
| Metrics | Prometheus + SLO alerts on error rate and latency |
| Resilience | Circuit breaker + retry + timeout + bulkhead |

## Course Complete

Congratulations — you have completed 90 parts covering Java fundamentals through enterprise Spring Boot production systems. You are now equipped to build, test, secure, and operate production-grade Java applications.

**Recommended next steps**:
- Deploy the e-commerce project end-to-end on Kubernetes
- Set up a CI/CD pipeline with the GitHub Actions patterns from Part 034
- Configure observability with Grafana + Prometheus + Loki using the metrics from this part
