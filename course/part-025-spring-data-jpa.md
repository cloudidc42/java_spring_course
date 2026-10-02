# Part 025: Spring Data JPA

Spring Data JPA abstracts the persistence layer and reduces boilerplate code for database operations. It builds on top of JPA (Java Persistence API) and Hibernate, providing powerful repository abstractions and query generation.

---

## 1. Project Setup

### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- For production use PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### application.properties

```properties
# H2 in-memory (for development/testing)
spring.datasource.url=jdbc:h2:mem:blogdb;DB_CLOSE_DELAY=-1
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=true

# JPA/Hibernate settings
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true

# Production PostgreSQL (comment out H2 above)
# spring.datasource.url=jdbc:postgresql://localhost:5432/blogdb
# spring.datasource.username=bloguser
# spring.datasource.password=secret
# spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
# spring.jpa.hibernate.ddl-auto=validate
```

---

## 2. JPA Entity Annotations

### Basic Entity Example

```java
package com.example.blog.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity                          // Marks this class as a JPA entity
@Table(
    name = "posts",              // Table name in database
    indexes = {
        @Index(name = "idx_post_slug", columnList = "slug", unique = true),
        @Index(name = "idx_post_status", columnList = "status")
    },
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_post_slug", columnNames = {"slug"})
    }
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString(exclude = {"comments", "tags"})  // Avoid LazyInitializationException in toString
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // AUTO_INCREMENT
    private Long id;

    @Column(name = "title", nullable = false, length = 255)
    private String title;

    @Column(name = "slug", nullable = false, unique = true, length = 255)
    private String slug;

    @Column(name = "content", columnDefinition = "TEXT")  // Large text content
    private String content;

    @Column(name = "excerpt", length = 500)
    private String excerpt;

    @Enumerated(EnumType.STRING)  // Store enum as string (not ordinal)
    @Column(name = "status", nullable = false, length = 20)
    private PostStatus status = PostStatus.DRAFT;

    @Column(name = "view_count", nullable = false)
    private int viewCount = 0;

    @Column(name = "featured", nullable = false)
    private boolean featured = false;

    @Column(name = "published_at")
    private LocalDateTime publishedAt;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // Will be set up in relationships section
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}

// Enum for post status
enum PostStatus {
    DRAFT, PUBLISHED, ARCHIVED, SCHEDULED
}
```

### GeneratedValue Strategies

```java
@Entity
@Table(name = "demo_entities")
public class GeneratedValueDemo {

    // IDENTITY - database auto increment (MySQL, PostgreSQL SERIAL)
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long identityId;

    // SEQUENCE - uses database sequences (recommended for PostgreSQL)
    @Id
    @GeneratedValue(
        strategy = GenerationType.SEQUENCE,
        generator = "post_seq"
    )
    @SequenceGenerator(
        name = "post_seq",
        sequenceName = "post_sequence",
        allocationSize = 50  // Batch fetches 50 IDs at once for performance
    )
    private Long sequenceId;

    // UUID - generates UUID automatically
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private java.util.UUID uuidId;

    // Custom ID - you set it manually
    @Id
    private String customId;
}
```

---

## 3. Entity Relationships

### @OneToOne Relationship

```java
// User Profile - one user has one profile
@Entity
@Table(name = "users")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;

    // OneToOne - user owns the relationship
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;

    // OneToMany - user has many posts
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Post> posts = new ArrayList<>();

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}

@Entity
@Table(name = "user_profiles")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name", length = 100)
    private String firstName;

    @Column(name = "last_name", length = 100)
    private String lastName;

    @Column(name = "bio", columnDefinition = "TEXT")
    private String bio;

    @Column(name = "avatar_url", length = 500)
    private String avatarUrl;

    @Column(name = "website_url", length = 255)
    private String websiteUrl;

    // OneToOne - profile is owned by user
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false, unique = true)
    private User user;
}
```

### @OneToMany and @ManyToOne

```java
@Entity
@Table(name = "categories")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String name;

    @Column(unique = true, length = 100)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String description;

    // Self-referential: parent category
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Category parent;

    // Self-referential: child categories
    @OneToMany(mappedBy = "parent")
    private List<Category> children = new ArrayList<>();

    // OneToMany with post
    @OneToMany(mappedBy = "category")
    private List<Post> posts = new ArrayList<>();
}

@Entity
@Table(name = "comments")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
@ToString(exclude = {"post", "author", "replies"})
public class Comment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String content;

    @Column(name = "approved", nullable = false)
    private boolean approved = false;

    // ManyToOne - many comments to one post
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id", nullable = false)
    private Post post;

    // ManyToOne - many comments to one author
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    // Self-referential comments (replies)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_comment_id")
    private Comment parent;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Comment> replies = new ArrayList<>();

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}
```

