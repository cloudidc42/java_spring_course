# Part 083: Enterprise Integration Testing

## Overview

Integration testing verifies that multiple components work together correctly. Unlike unit tests, integration tests load a real (or near-real) Spring application context and exercise actual interactions between layers: controllers, services, repositories, and external systems.

This part covers every major integration testing tool available in the Spring ecosystem, with a complete e-commerce test suite as the capstone example.

---

## 1. @SpringBootTest Modes

`@SpringBootTest` starts the full application context. The `webEnvironment` attribute controls how the web layer is configured.

```java
// MOCK (default) - loads full context, mock servlet environment, use MockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
class OrderServiceIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldCreateOrder() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                        {"productId": 1, "quantity": 2}
                        """))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").exists());
    }
}

// RANDOM_PORT - starts real HTTP server on random port, use RestAssured or TestRestTemplate
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderControllerRandomPortTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldReturnOrderById() {
        ResponseEntity<OrderDto> response = restTemplate.getForEntity(
                "/api/orders/1", OrderDto.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
    }
}

// DEFINED_PORT - starts real HTTP server on the configured port (server.port)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class OrderControllerDefinedPortTest {
    // Uses port from application.properties (usually 8080)
}

// NONE - loads context but no servlet environment (use for service-layer integration tests)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class OrderBusinessLogicTest {

    @Autowired
    private OrderService orderService;

    @Test
    void shouldCalculateTotalWithDiscount() {
        OrderRequest request = new OrderRequest(List.of(
                new OrderItem(1L, 3),
                new OrderItem(2L, 1)
        ));
        OrderSummary summary = orderService.calculateTotal(request);
        assertThat(summary.getDiscountApplied()).isTrue();
    }
}
```

---

## 2. MockMvc – Deep Dive

MockMvc is the preferred way to test Spring MVC controllers without starting a real server.

```java
// pom.xml dependencies
// spring-boot-starter-test includes MockMvc
// testImplementation 'org.springframework.boot:spring-boot-starter-test'

@SpringBootTest
@AutoConfigureMockMvc
class ProductControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setUp() {
        productRepository.deleteAll();
        productRepository.save(Product.builder()
                .name("Laptop")
                .price(new BigDecimal("999.99"))
                .stock(10)
                .build());
    }

    @Test
    void shouldReturnProductList() throws Exception {
        mockMvc.perform(get("/api/products")
                .accept(MediaType.APPLICATION_JSON))
                .andDo(print())  // prints request/response to console
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.content", hasSize(1)))
                .andExpect(jsonPath("$.content[0].name").value("Laptop"))
                .andExpect(jsonPath("$.content[0].price").value(999.99));
    }

    @Test
    void shouldCreateProduct() throws Exception {
        CreateProductRequest request = new CreateProductRequest("Phone", new BigDecimal("599.99"), 50);

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.id").isNumber())
                .andExpect(jsonPath("$.name").value("Phone"));
    }

    @Test
    void shouldReturn404WhenProductNotFound() throws Exception {
        mockMvc.perform(get("/api/products/99999"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.errorCode").value("PRODUCT_NOT_FOUND"));
    }

    @Test
    void shouldReturn400WhenRequestInvalid() throws Exception {
        // missing required 'name' field
        String invalidRequest = """
                {"price": -10.00, "stock": 5}
                """;

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidRequest))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors").isArray())
                .andExpect(jsonPath("$.errors[*].field", hasItem("name")));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void shouldDeleteProductAsAdmin() throws Exception {
        Product product = productRepository.findAll().getFirst();

        mockMvc.perform(delete("/api/products/" + product.getId()))
                .andExpect(status().isNoContent());

        assertThat(productRepository.findById(product.getId())).isEmpty();
    }

    @Test
    @WithMockUser(roles = "USER")
    void shouldReturn403WhenUserDeletesProduct() throws Exception {
        Product product = productRepository.findAll().getFirst();

        mockMvc.perform(delete("/api/products/" + product.getId()))
                .andExpect(status().isForbidden());
    }

    @Test
    void shouldHandleMultipartFileUpload() throws Exception {
        MockMultipartFile image = new MockMultipartFile(
                "image", "product.jpg", "image/jpeg", "fake-image-data".getBytes());

        mockMvc.perform(multipart("/api/products/1/image")
                .file(image))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.imageUrl").exists());
    }
}
```

