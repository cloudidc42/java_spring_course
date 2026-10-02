# Part 033: Microservices with Spring Boot

## เนื้อหาในส่วนนี้
- Microservices Architecture Concepts
- Service Discovery with Eureka
- API Gateway with Spring Cloud Gateway
- Inter-service Communication (RestTemplate, FeignClient, WebClient)
- Circuit Breaker with Resilience4j
- Distributed Configuration with Spring Cloud Config
- Distributed Tracing
- Event-Driven Communication with Kafka
- Complete Microservices Example

---

## 1. Microservices Architecture

```
Monolith vs Microservices:

MONOLITH:
┌─────────────────────────────────────┐
│  User | Order | Product | Payment   │
│  (All in one JVM)                   │
└─────────────────────────────────────┘
- Simple deployment
- Hard to scale individual parts
- Single point of failure

MICROSERVICES:
┌──────────┐  ┌──────────┐  ┌──────────┐
│   User   │  │  Order   │  │ Product  │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
     ↑              ↑              ↑
     └──────────────┴──────────────┘
              API Gateway
              
- Independent deployment
- Scale per service
- Technology heterogeneity
- Complex distributed system
```

### Spring Cloud Dependencies

```xml
<!-- Parent -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<properties>
    <spring-cloud.version>2023.0.0</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Eureka Client (for microservices) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    
    <!-- Eureka Server (for discovery service) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    
    <!-- Spring Cloud Gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    
    <!-- OpenFeign (declarative HTTP client) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    
    <!-- Resilience4j (Circuit Breaker) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>
    
    <!-- Config Server -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    
    <!-- Distributed Tracing -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
</dependencies>
```

---

## 2. Eureka Service Discovery

### Eureka Server

```java
// eureka-server/src/main/java/EurekaServerApplication.java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# eureka-server/src/main/resources/application.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false  # Don't register itself
    fetch-registry: false        # Don't fetch its own registry
  server:
    enable-self-preservation: false  # Dev only
    eviction-interval-timer-in-ms: 5000
```

### Eureka Client (Microservice Registration)

```java
// user-service/src/main/java/UserServiceApplication.java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient  // Register with Eureka
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# user-service/application.yml
server:
  port: 8081

spring:
  application:
    name: user-service  # Service name in Eureka

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.uuid}
    health-check-url-path: /actuator/health
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 10
```

---

## 3. OpenFeign - Declarative HTTP Client

```java
import org.springframework.cloud.openfeign.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// Enable Feign clients
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

// Feign client - declares HTTP interface
@FeignClient(name = "user-service",  // Eureka service name
             fallback = UserServiceFallback.class)
public interface UserServiceClient {
    
    @GetMapping("/api/users/{id}")
    UserDto getUserById(@PathVariable Long id);
    
    @GetMapping("/api/users")
    List<UserDto> getAllUsers();
    
    @PostMapping("/api/users")
    UserDto createUser(@RequestBody CreateUserRequest request);
    
    @GetMapping("/api/users/exists/{email}")
    boolean existsByEmail(@PathVariable String email);
}

// Fallback for when user-service is down
@Component
class UserServiceFallback implements UserServiceClient {
    
    @Override
    public UserDto getUserById(Long id) {
        return new UserDto(id, "Unknown User", "unknown@example.com");
    }
    
    @Override
    public List<UserDto> getAllUsers() {
        return List.of();
    }
    
    @Override
    public UserDto createUser(CreateUserRequest request) {
        throw new RuntimeException("User service unavailable");
    }
    
    @Override
    public boolean existsByEmail(String email) {
        return false;
    }
}

// Product service client
@FeignClient(name = "product-service", url = "${product-service.url:}")
public interface ProductServiceClient {
    
    @GetMapping("/api/products/{id}")
    ProductDto getProduct(@PathVariable String id);
    
    @PutMapping("/api/products/{id}/stock")
    void updateStock(@PathVariable String id, @RequestBody StockUpdateRequest request);
}

record UserDto(Long id, String name, String email) {}
record ProductDto(String id, String name, double price, int stock) {}
record CreateUserRequest(String name, String email) {}
record StockUpdateRequest(int quantity, String operation) {}
```

---

## 4. Circuit Breaker with Resilience4j

