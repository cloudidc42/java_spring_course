# Part 082: Multi-Module Spring Boot Projects

## Overview

As your codebase grows, splitting it into modules provides faster builds (only rebuild changed modules), clearer boundaries between components, enforced dependency rules, and the ability to publish shared libraries. This part builds a complete e-commerce monorepo with 8 Maven modules, covering everything from parent POM management to CI/CD optimization.

---

## 1. Project Structure

```
ecommerce-platform/
├── pom.xml                          (parent POM - no code)
├── commons/                         (shared utilities, exceptions, DTOs)
│   ├── pom.xml
│   └── src/main/java/com/example/commons/
├── api-contracts/                   (OpenAPI specs + generated DTOs)
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/example/api/
│       └── resources/openapi/
├── security/                        (shared auth/JWT logic)
│   ├── pom.xml
│   └── src/main/java/com/example/security/
├── order-service/                   (order management)
│   ├── pom.xml
│   └── src/main/java/com/example/order/
├── product-service/                 (product catalog)
│   ├── pom.xml
│   └── src/main/java/com/example/product/
├── payment-service/                 (payment processing)
│   ├── pom.xml
│   └── src/main/java/com/example/payment/
├── notification-service/            (email/SMS)
│   ├── pom.xml
│   └── src/main/java/com/example/notification/
└── e2e-tests/                       (end-to-end tests)
    ├── pom.xml
    └── src/test/java/com/example/e2e/
```

---

## 2. Parent POM

```xml
<!-- ecommerce-platform/pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>ecommerce-platform</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>E-Commerce Platform</name>
    <description>Multi-module e-commerce platform</description>

    <!-- List all modules in build order -->
    <modules>
        <module>commons</module>
        <module>api-contracts</module>
        <module>security</module>
        <module>order-service</module>
        <module>product-service</module>
        <module>payment-service</module>
        <module>notification-service</module>
        <module>e2e-tests</module>
    </modules>

    <!-- Inherit from Spring Boot parent for dependency management -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <properties>
        <!-- Java version - applied to all modules -->
        <java.version>21</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>

        <!-- Encoding -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Project version (use for cross-module dependencies) -->
        <ecommerce.version>1.0.0-SNAPSHOT</ecommerce.version>

        <!-- Third-party versions (not managed by Spring Boot) -->
        <spring-cloud.version>2023.0.0</spring-cloud.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <openapi-generator.version>7.1.0</openapi-generator.version>
        <testcontainers.version>1.19.3</testcontainers.version>
    </properties>

    <!-- Dependency management: versions without mandatory inclusion -->
    <dependencyManagement>
        <dependencies>
            <!-- Spring Cloud BOM -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- TestContainers BOM -->
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Internal modules - managed here so child modules don't specify versions -->
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>commons</artifactId>
                <version>${ecommerce.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>api-contracts</artifactId>
                <version>${ecommerce.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>security</artifactId>
                <version>${ecommerce.version}</version>
            </dependency>

            <!-- Third-party (not in Spring Boot BOM) -->
            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- Dependencies included in ALL modules -->
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <!-- Plugin management: versions and config for all modules -->
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
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
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok-mapstruct-binding</artifactId>
                                <version>0.2.0</version>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <configuration>
                        <argLine>-Xmx512m</argLine>
                        <excludes>
                            <exclude>**/*E2ETest.java</exclude>
                        </excludes>
                    </configuration>
                </plugin>

                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <executions>
                        <execution>
                            <goals>
                                <goal>integration-test</goal>
                                <goal>verify</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>

                <plugin>
                    <groupId>org.jacoco</groupId>
                    <artifactId>jacoco-maven-plugin</artifactId>
                    <version>0.8.11</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>prepare-agent</goal>
                            </goals>
                        </execution>
                        <execution>
                            <id>report</id>
                            <phase>test</phase>
                            <goals>
                                <goal>report</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>

        <!-- Plugins applied to ALL modules -->
        <plugins>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <!-- Profiles for different environments -->
    <profiles>
        <profile>
            <id>integration-tests</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration>
                            <excludes/>  <!-- Include everything -->
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>

        <profile>
            <id>release</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-source-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>attach-sources</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-javadoc-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>attach-javadocs</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>
```

---

## 3. Commons Module

