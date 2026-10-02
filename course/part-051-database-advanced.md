# Part 051: Advanced Database Patterns

## Table of Contents
1. [Database Sharding Concepts](#sharding)
2. [Read Replicas with Spring](#read-replicas)
3. [Optimistic vs Pessimistic Locking](#locking)
4. [Database Migrations with Flyway](#flyway)
5. [Soft Delete](#soft-delete)
6. [Auditing with Hibernate Envers](#envers)
7. [Full-Text Search with Hibernate Search](#full-text-search)
8. [Time-Series Data with TimescaleDB](#timescaledb)
9. [JSON Columns with PostgreSQL](#json-columns)
10. [Connection Pool Monitoring](#pool-monitoring)
11. [Real Example: E-Commerce with All Patterns](#real-example)
12. [Summary](#summary)

---

## 1. Database Sharding Concepts {#sharding}

Sharding splits data horizontally across multiple database instances.

```
┌─────────────────────────────────────────────────────┐
│                  Application Layer                    │
│                    (ShardRouter)                      │
│         hash(userId) % numShards → shardId           │
└──────────┬──────────────┬──────────────┬─────────────┘
           │              │              │
    ┌──────▼───┐    ┌──────▼───┐   ┌──────▼───┐
    │  Shard 0 │    │  Shard 1 │   │  Shard 2 │
    │ users    │    │ users    │   │ users    │
    │ 0..33%   │    │ 33..66%  │   │ 66..100% │
    └──────────┘    └──────────┘   └──────────┘
```

### Shard Configuration

```java
package com.example.database.sharding;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

/**
 * Routes queries to the appropriate shard based on the current shard context
 */
@Slf4j
public class ShardRoutingDataSource extends AbstractRoutingDataSource {

    private static final ThreadLocal<Integer> CURRENT_SHARD = new ThreadLocal<>();

    public static void setCurrentShard(int shardId) {
        CURRENT_SHARD.set(shardId);
    }

    public static void clearCurrentShard() {
        CURRENT_SHARD.remove();
    }

    @Override
    protected Object determineCurrentLookupKey() {
        Integer shard = CURRENT_SHARD.get();
        if (shard == null) {
            log.warn("No shard context set, using default shard 0");
            return 0;
        }
        return shard;
    }
}
```

```java
package com.example.database.sharding;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Configuration
@Profile("sharding")
public class ShardingConfig {

    @Value("${sharding.shard-count:3}")
    private int shardCount;

    @Bean
    public DataSource dataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();

        for (int i = 0; i < shardCount; i++) {
            targetDataSources.put(i, createShardDataSource(i));
        }

        ShardRoutingDataSource routingDataSource = new ShardRoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(targetDataSources.get(0));
        routingDataSource.afterPropertiesSet();

        return routingDataSource;
    }

    private DataSource createShardDataSource(int shardId) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(String.format(
                "jdbc:postgresql://shard-%d.db.example.com:5432/app_db", shardId));
        config.setUsername("appuser");
        config.setPassword("${DB_PASSWORD}");
        config.setMaximumPoolSize(10);
        config.setPoolName("shard-" + shardId + "-pool");
        return new HikariDataSource(config);
    }
}
```

### Shard-Aware Service

```java
package com.example.database.sharding;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class ShardAwareUserService {

    private final UserRepository userRepository;

    private static final int SHARD_COUNT = 3;

    public User findById(Long userId) {
        int shard = resolveShardForUser(userId);
        ShardRoutingDataSource.setCurrentShard(shard);
        try {
            return userRepository.findById(userId)
                    .orElseThrow(() -> new RuntimeException("User not found: " + userId));
        } finally {
            ShardRoutingDataSource.clearCurrentShard();
        }
    }

    public User save(User user) {
        int shard = resolveShardForUser(user.getId());
        ShardRoutingDataSource.setCurrentShard(shard);
        try {
            return userRepository.save(user);
        } finally {
            ShardRoutingDataSource.clearCurrentShard();
        }
    }

    private int resolveShardForUser(Long userId) {
        // Hash-based sharding: consistent distribution
        return (int) (Math.abs(userId.hashCode()) % SHARD_COUNT);
    }
}
```

---

## 2. Read Replicas with Spring {#read-replicas}

Route read queries to replicas, writes to primary.

```java
package com.example.database.replica;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

/**
 * Routes to read replica for read-only transactions,
 * primary for write transactions
 */
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {

    private static final String PRIMARY = "primary";
    private static final String REPLICA = "replica";

    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        String target = isReadOnly ? REPLICA : PRIMARY;
        return target;
    }
}
```

```java
package com.example.database.replica;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;
import org.springframework.context.annotation.Primary;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.util.Map;

@Configuration
public class ReplicaDataSourceConfig {

    @Value("${db.primary.url}")
    private String primaryUrl;

    @Value("${db.replica.url}")
    private String replicaUrl;

    @Value("${db.username}")
    private String username;

    @Value("${db.password}")
    private String password;

    @Bean(name = "primaryDataSource")
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(primaryUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(15);
        config.setPoolName("primary-pool");
        return new HikariDataSource(config);
    }

    @Bean(name = "replicaDataSource")
    public DataSource replicaDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(replicaUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(20);  // More connections for reads
        config.setReadOnly(true);
        config.setPoolName("replica-pool");
        return new HikariDataSource(config);
    }

    @Bean
    @Primary
    @DependsOn({"primaryDataSource", "replicaDataSource"})
    public DataSource routingDataSource() {
        ReadWriteRoutingDataSource routingDataSource = new ReadWriteRoutingDataSource();
        routingDataSource.setTargetDataSources(Map.of(
                "primary", primaryDataSource(),
                "replica", replicaDataSource()
        ));
        routingDataSource.setDefaultTargetDataSource(primaryDataSource());
        routingDataSource.afterPropertiesSet();
        return routingDataSource;
    }
}
```

### Using Read Replicas in Services

```java
package com.example.database.replica;

import com.example.database.entity.Product;
import com.example.database.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    /**
     * READ operation - uses replica
     */
    @Transactional(readOnly = true)
    public Page<Product> findAll(Pageable pageable) {
        return productRepository.findAll(pageable);
    }

    /**
     * READ operation - uses replica
     */
    @Transactional(readOnly = true)
    public Product findById(Long id) {
        return productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found: " + id));
    }

    /**
     * WRITE operation - uses primary
     */
    @Transactional
    public Product save(Product product) {
        return productRepository.save(product);
    }

    /**
     * WRITE operation - uses primary
     */
    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id);
    }
}
```

---

## 3. Optimistic vs Pessimistic Locking {#locking}

### Optimistic Locking

```java
package com.example.database.entity;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name = "inventory")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Inventory {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_id", nullable = false)
    private Long productId;

    @Column(nullable = false)
    private int quantity;

    @Column(nullable = false)
    private BigDecimal reservedQuantity;

    /**
     * @Version enables optimistic locking.
     * JPA checks: UPDATE ... WHERE id=? AND version=?
     * If another transaction already updated it, the WHERE clause finds 0 rows
     * → OptimisticLockException is thrown
     */
    @Version
    private Long version;
}
```

```java
package com.example.database.service;

import com.example.database.entity.Inventory;
import com.example.database.repository.InventoryRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class InventoryService {

    private final InventoryRepository inventoryRepository;

    /**
     * Optimistic locking: multiple threads can read simultaneously.
     * Conflict detected at commit time, not when reading.
     * Best for low-contention scenarios.
     */
    @Transactional
    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    public void deductStock(Long productId, int quantity) {
        Inventory inventory = inventoryRepository.findByProductId(productId)
                .orElseThrow(() -> new RuntimeException("Inventory not found: " + productId));

        if (inventory.getQuantity() < quantity) {
            throw new InsufficientStockException(
                    "Insufficient stock: available=" + inventory.getQuantity()
                    + ", requested=" + quantity);
        }

        inventory.setQuantity(inventory.getQuantity() - quantity);
        inventoryRepository.save(inventory);
        // ↑ If another transaction changed this row, this will throw
        // OptimisticLockException, and @Retryable will retry up to 3 times
    }
}
```

### Pessimistic Locking

```java
package com.example.database.repository;

import com.example.database.entity.Inventory;
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.jpa.repository.QueryHints;
import org.springframework.data.repository.query.Param;

import jakarta.persistence.QueryHint;
import java.util.Optional;

public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    /**
     * Pessimistic WRITE lock: SELECT ... FOR UPDATE
     * Only one transaction can hold the lock at a time.
     * Other transactions wait until released.
     * Best for high-contention scenarios where conflicts are frequent.
     */
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT i FROM Inventory i WHERE i.productId = :productId")
    @QueryHints({
        @QueryHint(name = "jakarta.persistence.lock.timeout", value = "5000")
    })
    Optional<Inventory> findByProductIdWithLock(@Param("productId") Long productId);

    /**
     * Pessimistic READ lock: SELECT ... FOR SHARE
     * Multiple readers allowed, but no writers.
     */
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT i FROM Inventory i WHERE i.productId = :productId")
    Optional<Inventory> findByProductIdForRead(@Param("productId") Long productId);

    Optional<Inventory> findByProductId(Long productId);
}
```

```java
package com.example.database.service;

import com.example.database.entity.Inventory;
import com.example.database.repository.InventoryRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class PessimisticInventoryService {

    private final InventoryRepository inventoryRepository;

    /**
     * Pessimistic locking: lock the row before reading.
     * Guarantees only one transaction can modify at a time.
     * May cause performance bottleneck if many transactions compete.
     */
    @Transactional
    public void deductStockPessimistic(Long productId, int quantity) {
        // Lock acquired here: other transactions wait
        Inventory inventory = inventoryRepository.findByProductIdWithLock(productId)
                .orElseThrow(() -> new RuntimeException("Inventory not found: " + productId));

        if (inventory.getQuantity() < quantity) {
            throw new InsufficientStockException(
                    "Insufficient stock: available=" + inventory.getQuantity());
        }

        inventory.setQuantity(inventory.getQuantity() - quantity);
        inventoryRepository.save(inventory);
        // Lock released when transaction commits
    }
}
```

---

## 4. Database Migrations with Flyway {#flyway}

### Flyway Configuration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    baseline-version: 0
    locations: classpath:db/migration
    schemas: public
    validate-on-migrate: true
    out-of-order: false  # Enforce strict ordering in production
    clean-disabled: true  # NEVER clean production database!

  jpa:
    hibernate:
      ddl-auto: validate  # Don't let Hibernate modify schema
```

### Migration Files

```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__create_products_table.sql
├── V3__add_product_indexes.sql
├── V4__create_orders_table.sql
├── V5__add_audit_columns.sql
├── V6__create_inventory_table.sql
├── V7__add_soft_delete.sql
└── R__create_views.sql  # Repeatable migration (R__ prefix)
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL UNIQUE,
    name        VARCHAR(100) NOT NULL,
    role        VARCHAR(50)  NOT NULL DEFAULT 'USER',
    active      BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active) WHERE active = TRUE;
```

```sql
-- V2__create_products_table.sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100)   NOT NULL,
    description TEXT,
    price       NUMERIC(10, 2) NOT NULL CHECK (price > 0),
    category    VARCHAR(50)    NOT NULL,
    owner_id    UUID           NOT NULL REFERENCES users(id),
    active      BOOLEAN        NOT NULL DEFAULT TRUE,
    deleted     BOOLEAN        NOT NULL DEFAULT FALSE,
    version     BIGINT         NOT NULL DEFAULT 0,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_owner   ON products(owner_id);
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_active   ON products(active) WHERE active = TRUE AND deleted = FALSE;
```

```sql
-- V5__add_audit_columns.sql
-- Add audit columns to all relevant tables
ALTER TABLE products ADD COLUMN IF NOT EXISTS created_by UUID REFERENCES users(id);
ALTER TABLE products ADD COLUMN IF NOT EXISTS updated_by UUID REFERENCES users(id);

ALTER TABLE orders ADD COLUMN IF NOT EXISTS created_by UUID REFERENCES users(id);
ALTER TABLE orders ADD COLUMN IF NOT EXISTS updated_by UUID REFERENCES users(id);
```

```sql
-- V7__add_soft_delete.sql
-- Add deleted_at timestamp for soft delete tracking
ALTER TABLE products ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP WITH TIME ZONE;
ALTER TABLE products ADD COLUMN IF NOT EXISTS deleted_by UUID REFERENCES users(id);

-- Partial index: only index non-deleted rows for performance
CREATE INDEX idx_products_not_deleted ON products(id) WHERE deleted = FALSE;
```

```sql
-- R__create_views.sql
-- Repeatable migration - re-runs when checksum changes
CREATE OR REPLACE VIEW active_products AS
SELECT p.*, u.name as owner_name
FROM products p
JOIN users u ON p.owner_id = u.id
WHERE p.deleted = FALSE AND p.active = TRUE;

CREATE OR REPLACE VIEW product_statistics AS
SELECT
    category,
    COUNT(*) as total_products,
    AVG(price) as avg_price,
    MIN(price) as min_price,
    MAX(price) as max_price
FROM products
WHERE deleted = FALSE
GROUP BY category;
```

---

## 5. Soft Delete {#soft-delete}

### Soft Delete Entity

```java
package com.example.database.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.*;

import java.math.BigDecimal;
import java.time.Instant;

/**
 * @SQLDelete: overrides DELETE with UPDATE to set deleted=true
 * @Where: appends WHERE deleted=false to all queries automatically
 */
@Entity
@Table(name = "products")
@SQLDelete(sql = "UPDATE products SET deleted = TRUE, deleted_at = NOW() WHERE id = ?")
@FilterDef(name = "deletedFilter", parameters = @ParamDef(name = "isDeleted", type = Boolean.class))
@Filter(name = "deletedFilter", condition = "deleted = :isDeleted")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Where(clause = "deleted = false")  // Applied automatically to all HQL/JPQL queries
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private String category;

    @Column(name = "owner_id", nullable = false)
    private String ownerId;

    @Column(nullable = false)
    @Builder.Default
    private boolean active = true;

    @Column(nullable = false)
    @Builder.Default
    private boolean deleted = false;

    @Column(name = "deleted_at")
    private Instant deletedAt;

    @Column(name = "deleted_by")
    private String deletedBy;

    @Version
    private Long version;

    @org.hibernate.annotations.CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @org.hibernate.annotations.UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

### Soft Delete Repository

```java
package com.example.database.repository;

import com.example.database.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // These automatically add WHERE deleted=false due to @Where annotation
    Optional<Product> findByIdAndOwnerId(Long id, String ownerId);
    List<Product> findByCategoryAndActiveTrue(String category);

    // Explicitly query deleted items (bypasses @Where)
    @Query(value = "SELECT * FROM products WHERE deleted = TRUE", nativeQuery = true)
    List<Product> findAllDeleted();

    // Find including deleted (bypass @Where)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdIncludingDeleted(@Param("id") Long id);

    // Hard delete for testing (USE WITH CAUTION)
    @Modifying
    @Query(value = "DELETE FROM products WHERE id = :id", nativeQuery = true)
    void hardDelete(@Param("id") Long id);

    // Restore a soft-deleted item
    @Modifying
    @Query("UPDATE Product p SET p.deleted = false, p.deletedAt = null, p.deletedBy = null WHERE p.id = :id")
    int restore(@Param("id") Long id);
}
```

### Soft Delete Service

```java
package com.example.database.service;

import com.example.database.entity.Product;
import com.example.database.repository.ProductRepository;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.hibernate.Session;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class SoftDeleteProductService {

    private final ProductRepository productRepository;
    private final EntityManager entityManager;

    /**
     * Soft delete: sets deleted=true, records who/when
     * The @SQLDelete annotation handles the actual SQL
     */
    @Transactional
    public void softDelete(Long productId, String deletedBy) {
        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new RuntimeException("Product not found: " + productId));

        product.setDeletedBy(deletedBy);
        product.setDeletedAt(Instant.now());
        productRepository.delete(product);
        // ↑ This calls: UPDATE products SET deleted=TRUE, deleted_at=NOW() WHERE id=?

        log.info("Soft deleted product: id={}, by={}", productId, deletedBy);
    }

    /**
     * Restore a soft-deleted product
     */
    @Transactional
    public void restore(Long productId) {
        int updated = productRepository.restore(productId);
        if (updated == 0) {
            throw new RuntimeException("Product not found or not deleted: " + productId);
        }
        log.info("Restored product: id={}", productId);
    }

    /**
     * Query including deleted items using Hibernate Filter
     */
    @Transactional(readOnly = true)
    public List<Product> findIncludingDeleted() {
        Session session = entityManager.unwrap(Session.class);

        // Disable the @Where filter temporarily
        session.enableFilter("deletedFilter").setParameter("isDeleted", true);

        List<Product> deletedProducts = productRepository.findAllDeleted();

        // Re-enable filter
        session.disableFilter("deletedFilter");

        return deletedProducts;
    }
}
```

---

## 6. Auditing with Hibernate Envers {#envers}

### Envers Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

```java
package com.example.database.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;
import org.hibernate.envers.RelationTargetAuditMode;

import java.math.BigDecimal;
import java.time.Instant;

/**
 * @Audited: Envers creates an audit table (products_aud)
 * and records every insert/update/delete with a revision number
 */
@Entity
@Table(name = "products")
@Audited
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuditedProduct {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private String category;

    // Audit the foreign key, not the full relation
    @Audited(targetAuditMode = RelationTargetAuditMode.NOT_AUDITED)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;

    // Skip auditing of computed/derived fields
    @NotAudited
    @Column(name = "search_vector")
    private String searchVector;

    @Column(nullable = false)
    @Builder.Default
    private boolean active = true;

    @Version
    private Long version;
}
```

### Custom Revision Entity (Track Who Changed What)

```java
package com.example.database.audit;

import jakarta.persistence.*;
import lombok.Data;
import org.hibernate.envers.RevisionEntity;
import org.hibernate.envers.RevisionNumber;
import org.hibernate.envers.RevisionTimestamp;

@Entity
@Table(name = "revisions")
@RevisionEntity(CustomRevisionListener.class)
@Data
public class CustomRevision {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "revision_seq")
    @SequenceGenerator(name = "revision_seq", sequenceName = "revision_sequence", allocationSize = 1)
    @RevisionNumber
    private Long id;

    @RevisionTimestamp
    @Column(name = "revision_timestamp")
    private Long timestamp;

    @Column(name = "changed_by")
    private String changedBy;  // User who made the change

    @Column(name = "change_reason")
    private String changeReason;  // Optional: reason for the change

    @Column(name = "ip_address")
    private String ipAddress;
}
```

```java
package com.example.database.audit;

import org.hibernate.envers.RevisionListener;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

public class CustomRevisionListener implements RevisionListener {

    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevision revision = (CustomRevision) revisionEntity;

        // Set the current user
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setChangedBy(auth.getName());
        } else {
            revision.setChangedBy("system");
        }

        // Set IP address
        var requestAttributes = RequestContextHolder.getRequestAttributes();
        if (requestAttributes instanceof ServletRequestAttributes sra) {
            String xff = sra.getRequest().getHeader("X-Forwarded-For");
            revision.setIpAddress(xff != null ? xff.split(",")[0].trim()
                    : sra.getRequest().getRemoteAddr());
        }
    }
}
```

### Querying Audit History

```java
package com.example.database.service;

import com.example.database.audit.CustomRevision;
import com.example.database.entity.AuditedProduct;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;
import org.hibernate.envers.query.AuditEntity;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.Date;
import java.util.List;

@Service
@RequiredArgsConstructor
public class AuditService {

    private final EntityManager entityManager;

    /**
     * Get full revision history for a product
     */
    @Transactional(readOnly = true)
    public List<Object[]> getProductHistory(Long productId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        return reader.createQuery()
                .forRevisionsOfEntity(AuditedProduct.class, false, true)
                .add(AuditEntity.id().eq(productId))
                .addOrder(AuditEntity.revisionNumber().asc())
                .getResultList();
    }

    /**
     * Get product state at a specific point in time
     */
    @Transactional(readOnly = true)
    public AuditedProduct getProductAtTime(Long productId, Instant at) {
        AuditReader reader = AuditReaderFactory.get(entityManager);
        return reader.find(AuditedProduct.class, productId, Date.from(at));
    }

    /**
     * Get all revisions with change metadata
     */
    @Transactional(readOnly = true)
    public List<Object[]> getProductRevisions(Long productId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        // Returns [AuditedProduct, CustomRevision, RevisionType]
        return reader.createQuery()
                .forRevisionsOfEntityWithChanges(AuditedProduct.class, true)
                .add(AuditEntity.id().eq(productId))
                .addOrder(AuditEntity.revisionNumber().desc())
                .setMaxResults(20)
                .getResultList();
    }

    /**
     * Find who changed a specific field
     */
    @Transactional(readOnly = true)
    public List<Object[]> getPriceChangeHistory(Long productId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        return reader.createQuery()
                .forRevisionsOfEntity(AuditedProduct.class, false, true)
                .add(AuditEntity.id().eq(productId))
                .add(AuditEntity.property("price").hasChanged())
                .addOrder(AuditEntity.revisionNumber().desc())
                .getResultList();
    }
}
```

---

## 7. Full-Text Search with Hibernate Search {#full-text-search}

### Hibernate Search Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.search</groupId>
    <artifactId>hibernate-search-mapper-orm</artifactId>
    <version>7.1.1.Final</version>
</dependency>
<dependency>
    <groupId>org.hibernate.search</groupId>
    <artifactId>hibernate-search-backend-lucene</artifactId>
    <version>7.1.1.Final</version>
</dependency>
```

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        search:
          backend:
            type: lucene
            directory:
              type: local-filesystem
              root: /var/lucene/indexes
            analysis:
              configurer: com.example.database.search.CustomAnalysisConfigurer
```

### Indexed Entity

```java
package com.example.database.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.search.mapper.pojo.mapping.definition.annotation.*;

import java.math.BigDecimal;

@Entity
@Table(name = "products")
@Indexed(index = "products")  // Creates a Lucene/Elasticsearch index
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SearchableProduct {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @FullTextField(analyzer = "english")  // Full-text, stemming, stop words
    @KeywordField(name = "name_sort",
                  normalizer = "lowercase",
                  sortable = org.hibernate.search.engine.backend.types.Sortable.YES)
    @Column(nullable = false)
    private String name;

    @FullTextField(analyzer = "english")
    private String description;

    @KeywordField  // Exact match (no analysis)
    @Column(nullable = false)
    private String category;

    @GenericField  // For range queries and sorting
    @Column(nullable = false)
    private BigDecimal price;

    @GenericField
    @Column(nullable = false)
    private boolean active;

    @IndexedEmbedded(includeDepth = 1)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id")
    private User owner;
}
```

### Custom Analysis Configuration

```java
package com.example.database.search;

import org.hibernate.search.backend.lucene.analysis.LuceneAnalysisConfigurationContext;
import org.hibernate.search.backend.lucene.analysis.LuceneAnalysisConfigurer;

public class CustomAnalysisConfigurer implements LuceneAnalysisConfigurer {

    @Override
    public void configure(LuceneAnalysisConfigurationContext context) {
        // English analyzer: tokenize, lowercase, stop words, stem
        context.analyzer("english").custom()
                .tokenizer("standard")
                .tokenFilter("lowercase")
                .tokenFilter("stop", f -> f
                    .param("words", "a,an,the,and,or,for,in,on,to,of,with"))
                .tokenFilter("snowball", f -> f
                    .param("language", "English"));

        // Lowercase normalizer for keywords
        context.normalizer("lowercase").custom()
                .tokenFilter("lowercase");
    }
}
```

### Full-Text Search Service

```java
package com.example.database.service;

import com.example.database.entity.SearchableProduct;
import jakarta.persistence.EntityManager;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.hibernate.search.mapper.orm.Search;
import org.hibernate.search.mapper.orm.session.SearchSession;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final EntityManager entityManager;

    /**
     * Full-text search with multiple criteria
     */
    @Transactional(readOnly = true)
    public List<SearchableProduct> search(String query, String category,
                                           BigDecimal minPrice, BigDecimal maxPrice,
                                           int page, int size) {
        SearchSession searchSession = Search.session(entityManager);

        return searchSession.search(SearchableProduct.class)
                .where(f -> f.bool()
                        // Full-text across name and description
                        .must(f.match()
                                .fields("name", "name^2.0", "description")  // name boosted 2x
                                .matching(query)
                                .fuzzy(1))  // Allow 1 typo
                        // Exact category filter
                        .filter(category != null
                                ? f.match().field("category").matching(category)
                                : f.matchAll())
                        // Price range filter
                        .filter(minPrice != null || maxPrice != null
                                ? f.range().field("price")
                                        .between(minPrice, maxPrice)
                                : f.matchAll())
                        // Only active products
                        .filter(f.match().field("active").matching(true))
                )
                .sort(s -> s.composite()
                        .add(s.score())  // Relevance score first
                        .add(s.field("price").asc().missing().last())
                )
                .fetch(page * size, size)
                .hits();
    }

    /**
     * Suggestions / autocomplete
     */
    @Transactional(readOnly = true)
    public List<String> suggest(String prefix) {
        SearchSession searchSession = Search.session(entityManager);

        return searchSession.search(SearchableProduct.class)
                .select(f -> f.field("name_sort", String.class))
                .where(f -> f.wildcard()
                        .field("name")
                        .matching(prefix.toLowerCase() + "*"))
                .fetch(10)
                .hits();
    }

    /**
     * Reindex all entities (run after bulk data changes)
     */
    @Transactional
    public void reindexAll() throws InterruptedException {
        log.info("Starting mass reindex...");
        Search.session(entityManager)
                .massIndexer(SearchableProduct.class)
                .threadsToLoadObjects(4)
                .batchSizeToLoadObjects(100)
                .startAndWait();
        log.info("Mass reindex complete");
    }
}
```

---

## 8. Time-Series Data with TimescaleDB {#timescaledb}

### TimescaleDB Setup

```sql
-- V10__create_timescale_tables.sql
-- Requires TimescaleDB extension installed in PostgreSQL
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE product_price_history (
    time        TIMESTAMP WITH TIME ZONE NOT NULL,
    product_id  BIGINT NOT NULL,
    price       NUMERIC(10, 2) NOT NULL,
    changed_by  UUID NOT NULL
);

-- Convert to hypertable (partitioned by time)
SELECT create_hypertable('product_price_history', 'time');

-- Add retention policy: auto-drop data older than 2 years
SELECT add_retention_policy('product_price_history', INTERVAL '2 years');

-- Create continuous aggregate for daily stats (auto-updated)
CREATE MATERIALIZED VIEW daily_price_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    product_id,
    FIRST(price, time) AS open_price,
    MAX(price) AS high_price,
    MIN(price) AS low_price,
    LAST(price, time) AS close_price,
    COUNT(*) AS num_changes
FROM product_price_history
GROUP BY bucket, product_id
WITH NO DATA;

-- Refresh policy: update aggregate every hour
SELECT add_continuous_aggregate_policy('daily_price_stats',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Time-Series Repository

```java
package com.example.database.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@Repository
public interface PriceHistoryRepository extends JpaRepository<ProductPriceHistory, Long> {

    /**
     * Get price history using TimescaleDB time_bucket function
     */
    @Query(value = """
        SELECT
            time_bucket('1 hour', time) AS bucket,
            product_id,
            FIRST(price, time) AS first_price,
            LAST(price, time) AS last_price,
            MAX(price) AS max_price,
            MIN(price) AS min_price
        FROM product_price_history
        WHERE product_id = :productId
          AND time >= :from
          AND time <= :to
        GROUP BY bucket, product_id
        ORDER BY bucket ASC
        """, nativeQuery = true)
    List<Object[]> findHourlyPriceHistory(
            @Param("productId") Long productId,
            @Param("from") Instant from,
            @Param("to") Instant to);

    /**
     * Get latest price change
     */
    @Query(value = """
        SELECT * FROM product_price_history
        WHERE product_id = :productId
        ORDER BY time DESC
        LIMIT 1
        """, nativeQuery = true)
    java.util.Optional<ProductPriceHistory> findLatestForProduct(@Param("productId") Long productId);
}
```

---

## 9. JSON Columns with PostgreSQL {#json-columns}

### JSONB Entity

```java
package com.example.database.entity;

import io.hypersistence.utils.hibernate.type.json.JsonBinaryType;
import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.annotations.Type;
import org.hibernate.type.SqlTypes;

import java.util.Map;

@Entity
@Table(name = "products")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductWithJson {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    /**
     * JSONB column: stored as binary JSON in PostgreSQL
     * Supports indexing and GIN/GiST indexes for efficient querying
     */
    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> attributes;  // Dynamic product attributes

    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private ProductMetadata metadata;  // Strongly-typed JSON

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ProductMetadata {
        private String sku;
        private String barcode;
        private Map<String, String> dimensions;
        private java.util.List<String> tags;
        private Map<String, String> localization;
    }
}
```

### JSON Migration

```sql
-- V11__add_json_columns.sql
ALTER TABLE products ADD COLUMN IF NOT EXISTS attributes JSONB;
ALTER TABLE products ADD COLUMN IF NOT EXISTS metadata   JSONB;

-- GIN index for full JSONB search
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Index a specific JSON path for frequent queries
CREATE INDEX idx_products_sku ON products((metadata->>'sku'));
```

### JSON Query Repository

```java
package com.example.database.repository;

import com.example.database.entity.ProductWithJson;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface JsonProductRepository extends JpaRepository<ProductWithJson, Long> {

    /**
     * Query a specific JSON field with PostgreSQL JSONB operators
     * ->> extracts as text
     * @> checks JSON containment
     */
    @Query(value = "SELECT * FROM products WHERE metadata->>'sku' = :sku",
           nativeQuery = true)
    java.util.Optional<ProductWithJson> findBySku(@Param("sku") String sku);

    /**
     * Find products with a specific tag in their JSON array
     */
    @Query(value = "SELECT * FROM products WHERE metadata->'tags' @> :tag::jsonb",
           nativeQuery = true)
    List<ProductWithJson> findByTag(@Param("tag") String tag);

    /**
     * Find products with specific attributes
     * @> is the JSON containment operator
     */
    @Query(value = "SELECT * FROM products WHERE attributes @> :attrs::jsonb",
           nativeQuery = true)
    List<ProductWithJson> findByAttributes(@Param("attrs") String attributesJson);

    /**
     * Count by JSON field value
     */
    @Query(value = "SELECT COUNT(*) FROM products WHERE attributes->>:key = :value",
           nativeQuery = true)
    long countByAttribute(@Param("key") String key, @Param("value") String value);
}
```

---

## 10. Connection Pool Monitoring {#pool-monitoring}

### HikariCP Metrics with Micrometer

```java
package com.example.database.monitoring;

import com.zaxxer.hikari.HikariDataSource;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@Slf4j
@Component("hikariPool")
@RequiredArgsConstructor
public class DatabasePoolHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;
    private final MeterRegistry meterRegistry;

    @Override
    public Health health() {
        if (!(dataSource instanceof HikariDataSource hikari)) {
            return Health.unknown().withDetail("type", "non-hikari").build();
        }

        var pool = hikari.getHikariPoolMXBean();
        if (pool == null) {
            return Health.down().withDetail("reason", "Pool not initialized").build();
        }

        int active = pool.getActiveConnections();
        int idle = pool.getIdleConnections();
        int total = pool.getTotalConnections();
        int waiting = pool.getThreadsAwaitingConnection();
        int maxSize = hikari.getMaximumPoolSize();

        float utilization = total > 0 ? (float) active / maxSize * 100 : 0;

        Health.Builder builder = utilization > 90 ? Health.outOfService() : Health.up();

        return builder
                .withDetail("pool", hikari.getPoolName())
                .withDetail("active", active)
                .withDetail("idle", idle)
                .withDetail("total", total)
                .withDetail("waiting", waiting)
                .withDetail("maxSize", maxSize)
                .withDetail("utilization", String.format("%.1f%%", utilization))
                .build();
    }

    @Scheduled(fixedDelay = 30_000)
    public void logPoolStats() {
        if (!(dataSource instanceof HikariDataSource hikari)) return;

        var pool = hikari.getHikariPoolMXBean();
        if (pool == null) return;

        int active = pool.getActiveConnections();
        int maxSize = hikari.getMaximumPoolSize();
        float utilization = (float) active / maxSize * 100;

        if (utilization > 80) {
            log.warn("High connection pool utilization: {:.1f}% ({}/{})",
                    utilization, active, maxSize);
        } else {
            log.debug("Pool stats: active={}/{}, idle={}, waiting={}",
                    active, maxSize, pool.getIdleConnections(),
                    pool.getThreadsAwaitingConnection());
        }
    }
}
```

---

## 11. Real Example: E-Commerce with Soft Delete + Auditing + Full-Text Search {#real-example}

### Product Entity (All Features Combined)

```java
package com.example.database.entity;

import io.hypersistence.utils.hibernate.type.json.JsonBinaryType;
import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.*;
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;
import org.hibernate.search.mapper.pojo.mapping.definition.annotation.*;
import org.hibernate.type.SqlTypes;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Map;
import java.util.Set;

@Entity
@Table(name = "products",
       indexes = {
           @Index(name = "idx_products_category", columnList = "category"),
           @Index(name = "idx_products_owner", columnList = "owner_id"),
           @Index(name = "idx_products_active", columnList = "active,deleted")
       })
@SQLDelete(sql = "UPDATE products SET deleted = TRUE, deleted_at = NOW() WHERE id = ? AND version = ?")
@Where(clause = "deleted = false")
@Audited
@Indexed(index = "products")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ECommerceProduct {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @FullTextField(analyzer = "english")
    @KeywordField(name = "name_sort", normalizer = "lowercase",
                  sortable = org.hibernate.search.engine.backend.types.Sortable.YES)
    @Column(nullable = false, length = 200)
    private String name;

    @FullTextField(analyzer = "english")
    @Column(columnDefinition = "TEXT")
    private String description;

    @GenericField(sortable = org.hibernate.search.engine.backend.types.Sortable.YES)
    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @KeywordField
    @Column(nullable = false, length = 100)
    private String category;

    @GenericField
    @Column(nullable = false)
    @Builder.Default
    private boolean active = true;

    @Column(name = "owner_id", nullable = false)
    private String ownerId;

    // JSONB for flexible attributes (size, color, material, etc.)
    @NotAudited
    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> attributes;

    // Tags as JSON array
    @NotAudited
    @Column(columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Set<String> tags;

    // Soft delete fields
    @Column(nullable = false)
    @Builder.Default
    private boolean deleted = false;

    @Column(name = "deleted_at")
    private Instant deletedAt;

    @Column(name = "deleted_by")
    private String deletedBy;

    // Optimistic locking
    @Version
    private Long version;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;

    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @Column(name = "updated_by")
    private String updatedBy;
}
```

### Product Service (All Patterns)

```java
package com.example.database.service;

import com.example.database.entity.ECommerceProduct;
import com.example.database.repository.ECommerceProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.hibernate.search.mapper.orm.Search;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import jakarta.persistence.EntityManager;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ECommerceProductService {

    private final ECommerceProductRepository productRepository;
    private final EntityManager entityManager;

    /**
     * Create product with audit tracking
     */
    @Transactional
    public ECommerceProduct create(ECommerceProduct product, String userId) {
        product.setCreatedBy(userId);
        product.setUpdatedBy(userId);
        ECommerceProduct saved = productRepository.save(product);
        log.info("Created product: id={}, name={}", saved.getId(), saved.getName());
        return saved;
    }

    /**
     * Update with optimistic locking + retry
     */
    @Transactional
    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2.0)
    )
    public ECommerceProduct update(Long id, ECommerceProduct updates, String userId) {
        ECommerceProduct product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found: " + id));

        product.setName(updates.getName());
        product.setDescription(updates.getDescription());
        product.setPrice(updates.getPrice());
        product.setCategory(updates.getCategory());
        product.setAttributes(updates.getAttributes());
        product.setTags(updates.getTags());
        product.setUpdatedBy(userId);

        return productRepository.save(product);
    }

    /**
     * Soft delete with tracking
     */
    @Transactional
    public void delete(Long id, String userId) {
        ECommerceProduct product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found: " + id));

        product.setDeletedBy(userId);
        product.setDeletedAt(Instant.now());
        productRepository.delete(product);  // @SQLDelete handles UPDATE
        log.info("Soft deleted product: id={}, by={}", id, userId);
    }

    /**
     * Full-text search
     */
    @Transactional(readOnly = true)
    public List<ECommerceProduct> search(String query, String category,
                                          BigDecimal minPrice, BigDecimal maxPrice) {
        return Search.session(entityManager)
                .search(ECommerceProduct.class)
                .where(f -> f.bool()
                        .must(f.match()
                                .fields("name^3.0", "description", "category")
                                .matching(query)
                                .fuzzy(1))
                        .filter(category != null
                                ? f.match().field("category").matching(category)
                                : f.matchAll())
                        .filter(minPrice != null || maxPrice != null
                                ? f.range().field("price").between(minPrice, maxPrice)
                                : f.matchAll())
                        .filter(f.match().field("active").matching(true))
                )
                .sort(s -> s.composite()
                        .add(s.score())
                        .add(s.field("price").asc()))
                .fetch(20)
                .hits();
    }

    /**
     * Read with replica routing
     */
    @Transactional(readOnly = true)
    public Page<ECommerceProduct> findAll(Pageable pageable) {
        return productRepository.findAll(pageable);
    }
}
```

---

## 12. Summary {#summary}

| Pattern | Use Case | Key Annotation/Config |
|---------|---------|----------------------|
| **Sharding** | Horizontal scaling beyond single DB | `AbstractRoutingDataSource` + ThreadLocal key |
| **Read Replicas** | Scale reads; offload analytics | `@Transactional(readOnly=true)` → replica |
| **Optimistic Locking** | Low-contention; detect conflicts late | `@Version` + `@Retryable` |
| **Pessimistic Locking** | High-contention; prevent conflicts | `@Lock(PESSIMISTIC_WRITE)` |
| **Flyway** | Versioned, repeatable schema migrations | `V{n}__{desc}.sql`, `R__{desc}.sql` |
| **Soft Delete** | Keep deleted data; GDPR audit trail | `@SQLDelete` + `@Where(deleted=false)` |
| **Hibernate Envers** | Full change history with who/when | `@Audited` + custom revision entity |
| **Hibernate Search** | Relevance search, fuzzy, autocomplete | `@Indexed`, `@FullTextField` + Lucene |
| **TimescaleDB** | Time-series metrics, price history | `create_hypertable()` + continuous aggregates |
| **JSON Columns** | Flexible dynamic attributes | `@JdbcTypeCode(SqlTypes.JSON)` + JSONB |
| **Pool Monitoring** | Detect connection exhaustion | HikariPoolMXBean + Micrometer metrics |

### Best Practices Checklist

```
[ ] Use @Transactional(readOnly=true) for all reads (enables replica routing)
[ ] Never use ddl-auto=create/update in production (use Flyway only)
[ ] Always set clean-disabled=true in Flyway production config
[ ] Apply @Where filter for soft delete; handle admin queries with native SQL
[ ] Use @Version for optimistic locking on frequently-updated entities
[ ] Use pessimistic locking only when conflict rate > 20-30%
[ ] Index JSONB columns with GIN for efficient querying
[ ] Set add_retention_policy on TimescaleDB hypertables
[ ] Monitor connection pool utilization; alert at > 80%
[ ] Run mass reindex after bulk data imports
```

---

> **Next: Part 052 - WebSocket and Real-time Features** — STOMP protocol, Spring WebSocket, JWT auth, Redis pub/sub for scaling, SSE, and a collaborative document editor example.
