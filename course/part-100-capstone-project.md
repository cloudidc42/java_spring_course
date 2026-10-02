# Part 100: Capstone — Enterprise E-Commerce Platform

## Course Complete!

This final part brings together every concept from Parts 001–099 into a complete, production-ready reference architecture for an enterprise e-commerce platform. Every technology decision is justified, every service has a clear responsibility, and the entire system is designed to scale.

---

## Platform Overview

```
ShopCore — Enterprise E-Commerce Platform

Mission: Enable businesses to sell at any scale with a reliable,
         observable, and maintainable platform.

Scale targets:
- 10,000 concurrent users
- 500,000 orders/day
- 99.9% availability (< 8.7 hours downtime/year)
- < 200ms p95 API response time
```

---

## Complete Architecture Diagram

```
                        ┌─────────────────────────────────────────────────────┐
                        │                    Clients                          │
                        │     Web (React)   Mobile (iOS/Android)   Partners  │
                        └────────────────────┬────────────────────────────────┘
                                             │ HTTPS
                        ┌────────────────────▼────────────────────────────────┐
                        │                CloudFlare CDN                        │
                        │        (DDoS protection, edge caching)              │
                        └────────────────────┬────────────────────────────────┘
                                             │
                        ┌────────────────────▼────────────────────────────────┐
                        │             API Gateway (Kong)                       │
                        │   Rate limiting │ Auth │ Routing │ Request logging  │
                        └───┬────────────┬────────────┬──────────────┬────────┘
                            │            │            │              │
               ┌────────────▼─┐  ┌───────▼────┐  ┌──▼──────────┐  ┌▼───────────────┐
               │  Catalog     │  │  Order     │  │  Payment    │  │  User          │
               │  Service     │  │  Service   │  │  Service    │  │  Service       │
               │  :8081       │  │  :8082     │  │  :8083      │  │  :8084         │
               └──────┬───────┘  └──────┬─────┘  └──────┬──────┘  └──────┬─────────┘
                      │                 │                │                │
               ┌──────▼───────┐  ┌──────▼─────┐  ┌──────▼──────┐  ┌──────▼─────────┐
               │  Postgres    │  │  Postgres  │  │  Postgres   │  │  Postgres      │
               │  (catalog)   │  │  (orders)  │  │  (payments) │  │  (users)       │
               └──────────────┘  └──────┬─────┘  └─────────────┘  └────────────────┘
                                        │ Kafka events
               ┌────────────────────────▼────────────────────────────────────────────┐
               │                     Apache Kafka                                     │
               │  Topics: order.created  order.shipped  payment.processed  user.reg  │
               └───┬────────────────────────┬────────────────────────────┬───────────┘
                   │                        │                            │
          ┌────────▼────────┐    ┌──────────▼────────┐    ┌────────────▼──────────┐
          │  Notification   │    │   Inventory        │    │   Analytics           │
          │  Service        │    │   Service          │    │   Service             │
          │  (email/SMS)    │    │   :8085            │    │   :8086               │
          └─────────────────┘    └────────────────────┘    └───────────────────────┘
                                                                        │
                                                             ┌──────────▼─────────┐
                                                             │   ClickHouse       │
                                                             │   (analytics DB)   │
                                                             └────────────────────┘

Shared infrastructure:
  Redis Cluster (caching, sessions, rate limiting)
  Elasticsearch (search)
  Keycloak (identity provider)
  Zipkin/Jaeger (distributed tracing)
  Prometheus + Grafana (metrics + dashboards)
  ELK Stack (centralized logging)
```

---

## All Services: Responsibilities

### 1. Catalog Service

```
Responsibility: Product information, categories, pricing, inventory display
Technology: Spring Boot + PostgreSQL + Elasticsearch + Redis
Team: Product Team (3 engineers)
```

```java
// src/main/java/com/shopcore/catalog/CatalogServiceApplication.java
package com.shopcore.catalog;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.data.elasticsearch.repository.config.EnableElasticsearchRepositories;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableCaching
@EnableAsync
@EnableJpaRepositories(basePackages = "com.shopcore.catalog.adapter.output.persistence")
@EnableElasticsearchRepositories(basePackages = "com.shopcore.catalog.adapter.output.search")
public class CatalogServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(CatalogServiceApplication.class, args);
    }
}
```

