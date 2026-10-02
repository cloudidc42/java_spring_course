# Part 96: Spring Data REST

Spring Data REST auto-exports Spring Data repositories as a hypermedia-driven REST API,
discovering your resources, generating HAL links, and handling pagination, sorting, and search
out of the box. This part shows every layer from basic setup to custom controllers, projections,
event hooks, and full integration tests.

---

## 1. Dependency Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
<!-- HAL Explorer (browser-based UI for HAL APIs) -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-rest-hal-explorer</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 2. Domain Entities

```java
// src/main/java/com/example/bookstore/domain/Author.java
package com.example.bookstore.domain;

import com.fasterxml.jackson.annotation.JsonIgnore;
import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "authors")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(nullable = false)
    private String firstName;

    @NotBlank
    @Column(nullable = false)
    private String lastName;

    @Column(unique = true)
    private String email;

    private String biography;

    // Suppress this large collection from default serialisation;
    // expose it only through the /authors/{id}/books link
    @JsonIgnore
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;
}
```

```java
// src/main/java/com/example/bookstore/domain/Book.java
package com.example.bookstore.domain;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.LocalDate;

@Entity
@Table(name = "books")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(nullable = false)
    private String title;

    @Column(unique = true, nullable = false, length = 13)
    private String isbn;

    @NotNull
    @DecimalMin("0.01")
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    private LocalDate publishedDate;

    @Column(columnDefinition = "TEXT")
    private String description;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "author_id")
    private Author author;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @Column(nullable = false)
    private boolean inStock = true;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;
}
```

```java
// src/main/java/com/example/bookstore/domain/Category.java
package com.example.bookstore.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "categories")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    @Column(nullable = false, unique = true)
    private String slug;
}
```

---

## 3. @RepositoryRestResource and @RestResource

```java
// src/main/java/com/example/bookstore/repository/BookRepository.java
package com.example.bookstore.repository;

import com.example.bookstore.domain.Book;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;

import java.math.BigDecimal;
import java.util.Optional;

// path      = URL segment (/api/books instead of the default /books)
// collectionResourceRel = HAL _links key name
@RepositoryRestResource(path = "books", collectionResourceRel = "books",
                        itemResourceRel = "book")
public interface BookRepository extends JpaRepository<Book, Long> {

    // Exposed as GET /api/books/search/findByIsbn?isbn=...
    Optional<Book> findByIsbn(@Param("isbn") String isbn);

    // Exposed as GET /api/books/search/findByTitle?title=...
    Page<Book> findByTitleContainingIgnoreCase(@Param("title") String title,
                                               Pageable pageable);

    // Exposed as GET /api/books/search/findByAuthor?authorId=...
    @RestResource(path = "findByAuthor", rel = "byAuthor")
    Page<Book> findByAuthorId(@Param("authorId") Long authorId, Pageable pageable);

    // Hidden from the /search endpoint — internal use only
    @RestResource(exported = false)
    Page<Book> findByInStockFalse(Pageable pageable);

    @Query("""
           SELECT b FROM Book b
           WHERE (:minPrice IS NULL OR b.price >= :minPrice)
             AND (:maxPrice IS NULL OR b.price <= :maxPrice)
             AND (:inStock  IS NULL OR b.inStock = :inStock)
           """)
    @RestResource(path = "findByFilters", rel = "byFilters")
    Page<Book> findByFilters(@Param("minPrice") BigDecimal minPrice,
                             @Param("maxPrice") BigDecimal maxPrice,
                             @Param("inStock")  Boolean    inStock,
                             Pageable pageable);
}
```

```java
// src/main/java/com/example/bookstore/repository/AuthorRepository.java
package com.example.bookstore.repository;

import com.example.bookstore.domain.Author;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(path = "authors", collectionResourceRel = "authors")
public interface AuthorRepository extends JpaRepository<Author, Long> {

    Page<Author> findByLastNameStartingWithIgnoreCase(@Param("lastName") String lastName,
                                                      Pageable pageable);
}
```

