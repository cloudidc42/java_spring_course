# Part 078: Testing Microservices

## Overview

Testing microservices is fundamentally different from testing monoliths. You have multiple independently deployable services communicating over HTTP, messaging systems, and databases. A bug in one service's API contract can silently break downstream services for days. This part covers a complete testing strategy: from fast unit tests to slow end-to-end tests, with contract testing as the key innovation that enables independent deployments.

---

## 1. The Testing Pyramid for Microservices

```
         /\
        /E2E\           Few, slow, brittle
       /------\
      /Contract\        Many, medium speed, catches API breaks
     /----------\
    / Integration\      Many, moderate speed, uses TestContainers
   /--------------\
  /   Unit Tests   \    Thousands, fast, no I/O
 /------------------\
```

### Test Distribution Guidelines

| Type | Count | Duration | Isolation |
|------|-------|----------|-----------|
| Unit | 70% | < 10ms | Full (mocks) |
| Integration | 20% | 100ms - 2s | Partial (real DB, mock HTTP) |
| Contract | 5% | 1-5s | Mock provider/consumer |
| E2E | 5% | 5-60s | None (full stack) |

---

## 2. Project Structure

```
order-service/
├── src/
│   ├── main/java/com/example/order/
│   │   ├── domain/
│   │   │   ├── Order.java
│   │   │   ├── OrderItem.java
│   │   │   └── OrderStatus.java
│   │   ├── service/
│   │   │   ├── OrderService.java
│   │   │   └── OrderPricingService.java
│   │   ├── repository/
│   │   │   └── OrderRepository.java
│   │   ├── web/
│   │   │   └── OrderController.java
│   │   └── client/
│   │       ├── ProductClient.java
│   │       └── PaymentClient.java
│   └── test/java/com/example/order/
│       ├── unit/
│       │   ├── OrderServiceTest.java
│       │   └── OrderPricingServiceTest.java
│       ├── integration/
│       │   ├── OrderRepositoryTest.java
│       │   └── OrderControllerIntegrationTest.java
│       ├── contract/
│       │   ├── OrderContractTest.java  (consumer)
│       │   └── contracts/              (producer)
│       └── e2e/
│           └── OrderE2ETest.java
```

---

## 3. Domain Classes

```java
// src/main/java/com/example/order/domain/Order.java
package com.example.order.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private UUID customerId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount = BigDecimal.ZERO;

    @Column(nullable = false)
    private LocalDateTime createdAt;

    private LocalDateTime confirmedAt;

    @PrePersist
    void prePersist() {
        createdAt = LocalDateTime.now();
    }

    // Domain methods
    public void addItem(OrderItem item) {
        item.setOrder(this);
        items.add(item);
        recalculateTotal();
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException(
                    "Cannot confirm order in status: " + status);
        }
        status = OrderStatus.CONFIRMED;
        confirmedAt = LocalDateTime.now();
    }

    public void cancel() {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new IllegalStateException(
                    "Cannot cancel order in status: " + status);
        }
        status = OrderStatus.CANCELLED;
    }

    private void recalculateTotal() {
        totalAmount = items.stream()
                .map(item -> item.getUnitPrice()
                        .multiply(BigDecimal.valueOf(item.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Getters and setters
    public UUID getId() { return id; }
    public UUID getCustomerId() { return customerId; }
    public void setCustomerId(UUID customerId) { this.customerId = customerId; }
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return items; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getConfirmedAt() { return confirmedAt; }
}
```

```java
// src/main/java/com/example/order/domain/OrderItem.java
package com.example.order.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.util.UUID;

@Entity
@Table(name = "order_items")
public class OrderItem {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    @Column(nullable = false)
    private UUID productId;

    @Column(nullable = false)
    private int quantity;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;

    // Getters and setters
    public UUID getId() { return id; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
    public UUID getProductId() { return productId; }
    public void setProductId(UUID productId) { this.productId = productId; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
}
```

---

## 4. Unit Testing Business Logic

