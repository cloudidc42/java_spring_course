# Part 024: Spring Boot Auto-configuration & Actuator

## เนื้อหาในส่วนนี้
- Spring Boot Auto-configuration internals
- application.properties / application.yml
- Profiles in Spring Boot
- Spring Boot Actuator
- Health Checks, Metrics, Info
- Custom Actuator Endpoints
- Production-ready Configuration
- Monitoring Integration

---

## 1. Spring Boot Auto-configuration

### How It Works

```java
// @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

/*
@EnableAutoConfiguration triggers Spring Boot to:
1. Scan all META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
2. Check @ConditionalOn* conditions for each auto-configuration class
3. Load configurations that meet their conditions

Example: When spring-boot-starter-web is on classpath:
- DispatcherServletAutoConfiguration → configures DispatcherServlet
- TomcatAutoConfiguration → starts embedded Tomcat
- JacksonAutoConfiguration → configures JSON serialization
- ErrorMvcAutoConfiguration → sets up error handling
*/

// See what's auto-configured
// Run: mvn spring-boot:run --debug
// Or set: logging.level.org.springframework.boot.autoconfigure=DEBUG
```

### Custom Auto-configuration

```java
import org.springframework.boot.autoconfigure.*;
import org.springframework.boot.autoconfigure.condition.*;
import org.springframework.context.annotation.*;

// Define auto-configuration class
@AutoConfiguration
@ConditionalOnClass(name = "com.example.SomeLibrary")
@ConditionalOnMissingBean(MyService.class)
@EnableConfigurationProperties(MyServiceProperties.class)
public class MyServiceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyServiceProperties properties) {
        return new MyServiceImpl(properties.getUrl(), properties.getTimeout());
    }
}

// Properties class
@ConfigurationProperties(prefix = "myservice")
public class MyServiceProperties {
    private String url = "http://localhost:8080";
    private int timeout = 5000;
    
    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }
    public int getTimeout() { return timeout; }
    public void setTimeout(int timeout) { this.timeout = timeout; }
}

// Register: src/main/resources/META-INF/spring/
// org.springframework.boot.autoconfigure.AutoConfiguration.imports
// com.example.MyServiceAutoConfiguration

interface MyService { void doSomething(); }
class MyServiceImpl implements MyService {
    MyServiceImpl(String url, int timeout) {}
    public void doSomething() {}
}
```

---

## 2. application.properties / application.yml

### Structure and Features

```yaml
# application.yml - Complete example

# Server configuration
server:
  port: 8080
  servlet:
    context-path: /api
  compression:
    enabled: true
    min-response-size: 1024
  error:
    include-message: always
    include-binding-errors: always
    include-stacktrace: never  # never in production!
  shutdown: graceful  # Wait for active requests before shutdown

# Application info
spring:
  application:
    name: my-application
  
  # Jackson JSON configuration  
  jackson:
    serialization:
      indent-output: true
      write-dates-as-timestamps: false
    deserialization:
      fail-on-unknown-properties: false
    default-property-inclusion: non_null
    time-zone: UTC
  
  # Database
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME:myuser}       # env var with default
    password: ${DB_PASSWORD:mypassword}
    driver-class-name: org.postgresql.Driver
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 300000
      connection-timeout: 20000
      pool-name: MyAppPool
  
  # JPA
  jpa:
    hibernate:
      ddl-auto: validate  # validate | update | create | create-drop | none
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_schema: public
  
  # Cache
  cache:
    type: redis  # simple | redis | caffeine | ehcache
    redis:
      time-to-live: 600000  # 10 minutes
  
  # Redis
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
  
  # Mail
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
  
  # Security
  security:
    user:
      name: admin           # Default user for basic auth
      password: secret
  
  # Lifecycle
  lifecycle:
    timeout-per-shutdown-phase: 30s

# Logging
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG
  file:
    name: logs/application.log
  pattern:
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
    console: "%clr(%d{HH:mm:ss.SSS}){faint} %clr(%-5level) %clr(%logger{36}){cyan} - %msg%n"

# Custom application properties
app:
  name: My Application
  version: 1.0.0
  features:
    dark-mode: true
    beta-features: false
  limits:
    max-file-size: 10MB
    max-requests-per-minute: 100
  cors:
    allowed-origins:
      - http://localhost:3000
      - https://myapp.com
```

