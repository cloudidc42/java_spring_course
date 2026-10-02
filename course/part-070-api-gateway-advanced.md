# Part 070: Advanced API Gateway Patterns

## Overview

An API gateway is more than a reverse proxy — it is the front door to your entire system. This
part turns Spring Cloud Gateway into a production-grade BFF (Backend for Frontend), aggregator,
traffic splitter, rate limiter, protocol translator, and circuit breaker, all with practical
runnable code.

---

## 1. Project Setup

### Maven dependencies (gateway service)

```xml
<dependencies>
    <!-- Spring Cloud Gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!-- Circuit breaker (Resilience4j) -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
    </dependency>

    <!-- Rate limiter (needs Redis) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>

    <!-- Service discovery -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <!-- Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <!-- gRPC (protocol translation) -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.60.1</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.60.1</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.3</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway

  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 100
            redis-rate-limiter.burstCapacity: 200
            key-resolver: "#{@userKeyResolver}"

      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER_URI:http://auth-server:9000}

resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5

services:
  product-service-url:   http://product-service
  order-service-url:     http://order-service
  user-service-url:      http://user-service
  inventory-service-url: http://inventory-service
  grpc-product-host:     product-service
  grpc-product-port:     9090
```

---

## 2. Basic Route Configuration

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;

@Configuration
public class GatewayRouteConfig {

    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        return builder.routes()

            // --- Product Service ---
            .route("product-service", r -> r
                .path("/api/v1/products/**")
                .filters(f -> f
                    .rewritePath("/api/v1/products/(?<segment>.*)",
                                 "/internal/products/${segment}")
                    .addRequestHeader("X-Gateway-Source", "api-gateway")
                    .addResponseHeader("X-Powered-By", "MyShop-Gateway")
                    .circuitBreaker(cb -> cb
                        .setName("product-cb")
                        .setFallbackUri("forward:/fallback/products"))
                )
                .uri("lb://product-service"))

            // --- Order Service ---
            .route("order-service", r -> r
                .path("/api/v1/orders/**")
                .and().method(HttpMethod.GET, HttpMethod.POST, HttpMethod.PUT, HttpMethod.DELETE)
                .filters(f -> f
                    .circuitBreaker(cb -> cb
                        .setName("order-cb")
                        .setFallbackUri("forward:/fallback/orders"))
                    .retry(config -> config
                        .setRetries(2)
                        .setMethods(HttpMethod.GET)
                        .setStatuses(
                            org.springframework.http.HttpStatus.BAD_GATEWAY,
                            org.springframework.http.HttpStatus.SERVICE_UNAVAILABLE))
                )
                .uri("lb://order-service"))

            // --- User Service ---
            .route("user-service", r -> r
                .path("/api/v1/users/**")
                .filters(f -> f
                    .circuitBreaker(cb -> cb
                        .setName("user-cb")
                        .setFallbackUri("forward:/fallback/users")))
                .uri("lb://user-service"))

            // --- Auth pass-through ---
            .route("auth-service", r -> r
                .path("/api/v1/auth/**")
                .uri("lb://auth-service"))

            .build();
    }
}
```

---

## 3. Custom Gateway Filters

### 3.1 Request correlation ID filter

```java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.UUID;

@Slf4j
@Component
public class CorrelationIdFilter implements GlobalFilter, Ordered {

    public static final String CORRELATION_HEADER = "X-Correlation-ID";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst(CORRELATION_HEADER);

        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        final String finalCorrelationId = correlationId;

        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header(CORRELATION_HEADER, finalCorrelationId)
            .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build())
            .then(Mono.fromRunnable(() ->
                exchange.getResponse().getHeaders()
                    .add(CORRELATION_HEADER, finalCorrelationId)
            ));
    }

    @Override
    public int getOrder() { return -100; }
}
```

### 3.2 Request/Response logging filter

```java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.time.Instant;

