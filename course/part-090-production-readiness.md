# Part 090: Production Readiness Checklist and Best Practices

## เนื้อหาในส่วนนี้
- Production-grade Spring Boot configuration
- Health checks and readiness/liveness probes
- Graceful shutdown
- Connection pool tuning
- JVM optimization flags
- Logging best practices (structured logging)
- Error handling strategy
- Feature flags and configuration management
- Secrets management with Vault
- Production deployment checklist

---

## 1. Application Configuration for Production

```yaml
# application-production.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
      connection-test-query: SELECT 1
  jpa:
    open-in-view: false           # Prevents lazy loading issues in controllers
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          fetch_size: 100
        order_inserts: true
        order_updates: true
        generate_statistics: false  # Disable in production (overhead)
  cache:
    type: redis
  data:
    redis:
      host: ${REDIS_HOST}
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 100MB

server:
  port: 8080
  shutdown: graceful
  tomcat:
    max-threads: 200
    min-spare-threads: 20
    accept-count: 100
    connection-timeout: 5000
    max-connections: 10000
  compression:
    enabled: true
    mime-types: application/json,application/xml,text/html,text/plain
    min-response-size: 1024
  forward-headers-strategy: framework   # For proxy/load balancer

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true          # /health/liveness and /health/readiness
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: production

logging:
  level:
    root: WARN
    com.myapp: INFO
    org.springframework.web: WARN
    org.hibernate.SQL: WARN
  pattern:
    console: ""   # Disable console in production - use file/stdout
  structured:
    format:
      console: ecs  # Elastic Common Schema JSON format
```

---

## 2. Custom Health Indicators

```java
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Component;
import java.util.Map;

// Custom health indicator for external dependencies
@Component
public class ExternalApiHealthIndicator extends AbstractHealthIndicator {
    
    private final ExternalApiClient externalApiClient;
    
    public ExternalApiHealthIndicator(ExternalApiClient externalApiClient) {
        super("External API health check failed");
        this.externalApiClient = externalApiClient;
    }
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            long start = System.currentTimeMillis();
            boolean reachable = externalApiClient.ping();
            long duration = System.currentTimeMillis() - start;
            
            if (reachable) {
                builder.up()
                    .withDetail("responseTime", duration + "ms")
                    .withDetail("url", externalApiClient.getBaseUrl());
            } else {
                builder.down()
                    .withDetail("reason", "API not reachable");
            }
        } catch (Exception e) {
            builder.down(e);
        }
    }
}

// Disk space health indicator
@Component  
public class DiskSpaceHealthIndicator extends AbstractHealthIndicator {
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        java.io.File disk = new java.io.File("/");
        long freeBytes = disk.getFreeSpace();
        long totalBytes = disk.getTotalSpace();
        double freePercent = (double) freeBytes / totalBytes * 100;
        
        if (freePercent < 5) {  // Less than 5% free
            builder.down()
                .withDetail("free", formatBytes(freeBytes))
                .withDetail("total", formatBytes(totalBytes))
                .withDetail("freePercent", String.format("%.1f%%", freePercent));
        } else {
            builder.up()
                .withDetail("free", formatBytes(freeBytes))
                .withDetail("freePercent", String.format("%.1f%%", freePercent));
        }
    }
    
    private String formatBytes(long bytes) {
        return String.format("%.2f GB", bytes / (1024.0 * 1024 * 1024));
    }
}

// Readiness probe: marks app as not ready during startup or maintenance
@Component
public class DatabaseReadinessIndicator implements org.springframework.boot.actuate.availability.ReadinessStateHealthIndicator {
    // Auto-configured via spring.boot.actuate.availability
}

// Liveness probe: marks app for restart if it's stuck
@Component
public class ApplicationLivenessIndicator implements org.springframework.boot.actuate.availability.LivenessStateHealthIndicator {
    // Auto-configured via spring.boot.actuate.availability
}
```

---

## 3. Graceful Shutdown

```java
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.*;
import org.springframework.stereotype.Component;
import java.util.concurrent.*;

@Configuration
public class GracefulShutdownConfig {
    
    @Bean
    public ExecutorService asyncExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
    
    @Bean
    public GracefulShutdownHandler gracefulShutdownHandler(ExecutorService asyncExecutor) {
        return new GracefulShutdownHandler(asyncExecutor);
    }
}

@Component
public class GracefulShutdownHandler 
        implements ApplicationListener<ContextClosedEvent> {
    
    private final ExecutorService executor;
    private final AtomicBoolean shuttingDown = new AtomicBoolean(false);
    
    public GracefulShutdownHandler(ExecutorService executor) {
        this.executor = executor;
    }
    
    public boolean isShuttingDown() {
        return shuttingDown.get();
    }
    
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        shuttingDown.set(true);
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}

// application.properties: server.shutdown=graceful
// spring.lifecycle.timeout-per-shutdown-phase=30s
```

