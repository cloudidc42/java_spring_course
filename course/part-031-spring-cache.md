# Part 031: Spring Cache Abstraction

## เนื้อหาในส่วนนี้
- Spring Cache Overview
- @Cacheable, @CachePut, @CacheEvict, @Caching
- Cache Providers: Simple, Caffeine, Redis, EhCache
- Cache Configuration
- Conditional Caching
- Cache Statistics
- Distributed Caching with Redis
- Cache Best Practices
- Complete Example

---

## 1. Spring Cache Overview

Spring Cache Abstraction ช่วยให้ใช้ caching ได้โดยไม่ต้องผูกกับ cache provider

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Caffeine (high-performance in-memory cache) -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>

<!-- Redis cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```java
// Enable caching
@SpringBootApplication
@EnableCaching
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 2. Core Cache Annotations

```java
import org.springframework.cache.annotation.*;
import org.springframework.stereotype.Service;
import java.util.*;
import java.time.LocalDateTime;

@Service
public class ProductCacheService {
    
    private final Map<Long, Product> database = new HashMap<>(Map.of(
        1L, new Product(1L, "Laptop", 999.99, "Electronics"),
        2L, new Product(2L, "Mouse", 29.99, "Electronics"),
        3L, new Product(3L, "Desk", 199.99, "Furniture")
    ));
    
    // === @Cacheable - cache the result ===
    // key = SpEL expression (default = all method args)
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        System.out.println("DB QUERY: Finding product " + id);
        simulateSlowQuery();
        return database.get(id);
    }
    
    // Conditional caching
    @Cacheable(value = "products", key = "#id", condition = "#id > 0")
    public Product findByIdConditional(Long id) {
        return database.get(id);
    }
    
    // Unless - don't cache if result meets condition
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product findByIdNoNullCache(Long id) {
        return database.get(id);
    }
    
    // Cache with complex key
    @Cacheable(value = "products-by-category", key = "#category + ':' + #page + ':' + #size")
    public List<Product> findByCategory(String category, int page, int size) {
        System.out.println("DB QUERY: Finding products in category " + category);
        return database.values().stream()
            .filter(p -> p.category().equals(category))
            .skip((long) page * size)
            .limit(size)
            .toList();
    }
    
    // === @CachePut - always update cache (no cache-aside) ===
    @CachePut(value = "products", key = "#result.id()")
    public Product save(Product product) {
        System.out.println("DB SAVE: Saving product " + product.id());
        database.put(product.id(), product);
        return product;
    }
    
    @CachePut(value = "products", key = "#id")
    public Product update(Long id, String newName, double newPrice) {
        Product updated = new Product(id, newName, newPrice, database.get(id).category());
        database.put(id, updated);
        return updated;
    }
    
    // === @CacheEvict - remove from cache ===
    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) {
        System.out.println("DB DELETE: Deleting product " + id);
        database.remove(id);
    }
    
    // Evict all entries in a cache
    @CacheEvict(value = "products", allEntries = true)
    public void clearProductCache() {
        System.out.println("Cache cleared");
    }
    
    // Evict before method (beforeInvocation = true)
    @CacheEvict(value = "products-by-category", allEntries = true, beforeInvocation = true)
    public void refreshCategories() {
        System.out.println("Refreshing all category caches");
    }
    
    // === @Caching - multiple cache operations ===
    @Caching(
        evict = {
            @CacheEvict(value = "products", key = "#product.id()"),
            @CacheEvict(value = "products-by-category", allEntries = true)
        },
        put = @CachePut(value = "products-updated", key = "#product.id()")
    )
    public Product updateProduct(Product product) {
        database.put(product.id(), product);
        return product;
    }
    
    private void simulateSlowQuery() {
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}

record Product(Long id, String name, double price, String category) {}
```

---

## 3. Cache Configuration - Caffeine

```java
import org.springframework.cache.*;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.*;
import com.github.benmanes.caffeine.cache.*;
import java.util.concurrent.TimeUnit;

@Configuration
@EnableCaching
public class CacheConfig {
    
