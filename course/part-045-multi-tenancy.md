# Part 045: Multi-Tenancy with Spring Boot

## Overview

Multi-tenancy allows a single application instance to serve multiple customers (tenants), each with isolated data. This part covers three strategies, focusing on the industry-standard schema-per-tenant approach with practical Spring Boot implementation.

---

## 1. Multi-Tenancy Strategies

| Strategy | Isolation | Cost | Complexity | Best For |
|----------|-----------|------|------------|----------|
| Database per tenant | Highest | Highest | High | Regulated industries, large tenants |
| Schema per tenant | High | Medium | Medium | SaaS, medium isolation needs |
| Row-level (shared tables) | Low | Lowest | Low | Simple apps, many small tenants |

This part focuses on **Schema-per-tenant** as it's the best balance for most SaaS applications.

---

## 2. Project Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/saasdb
    username: saasapp
    password: secret
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_schema: public   # Fallback schema
  flyway:
    enabled: true
    locations: classpath:db/migration/shared
    schemas: public
    baseline-on-migrate: true

app:
  tenants:
    default-schema: public
    allowed: tenant1,tenant2,tenant3  # In prod, fetch from DB
  jwt:
    secret: your-256-bit-secret-here-change-in-production
    expiration: 86400  # 24 hours
```

---

## 3. TenantContext (ThreadLocal-based)

```java
package com.example.multitenant.context;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class TenantContext {

    private static final Logger log = LoggerFactory.getLogger(TenantContext.class);

    // ThreadLocal holds tenant ID for the current request thread
    private static final ThreadLocal<String> CURRENT_TENANT = new InheritableThreadLocal<>();

    private TenantContext() {}

    public static void setCurrentTenant(String tenantId) {
        if (tenantId == null || tenantId.isBlank()) {
            throw new IllegalArgumentException("Tenant ID cannot be blank");
        }
        log.debug("Setting tenant context: {}", tenantId);
        CURRENT_TENANT.set(tenantId);
    }

    public static String getCurrentTenant() {
        return CURRENT_TENANT.get();
    }

    public static boolean hasTenant() {
        return CURRENT_TENANT.get() != null;
    }

    public static void clear() {
        log.debug("Clearing tenant context: {}", CURRENT_TENANT.get());
        CURRENT_TENANT.remove();
    }

    // Try-with-resources support for explicit context management
    public static TenantContextHolder withTenant(String tenantId) {
        String previousTenant = CURRENT_TENANT.get();
        setCurrentTenant(tenantId);
        return () -> {
            if (previousTenant != null) {
                CURRENT_TENANT.set(previousTenant);
            } else {
                CURRENT_TENANT.remove();
            }
        };
    }
}
```

```java
package com.example.multitenant.context;

@FunctionalInterface
public interface TenantContextHolder extends AutoCloseable {
    void close();
}
```

---

## 4. Tenant Resolution

### 4.1 Tenant Resolver Interface

```java
package com.example.multitenant.resolver;

import jakarta.servlet.http.HttpServletRequest;

public interface TenantResolver {
    String resolveTenant(HttpServletRequest request);
}
```

### 4.2 JWT-based Tenant Resolver

```java
package com.example.multitenant.resolver;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;

@Component
public class JwtTenantResolver implements TenantResolver {

    @Value("${app.jwt.secret}")
    private String jwtSecret;

    @Override
    public String resolveTenant(HttpServletRequest request) {
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return null;
        }

        String token = authHeader.substring(7);
        try {
            byte[] keyBytes = Base64.getDecoder().decode(jwtSecret);
            SecretKey key = new SecretKeySpec(keyBytes, "HmacSHA256");

            Claims claims = Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();

            // Tenant ID is stored in JWT claims
            return claims.get("tenantId", String.class);
        } catch (Exception e) {
            return null;
        }
    }
}
```

### 4.3 Subdomain-based Tenant Resolver

```java
package com.example.multitenant.resolver;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;

@Component
public class SubdomainTenantResolver implements TenantResolver {

    private static final String BASE_DOMAIN = "example.com";