```xml
<!-- commons/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>commons</artifactId>
    <name>Commons - Shared Library</name>
    <!-- No packaging = jar by default -->

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Commons is a library, NOT a Spring Boot fat JAR -->
            <!-- Do NOT include spring-boot-maven-plugin here -->
        </plugins>
    </build>
</project>
```

### Shared Exception Classes

```java
// commons/src/main/java/com/example/commons/exception/ApiException.java
package com.example.commons.exception;

import org.springframework.http.HttpStatus;

public class ApiException extends RuntimeException {

    private final HttpStatus status;
    private final String errorCode;
    private final Object details;

    public ApiException(HttpStatus status, String errorCode, String message) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
        this.details = null;
    }

    public ApiException(HttpStatus status, String errorCode, String message, Object details) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
        this.details = details;
    }

    public HttpStatus getStatus() { return status; }
    public String getErrorCode() { return errorCode; }
    public Object getDetails() { return details; }
}
```

```java
// commons/src/main/java/com/example/commons/exception/ResourceNotFoundException.java
package com.example.commons.exception;

import org.springframework.http.HttpStatus;

public class ResourceNotFoundException extends ApiException {

    public ResourceNotFoundException(String resourceType, Object id) {
        super(HttpStatus.NOT_FOUND,
              "RESOURCE_NOT_FOUND",
              resourceType + " not found with id: " + id);
    }
}
```

### Shared Error Response Handler

```java
// commons/src/main/java/com/example/commons/web/GlobalExceptionHandler.java
package com.example.commons.web;

import com.example.commons.exception.ApiException;
import com.example.commons.exception.ResourceNotFoundException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.time.Instant;
import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setProperty("errorCode", ex.getErrorCode());
        problem.setProperty("timestamp", Instant.now().toString());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(problem);
    }

    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ProblemDetail> handleApiException(ApiException ex) {
        log.warn("API exception: {} - {}", ex.getErrorCode(), ex.getMessage());

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                ex.getStatus(), ex.getMessage());
        problem.setType(URI.create("https://api.example.com/errors/" +
                ex.getErrorCode().toLowerCase().replace("_", "-")));
        problem.setProperty("errorCode", ex.getErrorCode());
        problem.setProperty("timestamp", Instant.now().toString());

        if (ex.getDetails() != null) {
            problem.setProperty("details", ex.getDetails());
        }

        return ResponseEntity.status(ex.getStatus()).body(problem);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidation(
            MethodArgumentNotValidException ex) {

        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.toMap(
                        FieldError::getField,
                        error -> error.getDefaultMessage() != null
                                ? error.getDefaultMessage() : "Invalid value",
                        (existing, replacement) -> existing
                ));

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setType(URI.create("https://api.example.com/errors/validation-failed"));
        problem.setProperty("errors", fieldErrors);
        problem.setProperty("timestamp", Instant.now().toString());

        return ResponseEntity.badRequest().body(problem);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
        problem.setType(URI.create("https://api.example.com/errors/internal-error"));
        problem.setProperty("timestamp", Instant.now().toString());

        return ResponseEntity.internalServerError().body(problem);
    }
}
```

### Shared Pagination

```java
// commons/src/main/java/com/example/commons/web/PageResponse.java
package com.example.commons.web;

import org.springframework.data.domain.Page;

import java.util.List;

public record PageResponse<T>(
        List<T> content,
        int page,
        int size,
        long totalElements,
        int totalPages,
        boolean last,
        boolean first
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
                page.getContent(),
                page.getNumber(),
                page.getSize(),
                page.getTotalElements(),
                page.getTotalPages(),
                page.isLast(),
                page.isFirst()
        );
    }
}
```

---

## 4. API Contracts Module

```xml
<!-- api-contracts/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <artifactId>api-contracts</artifactId>
    <name>API Contracts</name>

    <dependencies>
        <dependency>
            <groupId>io.swagger.core.v3</groupId>
            <artifactId>swagger-annotations</artifactId>
            <version>2.2.19</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
        </dependency>
        <dependency>
            <groupId>jakarta.validation</groupId>
            <artifactId>jakarta.validation-api</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.openapitools</groupId>
                <artifactId>openapi-generator-maven-plugin</artifactId>
                <version>${openapi-generator.version}</version>
                <executions>
                    <!-- Generate order service DTOs -->
                    <execution>
                        <id>generate-order-api</id>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                        <configuration>
                            <inputSpec>${project.basedir}/src/main/resources/openapi/order-api.yaml</inputSpec>
                            <generatorName>spring</generatorName>
                            <generateApiTests>false</generateApiTests>
                            <generateModelTests>false</generateModelTests>
                            <configOptions>
                                <basePackage>com.example.api.order</basePackage>
                                <modelPackage>com.example.api.order.model</modelPackage>
                                <apiPackage>com.example.api.order.api</apiPackage>
                                <interfaceOnly>true</interfaceOnly>
                                <useSpringBoot3>true</useSpringBoot3>
                                <useJakartaEe>true</useJakartaEe>
                                <useBeanValidation>true</useBeanValidation>
                            </configOptions>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### OpenAPI Spec

```yaml
# api-contracts/src/main/resources/openapi/order-api.yaml
openapi: 3.0.3
info:
  title: Order Service API
  description: API for managing customer orders
  version: 1.0.0
  contact:
    name: Platform Team
    email: platform@example.com

