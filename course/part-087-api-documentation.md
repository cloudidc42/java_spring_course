# Part 087: API Documentation and OpenAPI

## Overview

OpenAPI 3.0 is the industry standard for documenting REST APIs. Springdoc OpenAPI generates the specification automatically from your code, while annotations fine-tune what consumers see. This part builds a fully documented REST API with security schemes, examples, and client SDK generation.

---

## 1. Springdoc OpenAPI Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

```yaml
# application.yml
springdoc:
  api-docs:
    enabled: true
    path: /v3/api-docs
  swagger-ui:
    enabled: true
    path: /swagger-ui.html
    operations-sorter: alpha
    tags-sorter: alpha
    try-it-out-enabled: true
    filter: true
    display-request-duration: true
  show-actuator: false
  packages-to-scan: com.example.api.controllers
  paths-to-match: /api/**

# Disable in production
springdoc:
  api-docs:
    enabled: ${OPENAPI_ENABLED:true}
```

```java
// Global OpenAPI configuration
@Configuration
public class OpenApiConfiguration {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("E-Commerce API")
                        .description("RESTful API for the e-commerce platform. " +
                                "All dates are in ISO 8601 format. " +
                                "All monetary values are in USD unless otherwise specified.")
                        .version("v2.1.0")
                        .contact(new Contact()
                                .name("Platform Team")
                                .email("platform@example.com")
                                .url("https://developer.example.com"))
                        .license(new License()
                                .name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Full documentation")
                        .url("https://docs.example.com/api"))
                .addServersItem(new Server()
                        .url("https://api.example.com")
                        .description("Production"))
                .addServersItem(new Server()
                        .url("https://staging-api.example.com")
                        .description("Staging"))
                .addServersItem(new Server()
                        .url("http://localhost:8080")
                        .description("Local Development"))
                .addSecurityItem(new SecurityRequirement().addList("BearerAuth"))
                .components(new Components()
                        .addSecuritySchemes("BearerAuth", new SecurityScheme()
                                .type(SecurityScheme.Type.HTTP)
                                .scheme("bearer")
                                .bearerFormat("JWT")
                                .description("Enter JWT token obtained from /auth/login"))
                        .addSecuritySchemes("ApiKey", new SecurityScheme()
                                .type(SecurityScheme.Type.APIKEY)
                                .in(SecurityScheme.In.HEADER)
                                .name("X-API-Key")
                                .description("API key for service-to-service authentication")));
    }
}
```

---

## 2. @Operation, @ApiResponse, @Schema Annotations

