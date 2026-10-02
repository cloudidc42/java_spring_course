# Part 023: Spring MVC - Web Application Framework

## เนื้อหาในส่วนนี้
- Spring MVC Architecture (DispatcherServlet)
- @Controller vs @RestController
- Request Mapping
- Path Variables, Query Parameters, Request Body
- Response Handling
- Form Validation
- Exception Handling
- File Upload/Download
- Content Negotiation
- Complete REST API Example

---

## 1. Spring MVC Architecture

```
HTTP Request
    ↓
DispatcherServlet (Front Controller)
    ↓
HandlerMapping    → finds controller method
    ↓
HandlerAdapter    → calls controller method
    ↓
@Controller/@RestController method
    ↓
ViewResolver (for HTML) / MessageConverter (for JSON/XML)
    ↓
HTTP Response
```

### Maven Dependencies (Spring Boot)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <!-- Spring Boot Web (includes Spring MVC + Tomcat) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Bean Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    
    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 2. First Spring Boot Web Application

```java
// === Main Application ===
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class WebApplication {
    public static void main(String[] args) {
        SpringApplication.run(WebApplication.class, args);
    }
}

// application.properties:
// server.port=8080
// spring.application.name=my-web-app
// logging.level.org.springframework.web=DEBUG

// === Simple Controller ===
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

@Controller
public class HomeController {
    
    @GetMapping("/")
    @ResponseBody  // Return string directly (not view name)
    public String home() {
        return "Welcome to Spring MVC!";
    }
    
    @GetMapping("/hello")
    @ResponseBody
    public String hello(@RequestParam(defaultValue = "World") String name) {
        return "Hello, " + name + "!";
    }
}
```

---

## 3. @RestController and Request Mapping

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;
import java.util.*;

@RestController  // @Controller + @ResponseBody on all methods
@RequestMapping("/api/v1/products")  // Base URL prefix
public class ProductController {
    
    // GET /api/v1/products
    @GetMapping
    public List<Product> getAllProducts() {
        return List.of(
            new Product("1", "Laptop", 999.99),
            new Product("2", "Mouse", 29.99)
        );
    }
    
    // GET /api/v1/products/{id}
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable String id) {
        return new Product(id, "Product " + id, 99.99);
    }
    
    // GET /api/v1/products/{id}/variants/{variantId}
    @GetMapping("/{id}/variants/{variantId}")
    public String getVariant(@PathVariable String id, @PathVariable String variantId) {
        return "Product: " + id + ", Variant: " + variantId;
    }
    
    // GET /api/v1/products?category=electronics&page=0&size=10&sort=name
    @GetMapping("/search")
    public List<Product> searchProducts(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "name") String sort) {
        return List.of(new Product("1", "Laptop", 999.99));
    }
    
    // POST /api/v1/products
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // Returns 201
    public Product createProduct(@RequestBody CreateProductRequest request) {
        String id = UUID.randomUUID().toString();
        return new Product(id, request.name(), request.price());
    }
    
    // PUT /api/v1/products/{id}
    @PutMapping("/{id}")
    public Product updateProduct(@PathVariable String id, 
                                  @RequestBody UpdateProductRequest request) {
        return new Product(id, request.name(), request.price());
    }
    
    // PATCH /api/v1/products/{id}
    @PatchMapping("/{id}/price")
    public Product updatePrice(@PathVariable String id,
                                @RequestBody Map<String, Double> body) {
        double newPrice = body.get("price");
        return new Product(id, "Product " + id, newPrice);
    }
    
    // DELETE /api/v1/products/{id}
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)  // Returns 204
    public void deleteProduct(@PathVariable String id) {
        // Delete product
    }
    
    // Multiple paths
    @GetMapping({"/featured", "/bestsellers"})
    public List<Product> getFeatured() {
        return List.of(new Product("1", "Laptop", 999.99));
    }
    
    // With headers
    @GetMapping(value = "/", headers = "X-API-Version=2")
    public Map<String, String> homeV2() {
        return Map.of("version", "2", "status", "ok");
    }
    
    // Consumes/Produces
    @PostMapping(value = "/xml", 
                 consumes = MediaType.APPLICATION_XML_VALUE,
                 produces = MediaType.APPLICATION_XML_VALUE)
    public Product createFromXml(@RequestBody Product product) {
        return product;
    }
}

