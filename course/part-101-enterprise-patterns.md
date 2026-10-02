# Part 101: Enterprise Integration Patterns (EIP)

## เนื้อหาในส่วนนี้
- Message Channel Patterns
- Message Routing Patterns
- Message Transformation Patterns
- Endpoint Patterns
- System Management Patterns
- Implementing EIP with Spring Integration
- Real-world enterprise scenarios

---

## 1. EIP Overview

```
Enterprise Integration Patterns (Hohpe & Woolf, 2003)
65 patterns for messaging-based integration

Core concepts:
- Message       = data packet sent between systems
- Channel       = pipe connecting sender to receiver
- Endpoint      = interface between app and messaging system
- Router        = directs messages to different channels
- Transformer   = converts message format
- Filter        = removes unwanted messages
```

## 2. Message Channel Patterns

```java
import org.springframework.integration.dsl.*;
import org.springframework.integration.channel.*;
import org.springframework.messaging.*;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.context.annotation.*;
import org.springframework.integration.annotation.*;

@Configuration
@EnableIntegration
public class ChannelConfig {
    
    // Point-to-Point Channel (one consumer gets the message)
    @Bean
    public MessageChannel orderChannel() {
        return new DirectChannel();  // Synchronous, single consumer
    }
    
    // Publish-Subscribe Channel (all subscribers get it)
    @Bean
    public MessageChannel orderEventChannel() {
        return new PublishSubscribeChannel();  // Broadcast
    }
    
    // Datatype Channel (typed messages only)
    @Bean
    public MessageChannel typedOrderChannel() {
        return new QueueChannel();  // Async, buffered
    }
    
    // Dead Letter Channel (failed messages)
    @Bean
    public MessageChannel deadLetterChannel() {
        return new QueueChannel(100);  // Buffer 100 dead letters
    }
    
    // Priority Channel (higher priority first)
    @Bean
    public PriorityChannel priorityChannel() {
        return new PriorityChannel(
            Comparator.comparingInt(m -> (int) m.getHeaders()
                .getOrDefault("priority", 0)));
    }
}

// Sending messages
@Service
public class OrderChannelPublisher {
    
    @Autowired
    @Qualifier("orderChannel")
    private MessageChannel orderChannel;
    
    @Autowired
    @Qualifier("orderEventChannel")
    private MessageChannel orderEventChannel;
    
    public void sendOrder(Order order) {
        Message<Order> message = MessageBuilder
            .withPayload(order)
            .setHeader("orderId", order.id())
            .setHeader("priority", order.isPriority() ? 10 : 5)
            .setHeader("timestamp", System.currentTimeMillis())
            .build();
        
        orderChannel.send(message);
    }
    
    public void publishOrderEvent(OrderEvent event) {
        Message<OrderEvent> message = MessageBuilder
            .withPayload(event)
            .setHeader("eventType", event.type())
            .build();
        
        orderEventChannel.send(message);
    }
}

record Order(String id, String product, double total, boolean isPriority) {}
record OrderEvent(String type, String orderId) {}
```

---

## 3. Message Router Patterns

```java
@Configuration
public class RouterConfig {
    
    // Content-Based Router: route by message content
    @Bean
    public IntegrationFlow contentBasedRouter() {
        return IntegrationFlow.from("incomingOrders")
            .<Order, String>route(
                order -> {
                    if (order.total() > 1000) return "highValueOrders";
                    if (order.total() > 100) return "mediumValueOrders";
                    return "lowValueOrders";
                },
                mapping -> mapping
                    .channelMapping("highValueOrders", "highValueChannel")
                    .channelMapping("mediumValueOrders", "mediumValueChannel")
                    .channelMapping("lowValueOrders", "lowValueChannel")
                    .defaultOutputChannel("unknownChannel")
            )
            .get();
    }
    
    // Message Filter: drop unwanted messages
    @Bean
    public IntegrationFlow messageFilter() {
        return IntegrationFlow.from("allOrders")
            .filter(Order.class, order -> order.total() > 0,
                    spec -> spec.discardChannel("invalidOrders"))
            .channel("validOrders")
            .get();
    }
    
    // Splitter: split one message into many
    @Bean
    public IntegrationFlow orderSplitter() {
        return IntegrationFlow.from("batchOrders")
            .<List<Order>>split(List.class, List::stream)  // Split list into individual items
            .channel("individualOrders")
            .get();
    }
    
    // Aggregator: collect and combine messages
    @Bean
    public IntegrationFlow orderAggregator() {
        return IntegrationFlow.from("processedOrderItems")
            .aggregate(aggregatorSpec -> aggregatorSpec
                .correlationStrategy(m -> m.getHeaders().get("batchId"))
                .releaseStrategy(group -> group.size() == 5)  // Release when 5 items
                .outputProcessor(group -> group.getMessages().stream()
                    .map(m -> (Order) m.getPayload())
                    .collect(java.util.stream.Collectors.toList()))
            )
            .channel("completedBatches")
            .get();
    }
    
    // Recipient List Router: send to multiple channels
    @Bean
    public IntegrationFlow recipientList() {
        return IntegrationFlow.from("broadcastOrders")
            .routeToRecipients(r -> r
                .recipient("auditChannel")
                .recipient("analyticsChannel")
                .recipientFlow(f -> f
                    .filter((Order o) -> o.total() > 500)
                    .channel("highValueAuditChannel"))
            )
            .get();
    }
}
```

