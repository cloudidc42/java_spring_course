# Part 080: Spring Boot Admin and Management

## Overview

Spring Boot Admin (SBA) is a community project that provides a rich web UI for managing and monitoring Spring Boot applications. Instead of reading raw JSON from Actuator endpoints, SBA gives you a real dashboard: health status, live log streaming, JVM memory graphs, HTTP traces, and the ability to change log levels at runtime without restarting. This part covers everything from basic setup to multi-service monitoring with Slack alerts.

---

## 1. Architecture

```
┌─────────────────────────────────────────────┐
│           Spring Boot Admin Server           │
│           (admin-server:8090)                │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │  Web UI (React)                      │    │
│  │  - Service registry                   │    │
│  │  - Health dashboard                   │    │
│  │  - Metrics charts                     │    │
│  │  - Log streaming                      │    │
│  │  - JVM monitoring                     │    │
│  └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
          ▲                    ▲
          │ register           │ register
          │                    │
┌─────────────────┐  ┌─────────────────┐
│  order-service  │  │ product-service  │
│  :8080          │  │  :8081          │
│  /actuator/*    │  │  /actuator/*    │
└─────────────────┘  └─────────────────┘
```

---

## 2. Admin Server Setup

### Maven Dependencies

```xml
<!-- admin-server/pom.xml -->
<dependencies>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
        <version>3.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <!-- For Slack notifications -->
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-server-ui</artifactId>
        <version>3.2.0</version>
    </dependency>
</dependencies>
```

### Admin Server Application

```java
// src/main/java/com/example/admin/AdminServerApplication.java
package com.example.admin;

import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnableAdminServer
public class AdminServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }
}
```

### Admin Server Configuration

```yaml
# admin-server/src/main/resources/application.yml
server:
  port: 8090

spring:
  application:
    name: spring-boot-admin-server
  security:
    user:
      name: admin
      password: ${ADMIN_PASSWORD:changeme}
  boot:
    admin:
      ui:
        title: "My Platform Admin"
        brand: "<img src='assets/img/icon-spring-boot-admin.svg'><span>My Platform</span>"
      notify:
        slack:
          webhook-url: ${SLACK_WEBHOOK_URL:}
          channel: "#deployments"
          message: "*#{instance.registration.name}* (#{instance.id}) is *#{event.statusInfo.status}*"
        mail:
          enabled: true
          to: "ops-team@example.com"
          from: "admin-server@example.com"

# Mail configuration
  mail:
    host: ${SMTP_HOST:smtp.gmail.com}
    port: 587
    username: ${SMTP_USER:}
    password: ${SMTP_PASS:}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always

logging:
  level:
    de.codecentric.boot.admin: DEBUG
```

### Admin Server Security

```java
// src/main/java/com/example/admin/AdminServerSecurityConfig.java
package com.example.admin;

import de.codecentric.boot.admin.server.config.AdminServerProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.SavedRequestAwareAuthenticationSuccessHandler;
import org.springframework.security.web.csrf.CookieCsrfTokenRepository;
import org.springframework.security.web.util.matcher.AntPathRequestMatcher;

import java.util.UUID;

@Configuration
@EnableWebSecurity
public class AdminServerSecurityConfig {

    private final AdminServerProperties adminServer;

    public AdminServerSecurityConfig(AdminServerProperties adminServer) {
        this.adminServer = adminServer;
    }

    @Bean
    public SecurityFilterChain adminSecurityFilterChain(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler =
                new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminServer.path("/"));

        http
                .authorizeHttpRequests(auth -> auth
                        // Public endpoints
                        .requestMatchers(adminServer.path("/assets/**")).permitAll()
                        .requestMatchers(adminServer.path("/actuator/info")).permitAll()
                        .requestMatchers(adminServer.path("/actuator/health")).permitAll()
                        .requestMatchers(adminServer.path("/login")).permitAll()
                        // Client registration endpoint (for microservices to register)
                        .requestMatchers(HttpMethod.POST, adminServer.path("/instances")).permitAll()
                        .requestMatchers(HttpMethod.DELETE, adminServer.path("/instances/*")).permitAll()
                        // Everything else requires auth
                        .anyRequest().authenticated()
                )
                .formLogin(form -> form
                        .loginPage(adminServer.path("/login"))
                        .successHandler(successHandler)
                )
                .logout(logout -> logout
                        .logoutUrl(adminServer.path("/logout"))
                )
                .httpBasic(Customizer.withDefaults())
                .csrf(csrf -> csrf
                        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                        .ignoringRequestMatchers(
                                new AntPathRequestMatcher(adminServer.path("/instances"),
                                        HttpMethod.POST.toString()),
                                new AntPathRequestMatcher(adminServer.path("/instances/*"),
                                        HttpMethod.DELETE.toString()),
                                new AntPathRequestMatcher(adminServer.path("/actuator/**"))
                        )
                );

        return http.build();
    }
}
```

