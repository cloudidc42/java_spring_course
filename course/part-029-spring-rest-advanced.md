# Part 029: Advanced REST API Design

Building production-grade REST APIs requires more than basic CRUD operations. This part covers API versioning, HATEOAS, documentation, rate limiting, response standards, pagination, caching, and more.

---

## 1. Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- HATEOAS -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-hateoas</artifactId>
    </dependency>

    <!-- OpenAPI/Swagger -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.3.0</version>
    </dependency>

    <!-- Rate limiting with Bucket4j -->
    <dependency>
        <groupId>com.bucket4j</groupId>
        <artifactId>bucket4j-core</artifactId>
        <version>8.7.0</version>
    </dependency>

    <!-- Jackson XML support for content negotiation -->
    <dependency>
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-xml</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 2. API Versioning Strategies

### Strategy 1: URL Path Versioning

```java
// Version 1 controller
@RestController
@RequestMapping("/api/v1/posts")
@RequiredArgsConstructor
public class PostControllerV1 {

    private final PostService postService;

    @GetMapping
    public ResponseEntity<List<PostDTOV1>> getPosts() {
        return ResponseEntity.ok(
            postService.findAll().stream()
                .map(PostDTOV1::from)
                .toList()
        );
    }

    @GetMapping("/{id}")
    public ResponseEntity<PostDTOV1> getPost(@PathVariable Long id) {
        return postService.findById(id)
            .map(PostDTOV1::from)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}

// Version 2 controller - different response structure
@RestController
@RequestMapping("/api/v2/posts")
@RequiredArgsConstructor
public class PostControllerV2 {

    private final PostService postService;

    @GetMapping
    public ResponseEntity<ApiResponse<List<PostDTOV2>>> getPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        // V2 returns paginated response with ApiResponse wrapper
        Page<Post> posts = postService.findAllPublished(PageRequest.of(page, size));
        return ResponseEntity.ok(ApiResponse.success(
            posts.getContent().stream().map(PostDTOV2::from).toList(),
            PageMetadata.from(posts)
        ));
    }
}

// V1 DTO - simple flat structure
@Getter @Setter @Builder
public class PostDTOV1 {
    private Long id;
    private String title;
    private String content;
    private String authorUsername;

    public static PostDTOV1 from(Post post) {
        return PostDTOV1.builder()
            .id(post.getId())
            .title(post.getTitle())
            .content(post.getContent())
            .authorUsername(post.getAuthor().getUsername())
            .build();
    }
}

// V2 DTO - richer structure with nested objects
@Getter @Setter @Builder
public class PostDTOV2 {
    private Long id;
    private String title;
    private String slug;
    private String content;
    private String excerpt;
    private AuthorInfo author;
    private CategoryInfo category;
    private List<String> tags;
    private int viewCount;
    private boolean featured;
    private LocalDateTime publishedAt;
    private LocalDateTime createdAt;

    @Getter @AllArgsConstructor
    public static class AuthorInfo {
        private Long id;
        private String username;
        private String email;
    }

    @Getter @AllArgsConstructor
    public static class CategoryInfo {
        private Long id;
        private String name;
        private String slug;
    }

    public static PostDTOV2 from(Post post) {
        return PostDTOV2.builder()
            .id(post.getId())
            .title(post.getTitle())
            .slug(post.getSlug())
            .content(post.getContent())
            .excerpt(post.getExcerpt())
            .author(new AuthorInfo(
                post.getAuthor().getId(),
                post.getAuthor().getUsername(),
                post.getAuthor().getEmail()
            ))
            .category(post.getCategory() != null
                ? new CategoryInfo(
                    post.getCategory().getId(),
                    post.getCategory().getName(),
                    post.getCategory().getSlug()
                ) : null)
            .tags(post.getTags().stream().map(Tag::getName).toList())
            .viewCount(post.getViewCount())
            .featured(post.isFeatured())
            .publishedAt(post.getPublishedAt())
            .createdAt(post.getCreatedAt())
            .build();
    }
}
```

### Strategy 2: Header-Based Versioning

```java
@RestController
@RequestMapping("/api/posts")
@RequiredArgsConstructor
public class PostController {

    private final PostService postService;

    @GetMapping(headers = "X-API-Version=1")
    public ResponseEntity<List<PostDTOV1>> getPostsV1() {
        return ResponseEntity.ok(
            postService.findAll().stream().map(PostDTOV1::from).toList()
        );
    }

    @GetMapping(headers = "X-API-Version=2")
    public ResponseEntity<ApiResponse<List<PostDTOV2>>> getPostsV2(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        Page<Post> posts = postService.findAllPublished(PageRequest.of(page, size));
        return ResponseEntity.ok(
            ApiResponse.success(posts.getContent().stream().map(PostDTOV2::from).toList())
        );
    }
}

// Usage: GET /api/posts
// With header: X-API-Version: 1  -> V1 response
// With header: X-API-Version: 2  -> V2 response
```

