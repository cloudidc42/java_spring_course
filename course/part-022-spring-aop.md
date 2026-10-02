# Part 022: Spring AOP - Aspect-Oriented Programming

## เนื้อหาในส่วนนี้
- AOP Concepts: Aspect, Advice, Pointcut, JoinPoint, Weaving
- Spring AOP vs AspectJ
- @Before, @After, @Around, @AfterReturning, @AfterThrowing
- Pointcut Expressions
- Logging Aspect
- Performance Monitoring Aspect
- Security/Authorization Aspect
- Transaction Aspect (custom)
- Custom Annotations with AOP
- Real-world Complete Example

---

## 1. AOP Concepts

AOP (Aspect-Oriented Programming) ช่วยแยก cross-cutting concerns ออกจาก business logic

### Terminology

```
Cross-cutting concerns คือ functionality ที่ใช้งานทั่วทั้งระบบ เช่น:
- Logging          - Security
- Transactions     - Caching
- Error Handling   - Performance Monitoring
- Auditing         - Rate Limiting

โดยไม่ต้องเขียนซ้ำในทุก class
```

### Core Terms

```
┌─────────────────────────────────────────────────────────────────┐
│ ASPECT    = class ที่รวม cross-cutting logic                      │
│           (@Aspect annotated class)                              │
│                                                                  │
│ ADVICE    = action ที่ aspect ทำ                                   │
│           (@Before, @After, @Around, etc.)                       │
│                                                                  │
│ POINTCUT  = expression ที่ระบุว่า advice ทำงานที่ไหน               │
│           (execution(* com.example.service.*.*(..)))             │
│                                                                  │
│ JOIN POINT = point ใน execution (method call, exception, etc.)   │
│                                                                  │
│ WEAVING   = process ที่ inject aspect เข้า target                 │
│           (Spring uses proxy-based weaving)                      │
│                                                                  │
│ TARGET    = object ที่ถูก aspect ดูแล                              │
└─────────────────────────────────────────────────────────────────┘
```

### Maven Dependencies

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.1.1</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>6.1.1</version>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.21</version>
</dependency>
```

---

## 2. Enable AOP in Spring

```java
import org.springframework.context.annotation.*;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@ComponentScan("com.example")
@EnableAspectJAutoProxy  // Enable AOP support
public class AppConfig {
    // @EnableAspectJAutoProxy(proxyTargetClass = true) // Force CGLIB proxy
}

// With Spring Boot - just add dependency, @EnableAspectJAutoProxy auto-configured
```

---

## 3. Pointcut Expressions

```java
import org.aspectj.lang.annotation.*;

@Aspect
@Component
public class PointcutExamplesAspect {
    
    // === execution() - most commonly used ===
    // execution(modifiers? return-type declaring-type? method-name(param-types) throws?)
    
    // Any method in UserService
    @Pointcut("execution(* com.example.service.UserService.*(..))")
    public void anyUserServiceMethod() {}
    
    // Any public method returning String
    @Pointcut("execution(public String *(..))")
    public void anyPublicStringMethod() {}
    
    // Methods starting with "find"
    @Pointcut("execution(* find*(..))")
    public void findMethods() {}
    
    // Methods with exactly one String parameter
    @Pointcut("execution(* *(String))")
    public void singleStringParam() {}
    
    // Any method in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // Any method in service or sub-packages
    @Pointcut("execution(* com.example.service..*.*(..))")
    public void serviceLayerDeep() {}
    
    // === within() - type matching ===
    
    // Any method in class
    @Pointcut("within(com.example.service.UserService)")
    public void withinUserService() {}
    
    // Any method in package
    @Pointcut("within(com.example.service.*)")
    public void withinServicePackage() {}
    
    // Any method in classes annotated with @Service
    @Pointcut("within(@org.springframework.stereotype.Service *)")
    public void withinServiceAnnotated() {}
    
    // === @annotation() - method annotation matching ===
    
    // Methods annotated with @Transactional
    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethods() {}
    
    // Methods annotated with custom @Loggable
    @Pointcut("@annotation(com.example.annotation.Loggable)")
    public void loggableMethods() {}
    
    // === bean() - Spring bean name matching ===
    
    // Specific bean
    @Pointcut("bean(userService)")
    public void userServiceBean() {}
    
    // Beans matching pattern
    @Pointcut("bean(*Service)")
    public void allServiceBeans() {}
    
