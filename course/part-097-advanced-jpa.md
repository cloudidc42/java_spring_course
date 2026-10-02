# Part 097: Advanced JPA and Hibernate

## Introduction

Beyond basic CRUD, Hibernate offers powerful features for performance-critical applications: second-level caching, batch processing, inheritance strategies, and statistical monitoring. This part covers the advanced techniques needed for production-grade data access layers.

---

## Project Setup

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate.orm</groupId>
        <artifactId>hibernate-jcache</artifactId>
    </dependency>
    <dependency>
        <groupId>org.ehcache</groupId>
        <artifactId>ehcache</artifactId>
        <classifier>jakarta</classifier>
    </dependency>
    <!-- For Redis second-level cache -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-hibernate-6</artifactId>
        <version>3.25.0</version>
    </dependency>
</dependencies>
```

---

## Second-Level Cache with Ehcache

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheCacheRegionFactory
          javax:
            cache:
              provider: org.ehcache.jsr107.EhcacheCachingProvider
              uri: classpath:ehcache.xml
        generate_statistics: true
        session:
          events:
            log:
              LOG_QUERIES_SLOWER_THAN_MS: 100
    show-sql: false
    format-sql: true
    open-in-view: false  # NEVER enable this in production
```

```xml
<!-- src/main/resources/ehcache.xml -->
<config xmlns:xsi='http://www.w3.org/2001/XMLSchema-instance'
        xmlns='http://www.ehcache.org/v3'>

    <!-- Default cache for entities not explicitly configured -->
    <cache-template name="default">
        <expiry>
            <ttl unit="minutes">10</ttl>
        </expiry>
        <heap unit="entries">1000</heap>
    </cache-template>

    <!-- Product cache - longer TTL, larger size -->
    <cache alias="com.example.entity.Product" uses-template="default">
        <expiry>
            <ttl unit="hours">1</ttl>
        </expiry>
        <heap unit="entries">5000</heap>
        <offheap unit="MB">100</offheap>
    </cache>

    <!-- Category cache - static-ish data, long TTL -->
    <cache alias="com.example.entity.Category" uses-template="default">
        <expiry>
            <ttl unit="hours">24</ttl>
        </expiry>
        <heap unit="entries">100</heap>
    </cache>

    <!-- Query cache region -->
    <cache alias="org.hibernate.cache.internal.StandardQueryCache" uses-template="default">
        <expiry>
            <ttl unit="minutes">5</ttl>
        </expiry>
        <heap unit="entries">200</heap>
    </cache>

    <!-- Timestamps cache (required when query cache is enabled) -->
    <cache alias="org.hibernate.cache.spi.UpdateTimestampsCache" uses-template="default">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>
        <heap unit="entries">100</heap>
    </cache>
</config>
```

### Cacheable Entity

```java
// src/main/java/com/example/entity/Product.java
package com.example.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "products")
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // Second-level cache
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String sku;

    @Column(nullable = false)
    private String name;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    // Cache the collection too
    @OneToMany(mappedBy = "product", fetch = FetchType.LAZY)
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    @Builder.Default
    private List<ProductVariant> variants = new ArrayList<>();

    private boolean active;
    private int stockQuantity;
}
```

```java
// src/main/java/com/example/entity/Category.java
package com.example.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Table(name = "categories")
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)  // Rarely changes
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    private String description;
    private String iconUrl;
    private Integer sortOrder;
}
```

### Query Cache

```java
// src/main/java/com/example/repository/ProductRepository.java
package com.example.repository;

import com.example.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.QueryHints;
import org.springframework.data.jpa.repository.Query;

import jakarta.persistence.QueryHint;
import java.util.List;

import static org.hibernate.jpa.HibernateHints.HINT_CACHEABLE;
import static org.hibernate.jpa.HibernateHints.HINT_CACHE_REGION;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // Cached query - result set cached for 5 minutes
    @QueryHints({
        @QueryHint(name = HINT_CACHEABLE, value = "true"),
        @QueryHint(name = HINT_CACHE_REGION, value = "productsByCategory")
    })
    @Query("SELECT p FROM Product p WHERE p.category.name = :categoryName AND p.active = true")
    List<Product> findActiveByCategory(String categoryName);

    @QueryHints(@QueryHint(name = HINT_CACHEABLE, value = "true"))
    List<Product> findBySku(String sku);
}
```