```java
// src/test/java/com/example/order/unit/OrderTest.java
package com.example.order.unit;

import com.example.order.domain.Order;
import com.example.order.domain.OrderItem;
import com.example.order.domain.OrderStatus;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import java.math.BigDecimal;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

@DisplayName("Order Domain Tests")
class OrderTest {

    @Nested
    @DisplayName("Order creation")
    class OrderCreation {

        @Test
        @DisplayName("New order should have PENDING status")
        void newOrderShouldBePending() {
            Order order = createOrder();
            assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
        }

        @Test
        @DisplayName("New order should have zero total")
        void newOrderShouldHaveZeroTotal() {
            Order order = createOrder();
            assertThat(order.getTotalAmount()).isEqualByComparingTo(BigDecimal.ZERO);
        }
    }

    @Nested
    @DisplayName("Adding items")
    class AddingItems {

        @Test
        @DisplayName("Total should be sum of all items")
        void totalShouldBeCorrect() {
            Order order = createOrder();

            OrderItem item1 = createItem("10.00", 2);  // 20.00
            OrderItem item2 = createItem("5.50", 3);   // 16.50

            order.addItem(item1);
            order.addItem(item2);

            assertThat(order.getTotalAmount())
                    .isEqualByComparingTo(new BigDecimal("36.50"));
        }

        @Test
        @DisplayName("Item should be linked to order")
        void itemShouldBeLinkedToOrder() {
            Order order = createOrder();
            OrderItem item = createItem("10.00", 1);

            order.addItem(item);

            assertThat(item.getOrder()).isSameAs(order);
            assertThat(order.getItems()).contains(item);
        }
    }

    @Nested
    @DisplayName("Order confirmation")
    class OrderConfirmation {

        @Test
        @DisplayName("Pending order can be confirmed")
        void pendingOrderCanBeConfirmed() {
            Order order = createOrder();
            order.addItem(createItem("10.00", 1));

            order.confirm();

            assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
            assertThat(order.getConfirmedAt()).isNotNull();
        }

        @ParameterizedTest
        @EnumSource(value = OrderStatus.class,
                names = {"CONFIRMED", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"})
        @DisplayName("Non-pending order cannot be confirmed")
        void nonPendingOrderCannotBeConfirmed(OrderStatus status) {
            Order order = createOrderWithStatus(status);

            assertThatThrownBy(order::confirm)
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("Cannot confirm order");
        }
    }

    @Nested
    @DisplayName("Order cancellation")
    class OrderCancellation {

        @ParameterizedTest
        @EnumSource(value = OrderStatus.class,
                names = {"PENDING", "CONFIRMED", "PROCESSING"})
        @DisplayName("Order can be cancelled before shipping")
        void orderCanBeCancelledBeforeShipping(OrderStatus status) {
            Order order = createOrderWithStatus(status);

            order.cancel();

            assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
        }

        @ParameterizedTest
        @EnumSource(value = OrderStatus.class, names = {"SHIPPED", "DELIVERED"})
        @DisplayName("Shipped/delivered order cannot be cancelled")
        void shippedOrderCannotBeCancelled(OrderStatus status) {
            Order order = createOrderWithStatus(status);

            assertThatThrownBy(order::cancel)
                    .isInstanceOf(IllegalStateException.class)
                    .hasMessageContaining("Cannot cancel order");
        }
    }

    // Test helpers
    private Order createOrder() {
        Order order = new Order();
        order.setCustomerId(UUID.randomUUID());
        return order;
    }

    private Order createOrderWithStatus(OrderStatus status) {
        // Bypass normal flow using reflection for test setup
        Order order = createOrder();
        try {
            var statusField = Order.class.getDeclaredField("status");
            statusField.setAccessible(true);
            statusField.set(order, status);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return order;
    }

    private OrderItem createItem(String price, int quantity) {
        OrderItem item = new OrderItem();
        item.setProductId(UUID.randomUUID());
        item.setUnitPrice(new BigDecimal(price));
        item.setQuantity(quantity);
        return item;
    }
}
```

### Service Unit Test with Mocks

