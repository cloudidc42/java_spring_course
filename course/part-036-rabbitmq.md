# Part 036: RabbitMQ with Spring Boot

## Introduction

RabbitMQ is a battle-tested, open-source message broker implementing the AMQP (Advanced Message Queuing Protocol). It decouples services, enables asynchronous processing, and provides reliable message delivery with routing, filtering, and retry semantics. Spring AMQP provides a high-level abstraction over RabbitMQ that makes integration effortless.

---

## 1. AMQP Concepts

### Core Components

```
Producer ──► Exchange ──[binding + routing key]──► Queue ──► Consumer
```

| Component | Description |
|-----------|-------------|
| **Producer** | Application that sends messages |
| **Exchange** | Receives messages from producers; routes to queues based on rules |
| **Queue** | Buffer that stores messages |
| **Binding** | Link between exchange and queue with an optional routing key |
| **Routing Key** | Label attached to message; used by exchange for routing |
| **Consumer** | Application that receives messages from queues |
| **Virtual Host (vhost)** | Logical grouping of exchanges, queues, bindings, and permissions |

### Message Properties

```
Message
├── Headers (Map<String,Object>)
├── Body (byte[])
├── Routing Key
├── Exchange
├── Content-Type
├── Content-Encoding
├── Delivery Mode (1=transient, 2=persistent)
├── Priority (0-255)
├── Correlation ID
├── Reply To
├── Expiration (TTL in ms)
└── Message ID
```

---

## 2. RabbitMQ Exchange Types

### Direct Exchange

Routes messages to queues whose binding key exactly matches the routing key.

```
Producer ──► [direct exchange] ──► binding key: "order.created" ──► order-queue
                               ──► binding key: "order.shipped" ──► shipping-queue
```

### Topic Exchange

Routes using pattern matching with wildcards:
- `*` matches exactly one word
- `#` matches zero or more words

```
Pattern: "order.*"    matches: order.created, order.updated (NOT order.status.changed)
Pattern: "order.#"    matches: order.created, order.status.changed, order.a.b.c
Pattern: "#.error"    matches: order.error, payment.error, any.path.error
```

### Fanout Exchange

Broadcasts to all bound queues regardless of routing key.

```
Producer ──► [fanout exchange] ──► queue-1 (all queues receive)
                               ──► queue-2
                               ──► queue-3
```

### Headers Exchange

Routes based on message header attributes (not routing key).

---

## 3. Spring AMQP Setup and Configuration

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- For JSON serialization -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- Test support -->
    <dependency>
        <groupId>org.springframework.amqp</groupId>
        <artifactId>spring-rabbit-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yaml

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    connection-timeout: 10000

    # Listener configuration
    listener:
      simple:
        acknowledge-mode: MANUAL        # AUTO or MANUAL
        prefetch: 10                    # Max unacked messages per consumer
        default-requeue-rejected: false # Don't requeue on rejection (send to DLQ)
        retry:
          enabled: true
          initial-interval: 1000
          multiplier: 2
          max-attempts: 3
          max-interval: 10000

    # Publisher confirms
    publisher-confirm-type: CORRELATED
    publisher-returns: true
    template:
      mandatory: true
```

### Core Configuration Class

```java
// src/main/java/com/example/messaging/config/RabbitMQConfig.java
package com.example.messaging.config;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.config.RetryInterceptorBuilder;
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.retry.RejectAndDontRequeueRecoverer;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.retry.interceptor.RetryOperationsInterceptor;

@Configuration
public class RabbitMQConfig {

    // ─── Exchange Names ────────────────────────────────────────────────────
    public static final String ORDER_EXCHANGE      = "order.exchange";
    public static final String ORDER_DLX           = "order.dlx";           // Dead Letter Exchange
    public static final String NOTIFICATION_FANOUT = "notification.fanout";

