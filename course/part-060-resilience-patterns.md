# Part 060: Resilience Patterns for Microservices

## Overview

Microservices fail. Networks partition, downstream services become slow, and databases run out of connections. Resilience patterns prevent a single failure from cascading into a system-wide outage. This part covers every major resilience technique, implemented with **Resilience4j** in Spring Boot.

---

## Maven Dependencies

```xml
<dependencies>
    <!-- Resilience4j -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>2.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <!-- Rate limiting -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-ratelimiter</artifactId>
        <version>2.2.0</version>
    </dependency>
    <!-- Micrometer for metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <!-- Caffeine for caching fallback -->
    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>
</dependencies>
```

---

## 1. Circuit Breaker Deep Dive

### States and Transitions

```
CLOSED ──(failure rate > threshold)──> OPEN
OPEN   ──(wait duration elapsed)──> HALF_OPEN
HALF_OPEN ──(success)──> CLOSED
HALF_OPEN ──(failure)──> OPEN
```

```java
// Complete Circuit Breaker configuration
@Configuration
public class CircuitBreakerConfig {

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        io.github.resilience4j.circuitbreaker.CircuitBreakerConfig config =
            io.github.resilience4j.circuitbreaker.CircuitBreakerConfig.custom()
                // Failure rate threshold (opens CB when >= 50% of calls fail)
                .failureRateThreshold(50)
                // Minimum calls before calculating failure rate
                .minimumNumberOfCalls(10)
                // Sliding window: COUNT_BASED or TIME_BASED
                .slidingWindowType(
                    io.github.resilience4j.circuitbreaker.CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(20)
                // How long to stay OPEN before trying HALF_OPEN
                .waitDurationInOpenState(Duration.ofSeconds(30))
                // Number of calls in HALF_OPEN state
                .permittedNumberOfCallsInHalfOpenState(5)
                // Slow call threshold (also counts as failure)
                .slowCallRateThreshold(80)
                .slowCallDurationThreshold(Duration.ofSeconds(2))
                // Which exceptions count as failures
                .recordExceptions(IOException.class, TimeoutException.class,
                    HttpServerErrorException.class)
                // Which exceptions to ignore (don't count as failures)
                .ignoreExceptions(BusinessValidationException.class,
                    ResourceNotFoundException.class)
                .build();

        return CircuitBreakerRegistry.of(config);
    }
}

// Circuit Breaker with event listeners
@Service
@Slf4j
public class PaymentServiceClient {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final RestTemplate restTemplate;
    private final MeterRegistry meterRegistry;

    @PostConstruct
    public void setupCircuitBreakerEvents() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("payment-service");

        cb.getEventPublisher()
            .onStateTransition(event -> {
                log.warn("Circuit breaker '{}' state: {} -> {}",
                    event.getCircuitBreakerName(),
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState());

                meterRegistry.counter("circuit_breaker.state_transition",
                    "name", event.getCircuitBreakerName(),
                    "from", event.getStateTransition().getFromState().name(),
                    "to", event.getStateTransition().getToState().name()
                ).increment();
            })
            .onError(event ->
                log.error("CB error in '{}': {}",
                    event.getCircuitBreakerName(),
                    event.getThrowable().getMessage()))
            .onSuccess(event ->
                log.debug("CB success in '{}', duration: {}ms",
                    event.getCircuitBreakerName(),
                    event.getElapsedDuration().toMillis()));
    }

    @CircuitBreaker(name = "payment-service", fallbackMethod = "processPaymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            request,
            PaymentResult.class
        );
    }

    // Fallback method (same signature + Throwable parameter)
    public PaymentResult processPaymentFallback(PaymentRequest request, Throwable ex) {
        log.warn("Payment service circuit breaker open, using fallback. Error: {}",
            ex.getMessage());

        // Queue for later processing
        return PaymentResult.queued(request.getOrderId(),
            "Payment queued - service temporarily unavailable");
    }

    // Get current circuit breaker metrics
    public CircuitBreakerMetrics getMetrics() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("payment-service");
        CircuitBreaker.Metrics metrics = cb.getMetrics();

        return CircuitBreakerMetrics.builder()
            .state(cb.getState().name())
            .failureRate(metrics.getFailureRate())
            .slowCallRate(metrics.getSlowCallRate())
            .numberOfBufferedCalls(metrics.getNumberOfBufferedCalls())
            .numberOfFailedCalls(metrics.getNumberOfFailedCalls())
            .numberOfSuccessfulCalls(metrics.getNumberOfSuccessfulCalls())
            .build();
    }
}
```