### @ManyToMany Relationship

```java
@Entity
@Table(name = "tags")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String name;

    @Column(unique = true, length = 50)
    private String slug;

    // ManyToMany - tag side (inverse)
    @ManyToMany(mappedBy = "tags")
    private Set<Post> posts = new HashSet<>();
}

// Updated Post entity with ManyToMany
@Entity
@Table(name = "posts")
public class Post {
    // ... other fields ...

    // ManyToMany - owning side (creates join table)
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "post_tags",                          // Join table name
        joinColumns = @JoinColumn(name = "post_id"),         // FK to Post
        inverseJoinColumns = @JoinColumn(name = "tag_id")    // FK to Tag
    )
    private Set<Tag> tags = new HashSet<>();

    // OneToMany - post has many comments
    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("createdAt DESC")  // Order by field
    private List<Comment> comments = new ArrayList<>();

    // Helper methods for bidirectional relationships
    public void addComment(Comment comment) {
        comments.add(comment);
        comment.setPost(this);
    }

    public void removeComment(Comment comment) {
        comments.remove(comment);
        comment.setPost(null);
    }

    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getPosts().add(this);
    }

    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getPosts().remove(this);
    }
}
```

### Many-to-Many with Extra Columns (Join Entity)

```java
// When the join table needs extra columns, create a join entity
@Entity
@Table(name = "post_authors")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor
public class PostAuthor {

    @EmbeddedId
    private PostAuthorId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("postId")
    private Post post;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("userId")
    private User user;

    @Enumerated(EnumType.STRING)
    @Column(name = "role", length = 20)
    private AuthorRole role;  // PRIMARY, CO_AUTHOR, EDITOR

    @Column(name = "contribution_percentage")
    private Integer contributionPercentage;
}

@Embeddable
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @EqualsAndHashCode
public class PostAuthorId implements Serializable {
    @Column(name = "post_id")
    private Long postId;

    @Column(name = "user_id")
    private Long userId;
}

enum AuthorRole {
    PRIMARY, CO_AUTHOR, EDITOR
}
```

---

## 4. Spring Data Repositories

### JpaRepository Interface

```java
package com.example.blog.repository;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

// JpaRepository<Entity, PrimaryKeyType>
// Provides: save, findById, findAll, delete, count, existsById, etc.
@Repository
public interface PostRepository extends JpaRepository<Post, Long>,
        JpaSpecificationExecutor<Post> {  // For Specifications (Part 028)

    // Derived query methods (auto-generated from method name)
    Optional<Post> findBySlug(String slug);

    List<Post> findByStatus(PostStatus status);

    List<Post> findByAuthorId(Long authorId);

    List<Post> findByFeaturedTrue();

    List<Post> findByCategoryId(Long categoryId);

    boolean existsBySlug(String slug);

    long countByStatus(PostStatus status);

    // Find published posts ordered by published date
    List<Post> findByStatusOrderByPublishedAtDesc(PostStatus status);

    // Find by title containing (case insensitive)
    List<Post> findByTitleContainingIgnoreCase(String title);

    // Find published after date
    List<Post> findByPublishedAtAfter(LocalDateTime date);

    // Find by multiple statuses
    List<Post> findByStatusIn(List<PostStatus> statuses);

    // Find by author and status
    List<Post> findByAuthorIdAndStatus(Long authorId, PostStatus status);
}
```

### UserRepository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByUsername(String username);

    Optional<User> findByEmail(String email);

    boolean existsByUsername(String username);

    boolean existsByEmail(String email);

    List<User> findByUsernameContainingIgnoreCase(String username);
}
```

### TagRepository

```java
@Repository
public interface TagRepository extends JpaRepository<Tag, Long> {

    Optional<Tag> findBySlug(String slug);

    List<Tag> findByNameIn(List<String> names);

    List<Tag> findByNameContainingIgnoreCase(String name);
}
```

### CategoryRepository

```java
@Repository
public interface CategoryRepository extends JpaRepository<Category, Long> {

    Optional<Category> findBySlug(String slug);

    List<Category> findByParentIsNull();  // Top-level categories

    List<Category> findByParentId(Long parentId);
}
```

### CommentRepository

```java
@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {

    List<Comment> findByPostIdAndParentIsNull(Long postId);  // Top-level comments

    List<Comment> findByPostIdAndApprovedTrue(Long postId);

    long countByPostId(Long postId);

    long countByPostIdAndApprovedTrue(Long postId);
}
```

---

## 5. Derived Query Methods

Spring Data JPA generates queries from method names automatically:

```java
@Repository
public interface DerivedQueryExamples extends JpaRepository<Post, Long> {

    // === Find By === (returns list or optional)
    List<Post> findByTitle(String title);
    Optional<Post> findBySlug(String slug);  // Unique field - Optional

