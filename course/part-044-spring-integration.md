# Part 044: Spring Integration

## Overview

Spring Integration implements Enterprise Integration Patterns (EIP) for building messaging-driven architectures. It provides abstractions for channels, endpoints, transformers, routers, and adapters that connect disparate systems with a clean, testable API.

---

## 1. Enterprise Integration Patterns (EIP) Overview

The key patterns in Spring Integration:

```
External Source → [Inbound Adapter] → Channel → [Filter] → Channel
                                                          → [Transformer] → Channel
                                                          → [Router] → Channel A
                                                                     → Channel B
                                                                     → Channel C
                                               → [Aggregator] → Channel
                                               → [Service Activator] → External Service
                                               → [Outbound Adapter] → External Destination
```

---

## 2. Project Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-integration</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-file</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-http</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-jms</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-mail</artifactId>
    </dependency>
</dependencies>
```

---

## 3. Core Concepts: Message, MessageChannel, MessageHandler

### 3.1 Messages

```java
package com.example.integration.core;

import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;

import java.util.Map;

public class MessageExamples {

    public void createMessages() {
        // Simple message
        Message<String> textMessage = MessageBuilder
            .withPayload("Hello, Integration!")
            .build();

        System.out.println("Payload: " + textMessage.getPayload());
        System.out.println("Message ID: " + textMessage.getHeaders().getId());

        // Message with custom headers
        Message<String> messageWithHeaders = MessageBuilder
            .withPayload("Order confirmed")
            .setHeader("orderId", 12345L)
            .setHeader("priority", "HIGH")
            .setHeader("correlationId", "corr-001")
            .setHeader("source", "order-service")
            .build();

        MessageHeaders headers = messageWithHeaders.getHeaders();
        Long orderId = (Long) headers.get("orderId");
        System.out.println("Order ID from header: " + orderId);

        // Copy and add headers
        Message<String> enrichedMessage = MessageBuilder
            .fromMessage(textMessage)
            .setHeader("processed", true)
            .setHeader("timestamp", System.currentTimeMillis())
            .build();
    }
}
```

### 3.2 Message Channels

```java
package com.example.integration.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.channel.ExecutorChannel;
import org.springframework.integration.channel.PriorityChannel;
import org.springframework.integration.channel.PublishSubscribeChannel;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

@Configuration
public class ChannelConfig {

    // DirectChannel: synchronous, point-to-point
    @Bean
    public DirectChannel orderChannel() {
        return new DirectChannel();
    }

    // QueueChannel: asynchronous, buffered
    @Bean
    public QueueChannel orderQueueChannel() {
        return new QueueChannel(1000); // capacity
    }

    // PriorityChannel: ordered by priority header
    @Bean
    public PriorityChannel priorityOrderChannel() {
        return new PriorityChannel(500);
    }

    // PublishSubscribeChannel: fan-out to multiple subscribers
    @Bean
    public PublishSubscribeChannel orderEventChannel() {
        return new PublishSubscribeChannel();
    }

    // ExecutorChannel: async with thread pool
    @Bean
    public ExecutorChannel asyncProcessingChannel() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.initialize();
        return new ExecutorChannel(executor);
    }

    // Error channel
    @Bean
    public DirectChannel errorChannel() {
        return new DirectChannel();
    }
}
```

---

## 4. Integration Flows with Java DSL

### 4.1 Basic Integration Flow

```java
package com.example.integration.flow;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.MessageChannels;

@Configuration
public class BasicFlowConfig {

    private static final Logger log = LoggerFactory.getLogger(BasicFlowConfig.class);

    @Bean
    public IntegrationFlow simpleFlow() {
        return IntegrationFlow
            .from("inputChannel")                    // Start from channel
            .filter(String.class, msg -> !msg.isBlank())  // Filter empty messages
            .transform(String.class, String::toUpperCase)  // Transform
            .handle(msg -> log.info("Received: {}", msg.getPayload())) // Terminal handler
            .get();
    }

