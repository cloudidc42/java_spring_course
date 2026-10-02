# Part 028: Advanced Spring Data

This part covers advanced patterns for data access: custom repositories, specifications, QueryDSL, projections, N+1 problem solutions, soft delete, optimistic locking, caching, and database migrations.

---

## 1. Custom Repository Implementations

### The Pattern

Spring Data allows you to add custom methods to repositories by creating an interface and implementation pair.

```java
// Step 1: Define custom interface
public interface PostRepositoryCustom {
    List<Post> findPostsWithComplexCriteria(PostSearchCriteria criteria);
    void updatePostStats(Long postId, int viewIncrement);
    Map<String, Long> getPostCountByMonth(int year);
}

// Step 2: Implement the interface
@Repository
@RequiredArgsConstructor
public class PostRepositoryCustomImpl implements PostRepositoryCustom {

    // Use EntityManager directly for full JPA control
    private final EntityManager entityManager;

    // Use JdbcTemplate for native SQL when needed
    private final JdbcTemplate jdbcTemplate;

    @Override
    public List<Post> findPostsWithComplexCriteria(PostSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Post> query = cb.createQuery(Post.class);
        Root<Post> root = query.from(Post.class);

        List<Predicate> predicates = new ArrayList<>();

        if (criteria.getStatus() != null) {
            predicates.add(cb.equal(root.get("status"), criteria.getStatus()));
        }
        if (criteria.getKeyword() != null && !criteria.getKeyword().isBlank()) {
            String pattern = "%" + criteria.getKeyword().toLowerCase() + "%";
            predicates.add(cb.or(
                cb.like(cb.lower(root.get("title")), pattern),
                cb.like(cb.lower(root.get("content")), pattern)
            ));
        }
        if (criteria.getFromDate() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("publishedAt"), criteria.getFromDate()));
        }
        if (criteria.getToDate() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("publishedAt"), criteria.getToDate()));
        }
        if (criteria.getCategoryId() != null) {
            predicates.add(cb.equal(root.get("category").get("id"), criteria.getCategoryId()));
        }
        if (criteria.isFeaturedOnly()) {
            predicates.add(cb.isTrue(root.get("featured")));
        }

        query.where(predicates.toArray(new Predicate[0]));
        query.orderBy(cb.desc(root.get("publishedAt")));

        return entityManager.createQuery(query)
            .setMaxResults(criteria.getLimit())
            .setFirstResult(criteria.getOffset())
            .getResultList();
    }

    @Override
    @Transactional
    public void updatePostStats(Long postId, int viewIncrement) {
        entityManager.createQuery(
            "UPDATE Post p SET p.viewCount = p.viewCount + :increment WHERE p.id = :id"
        )
        .setParameter("increment", viewIncrement)
        .setParameter("id", postId)
        .executeUpdate();
    }

    @Override
    public Map<String, Long> getPostCountByMonth(int year) {
        String sql = """
            SELECT TO_CHAR(published_at, 'YYYY-MM') as month, COUNT(*) as count
            FROM posts
            WHERE EXTRACT(YEAR FROM published_at) = ?
            AND status = 'PUBLISHED'
            GROUP BY TO_CHAR(published_at, 'YYYY-MM')
            ORDER BY month
            """;

        Map<String, Long> result = new LinkedHashMap<>();
        jdbcTemplate.query(sql, rs -> {
            result.put(rs.getString("month"), rs.getLong("count"));
        }, year);
        return result;
    }
}

// Step 3: Extend both interfaces in the repository
@Repository
public interface PostRepository extends JpaRepository<Post, Long>,
        JpaSpecificationExecutor<Post>,
        PostRepositoryCustom {   // Add custom interface here

    // Spring Data derived methods...
    Optional<Post> findBySlug(String slug);
    Page<Post> findByStatus(PostStatus status, Pageable pageable);
}

// Search criteria DTO
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class PostSearchCriteria {
    private PostStatus status;
    private String keyword;
    private LocalDateTime fromDate;
    private LocalDateTime toDate;
    private Long categoryId;
    private boolean featuredOnly;
    private int limit = 10;
    private int offset = 0;
}
```

---

## 2. Specifications (Criteria API)

Specifications allow you to build dynamic queries in a reusable, composable way.

