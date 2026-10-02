# Part 094: Hexagonal Architecture (Ports & Adapters)

## Introduction

Hexagonal Architecture, coined by Alistair Cockburn, organizes code so that the business domain is at the center and all external concerns (databases, REST APIs, message brokers) attach via explicit interfaces called **ports**. The implementations of those interfaces are called **adapters**.

**Core principle**: The domain core has zero dependencies on frameworks, databases, or infrastructure. It only knows about itself.

```
         [ REST Controller ] ← Input Adapter
              ↓
         [ Use Case Port ]  ← Input Port (interface)
              ↓
    [ Domain Core / Application ]
              ↓
         [ Repository Port ] ← Output Port (interface)
              ↓
         [ JPA Adapter ]    ← Output Adapter
```

---

## Package Structure

```
src/main/java/com/example/loan/
├── domain/                          ← PURE DOMAIN (no Spring, no JPA)
│   ├── model/
│   │   ├── Loan.java
│   │   ├── LoanApplication.java
│   │   ├── Applicant.java
│   │   ├── Money.java
│   │   └── LoanStatus.java
│   ├── service/                     ← Domain services
│   │   ├── RiskCalculator.java
│   │   └── LoanEligibilityService.java
│   └── exception/
│       ├── LoanNotFoundException.java
│       └── IneligibleApplicantException.java
│
├── application/                     ← APPLICATION LAYER (use cases)
│   ├── port/
│   │   ├── input/                   ← Input Ports (what the domain exposes)
│   │   │   ├── ApplyForLoanUseCase.java
│   │   │   ├── ApproveLoanUseCase.java
│   │   │   └── GetLoanStatusUseCase.java
│   │   └── output/                  ← Output Ports (what the domain needs)
│   │       ├── LoanRepository.java
│   │       ├── ApplicantRepository.java
│   │       ├── CreditScoreProvider.java
│   │       ├── NotificationPort.java
│   │       └── AuditPort.java
│   └── service/                     ← Use case implementations
│       ├── ApplyForLoanService.java
│       ├── ApproveLoanService.java
│       └── GetLoanStatusService.java
│
└── adapter/                         ← ADAPTERS (Spring, JPA, etc. live here)
    ├── input/
    │   ├── rest/                    ← REST Input Adapter
    │   │   ├── LoanController.java
    │   │   ├── dto/
    │   │   └── mapper/
    │   ├── cli/                     ← CLI Input Adapter
    │   └── messaging/               ← Kafka Consumer Input Adapter
    │       └── LoanApplicationConsumer.java
    └── output/
        ├── persistence/             ← JPA Output Adapter
        │   ├── LoanJpaRepository.java
        │   ├── LoanPersistenceAdapter.java
        │   ├── entity/
        │   └── mapper/
        ├── notification/            ← Email/SMS Output Adapter
        │   └── EmailNotificationAdapter.java
        ├── creditbureau/            ← External API Output Adapter
        │   └── EquifaxCreditScoreAdapter.java
        └── audit/
            └── DatabaseAuditAdapter.java
```

---

## Domain Model (Pure Java, Zero Framework Dependencies)

```java
// src/main/java/com/example/loan/domain/model/Money.java
package com.example.loan.domain.model;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;

/**
 * Value object representing monetary amount.
 * Immutable by design.
 */
public final class Money {

    private final BigDecimal amount;
    private final Currency currency;

    private Money(BigDecimal amount, Currency currency) {
        if (amount == null) throw new IllegalArgumentException("Amount cannot be null");
        if (currency == null) throw new IllegalArgumentException("Currency cannot be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative: " + amount);
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Money of(BigDecimal amount, String currencyCode) {
        return new Money(amount, Currency.getInstance(currencyCode));
    }

    public static Money of(double amount, String currencyCode) {
        return of(BigDecimal.valueOf(amount), currencyCode);
    }

    public static Money usd(double amount) {
        return of(amount, "USD");
    }

    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.subtract(other.amount), this.currency);
    }

    public Money multiply(BigDecimal factor) {
        return new Money(this.amount.multiply(factor), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        requireSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isLessThan(Money other) {
        requireSameCurrency(other);
        return this.amount.compareTo(other.amount) < 0;
    }

    private void requireSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Cannot operate on different currencies: " + this.currency + " vs " + other.currency);
        }
    }

    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money money)) return false;
        return Objects.equals(amount, money.amount) && Objects.equals(currency, money.currency);
    }

    @Override
    public int hashCode() { return Objects.hash(amount, currency); }

    @Override
    public String toString() { return currency.getSymbol() + amount; }
}
```