@Slf4j
@Component
public class AccessLogFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Instant start = Instant.now();
        String path   = exchange.getRequest().getPath().value();
        String method = exchange.getRequest().getMethod().name();
        String corrId = exchange.getRequest().getHeaders()
            .getFirst(CorrelationIdFilter.CORRELATION_HEADER);

        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                int status = exchange.getResponse().getStatusCode() != null
                    ? exchange.getResponse().getStatusCode().value() : 0;
                long ms = Duration.between(start, Instant.now()).toMillis();
                log.info("{} {} {} {}ms corrId={}", method, path, status, ms, corrId);
            }));
    }

    @Override
    public int getOrder() { return -90; }
}
```

### 3.3 JWT user-info extraction filter (inject user ID downstream)

```java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Slf4j
@Component
public class UserContextFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .flatMap(ctx -> {
                if (ctx.getAuthentication() instanceof JwtAuthenticationToken jwtAuth) {
                    Jwt jwt = (Jwt) jwtAuth.getPrincipal();
                    String userId = jwt.getSubject();
                    String roles  = String.join(",", jwtAuth.getAuthorities().stream()
                        .map(a -> a.getAuthority()).toList());

                    var request = exchange.getRequest().mutate()
                        .header("X-User-ID",    userId)
                        .header("X-User-Roles", roles)
                        .build();
                    return chain.filter(exchange.mutate().request(request).build());
                }
                return chain.filter(exchange);
            })
            .switchIfEmpty(chain.filter(exchange));
    }

    @Override
    public int getOrder() { return -80; }
}
```

### 3.4 Request transformation filter (add/modify body)

```java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.*;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.core.io.buffer.*;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;

/**
 * Adds a metadata field to JSON request bodies.
 * Usage in route: filters: - AddMetadata
 */
@Slf4j
@Component
public class AddMetadataFilterFactory
        extends AbstractGatewayFilterFactory<AddMetadataFilterFactory.Config> {

    public AddMetadataFilterFactory() { super(Config.class); }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            if (!isJsonRequest(exchange)) return chain.filter(exchange);

            return exchange.getRequest().getBody()
                .collectList()
                .flatMap(dataBuffers -> {
                    String originalBody = dataBuffers.stream()
                        .map(buf -> buf.toString(StandardCharsets.UTF_8))
                        .reduce("", String::concat);

                    String modified = injectMetadata(originalBody, exchange);

                    byte[] modifiedBytes = modified.getBytes(StandardCharsets.UTF_8);
                    DataBufferFactory factory = exchange.getResponse().bufferFactory();
                    DataBuffer buffer = factory.wrap(modifiedBytes);

                    ServerHttpRequest request = new ServerHttpRequestDecorator(exchange.getRequest()) {
                        @Override
                        public Flux<DataBuffer> getBody() { return Flux.just(buffer); }
                    };

                    return chain.filter(exchange.mutate().request(request).build());
                });
        };
    }

    private boolean isJsonRequest(ServerWebExchange exchange) {
        var contentType = exchange.getRequest().getHeaders().getContentType();
        return contentType != null &&
               contentType.toString().contains("application/json");
    }

    private String injectMetadata(String body, ServerWebExchange exchange) {
        if (!body.trim().startsWith("{")) return body;
        String gatewayMeta = "\"_gateway\":{\"source\":\"api-gateway\",\"timestamp\":"
                             + System.currentTimeMillis() + "},";
        return "{" + gatewayMeta + body.substring(1);
    }

    public static class Config {}
}
```

---

## 4. Gateway as BFF (Backend for Frontend)

### Mobile BFF — aggregates data for a single screen

```java
package com.example.gateway.bff;

import lombok.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.List;
import java.util.Map;

@Service
@RequiredArgsConstructor
public class MobileBffService {

    private final WebClient.Builder webClientBuilder;

    @Value("${services.product-service-url}")   private String productUrl;
    @Value("${services.order-service-url}")      private String orderUrl;
    @Value("${services.user-service-url}")       private String userUrl;
    @Value("${services.inventory-service-url}")  private String inventoryUrl;

    /**
     * Home screen aggregation: user profile + recent orders + featured products.
     * All 3 calls are fired in parallel.
     */
    public Mono<HomeScreenResponse> getHomeScreen(String userId, String authHeader) {
        WebClient productClient  = webClientBuilder.baseUrl(productUrl).build();
        WebClient orderClient    = webClientBuilder.baseUrl(orderUrl).build();
        WebClient userClient     = webClientBuilder.baseUrl(userUrl).build();

        Mono<UserProfileDto>       profileMono = userClient.get()
            .uri("/internal/users/{id}", userId)
            .header("Authorization", authHeader)
            .retrieve()
            .bodyToMono(UserProfileDto.class)
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(new UserProfileDto());

        Mono<List<OrderSummaryDto>> ordersMono = orderClient.get()
            .uri("/internal/orders?userId={id}&limit=5", userId)
            .header("Authorization", authHeader)
            .retrieve()
            .bodyToFlux(OrderSummaryDto.class)
            .collectList()
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(List.of());

        Mono<List<ProductSummaryDto>> featuredMono = productClient.get()
            .uri("/internal/products/featured?limit=10")
            .retrieve()
            .bodyToFlux(ProductSummaryDto.class)
            .collectList()
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(List.of());

        return Mono.zip(profileMono, ordersMono, featuredMono)
            .map(tuple -> HomeScreenResponse.builder()
                .user(tuple.getT1())
                .recentOrders(tuple.getT2())
                .featuredProducts(tuple.getT3())
                .build());
    }