```java
@RestController
@RequestMapping("/api/v1/products")
@Tag(name = "Products", description = "Product catalog management")
@Slf4j
public class ProductController {

    private final ProductService productService;

    @Operation(
        summary = "List all products",
        description = "Returns a paginated list of products. Supports filtering by category, price range, and availability.",
        operationId = "listProducts",
        tags = {"Products"}
    )
    @ApiResponses({
        @ApiResponse(
            responseCode = "200",
            description = "Products retrieved successfully",
            content = @Content(
                mediaType = MediaType.APPLICATION_JSON_VALUE,
                schema = @Schema(implementation = ProductPageResponse.class),
                examples = @ExampleObject(
                    name = "Example response",
                    value = """
                            {
                              "content": [
                                {
                                  "id": 1,
                                  "name": "Laptop Pro X1",
                                  "price": 1299.99,
                                  "stock": 45,
                                  "category": "electronics"
                                }
                              ],
                              "page": 0,
                              "size": 20,
                              "totalElements": 150,
                              "totalPages": 8
                            }
                            """
                )
            )
        ),
        @ApiResponse(responseCode = "400", description = "Invalid query parameters",
                content = @Content(schema = @Schema(implementation = ApiError.class))),
        @ApiResponse(responseCode = "401", description = "Authentication required"),
        @ApiResponse(responseCode = "500", description = "Internal server error",
                content = @Content(schema = @Schema(implementation = ApiError.class)))
    })
    @GetMapping
    public ResponseEntity<Page<ProductDto>> listProducts(
            @Parameter(description = "Filter by category", example = "electronics")
            @RequestParam(required = false) String category,

            @Parameter(description = "Minimum price (inclusive)", example = "10.00")
            @RequestParam(required = false) BigDecimal minPrice,

            @Parameter(description = "Maximum price (inclusive)", example = "999.99")
            @RequestParam(required = false) BigDecimal maxPrice,

            @Parameter(description = "Filter by availability")
            @RequestParam(required = false, defaultValue = "true") boolean inStock,

            @Parameter(hidden = true)  // hide from docs
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {

        return ResponseEntity.ok(productService.findAll(
                new ProductFilter(category, minPrice, maxPrice, inStock), pageable));
    }

    @Operation(
        summary = "Get product by ID",
        description = "Retrieves a single product by its unique identifier."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Product found",
                content = @Content(schema = @Schema(implementation = ProductDto.class))),
        @ApiResponse(responseCode = "404", description = "Product not found",
                content = @Content(
                    schema = @Schema(implementation = ApiError.class),
                    examples = @ExampleObject(value = """
                            {"code": "PRODUCT_NOT_FOUND", "message": "Product with ID 99 not found"}
                            """)))
    })
    @GetMapping("/{id}")
    public ResponseEntity<ProductDto> getProduct(
            @Parameter(description = "Product ID", required = true, example = "42")
            @PathVariable Long id) {
        return ResponseEntity.ok(productService.findById(id));
    }

    @Operation(
        summary = "Create a new product",
        description = "Creates a new product in the catalog. Requires ADMIN role.",
        security = @SecurityRequirement(name = "BearerAuth")
    )
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Product created",
                headers = @Header(name = "Location",
                        description = "URL of the created product",
                        schema = @Schema(type = "string", example = "/api/v1/products/42"))),
        @ApiResponse(responseCode = "400", description = "Invalid request body"),
        @ApiResponse(responseCode = "403", description = "Insufficient permissions")
    })
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDto> createProduct(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Product details",
                required = true,
                content = @Content(
                    schema = @Schema(implementation = CreateProductRequest.class),
                    examples = {
                        @ExampleObject(name = "Electronics example",
                            value = """
                                    {
                                      "name": "Wireless Headphones",
                                      "price": 199.99,
                                      "stock": 100,
                                      "category": "electronics",
                                      "sku": "ELEC-00123"
                                    }
                                    """),
                        @ExampleObject(name = "Books example",
                            value = """
                                    {
                                      "name": "Clean Code",
                                      "price": 39.99,
                                      "stock": 250,
                                      "category": "books",
                                      "sku": "BOOK-00456"
                                    }
                                    """)
                    }
                )
            )
            @Valid @RequestBody CreateProductRequest request) {

        ProductDto created = productService.create(request);
        URI location = URI.create("/api/v1/products/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @Operation(summary = "Update product", security = @SecurityRequirement(name = "BearerAuth"))
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDto> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        return ResponseEntity.ok(productService.update(id, request));
    }

    @Operation(
        summary = "Delete product",
        description = "Soft-deletes a product. Deleted products are not shown in listings but order history is preserved.",
        security = @SecurityRequirement(name = "BearerAuth")
    )
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteProduct(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

---

## 3. Schema Annotations

```java
@Schema(
    name = "Product",
    description = "Represents a product in the catalog"
)
public class ProductDto {

    @Schema(description = "Unique product identifier", example = "42", accessMode = Schema.AccessMode.READ_ONLY)
    private Long id;

    @Schema(description = "Product display name", example = "Wireless Headphones", minLength = 3, maxLength = 100)
    private String name;

    @Schema(description = "Current retail price in USD", example = "199.99",
            minimum = "0.01", maximum = "999999.99")
    private BigDecimal price;

    @Schema(description = "Available units in stock", example = "150", minimum = "0")
    private Integer stock;

    @Schema(description = "Product category", example = "electronics",
            allowableValues = {"electronics", "books", "clothing", "home", "sports"})
    private String category;

    @Schema(description = "Stock-keeping unit", example = "ELEC-00123",
            pattern = "^[A-Z]{2,4}-\\d{4,6}$")
    private String sku;

    @Schema(description = "Whether the product is available for purchase",
            example = "true", accessMode = Schema.AccessMode.READ_ONLY)
    private boolean active;

    @Schema(description = "Timestamp when product was created", example = "2024-01-15T10:30:00Z",
            accessMode = Schema.AccessMode.READ_ONLY)
    private Instant createdAt;

    @Schema(hidden = true)  // internal field, not shown in docs
    private Long internalCategoryId;
}

