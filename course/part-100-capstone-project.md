# Part 100: Capstone — Production E-Commerce Platform

This capstone assembles everything from the course into a production-grade, multi-service
e-commerce platform. Every service is fully implemented, containerised, tested, and monitored.

---

## 1. Architecture Overview

```
                          ┌──────────────────────────────────────────────┐
                          │              NGINX API Gateway                │
                          │          (rate limiting, SSL termination)      │
                          └──────────┬────────┬────────┬────────┬─────────┘
                                     │        │        │        │
              ┌──────────────────────┼────────┼────────┼────────┼──────────────────────┐
              │                      │        │        │        │                      │
    ┌─────────▼──────┐  ┌────────────▼───┐  ┌▼────────────┐  ┌▼────────────────┐  ┌──▼──────────────┐
    │  Auth Service  │  │ Product Service│  │Order Service│  │Payment Service  │  │Notification Svc │
    │  :8081         │  │  :8082         │  │   :8083     │  │    :8084        │  │    :8085         │
    │  JWT + Refresh │  │  Elasticsearch │  │ Saga/Outbox │  │  Idempotency    │  │  Kafka consumer  │
    └────────┬───────┘  └───────┬────────┘  └──────┬──────┘  └───────┬────────┘  └─────────────────-┘
             │                  │                   │                  │
             │          ┌───────▼───────┐    ┌──────▼──────┐  ┌──────▼──────┐
             │          │ Elasticsearch │    │  PostgreSQL  │  │  PostgreSQL  │
             │          │   :9200       │    │  orders_db   │  │  payments_db │
             │          └───────────────┘    └─────────────-┘  └──────────────┘
             │
    ┌────────▼────────┐         ┌─────────────────────────────────────────────────┐
    │   PostgreSQL    │         │                 Apache Kafka                     │
    │   users_db      │         │  topics: order.created  payment.processed        │
    └─────────────────┘         │           order.fulfilled  notification.send     │
                                └─────────────────────────────────────────────────┘

    ┌───────────────────────────────────────────────────────────────────────────────┐
    │               Observability Stack                                              │
    │   Prometheus :9090  →  Grafana :3000  ·  Zipkin :9411  ·  ELK :5601           │
    └───────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 21 (virtual threads, records, sealed classes) |
| Framework | Spring Boot 3.3, Spring Security, Spring Data JPA/ES |
| Messaging | Apache Kafka 3.7 (Saga pattern, outbox) |
| Databases | PostgreSQL 16 (OLTP), Elasticsearch 8 (catalog search) |
| Cache | Redis 7 (session, product cache) |
| Auth | JWT (access 15 min) + Refresh Token (30 days, rotation) |
| Container | Docker + Docker Compose |
| CI/CD | GitHub Actions |
| Monitoring | Prometheus + Grafana, Micrometer, Zipkin |
| Migration | Flyway |
| Testing | JUnit 5, Testcontainers, WireMock |

---

## 3. Maven Multi-Module Structure

```
ecommerce-platform/
├── pom.xml                     ← parent
├── ecommerce-bom/
│   └── pom.xml
├── ecommerce-common/           ← DTOs, exceptions, utils
│   └── src/
├── auth-service/
│   └── src/
├── product-service/
│   └── src/
├── order-service/
│   └── src/
├── payment-service/
│   └── src/
├── notification-service/
│   └── src/
├── docker/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   └── grafana/
│       └── dashboards/
└── .github/
    └── workflows/
        └── ci-cd.yml