    // === Comparison operators ===
    List<Post> findByViewCountGreaterThan(int count);
    List<Post> findByViewCountGreaterThanEqual(int count);
    List<Post> findByViewCountLessThan(int count);
    List<Post> findByViewCountBetween(int min, int max);
    List<Post> findByPublishedAtBefore(LocalDateTime date);
    List<Post> findByPublishedAtAfter(LocalDateTime date);

    // === String operators ===
    List<Post> findByTitleContaining(String keyword);
    List<Post> findByTitleContainingIgnoreCase(String keyword);
    List<Post> findByTitleStartingWith(String prefix);
    List<Post> findByTitleEndingWith(String suffix);
    List<Post> findByTitleLike(String pattern);  // % wildcard

    // === Boolean operators ===
    List<Post> findByFeaturedTrue();
    List<Post> findByFeaturedFalse();

    // === Null checks ===
    List<Post> findByPublishedAtIsNull();
    List<Post> findByPublishedAtIsNotNull();
    List<Post> findByCategoryIsNull();

    // === Collection operators ===
    List<Post> findByStatusIn(Collection<PostStatus> statuses);
    List<Post> findByStatusNotIn(Collection<PostStatus> statuses);

    // === AND / OR ===
    List<Post> findByStatusAndFeaturedTrue(PostStatus status);
    List<Post> findByTitleContainingOrContentContaining(String title, String content);
    List<Post> findByStatusAndCategoryIdAndFeaturedTrue(PostStatus status, Long categoryId);

    // === Sorting ===
    List<Post> findByStatusOrderByCreatedAtDesc(PostStatus status);
    List<Post> findByStatusOrderByViewCountDescTitleAsc(PostStatus status);

    // === Limiting ===
    List<Post> findTop5ByStatusOrderByViewCountDesc(PostStatus status);
    List<Post> findFirst10ByOrderByCreatedAtDesc();
    Optional<Post> findFirstByStatusOrderByPublishedAtDesc(PostStatus status);

    // === Distinct ===
    List<Post> findDistinctByTagsName(String tagName);

    // === Count / Exists / Delete ===
    long countByStatus(PostStatus status);
    boolean existsBySlug(String slug);
    void deleteByStatus(PostStatus status);  // @Transactional required
    long deleteByCreatedAtBefore(LocalDateTime date);

    // === Nested property ===
    List<Post> findByAuthorUsername(String username);         // Post -> author -> username
    List<Post> findByCategorySlug(String slug);              // Post -> category -> slug
    List<Post> findByTagsName(String tagName);               // Post -> tags -> name (collection)
    List<Post> findByAuthorProfileFirstName(String name);    // Post -> author -> profile -> firstName
}
```

---

## 6. @Query Annotation

### JPQL Queries

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // JPQL - uses entity and field names (not table/column names)
    @Query("SELECT p FROM Post p WHERE p.status = 'PUBLISHED' ORDER BY p.publishedAt DESC")
    List<Post> findAllPublished();

    // Positional parameters
    @Query("SELECT p FROM Post p WHERE p.title LIKE %?1% OR p.content LIKE %?1%")
    List<Post> searchByKeyword(String keyword);

    // Named parameters (preferred)
    @Query("SELECT p FROM Post p WHERE p.status = :status AND p.category.id = :categoryId")
    List<Post> findByStatusAndCategory(@Param("status") PostStatus status,
                                       @Param("categoryId") Long categoryId);

    // JOIN FETCH to avoid N+1 problem
    @Query("SELECT DISTINCT p FROM Post p " +
           "LEFT JOIN FETCH p.tags " +
           "LEFT JOIN FETCH p.author " +
           "WHERE p.status = :status")
    List<Post> findPublishedWithTagsAndAuthor(@Param("status") PostStatus status);

    // Aggregate functions
    @Query("SELECT COUNT(p) FROM Post p WHERE p.author.id = :authorId AND p.status = 'PUBLISHED'")
    long countPublishedByAuthor(@Param("authorId") Long authorId);

    // GROUP BY with aggregate
    @Query("SELECT p.category.name, COUNT(p) FROM Post p " +
           "WHERE p.status = 'PUBLISHED' " +
           "GROUP BY p.category.name " +
           "ORDER BY COUNT(p) DESC")
    List<Object[]> countPostsByCategory();

    // Subquery
    @Query("SELECT p FROM Post p WHERE p.viewCount = " +
           "(SELECT MAX(p2.viewCount) FROM Post p2 WHERE p2.status = 'PUBLISHED')")
    Optional<Post> findMostViewedPost();

    // Collection parameter
    @Query("SELECT p FROM Post p JOIN p.tags t WHERE t.name IN :tagNames")
    List<Post> findByTagNames(@Param("tagNames") List<String> tagNames);

    // Date range
    @Query("SELECT p FROM Post p WHERE p.publishedAt BETWEEN :start AND :end")
    List<Post> findByPublishedAtBetween(@Param("start") LocalDateTime start,
                                        @Param("end") LocalDateTime end);

    // With Pageable parameter
    @Query("SELECT p FROM Post p WHERE p.status = :status ORDER BY p.publishedAt DESC")
    Page<Post> findByStatusPaged(@Param("status") PostStatus status, Pageable pageable);

    // Projection (returning specific fields)
    @Query("SELECT p.id, p.title, p.slug, p.publishedAt FROM Post p WHERE p.status = 'PUBLISHED'")
    List<Object[]> findPublishedTitles();
}
```

