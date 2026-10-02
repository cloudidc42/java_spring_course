# Part 059: Microservices Design Patterns

## Overview

This part covers the essential design patterns for building robust, scalable microservices. We'll explore how large companies like Amazon, Netflix, and Uber solve distributed systems challenges using proven patterns. By the end, you'll understand how to implement these patterns in real Spring Boot applications.

---

## 1. Saga Pattern

The Saga pattern manages distributed transactions across multiple microservices. Instead of a single ACID transaction, you break it into a sequence of local transactions with compensating actions on failure.

### 1.1 Choreography-based Saga

Each service publishes events and reacts to events from other services — no central coordinator.

```java
// Order Service - publishes OrderCreated event
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository,
                        ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    public Order createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .totalAmount(request.getTotalAmount())
            .status(OrderStatus.PENDING)
            .build();

        order = orderRepository.save(order);

        // Publish domain event - other services react to this
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        ));

        return order;
    }

    public void approveOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.setStatus(OrderStatus.APPROVED);
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderApprovedEvent(orderId));
    }

    public void rejectOrder(Long orderId, String reason) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        order.setStatus(OrderStatus.REJECTED);
        order.setRejectionReason(reason);
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderRejectedEvent(orderId, reason));
    }
}

// Payment Service - reacts to OrderCreated, publishes PaymentProcessed/Failed
@Service
public class PaymentEventHandler {

    private final PaymentService paymentService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order.created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            Payment payment = paymentService.processPayment(
                event.getOrderId(),
                event.getCustomerId(),
                event.getTotalAmount()
            );

            kafkaTemplate.send("payment.processed",
                new PaymentProcessedEvent(event.getOrderId(), payment.getId()));

        } catch (InsufficientFundsException e) {
            kafkaTemplate.send("payment.failed",
                new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }

    // Compensating transaction: reverse payment if inventory reservation fails
    @KafkaListener(topics = "inventory.reservation.failed")
    public void handleInventoryFailed(InventoryReservationFailedEvent event) {
        paymentService.refundPayment(event.getOrderId());
        kafkaTemplate.send("payment.refunded",
            new PaymentRefundedEvent(event.getOrderId()));
    }
}

// Inventory Service - reacts to PaymentProcessed
@Service
public class InventoryEventHandler {

    private final InventoryService inventoryService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "payment.processed")
    public void handlePaymentProcessed(PaymentProcessedEvent event) {
        try {
            inventoryService.reserveItems(event.getOrderId());
            kafkaTemplate.send("inventory.reserved",
                new InventoryReservedEvent(event.getOrderId()));
        } catch (InsufficientStockException e) {
            kafkaTemplate.send("inventory.reservation.failed",
                new InventoryReservationFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }
}
```

### 1.2 Orchestration-based Saga

A central orchestrator directs each step and handles failures.

```java
// Saga Orchestrator
@Service
@Transactional
public class OrderSagaOrchestrator {

    private final SagaInstanceRepository sagaRepository;
    private final PaymentServiceClient paymentClient;
    private final InventoryServiceClient inventoryClient;
    private final ShippingServiceClient shippingClient;

    public void startOrderSaga(Long orderId) {
        SagaInstance saga = SagaInstance.builder()
            .orderId(orderId)
            .currentStep(SagaStep.PAYMENT)
            .status(SagaStatus.STARTED)
            .build();

        sagaRepository.save(saga);
        processPaymentStep(saga);
    }

    private void processPaymentStep(SagaInstance saga) {
        try {
            PaymentResult result = paymentClient.processPayment(saga.getOrderId());
            saga.setPaymentId(result.getPaymentId());
            saga.setCurrentStep(SagaStep.INVENTORY);
            sagaRepository.save(saga);
            processInventoryStep(saga);
        } catch (Exception e) {
            handlePaymentFailure(saga, e.getMessage());
        }
    }

    private void processInventoryStep(SagaInstance saga) {
        try {
            inventoryClient.reserveItems(saga.getOrderId());
            saga.setCurrentStep(SagaStep.SHIPPING);
            sagaRepository.save(saga);
            processShippingStep(saga);
        } catch (Exception e) {
            compensatePayment(saga);
            handleInventoryFailure(saga, e.getMessage());
        }
    }

    private void processShippingStep(SagaInstance saga) {
        try {
            shippingClient.scheduleShipment(saga.getOrderId());
            saga.setStatus(SagaStatus.COMPLETED);
            sagaRepository.save(saga);
            notifySuccess(saga);
        } catch (Exception e) {
            compensateInventory(saga);
            compensatePayment(saga);
            handleShippingFailure(saga, e.getMessage());
        }
    }

    // Compensating transactions
    private void compensatePayment(SagaInstance saga) {
        if (saga.getPaymentId() != null) {
            paymentClient.refundPayment(saga.getPaymentId());
        }
    }

    private void compensateInventory(SagaInstance saga) {
        inventoryClient.releaseReservation(saga.getOrderId());
    }

    private void handlePaymentFailure(SagaInstance saga, String reason) {
        saga.setStatus(SagaStatus.FAILED);
        saga.setFailureReason(reason);
        sagaRepository.save(saga);
    }

    private void handleInventoryFailure(SagaInstance saga, String reason) {
        saga.setStatus(SagaStatus.COMPENSATING);
        saga.setFailureReason(reason);
        sagaRepository.save(saga);
    }

    private void handleShippingFailure(SagaInstance saga, String reason) {
        saga.setStatus(SagaStatus.COMPENSATING);
        saga.setFailureReason(reason);
        sagaRepository.save(saga);
    }

    private void notifySuccess(SagaInstance saga) {
        // Notify order service that saga completed successfully
    }
}

// Saga domain model
@Entity
@Table(name = "saga_instances")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SagaInstance {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long orderId;
    private Long paymentId;

    @Enumerated(EnumType.STRING)
    private SagaStep currentStep;

    @Enumerated(EnumType.STRING)
    private SagaStatus status;

    private String failureReason;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

public enum SagaStep { PAYMENT, INVENTORY, SHIPPING }
public enum SagaStatus { STARTED, COMPLETED, FAILED, COMPENSATING }
```