    @Bean
    public IntegrationFlow transformAndRoute() {
        return IntegrationFlow
            .from("rawDataChannel")
            .<String, String>transform(s -> s.trim().toLowerCase())
            .<String, Boolean>route(
                s -> s.startsWith("order"),
                mapping -> mapping
                    .channelMapping(true, "orderProcessingChannel")
                    .channelMapping(false, "generalProcessingChannel")
            )
            .get();
    }
}
```

---

## 5. File Adapter

### 5.1 Inbound File Adapter (reading files)

```java
package com.example.integration.file;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.Pollers;
import org.springframework.integration.file.dsl.Files;
import org.springframework.integration.file.filters.AcceptOnceFileListFilter;
import org.springframework.integration.file.filters.CompositeFileListFilter;
import org.springframework.integration.file.filters.SimplePatternFileListFilter;

import java.io.File;
import java.time.Duration;
import java.util.List;

@Configuration
public class FileInboundConfig {

    private static final String IMPORT_DIR = "/data/import";
    private static final String ARCHIVE_DIR = "/data/archive";
    private static final String ERROR_DIR = "/data/error";

    @Bean
    public IntegrationFlow fileInboundFlow() {
        CompositeFileListFilter<File> filter = new CompositeFileListFilter<>(
            List.of(
                new SimplePatternFileListFilter("*.csv"),  // Only CSV files
                new AcceptOnceFileListFilter<>()          // Process each file once
            )
        );

        return IntegrationFlow
            .from(
                Files.inboundAdapter(new File(IMPORT_DIR))
                    .filter(filter)
                    .preventDuplicates(true),
                spec -> spec.poller(
                    Pollers.fixedDelay(Duration.ofSeconds(5))
                        .maxMessagesPerPoll(10)
                )
            )
            .log("File picked up: #{payload.name}")
            .transform(Files.toStringTransformer("UTF-8"))  // File → String
            .channel("fileContentChannel")
            .get();
    }

    @Bean
    public IntegrationFlow fileContentProcessingFlow() {
        return IntegrationFlow
            .from("fileContentChannel")
            .<String>handle((payload, headers) -> {
                // Process file content (string)
                String[] lines = payload.split("\n");
                System.out.printf("Processing file with %d lines%n", lines.length);
                return payload;
            })
            .channel("processedFileChannel")
            .get();
    }
}
```

### 5.2 Outbound File Adapter (writing files)

```java
package com.example.integration.file;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.file.dsl.Files;
import org.springframework.integration.file.support.FileExistsMode;

import java.io.File;

@Configuration
public class FileOutboundConfig {

    @Bean
    public IntegrationFlow fileOutboundFlow() {
        return IntegrationFlow
            .from("outputChannel")
            .<String>transform(payload ->
                "[" + java.time.Instant.now() + "] " + payload
            )
            .handle(
                Files.outboundAdapter(new File("/data/output"))
                    .fileNameGenerator(msg -> {
                        String correlationId = (String) msg.getHeaders()
                            .getOrDefault("correlationId", "unknown");
                        return correlationId + ".txt";
                    })
                    .fileExistsMode(FileExistsMode.APPEND)
                    .appendNewLine(true)
                    .autoCreateDirectory(true)
            )
            .get();
    }
}
```

---

## 6. HTTP Gateway

### 6.1 Inbound HTTP Gateway

```java
package com.example.integration.http;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.http.dsl.Http;
import org.springframework.web.bind.annotation.RequestMethod;

@Configuration
public class HttpGatewayConfig {

    // HTTP inbound gateway: HTTP request → Integration channel
    @Bean
    public IntegrationFlow httpInboundGatewayFlow() {
        return IntegrationFlow
            .from(
                Http.inboundGateway("/api/integration/process")
                    .requestMapping(m -> m.methods(RequestMethod.POST))
                    .requestPayloadType(OrderRequest.class)
                    .replyTimeout(30_000L)  // 30 second timeout
            )
            .log("Received HTTP request")
            .channel("orderProcessingChannel")
            .get();
    }

