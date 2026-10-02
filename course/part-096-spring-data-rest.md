# Part 096: Spring Data REST

## Introduction

Spring Data REST builds on top of Spring Data repositories and automatically exposes them as hypermedia-driven REST APIs (HATEOAS). With minimal configuration, you get full CRUD endpoints, pagination, sorting, and navigable links.

---

## Project Setup

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-rest</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.data</groupId>
        <artifactId>spring-data-rest-hal-explorer</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  data:
    rest:
      base-path: /api
      default-page-size: 20
      max-page-size: 100
      sort-param-name: sort
      limit-param-name: size
      page-param-name: page
      return-body-on-create: true
      return-body-on-update: true
      detection-strategy: annotated  # Only expose @RepositoryRestResource repos
```

---

## Domain Entities

```java
// src/main/java/com/example/blog/entity/Author.java
package com.example.blog.entity;

import com.fasterxml.jackson.annotation.JsonIgnore;
import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "authors")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    @NotBlank
    @Size(min = 3, max = 50)
    private String username;

    @Column(nullable = false)
    @NotBlank
    @Email
    private String email;

    @Column(nullable = false)
    @NotBlank
    @Size(min = 2, max = 100)
    private String displayName;

    @Column(length = 500)
    @Size(max = 500)
    private String bio;

    private String avatarUrl;

    @Column(nullable = false)
    private LocalDateTime joinedAt;

    // Hide sensitive fields from REST responses
    @JsonIgnore
    private String passwordHash;

    @JsonIgnore
    private boolean locked;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JsonIgnore  // Prevent infinite recursion; use projection or link traversal instead
    @Builder.Default
    private List<Post> posts = new ArrayList<>();

    @PrePersist
    void prePersist() {
        if (joinedAt == null) joinedAt = LocalDateTime.now();
    }
}
```

```java
// src/main/java/com/example/blog/entity/Post.java
package com.example.blog.entity;

import com.fasterxml.jackson.annotation.JsonIgnore;
import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

@Entity
@Table(name = "posts")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    @NotBlank
    @Size(max = 200)
    private String title;

    @Column(nullable = false, unique = true, length = 220)
    private String slug;

    @Column(nullable = false, columnDefinition = "TEXT")
    @NotBlank
    private String content;

    @Column(length = 500)
    @Size(max = 500)
    private String summary;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    @Builder.Default
    private PostStatus status = PostStatus.DRAFT;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private Author author;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "post_tags",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    @Builder.Default
    private Set<Tag> tags = new HashSet<>();

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JsonIgnore
    @Builder.Default
    private List<Comment> comments = new ArrayList<>();

    @Column(nullable = false)
    private LocalDateTime createdAt;

    private LocalDateTime publishedAt;
    private LocalDateTime updatedAt;

    @Column(nullable = false)
    @Builder.Default
    private int viewCount = 0;

    @Column(nullable = false)
    @Builder.Default
    private int likeCount = 0;

    @PrePersist
    void prePersist() {
        if (createdAt == null) createdAt = LocalDateTime.now();
        if (slug == null) slug = generateSlug(title);
    }

    @PreUpdate
    void preUpdate() {
        updatedAt = LocalDateTime.now();
    }

    private String generateSlug(String title) {
        return title.toLowerCase()
            .replaceAll("[^a-z0-9\\s-]", "")
            .replaceAll("\\s+", "-")
            .replaceAll("-+", "-");
    }
}
```

```java
// src/main/java/com/example/blog/entity/Tag.java
package com.example.blog.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;

@Entity
@Table(name = "tags")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    @NotBlank
    @Size(max = 50)
    private String name;

    @Column(length = 200)
    private String description;

    private String color;  // hex color for UI
}
```

```java
// src/main/java/com/example/blog/entity/Comment.java
package com.example.blog.entity;

import com.fasterxml.jackson.annotation.JsonIgnore;
import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;

import java.time.LocalDateTime;

@Entity
@Table(name = "comments")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Comment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    @JsonIgnore
    private Post post;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private Author author;

    @Column(nullable = false, columnDefinition = "TEXT")
    @NotBlank
    @Size(min = 1, max = 2000)
    private String content;

    @Column(nullable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    private boolean approved;

    @PrePersist
    void prePersist() {
        if (createdAt == null) createdAt = LocalDateTime.now();
    }
}
```

```java
// src/main/java/com/example/blog/entity/PostStatus.java
package com.example.blog.entity;