    // === Simple Caffeine configuration ===
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(caffeine());
        return manager;
    }
    
    @Bean
    public Caffeine<Object, Object> caffeine() {
        return Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .maximumSize(1000)
            .recordStats();
    }
    
    // === Per-cache configuration ===
    @Bean
    public CacheManager customCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager() {
            @Override
            protected com.github.benmanes.caffeine.cache.Cache<Object, Object> 
                    createNativeCaffeineCache(String name) {
                return switch (name) {
                    case "products" -> Caffeine.newBuilder()
                        .expireAfterWrite(10, TimeUnit.MINUTES)
                        .maximumSize(500)
                        .recordStats()
                        .build();
                    case "users" -> Caffeine.newBuilder()
                        .expireAfterAccess(30, TimeUnit.MINUTES)
                        .maximumSize(1000)
                        .recordStats()
                        .build();
                    case "categories" -> Caffeine.newBuilder()
                        .expireAfterWrite(1, TimeUnit.HOURS)
                        .maximumSize(100)
                        .build();
                    default -> Caffeine.newBuilder()
                        .expireAfterWrite(5, TimeUnit.MINUTES)
                        .maximumSize(200)
                        .build();
                };
            }
        };
        manager.setCacheNames(java.util.List.of("products", "users", "categories"));
        return manager;
    }
}
```

```yaml
# application.yml - Caffeine via properties
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=500,expireAfterWrite=10m,recordStats
    cache-names: products, users, categories
```

---

## 4. Redis Cache Configuration

```java
import org.springframework.context.annotation.*;
import org.springframework.data.redis.cache.*;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.*;
import java.time.Duration;
import java.util.*;