```java
// src/main/java/com/example/bookstore/repository/CategoryRepository.java
package com.example.bookstore.repository;

import com.example.bookstore.domain.Category;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import java.util.Optional;

@RepositoryRestResource(path = "categories")
public interface CategoryRepository extends JpaRepository<Category, Long> {

    Optional<Category> findBySlug(@Param("slug") String slug);
}
```

---

## 4. Global Spring Data REST Configuration

```java
// src/main/java/com/example/bookstore/config/RestConfig.java
package com.example.bookstore.config;

import com.example.bookstore.domain.Author;
import com.example.bookstore.domain.Book;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.rest.core.config.RepositoryRestConfiguration;
import org.springframework.data.rest.core.event.ValidatingRepositoryEventListener;
import org.springframework.data.rest.webmvc.config.RepositoryRestConfigurer;
import org.springframework.validation.beanvalidation.LocalValidatorFactoryBean;
import org.springframework.web.servlet.config.annotation.CorsRegistry;

@Configuration
public class RestConfig implements RepositoryRestConfigurer {

    private final LocalValidatorFactoryBean validator;

    public RestConfig(LocalValidatorFactoryBean validator) {
        this.validator = validator;
    }

    @Override
    public void configureRepositoryRestConfiguration(RepositoryRestConfiguration config,
                                                     CorsRegistry cors) {
        // Expose entity IDs in responses (hidden by default)
        config.exposeIdsFor(Book.class, Author.class);

        // Base path for all Spring Data REST endpoints
        config.setBasePath("/api");

        // Default page size
        config.setDefaultPageSize(20);
        config.setMaxPageSize(100);

        // Use page/size/sort parameter names matching our API convention
        config.setPageParamName("page");
        config.setSizeParamName("size");
        config.setSortParamName("sort");

        // Return 201 Created instead of 200 OK for POST
        config.setReturnBodyOnCreate(true);
        config.setReturnBodyOnUpdate(true);

        cors.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS");
    }

    @Override
    public void configureValidatingRepositoryEventListener(
            ValidatingRepositoryEventListener listener) {
        listener.addValidator("beforeCreate", validator);
        listener.addValidator("beforeSave",   validator);
    }
}
```

---

## 5. Projections

Projections let you return a subset of fields (or computed fields) without exposing the full entity.

```java
// src/main/java/com/example/bookstore/projection/BookSummary.java
package com.example.bookstore.projection;

import com.example.bookstore.domain.Book;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.rest.core.config.Projection;

import java.math.BigDecimal;

// name      = the URL excerpt parameter: /api/books?projection=summary
// types     = the entity this projection applies to
@Projection(name = "summary", types = { Book.class })
public interface BookSummary {

    Long   getId();
    String getTitle();
    String getIsbn();
    BigDecimal getPrice();
    boolean isInStock();

    // Computed via SpEL — combine author first and last name
    @Value("#{target.author.firstName + ' ' + target.author.lastName}")
    String getAuthorFullName();

    // Nested projection: expose category name without loading the whole Category
    @Value("#{target.category != null ? target.category.getName() : 'Uncategorised'}")
    String getCategoryName();
}
```

```java
// src/main/java/com/example/bookstore/projection/BookDetail.java
package com.example.bookstore.projection;

import com.example.bookstore.domain.Book;
import org.springframework.data.rest.core.config.Projection;

import java.math.BigDecimal;
import java.time.LocalDate;

@Projection(name = "detail", types = { Book.class })
public interface BookDetail {

    Long   getId();
    String getTitle();
    String getIsbn();
    BigDecimal getPrice();
    String getDescription();
    LocalDate getPublishedDate();
    boolean isInStock();
    AuthorInfo getAuthor();
    CategoryInfo getCategory();

    interface AuthorInfo {
        Long   getId();
        String getFirstName();
        String getLastName();
        String getEmail();
    }

    interface CategoryInfo {
        Long   getId();
        String getName();
        String getSlug();
    }
}
```

```java
// src/main/java/com/example/bookstore/projection/AuthorSummary.java
package com.example.bookstore.projection;

import com.example.bookstore.domain.Author;
import org.springframework.data.rest.core.config.Projection;

@Projection(name = "summary", types = { Author.class })
public interface AuthorSummary {

    Long   getId();
    String getFirstName();
    String getLastName();
}
```

