# Part 021: Spring Framework - IoC Container & Dependency Injection

## เนื้อหาในส่วนนี้
- Spring Framework Overview
- Inversion of Control (IoC) Concept
- ApplicationContext และ BeanFactory
- Bean Definition, Scope, Lifecycle
- Annotation-based Configuration
- Java-based Configuration
- Dependency Injection Patterns
- Profiles และ Conditional Beans
- Real-world Application Example

---

## 1. Spring Framework Overview

Spring Framework คือ comprehensive programming and configuration model สำหรับ Java enterprise applications

### Core Features:
- **IoC Container**: จัดการ object lifecycle และ dependencies
- **AOP**: Aspect-Oriented Programming สำหรับ cross-cutting concerns
- **Data Access**: JDBC, ORM, Transactions
- **Web MVC**: Web framework สำหรับ REST APIs และ web apps
- **Testing**: First-class testing support

### Spring Ecosystem:
```
Spring Framework (Core)
├── Spring Boot         → Auto-configuration, embedded server
├── Spring Data         → Database access (JPA, MongoDB, Redis)
├── Spring Security     → Authentication, Authorization
├── Spring Cloud        → Microservices, distributed systems
├── Spring Batch        → Batch processing
└── Spring Integration  → Enterprise integration patterns
```

### Maven Dependencies

```xml
<!-- pom.xml for standalone Spring application -->
<dependencies>
    <!-- Spring Core (IoC Container) -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>6.1.1</version>
    </dependency>
    
    <!-- Spring AOP -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>6.1.1</version>
    </dependency>
    
    <!-- Logging -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.4.14</version>
    </dependency>
    
    <!-- Testing -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>6.1.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 2. IoC Concept - Inversion of Control

**Traditional approach** (Tight coupling):
```java
// Without IoC - objects create their own dependencies
class OrderService {
    // Tight coupling - hard to test, hard to change
    private EmailService emailService = new EmailServiceImpl();
    private OrderRepository orderRepository = new JdbcOrderRepository();
    
    void placeOrder(Order order) {
        orderRepository.save(order);
        emailService.send(order.getCustomerEmail(), "Order confirmed");
    }
}
```

**IoC approach** (Loose coupling):
```java
// With IoC - dependencies are injected from outside
class OrderService {
    private final EmailService emailService;     // Interface, not implementation
    private final OrderRepository orderRepository;
    
    // Dependencies injected via constructor
    OrderService(EmailService emailService, OrderRepository orderRepository) {
        this.emailService = emailService;
        this.orderRepository = orderRepository;
    }
    
    void placeOrder(Order order) {
        orderRepository.save(order);
        emailService.send(order.getCustomerEmail(), "Order confirmed");
    }
}
// Now Spring manages: creating instances, injecting dependencies, lifecycle
```

---

## 3. First Spring Application

```java
// === Interfaces ===
public interface GreetingService {
    String greet(String name);
}

public interface MessageFormatter {
    String format(String message);
}

// === Implementations ===
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;

@Service  // Marks this as a Spring-managed bean
public class GreetingServiceImpl implements GreetingService {
    
    private final MessageFormatter formatter;
    
    // Constructor injection (recommended)
    public GreetingServiceImpl(MessageFormatter formatter) {
        this.formatter = formatter;
    }
    
    @Override
    public String greet(String name) {
        String message = "Hello, " + name + "!";
        return formatter.format(message);
    }
}

@Component  // Generic Spring component
public class UpperCaseFormatter implements MessageFormatter {
    
    @Override
    public String format(String message) {
        return message.toUpperCase();
    }
}

// === Configuration ===
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.example")  // Scan for @Component, @Service, etc.
public class AppConfig {
    // Spring will auto-detect beans via component scanning
}

// === Main Application ===
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Main {
    public static void main(String[] args) {
        // Create Spring container
        ApplicationContext context = 
            new AnnotationConfigApplicationContext(AppConfig.class);
        
        // Get bean from container
        GreetingService greetingService = context.getBean(GreetingService.class);
        
        System.out.println(greetingService.greet("World"));     // HELLO, WORLD!
        System.out.println(greetingService.greet("Spring"));    // HELLO, SPRING!
        
        // Close context
        ((AnnotationConfigApplicationContext) context).close();
    }
}
```

---

## 4. Bean Annotations

```java
import org.springframework.stereotype.*;
import org.springframework.context.annotation.*;

