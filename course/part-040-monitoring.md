# Part 040: Monitoring with Prometheus & Grafana

## Introduction

Observability is the ability to understand what your system is doing from its external outputs. For production Spring Boot microservices, this means three pillars: **Metrics** (what happened numerically), **Logs** (what happened in detail), and **Traces** (how a request flowed through services). This part builds a complete observability stack using industry-standard tools: Micrometer, Prometheus, Grafana, and the ELK stack.

---

## 1. Observability: Metrics, Logs, Traces

```
┌────────────────────────────────────────────────────────────┐
│                   THREE PILLARS OF OBSERVABILITY            │
│                                                            │
│  METRICS          LOGS              TRACES                 │
│  ─────────────    ─────────────     ──────────────────     │
│  "What?"          "Why?"            "Where?"               │
│  Quantitative     Qualitative       Distributed            │
│                                                            │
│  CPU usage        Error message     Request path           │
│  Request rate     Stack trace       Latency per hop        │
│  DB pool size     User action       Service dependencies   │
│                                                            │
│  Prometheus       Elasticsearch     Zipkin / Jaeger        │
│  Grafana          Kibana            Tempo                  │
└────────────────────────────────────────────────────────────┘
```

| Pillar | Tool | Spring Integration |
|--------|------|-------------------|
| Metrics | Prometheus + Grafana | Micrometer |
| Logs | ELK Stack | Logback + Logstash appender |
| Traces | Zipkin / Jaeger | Spring Cloud Sleuth / Micrometer Tracing |

---

## 2. Micrometer Metrics in Spring Boot

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Distributed tracing -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>

    <!-- Logstash JSON encoder for structured logging -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>
</dependencies>
```

### application.yaml

```yaml
spring:
  application:
    name: order-service

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,threaddump,heapdump
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
        descriptions: true
        step: 30s
    tags:
      # Common tags added to ALL metrics
      application: ${spring.application.name}
      environment: ${APP_ENVIRONMENT:development}
      region: ${APP_REGION:us-east-1}
    distribution:
      # Enable percentile histograms for HTTP requests
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.75, 0.95, 0.99, 0.999
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms, 1s, 2s

  # Distributed tracing
  tracing:
    sampling:
      probability: 1.0  # 100% sampling (reduce in production)

# Zipkin (or any OTLP-compatible collector)
management.zipkin.tracing.endpoint: http://zipkin:9411/api/v2/spans
```

---

## 3. Prometheus Scraping Configuration

### Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s        # How often to scrape
  evaluation_interval: 15s    # How often to evaluate rules
  external_labels:
    monitor: 'spring-boot-monitor'

# Alerting rules
rule_files:
  - "alert_rules.yml"

# Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

scrape_configs:
  # Spring Boot services
  - job_name: 'order-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - order-service:8080
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  - job_name: 'user-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - user-service:8080

  - job_name: 'product-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - product-service:8080

  # Kubernetes pod discovery (for dynamic environments)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app

  # Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (system metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Kubernetes Annotation-based Scraping

```yaml
# k8s/deployment.yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/actuator/prometheus"
    prometheus.io/port: "8080"
```

---

## 4. Custom Counters, Gauges, Timers, DistributionSummary

### Metrics Configuration Bean

```java
// src/main/java/com/example/monitoring/metrics/AppMetrics.java
package com.example.monitoring.metrics;

import io.micrometer.core.instrument.*;
import io.micrometer.core.instrument.binder.MeterBinder;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

@Component
public class AppMetrics implements MeterBinder {

    // ─── Counters ─────────────────────────────────────────────────────────
    private Counter ordersCreatedCounter;
    private Counter ordersFailedCounter;
    private Counter paymentSuccessCounter;
    private Counter paymentFailureCounter;

    // ─── Gauges ───────────────────────────────────────────────────────────
    private final AtomicInteger activeOrdersGauge = new AtomicInteger(0);
    private final AtomicInteger pendingPaymentsGauge = new AtomicInteger(0);
    private final AtomicLong inventoryGauge = new AtomicLong(0);

