# Part 062: API Design and Versioning

## Overview

A well-designed API is a product that developers love to use. This part covers RESTful design principles, versioning strategies, HATEOAS, OpenAPI documentation, and a complete v1-to-v2 migration example. Good API design reduces breaking changes and lets your system evolve gracefully.

---

## Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- OpenAPI 3.0 documentation -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.3.0</version>
    </dependency>
    <!-- HATEOAS -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-hateoas</artifactId>
    </dependency>
    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
</dependencies>
```

---

## 1. RESTful API Design Principles

### Resource Naming Conventions

```java
// GOOD resource naming:
// Collections (plural nouns):
//   GET  /api/v1/orders          - list orders
//   POST /api/v1/orders          - create order
//   GET  /api/v1/orders/{id}     - get one order
//   PUT  /api/v1/orders/{id}     - replace order
//   PATCH /api/v1/orders/{id}    - partial update
//   DELETE /api/v1/orders/{id}   - delete order

// Nested resources (relationships):
//   GET  /api/v1/orders/{id}/items        - get order items
//   POST /api/v1/orders/{id}/items        - add item to order
//   DELETE /api/v1/orders/{id}/items/{itemId}

// Actions (when REST nouns don't fit):
//   POST /api/v1/orders/{id}/cancel       - cancel an order
//   POST /api/v1/orders/{id}/ship         - ship an order
//   POST /api/v1/accounts/{id}/activate   - activate account

// Query parameters for filtering, sorting, pagination:
//   GET /api/v1/orders?status=PENDING&page=0&size=20&sort=createdAt,desc
//   GET /api/v1/products?category=electronics&minPrice=100&maxPrice=500

// BAD naming (avoid):
//   GET  /api/v1/getOrders        - verb in path
//   POST /api/v1/order/create     - verb + singular
//   GET  /api/v1/orders/list      - redundant "list"
//   POST /api/v1/cancelOrder/123  - verb as path

@RestController
@RequestMapping("/api/v1/orders")
@Validated
@Slf4j
public class OrderControllerV1 {

    private final OrderService orderService;