    // HTTP outbound gateway: Integration channel → HTTP request
    @Bean
    public IntegrationFlow httpOutboundGatewayFlow() {
        return IntegrationFlow
            .from("externalCallChannel")
            .handle(
                Http.outboundGateway("https://external-api.example.com/data")
                    .httpMethod(org.springframework.http.HttpMethod.GET)
                    .expectedResponseType(String.class)
                    .requestFactory(new org.springframework.http.client
                        .SimpleClientHttpRequestFactory())
            )
            .channel("externalResponseChannel")
            .get();
    }
}

record OrderRequest(Long id, String item, int quantity, double price) {}
```

---

## 7. JMS / AMQP Channel Adapters

### 7.1 AMQP (RabbitMQ) Integration

```java
package com.example.integration.amqp;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.amqp.dsl.Amqp;
import org.springframework.integration.dsl.IntegrationFlow;

@Configuration
public class AmqpIntegrationConfig {

    // Declare RabbitMQ infrastructure
    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("orders.incoming")
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-message-ttl", 3_600_000) // 1 hour TTL
            .build();
    }

    @Bean
    public Queue orderDlq() {
        return QueueBuilder.durable("orders.dlq").build();
    }

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("orders.exchange");
    }

    @Bean
    public FanoutExchange orderDlx() {
        return new FanoutExchange("orders.dlx");
    }

    @Bean
    public Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
            .to(orderExchange())
            .with("orders.#");
    }

    @Bean
    public Binding dlqBinding() {
        return BindingBuilder.bind(orderDlq()).to(orderDlx());
    }

    // Inbound AMQP adapter: RabbitMQ → Integration channel
    @Bean
    public IntegrationFlow amqpInboundFlow(
            org.springframework.amqp.rabbit.connection.ConnectionFactory connectionFactory
    ) {
        return IntegrationFlow
            .from(
                Amqp.inboundAdapter(connectionFactory, "orders.incoming")
                    .configureContainer(c -> c
                        .concurrentConsumers(3)
                        .maxConcurrentConsumers(10)
                        .prefetchCount(100)
                    )
            )
            .log("Received from RabbitMQ")
            .<String, String>transform(payload -> "Processed: " + payload)
            .channel("processedOrderChannel")
            .get();
    }

    // Outbound AMQP adapter: Integration channel → RabbitMQ
    @Bean
    public IntegrationFlow amqpOutboundFlow(
            org.springframework.amqp.rabbit.core.RabbitTemplate rabbitTemplate
    ) {
        return IntegrationFlow
            .from("outboundAmqpChannel")
            .handle(
                Amqp.outboundAdapter(rabbitTemplate)
                    .exchangeName("orders.exchange")
                    .routingKey("orders.processed")
            )
            .get();
    }
}
```

---

## 8. Service Activator

```java
package com.example.integration.activator;

import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.Message;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class OrderServiceActivator {

    // Annotation-based service activator
    @ServiceActivator(inputChannel = "orderProcessingChannel",
                     outputChannel = "orderResultChannel")
    public String processOrder(@Payload String orderData,
                               @Header(value = "orderId", required = false) Long orderId) {
        System.out.printf("Processing order %d: %s%n", orderId, orderData);
        // Process the order
        return "Order " + orderId + " processed successfully";
    }

    // Message-aware service activator
    @ServiceActivator(inputChannel = "rawOrderChannel")
    public Message<String> processRawOrder(Message<String> message) {
        String processed = message.getPayload().toUpperCase();
        return org.springframework.messaging.support.MessageBuilder
            .withPayload(processed)
            .copyHeaders(message.getHeaders())
            .setHeader("processedAt", java.time.Instant.now().toString())
            .build();
    }
}
```

```java
package com.example.integration.flow;