// @Component - generic Spring component
@Component
public class GenericComponent { }

// @Service - service layer (business logic)
@Service
public class UserService { }

// @Repository - data access layer
@Repository
public class UserRepository { }

// @Controller - web layer (Spring MVC)
@Controller
public class UserController { }

// @RestController - @Controller + @ResponseBody
@RestController
public class UserApiController { }

// Custom bean name
@Service("myCustomService")
public class MyService { }

// @Component with custom name
@Component("emailSender")
public class SmtpEmailSender { }
```

---

## 5. Java-based Configuration (@Configuration & @Bean)

```java
import org.springframework.context.annotation.*;
import org.springframework.beans.factory.annotation.Value;

@Configuration
public class DatabaseConfig {
    
    // @Value injects values from properties
    @Value("${db.url:jdbc:h2:mem:testdb}")
    private String dbUrl;
    
    @Value("${db.username:sa}")
    private String dbUsername;
    
    @Value("${db.password:}")
    private String dbPassword;
    
    // @Bean - define a bean explicitly
    @Bean
    public DataSource dataSource() {
        // This would normally return a real DataSource
        System.out.println("Creating DataSource: " + dbUrl);
        return new SimpleDataSource(dbUrl, dbUsername, dbPassword);
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        // Spring injects dataSource bean automatically
        return new JdbcTemplate(dataSource);
    }
    
    // Named bean
    @Bean("primaryDataSource")
    public DataSource primaryDataSource() {
        return new SimpleDataSource("jdbc:mysql://primary:3306/mydb", "root", "");
    }
    
    @Bean("secondaryDataSource")
    public DataSource secondaryDataSource() {
        return new SimpleDataSource("jdbc:mysql://secondary:3306/mydb", "root", "");
    }
}

@Configuration
public class AppConfig {
    
    // Import other config classes
    @Bean
    public UserRepository userRepository(DataSource dataSource) {
        return new JdbcUserRepository(dataSource);
    }
    
    @Bean
    public UserService userService(UserRepository userRepository, 
                                   EmailService emailService) {
        return new UserServiceImpl(userRepository, emailService);
    }
    
    @Bean
    public EmailService emailService() {
        return new SmtpEmailService("smtp.gmail.com", 587);
    }
}

// Stub implementations for examples
interface DataSource {}
class SimpleDataSource implements DataSource {
    SimpleDataSource(String url, String user, String pass) {}
}
class JdbcTemplate {
    JdbcTemplate(DataSource ds) {}
}
interface UserRepository {}
class JdbcUserRepository implements UserRepository {
    JdbcUserRepository(DataSource ds) {}
}
interface EmailService {}
class SmtpEmailService implements EmailService {
    SmtpEmailService(String host, int port) {}
}
interface UserService {}
class UserServiceImpl implements UserService {
    UserServiceImpl(UserRepository repo, EmailService es) {}
}
```

---

## 6. Dependency Injection Types

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

// === 1. Constructor Injection (RECOMMENDED) ===
@Service
public class ProductService {
    
    private final ProductRepository productRepository;
    private final InventoryService inventoryService;
    private final PricingService pricingService;
    
    // @Autowired optional when only one constructor
    @Autowired
    public ProductService(
            ProductRepository productRepository,
            InventoryService inventoryService,
            PricingService pricingService) {
        this.productRepository = productRepository;
        this.inventoryService = inventoryService;
        this.pricingService = pricingService;
    }
    
    public ProductDetails getProductDetails(String productId) {
        var product = productRepository.findById(productId);
        var stock = inventoryService.getStock(productId);
        var price = pricingService.getPrice(productId);
        return new ProductDetails(product, stock, price);
    }
}

// === 2. Setter Injection ===
@Service
public class NotificationService {
    
    private EmailService emailService;
    private SmsService smsService;
    
    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
    
    @Autowired(required = false)  // Optional dependency
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }
    
    public void notify(String message) {
        emailService.send(message);
        if (smsService != null) {
            smsService.send(message);
        }
    }
}

// === 3. Field Injection (NOT recommended for production) ===
@Service
public class ReportService {
    
    @Autowired  // Avoid this in production code
    private ReportRepository reportRepository;
    
    @Autowired
    private ExportService exportService;
}

// Why constructor injection is preferred:
// 1. Immutable dependencies (final fields)
// 2. Makes dependencies explicit
// 3. Easier to test (no Spring context needed)
// 4. Fails fast if dependency is missing
// 5. No circular dependency risk at runtime
```

