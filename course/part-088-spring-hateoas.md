# Part 088: HATEOAS and Hypermedia APIs

## Overview

HATEOAS (Hypermedia as the Engine of Application State) is the highest level of the Richardson Maturity Model. APIs return links along with data, enabling clients to discover capabilities dynamically without hard-coded URLs. Spring HATEOAS provides the tools to implement this elegantly.

---

## 1. HATEOAS Concept and Richardson Maturity Model

```
Level 0 – One endpoint (RPC over HTTP)
  POST /api  {"action": "getOrder", "id": 1}

Level 1 – Resources (nouns in URLs)
  GET /orders/1
  POST /orders

Level 2 – HTTP verbs + status codes
  GET    /orders/1      → 200 OK
  POST   /orders        → 201 Created
  DELETE /orders/1      → 204 No Content
  GET    /orders/999    → 404 Not Found

Level 3 – HATEOAS (links in responses)
  GET /orders/1
  → {
      "id": 1,
      "status": "PENDING",
      "_links": {
        "self": {"href": "/orders/1"},
        "confirm": {"href": "/orders/1/confirm"},
        "cancel": {"href": "/orders/1/cancel"},
        "customer": {"href": "/customers/42"}
      }
    }
```

---

## 2. Spring HATEOAS Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
// Enable hypermedia support (HAL is the default format)
@SpringBootApplication
@EnableHypermediaSupport(type = HypermediaType.HAL)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 3. EntityModel – Single Resource

```java
// OrderModel wraps OrderDto and adds links
public class OrderModel extends RepresentationModel<OrderModel> {

    private Long id;
    private String status;
    private BigDecimal totalAmount;
    private Instant createdAt;

    // getters/setters
}

@Component
public class OrderModelAssembler implements RepresentationModelAssembler<Order, OrderModel> {

    @Override
    public OrderModel toModel(Order order) {
        OrderModel model = new OrderModel();
        model.setId(order.getId());
        model.setStatus(order.getStatus().name());
        model.setTotalAmount(order.getTotalAmount());
        model.setCreatedAt(order.getCreatedAt());

        // Self link
        model.add(linkTo(methodOn(OrderController.class).getOrder(order.getId())).withSelfRel());

        // Action links based on current state
        if (order.getStatus() == OrderStatus.PENDING) {
            model.add(linkTo(methodOn(OrderController.class)
                    .confirmOrder(order.getId())).withRel("confirm"));
            model.add(linkTo(methodOn(OrderController.class)
                    .cancelOrder(order.getId(), null)).withRel("cancel"));
        }

        if (order.getStatus() == OrderStatus.CONFIRMED) {
            model.add(linkTo(methodOn(OrderController.class)
                    .cancelOrder(order.getId(), null)).withRel("cancel"));
        }

        if (order.getStatus() == OrderStatus.SHIPPED) {
            model.add(linkTo(methodOn(OrderController.class)
                    .trackOrder(order.getId())).withRel("track"));
        }

        // Related resources
        model.add(linkTo(methodOn(CustomerController.class)
                .getCustomer(order.getCustomerId())).withRel("customer"));

        model.add(linkTo(methodOn(OrderController.class)
                .listOrders(null, null, Pageable.unpaged())).withRel("orders"));

        return model;
    }
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;
    private final OrderModelAssembler assembler;

    @GetMapping("/{id}")
    public ResponseEntity<OrderModel> getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id);
        return ResponseEntity.ok(assembler.toModel(order));
    }

    @PostMapping("/{id}/confirm")
    public ResponseEntity<OrderModel> confirmOrder(@PathVariable Long id) {
        Order order = orderService.confirm(id);
        return ResponseEntity.ok(assembler.toModel(order));
    }

    @PostMapping("/{id}/cancel")
    public ResponseEntity<OrderModel> cancelOrder(
            @PathVariable Long id,
            @RequestBody(required = false) CancelOrderRequest request) {
        Order order = orderService.cancel(id,
                request != null ? request.getReason() : "Customer request");
        return ResponseEntity.ok(assembler.toModel(order));
    }

    @GetMapping("/{id}/track")
    public ResponseEntity<TrackingModel> trackOrder(@PathVariable Long id) {
        return ResponseEntity.ok(trackingService.getTracking(id));
    }

    // ... listOrders
}
```

