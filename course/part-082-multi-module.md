# Part 82: Maven/Gradle Multi-Module Projects

Multi-module builds let you split a large application into cohesive, independently compilable units
while sharing configuration, versions, and libraries across the board. This part covers every layer
from the parent POM down to incremental Gradle builds and Spring Boot module packaging.

---

## 1. Maven Parent POM Structure

The parent POM lives at the root and defines everything shared: plugin versions, dependency
management, build lifecycle, and encoding settings.

```xml
<!-- pom.xml (root parent) -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.shop</groupId>
    <artifactId>shop-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>Shop Parent</name>
    <description>Parent POM for the Shop microservices platform</description>

    <!-- Declare every child module here -->
    <modules>
        <module>shop-bom</module>
        <module>shop-common</module>
        <module>shop-api</module>
        <module>shop-service</module>
        <module>shop-persistence</module>
        <module>shop-web</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.3.0</spring-boot.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <lombok.version>1.18.32</lombok.version>
        <testcontainers.version>1.19.8</testcontainers.version>
    </properties>

    <!--
        dependencyManagement pins versions for all children but does NOT add the
        dependency unless a child explicitly declares it (without a version tag).
    -->
    <dependencyManagement>
        <dependencies>
            <!-- Import Spring Boot BOM first so its managed versions apply -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <!-- Import our own BOM (defined below) -->
            <dependency>
                <groupId>com.example.shop</groupId>
                <artifactId>shop-bom</artifactId>
                <version>${project.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
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

    <!-- Dependencies declared here are inherited by EVERY child module -->
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
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
                        <!-- Only the web module is an executable JAR -->
                        <skip>true</skip>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                                <goal>build-info</goal>
                            </goals>
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
                    <version>3.3.1</version>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <version>3.3.1</version>
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
                    <version>0.8.12</version>
                    <executions>
                        <execution>
                            <goals><goal>prepare-agent</goal></goals>
                        </execution>
                        <execution>
                            <id>report</id>
                            <phase>test</phase>
                            <goals><goal>report</goal></goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

---

## 2. Bill of Materials (BOM) Module

The BOM module centralises the version of every internal artifact so downstream projects can import
it with a single line and get consistent versions of everything.

```xml
<!-- shop-bom/pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.shop</groupId>
        <artifactId>shop-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>shop-bom</artifactId>
    <packaging>pom</packaging>
    <name>Shop BOM</name>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.example.shop</groupId>
                <artifactId>shop-common</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example.shop</groupId>
                <artifactId>shop-api</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example.shop</groupId>
                <artifactId>shop-service</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example.shop</groupId>
                <artifactId>shop-persistence</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

---

## 3. Shared Common Module

```xml
<!-- shop-common/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.shop</groupId>
        <artifactId>shop-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>shop-common</artifactId>
    <name>Shop Common</name>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
    </dependencies>
</project>
```

```java
// shop-common/src/main/java/com/example/shop/common/exception/BusinessException.java
package com.example.shop.common.exception;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {

    private final String errorCode;

    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public BusinessException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }
}
```

```java
// shop-common/src/main/java/com/example/shop/common/dto/PageResponse.java
package com.example.shop.common.dto;

import lombok.Builder;
import lombok.Value;
import java.util.List;

@Value
@Builder
public class PageResponse<T> {
    List<T> content;
    int page;
    int size;
    long totalElements;
    int totalPages;
    boolean last;

    public static <T> PageResponse<T> of(List<T> content, int page, int size, long total) {
        int totalPages = (int) Math.ceil((double) total / size);
        return PageResponse.<T>builder()
                .content(content)
                .page(page)
                .size(size)
                .totalElements(total)
                .totalPages(totalPages)
                .last(page >= totalPages - 1)
                .build();
    }
}
```

```java
// shop-common/src/main/java/com/example/shop/common/util/SlugUtils.java
package com.example.shop.common.util;

import java.text.Normalizer;
import java.util.Locale;
import java.util.regex.Pattern;

public final class SlugUtils {

    private static final Pattern NON_LATIN    = Pattern.compile("[^\\w-]");
    private static final Pattern WHITESPACE   = Pattern.compile("[\\s]");
    private static final Pattern MULTIPLE_DASH= Pattern.compile("-{2,}");

    private SlugUtils() {}

    public static String toSlug(String input) {
        String normalized = Normalizer.normalize(input, Normalizer.Form.NFD);
        String slug = NON_LATIN.matcher(
                        WHITESPACE.matcher(normalized.toLowerCase(Locale.ENGLISH)).replaceAll("-"))
                .replaceAll("");
        return MULTIPLE_DASH.matcher(slug).replaceAll("-");
    }
}
```