---

## 6. Excerpts

An excerpt projection is automatically applied to embedded items in collection resources.

```java
// src/main/java/com/example/bookstore/repository/BookRepository.java
// Add excerptProjection to the annotation:
@RepositoryRestResource(
    path                = "books",
    collectionResourceRel = "books",
    excerptProjection   = BookSummary.class   // automatically applied for collections
)
public interface BookRepository extends JpaRepository<Book, Long> { /* ... */ }
```

---

## 7. Repository Event Handlers

```java
// src/main/java/com/example/bookstore/handler/BookEventHandler.java
package com.example.bookstore.handler;

import com.example.bookstore.domain.Book;
import com.example.bookstore.service.IsbnValidationService;
import com.example.bookstore.service.SlugService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.rest.core.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RepositoryEventHandler
@RequiredArgsConstructor
@Slf4j
public class BookEventHandler {

    private final IsbnValidationService isbnValidator;
    private final SlugService slugService;

    @HandleBeforeCreate
    public void handleBeforeCreate(Book book) {
        log.debug("Before create: validating ISBN {}", book.getIsbn());
        isbnValidator.validate(book.getIsbn());
        if (book.getDescription() == null) {
            book.setDescription("No description available.");
        }
    }

    @HandleAfterCreate
    public void handleAfterCreate(Book book) {
        log.info("Book created: id={} title={}", book.getId(), book.getTitle());
        // e.g. push an event to Kafka, update a search index, send a notification
    }

    @HandleBeforeSave
    public void handleBeforeSave(Book book) {
        log.debug("Before save: book id={}", book.getId());
        // Validate stock transitions, etc.
        if (!book.isInStock() && book.getPrice().signum() == 0) {
            throw new IllegalStateException("Out-of-stock books cannot have zero price");
        }
    }

    @HandleAfterSave
    public void handleAfterSave(Book book) {
        log.info("Book updated: id={}", book.getId());
    }

    @HandleBeforeDelete
    public void handleBeforeDelete(Book book) {
        log.warn("Deleting book: id={} isbn={}", book.getId(), book.getIsbn());
    }

    @HandleAfterDelete
    public void handleAfterDelete(Book book) {
        log.info("Book deleted: id={}", book.getId());
        // Trigger search index deletion, cache invalidation, etc.
    }
}
```

```java
// src/main/java/com/example/bookstore/handler/AuthorEventHandler.java
package com.example.bookstore.handler;

import com.example.bookstore.domain.Author;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.rest.core.annotation.*;
import org.springframework.stereotype.Component;

@Component
@RepositoryEventHandler
@Slf4j
public class AuthorEventHandler {

    @HandleBeforeCreate
    public void handleBeforeCreate(Author author) {
        // Normalize email to lowercase
        if (author.getEmail() != null) {
            author.setEmail(author.getEmail().toLowerCase());
        }
    }

    @HandleBeforeSave
    public void handleBeforeSave(Author author) {
        if (author.getEmail() != null) {
            author.setEmail(author.getEmail().toLowerCase());
        }
    }
}
```

---

## 8. Custom Controllers Alongside Spring Data REST

Sometimes you need non-CRUD endpoints. Extend `RepositoryRestController` to keep Spring Data
REST's routing and content negotiation while adding your own methods.