---

## 3. Registering Client Applications

### Client Dependencies

```xml
<!-- In each microservice's pom.xml -->
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
    <version>3.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Client Configuration

```yaml
# order-service/src/main/resources/application.yml
spring:
  application:
    name: order-service
  boot:
    admin:
      client:
        url: ${ADMIN_SERVER_URL:http://admin-server:8090}
        username: admin
        password: ${ADMIN_PASSWORD:changeme}
        instance:
          # What the admin server should show
          name: ${spring.application.name}
          service-url: http://${HOSTNAME:localhost}:${server.port:8080}
          metadata:
            environment: ${ENVIRONMENT:local}
            version: ${APP_VERSION:unknown}
            git-commit: ${GIT_COMMIT:unknown}
          prefer-ip: true
        register-once: false
        period: 10000     # Registration interval in ms
        connect-timeout: 5000

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers,
                 httptrace,threaddump,heapdump,env,
                 scheduledtasks,caches,flyway,liquibase
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
    loggers:
      enabled: true
    httptrace:
      enabled: true
    env:
      enabled: true
      show-values: when-authorized
  info:
    env:
      enabled: true
    git:
      mode: full
    build:
      enabled: true

info:
  app:
    name: '@project.name@'
    version: '@project.version@'
    description: '@project.description@'
  java:
    version: '@java.version@'
```

---

## 4. Custom Info Contributors

```java
// src/main/java/com/example/actuator/CustomInfoContributor.java
package com.example.actuator;

import com.example.repository.OrderRepository;
import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.LinkedHashMap;
import java.util.Map;

@Component
public class CustomInfoContributor implements InfoContributor {

    private static final LocalDateTime START_TIME = LocalDateTime.now();

    private final OrderRepository orderRepository;

    public CustomInfoContributor(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> runtimeInfo = new LinkedHashMap<>();
        runtimeInfo.put("startTime", START_TIME.toString());
        runtimeInfo.put("uptime", java.time.Duration.between(START_TIME, LocalDateTime.now()).toString());

        Map<String, Object> dbStats = new LinkedHashMap<>();
        try {
            dbStats.put("totalOrders", orderRepository.count());
            dbStats.put("pendingOrders",
                    orderRepository.countByStatus("PENDING"));
        } catch (Exception e) {
            dbStats.put("error", "Unable to fetch stats");
        }

        builder.withDetail("runtime", runtimeInfo);
        builder.withDetail("database", dbStats);
        builder.withDetail("featureFlags", Map.of(
                "newCheckoutFlow", System.getenv("FF_NEW_CHECKOUT") != null,
                "multiCurrency", true
        ));
    }
}
```

---

## 5. Log Level Management

### Runtime Log Level Change via API

```java
// src/main/java/com/example/admin/LoggingController.java
package com.example.admin;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.LoggerContext;
import org.slf4j.LoggerFactory;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/internal/logging")
@PreAuthorize("hasRole('ADMIN')")
public class LoggingController {

    @GetMapping("/loggers")
    public ResponseEntity<List<Map<String, String>>> listLoggers() {
        LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();

        List<Map<String, String>> loggers = context.getLoggerList().stream()
                .filter(logger -> logger.getLevel() != null)
                .map(logger -> Map.of(
                        "name", logger.getName(),
                        "level", logger.getEffectiveLevel().toString()
                ))
                .collect(Collectors.toList());

        return ResponseEntity.ok(loggers);
    }

    @PostMapping("/loggers/{loggerName}")
    public ResponseEntity<Void> setLogLevel(
            @PathVariable String loggerName,
            @RequestParam String level) {

        LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();
        ch.qos.logback.classic.Logger logger = context.getLogger(loggerName);

        if (logger != null) {
            logger.setLevel(Level.toLevel(level.toUpperCase()));
            return ResponseEntity.ok().build();
        }

        return ResponseEntity.notFound().build();
    }
}
```

### Log Level Change via Actuator (preferred)

```bash
# Change log level via Spring Boot Actuator endpoint
curl -X POST http://localhost:8080/actuator/loggers/com.example.order \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'

# Reset to default
curl -X POST http://localhost:8080/actuator/loggers/com.example.order \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": null}'
```

---

## 6. Custom Notification Strategy

```java
// src/main/java/com/example/admin/notification/SlackNotifier.java
package com.example.admin.notification;

import de.codecentric.boot.admin.server.domain.entities.Instance;
import de.codecentric.boot.admin.server.domain.entities.InstanceRepository;
import de.codecentric.boot.admin.server.domain.events.InstanceEvent;
import de.codecentric.boot.admin.server.domain.events.InstanceStatusChangedEvent;
import de.codecentric.boot.admin.server.notify.AbstractStatusChangeNotifier;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.web.client.RestTemplate;
import reactor.core.publisher.Mono;

import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SlackNotifier extends AbstractStatusChangeNotifier {

    private static final Logger log = LoggerFactory.getLogger(SlackNotifier.class);

    private final String webhookUrl;
    private final RestTemplate restTemplate;
    private String channel = "#deployments";
    private String username = "Spring Boot Admin";

    public SlackNotifier(InstanceRepository repository,
                         String webhookUrl,
                         RestTemplate restTemplate) {
        super(repository);
        this.webhookUrl = webhookUrl;
        this.restTemplate = restTemplate;
    }

    @Override
    protected Mono<Void> doNotify(InstanceEvent event, Instance instance) {
        if (!(event instanceof InstanceStatusChangedEvent statusEvent)) {
            return Mono.empty();
        }

        String prevStatus = statusEvent.getLastStatus();
        String newStatus = instance.getStatusInfo().getStatus();

        String color = getColor(newStatus);
        String emoji = getEmoji(newStatus);

        Map<String, Object> attachment = new HashMap<>();
        attachment.put("color", color);
        attachment.put("title", emoji + " " + instance.getRegistration().getName());
        attachment.put("fields", List.of(
                Map.of("title", "Status", "value",
                        prevStatus + " → " + newStatus, "short", true),
                Map.of("title", "Instance", "value",
                        instance.getId().getValue(), "short", true),
                Map.of("title", "URL", "value",
                        instance.getRegistration().getServiceUrl(), "short", false),
                Map.of("title", "Time", "value",
                        Instant.now().toString(), "short", false)
        ));

        Map<String, Object> payload = new HashMap<>();
        payload.put("channel", channel);
        payload.put("username", username);
        payload.put("attachments", List.of(attachment));

        return Mono.fromRunnable(() -> {
            try {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);

                restTemplate.postForEntity(
                        webhookUrl,
                        new HttpEntity<>(payload, headers),
                        String.class
                );
                log.info("Slack notification sent for instance: {}", instance.getId());
            } catch (Exception e) {
                log.error("Failed to send Slack notification", e);
            }
        });
    }

    private String getColor(String status) {
        return switch (status) {
            case "UP" -> "good";
            case "DOWN" -> "danger";
            case "OFFLINE" -> "danger";
            case "UNKNOWN" -> "warning";
            default -> "#gray";
        };
    }

    private String getEmoji(String status) {
        return switch (status) {
            case "UP" -> ":white_check_mark:";
            case "DOWN" -> ":red_circle:";
            case "OFFLINE" -> ":black_circle:";
            case "UNKNOWN" -> ":question:";
            default -> ":grey_question:";
        };
    }

    public void setChannel(String channel) {
        this.channel = channel;
    }

    public void setUsername(String username) {
        this.username = username;
    }
}
```

### Notification Bean Configuration

```java
// src/main/java/com/example/admin/NotificationConfig.java
package com.example.admin;