---

## 4. API Module (DTOs and Interfaces)

```xml
<!-- shop-api/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.shop</groupId>
        <artifactId>shop-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>shop-api</artifactId>
    <name>Shop API</name>

    <dependencies>
        <dependency>
            <groupId>com.example.shop</groupId>
            <artifactId>shop-common</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
</project>
```

```java
// shop-api/src/main/java/com/example/shop/api/product/ProductRequest.java
package com.example.shop.api.product;

import jakarta.validation.constraints.*;
import lombok.Data;
import java.math.BigDecimal;

@Data
public class ProductRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 3, max = 200)
    private String name;

    @NotBlank
    private String description;

    @NotNull
    @DecimalMin(value = "0.01", message = "Price must be positive")
    private BigDecimal price;

    @NotNull
    @Min(0)
    private Integer stockQuantity;

    @NotBlank
    private String categorySlug;
}
```

```java
// shop-api/src/main/java/com/example/shop/api/product/ProductResponse.java
package com.example.shop.api.product;

import lombok.Builder;
import lombok.Value;
import java.math.BigDecimal;
import java.time.Instant;

@Value
@Builder
public class ProductResponse {
    Long id;
    String name;
    String slug;
    String description;
    BigDecimal price;
    Integer stockQuantity;
    String categoryName;
    Instant createdAt;
    Instant updatedAt;
}
```

---

## 5. Persistence Module

```xml
<!-- shop-persistence/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.shop</groupId>
        <artifactId>shop-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>shop-persistence</artifactId>
    <name>Shop Persistence</name>

    <dependencies>
        <dependency>
            <groupId>com.example.shop</groupId>
            <artifactId>shop-common</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

```java
// shop-persistence/src/main/java/com/example/shop/persistence/entity/Product.java
package com.example.shop.persistence.entity;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "products")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String name;

    @Column(nullable = false, unique = true)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stockQuantity;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "category_id")
    private Category category;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;
}
```

```java
// shop-persistence/src/main/java/com/example/shop/persistence/repository/ProductRepository.java
package com.example.shop.persistence.repository;

import com.example.shop.persistence.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.math.BigDecimal;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {

    Optional<Product> findBySlug(String slug);

    Page<Product> findByCategorySlug(String categorySlug, Pageable pageable);

    @Query("""
           SELECT p FROM Product p
           JOIN FETCH p.category
           WHERE (:minPrice IS NULL OR p.price >= :minPrice)
             AND (:maxPrice IS NULL OR p.price <= :maxPrice)
             AND (:categorySlug IS NULL OR p.category.slug = :categorySlug)
           """)
    Page<Product> findByFilters(
            @Param("minPrice") BigDecimal minPrice,
            @Param("maxPrice") BigDecimal maxPrice,
            @Param("categorySlug") String categorySlug,
            Pageable pageable);

    boolean existsBySlug(String slug);
}
```

---

## 6. Service Module

```xml
<!-- shop-service/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.shop</groupId>
        <artifactId>shop-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>shop-service</artifactId>
    <name>Shop Service</name>

    <dependencies>
        <dependency>
            <groupId>com.example.shop</groupId>
            <artifactId>shop-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example.shop</groupId>
            <artifactId>shop-persistence</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
    </dependencies>
</project>
```

```java
// shop-service/src/main/java/com/example/shop/service/product/ProductMapper.java
package com.example.shop.service.product;