---

## 4. Structured Logging

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.IOException;
import java.util.UUID;

// MDC filter - adds correlation IDs to all log lines
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestLoggingFilter extends OncePerRequestFilter {
    
    private static final String REQUEST_ID = "requestId";
    private static final String USER_ID = "userId";
    private static final String SESSION_ID = "sessionId";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain chain) throws ServletException, IOException {
        try {
            String requestId = getOrCreateRequestId(request);
            MDC.put(REQUEST_ID, requestId);
            MDC.put(SESSION_ID, request.getSession(false) != null ? 
                request.getSession().getId().substring(0, 8) : "none");
            
            // Add user ID if authenticated
            var auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth != null && auth.isAuthenticated() && !auth.getName().equals("anonymousUser")) {
                MDC.put(USER_ID, auth.getName());
            }
            
            // Add to response header for client correlation
            response.setHeader("X-Request-ID", requestId);
            
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
    
    private String getOrCreateRequestId(HttpServletRequest request) {
        String id = request.getHeader("X-Request-ID");
        return (id != null && !id.isBlank()) ? id : UUID.randomUUID().toString().substring(0, 8);
    }
}

// Structured log output
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    
    public Order processOrder(CreateOrderRequest request) {
        log.info("Processing order",
            Map.of(
                "customerId", request.customerId(),
                "items", request.items().size(),
                "totalAmount", request.totalAmount()
            )
        );
        
        // Structured logging with key=value (auto-parsed by Elasticsearch)
        log.atInfo()
            .addKeyValue("orderId", "ORD-123")
            .addKeyValue("customerId", request.customerId())
            .addKeyValue("processingTime", 45)
            .log("Order created successfully");
        
        return null;
    }
}

import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;
import java.util.Map;
import java.util.concurrent.atomic.AtomicBoolean;
```

---

## 5. Global Exception Handling

```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.WebRequest;
import java.time.Instant;
import java.util.*;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    
    // Business exceptions
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex, WebRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setProperty("timestamp", Instant.now());
        problem.setProperty("requestId", MDC.get("requestId"));
        return problem;
    }
    
    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusinessException(BusinessException ex, WebRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
        problem.setTitle("Business Rule Violation");
        problem.setProperty("errorCode", ex.getErrorCode());
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
    
    // Validation errors
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            org.springframework.web.bind.MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        
        Map<String, List<String>> fieldErrors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.computeIfAbsent(error.getField(), k -> new ArrayList<>())
                      .add(error.getDefaultMessage()));
        
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setDetail("One or more fields have validation errors");
        problem.setProperty("errors", fieldErrors);
        problem.setProperty("timestamp", Instant.now());
        
        return ResponseEntity.badRequest().body(problem);
    }
    
    // Security exceptions (don't expose details)
    @ExceptionHandler(AccessDeniedException.class)
    public ProblemDetail handleAccessDenied(AccessDeniedException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.FORBIDDEN, "Access denied");
    }
    
    // Catch-all (don't expose internal details!)
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleAll(Exception ex, WebRequest request) {
        log.error("Unhandled exception", ex);
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        problem.setProperty("requestId", MDC.get("requestId"));
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}

// Domain exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Object id) {
        super(resource + " not found with id: " + id);
    }
}

public class BusinessException extends RuntimeException {
    private final String errorCode;
    
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() { return errorCode; }
}

import org.slf4j.MDC;
import org.springframework.security.access.AccessDeniedException;
import lombok.extern.slf4j.Slf4j;
```

---

## 6. JVM Production Flags

```bash
# Dockerfile / Kubernetes deployment
JAVA_OPTS="\
  # Memory settings
  -Xms512m \
  -Xmx512m \
  -XX:MaxMetaspaceSize=256m \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  
  # GC: G1GC (default Java 9+) or ZGC for low-latency
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=8m \
  
  # JIT / performance
  -XX:+TieredCompilation \
  -XX:+UseStringDeduplication \
  
  # Diagnostics
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -XX:+ExitOnOutOfMemoryError \
  
  # Flight Recorder (profiling with near-zero overhead)
  -XX:+UnlockDiagnosticVMOptions \
  -XX:+DebugNonSafepoints \
  
  # Encoding
  -Dfile.encoding=UTF-8 \
  -Djava.security.egd=file:/dev/./urandom"