### Post Specifications

```java
package com.example.blog.specification;

import com.example.blog.entity.Post;
import com.example.blog.entity.PostStatus;
import jakarta.persistence.criteria.*;
import org.springframework.data.jpa.domain.Specification;

import java.time.LocalDateTime;
import java.util.List;

public class PostSpecifications {

    // Private constructor - utility class
    private PostSpecifications() {}

    public static Specification<Post> hasStatus(PostStatus status) {
        return (root, query, cb) ->
            status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<Post> isFeatured() {
        return (root, query, cb) -> cb.isTrue(root.get("featured"));
    }

    public static Specification<Post> titleContains(String keyword) {
        return (root, query, cb) ->
            keyword == null || keyword.isBlank() ? null :
                cb.like(cb.lower(root.get("title")),
                        "%" + keyword.toLowerCase() + "%");
    }

    public static Specification<Post> contentContains(String keyword) {
        return (root, query, cb) ->
            keyword == null || keyword.isBlank() ? null :
                cb.like(cb.lower(root.get("content")),
                        "%" + keyword.toLowerCase() + "%");
    }

    public static Specification<Post> titleOrContentContains(String keyword) {
        return (root, query, cb) -> {
            if (keyword == null || keyword.isBlank()) return null;
            String pattern = "%" + keyword.toLowerCase() + "%";
            return cb.or(
                cb.like(cb.lower(root.get("title")), pattern),
                cb.like(cb.lower(root.get("content")), pattern)
            );
        };
    }

    public static Specification<Post> publishedAfter(LocalDateTime date) {
        return (root, query, cb) ->
            date == null ? null : cb.greaterThanOrEqualTo(root.get("publishedAt"), date);
    }

    public static Specification<Post> publishedBefore(LocalDateTime date) {
        return (root, query, cb) ->
            date == null ? null : cb.lessThanOrEqualTo(root.get("publishedAt"), date);
    }

    public static Specification<Post> inCategory(Long categoryId) {
        return (root, query, cb) ->
            categoryId == null ? null :
                cb.equal(root.get("category").get("id"), categoryId);
    }

    public static Specification<Post> byAuthor(Long authorId) {
        return (root, query, cb) ->
            authorId == null ? null :
                cb.equal(root.get("author").get("id"), authorId);
    }

    public static Specification<Post> hasTag(String tagName) {
        return (root, query, cb) -> {
            if (tagName == null || tagName.isBlank()) return null;
            // Join with tags collection
            Join<Object, Object> tagsJoin = root.join("tags", JoinType.INNER);
            query.distinct(true);  // Avoid duplicates from join
            return cb.equal(cb.lower(tagsJoin.get("name")), tagName.toLowerCase());
        };
    }

    public static Specification<Post> hasViewCountGreaterThan(int count) {
        return (root, query, cb) ->
            cb.greaterThan(root.get("viewCount"), count);
    }

    // Combined specification for "published" posts
    public static Specification<Post> isPublished() {
        return hasStatus(PostStatus.PUBLISHED)
            .and(publishedAfter(null));  // publishedAt is not null
    }
}
```

### Using Specifications

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PostSearchService {

    private final PostRepository postRepository;

    public Page<Post> search(PostSearchRequest request, Pageable pageable) {
        // Build specification dynamically
        Specification<Post> spec = Specification.where(null);

        if (request.getStatus() != null) {
            spec = spec.and(PostSpecifications.hasStatus(request.getStatus()));
        }
        if (request.getKeyword() != null) {
            spec = spec.and(PostSpecifications.titleOrContentContains(request.getKeyword()));
        }
        if (request.getFromDate() != null) {
            spec = spec.and(PostSpecifications.publishedAfter(request.getFromDate()));
        }
        if (request.getToDate() != null) {
            spec = spec.and(PostSpecifications.publishedBefore(request.getToDate()));
        }
        if (request.getCategoryId() != null) {
            spec = spec.and(PostSpecifications.inCategory(request.getCategoryId()));
        }
        if (request.getAuthorId() != null) {
            spec = spec.and(PostSpecifications.byAuthor(request.getAuthorId()));
        }
        if (request.getTagName() != null) {
            spec = spec.and(PostSpecifications.hasTag(request.getTagName()));
        }
        if (request.isFeaturedOnly()) {
            spec = spec.and(PostSpecifications.isFeatured());
        }

        return postRepository.findAll(spec, pageable);
    }

    // Pre-built for common queries
    public Page<Post> findPublishedPosts(Pageable pageable) {
        Specification<Post> spec = PostSpecifications.hasStatus(PostStatus.PUBLISHED);
        return postRepository.findAll(spec, pageable);
    }

    public Page<Post> findFeaturedPublishedPosts(Pageable pageable) {
        Specification<Post> spec = PostSpecifications.hasStatus(PostStatus.PUBLISHED)
            .and(PostSpecifications.isFeatured());
        return postRepository.findAll(spec, pageable);
    }
}
```

### Specifications for Complex Joins

```java
public class UserSpecifications {