import com.example.shop.api.product.ProductRequest;
import com.example.shop.api.product.ProductResponse;
import com.example.shop.common.util.SlugUtils;
import com.example.shop.persistence.entity.Product;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface ProductMapper {

    @Mapping(target = "categoryName", source = "category.name")
    ProductResponse toResponse(Product product);

    @Mapping(target = "id",        ignore = true)
    @Mapping(target = "slug",      expression = "java(com.example.shop.common.util.SlugUtils.toSlug(request.getName()))")
    @Mapping(target = "category",  ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Product toEntity(ProductRequest request);

    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    @Mapping(target = "id",        ignore = true)
    @Mapping(target = "slug",      ignore = true)
    @Mapping(target = "category",  ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    void updateEntity(ProductRequest request, @MappingTarget Product product);
}
```

```java
// shop-service/src/main/java/com/example/shop/service/product/ProductService.java
package com.example.shop.service.product;

import com.example.shop.api.product.ProductRequest;
import com.example.shop.api.product.ProductResponse;
import com.example.shop.common.dto.PageResponse;
import com.example.shop.common.exception.BusinessException;
import com.example.shop.common.util.SlugUtils;
import com.example.shop.persistence.entity.Category;
import com.example.shop.persistence.entity.Product;
import com.example.shop.persistence.repository.CategoryRepository;
import com.example.shop.persistence.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper productMapper;

    @Cacheable(value = "products", key = "#slug")
    public ProductResponse findBySlug(String slug) {
        return productRepository.findBySlug(slug)
                .map(productMapper::toResponse)
                .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND",
                        "Product not found: " + slug));
    }

    public PageResponse<ProductResponse> findAll(int page, int size) {
        Page<Product> found = productRepository.findAll(
                PageRequest.of(page, size, Sort.by("createdAt").descending()));
        return PageResponse.of(
                found.getContent().stream().map(productMapper::toResponse).toList(),
                page, size, found.getTotalElements());
    }

    @Transactional
    @CacheEvict(value = "products", allEntries = true)
    public ProductResponse create(ProductRequest request) {
        String slug = SlugUtils.toSlug(request.getName());
        if (productRepository.existsBySlug(slug)) {
            throw new BusinessException("SLUG_CONFLICT", "Product slug already taken: " + slug);
        }
        Category category = categoryRepository.findBySlug(request.getCategorySlug())
                .orElseThrow(() -> new BusinessException("CATEGORY_NOT_FOUND",
                        "Category not found: " + request.getCategorySlug()));
        Product product = productMapper.toEntity(request);
        product.setCategory(category);
        return productMapper.toResponse(productRepository.save(product));
    }

    @Transactional
    @CacheEvict(value = "products", allEntries = true)
    public ProductResponse update(Long id, ProductRequest request) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new BusinessException("PRODUCT_NOT_FOUND",
                        "Product not found: " + id));
        productMapper.updateEntity(request, product);
        return productMapper.toResponse(productRepository.save(product));
    }

    @Transactional
    @CacheEvict(value = "products", allEntries = true)
    public void delete(Long id) {
        if (!productRepository.existsById(id)) {
            throw new BusinessException("PRODUCT_NOT_FOUND", "Product not found: " + id);
        }
        productRepository.deleteById(id);
    }
}
```

---

## 7. Web Module (Executable Spring Boot App)

```xml
<!-- shop-web/pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example.shop</groupId>
        <artifactId>shop-parent</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>

    <artifactId>shop-web</artifactId>
    <name>Shop Web</name>

    <dependencies>
        <dependency>
            <groupId>com.example.shop</groupId>
            <artifactId>shop-service</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Override parent skip=true for this module only -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <skip>false</skip>
                    <!-- Attach the original non-repackaged JAR for classpath use -->
                    <classifier>exec</classifier>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

```java
// shop-web/src/main/java/com/example/shop/web/ShopApplication.java
package com.example.shop.web;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@SpringBootApplication(scanBasePackages = "com.example.shop")
@EntityScan(basePackages = "com.example.shop.persistence.entity")
@EnableJpaRepositories(basePackages = "com.example.shop.persistence.repository")
@EnableJpaAuditing
@EnableCaching
public class ShopApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShopApplication.class, args);
    }
}
```

