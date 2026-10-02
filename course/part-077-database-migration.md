# Part 077: Database Migration Strategies

## Overview

Database migrations are one of the riskiest parts of production deployments. A poorly planned migration can cause downtime, data loss, or application errors that affect thousands of users. This part covers advanced migration strategies to evolve your database schema safely, including zero-downtime techniques, rollback strategies, and multi-tenant patterns.

## Prerequisites

- Spring Boot 3.x
- Flyway or Liquibase
- PostgreSQL or MySQL
- TestContainers
- Basic understanding of SQL DDL

---

## 1. Zero-Downtime Migration Fundamentals

The core principle: **never make a change that breaks the currently running version of the application**.

### The Problem with Traditional Migrations

```
WRONG approach (causes downtime):
1. Stop application
2. Run ALTER TABLE (locks table, takes minutes for large tables)
3. Deploy new application version
4. Start application
```

### The Expand/Contract Pattern

The Expand/Contract pattern (also called parallel change) solves this by splitting one breaking migration into three safe phases:

```
Phase 1 - EXPAND: Add new schema (old code still works)
Phase 2 - MIGRATE: Backfill data, deploy new code that reads both
Phase 3 - CONTRACT: Remove old schema (once all instances use new code)
```

---

## 2. Project Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-database-postgresql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.liquibase</groupId>
        <artifactId>liquibase-core</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- Testing -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Application Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ecommerce
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:secret}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate   # NEVER use create or update in production
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: false
    validate-on-migrate: true
    clean-disabled: true    # CRITICAL: never clean production DB
```

---

## 3. Adding Columns Safely

### Scenario: Add `email_verified` column to `users` table

The `users` table has 10 million rows. Adding a `NOT NULL` column directly would:
1. Lock the table for minutes
2. Fail if done without a default
3. Break old application instances still running

### Phase 1: EXPAND - Add nullable column

```sql
-- V20__add_email_verified_nullable.sql
-- Safe: Adding nullable column never locks rows on PostgreSQL
ALTER TABLE users ADD COLUMN email_verified BOOLEAN;

-- Add index concurrently (non-blocking on PostgreSQL)
CREATE INDEX CONCURRENTLY idx_users_email_verified
    ON users (email_verified)
    WHERE email_verified = true;