    @Override
    public String resolveTenant(HttpServletRequest request) {
        String host = request.getServerName();
        // For subdomain like "acme.example.com" → "acme"
        if (host != null && host.endsWith("." + BASE_DOMAIN)) {
            String subdomain = host.substring(0, host.indexOf("." + BASE_DOMAIN));
            return subdomain.isBlank() ? null : subdomain.toLowerCase();
        }
        return null;
    }
}
```

### 4.4 Header-based Tenant Resolver

```java
package com.example.multitenant.resolver;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;

@Component
public class HeaderTenantResolver implements TenantResolver {

    private static final String TENANT_HEADER = "X-Tenant-ID";

    @Override
    public String resolveTenant(HttpServletRequest request) {
        String tenantId = request.getHeader(TENANT_HEADER);
        return (tenantId != null && !tenantId.isBlank()) ? tenantId.toLowerCase() : null;
    }
}
```

### 4.5 Composite Resolver (try multiple strategies)

```java
package com.example.multitenant.resolver;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class CompositeTenantResolver implements TenantResolver {

    // Ordered list: JWT → Header → Subdomain
    private final List<TenantResolver> resolvers;

    public CompositeTenantResolver(
            JwtTenantResolver jwtResolver,
            HeaderTenantResolver headerResolver,
            SubdomainTenantResolver subdomainResolver
    ) {
        this.resolvers = List.of(jwtResolver, headerResolver, subdomainResolver);
    }

    @Override
    public String resolveTenant(HttpServletRequest request) {
        return resolvers.stream()
            .map(resolver -> resolver.resolveTenant(request))
            .filter(tenant -> tenant != null && !tenant.isBlank())
            .findFirst()
            .orElse(null);
    }
}
```

---

## 5. Tenant Filter (HTTP Request Interception)

```java
package com.example.multitenant.filter;

import com.example.multitenant.context.TenantContext;
import com.example.multitenant.resolver.CompositeTenantResolver;
import com.example.multitenant.service.TenantValidationService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
public class TenantFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(TenantFilter.class);

    private static final List<String> PUBLIC_PATHS = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/actuator/health",
        "/api/tenants/register"
    );

    @Autowired
    private CompositeTenantResolver tenantResolver;

    @Autowired
    private TenantValidationService tenantValidationService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        String requestPath = request.getRequestURI();

        // Skip tenant resolution for public paths
        if (isPublicPath(requestPath)) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            String tenantId = tenantResolver.resolveTenant(request);

            if (tenantId == null) {
                log.warn("No tenant found for request: {}", requestPath);
                response.sendError(HttpServletResponse.SC_BAD_REQUEST,
                    "Tenant identifier is required");
                return;
            }

            // Validate tenant exists and is active
            if (!tenantValidationService.isValidTenant(tenantId)) {
                log.warn("Invalid or inactive tenant: {}", tenantId);
                response.sendError(HttpServletResponse.SC_FORBIDDEN,
                    "Invalid or inactive tenant: " + tenantId);
                return;
            }

            TenantContext.setCurrentTenant(tenantId);
            log.debug("Tenant context set: {}", tenantId);

            filterChain.doFilter(request, response);

        } finally {
            // CRITICAL: Always clear the tenant context to prevent leaks
            TenantContext.clear();
        }
    }

    private boolean isPublicPath(String path) {
        return PUBLIC_PATHS.stream().anyMatch(path::startsWith);
    }
}
```

---

## 6. Schema-based Multi-Tenancy with Hibernate

### 6.1 Hibernate Multi-Tenant Connection Provider

```java
package com.example.multitenant.hibernate;

import com.example.multitenant.context.TenantContext;
import org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