servers:
  - url: http://localhost:8080
    description: Local development

paths:
  /api/orders:
    get:
      operationId: listOrders
      summary: List orders for a customer
      parameters:
        - name: customerId
          in: query
          schema:
            type: string
            format: uuid
        - name: page
          in: query
          schema:
            type: integer
            default: 0
        - name: size
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Orders list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderPage'
    post:
      operationId: createOrder
      summary: Create a new order
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'

  /api/orders/{orderId}:
    get:
      operationId: getOrder
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Order details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [customerId, items]
      properties:
        customerId:
          type: string
          format: uuid
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItemRequest'
        shippingAddress:
          $ref: '#/components/schemas/Address'

    OrderItemRequest:
      type: object
      required: [productId, quantity]
      properties:
        productId:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1

    OrderResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        customerId:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED]
        totalAmount:
          type: number
          format: double
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItemResponse'
        createdAt:
          type: string
          format: date-time

    OrderItemResponse:
      type: object
      properties:
        productId:
          type: string
          format: uuid
        quantity:
          type: integer
        unitPrice:
          type: number

    OrderPage:
      type: object
      properties:
        content:
          type: array
          items:
            $ref: '#/components/schemas/OrderResponse'
        page:
          type: integer
        size:
          type: integer
        totalElements:
          type: integer
          format: int64
        totalPages:
          type: integer

    Address:
      type: object
      required: [street, city, country]
      properties:
        street:
          type: string
        city:
          type: string
        postalCode:
          type: string
        country:
          type: string

  responses:
    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetail'
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetail'

    ProblemDetail:
      type: object
      properties:
        type:
          type: string
        title:
          type: string
        status:
          type: integer
        detail:
          type: string
        errorCode:
          type: string
```

---

## 5. Security Module

```xml
<!-- security/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <artifactId>security</artifactId>
    <name>Security - Shared Auth Library</name>

    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>commons</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>
    </dependencies>
</project>
```

### Shared Security Auto-Configuration

```java
// security/src/main/java/com/example/security/EcommerceSecurityAutoConfiguration.java
package com.example.security;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@AutoConfiguration
@EnableWebSecurity
@EnableMethodSecurity
@ConditionalOnProperty(name = "ecommerce.security.enabled", matchIfMissing = true)
public class EcommerceSecurityAutoConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .sessionManagement(session -> session
                        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(
                                "/actuator/health",
                                "/actuator/info",
                                "/v3/api-docs/**",
                                "/swagger-ui/**"
                        ).permitAll()
                        .anyRequest().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(jwt -> jwt
                                .jwtAuthenticationConverter(
                                        new EcommerceJwtConverter())
                        )
                )
                .csrf(csrf -> csrf.disable())
                .build();
    }
}
```

```java
// security/src/main/java/com/example/security/EcommerceJwtConverter.java
package com.example.security;

import org.springframework.core.convert.converter.Converter;
import org.springframework.security.authentication.AbstractAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class EcommerceJwtConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    private final JwtGrantedAuthoritiesConverter defaultConverter =
            new JwtGrantedAuthoritiesConverter();

    @Override
    public AbstractAuthenticationToken convert(Jwt jwt) {
        Collection<GrantedAuthority> authorities = Stream.concat(
                defaultConverter.convert(jwt).stream(),
                extractRoles(jwt).stream()
        ).collect(Collectors.toSet());

        return new JwtAuthenticationToken(jwt, authorities, jwt.getSubject());
    }

    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractRoles(Jwt jwt) {
        var realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess == null) return List.of();

        var roles = (List<String>) realmAccess.get("roles");
        if (roles == null) return List.of();

        return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                .collect(Collectors.toList());
    }
}
```

```
# Auto-configuration registration
# security/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.security.EcommerceSecurityAutoConfiguration
```

---

## 6. Order Service Module (Consumes Internal Modules)

```xml
<!-- order-service/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <artifactId>order-service</artifactId>
    <name>Order Service</name>

    <dependencies>
        <!-- Internal modules - no version needed (managed in parent) -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>commons</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>api-contracts</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>security</artifactId>
        </dependency>

        <!-- Spring Boot starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Test dependencies -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- This module IS a deployable Spring Boot app -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Order Controller Implementing API Contract