    // ─── Queue Names ───────────────────────────────────────────────────────
    public static final String ORDER_CREATED_QUEUE  = "order.created.queue";
    public static final String ORDER_SHIPPED_QUEUE  = "order.shipped.queue";
    public static final String ORDER_DLQ            = "order.dlq";          // Dead Letter Queue
    public static final String EMAIL_QUEUE          = "notification.email.queue";
    public static final String SMS_QUEUE            = "notification.sms.queue";
    public static final String PUSH_QUEUE           = "notification.push.queue";

    // ─── Routing Keys ──────────────────────────────────────────────────────
    public static final String ORDER_CREATED_KEY = "order.created";
    public static final String ORDER_SHIPPED_KEY = "order.shipped";

    // ─── Exchanges ────────────────────────────────────────────────────────

    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder.topicExchange(ORDER_EXCHANGE)
            .durable(true)
            .build();
    }

    @Bean
    public DirectExchange orderDeadLetterExchange() {
        return ExchangeBuilder.directExchange(ORDER_DLX)
            .durable(true)
            .build();
    }

    @Bean
    public FanoutExchange notificationFanoutExchange() {
        return ExchangeBuilder.fanoutExchange(NOTIFICATION_FANOUT)
            .durable(true)
            .build();
    }

    // ─── Queues ───────────────────────────────────────────────────────────

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable(ORDER_CREATED_QUEUE)
            // Route rejected messages to DLX
            .withArgument("x-dead-letter-exchange", ORDER_DLX)
            .withArgument("x-dead-letter-routing-key", ORDER_CREATED_QUEUE)
            // Message TTL: expire after 30 minutes if not consumed
            .withArgument("x-message-ttl", 1_800_000)
            // Max queue length (optional)
            .withArgument("x-max-length", 10_000)
            .build();
    }

    @Bean
    public Queue orderShippedQueue() {
        return QueueBuilder.durable(ORDER_SHIPPED_QUEUE)
            .withArgument("x-dead-letter-exchange", ORDER_DLX)
            .withArgument("x-dead-letter-routing-key", ORDER_SHIPPED_QUEUE)
            .build();
    }

    @Bean
    public Queue orderDeadLetterQueue() {
        return QueueBuilder.durable(ORDER_DLQ)
            .build();
    }

    @Bean
    public Queue emailQueue() {
        return QueueBuilder.durable(EMAIL_QUEUE).build();
    }

    @Bean
    public Queue smsQueue() {
        return QueueBuilder.durable(SMS_QUEUE).build();
    }

    @Bean
    public Queue pushQueue() {
        return QueueBuilder.durable(PUSH_QUEUE).build();
    }

    // ─── Bindings ─────────────────────────────────────────────────────────

    @Bean
    public Binding orderCreatedBinding(Queue orderCreatedQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderCreatedQueue)
            .to(orderExchange)
            .with(ORDER_CREATED_KEY);
    }

    @Bean
    public Binding orderShippedBinding(Queue orderShippedQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderShippedQueue)
            .to(orderExchange)
            .with(ORDER_SHIPPED_KEY);
    }

    @Bean
    public Binding orderDlqBinding(Queue orderDeadLetterQueue, DirectExchange orderDeadLetterExchange) {
        return BindingBuilder.bind(orderDeadLetterQueue)
            .to(orderDeadLetterExchange)
            .with(ORDER_CREATED_QUEUE);
    }

    // Fanout bindings (no routing key needed)
    @Bean
    public Binding emailFanoutBinding(Queue emailQueue, FanoutExchange notificationFanoutExchange) {
        return BindingBuilder.bind(emailQueue).to(notificationFanoutExchange);
    }

    @Bean
    public Binding smsFanoutBinding(Queue smsQueue, FanoutExchange notificationFanoutExchange) {
        return BindingBuilder.bind(smsQueue).to(notificationFanoutExchange);
    }

    @Bean
    public Binding pushFanoutBinding(Queue pushQueue, FanoutExchange notificationFanoutExchange) {
        return BindingBuilder.bind(pushQueue).to(notificationFanoutExchange);
    }

    // ─── Converters ───────────────────────────────────────────────────────

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    // ─── RabbitTemplate ───────────────────────────────────────────────────

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());

        // Publisher confirms callback
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                System.err.println("Message NOT confirmed by broker: " + cause);
                // TODO: implement retry logic or alert
            }
        });

        // Publisher returns callback (message returned when no queue matches)
        template.setReturnsCallback(returned -> {
            System.err.println("Message returned: " + returned.getMessage()
                + " replyCode=" + returned.getReplyCode()
                + " replyText=" + returned.getReplyText());
        });

        return template;
    }

    // ─── Listener Container Factory ───────────────────────────────────────

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(jsonMessageConverter());
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        factory.setPrefetchCount(10);
        factory.setDefaultRequeueRejected(false);
        return factory;
    }
}
```

---

## 4. @RabbitListener and RabbitTemplate

### Sending Messages with RabbitTemplate

```java
// src/main/java/com/example/messaging/service/OrderEventPublisher.java
package com.example.messaging.service;