    // === Combining pointcuts ===
    
    @Pointcut("serviceLayer() && !transactionalMethods()")
    public void nonTransactionalServiceMethods() {}
    
    @Pointcut("serviceLayer() || withinServiceAnnotated()")
    public void allServiceRelated() {}
    
    // Actual advice using combined pointcut
    @Before("serviceLayer()")
    public void beforeServiceMethod() {
        System.out.println("Before any service method");
    }
}
```

---

## 4. Advice Types

```java
import org.aspectj.lang.*;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class AdviceTypesExample {
    
    // === @Before - runs before method ===
    @Before("execution(* com.example.service.*.*(..))")
    public void beforeAdvice(JoinPoint joinPoint) {
        System.out.println("BEFORE: " + joinPoint.getSignature().getName());
        
        // Access method arguments
        Object[] args = joinPoint.getArgs();
        for (Object arg : args) {
            System.out.println("  Argument: " + arg);
        }
        
        // Access target object
        Object target = joinPoint.getTarget();
        System.out.println("  Target: " + target.getClass().getSimpleName());
    }
    
    // === @AfterReturning - runs after successful return ===
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result"  // bind return value
    )
    public void afterReturningAdvice(JoinPoint joinPoint, Object result) {
        System.out.println("AFTER_RETURNING: " + joinPoint.getSignature().getName());
        System.out.println("  Returned: " + result);
    }
    
    // === @AfterThrowing - runs after exception ===
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "exception"  // bind exception
    )
    public void afterThrowingAdvice(JoinPoint joinPoint, Exception exception) {
        System.out.println("AFTER_THROWING: " + joinPoint.getSignature().getName());
        System.out.println("  Exception: " + exception.getMessage());
    }
    
    // === @After (finally) - runs always ===
    @After("execution(* com.example.service.*.*(..))")
    public void afterFinallyAdvice(JoinPoint joinPoint) {
        System.out.println("AFTER_FINALLY: " + joinPoint.getSignature().getName());
        // Runs whether method succeeded or threw exception
    }
    
    // === @Around - most powerful, wraps the method ===
    @Around("execution(* com.example.service.*.*(..))")
    public Object aroundAdvice(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("AROUND_BEFORE: " + pjp.getSignature().getName());
        
        try {
            // Proceed with original method call
            Object result = pjp.proceed();
            System.out.println("AROUND_AFTER_SUCCESS");
            return result;
            
        } catch (Throwable t) {
            System.out.println("AROUND_AFTER_EXCEPTION: " + t.getMessage());
            throw t;  // Re-throw or handle
        } finally {
            System.out.println("AROUND_FINALLY");
        }
    }
    
    // @Around can also modify arguments
    @Around("execution(* com.example.service.UserService.saveUser(..))")
    public Object sanitizeInput(ProceedingJoinPoint pjp) throws Throwable {
        Object[] args = pjp.getArgs();
        
        // Sanitize string arguments
        for (int i = 0; i < args.length; i++) {
            if (args[i] instanceof String s) {
                args[i] = s.trim().toLowerCase();
            }
        }
        
        // Proceed with modified arguments
        return pjp.proceed(args);
    }
}
```

---

## 5. Logging Aspect

```java
import org.aspectj.lang.*;
import org.aspectj.lang.annotation.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import java.util.Arrays;

@Aspect
@Component
public class LoggingAspect {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);
    
    // Log all service layer methods
    @Around("execution(* com.example..service..*.*(..))")
    public Object logServiceMethod(ProceedingJoinPoint pjp) throws Throwable {
        String className = pjp.getTarget().getClass().getSimpleName();
        String methodName = pjp.getSignature().getName();
        Object[] args = pjp.getArgs();
        
        logger.debug("Entering [{}.{}] with args: {}", 
            className, methodName, Arrays.toString(args));
        
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = pjp.proceed();
            long duration = System.currentTimeMillis() - startTime;
            
            logger.debug("Exiting [{}.{}] with result: {} ({}ms)", 
                className, methodName, result, duration);
            
            return result;
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            logger.error("Exception in [{}.{}] after {}ms: {}", 
                className, methodName, duration, e.getMessage());
            throw e;
        }
    }
    
    // Log REST controller requests
    @Before("within(@org.springframework.web.bind.annotation.RestController *)")
    public void logRequest(JoinPoint jp) {
        logger.info("Request: {}.{}({})", 
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            Arrays.toString(jp.getArgs()));
    }
    
    // Log repository calls
    @Around("execution(* com.example..repository..*.*(..))")
    public Object logRepositoryMethod(ProceedingJoinPoint pjp) throws Throwable {
        if (logger.isDebugEnabled()) {
            logger.debug("DB: {}", pjp.getSignature().toShortString());
        }
        return pjp.proceed();
    }
}
```

---

## 6. Performance Monitoring Aspect

```java
import org.aspectj.lang.*;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.*;
import java.util.*;

