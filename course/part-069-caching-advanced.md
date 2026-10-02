# Part 069: Advanced Caching Strategies

## Overview

A poor caching strategy can be worse than no cache at all — stale data, stampedes, and hot-key
overload are production killers. This part covers every major caching pattern with concrete
Spring Boot code, Redis data structures, Lua scripting for atomic operations, multi-level L1+L2
caching, and a complete flash-sale inventory system that survives thousands of concurrent buyers.

---

## 1. Project Setup

### Maven dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Caffeine L1 cache -->
    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2
          max-wait: 200ms

  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=300s

app:
  cache:
    product-ttl: 600        # seconds
    category-ttl: 3600
    flash-sale-ttl: 86400
    l1-ttl: 60              # local Caffeine TTL
    l2-ttl: 600             # Redis TTL
```

---

## 2. Redis Configuration

```java
package com.example.cache.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.*;
import org.springframework.data.redis.serializer.*;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        Jackson2JsonRedisSerializer<Object> jsonSerializer = jsonSerializer();
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);

        template.afterPropertiesSet();
        return template;
    }

    @Bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }

    private Jackson2JsonRedisSerializer<Object> jsonSerializer() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        return new Jackson2JsonRedisSerializer<>(mapper, Object.class);
    }
}
```

---

## 3. Caching Patterns Explained

### 3.1 Cache-Aside (Lazy Loading)

The application checks cache first; on a miss it fetches from DB and populates cache.

```java
package com.example.cache.service;

import com.example.cache.domain.Product;
import com.example.cache.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

@Slf4j
@Service
@RequiredArgsConstructor
public class CacheAsideProductService {

    private static final String KEY_PREFIX = "product:";

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    public Optional<Product> findById(Long id) {
        String key = KEY_PREFIX + id;

        // 1. Check cache
        Product cached = (Product) redisTemplate.opsForValue().get(key);
        if (cached != null) {
            log.debug("Cache HIT product:{}", id);
            return Optional.of(cached);
        }

        // 2. Cache MISS — load from database
        log.debug("Cache MISS product:{}", id);
        Optional<Product> product = productRepository.findById(id);

        // 3. Populate cache (only if found)
        product.ifPresent(p ->
            redisTemplate.opsForValue().set(key, p, Duration.ofMinutes(10))
        );

        return product;
    }

    public void evict(Long id) {
        redisTemplate.delete(KEY_PREFIX + id);
    }

    public void evictAll() {
        var keys = redisTemplate.keys(KEY_PREFIX + "*");
        if (keys != null && !keys.isEmpty()) redisTemplate.delete(keys);
    }
}
```

### 3.2 Read-Through Cache (Spring Cache abstraction)

Spring manages the cache transparently — `@Cacheable` handles lookup and population.

```java
package com.example.cache.service;

import com.example.cache.domain.Product;
import com.example.cache.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
@RequiredArgsConstructor
@CacheConfig(cacheNames = "products")
public class ReadThroughProductService {

    private final ProductRepository productRepository;

    @Cacheable(key = "#id", unless = "#result == null")
    @Transactional(readOnly = true)
    public Product findById(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    /**
     * Write-through: update DB and immediately update cache.
     */
    @CachePut(key = "#result.id")
    @Transactional
    public Product save(Product product) {
        return productRepository.save(product);
    }

    /**
     * Cache eviction on delete.
     */
    @CacheEvict(key = "#id")
    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    /**
     * Evict all entries in the 'products' cache.
     */
    @CacheEvict(allEntries = true)
    public void clearAll() {}
}
```

### 3.3 Write-Through Cache

Write goes to both cache and DB atomically (implemented here manually for clarity).

```java
package com.example.cache.service;

import com.example.cache.domain.Product;
import com.example.cache.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Duration;

@Service
@RequiredArgsConstructor
public class WriteThroughProductService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private String key(Long id) { return "product:" + id; }

    /**
     * Write-through: persist to DB then immediately write to cache.
     */
    @Transactional
    public Product save(Product product) {
        Product saved = productRepository.save(product);
        redisTemplate.opsForValue().set(key(saved.getId()), saved, Duration.ofMinutes(10));
        return saved;
    }

    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id);
        redisTemplate.delete(key(id));
    }
}
```

### 3.4 Write-Behind Cache (Async write to DB)

Cache is updated immediately; DB write is queued and executed asynchronously.

```java
package com.example.cache.service;