---

## Batch Inserts and Updates

```java
// application.yml additions for batching
// spring:
//   jpa:
//     properties:
//       hibernate:
//         jdbc:
//           batch_size: 50
//           batch_versioned_data: true
//         order_inserts: true
//         order_updates: true

// src/main/java/com/example/service/ProductImportService.java
package com.example.service;

import com.example.entity.Product;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductImportService {

    private final EntityManager entityManager;

    private static final int BATCH_SIZE = 50;

    /**
     * Efficiently insert thousands of products using batch inserts.
     * Clear the persistence context every BATCH_SIZE to prevent OutOfMemoryError.
     */
    @Transactional
    public void importProducts(List<Product> products) {
        log.info("Starting batch import of {} products", products.size());

        for (int i = 0; i < products.size(); i++) {
            entityManager.persist(products.get(i));

            // Flush and clear every BATCH_SIZE items
            if (i > 0 && i % BATCH_SIZE == 0) {
                entityManager.flush();
                entityManager.clear();
                log.debug("Flushed batch at index {}", i);
            }
        }

        // Final flush for remaining items
        entityManager.flush();
        entityManager.clear();
        log.info("Batch import completed");
    }

    /**
     * Batch update: update all prices in a category by a factor
     */
    @Transactional
    public int updatePricesByCategory(String categoryName, double factor) {
        // JPQL bulk UPDATE - bypasses second-level cache!
        int updated = entityManager.createQuery(
            "UPDATE Product p SET p.price = p.price * :factor " +
            "WHERE p.category.name = :categoryName AND p.active = true"
        )
        .setParameter("factor", factor)
        .setParameter("categoryName", categoryName)
        .executeUpdate();

        // Evict cache after bulk update
        entityManager.getEntityManagerFactory().getCache()
            .evictAll();  // Or evict specific region

        log.info("Updated {} product prices in category {}", updated, categoryName);
        return updated;
    }

    /**
     * Bulk delete with JPQL
     */
    @Transactional
    public int deleteDiscontinuedProducts() {
        return entityManager.createQuery(
            "DELETE FROM Product p WHERE p.active = false AND p.stockQuantity = 0"
        ).executeUpdate();
    }
}
```

---

## Native Queries with Result Mapping

```java
// src/main/java/com/example/dto/ProductSalesSummary.java
package com.example.dto;

/**
 * Projection interface for native query results
 */
public interface ProductSalesSummary {
    String getProductName();
    String getCategoryName();
    Long getTotalOrders();
    Double getTotalRevenue();
    Double getAverageOrderValue();
}
```

```java
// src/main/java/com/example/repository/SalesRepository.java
package com.example.repository;

import com.example.dto.ProductSalesSummary;
import com.example.entity.Product;
import jakarta.persistence.ColumnResult;
import jakarta.persistence.ConstructorResult;
import jakarta.persistence.SqlResultSetMapping;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.NativeQuery;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDate;
import java.util.List;

public interface SalesRepository extends JpaRepository<Product, Long> {

    // Native query returning interface projection
    @Query(value = """
        SELECT
            p.name as productName,
            c.name as categoryName,
            COUNT(oi.id) as totalOrders,
            SUM(oi.quantity * oi.unit_price) as totalRevenue,
            AVG(oi.quantity * oi.unit_price) as averageOrderValue
        FROM products p
        JOIN categories c ON c.id = p.category_id
        JOIN order_items oi ON oi.product_id = p.id
        JOIN orders o ON o.id = oi.order_id
        WHERE o.created_at BETWEEN :from AND :to
          AND o.status = 'DELIVERED'
        GROUP BY p.id, p.name, c.name
        ORDER BY totalRevenue DESC
        LIMIT :limit
        """, nativeQuery = true)
    List<ProductSalesSummary> findTopProductsBySales(
        @Param("from") LocalDate from,
        @Param("to") LocalDate to,
        @Param("limit") int limit
    );

    // Native query with DTO constructor
    @Query(value = """
        SELECT
            p.id,
            p.name,
            p.price,
            COALESCE(SUM(oi.quantity), 0) AS sold_count
        FROM products p
        LEFT JOIN order_items oi ON oi.product_id = p.id
        WHERE p.category_id = :categoryId
        GROUP BY p.id, p.name, p.price
        HAVING COALESCE(SUM(oi.quantity), 0) < :threshold
        ORDER BY sold_count ASC
        """, nativeQuery = true)
    List<Object[]> findSlowMovingProducts(
        @Param("categoryId") Long categoryId,
        @Param("threshold") int threshold
    );
}
```