```java
// Architecture: Hexagonal (from Part 094)
// Domain:
//   Product (aggregate root)
//   Category (aggregate root)
//   PriceRule (value object)
//   ProductVariant (entity within Product)
//
// Input Ports:
//   SearchProductsUseCase
//   GetProductUseCase
//   UpdateProductUseCase
//
// Output Ports:
//   ProductRepository (JPA)
//   ProductSearchRepository (Elasticsearch)
//   PricingPort (rule engine)
//   CachePort (Redis)
```

### 2. Order Service

```
Responsibility: Order lifecycle management, order state machine
Technology: Spring Boot + PostgreSQL + Spring State Machine + Kafka (producer)
Team: Commerce Team (4 engineers)
```

```java
// Key state machine transitions (from Part 091):
// CART → CHECKOUT → PENDING_PAYMENT → PAYMENT_RECEIVED
//   → CONFIRMED → PROCESSING → SHIPPED → DELIVERED
//
// Cancellation: from any pre-SHIPPED state
// Refund: from DELIVERED within 30 days

// Events produced to Kafka:
// - order.created
// - order.confirmed
// - order.shipped
// - order.delivered
// - order.cancelled
```

### 3. Payment Service

```
Responsibility: Payment processing, refunds, fraud detection
Technology: Spring Boot + PostgreSQL + Stripe/PayPal integration
Team: Payments Team (3 engineers, PCI DSS scope)
Security: PCI DSS Level 1 compliance required
```

```java
// src/main/java/com/shopcore/payment/adapter/output/stripe/StripePaymentAdapter.java
package com.shopcore.payment.adapter.output.stripe;

import com.shopcore.payment.application.port.output.PaymentGateway;
import com.stripe.Stripe;
import com.stripe.model.PaymentIntent;
import com.stripe.param.PaymentIntentCreateParams;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;

@Slf4j
@Component
@RequiredArgsConstructor
public class StripePaymentAdapter implements PaymentGateway {

    @Value("${stripe.api-key}")
    private String apiKey;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        Stripe.apiKey = apiKey;

        try {
            long amountInCents = request.amount()
                .multiply(BigDecimal.valueOf(100)).longValueExact();

            PaymentIntent intent = PaymentIntent.create(
                PaymentIntentCreateParams.builder()
                    .setAmount(amountInCents)
                    .setCurrency(request.currency().toLowerCase())
                    .setPaymentMethod(request.paymentMethodId())
                    .setConfirm(true)
                    .putMetadata("order_id", request.orderId())
                    .setReturnUrl("https://shopcore.example.com/payment/return")
                    .build()
            );

            return PaymentResult.success(intent.getId(), request.amount());

        } catch (com.stripe.exception.CardException e) {
            log.warn("Card declined for order {}: {}", request.orderId(), e.getDeclineCode());
            return PaymentResult.failure(e.getDeclineCode(), e.getMessage());
        } catch (Exception e) {
            log.error("Stripe error for order {}", request.orderId(), e);
            throw new PaymentGatewayException("Payment processing failed", e);
        }
    }
}
```

### 4. User Service

```
Responsibility: User accounts, profiles, addresses, wishlists
Technology: Spring Boot + PostgreSQL + Keycloak integration
Team: Platform Team (2 engineers)
```

### 5. Inventory Service

```
Responsibility: Stock levels, warehouse management, reservations
Technology: Spring Boot + PostgreSQL + Redis (optimistic locking for reservations)
Team: Commerce Team (2 engineers)
```