    public static Specification<User> hasPostedInCategory(Long categoryId) {
        return (root, query, cb) -> {
            // User -> posts -> category
            Join<User, Post> postsJoin = root.join("posts", JoinType.INNER);
            query.distinct(true);
            return cb.equal(postsJoin.get("category").get("id"), categoryId);
        };
    }

    public static Specification<User> hasMinimumPosts(long minPosts) {
        return (root, query, cb) -> {
            // Subquery to count posts
            Subquery<Long> subquery = query.subquery(Long.class);
            Root<Post> postRoot = subquery.from(Post.class);
            subquery.select(cb.count(postRoot));
            subquery.where(
                cb.equal(postRoot.get("author"), root),
                cb.equal(postRoot.get("status"), PostStatus.PUBLISHED)
            );
            return cb.greaterThanOrEqualTo(subquery, minPosts);
        };
    }
}
```

---

## 3. Projections

Projections let you fetch only specific fields instead of full entities.

### Interface-Based Projections

```java
// Closed projection - fetch only specified fields
public interface PostSummaryProjection {
    Long getId();
    String getTitle();
    String getSlug();
    String getExcerpt();
    LocalDateTime getPublishedAt();
    int getViewCount();

    // Nested projection
    AuthorProjection getAuthor();

    interface AuthorProjection {
        String getUsername();
    }

    // Computed property using SpEL
    @Value("#{target.title + ' by ' + target.author.username}")
    String getTitleWithAuthor();
}

// Open projection - loads full entity, then filters (less efficient)
public interface PostOpenProjection {
    String getTitle();

    @Value("#{target.title} - #{target.viewCount} views")
    String getTitleWithViews();
}
```

### Class-Based Projections (DTO Projections)

```java
// Constructor expression projection
public class PostSummaryDTO {
    private final Long id;
    private final String title;
    private final String slug;
    private final String authorUsername;
    private final LocalDateTime publishedAt;
    private final int viewCount;

    // Constructor matching JPQL constructor expression
    public PostSummaryDTO(Long id, String title, String slug,
                          String authorUsername, LocalDateTime publishedAt, int viewCount) {
        this.id = id;
        this.title = title;
        this.slug = slug;
        this.authorUsername = authorUsername;
        this.publishedAt = publishedAt;
        this.viewCount = viewCount;
    }

    // Getters...
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public String getSlug() { return slug; }
    public String getAuthorUsername() { return authorUsername; }
    public LocalDateTime getPublishedAt() { return publishedAt; }
    public int getViewCount() { return viewCount; }
}
```

### Using Projections in Repository

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // Interface projection
    List<PostSummaryProjection> findByStatus(PostStatus status);

    // Class projection via JPQL constructor expression
    @Query("SELECT new com.example.blog.dto.PostSummaryDTO(" +
           "p.id, p.title, p.slug, p.author.username, p.publishedAt, p.viewCount) " +
           "FROM Post p WHERE p.status = 'PUBLISHED' ORDER BY p.publishedAt DESC")
    List<PostSummaryDTO> findPublishedSummaries();

    // Dynamic projection - pass the desired type at call time
    <T> List<T> findByAuthorId(Long authorId, Class<T> type);

    // Dynamic projection with pageable
    <T> Page<T> findByStatus(PostStatus status, Class<T> type, Pageable pageable);
}

// Service using dynamic projections
@Service
public class PostService {

    private final PostRepository postRepository;

    // Can request different projections dynamically
    public List<PostSummaryProjection> getPostSummaries(Long authorId) {
        return postRepository.findByAuthorId(authorId, PostSummaryProjection.class);
    }

    public List<PostSummaryDTO> getPostDTOs(Long authorId) {
        return postRepository.findByAuthorId(authorId, PostSummaryDTO.class);
    }
}
```