import com.example.messaging.config.RabbitMQConfig;
import com.example.messaging.model.OrderCreatedEvent;
import com.example.messaging.model.OrderShippedEvent;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessageBuilder;
import org.springframework.amqp.core.MessageProperties;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

import java.util.UUID;

@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishOrderCreated(OrderCreatedEvent event) {
        String correlationId = UUID.randomUUID().toString();
        CorrelationData correlationData = new CorrelationData(correlationId);

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_CREATED_KEY,
            event,
            message -> {
                // Enrich message properties
                MessageProperties props = message.getMessageProperties();
                props.setMessageId(correlationId);
                props.setContentType(MessageProperties.CONTENT_TYPE_JSON);
                props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);  // Survive broker restart
                props.setHeader("eventType", "ORDER_CREATED");
                props.setHeader("version", "1.0");
                props.setHeader("source", "order-service");
                return message;
            },
            correlationData
        );
    }

    public void publishOrderShipped(OrderShippedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_SHIPPED_KEY,
            event
        );
    }

    // Send to fanout (broadcast to all notification queues)
    public void broadcastNotification(Object notification) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.NOTIFICATION_FANOUT,
            "",   // Routing key ignored for fanout
            notification
        );
    }

    // Request-Reply pattern
    public Object sendAndReceive(String exchange, String routingKey, Object payload) {
        return rabbitTemplate.convertSendAndReceive(exchange, routingKey, payload);
    }
}
```

### Receiving Messages with @RabbitListener

```java
// src/main/java/com/example/messaging/listener/OrderCreatedListener.java
package com.example.messaging.listener;