@Schema(description = "Request to create a new product")
public class CreateProductRequest {

    @Schema(description = "Product name", required = true, example = "Wireless Headphones")
    @NotBlank
    private String name;

    @Schema(description = "Price in USD", required = true, example = "199.99")
    @NotNull
    @DecimalMin("0.01")
    private BigDecimal price;

    @Schema(description = "Initial stock quantity", required = true, example = "100")
    @NotNull
    @Min(0)
    private Integer stock;

    @Schema(description = "Product category", required = true, example = "electronics")
    @NotBlank
    private String category;
}

// Enum documentation
@Schema(description = "Order status values")
public enum OrderStatus {

    @Schema(description = "Order placed but not yet confirmed")
    PENDING,

    @Schema(description = "Payment confirmed and order is being processed")
    CONFIRMED,

    @Schema(description = "Order has been shipped to the customer")
    SHIPPED,

    @Schema(description = "Order received by the customer")
    DELIVERED,

    @Schema(description = "Order was cancelled")
    CANCELLED
}
```

---

## 4. Multiple Response Schemas with @Content

```java
@Operation(summary = "Process payment")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Payment processed",
        content = {
            @Content(mediaType = "application/json",
                schema = @Schema(implementation = PaymentSuccessResponse.class)),
            @Content(mediaType = "application/xml",
                schema = @Schema(implementation = PaymentSuccessResponse.class))
        }),
    @ApiResponse(responseCode = "402", description = "Payment failed",
        content = @Content(
            mediaType = "application/json",
            schema = @Schema(oneOf = {
                CardDeclinedError.class,
                InsufficientFundsError.class,
                FraudDetectedError.class
            })
        ))
})
@PostMapping("/payments")
public ResponseEntity<?> processPayment(@Valid @RequestBody PaymentRequest request) {
    return ResponseEntity.ok(paymentService.process(request));
}
```

---

## 5. Security Schemes in OpenAPI

```java
// Different security schemes for different endpoints
@Configuration
public class SecuritySchemeConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
                .components(new Components()
                        // OAuth2 scheme
                        .addSecuritySchemes("OAuth2", new SecurityScheme()
                                .type(SecurityScheme.Type.OAUTH2)
                                .flows(new OAuthFlows()
                                        .authorizationCode(new OAuthFlow()
                                                .authorizationUrl("http://localhost:9000/oauth2/authorize")
                                                .tokenUrl("http://localhost:9000/oauth2/token")
                                                .scopes(new Scopes()
                                                        .addString("openid", "OpenID Connect")
                                                        .addString("profile", "User profile")
                                                        .addString("orders:read", "Read orders")
                                                        .addString("orders:write", "Create/modify orders")))
                                        .clientCredentials(new OAuthFlow()
                                                .tokenUrl("http://localhost:9000/oauth2/token")
                                                .scopes(new Scopes()
                                                        .addString("inventory:read", "Read inventory")))))
                        // Basic auth for internal tools
                        .addSecuritySchemes("BasicAuth", new SecurityScheme()
                                .type(SecurityScheme.Type.HTTP)
                                .scheme("basic")));
    }
}

// Use specific scheme on individual operations
@Operation(
    summary = "Admin dashboard stats",
    security = @SecurityRequirement(name = "BasicAuth")
)
@GetMapping("/admin/stats")
public AdminStatsDto getStats() { ... }
```

---

## 6. API Versioning in OpenAPI

```java
// Strategy 1: URL path versioning - separate @Tag per version
@RestController
@RequestMapping("/api/v1/products")
@Tag(name = "Products V1", description = "Legacy product API (deprecated)")
public class ProductControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/products")
@Tag(name = "Products V2", description = "Current product API with enhanced features")
public class ProductControllerV2 { ... }

// Strategy 2: Multiple OpenAPI groups
@Configuration
public class MultiVersionOpenApiConfig {

    @Bean
    public GroupedOpenApi v1Api() {
        return GroupedOpenApi.builder()
                .group("v1")
                .displayName("API v1 (Deprecated)")
                .pathsToMatch("/api/v1/**")
                .addOperationCustomizer((operation, handlerMethod) -> {
                    operation.deprecated(true);
                    return operation;
                })
                .build();
    }

    @Bean
    public GroupedOpenApi v2Api() {
        return GroupedOpenApi.builder()
                .group("v2")
                .displayName("API v2 (Current)")
                .pathsToMatch("/api/v2/**")
                .build();
    }

