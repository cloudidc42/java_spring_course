# Part 089: Security Auditing and Compliance

## Overview

Production systems must track who changed what and when. This part covers Spring Data JPA Auditing for automatic timestamps, Hibernate Envers for full entity history, audit log services, security event logging, GDPR compliance with data masking, and the right-to-be-forgotten implementation.

---

## 1. Spring Data JPA Auditing

```xml
<!-- pom.xml – included in spring-boot-starter-data-jpa -->
```

```java
// Enable JPA auditing in the application
@SpringBootApplication
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class Application { ... }

// Base entity with audit fields
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by", nullable = false, length = 100)
    private String updatedBy;
}

// Domain entities extend the base
@Entity
@Table(name = "products")
public class Product extends AuditableEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private BigDecimal price;
    private Integer stock;
}

@Entity
@Table(name = "orders")
public class Order extends AuditableEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private BigDecimal totalAmount;
}
```

---

## 2. AuditorAware Configuration

```java
// Provides the "current user" for @CreatedBy / @LastModifiedBy
@Component("auditorProvider")
public class SpringSecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .filter(auth -> auth.isAuthenticated() && !(auth instanceof AnonymousAuthenticationToken))
                .map(Authentication::getName);
    }
}

// For batch jobs or scheduled tasks, set a system user
@Configuration
public class AuditConfiguration {

    @Bean("auditorProvider")
    public AuditorAware<String> auditorProvider() {
        return () -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();

            if (auth == null || !auth.isAuthenticated() || auth instanceof AnonymousAuthenticationToken) {
                // Check for system context (batch jobs, scheduled tasks)
                String systemUser = SystemContext.getCurrentUser();
                return Optional.ofNullable(systemUser).or(() -> Optional.of("SYSTEM"));
            }

            // JWT-based authentication
            if (auth.getPrincipal() instanceof Jwt jwt) {
                return Optional.ofNullable(jwt.getClaimAsString("user_id"))
                        .or(() -> Optional.of(jwt.getSubject()));
            }

            return Optional.of(auth.getName());
        };
    }
}

// System context for batch/scheduled jobs
public class SystemContext {

    private static final ThreadLocal<String> currentUser = new ThreadLocal<>();

    public static void runAs(String userId, Runnable task) {
        currentUser.set(userId);
        try {
            task.run();
        } finally {
            currentUser.remove();
        }
    }

    public static String getCurrentUser() {
        return currentUser.get();
    }
}

// Usage in a scheduled job
@Scheduled(cron = "0 0 2 * * *")
public void nightlyPriceRecalculation() {
    SystemContext.runAs("SCHEDULER-001", () -> {
        priceService.recalculateAll();  // changes tracked with "SCHEDULER-001" as user
    });
}
```

---

## 3. Custom Auditing with AuditingEntityListener

```java
// Extended audit listener with change detection
@Component
public class ExtendedAuditingEntityListener {

    private static final Logger log = LoggerFactory.getLogger(ExtendedAuditingEntityListener.class);

    @PrePersist
    public void prePersist(AuditableEntity entity) {
        Instant now = Instant.now();
        entity.setCreatedAt(now);
        entity.setUpdatedAt(now);
        entity.setCreatedBy(getCurrentUser());
        entity.setUpdatedBy(getCurrentUser());
    }

    @PreUpdate
    public void preUpdate(AuditableEntity entity) {
        entity.setUpdatedAt(Instant.now());
        entity.setUpdatedBy(getCurrentUser());
    }

    private String getCurrentUser() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .map(Authentication::getName)
                .orElse("SYSTEM");
    }
}

// Version tracking entity
@MappedSuperclass
public abstract class VersionedAuditableEntity extends AuditableEntity {

    @Version
    @Column(name = "version", nullable = false)
    private Long version = 0L;

    // Soft delete support
    @Column(name = "deleted_at")
    private Instant deletedAt;

    @Column(name = "deleted_by", length = 100)
    private String deletedBy;

    public boolean isDeleted() {
        return deletedAt != null;
    }

    public void softDelete() {
        this.deletedAt = Instant.now();
        this.deletedBy = Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .map(Authentication::getName).orElse("SYSTEM");
    }
}
```