### YAML Configuration for Circuit Breaker

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        failure-rate-threshold: 50
        minimum-number-of-calls: 10
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 2s
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.exception.BusinessValidationException
      inventory-service:
        failure-rate-threshold: 60
        sliding-window-size: 30
        wait-duration-in-open-state: 60s
```

---

## 2. Bulkhead Pattern

Isolate resource pools to prevent one failing service from consuming all threads.

### Thread Pool Bulkhead

```java
// Thread pool bulkhead: each downstream gets its own thread pool
@Configuration
public class BulkheadConfig {

    @Bean
    public ThreadPoolBulkheadRegistry threadPoolBulkheadRegistry() {
        ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)
            .coreThreadPoolSize(5)
            .queueCapacity(20)
            .keepAliveDuration(Duration.ofSeconds(20))
            .writableStackTraceEnabled(true)
            .build();

        return ThreadPoolBulkheadRegistry.of(config);
    }
}

@Service
@Slf4j
public class NotificationServiceClient {

    private final ThreadPoolBulkheadRegistry bulkheadRegistry;
    private final RestTemplate restTemplate;

    // Thread pool bulkhead (async)
    @Bulkhead(name = "notification-service", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<Void> sendNotification(NotificationRequest request) {
        return CompletableFuture.runAsync(() -> {
            restTemplate.postForObject(
                "http://notification-service/api/notifications",
                request,
                Void.class
            );
        });
    }

    // Fallback when thread pool is saturated
    public CompletableFuture<Void> sendNotificationFallback(
            NotificationRequest request, BulkheadFullException ex) {
        log.warn("Notification bulkhead full, dropping notification for order {}",
            request.getOrderId());
        // Could push to a dead-letter queue instead
        return CompletableFuture.completedFuture(null);
    }
}

// Semaphore bulkhead (synchronous)
@Service
public class InventoryServiceClient {

    private final BulkheadRegistry bulkheadRegistry;
    private final RestTemplate restTemplate;

    @PostConstruct
    public void configureBulkhead() {
        // Custom config for this specific service
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(15)
            .maxWaitDuration(Duration.ofMillis(500))
            .writableStackTraceEnabled(true)
            .build();

        bulkheadRegistry.bulkhead("inventory-service", config);
    }

    @Bulkhead(name = "inventory-service", fallbackMethod = "checkInventoryFallback")
    public InventoryStatus checkInventory(String productId, int quantity) {
        return restTemplate.getForObject(
            "http://inventory-service/api/inventory/{productId}/check?quantity={qty}",
            InventoryStatus.class,
            productId, quantity
        );
    }

