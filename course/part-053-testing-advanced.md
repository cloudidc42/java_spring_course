# Part 053: Advanced Testing Strategies

## เนื้อหาในส่วนนี้
- Contract Testing with Spring Cloud Contract
- Consumer-Driven Contract Testing
- TestContainers for integration tests
- Mutation Testing with PIT
- Performance/Load Testing with Gatling
- Property-Based Testing
- Architecture Testing with ArchUnit
- Test Doubles: Spy, Stub, Fake, Mock
- Test Data Builders and Object Mothers
- Testing Kafka consumers and producers

---

## 1. Spring Cloud Contract

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>

<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>4.1.0</version>
    <extensions>true</extensions>
</plugin>
```

```groovy
// src/test/resources/contracts/order/shouldReturnOrder.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return order by id"
    
    request {
        method GET()
        url "/api/orders/ORD-001"
        headers {
            contentType(applicationJson())
        }
    }
    
    response {
        status OK()
        body([
            orderId: "ORD-001",
            userId: 1,
            total: 99.99,
            status: "CONFIRMED"
        ])
        headers {
            contentType(applicationJson())
        }
    }
}
```

```java
// Producer side: base test class
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import io.restassured.RestAssured;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class ContractBaseTest {
    
    @LocalServerPort
    int port;
    
    @BeforeEach
    void setup() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
        
        // Set up test data
        setupOrderTestData();
    }
    
    void setupOrderTestData() {
        // Pre-create the order ORD-001 in test DB
    }
}

// Consumer side: use stub runner
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:order-service:+:stubs:8082",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderServiceConsumerTest {
    
    @Autowired
    UserServiceClient orderClient;  // Feign client
    
    @Test
    void shouldGetOrderFromStub() {
        // Stub Runner auto-starts the stub on port 8082
        // No need to run real order-service
        var order = orderClient.getOrder("ORD-001");
        
        assertThat(order.orderId()).isEqualTo("ORD-001");
        assertThat(order.total()).isEqualTo(99.99);
    }
}
```

---

## 2. TestContainers

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>redis</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
import org.testcontainers.containers.*;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.*;

@Testcontainers
@SpringBootTest
class OrderRepositoryIntegrationTest {
    
    // Shared containers (reused across tests for speed)
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("db/test-data.sql");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379)
        .waitingFor(Wait.forLogMessage(".*Ready to accept connections.*", 1));
    
    // Inject container properties into Spring context
    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
    
    @Autowired
    OrderRepository orderRepository;
    
    @Test
    void shouldSaveAndFindOrder() {
        var order = new Order("ORD-TEST", 1L, List.of("P1"), 99.99, "PENDING");
        orderRepository.save(order);
        
        var found = orderRepository.findById("ORD-TEST");
        
        assertThat(found).isPresent();
        assertThat(found.get().total()).isEqualTo(99.99);
    }
}

// Reusable base class for all integration tests
@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
public abstract class AbstractIntegrationTest {
    
    static final PostgreSQLContainer<?> postgres;
    static final KafkaContainer kafka;
    
    static {
        postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
        kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
        
        // Start containers once, share across test classes
        Startables.deepStart(postgres, kafka).join();
    }
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}

// Extend for specific tests
class UserServiceIntegrationTest extends AbstractIntegrationTest {
    
    @Autowired
    MockMvc mockMvc;
    
    @Test
    void shouldCreateUser() throws Exception {
        mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"name\":\"John\",\"email\":\"john@test.com\"}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.email").value("john@test.com"));
    }
}

import org.testcontainers.utility.DockerImageName;
import org.testcontainers.lifecycle.Startables;
import org.springframework.test.web.servlet.MockMvc;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.assertj.core.api.Assertions.*;
import java.util.List;
```

---

## 3. Mutation Testing with PIT

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.0</version>
    <dependencies>
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <targetClasses>
            <param>com.example.service.*</param>
        </targetClasses>
        <targetTests>
            <param>com.example.*Test</param>
        </targetTests>
        <mutators>STRONGER</mutators>
        <outputFormats>
            <param>HTML</param>
            <param>XML</param>
        </outputFormats>
        <threads>4</threads>
    </configuration>
</plugin>
```

```bash
# Run mutation tests
mvn org.pitest:pitest-maven:mutationCoverage

# Report at target/pit-reports/YYYYMMDD*/index.html
# Mutation score: % of mutations caught by tests
# Target: >80% mutation score for critical business logic
```

```java
// Example: Service with business logic
public class PricingService {
    