```java
// src/main/java/com/shopcore/inventory/service/StockReservationService.java
package com.shopcore.inventory.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Duration;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class StockReservationService {

    private final StockRepository stockRepository;
    private final RedisTemplate<String, Integer> redisTemplate;

    private static final Duration RESERVATION_TTL = Duration.ofMinutes(30);
    private static final String RESERVATION_KEY = "stock:reservation:";

    /**
     * Reserve stock atomically using Redis Lua script.
     * Returns false if insufficient stock.
     */
    public boolean reserveStock(String orderId, List<ReservationItem> items) {
        // Use Redis for fast availability check (eventual consistency with DB)
        for (ReservationItem item : items) {
            String key = RESERVATION_KEY + item.productId();
            Integer available = redisTemplate.opsForValue().get(key);

            if (available == null) {
                // Cache miss - load from DB
                available = stockRepository.getAvailableStock(item.productId());
                redisTemplate.opsForValue().set(key, available, Duration.ofMinutes(5));
            }

            if (available < item.quantity()) {
                log.info("Insufficient stock for product {}: needed {}, available {}",
                    item.productId(), item.quantity(), available);
                return false;
            }
        }

        // All items available - create reservations in DB
        items.forEach(item ->
            stockRepository.createReservation(orderId, item.productId(),
                item.quantity(), RESERVATION_TTL));

        return true;
    }

    @Transactional
    public void confirmReservation(String orderId) {
        stockRepository.confirmReservation(orderId);
        // Invalidate cache for reserved products
        stockRepository.getReservationItems(orderId)
            .forEach(item -> redisTemplate.delete(RESERVATION_KEY + item.productId()));
    }

    @Transactional
    public void releaseReservation(String orderId) {
        stockRepository.releaseReservation(orderId);
        stockRepository.getReservationItems(orderId)
            .forEach(item -> redisTemplate.delete(RESERVATION_KEY + item.productId()));
    }

    record ReservationItem(String productId, int quantity) {}
}
```

### 6. Notification Service

```
Responsibility: Email, SMS, push notifications triggered by domain events
Technology: Spring Boot + Kafka (consumer) + SendGrid + Twilio
Team: Platform Team (1 engineer)
```

```java
// src/main/java/com/shopcore/notification/consumer/OrderEventConsumer.java
package com.shopcore.notification.consumer;

import com.shopcore.events.OrderCreatedEvent;
import com.shopcore.events.OrderShippedEvent;
import com.shopcore.notification.service.EmailService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final EmailService emailService;
    private final PushNotificationService pushService;

    @KafkaListener(
        topics = "order.created",
        groupId = "notification-service",
        concurrency = "3"
    )
    public void handleOrderCreated(OrderCreatedEvent event, Acknowledgment ack) {
        log.info("Processing order.created event for order: {}", event.getOrderId());
        try {
            emailService.sendOrderConfirmation(event.getCustomerEmail(),
                event.getOrderId(), event.getItems());
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to send order confirmation for {}: {}",
                event.getOrderId(), e.getMessage());
            // Don't ack - message will be retried
        }
    }

    @KafkaListener(topics = "order.shipped", groupId = "notification-service")
    public void handleOrderShipped(OrderShippedEvent event, Acknowledgment ack) {
        emailService.sendShippingNotification(event.getCustomerEmail(),
            event.getOrderId(), event.getTrackingNumber());
        pushService.sendTrackingUpdate(event.getCustomerId(), event.getTrackingNumber());
        ack.acknowledge();
    }
}
```

### 7. Analytics Service

```
Responsibility: Business intelligence, reporting, funnel analysis
Technology: Spring Boot + ClickHouse + Kafka (consumer)
Team: Data Team (1 engineer)
Access: Read-only, no writes to other services
```

---

## Technology Stack Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Language** | Java 21 | LTS, virtual threads, records, sealed classes |
| **Framework** | Spring Boot 3.2 | Ecosystem maturity, team expertise, native image support |
| **Persistence** | PostgreSQL 16 | ACID, JSON support, mature tooling |
| **Cache** | Redis 7 Cluster | Horizontal scaling, pub/sub, Lua atomicity |
| **Messaging** | Apache Kafka | High throughput, durable, event replay |
| **Search** | Elasticsearch 8 | Full-text search, facets, analyzers |
| **Identity** | Keycloak 23 | Open source, OAuth2/OIDC, fine-grained auth |
| **API Gateway** | Kong | Plugin ecosystem, rate limiting, transforms |
| **Service Mesh** | Istio | mTLS, traffic management, observability |
| **Container** | Docker + Kubernetes | Industry standard, horizontal scaling |
| **IaC** | Terraform + Helm | Declarative, reproducible environments |
| **Observability** | Prometheus + Grafana + Jaeger | Open source stack, wide support |
| **CI/CD** | GitHub Actions | Integrated with repo, fast, flexible |
| **Secrets** | HashiCorp Vault | Dynamic secrets, rotation, audit trail |