---

## Hibernate Filters

```java
// src/main/java/com/example/entity/SoftDeletableProduct.java
package com.example.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.*;

import java.time.LocalDateTime;

/**
 * Soft-deletable entity using Hibernate @Filter.
 * The filter is inactive by default and must be explicitly enabled per session.
 */
@Entity
@Table(name = "products")
@FilterDefs({
    @FilterDef(
        name = "activeFilter",
        parameters = @ParamDef(name = "active", type = Boolean.class)
    ),
    @FilterDef(
        name = "tenantFilter",
        parameters = @ParamDef(name = "tenantId", type = Long.class)
    ),
    @FilterDef(
        name = "notDeleted",
        defaultCondition = "deleted_at IS NULL"
    )
})
@Filters({
    @Filter(name = "activeFilter", condition = "active = :active"),
    @Filter(name = "tenantFilter", condition = "tenant_id = :tenantId"),
    @Filter(name = "notDeleted")
})
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SoftDeletableProduct {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private boolean active;
    private Long tenantId;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    public void softDelete() {
        this.deletedAt = LocalDateTime.now();
    }
}
```

```java
// src/main/java/com/example/service/TenantProductService.java
package com.example.service;

import com.example.entity.SoftDeletableProduct;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.hibernate.Session;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class TenantProductService {

    private final EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<SoftDeletableProduct> getActiveProductsForTenant(Long tenantId) {
        Session session = entityManager.unwrap(Session.class);

        // Enable filters for this session
        session.enableFilter("notDeleted");
        session.enableFilter("tenantFilter")
            .setParameter("tenantId", tenantId);
        session.enableFilter("activeFilter")
            .setParameter("active", true);

        return entityManager.createQuery(
            "FROM SoftDeletableProduct p", SoftDeletableProduct.class
        ).getResultList();

        // Filters are automatically removed when session closes
    }

    @Transactional
    public void softDeleteProduct(Long productId) {
        SoftDeletableProduct product = entityManager.find(
            SoftDeletableProduct.class, productId);
        if (product != null) {
            product.softDelete();
        }
    }
}
```

---

## Inheritance Mapping Strategies

```java
// STRATEGY 1: SINGLE_TABLE - All types in one table (fastest queries, allows nullable columns)
// Good for: type hierarchies with mostly shared columns, read-heavy workloads

@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private java.math.BigDecimal amount;
    private java.time.LocalDateTime processedAt;
    private String status;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
@Getter
@Setter
@NoArgsConstructor
public class CreditCardPayment extends Payment {
    private String cardLastFour;  // Nullable for other types
    private String cardNetwork;
    private String authorizationCode;
}

@Entity
@DiscriminatorValue("BANK_TRANSFER")
@Getter
@Setter
@NoArgsConstructor
public class BankTransferPayment extends Payment {
    private String bankCode;      // Nullable for other types
    private String accountNumber;
    private String transferReference;
}

@Entity
@DiscriminatorValue("CRYPTO")
@Getter
@Setter
@NoArgsConstructor
public class CryptoPayment extends Payment {
    private String walletAddress; // Nullable for other types
    private String transactionHash;
    private String currency;
}
```

```java
// STRATEGY 2: JOINED - Each class has its own table (normalized, slower queries)
// Good for: type hierarchies with few shared columns, write-heavy workloads

@Entity
@Table(name = "vehicles")
@Inheritance(strategy = InheritanceType.JOINED)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public abstract class Vehicle {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String make;
    private String model;
    private int year;
    private String vin;
}

@Entity
@Table(name = "cars")
@PrimaryKeyJoinColumn(name = "vehicle_id")
@Getter
@Setter
@NoArgsConstructor
public class Car extends Vehicle {
    private int doorCount;
    private String bodyStyle;  // SEDAN, SUV, COUPE, etc.
    private boolean hasAutoTransmission;
}

@Entity
@Table(name = "trucks")
@PrimaryKeyJoinColumn(name = "vehicle_id")
@Getter
@Setter
@NoArgsConstructor
public class Truck extends Vehicle {
    private double payloadCapacityTons;
    private boolean hasTrailerHitch;
    private int axleCount;
}
```