### Native SQL Queries

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // Native SQL query
    @Query(value = "SELECT * FROM posts WHERE status = 'PUBLISHED' ORDER BY published_at DESC LIMIT :limit",
           nativeQuery = true)
    List<Post> findRecentPublished(@Param("limit") int limit);

    // Native query with join
    @Query(value = """
        SELECT p.*, u.username as author_name
        FROM posts p
        JOIN users u ON p.author_id = u.id
        WHERE p.status = 'PUBLISHED'
        AND p.category_id = :categoryId
        ORDER BY p.view_count DESC
        """,
           nativeQuery = true)
    List<Object[]> findPublishedByCategoryNative(@Param("categoryId") Long categoryId);

    // Native query with pagination requires countQuery
    @Query(
        value = "SELECT * FROM posts WHERE status = :status",
        countQuery = "SELECT count(*) FROM posts WHERE status = :status",
        nativeQuery = true
    )
    Page<Post> findByStatusNative(@Param("status") String status, Pageable pageable);

    // Full-text search (PostgreSQL specific)
    @Query(value = """
        SELECT * FROM posts
        WHERE to_tsvector('english', title || ' ' || content) @@ plainto_tsquery('english', :query)
        AND status = 'PUBLISHED'
        ORDER BY ts_rank(to_tsvector('english', title || ' ' || content), 
                         plainto_tsquery('english', :query)) DESC
        """,
           nativeQuery = true)
    List<Post> fullTextSearch(@Param("query") String query);
}
```

### Modifying Queries

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // Update query - requires @Modifying and @Transactional
    @Modifying
    @Transactional
    @Query("UPDATE Post p SET p.viewCount = p.viewCount + 1 WHERE p.id = :id")
    int incrementViewCount(@Param("id") Long id);

    @Modifying
    @Transactional
    @Query("UPDATE Post p SET p.status = :newStatus WHERE p.status = :oldStatus")
    int bulkUpdateStatus(@Param("oldStatus") PostStatus oldStatus,
                         @Param("newStatus") PostStatus newStatus);

    @Modifying
    @Transactional
    @Query("DELETE FROM Post p WHERE p.status = 'DRAFT' AND p.createdAt < :cutoffDate")
    int deleteOldDrafts(@Param("cutoffDate") LocalDateTime cutoffDate);

    // Clear persistence context after modifying (prevents stale cache issues)
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Transactional
    @Query("UPDATE Post p SET p.featured = false WHERE p.featured = true")
    int unfeaturedAll();
}
```

---

## 7. Pagination and Sorting

### Pagination with Pageable

```java
@Service
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepository;

    // Basic pagination
    public Page<Post> getPosts(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return postRepository.findAll(pageable);
    }

    // Pagination with sorting
    public Page<Post> getPostsSorted(int page, int size, String sortField, String direction) {
        Sort sort = direction.equalsIgnoreCase("asc")
            ? Sort.by(sortField).ascending()
            : Sort.by(sortField).descending();
        Pageable pageable = PageRequest.of(page, size, sort);
        return postRepository.findAll(pageable);
    }

    // Multi-field sorting
    public Page<Post> getPostsMultiSort(int page, int size) {
        Sort sort = Sort.by(
            Sort.Order.desc("featured"),      // Featured first
            Sort.Order.desc("publishedAt"),   // Then newest first
            Sort.Order.asc("title")           // Then alphabetical
        );
        Pageable pageable = PageRequest.of(page, size, sort);
        return postRepository.findByStatus(PostStatus.PUBLISHED, pageable);
    }

    // Slice - lightweight alternative (no total count)
    public Slice<Post> getPostsSlice(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        return postRepository.findSliceByStatus(PostStatus.PUBLISHED, pageable);
    }

    // Working with Page response
    public Map<String, Object> getPageResponse(int page, int size) {
        Pageable pageable = PageRequest.of(page, size);
        Page<Post> pageResult = postRepository.findAll(pageable);

        Map<String, Object> response = new HashMap<>();
        response.put("content", pageResult.getContent());        // List of items
        response.put("pageNumber", pageResult.getNumber());       // Current page (0-based)
        response.put("pageSize", pageResult.getSize());           // Items per page
        response.put("totalElements", pageResult.getTotalElements()); // Total items
        response.put("totalPages", pageResult.getTotalPages());   // Total pages
        response.put("first", pageResult.isFirst());              // Is first page?
        response.put("last", pageResult.isLast());                // Is last page?
        response.put("hasNext", pageResult.hasNext());            // Has next page?
        response.put("hasPrevious", pageResult.hasPrevious());    // Has previous page?
        return response;
    }
}
```