import com.example.cache.domain.Product;
import com.example.cache.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Set;

@Slf4j
@Service
@RequiredArgsConstructor
public class WriteBehindProductService {

    private static final String DIRTY_SET_KEY = "product:dirty";
    private static final String KEY_PREFIX    = "product:";

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    /**
     * Write to cache and mark as dirty.
     */
    public void update(Product product) {
        String key = KEY_PREFIX + product.getId();
        redisTemplate.opsForValue().set(key, product, Duration.ofHours(1));
        redisTemplate.opsForSet().add(DIRTY_SET_KEY, product.getId().toString());
    }

    /**
     * Flush dirty products to DB every 30 seconds.
     */
    @Scheduled(fixedDelay = 30_000)
    public void flushDirty() {
        Set<Object> dirtyIds = redisTemplate.opsForSet().members(DIRTY_SET_KEY);
        if (dirtyIds == null || dirtyIds.isEmpty()) return;

        log.info("Flushing {} dirty products to DB", dirtyIds.size());
        for (Object idObj : dirtyIds) {
            try {
                Long id = Long.parseLong(idObj.toString());
                Product p = (Product) redisTemplate.opsForValue().get(KEY_PREFIX + id);
                if (p != null) productRepository.save(p);
                redisTemplate.opsForSet().remove(DIRTY_SET_KEY, idObj);
            } catch (Exception e) {
                log.error("Failed to flush product id={}", idObj, e);
            }
        }
    }
}
```

---

## 4. Redis Data Structures for Caching

### 4.1 Hash — user session / product details

```java
package com.example.cache.redis;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class RedisHashService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void setProductHash(Long id, Map<String, Object> fields) {
        String key = "product:hash:" + id;
        redisTemplate.opsForHash().putAll(key, fields);
        redisTemplate.expire(key, Duration.ofMinutes(10));
    }

    public Object getField(Long id, String field) {
        return redisTemplate.opsForHash().get("product:hash:" + id, field);
    }

    public Map<Object, Object> getAll(Long id) {
        return redisTemplate.opsForHash().entries("product:hash:" + id);
    }

    public void incrementField(Long id, String field, long delta) {
        redisTemplate.opsForHash().increment("product:hash:" + id, field, delta);
    }
}
```

### 4.2 Sorted Set — leaderboard / recently viewed

```java
package com.example.cache.redis;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ZSetOperations;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
@RequiredArgsConstructor
public class LeaderboardService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String KEY = "product:view_count";

    public void recordView(String productId) {
        redisTemplate.opsForZSet().incrementScore(KEY, productId, 1.0);
    }

    /**
     * Top N most-viewed products.
     */
    public Set<ZSetOperations.TypedTuple<Object>> getTopProducts(int n) {
        return redisTemplate.opsForZSet()
            .reverseRangeWithScores(KEY, 0, n - 1);
    }

    /**
     * Rank of a specific product (0-indexed, lower = better).
     */
    public Long getRank(String productId) {
        return redisTemplate.opsForZSet().reverseRank(KEY, productId);
    }
}
```

### 4.3 List — recent items / queue

```java
package com.example.cache.redis;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;

@Service
@RequiredArgsConstructor
public class RecentlyViewedService {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final int MAX_RECENT = 20;

    public void recordView(Long userId, Long productId) {
        String key = "user:recent:" + userId;
        // Add to front
        redisTemplate.opsForList().leftPush(key, productId.toString());
        // Trim to max size
        redisTemplate.opsForList().trim(key, 0, MAX_RECENT - 1);
        // Reset TTL
        redisTemplate.expire(key, Duration.ofDays(30));
    }

    public List<Object> getRecentProducts(Long userId) {
        return redisTemplate.opsForList().range("user:recent:" + userId, 0, MAX_RECENT - 1);
    }
}
```

### 4.4 Set — unique visitors

```java
package com.example.cache.redis;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.LocalDate;

@Service
@RequiredArgsConstructor
public class UniqueVisitorService {

    private final RedisTemplate<String, Object> redisTemplate;