---

## 4. CollectionModel – Resource Collections

```java
@GetMapping
public ResponseEntity<CollectionModel<OrderModel>> listOrders(
        @RequestParam(required = false) String status,
        @RequestParam(required = false) String customerId,
        Pageable pageable) {

    List<Order> orders = orderService.findAll(status, customerId, pageable);

    List<OrderModel> models = orders.stream()
            .map(assembler::toModel)
            .collect(Collectors.toList());

    CollectionModel<OrderModel> collection = CollectionModel.of(models);

    // Self link for the collection
    collection.add(linkTo(methodOn(OrderController.class)
            .listOrders(status, customerId, pageable)).withSelfRel());

    // Navigation links
    collection.add(linkTo(methodOn(OrderController.class)
            .listOrders(null, null, Pageable.unpaged())).withRel("all"));

    return ResponseEntity.ok(collection);
}
```

---

## 5. PagedModel – Paginated Collections

```java
@Component
public class OrderPagedModelAssembler {

    private final OrderModelAssembler orderAssembler;
    private final PagedResourcesAssembler<Order> pagedAssembler;

    @GetMapping
    public ResponseEntity<PagedModel<OrderModel>> listOrdersPaged(
            @RequestParam(required = false) String status,
            @PageableDefault(size = 20) Pageable pageable) {

        Page<Order> page = orderService.findAll(status, pageable);

        // PagedResourcesAssembler adds pagination links automatically
        PagedModel<OrderModel> model = pagedAssembler.toModel(page, assembler);

        // Manual extra links
        model.add(linkTo(methodOn(OrderController.class)
                .createOrder(null)).withRel("create"));

        return ResponseEntity.ok(model);
    }
}

// Response:
// {
//   "_embedded": {
//     "orders": [...]
//   },
//   "_links": {
//     "first": {"href": "/api/orders?page=0&size=20"},
//     "self":  {"href": "/api/orders?page=2&size=20"},
//     "next":  {"href": "/api/orders?page=3&size=20"},
//     "last":  {"href": "/api/orders?page=9&size=20"}
//   },
//   "page": {
//     "size": 20,
//     "totalElements": 200,
//     "totalPages": 10,
//     "number": 2
//   }
// }
```

---

## 6. LinkBuilder – Generating Links

```java
@Component
public class LinkBuilderExamples {

    public void demonstrateLinkBuilding() {
        // Method-on style (type-safe, recommended)
        Link selfLink = linkTo(methodOn(OrderController.class).getOrder(1L)).withSelfRel();
        // href: /api/orders/1

        // Builder with expand (for URI templates)
        Link searchLink = linkTo(OrderController.class)
                .slash("search")
                .withRel("search");
        // href: /api/orders/search

        // Custom rel names
        Link invoiceLink = linkTo(methodOn(OrderController.class).getOrder(1L))
                .slash("invoice")
                .withRel(IanaLinkRelations.DESCRIBEDBY);

        // With query parameters (URI template)
        UriComponentsBuilder builder = linkTo(OrderController.class).toUriComponentsBuilder();
        Link templateLink = Link.of(builder.queryParam("status", "{status}").toUriString())
                .withRel("search");
        // href: /api/orders{?status}

        // Affordance – links WITH form metadata (HAL-FORMS)
        Link createLink = Affordances.of(linkTo(methodOn(OrderController.class)
                    .createOrder(null)).withRel("create"))
                .afford(HttpMethod.POST)
                .withInput(CreateOrderRequest.class)
                .withOutput(OrderModel.class)
                .toLink();
    }
}
```

---

## 7. HAL (Hypertext Application Language)

HAL is the default format used by Spring HATEOAS. The response format:

```json
// GET /api/orders/1
// Accept: application/hal+json

{
  "id": 1,
  "status": "PENDING",
  "totalAmount": 199.98,
  "createdAt": "2024-01-25T14:30:00Z",
  "_links": {
    "self": {
      "href": "http://localhost:8080/api/orders/1"
    },
    "confirm": {
      "href": "http://localhost:8080/api/orders/1/confirm"
    },
    "cancel": {
      "href": "http://localhost:8080/api/orders/1/cancel"
    },
    "customer": {
      "href": "http://localhost:8080/api/customers/CUST-001234"
    },
    "orders": {
      "href": "http://localhost:8080/api/orders"
    }
  }
}
```

