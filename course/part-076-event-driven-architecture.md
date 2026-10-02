# Part 076 – Advanced Event-Driven Architecture

## Dependencies (pom.xml)

```xml
<dependencies>
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <!-- Kafka Streams -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
    </dependency>
    <!-- Avro -->
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.11.3</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.6.0</version>
    </dependency>
    <!-- Spring Integration -->
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-jdbc</artifactId>
    </dependency>
    <!-- JPA for Outbox -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>confluent</id>
        <url>https://packages.confluent.io/maven/</url>
    </repository>
</repositories>
```

---

## 1. Application Properties

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: event-driven-demo
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 1
        transactional.id: order-producer-tx
    consumer:
      group-id: order-consumer-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events.*"
        isolation.level: read_committed
    streams:
      application-id: order-streams-app
      bootstrap-servers: localhost:9092
      default-key-serde: org.apache.kafka.common.serialization.Serdes$StringSerde
      default-value-serde: org.springframework.kafka.support.serializer.JsonSerde
      properties:
        commit.interval.ms: 1000
        cache.max.bytes.buffering: 10485760

  datasource:
    url: jdbc:postgresql://localhost:5432/eventstore
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: update

schema.registry.url: http://localhost:8081
```

---

## 2. Domain Events

```java
package com.example.events;

import java.time.Instant;
import java.util.UUID;

public abstract class DomainEvent {
    private final String eventId;
    private final String eventType;
    private final Instant occurredAt;
    private final int version;

    protected DomainEvent(String eventType, int version) {
        this.eventId = UUID.randomUUID().toString();
        this.eventType = eventType;
        this.occurredAt = Instant.now();
        this.version = version;
    }

    public String getEventId() { return eventId; }
    public String getEventType() { return eventType; }
    public Instant getOccurredAt() { return occurredAt; }
    public int getVersion() { return version; }
}
```

```java
package com.example.events;

import java.math.BigDecimal;

public class OrderPlacedEvent extends DomainEvent {
    private String orderId;
    private String customerId;
    private BigDecimal totalAmount;
    private String currency;

    public OrderPlacedEvent() { super("ORDER_PLACED", 1); }

    public OrderPlacedEvent(String orderId, String customerId, BigDecimal totalAmount) {
        super("ORDER_PLACED", 1);
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.currency = "USD";
    }

    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
}
```

```java
package com.example.events;

public class OrderShippedEvent extends DomainEvent {
    private String orderId;
    private String trackingNumber;
    private String carrier;

    public OrderShippedEvent() { super("ORDER_SHIPPED", 1); }

    public OrderShippedEvent(String orderId, String trackingNumber, String carrier) {
        super("ORDER_SHIPPED", 1);
        this.orderId = orderId;
        this.trackingNumber = trackingNumber;
        this.carrier = carrier;
    }

    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }
    public String getTrackingNumber() { return trackingNumber; }
    public void setTrackingNumber(String trackingNumber) { this.trackingNumber = trackingNumber; }
    public String getCarrier() { return carrier; }
    public void setCarrier(String carrier) { this.carrier = carrier; }
}
```

---

## 3. Kafka Producer with Transactional API (Exactly-Once)

```java
package com.example.kafka;

import com.example.events.DomainEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.concurrent.CompletableFuture;