---

## 2. Outbox Pattern

The Outbox Pattern solves the "dual write" problem: writing to a database and publishing a message must be atomic.

```java
// Outbox table entity
@Entity
@Table(name = "outbox_events")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String aggregateType;   // "Order"
    private String aggregateId;     // order ID
    private String eventType;       // "OrderCreated"

    @Column(columnDefinition = "TEXT")
    private String payload;         // JSON payload

    @Enumerated(EnumType.STRING)
    private OutboxStatus status;    // PENDING, SENT, FAILED

    private Integer retryCount;
    private LocalDateTime createdAt;
    private LocalDateTime processedAt;
}

// Order service writes to both order table and outbox in ONE transaction
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxEventRepository outboxRepository;
    private final ObjectMapper objectMapper;

    public Order createOrder(CreateOrderRequest request) throws JsonProcessingException {
        // 1. Save order
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .totalAmount(request.getTotalAmount())
            .status(OrderStatus.PENDING)
            .build();
        order = orderRepository.save(order);

        // 2. Write to outbox (same transaction!)
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount(),
            LocalDateTime.now()
        );

        OutboxEvent outboxEvent = OutboxEvent.builder()
            .aggregateType("Order")
            .aggregateId(order.getId().toString())
            .eventType("OrderCreated")
            .payload(objectMapper.writeValueAsString(event))
            .status(OutboxStatus.PENDING)
            .retryCount(0)
            .createdAt(LocalDateTime.now())
            .build();

        outboxRepository.save(outboxEvent);

        // Both saved atomically - if either fails, both roll back
        return order;
    }
}

// Outbox Relay - polls outbox and publishes to Kafka
@Component
@Slf4j
public class OutboxEventRelay {

    private final OutboxEventRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 1000)  // Poll every second
    @Transactional
    public void processOutboxEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository
            .findTop100ByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING);

        for (OutboxEvent event : pendingEvents) {
            try {
                String topic = resolveTopicName(event.getEventType());
                kafkaTemplate.send(topic, event.getAggregateId(), event.getPayload())
                    .get(5, TimeUnit.SECONDS);  // Wait for ack

                event.setStatus(OutboxStatus.SENT);
                event.setProcessedAt(LocalDateTime.now());
                outboxRepository.save(event);

            } catch (Exception e) {
                log.error("Failed to publish outbox event {}", event.getId(), e);
                event.setRetryCount(event.getRetryCount() + 1);

                if (event.getRetryCount() >= 3) {
                    event.setStatus(OutboxStatus.FAILED);
                    log.error("Outbox event {} exceeded max retries, marked as FAILED", event.getId());
                }
                outboxRepository.save(event);
            }
        }
    }

    private String resolveTopicName(String eventType) {
        return switch (eventType) {
            case "OrderCreated" -> "order.created";
            case "OrderApproved" -> "order.approved";
            case "OrderRejected" -> "order.rejected";
            default -> throw new IllegalArgumentException("Unknown event type: " + eventType);
        };
    }
}

// Outbox repository
@Repository
public interface OutboxEventRepository extends JpaRepository<OutboxEvent, Long> {
    List<OutboxEvent> findTop100ByStatusOrderByCreatedAtAsc(OutboxStatus status);
    List<OutboxEvent> findByStatusAndRetryCountLessThan(OutboxStatus status, int maxRetries);
}
```

---

## 3. Inbox Pattern (Idempotent Consumer)

The Inbox Pattern prevents duplicate message processing by tracking processed message IDs.

