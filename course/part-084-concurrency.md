# Part 084: Java Concurrency in Spring Applications

## Overview

Concurrency is one of the most challenging aspects of production Java systems. This part covers thread safety in Spring beans, advanced CompletableFuture patterns, Virtual Threads (Java 21), Structured Concurrency, and builds a real parallel product enrichment pipeline.

---

## 1. Thread Safety in Spring Beans

Spring beans are **singletons by default**. Singleton beans shared across request threads must be stateless or use thread-safe data structures.

```java
// UNSAFE: mutable field in singleton bean
@Service  // singleton
public class UnsafeCounterService {
    private int count = 0;  // DANGER: shared mutable state

    public int increment() {
        return ++count;  // NOT thread-safe: read-modify-write is not atomic
    }
}

// SAFE: use AtomicInteger
@Service
public class SafeCounterService {
    private final AtomicInteger count = new AtomicInteger(0);

    public int increment() {
        return count.incrementAndGet();  // atomic
    }

    public int get() {
        return count.get();
    }
}

// SAFE: use ConcurrentHashMap
@Service
public class RequestTrackerService {
    private final ConcurrentHashMap<String, AtomicInteger> requestCounts = new ConcurrentHashMap<>();

    public void record(String endpoint) {
        requestCounts.computeIfAbsent(endpoint, k -> new AtomicInteger(0))
                     .incrementAndGet();
    }

    public int getCount(String endpoint) {
        AtomicInteger counter = requestCounts.get(endpoint);
        return counter == null ? 0 : counter.get();
    }

    public Map<String, Integer> getSnapshot() {
        Map<String, Integer> snapshot = new HashMap<>();
        requestCounts.forEach((k, v) -> snapshot.put(k, v.get()));
        return Collections.unmodifiableMap(snapshot);
    }
}

// SAFE: stateless – all state in method parameters
@Service
public class StatelessPricingService {

    public BigDecimal calculateTotal(List<OrderItem> items, DiscountPolicy policy) {
        BigDecimal subtotal = items.stream()
                .map(item -> item.getUnitPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        return policy.apply(subtotal);
    }
}
```

---

## 2. synchronized and ReentrantLock

```java
@Service
public class InventoryService {

    private final Map<Long, Integer> stock = new HashMap<>();
    // One lock per product to maximize concurrency
    private final ConcurrentHashMap<Long, ReentrantLock> locks = new ConcurrentHashMap<>();

    public boolean reserve(Long productId, int quantity) {
        ReentrantLock lock = locks.computeIfAbsent(productId, k -> new ReentrantLock());
        lock.lock();
        try {
            int current = stock.getOrDefault(productId, 0);
            if (current < quantity) {
                return false;
            }
            stock.put(productId, current - quantity);
            return true;
        } finally {
            lock.unlock();  // ALWAYS release in finally
        }
    }

    // Try-lock with timeout to prevent deadlock
    public boolean tryReserve(Long productId, int quantity, long timeoutMs) throws InterruptedException {
        ReentrantLock lock = locks.computeIfAbsent(productId, k -> new ReentrantLock());

        if (!lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS)) {
            throw new InventoryLockTimeoutException("Could not acquire lock for product " + productId);
        }
        try {
            int current = stock.getOrDefault(productId, 0);
            if (current < quantity) {
                return false;
            }
            stock.put(productId, current - quantity);
            return true;
        } finally {
            lock.unlock();
        }
    }

    // ReadWriteLock: multiple readers, exclusive writer
    private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private final Map<Long, ProductInfo> productCache = new HashMap<>();

    public ProductInfo getProduct(Long id) {
        readWriteLock.readLock().lock();
        try {
            return productCache.get(id);
        } finally {
            readWriteLock.readLock().unlock();
        }
    }

    public void updateProduct(Long id, ProductInfo info) {
        readWriteLock.writeLock().lock();
        try {
            productCache.put(id, info);
        } finally {
            readWriteLock.writeLock().unlock();
        }
    }
}
```

---

## 3. java.util.concurrent Utilities

### 3.1 CountDownLatch