@Service
public class OrderEventProducer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventProducer.class);
    private static final String ORDERS_TOPIC = "orders";
    private static final String ORDER_EVENTS_TOPIC = "order-events";

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // Async send with callback
    public void sendOrderEvent(String orderId, DomainEvent event) {
        CompletableFuture<SendResult<String, Object>> future =
                kafkaTemplate.send(ORDER_EVENTS_TOPIC, orderId, event);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                log.info("Sent event={} orderId={} partition={} offset={}",
                        event.getEventType(), orderId,
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
            } else {
                log.error("Failed to send event={} orderId={}: {}",
                        event.getEventType(), orderId, ex.getMessage());
                // In production: persist to dead-letter store or retry
            }
        });
    }

    // Transactional send – atomically send to multiple topics
    @Transactional("kafkaTransactionManager")
    public void sendOrderWithInventoryUpdate(String orderId, DomainEvent orderEvent,
                                              DomainEvent inventoryEvent) {
        kafkaTemplate.send(ORDER_EVENTS_TOPIC, orderId, orderEvent);
        kafkaTemplate.send("inventory-events", orderId, inventoryEvent);
        log.info("Sent transactional events for orderId={}", orderId);
        // If any exception occurs here, both sends are rolled back
    }

    // Send with headers for schema versioning
    public void sendVersionedEvent(String topic, String key, DomainEvent event) {
        org.apache.kafka.clients.producer.ProducerRecord<String, Object> record =
                new org.apache.kafka.clients.producer.ProducerRecord<>(topic, key, event);
        record.headers()
                .add("eventType", event.getEventType().getBytes())
                .add("eventVersion", String.valueOf(event.getVersion()).getBytes())
                .add("contentType", "application/json".getBytes())
                .add("source", "order-service".getBytes());

        kafkaTemplate.send(record).whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to send versioned event: {}", ex.getMessage());
            }
        });
    }
}
```

---

## 4. Kafka Producer Configuration (Idempotent)

```java
package com.example.kafka;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;
import org.springframework.kafka.transaction.KafkaTransactionManager;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        // Exactly-once semantics
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        // Transactional ID for EOS
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-tx-1");
        // Compression for throughput
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);

        DefaultKafkaProducerFactory<String, Object> factory =
                new DefaultKafkaProducerFactory<>(props);
        factory.setTransactionIdPrefix("order-tx-");
        return factory;
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    public KafkaTransactionManager<String, Object> kafkaTransactionManager() {
        return new KafkaTransactionManager<>(producerFactory());
    }
}
```

---

## 5. Kafka Consumer with Dead Letter Topic

```java
package com.example.kafka;

import com.example.events.OrderPlacedEvent;
import com.example.events.OrderShippedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.DltHandler;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.retrytopic.TopicSuffixingStrategy;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.retry.annotation.Backoff;
import org.springframework.stereotype.Service;

@Service
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    // Automatic retry with Dead Letter Topic
    @RetryableTopic(
            attempts = "4",
            backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000),
            topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
            dltTopicSuffix = "-dead-letter",
            autoCreateTopics = "true"
    )
    @KafkaListener(
            topics = "order-events",
            groupId = "order-consumer-group",
            containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvent(
            @Payload String rawPayload,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            @Header(KafkaHeaders.RECEIVED_KEY) String key) {

        log.info("Received event from topic={} partition={} offset={} key={}",
                topic, partition, offset, key);

        // Simulate processing – throw to trigger retry
        if (key.startsWith("FAIL-")) {
            throw new RuntimeException("Simulated processing failure for key: " + key);
        }

        log.info("Successfully processed event for key={}", key);
    }

    @KafkaListener(topics = "order-placed", groupId = "order-processor")
    public void handleOrderPlaced(@Payload OrderPlacedEvent event,
                                   @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.info("Order placed: orderId={} customerId={} amount={}",
                event.getOrderId(), event.getCustomerId(), event.getTotalAmount());
        // Business logic: reserve inventory, calculate taxes, etc.
    }

    @KafkaListener(topics = "order-shipped", groupId = "notification-group")
    public void handleOrderShipped(@Payload OrderShippedEvent event) {
        log.info("Order shipped: orderId={} tracking={} carrier={}",
                event.getOrderId(), event.getTrackingNumber(), event.getCarrier());
        // Send email/SMS notification
    }

    // Dead Letter Topic handler – called after all retries exhausted
    @DltHandler
    public void handleDeadLetter(
            @Payload String rawPayload,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {

        log.error("Dead letter received from topic={} exception={} payload={}",
                topic, exceptionMessage, rawPayload);
        // Save to database, send alert, manual review queue
    }
}
```

---

## 6. Kafka Consumer Configuration

```java
package com.example.kafka;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.support.serializer.ErrorHandlingDeserializer;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-consumer-group");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // Read only committed messages (EOS)
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        // Error-handling deserializer wraps bad messages
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, StringDeserializer.class);
        props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.events.*");
        props.put(JsonDeserializer.USE_TYPE_INFO_HEADERS, false);
        props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, "com.example.events.DomainEvent");

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
        factory.getContainerProperties().setSyncCommits(true);
        return factory;
    }
}
```

---

## 7. Kafka Streams – KStream Processing

```java
package com.example.streams;

import com.example.events.OrderPlacedEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafkaStreams;