    /**
     * Product detail screen: product + inventory + related products.
     */
    public Mono<ProductDetailResponse> getProductDetail(Long productId) {
        WebClient productClient   = webClientBuilder.baseUrl(productUrl).build();
        WebClient inventoryClient = webClientBuilder.baseUrl(inventoryUrl).build();

        Mono<ProductDto> productMono = productClient.get()
            .uri("/internal/products/{id}", productId)
            .retrieve()
            .bodyToMono(ProductDto.class)
            .timeout(Duration.ofSeconds(3));

        Mono<InventoryDto> inventoryMono = inventoryClient.get()
            .uri("/internal/inventory/{id}", productId)
            .retrieve()
            .bodyToMono(InventoryDto.class)
            .timeout(Duration.ofSeconds(2))
            .onErrorReturn(new InventoryDto(productId, 0, "UNKNOWN"));

        Mono<List<ProductSummaryDto>> relatedMono = productClient.get()
            .uri("/internal/products/{id}/related?limit=6", productId)
            .retrieve()
            .bodyToFlux(ProductSummaryDto.class)
            .collectList()
            .timeout(Duration.ofSeconds(2))
            .onErrorReturn(List.of());

        return Mono.zip(productMono, inventoryMono, relatedMono)
            .map(t -> ProductDetailResponse.builder()
                .product(t.getT1())
                .inventory(t.getT2())
                .relatedProducts(t.getT3())
                .build());
    }
}
```

### BFF DTOs

```java
package com.example.gateway.bff;

import lombok.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;

@Data @Builder @NoArgsConstructor @AllArgsConstructor
class HomeScreenResponse {
    private UserProfileDto user;
    private List<OrderSummaryDto> recentOrders;
    private List<ProductSummaryDto> featuredProducts;
}

@Data @NoArgsConstructor @AllArgsConstructor
class UserProfileDto {
    private String id;
    private String name;
    private String email;
    private String avatarUrl;
}

@Data @NoArgsConstructor @AllArgsConstructor
class OrderSummaryDto {
    private String id;
    private String orderNumber;
    private String status;
    private BigDecimal total;
    private Instant createdAt;
}

@Data @NoArgsConstructor @AllArgsConstructor
class ProductSummaryDto {
    private Long   id;
    private String name;
    private String thumbnailUrl;
    private BigDecimal price;
    private Double rating;
}

@Data @Builder @NoArgsConstructor @AllArgsConstructor
class ProductDetailResponse {
    private ProductDto product;
    private InventoryDto inventory;
    private List<ProductSummaryDto> relatedProducts;
}

@Data @NoArgsConstructor @AllArgsConstructor
class ProductDto {
    private Long   id;
    private String name;
    private String description;
    private BigDecimal price;
}

@Data @AllArgsConstructor @NoArgsConstructor
class InventoryDto {
    private Long productId;
    private int  stock;
    private String status; // IN_STOCK | LOW_STOCK | OUT_OF_STOCK | UNKNOWN
}
```

### BFF Controller

```java
package com.example.gateway.bff;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/bff")
@RequiredArgsConstructor
public class BffController {

    private final MobileBffService mobileBffService;

    @GetMapping("/mobile/home")
    public Mono<ResponseEntity<HomeScreenResponse>> mobileHome(
            @AuthenticationPrincipal Jwt jwt,
            @RequestHeader(value = "Authorization", required = false) String auth) {
        return mobileBffService.getHomeScreen(jwt.getSubject(), auth)
            .map(ResponseEntity::ok);
    }

    @GetMapping("/mobile/products/{id}")
    public Mono<ResponseEntity<ProductDetailResponse>> productDetail(
            @PathVariable Long id) {
        return mobileBffService.getProductDetail(id)
            .map(ResponseEntity::ok);
    }
}
```

---

## 5. Rate Limiting per User Tier

```java
package com.example.gateway.ratelimit;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.cloud.gateway.filter.ratelimit.RateLimiter;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Slf4j
@Component("userKeyResolver")
public class UserTierKeyResolver implements KeyResolver {