import com.example.admin.notification.SlackNotifier;
import de.codecentric.boot.admin.server.domain.entities.InstanceRepository;
import de.codecentric.boot.admin.server.notify.CompositeNotifier;
import de.codecentric.boot.admin.server.notify.Notifier;
import de.codecentric.boot.admin.server.notify.RemindingNotifier;
import de.codecentric.boot.admin.server.notify.filter.FilteringNotifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.web.client.RestTemplate;

import java.time.Duration;
import java.util.List;

@Configuration
public class NotificationConfig {

    @Value("${spring.boot.admin.notify.slack.webhook-url:}")
    private String slackWebhookUrl;

    @Bean
    public SlackNotifier slackNotifier(InstanceRepository repository,
                                        RestTemplate restTemplate) {
        SlackNotifier notifier = new SlackNotifier(repository, slackWebhookUrl, restTemplate);
        notifier.setChannel("#platform-alerts");
        notifier.setEnabled(!slackWebhookUrl.isEmpty());
        return notifier;
    }

    @Bean
    public FilteringNotifier filteringNotifier(InstanceRepository repository,
                                                List<Notifier> notifiers) {
        CompositeNotifier composite = new CompositeNotifier(notifiers);
        FilteringNotifier filter = new FilteringNotifier(composite, repository);

        // Don't notify for "UNKNOWN" status (happens during startup)
        // This is configured via the UI normally
        return filter;
    }