```java
// src/main/java/com/example/loan/domain/model/LoanApplication.java
package com.example.loan.domain.model;

import com.example.loan.domain.exception.IneligibleApplicantException;
import com.example.loan.domain.exception.InvalidLoanStateException;

import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Root aggregate of the loan application.
 * Contains all business rules as methods.
 */
public class LoanApplication {

    private final String id;
    private final String applicantId;
    private final Money requestedAmount;
    private final int termMonths;
    private LoanStatus status;
    private String rejectionReason;
    private String approvedBy;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private int creditScore;
    private double debtToIncomeRatio;

    // Static factory for new applications
    public static LoanApplication create(String applicantId, Money requestedAmount,
                                          int termMonths) {
        if (requestedAmount.isLessThan(Money.usd(1000))) {
            throw new IllegalArgumentException("Minimum loan amount is $1,000");
        }
        if (termMonths < 12 || termMonths > 360) {
            throw new IllegalArgumentException("Loan term must be between 12 and 360 months");
        }

        LoanApplication app = new LoanApplication();
        app.id = UUID.randomUUID().toString();
        app.applicantId = applicantId;
        app.requestedAmount = requestedAmount;
        app.termMonths = termMonths;
        app.status = LoanStatus.DRAFT;
        app.createdAt = LocalDateTime.now();
        app.updatedAt = LocalDateTime.now();
        return app;
    }

    // Reconstitution from persistence (no UUID generation)
    public static LoanApplication reconstitute(String id, String applicantId,
                                                Money requestedAmount, int termMonths,
                                                LoanStatus status, int creditScore,
                                                double debtToIncomeRatio, LocalDateTime createdAt) {
        LoanApplication app = new LoanApplication();
        app.id = id;
        app.applicantId = applicantId;
        app.requestedAmount = requestedAmount;
        app.termMonths = termMonths;
        app.status = status;
        app.creditScore = creditScore;
        app.debtToIncomeRatio = debtToIncomeRatio;
        app.createdAt = createdAt;
        app.updatedAt = LocalDateTime.now();
        return app;
    }

    private LoanApplication() {}

    // Business rules as domain methods
    public void submit(int creditScore, double debtToIncomeRatio) {
        requireStatus(LoanStatus.DRAFT);
        this.creditScore = creditScore;
        this.debtToIncomeRatio = debtToIncomeRatio;
        this.status = LoanStatus.SUBMITTED;
        this.updatedAt = LocalDateTime.now();
    }

    public void underwrite() {
        requireStatus(LoanStatus.SUBMITTED);
        this.status = LoanStatus.UNDER_REVIEW;
        this.updatedAt = LocalDateTime.now();
    }

    public void approve(String approverUserId) {
        requireStatus(LoanStatus.UNDER_REVIEW);

        if (creditScore < 580) {
            throw new IneligibleApplicantException(
                "Credit score " + creditScore + " is below minimum threshold of 580");
        }
        if (debtToIncomeRatio > 0.43) {
            throw new IneligibleApplicantException(
                "Debt-to-income ratio " + debtToIncomeRatio + " exceeds maximum of 0.43");
        }

        this.status = LoanStatus.APPROVED;
        this.approvedBy = approverUserId;
        this.updatedAt = LocalDateTime.now();
    }

    public void reject(String reason) {
        if (status != LoanStatus.SUBMITTED && status != LoanStatus.UNDER_REVIEW) {
            throw new InvalidLoanStateException("Cannot reject a loan in status: " + status);
        }
        this.status = LoanStatus.REJECTED;
        this.rejectionReason = reason;
        this.updatedAt = LocalDateTime.now();
    }

    public void disburse() {
        requireStatus(LoanStatus.APPROVED);
        this.status = LoanStatus.DISBURSED;
        this.updatedAt = LocalDateTime.now();
    }

    public boolean isEligibleForFastTrack() {
        return creditScore >= 750 && debtToIncomeRatio <= 0.20
            && requestedAmount.isLessThan(Money.usd(50_000));
    }

    private void requireStatus(LoanStatus expected) {
        if (this.status != expected) {
            throw new InvalidLoanStateException(
                "Expected status " + expected + " but was " + this.status);
        }
    }

    // Getters (no setters — immutable via business methods)
    public String getId() { return id; }
    public String getApplicantId() { return applicantId; }
    public Money getRequestedAmount() { return requestedAmount; }
    public int getTermMonths() { return termMonths; }
    public LoanStatus getStatus() { return status; }
    public String getRejectionReason() { return rejectionReason; }
    public String getApprovedBy() { return approvedBy; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public int getCreditScore() { return creditScore; }
    public double getDebtToIncomeRatio() { return debtToIncomeRatio; }
}
```