record Product(String id, String name, double price) {}
record CreateProductRequest(String name, double price, String category) {}
record UpdateProductRequest(String name, double price) {}
```

---

## 4. ResponseEntity - Full HTTP Control

```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import java.net.URI;
import java.util.*;

@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final Map<Long, UserDto> users = new HashMap<>();
    private long nextId = 1;
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        UserDto user = users.get(id);
        
        if (user == null) {
            return ResponseEntity.notFound().build();  // 404
        }
        
        return ResponseEntity.ok(user);  // 200 + body
    }
    
    @PostMapping
    public ResponseEntity<UserDto> createUser(@RequestBody CreateUserRequest request) {
        long id = nextId++;
        UserDto user = new UserDto(id, request.name(), request.email());
        users.put(id, user);
        
        URI location = URI.create("/api/users/" + id);
        
        return ResponseEntity
            .created(location)          // 201
            .header("X-User-Id", String.valueOf(id))
            .body(user);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<UserDto> updateUser(@PathVariable Long id,
                                               @RequestBody CreateUserRequest request) {
        if (!users.containsKey(id)) {
            return ResponseEntity.notFound().build();  // 404
        }
        
        UserDto updated = new UserDto(id, request.name(), request.email());
        users.put(id, updated);
        
        return ResponseEntity.ok(updated);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        if (!users.containsKey(id)) {
            return ResponseEntity.notFound().build();
        }
        users.remove(id);
        return ResponseEntity.noContent().build();  // 204
    }
    
    @GetMapping
    public ResponseEntity<List<UserDto>> getAllUsers(
            @RequestHeader(value = "Accept-Language", defaultValue = "en") String lang) {
        
        return ResponseEntity.ok()
            .header("Content-Language", lang)
            .header("X-Total-Count", String.valueOf(users.size()))
            .cacheControl(CacheControl.maxAge(30, java.util.concurrent.TimeUnit.SECONDS))
            .body(new ArrayList<>(users.values()));
    }
    
    // Custom status codes
    @GetMapping("/status-examples")
    public ResponseEntity<String> statusExamples(@RequestParam int code) {
        return switch (code) {
            case 200 -> ResponseEntity.ok("OK");
            case 201 -> ResponseEntity.created(URI.create("/example")).build();
            case 204 -> ResponseEntity.noContent().build();
            case 400 -> ResponseEntity.badRequest().body("Bad Request");
            case 401 -> ResponseEntity.status(HttpStatus.UNAUTHORIZED).body("Unauthorized");
            case 403 -> ResponseEntity.status(HttpStatus.FORBIDDEN).body("Forbidden");
            case 404 -> ResponseEntity.notFound().build();
            case 409 -> ResponseEntity.status(HttpStatus.CONFLICT).body("Conflict");
            case 422 -> ResponseEntity.unprocessableEntity().body("Unprocessable");
            case 500 -> ResponseEntity.internalServerError().body("Server Error");
            default -> ResponseEntity.status(code).body("Custom: " + code);
        };
    }
}

record UserDto(Long id, String name, String email) {}
record CreateUserRequest(String name, String email) {}
```

---

## 5. Bean Validation

```java
import jakarta.validation.constraints.*;
import jakarta.validation.*;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;
import java.time.LocalDate;
import java.util.List;