```java
// src/test/java/com/example/order/unit/OrderServiceTest.java
package com.example.order.unit;

import com.example.order.client.ProductClient;
import com.example.order.client.dto.ProductDto;
import com.example.order.domain.Order;
import com.example.order.domain.OrderStatus;
import com.example.order.repository.OrderRepository;
import com.example.order.service.OrderService;
import com.example.order.web.dto.CreateOrderRequest;
import com.example.order.web.dto.OrderItemRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private ProductClient productClient;

    @InjectMocks
    private OrderService orderService;

    private UUID customerId;
    private UUID productId;

    @BeforeEach
    void setUp() {
        customerId = UUID.randomUUID();
        productId = UUID.randomUUID();
    }

    @Test
    void createOrderShouldFetchProductPriceAndSaveOrder() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
                customerId,
                List.of(new OrderItemRequest(productId, 2))
        );

        ProductDto product = new ProductDto(productId, "Widget", new BigDecimal("15.99"), 100);
        when(productClient.getProduct(productId)).thenReturn(product);

        Order savedOrder = new Order();
        when(orderRepository.save(any(Order.class))).thenReturn(savedOrder);

        // When
        Order result = orderService.createOrder(request);

        // Then
        ArgumentCaptor<Order> orderCaptor = ArgumentCaptor.forClass(Order.class);
        verify(orderRepository).save(orderCaptor.capture());

        Order capturedOrder = orderCaptor.getValue();
        assertThat(capturedOrder.getCustomerId()).isEqualTo(customerId);
        assertThat(capturedOrder.getItems()).hasSize(1);
        assertThat(capturedOrder.getItems().get(0).getQuantity()).isEqualTo(2);
        assertThat(capturedOrder.getItems().get(0).getUnitPrice())
                .isEqualByComparingTo("15.99");
        assertThat(capturedOrder.getTotalAmount())
                .isEqualByComparingTo("31.98");

        verify(productClient, times(1)).getProduct(productId);
    }

    @Test
    void createOrderShouldThrowWhenProductNotFound() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
                customerId,
                List.of(new OrderItemRequest(productId, 1))
        );

        when(productClient.getProduct(productId))
                .thenThrow(new ProductNotFoundException(productId));

        // When/Then
        assertThatThrownBy(() -> orderService.createOrder(request))
                .isInstanceOf(ProductNotFoundException.class);

        verify(orderRepository, never()).save(any());
    }

    @Test
    void confirmOrderShouldChangeStatusToConfirmed() {
        // Given
        UUID orderId = UUID.randomUUID();
        Order order = new Order();
        order.setCustomerId(customerId);

        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
        when(orderRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        // When
        Order result = orderService.confirmOrder(orderId, customerId);

        // Then
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(orderRepository).save(order);
    }

    @Test
    void confirmOrderShouldThrowWhenOrderBelongsToOtherCustomer() {
        // Given
        UUID orderId = UUID.randomUUID();
        UUID otherCustomerId = UUID.randomUUID();
        Order order = new Order();
        order.setCustomerId(otherCustomerId);  // Different customer

        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));

        // When/Then
        assertThatThrownBy(() -> orderService.confirmOrder(orderId, customerId))
                .isInstanceOf(OrderAccessDeniedException.class);

        verify(orderRepository, never()).save(any());
    }
}
```

---

## 5. Integration Testing with TestContainers

```java
// src/test/java/com/example/order/integration/OrderRepositoryTest.java
package com.example.order.integration;

import com.example.order.domain.Order;
import com.example.order.domain.OrderItem;
import com.example.order.domain.OrderStatus;
import com.example.order.repository.OrderRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldSaveAndRetrieveOrderWithItems() {
        // Given
        Order order = new Order();
        order.setCustomerId(UUID.randomUUID());

        OrderItem item = new OrderItem();
        item.setProductId(UUID.randomUUID());
        item.setQuantity(3);
        item.setUnitPrice(new BigDecimal("19.99"));
        order.addItem(item);

        // When
        Order saved = orderRepository.save(order);
        Order found = orderRepository.findById(saved.getId()).orElseThrow();

        // Then
        assertThat(found.getItems()).hasSize(1);
        assertThat(found.getTotalAmount()).isEqualByComparingTo("59.97");
        assertThat(found.getStatus()).isEqualTo(OrderStatus.PENDING);
    }

    @Test
    void shouldFindOrdersByCustomerId() {
        // Given
        UUID customerId = UUID.randomUUID();
        UUID otherCustomerId = UUID.randomUUID();

        Order order1 = new Order();
        order1.setCustomerId(customerId);
        Order order2 = new Order();
        order2.setCustomerId(customerId);
        Order order3 = new Order();
        order3.setCustomerId(otherCustomerId);

        orderRepository.saveAll(List.of(order1, order2, order3));

        // When
        List<Order> orders = orderRepository.findByCustomerId(customerId);

        // Then
        assertThat(orders).hasSize(2);
        assertThat(orders).allMatch(o -> o.getCustomerId().equals(customerId));
    }
}
```