public enum PostStatus {
    DRAFT, PUBLISHED, ARCHIVED, SCHEDULED
}
```

---

## Spring Data REST Repositories

```java
// src/main/java/com/example/blog/repository/PostRepository.java
package com.example.blog.repository;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;
import org.springframework.security.access.prepost.PreAuthorize;

import java.time.LocalDateTime;
import java.util.Optional;

@RepositoryRestResource(
    path = "posts",             // URL path: /api/posts
    collectionResourceRel = "posts",   // JSON key for collection
    itemResourceRel = "post"           // JSON key for single item
)
public interface PostRepository extends JpaRepository<Post, Long> {

    // Exposed as: GET /api/posts/search/findBySlug?slug=my-post
    @RestResource(path = "findBySlug", rel = "bySlug")
    Optional<Post> findBySlug(@Param("slug") String slug);

    // Exposed as: GET /api/posts/search/findByStatus?status=PUBLISHED
    @RestResource(path = "findByStatus", rel = "byStatus")
    Page<Post> findByStatus(@Param("status") PostStatus status, Pageable pageable);

    // Exposed as: GET /api/posts/search/findByAuthorUsername?username=alice
    @RestResource(path = "findByAuthorUsername", rel = "byAuthorUsername")
    Page<Post> findByAuthorUsername(@Param("username") String username, Pageable pageable);

    // Full-text search: GET /api/posts/search/search?q=spring+boot
    @RestResource(path = "search", rel = "search")
    @Query("SELECT p FROM Post p WHERE " +
           "LOWER(p.title) LIKE LOWER(CONCAT('%', :q, '%')) OR " +
           "LOWER(p.content) LIKE LOWER(CONCAT('%', :q, '%'))")
    Page<Post> search(@Param("q") String q, Pageable pageable);

    // Exposed as: GET /api/posts/search/findByTagName?name=spring
    @RestResource(path = "findByTagName", rel = "byTag")
    @Query("SELECT p FROM Post p JOIN p.tags t WHERE t.name = :name")
    Page<Post> findByTagName(@Param("name") String name, Pageable pageable);

    // Recent posts: GET /api/posts/search/recent
    @RestResource(path = "recent", rel = "recent")
    Page<Post> findByStatusOrderByPublishedAtDesc(
        @Param("status") PostStatus status, Pageable pageable);

    // Date range: GET /api/posts/search/findByDateRange?from=...&to=...
    @RestResource(path = "findByDateRange", rel = "byDateRange")
    Page<Post> findByPublishedAtBetween(
        @Param("from") LocalDateTime from,
        @Param("to") LocalDateTime to,
        Pageable pageable);

    // Hidden from REST (internal use only)
    @RestResource(exported = false)
    @Query("SELECT p FROM Post p WHERE p.status = 'DRAFT' AND p.author.id = :authorId")
    Page<Post> findDraftsByAuthor(@Param("authorId") Long authorId, Pageable pageable);

    // Security: only admins can delete
    @Override
    @PreAuthorize("hasRole('ADMIN')")
    void deleteById(Long id);
}
```

```java
// src/main/java/com/example/blog/repository/AuthorRepository.java
package com.example.blog.repository;

import com.example.blog.entity.Author;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;
import org.springframework.security.access.prepost.PreAuthorize;

import java.util.Optional;

@RepositoryRestResource(path = "authors")
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @RestResource(path = "findByUsername", rel = "byUsername")
    Optional<Author> findByUsername(@Param("username") String username);

    @RestResource(path = "findByEmail", rel = "byEmail")
    Optional<Author> findByEmail(@Param("email") String email);

    // Only admin can create/delete authors
    @Override
    @PreAuthorize("hasRole('ADMIN')")
    <S extends Author> S save(S entity);

    @Override
    @PreAuthorize("hasRole('ADMIN')")
    void deleteById(Long id);
}
```

```java
// src/main/java/com/example/blog/repository/TagRepository.java
package com.example.blog.repository;

import com.example.blog.entity.Tag;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import org.springframework.data.rest.core.annotation.RestResource;

import java.util.Optional;

@RepositoryRestResource(path = "tags")
public interface TagRepository extends JpaRepository<Tag, Long> {

    @RestResource(path = "findByName", rel = "byName")
    Optional<Tag> findByName(@Param("name") String name);
}
```

---

## Projections and Excerpts

```java
// src/main/java/com/example/blog/projection/PostSummaryProjection.java
package com.example.blog.projection;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import org.springframework.data.rest.core.config.Projection;