```java
// src/main/java/com/example/loan/domain/model/LoanStatus.java
package com.example.loan.domain.model;

public enum LoanStatus {
    DRAFT,
    SUBMITTED,
    UNDER_REVIEW,
    APPROVED,
    REJECTED,
    DISBURSED,
    CLOSED
}
```

---

## Input Ports (Use Case Interfaces)

```java
// src/main/java/com/example/loan/application/port/input/ApplyForLoanUseCase.java
package com.example.loan.application.port.input;

import com.example.loan.domain.model.LoanApplication;
import java.math.BigDecimal;

public interface ApplyForLoanUseCase {

    record ApplyForLoanCommand(
        String applicantId,
        BigDecimal amount,
        String currency,
        int termMonths,
        String purpose
    ) {
        public ApplyForLoanCommand {
            if (applicantId == null || applicantId.isBlank()) {
                throw new IllegalArgumentException("applicantId is required");
            }
            if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
                throw new IllegalArgumentException("amount must be positive");
            }
        }
    }

    LoanApplication apply(ApplyForLoanCommand command);
}
```

```java
// src/main/java/com/example/loan/application/port/input/ApproveLoanUseCase.java
package com.example.loan.application.port.input;

import com.example.loan.domain.model.LoanApplication;

public interface ApproveLoanUseCase {

    record ApproveCommand(String loanId, String approverUserId) {}
    record RejectCommand(String loanId, String reason, String reviewerUserId) {}

    LoanApplication approve(ApproveCommand command);
    LoanApplication reject(RejectCommand command);
}
```

```java
// src/main/java/com/example/loan/application/port/input/GetLoanStatusUseCase.java
package com.example.loan.application.port.input;

import com.example.loan.domain.model.LoanApplication;
import java.util.List;

public interface GetLoanStatusUseCase {
    LoanApplication getLoan(String loanId);
    List<LoanApplication> getLoansForApplicant(String applicantId);
}
```

---

## Output Ports

```java
// src/main/java/com/example/loan/application/port/output/LoanRepository.java
package com.example.loan.application.port.output;

import com.example.loan.domain.model.LoanApplication;
import com.example.loan.domain.model.LoanStatus;

import java.util.List;
import java.util.Optional;

/**
 * Output port for loan persistence.
 * The domain defines this interface; adapters implement it.
 */
public interface LoanRepository {
    LoanApplication save(LoanApplication application);
    Optional<LoanApplication> findById(String id);
    List<LoanApplication> findByApplicantId(String applicantId);
    List<LoanApplication> findByStatus(LoanStatus status);
    boolean existsById(String id);
}
```

```java
// src/main/java/com/example/loan/application/port/output/CreditScoreProvider.java
package com.example.loan.application.port.output;

public interface CreditScoreProvider {

    record CreditReport(
        String applicantId,
        int creditScore,
        double debtToIncomeRatio,
        int accountCount,
        int delinquentAccounts
    ) {}

    CreditReport getCreditReport(String applicantId);
}
```

```java
// src/main/java/com/example/loan/application/port/output/NotificationPort.java
package com.example.loan.application.port.output;

import com.example.loan.domain.model.LoanApplication;

public interface NotificationPort {
    void notifyApplicationReceived(LoanApplication application);
    void notifyApproval(LoanApplication application);
    void notifyRejection(LoanApplication application, String reason);
}
```

```java
// src/main/java/com/example/loan/application/port/output/AuditPort.java
package com.example.loan.application.port.output;

import com.example.loan.domain.model.LoanStatus;

public interface AuditPort {
    void recordStatusChange(String loanId, LoanStatus from, LoanStatus to, String performedBy);
    void recordDataAccess(String loanId, String accessedBy, String operation);
}
```

---

## Application Services (Use Case Implementations)