// === Request DTOs with validation ===
public record RegisterUserRequest(
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    String name,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = ".*[A-Z].*", message = "Password must contain uppercase letter")
    @Pattern(regexp = ".*[0-9].*", message = "Password must contain digit")
    String password,
    
    @NotNull(message = "Birth date is required")
    @Past(message = "Birth date must be in the past")
    LocalDate birthDate,
    
    @Min(value = 18, message = "Must be at least 18 years old")
    @Max(value = 120, message = "Age seems unrealistic")
    int age,
    
    @NotEmpty(message = "At least one role is required")
    List<@NotBlank String> roles
) {}

// === Custom Validator ===
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target({ElementType.FIELD})
@Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
@Documented
@interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    // Normally would check database
    private static final java.util.Set<String> EXISTING_EMAILS = 
        java.util.Set.of("existing@example.com");
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return email == null || !EXISTING_EMAILS.contains(email);
    }
}

// === Validation Groups ===
interface CreateGroup {}
interface UpdateGroup {}

record ProductRequest(
    @NotBlank(groups = CreateGroup.class) String name,
    @Positive double price,
    @NotNull(groups = CreateGroup.class) String category
) {}

// === Controller with Validation ===
@RestController
@RequestMapping("/api/users")
@Validated
public class ValidatedUserController {
    
    // @Valid triggers validation, @RequestBody binds JSON
    @PostMapping
    public ResponseEntity<String> register(@Valid @RequestBody RegisterUserRequest request) {
        return ResponseEntity.ok("Registered: " + request.name());
    }
    
    // Validated groups
    @PostMapping("/product")
    public ResponseEntity<String> createProduct(
            @Validated(CreateGroup.class) @RequestBody ProductRequest request) {
        return ResponseEntity.ok("Created: " + request.name());
    }
    
    @PutMapping("/product/{id}")
    public ResponseEntity<String> updateProduct(
            @PathVariable String id,
            @Validated(UpdateGroup.class) @RequestBody ProductRequest request) {
        return ResponseEntity.ok("Updated: " + id);
    }
    
    // Path variable validation
    @GetMapping("/{id}")
    public ResponseEntity<String> getUser(
            @PathVariable @Min(1) @Max(Long.MAX_VALUE) long id) {
        return ResponseEntity.ok("User: " + id);
    }
    