---

## 4. Hibernate Envers – Entity History

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

```java
// Enable versioning on an entity
@Entity
@Table(name = "products")
@Audited  // Envers will track all changes
public class Product extends AuditableEntity {

    @Id
    @GeneratedValue
    private Long id;

    @Audited
    private String name;

    @Audited
    private BigDecimal price;

    @Audited
    private Integer stock;

    @NotAudited  // exclude from history (e.g., computed fields)
    private String searchVector;
}

// Custom revision entity to capture extra metadata
@Entity
@RevisionEntity(CustomRevisionListener.class)
@Table(name = "revinfo")
public class CustomRevisionEntity extends DefaultRevisionEntity {

    @Column(name = "user_id", length = 100)
    private String userId;

    @Column(name = "ip_address", length = 50)
    private String ipAddress;

    @Column(name = "request_id", length = 100)
    private String requestId;

    @Enumerated(EnumType.STRING)
    @Column(name = "revision_type")
    private RevisionType revisionType;
}

@Component
public class CustomRevisionListener implements RevisionListener {

    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevisionEntity rev = (CustomRevisionEntity) revisionEntity;

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null) {
            rev.setUserId(auth.getName());
        }

        // Try to get request metadata
        try {
            ServletRequestAttributes attrs =
                    (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
            HttpServletRequest request = attrs.getRequest();
            rev.setIpAddress(getClientIp(request));
            rev.setRequestId(request.getHeader("X-Request-ID"));
        } catch (IllegalStateException e) {
            // Not in a request context (e.g., batch job)
            rev.setIpAddress("batch");
        }
    }

    private String getClientIp(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff != null && !xff.isBlank()) {
            return xff.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

### Querying Envers History

```java
@Service
public class ProductAuditService {

    @PersistenceContext
    private EntityManager entityManager;

    // Get all revisions of a product
    public List<ProductRevision> getHistory(Long productId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        List<Number> revisionNumbers = reader.getRevisions(Product.class, productId);

        return revisionNumbers.stream()
                .map(revNum -> {
                    Product snapshot = reader.find(Product.class, productId, revNum);
                    CustomRevisionEntity rev = reader.findRevision(CustomRevisionEntity.class, revNum);

                    return ProductRevision.builder()
                            .revisionNumber(revNum.longValue())
                            .revisionDate(rev.getRevisionDate().toInstant())
                            .revisedBy(rev.getUserId())
                            .ipAddress(rev.getIpAddress())
                            .product(snapshot)
                            .build();
                })
                .collect(Collectors.toList());
    }

    // Get product state at a specific point in time
    public Optional<Product> getProductAtTime(Long productId, Instant timestamp) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        try {
            Product snapshot = reader.find(Product.class, productId, Date.from(timestamp));
            return Optional.ofNullable(snapshot);
        } catch (NotValidDataAccessApiUsageException e) {
            return Optional.empty();
        }
    }

    // Query using Audit Query API
    public List<Product> getProductChangesInDateRange(Instant from, Instant to) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        return (List<Product>) reader.createQuery()
                .forRevisionsOfEntity(Product.class, true, false)
                .addOrder(AuditEntity.revisionNumber().asc())
                .add(AuditEntity.revisionProperty("timestamp").gt(from.toEpochMilli()))
                .add(AuditEntity.revisionProperty("timestamp").lt(to.toEpochMilli()))
                .getResultList();
    }

    // Get entities changed by a specific user
    public List<ProductAuditEntry> getChangedBy(String userId) {
        AuditReader reader = AuditReaderFactory.get(entityManager);

        List<Object[]> results = reader.createQuery()
                .forRevisionsOfEntity(Product.class, false, true)
                .add(AuditEntity.revisionProperty("userId").eq(userId))
                .getResultList();

        return results.stream()
                .map(row -> {
                    Product product = (Product) row[0];
                    CustomRevisionEntity rev = (CustomRevisionEntity) row[1];
                    RevisionType type = (RevisionType) row[2];
                    return new ProductAuditEntry(product, rev, type);
                })
                .collect(Collectors.toList());
    }
}
```

---

## 5. Audit Log Service

```java
// Dedicated audit log table separate from Envers (for business events, not just data changes)