---

## 7. @Qualifier and @Primary

```java
import org.springframework.beans.factory.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.context.annotation.*;

// Multiple implementations of same interface
public interface PaymentProcessor {
    void process(double amount);
}

@Component("creditCardProcessor")
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public void process(double amount) {
        System.out.println("Processing credit card payment: " + amount);
    }
}

@Component("paypalProcessor")
public class PayPalProcessor implements PaymentProcessor {
    @Override
    public void process(double amount) {
        System.out.println("Processing PayPal payment: " + amount);
    }
}

@Primary  // Default when no qualifier specified
@Component("stripeProcessor")
public class StripeProcessor implements PaymentProcessor {
    @Override
    public void process(double amount) {
        System.out.println("Processing Stripe payment: " + amount);
    }
}

@Service
public class PaymentService {
    
    private final PaymentProcessor defaultProcessor;    // Gets @Primary (Stripe)
    private final PaymentProcessor creditCardProcessor; // Qualified
    private final PaymentProcessor paypalProcessor;     // Qualified
    
    @Autowired
    public PaymentService(
            PaymentProcessor defaultProcessor,
            @Qualifier("creditCardProcessor") PaymentProcessor creditCardProcessor,
            @Qualifier("paypalProcessor") PaymentProcessor paypalProcessor) {
        this.defaultProcessor = defaultProcessor;
        this.creditCardProcessor = creditCardProcessor;
        this.paypalProcessor = paypalProcessor;
    }
    
    public void processPayment(String method, double amount) {
        switch (method.toLowerCase()) {
            case "credit" -> creditCardProcessor.process(amount);
            case "paypal" -> paypalProcessor.process(amount);
            default -> defaultProcessor.process(amount);
        }
    }
}
```

---

## 8. Bean Scopes

```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.*;
import org.springframework.web.context.annotation.*;

// === Singleton (default) - one instance per Spring container ===
@Service
// @Scope("singleton") // default, can be omitted
public class UserService2 {
    private int callCount = 0;
    
    public int getCallCount() { return ++callCount; }
}

// === Prototype - new instance every time ===
@Component
@Scope("prototype")
public class ReportGenerator {
    private final String reportId = java.util.UUID.randomUUID().toString();
    
    public String getReportId() { return reportId; }
}

// === Request scope - one per HTTP request (web context) ===
@Component
@RequestScope
public class RequestContext {
    private String requestId = java.util.UUID.randomUUID().toString();
    private long startTime = System.currentTimeMillis();
    
    public String getRequestId() { return requestId; }
    public long getElapsedMs() { return System.currentTimeMillis() - startTime; }
}

// === Session scope - one per HTTP session ===
@Component
@SessionScope
public class ShoppingCartSession {
    private final java.util.List<String> items = new java.util.ArrayList<>();
    
    public void addItem(String item) { items.add(item); }
    public java.util.List<String> getItems() { return java.util.List.copyOf(items); }
}

// === Demonstrating scope behavior ===
public class ScopeDemo {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext(AppScopeConfig.class);
        
        // Singleton - same instance
        UserService2 s1 = context.getBean(UserService2.class);
        UserService2 s2 = context.getBean(UserService2.class);
        System.out.println(s1 == s2);              // true
        System.out.println(s1.getCallCount());     // 1
        System.out.println(s2.getCallCount());     // 2 (same instance)
        
        // Prototype - different instances
        ReportGenerator r1 = context.getBean(ReportGenerator.class);
        ReportGenerator r2 = context.getBean(ReportGenerator.class);
        System.out.println(r1 == r2);              // false
        System.out.println(r1.getReportId().equals(r2.getReportId())); // false
        
        context.close();
    }
}

@Configuration
@ComponentScan("com.example")
class AppScopeConfig {}
```