```java
// Inbox table
@Entity
@Table(name = "inbox_messages")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class InboxMessage {

    @Id
    private String messageId;       // Kafka message ID or event ID

    private String topic;
    private String eventType;

    @Column(columnDefinition = "TEXT")
    private String payload;

    @Enumerated(EnumType.STRING)
    private InboxStatus status;     // PROCESSING, PROCESSED, FAILED

    private LocalDateTime receivedAt;
    private LocalDateTime processedAt;
}

// Idempotent event consumer
@Service
@Slf4j
public class IdempotentOrderEventConsumer {

    private final InboxMessageRepository inboxRepository;
    private final OrderService orderService;
    private final TransactionTemplate transactionTemplate;

    @KafkaListener(topics = "payment.processed", groupId = "order-service")
    public void handlePaymentProcessed(
            @Payload String payload,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header("eventId") String eventId) {

        // Idempotency check - has this message been processed before?
        if (inboxRepository.existsById(eventId)) {
            log.info("Duplicate message detected, skipping: {}", eventId);
            return;
        }

        transactionTemplate.execute(status -> {
            // Mark as processing
            InboxMessage message = InboxMessage.builder()
                .messageId(eventId)
                .topic(topic)
                .eventType("PaymentProcessed")
                .payload(payload)
                .status(InboxStatus.PROCESSING)
                .receivedAt(LocalDateTime.now())
                .build();
            inboxRepository.save(message);

            try {
                // Process the event
                PaymentProcessedEvent event = parseEvent(payload, PaymentProcessedEvent.class);
                orderService.approveOrder(event.getOrderId());

                // Mark as processed
                message.setStatus(InboxStatus.PROCESSED);
                message.setProcessedAt(LocalDateTime.now());
                inboxRepository.save(message);

            } catch (Exception e) {
                message.setStatus(InboxStatus.FAILED);
                inboxRepository.save(message);
                log.error("Failed to process message {}", eventId, e);
                status.setRollbackOnly();
            }

            return null;
        });
    }

    private <T> T parseEvent(String payload, Class<T> type) {
        try {
            return new ObjectMapper().readValue(payload, type);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to parse event payload", e);
        }
    }
}
```

---

## 4. API Composition Pattern

Aggregate data from multiple services to satisfy a query.

```java
// API Composition for Order details (order + customer + payment + shipping)
@Service
@Slf4j
public class OrderDetailsCompositor {

    private final OrderServiceClient orderClient;
    private final CustomerServiceClient customerClient;
    private final PaymentServiceClient paymentClient;
    private final ShippingServiceClient shippingClient;

    // Sequential composition (simple but slow)
    public OrderDetailsDto getOrderDetailsSequential(Long orderId) {
        OrderDto order = orderClient.getOrder(orderId);
        CustomerDto customer = customerClient.getCustomer(order.getCustomerId());
        PaymentDto payment = paymentClient.getPaymentByOrder(orderId);
        ShippingDto shipping = shippingClient.getShippingByOrder(orderId);

        return OrderDetailsDto.builder()
            .order(order)
            .customer(customer)
            .payment(payment)
            .shipping(shipping)
            .build();
    }

    // Parallel composition (faster - concurrent calls)
    public OrderDetailsDto getOrderDetailsParallel(Long orderId) {
        OrderDto order = orderClient.getOrder(orderId);

        // Fetch customer, payment, and shipping in parallel
        CompletableFuture<CustomerDto> customerFuture =
            CompletableFuture.supplyAsync(() -> customerClient.getCustomer(order.getCustomerId()));

        CompletableFuture<PaymentDto> paymentFuture =
            CompletableFuture.supplyAsync(() -> paymentClient.getPaymentByOrder(orderId));

        CompletableFuture<ShippingDto> shippingFuture =
            CompletableFuture.supplyAsync(() -> shippingClient.getShippingByOrder(orderId));

        try {
            CompletableFuture.allOf(customerFuture, paymentFuture, shippingFuture).join();

            return OrderDetailsDto.builder()
                .order(order)
                .customer(customerFuture.get())
                .payment(paymentFuture.get())
                .shipping(shippingFuture.get())
                .build();

        } catch (Exception e) {
            log.error("Error composing order details for order {}", orderId, e);
            // Return partial result with available data
            return buildPartialResult(order, customerFuture, paymentFuture, shippingFuture);
        }
    }

    private OrderDetailsDto buildPartialResult(
            OrderDto order,
            CompletableFuture<CustomerDto> customerFuture,
            CompletableFuture<PaymentDto> paymentFuture,
            CompletableFuture<ShippingDto> shippingFuture) {

        return OrderDetailsDto.builder()
            .order(order)
            .customer(getOrNull(customerFuture))
            .payment(getOrNull(paymentFuture))
            .shipping(getOrNull(shippingFuture))
            .build();
    }

    private <T> T getOrNull(CompletableFuture<T> future) {
        try {
            return future.isDone() ? future.get() : null;
        } catch (Exception e) {
            return null;
        }
    }
}

// REST client with resilience
@Component
@Slf4j
public class OrderServiceClient {

    private final RestTemplate restTemplate;
    private final String orderServiceUrl;

    public OrderServiceClient(RestTemplate restTemplate,
                               @Value("${services.order.url}") String orderServiceUrl) {
        this.restTemplate = restTemplate;
        this.orderServiceUrl = orderServiceUrl;
    }

    @CircuitBreaker(name = "orderService", fallbackMethod = "getOrderFallback")
    @Retry(name = "orderService")
    public OrderDto getOrder(Long orderId) {
        return restTemplate.getForObject(
            orderServiceUrl + "/orders/{id}",
            OrderDto.class,
            orderId
        );
    }

    public OrderDto getOrderFallback(Long orderId, Exception ex) {
        log.warn("Order service unavailable, returning cached/default for order {}", orderId);
        return OrderDto.builder()
            .id(orderId)
            .status("UNKNOWN")
            .build();
    }
}
```