---

## 4. Message Transformation Patterns

```java
@Configuration
public class TransformerConfig {
    
    // Message Transformer: convert payload
    @Bean
    public IntegrationFlow orderTransformer() {
        return IntegrationFlow.from("rawOrders")
            .<RawOrder, Order>transform(raw -> new Order(
                raw.orderId(),
                raw.productName(),
                raw.quantity() * raw.unitPrice(),
                raw.quantity() > 10
            ))
            .channel("processedOrders")
            .get();
    }
    
    // Enricher: add data from external source
    @Bean
    public IntegrationFlow orderEnricher(CustomerRepository customerRepo) {
        return IntegrationFlow.from("ordersWithCustomerId")
            .<Order>enrich(e -> e
                .<Order>requestPayloadExpression("payload.customerId")
                .requestChannel("customerLookupChannel")
                .propertyExpression("customerName", "payload.name")
                .propertyExpression("customerEmail", "payload.email")
            )
            .channel("enrichedOrders")
            .get();
    }
    
    // Claim Check: store large payload, pass reference
    @Bean
    public IntegrationFlow claimCheck(MessageStore messageStore) {
        return IntegrationFlow.from("largeOrders")
            .claimCheckIn(messageStore)  // Store payload, pass claim
            .channel("orderReferences")
            .get();
    }
    
    @Bean
    public IntegrationFlow claimCheckOut(MessageStore messageStore) {
        return IntegrationFlow.from("orderReferencesToProcess")
            .claimCheckOut(messageStore)  // Retrieve original payload
            .channel("retrievedOrders")
            .get();
    }
    
    @Bean
    public MessageStore messageStore() {
        return new SimpleMessageStore();
    }
}

record RawOrder(String orderId, String productName, int quantity, double unitPrice, String customerId) {}
```

---

## 5. Endpoint Patterns

```java
@Configuration
public class EndpointConfig {
    
    // Polling Consumer: poll channel at intervals
    @Bean
    public IntegrationFlow pollingConsumer() {
        return IntegrationFlow.from(
                "orderQueue",
                e -> e.poller(Pollers.fixedDelay(1000)  // Poll every 1s
                    .maxMessagesPerPoll(10)              // Process 10 at a time
                    .errorChannel("errorChannel"))
            )
            .handle(Order.class, (order, headers) -> {
                System.out.println("Processing order: " + order.id());
                return null;
            })
            .get();
    }
    
    // Event-Driven Consumer: react when message arrives
    @Bean
    public IntegrationFlow eventDrivenConsumer() {
        return IntegrationFlow.from("directOrderChannel")  // DirectChannel
            .handle(Order.class, (order, headers) -> {
                processOrder(order);
                return null;
            })
            .get();
    }
    
    // Competing Consumers: multiple consumers on same channel
    @Bean
    public IntegrationFlow competingConsumers() {
        return IntegrationFlow.from("workQueue")
            .channel(channels -> channels.executor(
                java.util.concurrent.Executors.newFixedThreadPool(5)))  // 5 concurrent consumers
            .handle(Order.class, this::processOrderConcurrently)
            .get();
    }
    
    // Message Dispatcher: service activator with router logic
    @ServiceActivator(inputChannel = "commandChannel")
    public void handleCommand(Message<Command> message) {
        Command cmd = message.getPayload();
        switch (cmd.type()) {
            case "CREATE_ORDER" -> createOrder(cmd);
            case "CANCEL_ORDER" -> cancelOrder(cmd);
            case "UPDATE_ORDER" -> updateOrder(cmd);
            default -> throw new IllegalArgumentException("Unknown command: " + cmd.type());
        }
    }
    
    private void processOrder(Order order) { System.out.println("Processing: " + order.id()); }
    private void processOrderConcurrently(Order order, MessageHeaders headers) {}
    private void createOrder(Command cmd) {}
    private void cancelOrder(Command cmd) {}
    private void updateOrder(Command cmd) {}
}

record Command(String type, String entityId, java.util.Map<String, Object> payload) {}
```

---

## 6. System Management Patterns