```java
// STRATEGY 3: TABLE_PER_CLASS - Each concrete class has its own complete table
// Good for: types that are never queried polymorphically

@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public abstract class Notification {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)  // Must use AUTO, not IDENTITY
    private Long id;
    private String recipientId;
    private String subject;
    private java.time.LocalDateTime sentAt;
    private boolean read;
}

@Entity
@Table(name = "email_notifications")
@Getter
@Setter
@NoArgsConstructor
public class EmailNotification extends Notification {
    private String toAddress;
    private String fromAddress;
    private String bodyHtml;
    private String bodyText;
}

@Entity
@Table(name = "push_notifications")
@Getter
@Setter
@NoArgsConstructor
public class PushNotification extends Notification {
    private String deviceToken;
    private String payload;
    private String imageUrl;
}
```

---

## Embeddables and Element Collections

```java
// src/main/java/com/example/entity/embeddable/Address.java
package com.example.entity.embeddable;

import jakarta.persistence.Embeddable;
import lombok.*;

@Embeddable
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Address {
    private String street;
    private String city;
    private String state;
    private String zipCode;
    private String country;
}
```

```java
// src/main/java/com/example/entity/Customer.java
package com.example.entity;

import com.example.entity.embeddable.Address;
import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Entity
@Table(name = "customers")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;

    // Embedded value object - columns live in customers table
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "billing_street")),
        @AttributeOverride(name = "city", column = @Column(name = "billing_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "billing_zip")),
        @AttributeOverride(name = "country", column = @Column(name = "billing_country"))
    })
    private Address billingAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "shipping_street")),
        @AttributeOverride(name = "city", column = @Column(name = "shipping_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "shipping_zip")),
        @AttributeOverride(name = "country", column = @Column(name = "shipping_country"))
    })
    private Address shippingAddress;

    // Element Collection - stored in a separate table (customer_phone_numbers)
    @ElementCollection
    @CollectionTable(
        name = "customer_phone_numbers",
        joinColumns = @JoinColumn(name = "customer_id")
    )
    @Column(name = "phone_number")
    @Builder.Default
    private List<String> phoneNumbers = new ArrayList<>();

    // Element Collection with a complex embeddable
    @ElementCollection
    @CollectionTable(
        name = "customer_addresses",
        joinColumns = @JoinColumn(name = "customer_id")
    )
    @Builder.Default
    private List<Address> additionalAddresses = new ArrayList<>();

    // Map element collection
    @ElementCollection
    @CollectionTable(
        name = "customer_preferences",
        joinColumns = @JoinColumn(name = "customer_id")
    )
    @MapKeyColumn(name = "preference_key")
    @Column(name = "preference_value")
    @Builder.Default
    private Map<String, String> preferences = new HashMap<>();

    // JSON stored as column (Hibernate 6+)
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;
}
```

---

## JPA Metamodel for Type-Safe Queries