---

## 5. CQRS Pattern

Separate the read and write models for better scalability.

```java
// ===== COMMAND SIDE (Write Model) =====

// Commands
public record CreateProductCommand(
    String name,
    String description,
    BigDecimal price,
    int stockQuantity
) {}

public record UpdatePriceCommand(String productId, BigDecimal newPrice) {}
public record ReserveStockCommand(String productId, int quantity) {}

// Write model entity - optimized for consistency
@Entity
@Table(name = "products")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String name;

    @Column(length = 2000)
    private String description;

    private BigDecimal price;
    private int stockQuantity;
    private int reservedQuantity;

    @Version
    private Long version;   // Optimistic locking

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    public void reserveStock(int quantity) {
        if (availableQuantity() < quantity) {
            throw new InsufficientStockException(
                "Available: " + availableQuantity() + ", Requested: " + quantity);
        }
        this.reservedQuantity += quantity;
    }

    public int availableQuantity() {
        return stockQuantity - reservedQuantity;
    }
}

// Command handler
@Service
@Transactional
public class ProductCommandHandler {

    private final ProductRepository productRepository;
    private final ProductEventPublisher eventPublisher;

    public String handle(CreateProductCommand command) {
        Product product = Product.builder()
            .name(command.name())
            .description(command.description())
            .price(command.price())
            .stockQuantity(command.stockQuantity())
            .reservedQuantity(0)
            .build();

        product = productRepository.save(product);
        eventPublisher.publish(new ProductCreatedEvent(product));
        return product.getId();
    }

    public void handle(UpdatePriceCommand command) {
        Product product = productRepository.findById(command.productId())
            .orElseThrow(() -> new ProductNotFoundException(command.productId()));

        BigDecimal oldPrice = product.getPrice();
        product.setPrice(command.newPrice());
        productRepository.save(product);

        eventPublisher.publish(new ProductPriceUpdatedEvent(
            product.getId(), oldPrice, command.newPrice()));
    }
}

// ===== QUERY SIDE (Read Model) =====

// Denormalized read model - optimized for queries
@Document(collection = "product_views")  // MongoDB for read model
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductView {

    @Id
    private String id;

    private String name;
    private String description;
    private BigDecimal price;
    private int availableStock;
    private String categoryName;
    private double averageRating;
    private int reviewCount;
    private List<String> tags;
    private LocalDateTime updatedAt;
}

// Query handler
@Service
public class ProductQueryHandler {

    private final ProductViewRepository productViewRepository;

    public ProductView getProduct(String productId) {
        return productViewRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    public Page<ProductView> searchProducts(ProductSearchQuery query, Pageable pageable) {
        if (query.getCategoryId() != null && query.getMinPrice() != null) {
            return productViewRepository.findByCategoryIdAndPriceBetween(
                query.getCategoryId(),
                query.getMinPrice(),
                query.getMaxPrice(),
                pageable
            );
        }
        if (query.getSearchText() != null) {
            return productViewRepository.findByNameContainingIgnoreCase(
                query.getSearchText(), pageable);
        }
        return productViewRepository.findAll(pageable);
    }
}

// Read model updater - listens to events and updates MongoDB
@Component
@Slf4j
public class ProductViewUpdater {

    private final ProductViewRepository productViewRepository;
    private final CategoryServiceClient categoryClient;

    @EventListener
    public void on(ProductCreatedEvent event) {
        String categoryName = categoryClient.getCategoryName(event.getCategoryId());

        ProductView view = ProductView.builder()
            .id(event.getProductId())
            .name(event.getName())
            .description(event.getDescription())
            .price(event.getPrice())
            .availableStock(event.getStockQuantity())
            .categoryName(categoryName)
            .averageRating(0.0)
            .reviewCount(0)
            .tags(event.getTags())
            .updatedAt(LocalDateTime.now())
            .build();

        productViewRepository.save(view);
    }

    @EventListener
    public void on(ProductPriceUpdatedEvent event) {
        productViewRepository.findById(event.getProductId()).ifPresent(view -> {
            view.setPrice(event.getNewPrice());
            view.setUpdatedAt(LocalDateTime.now());
            productViewRepository.save(view);
        });
    }
}
```