    // ─── Timers ───────────────────────────────────────────────────────────
    private Timer orderProcessingTimer;
    private Timer paymentProcessingTimer;
    private Timer dbQueryTimer;

    // ─── Distribution Summary ─────────────────────────────────────────────
    private DistributionSummary orderAmountSummary;
    private DistributionSummary requestSizeSummary;

    @Override
    public void bindTo(MeterRegistry registry) {
        // ─── Counters ─────────────────────────────────────────────────────
        ordersCreatedCounter = Counter.builder("orders.created.total")
            .description("Total number of orders created")
            .tag("service", "order-service")
            .register(registry);

        ordersFailedCounter = Counter.builder("orders.failed.total")
            .description("Total number of orders that failed")
            .tag("service", "order-service")
            .register(registry);

        paymentSuccessCounter = Counter.builder("payments.total")
            .description("Total payments processed")
            .tag("status", "success")
            .register(registry);

        paymentFailureCounter = Counter.builder("payments.total")
            .description("Total payments processed")
            .tag("status", "failure")
            .register(registry);

        // ─── Gauges ───────────────────────────────────────────────────────
        Gauge.builder("orders.active", activeOrdersGauge, AtomicInteger::get)
            .description("Number of currently active (unprocessed) orders")
            .register(registry);

        Gauge.builder("payments.pending", pendingPaymentsGauge, AtomicInteger::get)
            .description("Number of pending payment confirmations")
            .register(registry);

        Gauge.builder("inventory.total", inventoryGauge, AtomicLong::get)
            .description("Total inventory count across all products")
            .register(registry);

        // ─── Timers ───────────────────────────────────────────────────────
        orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Time taken to process an order end-to-end")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .sla(
                java.time.Duration.ofMillis(100),
                java.time.Duration.ofMillis(500),
                java.time.Duration.ofSeconds(1)
            )
            .register(registry);

        paymentProcessingTimer = Timer.builder("payments.processing.duration")
            .description("Time to process payment")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        dbQueryTimer = Timer.builder("db.query.duration")
            .description("Database query execution time")
            .register(registry);

        // ─── Distribution Summary ──────────────────────────────────────────
        orderAmountSummary = DistributionSummary.builder("orders.amount")
            .description("Distribution of order amounts in USD")
            .baseUnit("USD")
            .publishPercentiles(0.5, 0.75, 0.95, 0.99)
            .publishPercentileHistogram()
            .minimumExpectedValue(1.0)
            .maximumExpectedValue(10_000.0)
            .register(registry);

        requestSizeSummary = DistributionSummary.builder("http.request.size")
            .description("HTTP request payload sizes in bytes")
            .baseUnit("bytes")
            .register(registry);
    }

    // ─── Public methods for use in services ───────────────────────────────

    public void recordOrderCreated() {
        ordersCreatedCounter.increment();
        activeOrdersGauge.incrementAndGet();
    }

    public void recordOrderFailed(String reason) {
        ordersFailedCounter.increment();
        activeOrdersGauge.decrementAndGet();
    }

    public void recordOrderCompleted(double amountUsd) {
        activeOrdersGauge.decrementAndGet();
        orderAmountSummary.record(amountUsd);
    }

    public Timer.Sample startOrderTimer() {
        return Timer.start();
    }

    public void stopOrderTimer(Timer.Sample sample) {
        sample.stop(orderProcessingTimer);
    }

    public void recordPaymentSuccess() {
        paymentSuccessCounter.increment();
        pendingPaymentsGauge.decrementAndGet();
    }

    public void recordPaymentFailure() {
        paymentFailureCounter.increment();
        pendingPaymentsGauge.decrementAndGet();
    }

    public void updateInventoryCount(long count) {
        inventoryGauge.set(count);
    }
}
```

### Using Metrics in Service Layer

```java
// src/main/java/com/example/monitoring/service/OrderService.java
package com.example.monitoring.service;