### Externalized Configuration Sources (Priority order)

```
1. Command line arguments:
   --server.port=9090

2. OS environment variables:
   SERVER_PORT=9090

3. application-{profile}.properties/yml
   
4. application.properties/yml

5. @PropertySource annotations

6. Default values in code

Command line: java -jar app.jar --server.port=9090 --spring.profiles.active=prod
```

---

## 3. Spring Profiles in Boot

```yaml
# application.yml - Default/shared config
spring:
  application:
    name: my-app
  jpa:
    open-in-view: false

logging:
  level:
    root: INFO

---
# application-development.yml
spring:
  config:
    activate:
      on-profile: development
  
  datasource:
    url: jdbc:h2:mem:devdb
    username: sa
    password:
  
  h2:
    console:
      enabled: true
      path: /h2-console
  
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop

logging:
  level:
    com.example: DEBUG

---  
# application-test.yml
spring:
  config:
    activate:
      on-profile: test
  
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    username: sa
    password:
  
  jpa:
    hibernate:
      ddl-auto: create-drop

---
# application-production.yml
spring:
  config:
    activate:
      on-profile: production
  
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 50
  
  jpa:
    hibernate:
      ddl-auto: validate

server:
  error:
    include-stacktrace: never
    include-message: never

logging:
  level:
    root: WARN
    com.example: INFO
```

```java
// Profile-specific beans
@Configuration
public class DataInitializerConfig {
    
    @Bean
    @Profile("development")
    CommandLineRunner devDataInitializer(UserRepository userRepository) {
        return args -> {
            System.out.println("Loading development test data...");
            // Load test data
        };
    }
    
    @Bean
    @Profile("production")
    CommandLineRunner prodHealthCheck() {
        return args -> {
            System.out.println("Production health checks running...");
        };
    }
}

// Activate profiles
// Method 1: application.properties
// spring.profiles.active=development

// Method 2: JVM args
// java -jar app.jar -Dspring.profiles.active=production

// Method 3: Environment variable
// SPRING_PROFILES_ACTIVE=production java -jar app.jar

// Method 4: In tests
@SpringBootTest
@ActiveProfiles("test")
class MyIntegrationTest {}
```

---

## 4. Spring Boot Actuator

```xml
<!-- Add to pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml - Actuator configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,beans,loggers,httptrace,threaddump,heapdump
        # include: "*"  # Expose all (careful in production!)
        exclude: shutdown  # Never expose shutdown in production
      base-path: /actuator
  
  endpoint:
    health:
      show-details: always  # always | when-authorized | never
      show-components: always
    env:
      show-values: always  # Show property values
    loggers:
      enabled: true
  
  health:
    defaults:
      enabled: true
    diskspace:
      enabled: true
      threshold: 524288000  # 500MB
  
  info:
    env:
      enabled: true
    git:
      enabled: true
      mode: full  # simple | full

# Info endpoint data
info:
  app:
    name: ${spring.application.name}
    version: @project.version@
    description: My Application
  build:
    java-version: @java.version@
    spring-boot-version: @project.parent.version@
```

### Built-in Endpoints

```
GET /actuator              - List all enabled endpoints
GET /actuator/health       - Application health status
GET /actuator/info         - Application information
GET /actuator/metrics      - Available metrics
GET /actuator/metrics/{name} - Specific metric
GET /actuator/env          - Environment properties
GET /actuator/beans        - All Spring beans
GET /actuator/loggers      - Logger configuration
POST /actuator/loggers/{name} - Change log level at runtime!
GET /actuator/httptrace    - HTTP request traces
GET /actuator/threaddump   - JVM thread dump
GET /actuator/heapdump     - JVM heap dump
GET /actuator/mappings     - Request mappings
GET /actuator/configprops  - Configuration properties
POST /actuator/shutdown    - Shutdown application (DISABLED by default!)
```