    // GET collection with filtering, sorting, pagination
    @GetMapping
    public ResponseEntity<Page<OrderSummaryDto>> listOrders(
            @RequestParam(required = false) OrderStatus status,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
                LocalDate from,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
                LocalDate to,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir,
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size) {

        Sort sort = Sort.by(
            "asc".equalsIgnoreCase(sortDir) ? Sort.Direction.ASC : Sort.Direction.DESC,
            sortBy
        );
        Pageable pageable = PageRequest.of(page, size, sort);

        OrderFilter filter = OrderFilter.builder()
            .status(status)
            .fromDate(from)
            .toDate(to)
            .build();

        Page<OrderSummaryDto> orders = orderService.findOrders(filter, pageable);

        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(orders.getTotalElements()))
            .body(orders);
    }

    // GET single resource
    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST - create resource
    @PostMapping
    public ResponseEntity<OrderDto> createOrder(
            @RequestBody @Valid CreateOrderRequest request,
            UriComponentsBuilder uriBuilder) {

        OrderDto created = orderService.createOrder(request);

        URI location = uriBuilder
            .path("/api/v1/orders/{id}")
            .buildAndExpand(created.getId())
            .toUri();

        // 201 Created with Location header pointing to new resource
        return ResponseEntity.created(location).body(created);
    }

    // PUT - full replacement
    @PutMapping("/{id}")
    public ResponseEntity<OrderDto> replaceOrder(
            @PathVariable Long id,
            @RequestBody @Valid ReplaceOrderRequest request) {

        if (!orderService.exists(id)) {
            return ResponseEntity.notFound().build();
        }

        OrderDto updated = orderService.replaceOrder(id, request);
        return ResponseEntity.ok(updated);
    }

    // PATCH - partial update
    @PatchMapping("/{id}")
    public ResponseEntity<OrderDto> updateOrder(
            @PathVariable Long id,
            @RequestBody @Valid PatchOrderRequest request) {

        return orderService.patchOrder(id, request)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // DELETE
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        if (!orderService.exists(id)) {
            return ResponseEntity.notFound().build();
        }
        orderService.deleteOrder(id);
        return ResponseEntity.noContent().build();  // 204 No Content
    }

    // Action endpoint (not a CRUD operation)
    @PostMapping("/{id}/cancel")
    public ResponseEntity<OrderDto> cancelOrder(
            @PathVariable Long id,
            @RequestBody @Valid CancelOrderRequest request) {

        try {
            OrderDto cancelled = orderService.cancelOrder(id, request.getReason());
            return ResponseEntity.ok(cancelled);
        } catch (OrderAlreadyCompletedException e) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(null);  // Or return error body
        }
    }

    // Nested resource
    @GetMapping("/{id}/items")
    public ResponseEntity<List<OrderItemDto>> getOrderItems(@PathVariable Long id) {
        List<OrderItemDto> items = orderService.getOrderItems(id);
        return ResponseEntity.ok(items);
    }

    @PostMapping("/{id}/items")
    public ResponseEntity<OrderItemDto> addOrderItem(
            @PathVariable Long id,
            @RequestBody @Valid AddOrderItemRequest request,
            UriComponentsBuilder uriBuilder) {

        OrderItemDto item = orderService.addItem(id, request);

        URI location = uriBuilder
            .path("/api/v1/orders/{orderId}/items/{itemId}")
            .buildAndExpand(id, item.getId())
            .toUri();

        return ResponseEntity.created(location).body(item);
    }
}
```

---

## 2. HTTP Methods Semantics

```java
// HTTP Method semantics reference:
//
// GET:    Safe + Idempotent. No side effects. Cacheable.
// POST:   Not safe, not idempotent. Creates resources.
// PUT:    Not safe, idempotent. Full replacement.
// PATCH:  Not safe, not necessarily idempotent. Partial update.
// DELETE: Not safe, idempotent. Removes resource.
// HEAD:   Like GET but no body. Check if resource exists.
// OPTIONS: Describe what's supported. Used for CORS preflight.

// Idempotency example
@PutMapping("/{id}")
public ResponseEntity<OrderDto> replaceOrder(@PathVariable Long id,
        @RequestBody ReplaceOrderRequest request) {
    // Safe to call multiple times - same result each time
    OrderDto result = orderService.replace(id, request);
    return ResponseEntity.ok(result);
}

// POST is not idempotent - use idempotency keys to make it safe
@PostMapping
public ResponseEntity<OrderDto> createOrder(
        @RequestBody CreateOrderRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {

    // Check if this request was already processed
    if (idempotencyKey != null) {
        Optional<OrderDto> existing = idempotencyService.findResponse(idempotencyKey);
        if (existing.isPresent()) {
            return ResponseEntity.ok()
                .header("Idempotency-Result", "DUPLICATE")
                .body(existing.get());
        }
    }

    OrderDto created = orderService.createOrder(request);

    if (idempotencyKey != null) {
        idempotencyService.saveResponse(idempotencyKey, created);
    }

    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
```

---

## 3. API Versioning Strategies

### Strategy 1: URL Path Versioning

```java
// Most common and visible approach
// v1 controller
@RestController
@RequestMapping("/api/v1/products")
public class ProductControllerV1 {

    @GetMapping("/{id}")
    public ResponseEntity<ProductDtoV1> getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);
        return ResponseEntity.ok(ProductDtoV1.from(product));
    }
}

// v2 controller with breaking changes
@RestController
@RequestMapping("/api/v2/products")
public class ProductControllerV2 {