```java
// shop-web/src/main/java/com/example/shop/web/controller/ProductController.java
package com.example.shop.web.controller;

import com.example.shop.api.product.ProductRequest;
import com.example.shop.api.product.ProductResponse;
import com.example.shop.common.dto.PageResponse;
import com.example.shop.service.product.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public PageResponse<ProductResponse> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return productService.findAll(page, size);
    }

    @GetMapping("/{slug}")
    public ProductResponse findBySlug(@PathVariable String slug) {
        return productService.findBySlug(slug);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@Valid @RequestBody ProductRequest request) {
        return productService.create(request);
    }

    @PutMapping("/{id}")
    public ProductResponse update(@PathVariable Long id,
                                  @Valid @RequestBody ProductRequest request) {
        return productService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

---

## 8. Gradle Multi-Module with settings.gradle

```groovy
// settings.gradle (root)
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
    plugins {
        id 'org.springframework.boot'       version '3.3.0'
        id 'io.spring.dependency-management' version '1.1.5'
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
    }
    // Link to the version catalog
    versionCatalogs {
        libs {
            from(files("gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = 'shop'

include 'shop-common'
include 'shop-api'
include 'shop-persistence'
include 'shop-service'
include 'shop-web'

// Composite build: include a locally developed library as a composite
includeBuild('../shop-shared-lib') {
    dependencySubstitution {
        substitute module('com.example:shop-shared-lib') using project(':')
    }
}
```

---

## 9. Gradle Version Catalog (libs.versions.toml)

```toml
# gradle/libs.versions.toml
[versions]
spring-boot        = "3.3.0"
mapstruct          = "1.5.5.Final"
lombok             = "1.18.32"
testcontainers     = "1.19.8"
flyway             = "10.15.0"
postgresql         = "42.7.3"
jackson            = "2.17.1"

[libraries]
spring-boot-web        = { module = "org.springframework.boot:spring-boot-starter-web" }
spring-boot-jpa        = { module = "org.springframework.boot:spring-boot-starter-data-jpa" }
spring-boot-security   = { module = "org.springframework.boot:spring-boot-starter-security" }
spring-boot-actuator   = { module = "org.springframework.boot:spring-boot-starter-actuator" }
spring-boot-validation = { module = "org.springframework.boot:spring-boot-starter-validation" }
spring-boot-cache      = { module = "org.springframework.boot:spring-boot-starter-cache" }
spring-boot-test       = { module = "org.springframework.boot:spring-boot-starter-test" }
mapstruct              = { module = "org.mapstruct:mapstruct",           version.ref = "mapstruct" }
mapstruct-processor    = { module = "org.mapstruct:mapstruct-processor", version.ref = "mapstruct" }
lombok                 = { module = "org.projectlombok:lombok",          version.ref = "lombok" }
flyway-core            = { module = "org.flywaydb:flyway-core",          version.ref = "flyway" }
postgresql             = { module = "org.postgresql:postgresql",         version.ref = "postgresql" }
testcontainers-bom     = { module = "org.testcontainers:testcontainers-bom", version.ref = "testcontainers" }
testcontainers-pg      = { module = "org.testcontainers:postgresql" }
h2                     = { module = "com.h2database:h2" }

[bundles]
spring-web  = ["spring-boot-web", "spring-boot-validation"]
persistence = ["spring-boot-jpa", "flyway-core", "postgresql"]

[plugins]
spring-boot          = { id = "org.springframework.boot",        version.ref = "spring-boot" }
spring-dep-mgmt      = { id = "io.spring.dependency-management", version = "1.1.5" }
```

---

## 10. Root build.gradle

```groovy
// build.gradle (root)
plugins {
    alias(libs.plugins.spring.boot)       apply false
    alias(libs.plugins.spring.dep.mgmt)   apply false
}

// Configuration shared by every subproject
subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'

    group = 'com.example.shop'
    version = '1.0.0-SNAPSHOT'

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(21)
        }
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.boot:spring-boot-dependencies:${libs.versions.springBoot.get()}"
        }
    }

    dependencies {
        compileOnly         libs.lombok
        annotationProcessor libs.lombok
        testImplementation  libs.spring.boot.test
    }

    compileJava {
        options.compilerArgs += [
            '-Amapstruct.defaultComponentModel=spring',
            '-parameters'
        ]
    }

    test {
        useJUnitPlatform()
    }
}
```

---

## 11. Subproject build.gradle Files

```groovy
// shop-persistence/build.gradle
dependencies {
    implementation project(':shop-common')
    implementation(libs.bundles.persistence)
    annotationProcessor libs.mapstruct.processor
    testImplementation  libs.h2
    testImplementation  libs.testcontainers.pg
}
```

```groovy
// shop-service/build.gradle
dependencies {
    implementation project(':shop-api')
    implementation project(':shop-persistence')
    implementation libs.mapstruct
    annotationProcessor libs.mapstruct.processor
    annotationProcessor libs.lombok
    implementation libs.spring.boot.cache
}
```

```groovy
// shop-web/build.gradle
plugins {
    alias(libs.plugins.spring.boot)
}