```java
import io.github.resilience4j.circuitbreaker.*;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import java.util.concurrent.CompletableFuture;

@Service
public class ResilientOrderService {
    
    private final UserServiceClient userClient;
    private final ProductServiceClient productClient;
    
    ResilientOrderService(UserServiceClient userClient, ProductServiceClient productClient) {
        this.userClient = userClient;
        this.productClient = productClient;
    }
    
    // Circuit Breaker - automatically opens when error rate too high
    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    public UserDto getUser(Long userId) {
        return userClient.getUserById(userId);
    }
    
    private UserDto getUserFallback(Long userId, Throwable t) {
        System.err.println("Circuit open, using fallback for user " + userId + ": " + t.getMessage());
        return new UserDto(userId, "Unknown", "unknown@example.com");
    }
    
    // Retry - retry on failure
    @Retry(name = "productService", fallbackMethod = "getProductFallback")
    @CircuitBreaker(name = "productService")
    public ProductDto getProduct(String productId) {
        return productClient.getProduct(productId);
    }
    
    private ProductDto getProductFallback(String productId, Throwable t) {
        System.err.println("All retries exhausted for product " + productId);
        throw new RuntimeException("Product service unavailable: " + productId);
    }
    
    // Bulkhead - limit concurrent calls
    @Bulkhead(name = "inventoryService", type = Bulkhead.Type.SEMAPHORE)
    public String checkInventory(String productId) {
        // Max concurrent calls limited by bulkhead config
        return "In Stock";
    }
    
    // Time Limiter - timeout
    @TimeLimiter(name = "externalService")
    public CompletableFuture<String> callExternalService() {
        return CompletableFuture.supplyAsync(() -> {
            // Will fail if takes longer than configured timeout
            return "External service response";
        });
    }
    
    // Combine Circuit Breaker + Retry + Time Limiter
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<String> processPayment(double amount) {
        return CompletableFuture.supplyAsync(() -> {
            // Call payment service
            return "PAYMENT_SUCCESS_" + System.currentTimeMillis();
        });
    }
    
    private CompletableFuture<String> paymentFallback(double amount, Throwable t) {
        return CompletableFuture.completedFuture("PAYMENT_QUEUED_FOR_RETRY");
    }
}
```

```yaml
# Resilience4j configuration in application.yml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        slidingWindowSize: 10                # Look at last 10 calls
        minimumNumberOfCalls: 5              # Minimum calls before CB logic
        failureRateThreshold: 50             # Open if >50% fail
        slowCallRateThreshold: 80            # Open if >80% are slow
        slowCallDurationThreshold: 2s        # What's "slow"
        waitDurationInOpenState: 10s         # Wait before trying again
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
      productService:
        failureRateThreshold: 60
        waitDurationInOpenState: 20s
      paymentService:
        failureRateThreshold: 30             # Stricter for payments
        waitDurationInOpenState: 30s
  
  retry:
    instances:
      productService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - org.springframework.web.client.RestClientException
          - feign.FeignException
      paymentService:
        maxAttempts: 2
        waitDuration: 1s
  
  bulkhead:
    instances:
      inventoryService:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms
  
  timelimiter:
    instances:
      externalService:
        timeoutDuration: 5s
      paymentService:
        timeoutDuration: 3s
```

---

## 5. Spring Cloud Gateway

```yaml
# gateway-service/application.yml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  
  cloud:
    gateway:
      routes:
        # Route to user-service
        - id: user-service
          uri: lb://user-service  # lb:// uses load balancer (Eureka)
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=0
            - AddRequestHeader=X-Gateway-Version, 2.0
            - AddResponseHeader=X-Response-Time, ${responseTime}
        
        # Route to order-service
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /internal/orders/${segment}
        
        # Route with rate limiting
        - id: product-service-limited
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
                key-resolver: "#{@userKeyResolver}"
        
        # Route with circuit breaker
        - id: payment-service
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - name: CircuitBreaker
              args:
                name: paymentCircuitBreaker
                fallbackUri: forward:/fallback/payment
      
      # Global filters
      default-filters:
        - name: Retry
          args:
            retries: 3
            methods: GET
            backoff:
              firstBackoff: 50ms
              maxBackoff: 500ms
        
        - AddRequestHeader=X-Request-Id, ${java.util.UUID.randomUUID()}

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```java
// gateway-service/GatewayApplication.java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.context.annotation.*;
import org.springframework.cloud.gateway.filter.*;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@SpringBootApplication
@EnableDiscoveryClient
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}

// Custom Global Filter
@Component
class AuthenticationFilter implements GlobalFilter, org.springframework.core.Ordered {
    
    private static final List<String> OPEN_ENDPOINTS = List.of(
        "/api/auth/login", "/api/auth/register", "/actuator/health"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().toString();
        
        if (OPEN_ENDPOINTS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);  // Skip auth
        }
        
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        
        if (token == null || !token.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // Validate token and add user info to header
        String userId = validateAndExtractUserId(token);
        ServerWebExchange modifiedExchange = exchange.mutate()
            .request(r -> r.header("X-User-Id", userId))
            .build();
        
        return chain.filter(modifiedExchange);
    }
    