@Component
public class SchemaMultiTenantConnectionProvider
        implements MultiTenantConnectionProvider<String> {

    private static final Logger log = LoggerFactory.getLogger(
        SchemaMultiTenantConnectionProvider.class);

    private static final String DEFAULT_SCHEMA = "public";

    @Autowired
    private DataSource dataSource;

    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSource.getConnection();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        log.debug("Getting connection for tenant: {}", tenantId);
        Connection connection = dataSource.getConnection();

        // Set PostgreSQL search_path to tenant's schema
        String schema = tenantId != null ? tenantId : DEFAULT_SCHEMA;
        connection.setSchema(schema);

        // Alternatively, use SQL: SET search_path = tenantId
        try (var stmt = connection.createStatement()) {
            stmt.execute("SET search_path = " + schema + ", public");
        }

        return connection;
    }

    @Override
    public void releaseConnection(String tenantId, Connection connection) throws SQLException {
        // Reset schema to default before returning to pool
        try (var stmt = connection.createStatement()) {
            stmt.execute("SET search_path = " + DEFAULT_SCHEMA);
        }
        connection.close();
    }

    @Override
    public boolean supportsAggressiveRelease() {
        return false;
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return false;
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        throw new UnsupportedOperationException("Cannot unwrap: " + unwrapType);
    }
}
```

### 6.2 Tenant Identifier Resolver

```java
package com.example.multitenant.hibernate;

import com.example.multitenant.context.TenantContext;
import org.hibernate.context.spi.CurrentTenantIdentifierResolver;
import org.springframework.stereotype.Component;

@Component
public class TenantIdentifierResolver
        implements CurrentTenantIdentifierResolver<String> {

    private static final String DEFAULT_TENANT = "public";

    @Override
    public String resolveCurrentTenantIdentifier() {
        String tenantId = TenantContext.getCurrentTenant();
        if (tenantId == null || tenantId.isBlank()) {
            return DEFAULT_TENANT;
        }
        return tenantId;
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
```

### 6.3 JPA Configuration for Multi-Tenancy

```java
package com.example.multitenant.config;

import com.example.multitenant.hibernate.SchemaMultiTenantConnectionProvider;
import com.example.multitenant.hibernate.TenantIdentifierResolver;
import org.hibernate.cfg.AvailableSettings;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.orm.jpa.HibernatePropertiesCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;

@Configuration
public class MultiTenantJpaConfig {

    @Autowired
    private SchemaMultiTenantConnectionProvider connectionProvider;

    @Autowired
    private TenantIdentifierResolver tenantIdentifierResolver;

    @Bean
    public HibernatePropertiesCustomizer hibernateMultiTenancyCustomizer() {
        return hibernateProperties -> {
            hibernateProperties.put(
                AvailableSettings.MULTI_TENANT_CONNECTION_PROVIDER,
                connectionProvider
            );
            hibernateProperties.put(
                AvailableSettings.MULTI_TENANT_IDENTIFIER_RESOLVER,
                tenantIdentifierResolver
            );
            // Enable schema-per-tenant mode
            hibernateProperties.put(
                AvailableSettings.JAKARTA_HBM2DDL_DB_SCHEMA_FILTER_PROVIDER,
                "schema"
            );
        };
    }
}
```

---

## 7. DataSource Routing (Alternative Approach)

```java
package com.example.multitenant.datasource;

import com.example.multitenant.context.TenantContext;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class TenantRoutingDataSource extends AbstractRoutingDataSource {

    private static final Logger log = LoggerFactory.getLogger(TenantRoutingDataSource.class);

    // Cache for tenant-specific data sources
    private final Map<String, DataSource> tenantDataSources = new ConcurrentHashMap<>();

    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getCurrentTenant();
    }

    // Called when a lookup key is not found in the static map
    @Override
    protected DataSource determineTargetDataSource() {
        String tenantId = (String) determineCurrentLookupKey();
        if (tenantId == null) {
            return super.determineTargetDataSource(); // Fall back to default
        }

        return tenantDataSources.computeIfAbsent(tenantId, this::createTenantDataSource);
    }

    private DataSource createTenantDataSource(String tenantId) {
        log.info("Creating data source for tenant: {}", tenantId);
        // For database-per-tenant strategy
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/" + tenantId + "_db");
        config.setUsername("app_user");
        config.setPassword("secret");
        config.setMaximumPoolSize(5);
        config.setPoolName("pool-" + tenantId);
        return new HikariDataSource(config);
    }
}
```

---

## 8. Tenant Entity and Repository

```java
package com.example.multitenant.entity;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "tenants", schema = "public")  // Tenant registry lives in public schema
public class Tenant {