import com.example.monitoring.metrics.AppMetrics;
import com.example.monitoring.model.Order;
import com.example.monitoring.repository.OrderRepository;
import io.micrometer.core.annotation.Timed;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;
    private final AppMetrics metrics;
    private final MeterRegistry meterRegistry;

    public OrderService(OrderRepository orderRepository,
                        AppMetrics metrics,
                        MeterRegistry meterRegistry) {
        this.orderRepository = orderRepository;
        this.metrics = metrics;
        this.meterRegistry = meterRegistry;
    }

    @Timed(value = "orders.create.time", description = "Time to create an order")
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Timer.Sample sample = metrics.startOrderTimer();

        try {
            log.info("Creating order for customer: {}", request.getCustomerId());

            Order order = new Order();
            order.setCustomerId(request.getCustomerId());
            order.setAmount(request.getAmount());
            order.setStatus(OrderStatus.PENDING);

            Order saved = orderRepository.save(order);

            metrics.recordOrderCreated();
            meterRegistry.counter("orders.by.customer",
                "customerId", request.getCustomerId(),
                "tier", request.getCustomerTier()
            ).increment();

            return saved;

        } catch (Exception e) {
            metrics.recordOrderFailed(e.getClass().getSimpleName());
            throw e;
        } finally {
            metrics.stopOrderTimer(sample);
        }
    }

    @Timed(value = "orders.complete.time")
    @Transactional
    public Order completeOrder(String orderId, BigDecimal finalAmount) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(OrderStatus.COMPLETED);
        Order saved = orderRepository.save(order);

        metrics.recordOrderCompleted(finalAmount.doubleValue());

        return saved;
    }
}
```

---

## 5. Spring Boot Actuator Metrics Endpoint

### Available Metrics

```bash
# List all metric names
curl http://localhost:8080/actuator/metrics | jq '.names[]'

# Get specific metric
curl "http://localhost:8080/actuator/metrics/http.server.requests" | jq .

# Response:
# {
#   "name": "http.server.requests",
#   "description": "...",
#   "baseUnit": "seconds",
#   "measurements": [
#     {"statistic": "COUNT", "value": 1523},
#     {"statistic": "TOTAL_TIME", "value": 45.231},
#     {"statistic": "MAX", "value": 2.156}
#   ],
#   "availableTags": [
#     {"tag": "uri", "values": ["/api/orders", "/api/users"]},
#     {"tag": "method", "values": ["GET", "POST"]},
#     {"tag": "status", "values": ["200", "201", "404", "500"]}
#   ]
# }

# Filter by tag
curl "http://localhost:8080/actuator/metrics/http.server.requests?tag=status:500"

# Raw Prometheus format
curl http://localhost:8080/actuator/prometheus
```

### Custom Info Endpoint

```java
// src/main/java/com/example/monitoring/actuator/AppInfoContributor.java
package com.example.monitoring.actuator;

import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

import java.util.LinkedHashMap;
import java.util.Map;

@Component
public class AppInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> details = new LinkedHashMap<>();
        details.put("name", "Order Service");
        details.put("description", "Handles order processing and fulfillment");
        details.put("team", "Platform Engineering");
        details.put("slack-channel", "#order-service-alerts");
        details.put("runbook", "https://wiki.company.com/order-service");

        builder.withDetail("app", details);
    }
}
```

---

## 6. Grafana Dashboard Setup

### Grafana Datasource Configuration

```yaml
# grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    access: proxy
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo

  - name: Tempo
    type: tempo
    url: http://tempo:3200
    uid: tempo
    jsonData:
      httpMethod: GET
      tracesToLogs:
        datasourceUid: loki
        tags: ['app', 'instance']

  - name: Loki
    type: loki
    url: http://loki:3100
    uid: loki