```java
@Configuration
public class ManagementConfig {
    
    // Wire Tap: inspect messages without disrupting flow
    @Bean
    public IntegrationFlow wireTap() {
        return IntegrationFlow.from("mainOrderFlow")
            .wireTap("auditChannel")     // Copy to audit (doesn't block main flow)
            .handle(Order.class, (order, h) -> {
                // Main processing continues
                processMainOrder(order);
                return null;
            })
            .get();
    }
    
    // Message History: track message through system
    @Bean
    public IntegrationFlow messageHistory() {
        return IntegrationFlow.from("trackedOrders")
            .enrichHeaders(h -> h.header(IntegrationMessageHeaderAccessor.CORRELATION_ID,
                                         java.util.UUID.randomUUID().toString()))
            .channel("processingChannel")
            .get();
    }
    
    // Control Bus: manage components at runtime
    @Bean
    @ServiceActivator(inputChannel = "controlChannel")
    public ExpressionControlBusFactoryBean controlBus() {
        return new ExpressionControlBusFactoryBean();
    }
    // Send "@myPoller.stop()" to controlChannel to stop a poller
    
    private void processMainOrder(Order order) {}
}

import org.springframework.integration.dsl.IntegrationFlow;
import org.springframework.integration.dsl.Pollers;
import org.springframework.integration.store.MessageStore;
import org.springframework.integration.store.SimpleMessageStore;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.integration.channel.QueueChannel;
import org.springframework.integration.channel.PublishSubscribeChannel;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.channel.PriorityChannel;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.controlbus.ExpressionControlBusFactoryBean;
import org.springframework.integration.support.IntegrationMessageHeaderAccessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.Message;
import java.util.List;
import java.util.Comparator;
```

---

## 7. Complete Order Processing Pipeline

```java
@Configuration
@EnableIntegration
public class OrderProcessingPipeline {
    
    @Bean
    public IntegrationFlow orderPipeline(
            OrderValidator validator,
            OrderEnricher enricher,
            OrderProcessor processor,
            NotificationService notifier) {
        
        return IntegrationFlow.from("orderIngress")
        
            // Step 1: Filter invalid orders
            .filter(Order.class, validator::isValid,
                    f -> f.discardChannel("invalidOrdersChannel"))
            
            // Step 2: Enrich with customer data
            .<Order>transform(enricher::enrich)
            
            // Step 3: Route by priority
            .<Order, String>route(
                o -> o.isPriority() ? "priority" : "normal",
                m -> m
                    .channelMapping("priority", "priorityProcessor")
                    .channelMapping("normal", "normalProcessor")
            )
            .get();
    }
    
    // Priority order flow
    @Bean
    public IntegrationFlow priorityOrderFlow(OrderProcessor processor) {
        return IntegrationFlow.from("priorityProcessor")
            .handle(Order.class, (o, h) -> {
                System.out.println("[PRIORITY] Processing: " + o.id());
                return processor.processPriority(o);
            })
            .channel("processedOrders")
            .get();
    }
    
    // Normal order flow with batching
    @Bean
    public IntegrationFlow normalOrderFlow(OrderProcessor processor) {
        return IntegrationFlow.from("normalProcessor")
            .aggregate(a -> a
                .correlationExpression("'normal-batch'")
                .releaseStrategy(g -> g.size() == 10 || 
                    System.currentTimeMillis() - (long) g.getOne()
                        .getHeaders().getTimestamp() > 5000)  // Release if 10 items or 5s passed
            )
            .<List<Order>>transform(orders -> {
                System.out.println("Processing batch of " + orders.size() + " orders");
                return orders.stream().map(processor::process).toList();
            })
            .split()  // Split processed list back to individual results
            .channel("processedOrders")
            .get();
    }
    
    // Processed orders channel
    @Bean
    public IntegrationFlow processedOrderHandler(NotificationService notifier) {
        return IntegrationFlow.from("processedOrders")
            .wireTap("auditChannel")  // Audit without blocking
            .handle(ProcessedOrder.class, (result, h) -> {
                notifier.notifyCustomer(result);
                return null;
            })
            .get();
    }
    
    // Error handling
    @Bean
    public IntegrationFlow errorHandler() {
        return IntegrationFlow.from("errorChannel")
            .<MessagingException>handle((ex, h) -> {
                System.err.println("Integration error: " + ex.getMessage());
                return null;
            })
            .get();
    }
}

@Component
class OrderValidator { boolean isValid(Order o) { return o.total() > 0; } }

@Component
class OrderEnricher { Order enrich(Order o) { return o; /* add customer data */ } }

@Component
class OrderProcessor { 
    ProcessedOrder processPriority(Order o) { return new ProcessedOrder(o.id(), "PRIORITY_DONE"); }
    ProcessedOrder process(Order o) { return new ProcessedOrder(o.id(), "DONE"); }
}

@Component
class NotificationService { void notifyCustomer(ProcessedOrder r) { System.out.println("Notified: " + r.orderId()); } }

record ProcessedOrder(String orderId, String status) {}

import org.springframework.messaging.MessagingException;
```

---

## สรุป Part 101

| Pattern | Type | Spring Integration |
|---------|------|-------------------|
| Point-to-Point | Channel | `DirectChannel` |
| Publish-Subscribe | Channel | `PublishSubscribeChannel` |
| Content-Based Router | Router | `.route()` |
| Filter | Router | `.filter()` |
| Splitter | Router | `.split()` |
| Aggregator | Router | `.aggregate()` |
| Transformer | Transform | `.<In, Out>transform()` |
| Wire Tap | Management | `.wireTap()` |

---

**Part 102:** Apache Camel Integration Framework
