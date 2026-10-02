# Part 100: Capstone Project — Complete Production E-Commerce Platform

## เนื้อหาในส่วนนี้
- Final project architecture overview
- Complete multi-module project structure
- All services integrated and working together
- Deployment with Docker Compose
- Monitoring, security, and observability
- What makes a world-class Java developer

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                       │
│   Browser / Mobile App / Third-party APIs                           │
└─────────────────────────┬───────────────────────────────────────────┘
                           │ HTTPS
                    ┌──────▼──────┐
                    │  API Gateway │  Spring Cloud Gateway
                    │  :8080       │  (Auth, Rate Limiting, Routing)
                    └──────┬───────┘
                           │
        ┌──────────────────┼─────────────────────┐
        │                  │                     │
   ┌────▼─────┐    ┌───────▼──────┐    ┌────────▼─────┐
   │  User    │    │   Product    │    │   Order      │
   │ Service  │    │   Service    │    │   Service    │
   │  :8081   │    │   :8082      │    │   :8083      │
   └────┬─────┘    └───────┬──────┘    └────────┬─────┘
        │                  │                     │
   ┌────▼─────┐    ┌───────▼──────┐    ┌────────▼─────┐
   │PostgreSQL│    │  PostgreSQL  │    │  PostgreSQL  │
   │   +      │    │     +        │    │      +       │
   │  Redis   │    │Elasticsearch │    │    Kafka     │
   └──────────┘    └──────────────┘    └─────┬────────┘
                                             │
                                    ┌────────▼─────┐
                                    │   Payment    │
                                    │   Service    │
                                    │   :8084      │
                                    └─────┬────────┘
                                          │
                                 ┌────────▼──────┐
                                 │ Notification  │
                                 │   Service     │
                                 │   :8085       │
                                 └───────────────┘
                                 
Observability Stack:
  Prometheus + Grafana (metrics)
  Jaeger (distributed tracing)
  ELK Stack (centralized logging)
  Spring Boot Admin (service management)
```

---

## 2. Project Structure (Multi-Module Maven)

```
ecommerce-platform/
├── pom.xml                          # Parent POM
├── ecommerce-bom/                   # Bill of Materials
│   └── pom.xml
├── ecommerce-commons/               # Shared DTOs, utilities, exceptions
│   ├── src/main/java/
│   │   └── com/ecommerce/common/
│   │       ├── dto/                 # Shared DTOs
│   │       ├── exception/           # Common exceptions
│   │       ├── security/            # JWT utility
│   │       └── event/               # Domain event base classes
│   └── pom.xml
├── api-gateway/                     # Spring Cloud Gateway
├── user-service/                    # User management
├── product-service/                 # Product catalog + search
├── order-service/                   # Order processing + Saga
├── payment-service/                 # Payment processing
├── notification-service/            # Email/SMS/Push
├── infrastructure/
│   ├── docker-compose.yml           # Full stack
│   ├── kubernetes/                  # K8s manifests
│   └── monitoring/                  # Prometheus/Grafana configs
└── README.md
```

---

## 3. Parent POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.ecommerce</groupId>
    <artifactId>ecommerce-platform</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
    </parent>
    
    <modules>
        <module>ecommerce-bom</module>
        <module>ecommerce-commons</module>
        <module>api-gateway</module>
        <module>user-service</module>
        <module>product-service</module>
        <module>order-service</module>
        <module>payment-service</module>
        <module>notification-service</module>
    </modules>
    
    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2023.0.3</spring-cloud.version>
        <testcontainers.version>1.20.1</testcontainers.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.ecommerce</groupId>
                <artifactId>ecommerce-bom</artifactId>
                <version>${project.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

---

## 4. Order Service with Saga Pattern

```java
package com.ecommerce.order;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.*;
import org.springframework.transaction.annotation.Transactional;

// Order Saga Orchestrator
@Service
public class OrderSagaOrchestrator {
    
    private final KafkaTemplate<String, Object> kafka;
    private final OrderRepository orderRepository;
    private final SagaStateRepository sagaStateRepository;
    
    public OrderSagaOrchestrator(KafkaTemplate<String, Object> kafka,
                                   OrderRepository orderRepository,
                                   SagaStateRepository sagaStateRepository) {
        this.kafka = kafka;
        this.orderRepository = orderRepository;
        this.sagaStateRepository = sagaStateRepository;
    }
    
    // Step 1: Start saga when order is created
    @Transactional
    public void startSaga(Order order) {
        SagaState saga = SagaState.begin(order.getId());
        sagaStateRepository.save(saga);
        
        // Reserve inventory
        kafka.send("inventory.reserve", new ReserveInventoryCommand(
            order.getId(),
            order.getItems().stream()
                .map(i -> new InventoryItem(i.getProductId(), i.getQuantity()))
                .toList()
        ));
    }
    
    // Step 2: Inventory reserved → process payment
    @KafkaListener(topics = "inventory.reserved")
    @Transactional
    public void onInventoryReserved(InventoryReservedEvent event) {
        Order order = orderRepository.findById(event.orderId()).orElseThrow();
        
        kafka.send("payment.process", new ProcessPaymentCommand(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount()
        ));
    }
    
