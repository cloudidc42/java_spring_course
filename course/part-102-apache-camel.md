# Part 102: Apache Camel Integration

## เนื้อหาในส่วนนี้
- Apache Camel คืออะไรและทำไมต้องใช้
- Camel with Spring Boot
- Route DSL (Java, XML, YAML)
- Components (File, HTTP, JMS, Kafka, SQL)
- Enterprise Integration Patterns in Camel
- Error Handling and Dead Letter Channel
- Type Converters
- Data Formats (JSON, XML, CSV)
- Testing Camel Routes
- Camel vs Spring Integration

---

## 1. Apache Camel Fundamentals

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-spring-boot-starter</artifactId>
    <version>4.3.0</version>
</dependency>
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-file-starter</artifactId>
    <version>4.3.0</version>
</dependency>
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-http-starter</artifactId>
    <version>4.3.0</version>
</dependency>
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-jackson-starter</artifactId>
    <version>4.3.0</version>
</dependency>
<dependency>
    <groupId>org.apache.camel.springboot</groupId>
    <artifactId>camel-kafka-starter</artifactId>
    <version>4.3.0</version>
</dependency>
```

```yaml
# application.yml
camel:
  springboot:
    main-run-controller: true    # Keep Camel running
  context:
    name: my-camel-context
    stream-caching: true         # Cache streams for re-reading
```

---

## 2. Basic Routes

```java
import org.apache.camel.*;
import org.apache.camel.builder.RouteBuilder;
import org.apache.camel.model.dataformat.*;
import org.springframework.stereotype.Component;

// Simple route: from file → log → to another location
@Component
public class FileProcessingRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        // Route 1: Process CSV files
        from("file:data/input?noop=false&include=.*\\.csv&delay=5000")
            .routeId("csv-processor")
            .log("Processing file: ${header.CamelFileName}")
            .unmarshal().csv()                  // Parse CSV
            .split(body())                      // Split rows
            .log("Row: ${body}")
            .to("direct:processRow")            // Send to sub-route
            .end()
            .to("file:data/processed");         // Move to processed
        
        // Sub-route: process individual row
        from("direct:processRow")
            .routeId("row-processor")
            .filter(body().isNotNull())
            .process(exchange -> {
                List<String> row = exchange.getIn().getBody(List.class);
                String processed = String.join("|", row);
                exchange.getIn().setBody(processed);
            })
            .to("log:processed-row?level=DEBUG");
    }
}

// HTTP route: poll REST API and transform
@Component
public class HttpPollingRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        // Poll weather API every 60s
        from("timer:weather?period=60000")
            .routeId("weather-poller")
            .setHeader("Accept", constant("application/json"))
            .to("https://api.weather.com/v1/current?q=Bangkok&apiKey={{weather.api.key}}")
            .unmarshal().json()                 // Deserialize JSON
            .process(exchange -> {
                Map weather = exchange.getIn().getBody(Map.class);
                System.out.println("Temperature: " + weather.get("temp") + "°C");
            })
            .to("direct:storeWeather");
    }
}