### Repository with Pagination

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // Returns Page<T> - includes total count (extra COUNT query)
    Page<Post> findByStatus(PostStatus status, Pageable pageable);

    // Returns Slice<T> - no total count (faster for infinite scroll)
    Slice<Post> findSliceByStatus(PostStatus status, Pageable pageable);

    // Returns List<T> with pageable (no wrapper, no count)
    List<Post> findByAuthorId(Long authorId, Pageable pageable);

    // With custom query
    @Query("SELECT p FROM Post p JOIN p.tags t WHERE t.id = :tagId ORDER BY p.publishedAt DESC")
    Page<Post> findByTagId(@Param("tagId") Long tagId, Pageable pageable);
}
```

### REST Controller with Pagination

```java
@RestController
@RequestMapping("/api/posts")
@RequiredArgsConstructor
public class PostController {

    private final PostService postService;

    @GetMapping
    public ResponseEntity<Map<String, Object>> getPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir,
            @RequestParam(required = false) String status) {

        Sort sort = sortDir.equalsIgnoreCase("asc")
            ? Sort.by(sortBy).ascending()
            : Sort.by(sortBy).descending();
        Pageable pageable = PageRequest.of(page, size, sort);

        Page<Post> posts = (status != null)
            ? postService.findByStatus(PostStatus.valueOf(status.toUpperCase()), pageable)
            : postService.findAll(pageable);

        Map<String, Object> response = new LinkedHashMap<>();
        response.put("data", posts.getContent());
        response.put("pagination", Map.of(
            "page", posts.getNumber(),
            "size", posts.getSize(),
            "totalElements", posts.getTotalElements(),
            "totalPages", posts.getTotalPages(),
            "hasNext", posts.hasNext(),
            "hasPrevious", posts.hasPrevious()
        ));

        return ResponseEntity.ok(response);
    }
}
```

---

## 8. @Transactional

```java
@Service
@Transactional(readOnly = true)  // Default: all methods are read-only
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepository;
    private final CommentRepository commentRepository;
    private final TagRepository tagRepository;

    // Read operation - inherits readOnly = true
    public Optional<Post> findById(Long id) {
        return postRepository.findById(id);
    }

    // Write operation - override with readOnly = false
    @Transactional  // readOnly defaults to false
    public Post createPost(Post post) {
        return postRepository.save(post);
    }

    @Transactional(
        propagation = Propagation.REQUIRED,        // Join existing or create new transaction
        isolation = Isolation.READ_COMMITTED,       // Prevent dirty reads
        rollbackFor = Exception.class,             // Rollback on any exception
        noRollbackFor = BusinessException.class    // Don't rollback on this exception
    )
    public Post publishPost(Long postId) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + postId));

        if (post.getStatus() == PostStatus.PUBLISHED) {
            throw new BusinessException("Post already published");
        }

        post.setStatus(PostStatus.PUBLISHED);
        post.setPublishedAt(LocalDateTime.now());
        return postRepository.save(post);
    }

    // Nested transaction with REQUIRES_NEW - runs in separate transaction
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAction(String action, Long entityId) {
        // This transaction commits independently
        // Even if outer transaction rolls back, this will commit
        actionLogRepository.save(new ActionLog(action, entityId, LocalDateTime.now()));
    }

    // MANDATORY - must run inside an existing transaction
    @Transactional(propagation = Propagation.MANDATORY)
    protected void validatePostForPublish(Post post) {
        if (post.getTitle() == null || post.getTitle().isBlank()) {
            throw new ValidationException("Post title is required");
        }
        if (post.getContent() == null || post.getContent().isBlank()) {
            throw new ValidationException("Post content is required");
        }
    }
}

// Propagation types:
// REQUIRED (default) - join existing or create new
// REQUIRES_NEW - always create new, suspend existing
// NESTED - nested savepoint in same transaction
// SUPPORTS - join if exists, non-transactional if not
// NOT_SUPPORTED - non-transactional, suspend existing
// NEVER - non-transactional, throw if existing
// MANDATORY - must join existing, throw if not