    public double calculateDiscount(double price, int quantity) {
        if (quantity >= 10) {
            return price * 0.20;  // 20% discount
        } else if (quantity >= 5) {
            return price * 0.10;  // 10% discount
        }
        return 0;
    }
    
    public double applyVAT(double price, boolean vatExempt) {
        return vatExempt ? price : price * 1.07;  // 7% VAT
    }
}

// Tests that survive mutation testing (thorough boundary tests)
class PricingServiceMutationTest {
    
    PricingService service = new PricingService();
    
    @Test
    void discount_exactlyTen_is20Percent() {
        assertThat(service.calculateDiscount(100.0, 10)).isEqualTo(20.0);
    }
    
    @Test
    void discount_eleven_is20Percent() {
        assertThat(service.calculateDiscount(100.0, 11)).isEqualTo(20.0);
    }
    
    @Test
    void discount_nine_is10Percent() {
        assertThat(service.calculateDiscount(100.0, 9)).isEqualTo(10.0);
    }
    
    @Test
    void discount_exactlyFive_is10Percent() {
        assertThat(service.calculateDiscount(100.0, 5)).isEqualTo(10.0);
    }
    
    @Test
    void discount_four_isZero() {
        assertThat(service.calculateDiscount(100.0, 4)).isEqualTo(0.0);
    }
    
    @Test
    void vat_notExempt_adds7Percent() {
        assertThat(service.applyVAT(100.0, false)).isEqualTo(107.0);
    }
    
    @Test
    void vat_exempt_noChange() {
        assertThat(service.applyVAT(100.0, true)).isEqualTo(100.0);
    }
}
```

---

## 4. Architecture Testing with ArchUnit

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.2.1</version>
    <scope>test</scope>
</dependency>
```

```java
import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.lang.ArchRule;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.*;
import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "com.example")
class ArchitectureTest {
    
    // Layer architecture rules
    @ArchTest
    static final ArchRule layeringRule = layeredArchitecture()
        .consideringAllDependencies()
        .layer("Controller").definedBy("..controller..")
        .layer("Service").definedBy("..service..")
        .layer("Repository").definedBy("..repository..")
        .layer("Domain").definedBy("..domain..")
        
        .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
        .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
        .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
        .whereLayer("Domain").mayOnlyBeAccessedByLayers("Controller", "Service", "Repository");
    
    // Controllers must have @RestController
    @ArchTest
    static final ArchRule controllerRule = classes()
        .that().resideInAPackage("..controller..")
        .should().beAnnotatedWith(org.springframework.web.bind.annotation.RestController.class);
    
    // Services must have @Service or @Component
    @ArchTest
    static final ArchRule serviceAnnotationRule = classes()
        .that().resideInAPackage("..service..")
        .and().haveNameMatching(".*ServiceImpl")
        .should().beAnnotatedWith(org.springframework.stereotype.Service.class);
    
    // Repositories should extend JpaRepository
    @ArchTest
    static final ArchRule repositoryRule = classes()
        .that().resideInAPackage("..repository..")
        .and().areInterfaces()
        .should().beAssignableTo(org.springframework.data.jpa.repository.JpaRepository.class);
    
    // No cycles between packages
    @ArchTest
    static final ArchRule noCyclesRule = com.tngtech.archunit.library.dependencies.SlicesRuleDefinition
        .slices()
        .matching("com.example.(*)..")
        .should().beFreeOfCycles();
    
    // Domain classes should not depend on Spring
    @ArchTest
    static final ArchRule domainIndependence = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("org.springframework..");
    
    // No field injection (use constructor injection)
    @ArchTest
    static final ArchRule noFieldInjection = noFields()
        .should().beAnnotatedWith(org.springframework.beans.factory.annotation.Autowired.class)
        .as("Use constructor injection instead of @Autowired on fields");
    
    // Controllers should not call repositories directly
    @ArchTest
    static final ArchRule controllerNotAccessRepository = noClasses()
        .that().resideInAPackage("..controller..")
        .should().dependOnClassesThat()
        .resideInAPackage("..repository..")
        .as("Controllers should not access repositories directly");
}
```

---

## 5. Testing Kafka

