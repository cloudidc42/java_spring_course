# Part 070 – Spring Cloud Gateway Advanced

## Dependencies (pom.xml)

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
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

---

## 1. Route Configuration via YAML

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
        - name: RequestRateLimiter
          args:
            redis-rate-limiter.replenishRate: 10
            redis-rate-limiter.burstCapacity: 20
            redis-rate-limiter.requestedTokens: 1
            key-resolver: "#{@ipKeyResolver}"
      routes:
        # User service route with Path predicate
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users(?<segment>/?.*), /users${segment}
            - AddRequestHeader=X-Gateway-Source, api-gateway
            - AddResponseHeader=X-Response-Time, "#{T(java.time.Instant).now()}"
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users

        # Order service route with multiple predicates
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST
            - Header=X-Request-Id, \d+
          filters:
            - RewritePath=/api/orders(?<segment>/?.*), /orders${segment}
            - name: Retry
              args:
                retries: 3
                statuses: SERVICE_UNAVAILABLE, INTERNAL_SERVER_ERROR
                methods: GET
                backoff:
                  firstBackoff: 100ms
                  maxBackoff: 500ms
                  factor: 2

        # Product service with host predicate
        - id: product-service-v1
          uri: lb://product-service
          predicates:
            - Host=api.example.com
            - Path=/products/**
          filters:
            - AddRequestHeader=X-API-Version, v1

        # Admin routes with cookie-based predicate
        - id: admin-service
          uri: lb://admin-service
          predicates:
            - Path=/admin/**
            - Cookie=session, [a-f0-9]{32}
          filters:
            - AddRequestHeader=X-Admin-Request, true

        # Query parameter predicate
        - id: search-service
          uri: lb://search-service
          predicates:
            - Path=/search
            - Query=q
          filters:
            - AddRequestHeader=X-Search-Request, true

  data:
    redis:
      host: localhost
      port: 6379

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,gateway
  endpoint:
    gateway:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}

resilience4j:
  circuitbreaker:
    instances:
      userServiceCB:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
      orderServiceCB:
        slidingWindowSize: 5
        failureRateThreshold: 60
        waitDurationInOpenState: 15s
```

---

## 2. Route Configuration via Java Code (RouteLocator)

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;

import java.time.Duration;

@Configuration
public class GatewayRoutesConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()

            // Payment service with strict predicates and circuit breaker
            .route("payment-service", r -> r
                .path("/api/payments/**")
                .and()
                .method(HttpMethod.POST, HttpMethod.GET)
                .and()
                .header("Content-Type", "application/json")
                .filters(f -> f
                    .rewritePath("/api/payments(?<segment>/?.*)", "/payments${segment}")
                    .addRequestHeader("X-Service-Name", "payment-service")
                    .addRequestHeader("X-Gateway-Timestamp",
                            String.valueOf(System.currentTimeMillis()))
                    .circuitBreaker(c -> c
                        .setName("paymentCB")
                        .setFallbackUri("forward:/fallback/payment"))
                    .retry(config -> config
                        .setRetries(2)
                        .setStatuses(HttpStatus.SERVICE_UNAVAILABLE)
                        .setMethods(HttpMethod.GET)
                        .setBackoff(Duration.ofMillis(100), Duration.ofMillis(1000), 2, false))
                )
                .uri("lb://payment-service"))

            // Inventory service with rate limiting per user
            .route("inventory-service", r -> r
                .path("/api/inventory/**")
                .filters(f -> f
                    .rewritePath("/api/inventory(?<segment>/?.*)", "/inventory${segment}")
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                    .addResponseHeader("X-Cache-Control", "no-store")
                )
                .uri("lb://inventory-service"))

            // Legacy service: strip prefix and map to v2
            .route("legacy-catalog", r -> r
                .path("/catalog/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .prefixPath("/api/v2")
                    .setStatus(HttpStatus.OK)
                    .addRequestHeader("X-Forwarded-By", "gateway")
                )
                .uri("lb://catalog-service"))

            // Redirect from old path
            .route("redirect-old-api", r -> r
                .path("/v1/**")
                .filters(f -> f.redirect(301, "http://api.example.com/v2/"))
                .uri("http://api.example.com"))

            .build();
    }

    @Bean
    public org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter redisRateLimiter() {
        return new org.springframework.cloud.gateway.filter.ratelimit.RedisRateLimiter(10, 20, 1);
    }

    @Bean
    public org.springframework.cloud.gateway.filter.ratelimit.KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            if (userId != null) {
                return reactor.core.publisher.Mono.just(userId);
            }
            String ip = exchange.getRequest().getRemoteAddress() != null
                    ? exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                    : "unknown";
            return reactor.core.publisher.Mono.just(ip);
        };
    }
}
```

---

## 3. IP-Based Key Resolver for Rate Limiting

```java
package com.example.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

@Configuration
public class RateLimitConfig {

    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            String xForwardedFor = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
            if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
                return Mono.just(xForwardedFor.split(",")[0].trim());
            }
            if (exchange.getRequest().getRemoteAddress() != null) {
                return Mono.just(exchange.getRequest().getRemoteAddress()
                        .getAddress().getHostAddress());
            }
            return Mono.just("unknown");
        };
    }

    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> {
            String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
            if (apiKey != null && !apiKey.isEmpty()) {
                return Mono.just("api-key:" + apiKey);
            }
            return Mono.just("anonymous");
        };
    }

    @Bean
    public KeyResolver userIdKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            return Mono.just(userId != null ? "user:" + userId : "anonymous");
        };
    }
}
```

---

## 4. Custom Global Filter – Logging and Tracing

```java
package com.example.gateway.filter;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.UUID;

@Component
public class RequestLoggingGlobalFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingGlobalFilter.class);
    private static final String REQUEST_ID_HEADER = "X-Request-Id";
    private static final String START_TIME_ATTR = "startTime";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // Inject request ID if not present
        String requestId = request.getHeaders().getFirst(REQUEST_ID_HEADER);
        if (requestId == null || requestId.isEmpty()) {
            requestId = UUID.randomUUID().toString();
            ServerHttpRequest mutatedRequest = request.mutate()
                    .header(REQUEST_ID_HEADER, requestId)
                    .build();
            exchange = exchange.mutate().request(mutatedRequest).build();
        }

        final String finalRequestId = requestId;
        exchange.getAttributes().put(START_TIME_ATTR, Instant.now().toEpochMilli());

        log.info("Incoming request: id={} method={} path={} remoteAddr={}",
                finalRequestId,
                request.getMethod(),
                request.getURI().getPath(),
                request.getRemoteAddress());

        ServerWebExchange finalExchange = exchange;
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            ServerHttpResponse response = finalExchange.getResponse();
            Long startTime = finalExchange.getAttribute(START_TIME_ATTR);
            long duration = startTime != null ? Instant.now().toEpochMilli() - startTime : -1;

            log.info("Response: id={} status={} duration={}ms",
                    finalRequestId,
                    response.getStatusCode(),
                    duration);

            // Add response headers for tracing
            response.getHeaders().add("X-Request-Id", finalRequestId);
            response.getHeaders().add("X-Response-Time", duration + "ms");
        }));
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

---

## 5. Custom Global Filter – JWT Authentication

```java
package com.example.gateway.filter;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.security.Key;
import java.util.Arrays;
import java.util.List;

@Component
public class JwtAuthenticationGlobalFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(JwtAuthenticationGlobalFilter.class);
    private final AntPathMatcher pathMatcher = new AntPathMatcher();

    @Value("${jwt.secret:mySecretKey1234567890abcdefghijklmnop}")
    private String secretKey;

    private final List<String> excludedPaths = Arrays.asList(
            "/api/auth/**",
            "/actuator/**",
            "/fallback/**",
            "/api/public/**"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // Skip authentication for excluded paths
        boolean isExcluded = excludedPaths.stream()
                .anyMatch(p -> pathMatcher.match(p, path));
        if (isExcluded) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorizedResponse(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);
        try {
            Claims claims = parseToken(token);
            String userId = claims.getSubject();
            String roles = claims.get("roles", String.class);

            // Forward user info to downstream services
            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                    .header("X-User-Id", userId)
                    .header("X-User-Roles", roles != null ? roles : "")
                    .header("X-Token-Expiry", String.valueOf(claims.getExpiration().getTime()))
                    .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());
        } catch (JwtException e) {
            log.warn("Invalid JWT token for path {}: {}", path, e.getMessage());
            return unauthorizedResponse(exchange, "Invalid or expired token");
        }
    }

    private Claims parseToken(String token) {
        Key key = Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
        return Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    private Mono<Void> unauthorizedResponse(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().add("Content-Type", "application/json");
        byte[] bytes = ("{\"error\":\"" + message + "\"}").getBytes(StandardCharsets.UTF_8);
        org.springframework.core.io.buffer.DataBuffer buffer =
                response.bufferFactory().wrap(bytes);
        return response.writeWith(Mono.just(buffer));
    }

    @Override
    public int getOrder() {
        return -100; // Run after logging filter but before routing
    }
}
```

---

## 6. Custom GatewayFilter Factory – Request Validation

```java
package com.example.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.Arrays;
import java.util.List;

@Component
public class ApiKeyValidationGatewayFilterFactory
        extends AbstractGatewayFilterFactory<ApiKeyValidationGatewayFilterFactory.Config> {

    public ApiKeyValidationGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");

            if (apiKey == null || !config.getValidKeys().contains(apiKey)) {
                exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                return exchange.getResponse().setComplete();
            }

            // Append API key owner info
            return chain.filter(exchange.mutate()
                    .request(exchange.getRequest().mutate()
                            .header("X-API-Key-Owner", config.getKeyOwner(apiKey))
                            .build())
                    .build());
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("keys");
    }

    public static class Config {
        private List<String> validKeys;
        private java.util.Map<String, String> keyOwnerMap = new java.util.HashMap<>();

        public List<String> getValidKeys() { return validKeys; }
        public void setValidKeys(List<String> validKeys) { this.validKeys = validKeys; }
        public void setKeys(String keys) { this.validKeys = Arrays.asList(keys.split(",")); }

        public String getKeyOwner(String key) {
            return keyOwnerMap.getOrDefault(key, "unknown");
        }
    }
}
```

---

## 7. Custom GatewayFilter – Request Body Modification

```java
package com.example.gateway.filter;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.core.io.buffer.DataBufferUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpRequestDecorator;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.util.Map;

@Component
public class RequestBodyEnrichmentFilterFactory
        extends AbstractGatewayFilterFactory<RequestBodyEnrichmentFilterFactory.Config> {

    private final ObjectMapper objectMapper;

    public RequestBodyEnrichmentFilterFactory(ObjectMapper objectMapper) {
        super(Config.class);
        this.objectMapper = objectMapper;
    }

    @Override
    @SuppressWarnings("unchecked")
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            if (exchange.getRequest().getMethod() != HttpMethod.POST
                    && exchange.getRequest().getMethod() != HttpMethod.PUT) {
                return chain.filter(exchange);
            }

            return DataBufferUtils.join(exchange.getRequest().getBody())
                    .flatMap(dataBuffer -> {
                        byte[] bytes = new byte[dataBuffer.readableByteCount()];
                        dataBuffer.read(bytes);
                        DataBufferUtils.release(dataBuffer);

                        try {
                            String body = new String(bytes, StandardCharsets.UTF_8);
                            Map<String, Object> bodyMap = objectMapper.readValue(body, Map.class);

                            // Enrich with gateway metadata
                            bodyMap.put("_gateway_timestamp", System.currentTimeMillis());
                            bodyMap.put("_gateway_version", "v2");

                            String enrichedBody = objectMapper.writeValueAsString(bodyMap);
                            byte[] enrichedBytes = enrichedBody.getBytes(StandardCharsets.UTF_8);

                            ServerHttpRequest mutatedRequest = new ServerHttpRequestDecorator(exchange.getRequest()) {
                                @Override
                                public Flux<DataBuffer> getBody() {
                                    DataBuffer buffer = exchange.getResponse().bufferFactory()
                                            .wrap(enrichedBytes);
                                    return Flux.just(buffer);
                                }

                                @Override
                                public HttpHeaders getHeaders() {
                                    HttpHeaders headers = new HttpHeaders();
                                    headers.putAll(super.getHeaders());
                                    headers.setContentLength(enrichedBytes.length);
                                    return headers;
                                }
                            };

                            return chain.filter(exchange.mutate().request(mutatedRequest).build());
                        } catch (Exception e) {
                            return chain.filter(exchange);
                        }
                    });
        };
    }

    public static class Config {}
}
```

---

## 8. Fallback Controller

```java
package com.example.gateway.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @GetMapping("/{service}")
    public Mono<ResponseEntity<Map<String, Object>>> serviceFallback(@PathVariable String service) {
        Map<String, Object> body = Map.of(
                "service", service,
                "status", "unavailable",
                "message", "The " + service + " service is temporarily unavailable. Please try again later.",
                "timestamp", LocalDateTime.now().toString(),
                "retryAfter", 30
        );
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(body));
    }

    @GetMapping("/payment")
    public Mono<ResponseEntity<Map<String, Object>>> paymentFallback() {
        Map<String, Object> body = Map.of(
                "service", "payment",
                "status", "circuit_open",
                "message", "Payment service circuit breaker is open. Your request has been queued.",
                "timestamp", LocalDateTime.now().toString()
        );
        return Mono.just(ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(body));
    }
}
```

---

## 9. Gateway Metrics Configuration

```java
package com.example.gateway.config;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Tag;
import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class MetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
                .commonTags(List.of(
                        Tag.of("application", "api-gateway"),
                        Tag.of("environment", "production")
                ));
    }
}
```

```java
package com.example.gateway.filter;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.time.Instant;

@Component
public class MetricsGlobalFilter implements GlobalFilter, Ordered {

    private final MeterRegistry meterRegistry;

    public MetricsGlobalFilter(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Instant start = Instant.now();
        String path = exchange.getRequest().getURI().getPath();
        String method = exchange.getRequest().getMethod().name();
        String routeId = extractRouteId(path);

        Counter.builder("gateway.requests.total")
                .tag("method", method)
                .tag("route", routeId)
                .register(meterRegistry)
                .increment();

        return chain.filter(exchange).doOnSuccess(v -> {
            int statusCode = exchange.getResponse().getStatusCode() != null
                    ? exchange.getResponse().getStatusCode().value() : 0;

            Timer.builder("gateway.request.duration")
                    .tag("method", method)
                    .tag("route", routeId)
                    .tag("status", String.valueOf(statusCode))
                    .register(meterRegistry)
                    .record(Duration.between(start, Instant.now()));

            if (statusCode >= 400) {
                Counter.builder("gateway.requests.errors")
                        .tag("method", method)
                        .tag("route", routeId)
                        .tag("status", String.valueOf(statusCode))
                        .register(meterRegistry)
                        .increment();
            }
        });
    }

    private String extractRouteId(String path) {
        if (path.startsWith("/api/users")) return "user-service";
        if (path.startsWith("/api/orders")) return "order-service";
        if (path.startsWith("/api/payments")) return "payment-service";
        if (path.startsWith("/api/inventory")) return "inventory-service";
        return "unknown";
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }
}
```

---

## 10. CORS Configuration

```java
package com.example.gateway.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

@Configuration
public class CorsConfig {

    @Bean
    public CorsWebFilter corsWebFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.setAllowedOriginPatterns(List.of("https://*.example.com", "http://localhost:*"));
        config.setAllowedHeaders(Arrays.asList(
                "Authorization", "Content-Type", "X-Request-Id",
                "X-API-Key", "Accept", "Cache-Control"
        ));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"));
        config.setExposedHeaders(Arrays.asList("X-Request-Id", "X-Response-Time", "X-Total-Count"));
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return new CorsWebFilter(source);
    }
}
```

---

## 11. Route Predicate Factory – Business Hours

```java
package com.example.gateway.predicate;

import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.time.DayOfWeek;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;

@Component
public class BusinessHoursRoutePredicateFactory
        extends AbstractRoutePredicateFactory<BusinessHoursRoutePredicateFactory.Config> {

    public BusinessHoursRoutePredicateFactory() {
        super(Config.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        return exchange -> {
            LocalDateTime now = LocalDateTime.now();
            DayOfWeek day = now.getDayOfWeek();
            LocalTime time = now.toLocalTime();

            boolean isWeekday = day != DayOfWeek.SATURDAY && day != DayOfWeek.SUNDAY;
            boolean isBusinessHours = time.isAfter(config.getStartTime())
                    && time.isBefore(config.getEndTime());

            return isWeekday && isBusinessHours;
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("startTime", "endTime");
    }

    public static class Config {
        private LocalTime startTime = LocalTime.of(9, 0);
        private LocalTime endTime = LocalTime.of(17, 0);

        public LocalTime getStartTime() { return startTime; }
        public void setStartTime(LocalTime startTime) { this.startTime = startTime; }
        public LocalTime getEndTime() { return endTime; }
        public void setEndTime(LocalTime endTime) { this.endTime = endTime; }
    }
}
```

---

## 12. Dynamic Route Management (CRUD via API)

```java
package com.example.gateway.controller;

import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.cloud.gateway.filter.FilterDefinition;
import org.springframework.cloud.gateway.handler.predicate.PredicateDefinition;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionWriter;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;

import java.net.URI;
import java.util.List;
import java.util.UUID;

@RestController
@RequestMapping("/admin/routes")
public class DynamicRouteController {

    private final RouteDefinitionWriter routeDefinitionWriter;
    private final ApplicationEventPublisher eventPublisher;

    public DynamicRouteController(RouteDefinitionWriter routeDefinitionWriter,
                                  ApplicationEventPublisher eventPublisher) {
        this.routeDefinitionWriter = routeDefinitionWriter;
        this.eventPublisher = eventPublisher;
    }

    @PostMapping
    public Mono<ResponseEntity<String>> addRoute(@RequestBody RouteDefinitionRequest request) {
        RouteDefinition definition = new RouteDefinition();
        definition.setId(request.getId() != null ? request.getId() : UUID.randomUUID().toString());
        definition.setUri(URI.create(request.getUri()));

        PredicateDefinition pathPredicate = new PredicateDefinition();
        pathPredicate.setName("Path");
        pathPredicate.addArg("pattern", request.getPath());
        definition.setPredicates(List.of(pathPredicate));

        if (request.getStripPrefix() != null) {
            FilterDefinition stripFilter = new FilterDefinition();
            stripFilter.setName("StripPrefix");
            stripFilter.addArg("parts", request.getStripPrefix().toString());
            definition.setFilters(List.of(stripFilter));
        }

        return routeDefinitionWriter.save(Mono.just(definition))
                .doOnSuccess(v -> refreshRoutes())
                .thenReturn(ResponseEntity.ok("Route added: " + definition.getId()));
    }

    @DeleteMapping("/{routeId}")
    public Mono<ResponseEntity<String>> deleteRoute(@PathVariable String routeId) {
        return routeDefinitionWriter.delete(Mono.just(routeId))
                .doOnSuccess(v -> refreshRoutes())
                .thenReturn(ResponseEntity.ok("Route deleted: " + routeId))
                .onErrorResume(e -> Mono.just(
                        ResponseEntity.notFound().<String>build()));
    }

    private void refreshRoutes() {
        eventPublisher.publishEvent(new RefreshRoutesEvent(this));
    }

    public static class RouteDefinitionRequest {
        private String id;
        private String uri;
        private String path;
        private Integer stripPrefix;

        public String getId() { return id; }
        public void setId(String id) { this.id = id; }
        public String getUri() { return uri; }
        public void setUri(String uri) { this.uri = uri; }
        public String getPath() { return path; }
        public void setPath(String path) { this.path = path; }
        public Integer getStripPrefix() { return stripPrefix; }
        public void setStripPrefix(Integer stripPrefix) { this.stripPrefix = stripPrefix; }
    }
}
```

---

## 13. Load Balancing with Eureka

```yaml
# Eureka client configuration for the gateway
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
    registry-fetch-interval-seconds: 5
    fetch-registry: true
    register-with-eureka: true
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
    lease-expiration-duration-in-seconds: 30
    metadata-map:
      version: "2.0"
      zone: "primary"
```

```java
package com.example.gateway.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClients;
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;

@Configuration
@LoadBalancerClients({
    @LoadBalancerClient(name = "user-service", configuration = RandomLBConfig.class),
    @LoadBalancerClient(name = "order-service", configuration = RandomLBConfig.class)
})
public class LoadBalancerConfig {
}

class RandomLBConfig {
    @Bean
    public ReactorLoadBalancer<org.springframework.cloud.client.ServiceInstance> randomLoadBalancer(
            Environment environment, LoadBalancerClientFactory factory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
                factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

---

## 14. Response Caching Filter

```java
package com.example.gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.data.redis.core.ReactiveRedisTemplate;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.time.Duration;

@Component
public class ResponseCacheGatewayFilterFactory
        extends AbstractGatewayFilterFactory<ResponseCacheGatewayFilterFactory.Config> {

    private final ReactiveRedisTemplate<String, byte[]> redisTemplate;

    public ResponseCacheGatewayFilterFactory(ReactiveRedisTemplate<String, byte[]> redisTemplate) {
        super(Config.class);
        this.redisTemplate = redisTemplate;
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            if (exchange.getRequest().getMethod() != HttpMethod.GET) {
                return chain.filter(exchange);
            }

            String cacheKey = "gateway:cache:" + exchange.getRequest().getURI().toString();

            return redisTemplate.opsForValue().get(cacheKey)
                    .flatMap(cachedBytes -> {
                        exchange.getResponse().setStatusCode(HttpStatus.OK);
                        exchange.getResponse().getHeaders().add("X-Cache", "HIT");
                        org.springframework.core.io.buffer.DataBuffer buffer =
                                exchange.getResponse().bufferFactory().wrap(cachedBytes);
                        return exchange.getResponse().writeWith(Mono.just(buffer));
                    })
                    .switchIfEmpty(chain.filter(exchange)
                            .doOnSuccess(v -> {
                                // Cache the response for TTL duration
                                // In production: use ServerHttpResponseDecorator to capture body
                                exchange.getResponse().getHeaders().add("X-Cache", "MISS");
                            }));
        };
    }

    public static class Config {
        private Duration ttl = Duration.ofMinutes(5);

        public Duration getTtl() { return ttl; }
        public void setTtl(Duration ttl) { this.ttl = ttl; }
    }
}
```

---

## 15. Full Application Entry Point

```java
package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

---

## 16. Gateway Integration Tests

```java
package com.example.gateway;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GatewayRoutingTest {

    @LocalServerPort
    private int port;

    @Autowired
    private WebTestClient webTestClient;

    @Autowired
    private RouteLocator routeLocator;

    @Test
    void gatewayHasExpectedRoutes() {
        long routeCount = routeLocator.getRoutes()
                .filter(route -> route.getId() != null)
                .count()
                .block();
        assertThat(routeCount).isGreaterThan(0);
    }

    @Test
    void fallbackEndpointReturns503() {
        webTestClient
                .mutate()
                .responseTimeout(Duration.ofSeconds(10))
                .build()
                .get()
                .uri("/fallback/unknown-service")
                .exchange()
                .expectStatus().isEqualTo(503)
                .expectBody()
                .jsonPath("$.status").isEqualTo("unavailable");
    }

    @Test
    void healthEndpointIsAccessible() {
        webTestClient
                .get()
                .uri("/actuator/health")
                .exchange()
                .expectStatus().isOk()
                .expectBody()
                .jsonPath("$.status").isEqualTo("UP");
    }

    @Test
    void requestWithoutJwtIsRejected() {
        webTestClient
                .get()
                .uri("/api/users/1")
                .exchange()
                .expectStatus().isUnauthorized()
                .expectBody()
                .jsonPath("$.error").exists();
    }
}
```

---

## Summary

| Feature | Config |
|---|---|
| Path predicate | `Path=/api/users/**` |
| Host predicate | `Host=api.example.com` |
| Method predicate | `Method=GET,POST` |
| Header predicate | `Header=X-Request-Id, \\d+` |
| Cookie predicate | `Cookie=session, [a-f0-9]{32}` |
| Query predicate | `Query=q` |
| RewritePath filter | `RewritePath=/api(?<s>/.*), ${s}` |
| Rate limiting | `RequestRateLimiter` + Redis |
| Circuit breaker | `CircuitBreaker` + Resilience4j |
| Retry filter | `Retry` with backoff |
| JWT validation | `GlobalFilter` with JWTS parser |
| Custom filter | `AbstractGatewayFilterFactory` |
| Load balancing | `lb://service-name` + Eureka |
| Dynamic routes | `RouteDefinitionWriter` |
| Metrics | Micrometer + Prometheus |