### Controller Integration Test

```java
// src/test/java/com/example/order/integration/OrderControllerIntegrationTest.java
package com.example.order.integration;

import com.example.order.client.ProductClient;
import com.example.order.client.dto.ProductDto;
import com.example.order.web.dto.CreateOrderRequest;
import com.example.order.web.dto.OrderItemRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@Testcontainers
class OrderControllerIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private ProductClient productClient;  // Mock external service

    @Test
    void createOrderShouldReturn201WithOrderDetails() throws Exception {
        // Given
        UUID productId = UUID.randomUUID();
        when(productClient.getProduct(productId))
                .thenReturn(new ProductDto(productId, "Widget", new BigDecimal("9.99"), 50));

        CreateOrderRequest request = new CreateOrderRequest(
                UUID.randomUUID(),
                List.of(new OrderItemRequest(productId, 2))
        );

        // When/Then
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value("PENDING"))
                .andExpect(jsonPath("$.totalAmount").value(19.98))
                .andExpect(jsonPath("$.items").isArray())
                .andExpect(jsonPath("$.items[0].quantity").value(2));
    }

    @Test
    void getOrderShouldReturn404ForNonExistentOrder() throws Exception {
        mockMvc.perform(get("/api/orders/{id}", UUID.randomUUID()))
                .andExpect(status().isNotFound());
    }

    @Test
    void createOrderWithInvalidRequestShouldReturn400() throws Exception {
        // Missing required fields
        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors").isArray());
    }
}
```

---

## 6. Contract Testing with Spring Cloud Contract

### Add Dependencies

```xml
<!-- pom.xml for producer (product-service) -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <extensions>true</extensions>
    <configuration>
        <baseClassForTests>
            com.example.product.contract.BaseContractTest
        </baseClassForTests>
    </configuration>
</plugin>
```

### Define Contract (Producer Side)

```groovy
// src/test/resources/contracts/product/should_return_product_by_id.groovy
package contracts.product

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return product details by ID"

    request {
        method GET()
        url "/api/products/550e8400-e29b-41d4-a716-446655440000"
        headers {
            contentType(applicationJson())
        }
    }

    response {
        status 200
        headers {
            contentType(applicationJson())
        }
        body(
            id: "550e8400-e29b-41d4-a716-446655440000",
            name: $(producer(regex("[A-Za-z0-9 ]+")), consumer("Widget Pro")),
            price: $(producer(regex("[0-9]+\\.[0-9]{2}")), consumer("15.99")),
            stockQuantity: $(producer(positiveInt()), consumer(100))
        )
        bodyMatchers {
            jsonPath('$.price', byRegex("[0-9]+\\.[0-9]{2}"))
            jsonPath('$.stockQuantity', byType())
        }
    }
}
```

```groovy
// src/test/resources/contracts/product/should_return_404_for_missing_product.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should return 404 when product not found"

    request {
        method GET()
        url "/api/products/00000000-0000-0000-0000-000000000000"
    }

    response {
        status 404
        headers {
            contentType(applicationJson())
        }
        body(
            error: "Product not found",
            productId: "00000000-0000-0000-0000-000000000000"
        )
    }
}
```

### Base Test Class (Producer)

```java
// src/test/java/com/example/product/contract/BaseContractTest.java
package com.example.product.contract;

import com.example.product.domain.Product;
import com.example.product.service.ProductService;
import com.example.product.web.ProductController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.web.context.WebApplicationContext;

import java.math.BigDecimal;
import java.util.Optional;
import java.util.UUID;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;

@SpringBootTest
public abstract class BaseContractTest {

    @Autowired
    private WebApplicationContext context;

    @MockBean
    private ProductService productService;

    @BeforeEach
    void setup() {
        RestAssuredMockMvc.webAppContextSetup(context);

        UUID existingProductId = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");
        Product product = new Product();
        product.setId(existingProductId);
        product.setName("Widget Pro");
        product.setPrice(new BigDecimal("15.99"));
        product.setStockQuantity(100);

        when(productService.findById(existingProductId))
                .thenReturn(Optional.of(product));

        when(productService.findById(
                UUID.fromString("00000000-0000-0000-0000-000000000000")))
                .thenReturn(Optional.empty());
    }
}
```