```java
// order-service/src/main/java/com/example/order/web/OrderController.java
package com.example.order.web;

import com.example.api.order.api.OrdersApi;              // Generated from OpenAPI
import com.example.api.order.model.CreateOrderRequest;   // Generated DTOs
import com.example.api.order.model.OrderResponse;
import com.example.api.order.model.OrderPage;
import com.example.commons.web.PageResponse;
import com.example.order.service.OrderService;
import org.springframework.data.domain.PageRequest;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.RestController;

import java.net.URI;
import java.util.UUID;

/**
 * Implements the generated API interface from api-contracts module.
 * This ensures the implementation always matches the OpenAPI spec.
 */
@RestController
public class OrderController implements OrdersApi {

    private final OrderService orderService;
    private final OrderMapper orderMapper;

    public OrderController(OrderService orderService, OrderMapper orderMapper) {
        this.orderService = orderService;
        this.orderMapper = orderMapper;
    }

    @Override
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<OrderPage> listOrders(UUID customerId, Integer page, Integer size) {
        var orders = orderService.findByCustomerId(
                customerId, PageRequest.of(page != null ? page : 0, size != null ? size : 20));

        OrderPage response = new OrderPage()
                .content(orders.getContent().stream()
                        .map(orderMapper::toResponse)
                        .toList())
                .page(orders.getNumber())
                .size(orders.getSize())
                .totalElements(orders.getTotalElements())
                .totalPages(orders.getTotalPages());

        return ResponseEntity.ok(response);
    }

    @Override
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<OrderResponse> createOrder(CreateOrderRequest request) {
        var order = orderService.createOrder(orderMapper.toDomain(request));
        OrderResponse response = orderMapper.toResponse(order);
        return ResponseEntity.created(
                URI.create("/api/orders/" + order.getId()))
                .body(response);
    }

    @Override
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<OrderResponse> getOrder(UUID orderId) {
        return orderService.findById(orderId)
                .map(orderMapper::toResponse)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

---

## 7. Gradle Multi-Project Build

### Root build.gradle

```groovy
// build.gradle (root)
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0' apply false  // apply false = only manage version
    id 'io.spring.dependency-management' version '1.1.4' apply false
}

// Configuration applied to ALL subprojects
allprojects {
    group = 'com.example'
    version = '1.0.0-SNAPSHOT'

    repositories {
        mavenCentral()
        maven {
            url 'https://repo.spring.io/milestone'
        }
    }
}

// Configuration applied to subprojects only (not root)
subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'

    java {
        sourceCompatibility = JavaVersion.VERSION_21
        targetCompatibility = JavaVersion.VERSION_21
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:2023.0.0"
            mavenBom "org.testcontainers:testcontainers-bom:1.19.3"
        }
    }

    dependencies {
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }

    test {
        useJUnitPlatform()
        maxHeapSize = '512m'
        exclude '**/*E2ETest*'
    }

    // Enforce minimum test coverage
    tasks.named('test') {
        finalizedBy tasks.named('jacocoTestReport', { } )
    }
}
```

### settings.gradle

```groovy
// settings.gradle
rootProject.name = 'ecommerce-platform'

include 'commons'
include 'api-contracts'
include 'security'
include 'order-service'
include 'product-service'
include 'payment-service'
include 'notification-service'
include 'e2e-tests'
```

### Service build.gradle

```groovy
// order-service/build.gradle
plugins {
    id 'org.springframework.boot'
}

dependencies {
    implementation project(':commons')
    implementation project(':api-contracts')
    implementation project(':security')

    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'

    runtimeOnly 'org.postgresql:postgresql'

    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'
}

bootJar {
    archiveFileName = 'order-service.jar'
}
```

---

## 8. Build Optimization

### Parallel Builds

```bash
# Maven: build all modules in parallel (up to 4 threads)
mvn package -T 4