    @GetMapping("/{id}")
    public ResponseEntity<ProductDtoV2> getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);
        return ResponseEntity.ok(ProductDtoV2.from(product));
    }
}
```

### Strategy 2: Header Versioning

```java
// Version specified in Accept or custom header
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping(value = "/{id}",
        headers = "X-API-Version=1")
    public ResponseEntity<ProductDtoV1> getProductV1(@PathVariable Long id) {
        return ResponseEntity.ok(ProductDtoV1.from(productService.findById(id)));
    }

    @GetMapping(value = "/{id}",
        headers = "X-API-Version=2")
    public ResponseEntity<ProductDtoV2> getProductV2(@PathVariable Long id) {
        return ResponseEntity.ok(ProductDtoV2.from(productService.findById(id)));
    }
}
```

### Strategy 3: Media Type (Content Negotiation) Versioning

```java
// Version encoded in media type
// GET /api/products/1
// Accept: application/vnd.example.product.v1+json

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private static final String MEDIA_TYPE_V1 = "application/vnd.example.product.v1+json";
    private static final String MEDIA_TYPE_V2 = "application/vnd.example.product.v2+json";

    @GetMapping(value = "/{id}", produces = MEDIA_TYPE_V1)
    public ResponseEntity<ProductDtoV1> getProductV1(@PathVariable Long id) {
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(MEDIA_TYPE_V1))
            .body(ProductDtoV1.from(productService.findById(id)));
    }

    @GetMapping(value = "/{id}", produces = MEDIA_TYPE_V2)
    public ResponseEntity<ProductDtoV2> getProductV2(@PathVariable Long id) {
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(MEDIA_TYPE_V2))
            .body(ProductDtoV2.from(productService.findById(id)));
    }
}
```

---

## 4. Implementing Versioning in Spring Boot (Custom Annotation Approach)

```java
// Custom @ApiVersion annotation
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ApiVersion {
    int[] value();  // Supported versions
}

// Custom RequestMappingHandlerMapping for annotation-based versioning
@Component
public class ApiVersionRequestMappingHandlerMapping
        extends RequestMappingHandlerMapping {

    @Override
    protected RequestCondition<?> getCustomTypeCondition(Class<?> handlerType) {
        ApiVersion typeAnnotation = AnnotationUtils.findAnnotation(
            handlerType, ApiVersion.class);
        return createCondition(typeAnnotation);
    }

    @Override
    protected RequestCondition<?> getCustomMethodCondition(Method method) {
        ApiVersion methodAnnotation = AnnotationUtils.findAnnotation(
            method, ApiVersion.class);
        return createCondition(methodAnnotation);
    }

    private ApiVersionCondition createCondition(ApiVersion annotation) {
        return annotation != null ? new ApiVersionCondition(annotation.value()) : null;
    }
}

// Custom RequestCondition
public class ApiVersionCondition
        implements RequestCondition<ApiVersionCondition> {

    private final int[] supportedVersions;

    public ApiVersionCondition(int[] versions) {
        this.supportedVersions = Arrays.copyOf(versions, versions.length);
        Arrays.sort(this.supportedVersions);
    }

    @Override
    public ApiVersionCondition combine(ApiVersionCondition other) {
        // Method-level annotation takes precedence over class-level
        return other;
    }

    @Override
    public ApiVersionCondition getMatchingCondition(HttpServletRequest request) {
        int requestedVersion = extractVersion(request);
        for (int version : supportedVersions) {
            if (requestedVersion == version) {
                return this;
            }
        }
        return null;
    }

    @Override
    public int compareTo(ApiVersionCondition other, HttpServletRequest request) {
        // Prefer higher version numbers
        return other.supportedVersions[other.supportedVersions.length - 1]
            - this.supportedVersions[this.supportedVersions.length - 1];
    }

    private int extractVersion(HttpServletRequest request) {
        String versionHeader = request.getHeader("X-API-Version");
        if (versionHeader != null) {
            try {
                return Integer.parseInt(versionHeader.trim());
            } catch (NumberFormatException e) {
                return 1;
            }
        }

        // Extract from URL path: /api/v2/...
        String path = request.getRequestURI();
        Pattern pattern = Pattern.compile("/api/v(\\d+)/");
        Matcher matcher = pattern.matcher(path);
        if (matcher.find()) {
            return Integer.parseInt(matcher.group(1));
        }

        return 1; // Default to v1
    }
}