```java
// src/main/java/com/example/bookstore/controller/BookCustomController.java
package com.example.bookstore.controller;

import com.example.bookstore.domain.Book;
import com.example.bookstore.repository.BookRepository;
import com.example.bookstore.service.BookImportService;
import com.example.bookstore.service.BookStatsService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.rest.webmvc.RepositoryRestController;
import org.springframework.hateoas.EntityModel;
import org.springframework.hateoas.server.RepresentationModelAssembler;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.Map;

@RepositoryRestController           // keeps SDR's content-type negotiation
@RequestMapping("/books")           // path is relative to the SDR base path (/api)
@RequiredArgsConstructor
public class BookCustomController {

    private final BookRepository    bookRepository;
    private final BookImportService importService;
    private final BookStatsService  statsService;

    // Custom action: mark a book as out-of-stock
    // PUT /api/books/{id}/stock  {"inStock": false}
    @PutMapping("/{id}/stock")
    public ResponseEntity<EntityModel<Book>> updateStock(
            @PathVariable Long id,
            @RequestBody Map<String, Boolean> body) {

        Book book = bookRepository.findById(id)
                .orElseThrow(() -> new BookNotFoundException(id));
        book.setInStock(body.getOrDefault("inStock", book.isInStock()));
        bookRepository.save(book);

        EntityModel<Book> model = EntityModel.of(book);
        return ResponseEntity.ok(model);
    }

    // Bulk import from CSV
    // POST /api/books/import
    @PostMapping("/import")
    public ResponseEntity<Map<String, Object>> importCsv(
            @RequestParam("file") MultipartFile file) throws Exception {
        int imported = importService.importFromCsv(file.getInputStream());
        return ResponseEntity.ok(Map.of("imported", imported));
    }

    // Statistics endpoint
    // GET /api/books/stats
    @GetMapping("/stats")
    public ResponseEntity<Map<String, Object>> stats() {
        return ResponseEntity.ok(statsService.compute());
    }
}
```

```java
// src/main/java/com/example/bookstore/controller/BookNotFoundException.java
package com.example.bookstore.controller;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(HttpStatus.NOT_FOUND)
public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(Long id) {
        super("Book not found: " + id);
    }
}
```

---

## 9. Validation Hook

```java
// src/main/java/com/example/bookstore/service/IsbnValidationService.java
package com.example.bookstore.service;

import org.springframework.stereotype.Service;

@Service
public class IsbnValidationService {

    public void validate(String isbn) {
        if (isbn == null) throw new IllegalArgumentException("ISBN is required");
        String digits = isbn.replaceAll("[- ]", "");
        if (digits.length() == 10) {
            validateIsbn10(digits);
        } else if (digits.length() == 13) {
            validateIsbn13(digits);
        } else {
            throw new IllegalArgumentException("ISBN must be 10 or 13 digits: " + isbn);
        }
    }

    private void validateIsbn10(String digits) {
        int sum = 0;
        for (int i = 0; i < 9; i++) {
            sum += (digits.charAt(i) - '0') * (10 - i);
        }
        char last = digits.charAt(9);
        sum += (last == 'X' || last == 'x') ? 10 : (last - '0');
        if (sum % 11 != 0) {
            throw new IllegalArgumentException("Invalid ISBN-10 checksum: " + digits);
        }
    }

    private void validateIsbn13(String digits) {
        int sum = 0;
        for (int i = 0; i < 12; i++) {
            sum += (digits.charAt(i) - '0') * (i % 2 == 0 ? 1 : 3);
        }
        int check = (10 - (sum % 10)) % 10;
        if (check != (digits.charAt(12) - '0')) {
            throw new IllegalArgumentException("Invalid ISBN-13 checksum: " + digits);
        }
    }
}
```

---

## 10. application.yml

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: bookstore-api

  datasource:
    url: jdbc:postgresql://localhost:5432/bookstoredb
    username: ${DB_USER:bookstore}
    password: ${DB_PASS:bookstore}

  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        default_batch_fetch_size: 25

  data:
    rest:
      base-path: /api
      default-page-size: 20
      max-page-size: 100
      return-body-on-create: true
      return-body-on-update: true

  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080
```

---

## 11. Flyway Migration

```sql
-- src/main/resources/db/migration/V1__init.sql