    @Override
    public Mono<String> resolve(org.springframework.web.server.ServerWebExchange exchange) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> {
                if (ctx.getAuthentication() instanceof JwtAuthenticationToken jwtAuth) {
                    Jwt jwt = (Jwt) jwtAuth.getPrincipal();
                    String tier = jwt.getClaimAsString("tier"); // "FREE" | "BASIC" | "PRO"
                    // Key = tier:userId so different tiers get separate buckets
                    return (tier == null ? "FREE" : tier.toUpperCase()) + ":" + jwt.getSubject();
                }
                return "ANONYMOUS:" + exchange.getRequest().getRemoteAddress();
            })
            .defaultIfEmpty("ANONYMOUS:" +
                exchange.getRequest().getRemoteAddress().getHostString());
    }
}
```

```java
package com.example.gateway.ratelimit;

import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RateLimiterConfig {

    /** Free tier: 10 req/s, burst of 20. */
    @Bean("freeRateLimiter")
    public RedisRateLimiter freeRateLimiter() {
        return new RedisRateLimiter(10, 20, 1);
    }

    /** Basic tier: 60 req/s, burst of 120. */
    @Bean("basicRateLimiter")
    public RedisRateLimiter basicRateLimiter() {
        return new RedisRateLimiter(60, 120, 1);
    }

    /** Pro tier: 300 req/s, burst of 600. */
    @Bean("proRateLimiter")
    public RedisRateLimiter proRateLimiter() {
        return new RedisRateLimiter(300, 600, 1);
    }
}
```

### Tier-aware routing in Java config

```java
package com.example.gateway.config;

import com.example.gateway.ratelimit.*;
import lombok.RequiredArgsConstructor;
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.oauth2.jwt.Jwt;
import reactor.core.publisher.Mono;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;

@Configuration
@RequiredArgsConstructor
public class TieredRateLimitRouteConfig {

    private final RedisRateLimiter freeRateLimiter;
    private final RedisRateLimiter basicRateLimiter;
    private final RedisRateLimiter proRateLimiter;
    private final UserTierKeyResolver keyResolver;

    @Bean
    public RouteLocator tieredRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // All search traffic with tier-based rate limiting
            .route("search-tier-free", r -> r
                .path("/api/v1/search/**")
                .and().header("X-User-Tier", "FREE")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(freeRateLimiter)
                        .setKeyResolver(keyResolver)))
                .uri("lb://search-service"))

            .route("search-tier-basic", r -> r
                .path("/api/v1/search/**")
                .and().header("X-User-Tier", "BASIC")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(basicRateLimiter)
                        .setKeyResolver(keyResolver)))
                .uri("lb://search-service"))

            .route("search-tier-pro", r -> r
                .path("/api/v1/search/**")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(proRateLimiter)
                        .setKeyResolver(keyResolver)))
                .uri("lb://search-service"))
            .build();
    }
}
```

---

## 6. Traffic Splitting — Canary and A/B Testing

```java
package com.example.gateway.trafficsplit;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

/**
 * Canary deployment: route X% of traffic to v2.
 *
 * Sets header X-Route-Target: canary or stable which downstream routes match.
 */
@Slf4j
@Component
public class CanaryRoutingFilter implements GlobalFilter, Ordered {

    private static final double CANARY_PERCENTAGE = 0.10; // 10%

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Stable traffic already has a cookie → keep them on same version
        String existingVariant = getCookieValue(exchange, "variant");
        if (existingVariant != null) {
            return chain.filter(taggedExchange(exchange, existingVariant));
        }

        String variant = Math.random() < CANARY_PERCENTAGE ? "canary" : "stable";
        var mutated = taggedExchange(exchange, variant);

        // Set cookie so user stays on same variant
        mutated.getResponse().addCookie(
            org.springframework.http.ResponseCookie
                .from("variant", variant)
                .path("/")
                .maxAge(java.time.Duration.ofHours(1))
                .build()
        );