# Maven: use 1 thread per CPU core
mvn package -T 1C

# Gradle: parallel by default in Gradle 7+, but you can tune:
./gradlew build --parallel --max-workers=4
```

### Incremental Builds (Only Build Changed Modules)

```bash
# Gradle: only rebuild modules that changed
./gradlew :order-service:build --configuration-cache

# Maven: build only a specific module and its dependencies
mvn package -pl order-service -am

# Maven: build only modules that have changed since last commit
mvn package -pl $(git diff --name-only HEAD~1 | grep "src/" | cut -d/ -f1 | sort -u | tr '\n' ',')
```

### Maven Build Cache (Daemon + Local Cache)

```xml
<!-- .mvn/maven.config -->
-Dmaven.build.cache.enabled=true
-T 1C
--fail-at-end
```

### Gradle Build Cache Configuration

```groovy
// gradle.properties
org.gradle.caching=true
org.gradle.parallel=true
org.gradle.daemon=true
org.gradle.jvmargs=-Xmx2g -XX:+HeapDumpOnOutOfMemoryError
org.gradle.configuration-cache=true
```

---

## 9. Inter-Module Testing

```java
// e2e-tests/pom.xml (depends on all services for integration tests)
```

```xml
<!-- e2e-tests/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-platform</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <artifactId>e2e-tests</artifactId>
    <name>End-to-End Tests</name>

    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>api-contracts</artifactId>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <skipTests>${skipE2ETests}</skipTests>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

```java
// e2e-tests/src/test/java/com/example/e2e/OrderFlowE2ETest.java
package com.example.e2e;

import com.example.api.order.model.CreateOrderRequest;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.DockerComposeContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.io.File;
import java.util.List;
import java.util.UUID;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@Testcontainers
class OrderFlowE2ETest {

    @Container
    static DockerComposeContainer<?> environment =
            new DockerComposeContainer<>(new File("docker-compose-test.yml"))
                    .withExposedService("order-service", 8080)
                    .withExposedService("product-service", 8081)
                    .withLocalCompose(true);

    @BeforeAll
    static void setUp() {
        String orderServiceHost = environment.getServiceHost("order-service", 8080);
        int orderServicePort = environment.getServicePort("order-service", 8080);
        RestAssured.baseURI = "http://" + orderServiceHost + ":" + orderServicePort;
    }

    @Test
    void shouldCreateAndConfirmOrder() {
        UUID customerId = UUID.randomUUID();
        UUID productId = UUID.randomUUID();

        // Create order
        String orderId = given()
                .contentType(ContentType.JSON)
                .header("Authorization", "Bearer " + getTestToken(customerId))
                .body(new CreateOrderRequest()
                        .customerId(customerId)
                        .items(List.of()))
                .when()
                .post("/api/orders")
                .then()
                .statusCode(201)
                .body("status", equalTo("PENDING"))
                .extract()
                .jsonPath()
                .getString("id");

        // Confirm order
        given()
                .contentType(ContentType.JSON)
                .header("Authorization", "Bearer " + getTestToken(customerId))
                .when()
                .post("/api/orders/{orderId}/confirm", orderId)
                .then()
                .statusCode(200)
                .body("status", equalTo("CONFIRMED"));
    }

    private String getTestToken(UUID userId) {
        // Get test token from auth service
        return given()
                .contentType(ContentType.URLENC)
                .formParam("grant_type", "client_credentials")
                .formParam("client_id", "test-client")
                .formParam("client_secret", "test-secret")
                .when()
                .post("http://keycloak:8080/auth/realms/test/protocol/openid-connect/token")
                .then()
                .extract()
                .jsonPath()
                .getString("access_token");
    }
}
```

---

## 10. CI/CD for Multi-Module Projects

### GitHub Actions Workflow