### Strategy 3: Accept Header / Content Negotiation

```java
@RestController
@RequestMapping("/api/posts")
@RequiredArgsConstructor
public class PostController {

    @GetMapping(produces = "application/vnd.blog.v1+json")
    public ResponseEntity<List<PostDTOV1>> getPostsV1() {
        return ResponseEntity.ok(postService.findAll().stream()
            .map(PostDTOV1::from).toList());
    }

    @GetMapping(produces = "application/vnd.blog.v2+json")
    public ResponseEntity<List<PostDTOV2>> getPostsV2() {
        return ResponseEntity.ok(postService.findAll().stream()
            .map(PostDTOV2::from).toList());
    }

    // Fallback: default to v2
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<List<PostDTOV2>> getPosts() {
        return ResponseEntity.ok(postService.findAll().stream()
            .map(PostDTOV2::from).toList());
    }
}

// Usage: GET /api/posts
// Accept: application/vnd.blog.v1+json  -> V1
// Accept: application/vnd.blog.v2+json  -> V2
```

### Strategy 4: Query Parameter Versioning

```java
@RestController
@RequestMapping("/api/posts")
@RequiredArgsConstructor
public class PostController {

    @GetMapping(params = "version=1")
    public ResponseEntity<List<PostDTOV1>> getPostsV1() {
        return ResponseEntity.ok(postService.findAll().stream()
            .map(PostDTOV1::from).toList());
    }

    @GetMapping(params = "version=2")
    public ResponseEntity<List<PostDTOV2>> getPostsV2() {
        return ResponseEntity.ok(postService.findAll().stream()
            .map(PostDTOV2::from).toList());
    }
}

// Usage: GET /api/posts?version=1  OR  GET /api/posts?version=2
```

---

## 3. HATEOAS

HATEOAS (Hypermedia As The Engine Of Application State) adds navigational links to API responses.

### Model with Links

```java
// Post response model with HATEOAS links
public class PostModel extends RepresentationModel<PostModel> {

    private Long id;
    private String title;
    private String slug;
    private String content;
    private String excerpt;
    private String authorUsername;
    private String categoryName;
    private List<String> tags;
    private int viewCount;
    private LocalDateTime publishedAt;

    public static PostModel from(Post post) {
        PostModel model = new PostModel();
        model.id = post.getId();
        model.title = post.getTitle();
        model.slug = post.getSlug();
        model.content = post.getContent();
        model.excerpt = post.getExcerpt();
        model.authorUsername = post.getAuthor().getUsername();
        model.categoryName = post.getCategory() != null ? post.getCategory().getName() : null;
        model.tags = post.getTags().stream().map(Tag::getName).toList();
        model.viewCount = post.getViewCount();
        model.publishedAt = post.getPublishedAt();
        return model;
    }

    // Getters...
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public String getSlug() { return slug; }
    public String getContent() { return content; }
    public String getExcerpt() { return excerpt; }
    public String getAuthorUsername() { return authorUsername; }
    public String getCategoryName() { return categoryName; }
    public List<String> getTags() { return tags; }
    public int getViewCount() { return viewCount; }
    public LocalDateTime getPublishedAt() { return publishedAt; }
}
```

### ModelAssembler

```java
@Component
public class PostModelAssembler implements RepresentationModelAssembler<Post, PostModel> {

    @Override
    public PostModel toModel(Post post) {
        PostModel model = PostModel.from(post);

        // Self link
        model.add(linkTo(methodOn(PostController.class).getPost(post.getId())).withSelfRel());

        // Related links
        model.add(linkTo(methodOn(PostController.class).getPosts(0, 10, "publishedAt", "desc", null))
            .withRel("posts"));

        if (post.getCategory() != null) {
            model.add(linkTo(methodOn(CategoryController.class)
                .getCategory(post.getCategory().getId()))
                .withRel("category"));
        }

        model.add(linkTo(methodOn(CommentController.class).getPostComments(post.getId()))
            .withRel("comments"));

        model.add(linkTo(methodOn(UserController.class).getUser(post.getAuthor().getId()))
            .withRel("author"));

        return model;
    }

    @Override
    public CollectionModel<PostModel> toCollectionModel(Iterable<? extends Post> posts) {
        CollectionModel<PostModel> models = RepresentationModelAssembler.super.toCollectionModel(posts);

        // Add collection-level links
        models.add(linkTo(methodOn(PostController.class)
            .getPosts(0, 10, "publishedAt", "desc", null)).withSelfRel());

        return models;
    }
}
```

### HATEOAS Controller