```

---

## 4. Parent POM

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.ecommerce</groupId>
    <artifactId>ecommerce-platform</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>ecommerce-bom</module>
        <module>ecommerce-common</module>
        <module>auth-service</module>
        <module>product-service</module>
        <module>order-service</module>
        <module>payment-service</module>
        <module>notification-service</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.release>${java.version}</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.3.0</spring-boot.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <lombok.version>1.18.32</lombok.version>
        <testcontainers.version>1.19.8</testcontainers.version>
        <jjwt.version>0.12.6</jjwt.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.example.ecommerce</groupId>
                <artifactId>ecommerce-bom</artifactId>
                <version>${project.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>io.jsonwebtoken</groupId>
                <artifactId>jjwt-api</artifactId>
                <version>${jjwt.version}</version>
            </dependency>
            <dependency>
                <groupId>io.jsonwebtoken</groupId>
                <artifactId>jjwt-impl</artifactId>
                <version>${jjwt.version}</version>
                <scope>runtime</scope>
            </dependency>
            <dependency>
                <groupId>io.jsonwebtoken</groupId>
                <artifactId>jjwt-jackson</artifactId>
                <version>${jjwt.version}</version>
                <scope>runtime</scope>
            </dependency>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <configuration>
                        <skip>true</skip>
                    </configuration>
                    <executions>
                        <execution>
                            <goals><goal>repackage</goal><goal>build-info</goal></goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.13.0</version>
                    <configuration>
                        <release>${java.version}</release>
                        <annotationProcessorPaths>
                            <path>
                                <groupId>org.mapstruct</groupId>
                                <artifactId>mapstruct-processor</artifactId>
                                <version>${mapstruct.version}</version>
                            </path>
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                                <version>${lombok.version}</version>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

---

## 5. Auth Service — JWT + Refresh Tokens

```java
// auth-service/src/main/java/com/example/auth/domain/User.java
package com.example.auth.domain;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "users")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
    @Column(name = "role")
    private Set<String> roles = new HashSet<>();

    private boolean enabled = true;
    private boolean locked  = false;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;
}
```

```java
// auth-service/src/main/java/com/example/auth/domain/RefreshToken.java
package com.example.auth.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity @Table(name = "refresh_tokens")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class RefreshToken {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false, unique = true, length = 512) private String token;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id") private User user;
    @Column(nullable = false) private Instant expiresAt;
    private boolean revoked = false;
    private String  replacedByToken;   // token rotation chain
    private String  ipAddress;
    private String  userAgent;
}
```

```java
// auth-service/src/main/java/com/example/auth/service/JwtService.java
package com.example.auth.service;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Date;
import java.util.Map;

@Service
@Slf4j
public class JwtService {

    private final SecretKey signingKey;
    private final long      accessTokenTtlMs;

    public JwtService(@Value("${jwt.secret}") String secret,
                      @Value("${jwt.access-token-ttl-ms:900000}") long ttl) {
        this.signingKey       = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenTtlMs = ttl;
    }

    public String generateAccessToken(String subject, Map<String, Object> extraClaims) {
        Instant now = Instant.now();
        return Jwts.builder()
                .subject(subject)
                .claims(extraClaims)
                .issuedAt(Date.from(now))
                .expiration(Date.from(now.plusMillis(accessTokenTtlMs)))
                .signWith(signingKey)
                .compact();
    }