@Entity
@Table(name = "audit_logs", indexes = {
    @Index(name = "idx_audit_entity", columnList = "entity_type, entity_id"),
    @Index(name = "idx_audit_user", columnList = "user_id"),
    @Index(name = "idx_audit_time", columnList = "created_at"),
    @Index(name = "idx_audit_action", columnList = "action")
})
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", length = 100)
    private String userId;

    @Column(name = "user_email", length = 200)
    private String userEmail;

    @Column(name = "action", nullable = false, length = 100)
    @Enumerated(EnumType.STRING)
    private AuditAction action;

    @Column(name = "entity_type", nullable = false, length = 100)
    private String entityType;

    @Column(name = "entity_id", length = 100)
    private String entityId;

    @Column(name = "description", length = 500)
    private String description;

    @Column(name = "old_value", columnDefinition = "jsonb")
    private String oldValue;

    @Column(name = "new_value", columnDefinition = "jsonb")
    private String newValue;

    @Column(name = "ip_address", length = 50)
    private String ipAddress;

    @Column(name = "user_agent", length = 500)
    private String userAgent;

    @Column(name = "request_id", length = 100)
    private String requestId;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "metadata", columnDefinition = "jsonb")
    private String metadata;
}

public enum AuditAction {
    CREATE, UPDATE, DELETE, VIEW, LOGIN, LOGOUT,
    EXPORT, IMPORT, APPROVE, REJECT, LOCK, UNLOCK,
    PASSWORD_CHANGE, PERMISSION_CHANGE
}

// Audit log service
@Service
@Slf4j
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;
    private final ObjectMapper objectMapper;
    private final ApplicationEventPublisher eventPublisher;

    @Async("auditExecutor")
    public void log(AuditEntry entry) {
        try {
            AuditLog auditLog = buildAuditLog(entry);
            auditLogRepository.save(auditLog);
            eventPublisher.publishEvent(new AuditEvent(auditLog));
        } catch (Exception e) {
            log.error("Failed to write audit log entry: {}", entry, e);
            // Audit failures should not break the business operation
        }
    }

    private AuditLog buildAuditLog(AuditEntry entry) {
        return AuditLog.builder()
                .userId(entry.getUserId())
                .userEmail(entry.getUserEmail())
                .action(entry.getAction())
                .entityType(entry.getEntityType())
                .entityId(entry.getEntityId())
                .description(entry.getDescription())
                .oldValue(toJson(entry.getOldValue()))
                .newValue(toJson(entry.getNewValue()))
                .ipAddress(entry.getIpAddress())
                .userAgent(entry.getUserAgent())
                .requestId(entry.getRequestId())
                .createdAt(Instant.now())
                .metadata(toJson(entry.getMetadata()))
                .build();
    }

    private String toJson(Object value) {
        if (value == null) return null;
        try {
            return objectMapper.writeValueAsString(value);
        } catch (JsonProcessingException e) {
            return value.toString();
        }
    }

    // Query audit logs
    public Page<AuditLog> findLogs(AuditLogFilter filter, Pageable pageable) {
        return auditLogRepository.findAll(buildSpec(filter), pageable);
    }

    private Specification<AuditLog> buildSpec(AuditLogFilter filter) {
        return Specification
                .where(filter.getUserId() != null ?
                        (root, q, cb) -> cb.equal(root.get("userId"), filter.getUserId()) : null)
                .and(filter.getAction() != null ?
                        (root, q, cb) -> cb.equal(root.get("action"), filter.getAction()) : null)
                .and(filter.getEntityType() != null ?
                        (root, q, cb) -> cb.equal(root.get("entityType"), filter.getEntityType()) : null)
                .and(filter.getFrom() != null ?
                        (root, q, cb) -> cb.greaterThanOrEqualTo(root.get("createdAt"), filter.getFrom()) : null)
                .and(filter.getTo() != null ?
                        (root, q, cb) -> cb.lessThanOrEqualTo(root.get("createdAt"), filter.getTo()) : null);
    }
}