```java
@RestController
@RequestMapping("/api/v2/posts")
@RequiredArgsConstructor
public class PostHateoasController {

    private final PostService postService;
    private final PostModelAssembler assembler;

    @GetMapping
    public ResponseEntity<PagedModel<PostModel>> getPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "publishedAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir,
            PagedResourcesAssembler<Post> pagedAssembler) {

        Sort sort = Sort.by(Sort.Direction.fromString(sortDir), sortBy);
        Pageable pageable = PageRequest.of(page, size, sort);
        Page<Post> posts = postService.findAllPublished(pageable);

        // pagedAssembler automatically adds next/prev/first/last links
        PagedModel<PostModel> pagedModel = pagedAssembler.toModel(posts, assembler);

        return ResponseEntity.ok(pagedModel);
    }

    @GetMapping("/{id}")
    public ResponseEntity<PostModel> getPost(@PathVariable Long id) {
        return postService.findById(id)
            .map(assembler::toModel)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('AUTHOR', 'ADMIN')")
    public ResponseEntity<PostModel> createPost(
            @Valid @RequestBody CreatePostRequest request,
            @AuthenticationPrincipal UserPrincipal principal) {
        Post post = postService.createPost(request, principal.getUsername());
        PostModel model = assembler.toModel(post);

        return ResponseEntity
            .created(model.getRequiredLink(IanaLinkRelations.SELF).toUri())
            .body(model);
    }
}

// Example HATEOAS response:
// {
//   "id": 1,
//   "title": "Spring Data JPA Guide",
//   "_links": {
//     "self": { "href": "http://localhost:8080/api/v2/posts/1" },
//     "posts": { "href": "http://localhost:8080/api/v2/posts?page=0&size=10" },
//     "category": { "href": "http://localhost:8080/api/v2/categories/3" },
//     "comments": { "href": "http://localhost:8080/api/v2/posts/1/comments" },
//     "author": { "href": "http://localhost:8080/api/v2/users/42" }
//   }
// }
```

---

## 4. OpenAPI / Swagger Documentation

```java
// OpenAPI configuration
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Blog API")
                .version("2.0.0")
                .description("RESTful Blog API built with Spring Boot 3")
                .contact(new Contact()
                    .name("Dev Team")
                    .email("dev@example.com")
                    .url("https://example.com"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://apache.org/licenses/LICENSE-2.0"))
            )
            .externalDocs(new ExternalDocumentation()
                .description("Full Documentation")
                .url("https://docs.example.com")
            )
            .servers(List.of(
                new Server().url("http://localhost:8080").description("Local dev"),
                new Server().url("https://api.example.com").description("Production")
            ))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .name("bearerAuth")
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")
                        .description("JWT token obtained from /api/auth/login")
                )
            );
    }
}

// Controller with OpenAPI annotations
@RestController
@RequestMapping("/api/v2/posts")
@Tag(name = "Posts", description = "Post management APIs")
@RequiredArgsConstructor
public class PostController {

    @Operation(
        summary = "Get all published posts",
        description = "Returns a paginated list of published posts, newest first",
        parameters = {
            @Parameter(name = "page", description = "Page number (0-indexed)", example = "0"),
            @Parameter(name = "size", description = "Page size", example = "10"),
            @Parameter(name = "category", description = "Filter by category slug")
        },
        responses = {
            @ApiResponse(
                responseCode = "200",
                description = "Successful",
                content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = PostPageResponse.class)
                )
            )
        }
    )
    @GetMapping
    public ResponseEntity<ApiResponse<List<PostDTOV2>>> getPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        // ...
    }

    @Operation(
        summary = "Get post by ID",
        responses = {
            @ApiResponse(responseCode = "200", description = "Post found"),
            @ApiResponse(responseCode = "404", description = "Post not found",
                content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
        }
    )
    @GetMapping("/{id}")
    public ResponseEntity<PostDTOV2> getPost(
            @Parameter(description = "Post ID", required = true)
            @PathVariable Long id) {
        return postService.findById(id)
            .map(PostDTOV2::from)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @Operation(
        summary = "Create a new post",
        security = @SecurityRequirement(name = "bearerAuth"),
        responses = {
            @ApiResponse(responseCode = "201", description = "Post created"),
            @ApiResponse(responseCode = "400", description = "Validation error"),
            @ApiResponse(responseCode = "401", description = "Unauthorized")
        }
    )
    @PostMapping
    @PreAuthorize("hasAnyRole('AUTHOR', 'ADMIN')")
    public ResponseEntity<PostDTOV2> createPost(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Post creation data",
                required = true,
                content = @Content(schema = @Schema(implementation = CreatePostRequest.class))
            )
            @Valid @RequestBody CreatePostRequest request,
            @AuthenticationPrincipal UserPrincipal principal) {
        Post post = postService.createPost(request, principal.getUsername());
        return ResponseEntity.status(HttpStatus.CREATED).body(PostDTOV2.from(post));
    }
}

// DTO with Swagger schema annotations
@Schema(description = "Request body for creating a post")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class CreatePostRequest {

    @Schema(description = "Post title", example = "Getting Started with Spring Boot", required = true)
    @NotBlank(message = "Title is required")
    @Size(max = 255)
    private String title;

    @Schema(description = "Post content in Markdown", required = true)
    @NotBlank(message = "Content is required")
    private String content;

    @Schema(description = "Short summary (max 500 chars)", example = "Learn Spring Boot basics...")
    @Size(max = 500)
    private String excerpt;

    @Schema(description = "Category ID")
    private Long categoryId;

    @Schema(description = "Tag IDs to assign to this post")
    private Set<Long> tagIds;
}
```