    // Query param validation
    @GetMapping
    public ResponseEntity<String> list(
            @RequestParam @Min(0) int page,
            @RequestParam @Min(1) @Max(100) int size) {
        return ResponseEntity.ok("page=" + page + ", size=" + size);
    }
}
```

---

## 6. Exception Handling

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import java.time.Instant;
import java.util.*;

// === Custom Exceptions ===
class ResourceNotFoundException extends RuntimeException {
    private final String resourceType;
    private final Object resourceId;
    
    ResourceNotFoundException(String resourceType, Object resourceId) {
        super(resourceType + " not found with id: " + resourceId);
        this.resourceType = resourceType;
        this.resourceId = resourceId;
    }
    
    String getResourceType() { return resourceType; }
    Object getResourceId() { return resourceId; }
}

class DuplicateResourceException extends RuntimeException {
    DuplicateResourceException(String message) { super(message); }
}

class BusinessRuleViolationException extends RuntimeException {
    private final String errorCode;
    
    BusinessRuleViolationException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    String getErrorCode() { return errorCode; }
}

// === Error Response DTOs ===
record ErrorResponse(
    Instant timestamp,
    int status,
    String error,
    String message,
    String path,
    Map<String, List<String>> validationErrors
) {
    static ErrorResponse of(int status, String error, String message, String path) {
        return new ErrorResponse(Instant.now(), status, error, message, path, null);
    }
    
    static ErrorResponse withValidation(int status, String error, String message, 
                                         String path, Map<String, List<String>> errors) {
        return new ErrorResponse(Instant.now(), status, error, message, path, errors);
    }
}

// === Global Exception Handler ===
@RestControllerAdvice  // @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {
    
    // Handle specific exception
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex,
                                         jakarta.servlet.http.HttpServletRequest request) {
        return ErrorResponse.of(404, "Not Found", ex.getMessage(), request.getRequestURI());
    }
    
    // Handle duplicate resource
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex,
                                                          jakarta.servlet.http.HttpServletRequest request) {
        ErrorResponse response = ErrorResponse.of(409, "Conflict", ex.getMessage(), 
                                                    request.getRequestURI());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(response);
    }
    
    // Handle business rule violations
    @ExceptionHandler(BusinessRuleViolationException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    public Map<String, String> handleBusinessRule(BusinessRuleViolationException ex) {
        return Map.of(
            "errorCode", ex.getErrorCode(),
            "message", ex.getMessage()
        );
    }
    
    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex,
                                           jakarta.servlet.http.HttpServletRequest request) {
        Map<String, List<String>> errors = new HashMap<>();
        
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = error instanceof FieldError fe 
                ? fe.getField() : error.getObjectName();
            String message = error.getDefaultMessage();
            errors.computeIfAbsent(fieldName, k -> new ArrayList<>()).add(message);
        });
        
        return ErrorResponse.withValidation(400, "Bad Request", 
            "Validation failed", request.getRequestURI(), errors);
    }
    
    // Handle constraint violations
    @ExceptionHandler(jakarta.validation.ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleConstraintViolation(
            jakarta.validation.ConstraintViolationException ex,
            jakarta.servlet.http.HttpServletRequest request) {
        
        Map<String, List<String>> errors = new HashMap<>();
        ex.getConstraintViolations().forEach(violation -> {
            String field = violation.getPropertyPath().toString();
            errors.computeIfAbsent(field, k -> new ArrayList<>())
                  .add(violation.getMessage());
        });
        
        return ErrorResponse.withValidation(400, "Bad Request",
            "Constraint violation", request.getRequestURI(), errors);
    }
    
    // Handle illegal arguments
    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleIllegalArgument(IllegalArgumentException ex,
                                                 jakarta.servlet.http.HttpServletRequest request) {
        return ErrorResponse.of(400, "Bad Request", ex.getMessage(), request.getRequestURI());
    }
    
    // Catch-all for unexpected exceptions
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex,
                                        jakarta.servlet.http.HttpServletRequest request) {
        // Log the exception (don't expose internal details)
        System.err.println("Unexpected error: " + ex.getMessage());
        return ErrorResponse.of(500, "Internal Server Error", 
            "An unexpected error occurred", request.getRequestURI());
    }
}

// === Controller throwing exceptions ===
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final Map<String, Object> orders = new HashMap<>();
    
    @GetMapping("/{id}")
    public Object getOrder(@PathVariable String id) {
        Object order = orders.get(id);
        if (order == null) {
            throw new ResourceNotFoundException("Order", id);
        }
        return order;
    }
    
    @PostMapping
    public ResponseEntity<String> createOrder(@Valid @RequestBody Map<String, Object> body) {
        String customerId = (String) body.get("customerId");
        if (customerId == null) {
            throw new BusinessRuleViolationException("MISSING_CUSTOMER", 
                "Order must have a customer");
        }
        String id = UUID.randomUUID().toString();
        orders.put(id, body);
        return ResponseEntity.created(URI.create("/api/orders/" + id)).body(id);
    }
}
```

---

## 7. Request/Response Headers and Cookies