// Use the annotation
@RestController
@RequestMapping("/api/v{version}/products")
@ApiVersion({1, 2})  // This controller handles both v1 and v2
public class ProductController {

    @GetMapping("/{id}")
    @ApiVersion({1})
    public ProductDtoV1 getProductV1(@PathVariable Long id) {
        return ProductDtoV1.from(productService.findById(id));
    }

    @GetMapping("/{id}")
    @ApiVersion({2})
    public ProductDtoV2 getProductV2(@PathVariable Long id) {
        return ProductDtoV2.from(productService.findById(id));
    }
}
```

---

## 5. Backward Compatibility Rules

```java
// V1 DTO - original
public record ProductDtoV1(
    Long id,
    String name,
    BigDecimal price,
    String description
) {
    public static ProductDtoV1 from(Product product) {
        return new ProductDtoV1(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getDescription()
        );
    }
}

// V2 DTO - added fields, changed structure
// BACKWARD COMPATIBLE changes:
//   - Adding optional fields (nullable or with defaults)
//   - Adding new endpoints
//   - Relaxing validation rules
//
// BREAKING changes (require new version):
//   - Removing fields
//   - Renaming fields
//   - Changing field types
//   - Changing response structure
//   - Adding required fields to requests
//   - Changing status codes
//   - Tightening validation

public record ProductDtoV2(
    Long id,
    String name,
    BigDecimal price,
    String description,
    // New in V2:
    CategoryDto category,       // Was a String categoryName in V1
    List<String> tags,          // New field
    double rating,              // New field
    String sku,                 // New field
    ProductStatus status,       // Was inferred from other fields in V1
    LocalDateTime createdAt,    // New field
    LocalDateTime updatedAt     // New field
) {
    public static ProductDtoV2 from(Product product) {
        return new ProductDtoV2(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getDescription(),
            CategoryDto.from(product.getCategory()),
            product.getTags(),
            product.getAverageRating(),
            product.getSku(),
            product.getStatus(),
            product.getCreatedAt(),
            product.getUpdatedAt()
        );
    }
}

// Deprecation annotation on V1 endpoints
@Deprecated(since = "2024-01", forRemoval = true)
// Planned removal: 2025-01
@GetMapping("/{id}")
@ApiVersion({1})
public ResponseEntity<ProductDtoV1> getProductV1(@PathVariable Long id) {
    ProductDtoV1 product = ProductDtoV1.from(productService.findById(id));

    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Sat, 01 Jan 2025 00:00:00 GMT")
        .header("Link", "</api/v2/products/" + id + ">; rel=\"successor-version\"")
        .body(product);
}
```

---

## 6. HATEOAS with Spring

```java
// HAL (Hypertext Application Language) response
@RestController
@RequestMapping("/api/v2/orders")
public class HateoasOrderController {

    private final OrderService orderService;
    private final OrderModelAssembler assembler;

    @GetMapping("/{id}")
    public EntityModel<OrderDto> getOrder(@PathVariable Long id) {
        OrderDto order = orderService.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));

        return assembler.toModel(order);
    }

    @GetMapping
    public CollectionModel<EntityModel<OrderDto>> listOrders(Pageable pageable) {
        Page<OrderDto> orders = orderService.findAll(pageable);

        List<EntityModel<OrderDto>> orderModels = orders.stream()
            .map(assembler::toModel)
            .collect(Collectors.toList());

        // Include pagination links
        Link selfLink = linkTo(methodOn(HateoasOrderController.class)
            .listOrders(pageable)).withSelfRel();

        return CollectionModel.of(orderModels,
            selfLink,
            Link.of(selfLink.getHref() + "?page=" + orders.getNumber() + "&size=" + orders.getSize())
                .withSelfRel()
        );
    }
}