### Record Projections (Java 16+)

```java
// Java record as projection (Spring Data 3.x supports this)
public record PostTitleSlug(String title, String slug) {}

@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    @Query("SELECT new com.example.blog.dto.PostTitleSlug(p.title, p.slug) " +
           "FROM Post p WHERE p.status = 'PUBLISHED'")
    List<PostTitleSlug> findPublishedTitleSlugs();
}
```

---

## 4. N+1 Problem and Solutions

### The N+1 Problem

```java
// PROBLEM: N+1 queries
// This runs 1 query to get posts, then N queries to get each author
@Service
public class BadPostService {

    public List<String> getPostTitlesWithAuthor() {
        List<Post> posts = postRepository.findAll();  // Query 1: SELECT * FROM posts (N rows)
        return posts.stream()
            .map(post -> post.getTitle() + " by " + post.getAuthor().getUsername())
            // Each getAuthor() triggers: SELECT * FROM users WHERE id = ?  (N more queries!)
            .toList();
    }
}
```

### Solution 1: JOIN FETCH in JPQL

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // Fetch author in same query
    @Query("SELECT p FROM Post p JOIN FETCH p.author WHERE p.status = :status")
    List<Post> findByStatusWithAuthor(@Param("status") PostStatus status);

    // Fetch multiple associations (be careful with collections - use Set or multiple queries)
    @Query("SELECT DISTINCT p FROM Post p " +
           "JOIN FETCH p.author " +
           "JOIN FETCH p.category " +
           "WHERE p.status = 'PUBLISHED'")
    List<Post> findPublishedWithAuthorAndCategory();

    // CAUTION: Don't JOIN FETCH multiple collection associations in one query
    // This causes a MultipleBagFetchException or Cartesian product issue
    // Instead, use separate queries or @EntityGraph
}
```

### Solution 2: @EntityGraph

```java
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // Named @EntityGraph defined on the entity
    @EntityGraph(attributePaths = {"author", "category"})
    List<Post> findByStatus(PostStatus status);

    // Named entity graph (defined on entity class)
    @EntityGraph(value = "Post.withTagsAndAuthor")
    Optional<Post> findById(Long id);

    // Entity graph on custom query
    @Query("SELECT p FROM Post p WHERE p.featured = true")
    @EntityGraph(attributePaths = {"author", "tags"})
    List<Post> findFeaturedWithDetails();
}

// Define named entity graphs on entity
@Entity
@NamedEntityGraphs({
    @NamedEntityGraph(
        name = "Post.summary",
        attributeNodes = {
            @NamedAttributeNode("author"),
            @NamedAttributeNode("category")
        }
    ),
    @NamedEntityGraph(
        name = "Post.withTagsAndAuthor",
        attributeNodes = {
            @NamedAttributeNode("author"),
            @NamedAttributeNode(value = "author", subgraph = "author.profile"),
            @NamedAttributeNode("tags")
        },
        subgraphs = {
            @NamedSubgraph(
                name = "author.profile",
                attributeNodes = @NamedAttributeNode("profile")
            )
        }
    )
})
public class Post {
    // entity fields...
}
```

### Solution 3: Batch Loading

```java
# application.properties
# Hibernate batch loading - load N entities in one query
spring.jpa.properties.hibernate.default_batch_fetch_size=25
spring.jpa.properties.hibernate.jdbc.batch_size=25

# This turns N+1 queries into ceil(N/25) queries
```

```java
@Entity
public class Post {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 25)  // Load 25 authors at a time
    @JoinColumn(name = "author_id")
    private User author;

    @ManyToMany(fetch = FetchType.LAZY)
    @BatchSize(size = 25)
    @JoinTable(...)
    private Set<Tag> tags;
}
```

### Solution 4: DTO with Constructor Projection

```java
// Most efficient: select only what you need
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    @Query("""
        SELECT new com.example.blog.dto.PostWithAuthorDTO(
            p.id, p.title, p.slug, p.viewCount, p.publishedAt,
            p.author.username, p.author.email
        )
        FROM Post p
        WHERE p.status = 'PUBLISHED'
        ORDER BY p.publishedAt DESC
        """)
    List<PostWithAuthorDTO> findPublishedPostsWithAuthor();
}