    @Id
    private String id;  // e.g., "acme", "globex"

    @Column(nullable = false)
    private String name;

    @Column(unique = true, nullable = false)
    private String subdomain;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TenantStatus status = TenantStatus.ACTIVE;

    @Column(nullable = false)
    private String plan = "free";

    private Instant createdAt = Instant.now();
    private Instant updatedAt = Instant.now();

    public enum TenantStatus {
        PENDING_SETUP, ACTIVE, SUSPENDED, DELETED
    }

    public Tenant() {}

    public Tenant(String id, String name, String subdomain, String email) {
        this.id = id;
        this.name = name;
        this.subdomain = subdomain;
        this.email = email;
    }

    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getSubdomain() { return subdomain; }
    public void setSubdomain(String subdomain) { this.subdomain = subdomain; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public TenantStatus getStatus() { return status; }
    public void setStatus(TenantStatus status) { this.status = status; }
    public String getPlan() { return plan; }
    public void setPlan(String plan) { this.plan = plan; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(Instant updatedAt) { this.updatedAt = updatedAt; }
}
```

```java
package com.example.multitenant.repository;

import com.example.multitenant.entity.Tenant;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface TenantRepository extends JpaRepository<Tenant, String> {

    Optional<Tenant> findBySubdomain(String subdomain);

    @Query("SELECT t FROM Tenant t WHERE t.id = :id AND t.status = 'ACTIVE'")
    Optional<Tenant> findActiveTenant(String id);

    boolean existsBySubdomain(String subdomain);
}
```

---

## 9. Flyway Multi-Tenant Migrations

### 9.1 Migration File Structure

```
src/main/resources/
  db/
    migration/
      shared/                          # Public schema (tenant registry)
        V1__create_tenants_table.sql
      tenant/                          # Per-tenant schema template
        V1__create_tenant_schema.sql
        V2__create_users_table.sql
        V3__create_products_table.sql
```

### 9.2 Shared Schema Migrations

```sql
-- db/migration/shared/V1__create_tenants_table.sql
CREATE TABLE IF NOT EXISTS public.tenants (
    id          VARCHAR(50) PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    subdomain   VARCHAR(100) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    plan        VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_tenants_subdomain ON public.tenants(subdomain);
CREATE INDEX IF NOT EXISTS idx_tenants_status ON public.tenants(status);
```

### 9.3 Per-Tenant Schema Migrations

```sql
-- db/migration/tenant/V1__create_tenant_schema.sql
-- This runs with search_path already set to tenant's schema

CREATE TABLE IF NOT EXISTS users (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(50) UNIQUE NOT NULL,
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(30) NOT NULL DEFAULT 'USER',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at   TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS audit_log (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),
    action      VARCHAR(100) NOT NULL,
    resource    VARCHAR(100),
    details     JSONB,
    ip_address  INET,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 9.4 Tenant Provisioning Service (runs Flyway per tenant)

```java
package com.example.multitenant.service;

import com.example.multitenant.entity.Tenant;
import com.example.multitenant.repository.TenantRepository;
import org.flywaydb.core.Flyway;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.Statement;

@Service
public class TenantProvisioningService {

    private static final Logger log = LoggerFactory.getLogger(TenantProvisioningService.class);

    @Autowired
    private TenantRepository tenantRepository;

    @Autowired
    private DataSource dataSource;

    @Transactional
    public Tenant provisionTenant(String tenantId, String name, String subdomain, String email) {
        log.info("Provisioning new tenant: {} ({})", name, tenantId);

        // 1. Validate tenant doesn't exist
        if (tenantRepository.existsById(tenantId)) {
            throw new IllegalArgumentException("Tenant already exists: " + tenantId);
        }

        // 2. Create schema in database
        createSchema(tenantId);

        // 3. Run migrations on new schema
        runMigrationsForTenant(tenantId);

        // 4. Save tenant record in public schema
        Tenant tenant = new Tenant(tenantId, name, subdomain, email);
        tenant.setStatus(Tenant.TenantStatus.ACTIVE);
        Tenant saved = tenantRepository.save(tenant);

        log.info("Tenant {} provisioned successfully", tenantId);
        return saved;
    }

    private void createSchema(String tenantId) {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {
            // Sanitize tenant ID to prevent SQL injection
            String sanitizedId = tenantId.replaceAll("[^a-zA-Z0-9_]", "");
            stmt.execute("CREATE SCHEMA IF NOT EXISTS " + sanitizedId);
            log.info("Schema created: {}", sanitizedId);
        } catch (Exception e) {
            throw new RuntimeException("Failed to create schema for tenant: " + tenantId, e);
        }
    }

    private void runMigrationsForTenant(String tenantId) {
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .schemas(tenantId)
            .defaultSchema(tenantId)
            .locations("classpath:db/migration/tenant")
            .table("flyway_schema_history")
            .baselineOnMigrate(true)
            .load();

        flyway.migrate();
        log.info("Migrations completed for tenant schema: {}", tenantId);
    }

    @Transactional
    public void suspendTenant(String tenantId) {
        tenantRepository.findById(tenantId).ifPresent(tenant -> {
            tenant.setStatus(Tenant.TenantStatus.SUSPENDED);
            tenant.setUpdatedAt(java.time.Instant.now());
            tenantRepository.save(tenant);
            log.info("Tenant {} suspended", tenantId);
        });
    }
}
```

### 9.5 Tenant Validation Service

```java
package com.example.multitenant.service;

import com.example.multitenant.entity.Tenant;
import com.example.multitenant.repository.TenantRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class TenantValidationService {

    @Autowired
    private TenantRepository tenantRepository;

    @Cacheable(value = "tenantValidation", key = "#tenantId")
    public boolean isValidTenant(String tenantId) {
        if (tenantId == null || tenantId.isBlank()) return false;
        // Only allow alphanumeric + underscore to prevent schema injection
        if (!tenantId.matches("^[a-zA-Z0-9_]+$")) return false;

        return tenantRepository.findActiveTenant(tenantId).isPresent();
    }
}
```

---

## 10. Caching Per Tenant

```java
package com.example.multitenant.cache;

import com.example.multitenant.context.TenantContext;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class TenantAwareCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        // Cache keys automatically include tenant context
        return new TenantAwareCacheManager();
    }
}
```

```java
package com.example.multitenant.cache;

import com.example.multitenant.context.TenantContext;
import org.springframework.cache.Cache;
import org.springframework.cache.concurrent.ConcurrentMapCache;
import org.springframework.cache.support.AbstractCacheManager;

import java.util.Collection;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class TenantAwareCacheManager extends AbstractCacheManager {

    // cacheName -> (tenantId -> cache)
    private final Map<String, Map<String, Cache>> caches = new ConcurrentHashMap<>();

    @Override
    protected Collection<? extends Cache> loadCaches() {
        return java.util.Collections.emptyList();
    }

    @Override
    protected Cache getMissingCache(String name) {
        String tenantId = TenantContext.getCurrentTenant();
        String cacheKey = tenantId != null ? name + ":" + tenantId : name;

        return caches
            .computeIfAbsent(name, k -> new ConcurrentHashMap<>())
            .computeIfAbsent(cacheKey, k -> new ConcurrentMapCache(cacheKey));
    }
}
```

```java
package com.example.multitenant.service;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class TenantAwareProductService {

    // Cache key automatically includes tenant context via TenantAwareCacheManager
    @Cacheable(value = "products", key = "#category")
    public List<String> getProductsByCategory(String category) {
        // This query automatically runs in the current tenant's schema
        // because Hibernate sets the schema connection property
        return List.of("Product A", "Product B");
    }
}
```

---

## 11. Spring Security Integration (Tenant-Aware UserDetailsService)

```java
package com.example.multitenant.security;

import com.example.multitenant.context.TenantContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.User;
import org.springframework.stereotype.Service;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.util.List;

@Service
public class TenantAwareUserDetailsService implements UserDetailsService {

    @Autowired
    private DataSource dataSource;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        String tenantId = TenantContext.getCurrentTenant();
        if (tenantId == null) {
            throw new UsernameNotFoundException("No tenant context - cannot load user");
        }

        // Query the tenant's schema for the user
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("SET search_path = " + tenantId + ", public");

            try (PreparedStatement ps = conn.prepareStatement(
                    "SELECT username, password_hash, role, is_active FROM users WHERE username = ?"
            )) {
                ps.setString(1, username);
                ResultSet rs = ps.executeQuery();

                if (!rs.next()) {
                    throw new UsernameNotFoundException(
                        "User not found: " + username + " in tenant: " + tenantId
                    );
                }

                if (!rs.getBoolean("is_active")) {
                    throw new UsernameNotFoundException("User is disabled: " + username);
                }

                return User.builder()
                    .username(rs.getString("username"))
                    .password(rs.getString("password_hash"))
                    .authorities(List.of(new SimpleGrantedAuthority("ROLE_" + rs.getString("role"))))
                    .build();
            }
        } catch (UsernameNotFoundException e) {
            throw e;
        } catch (Exception e) {
            throw new UsernameNotFoundException(
                "Failed to load user: " + username + " for tenant: " + tenantId, e
            );
        }
    }
}
```

```java
package com.example.multitenant.security;

import com.example.multitenant.filter.TenantFilter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class MultiTenantSecurityConfig {

    @Autowired
    private TenantFilter tenantFilter;

    @Autowired
    private JwtAuthenticationFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/tenants/register").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            // Tenant filter runs BEFORE JWT filter
            .addFilterBefore(tenantFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }
}
```

```java
package com.example.multitenant.security;

import com.example.multitenant.context.TenantContext;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtService jwtService;

    @Autowired
    private TenantAwareUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain
    ) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);
        String username = jwtService.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            if (jwtService.isTokenValid(token)) {
                // TenantContext already set by TenantFilter
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        chain.doFilter(request, response);
    }
}
```

---

## 12. JWT Service

```java
package com.example.multitenant.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.time.Instant;
import java.util.Base64;
import java.util.Date;
import java.util.Map;

@Service
public class JwtService {

    @Value("${app.jwt.secret}")
    private String jwtSecret;

    @Value("${app.jwt.expiration}")
    private long expirationSeconds;

    public String generateToken(String username, String tenantId, String role) {
        byte[] keyBytes = Base64.getDecoder().decode(jwtSecret);
        SecretKey key = Keys.hmacShaKeyFor(keyBytes);

        return Jwts.builder()
            .subject(username)
            .claims(Map.of(
                "tenantId", tenantId,
                "role", role
            ))
            .issuedAt(new Date())
            .expiration(Date.from(Instant.now().plusSeconds(expirationSeconds)))
            .signWith(key)
            .compact();
    }

    public String extractUsername(String token) {
        return parseClaims(token).getSubject();
    }

    public String extractTenantId(String token) {
        return parseClaims(token).get("tenantId", String.class);
    }

    public boolean isTokenValid(String token) {
        try {
            Claims claims = parseClaims(token);
            return !claims.getExpiration().before(new Date());
        } catch (Exception e) {
            return false;
        }
    }

    private Claims parseClaims(String token) {
        byte[] keyBytes = Base64.getDecoder().decode(jwtSecret);
        SecretKey key = Keys.hmacShaKeyFor(keyBytes);

        return Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

---

## 13. Tenant Registration Controller

```java
package com.example.multitenant.controller;

import com.example.multitenant.entity.Tenant;
import com.example.multitenant.service.TenantProvisioningService;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/tenants")
public class TenantController {

    @Autowired
    private TenantProvisioningService provisioningService;

    @PostMapping("/register")
    public ResponseEntity<?> registerTenant(@Valid @RequestBody TenantRegistrationRequest request) {
        try {
            Tenant tenant = provisioningService.provisionTenant(
                request.subdomain(),  // Use subdomain as tenant ID
                request.companyName(),
                request.subdomain(),
                request.email()
            );

            return ResponseEntity.ok(Map.of(
                "tenantId", tenant.getId(),
                "name", tenant.getName(),
                "subdomain", tenant.getSubdomain(),
                "status", tenant.getStatus()
            ));
        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().body(Map.of("error", e.getMessage()));
        }
    }

    @PostMapping("/{tenantId}/suspend")
    public ResponseEntity<?> suspendTenant(@PathVariable String tenantId) {
        provisioningService.suspendTenant(tenantId);
        return ResponseEntity.ok(Map.of("message", "Tenant suspended: " + tenantId));
    }

    record TenantRegistrationRequest(
        @NotBlank @Pattern(regexp = "^[a-z0-9-]+$",
            message = "Subdomain must be lowercase alphanumeric with hyphens")
        String subdomain,

        @NotBlank String companyName,

        @Email @NotBlank String email
    ) {}
}
```

---

## 14. Testing Multi-Tenant Applications

```java
package com.example.multitenant.test;

import com.example.multitenant.context.TenantContext;
import com.example.multitenant.service.TenantAwareProductService;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.jdbc.Sql;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringBootTest
class MultiTenantServiceTest {

    @Autowired
    private TenantAwareProductService productService;

    @BeforeEach
    void setUp() {
        // Set tenant context before each test
        TenantContext.setCurrentTenant("tenant1");
    }

    @AfterEach
    void tearDown() {
        // CRITICAL: Always clear after test
        TenantContext.clear();
    }

    @Test
    void shouldLoadProductsForTenant1() {
        assertThat(TenantContext.getCurrentTenant()).isEqualTo("tenant1");
        // Products would be loaded from tenant1 schema
        // Test your assertions here
    }

    @Test
    void shouldIsolateTenantData() {
        // Switch to tenant2 mid-test
        TenantContext.setCurrentTenant("tenant2");
        assertThat(TenantContext.getCurrentTenant()).isEqualTo("tenant2");
        // tenant2's data is separate from tenant1
    }

    @Test
    void shouldFailWithoutTenantContext() {
        TenantContext.clear();
        // Attempting to access data without tenant context should fail
        assertThat(TenantContext.hasTenant()).isFalse();
    }
}
```

```java
package com.example.multitenant.test;

import com.example.multitenant.context.TenantContext;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class TenantResolutionIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldResolveTenantFromHeader() throws Exception {
        mockMvc.perform(
            get("/api/products")
                .header("X-Tenant-ID", "tenant1")
                .header("Authorization", "Bearer valid-jwt-token")
                .contentType(MediaType.APPLICATION_JSON)
        )
        .andExpect(status().isOk());
    }

    @Test
    void shouldReturn400WithoutTenantHeader() throws Exception {
        mockMvc.perform(
            get("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
        )
        .andExpect(status().isBadRequest());
    }
}
```

---

## Summary

| Strategy | Schema | Isolation | Migration | Use Case |
|----------|--------|-----------|-----------|---------|
| Database per tenant | Separate DB | Maximum | Per-DB Flyway | Regulated, large tenants |
| Schema per tenant | Separate schema | High | Per-schema Flyway | SaaS with medium tenants |
| Row-level | Shared tables | Minimum | Single migration | Simple apps, micro-tenants |

**Implementation components:**

| Component | Role |
|-----------|------|
| `TenantContext` | Thread-local tenant ID storage |
| `TenantFilter` | Resolves and sets tenant per HTTP request |
| `CompositeTenantResolver` | Tries JWT → Header → Subdomain |
| `SchemaMultiTenantConnectionProvider` | Sets PostgreSQL schema per connection |
| `TenantIdentifierResolver` | Tells Hibernate current tenant |
| `TenantProvisioningService` | Creates schema + runs Flyway |
| `TenantAwareUserDetailsService` | Loads users from correct schema |
| `TenantAwareCacheManager` | Namespaces cache by tenant |

---

## Next Part Preview

**Part 046: Event Sourcing and CQRS** — We'll implement Command Query Responsibility Segregation (CQRS) and Event Sourcing with a Bank Account aggregate, custom EventStore, read model projections, and the Saga pattern for distributed transactions.