---

## 6. Database per Service Pattern

```java
// Each service has its own database configuration
// Order Service - uses PostgreSQL
@Configuration
public class OrderDatabaseConfig {

    @Bean
    @ConfigurationProperties(prefix = "order.datasource")
    public DataSource orderDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean orderEntityManagerFactory(
            @Qualifier("orderDataSource") DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean factory =
            new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setPackagesToScan("com.example.order.domain");
        factory.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        return factory;
    }
}

// application.yml for multi-database setup
// ---
// order:
//   datasource:
//     url: jdbc:postgresql://order-db:5432/orders
//     username: order_user
//     password: ${ORDER_DB_PASSWORD}
//
// inventory:
//   datasource:
//     url: jdbc:postgresql://inventory-db:5432/inventory
//     username: inventory_user
//     password: ${INVENTORY_DB_PASSWORD}
```

---

## 7. Strangler Fig Pattern

Gradually replace a monolith by routing traffic incrementally.

```java
// Feature flag-based routing to new microservice
@Component
@Slf4j
public class StranglerFigRouter {

    private final FeatureFlagService featureFlags;
    private final LegacyOrderClient legacyClient;
    private final NewOrderServiceClient newClient;

    public OrderDto createOrder(CreateOrderRequest request) {
        if (featureFlags.isEnabled("new-order-service",
                Map.of("customerId", request.getCustomerId()))) {
            log.info("Routing to new order service for customer {}", request.getCustomerId());
            try {
                return newClient.createOrder(request);
            } catch (Exception e) {
                log.warn("New order service failed, falling back to legacy", e);
                return legacyClient.createOrder(request);
            }
        }
        return legacyClient.createOrder(request);
    }
}

// Feature flag service with percentage rollout
@Service
public class FeatureFlagService {

    private final FeatureFlagRepository flagRepository;

    public boolean isEnabled(String flagName, Map<String, Object> context) {
        FeatureFlag flag = flagRepository.findByName(flagName)
            .orElse(FeatureFlag.disabled(flagName));

        if (!flag.isEnabled()) return false;

        // Percentage rollout based on customer ID
        if (context.containsKey("customerId") && flag.getRolloutPercentage() < 100) {
            String customerId = context.get("customerId").toString();
            int hash = Math.abs(customerId.hashCode() % 100);
            return hash < flag.getRolloutPercentage();
        }

        return true;
    }
}

// Spring Cloud Gateway-based routing (config approach)
// gateway.yml example:
// spring:
//   cloud:
//     gateway:
//       routes:
//         - id: new-order-service
//           uri: lb://order-service-v2
//           predicates:
//             - Path=/api/orders/**
//             - Header=X-Feature-Flag, new-order-service
//         - id: legacy-order-service
//           uri: lb://legacy-monolith
//           predicates:
//             - Path=/api/orders/**
```

---

## 8. Anti-Corruption Layer

Translate between the domain models of two services to prevent pollution.

```java
// Legacy payment system returns old-format data
// Our domain uses modern, clean model

// Legacy DTO (from old payment system)
public class LegacyPaymentResponse {
    public String pmtId;
    public String pmtStat;  // "S" = Success, "F" = Failed, "P" = Pending
    public double pmtAmt;
    public String pmtDt;    // "YYYY/MM/DD HH:mm:ss"
    public String custNo;
}

// Our domain model
public record Payment(
    String id,
    PaymentStatus status,
    BigDecimal amount,
    LocalDateTime processedAt,
    String customerId
) {}

public enum PaymentStatus { SUCCESS, FAILED, PENDING }

// Anti-corruption layer: translates legacy to our domain
@Component
public class PaymentAntiCorruptionLayer {

    private static final DateTimeFormatter LEGACY_DATE_FORMAT =
        DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");

    public Payment translate(LegacyPaymentResponse legacy) {
        return new Payment(
            legacy.pmtId,
            translateStatus(legacy.pmtStat),
            BigDecimal.valueOf(legacy.pmtAmt),
            LocalDateTime.parse(legacy.pmtDt, LEGACY_DATE_FORMAT),
            legacy.custNo
        );
    }

    private PaymentStatus translateStatus(String legacyStatus) {
        return switch (legacyStatus) {
            case "S" -> PaymentStatus.SUCCESS;
            case "F" -> PaymentStatus.FAILED;
            case "P" -> PaymentStatus.PENDING;
            default -> throw new IllegalArgumentException(
                "Unknown legacy payment status: " + legacyStatus);
        };
    }

    public LegacyPaymentRequest translateRequest(ProcessPaymentCommand command) {
        LegacyPaymentRequest request = new LegacyPaymentRequest();
        request.custNo = command.customerId();
        request.pmtAmt = command.amount().doubleValue();
        request.pmtTyp = "CC";  // Credit card - legacy always needs this
        return request;
    }
}

// Client wraps legacy system with ACL
@Component
public class LegacyPaymentClient {

    private final RestTemplate restTemplate;
    private final PaymentAntiCorruptionLayer acl;

    @Value("${legacy.payment.url}")
    private String legacyUrl;

    public Payment processPayment(ProcessPaymentCommand command) {
        LegacyPaymentRequest legacyRequest = acl.translateRequest(command);

        LegacyPaymentResponse legacyResponse = restTemplate.postForObject(
            legacyUrl + "/pmt/process",
            legacyRequest,
            LegacyPaymentResponse.class
        );

        return acl.translate(legacyResponse);
    }
}
```