// Model assembler with links
@Component
public class OrderModelAssembler
        implements RepresentationModelAssembler<OrderDto, EntityModel<OrderDto>> {

    @Override
    public EntityModel<OrderDto> toModel(OrderDto order) {
        List<Link> links = new ArrayList<>();

        // Self link
        links.add(linkTo(methodOn(HateoasOrderController.class)
            .getOrder(order.getId())).withSelfRel());

        // Links to related resources
        links.add(linkTo(methodOn(HateoasOrderController.class)
            .getOrderItems(order.getId())).withRel("items"));

        // Conditional links based on state
        if (order.getStatus() == OrderStatus.PENDING) {
            links.add(linkTo(methodOn(HateoasOrderController.class)
                .cancelOrder(order.getId(), null)).withRel("cancel"));

            links.add(linkTo(methodOn(HateoasOrderController.class)
                .confirmOrder(order.getId())).withRel("confirm"));
        }

        if (order.getStatus() == OrderStatus.SHIPPED) {
            links.add(Link.of("/api/v2/shipments/" + order.getShipmentId())
                .withRel("shipment"));
        }

        // Link to customer
        links.add(Link.of("/api/v2/customers/" + order.getCustomerId())
            .withRel("customer"));

        return EntityModel.of(order, links);
    }
}

// Response structure (HAL JSON):
// {
//   "id": 123,
//   "status": "PENDING",
//   "totalAmount": 149.99,
//   "_links": {
//     "self": { "href": "/api/v2/orders/123" },
//     "items": { "href": "/api/v2/orders/123/items" },
//     "cancel": { "href": "/api/v2/orders/123/cancel" },
//     "confirm": { "href": "/api/v2/orders/123/confirm" },
//     "customer": { "href": "/api/v2/customers/456" }
//   }
// }
```

---

## 7. OpenAPI 3.0 Specification with Springdoc

```java
// OpenAPI configuration
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("E-Commerce API")
                .version("v2.0.0")
                .description("RESTful API for the E-Commerce platform")
                .contact(new Contact()
                    .name("API Team")
                    .email("api@example.com")
                    .url("https://docs.example.com"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .externalDocs(new ExternalDocumentation()
                .description("Full API Documentation")
                .url("https://docs.example.com/api"))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")
                        .description("JWT authentication token"))
                .addSchemas("Error",
                    new Schema<>()
                        .type("object")
                        .addProperty("code", new Schema<>().type("string"))
                        .addProperty("message", new Schema<>().type("string"))
                        .addProperty("details", new Schema<>().type("array"))))
            .addTagsItem(new Tag()
                .name("Orders")
                .description("Order management operations"))
            .addTagsItem(new Tag()
                .name("Products")
                .description("Product catalog operations"));
    }
}

// Documented controller
@RestController
@RequestMapping("/api/v2/products")
@Tag(name = "Products", description = "Product catalog management")
@Slf4j
public class DocumentedProductController {

    @Operation(
        summary = "Get product by ID",
        description = "Returns a single product with full details including category and ratings",
        responses = {
            @ApiResponse(
                responseCode = "200",
                description = "Product found",
                content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = ProductDtoV2.class)
                )
            ),
            @ApiResponse(
                responseCode = "404",
                description = "Product not found",
                content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(ref = "#/components/schemas/Error")
                )
            )
        }
    )
    @GetMapping("/{id}")
    public ResponseEntity<ProductDtoV2> getProduct(
            @Parameter(
                description = "Product ID",
                required = true,
                example = "12345"
            )
            @PathVariable Long id) {

        return productService.findById(id)
            .map(product -> ResponseEntity.ok(ProductDtoV2.from(product)))
            .orElse(ResponseEntity.notFound().build());
    }

    @Operation(
        summary = "Search products",
        description = "Search and filter products with pagination support"
    )
    @Parameters({
        @Parameter(name = "q", description = "Search text", example = "laptop"),
        @Parameter(name = "category", description = "Category ID"),
        @Parameter(name = "minPrice", description = "Minimum price", example = "100.00"),
        @Parameter(name = "maxPrice", description = "Maximum price", example = "999.99"),
        @Parameter(name = "page", description = "Page number (0-based)", example = "0"),
        @Parameter(name = "size", description = "Page size (max 100)", example = "20")
    })
    @GetMapping
    public ResponseEntity<Page<ProductDtoV2>> searchProducts(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) Long category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {

        Page<ProductDtoV2> products = productService.search(
            q, category, minPrice, maxPrice, pageable);
        return ResponseEntity.ok(products);
    }

    @Operation(summary = "Create a new product")
    @SecurityRequirement(name = "bearerAuth")
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDtoV2> createProduct(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Product details",
                required = true,
                content = @Content(schema = @Schema(implementation = CreateProductRequest.class))
            )
            @RequestBody @Valid CreateProductRequest request,
            UriComponentsBuilder uriBuilder) {

        ProductDtoV2 created = productService.create(request);
        URI location = uriBuilder.path("/api/v2/products/{id}")
            .buildAndExpand(created.getId())
            .toUri();
        return ResponseEntity.created(location).body(created);
    }
}