// AOP-based automatic audit logging
@Aspect
@Component
public class AuditLoggingAspect {

    private final AuditLogService auditLogService;
    private final HttpServletRequest httpRequest;

    @Around("@annotation(audited)")
    public Object auditMethod(ProceedingJoinPoint jp, Audited audited) throws Throwable {
        Object oldValue = null;
        Object newValue = null;
        String entityId = null;

        // Capture before state
        if (audited.captureOldValue()) {
            try {
                Object arg = jp.getArgs()[0];
                if (arg instanceof Long id) {
                    oldValue = loadEntity(audited.entityType(), id);
                    entityId = id.toString();
                }
            } catch (Exception e) {
                log.debug("Could not capture old value", e);
            }
        }

        // Execute
        Object result = jp.proceed();

        // Capture after state
        if (audited.captureNewValue() && result != null) {
            newValue = result;
            if (result instanceof AuditableEntity entity) {
                entityId = entity.getId() != null ? entity.getId().toString() : entityId;
            }
        }

        // Log
        auditLogService.log(AuditEntry.builder()
                .userId(getCurrentUserId())
                .userEmail(getCurrentUserEmail())
                .action(audited.action())
                .entityType(audited.entityType())
                .entityId(entityId)
                .description(audited.description())
                .oldValue(oldValue)
                .newValue(newValue)
                .ipAddress(getClientIp())
                .userAgent(httpRequest.getHeader("User-Agent"))
                .requestId(httpRequest.getHeader("X-Request-ID"))
                .build());

        return result;
    }

    private String getCurrentUserId() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .map(Authentication::getName).orElse("ANONYMOUS");
    }
}

// Annotation for methods to audit
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    AuditAction action();
    String entityType();
    String description() default "";
    boolean captureOldValue() default false;
    boolean captureNewValue() default true;
}

// Usage
@Service
public class ProductService {

    @Audited(action = AuditAction.CREATE, entityType = "Product", captureNewValue = true)
    public Product createProduct(CreateProductRequest request) { ... }

    @Audited(action = AuditAction.UPDATE, entityType = "Product",
             captureOldValue = true, captureNewValue = true)
    public Product updateProduct(Long id, UpdateProductRequest request) { ... }

    @Audited(action = AuditAction.DELETE, entityType = "Product", captureOldValue = true)
    public void deleteProduct(Long id) { ... }
}
```

---

## 6. Security Event Logging

```java
// Spring Security audit events
@Component
public class SecurityEventLogger implements ApplicationListener<AbstractAuthenticationEvent> {

    private final AuditLogService auditLogService;

    @Override
    public void onApplicationEvent(AbstractAuthenticationEvent event) {
        if (event instanceof AuthenticationSuccessEvent success) {
            auditLogService.log(AuditEntry.builder()
                    .userId(success.getAuthentication().getName())
                    .action(AuditAction.LOGIN)
                    .entityType("Authentication")
                    .description("Successful login")
                    .build());
        } else if (event instanceof AbstractAuthenticationFailureEvent failure) {
            auditLogService.log(AuditEntry.builder()
                    .userId(failure.getAuthentication().getName())
                    .action(AuditAction.LOGIN)
                    .entityType("Authentication")
                    .description("Failed login: " + failure.getException().getMessage())
                    .metadata(Map.of("reason", failure.getException().getClass().getSimpleName()))
                    .build());
        }
    }
}

// Log authorization failures
@Component
public class SecurityAuditEventHandler {

    private final AuditLogService auditLogService;