```java
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import jakarta.servlet.http.*;
import java.util.*;

@RestController
@RequestMapping("/api/headers")
public class HeadersController {
    
    // Access request headers
    @GetMapping("/read")
    public Map<String, String> readHeaders(
            @RequestHeader("User-Agent") String userAgent,
            @RequestHeader(value = "Accept-Language", defaultValue = "en") String lang,
            @RequestHeader(value = "Authorization", required = false) String auth,
            @RequestHeader HttpHeaders allHeaders) {
        
        Map<String, String> result = new HashMap<>();
        result.put("userAgent", userAgent);
        result.put("language", lang);
        result.put("authenticated", auth != null ? "yes" : "no");
        result.put("contentType", String.valueOf(allHeaders.getContentType()));
        return result;
    }
    
    // Set response headers
    @GetMapping("/write")
    public ResponseEntity<String> writeHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.add("X-Custom-Header", "my-value");
        headers.add("X-Request-Id", UUID.randomUUID().toString());
        headers.setCacheControl(CacheControl.noCache());
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        return ResponseEntity.ok()
            .headers(headers)
            .body("{\"status\": \"ok\"}");
    }
    
    // Cookie handling
    @GetMapping("/cookies/read")
    public Map<String, String> readCookies(
            @CookieValue(value = "sessionId", required = false) String sessionId,
            @CookieValue(value = "theme", defaultValue = "light") String theme) {
        return Map.of("sessionId", sessionId != null ? sessionId : "none", 
                      "theme", theme);
    }
    
    @GetMapping("/cookies/write")
    public ResponseEntity<String> writeCookies(HttpServletResponse response) {
        Cookie sessionCookie = new Cookie("sessionId", UUID.randomUUID().toString());
        sessionCookie.setPath("/");
        sessionCookie.setHttpOnly(true);
        sessionCookie.setSecure(true);
        sessionCookie.setMaxAge(3600);  // 1 hour
        response.addCookie(sessionCookie);
        
        // Using ResponseCookie (Spring 5.3+)
        org.springframework.http.ResponseCookie cookie = 
            org.springframework.http.ResponseCookie.from("theme", "dark")
                .path("/")
                .httpOnly(false)
                .secure(false)
                .maxAge(java.time.Duration.ofDays(365))
                .sameSite("Lax")
                .build();
        
        return ResponseEntity.ok()
            .header(HttpHeaders.SET_COOKIE, cookie.toString())
            .body("Cookies set");
    }
}
```

---

## 8. File Upload and Download

```java
import org.springframework.core.io.*;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import java.io.*;
import java.nio.file.*;
import java.util.*;

@RestController
@RequestMapping("/api/files")
public class FileController {
    
    private static final String UPLOAD_DIR = System.getProperty("java.io.tmpdir") + "/uploads/";
    
    // Single file upload
    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> uploadFile(
            @RequestParam("file") MultipartFile file) {
        
        if (file.isEmpty()) {
            return ResponseEntity.badRequest()
                .body(Map.of("error", "File is empty"));
        }
        
        String filename = UUID.randomUUID() + "_" + 
            org.springframework.util.StringUtils.cleanPath(file.getOriginalFilename());
        
        try {
            Path uploadPath = Paths.get(UPLOAD_DIR);
            Files.createDirectories(uploadPath);
            
            Path filePath = uploadPath.resolve(filename);
            Files.copy(file.getInputStream(), filePath, StandardCopyOption.REPLACE_EXISTING);
            
            return ResponseEntity.ok(Map.of(
                "filename", filename,
                "size", String.valueOf(file.getSize()),
                "contentType", file.getContentType()
            ));
            
        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", "Failed to upload: " + e.getMessage()));
        }
    }
    
    // Multiple file upload
    @PostMapping("/upload-multiple")
    public ResponseEntity<List<Map<String, String>>> uploadMultiple(
            @RequestParam("files") List<MultipartFile> files) {
        
        List<Map<String, String>> results = new ArrayList<>();
        
        for (MultipartFile file : files) {
            if (!file.isEmpty()) {
                try {
                    Path uploadPath = Paths.get(UPLOAD_DIR);
                    Files.createDirectories(uploadPath);
                    
                    String filename = UUID.randomUUID() + "_" + file.getOriginalFilename();
                    Files.copy(file.getInputStream(), uploadPath.resolve(filename));
                    
                    results.add(Map.of("filename", filename, "status", "success"));
                } catch (IOException e) {
                    results.add(Map.of("filename", file.getOriginalFilename(), 
                                       "status", "failed"));
                }
            }
        }
        
        return ResponseEntity.ok(results);
    }
    
    // File download
    @GetMapping("/download/{filename}")
    public ResponseEntity<Resource> downloadFile(@PathVariable String filename) {
        try {
            Path filePath = Paths.get(UPLOAD_DIR).resolve(filename).normalize();
            Resource resource = new FileSystemResource(filePath);
            
            if (!resource.exists()) {
                return ResponseEntity.notFound().build();
            }
            
            String contentType = Files.probeContentType(filePath);
            if (contentType == null) contentType = "application/octet-stream";
            
            return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(contentType))
                .header(HttpHeaders.CONTENT_DISPOSITION, 
                    "attachment; filename=\"" + resource.getFilename() + "\"")
                .body(resource);
                
        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
    
    // Serve file inline (for images, PDFs)
    @GetMapping("/view/{filename}")
    public ResponseEntity<Resource> viewFile(@PathVariable String filename) {
        try {
            Path filePath = Paths.get(UPLOAD_DIR).resolve(filename).normalize();
            Resource resource = new FileSystemResource(filePath);
            
            if (!resource.exists()) {
                return ResponseEntity.notFound().build();
            }
            
            String contentType = Files.probeContentType(filePath);
            
            return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(contentType != null ? contentType : "application/octet-stream"))
                .header(HttpHeaders.CONTENT_DISPOSITION, 
                    "inline; filename=\"" + resource.getFilename() + "\"")
                .body(resource);
                
        } catch (IOException e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
}
```