// Schema annotations on DTOs
@Schema(description = "Product details")
public record ProductDtoV2(
    @Schema(description = "Unique product ID", example = "12345")
    Long id,

    @Schema(description = "Product name", example = "Gaming Laptop Pro")
    String name,

    @Schema(description = "Price in USD", example = "1299.99")
    BigDecimal price,

    @Schema(description = "Product category")
    CategoryDto category,

    @Schema(description = "Average customer rating (0-5)", example = "4.5")
    double rating,

    @Schema(description = "Stock-keeping unit", example = "LAPTOP-PRO-001")
    String sku,

    @Schema(description = "Product availability status")
    ProductStatus status
) {}

// application.yml for OpenAPI
// springdoc:
//   api-docs:
//     path: /api-docs
//   swagger-ui:
//     path: /swagger-ui.html
//     operations-sorter: method
//     tags-sorter: alpha
//     display-request-duration: true
//   default-produces-media-type: application/json
//   show-actuator: false
```

---

## 8. API Versioning with Deprecation Headers

```java
// Deprecation interceptor - adds deprecation headers to V1 responses
@Component
public class ApiDeprecationInterceptor implements HandlerInterceptor {

    private final Map<String, DeprecationInfo> deprecations = Map.of(
        "/api/v1/", new DeprecationInfo(
            "2024-01-01",
            "2025-01-01",
            "https://docs.example.com/api/v2/migration"
        )
    );

    @Override
    public boolean preHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler) {

        String path = request.getRequestURI();

        deprecations.entrySet().stream()
            .filter(e -> path.startsWith(e.getKey()))
            .findFirst()
            .ifPresent(e -> addDeprecationHeaders(response, e.getValue()));

        return true;
    }

    private void addDeprecationHeaders(HttpServletResponse response,
            DeprecationInfo info) {
        response.setHeader("Deprecation", info.deprecatedSince());
        response.setHeader("Sunset", info.sunsetDate());
        response.setHeader("Link",
            info.migrationGuideUrl() + "; rel=\"successor-version\"");
    }

    record DeprecationInfo(
        String deprecatedSince,
        String sunsetDate,
        String migrationGuideUrl
    ) {}
}
```

---

## 9. Real Example: API v1 to v2 Migration with Compatibility

```java
// V1 Customer model (original)
public record CustomerDtoV1(
    Long id,
    String firstName,
    String lastName,
    String email,
    String phone,
    String address      // Single string: "123 Main St, NYC, NY 10001"
) {}

// V2 Customer model (improved)
public record CustomerDtoV2(
    Long id,
    String fullName,            // CHANGED: firstName + lastName merged
    String displayName,         // NEW
    String email,
    String phone,
    AddressDto billingAddress,  // CHANGED: structured address
    AddressDto shippingAddress, // NEW
    String tier,                // NEW: BRONZE, SILVER, GOLD
    LocalDateTime memberSince,  // NEW
    boolean emailVerified       // NEW
) {}

public record AddressDto(
    String street,
    String city,
    String state,
    String postalCode,
    String country
) {}

// Compatibility layer: V1 request accepts old format, internally uses V2
@RestController
@RequestMapping("/api")
@Slf4j
public class CustomerCompatibilityController {

    private final CustomerServiceV2 customerService;
    private final CustomerMigrationConverter converter;