```java
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.kafka.test.EmbeddedKafkaBroker;
import org.springframework.kafka.core.*;
import org.springframework.kafka.test.utils.KafkaTestUtils;
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerRecord;

@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"order-created", "payment-processed"},
    brokerProperties = {"listeners=PLAINTEXT://localhost:9092"}
)
class KafkaIntegrationTest {
    
    @Autowired
    OrderEventPublisher publisher;
    
    @Autowired
    EmbeddedKafkaBroker embeddedKafka;
    
    Consumer<String, OrderCreatedEvent> consumer;
    
    @BeforeEach
    void setUp() {
        Map<String, Object> props = KafkaTestUtils.consumerProps(
            "test-group", "true", embeddedKafka);
        consumer = new org.apache.kafka.clients.consumer.KafkaConsumer<>(props);
        embeddedKafka.consumeFromAnEmbeddedTopic(consumer, "order-created");
    }
    
    @AfterEach
    void tearDown() {
        consumer.close();
    }
    
    @Test
    void publisherShouldSendOrderCreatedEvent() throws Exception {
        var event = new OrderCreatedEvent("ORD-TEST", 1L, List.of("P1"), 99.99);
        publisher.publishOrderCreated(event);
        
        // Wait and consume message
        ConsumerRecord<String, OrderCreatedEvent> record =
            KafkaTestUtils.getSingleRecord(consumer, "order-created", Duration.ofSeconds(5));
        
        assertThat(record.key()).isEqualTo("ORD-TEST");
        assertThat(record.value().orderId()).isEqualTo("ORD-TEST");
        assertThat(record.value().total()).isEqualTo(99.99);
    }
    
    @Test
    void consumerShouldProcessPaymentEvent() throws Exception {
        // Publish to payment-processed topic
        Map<String, Object> producerProps = KafkaTestUtils.producerProps(embeddedKafka);
        KafkaTemplate<String, Object> template = new KafkaTemplate<>(
            new DefaultKafkaProducerFactory<>(producerProps));
        
        var paymentEvent = new PaymentProcessedEvent("ORD-001", "TXN-123", "COMPLETED");
        template.send("payment-processed", "ORD-001", paymentEvent).get();
        
        // Verify the consumer processed it
        await().atMost(10, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                var order = orderRepository.findById("ORD-001");
                assertThat(order).isPresent();
                assertThat(order.get().status()).isEqualTo("CONFIRMED");
            });
    }
}

import java.time.Duration;
import java.util.*;
import java.util.concurrent.TimeUnit;
import static org.awaitility.Awaitility.await;
```

---

## 6. Test Data Builders

```java
// Builder pattern for test data (removes boilerplate from tests)
public class OrderTestBuilder {
    private String orderId = "ORD-001";
    private Long userId = 1L;
    private List<String> productIds = List.of("P1", "P2");
    private double total = 99.99;
    private String status = "PENDING";
    private java.time.LocalDateTime createdAt = java.time.LocalDateTime.now();
    
    public static OrderTestBuilder anOrder() {
        return new OrderTestBuilder();
    }
    
    public OrderTestBuilder withOrderId(String orderId) {
        this.orderId = orderId;
        return this;
    }
    
    public OrderTestBuilder withUserId(Long userId) {
        this.userId = userId;
        return this;
    }
    
    public OrderTestBuilder withProducts(String... productIds) {
        this.productIds = List.of(productIds);
        return this;
    }
    
    public OrderTestBuilder withTotal(double total) {
        this.total = total;
        return this;
    }
    
    public OrderTestBuilder withStatus(String status) {
        this.status = status;
        return this;
    }
    
    public OrderTestBuilder confirmed() {
        return withStatus("CONFIRMED");
    }
    
    public OrderTestBuilder failed() {
        return withStatus("PAYMENT_FAILED");
    }
    
    public Order build() {
        return new Order(orderId, userId, productIds, total, status);
    }
    
    public Order save(OrderRepository repo) {
        return repo.save(build());
    }
}

// Object Mother: pre-built scenarios
public class OrderScenarios {
    public static Order pendingOrder() {
        return OrderTestBuilder.anOrder()
            .withOrderId("ORD-PENDING")
            .withUserId(100L)
            .build();
    }
    
    public static Order confirmedHighValueOrder() {
        return OrderTestBuilder.anOrder()
            .withOrderId("ORD-HV")
            .withProducts("LAPTOP", "MONITOR", "KEYBOARD")
            .withTotal(2499.99)
            .confirmed()
            .build();
    }
}

// Usage in tests
@Test
void vipDiscountApplied_forHighValueConfirmedOrder() {
    var order = OrderScenarios.confirmedHighValueOrder();
    var discount = discountService.calculate(order);
    assertThat(discount.percentage()).isEqualTo(15);
}

@Test
void pendingOrderTransitionsToConfirmed() {
    var order = OrderTestBuilder.anOrder()
        .withOrderId("ORD-001")
        .withStatus("PENDING")
        .build();
    
    var updated = orderService.confirm(order);
    
    assertThat(updated.status()).isEqualTo("CONFIRMED");
}
```

---

## 7. Property-Based Testing with jqwik