---

## 9. Interceptors and Filters

```java
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.config.annotation.*;
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

// === Handler Interceptor ===
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {
    
    private static final String START_TIME_ATTR = "requestStartTime";
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                              Object handler) {
        request.setAttribute(START_TIME_ATTR, System.currentTimeMillis());
        System.out.printf("[%s] %s %s%n", 
            java.time.LocalTime.now(), request.getMethod(), request.getRequestURI());
        return true;  // Continue processing
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, org.springframework.web.servlet.ModelAndView modelAndView) {
        // Called after handler but before view rendering
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        Long startTime = (Long) request.getAttribute(START_TIME_ATTR);
        if (startTime != null) {
            long duration = System.currentTimeMillis() - startTime;
            System.out.printf("[%s] %s %s - %d (%dms)%n",
                java.time.LocalTime.now(), request.getMethod(), 
                request.getRequestURI(), response.getStatus(), duration);
        }
    }
}

// === Authentication Interceptor ===
@Component
public class ApiKeyInterceptor implements HandlerInterceptor {
    
    private static final String VALID_API_KEY = "secret-api-key";
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                              Object handler) throws Exception {
        
        // Skip for public endpoints
        String path = request.getRequestURI();
        if (path.startsWith("/api/public") || path.equals("/health")) {
            return true;
        }
        
        String apiKey = request.getHeader("X-API-Key");
        if (VALID_API_KEY.equals(apiKey)) {
            return true;
        }
        
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.getWriter().write("{\"error\": \"Invalid or missing API key\"}");
        return false;  // Stop processing
    }
}

// === Register Interceptors ===
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    private final RequestLoggingInterceptor loggingInterceptor;
    private final ApiKeyInterceptor apiKeyInterceptor;
    
    WebMvcConfig(RequestLoggingInterceptor loggingInterceptor, 
                 ApiKeyInterceptor apiKeyInterceptor) {
        this.loggingInterceptor = loggingInterceptor;
        this.apiKeyInterceptor = apiKeyInterceptor;
    }
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Add logging to all requests
        registry.addInterceptor(loggingInterceptor);
        
        // Add auth to API requests only
        registry.addInterceptor(apiKeyInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/public/**", "/api/auth/**");
    }
    
    // CORS configuration
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000", "https://myapp.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}

// === Servlet Filter (lower-level than Interceptor) ===
@Component
public class RequestIdFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                         FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        // Add request ID
        String requestId = UUID.randomUUID().toString();
        httpResponse.setHeader("X-Request-Id", requestId);
        
        chain.doFilter(request, response);  // Continue chain
    }
}
```