### Consumer Side Contract Test

```java
// src/test/java/com/example/order/contract/ProductClientContractTest.java
package com.example.order.contract;

import com.example.order.client.ProductClient;
import com.example.order.client.dto.ProductDto;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import java.math.BigDecimal;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@AutoConfigureStubRunner(
        ids = "com.example:product-service:+:stubs:8090",
        stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class ProductClientContractTest {

    @Autowired
    private ProductClient productClient;

    @Test
    void shouldGetProductById() {
        UUID productId = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");

        ProductDto product = productClient.getProduct(productId);

        assertThat(product).isNotNull();
        assertThat(product.id()).isEqualTo(productId);
        assertThat(product.price()).isGreaterThan(BigDecimal.ZERO);
        assertThat(product.stockQuantity()).isGreaterThan(0);
    }
}
```

---

## 7. Testing Event-Driven Flows

### Kafka Integration Test

```java
// src/test/java/com/example/order/integration/OrderEventTest.java
package com.example.order.integration;

import com.example.order.domain.Order;
import com.example.order.event.OrderCreatedEvent;
import com.example.order.service.OrderService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.time.Duration;
import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class OrderEventTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    static KafkaContainer kafka =
            new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    private OrderService orderService;

    @Autowired
    private ObjectMapper objectMapper;

    private KafkaConsumer<String, String> consumer;

    @BeforeEach
    void setUpConsumer() {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafka.getBootstrapServers());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-" + UUID.randomUUID());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        consumer = new KafkaConsumer<>(props);
        consumer.subscribe(List.of("order.created"));
    }

    @AfterEach
    void tearDown() {
        consumer.close();
    }

    @Test
    void createOrderShouldPublishOrderCreatedEvent() throws Exception {
        // Given
        UUID customerId = UUID.randomUUID();

        // When
        Order order = orderService.createOrderForTest(customerId);

        // Then - wait for Kafka message
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(10));

        assertThat(records.count()).isEqualTo(1);

        String payload = records.iterator().next().value();
        OrderCreatedEvent event = objectMapper.readValue(payload, OrderCreatedEvent.class);

        assertThat(event.orderId()).isEqualTo(order.getId());
        assertThat(event.customerId()).isEqualTo(customerId);
        assertThat(event.timestamp()).isNotNull();
    }
}
```

### Testing with Awaitility

```java
// src/test/java/com/example/order/integration/OrderSagaTest.java
package com.example.order.integration;

import com.example.order.domain.Order;
import com.example.order.domain.OrderStatus;
import com.example.order.repository.OrderRepository;
import com.example.order.service.OrderService;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.time.Duration;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class OrderSagaTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void orderShouldBeConfirmedAfterPaymentSuccess() {
        // Given
        Order order = orderService.createOrderForTest(UUID.randomUUID());

        // When - payment service publishes payment.succeeded event
        // (simulated via direct message publish in test)
        orderService.simulatePaymentSuccess(order.getId());

        // Then - eventually order status changes to CONFIRMED
        Awaitility.await()
                .atMost(Duration.ofSeconds(10))
                .pollInterval(Duration.ofMillis(200))
                .untilAsserted(() -> {
                    Order updated = orderRepository.findById(order.getId()).orElseThrow();
                    assertThat(updated.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
                });
    }
}
```

---

## 8. Chaos Testing