import java.time.Duration;

@Configuration
@EnableKafkaStreams
public class OrderStreamProcessor {

    private static final Logger log = LoggerFactory.getLogger(OrderStreamProcessor.class);
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Autowired
    public void buildPipeline(StreamsBuilder builder) {
        // Source stream from order-placed topic
        KStream<String, String> ordersStream = builder.stream(
                "order-placed",
                Consumed.with(Serdes.String(), Serdes.String())
        );

        // Filter high-value orders and route to premium topic
        ordersStream
                .filter((key, value) -> isHighValueOrder(value))
                .peek((key, value) -> log.info("High-value order: key={}", key))
                .to("high-value-orders", Produced.with(Serdes.String(), Serdes.String()));

        // Branch by currency
        Map<String, KStream<String, String>> branches = ordersStream.split(Named.as("branch-"))
                .branch((key, value) -> containsCurrency(value, "USD"), Branched.as("usd"))
                .branch((key, value) -> containsCurrency(value, "EUR"), Branched.as("eur"))
                .defaultBranch(Branched.as("other"));

        branches.get("branch-usd").to("orders-usd");
        branches.get("branch-eur").to("orders-eur");

        // Map/transform: enrich each order with processing metadata
        ordersStream
                .mapValues(value -> enrichOrder(value))
                .to("enriched-orders");

        // FlatMap: one event → multiple notifications
        ordersStream
                .flatMapValues(value -> generateNotifications(value))
                .to("notifications");
    }