---

## 10. Complete REST API - Book Store

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;
import org.springframework.stereotype.*;
import jakarta.validation.constraints.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.time.LocalDate;

// === Domain ===
record Book(
    String id,
    String title,
    String author,
    String isbn,
    double price,
    String category,
    LocalDate publishedDate,
    boolean available
) {}

// === DTOs ===
record CreateBookRequest(
    @NotBlank String title,
    @NotBlank String author,
    @NotBlank @Pattern(regexp = "\\d{13}") String isbn,
    @Positive double price,
    @NotBlank String category,
    LocalDate publishedDate
) {}

record UpdateBookRequest(
    String title,
    String author,
    Double price,
    String category
) {}

record BookSummary(String id, String title, String author, double price) {
    static BookSummary from(Book book) {
        return new BookSummary(book.id(), book.title(), book.author(), book.price());
    }
}

record PagedResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages
) {}

// === Exceptions ===
class BookNotFoundException extends RuntimeException {
    BookNotFoundException(String id) { super("Book not found: " + id); }
}

class IsbnAlreadyExistsException extends RuntimeException {
    IsbnAlreadyExistsException(String isbn) { super("ISBN already exists: " + isbn); }
}

// === Service ===
@Service
class BookService {
    
    private final Map<String, Book> books = new ConcurrentHashMap<>();
    
    List<BookSummary> findAll(String category, int page, int size) {
        List<Book> filtered = books.values().stream()
            .filter(b -> category == null || b.category().equals(category))
            .sorted(Comparator.comparing(Book::title))
            .toList();
        
        int fromIndex = page * size;
        int toIndex = Math.min(fromIndex + size, filtered.size());
        
        if (fromIndex >= filtered.size()) return List.of();
        
        return filtered.subList(fromIndex, toIndex).stream()
            .map(BookSummary::from)
            .toList();
    }
    
    Book findById(String id) {
        Book book = books.get(id);
        if (book == null) throw new BookNotFoundException(id);
        return book;
    }
    
    Book create(CreateBookRequest request) {
        boolean isbnExists = books.values().stream()
            .anyMatch(b -> b.isbn().equals(request.isbn()));
        if (isbnExists) throw new IsbnAlreadyExistsException(request.isbn());
        
        String id = UUID.randomUUID().toString();
        Book book = new Book(id, request.title(), request.author(), request.isbn(),
            request.price(), request.category(), 
            request.publishedDate() != null ? request.publishedDate() : LocalDate.now(),
            true);
        books.put(id, book);
        return book;
    }
    
    Book update(String id, UpdateBookRequest request) {
        Book existing = findById(id);
        Book updated = new Book(
            id,
            request.title() != null ? request.title() : existing.title(),
            request.author() != null ? request.author() : existing.author(),
            existing.isbn(),
            request.price() != null ? request.price() : existing.price(),
            request.category() != null ? request.category() : existing.category(),
            existing.publishedDate(),
            existing.available()
        );
        books.put(id, updated);
        return updated;
    }
    
    void delete(String id) {
        if (!books.containsKey(id)) throw new BookNotFoundException(id);
        books.remove(id);
    }
    
    long count() { return books.size(); }
}

// === Controller ===
@RestController
@RequestMapping("/api/v1/books")
class BookController {
    
    private final BookService bookService;
    
    BookController(BookService bookService) {
        this.bookService = bookService;
    }
    
