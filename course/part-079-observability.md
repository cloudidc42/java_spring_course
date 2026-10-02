# Part 079: Complete Observability Stack

## Overview

Observability is the ability to understand what your system is doing from the outside, using its outputs. It has three pillars: **Metrics** (what is happening), **Logs** (why it happened), and **Traces** (where time was spent). This part sets up a complete Grafana stack with Prometheus + Loki + Tempo using OpenTelemetry, giving you deep visibility into your Spring Boot applications.

---

## 1. The Three Pillars

| Pillar | Tool | Question Answered |
|--------|------|-------------------|
| Metrics | Prometheus + Grafana | Is the system healthy? Are SLOs being met? |
| Logs | Loki + Grafana | What happened exactly? What errors occurred? |
| Traces | Tempo + Grafana | Where did the request spend time? What caused slowness? |

### Why OpenTelemetry?

OpenTelemetry is vendor-neutral. You instrument once, export anywhere:
- Change from Jaeger to Tempo: change one config line
- Add a new backend: add an exporter
- Mix tools: OTLP → Collector → multiple backends

---

## 2. Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Micrometer + Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- OpenTelemetry for traces -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>

    <!-- Loki logging appender -->
    <dependency>
        <groupId>com.github.loki4j</groupId>
        <artifactId>loki-logback-appender</artifactId>
        <version>1.5.1</version>
    </dependency>

    <!-- Structured logging with Logstash encoder -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>
</dependencies>
```

---

## 3. Application Configuration

```yaml
# application.yml
spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,env
  endpoint:
    health:
      show-details: always
      show-components: always
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
      version: ${APP_VERSION:unknown}
    distribution:
      percentiles-histogram:
        http.server.requests: true
        order.processing.time: true
      percentiles:
        http.server.requests: [0.5, 0.90, 0.95, 0.99]
        order.processing.time: [0.5, 0.90, 0.95, 0.99]
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms, 1s
  tracing:
    sampling:
      probability: 1.0  # 100% in dev; use 0.1 in production

# OpenTelemetry exporter
otel:
  exporter:
    otlp:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318}
      protocol: http/protobuf
  resource:
    attributes:
      service.name: ${spring.application.name}
      service.version: ${APP_VERSION:unknown}
      deployment.environment: ${ENVIRONMENT:local}
```

---

## 4. Structured Logging with Logback + Loki

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="environment" source="ENVIRONMENT" defaultValue="local"/>

    <!-- Console appender with JSON format for local development -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>orderId</includeMdcKeyName>
            <customFields>{"app":"${appName}","env":"${environment}"}</customFields>
        </encoder>
    </appender>

    <!-- Loki appender for centralized log aggregation -->
    <appender name="LOKI" class="com.github.loki4j.logback.Loki4jAppender">
        <http>
            <url>${LOKI_URL:-http://localhost:3100}/loki/api/v1/push</url>
        </http>
        <format>
            <label>
                <pattern>app=${appName},env=${environment},level=%level,host=${HOSTNAME}</pattern>
            </label>
            <message class="com.github.loki4j.logback.JsonLayout">
                <includeLevel>true</includeLevel>
                <includeTimestamp>true</includeTimestamp>
                <includeThreadName>true</includeThreadName>
                <includeLoggerName>true</includeLoggerName>
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>spanId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
                <includeMdcKeyName>orderId</includeMdcKeyName>
            </message>
        </format>
        <batchMaxItems>1000</batchMaxItems>
        <batchTimeoutMs>5000</batchTimeoutMs>
    </appender>

    <!-- Async wrapper for performance -->
    <appender name="ASYNC_LOKI" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="LOKI"/>
        <queueSize>10000</queueSize>
        <neverBlock>true</neverBlock>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="ASYNC_LOKI"/>
    </root>

    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate.SQL" level="DEBUG"/>
</configuration>
```

---

## 5. Metrics with Micrometer

### Custom Metrics

