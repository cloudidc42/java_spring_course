# Part 89: Spring Security Auditing and Compliance

Production applications must record who changed what data and when, enforce fine-grained access
rules on individual records, and keep an immutable revision history for compliance. This part
covers every layer from method-level security through ACL to GDPR-ready audit trails.

---

## 1. Enabling Method Security

```java
// src/main/java/com/example/audit/config/SecurityConfig.java
package com.example.audit.config;

import com.example.audit.security.CustomPermissionEvaluator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler;
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(
    prePostEnabled  = true,  // @PreAuthorize / @PostAuthorize
    securedEnabled  = true,  // @Secured
    jsr250Enabled   = true   // @RolesAllowed
)
public class SecurityConfig {

    @Bean
    public MethodSecurityExpressionHandler methodSecurityExpressionHandler(
            CustomPermissionEvaluator permissionEvaluator) {
        DefaultMethodSecurityExpressionHandler handler =
                new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(permissionEvaluator);
        return handler;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(c -> c.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

---

## 2. @PreAuthorize and @PostAuthorize with SpEL

```java
// src/main/java/com/example/audit/service/DocumentService.java
package com.example.audit.service;

import com.example.audit.domain.Document;
import com.example.audit.domain.DocumentStatus;
import com.example.audit.repository.DocumentRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class DocumentService {

    private final DocumentRepository documentRepository;

    // User must have ROLE_ADMIN or own the document (ownerUsername == current user)
    @PreAuthorize("hasRole('ADMIN') or #ownerUsername == authentication.name")
    public List<Document> findByOwner(String ownerUsername) {
        return documentRepository.findByOwnerUsername(ownerUsername);
    }

    // Load first, then check: the returned object must be owned by the caller
    @PostAuthorize("hasRole('ADMIN') or returnObject.ownerUsername == authentication.name")
    public Document findById(Long id) {
        return documentRepository.findById(id)
                .orElseThrow(() -> new DocumentNotFoundException(id));
    }

    // Only ADMIN or the document owner may delete it; document must be in DRAFT status
    @PreAuthorize("""
        (hasRole('ADMIN') or @documentOwnerChecker.isOwner(#id, authentication.name))
        and @documentOwnerChecker.isDraft(#id)
        """)
    @Transactional
    public void delete(Long id) {
        documentRepository.deleteById(id);
    }

    // Custom permission evaluator: hasPermission(domainObject, permission)
    @PreAuthorize("hasPermission(#id, 'Document', 'WRITE')")
    @Transactional
    public Document update(Long id, String content) {
        Document doc = documentRepository.findById(id)
                .orElseThrow(() -> new DocumentNotFoundException(id));
        doc.setContent(content);
        return documentRepository.save(doc);
    }

    // Only managers of the document's department may approve it
    @PreAuthorize("hasRole('MANAGER') and @departmentChecker.manages(authentication, #id)")
    @Transactional
    public Document approve(Long id) {
        Document doc = documentRepository.findById(id)
                .orElseThrow(() -> new DocumentNotFoundException(id));
        doc.setStatus(DocumentStatus.APPROVED);
        return documentRepository.save(doc);
    }
}
```

---

## 3. Custom Permission Evaluator

```java
// src/main/java/com/example/audit/security/CustomPermissionEvaluator.java
package com.example.audit.security;

import com.example.audit.domain.AclEntry;
import com.example.audit.repository.AclEntryRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

import java.io.Serializable;

@Component
@RequiredArgsConstructor
public class CustomPermissionEvaluator implements PermissionEvaluator {

    private final AclEntryRepository aclEntryRepository;

    @Override
    public boolean hasPermission(Authentication authentication,
                                 Object targetDomainObject,
                                 Object permission) {
        if (authentication == null || targetDomainObject == null) {
            return false;
        }
        // Fall back to class-based check for domain objects passed directly
        return checkPermission(authentication.getName(),
                targetDomainObject.getClass().getSimpleName(),
                Long.parseLong(targetDomainObject.toString()),
                permission.toString());
    }

    @Override
    public boolean hasPermission(Authentication authentication,
                                 Serializable targetId,
                                 String targetType,
                                 Object permission) {
        if (authentication == null || targetId == null) {
            return false;
        }
        return checkPermission(authentication.getName(),
                targetType, (Long) targetId, permission.toString());
    }