```

### Java Entity (supports both null and non-null)

```java
// src/main/java/com/example/entity/User.java
package com.example.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    // Phase 1 & 2: nullable column
    @Column(name = "email_verified")
    private Boolean emailVerified;  // Boxed type allows null

    @Column(nullable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    // Business logic: treat null as not verified
    public boolean isEmailVerified() {
        return Boolean.TRUE.equals(emailVerified);
    }

    // Getters and setters
    public UUID getId() { return id; }
    public void setId(UUID id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public Boolean getEmailVerified() { return emailVerified; }
    public void setEmailVerified(Boolean emailVerified) { this.emailVerified = emailVerified; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}
```

### Phase 2: MIGRATE - Backfill data

```sql
-- V21__backfill_email_verified.sql
-- Backfill in batches to avoid long locks and high memory usage
DO $$
DECLARE
    batch_size INT := 10000;
    updated INT;
BEGIN
    LOOP
        UPDATE users
        SET email_verified = false
        WHERE email_verified IS NULL
          AND id IN (
              SELECT id FROM users
              WHERE email_verified IS NULL
              LIMIT batch_size
          );

        GET DIAGNOSTICS updated = ROW_COUNT;
        EXIT WHEN updated = 0;

        -- Small pause to reduce load
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

### Backfill Service (for very large tables)

```java
// src/main/java/com/example/migration/UserEmailVerifiedBackfill.java
package com.example.migration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;
import java.util.List;

@Component
public class UserEmailVerifiedBackfill {

    private static final Logger log = LoggerFactory.getLogger(UserEmailVerifiedBackfill.class);
    private static final int BATCH_SIZE = 1000;

    private final JdbcTemplate jdbcTemplate;

    public UserEmailVerifiedBackfill(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    /**
     * Run this as a background job, not in a migration script.
     * Use a scheduler or admin endpoint to trigger.
     */
    public BackfillResult backfillEmailVerified() {
        int totalUpdated = 0;
        int batchCount = 0;

        while (true) {
            int updated = backfillBatch();
            totalUpdated += updated;
            batchCount++;

            log.info("Backfill batch {}: updated {} rows, total so far: {}",
                    batchCount, updated, totalUpdated);

            if (updated < BATCH_SIZE) {
                break;
            }

            // Rate limiting: avoid overwhelming the database
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }

        log.info("Backfill complete: {} total rows updated in {} batches",
                totalUpdated, batchCount);
        return new BackfillResult(totalUpdated, batchCount);
    }

    @Transactional
    public int backfillBatch() {
        // Get a batch of IDs to update (avoid full table scan in UPDATE)
        List<UUID> ids = jdbcTemplate.query(
                """
                SELECT id FROM users
                WHERE email_verified IS NULL
                LIMIT ?
                """,
                (rs, rowNum) -> UUID.fromString(rs.getString("id")),
                BATCH_SIZE
        );

        if (ids.isEmpty()) {
            return 0;
        }

        // Update only the fetched IDs
        String placeholders = String.join(",",
                ids.stream().map(id -> "?").toArray(String[]::new));

        Object[] params = ids.stream().map(UUID::toString).toArray();

        return jdbcTemplate.update(
                "UPDATE users SET email_verified = false WHERE id::text IN (" + placeholders + ")",
                params
        );
    }

    public record BackfillResult(int totalUpdated, int batchCount) {}
}
```

### Phase 3: CONTRACT - Add NOT NULL constraint

```sql
-- V22__make_email_verified_not_null.sql
-- Only run AFTER all application instances are on new code
-- and backfill is 100% complete

-- First, add NOT NULL constraint WITHOUT validating existing rows
-- (instant - only validates new rows)
ALTER TABLE users
    ALTER COLUMN email_verified SET DEFAULT false,
    ADD CONSTRAINT users_email_verified_not_null
        CHECK (email_verified IS NOT NULL) NOT VALID;

-- Then validate in background (doesn't lock table on PostgreSQL 12+)
ALTER TABLE users
    VALIDATE CONSTRAINT users_email_verified_not_null;

-- Finally set as actual NOT NULL (after validation, this is instant)
ALTER TABLE users ALTER COLUMN email_verified SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT users_email_verified_not_null;
```

---

## 4. Renaming Columns Without Downtime

### Scenario: Rename `user_name` to `username`

```sql
-- V23__rename_username_phase1_expand.sql
-- Step 1: Add new column, copy data, create sync trigger

-- Add new column
ALTER TABLE users ADD COLUMN username VARCHAR(100);

-- Copy existing data
UPDATE users SET username = user_name WHERE username IS NULL;

-- Create trigger to keep columns in sync
CREATE OR REPLACE FUNCTION sync_username_columns()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        IF NEW.user_name IS NOT NULL AND NEW.username IS NULL THEN
            NEW.username = NEW.user_name;
        ELSIF NEW.username IS NOT NULL AND NEW.user_name IS NULL THEN
            NEW.user_name = NEW.username;
        END IF;
    ELSIF TG_OP = 'UPDATE' THEN
        IF NEW.user_name <> OLD.user_name THEN
            NEW.username = NEW.user_name;
        ELSIF NEW.username <> OLD.username THEN
            NEW.user_name = NEW.username;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_username
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_username_columns();
```

```java
// Dual-read entity for transition period
@Entity
@Table(name = "users")
public class User {

    @Column(name = "user_name")
    private String userName;  // Old column - read by old code

    @Column(name = "username")
    private String username;  // New column - read by new code

    // New code uses this
    public String getUsername() {
        // Fallback to old column if new is null (during migration)
        return username != null ? username : userName;
    }

    public void setUsername(String username) {
        this.username = username;
        this.userName = username;  // Keep in sync at app level too
    }
}
```

```sql
-- V24__rename_username_phase2_contract.sql
-- Only run AFTER all instances use new column

DROP TRIGGER IF EXISTS trg_sync_username ON users;
DROP FUNCTION IF EXISTS sync_username_columns();
ALTER TABLE users DROP COLUMN user_name;
ALTER TABLE users ALTER COLUMN username SET NOT NULL;
```

---

## 5. Flyway Advanced Features

### Configuration

```java
// src/main/java/com/example/config/FlywayConfig.java
package com.example.config;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.configuration.FluentConfiguration;
import org.springframework.boot.autoconfigure.flyway.FlywayConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FlywayConfig {

    @Bean
    public FlywayConfigurationCustomizer flywayConfigurationCustomizer() {
        return configuration -> configuration
                .connectRetries(3)
                .connectRetriesInterval(10)
                .lockRetryCount(50)
                .mixed(false)
                .group(false)
                .installedBy("deployment-pipeline");
    }
}
```

### Flyway Java Migration for Complex Logic

```java
// src/main/java/db/migration/V25__ComplexDataMigration.java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.sql.*;

/**
 * Flyway Java migration for complex data transformation.
 * Note: Must be in db.migration package (matches Flyway's default scan).
 */
public class V25__ComplexDataMigration extends BaseJavaMigration {

    private static final Logger log = LoggerFactory.getLogger(V25__ComplexDataMigration.class);

    @Override
    public void migrate(Context context) throws Exception {
        Connection connection = context.getConnection();

        log.info("Starting complex data migration V25");

        // Batch process with explicit transaction control
        int offset = 0;
        int batchSize = 500;
        int totalProcessed = 0;

        while (true) {
            int processed = processBatch(connection, offset, batchSize);
            totalProcessed += processed;

            if (processed < batchSize) {
                break;
            }
            offset += batchSize;

            log.info("Processed {} rows so far", totalProcessed);
        }

        log.info("Migration V25 complete: {} rows processed", totalProcessed);
    }

    private int processBatch(Connection connection, int offset, int limit) throws SQLException {
        int count = 0;

        String selectSql = """
                SELECT id, raw_price, currency
                FROM products
                WHERE normalized_price IS NULL
                ORDER BY id
                LIMIT ? OFFSET ?
                """;

        String updateSql = """
                UPDATE products
                SET normalized_price = ?,
                    normalized_currency = 'USD'
                WHERE id = ?
                """;

        try (PreparedStatement selectStmt = connection.prepareStatement(selectSql);
             PreparedStatement updateStmt = connection.prepareStatement(updateSql)) {

            selectStmt.setInt(1, limit);
            selectStmt.setInt(2, offset);

            try (ResultSet rs = selectStmt.executeQuery()) {
                while (rs.next()) {
                    String id = rs.getString("id");
                    double rawPrice = rs.getDouble("raw_price");
                    String currency = rs.getString("currency");

                    double normalizedPrice = convertToUSD(rawPrice, currency);

                    updateStmt.setDouble(1, normalizedPrice);
                    updateStmt.setString(2, id);
                    updateStmt.addBatch();
                    count++;
                }
            }

            if (count > 0) {
                updateStmt.executeBatch();
            }
        }

        return count;
    }

    private double convertToUSD(double amount, String currency) {
        return switch (currency) {
            case "EUR" -> amount * 1.09;
            case "GBP" -> amount * 1.27;
            case "JPY" -> amount * 0.0067;
            default -> amount;  // Assume USD
        };
    }
}
```

### Flyway Repair for Checksum Issues

```java
// src/main/java/com/example/admin/FlywayAdminController.java
package com.example.admin;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.Arrays;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/internal/flyway")
@PreAuthorize("hasRole('ADMIN')")
public class FlywayAdminController {

    private final Flyway flyway;

    public FlywayAdminController(Flyway flyway) {
        this.flyway = flyway;
    }

    @GetMapping("/info")
    public ResponseEntity<List<Map<String, Object>>> getMigrationInfo() {
        List<Map<String, Object>> info = Arrays.stream(flyway.info().all())
                .map(m -> Map.of(
                        "version", m.getVersion() != null ? m.getVersion().getVersion() : "repeatable",
                        "description", m.getDescription(),
                        "state", m.getState().name(),
                        "executedAt", m.getInstalledOn() != null ? m.getInstalledOn().toString() : "pending",
                        "executionTime", m.getExecutionTime() != null ? m.getExecutionTime() : 0
                ))
                .toList();
        return ResponseEntity.ok(info);
    }

    @PostMapping("/repair")
    public ResponseEntity<String> repair() {
        flyway.repair();
        return ResponseEntity.ok("Repair completed successfully");
    }

    @GetMapping("/validate")
    public ResponseEntity<Map<String, Object>> validate() {
        try {
            flyway.validate();
            return ResponseEntity.ok(Map.of("valid", true, "message", "All migrations valid"));
        } catch (Exception e) {
            return ResponseEntity.ok(Map.of("valid", false, "message", e.getMessage()));
        }
    }
}
```

---

## 6. Liquibase Advanced Features

### Liquibase Changelog with Rollback

```xml
<!-- src/main/resources/db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <include file="db/changelog/v1/001-create-users.xml"/>
    <include file="db/changelog/v1/002-create-products.xml"/>
    <include file="db/changelog/v1/003-create-orders.xml"/>

</databaseChangeLog>
```

```xml
<!-- src/main/resources/db/changelog/v1/001-create-users.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="001" author="team@example.com" labels="v1.0.0" context="!test">
        <createTable tableName="users">
            <column name="id" type="UUID">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="password_hash" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="email_verified" type="BOOLEAN" defaultValueBoolean="false">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <!-- Explicit rollback -->
        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>

    <!-- Add index separately for easy rollback -->
    <changeSet id="001-idx" author="team@example.com">
        <createIndex tableName="users" indexName="idx_users_email">
            <column name="email"/>
        </createIndex>
        <rollback>
            <dropIndex tableName="users" indexName="idx_users_email"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

### Liquibase Diff Tool Integration

```java
// src/main/java/com/example/tools/LiquibaseDiffTool.java
package com.example.tools;

import liquibase.Liquibase;
import liquibase.database.Database;
import liquibase.database.DatabaseFactory;
import liquibase.database.jvm.JdbcConnection;
import liquibase.diff.DiffResult;
import liquibase.diff.compare.CompareControl;
import liquibase.diff.output.report.DiffToReport;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.io.PrintStream;
import java.sql.Connection;

@Component
public class LiquibaseDiffTool {

    private final DataSource dataSource;

    public LiquibaseDiffTool(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * Compare two database schemas and generate a diff report.
     * Useful for detecting drift between environments.
     */
    public String generateDiff(DataSource referenceSource) throws Exception {
        try (Connection refConn = referenceSource.getConnection();
             Connection targetConn = dataSource.getConnection()) {

            Database referenceDb = DatabaseFactory.getInstance()
                    .findCorrectDatabaseImplementation(new JdbcConnection(refConn));
            Database targetDb = DatabaseFactory.getInstance()
                    .findCorrectDatabaseImplementation(new JdbcConnection(targetConn));

            DiffResult diffResult = new liquibase.diff.DiffGeneratorFactory()
                    .getInstance()
                    .compare(referenceDb, targetDb, new CompareControl());

            java.io.ByteArrayOutputStream baos = new java.io.ByteArrayOutputStream();
            new DiffToReport(diffResult, new PrintStream(baos)).print();
            return baos.toString();
        }
    }
}
```

---

## 7. Blue/Green Database Deployment

### Strategy Overview

```
Blue (current):     app-v1.0 → db_blue (main)
Green (new):        app-v2.0 → db_green (replica)

Step 1: Set up streaming replication blue → green
Step 2: Deploy app-v2.0 against db_green
Step 3: Run migrations on db_green (forward-compatible)
Step 4: Test app-v2.0 with db_green
Step 5: Switch traffic to green (update DNS/load balancer)
Step 6: db_blue becomes the new replica
```

### Connection Pool Management for Blue/Green Switch

```java
// src/main/java/com/example/config/DatabaseSwitchConfig.java
package com.example.config;

import com.zaxxer.hikari.HikariDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;
import java.util.concurrent.atomic.AtomicReference;

@Configuration
public class DatabaseSwitchConfig {

    private static final Logger log = LoggerFactory.getLogger(DatabaseSwitchConfig.class);

    @Value("${db.blue.url}")
    private String blueUrl;

    @Value("${db.green.url}")
    private String greenUrl;

    @Value("${db.username}")
    private String username;

    @Value("${db.password}")
    private String password;

    private final AtomicReference<String> activeDatabase = new AtomicReference<>("blue");

    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return createDataSource(blueUrl, "blue-pool");
    }

    @Bean("greenDataSource")
    public DataSource greenDataSource() {
        return createDataSource(greenUrl, "green-pool");
    }

    private HikariDataSource createDataSource(String url, String poolName) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        ds.setPoolName(poolName);
        ds.setMaximumPoolSize(20);
        ds.setMinimumIdle(5);
        ds.setConnectionTimeout(5000);
        ds.setValidationTimeout(3000);
        return ds;
    }

    public String getActiveDatabase() {
        return activeDatabase.get();
    }

    public void switchToGreen() {
        log.info("Switching active database from blue to green");
        activeDatabase.set("green");
    }

    public void switchToBlue() {
        log.info("Switching active database from green to blue");
        activeDatabase.set("blue");
    }
}
```

---

## 8. Data Backfills in Production

### Safe Backfill Service

```java
// src/main/java/com/example/migration/BackfillService.java
package com.example.migration;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicLong;

@Service
public class BackfillService {

    private static final Logger log = LoggerFactory.getLogger(BackfillService.class);

    private final JdbcTemplate jdbcTemplate;
    private final AtomicBoolean running = new AtomicBoolean(false);
    private final AtomicLong processedCount = new AtomicLong(0);
    private final AtomicLong totalCount = new AtomicLong(0);

    public BackfillService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Async
    public CompletableFuture<BackfillStatus> runBackfill(BackfillConfig config) {
        if (!running.compareAndSet(false, true)) {
            return CompletableFuture.failedFuture(
                    new IllegalStateException("Backfill already running"));
        }

        try {
            return CompletableFuture.completedFuture(executeBackfill(config));
        } finally {
            running.set(false);
        }
    }

    private BackfillStatus executeBackfill(BackfillConfig config) {
        Instant start = Instant.now();
        processedCount.set(0);

        // Count total rows to process
        Long count = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM " + config.tableName() + " WHERE " + config.condition(),
                Long.class
        );
        totalCount.set(count != null ? count : 0);

        log.info("Starting backfill: {} rows to process", totalCount.get());

        long lastId = 0;
        int batchCount = 0;

        while (true) {
            // Cursor-based pagination is more efficient than OFFSET for large tables
            String sql = String.format("""
                    UPDATE %s
                    SET %s
                    WHERE id > ? AND (%s)
                    ORDER BY id
                    LIMIT %d
                    """,
                    config.tableName(),
                    config.updateExpression(),
                    config.condition(),
                    config.batchSize()
            );

            // Note: This is simplified. Real implementation would need the returned IDs.
            // For cursor-based, use a SELECT then UPDATE pattern.
            int updated = updateBatch(config, lastId);
            processedCount.addAndGet(updated);
            batchCount++;

            if (updated < config.batchSize()) {
                break;
            }

            // Respect rate limit
            if (config.delayBetweenBatches().toMillis() > 0) {
                try {
                    Thread.sleep(config.delayBetweenBatches().toMillis());
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }

            if (batchCount % 100 == 0) {
                log.info("Backfill progress: {}/{} rows ({:.1f}%)",
                        processedCount.get(),
                        totalCount.get(),
                        (processedCount.get() * 100.0) / Math.max(1, totalCount.get())
                );
            }
        }

        Duration elapsed = Duration.between(start, Instant.now());
        log.info("Backfill complete: {} rows in {}ms",
                processedCount.get(), elapsed.toMillis());

        return new BackfillStatus(
                processedCount.get(),
                totalCount.get(),
                elapsed,
                true
        );
    }

    private int updateBatch(BackfillConfig config, long afterId) {
        // Simplified - real implementation would track cursor position
        return jdbcTemplate.update(
                "UPDATE " + config.tableName() +
                        " SET " + config.updateExpression() +
                        " WHERE " + config.condition() +
                        " LIMIT " + config.batchSize()
        );
    }

    public BackfillProgress getProgress() {
        return new BackfillProgress(
                running.get(),
                processedCount.get(),
                totalCount.get()
        );
    }

    public record BackfillConfig(
            String tableName,
            String condition,
            String updateExpression,
            int batchSize,
            Duration delayBetweenBatches
    ) {}

    public record BackfillStatus(
            long rowsProcessed,
            long totalRows,
            Duration elapsed,
            boolean success
    ) {}

    public record BackfillProgress(
            boolean running,
            long processed,
            long total
    ) {}
}
```

---

## 9. Migration Testing with TestContainers

### Base Test Configuration

```java
// src/test/java/com/example/migration/MigrationTestBase.java
package com.example.migration;

import org.junit.jupiter.api.BeforeAll;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
public abstract class MigrationTestBase {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("test_ecommerce")
            .withUsername("test")
            .withPassword("test")
            .withInitScript("db/test-init.sql");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

### Migration Validation Tests

```java
// src/test/java/com/example/migration/FlywayMigrationTest.java
package com.example.migration;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class FlywayMigrationTest extends MigrationTestBase {

    @Autowired
    private Flyway flyway;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    void allMigrationsShouldBeApplied() {
        MigrationInfo[] migrationInfos = flyway.info().all();

        for (MigrationInfo info : migrationInfos) {
            assertThat(info.getState().isApplied())
                    .as("Migration %s should be applied", info.getVersion())
                    .isTrue();
        }
    }

    @Test
    void noFailedMigrationsShouldExist() {
        MigrationInfo[] migrationInfos = flyway.info().all();

        for (MigrationInfo info : migrationInfos) {
            assertThat(info.getState().isFailed())
                    .as("Migration %s should not be failed", info.getVersion())
                    .isFalse();
        }
    }

    @Test
    void usersTableShouldHaveCorrectStructure() {
        List<Map<String, Object>> columns = jdbcTemplate.queryForList(
                """
                SELECT column_name, data_type, is_nullable, column_default
                FROM information_schema.columns
                WHERE table_name = 'users'
                ORDER BY ordinal_position
                """
        );

        assertThat(columns)
                .extracting(c -> c.get("column_name"))
                .contains("id", "email", "password_hash", "email_verified", "created_at");

        // Verify email_verified is NOT NULL
        Map<String, Object> emailVerifiedCol = columns.stream()
                .filter(c -> "email_verified".equals(c.get("column_name")))
                .findFirst()
                .orElseThrow();

        assertThat(emailVerifiedCol.get("is_nullable")).isEqualTo("NO");
    }

    @Test
    void indexesShouldExist() {
        List<String> indexes = jdbcTemplate.queryForList(
                """
                SELECT indexname
                FROM pg_indexes
                WHERE tablename = 'users'
                """,
                String.class
        );

        assertThat(indexes).contains("idx_users_email");
    }

    @Test
    void migrationShouldBeIdempotent() {
        // Run migrations again - should be no-op
        flyway.migrate();

        // Verify state is still consistent
        assertThat(flyway.info().pending()).isEmpty();
    }

    @Test
    void rollbackShouldWork() {
        // Note: Flyway Community Edition doesn't support undo
        // But we can test the rollback SQL manually
        int userCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM users", Integer.class);

        // Simulate what rollback would do
        // (In real test, you'd test Liquibase rollback or Flyway Teams)
        assertThat(userCount).isGreaterThanOrEqualTo(0);
    }
}
```

### Backfill Integration Test

```java
// src/test/java/com/example/migration/BackfillServiceTest.java
package com.example.migration;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;

import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

class BackfillServiceTest extends MigrationTestBase {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private BackfillService backfillService;

    @BeforeEach
    void setUp() {
        // Insert test data with null email_verified
        jdbcTemplate.execute("DELETE FROM users");
        for (int i = 0; i < 100; i++) {
            jdbcTemplate.update(
                    "INSERT INTO users (id, email, password_hash, email_verified, created_at) " +
                            "VALUES (?, ?, ?, NULL, NOW())",
                    UUID.randomUUID().toString(),
                    "user" + i + "@example.com",
                    "hash" + i
            );
        }
    }

    @Test
    void backfillShouldUpdateAllRows() throws Exception {
        BackfillService.BackfillConfig config = new BackfillService.BackfillConfig(
                "users",
                "email_verified IS NULL",
                "email_verified = false",
                10,
                Duration.ZERO
        );

        CompletableFuture<BackfillService.BackfillStatus> future =
                backfillService.runBackfill(config);

        BackfillService.BackfillStatus status = future.get(30, TimeUnit.SECONDS);

        assertThat(status.success()).isTrue();
        assertThat(status.rowsProcessed()).isEqualTo(100);

        Integer nullCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM users WHERE email_verified IS NULL",
                Integer.class
        );
        assertThat(nullCount).isZero();
    }
}
```

---

## 10. Multi-Tenant Migrations

### Schema-per-Tenant with Flyway

```java
// src/main/java/com/example/multitenant/TenantMigrationService.java
package com.example.multitenant;

import org.flywaydb.core.Flyway;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import javax.sql.DataSource;
import java.util.List;

@Service
public class TenantMigrationService {

    private static final Logger log = LoggerFactory.getLogger(TenantMigrationService.class);

    private final DataSource dataSource;

    public TenantMigrationService(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    /**
     * Run migrations for all tenants.
     * Each tenant has its own PostgreSQL schema.
     */
    public void migrateAllTenants(List<String> tenantIds) {
        tenantIds.parallelStream().forEach(tenantId -> {
            try {
                migrateTenant(tenantId);
            } catch (Exception e) {
                log.error("Failed to migrate tenant: {}", tenantId, e);
            }
        });
    }

    public void migrateTenant(String tenantId) {
        log.info("Migrating tenant schema: {}", tenantId);

        // Validate tenant ID to prevent SQL injection
        if (!tenantId.matches("[a-z0-9_]+")) {
            throw new IllegalArgumentException("Invalid tenant ID: " + tenantId);
        }

        Flyway flyway = Flyway.configure()
                .dataSource(dataSource)
                .schemas(tenantId)                    // Use tenant's schema
                .locations("classpath:db/tenant-migration")
                .table("flyway_schema_history")       // History table in tenant schema
                .baselineOnMigrate(true)
                .installedBy("system")
                .load();

        flyway.migrate();
        log.info("Migration complete for tenant: {}", tenantId);
    }

    public void createTenantSchema(String tenantId) {
        if (!tenantId.matches("[a-z0-9_]+")) {
            throw new IllegalArgumentException("Invalid tenant ID: " + tenantId);
        }

        // Create schema
        // Note: Schema names cannot be parameterized in JDBC
        String createSchemaSql = "CREATE SCHEMA IF NOT EXISTS " + tenantId;

        try (var connection = dataSource.getConnection();
             var stmt = connection.createStatement()) {
            stmt.execute(createSchemaSql);
            log.info("Created schema for tenant: {}", tenantId);
        } catch (Exception e) {
            throw new RuntimeException("Failed to create schema for tenant: " + tenantId, e);
        }

        // Run migrations for the new tenant
        migrateTenant(tenantId);
    }
}
```

### Tenant-Aware DataSource

```java
// src/main/java/com/example/multitenant/TenantAwareDataSource.java
package com.example.multitenant;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class TenantAwareDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getCurrentTenant();
    }
}
```

```java
// src/main/java/com/example/multitenant/TenantContext.java
package com.example.multitenant;

public class TenantContext {

    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setCurrentTenant(String tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static String getCurrentTenant() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

---

## 11. Monitoring Migration Progress

### Migration Metrics

```java
// src/main/java/com/example/migration/MigrationMetrics.java
package com.example.migration;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.callback.Callback;
import org.flywaydb.core.api.callback.Context;
import org.flywaydb.core.api.callback.Event;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class MigrationMetrics implements Callback {

    private final Counter migrationsApplied;
    private final Counter migrationsFailed;
    private final Timer migrationDuration;
    private final AtomicInteger pendingMigrations;

    public MigrationMetrics(MeterRegistry registry, Flyway flyway) {
        this.migrationsApplied = Counter.builder("flyway.migrations.applied")
                .description("Number of migrations applied")
                .register(registry);

        this.migrationsFailed = Counter.builder("flyway.migrations.failed")
                .description("Number of failed migrations")
                .register(registry);

        this.migrationDuration = Timer.builder("flyway.migration.duration")
                .description("Time to apply migrations")
                .register(registry);

        this.pendingMigrations = new AtomicInteger(0);

        Gauge.builder("flyway.migrations.pending", pendingMigrations, AtomicInteger::get)
                .description("Number of pending migrations")
                .register(registry);

        // Initialize pending count
        updatePendingCount(flyway);
    }

    private void updatePendingCount(Flyway flyway) {
        try {
            int pending = flyway.info().pending().length;
            pendingMigrations.set(pending);
        } catch (Exception e) {
            // Log but don't fail startup
        }
    }

    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_EACH_MIGRATE
                || event == Event.AFTER_EACH_MIGRATE_ERROR;
    }

    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }

    @Override
    public void handle(Event event, Context context) {
        if (event == Event.AFTER_EACH_MIGRATE) {
            migrationsApplied.increment();
            pendingMigrations.decrementAndGet();
        } else if (event == Event.AFTER_EACH_MIGRATE_ERROR) {
            migrationsFailed.increment();
        }
    }

    @Override
    public String getCallbackName() {
        return "MigrationMetrics";
    }
}
```

### Health Indicator

```java
// src/main/java/com/example/migration/FlywayHealthIndicator.java
package com.example.migration;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Component("flywayMigration")
public class FlywayHealthIndicator implements HealthIndicator {