```java
// src/main/java/com/example/metrics/OrderMetrics.java
package com.example.metrics;

import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class OrderMetrics {

    private final Counter ordersCreated;
    private final Counter ordersCompleted;
    private final Counter ordersCancelled;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final DistributionSummary orderValueDistribution;
    private final AtomicInteger pendingOrders;
    private final AtomicLong totalRevenue;

    public OrderMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("order.created")
                .description("Total number of orders created")
                .tag("service", "order-service")
                .register(registry);

        this.ordersCompleted = Counter.builder("order.completed")
                .description("Total number of orders completed")
                .register(registry);

        this.ordersCancelled = Counter.builder("order.cancelled")
                .description("Total number of orders cancelled")
                .register(registry);

        this.ordersFailed = Counter.builder("order.failed")
                .description("Total number of failed orders")
                .register(registry);

        this.orderProcessingTime = Timer.builder("order.processing.time")
                .description("Time to process an order from creation to confirmation")
                .publishPercentiles(0.5, 0.95, 0.99)
                .publishPercentileHistogram()
                .register(registry);

        this.orderValueDistribution = DistributionSummary.builder("order.value")
                .description("Distribution of order values in cents")
                .baseUnit("cents")
                .publishPercentiles(0.5, 0.90, 0.99)
                .publishPercentileHistogram()
                .scale(100)  // Convert dollars to cents
                .register(registry);

        this.pendingOrders = new AtomicInteger(0);
        Gauge.builder("order.pending.count", pendingOrders, AtomicInteger::get)
                .description("Current number of pending orders")
                .register(registry);

        this.totalRevenue = new AtomicLong(0);
        Gauge.builder("order.revenue.total.cents", totalRevenue, AtomicLong::get)
                .description("Total revenue in cents")
                .register(registry);
    }

    public void recordOrderCreated(BigDecimal amount) {
        ordersCreated.increment();
        pendingOrders.incrementAndGet();
        orderValueDistribution.record(amount.doubleValue());
    }

    public void recordOrderCompleted(BigDecimal amount, long processingTimeMs) {
        ordersCompleted.increment();
        pendingOrders.decrementAndGet();
        orderProcessingTime.record(java.time.Duration.ofMillis(processingTimeMs));
        totalRevenue.addAndGet(amount.multiply(BigDecimal.valueOf(100)).longValue());
    }

    public void recordOrderCancelled() {
        ordersCancelled.increment();
        pendingOrders.decrementAndGet();
    }

    public void recordOrderFailed(String reason) {
        ordersFailed.increment(
                Tags.of("reason", reason)
        );
        pendingOrders.decrementAndGet();
    }
}
```

### Timed Annotation Usage

```java
// src/main/java/com/example/service/OrderService.java
package com.example.service;

import com.example.metrics.OrderMetrics;
import io.micrometer.core.annotation.Timed;
import io.micrometer.core.instrument.MeterRegistry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;
    private final OrderMetrics orderMetrics;
    private final MeterRegistry registry;

    public OrderService(OrderRepository orderRepository,
                        OrderMetrics orderMetrics,
                        MeterRegistry registry) {
        this.orderRepository = orderRepository;
        this.orderMetrics = orderMetrics;
        this.registry = registry;
    }

    @Timed(value = "order.service.create",
           description = "Time to create an order",
           percentiles = {0.5, 0.95, 0.99})
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        log.info("Creating order for customer: {}", request.customerId());

        Order order = buildOrder(request);
        Order saved = orderRepository.save(order);

        orderMetrics.recordOrderCreated(saved.getTotalAmount());

        log.info("Order created: orderId={}, amount={}",
                saved.getId(), saved.getTotalAmount());

        return saved;
    }

    @Timed(value = "order.service.confirm")
    @Transactional
    public Order confirmOrder(java.util.UUID orderId) {
        log.info("Confirming order: {}", orderId);

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        long createdEpoch = order.getCreatedAt().toEpochSecond(
                java.time.ZoneOffset.UTC);
        long processingMs = (System.currentTimeMillis() / 1000 - createdEpoch) * 1000;

        order.confirm();
        Order saved = orderRepository.save(order);

        orderMetrics.recordOrderCompleted(saved.getTotalAmount(), processingMs);

        log.info("Order confirmed: orderId={}", orderId);
        return saved;
    }
}
```

---

## 6. Distributed Tracing with OpenTelemetry

### Manual Span Creation