```xml
<dependency>
    <groupId>net.jqwik</groupId>
    <artifactId>jqwik</artifactId>
    <version>1.8.2</version>
    <scope>test</scope>
</dependency>
```

```java
import net.jqwik.api.*;
import net.jqwik.api.constraints.*;

class PricingPropertyTest {
    
    PricingService service = new PricingService();
    
    // Property: discount never exceeds price
    @Property
    void discountNeverExceedsPrice(
            @ForAll @DoubleRange(min = 0, max = 10000) double price,
            @ForAll @IntRange(min = 0, max = 100) int quantity) {
        
        double discount = service.calculateDiscount(price, quantity);
        
        assertThat(discount).isLessThanOrEqualTo(price);
        assertThat(discount).isGreaterThanOrEqualTo(0);
    }
    
    // Property: VAT always increases or equal to original price
    @Property
    void vatNeverReducesPrice(@ForAll @DoubleRange(min = 0, max = 10000) double price) {
        double withVat = service.applyVAT(price, false);
        assertThat(withVat).isGreaterThanOrEqualTo(price);
    }
    
    // Property: string reversal
    @Property
    void reversedTwiceIsOriginal(@ForAll @NotEmpty String original) {
        String reversed = new StringBuilder(original).reverse().toString();
        String doubleReversed = new StringBuilder(reversed).reverse().toString();
        assertThat(doubleReversed).isEqualTo(original);
    }
    
    // Property: sorting preserves all elements
    @Property
    void sortPreservesElements(@ForAll List<@IntRange(min = 0, max = 1000) Integer> list) {
        List<Integer> sorted = new java.util.ArrayList<>(list);
        java.util.Collections.sort(sorted);
        
        assertThat(sorted).containsExactlyInAnyOrderElementsOf(list);
        assertThat(sorted).isSorted();
    }
}
```

---

## 8. Performance Testing with Gatling

```java
// src/test/java/simulations/OrderSimulation.java
import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;
import java.time.Duration;

public class OrderSimulation extends Simulation {
    
    HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json")
        .shareConnections();
    
    // Get auth token
    ChainBuilder login = exec(
        http("Login")
            .post("/api/auth/login")
            .body(StringBody("{\"email\":\"test@example.com\",\"password\":\"password\"}"))
            .check(jsonPath("$.token").saveAs("authToken"))
    );
    
    // Create order scenario
    ChainBuilder createOrder = exec(
        http("Create Order")
            .post("/api/orders")
            .header("Authorization", "Bearer #{authToken}")
            .body(StringBody("{\"productIds\":[\"P1\",\"P2\"]}"))
            .check(status().is(201))
            .check(jsonPath("$.orderId").saveAs("orderId"))
    ).pause(Duration.ofMillis(100));
    
    // Get order
    ChainBuilder getOrder = exec(
        http("Get Order")
            .get("/api/orders/#{orderId}")
            .header("Authorization", "Bearer #{authToken}")
            .check(status().is(200))
    );
    
    // Scenario: User creates and retrieves order
    ScenarioBuilder orderScenario = scenario("Order Flow")
        .exec(login)
        .pause(Duration.ofMillis(200))
        .repeat(5).on(
            exec(createOrder)
            .exec(getOrder)
            .pause(Duration.ofMillis(500))
        );
    
    {
        setUp(
            // Ramp up: 0 → 100 users over 30s, sustain for 60s
            orderScenario.injectOpen(
                nothingFor(5),
                rampUsers(50).during(30),
                constantUsersPerSec(100).during(60),
                rampUsersPerSec(100).to(0).during(10)
            )
        )
        .protocols(httpProtocol)
        .assertions(
            global().responseTime().percentile3().lte(500),  // p99 < 500ms
            global().successfulRequests().percent().gte(99.0),  // 99% success
            forAll().failedRequests().count().lte(10L)
        );
    }
}
```

```bash
# Run Gatling
mvn gatling:test -Dgatling.simulationClass=simulations.OrderSimulation
# Report: target/gatling/ordersimulation-*/index.html
```

---

## สรุป Part 053

| Strategy | Tool | Purpose |
|----------|------|---------|
| Contract Testing | Spring Cloud Contract | Consumer-Producer contract |
| Integration Tests | TestContainers | Real DB/Kafka in tests |
| Mutation Testing | PIT | Verify test quality |
| Architecture Tests | ArchUnit | Enforce layer rules |
| Kafka Testing | EmbeddedKafka | Test async messaging |
| Property Tests | jqwik | Random input testing |
| Load Testing | Gatling | Performance validation |

---

**Part 054:** Spring Boot Security Advanced - OAuth2 Resource Server, Method Security, CORS