---

## 5. Health Indicators

```java
import org.springframework.boot.actuate.health.*;
import org.springframework.stereotype.Component;
import java.sql.Connection;
import java.util.*;

// === Custom Health Indicator ===
@Component("externalService")
public class ExternalServiceHealthIndicator implements HealthIndicator {
    
    private final String serviceUrl;
    
    ExternalServiceHealthIndicator() {
        this.serviceUrl = "https://api.external-service.com/health";
    }
    
    @Override
    public Health health() {
        try {
            // Simulate health check
            boolean isUp = checkExternalService();
            
            if (isUp) {
                return Health.up()
                    .withDetail("service", "External API")
                    .withDetail("url", serviceUrl)
                    .withDetail("responseTime", "45ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "External API")
                    .withDetail("error", "Service returned 503")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "External API")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
    
    private boolean checkExternalService() {
        // In real code: make HTTP request and check response
        return true;
    }
}

// === Composite Health Indicator ===
@Component
public class DatabaseHealthContributor implements CompositeHealthContributor {
    
    private final Map<String, HealthContributor> contributors;
    
    DatabaseHealthContributor() {
        this.contributors = new LinkedHashMap<>();
        contributors.put("primaryDb", () -> checkDatabase("primary"));
        contributors.put("cacheDb", () -> checkDatabase("cache"));
    }
    
    @Override
    public HealthContributor getContributor(String name) {
        return contributors.get(name);
    }
    
    @Override
    public Iterator<NamedContributor<HealthContributor>> iterator() {
        return contributors.entrySet().stream()
            .map(entry -> NamedContributor.of(entry.getKey(), entry.getValue()))
            .iterator();
    }
    
    private Health checkDatabase(String name) {
        try {
            // Check database connection
            return Health.up()
                .withDetail("database", name)
                .withDetail("connections", "5/20")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("database", name)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Health check response example:
/*
GET /actuator/health
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": { "database": "PostgreSQL", "validationQuery": "isValid()" }
    },
    "diskSpace": {
      "status": "UP",
      "details": { "total": 100GB, "free": 60GB, "threshold": 500MB }
    },
    "externalService": {
      "status": "UP",
      "details": { "service": "External API", "responseTime": "45ms" }
    },
    "redis": {
      "status": "UP",
      "details": { "version": "7.0.11" }
    }
  }
}
*/
```

---

## 6. Metrics with Micrometer

```java
import io.micrometer.core.instrument.*;
import io.micrometer.core.annotation.*;
import org.springframework.stereotype.Service;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

@Service
public class OrderMetricsService {
    
    private final MeterRegistry registry;
    private final Counter orderCounter;
    private final Counter failedOrderCounter;
    private final Timer orderProcessingTimer;
    private final AtomicInteger activeOrders = new AtomicInteger(0);
    
    public OrderMetricsService(MeterRegistry registry) {
        this.registry = registry;
        
        // Counter - counts events
        this.orderCounter = Counter.builder("orders.total")
            .description("Total number of orders")
            .tag("type", "all")
            .register(registry);
        
        this.failedOrderCounter = Counter.builder("orders.failed")
            .description("Number of failed orders")
            .register(registry);
        
        // Timer - measures duration
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        // Gauge - current state value
        Gauge.builder("orders.active", activeOrders, AtomicInteger::get)
            .description("Currently active orders")
            .register(registry);
    }
    
    public String processOrder(String orderId) {
        activeOrders.incrementAndGet();
        
        return orderProcessingTimer.record(() -> {
            try {
                // Process order...
                Thread.sleep((long)(Math.random() * 100));
                orderCounter.increment();
                return "Order " + orderId + " processed";
            } catch (Exception e) {
                failedOrderCounter.increment();
                throw new RuntimeException("Order processing failed", e);
            } finally {
                activeOrders.decrementAndGet();
            }
        });
    }
}

// === @Timed annotation (requires @EnableAspectJAutoProxy) ===
@Service
public class UserMetricsService {
    
    @Timed(value = "user.service.findUser", description = "Time to find a user",
           percentiles = {0.5, 0.95, 0.99})
    public String findUser(Long id) {
        // Automatically timed
        return "User " + id;
    }
    
    @Timed(value = "user.service.createUser", longTask = true)
    public String createUser(String name) {
        return "Created: " + name;
    }
}

// === Custom Metrics Endpoint ===
/*
GET /actuator/metrics/orders.total
{
  "name": "orders.total",
  "description": "Total number of orders",
  "measurements": [{"statistic": "COUNT", "value": 1234}],
  "availableTags": [{"tag": "type", "values": ["all"]}]
}

GET /actuator/metrics/orders.processing.time
{
  "name": "orders.processing.time",
  "measurements": [
    {"statistic": "COUNT", "value": 500},
    {"statistic": "TOTAL_TIME", "value": 12.5},
    {"statistic": "MAX", "value": 0.2},
    {"statistic": "PERCENTILE", "value": 0.05}
  ]
}
*/
```