```java
// src/main/java/com/example/tracing/OrderTracingService.java
package com.example.tracing;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.tracing.Tracer;
import io.micrometer.tracing.Span;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.UUID;
import java.util.function.Supplier;

@Service
public class OrderTracingService {

    private static final Logger log = LoggerFactory.getLogger(OrderTracingService.class);

    private final Tracer tracer;
    private final ObservationRegistry observationRegistry;

    public OrderTracingService(Tracer tracer, ObservationRegistry observationRegistry) {
        this.tracer = tracer;
        this.observationRegistry = observationRegistry;
    }

    /**
     * Execute operation within a custom span.
     * Adds contextual information to the trace.
     */
    public <T> T withSpan(String spanName, UUID orderId, Supplier<T> operation) {
        Span span = tracer.nextSpan()
                .name(spanName)
                .tag("order.id", orderId.toString())
                .start();

        try (var ws = tracer.withSpan(span)) {
            // Tags added to span (visible in Tempo)
            span.tag("order.service", "order-service");

            T result = operation.get();

            span.tag("order.result", "success");
            return result;

        } catch (Exception e) {
            span.tag("order.result", "error");
            span.tag("error.message", e.getMessage());
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }

    /**
     * Use Observation API for both metrics and traces in one call.
     */
    public <T> T observe(String name, UUID orderId, Supplier<T> operation) {
        return Observation.createNotStarted(name, observationRegistry)
                .lowCardinalityKeyValue("service", "order-service")
                .highCardinalityKeyValue("order.id", orderId.toString())
                .observe(operation::get);
    }
}
```

### Trace Context Propagation in HTTP Client

```java
// src/main/java/com/example/client/ProductClient.java
package com.example.client;

import io.micrometer.observation.ObservationRegistry;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.client.observation.DefaultClientRequestObservationConvention;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.util.UUID;

@Component
public class ProductClient {

    private final RestClient restClient;

    public ProductClient(
            RestClient.Builder builder,
            ObservationRegistry observationRegistry,
            @Value("${services.product.base-url:http://product-service:8080}") String baseUrl) {

        // RestClient with observation support automatically propagates trace context
        this.restClient = builder
                .baseUrl(baseUrl)
                .observationRegistry(observationRegistry)
                .observationConvention(new DefaultClientRequestObservationConvention())
                .build();
    }

    public ProductDto getProduct(UUID productId) {
        return restClient.get()
                .uri("/api/products/{id}", productId)
                .retrieve()
                .body(ProductDto.class);
    }
}
```

### Trace Context in MDC (Links Logs to Traces)

```java
// src/main/java/com/example/filter/TraceContextFilter.java
package com.example.filter;

import io.micrometer.tracing.Tracer;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Order(1)
public class TraceContextFilter implements Filter {

    private final Tracer tracer;

    public TraceContextFilter(Tracer tracer) {
        this.tracer = tracer;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        // Add trace/span IDs to MDC for log correlation
        var currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            MDC.put("traceId", currentSpan.context().traceId());
            MDC.put("spanId", currentSpan.context().spanId());
        }

        // Add user context if available
        String userId = ((HttpServletRequest) request).getHeader("X-User-Id");
        if (userId != null) {
            MDC.put("userId", userId);
        }

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("traceId");
            MDC.remove("spanId");
            MDC.remove("userId");
            MDC.remove("orderId");
        }
    }
}
```

---

## 7. Custom Actuator Endpoints

```java
// src/main/java/com/example/actuator/OrderStatsEndpoint.java
package com.example.actuator;

import com.example.repository.OrderRepository;
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.Map;

@Component
@Endpoint(id = "order-stats")
public class OrderStatsEndpoint {

    private final OrderRepository orderRepository;

    public OrderStatsEndpoint(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @ReadOperation
    public Map<String, Object> stats() {
        LocalDateTime since = LocalDateTime.now().minusHours(1);

        return Map.of(
                "lastHour", Map.of(
                        "created", orderRepository.countByCreatedAtAfter(since),
                        "confirmed", orderRepository.countByStatusAndCreatedAtAfter("CONFIRMED", since),
                        "cancelled", orderRepository.countByStatusAndCreatedAtAfter("CANCELLED", since)
                ),
                "revenue", Map.of(
                        "lastHour", orderRepository.sumTotalAmountByCreatedAtAfter(since)
                                .orElse(BigDecimal.ZERO),
                        "today", orderRepository.sumTotalAmountByCreatedAtAfter(
                                LocalDateTime.now().withHour(0).withMinute(0))
                                .orElse(BigDecimal.ZERO)
                )
        );
    }
}
```