---

## Development Workflow and Standards

```bash
# Branch naming convention
feature/SHOP-1234-add-variant-pricing
bugfix/SHOP-5678-fix-inventory-deadlock
hotfix/SHOP-9012-payment-timeout

# Commit message format (Conventional Commits)
feat(catalog): add variant-level pricing rules (#1234)
fix(inventory): prevent deadlock on concurrent reservations (#5678)
perf(order): replace N+1 queries with single aggregation query (#9012)

# Code review requirements
- 2 approvals required for main branch
- 1 approval for hotfixes (immediate review after merge)
- All CI checks must pass
- No commented-out code
- Test coverage must not decrease

# Definition of Done
✓ Unit tests written (coverage >= 80% for new code)
✓ Integration tests for happy path and key error cases
✓ API documentation updated (OpenAPI spec)
✓ Architecture Decision Record (ADR) written for significant decisions
✓ Monitoring dashboard updated (Grafana)
✓ Runbook updated if operational behavior changes
✓ Security scan passed (OWASP, Trivy)
✓ Load tested for endpoints expected to handle > 100 rps
```

---

## Deployment Architecture (Kubernetes)

```yaml
# k8s/catalog-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: production
  labels:
    app: catalog-service
    version: "2.1.0"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deployments
  selector:
    matchLabels:
      app: catalog-service
  template:
    metadata:
      labels:
        app: catalog-service
        version: "2.1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: catalog-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: catalog-service
          image: registry.example.com/shopcore/catalog-service:2.1.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalog-db-credentials
                  key: password
            - name: JAVA_OPTS
              value: >-
                -XX:+UseZGC
                -Xms512m -Xmx2g
                -XX:MaxDirectMemorySize=256m
                -Dserver.tomcat.threads.max=200
          resources:
            requests:
              memory: "768Mi"
              cpu: "250m"
            limits:
              memory: "2.5Gi"
              cpu: "2000m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            failureThreshold: 4
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: catalog-service
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalog-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalog-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
```

---

## Monitoring and Alerting

```java
// src/main/java/com/shopcore/catalog/monitoring/CatalogMetrics.java
package com.shopcore.catalog.monitoring;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;

@Component
@RequiredArgsConstructor
public class CatalogMetrics {

    private final MeterRegistry registry;

    // Business metrics (what matters to the business)
    public void recordProductSearch(String query, int resultCount, long durationMs) {
        Counter.builder("catalog.search.requests")
            .description("Number of product searches")
            .tag("has_results", String.valueOf(resultCount > 0))
            .register(registry)
            .increment();

        Timer.builder("catalog.search.duration")
            .description("Product search duration")
            .tag("result_range", resultCount == 0 ? "empty" :
                resultCount < 10 ? "low" : resultCount < 50 ? "medium" : "high")
            .register(registry)
            .record(durationMs, java.util.concurrent.TimeUnit.MILLISECONDS);
    }

    public void recordProductView(String productId, String source) {
        Counter.builder("catalog.product.views")
            .description("Product page views")
            .tag("source", source)
            .register(registry)
            .increment();
    }

    public void recordOutOfStock(String productId, String categoryName) {
        Counter.builder("catalog.stock.out_of_stock_views")
            .description("Views of out-of-stock products")
            .tag("category", categoryName)
            .register(registry)
            .increment();
    }
}
```

```yaml
# monitoring/alerts.yaml (Prometheus rules)
groups:
  - name: shopcore-alerts
    rules:
      # Service availability
      - alert: ServiceDown
        expr: up{job=~".*-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          runbook_url: "https://runbook.shopcore.internal/service-down"

      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
          / rate(http_server_requests_seconds_count[5m]) > 0.01
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.uri }}"

      # Slow responses
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95,
            rate(http_server_requests_seconds_bucket[5m])) > 1.0
        for: 5m
        labels:
          severity: warning

      # Database connection pool exhaustion
      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_pending{pool="HikariPool-1"} > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool exhausted - {{ $value }} threads waiting"

      # Business alert: Zero orders in 15 minutes
      - alert: ZeroOrdersAlert
        expr: increase(order_created_total[15m]) == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "No orders created in the last 15 minutes"
          description: "This is abnormal - investigate immediately"

      # Payment failure spike
      - alert: PaymentFailureSpike
        expr: |
          rate(payment_failures_total[5m]) > 0.1
        for: 3m
        labels:
          severity: critical
```