import com.example.messaging.config.RabbitMQConfig;
import com.example.messaging.model.OrderCreatedEvent;
import com.example.messaging.service.InventoryService;
import com.rabbitmq.client.Channel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class OrderCreatedListener {

    private static final Logger log = LoggerFactory.getLogger(OrderCreatedListener.class);

    private final InventoryService inventoryService;

    public OrderCreatedListener(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @RabbitListener(
        queues = RabbitMQConfig.ORDER_CREATED_QUEUE,
        containerFactory = "rabbitListenerContainerFactory"
    )
    public void handleOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header(required = false, name = "x-death") Object xDeath,
            Channel channel,
            Message message) throws IOException {

        long deliveryTag = message.getMessageProperties().getDeliveryTag();

        try {
            log.info("Processing order created event: orderId={}", event.getOrderId());

            // Process the event
            inventoryService.reserveItems(event.getOrderId(), event.getItems());

            // Acknowledge: message successfully processed
            channel.basicAck(deliveryTag, false);
            log.info("Order {} processed and acknowledged", event.getOrderId());

        } catch (IllegalArgumentException e) {
            // Permanent failure: reject without requeue → goes to DLQ
            log.error("Permanent failure processing order {}: {}", event.getOrderId(), e.getMessage());
            channel.basicReject(deliveryTag, false);  // false = don't requeue

        } catch (Exception e) {
            // Transient failure: decide based on retry count
            Integer retryCount = message.getMessageProperties().getXDeathHeader() != null
                ? extractRetryCount(message) : 0;

            if (retryCount < 3) {
                log.warn("Transient failure (attempt {}), requeueing: {}", retryCount + 1, e.getMessage());
                channel.basicNack(deliveryTag, false, true);   // requeue = true
            } else {
                log.error("Max retries reached for order {}, sending to DLQ", event.getOrderId());
                channel.basicReject(deliveryTag, false);        // send to DLQ
            }
        }
    }

    private int extractRetryCount(Message message) {
        Object xDeath = message.getMessageProperties().getXDeathHeader();
        if (xDeath instanceof java.util.List<?> list && !list.isEmpty()) {
            Object first = list.get(0);
            if (first instanceof java.util.Map<?, ?> map) {
                Object count = map.get("count");
                if (count instanceof Number num) {
                    return num.intValue();
                }
            }
        }
        return 0;
    }
}
```

---

## 5. Message Acknowledgment Modes

### AUTO Mode (Default — Spring-managed)

```java
// AUTO mode: Spring acks automatically when method returns normally,
// rejects on exception

@Component
public class AutoAckListener {

    @RabbitListener(queues = "my-queue", ackMode = "AUTO")
    public void handle(MyMessage message) {
        // Ack sent automatically on return
        // Nack sent automatically on exception
        processMessage(message);
    }
}
```

### MANUAL Mode (Full control)

```java
@Component
public class ManualAckListener {

    @RabbitListener(queues = "my-queue", ackMode = "MANUAL")
    public void handle(MyMessage message, Channel channel,
                       @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        try {
            processMessage(message);
            channel.basicAck(deliveryTag, false);       // Acknowledge

        } catch (RecoverableException e) {
            channel.basicNack(deliveryTag, false, true); // Nack + requeue

        } catch (Exception e) {
            channel.basicReject(deliveryTag, false);     // Reject → DLQ
        }
    }
}
```

### NONE Mode (No acknowledgment — fire and forget)

```java
@RabbitListener(queues = "my-queue", ackMode = "NONE")
public void handle(MyMessage message) {
    // No ack sent; message is auto-removed from queue
    processMessage(message);
}
```

---

## 6. Dead Letter Queue (DLQ) and Retry

### Message Lifecycle with DLQ

```
Producer → Exchange → Queue (TTL=30min, DLX=order.dlx)
                        ↓
               Consumer (fails 3 times)
                        ↓ basicReject(false)
               Dead Letter Exchange (order.dlx)
                        ↓
               Dead Letter Queue (order.dlq)
                        ↓
               DLQ Consumer (alert, investigate, reprocess)
```

### DLQ Consumer

```java
// src/main/java/com/example/messaging/listener/DeadLetterListener.java
package com.example.messaging.listener;

import com.example.messaging.config.RabbitMQConfig;
import com.example.messaging.service.AlertService;
import com.rabbitmq.client.Channel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Instant;
import java.util.List;
import java.util.Map;

@Component
public class DeadLetterListener {

    private static final Logger log = LoggerFactory.getLogger(DeadLetterListener.class);

    private final AlertService alertService;

    public DeadLetterListener(AlertService alertService) {
        this.alertService = alertService;
    }