    @Autowired
    public void buildAggregations(StreamsBuilder builder) {
        KStream<String, String> stream = builder.stream(
                "order-placed",
                Consumed.with(Serdes.String(), Serdes.String())
        );

        // Tumbling window: count orders per customer per 5-minute window
        KTable<Windowed<String>, Long> orderCounts = stream
                .groupByKey()
                .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
                .count(Materialized.as("order-count-store"));

        orderCounts.toStream()
                .map((windowedKey, count) -> KeyValue.pair(
                        windowedKey.key(),
                        windowedKey.key() + "::" + count
                ))
                .to("order-count-per-window");

        // Hopping window: sliding count over 10 minutes, advancing every 2 minutes
        KTable<Windowed<String>, Long> slidingCounts = stream
                .groupByKey()
                .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(10), Duration.ofMinutes(1))
                        .advanceBy(Duration.ofMinutes(2)))
                .count(Materialized.as("sliding-order-count-store"));

        slidingCounts.toStream()
                .peek((key, count) -> log.debug("Sliding count: key={} count={}", key.key(), count))
                .to("sliding-order-counts");
    }

    private boolean isHighValueOrder(String value) {
        try {
            com.fasterxml.jackson.databind.JsonNode node = objectMapper.readTree(value);
            double amount = node.path("totalAmount").asDouble(0);
            return amount > 1000.0;
        } catch (Exception e) {
            return false;
        }
    }

    private boolean containsCurrency(String value, String currency) {
        return value != null && value.contains("\"currency\":\"" + currency + "\"");
    }

    private String enrichOrder(String value) {
        try {
            com.fasterxml.jackson.databind.node.ObjectNode node =
                    (com.fasterxml.jackson.databind.node.ObjectNode) objectMapper.readTree(value);
            node.put("enrichedAt", java.time.Instant.now().toString());
            node.put("processingNode", java.net.InetAddress.getLocalHost().getHostName());
            return objectMapper.writeValueAsString(node);
        } catch (Exception e) {
            return value;
        }
    }

    private java.util.List<String> generateNotifications(String value) {
        try {
            com.fasterxml.jackson.databind.JsonNode node = objectMapper.readTree(value);
            String customerId = node.path("customerId").asText();
            String orderId = node.path("orderId").asText();
            return java.util.Arrays.asList(
                    "{\"channel\":\"email\",\"to\":\"" + customerId + "\",\"orderId\":\"" + orderId + "\"}",
                    "{\"channel\":\"sms\",\"to\":\"" + customerId + "\",\"orderId\":\"" + orderId + "\"}"
            );
        } catch (Exception e) {
            return java.util.Collections.emptyList();
        }
    }
}
```

---

## 8. Kafka Streams – KTable and Stream-Table Join

```java
package com.example.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OrderInventoryJoinProcessor {

    @Autowired
    public void buildJoinPipeline(StreamsBuilder builder) {
        // Orders stream
        KStream<String, String> orders = builder.stream(
                "order-placed",
                Consumed.with(Serdes.String(), Serdes.String())
        );

        // Inventory table (compacted topic – KTable)
        KTable<String, String> inventory = builder.table(
                "inventory-snapshot",
                Consumed.with(Serdes.String(), Serdes.String()),
                Materialized.as("inventory-store")
        );

        // Stream-Table join: enrich each order with current inventory level
        orders.join(
                inventory,
                (orderJson, inventoryJson) -> mergeOrderWithInventory(orderJson, inventoryJson),
                Joined.with(Serdes.String(), Serdes.String(), Serdes.String())
        ).to("enriched-orders-with-inventory");

        // Left join: keep orders even if no inventory record
        orders.leftJoin(
                inventory,
                (orderJson, inventoryJson) ->
                        inventoryJson != null
                                ? mergeOrderWithInventory(orderJson, inventoryJson)
                                : addMissingInventoryFlag(orderJson)
        ).to("all-orders-enriched");

        // KTable-KTable join: join two state tables
        KTable<String, String> customerTable = builder.table(
                "customers-snapshot",
                Consumed.with(Serdes.String(), Serdes.String())
        );

        inventory.join(
                customerTable,
                (inventoryJson, customerJson) -> mergeInventoryWithCustomer(inventoryJson, customerJson)
        ).toStream().to("inventory-with-customer");
    }

    private String mergeOrderWithInventory(String order, String inventory) {
        return "{\"order\":" + order + ",\"inventory\":" + inventory + "}";
    }

    private String addMissingInventoryFlag(String order) {
        return order.replace("}", ",\"inventoryMissing\":true}");
    }

    private String mergeInventoryWithCustomer(String inventory, String customer) {
        return "{\"inventory\":" + inventory + ",\"customer\":" + customer + "}";
    }
}
```

---

## 9. Outbox Pattern – Entity and Repository

```java
package com.example.outbox;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "outbox_events",
       indexes = @Index(columnList = "status, created_at"))
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String eventType;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String payload;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private OutboxStatus status = OutboxStatus.PENDING;

    @Column(nullable = false)
    private Instant createdAt = Instant.now();

    private Instant processedAt;
    private int retryCount = 0;
    private String errorMessage;

    public enum OutboxStatus { PENDING, SENT, FAILED }

    // No-arg constructor for JPA
    public OutboxEvent() {}

    public OutboxEvent(String aggregateType, String aggregateId,
                       String eventType, String payload) {
        this.aggregateType = aggregateType;
        this.aggregateId = aggregateId;
        this.eventType = eventType;
        this.payload = payload;
    }

    public String getId() { return id; }
    public String getAggregateType() { return aggregateType; }
    public String getAggregateId() { return aggregateId; }
    public String getEventType() { return eventType; }
    public String getPayload() { return payload; }
    public OutboxStatus getStatus() { return status; }
    public void setStatus(OutboxStatus status) { this.status = status; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getProcessedAt() { return processedAt; }
    public void setProcessedAt(Instant processedAt) { this.processedAt = processedAt; }
    public int getRetryCount() { return retryCount; }
    public void incrementRetry() { this.retryCount++; }
    public String getErrorMessage() { return errorMessage; }
    public void setErrorMessage(String errorMessage) { this.errorMessage = errorMessage; }
}
```

```java
package com.example.outbox;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface OutboxEventRepository extends JpaRepository<OutboxEvent, String> {

    @Query("""
        SELECT e FROM OutboxEvent e
        WHERE e.status = 'PENDING'
        ORDER BY e.createdAt ASC
        LIMIT 100
    """)
    List<OutboxEvent> findPendingEvents();

    @Query("""
        SELECT e FROM OutboxEvent e
        WHERE e.status = 'FAILED' AND e.retryCount < 3
        ORDER BY e.createdAt ASC
        LIMIT 50
    """)
    List<OutboxEvent> findRetryableFailedEvents();
}
```

---

## 10. Outbox Polling Publisher (Transactional Outbox)

```java
package com.example.outbox;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Service
@EnableScheduling
public class OutboxPollingPublisher {

    private static final Logger log = LoggerFactory.getLogger(OutboxPollingPublisher.class);