---

## 9. Service Mesh with Istio Concepts

```java
// Istio handles cross-cutting concerns (mTLS, retry, circuit breaking)
// at the infrastructure level — minimal code changes needed in services

// But you may need to propagate trace headers
@Component
public class IstioHeaderPropagationInterceptor implements ClientHttpRequestInterceptor {

    private static final List<String> PROPAGATION_HEADERS = List.of(
        "x-request-id",
        "x-b3-traceid",
        "x-b3-spanid",
        "x-b3-parentspanid",
        "x-b3-sampled",
        "x-b3-flags",
        "x-ot-span-context",
        "x-envoy-force-trace"
    );

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        RequestContextHolder.getRequestAttributes();
        HttpServletRequest currentRequest = getCurrentRequest();

        if (currentRequest != null) {
            for (String header : PROPAGATION_HEADERS) {
                String value = currentRequest.getHeader(header);
                if (value != null) {
                    request.getHeaders().add(header, value);
                }
            }
        }

        return execution.execute(request, body);
    }

    private HttpServletRequest getCurrentRequest() {
        try {
            ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            return attrs.getRequest();
        } catch (IllegalStateException e) {
            return null;
        }
    }
}

// Istio VirtualService equivalent in Java (for testing/documentation)
// In real Istio you'd use YAML, but here's the concept expressed in code:
@Configuration
public class IstioLikeConfig {
    // Istio handles these at infra level:
    // - Retry: retry 3 times on 5xx, 500ms delay
    // - Circuit Breaker: open after 50% failure in 30s window
    // - mTLS: all service-to-service communication encrypted
    // - Traffic splitting: 90% v1, 10% v2
    // - Rate limiting: 100 req/s per service

    // We just configure distributed tracing
    @Bean
    public Tracer tracer() {
        return Tracing.newBuilder()
            .localServiceName("order-service")
            .spanReporter(AsyncReporter.create(URLConnectionSender.create("http://zipkin:9411/api/v2/spans")))
            .build()
            .tracer();
    }
}
```

---

## 10. Sidecar Pattern

```java
// The application service (business logic only)
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        // Pure business logic - no logging infrastructure concerns
        Order order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(OrderDto.from(order));
    }
}

// Sidecar handles:
// - Access logging (Envoy/NGINX sidecar)
// - TLS termination
// - Circuit breaking
// - Metrics collection
// - Health checking

// application.yml for sidecar-aware configuration:
// server:
//   port: 8080        # App listens on 8080
//
// management:
//   server:
//     port: 8081      # Actuator on separate port (for sidecar only)
//   endpoints:
//     web:
//       exposure:
//         include: health,metrics,prometheus
//
// # Sidecar (Envoy) listens on 80/443, forwards to localhost:8080

// Sidecar logging configuration
@Configuration
public class SidecarLoggingConfig {

    // Structured logging for sidecar aggregation
    @Bean
    public LoggingEventCompositeJsonEncoder jsonEncoder() {
        LoggingEventCompositeJsonEncoder encoder = new LoggingEventCompositeJsonEncoder();
        // Configure JSON fields for log aggregation
        return encoder;
    }
}
```

---

## 11. Ambassador Pattern