```java
// src/main/java/com/example/loan/application/service/ApplyForLoanService.java
package com.example.loan.application.service;

import com.example.loan.application.port.input.ApplyForLoanUseCase;
import com.example.loan.application.port.output.*;
import com.example.loan.domain.model.LoanApplication;
import com.example.loan.domain.model.LoanStatus;
import com.example.loan.domain.model.Money;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class ApplyForLoanService implements ApplyForLoanUseCase {

    private final LoanRepository loanRepository;
    private final ApplicantRepository applicantRepository;
    private final CreditScoreProvider creditScoreProvider;
    private final NotificationPort notificationPort;
    private final AuditPort auditPort;

    @Override
    public LoanApplication apply(ApplyForLoanCommand command) {
        log.info("Processing loan application for applicant: {}", command.applicantId());

        // 1. Validate applicant exists
        if (!applicantRepository.existsById(command.applicantId())) {
            throw new IllegalArgumentException("Applicant not found: " + command.applicantId());
        }

        // 2. Create domain object (business rules in domain)
        Money amount = Money.of(command.amount(), command.currency());
        LoanApplication application = LoanApplication.create(
            command.applicantId(), amount, command.termMonths()
        );

        // 3. Fetch credit report via output port
        CreditScoreProvider.CreditReport creditReport =
            creditScoreProvider.getCreditReport(command.applicantId());

        // 4. Apply domain method (validates business rules)
        application.submit(creditReport.creditScore(), creditReport.debtToIncomeRatio());

        // 5. Persist via output port
        LoanApplication saved = loanRepository.save(application);

        // 6. Side effects (notifications, audit)
        notificationPort.notifyApplicationReceived(saved);
        auditPort.recordStatusChange(saved.getId(), LoanStatus.DRAFT, LoanStatus.SUBMITTED,
            "SYSTEM");

        // 7. Fast-track if eligible
        if (saved.isEligibleForFastTrack()) {
            log.info("Loan {} eligible for fast-track approval", saved.getId());
            saved.underwrite();
            saved.approve("AUTO_UNDERWRITER");
            loanRepository.save(saved);
            notificationPort.notifyApproval(saved);
        }

        return saved;
    }
}
```

```java
// src/main/java/com/example/loan/application/service/ApproveLoanService.java
package com.example.loan.application.service;

import com.example.loan.application.port.input.ApproveLoanUseCase;
import com.example.loan.application.port.output.AuditPort;
import com.example.loan.application.port.output.LoanRepository;
import com.example.loan.application.port.output.NotificationPort;
import com.example.loan.domain.exception.LoanNotFoundException;
import com.example.loan.domain.model.LoanApplication;
import com.example.loan.domain.model.LoanStatus;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class ApproveLoanService implements ApproveLoanUseCase {

    private final LoanRepository loanRepository;
    private final NotificationPort notificationPort;
    private final AuditPort auditPort;

    @Override
    public LoanApplication approve(ApproveCommand command) {
        LoanApplication application = findLoanOrThrow(command.loanId());
        LoanStatus previousStatus = application.getStatus();

        application.approve(command.approverUserId());

        LoanApplication saved = loanRepository.save(application);

        notificationPort.notifyApproval(saved);
        auditPort.recordStatusChange(saved.getId(), previousStatus,
            LoanStatus.APPROVED, command.approverUserId());

        log.info("Loan {} approved by {}", command.loanId(), command.approverUserId());
        return saved;
    }

    @Override
    public LoanApplication reject(RejectCommand command) {
        LoanApplication application = findLoanOrThrow(command.loanId());
        LoanStatus previousStatus = application.getStatus();

        application.reject(command.reason());

        LoanApplication saved = loanRepository.save(application);

        notificationPort.notifyRejection(saved, command.reason());
        auditPort.recordStatusChange(saved.getId(), previousStatus,
            LoanStatus.REJECTED, command.reviewerUserId());

        log.info("Loan {} rejected. Reason: {}", command.loanId(), command.reason());
        return saved;
    }

    private LoanApplication findLoanOrThrow(String loanId) {
        return loanRepository.findById(loanId)
            .orElseThrow(() -> new LoanNotFoundException("Loan not found: " + loanId));
    }
}
```

---

## Input Adapters

### REST Input Adapter