```json
// GET /api/orders
// (collection with embedded resources)

{
  "_embedded": {
    "orders": [
      {
        "id": 1,
        "status": "PENDING",
        "_links": {
          "self": {"href": "http://localhost:8080/api/orders/1"}
        }
      },
      {
        "id": 2,
        "status": "CONFIRMED",
        "_links": {
          "self": {"href": "http://localhost:8080/api/orders/2"}
        }
      }
    ]
  },
  "_links": {
    "self": {"href": "http://localhost:8080/api/orders?page=0&size=20"},
    "create": {"href": "http://localhost:8080/api/orders"}
  },
  "page": {
    "size": 20,
    "totalElements": 2,
    "totalPages": 1,
    "number": 0
  }
}
```

---

## 8. HAL-FORMS for Write Operations

HAL-FORMS extends HAL with form metadata for write operations.

```java
// Enable HAL-FORMS
@SpringBootApplication
@EnableHypermediaSupport(type = {HypermediaType.HAL, HypermediaType.HAL_FORMS})
public class Application { ... }

// Return HAL-FORMS response
@GetMapping(value = "/{id}", produces = {
        MediaType.APPLICATION_JSON_VALUE,
        "application/hal+json",
        "application/prs.hal-forms+json"  // HAL-FORMS media type
})
public ResponseEntity<EntityModel<OrderModel>> getOrderWithForms(@PathVariable Long id) {
    Order order = orderService.findById(id);
    OrderModel model = assembler.toModel(order);

    EntityModel<OrderModel> entity = EntityModel.of(model);

    // Add affordances (form metadata)
    if (order.getStatus() == OrderStatus.PENDING) {
        entity.add(Affordances.of(
                linkTo(methodOn(OrderController.class).cancelOrder(id, null)).withRel("cancel"))
                .afford(HttpMethod.POST)
                .withInput(CancelOrderRequest.class)
                .withName("cancel-order")
                .toLink());
    }

    return ResponseEntity.ok(entity);
}

// HAL-FORMS response includes _templates:
// {
//   "id": 1,
//   "status": "PENDING",
//   "_links": {
//     "self": {"href": "/api/orders/1"},
//     "cancel": {"href": "/api/orders/1/cancel"}
//   },
//   "_templates": {
//     "cancel-order": {
//       "method": "POST",
//       "contentType": "application/json",
//       "properties": [
//         {"name": "reason", "type": "text", "required": false}
//       ]
//     }
//   }
// }
```

---

## 9. Spring Data REST

Spring Data REST automatically exposes your JPA repositories as hypermedia-driven REST endpoints.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

```java
// Just annotate your repository - Spring Data REST does the rest
@RepositoryRestResource(
    path = "products",
    collectionResourceRel = "products",
    itemResourceRel = "product"
)
public interface ProductRepository extends PagingAndSortingRepository<Product, Long> {

    // Custom finder exposed as /products/search/findByCategory?category=electronics
    @RestResource(path = "findByCategory", rel = "by-category")
    Page<Product> findByCategory(@Param("category") String category, Pageable pageable);

    // Hide this method from REST
    @Override
    @RestResource(exported = false)
    void deleteById(Long id);
}

// Customize Spring Data REST behavior
@Configuration
public class SpringDataRestConfiguration implements RepositoryRestConfigurer {

    @Override
    public void configureRepositoryRestConfiguration(RepositoryRestConfiguration config,
            CorsRegistry cors) {
        config.setBasePath("/data");
        config.setDefaultPageSize(20);
        config.setMaxPageSize(100);
        config.setDefaultMediaType(MediaType.parseMediaType("application/hal+json"));
        config.setReturnBodyOnCreate(true);
        config.setReturnBodyOnUpdate(true);

        // Expose entity IDs (hidden by default)
        config.exposeIdsFor(Product.class, Order.class, Customer.class);

        // CORS
        cors.addMapping("/data/**")
                .allowedOrigins("https://app.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH");
    }

    @Override
    public void configureValidatingRepositoryEventListener(
            ValidatingRepositoryEventListener listener) {
        listener.addValidator("beforeCreate", productValidator);
        listener.addValidator("beforeSave", productValidator);
    }
}

// Spring Data REST events
@Component
@RepositoryEventHandler(Product.class)
public class ProductEventHandler {

    @HandleBeforeCreate
    public void handleBeforeCreate(Product product) {
        product.setCreatedAt(Instant.now());
        product.setSlug(SlugUtils.toSlug(product.getName()));
    }

    @HandleAfterCreate
    public void handleAfterCreate(Product product) {
        searchIndexService.index(product);
        eventPublisher.publishEvent(new ProductCreatedEvent(product));
    }

    @HandleBeforeDelete
    public void handleBeforeDelete(Product product) {
        // Prevent deletion of products with active orders
        if (orderRepository.existsByProductIdAndStatusIn(product.getId(),
                List.of(OrderStatus.PENDING, OrderStatus.CONFIRMED))) {
            throw new ResourceChangeProhibited("Cannot delete product with active orders");
        }
    }
}
```