    @Override
    public int getOrder() { return -100; }  // High priority
    
    private String validateAndExtractUserId(String token) {
        // Validate JWT and extract user ID
        return "user-123";  // Simplified
    }
}

// Logging Filter
@Component
class LoggingFilter implements GlobalFilter, org.springframework.core.Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                long duration = System.currentTimeMillis() - startTime;
                System.out.printf("[GATEWAY] %s %s → %d (%dms)%n",
                    exchange.getRequest().getMethod(),
                    exchange.getRequest().getPath(),
                    exchange.getResponse().getStatusCode() != null 
                        ? exchange.getResponse().getStatusCode().value() : 0,
                    duration);
            }));
    }
    
    @Override
    public int getOrder() { return -50; }
}

// Fallback Controller
@RestController
class FallbackController {
    
    @GetMapping("/fallback/payment")
    public Map<String, String> paymentFallback() {
        return Map.of(
            "status", "SERVICE_UNAVAILABLE",
            "message", "Payment service is currently unavailable",
            "timestamp", java.time.Instant.now().toString()
        );
    }
}

// Rate limiter key resolver
@Configuration
class RateLimiterConfig {
    
    @Bean("userKeyResolver")
    public reactor.core.publisher.Mono<String> userKeyResolver() {
        return null; // Simplified - normally would extract user from JWT
    }
    
    // Real implementation:
    @Bean("ipKeyResolver")
    org.springframework.cloud.gateway.filter.ratelimit.KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }
}
```

---

## 6. Distributed Configuration - Spring Cloud Config

### Config Server

```java
// config-server/ConfigServerApplication.java
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# config-server/application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: "{application}"
          clone-on-start: true
        # Or file system (for local dev):
        # native:
        #   search-locations: classpath:/config

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### Config Client

```yaml
# microservice/bootstrap.yml (loads before application.yml)
spring:
  application:
    name: order-service
  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true  # Fail if config server unavailable
      retry:
        max-attempts: 6
        initial-interval: 1000
      import: optional:configserver:

# Config files in Git repo:
# order-service.yml         - default config
# order-service-dev.yml     - dev profile
# order-service-prod.yml    - prod profile
```

```java
// Refresh config at runtime without restart
@RestController
@RefreshScope  // Refresh beans when /actuator/refresh called
public class ConfigController {
    
    @Value("${feature.new-checkout:false}")
    private boolean newCheckoutEnabled;
    
    @Value("${shipping.cost.standard:5.99}")
    private double standardShipping;
    
    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of(
            "newCheckoutEnabled", newCheckoutEnabled,
            "standardShipping", standardShipping
        );
    }
}
// POST /actuator/refresh → refreshes all @RefreshScope beans
```

---

## 7. Distributed Tracing with Micrometer/Zipkin

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 0.1 in prod
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

spring:
  application:
    name: order-service
```

```java
// Tracing is automatic with micrometer-tracing-bridge-brave
// Every HTTP request gets a traceId and spanId

// Manual span creation
import io.micrometer.tracing.*;

@Service
public class TracedOrderService {
    
    private final Tracer tracer;
    private final UserServiceClient userClient;
    
    TracedOrderService(Tracer tracer, UserServiceClient userClient) {
        this.tracer = tracer;
        this.userClient = userClient;
    }
    
    public String processOrder(Long userId, List<String> productIds) {
        // Create a custom span
        Span orderSpan = tracer.nextSpan().name("process-order").start();
        
        try (Tracer.SpanInScope ws = tracer.withSpan(orderSpan)) {
            orderSpan.tag("userId", String.valueOf(userId));
            orderSpan.tag("productCount", String.valueOf(productIds.size()));
            
            // User lookup (automatic tracing via Feign)
            UserDto user = userClient.getUserById(userId);
            orderSpan.tag("userEmail", user.email());
            
            // Process products
            Span productSpan = tracer.nextSpan().name("process-products").start();
            try (Tracer.SpanInScope ps = tracer.withSpan(productSpan)) {
                productIds.forEach(id -> {
                    productSpan.tag("productId", id);
                    // process each product
                });
            } finally {
                productSpan.end();
            }
            
            return "ORDER-" + System.currentTimeMillis();
            
        } catch (Exception e) {
            orderSpan.error(e);
            throw e;
        } finally {
            orderSpan.end();
        }
    }
}