    @Bean
    @Primary
    public RemindingNotifier remindingNotifier(FilteringNotifier filteringNotifier,
                                               InstanceRepository repository) {
        RemindingNotifier remindingNotifier = new RemindingNotifier(
                filteringNotifier, repository);

        // Re-notify every 10 minutes if still DOWN
        remindingNotifier.setReminderPeriod(Duration.ofMinutes(10));

        // Stop reminding after 2 hours
        remindingNotifier.setCheckReminderInSeconds(10);

        return remindingNotifier;
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

## 7. JVM Monitoring

### Custom JVM Metrics

```java
// src/main/java/com/example/metrics/JvmDiagnosticsEndpoint.java
package com.example.metrics;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.*;
import io.micrometer.core.instrument.binder.system.ProcessorMetrics;
import io.micrometer.core.instrument.binder.system.UptimeMetrics;
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.stereotype.Component;

import java.lang.management.*;
import java.util.*;

@Component
@Endpoint(id = "jvm-diagnostics")
public class JvmDiagnosticsEndpoint {

    @ReadOperation
    public Map<String, Object> jvmDiagnostics() {
        Map<String, Object> result = new LinkedHashMap<>();

        // Memory
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        result.put("heapMemory", formatMemoryUsage(memoryBean.getHeapMemoryUsage()));
        result.put("nonHeapMemory", formatMemoryUsage(memoryBean.getNonHeapMemoryUsage()));

        // Memory pools
        List<Map<String, Object>> pools = new ArrayList<>();
        for (MemoryPoolMXBean pool : ManagementFactory.getMemoryPoolMXBeans()) {
            Map<String, Object> poolInfo = new LinkedHashMap<>();
            poolInfo.put("name", pool.getName());
            poolInfo.put("type", pool.getType().toString());
            poolInfo.put("usage", formatMemoryUsage(pool.getUsage()));
            pools.add(poolInfo);
        }
        result.put("memoryPools", pools);

        // Threads
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        result.put("threads", Map.of(
                "total", threadBean.getThreadCount(),
                "daemon", threadBean.getDaemonThreadCount(),
                "peak", threadBean.getPeakThreadCount(),
                "started", threadBean.getTotalStartedThreadCount()
        ));

        // GC
        List<Map<String, Object>> gcStats = new ArrayList<>();
        for (GarbageCollectorMXBean gc : ManagementFactory.getGarbageCollectorMXBeans()) {
            gcStats.add(Map.of(
                    "name", gc.getName(),
                    "collectionCount", gc.getCollectionCount(),
                    "collectionTimeMs", gc.getCollectionTime()
            ));
        }
        result.put("garbageCollectors", gcStats);

        // Runtime
        RuntimeMXBean runtimeBean = ManagementFactory.getRuntimeMXBean();
        result.put("runtime", Map.of(
                "jvmName", runtimeBean.getVmName(),
                "jvmVersion", runtimeBean.getVmVersion(),
                "uptimeMs", runtimeBean.getUptime(),
                "arguments", runtimeBean.getInputArguments()
        ));

        return result;
    }

    private Map<String, Object> formatMemoryUsage(MemoryUsage usage) {
        if (usage == null) return Map.of();

        return Map.of(
                "usedMB", usage.getUsed() / (1024 * 1024),
                "committedMB", usage.getCommitted() / (1024 * 1024),
                "maxMB", usage.getMax() > 0 ? usage.getMax() / (1024 * 1024) : -1,
                "usagePercent", usage.getMax() > 0
                        ? (usage.getUsed() * 100.0) / usage.getMax()
                        : -1
        );
    }
}
```

### Thread Dump Analysis

```java
// src/main/java/com/example/actuator/ThreadDumpAnalyzer.java
package com.example.actuator;

import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.*;
import java.util.stream.Collectors;

@Component
@Endpoint(id = "thread-analysis")
public class ThreadDumpAnalyzer {

    @ReadOperation
    public Map<String, Object> analyzeThreads() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] allThreads = threadBean.dumpAllThreads(true, true);

        // Detect deadlocks
        long[] deadlockedThreads = threadBean.findDeadlockedThreads();

        // Group by state
        Map<Thread.State, List<String>> threadsByState = Arrays.stream(allThreads)
                .collect(Collectors.groupingBy(
                        ThreadInfo::getThreadState,
                        Collectors.mapping(ThreadInfo::getThreadName, Collectors.toList())
                ));

        // Find blocked threads
        List<Map<String, Object>> blockedThreads = Arrays.stream(allThreads)
                .filter(t -> t.getThreadState() == Thread.State.BLOCKED
                        || t.getThreadState() == Thread.State.WAITING)
                .map(t -> {
                    Map<String, Object> info = new LinkedHashMap<>();
                    info.put("name", t.getThreadName());
                    info.put("state", t.getThreadState().toString());
                    info.put("blockedOnLock", t.getLockName());
                    info.put("blockedByThread", t.getLockOwnerName());
                    return info;
                })
                .collect(Collectors.toList());

        return Map.of(
                "totalThreads", allThreads.length,
                "deadlocks", deadlockedThreads != null ? deadlockedThreads.length : 0,
                "deadlockedThreadIds",
                        deadlockedThreads != null ? Arrays.asList(deadlockedThreads) : List.of(),
                "threadsByState", threadsByState.entrySet().stream()
                        .collect(Collectors.toMap(
                                e -> e.getKey().toString(),
                                e -> Map.of("count", e.getValue().size(), "threads", e.getValue())
                        )),
                "blockedOrWaiting", blockedThreads
        );
    }
}
```

---

## 8. HTTP Trace Viewer

```java
// src/main/java/com/example/actuator/HttpTraceConfig.java
package com.example.actuator;

import org.springframework.boot.actuate.web.exchanges.HttpExchange;
import org.springframework.boot.actuate.web.exchanges.HttpExchangeRepository;
import org.springframework.boot.actuate.web.exchanges.InMemoryHttpExchangeRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HttpTraceConfig {

    @Bean
    public HttpExchangeRepository httpExchangeRepository() {
        InMemoryHttpExchangeRepository repository = new InMemoryHttpExchangeRepository();
        repository.setCapacity(1000);  // Keep last 1000 exchanges
        return repository;
    }
}
```

### HTTP Exchange Filter (for custom fields)

```java
// src/main/java/com/example/actuator/CustomHttpExchangeFilter.java
package com.example.actuator;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.boot.actuate.web.exchanges.HttpExchangeRepository;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Instant;
import java.util.*;

@Component
@Order(Integer.MIN_VALUE)
public class CustomHttpExchangeFilter implements Filter {

    private static final List<String> EXCLUDED_PATHS = List.of(
            "/actuator/",
            "/health",
            "/favicon.ico"
    );

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        // Skip actuator and health endpoints
        String path = httpRequest.getRequestURI();
        boolean shouldExclude = EXCLUDED_PATHS.stream()
                .anyMatch(path::startsWith);

        if (!shouldExclude) {
            // Add request ID for correlation
            String requestId = UUID.randomUUID().toString().substring(0, 8);
            httpRequest.setAttribute("X-Request-Id", requestId);
        }

        chain.doFilter(request, response);
    }
}
```

---

## 9. SBA with Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  # Spring Boot Admin Server
  admin-server:
    image: my-registry/admin-server:latest
    build:
      context: ./admin-server
    ports:
      - "8090:8090"
    environment:
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:-admin123}
      SLACK_WEBHOOK_URL: ${SLACK_WEBHOOK_URL:-}
      SPRING_MAIL_HOST: mailhog
      SPRING_MAIL_PORT: 1025
    networks:
      - platform-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8090/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Order Service
  order-service:
    image: my-registry/order-service:latest
    build:
      context: ./order-service
    ports:
      - "8080:8080"
    environment:
      SPRING_BOOT_ADMIN_CLIENT_URL: http://admin-server:8090
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:-admin123}
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/orders
      ENVIRONMENT: docker
    depends_on:
      - postgres
      - admin-server
    networks:
      - platform-network

  # Product Service
  product-service:
    image: my-registry/product-service:latest
    ports:
      - "8081:8080"
    environment:
      SPRING_BOOT_ADMIN_CLIENT_URL: http://admin-server:8090
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:-admin123}
    depends_on:
      - admin-server
    networks:
      - platform-network

  # Payment Service
  payment-service:
    image: my-registry/payment-service:latest
    ports:
      - "8082:8080"
    environment:
      SPRING_BOOT_ADMIN_CLIENT_URL: http://admin-server:8090
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:-admin123}
    depends_on:
      - admin-server
    networks:
      - platform-network

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_MULTIPLE_DATABASES: orders,products
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - platform-network

  # MailHog for email testing
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI
    networks:
      - platform-network