        return chain.filter(mutated);
    }

    private ServerWebExchange taggedExchange(ServerWebExchange exchange, String variant) {
        var request = exchange.getRequest().mutate()
            .header("X-Route-Target", variant)
            .build();
        return exchange.mutate().request(request).build();
    }

    private String getCookieValue(ServerWebExchange exchange, String name) {
        var cookie = exchange.getRequest().getCookies().getFirst(name);
        return cookie != null ? cookie.getValue() : null;
    }

    @Override
    public int getOrder() { return -70; }
}
```

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CanaryRouteConfig {

    @Bean
    public RouteLocator canaryRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // Canary version (new code under test)
            .route("product-v2-canary", r -> r
                .path("/api/v1/products/**")
                .and().header("X-Route-Target", "canary")
                .filters(f -> f.addResponseHeader("X-Version", "v2"))
                .uri("lb://product-service-v2"))

            // Stable version (current production)
            .route("product-v1-stable", r -> r
                .path("/api/v1/products/**")
                .filters(f -> f.addResponseHeader("X-Version", "v1"))
                .uri("lb://product-service"))
            .build();
    }
}
```

### A/B testing by user ID hash

```java
package com.example.gateway.trafficsplit;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.security.core.context.ReactiveSecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

/**
 * Deterministic A/B split by user ID.
 * Users consistently land in group A or B across sessions.
 */
@Component
public class AbTestingFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .flatMap(ctx -> {
                String group = "A"; // default
                if (ctx.getAuthentication() instanceof JwtAuthenticationToken jwtAuth) {
                    Jwt jwt = (Jwt) jwtAuth.getPrincipal();
                    int hash = Math.abs(jwt.getSubject().hashCode());
                    group = (hash % 2 == 0) ? "A" : "B";
                }
                var req = exchange.getRequest().mutate()
                    .header("X-AB-Group", group)
                    .build();
                return chain.filter(exchange.mutate().request(req).build());
            })
            .switchIfEmpty(chain.filter(exchange));
    }

    @Override
    public int getOrder() { return -65; }
}
```

---

## 7. Circuit Breaker at Gateway Level

```java
package com.example.gateway.config;

import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.timelimiter.TimeLimiterConfig;
import org.springframework.cloud.circuitbreaker.resilience4j.ReactiveResilience4JCircuitBreakerFactory;
import org.springframework.cloud.circuitbreaker.resilience4j.Resilience4JConfigBuilder;
import org.springframework.cloud.client.circuitbreaker.Customizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class CircuitBreakerConfig2 {

    @Bean
    public Customizer<ReactiveResilience4JCircuitBreakerFactory> circuitBreakerCustomizer() {
        return factory -> {
            factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(5))
                    .build())
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(10)
                    .failureRateThreshold(50)
                    .waitDurationInOpenState(Duration.ofSeconds(10))
                    .permittedNumberOfCallsInHalfOpenState(5)
                    .build())
                .build());

            // More permissive config for non-critical services
            factory.configure(builder -> builder
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(2))
                    .build())
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(20)
                    .failureRateThreshold(70)
                    .build())
                .build(), "recommendation-cb");
        };
    }
}
```

### Fallback controller

```java
package com.example.gateway.fallback;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @GetMapping("/products")
    public Mono<ResponseEntity<Map<String, Object>>> productsFallback() {
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "status", 503,
                "message", "Product service is temporarily unavailable. Please try again later.",
                "timestamp", Instant.now().toString(),
                "data", List.of()
            )));
    }

    @GetMapping("/orders")
    public Mono<ResponseEntity<Map<String, Object>>> ordersFallback() {
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "status", 503,
                "message", "Order service is temporarily unavailable.",
                "timestamp", Instant.now().toString()
            )));
    }

    @GetMapping("/users")
    public Mono<ResponseEntity<Map<String, Object>>> usersFallback() {
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "status", 503,
                "message", "User service is temporarily unavailable.",
                "timestamp", Instant.now().toString()
            )));
    }
}
```

---

## 8. Protocol Translation — REST to gRPC

```java
package com.example.gateway.grpc;

import io.grpc.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Slf4j
@Component
public class GrpcChannelManager {

    @Value("${services.grpc-product-host}")
    private String productHost;

    @Value("${services.grpc-product-port}")
    private int productPort;

    private ManagedChannel productChannel;

    @PostConstruct
    public void init() {
        productChannel = ManagedChannelBuilder
            .forAddress(productHost, productPort)
            .usePlaintext()
            .enableRetry()
            .maxRetryAttempts(3)
            .build();
        log.info("gRPC channel to product-service initialized");
    }

    @PreDestroy
    public void shutdown() {
        if (productChannel != null) productChannel.shutdownNow();
    }

    public ManagedChannel getProductChannel() { return productChannel; }
}
```