```java
// src/main/java/com/example/loan/adapter/input/rest/LoanController.java
package com.example.loan.adapter.input.rest;

import com.example.loan.adapter.input.rest.dto.*;
import com.example.loan.adapter.input.rest.mapper.LoanRestMapper;
import com.example.loan.application.port.input.ApplyForLoanUseCase;
import com.example.loan.application.port.input.ApproveLoanUseCase;
import com.example.loan.application.port.input.GetLoanStatusUseCase;
import com.example.loan.domain.model.LoanApplication;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@Slf4j
@RestController
@RequestMapping("/api/v1/loans")
@RequiredArgsConstructor
@Tag(name = "Loans", description = "Loan application management")
public class LoanController {

    private final ApplyForLoanUseCase applyForLoanUseCase;
    private final ApproveLoanUseCase approveLoanUseCase;
    private final GetLoanStatusUseCase getLoanStatusUseCase;
    private final LoanRestMapper mapper;

    @PostMapping
    @Operation(summary = "Submit a new loan application")
    public ResponseEntity<LoanResponse> applyForLoan(
            @Valid @RequestBody LoanApplicationRequest request,
            @AuthenticationPrincipal Jwt jwt) {

        String applicantId = jwt.getSubject();

        ApplyForLoanUseCase.ApplyForLoanCommand command =
            new ApplyForLoanUseCase.ApplyForLoanCommand(
                applicantId,
                request.amount(),
                request.currency(),
                request.termMonths(),
                request.purpose()
            );

        LoanApplication application = applyForLoanUseCase.apply(command);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(mapper.toResponse(application));
    }

    @GetMapping("/{loanId}")
    @Operation(summary = "Get loan status")
    public ResponseEntity<LoanResponse> getLoan(@PathVariable String loanId) {
        LoanApplication application = getLoanStatusUseCase.getLoan(loanId);
        return ResponseEntity.ok(mapper.toResponse(application));
    }

    @GetMapping("/applicant/{applicantId}")
    @Operation(summary = "Get all loans for an applicant")
    @PreAuthorize("hasRole('ADMIN') or #applicantId == authentication.name")
    public ResponseEntity<List<LoanResponse>> getLoansForApplicant(
            @PathVariable String applicantId) {
        List<LoanApplication> loans = getLoanStatusUseCase.getLoansForApplicant(applicantId);
        return ResponseEntity.ok(mapper.toResponseList(loans));
    }

    @PostMapping("/{loanId}/approve")
    @PreAuthorize("hasRole('UNDERWRITER') or hasRole('ADMIN')")
    @Operation(summary = "Approve a loan application")
    public ResponseEntity<LoanResponse> approveLoan(
            @PathVariable String loanId,
            @AuthenticationPrincipal Jwt jwt) {

        ApproveLoanUseCase.ApproveCommand command =
            new ApproveLoanUseCase.ApproveCommand(loanId, jwt.getSubject());

        LoanApplication approved = approveLoanUseCase.approve(command);
        return ResponseEntity.ok(mapper.toResponse(approved));
    }

    @PostMapping("/{loanId}/reject")
    @PreAuthorize("hasRole('UNDERWRITER') or hasRole('ADMIN')")
    @Operation(summary = "Reject a loan application")
    public ResponseEntity<LoanResponse> rejectLoan(
            @PathVariable String loanId,
            @Valid @RequestBody RejectLoanRequest request,
            @AuthenticationPrincipal Jwt jwt) {

        ApproveLoanUseCase.RejectCommand command =
            new ApproveLoanUseCase.RejectCommand(loanId, request.reason(), jwt.getSubject());

        LoanApplication rejected = approveLoanUseCase.reject(command);
        return ResponseEntity.ok(mapper.toResponse(rejected));
    }
}
```

### REST DTOs

```java
// src/main/java/com/example/loan/adapter/input/rest/dto/LoanApplicationRequest.java
package com.example.loan.adapter.input.rest.dto;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record LoanApplicationRequest(
    @NotNull @DecimalMin("1000.00") BigDecimal amount,
    @NotBlank @Pattern(regexp = "[A-Z]{3}") String currency,
    @Min(12) @Max(360) int termMonths,
    @NotBlank @Size(max = 500) String purpose
) {}
```

```java
// src/main/java/com/example/loan/adapter/input/rest/dto/LoanResponse.java
package com.example.loan.adapter.input.rest.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public record LoanResponse(
    String id,
    String applicantId,
    BigDecimal amount,
    String currency,
    int termMonths,
    String status,
    String rejectionReason,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}
```

### REST Mapper

```java
// src/main/java/com/example/loan/adapter/input/rest/mapper/LoanRestMapper.java
package com.example.loan.adapter.input.rest.mapper;

import com.example.loan.adapter.input.rest.dto.LoanResponse;
import com.example.loan.domain.model.LoanApplication;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.stream.Collectors;

@Component
public class LoanRestMapper {

    public LoanResponse toResponse(LoanApplication application) {
        return new LoanResponse(
            application.getId(),
            application.getApplicantId(),
            application.getRequestedAmount().getAmount(),
            application.getRequestedAmount().getCurrency().getCurrencyCode(),
            application.getTermMonths(),
            application.getStatus().name(),
            application.getRejectionReason(),
            application.getCreatedAt(),
            application.getUpdatedAt()
        );
    }

    public List<LoanResponse> toResponseList(List<LoanApplication> applications) {
        return applications.stream().map(this::toResponse).collect(Collectors.toList());
    }
}
```