```java
// Metamodel is generated by annotation processor
// Add to pom.xml:
// <dependency>
//   <groupId>org.hibernate.orm</groupId>
//   <artifactId>hibernate-jpamodelgen</artifactId>
//   <scope>provided</scope>
// </dependency>

// After build, Product_.java is generated in target/generated-sources:
// @StaticMetamodel(Product.class)
// public abstract class Product_ {
//     public static volatile SingularAttribute<Product, Long> id;
//     public static volatile SingularAttribute<Product, String> name;
//     public static volatile SingularAttribute<Product, BigDecimal> price;
//     public static volatile SingularAttribute<Product, Category> category;
//     ...
// }

// src/main/java/com/example/repository/TypeSafeProductRepository.java
package com.example.repository;

import com.example.entity.Product;
import com.example.entity.Product_;
import jakarta.persistence.EntityManager;
import jakarta.persistence.criteria.*;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Repository
@RequiredArgsConstructor
public class TypeSafeProductRepository {

    private final EntityManager em;

    public List<Product> findByDynamicCriteria(
            String nameContains,
            BigDecimal minPrice,
            BigDecimal maxPrice,
            String categoryName,
            boolean activeOnly) {

        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);

        List<Predicate> predicates = new ArrayList<>();

        if (nameContains != null && !nameContains.isBlank()) {
            predicates.add(cb.like(
                cb.lower(root.get(Product_.name)),
                "%" + nameContains.toLowerCase() + "%"
            ));
        }

        if (minPrice != null) {
            predicates.add(cb.greaterThanOrEqualTo(
                root.get(Product_.price), minPrice));
        }

        if (maxPrice != null) {
            predicates.add(cb.lessThanOrEqualTo(
                root.get(Product_.price), maxPrice));
        }

        if (categoryName != null) {
            Join<Product, com.example.entity.Category> categoryJoin =
                root.join(Product_.category, JoinType.LEFT);
            predicates.add(cb.equal(
                categoryJoin.get(com.example.entity.Category_.name), categoryName));
        }

        if (activeOnly) {
            predicates.add(cb.isTrue(root.get(Product_.active)));
        }

        cq.where(predicates.toArray(new Predicate[0]));
        cq.orderBy(cb.asc(root.get(Product_.price)));

        return em.createQuery(cq).getResultList();
    }

    // Type-safe aggregate query
    public BigDecimal getTotalInventoryValue() {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<BigDecimal> cq = cb.createQuery(BigDecimal.class);
        Root<Product> root = cq.from(Product.class);

        Expression<BigDecimal> stockValue = cb.prod(
            root.get(Product_.price),
            cb.toBigDecimal(cb.toInteger(root.get(Product_.stockQuantity)))
        );

        cq.select(cb.sum(stockValue))
          .where(cb.isTrue(root.get(Product_.active)));

        BigDecimal result = em.createQuery(cq).getSingleResult();
        return result != null ? result : BigDecimal.ZERO;
    }
}
```

---

## Hibernate Statistics and Monitoring

```java
// src/main/java/com/example/monitoring/HibernateStatisticsEndpoint.java
package com.example.monitoring;

import lombok.RequiredArgsConstructor;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.stereotype.Component;

import jakarta.persistence.EntityManagerFactory;
import java.util.LinkedHashMap;
import java.util.Map;

@Component
@Endpoint(id = "hibernate-stats")
@RequiredArgsConstructor
public class HibernateStatisticsEndpoint {

    private final EntityManagerFactory emf;

    @ReadOperation
    public Map<String, Object> stats() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();

        Map<String, Object> result = new LinkedHashMap<>();

        // Session stats
        result.put("sessions.opened", stats.getSessionOpenCount());
        result.put("sessions.closed", stats.getSessionCloseCount());

        // Query stats
        result.put("queries.executed", stats.getQueryExecutionCount());
        result.put("queries.max_time_ms", stats.getQueryExecutionMaxTime());
        result.put("queries.slowest", stats.getQueryExecutionMaxTimeQueryString());

        // Second-level cache stats
        result.put("cache.hits", stats.getSecondLevelCacheHitCount());
        result.put("cache.misses", stats.getSecondLevelCacheMissCount());
        result.put("cache.puts", stats.getSecondLevelCachePutCount());
        result.put("cache.hit_ratio",
            calculateRatio(stats.getSecondLevelCacheHitCount(),
                stats.getSecondLevelCacheHitCount() + stats.getSecondLevelCacheMissCount()));

        // Query cache stats
        result.put("query_cache.hits", stats.getQueryCacheHitCount());
        result.put("query_cache.misses", stats.getQueryCacheMissCount());

        // Entity stats
        result.put("entities.loaded", stats.getEntityLoadCount());
        result.put("entities.fetched", stats.getEntityFetchCount());
        result.put("entities.inserted", stats.getEntityInsertCount());
        result.put("entities.updated", stats.getEntityUpdateCount());
        result.put("entities.deleted", stats.getEntityDeleteCount());

        // Collection stats
        result.put("collections.loaded", stats.getCollectionLoadCount());
        result.put("collections.fetched", stats.getCollectionFetchCount());

        // Connection stats
        result.put("connections.obtained", stats.getConnectCount());
        result.put("flushes", stats.getFlushCount());
        result.put("transactions", stats.getTransactionCount());

        return result;
    }

    private double calculateRatio(long hits, long total) {
        return total == 0 ? 0.0 : (double) hits / total;
    }
}
```