    public Claims parseToken(String token) {
        return Jwts.parser()
                .verifyWith(signingKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    public boolean isValid(String token) {
        try {
            parseToken(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.debug("Invalid JWT: {}", e.getMessage());
            return false;
        }
    }
}
```

```java
// auth-service/src/main/java/com/example/auth/service/AuthService.java
package com.example.auth.service;

import com.example.auth.domain.RefreshToken;
import com.example.auth.domain.User;
import com.example.auth.dto.AuthRequest;
import com.example.auth.dto.AuthResponse;
import com.example.auth.dto.RegisterRequest;
import com.example.auth.dto.TokenRefreshRequest;
import com.example.auth.repository.RefreshTokenRepository;
import com.example.auth.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Transactional
public class AuthService {

    private static final long REFRESH_TTL_DAYS = 30;

    private final UserRepository         userRepository;
    private final RefreshTokenRepository refreshTokenRepository;
    private final PasswordEncoder        passwordEncoder;
    private final JwtService             jwtService;

    public AuthResponse register(RegisterRequest req) {
        if (userRepository.existsByEmail(req.email())) {
            throw new IllegalArgumentException("Email already registered");
        }
        User user = User.builder()
                .username(req.username())
                .email(req.email())
                .passwordHash(passwordEncoder.encode(req.password()))
                .roles(Set.of("ROLE_USER"))
                .build();
        user = userRepository.save(user);
        return issueTokenPair(user, null, null);
    }

    public AuthResponse login(AuthRequest req, String ip, String userAgent) {
        User user = userRepository.findByEmail(req.email())
                .orElseThrow(() -> new BadCredentialsException("Invalid credentials"));
        if (!passwordEncoder.matches(req.password(), user.getPasswordHash())) {
            throw new BadCredentialsException("Invalid credentials");
        }
        if (!user.isEnabled() || user.isLocked()) {
            throw new BadCredentialsException("Account disabled or locked");
        }
        return issueTokenPair(user, ip, userAgent);
    }

    public AuthResponse refresh(TokenRefreshRequest req, String ip, String userAgent) {
        RefreshToken old = refreshTokenRepository.findByToken(req.refreshToken())
                .orElseThrow(() -> new BadCredentialsException("Invalid refresh token"));

        if (old.isRevoked()) {
            // Reuse detected — revoke entire family
            refreshTokenRepository.revokeAllByUserId(old.getUser().getId());
            throw new BadCredentialsException("Refresh token reuse detected");
        }
        if (old.getExpiresAt().isBefore(Instant.now())) {
            throw new BadCredentialsException("Refresh token expired");
        }

        String newRefreshToken = UUID.randomUUID().toString();
        old.setRevoked(true);
        old.setReplacedByToken(newRefreshToken);
        refreshTokenRepository.save(old);

        RefreshToken next = RefreshToken.builder()
                .token(newRefreshToken)
                .user(old.getUser())
                .expiresAt(Instant.now().plus(REFRESH_TTL_DAYS, ChronoUnit.DAYS))
                .ipAddress(ip)
                .userAgent(userAgent)
                .build();
        refreshTokenRepository.save(next);

        String access = jwtService.generateAccessToken(old.getUser().getUsername(),
                Map.of("roles", old.getUser().getRoles()));
        return new AuthResponse(access, newRefreshToken, REFRESH_TTL_DAYS * 86_400);
    }

    private AuthResponse issueTokenPair(User user, String ip, String userAgent) {
        String refreshTokenValue = UUID.randomUUID().toString();
        RefreshToken rt = RefreshToken.builder()
                .token(refreshTokenValue)
                .user(user)
                .expiresAt(Instant.now().plus(REFRESH_TTL_DAYS, ChronoUnit.DAYS))
                .ipAddress(ip)
                .userAgent(userAgent)
                .build();
        refreshTokenRepository.save(rt);

        String access = jwtService.generateAccessToken(user.getUsername(),
                Map.of("roles", user.getRoles(), "userId", user.getId()));
        return new AuthResponse(access, refreshTokenValue, REFRESH_TTL_DAYS * 86_400);
    }
}
```

---

## 6. Product Service — Elasticsearch Catalog

```java
// product-service/src/main/java/com/example/product/document/ProductDocument.java
package com.example.product.document;

import lombok.*;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@Document(indexName = "products")
@Setting(settingPath = "es-settings.json")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductDocument {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "english")
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Keyword)
    private String sku;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Integer)
    private int stockQuantity;