---

## 9. Bean Lifecycle

```java
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.*;
import jakarta.annotation.*;
import org.springframework.context.*;

// === Method 1: @PostConstruct and @PreDestroy (Recommended) ===
@Service
public class CacheService {
    
    private java.util.Map<String, Object> cache;
    
    @PostConstruct
    public void init() {
        cache = new java.util.concurrent.ConcurrentHashMap<>();
        loadInitialData();
        System.out.println("CacheService initialized with " + cache.size() + " entries");
    }
    
    @PreDestroy
    public void cleanup() {
        cache.clear();
        System.out.println("CacheService cleaned up");
    }
    
    private void loadInitialData() {
        cache.put("config.timeout", 30);
        cache.put("config.maxRetries", 3);
    }
    
    public Object get(String key) { return cache.get(key); }
    public void put(String key, Object value) { cache.put(key, value); }
}

// === Method 2: @Bean with init/destroy methods ===
@Configuration
public class LifecycleConfig {
    
    @Bean(initMethod = "connect", destroyMethod = "disconnect")
    public DatabaseConnectionPool connectionPool() {
        return new DatabaseConnectionPool("jdbc:postgresql://localhost/mydb", 10);
    }
}

class DatabaseConnectionPool {
    private final String url;
    private final int maxConnections;
    
    DatabaseConnectionPool(String url, int maxConnections) {
        this.url = url;
        this.maxConnections = maxConnections;
    }
    
    public void connect() {
        System.out.println("Connection pool connected to: " + url);
    }
    
    public void disconnect() {
        System.out.println("Connection pool disconnected");
    }
}

// === Method 3: Implement Spring lifecycle interfaces ===
@Service
public class MetricsCollector implements InitializingBean, DisposableBean, 
                                         ApplicationContextAware {
    
    private ApplicationContext applicationContext;
    private java.util.Timer timer;
    
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.applicationContext = ctx;
        System.out.println("ApplicationContext set");
    }
    
    @Override
    public void afterPropertiesSet() {
        timer = new java.util.Timer(true);
        timer.scheduleAtFixedRate(
            new java.util.TimerTask() {
                @Override
                public void run() {
                    collectMetrics();
                }
            }, 
            0, 5000
        );
        System.out.println("MetricsCollector started");
    }
    
    @Override
    public void destroy() {
        if (timer != null) timer.cancel();
        System.out.println("MetricsCollector stopped");
    }
    
    private void collectMetrics() {
        // Collect application metrics
        Runtime rt = Runtime.getRuntime();
        long usedMemory = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
        System.out.println("Memory used: " + usedMemory + " MB");
    }
}

// === Full lifecycle order ===
/*
Bean lifecycle in Spring:
1. Bean instantiation (constructor)
2. Dependency injection (setters/fields)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. @PostConstruct / InitializingBean.afterPropertiesSet() / init-method
7. Bean is ready for use
...
8. @PreDestroy / DisposableBean.destroy() / destroy-method
9. Bean is removed from container
*/
```

---

## 10. Properties and @Value

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.env.Environment;

// === application.properties ===
// app.name=My Application
// app.version=1.0.0
// server.port=8080
// db.pool.size=10
// feature.new-ui.enabled=true
// admin.emails=admin1@example.com,admin2@example.com

@Configuration
@PropertySource("classpath:application.properties")
@PropertySource(value = "classpath:application-${spring.profiles.active}.properties",
                ignoreResourceNotFound = true)