---

## Output Adapters

### JPA Persistence Adapter

```java
// src/main/java/com/example/loan/adapter/output/persistence/entity/LoanEntity.java
package com.example.loan.adapter.output.persistence.entity;

import com.example.loan.domain.model.LoanStatus;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "loans")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class LoanEntity {

    @Id
    private String id;

    @Column(nullable = false)
    private String applicantId;

    @Column(nullable = false, precision = 15, scale = 2)
    private BigDecimal amount;

    @Column(nullable = false, length = 3)
    private String currency;

    @Column(nullable = false)
    private int termMonths;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private LoanStatus status;

    private int creditScore;
    private double debtToIncomeRatio;
    private String rejectionReason;
    private String approvedBy;
    private String purpose;

    @Column(nullable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;
}
```

```java
// src/main/java/com/example/loan/adapter/output/persistence/LoanJpaRepository.java
package com.example.loan.adapter.output.persistence;

import com.example.loan.adapter.output.persistence.entity.LoanEntity;
import com.example.loan.domain.model.LoanStatus;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

interface LoanJpaRepository extends JpaRepository<LoanEntity, String> {
    List<LoanEntity> findByApplicantId(String applicantId);
    List<LoanEntity> findByStatus(LoanStatus status);
}
```

```java
// src/main/java/com/example/loan/adapter/output/persistence/LoanPersistenceAdapter.java
package com.example.loan.adapter.output.persistence;

import com.example.loan.adapter.output.persistence.entity.LoanEntity;
import com.example.loan.application.port.output.LoanRepository;
import com.example.loan.domain.model.LoanApplication;
import com.example.loan.domain.model.LoanStatus;
import com.example.loan.domain.model.Money;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

@Component
@RequiredArgsConstructor
class LoanPersistenceAdapter implements LoanRepository {

    private final LoanJpaRepository jpaRepository;

    @Override
    public LoanApplication save(LoanApplication application) {
        LoanEntity entity = toEntity(application);
        LoanEntity saved = jpaRepository.save(entity);
        return toDomain(saved);
    }

    @Override
    public Optional<LoanApplication> findById(String id) {
        return jpaRepository.findById(id).map(this::toDomain);
    }

    @Override
    public List<LoanApplication> findByApplicantId(String applicantId) {
        return jpaRepository.findByApplicantId(applicantId)
            .stream().map(this::toDomain).collect(Collectors.toList());
    }

    @Override
    public List<LoanApplication> findByStatus(LoanStatus status) {
        return jpaRepository.findByStatus(status)
            .stream().map(this::toDomain).collect(Collectors.toList());
    }

    @Override
    public boolean existsById(String id) {
        return jpaRepository.existsById(id);
    }

    private LoanEntity toEntity(LoanApplication app) {
        return LoanEntity.builder()
            .id(app.getId())
            .applicantId(app.getApplicantId())
            .amount(app.getRequestedAmount().getAmount())
            .currency(app.getRequestedAmount().getCurrency().getCurrencyCode())
            .termMonths(app.getTermMonths())
            .status(app.getStatus())
            .creditScore(app.getCreditScore())
            .debtToIncomeRatio(app.getDebtToIncomeRatio())
            .rejectionReason(app.getRejectionReason())
            .approvedBy(app.getApprovedBy())
            .createdAt(app.getCreatedAt())
            .updatedAt(app.getUpdatedAt())
            .build();
    }

    private LoanApplication toDomain(LoanEntity entity) {
        return LoanApplication.reconstitute(
            entity.getId(),
            entity.getApplicantId(),
            Money.of(entity.getAmount(), entity.getCurrency()),
            entity.getTermMonths(),
            entity.getStatus(),
            entity.getCreditScore(),
            entity.getDebtToIncomeRatio(),
            entity.getCreatedAt()
        );
    }
}
```

### External Credit Bureau Adapter