    @RabbitListener(queues = RabbitMQConfig.ORDER_DLQ)
    public void handleDeadLetter(Message message, Channel channel,
                                 @Header("x-death") List<Map<String, Object>> xDeath)
            throws IOException {

        long deliveryTag = message.getMessageProperties().getDeliveryTag();

        try {
            String originalQueue = extractOriginalQueue(xDeath);
            String reason = extractReason(xDeath);
            int deathCount = extractDeathCount(xDeath);

            log.error("Dead letter received: queue={}, reason={}, deaths={}, body={}",
                originalQueue, reason, deathCount,
                new String(message.getBody()));

            // Alert operations team
            alertService.sendAlert(
                "DLQ Message",
                String.format("Message in DLQ from %s (died %d times): %s",
                    originalQueue, deathCount, reason)
            );

            // Ack the DLQ message (remove it)
            channel.basicAck(deliveryTag, false);

        } catch (Exception e) {
            log.error("Failed to process DLQ message", e);
            channel.basicAck(deliveryTag, false);  // Still ack to prevent loop
        }
    }

    private String extractOriginalQueue(List<Map<String, Object>> xDeath) {
        if (xDeath != null && !xDeath.isEmpty()) {
            Object queue = xDeath.get(0).get("queue");
            return queue != null ? queue.toString() : "unknown";
        }
        return "unknown";
    }

    private String extractReason(List<Map<String, Object>> xDeath) {
        if (xDeath != null && !xDeath.isEmpty()) {
            Object reason = xDeath.get(0).get("reason");
            return reason != null ? reason.toString() : "unknown";
        }
        return "unknown";
    }

    private int extractDeathCount(List<Map<String, Object>> xDeath) {
        if (xDeath != null && !xDeath.isEmpty()) {
            Object count = xDeath.get(0).get("count");
            if (count instanceof Number n) return n.intValue();
        }
        return 0;
    }
}
```

### Retry with Delay Queue Pattern

```java
// Retry with exponential backoff using delayed message exchange
// Requires rabbitmq_delayed_message_exchange plugin

@Configuration
public class RetryConfig {

    public static final String RETRY_EXCHANGE   = "order.retry.exchange";
    public static final String RETRY_QUEUE      = "order.retry.queue";

    @Bean
    public CustomExchange retryExchange() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-delayed-type", "direct");
        return new CustomExchange(RETRY_EXCHANGE, "x-delayed-message", true, false, args);
    }

    @Bean
    public Queue retryQueue() {
        return QueueBuilder.durable(RETRY_QUEUE).build();
    }

    @Bean
    public Binding retryBinding(Queue retryQueue, CustomExchange retryExchange) {
        return BindingBuilder.bind(retryQueue).to(retryExchange)
            .with(RETRY_QUEUE).noargs();
    }
}
```

```java
// src/main/java/com/example/messaging/service/RetryPublisher.java
package com.example.messaging.service;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

@Service
public class RetryPublisher {

    private final RabbitTemplate rabbitTemplate;

    public RetryPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void retryWithDelay(Object message, int retryCount) {
        long delayMs = (long) Math.pow(2, retryCount) * 1000; // 2s, 4s, 8s...

        rabbitTemplate.convertAndSend(
            RetryConfig.RETRY_EXCHANGE,
            RetryConfig.RETRY_QUEUE,
            message,
            msg -> {
                msg.getMessageProperties().setHeader("x-delay", delayMs);
                msg.getMessageProperties().setHeader("retry-count", retryCount + 1);
                return msg;
            }
        );
    }
}
```

---

## 7. Publisher Confirms and Returns

### Synchronous Confirm (wait for ack)

```java
// src/main/java/com/example/messaging/service/ConfirmedPublisher.java
package com.example.messaging.service;

import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;

import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
public class ConfirmedPublisher {

    private final RabbitTemplate rabbitTemplate;