    private final OutboxEventRepository outboxRepo;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public OutboxPollingPublisher(OutboxEventRepository outboxRepo,
                                   KafkaTemplate<String, Object> kafkaTemplate,
                                   ObjectMapper objectMapper) {
        this.outboxRepo = outboxRepo;
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepo.findPendingEvents();
        if (pending.isEmpty()) return;

        log.debug("Processing {} pending outbox events", pending.size());

        for (OutboxEvent event : pending) {
            try {
                String topic = topicForEventType(event.getEventType());
                Object payload = objectMapper.readValue(event.getPayload(), Object.class);

                kafkaTemplate.send(topic, event.getAggregateId(), payload)
                        .whenComplete((result, ex) -> {
                            if (ex == null) {
                                event.setStatus(OutboxEvent.OutboxStatus.SENT);
                                event.setProcessedAt(Instant.now());
                                outboxRepo.save(event);
                                log.debug("Published outbox event id={} type={}",
                                        event.getId(), event.getEventType());
                            } else {
                                handlePublishFailure(event, ex);
                            }
                        });
            } catch (Exception ex) {
                handlePublishFailure(event, ex);
            }
        }
    }

    @Scheduled(fixedDelay = 30000)
    @Transactional
    public void retryFailedEvents() {
        List<OutboxEvent> retryable = outboxRepo.findRetryableFailedEvents();
        if (retryable.isEmpty()) return;

        log.info("Retrying {} failed outbox events", retryable.size());
        for (OutboxEvent event : retryable) {
            event.setStatus(OutboxEvent.OutboxStatus.PENDING);
            outboxRepo.save(event);
        }
    }

    private void handlePublishFailure(OutboxEvent event, Throwable ex) {
        event.setStatus(OutboxEvent.OutboxStatus.FAILED);
        event.incrementRetry();
        event.setErrorMessage(ex.getMessage());
        outboxRepo.save(event);
        log.error("Failed to publish outbox event id={}: {}", event.getId(), ex.getMessage());
    }

    private String topicForEventType(String eventType) {
        return switch (eventType) {
            case "ORDER_PLACED"   -> "order-placed";
            case "ORDER_SHIPPED"  -> "order-shipped";
            case "ORDER_CANCELLED"-> "order-cancelled";
            default               -> "domain-events";
        };
    }
}
```

---

## 11. Transactional Order Service with Outbox

```java
package com.example.service;

import com.example.events.OrderPlacedEvent;
import com.example.outbox.OutboxEvent;
import com.example.outbox.OutboxEventRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.UUID;

@Service
public class OrderService {

    private final OutboxEventRepository outboxRepo;
    private final ApplicationEventPublisher eventPublisher;
    private final ObjectMapper objectMapper;

    public OrderService(OutboxEventRepository outboxRepo,
                        ApplicationEventPublisher eventPublisher,
                        ObjectMapper objectMapper) {
        this.outboxRepo = outboxRepo;
        this.eventPublisher = eventPublisher;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public String placeOrder(String customerId, BigDecimal amount) {
        String orderId = UUID.randomUUID().toString();

        // 1. Persist order to database (your Order entity save would go here)

        // 2. Write to outbox in same transaction (atomicity guaranteed)
        OrderPlacedEvent event = new OrderPlacedEvent(orderId, customerId, amount);
        try {
            String payload = objectMapper.writeValueAsString(event);
            OutboxEvent outboxEvent = new OutboxEvent(
                    "Order", orderId, "ORDER_PLACED", payload
            );
            outboxRepo.save(outboxEvent);
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize event", e);
        }

        // 3. Publish Spring ApplicationEvent for in-process listeners
        eventPublisher.publishEvent(new OrderPlacedSpringEvent(this, orderId, customerId, amount));

        return orderId;
    }

    // Spring Application Event (in-process, synchronous)
    public static class OrderPlacedSpringEvent extends org.springframework.context.ApplicationEvent {
        private final String orderId;
        private final String customerId;
        private final BigDecimal amount;

        public OrderPlacedSpringEvent(Object source, String orderId,
                                      String customerId, BigDecimal amount) {
            super(source);
            this.orderId = orderId;
            this.customerId = customerId;
            this.amount = amount;
        }

        public String getOrderId() { return orderId; }
        public String getCustomerId() { return customerId; }
        public BigDecimal getAmount() { return amount; }
    }
}
```

---

## 12. Domain Event Listeners (ApplicationEventPublisher)

```java
package com.example.service;

import com.example.service.OrderService.OrderPlacedSpringEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

@Component
public class OrderEventListeners {