---

## 10. Custom Link Relations

```java
// Define custom link relations as constants
public class ECommerceRelations {

    public static final LinkRelation CONFIRM = LinkRelation.of("confirm");
    public static final LinkRelation CANCEL = LinkRelation.of("cancel");
    public static final LinkRelation TRACK = LinkRelation.of("track");
    public static final LinkRelation INVOICE = LinkRelation.of("invoice");
    public static final LinkRelation RETURN = LinkRelation.of("return");
    public static final LinkRelation REORDER = LinkRelation.of("reorder");

    // Registered link relations from IANA
    public static final LinkRelation COLLECTION = IanaLinkRelations.COLLECTION;
    public static final LinkRelation FIRST = IanaLinkRelations.FIRST;
    public static final LinkRelation NEXT = IanaLinkRelations.NEXT;
    public static final LinkRelation PREV = IanaLinkRelations.PREV;
    public static final LinkRelation LAST = IanaLinkRelations.LAST;
}

// Usage
model.add(linkTo(methodOn(OrderController.class).trackOrder(order.getId()))
        .withRel(ECommerceRelations.TRACK));

model.add(linkTo(methodOn(InvoiceController.class).getInvoice(order.getId()))
        .withRel(ECommerceRelations.INVOICE));

if (order.getStatus() == OrderStatus.DELIVERED) {
    model.add(linkTo(methodOn(ReturnController.class).initiateReturn(order.getId(), null))
            .withRel(ECommerceRelations.RETURN));
    model.add(linkTo(methodOn(OrderController.class).createOrder(null))
            .withRel(ECommerceRelations.REORDER)
            .expand(Map.of("orderId", order.getId())));
}
```

---

## 11. Embedded Resources

```java
// Include related resources inline (no extra round trip needed)
public class OrderDetailModel extends RepresentationModel<OrderDetailModel> {

    private Long id;
    private String status;
    private BigDecimal totalAmount;

    // Embedded customer
    @JsonUnwrapped(prefix = "customer_")  // or use _embedded
    private CustomerSummaryModel customer;

    // Embedded items
    private List<OrderItemModel> items;
}

@Component
public class OrderDetailModelAssembler implements RepresentationModelAssembler<Order, OrderDetailModel> {

    private final CustomerService customerService;
    private final CustomerSummaryAssembler customerAssembler;

    @Override
    public OrderDetailModel toModel(Order order) {
        OrderDetailModel model = new OrderDetailModel();
        model.setId(order.getId());
        model.setStatus(order.getStatus().name());
        model.setTotalAmount(order.getTotalAmount());

        // Embed customer (avoids separate GET /customers/{id})
        customerService.findById(order.getCustomerId())
                .map(customerAssembler::toModel)
                .ifPresent(model::setCustomer);

        // Embed items
        model.setItems(order.getItems().stream()
                .map(this::toItemModel)
                .collect(Collectors.toList()));

        model.add(linkTo(methodOn(OrderController.class).getOrder(order.getId())).withSelfRel());
        model.add(linkTo(methodOn(CustomerController.class)
                .getCustomer(order.getCustomerId())).withRel("customer"));

        return model;
    }

    private OrderItemModel toItemModel(OrderItem item) {
        OrderItemModel model = new OrderItemModel();
        model.setProductId(item.getProductId());
        model.setQuantity(item.getQuantity());
        model.setUnitPrice(item.getUnitPrice());

        model.add(linkTo(methodOn(ProductController.class)
                .getProduct(item.getProductId())).withRel("product"));

        return model;
    }
}
```