```java
// src/main/java/com/example/loan/adapter/output/creditbureau/EquifaxCreditScoreAdapter.java
package com.example.loan.adapter.output.creditbureau;

import com.example.loan.application.port.output.CreditScoreProvider;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Slf4j
@Component
@RequiredArgsConstructor
class EquifaxCreditScoreAdapter implements CreditScoreProvider {

    private final RestClient restClient;

    @Value("${credit-bureau.api-key}")
    private String apiKey;

    @Override
    @Cacheable(value = "creditReports", key = "#applicantId", unless = "#result == null")
    public CreditReport getCreditReport(String applicantId) {
        log.info("Fetching credit report for applicant: {}", applicantId);

        try {
            EquifaxResponse response = restClient.get()
                .uri("/v2/credit-reports/{id}", applicantId)
                .header("X-Api-Key", apiKey)
                .retrieve()
                .body(EquifaxResponse.class);

            return new CreditReport(
                applicantId,
                response.fico(),
                response.dti(),
                response.totalAccounts(),
                response.delinquencies()
            );

        } catch (Exception e) {
            log.error("Failed to fetch credit report for {}: {}", applicantId, e.getMessage());
            // In production: circuit breaker, fallback, etc.
            throw new CreditBureauException("Could not retrieve credit report", e);
        }
    }

    record EquifaxResponse(int fico, double dti, int totalAccounts, int delinquencies) {}
}
```

---

## Domain Unit Tests

```java
// src/test/java/com/example/loan/domain/LoanApplicationTest.java
package com.example.loan.domain;

import com.example.loan.domain.exception.IneligibleApplicantException;
import com.example.loan.domain.exception.InvalidLoanStateException;
import com.example.loan.domain.model.LoanApplication;
import com.example.loan.domain.model.LoanStatus;
import com.example.loan.domain.model.Money;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.assertj.core.api.Assertions.*;

class LoanApplicationTest {

    @Test
    void createLoanWithValidDataSucceeds() {
        LoanApplication app = LoanApplication.create(
            "applicant-1", Money.usd(10_000), 60
        );

        assertThat(app.getStatus()).isEqualTo(LoanStatus.DRAFT);
        assertThat(app.getId()).isNotNull();
        assertThat(app.getApplicantId()).isEqualTo("applicant-1");
    }

    @Test
    void createLoanBelowMinimumThrowsException() {
        assertThatThrownBy(() ->
            LoanApplication.create("app-1", Money.usd(500), 12)
        ).isInstanceOf(IllegalArgumentException.class)
         .hasMessageContaining("Minimum loan amount");
    }

    @ParameterizedTest
    @ValueSource(ints = {6, 11, 361, 400})
    void createLoanWithInvalidTermThrowsException(int invalidTerm) {
        assertThatThrownBy(() ->
            LoanApplication.create("app-1", Money.usd(10_000), invalidTerm)
        ).isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void submitTransitionsFromDraftToSubmitted() {
        LoanApplication app = LoanApplication.create("app-1", Money.usd(5_000), 36);
        app.submit(700, 0.30);

        assertThat(app.getStatus()).isEqualTo(LoanStatus.SUBMITTED);
        assertThat(app.getCreditScore()).isEqualTo(700);
    }

    @Test
    void submitFromWrongStateThrowsException() {
        LoanApplication app = LoanApplication.create("app-1", Money.usd(5_000), 36);
        app.submit(700, 0.30);

        assertThatThrownBy(() -> app.submit(720, 0.25))
            .isInstanceOf(InvalidLoanStateException.class);
    }

    @Test
    void approveLoanWithGoodCreditSucceeds() {
        LoanApplication app = createSubmittedLoan(750, 0.20);
        app.underwrite();
        app.approve("underwriter-1");

        assertThat(app.getStatus()).isEqualTo(LoanStatus.APPROVED);
        assertThat(app.getApprovedBy()).isEqualTo("underwriter-1");
    }

    @Test
    void approveLoanWithLowCreditScoreThrowsException() {
        LoanApplication app = createSubmittedLoan(550, 0.20);
        app.underwrite();

        assertThatThrownBy(() -> app.approve("underwriter-1"))
            .isInstanceOf(IneligibleApplicantException.class)
            .hasMessageContaining("Credit score");
    }

    @Test
    void approveLoanWithHighDTIThrowsException() {
        LoanApplication app = createSubmittedLoan(750, 0.50);
        app.underwrite();

        assertThatThrownBy(() -> app.approve("underwriter-1"))
            .isInstanceOf(IneligibleApplicantException.class)
            .hasMessageContaining("Debt-to-income");
    }

    @Test
    void fastTrackEligibilityCheck() {
        // Eligible: high score, low DTI, small amount
        LoanApplication eligible = createSubmittedLoan(780, 0.15);
        assertThat(eligible.isEligibleForFastTrack()).isTrue();

        // Not eligible: high amount
        LoanApplication tooLarge = LoanApplication.create("app-1", Money.usd(100_000), 120);
        tooLarge.submit(780, 0.15);
        assertThat(tooLarge.isEligibleForFastTrack()).isFalse();
    }

    private LoanApplication createSubmittedLoan(int creditScore, double dti) {
        LoanApplication app = LoanApplication.create("app-1", Money.usd(20_000), 60);
        app.submit(creditScore, dti);
        return app;
    }
}
```