    @Bean
    public GroupedOpenApi internalApi() {
        return GroupedOpenApi.builder()
                .group("internal")
                .displayName("Internal API")
                .pathsToMatch("/internal/**")
                .addOpenApiCustomizer(openApi ->
                        openApi.getInfo().setTitle("Internal API - Not for public use"))
                .build();
    }
}

// Strategy 3: Header-based versioning
@Operation(
    summary = "Get product",
    description = "Supports API-Version header: 1 (default) or 2"
)
@GetMapping(value = "/{id}", headers = "API-Version=2")
public ResponseEntity<ProductDtoV2> getProductV2(@PathVariable Long id) { ... }
```

---

## 7. Generating Client SDKs from OpenAPI Spec

```xml
<!-- pom.xml: Generate TypeScript/Java client at build time -->
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>7.2.0</version>
    <executions>
        <!-- TypeScript client for frontend -->
        <execution>
            <id>typescript-client</id>
            <goals><goal>generate</goal></goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/openapi.yml</inputSpec>
                <generatorName>typescript-axios</generatorName>
                <output>${project.build.directory}/generated-sources/typescript-client</output>
                <configOptions>
                    <supportsES6>true</supportsES6>
                    <npmName>@company/ecommerce-api-client</npmName>
                    <npmVersion>${project.version}</npmVersion>
                </configOptions>
            </configuration>
        </execution>

        <!-- Java client for microservice-to-microservice -->
        <execution>
            <id>java-client</id>
            <goals><goal>generate</goal></goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/openapi.yml</inputSpec>
                <generatorName>java</generatorName>
                <output>${project.build.directory}/generated-sources/java-client</output>
                <configOptions>
                    <library>webclient</library>
                    <apiPackage>com.example.client.api</apiPackage>
                    <modelPackage>com.example.client.model</modelPackage>
                    <dateLibrary>java8</dateLibrary>
                    <useJakartaEe>true</useJakartaEe>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```bash
# Generate spec from running app
curl http://localhost:8080/v3/api-docs > openapi.json

# Or YAML
curl http://localhost:8080/v3/api-docs.yaml > openapi.yaml

# CLI generation
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./frontend/src/api-client \
  --additional-properties=supportsES6=true

# Python client
openapi-generator-cli generate \
  -i openapi.yaml \
  -g python \
  -o ./clients/python \
  --package-name ecommerce_client
```

---

## 8. Contract-First Development

```yaml
# src/main/resources/openapi.yml
# Write spec first, generate server stubs

openapi: "3.0.3"
info:
  title: Order Service API
  version: "1.0.0"

paths:
  /api/v1/orders:
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
          $ref: '#/components/responses/ValidationError'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [customerId, items]
      properties:
        customerId:
          type: string
          pattern: '^CUST-\d{6}$'
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItemRequest'

    OrderItemRequest:
      type: object
      required: [productId, quantity]
      properties:
        productId:
          type: integer
          format: int64
        quantity:
          type: integer
          minimum: 1
          maximum: 1000

    OrderResponse:
      type: object
      properties:
        id:
          type: integer
          format: int64
        status:
          $ref: '#/components/schemas/OrderStatus'
        totalAmount:
          type: number
          format: double

    OrderStatus:
      type: string
      enum: [PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED]

  responses:
    ValidationError:
      description: Request validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ApiError'
```

```xml
<!-- Generate server stubs from spec -->
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>generate-server</id>
            <goals><goal>generate</goal></goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/openapi.yml</inputSpec>
                <generatorName>spring</generatorName>
                <configOptions>
                    <interfaceOnly>true</interfaceOnly>  <!-- generate only interfaces -->
                    <useSpringBoot3>true</useSpringBoot3>
                    <useTags>true</useTags>
                    <apiPackage>com.example.api.generated</apiPackage>
                    <modelPackage>com.example.api.model</modelPackage>
                    <dateLibrary>java8</dateLibrary>
                    <useJakartaEe>true</useJakartaEe>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```java
// Implement the generated interface
@RestController
public class OrderApiImpl implements OrdersApi {  // generated interface

    private final OrderService orderService;

    @Override
    public ResponseEntity<OrderResponse> createOrder(CreateOrderRequest request) {
        OrderResponse result = orderService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(result);
    }
}
```

---

## 9. ReDoc vs Swagger UI

```java
// Serve both UIs
@Configuration
public class ApiDocsConfiguration {