@Aspect
@Component
public class PerformanceMonitoringAspect {
    
    // Track method call statistics
    private final ConcurrentHashMap<String, MethodStats> stats = new ConcurrentHashMap<>();
    
    @Around("execution(* com.example.service..*.*(..))")
    public Object monitorPerformance(ProceedingJoinPoint pjp) throws Throwable {
        String methodKey = pjp.getSignature().getDeclaringTypeName() + "." 
                         + pjp.getSignature().getName();
        
        long start = System.currentTimeMillis();
        boolean success = true;
        
        try {
            return pjp.proceed();
        } catch (Exception e) {
            success = false;
            throw e;
        } finally {
            long duration = System.currentTimeMillis() - start;
            
            stats.computeIfAbsent(methodKey, k -> new MethodStats())
                 .record(duration, success);
            
            // Alert on slow methods
            if (duration > 1000) {
                System.err.printf("SLOW METHOD: %s took %dms%n", methodKey, duration);
            }
        }
    }
    
    public Map<String, MethodStats> getStats() {
        return Collections.unmodifiableMap(stats);
    }
    
    public void printReport() {
        System.out.println("\n=== Performance Report ===");
        stats.entrySet().stream()
            .sorted(Map.Entry.<String, MethodStats>comparingByValue(
                Comparator.comparingLong(MethodStats::getTotalTime).reversed()))
            .forEach(entry -> {
                MethodStats s = entry.getValue();
                System.out.printf("%-60s | calls=%d, avg=%dms, max=%dms, errors=%d%n",
                    entry.getKey(), s.getCallCount(), s.getAvgTime(), 
                    s.getMaxTime(), s.getErrorCount());
            });
    }
    
    // Statistics tracker
    static class MethodStats {
        private final AtomicLong callCount = new AtomicLong();
        private final AtomicLong totalTime = new AtomicLong();
        private final AtomicLong maxTime = new AtomicLong();
        private final AtomicLong errorCount = new AtomicLong();
        
        void record(long duration, boolean success) {
            callCount.incrementAndGet();
            totalTime.addAndGet(duration);
            maxTime.updateAndGet(current -> Math.max(current, duration));
            if (!success) errorCount.incrementAndGet();
        }
        
        long getCallCount() { return callCount.get(); }
        long getTotalTime() { return totalTime.get(); }
        long getAvgTime() { 
            long count = callCount.get();
            return count == 0 ? 0 : totalTime.get() / count;
        }
        long getMaxTime() { return maxTime.get(); }
        long getErrorCount() { return errorCount.get(); }
    }
}
```

---

## 7. Custom Annotation-based AOP

```java
import java.lang.annotation.*;