import java.time.LocalDateTime;

/**
 * Returns only a subset of Post fields.
 * Access via: GET /api/posts?projection=summary
 * Or: GET /api/posts/1?projection=summary
 */
@Projection(name = "summary", types = {Post.class})
public interface PostSummaryProjection {
    Long getId();
    String getTitle();
    String getSlug();
    String getSummary();
    PostStatus getStatus();
    LocalDateTime getPublishedAt();
    int getViewCount();
    int getLikeCount();
    AuthorInfo getAuthor();

    // Nested projection (Spring Data REST supports interface nesting)
    interface AuthorInfo {
        Long getId();
        String getUsername();
        String getDisplayName();
        String getAvatarUrl();
    }
}
```

```java
// src/main/java/com/example/blog/projection/PostDetailProjection.java
package com.example.blog.projection;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import com.example.blog.entity.Tag;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.rest.core.config.Projection;

import java.time.LocalDateTime;
import java.util.List;

@Projection(name = "detail", types = {Post.class})
public interface PostDetailProjection {
    Long getId();
    String getTitle();
    String getSlug();
    String getContent();
    String getSummary();
    PostStatus getStatus();
    LocalDateTime getCreatedAt();
    LocalDateTime getPublishedAt();
    LocalDateTime getUpdatedAt();
    int getViewCount();
    int getLikeCount();

    List<Tag> getTags();

    AuthorDetail getAuthor();

    // Computed field using SpEL
    @Value("#{target.tags.size()}")
    int getTagCount();

    @Value("#{target.publishedAt != null ? 'published' : 'draft'}")
    String getDisplayStatus();

    interface AuthorDetail {
        Long getId();
        String getUsername();
        String getDisplayName();
        String getBio();
        String getAvatarUrl();
    }
}
```

```java
// Register projection as excerpt (used in collection responses)
// src/main/java/com/example/blog/repository/PostRepository.java - update with excerpt
// @RepositoryRestResource(
//     path = "posts",
//     excerptProjection = PostSummaryProjection.class  // Used in collections
// )
```

---

## Event Handlers

```java
// src/main/java/com/example/blog/event/PostEventHandler.java
package com.example.blog.event;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import com.example.blog.service.SlugService;
import com.example.blog.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.rest.core.annotation.*;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Slf4j
@Component
@RepositoryEventHandler  // Tell Spring this class handles repository events
@RequiredArgsConstructor
public class PostEventHandler {

    private final SlugService slugService;
    private final NotificationService notificationService;

    @HandleBeforeCreate
    public void handleBeforeCreate(Post post) {
        log.debug("Before creating post: {}", post.getTitle());

        // Generate slug from title
        if (post.getSlug() == null || post.getSlug().isBlank()) {
            post.setSlug(slugService.generateUniqueSlug(post.getTitle()));
        }

        // Set publish time if publishing immediately
        if (post.getStatus() == PostStatus.PUBLISHED && post.getPublishedAt() == null) {
            post.setPublishedAt(LocalDateTime.now());
        }
    }

    @HandleAfterCreate
    public void handleAfterCreate(Post post) {
        log.info("Post created: {} (ID: {})", post.getTitle(), post.getId());

        if (post.getStatus() == PostStatus.PUBLISHED) {
            notificationService.notifySubscribersNewPost(post);
        }
    }

    @HandleBeforeSave  // Called on updates (PUT/PATCH)
    public void handleBeforeSave(Post post) {
        log.debug("Before saving post: {}", post.getId());

        // Auto-set published time when status changes to PUBLISHED
        if (post.getStatus() == PostStatus.PUBLISHED && post.getPublishedAt() == null) {
            post.setPublishedAt(LocalDateTime.now());
        }
    }

    @HandleAfterSave
    public void handleAfterSave(Post post) {
        log.info("Post updated: {} (ID: {})", post.getTitle(), post.getId());
    }

    @HandleBeforeDelete
    public void handleBeforeDelete(Post post) {
        log.warn("Deleting post: {} (ID: {})", post.getTitle(), post.getId());
        // Could throw exception here to prevent deletion
        if (post.getStatus() == PostStatus.PUBLISHED) {
            log.warn("Deleting a published post!");
        }
    }