```java
// Wait for all tasks to finish before proceeding
@Service
public class BulkImportService {

    public ImportResult importProducts(List<ProductCsvRow> rows) throws InterruptedException {
        int workerCount = Math.min(rows.size(), Runtime.getRuntime().availableProcessors());
        CountDownLatch latch = new CountDownLatch(rows.size());
        List<ImportError> errors = new CopyOnWriteArrayList<>();

        ExecutorService executor = Executors.newFixedThreadPool(workerCount);

        for (ProductCsvRow row : rows) {
            executor.submit(() -> {
                try {
                    importRow(row);
                } catch (Exception e) {
                    errors.add(new ImportError(row.getLineNumber(), e.getMessage()));
                } finally {
                    latch.countDown();  // always decrement, even on error
                }
            });
        }

        boolean completed = latch.await(5, TimeUnit.MINUTES);
        executor.shutdown();

        if (!completed) {
            throw new ImportTimeoutException("Import did not complete within 5 minutes");
        }

        return new ImportResult(rows.size() - errors.size(), errors);
    }

    private void importRow(ProductCsvRow row) {
        // ... import logic
    }
}
```

### 3.2 CyclicBarrier

```java
// Synchronize phases of a parallel computation
@Service
public class PriceRecalculationService {

    private static final int THREAD_COUNT = 4;

    public void recalculateAllPrices(List<Product> products) throws Exception {
        List<List<Product>> partitions = partition(products, THREAD_COUNT);
        CyclicBarrier barrier = new CyclicBarrier(THREAD_COUNT, () ->
                log.info("All threads completed phase, moving to next phase"));

        List<Thread> threads = new ArrayList<>();
        for (List<Product> partition : partitions) {
            Thread t = Thread.ofVirtual().start(() -> {
                try {
                    // Phase 1: fetch cost data
                    fetchCosts(partition);
                    barrier.await();  // wait for all threads to finish phase 1

                    // Phase 2: apply margin rules
                    applyMargins(partition);
                    barrier.await();  // wait for all threads to finish phase 2

                    // Phase 3: persist
                    persistPrices(partition);
                } catch (Exception e) {
                    Thread.currentThread().interrupt();
                }
            });
            threads.add(t);
        }

        for (Thread t : threads) {
            t.join();
        }
    }

    private <T> List<List<T>> partition(List<T> list, int size) {
        List<List<T>> result = new ArrayList<>();
        for (int i = 0; i < list.size(); i += size) {
            result.add(list.subList(i, Math.min(i + size, list.size())));
        }
        return result;
    }

    private void fetchCosts(List<Product> products) { /* ... */ }
    private void applyMargins(List<Product> products) { /* ... */ }
    private void persistPrices(List<Product> products) { /* ... */ }
}
```

### 3.3 Semaphore

```java
// Rate-limit calls to an external service
@Service
public class ExternalCatalogClient {

    // Limit to 10 concurrent requests to external API
    private final Semaphore semaphore = new Semaphore(10, true);  // fair

    public ProductDetails fetch(Long productId) {
        try {
            semaphore.acquire();
            try {
                return callExternalApi(productId);
            } finally {
                semaphore.release();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new ServiceUnavailableException("Interrupted while waiting for API slot");
        }
    }

    public Optional<ProductDetails> tryFetch(Long productId) {
        if (!semaphore.tryAcquire()) {
            log.warn("API rate limit reached, skipping product {}", productId);
            return Optional.empty();
        }
        try {
            return Optional.of(callExternalApi(productId));
        } finally {
            semaphore.release();
        }
    }

    private ProductDetails callExternalApi(Long productId) {
        // HTTP call to external service
        return restClient.get()
                .uri("/products/{id}", productId)
                .retrieve()
                .body(ProductDetails.class);
    }
}
```

---

## 4. CompletableFuture Advanced Patterns