---

## Security Architecture

```
Authentication:
  - Users: OAuth2/OIDC via Keycloak
  - Services: mTLS (Istio) + OAuth2 Client Credentials
  - Admin: MFA required

Authorization:
  - User-level: JWT roles (ROLE_CUSTOMER, ROLE_MERCHANT, ROLE_ADMIN)
  - Service-level: OPA policies
  - Resource-level: Data isolation per tenant

Data Security:
  - Encryption at rest: AWS RDS encrypted
  - Encryption in transit: TLS 1.3 everywhere (Istio)
  - Secrets: HashiCorp Vault with rotation
  - PII: Pseudonymized in analytics, encrypted in storage

Network Security:
  - Network Policies per namespace
  - No direct access to databases from outside cluster
  - API Gateway is the single entry point
  - VPC with private subnets for all services
```

---

## Performance Targets and SLAs

| Endpoint | Target p50 | Target p95 | Target p99 |
|---|---|---|---|
| Product search | 50ms | 200ms | 500ms |
| Product detail | 20ms | 80ms | 200ms |
| Order creation | 100ms | 400ms | 1000ms |
| Payment processing | 300ms | 800ms | 2000ms |
| Cart operations | 30ms | 100ms | 300ms |
| User profile | 20ms | 60ms | 150ms |
| Inventory check | 10ms | 40ms | 100ms |

**Availability SLA: 99.9% (8.7 hours downtime/year)**

```java
// src/main/java/com/shopcore/shared/sla/SlaMonitor.java
package com.shopcore.shared.sla;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

/**
 * AOP aspect to automatically record SLA compliance for all endpoints.
 */
@Aspect
@Component
@RequiredArgsConstructor
public class SlaMonitor {

    private final MeterRegistry registry;
    private final SlaTargets slaTargets;

    @Around("@annotation(slaTracked)")
    public Object trackSla(ProceedingJoinPoint pjp, SlaTracked slaTracked) throws Throwable {
        String operationName = slaTracked.value();
        long start = System.nanoTime();

        try {
            Object result = pjp.proceed();
            long durationMs = (System.nanoTime() - start) / 1_000_000;

            recordSlaCompliance(operationName, durationMs, true);
            return result;
        } catch (Exception e) {
            long durationMs = (System.nanoTime() - start) / 1_000_000;
            recordSlaCompliance(operationName, durationMs, false);
            throw e;
        }
    }

    private void recordSlaCompliance(String operation, long durationMs, boolean success) {
        SlaTargets.Target target = slaTargets.getTarget(operation);
        boolean withinSla = success && (target == null || durationMs <= target.p95());

        registry.counter("sla.compliance",
            "operation", operation,
            "met", String.valueOf(withinSla)
        ).increment();
    }
}
```

---

## Team Structure and Code Ownership

```
CTO
└── VP Engineering
    ├── Commerce Team (6 engineers)
    │   ├── Order Service (CODEOWNERS: @commerce-team)
    │   ├── Cart Service (CODEOWNERS: @commerce-team)
    │   └── Inventory Service (CODEOWNERS: @commerce-team)
    ├── Product Team (4 engineers)
    │   ├── Catalog Service (CODEOWNERS: @product-team)
    │   └── Search Integration (CODEOWNERS: @product-team)
    ├── Payments Team (4 engineers, PCI DSS)
    │   └── Payment Service (CODEOWNERS: @payments-team)
    ├── Platform Team (5 engineers)
    │   ├── User Service (CODEOWNERS: @platform-team)
    │   ├── Notification Service (CODEOWNERS: @platform-team)
    │   ├── API Gateway config (CODEOWNERS: @platform-team)
    │   └── Kubernetes manifests (CODEOWNERS: @platform-team)
    └── Data Team (2 engineers)
        └── Analytics Service (CODEOWNERS: @data-team)
```