public class PropertyConfig {
    
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.version:0.0.1}")  // Default value
    private String appVersion;
    
    @Value("${server.port:8080}")
    private int serverPort;
    
    @Value("${db.pool.size:5}")
    private int dbPoolSize;
    
    @Value("${feature.new-ui.enabled:false}")
    private boolean newUiEnabled;
    
    // Inject list
    @Value("${admin.emails}")
    private String[] adminEmails;
    
    // SpEL (Spring Expression Language)
    @Value("#{systemProperties['user.home']}")
    private String userHome;
    
    @Value("#{T(java.lang.Math).PI}")
    private double pi;
    
    @Value("#{${app.version}.split('\\.')[0]}")  // Extract major version
    private String majorVersion;
    
    @Bean
    public AppInfo appInfo() {
        return new AppInfo(appName, appVersion, serverPort);
    }
}

record AppInfo(String name, String version, int port) {}

// === @ConfigurationProperties (cleaner approach) ===
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.*;

@ConfigurationProperties(prefix = "app")
@Validated  // Enable JSR-303 validation
public class AppProperties {
    
    @jakarta.validation.constraints.NotBlank
    private String name;
    
    private String version = "0.0.1";
    
    private Database database = new Database();
    
    private Features features = new Features();
    
    public static class Database {
        private String url;
        private String username;
        private String password;
        private int poolSize = 10;
        
        // getters and setters...
        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }
        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public String getPassword() { return password; }
        public void setPassword(String password) { this.password = password; }
        public int getPoolSize() { return poolSize; }
        public void setPoolSize(int poolSize) { this.poolSize = poolSize; }
    }
    
    public static class Features {
        private boolean newUiEnabled = false;
        private boolean darkModeEnabled = true;
        
        public boolean isNewUiEnabled() { return newUiEnabled; }
        public void setNewUiEnabled(boolean enabled) { this.newUiEnabled = enabled; }
        public boolean isDarkModeEnabled() { return darkModeEnabled; }
        public void setDarkModeEnabled(boolean enabled) { this.darkModeEnabled = enabled; }
    }
    
    // getters and setters...
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getVersion() { return version; }
    public void setVersion(String version) { this.version = version; }
    public Database getDatabase() { return database; }
    public Features getFeatures() { return features; }
}

// application.yml (YAML format - cleaner than .properties)
/*
app:
  name: My Application
  version: 1.0.0
  database:
    url: jdbc:postgresql://localhost/mydb
    username: myuser
    password: secret
    pool-size: 20
  features:
    new-ui-enabled: true
    dark-mode-enabled: true
*/
```

---

## 11. Spring Profiles

```java
import org.springframework.context.annotation.*;

// === Bean only in specific profile ===
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Profile("development")
    public DataSource developmentDataSource() {
        System.out.println("Using H2 in-memory database");
        return new SimpleDataSource("jdbc:h2:mem:devdb", "sa", "");
    }
    
    @Bean
    @Profile("staging")
    public DataSource stagingDataSource() {
        System.out.println("Using staging database");
        return new SimpleDataSource("jdbc:postgresql://staging-host/mydb", "user", "pass");
    }
    
    @Bean
    @Profile("production")
    public DataSource productionDataSource() {
        System.out.println("Using production database");
        return new SimpleDataSource("jdbc:postgresql://prod-host/mydb", "user", "pass");
    }
    
    // Available in development AND test profiles
    @Bean
    @Profile({"development", "test"})
    public DataLoader dataLoader() {
        return new DataLoader();
    }
    
    // Available in all profiles EXCEPT production
    @Bean
    @Profile("!production")
    public DebugConsole debugConsole() {
        return new DebugConsole();
    }
}

class DataLoader {}
class DebugConsole {}

// === Profile-specific component ===
@Service
@Profile("production")
public class ProductionEmailService implements EmailService2 {
    @Override
    public void send(String to, String subject, String body) {
        System.out.println("PROD: Sending real email to: " + to);
    }
}

@Service
@Profile({"development", "test"})
public class MockEmailService implements EmailService2 {
    private final java.util.List<String> sentEmails = new java.util.ArrayList<>();
    
    @Override
    public void send(String to, String subject, String body) {
        sentEmails.add(to + ": " + subject);
        System.out.println("MOCK: Would send email to: " + to);
    }
    