    private boolean checkPermission(String username,
                                    String objectType,
                                    Long objectId,
                                    String permission) {
        // Check ACL table: does this user (or any of their granted roles) have this permission?
        return aclEntryRepository.hasPermission(username, objectType, objectId, permission);
    }
}
```

---

## 4. ACL (Access Control Lists)

```java
// src/main/java/com/example/audit/domain/AclEntry.java
package com.example.audit.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "acl_entries",
       uniqueConstraints = @UniqueConstraint(
               columnNames = {"principal", "object_type", "object_id", "permission"}))
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class AclEntry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String principal;          // username or ROLE_XXX

    @Column(name = "object_type", nullable = false)
    private String objectType;         // e.g. "Document"

    @Column(name = "object_id", nullable = false)
    private Long objectId;

    @Column(nullable = false)
    private String permission;         // READ, WRITE, DELETE, APPROVE
}
```

```java
// src/main/java/com/example/audit/repository/AclEntryRepository.java
package com.example.audit.repository;

import com.example.audit.domain.AclEntry;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface AclEntryRepository extends JpaRepository<AclEntry, Long> {

    @Query("""
           SELECT COUNT(a) > 0 FROM AclEntry a
           WHERE a.objectType = :type
             AND a.objectId   = :id
             AND a.permission = :perm
             AND (a.principal = :username
                  OR a.principal IN (
                     SELECT 'ROLE_' || r FROM UserRole r WHERE r.username = :username))
           """)
    boolean hasPermission(@Param("username") String username,
                          @Param("type")     String objectType,
                          @Param("id")       Long   objectId,
                          @Param("perm")     String permission);
}
```

```java
// src/main/java/com/example/audit/service/AclService.java
package com.example.audit.service;

import com.example.audit.domain.AclEntry;
import com.example.audit.repository.AclEntryRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class AclService {

    private final AclEntryRepository aclEntryRepository;

    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public AclEntry grant(String principal, String objectType, Long objectId, String permission) {
        AclEntry entry = AclEntry.builder()
                .principal(principal)
                .objectType(objectType)
                .objectId(objectId)
                .permission(permission)
                .build();
        return aclEntryRepository.save(entry);
    }

    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public void revoke(String principal, String objectType, Long objectId, String permission) {
        aclEntryRepository.findAll().stream()
                .filter(e -> e.getPrincipal().equals(principal)
                        && e.getObjectType().equals(objectType)
                        && e.getObjectId().equals(objectId)
                        && e.getPermission().equals(permission))
                .forEach(aclEntryRepository::delete);
    }
}
```

---

## 5. Audit Logging with AuditingEntityListener

```java
// src/main/java/com/example/audit/config/AuditConfig.java
package com.example.audit.config;

import com.example.audit.security.SecurityAuditorAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorAware")
public class AuditConfig {

    @Bean
    public AuditorAware<String> auditorAware() {
        return new SecurityAuditorAware();
    }
}
```

```java
// src/main/java/com/example/audit/security/SecurityAuditorAware.java
package com.example.audit.security;

import org.springframework.data.domain.AuditorAware;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;

import java.util.Optional;

public class SecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()
                || "anonymousUser".equals(auth.getPrincipal())) {
            return Optional.of("system");
        }
        return Optional.of(auth.getName());
    }
}
```

```java
// src/main/java/com/example/audit/domain/Auditable.java
package com.example.audit.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.*;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.Instant;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter @Setter
public abstract class Auditable {

    @CreatedBy
    @Column(name = "created_by", nullable = false, updatable = false, length = 100)
    private String createdBy;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedBy
    @Column(name = "last_modified_by", nullable = false, length = 100)
    private String lastModifiedBy;

    @LastModifiedDate
    @Column(name = "last_modified_at", nullable = false)
    private Instant lastModifiedAt;

