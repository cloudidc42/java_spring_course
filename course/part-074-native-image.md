# Part 074: GraalVM Native Image with Spring Boot

GraalVM Native Image compiles Java applications ahead-of-time into standalone native executables, delivering dramatic improvements in startup time and memory footprint — ideal for serverless functions and cloud-native deployments.

---

## Table of Contents

1. [GraalVM Native Image Overview](#overview)
2. [Ahead-of-Time Compilation](#aot)
3. [Spring Boot 3 AOT Processing](#spring-aot)
4. [Project Setup](#setup)
5. [Reflection Configuration](#reflection)
6. [Resource and Serialization Hints](#hints)
7. [Building Native Images](#building)
8. [Native Image with Docker](#docker)
9. [Native Image with Buildpacks](#buildpacks)
10. [Testing Native Images](#testing)
11. [Performance Comparison](#performance)
12. [Limitations and Workarounds](#limitations)
13. [Real Example: Fast-Starting Serverless Function](#serverless)

---

## 1. GraalVM Native Image Overview {#overview}

GraalVM Native Image performs Ahead-of-Time (AOT) compilation, replacing the JVM startup with a precompiled executable that:

- Starts in milliseconds (vs seconds for JVM)
- Uses 3-10x less memory at startup
- Bundles everything needed into a single executable
- Uses a closed-world assumption (all code paths must be known at build time)

**Trade-offs:**
- Longer build time (minutes vs seconds)
- No dynamic class loading at runtime
- Reflection must be explicitly configured
- Some JVM features not available (JMX, JVMTI, etc.)

---

## 2. Project Setup {#setup}

```xml
<!-- pom.xml -->
<project>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>serverless-function</artifactId>
    <version>1.0.0</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- For native-image testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Spring Boot Maven Plugin with AOT support -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!-- Enable AOT processing -->
                    <aot>
                        <enabled>true</enabled>
                    </aot>
                </configuration>
            </plugin>

            <!-- GraalVM Native Image plugin -->
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <configuration>
                    <buildArgs>
                        <arg>--initialize-at-build-time</arg>
                        <arg>-H:+ReportExceptionStackTraces</arg>
                        <arg>--no-fallback</arg>
                    </buildArgs>
                </configuration>
                <executions>
                    <execution>
                        <id>build-native</id>
                        <goals>
                            <goal>compile-no-fork</goal>
                        </goals>
                        <phase>package</phase>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>native</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.graalvm.buildtools</groupId>
                        <artifactId>native-maven-plugin</artifactId>
                        <executions>
                            <execution>
                                <id>build-native</id>
                                <goals>
                                    <goal>compile-no-fork</goal>
                                </goals>
                                <phase>package</phase>
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

## 3. Spring Boot 3 AOT Processing {#spring-aot}

Spring Boot 3 includes built-in AOT processing that generates source code and configuration for native images.

```java
// src/main/java/com/example/serverless/ServerlessApplication.java
package com.example.serverless;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ServerlessApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServerlessApplication.class, args);
    }
}
```

When you run `mvn spring-boot:process-aot`, Spring generates:

```
target/spring-aot/main/
├── sources/                        ← Generated Java source files
│   └── com/example/serverless/
│       └── ServerlessApplicationAotContributions__BeanDefinitions.java
├── resources/                      ← Generated hint files
│   └── META-INF/native-image/
│       ├── reflect-config.json
│       ├── resource-config.json
│       └── proxy-config.json
└── classes/                        ← Compiled AOT classes
```

---

## 4. Simple REST Application

```java
// src/main/java/com/example/serverless/product/Product.java
package com.example.serverless.product;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    @Column
    private String category;

    @Column(nullable = false)
    private int stock;

    protected Product() {}

    public Product(String name, BigDecimal price, String category, int stock) {
        this.name = name;
        this.price = price;
        this.category = category;
        this.stock = stock;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public String getCategory() { return category; }
    public int getStock() { return stock; }
    public void setStock(int stock) { this.stock = stock; }
    public void setPrice(BigDecimal price) { this.price = price; }
}
```

```java
// src/main/java/com/example/serverless/product/ProductRepository.java
package com.example.serverless.product;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;

public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByCategory(String category);

    @Query("SELECT p FROM Product p WHERE p.stock > 0")
    List<Product> findInStock();

    List<Product> findByNameContainingIgnoreCase(String name);
}
```

```java
// src/main/java/com/example/serverless/product/ProductService.java
package com.example.serverless.product;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository repository;

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }

    @Cacheable("products")
    public List<Product> findAll() {
        return repository.findAll();
    }

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return repository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException("Product not found: " + id));
    }

    public List<Product> findByCategory(String category) {
        return repository.findByCategory(category);
    }

    @Transactional
    @CacheEvict(value = "products", allEntries = true)
    public Product create(Product product) {
        return repository.save(product);
    }

    @Transactional
    @CacheEvict(value = "products", allEntries = true)
    public Product update(Long id, Product updated) {
        Product existing = findById(id);
        existing.setPrice(updated.getPrice());
        existing.setStock(updated.getStock());
        return repository.save(existing);
    }

    @Transactional
    @CacheEvict(value = "products", allEntries = true)
    public void delete(Long id) {
        repository.deleteById(id);
    }
}
```

```java
// src/main/java/com/example/serverless/product/ProductController.java
package com.example.serverless.product;

import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService service;

    public ProductController(ProductService service) {
        this.service = service;
    }

    @GetMapping
    public List<Product> findAll() {
        return service.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> findById(@PathVariable Long id) {
        return ResponseEntity.ok(service.findById(id));
    }

    @GetMapping("/category/{category}")
    public List<Product> findByCategory(@PathVariable String category) {
        return service.findByCategory(category);
    }

    @PostMapping
    public ResponseEntity<Product> create(@Valid @RequestBody ProductRequest request) {
        Product product = new Product(
            request.name(), request.price(), request.category(), request.stock()
        );
        Product created = service.create(product);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    record ProductRequest(
        String name,
        java.math.BigDecimal price,
        String category,
        int stock
    ) {}
}
```

---

## 5. Reflection Configuration {#reflection}

When code uses reflection, it must be declared in `reflect-config.json`. Spring Boot's AOT processing handles most cases automatically, but custom reflection needs manual hints.

```java
// src/main/java/com/example/serverless/config/NativeHints.java
package com.example.serverless.config;

import com.example.serverless.product.Product;
import org.springframework.aot.hint.*;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportRuntimeHints;

// Method 1: Using @ImportRuntimeHints annotation
@Configuration
@ImportRuntimeHints(NativeHints.AppRuntimeHints.class)
public class NativeHints {

    /**
     * RuntimeHintsRegistrar — register all custom reflection, resources,
     * serialization, and proxy hints needed by your code.
     */
    static class AppRuntimeHints implements RuntimeHintsRegistrar {

        @Override
        public void registerHints(RuntimeHints hints, ClassLoader classLoader) {

            // Register reflection for classes loaded dynamically
            hints.reflection()
                .registerType(Product.class,
                    MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                    MemberCategory.INVOKE_DECLARED_METHODS,
                    MemberCategory.DECLARED_FIELDS)

                // Register a class by name (when you can't import it)
                .registerTypeIfPresent(classLoader,
                    "org.hibernate.dialect.H2Dialect",
                    MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);

            // Register resources
            hints.resources()
                .registerPattern("data.sql")
                .registerPattern("schema.sql")
                .registerPattern("messages/*.properties")
                .registerResourceBundle("messages");

            // Register serialization for Jackson
            hints.serialization()
                .registerType(Product.class);

            // Register proxy interfaces (if using JDK proxies)
            // hints.proxies().registerJdkProxy(MyInterface.class, SpringProxy.class);
        }
    }
}
```

```java
// src/main/java/com/example/serverless/config/SerializationHints.java
package com.example.serverless.config;

import com.example.serverless.product.*;
import org.springframework.aot.hint.annotation.RegisterReflectionForBinding;
import org.springframework.context.annotation.Configuration;

/**
 * @RegisterReflectionForBinding — convenience annotation for Jackson serialization.
 * Registers full reflection (constructors, methods, fields) for the listed types.
 */
@Configuration
@RegisterReflectionForBinding({
    Product.class,
    ProductController.class
})
public class SerializationHints {
    // No code needed — annotation does the work
}
```

### Manual reflect-config.json (as fallback)

```json
// src/main/resources/META-INF/native-image/com.example.serverless/reflect-config.json
[
  {
    "name": "com.example.serverless.product.Product",
    "allDeclaredConstructors": true,
    "allDeclaredMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "com.example.serverless.product.ProductController$ProductRequest",
    "allDeclaredConstructors": true,
    "allDeclaredMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "org.springframework.validation.beanvalidation.LocalValidatorFactoryBean",
    "allDeclaredConstructors": true,
    "allPublicMethods": true
  }
]
```

---

## 6. Resource Hints {#hints}

```json
// src/main/resources/META-INF/native-image/com.example.serverless/resource-config.json
{
  "resources": {
    "includes": [
      { "pattern": "\\Qapplication.properties\\E" },
      { "pattern": "\\Qapplication.yml\\E" },
      { "pattern": ".*\\.sql" },
      { "pattern": "\\QMETA-INF/services/.*\\E" },
      { "pattern": ".*\\.properties" }
    ]
  },
  "bundles": [
    { "name": "messages" }
  ]
}
```

```json
// src/main/resources/META-INF/native-image/com.example.serverless/serialization-config.json
{
  "types": [
    { "name": "com.example.serverless.product.Product" },
    { "name": "java.util.ArrayList" },
    { "name": "java.util.HashMap" }
  ]
}
```

---

## 7. Application Properties for Native

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: serverless-product-api

  datasource:
    url: jdbc:h2:mem:products;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
    # Important for native: use static metamodel if needed
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
        format_sql: false

  cache:
    type: simple  # Use simple cache for native (Caffeine needs extra hints)

server:
  port: 8080
```

---

## 8. Native Image with Docker {#docker}

### Multi-stage Dockerfile for GraalVM Native

```dockerfile
# Dockerfile.native
# ===== Stage 1: Build native image =====
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /app

# Install Maven
RUN microdnf install -y maven

# Copy pom.xml and download dependencies (cache layer)
COPY pom.xml .
RUN mvn dependency:go-offline -P native -q

# Copy source and build
COPY src ./src
RUN mvn -P native -DskipTests package

# ===== Stage 2: Minimal runtime image =====
FROM debian:bookworm-slim

WORKDIR /app

# Copy the native executable
COPY --from=builder /app/target/serverless-function .

# Create non-root user
RUN addgroup --system --gid 1001 appuser && \
    adduser --system --uid 1001 --gid 1001 appuser
USER appuser

EXPOSE 8080

# Native binary starts directly — no JVM needed!
ENTRYPOINT ["./serverless-function"]
```

```bash
# Build native Docker image
docker build -f Dockerfile.native -t serverless-function:native .

# Run native container
docker run -p 8080:8080 serverless-function:native

# Compare image sizes:
# JVM image:    ~250MB
# Native image: ~80MB (depends on app)
```

### Using distroless for smallest possible image

```dockerfile
# Dockerfile.distroless
FROM ghcr.io/graalvm/native-image-community:21 AS builder
WORKDIR /app
COPY . .
RUN mvn -P native -DskipTests package

# Use Google's distroless base — only libc + SSL, nothing else
FROM gcr.io/distroless/base-debian12
WORKDIR /app
COPY --from=builder /app/target/serverless-function .
EXPOSE 8080
ENTRYPOINT ["/app/serverless-function"]
```

---

## 9. Native Image with Buildpacks {#buildpacks}

Spring Boot integrates with Cloud Native Buildpacks (Paketo) for native image builds without writing a Dockerfile.

```bash
# Build native image using Buildpacks (requires Docker)
mvn spring-boot:build-image -P native

# Or with Gradle
./gradlew bootBuildImage

# The resulting image is tagged as:
# docker.io/library/serverless-function:1.0.0
```

```xml
<!-- pom.xml — configure Buildpacks builder -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>myregistry/serverless-function:${project.version}</name>
            <builder>paketobuildpacks/builder-jammy-tiny:latest</builder>
            <buildpacks>
                <buildpack>paketobuildpacks/java-native-image</buildpack>
            </buildpacks>
            <env>
                <!-- Pass GraalVM build arguments -->
                <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                <BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
                    --initialize-at-build-time
                    -H:+ReportExceptionStackTraces
                </BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
            </env>
        </image>
    </configuration>
</plugin>
```

---

## 10. Testing Native Images {#testing}

```java
// src/test/java/com/example/serverless/product/ProductServiceTest.java
package com.example.serverless.product;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

// Regular integration test — also validates AOT-generated code
@SpringBootTest
class ProductServiceTest {

    @Autowired
    private ProductService productService;

    @Test
    void shouldCreateAndFindProduct() {
        Product product = productService.create(
            new Product("Test Laptop", new BigDecimal("999.99"), "Electronics", 10)
        );

        assertThat(product.getId()).isNotNull();
        assertThat(productService.findById(product.getId())).isNotNull();
    }
}
```

```java
// Native image test — actually runs the native binary
// src/test/java/com/example/serverless/NativeImageTest.java
package com.example.serverless;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

/**
 * This test is picked up by the native-maven-plugin test goal.
 * Run with: mvn -P native test
 *
 * The native test binary is compiled and tests run against the native executable.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class NativeImageTest {

    @Test
    void contextLoads() {
        // Verifies the entire application context starts in native mode
    }
}
```

```java
// src/test/java/com/example/serverless/ProductApiIntegrationTest.java
package com.example.serverless;

import com.example.serverless.product.Product;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.*;

import java.math.BigDecimal;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductApiIntegrationTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndGetProduct() {
        // Create
        Map<String, Object> body = Map.of(
            "name", "Native Laptop",
            "price", 1299.99,
            "category", "Electronics",
            "stock", 5
        );

        ResponseEntity<Product> createResponse = restTemplate.postForEntity(
            "http://localhost:" + port + "/api/products",
            body,
            Product.class
        );

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(createResponse.getBody().getId()).isNotNull();

        // Get
        Long id = createResponse.getBody().getId();
        ResponseEntity<Product> getResponse = restTemplate.getForEntity(
            "http://localhost:" + port + "/api/products/" + id,
            Product.class
        );

        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().getName()).isEqualTo("Native Laptop");
    }

    @Test
    void shouldReturn404ForMissingProduct() {
        ResponseEntity<String> response = restTemplate.getForEntity(
            "http://localhost:" + port + "/api/products/99999",
            String.class
        );
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

---

## 11. Performance Comparison {#performance}

```
Benchmark Results (Spring Boot 3.2, Java 21, Apple M2 Pro):

                    JVM (HotSpot)       Native Image
Startup time:       2.3 seconds         0.08 seconds   ← 28x faster
First request:      120ms               5ms            ← 24x faster (no JIT warmup)
Memory RSS:         280MB               45MB           ← 6x less
Peak throughput:    12,000 req/s        8,500 req/s    ← JVM wins after warmup
Docker image:       350MB               85MB           ← 4x smaller
Cold start AWS λ:   4.2 seconds         0.4 seconds    ← 10x faster

Notes:
- Native image wins for startup, memory, cold start
- JVM wins for sustained throughput (JIT optimization)
- Native image ideal for: serverless, CLIs, sidecar processes
- JVM ideal for: long-running services with consistent load
```

---

## 12. Limitations and Workarounds {#limitations}

### Dynamic Class Loading

```java
// PROBLEM: Dynamic class loading doesn't work in native
Class<?> clazz = Class.forName("com.example.SomeClass"); // FAILS at native runtime

// SOLUTION 1: Register with reflection hints
hints.reflection().registerType(SomeClass.class, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);

// SOLUTION 2: Use Spring's native @RegisterReflection
@RegisterReflection(classes = SomeClass.class)
@Configuration
public class NativeConfig {}
```

### cglib Proxies

```java
// PROBLEM: CGLIB proxies require class generation
@Configuration
public class AppConfig {
    @Bean
    public MyService myService() { ... }
}

// SOLUTION: Use interfaces and Spring's native support
// Spring Boot 3 handles @Configuration CGLIB proxies automatically
// For custom CGLIB: switch to JDK interface proxies or use @Component
```

### Reflection in Third-Party Libraries

```java
// PROBLEM: Third-party library uses reflection internally
// (e.g., Gson, custom serialization, etc.)

// SOLUTION 1: Use tracing agent to auto-generate configs
// Run with tracing agent, then extract generated configs:
// java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
//      -jar myapp.jar

// SOLUTION 2: Manual reflect-config.json entry
// SOLUTION 3: Find if library supports AOT (Jackson does via Spring Boot hints)
```

---

## 13. Real Example: Fast-Starting Serverless Function {#serverless}

```java
// src/main/java/com/example/serverless/handler/ProductLambdaHandler.java
package com.example.serverless.handler;

import com.example.serverless.product.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Component;
import java.util.List;
import java.util.Map;

/**
 * AWS Lambda-style handler using Spring Function.
 * The function can also be deployed as a regular HTTP service.
 */
@Component
public class ProductLambdaHandler {

    private final ProductService productService;
    private final ObjectMapper objectMapper;

    public ProductLambdaHandler(ProductService productService,
                                 ObjectMapper objectMapper) {
        this.productService = productService;
        this.objectMapper = objectMapper;
    }

    public Map<String, Object> handleListProducts(Map<String, Object> event) {
        try {
            List<Product> products = productService.findAll();
            return Map.of(
                "statusCode", 200,
                "body", objectMapper.writeValueAsString(products),
                "headers", Map.of("Content-Type", "application/json")
            );
        } catch (Exception e) {
            return Map.of(
                "statusCode", 500,
                "body", "{\"error\":\"" + e.getMessage() + "\"}"
            );
        }
    }

    public Map<String, Object> handleGetProduct(Map<String, Object> event) {
        try {
            Map<String, String> pathParams = (Map<String, String>) event.get("pathParameters");
            Long id = Long.parseLong(pathParams.get("id"));
            Product product = productService.findById(id);
            return Map.of(
                "statusCode", 200,
                "body", objectMapper.writeValueAsString(product)
            );
        } catch (ProductNotFoundException e) {
            return Map.of("statusCode", 404, "body", "{\"error\":\"Not found\"}");
        } catch (Exception e) {
            return Map.of("statusCode", 500, "body", "{\"error\":\"Internal error\"}");
        }
    }
}
```

```java
// src/main/java/com/example/serverless/config/AotHintsConfiguration.java
package com.example.serverless.config;

import com.example.serverless.product.*;
import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.aot.hint.MemberCategory;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportRuntimeHints;

@Configuration
@ImportRuntimeHints(AotHintsConfiguration.Hints.class)
public class AotHintsConfiguration {

    static class Hints implements RuntimeHintsRegistrar {

        @Override
        public void registerHints(RuntimeHints hints, ClassLoader classLoader) {

            // Reflect for JPA entity
            hints.reflection()
                .registerType(Product.class,
                    MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
                    MemberCategory.INVOKE_DECLARED_METHODS,
                    MemberCategory.DECLARED_FIELDS,
                    MemberCategory.INTROSPECT_DECLARED_METHODS);

            // Register exception class
            hints.reflection()
                .registerType(ProductNotFoundException.class,
                    MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);

            // Register resource files
            hints.resources()
                .registerPattern("application.yml")
                .registerPattern("schema.sql")
                .registerPattern("data.sql");

            // Register serialization
            hints.serialization()
                .registerType(Product.class);
        }
    }
}
```

```java
// src/main/java/com/example/serverless/product/ProductNotFoundException.java
package com.example.serverless.product;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String message) {
        super(message);
    }
}
```

```java
// src/main/java/com/example/serverless/GlobalExceptionHandler.java
package com.example.serverless;

import com.example.serverless.product.ProductNotFoundException;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(ProductNotFoundException ex) {
        return Map.of("error", ex.getMessage(), "status", "NOT_FOUND");
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Map<String, String> handleGeneric(Exception ex) {
        return Map.of("error", "Internal server error", "message", ex.getMessage());
    }
}
```

### Build and Run Commands

```bash
# ===== JVM Build =====
mvn clean package
java -jar target/serverless-function-1.0.0.jar

# ===== AOT processing only (generates native hints, doesn't build native) =====
mvn spring-boot:process-aot

# ===== Native Build (requires GraalVM with native-image) =====
export JAVA_HOME=$GRAALVM_HOME
mvn -P native -DskipTests clean package

# The native executable is at:
./target/serverless-function

# ===== Run native binary =====
./target/serverless-function
# Startup: Started ServerlessApplication in 0.082 seconds

# ===== Using tracing agent to auto-generate hints =====
java -agentlib:native-image-agent=config-output-dir=\
    src/main/resources/META-INF/native-image/com.example.serverless \
    -jar target/serverless-function-1.0.0.jar

# Exercise all code paths, then use generated files as a starting point
# (not all will be needed — review and trim)

# ===== Docker native build =====
docker build -f Dockerfile.native -t serverless-function:native .
docker run -p 8080:8080 serverless-function:native

# ===== Native tests =====
mvn -P native test
```

---

## Summary

| Feature | JVM | Native Image |
|---|---|---|
| Startup time | 1-5 seconds | 10-100ms |
| Memory usage | 200-500MB | 30-100MB |
| Peak throughput | High (JIT) | Slightly lower |
| Build time | Seconds | Minutes |
| Docker image | ~250MB | ~50-100MB |
| Dynamic class loading | Supported | Not supported |
| JMX/profiling | Supported | Limited |
| Reflection | No config needed | Must declare |
| Best for | Long-running services | Serverless, CLI, short-lived |

### AOT Checklist

- Run `mvn spring-boot:process-aot` and verify no errors
- Add `@RegisterReflectionForBinding` for custom DTO classes
- Register resources in `resource-config.json`
- Use tracing agent to catch missing hints
- Test with `-P native test` before deployment
- Use Actuator's `/actuator/aottest` endpoint to validate

---

## Next Part Preview

**Part 075: Spring AI — Integrating AI into Spring Boot** — add LLM capabilities to your Spring Boot apps with Spring AI, implementing chat, RAG with vector stores, streaming responses, function calling, and building an AI-powered customer support chatbot.