    private static final Logger log = LoggerFactory.getLogger(OrderEventListeners.class);

    // Runs AFTER the transaction commits – safe to do external calls
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void onOrderPlacedAfterCommit(OrderPlacedSpringEvent event) {
        log.info("TX committed. Triggering post-commit actions for orderId={}",
                event.getOrderId());
        // Send welcome email, update analytics, etc.
    }

    // Runs BEFORE commit – can still roll back the transaction
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void onOrderPlacedBeforeCommit(OrderPlacedSpringEvent event) {
        log.debug("Before commit: validating order orderId={}", event.getOrderId());
        // Synchronous validation that can abort transaction if needed
    }

    // Always fires regardless of transaction outcome
    @EventListener
    public void onOrderPlacedAlways(OrderPlacedSpringEvent event) {
        log.debug("Order event received (always): orderId={}", event.getOrderId());
    }

    // On rollback: compensate
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderPlacedRollback(OrderPlacedSpringEvent event) {
        log.warn("Transaction rolled back for orderId={}. Compensation required.",
                event.getOrderId());
        // Compensating transaction logic
    }
}
```

---

## 13. Avro Schema and Serializer

```json
// src/main/avro/OrderEvent.avsc
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.example.avro",
  "fields": [
    { "name": "eventId",    "type": "string" },
    { "name": "eventType",  "type": "string" },
    { "name": "orderId",    "type": "string" },
    { "name": "customerId", "type": "string" },
    { "name": "totalAmount","type": "double" },
    { "name": "currency",   "type": "string", "default": "USD" },
    { "name": "occurredAt", "type": "long",   "logicalType": "timestamp-millis" },
    { "name": "version",    "type": "int",    "default": 1 }
  ]
}
```

```java
package com.example.kafka;

import io.confluent.kafka.serializers.KafkaAvroSerializer;
import io.confluent.kafka.serializers.KafkaAvroDeserializer;
import io.confluent.kafka.serializers.AbstractKafkaSchemaSerDeConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class AvroKafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${schema.registry.url}")
    private String schemaRegistryUrl;

    @Bean
    public ProducerFactory<String, Object> avroProducerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put(AbstractKafkaSchemaSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, schemaRegistryUrl);
        props.put(AbstractKafkaSchemaSerDeConfig.AUTO_REGISTER_SCHEMAS, true);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, Object> avroKafkaTemplate() {
        return new KafkaTemplate<>(avroProducerFactory());
    }

    @Bean
    public ConsumerFactory<String, Object> avroConsumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "avro-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put(AbstractKafkaSchemaSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, schemaRegistryUrl);
        props.put(KafkaAvroDeserializer.SPECIFIC_AVRO_READER_CONFIG, true);
        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

---

## 14. Event Audit Log

```java
package com.example.audit;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "event_audit_log",
       indexes = {
           @Index(columnList = "aggregate_id"),
           @Index(columnList = "event_type"),
           @Index(columnList = "occurred_at")
       })
public class EventAuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String eventId;

    @Column(nullable = false)
    private String eventType;

    @Column(nullable = false)
    private String aggregateType;

    @Column(name = "aggregate_id", nullable = false)
    private String aggregateId;

    @Column(columnDefinition = "TEXT")
    private String payload;

    @Column(name = "occurred_at", nullable = false)
    private Instant occurredAt;

    private String sourceService;
    private String correlationId;
    private int schemaVersion;

    public EventAuditLog() {}

    public EventAuditLog(String eventId, String eventType, String aggregateType,
                          String aggregateId, String payload) {
        this.eventId = eventId;
        this.eventType = eventType;
        this.aggregateType = aggregateType;
        this.aggregateId = aggregateId;
        this.payload = payload;
        this.occurredAt = Instant.now();
    }

    public String getId() { return id; }
    public String getEventId() { return eventId; }
    public String getEventType() { return eventType; }
    public String getAggregateType() { return aggregateType; }
    public String getAggregateId() { return aggregateId; }
    public String getPayload() { return payload; }
    public Instant getOccurredAt() { return occurredAt; }
    public void setSourceService(String sourceService) { this.sourceService = sourceService; }
    public void setCorrelationId(String correlationId) { this.correlationId = correlationId; }
    public void setSchemaVersion(int version) { this.schemaVersion = version; }
}
```