// Isolation levels:
// DEFAULT - database default
// READ_UNCOMMITTED - dirty reads allowed (fastest, least safe)
// READ_COMMITTED - no dirty reads (PostgreSQL default)
// REPEATABLE_READ - no non-repeatable reads
// SERIALIZABLE - no phantom reads (slowest, safest)
```

---

## 9. JPA Auditing

```java
// Enable auditing in configuration
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            // Get current user from Spring Security
            return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .filter(auth -> auth.isAuthenticated())
                .map(Authentication::getName);
        };
    }
}

// Base entity with audit fields
@MappedSuperclass  // Not a table itself; fields inherited by subclasses
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class BaseAuditEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(name = "created_by", length = 100, updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by", length = 100)
    private String updatedBy;
}

// Entity using base audit
@Entity
@Table(name = "posts")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Post extends BaseAuditEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    // Other fields...
    // No need for createdAt, updatedAt, createdBy, updatedBy - inherited
}

// Version field for optimistic locking (covered in Part 028)
@Entity
@Table(name = "posts")
public class PostWithVersion extends BaseAuditEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private Long version;  // Automatically managed by JPA

    private String title;
}
```

---

## 10. Complete Blog System Example

### Domain Model

```java
// Complete Post entity with all relationships
@Entity
@Table(name = "posts")
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
@ToString(exclude = {"comments", "tags", "author", "category"})
@EqualsAndHashCode(of = "id")
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 255)
    private String title;

    @Column(nullable = false, unique = true, length = 255)
    private String slug;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Column(length = 500)
    private String excerpt;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    @Builder.Default
    private PostStatus status = PostStatus.DRAFT;

    @Column(nullable = false)
    @Builder.Default
    private int viewCount = 0;

    @Column(nullable = false)
    @Builder.Default
    private boolean featured = false;

    @Column
    private LocalDateTime publishedAt;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column
    private LocalDateTime updatedAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "post_tags",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    @Builder.Default
    private Set<Tag> tags = new HashSet<>();

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("createdAt DESC")
    @Builder.Default
    private List<Comment> comments = new ArrayList<>();
}
```

### Service Layer

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class PostService {

    private final PostRepository postRepository;
    private final UserRepository userRepository;
    private final CategoryRepository categoryRepository;
    private final TagRepository tagRepository;
    private final SlugUtils slugUtils;

    public Page<Post> findAllPublished(Pageable pageable) {
        return postRepository.findByStatus(PostStatus.PUBLISHED, pageable);
    }

    public Optional<Post> findBySlug(String slug) {
        return postRepository.findBySlug(slug);
    }

    public Page<Post> findByCategory(Long categoryId, Pageable pageable) {
        return postRepository.findByCategoryIdAndStatus(categoryId, PostStatus.PUBLISHED, pageable);
    }

    public Page<Post> findByTag(String tagSlug, Pageable pageable) {
        return postRepository.findByTagSlugAndStatus(tagSlug, PostStatus.PUBLISHED, pageable);
    }

    public Page<Post> search(String keyword, Pageable pageable) {
        return postRepository.searchPublished(keyword, pageable);
    }

    @Transactional
    public Post createPost(CreatePostRequest request, String authorUsername) {
        User author = userRepository.findByUsername(authorUsername)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + authorUsername));

        String slug = slugUtils.generateUniqueSlug(request.getTitle(), postRepository::existsBySlug);

        Category category = null;
        if (request.getCategoryId() != null) {
            category = categoryRepository.findById(request.getCategoryId())
                .orElseThrow(() -> new CategoryNotFoundException("Category not found"));
        }

        Set<Tag> tags = new HashSet<>();
        if (request.getTagIds() != null && !request.getTagIds().isEmpty()) {
            tags = new HashSet<>(tagRepository.findAllById(request.getTagIds()));
        }

        Post post = Post.builder()
            .title(request.getTitle())
            .slug(slug)
            .content(request.getContent())
            .excerpt(request.getExcerpt())
            .status(PostStatus.DRAFT)
            .author(author)
            .category(category)
            .tags(tags)
            .build();

        return postRepository.save(post);
    }

    @Transactional
    public Post updatePost(Long postId, UpdatePostRequest request, String currentUsername) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + postId));

        if (!post.getAuthor().getUsername().equals(currentUsername)) {
            throw new AccessDeniedException("You can only edit your own posts");
        }

        post.setTitle(request.getTitle());
        post.setContent(request.getContent());
        post.setExcerpt(request.getExcerpt());

        if (request.getCategoryId() != null) {
            Category category = categoryRepository.findById(request.getCategoryId())
                .orElseThrow(() -> new CategoryNotFoundException("Category not found"));
            post.setCategory(category);
        }

        if (request.getTagIds() != null) {
            Set<Tag> tags = new HashSet<>(tagRepository.findAllById(request.getTagIds()));
            post.setTags(tags);
        }

        return postRepository.save(post);
    }

    @Transactional
    public Post publishPost(Long postId) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + postId));

        if (post.getStatus() == PostStatus.PUBLISHED) {
            throw new IllegalStateException("Post is already published");
        }

        post.setStatus(PostStatus.PUBLISHED);
        post.setPublishedAt(LocalDateTime.now());
        return postRepository.save(post);
    }

    @Transactional
    public void incrementViewCount(Long postId) {
        postRepository.incrementViewCount(postId);
    }

    @Transactional
    public void deletePost(Long postId, String currentUsername) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + postId));

        if (!post.getAuthor().getUsername().equals(currentUsername)) {
            throw new AccessDeniedException("You can only delete your own posts");
        }

        postRepository.delete(post);
    }
}
```