    @Version
    private Long version;
}
```

```java
// src/main/java/com/example/audit/domain/Document.java
package com.example.audit.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "documents")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Document extends Auditable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Column(nullable = false, length = 50)
    private String ownerUsername;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private DocumentStatus status = DocumentStatus.DRAFT;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
}
```

---

## 6. Spring Data Envers — Full Revision History

Spring Data Envers wraps Hibernate Envers to expose revision queries through standard
Spring Data repositories.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-envers</artifactId>
</dependency>
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
```

```java
// src/main/java/com/example/audit/domain/Document.java  (add @Audited)
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;

@Entity
@Table(name = "documents")
@Audited                               // Envers tracks every INSERT/UPDATE/DELETE
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Document extends Auditable {
    // ... same fields as above ...

    @NotAudited                        // skip large binary fields from revision table
    @Lob
    private byte[] attachment;
}
```

```java
// src/main/java/com/example/audit/repository/DocumentRevisionRepository.java
package com.example.audit.repository;

import com.example.audit.domain.Document;
import org.springframework.data.history.Revision;
import org.springframework.data.history.Revisions;
import org.springframework.data.repository.history.RevisionRepository;
import org.springframework.data.jpa.repository.JpaRepository;

public interface DocumentRepository
        extends JpaRepository<Document, Long>,
                RevisionRepository<Document, Long, Long> {

    java.util.List<Document> findByOwnerUsername(String ownerUsername);
}
```

```java
// src/main/java/com/example/audit/service/RevisionService.java
package com.example.audit.service;

import com.example.audit.domain.Document;
import com.example.audit.dto.RevisionDto;
import com.example.audit.repository.DocumentRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.history.Revision;
import org.springframework.data.history.Revisions;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class RevisionService {

    private final DocumentRepository documentRepository;

    @PreAuthorize("hasAnyRole('ADMIN','AUDITOR')")
    public List<RevisionDto> getRevisionHistory(Long documentId) {
        Revisions<Long, Document> revisions =
                documentRepository.findRevisions(documentId);

        return revisions.getContent().stream()
                .map(rev -> RevisionDto.builder()
                        .revisionNumber(rev.getRequiredRevisionNumber())
                        .revisionType(rev.getMetadata().getRevisionType().name())
                        .modifiedBy(rev.getEntity().getLastModifiedBy())
                        .modifiedAt(rev.getEntity().getLastModifiedAt())
                        .snapshot(rev.getEntity())
                        .build())
                .toList();
    }

    @PreAuthorize("hasAnyRole('ADMIN','AUDITOR')")
    public Document findAtRevision(Long documentId, Long revisionNumber) {
        return documentRepository.findRevision(documentId, revisionNumber)
                .map(Revision::getEntity)
                .orElseThrow(() -> new RuntimeException(
                        "Revision " + revisionNumber + " not found for document " + documentId));
    }
}
```

```java
// src/main/java/com/example/audit/dto/RevisionDto.java
package com.example.audit.dto;

import com.example.audit.domain.Document;
import lombok.Builder;
import lombok.Value;
import java.time.Instant;

@Value
@Builder
public class RevisionDto {
    Long revisionNumber;
    String revisionType;          // ADD, MOD, DEL
    String modifiedBy;
    Instant modifiedAt;
    Document snapshot;
}
```

---

## 7. Application-Level Audit Trail with Database Storage

For compliance you often need a structured, searchable audit log independent of Envers.

```java
// src/main/java/com/example/audit/domain/AuditLog.java
package com.example.audit.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "audit_logs",
       indexes = {
           @Index(name = "idx_audit_entity", columnList = "entity_type, entity_id"),
           @Index(name = "idx_audit_user",   columnList = "username"),
           @Index(name = "idx_audit_time",   columnList = "occurred_at")
       })
@Getter @Builder @NoArgsConstructor @AllArgsConstructor
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "entity_type", nullable = false, length = 100)
    private String entityType;

    @Column(name = "entity_id", nullable = false)
    private Long entityId;

    @Column(nullable = false, length = 50)
    private String action;              // CREATED, UPDATED, DELETED, APPROVED, etc.

    @Column(nullable = false, length = 100)
    private String username;

    @Column(name = "ip_address", length = 45)
    private String ipAddress;

    @Column(name = "old_value", columnDefinition = "TEXT")
    private String oldValue;            // JSON snapshot before change

    @Column(name = "new_value", columnDefinition = "TEXT")
    private String newValue;            // JSON snapshot after change

    @Column(name = "occurred_at", nullable = false)
    private Instant occurredAt;
}
```