    public InventoryStatus checkInventoryFallback(
            String productId, int quantity, BulkheadFullException ex) {
        log.warn("Inventory bulkhead full for product {}", productId);
        return InventoryStatus.unknown(productId);
    }
}
```

### YAML for Bulkheads

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      notification-service:
        max-thread-pool-size: 10
        core-thread-pool-size: 5
        queue-capacity: 20
        keep-alive-duration: 20s
  bulkhead:
    instances:
      inventory-service:
        max-concurrent-calls: 15
        max-wait-duration: 500ms
      payment-service:
        max-concurrent-calls: 25
        max-wait-duration: 1s
```

---

## 3. Retry with Exponential Backoff and Jitter

```java
// Retry configuration with jitter to prevent thundering herd
@Configuration
public class RetryConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        io.github.resilience4j.retry.RetryConfig config =
            io.github.resilience4j.retry.RetryConfig.custom()
                .maxAttempts(4)
                // Exponential backoff: 1s, 2s, 4s
                .intervalFunction(IntervalFunction.ofExponentialBackoff(
                    Duration.ofSeconds(1), 2.0))
                // Add random jitter (±25%) to prevent thundering herd
                .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                    Duration.ofSeconds(1), 2.0, 0.25))
                // Only retry on transient errors
                .retryExceptions(IOException.class, TimeoutException.class,
                    ServiceUnavailableException.class)
                // Don't retry on business errors
                .ignoreExceptions(BusinessValidationException.class,
                    IllegalArgumentException.class)
                // Retry on specific HTTP status codes
                .retryOnResult(response -> {
                    if (response instanceof ResponseEntity<?> r) {
                        return r.getStatusCode().is5xxServerError();
                    }
                    return false;
                })
                .build();

        return RetryRegistry.of(config);
    }
}

// Manual retry with custom jitter
@Component
@Slf4j
public class RetryableHttpClient {

    private final RetryRegistry retryRegistry;
    private final RestTemplate restTemplate;
    private final Random random = new Random();

    @Retry(name = "external-api", fallbackMethod = "callExternalApiFallback")
    public <T> T callWithRetry(String url, Class<T> responseType) {
        return restTemplate.getForObject(url, responseType);
    }

    // Programmatic retry with full control
    public <T> T callWithCustomRetry(String url, Class<T> responseType) {
        Retry retry = retryRegistry.retry("custom-retry",
            io.github.resilience4j.retry.RetryConfig.custom()
                .maxAttempts(5)
                .intervalBiFunction((attempt, either) -> {
                    // Custom backoff: base delay + jitter
                    long baseDelay = (long) Math.pow(2, attempt - 1) * 1000; // 1s, 2s, 4s, 8s
                    long jitter = (long) (random.nextDouble() * baseDelay * 0.3); // ±30%
                    long delay = baseDelay + jitter;
                    log.info("Retry attempt {}, waiting {}ms", attempt, delay);
                    return delay;
                })
                .build());

        return Retry.decorateSupplier(retry,
            () -> restTemplate.getForObject(url, responseType)).get();
    }

    public <T> T callExternalApiFallback(String url, Class<T> responseType, Throwable ex) {
        log.error("All retries exhausted for URL: {}", url, ex);
        throw new ServiceUnavailableException("Service unavailable after retries: " + url);
    }
}
```

### YAML for Retry

```yaml
resilience4j:
  retry:
    instances:
      external-api:
        max-attempts: 4
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        exponential-max-wait-duration: 30s
        randomized-wait-factor: 0.25
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.exception.BusinessValidationException
```

---

## 4. Rate Limiting

### Token Bucket Algorithm

```java
// Token bucket: allows burst traffic
@Configuration
public class RateLimiterConfig {

    @Bean
    public RateLimiterRegistry rateLimiterRegistry() {
        io.github.resilience4j.ratelimiter.RateLimiterConfig config =
            io.github.resilience4j.ratelimiter.RateLimiterConfig.custom()
                // Refresh period: how often tokens are refilled
                .limitRefreshPeriod(Duration.ofSeconds(1))
                // Tokens per refresh period
                .limitForPeriod(100)
                // Max time to wait for a permit
                .timeoutDuration(Duration.ofMillis(500))
                .build();

        return RateLimiterRegistry.of(config);
    }
}

// Apply rate limiting
@RestController
@RequestMapping("/api/payments")
@Slf4j
public class PaymentController {

    private final PaymentService paymentService;
    private final RateLimiterRegistry rateLimiterRegistry;

    // Annotation-based rate limiting
    @PostMapping
    @RateLimiter(name = "payment-api", fallbackMethod = "processPaymentRateLimitFallback")
    public ResponseEntity<PaymentResult> processPayment(@RequestBody PaymentRequest request) {
        PaymentResult result = paymentService.process(request);
        return ResponseEntity.ok(result);
    }

    public ResponseEntity<PaymentResult> processPaymentRateLimitFallback(
            PaymentRequest request, RequestNotPermitted ex) {
        log.warn("Rate limit exceeded for payment request");
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS)
            .header("Retry-After", "1")
            .body(PaymentResult.rateLimited("Too many requests, please retry after 1 second"));
    }
}

// Custom sliding window rate limiter
@Component
public class SlidingWindowRateLimiter {

    // Redis-based sliding window for distributed rate limiting
    private final RedisTemplate<String, String> redisTemplate;

    public boolean isAllowed(String clientId, int maxRequests, Duration window) {
        long now = System.currentTimeMillis();
        long windowStart = now - window.toMillis();
        String key = "rate_limit:" + clientId;

        // Sliding window log algorithm using Redis sorted set
        redisTemplate.opsForZSet().removeRangeByScore(key, 0, windowStart);

        Long currentCount = redisTemplate.opsForZSet().size(key);
        if (currentCount != null && currentCount >= maxRequests) {
            return false;
        }

        redisTemplate.opsForZSet().add(key, UUID.randomUUID().toString(), now);
        redisTemplate.expire(key, window);
        return true;
    }
}

// Rate limiter filter for API gateway
@Component
@Slf4j
public class RateLimitingFilter implements Filter {

    private final SlidingWindowRateLimiter rateLimiter;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String clientId = extractClientId(request);
        String endpoint = request.getRequestURI();

        int limit = getLimit(endpoint);
        Duration window = Duration.ofMinutes(1);

        if (!rateLimiter.isAllowed(clientId, limit, window)) {
            log.warn("Rate limit exceeded for client {} on endpoint {}", clientId, endpoint);
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("X-RateLimit-Limit", String.valueOf(limit));
            response.setHeader("Retry-After", "60");
            response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(req, res);
    }

    private String extractClientId(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null) return "api:" + apiKey;

        String jwt = request.getHeader("Authorization");
        if (jwt != null && jwt.startsWith("Bearer ")) {
            return "jwt:" + extractSubject(jwt.substring(7));
        }

        return "ip:" + request.getRemoteAddr();
    }

    private int getLimit(String endpoint) {
        if (endpoint.startsWith("/api/payments")) return 10;
        if (endpoint.startsWith("/api/orders")) return 100;
        return 1000;
    }

    private String extractSubject(String jwt) {
        // Decode JWT to extract subject
        return jwt.substring(0, 8); // Simplified
    }
}
```

### YAML for Rate Limiter

```yaml
resilience4j:
  ratelimiter:
    instances:
      payment-api:
        limit-for-period: 10
        limit-refresh-period: 1s
        timeout-duration: 500ms
      search-api:
        limit-for-period: 50
        limit-refresh-period: 1s
        timeout-duration: 0
```

---

## 5. Timeout Patterns

```java
// TimeLimiter: wraps async calls with timeout
@Configuration
public class TimeLimiterConfig {

    @Bean
    public TimeLimiterRegistry timeLimiterRegistry() {
        io.github.resilience4j.timelimiter.TimeLimiterConfig config =
            io.github.resilience4j.timelimiter.TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(3))
                .cancelRunningFuture(true)
                .build();

        return TimeLimiterRegistry.of(config);
    }
}

// Timeout on synchronous call using annotation
@Service
public class ReportService {

    private final ReportRepository reportRepository;
    private final ExecutorService executorService = Executors.newFixedThreadPool(10);

    @TimeLimiter(name = "report-generation", fallbackMethod = "generateReportFallback")
    public CompletableFuture<byte[]> generateReport(ReportRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // Long-running report generation
            return reportRepository.generateReport(request);
        }, executorService);
    }

    public CompletableFuture<byte[]> generateReportFallback(
            ReportRequest request, TimeoutException ex) {
        log.warn("Report generation timed out for request: {}", request);
        // Return cached or partial report
        return CompletableFuture.completedFuture(
            getCachedReport(request).orElse(EMPTY_REPORT));
    }

    private Optional<byte[]> getCachedReport(ReportRequest request) {
        return Optional.empty(); // Check cache
    }

    private static final byte[] EMPTY_REPORT = new byte[0];
}

// Timeout with RestTemplate
@Configuration
public class RestTemplateTimeoutConfig {

    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(2));
        factory.setConnectionRequestTimeout(Duration.ofSeconds(1));

        RestTemplate restTemplate = new RestTemplate(factory);
        // Read timeout is set per-request via RequestConfig

        return restTemplate;
    }

    // Per-endpoint timeout using WebClient
    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
            .responseTimeout(Duration.ofSeconds(5))
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(5))
                .addHandlerLast(new WriteTimeoutHandler(5)));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

### YAML for TimeLimiter

```yaml
resilience4j:
  timelimiter:
    instances:
      report-generation:
        timeout-duration: 30s
        cancel-running-future: true
      payment-service:
        timeout-duration: 3s
        cancel-running-future: true
```

---

## 6. Fallback Strategies

```java
// Multiple fallback strategies with priority order
@Service
@Slf4j
public class ProductCatalogService {