```

### Dashboard as Code (JSON)

```json
{
  "dashboard": {
    "title": "Spring Boot Service Dashboard",
    "uid": "spring-boot-overview",
    "refresh": "30s",
    "panels": [
      {
        "title": "Request Rate",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"order-service\"}[5m]))",
          "legendFormat": "req/s"
        }]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"order-service\",status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count{application=\"order-service\"}[5m])) * 100",
          "legendFormat": "% errors"
        }]
      },
      {
        "title": "P95 Latency",
        "type": "stat",
        "targets": [{
          "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{application=\"order-service\"}[5m])) by (le)) * 1000",
          "legendFormat": "ms"
        }]
      },
      {
        "title": "JVM Heap Usage",
        "type": "gauge",
        "targets": [{
          "expr": "jvm_memory_used_bytes{area=\"heap\",application=\"order-service\"} / jvm_memory_max_bytes{area=\"heap\",application=\"order-service\"} * 100"
        }]
      }
    ]
  }
}
```

### Key PromQL Queries for Spring Boot

```promql
# ─── HTTP Metrics ─────────────────────────────────────────────────────────
# Request rate per second (5m window)
sum(rate(http_server_requests_seconds_count[5m])) by (application, uri)

# Error rate (5xx)
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
/
sum(rate(http_server_requests_seconds_count[5m])) by (application)

# P50, P95, P99 latency
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application)
)

# ─── JVM Metrics ──────────────────────────────────────────────────────────
# Heap usage %
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# GC pause rate
rate(jvm_gc_pause_seconds_sum[5m])

# Thread count
jvm_threads_live_threads

# ─── Database Pool ────────────────────────────────────────────────────────
# HikariCP active connections
hikaricp_connections_active

# Pool utilization
hikaricp_connections_active / hikaricp_connections_max * 100

# ─── Custom Business Metrics ──────────────────────────────────────────────
# Orders per minute
sum(rate(orders_created_total[1m])) * 60

# Payment failure rate
sum(rate(payments_total{status="failure"}[5m])) /
sum(rate(payments_total[5m])) * 100

# P95 order amount
histogram_quantile(0.95, sum(rate(orders_amount_bucket[5m])) by (le))
```

---

## 7. Alerting Rules in Prometheus

```yaml
# prometheus/alert_rules.yml
groups:
  - name: spring-boot-alerts
    interval: 30s
    rules:
      # ─── Availability ─────────────────────────────────────────────────
      - alert: ServiceDown
        expr: up{job=~".*-service"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.job }} is DOWN"
          description: "{{ $labels.instance }} has been down for more than 1 minute."
          runbook: "https://wiki.company.com/runbooks/service-down"

      # ─── Error Rate ───────────────────────────────────────────────────
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
          /
          sum(rate(http_server_requests_seconds_count[5m])) by (application)
          > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }} over 5 minutes."

      - alert: CriticalErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
          /
          sum(rate(http_server_requests_seconds_count[5m])) by (application)
          > 0.20
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical error rate on {{ $labels.application }}"

      # ─── Latency ──────────────────────────────────────────────────────
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application)
          ) > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency on {{ $labels.application }}"
          description: "P99 latency is {{ $value | humanizeDuration }} (threshold: 2s)"

      # ─── JVM ──────────────────────────────────────────────────────────
      - alert: HighHeapUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"}
          > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High JVM heap usage on {{ $labels.instance }}"
          description: "Heap is at {{ $value | humanizePercentage }}"

      - alert: HeapOOMRisk
        expr: |
          jvm_memory_used_bytes{area="heap"}
          /
          jvm_memory_max_bytes{area="heap"}
          > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "JVM heap near OOM on {{ $labels.instance }}"

      # ─── Database ─────────────────────────────────────────────────────
      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "DB connection pool near exhaustion"
          description: "Pool is {{ $value | humanizePercentage }} full"

      # ─── Business Metrics ─────────────────────────────────────────────
      - alert: NoOrdersCreated
        expr: sum(rate(orders_created_total[10m])) == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "No orders created in 10 minutes"
          description: "Possible issue with order intake pipeline"