    @GetMapping
    public ResponseEntity<PagedResponse<BookSummary>> getAllBooks(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "10") @Min(1) @Max(100) int size) {
        
        List<BookSummary> books = bookService.findAll(category, page, size);
        long total = bookService.count();
        int totalPages = (int) Math.ceil((double) total / size);
        
        return ResponseEntity.ok(new PagedResponse<>(books, page, size, total, totalPages));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Book> getBook(@PathVariable String id) {
        return ResponseEntity.ok(bookService.findById(id));
    }
    
    @PostMapping
    public ResponseEntity<Book> createBook(@Valid @RequestBody CreateBookRequest request) {
        Book created = bookService.create(request);
        URI location = URI.create("/api/v1/books/" + created.id());
        return ResponseEntity.created(location).body(created);
    }
    
    @PatchMapping("/{id}")
    public ResponseEntity<Book> updateBook(
            @PathVariable String id,
            @RequestBody UpdateBookRequest request) {
        return ResponseEntity.ok(bookService.update(id, request));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteBook(@PathVariable String id) {
        bookService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    @GetMapping("/search")
    public ResponseEntity<List<BookSummary>> searchBooks(
            @RequestParam String query,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        // Simple text search
        List<BookSummary> results = bookService.findAll(null, 0, Integer.MAX_VALUE)
            .stream()
            .filter(b -> b.title().toLowerCase().contains(query.toLowerCase()) ||
                        b.author().toLowerCase().contains(query.toLowerCase()))
            .skip((long) page * size)
            .limit(size)
            .toList();
        
        return ResponseEntity.ok(results);
    }
}

// === Exception Handler ===
@RestControllerAdvice
class BookStoreExceptionHandler {
    
    @ExceptionHandler(BookNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(BookNotFoundException ex) {
        return Map.of("error", "Not Found", "message", ex.getMessage());
    }
    
    @ExceptionHandler(IsbnAlreadyExistsException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public Map<String, String> handleDuplicate(IsbnAlreadyExistsException ex) {
        return Map.of("error", "Conflict", "message", ex.getMessage());
    }
    
    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> handleValidation(
            org.springframework.web.bind.MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        
        return Map.of("error", "Validation Failed", "fields", errors);
    }
}

// === Application ===
@SpringBootApplication
public class BookStoreApplication {
    public static void main(String[] args) {
        SpringApplication.run(BookStoreApplication.class, args);
    }
}

/*
Test the API:
# Create book
curl -X POST http://localhost:8080/api/v1/books \
  -H "Content-Type: application/json" \
  -d '{"title":"Clean Code","author":"Robert Martin","isbn":"9780132350884","price":45.99,"category":"Technology"}'

# Get all books
curl http://localhost:8080/api/v1/books

# Get by ID
curl http://localhost:8080/api/v1/books/{id}

# Update
curl -X PATCH http://localhost:8080/api/v1/books/{id} \
  -H "Content-Type: application/json" \
  -d '{"price":39.99}'

# Delete
curl -X DELETE http://localhost:8080/api/v1/books/{id}

# Search
curl "http://localhost:8080/api/v1/books/search?query=clean"
*/
```

---

## สรุป Part 023

| หัวข้อ | Key Points |
|--------|-----------|
| @RestController | @Controller + @ResponseBody |
| @RequestMapping | URL mapping, method, headers, consumes, produces |
| @PathVariable | URL path parameters |
| @RequestParam | Query string parameters |
| @RequestBody | JSON request body |
| @RequestHeader | HTTP headers |
| @CookieValue | Cookie values |
| ResponseEntity | Full control over HTTP response |
| @Valid | Trigger bean validation |
| @ExceptionHandler | Handle specific exceptions |
| @RestControllerAdvice | Global exception handling |
| HandlerInterceptor | Pre/post processing of requests |
| Filter | Low-level servlet processing |

---

**Part 024:** Spring Boot Auto-configuration & Actuator
- How auto-configuration works
- application.properties / application.yml
- Spring Profiles in Boot
- Spring Boot Actuator - health, metrics, info
- Custom actuator endpoints
- Production-ready features