networks:
  platform-network:
    driver: bridge

volumes:
  postgres-data:
```

---

## 10. Multi-Instance Monitoring

### Scaling Detection

```java
// src/main/java/com/example/admin/InstanceHealthDashboard.java
package com.example.admin;

import de.codecentric.boot.admin.server.domain.entities.Instance;
import de.codecentric.boot.admin.server.domain.entities.InstanceRepository;
import de.codecentric.boot.admin.server.domain.values.StatusInfo;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/dashboard")
public class InstanceHealthDashboard {

    private final InstanceRepository instanceRepository;

    public InstanceHealthDashboard(InstanceRepository instanceRepository) {
        this.instanceRepository = instanceRepository;
    }

    @GetMapping("/overview")
    public Mono<Map<String, Object>> getOverview() {
        return instanceRepository.findAll()
                .collectList()
                .map(instances -> {
                    long totalInstances = instances.size();
                    long upInstances = instances.stream()
                            .filter(i -> "UP".equals(i.getStatusInfo().getStatus()))
                            .count();
                    long downInstances = instances.stream()
                            .filter(i -> "DOWN".equals(i.getStatusInfo().getStatus()))
                            .count();
                    long unknownInstances = totalInstances - upInstances - downInstances;

                    Map<String, List<Map<String, String>>> byApplication =
                            instances.stream()
                                    .collect(Collectors.groupingBy(
                                            i -> i.getRegistration().getName(),
                                            Collectors.mapping(
                                                    i -> Map.of(
                                                            "id", i.getId().getValue(),
                                                            "status", i.getStatusInfo().getStatus(),
                                                            "url", i.getRegistration().getServiceUrl()
                                                    ),
                                                    Collectors.toList()
                                            )
                                    ));

                    return Map.of(
                            "summary", Map.of(
                                    "total", totalInstances,
                                    "up", upInstances,
                                    "down", downInstances,
                                    "unknown", unknownInstances,
                                    "healthPercentage", totalInstances > 0
                                            ? (upInstances * 100.0) / totalInstances
                                            : 0
                            ),
                            "applications", byApplication
                    );
                });
    }
}
```

---

## 11. Custom Views and Panels

### Custom Actuator View Registration

```java
// src/main/java/com/example/admin/CustomViewConfig.java
package com.example.admin;