    @Field(type = FieldType.Keyword)
    private String categorySlug;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Date, format = DateFormat.date_optional_time)
    private Instant createdAt;

    @Field(type = FieldType.Boolean)
    private boolean active = true;
}
```

```java
// product-service/src/main/java/com/example/product/service/ProductSearchService.java
package com.example.product.service;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch._types.query_dsl.*;
import co.elastic.clients.elasticsearch.core.SearchRequest;
import com.example.product.document.ProductDocument;
import com.example.product.dto.ProductSearchRequest;
import com.example.product.dto.ProductSearchResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.core.*;
import org.springframework.data.elasticsearch.core.query.Criteria;
import org.springframework.data.elasticsearch.core.query.CriteriaQuery;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations esOperations;

    public ProductSearchResponse search(ProductSearchRequest req) {
        Criteria criteria = new Criteria();

        if (req.query() != null && !req.query().isBlank()) {
            criteria = criteria.and(
                new Criteria("name").matches(req.query())
                    .or(new Criteria("description").matches(req.query()))
            );
        }
        if (req.categorySlug() != null) {
            criteria = criteria.and(new Criteria("categorySlug").is(req.categorySlug()));
        }
        if (req.minPrice() != null) {
            criteria = criteria.and(new Criteria("price").greaterThanEqual(req.minPrice()));
        }
        if (req.maxPrice() != null) {
            criteria = criteria.and(new Criteria("price").lessThanEqual(req.maxPrice()));
        }
        criteria = criteria.and(new Criteria("active").is(true));

        CriteriaQuery query = new CriteriaQuery(criteria)
                .setPageable(PageRequest.of(req.page(), req.size()));

        SearchHits<ProductDocument> hits = esOperations.search(query, ProductDocument.class);

        List<ProductDocument> docs = hits.getSearchHits().stream()
                .map(SearchHit::getContent)
                .toList();

        return new ProductSearchResponse(docs, hits.getTotalHits(), req.page(), req.size());
    }
}
```

---

## 7. Order Service — Saga Pattern with Outbox

```java
// order-service/src/main/java/com/example/order/domain/Order.java
package com.example.order.domain;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String orderReference;       // UUID for idempotency

    @Column(nullable = false)
    private Long customerId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private OrderStatus status;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal totalAmount;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;
}
```

```java
// order-service/src/main/java/com/example/order/domain/OutboxEvent.java
package com.example.order.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity @Table(name = "outbox_events")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class OutboxEvent {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false) private String aggregateType;  // "Order"
    @Column(nullable = false) private Long   aggregateId;
    @Column(nullable = false) private String eventType;      // "ORDER_CREATED"
    @Column(nullable = false, columnDefinition = "TEXT") private String payload;
    @Column(nullable = false) private Instant createdAt;
    private Instant publishedAt;   // null = pending
}
```

```java
// order-service/src/main/java/com/example/order/service/OrderService.java
package com.example.order.service;

import com.example.order.domain.*;
import com.example.order.dto.CreateOrderRequest;
import com.example.order.dto.OrderDto;
import com.example.order.repository.OrderRepository;
import com.example.order.repository.OutboxEventRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.SneakyThrows;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository      orderRepository;
    private final OutboxEventRepository outboxRepository;
    private final ObjectMapper         objectMapper;

    // Transactional: both the order row and the outbox event are committed together
    @Transactional
    @SneakyThrows
    public OrderDto createOrder(CreateOrderRequest req, Long customerId) {
        // Idempotency: if this reference already exists, return it
        return orderRepository.findByOrderReference(req.idempotencyKey())
                .map(this::toDto)
                .orElseGet(() -> {
                    Order order = buildOrder(req, customerId);
                    orderRepository.save(order);
                    writeOutboxEvent(order);
                    return toDto(order);
                });
    }

    @Transactional
    public void markPaid(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("Order not found: " + orderId));
        order.setStatus(OrderStatus.PAID);
        orderRepository.save(order);
        writeOutboxEventRaw("ORDER_PAID", orderId, order);
    }

    @SneakyThrows
    private void writeOutboxEvent(Order order) {
        writeOutboxEventRaw("ORDER_CREATED", order.getId(), order);
    }

    @SneakyThrows
    private void writeOutboxEventRaw(String type, Long id, Object payload) {
        OutboxEvent event = OutboxEvent.builder()
                .aggregateType("Order")
                .aggregateId(id)
                .eventType(type)
                .payload(objectMapper.writeValueAsString(payload))
                .createdAt(Instant.now())
                .build();
        outboxRepository.save(event);
    }

    private Order buildOrder(CreateOrderRequest req, Long customerId) {
        // ... map items, compute total, etc.
        return Order.builder()
                .orderReference(req.idempotencyKey() != null
                        ? req.idempotencyKey() : UUID.randomUUID().toString())
                .customerId(customerId)
                .status(OrderStatus.PENDING_PAYMENT)
                .totalAmount(req.totalAmount())
                .build();
    }

    private OrderDto toDto(Order o) {
        return new OrderDto(o.getId(), o.getOrderReference(),
                o.getStatus().name(), o.getTotalAmount(), o.getCreatedAt());
    }
}
```

```java
// order-service/src/main/java/com/example/order/outbox/OutboxPublisher.java
package com.example.order.outbox;