```
# .github/CODEOWNERS
/services/order-service/           @shopcore/commerce-team
/services/catalog-service/         @shopcore/product-team
/services/payment-service/         @shopcore/payments-team @shopcore/security-team
/infrastructure/kubernetes/        @shopcore/platform-team
/infrastructure/terraform/         @shopcore/platform-team
/.github/workflows/                @shopcore/platform-team
```

---

## Runbook: Common Operations

### Runbook 1: Service Restart

```bash
#!/bin/bash
# runbooks/restart-service.sh

SERVICE_NAME=${1:-"catalog-service"}
NAMESPACE=${2:-"production"}

echo "Restarting $SERVICE_NAME in $NAMESPACE"

# 1. Check current status
kubectl get pods -n $NAMESPACE -l app=$SERVICE_NAME

# 2. Rolling restart (zero downtime)
kubectl rollout restart deployment/$SERVICE_NAME -n $NAMESPACE

# 3. Wait for rollout
kubectl rollout status deployment/$SERVICE_NAME -n $NAMESPACE --timeout=5m

# 4. Verify pods are healthy
kubectl get pods -n $NAMESPACE -l app=$SERVICE_NAME

echo "Restart complete"
```

### Runbook 2: Database Connection Issues

```bash
# runbooks/db-diagnostics.sh

echo "=== Database Connection Pool Status ==="
curl -s http://catalog-service:8080/actuator/metrics/hikaricp.connections.active | jq
curl -s http://catalog-service:8080/actuator/metrics/hikaricp.connections.pending | jq

echo "=== PostgreSQL Active Connections ==="
kubectl exec -n production deploy/postgres -- psql -U postgres -c \
  "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

echo "=== Long Running Queries ==="
kubectl exec -n production deploy/postgres -- psql -U postgres -c \
  "SELECT pid, now() - query_start AS duration, query
   FROM pg_stat_activity
   WHERE state != 'idle'
   ORDER BY duration DESC LIMIT 10;"

echo "=== Blocking Queries ==="
kubectl exec -n production deploy/postgres -- psql -U postgres -c \
  "SELECT pid, usename, pg_blocking_pids(pid) AS blocked_by, query
   FROM pg_stat_activity
   WHERE cardinality(pg_blocking_pids(pid)) > 0;"
```

### Runbook 3: Cache Invalidation

```bash
# runbooks/cache-invalidation.sh

SERVICE=${1:-"catalog-service"}
CACHE_KEY_PATTERN=${2:-"*"}

echo "Invalidating cache for $SERVICE (pattern: $CACHE_KEY_PATTERN)"

# Via Actuator endpoint (if enabled)
curl -X POST "http://$SERVICE:8080/actuator/caches" \
  -H "Content-Type: application/json" \
  -d '{"name": "products"}'

# Or directly via Redis
kubectl exec -n production deploy/redis -- redis-cli \
  --scan --pattern "$CACHE_KEY_PATTERN" | xargs redis-cli DEL

echo "Cache invalidation complete"
```

### Runbook 4: Scale Service

```bash
# runbooks/scale-service.sh

SERVICE=$1
REPLICAS=$2
NAMESPACE=${3:-"production"}

echo "Scaling $SERVICE to $REPLICAS replicas"

kubectl scale deployment/$SERVICE \
  --replicas=$REPLICAS \
  -n $NAMESPACE

# Temporarily disable HPA to prevent it from overriding
kubectl annotate hpa/$SERVICE-hpa \
  "autoscaling.alpha.kubernetes.io/current-metrics-" \
  -n $NAMESPACE

echo "Check status with: kubectl get pods -n $NAMESPACE -l app=$SERVICE"
```

---

## Future Roadmap

### Quarter 1 (Next 3 months)
- [ ] Implement GraalVM native image builds (50% faster startup, 70% lower memory)
- [ ] Add Product Recommendation Engine (ML service with TensorFlow Serving)
- [ ] Multi-region active-active deployment (AWS us-east-1 + eu-west-1)
- [ ] Real-time inventory updates via WebSocket