```java
// src/main/java/com/example/monitoring/SlowQueryLogger.java
package com.example.monitoring;

import lombok.extern.slf4j.Slf4j;
import net.ttddyy.dsproxy.listener.QueryExecutionListener;
import net.ttddyy.dsproxy.listener.logging.DefaultQueryLogEntryCreator;
import net.ttddyy.dsproxy.listener.logging.SystemOutQueryLoggingListener;
import net.ttddyy.dsproxy.support.ProxyDataSourceBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;
import java.util.concurrent.TimeUnit;

@Slf4j
@Configuration
@ConditionalOnProperty(name = "datasource.proxy.enabled", havingValue = "true")
public class SlowQueryLogger {

    @Value("${datasource.proxy.slow_threshold_ms:100}")
    private long slowThresholdMs;

    /**
     * Wraps DataSource with a proxy that logs slow queries.
     * Requires datasource-proxy library.
     */
    @Bean
    public DataSource dataSource(DataSource originalDataSource) {
        return ProxyDataSourceBuilder
            .create(originalDataSource)
            .name("SlowQueryProxy")
            .logSlowQueryToSysOut(slowThresholdMs, TimeUnit.MILLISECONDS)
            .countQuery()
            .build();
    }
}
```

---

## HikariCP Connection Pool Monitoring

```java
// src/main/java/com/example/monitoring/HikariMetricsConfig.java
package com.example.monitoring;

import com.zaxxer.hikari.HikariDataSource;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.annotation.PostConstruct;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@Slf4j
@Component
@RequiredArgsConstructor
public class HikariMetricsConfig implements HealthIndicator {

    private final DataSource dataSource;
    private final MeterRegistry meterRegistry;

    @PostConstruct
    public void registerMetrics() {
        if (dataSource instanceof HikariDataSource hikariDs) {
            var pool = hikariDs.getHikariPoolMXBean();
            if (pool != null) {
                Gauge.builder("hikari.connections.active", pool,
                    p -> p.getActiveConnections())
                    .register(meterRegistry);
                Gauge.builder("hikari.connections.idle", pool,
                    p -> p.getIdleConnections())
                    .register(meterRegistry);
                Gauge.builder("hikari.connections.pending", pool,
                    p -> p.getThreadsAwaitingConnection())
                    .register(meterRegistry);
                Gauge.builder("hikari.connections.total", pool,
                    p -> p.getTotalConnections())
                    .register(meterRegistry);
            }
        }
    }

    @Override
    public Health health() {
        if (dataSource instanceof HikariDataSource hikariDs) {
            var pool = hikariDs.getHikariPoolMXBean();
            if (pool != null) {
                int active = pool.getActiveConnections();
                int total = pool.getTotalConnections();
                int pending = pool.getThreadsAwaitingConnection();

                if (pending > 5) {
                    return Health.down()
                        .withDetail("active", active)
                        .withDetail("total", total)
                        .withDetail("pending", pending)
                        .withDetail("reason", "High connection wait time")
                        .build();
                }

                return Health.up()
                    .withDetail("active", active)
                    .withDetail("idle", pool.getIdleConnections())
                    .withDetail("total", total)
                    .withDetail("pending", pending)
                    .build();
            }
        }
        return Health.unknown().build();
    }
}
```

---

## Advanced Repository: N+1 Prevention

```java
// src/main/java/com/example/repository/OrderRepository.java
package com.example.repository;

import com.example.entity.Order;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;
import java.util.Optional;

public interface OrderRepository extends JpaRepository<Order, Long> {

    // BAD: Causes N+1 (1 query for orders + N queries for items)
    // List<Order> findByCustomerId(Long customerId);

    // GOOD: EntityGraph loads associations in one query
    @EntityGraph(attributePaths = {"items", "items.product"})
    List<Order> findByCustomerId(Long customerId);

    // Alternative: JOIN FETCH in JPQL (also prevents N+1)
    @Query("SELECT DISTINCT o FROM Order o " +
           "LEFT JOIN FETCH o.items i " +
           "LEFT JOIN FETCH i.product p " +
           "WHERE o.customerId = :customerId")
    List<Order> findByCustomerIdWithItems(Long customerId);

    // Named EntityGraph (defined on entity)
    @EntityGraph("Order.withCustomerAndItems")
    Optional<Order> findWithDetailsById(Long id);
}
```

