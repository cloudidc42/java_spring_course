# Part 086: Advanced Data Validation

## Overview

Jakarta Bean Validation (formerly JSR-380) integrates deeply with Spring. This part moves beyond `@NotNull` and `@Size` to cover custom constraints, cross-field validation, group sequencing, and programmatic validation — with a complete order creation validation suite as the capstone.

---

## 1. Built-in Annotations

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
// All common built-in constraints
public class ProductRequest {

    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 100, message = "Name must be 3-100 characters")
    private String name;

    @NotNull
    @DecimalMin(value = "0.01", message = "Price must be positive")
    @DecimalMax(value = "999999.99", message = "Price cannot exceed 999,999.99")
    @Digits(integer = 6, fraction = 2, message = "Price format: up to 6 digits and 2 decimals")
    private BigDecimal price;

    @NotNull
    @Min(value = 0, message = "Stock cannot be negative")
    @Max(value = 100000, message = "Stock cannot exceed 100,000")
    private Integer stock;

    @Email(message = "Invalid supplier email")
    private String supplierEmail;

    @URL(message = "Invalid image URL")
    private String imageUrl;

    @Pattern(regexp = "^[A-Z]{2,4}-\\d{4,6}$", message = "SKU format: XX-0000 (e.g. ELEC-12345)")
    @NotBlank
    private String sku;

    @Future(message = "Launch date must be in the future")
    private LocalDate launchDate;

    @PastOrPresent
    private LocalDate manufactureDate;

    @NotEmpty(message = "At least one category is required")
    @Size(max = 5, message = "Maximum 5 categories per product")
    private List<@NotBlank String> categories;

    @Valid  // cascade validation into nested object
    @NotNull
    private DimensionsDto dimensions;
}

public class DimensionsDto {

    @Positive
    private Double width;

    @Positive
    private Double height;

    @Positive
    private Double depth;

    @NotBlank
    @Pattern(regexp = "cm|mm|in", message = "Unit must be cm, mm, or in")
    private String unit;
}
```

---

## 2. Custom Constraint Annotations

```java
// Step 1: Create the annotation
@Documented
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueEmail {

    String message() default "Email address is already registered";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}

// Step 2: Create the validator
@Component  // must be a Spring bean to use @Autowired
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final CustomerRepository customerRepository;

    public UniqueEmailValidator(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isBlank()) {
            return true;  // let @NotBlank handle null/blank
        }
        return !customerRepository.existsByEmailIgnoreCase(email);
    }
}

// Step 3: Use it
public class RegistrationRequest {

    @NotBlank
    @Email
    @UniqueEmail
    private String email;

    @NotBlank
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}
```

### Custom Constraint with Parameters

```java
@Documented
@Constraint(validatedBy = AllowedValuesValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface AllowedValues {

    String[] values();

    String message() default "Value must be one of: {values}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}

public class AllowedValuesValidator implements ConstraintValidator<AllowedValues, String> {

    private Set<String> allowed;

    @Override
    public void initialize(AllowedValues annotation) {
        this.allowed = Set.of(annotation.values());
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true;
        if (!allowed.contains(value)) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                    "Value '" + value + "' is not allowed. Valid values: " + allowed)
                    .addConstraintViolation();
            return false;
        }
        return true;
    }
}

// Usage
public class OrderRequest {

    @AllowedValues(values = {"STANDARD", "EXPRESS", "OVERNIGHT"})
    private String shippingMethod;