```java
// src/main/java/com/example/audit/service/AuditLogService.java
package com.example.audit.service;

import com.example.audit.domain.AuditLog;
import com.example.audit.repository.AuditLogRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import lombok.SneakyThrows;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.time.Instant;

@Service
@RequiredArgsConstructor
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;
    private final ObjectMapper objectMapper;

    @SneakyThrows
    @Transactional(propagation = Propagation.REQUIRES_NEW) // always commit even if caller rolls back
    public void log(String entityType, Long entityId, String action,
                    Object before, Object after) {
        String username = resolveUsername();
        String ip       = resolveIp();

        AuditLog entry = AuditLog.builder()
                .entityType(entityType)
                .entityId(entityId)
                .action(action)
                .username(username)
                .ipAddress(ip)
                .oldValue(before != null ? objectMapper.writeValueAsString(before) : null)
                .newValue(after  != null ? objectMapper.writeValueAsString(after)  : null)
                .occurredAt(Instant.now())
                .build();

        auditLogRepository.save(entry);
    }

    private String resolveUsername() {
        var auth = SecurityContextHolder.getContext().getAuthentication();
        return (auth != null && auth.isAuthenticated()) ? auth.getName() : "anonymous";
    }

    private String resolveIp() {
        try {
            ServletRequestAttributes attrs =
                    (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs == null) return "N/A";
            HttpServletRequest req = attrs.getRequest();
            String forwarded = req.getHeader("X-Forwarded-For");
            return (forwarded != null) ? forwarded.split(",")[0].trim()
                                       : req.getRemoteAddr();
        } catch (Exception e) {
            return "N/A";
        }
    }
}
```

---

## 8. Audit Aspect — Zero-Intrusion Logging

```java
// src/main/java/com/example/audit/aspect/AuditAspect.java
package com.example.audit.aspect;

import com.example.audit.annotation.Audited;
import com.example.audit.service.AuditLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class AuditAspect {

    private final AuditLogService auditLogService;

    @Around("@annotation(com.example.audit.annotation.Audited)")
    public Object audit(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature sig = (MethodSignature) joinPoint.getSignature();
        Method method = sig.getMethod();
        Audited annotation = method.getAnnotation(Audited.class);

        Object[] args = joinPoint.getArgs();
        Object before = args.length > 0 ? args[0] : null;
        Object result;
        try {
            result = joinPoint.proceed();
        } catch (Throwable t) {
            log.warn("Audited method threw exception: {}", t.getMessage());
            throw t;
        }
        try {
            auditLogService.log(
                    annotation.entity(),
                    resolveEntityId(args),
                    annotation.action(),
                    before,
                    result
            );
        } catch (Exception e) {
            log.error("Failed to write audit log", e);
        }
        return result;
    }

    private Long resolveEntityId(Object[] args) {
        for (Object arg : args) {
            if (arg instanceof Long l) return l;
        }
        return -1L;
    }
}
```

```java
// src/main/java/com/example/audit/annotation/Audited.java
package com.example.audit.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Audited {
    String entity();
    String action();
}
```

```java
// Usage on DocumentService methods:
@Audited(entity = "Document", action = "CREATED")
@Transactional
public Document create(DocumentRequest request) { /* ... */ }

@Audited(entity = "Document", action = "UPDATED")
@Transactional
public Document update(Long id, DocumentRequest request) { /* ... */ }
```

---

## 9. Security Event Listeners

Spring Security publishes events for every authentication attempt and authorization decision.
Listen to these for intrusion detection, account locking, and compliance reporting.