@Configuration
@EnableCaching
public class RedisCacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        
        // Default config
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));
        
        // Per-cache configs
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        
        cacheConfigs.put("products", defaultConfig
            .entryTtl(Duration.ofMinutes(30)));
        
        cacheConfigs.put("users", defaultConfig
            .entryTtl(Duration.ofHours(1)));
        
        cacheConfigs.put("sessions", defaultConfig
            .entryTtl(Duration.ofHours(24)));
        
        cacheConfigs.put("short-lived", defaultConfig
            .entryTtl(Duration.ofSeconds(30)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

```yaml
# application.yml - Redis configuration
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          min-idle: 2
          max-idle: 4
          max-wait: 1000ms
  
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes (ms)
      cache-null-values: false
      key-prefix: "myapp:"
      use-key-prefix: true
```

---

## 5. Cache Statistics and Monitoring

```java
import org.springframework.cache.*;
import org.springframework.cache.caffeine.CaffeineCache;
import org.springframework.stereotype.Component;
import com.github.benmanes.caffeine.cache.stats.CacheStats;
import java.util.*;

@Component
public class CacheStatsMonitor {
    
    private final CacheManager cacheManager;
    
    CacheStatsMonitor(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    public Map<String, Map<String, Object>> getAllStats() {
        Map<String, Map<String, Object>> result = new LinkedHashMap<>();
        
        cacheManager.getCacheNames().forEach(cacheName -> {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache instanceof CaffeineCache caffeineCache) {
                CacheStats stats = caffeineCache.getNativeCache().stats();
                
                Map<String, Object> statsMap = new LinkedHashMap<>();
                statsMap.put("hitCount", stats.hitCount());
                statsMap.put("missCount", stats.missCount());
                statsMap.put("loadCount", stats.loadCount());
                statsMap.put("hitRate", String.format("%.2f%%", stats.hitRate() * 100));
                statsMap.put("missRate", String.format("%.2f%%", stats.missRate() * 100));
                statsMap.put("evictionCount", stats.evictionCount());
                statsMap.put("estimatedSize", caffeineCache.getNativeCache().estimatedSize());
                statsMap.put("averageLoadPenalty", 
                    String.format("%.2fms", stats.averageLoadPenalty() / 1_000_000));
                
                result.put(cacheName, statsMap);
            }
        });
        
        return result;
    }
    
    public void printReport() {
        System.out.println("\n=== Cache Statistics ===");
        getAllStats().forEach((cache, stats) -> {
            System.out.println("Cache: " + cache);
            stats.forEach((k, v) -> System.out.println("  " + k + ": " + v));
        });
    }
}

// Cache Stats Actuator Endpoint
import org.springframework.boot.actuate.endpoint.annotation.*;

@org.springframework.stereotype.Component
@Endpoint(id = "cache-stats")
class CacheStatsEndpoint2 {
    
    private final CacheStatsMonitor monitor;
    
    CacheStatsEndpoint2(CacheStatsMonitor monitor) {
        this.monitor = monitor;
    }
    
    @ReadOperation
    public Map<String, Map<String, Object>> cacheStats() {
        return monitor.getAllStats();
    }
    
    @WriteOperation
    public void clearCache(@Selector String cacheName) {
        // Clear specific cache
    }
}
```

---

## 6. Advanced Caching Patterns

```java
import org.springframework.cache.*;
import org.springframework.stereotype.Service;
import java.util.*;
import java.util.concurrent.CompletableFuture;

@Service
public class AdvancedCachingService {
    
    private final CacheManager cacheManager;
    
    AdvancedCachingService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    // === Programmatic cache access ===
    public Product getProductProgrammatic(Long id) {
        Cache cache = cacheManager.getCache("products");
        
        // Try cache first
        Cache.ValueWrapper cached = cache.get(id);
        if (cached != null) {
            System.out.println("Cache hit for: " + id);
            return (Product) cached.get();
        }
        
        // Load from DB
        System.out.println("Cache miss, loading: " + id);
        Product product = loadFromDatabase(id);
        
        // Put in cache
        if (product != null) {
            cache.put(id, product);
        }
        
        return product;
    }
    
    // get with loader
    public Product getOrLoad(Long id) {
        Cache cache = cacheManager.getCache("products");
        return cache.get(id, () -> {
            System.out.println("Loading from DB: " + id);
            return loadFromDatabase(id);
        });
    }
    
    // === Bulk cache loading (cache warming) ===
    @org.springframework.boot.context.event.EventListener(
        org.springframework.context.event.ContextRefreshedEvent.class)
    public void warmUpCaches() {
        System.out.println("Warming up caches...");
        Cache cache = cacheManager.getCache("products");
        
        // Load frequently accessed data into cache
        List<Product> hotProducts = loadHotProducts();
        hotProducts.forEach(p -> cache.put(p.id(), p));
        
        System.out.println("Warmed up " + hotProducts.size() + " products");
    }
    
    // === Cache-aside pattern with TTL awareness ===
    @Cacheable(value = "time-aware", key = "#id",
               unless = "#result == null")
    public TimestampedValue<Product> getWithTimestamp(Long id) {
        Product product = loadFromDatabase(id);
        return new TimestampedValue<>(product, System.currentTimeMillis());
    }
    
    private Product loadFromDatabase(Long id) {
        // Simulate DB call
        return new Product(id, "Product " + id, 99.99 * id, "Electronics");
    }
    
    private List<Product> loadHotProducts() {
        return List.of(
            new Product(1L, "Laptop", 999.99, "Electronics"),
            new Product(2L, "Mouse", 29.99, "Electronics")
        );
    }
}

record TimestampedValue<T>(T value, long timestamp) {
    boolean isExpired(long ttlMs) {
        return System.currentTimeMillis() - timestamp > ttlMs;
    }
}
```

---

## 7. Complete E-Commerce Cache Example

```java
import org.springframework.cache.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;
import java.math.BigDecimal;

// Domain
record ProductDetail(Long id, String name, BigDecimal price, 
                     String description, String category, int stock) {}
record CategoryInfo(Long id, String name, int productCount) {}
record SearchResult(List<ProductDetail> products, int total, int page) {}

// === Repository ===
@Repository
class ProductCatalogRepository {
    
    private static final Map<Long, ProductDetail> db = new HashMap<>(Map.of(
        1L, new ProductDetail(1L, "MacBook Pro", new BigDecimal("2499.99"), 
            "Latest MacBook", "Laptops", 50),
        2L, new ProductDetail(2L, "iPhone 15", new BigDecimal("999.99"), 
            "Latest iPhone", "Phones", 100),
        3L, new ProductDetail(3L, "AirPods Pro", new BigDecimal("249.99"), 
            "Noise cancelling", "Audio", 200)
    ));
    
    ProductDetail findById(Long id) { return db.get(id); }
    
    List<ProductDetail> findByCategory(String category) {
        return db.values().stream()
            .filter(p -> p.category().equals(category))
            .toList();
    }
    
    List<ProductDetail> search(String query) {
        String lower = query.toLowerCase();
        return db.values().stream()
            .filter(p -> p.name().toLowerCase().contains(lower) || 
                        p.description().toLowerCase().contains(lower))
            .toList();
    }
    
    void save(ProductDetail product) { db.put(product.id(), product); }
    void delete(Long id) { db.remove(id); }
    long count() { return db.size(); }
}

// === Service with Caching ===
@Service
@CacheConfig(cacheNames = "products")  // Default cache for this class
class ProductCatalogService {
    
    private final ProductCatalogRepository repository;
    
    ProductCatalogService(ProductCatalogRepository repository) {
        this.repository = repository;
    }
    
    @Cacheable(key = "#id")
    public ProductDetail getProduct(Long id) {
        System.out.println("  [DB] Loading product: " + id);
        return repository.findById(id);
    }
    
    @Cacheable(cacheNames = "products-by-category", key = "#category")
    public List<ProductDetail> getByCategory(String category) {
        System.out.println("  [DB] Loading category: " + category);
        return repository.findByCategory(category);
    }
    
    @Cacheable(cacheNames = "search-results", 
               key = "#query.toLowerCase() + ':' + #page",
               unless = "#result.products().isEmpty()")
    public SearchResult search(String query, int page) {
        System.out.println("  [DB] Searching: " + query + " (page " + page + ")");
        List<ProductDetail> results = repository.search(query);
        int size = 10;
        int from = page * size;
        int to = Math.min(from + size, results.size());
        
        List<ProductDetail> paged = from < results.size() 
            ? results.subList(from, to) : List.of();
        return new SearchResult(paged, results.size(), page);
    }
    
    @CachePut(key = "#result.id()")
    @CacheEvict(cacheNames = "products-by-category", allEntries = true)
    public ProductDetail createProduct(ProductDetail product) {
        System.out.println("  [DB] Creating product: " + product.name());
        repository.save(product);
        return product;
    }
    
    @Caching(
        put = @CachePut(key = "#product.id()"),
        evict = {
            @CacheEvict(cacheNames = "products-by-category", allEntries = true),
            @CacheEvict(cacheNames = "search-results", allEntries = true)
        }
    )
    public ProductDetail updateProduct(ProductDetail product) {
        System.out.println("  [DB] Updating product: " + product.id());
        repository.save(product);
        return product;
    }
    
    @Caching(evict = {
        @CacheEvict(key = "#id"),
        @CacheEvict(cacheNames = "products-by-category", allEntries = true),
        @CacheEvict(cacheNames = "search-results", allEntries = true)
    })
    public void deleteProduct(Long id) {
        System.out.println("  [DB] Deleting product: " + id);
        repository.delete(id);
    }
    
    @Cacheable(cacheNames = "product-count")
    public long getProductCount() {
        return repository.count();
    }
    
    @CacheEvict(cacheNames = {"products", "products-by-category", "search-results", "product-count"}, 
                allEntries = true)
    public void clearAllCaches() {
        System.out.println("All caches cleared");
    }
}

// === Demo ===
public class CacheDemo {
    public static void main(String[] args) {
        var context = new org.springframework.context.annotation
            .AnnotationConfigApplicationContext();
        context.register(CacheConfig.class);
        context.scan(""); // scan all
        context.refresh();
        
        ProductCatalogService service = context.getBean(ProductCatalogService.class);
        
        System.out.println("=== First access (DB queries) ===");
        service.getProduct(1L);
        service.getProduct(2L);
        service.getByCategory("Laptops");
        
        System.out.println("\n=== Second access (from cache) ===");
        service.getProduct(1L);   // Cache hit
        service.getProduct(2L);   // Cache hit
        service.getByCategory("Laptops"); // Cache hit
        
        System.out.println("\n=== Update product (cache updated) ===");
        ProductDetail updated = new ProductDetail(1L, "MacBook Pro M3", 
            new BigDecimal("2699.99"), "M3 chip", "Laptops", 45);
        service.updateProduct(updated);
        
        System.out.println("\n=== Get updated product (from cache) ===");
        ProductDetail cached = service.getProduct(1L);
        System.out.println("Got: " + cached.name());
        
        context.close();
    }
}

@org.springframework.context.annotation.Configuration
@org.springframework.context.annotation.EnableCaching
@org.springframework.context.annotation.ComponentScan
class CacheConfig2 {
    @org.springframework.context.annotation.Bean
    public org.springframework.cache.CacheManager cacheManager() {
        return new org.springframework.cache.concurrent.ConcurrentMapCacheManager(
            "products", "products-by-category", "search-results", "product-count");
    }
}
```

---

## สรุป Part 031

| Annotation | Description |
|-----------|-------------|
| @Cacheable | Cache method result |
| @CachePut | Always update cache |
| @CacheEvict | Remove from cache |
| @Caching | Combine multiple operations |
| @CacheConfig | Default cache config for class |
| @EnableCaching | Enable Spring cache |

### Cache Providers
- **Simple**: ConcurrentHashMap, for tests
- **Caffeine**: High performance, in-memory, production grade
- **Redis**: Distributed cache, shared across instances
- **EhCache**: Enterprise, supports disk overflow

---

**Part 032:** Spring WebFlux - Reactive Programming