    // Swagger UI: /swagger-ui.html
    // Already configured by springdoc

    // ReDoc: serve at /redoc
    @Bean
    public RouterFunction<ServerResponse> redocRouter() {
        return RouterFunctions.route()
                .GET("/redoc", request -> {
                    String html = """
                            <!DOCTYPE html>
                            <html>
                            <head>
                                <title>API Documentation</title>
                                <meta charset="utf-8"/>
                                <meta name="viewport" content="width=device-width, initial-scale=1">
                                <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700|Roboto:300,400,700" rel="stylesheet">
                                <style>body { margin: 0; padding: 0; }</style>
                            </head>
                            <body>
                                <redoc spec-url='/v3/api-docs'></redoc>
                                <script src="https://cdn.jsdelivr.net/npm/redoc@latest/bundles/redoc.standalone.js"></script>
                            </body>
                            </html>
                            """;
                    return ServerResponse.ok()
                            .contentType(MediaType.TEXT_HTML)
                            .bodyValue(html);
                })
                .build();
    }
}
```

---

## 10. OpenAPI Testing with RestAssured

```java
// Validate responses against OpenAPI spec
// pom.xml: io.rest-assured:json-schema-validator

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductApiContractTest {

    @LocalServerPort
    private int port;

    private static JsonSchema productSchema;
    private static JsonSchema productPageSchema;

    @BeforeAll
    static void loadSchemas() throws IOException {
        // Load schemas extracted from OpenAPI spec
        productSchema = JsonSchemaFactory.getInstance()
                .getSchema(ProductApiContractTest.class
                        .getResourceAsStream("/schemas/product.json"));

        productPageSchema = JsonSchemaFactory.getInstance()
                .getSchema(ProductApiContractTest.class
                        .getResourceAsStream("/schemas/product-page.json"));
    }

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        RestAssured.basePath = "/api/v1";
    }

    @Test
    void getProduct_shouldMatchSchema() {
        String responseBody = given()
                .get("/products/1")
                .then()
                .statusCode(200)
                .extract().asString();

        Set<ValidationMessage> errors = productSchema.validate(JsonLoader.fromString(responseBody));
        assertThat(errors).as("Schema violations: " + errors).isEmpty();
    }

    @Test
    void listProducts_shouldMatchPageSchema() {
        String responseBody = given()
                .get("/products")
                .then()
                .statusCode(200)
                .extract().asString();

        Set<ValidationMessage> errors = productPageSchema.validate(JsonLoader.fromString(responseBody));
        assertThat(errors).as("Schema violations: " + errors).isEmpty();
    }

    @Test
    void createProduct_missingRequiredFields_shouldReturn400() {
        given()
                .contentType(ContentType.JSON)
                .body("{}")
                .post("/products")
                .then()
                .statusCode(400)
                .body("code", equalTo("VALIDATION_FAILED"))
                .body("errors.field", hasItems("name", "price", "stock"));
    }
}
```

---

## 11. Real Example: Fully Documented REST API

```java
@RestController
@RequestMapping("/api/v2/orders")
@Tag(name = "Orders V2", description = "Order management API")
@Validated
@Slf4j
public class OrderControllerV2 {

    private final OrderService orderService;