/*
Zipkin UI: http://localhost:9411
Shows distributed traces across all microservices

Trace example:
order-service [100ms]
  └─ user-service [20ms]
  └─ product-service [30ms]
  └─ payment-service [40ms]
*/
```

---

## 8. Event-Driven with Kafka

```xml
<!-- Kafka dependency -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-service-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.*"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

```java
import org.springframework.kafka.core.*;
import org.springframework.kafka.annotation.*;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.stereotype.Service;
import org.springframework.context.annotation.Configuration;
import org.apache.kafka.clients.admin.NewTopic;

// Topics
@Configuration
class KafkaTopicConfig {
    
    @Bean
    public NewTopic orderCreatedTopic() {
        return new NewTopic("order-created", 3, (short) 1);  // 3 partitions, 1 replica
    }
    
    @Bean
    public NewTopic orderShippedTopic() {
        return new NewTopic("order-shipped", 3, (short) 1);
    }
    
    @Bean
    public NewTopic paymentProcessedTopic() {
        return new NewTopic("payment-processed", 3, (short) 1);
    }
}

// Events
record OrderCreatedEvent(String orderId, Long userId, List<String> productIds, double total) {}
record OrderShippedEvent(String orderId, String trackingNumber, String carrier) {}
record PaymentProcessedEvent(String orderId, String transactionId, String status) {}

// Producer Service
@Service
public class OrderEventPublisher {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    OrderEventPublisher(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    public void publishOrderCreated(OrderCreatedEvent event) {
        kafkaTemplate.send("order-created", event.orderId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    System.err.println("Failed to send order-created event: " + ex.getMessage());
                } else {
                    System.out.println("Published order-created: " + event.orderId() +
                        " to partition " + result.getRecordMetadata().partition());
                }
            });
    }
    
    public void publishOrderShipped(OrderShippedEvent event) {
        kafkaTemplate.send("order-shipped", event.orderId(), event);
    }
}

// Consumer Service (in notification-service)
@Service
public class NotificationEventConsumer {
    
    @KafkaListener(topics = "order-created", groupId = "notification-service")
    public void handleOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {
        
        System.out.printf("Received from topic=%s partition=%d offset=%d: orderId=%s%n",
            topic, partition, offset, event.orderId());
        
        // Send email notification
        sendOrderConfirmationEmail(event);
    }
    
    @KafkaListener(topics = "order-shipped", groupId = "notification-service")
    public void handleOrderShipped(OrderShippedEvent event) {
        System.out.println("Order shipped: " + event.orderId() + 
            " tracking: " + event.trackingNumber());
        sendShipmentNotification(event);
    }
    
    // Listen to multiple topics
    @KafkaListener(topics = {"order-created", "order-shipped"}, 
                   groupId = "audit-service")
    public void auditEvents(String rawMessage, 
                            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        System.out.println("Audit [" + topic + "]: " + rawMessage);
    }
    
    private void sendOrderConfirmationEmail(OrderCreatedEvent event) {
        System.out.println("📧 Sending order confirmation for: " + event.orderId());
    }
    
    private void sendShipmentNotification(OrderShippedEvent event) {
        System.out.println("📦 Sending shipment notification: " + event.trackingNumber());
    }
}

// Consumer with error handling
@Service
public class PaymentEventConsumer {
    
    @KafkaListener(topics = "payment-processed")
    public void handlePayment(PaymentProcessedEvent event) {
        if ("FAILED".equals(event.status())) {
            throw new RuntimeException("Payment failed, retrying...");
        }
        System.out.println("Payment processed: " + event.transactionId());
    }
}

// Dead Letter Queue for failed messages
@Configuration
class KafkaDeadLetterConfig {
    
    @Bean
    public org.springframework.kafka.listener.DeadLetterPublishingRecoverer deadLetterRecoverer(
            KafkaTemplate<String, Object> template) {
        return new org.springframework.kafka.listener.DeadLetterPublishingRecoverer(template);
    }
    
    @Bean
    public org.springframework.kafka.listener.DefaultErrorHandler errorHandler(
            org.springframework.kafka.listener.DeadLetterPublishingRecoverer recoverer) {
        var backoff = new org.springframework.util.backoff.FixedBackOff(1000L, 3L);
        return new org.springframework.kafka.listener.DefaultErrorHandler(recoverer, backoff);
    }
}
```

---

## 9. Complete Microservices Example