```java
package com.example.audit;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.time.Instant;
import java.util.List;

public interface EventAuditLogRepository extends JpaRepository<EventAuditLog, String> {

    List<EventAuditLog> findByAggregateIdOrderByOccurredAtAsc(String aggregateId);

    List<EventAuditLog> findByEventTypeAndOccurredAtBetween(
            String eventType, Instant from, Instant to);

    @Query("""
        SELECT e FROM EventAuditLog e
        WHERE e.aggregateId = :aggregateId
        ORDER BY e.occurredAt ASC
    """)
    List<EventAuditLog> findEventReplay(String aggregateId);
}
```

```java
package com.example.audit;

import com.example.events.DomainEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class EventAuditService {

    private static final Logger log = LoggerFactory.getLogger(EventAuditService.class);
    private final EventAuditLogRepository auditRepo;
    private final ObjectMapper objectMapper;

    public EventAuditService(EventAuditLogRepository auditRepo, ObjectMapper objectMapper) {
        this.auditRepo = auditRepo;
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = {"order-placed", "order-shipped", "order-cancelled"},
                   groupId = "audit-log-consumer")
    public void auditDomainEvent(@Payload String rawPayload,
            org.springframework.messaging.handler.annotation.Header(
                    org.springframework.kafka.support.KafkaHeaders.RECEIVED_TOPIC) String topic) {
        try {
            com.fasterxml.jackson.databind.JsonNode node = objectMapper.readTree(rawPayload);
            String eventId = node.path("eventId").asText("unknown");
            String eventType = node.path("eventType").asText(topic);
            String aggregateId = node.path("orderId").asText(
                    node.path("aggregateId").asText("unknown"));

            EventAuditLog auditLog = new EventAuditLog(
                    eventId, eventType, "Order", aggregateId, rawPayload);
            auditLog.setSourceService("order-service");
            auditLog.setSchemaVersion(node.path("version").asInt(1));
            auditRepo.save(auditLog);
            log.debug("Audited event: type={} aggregateId={}", eventType, aggregateId);
        } catch (Exception e) {
            log.error("Failed to audit event from topic {}: {}", topic, e.getMessage());
        }
    }

    public java.util.List<EventAuditLog> replayEvents(String aggregateId) {
        return auditRepo.findEventReplay(aggregateId);
    }
}
```

---

## 15. Kafka Topics Configuration (Auto-Create)

```java
package com.example.kafka;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic orderPlacedTopic() {
        return TopicBuilder.name("order-placed")
                .partitions(6)
                .replicas(3)
                .config("retention.ms", String.valueOf(7 * 24 * 60 * 60 * 1000L)) // 7 days
                .config("min.insync.replicas", "2")
                .build();
    }

    @Bean
    public NewTopic orderShippedTopic() {
        return TopicBuilder.name("order-shipped")
                .partitions(3)
                .replicas(3)
                .build();
    }

    @Bean
    public NewTopic orderEventsDltTopic() {
        return TopicBuilder.name("order-events-dead-letter")
                .partitions(3)
                .replicas(3)
                .config("retention.ms", String.valueOf(30L * 24 * 60 * 60 * 1000L)) // 30 days
                .build();
    }

    @Bean
    public NewTopic inventorySnapshotTopic() {
        return TopicBuilder.name("inventory-snapshot")
                .partitions(3)
                .replicas(3)
                .compact()  // Log compaction for KTable
                .build();
    }
}
```

---

## 16. Kafka Streams Integration Tests