---

## 12. HATEOAS Client Consumption

```java
// Consuming a HATEOAS API from a Java client

@Service
public class OrderApiClient {

    private final RestTemplate restTemplate;
    private final String baseUrl;

    public OrderApiClient(RestTemplate restTemplate, @Value("${order.service.url}") String baseUrl) {
        this.restTemplate = restTemplate;
        this.baseUrl = baseUrl;
    }

    public EntityModel<OrderModel> getOrder(Long id) {
        // Spring HATEOAS provides Traverson and HypermediaRestTemplate for client-side traversal
        Traverson traverson = new Traverson(URI.create(baseUrl), MediaTypes.HAL_JSON);

        ParameterizedTypeReference<EntityModel<OrderModel>> type =
                new ParameterizedTypeReference<>() {};

        return traverson
                .follow("orders", "search", "by-id")
                .withTemplateParameters(Map.of("id", id))
                .toObject(type);
    }

    // Follow links dynamically
    public OrderModel confirmOrderViaLinks(Long orderId) {
        // 1. Get order resource
        EntityModel<OrderModel> orderResource = getOrder(orderId);
        OrderModel order = orderResource.getContent();

        // 2. Find confirm link
        Link confirmLink = orderResource.getLink("confirm")
                .orElseThrow(() -> new IllegalStateException(
                        "Order cannot be confirmed (no 'confirm' link present). Status: " + order.getStatus()));

        // 3. Follow the link
        ResponseEntity<OrderModel> response = restTemplate.exchange(
                confirmLink.getHref(),
                HttpMethod.POST,
                null,
                OrderModel.class);

        return response.getBody();
    }
}

// WebClient-based HATEOAS client
@Service
public class ReactiveOrderClient {

    private final WebClient webClient;

    public Mono<OrderModel> getOrder(Long id) {
        return webClient.get()
                .uri("/api/orders/{id}", id)
                .accept(MediaTypes.HAL_JSON)
                .retrieve()
                .bodyToMono(new ParameterizedTypeReference<EntityModel<OrderModel>>() {})
                .map(EntityModel::getContent);
    }
}
```

---

## 13. Real Example: Self-Describing API with Full Hypermedia