    @HandleAfterDelete
    public void handleAfterDelete(Post post) {
        log.info("Post deleted: {}", post.getId());
    }
}
```

```java
// src/main/java/com/example/blog/event/CommentEventHandler.java
package com.example.blog.event;

import com.example.blog.entity.Comment;
import com.example.blog.service.ModerationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.rest.core.annotation.*;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RepositoryEventHandler
@RequiredArgsConstructor
public class CommentEventHandler {

    private final ModerationService moderationService;

    @HandleBeforeCreate
    public void handleBeforeCreate(Comment comment) {
        // Auto-moderate comments
        boolean safe = moderationService.isSafe(comment.getContent());
        comment.setApproved(safe);

        if (!safe) {
            log.warn("Comment flagged for moderation: {}", comment.getId());
        }
    }

    @HandleAfterCreate
    public void handleAfterCreate(Comment comment) {
        if (!comment.isApproved()) {
            moderationService.queueForReview(comment);
        }
    }
}
```

---

## Validation

```java
// src/main/java/com/example/blog/validation/PostValidator.java
package com.example.blog.validation;

import com.example.blog.entity.Post;
import com.example.blog.repository.PostRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;

/**
 * Custom validator for Spring Data REST.
 * Must be named: "beforeCreate{EntityName}Validator" or "beforeSave{EntityName}Validator"
 * or registered via RepositoryRestConfigurer.
 */
@Component("beforeCreatePostValidator")
@RequiredArgsConstructor
public class PostValidator implements Validator {

    private final PostRepository postRepository;

    @Override
    public boolean supports(Class<?> clazz) {
        return Post.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        Post post = (Post) target;

        if (post.getTitle() == null || post.getTitle().isBlank()) {
            errors.rejectValue("title", "title.required", "Title is required");
        } else if (post.getTitle().length() < 5) {
            errors.rejectValue("title", "title.tooShort", "Title must be at least 5 characters");
        }

        if (post.getContent() == null || post.getContent().isBlank()) {
            errors.rejectValue("content", "content.required", "Content is required");
        } else if (post.getContent().length() < 100) {
            errors.rejectValue("content", "content.tooShort",
                "Content must be at least 100 characters");
        }

        if (post.getSlug() != null && !post.getSlug().isBlank()) {
            // Check slug uniqueness
            postRepository.findBySlug(post.getSlug()).ifPresent(existing -> {
                if (!existing.getId().equals(post.getId())) {
                    errors.rejectValue("slug", "slug.duplicate",
                        "Slug already in use: " + post.getSlug());
                }
            });
        }
    }
}
```

```java
// Register validators via configuration
// src/main/java/com/example/blog/config/RestConfig.java
package com.example.blog.config;

import com.example.blog.validation.PostValidator;
import lombok.RequiredArgsConstructor;
import org.springframework.data.rest.core.event.ValidatingRepositoryEventListener;
import org.springframework.data.rest.webmvc.config.RepositoryRestConfigurer;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.config.annotation.CorsRegistry;

@Component
@RequiredArgsConstructor
public class RestConfig implements RepositoryRestConfigurer {

    private final PostValidator postValidator;

    @Override
    public void configureValidatingRepositoryEventListener(
            ValidatingRepositoryEventListener validatingListener) {
        validatingListener.addValidator("beforeCreate", postValidator);
        validatingListener.addValidator("beforeSave", postValidator);
    }