    public java.util.List<String> getSentEmails() { return sentEmails; }
}

interface EmailService2 {
    void send(String to, String subject, String body);
}

// === Activate profiles ===
/*
Via JVM argument:
    -Dspring.profiles.active=production

Via environment variable:
    SPRING_PROFILES_ACTIVE=production

Via code:
*/
public class ProfileDemo {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext();
        context.getEnvironment().setActiveProfiles("development");
        context.register(DataSourceConfig.class);
        context.refresh();
        
        // Or use @ActiveProfiles in tests
    }
}

// === Programmatic profile checking ===
@Service
public class FeatureFlagService {
    
    private final org.springframework.core.env.Environment environment;
    
    public FeatureFlagService(org.springframework.core.env.Environment environment) {
        this.environment = environment;
    }
    
    public boolean isDevelopment() {
        return environment.acceptsProfiles(
            org.springframework.core.env.Profiles.of("development"));
    }
    
    public boolean isProduction() {
        return environment.acceptsProfiles(
            org.springframework.core.env.Profiles.of("production"));
    }
}
```

---

## 12. Conditional Beans

```java
import org.springframework.context.annotation.*;
import org.springframework.boot.autoconfigure.condition.*;

// === @Conditional annotations ===
@Configuration
public class ConditionalConfig {
    
    // Only if property exists and equals value
    @Bean
    @ConditionalOnProperty(name = "cache.enabled", havingValue = "true", 
                           matchIfMissing = false)
    public CacheManager cacheManager() {
        return new InMemoryCacheManager();
    }
    
    // Only if class is on classpath
    @Bean
    @ConditionalOnClass(name = "com.fasterxml.jackson.databind.ObjectMapper")
    public JsonSerializer jsonSerializer() {
        return new JacksonJsonSerializer();
    }
    
    // Only if another bean is missing
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager noOpCacheManager() {
        return new NoOpCacheManager();
    }
    
    // Only if expression evaluates to true
    @Bean
    @ConditionalOnExpression("${feature.analytics.enabled:false} && ${env} != 'test'")
    public AnalyticsService analyticsService() {
        return new GoogleAnalyticsService();
    }
}

interface CacheManager { Object get(String key); void put(String key, Object value); }
class InMemoryCacheManager implements CacheManager {
    private final java.util.Map<String, Object> cache = new java.util.HashMap<>();
    public Object get(String key) { return cache.get(key); }
    public void put(String key, Object value) { cache.put(key, value); }
}
class NoOpCacheManager implements CacheManager {
    public Object get(String key) { return null; }
    public void put(String key, Object value) {}
}
interface JsonSerializer {}
class JacksonJsonSerializer implements JsonSerializer {}
interface AnalyticsService {}
class GoogleAnalyticsService implements AnalyticsService {}

// === Custom Condition ===
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = System.getProperty("os.name", "").toLowerCase();
        return osName.contains("linux");
    }
}

@Bean
@Conditional(OnLinuxCondition.class)
public SystemMonitor linuxSystemMonitor() {
    return new LinuxSystemMonitor();
}

interface SystemMonitor {}
class LinuxSystemMonitor implements SystemMonitor {}
```

---

## 13. ApplicationContext Events

```java
import org.springframework.context.*;
import org.springframework.context.event.*;
import org.springframework.stereotype.*;

// === Custom Events ===
public class UserRegisteredEvent extends ApplicationEvent {
    private final String userId;
    private final String email;
    
    public UserRegisteredEvent(Object source, String userId, String email) {
        super(source);
        this.userId = userId;
        this.email = email;
    }
    
    public String getUserId() { return userId; }
    public String getEmail() { return email; }
}

// === Event Publisher ===
@Service
public class RegistrationService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public RegistrationService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    public void registerUser(String userId, String email) {
        // ... save user to database ...
        System.out.println("User registered: " + email);
        
        // Publish event - decoupled from handlers
        eventPublisher.publishEvent(new UserRegisteredEvent(this, userId, email));
    }
}

// === Event Listeners ===
@Component
public class WelcomeEmailListener {
    
    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        System.out.println("Sending welcome email to: " + event.getEmail());
    }
}