```java
// ===== Complete HATEOAS API =====

// Entry point – API root that exposes all available resources
@RestController
@RequestMapping("/api")
public class ApiRootController {

    @GetMapping
    public ResponseEntity<RepresentationModel<?>> root() {
        RepresentationModel<?> root = new RepresentationModel<>();

        root.add(linkTo(methodOn(ProductController.class)
                .listProducts(null, null, null, true, Pageable.unpaged())).withRel("products"));

        root.add(linkTo(methodOn(OrderController.class)
                .listOrdersPaged(null, Pageable.unpaged())).withRel("orders"));

        root.add(linkTo(methodOn(CustomerController.class)
                .listCustomers(Pageable.unpaged())).withRel("customers"));

        root.add(linkTo(methodOn(CategoryController.class)
                .listCategories()).withRel("categories"));

        root.add(Link.of("/api/search{?q,type}", "search"));  // URI template

        // Profile links (API documentation)
        root.add(Link.of("/api-docs/orders", IanaLinkRelations.PROFILE));

        return ResponseEntity.ok(root);
    }
}

// ===== Product API =====

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;
    private final ProductModelAssembler assembler;
    private final PagedResourcesAssembler<Product> pagedAssembler;

    @GetMapping
    public ResponseEntity<PagedModel<ProductModel>> listProducts(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "true") boolean inStock,
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {

        Page<Product> products = productService.findAll(
                new ProductFilter(category, minPrice, maxPrice, inStock), pageable);

        PagedModel<ProductModel> model = pagedAssembler.toModel(products, assembler,
                linkTo(methodOn(ProductController.class)
                        .listProducts(category, minPrice, maxPrice, inStock, pageable)).withSelfRel());

        // Add search affordance
        model.add(Link.of(
                linkTo(ProductController.class).toUri() + "{?category,minPrice,maxPrice,inStock,page,size,sort}",
                "search"));

        return ResponseEntity.ok(model);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductModel> getProduct(@PathVariable Long id) {
        Product product = productService.findById(id);
        return ResponseEntity.ok(assembler.toModel(product));
    }
}

// ===== Model Assembler =====

@Component
public class ProductModelAssembler implements RepresentationModelAssembler<Product, ProductModel> {

    @Override
    public ProductModel toModel(Product product) {
        ProductModel model = ProductModel.builder()
                .id(product.getId())
                .name(product.getName())
                .price(product.getPrice())
                .stock(product.getStock())
                .category(product.getCategory())
                .sku(product.getSku())
                .active(product.isActive())
                .build();

        // Always present
        model.add(linkTo(methodOn(ProductController.class).getProduct(product.getId())).withSelfRel());
        model.add(linkTo(methodOn(ProductController.class)
                .listProducts(product.getCategory(), null, null, true, Pageable.unpaged()))
                .withRel("by-category"));
        model.add(linkTo(methodOn(ProductController.class)
                .listProducts(null, null, null, true, Pageable.unpaged()))
                .withRel(IanaLinkRelations.COLLECTION));

        // Conditional links based on state
        if (product.getStock() > 0 && product.isActive()) {
            model.add(Affordances.of(
                    linkTo(methodOn(CartController.class).addToCart(null, null)).withRel("add-to-cart"))
                    .afford(HttpMethod.POST)
                    .withInput(AddToCartRequest.class)
                    .withName("add-to-cart")
                    .toLink());
        }

        if (product.getStock() == 0) {
            model.add(linkTo(methodOn(WishlistController.class)
                    .addToWishlist(null, product.getId())).withRel("notify-when-available"));
        }

        // Admin-only links (add conditionally based on security context)
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            model.add(Affordances.of(
                    linkTo(methodOn(ProductController.class).updateProduct(product.getId(), null)).withRel("edit"))
                    .afford(HttpMethod.PUT)
                    .withInput(UpdateProductRequest.class)
                    .withName("update")
                    .toLink());

            model.add(Affordances.of(
                    linkTo(methodOn(ProductController.class).deleteProduct(product.getId())).withRel("delete"))
                    .afford(HttpMethod.DELETE)
                    .withName("delete")
                    .toLink());
        }

        return model;
    }
}

// ===== ProductModel =====

@Getter
@Setter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ProductModel extends RepresentationModel<ProductModel> {

    @JsonProperty("id")
    private Long id;

    private String name;
    private BigDecimal price;
    private Integer stock;
    private String category;
    private String sku;
    private boolean active;
}

// ===== Order Controller with full HATEOAS =====

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;
    private final OrderModelAssembler assembler;
    private final PagedResourcesAssembler<Order> pagedAssembler;

    @PostMapping
    public ResponseEntity<OrderModel> createOrder(
            @Valid @RequestBody CreateOrderRequest request,
            @AuthenticationPrincipal Jwt jwt) {

        Order order = orderService.create(jwt.getClaimAsString("user_id"), request);
        OrderModel model = assembler.toModel(order);

        return ResponseEntity
                .created(model.getRequiredLink(IanaLinkRelations.SELF).toUri())
                .body(model);
    }

    @GetMapping
    public ResponseEntity<PagedModel<OrderModel>> listOrdersPaged(
            @RequestParam(required = false) String status,
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {

        Page<Order> orders = orderService.findAll(status, pageable);
        return ResponseEntity.ok(pagedAssembler.toModel(orders, assembler));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderModel> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(assembler.toModel(orderService.findById(id)));
    }

    @PostMapping("/{id}/confirm")
    public ResponseEntity<OrderModel> confirmOrder(@PathVariable Long id) {
        return ResponseEntity.ok(assembler.toModel(orderService.confirm(id)));
    }

    @PostMapping("/{id}/cancel")
    public ResponseEntity<OrderModel> cancelOrder(
            @PathVariable Long id,
            @RequestBody(required = false) CancelOrderRequest request) {
        return ResponseEntity.ok(assembler.toModel(
                orderService.cancel(id, request != null ? request.getReason() : "User cancelled")));
    }

    @GetMapping("/{id}/track")
    public ResponseEntity<TrackingModel> trackOrder(@PathVariable Long id) {
        return ResponseEntity.ok(trackingService.getTracking(id));
    }
}

// ===== OrderModelAssembler (complete) =====

@Component
public class OrderModelAssembler implements RepresentationModelAssembler<Order, OrderModel> {

    @Override
    public OrderModel toModel(Order order) {
        OrderModel model = buildModel(order);

        // Self
        model.add(linkTo(methodOn(OrderController.class).getOrder(order.getId())).withSelfRel());

        // Collection
        model.add(linkTo(methodOn(OrderController.class)
                .listOrdersPaged(null, Pageable.unpaged())).withRel(IanaLinkRelations.COLLECTION));

        // State-driven action links
        addActionLinks(model, order);

        // Related resources
        model.add(linkTo(methodOn(CustomerController.class)
                .getCustomer(order.getCustomerId())).withRel("customer"));

        model.add(linkTo(methodOn(InvoiceController.class)
                .getInvoice(order.getId())).withRel("invoice"));

        return model;
    }

    private void addActionLinks(OrderModel model, Order order) {
        switch (order.getStatus()) {
            case PENDING -> {
                model.add(Affordances.of(
                        linkTo(methodOn(OrderController.class).confirmOrder(order.getId())).withRel("confirm"))
                        .afford(HttpMethod.POST)
                        .withName("confirm-order")
                        .toLink());

                model.add(Affordances.of(
                        linkTo(methodOn(OrderController.class).cancelOrder(order.getId(), null)).withRel("cancel"))
                        .afford(HttpMethod.POST)
                        .withInput(CancelOrderRequest.class)
                        .withName("cancel-order")
                        .toLink());
            }
            case CONFIRMED -> {
                model.add(Affordances.of(
                        linkTo(methodOn(OrderController.class).cancelOrder(order.getId(), null)).withRel("cancel"))
                        .afford(HttpMethod.POST)
                        .withInput(CancelOrderRequest.class)
                        .withName("cancel-order")
                        .toLink());
            }
            case SHIPPED -> {
                model.add(linkTo(methodOn(OrderController.class)
                        .trackOrder(order.getId())).withRel("track"));
            }
            case DELIVERED -> {
                model.add(linkTo(methodOn(ReturnController.class)
                        .initiateReturn(order.getId(), null)).withRel("return"));
                model.add(linkTo(methodOn(OrderController.class)
                        .createOrder(null)).withRel("reorder"));
            }
            default -> {}
        }
    }

    private OrderModel buildModel(Order order) {
        return OrderModel.builder()
                .id(order.getId())
                .status(order.getStatus().name())
                .totalAmount(order.getTotalAmount())
                .createdAt(order.getCreatedAt())
                .items(order.getItems().stream()
                        .map(this::toItemModel)
                        .collect(Collectors.toList()))
                .build();
    }

    private OrderItemModel toItemModel(OrderItem item) {
        OrderItemModel model = OrderItemModel.builder()
                .productId(item.getProductId())
                .productName(item.getProductName())
                .quantity(item.getQuantity())
                .unitPrice(item.getUnitPrice())
                .lineTotal(item.getLineTotal())
                .build();

        model.add(linkTo(methodOn(ProductController.class)
                .getProduct(item.getProductId())).withRel("product"));

        return model;
    }
}
```

---

## Summary

| Concept | Spring HATEOAS class |
|---|---|
| Single resource | `RepresentationModel<T>` or `EntityModel<T>` |
| Collection | `CollectionModel<T>` |
| Paged collection | `PagedModel<T>` |
| Link building | `linkTo(methodOn(...))` |
| Link relations | `IanaLinkRelations`, custom `LinkRelation.of(...)` |
| Write affordances | `Affordances.of(...).afford(HttpMethod.POST)` |
| HAL format | default, `application/hal+json` |
| HAL-FORMS | `application/prs.hal-forms+json` |
| Auto-expose JPA | `@RepositoryRestResource` |
| Client traversal | `Traverson` |
| Assembler pattern | `RepresentationModelAssembler<T, M>` |

## Next Part Preview

**Part 089** covers Security Auditing and Compliance — Spring Data JPA Auditing, Hibernate Envers entity history, GDPR compliance, PII masking, and a full audit trail implementation.