```yaml
# .github/workflows/build.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'

jobs:
  # Detect which modules changed
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      commons: ${{ steps.filter.outputs.commons }}
      order-service: ${{ steps.filter.outputs.order-service }}
      product-service: ${{ steps.filter.outputs.product-service }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            commons:
              - 'commons/**'
            order-service:
              - 'order-service/**'
              - 'commons/**'
              - 'api-contracts/**'
              - 'security/**'
            product-service:
              - 'product-service/**'
              - 'commons/**'
              - 'api-contracts/**'

  # Build and test all modules
  build:
    needs: detect-changes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'

      # Cache Maven dependencies
      - uses: actions/cache@v3
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # Build all modules in parallel
      - name: Build and test
        run: mvn -B package -T 1C --fail-at-end
        env:
          MAVEN_OPTS: "-Xmx2g"

      # Upload test results
      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: '**/target/surefire-reports/*.xml'

      # Upload coverage report
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: '**/target/site/jacoco/jacoco.xml'

  # Deploy order-service if it or its deps changed
  deploy-order-service:
    needs: [build, detect-changes]
    if: |
      needs.detect-changes.outputs.order-service == 'true'
      && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'

      - name: Build order-service Docker image
        run: |
          mvn -B package -pl order-service -am -DskipTests
          docker build -t order-service:${{ github.sha }} order-service/

      - name: Push to ECR and deploy
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
          docker tag order-service:${{ github.sha }} $ECR_REGISTRY/order-service:${{ github.sha }}
          docker push $ECR_REGISTRY/order-service:${{ github.sha }}
          # Trigger ECS deployment or Kubernetes rollout
```

---

## 11. Monorepo vs Polyrepo Trade-offs

| Aspect | Monorepo | Polyrepo |
|--------|---------|---------|
| Code sharing | Easy (local modules) | Harder (publish to registry) |
| Atomic changes | Easy (one commit changes multiple services) | Requires coordinated PRs |
| Build time | Slower (build entire repo) | Faster per repo (smaller scope) |
| CI/CD | Complex (smart change detection needed) | Simple (one service = one pipeline) |
| Team autonomy | Lower (shared build, shared dependencies) | Higher (team owns the build) |
| Dependency drift | Minimal (shared parent POM) | Common |
| Tooling | Needs advanced tools (Turborepo, Nx, Bazel) | Standard tools |
| IDE performance | Can be slow for large repos | Fast |

---

## 12. Publishing to Maven Central

```xml
<!-- commons/pom.xml (for publishable library) -->
<distributionManagement>
    <snapshotRepository>
        <id>ossrh</id>
        <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    <repository>
        <id>ossrh</id>
        <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
</distributionManagement>

<build>
    <plugins>
        <plugin>
            <groupId>org.sonatype.plugins</groupId>
            <artifactId>nexus-staging-maven-plugin</artifactId>
            <version>1.6.13</version>
            <extensions>true</extensions>
            <configuration>
                <serverId>ossrh</serverId>
                <nexusUrl>https://s01.oss.sonatype.org/</nexusUrl>
                <autoReleaseAfterClose>true</autoReleaseAfterClose>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-gpg-plugin</artifactId>
            <executions>
                <execution>
                    <id>sign-artifacts</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>sign</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

```bash
# Release steps
mvn versions:set -DnewVersion=1.0.0
mvn clean deploy -P release -pl commons,api-contracts,security
git tag v1.0.0
git push --tags
mvn versions:set -DnewVersion=1.0.1-SNAPSHOT
```

---

## Summary

| Module | Type | Contains | Consumed By |
|--------|------|----------|-------------|
| commons | Library JAR | Exceptions, Page response, utilities | All services |
| api-contracts | Library JAR | OpenAPI specs, generated DTOs | All services |
| security | Auto-config JAR | JWT converter, security config | All services |
| order-service | Spring Boot JAR | Order business logic, REST API | Deployed |
| product-service | Spring Boot JAR | Product catalog, inventory | Deployed |
| payment-service | Spring Boot JAR | Payment processing | Deployed |
| notification-service | Spring Boot JAR | Email/SMS | Deployed |
| e2e-tests | Test-only JAR | End-to-end tests | CI only |

### Build Commands

```bash
# Build all
mvn package -T 1C

# Build specific module and its deps
mvn package -pl order-service -am

# Skip tests for speed
mvn package -DskipTests -T 1C

# Run integration tests
mvn verify -Pintegration-tests

# Release
mvn versions:set -DnewVersion=1.1.0 && mvn deploy -Prelease

# Gradle equivalents
./gradlew build --parallel
./gradlew :order-service:build
./gradlew build -x test
```

---

## Next Part Preview

**Part 083: Spring Boot Performance Tuning** - We'll dive deep into JVM tuning for Spring Boot: G1GC vs ZGC garbage collectors, connection pool sizing, slow query detection, caching with Caffeine and Redis, and profiling with async-profiler and JFR.
