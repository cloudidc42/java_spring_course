# Part 057: Spring Session and Distributed Sessions

## Overview

The built-in HTTP session (`HttpSession`) is stored in the application's memory. This works fine for a single-instance application, but breaks in a distributed environment where load-balanced requests can land on any instance. Spring Session decouples session data from the servlet container and stores it in a shared backend — typically Redis — so all instances share the same session state.

By the end of this part you will be able to:
- Understand why distributed sessions are required
- Configure Spring Session with Redis
- Control session serialization, timeout, and expiry
- Handle concurrent sessions and remember-me
- Integrate Spring Session with Spring Security
- Test sessions in a multi-instance scenario

---

## Table of Contents

1. [HTTP Session Limitations](#1-http-session-limitations)
2. [Spring Session Overview](#2-spring-session-overview)
3. [Project Setup](#3-project-setup)
4. [Spring Session with Redis](#4-spring-session-with-redis)
5. [Session Serialization](#5-session-serialization)
6. [Session Timeout and Expiry](#6-session-timeout-and-expiry)
7. [Concurrent Session Control](#7-concurrent-session-control)
8. [Remember-Me with Spring Session](#8-remember-me-with-spring-session)
9. [Cookie vs Header-Based Session](#9-cookie-vs-header-based-session)
10. [Session Event Handling](#10-session-event-handling)
11. [Spring Session with Spring Security](#11-spring-session-with-spring-security)
12. [Real Example: Multi-Instance App with Shared Redis Sessions](#12-real-example-multi-instance-app-with-shared-redis-sessions)
13. [Summary](#13-summary)

---

## 1. HTTP Session Limitations

### In-Memory Session Problems

When running multiple instances of your application:

```
Client  ──→  Load Balancer  ──→  Instance A (session here)
                             ↕
                           Instance B (no session!)
```

If a request lands on **Instance B** but the session is on **Instance A**, the user is treated as a stranger — logged out, cart lost, etc.

**Workarounds without Spring Session:**

| Option              | Drawback                                                    |
|---------------------|-------------------------------------------------------------|
| Sticky sessions     | Breaks when instance restarts; uneven load distribution     |
| Session replication | High memory/network overhead, complex configuration         |
| JWT (stateless)     | No server-side invalidation, larger token, no fine control  |

**Spring Session solution:** Store sessions in Redis. Any instance can read any session.

---

## 2. Spring Session Overview

Spring Session provides:
- `SessionRepository` abstraction over session stores (Redis, JDBC, Hazelcast, MongoDB)
- `HttpSessionIdResolver` for cookie or header-based session IDs
- Transparent `HttpSession` API — your existing code needs no changes
- Integration hooks for Spring Security

### Architecture

```
HTTP Request
  │
  ▼
SessionRepositoryFilter  ←── intercepts all requests
  │
  ├── reads session ID from cookie / header
  ├── loads session from Redis
  ├── wraps request with virtual HttpSession
  │
  ▼
Your Controller / Spring Security
  │
  ▼
Changes propagated back to Redis on response
```

---

## 3. Project Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Web MVC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Spring Session with Redis -->
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>

    <!-- Spring Data Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Jedis or Lettuce (Lettuce is default) -->
    <!-- For Jedis:
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
    </dependency>
    -->

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
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### application.yml

```yaml
spring:
  session:
    store-type: redis           # Use Redis as session store
    redis:
      namespace: "myapp:session"    # Redis key prefix
      flush-mode: on-save           # or immediate
      cleanup-cron: "0 * * * * *"   # hourly cleanup of expired sessions
    timeout: 30m                    # Session timeout

  data:
    redis:
      host: localhost
      port: 6379
      password: ""
      timeout: 2s
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 2

server:
  servlet:
    session:
      cookie:
        name: SESSION          # Custom cookie name
        http-only: true
        secure: true           # HTTPS only in production
        same-site: Lax
        max-age: 1800          # 30 minutes (should match spring.session.timeout)

logging:
  level:
    org.springframework.session: DEBUG
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass changeme
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Two app instances to demo distributed sessions
  app1:
    build: .
    environment:
      - SERVER_PORT=8081
      - SPRING_REDIS_PASSWORD=changeme
    ports:
      - "8081:8081"
    depends_on:
      - redis

  app2:
    build: .
    environment:
      - SERVER_PORT=8082
      - SPRING_REDIS_PASSWORD=changeme
    ports:
      - "8082:8082"
    depends_on:
      - redis

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "80:80"
    depends_on:
      - app1
      - app2

volumes:
  redis_data:
```

---

## 4. Spring Session with Redis

### Auto-Configuration

With `spring-session-data-redis` on the classpath and `spring.session.store-type=redis`, Spring Boot configures everything automatically. The `@EnableRedisHttpSession` annotation is optional when using Boot's auto-configuration.

### Explicit Configuration (for advanced use)

```java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisIndexedHttpSession;
import org.springframework.session.web.http.CookieHttpSessionIdResolver;
import org.springframework.session.web.http.DefaultCookieSerializer;
import org.springframework.session.web.http.HttpSessionIdResolver;

import java.time.Duration;

@Configuration
// @EnableRedisIndexedHttpSession provides index-based lookups by principal
// (needed for concurrent session control, logout all sessions, etc.)
@EnableRedisIndexedHttpSession(
    maxInactiveIntervalInSeconds = 1800,  // 30 minutes
    redisNamespace = "myapp:session"
)
public class SpringSessionConfig {

    // Use JSON serialization instead of default Java serialization
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }

    // Configure cookie settings programmatically
    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        CookieHttpSessionIdResolver resolver = new CookieHttpSessionIdResolver();

        DefaultCookieSerializer cookieSerializer = new DefaultCookieSerializer();
        cookieSerializer.setCookieName("SESSION");
        cookieSerializer.setCookieMaxAge(1800);    // 30 minutes
        cookieSerializer.setUseHttpOnlyCookie(true);
        cookieSerializer.setUseSecureCookie(false); // true in production
        cookieSerializer.setSameSite("Lax");
        cookieSerializer.setCookiePath("/");
        // For subdomains:
        // cookieSerializer.setDomainName(".example.com");

        resolver.setCookieSerializer(cookieSerializer);
        return resolver;
    }
}
```

### Using HttpSession in Controller

```java
package com.example.app.controller;

import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/session")
@RequiredArgsConstructor
public class SessionController {

    // HttpSession API is exactly the same — Spring Session intercepts transparently
    @GetMapping("/info")
    public Map<String, Object> sessionInfo(HttpSession session) {
        Map<String, Object> info = new HashMap<>();
        info.put("sessionId", session.getId());
        info.put("creationTime", session.getCreationTime());
        info.put("lastAccessedTime", session.getLastAccessedTime());
        info.put("maxInactiveInterval", session.getMaxInactiveInterval());

        // Read session attributes
        Object cart = session.getAttribute("cart");
        info.put("cart", cart);

        return info;
    }

    @PostMapping("/cart/add")
    public Map<String, Object> addToCart(
            HttpSession session,
            @RequestParam Long productId,
            @RequestParam int quantity) {

        @SuppressWarnings("unchecked")
        Map<Long, Integer> cart = (Map<Long, Integer>)
            session.getAttribute("cart");

        if (cart == null) {
            cart = new HashMap<>();
        }

        cart.merge(productId, quantity, Integer::sum);
        session.setAttribute("cart", cart);  // stored in Redis

        return Map.of(
            "sessionId", session.getId(),
            "cartSize", cart.size(),
            "message", "Added to cart"
        );
    }

    @GetMapping("/cart")
    public Object getCart(HttpSession session) {
        return session.getAttribute("cart");
    }

    @DeleteMapping("/cart")
    public Map<String, String> clearCart(HttpSession session) {
        session.removeAttribute("cart");
        return Map.of("message", "Cart cleared");
    }

    @DeleteMapping("/invalidate")
    public Map<String, String> logout(HttpSession session) {
        session.invalidate();  // removes from Redis
        return Map.of("message", "Session invalidated");
    }
}
```

---

## 5. Session Serialization

### JSON Serialization with Jackson

```java
package com.example.app.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.security.jackson2.SecurityJackson2Modules;

@Configuration
public class SessionSerializationConfig {

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        ObjectMapper mapper = new ObjectMapper();

        // Java time support
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        // Spring Security type serialization (needed for Authentication objects)
        mapper.registerModules(SecurityJackson2Modules.getModules(getClass().getClassLoader()));

        // Store Java type info in JSON (required for deserialization)
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );

        return new GenericJackson2JsonRedisSerializer(mapper);
    }
}
```

### Session-Serializable Objects

Objects stored in the session must be serializable:

```java
package com.example.app.model;

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;

import java.io.Serial;
import java.io.Serializable;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

@Data
public class ShoppingCart implements Serializable {

    @Serial
    private static final long serialVersionUID = 1L;

    private List<CartItem> items = new ArrayList<>();

    public void addItem(CartItem item) {
        items.stream()
            .filter(i -> i.getProductId().equals(item.getProductId()))
            .findFirst()
            .ifPresentOrElse(
                existing -> existing.setQuantity(existing.getQuantity() + item.getQuantity()),
                () -> items.add(item)
            );
    }

    public void removeItem(Long productId) {
        items.removeIf(i -> i.getProductId().equals(productId));
    }

    public BigDecimal getTotal() {
        return items.stream()
            .map(i -> i.getPrice().multiply(new BigDecimal(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    @Data
    public static class CartItem implements Serializable {
        @Serial
        private static final long serialVersionUID = 1L;

        private Long productId;
        private String productName;
        private BigDecimal price;
        private int quantity;

        @JsonCreator
        public CartItem(
                @JsonProperty("productId") Long productId,
                @JsonProperty("productName") String productName,
                @JsonProperty("price") BigDecimal price,
                @JsonProperty("quantity") int quantity) {
            this.productId = productId;
            this.productName = productName;
            this.price = price;
            this.quantity = quantity;
        }
    }
}
```

---

## 6. Session Timeout and Expiry

### Timeout Configuration

```java
package com.example.app.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisHttpSession;

@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800) // 30 minutes
public class SessionTimeoutConfig {
    // Spring sets TTL on Redis keys automatically based on this value
}
```

### Per-Session Timeout Override

```java
package com.example.app.service;

import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Service;

@Service
public class SessionService {

    // Extend session for "remember me" users
    public void extendSessionForRememberMe(HttpSession session) {
        // 7 days
        session.setMaxInactiveInterval(7 * 24 * 60 * 60);
    }

    // Shorten session for sensitive operations
    public void setShortSessionTimeout(HttpSession session) {
        // 5 minutes
        session.setMaxInactiveInterval(5 * 60);
    }

    // Restore default
    public void restoreDefaultTimeout(HttpSession session) {
        session.setMaxInactiveInterval(30 * 60);  // 30 minutes
    }
}
```

### Proactive Session Expiry Check

```java
package com.example.app.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.session.FindByIndexNameSessionRepository;
import org.springframework.session.Session;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.Set;

@Slf4j
@Service
@RequiredArgsConstructor
public class SessionManagementService {

    private final FindByIndexNameSessionRepository<? extends Session> sessionRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    // Find all sessions for a user
    public Map<String, ? extends Session> getSessionsForUser(String username) {
        return sessionRepository.findByPrincipalName(username);
    }

    // Invalidate all sessions for a user (force logout everywhere)
    public void invalidateAllSessionsForUser(String username) {
        Map<String, ? extends Session> sessions = getSessionsForUser(username);
        sessions.keySet().forEach(sessionId -> {
            sessionRepository.deleteById(sessionId);
            log.info("Invalidated session {} for user {}", sessionId, username);
        });
    }

    // Count active sessions
    public int getActiveSessionCount(String username) {
        return getSessionsForUser(username).size();
    }

    // Get all active session keys in Redis
    public Set<String> getAllSessionKeys() {
        return redisTemplate.keys("myapp:session:*");
    }
}
```

---

## 7. Concurrent Session Control

Spring Security's concurrent session control limits how many sessions a single user can have simultaneously.

```java
package com.example.app.config;

import com.example.app.service.CustomUserDetailsService;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.session.SessionRegistry;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.session.HttpSessionEventPublisher;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .invalidateHttpSession(true)
                .deleteCookies("SESSION")
                .logoutSuccessUrl("/login?logout")
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(
                    org.springframework.security.config.http.SessionCreationPolicy.IF_REQUIRED
                )
                .maximumSessions(1)              // Only 1 session per user
                .maxSessionsPreventsLogin(false)  // false: new login kicks old session
                                                  // true:  new login is rejected
                .sessionRegistry(sessionRegistry())
                .expiredUrl("/login?expired")     // redirect when old session is kicked
            );

        return http.build();
    }

    @Bean
    public SessionRegistry sessionRegistry() {
        // Spring Session Redis provides a Redis-backed registry
        return new org.springframework.session.security.SpringSessionBackedSessionRegistry<>(
            findByIndexNameSessionRepository()
        );
    }

    @Bean
    public FindByIndexNameSessionRepository<? extends Session> findByIndexNameSessionRepository() {
        // Auto-configured by Spring Session — inject here
        throw new UnsupportedOperationException("Auto-configured by Spring Boot");
        // In practice, @Autowire it
    }

    // REQUIRED: publishes HttpSessionDestroyedEvent when session is invalidated
    // (needed for concurrent session control)
    @Bean
    public HttpSessionEventPublisher httpSessionEventPublisher() {
        return new HttpSessionEventPublisher();
    }
}
```

### Practical Security Config

```java
package com.example.app.config;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.session.SessionRegistry;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.session.HttpSessionEventPublisher;
import org.springframework.session.FindByIndexNameSessionRepository;
import org.springframework.session.Session;
import org.springframework.session.security.SpringSessionBackedSessionRegistry;

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class PracticalSecurityConfig {

    private final FindByIndexNameSessionRepository<? extends Session> sessionRepository;

    @Bean
    public SessionRegistry sessionRegistry() {
        return new SpringSessionBackedSessionRegistry<>(sessionRepository);
    }

    @Bean
    public HttpSessionEventPublisher httpSessionEventPublisher() {
        return new HttpSessionEventPublisher();
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**", "/login", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginProcessingUrl("/login")
                .defaultSuccessUrl("/home", true)
                .failureUrl("/login?error")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .invalidateHttpSession(true)
                .deleteCookies("SESSION")
                .clearAuthentication(true)
                .permitAll()
            )
            .sessionManagement(session -> session
                .maximumSessions(2)           // Allow 2 concurrent sessions
                .maxSessionsPreventsLogin(false)
                .sessionRegistry(sessionRegistry())
                .expiredUrl("/login?expired")
            );

        return http.build();
    }
}
```

---

## 8. Remember-Me with Spring Session

```java
package com.example.app.config;

import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.rememberme.PersistentTokenRepository;
import org.springframework.security.web.authentication.rememberme.InMemoryTokenRepositoryImpl;

@Configuration
@RequiredArgsConstructor
public class RememberMeConfig {

    private final UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain rememberMeFilterChain(HttpSecurity http) throws Exception {
        http
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            )
            .rememberMe(rm -> rm
                .rememberMeParameter("remember-me")  // form checkbox name
                .tokenValiditySeconds(7 * 24 * 60 * 60)  // 7 days
                .key("unique-and-secret-key")
                .userDetailsService(userDetailsService)
                .rememberMeCookieName("REMEMBER_ME")
                // Use persistent tokens (stored in DB) for production:
                // .tokenRepository(persistentTokenRepository())
                // .alwaysRemember(false)
            )
            .sessionManagement(session -> session
                // When remember-me is used, create a new session
                .sessionCreationPolicy(
                    org.springframework.security.config.http.SessionCreationPolicy.IF_REQUIRED
                )
            );

        return http.build();
    }

    // Extend session lifetime when remember-me is active
    @Bean
    public RememberMeSessionExtender rememberMeSessionExtender() {
        return new RememberMeSessionExtender();
    }
}

// Custom handler to extend session TTL when remember-me authenticates
class RememberMeSessionExtender
        implements org.springframework.security.web.authentication.rememberme.RememberMeAuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(
            jakarta.servlet.http.HttpServletRequest request,
            jakarta.servlet.http.HttpServletResponse response,
            org.springframework.security.core.Authentication authentication) {

        jakarta.servlet.http.HttpSession session = request.getSession(false);
        if (session != null) {
            // 7 days for remember-me sessions
            session.setMaxInactiveInterval(7 * 24 * 60 * 60);
        }
    }
}
```

---

## 9. Cookie vs Header-Based Session

### Header-Based Session (for APIs / mobile apps)

```java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.web.http.HeaderHttpSessionIdResolver;
import org.springframework.session.web.http.HttpSessionIdResolver;

@Configuration
public class HeaderSessionConfig {

    // Client sends: X-Auth-Token: <session-id>
    // Server returns: X-Auth-Token: <new-session-id>
    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        return HeaderHttpSessionIdResolver.xAuthToken();
        // Or use a custom header name:
        // return new HeaderHttpSessionIdResolver("X-Session-Id");
    }
}
```

### Usage with `X-Auth-Token`

```
POST /login
Content-Type: application/json
{ "username": "alice", "password": "secret" }

Response:
HTTP/1.1 200 OK
X-Auth-Token: 4b2e3a7f-8c1d-4e5f-9a2b-3c4d5e6f7g8h
Content-Type: application/json
{ "username": "alice", "roles": ["USER"] }

---

GET /api/profile
X-Auth-Token: 4b2e3a7f-8c1d-4e5f-9a2b-3c4d5e6f7g8h

Response:
HTTP/1.1 200 OK
{ "id": 1, "name": "Alice" }
```

### Dual Resolver (cookie for browser, header for API)

```java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.session.web.http.*;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import java.util.List;

@Configuration
public class DualSessionConfig {

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        return new HttpSessionIdResolver() {
            private final HeaderHttpSessionIdResolver header =
                HeaderHttpSessionIdResolver.xAuthToken();
            private final CookieHttpSessionIdResolver cookie =
                new CookieHttpSessionIdResolver();

            @Override
            public List<String> resolveSessionIds(HttpServletRequest request) {
                // If X-Auth-Token header present, use header resolver
                if (request.getHeader("X-Auth-Token") != null) {
                    return header.resolveSessionIds(request);
                }
                // Otherwise use cookie resolver
                return cookie.resolveSessionIds(request);
            }

            @Override
            public void setSessionId(HttpServletRequest request,
                                     HttpServletResponse response, String sessionId) {
                header.setSessionId(request, response, sessionId);
                cookie.setSessionId(request, response, sessionId);
            }

            @Override
            public void expireSession(HttpServletRequest request,
                                      HttpServletResponse response) {
                header.expireSession(request, response);
                cookie.expireSession(request, response);
            }
        };
    }
}
```

---

## 10. Session Event Handling

### Listening to Session Events

```java
package com.example.app.listener;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.session.events.SessionCreatedEvent;
import org.springframework.session.events.SessionDeletedEvent;
import org.springframework.session.events.SessionExpiredEvent;
import org.springframework.stereotype.Component;

import java.time.Instant;

@Slf4j
@Component
public class SessionEventListener {

    @EventListener
    public void onSessionCreated(SessionCreatedEvent event) {
        String sessionId = event.getSessionId();
        log.info("Session created: {}", sessionId);

        // Access session attributes
        var session = event.getSession();
        log.debug("Session created at: {}", session.getCreationTime());

        // Track metrics
        // metricsService.incrementActiveSessionCount();
    }

    @EventListener
    public void onSessionExpired(SessionExpiredEvent event) {
        String sessionId = event.getSessionId();
        log.info("Session expired: {}", sessionId);

        // Check who the session belonged to
        var session = event.getSession();
        SecurityContext secCtx = (SecurityContext)
            session.getAttribute("SPRING_SECURITY_CONTEXT");

        if (secCtx != null && secCtx.getAuthentication() != null) {
            String username = secCtx.getAuthentication().getName();
            log.info("Session for user '{}' expired", username);

            // e.g., audit log, notification, cleanup
            // auditService.recordSessionExpiry(username, sessionId);
        }
    }

    @EventListener
    public void onSessionDeleted(SessionDeletedEvent event) {
        String sessionId = event.getSessionId();
        log.info("Session deleted (logout): {}", sessionId);
        // metricsService.decrementActiveSessionCount();
    }
}
```

### Custom Session Listener for Security Events

```java
package com.example.app.listener;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.ApplicationListener;
import org.springframework.security.core.session.SessionDestroyedEvent;
import org.springframework.security.core.session.SessionInformationExpiredEvent;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class SecuritySessionListener
        implements ApplicationListener<SessionDestroyedEvent> {

    @Override
    public void onApplicationEvent(SessionDestroyedEvent event) {
        event.getSecurityContexts().forEach(ctx -> {
            if (ctx.getAuthentication() != null) {
                log.info("Security session destroyed for user: {}",
                    ctx.getAuthentication().getName());
            }
        });
    }
}
```

---

## 11. Spring Session with Spring Security

### Complete Security Integration

```java
package com.example.app.security;

import com.example.app.model.User;
import com.example.app.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.*;
import org.springframework.stereotype.Service;

import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username
            ));

        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .authorities(user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList()))
            .accountExpired(!user.isActive())
            .accountLocked(user.isLocked())
            .credentialsExpired(false)
            .disabled(!user.isActive())
            .build();
    }
}
```

### Session-Aware Authentication Endpoint

```java
package com.example.app.controller;

import com.example.app.dto.LoginRequest;
import com.example.app.dto.LoginResponse;
import com.example.app.service.SessionManagementService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.context.HttpSessionSecurityContextRepository;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authManager;
    private final SessionManagementService sessionService;

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @RequestBody LoginRequest request,
            HttpServletRequest httpRequest) {

        // Authenticate
        Authentication auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword()
            )
        );

        // Store authentication in security context
        SecurityContext ctx = SecurityContextHolder.createEmptyContext();
        ctx.setAuthentication(auth);
        SecurityContextHolder.setContext(ctx);

        // Create session and store security context
        HttpSession session = httpRequest.getSession(true);
        session.setAttribute(
            HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY,
            ctx
        );

        return ResponseEntity.ok(LoginResponse.builder()
            .sessionId(session.getId())
            .username(auth.getName())
            .roles(auth.getAuthorities().stream()
                .map(Object::toString)
                .toList())
            .build()
        );
    }

    @PostMapping("/logout")
    public ResponseEntity<Map<String, String>> logout(
            HttpServletRequest request,
            HttpSession session) {
        SecurityContextHolder.clearContext();
        session.invalidate();
        return ResponseEntity.ok(Map.of("message", "Logged out successfully"));
    }

    @PostMapping("/logout-all-devices")
    public ResponseEntity<Map<String, Object>> logoutAllDevices(
            Authentication authentication) {
        String username = authentication.getName();
        int count = sessionService.getActiveSessionCount(username);
        sessionService.invalidateAllSessionsForUser(username);
        return ResponseEntity.ok(Map.of(
            "message", "All sessions invalidated",
            "sessionsRevoked", count
        ));
    }

    @GetMapping("/sessions")
    public ResponseEntity<?> activeSessions(Authentication authentication) {
        String username = authentication.getName();
        var sessions = sessionService.getSessionsForUser(username);
        return ResponseEntity.ok(Map.of(
            "activeSessions", sessions.size(),
            "sessionIds", sessions.keySet()
        ));
    }
}
```

---

## 12. Real Example: Multi-Instance App with Shared Redis Sessions

### Complete Demo Application

#### Shopping Cart Service

```java
package com.example.app.service;

import com.example.app.model.ShoppingCart;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class CartService {

    private static final String CART_KEY = "SHOPPING_CART";

    public ShoppingCart getCart(HttpSession session) {
        ShoppingCart cart = (ShoppingCart) session.getAttribute(CART_KEY);
        if (cart == null) {
            cart = new ShoppingCart();
            session.setAttribute(CART_KEY, cart);
        }
        return cart;
    }

    public ShoppingCart addItem(HttpSession session, Long productId,
                                 String name, BigDecimal price, int qty) {
        ShoppingCart cart = getCart(session);
        cart.addItem(new ShoppingCart.CartItem(productId, name, price, qty));
        session.setAttribute(CART_KEY, cart);  // persist to Redis
        return cart;
    }

    public ShoppingCart removeItem(HttpSession session, Long productId) {
        ShoppingCart cart = getCart(session);
        cart.removeItem(productId);
        session.setAttribute(CART_KEY, cart);
        return cart;
    }

    public void clearCart(HttpSession session) {
        session.removeAttribute(CART_KEY);
    }

    public Optional<ShoppingCart> peekCart(HttpSession session) {
        return Optional.ofNullable((ShoppingCart) session.getAttribute(CART_KEY));
    }
}
```

#### Distributed Session Demo Controller

```java
package com.example.app.controller;

import com.example.app.model.ShoppingCart;
import com.example.app.service.CartService;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.Map;

@RestController
@RequestMapping("/api/cart")
@RequiredArgsConstructor
public class CartController {

    private final CartService cartService;

    @Value("${server.port:8080}")
    private String serverPort;

    @GetMapping
    public ResponseEntity<Map<String, Object>> getCart(HttpSession session) {
        ShoppingCart cart = cartService.getCart(session);
        return ResponseEntity.ok(Map.of(
            "sessionId", session.getId(),
            "servingInstance", getInstanceId(),  // shows which instance served this
            "cart", cart
        ));
    }

    @PostMapping("/items")
    public ResponseEntity<Map<String, Object>> addItem(
            HttpSession session,
            @RequestParam Long productId,
            @RequestParam String name,
            @RequestParam BigDecimal price,
            @RequestParam(defaultValue = "1") int quantity) {

        ShoppingCart cart = cartService.addItem(session, productId, name, price, quantity);
        return ResponseEntity.ok(Map.of(
            "sessionId", session.getId(),
            "servingInstance", getInstanceId(),
            "cart", cart,
            "message", "Item added. Session is persisted in Redis — try the other instance!"
        ));
    }

    @DeleteMapping("/items/{productId}")
    public ResponseEntity<ShoppingCart> removeItem(
            HttpSession session,
            @PathVariable Long productId) {
        return ResponseEntity.ok(cartService.removeItem(session, productId));
    }

    @DeleteMapping
    public ResponseEntity<Map<String, String>> clearCart(HttpSession session) {
        cartService.clearCart(session);
        return ResponseEntity.ok(Map.of("message", "Cart cleared"));
    }

    private String getInstanceId() {
        try {
            return InetAddress.getLocalHost().getHostName() + ":" + serverPort;
        } catch (UnknownHostException e) {
            return "unknown:" + serverPort;
        }
    }
}
```

### Integration Test

```java
package com.example.app;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockHttpSession;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class DistributedSessionIntegrationTest {

    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    @DynamicPropertySource
    static void redisProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }

    @Autowired
    private MockMvc mockMvc;

    @Test
    void sessionPersistedInRedis_SameSessionAcrossRequests() throws Exception {
        // First request — add to cart
        MvcResult result = mockMvc.perform(
            post("/api/cart/items")
                .with(user("alice").roles("USER"))
                .param("productId", "1")
                .param("name", "Test Product")
                .param("price", "29.99")
                .param("quantity", "2")
        )
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.cart.items[0].productId").value(1))
        .andReturn();

        // Extract session cookie
        MockHttpSession session = (MockHttpSession) result.getRequest().getSession(false);

        // Second request with same session (simulating different instance)
        mockMvc.perform(
            get("/api/cart")
                .with(user("alice").roles("USER"))
                .session(session)
        )
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.cart.items").isArray())
        .andExpect(jsonPath("$.cart.items[0].productId").value(1));
    }

    @Test
    void sessionInvalidatedOnLogout() throws Exception {
        MvcResult loginResult = mockMvc.perform(
            post("/api/cart/items")
                .with(user("bob").roles("USER"))
                .param("productId", "2")
                .param("name", "Another Product")
                .param("price", "49.99")
                .param("quantity", "1")
        )
        .andExpect(status().isOk())
        .andReturn();

        MockHttpSession session = (MockHttpSession) loginResult.getRequest().getSession(false);
        assert session != null;

        // Invalidate session
        mockMvc.perform(
            post("/api/auth/logout")
                .with(user("bob").roles("USER"))
                .session(session)
        )
        .andExpect(status().isOk());

        // Session should be gone
        assert session.isInvalid();
    }

    @Test
    void concurrentSessionsLimited() throws Exception {
        // Login from device 1
        mockMvc.perform(
            post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"username": "charlie", "password": "password"}
                    """)
        )
        .andExpect(status().isOk());

        // If maximumSessions(1), logging in from device 2 should expire device 1
        // (tested via session registry behavior in actual concurrent scenario)
    }
}
```

### Nginx Configuration for Load Balancing (No Sticky Sessions Needed)

```nginx
# nginx.conf
upstream app_backend {
    # NO ip_hash needed! Spring Session handles state in Redis
    server app1:8081;
    server app2:8082;
}

server {
    listen 80;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 13. Summary

| Feature                         | Spring Session API                                            |
|---------------------------------|---------------------------------------------------------------|
| Store type                      | `spring.session.store-type=redis`                            |
| Serialization                   | `springSessionDefaultRedisSerializer()` bean                 |
| Timeout                         | `spring.session.timeout` or `session.setMaxInactiveInterval` |
| Namespace                       | `spring.session.redis.namespace`                             |
| Cookie name                     | `DefaultCookieSerializer.setCookieName()`                    |
| Header-based session            | `HeaderHttpSessionIdResolver.xAuthToken()`                   |
| Concurrent session control      | `.sessionManagement().maximumSessions(n)`                    |
| Session registry                | `SpringSessionBackedSessionRegistry`                         |
| Force logout all devices        | `FindByIndexNameSessionRepository.findByPrincipalName()`     |
| Session events                  | `@EventListener` on `SessionCreatedEvent`, `SessionExpiredEvent` |
| Remember-me                     | `.rememberMe().tokenValiditySeconds(7_days)`                 |

### Key Takeaways

1. Spring Session is transparent — your `HttpSession` code needs zero changes.
2. Use `@EnableRedisIndexedHttpSession` to enable principal-based lookups (needed for "logout all" and concurrent session control).
3. Always use JSON serialization — it's readable, debuggable, and survives class refactoring better than Java serialization.
4. `FindByIndexNameSessionRepository` lets you find, list, and invalidate sessions by username — essential for security.
5. With Spring Session + Redis, you can remove sticky sessions from your load balancer completely.

---

## Next Part Preview

**Part 058: Distributed Tracing and Logging** — Micrometer Tracing, trace/span IDs in logs, custom spans, baggage propagation, OpenTelemetry, Jaeger integration, and structured JSON logging with ELK.