### Full Repository with All Query Types

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long>,
        JpaSpecificationExecutor<Post> {

    // Derived queries
    Optional<Post> findBySlug(String slug);
    boolean existsBySlug(String slug);
    Page<Post> findByStatus(PostStatus status, Pageable pageable);
    Page<Post> findByCategoryIdAndStatus(Long categoryId, PostStatus status, Pageable pageable);

    // JPQL with JOIN
    @Query("""
        SELECT DISTINCT p FROM Post p
        JOIN p.tags t
        WHERE t.slug = :tagSlug AND p.status = :status
        ORDER BY p.publishedAt DESC
        """)
    Page<Post> findByTagSlugAndStatus(@Param("tagSlug") String tagSlug,
                                      @Param("status") PostStatus status,
                                      Pageable pageable);

    // Full-text search
    @Query("""
        SELECT p FROM Post p
        WHERE p.status = 'PUBLISHED'
        AND (LOWER(p.title) LIKE LOWER(CONCAT('%', :keyword, '%'))
             OR LOWER(p.content) LIKE LOWER(CONCAT('%', :keyword, '%')))
        """)
    Page<Post> searchPublished(@Param("keyword") String keyword, Pageable pageable);

    // View count increment
    @Modifying
    @Transactional
    @Query("UPDATE Post p SET p.viewCount = p.viewCount + 1 WHERE p.id = :id")
    int incrementViewCount(@Param("id") Long id);

    // Featured posts
    @Query("""
        SELECT p FROM Post p
        LEFT JOIN FETCH p.tags
        LEFT JOIN FETCH p.author a
        LEFT JOIN FETCH a.profile
        WHERE p.featured = true AND p.status = 'PUBLISHED'
        ORDER BY p.publishedAt DESC
        """)
    List<Post> findFeaturedWithDetails();

    // Statistics query
    @Query("""
        SELECT new com.example.blog.dto.CategoryPostCount(p.category.name, COUNT(p))
        FROM Post p
        WHERE p.status = 'PUBLISHED' AND p.category IS NOT NULL
        GROUP BY p.category.name
        ORDER BY COUNT(p) DESC
        """)
    List<CategoryPostCount> countPublishedPostsByCategory();

    // Posts by author with pagination
    @Query(value = """
        SELECT p FROM Post p
        WHERE p.author.username = :username
        ORDER BY p.createdAt DESC
        """)
    Page<Post> findByAuthorUsername(@Param("username") String username, Pageable pageable);
}
```

### DTO Classes

```java
// Request DTOs
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class CreatePostRequest {
    @NotBlank(message = "Title is required")
    @Size(max = 255)
    private String title;

    @NotBlank(message = "Content is required")
    private String content;

    @Size(max = 500)
    private String excerpt;

    private Long categoryId;
    private Set<Long> tagIds;
}

@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class UpdatePostRequest {
    @NotBlank(message = "Title is required")
    @Size(max = 255)
    private String title;

    @NotBlank(message = "Content is required")
    private String content;

    @Size(max = 500)
    private String excerpt;

    private Long categoryId;
    private Set<Long> tagIds;
}

// Response DTOs
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class PostSummaryDTO {
    private Long id;
    private String title;
    private String slug;
    private String excerpt;
    private String authorUsername;
    private String categoryName;
    private List<String> tagNames;
    private int viewCount;
    private boolean featured;
    private LocalDateTime publishedAt;

    public static PostSummaryDTO from(Post post) {
        return PostSummaryDTO.builder()
            .id(post.getId())
            .title(post.getTitle())
            .slug(post.getSlug())
            .excerpt(post.getExcerpt())
            .authorUsername(post.getAuthor().getUsername())
            .categoryName(post.getCategory() != null ? post.getCategory().getName() : null)
            .tagNames(post.getTags().stream().map(Tag::getName).toList())
            .viewCount(post.getViewCount())
            .featured(post.isFeatured())
            .publishedAt(post.getPublishedAt())
            .build();
    }
}