---

## 5. Content Negotiation (JSON + XML)

```java
// Enable XML support in application.properties
// spring.mvc.contentnegotiation.favor-parameter=true
// spring.mvc.contentnegotiation.parameter-name=format

// DTO with XML support
@JacksonXmlRootElement(localName = "post")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class PostDTOV2 {

    private Long id;

    @JacksonXmlProperty(localName = "post-title")
    private String title;

    private String slug;
    private String content;
    private String excerpt;

    @JacksonXmlProperty(localName = "author-username")
    private String authorUsername;

    @JacksonXmlElementWrapper(localName = "tags")
    @JacksonXmlProperty(localName = "tag")
    private List<String> tags;

    private LocalDateTime publishedAt;
}

// Controller accepting/producing both JSON and XML
@RestController
@RequestMapping("/api/v2/posts")
public class PostController {

    @GetMapping(
        value = "/{id}",
        produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE}
    )
    public ResponseEntity<PostDTOV2> getPost(@PathVariable Long id) {
        return postService.findById(id)
            .map(PostDTOV2::from)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping(
        consumes = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE},
        produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE}
    )
    public ResponseEntity<PostDTOV2> createPost(@RequestBody CreatePostRequest request) {
        // ...
    }
}

// Usage:
// GET /api/v2/posts/1           -> JSON (default)
// GET /api/v2/posts/1           -> Accept: application/xml  -> XML
// GET /api/v2/posts/1?format=xml (with parameter strategy enabled) -> XML
```

---

## 6. API Response Standards

### JSend Response Wrapper

```java
// JSend-style response (success/fail/error)
@Getter
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    private final String status;    // "success", "fail", "error"
    private final T data;
    private final String message;
    private final Integer code;
    private final PageMetadata pagination;

    private ApiResponse(String status, T data, String message, Integer code, PageMetadata pagination) {
        this.status = status;
        this.data = data;
        this.message = message;
        this.code = code;
        this.pagination = pagination;
    }

    // Success with data
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>("success", data, null, null, null);
    }

    // Success with data and pagination
    public static <T> ApiResponse<T> success(T data, PageMetadata pagination) {
        return new ApiResponse<>("success", data, null, null, pagination);
    }

    // Validation failures (fail)
    public static <T> ApiResponse<T> fail(T data) {
        return new ApiResponse<>("fail", data, null, null, null);
    }

    // Server errors (error)
    public static <T> ApiResponse<T> error(String message, Integer code) {
        return new ApiResponse<>("error", null, message, code, null);
    }
}

// Pagination metadata
@Getter @Builder
public class PageMetadata {
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
    private boolean first;
    private boolean last;
    private boolean hasNext;
    private boolean hasPrevious;

    public static PageMetadata from(Page<?> page) {
        return PageMetadata.builder()
            .page(page.getNumber())
            .size(page.getSize())
            .totalElements(page.getTotalElements())
            .totalPages(page.getTotalPages())
            .first(page.isFirst())
            .last(page.isLast())
            .hasNext(page.hasNext())
            .hasPrevious(page.hasPrevious())
            .build();
    }
}
```

### Problem Details (RFC 7807)