```java
// src/main/java/com/example/audit/listener/SecurityEventListener.java
package com.example.audit.listener;

import com.example.audit.service.AuditLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.security.authentication.event.*;
import org.springframework.security.authorization.event.AuthorizationDeniedEvent;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
@Slf4j
public class SecurityEventListener {

    private final AuditLogService auditLogService;
    private final LoginAttemptService loginAttemptService;

    @EventListener
    public void onSuccess(AuthenticationSuccessEvent event) {
        Authentication auth = event.getAuthentication();
        log.info("Successful login: {}", auth.getName());
        loginAttemptService.loginSucceeded(auth.getName());
        auditLogService.log("Authentication", null, "LOGIN_SUCCESS",
                null, auth.getName());
    }

    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        String reason   = event.getException().getMessage();
        log.warn("Failed login for {}: {}", username, reason);
        loginAttemptService.loginFailed(username);
        auditLogService.log("Authentication", null, "LOGIN_FAILURE",
                null, username + ": " + reason);
    }

    @EventListener
    public void onAccessDenied(AuthorizationDeniedEvent event) {
        Authentication auth = event.getAuthentication().get();
        log.warn("Access denied for {} on {}",
                auth.getName(), event.getAuthorizationDecision());
        auditLogService.log("Authorization", null, "ACCESS_DENIED",
                null, auth.getName());
    }

    @EventListener
    public void onLogout(LogoutSuccessEvent event) {
        auditLogService.log("Authentication", null, "LOGOUT",
                null, event.getAuthentication().getName());
    }
}
```

```java
// src/main/java/com/example/audit/listener/LoginAttemptService.java
package com.example.audit.listener;

import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class LoginAttemptService {

    private static final int MAX_ATTEMPTS = 5;
    private final LoadingCache<String, Integer> attemptsCache;

    public LoginAttemptService() {
        attemptsCache = CacheBuilder.newBuilder()
                .expireAfterWrite(1, TimeUnit.HOURS)
                .build(CacheLoader.from(k -> 0));
    }

    public void loginSucceeded(String username) {
        attemptsCache.invalidate(username);
    }

    public void loginFailed(String username) {
        int attempts;
        try {
            attempts = attemptsCache.get(username);
        } catch (ExecutionException e) {
            attempts = 0;
        }
        attemptsCache.put(username, ++attempts);
        if (attempts >= MAX_ATTEMPTS) {
            log.warn("Account {} locked after {} failed attempts", username, attempts);
        }
    }

    public boolean isBlocked(String username) {
        try {
            return attemptsCache.get(username) >= MAX_ATTEMPTS;
        } catch (ExecutionException e) {
            return false;
        }
    }
}
```

---

## 10. GDPR Compliance Patterns

### 10.1 Data Retention

```java
// src/main/java/com/example/audit/gdpr/DataRetentionService.java
package com.example.audit.gdpr;

import com.example.audit.repository.AuditLogRepository;
import com.example.audit.repository.UserProfileRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Service
@RequiredArgsConstructor
@Slf4j
public class DataRetentionService {

    private static final int AUDIT_LOG_RETENTION_DAYS = 730;  // 2 years
    private static final int INACTIVE_USER_DAYS       = 1095; // 3 years

    private final AuditLogRepository  auditLogRepository;
    private final UserProfileRepository userProfileRepository;

    @Scheduled(cron = "0 0 2 * * *")   // 02:00 every night
    @Transactional
    public void purgeExpiredAuditLogs() {
        Instant cutoff = Instant.now().minus(AUDIT_LOG_RETENTION_DAYS, ChronoUnit.DAYS);
        long deleted = auditLogRepository.deleteByOccurredAtBefore(cutoff);
        log.info("Purged {} audit log entries older than {}", deleted, cutoff);
    }

    @Scheduled(cron = "0 30 2 * * SUN")  // 02:30 every Sunday
    @Transactional
    public void anonymiseInactiveUsers() {
        Instant cutoff = Instant.now().minus(INACTIVE_USER_DAYS, ChronoUnit.DAYS);
        int count = userProfileRepository.anonymiseInactiveSince(cutoff);
        log.info("Anonymised {} inactive user profiles", count);
    }
}
```

### 10.2 Right to Erasure (GDPR Article 17)