```java
package com.example.gateway.grpc;

import com.example.grpc.ProductServiceGrpc;
import com.example.grpc.ProductProto.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.util.concurrent.TimeUnit;

/**
 * Translates REST calls to gRPC calls on the product service.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class RestToGrpcBridge {

    private final GrpcChannelManager channelManager;

    public Mono<ProductResponse> getProduct(Long productId) {
        return Mono.fromCallable(() -> {
            ProductServiceGrpc.ProductServiceBlockingStub stub =
                ProductServiceGrpc.newBlockingStub(channelManager.getProductChannel())
                    .withDeadlineAfter(3, TimeUnit.SECONDS);

            GetProductRequest request = GetProductRequest.newBuilder()
                .setProductId(productId)
                .build();

            return stub.getProduct(request);
        }).subscribeOn(Schedulers.boundedElastic());
    }

    public Mono<SearchProductsResponse> searchProducts(String keyword, int page, int size) {
        return Mono.fromCallable(() -> {
            ProductServiceGrpc.ProductServiceBlockingStub stub =
                ProductServiceGrpc.newBlockingStub(channelManager.getProductChannel())
                    .withDeadlineAfter(5, TimeUnit.SECONDS);

            SearchProductsRequest request = SearchProductsRequest.newBuilder()
                .setKeyword(keyword)
                .setPage(page)
                .setSize(size)
                .build();

            return stub.searchProducts(request);
        }).subscribeOn(Schedulers.boundedElastic());
    }
}
```

### REST controller that uses the bridge

```java
package com.example.gateway.grpc;

import com.example.grpc.ProductProto.*;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/api/v1/grpc/products")
@RequiredArgsConstructor
public class ProductGrpcController {

    private final RestToGrpcBridge bridge;

    @GetMapping("/{id}")
    public Mono<ResponseEntity<ProductResponse>> getProduct(@PathVariable Long id) {
        return bridge.getProduct(id)
            .map(ResponseEntity::ok)
            .onErrorReturn(ResponseEntity.internalServerError().build());
    }

    @GetMapping("/search")
    public Mono<ResponseEntity<SearchProductsResponse>> search(
            @RequestParam String q,
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "20") int size) {
        return bridge.searchProducts(q, page, size)
            .map(ResponseEntity::ok)
            .onErrorReturn(ResponseEntity.internalServerError().build());
    }
}
```

---

## 9. Gateway Caching

```java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import org.springframework.http.*;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.http.server.reactive.ServerHttpResponseDecorator;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Set;

/**
 * Simple response caching for GET requests to cacheable paths.
 * Uses Redis as shared cache so all gateway instances share it.
 */
@Slf4j
@Component
public class ResponseCacheFilter implements GlobalFilter, Ordered {

    private static final Set<String> CACHEABLE_PREFIXES = Set.of(
        "/api/v1/products/",
        "/api/v1/categories"
    );
    private static final Duration CACHE_TTL = Duration.ofMinutes(5);

    private final ReactiveStringRedisTemplate redisTemplate;

    public ResponseCacheFilter(ReactiveStringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        if (!isGetRequest(exchange) || !isCacheable(exchange)) {
            return chain.filter(exchange);
        }

        String cacheKey = "gateway:cache:" + exchange.getRequest().getURI().toString();

        return redisTemplate.opsForValue().get(cacheKey)
            .flatMap(cached -> {
                log.debug("Gateway cache HIT: {}", cacheKey);
                ServerHttpResponse response = exchange.getResponse();
                response.setStatusCode(HttpStatus.OK);
                response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
                response.getHeaders().add("X-Cache", "HIT");
                byte[] bytes = cached.getBytes(StandardCharsets.UTF_8);
                DataBuffer buffer = response.bufferFactory().wrap(bytes);
                return response.writeWith(Mono.just(buffer));
            })
            .switchIfEmpty(Mono.defer(() -> {
                log.debug("Gateway cache MISS: {}", cacheKey);
                return captureAndCacheResponse(exchange, chain, cacheKey);
            }));
    }

    private Mono<Void> captureAndCacheResponse(ServerWebExchange exchange,
                                                GatewayFilterChain chain,
                                                String cacheKey) {
        StringBuilder bodyHolder = new StringBuilder();
        ServerHttpResponse original = exchange.getResponse();

        ServerHttpResponseDecorator decorator = new ServerHttpResponseDecorator(original) {
            @Override
            public Mono<Void> writeWith(org.reactivestreams.Publisher<? extends DataBuffer> body) {
                return super.writeWith(Flux.from(body).doOnNext(buf -> {
                    if (getStatusCode() == HttpStatus.OK) {
                        bodyHolder.append(buf.toString(StandardCharsets.UTF_8));
                    }
                })).then(Mono.defer(() -> {
                    if (!bodyHolder.isEmpty()) {
                        return redisTemplate.opsForValue()
                            .set(cacheKey, bodyHolder.toString(), CACHE_TTL)
                            .then();
                    }
                    return Mono.empty();
                }));
            }
        };

        return chain.filter(exchange.mutate().response(decorator).build());
    }

    private boolean isGetRequest(ServerWebExchange e) {
        return HttpMethod.GET.equals(e.getRequest().getMethod());
    }

    private boolean isCacheable(ServerWebExchange e) {
        String path = e.getRequest().getPath().value();
        return CACHEABLE_PREFIXES.stream().anyMatch(path::startsWith);
    }

    @Override
    public int getOrder() { return -60; }
}
```