    public void recordVisit(Long productId, String visitorId) {
        String key = "product:visitors:" + productId + ":" + LocalDate.now();
        redisTemplate.opsForSet().add(key, visitorId);
        redisTemplate.expire(key, Duration.ofDays(7));
    }

    public long countUniqueVisitors(Long productId) {
        String key = "product:visitors:" + productId + ":" + LocalDate.now();
        Long size = redisTemplate.opsForSet().size(key);
        return size != null ? size : 0L;
    }
}
```

---

## 5. Redis Lua Scripting for Atomic Operations

Lua scripts run atomically on the Redis server — no race conditions.

```java
package com.example.cache.redis;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class LuaScriptService {

    private final StringRedisTemplate redisTemplate;

    // Atomic decrement only if value > 0 — returns new value or -1 if already 0
    private static final DefaultRedisScript<Long> DECREMENT_IF_POSITIVE;

    // Atomic compare-and-set — returns 1 if updated, 0 otherwise
    private static final DefaultRedisScript<Long> COMPARE_AND_SET;

    // Rate limiter: allows N requests per window; returns 1=allowed, 0=denied
    private static final DefaultRedisScript<Long> RATE_LIMIT;

    static {
        DECREMENT_IF_POSITIVE = new DefaultRedisScript<>(
            """
            local current = tonumber(redis.call('GET', KEYS[1]))
            if current == nil then return -2 end
            if current <= 0 then return -1 end
            return redis.call('DECRBY', KEYS[1], ARGV[1])
            """,
            Long.class
        );

        COMPARE_AND_SET = new DefaultRedisScript<>(
            """
            local current = redis.call('GET', KEYS[1])
            if current == ARGV[1] then
                redis.call('SET', KEYS[1], ARGV[2])
                return 1
            end
            return 0
            """,
            Long.class
        );

        RATE_LIMIT = new DefaultRedisScript<>(
            """
            local key     = KEYS[1]
            local limit   = tonumber(ARGV[1])
            local window  = tonumber(ARGV[2])  -- seconds
            local current = redis.call('INCR', key)
            if current == 1 then
                redis.call('EXPIRE', key, window)
            end
            if current > limit then
                return 0
            end
            return 1
            """,
            Long.class
        );
    }

    public LuaScriptService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * Atomically decrement inventory by `amount`. Returns new stock, or -1 if insufficient.
     */
    public long decrementInventory(String inventoryKey, long amount) {
        Long result = redisTemplate.execute(
            DECREMENT_IF_POSITIVE,
            List.of(inventoryKey),
            String.valueOf(amount)
        );
        return result != null ? result : -1L;
    }

    /**
     * Rate limiter: returns true if request is allowed.
     */
    public boolean isAllowed(String userId, String endpoint, int limit, int windowSeconds) {
        String key = "ratelimit:" + userId + ":" + endpoint;
        Long result = redisTemplate.execute(
            RATE_LIMIT,
            List.of(key),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );
        return Long.valueOf(1L).equals(result);
    }
}
```

---

## 6. Redis Pub/Sub

```java
package com.example.cache.pubsub;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.*;
import org.springframework.data.redis.listener.adapter.MessageListenerAdapter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;