---

## 8. SLO Monitoring

```java
// src/main/java/com/example/slo/SloConfiguration.java
package com.example.slo;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.MeterBinder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SloConfiguration {

    /**
     * Configure SLO thresholds for HTTP requests.
     * These appear as histogram buckets in Prometheus.
     */
    @Bean
    public MeterBinder httpSloMetrics() {
        return registry -> {
            // Register SLO violation counters
            io.micrometer.core.instrument.Counter.builder("slo.violation.http")
                    .tag("threshold", "200ms")
                    .description("Number of requests exceeding 200ms SLO")
                    .register(registry);

            io.micrometer.core.instrument.Counter.builder("slo.violation.http")
                    .tag("threshold", "1s")
                    .description("Number of requests exceeding 1s SLO")
                    .register(registry);
        };
    }
}
```

### SLO Interceptor

```java
// src/main/java/com/example/slo/SloInterceptor.java
package com.example.slo;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

@Component
public class SloInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(SloInterceptor.class);
    private static final String START_TIME = "requestStartTime";

    private final Counter slo200msViolations;
    private final Counter slo1sViolations;

    public SloInterceptor(MeterRegistry registry) {
        this.slo200msViolations = registry.counter("slo.violation.http",
                "threshold", "200ms");
        this.slo1sViolations = registry.counter("slo.violation.http",
                "threshold", "1s");
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        request.setAttribute(START_TIME, System.currentTimeMillis());
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        Long startTime = (Long) request.getAttribute(START_TIME);
        if (startTime == null) return;

        long elapsedMs = System.currentTimeMillis() - startTime;

        if (elapsedMs > 1000) {
            slo1sViolations.increment();
            log.warn("SLO VIOLATION: Request exceeded 1s. URI={}, elapsed={}ms",
                    request.getRequestURI(), elapsedMs);
        } else if (elapsedMs > 200) {
            slo200msViolations.increment();
            log.debug("SLO warning: Request exceeded 200ms. URI={}, elapsed={}ms",
                    request.getRequestURI(), elapsedMs);
        }
    }
}
```

---

## 9. Complete Docker Compose Stack

```yaml
# docker-compose-observability.yml
version: '3.9'

services:
  # Your Spring Boot application
  order-service:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/orders
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4318
      LOKI_URL: http://loki:3100
      ENVIRONMENT: docker
    depends_on:
      - postgres
      - otel-collector
      - loki

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  # OpenTelemetry Collector - receives OTLP, forwards to backends
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics
    depends_on:
      - tempo
      - loki

  # Prometheus - metrics storage
  prometheus:
    image: prom/prometheus:v2.48.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'
      - '--storage.tsdb.retention.time=7d'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  # Grafana - visualization
  grafana:
    image: grafana/grafana:10.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
      - tempo

  # Loki - log aggregation
  loki:
    image: grafana/loki:2.9.3
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    ports:
      - "3100:3100"

  # Tempo - distributed tracing
  tempo:
    image: grafana/tempo:2.3.1
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
      - tempo-data:/var/tempo
    ports:
      - "3200:3200"
      - "4317:4317"  # OTLP gRPC

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  tempo-data:
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s

  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8889']

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files:
  - "alerts/*.yml"
```

### OpenTelemetry Collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 512
  resource:
    attributes:
      - action: insert
        key: loki.resource.labels
        value: service.name, deployment.environment

exporters:
  # Export traces to Tempo
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  # Export logs to Loki
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

  # Export metrics to Prometheus (via remote write)
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp/tempo, debug]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [loki]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
```

### Loki Configuration

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  chunk_idle_period: 3m
  chunk_block_size: 262144
  chunk_retain_period: 1m

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  max_entries_limit_per_query: 50000

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: filesystem
```

### Tempo Configuration

```yaml
# tempo.yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 48h

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal

querier:
  max_concurrent_queries: 20

query_frontend:
  max_retries: 2
  search:
    max_duration: 0

metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write
        send_exemplars: true
```