```java
@Service
@Slf4j
public class ProductEnrichmentService {

    private final PricingClient pricingClient;
    private final InventoryClient inventoryClient;
    private final RatingsClient ratingsClient;
    private final Executor enrichmentExecutor;

    // Parallel fetch from 3 services, combine results
    public CompletableFuture<EnrichedProduct> enrich(Long productId) {
        CompletableFuture<PricingInfo> pricingFuture =
                CompletableFuture.supplyAsync(() -> pricingClient.fetch(productId), enrichmentExecutor);

        CompletableFuture<InventoryInfo> inventoryFuture =
                CompletableFuture.supplyAsync(() -> inventoryClient.fetch(productId), enrichmentExecutor);

        CompletableFuture<RatingsInfo> ratingsFuture =
                CompletableFuture.supplyAsync(() -> ratingsClient.fetch(productId), enrichmentExecutor);

        return CompletableFuture
                .allOf(pricingFuture, inventoryFuture, ratingsFuture)
                .thenApply(v -> EnrichedProduct.builder()
                        .productId(productId)
                        .pricing(pricingFuture.join())
                        .inventory(inventoryFuture.join())
                        .ratings(ratingsFuture.join())
                        .build());
    }

    // With timeout and fallback
    public CompletableFuture<EnrichedProduct> enrichWithFallback(Long productId) {
        CompletableFuture<PricingInfo> pricingFuture =
                CompletableFuture.supplyAsync(() -> pricingClient.fetch(productId), enrichmentExecutor)
                        .orTimeout(2, TimeUnit.SECONDS)
                        .exceptionally(ex -> PricingInfo.defaultPricing());

        CompletableFuture<InventoryInfo> inventoryFuture =
                CompletableFuture.supplyAsync(() -> inventoryClient.fetch(productId), enrichmentExecutor)
                        .completeOnTimeout(InventoryInfo.unknown(), 2, TimeUnit.SECONDS);

        CompletableFuture<RatingsInfo> ratingsFuture =
                CompletableFuture.supplyAsync(() -> ratingsClient.fetch(productId), enrichmentExecutor)
                        .orTimeout(2, TimeUnit.SECONDS)
                        .exceptionally(ex -> RatingsInfo.noRatings());

        return CompletableFuture
                .allOf(pricingFuture, inventoryFuture, ratingsFuture)
                .thenApply(v -> EnrichedProduct.builder()
                        .productId(productId)
                        .pricing(pricingFuture.join())
                        .inventory(inventoryFuture.join())
                        .ratings(ratingsFuture.join())
                        .build());
    }

    // anyOf: use whichever result arrives first (e.g., primary + backup service)
    public CompletableFuture<PricingInfo> fetchPricingWithBackup(Long productId) {
        CompletableFuture<PricingInfo> primary =
                CompletableFuture.supplyAsync(() -> pricingClient.fetch(productId));

        CompletableFuture<PricingInfo> backup =
                CompletableFuture.supplyAsync(() -> pricingClient.fetchFromBackup(productId));

        return CompletableFuture.anyOf(primary, backup)
                .thenApply(result -> (PricingInfo) result);
    }

    // Fan-out: enrich multiple products in parallel
    public List<EnrichedProduct> enrichBatch(List<Long> productIds) {
        List<CompletableFuture<EnrichedProduct>> futures = productIds.stream()
                .map(this::enrichWithFallback)
                .collect(Collectors.toList());

        return futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
    }

    // Error handling in pipelines
    public CompletableFuture<OrderConfirmation> processOrder(OrderRequest request) {
        return CompletableFuture
                .supplyAsync(() -> validateOrder(request))
                .thenCompose(validOrder -> reserveInventory(validOrder))
                .thenCompose(reservation -> chargePayment(reservation))
                .thenCompose(payment -> sendConfirmation(payment))
                .handle((confirmation, ex) -> {
                    if (ex != null) {
                        log.error("Order processing failed", ex);
                        compensate(request);  // rollback
                        throw new OrderProcessingException("Order failed: " + ex.getMessage(), ex);
                    }
                    return confirmation;
                });
    }

    // Chaining with thenCompose (flatMap for CompletableFuture)
    public CompletableFuture<String> getCustomerEmail(Long orderId) {
        return CompletableFuture
                .supplyAsync(() -> orderRepository.findById(orderId).orElseThrow())
                .thenCompose(order ->
                        CompletableFuture.supplyAsync(() ->
                                customerRepository.findById(order.getCustomerId()).orElseThrow().getEmail()));
    }
}
```

---

## 5. Fork/Join Framework

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