```java
// RFC 7807 Problem Details response
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ProblemDetail {
    private String type;        // URI identifying the problem type
    private String title;       // Short summary
    private int status;         // HTTP status code
    private String detail;      // Human-readable explanation
    private String instance;    // URI identifying the specific occurrence
    private Map<String, Object> extensions;  // Extra properties

    // Factory methods
    public static ProblemDetail forStatus(HttpStatus status) {
        return ProblemDetail.builder()
            .type("about:blank")
            .title(status.getReasonPhrase())
            .status(status.value())
            .build();
    }

    public static ProblemDetail forStatusAndDetail(HttpStatus status, String detail) {
        ProblemDetail problem = forStatus(status);
        problem.setDetail(detail);
        return problem;
    }
}

// Spring 6 has built-in ProblemDetail support:
// org.springframework.http.ProblemDetail
// Use it instead:

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(PostNotFoundException.class)
    public ResponseEntity<org.springframework.http.ProblemDetail> handleNotFound(
            PostNotFoundException ex, HttpServletRequest request) {

        org.springframework.http.ProblemDetail problem =
            org.springframework.http.ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", LocalDateTime.now());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<org.springframework.http.ProblemDetail> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(error -> errors.put(error.getField(), error.getDefaultMessage()));

        org.springframework.http.ProblemDetail problem =
            org.springframework.http.ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        problem.setDetail("One or more fields failed validation");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("errors", errors);
        problem.setProperty("timestamp", LocalDateTime.now());

        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<org.springframework.http.ProblemDetail> handleGeneral(
            Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception: ", ex);

        org.springframework.http.ProblemDetail problem =
            org.springframework.http.ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problem.setTitle("Internal Server Error");
        problem.setDetail("An unexpected error occurred");
        problem.setInstance(URI.create(request.getRequestURI()));

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .body(problem);
    }
}
```

---

## 7. Rate Limiting with Bucket4j

```java
// Rate limiting filter using Bucket4j
@Component
@Order(1)  // Run before other filters
@Slf4j
public class RateLimitingFilter extends OncePerRequestFilter {

    // Store buckets per IP address
    private final Map<String, Bucket> ipBuckets = new ConcurrentHashMap<>();

    // Store buckets per user
    private final Map<String, Bucket> userBuckets = new ConcurrentHashMap<>();

    private Bucket createBucketForAnonymous() {
        // 100 requests per minute for anonymous users
        return Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
            .build();
    }

    private Bucket createBucketForAuthenticatedUser() {
        // 1000 requests per minute for authenticated users
        return Bucket.builder()
            .addLimit(Bandwidth.classic(1000, Refill.intervally(1000, Duration.ofMinutes(1))))
            .build();
    }

    private Bucket createBucketForAdmin() {
        // No rate limiting for admins (or very high limit)
        return Bucket.builder()
            .addLimit(Bandwidth.classic(10000, Refill.intervally(10000, Duration.ofMinutes(1))))
            .build();
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        // Skip rate limiting for swagger UI and health check
        String path = request.getRequestURI();
        if (path.startsWith("/swagger-ui") || path.startsWith("/actuator/health")) {
            chain.doFilter(request, response);
            return;
        }

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String key;
        Bucket bucket;

        if (auth != null && auth.isAuthenticated() && !"anonymousUser".equals(auth.getPrincipal())) {
            key = "user:" + auth.getName();
            boolean isAdmin = auth.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
            bucket = isAdmin
                ? userBuckets.computeIfAbsent(key, k -> createBucketForAdmin())
                : userBuckets.computeIfAbsent(key, k -> createBucketForAuthenticatedUser());
        } else {
            key = "ip:" + getClientIP(request);
            bucket = ipBuckets.computeIfAbsent(key, k -> createBucketForAnonymous());
        }

        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);

        if (probe.isConsumed()) {
            // Add rate limit headers
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("X-RateLimit-Remaining", String.valueOf(probe.getRemainingTokens()));
            response.setHeader("X-RateLimit-Reset",
                String.valueOf(System.currentTimeMillis() + probe.getNanosToWaitForRefill() / 1_000_000));
            chain.doFilter(request, response);
        } else {
            long waitSeconds = probe.getNanosToWaitForRefill() / 1_000_000_000;
            response.setHeader("Retry-After", String.valueOf(waitSeconds));
            response.setHeader("X-RateLimit-Remaining", "0");
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {
                  "status": 429,
                  "error": "Too Many Requests",
                  "message": "Rate limit exceeded. Try again in %d seconds"
                }
                """.formatted(waitSeconds));
            log.warn("Rate limit exceeded for: {}", key);
        }
    }

    private String getClientIP(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## 8. Pagination Standards

### Offset-Based Pagination

```java
@RestController
@RequestMapping("/api/v2/posts")
@RequiredArgsConstructor
public class PostController {

