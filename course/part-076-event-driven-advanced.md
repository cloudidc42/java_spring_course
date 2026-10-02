# Part 076: Advanced Event-Driven Architecture

This part covers advanced event-driven patterns using Apache Kafka: Kafka Streams for real-time processing, windowing, joins, state stores, exactly-once semantics, Avro with Schema Registry, and building a real-time inventory tracking system.

---

## Table of Contents

1. [Event-Driven vs Message-Driven Systems](#overview)
2. [Project Setup](#setup)
3. [Kafka Streams Basics](#streams-basics)
4. [Windowing Operations](#windowing)
5. [Kafka Streams Joins](#joins)
6. [State Stores](#state-stores)
7. [Kafka Connect for Data Pipelines](#connect)
8. [Dead Letter Topics and Error Handling](#error-handling)
9. [Exactly-Once Semantics](#eos)
10. [Event Versioning and Schema Evolution](#versioning)
11. [Apache Avro with Schema Registry](#avro)
12. [CQRS with Kafka and Event Sourcing](#cqrs)
13. [Real Example: Real-Time Inventory Tracking](#inventory)

---

## 1. Event-Driven vs Message-Driven Systems {#overview}

| Aspect | Message-Driven | Event-Driven |
|---|---|---|
| Focus | Commands/requests | Facts that happened |
| Consumer | Usually 1 consumer | Multiple independent consumers |
| Examples | "ProcessOrder" command | "OrderPlaced" event |
| Coupling | Point-to-point | Publisher doesn't know consumers |
| Replay | Usually not | Can replay from offset |

---

## 2. Project Setup {#setup}

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Kafka Streams with Spring -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
    </dependency>

    <!-- Avro + Schema Registry -->
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.5.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.11.3</version>
    </dependency>

    <!-- Kafka Connect (embedded for testing) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

```yaml
# src/main/resources/application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: inventory-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      properties:
        schema.registry.url: http://localhost:8081
        specific.avro.reader: true

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      properties:
        schema.registry.url: http://localhost:8081
        # Enable exactly-once semantics
        enable.idempotence: true
        acks: all
        retries: 2147483647
        max.in.flight.requests.per.connection: 5

    streams:
      application-id: inventory-streams
      bootstrap-servers: localhost:9092
      default-key-serde: org.apache.kafka.common.serialization.Serdes$StringSerde
      default-value-serde: org.apache.kafka.common.serialization.Serdes$StringSerde
      properties:
        processing.guarantee: exactly_once_v2
        commit.interval.ms: 100
        schema.registry.url: http://localhost:8081
        default.deserialization.exception.handler: >
          org.apache.kafka.streams.errors.LogAndContinueExceptionHandler

kafka:
  topics:
    inventory-events: inventory-events
    order-events: order-events
    product-events: product-events
    inventory-updates: inventory-updates
    low-stock-alerts: low-stock-alerts
    dead-letter: dead-letter-queue
```

---

## 3. Kafka Streams Basics {#streams-basics}

```java
// src/main/java/com/example/inventory/streams/InventoryStreamsConfig.java
package com.example.inventory.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafkaStreams;

@Configuration
@EnableKafkaStreams
public class InventoryStreamsConfig {

    /**
     * Simple stream: filter and transform events
     */
    @Bean
    public KStream<String, String> filterHighValueOrders(StreamsBuilder streamsBuilder) {
        KStream<String, String> orderStream = streamsBuilder.stream("order-events");

        return orderStream
            .filter((key, value) -> {
                // Filter orders over $100
                try {
                    OrderEvent event = parseOrderEvent(value);
                    return event.amount() > 100.0;
                } catch (Exception e) {
                    return false;
                }
            })
            .mapValues(value -> {
                OrderEvent event = parseOrderEvent(value);
                return "HIGH_VALUE:" + event.orderId() + ":" + event.amount();
            })
            .peek((key, value) -> System.out.println("High-value order: " + value));
    }

    private OrderEvent parseOrderEvent(String value) {
        // In practice, use proper deserialization (Avro, JSON)
        String[] parts = value.split(",");
        return new OrderEvent(parts[0], Double.parseDouble(parts[1]), parts[2]);
    }

    record OrderEvent(String orderId, double amount, String customerId) {}
}
```

```java
// src/main/java/com/example/inventory/streams/InventoryUpdateStream.java
package com.example.inventory.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class InventoryUpdateStream {

    @Autowired
    public void buildPipeline(StreamsBuilder streamsBuilder) {
        KStream<String, String> inventoryEvents = streamsBuilder.stream(
            "inventory-events",
            Consumed.with(Serdes.String(), Serdes.String())
        );

        // Branch: separate decrease vs increase events
        Map<String, KStream<String, String>> branches = inventoryEvents
            .split(Named.as("inventory-"))
            .branch((key, value) -> value.startsWith("DECREASE"),
                    Branched.as("decrease"))
            .branch((key, value) -> value.startsWith("INCREASE"),
                    Branched.as("increase"))
            .defaultBranch(Branched.as("other"));

        // Process decreases — check for low stock
        branches.get("inventory-decrease")
            .mapValues(value -> processDecrease(value))
            .filter((key, value) -> isLowStock(value))
            .to("low-stock-alerts", Produced.with(Serdes.String(), Serdes.String()));

        // Process increases — update availability
        branches.get("inventory-increase")
            .mapValues(value -> processIncrease(value))
            .to("inventory-updates", Produced.with(Serdes.String(), Serdes.String()));
    }

    private String processDecrease(String event) {
        return "PROCESSED:" + event;
    }

    private String processIncrease(String event) {
        return "PROCESSED:" + event;
    }

    private boolean isLowStock(String processedEvent) {
        return processedEvent.contains("LOW_STOCK");
    }
}
```

---

## 4. Windowing Operations {#windowing}

```java
// src/main/java/com/example/inventory/streams/WindowedAggregationStream.java
package com.example.inventory.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class WindowedAggregationStream {

    @Autowired
    public void buildPipeline(StreamsBuilder builder) {
        KStream<String, Long> salesStream = builder.stream(
            "sale-events",
            Consumed.with(Serdes.String(), Serdes.Long())
        );

        // ===== TUMBLING WINDOW =====
        // Non-overlapping, fixed-size windows (e.g., hourly totals)
        salesStream
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .reduce(Long::sum, Materialized.as("hourly-sales-store"))
            .toStream()
            .map((windowedKey, total) -> {
                String key = windowedKey.key();
                long windowStart = windowedKey.window().start();
                long windowEnd = windowedKey.window().end();
                String value = String.format(
                    "{\"product\":\"%s\",\"total\":%d,\"windowStart\":%d,\"windowEnd\":%d}",
                    key, total, windowStart, windowEnd
                );
                return new org.apache.kafka.streams.KeyValue<>(key, value);
            })
            .to("hourly-sales-aggregates");

        // ===== SLIDING WINDOW =====
        // Overlapping windows that advance by event time
        // (timeDifference parameter: max time diff between events in same window)
        salesStream
            .groupByKey()
            .windowedBy(SlidingWindows.ofTimeDifferenceAndGrace(
                Duration.ofMinutes(5),   // window covers events within 5 min of each other
                Duration.ofSeconds(30)   // grace period for late events
            ))
            .count(Materialized.as("sliding-sales-count"))
            .toStream()
            .map((windowedKey, count) ->
                new org.apache.kafka.streams.KeyValue<>(
                    windowedKey.key(),
                    "SLIDING:" + count + " sales in window"
                )
            )
            .to("sliding-sales-stats");

        // ===== SESSION WINDOW =====
        // Dynamically sized windows based on activity gaps
        // Perfect for user session analytics
        KStream<String, String> userEvents = builder.stream("user-events");

        userEvents
            .groupByKey(Grouped.with(Serdes.String(), Serdes.String()))
            .windowedBy(SessionWindows.ofInactivityGapAndGrace(
                Duration.ofMinutes(30),  // New session if 30-min gap
                Duration.ofMinutes(5)    // Grace period
            ))
            .count(Materialized.as("session-event-count"))
            .toStream()
            .map((windowedKey, count) -> {
                long duration = windowedKey.window().end() - windowedKey.window().start();
                return new org.apache.kafka.streams.KeyValue<>(
                    windowedKey.key(),
                    String.format("{\"user\":\"%s\",\"events\":%d,\"duration_ms\":%d}",
                        windowedKey.key(), count, duration)
                );
            })
            .to("user-session-stats");
    }
}
```

---

## 5. Kafka Streams Joins {#joins}

```java
// src/main/java/com/example/inventory/streams/OrderEnrichmentStream.java
package com.example.inventory.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.time.Duration;

@Component
public class OrderEnrichmentStream {

    @Autowired
    public void buildPipeline(StreamsBuilder builder) {

        // ===== STREAM-STREAM JOIN =====
        // Join two event streams within a time window
        KStream<String, String> orderStream = builder.stream("orders");
        KStream<String, String> paymentStream = builder.stream("payments");

        KStream<String, String> paidOrders = orderStream.join(
            paymentStream,
            (order, payment) -> {
                // Both order and payment arrived within the window
                return String.format("{\"order\":%s,\"payment\":%s}", order, payment);
            },
            JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10)),
            StreamJoined.with(Serdes.String(), Serdes.String(), Serdes.String())
        );

        paidOrders.to("paid-orders");

        // ===== STREAM-TABLE JOIN =====
        // Enrich stream events with table (KTable) data
        KStream<String, String> inventoryEvents = builder.stream("inventory-events");

        KTable<String, String> productTable = builder.table(
            "products",
            Consumed.with(Serdes.String(), Serdes.String()),
            Materialized.as("product-table")
        );

        KStream<String, String> enrichedInventory = inventoryEvents.join(
            productTable,
            (inventoryEvent, productData) -> {
                // inventoryEvent: "DECREASE,5"
                // productData: "{\"name\":\"Laptop\",\"category\":\"Electronics\"}"
                return String.format(
                    "{\"event\":%s,\"product\":%s}",
                    inventoryEvent, productData
                );
            }
        );

        enrichedInventory.to("enriched-inventory-events");

        // ===== TABLE-TABLE JOIN =====
        // Join two tables (both are change-logs)
        KTable<String, String> customerTable = builder.table("customers");
        KTable<String, String> accountTable = builder.table("accounts");

        KTable<String, String> customerWithAccount = customerTable.join(
            accountTable,
            (customer, account) -> customer + "|" + account
        );

        customerWithAccount.toStream().to("customer-accounts");

        // ===== LEFT JOIN =====
        // Include all orders even if payment not received
        KStream<String, String> allOrders = orderStream.leftJoin(
            paymentStream,
            (order, payment) -> {
                if (payment == null) {
                    return String.format("{\"order\":%s,\"paid\":false}", order);
                }
                return String.format("{\"order\":%s,\"paid\":true,\"payment\":%s}",
                    order, payment);
            },
            JoinWindows.ofTimeDifferenceWithNoGrace(Duration.ofMinutes(10))
        );

        allOrders.to("all-orders-with-payment-status");
    }
}
```

---

## 6. State Stores {#state-stores}

```java
// src/main/java/com/example/inventory/streams/InventoryStateStore.java
package com.example.inventory.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.state.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.config.StreamsBuilderFactoryBean;
import org.springframework.stereotype.Component;

@Component
public class InventoryStateStore {

    private static final String INVENTORY_STORE = "inventory-store";

    @Autowired
    public void buildPipeline(StreamsBuilder builder) {

        // Define a persistent state store
        Materialized<String, Long, KeyValueStore<org.apache.kafka.common.utils.Bytes, byte[]>>
            materializedStore = Materialized.<String, Long, KeyValueStore<
                org.apache.kafka.common.utils.Bytes, byte[]>>as(INVENTORY_STORE)
                .withKeySerde(Serdes.String())
                .withValueSerde(Serdes.Long())
                .withLoggingEnabled(java.util.Map.of(
                    "cleanup.policy", "compact",
                    "min.insync.replicas", "1"
                ));

        // Aggregate inventory levels into the state store
        builder.stream("inventory-events",
                    Consumed.with(Serdes.String(), Serdes.Long()))
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Long()))
            .aggregate(
                () -> 0L,                                    // Initial value
                (productId, delta, currentStock) -> {       // Aggregation logic
                    long newStock = currentStock + delta;
                    return Math.max(0, newStock);            // Never negative
                },
                materializedStore
            )
            .toStream()
            .to("current-inventory");
    }
}
```

```java
// src/main/java/com/example/inventory/service/InventoryQueryService.java
package com.example.inventory.service;

import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StoreQueryParameters;
import org.apache.kafka.streams.state.*;
import org.springframework.kafka.config.StreamsBuilderFactoryBean;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
public class InventoryQueryService {

    private final StreamsBuilderFactoryBean streamsBuilderFactoryBean;
    private static final String INVENTORY_STORE = "inventory-store";

    public InventoryQueryService(StreamsBuilderFactoryBean streamsBuilderFactoryBean) {
        this.streamsBuilderFactoryBean = streamsBuilderFactoryBean;
    }

    /**
     * Interactive queries — read state from Kafka Streams state store
     * without going back to Kafka
     */
    public Long getInventoryLevel(String productId) {
        KafkaStreams streams = streamsBuilderFactoryBean.getKafkaStreams();
        if (streams == null) {
            throw new RuntimeException("Kafka Streams not initialized");
        }

        ReadOnlyKeyValueStore<String, Long> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                INVENTORY_STORE,
                QueryableStoreTypes.keyValueStore()
            )
        );

        Long level = store.get(productId);
        return level != null ? level : 0L;
    }

    /**
     * Get all product inventory levels
     */
    public Map<String, Long> getAllInventoryLevels() {
        KafkaStreams streams = streamsBuilderFactoryBean.getKafkaStreams();
        ReadOnlyKeyValueStore<String, Long> store = streams.store(
            StoreQueryParameters.fromNameAndType(
                INVENTORY_STORE,
                QueryableStoreTypes.keyValueStore()
            )
        );

        Map<String, Long> levels = new HashMap<>();
        try (KeyValueIterator<String, Long> iterator = store.all()) {
            iterator.forEachRemaining(kv -> levels.put(kv.key, kv.value));
        }
        return levels;
    }

    /**
     * Get products with stock below threshold
     */
    public List<String> getLowStockProducts(long threshold) {
        return getAllInventoryLevels().entrySet().stream()
            .filter(entry -> entry.getValue() <= threshold)
            .map(Map.Entry::getKey)
            .toList();
    }

    /**
     * Query windowed store for time-based aggregations
     */
    public Map<Long, Long> getHourlySalesForProduct(String productId) {
        KafkaStreams streams = streamsBuilderFactoryBean.getKafkaStreams();
        ReadOnlyWindowStore<String, Long> windowStore = streams.store(
            StoreQueryParameters.fromNameAndType(
                "hourly-sales-store",
                QueryableStoreTypes.windowStore()
            )
        );

        Map<Long, Long> hourlySales = new HashMap<>();
        long now = System.currentTimeMillis();
        long oneDayAgo = now - 24 * 60 * 60 * 1000;

        try (WindowStoreIterator<Long> iterator = windowStore.fetch(
                productId, oneDayAgo, now)) {
            iterator.forEachRemaining(entry ->
                hourlySales.put(entry.key, entry.value));
        }

        return hourlySales;
    }
}
```

---

## 7. Dead Letter Topics and Error Handling {#error-handling}

```java
// src/main/java/com/example/inventory/error/DeadLetterTopicHandler.java
package com.example.inventory.error;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.support.ExponentialBackOffWithMaxRetries;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Map;

@Configuration
public class DeadLetterTopicHandler {

    private static final Logger log = LoggerFactory.getLogger(DeadLetterTopicHandler.class);

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public DeadLetterTopicHandler(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * Configure error handler with DLT (Dead Letter Topic) publishing
     */
    @Bean
    public DefaultErrorHandler errorHandler() {
        // Custom DLT publisher with metadata
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            // Compute DLT topic name from original topic
            (record, ex) -> {
                log.error("Failed to process record from topic={}, partition={}, offset={}",
                    record.topic(), record.partition(), record.offset(), ex);

                // Add error metadata as headers
                String dltTopic = record.topic() + ".DLT";
                return new ProducerRecord<>(
                    dltTopic,
                    record.partition(),
                    record.key(),
                    record.value()
                );
            }
        );

        // Add error information as headers
        recoverer.setHeadersFunction((consumerRecord, exception) ->
            new org.apache.kafka.common.header.internals.RecordHeaders()
                .add("X-Error-Message", exception.getMessage().getBytes())
                .add("X-Error-Class", exception.getClass().getName().getBytes())
                .add("X-Original-Topic", consumerRecord.topic().getBytes())
                .add("X-Original-Partition",
                    String.valueOf(consumerRecord.partition()).getBytes())
                .add("X-Original-Offset",
                    String.valueOf(consumerRecord.offset()).getBytes())
        );

        // Retry with exponential backoff before sending to DLT
        ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(3);
        backOff.setInitialInterval(1000);  // 1 second
        backOff.setMultiplier(2.0);        // Double each time
        backOff.setMaxInterval(10000);     // Max 10 seconds

        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);

        // Don't retry these — they will always fail
        errorHandler.addNotRetryableExceptions(
            java.io.InvalidClassException.class,
            com.fasterxml.jackson.core.JsonParseException.class
        );

        return errorHandler;
    }
}
```

```java
// src/main/java/com/example/inventory/consumer/InventoryEventConsumer.java
package com.example.inventory.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Component
public class InventoryEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(InventoryEventConsumer.class);

    /**
     * Process inventory events with manual acknowledgment
     */
    @KafkaListener(
        topics = "${kafka.topics.inventory-events}",
        groupId = "inventory-processor",
        concurrency = "3"  // 3 concurrent consumer threads
    )
    public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            log.debug("Processing event: topic={}, partition={}, offset={}",
                record.topic(), record.partition(), record.offset());

            processInventoryEvent(record.value());

            ack.acknowledge(); // Manual ack after successful processing

        } catch (RetryableException e) {
            log.warn("Retryable error for offset {}: {}", record.offset(), e.getMessage());
            throw e; // Spring Kafka will retry based on error handler
        } catch (Exception e) {
            log.error("Fatal error processing event, sending to DLT", e);
            throw e; // DLT handler will catch this
        }
    }

    /**
     * Consume from Dead Letter Topic for monitoring/reprocessing
     */
    @KafkaListener(
        topics = "${kafka.topics.inventory-events}.DLT",
        groupId = "dlt-monitor"
    )
    public void consumeDlt(ConsumerRecord<String, String> record) {
        String errorMessage = record.headers().lastHeader("X-Error-Message") != null
            ? new String(record.headers().lastHeader("X-Error-Message").value())
            : "Unknown error";

        log.error("DLT message: topic={}, offset={}, error={}",
            record.topic(), record.offset(), errorMessage);

        // Alert on-call, store for analysis, etc.
        saveDltRecord(record, errorMessage);
    }

    /**
     * Batch consumer for high-throughput scenarios
     */
    @KafkaListener(
        topics = "${kafka.topics.inventory-events}",
        groupId = "batch-processor",
        containerFactory = "batchKafkaListenerContainerFactory"
    )
    public void consumeBatch(java.util.List<ConsumerRecord<String, String>> records,
                              Acknowledgment ack) {
        log.info("Processing batch of {} records", records.size());
        try {
            processBatch(records);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Batch processing failed, will retry", e);
            ack.nack(0, java.time.Duration.ofSeconds(5)); // Nack from first record, retry in 5s
        }
    }

    private void processInventoryEvent(String event) {
        // Business logic here
    }

    private void saveDltRecord(ConsumerRecord<String, String> record, String error) {
        // Save to DB for analysis
    }

    private void processBatch(java.util.List<ConsumerRecord<String, String>> records) {
        // Batch processing logic
    }

    static class RetryableException extends RuntimeException {
        RetryableException(String msg) { super(msg); }
    }
}
```

---

## 8. Exactly-Once Semantics {#eos}

```java
// src/main/java/com/example/inventory/config/ExactlyOnceConfig.java
package com.example.inventory.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;
import org.springframework.kafka.transaction.KafkaTransactionManager;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class ExactlyOnceConfig {

    @Bean
    public ProducerFactory<String, Object> exactlyOnceProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            "io.confluent.kafka.serializers.KafkaAvroSerializer");

        // Exactly-once producer settings
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,
            "inventory-producer-${spring.application.instance-id:0}");

        DefaultKafkaProducerFactory<String, Object> factory =
            new DefaultKafkaProducerFactory<>(config);
        factory.setTransactionIdPrefix("inv-");
        return factory;
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public KafkaTransactionManager<String, Object> kafkaTransactionManager(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTransactionManager<>(producerFactory);
    }
}
```

```java
// src/main/java/com/example/inventory/service/TransactionalInventoryService.java
package com.example.inventory.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class TransactionalInventoryService {

    private static final Logger log = LoggerFactory.getLogger(TransactionalInventoryService.class);

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final InventoryRepository inventoryRepository;

    public TransactionalInventoryService(KafkaTemplate<String, Object> kafkaTemplate,
                                          InventoryRepository inventoryRepository) {
        this.kafkaTemplate = kafkaTemplate;
        this.inventoryRepository = inventoryRepository;
    }

    /**
     * Exactly-once: DB update and Kafka publish in same transaction.
     * Uses Kafka transactions — NOT JPA transactions.
     * For both DB + Kafka, you need a distributed transaction or outbox pattern.
     */
    public void updateInventoryWithExactlyOnce(String productId, long delta) {
        kafkaTemplate.executeInTransaction(operations -> {
            // This entire block is one Kafka transaction
            operations.send("inventory-updates",
                productId,
                createUpdateEvent(productId, delta)
            );

            operations.send("inventory-events",
                productId,
                createAuditEvent(productId, delta)
            );

            // If any send fails, the whole transaction is aborted
            // Consumers with isolation.level=read_committed only see committed messages
            return null;
        });
    }

    /**
     * Outbox Pattern: write to DB and Kafka atomically.
     * This is the recommended approach for exactly-once cross-system semantics.
     */
    @Transactional  // Database transaction
    public void updateInventoryWithOutbox(String productId, long delta) {
        // 1. Update inventory in database
        inventoryRepository.updateStock(productId, delta);

        // 2. Write event to outbox table (same DB transaction)
        OutboxEvent event = new OutboxEvent(
            productId,
            "INVENTORY_UPDATED",
            createUpdateEvent(productId, delta)
        );
        inventoryRepository.saveOutboxEvent(event);

        // 3. A separate outbox relay process reads the table and publishes to Kafka
        // (Debezium CDC is ideal for this)
    }

    private String createUpdateEvent(String productId, long delta) {
        return String.format("{\"productId\":\"%s\",\"delta\":%d,\"ts\":%d}",
            productId, delta, System.currentTimeMillis());
    }

    private String createAuditEvent(String productId, long delta) {
        return String.format("{\"event\":\"STOCK_CHANGE\",\"productId\":\"%s\",\"delta\":%d}",
            productId, delta);
    }

    interface InventoryRepository {
        void updateStock(String productId, long delta);
        void saveOutboxEvent(OutboxEvent event);
    }

    record OutboxEvent(String aggregateId, String eventType, String payload) {}
}
```

---

## 9. Apache Avro with Schema Registry {#avro}

```json
// src/main/avro/inventory_event.avsc
{
  "namespace": "com.example.inventory.avro",
  "type": "record",
  "name": "InventoryEvent",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventType", "type": {"type": "enum", "name": "EventType",
      "symbols": ["STOCK_DECREASED", "STOCK_INCREASED", "STOCK_RESERVED", "RESERVATION_RELEASED"]}},
    {"name": "productId", "type": "string"},
    {"name": "warehouseId", "type": "string"},
    {"name": "quantity", "type": "long"},
    {"name": "previousStock", "type": "long"},
    {"name": "currentStock", "type": "long"},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "metadata", "type": {"type": "map", "values": "string"}, "default": {}}
  ]
}
```

```json
// src/main/avro/product.avsc — schema for KTable
{
  "namespace": "com.example.inventory.avro",
  "type": "record",
  "name": "Product",
  "fields": [
    {"name": "productId", "type": "string"},
    {"name": "name", "type": "string"},
    {"name": "category", "type": "string"},
    {"name": "minimumStock", "type": "long", "default": 10},
    {"name": "maximumStock", "type": "long", "default": 1000},
    {"name": "version", "type": "int", "default": 1}
  ]
}
```

```java
// src/main/java/com/example/inventory/avro/InventoryEventProducer.java
package com.example.inventory.avro;

import com.example.inventory.avro.InventoryEvent;
import com.example.inventory.avro.EventType;
import io.confluent.kafka.serializers.KafkaAvroSerializer;
import io.confluent.kafka.serializers.AbstractKafkaSchemaSerDeConfig;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

@Component
public class InventoryEventProducer {

    private final KafkaTemplate<String, InventoryEvent> kafkaTemplate;

    @Value("${kafka.topics.inventory-events}")
    private String inventoryEventsTopic;

    public InventoryEventProducer(KafkaTemplate<String, InventoryEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishStockDecrease(String productId, String warehouseId,
                                      long quantity, long previousStock, long currentStock) {
        InventoryEvent event = InventoryEvent.newBuilder()
            .setEventId(UUID.randomUUID().toString())
            .setEventType(EventType.STOCK_DECREASED)
            .setProductId(productId)
            .setWarehouseId(warehouseId)
            .setQuantity(quantity)
            .setPreviousStock(previousStock)
            .setCurrentStock(currentStock)
            .setTimestamp(Instant.now().toEpochMilli())
            .setMetadata(Map.of("source", "inventory-service", "version", "1.0"))
            .build();

        kafkaTemplate.send(inventoryEventsTopic, productId, event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    throw new RuntimeException("Failed to publish inventory event", ex);
                }
            });
    }
}
```

### Schema Evolution (Backward Compatibility)

```json
// v2 of inventory_event.avsc — BACKWARD COMPATIBLE (new optional field with default)
{
  "namespace": "com.example.inventory.avro",
  "type": "record",
  "name": "InventoryEvent",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventType", "type": {"type": "enum", "name": "EventType",
      "symbols": ["STOCK_DECREASED", "STOCK_INCREASED", "STOCK_RESERVED",
                  "RESERVATION_RELEASED", "STOCK_ADJUSTED"]}},
    {"name": "productId", "type": "string"},
    {"name": "warehouseId", "type": "string"},
    {"name": "quantity", "type": "long"},
    {"name": "previousStock", "type": "long"},
    {"name": "currentStock", "type": "long"},
    {"name": "timestamp", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "metadata", "type": {"type": "map", "values": "string"}, "default": {}},
    {"name": "correlationId", "type": ["null", "string"], "default": null},
    {"name": "userId", "type": ["null", "string"], "default": null}
  ]
}
```

---

## 10. CQRS with Kafka and Event Sourcing {#cqrs}

```java
// src/main/java/com/example/inventory/cqrs/InventoryCommandService.java
package com.example.inventory.cqrs;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import java.util.UUID;

/**
 * COMMAND side: accepts commands, validates, emits events to Kafka
 */
@Service
public class InventoryCommandService {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public InventoryCommandService(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void decreaseStock(String productId, long quantity, String orderId) {
        // Validate (could read from query side or validate locally)
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");

        String event = buildEvent("STOCK_DECREASED", productId, quantity, orderId);
        kafkaTemplate.send("inventory-events", productId, event);
    }

    public void increaseStock(String productId, long quantity, String referenceId) {
        String event = buildEvent("STOCK_INCREASED", productId, quantity, referenceId);
        kafkaTemplate.send("inventory-events", productId, event);
    }

    public void reserveStock(String productId, long quantity, String orderId) {
        String event = buildEvent("STOCK_RESERVED", productId, quantity, orderId);
        kafkaTemplate.send("inventory-events", productId, event);
    }

    private String buildEvent(String type, String productId, long quantity, String ref) {
        return String.format(
            "{\"eventId\":\"%s\",\"type\":\"%s\",\"productId\":\"%s\",\"quantity\":%d,\"ref\":\"%s\",\"ts\":%d}",
            UUID.randomUUID(), type, productId, quantity, ref, System.currentTimeMillis()
        );
    }
}
```

```java
// src/main/java/com/example/inventory/cqrs/InventoryProjection.java
package com.example.inventory.cqrs;

import jakarta.persistence.*;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

/**
 * QUERY side: consumes events from Kafka, builds a read-optimized view in DB
 */
@Component
public class InventoryProjection {

    private final InventoryReadRepository readRepository;
    private final ObjectMapper objectMapper;

    public InventoryProjection(InventoryReadRepository readRepository,
                                ObjectMapper objectMapper) {
        this.readRepository = readRepository;
        this.objectMapper = objectMapper;
    }

    @KafkaListener(topics = "inventory-events", groupId = "inventory-projection")
    @Transactional
    public void on(String eventJson) throws Exception {
        Map<String, Object> event = objectMapper.readValue(eventJson, Map.class);
        String type = (String) event.get("type");
        String productId = (String) event.get("productId");
        long quantity = ((Number) event.get("quantity")).longValue();

        InventoryReadModel model = readRepository.findByProductId(productId)
            .orElse(new InventoryReadModel(productId, 0L, 0L));

        switch (type) {
            case "STOCK_INCREASED" -> model.setCurrentStock(model.getCurrentStock() + quantity);
            case "STOCK_DECREASED" -> model.setCurrentStock(
                Math.max(0, model.getCurrentStock() - quantity));
            case "STOCK_RESERVED" -> {
                model.setReservedStock(model.getReservedStock() + quantity);
                model.setAvailableStock(model.getCurrentStock() - model.getReservedStock());
            }
        }

        model.setLastUpdated(System.currentTimeMillis());
        readRepository.save(model);
    }

    interface InventoryReadRepository {
        java.util.Optional<InventoryReadModel> findByProductId(String productId);
        void save(InventoryReadModel model);
    }

    @Entity
    @Table(name = "inventory_read_model")
    static class InventoryReadModel {
        @Id
        private String productId;
        private long currentStock;
        private long reservedStock;
        private long availableStock;
        private long lastUpdated;

        public InventoryReadModel() {}
        public InventoryReadModel(String productId, long currentStock, long reservedStock) {
            this.productId = productId;
            this.currentStock = currentStock;
            this.reservedStock = reservedStock;
            this.availableStock = currentStock - reservedStock;
        }

        public String getProductId() { return productId; }
        public long getCurrentStock() { return currentStock; }
        public void setCurrentStock(long s) { this.currentStock = s; }
        public long getReservedStock() { return reservedStock; }
        public void setReservedStock(long s) { this.reservedStock = s; }
        public long getAvailableStock() { return availableStock; }
        public void setAvailableStock(long s) { this.availableStock = s; }
        public long getLastUpdated() { return lastUpdated; }
        public void setLastUpdated(long t) { this.lastUpdated = t; }
    }
}
```

---

## 11. Real Example: Real-Time Inventory Tracking with Kafka Streams {#inventory}

```java
// src/main/java/com/example/inventory/streams/InventoryTrackingTopology.java
package com.example.inventory.streams;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.streams.state.Stores;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class InventoryTrackingTopology {

    private static final long LOW_STOCK_THRESHOLD = 10;
    private static final long CRITICAL_STOCK_THRESHOLD = 3;

    @Autowired
    public void buildTopology(StreamsBuilder builder) {

        // ===== INPUT STREAMS =====
        KStream<String, String> inventoryEvents = builder.stream("inventory-events");
        KStream<String, String> orderEvents = builder.stream("order-events");
        KTable<String, String> products = builder.table("products");
        KTable<String, String> warehouses = builder.table("warehouses");

        // ===== RUNNING INVENTORY TOTALS =====
        KTable<String, Long> inventoryLevels = inventoryEvents
            .mapValues(event -> parseQuantityDelta(event))
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Long()))
            .reduce(
                Long::sum,
                Materialized.<String, Long>as(
                    Stores.persistentKeyValueStore("inventory-levels"))
                    .withKeySerde(Serdes.String())
                    .withValueSerde(Serdes.Long())
            );

        // ===== ENRICH WITH PRODUCT DATA =====
        KStream<String, String> enrichedLevels = inventoryLevels
            .toStream()
            .join(
                products,
                (stockLevel, productData) ->
                    enrichInventoryData(stockLevel, productData)
            );

        enrichedLevels.to("enriched-inventory");

        // ===== LOW STOCK DETECTION =====
        KStream<String, String> lowStockStream = inventoryLevels.toStream()
            .filter((productId, stock) -> stock != null && stock <= LOW_STOCK_THRESHOLD)
            .mapValues((productId, stock) -> buildAlertMessage(productId, stock));

        lowStockStream.to("low-stock-alerts");

        // ===== CRITICAL STOCK ALERTS =====
        inventoryLevels.toStream()
            .filter((productId, stock) -> stock != null && stock <= CRITICAL_STOCK_THRESHOLD)
            .mapValues((productId, stock) ->
                String.format("{\"severity\":\"CRITICAL\",\"productId\":\"%s\",\"stock\":%d}",
                    productId, stock))
            .to("critical-stock-alerts");

        // ===== ORDER FULFILLMENT CHECK =====
        // When order arrives, check if inventory is sufficient
        KStream<String, String> fulfillmentResults = orderEvents
            .join(
                inventoryLevels,
                (order, currentStock) -> checkFulfillment(order, currentStock),
                Joined.with(Serdes.String(), Serdes.String(), Serdes.Long())
            );

        // Route to appropriate topic based on result
        Map<String, KStream<String, String>> branches = fulfillmentResults
            .split(Named.as("fulfillment-"))
            .branch((k, v) -> v.contains("\"fulfillable\":true"), Branched.as("ok"))
            .branch((k, v) -> v.contains("\"fulfillable\":false"), Branched.as("backorder"))
            .defaultBranch(Branched.as("error"));

        branches.get("fulfillment-ok").to("fulfillable-orders");
        branches.get("fulfillment-backorder").to("backorder-orders");

        // ===== HOURLY INVENTORY SNAPSHOT =====
        inventoryLevels.toStream()
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Long()))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .reduce(
                (a, b) -> b,  // Keep latest value in window
                Materialized.as("hourly-inventory-snapshots")
            )
            .toStream()
            .map((windowedKey, level) -> new KeyValue<>(
                windowedKey.key(),
                String.format("{\"productId\":\"%s\",\"stock\":%d,\"hour\":%d}",
                    windowedKey.key(), level, windowedKey.window().start())
            ))
            .to("inventory-snapshots");

        // ===== REORDER RECOMMENDATIONS =====
        inventoryLevels.toStream()
            .join(
                products,
                (stock, productData) -> buildReorderRecommendation(stock, productData),
                Joined.with(Serdes.String(), Serdes.Long(), Serdes.String())
            )
            .filter((productId, recommendation) -> recommendation != null)
            .to("reorder-recommendations");
    }

    private long parseQuantityDelta(String event) {
        try {
            // Parse delta from event JSON: {"type":"DECREASE","quantity":5}
            if (event.contains("\"DECREASE\"")) {
                int qty = extractQuantity(event);
                return -qty;
            } else if (event.contains("\"INCREASE\"")) {
                return extractQuantity(event);
            }
        } catch (Exception e) {
            System.err.println("Error parsing event: " + e.getMessage());
        }
        return 0L;
    }

    private int extractQuantity(String event) {
        int start = event.indexOf("\"quantity\":") + 11;
        int end = event.indexOf(",", start);
        if (end == -1) end = event.indexOf("}", start);
        return Integer.parseInt(event.substring(start, end).trim());
    }

    private String enrichInventoryData(long stockLevel, String productData) {
        return String.format(
            "{\"stock\":%d,\"product\":%s,\"lowStock\":%b}",
            stockLevel, productData, stockLevel <= LOW_STOCK_THRESHOLD
        );
    }

    private String buildAlertMessage(String productId, long stock) {
        String severity = stock <= CRITICAL_STOCK_THRESHOLD ? "CRITICAL" : "WARNING";
        return String.format(
            "{\"severity\":\"%s\",\"productId\":\"%s\",\"currentStock\":%d,\"threshold\":%d}",
            severity, productId, stock, LOW_STOCK_THRESHOLD
        );
    }

    private String checkFulfillment(String order, long currentStock) {
        try {
            int requiredQty = extractQuantity(order);
            boolean fulfillable = currentStock >= requiredQty;
            return String.format("{\"order\":%s,\"fulfillable\":%b,\"available\":%d}",
                order, fulfillable, currentStock);
        } catch (Exception e) {
            return String.format("{\"error\":\"%s\"}", e.getMessage());
        }
    }

    private String buildReorderRecommendation(long currentStock, String productData) {
        if (currentStock > LOW_STOCK_THRESHOLD) return null;

        return String.format(
            "{\"recommendation\":\"REORDER\",\"currentStock\":%d,\"product\":%s}",
            currentStock, productData
        );
    }
}
```

```java
// src/main/java/com/example/inventory/api/InventoryQueryController.java
package com.example.inventory.api;

import com.example.inventory.service.InventoryQueryService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.*;

@RestController
@RequestMapping("/api/inventory")
public class InventoryQueryController {

    private final InventoryQueryService queryService;

    public InventoryQueryController(InventoryQueryService queryService) {
        this.queryService = queryService;
    }

    @GetMapping("/products/{productId}/stock")
    public ResponseEntity<Map<String, Object>> getStock(@PathVariable String productId) {
        Long level = queryService.getInventoryLevel(productId);
        return ResponseEntity.ok(Map.of(
            "productId", productId,
            "currentStock", level,
            "isLow", level <= 10,
            "isCritical", level <= 3
        ));
    }

    @GetMapping("/products/low-stock")
    public ResponseEntity<List<String>> getLowStock(
            @RequestParam(defaultValue = "10") long threshold) {
        return ResponseEntity.ok(queryService.getLowStockProducts(threshold));
    }

    @GetMapping("/products/all")
    public ResponseEntity<Map<String, Long>> getAllStock() {
        return ResponseEntity.ok(queryService.getAllInventoryLevels());
    }

    @GetMapping("/products/{productId}/hourly-sales")
    public ResponseEntity<Map<Long, Long>> getHourlySales(@PathVariable String productId) {
        return ResponseEntity.ok(queryService.getHourlySalesForProduct(productId));
    }
}
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_HOST_NAME: schema-registry

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
      - schema-registry
    ports:
      - "8082:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: inventorydb
      POSTGRES_USER: inventory
      POSTGRES_PASSWORD: inventory123
    ports:
      - "5432:5432"
```

### Kafka Topic Creation Script

```bash
#!/bin/bash
# scripts/create-topics.sh

KAFKA=localhost:9092
PARTITIONS=6
REPLICATION=1

create_topic() {
    kafka-topics.sh --create \
        --bootstrap-server $KAFKA \
        --topic $1 \
        --partitions $PARTITIONS \
        --replication-factor $REPLICATION \
        --if-not-exists \
        --config "$2"
}

# High-throughput event topics
create_topic "inventory-events" "retention.ms=604800000,cleanup.policy=delete"
create_topic "order-events" "retention.ms=604800000"
create_topic "product-events" "cleanup.policy=compact"

# Output topics
create_topic "enriched-inventory" "retention.ms=86400000"
create_topic "low-stock-alerts" "retention.ms=86400000"
create_topic "critical-stock-alerts" "retention.ms=86400000"
create_topic "fulfillable-orders" "retention.ms=604800000"
create_topic "backorder-orders" "retention.ms=604800000"
create_topic "reorder-recommendations" "retention.ms=86400000"
create_topic "inventory-snapshots" "retention.ms=2592000000"  # 30 days

# DLT topics (Dead Letter Topics)
create_topic "inventory-events.DLT" "retention.ms=2592000000"
create_topic "order-events.DLT" "retention.ms=2592000000"

# Compacted reference topics (KTable sources)
create_topic "products" "cleanup.policy=compact,min.insync.replicas=1"
create_topic "warehouses" "cleanup.policy=compact"

echo "Topics created successfully"
kafka-topics.sh --list --bootstrap-server $KAFKA
```

---

## Summary

| Pattern | Tool | Use Case |
|---|---|---|
| Stream processing | Kafka Streams | Real-time transformations |
| Tumbling windows | `TimeWindows.ofSizeWithNoGrace()` | Fixed-period aggregations (hourly totals) |
| Sliding windows | `SlidingWindows.ofTimeDifference()` | Moving averages |
| Session windows | `SessionWindows.ofInactivityGap()` | User session analytics |
| Stream-stream join | `.join()` with JoinWindows | Correlate related events |
| Stream-table join | `.join(KTable)` | Enrich events with reference data |
| State store | `Materialized.as(storeName)` | Running totals, counts |
| Interactive queries | `streams.store()` | Query Kafka state without DB |
| Exactly-once | `processing.guarantee=exactly_once_v2` | Financial, inventory |
| Outbox pattern | DB table + CDC | Atomic DB+Kafka writes |
| CQRS | Separate command/query services | Scale reads independently |
| Schema Registry | Avro + Confluent SR | Schema evolution, compatibility |
| Dead letter topic | `DeadLetterPublishingRecoverer` | Error handling, replay |

### Event-Driven Best Practices

1. **Design events as immutable facts** — "StockDecreased" not "SetStock"
2. **Use event sourcing** for audit-critical data like inventory
3. **Schema Registry is mandatory** for production — prevents breaking changes
4. **Exactly-once** is expensive — use it only for financial/inventory data
5. **Always have a DLT** — never silently drop failed events
6. **Use compacted topics** for reference data (KTables)
7. **Monitor consumer lag** — it's your SLA indicator

---

## Next Part Preview

**Part 077: Observability and Distributed Tracing** — instrument your microservices with OpenTelemetry, Micrometer, Prometheus, Grafana, and distributed tracing with Zipkin/Jaeger. Learn to correlate traces across service boundaries and build SLI/SLO dashboards.