---

## 7. Custom Actuator Endpoints

```java
import org.springframework.boot.actuate.endpoint.annotation.*;
import org.springframework.stereotype.Component;
import java.util.*;

// === Custom Actuator Endpoint ===
@Component
@Endpoint(id = "features")  // /actuator/features
public class FeaturesEndpoint {
    
    private final Map<String, Boolean> features = new HashMap<>(Map.of(
        "darkMode", true,
        "betaUI", false,
        "analytics", true
    ));
    
    @ReadOperation   // GET /actuator/features
    public Map<String, Boolean> getFeatures() {
        return Map.copyOf(features);
    }
    
    @ReadOperation   // GET /actuator/features/{name}
    public Map<String, Object> getFeature(@Selector String name) {
        Boolean enabled = features.get(name);
        if (enabled == null) {
            return Map.of("error", "Feature not found: " + name);
        }
        return Map.of("name", name, "enabled", enabled);
    }
    
    @WriteOperation  // POST /actuator/features/{name}
    public void toggleFeature(@Selector String name, Boolean enabled) {
        features.put(name, enabled);
    }
    
    @DeleteOperation  // DELETE /actuator/features/{name}
    public void deleteFeature(@Selector String name) {
        features.remove(name);
    }
}

// === Cache Stats Endpoint ===
@Component
@Endpoint(id = "cache-stats")
public class CacheStatsEndpoint {
    
    record CacheStats(String name, int size, long hits, long misses, double hitRate) {}
    
    private final Map<String, long[]> stats = new HashMap<>();  // [hits, misses]
    
    @ReadOperation
    public List<CacheStats> getCacheStats() {
        return List.of(
            new CacheStats("users", 250, 5000, 100, 98.0),
            new CacheStats("products", 1200, 15000, 800, 94.9)
        );
    }
    
    @WriteOperation
    public void clearCache(@Selector(match = Selector.Match.ALL_REMAINING) String[] cacheNames) {
        for (String name : cacheNames) {
            System.out.println("Clearing cache: " + name);
            stats.remove(name);
        }
    }
}

// === Application Info Contributor ===
import org.springframework.boot.actuate.info.*;

@Component
public class AppInfoContributor implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("app", Map.of(
            "name", "My Application",
            "environment", System.getProperty("spring.profiles.active", "default"),
            "startedAt", java.time.Instant.now().toString(),
            "javaVersion", System.getProperty("java.version"),
            "osName", System.getProperty("os.name")
        ));
    }
}

// GET /actuator/info response:
/*
{
  "app": {
    "name": "My Application",
    "version": "1.0.0",
    "environment": "production"
  },
  "git": {
    "branch": "main",
    "commit": {
      "id": "abc1234",
      "time": "2024-01-15T10:30:00Z"
    }
  },
  "build": {
    "java-version": "21",
    "spring-boot-version": "3.2.0"
  }
}
*/
```

---