@Getter @AllArgsConstructor
public class PostWithAuthorDTO {
    private Long id;
    private String title;
    private String slug;
    private int viewCount;
    private LocalDateTime publishedAt;
    private String authorUsername;
    private String authorEmail;
}
```

---

## 5. Soft Delete

Soft delete marks records as deleted instead of removing them.

```java
// Base entity with soft delete
@MappedSuperclass
@Getter @Setter
public abstract class SoftDeletableEntity {

    @Column(name = "deleted", nullable = false)
    private boolean deleted = false;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @Column(name = "deleted_by", length = 100)
    private String deletedBy;
}

// Entity with soft delete
@Entity
@Table(name = "posts")
@Where(clause = "deleted = false")          // Hibernate: automatically exclude deleted
@SQLDelete(sql = "UPDATE posts SET deleted = true, deleted_at = NOW() WHERE id = ?")
// SQLDelete: intercepts delete() call and runs UPDATE instead
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Post extends SoftDeletableEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String content;
    // ...other fields
}

// Repository for soft delete
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    // @Where is applied automatically - only non-deleted posts
    List<Post> findAll();
    Optional<Post> findById(Long id);

    // To find deleted posts, use native query (bypasses @Where)
    @Query(value = "SELECT * FROM posts WHERE deleted = true", nativeQuery = true)
    List<Post> findDeleted();

    @Query(value = "SELECT * FROM posts", nativeQuery = true)
    List<Post> findAllIncludingDeleted();
}

// Service with soft delete
@Service
@RequiredArgsConstructor
@Transactional
public class PostService {

    private final PostRepository postRepository;

    // delete() calls SQLDelete UPDATE statement
    public void softDelete(Long postId, String username) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException("Post not found"));
        post.setDeletedBy(username);
        postRepository.save(post);  // Save deletedBy first
        postRepository.delete(post);  // Triggers SQLDelete
    }

    // Restore a soft-deleted post
    @Transactional
    public void restore(Long postId) {
        // Need native query since @Where excludes deleted posts
        postRepository.restoreById(postId);
    }
}

// Add restore to repository
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    @Modifying
    @Query(value = "UPDATE posts SET deleted = false, deleted_at = NULL, deleted_by = NULL WHERE id = :id",
           nativeQuery = true)
    void restoreById(@Param("id") Long id);

    @Query(value = "SELECT * FROM posts WHERE deleted = true ORDER BY deleted_at DESC",
           nativeQuery = true)
    List<Post> findSoftDeleted();
}
```

---

## 6. Optimistic Locking with @Version

```java
@Entity
@Table(name = "posts")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String content;

    // Version field for optimistic locking
    // JPA automatically increments this on every update
    // If the version in DB differs from the version in memory -> OptimisticLockException
    @Version
    private Long version;
}

// Handling optimistic lock exceptions
@Service
@RequiredArgsConstructor
@Slf4j
public class PostService {

    private final PostRepository postRepository;

    @Retryable(
        value = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100)
    )
    @Transactional
    public Post updatePost(Long postId, UpdatePostRequest request) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException("Post not found"));

        post.setTitle(request.getTitle());
        post.setContent(request.getContent());

        // If another transaction modified this post between our read and save:
        // JPA throws OptimisticLockingFailureException (wraps OptimisticLockException)
        return postRepository.save(post);
    }

    @Recover
    public Post recoverFromOptimisticLock(OptimisticLockingFailureException ex, Long postId,
                                          UpdatePostRequest request) {
        log.error("Failed to update post {} after retries: {}", postId, ex.getMessage());
        throw new ConcurrentModificationException("Post was modified by another user, please try again");
    }
}

// REST controller returning version to client
@RestController
@RequestMapping("/api/posts")
public class PostController {