    @Operation(
        operationId = "createOrder",
        summary = "Create a new order",
        description = """
                Creates a new order in the system. The order goes through the following lifecycle:
                
                1. **PENDING** - Order created, awaiting payment
                2. **CONFIRMED** - Payment processed successfully
                3. **SHIPPED** - Order dispatched from warehouse
                4. **DELIVERED** - Order received by customer
                
                Orders that remain PENDING for more than 24 hours are automatically cancelled.
                """,
        security = @SecurityRequirement(name = "BearerAuth")
    )
    @ApiResponses({
        @ApiResponse(
            responseCode = "201",
            description = "Order created successfully",
            headers = {
                @Header(name = "Location", description = "URL of created order",
                        schema = @Schema(example = "/api/v2/orders/12345")),
                @Header(name = "X-Order-Id", description = "Order ID",
                        schema = @Schema(type = "integer", example = "12345"))
            },
            content = @Content(
                schema = @Schema(implementation = OrderResponse.class),
                examples = @ExampleObject(
                    name = "Successful order",
                    value = """
                            {
                              "id": 12345,
                              "status": "PENDING",
                              "customerId": "CUST-001234",
                              "items": [
                                {
                                  "productId": 42,
                                  "productName": "Wireless Headphones",
                                  "quantity": 2,
                                  "unitPrice": 199.99,
                                  "lineTotal": 399.98
                                }
                              ],
                              "subtotal": 399.98,
                              "tax": 32.00,
                              "shippingCost": 9.99,
                              "totalAmount": 441.97,
                              "estimatedDelivery": "2024-02-01",
                              "createdAt": "2024-01-25T14:30:00Z"
                            }
                            """
                )
            )
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Validation failed",
            content = @Content(
                schema = @Schema(implementation = ApiError.class),
                examples = @ExampleObject(
                    name = "Validation error",
                    value = """
                            {
                              "code": "VALIDATION_FAILED",
                              "message": "Request validation failed",
                              "errors": [
                                {"field": "items[0].quantity", "message": "Must be at least 1"},
                                {"field": "shippingAddress.zipCode", "message": "Invalid ZIP code format"}
                              ]
                            }
                            """
                )
            )
        ),
        @ApiResponse(responseCode = "401", description = "Authentication required"),
        @ApiResponse(responseCode = "402", description = "Payment failed",
                content = @Content(schema = @Schema(implementation = PaymentError.class))),
        @ApiResponse(responseCode = "409", description = "Insufficient stock",
                content = @Content(
                    schema = @Schema(implementation = StockError.class),
                    examples = @ExampleObject(
                        value = """
                                {
                                  "code": "INSUFFICIENT_STOCK",
                                  "productId": 42,
                                  "requested": 5,
                                  "available": 2
                                }
                                """
                    )))
    })
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Order details",
                required = true,
                content = @Content(
                    schema = @Schema(implementation = CreateOrderRequest.class),
                    examples = {
                        @ExampleObject(
                            name = "Simple order",
                            summary = "Single item, standard shipping",
                            value = """
                                    {
                                      "customerId": "CUST-001234",
                                      "items": [
                                        {"productId": 42, "quantity": 1}
                                      ],
                                      "shippingMethod": "STANDARD",
                                      "shippingAddress": {
                                        "street": "123 Main St",
                                        "city": "Springfield",
                                        "state": "IL",
                                        "zipCode": "62701",
                                        "countryCode": "US"
                                      },
                                      "payment": {
                                        "method": "CREDIT_CARD",
                                        "token": "tok_visa"
                                      }
                                    }
                                    """
                        ),
                        @ExampleObject(
                            name = "Multi-item express order",
                            summary = "Multiple items with express shipping and promo code",
                            value = """
                                    {
                                      "customerId": "CUST-001234",
                                      "items": [
                                        {"productId": 42, "quantity": 2},
                                        {"productId": 87, "quantity": 1}
                                      ],
                                      "shippingMethod": "EXPRESS",
                                      "promoCode": "SUMMER20",
                                      "shippingAddress": {
                                        "street": "456 Oak Ave",
                                        "city": "Chicago",
                                        "state": "IL",
                                        "zipCode": "60601",
                                        "countryCode": "US"
                                      },
                                      "payment": {
                                        "method": "PAYPAL",
                                        "token": "EC-PAYPAL-TOKEN"
                                      }
                                    }
                                    """
                        )
                    }
                )
            )
            @Valid @RequestBody CreateOrderRequest request,
            @Parameter(hidden = true) @AuthenticationPrincipal Jwt jwt) {

        String userId = jwt.getClaimAsString("user_id");
        OrderResponse order = orderService.createOrder(userId, request);

        return ResponseEntity.status(HttpStatus.CREATED)
                .location(URI.create("/api/v2/orders/" + order.getId()))
                .header("X-Order-Id", String.valueOf(order.getId()))
                .body(order);
    }

    @Operation(
        summary = "Get order",
        description = "Retrieves order details. Users can only see their own orders unless they have ADMIN role.",
        security = @SecurityRequirement(name = "BearerAuth")
    )
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#id, authentication)")
    public ResponseEntity<OrderResponse> getOrder(
            @Parameter(description = "Order ID", required = true, example = "12345")
            @PathVariable @Positive Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }

    @Operation(
        summary = "List orders",
        description = "Returns paginated list of orders. Admins see all orders; regular users see only their own.",
        security = @SecurityRequirement(name = "BearerAuth")
    )
    @GetMapping
    public ResponseEntity<Page<OrderSummaryResponse>> listOrders(
            @Parameter(description = "Filter by status") @RequestParam(required = false) OrderStatus status,
            @Parameter(description = "Filter from date", example = "2024-01-01")
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @Parameter(description = "Filter to date", example = "2024-12-31")
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to,
            @ParameterObject @PageableDefault(size = 20) Pageable pageable,
            @Parameter(hidden = true) @AuthenticationPrincipal Jwt jwt) {

        return ResponseEntity.ok(orderService.findOrders(jwt, status, from, to, pageable));
    }

    @Operation(
        summary = "Cancel order",
        description = "Cancels an order. Only PENDING or CONFIRMED orders can be cancelled.",
        security = @SecurityRequirement(name = "BearerAuth")
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Order cancelled"),
        @ApiResponse(responseCode = "409", description = "Order cannot be cancelled in current status",
            content = @Content(examples = @ExampleObject(
                value = """
                        {
                          "code": "INVALID_STATUS_TRANSITION",
                          "message": "Cannot cancel order in SHIPPED status",
                          "currentStatus": "SHIPPED",
                          "allowedStatuses": ["PENDING", "CONFIRMED"]
                        }
                        """
            )))
    })
    @PostMapping("/{id}/cancel")
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#id, authentication)")
    public ResponseEntity<OrderResponse> cancelOrder(
            @PathVariable @Positive Long id,
            @Valid @RequestBody CancelOrderRequest request) {
        return ResponseEntity.ok(orderService.cancel(id, request.getReason()));
    }
}