```java
package com.example.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.TestInputTopic;
import org.apache.kafka.streams.TestOutputTopic;
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.kstream.KStream;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Properties;

import static org.assertj.core.api.Assertions.assertThat;

class OrderStreamProcessorTest {

    private TopologyTestDriver testDriver;
    private TestInputTopic<String, String> inputTopic;
    private TestOutputTopic<String, String> outputTopic;

    @BeforeEach
    void setUp() {
        Properties props = new Properties();
        props.put("application.id", "order-stream-test");
        props.put("bootstrap.servers", "dummy:9092");
        props.put("default.key.serde", Serdes.StringSerde.class.getName());
        props.put("default.value.serde", Serdes.StringSerde.class.getName());

        StreamsBuilder builder = new StreamsBuilder();

        // Recreate simple topology for unit testing
        KStream<String, String> stream = builder.stream("order-placed");
        stream.filter((key, value) -> value != null && value.contains("\"totalAmount\":2000"))
              .to("high-value-orders");

        testDriver = new TopologyTestDriver(builder.build(), props);

        inputTopic = testDriver.createInputTopic(
                "order-placed", Serdes.String().serializer(), Serdes.String().serializer());
        outputTopic = testDriver.createOutputTopic(
                "high-value-orders", Serdes.String().deserializer(), Serdes.String().deserializer());
    }

    @AfterEach
    void tearDown() {
        testDriver.close();
    }

    @Test
    void highValueOrdersAreRouted() {
        inputTopic.pipeInput("order-1",
                "{\"orderId\":\"order-1\",\"totalAmount\":2000,\"currency\":\"USD\"}");
        inputTopic.pipeInput("order-2",
                "{\"orderId\":\"order-2\",\"totalAmount\":50,\"currency\":\"USD\"}");

        assertThat(outputTopic.readRecordsToList()).hasSize(1);
        assertThat(outputTopic.readRecord().getKey()).isEqualTo("order-1");
    }

    @Test
    void lowValueOrdersAreNotRouted() {
        inputTopic.pipeInput("order-3",
                "{\"orderId\":\"order-3\",\"totalAmount\":100,\"currency\":\"USD\"}");

        assertThat(outputTopic.isEmpty()).isTrue();
    }
}
```

---

## 17. Kafka Integration Tests with Embedded Broker

```java
package com.example.kafka;

import com.example.events.OrderPlacedEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.kafka.test.utils.KafkaTestUtils;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.KafkaMessageListenerContainer;
import org.springframework.kafka.listener.MessageListener;

import java.math.BigDecimal;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"order-placed", "order-events", "order-events-dead-letter"},
    brokerProperties = {
        "transaction.state.log.replication.factor=1",
        "transaction.state.log.min.isr=1"
    }
)
class KafkaProducerConsumerTest {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private ConsumerFactory<String, Object> consumerFactory;

    @Test
    void producerSendsAndConsumerReceivesOrderEvent() throws Exception {
        BlockingQueue<ConsumerRecord<String, Object>> records = new LinkedBlockingQueue<>();

        ContainerProperties containerProps = new ContainerProperties("order-placed");
        containerProps.setMessageListener((MessageListener<String, Object>) records::offer);

        KafkaMessageListenerContainer<String, Object> container =
                new KafkaMessageListenerContainer<>(consumerFactory, containerProps);
        container.start();

        Thread.sleep(500); // Wait for container to start

        OrderPlacedEvent event = new OrderPlacedEvent("order-test-1", "customer-1", BigDecimal.TEN);
        kafkaTemplate.send("order-placed", "order-test-1", event);

        ConsumerRecord<String, Object> received = records.poll(10, TimeUnit.SECONDS);
        assertThat(received).isNotNull();
        assertThat(received.key()).isEqualTo("order-test-1");

        container.stop();
    }
}
```

---

## Summary

| Feature | Implementation |
|---|---|
| Exactly-once producer | `enable.idempotence=true` + `TRANSACTIONAL_ID` |
| Transactional send | `@Transactional("kafkaTransactionManager")` |
| Dead Letter Topic | `@RetryableTopic` + `@DltHandler` |
| Outbox Pattern | `OutboxEvent` entity + polling `@Scheduled` publisher |
| KStream filtering | `.filter(predicate).to(topic)` |
| KStream windowing | `windowedBy(TimeWindows.ofSizeWithNoGrace(...))` |
| Stream-Table join | `kstream.join(ktable, joiner)` |
| Avro serialization | `KafkaAvroSerializer` + Schema Registry |
| Domain events (in-process) | `ApplicationEventPublisher` + `@TransactionalEventListener` |
| Event audit log | `@KafkaListener` on all topics → persist to DB |
| Streams unit test | `TopologyTestDriver` + `TestInputTopic`/`TestOutputTopic` |
| Integration test | `@EmbeddedKafka` |
