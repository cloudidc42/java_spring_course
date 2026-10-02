# Part 058: Distributed Tracing and Logging

## Overview

In a microservices architecture, a single user request may span dozens of services. When something goes wrong — or is slow — you need to trace that request across all services and correlate the logs. This is **distributed tracing**. Combined with structured logging and a log aggregation pipeline, you get full **observability**: the ability to understand what your system is doing at any moment.

Spring Cloud Sleuth has been superseded by **Micrometer Tracing** (as of Spring Boot 3.x). Micrometer provides a vendor-neutral tracing API that works with multiple backends (Jaeger, Zipkin, OpenTelemetry Collector).

By the end of this part you will be able to:
- Understand distributed tracing concepts
- Configure Micrometer Tracing with Jaeger
- Add trace/span IDs to all log lines automatically
- Create custom spans for business operations
- Propagate baggage across service boundaries
- Set up structured JSON logging with Logback
- Correlate logs with traces in ELK Stack
- Integrate with OpenTelemetry

---

## Table of Contents

1. [Distributed Tracing Concepts](#1-distributed-tracing-concepts)
2. [Project Setup — Micrometer Tracing](#2-project-setup--micrometer-tracing)
3. [Auto-configured Trace IDs in Logs](#3-auto-configured-trace-ids-in-logs)
4. [Custom Spans with Tracer](#4-custom-spans-with-tracer)
5. [Baggage Propagation](#5-baggage-propagation)
6. [MDC and Structured Logging](#6-mdc-and-structured-logging)
7. [Logback JSON Encoder](#7-logback-json-encoder)
8. [OpenTelemetry Integration](#8-opentelemetry-integration)
9. [Jaeger as Tracing Backend](#9-jaeger-as-tracing-backend)
10. [Log Aggregation with ELK Stack](#10-log-aggregation-with-elk-stack)
11. [Real Example: Complete Observability Setup](#11-real-example-complete-observability-setup)
12. [Summary](#12-summary)

---

## 1. Distributed Tracing Concepts

### Trace, Span, Baggage

```
User Request → Service A → Service B → Database
                           ↘ Service C → Cache

Trace ID: abc123  (one ID for the entire request chain)
│
├── Span A: "service-a.handle-request"   (traceId=abc123, spanId=001, parent=null)
│   ├── Span B: "service-b.call"          (traceId=abc123, spanId=002, parent=001)
│   │   └── Span DB: "db.query"          (traceId=abc123, spanId=004, parent=002)
│   └── Span C: "service-c.call"         (traceId=abc123, spanId=003, parent=001)
│       └── Span Cache: "cache.get"      (traceId=abc123, spanId=005, parent=003)
```

| Term      | Definition                                                                   |
|-----------|------------------------------------------------------------------------------|
| Trace     | Complete end-to-end record of a request across all services                  |
| Span      | A single unit of work within a trace (one service, one operation)            |
| Parent    | The span that caused this span to be created                                 |
| TraceId   | Shared across all spans in a request — the "correlation ID"                  |
| SpanId    | Unique ID per span                                                           |
| Baggage   | Key-value pairs attached to a trace and propagated to all downstream services|
| Sampling  | Deciding which requests to trace (100% in dev, 1-10% in prod)               |

### W3C Trace Context (standard HTTP propagation)

Distributed tracing propagates context in HTTP headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ trace-id (16 bytes)                  span-id  ^^ flags
tracestate:  vendor-specific state
```

---

## 2. Project Setup — Micrometer Tracing

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Web / WebFlux -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Micrometer Tracing (bridge) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
        <!-- Or for OpenTelemetry:
        <artifactId>micrometer-tracing-bridge-otel</artifactId> -->
    </dependency>

    <!-- Zipkin reporter (also used for Jaeger via Zipkin compatible endpoint) -->
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>

    <!-- OpenTelemetry exporter (alternative to Zipkin) -->
    <!--
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
    -->

    <!-- Spring Actuator (exposes /actuator/metrics, /actuator/health) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Feign for inter-service calls (auto-instruments with tracing) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!-- Logback JSON encoder for structured logging -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### application.yml

```yaml
spring:
  application:
    name: order-service    # Appears in traces as the service name

management:
  tracing:
    sampling:
      probability: 1.0     # 100% in dev; use 0.1 (10%) in production
  zipkin:
    tracing:
      endpoint: http://jaeger:9411/api/v2/spans  # Jaeger Zipkin-compatible endpoint

logging:
  pattern:
    # Add trace/span to pattern (for plain text logging)
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
  level:
    root: INFO
    com.example: DEBUG
```

---

## 3. Auto-configured Trace IDs in Logs

Once Micrometer Tracing is configured, **every log line in an HTTP request context** automatically includes `traceId` and `spanId` via MDC.

### What You See in Logs

```
# Plain text (without JSON encoder):
INFO  [order-service,4bf92f3577b34da6,a3ce929d0e0e4736] c.e.OrderService - Processing order 42

# With JSON encoder:
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Processing order 42",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "a3ce929d0e0e4736",
  "service": "order-service"
}
```

### Propagation in RestTemplate

```java
package com.example.order.config;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.observation.ClientRequestObservationConvention;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    // Tracing is auto-applied to RestTemplate when Micrometer Tracing is on the classpath
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .build();
        // The builder automatically adds ClientHttpRequestInterceptor that
        // injects traceparent / b3 headers into outgoing requests
    }
}
```

### Propagation in WebClient

```java
package com.example.order.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {

    // Tracing is auto-applied to WebClient when Micrometer Tracing is present
    @Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
        // ExchangeFilterFunction for trace propagation is auto-configured
    }
}
```

---

## 4. Custom Spans with Tracer

### Creating Spans Programmatically

```java
package com.example.order.service;

import io.micrometer.tracing.Span;
import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.annotation.NewSpan;
import io.micrometer.tracing.annotation.SpanTag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderProcessingService {

    private final Tracer tracer;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;

    // --- Annotation-based span ---

    @NewSpan("order.process")     // Creates a new child span named "order.process"
    public Order processOrder(
            @SpanTag("order.id") String orderId,       // adds tag to the span
            @SpanTag("customer.id") String customerId) {

        log.info("Processing order {} for customer {}", orderId, customerId);
        // traceId and spanId are automatically in MDC here

        paymentService.charge(orderId);
        inventoryService.reserve(orderId);

        return new Order(orderId, "PROCESSED");
    }

    // --- Programmatic span ---

    public PaymentResult processPayment(String orderId, double amount) {
        // Create a child span manually
        Span span = tracer.nextSpan()
            .name("payment.process")
            .tag("order.id", orderId)
            .tag("payment.amount", String.valueOf(amount))
            .start();

        // Make this span the current span in this thread
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            log.info("Starting payment for order {}, amount={}", orderId, amount);

            // All log statements here will carry this span's ID
            PaymentResult result = callPaymentGateway(orderId, amount);

            span.tag("payment.status", result.getStatus());
            log.info("Payment completed: {}", result.getStatus());

            return result;

        } catch (Exception e) {
            span.tag("error", e.getMessage());
            span.error(e);
            log.error("Payment failed for order {}: {}", orderId, e.getMessage());
            throw e;
        } finally {
            span.end();  // ALWAYS end the span
        }
    }

    // --- Span with events ---

    public void fulfillOrder(String orderId) {
        Span span = tracer.nextSpan().name("order.fulfill").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.event("picking-started");
            log.info("Picking items for order {}", orderId);
            pickItems(orderId);

            span.event("packing-started");
            log.info("Packing order {}", orderId);
            packOrder(orderId);

            span.event("shipping-started");
            log.info("Shipping order {}", orderId);
            shipOrder(orderId);

        } finally {
            span.end();
        }
    }

    // --- Async span continuation ---

    public void processAsync(String orderId) {
        Span span = tracer.nextSpan().name("order.async").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            // Capture the current span for use in the async thread
            Span currentSpan = tracer.currentSpan();

            java.util.concurrent.CompletableFuture.runAsync(() -> {
                // Re-attach span in the new thread
                try (Tracer.SpanInScope asyncScope = tracer.withSpan(currentSpan)) {
                    log.info("Async processing order {} — trace still attached", orderId);
                    doAsyncWork(orderId);
                }
            });
        } finally {
            span.end();
        }
    }

    private PaymentResult callPaymentGateway(String orderId, double amount) {
        return new PaymentResult("SUCCESS", orderId);
    }

    private void pickItems(String orderId) {}
    private void packOrder(String orderId) {}
    private void shipOrder(String orderId) {}
    private void doAsyncWork(String orderId) {}
}
```

### @NewSpan with AOP

```java
package com.example.order.config;

import io.micrometer.tracing.annotation.EnabledAspect;
import io.micrometer.tracing.annotation.NewSpanAspect;
import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.handler.DefaultTracingObservationHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class TracingAopConfig {

    // Enable @NewSpan and @ContinueSpan annotations via AOP
    @Bean
    public NewSpanAspect newSpanAspect(Tracer tracer) {
        return new NewSpanAspect(tracer);
    }
}
```

---

## 5. Baggage Propagation

Baggage is key-value data attached to a trace and automatically propagated to all downstream services in HTTP headers.

### Configuration

```yaml
# application.yml
management:
  tracing:
    baggage:
      enabled: true
      correlation:
        enabled: true
        fields:
          - tenant-id      # added to MDC automatically
          - user-id
      remote-fields:
        - tenant-id        # propagated in HTTP headers
        - user-id
        - correlation-id
```

### Using Baggage

```java
package com.example.order.service;

import io.micrometer.tracing.BaggageField;
import io.micrometer.tracing.Tracer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class BaggageService {

    private final Tracer tracer;

    // Set baggage (in a gateway or auth filter)
    public void attachUserBaggage(String userId, String tenantId) {
        BaggageField userIdField = BaggageField.create("user-id");
        BaggageField tenantIdField = BaggageField.create("tenant-id");

        userIdField.updateValue(userId);
        tenantIdField.updateValue(tenantId);

        // Now user-id and tenant-id are:
        // 1. Available in all log statements via MDC (if configured above)
        // 2. Propagated to downstream services via HTTP headers
        log.info("Baggage set for user={}, tenant={}", userId, tenantId);
    }

    // Read baggage (in downstream service)
    public String getUserId() {
        BaggageField userIdField = BaggageField.create("user-id");
        String value = userIdField.getValue();
        log.debug("Read user-id from baggage: {}", value);
        return value;
    }

    public String getTenantId() {
        return BaggageField.create("tenant-id").getValue();
    }

    // Business operation that uses baggage
    public Order placeOrder(OrderRequest request) {
        String tenantId = getTenantId();
        String userId = getUserId();

        log.info("Placing order for tenant={}, user={}", tenantId, userId);
        // tenantId and userId automatically appear in all logs in this trace
        // AND are sent downstream when calling other services

        return processOrderForTenant(request, tenantId);
    }

    private Order processOrderForTenant(OrderRequest req, String tenantId) {
        return new Order();
    }
}
```

### Baggage Filter (extract from JWT/header in gateway)

```java
package com.example.gateway.filter;

import io.micrometer.tracing.BaggageField;
import io.micrometer.tracing.Tracer;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Slf4j
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
@RequiredArgsConstructor
public class BaggageInjectionFilter implements Filter {

    private final Tracer tracer;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;

        // Extract from request header (e.g., set by API gateway after JWT validation)
        String userId = request.getHeader("X-User-Id");
        String tenantId = request.getHeader("X-Tenant-Id");
        String correlationId = request.getHeader("X-Correlation-Id");

        if (userId != null) {
            BaggageField.create("user-id").updateValue(userId);
        }
        if (tenantId != null) {
            BaggageField.create("tenant-id").updateValue(tenantId);
        }
        if (correlationId != null) {
            BaggageField.create("correlation-id").updateValue(correlationId);
        }

        log.debug("Baggage injected: userId={}, tenantId={}", userId, tenantId);
        chain.doFilter(req, res);
    }
}
```

---

## 6. MDC and Structured Logging

### MDC Manual Management

```java
package com.example.order.service;

import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
public class MdcService {

    // Add extra context to MDC for business-level correlation
    public void processOrderWithContext(String orderId, String userId) {
        // MDC keys added to every log statement while this block runs
        try {
            MDC.put("orderId", orderId);
            MDC.put("userId", userId);
            MDC.put("requestId", UUID.randomUUID().toString());

            log.info("Starting order processing");
            validateOrder(orderId);
            log.info("Order validated");
            processPayment(orderId);
            log.info("Payment processed");
            // All these log statements now include orderId, userId, requestId

        } finally {
            // Always clean up MDC — thread pool reuse would leak context otherwise
            MDC.remove("orderId");
            MDC.remove("userId");
            MDC.remove("requestId");
        }
    }

    // Using try-with-resources for safe MDC management
    public void processWithAutoClose(String orderId) {
        try (var ignored = MdcCloseable.of("orderId", orderId)) {
            log.info("Processing — orderId is in MDC");
            // on close, MDC.remove("orderId") is called automatically
        }
    }

    // Set entire MDC context (for async/new thread)
    public void preserveContextInAsync(String orderId) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();

        java.util.concurrent.CompletableFuture.runAsync(() -> {
            if (contextMap != null) {
                MDC.setContextMap(contextMap);  // restore context in new thread
            }
            try {
                log.info("Async processing order {}", orderId);
            } finally {
                MDC.clear();
            }
        });
    }

    private void validateOrder(String id) {}
    private void processPayment(String id) {}

    // Utility class for AutoCloseable MDC
    public record MdcCloseable(String key) implements AutoCloseable {
        public static MdcCloseable of(String key, String value) {
            MDC.put(key, value);
            return new MdcCloseable(key);
        }

        @Override
        public void close() {
            MDC.remove(key);
        }
    }
}
```

---

## 7. Logback JSON Encoder

### logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- Include Spring Boot defaults -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="APP_NAME"
                    source="spring.application.name" defaultValue="app"/>
    <springProperty scope="context" name="APP_ENV"
                    source="spring.profiles.active" defaultValue="default"/>

    <!-- ===== Console Appender (JSON for production, human-readable for dev) ===== -->
    <springProfile name="!dev">
        <!-- Production: JSON output for log aggregation (Logstash, Filebeat) -->
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <!-- Standard fields -->
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>spanId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
                <includeMdcKeyName>tenantId</includeMdcKeyName>
                <includeMdcKeyName>orderId</includeMdcKeyName>
                <includeCallerData>false</includeCallerData>

                <!-- Static fields added to every log entry -->
                <customFields>{"service":"${APP_NAME}","env":"${APP_ENV}"}</customFields>

                <!-- Field name mappings -->
                <fieldNames>
                    <timestamp>@timestamp</timestamp>
                    <message>message</message>
                    <logger>logger</logger>
                    <thread>thread</thread>
                    <level>level</level>
                    <levelValue>[ignore]</levelValue>
                </fieldNames>

                <!-- Throw stack traces as a JSON array -->
                <throwableConverter
                    class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                    <maxDepthPerCause>10</maxDepthPerCause>
                    <shortenedClassNameLength>20</shortenedClassNameLength>
                    <rootCauseFirst>true</rootCauseFirst>
                </throwableConverter>
            </encoder>
        </appender>
    </springProfile>

    <springProfile name="dev">
        <!-- Development: human-readable with color -->
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>
                    %clr(%d{HH:mm:ss.SSS}){faint} %clr(%5p){highlight}
                    %clr([${APP_NAME},%X{traceId:-},%X{spanId:-}]){yellow}
                    %clr(%-40.40logger{39}){cyan} : %m%n%xEx
                </pattern>
            </encoder>
        </appender>
    </springProfile>

    <!-- ===== File Appender (JSON, always) ===== -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/${APP_NAME}-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"${APP_NAME}","env":"${APP_ENV}"}</customFields>
        </encoder>
    </appender>

    <!-- ===== Async Appender (non-blocking) ===== -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <neverBlock>false</neverBlock>
        <appender-ref ref="FILE"/>
    </appender>

    <!-- ===== Logger configurations ===== -->
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
    <logger name="org.hibernate.type.descriptor.sql" level="TRACE"/>

    <!-- Reduce noise from tracing internals -->
    <logger name="io.micrometer" level="WARN"/>
    <logger name="zipkin2.reporter" level="WARN"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </root>

</configuration>
```

### Sample JSON Log Output

```json
{
  "@timestamp": "2024-01-15T10:30:45.123+00:00",
  "level": "INFO",
  "message": "Processing order ORD-001 for customer C-100",
  "logger": "c.e.order.service.OrderService",
  "thread": "http-nio-8080-exec-1",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "a3ce929d0e0e4736",
  "userId": "U-100",
  "tenantId": "ACME",
  "orderId": "ORD-001",
  "service": "order-service",
  "env": "production"
}
```

---

## 8. OpenTelemetry Integration

### OTel with Micrometer Bridge

```xml
<!-- pom.xml — OTel variant -->
<dependencies>
    <!-- Use OTel bridge instead of Brave -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>

    <!-- Export to OTLP collector -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>

    <!-- Resource detectors for auto-populating service.name, etc. -->
    <dependency>
        <groupId>io.opentelemetry.instrumentation</groupId>
        <artifactId>opentelemetry-spring-boot-starter</artifactId>
        <version>2.0.0</version>
    </dependency>
</dependencies>
```

### OTel Configuration

```yaml
# application.yml — OTel
spring:
  application:
    name: order-service

management:
  tracing:
    sampling:
      probability: 1.0

otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4318   # HTTP endpoint
      # endpoint: grpc://otel-collector:4317  # gRPC endpoint
  resource:
    attributes:
      service.name: ${spring.application.name}
      service.version: ${app.version:1.0.0}
      deployment.environment: ${spring.profiles.active:default}
```

### Programmatic OTel Span

```java
package com.example.order.service;

import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class OtelTracingService {

    private final Tracer tracer = GlobalOpenTelemetry.getTracer(
        "com.example.order", "1.0.0"
    );

    public Order processOrder(String orderId) {
        Span span = tracer.spanBuilder("order.process")
            .setSpanKind(SpanKind.INTERNAL)
            .setAttribute("order.id", orderId)
            .setAttribute("order.source", "web")
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            log.info("Processing order {}", orderId);

            Order order = doProcess(orderId);

            span.setAttribute("order.status", order.getStatus());
            span.setStatus(StatusCode.OK);
            return order;

        } catch (Exception e) {
            span.recordException(e, Attributes.of(
                AttributeKey.booleanKey("exception.escaped"), true
            ));
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }

    private Order doProcess(String orderId) {
        return new Order(orderId, "PROCESSED");
    }
}
```

---

## 9. Jaeger as Tracing Backend

### Docker Compose with Jaeger

```yaml
# docker-compose.yml
version: '3.8'
services:

  jaeger:
    image: jaegertracing/all-in-one:1.52
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411    # Accept Zipkin-format spans
      - COLLECTOR_OTLP_ENABLED=true          # Accept OTLP spans
    ports:
      - "6831:6831/udp"  # Jaeger Thrift compact
      - "6832:6832/udp"  # Jaeger Thrift binary
      - "9411:9411"      # Zipkin-compatible endpoint
      - "4317:4317"      # OTLP gRPC
      - "4318:4318"      # OTLP HTTP
      - "16686:16686"    # Jaeger UI

  order-service:
    build: ./order-service
    environment:
      - SPRING_APPLICATION_NAME=order-service
      - MANAGEMENT_ZIPKIN_TRACING_ENDPOINT=http://jaeger:9411/api/v2/spans
      - MANAGEMENT_TRACING_SAMPLING_PROBABILITY=1.0
    ports:
      - "8080:8080"
    depends_on:
      - jaeger

  inventory-service:
    build: ./inventory-service
    environment:
      - SPRING_APPLICATION_NAME=inventory-service
      - MANAGEMENT_ZIPKIN_TRACING_ENDPOINT=http://jaeger:9411/api/v2/spans
      - MANAGEMENT_TRACING_SAMPLING_PROBABILITY=1.0
    ports:
      - "8081:8081"
    depends_on:
      - jaeger
```

### Verify Tracing Across Services

```java
package com.example.order.controller;

import com.example.order.service.OrderService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

@Slf4j
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;
    private final RestTemplate restTemplate;

    @PostMapping
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        log.info("Received order request for customer {}", request.getCustomerId());

        // Trace propagation happens automatically via RestTemplate
        // The call to inventory-service will carry traceparent header
        InventoryCheckResult inventory = restTemplate.postForObject(
            "http://inventory-service/api/inventory/check",
            request.getItems(),
            InventoryCheckResult.class
        );

        if (!inventory.isAvailable()) {
            throw new InsufficientInventoryException();
        }

        Order order = orderService.create(request);
        log.info("Order {} created successfully", order.getId());
        return order;
    }
}
```

---

## 10. Log Aggregation with ELK Stack

### Complete ELK Docker Compose

```yaml
# docker-compose-elk.yml
version: '3.8'
services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"   # Beats input
      - "5000:5000"   # TCP input
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./logs:/logs:ro    # Mount application logs directory
    depends_on:
      - logstash

volumes:
  es_data:
```

### Logstash Pipeline

```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Parse JSON log entries
  json {
    source => "message"
    skip_on_invalid_json => true
  }

  # Rename @timestamp from log (Logstash has its own)
  if [timestamp] {
    date {
      match => ["[timestamp]", "ISO8601"]
      target => "@timestamp"
    }
    mutate { remove_field => ["timestamp"] }
  }

  # Enrich with geo data from IP (optional)
  # geoip { source => "client_ip" }

  # Parse stack traces
  if [stack_trace] {
    mutate {
      gsub => ["stack_trace", "\\n", "\n"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    # Index per service and day: order-service-2024.01.15
    index => "%{[service]:app}-%{+YYYY.MM.dd}"
    manage_template => false
  }
  # Debug output (remove in production)
  # stdout { codec => rubydebug }
}
```

### Filebeat Configuration

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /logs/*.log
    json.keys_under_root: true
    json.add_error_key: true
    json.message_key: message
    multiline.pattern: '^{'           # JSON always starts with {
    multiline.negate: true
    multiline.match: after

output.logstash:
  hosts: ["logstash:5044"]

# Alternative: direct to Elasticsearch
# output.elasticsearch:
#   hosts: ["elasticsearch:9200"]
#   indices:
#     - index: "%{[service]:app}-%{+yyyy.MM.dd}"
```

### Kibana Index Pattern and Queries

```
# Kibana KQL queries for tracing-correlated log search

# Find all logs for a specific trace
traceId: "4bf92f3577b34da6a3ce929d0e0e4736"

# Find errors with their trace IDs
level: "ERROR" AND service: "order-service"

# Find slow operations
message: "Processing order" AND service: *

# Find all activity for a user
userId: "U-100"

# Multi-service trace view
traceId: "4bf92f3577b34da6a3ce929d0e0e4736" | sort by @timestamp
```

---

## 11. Real Example: Complete Observability Setup

### Multi-Service Tracing Demo

#### Service A: Order Service

```java
package com.example.order;

import io.micrometer.tracing.BaggageField;
import io.micrometer.tracing.Span;
import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.annotation.NewSpan;
import io.micrometer.tracing.annotation.SpanTag;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.client.RestTemplate;

import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final Tracer tracer;
    private final RestTemplate restTemplate;

    @PostMapping
    @NewSpan("order.create.http")
    public Map<String, Object> createOrder(
            @RequestBody Map<String, Object> body,
            @RequestHeader(value = "X-User-Id", required = false) String userId) {

        // Set baggage — will propagate to downstream services
        if (userId != null) {
            BaggageField.create("user-id").updateValue(userId);
        }

        String orderId = "ORD-" + System.currentTimeMillis();
        log.info("Creating order {} for user {}", orderId, userId);

        // Call inventory service (trace propagated automatically)
        Map<String, Object> inventoryResult = callInventoryService(orderId);
        log.info("Inventory check result: {}", inventoryResult.get("status"));

        // Call payment service
        Map<String, Object> paymentResult = callPaymentService(orderId);
        log.info("Payment result: {}", paymentResult.get("status"));

        return Map.of(
            "orderId", orderId,
            "traceId", getCurrentTraceId(),
            "status", "CREATED",
            "inventory", inventoryResult.get("status"),
            "payment", paymentResult.get("status")
        );
    }

    @NewSpan("inventory.check")
    private Map<String, Object> callInventoryService(
            @SpanTag("order.id") String orderId) {
        log.info("Calling inventory service for order {}", orderId);
        return restTemplate.getForObject(
            "http://localhost:8081/api/inventory/check?orderId=" + orderId,
            Map.class
        );
    }

    @NewSpan("payment.process")
    private Map<String, Object> callPaymentService(
            @SpanTag("order.id") String orderId) {
        log.info("Calling payment service for order {}", orderId);
        return restTemplate.getForObject(
            "http://localhost:8082/api/payments/process?orderId=" + orderId,
            Map.class
        );
    }

    private String getCurrentTraceId() {
        Span currentSpan = tracer.currentSpan();
        return currentSpan != null
            ? currentSpan.context().traceId()
            : "no-trace";
    }
}
```

#### Service B: Inventory Service

```java
package com.example.inventory;

import io.micrometer.tracing.BaggageField;
import io.micrometer.tracing.Tracer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/inventory")
@RequiredArgsConstructor
public class InventoryController {

    private final Tracer tracer;

    @GetMapping("/check")
    public Map<String, Object> checkInventory(@RequestParam String orderId) {
        // Baggage automatically received from upstream
        String userId = BaggageField.create("user-id").getValue();

        log.info("Checking inventory for order={}, user={}", orderId, userId);
        // This log line carries the SAME traceId as the order-service log!

        // Simulate inventory check
        boolean inStock = Math.random() > 0.1;

        log.info("Inventory check complete for order {}: inStock={}", orderId, inStock);

        return Map.of(
            "orderId", orderId,
            "status", inStock ? "AVAILABLE" : "OUT_OF_STOCK",
            "traceId", getCurrentTraceId(),
            "instance", "inventory-service"
        );
    }

    private String getCurrentTraceId() {
        var span = tracer.currentSpan();
        return span != null ? span.context().traceId() : "no-trace";
    }
}
```

### Observability Configuration Class

```java
package com.example.order.config;

import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.aop.ObservedAspect;
import io.micrometer.tracing.Tracer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.actuate.autoconfigure.observation.ObservationRegistryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Slf4j
@Configuration
public class ObservabilityConfig {

    @Value("${spring.application.name}")
    private String appName;

    // Enables @Observed annotation on classes and methods
    @Bean
    public ObservedAspect observedAspect(ObservationRegistry observationRegistry) {
        return new ObservedAspect(observationRegistry);
    }

    // Customize observation registry
    @Bean
    public ObservationRegistryCustomizer<ObservationRegistry> observationRegistryCustomizer() {
        return registry -> registry
            .observationConfig()
            .observationFilter(observation -> {
                // Add service name to every observation
                observation.highCardinalityKeyValue("service.name", appName);
                return true;
            });
    }
}
```

### Using @Observed Annotation

```java
package com.example.order.service;

import io.micrometer.observation.annotation.Observed;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
@Observed(name = "order.service", contextualName = "OrderProcessingService")
public class ObservedOrderService {

    // @Observed on the class wraps ALL methods

    public Order createOrder(String customerId, double amount) {
        log.info("Creating order for customer={}, amount={}", customerId, amount);
        return new Order("ORD-001", "CREATED");
    }

    // Override per-method
    @Observed(
        name = "order.cancel",
        contextualName = "cancel-order",
        lowCardinalityKeyValues = {"operation", "cancel"}
    )
    public void cancelOrder(String orderId) {
        log.info("Cancelling order {}", orderId);
    }
}
```

### Integration Test for Tracing

```java
package com.example.order;

import io.micrometer.tracing.test.SampleTestRunner;
import io.micrometer.tracing.test.simple.SpansAssert;
import io.micrometer.tracing.test.simple.SimpleSpan;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.ResponseEntity;

import java.util.List;
import java.util.Map;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class TracingIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void traceIdInResponse() {
        ResponseEntity<Map> response = restTemplate.postForEntity(
            "/api/orders",
            Map.of("customerId", "C-001", "amount", 99.99),
            Map.class
        );

        assert response.getStatusCode().is2xxSuccessful();
        assert response.getBody().containsKey("traceId");
        assert ((String) response.getBody().get("traceId")).length() == 32;
    }

    @Test
    void traceIdInLogs() {
        // Use log capture to verify trace ID appears in log output
        // (requires log capture library like spring-test's OutputCaptureExtension)
    }
}
```

---

## 12. Summary

| Feature                        | API / Configuration                                              |
|--------------------------------|------------------------------------------------------------------|
| Auto-configured tracing        | `spring-boot-starter-actuator` + `micrometer-tracing-bridge-*` |
| Sampling rate                  | `management.tracing.sampling.probability`                       |
| TraceId in logs                | Auto via MDC when Micrometer Tracing is on classpath            |
| Jaeger exporter                | `management.zipkin.tracing.endpoint=http://jaeger:9411/...`     |
| OTLP exporter                  | `otel.exporter.otlp.endpoint`                                   |
| Custom span (annotation)       | `@NewSpan("name")` + `@SpanTag`                                 |
| Custom span (programmatic)     | `Tracer.nextSpan().name().tag().start()` … `span.end()`         |
| Baggage set/get                | `BaggageField.create("key").updateValue("val")` / `.getValue()` |
| Baggage in MDC                 | `management.tracing.baggage.correlation.fields`                 |
| Baggage in HTTP headers        | `management.tracing.baggage.remote-fields`                      |
| JSON logging                   | `logstash-logback-encoder` + `LogstashEncoder` in logback.xml   |
| MDC                            | `MDC.put("key", "val")` … `MDC.remove("key")`                  |
| @Observed (metrics + traces)   | `ObservedAspect` bean + `@Observed` annotation                  |
| Log aggregation                | Filebeat → Logstash → Elasticsearch → Kibana                    |

### Key Takeaways

1. Micrometer Tracing replaces Spring Cloud Sleuth in Spring Boot 3.x — the API is nearly identical but vendor-neutral.
2. TraceId/SpanId are automatically added to MDC — no code change required for logs to include them.
3. Use baggage for business context (userId, tenantId) that should flow end-to-end across all services.
4. `@NewSpan` + `@SpanTag` give you structured span data in Jaeger with no boilerplate.
5. JSON logs + ELK + Jaeger = full observability: filter logs by traceId to see every log line from a complete request.
6. In production, set `probability: 0.1` (10%) and implement head-based or tail-based sampling to reduce overhead.

---

## Next Part Preview

**Part 059: Spring Batch** — Batch processing with chunk-oriented processing, job scheduling, restartability, skip/retry policies, partitioned steps, and production-grade job monitoring.