```

---

## 7. HashiCorp Vault Secrets Management

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml (or application.yml with spring.config.import)
spring:
  config:
    import: "vault://"
  cloud:
    vault:
      host: vault.internal
      port: 8200
      scheme: https
      authentication: KUBERNETES   # or TOKEN for dev
      kubernetes:
        role: my-app
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kv:
        enabled: true
        backend: secret
        default-context: myapp    # reads secret/myapp
        application-name: myapp
        profiles:
          - production
```

```java
// Secrets are automatically injected as Spring properties
// secret/myapp/production in Vault → @Value("${db.password}")
@Configuration
public class DatabaseConfig {
    
    @Value("${db.url}")
    private String dbUrl;
    
    @Value("${db.username}")
    private String dbUsername;
    
    @Value("${db.password}")
    private String dbPassword;
    
    @Bean
    public DataSource dataSource() {
        var config = new HikariConfig();
        config.setJdbcUrl(dbUrl);
        config.setUsername(dbUsername);
        config.setPassword(dbPassword);
        return new HikariDataSource(config);
    }
}

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
```

---

## 8. Feature Flags

```xml
<dependency>
    <groupId>io.getunleash</groupId>
    <artifactId>unleash-client-java</artifactId>
    <version>9.2.0</version>
</dependency>
```

```java
@Configuration
public class UnleashConfig {
    
    @Bean
    public Unleash unleash(@Value("${unleash.api-url}") String apiUrl,
                            @Value("${unleash.api-token}") String apiToken) {
        UnleashConfig config = UnleashConfig.newBuilder()
            .appName("my-app")
            .instanceId("instance-" + java.net.InetAddress.getLoopbackAddress().getHostName())
            .unleashAPI(apiUrl)
            .customHttpHeader("Authorization", apiToken)
            .build();
        return new DefaultUnleash(config);
    }
}

@Service
public class CheckoutService {
    
    private final Unleash unleash;
    
    public CheckoutService(Unleash unleash) {
        this.unleash = unleash;
    }
    
    public CheckoutResult checkout(Cart cart, User user) {
        // Gradual rollout to % of users
        UnleashContext context = UnleashContext.newBuilder()
            .userId(user.getId().toString())
            .addProperty("plan", user.getPlan())
            .build();
        
        if (unleash.isEnabled("new-checkout-flow", context)) {
            return newCheckoutFlow(cart, user);
        } else {
            return legacyCheckoutFlow(cart, user);
        }
    }
    
    private CheckoutResult newCheckoutFlow(Cart cart, User user) {
        // New implementation
        return new CheckoutResult("SUCCESS", "new-flow");
    }
    
    private CheckoutResult legacyCheckoutFlow(Cart cart, User user) {
        return new CheckoutResult("SUCCESS", "legacy-flow");
    }
}

record CheckoutResult(String status, String flow) {}
record Cart(String id) {}
record User(Long id, String plan) {}

import io.getunleash.*;
```

---

## Production Readiness Checklist

| Category | Item | Done? |
|----------|------|-------|
| **Config** | No hardcoded secrets | ✅ |
| **Config** | Profiles (dev/staging/prod) separated | ✅ |
| **Config** | External config (Vault/ConfigServer) | ✅ |
| **Health** | `/actuator/health/liveness` configured | ✅ |
| **Health** | `/actuator/health/readiness` configured | ✅ |
| **Health** | External service health checks | ✅ |
| **Shutdown** | Graceful shutdown enabled | ✅ |
| **Logging** | Structured JSON logging | ✅ |
| **Logging** | Correlation IDs in MDC | ✅ |
| **Logging** | No sensitive data in logs | ✅ |
| **Errors** | Global exception handler | ✅ |
| **Errors** | No stack traces in responses | ✅ |
| **DB** | HikariCP pool tuned | ✅ |
| **DB** | `spring.jpa.open-in-view=false` | ✅ |
| **JVM** | Container-aware memory (`-XX:+UseContainerSupport`) | ✅ |
| **JVM** | OOM heap dump configured | ✅ |
| **Security** | HTTPS only | ✅ |
| **Security** | Security headers (HSTS, CSP) | ✅ |
| **Security** | Actuator secured | ✅ |
| **Monitoring** | Prometheus metrics exported | ✅ |
| **Monitoring** | Business metrics instrumented | ✅ |
| **Tracing** | Distributed tracing enabled | ✅ |
| **Testing** | Load test before launch | ✅ |
| **Rollout** | Feature flags for new features | ✅ |

---

**Part 091:** Spring State Machine - workflows and stateful processes