```java
// Ambassador: a proxy that handles cross-cutting concerns for external service calls

@Component
@Slf4j
public class ExternalPaymentGatewayAmbassador {

    private final RestTemplate restTemplate;
    private final RetryTemplate retryTemplate;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final MeterRegistry meterRegistry;

    public PaymentResult processPayment(PaymentRequest request) {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("payment-gateway");
        Counter successCounter = meterRegistry.counter("payment.gateway.calls", "result", "success");
        Counter failureCounter = meterRegistry.counter("payment.gateway.calls", "result", "failure");
        Timer timer = meterRegistry.timer("payment.gateway.duration");

        return timer.record(() ->
            cb.executeSupplier(() ->
                retryTemplate.execute(context -> {
                    try {
                        log.info("Calling payment gateway, attempt {}", context.getRetryCount() + 1);

                        HttpHeaders headers = new HttpHeaders();
                        headers.set("Authorization", "Bearer " + getToken());
                        headers.set("Idempotency-Key", request.getIdempotencyKey());
                        headers.setContentType(MediaType.APPLICATION_JSON);

                        ResponseEntity<PaymentGatewayResponse> response = restTemplate.exchange(
                            "https://payment-gateway.example.com/v1/payments",
                            HttpMethod.POST,
                            new HttpEntity<>(request, headers),
                            PaymentGatewayResponse.class
                        );

                        successCounter.increment();
                        return mapResponse(response.getBody());

                    } catch (HttpServerErrorException e) {
                        failureCounter.increment();
                        log.warn("Payment gateway error: {}", e.getStatusCode());
                        throw e;
                    }
                })
            )
        );
    }

    private String getToken() {
        // Token caching and refresh logic
        return "cached-jwt-token";
    }

    private PaymentResult mapResponse(PaymentGatewayResponse response) {
        return PaymentResult.builder()
            .transactionId(response.getTransactionId())
            .status(PaymentStatus.valueOf(response.getStatus()))
            .amount(response.getAmount())
            .build();
    }
}
```

---

## 12. Real Example: E-Commerce Order Saga with Compensation

```java
// Complete end-to-end implementation of Order Saga

// === Domain Events ===
public record OrderCreatedEvent(Long orderId, Long customerId, BigDecimal amount, List<OrderItem> items) {}
public record PaymentProcessedEvent(Long orderId, String transactionId) {}
public record PaymentFailedEvent(Long orderId, String reason) {}
public record InventoryReservedEvent(Long orderId, List<ReservationItem> reservations) {}
public record InventoryFailedEvent(Long orderId, String reason) {}
public record ShipmentScheduledEvent(Long orderId, String trackingNumber, LocalDate estimatedDelivery) {}
public record OrderCompletedEvent(Long orderId) {}
public record OrderCancelledEvent(Long orderId, String reason) {}

// === Saga State Machine ===
@Entity
@Table(name = "order_sagas")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderSaga {

    @Id
    private Long orderId;

    @Enumerated(EnumType.STRING)
    private SagaState state;

    private String paymentTransactionId;

    @ElementCollection
    @CollectionTable(name = "saga_reservations")
    private List<String> reservationIds = new ArrayList<>();

    private String trackingNumber;
    private String failureReason;

    @CreatedDate
    private LocalDateTime startedAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

public enum SagaState {
    STARTED,
    AWAITING_PAYMENT,
    PAYMENT_DONE,
    INVENTORY_RESERVED,
    SHIPMENT_SCHEDULED,
    COMPLETED,
    COMPENSATING_INVENTORY,
    COMPENSATING_PAYMENT,
    CANCELLED,
    FAILED
}

// === Saga Manager ===
@Service
@Slf4j
public class EcommerceOrderSaga {

    private final OrderSagaRepository sagaRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    // ---- Step 1: Order Created -> Reserve Payment ----
    @KafkaListener(topics = "order.created", groupId = "saga-manager")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Saga starting for order {}", event.orderId());

        OrderSaga saga = OrderSaga.builder()
            .orderId(event.orderId())
            .state(SagaState.AWAITING_PAYMENT)
            .startedAt(LocalDateTime.now())
            .build();
        sagaRepository.save(saga);

        kafkaTemplate.send("payment.process-request",
            Map.of(
                "orderId", event.orderId(),
                "customerId", event.customerId(),
                "amount", event.amount(),
                "idempotencyKey", "order-" + event.orderId()
            ));
    }

    // ---- Step 2a: Payment OK -> Reserve Inventory ----
    @KafkaListener(topics = "payment.processed", groupId = "saga-manager")
    @Transactional
    public void onPaymentProcessed(PaymentProcessedEvent event) {
        OrderSaga saga = sagaRepository.findById(event.orderId()).orElseThrow();

        if (saga.getState() != SagaState.AWAITING_PAYMENT) {
            log.warn("Received payment processed but saga state is {}. Ignoring.", saga.getState());
            return;
        }

        saga.setState(SagaState.PAYMENT_DONE);
        saga.setPaymentTransactionId(event.transactionId());
        sagaRepository.save(saga);

        kafkaTemplate.send("inventory.reserve-request",
            Map.of("orderId", event.orderId()));
    }

    // ---- Step 2b: Payment Failed -> Cancel Order ----
    @KafkaListener(topics = "payment.failed", groupId = "saga-manager")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        log.error("Payment failed for order {}: {}", event.orderId(), event.reason());
        OrderSaga saga = sagaRepository.findById(event.orderId()).orElseThrow();

        saga.setState(SagaState.CANCELLED);
        saga.setFailureReason("Payment failed: " + event.reason());
        sagaRepository.save(saga);

        kafkaTemplate.send("order.cancel-request",
            Map.of("orderId", event.orderId(), "reason", event.reason()));
    }

    // ---- Step 3a: Inventory Reserved -> Schedule Shipment ----
    @KafkaListener(topics = "inventory.reserved", groupId = "saga-manager")
    @Transactional
    public void onInventoryReserved(InventoryReservedEvent event) {
        OrderSaga saga = sagaRepository.findById(event.orderId()).orElseThrow();

        saga.setState(SagaState.INVENTORY_RESERVED);
        sagaRepository.save(saga);

        kafkaTemplate.send("shipping.schedule-request",
            Map.of("orderId", event.orderId()));
    }

    // ---- Step 3b: Inventory Failed -> Refund Payment ----
    @KafkaListener(topics = "inventory.failed", groupId = "saga-manager")
    @Transactional
    public void onInventoryFailed(InventoryFailedEvent event) {
        log.error("Inventory reservation failed for order {}: {}", event.orderId(), event.reason());
        OrderSaga saga = sagaRepository.findById(event.orderId()).orElseThrow();

        saga.setState(SagaState.COMPENSATING_PAYMENT);
        saga.setFailureReason("Inventory failed: " + event.reason());
        sagaRepository.save(saga);

        // Compensate: refund payment
        kafkaTemplate.send("payment.refund-request",
            Map.of(
                "orderId", event.orderId(),
                "transactionId", saga.getPaymentTransactionId()
            ));

        kafkaTemplate.send("order.cancel-request",
            Map.of("orderId", event.orderId(), "reason", event.reason()));
    }

    // ---- Step 4: Shipment Scheduled -> Complete Order ----
    @KafkaListener(topics = "shipping.scheduled", groupId = "saga-manager")
    @Transactional
    public void onShipmentScheduled(ShipmentScheduledEvent event) {
        OrderSaga saga = sagaRepository.findById(event.orderId()).orElseThrow();

        saga.setState(SagaState.COMPLETED);
        saga.setTrackingNumber(event.trackingNumber());
        sagaRepository.save(saga);

        kafkaTemplate.send("order.completed",
            Map.of(
                "orderId", event.orderId(),
                "trackingNumber", event.trackingNumber(),
                "estimatedDelivery", event.estimatedDelivery()
            ));

        log.info("Order {} saga completed successfully. Tracking: {}",
            event.orderId(), event.trackingNumber());
    }
}

// === Monitoring Endpoint ===
@RestController
@RequestMapping("/api/sagas")
public class SagaMonitorController {

    private final OrderSagaRepository sagaRepository;

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderSaga> getSagaStatus(@PathVariable Long orderId) {
        return sagaRepository.findById(orderId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/failed")
    public List<OrderSaga> getFailedSagas() {
        return sagaRepository.findByStateIn(
            List.of(SagaState.FAILED, SagaState.CANCELLED));
    }

    @GetMapping("/stats")
    public Map<SagaState, Long> getSagaStats() {
        return sagaRepository.countByState();
    }
}
```