---

## 10. Grafana Dashboard Provisioning

```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo

  - name: Loki
    type: loki
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: '"traceId":"(\w+)"'
          name: TraceID
          url: '$${__value.raw}'

  - name: Tempo
    type: tempo
    url: http://tempo:3200
    uid: tempo
    jsonData:
      httpMethod: GET
      serviceMap:
        datasourceUid: prometheus
      nodeGraph:
        enabled: true
      lokiSearch:
        datasourceUid: loki
```

### Dashboard as Code (JSON Model)

```java
// src/main/java/com/example/grafana/GrafanaDashboardExporter.java
package com.example.grafana;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

/**
 * Exports Grafana dashboards programmatically.
 * Useful for version-controlling dashboards as code.
 */
@Component
public class GrafanaDashboardExporter {

    private final RestClient grafanaClient;
    private final ObjectMapper objectMapper;

    public GrafanaDashboardExporter(
            @org.springframework.beans.factory.annotation.Value("${grafana.url:http://localhost:3000}") String grafanaUrl,
            @org.springframework.beans.factory.annotation.Value("${grafana.api-key:}") String apiKey,
            ObjectMapper objectMapper) {

        this.objectMapper = objectMapper;
        this.grafanaClient = RestClient.builder()
                .baseUrl(grafanaUrl)
                .defaultHeader("Authorization", "Bearer " + apiKey)
                .build();
    }

    public String exportDashboard(String dashboardUid) {
        return grafanaClient.get()
                .uri("/api/dashboards/uid/{uid}", dashboardUid)
                .retrieve()
                .body(String.class);
    }

    /**
     * Create a dashboard programmatically.
     * Returns the dashboard URL.
     */
    public String createOrderDashboard() throws Exception {
        ObjectNode dashboard = objectMapper.createObjectNode();
        dashboard.put("title", "Order Service Dashboard");
        dashboard.put("refresh", "10s");
        dashboard.put("schemaVersion", 38);

        ArrayNode panels = dashboard.putArray("panels");

        // Add request rate panel
        ObjectNode requestRate = panels.addObject();
        requestRate.put("title", "Request Rate (req/s)");
        requestRate.put("type", "timeseries");
        requestRate.put("gridPos", objectMapper.readTree("""
            {"x":0,"y":0,"w":12,"h":8}
        """));

        ArrayNode targets = requestRate.putArray("targets");
        ObjectNode target = targets.addObject();
        target.put("expr", "rate(http_server_requests_seconds_count{application='order-service'}[1m])");
        target.put("legendFormat", "{{method}} {{uri}} {{status}}");

        ObjectNode wrapper = objectMapper.createObjectNode();
        wrapper.set("dashboard", dashboard);
        wrapper.put("folderId", 0);
        wrapper.put("overwrite", true);

        return grafanaClient.post()
                .uri("/api/dashboards/db")
                .contentType(org.springframework.http.MediaType.APPLICATION_JSON)
                .body(wrapper.toString())
                .retrieve()
                .body(String.class);
    }
}
```

---

## 11. Alerting Rules

```yaml
# prometheus-alerts/order-service.yml
groups:
  - name: order-service
    interval: 30s
    rules:
      # High error rate alert
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{
            application="order-service",
            status=~"5.."
          }[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate in order-service"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"
          runbook_url: "https://wiki.example.com/runbooks/order-service-errors"

      # High latency alert
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket{
              application="order-service"
            }[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency exceeds 1s"
          description: "P99 latency is {{ $value }}s"

      # Order processing stuck
      - alert: OrdersStuck
        expr: |
          increase(order_created_total[1h]) > 0
          and
          increase(order_completed_total[1h]) == 0
        for: 15m
        labels:
          severity: critical
        annotations:
          summary: "Orders are being created but none completed"
          description: "Possible downstream service failure"

      # Error budget burn rate
      - alert: ErrorBudgetBurnRate
        expr: |
          (
            rate(http_server_requests_seconds_count{
              application="order-service",
              status=~"5.."
            }[1h])
            /
            rate(http_server_requests_seconds_count{
              application="order-service"
            }[1h])
          ) > 0.001 * 14.4
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning too fast"
          description: "At this rate, 30-day error budget will be exhausted in less than 2 days"
```