    private final ProductRepository localRepo;
    private final ProductServiceClient remoteClient;
    private final CacheManager cacheManager;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @CircuitBreaker(name = "product-service", fallbackMethod = "getFromCache")
    @TimeLimiter(name = "product-service")
    public CompletableFuture<ProductDto> getProduct(String productId) {
        return CompletableFuture.supplyAsync(() -> remoteClient.getProduct(productId));
    }

    // Fallback 1: Try cache
    public CompletableFuture<ProductDto> getFromCache(
            String productId, Exception ex) {
        log.warn("Product service unavailable, trying cache for {}", productId);

        Cache cache = cacheManager.getCache("products");
        if (cache != null) {
            ProductDto cached = cache.get(productId, ProductDto.class);
            if (cached != null) {
                log.info("Cache hit for product {}", productId);
                return CompletableFuture.completedFuture(cached);
            }
        }

        return getFromLocalDatabase(productId, ex);
    }

    // Fallback 2: Try local DB
    public CompletableFuture<ProductDto> getFromLocalDatabase(
            String productId, Exception ex) {
        log.warn("Cache miss, trying local DB for product {}", productId);

        return productRepository.findById(productId)
            .map(product -> CompletableFuture.completedFuture(ProductDto.from(product)))
            .orElseGet(() -> getDefaultProduct(productId, ex));
    }