---

## Configuration Reference

```yaml
# application.yml for saga-enabled microservice
spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: saga-manager
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 1

resilience4j:
  circuitbreaker:
    instances:
      payment-gateway:
        failure-rate-threshold: 50
        slow-call-duration-threshold: 2s
        permitted-number-of-calls-in-half-open-state: 10
        sliding-window-size: 100
        wait-duration-in-open-state: 30s

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,circuitbreakers,saga-stats
```

---

## Summary

| Pattern | Purpose | Use When |
|---------|---------|----------|
| Saga (Choreography) | Distributed transactions | Loose coupling, event-driven |
| Saga (Orchestration) | Distributed transactions | Complex flows, clear state machine |
| Outbox | Reliable event publishing | Must not lose events |
| Inbox | Idempotent consumers | Prevent duplicate processing |
| API Composition | Query aggregation | Need data from multiple services |
| CQRS | Read/write separation | High read load, complex queries |
| Database per Service | Data isolation | True service independence |
| Strangler Fig | Monolith migration | Incremental modernization |
| Anti-Corruption Layer | Model translation | Integrating legacy systems |
| Service Mesh | Cross-cutting concerns | Infrastructure-level resilience |
| Sidecar | Capability injection | Non-business infrastructure |
| Ambassador | External service proxy | Third-party API integration |

## Next Part Preview

**Part 060: Resilience Patterns for Microservices** — We'll deep dive into Circuit Breaker, Bulkhead, Retry with jitter, Rate Limiting, and how to build a payment service that gracefully handles every failure mode.