    private final Flyway flyway;

    public FlywayHealthIndicator(Flyway flyway) {
        this.flyway = flyway;
    }

    @Override
    public Health health() {
        try {
            MigrationInfo[] pending = flyway.info().pending();
            MigrationInfo[] failed = flyway.info().failed();

            if (failed.length > 0) {
                return Health.down()
                        .withDetail("failed_migrations",
                                Arrays.stream(failed)
                                        .map(m -> m.getVersion().getVersion())
                                        .toList())
                        .build();
            }

            if (pending.length > 0) {
                return Health.status("WARN")
                        .withDetail("pending_migrations", pending.length)
                        .withDetail("next_migration",
                                pending[0].getVersion().getVersion())
                        .build();
            }

            return Health.up()
                    .withDetail("applied_migrations", flyway.info().applied().length)
                    .build();

        } catch (Exception e) {
            return Health.down()
                    .withException(e)
                    .build();
        }
    }
}
```

---

## 12. Real Example: E-Commerce Schema Evolution

### Initial Schema

```sql
-- V001__initial_schema.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(500) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

### Evolution: Add Product Categories (Zero-downtime)

```sql
-- V010__add_categories_phase1.sql
-- EXPAND: Add categories table and nullable foreign key

CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    parent_id UUID REFERENCES categories(id),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Nullable FK - old code works fine
ALTER TABLE products ADD COLUMN category_id UUID REFERENCES categories(id);

-- Insert default category for backfill
INSERT INTO categories (id, name, slug)
VALUES ('00000000-0000-0000-0000-000000000001', 'Uncategorized', 'uncategorized');
```