    // Fallback 3: Return default/placeholder
    public CompletableFuture<ProductDto> getDefaultProduct(
            String productId, Exception ex) {
        log.error("All fallbacks exhausted for product {}. Returning placeholder.", productId);

        // Queue for async retry
        kafkaTemplate.send("product.retry-fetch",
            Map.of("productId", productId, "timestamp", Instant.now()));

        return CompletableFuture.completedFuture(
            ProductDto.placeholder(productId, "Product temporarily unavailable"));
    }
}

// Cache-based fallback with TTL
@Service
public class ExchangeRateService {

    private final ExchangeRateClient remoteClient;

    // Caffeine cache with 1-hour TTL
    private final Cache<String, BigDecimal> exchangeRateCache = Caffeine.newBuilder()
        .expireAfterWrite(1, TimeUnit.HOURS)
        .maximumSize(100)
        .build();

    @CircuitBreaker(name = "exchange-rate", fallbackMethod = "getExchangeRateFallback")
    public BigDecimal getExchangeRate(String from, String to) {
        BigDecimal rate = remoteClient.getRate(from, to);
        // Update cache with fresh value
        exchangeRateCache.put(from + "-" + to, rate);
        return rate;
    }

    public BigDecimal getExchangeRateFallback(String from, String to, Exception ex) {
        log.warn("Using cached exchange rate for {}/{}", from, to);

        BigDecimal cached = exchangeRateCache.getIfPresent(from + "-" + to);
        if (cached != null) return cached;

        // Last resort: use hardcoded defaults
        log.error("No cached rate for {}/{}, using default 1.0", from, to);
        return BigDecimal.ONE;
    }
}
```

---

## 7. Health Checks and Readiness

```java
// Custom health indicators
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient paymentClient;
    private final CircuitBreakerRegistry cbRegistry;