import com.example.integration.activator.OrderServiceActivator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;

@Configuration
public class ServiceActivatorFlowConfig {

    @Bean
    public IntegrationFlow serviceActivatorFlow(OrderServiceActivator activator) {
        return IntegrationFlow
            .from("incomingChannel")
            .<String>handle(activator, "processOrder")
            .channel("resultChannel")
            .get();
    }
}
```

---

## 9. Transformer and Router

### 9.1 Transformers

```java
package com.example.integration.transform;

import com.example.integration.http.OrderRequest;
import org.springframework.integration.annotation.Transformer;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

import java.time.Instant;

@Component
public class OrderTransformer {

    // Annotation-based transformer
    @Transformer(inputChannel = "rawOrderChannel", outputChannel = "enrichedOrderChannel")
    public Message<OrderRequest> enrichOrder(Message<String> rawMessage) {
        String raw = rawMessage.getPayload();
        String[] parts = raw.split(",");

        OrderRequest order = new OrderRequest(
            Long.parseLong(parts[0]),
            parts[1],
            Integer.parseInt(parts[2]),
            Double.parseDouble(parts[3])
        );

        return MessageBuilder.withPayload(order)
            .copyHeaders(rawMessage.getHeaders())
            .setHeader("enrichedAt", Instant.now().toString())
            .setHeader("source", "transformer")
            .build();
    }
}
```

```java
package com.example.integration.flow;

import com.example.integration.http.OrderRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.transformer.GenericTransformer;

@Configuration
public class TransformerFlowConfig {

    @Bean
    public IntegrationFlow transformationFlow() {
        return IntegrationFlow
            .from("csvDataChannel")
            // Lambda transformer
            .<String, String[]>transform(csv -> csv.split(","))
            // Convert array to order
            .<String[], OrderRequest>transform(parts -> new OrderRequest(
                Long.parseLong(parts[0]),
                parts[1],
                Integer.parseInt(parts[2]),
                Double.parseDouble(parts[3])
            ))
            // Add metadata
            .enrichHeaders(h -> h
                .header("processedAt", java.time.Instant.now().toString())
                .headerExpression("orderId", "payload.id()")
            )
            .channel("orderEnrichedChannel")
            .get();
    }
}
```

### 9.2 Router

```java
package com.example.integration.router;

import com.example.integration.http.OrderRequest;
import org.springframework.integration.annotation.Router;
import org.springframework.stereotype.Component;

@Component
public class OrderPriorityRouter {

    @Router(inputChannel = "orderRoutingChannel")
    public String routeByPriority(OrderRequest order) {
        if (order.price() > 10_000) {
            return "premiumOrderChannel";
        } else if (order.price() > 1_000) {
            return "standardOrderChannel";
        } else {
            return "economyOrderChannel";
        }
    }
}
```

```java
package com.example.integration.flow;

import com.example.integration.http.OrderRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;

@Configuration
public class RouterFlowConfig {

    @Bean
    public IntegrationFlow routerFlow() {
        return IntegrationFlow
            .from("orderInputChannel")
            .<OrderRequest, String>route(
                order -> {
                    if (order.price() > 10_000) return "PREMIUM";
                    if (order.price() > 1_000) return "STANDARD";
                    return "ECONOMY";
                },
                mapping -> mapping
                    .channelMapping("PREMIUM", "premiumOrderChannel")
                    .channelMapping("STANDARD", "standardOrderChannel")
                    .channelMapping("ECONOMY", "economyOrderChannel")
                    .defaultOutputChannel("unknownOrderChannel")
            )
            .get();
    }

    // Header-based routing
    @Bean
    public IntegrationFlow headerRouterFlow() {
        return IntegrationFlow
            .from("headerRoutingChannel")
            .routeByException(mapping -> mapping
                .channelMapping(IllegalArgumentException.class, "validationErrorChannel")
                .channelMapping(RuntimeException.class, "runtimeErrorChannel")
            )
            .get();
    }
}
```

---

## 10. Aggregator and Splitter

### 10.1 Splitter - one message to many

```java
package com.example.integration.flow;