    // Step 3: Payment success → confirm order
    @KafkaListener(topics = "payment.processed")
    @Transactional
    public void onPaymentProcessed(PaymentProcessedEvent event) {
        Order order = orderRepository.findById(event.orderId()).orElseThrow();
        order.confirm();
        orderRepository.save(order);
        
        // Notify customer
        kafka.send("notification.send", new SendNotificationCommand(
            order.getCustomerId(),
            "ORDER_CONFIRMED",
            "Your order #" + order.getId() + " has been confirmed!"
        ));
    }
    
    // Compensation: inventory reservation failed
    @KafkaListener(topics = "inventory.reserve.failed")
    @Transactional
    public void onInventoryReserveFailed(InventoryReserveFailedEvent event) {
        Order order = orderRepository.findById(event.orderId()).orElseThrow();
        order.cancel("OUT_OF_STOCK");
        orderRepository.save(order);
        
        kafka.send("notification.send", new SendNotificationCommand(
            order.getCustomerId(),
            "ORDER_CANCELLED",
            "Sorry, your order was cancelled: item(s) out of stock"
        ));
    }
    
    // Compensation: payment failed → release inventory
    @KafkaListener(topics = "payment.failed")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepository.findById(event.orderId()).orElseThrow();
        order.cancel("PAYMENT_FAILED");
        orderRepository.save(order);
        
        // Release reserved inventory
        kafka.send("inventory.release", new ReleaseInventoryCommand(event.orderId()));
        
        kafka.send("notification.send", new SendNotificationCommand(
            order.getCustomerId(),
            "ORDER_CANCELLED",
            "Your order was cancelled: payment failed. Please try again."
        ));
    }
}