---

## 10. Request Validation at Gateway

```java
package com.example.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;

/**
 * Rejects obviously malformed requests before they reach downstream services.
 */
@Slf4j
@Component
public class RequestValidationFilter implements GlobalFilter, Ordered {

    private static final int MAX_PATH_LENGTH  = 512;
    private static final int MAX_QUERY_LENGTH = 1024;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        var request = exchange.getRequest();
        String path  = request.getPath().value();
        String query = request.getURI().getQuery();

        // Reject oversized paths
        if (path.length() > MAX_PATH_LENGTH) {
            return reject(exchange, "Request path too long");
        }

        // Reject oversized query strings
        if (query != null && query.length() > MAX_QUERY_LENGTH) {
            return reject(exchange, "Query string too long");
        }

        // Basic path traversal check
        if (path.contains("..") || path.contains("%2e%2e")) {
            return reject(exchange, "Invalid path");
        }

        // Required API version header for non-auth paths
        if (!path.startsWith("/api/v1/auth") && !path.startsWith("/bff") &&
            !path.startsWith("/fallback") && !path.startsWith("/actuator")) {
            String version = request.getHeaders().getFirst("X-API-Version");
            if (StringUtils.hasText(version) && !"1".equals(version) && !"2".equals(version)) {
                return reject(exchange, "Unsupported API version: " + version);
            }
        }

        return chain.filter(exchange);
    }

    private Mono<Void> reject(ServerWebExchange exchange, String reason) {
        log.warn("Request rejected: {} path={}", reason,
                 exchange.getRequest().getPath().value());
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.BAD_REQUEST);
        byte[] body = ("{\"error\":\"" + reason + "\"}").getBytes(StandardCharsets.UTF_8);
        return response.writeWith(Mono.just(response.bufferFactory().wrap(body)));
    }

    @Override
    public int getOrder() { return -110; }  // Before all others
}
```

---

## 11. Security Configuration

```java
package com.example.gateway.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain securityFilterChain(ServerHttpSecurity http) {
        return http
            .csrf(ServerHttpSecurity.CsrfSpec::disable)
            .authorizeExchange(exchanges -> exchanges
                // Public endpoints
                .pathMatchers(
                    "/api/v1/auth/**",
                    "/api/v1/products/**",       // GET only — handled below for writes
                    "/api/v1/categories/**",
                    "/fallback/**",
                    "/actuator/health"
                ).permitAll()

                // BFF endpoints require authentication
                .pathMatchers("/bff/**").authenticated()

                // Admin operations
                .pathMatchers("/api/v1/admin/**")
                    .hasRole("ADMIN")

                // Everything else requires auth
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> {}))
            .build();
    }
}
```

---

## 12. Real Example — BFF for Mobile and Web Clients

### Web BFF — returns full data for desktop