    @GetMapping
    public ResponseEntity<ApiResponse<List<PostDTOV2>>> getPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "publishedAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String tag,
            @RequestParam(required = false) String keyword,
            HttpServletRequest request) {

        // Validate and cap page size
        size = Math.min(size, 100);  // Max 100 per page

        Sort sort = sortDir.equalsIgnoreCase("asc")
            ? Sort.by(sortBy).ascending()
            : Sort.by(sortBy).descending();
        Pageable pageable = PageRequest.of(page, size, sort);

        Page<Post> posts = postService.search(keyword, category, tag, pageable);

        // Build pagination links
        String baseUrl = request.getRequestURL().toString() + "?" +
            "size=" + size + "&sortBy=" + sortBy + "&sortDir=" + sortDir;

        List<String> links = new ArrayList<>();
        links.add("<" + baseUrl + "&page=0>; rel=\"first\"");
        links.add("<" + baseUrl + "&page=" + posts.getTotalPages() + ">; rel=\"last\"");
        if (posts.hasNext()) {
            links.add("<" + baseUrl + "&page=" + (page + 1) + ">; rel=\"next\"");
        }
        if (posts.hasPrevious()) {
            links.add("<" + baseUrl + "&page=" + (page - 1) + ">; rel=\"prev\"");
        }

        return ResponseEntity.ok()
            .header("Link", String.join(", ", links))
            .header("X-Total-Count", String.valueOf(posts.getTotalElements()))
            .header("X-Page-Number", String.valueOf(posts.getNumber()))
            .header("X-Page-Size", String.valueOf(posts.getSize()))
            .header("X-Total-Pages", String.valueOf(posts.getTotalPages()))
            .body(ApiResponse.success(
                posts.getContent().stream().map(PostDTOV2::from).toList(),
                PageMetadata.from(posts)
            ));
    }
}
```

### Cursor-Based Pagination (for large datasets)

```java
@RestController
@RequestMapping("/api/v2/posts/feed")
@RequiredArgsConstructor
public class PostFeedController {

    private final PostRepository postRepository;

    // Cursor-based pagination - more efficient for large tables
    // Cursor is the last-seen item's key (e.g., publishedAt:id)
    @GetMapping
    public ResponseEntity<CursorPage<PostDTOV2>> getFeed(
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String after,  // opaque cursor
            @RequestParam(required = false) String before) {

        size = Math.min(size, 100);

        CursorDecoded cursor = after != null ? decodeCursor(after) : null;
        LocalDateTime cursorDate = cursor != null ? cursor.getPublishedAt() : LocalDateTime.now();
        Long cursorId = cursor != null ? cursor.getId() : Long.MAX_VALUE;

        // Fetch one extra to know if there's a next page
        Pageable pageable = PageRequest.of(0, size + 1);
        List<Post> posts = postRepository.findForCursor(cursorDate, cursorId, pageable);

        boolean hasMore = posts.size() > size;
        if (hasMore) {
            posts = posts.subList(0, size);
        }

        String nextCursor = hasMore && !posts.isEmpty()
            ? encodeCursor(posts.get(posts.size() - 1))
            : null;

        return ResponseEntity.ok(CursorPage.<PostDTOV2>builder()
            .data(posts.stream().map(PostDTOV2::from).toList())
            .nextCursor(nextCursor)
            .hasMore(hasMore)
            .build());
    }

    private String encodeCursor(Post post) {
        // Encode composite cursor (publishedAt + id for uniqueness)
        String raw = post.getPublishedAt().toString() + ":" + post.getId();
        return Base64.getUrlEncoder().encodeToString(raw.getBytes());
    }

    private CursorDecoded decodeCursor(String cursor) {
        String raw = new String(Base64.getUrlDecoder().decode(cursor));
        String[] parts = raw.split(":");
        return new CursorDecoded(LocalDateTime.parse(parts[0]), Long.parseLong(parts[1]));
    }

    private record CursorDecoded(LocalDateTime publishedAt, Long id) {}
}

@Getter @Builder
public class CursorPage<T> {
    private List<T> data;
    private String nextCursor;
    private String prevCursor;
    private boolean hasMore;
    private int count;
}

// Repository method for cursor pagination
@Repository
public interface PostRepository extends JpaRepository<Post, Long> {

    @Query("""
        SELECT p FROM Post p
        WHERE p.status = 'PUBLISHED'
        AND (p.publishedAt < :cursorDate
             OR (p.publishedAt = :cursorDate AND p.id < :cursorId))
        ORDER BY p.publishedAt DESC, p.id DESC
        """)
    List<Post> findForCursor(@Param("cursorDate") LocalDateTime cursorDate,
                              @Param("cursorId") Long cursorId,
                              Pageable pageable);
}
```

---

## 9. Filtering and Sorting

```java
// Generic filter/sort support
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class PostFilterRequest {

    private String keyword;
    private PostStatus status;
    private Long categoryId;
    private String categorySlug;
    private String tagName;
    private Long authorId;
    private Boolean featured;
    private LocalDateTime fromDate;
    private LocalDateTime toDate;
    private Integer minViewCount;

    // Sorting
    private String sortBy;     // field name
    private String sortDir;    // asc/desc

    // Pagination
    private Integer page;
    private Integer size;

    // Parse from request parameters
    public static PostFilterRequest fromParams(
            String keyword, String status, Long categoryId,
            String tag, String sortBy, String sortDir,
            int page, int size) {
        return PostFilterRequest.builder()
            .keyword(keyword)
            .status(status != null ? PostStatus.valueOf(status.toUpperCase()) : PostStatus.PUBLISHED)
            .categoryId(categoryId)
            .tagName(tag)
            .sortBy(sortBy != null ? sortBy : "publishedAt")
            .sortDir(sortDir != null ? sortDir : "desc")
            .page(page)
            .size(Math.min(size, 100))
            .build();
    }

    public Pageable toPageable() {
        String field = allowedSortFields().getOrDefault(sortBy, "publishedAt");
        Sort sort = "asc".equalsIgnoreCase(sortDir)
            ? Sort.by(field).ascending()
            : Sort.by(field).descending();
        return PageRequest.of(page != null ? page : 0, size != null ? size : 10, sort);
    }

    // Whitelist of sortable fields (prevent injection)
    private Map<String, String> allowedSortFields() {
        return Map.of(
            "publishedAt", "publishedAt",
            "viewCount", "viewCount",
            "title", "title",
            "createdAt", "createdAt"
        );
    }
}