```

### Alertmanager Configuration

```yaml
# alertmanager/alertmanager.yml
global:
  smtp_from: 'alerts@company.com'
  smtp_smarthost: 'mail.company.com:587'

route:
  group_by: ['alertname', 'application']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@company.com'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_KEY}'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
```

---

## 8. Log Aggregation with ELK Stack

### Elasticsearch, Logstash, Kibana Setup

```yaml
# docker-compose-elk.yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200/_cluster/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    ports:
      - "5044:5044"    # Beats input
      - "5001:5001"    # TCP input for direct log shipping
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

### Logstash Pipeline

```ruby
# logstash/pipeline/spring-boot.conf
input {
  tcp {
    port => 5001
    codec => json_lines
  }
  beats {
    port => 5044
  }
}

filter {
  # Parse Spring Boot JSON logs
  if [type] == "spring-boot" {
    mutate {
      rename => { "level" => "log_level" }
      rename => { "logger_name" => "logger" }
    }

    # Extract trace IDs for correlation
    if [mdc][traceId] {
      mutate {
        add_field => { "trace_id" => "%{[mdc][traceId]}" }
        add_field => { "span_id" => "%{[mdc][spanId]}" }
      }
    }

    # Parse exception stack traces
    if [stack_trace] {
      mutate {
        add_tag => ["exception"]
      }
    }

    # Add Geo-IP from client IP
    if [clientIp] {
      geoip {
        source => "clientIp"
        target => "geo"
      }
    }
  }

  # Drop health check logs to reduce noise
  if [message] =~ "actuator/health" {
    drop {}
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "spring-boot-%{[application]}-%{+YYYY.MM.dd}"
    template_name => "spring-boot"
  }

  # Debug output
  stdout {
    codec => rubydebug
  }
}
```

---

## 9. Structured Logging with Logback JSON