    @EventListener
    public void onAuthorizationDenied(AuthorizationDeniedEvent<?> event) {
        Authentication auth = event.getAuthentication().get();
        auditLogService.log(AuditEntry.builder()
                .userId(auth.getName())
                .action(AuditAction.VIEW)
                .entityType("AccessControl")
                .description("Access denied: " + event.getAuthorizationDecision().toString())
                .build());
    }
}
```

---

## 7. GDPR Compliance – Data Masking

```java
// PII field masking for audit logs and logging
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonSerialize(using = MaskedFieldSerializer.class)
public @interface MaskedField {
    MaskType type() default MaskType.PARTIAL;

    enum MaskType {
        FULL,        // ****
        PARTIAL,     // jo**@ex*****.com
        EMAIL,       // j***@example.com
        PHONE,       // ***-***-1234
        CREDIT_CARD  // ****-****-****-1234
    }
}

public class MaskedFieldSerializer extends JsonSerializer<String> {

    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider provider)
            throws IOException {
        if (value == null) {
            gen.writeNull();
            return;
        }

        JsonStreamContext context = gen.getOutputContext();
        String fieldName = context.getCurrentName();
        // Look up annotation from the enclosing object
        // In a real impl, you'd use BeanProperty or serialize with context
        gen.writeString(maskPartial(value));
    }

    public static String maskEmail(String email) {
        if (email == null || !email.contains("@")) return "***";
        String[] parts = email.split("@");
        String local = parts[0];
        String domain = parts[1];
        String maskedLocal = local.length() <= 2 ?
                "*".repeat(local.length()) :
                local.charAt(0) + "*".repeat(local.length() - 1);
        return maskedLocal + "@" + domain;
    }

    public static String maskPhone(String phone) {
        if (phone == null || phone.length() < 4) return "****";
        return "*".repeat(phone.length() - 4) + phone.substring(phone.length() - 4);
    }

    public static String maskCreditCard(String cardNumber) {
        if (cardNumber == null) return null;
        String digits = cardNumber.replaceAll("[^0-9]", "");
        if (digits.length() < 4) return "****";
        return "*".repeat(digits.length() - 4) + digits.substring(digits.length() - 4);
    }

    public static String maskPartial(String value) {
        if (value == null || value.length() <= 2) return "**";
        int visibleChars = Math.max(1, value.length() / 4);
        return value.substring(0, visibleChars) + "*".repeat(value.length() - visibleChars);
    }
}

// DTO with masked fields for logging/display
public class CustomerAuditDto {

    private String id;

    @MaskedField(type = MaskedField.MaskType.EMAIL)
    private String email;

    @MaskedField(type = MaskedField.MaskType.PARTIAL)
    private String firstName;

    @MaskedField(type = MaskedField.MaskType.FULL)
    private String lastName;

    @MaskedField(type = MaskedField.MaskType.PHONE)
    private String phone;

    @MaskedField(type = MaskedField.MaskType.CREDIT_CARD)
    private String defaultCardNumber;
}

// Logback masking for log output
public class PiiMaskingConverter extends ClassicConverter {

    private static final Pattern EMAIL_PATTERN =
            Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b");

    private static final Pattern CARD_PATTERN =
            Pattern.compile("\\b(?:\\d[ -]?){13,16}\\b");

    @Override
    public String convert(ILoggingEvent event) {
        String message = event.getFormattedMessage();
        message = EMAIL_PATTERN.matcher(message).replaceAll(m -> maskEmail(m.group()));
        message = CARD_PATTERN.matcher(message).replaceAll("****-****-****-****");
        return message;
    }

    private String maskEmail(String email) {
        return MaskedFieldSerializer.maskEmail(email);
    }
}
```

---

## 8. GDPR – Right to Be Forgotten

```java
// Customer data erasure service
@Service
@Slf4j
public class CustomerDataErasureService {

    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    private final AuditLogRepository auditLogRepository;
    private final AuditLogService auditLogService;
    private final EmailService emailService;