```java
// src/main/java/com/example/chaos/ChaosFilter.java
package com.example.chaos;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Random;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Chaos filter for testing resilience.
 * Enable with: chaos.enabled=true
 */
@Component
@ConditionalOnProperty(name = "chaos.enabled", havingValue = "true")
public class ChaosFilter implements Filter {

    private static final Logger log = LoggerFactory.getLogger(ChaosFilter.class);

    private final ChaosConfig config;
    private final Random random = new Random();
    private final AtomicInteger requestCount = new AtomicInteger(0);

    public ChaosFilter(ChaosConfig config) {
        this.config = config;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        int count = requestCount.incrementAndGet();

        // Random latency injection
        if (config.isLatencyEnabled() && random.nextDouble() < config.getLatencyProbability()) {
            long delay = config.getMinLatencyMs() +
                    (long) (random.nextDouble() * (config.getMaxLatencyMs() - config.getMinLatencyMs()));
            log.debug("Injecting {}ms latency (chaos)", delay);
            try {
                Thread.sleep(delay);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        // Random error injection
        if (config.isErrorEnabled() && random.nextDouble() < config.getErrorProbability()) {
            log.debug("Injecting 503 error (chaos)");
            ((HttpServletResponse) response).sendError(503, "Chaos: Service Unavailable");
            return;
        }

        // Periodic error (every N requests)
        if (config.getFailEveryN() > 0 && count % config.getFailEveryN() == 0) {
            log.debug("Injecting periodic 500 error (chaos, request #{})", count);
            ((HttpServletResponse) response).sendError(500, "Chaos: Periodic Error");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

```java
// src/main/java/com/example/chaos/ChaosConfig.java
package com.example.chaos;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "chaos")
public class ChaosConfig {
    private boolean enabled = false;
    private boolean latencyEnabled = false;
    private double latencyProbability = 0.1;
    private long minLatencyMs = 100;
    private long maxLatencyMs = 2000;
    private boolean errorEnabled = false;
    private double errorProbability = 0.05;
    private int failEveryN = 0;

    // Getters and setters
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public boolean isLatencyEnabled() { return latencyEnabled; }
    public void setLatencyEnabled(boolean latencyEnabled) { this.latencyEnabled = latencyEnabled; }
    public double getLatencyProbability() { return latencyProbability; }
    public void setLatencyProbability(double latencyProbability) { this.latencyProbability = latencyProbability; }
    public long getMinLatencyMs() { return minLatencyMs; }
    public void setMinLatencyMs(long minLatencyMs) { this.minLatencyMs = minLatencyMs; }
    public long getMaxLatencyMs() { return maxLatencyMs; }
    public void setMaxLatencyMs(long maxLatencyMs) { this.maxLatencyMs = maxLatencyMs; }
    public boolean isErrorEnabled() { return errorEnabled; }
    public void setErrorEnabled(boolean errorEnabled) { this.errorEnabled = errorEnabled; }
    public double getErrorProbability() { return errorProbability; }
    public void setErrorProbability(double errorProbability) { this.errorProbability = errorProbability; }
    public int getFailEveryN() { return failEveryN; }
    public void setFailEveryN(int failEveryN) { this.failEveryN = failEveryN; }
}
```

### Resilience Test

```java
// src/test/java/com/example/order/chaos/ResilienceTest.java
package com.example.order.chaos;

import com.example.order.client.ProductClient;
import com.example.order.client.dto.ProductDto;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.TestPropertySource;

import java.math.BigDecimal;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

@SpringBootTest
@TestPropertySource(properties = {
        "resilience4j.circuitbreaker.instances.productService.failure-rate-threshold=50",
        "resilience4j.circuitbreaker.instances.productService.minimum-number-of-calls=5",
        "resilience4j.retry.instances.productService.max-attempts=3"
})
class ResilienceTest {

    @Autowired
    private ProductClient productClient;

    @MockBean
    private ProductServiceBackend productServiceBackend;  // Underlying HTTP client

    @Test
    void circuitBreakerShouldOpenAfterFailures() {
        UUID productId = UUID.randomUUID();
        AtomicInteger callCount = new AtomicInteger(0);

        // First 5 calls fail
        when(productServiceBackend.getProduct(productId))
                .thenAnswer(inv -> {
                    if (callCount.incrementAndGet() <= 5) {
                        throw new RuntimeException("Service unavailable");
                    }
                    return new ProductDto(productId, "Widget", BigDecimal.TEN, 10);
                });

        // After 5 failures, circuit should be open
        // and subsequent calls should fail fast without calling the backend
        int callsBefore = callCount.get();
        for (int i = 0; i < 10; i++) {
            try {
                productClient.getProduct(productId);
            } catch (Exception ignored) {}
        }

        // Circuit breaker should have prevented many calls to backend
        assertThat(callCount.get()).isLessThan(callsBefore + 10);
    }
}
```

---

## 9. Performance Testing

```java
// src/test/java/com/example/order/performance/OrderServicePerformanceTest.java
package com.example.order.performance;