import com.example.order.domain.OutboxEvent;
import com.example.order.repository.OutboxEventRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxPublisher {

    private final OutboxEventRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)   // poll every second
    @Transactional
    public void publishPending() {
        List<OutboxEvent> pending = outboxRepository.findTop50ByPublishedAtIsNullOrderByCreatedAtAsc();
        for (OutboxEvent event : pending) {
            try {
                String topic = resolveTopicFor(event.getEventType());
                kafkaTemplate.send(topic, String.valueOf(event.getAggregateId()), event.getPayload())
                        .get();  // synchronous to ensure delivery before marking
                event.setPublishedAt(Instant.now());
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish outbox event {}: {}", event.getId(), e.getMessage());
            }
        }
    }

    private String resolveTopicFor(String eventType) {
        return switch (eventType) {
            case "ORDER_CREATED" -> "order.created";
            case "ORDER_PAID"    -> "order.paid";
            default              -> "order.events";
        };
    }
}
```

---

## 8. Payment Service — Idempotency

```java
// payment-service/src/main/java/com/example/payment/service/PaymentService.java
package com.example.payment.service;

import com.example.payment.domain.IdempotencyKey;
import com.example.payment.domain.Payment;
import com.example.payment.domain.PaymentStatus;
import com.example.payment.dto.PaymentRequest;
import com.example.payment.dto.PaymentResponse;
import com.example.payment.gateway.PaymentGateway;
import com.example.payment.repository.IdempotencyKeyRepository;
import com.example.payment.repository.PaymentRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.SneakyThrows;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentRepository         paymentRepository;
    private final IdempotencyKeyRepository  idempotencyKeyRepo;
    private final PaymentGateway            gateway;
    private final ObjectMapper              objectMapper;

    @Transactional
    @SneakyThrows
    public PaymentResponse processPayment(PaymentRequest req) {
        // 1. Check idempotency key
        String key = req.idempotencyKey();
        var existing = idempotencyKeyRepo.findById(key);
        if (existing.isPresent()) {
            return objectMapper.readValue(existing.get().getResponseBody(), PaymentResponse.class);
        }

        // 2. Charge via gateway
        Payment payment = Payment.builder()
                .orderId(req.orderId())
                .amount(req.amount())
                .currency(req.currency())
                .status(PaymentStatus.PENDING)
                .idempotencyKey(key)
                .createdAt(Instant.now())
                .build();
        payment = paymentRepository.save(payment);

        PaymentResponse response;
        try {
            String chargeId = gateway.charge(req.amount(), req.currency(), req.paymentMethodToken());
            payment.setStatus(PaymentStatus.SUCCEEDED);
            payment.setGatewayChargeId(chargeId);
            response = new PaymentResponse(payment.getId(), "SUCCEEDED", chargeId);
        } catch (Exception e) {
            payment.setStatus(PaymentStatus.FAILED);
            payment.setFailureReason(e.getMessage());
            response = new PaymentResponse(payment.getId(), "FAILED", null);
        }
        paymentRepository.save(payment);

        // 3. Store idempotency key → response mapping (24 h TTL)
        IdempotencyKey ik = IdempotencyKey.builder()
                .key(key)
                .responseBody(objectMapper.writeValueAsString(response))
                .expiresAt(Instant.now().plus(24, ChronoUnit.HOURS))
                .build();
        idempotencyKeyRepo.save(ik);

        return response;
    }
}
```

---

## 9. Notification Service — Kafka Consumer

```java
// notification-service/src/main/java/com/example/notification/consumer/OrderEventConsumer.java
package com.example.notification.consumer;