    @Transactional
    public ErasureResult eraseCustomerData(String customerId, String requestedBy) {
        log.info("Processing data erasure request for customer {} by {}", customerId, requestedBy);

        Customer customer = customerRepository.findById(customerId)
                .orElseThrow(() -> new CustomerNotFoundException(customerId));

        // Check if erasure is legally permissible
        validateErasureEligibility(customer);

        String originalEmail = customer.getEmail();
        ErasureResult.Builder result = ErasureResult.builder()
                .customerId(customerId)
                .requestedBy(requestedBy)
                .erasedAt(Instant.now());

        // 1. Anonymize customer personal data (preserve for legal/financial records)
        customer.anonymize();
        customerRepository.save(customer);
        result.step("Customer PII anonymized");

        // 2. Anonymize order history (keep order data for accounting, remove PII)
        int ordersAnonymized = orderRepository.anonymizeByCustomerId(customerId);
        result.step("Orders anonymized: " + ordersAnonymized);

        // 3. Delete non-essential data
        reviewRepository.deleteByCustomerId(customerId);
        wishlistRepository.deleteByCustomerId(customerId);
        result.step("Reviews and wishlist deleted");

        // 4. Anonymize audit logs (keep audit trail, anonymize identity)
        auditLogRepository.anonymizeByUserId(customerId);
        result.step("Audit logs anonymized");

        // 5. Record the erasure request itself
        auditLogService.log(AuditEntry.builder()
                .userId(requestedBy)
                .action(AuditAction.DELETE)
                .entityType("CustomerData")
                .entityId(customerId)
                .description("GDPR data erasure completed")
                .metadata(Map.of(
                        "requestedBy", requestedBy,
                        "erasedAt", Instant.now().toString()
                ))
                .build());

        // 6. Send confirmation
        emailService.sendErasureConfirmation(originalEmail, result.build());

        return result.build();
    }

    private void validateErasureEligibility(Customer customer) {
        // Cannot erase if there are active legal obligations
        if (orderRepository.hasActiveOrders(customer.getId())) {
            throw new ErasureNotPermittedException(
                    "Cannot erase data for customer with active orders. " +
                    "Please wait until all orders are completed or cancelled.");
        }

        if (subscriptionRepository.hasActiveSubscriptions(customer.getId())) {
            throw new ErasureNotPermittedException(
                    "Cannot erase data for customer with active subscriptions.");
        }
    }
}

// Customer anonymization
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    private String id;
    private String email;
    private String firstName;
    private String lastName;
    private String phone;
    private boolean deleted;

    public void anonymize() {
        this.email = "deleted-" + this.id + "@deleted.invalid";
        this.firstName = "Deleted";
        this.lastName = "User";
        this.phone = null;
        this.deleted = true;
    }
}

// GDPR erasure endpoint
@RestController
@RequestMapping("/api/gdpr")
@Slf4j
public class GdprController {

    private final CustomerDataErasureService erasureService;
    private final CustomerDataExportService exportService;

    // Right to be forgotten
    @DeleteMapping("/customers/{id}/data")
    @PreAuthorize("hasRole('ADMIN') or @gdprSecurity.isOwnRequest(#id, authentication)")
    public ResponseEntity<ErasureResult> eraseCustomerData(
            @PathVariable String id,
            @AuthenticationPrincipal Jwt jwt) {

        ErasureResult result = erasureService.eraseCustomerData(id, jwt.getSubject());
        return ResponseEntity.ok(result);
    }

    // Right to data portability
    @GetMapping("/customers/{id}/export")
    @PreAuthorize("hasRole('ADMIN') or @gdprSecurity.isOwnRequest(#id, authentication)")
    public ResponseEntity<Resource> exportCustomerData(@PathVariable String id) {
        byte[] exportData = exportService.exportCustomerData(id);

        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType("application/zip"))
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"customer-data-export.zip\"")
                .body(new ByteArrayResource(exportData));
    }

    // Data access request
    @GetMapping("/customers/{id}/data-summary")
    @PreAuthorize("hasRole('ADMIN') or @gdprSecurity.isOwnRequest(#id, authentication)")
    public ResponseEntity<CustomerDataSummary> getDataSummary(@PathVariable String id) {
        return ResponseEntity.ok(exportService.getDataSummary(id));
    }
}
```

---

## 9. PII Data Handling

```java
// Encrypt PII fields at database level
@Entity
@Table(name = "customers")
public class Customer {