    public ConfirmedPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public boolean publishWithConfirm(String exchange, String routingKey, Object message)
            throws InterruptedException {

        String msgId = UUID.randomUUID().toString();
        CorrelationData correlationData = new CorrelationData(msgId);

        rabbitTemplate.convertAndSend(exchange, routingKey, message, correlationData);

        // Wait up to 5 seconds for broker confirm
        CorrelationData.Confirm confirm = correlationData.getFuture().get(5, TimeUnit.SECONDS);

        if (confirm != null && confirm.isAck()) {
            return true;
        } else {
            String cause = confirm != null ? confirm.getReason() : "timeout";
            throw new RuntimeException("Message not confirmed: " + cause);
        }
    }
}
```

### Asynchronous Confirms via Callback

```java
// Configured in RabbitTemplate (see Section 3)
// publisher-confirm-type: CORRELATED enables broker acks

// The confirm callback is set on the template:
template.setConfirmCallback((correlationData, ack, cause) -> {
    if (ack) {
        log.info("Message confirmed: id={}", correlationData.getId());
        // Update outbox status, etc.
    } else {
        log.error("Message NOT acked: id={}, reason={}", correlationData.getId(), cause);
        // Retry, alert, persist for later processing
    }
});
```

---

## 8. RabbitMQ with Spring Boot Virtual Host

### Separate vhosts per environment

```yaml
# production config
spring:
  rabbitmq:
    virtual-host: /production
    username: prod-user
    password: ${RABBITMQ_PASSWORD}

# staging config
spring:
  rabbitmq:
    virtual-host: /staging
    username: staging-user
    password: ${RABBITMQ_STAGING_PASSWORD}
```

### Multiple Connection Factories (multi-vhost)

```java
// src/main/java/com/example/messaging/config/MultiVhostConfig.java
package com.example.messaging.config;

import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MultiVhostConfig {

    @Bean
    @Primary
    public ConnectionFactory primaryConnectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory("localhost");
        factory.setVirtualHost("/orders");
        factory.setUsername("order-user");
        factory.setPassword("order-pass");
        return factory;
    }

    @Bean
    public ConnectionFactory auditConnectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory("localhost");
        factory.setVirtualHost("/audit");
        factory.setUsername("audit-user");
        factory.setPassword("audit-pass");
        return factory;
    }

    @Bean
    public RabbitTemplate auditRabbitTemplate(@Qualifier("auditConnectionFactory")
                                              ConnectionFactory connectionFactory) {
        return new RabbitTemplate(connectionFactory);
    }
}
```

---

## 9. Priority Queues

### Configuration

```java
// Priority queue: higher number = higher priority
@Bean
public Queue priorityQueue() {
    return QueueBuilder.durable("priority.order.queue")
        .withArgument("x-max-priority", 10)  // Priority levels 0-10
        .build();
}
```

### Sending with Priority

```java
public void sendWithPriority(OrderCreatedEvent event, int priority) {
    rabbitTemplate.convertAndSend(
        ORDER_EXCHANGE,
        ORDER_CREATED_KEY,
        event,
        message -> {
            message.getMessageProperties().setPriority(priority);
            return message;
        }
    );
}

// Premium customer gets higher priority
public void processOrder(Order order) {
    int priority = order.getCustomerTier() == CustomerTier.PREMIUM ? 9 : 1;
    sendWithPriority(new OrderCreatedEvent(order), priority);
}
```

---

## 10. Message TTL and Expiry

```java
// Per-queue TTL (set on queue definition — see RabbitMQConfig above)
// x-message-ttl: 1_800_000  // All messages expire after 30 min

// Per-message TTL (override for individual messages)
public void sendWithTTL(Object message, long ttlMs) {
    rabbitTemplate.convertAndSend(
        ORDER_EXCHANGE, ORDER_CREATED_KEY, message,
        msg -> {
            msg.getMessageProperties().setExpiration(String.valueOf(ttlMs));
            return msg;
        }
    );
}

// Expiry triggers DLX routing (if configured)
// Without DLX, expired messages are simply dropped
```

---

## 11. Real Example: Order Processing with DLQ Retry

### Domain Models

```java
// src/main/java/com/example/messaging/model/OrderCreatedEvent.java
package com.example.messaging.model;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