// Records for events/commands
record ReserveInventoryCommand(String orderId, java.util.List<InventoryItem> items) {}
record InventoryItem(String productId, int quantity) {}
record InventoryReservedEvent(String orderId) {}
record ProcessPaymentCommand(String orderId, String customerId, java.math.BigDecimal amount) {}
record PaymentProcessedEvent(String orderId, String paymentId) {}
record PaymentFailedEvent(String orderId, String reason) {}
record InventoryReserveFailedEvent(String orderId, String reason) {}
record ReleaseInventoryCommand(String orderId) {}
record SendNotificationCommand(String userId, String type, String message) {}
```

---

## 5. API Gateway Configuration

```yaml
# api-gateway/src/main/resources/application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "https://myapp.com"
            allowedMethods: [GET, POST, PUT, DELETE, OPTIONS]
            allowedHeaders: ["*"]
            allowCredentials: true
      routes:
        - id: user-service
          uri: lb://user-service
          predicates: [Path=/api/users/**, /api/auth/**]
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
            - name: CircuitBreaker
              args:
                name: user-service
                fallbackUri: forward:/fallback/users
        
        - id: product-service
          uri: lb://product-service
          predicates: [Path=/api/products/**, /api/search/**]
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 50
                redis-rate-limiter.burstCapacity: 100
        
        - id: order-service
          uri: lb://order-service
          predicates: [Path=/api/orders/**]
          filters:
            - AuthFilter  # Custom JWT validation
            - name: Retry
              args:
                retries: 3
                methods: GET
```

---

## 6. Complete Docker Compose

```yaml
# infrastructure/docker-compose.yml
version: '3.9'

services:
  # Infrastructure
  postgres-users:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: users_db
      POSTGRES_USER: users
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret}
    volumes:
      - postgres-users-data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U users
      interval: 10s
      retries: 5
  
  postgres-orders:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders_db
      POSTGRES_USER: orders
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret}
    volumes:
      - postgres-orders-data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U orders
      interval: 10s
      retries: 5
  
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD:-secret}
    volumes:
      - redis-data:/data
    healthcheck:
      test: redis-cli ping
      interval: 10s
  
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    healthcheck:
      test: kafka-broker-api-versions --bootstrap-server localhost:9092
      interval: 30s
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  
  elasticsearch:
    image: elasticsearch:8.13.4
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: -Xms512m -Xmx512m
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    healthcheck:
      test: curl -f http://localhost:9200/_cluster/health
      interval: 30s
  
  # Application Services
  user-service:
    build: ./user-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-users:5432/users_db
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD:-secret}
      SPRING_DATA_REDIS_HOST: redis
      JWT_SECRET: ${JWT_SECRET:-change-in-production-use-256-bit-secret}
    depends_on:
      postgres-users:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: curl -f http://localhost:8081/actuator/health/readiness
      interval: 30s
      start_period: 60s
  
  product-service:
    build: ./product-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-products:5432/products_db
      SPRING_ELASTICSEARCH_URIS: http://elasticsearch:9200
    depends_on:
      elasticsearch:
        condition: service_healthy
  
  order-service:
    build: ./order-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-orders:5432/orders_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      kafka:
        condition: service_healthy
      postgres-orders:
        condition: service_healthy
  
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      SPRING_CLOUD_GATEWAY_ROUTES_0_URI: http://user-service:8081
      SPRING_CLOUD_GATEWAY_ROUTES_1_URI: http://product-service:8082
      SPRING_CLOUD_GATEWAY_ROUTES_2_URI: http://order-service:8083
    depends_on:
      - user-service
      - product-service
      - order-service
  
  # Observability
  prometheus:
    image: prom/prometheus:v2.52.0
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
  
  grafana:
    image: grafana/grafana:10.4.2
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
  
  jaeger:
    image: jaegertracing/all-in-one:1.57
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTel gRPC
      - "4318:4318"    # OTel HTTP
  
  spring-boot-admin:
    image: michayaak/spring-boot-admin:3.3.1
    ports:
      - "8090:8090"

volumes:
  postgres-users-data:
  postgres-orders-data:
  redis-data:
  elasticsearch-data:
  grafana-data:
```

---

## 7. GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, product-service, order-service, payment-service]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven
      - name: Run Tests
        run: |
          cd ${{ matrix.service }}
          ../mvnw verify -P integration-test
      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          files: ${{ matrix.service }}/target/site/jacoco/jacoco.xml
  
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: OWASP Dependency Check
        run: ./mvnw dependency-check:aggregate
      - name: Trivy Security Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'HIGH,CRITICAL'
  
  build-and-push:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: [user-service, product-service, order-service, api-gateway]
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker Image
        run: |
          cd ${{ matrix.service }}
          docker build -t myorg/${{ matrix.service }}:${{ github.sha }} .
      - name: Push to Registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin
          docker push myorg/${{ matrix.service }}:${{ github.sha }}
  
  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/user-service \
            user-service=myorg/user-service:${{ github.sha }} \
            --namespace=staging
          kubectl rollout status deployment/user-service --namespace=staging
  
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Production (Canary)
        run: |
          # Deploy to 10% of traffic first
          kubectl apply -f k8s/canary-deployment.yml
          sleep 300  # 5 min observation
          kubectl apply -f k8s/full-deployment.yml
```

---

## 8. World-Class Java Developer Path

```
FOUNDATION (Parts 001-020)
├── Java syntax, OOP, generics, streams
├── Collections, concurrency, I/O
├── Testing (JUnit 5, Mockito)
└── Build tools (Maven, Gradle)

SPRING MASTERY (Parts 021-034)
├── Spring Boot, IoC/DI, AOP
├── Spring MVC, REST APIs
├── Spring Security, JWT/OAuth2
├── Spring Data JPA, Testing
├── Docker, Microservices basics
└── Spring Cloud (Gateway, Eureka)

ENTERPRISE ARCHITECTURE (Parts 035-070)
├── Kubernetes, Service Mesh
├── Kafka, RabbitMQ messaging
├── GraphQL, gRPC
├── Monitoring (Prometheus, Grafana, Jaeger)
├── DDD, Clean Architecture, Hexagonal
├── Event Sourcing, CQRS, Saga Pattern
├── Multi-tenancy, Advanced Security
├── Performance Tuning, Native Image
└── Spring AI, Advanced Patterns

PRODUCTION EXCELLENCE (Parts 071-100)
├── Advanced testing strategies
├── DevOps, CI/CD pipelines
├── Cloud (AWS, GCP) deployment
├── Security hardening
├── Observability and SRE practices
├── Performance testing (Gatling, k6)
└── Production readiness checklist

BEYOND WORLD-CLASS (Parts 101-104+)
├── Enterprise Integration Patterns
├── Apache Camel
├── JVM Internals
└── Java Security (Cryptography)

Key Mindset for World-Class Engineers:
✓ Write code that others can maintain
✓ Test before writing production code
✓ Security is everyone's responsibility
✓ Measure everything, optimize with data
✓ Design for failure, recover gracefully
✓ Keep learning - Java ecosystem evolves fast
```

---

## สรุป — จบหลักสูตร Java & Spring Boot

**คุณเรียนรู้อะไรบ้าง?**

| ระดับ | ทักษะ |
|-------|-------|
| พื้นฐาน | Java SE, OOP, Collections, Streams, Testing |
| กลาง | Spring Boot, REST, Security, JPA, Docker |
| ขั้นสูง | Microservices, Kafka, K8s, OAuth2, DDD |
| ระดับโลก | Architecture patterns, Performance, Cloud, AI |

**เส้นทางต่อไป:**
- 🏆 Java Champion certification
- ☁️ AWS/GCP/Azure certification  
- 🔒 Security specialist (CISSP, CEH)
- 📊 Data Engineering (Apache Spark, Flink)
- 🤖 AI/ML integration specialist

**"The best code is the code that works in production, is maintainable, and the team can sleep soundly at night."**

---

**ขอแสดงความยินดี! คุณเรียนจบหลักสูตร Java & Spring Boot ระดับโลกแล้ว 🎉**