    @Id
    private String id;

    // Encrypted at DB level using converter
    @Column(name = "email_encrypted")
    @Convert(converter = EncryptedStringConverter.class)
    private String email;

    // Store hash for lookup (email is searchable but PII is protected)
    @Column(name = "email_hash", unique = true)
    private String emailHash;

    public void setEmail(String email) {
        this.email = email;
        this.emailHash = HashUtils.sha256(email.toLowerCase());
    }
}

@Converter
@Component
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    private final EncryptionService encryptionService;

    @Override
    public String convertToDatabaseColumn(String plainText) {
        if (plainText == null) return null;
        return encryptionService.encrypt(plainText);
    }

    @Override
    public String convertToEntityAttribute(String cipherText) {
        if (cipherText == null) return null;
        return encryptionService.decrypt(cipherText);
    }
}

@Service
public class EncryptionService {

    private final SecretKey secretKey;

    public EncryptionService(@Value("${app.encryption.key}") String base64Key) {
        byte[] keyBytes = Base64.getDecoder().decode(base64Key);
        this.secretKey = new SecretKeySpec(keyBytes, "AES");
    }

    public String encrypt(String plainText) {
        try {
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            byte[] iv = new byte[12];
            new SecureRandom().nextBytes(iv);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, new GCMParameterSpec(128, iv));

            byte[] encrypted = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));
            byte[] combined = new byte[iv.length + encrypted.length];
            System.arraycopy(iv, 0, combined, 0, iv.length);
            System.arraycopy(encrypted, 0, combined, iv.length, encrypted.length);

            return Base64.getEncoder().encodeToString(combined);
        } catch (Exception e) {
            throw new EncryptionException("Encryption failed", e);
        }
    }

    public String decrypt(String cipherText) {
        try {
            byte[] combined = Base64.getDecoder().decode(cipherText);
            byte[] iv = Arrays.copyOfRange(combined, 0, 12);
            byte[] encrypted = Arrays.copyOfRange(combined, 12, combined.length);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE, secretKey, new GCMParameterSpec(128, iv));

            return new String(cipher.doFinal(encrypted), StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new EncryptionException("Decryption failed", e);
        }
    }
}
```

---

## 10. Audit Report Generation

```java
@Service
public class AuditReportService {

    private final AuditLogRepository auditLogRepository;
    private final ProductAuditService productAuditService;

    // Generate compliance report
    public ComplianceReport generateReport(LocalDate from, LocalDate to) {
        Instant fromInstant = from.atStartOfDay(ZoneOffset.UTC).toInstant();
        Instant toInstant = to.plusDays(1).atStartOfDay(ZoneOffset.UTC).toInstant();

        List<AuditLog> logs = auditLogRepository.findByCreatedAtBetween(fromInstant, toInstant);

        return ComplianceReport.builder()
                .period(from + " to " + to)
                .generatedAt(Instant.now())
                .totalEvents(logs.size())
                .eventsByAction(groupByAction(logs))
                .eventsByUser(groupByUser(logs))
                .failedLoginAttempts(countFailedLogins(logs))
                .dataExports(countExports(logs))
                .dataErasures(countErasures(logs))
                .build();
    }

    private Map<String, Long> groupByAction(List<AuditLog> logs) {
        return logs.stream()
                .collect(Collectors.groupingBy(
                        l -> l.getAction().name(),
                        Collectors.counting()));
    }

    private Map<String, Long> groupByUser(List<AuditLog> logs) {
        return logs.stream()
                .filter(l -> l.getUserId() != null)
                .collect(Collectors.groupingBy(AuditLog::getUserId, Collectors.counting()));
    }
}