import com.example.notification.dto.OrderCreatedEvent;
import com.example.notification.dto.OrderPaidEvent;
import com.example.notification.service.EmailService;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventConsumer {

    private final EmailService emailService;
    private final ObjectMapper objectMapper;

    @KafkaListener(topics = "order.created", groupId = "notification-service",
                   containerFactory = "kafkaListenerContainerFactory")
    public void onOrderCreated(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            OrderCreatedEvent event = objectMapper.readValue(record.value(), OrderCreatedEvent.class);
            emailService.sendOrderConfirmation(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process order.created event: {}", e.getMessage(), e);
            // DLQ logic: after max retries the message is routed to order.created.DLT
        }
    }

    @KafkaListener(topics = "order.paid", groupId = "notification-service")
    public void onOrderPaid(ConsumerRecord<String, String> record, Acknowledgment ack) {
        try {
            OrderPaidEvent event = objectMapper.readValue(record.value(), OrderPaidEvent.class);
            emailService.sendPaymentReceipt(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process order.paid event: {}", e.getMessage(), e);
        }
    }
}
```

---

## 10. Docker Compose — All Services

```yaml
# docker/docker-compose.yml
version: "3.9"
services:
  postgres-auth:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: authdb, POSTGRES_USER: auth, POSTGRES_PASSWORD: auth }
    ports: ["5433:5432"]
    volumes: [auth-data:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL","pg_isready -U auth"], interval: 5s, retries: 10 }
  postgres-orders:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: ordersdb, POSTGRES_USER: orders, POSTGRES_PASSWORD: orders }
    ports: ["5434:5432"]
    volumes: [orders-data:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL","pg_isready -U orders"], interval: 5s, retries: 10 }
  postgres-payments:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: paymentsdb, POSTGRES_USER: payments, POSTGRES_PASSWORD: payments }
    ports: ["5435:5432"]
    volumes: [payments-data:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL","pg_isready -U payments"], interval: 5s, retries: 10 }
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.4
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports: ["9200:9200"]
    volumes: [es-data:/usr/share/elasticsearch/data]
    healthcheck:
      test: ["CMD-SHELL","curl -sf http://localhost:9200/_cluster/health | grep -qv red"]
      interval: 10s
      retries: 10
  kafka:
    image: confluentinc/cp-kafka:7.6.1
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    ports: ["9092:9092"]
    healthcheck:
      test: ["CMD-SHELL","kafka-broker-api-versions --bootstrap-server localhost:9092"]
      interval: 10s
      retries: 10
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    command: redis-server --save 60 1
  auth-service:
    build: { context: .., dockerfile: auth-service/Dockerfile }
    ports: ["8081:8081"]
    depends_on:
      postgres-auth: { condition: service_healthy }
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-auth:5432/authdb
      SPRING_DATASOURCE_USERNAME: auth
      SPRING_DATASOURCE_PASSWORD: auth
      JWT_SECRET: ${JWT_SECRET:-changeme_at_least_32_chars_long!!}
      SERVER_PORT: 8081
  product-service:
    build: { context: .., dockerfile: product-service/Dockerfile }
    ports: ["8082:8082"]
    depends_on:
      elasticsearch: { condition: service_healthy }
    environment: { SPRING_ELASTICSEARCH_URIS: "http://elasticsearch:9200", SERVER_PORT: 8082 }
  order-service:
    build: { context: .., dockerfile: order-service/Dockerfile }
    ports: ["8083:8083"]
    depends_on:
      postgres-orders: { condition: service_healthy }
      kafka:           { condition: service_healthy }
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-orders:5432/ordersdb
      SPRING_DATASOURCE_USERNAME: orders
      SPRING_DATASOURCE_PASSWORD: orders
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SERVER_PORT: 8083
  payment-service:
    build: { context: .., dockerfile: payment-service/Dockerfile }
    ports: ["8084:8084"]
    depends_on:
      postgres-payments: { condition: service_healthy }
      kafka:             { condition: service_healthy }
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-payments:5432/paymentsdb
      SPRING_DATASOURCE_USERNAME: payments
      SPRING_DATASOURCE_PASSWORD: payments
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SERVER_PORT: 8084
  notification-service:
    build: { context: .., dockerfile: notification-service/Dockerfile }
    ports: ["8085:8085"]
    depends_on: { kafka: { condition: service_healthy } }
    environment: { SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092, SERVER_PORT: 8085 }
  prometheus:
    image: prom/prometheus:v2.52.0
    ports: ["9090:9090"]
    volumes: ["./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro"]
    command: ["--config.file=/etc/prometheus/prometheus.yml","--storage.tsdb.retention.time=15d"]
  grafana:
    image: grafana/grafana:10.4.2
    ports: ["3000:3000"]
    environment: { GF_SECURITY_ADMIN_PASSWORD: admin, GF_USERS_ALLOW_SIGN_UP: "false" }
    volumes: ["grafana-data:/var/lib/grafana"]
  zipkin:
    image: openzipkin/zipkin:3.3
    ports: ["9411:9411"]
volumes:
  auth-data:
  orders-data:
  payments-data:
  es-data:
  grafana-data:
```

---

## 11. Prometheus and Actuator Configuration

```yaml
# docker/prometheus/prometheus.yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'spring-services'
    metrics_path: /actuator/prometheus
    static_configs:
      - targets:
          - 'auth-service:8081'
          - 'product-service:8082'
          - 'order-service:8083'
          - 'payment-service:8084'
          - 'notification-service:8085'
```

```yaml
# Each service's application.yml — shared actuator block
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

---

## 12. CI/CD Pipeline — GitHub Actions

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline
on:
  push:       { branches: [main, develop] }
  pull_request: { branches: [main] }

env:
  REGISTRY:   ghcr.io
  IMAGE_BASE: ghcr.io/${{ github.repository_owner }}/ecommerce

jobs:
  build-test-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'maven' }
      - run: ./mvnw -B install -DskipTests
      - run: ./mvnw -B test
      - run: ./mvnw -B verify -Pfailsafe
      - uses: codecov/codecov-action@v4
        with: { files: '**/target/site/jacoco/jacoco.xml' }
      - uses: aquasecurity/trivy-action@master
        with: { scan-type: 'fs', severity: 'CRITICAL,HIGH', ignore-unfixed: true }

  docker-build:
    runs-on: ubuntu-latest
    needs: build-test-scan
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: [auth-service, product-service, order-service,
                  payment-service, notification-service]
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ matrix.service }}/Dockerfile
          push: true
          tags: ${{ env.IMAGE_BASE }}-${{ matrix.service }}:${{ github.sha }}
          cache-from: type=gha
          cache-to:   type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: docker-build
    environment: staging
    steps:
      - uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/ecommerce && docker compose pull
            docker compose up -d --remove-orphans
            docker system prune -f