// Divide-and-conquer for large data processing
public class ParallelProductIndexer extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 100;
    private final List<Product> products;
    private final SearchIndexClient indexClient;

    public ParallelProductIndexer(List<Product> products, SearchIndexClient indexClient) {
        this.products = products;
        this.indexClient = indexClient;
    }

    @Override
    protected Integer compute() {
        if (products.size() <= THRESHOLD) {
            return indexProducts(products);
        }

        int mid = products.size() / 2;
        ParallelProductIndexer left = new ParallelProductIndexer(
                products.subList(0, mid), indexClient);
        ParallelProductIndexer right = new ParallelProductIndexer(
                products.subList(mid, products.size()), indexClient);

        left.fork();   // async execution in pool
        int rightResult = right.compute();   // run right in current thread
        int leftResult = left.join();         // wait for left

        return leftResult + rightResult;
    }

    private int indexProducts(List<Product> batch) {
        indexClient.bulkIndex(batch);
        return batch.size();
    }
}

@Service
public class SearchIndexService {

    private final ForkJoinPool forkJoinPool = new ForkJoinPool(
            Runtime.getRuntime().availableProcessors());

    public int reindexAll(List<Product> products) {
        ParallelProductIndexer task = new ParallelProductIndexer(products, searchIndexClient);
        return forkJoinPool.invoke(task);
    }
}
```

---

## 6. ExecutorService Configuration

```java
// Configuration for custom thread pools
@Configuration
public class ExecutorConfiguration {

    @Bean(name = "enrichmentExecutor")
    public Executor enrichmentExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(500);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("enrich-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    @Bean(name = "reportExecutor")
    public Executor reportExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("report-");
        executor.initialize();
        return executor;
    }