    @PutMapping("/{id}")
    public ResponseEntity<PostDTO> updatePost(
            @PathVariable Long id,
            @RequestBody UpdatePostRequest request) {
        // Client sends version in request body
        // Server verifies it matches current version
        Post post = postService.updatePost(id, request);
        return ResponseEntity.ok(PostDTO.from(post));
    }
}

// DTO with version
@Getter @Setter @Builder
public class PostDTO {
    private Long id;
    private String title;
    private String content;
    private Long version;  // Send back to client for next update

    public static PostDTO from(Post post) {
        return PostDTO.builder()
            .id(post.getId())
            .title(post.getTitle())
            .content(post.getContent())
            .version(post.getVersion())  // Client must send this back
            .build();
    }
}
```

---

## 7. Database Auditing

```java
// Enable auditing with custom AuditorAware
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaAuditConfig {

    @Bean
    @ConditionalOnMissingBean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .filter(auth -> !"anonymousUser".equals(auth.getPrincipal()))
            .map(Authentication::getName);
    }
}

// Comprehensive audit base entity
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class FullAuditEntity {

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

// Audit log entity for complete change tracking
@Entity
@Table(name = "audit_logs")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    private String entityType;       // "Post", "User", etc.

    @Column(nullable = false)
    private Long entityId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private AuditAction action;      // CREATE, UPDATE, DELETE

    @Column(columnDefinition = "TEXT")
    private String oldValues;        // JSON of old state

    @Column(columnDefinition = "TEXT")
    private String newValues;        // JSON of new state

    @Column(nullable = false, length = 100)
    private String performedBy;

    @Column(nullable = false)
    private LocalDateTime performedAt;
}

enum AuditAction { CREATE, UPDATE, DELETE }

// AuditLog service
@Service
@RequiredArgsConstructor
@Transactional(propagation = Propagation.REQUIRES_NEW)  // Independent transaction
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;
    private final ObjectMapper objectMapper;

    public <T> void logCreate(T entity, String entityType, Long entityId) {
        AuditLog log = AuditLog.builder()
            .entityType(entityType)
            .entityId(entityId)
            .action(AuditAction.CREATE)
            .newValues(toJson(entity))
            .performedBy(SecurityUtils.getCurrentUsername())
            .performedAt(LocalDateTime.now())
            .build();
        auditLogRepository.save(log);
    }

    public <T> void logUpdate(T oldEntity, T newEntity, String entityType, Long entityId) {
        AuditLog log = AuditLog.builder()
            .entityType(entityType)
            .entityId(entityId)
            .action(AuditAction.UPDATE)
            .oldValues(toJson(oldEntity))
            .newValues(toJson(newEntity))
            .performedBy(SecurityUtils.getCurrentUsername())
            .performedAt(LocalDateTime.now())
            .build();
        auditLogRepository.save(log);
    }

    private String toJson(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            return "{}";
        }
    }
}
```

---

## 8. Second-Level Cache with Caffeine

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```java
// Enable caching
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(500)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats()  // Enable statistics
        );
        return cacheManager;
    }
}

// Cache in service
@Service
@RequiredArgsConstructor
@Slf4j
public class CategoryService {

    private final CategoryRepository categoryRepository;

    // Cache the result - key is the method's return value cached by 'categories'
    @Cacheable(value = "categories", key = "#root.methodName")
    public List<Category> findAllCategories() {
        log.info("Fetching categories from database");  // Only logged on cache miss
        return categoryRepository.findAll();
    }

    @Cacheable(value = "categories", key = "#id")
    public Optional<Category> findById(Long id) {
        return categoryRepository.findById(id);
    }

    // Evict specific cache entry
    @CacheEvict(value = "categories", key = "#result.id")
    @Transactional
    public Category createCategory(CreateCategoryRequest request) {
        Category category = Category.builder()
            .name(request.getName())
            .slug(slugUtils.toSlug(request.getName()))
            .build();
        return categoryRepository.save(category);
    }

    // Evict ALL entries in cache
    @CacheEvict(value = "categories", allEntries = true)
    @Transactional
    public Category updateCategory(Long id, UpdateCategoryRequest request) {
        Category category = categoryRepository.findById(id)
            .orElseThrow(() -> new CategoryNotFoundException("Category not found"));
        category.setName(request.getName());
        return categoryRepository.save(category);
    }