---

## 3. RestAssured Integration

RestAssured provides a fluent DSL for HTTP testing, especially useful with `RANDOM_PORT`.

```java
// pom.xml
// <dependency>
//   <groupId>io.rest-assured</groupId>
//   <artifactId>spring-mock-mvc</artifactId>
//   <scope>test</scope>
// </dependency>

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderRestAssuredTest {

    @LocalServerPort
    private int port;

    @Autowired
    private OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api";
    }

    @Test
    void shouldCreateAndRetrieveOrder() {
        // Create
        Integer orderId = given()
                .contentType(ContentType.JSON)
                .body("""
                        {
                          "customerId": 1,
                          "items": [
                            {"productId": 1, "quantity": 2}
                          ]
                        }
                        """)
                .when()
                .post("/orders")
                .then()
                .statusCode(201)
                .extract()
                .path("id");

        // Retrieve
        given()
                .pathParam("id", orderId)
                .when()
                .get("/orders/{id}")
                .then()
                .statusCode(200)
                .body("id", equalTo(orderId))
                .body("status", equalTo("PENDING"));
    }

    @Test
    void shouldFilterOrdersByStatus() {
        given()
                .queryParam("status", "PENDING")
                .queryParam("page", 0)
                .queryParam("size", 10)
                .when()
                .get("/orders")
                .then()
                .statusCode(200)
                .body("content.size()", greaterThan(0))
                .body("content.status", everyItem(equalTo("PENDING")));
    }

    @Test
    void shouldValidateResponseSchema() {
        given()
                .when()
                .get("/orders/1")
                .then()
                .statusCode(200)
                .body(matchesJsonSchemaInClasspath("schemas/order-response.json"));
    }

    // JSON schema file: src/test/resources/schemas/order-response.json
    /*
    {
      "$schema": "http://json-schema.org/draft-07/schema#",
      "type": "object",
      "required": ["id", "status", "totalAmount", "items"],
      "properties": {
        "id": { "type": "integer" },
        "status": { "type": "string", "enum": ["PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"] },
        "totalAmount": { "type": "number" },
        "items": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["productId", "quantity", "unitPrice"],
            "properties": {
              "productId": { "type": "integer" },
              "quantity": { "type": "integer" },
              "unitPrice": { "type": "number" }
            }
          }
        }
      }
    }
    */
}
```

---

## 4. Test Slices

Spring Boot provides "sliced" test contexts that load only part of the application context, making tests faster and more focused.