import com.example.integration.http.OrderRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;

import java.util.Arrays;
import java.util.List;

@Configuration
public class SplitterConfig {

    @Bean
    public IntegrationFlow orderSplitterFlow() {
        return IntegrationFlow
            .from("batchOrderChannel")
            // Split a list of orders into individual messages
            .split()  // Splits Collection/array payloads into individual messages
            .channel("singleOrderChannel")
            .get();
    }

    @Bean
    public IntegrationFlow csvSplitterFlow() {
        return IntegrationFlow
            .from("csvFileChannel")
            // Custom splitter: file content → individual lines
            .<String, List<String>>split(
                content -> Arrays.asList(content.split("\n"))
            )
            .filter(String.class, line -> !line.startsWith("#")) // Skip comment lines
            .channel("csvLineChannel")
            .get();
    }
}
```

### 10.2 Aggregator - many messages to one

```java
package com.example.integration.aggregator;

import org.springframework.integration.annotation.Aggregator;
import org.springframework.integration.annotation.CorrelationStrategy;
import org.springframework.integration.annotation.ReleaseStrategy;
import org.springframework.integration.store.MessageGroup;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class OrderAggregator {

    // Correlate messages by orderId header
    @CorrelationStrategy
    public Object correlateByOrderId(org.springframework.messaging.Message<?> message) {
        return message.getHeaders().get("orderId");
    }

    // Release when we have all 3 parts (inventory + payment + shipping)
    @ReleaseStrategy
    public boolean releaseWhenComplete(MessageGroup group) {
        return group.size() >= 3;
    }

    // Aggregate all parts into a single message
    @Aggregator(inputChannel = "orderPartsChannel",
               outputChannel = "completedOrderChannel")
    public String aggregateOrderParts(List<String> parts) {
        return String.join(" | ", parts);
    }
}
```

```java
package com.example.integration.flow;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.support.MessageBuilder;

import java.util.List;

@Configuration
public class AggregatorFlowConfig {

    @Bean
    public IntegrationFlow orderAggregatorFlow() {
        return IntegrationFlow
            .from("orderPartsInputChannel")
            .aggregate(aggregator -> aggregator
                .correlationExpression("headers['orderId']")
                .releaseStrategy(group -> group.size() >= 3)
                .outputProcessor(group -> {
                    List<String> parts = group.getMessages().stream()
                        .map(m -> (String) m.getPayload())
                        .toList();
                    return MessageBuilder
                        .withPayload(String.join("|", parts))
                        .copyHeadersIfAbsent(
                            group.getMessages().iterator().next().getHeaders()
                        )
                        .build();
                })
                .expireGroupsUponCompletion(true)
                .groupTimeout(30_000L) // 30 second timeout
                .discardChannel("incompleteOrderChannel")
            )
            .channel("completedOrderChannel")
            .get();
    }
}
```

---

## 11. Error Channels and Exception Handling

```java
package com.example.integration.error;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHandlingException;

@Configuration
public class ErrorHandlingConfig {

    private static final Logger log = LoggerFactory.getLogger(ErrorHandlingConfig.class);

    // Error channel handler - catches all unhandled errors
    @Bean
    public IntegrationFlow globalErrorHandler() {
        return IntegrationFlow
            .from("errorChannel")
            .<MessageHandlingException>handle((payload, headers) -> {
                log.error("Integration error occurred:");
                log.error("  Error: {}", payload.getMessage());
                log.error("  Cause: {}", payload.getCause() != null ?
                    payload.getCause().getMessage() : "none");

                Message<?> failedMessage = payload.getFailedMessage();
                if (failedMessage != null) {
                    log.error("  Failed payload: {}", failedMessage.getPayload());
                    log.error("  Failed headers: {}", failedMessage.getHeaders());
                }

                // Route to dead letter queue or alert system
                return null; // Don't return a reply
            })
            .get();
    }