dependencies {
    implementation project(':shop-service')
    implementation(libs.bundles.spring.web)
    implementation libs.spring.boot.security
    implementation libs.spring.boot.actuator
    runtimeOnly     libs.postgresql
}

// Build the executable fat JAR; other modules skip this
bootJar {
    enabled = true
    archiveClassifier = 'boot'
}
jar {
    enabled = true  // keep the plain JAR for inter-module classpath
}
```

---

## 12. Composite Builds

A composite build lets you develop a library (e.g., `shop-shared-lib`) side-by-side with its
consumer without publishing to a Maven repository every time.

```groovy
// shop-shared-lib/settings.gradle (the included build)
rootProject.name = 'shop-shared-lib'
```

```groovy
// shop-shared-lib/build.gradle
plugins {
    id 'java-library'
}

group   = 'com.example'
version = '1.0.0-SNAPSHOT'

java {
    toolchain { languageVersion = JavaLanguageVersion.of(21) }
}

dependencies {
    api 'org.slf4j:slf4j-api:2.0.13'
}
```

```groovy
// In the consumer (shop-common/build.gradle), use the substituted dependency:
dependencies {
    implementation 'com.example:shop-shared-lib'  // resolved from includeBuild
}
```

---

## 13. Incremental / Changed-Module Builds with Gradle

Gradle tracks task inputs and outputs by hash so it skips tasks whose inputs have not changed.
For CI pipelines you can combine this with `--configuration-cache` and `--build-cache`.

```bash
# Build only the changed module and its dependents
./gradlew :shop-web:bootJar --configuration-cache --build-cache

# Run tests only in affected modules (determined by --parallel dependency graph)
./gradlew test --parallel --continue

# Show what tasks will execute without running them
./gradlew :shop-service:test --dry-run

# Enable the Gradle build cache (local and remote)
# gradle.properties
org.gradle.caching=true
org.gradle.configureondemand=true
org.gradle.parallel=true
org.gradle.daemon=true
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
```

For Maven, use the `--also-make` / `--projects` flags:

```bash
# Build only shop-web and everything it depends on
mvn install -pl shop-web -am

# Skip tests in unchanged modules
mvn install -pl shop-service,shop-web -am -DskipTests=false
```

---

## 14. Packaging and Deployment

```dockerfile
# shop-web/Dockerfile — multi-stage to minimise image size
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY . .
RUN ./mvnw -pl shop-web -am -DskipTests package

# Extract layers for faster rebuilds
FROM eclipse-temurin:21-jre-alpine AS layers
WORKDIR /build
COPY --from=builder /build/shop-web/target/shop-web-1.0.0-SNAPSHOT-exec.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
WORKDIR /app
COPY --from=layers /build/dependencies/          ./
COPY --from=layers /build/spring-boot-loader/    ./
COPY --from=layers /build/snapshot-dependencies/ ./
COPY --from=layers /build/application/           ./
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB:       shopdb
      POSTGRES_USER:     shop
      POSTGRES_PASSWORD: shop
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data

  shop-web:
    build:
      context: .
      dockerfile: shop-web/Dockerfile
    depends_on: [db]
    environment:
      DB_USER: shop
      DB_PASS: shop
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/shopdb
    ports: ["8080:8080"]

volumes:
  pgdata:
```

---

## Summary

| Concept | Maven | Gradle |
|---|---|---|
| Root descriptor | `pom.xml` with `<packaging>pom</packaging>` | `settings.gradle` |
| List sub-projects | `<modules>` | `include '...'` |
| Version pinning | `<dependencyManagement>` | `dependencyManagement {}` or version catalog |
| BOM import | `<scope>import</scope>` | `mavenBom(...)` |
| Local library | (reactor) | `includeBuild(...)` |
| Incremental | `-pl ... -am` | `--build-cache --configuration-cache` |
| Executable artifact | `spring-boot-maven-plugin` | `bootJar` |

Building only changed modules in CI cuts feedback cycles from minutes to seconds on large repos.