### 4.1 @WebMvcTest – Controller Layer Only

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;  // only the service is mocked

    @MockBean
    private ProductMapper productMapper;

    @Test
    void shouldDelegateToService() throws Exception {
        when(productService.findById(1L))
                .thenReturn(Optional.of(new ProductDto(1L, "Laptop", new BigDecimal("999.99"))));

        mockMvc.perform(get("/api/products/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("Laptop"));

        verify(productService).findById(1L);
    }

    @Test
    void shouldReturn404WhenServiceReturnsEmpty() throws Exception {
        when(productService.findById(99L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/products/99"))
                .andExpect(status().isNotFound());
    }
}
```

### 4.2 @DataJpaTest – Repository Layer Only

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class ProductRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindProductsByPriceRange() {
        entityManager.persist(Product.builder().name("Budget").price(new BigDecimal("99.99")).stock(20).build());
        entityManager.persist(Product.builder().name("Mid").price(new BigDecimal("499.99")).stock(15).build());
        entityManager.persist(Product.builder().name("Premium").price(new BigDecimal("1499.99")).stock(5).build());
        entityManager.flush();

        List<Product> results = productRepository.findByPriceBetween(
                new BigDecimal("100.00"), new BigDecimal("1000.00"));

        assertThat(results).hasSize(1);
        assertThat(results.getFirst().getName()).isEqualTo("Mid");
    }

    @Test
    void shouldReturnTopSellingProducts() {
        // ... setup order data
        List<ProductSalesDto> topSellers = productRepository.findTopSelling(5);
        assertThat(topSellers).hasSize(5);
        assertThat(topSellers.getFirst().getTotalSold())
                .isGreaterThanOrEqualTo(topSellers.get(1).getTotalSold());
    }
}
```

### 4.3 @DataRedisTest

```java
@DataRedisTest
@Testcontainers
class ProductCacheRepositoryTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureRedis(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private ProductCacheRepository cacheRepository;

    @Test
    void shouldCacheAndRetrieveProduct() {
        ProductDto product = new ProductDto(1L, "Laptop", new BigDecimal("999.99"));
        cacheRepository.save(product);

        Optional<ProductDto> cached = cacheRepository.findById(1L);

        assertThat(cached).isPresent();
        assertThat(cached.get().getName()).isEqualTo("Laptop");
    }

    @Test
    void shouldExpireCacheEntry() throws InterruptedException {
        cacheRepository.saveWithTtl(new ProductDto(2L, "Phone", new BigDecimal("599.99")), Duration.ofSeconds(1));
        Thread.sleep(1500);
        assertThat(cacheRepository.findById(2L)).isEmpty();
    }
}
```

### 4.4 @JsonTest

```java
@JsonTest
class OrderDtoJsonTest {

    @Autowired
    private JacksonTester<OrderDto> json;

    @Test
    void shouldSerializeOrderDto() throws IOException {
        OrderDto order = OrderDto.builder()
                .id(1L)
                .status(OrderStatus.PENDING)
                .totalAmount(new BigDecimal("199.98"))
                .createdAt(LocalDateTime.of(2024, 1, 15, 10, 30, 0))
                .build();

        assertThat(json.write(order))
                .hasJsonPathNumberValue("$.id", 1)
                .hasJsonPathStringValue("$.status", "PENDING")
                .hasJsonPathNumberValue("$.totalAmount", 199.98)
                .hasJsonPathStringValue("$.createdAt", "2024-01-15T10:30:00");
    }

    @Test
    void shouldDeserializeOrderDto() throws IOException {
        String content = """
                {
                  "id": 1,
                  "status": "CONFIRMED",
                  "totalAmount": 299.99,
                  "createdAt": "2024-01-15T10:30:00"
                }
                """;

        OrderDto order = json.parse(content).getObject();

        assertThat(order.getId()).isEqualTo(1L);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(order.getTotalAmount()).isEqualByComparingTo("299.99");
    }
}
```

---

## 5. TestConfiguration vs @MockBean

```java
// @MockBean replaces a bean with a Mockito mock – reloads context between tests
// @TestConfiguration adds beans to the test context without replacing

// Prefer @TestConfiguration for performance when the real implementation can be faked:

@TestConfiguration
class TestEmailConfiguration {

    @Bean
    @Primary
    public EmailService emailService() {
        return new FakeEmailService();  // in-memory implementation
    }

    static class FakeEmailService implements EmailService {
        private final List<SentEmail> sentEmails = new ArrayList<>();

        @Override
        public void sendOrderConfirmation(Order order, String email) {
            sentEmails.add(new SentEmail(email, "Order confirmed: " + order.getId()));
        }

        public List<SentEmail> getSentEmails() {
            return Collections.unmodifiableList(sentEmails);
        }
    }
}

@SpringBootTest
@Import(TestEmailConfiguration.class)
class OrderEmailIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private TestEmailConfiguration.FakeEmailService emailService;

    @Test
    void shouldSendConfirmationEmailOnOrderCreation() {
        orderService.createOrder(new OrderRequest("user@example.com", List.of(new OrderItem(1L, 2))));

        assertThat(emailService.getSentEmails()).hasSize(1);
        assertThat(emailService.getSentEmails().getFirst().getRecipient())
                .isEqualTo("user@example.com");
    }
}
```

---

## 6. WireMock for External Service Mocking

```java
// pom.xml
// <dependency>
//   <groupId>org.springframework.cloud</groupId>
//   <artifactId>spring-cloud-contract-wiremock</artifactId>
//   <scope>test</scope>
// </dependency>

@SpringBootTest
@AutoConfigureWireMock(port = 0)  // random port, sets wiremock.server.port property
class PaymentServiceIntegrationTest {

    @Autowired
    private PaymentService paymentService;

    @Test
    void shouldProcessPaymentSuccessfully() {
        stubFor(post(urlEqualTo("/payment/process"))
                .withHeader("Content-Type", containing("application/json"))
                .withRequestBody(matchingJsonPath("$.amount"))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                                {
                                  "transactionId": "TXN-12345",
                                  "status": "SUCCESS"
                                }
                                """)));

        PaymentResult result = paymentService.charge(
                new PaymentRequest("card-token-abc", new BigDecimal("99.99"), "USD"));

        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getTransactionId()).isEqualTo("TXN-12345");
    }

    @Test
    void shouldHandlePaymentProviderTimeout() {
        stubFor(post(urlEqualTo("/payment/process"))
                .willReturn(aResponse()
                        .withFixedDelay(5000)  // 5 second delay
                        .withStatus(200)));

        assertThatThrownBy(() -> paymentService.charge(
                new PaymentRequest("card-token-abc", new BigDecimal("99.99"), "USD")))
                .isInstanceOf(PaymentTimeoutException.class);
    }

    @Test
    void shouldRetryOnTransientFailure() {
        // Fail twice, succeed on third attempt
        stubFor(post(urlEqualTo("/payment/process"))
                .inScenario("Retry Scenario")
                .whenScenarioStateIs(Scenario.STARTED)
                .willReturn(serverError())
                .willSetStateTo("Second attempt"));

        stubFor(post(urlEqualTo("/payment/process"))
                .inScenario("Retry Scenario")
                .whenScenarioStateIs("Second attempt")
                .willReturn(serverError())
                .willSetStateTo("Third attempt"));

        stubFor(post(urlEqualTo("/payment/process"))
                .inScenario("Retry Scenario")
                .whenScenarioStateIs("Third attempt")
                .willReturn(aResponse().withStatus(200)
                        .withBody("""
                                {"transactionId": "TXN-99999", "status": "SUCCESS"}
                                """)));

        PaymentResult result = paymentService.charge(
                new PaymentRequest("card-token-abc", new BigDecimal("99.99"), "USD"));

        assertThat(result.isSuccess()).isTrue();
        verify(3, postRequestedFor(urlEqualTo("/payment/process")));
    }
}

// application-test.properties
// payment.provider.url=http://localhost:${wiremock.server.port}
```

---

## 7. WireMock with Stub Mapping Files

```java
// src/test/resources/mappings/get-product-1.json
/*
{
  "request": {
    "method": "GET",
    "url": "/catalog/products/1"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "bodyFileName": "product-1-response.json"
  }
}
*/

// src/test/resources/__files/product-1-response.json
/*
{
  "id": 1,
  "name": "Laptop Pro",
  "price": 1299.99,
  "available": true
}
*/

@SpringBootTest
@AutoConfigureWireMock(port = 0, stubs = "classpath:/mappings")
class CatalogClientIntegrationTest {

    @Autowired
    private CatalogClient catalogClient;

    @Test
    void shouldLoadProductFromCatalogService() {
        ProductDetails product = catalogClient.getProduct(1L);

        assertThat(product.getName()).isEqualTo("Laptop Pro");
        assertThat(product.isAvailable()).isTrue();
    }
}
```

---

## 8. Awaitility for Async Assertions

```java
// pom.xml
// <dependency>
//   <groupId>org.awaitility</groupId>
//   <artifactId>awaitility</artifactId>
//   <scope>test</scope>
// </dependency>

@SpringBootTest
class AsyncOrderProcessingTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldProcessOrderAsynchronously() {
        Long orderId = orderService.submitOrder(new OrderRequest(
                "customer@example.com",
                List.of(new OrderItem(1L, 2))));

        // Order is initially PENDING
        assertThat(orderRepository.findById(orderId)).isPresent()
                .hasValueSatisfying(o -> assertThat(o.getStatus()).isEqualTo(OrderStatus.PENDING));

        // Wait for async processing to complete
        await()
                .atMost(10, SECONDS)
                .pollInterval(500, MILLISECONDS)
                .untilAsserted(() -> {
                    Optional<Order> order = orderRepository.findById(orderId);
                    assertThat(order).isPresent();
                    assertThat(order.get().getStatus()).isEqualTo(OrderStatus.CONFIRMED);
                });
    }

    @Test
    void shouldPublishEventAfterOrderConfirmed() {
        TestEventCapture<OrderConfirmedEvent> eventCapture = new TestEventCapture<>();

        Long orderId = orderService.submitOrder(new OrderRequest(
                "customer@example.com",
                List.of(new OrderItem(1L, 1))));

        await()
                .atMost(Duration.ofSeconds(5))
                .until(() -> eventCapture.getEvents().stream()
                        .anyMatch(e -> e.getOrderId().equals(orderId)));

        assertThat(eventCapture.getEvents())
                .anySatisfy(event -> {
                    assertThat(event.getOrderId()).isEqualTo(orderId);
                    assertThat(event.getCustomerEmail()).isEqualTo("customer@example.com");
                });
    }
}

// Reusable test event listener
@Component
@ConditionalOnTest  // custom annotation for test-only beans
class TestEventCapture<T> implements ApplicationListener<T> {

    private final List<T> events = new CopyOnWriteArrayList<>();

    @Override
    public void onApplicationEvent(T event) {
        events.add(event);
    }

    public List<T> getEvents() {
        return Collections.unmodifiableList(events);
    }

    public void clear() {
        events.clear();
    }
}
```

---

## 9. Database State Assertion

```java
// Use AssertJ DB for fluent database assertions
// pom.xml
// <dependency>
//   <groupId>org.assertj</groupId>
//   <artifactId>assertj-db</artifactId>
//   <version>3.0.0</version>
//   <scope>test</scope>
// </dependency>

@SpringBootTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // no auto-rollback
class OrderPersistenceTest {

    @Autowired
    private DataSource dataSource;

    @Autowired
    private OrderService orderService;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    private SoftAssertions softly;

    @BeforeEach
    void setUp() {
        softly = new SoftAssertions();
        jdbcTemplate.execute("DELETE FROM order_items");
        jdbcTemplate.execute("DELETE FROM orders");
    }

    @Test
    void shouldPersistOrderWithItems() {
        OrderRequest request = new OrderRequest("cust-1",
                List.of(new OrderItem(10L, 3), new OrderItem(20L, 1)));

        Long orderId = orderService.createOrder(request);

        // Assert using JDBC directly
        Map<String, Object> order = jdbcTemplate.queryForMap(
                "SELECT * FROM orders WHERE id = ?", orderId);

        softly.assertThat(order.get("customer_id")).isEqualTo("cust-1");
        softly.assertThat(order.get("status")).isEqualTo("PENDING");
        softly.assertThat(order.get("total_amount")).isNotNull();

        List<Map<String, Object>> items = jdbcTemplate.queryForList(
                "SELECT * FROM order_items WHERE order_id = ?", orderId);

        softly.assertThat(items).hasSize(2);
        softly.assertAll();
    }

    @Test
    void shouldNotLeaveOrphanedItemsAfterCancellation() {
        Long orderId = orderService.createOrder(new OrderRequest("cust-1",
                List.of(new OrderItem(10L, 2))));

        orderService.cancelOrder(orderId, "Customer request");

        Integer orphanCount = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM order_items WHERE order_id = ? " +
                        "AND deleted_at IS NULL", Integer.class, orderId);

        assertThat(orphanCount).isZero();
    }
}
```

---

## 10. Transactional Test Rollback

```java
// @Transactional on a test class causes rollback after each test by default
@SpringBootTest
@Transactional
class CustomerRepositoryTransactionalTest {

    @Autowired
    private CustomerRepository customerRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldSaveAndFindCustomer() {
        Customer customer = Customer.builder()
                .email("john@example.com")
                .firstName("John")
                .lastName("Doe")
                .build();

        Customer saved = customerRepository.save(customer);
        entityManager.flush(); // force SQL to run before assertions

        Customer found = customerRepository.findByEmail("john@example.com")
                .orElseThrow();

        assertThat(found.getFirstName()).isEqualTo("John");
        // After test, the transaction is ROLLED BACK
        // "john@example.com" customer will NOT exist in the database
    }

    @Test
    @Rollback(false)  // explicitly commit to test across transactions
    void shouldPersistCustomerAcrossTransactions() {
        Customer customer = customerRepository.save(Customer.builder()
                .email("jane@example.com")
                .firstName("Jane")
                .lastName("Smith")
                .build());
        // This will be committed
    }
}
```

---

## 11. Testing Scheduled Jobs

```java
@SpringBootTest
class OrderCleanupJobTest {

    @Autowired
    private OrderCleanupJob cleanupJob;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private Clock clock;  // injectable clock for time-manipulation

    @Test
    void shouldCancelExpiredPendingOrders() {
        // Create orders with different ages
        Order recentOrder = createPendingOrder(clock.instant().minus(Duration.ofHours(1)));
        Order expiredOrder1 = createPendingOrder(clock.instant().minus(Duration.ofHours(25)));
        Order expiredOrder2 = createPendingOrder(clock.instant().minus(Duration.ofHours(48)));

        // Run the scheduled job manually
        cleanupJob.cancelExpiredOrders();

        assertThat(orderRepository.findById(recentOrder.getId()))
                .hasValueSatisfying(o -> assertThat(o.getStatus()).isEqualTo(OrderStatus.PENDING));

        assertThat(orderRepository.findById(expiredOrder1.getId()))
                .hasValueSatisfying(o -> assertThat(o.getStatus()).isEqualTo(OrderStatus.CANCELLED));

        assertThat(orderRepository.findById(expiredOrder2.getId()))
                .hasValueSatisfying(o -> assertThat(o.getStatus()).isEqualTo(OrderStatus.CANCELLED));
    }

    private Order createPendingOrder(Instant createdAt) {
        Order order = Order.builder()
                .status(OrderStatus.PENDING)
                .createdAt(createdAt)
                .build();
        return orderRepository.save(order);
    }
}

// OrderCleanupJob.java
@Component
@Slf4j
public class OrderCleanupJob {

    private final OrderRepository orderRepository;
    private final Clock clock;

    public OrderCleanupJob(OrderRepository orderRepository, Clock clock) {
        this.orderRepository = orderRepository;
        this.clock = clock;
    }

    @Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
    public void cancelExpiredOrders() {
        Instant cutoff = clock.instant().minus(Duration.ofHours(24));
        int cancelled = orderRepository.cancelOrdersOlderThan(cutoff);
        log.info("Cancelled {} expired pending orders", cancelled);
    }
}

// Test configuration for fixed clock
@TestConfiguration
class ClockTestConfiguration {

    @Bean
    @Primary
    public Clock fixedClock() {
        return Clock.fixed(Instant.parse("2024-06-15T12:00:00Z"), ZoneOffset.UTC);
    }
}
```

---

## 12. Full Integration Test Suite – E-Commerce Example

```java
// Base test class with common setup
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("integration-test")
abstract class BaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("ecommerce_test")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379)
            .withReuse(true);

    @Container
    static GenericContainer<?> wiremock = new GenericContainer<>("wiremock/wiremock:3.3.1")
            .withExposedPorts(8080)
            .withReuse(true);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
        registry.add("payment.provider.url",
                () -> "http://" + wiremock.getHost() + ":" + wiremock.getMappedPort(8080));
    }

    @LocalServerPort
    protected int port;

    @Autowired
    protected JdbcTemplate jdbcTemplate;

    @BeforeEach
    void baseSetUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v1";
        cleanDatabase();
    }

    protected void cleanDatabase() {
        jdbcTemplate.execute("TRUNCATE TABLE order_items, orders, inventory_events, products, customers RESTART IDENTITY CASCADE");
    }
}

// ===== E-Commerce Order Flow Test =====

class ECommerceOrderFlowTest extends BaseIntegrationTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private CustomerRepository customerRepository;

    @BeforeEach
    void setUp() {
        // Seed reference data
        customerRepository.save(Customer.builder()
                .id("CUST-001")
                .email("alice@example.com")
                .firstName("Alice")
                .build());

        productRepository.saveAll(List.of(
                Product.builder().id(1L).name("Laptop").price(new BigDecimal("999.99")).stock(10).build(),
                Product.builder().id(2L).name("Mouse").price(new BigDecimal("29.99")).stock(100).build()
        ));

        // WireMock stub for payment provider
        stubFor(post(urlEqualTo("/payments/charge"))
                .willReturn(okJson("""
                        {"transactionId": "TXN-001", "status": "SUCCESS"}
                        """)));
    }

    @Test
    @DisplayName("Full order lifecycle: browse → add to cart → checkout → payment → confirmation")
    void shouldCompleteFullOrderLifecycle() {
        // 1. Browse products
        List<Map> products = given()
                .queryParam("category", "electronics")
                .get("/products")
                .then().statusCode(200)
                .extract().path("content");

        assertThat(products).isNotEmpty();

        // 2. Create cart
        String cartId = given()
                .body("""
                        {"customerId": "CUST-001"}
                        """)
                .contentType(ContentType.JSON)
                .post("/carts")
                .then().statusCode(201)
                .extract().path("cartId");

        // 3. Add items to cart
        given()
                .body("""
                        {"productId": 1, "quantity": 1}
                        """)
                .contentType(ContentType.JSON)
                .post("/carts/{cartId}/items", cartId)
                .then().statusCode(200);

        given()
                .body("""
                        {"productId": 2, "quantity": 2}
                        """)
                .contentType(ContentType.JSON)
                .post("/carts/{cartId}/items", cartId)
                .then().statusCode(200);

        // 4. View cart
        given()
                .get("/carts/{cartId}", cartId)
                .then().statusCode(200)
                .body("items.size()", equalTo(2))
                .body("totalAmount", equalTo(1059.97F));

        // 5. Checkout
        Integer orderId = given()
                .body("""
                        {
                          "cartId": "%s",
                          "paymentMethod": {"type": "CARD", "token": "tok_visa"},
                          "shippingAddress": {
                            "street": "123 Main St",
                            "city": "Springfield",
                            "zip": "12345"
                          }
                        }
                        """.formatted(cartId))
                .contentType(ContentType.JSON)
                .post("/checkout")
                .then().statusCode(201)
                .body("status", equalTo("CONFIRMED"))
                .extract().path("orderId");

        // 6. Verify order created
        given()
                .get("/orders/{orderId}", orderId)
                .then().statusCode(200)
                .body("id", equalTo(orderId))
                .body("status", equalTo("CONFIRMED"))
                .body("items.size()", equalTo(2));

        // 7. Verify inventory reduced
        given()
                .get("/products/1")
                .then().statusCode(200)
                .body("stock", equalTo(9));  // was 10, bought 1

        // 8. Verify payment was called
        verify(postRequestedFor(urlEqualTo("/payments/charge"))
                .withRequestBody(matchingJsonPath("$.amount", equalTo("1059.97"))));
    }

    @Test
    @DisplayName("Order cancellation restores inventory")
    void shouldRestoreInventoryOnCancellation() {
        // Create and place order
        Integer orderId = placeOrder("CUST-001", List.of(
                Map.of("productId", 1, "quantity", 3)));

        // Verify stock reduced
        assertThat(getProductStock(1L)).isEqualTo(7);  // 10 - 3

        // Cancel order
        given()
                .body("""
                        {"reason": "Changed my mind"}
                        """)
                .contentType(ContentType.JSON)
                .post("/orders/{orderId}/cancel", orderId)
                .then().statusCode(200);

        // Verify stock restored
        await().atMost(5, SECONDS).untilAsserted(() ->
                assertThat(getProductStock(1L)).isEqualTo(10));
    }

    @Test
    @DisplayName("Should reject order when stock insufficient")
    void shouldRejectOrderWhenStockInsufficient() {
        given()
                .body("""
                        {
                          "customerId": "CUST-001",
                          "items": [{"productId": 1, "quantity": 999}]
                        }
                        """)
                .contentType(ContentType.JSON)
                .post("/orders")
                .then().statusCode(409)
                .body("errorCode", equalTo("INSUFFICIENT_STOCK"))
                .body("productId", equalTo(1));
    }

    @Test
    @DisplayName("Should handle payment failure gracefully")
    void shouldHandlePaymentFailure() {
        // Override WireMock to return failure
        stubFor(post(urlEqualTo("/payments/charge"))
                .willReturn(aResponse()
                        .withStatus(402)
                        .withBody("""
                                {"error": "CARD_DECLINED", "message": "Insufficient funds"}
                                """)));

        given()
                .body("""
                        {
                          "customerId": "CUST-001",
                          "items": [{"productId": 1, "quantity": 1}],
                          "paymentMethod": {"type": "CARD", "token": "tok_declined"}
                        }
                        """)
                .contentType(ContentType.JSON)
                .post("/orders")
                .then().statusCode(402)
                .body("errorCode", equalTo("PAYMENT_FAILED"));

        // Verify inventory NOT reduced (payment failed before reservation)
        assertThat(getProductStock(1L)).isEqualTo(10);
    }

    // Helper methods
    private Integer placeOrder(String customerId, List<Map<String, Object>> items) {
        return given()
                .body(Map.of("customerId", customerId, "items", items))
                .contentType(ContentType.JSON)
                .post("/orders")
                .then().statusCode(201)
                .extract().path("orderId");
    }

    private int getProductStock(Long productId) {
        return given()
                .get("/products/{id}", productId)
                .then().statusCode(200)
                .extract().path("stock");
    }
}
```

---

## 13. Test Profiles and Properties

```yaml
# src/test/resources/application-integration-test.yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  flyway:
    enabled: true
    locations: classpath:db/migration,classpath:db/testdata

logging:
  level:
    org.springframework.web: DEBUG
    com.example: DEBUG
    org.hibernate.SQL: DEBUG

# Disable scheduled jobs during tests
app:
  scheduling:
    enabled: false
  async:
    core-pool-size: 2
    max-pool-size: 4
```

---

## 14. Test Data Builders

```java
// Fluent test data builders for readability
public class TestDataBuilders {

    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    public static class OrderBuilder {
        private String customerId = "CUST-001";
        private OrderStatus status = OrderStatus.PENDING;
        private List<OrderItem> items = new ArrayList<>();
        private BigDecimal totalAmount = BigDecimal.ZERO;

        public OrderBuilder forCustomer(String customerId) {
            this.customerId = customerId;
            return this;
        }

        public OrderBuilder withStatus(OrderStatus status) {
            this.status = status;
            return this;
        }

        public OrderBuilder withItem(Long productId, int quantity, BigDecimal price) {
            items.add(new OrderItem(productId, quantity, price));
            totalAmount = totalAmount.add(price.multiply(BigDecimal.valueOf(quantity)));
            return this;
        }

        public Order build() {
            return Order.builder()
                    .customerId(customerId)
                    .status(status)
                    .items(items)
                    .totalAmount(totalAmount)
                    .createdAt(Instant.now())
                    .build();
        }
    }
}

// Usage in tests:
class OrderServiceTest {

    @Test
    void shouldCalculateDiscount() {
        Order order = anOrder()
                .forCustomer("CUST-VIP-001")
                .withItem(1L, 5, new BigDecimal("99.99"))
                .withItem(2L, 2, new BigDecimal("49.99"))
                .withStatus(OrderStatus.PENDING)
                .build();

        BigDecimal discount = discountService.calculate(order);
        assertThat(discount).isGreaterThan(BigDecimal.ZERO);
    }
}
```

---

## Summary

| Feature | Tool | Use Case |
|---|---|---|
| Full context test | `@SpringBootTest` | End-to-end integration tests |
| Controller-only | `@WebMvcTest` | Controller logic, request mapping |
| Repository-only | `@DataJpaTest` | Database queries, JPA mapping |
| Redis-only | `@DataRedisTest` | Cache operations |
| JSON serialization | `@JsonTest` | Jackson configuration |
| HTTP testing | `MockMvc` | Controller tests without real server |
| HTTP testing | `RestAssured` | BDD-style with real server |
| External mocking | `WireMock` | HTTP dependencies |
| Async assertions | `Awaitility` | Event-driven, async flows |
| Fake beans | `@TestConfiguration` | Fast alternatives to real beans |
| Rollback | `@Transactional` on test | Isolate DB state per test |

## Next Part Preview

**Part 084** covers Java Concurrency in Spring Applications — thread safety, CompletableFuture patterns, Virtual Threads, and building a parallel product enrichment pipeline.