## 8. Production-Ready Configuration

```yaml
# application-production.yml

server:
  port: 8080
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    connection-timeout: 20000
    max-connections: 8192
    accept-count: 100
  shutdown: graceful  # Complete active requests before shutdown
  
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # Wait up to 30s for active requests

# Actuator - minimal exposure in production
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
        exclude: env,beans,configprops,heapdump
  endpoint:
    health:
      show-details: when_authorized  # Only for authenticated users
    shutdown:
      enabled: false  # NEVER enable in production
  
  # Secure actuator endpoints
  server:
    port: 8081  # Run actuator on different port
    address: 127.0.0.1  # Only accessible from localhost

# Security
spring:
  security:
    user:
      name: actuator-user
      password: ${ACTUATOR_PASSWORD}
      roles: ACTUATOR

# JVM tuning hints (set via JAVA_OPTS)
# -Xmx2g -Xms512m
# -XX:+UseG1GC
# -XX:MaxGCPauseMillis=200
# -XX:+HeapDumpOnOutOfMemoryError
# -XX:HeapDumpPath=/app/logs/heap-dump.hprof
```

```java
// === Graceful Shutdown Hook ===
@Component
public class GracefulShutdownHandler implements 
        org.springframework.boot.web.server.GracefulShutdownCallback {
    
    @Override
    public void shutdownComplete(org.springframework.boot.web.server.GracefulShutdownResult result) {
        System.out.println("Server shutdown complete: " + result);
    }
}

// === Production Spring Boot Application Setup ===
@SpringBootApplication
public class ProductionApplication {
    
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(ProductionApplication.class);
        
        // Customize startup
        app.setBannerMode(org.springframework.boot.Banner.Mode.OFF);
        app.addListeners(new ApplicationStartupListener());
        
        ConfigurableApplicationContext context = app.run(args);
        
        // Register shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutdown hook triggered");
        }));
    }
    
    static class ApplicationStartupListener implements 
            org.springframework.context.ApplicationListener<
                org.springframework.boot.context.event.ApplicationReadyEvent> {
        
        @Override
        public void onApplicationEvent(
                org.springframework.boot.context.event.ApplicationReadyEvent event) {
            
            System.out.println("Application started successfully");
            System.out.println("Profile: " + 
                String.join(", ", event.getApplicationContext()
                    .getEnvironment().getActiveProfiles()));
        }
    }
}
```

---

## 9. CommandLineRunner and ApplicationRunner

```java
import org.springframework.boot.*;
import org.springframework.stereotype.Component;
import java.util.Arrays;

// Runs after ApplicationContext is fully started
@Component
@Order(1)  // Execution order
public class DatabaseInitRunner implements CommandLineRunner {
    
    @Override
    public void run(String... args) throws Exception {
        System.out.println("CommandLineRunner: args = " + Arrays.toString(args));
        System.out.println("Initializing database...");
        // Load initial data, run migrations, etc.
    }
}

@Component
@Order(2)
public class CacheWarmupRunner implements ApplicationRunner {
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        // ApplicationArguments gives more structured access
        System.out.println("Non-option args: " + args.getNonOptionArgs());
        System.out.println("Option names: " + args.getOptionNames());
        
        if (args.containsOption("warm-cache")) {
            System.out.println("Warming up caches...");
        }
    }
}

// Usage: java -jar app.jar --warm-cache arg1 arg2

// === Banner Customization ===
// src/main/resources/banner.txt
/*
  __  __         ___ ___
 |  \/  |_   _  / __| _ \  _ __ _ _ ___ _ ___
 | |\/| | | | | \__ \  _/ | '_ \ '_/ _ \ '_ \
 |_|  |_|\_, | |___/_|   | .__/_| \___/ .__/
         |__/             |_|          |_|
Version: ${spring.application.version}
Spring Boot: ${spring-boot.version}
*/
```

---

## 10. Spring Boot DevTools