// Service using filter request
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PostService {

    private final PostRepository postRepository;

    public Page<Post> findPosts(PostFilterRequest filter) {
        Specification<Post> spec = buildSpecification(filter);
        Pageable pageable = filter.toPageable();
        return postRepository.findAll(spec, pageable);
    }

    private Specification<Post> buildSpecification(PostFilterRequest filter) {
        return Specification.where(PostSpecifications.hasStatus(filter.getStatus()))
            .and(PostSpecifications.titleOrContentContains(filter.getKeyword()))
            .and(PostSpecifications.inCategory(filter.getCategoryId()))
            .and(PostSpecifications.hasTag(filter.getTagName()))
            .and(PostSpecifications.byAuthor(filter.getAuthorId()))
            .and(filter.getFeatured() != null && filter.getFeatured()
                ? PostSpecifications.isFeatured() : null)
            .and(PostSpecifications.publishedAfter(filter.getFromDate()))
            .and(PostSpecifications.publishedBefore(filter.getToDate()))
            .and(filter.getMinViewCount() != null
                ? PostSpecifications.hasViewCountGreaterThan(filter.getMinViewCount()) : null);
    }
}
```

---

## 10. ETag and HTTP Caching Headers

```java
@RestController
@RequestMapping("/api/v2/posts")
@RequiredArgsConstructor
public class PostController {

    private final PostService postService;

    @GetMapping("/{id}")
    public ResponseEntity<PostDTOV2> getPost(
            @PathVariable Long id,
            @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {

        Post post = postService.findById(id)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + id));

        // Generate ETag based on last modified time and version
        String etag = "\"" + post.getUpdatedAt().toEpochSecond(ZoneOffset.UTC) + "-" + post.getId() + "\"";

        // Return 304 Not Modified if ETag matches
        if (etag.equals(ifNoneMatch)) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED)
                .eTag(etag)
                .build();
        }

        return ResponseEntity.ok()
            .eTag(etag)
            .lastModified(post.getUpdatedAt().toInstant(ZoneOffset.UTC))
            .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES)
                .mustRevalidate()
                .noTransform())
            .body(PostDTOV2.from(post));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<List<PostDTOV2>>> getPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {

        Page<Post> posts = postService.findAllPublished(PageRequest.of(page, size));

        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(1, TimeUnit.MINUTES).noTransform())
            .header("Vary", "Accept-Encoding, Accept")  // Cache varies by encoding
            .body(ApiResponse.success(
                posts.getContent().stream().map(PostDTOV2::from).toList(),
                PageMetadata.from(posts)
            ));
    }

    // Static/rarely-changing resources
    @GetMapping("/categories")
    public ResponseEntity<List<CategoryDTO>> getCategories() {
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS)
                .staleWhileRevalidate(Duration.ofMinutes(10))
                .cachePublic())
            .body(categoryService.findAll().stream()
                .map(CategoryDTO::from).toList());
    }
}
```

---

## 11. Complete API with All Features

```java
@RestController
@RequestMapping("/api/v2/posts")
@Tag(name = "Posts V2", description = "Blog post management APIs - v2")
@RequiredArgsConstructor
@Slf4j
public class PostApiV2Controller {

    private final PostService postService;
    private final PostModelAssembler assembler;