### logback-spring.xml

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- Import Spring Boot defaults -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="APP_NAME" source="spring.application.name"/>
    <springProperty scope="context" name="APP_ENV" source="app.environment" defaultValue="local"/>

    <!-- Console appender (plain text for local dev) -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}/%X{spanId}] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- JSON appender for production (Logstash-compatible) -->
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- Custom fields added to every log entry -->
            <customFields>{"application":"${APP_NAME}","environment":"${APP_ENV}"}</customFields>

            <!-- Include MDC fields (traceId, spanId, userId, etc.) -->
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>requestId</includeMdcKeyName>
            <includeMdcKeyName>sessionId</includeMdcKeyName>

            <!-- Field renaming for ELK compatibility -->
            <fieldNames>
                <timestamp>@timestamp</timestamp>
                <message>message</message>
                <logger>logger_name</logger>
                <thread>thread_name</thread>
                <level>level</level>
                <levelValue>level_value</levelValue>
            </fieldNames>

            <!-- Include stack trace as structured array -->
            <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                <maxDepthPerCause>20</maxDepthPerCause>
                <rootCauseFirst>true</rootCauseFirst>
            </throwableConverter>
        </encoder>
    </appender>

    <!-- Async appender to prevent logging from slowing down the app -->
    <appender name="ASYNC_JSON" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>1000</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <appender-ref ref="JSON"/>
    </appender>

    <!-- File appender for local debugging -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/${APP_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/${APP_NAME}-%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>7</maxHistory>
            <totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <!-- Reduce noisy framework loggers -->
    <logger name="org.springframework" level="WARN"/>
    <logger name="org.hibernate" level="WARN"/>
    <logger name="com.zaxxer.hikari" level="INFO"/>

    <!-- Application loggers -->
    <logger name="com.example" level="DEBUG"/>

    <!-- Root logger -->
    <root level="INFO">
        <springProfile name="local,test">
            <appender-ref ref="CONSOLE"/>
        </springProfile>
        <springProfile name="production,staging">
            <appender-ref ref="ASYNC_JSON"/>
        </springProfile>
    </root>
</configuration>
```

### Sample JSON Log Output

```json
{
  "@timestamp": "2024-01-15T10:23:45.123Z",
  "level": "INFO",
  "logger_name": "com.example.service.OrderService",
  "thread_name": "http-nio-8080-exec-3",
  "message": "Order created successfully",
  "application": "order-service",
  "environment": "production",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "userId": "user-123",
  "requestId": "req-abc-456",
  "orderId": "ord-789",
  "amount": 125.50,
  "duration_ms": 45
}
```

---

## 10. Correlation IDs Across Services

### MDC Filter for Incoming Requests

```java
// src/main/java/com/example/monitoring/filter/CorrelationIdFilter.java
package com.example.monitoring.filter;

import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.slf4j.MDC;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
@Order(1)
public class CorrelationIdFilter implements Filter {

    public static final String CORRELATION_HEADER = "X-Correlation-ID";
    public static final String REQUEST_ID_HEADER  = "X-Request-ID";

    private static final String MDC_CORRELATION_ID = "correlationId";
    private static final String MDC_REQUEST_ID     = "requestId";
    private static final String MDC_USER_AGENT     = "userAgent";
    private static final String MDC_CLIENT_IP      = "clientIp";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        try {
            // Use existing correlation ID or create new one
            String correlationId = httpRequest.getHeader(CORRELATION_HEADER);
            if (correlationId == null || correlationId.isBlank()) {
                correlationId = UUID.randomUUID().toString();
            }

            String requestId = UUID.randomUUID().toString();

            // Set MDC for this request's thread
            MDC.put(MDC_CORRELATION_ID, correlationId);
            MDC.put(MDC_REQUEST_ID, requestId);
            MDC.put(MDC_CLIENT_IP, getClientIp(httpRequest));
            MDC.put(MDC_USER_AGENT, httpRequest.getHeader("User-Agent"));

            // Propagate to response headers
            httpResponse.setHeader(CORRELATION_HEADER, correlationId);
            httpResponse.setHeader(REQUEST_ID_HEADER, requestId);

            chain.doFilter(request, response);

        } finally {
            MDC.clear();  // Always clear MDC to prevent leaks in thread pools
        }
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff != null && !xff.isEmpty()) {
            return xff.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

### WebClient with Correlation ID Propagation

```java
// src/main/java/com/example/monitoring/client/CorrelatingWebClient.java
package com.example.monitoring.client;

import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.ClientRequest;
import org.springframework.web.reactive.function.client.ExchangeFilterFunction;
import org.springframework.web.reactive.function.client.WebClient;

@Component
public class CorrelatingWebClient {

    private final WebClient webClient;

    public CorrelatingWebClient(WebClient.Builder builder) {
        this.webClient = builder
            .filter(propagateCorrelationId())
            .filter(logRequest())
            .build();
    }

    private ExchangeFilterFunction propagateCorrelationId() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            String correlationId = MDC.get("correlationId");
            if (correlationId != null) {
                return reactor.core.publisher.Mono.just(
                    ClientRequest.from(request)
                        .header("X-Correlation-ID", correlationId)
                        .build()
                );
            }
            return reactor.core.publisher.Mono.just(request);
        });
    }

    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            org.slf4j.LoggerFactory.getLogger(getClass())
                .debug("Outgoing request: {} {}", request.method(), request.url());
            return reactor.core.publisher.Mono.just(request);
        });
    }

    public WebClient getWebClient() {
        return webClient;
    }
}
```

### Distributed Tracing with Micrometer

```java
// src/main/java/com/example/monitoring/service/TracedOrderService.java
package com.example.monitoring.service;

import io.micrometer.observation.Observation;
import io.micrometer.observation.ObservationRegistry;
import io.micrometer.observation.annotation.Observed;
import org.springframework.stereotype.Service;

@Service
public class TracedOrderService {

    private final ObservationRegistry observationRegistry;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;

    public TracedOrderService(ObservationRegistry observationRegistry,
                               PaymentService paymentService,
                               InventoryService inventoryService) {
        this.observationRegistry = observationRegistry;
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }

    @Observed(name = "order.process",
              contextualName = "processing order",
              lowCardinalityKeyValues = {"service", "order-service"})
    public OrderResult processOrder(OrderRequest request) {
        return Observation.createNotStarted("order.process", observationRegistry)
            .lowCardinalityKeyValue("customerId", request.getCustomerId())
            .highCardinalityKeyValue("orderId", request.getOrderId())
            .observe(() -> {
                // Reserve inventory (creates child span)
                inventoryService.reserve(request.getItems());

                // Process payment (creates child span)
                PaymentResult payment = paymentService.charge(
                    request.getCustomerId(),
                    request.getTotalAmount()
                );

                return new OrderResult(request.getOrderId(), payment.getTransactionId());
            });
    }
}
```

---

## 11. Real Example: Complete Observability Stack with docker-compose

### Full docker-compose.yaml

```yaml
# docker-compose-observability.yaml
version: '3.8'

services:
  # ─── Spring Boot Services ─────────────────────────────────────────────
  order-service:
    build:
      context: ./order-service
    environment:
      SPRING_PROFILES_ACTIVE: production
      APP_ENVIRONMENT: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/orders
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      MANAGEMENT_ZIPKIN_TRACING_ENDPOINT: http://zipkin:9411/api/v2/spans
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - zipkin
    labels:
      prometheus.io/scrape: "true"
      prometheus.io/path: "/actuator/prometheus"
      prometheus.io/port: "8080"

  user-service:
    build:
      context: ./user-service
    environment:
      SPRING_PROFILES_ACTIVE: production
      APP_ENVIRONMENT: docker
    ports:
      - "8081:8080"

  # ─── Database ─────────────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data

  # ─── Metrics Stack ────────────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

  alertmanager:
    image: prom/alertmanager:v0.26.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml

  # ─── Tracing ──────────────────────────────────────────────────────────
  zipkin:
    image: openzipkin/zipkin:3.0
    ports:
      - "9411:9411"

  # ─── Logging Stack (ELK) ──────────────────────────────────────────────
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    ports:
      - "5001:5001"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

  # ─── Node Exporter (host metrics) ─────────────────────────────────────
  node-exporter:
    image: prom/node-exporter:v1.7.0
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  # ─── Cadvisor (container metrics) ─────────────────────────────────────
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    ports:
      - "8081:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

volumes:
  postgres-data:
  prometheus-data:
  grafana-data:
  es-data:
```

### Startup and Verification

```bash
# Start the complete stack
docker-compose -f docker-compose-observability.yaml up -d

# Check all services are healthy
docker-compose ps

# Verify metrics are being scraped
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Test alert rules
curl http://localhost:9090/api/v1/rules | jq '.data.groups[].rules[] | {name: .name, state: .state}'

# Check Prometheus metrics endpoint
curl http://localhost:8080/actuator/prometheus | head -50

# Verify Zipkin is receiving traces
curl http://localhost:9411/api/v2/services

# Search Elasticsearch
curl "http://localhost:9200/spring-boot-order-service-*/_count" | jq .

# Open dashboards
echo "Grafana: http://localhost:3000 (admin/admin)"
echo "Kibana: http://localhost:5601"
echo "Prometheus: http://localhost:9090"
echo "Zipkin: http://localhost:9411"
```

### Kibana Index Pattern Setup

```bash
# Create Kibana index pattern via API
curl -X POST "http://localhost:5601/api/index_patterns/index_pattern" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "index_pattern": {
      "title": "spring-boot-*",
      "timeFieldName": "@timestamp"
    }
  }'
```

### Operational Runbook Template

```java
// src/main/java/com/example/monitoring/runbook/OperationalChecks.java
package com.example.monitoring.runbook;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Comprehensive health check for operational use.
 * Accessible at /actuator/health/operational
 */
@Component("operational")
public class OperationalChecks implements HealthIndicator {

    private final MeterRegistry meterRegistry;

    public OperationalChecks(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public Health health() {
        Map<String, Object> details = new LinkedHashMap<>();

        // Check error rate
        double errorRate = getMetricValue("http_server_requests_seconds_count",
            "status", "5xx");
        details.put("httpErrorRate", String.format("%.2f%%", errorRate * 100));

        // Check active threads
        double activeThreads = getMetricValue("jvm_threads_live_threads");
        details.put("activeThreads", (int) activeThreads);

        // Check heap usage
        double heapUsed = getMetricValue("jvm_memory_used_bytes", "area", "heap");
        double heapMax = getMetricValue("jvm_memory_max_bytes", "area", "heap");
        double heapPercent = heapMax > 0 ? (heapUsed / heapMax) * 100 : 0;
        details.put("heapUsage", String.format("%.1f%%", heapPercent));

        boolean healthy = errorRate < 0.1 && heapPercent < 90;

        return healthy
            ? Health.up().withDetails(details).build()
            : Health.down().withDetails(details).build();
    }

    private double getMetricValue(String name, String... tags) {
        try {
            var gauge = meterRegistry.find(name).gauge();
            return gauge != null ? gauge.value() : 0.0;
        } catch (Exception e) {
            return 0.0;
        }
    }
}
```

---

## Summary Table

| Pillar | Tool | Spring Integration | Key Class |
|--------|------|-------------------|-----------|
| Metrics | Prometheus | Micrometer | `MeterRegistry`, `Counter`, `Timer`, `Gauge` |
| Metrics UI | Grafana | Dashboard JSON | PromQL queries |
| Alerting | Alertmanager | Prometheus rules | `alert_rules.yml` |
| Logs | Elasticsearch | LogstashEncoder | `logback-spring.xml` |
| Logs UI | Kibana | Index patterns | JSON structured logs |
| Tracing | Zipkin | Micrometer Tracing | `ObservationRegistry`, `@Observed` |
| Correlation | MDC | Servlet Filter | `CorrelationIdFilter` |
| HTTP metrics | Auto | Spring Boot Actuator | `http.server.requests.*` |
| JVM metrics | Auto | Micrometer | `jvm.memory.*`, `jvm.gc.*` |
| Custom metrics | Manual | `AppMetrics` bean | `@Timed`, custom `Counter` |

### SRE Golden Signals Checklist

| Signal | PromQL | Alert Threshold |
|--------|--------|----------------|
| Latency (P99) | `histogram_quantile(0.99, ...)` | > 2s |
| Traffic | `rate(http_server_requests_count[5m])` | sudden drop > 50% |
| Errors | `rate(count{status=~"5.."}[5m]) / rate(count[5m])` | > 5% |
| Saturation (CPU) | `process_cpu_usage` | > 80% |
| Saturation (Heap) | `jvm_memory_used/jvm_memory_max` | > 85% |
| Saturation (DB Pool) | `hikaricp_connections_active/max` | > 90% |

---

## What's Next

This is the final part of the current module! You now have a complete Java & Spring Boot course covering:

- **Core Java** (Parts 001–020): Language fundamentals, OOP, functional programming, concurrency
- **Spring Core** (Parts 021–024): IoC, DI, AOP, Actuator
- **Data & Security** (Parts 025–028): JPA, Spring Security with JWT
- **Advanced Spring** (Parts 031–034): Caching, WebFlux, Microservices, Docker
- **Cloud Native** (Parts 035–040): Kubernetes, RabbitMQ, OAuth2/OIDC, GraphQL, gRPC, Observability

Continue your journey with:
- **Event Sourcing & CQRS** with Spring and Axon Framework
- **Service Mesh** with Istio and Spring Boot
- **Cloud-Native Patterns** with AWS/GCP Spring Cloud integrations
- **Performance Tuning** for Spring Boot production workloads

---

*End of Part 040: Monitoring with Prometheus & Grafana*