// === Custom Annotations ===

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Loggable {
    String value() default "";
    boolean logArgs() default true;
    boolean logResult() default true;
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimit {
    int maxRequests() default 100;
    int windowSeconds() default 60;
}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequiresPermission {
    String[] value();
    boolean requireAll() default false; // false = any, true = all
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Cacheable2 {
    String key() default "";
    int ttlSeconds() default 300;
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Retry {
    int maxAttempts() default 3;
    long delayMs() default 1000;
    Class<? extends Exception>[] retryOn() default {Exception.class};
}
```

```java
import org.aspectj.lang.*;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;
import java.lang.reflect.Method;
import java.util.*;
import java.util.concurrent.*;

// === @Loggable Aspect ===
@Aspect
@Component
public class LoggableAspect {
    
    @Around("@annotation(loggable)")
    public Object handleLoggable(ProceedingJoinPoint pjp, Loggable loggable) throws Throwable {
        String label = loggable.value().isEmpty() 
            ? pjp.getSignature().getName() 
            : loggable.value();
        
        if (loggable.logArgs()) {
            System.out.printf("[LOG] %s called with: %s%n", 
                label, Arrays.toString(pjp.getArgs()));
        } else {
            System.out.printf("[LOG] %s called%n", label);
        }
        
        Object result = pjp.proceed();
        
        if (loggable.logResult()) {
            System.out.printf("[LOG] %s returned: %s%n", label, result);
        }
        
        return result;
    }
}

// === @RateLimit Aspect ===
@Aspect
@Component
public class RateLimitAspect {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Around("@annotation(rateLimit)")
    public Object handleRateLimit(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        String key = pjp.getSignature().toLongString();
        
        RateLimiter limiter = limiters.computeIfAbsent(key, k ->
            new RateLimiter(rateLimit.maxRequests(), rateLimit.windowSeconds()));
        
        if (!limiter.tryAcquire()) {
            throw new RateLimitExceededException(
                "Rate limit exceeded: max " + rateLimit.maxRequests() + 
                " requests per " + rateLimit.windowSeconds() + " seconds");
        }
        
        return pjp.proceed();
    }
    
    // Simple sliding window rate limiter
    static class RateLimiter {
        private final int maxRequests;
        private final long windowMs;
        private final Queue<Long> requests = new ConcurrentLinkedQueue<>();
        
        RateLimiter(int maxRequests, int windowSeconds) {
            this.maxRequests = maxRequests;
            this.windowMs = windowSeconds * 1000L;
        }
        
        boolean tryAcquire() {
            long now = System.currentTimeMillis();
            long windowStart = now - windowMs;
            
            // Remove old requests outside window
            while (!requests.isEmpty() && requests.peek() < windowStart) {
                requests.poll();
            }
            
            if (requests.size() < maxRequests) {
                requests.offer(now);
                return true;
            }
            return false;
        }
    }
}

class RateLimitExceededException extends RuntimeException {
    RateLimitExceededException(String message) { super(message); }
}

// === @RequiresPermission Aspect ===
@Aspect
@Component
public class SecurityAspect {
    
    // Simulated security context
    private final ThreadLocal<Set<String>> currentPermissions = 
        ThreadLocal.withInitial(HashSet::new);
    
    @Around("@annotation(requiresPermission) || @within(requiresPermission)")
    public Object checkPermission(ProceedingJoinPoint pjp, 
                                  RequiresPermission requiresPermission) throws Throwable {
        Set<String> required = Set.of(requiresPermission.value());
        Set<String> current = currentPermissions.get();
        
        boolean hasAccess = requiresPermission.requireAll()
            ? current.containsAll(required)
            : required.stream().anyMatch(current::contains);
        
        if (!hasAccess) {
            throw new AccessDeniedException(
                "Required permissions: " + required + 
                ", current permissions: " + current);
        }
        
        return pjp.proceed();
    }
    
    // Set permissions for current thread (simulated auth)
    public void setPermissions(String... permissions) {
        currentPermissions.set(new HashSet<>(Arrays.asList(permissions)));
    }
    
    public void clearPermissions() {
        currentPermissions.remove();
    }
}

class AccessDeniedException extends RuntimeException {
    AccessDeniedException(String message) { super(message); }
}

// === @Cacheable2 Aspect ===
@Aspect
@Component
public class CachingAspect {
    
    private final Map<String, CacheEntry> cache = new ConcurrentHashMap<>();
    
    @Around("@annotation(cacheable)")
    public Object handleCache(ProceedingJoinPoint pjp, Cacheable2 cacheable) throws Throwable {
        String cacheKey = buildKey(pjp, cacheable);
        
        CacheEntry entry = cache.get(cacheKey);
        if (entry != null && !entry.isExpired()) {
            System.out.println("[CACHE HIT] " + cacheKey);
            return entry.value();
        }
        
        Object result = pjp.proceed();
        
        if (result != null) {
            cache.put(cacheKey, new CacheEntry(result, 
                System.currentTimeMillis() + cacheable.ttlSeconds() * 1000L));
            System.out.println("[CACHE SET] " + cacheKey);
        }
        
        return result;
    }
    
    private String buildKey(ProceedingJoinPoint pjp, Cacheable2 cacheable) {
        String keyTemplate = cacheable.key();
        if (!keyTemplate.isEmpty()) {
            return keyTemplate + ":" + Arrays.toString(pjp.getArgs());
        }
        return pjp.getSignature().getName() + ":" + Arrays.toString(pjp.getArgs());
    }
    
    record CacheEntry(Object value, long expiresAt) {
        boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
    }
}

// === @Retry Aspect ===
@Aspect
@Component
public class RetryAspect {
    
    @Around("@annotation(retry)")
    public Object handleRetry(ProceedingJoinPoint pjp, Retry retry) throws Throwable {
        int attempts = 0;
        Throwable lastException = null;
        
        while (attempts < retry.maxAttempts()) {
            try {
                return pjp.proceed();
            } catch (Throwable t) {
                attempts++;
                lastException = t;
                
                // Check if we should retry this exception type
                boolean shouldRetry = Arrays.stream(retry.retryOn())
                    .anyMatch(retryType -> retryType.isAssignableFrom(t.getClass()));
                
                if (!shouldRetry || attempts >= retry.maxAttempts()) {
                    throw t;
                }
                
                System.out.printf("[RETRY] Attempt %d/%d failed for %s: %s%n",
                    attempts, retry.maxAttempts(), 
                    pjp.getSignature().getName(), t.getMessage());
                
                if (retry.delayMs() > 0) {
                    Thread.sleep(retry.delayMs());
                }
            }
        }
        
        throw lastException;
    }
}
```

---

## 8. Usage Examples

```java
import org.springframework.stereotype.Service;
import java.util.*;

// Service using custom annotations
@Service
public class ProductService2 {
    
    private final Map<String, String> db = new HashMap<>(Map.of(
        "P001", "Laptop Pro",
        "P002", "Wireless Mouse"
    ));
    
    @Loggable(value = "Find Product", logArgs = true, logResult = true)
    @Cacheable2(key = "product", ttlSeconds = 300)
    public String findProduct(String id) {
        return db.get(id);
    }
    
    @RateLimit(maxRequests = 10, windowSeconds = 60)
    @Loggable("Search Products")
    public List<String> searchProducts(String query) {
        return db.values().stream()
            .filter(name -> name.toLowerCase().contains(query.toLowerCase()))
            .toList();
    }
    
    @RequiresPermission({"PRODUCT_WRITE"})
    @Loggable("Create Product")
    public String createProduct(String id, String name) {
        db.put(id, name);
        return id;
    }
    
    @RequiresPermission({"PRODUCT_WRITE", "ADMIN"})
    @Loggable("Delete Product")
    public boolean deleteProduct(String id) {
        return db.remove(id) != null;
    }
    
    private int retryCount = 0;
    
    @Retry(maxAttempts = 3, delayMs = 500, retryOn = {RuntimeException.class})
    @Loggable("Unreliable Operation")
    public String unreliableOperation() {
        retryCount++;
        if (retryCount < 3) {
            throw new RuntimeException("Temporary failure (attempt " + retryCount + ")");
        }
        return "Success after " + retryCount + " attempts";
    }
}

// Main demo
public class AopDemo {
    public static void main(String[] args) {
        var context = new org.springframework.context.annotation
            .AnnotationConfigApplicationContext(AopConfig.class);
        
        ProductService2 service = context.getBean(ProductService2.class);
        SecurityAspect security = context.getBean(SecurityAspect.class);
        PerformanceMonitoringAspect monitor = context.getBean(PerformanceMonitoringAspect.class);
        
        // Test caching
        System.out.println("=== Cache Test ===");
        service.findProduct("P001");  // CACHE SET
        service.findProduct("P001");  // CACHE HIT
        service.findProduct("P001");  // CACHE HIT
        
        // Test security
        System.out.println("\n=== Security Test ===");
        security.setPermissions("PRODUCT_READ");
        
        try {
            service.createProduct("P003", "Keyboard"); // Should fail - no PRODUCT_WRITE
        } catch (AccessDeniedException e) {
            System.out.println("Access denied (expected): " + e.getMessage());
        }
        
        security.setPermissions("PRODUCT_READ", "PRODUCT_WRITE");
        service.createProduct("P003", "Keyboard"); // Should succeed
        
        // Test retry
        System.out.println("\n=== Retry Test ===");
        try {
            String result = service.unreliableOperation();
            System.out.println("Result: " + result);
        } catch (Exception e) {
            System.out.println("Failed: " + e.getMessage());
        }
        
        // Print performance report
        monitor.printReport();
        
        security.clearPermissions();
        context.close();
    }
}

@org.springframework.context.annotation.Configuration
@org.springframework.context.annotation.ComponentScan("com.example.aop")
@org.springframework.context.annotation.EnableAspectJAutoProxy
class AopConfig {}
```

---

## 9. AOP Ordering

```java
import org.aspectj.lang.annotation.*;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

// Control order of multiple aspects with @Order (lower number = higher priority)

@Aspect
@Component
@Order(1)  // Runs outermost (first before, last after)
public class SecurityAspect2 {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void checkSecurity() {
        System.out.println("[Order 1] Security check");
    }
    
    @After("execution(* com.example.service.*.*(..))")
    public void afterSecurity() {
        System.out.println("[Order 1] After security");
    }
}

@Aspect
@Component
@Order(2)  // Runs middle
public class TransactionAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void beginTransaction() {
        System.out.println("[Order 2] Begin transaction");
    }
    
    @After("execution(* com.example.service.*.*(..))")
    public void endTransaction() {
        System.out.println("[Order 2] End transaction");
    }
}

@Aspect
@Component
@Order(3)  // Runs innermost (last before, first after)
public class LoggingAspect2 {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() {
        System.out.println("[Order 3] Logging before");
    }
    
    @After("execution(* com.example.service.*.*(..))")
    public void logAfter() {
        System.out.println("[Order 3] Logging after");
    }
}

/*
Execution order for method call:
[Order 1] Security check        → before (order 1)
[Order 2] Begin transaction     → before (order 2)
[Order 3] Logging before        → before (order 3)
          ... METHOD EXECUTES ...
[Order 3] Logging after         → after (order 3)
[Order 2] End transaction       → after (order 2)
[Order 1] After security        → after (order 1)
*/
```

---

## 10. Complete Real-world Example: Audit Trail System

```java
import org.aspectj.lang.*;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import java.lang.annotation.*;
import java.time.*;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

// === Audit Annotation ===
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@interface Audited {
    String action() default "";
    String entity() default "";
}

// === Audit Record ===
record AuditRecord(
    String id,
    Instant timestamp,
    String user,
    String action,
    String entity,
    String methodName,
    String[] args,
    String result,
    boolean success,
    String errorMessage
) {}

// === Audit Repository ===
@org.springframework.stereotype.Repository
class AuditRepository {
    private final List<AuditRecord> records = new CopyOnWriteArrayList<>();
    
    void save(AuditRecord record) {
        records.add(record);
    }
    
    List<AuditRecord> findAll() { return List.copyOf(records); }
    
    List<AuditRecord> findByUser(String user) {
        return records.stream()
            .filter(r -> r.user().equals(user))
            .toList();
    }
    
    List<AuditRecord> findByEntity(String entity) {
        return records.stream()
            .filter(r -> r.entity().equals(entity))
            .toList();
    }
}

// === Simulated Security Context ===
class SecurityContext {
    private static final ThreadLocal<String> currentUser = ThreadLocal.withInitial(() -> "anonymous");
    
    static String getCurrentUser() { return currentUser.get(); }
    static void setCurrentUser(String user) { currentUser.set(user); }
    static void clear() { currentUser.remove(); }
}

// === Audit Aspect ===
@Aspect
@org.springframework.stereotype.Component
class AuditAspect {
    
    private final AuditRepository auditRepository;
    
    AuditAspect(AuditRepository auditRepository) {
        this.auditRepository = auditRepository;
    }
    
    @Around("@annotation(audited)")
    public Object auditMethod(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        
        String action = audited.action().isEmpty() 
            ? signature.getName() : audited.action();
        String entity = audited.entity().isEmpty() 
            ? signature.getDeclaringType().getSimpleName() : audited.entity();
        
        String[] args = Arrays.stream(pjp.getArgs())
            .map(arg -> arg != null ? arg.toString() : "null")
            .toArray(String[]::new);
        
        String recordId = UUID.randomUUID().toString();
        String user = SecurityContext.getCurrentUser();
        Instant timestamp = Instant.now();
        
        try {
            Object result = pjp.proceed();
            
            auditRepository.save(new AuditRecord(
                recordId, timestamp, user, action, entity,
                signature.getName(), args, 
                result != null ? result.toString() : "void",
                true, null
            ));
            
            return result;
            
        } catch (Throwable t) {
            auditRepository.save(new AuditRecord(
                recordId, timestamp, user, action, entity,
                signature.getName(), args, null,
                false, t.getMessage()
            ));
            throw t;
        }
    }
}

// === Services using @Audited ===
@org.springframework.stereotype.Service
class BankAccountService {
    
    private final Map<String, Double> accounts = new HashMap<>(Map.of(
        "ACC001", 1000.0,
        "ACC002", 500.0
    ));
    
    @Audited(action = "DEPOSIT", entity = "BankAccount")
    public double deposit(String accountId, double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        accounts.merge(accountId, amount, Double::sum);
        return accounts.get(accountId);
    }
    
    @Audited(action = "WITHDRAW", entity = "BankAccount")
    public double withdraw(String accountId, double amount) {
        double balance = accounts.getOrDefault(accountId, 0.0);
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        accounts.put(accountId, balance - amount);
        return accounts.get(accountId);
    }
    
    @Audited(action = "TRANSFER", entity = "BankAccount")
    public void transfer(String fromId, String toId, double amount) {
        withdraw(fromId, amount);
        deposit(toId, amount);
    }
    
    public double getBalance(String accountId) {
        return accounts.getOrDefault(accountId, 0.0);
    }
}

// === Configuration ===
@org.springframework.context.annotation.Configuration
@org.springframework.context.annotation.ComponentScan
@org.springframework.context.annotation.EnableAspectJAutoProxy
class AuditConfig {}

// === Demo ===
public class AuditDemo {
    public static void main(String[] args) {
        var context = new org.springframework.context.annotation
            .AnnotationConfigApplicationContext(AuditConfig.class);
        
        BankAccountService bankService = context.getBean(BankAccountService.class);
        AuditRepository auditRepo = context.getBean(AuditRepository.class);
        
        // Simulate user Alice
        SecurityContext.setCurrentUser("alice@example.com");
        
        bankService.deposit("ACC001", 500.0);
        bankService.withdraw("ACC001", 200.0);
        
        // Simulate user Bob
        SecurityContext.setCurrentUser("bob@example.com");
        bankService.transfer("ACC002", "ACC001", 100.0);
        
        // Simulate failed operation
        try {
            SecurityContext.setCurrentUser("charlie@example.com");
            bankService.withdraw("ACC002", 1000.0); // Insufficient funds
        } catch (Exception e) {
            System.out.println("Expected error: " + e.getMessage());
        }
        
        SecurityContext.clear();
        
        // Print audit trail
        System.out.println("\n=== AUDIT TRAIL ===");
        auditRepo.findAll().forEach(record -> {
            System.out.printf("[%s] User: %-25s | Action: %-15s | Entity: %-15s | Success: %s%s%n",
                record.timestamp().toString().substring(0, 19),
                record.user(),
                record.action(),
                record.entity(),
                record.success(),
                record.success() ? "" : " | Error: " + record.errorMessage()
            );
        });
        
        System.out.println("\n=== Alice's Activity ===");
        auditRepo.findByUser("alice@example.com").forEach(r -> 
            System.out.println("  " + r.action() + " -> " + 
                (r.success() ? "OK" : "FAILED")));
        
        System.out.println("\nFinal balances:");
        System.out.println("ACC001: " + bankService.getBalance("ACC001"));
        System.out.println("ACC002: " + bankService.getBalance("ACC002"));
        
        context.close();
    }
}
```

---

## สรุป Part 022

| Advice Type | When Runs |
|------------|-----------|
| @Before | ก่อน method เรียก |
| @AfterReturning | หลัง method return ปกติ |
| @AfterThrowing | หลัง method throw exception |
| @After | เสมอ (finally) |
| @Around | ห่อ method ทั้งหมด (มีอำนาจสูงสุด) |

### Best Practices
- ใช้ constructor injection สำหรับ aspect dependencies
- @Around สำหรับ use cases ที่ต้องการ modify args หรือ result
- @Before/@After สำหรับ simple pre/post processing
- ใช้ @Order เพื่อควบคุม aspect ordering
- Custom annotations ช่วยให้ code อ่านเข้าใจง่าย

---

**Part 023:** Spring MVC - Web Application Framework
- DispatcherServlet architecture
- @Controller vs @RestController
- Request mapping, path variables, query params
- Request/Response handling
- Form handling และ validation
- Exception handling (@ExceptionHandler, @ControllerAdvice)