    // Flow with error handling
    @Bean
    public IntegrationFlow processingFlowWithErrorHandling() {
        return IntegrationFlow
            .from("processingChannel")
            .<String>handle((payload, headers) -> {
                try {
                    return processPayload(payload);
                } catch (Exception e) {
                    // Wrap and re-throw - will be caught by error channel
                    throw new RuntimeException("Failed to process: " + payload, e);
                }
            })
            .channel("successChannel")
            .get();
    }

    private String processPayload(String payload) {
        if (payload == null || payload.isBlank()) {
            throw new IllegalArgumentException("Payload cannot be blank");
        }
        return payload.toUpperCase();
    }
}
```

---

## 12. Integration Testing

```java
package com.example.integration.test;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.MessageChannels;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.integration.test.mock.MockIntegration;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.test.context.ContextConfiguration;

import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@EnableIntegration
class IntegrationFlowTest {

    @Autowired
    private MessageChannel inputChannel;

    @Autowired
    private QueueChannel resultChannel;

    @Test
    void testOrderProcessingFlow() {
        // Send a message
        Message<String> message = MessageBuilder
            .withPayload("  test order  ")
            .setHeader("orderId", 42L)
            .build();

        inputChannel.send(message);

        // Receive and verify
        Message<?> received = resultChannel.receive(5000); // 5 second timeout
        assertThat(received).isNotNull();
        assertThat(received.getPayload()).isEqualTo("TEST ORDER");
        assertThat(received.getHeaders().get("orderId")).isEqualTo(42L);
    }

    @Test
    void testNullFilteringFlow() {
        // Send blank message - should be filtered
        Message<String> blankMessage = MessageBuilder
            .withPayload("   ")
            .build();

        inputChannel.send(blankMessage);

        // Should not appear in result channel
        Message<?> received = resultChannel.receive(500); // Short timeout
        assertThat(received).isNull();
    }
}
```

```java
package com.example.integration.test;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.test.mock.MockIntegration;
import org.springframework.integration.test.mock.MockMessageHandler;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.integration.test.mock.MockIntegration.mockMessageHandler;

@SpringBootTest(classes = {
    com.example.integration.flow.BasicFlowConfig.class,
    IntegrationMockTest.TestConfig.class
})
class IntegrationMockTest {

    @Autowired
    private DirectChannel inputChannel;

    @TestConfiguration
    static class TestConfig {

        @Bean
        public DirectChannel inputChannel() {
            return new DirectChannel();
        }

        @Bean
        public QueueChannel testOutputChannel() {
            return new QueueChannel();
        }

        @Bean
        public IntegrationFlow testFlow(DirectChannel inputChannel, QueueChannel testOutputChannel) {
            return IntegrationFlow
                .from(inputChannel)
                .filter(String.class, s -> !s.isBlank())
                .transform(String.class, String::toUpperCase)
                .channel(testOutputChannel)
                .get();
        }
    }

    @Autowired
    private QueueChannel testOutputChannel;

    @Test
    void shouldTransformAndFilter() {
        inputChannel.send(MessageBuilder.withPayload("hello").build());
        inputChannel.send(MessageBuilder.withPayload("  ").build()); // Should be filtered

        Message<?> result = testOutputChannel.receive(1000);
        assertThat(result).isNotNull();
        assertThat(result.getPayload()).isEqualTo("HELLO");

        // Second message filtered - nothing more
        assertThat(testOutputChannel.receive(500)).isNull();
    }
}
```

---

## 13. Real Example: File Processing Pipeline with Multiple Channels

```java
package com.example.integration.pipeline;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.MessageChannels;
import org.springframework.integration.dsl.Pollers;
import org.springframework.integration.file.dsl.Files;
import org.springframework.integration.file.filters.AcceptOnceFileListFilter;
import org.springframework.integration.file.filters.CompositeFileListFilter;
import org.springframework.integration.file.filters.SimplePatternFileListFilter;
import org.springframework.integration.file.support.FileExistsMode;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import java.io.File;
import java.time.Duration;
import java.time.Instant;
import java.util.List;