// Export audit logs to CSV
@RestController
@RequestMapping("/api/admin/audit")
@PreAuthorize("hasRole('COMPLIANCE_OFFICER')")
public class AuditExportController {

    private final AuditLogService auditLogService;

    @GetMapping(value = "/export", produces = "text/csv")
    public ResponseEntity<StreamingResponseBody> exportAuditLogs(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to) {

        StreamingResponseBody body = outputStream -> {
            try (var writer = new PrintWriter(new OutputStreamWriter(outputStream))) {
                writer.println("id,userId,action,entityType,entityId,createdAt,ipAddress");

                auditLogService.streamLogs(from, to, log -> {
                    writer.printf("%d,%s,%s,%s,%s,%s,%s%n",
                            log.getId(),
                            log.getUserId(),
                            log.getAction(),
                            log.getEntityType(),
                            log.getEntityId(),
                            log.getCreatedAt(),
                            log.getIpAddress());
                });
            }
        };

        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=audit-" + from + "-" + to + ".csv")
                .body(body);
    }
}
```

---

## 11. Compliance Testing

```java
@SpringBootTest
class GdprComplianceTest {

    @Autowired
    private CustomerDataErasureService erasureService;

    @Autowired
    private CustomerRepository customerRepository;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private AuditLogRepository auditLogRepository;

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void shouldEraseAllPersonalData() {
        // Setup: create customer with data
        Customer customer = createCustomerWithData();
        String customerId = customer.getId();
        String originalEmail = customer.getEmail();

        // Execute erasure
        ErasureResult result = erasureService.eraseCustomerData(customerId, "admin");

        // Verify PII is gone
        Customer erased = customerRepository.findById(customerId).orElseThrow();
        assertThat(erased.getEmail()).doesNotContain("@").contains("deleted");
        assertThat(erased.getFirstName()).isEqualTo("Deleted");
        assertThat(erased.getLastName()).isEqualTo("User");
        assertThat(erased.getPhone()).isNull();

        // Verify audit logs don't contain original email
        List<AuditLog> logs = auditLogRepository.findAll();
        logs.forEach(log -> {
            assertThat(log.getDescription()).doesNotContain(originalEmail);
            if (log.getNewValue() != null) {
                assertThat(log.getNewValue()).doesNotContain(originalEmail);
            }
        });

        // Verify erasure was recorded
        assertThat(result.getSteps()).isNotEmpty();
    }

    @Test
    void shouldNotEraseDataWithActiveOrders() {
        Customer customer = createCustomerWithActiveOrder();

        assertThatThrownBy(() -> erasureService.eraseCustomerData(
                customer.getId(), "admin"))
                .isInstanceOf(ErasureNotPermittedException.class)
                .hasMessageContaining("active orders");
    }
}
```

---

## Summary

| Feature | Implementation |
|---|---|
| Automatic timestamps | `@CreatedDate`, `@LastModifiedDate` + `@EnableJpaAuditing` |
| Automatic user tracking | `@CreatedBy`, `@LastModifiedBy` + `AuditorAware` |
| Entity history | Hibernate Envers `@Audited` |
| Custom revision metadata | `@RevisionEntity` + `RevisionListener` |
| Business audit log | `AuditLog` entity + `AuditLogService` |
| Auto-audit via AOP | `@Audited` annotation + `AuditLoggingAspect` |
| Security events | `ApplicationListener<AbstractAuthenticationEvent>` |
| PII masking | `@MaskedField` + Jackson serializer |
| PII encryption at rest | `@Convert` + `AttributeConverter` |
| GDPR erasure | `CustomerDataErasureService` + anonymization |
| Data export | Streaming CSV with `StreamingResponseBody` |
| Compliance reports | `AuditReportService` |

## Next Part Preview

**Part 090** covers Production Readiness — connection pool sizing, JVM tuning, graceful shutdown, secret management, metrics/alerting, and a production-ready Spring Boot checklist.