```java
// src/main/java/com/example/audit/gdpr/ErasureService.java
package com.example.audit.gdpr;

import com.example.audit.domain.ErasureRequest;
import com.example.audit.domain.ErasureStatus;
import com.example.audit.domain.UserProfile;
import com.example.audit.repository.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Service
@RequiredArgsConstructor
@Slf4j
public class ErasureService {

    private final UserProfileRepository    userProfileRepository;
    private final DocumentRepository       documentRepository;
    private final AuditLogRepository       auditLogRepository;
    private final ErasureRequestRepository erasureRequestRepository;

    @PreAuthorize("hasRole('DPO') or #username == authentication.name")
    @Transactional
    public ErasureRequest requestErasure(String username, String reason) {
        // 1. Soft-delete user data
        UserProfile profile = userProfileRepository.findByUsername(username)
                .orElseThrow(() -> new RuntimeException("User not found: " + username));

        profile.setEmail("erased-" + profile.getId() + "@deleted.invalid");
        profile.setFirstName("Erased");
        profile.setLastName("User");
        profile.setPhoneNumber(null);
        profile.setDateOfBirth(null);
        profile.setErased(true);
        profile.setErasedAt(Instant.now());
        userProfileRepository.save(profile);

        // 2. Anonymise documents owned by this user
        documentRepository.anonymiseByOwner(username);

        // 3. Keep audit logs but redact PII within them (obligation to keep security logs)
        auditLogRepository.redactUsername(username, "erased-user-" + profile.getId());

        // 4. Record the erasure request itself for accountability
        ErasureRequest req = ErasureRequest.builder()
                .subjectUsername(username)
                .requestedBy(username)
                .reason(reason)
                .status(ErasureStatus.COMPLETED)
                .completedAt(Instant.now())
                .build();

        log.info("GDPR erasure completed for user {}", username);
        return erasureRequestRepository.save(req);
    }
}
```

---

## 11. Envers Custom Revision Entity

```java
// src/main/java/com/example/audit/domain/CustomRevisionEntity.java
package com.example.audit.domain;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import lombok.Getter;
import lombok.Setter;
import org.hibernate.envers.DefaultRevisionEntity;
import org.hibernate.envers.RevisionEntity;

@Entity
@Table(name = "revinfo")
@RevisionEntity(CustomRevisionListener.class)
@Getter @Setter
public class CustomRevisionEntity extends DefaultRevisionEntity {

    @Column(name = "username", length = 100)
    private String username;

    @Column(name = "ip_address", length = 45)
    private String ipAddress;
}
```

```java
// src/main/java/com/example/audit/domain/CustomRevisionListener.java
package com.example.audit.domain;

import org.hibernate.envers.RevisionListener;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

public class CustomRevisionListener implements RevisionListener {

    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevisionEntity rev = (CustomRevisionEntity) revisionEntity;

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        rev.setUsername(auth != null ? auth.getName() : "system");

        try {
            ServletRequestAttributes attrs =
                    (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs != null) {
                String ip = attrs.getRequest().getHeader("X-Forwarded-For");
                rev.setIpAddress(ip != null ? ip.split(",")[0].trim()
                                            : attrs.getRequest().getRemoteAddr());
            }
        } catch (Exception ignored) {}
    }
}
```

---

## 12. Audit Controller

```java
// src/main/java/com/example/audit/controller/AuditController.java
package com.example.audit.controller;

import com.example.audit.domain.AuditLog;
import com.example.audit.dto.RevisionDto;
import com.example.audit.repository.AuditLogRepository;
import com.example.audit.service.RevisionService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.List;

@RestController
@RequestMapping("/api/v1/audit")
@RequiredArgsConstructor
@PreAuthorize("hasAnyRole('ADMIN','AUDITOR')")
public class AuditController {

    private final AuditLogRepository auditLogRepository;
    private final RevisionService    revisionService;

    @GetMapping("/logs")
    public Page<AuditLog> getLogs(
            @RequestParam(required = false) String entityType,
            @RequestParam(required = false) String username,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
                Instant from,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME)
                Instant to,
            Pageable pageable) {
        return auditLogRepository.findByFilters(entityType, username, from, to, pageable);
    }

    @GetMapping("/documents/{id}/history")
    public List<RevisionDto> getDocumentHistory(@PathVariable Long id) {
        return revisionService.getRevisionHistory(id);
    }

    @GetMapping("/documents/{id}/history/{revision}")
    public Object getDocumentAtRevision(@PathVariable Long id,
                                        @PathVariable Long revision) {
        return revisionService.findAtRevision(id, revision);
    }
}
```