@Configuration
public class FilePipelineConfig {

    private static final Logger log = LoggerFactory.getLogger(FilePipelineConfig.class);

    // Step 1: Pick up files from inbox directory
    @Bean
    public IntegrationFlow inboxFilePickup() {
        CompositeFileListFilter<File> filter = new CompositeFileListFilter<>(List.of(
            new SimplePatternFileListFilter("orders-*.csv"),
            new AcceptOnceFileListFilter<>()
        ));

        return IntegrationFlow
            .from(
                Files.inboundAdapter(new File("/data/inbox"))
                    .filter(filter)
                    .preventDuplicates(true),
                spec -> spec.poller(Pollers.fixedDelay(Duration.ofSeconds(10)))
            )
            .enrichHeaders(h -> h
                .headerFunction("sourceFile", m -> ((File) m.getPayload()).getName())
                .headerFunction("receivedAt", m -> Instant.now().toString())
            )
            .log(msg -> "Picked up file: " + msg.getHeaders().get("sourceFile"))
            .channel("rawFileChannel")
            .get();
    }

    // Step 2: Read and split into lines
    @Bean
    public IntegrationFlow fileToLinesFlow() {
        return IntegrationFlow
            .from("rawFileChannel")
            .transform(Files.toStringTransformer("UTF-8"))
            .<String, List<String>>split(
                content -> List.of(content.split("\n"))
            )
            // Skip header line
            .filter(String.class, line -> !line.startsWith("orderId") && !line.isBlank())
            .channel("csvLineChannel")
            .get();
    }

    // Step 3: Parse CSV lines into order records
    @Bean
    public IntegrationFlow lineParsingFlow() {
        return IntegrationFlow
            .from("csvLineChannel")
            .<String, OrderRecord>transform(line -> {
                String[] parts = line.split(",");
                if (parts.length < 5) {
                    throw new IllegalArgumentException("Invalid line format: " + line);
                }
                return new OrderRecord(
                    parts[0].trim(),                // orderId
                    parts[1].trim(),                // customerId
                    parts[2].trim(),                // productId
                    Integer.parseInt(parts[3].trim()), // quantity
                    Double.parseDouble(parts[4].trim()) // amount
                );
            })
            .channel("parsedOrderChannel")
            .get();
    }

    // Step 4: Route by order amount
    @Bean
    public IntegrationFlow orderRoutingFlow() {
        return IntegrationFlow
            .from("parsedOrderChannel")
            .<OrderRecord, String>route(
                order -> {
                    if (order.amount() > 10_000) return "HIGH_VALUE";
                    if (order.amount() > 1_000) return "STANDARD";
                    return "MICRO";
                },
                mapping -> mapping
                    .channelMapping("HIGH_VALUE", "highValueOrderChannel")
                    .channelMapping("STANDARD", "standardOrderChannel")
                    .channelMapping("MICRO", "microOrderChannel")
            )
            .get();
    }

    // Step 5a: Process high-value orders with extra validation
    @Bean
    public IntegrationFlow highValueOrderFlow() {
        return IntegrationFlow
            .from("highValueOrderChannel")
            .<OrderRecord>handle((order, headers) -> {
                log.info("HIGH VALUE order: {} amount: ${}", order.orderId(), order.amount());
                // Extra validation, fraud checks, etc.
                return order;
            })
            .handle(
                Files.outboundAdapter(new File("/data/processed/high-value"))
                    .fileNameGenerator(msg -> {
                        OrderRecord o = (OrderRecord) msg.getPayload();
                        return o.orderId() + ".json";
                    })
                    .fileExistsMode(FileExistsMode.REPLACE)
                    .autoCreateDirectory(true)
            )
            .get();
    }