import com.example.order.service.OrderService;
import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Tag("performance")
class OrderServicePerformanceTest {

    @Autowired
    private OrderService orderService;

    @Test
    void getOrderShouldCompleteWithin100ms() {
        // Warm up
        UUID orderId = orderService.createOrderForTest(UUID.randomUUID()).getId();

        // Measure
        Instant start = Instant.now();
        for (int i = 0; i < 100; i++) {
            orderService.findById(orderId);
        }
        Duration elapsed = Duration.between(start, Instant.now());

        // 100 requests should take less than 10 seconds (100ms average)
        assertThat(elapsed.toMillis() / 100).isLessThan(100);
    }

    @Test
    void shouldHandleConcurrentOrderCreation() throws InterruptedException {
        int threadCount = 20;
        int ordersPerThread = 10;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(threadCount);
        List<Exception> errors = new CopyOnWriteArrayList<>();

        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    for (int j = 0; j < ordersPerThread; j++) {
                        orderService.createOrderForTest(UUID.randomUUID());
                    }
                } catch (Exception e) {
                    errors.add(e);
                } finally {
                    latch.countDown();
                }
            });
        }

        boolean completed = latch.await(30, TimeUnit.SECONDS);
        executor.shutdown();

        assertThat(completed).isTrue();
        assertThat(errors).isEmpty();
    }
}
```

---

## 10. API Backward Compatibility Testing

```java
// src/test/java/com/example/order/compatibility/ApiCompatibilityTest.java
package com.example.order.compatibility;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Tests that ensure existing API contracts are not broken.
 * These tests reflect what consumers currently depend on.
 */
@SpringBootTest
@AutoConfigureMockMvc
class ApiCompatibilityTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void orderResponseShouldContainRequiredFields() throws Exception {
        // Fields that existing consumers depend on
        MvcResult result = mockMvc.perform(get("/api/orders/{id}", "existing-order-id"))
                .andExpect(status().isOk())
                .andReturn();

        JsonNode response = objectMapper.readTree(result.getResponse().getContentAsString());

        // These fields MUST exist and not be removed
        assertThat(response.has("id")).isTrue();
        assertThat(response.has("status")).isTrue();
        assertThat(response.has("totalAmount")).isTrue();
        assertThat(response.has("createdAt")).isTrue();
        assertThat(response.has("items")).isTrue();

        // items array must have these fields
        if (response.get("items").size() > 0) {
            JsonNode item = response.get("items").get(0);
            assertThat(item.has("productId")).isTrue();
            assertThat(item.has("quantity")).isTrue();
            assertThat(item.has("unitPrice")).isTrue();
        }
    }

    @Test
    void orderStatusValuesShouldBeBackwardCompatible() throws Exception {
        // The status field should only ever return these known values
        String[] allowedStatuses = {
                "PENDING", "CONFIRMED", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"
        };

        // Verify the enum values are documented in OpenAPI spec
        MvcResult apiDocs = mockMvc.perform(get("/v3/api-docs"))
                .andExpect(status().isOk())
                .andReturn();

        String docs = apiDocs.getResponse().getContentAsString();
        for (String status : allowedStatuses) {
            assertThat(docs).contains(status);
        }
    }
}
```

---

## 11. Test Data Management

```java
// src/test/java/com/example/order/testdata/OrderTestDataFactory.java
package com.example.order.testdata;

import com.example.order.domain.Order;
import com.example.order.domain.OrderItem;
import com.example.order.domain.OrderStatus;
import com.example.order.repository.OrderRepository;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.UUID;

/**
 * Factory for creating test data consistently across tests.
 * Implements Builder pattern for flexible test setup.
 */
@Component
public class OrderTestDataFactory {

    private final OrderRepository orderRepository;