    @Override
    public Health health() {
        CircuitBreaker cb = cbRegistry.circuitBreaker("payment-gateway");
        CircuitBreaker.State state = cb.getState();

        if (state == CircuitBreaker.State.OPEN) {
            return Health.down()
                .withDetail("circuitBreaker", "OPEN")
                .withDetail("failureRate", cb.getMetrics().getFailureRate())
                .build();
        }

        try {
            boolean isUp = paymentClient.ping();
            if (isUp) {
                return Health.up()
                    .withDetail("circuitBreaker", state.name())
                    .withDetail("failureRate", cb.getMetrics().getFailureRate())
                    .build();
            } else {
                return Health.down().withDetail("reason", "Ping returned false").build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Readiness vs Liveness probes
@Component
public class ApplicationReadinessIndicator implements ApplicationListener<ApplicationReadyEvent> {

    private volatile boolean ready = false;
    private final DataSource dataSource;
    private final KafkaAdmin kafkaAdmin;

    @EventListener
    public void onApplicationEvent(ApplicationReadyEvent event) {
        try {
            // Check all dependencies are ready
            checkDatabase();
            checkKafka();
            ready = true;
            log.info("Application is ready to accept traffic");
        } catch (Exception e) {
            log.error("Application readiness check failed", e);
            ready = false;
        }
    }

    private void checkDatabase() throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            if (!conn.isValid(5)) {
                throw new IllegalStateException("Database connection invalid");
            }
        }
    }

    private void checkKafka() {
        // Verify Kafka topics exist
        kafkaAdmin.describeTopics("order.created", "payment.processed");
    }

    public boolean isReady() {
        return ready;
    }
}

// Health check endpoint configuration
@Configuration
public class ActuatorHealthConfig {

    @Bean
    public HealthEndpoint healthEndpoint(HealthContributorRegistry registry) {
        return new HealthEndpoint(registry,
            HealthEndpointGroups.of(registry,
                WebEndpointProperties.Exposure.include("health")));
    }
}
```

```yaml
# application.yml health configuration
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,prometheus
  endpoint:
    health:
      show-details: always
      show-components: always
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState,diskSpace
        readiness:
          include: readinessState,db,kafka,paymentGateway
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
    circuitbreakers:
      enabled: true
    ratelimiters:
      enabled: true
```

---

## 8. Graceful Degradation

```java
// Feature flags for graceful degradation
@Service
@Slf4j
public class RecommendationService {

    private final RecommendationEngineClient engineClient;
    private final PopularProductsCache popularCache;
    private final FeatureFlagService featureFlags;

    public List<ProductDto> getRecommendations(String customerId) {
        if (!featureFlags.isEnabled("ai-recommendations")) {
            log.debug("AI recommendations disabled, using popular products");
            return popularCache.getTopProducts(10);
        }

        try {
            return engineClient.getPersonalizedRecommendations(customerId, 10);
        } catch (Exception e) {
            log.warn("Recommendation engine unavailable, degrading gracefully", e);
            return popularCache.getTopProducts(10);
        }
    }
}

// Graceful shutdown with in-flight request handling
@Component
@Slf4j
public class GracefulShutdownHandler implements ApplicationListener<ContextClosingEvent> {

    private final ThreadPoolTaskExecutor taskExecutor;
    private volatile boolean shutdownInitiated = false;

    @Override
    public void onApplicationEvent(ContextClosingEvent event) {
        log.info("Graceful shutdown initiated");
        shutdownInitiated = true;

        // Allow in-flight requests to complete
        taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        taskExecutor.setAwaitTerminationSeconds(30);
        taskExecutor.shutdown();

        log.info("Graceful shutdown complete");
    }

    public boolean isShuttingDown() {
        return shutdownInitiated;
    }
}

// Request filter to reject new requests during shutdown
@Component
public class ShutdownAwareFilter extends OncePerRequestFilter {

    private final GracefulShutdownHandler shutdownHandler;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        if (shutdownHandler.isShuttingDown()) {
            response.setStatus(HttpStatus.SERVICE_UNAVAILABLE.value());
            response.setHeader("Connection", "close");
            response.getWriter().write("{\"error\": \"Service is shutting down\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

---

## 9. Chaos Engineering

```java
// Chaos Monkey for Spring Boot (Chaos Monkey for Spring)
// Add dependency: de.codecentric:chaos-monkey-spring-boot

@Configuration
@Profile("chaos")
public class ChaosMonkeyConfig {
    // Chaos Monkey activates via Spring profile
    // Configuration via actuator:
    // POST /actuator/chaosmonkey/enable
    // POST /actuator/chaosmonkey/assaults with:
    // {
    //   "level": 5,
    //   "latencyActive": true,
    //   "latencyRangeStart": 2000,
    //   "latencyRangeEnd": 5000,
    //   "exceptionsActive": true,
    //   "exception": { "type": "java.lang.RuntimeException", "arguments": ["Chaos!"] },
    //   "killApplicationActive": false
    // }
}

// Custom chaos injection for testing
@Component
@Profile("chaos-test")
@Slf4j
public class ChaosInjector {

    private final Random random = new Random();
    private volatile boolean chaosEnabled = false;
    private volatile double failureRate = 0.1; // 10% failure rate
    private volatile long latencyMs = 0;

    public void maybeInjectChaos(String operation) {
        if (!chaosEnabled) return;

        // Inject latency
        if (latencyMs > 0) {
            try {
                Thread.sleep(latencyMs + (long)(random.nextDouble() * latencyMs * 0.5));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        // Inject failures
        if (random.nextDouble() < failureRate) {
            log.warn("Chaos: injecting failure for operation {}", operation);
            throw new RuntimeException("Chaos-injected failure for: " + operation);
        }
    }

    public void enable(double failureRate, long latencyMs) {
        this.failureRate = failureRate;
        this.latencyMs = latencyMs;
        this.chaosEnabled = true;
        log.warn("CHAOS ENABLED: failureRate={}, latencyMs={}", failureRate, latencyMs);
    }

    public void disable() {
        this.chaosEnabled = false;
        log.info("Chaos disabled");
    }
}
```

---

## 10. SLA, SLO, SLI Definitions and Monitoring

```java
// SLI (Service Level Indicator) metrics collection
@Component
@Slf4j
public class SliMetricsCollector {

    private final MeterRegistry registry;

    // SLI 1: Availability (% of successful requests)
    private final Counter totalRequests;
    private final Counter successfulRequests;

    // SLI 2: Latency (% of requests under threshold)
    private final Timer requestDuration;

    // SLI 3: Error rate
    private final Counter errorRequests;

    public SliMetricsCollector(MeterRegistry registry) {
        this.registry = registry;
        this.totalRequests = registry.counter("http.requests.total");
        this.successfulRequests = registry.counter("http.requests.success");
        this.errorRequests = registry.counter("http.requests.error");
        this.requestDuration = registry.timer("http.request.duration");
    }

    // Record request outcome
    public void recordRequest(boolean success, Duration duration) {
        totalRequests.increment();
        requestDuration.record(duration);

        if (success) {
            successfulRequests.increment();
        } else {
            errorRequests.increment();
        }
    }

    // Calculate availability SLI
    public double getAvailabilitySli() {
        double total = totalRequests.count();
        if (total == 0) return 1.0;
        return successfulRequests.count() / total;
    }

    // Check if meeting SLO (99.9% availability = 3 nines)
    public boolean isMeetingSlo(double sloTarget) {
        return getAvailabilitySli() >= sloTarget;
    }
}

// SLO monitoring and alerting
@Component
@Slf4j
public class SloMonitor {

    private final SliMetricsCollector sliCollector;
    private final AlertService alertService;

    private static final double AVAILABILITY_SLO = 0.999;   // 99.9%
    private static final double LATENCY_P99_SLO_MS = 200.0;  // 200ms p99

    @Scheduled(fixedRate = 60_000) // Check every minute
    public void checkSlos() {
        double availability = sliCollector.getAvailabilitySli();
        boolean meetingAvailabilitySlo = availability >= AVAILABILITY_SLO;

        if (!meetingAvailabilitySlo) {
            double burnRate = (1 - availability) / (1 - AVAILABILITY_SLO);
            alertService.sendAlert(Alert.builder()
                .severity(burnRate > 10 ? AlertSeverity.CRITICAL : AlertSeverity.WARNING)
                .title("SLO Violation: Availability")
                .message(String.format(
                    "Current availability: %.4f%% (SLO: %.3f%%), burn rate: %.2fx",
                    availability * 100, AVAILABILITY_SLO * 100, burnRate))
                .build());
        }

        log.info("SLO Status - Availability: {:.4f}% ({})",
            availability * 100,
            meetingAvailabilitySlo ? "OK" : "VIOLATED");
    }
}
```

---

## 11. Complete Example: Payment Service with All Resilience Patterns

```java
// Payment service with all resilience patterns applied

@Service
@Slf4j
public class ResilientPaymentService {

    private final PaymentGatewayClient gatewayClient;
    private final PaymentRepository paymentRepository;
    private final CacheManager cacheManager;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final SliMetricsCollector sliCollector;

    // Combined: CircuitBreaker + Retry + RateLimiter + TimeLimiter + Bulkhead
    @CircuitBreaker(name = "payment-gateway", fallbackMethod = "processPaymentFallback")
    @Retry(name = "payment-gateway")
    @RateLimiter(name = "payment-api")
    @TimeLimiter(name = "payment-gateway")
    @Bulkhead(name = "payment-gateway", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<PaymentResult> processPayment(PaymentCommand command) {
        long startTime = System.currentTimeMillis();

        return CompletableFuture.supplyAsync(() -> {
            validatePayment(command);

            PaymentGatewayRequest gatewayRequest = PaymentGatewayRequest.builder()
                .amount(command.getAmount())
                .currency(command.getCurrency())
                .customerId(command.getCustomerId())
                .orderId(command.getOrderId())
                .idempotencyKey("payment-" + command.getOrderId())
                .build();

            PaymentGatewayResponse response = gatewayClient.charge(gatewayRequest);

            // Save successful payment
            Payment payment = Payment.builder()
                .orderId(command.getOrderId())
                .customerId(command.getCustomerId())
                .amount(command.getAmount())
                .transactionId(response.getTransactionId())
                .status(PaymentStatus.SUCCESS)
                .processedAt(LocalDateTime.now())
                .build();

            paymentRepository.save(payment);

            // Record SLI success
            sliCollector.recordRequest(true,
                Duration.ofMillis(System.currentTimeMillis() - startTime));

            return PaymentResult.success(payment);
        });
    }

    // Fallback: cascade through strategies
    public CompletableFuture<PaymentResult> processPaymentFallback(
            PaymentCommand command, Throwable ex) {

        log.error("Payment gateway unavailable: {}", ex.getMessage());
        sliCollector.recordRequest(false, Duration.ZERO);

        // Strategy 1: Check if payment was already processed (idempotency)
        Optional<Payment> existing = paymentRepository
            .findByOrderIdAndStatus(command.getOrderId(), PaymentStatus.SUCCESS);
        if (existing.isPresent()) {
            log.info("Payment already processed for order {}", command.getOrderId());
            return CompletableFuture.completedFuture(PaymentResult.success(existing.get()));
        }

        // Strategy 2: Queue for async processing
        if (isTransientFailure(ex)) {
            return queuePaymentForRetry(command);
        }

        // Strategy 3: Return queued result
        return CompletableFuture.completedFuture(
            PaymentResult.queued(command.getOrderId(),
                "Payment queued for processing. You'll receive confirmation shortly."));
    }

    private CompletableFuture<PaymentResult> queuePaymentForRetry(PaymentCommand command) {
        kafkaTemplate.send("payment.retry-queue",
            Map.of(
                "command", command,
                "queuedAt", Instant.now().toString(),
                "maxRetries", 3
            ));

        log.info("Payment queued for retry for order {}", command.getOrderId());

        Payment pendingPayment = Payment.builder()
            .orderId(command.getOrderId())
            .customerId(command.getCustomerId())
            .amount(command.getAmount())
            .status(PaymentStatus.PENDING)
            .processedAt(LocalDateTime.now())
            .build();

        paymentRepository.save(pendingPayment);

        return CompletableFuture.completedFuture(
            PaymentResult.pending(command.getOrderId(),
                "Payment is being processed. Check back in a few minutes."));
    }

    private boolean isTransientFailure(Throwable ex) {
        return ex instanceof IOException
            || ex instanceof TimeoutException
            || (ex instanceof HttpServerErrorException hse
                && hse.getStatusCode().is5xxServerError());
    }

    private void validatePayment(PaymentCommand command) {
        if (command.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new BusinessValidationException("Amount must be positive");
        }
        if (command.getCustomerId() == null) {
            throw new BusinessValidationException("Customer ID is required");
        }
    }
}

// Retry processor for queued payments
@Component
@Slf4j
public class PaymentRetryProcessor {

    private final PaymentService paymentService;

    @KafkaListener(topics = "payment.retry-queue", groupId = "payment-retry")
    public void processRetryPayment(Map<String, Object> message) {
        PaymentCommand command = mapToCommand(message);
        int maxRetries = (int) message.getOrDefault("maxRetries", 3);

        log.info("Retrying payment for order {}", command.getOrderId());

        try {
            paymentService.processPayment(command).get(10, TimeUnit.SECONDS);
        } catch (Exception e) {
            log.error("Retry failed for order {}", command.getOrderId(), e);
            // Will be retried again by Kafka consumer
        }
    }

    private PaymentCommand mapToCommand(Map<String, Object> message) {
        // Map message fields to PaymentCommand
        return PaymentCommand.builder()
            .orderId(Long.parseLong(message.get("orderId").toString()))
            .customerId(message.get("customerId").toString())
            .amount(new BigDecimal(message.get("amount").toString()))
            .currency(message.getOrDefault("currency", "USD").toString())
            .build();
    }
}
```

### Full YAML Configuration

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payment-gateway:
        failure-rate-threshold: 50
        minimum-number-of-calls: 10
        sliding-window-size: 20
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 2s
  retry:
    instances:
      payment-gateway:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        randomized-wait-factor: 0.3
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.exception.BusinessValidationException
  ratelimiter:
    instances:
      payment-api:
        limit-for-period: 100
        limit-refresh-period: 1s
        timeout-duration: 500ms
  timelimiter:
    instances:
      payment-gateway:
        timeout-duration: 5s
        cancel-running-future: true
  thread-pool-bulkhead:
    instances:
      payment-gateway:
        max-thread-pool-size: 20
        core-thread-pool-size: 10
        queue-capacity: 50
        keep-alive-duration: 30s
```

---

## Summary

| Pattern | Library/Tool | Key Config |
|---------|-------------|-----------|
| Circuit Breaker | Resilience4j | failureRateThreshold, slidingWindowSize |
| Bulkhead (Thread Pool) | Resilience4j | maxThreadPoolSize, queueCapacity |
| Bulkhead (Semaphore) | Resilience4j | maxConcurrentCalls, maxWaitDuration |
| Retry | Resilience4j | maxAttempts, exponentialBackoff, jitter |
| Rate Limiter | Resilience4j | limitForPeriod, limitRefreshPeriod |
| Time Limiter | Resilience4j | timeoutDuration |
| Fallback | Custom code | Cache, queue, default value |
| Health Check | Spring Actuator | /actuator/health |
| Graceful Degradation | Feature flags | FeatureFlagService |
| Chaos Engineering | Chaos Monkey | @ChaosMonkey annotations |

## Next Part Preview

**Part 061: Advanced Spring Security** — Custom authentication providers, MFA with TOTP, brute force protection, audit trails, and a complete secure banking application example.