### Quarter 2
- [ ] Migrate to Virtual Threads (Java 21) for improved throughput
- [ ] Add A/B testing framework for checkout flow
- [ ] Implement CQRS for Catalog Service (separate read/write models)
- [ ] GraphQL federation layer for flexible client queries

### Quarter 3
- [ ] Event Sourcing for Order Service (complete audit trail)
- [ ] Add Machine Learning fraud detection in Payment Service
- [ ] Multi-tenant architecture for white-label customers
- [ ] CDP integration for personalization

### Quarter 4
- [ ] Chaos Engineering practice with Chaos Monkey for Kubernetes
- [ ] Implement SRE practices (SLO/SLI dashboards, error budgets)
- [ ] Evaluate Dapr for service-to-service communication abstraction
- [ ] Green IT: carbon footprint monitoring per service

---

## Course Learning Path Summary

```
Foundation (Parts 001–020)
  Java fundamentals → OOP → Generics → Collections → Streams
  Concurrency → Modern Java → Design Patterns → Testing → Build tools

Spring Core (Parts 021–040)
  IoC/DI → AOP → MVC → Boot → Data JPA → Security → Testing
  Cache → WebFlux → Microservices → Docker → Kubernetes

Integration & Messaging (Parts 036–050)
  RabbitMQ → Kafka → gRPC → REST advanced → Cloud AWS

Advanced Patterns (Parts 051–070)
  DDD → CQRS/ES → Saga → Circuit Breaker → Distributed tracing
  Feature flags → API versioning → GraphQL

Production Engineering (Parts 071–090)
  Observability → Alerting → Chaos engineering → SRE → Cost optimization
  Zero-downtime deployments → Database migration → Performance tuning

Expert Level (Parts 091–100)
  State machines → Functional programming → Protocol Buffers & gRPC
  Hexagonal architecture → Testing patterns → Spring Data REST
  Advanced JPA → Microservices security → Performance testing
  Capstone project (this part!)
```

---

## Skills You Now Have

| Skill | Demonstrated In |
|---|---|
| Design scalable distributed systems | This part, Parts 033–035 |
| Model complex domains with DDD | Parts 051, 094 |
| Write maintainable, testable code | Parts 019, 030, 095 |
| Secure microservices with mTLS/JWT | Parts 026–027, 098 |
| Profile and optimize JVM applications | Part 099, 041 |
| Design event-driven architectures | Parts 036, 052–053 |
| Build reactive applications | Part 032 |
| Operate Kubernetes deployments | Parts 035, 100 |
| Implement observability | Parts 024, 073–074 |
| Apply functional patterns | Parts 013–014, 092 |

---

## What's Next?

**Specialize deeper:**
- Platform Engineering: Kubernetes operators, custom controllers
- Data Engineering: Apache Flink, Kafka Streams for real-time processing
- ML Engineering: Serving ML models alongside Spring Boot services
- Security Engineering: Penetration testing, threat modeling

**Stay current:**
- Follow Spring Blog (spring.io/blog)
- JVM weekly newsletter
- InfoQ architecture articles
- SpringOne conference talks (free on YouTube)

**Contribute:**
- Open source contributions to Spring ecosystem
- Write about what you've built
- Mentor junior developers using this course

---

*This course took you from writing your first Java variable to designing an enterprise platform that handles 500,000 orders per day. The journey of 100 parts reflects the real journey of becoming a senior Java/Spring engineer.*

*The best engineers never stop learning. Every production incident is a lesson. Every code review is an opportunity. Every new feature is a chance to apply these patterns better than you did before.*

*Go build something great.*

---

## Course Complete! 🎓

```
   ╔══════════════════════════════════════════════════════════╗
   ║                                                          ║
   ║   Congratulations on completing the Java & Spring        ║
   ║   Boot Course — Parts 001 through 100!                  ║
   ║                                                          ║
   ║   You've covered:                                        ║
   ║   • Core Java (Variables → Concurrency → Modern Java)   ║
   ║   • Spring Framework (IoC → AOP → MVC → Security)       ║
   ║   • Microservices (Design → Deploy → Operate)           ║
   ║   • Architecture (Hexagonal → DDD → CQRS)               ║
   ║   • Production Engineering (Observability → Performance)║
   ║                                                          ║
   ╚══════════════════════════════════════════════════════════╝
```
