# Part 056: Reactive Database Access with R2DBC

## Overview

R2DBC (Reactive Relational Database Connectivity) brings reactive, non-blocking database access to relational databases. Unlike JDBC (which blocks a thread per query), R2DBC uses the Reactive Streams specification, returning `Mono` and `Flux` instead of blocking results. This pairs naturally with Spring WebFlux to build end-to-end non-blocking applications capable of handling high concurrency with a small thread pool.

By the end of this part you will be able to:
- Set up Spring Data R2DBC with PostgreSQL
- Map entities and use reactive repositories
- Use `DatabaseClient` for custom SQL queries
- Handle transactions in reactive context
- Configure connection pooling
- Build a complete reactive REST API

---

## Table of Contents

1. [R2DBC Concepts](#1-r2dbc-concepts)
2. [Project Setup](#2-project-setup)
3. [Entity Mapping with Annotations](#3-entity-mapping-with-annotations)
4. [R2dbcRepository](#4-r2dbcrepository)
5. [DatabaseClient for Custom Queries](#5-databaseclient-for-custom-queries)
6. [Reactive Transactions](#6-reactive-transactions)
7. [Connection Pool Configuration](#7-connection-pool-configuration)
8. [Auditing and Events](#8-auditing-and-events)
9. [Combining R2DBC with Spring WebFlux](#9-combining-r2dbc-with-spring-webflux)
10. [N+1 Problem and Solutions](#10-n1-problem-and-solutions)
11. [Real Example: Reactive REST API with PostgreSQL](#11-real-example-reactive-rest-api-with-postgresql)
12. [Summary](#12-summary)

---

## 1. R2DBC Concepts

### Why R2DBC?

| Aspect          | JDBC (Blocking)                      | R2DBC (Reactive)                          |
|-----------------|--------------------------------------|-------------------------------------------|
| Thread model    | 1 thread per DB connection           | Event loop, shared threads                |
| Blocking        | Yes — thread waits for DB response   | No — returns immediately, callback later  |
| Scalability     | Limited by thread pool size          | High — handles thousands of concurrent ops|
| Backpressure    | Not supported                        | Built-in via Reactive Streams             |
| Use with        | Spring MVC / blocking services       | Spring WebFlux / reactive pipelines       |

### Reactive Streams in Database Context

```
HTTP Request (Netty) 
  → Controller (returns Flux/Mono)
  → Service (reactive logic)
  → Repository (R2DBC, non-blocking)
  → PostgreSQL (I/O callback)
  → Back-propagated to HTTP response
```

No thread is ever blocked waiting for I/O.

---

## 2. Project Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring WebFlux (reactive web) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <!-- Spring Data R2DBC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-r2dbc</artifactId>
    </dependency>

    <!-- R2DBC PostgreSQL driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>r2dbc-postgresql</artifactId>
    </dependency>

    <!-- R2DBC Connection Pool -->
    <dependency>
        <groupId>io.r2dbc</groupId>
        <artifactId>r2dbc-pool</artifactId>
    </dependency>

    <!-- Flyway for schema migrations (uses JDBC for migration) -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>  <!-- JDBC driver for Flyway -->
    </dependency>

    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>r2dbc</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/shop
    username: shopuser
    password: shoppass
    pool:
      initial-size: 5
      max-size: 20
      max-idle-time: 30m
      validation-query: SELECT 1

  # Flyway uses JDBC (blocking) for schema migrations only
  flyway:
    url: jdbc:postgresql://localhost:5432/shop
    user: shopuser
    password: shoppass
    locations: classpath:db/migration

logging:
  level:
    org.springframework.r2dbc: DEBUG
    io.r2dbc.postgresql.QUERY: DEBUG
    io.r2dbc.postgresql.PARAM: DEBUG
```

### Database Migration (Flyway)

```sql
-- src/main/resources/db/migration/V1__create_tables.sql

CREATE TABLE categories (
    id        BIGSERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL UNIQUE,
    slug      VARCHAR(100) NOT NULL UNIQUE,
    active    BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
    id             BIGSERIAL PRIMARY KEY,
    name           VARCHAR(255) NOT NULL,
    description    TEXT,
    sku            VARCHAR(100) UNIQUE,
    price          NUMERIC(12, 2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    category_id    BIGINT REFERENCES categories(id) ON DELETE SET NULL,
    active         BOOLEAN NOT NULL DEFAULT true,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_active ON products(active);
CREATE INDEX idx_products_price ON products(price);

CREATE TABLE product_tags (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    tag        VARCHAR(50) NOT NULL,
    PRIMARY KEY (product_id, tag)
);
```

---

## 3. Entity Mapping with Annotations

### Product Entity

```java
package com.example.shop.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.*;
import org.springframework.data.relational.core.mapping.Column;
import org.springframework.data.relational.core.mapping.Table;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Table("products")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    @Id
    private Long id;

    @Column("name")
    private String name;

    @Column("description")
    private String description;

    @Column("sku")
    private String sku;

    @Column("price")
    private BigDecimal price;

    @Column("stock_quantity")
    private Integer stockQuantity;

    @Column("category_id")
    private Long categoryId;  // foreign key — no @ManyToOne in R2DBC

    @Column("active")
    private boolean active;

    @CreatedDate
    @Column("created_at")
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column("updated_at")
    private LocalDateTime updatedAt;

    @Version
    private Long version;  // optimistic locking
}
```

### Category Entity

```java
package com.example.shop.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.*;
import org.springframework.data.relational.core.mapping.Column;
import org.springframework.data.relational.core.mapping.Table;

import java.time.LocalDateTime;

@Table("categories")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Category {

    @Id
    private Long id;

    @Column("name")
    private String name;

    @Column("slug")
    private String slug;

    @Column("active")
    private boolean active;

    @CreatedDate
    @Column("created_at")
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column("updated_at")
    private LocalDateTime updatedAt;
}
```

### R2DBC Configuration

```java
package com.example.shop.config;

import io.r2dbc.spi.ConnectionFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.r2dbc.config.EnableR2dbcAuditing;
import org.springframework.data.r2dbc.repository.config.EnableR2dbcRepositories;
import org.springframework.r2dbc.connection.R2dbcTransactionManager;
import org.springframework.transaction.ReactiveTransactionManager;

@Configuration
@EnableR2dbcRepositories
@EnableR2dbcAuditing
public class R2dbcConfig {

    @Bean
    public ReactiveTransactionManager transactionManager(ConnectionFactory connectionFactory) {
        return new R2dbcTransactionManager(connectionFactory);
    }
}
```

---

## 4. R2dbcRepository

### Repository Interface

```java
package com.example.shop.repository;

import com.example.shop.model.Product;
import org.springframework.data.domain.Pageable;
import org.springframework.data.r2dbc.repository.Modifying;
import org.springframework.data.r2dbc.repository.Query;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;
import org.springframework.data.repository.reactive.ReactiveSortingRepository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public interface ProductRepository
        extends ReactiveCrudRepository<Product, Long>,
                ReactiveSortingRepository<Product, Long> {

    // --- Derived query methods ---

    Flux<Product> findByActiveIsTrue();

    Flux<Product> findByCategoryId(Long categoryId);

    Flux<Product> findByCategoryIdAndActiveIsTrue(Long categoryId, Pageable pageable);

    Flux<Product> findByPriceLessThanEqualAndActiveIsTrue(BigDecimal maxPrice);

    Flux<Product> findByPriceBetweenAndActiveIsTrue(BigDecimal min, BigDecimal max);

    Mono<Product> findBySku(String sku);

    Mono<Boolean> existsBySku(String sku);

    Mono<Long> countByCategoryId(Long categoryId);

    Flux<Product> findByStockQuantityLessThanAndActiveIsTrue(int threshold);

    Flux<Product> findByCreatedAtAfterAndActiveIsTrue(LocalDateTime after);

    // --- @Query with SQL ---

    @Query("SELECT * FROM products WHERE active = true ORDER BY created_at DESC LIMIT :limit OFFSET :offset")
    Flux<Product> findRecentActive(int limit, int offset);

    @Query("""
        SELECT p.* FROM products p
        WHERE p.active = true
          AND (:categoryId IS NULL OR p.category_id = :categoryId)
          AND (:minPrice IS NULL OR p.price >= :minPrice)
          AND (:maxPrice IS NULL OR p.price <= :maxPrice)
        ORDER BY p.price ASC
        LIMIT :limit OFFSET :offset
        """)
    Flux<Product> searchProducts(
        Long categoryId,
        BigDecimal minPrice,
        BigDecimal maxPrice,
        int limit,
        int offset
    );

    @Query("SELECT COUNT(*) FROM products WHERE active = true AND category_id = :categoryId")
    Mono<Long> countActiveByCategory(Long categoryId);

    // Modifying queries
    @Modifying
    @Query("UPDATE products SET active = false WHERE id = :id")
    Mono<Integer> softDeleteById(Long id);

    @Modifying
    @Query("""
        UPDATE products
        SET stock_quantity = stock_quantity - :quantity,
            updated_at = now()
        WHERE id = :id AND stock_quantity >= :quantity
        """)
    Mono<Integer> decrementStock(Long id, int quantity);

    @Modifying
    @Query("UPDATE products SET active = false WHERE created_at < :cutoff AND active = true")
    Mono<Integer> deactivateOldProducts(LocalDateTime cutoff);
}
```

### Using the Repository

```java
package com.example.shop.service;

import com.example.shop.model.Product;
import com.example.shop.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    public Flux<Product> findAll() {
        return productRepository.findAll();
    }

    public Mono<Product> findById(Long id) {
        return productRepository.findById(id)
            .switchIfEmpty(Mono.error(new ProductNotFoundException(id)));
    }

    public Mono<Product> findBySku(String sku) {
        return productRepository.findBySku(sku)
            .switchIfEmpty(Mono.error(new ProductNotFoundException("SKU: " + sku)));
    }

    public Flux<Product> findByCategory(Long categoryId, int page, int size) {
        return productRepository.findByCategoryIdAndActiveIsTrue(
            categoryId,
            PageRequest.of(page, size)
        );
    }

    public Mono<Product> create(Product product) {
        return productRepository.existsBySku(product.getSku())
            .flatMap(exists -> {
                if (exists) {
                    return Mono.error(new DuplicateSkuException(product.getSku()));
                }
                product.setActive(true);
                product.setCreatedAt(LocalDateTime.now());
                return productRepository.save(product);
            });
    }

    public Mono<Product> update(Long id, Product updated) {
        return productRepository.findById(id)
            .switchIfEmpty(Mono.error(new ProductNotFoundException(id)))
            .flatMap(existing -> {
                existing.setName(updated.getName());
                existing.setDescription(updated.getDescription());
                existing.setPrice(updated.getPrice());
                existing.setStockQuantity(updated.getStockQuantity());
                existing.setCategoryId(updated.getCategoryId());
                existing.setUpdatedAt(LocalDateTime.now());
                return productRepository.save(existing);
            });
    }

    public Mono<Void> delete(Long id) {
        return productRepository.existsById(id)
            .flatMap(exists -> {
                if (!exists) return Mono.error(new ProductNotFoundException(id));
                return productRepository.deleteById(id);
            });
    }

    public Mono<Integer> decrementStock(Long id, int quantity) {
        return productRepository.decrementStock(id, quantity)
            .flatMap(updated -> {
                if (updated == 0) {
                    return Mono.error(new InsufficientStockException(id));
                }
                return Mono.just(updated);
            });
    }

    public Flux<Product> findLowStock(int threshold) {
        return productRepository.findByStockQuantityLessThanAndActiveIsTrue(threshold);
    }
}
```

---

## 5. DatabaseClient for Custom Queries

`DatabaseClient` gives you direct SQL access with full reactive support.

```java
package com.example.shop.service;

import com.example.shop.model.Product;
import com.example.shop.dto.*;
import io.r2dbc.spi.Row;
import io.r2dbc.spi.RowMetadata;
import lombok.RequiredArgsConstructor;
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.function.BiFunction;

@Service
@RequiredArgsConstructor
public class AdvancedQueryService {

    private final DatabaseClient databaseClient;

    // --- Custom row mapper ---

    private static final BiFunction<Row, RowMetadata, ProductSummary> SUMMARY_MAPPER =
        (row, meta) -> ProductSummary.builder()
            .id(row.get("id", Long.class))
            .name(row.get("name", String.class))
            .price(row.get("price", BigDecimal.class))
            .category(row.get("category_name", String.class))
            .stockQuantity(row.get("stock_quantity", Integer.class))
            .build();

    // --- JOIN query (products with category names) ---

    public Flux<ProductSummary> findWithCategory(Long categoryId) {
        String sql = """
            SELECT p.id, p.name, p.price, p.stock_quantity,
                   c.name AS category_name
            FROM products p
            LEFT JOIN categories c ON p.category_id = c.id
            WHERE (:categoryId IS NULL OR p.category_id = :categoryId)
              AND p.active = true
            ORDER BY p.created_at DESC
            """;

        return databaseClient.sql(sql)
            .bind("categoryId", categoryId != null
                ? io.r2dbc.spi.Parameter.fromOrEmpty(categoryId, Long.class)
                : io.r2dbc.spi.Parameter.fromOrEmpty(null, Long.class))
            .map(SUMMARY_MAPPER)
            .all();
    }

    // --- Full-text search with PostgreSQL tsvector ---

    public Flux<Product> fullTextSearch(String query) {
        String sql = """
            SELECT *
            FROM products
            WHERE active = true
              AND to_tsvector('english', name || ' ' || COALESCE(description, ''))
                  @@ plainto_tsquery('english', :query)
            ORDER BY ts_rank(
                to_tsvector('english', name || ' ' || COALESCE(description, '')),
                plainto_tsquery('english', :query)
            ) DESC
            LIMIT 50
            """;

        return databaseClient.sql(sql)
            .bind("query", query)
            .mapProperties(Product.class)
            .all();
    }

    // --- Aggregation query ---

    public Flux<CategoryStats> getCategoryStats() {
        String sql = """
            SELECT
                c.name AS category,
                COUNT(p.id) AS product_count,
                AVG(p.price) AS avg_price,
                SUM(p.stock_quantity) AS total_stock,
                MIN(p.price) AS min_price,
                MAX(p.price) AS max_price
            FROM categories c
            LEFT JOIN products p ON c.id = p.category_id AND p.active = true
            WHERE c.active = true
            GROUP BY c.id, c.name
            ORDER BY product_count DESC
            """;

        return databaseClient.sql(sql)
            .map((row, meta) -> CategoryStats.builder()
                .category(row.get("category", String.class))
                .productCount(row.get("product_count", Long.class))
                .avgPrice(row.get("avg_price", BigDecimal.class))
                .totalStock(row.get("total_stock", Long.class))
                .minPrice(row.get("min_price", BigDecimal.class))
                .maxPrice(row.get("max_price", BigDecimal.class))
                .build()
            )
            .all();
    }

    // --- Batch insert ---

    public Mono<Long> batchInsertProducts(java.util.List<Product> products) {
        String sql = """
            INSERT INTO products (name, description, sku, price, stock_quantity, category_id, active, created_at, updated_at)
            VALUES (:name, :description, :sku, :price, :stockQuantity, :categoryId, :active, :createdAt, :updatedAt)
            """;

        return Flux.fromIterable(products)
            .flatMap(p ->
                databaseClient.sql(sql)
                    .bind("name", p.getName())
                    .bind("description", p.getDescription() != null ? p.getDescription() : "")
                    .bind("sku", p.getSku())
                    .bind("price", p.getPrice())
                    .bind("stockQuantity", p.getStockQuantity())
                    .bind("categoryId",
                        io.r2dbc.spi.Parameter.fromOrEmpty(p.getCategoryId(), Long.class))
                    .bind("active", p.isActive())
                    .bind("createdAt", LocalDateTime.now())
                    .bind("updatedAt", LocalDateTime.now())
                    .fetch().rowsUpdated()
            )
            .reduce(0L, Long::sum);
    }

    // --- Named parameter update ---

    public Mono<Integer> updatePricesByCategoryId(Long categoryId, BigDecimal multiplier) {
        return databaseClient.sql("""
            UPDATE products
            SET price = price * :multiplier,
                updated_at = now()
            WHERE category_id = :categoryId
              AND active = true
            """)
            .bind("categoryId", categoryId)
            .bind("multiplier", multiplier)
            .fetch()
            .rowsUpdated()
            .map(Long::intValue);
    }
}
```

---

## 6. Reactive Transactions

### Declarative Transactions

```java
package com.example.shop.service;

import com.example.shop.model.*;
import com.example.shop.repository.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderItemRepository orderItemRepository;
    private final ProductRepository productRepository;

    @Transactional
    public Mono<Order> createOrder(String customerId, List<OrderItemRequest> items) {
        // 1. Validate and decrement stock for all items atomically
        Flux<OrderItem> stockChecks = Flux.fromIterable(items)
            .flatMap(item ->
                productRepository.decrementStock(item.getProductId(), item.getQuantity())
                    .flatMap(updated -> {
                        if (updated == 0) {
                            return Mono.error(new InsufficientStockException(
                                item.getProductId()
                            ));
                        }
                        return productRepository.findById(item.getProductId())
                            .map(product -> OrderItem.builder()
                                .productId(item.getProductId())
                                .quantity(item.getQuantity())
                                .unitPrice(product.getPrice())
                                .build()
                            );
                    })
            );

        // 2. Collect all order items and compute total
        return stockChecks.collectList()
            .flatMap(orderItems -> {
                BigDecimal total = orderItems.stream()
                    .map(oi -> oi.getUnitPrice().multiply(new BigDecimal(oi.getQuantity())))
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

                Order order = Order.builder()
                    .customerId(customerId)
                    .totalAmount(total)
                    .status("PENDING")
                    .createdAt(LocalDateTime.now())
                    .build();

                // 3. Save order, then save all items
                return orderRepository.save(order)
                    .flatMap(savedOrder -> {
                        orderItems.forEach(oi -> oi.setOrderId(savedOrder.getId()));
                        return orderItemRepository.saveAll(orderItems)
                            .collectList()
                            .thenReturn(savedOrder);
                    });
            });
    }
}
```

### Programmatic Transactions

```java
package com.example.shop.service;

import lombok.RequiredArgsConstructor;
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.stereotype.Service;
import org.springframework.transaction.ReactiveTransactionManager;
import org.springframework.transaction.reactive.TransactionalOperator;
import reactor.core.publisher.Mono;

@Service
@RequiredArgsConstructor
public class TransferService {

    private final DatabaseClient databaseClient;
    private final ReactiveTransactionManager txManager;

    public Mono<Void> transferStock(Long fromProductId, Long toProductId, int quantity) {
        TransactionalOperator txOperator = TransactionalOperator.create(txManager);

        Mono<Void> transfer =
            // Step 1: Decrement source
            databaseClient.sql("""
                UPDATE products
                SET stock_quantity = stock_quantity - :qty
                WHERE id = :id AND stock_quantity >= :qty
                """)
                .bind("qty", quantity)
                .bind("id", fromProductId)
                .fetch().rowsUpdated()
                .flatMap(updated -> {
                    if (updated == 0) {
                        return Mono.error(new RuntimeException("Insufficient stock in source"));
                    }
                    // Step 2: Increment destination
                    return databaseClient.sql("""
                        UPDATE products
                        SET stock_quantity = stock_quantity + :qty
                        WHERE id = :id
                        """)
                        .bind("qty", quantity)
                        .bind("id", toProductId)
                        .fetch().rowsUpdated();
                })
                .then();

        // Wrap with transaction
        return txOperator.transactional(transfer);
    }
}
```

---

## 7. Connection Pool Configuration

### Explicit R2DBC Pool Setup

```java
package com.example.shop.config;

import io.r2dbc.pool.ConnectionPool;
import io.r2dbc.pool.ConnectionPoolConfiguration;
import io.r2dbc.postgresql.PostgresqlConnectionConfiguration;
import io.r2dbc.postgresql.PostgresqlConnectionFactory;
import io.r2dbc.spi.ConnectionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.r2dbc.config.AbstractR2dbcConfiguration;

import java.time.Duration;

@Configuration
public class R2dbcPoolConfig extends AbstractR2dbcConfiguration {

    @Value("${spring.r2dbc.host:localhost}")
    private String host;

    @Value("${spring.r2dbc.port:5432}")
    private int port;

    @Value("${spring.r2dbc.database:shop}")
    private String database;

    @Value("${spring.r2dbc.username:shopuser}")
    private String username;

    @Value("${spring.r2dbc.password:shoppass}")
    private String password;

    @Override
    @Bean
    public ConnectionFactory connectionFactory() {
        PostgresqlConnectionFactory baseFactory = new PostgresqlConnectionFactory(
            PostgresqlConnectionConfiguration.builder()
                .host(host)
                .port(port)
                .database(database)
                .username(username)
                .password(password)
                .connectTimeout(Duration.ofSeconds(5))
                .statementTimeout(Duration.ofSeconds(30))
                .build()
        );

        ConnectionPoolConfiguration poolConfig = ConnectionPoolConfiguration.builder(baseFactory)
            .name("shop-pool")
            .initialSize(5)
            .maxSize(20)
            .minIdle(2)
            .maxIdleTime(Duration.ofMinutes(30))
            .maxLifeTime(Duration.ofHours(1))
            .maxAcquireTime(Duration.ofSeconds(3))  // timeout to acquire a connection
            .validationQuery("SELECT 1")
            .acquireRetry(3)
            .build();

        return new ConnectionPool(poolConfig);
    }
}
```

---

## 8. Auditing and Events

### Enable Auditing

```java
package com.example.shop.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.ReactiveAuditorAware;
import org.springframework.data.r2dbc.config.EnableR2dbcAuditing;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import reactor.core.publisher.Mono;

@Configuration
@EnableR2dbcAuditing
public class AuditingConfig {

    @Bean
    public ReactiveAuditorAware<String> auditorProvider() {
        return () -> ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .switchIfEmpty(Mono.just("system"));
    }
}
```

### Lifecycle Callbacks with @BeforeConvert

```java
package com.example.shop.listener;

import com.example.shop.model.Product;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.r2dbc.mapping.event.BeforeConvertCallback;
import reactor.core.publisher.Mono;

import java.time.LocalDateTime;

@Slf4j
@Configuration
public class ProductCallbacks {

    @Bean
    public BeforeConvertCallback<Product> productBeforeConvert() {
        return (product, table) -> {
            if (product.getCreatedAt() == null) {
                product.setCreatedAt(LocalDateTime.now());
            }
            product.setUpdatedAt(LocalDateTime.now());

            // Generate slug from name
            if (product.getSku() == null && product.getName() != null) {
                product.setSku(generateSku(product.getName()));
            }

            log.debug("BeforeConvert: product {}", product.getId());
            return Mono.just(product);
        };
    }

    private String generateSku(String name) {
        return name.toUpperCase()
            .replaceAll("[^A-Z0-9]", "-")
            .replaceAll("-+", "-")
            .substring(0, Math.min(20, name.length()))
            + "-" + System.currentTimeMillis() % 10000;
    }
}
```

---

## 9. Combining R2DBC with Spring WebFlux

### Reactive REST Controller

```java
package com.example.shop.controller;

import com.example.shop.dto.*;
import com.example.shop.model.Product;
import com.example.shop.service.ProductService;
import com.example.shop.service.AdvancedQueryService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.time.Duration;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ReactiveProductController {

    private final ProductService productService;
    private final AdvancedQueryService advancedQueryService;

    @GetMapping
    public Flux<Product> listAll() {
        return productService.findAll();
    }

    @GetMapping("/{id}")
    public Mono<Product> getById(@PathVariable Long id) {
        return productService.findById(id);
    }

    @GetMapping("/sku/{sku}")
    public Mono<Product> getBySku(@PathVariable String sku) {
        return productService.findBySku(sku);
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Product> streamAll() {
        // Server-Sent Events: stream products one by one
        return productService.findAll()
            .delayElements(Duration.ofMillis(100));
    }

    @GetMapping("/search")
    public Flux<ProductSummary> search(@RequestParam String q) {
        return advancedQueryService.fullTextSearch(q)
            .map(this::toSummary);
    }

    @GetMapping("/analytics/categories")
    public Flux<CategoryStats> categoryStats() {
        return advancedQueryService.getCategoryStats();
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<Product> create(@Valid @RequestBody CreateProductRequest request) {
        return productService.create(toProduct(request));
    }

    @PutMapping("/{id}")
    public Mono<Product> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        return productService.update(id, toProduct(request));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public Mono<Void> delete(@PathVariable Long id) {
        return productService.delete(id);
    }

    @PatchMapping("/{id}/stock/decrement")
    public Mono<Void> decrementStock(
            @PathVariable Long id,
            @RequestParam int quantity) {
        return productService.decrementStock(id, quantity).then();
    }

    private ProductSummary toSummary(Product p) {
        return ProductSummary.builder()
            .id(p.getId())
            .name(p.getName())
            .price(p.getPrice())
            .stockQuantity(p.getStockQuantity())
            .build();
    }

    private Product toProduct(CreateProductRequest req) {
        return Product.builder()
            .name(req.getName())
            .description(req.getDescription())
            .sku(req.getSku())
            .price(req.getPrice())
            .stockQuantity(req.getStockQuantity())
            .categoryId(req.getCategoryId())
            .active(true)
            .build();
    }

    private Product toProduct(UpdateProductRequest req) {
        return Product.builder()
            .name(req.getName())
            .description(req.getDescription())
            .price(req.getPrice())
            .stockQuantity(req.getStockQuantity())
            .categoryId(req.getCategoryId())
            .build();
    }
}
```

### Global Error Handling for WebFlux

```java
package com.example.shop.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import reactor.core.publisher.Mono;

@RestControllerAdvice
public class ReactiveExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public Mono<ProblemDetail> handleNotFound(ProductNotFoundException ex) {
        ProblemDetail detail = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage()
        );
        detail.setTitle("Product Not Found");
        return Mono.just(detail);
    }

    @ExceptionHandler(InsufficientStockException.class)
    public Mono<ProblemDetail> handleInsufficientStock(InsufficientStockException ex) {
        ProblemDetail detail = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, ex.getMessage()
        );
        detail.setTitle("Insufficient Stock");
        return Mono.just(detail);
    }

    @ExceptionHandler(DuplicateSkuException.class)
    public Mono<ProblemDetail> handleDuplicateSku(DuplicateSkuException ex) {
        ProblemDetail detail = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, "SKU already exists: " + ex.getSku()
        );
        detail.setTitle("Duplicate SKU");
        return Mono.just(detail);
    }
}
```

---

## 10. N+1 Problem and Solutions

### The N+1 Problem in R2DBC

R2DBC has no lazy loading or JPA-style relationships. If you need to fetch products with their category names, a naive approach causes N+1:

```java
// BAD — N+1: 1 query for products + N queries for categories
Flux<ProductWithCategory> bad = productRepository.findAll()
    .flatMap(product ->
        categoryRepository.findById(product.getCategoryId())  // N queries!
            .map(cat -> new ProductWithCategory(product, cat))
    );
```

### Solutions

```java
// GOOD — Solution 1: JOIN in SQL
// DatabaseClient with a JOIN query returns everything in one query
Flux<ProductSummary> good = databaseClient.sql("""
    SELECT p.*, c.name AS category_name
    FROM products p
    LEFT JOIN categories c ON p.category_id = c.id
    WHERE p.active = true
    """)
    .map(SUMMARY_MAPPER)
    .all();

// GOOD — Solution 2: flatMap with concatMap (sequential, preserves order)
// Use when you truly need N queries but want sequential execution
Flux<ProductWithCategory> sequential = productRepository.findAll()
    .concatMap(product ->                     // concatMap: sequential
        categoryRepository.findById(product.getCategoryId())
            .map(cat -> new ProductWithCategory(product, cat))
    );

// GOOD — Solution 3: flatMap with concurrency limit
Flux<ProductWithCategory> bounded = productRepository.findAll()
    .flatMap(
        product -> categoryRepository.findById(product.getCategoryId())
            .map(cat -> new ProductWithCategory(product, cat)),
        8  // max 8 concurrent inner subscriptions
    );

// GOOD — Solution 4: Batch load categories by ID
Mono<List<ProductWithCategory>> batched = productRepository.findAll()
    .collectList()
    .flatMap(products -> {
        var categoryIds = products.stream()
            .map(Product::getCategoryId)
            .filter(id -> id != null)
            .collect(java.util.stream.Collectors.toSet());

        return categoryRepository.findAllById(categoryIds)
            .collectMap(Category::getId)
            .map(categoryMap ->
                products.stream()
                    .map(p -> new ProductWithCategory(
                        p,
                        categoryMap.get(p.getCategoryId())
                    ))
                    .toList()
            );
    });
```

---

## 11. Real Example: Reactive REST API with PostgreSQL

### Complete DTOs

```java
package com.example.shop.dto;

import jakarta.validation.constraints.*;
import lombok.Builder;
import lombok.Data;

import java.math.BigDecimal;

@Data
public class CreateProductRequest {
    @NotBlank
    private String name;
    private String description;
    @NotBlank
    private String sku;
    @NotNull @DecimalMin("0.01")
    private BigDecimal price;
    @Min(0)
    private int stockQuantity;
    private Long categoryId;
}

@Data
class UpdateProductRequest {
    @NotBlank
    private String name;
    private String description;
    @NotNull @DecimalMin("0.01")
    private BigDecimal price;
    @Min(0)
    private int stockQuantity;
    private Long categoryId;
}

@Data
@Builder
class ProductSummary {
    private Long id;
    private String name;
    private BigDecimal price;
    private String category;
    private Integer stockQuantity;
}

@Data
@Builder
class CategoryStats {
    private String category;
    private Long productCount;
    private BigDecimal avgPrice;
    private Long totalStock;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
}
```

### Integration Test

```java
package com.example.shop;

import com.example.shop.dto.CreateProductRequest;
import com.example.shop.model.Product;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.r2dbc.core.DatabaseClient;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import reactor.test.StepVerifier;

import java.math.BigDecimal;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class ReactiveProductApiTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16")
            .withDatabaseName("shop_test")
            .withUsername("shopuser")
            .withPassword("shoppass");

    @DynamicPropertySource
    static void postgresProperties(DynamicPropertyRegistry registry) {
        String r2dbcUrl = "r2dbc:postgresql://" +
            postgres.getHost() + ":" + postgres.getMappedPort(5432) + "/shop_test";
        registry.add("spring.r2dbc.url", () -> r2dbcUrl);
        registry.add("spring.r2dbc.username", postgres::getUsername);
        registry.add("spring.r2dbc.password", postgres::getPassword);
        registry.add("spring.flyway.url", postgres::getJdbcUrl);
        registry.add("spring.flyway.user", postgres::getUsername);
        registry.add("spring.flyway.password", postgres::getPassword);
    }

    @Autowired
    private WebTestClient webTestClient;

    @Autowired
    private DatabaseClient databaseClient;

    @Test
    void shouldCreateProduct() {
        CreateProductRequest request = new CreateProductRequest();
        request.setName("Test Laptop");
        request.setSku("LAP-001");
        request.setPrice(new BigDecimal("999.99"));
        request.setStockQuantity(100);

        webTestClient.post()
            .uri("/api/v1/products")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(Product.class)
            .value(product -> {
                assert product.getId() != null;
                assert product.getName().equals("Test Laptop");
                assert product.getPrice().compareTo(new BigDecimal("999.99")) == 0;
            });
    }

    @Test
    void shouldReturnNotFoundForMissingProduct() {
        webTestClient.get()
            .uri("/api/v1/products/99999")
            .exchange()
            .expectStatus().isNotFound();
    }

    @Test
    void shouldStreamProducts() {
        // Insert test data first
        databaseClient.sql("""
            INSERT INTO products (name, sku, price, stock_quantity, active, created_at, updated_at)
            VALUES ('Stream Test', 'STR-001', 10.00, 5, true, now(), now())
            """).fetch().rowsUpdated().block();

        webTestClient.get()
            .uri("/api/v1/products/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentTypeCompatibleWith(MediaType.TEXT_EVENT_STREAM)
            .returnResult(Product.class)
            .getResponseBody()
            .take(1)
            .as(StepVerifier::create)
            .expectNextCount(1)
            .verifyComplete();
    }

    @Test
    void shouldDecrementStockAtomically() {
        // Insert product with known stock
        Long productId = databaseClient.sql("""
            INSERT INTO products (name, sku, price, stock_quantity, active, created_at, updated_at)
            VALUES ('Stock Test', 'STK-001', 20.00, 10, true, now(), now())
            RETURNING id
            """)
            .map((row, meta) -> row.get("id", Long.class))
            .one()
            .block();

        webTestClient.patch()
            .uri("/api/v1/products/{id}/stock/decrement?quantity=3", productId)
            .exchange()
            .expectStatus().isOk();

        // Verify stock decremented
        Long stock = databaseClient.sql(
            "SELECT stock_quantity FROM products WHERE id = :id"
        ).bind("id", productId)
            .map((row, meta) -> row.get("stock_quantity", Integer.class).longValue())
            .one()
            .block();

        assert stock == 7;
    }
}
```

---

## 12. Summary

| Feature                         | R2DBC API                                                     |
|---------------------------------|---------------------------------------------------------------|
| Entity mapping                  | `@Table`, `@Column`, `@Id`                                   |
| Auditing                        | `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`            |
| Optimistic locking              | `@Version`                                                   |
| Repository                      | `ReactiveCrudRepository`, `ReactiveSortingRepository`        |
| Custom SQL                      | `@Query("SELECT ...")` on repository methods                 |
| Full control                    | `DatabaseClient` with `.sql().bind().map().all()`            |
| Transactions (declarative)      | `@Transactional` with `ReactiveTransactionManager`           |
| Transactions (programmatic)     | `TransactionalOperator.transactional(mono)`                  |
| Connection pooling              | `r2dbc-pool` with `ConnectionPoolConfiguration`              |
| Lifecycle callbacks             | `BeforeConvertCallback<T>`                                   |
| Error handling                  | `switchIfEmpty()`, `onErrorResume()`, `@ExceptionHandler`    |
| N+1 prevention                  | JOIN queries with `DatabaseClient`, batch loading            |

### Key Takeaways

1. R2DBC has no lazy loading — always use JOIN queries or batch loading to avoid N+1.
2. `@Transactional` works in reactive context with `R2dbcTransactionManager`.
3. Use `concatMap` for sequential operations, `flatMap` with concurrency limit for parallel.
4. `DatabaseClient` is your escape hatch for raw SQL — use it for complex queries.
5. R2DBC + WebFlux makes sense for high-concurrency, I/O-bound workloads; don't use it for CPU-bound tasks.

---

## Next Part Preview

**Part 057: Spring Session and Distributed Sessions** — Managing HTTP sessions in a multi-instance environment using Redis, session serialization, timeouts, and integration with Spring Security.