@Component
public class AuditLogListener {
    
    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        System.out.println("Audit: New user registered - " + event.getUserId());
    }
}

@Component
public class OnboardingWorkflowListener {
    
    @EventListener
    @Async  // Run asynchronously
    public void onUserRegistered(UserRegisteredEvent event) {
        System.out.println("Starting onboarding for: " + event.getUserId());
        // ... trigger onboarding workflow ...
    }
}

// === Built-in Events ===
@Component
public class ApplicationStartupListener {
    
    @EventListener(ContextRefreshedEvent.class)
    public void onContextRefreshed() {
        System.out.println("Spring context refreshed - application ready");
    }
    
    @EventListener(ContextStartedEvent.class)
    public void onContextStarted() {
        System.out.println("Spring context started");
    }
    
    @EventListener(ContextClosedEvent.class)
    public void onContextClosed() {
        System.out.println("Spring context closing - cleanup resources");
    }
    
    // ApplicationListener alternative
    @EventListener
    public void handleContextRefresh(ContextRefreshedEvent event) {
        ApplicationContext ctx = event.getApplicationContext();
        System.out.println("Beans in context: " + ctx.getBeanDefinitionCount());
    }
}
```

---

## 14. Complete IoC Example: Order Processing System

```java
import org.springframework.context.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.context.ApplicationEventPublisher;
import java.util.*;

// === Domain ===
record Product2(String id, String name, double price) {}
record Order2(String id, String customerId, List<OrderItem2> items) {
    double getTotal() {
        return items.stream().mapToDouble(OrderItem2::subtotal).sum();
    }
}
record OrderItem2(Product2 product, int quantity) {
    double subtotal() { return product.price() * quantity; }
}

// === Events ===
class OrderPlacedEvent extends org.springframework.context.ApplicationEvent {
    private final Order2 order;
    OrderPlacedEvent(Object source, Order2 order) {
        super(source);
        this.order = order;
    }
    Order2 getOrder() { return order; }
}

class OrderShippedEvent extends org.springframework.context.ApplicationEvent {
    private final String orderId;
    OrderShippedEvent(Object source, String orderId) {
        super(source);
        this.orderId = orderId;
    }
    String getOrderId() { return orderId; }
}

// === Repositories ===
interface OrderRepository2 {
    void save(Order2 order);
    Optional<Order2> findById(String id);
    List<Order2> findByCustomerId(String customerId);
}

interface ProductRepository2 {
    Optional<Product2> findById(String id);
}

@Repository
class InMemoryOrderRepository implements OrderRepository2 {
    private final Map<String, Order2> orders = new HashMap<>();
    
    @Override
    public void save(Order2 order) {
        orders.put(order.id(), order);
    }
    
    @Override
    public Optional<Order2> findById(String id) {
        return Optional.ofNullable(orders.get(id));
    }
    
    @Override
    public List<Order2> findByCustomerId(String customerId) {
        return orders.values().stream()
            .filter(o -> o.customerId().equals(customerId))
            .toList();
    }
}

@Repository
class InMemoryProductRepository implements ProductRepository2 {
    private static final Map<String, Product2> CATALOG = Map.of(
        "P001", new Product2("P001", "Laptop", 999.99),
        "P002", new Product2("P002", "Mouse", 29.99),
        "P003", new Product2("P003", "Keyboard", 79.99)
    );
    
    @Override
    public Optional<Product2> findById(String id) {
        return Optional.ofNullable(CATALOG.get(id));
    }
}

// === Services ===
@Service
class OrderService2 {
    
    private final OrderRepository2 orderRepository;
    private final ProductRepository2 productRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    OrderService2(OrderRepository2 orderRepository, 
                  ProductRepository2 productRepository,
                  ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.eventPublisher = eventPublisher;
    }
    