public class OrderCreatedEvent {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private Instant createdAt;

    // Constructors
    public OrderCreatedEvent() {}

    public OrderCreatedEvent(String orderId, String customerId,
                              List<OrderItem> items, BigDecimal totalAmount) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.items = items;
        this.totalAmount = totalAmount;
        this.createdAt = Instant.now();
    }

    // Getters and setters
    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }

    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }

    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }

    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }

    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }
}
```

```java
// src/main/java/com/example/messaging/model/OrderItem.java
package com.example.messaging.model;

import java.math.BigDecimal;

public class OrderItem {
    private String productId;
    private int quantity;
    private BigDecimal unitPrice;

    public OrderItem() {}

    public OrderItem(String productId, int quantity, BigDecimal unitPrice) {
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }

    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }

    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
}
```

### Order REST Controller

```java
// src/main/java/com/example/messaging/controller/OrderController.java
package com.example.messaging.controller;

import com.example.messaging.model.OrderCreatedEvent;
import com.example.messaging.model.OrderItem;
import com.example.messaging.service.OrderEventPublisher;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderEventPublisher publisher;

    public OrderController(OrderEventPublisher publisher) {
        this.publisher = publisher;
    }

    @PostMapping
    public ResponseEntity<Map<String, String>> createOrder(@RequestBody CreateOrderRequest request) {
        String orderId = UUID.randomUUID().toString();

        OrderCreatedEvent event = new OrderCreatedEvent(
            orderId,
            request.getCustomerId(),
            request.getItems(),
            request.getItems().stream()
                .map(i -> i.getUnitPrice().multiply(BigDecimal.valueOf(i.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add)
        );

        publisher.publishOrderCreated(event);

        return ResponseEntity.accepted().body(Map.of(
            "orderId", orderId,
            "status", "PROCESSING"
        ));
    }
}
```

### Inventory Service (Consumer)

```java
// src/main/java/com/example/messaging/service/InventoryService.java
package com.example.messaging.service;

import com.example.messaging.model.OrderItem;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;

@Service
public class InventoryService {

    private static final Logger log = LoggerFactory.getLogger(InventoryService.class);

    // In-memory inventory for demo
    private final Map<String, Integer> inventory = new HashMap<>(Map.of(
        "PROD-001", 100,
        "PROD-002", 50,
        "PROD-003", 0    // Out of stock
    ));

    private final Random random = new Random();

    public void reserveItems(String orderId, List<OrderItem> items) {
        log.info("Reserving inventory for order: {}", orderId);

        // Simulate random transient failure (10% chance)
        if (random.nextInt(10) == 0) {
            throw new RuntimeException("Transient database error - will retry");
        }

        for (OrderItem item : items) {
            Integer available = inventory.get(item.getProductId());
            if (available == null) {
                throw new IllegalArgumentException(
                    "Product not found: " + item.getProductId()  // Permanent failure
                );
            }
            if (available < item.getQuantity()) {
                throw new IllegalArgumentException(
                    "Insufficient stock for: " + item.getProductId() // Permanent failure
                );
            }
            inventory.put(item.getProductId(), available - item.getQuantity());
            log.info("Reserved {} units of {} for order {}",
                item.getQuantity(), item.getProductId(), orderId);
        }
    }
}
```

### Full Processing Flow with DLQ

```java
// src/main/java/com/example/messaging/listener/OrderProcessingListener.java
package com.example.messaging.listener;

import com.example.messaging.config.RabbitMQConfig;
import com.example.messaging.model.OrderCreatedEvent;
import com.example.messaging.service.InventoryService;
import com.example.messaging.service.NotificationService;
import com.rabbitmq.client.Channel;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.io.IOException;

@Component
public class OrderProcessingListener {

    private static final Logger log = LoggerFactory.getLogger(OrderProcessingListener.class);
    private static final int MAX_RETRIES = 3;

    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    public OrderProcessingListener(InventoryService inventoryService,
                                   NotificationService notificationService) {
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
    }

    @RabbitListener(queues = RabbitMQConfig.ORDER_CREATED_QUEUE)
    @Transactional
    public void processOrder(OrderCreatedEvent event, Channel channel, Message message)
            throws IOException {

        long tag = message.getMessageProperties().getDeliveryTag();
        String orderId = event.getOrderId();

        try {
            log.info("[ORDER] Processing: orderId={}, customerId={}",
                orderId, event.getCustomerId());

            // Step 1: Reserve inventory
            inventoryService.reserveItems(orderId, event.getItems());

            // Step 2: Send confirmation notification
            notificationService.sendOrderConfirmation(event);

            // Step 3: Acknowledge success
            channel.basicAck(tag, false);
            log.info("[ORDER] Successfully processed: orderId={}", orderId);

        } catch (IllegalArgumentException e) {
            // Permanent failure: bad data, won't recover
            log.error("[ORDER] Permanent failure, rejecting to DLQ: orderId={}, error={}",
                orderId, e.getMessage());
            channel.basicReject(tag, false);  // → DLQ

        } catch (Exception e) {
            // Transient failure: retry with requeue
            int deathCount = getDeathCount(message);

            if (deathCount < MAX_RETRIES) {
                log.warn("[ORDER] Transient failure (attempt {}/{}), requeueing: orderId={}, error={}",
                    deathCount + 1, MAX_RETRIES, orderId, e.getMessage());
                channel.basicNack(tag, false, true);

            } else {
                log.error("[ORDER] Max retries exceeded, sending to DLQ: orderId={}", orderId);
                channel.basicReject(tag, false);
            }
        }
    }

    private int getDeathCount(Message message) {
        Object xDeath = message.getMessageProperties().getXDeathHeader();
        if (xDeath instanceof java.util.List<?> list && !list.isEmpty()) {
            Object first = list.get(0);
            if (first instanceof java.util.Map<?, ?> map) {
                Object count = map.get("count");
                if (count instanceof Number n) return n.intValue();
            }
        }
        return 0;
    }
}
```

### docker-compose.yaml for Local Development

```yaml
# docker-compose.yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI (admin/admin)
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_RABBITMQ_PORT: 5672
    depends_on:
      rabbitmq:
        condition: service_healthy

volumes:
  rabbitmq-data:
```

---

## Summary Table

| Topic | Key Class/Annotation | Notes |
|-------|---------------------|-------|
| Exchange types | `TopicExchange`, `FanoutExchange`, `DirectExchange` | Topic is most flexible |
| Queue declaration | `QueueBuilder.durable()` | Always use durable for production |
| Sending | `RabbitTemplate.convertAndSend()` | Set message converter |
| Receiving | `@RabbitListener` | Use `ackMode = "MANUAL"` for reliability |
| Ack | `channel.basicAck()` | Must ack or nack every message |
| Nack | `channel.basicNack()` | With `requeue=true` for retry |
| Reject → DLQ | `channel.basicReject(tag, false)` | DLX must be configured |
| Publisher confirms | `CorrelationData` + `setConfirmCallback` | Enable `publisher-confirm-type: CORRELATED` |
| Priority queue | `x-max-priority` argument | Higher number = higher priority |
| Message TTL | `x-message-ttl` argument or `setExpiration()` | Expired → DLX if configured |
| JSON conversion | `Jackson2JsonMessageConverter` | Register on template + factory |

---

## What's Next

**Part 037: OAuth2 and OIDC with Spring Security** — Secure your APIs with industry-standard protocols. We'll implement Authorization Code flow, JWT validation with JWKS, Keycloak integration, role-based access control, and multi-tenant applications.

---

*End of Part 036: RabbitMQ with Spring Boot*