import java.util.List;
import java.util.Map;
```

---

## 3. Enterprise Integration Patterns in Camel

```java
@Component
public class EIPRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        
        // Content-Based Router
        from("direct:orders")
            .routeId("order-router")
            .choice()
                .when(header("orderType").isEqualTo("PRIORITY"))
                    .to("direct:priorityOrders")
                .when(simple("${body.total} > 1000"))
                    .to("direct:highValueOrders")
                .otherwise()
                    .to("direct:normalOrders")
            .end();
        
        // Message Filter
        from("direct:allProducts")
            .filter(simple("${body.stock} > 0"))
            .to("direct:inStockProducts");
        
        // Splitter
        from("direct:orderBatch")
            .split(body())                  // Split list
                .parallelProcessing()       // Process in parallel
                .to("direct:singleOrder")
            .end()
            .log("Batch complete");
        
        // Aggregator: collect messages with same correlationId
        from("direct:partialData")
            .aggregate(header("correlationId"), 
                       new org.apache.camel.processor.aggregate.GroupedBodyAggregationStrategy())
            .completionSize(5)              // Complete when 5 messages
            .completionTimeout(10000)       // Or after 10s
            .to("direct:completeData");
        
        // Scatter-Gather: send to multiple, wait for all
        from("direct:priceRequest")
            .multicast()
            .parallelProcessing()
            .to("direct:supplier1", "direct:supplier2", "direct:supplier3")
            .end()
            .log("All prices received");
        
        // Recipient List: dynamic routing
        from("direct:dynamicRoute")
            .recipientList(header("destinations").tokenize(","))
            .ignoreInvalidEndpoints();
        
        // Load Balancer
        from("direct:loadBalanced")
            .loadBalance()
            .roundRobin()
            .to("direct:server1", "direct:server2", "direct:server3")
            .end();
    }
}
```

---

## 4. Kafka Integration

```java
@Component
public class KafkaRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        
        // Consume from Kafka → process → produce to another topic
        from("kafka:orders?brokers=localhost:9092&groupId=order-processor&autoOffsetReset=earliest")
            .routeId("kafka-order-consumer")
            .log("Received from Kafka: ${body}")
            .unmarshal().json(OrderEvent.class)
            .process(exchange -> {
                OrderEvent event = exchange.getIn().getBody(OrderEvent.class);
                // Process the order event
                ProcessedOrder result = processOrder(event);
                exchange.getIn().setBody(result);
            })
            .marshal().json()
            .to("kafka:processed-orders?brokers=localhost:9092");
        
        // Timer → generate order → produce to Kafka
        from("timer:orderGenerator?period=5000")
            .routeId("order-generator")
            .process(exchange -> {
                OrderEvent order = new OrderEvent("ORD-" + System.currentTimeMillis(), 99.99);
                exchange.getIn().setBody(order);
            })
            .marshal().json()
            .to("kafka:orders?brokers=localhost:9092");
    }
    
    private ProcessedOrder processOrder(OrderEvent event) {
        return new ProcessedOrder(event.orderId(), "PROCESSED");
    }
}

record OrderEvent(String orderId, double total) {}
record ProcessedOrder(String orderId, String status) {}
```

---

## 5. Error Handling

```java
@Component
public class ErrorHandlingRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        
        // Global error handler with retry and DLQ
        errorHandler(
            deadLetterChannel("direct:deadLetterQueue")
                .maximumRedeliveries(3)
                .redeliveryDelay(1000)
                .backOffMultiplier(2.0)
                .useExponentialBackOff()
                .retryAttemptedLogLevel(LoggingLevel.WARN)
                .deadLetterHandleNewException(false)
        );
        
        // Route-specific error handling
        from("direct:riskyOperation")
            .onException(RuntimeException.class)
                .maximumRedeliveries(2)
                .redeliveryDelay(500)
                .handled(true)
                .log("Error handled: ${exception.message}")
                .to("direct:compensate")
            .end()
            .to("direct:mainFlow");
        
        // Circuit Breaker
        from("direct:externalApi")
            .circuitBreaker()
                .resilience4jConfiguration()
                    .failureRateThreshold(50)
                    .slowCallRateThreshold(80)
                    .waitDurationInOpenState(10000)
                .end()
                .to("https://external-api.example.com/data")
            .onFallback()
                .setBody(constant("{\"status\":\"fallback\"}"))
            .end();
        
        // Dead letter queue handler
        from("direct:deadLetterQueue")
            .log("Dead letter: ${exception.message} for ${body}")
            .to("log:dead-letters?level=ERROR");
    }
}

import org.apache.camel.LoggingLevel;
```

---

## 6. Data Formats and Type Converters

```java
@Component
public class DataFormatRoute extends RouteBuilder {
    