    // V1 endpoint: still works, returns V1 format
    @GetMapping("/v1/customers/{id}")
    @Deprecated
    public ResponseEntity<CustomerDtoV1> getCustomerV1(@PathVariable Long id) {
        CustomerDtoV2 v2 = customerService.findById(id);
        CustomerDtoV1 v1 = converter.downgradeToV1(v2);

        return ResponseEntity.ok()
            .header("Deprecation", "true")
            .header("Sunset", "2025-01-01T00:00:00Z")
            .header("Link", "/api/v2/customers/" + id + "; rel=\"successor-version\"")
            .body(v1);
    }

    // V2 endpoint: new format
    @GetMapping("/v2/customers/{id}")
    public ResponseEntity<CustomerDtoV2> getCustomerV2(@PathVariable Long id) {
        return ResponseEntity.ok(customerService.findById(id));
    }

    // V1 create: accepts old format
    @PostMapping("/v1/customers")
    @Deprecated
    public ResponseEntity<CustomerDtoV1> createCustomerV1(
            @RequestBody @Valid CreateCustomerV1Request request,
            UriComponentsBuilder uriBuilder) {

        // Convert V1 request to V2 format
        CreateCustomerV2Request v2Request = converter.upgradeCreateRequest(request);
        CustomerDtoV2 created = customerService.create(v2Request);
        CustomerDtoV1 v1Response = converter.downgradeToV1(created);

        URI location = uriBuilder.path("/api/v1/customers/{id}")
            .buildAndExpand(created.getId()).toUri();

        return ResponseEntity.created(location).body(v1Response);
    }

    // V2 create: new format with structured address
    @PostMapping("/v2/customers")
    public ResponseEntity<CustomerDtoV2> createCustomerV2(
            @RequestBody @Valid CreateCustomerV2Request request,
            UriComponentsBuilder uriBuilder) {

        CustomerDtoV2 created = customerService.create(request);
        URI location = uriBuilder.path("/api/v2/customers/{id}")
            .buildAndExpand(created.getId()).toUri();

        return ResponseEntity.created(location).body(created);
    }
}

// Converter: translates between V1 and V2 formats
@Component
public class CustomerMigrationConverter {

    // V2 -> V1 (downgrade for backward compatibility)
    public CustomerDtoV1 downgradeToV1(CustomerDtoV2 v2) {
        String[] nameParts = splitFullName(v2.fullName());

        String legacyAddress = v2.billingAddress() != null
            ? formatAddressV1(v2.billingAddress())
            : null;

        return new CustomerDtoV1(
            v2.id(),
            nameParts.length > 0 ? nameParts[0] : "",
            nameParts.length > 1 ? nameParts[1] : "",
            v2.email(),
            v2.phone(),
            legacyAddress
        );
    }

    // V1 Create -> V2 Create (upgrade for new system)
    public CreateCustomerV2Request upgradeCreateRequest(CreateCustomerV1Request v1) {
        AddressDto address = v1.address() != null
            ? parseAddressV1(v1.address())
            : null;

        return new CreateCustomerV2Request(
            v1.firstName() + " " + v1.lastName(),
            v1.email(),
            v1.phone(),
            address,
            address  // Use same address for both billing/shipping
        );
    }

    private String[] splitFullName(String fullName) {
        if (fullName == null) return new String[]{};
        int lastSpace = fullName.lastIndexOf(' ');
        if (lastSpace == -1) return new String[]{fullName};
        return new String[]{fullName.substring(0, lastSpace),
                            fullName.substring(lastSpace + 1)};
    }

    private String formatAddressV1(AddressDto address) {
        return String.format("%s, %s, %s %s",
            address.street(), address.city(), address.state(), address.postalCode());
    }