// Error response schemas
@Schema(description = "Standard API error response")
public record ApiError(
    @Schema(description = "Machine-readable error code", example = "VALIDATION_FAILED")
    String code,

    @Schema(description = "Human-readable error message", example = "Request validation failed")
    String message,

    @Schema(description = "Field-level error details (for validation errors)")
    List<FieldErrorDto> errors
) {}

@Schema(description = "Stock availability error")
public record StockError(
    @Schema(example = "INSUFFICIENT_STOCK") String code,
    @Schema(example = "42") Long productId,
    @Schema(example = "Wireless Headphones") String productName,
    @Schema(example = "5") int requested,
    @Schema(example = "2") int available
) {}
```

---

## 12. Hiding Internal Endpoints

```java
@Configuration
public class OpenApiCustomizerConfig {

    @Bean
    public OpenApiCustomizer removeInternalEndpoints() {
        return openApi -> {
            if ("production".equals(activeProfile)) {
                // Remove internal/admin paths from the published spec
                openApi.getPaths().entrySet().removeIf(entry ->
                        entry.getKey().startsWith("/internal") ||
                        entry.getKey().startsWith("/actuator"));
            }
        };
    }

    // Add X-Request-ID to all operations
    @Bean
    public OpenApiCustomizer addRequestIdHeader() {
        return openApi -> openApi.getPaths().values().forEach(pathItem ->
                pathItem.readOperations().forEach(operation ->
                        operation.addParametersItem(new Parameter()
                                .in("header")
                                .name("X-Request-ID")
                                .description("Client-generated request trace ID")
                                .schema(new StringSchema().example("550e8400-e29b-41d4-a716-446655440000"))
                                .required(false))));
    }
}
```

---

## Summary

| Feature | Tool / Annotation |
|---|---|
| Auto-generate spec | `springdoc-openapi-starter-webmvc-ui` |
| Global metadata | `OpenAPI` bean |
| Operation docs | `@Operation`, `@Parameter` |
| Response docs | `@ApiResponse`, `@ApiResponses` |
| Schema docs | `@Schema` |
| Examples | `@ExampleObject` |
| Security | `@SecurityRequirement`, `SecurityScheme` |
| Multi-version | `GroupedOpenApi` |
| Swagger UI | `/swagger-ui.html` |
| ReDoc | Custom router |
| Client SDK gen | `openapi-generator-maven-plugin` |
| Contract testing | `RestAssured + JsonSchema` |

## Next Part Preview

**Part 088** covers HATEOAS and Hypermedia APIs — Spring HATEOAS, HAL, HAL-FORMS, Spring Data REST, and building a fully self-describing API.