```xml
<!-- pom.xml - DevTools for development -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

```yaml
# DevTools configuration
spring:
  devtools:
    restart:
      enabled: true
      exclude: static/**,templates/**,*.html
      additional-paths: src/main/java
      trigger-file: .trigger  # Restart only when this file changes
    livereload:
      enabled: true  # Browser auto-reload
      port: 35729

# DevTools is AUTOMATICALLY disabled in production (not loaded for packaged JARs)
```

---

## 11. Spring Boot Testing

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.test.web.servlet.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.*;

// === Full Integration Test ===
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class ApplicationIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void healthEndpointIsUp() {
        ResponseEntity<String> response = restTemplate.getForEntity("/actuator/health", String.class);
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
    
    @Test
    void createAndGetBook() {
        // Create
        Map<String, Object> book = new HashMap<>();
        book.put("title", "Test Book");
        book.put("author", "Test Author");
        book.put("isbn", "9781234567890");
        book.put("price", 29.99);
        book.put("category", "Technology");
        
        ResponseEntity<Map> created = restTemplate.postForEntity("/api/v1/books", book, Map.class);
        assertEquals(HttpStatus.CREATED, created.getStatusCode());
        
        String id = (String) created.getBody().get("id");
        assertNotNull(id);
        
        // Get
        ResponseEntity<Map> found = restTemplate.getForEntity("/api/v1/books/" + id, Map.class);
        assertEquals(HttpStatus.OK, found.getStatusCode());
        assertEquals("Test Book", found.getBody().get("title"));
    }
}

// === MockMvc Test (Faster - no real HTTP) ===
@WebMvcTest(BookController.class)  // Only loads web layer
class BookControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean  // Mock service layer
    private BookService bookService;
    
    @Autowired
    private com.fasterxml.jackson.databind.ObjectMapper objectMapper;
    
    @Test
    void getBookReturnsBook() throws Exception {
        Book book = new Book("1", "Clean Code", "Martin", "9780132350884", 45.99, "Tech", null, true);
        Mockito.when(bookService.findById("1")).thenReturn(book);
        
        mockMvc.perform(get("/api/v1/books/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.title").value("Clean Code"))
            .andExpect(jsonPath("$.author").value("Martin"))
            .andExpect(jsonPath("$.price").value(45.99));
    }
    
    @Test
    void createBookValidationFails() throws Exception {
        Map<String, Object> invalidBook = Map.of("title", ""); // Missing required fields
        
        mockMvc.perform(post("/api/v1/books")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidBook)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.error").exists());
    }
    
    @Test
    void deleteBookReturnsNoContent() throws Exception {
        Mockito.doNothing().when(bookService).delete("1");
        
        mockMvc.perform(delete("/api/v1/books/1"))
            .andExpect(status().isNoContent());
    }
    
    @Test
    void getBookNotFound() throws Exception {
        Mockito.when(bookService.findById("999"))
            .thenThrow(new BookNotFoundException("999"));
        
        mockMvc.perform(get("/api/v1/books/999"))
            .andExpect(status().isNotFound());
    }
}
```

---

## สรุป Part 024

| หัวข้อ | Key Points |
|--------|-----------|
| Auto-configuration | @ConditionalOn* conditions, AutoConfiguration.imports |
| application.yml | Structured config, environment variable interpolation |
| Profiles | Environment-specific config with spring.profiles.active |
| Actuator | /health, /info, /metrics, /loggers, /env |
| Health Indicators | HealthIndicator, CompositeHealthContributor |
| Micrometer | Counter, Timer, Gauge, @Timed |
| Custom Endpoints | @Endpoint with @ReadOperation/@WriteOperation |
| Graceful Shutdown | server.shutdown: graceful |
| DevTools | Hot reload, LiveReload |
| Testing | @SpringBootTest, @WebMvcTest, TestRestTemplate |

---

**Part 025:** Spring Data JPA - Database Integration
- JPA entities and relationships (OneToMany, ManyToMany, etc.)
- Spring Data Repository interfaces
- JPQL and native queries
- Pagination and sorting
- Transactions
- Auditing with @CreatedDate, @LastModifiedDate