    @Override
    public void configureRepositoryRestConfiguration(
            org.springframework.data.rest.core.config.RepositoryRestConfiguration config,
            CorsRegistry cors) {

        // Expose IDs in responses (hidden by default)
        config.exposeIdsFor(
            com.example.blog.entity.Post.class,
            com.example.blog.entity.Author.class,
            com.example.blog.entity.Tag.class
        );

        // CORS configuration
        cors.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000", "https://myblog.example.com")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```

---

## Security Configuration

```java
// src/main/java/com/example/blog/config/SecurityConfig.java
package com.example.blog.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Public read access to posts, tags, authors
                .requestMatchers(HttpMethod.GET, "/api/posts/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/tags/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/authors/**").permitAll()
                // Public search endpoints
                .requestMatchers(HttpMethod.GET, "/api/**").permitAll()
                // HAL explorer
                .requestMatchers("/explorer/**").permitAll()
                // Creating posts requires authentication
                .requestMatchers(HttpMethod.POST, "/api/posts").authenticated()
                .requestMatchers(HttpMethod.PUT, "/api/posts/**").authenticated()
                .requestMatchers(HttpMethod.PATCH, "/api/posts/**").authenticated()
                .requestMatchers(HttpMethod.DELETE, "/api/posts/**").hasRole("ADMIN")
                // Comments
                .requestMatchers(HttpMethod.POST, "/api/comments").authenticated()
                .requestMatchers(HttpMethod.DELETE, "/api/comments/**").hasRole("ADMIN")
                // Admin operations
                .requestMatchers("/api/authors/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .csrf(csrf -> csrf.disable()) // Disable for API usage
            .httpBasic(org.springframework.security.config.Customizer.withDefaults());

        return http.build();
    }
}
```

---

## HATEOAS Response Structure

When you `GET /api/posts/1`, the response looks like:

```json
{
  "_links": {
    "self": { "href": "http://localhost:8080/api/posts/1" },
    "post": { "href": "http://localhost:8080/api/posts/1{?projection}", "templated": true },
    "author": { "href": "http://localhost:8080/api/posts/1/author" },
    "tags": { "href": "http://localhost:8080/api/posts/1/tags" },
    "comments": { "href": "http://localhost:8080/api/posts/1/comments" }
  },
  "id": 1,
  "title": "Getting Started with Spring Boot",
  "slug": "getting-started-with-spring-boot",
  "content": "...",
  "status": "PUBLISHED",
  "viewCount": 1234,
  "likeCount": 56,
  "createdAt": "2024-01-15T10:30:00",
  "publishedAt": "2024-01-15T11:00:00"
}
```

Collection response (`GET /api/posts`):

```json
{
  "_embedded": {
    "posts": [
      { "id": 1, "title": "...", "_links": { "self": {...} } }
    ]
  },
  "_links": {
    "self": { "href": "http://localhost:8080/api/posts{?page,size,sort}", "templated": true },
    "first": { "href": "http://localhost:8080/api/posts?page=0&size=20" },
    "next": { "href": "http://localhost:8080/api/posts?page=1&size=20" },
    "last": { "href": "http://localhost:8080/api/posts?page=4&size=20" },
    "search": { "href": "http://localhost:8080/api/posts/search" }
  },
  "page": {
    "size": 20,
    "totalElements": 95,
    "totalPages": 5,
    "number": 0
  }
}
```

---

## Custom Endpoints Beyond Repositories

```java
// src/main/java/com/example/blog/controller/PostActionsController.java
package com.example.blog.controller;

import com.example.blog.entity.Post;
import com.example.blog.repository.PostRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.rest.webmvc.PersistentEntityResourceAssembler;
import org.springframework.data.rest.webmvc.RepositoryRestController;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.web.bind.annotation.*;

/**
 * @RepositoryRestController integrates with Spring Data REST's
 * link building and projection system.
 */
@RepositoryRestController
@RequiredArgsConstructor
public class PostActionsController {

    private final PostRepository postRepository;

    @PostMapping("/posts/{id}/like")
    public ResponseEntity<?> likePost(@PathVariable Long id) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException(id));

        post.setLikeCount(post.getLikeCount() + 1);
        postRepository.save(post);

        return ResponseEntity.ok().build();
    }

    @PostMapping("/posts/{id}/publish")
    public ResponseEntity<Post> publishPost(
            @PathVariable Long id,
            PersistentEntityResourceAssembler assembler) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException(id));

        post.setStatus(com.example.blog.entity.PostStatus.PUBLISHED);
        post.setPublishedAt(java.time.LocalDateTime.now());
        Post saved = postRepository.save(post);

        return ResponseEntity.ok(saved);
    }

    @PostMapping("/posts/{id}/archive")
    public ResponseEntity<?> archivePost(@PathVariable Long id) {
        postRepository.findById(id).ifPresent(post -> {
            post.setStatus(com.example.blog.entity.PostStatus.ARCHIVED);
            postRepository.save(post);
        });
        return ResponseEntity.noContent().build();
    }
}
```

---

## Integration Test

```java
// src/test/java/com/example/blog/BlogApiIntegrationTest.java
package com.example.blog;

import com.example.blog.entity.Author;
import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import com.example.blog.repository.AuthorRepository;
import com.example.blog.repository.PostRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class BlogApiIntegrationTest {

    @Autowired MockMvc mockMvc;
    @Autowired PostRepository postRepository;
    @Autowired AuthorRepository authorRepository;

    private Author testAuthor;
    private Post testPost;

    @BeforeEach
    void setUp() {
        testAuthor = authorRepository.save(Author.builder()
            .username("testuser")
            .email("test@example.com")
            .displayName("Test User")
            .build());

        testPost = postRepository.save(Post.builder()
            .title("Test Post About Spring Boot")
            .content("This is a lengthy piece of content about Spring Boot that exceeds 100 characters.")
            .status(PostStatus.PUBLISHED)
            .author(testAuthor)
            .build());
    }

    @Test
    void getPostsReturnsPaginatedList() throws Exception {
        mockMvc.perform(get("/api/posts")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$._embedded.posts").isArray())
            .andExpect(jsonPath("$.page.totalElements").isNumber())
            .andExpect(jsonPath("$._links.self").exists());
    }

    @Test
    void getPostByIdReturnsPost() throws Exception {
        mockMvc.perform(get("/api/posts/{id}", testPost.getId())
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.title").value("Test Post About Spring Boot"))
            .andExpect(jsonPath("$._links.self").exists())
            .andExpect(jsonPath("$._links.author").exists())
            .andExpect(jsonPath("$._links.tags").exists());
    }

    @Test
    void getPostWithSummaryProjection() throws Exception {
        mockMvc.perform(get("/api/posts/{id}", testPost.getId())
                .param("projection", "summary")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.title").exists())
            .andExpect(jsonPath("$.author.displayName").exists())
            .andExpect(jsonPath("$.content").doesNotExist()); // content excluded from summary
    }

    @Test
    void searchPostsReturnsMatchingResults() throws Exception {
        mockMvc.perform(get("/api/posts/search/search")
                .param("q", "Spring Boot")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$._embedded.posts").isArray())
            .andExpect(jsonPath("$._embedded.posts[0].title",
                containsString("Spring Boot")));
    }

    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void createPostReturns201() throws Exception {
        String postJson = """
            {
                "title": "New Spring Data REST Post",
                "content": "A comprehensive guide to using Spring Data REST for building APIs automatically from repositories.",
                "status": "DRAFT",
                "author": "/api/authors/%d"
            }
            """.formatted(testAuthor.getId());

        mockMvc.perform(post("/api/posts")
                .contentType(MediaType.APPLICATION_JSON)
                .content(postJson)
                .with(csrf()))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.title").value("New Spring Data REST Post"))
            .andExpect(jsonPath("$._links.self").exists());
    }

    @Test
    void createPostWithoutAuthReturns401() throws Exception {
        mockMvc.perform(post("/api/posts")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\": \"test\", \"content\": \"test\"}")
                .with(csrf()))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void searchByStatusReturnsPublishedPosts() throws Exception {
        mockMvc.perform(get("/api/posts/search/findByStatus")
                .param("status", "PUBLISHED")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$._embedded.posts[*].status",
                everyItem(is("PUBLISHED"))));
    }
}
```

---

## Summary

| Feature | Annotation/Interface | Description |
|---|---|---|
| Expose repository | `@RepositoryRestResource` | Auto-generates CRUD endpoints |
| Custom search | `@RestResource` on method | Creates `/search/methodName` endpoint |
| Hide from REST | `@RestResource(exported = false)` | Repository method not exposed |
| Projection | `@Projection` | Subset of fields for a resource |
| Excerpt | `excerptProjection` in `@RepositoryRestResource` | Used in collection responses |
| Events | `@HandleBeforeCreate`, etc. | Lifecycle hooks |
| Custom controller | `@RepositoryRestController` | Integrates with HAL link building |
| Validation | `Validator` named `beforeCreate{Entity}Validator` | Validates before save |
| Configuration | `RepositoryRestConfigurer` | CORS, ID exposure, validators |
| Hide JSON fields | `@JsonIgnore` | Exclude from REST representation |

### Key Takeaways
- Spring Data REST is great for rapid prototyping and internal services
- `@RepositoryRestResource(excerptProjection = ...)` controls what shows in lists
- Use `@RestResource(exported = false)` to keep repository methods internal
- Event handlers replace service layer lifecycle logic
- Method security (`@PreAuthorize`) works seamlessly on repository methods
- HAL Explorer (`/explorer`) provides a browsable interface for development

---

## Next Part Preview

**Part 097: Advanced JPA and Hibernate** covers second-level caching with Ehcache, bulk operations, native query result mapping, Hibernate Filters, all three inheritance strategies, and monitoring query performance with Hibernate statistics.