    @GetMapping
    @Operation(summary = "Search posts with filtering, sorting, and pagination")
    public ResponseEntity<ApiResponse<List<PostDTOV2>>> searchPosts(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String status,
            @RequestParam(required = false) Long categoryId,
            @RequestParam(required = false) String tag,
            @RequestParam(required = false) Long authorId,
            @RequestParam(required = false) Boolean featured,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
                LocalDateTime fromDate,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
                LocalDateTime toDate,
            @RequestParam(defaultValue = "publishedAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            HttpServletRequest request) {

        PostFilterRequest filter = PostFilterRequest.builder()
            .keyword(keyword)
            .status(status != null ? PostStatus.valueOf(status.toUpperCase()) : PostStatus.PUBLISHED)
            .categoryId(categoryId)
            .tagName(tag)
            .authorId(authorId)
            .featured(featured)
            .fromDate(fromDate)
            .toDate(toDate)
            .sortBy(sortBy)
            .sortDir(sortDir)
            .page(page)
            .size(size)
            .build();

        Page<Post> posts = postService.findPosts(filter);

        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.SECONDS))
            .header("X-Total-Count", String.valueOf(posts.getTotalElements()))
            .body(ApiResponse.success(
                posts.getContent().stream().map(PostDTOV2::from).toList(),
                PageMetadata.from(posts)
            ));
    }

    @GetMapping("/{id}")
    public ResponseEntity<PostDTOV2> getPost(
            @PathVariable Long id,
            @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {

        Post post = postService.findById(id)
            .orElseThrow(() -> new PostNotFoundException("Post not found: " + id));

        String etag = generateEtag(post);

        if (etag.equals(ifNoneMatch)) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED).eTag(etag).build();
        }

        // Async increment view count
        postService.incrementViewCountAsync(id);

        return ResponseEntity.ok()
            .eTag(etag)
            .lastModified(post.getUpdatedAt().toInstant(ZoneOffset.UTC))
            .cacheControl(CacheControl.maxAge(5, TimeUnit.MINUTES))
            .body(PostDTOV2.from(post));
    }

    @PostMapping
    @PreAuthorize("hasAnyRole('AUTHOR', 'ADMIN')")
    public ResponseEntity<PostDTOV2> createPost(
            @Valid @RequestBody CreatePostRequest request,
            @AuthenticationPrincipal UserPrincipal principal) {

        Post post = postService.createPost(request, principal.getUsername());

        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(post.getId())
            .toUri();

        return ResponseEntity.created(location).body(PostDTOV2.from(post));
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @postSecurityService.isOwner(#id, authentication.name)")
    public ResponseEntity<PostDTOV2> updatePost(
            @PathVariable Long id,
            @Valid @RequestBody UpdatePostRequest request) {

        Post post = postService.updatePost(id, request);
        return ResponseEntity.ok(PostDTOV2.from(post));
    }

    @PatchMapping("/{id}/publish")
    @PreAuthorize("hasRole('ADMIN') or @postSecurityService.isOwner(#id, authentication.name)")
    public ResponseEntity<PostDTOV2> publishPost(@PathVariable Long id) {
        Post post = postService.publishPost(id);
        return ResponseEntity.ok(PostDTOV2.from(post));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @postSecurityService.isOwner(#id, authentication.name)")
    public ResponseEntity<Void> deletePost(@PathVariable Long id) {
        postService.deletePost(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/{id}/comments")
    public ResponseEntity<ApiResponse<List<CommentDTO>>> getPostComments(
            @PathVariable Long id,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        Page<Comment> comments = commentService.findByPostId(id, PageRequest.of(page, size));
        return ResponseEntity.ok(ApiResponse.success(
            comments.getContent().stream().map(CommentDTO::from).toList(),
            PageMetadata.from(comments)
        ));
    }

    private String generateEtag(Post post) {
        long lastModified = post.getUpdatedAt().toEpochSecond(ZoneOffset.UTC);
        return "\"" + Integer.toHexString((int) (lastModified ^ post.getId())) + "\"";
    }
}
```

---

## 12. Summary Table

| Feature | Approach | Library/Annotation |
|---|---|---|
| URL versioning | `/api/v1/`, `/api/v2/` | Controller mapping |
| Header versioning | `headers = "X-API-Version=1"` | `@GetMapping` attribute |
| Media type versioning | `produces = "vnd.app.v1+json"` | `@GetMapping` produces |
| HATEOAS links | `RepresentationModel` | Spring HATEOAS |
| API documentation | `@Operation`, `@ApiResponse` | SpringDoc OpenAPI |
| JSON/XML support | `@JacksonXmlRootElement` | Jackson XML |
| Rate limiting | `Bucket`, `Bandwidth` | Bucket4j |
| JSend response | `ApiResponse<T>` | Custom wrapper |
| Problem Details | `ProblemDetail` | Spring 6 built-in |
| Cursor pagination | Opaque Base64 cursor | Custom |
| Offset pagination | `Pageable`, `Page<T>` | Spring Data |
| ETag caching | `ResponseEntity.eTag()` | Spring MVC |
| Cache-Control | `CacheControl.maxAge()` | Spring MVC |

---

## Next Part

**Part 030** covers Spring Testing: unit tests with Mockito, `@WebMvcTest` for controllers, `@DataJpaTest` for repositories, `@SpringBootTest` for integration tests, TestContainers, MockMvc, RestAssured, security testing, and a complete test suite for a REST API.