---

## 13. Integration Tests

```java
// src/test/java/com/example/audit/service/DocumentServiceSecurityTest.java
package com.example.audit.service;

import com.example.audit.domain.Document;
import com.example.audit.domain.DocumentStatus;
import com.example.audit.repository.DocumentRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.*;

@SpringBootTest
@Transactional
class DocumentServiceSecurityTest {

    @Autowired DocumentService   documentService;
    @Autowired DocumentRepository documentRepository;

    private Document savedDocument;

    @BeforeEach
    void setup() {
        savedDocument = documentRepository.save(Document.builder()
                .title("Test Doc")
                .content("Content")
                .ownerUsername("alice")
                .status(DocumentStatus.DRAFT)
                .build());
    }

    @Test
    @WithMockUser(username = "alice")
    void ownerCanReadTheirDocument() {
        Document doc = documentService.findById(savedDocument.getId());
        assertThat(doc.getTitle()).isEqualTo("Test Doc");
    }

    @Test
    @WithMockUser(username = "bob")
    void nonOwnerCannotReadDocument() {
        assertThatThrownBy(() -> documentService.findById(savedDocument.getId()))
                .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void adminCanReadAnyDocument() {
        Document doc = documentService.findById(savedDocument.getId());
        assertThat(doc).isNotNull();
    }

    @Test
    @WithMockUser(username = "alice")
    void ownerCanDeleteDraftDocument() {
        assertThatNoException()
                .isThrownBy(() -> documentService.delete(savedDocument.getId()));
    }

    @Test
    @WithMockUser(username = "bob")
    void nonOwnerCannotDeleteDocument() {
        assertThatThrownBy(() -> documentService.delete(savedDocument.getId()))
                .isInstanceOf(AccessDeniedException.class);
    }
}
```

---

## 14. Flyway Migration for Audit Schema

```sql
-- src/main/resources/db/migration/V3__audit_tables.sql

CREATE TABLE audit_logs (
    id           BIGSERIAL    PRIMARY KEY,
    entity_type  VARCHAR(100) NOT NULL,
    entity_id    BIGINT,
    action       VARCHAR(50)  NOT NULL,
    username     VARCHAR(100) NOT NULL,
    ip_address   VARCHAR(45),
    old_value    TEXT,
    new_value    TEXT,
    occurred_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_entity ON audit_logs (entity_type, entity_id);
CREATE INDEX idx_audit_user   ON audit_logs (username);
CREATE INDEX idx_audit_time   ON audit_logs (occurred_at DESC);

CREATE TABLE acl_entries (
    id          BIGSERIAL    PRIMARY KEY,
    principal   VARCHAR(200) NOT NULL,
    object_type VARCHAR(100) NOT NULL,
    object_id   BIGINT       NOT NULL,
    permission  VARCHAR(50)  NOT NULL,
    UNIQUE (principal, object_type, object_id, permission)
);

CREATE INDEX idx_acl_lookup ON acl_entries (object_type, object_id, permission);

CREATE TABLE erasure_requests (
    id                BIGSERIAL    PRIMARY KEY,
    subject_username  VARCHAR(100) NOT NULL,
    requested_by      VARCHAR(100) NOT NULL,
    reason            TEXT,
    status            VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    requested_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    completed_at      TIMESTAMPTZ
);
```

---

## Summary

| Feature | Mechanism |
|---|---|
| Method-level rules | `@PreAuthorize` / `@PostAuthorize` with SpEL |
| Fine-grained object rules | `hasPermission()` + `PermissionEvaluator` |
| Role guards | `@Secured` / `@RolesAllowed` |
| Who created / modified | `@CreatedBy` / `@LastModifiedBy` + `AuditorAware` |
| Full field-level history | Hibernate Envers + `@Audited` |
| Structured audit trail | `AuditLog` entity + `AuditLogService` |
| Zero-intrusion logging | `@Audited` annotation + AOP aspect |
| Intrusion detection | `AuthenticationSuccessEvent` / `AbstractAuthenticationFailureEvent` |
| GDPR erasure | `ErasureService` — anonymise, then record the erasure itself |
| GDPR retention | Scheduled job purging logs past the retention window |