// Projection DTO for statistics
public record CategoryPostCount(String categoryName, Long postCount) {}
```

### Slug Utility

```java
@Component
public class SlugUtils {

    public String toSlug(String title) {
        if (title == null) return "";
        return title.toLowerCase()
            .replaceAll("[^a-z0-9\\s-]", "")   // Remove special chars
            .replaceAll("\\s+", "-")            // Replace spaces with hyphens
            .replaceAll("-+", "-")              // Remove duplicate hyphens
            .replaceAll("^-|-$", "");           // Trim leading/trailing hyphens
    }

    public String generateUniqueSlug(String title, Predicate<String> existsCheck) {
        String baseSlug = toSlug(title);
        String slug = baseSlug;
        int counter = 1;
        while (existsCheck.test(slug)) {
            slug = baseSlug + "-" + counter++;
        }
        return slug;
    }
}
```

### Data Initialization

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DataInitializer implements ApplicationRunner {

    private final UserRepository userRepository;
    private final CategoryRepository categoryRepository;
    private final TagRepository tagRepository;
    private final PostRepository postRepository;
    private final PasswordEncoder passwordEncoder;

    @Override
    @Transactional
    public void run(ApplicationArguments args) {
        if (userRepository.count() > 0) {
            log.info("Data already initialized, skipping...");
            return;
        }

        log.info("Initializing blog data...");

        // Create users
        User admin = User.builder()
            .username("admin")
            .email("admin@example.com")
            .password(passwordEncoder.encode("password"))
            .build();
        userRepository.save(admin);

        User author = User.builder()
            .username("jsmith")
            .email("jsmith@example.com")
            .password(passwordEncoder.encode("password"))
            .build();
        userRepository.save(author);

        // Create categories
        Category javaCategory = categoryRepository.save(
            Category.builder().name("Java").slug("java").build()
        );
        Category springCategory = categoryRepository.save(
            Category.builder().name("Spring").slug("spring").parent(javaCategory).build()
        );

        // Create tags
        Tag springTag = tagRepository.save(Tag.builder().name("spring").slug("spring").build());
        Tag jpaTag = tagRepository.save(Tag.builder().name("jpa").slug("jpa").build());
        Tag tutorialTag = tagRepository.save(Tag.builder().name("tutorial").slug("tutorial").build());

        // Create posts
        Post post = Post.builder()
            .title("Getting Started with Spring Data JPA")
            .slug("getting-started-spring-data-jpa")
            .content("Spring Data JPA makes it easy to implement JPA-based repositories...")
            .excerpt("Learn how to use Spring Data JPA in your Spring Boot application")
            .status(PostStatus.PUBLISHED)
            .publishedAt(LocalDateTime.now())
            .author(author)
            .category(springCategory)
            .tags(Set.of(springTag, jpaTag, tutorialTag))
            .build();
        postRepository.save(post);

        log.info("Blog data initialized successfully!");
    }
}
```

---

## 11. Summary Table

| Concept | Annotation/Class | Purpose |
|---|---|---|
| Map class to table | `@Entity`, `@Table` | Define entity mapping |
| Primary key | `@Id`, `@GeneratedValue` | Identity field configuration |
| Column mapping | `@Column` | Customize column properties |
| One-to-one | `@OneToOne` | 1:1 relationship |
| One-to-many | `@OneToMany` | 1:N relationship (parent side) |
| Many-to-one | `@ManyToOne` | N:1 relationship (child side) |
| Many-to-many | `@ManyToMany`, `@JoinTable` | M:N relationship with join table |
| CRUD operations | `JpaRepository<T, ID>` | Standard data access methods |
| Auto query generation | Derived method names | `findByXxx`, `countByXxx` |
| Custom JPQL | `@Query` | JPQL queries with entity names |
| Native SQL | `@Query(nativeQuery=true)` | Raw SQL queries |
| Update/Delete | `@Modifying` | Bulk modify operations |
| Pagination | `Pageable`, `Page<T>` | Page results with metadata |
| Lightweight paging | `Slice<T>` | Paging without total count |
| Sorting | `Sort`, `Sort.by()` | Ordered results |
| Transaction | `@Transactional` | ACID transaction boundary |
| Audit dates | `@CreatedDate`, `@LastModifiedDate` | Auto-fill timestamp fields |
| Audit users | `@CreatedBy`, `@LastModifiedBy` | Auto-fill author fields |

---

## Next Part

**Part 026** covers Spring Security basics: authentication, authorization, `SecurityFilterChain`, `UserDetailsService`, BCrypt password encoding, role-based access control, and method-level security with `@PreAuthorize`.