```java
package com.example.gateway.bff;

import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.List;

@Service
@RequiredArgsConstructor
public class WebBffService {

    private final WebClient.Builder builder;

    @Value("${services.product-service-url}")  private String productUrl;
    @Value("${services.order-service-url}")     private String orderUrl;
    @Value("${services.user-service-url}")      private String userUrl;

    /**
     * Product catalog page for web: full product list + category tree + brand list.
     */
    public Mono<CatalogPageResponse> getCatalogPage(Long categoryId, String auth) {
        WebClient product = builder.baseUrl(productUrl).build();

        Mono<List<ProductSummaryDto>> productsMono = product.get()
            .uri("/internal/products?categoryId={id}&limit=40", categoryId)
            .header("Authorization", auth)
            .retrieve()
            .bodyToFlux(ProductSummaryDto.class)
            .collectList()
            .timeout(Duration.ofSeconds(5))
            .onErrorReturn(List.of());

        Mono<List<CategoryDto>> categoriesMono = product.get()
            .uri("/internal/categories/tree")
            .retrieve()
            .bodyToFlux(CategoryDto.class)
            .collectList()
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(List.of());

        Mono<List<BrandDto>> brandsMono = product.get()
            .uri("/internal/brands?categoryId={id}", categoryId)
            .retrieve()
            .bodyToFlux(BrandDto.class)
            .collectList()
            .timeout(Duration.ofSeconds(3))
            .onErrorReturn(List.of());

        return Mono.zip(productsMono, categoriesMono, brandsMono)
            .map(t -> CatalogPageResponse.builder()
                .products(t.getT1())
                .categoryTree(t.getT2())
                .brands(t.getT3())
                .build());
    }

    @lombok.Data @lombok.Builder @lombok.NoArgsConstructor @lombok.AllArgsConstructor
    public static class CatalogPageResponse {
        private List<ProductSummaryDto> products;
        private List<CategoryDto>       categoryTree;
        private List<BrandDto>          brands;
    }

    @lombok.Data @lombok.NoArgsConstructor @lombok.AllArgsConstructor
    public static class CategoryDto { private Long id; private String name; private Long parentId; }

    @lombok.Data @lombok.NoArgsConstructor @lombok.AllArgsConstructor
    public static class BrandDto { private Long id; private String name; private String logoUrl; }
}
```

```java
package com.example.gateway.bff;

import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/bff/web")
@RequiredArgsConstructor
public class WebBffController {

    private final WebBffService webBffService;

    @GetMapping("/catalog")
    public Mono<ResponseEntity<WebBffService.CatalogPageResponse>> catalog(
            @RequestParam(required = false) Long categoryId,
            @RequestHeader(value = "Authorization", required = false) String auth) {
        return webBffService.getCatalogPage(categoryId, auth)
            .map(ResponseEntity::ok);
    }
}
```

---

## 13. Gateway Actuator and Monitoring

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, gateway, prometheus
  endpoint:
    health:
      show-details: when-authorized
    gateway:
      enabled: true
```

```java
package com.example.gateway.monitor;

import lombok.RequiredArgsConstructor;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.util.List;

@RestController
@RequestMapping("/admin/gateway")
@RequiredArgsConstructor
public class GatewayAdminController {

    private final RouteLocator routeLocator;

    @GetMapping("/routes")
    public Mono<ResponseEntity<List<String>>> listRoutes() {
        return routeLocator.getRoutes()
            .map(r -> r.getId() + " → " + r.getUri())
            .collectList()
            .map(ResponseEntity::ok);
    }
}
```

---

## Summary Table

| Pattern | Spring Component | Class |
|---|---|---|
| Route config | `RouteLocatorBuilder` | `GatewayRouteConfig` |
| Correlation ID | `GlobalFilter` | `CorrelationIdFilter` |
| Access logging | `GlobalFilter` | `AccessLogFilter` |
| JWT user injection | `GlobalFilter` | `UserContextFilter` |
| Request body rewrite | `AbstractGatewayFilterFactory` | `AddMetadataFilterFactory` |
| BFF aggregation | `WebClient.zip()` | `MobileBffService` / `WebBffService` |
| Tier rate limiting | `RedisRateLimiter` + `KeyResolver` | `UserTierKeyResolver` |
| Canary routing | `GlobalFilter` + route predicate | `CanaryRoutingFilter` |
| A/B by user ID | `GlobalFilter` | `AbTestingFilter` |
| Circuit breaker | `spring-cloud-gateway` + Resilience4j | `CircuitBreakerConfig2` |
| Fallback responses | `@RestController` | `FallbackController` |
| REST→gRPC bridge | gRPC blocking stub | `RestToGrpcBridge` |
| Response cache | `GlobalFilter` + Redis | `ResponseCacheFilter` |
| Request validation | `GlobalFilter` | `RequestValidationFilter` |
| Security | `ServerHttpSecurity` reactive | `SecurityConfig` |

---

## Next Part Preview

**Part 071: Event-Driven Architecture with Apache Kafka** — design event-driven microservices with
Kafka producers, consumers, consumer groups, partitioning strategies, exactly-once semantics,
schema registry with Avro, dead-letter queues, and a complete order processing saga using the
transactional outbox pattern.