    // Virtual thread executor (Java 21+)
    @Bean(name = "virtualThreadExecutor")
    public ExecutorService virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

---

## 7. @Async with Custom Thread Pools

```java
@Service
@Slf4j
public class NotificationService {

    @Async("enrichmentExecutor")
    public CompletableFuture<Void> sendOrderNotification(Long orderId) {
        log.info("Sending notification for order {} on thread {}", orderId, Thread.currentThread().getName());
        // ... send email, push notification, etc.
        return CompletableFuture.completedFuture(null);
    }

    @Async("reportExecutor")
    public CompletableFuture<ReportResult> generateReport(ReportRequest request) {
        log.info("Generating report on thread {}", Thread.currentThread().getName());
        ReportResult result = reportGenerator.generate(request);
        return CompletableFuture.completedFuture(result);
    }

    // Return void for fire-and-forget
    @Async
    public void recordAuditEvent(AuditEvent event) {
        auditRepository.save(event);
    }
}

// Enable async and configure default executor
@Configuration
@EnableAsync
public class AsyncConfiguration implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-default-");
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> {
            log.error("Uncaught async exception in {}: {}", method.getName(), throwable.getMessage(), throwable);
            // could publish to error tracking, Slack, etc.
        };
    }
}
```

---

## 8. ThreadLocal and RequestContextHolder

```java
// ThreadLocal for per-request data
public class RequestContext {

    private static final ThreadLocal<RequestContextData> context = new ThreadLocal<>();

    public static void set(RequestContextData data) {
        context.set(data);
    }

    public static RequestContextData get() {
        RequestContextData data = context.get();
        if (data == null) {
            throw new IllegalStateException("No request context available");
        }
        return data;
    }

    public static void clear() {
        context.remove();
    }

    public record RequestContextData(String requestId, String userId, String tenantId) {}
}

// Interceptor that populates ThreadLocal
@Component
public class RequestContextInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String requestId = Optional.ofNullable(request.getHeader("X-Request-ID"))
                .orElse(UUID.randomUUID().toString());
        String userId = extractUserId(request);
        String tenantId = request.getHeader("X-Tenant-ID");

        RequestContext.set(new RequestContext.RequestContextData(requestId, userId, tenantId));
        response.setHeader("X-Request-ID", requestId);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        RequestContext.clear();  // CRITICAL: prevent memory leaks
    }

    private String extractUserId(HttpServletRequest request) {
        // extract from JWT or session
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null ? auth.getName() : "anonymous";
    }
}

// Spring's RequestContextHolder usage
@Service
public class TenantAwareService {

    public String getCurrentTenantId() {
        ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
        return attrs.getRequest().getHeader("X-Tenant-ID");
    }
}

// InheritableThreadLocal – passes context to child threads
public class InheritableRequestContext {
    private static final InheritableThreadLocal<String> tenantId = new InheritableThreadLocal<>();

    public static void setTenantId(String id) { tenantId.set(id); }
    public static String getTenantId() { return tenantId.get(); }
    public static void clear() { tenantId.remove(); }
}
```

---

## 9. Virtual Threads in Spring Boot 3

```java
// application.yml for Spring Boot 3.2+
// spring:
//   threads:
//     virtual:
//       enabled: true  # enables virtual threads for Tomcat/Jetty/Undertow

@Configuration
public class VirtualThreadConfiguration {

    // Virtual thread executor for @Async
    @Bean(name = "virtualThreadExecutor")
    public ExecutorService virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    // Custom factory for scheduled tasks
    @Bean
    public TaskScheduler virtualThreadTaskScheduler() {
        SimpleAsyncTaskScheduler scheduler = new SimpleAsyncTaskScheduler();
        scheduler.setVirtualThreads(true);
        return scheduler;
    }
}

@Service
public class VirtualThreadService {

    @Async("virtualThreadExecutor")
    public CompletableFuture<String> fetchDataAsync(String url) {
        // Virtual threads handle blocking I/O efficiently
        // Blocking here does NOT block a OS thread
        String data = httpClient.get(url);
        return CompletableFuture.completedFuture(data);
    }

    public List<String> fetchAll(List<String> urls) throws Exception {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = urls.stream()
                    .map(url -> executor.submit(() -> httpClient.get(url)))
                    .collect(Collectors.toList());

            List<String> results = new ArrayList<>();
            for (Future<String> future : futures) {
                results.add(future.get(5, TimeUnit.SECONDS));
            }
            return results;
        }
        // executor.close() waits for all futures and closes automatically
    }
}
```

---

## 10. Structured Concurrency (Java 21)

```java
import java.util.concurrent.StructuredTaskScope;

// Structured concurrency: all subtasks scoped to parent
@Service
public class StructuredProductFetcher {

    public EnrichedProduct fetchEnriched(Long productId) throws Exception {
        // ShutdownOnFailure: if any task fails, cancel all others
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            StructuredTaskScope.Subtask<PricingInfo> pricingTask =
                    scope.fork(() -> pricingClient.fetch(productId));

            StructuredTaskScope.Subtask<InventoryInfo> inventoryTask =
                    scope.fork(() -> inventoryClient.fetch(productId));

            StructuredTaskScope.Subtask<RatingsInfo> ratingsTask =
                    scope.fork(() -> ratingsClient.fetch(productId));

            scope.join();           // wait for all tasks
            scope.throwIfFailed();  // propagate first exception

            return EnrichedProduct.builder()
                    .productId(productId)
                    .pricing(pricingTask.get())
                    .inventory(inventoryTask.get())
                    .ratings(ratingsTask.get())
                    .build();
        }
    }

    // ShutdownOnSuccess: return first successful result
    public PricingInfo fetchFromAnySource(Long productId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<PricingInfo>()) {
            scope.fork(() -> pricingClient.fetchFromPrimary(productId));
            scope.fork(() -> pricingClient.fetchFromBackup(productId));
            scope.fork(() -> pricingClient.fetchFromCache(productId));

            scope.join();
            return scope.result();  // first non-null result
        }
    }
}
```

---

## 11. Race Condition Detection and Prevention

```java
// Pattern: optimistic locking with JPA version field
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int stock;

    @Version  // incremented on each UPDATE; stale reads throw OptimisticLockException
    private Long version;
}

@Service
public class StockService {

    @Retryable(retryFor = OptimisticLockException.class, maxAttempts = 3,
               backoff = @Backoff(delay = 100, multiplier = 2))
    @Transactional
    public void decrementStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));

        if (product.getStock() < quantity) {
            throw new InsufficientStockException(productId, quantity, product.getStock());
        }

        product.setStock(product.getStock() - quantity);
        productRepository.save(product);  // throws OptimisticLockException if version mismatch
    }

    // Pessimistic lock for high-contention scenarios
    @Transactional
    public void decrementStockPessimistic(Long productId, int quantity) {
        Product product = productRepository.findByIdWithLock(productId)  // SELECT FOR UPDATE
                .orElseThrow(() -> new ProductNotFoundException(productId));

        if (product.getStock() < quantity) {
            throw new InsufficientStockException(productId, quantity, product.getStock());
        }

        product.setStock(product.getStock() - quantity);
    }
}

// Repository with pessimistic lock
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}
```

---

## 12. Real Example: Parallel Product Enrichment Pipeline

```java
// ===== Domain objects =====

public record ProductEnrichmentRequest(Long productId, List<EnrichmentType> types) {}

public enum EnrichmentType { PRICING, INVENTORY, RATINGS, IMAGES, DESCRIPTION }

@Builder
public record EnrichedProduct(
    Long productId,
    PricingInfo pricing,
    InventoryInfo inventory,
    RatingsInfo ratings,
    List<String> imageUrls,
    String description,
    Duration processingTime
) {}

// ===== Pipeline Implementation =====

@Service
@Slf4j
public class ProductEnrichmentPipeline {

    private final Map<EnrichmentType, EnrichmentStep> steps;
    private final MeterRegistry meterRegistry;
    private final ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();

    public ProductEnrichmentPipeline(
            PricingEnrichmentStep pricingStep,
            InventoryEnrichmentStep inventoryStep,
            RatingsEnrichmentStep ratingsStep,
            ImagesEnrichmentStep imagesStep,
            DescriptionEnrichmentStep descriptionStep,
            MeterRegistry meterRegistry) {
        this.steps = Map.of(
                EnrichmentType.PRICING, pricingStep,
                EnrichmentType.INVENTORY, inventoryStep,
                EnrichmentType.RATINGS, ratingsStep,
                EnrichmentType.IMAGES, imagesStep,
                EnrichmentType.DESCRIPTION, descriptionStep
        );
        this.meterRegistry = meterRegistry;
    }

    public EnrichedProduct enrich(ProductEnrichmentRequest request) {
        Instant start = Instant.now();
        Map<EnrichmentType, Object> results = new ConcurrentHashMap<>();
        List<CompletableFuture<Void>> futures = new ArrayList<>();

        for (EnrichmentType type : request.types()) {
            EnrichmentStep step = steps.get(type);
            if (step == null) {
                log.warn("No step registered for enrichment type {}", type);
                continue;
            }

            CompletableFuture<Void> future = CompletableFuture
                    .supplyAsync(() -> {
                        Timer.Sample sample = Timer.start(meterRegistry);
                        try {
                            Object result = step.enrich(request.productId());
                            results.put(type, result);
                            sample.stop(meterRegistry.timer("enrichment.step",
                                    "type", type.name(), "status", "success"));
                            return result;
                        } catch (Exception e) {
                            log.warn("Enrichment step {} failed for product {}: {}",
                                    type, request.productId(), e.getMessage());
                            results.put(type, step.defaultValue());
                            sample.stop(meterRegistry.timer("enrichment.step",
                                    "type", type.name(), "status", "failed"));
                            return step.defaultValue();
                        }
                    }, virtualExecutor)
                    .orTimeout(3, TimeUnit.SECONDS)
                    .exceptionally(ex -> {
                        log.error("Enrichment step {} timed out for product {}", type, request.productId());
                        results.put(type, step.defaultValue());
                        return null;
                    })
                    .thenAccept(v -> {});  // convert to Void

            futures.add(future);
        }

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

        return EnrichedProduct.builder()
                .productId(request.productId())
                .pricing((PricingInfo) results.getOrDefault(EnrichmentType.PRICING, PricingInfo.defaultPricing()))
                .inventory((InventoryInfo) results.getOrDefault(EnrichmentType.INVENTORY, InventoryInfo.unknown()))
                .ratings((RatingsInfo) results.getOrDefault(EnrichmentType.RATINGS, RatingsInfo.noRatings()))
                .imageUrls((List<String>) results.getOrDefault(EnrichmentType.IMAGES, List.of()))
                .description((String) results.getOrDefault(EnrichmentType.DESCRIPTION, ""))
                .processingTime(Duration.between(start, Instant.now()))
                .build();
    }

    // Batch enrichment with controlled concurrency
    public List<EnrichedProduct> enrichBatch(List<ProductEnrichmentRequest> requests, int maxConcurrency) {
        Semaphore semaphore = new Semaphore(maxConcurrency);

        List<CompletableFuture<EnrichedProduct>> futures = requests.stream()
                .map(request -> CompletableFuture.supplyAsync(() -> {
                    try {
                        semaphore.acquire();
                        try {
                            return enrich(request);
                        } finally {
                            semaphore.release();
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException("Interrupted during enrichment", e);
                    }
                }, virtualExecutor))
                .collect(Collectors.toList());

        return futures.stream()
                .map(f -> f.exceptionally(ex -> {
                    log.error("Batch enrichment failed", ex);
                    return null;
                }))
                .map(CompletableFuture::join)
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
    }
}

// ===== Enrichment Step Contract =====

public interface EnrichmentStep<T> {
    T enrich(Long productId);
    T defaultValue();
}

@Component
public class PricingEnrichmentStep implements EnrichmentStep<PricingInfo> {

    private final PricingClient pricingClient;
    private final Cache<Long, PricingInfo> cache;

    public PricingEnrichmentStep(PricingClient pricingClient) {
        this.pricingClient = pricingClient;
        this.cache = Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(Duration.ofMinutes(5))
                .build();
    }

    @Override
    public PricingInfo enrich(Long productId) {
        return cache.get(productId, id -> pricingClient.fetchCurrentPricing(id));
    }

    @Override
    public PricingInfo defaultValue() {
        return PricingInfo.defaultPricing();
    }
}

// ===== REST Controller =====

@RestController
@RequestMapping("/api/products")
public class ProductEnrichmentController {

    private final ProductEnrichmentPipeline pipeline;

    @GetMapping("/{id}/enriched")
    public ResponseEntity<EnrichedProduct> getEnriched(
            @PathVariable Long id,
            @RequestParam(defaultValue = "PRICING,INVENTORY,RATINGS") Set<EnrichmentType> types) {

        EnrichedProduct result = pipeline.enrich(new ProductEnrichmentRequest(id, new ArrayList<>(types)));
        return ResponseEntity.ok()
                .header("X-Processing-Time-Ms", String.valueOf(result.processingTime().toMillis()))
                .body(result);
    }

    @PostMapping("/batch/enrich")
    public ResponseEntity<List<EnrichedProduct>> enrichBatch(
            @RequestBody List<Long> productIds,
            @RequestParam(defaultValue = "PRICING,INVENTORY") Set<EnrichmentType> types) {

        List<ProductEnrichmentRequest> requests = productIds.stream()
                .map(id -> new ProductEnrichmentRequest(id, new ArrayList<>(types)))
                .collect(Collectors.toList());

        List<EnrichedProduct> results = pipeline.enrichBatch(requests, 20);
        return ResponseEntity.ok(results);
    }
}
```

---

## 13. Monitoring Thread Pools

```java
@Configuration
public class ThreadPoolMonitoring {

    @Bean
    public MeterBinder enrichmentExecutorMetrics(@Qualifier("enrichmentExecutor") Executor executor) {
        if (executor instanceof ThreadPoolTaskExecutor taskExecutor) {
            return registry -> {
                Gauge.builder("thread.pool.active", taskExecutor, e -> e.getActiveCount())
                        .tag("pool", "enrichment")
                        .description("Active threads in enrichment pool")
                        .register(registry);

                Gauge.builder("thread.pool.size", taskExecutor, e -> e.getPoolSize())
                        .tag("pool", "enrichment")
                        .register(registry);

                Gauge.builder("thread.pool.queue.size", taskExecutor,
                        e -> e.getThreadPoolExecutor().getQueue().size())
                        .tag("pool", "enrichment")
                        .register(registry);
            };
        }
        return registry -> {};
    }
}
```

---

## Summary

| Tool | Best For |
|---|---|
| `AtomicInteger/Long` | Simple counters and numeric state |
| `ConcurrentHashMap` | Concurrent key-value lookups |
| `ReentrantLock` | Fine-grained locking with fairness |
| `ReadWriteLock` | Many readers, few writers |
| `CountDownLatch` | Wait for N tasks to complete |
| `CyclicBarrier` | Synchronize phases across threads |
| `Semaphore` | Limit concurrency / rate limiting |
| `CompletableFuture` | Async pipelines, fan-out/fan-in |
| `ForkJoinPool` | Recursive divide-and-conquer |
| `@Async` | Fire-and-forget Spring beans |
| `Virtual Threads` | High-concurrency I/O bound work |
| `StructuredTaskScope` | Scoped parallel subtask execution |
| `@Version` (JPA) | Optimistic locking for stale writes |

## Next Part Preview

**Part 085** covers Spring Authorization Server — building a full OAuth2 + OIDC authorization server with PKCE, token customization, and SSO for microservices.