import de.codecentric.boot.admin.server.ui.config.AdminServerUiProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;

import jakarta.annotation.PostConstruct;
import java.util.List;

@Configuration
public class CustomViewConfig {

    @Autowired
    private AdminServerUiProperties adminServerUiProperties;

    @PostConstruct
    public void configureViews() {
        // Add custom external links to the SBA menu
        adminServerUiProperties.getExternalViews().add(
                new AdminServerUiProperties.ExternalView(
                        "Grafana",
                        "http://grafana:3000",
                        "_blank",
                        "#grafana-icon",
                        List.of("ADMIN"),
                        false
                )
        );

        adminServerUiProperties.getExternalViews().add(
                new AdminServerUiProperties.ExternalView(
                        "Prometheus",
                        "http://prometheus:9090",
                        "_blank",
                        "#prometheus-icon",
                        List.of("ADMIN"),
                        false
                )
        );
    }
}
```

---

## 12. Spring Boot Admin Security for Multi-Tenant

```java
// src/main/java/com/example/admin/MultiTenantSecurityConfig.java
package com.example.admin;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class MultiTenantSecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder encoder) {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();

        // Admin: can see all services, change log levels, trigger actions
        manager.createUser(User.builder()
                .username("admin")
                .password(encoder.encode("admin-password"))
                .roles("ADMIN", "USER")
                .build());

        // Viewer: can only read metrics and health
        manager.createUser(User.builder()
                .username("viewer")
                .password(encoder.encode("viewer-password"))
                .roles("USER")
                .build());

        // Service account: for microservices to register
        manager.createUser(User.builder()
                .username("service-account")
                .password(encoder.encode("service-password"))
                .roles("SERVICE")
                .build());

        return manager;
    }
}
```

---

## Summary

| Feature | How to Access | Notes |
|---------|--------------|-------|
| Service Registry | SBA UI → Applications | All registered instances |
| Health Status | SBA UI → Instance → Details | Uses /actuator/health |
| Metrics | SBA UI → Instance → Metrics | Real-time charts |
| Log Streaming | SBA UI → Instance → Logfile | Requires logging.file.name |
| Log Level | SBA UI → Instance → Loggers | Changes take effect immediately |
| HTTP Traces | SBA UI → Instance → HTTP Traces | Last 1000 requests |
| Thread Dump | SBA UI → Instance → Threads | Live thread state |
| Heap Dump | SBA UI → Instance → Heap Dump | Downloads .hprof file |
| Environment | SBA UI → Instance → Environment | Configuration properties |
| JVM Memory | SBA UI → Instance → JVM | Memory pool graphs |

### Key URLs

| Endpoint | URL |
|----------|-----|
| SBA Web UI | http://admin-server:8090 |
| SBA REST API | http://admin-server:8090/api/applications |
| Client Health | http://service:8080/actuator/health |
| Client Metrics | http://service:8080/actuator/metrics |
| Client Loggers | http://service:8080/actuator/loggers |

---

## Next Part Preview

**Part 081: Spring Boot on AWS Lambda** - We'll build Spring Boot functions for AWS Lambda using spring-cloud-function, optimize cold starts with SnapStart, and build a real image processing pipeline with S3 and SQS triggers.