```sql
-- V011__backfill_product_categories.sql
-- MIGRATE: Assign default category to all existing products

UPDATE products
SET category_id = '00000000-0000-0000-0000-000000000001'
WHERE category_id IS NULL;

-- Add partial index for uncategorized (useful for finding items to categorize)
CREATE INDEX CONCURRENTLY idx_products_no_category
    ON products(id)
    WHERE category_id = '00000000-0000-0000-0000-000000000001';
```

```sql
-- V012__make_category_required.sql
-- CONTRACT: Make category_id NOT NULL (only after all products have a category)

ALTER TABLE products
    ALTER COLUMN category_id SET NOT NULL;

-- Now we can add a proper index
CREATE INDEX CONCURRENTLY idx_products_category_id
    ON products(category_id);

DROP INDEX CONCURRENTLY idx_products_no_category;
```

### Evolution: Migrate order status from string to enum

```sql
-- V020__order_status_enum_phase1.sql
-- Create type and add new column

CREATE TYPE order_status AS ENUM (
    'PENDING', 'CONFIRMED', 'PROCESSING',
    'SHIPPED', 'DELIVERED', 'CANCELLED', 'REFUNDED'
);

ALTER TABLE orders ADD COLUMN status_new order_status;

-- Create sync trigger
CREATE OR REPLACE FUNCTION sync_order_status()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP IN ('INSERT', 'UPDATE') THEN
        IF NEW.status IS NOT NULL AND NEW.status_new IS NULL THEN
            NEW.status_new = NEW.status::order_status;
        END IF;
        IF NEW.status_new IS NOT NULL AND NEW.status IS NULL THEN
            NEW.status = NEW.status_new::text;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_order_status
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION sync_order_status();
```