@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer listenerContainer(
            org.springframework.data.redis.connection.RedisConnectionFactory factory,
            MessageListenerAdapter cacheInvalidationListener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(
            cacheInvalidationListener,
            new ChannelTopic("cache:invalidation")
        );
        return container;
    }

    @Bean
    public MessageListenerAdapter cacheInvalidationListener(CacheInvalidationSubscriber subscriber) {
        return new MessageListenerAdapter(subscriber, "onMessage");
    }
}
```

```java
package com.example.cache.pubsub;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class CacheInvalidationSubscriber {

    private final CacheManager cacheManager;

    public CacheInvalidationSubscriber(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    /** Called when a message arrives on cache:invalidation channel. */
    public void onMessage(String message) {
        log.info("Cache invalidation request received: {}", message);
        // Format: "products:123" → clear entry 123 in products cache
        String[] parts = message.split(":");
        if (parts.length == 2) {
            var cache = cacheManager.getCache(parts[0]);
            if (cache != null) cache.evict(parts[1]);
        }
    }
}
```

```java
package com.example.cache.pubsub;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CacheInvalidationPublisher {

    private final StringRedisTemplate redisTemplate;

    public void publish(String cacheName, String key) {
        redisTemplate.convertAndSend("cache:invalidation", cacheName + ":" + key);
    }
}
```

---

## 7. Redis Streams

```java
package com.example.cache.streams;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.stream.*;
import org.springframework.data.redis.core.StreamOperations;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderEventStreamService {

    private final StringRedisTemplate redisTemplate;

    private static final String STREAM_KEY    = "stream:order-events";
    private static final String CONSUMER_GROUP = "notification-service";

    /** Publish an event to the stream. */
    public RecordId publish(String eventType, String orderId, String payload) {
        StreamOperations<String, String, String> ops = redisTemplate.opsForStream();
        return ops.add(STREAM_KEY, Map.of(
            "eventType", eventType,
            "orderId",   orderId,
            "payload",   payload,
            "ts",        String.valueOf(System.currentTimeMillis())
        ));
    }

    /** Create consumer group (call once at startup). */
    public void createGroup() {
        try {
            redisTemplate.opsForStream()
                .createGroup(STREAM_KEY, ReadOffset.latest(), CONSUMER_GROUP);
        } catch (Exception e) {
            log.debug("Consumer group already exists: {}", e.getMessage());
        }
    }

    /** Read pending messages from the group. */
    public List<MapRecord<String, String, String>> read(String consumerName, int count) {
        StreamOperations<String, String, String> ops = redisTemplate.opsForStream();
        return ops.read(
            Consumer.from(CONSUMER_GROUP, consumerName),
            StreamReadOptions.empty().count(count),
            StreamOffset.create(STREAM_KEY, ReadOffset.lastConsumed())
        );
    }

    /** Acknowledge a processed message. */
    public void ack(String... recordIds) {
        redisTemplate.opsForStream().acknowledge(STREAM_KEY, CONSUMER_GROUP, recordIds);
    }
}
```

---

## 8. Cache Stampede Prevention

When a cached value expires, thousands of concurrent requests can hammer the DB simultaneously.
Solutions: probabilistic early expiration, locking, and background refresh.

```java
package com.example.cache.stampede;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

@Slf4j
@Service
@RequiredArgsConstructor
public class StampedeProtectedCacheService {

    private final RedisTemplate<String, Object> redisTemplate;

    // Try to acquire a "rebuild lock" for this key; returns 1=got lock, 0=already locked
    private static final DefaultRedisScript<Long> TRY_LOCK = new DefaultRedisScript<>(
        "return redis.call('SET', KEYS[1], '1', 'NX', 'PX', ARGV[1])",
        Long.class
    );

    /**
     * Cache-aside with mutex lock to prevent stampede.
     * Only one thread rebuilds; others wait up to lockTtl.
     */
    public <T> T getOrCompute(String key, Supplier<T> loader,
                              Duration cacheTtl, Duration lockTtl,
                              Class<T> type) {

        // 1. Try cache
        Object cached = redisTemplate.opsForValue().get(key);
        if (cached != null) return type.cast(cached);

        // 2. Try to acquire lock
        String lockKey = key + ":lock";
        Long locked = redisTemplate.execute(
            TRY_LOCK, List.of(lockKey),
            String.valueOf(lockTtl.toMillis())
        );

        if (Long.valueOf(1L).equals(locked)) {
            // Got lock — we rebuild
            try {
                // Double-check after acquiring lock
                cached = redisTemplate.opsForValue().get(key);
                if (cached != null) return type.cast(cached);

                T value = loader.get();
                if (value != null)
                    redisTemplate.opsForValue().set(key, value, cacheTtl);
                return value;
            } finally {
                redisTemplate.delete(lockKey);
            }
        } else {
            // Someone else is rebuilding — wait briefly and re-read
            log.debug("Waiting for cache rebuild of key={}", key);
            try { Thread.sleep(50); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            cached = redisTemplate.opsForValue().get(key);
            if (cached != null) return type.cast(cached);
            // Still nothing — fall through to direct load (last resort)
            return loader.get();
        }
    }

    /**
     * Probabilistic early expiration (XFetch algorithm).
     * Starts refreshing slightly before expiry with increasing probability.
     * Eliminates stampedes without explicit locking.
     */
    public <T> T getWithEarlyExpiration(String key, Supplier<T> loader,
                                        Duration ttl, Class<T> type) {
        long now = System.currentTimeMillis();
        Long remaining = redisTemplate.getExpire(key, TimeUnit.MILLISECONDS);
        Object value = redisTemplate.opsForValue().get(key);

        // If absent or TTL unknown → load immediately
        if (value == null || remaining == null || remaining < 0) {
            T loaded = loader.get();
            redisTemplate.opsForValue().set(key, loaded, ttl);
            return loaded;
        }

        // Probabilistic early recompute:
        // P(recompute) = -β * log(remaining/ttlMs) where β = 1 (tunable)
        double beta = 1.0;
        double rand = Math.random();
        double prob = -beta * Math.log((double) remaining / ttl.toMillis());
        if (rand < prob) {
            log.debug("XFetch: probabilistic early recompute for key={}", key);
            T loaded = loader.get();
            redisTemplate.opsForValue().set(key, loaded, ttl);
            return loaded;
        }

        return type.cast(value);
    }
}
```

---

## 9. Hot Key Problem and Solutions

```java
package com.example.cache.hotkey;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;

/**
 * Hot key mitigation strategies:
 * 1. Local replica cache (read from many local copies instead of one Redis key).
 * 2. Key sharding (fan out writes to N copies, round-robin reads).
 */
@Service
@RequiredArgsConstructor
public class HotKeyMitigationService {

    private static final int SHARDS = 10;

    private final RedisTemplate<String, Object> redisTemplate;

    // Each JVM maintains a local Caffeine copy — see MultiLevelCacheService
    private final java.util.concurrent.ConcurrentHashMap<String, Object> localReplica =
        new java.util.concurrent.ConcurrentHashMap<>();

    /** Write to all shards. */
    public void putSharded(String baseKey, Object value, Duration ttl) {
        for (int i = 0; i < SHARDS; i++) {
            redisTemplate.opsForValue().set(shardKey(baseKey, i), value, ttl);
        }
    }

    /** Read from a random shard. */
    public Optional<Object> getSharded(String baseKey) {
        int shard = (int) (Math.random() * SHARDS);
        Object value = redisTemplate.opsForValue().get(shardKey(baseKey, shard));
        return Optional.ofNullable(value);
    }

    private String shardKey(String baseKey, int shard) {
        return baseKey + ":shard:" + shard;
    }
}
```

---

## 10. Multi-Level Cache (L1 Caffeine + L2 Redis)

```java
package com.example.cache.multilevel;

import com.example.cache.pubsub.CacheInvalidationPublisher;
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.Optional;
import java.util.concurrent.TimeUnit;
import java.util.function.Supplier;

@Slf4j
@Service
public class MultiLevelCacheService {

    private final Cache<String, Object> l1;             // Caffeine (in-process)
    private final RedisTemplate<String, Object> l2;      // Redis (shared)
    private final CacheInvalidationPublisher publisher;

    private final Duration l1Ttl;
    private final Duration l2Ttl;

    public MultiLevelCacheService(
            RedisTemplate<String, Object> redisTemplate,
            CacheInvalidationPublisher publisher,
            @Value("${app.cache.l1-ttl:60}") int l1Seconds,
            @Value("${app.cache.l2-ttl:600}") int l2Seconds) {

        this.l2        = redisTemplate;
        this.publisher = publisher;
        this.l1Ttl     = Duration.ofSeconds(l1Seconds);
        this.l2Ttl     = Duration.ofSeconds(l2Seconds);

        this.l1 = Caffeine.newBuilder()
            .maximumSize(50_000)
            .expireAfterWrite(l1Seconds, TimeUnit.SECONDS)
            .recordStats()
            .build();
    }

    /**
     * Read from L1 → L2 → loader, populating each level on miss.
     */
    @SuppressWarnings("unchecked")
    public <T> T get(String key, Supplier<T> loader, Class<T> type) {
        // L1 hit
        Object l1Value = l1.getIfPresent(key);
        if (l1Value != null) {
            log.trace("L1 HIT key={}", key);
            return type.cast(l1Value);
        }

        // L2 hit
        Object l2Value = l2.opsForValue().get(key);
        if (l2Value != null) {
            log.trace("L2 HIT key={}", key);
            l1.put(key, l2Value);      // backfill L1
            return type.cast(l2Value);
        }

        // Miss → load from source
        log.debug("Cache MISS key={}", key);
        T value = loader.get();
        if (value != null) {
            l1.put(key, value);
            l2.opsForValue().set(key, value, l2Ttl);
        }
        return value;
    }

    /**
     * Invalidate from both levels and broadcast to other JVM instances.
     */
    public void evict(String cacheName, String key) {
        l1.invalidate(key);
        l2.delete(key);
        publisher.publish(cacheName, key);  // tells other nodes to evict L1
    }

    public void put(String key, Object value) {
        l1.put(key, value);
        l2.opsForValue().set(key, value, l2Ttl);
    }

    /** Print Caffeine stats (hit rate, eviction count, etc.) */
    public String l1Stats() {
        return l1.stats().toString();
    }
}
```

---

## 11. Cache Warming

```java
package com.example.cache.warming;

import com.example.cache.domain.Product;
import com.example.cache.repository.ProductRepository;
import com.example.cache.multilevel.MultiLevelCacheService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.data.domain.*;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class CacheWarmer {

    private final ProductRepository productRepository;
    private final MultiLevelCacheService cacheService;

    /**
     * Warm up the top 1000 most-viewed products after application starts.
     * Runs asynchronously so it doesn't delay startup.
     */
    @EventListener(ApplicationReadyEvent.class)
    @Async
    public void warmProductCache() {
        log.info("Starting cache warming...");
        int page = 0;
        int size = 100;
        int total = 0;

        while (total < 1000) {
            List<Product> products = productRepository.findTopByViewCountDesc(
                PageRequest.of(page++, size)).getContent();
            if (products.isEmpty()) break;

            for (Product p : products) {
                cacheService.put("product:" + p.getId(), p);
            }
            total += products.size();
        }
        log.info("Cache warming complete. Warmed {} products.", total);
    }
}
```

---

## 12. Real Example — Flash Sale Inventory with Redis Atomic Operations

```java
package com.example.cache.flashsale;

import jakarta.persistence.*;
import lombok.*;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "flash_sales")
@Data @NoArgsConstructor @AllArgsConstructor @Builder
public class FlashSale {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long productId;
    private int totalStock;
    private int reservedStock;
    private BigDecimal salePrice;
    private Instant startAt;
    private Instant endAt;
    private boolean active;
}
```

```java
package com.example.cache.flashsale;

import com.example.cache.redis.LuaScriptService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Duration;

@Slf4j
@Service
@RequiredArgsConstructor
public class FlashSaleService {

    private final FlashSaleRepository flashSaleRepository;
    private final FlashSaleOrderRepository orderRepository;
    private final StringRedisTemplate redisTemplate;
    private final LuaScriptService luaService;

    private String stockKey(Long saleId)  { return "flash:stock:"  + saleId; }
    private String soldKey(Long saleId)   { return "flash:sold:"   + saleId; }
    private String userKey(Long saleId, Long userId) {
        return "flash:user:" + saleId + ":" + userId;
    }

    /**
     * Load flash-sale stock into Redis when sale starts.
     */
    @Transactional
    public void initializeSale(Long saleId) {
        FlashSale sale = flashSaleRepository.findById(saleId).orElseThrow();
        int available = sale.getTotalStock() - sale.getReservedStock();

        redisTemplate.opsForValue().set(
            stockKey(saleId),
            String.valueOf(available),
            Duration.ofSeconds(86400)
        );
        redisTemplate.opsForValue().set(soldKey(saleId), "0", Duration.ofSeconds(86400));
        log.info("Flash sale {} initialized with {} units", saleId, available);
    }

    /**
     * Attempt to purchase. Returns purchase result.
     *
     * Uses Lua script for atomic: check user already bought + decrement stock.
     */
    public PurchaseResult purchase(Long saleId, Long userId, int quantity) {
        // 1. Check if user already purchased (per-user deduplification)
        String userBought = redisTemplate.opsForValue().get(userKey(saleId, userId));
        if (userBought != null) {
            return PurchaseResult.alreadyPurchased();
        }

        // 2. Atomically decrement stock
        long newStock = luaService.decrementInventory(stockKey(saleId), quantity);

        if (newStock == -1) {
            return PurchaseResult.soldOut();
        }
        if (newStock == -2) {
            // Key doesn't exist — sale not initialized
            return PurchaseResult.saleNotActive();
        }

        // 3. Mark user as having purchased (prevent double purchase)
        redisTemplate.opsForValue().set(
            userKey(saleId, userId),
            String.valueOf(quantity),
            Duration.ofDays(1)
        );

        // 4. Increment sold counter
        redisTemplate.opsForValue().increment(soldKey(saleId), quantity);

        // 5. Persist order asynchronously
        persistOrderAsync(saleId, userId, quantity);

        log.info("Flash sale {} purchase: userId={} qty={} remainingStock={}",
                 saleId, userId, quantity, newStock);

        return PurchaseResult.success(newStock);
    }

    @org.springframework.scheduling.annotation.Async
    void persistOrderAsync(Long saleId, Long userId, int quantity) {
        try {
            FlashSale sale = flashSaleRepository.findById(saleId).orElseThrow();
            FlashSaleOrder order = FlashSaleOrder.builder()
                .saleId(saleId)
                .userId(userId)
                .quantity(quantity)
                .unitPrice(sale.getSalePrice())
                .status("PENDING")
                .build();
            orderRepository.save(order);
        } catch (Exception e) {
            log.error("Failed to persist flash sale order saleId={} userId={}",
                      saleId, userId, e);
        }
    }

    public long getRemainingStock(Long saleId) {
        String value = redisTemplate.opsForValue().get(stockKey(saleId));
        return value != null ? Long.parseLong(value) : 0L;
    }

    public long getSoldCount(Long saleId) {
        String value = redisTemplate.opsForValue().get(soldKey(saleId));
        return value != null ? Long.parseLong(value) : 0L;
    }
}
```

```java
package com.example.cache.flashsale;

import lombok.*;

@Data
@AllArgsConstructor(staticName = "of")
public class PurchaseResult {
    public enum Status { SUCCESS, SOLD_OUT, ALREADY_PURCHASED, SALE_NOT_ACTIVE }

    private final Status status;
    private final long remainingStock;
    private final String message;

    public boolean isSuccess() { return status == Status.SUCCESS; }

    static PurchaseResult success(long remaining) {
        return new PurchaseResult(Status.SUCCESS, remaining, "Purchase successful");
    }
    static PurchaseResult soldOut() {
        return new PurchaseResult(Status.SOLD_OUT, 0, "Sorry, sold out!");
    }
    static PurchaseResult alreadyPurchased() {
        return new PurchaseResult(Status.ALREADY_PURCHASED, -1, "You have already purchased this item");
    }
    static PurchaseResult saleNotActive() {
        return new PurchaseResult(Status.SALE_NOT_ACTIVE, -1, "Flash sale is not active");
    }
}
```

### Flash sale controller

```java
package com.example.cache.flashsale;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/flash-sales")
@RequiredArgsConstructor
public class FlashSaleController {

    private final FlashSaleService flashSaleService;

    @PostMapping("/{saleId}/purchase")
    public ResponseEntity<PurchaseResult> purchase(
            @PathVariable Long saleId,
            @RequestParam(defaultValue = "1") int quantity,
            @AuthenticationPrincipal Long userId) {

        PurchaseResult result = flashSaleService.purchase(saleId, userId, quantity);
        return result.isSuccess()
            ? ResponseEntity.ok(result)
            : ResponseEntity.status(409).body(result);
    }

    @GetMapping("/{saleId}/stock")
    public ResponseEntity<StockInfo> getStock(@PathVariable Long saleId) {
        long remaining = flashSaleService.getRemainingStock(saleId);
        long sold      = flashSaleService.getSoldCount(saleId);
        return ResponseEntity.ok(new StockInfo(remaining, sold));
    }

    record StockInfo(long remaining, long sold) {}
}
```

---

## 13. Cache Eviction Policies

### Redis config (redis.conf)

```
maxmemory 2gb
maxmemory-policy allkeys-lru
# Options:
#   noeviction          - return error when memory full
#   allkeys-lru         - evict least recently used keys across all keys
#   volatile-lru        - evict LRU keys that have TTL set
#   allkeys-lfu         - evict least frequently used (Redis 4.0+)
#   volatile-lfu        - evict LFU keys with TTL
#   allkeys-random      - evict random keys
#   volatile-random     - evict random keys with TTL
#   volatile-ttl        - evict keys with shortest TTL first
```

### Caffeine eviction in Spring Cache

```java
package com.example.cache.config;

import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class CaffeineCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(
            Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(5, TimeUnit.MINUTES)
                .expireAfterAccess(2, TimeUnit.MINUTES)
                .recordStats()
        );
        return manager;
    }
}
```

---

## 14. Cache Consistency with Database

```java
package com.example.cache.consistency;