    Order2 placeOrder(String customerId, Map<String, Integer> productQuantities) {
        List<OrderItem2> items = new ArrayList<>();
        
        for (Map.Entry<String, Integer> entry : productQuantities.entrySet()) {
            Product2 product = productRepository.findById(entry.getKey())
                .orElseThrow(() -> new RuntimeException("Product not found: " + entry.getKey()));
            items.add(new OrderItem2(product, entry.getValue()));
        }
        
        Order2 order = new Order2(
            UUID.randomUUID().toString(),
            customerId,
            List.copyOf(items)
        );
        
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderPlacedEvent(this, order));
        
        return order;
    }
    
    void shipOrder(String orderId) {
        Order2 order = orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("Order not found: " + orderId));
        
        eventPublisher.publishEvent(new OrderShippedEvent(this, orderId));
    }
}

// === Event Handlers ===
@Component
class OrderNotificationService {
    
    @org.springframework.context.event.EventListener
    void onOrderPlaced(OrderPlacedEvent event) {
        Order2 order = event.getOrder();
        System.out.printf("📧 Order confirmation sent for order %s (total: $%.2f)%n",
            order.id(), order.getTotal());
    }
    
    @org.springframework.context.event.EventListener
    void onOrderShipped(OrderShippedEvent event) {
        System.out.printf("📦 Shipment notification sent for order %s%n", 
            event.getOrderId());
    }
}

@Component
class OrderAuditService {
    private final List<String> auditLog = new ArrayList<>();
    
    @org.springframework.context.event.EventListener
    void onOrderPlaced(OrderPlacedEvent event) {
        String entry = String.format("[%s] ORDER_PLACED: orderId=%s, customerId=%s, total=%.2f",
            java.time.Instant.now(), event.getOrder().id(), 
            event.getOrder().customerId(), event.getOrder().getTotal());
        auditLog.add(entry);
        System.out.println("📋 Audit: " + entry);
    }
    
    List<String> getAuditLog() { return List.copyOf(auditLog); }
}

// === Configuration ===
@Configuration
@ComponentScan("com.example.order")
class OrderConfig {
    // All beans are discovered via @ComponentScan
}

// === Main Application ===
public class OrderProcessingApp {
    public static void main(String[] args) {
        var context = new AnnotationConfigApplicationContext();
        context.scan(""); // Scan current package
        context.register(OrderConfig.class);
        context.refresh();
        
        OrderService2 orderService = context.getBean(OrderService2.class);
        
        // Place an order
        Order2 order = orderService.placeOrder(
            "CUSTOMER-001",
            Map.of("P001", 1, "P002", 2)
        );
        
        System.out.println("\n=== Order Placed ===");
        System.out.printf("Order ID: %s%n", order.id());
        System.out.printf("Customer: %s%n", order.customerId());
        System.out.println("Items:");
        order.items().forEach(item -> 
            System.out.printf("  - %s x%d = $%.2f%n",
                item.product().name(), item.quantity(), item.subtotal()));
        System.out.printf("Total: $%.2f%n", order.getTotal());
        
        // Ship the order
        System.out.println("\n=== Shipping Order ===");
        orderService.shipOrder(order.id());
        
        // Check audit log
        OrderAuditService auditService = context.getBean(OrderAuditService.class);
        System.out.println("\n=== Audit Log ===");
        auditService.getAuditLog().forEach(System.out::println);
        
        context.close();
    }
}
```

---

## สรุป Part 021

| Concept | Description |
|---------|-------------|
| IoC Container | Spring manages object creation and wiring |
| ApplicationContext | Full-featured Spring container |
| @Component, @Service, @Repository | Mark classes as Spring beans |
| @Configuration + @Bean | Java-based bean definition |
| Constructor Injection | Recommended DI approach |
| @Qualifier, @Primary | Resolve ambiguity with multiple beans |
| @Scope | Singleton, Prototype, Request, Session |
| @PostConstruct, @PreDestroy | Bean lifecycle callbacks |
| @Profile | Environment-specific beans |
| ApplicationEventPublisher | Decoupled event communication |

---

**Part 022:** Spring AOP - Aspect-Oriented Programming
- What is AOP and why use it?
- @Aspect, @Before, @After, @Around, @AfterReturning, @AfterThrowing
- Pointcut expressions
- Logging, Transaction, Security aspects
- Custom annotations with AOP