    @AllowedValues(values = {"CREDIT_CARD", "DEBIT_CARD", "PAYPAL", "BANK_TRANSFER"})
    private String paymentMethod;
}
```

### Collection Element Validator

```java
@Documented
@Constraint(validatedBy = NoNullElementsValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface NoNullElements {
    String message() default "Collection must not contain null elements";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class NoNullElementsValidator implements ConstraintValidator<NoNullElements, Collection<?>> {

    @Override
    public boolean isValid(Collection<?> collection, ConstraintValidatorContext context) {
        if (collection == null) return true;
        return collection.stream().noneMatch(Objects::isNull);
    }
}
```

---

## 3. Cross-Field Validation

```java
// Approach 1: Class-level constraint
@Documented
@Constraint(validatedBy = DateRangeValidator.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidDateRange {

    String startField();
    String endField();
    String message() default "End date must be after start date";

    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class DateRangeValidator implements ConstraintValidator<ValidDateRange, Object> {

    private String startField;
    private String endField;
    private String message;

    @Override
    public void initialize(ValidDateRange annotation) {
        this.startField = annotation.startField();
        this.endField = annotation.endField();
        this.message = annotation.message();
    }

    @Override
    public boolean isValid(Object object, ConstraintValidatorContext context) {
        try {
            LocalDate start = (LocalDate) BeanUtils.getPropertyDescriptor(
                    object.getClass(), startField).getReadMethod().invoke(object);
            LocalDate end = (LocalDate) BeanUtils.getPropertyDescriptor(
                    object.getClass(), endField).getReadMethod().invoke(object);

            if (start == null || end == null) return true;

            if (!end.isAfter(start)) {
                context.disableDefaultConstraintViolation();
                context.buildConstraintViolationWithTemplate(message)
                        .addPropertyNode(endField)
                        .addConstraintViolation();
                return false;
            }
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}

@ValidDateRange(startField = "startDate", endField = "endDate",
        message = "Promotion end date must be after start date")
public class PromotionRequest {

    @NotNull
    private LocalDate startDate;

    @NotNull
    private LocalDate endDate;

    @NotBlank
    private String name;

    @DecimalMin("0")
    @DecimalMax("100")
    private BigDecimal discountPercent;
}

// Approach 2: Custom class-level annotation with multiple field pairs
@ValidPasswordMatch
public class PasswordChangeRequest {

    @NotBlank
    @Size(min = 8)
    private String newPassword;

    @NotBlank
    private String confirmPassword;
}

@Documented
@Constraint(validatedBy = PasswordMatchValidator.class)
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PasswordMatchValidator implements ConstraintValidator<ValidPasswordMatch, PasswordChangeRequest> {

    @Override
    public boolean isValid(PasswordChangeRequest request, ConstraintValidatorContext context) {
        if (request.getNewPassword() == null || request.getConfirmPassword() == null) {
            return true;
        }
        boolean matches = request.getNewPassword().equals(request.getConfirmPassword());
        if (!matches) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate("Passwords do not match")
                    .addPropertyNode("confirmPassword")
                    .addConstraintViolation();
        }
        return matches;
    }
}
```

---

## 4. Group-Based Validation

```java
// Define groups
public interface CreateGroup {}
public interface UpdateGroup {}
public interface AdminGroup {}

// DTO with group-specific constraints
public class CustomerDto {

    // Only checked during update (ID must be present)
    @NotNull(groups = UpdateGroup.class)
    private Long id;

    // Required when creating, optional when updating (partial update)
    @NotBlank(groups = CreateGroup.class)
    @Size(max = 50)
    private String firstName;

    @NotBlank(groups = CreateGroup.class)
    @Size(max = 50)
    private String lastName;

    @NotBlank(groups = CreateGroup.class)
    @Email
    @UniqueEmail(groups = CreateGroup.class)  // uniqueness only on create
    private String email;

    // Always validated
    @Size(min = 5, max = 20)
    @Pattern(regexp = "\\+?[0-9]{5,20}")
    private String phone;

    // Only admins can set this
    @NotNull(groups = AdminGroup.class)
    private CustomerTier tier;
}

// Controller using groups
@RestController
@RequestMapping("/api/customers")
@Validated
public class CustomerController {

    @PostMapping
    public ResponseEntity<CustomerDto> create(
            @Validated(CreateGroup.class) @RequestBody CustomerDto dto) {
        return ResponseEntity.status(HttpStatus.CREATED).body(customerService.create(dto));
    }

    @PutMapping("/{id}")
    public ResponseEntity<CustomerDto> update(
            @PathVariable Long id,
            @Validated(UpdateGroup.class) @RequestBody CustomerDto dto) {
        return ResponseEntity.ok(customerService.update(id, dto));
    }

    @PatchMapping("/{id}/tier")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<CustomerDto> updateTier(
            @PathVariable Long id,
            @Validated({UpdateGroup.class, AdminGroup.class}) @RequestBody CustomerDto dto) {
        return ResponseEntity.ok(customerService.updateTier(id, dto));
    }
}
```

### Group Sequences

```java
// Validate groups in order; stop on first failure
@GroupSequence({Default.class, CreateGroup.class, UniqueCheckGroup.class})
public interface OrderedCreateValidation {}

// Use with @ScriptAssert for complex expressions
@ScriptAssert(
    lang = "groovy",
    script = "_this.creditLimit == null || _this.creditLimit.compareTo(_this.balance) >= 0",
    message = "Credit limit cannot be less than current balance"
)
public class AccountDto {

    @NotNull
    @DecimalMin("0")
    private BigDecimal creditLimit;

    @NotNull
    @DecimalMin("0")
    private BigDecimal balance;
}
```

---

## 5. Cascaded Validation with @Valid

```java
// Deeply nested validation
public class OrderRequest {

    @NotBlank
    private String customerId;

    @NotEmpty(message = "Order must have at least one item")
    @Size(max = 50, message = "Order cannot exceed 50 items")
    private List<@Valid @NotNull OrderItemRequest> items;

    @Valid
    @NotNull
    private ShippingAddressRequest shippingAddress;

    @Valid
    private BillingAddressRequest billingAddress;  // optional but validated if present

    @Valid
    @NotNull
    private PaymentRequest payment;
}

public class OrderItemRequest {

    @NotNull(message = "Product ID is required")
    @Positive
    private Long productId;

    @NotNull
    @Min(value = 1, message = "Quantity must be at least 1")
    @Max(value = 1000, message = "Quantity cannot exceed 1000")
    private Integer quantity;

    @Valid
    private CustomizationRequest customization;  // optional
}

public class ShippingAddressRequest {

    @NotBlank
    @Size(max = 100)
    private String street;

    @NotBlank
    @Size(max = 50)
    private String city;

    @NotBlank
    @Pattern(regexp = "[A-Z]{2}", message = "State must be 2-letter code")
    private String state;

    @NotBlank
    @Pattern(regexp = "\\d{5}(-\\d{4})?", message = "Invalid ZIP code format")
    private String zipCode;

    @NotBlank
    @Pattern(regexp = "[A-Z]{2}", message = "Country must be ISO 3166-1 alpha-2 code")
    private String countryCode;
}
```

---

## 6. Programmatic Validation with Validator

```java
@Service
public class OrderValidationService {

    private final Validator validator;
    private final ProductRepository productRepository;

    public OrderValidationService(Validator validator, ProductRepository productRepository) {
        this.validator = validator;
        this.productRepository = productRepository;
    }

    public void validate(OrderRequest request) {
        // Bean Validation
        Set<ConstraintViolation<OrderRequest>> violations = validator.validate(request);
        if (!violations.isEmpty()) {
            throw new ValidationException(buildErrorDetails(violations));
        }

        // Business rule validation
        validateBusinessRules(request);
    }

    public void validateForGroup(OrderRequest request, Class<?>... groups) {
        Set<ConstraintViolation<OrderRequest>> violations = validator.validate(request, groups);
        if (!violations.isEmpty()) {
            throw new ValidationException(buildErrorDetails(violations));
        }
    }

    private void validateBusinessRules(OrderRequest request) {
        List<ValidationError> errors = new ArrayList<>();

        // Check products exist and are active
        for (OrderItemRequest item : request.getItems()) {
            Product product = productRepository.findById(item.getProductId()).orElse(null);
            if (product == null) {
                errors.add(new ValidationError(
                        "items[" + request.getItems().indexOf(item) + "].productId",
                        "Product not found: " + item.getProductId()));
            } else if (!product.isActive()) {
                errors.add(new ValidationError(
                        "items[" + request.getItems().indexOf(item) + "].productId",
                        "Product is not available: " + product.getName()));
            } else if (product.getStock() < item.getQuantity()) {
                errors.add(new ValidationError(
                        "items[" + request.getItems().indexOf(item) + "].quantity",
                        String.format("Insufficient stock. Requested: %d, Available: %d",
                                item.getQuantity(), product.getStock())));
            }
        }

        if (!errors.isEmpty()) {
            throw new BusinessValidationException(errors);
        }
    }

    private List<ValidationError> buildErrorDetails(Set<ConstraintViolation<?>> violations) {
        return violations.stream()
                .map(v -> new ValidationError(
                        v.getPropertyPath().toString(),
                        v.getMessage()))
                .sorted(Comparator.comparing(ValidationError::field))
                .collect(Collectors.toList());
    }
}
```

---

## 7. Method-Level Validation with @Validated

```java
// Enable method validation on class
@Service
@Validated
public class ProductService {

    // Validate method parameters
    public ProductDto updatePrice(
            @Positive(message = "Product ID must be positive") Long productId,
            @NotNull @DecimalMin("0.01") BigDecimal newPrice) {

        Product product = productRepository.findById(productId)
                .orElseThrow(() -> new ProductNotFoundException(productId));
        product.setPrice(newPrice);
        return mapper.toDto(productRepository.save(product));
    }

    // Validate return value
    @Valid
    @NotNull
    public ProductDto findById(Long id) {
        return productRepository.findById(id)
                .map(mapper::toDto)
                .orElseThrow(() -> new ProductNotFoundException(id));
    }

    // Validate collection elements in parameters
    public List<ProductDto> findByIds(
            @NotEmpty @Size(max = 100) List<@NotNull @Positive Long> ids) {
        return productRepository.findAllById(ids).stream()
                .map(mapper::toDto)
                .collect(Collectors.toList());
    }
}

// Handle ConstraintViolationException from method validation
@RestControllerAdvice
public class MethodValidationExceptionHandler {

    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraintViolation(ConstraintViolationException ex) {
        List<FieldError> errors = ex.getConstraintViolations().stream()
                .map(violation -> new FieldError(
                        extractParameterName(violation.getPropertyPath()),
                        violation.getMessage(),
                        String.valueOf(violation.getInvalidValue())))
                .collect(Collectors.toList());

        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_FAILED", errors));
    }

    private String extractParameterName(Path propertyPath) {
        String fullPath = propertyPath.toString();
        // Path format: "methodName.paramName" or "methodName.paramName[0].field"
        int dotIndex = fullPath.indexOf('.');
        return dotIndex >= 0 ? fullPath.substring(dotIndex + 1) : fullPath;
    }
}
```

---

## 8. Request Validation – Controller Level

```java
@RestController
@RequestMapping("/api/orders")
@Validated
public class OrderController {

    // Request body validation
    @PostMapping
    public ResponseEntity<OrderDto> createOrder(
            @Valid @RequestBody OrderRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(orderService.create(request));
    }

    // Path variable validation
    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(
            @PathVariable @Positive(message = "Order ID must be positive") Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }

    // Query parameter validation
    @GetMapping
    public ResponseEntity<Page<OrderDto>> listOrders(
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
            @RequestParam(required = false) @AllowedValues(values = {"PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"}) String status,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) @PastOrPresent LocalDate from,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {

        return ResponseEntity.ok(orderService.findAll(page, size, status, from, to));
    }

    // Handle @Valid failure
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationError(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(error -> new FieldError(
                        error.getField(),
                        error.getDefaultMessage(),
                        String.valueOf(error.getRejectedValue())))
                .collect(Collectors.toList());

        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_FAILED", "Request validation failed", errors));
    }
}
```

---

## 9. Custom Validation Error Messages

```properties
# src/main/resources/ValidationMessages.properties

# Custom messages for built-in constraints
jakarta.validation.constraints.NotNull.message=This field is required
jakarta.validation.constraints.NotBlank.message=This field must not be blank
jakarta.validation.constraints.Size.message=Must be between {min} and {max} characters
jakarta.validation.constraints.Email.message=Please enter a valid email address
jakarta.validation.constraints.Min.message=Must be at least {value}
jakarta.validation.constraints.Max.message=Must be at most {value}

# Custom constraint messages
UniqueEmail.message=This email address is already registered
AllowedValues.message=Invalid value ''{validatedValue}''. Allowed values: {values}
ValidDateRange.message={endField} must be after {startField}
```

```java
// Interpolation of constraint parameters in messages
@Constraint(validatedBy = PhoneValidator.class)
public @interface ValidPhone {
    String countryCode() default "US";
    String message() default "Invalid phone number for country {countryCode}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Using MessageInterpolator for dynamic messages
@Component
public class MessageSourceConstraintValidatorFactory implements ConstraintValidatorFactory {

    private final ApplicationContext applicationContext;

    @Override
    public <T extends ConstraintValidator<?, ?>> T getInstance(Class<T> key) {
        try {
            return applicationContext.getBean(key);
        } catch (BeansException e) {
            try {
                return key.getDeclaredConstructor().newInstance();
            } catch (Exception ex) {
                throw new RuntimeException("Could not instantiate validator: " + key, ex);
            }
        }
    }

    @Override
    public void releaseInstance(ConstraintValidator<?, ?> instance) {
        // Spring manages lifecycle
    }
}
```

---

## 10. Unit Testing Validators

```java
class UniqueEmailValidatorTest {

    private UniqueEmailValidator validator;
    private CustomerRepository mockRepository;
    private ConstraintValidatorContext mockContext;

    @BeforeEach
    void setUp() {
        mockRepository = mock(CustomerRepository.class);
        mockContext = mock(ConstraintValidatorContext.class);
        validator = new UniqueEmailValidator(mockRepository);
    }

    @Test
    void shouldReturnTrueForUniqueEmail() {
        when(mockRepository.existsByEmailIgnoreCase("new@example.com")).thenReturn(false);

        assertThat(validator.isValid("new@example.com", mockContext)).isTrue();
    }

    @Test
    void shouldReturnFalseForDuplicateEmail() {
        when(mockRepository.existsByEmailIgnoreCase("existing@example.com")).thenReturn(true);

        assertThat(validator.isValid("existing@example.com", mockContext)).isFalse();
    }

    @Test
    void shouldReturnTrueForNull() {
        assertThat(validator.isValid(null, mockContext)).isTrue();
        verifyNoInteractions(mockRepository);
    }
}

// Integration test with Spring context
@SpringBootTest
class OrderRequestValidationTest {

    @Autowired
    private Validator validator;

    @Test
    void shouldPassValidOrderRequest() {
        OrderRequest request = OrderRequest.builder()
                .customerId("CUST-001")
                .items(List.of(new OrderItemRequest(1L, 2)))
                .shippingAddress(validAddress())
                .payment(new PaymentRequest("CREDIT_CARD", "tok_visa"))
                .build();

        Set<ConstraintViolation<OrderRequest>> violations = validator.validate(request);
        assertThat(violations).isEmpty();
    }

    @Test
    void shouldFailWhenItemsIsEmpty() {
        OrderRequest request = OrderRequest.builder()
                .customerId("CUST-001")
                .items(List.of())  // empty
                .build();

        Set<ConstraintViolation<OrderRequest>> violations = validator.validate(request);

        assertThat(violations)
                .extracting(v -> v.getPropertyPath().toString())
                .contains("items");

        assertThat(violations)
                .filteredOn(v -> v.getPropertyPath().toString().equals("items"))
                .extracting(ConstraintViolation::getMessage)
                .containsExactly("Order must have at least one item");
    }

    @Test
    void shouldFailCascadedValidationInItem() {
        OrderRequest request = OrderRequest.builder()
                .customerId("CUST-001")
                .items(List.of(new OrderItemRequest(null, 0)))  // invalid item
                .shippingAddress(validAddress())
                .build();

        Set<ConstraintViolation<OrderRequest>> violations = validator.validate(request);

        assertThat(violations).extracting(v -> v.getPropertyPath().toString())
                .containsExactlyInAnyOrder(
                        "items[0].productId",
                        "items[0].quantity");
    }

    private ShippingAddressRequest validAddress() {
        return ShippingAddressRequest.builder()
                .street("123 Main St")
                .city("Springfield")
                .state("IL")
                .zipCode("62701")
                .countryCode("US")
                .build();
    }
}
```

---

## 11. Real Example: Complex Order Validation

```java
// ===== Full validation stack for order creation =====

// Custom constraint for stock availability
@Documented
@Constraint(validatedBy = {})  // validated programmatically
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidOrderItems {
    String message() default "Order items are invalid";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Multi-level custom validator
@Component
public class OrderItemsValidator implements ConstraintValidator<ValidOrderItems, List<OrderItemRequest>> {

    private final ProductRepository productRepository;

    public OrderItemsValidator(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Override
    public boolean isValid(List<OrderItemRequest> items, ConstraintValidatorContext context) {
        if (items == null || items.isEmpty()) return true;  // handled by @NotEmpty

        boolean valid = true;
        context.disableDefaultConstraintViolation();

        // Check for duplicate products
        Map<Long, Long> productCount = items.stream()
                .filter(i -> i.getProductId() != null)
                .collect(Collectors.groupingBy(OrderItemRequest::getProductId, Collectors.counting()));

        for (Map.Entry<Long, Long> entry : productCount.entrySet()) {
            if (entry.getValue() > 1) {
                context.buildConstraintViolationWithTemplate(
                                "Duplicate product ID: " + entry.getKey() + ". Combine quantities instead.")
                        .addPropertyNode("items")
                        .addConstraintViolation();
                valid = false;
            }
        }

        // Batch load products for efficiency
        List<Long> productIds = items.stream()
                .map(OrderItemRequest::getProductId)
                .filter(Objects::nonNull)
                .collect(Collectors.toList());

        Map<Long, Product> products = productRepository.findAllById(productIds).stream()
                .collect(Collectors.toMap(Product::getId, p -> p));

        for (int i = 0; i < items.size(); i++) {
            OrderItemRequest item = items.get(i);
            if (item.getProductId() == null) continue;

            Product product = products.get(item.getProductId());
            if (product == null) {
                context.buildConstraintViolationWithTemplate(
                                "Product not found: " + item.getProductId())
                        .addPropertyNode("items[" + i + "].productId")
                        .addConstraintViolation();
                valid = false;
            } else if (!product.isActive()) {
                context.buildConstraintViolationWithTemplate(
                                "Product '" + product.getName() + "' is no longer available")
                        .addPropertyNode("items[" + i + "].productId")
                        .addConstraintViolation();
                valid = false;
            } else if (item.getQuantity() != null && product.getStock() < item.getQuantity()) {
                context.buildConstraintViolationWithTemplate(
                                String.format("Insufficient stock for '%s'. Available: %d, Requested: %d",
                                        product.getName(), product.getStock(), item.getQuantity()))
                        .addPropertyNode("items[" + i + "].quantity")
                        .addConstraintViolation();
                valid = false;
            }
        }

        return valid;
    }
}

// Complete order request with all validation layers
@ValidDateRange(startField = "scheduledDeliveryFrom", endField = "scheduledDeliveryTo",
        message = "Delivery window end must be after start",
        groups = Default.class)
public class CompleteOrderRequest {

    // Basic validation
    @NotBlank(message = "Customer ID is required")
    @Pattern(regexp = "CUST-\\d{6}", message = "Invalid customer ID format (CUST-000000)")
    private String customerId;

    // Cascaded validation
    @NotEmpty(message = "At least one item is required")
    @Size(max = 50, message = "Cannot order more than 50 different products at once")
    @ValidOrderItems  // database validation
    private List<@Valid @NotNull OrderItemRequest> items;

    @Valid
    @NotNull(message = "Shipping address is required")
    private ShippingAddressRequest shippingAddress;

    @Valid
    private BillingAddressRequest billingAddress;  // optional

    @Valid
    @NotNull(message = "Payment information is required")
    private PaymentRequest payment;

    // Cross-field validation
    @Future(message = "Delivery date must be in the future")
    private LocalDate scheduledDeliveryFrom;

    private LocalDate scheduledDeliveryTo;

    @AllowedValues(values = {"STANDARD", "EXPRESS", "OVERNIGHT", "PICKUP"})
    @NotBlank(message = "Shipping method is required")
    private String shippingMethod;

    // Conditional validation handled by service layer
    private String promoCode;

    @Size(max = 500, message = "Notes cannot exceed 500 characters")
    private String customerNotes;
}

// Service layer adds business rules
@Service
@Validated
public class OrderCreationService {

    private final Validator validator;
    private final CustomerRepository customerRepository;
    private final PromotionService promotionService;

    @Transactional
    public OrderDto createOrder(@Valid CompleteOrderRequest request) {
        // Bean validation already done by @Valid
        // Now add business rules

        // 1. Verify customer exists and is in good standing
        Customer customer = customerRepository.findById(request.getCustomerId())
                .orElseThrow(() -> new ValidationException(
                        List.of(new ValidationError("customerId", "Customer account not found"))));

        if (customer.isBlocked()) {
            throw new ValidationException(
                    List.of(new ValidationError("customerId", "Customer account is blocked")));
        }

        // 2. Validate promo code if provided
        Optional<Promotion> promotion = Optional.empty();
        if (request.getPromoCode() != null) {
            promotion = promotionService.validateAndGet(request.getPromoCode());
            if (promotion.isEmpty()) {
                throw new ValidationException(
                        List.of(new ValidationError("promoCode", "Invalid or expired promo code")));
            }
        }

        // 3. Create order
        return buildAndSaveOrder(customer, request, promotion.orElse(null));
    }

    private OrderDto buildAndSaveOrder(Customer customer, CompleteOrderRequest request,
                                        Promotion promotion) {
        // ... build order from request
        return new OrderDto();
    }
}

// Global exception handler for validation errors
@RestControllerAdvice
public class GlobalValidationHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleMethodArgumentNotValid(MethodArgumentNotValidException ex) {
        List<FieldErrorDto> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> new FieldErrorDto(e.getField(), e.getDefaultMessage(),
                        formatRejectedValue(e.getRejectedValue())))
                .collect(Collectors.toList());

        List<FieldErrorDto> globalErrors = ex.getBindingResult().getGlobalErrors().stream()
                .map(e -> new FieldErrorDto(e.getObjectName(), e.getDefaultMessage(), null))
                .collect(Collectors.toList());

        List<FieldErrorDto> allErrors = Stream.concat(fieldErrors.stream(), globalErrors.stream())
                .collect(Collectors.toList());

        return ResponseEntity.badRequest().body(
                new ApiError("VALIDATION_FAILED", "Request validation failed", allErrors));
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ApiError> handleBusinessValidation(ValidationException ex) {
        List<FieldErrorDto> errors = ex.getErrors().stream()
                .map(e -> new FieldErrorDto(e.field(), e.message(), null))
                .collect(Collectors.toList());

        return ResponseEntity.unprocessableEntity().body(
                new ApiError("BUSINESS_RULE_VIOLATION", "Business rule validation failed", errors));
    }

    private String formatRejectedValue(Object value) {
        if (value == null) return null;
        String str = value.toString();
        return str.length() > 100 ? str.substring(0, 100) + "..." : str;
    }
}

// DTOs for error responses
public record ApiError(String code, String message, List<FieldErrorDto> errors) {}
public record FieldErrorDto(String field, String message, String rejectedValue) {}
```

---

## Summary

| Feature | Annotation/Class | Notes |
|---|---|---|
| Built-in constraints | `@NotNull`, `@Size`, `@Email`, etc. | Jakarta Validation 3.x |
| Custom constraint | `@interface + ConstraintValidator` | Spring bean injection supported |
| Cross-field | Class-level `@Constraint` | Access all fields in validator |
| Groups | `groups = CreateGroup.class` | Conditional validation per scenario |
| Group sequence | `@GroupSequence` | Validate in order, stop on failure |
| Cascade | `@Valid` on field | Trigger nested object validation |
| Method-level | `@Validated` on class | Validate parameters and return values |
| Programmatic | `Validator.validate()` | In-service validation + business rules |
| Error format | `MethodArgumentNotValidException` | `@ExceptionHandler` for HTTP mapping |

## Next Part Preview

**Part 087** covers API Documentation with OpenAPI — springdoc setup, full annotation coverage, security schemes, and generating client SDKs from your spec.