import com.example.cache.domain.Product;
import com.example.cache.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.*;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
@CacheConfig(cacheNames = "products")
public class ConsistentProductService {

    private final ProductRepository productRepository;

    @Cacheable(key = "#id")
    @Transactional(readOnly = true)
    public Product findById(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    /**
     * Write pattern: update DB first, then evict (never update cache in write path).
     * Cache will be lazily repopulated on next read.
     * This avoids stale writes from concurrent updates.
     */
    @CacheEvict(key = "#product.id")
    @Transactional
    public Product update(Product product) {
        return productRepository.save(product);
    }

    /**
     * Refresh stale products every 10 minutes — proactive consistency maintenance.
     */
    @Scheduled(fixedDelay = 600_000)
    @CacheEvict(allEntries = true)
    public void refreshAll() {
        log.info("Scheduled full cache refresh triggered");
        // Next reads will repopulate from DB
    }
}
```

---

## 15. Testing

```java
package com.example.cache.flashsale;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class FlashSaleServiceTest {

    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired FlashSaleService service;

    @BeforeEach
    void init() {
        service.initializeSale(1L); // assumes FlashSale id=1 in test DB with stock=10
    }

    @Test
    void concurrentPurchasesDoNotOversell() throws InterruptedException {
        int threads = 50;
        int stockLimit = 10;
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        AtomicInteger successes = new AtomicInteger();
        AtomicInteger rejections = new AtomicInteger();
        CountDownLatch latch = new CountDownLatch(threads);

        for (long userId = 1; userId <= threads; userId++) {
            final long uid = userId;
            pool.submit(() -> {
                try {
                    PurchaseResult result = service.purchase(1L, uid, 1);
                    if (result.isSuccess()) successes.incrementAndGet();
                    else rejections.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(10, TimeUnit.SECONDS);
        pool.shutdown();

        assertThat(successes.get()).isLessThanOrEqualTo(stockLimit);
        assertThat(successes.get() + rejections.get()).isEqualTo(threads);
        assertThat(service.getRemainingStock(1L)).isGreaterThanOrEqualTo(0);
    }
}
```

---

## Summary Table

| Strategy | Use Case | Implementation |
|---|---|---|
| Cache-aside | Lazy load, simple reads | `CacheAsideProductService` |
| Read-through | Spring-managed transparent | `@Cacheable` / `ReadThroughProductService` |
| Write-through | Keep cache always fresh | `@CachePut` / `WriteThroughProductService` |
| Write-behind | High-write throughput | `WriteBehindProductService` |
| Stampede prevention | High-concurrency reads | `StampedeProtectedCacheService` |
| Hot-key sharding | Extreme traffic on one key | `HotKeyMitigationService` |
| Multi-level (L1+L2) | Minimize network round-trips | `MultiLevelCacheService` |
| Cache warming | Reduce cold-start latency | `CacheWarmer` |
| Atomic inventory | Flash-sale, stock operations | `LuaScriptService` + `FlashSaleService` |
| Pub/Sub invalidation | Cross-node L1 sync | `CacheInvalidationPublisher/Subscriber` |
| Redis Streams | Event streaming | `OrderEventStreamService` |
| Eviction policy | Redis memory management | `maxmemory-policy allkeys-lru` |

---

## Next Part Preview

**Part 070: Advanced API Gateway Patterns** — build a Spring Cloud Gateway BFF that aggregates
multiple microservices, transforms requests/responses, splits traffic for A/B tests and canary
releases, enforces per-tier rate limits, translates REST to gRPC, and runs circuit breakers at
the gateway level.