    public OrderTestDataFactory(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    public class OrderBuilder {
        private UUID customerId = UUID.randomUUID();
        private OrderStatus status = OrderStatus.PENDING;
        private boolean withItems = false;
        private int itemCount = 1;

        public OrderBuilder forCustomer(UUID customerId) {
            this.customerId = customerId;
            return this;
        }

        public OrderBuilder withStatus(OrderStatus status) {
            this.status = status;
            return this;
        }

        public OrderBuilder withItems(int count) {
            this.withItems = true;
            this.itemCount = count;
            return this;
        }

        public Order build() {
            Order order = new Order();
            order.setCustomerId(customerId);

            if (withItems) {
                for (int i = 0; i < itemCount; i++) {
                    OrderItem item = new OrderItem();
                    item.setProductId(UUID.randomUUID());
                    item.setQuantity(i + 1);
                    item.setUnitPrice(new BigDecimal("9.99"));
                    order.addItem(item);
                }
            }

            // Set status via reflection for test setup
            setStatus(order, status);

            return order;
        }

        public Order persist() {
            return orderRepository.save(build());
        }

        private void setStatus(Order order, OrderStatus status) {
            try {
                var field = Order.class.getDeclaredField("status");
                field.setAccessible(true);
                field.set(order, status);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```

---

## 12. Real Example: Full Testing Strategy for Order Management

### Test Configuration

```java
// src/test/java/com/example/order/TestOrderServiceApplication.java
package com.example.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

@TestConfiguration(proxyBeanMethods = false)
public class TestOrderServiceApplication {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>(DockerImageName.parse("postgres:16-alpine"));
    }

    @Bean
    @ServiceConnection
    KafkaContainer kafkaContainer() {
        return new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    }

    public static void main(String[] args) {
        SpringApplication
                .from(OrderServiceApplication::main)
                .with(TestOrderServiceApplication.class)
                .run(args);
    }
}
```

### Complete Order Flow Test

```java
// src/test/java/com/example/order/e2e/OrderFlowTest.java
package com.example.order.e2e;

import com.example.order.client.ProductClient;
import com.example.order.client.dto.ProductDto;
import com.example.order.domain.OrderStatus;
import com.example.order.testdata.OrderTestDataFactory;
import com.example.order.web.dto.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
class OrderFlowTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private ProductClient productClient;

    @Autowired
    private OrderTestDataFactory testDataFactory;

    @Test
    void completeOrderFlowFromCreationToConfirmation() throws Exception {
        // Setup
        UUID customerId = UUID.randomUUID();
        UUID productId = UUID.randomUUID();

        when(productClient.getProduct(productId))
                .thenReturn(new ProductDto(productId, "Widget", new BigDecimal("25.00"), 50));

        // Step 1: Create order
        CreateOrderRequest createRequest = new CreateOrderRequest(
                customerId,
                List.of(new OrderItemRequest(productId, 2))
        );

        MvcResult createResult = mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(createRequest)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value("PENDING"))
                .andExpect(jsonPath("$.totalAmount").value(50.0))
                .andReturn();

        String orderId = objectMapper.readTree(
                createResult.getResponse().getContentAsString()).get("id").asText();

        // Step 2: Confirm order
        mockMvc.perform(post("/api/orders/{id}/confirm", orderId)
                        .header("X-Customer-Id", customerId.toString()))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("CONFIRMED"));

        // Step 3: Verify order state
        mockMvc.perform(get("/api/orders/{id}", orderId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("CONFIRMED"))
                .andExpect(jsonPath("$.confirmedAt").isNotEmpty());

        // Step 4: Cancel should fail (already confirmed, then simulate shipping)
        // ... in a real test, you'd simulate the saga progressing
    }
}
```

---

## Summary

| Testing Layer | Tool/Framework | What It Tests | Speed |
|---------------|---------------|---------------|-------|
| Unit | JUnit 5 + Mockito | Business logic, domain rules | < 10ms |
| Repository | @DataJpaTest + TestContainers | SQL queries, JPA mappings | 200ms |
| Controller | MockMvc + MockBean | HTTP handling, validation | 50ms |
| Contract | Spring Cloud Contract | API compatibility | 1-2s |
| Event | KafkaContainer + Awaitility | Messaging flows | 2-5s |
| Performance | JUnit 5 + CompletableFuture | Throughput, latency | Variable |
| Chaos | Custom filter | Resilience, circuit breakers | Variable |
| E2E | Full SpringBoot + All containers | Complete flows | 5-30s |

---

## Next Part Preview

**Part 079: Complete Observability Stack** - We'll set up a full Grafana observability stack with Prometheus for metrics, Loki for logs, and Tempo for distributed traces, with OpenTelemetry auto-instrumentation.