    private AddressDto parseAddressV1(String legacyAddress) {
        // Parse "123 Main St, New York, NY 10001"
        String[] parts = legacyAddress.split(",\\s*");
        if (parts.length < 3) {
            return new AddressDto(legacyAddress, "", "", "", "US");
        }

        String street = parts[0].trim();
        String city = parts[1].trim();
        String[] statePostal = parts[2].trim().split("\\s+");
        String state = statePostal.length > 0 ? statePostal[0] : "";
        String postal = statePostal.length > 1 ? statePostal[1] : "";

        return new AddressDto(street, city, state, postal, "US");
    }
}

// API Changelog endpoint
@RestController
@RequestMapping("/api/changelog")
public class ApiChangelogController {

    @GetMapping
    public ResponseEntity<List<ApiChangelogEntry>> getChangelog() {
        return ResponseEntity.ok(List.of(
            new ApiChangelogEntry(
                "v2.0.0",
                "2024-01-15",
                "BREAKING",
                List.of(
                    "Customer name split into firstName/lastName is now merged into fullName",
                    "Address is now a structured object instead of a string",
                    "Added billingAddress and shippingAddress fields"
                )
            ),
            new ApiChangelogEntry(
                "v1.5.0",
                "2023-10-01",
                "NON_BREAKING",
                List.of(
                    "Added email verification status to customer response",
                    "Added customer tier (BRONZE/SILVER/GOLD)"
                )
            ),
            new ApiChangelogEntry(
                "v1.0.0",
                "2023-06-01",
                "INITIAL",
                List.of("Initial API release")
            )
        ));
    }
}

public record ApiChangelogEntry(
    String version,
    String releaseDate,
    String changeType,
    List<String> changes
) {}
```

---

## 10. Global Exception Handling

```java
// Consistent error responses across all API versions
@RestControllerAdvice
@Slf4j
public class GlobalApiExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getRejectedValue(), e.getDefaultMessage()))
            .collect(Collectors.toList());

        ApiError error = ApiError.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .code("VALIDATION_FAILED")
            .message("Request validation failed")
            .fieldErrors(fieldErrors)
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        ApiError error = ApiError.builder()
            .status(HttpStatus.NOT_FOUND.value())
            .code("RESOURCE_NOT_FOUND")
            .message(ex.getMessage())
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiError> handleAccessDenied(AccessDeniedException ex) {
        ApiError error = ApiError.builder()
            .status(HttpStatus.FORBIDDEN.value())
            .code("ACCESS_DENIED")
            .message("You don't have permission to perform this action")
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ApiError> handleBadRequest(HttpMessageNotReadableException ex) {
        ApiError error = ApiError.builder()
            .status(HttpStatus.BAD_REQUEST.value())
            .code("MALFORMED_REQUEST")
            .message("Request body is malformed or missing")
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);

        ApiError error = ApiError.builder()
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .code("INTERNAL_ERROR")
            .message("An unexpected error occurred. Please try again.")
            .timestamp(Instant.now())
            .build();

        return ResponseEntity.internalServerError().body(error);
    }
}

@Data
@Builder
public class ApiError {
    private int status;
    private String code;
    private String message;
    private List<FieldError> fieldErrors;
    private Instant timestamp;
    private String traceId;

    public record FieldError(String field, Object rejectedValue, String message) {}
}
```

---

## Summary

| Versioning Strategy | URL Example | Pros | Cons |
|--------------------|-------------|------|------|
| URL Path | `/api/v2/products` | Visible, easy to test | URL pollution |
| Header | `X-API-Version: 2` | Clean URLs | Less discoverable |
| Media Type | `Accept: application/vnd.v2+json` | Standards-based | Complex |
| Query Param | `/api/products?version=2` | Simple | Pollutes query string |

| Feature | Tool |
|---------|------|
| Documentation | Springdoc OpenAPI + Swagger UI |
| HATEOAS | spring-boot-starter-hateoas (HAL) |
| Validation | spring-boot-starter-validation |
| Backward compat | Converter/adapter pattern |
| Deprecation | Response headers (Deprecation, Sunset, Link) |

## Next Part Preview

**Part 063: Scheduled Tasks and Background Jobs** — @Scheduled, dynamic scheduling, ShedLock for distributed systems, Spring Batch, and Quartz Scheduler with real-world examples.