    // Cache the result and also evict another cache
    @Caching(
        evict = @CacheEvict(value = "categories", allEntries = true),
        put = @CachePut(value = "category-by-id", key = "#result.id")
    )
    @Transactional
    public Category deleteAndReturn(Long id) {
        Category category = categoryRepository.findById(id)
            .orElseThrow(() -> new CategoryNotFoundException("Category not found"));
        categoryRepository.delete(category);
        return category;
    }
}
```

---

## 9. Database Migrations with Flyway

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<!-- For PostgreSQL -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```properties
# application.properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
spring.flyway.validate-on-migrate=true
spring.jpa.hibernate.ddl-auto=validate  # Let Flyway manage schema
```

### Migration Files (in `src/main/resources/db/migration/`)

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) NOT NULL UNIQUE,
    email       VARCHAR(100) NOT NULL UNIQUE,
    password    VARCHAR(255) NOT NULL,
    enabled     BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);

-- V2__create_categories_table.sql
CREATE TABLE categories (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL UNIQUE,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    parent_id   BIGINT REFERENCES categories(id)
);

-- V3__create_posts_table.sql
CREATE TABLE posts (
    id           BIGSERIAL PRIMARY KEY,
    title        VARCHAR(255) NOT NULL,
    slug         VARCHAR(255) NOT NULL UNIQUE,
    content      TEXT,
    excerpt      VARCHAR(500),
    status       VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    featured     BOOLEAN NOT NULL DEFAULT false,
    view_count   INT NOT NULL DEFAULT 0,
    author_id    BIGINT NOT NULL REFERENCES users(id),
    category_id  BIGINT REFERENCES categories(id),
    published_at TIMESTAMP,
    created_at   TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_author ON posts(author_id);
CREATE INDEX idx_posts_category ON posts(category_id);

-- V4__create_tags_table.sql
CREATE TABLE tags (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    slug VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE post_tags (
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- V5__add_soft_delete_to_posts.sql
ALTER TABLE posts
    ADD COLUMN deleted     BOOLEAN NOT NULL DEFAULT false,
    ADD COLUMN deleted_at  TIMESTAMP,
    ADD COLUMN deleted_by  VARCHAR(100);

CREATE INDEX idx_posts_deleted ON posts(deleted);

-- V6__seed_initial_data.sql
INSERT INTO users (username, email, password)
VALUES ('admin', 'admin@example.com', '$2a$12$...bcrypt_hash...');

INSERT INTO categories (name, slug)
VALUES ('Technology', 'technology'),
       ('Programming', 'programming'),
       ('Spring', 'spring');
```

---

## 10. Database Migrations with Liquibase

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

```properties
# application.properties
spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.xml
spring.liquibase.enabled=true
```

```xml
<!-- src/main/resources/db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <include file="db/changelog/changes/001-create-users.xml"/>
    <include file="db/changelog/changes/002-create-posts.xml"/>
    <include file="db/changelog/changes/003-add-tags.xml"/>
</databaseChangeLog>
```

```xml
<!-- src/main/resources/db/changelog/changes/001-create-users.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog ...>

    <changeSet id="001-create-users-table" author="developer">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="username" type="VARCHAR(50)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="email" type="VARCHAR(100)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="password" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="enabled" type="BOOLEAN" defaultValueBoolean="true">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="NOW()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <createIndex tableName="users" indexName="idx_users_username">
            <column name="username"/>
        </createIndex>
    </changeSet>
</databaseChangeLog>
```

---

## 11. Complete Example: Putting It All Together