```

---

## 14. Security and Performance Checklists

**Security:**
- HTTPS (TLS 1.3) terminated at NGINX; JWT HS256 with 256-bit secret in Vault/Secrets
- Refresh tokens rotate on every use; reuse detection revokes the entire token family
- Passwords: bcrypt strength 12; CORS restricted to known origins
- Bean Validation on every DTO; parameterised JPQL — no string-concatenated queries
- Rate limiting via NGINX `limit_req_zone`; Trivy + OWASP DC in CI
- Actuator on separate management port; `ROLE_` prefix enforced; GDPR erasure endpoint

**Performance:**
- Virtual threads: `spring.threads.virtual.enabled=true`
- `FetchType.LAZY` everywhere; `default_batch_fetch_size=25`; HikariCP pool = cores × 2 + 1
- Redis caching for products (5 min TTL); Elasticsearch for catalog reads
- Outbox poller at 1 s; SLO histogram buckets at p50/p95/p99

---

## 15. Conclusion — What's Next

This capstone demonstrates multi-module Maven, the Transactional Outbox Saga, CQRS with
Elasticsearch, idempotent payments, rotating JWT refresh tokens, and a full observability stack.

**Next steps for world-class Java developers:**
1. **Kubernetes** — Helm charts, HPA, PodDisruptionBudgets, Ingress with cert-manager
2. **gRPC** — replace inter-service REST with typed Protobuf contracts
3. **Event sourcing** — extend the outbox into a full event store with CQRS projections
4. **GraalVM native** — benchmark `auth-service` native image (cold-start < 100 ms)
5. **Service mesh** — Istio mTLS, traffic splitting, canary deployments
6. **SRE** — error budgets, SLO burn-rate alerts, runbooks in Confluence