---

## 12. Health Checks and Probes

```java
// src/main/java/com/example/health/DatabaseHealthIndicator.java
package com.example.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.time.Duration;
import java.time.Instant;

@Component("database")
public class DatabaseHealthIndicator implements HealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    public DatabaseHealthIndicator(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Health health() {
        try {
            Instant start = Instant.now();
            Integer result = jdbcTemplate.queryForObject(
                    "SELECT 1", Integer.class);
            long elapsed = Duration.between(start, Instant.now()).toMillis();

            if (result == null || result != 1) {
                return Health.down()
                        .withDetail("error", "Unexpected result from health check query")
                        .build();
            }

            return Health.up()
                    .withDetail("queryTimeMs", elapsed)
                    .build();

        } catch (Exception e) {
            return Health.down()
                    .withException(e)
                    .build();
        }
    }
}
```

```java
// src/main/java/com/example/health/ExternalServiceHealthIndicator.java
package com.example.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.ReactiveHealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;
import reactor.core.publisher.Mono;

import java.time.Duration;

@Component("productService")
public class ExternalServiceHealthIndicator implements HealthIndicator {

    private final RestClient restClient;

    public ExternalServiceHealthIndicator(
            @org.springframework.beans.factory.annotation.Value(
                    "${services.product.base-url:http://product-service:8080}") String baseUrl) {
        this.restClient = RestClient.builder()
                .baseUrl(baseUrl)
                .build();
    }

    @Override
    public Health health() {
        try {
            restClient.get()
                    .uri("/actuator/health")
                    .retrieve()
                    .toBodilessEntity();

            return Health.up()
                    .withDetail("service", "product-service")
                    .build();
        } catch (Exception e) {
            return Health.down()
                    .withDetail("service", "product-service")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

---

## 13. Observation API (Unified Metrics + Traces)

```java
// src/main/java/com/example/observation/OrderObservationConfig.java
package com.example.observation;

import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.aop.ObservedAspect;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OrderObservationConfig {

    /**
     * Enable @Observed annotation on any Spring bean method.
     * Creates both a metric AND a trace span automatically.
     */
    @Bean
    public ObservedAspect observedAspect(ObservationRegistry registry) {
        return new ObservedAspect(registry);
    }
}
```

```java
// src/main/java/com/example/service/PaymentService.java
package com.example.service;

import io.micrometer.observation.annotation.Observed;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.UUID;

@Service
public class PaymentService {

    /**
     * @Observed creates a metric timer AND a trace span.
     * No manual instrumentation needed.
     */
    @Observed(
            name = "payment.process",
            contextualName = "processing-payment",
            lowCardinalityKeyValues = {"payment.provider", "stripe"}
    )
    public PaymentResult processPayment(UUID orderId, BigDecimal amount) {
        // Implementation
        return new PaymentResult(UUID.randomUUID(), "SUCCESS");
    }

    @Observed(name = "payment.refund")
    public RefundResult processRefund(UUID paymentId) {
        // Implementation
        return new RefundResult(UUID.randomUUID(), "SUCCESS");
    }

    public record PaymentResult(UUID paymentId, String status) {}
    public record RefundResult(UUID refundId, String status) {}
}
```

---

## Summary

| Component | Purpose | URL in Docker |
|-----------|---------|---------------|
| Spring Boot Actuator | Expose metrics endpoint | :8080/actuator/prometheus |
| Prometheus | Scrape and store metrics | :9090 |
| Loki | Collect and query logs | :3100 |
| Tempo | Collect and query traces | :3200 |
| Grafana | Visualize all three | :3000 |
| OTel Collector | Receive OTLP, forward to backends | :4318 |

### Key PromQL Queries for Grafana

```promql
# Request rate
rate(http_server_requests_seconds_count{application="order-service"}[1m])

# P99 latency
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))

# Error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
/ rate(http_server_requests_seconds_count[5m])

# Orders created per minute
rate(order_created_total[1m]) * 60

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}
```

---

## Next Part Preview

**Part 080: Spring Boot Admin and Management** - We'll set up Spring Boot Admin server to manage multiple service instances with a web UI: health dashboards, log level management, JVM monitoring, and Slack/email notifications.