### Application Service Test (Ports are Mocked)

```java
// src/test/java/com/example/loan/application/ApplyForLoanServiceTest.java
package com.example.loan.application;

import com.example.loan.application.port.input.ApplyForLoanUseCase;
import com.example.loan.application.port.output.*;
import com.example.loan.application.service.ApplyForLoanService;
import com.example.loan.domain.model.LoanApplication;
import com.example.loan.domain.model.LoanStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ApplyForLoanServiceTest {

    @Mock LoanRepository loanRepository;
    @Mock ApplicantRepository applicantRepository;
    @Mock CreditScoreProvider creditScoreProvider;
    @Mock NotificationPort notificationPort;
    @Mock AuditPort auditPort;

    ApplyForLoanService service;

    @BeforeEach
    void setUp() {
        service = new ApplyForLoanService(
            loanRepository, applicantRepository,
            creditScoreProvider, notificationPort, auditPort
        );
    }

    @Test
    void applyForLoanCreatesAndSavesApplication() {
        when(applicantRepository.existsById("applicant-1")).thenReturn(true);
        when(creditScoreProvider.getCreditReport("applicant-1"))
            .thenReturn(new CreditScoreProvider.CreditReport(
                "applicant-1", 680, 0.28, 5, 0));
        when(loanRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        var command = new ApplyForLoanUseCase.ApplyForLoanCommand(
            "applicant-1", new BigDecimal("15000"), "USD", 60, "Home renovation"
        );

        LoanApplication result = service.apply(command);

        assertThat(result.getStatus()).isEqualTo(LoanStatus.SUBMITTED);
        verify(loanRepository).save(any());
        verify(notificationPort).notifyApplicationReceived(any());
        verify(auditPort).recordStatusChange(any(), eq(LoanStatus.DRAFT),
            eq(LoanStatus.SUBMITTED), eq("SYSTEM"));
    }

    @Test
    void fastTrackApplicantGetsAutoApproval() {
        when(applicantRepository.existsById("applicant-1")).thenReturn(true);
        when(creditScoreProvider.getCreditReport("applicant-1"))
            .thenReturn(new CreditScoreProvider.CreditReport(
                "applicant-1", 780, 0.15, 10, 0));
        when(loanRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        var command = new ApplyForLoanUseCase.ApplyForLoanCommand(
            "applicant-1", new BigDecimal("20000"), "USD", 36, "Car purchase"
        );

        LoanApplication result = service.apply(command);

        assertThat(result.getStatus()).isEqualTo(LoanStatus.APPROVED);
        verify(notificationPort).notifyApproval(any());
    }
}
```

---

## Summary

| Layer | Package | Dependencies |
|---|---|---|
| Domain | `domain/` | None (pure Java) |
| Input Port | `application/port/input/` | Domain only |
| Output Port | `application/port/output/` | Domain only |
| Application Service | `application/service/` | Ports only |
| REST Adapter | `adapter/input/rest/` | Spring MVC, App Services |
| JPA Adapter | `adapter/output/persistence/` | Spring Data JPA |
| External Adapter | `adapter/output/creditbureau/` | Spring WebClient |

### Key Takeaways
- Domain objects validate their own state via business methods (not setters)
- `create()` vs `reconstitute()` factory methods separate new creation from DB loading
- Adapters map between domain models and infrastructure models — never pass JPA entities to the domain
- Use case interfaces define what the system can DO; adapter interfaces define what it NEEDS
- Unit tests for the domain run without Spring — they're pure Java and extremely fast
- The domain never imports from `adapter/` or framework packages

---

## Next Part Preview

**Part 095: Testing Patterns and Anti-Patterns** covers FIRST principles, test doubles taxonomy, sociable vs solitary unit tests, approval testing, and refactoring a poorly-tested codebase into a well-tested one with proper coverage that reflects business value.