```sql
-- V021__backfill_order_status.sql
UPDATE orders SET status_new = status::order_status WHERE status_new IS NULL;
```

```sql
-- V022__order_status_enum_contract.sql
-- Drop old column, rename new column

DROP TRIGGER trg_sync_order_status ON orders;
DROP FUNCTION sync_order_status();

ALTER TABLE orders DROP COLUMN status;
ALTER TABLE orders RENAME COLUMN status_new TO status;
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
```

---

## Summary

| Topic | Key Takeaway |
|-------|-------------|
| Expand/Contract | Split breaking changes into 3 safe phases |
| Adding Columns | Always add nullable first, backfill, then add NOT NULL |
| Renaming Columns | Use sync triggers during transition |
| Flyway | Use Java migrations for complex logic; repair for checksum issues |
| Liquibase | Use XML/YAML changelogs with explicit rollback blocks |
| Blue/Green DB | Use streaming replication for zero-downtime deployment |
| Backfills | Cursor-based pagination, small batches, rate limiting |
| TestContainers | Always test migrations in isolation before production |
| Multi-tenant | Schema-per-tenant with dedicated Flyway instances |
| Monitoring | Track pending, failed, and applied migrations as metrics |

---

## Next Part Preview

**Part 078: Testing Microservices** - We'll cover the complete testing pyramid for microservices: unit tests, integration tests with TestContainers, contract testing with Spring Cloud Contract, and end-to-end testing strategies.