```java
// src/main/java/com/example/entity/Order.java
@NamedEntityGraph(
    name = "Order.withCustomerAndItems",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items.product")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "items.product",
            attributeNodes = @NamedAttributeNode("product")
        )
    }
)
@Entity
public class Order {
    // ... fields
}
```

---

## Integration Test for Caching

```java
// src/test/java/com/example/ProductCacheIntegrationTest.java
package com.example;

import com.example.entity.Product;
import com.example.repository.ProductRepository;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import jakarta.persistence.EntityManagerFactory;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Transactional
class ProductCacheIntegrationTest {

    @Autowired ProductRepository productRepository;
    @Autowired EntityManagerFactory emf;

    private Statistics stats;

    @BeforeEach
    void setUp() {
        stats = emf.unwrap(SessionFactory.class).getStatistics();
        stats.clear();
    }

    @Test
    void secondReadHitsCache() {
        // First read - populates cache
        Product first = productRepository.findById(1L).orElseThrow();
        long hitsAfterFirst = stats.getSecondLevelCacheHitCount();
        long missesAfterFirst = stats.getSecondLevelCacheMissCount();

        assertThat(hitsAfterFirst).isEqualTo(0);    // Cache miss on first read
        assertThat(missesAfterFirst).isEqualTo(1);  // Fetched from DB

        // Second read - should hit cache
        Product second = productRepository.findById(1L).orElseThrow();
        long hitsAfterSecond = stats.getSecondLevelCacheHitCount();

        assertThat(hitsAfterSecond).isEqualTo(1);   // Cache hit on second read
        assertThat(first.getId()).isEqualTo(second.getId());
    }

    @Test
    void bulkUpdateInvalidatesCache() {
        // Load to populate cache
        productRepository.findById(1L);
        assertThat(stats.getSecondLevelCacheHitCount()).isEqualTo(0);

        // Bulk update bypasses entity cache
        // After bulk update you must call emf.getCache().evictAll()
        // or use Cache#evict(entityClass, id)
        emf.getCache().evictAll();

        // Next read should miss cache
        productRepository.findById(1L);
        assertThat(stats.getSecondLevelCacheMissCount()).isGreaterThan(0);
    }
}
```

---

## Summary

| Feature | Configuration | Notes |
|---|---|---|
| Second-level cache | `@Cache(usage = READ_WRITE)` | Requires JCache provider |
| Query cache | `HINT_CACHEABLE = "true"` | Stores result sets by query+params |
| Batch inserts | `hibernate.jdbc.batch_size=50` | `flush()` + `clear()` every batch |
| Bulk JPQL | `createQuery(...).executeUpdate()` | Bypasses cache — evict after |
| Native query | `@Query(nativeQuery=true)` | Returns interface projections |
| Filters | `@FilterDef` + `@Filter` | Enable per-session: `session.enableFilter()` |
| Single table | `SINGLE_TABLE` strategy | Fast reads, nullable columns |
| Joined table | `JOINED` strategy | Normalized, JOIN on reads |
| Table per class | `TABLE_PER_CLASS` | UNION on polymorphic queries |
| Embeddable | `@Embedded` | Columns in owner's table |
| Element collection | `@ElementCollection` | Separate table, no entity ID |
| Metamodel | `@StaticMetamodel` | Type-safe criteria queries |
| Statistics | `SessionFactory.getStatistics()` | Cache hit ratios, slow queries |

### Key Takeaways
- Never use `open-in-view = true` in production — it holds DB connections for the full HTTP request
- Batch inserts require `flush()` + `clear()` every BATCH_SIZE to prevent OOM
- Bulk JPQL UPDATE/DELETE bypasses the second-level cache — always evict after
- `SINGLE_TABLE` is fastest for reading but wastes space with nullable columns
- N+1 is usually solved with `@EntityGraph` or `JOIN FETCH` in JPQL

---

## Next Part Preview

**Part 098: Microservices Security Patterns** covers mTLS for service-to-service auth, JWT propagation, zero-trust networking, OPA for distributed authorization, secret rotation, and Istio service mesh security.