    // Step 5b: Process standard orders
    @Bean
    public IntegrationFlow standardOrderFlow() {
        return IntegrationFlow
            .from("standardOrderChannel")
            .<OrderRecord>handle((order, headers) -> {
                log.debug("Standard order: {}", order.orderId());
                return order;
            })
            .channel("persistenceChannel")
            .get();
    }

    // Step 5c: Batch micro orders
    @Bean
    public IntegrationFlow microOrderBatchFlow() {
        return IntegrationFlow
            .from("microOrderChannel")
            .aggregate(agg -> agg
                .correlationExpression("headers['sourceFile']") // Group by source file
                .releaseStrategy(group -> group.size() >= 100)  // Release every 100
                .groupTimeout(10_000L)  // Or after 10 seconds
                .expireGroupsUponCompletion(true)
            )
            .log(msg -> "Batched " + ((List<?>) msg.getPayload()).size() + " micro orders")
            .channel("microOrderBatchChannel")
            .get();
    }

    // Error handling flow
    @Bean
    public IntegrationFlow errorFlow() {
        return IntegrationFlow
            .from("errorChannel")
            .<org.springframework.messaging.MessageHandlingException>handle(
                (error, headers) -> {
                    log.error("Pipeline error: {} | Cause: {}",
                        error.getMessage(),
                        error.getCause() != null ? error.getCause().getMessage() : "none"
                    );

                    // Write failed record to error directory
                    if (error.getFailedMessage() != null) {
                        Object failedPayload = error.getFailedMessage().getPayload();
                        log.error("Failed payload: {}", failedPayload);
                    }

                    return null;
                }
            )
            .get();
    }
}

// Domain record
record OrderRecord(
    String orderId,
    String customerId,
    String productId,
    int quantity,
    double amount
) {}
```

### 13.1 Pipeline Monitoring

```java
package com.example.integration.monitoring;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.integration.support.management.IntegrationManagementConfigurer;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@Endpoint(id = "integration")
public class IntegrationMetricsEndpoint {

    @ReadOperation
    public Map<String, Object> integrationStats() {
        return Map.of(
            "status", "running",
            "channels", "see /actuator/integrationgraph",
            "description", "Spring Integration pipeline metrics"
        );
    }
}
```

```yaml
# application.yml - enable integration management
spring:
  integration:
    management:
      enabled: true
      statistics-enabled: true
      default-logging-enabled: false

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,integrationgraph
  endpoint:
    integrationgraph:
      enabled: true
```

---

## Summary

| Component | Pattern | Use Case |
|-----------|---------|---------|
| `DirectChannel` | Point-to-point sync | Simple routing within same thread |
| `QueueChannel` | Point-to-point async | Buffered, decoupled processing |
| `PublishSubscribeChannel` | Fan-out | Multiple consumers of same message |
| `ExecutorChannel` | Async pool | Parallel processing |
| `Transformer` | Message Translator EIP | Convert payload format/type |
| `Router` | Message Router EIP | Conditional routing by payload/header |
| `Splitter` | Splitter EIP | One message → many messages |
| `Aggregator` | Aggregator EIP | Many messages → one message |
| `Service Activator` | Endpoint EIP | Call a service with message payload |
| `File Inbound Adapter` | Channel Adapter EIP | Files as messages |
| `AMQP Adapter` | Channel Adapter EIP | RabbitMQ as message source/sink |
| `HTTP Gateway` | Messaging Gateway EIP | REST request/response over channels |
| `Error Channel` | Dead Letter EIP | Centralized error handling |

---

## Next Part Preview

**Part 045: Multi-Tenancy with Spring Boot** — We'll implement schema-per-tenant multi-tenancy with Hibernate, dynamic tenant resolution from JWT/subdomain/header, AbstractRoutingDataSource, Flyway migrations per tenant, and Spring Security integration.