```java
// Fully-featured post entity with all patterns
@Entity
@Table(name = "posts")
@EntityListeners(AuditingEntityListener.class)
@Where(clause = "deleted = false")
@SQLDelete(sql = "UPDATE posts SET deleted = true, deleted_at = NOW() WHERE id = ? AND version = ?")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
@EqualsAndHashCode(of = "id")
@ToString(exclude = {"author", "category", "tags", "comments"})
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

    @Column(nullable = false)
    @Builder.Default
    private boolean deleted = false;

    @Column
    private LocalDateTime deletedAt;

    @Column(length = 100)
    private String deletedBy;

    @Column
    private LocalDateTime publishedAt;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(length = 100, updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(length = 100)
    private String updatedBy;

    @Version
    private Long version;

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
    @BatchSize(size = 20)
    @Builder.Default
    private Set<Tag> tags = new HashSet<>();

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("createdAt DESC")
    @Builder.Default
    private List<Comment> comments = new ArrayList<>();
}

// Repository using all patterns
@Repository
public interface PostRepository extends JpaRepository<Post, Long>,
        JpaSpecificationExecutor<Post>,
        PostRepositoryCustom {

    // Projections
    <T> Page<T> findByStatus(PostStatus status, Class<T> type, Pageable pageable);

    // EntityGraph to avoid N+1
    @EntityGraph(attributePaths = {"author", "category", "tags"})
    Optional<Post> findDetailedById(Long id);

    // Soft delete restore
    @Modifying
    @Query(value = "UPDATE posts SET deleted = false, deleted_at = NULL, deleted_by = NULL, version = version + 1 WHERE id = :id",
           nativeQuery = true)
    void restoreById(@Param("id") Long id);
}

// Service using all features
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class AdvancedPostService {

    private final PostRepository postRepository;
    private final AuditLogService auditLogService;

    // Search with Specifications
    public Page<PostSummaryProjection> search(PostSearchRequest request, Pageable pageable) {
        Specification<Post> spec = buildSpec(request);
        return postRepository.findAll(spec, PostSummaryProjection.class, pageable);
    }

    private Specification<Post> buildSpec(PostSearchRequest request) {
        return Specification.where(PostSpecifications.hasStatus(request.getStatus()))
            .and(PostSpecifications.titleOrContentContains(request.getKeyword()))
            .and(PostSpecifications.inCategory(request.getCategoryId()))
            .and(PostSpecifications.publishedAfter(request.getFromDate()))
            .and(PostSpecifications.publishedBefore(request.getToDate()));
    }

    // Get detail with EntityGraph (no N+1)
    public PostDetailDTO getPostDetail(Long id) {
        Post post = postRepository.findDetailedById(id)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + id));
        return PostDetailDTO.from(post);
    }

    @Transactional
    public Post createPost(CreatePostRequest request, String username) {
        // ... create post ...
        Post saved = postRepository.save(post);
        auditLogService.logCreate(saved, "Post", saved.getId());
        return saved;
    }

    @Transactional
    public Post updatePost(Long id, UpdatePostRequest request) {
        Post oldPost = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + id));
        Post oldState = copyPost(oldPost);  // Capture old state before modification

        oldPost.setTitle(request.getTitle());
        oldPost.setContent(request.getContent());

        Post updated = postRepository.save(oldPost);
        auditLogService.logUpdate(oldState, updated, "Post", id);
        return updated;
    }

    @Transactional
    public void softDelete(Long id, String username) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + id));
        post.setDeletedBy(username);
        postRepository.save(post);
        postRepository.delete(post);  // Triggers @SQLDelete
        auditLogService.logCreate(post, "Post:DELETE", id);
    }
}
```

---

## 12. Summary Table

| Pattern | Class/Annotation | Use Case |
|---|---|---|
| Custom queries | `PostRepositoryCustom` | Complex queries, bulk operations |
| Dynamic queries | `Specification<T>` | Variable filter conditions |
| Field selection | Interface projection | Return subset of fields |
| DTO projection | Constructor expression | Efficient DTO query results |
| N+1 fix | `@EntityGraph` | Eager-load specific associations |
| N+1 fix | `JOIN FETCH` | JPQL eager fetch |
| Batch load | `@BatchSize` | Load collections in batches |
| Soft delete | `@Where`, `@SQLDelete` | Mark deleted, keep data |
| Optimistic lock | `@Version` | Prevent concurrent updates |
| Auditing | `@CreatedBy`, `@CreatedDate` | Auto-fill audit fields |
| Caching | `@Cacheable`, `@CacheEvict` | Cache query results |
| Schema migration | Flyway / Liquibase | Database versioning |

---

## Next Part

**Part 029** covers advanced REST API design: API versioning strategies, HATEOAS, OpenAPI/Swagger documentation, content negotiation, rate limiting, API response standards (JSend, Problem Details RFC 7807), pagination patterns, filtering, ETag caching, and building a complete production-ready API.