    @Override
    public void configure() {
        
        // JSON (un)marshal
        from("direct:jsonIn")
            .unmarshal().json(Order.class)    // JSON → Java
            .process(exchange -> {
                Order order = exchange.getIn().getBody(Order.class);
                exchange.getIn().setBody(new ProcessedOrder(order.id(), "DONE"));
            })
            .marshal().json()                 // Java → JSON
            .to("direct:jsonOut");
        
        // XML (un)marshal with JAXB
        from("direct:xmlIn")
            .unmarshal().jacksonXml(OrderXml.class)
            .marshal().json()
            .to("direct:xmlToJsonOut");
        
        // CSV processing
        from("direct:csvIn")
            .unmarshal().csv()
            .split(body())
            .process(exchange -> {
                List<String> row = exchange.getIn().getBody(List.class);
                System.out.println("CSV row: " + row);
            });
        
        // Avro (un)marshal
        // from("direct:avroIn")
        //     .unmarshal().avro(Order.getClassSchema())
        //     .marshal().json()
        //     .to("direct:avroOut");
    }
}

// Custom Type Converter
@Converter
@Component
public class OrderTypeConverter {
    
    @Converter
    public ProcessedOrder toProcessed(Order order) {
        return new ProcessedOrder(order.id(), "CONVERTED");
    }
    
    @Converter
    public byte[] toBytes(Order order) throws Exception {
        return new com.fasterxml.jackson.databind.ObjectMapper()
            .writeValueAsBytes(order);
    }
}

record Order(String id, String product, double total) {}

import org.apache.camel.Converter;
import java.util.List;
```

---

## 7. Testing Camel Routes

```java
import org.apache.camel.test.spring.junit5.*;
import org.apache.camel.*;
import org.apache.camel.component.mock.MockEndpoint;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
@CamelSpringBootTest
class OrderRouteTest {
    
    @Autowired
    CamelContext camelContext;
    
    @Autowired
    ProducerTemplate template;
    
    @EndpointInject("mock:result")
    MockEndpoint mockResult;
    
    @Test
    void testOrderRouting() throws InterruptedException {
        // Set expectations
        mockResult.expectedMessageCount(1);
        mockResult.expectedBodiesReceived(new ProcessedOrder("ORD-001", "DONE"));
        
        // Send test message
        template.sendBody("direct:orders", new Order("ORD-001", "Laptop", 999.0));
        
        // Verify
        mockResult.assertIsSatisfied();
    }
    
    @Test
    void testContentBasedRouter_highValue() throws InterruptedException {
        // Advice: intercept "direct:highValueOrders" and redirect to mock
        camelContext.getRouteDefinition("order-router")
            .adviceWith(camelContext, new org.apache.camel.builder.AdviceWithRouteBuilder() {
                @Override
                public void configure() {
                    interceptSendToEndpoint("direct:highValueOrders")
                        .skipSendToOriginalEndpoint()
                        .to("mock:highValue");
                }
            });
        
        MockEndpoint highValue = camelContext.getEndpoint("mock:highValue", MockEndpoint.class);
        highValue.expectedMessageCount(1);
        
        template.sendBody("direct:orders", new Order("ORD-002", "MacBook", 2499.0));
        
        highValue.assertIsSatisfied();
    }
    
    @Test
    void testWithHeaders() {
        // Send with headers
        var result = template.requestBodyAndHeader(
            "direct:dynamicRoute",
            "test message",
            "destinations", "direct:sink1,direct:sink2",
            String.class
        );
        
        org.assertj.core.api.Assertions.assertThat(result).isNotNull();
    }
}
```

---

## 8. Camel vs Spring Integration

| Feature | Apache Camel | Spring Integration |
|---------|-------------|-------------------|
| DSL | Java, XML, YAML, Groovy | Java DSL only |
| Components | 300+ out of the box | ~60 adapters |
| Learning curve | Moderate | Higher |
| EIP coverage | All 65+ patterns | Core patterns |
| Routing | Rich DSL with sugar | Explicit config |
| Testing | MockEndpoint, AdviceWith | MockMessageChannel |
| Community | Very large | Spring ecosystem |
| Cloud integration | Camel K (Kubernetes) | Spring Cloud |

---

## สรุป Part 102

```
Apache Camel core concepts:
- Route: from().process().to()
- Exchange: message container with In/Out
- Endpoint: URI-addressable component
- Component: factory for Endpoints
- Processor: transforms Exchange
- DataFormat: serialize/deserialize
- TypeConverter: convert between types
```

---

**Part 103:** Jakarta EE and Spring Boot - JTA Transactions, JMS, CDI
