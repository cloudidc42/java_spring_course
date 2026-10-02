# Part 050: API Security Best Practices

## Table of Contents
1. [OWASP API Security Top 10](#owasp-top-10)
2. [Input Validation and Sanitization](#input-validation)
3. [SQL Injection Prevention](#sql-injection)
4. [XSS Prevention](#xss-prevention)
5. [CSRF Protection](#csrf)
6. [Rate Limiting with Bucket4j](#rate-limiting)
7. [API Key Authentication](#api-key-auth)
8. [IP Allowlist/Blocklist](#ip-filtering)
9. [Request Signing (HMAC)](#request-signing)
10. [Sensitive Data Masking in Logs](#log-masking)
11. [Security Testing with OWASP ZAP](#zap-testing)
12. [Secure Configuration Management](#secure-config)
13. [Real Example: Hardened REST API](#real-example)
14. [Summary](#summary)

---

## 1. OWASP API Security Top 10 {#owasp-top-10}

| # | Vulnerability | Spring Boot Mitigation |
|---|--------------|----------------------|
| API1 | Broken Object Level Authorization | Check ownership on every resource access |
| API2 | Broken Authentication | JWT + Spring Security; rotate secrets |
| API3 | Broken Object Property Level Auth | Use DTOs; never expose full entities |
| API4 | Unrestricted Resource Consumption | Rate limiting; max payload size |
| API5 | Broken Function Level Authorization | Method security `@PreAuthorize` |
| API6 | Unrestricted Access to Sensitive Business Flows | Business rule validation + rate limit |
| API7 | Server Side Request Forgery (SSRF) | Validate all external URLs; allowlist |
| API8 | Security Misconfiguration | Disable debug; harden headers; audit |
| API9 | Improper Inventory Management | API versioning; deprecation headers |
| API10 | Unsafe Consumption of APIs | Validate all third-party responses |

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Rate limiting -->
    <dependency>
        <groupId>com.github.vladimir-bukhtoyarov</groupId>
        <artifactId>bucket4j-core</artifactId>
        <version>8.10.1</version>
    </dependency>
    <dependency>
        <groupId>com.github.vladimir-bukhtoyarov</groupId>
        <artifactId>bucket4j-redis</artifactId>
        <version>8.10.1</version>
    </dependency>

    <!-- HTML sanitization -->
    <dependency>
        <groupId>org.owasp.antisamy</groupId>
        <artifactId>antisamy</artifactId>
        <version>1.7.5</version>
    </dependency>

    <!-- JWT -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <!-- API key storage -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-crypto</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 2. Input Validation and Sanitization {#input-validation}

### Custom Validation Annotations

```java
package com.example.security.validation;

import jakarta.validation.Constraint;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import jakarta.validation.Payload;

import java.lang.annotation.*;
import java.util.regex.Pattern;

/**
 * Validates that a string contains no SQL injection patterns
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NoSqlInjection.Validator.class)
@Documented
public @interface NoSqlInjection {
    String message() default "Input contains invalid characters";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};

    class Validator implements ConstraintValidator<NoSqlInjection, String> {
        // Detect common SQL injection patterns
        private static final Pattern SQL_PATTERN = Pattern.compile(
            "(?i).*('\\s*(OR|AND)\\s+')|" +
            "(--\\s)|" +
            "(/\\*.*\\*/)|" +
            "(;\\s*(DROP|DELETE|INSERT|UPDATE|CREATE|ALTER|EXEC))|" +
            "(?i)(UNION\\s+SELECT)|" +
            "(xp_\\w+)|" +
            "(0x[0-9a-fA-F]+)",
            Pattern.DOTALL
        );

        @Override
        public boolean isValid(String value, ConstraintValidatorContext context) {
            if (value == null) return true;
            return !SQL_PATTERN.matcher(value).matches();
        }
    }
}
```

```java
package com.example.security.validation;

import jakarta.validation.Constraint;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import jakarta.validation.Payload;

import java.lang.annotation.*;
import java.net.URI;
import java.util.Set;

/**
 * Validates a URL is from an allowed domain (SSRF prevention)
 */
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = SafeUrl.Validator.class)
@Documented
public @interface SafeUrl {
    String message() default "URL domain is not allowed";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    String[] allowedDomains() default {};

    class Validator implements ConstraintValidator<SafeUrl, String> {
        private Set<String> allowedDomains;

        @Override
        public void initialize(SafeUrl annotation) {
            this.allowedDomains = Set.of(annotation.allowedDomains());
        }

        @Override
        public boolean isValid(String value, ConstraintValidatorContext context) {
            if (value == null || value.isBlank()) return true;

            try {
                URI uri = new URI(value);
                String host = uri.getHost();

                // Reject internal IPs (SSRF prevention)
                if (host == null) return false;
                if (host.equals("localhost") || host.equals("127.0.0.1")) return false;
                if (host.startsWith("10.") || host.startsWith("192.168.")
                    || host.startsWith("172.16.")) return false;

                // Check allowed domains if specified
                if (!allowedDomains.isEmpty()) {
                    return allowedDomains.stream()
                            .anyMatch(domain -> host.equals(domain)
                                    || host.endsWith("." + domain));
                }

                return true;
            } catch (Exception e) {
                return false;
            }
        }
    }
}
```

### Request DTO with Validation

```java
package com.example.security.dto;

import com.example.security.validation.NoSqlInjection;
import com.example.security.validation.SafeUrl;
import jakarta.validation.constraints.*;
import lombok.Data;
import org.hibernate.validator.constraints.Length;

@Data
public class CreateProductRequest {

    @NotBlank(message = "Name is required")
    @Length(min = 2, max = 100, message = "Name must be 2-100 characters")
    @Pattern(regexp = "^[a-zA-Z0-9\\s\\-_.,]+$",
             message = "Name contains invalid characters")
    @NoSqlInjection
    private String name;

    @NotBlank(message = "Description is required")
    @Length(max = 2000, message = "Description too long")
    @NoSqlInjection
    private String description;

    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be positive")
    @DecimalMax(value = "999999.99", message = "Price is too high")
    @Digits(integer = 6, fraction = 2, message = "Invalid price format")
    private java.math.BigDecimal price;

    @Min(value = 0, message = "Quantity cannot be negative")
    @Max(value = 100000, message = "Quantity is too high")
    private int quantity;

    @Email(message = "Invalid email format")
    @Length(max = 255)
    private String contactEmail;

    // Only specific domains allowed (SSRF prevention)
    @SafeUrl(allowedDomains = {"cdn.example.com", "images.example.com"})
    private String imageUrl;
}
```

### HTML Sanitizer Service

```java
package com.example.security.service;

import org.owasp.validator.html.*;
import org.springframework.stereotype.Service;

import java.io.InputStream;

@Service
public class HtmlSanitizerService {

    private final Policy policy;

    public HtmlSanitizerService() {
        try {
            // AntiSamy policy file - restricts allowed HTML tags/attributes
            InputStream policyStream = getClass().getResourceAsStream("/antisamy-policy.xml");
            this.policy = Policy.getInstance(policyStream);
        } catch (PolicyException e) {
            throw new RuntimeException("Failed to load AntiSamy policy", e);
        }
    }

    /**
     * Sanitize HTML input to prevent XSS
     * Allows basic formatting tags only
     */
    public String sanitize(String html) {
        if (html == null || html.isBlank()) return html;

        try {
            AntiSamy antiSamy = new AntiSamy();
            CleanResults results = antiSamy.scan(html, policy);

            if (!results.getErrorMessages().isEmpty()) {
                // Log what was stripped for monitoring
                results.getErrorMessages().forEach(msg ->
                    org.slf4j.LoggerFactory.getLogger(getClass())
                            .debug("AntiSamy stripped: {}", msg));
            }

            return results.getCleanHTML();
        } catch (Exception e) {
            // On error, strip all HTML to be safe
            return html.replaceAll("<[^>]*>", "");
        }
    }

    /**
     * Strip all HTML tags (for plain text fields)
     */
    public String stripHtml(String input) {
        if (input == null) return null;
        return input.replaceAll("<[^>]*>", "").trim();
    }

    /**
     * Escape HTML entities
     */
    public String escapeHtml(String input) {
        if (input == null) return null;
        return input
                .replace("&", "&amp;")
                .replace("<", "&lt;")
                .replace(">", "&gt;")
                .replace("\"", "&quot;")
                .replace("'", "&#x27;");
    }
}
```

---

## 3. SQL Injection Prevention {#sql-injection}

### Using JPA Named Parameters (Safe)

```java
package com.example.security.repository;

import com.example.security.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // ✅ Safe: Spring Data generates parameterized query
    Optional<Product> findByNameAndActiveTrue(String name);

    // ✅ Safe: Named parameter with @Param
    @Query("SELECT p FROM Product p WHERE p.category = :category AND p.price <= :maxPrice")
    List<Product> findByCategoryAndMaxPrice(@Param("category") String category,
                                             @Param("maxPrice") BigDecimal maxPrice);

    // ✅ Safe: Native query with named parameters
    @Query(value = "SELECT * FROM products WHERE name ILIKE :pattern AND active = true",
           nativeQuery = true)
    List<Product> searchByNamePattern(@Param("pattern") String pattern);

    // ✅ Safe: Criteria API (programmatic, no string concatenation)
    // See ProductRepositoryCustom for Criteria API examples

    // ❌ NEVER do this:
    // @Query("SELECT p FROM Product p WHERE p.name = '" + name + "'")
    // This is vulnerable to SQL injection
}
```

### Safe Criteria API for Dynamic Queries

```java
package com.example.security.repository;

import com.example.security.dto.ProductSearchRequest;
import com.example.security.entity.Product;
import jakarta.persistence.EntityManager;
import jakarta.persistence.criteria.*;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Repository;
import org.springframework.util.StringUtils;

import java.util.ArrayList;
import java.util.List;

@Repository
@RequiredArgsConstructor
public class ProductRepositoryCustomImpl implements ProductRepositoryCustom {

    private final EntityManager entityManager;

    /**
     * Dynamic search with Criteria API - completely safe from SQL injection
     */
    @Override
    public Page<Product> searchProducts(ProductSearchRequest request, Pageable pageable) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Product> query = cb.createQuery(Product.class);
        Root<Product> root = query.from(Product.class);

        List<Predicate> predicates = buildPredicates(cb, root, request);

        query.where(predicates.toArray(new Predicate[0]));

        // Apply sorting
        if (pageable.getSort().isSorted()) {
            List<Order> orders = new ArrayList<>();
            pageable.getSort().forEach(sort -> {
                if (sort.isAscending()) {
                    orders.add(cb.asc(root.get(sort.getProperty())));
                } else {
                    orders.add(cb.desc(root.get(sort.getProperty())));
                }
            });
            query.orderBy(orders);
        }

        // Execute with pagination
        List<Product> results = entityManager.createQuery(query)
                .setFirstResult((int) pageable.getOffset())
                .setMaxResults(pageable.getPageSize())
                .getResultList();

        // Count query for pagination
        CriteriaQuery<Long> countQuery = cb.createQuery(Long.class);
        Root<Product> countRoot = countQuery.from(Product.class);
        countQuery.select(cb.count(countRoot));
        List<Predicate> countPredicates = buildPredicates(cb, countRoot, request);
        countQuery.where(countPredicates.toArray(new Predicate[0]));
        Long total = entityManager.createQuery(countQuery).getSingleResult();

        return new PageImpl<>(results, pageable, total);
    }

    private List<Predicate> buildPredicates(CriteriaBuilder cb, Root<Product> root,
                                             ProductSearchRequest request) {
        List<Predicate> predicates = new ArrayList<>();

        // All parameters are bound as parameters, never concatenated
        predicates.add(cb.isTrue(root.get("active")));

        if (StringUtils.hasText(request.getName())) {
            // LIKE with % is safe because the value is a parameter
            predicates.add(cb.like(cb.lower(root.get("name")),
                    "%" + request.getName().toLowerCase() + "%"));
        }

        if (request.getCategory() != null) {
            predicates.add(cb.equal(root.get("category"), request.getCategory()));
        }

        if (request.getMinPrice() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("price"), request.getMinPrice()));
        }

        if (request.getMaxPrice() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("price"), request.getMaxPrice()));
        }

        return predicates;
    }
}
```

---

## 4. XSS Prevention {#xss-prevention}

### Security Headers Configuration

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter;
import org.springframework.web.filter.OncePerRequestFilter;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Security headers
            .headers(headers -> headers
                // X-Content-Type-Options: nosniff
                .contentTypeOptions(contentType -> {})
                // X-Frame-Options: DENY
                .frameOptions(frame -> frame.deny())
                // X-XSS-Protection (legacy, modern browsers use CSP)
                .xssProtection(xss -> xss.enable())
                // Content Security Policy - prevents XSS
                .contentSecurityPolicy(csp -> csp.policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self'; " +
                    "style-src 'self' 'unsafe-inline' fonts.googleapis.com; " +
                    "font-src 'self' fonts.gstatic.com; " +
                    "img-src 'self' data: https:; " +
                    "connect-src 'self' https://api.myapp.com; " +
                    "frame-ancestors 'none'; " +
                    "form-action 'self'; " +
                    "base-uri 'self'"
                ))
                // HSTS - force HTTPS
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true)
                )
                // Referrer Policy
                .referrerPolicy(referrer -> referrer
                    .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                )
                // Permissions Policy
                .permissionsPolicy(permissions -> permissions
                    .policy("camera=(), microphone=(), geolocation=(), payment=()")
                )
            )
            // Disable caching for API responses
            .sessionManagement(session -> session
                .sessionCreationPolicy(
                    org.springframework.security.config.http.SessionCreationPolicy.STATELESS)
            );

        return http.build();
    }
}
```

### Response Filter to Strip Sensitive Headers

```java
package com.example.security.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class SecurityResponseFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // Remove headers that expose server info
        httpResponse.setHeader("X-Powered-By", null);
        httpResponse.setHeader("Server", null);

        // Add security headers not handled by Spring Security
        httpResponse.setHeader("X-Content-Type-Options", "nosniff");
        httpResponse.setHeader("Cache-Control",
                "no-cache, no-store, must-revalidate, private");
        httpResponse.setHeader("Pragma", "no-cache");
        httpResponse.setHeader("Expires", "0");

        chain.doFilter(request, response);
    }
}
```

---

## 5. CSRF Protection {#csrf}

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.csrf.*;

@Configuration
public class CsrfConfig {

    /**
     * For REST APIs with stateless JWT:
     * - CSRF is not needed if you NEVER use cookies for auth tokens
     * - But needed if you use session-based auth or cookie tokens
     *
     * For cookie-based auth (e.g., session or httpOnly JWT cookie):
     */
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        // Option 1: Stateless REST API - disable CSRF (tokens in Authorization header)
        http.csrf(csrf -> csrf.disable());

        // Option 2: Cookie-based auth - use Double Submit Cookie pattern
        /*
        http.csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
        );
        */

        return http.build();
    }
}
```

---

## 6. Rate Limiting with Bucket4j {#rate-limiting}

### Rate Limit Filter

```java
package com.example.security.ratelimit;

import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.Cache;
import io.github.bucket4j.*;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Slf4j
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    // In-memory cache for buckets per IP/user
    // Production: use Redis-backed Bucket4j ProxyManager
    private final Cache<String, Bucket> bucketCache = Caffeine.newBuilder()
            .maximumSize(100_000)
            .expireAfterWrite(1, TimeUnit.HOURS)
            .build();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String key = resolveKey(request);
        Bucket bucket = bucketCache.get(key, k -> createBucket(request));

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        response.addHeader("X-Rate-Limit-Remaining",
                String.valueOf(probe.getRemainingTokens()));
        response.addHeader("X-Rate-Limit-Limit",
                String.valueOf(getRateLimit(request)));

        if (probe.isConsumed()) {
            filterChain.doFilter(request, response);
        } else {
            long waitSeconds = probe.getNanosToWaitForRefill() / 1_000_000_000;
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.addHeader("Retry-After", String.valueOf(waitSeconds));
            response.getWriter().write(String.format(
                "{\"error\":\"RATE_LIMIT_EXCEEDED\",\"retryAfter\":%d}", waitSeconds));

            log.warn("Rate limit exceeded for key: {}, path: {}",
                    key, request.getRequestURI());
        }
    }

    private String resolveKey(HttpServletRequest request) {
        // 1. Authenticated user ID
        var auth = org.springframework.security.core.context.SecurityContextHolder
                .getContext().getAuthentication();
        if (auth != null && auth.isAuthenticated() && !"anonymousUser".equals(auth.getPrincipal())) {
            return "user:" + auth.getName();
        }

        // 2. API key header
        String apiKey = request.getHeader("X-Api-Key");
        if (apiKey != null && !apiKey.isBlank()) {
            return "apikey:" + apiKey.substring(0, Math.min(apiKey.length(), 8));
        }

        // 3. Fall back to IP address
        return "ip:" + getClientIp(request);
    }

    private Bucket createBucket(HttpServletRequest request) {
        // Different limits for different endpoints
        String path = request.getRequestURI();

        if (path.startsWith("/api/auth/")) {
            // Strict limit on auth endpoints
            return Bucket.builder()
                    .addLimit(Bandwidth.classic(10, Refill.intervally(10, Duration.ofMinutes(1))))
                    .build();
        }

        if (path.startsWith("/api/admin/")) {
            // Admin endpoints - generous limit for authenticated admins
            return Bucket.builder()
                    .addLimit(Bandwidth.classic(1000, Refill.intervally(1000, Duration.ofMinutes(1))))
                    .build();
        }

        // Default API rate limit
        return Bucket.builder()
                .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
                .build();
    }

    private long getRateLimit(HttpServletRequest request) {
        String path = request.getRequestURI();
        if (path.startsWith("/api/auth/")) return 10;
        if (path.startsWith("/api/admin/")) return 1000;
        return 100;
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isBlank()) {
            return xForwardedFor.split(",")[0].trim();
        }
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isBlank()) {
            return xRealIp;
        }
        return request.getRemoteAddr();
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getRequestURI();
        // Skip rate limiting for actuator health check
        return path.startsWith("/actuator/health");
    }
}
```

### Redis-backed Rate Limiting (Production)

```java
package com.example.security.ratelimit;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.BucketConfiguration;
import io.github.bucket4j.Refill;
import io.github.bucket4j.distributed.proxy.ProxyManager;
import io.github.bucket4j.redis.lettuce.cas.LettuceBasedProxyManager;
import io.lettuce.core.RedisClient;
import io.lettuce.core.api.StatefulRedisConnection;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;

import java.time.Duration;

@Configuration
public class RateLimitConfig {

    @Bean
    public ProxyManager<String> rateLimitProxyManager(LettuceConnectionFactory connectionFactory) {
        // Use Redis for distributed rate limiting (multiple pod instances)
        StatefulRedisConnection<String, byte[]> connection =
                ((io.lettuce.core.api.StatefulRedisConnection<String, byte[]>)
                        connectionFactory.getNativeClient());

        return LettuceBasedProxyManager.builderFor(connection).build();
    }

    @Bean
    public BucketConfiguration defaultRateLimitConfig() {
        return BucketConfiguration.builder()
                .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
                .addLimit(Bandwidth.classic(1000, Refill.greedy(1000, Duration.ofHours(1))))
                .build();
    }
}
```

---

## 7. API Key Authentication {#api-key-auth}

### API Key Entity and Repository

```java
package com.example.security.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

@Entity
@Table(name = "api_keys",
       indexes = {
           @Index(name = "idx_api_key_hash", columnList = "key_hash", unique = true)
       })
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiKey {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(name = "key_hash", nullable = false, unique = true, length = 64)
    private String keyHash;  // SHA-256 hash of actual key - NEVER store plain key

    @Column(name = "key_prefix", nullable = false, length = 8)
    private String keyPrefix;  // First 8 chars - for display/logging only

    @Column(name = "name", nullable = false)
    private String name;  // Human-readable name

    @Column(name = "owner_id", nullable = false)
    private String ownerId;

    @ElementCollection
    @CollectionTable(name = "api_key_permissions",
                     joinColumns = @JoinColumn(name = "api_key_id"))
    @Column(name = "permission")
    private java.util.Set<String> permissions;

    @Column(name = "expires_at")
    private Instant expiresAt;

    @Column(name = "last_used_at")
    private Instant lastUsedAt;

    @Column(name = "active")
    @Builder.Default
    private boolean active = true;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;
}
```

### API Key Service

```java
package com.example.security.service;

import com.example.security.entity.ApiKey;
import com.example.security.repository.ApiKeyRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.crypto.codec.Hex;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.time.Instant;
import java.util.Base64;
import java.util.Optional;
import java.util.Set;

@Slf4j
@Service
@RequiredArgsConstructor
public class ApiKeyService {

    private final ApiKeyRepository apiKeyRepository;
    private final SecureRandom secureRandom = new SecureRandom();

    /**
     * Generate a new API key
     * Returns plain key ONCE - never stored in plain text
     */
    @Transactional
    public ApiKeyGenerationResult generateApiKey(String name, String ownerId,
                                                  Set<String> permissions,
                                                  Instant expiresAt) {
        // Generate cryptographically secure random key
        byte[] keyBytes = new byte[32];
        secureRandom.nextBytes(keyBytes);
        String plainKey = "mk_" + Base64.getUrlEncoder().withoutPadding()
                .encodeToString(keyBytes);

        String keyHash = hashKey(plainKey);
        String keyPrefix = plainKey.substring(0, 8);

        ApiKey apiKey = ApiKey.builder()
                .keyHash(keyHash)
                .keyPrefix(keyPrefix)
                .name(name)
                .ownerId(ownerId)
                .permissions(permissions)
                .expiresAt(expiresAt)
                .build();

        ApiKey saved = apiKeyRepository.save(apiKey);
        log.info("Generated API key: id={}, name={}, owner={}", saved.getId(), name, ownerId);

        return new ApiKeyGenerationResult(saved.getId(), plainKey, keyPrefix);
    }

    /**
     * Validate and retrieve an API key
     * This is called on every authenticated request
     */
    @Transactional
    public Optional<ApiKey> validateKey(String plainKey) {
        if (plainKey == null || !plainKey.startsWith("mk_")) {
            return Optional.empty();
        }

        String keyHash = hashKey(plainKey);
        Optional<ApiKey> apiKey = apiKeyRepository.findByKeyHash(keyHash);

        if (apiKey.isEmpty()) {
            log.warn("API key not found for prefix: {}",
                    plainKey.substring(0, Math.min(plainKey.length(), 8)));
            return Optional.empty();
        }

        ApiKey key = apiKey.get();

        if (!key.isActive()) {
            log.warn("Inactive API key used: id={}", key.getId());
            return Optional.empty();
        }

        if (key.getExpiresAt() != null && key.getExpiresAt().isBefore(Instant.now())) {
            log.warn("Expired API key used: id={}", key.getId());
            return Optional.empty();
        }

        // Update last used time (non-blocking update)
        apiKeyRepository.updateLastUsed(key.getId(), Instant.now());

        return Optional.of(key);
    }

    /**
     * Revoke an API key
     */
    @Transactional
    public void revokeKey(String keyId, String ownerId) {
        ApiKey key = apiKeyRepository.findByIdAndOwnerId(keyId, ownerId)
                .orElseThrow(() -> new RuntimeException("API key not found"));
        key.setActive(false);
        apiKeyRepository.save(key);
        log.info("Revoked API key: id={}", keyId);
    }

    private String hashKey(String plainKey) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(plainKey.getBytes());
            return new String(Hex.encode(hash));
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("SHA-256 not available", e);
        }
    }

    public record ApiKeyGenerationResult(String id, String plainKey, String prefix) {}
}
```

### API Key Authentication Filter

```java
package com.example.security.filter;

import com.example.security.entity.ApiKey;
import com.example.security.service.ApiKeyService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.stream.Collectors;

@Slf4j
@Component
@RequiredArgsConstructor
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    private final ApiKeyService apiKeyService;

    private static final String API_KEY_HEADER = "X-Api-Key";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        String apiKeyValue = request.getHeader(API_KEY_HEADER);

        if (apiKeyValue != null && !apiKeyValue.isBlank()
                && SecurityContextHolder.getContext().getAuthentication() == null) {

            apiKeyService.validateKey(apiKeyValue).ifPresentOrElse(
                    apiKey -> authenticateWithApiKey(apiKey),
                    () -> log.debug("Invalid API key attempt from IP: {}",
                            request.getRemoteAddr())
            );
        }

        filterChain.doFilter(request, response);
    }

    private void authenticateWithApiKey(ApiKey apiKey) {
        var authorities = apiKey.getPermissions().stream()
                .map(perm -> new SimpleGrantedAuthority("SCOPE_" + perm))
                .collect(Collectors.toList());

        var authentication = new UsernamePasswordAuthenticationToken(
                apiKey.getOwnerId(),
                null,
                authorities
        );

        SecurityContextHolder.getContext().setAuthentication(authentication);
        log.debug("API key authenticated: keyId={}, owner={}",
                apiKey.getId(), apiKey.getOwnerId());
    }
}
```

---

## 8. IP Allowlist/Blocklist {#ip-filtering}

```java
package com.example.security.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.net.InetAddress;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Component
public class IpFilterFilter implements Filter {

    @Value("${security.ip.blocklist:}")
    private Set<String> blocklist;

    @Value("${security.ip.allowlist:}")
    private Set<String> allowlist;

    @Value("${security.ip.admin-allowlist:127.0.0.1}")
    private Set<String> adminAllowlist;

    // Dynamic blocklist (updated at runtime without restart)
    private final Set<String> dynamicBlocklist = ConcurrentHashMap.newKeySet();

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String clientIp = getClientIp(httpRequest);

        // Check admin paths
        if (httpRequest.getRequestURI().startsWith("/api/admin/")) {
            if (!isAllowed(clientIp, adminAllowlist)) {
                log.warn("Blocked admin access from IP: {}", clientIp);
                httpResponse.setStatus(HttpStatus.FORBIDDEN.value());
                return;
            }
        }

        // Check blocklist
        if (isBlocked(clientIp)) {
            log.warn("Blocked request from blocklisted IP: {}", clientIp);
            httpResponse.setStatus(HttpStatus.FORBIDDEN.value());
            httpResponse.getWriter().write("{\"error\":\"ACCESS_DENIED\"}");
            return;
        }

        chain.doFilter(request, response);
    }

    public void blockIp(String ip) {
        dynamicBlocklist.add(ip);
        log.info("Dynamically blocked IP: {}", ip);
    }

    public void unblockIp(String ip) {
        dynamicBlocklist.remove(ip);
        log.info("Unblocked IP: {}", ip);
    }

    private boolean isBlocked(String ip) {
        return blocklist.contains(ip) || dynamicBlocklist.contains(ip);
    }

    private boolean isAllowed(String ip, Set<String> allowlist) {
        if (allowlist.isEmpty()) return true;
        return allowlist.contains(ip) || allowlist.contains("0.0.0.0/0");
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff != null) return xff.split(",")[0].trim();
        String xri = request.getHeader("X-Real-IP");
        if (xri != null) return xri.trim();
        return request.getRemoteAddr();
    }
}
```

---

## 9. Request Signing (HMAC) {#request-signing}

```java
package com.example.security.signing;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.util.ContentCachingRequestWrapper;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Base64;

@Slf4j
@Component
@RequiredArgsConstructor
public class HmacSignatureFilter extends OncePerRequestFilter {

    @Value("${security.hmac.secret}")
    private String hmacSecret;

    @Value("${security.hmac.max-timestamp-drift-seconds:300}")
    private long maxTimestampDrift;

    private static final String SIGNATURE_HEADER = "X-Signature";
    private static final String TIMESTAMP_HEADER = "X-Timestamp";
    private static final String NONCE_HEADER = "X-Nonce";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        // Only validate signed endpoints (e.g., webhooks)
        if (!request.getRequestURI().startsWith("/api/webhooks/")) {
            filterChain.doFilter(request, response);
            return;
        }

        ContentCachingRequestWrapper wrappedRequest =
                new ContentCachingRequestWrapper(request);

        try {
            validateSignature(wrappedRequest);
            filterChain.doFilter(wrappedRequest, response);
        } catch (SignatureException e) {
            log.warn("HMAC signature validation failed: {}", e.getMessage());
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("{\"error\":\"INVALID_SIGNATURE\"}");
        }
    }

    private void validateSignature(ContentCachingRequestWrapper request)
            throws SignatureException {
        String signature = request.getHeader(SIGNATURE_HEADER);
        String timestamp = request.getHeader(TIMESTAMP_HEADER);
        String nonce = request.getHeader(NONCE_HEADER);

        if (signature == null || timestamp == null || nonce == null) {
            throw new SignatureException("Missing required signature headers");
        }

        // Validate timestamp (replay attack prevention)
        long requestTime;
        try {
            requestTime = Long.parseLong(timestamp);
        } catch (NumberFormatException e) {
            throw new SignatureException("Invalid timestamp format");
        }

        long now = Instant.now().getEpochSecond();
        if (Math.abs(now - requestTime) > maxTimestampDrift) {
            throw new SignatureException("Request timestamp too old or in the future");
        }

        // Read body
        byte[] body;
        try {
            // Force body to be read and cached
            request.getInputStream().readAllBytes();
            body = request.getContentAsByteArray();
        } catch (IOException e) {
            throw new SignatureException("Failed to read request body");
        }

        // Build signed string: method + path + timestamp + nonce + body_hash
        String bodyHash = hashBody(body);
        String signedString = request.getMethod() + "\n" +
                request.getRequestURI() + "\n" +
                timestamp + "\n" +
                nonce + "\n" +
                bodyHash;

        String expectedSignature = computeHmac(signedString);

        // Constant-time comparison to prevent timing attacks
        if (!constantTimeEquals(expectedSignature, signature)) {
            throw new SignatureException("Signature mismatch");
        }
    }

    private String computeHmac(String data) throws SignatureException {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            SecretKeySpec keySpec = new SecretKeySpec(
                    hmacSecret.getBytes(StandardCharsets.UTF_8), "HmacSHA256");
            mac.init(keySpec);
            byte[] hmac = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hmac);
        } catch (Exception e) {
            throw new SignatureException("Failed to compute HMAC", e);
        }
    }

    private String hashBody(byte[] body) {
        try {
            java.security.MessageDigest digest =
                    java.security.MessageDigest.getInstance("SHA-256");
            return Base64.getEncoder().encodeToString(digest.digest(body));
        } catch (Exception e) {
            throw new RuntimeException("SHA-256 not available", e);
        }
    }

    private boolean constantTimeEquals(String a, String b) {
        if (a.length() != b.length()) return false;
        int result = 0;
        for (int i = 0; i < a.length(); i++) {
            result |= a.charAt(i) ^ b.charAt(i);
        }
        return result == 0;
    }

    static class SignatureException extends RuntimeException {
        SignatureException(String message) { super(message); }
        SignatureException(String message, Throwable cause) { super(message, cause); }
    }
}
```

---

## 10. Sensitive Data Masking in Logs {#log-masking}

### Log Masking Pattern Layout (Logback)

```xml
<!-- src/main/resources/logback-spring.xml -->
<configuration>
    <springProfile name="!local">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>

        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

### Masking MDC Filter

```java
package com.example.security.logging;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
public class LoggingMdcFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // Set trace ID for correlation
        String traceId = httpRequest.getHeader("X-Trace-Id");
        if (traceId == null) traceId = UUID.randomUUID().toString().substring(0, 8);

        MDC.put("traceId", traceId);
        MDC.put("method", httpRequest.getMethod());
        MDC.put("path", httpRequest.getRequestURI());

        // Mask IP for GDPR compliance
        String ip = httpRequest.getRemoteAddr();
        MDC.put("clientIp", maskIp(ip));

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }

    private String maskIp(String ip) {
        if (ip == null) return "unknown";
        // Show only first two octets: 192.168.x.x → 192.168.*.*
        String[] parts = ip.split("\\.");
        if (parts.length == 4) {
            return parts[0] + "." + parts[1] + ".*.*";
        }
        return "***";
    }
}
```

### Custom Log Converter for Sensitive Data

```java
package com.example.security.logging;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.regex.Pattern;

@Slf4j
@Aspect
@Component
public class SensitiveDataMaskingAspect {

    // Patterns for sensitive data
    private static final Pattern CREDIT_CARD = Pattern.compile(
            "\\b(?:\\d[ -]*?){13,16}\\b");
    private static final Pattern SSN = Pattern.compile(
            "\\b\\d{3}-\\d{2}-\\d{4}\\b");
    private static final Pattern EMAIL = Pattern.compile(
            "[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}");

    /**
     * Mask sensitive fields before logging in service layer
     */
    public static String maskSensitiveData(String text) {
        if (text == null) return null;

        text = CREDIT_CARD.matcher(text).replaceAll(match -> {
            String card = match.group().replaceAll("[ -]", "");
            return "****-****-****-" + card.substring(card.length() - 4);
        });

        text = SSN.matcher(text).replaceAll("***-**-****");

        text = EMAIL.matcher(text).replaceAll(match -> {
            String email = match.group();
            int atIndex = email.indexOf('@');
            String local = email.substring(0, atIndex);
            String domain = email.substring(atIndex);
            if (local.length() <= 2) return "**" + domain;
            return local.charAt(0) + "***" + local.charAt(local.length() - 1) + domain;
        });

        return text;
    }
}
```

---

## 11. Security Testing with OWASP ZAP {#zap-testing}

### ZAP Integration Test

```java
package com.example.security.test;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.ActiveProfiles;
import org.zaproxy.clientapi.core.*;
import org.zaproxy.clientapi.gen.*;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * OWASP ZAP automated security scanning test
 * Requires ZAP running locally or in Docker
 * docker run -p 8090:8090 owasp/zap2docker-stable zap.sh -daemon -port 8090
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OWASPZapSecurityTest {

    @LocalServerPort
    private int port;

    private static ClientApi zapApi;

    @BeforeAll
    static void startZap() throws Exception {
        zapApi = new ClientApi("localhost", 8090, "zap-api-key");
    }

    @Test
    void runActiveSecurityScan() throws Exception {
        String targetUrl = "http://localhost:" + port;

        // Spider the application
        ApiResponse spiderResponse = zapApi.spider.scan(targetUrl, "10", "true", "", "");
        String scanId = ((ApiResponseElement) spiderResponse).getValue();

        // Wait for spider to complete
        waitForScanCompletion(zapApi.spider, scanId, 60);

        // Run active scan (tries common attacks)
        ApiResponse activeScanResponse = zapApi.ascan.scan(
                targetUrl, "true", "false", "", null, null);
        String activeScanId = ((ApiResponseElement) activeScanResponse).getValue();

        waitForScanCompletion(zapApi.ascan, activeScanId, 120);

        // Get alerts
        ApiResponse alertsResponse = zapApi.core.alerts(targetUrl, null, null, null);
        List<ApiResponse> alerts = ((ApiResponseList) alertsResponse).getItems();

        // Filter high/critical alerts
        long highRiskAlerts = alerts.stream()
                .filter(alert -> {
                    try {
                        String risk = ((ApiResponseSet) alert).getValue("risk");
                        return "High".equals(risk) || "Critical".equals(risk);
                    } catch (Exception e) {
                        return false;
                    }
                })
                .count();

        assertThat(highRiskAlerts)
                .as("High/Critical security vulnerabilities found by OWASP ZAP")
                .isEqualTo(0);
    }

    private void waitForScanCompletion(Object scanApi, String scanId, int timeoutSeconds)
            throws Exception {
        long startTime = System.currentTimeMillis();
        while (true) {
            String progress;
            if (scanApi instanceof Spider) {
                progress = ((ApiResponseElement) ((Spider) scanApi).status(scanId)).getValue();
            } else {
                progress = ((ApiResponseElement) ((Ascan) scanApi).status(scanId)).getValue();
            }

            if ("100".equals(progress)) break;

            if (System.currentTimeMillis() - startTime > timeoutSeconds * 1000L) {
                throw new RuntimeException("Scan timed out after " + timeoutSeconds + "s");
            }

            Thread.sleep(2000);
        }
    }
}
```

---

## 12. Secure Configuration Management {#secure-config}

```yaml
# application-prod.yml
spring:
  # Never log full SQL in production
  jpa:
    show-sql: false
    properties:
      hibernate:
        format_sql: false

  # Disable Actuator endpoints that expose sensitive info
  boot:
    admin:
      client:
        enabled: false

# Actuator security
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      roles: ACTUATOR
    info:
      enabled: true
    env:
      enabled: false  # Never expose env vars!
    beans:
      enabled: false
    mappings:
      enabled: false
    conditions:
      enabled: false
    configprops:
      enabled: false

  # Secure actuator with separate credentials
  security:
    enabled: true

# Disable H2 console in production
spring:
  h2:
    console:
      enabled: false

# Server error handling - never expose stack traces
server:
  error:
    include-stacktrace: never
    include-message: never
    include-binding-errors: never
    include-exception: false
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
        same-site: strict

# Security configuration
security:
  hmac:
    secret: ${HMAC_SECRET}  # From Secrets Manager
  ip:
    admin-allowlist: 10.0.0.0/8,172.16.0.0/12
```

---

## 13. Real Example: Hardened REST API {#real-example}

### Security Filter Chain (Complete)

```java
package com.example.security.config;

import com.example.security.filter.ApiKeyAuthFilter;
import com.example.security.filter.IpFilterFilter;
import com.example.security.ratelimit.RateLimitFilter;
import com.example.security.signing.HmacSignatureFilter;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class HardenedSecurityConfig {

    private final ApiKeyAuthFilter apiKeyAuthFilter;
    private final RateLimitFilter rateLimitFilter;
    private final IpFilterFilter ipFilterFilter;
    private final HmacSignatureFilter hmacSignatureFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            // Stateless - no sessions
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Disable CSRF (stateless JWT API)
            .csrf(csrf -> csrf.disable())

            // Security headers
            .headers(headers -> headers
                .frameOptions(frame -> frame.deny())
                .contentTypeOptions(cto -> {})
                .xssProtection(xss -> xss.enable())
                .contentSecurityPolicy(csp -> csp.policyDirectives(
                    "default-src 'self'; frame-ancestors 'none'"))
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000).includeSubDomains(true).preload(true))
            )

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/webhooks/**").permitAll()  // Protected by HMAC
                .anyRequest().authenticated()
            )

            // JWT authentication
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> {})
            )

            // Custom filters (order matters)
            .addFilterBefore(ipFilterFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(rateLimitFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(apiKeyAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(hmacSignatureFilter, UsernamePasswordAuthenticationFilter.class)

            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, e) -> {
                    response.setStatus(401);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"error\":\"UNAUTHORIZED\"}");
                })
                .accessDeniedHandler((request, response, e) -> {
                    response.setStatus(403);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"error\":\"FORBIDDEN\"}");
                })
            )

            .build();
    }
}
```

### Secured Product Controller

```java
package com.example.security.controller;

import com.example.security.dto.CreateProductRequest;
import com.example.security.entity.Product;
import com.example.security.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

@Slf4j
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class SecuredProductController {

    private final ProductService productService;

    @GetMapping
    public Page<Product> listProducts(
            @PageableDefault(size = 20, max = 100) Pageable pageable) {
        return productService.findAll(pageable);
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasRole('USER') or hasScope('products:read')")
    public Product getProduct(@PathVariable Long id,
                               @AuthenticationPrincipal Jwt jwt) {
        Product product = productService.findById(id);

        // API1: Object Level Authorization - check ownership
        if (!product.getOwnerId().equals(jwt.getSubject()) && !hasAdminRole(jwt)) {
            throw new org.springframework.security.access.AccessDeniedException(
                    "Access denied to product: " + id);
        }

        return product;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @PreAuthorize("hasRole('USER') or hasScope('products:write')")
    public Product createProduct(@Valid @RequestBody CreateProductRequest request,
                                  @AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getSubject();
        log.info("Creating product: user={}, name={}", userId, request.getName());
        return productService.create(request, userId);
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('USER') or hasScope('products:write')")
    public Product updateProduct(@PathVariable Long id,
                                  @Valid @RequestBody CreateProductRequest request,
                                  @AuthenticationPrincipal Jwt jwt) {
        // Verify ownership before update (API1)
        productService.verifyOwnership(id, jwt.getSubject());
        return productService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
    public void deleteProduct(@PathVariable Long id,
                               @AuthenticationPrincipal Jwt jwt) {
        // Admin can delete any; user can only delete own
        if (!hasAdminRole(jwt)) {
            productService.verifyOwnership(id, jwt.getSubject());
        }
        productService.delete(id);
    }

    @GetMapping("/admin/all")
    @PreAuthorize("hasRole('ADMIN')")  // API5: Function Level Authorization
    public Page<Product> adminListAll(
            @PageableDefault(size = 50, max = 500) Pageable pageable) {
        return productService.findAllIncludingDeleted(pageable);
    }

    private boolean hasAdminRole(Jwt jwt) {
        var roles = jwt.getClaimAsStringList("roles");
        return roles != null && roles.contains("ADMIN");
    }
}
```

### Global Exception Handler (Secure)

```java
package com.example.security.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.authentication.InsufficientAuthenticationException;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.time.Instant;
import java.util.Map;
import java.util.stream.Collectors;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.toMap(
                        FieldError::getField,
                        f -> f.getDefaultMessage() != null ? f.getDefaultMessage() : "Invalid value",
                        (a, b) -> a
                ));

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setDetail("Request validation failed");
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ProblemDetail handleAccessDenied(AccessDeniedException ex) {
        // Don't log user-caused 403s at ERROR level
        log.debug("Access denied: {}", ex.getMessage());
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
        problem.setTitle("Access Denied");
        problem.setDetail("You don't have permission to access this resource");
        // ⚠️ Never include ex.getMessage() - may expose internals
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGenericException(Exception ex, WebRequest request) {
        // Generate error ID for correlation without exposing details
        String errorId = java.util.UUID.randomUUID().toString().substring(0, 8);
        log.error("Unhandled exception [errorId={}]: {}", errorId, ex.getMessage(), ex);

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problem.setTitle("Internal Server Error");
        problem.setDetail("An unexpected error occurred. Reference: " + errorId);
        // ⚠️ Never return stack trace or exception message to client
        return problem;
    }
}
```

---

## 14. Summary {#summary}

| Security Control | Implementation | Priority |
|----------------|----------------|---------|
| **Input Validation** | Jakarta Validation + custom annotations | Critical |
| **SQL Injection** | JPA named params + Criteria API | Critical |
| **XSS Prevention** | AntiSamy sanitizer + CSP headers | Critical |
| **CSRF** | Disabled for stateless JWT APIs | High |
| **Rate Limiting** | Bucket4j per-user/IP with Redis backend | High |
| **API Key Auth** | SHA-256 hashed keys + last-used tracking | High |
| **IP Filtering** | Allowlist for admin + dynamic blocklist | Medium |
| **Request Signing** | HMAC-SHA256 for webhooks + replay protection | High |
| **Log Masking** | Regex masking for PII in log output | Medium |
| **Security Headers** | CSP, HSTS, X-Frame-Options, Referrer-Policy | High |
| **Object Auth (BOLA)** | Check resource ownership per request | Critical |
| **Error Handling** | Generic errors + internal error IDs | Medium |
| **Dependency Scanning** | OWASP Dependency Check in CI | High |
| **Container Scanning** | Trivy/Grype on Docker images in CI | High |

### Quick Security Checklist

```
[ ] Never log passwords, tokens, API keys
[ ] Validate all input at controller entry point
[ ] Use parameterized queries (never string concat in SQL)
[ ] Check resource ownership on every request (BOLA)
[ ] Apply rate limiting on auth and public endpoints
[ ] Use HTTPS everywhere (HSTS)
[ ] Add Content-Security-Policy header
[ ] Rotate secrets regularly (prefer Secrets Manager)
[ ] Scan dependencies weekly (OWASP)
[ ] Scan container images on every build (Trivy)
[ ] Never expose stack traces in API responses
[ ] Disable unused Actuator endpoints
```

---

> **Next: Part 051 - Advanced Database Patterns** — Read replicas, optimistic/pessimistic locking, Flyway migrations, soft delete, Hibernate Envers auditing, full-text search, and JSON columns.