```
E-Commerce Microservices:

Port 8761: Eureka Server (Service Registry)
Port 8888: Config Server
Port 8080: API Gateway (Spring Cloud Gateway)
Port 8081: User Service
Port 8082: Product Service
Port 8083: Order Service
Port 8084: Payment Service
Port 8085: Notification Service

Kafka Topics:
- order-created
- payment-processed
- order-shipped

Flow:
1. Client → Gateway (auth filter)
2. Gateway → Order Service (create order)
3. Order Service → Product Service (check stock via Feign)
4. Order Service → User Service (get user via Feign)
5. Order Service → Kafka (publish order-created)
6. Payment Service ← Kafka (consume, process payment)
7. Payment Service → Kafka (publish payment-processed)
8. Order Service ← Kafka (update order status)
9. Notification Service ← Kafka (send emails)
```

```java
// Order Service - Orchestrates the flow
@RestController
@RequestMapping("/api/orders")
class OrderController2 {
    
    private final OrderOrchestrationService orchestrationService;
    
    OrderController2(OrderOrchestrationService orchestrationService) {
        this.orchestrationService = orchestrationService;
    }
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request,
                                                      @RequestHeader("X-User-Id") String userId) {
        OrderResponse order = orchestrationService.createOrder(Long.parseLong(userId), request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
    
    @GetMapping("/{orderId}")
    public OrderResponse getOrder(@PathVariable String orderId) {
        return orchestrationService.getOrder(orderId);
    }
}

@Service
class OrderOrchestrationService {
    
    private final UserServiceClient userClient;
    private final ProductServiceClient productClient;
    private final OrderEventPublisher eventPublisher;
    private final Map<String, OrderResponse> orders = new ConcurrentHashMap<>();
    
    OrderOrchestrationService(UserServiceClient userClient, 
                               ProductServiceClient productClient,
                               OrderEventPublisher eventPublisher) {
        this.userClient = userClient;
        this.productClient = productClient;
        this.eventPublisher = eventPublisher;
    }
    
    @CircuitBreaker(name = "createOrder", fallbackMethod = "createOrderFallback")
    @Transactional
    public OrderResponse createOrder(Long userId, CreateOrderRequest request) {
        // 1. Validate user
        UserDto user = userClient.getUserById(userId);
        
        // 2. Validate products and calculate total
        double total = 0;
        List<String> productIds = new ArrayList<>();
        for (String productId : request.productIds()) {
            ProductDto product = productClient.getProduct(productId);
            total += product.price();
            productIds.add(productId);
        }
        
        // 3. Create order
        String orderId = "ORD-" + System.currentTimeMillis();
        OrderResponse order = new OrderResponse(orderId, userId, productIds, total, "PENDING");
        orders.put(orderId, order);
        
        // 4. Publish event (triggers payment async)
        eventPublisher.publishOrderCreated(
            new OrderCreatedEvent(orderId, userId, productIds, total));
        
        return order;
    }
    
    private OrderResponse createOrderFallback(Long userId, CreateOrderRequest request, Throwable t) {
        throw new RuntimeException("Order creation failed: " + t.getMessage());
    }
    
    public OrderResponse getOrder(String orderId) {
        return Optional.ofNullable(orders.get(orderId))
            .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));
    }
    
    @KafkaListener(topics = "payment-processed")
    public void handlePaymentResult(PaymentProcessedEvent event) {
        String orderId = event.orderId();
        OrderResponse current = orders.get(orderId);
        if (current != null) {
            String newStatus = "COMPLETED".equals(event.status()) ? "CONFIRMED" : "PAYMENT_FAILED";
            orders.put(orderId, new OrderResponse(
                current.orderId(), current.userId(), current.productIds(), 
                current.total(), newStatus));
            System.out.println("Order " + orderId + " updated to: " + newStatus);
        }
    }
}

record CreateOrderRequest(List<String> productIds, String shippingAddress) {}
record OrderResponse(String orderId, Long userId, List<String> productIds, double total, String status) {}

import org.springframework.http.HttpStatus;
import org.springframework.transaction.annotation.Transactional;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
```

---

## สรุป Part 033

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Service Registry | Eureka | Service discovery |
| API Gateway | Spring Cloud Gateway | Single entry, routing, auth |
| HTTP Client | OpenFeign | Declarative REST client |
| Circuit Breaker | Resilience4j | Fault tolerance |
| Config Management | Spring Cloud Config | Centralized configuration |
| Distributed Tracing | Micrometer + Zipkin | Observe request flow |
| Messaging | Apache Kafka | Async event-driven |

---

**Part 034:** Docker & Containerization สำหรับ Spring Boot
- Dockerfile สำหรับ Spring Boot
- Docker Compose
- Multi-stage builds
- Docker best practices
- Container networking
- Volume management