CREATE TABLE categories (
    id    BIGSERIAL    PRIMARY KEY,
    name  VARCHAR(100) NOT NULL UNIQUE,
    slug  VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE authors (
    id           BIGSERIAL    PRIMARY KEY,
    first_name   VARCHAR(100) NOT NULL,
    last_name    VARCHAR(100) NOT NULL,
    email        VARCHAR(255) UNIQUE,
    biography    TEXT,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE TABLE books (
    id             BIGSERIAL      PRIMARY KEY,
    title          VARCHAR(255)   NOT NULL,
    isbn           VARCHAR(13)    NOT NULL UNIQUE,
    price          NUMERIC(10,2)  NOT NULL,
    published_date DATE,
    description    TEXT,
    in_stock       BOOLEAN        NOT NULL DEFAULT TRUE,
    author_id      BIGINT         NOT NULL REFERENCES authors(id),
    category_id    BIGINT         REFERENCES categories(id),
    created_at     TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_books_author   ON books (author_id);
CREATE INDEX idx_books_category ON books (category_id);
CREATE INDEX idx_books_isbn     ON books (isbn);
```

---

## 12. Integration Tests with @AutoConfigureMockMvc

```java
// src/test/java/com/example/bookstore/BookApiIntegrationTest.java
package com.example.bookstore;

import com.example.bookstore.domain.Author;
import com.example.bookstore.domain.Book;
import com.example.bookstore.domain.Category;
import com.example.bookstore.repository.AuthorRepository;
import com.example.bookstore.repository.BookRepository;
import com.example.bookstore.repository.CategoryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@ActiveProfiles("test")
class BookApiIntegrationTest {

    @Autowired MockMvc         mockMvc;
    @Autowired BookRepository  bookRepository;
    @Autowired AuthorRepository authorRepository;
    @Autowired CategoryRepository categoryRepository;

    private Author   savedAuthor;
    private Category savedCategory;
    private Book     savedBook;

    @BeforeEach
    void setup() {
        savedAuthor = authorRepository.save(Author.builder()
                .firstName("George").lastName("Orwell")
                .email("orwell@example.com").build());

        savedCategory = categoryRepository.save(Category.builder()
                .name("Dystopian").slug("dystopian").build());

        savedBook = bookRepository.save(Book.builder()
                .title("Nineteen Eighty-Four")
                .isbn("9780451524935")
                .price(new BigDecimal("12.99"))
                .author(savedAuthor)
                .category(savedCategory)
                .inStock(true)
                .build());
    }

    @Test
    void rootDiscoveryReturnsHalLinks() throws Exception {
        mockMvc.perform(get("/api")
                        .accept(MediaType.valueOf("application/hal+json")))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$._links.books").exists())
               .andExpect(jsonPath("$._links.authors").exists())
               .andExpect(jsonPath("$._links.categories").exists());
    }

    @Test
    void getBooksReturnsPaginatedHalCollection() throws Exception {
        mockMvc.perform(get("/api/books")
                        .accept(MediaType.valueOf("application/hal+json")))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$._embedded.books", hasSize(greaterThanOrEqualTo(1))))
               .andExpect(jsonPath("$.page.size").value(20))
               .andExpect(jsonPath("$._links.self").exists());
    }

    @Test
    void getBookByIdIncludesSelfAndAuthorLinks() throws Exception {
        mockMvc.perform(get("/api/books/{id}", savedBook.getId())
                        .accept(MediaType.valueOf("application/hal+json")))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.title").value("Nineteen Eighty-Four"))
               .andExpect(jsonPath("$.isbn").value("9780451524935"))
               .andExpect(jsonPath("$._links.self").exists())
               .andExpect(jsonPath("$._links.author").exists())
               .andExpect(jsonPath("$._links.category").exists());
    }

    @Test
    void createBookReturnsCreated() throws Exception {
        String json = """
                {
                  "title": "Animal Farm",
                  "isbn": "9780451526342",
                  "price": 9.99,
                  "inStock": true,
                  "author": "/api/authors/%d",
                  "category": "/api/categories/%d"
                }
                """.formatted(savedAuthor.getId(), savedCategory.getId());

        mockMvc.perform(post("/api/books")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(json))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.title").value("Animal Farm"))
               .andExpect(jsonPath("$._links.self").exists());
    }

    @Test
    void searchByIsbnReturnsBook() throws Exception {
        mockMvc.perform(get("/api/books/search/findByIsbn")
                        .param("isbn", "9780451524935")
                        .accept(MediaType.valueOf("application/hal+json")))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.title").value("Nineteen Eighty-Four"));
    }

    @Test
    void searchByTitleReturnsPaginatedResults() throws Exception {
        mockMvc.perform(get("/api/books/search/findByTitleContainingIgnoreCase")
                        .param("title", "eighty")
                        .accept(MediaType.valueOf("application/hal+json")))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$._embedded.books", hasSize(1)));
    }

    @Test
    void projectionSummaryReturnsFlattenedFields() throws Exception {
        mockMvc.perform(get("/api/books/{id}", savedBook.getId())
                        .param("projection", "summary")
                        .accept(MediaType.valueOf("application/hal+json")))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.authorFullName").value("George Orwell"))
               .andExpect(jsonPath("$.categoryName").value("Dystopian"));
    }

    @Test
    void updateBookWithPatchPartiallyUpdatesFields() throws Exception {
        String patch = """
                { "price": 14.99 }
                """;
        mockMvc.perform(patch("/api/books/{id}", savedBook.getId())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(patch))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.price").value(14.99));
    }

    @Test
    void deleteBookReturnsNoContent() throws Exception {
        mockMvc.perform(delete("/api/books/{id}", savedBook.getId()))
               .andExpect(status().isNoContent());
    }
}
```

---

## 13. Test application.yml

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
  flyway:
    enabled: false
  data:
    rest:
      base-path: /api
```

---

## 14. HAL Explorer

Once `spring-data-rest-hal-explorer` is on the classpath, navigate to
`http://localhost:8080/api/explorer/index.html`. The explorer auto-discovers all links from
the root `_links` response and provides a UI to browse, search, and mutate resources.

---

## 15. ALPS Metadata

Spring Data REST auto-generates ALPS (Application-Level Profile Semantics) metadata that describes
your API's type system. Access it at `/api/profile/books`:

```bash
curl http://localhost:8080/api/profile/books \
     -H 'Accept: application/alps+json' | jq .
```

Sample output:

```json
{
  "version": "1.0",
  "doc": {
    "href": "https://example.org/docs/",
    "value": "Book resource"
  },
  "descriptor": [
    { "id": "class field [Book]", "name": "title",    "type": "SEMANTIC" },
    { "id": "class field [Book]", "name": "isbn",     "type": "SEMANTIC" },
    { "id": "class field [Book]", "name": "price",    "type": "SEMANTIC" },
    { "id": "class field [Book]", "name": "inStock",  "type": "SEMANTIC" },
    { "id": "get-books",          "name": "books",    "type": "SAFE",
      "rt": "#book-representation" },
    { "id": "create-books",       "name": "books",    "type": "UNSAFE" }
  ]
}
```

---

## 16. Customising JSON Output

```java
// Suppress internal fields globally via Jackson mixin
// src/main/java/com/example/bookstore/config/JacksonConfig.java
package com.example.bookstore.config;

import com.example.bookstore.domain.Book;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;

@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper mapper = builder.build();
        mapper.addMixIn(Book.class, BookMixin.class);
        return mapper;
    }

    abstract static class BookMixin {
        @JsonIgnore
        abstract Void getCreatedAt();   // hide internal audit fields from clients
        @JsonIgnore
        abstract Void getUpdatedAt();
    }
}
```

---

## Summary

| Feature | How |
|---|---|
| Expose a repository | `@RepositoryRestResource` |
| Hide a finder | `@RestResource(exported = false)` |
| Custom search path | `@RestResource(path = "...", rel = "...")` |
| Partial response | `@Projection` + `?projection=<name>` |
| Auto-applied to collections | `excerptProjection` on `@RepositoryRestResource` |
| Pre/post hooks | `@RepositoryEventHandler` + `@HandleBeforeCreate` etc. |
| Extra endpoints | `@RepositoryRestController` |
| Validation | `ValidatingRepositoryEventListener` |
| API browser | HAL Explorer at `/api/explorer` |
| Machine-readable schema | ALPS at `/api/profile/<resource>` |
| Integration test | `@SpringBootTest` + `@AutoConfigureMockMvc` |
