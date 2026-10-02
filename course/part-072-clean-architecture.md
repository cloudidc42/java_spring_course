# Part 072: Clean Architecture with Spring Boot

Clean Architecture, popularized by Robert C. Martin ("Uncle Bob"), organizes code in concentric circles where inner layers know nothing about outer layers. This part implements Clean Architecture in a payment processing system.

---

## Table of Contents

1. [Clean Architecture Layers](#layers)
2. [The Dependency Rule](#dependency-rule)
3. [Entities (Enterprise Business Rules)](#entities)
4. [Use Cases (Application Business Rules)](#use-cases)
5. [Interface Adapters](#interface-adapters)
6. [Frameworks & Drivers](#frameworks)
7. [Use Case Pattern (Interactor)](#interactor)
8. [Input/Output Boundaries](#boundaries)
9. [Presenter Pattern](#presenter)
10. [Gateway Pattern](#gateway)
11. [Package Structure](#package-structure)
12. [Testing Each Layer](#testing)
13. [Comparison with Hexagonal Architecture](#comparison)
14. [Migration from 3-Layer Architecture](#migration)
15. [Real Example: Payment Processing](#payment-example)

---

## 1. Clean Architecture Layers {#layers}

```
                    ┌────────────────────────────────────────┐
                    │         Frameworks & Drivers            │  (Spring, JPA, HTTP)
                    │  ┌──────────────────────────────────┐  │
                    │  │      Interface Adapters          │  │  (Controllers, Presenters, Gateways)
                    │  │  ┌────────────────────────────┐  │  │
                    │  │  │   Application Business     │  │  │  (Use Cases / Interactors)
                    │  │  │       Rules                │  │  │
                    │  │  │  ┌─────────────────────┐  │  │  │
                    │  │  │  │  Enterprise Business │  │  │  │  (Entities)
                    │  │  │  │       Rules          │  │  │  │
                    │  │  │  └─────────────────────┘  │  │  │
                    │  │  └────────────────────────────┘  │  │
                    │  └──────────────────────────────────┘  │
                    └────────────────────────────────────────┘
```

**The Dependency Rule**: Source code dependencies can only point inward. Nothing in an inner circle can know about anything in an outer circle.

---

## 2. Package Structure {#package-structure}

```
src/main/java/com/example/payment/
├── domain/                           ← Enterprise Business Rules (innermost)
│   ├── entity/
│   │   ├── Payment.java
│   │   ├── PaymentId.java
│   │   └── Money.java
│   └── exception/
│       └── InsufficientFundsException.java
│
├── usecase/                          ← Application Business Rules
│   ├── ProcessPaymentUseCase.java    ← Use Case (Interactor)
│   ├── RefundPaymentUseCase.java
│   ├── port/
│   │   ├── in/
│   │   │   ├── ProcessPaymentInputPort.java   ← Input Boundary
│   │   │   └── RefundPaymentInputPort.java
│   │   └── out/
│   │       ├── SavePaymentOutputPort.java     ← Output Boundary
│   │       ├── LoadPaymentOutputPort.java
│   │       ├── PaymentGatewayOutputPort.java
│   │       └── NotificationOutputPort.java
│   └── model/
│       ├── ProcessPaymentCommand.java
│       ├── PaymentResult.java
│       └── RefundResult.java
│
├── adapter/                          ← Interface Adapters
│   ├── in/
│   │   ├── web/
│   │   │   ├── PaymentController.java
│   │   │   ├── request/
│   │   │   │   └── ProcessPaymentRequest.java
│   │   │   └── response/
│   │   │       └── PaymentResponse.java
│   │   └── messaging/
│   │       └── PaymentCommandConsumer.java
│   └── out/
│       ├── persistence/
│       │   ├── PaymentPersistenceAdapter.java
│       │   ├── entity/
│       │   │   └── PaymentJpaEntity.java
│       │   ├── repository/
│       │   │   └── SpringDataPaymentRepository.java
│       │   └── mapper/
│       │       └── PaymentPersistenceMapper.java
│       ├── payment/
│       │   └── StripePaymentGatewayAdapter.java
│       └── notification/
│           └── EmailNotificationAdapter.java
│
└── config/                           ← Configuration (outermost)
    └── PaymentConfig.java
```

---

## 3. Entities (Enterprise Business Rules) {#entities}

Entities encapsulate enterprise-wide business rules. They are the most stable, least likely to change.

```java
// src/main/java/com/example/payment/domain/entity/Payment.java
package com.example.payment.domain.entity;

import com.example.payment.domain.exception.PaymentException;

import java.time.Instant;
import java.util.Objects;

public class Payment {

    public enum Status {
        PENDING, PROCESSING, COMPLETED, FAILED, REFUNDED, PARTIALLY_REFUNDED
    }

    private final PaymentId id;
    private final String customerId;
    private final String merchantId;
    private final Money amount;
    private Money refundedAmount;
    private Status status;
    private String externalTransactionId;
    private String failureReason;
    private final Instant createdAt;
    private Instant processedAt;

    private Payment(PaymentId id, String customerId, String merchantId,
                    Money amount, Instant createdAt) {
        this.id = Objects.requireNonNull(id);
        this.customerId = Objects.requireNonNull(customerId);
        this.merchantId = Objects.requireNonNull(merchantId);
        this.amount = Objects.requireNonNull(amount);
        this.refundedAmount = Money.zero(amount.getCurrency());
        this.status = Status.PENDING;
        this.createdAt = Objects.requireNonNull(createdAt);
    }

    // Factory method
    public static Payment create(String customerId, String merchantId, Money amount) {
        return new Payment(
            PaymentId.generate(),
            customerId,
            merchantId,
            amount,
            Instant.now()
        );
    }

    // Reconstitution
    public static Payment reconstitute(PaymentId id, String customerId, String merchantId,
                                        Money amount, Money refundedAmount, Status status,
                                        String externalTransactionId, String failureReason,
                                        Instant createdAt, Instant processedAt) {
        Payment payment = new Payment(id, customerId, merchantId, amount, createdAt);
        payment.refundedAmount = refundedAmount;
        payment.status = status;
        payment.externalTransactionId = externalTransactionId;
        payment.failureReason = failureReason;
        payment.processedAt = processedAt;
        return payment;
    }

    // Business rules

    public void startProcessing() {
        if (status != Status.PENDING) {
            throw new PaymentException("Payment must be PENDING to start processing, but was: " + status);
        }
        this.status = Status.PROCESSING;
    }

    public void complete(String externalTransactionId) {
        if (status != Status.PROCESSING) {
            throw new PaymentException("Payment must be PROCESSING to complete, but was: " + status);
        }
        this.externalTransactionId = Objects.requireNonNull(externalTransactionId);
        this.status = Status.COMPLETED;
        this.processedAt = Instant.now();
    }

    public void fail(String reason) {
        if (status != Status.PROCESSING && status != Status.PENDING) {
            throw new PaymentException("Cannot fail payment in status: " + status);
        }
        this.failureReason = reason;
        this.status = Status.FAILED;
        this.processedAt = Instant.now();
    }

    public void refund(Money refundAmount) {
        if (status != Status.COMPLETED && status != Status.PARTIALLY_REFUNDED) {
            throw new PaymentException("Cannot refund payment in status: " + status);
        }

        Money newRefundedAmount = refundedAmount.add(refundAmount);
        if (newRefundedAmount.isGreaterThan(amount)) {
            throw new PaymentException(
                "Refund amount " + refundAmount + " exceeds remaining refundable amount");
        }

        this.refundedAmount = newRefundedAmount;
        if (refundedAmount.equals(amount)) {
            this.status = Status.REFUNDED;
        } else {
            this.status = Status.PARTIALLY_REFUNDED;
        }
    }

    public Money getRemainingRefundableAmount() {
        return amount.subtract(refundedAmount);
    }

    public boolean isRefundable() {
        return status == Status.COMPLETED || status == Status.PARTIALLY_REFUNDED;
    }

    // Getters
    public PaymentId getId() { return id; }
    public String getCustomerId() { return customerId; }
    public String getMerchantId() { return merchantId; }
    public Money getAmount() { return amount; }
    public Money getRefundedAmount() { return refundedAmount; }
    public Status getStatus() { return status; }
    public String getExternalTransactionId() { return externalTransactionId; }
    public String getFailureReason() { return failureReason; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getProcessedAt() { return processedAt; }
}
```

```java
// src/main/java/com/example/payment/domain/entity/Money.java
package com.example.payment.domain.entity;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Objects;

public final class Money {

    private final BigDecimal amount;
    private final String currency;

    private Money(BigDecimal amount, String currency) {
        if (amount == null) throw new IllegalArgumentException("Amount cannot be null");
        if (currency == null || currency.isBlank())
            throw new IllegalArgumentException("Currency cannot be blank");
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency.toUpperCase();
    }

    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }

    public static Money of(String amount, String currency) {
        return new Money(new BigDecimal(amount), currency);
    }

    public static Money zero(String currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        BigDecimal result = amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Result would be negative");
        return new Money(result, currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    public boolean isLessThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) < 0;
    }

    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch: " + currency + " vs " + other.currency);
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return Objects.equals(amount, money.amount) && Objects.equals(currency, money.currency);
    }

    @Override public int hashCode() { return Objects.hash(amount, currency); }

    @Override public String toString() { return amount.toPlainString() + " " + currency; }
}
```

```java
// src/main/java/com/example/payment/domain/entity/PaymentId.java
package com.example.payment.domain.entity;

import java.util.Objects;
import java.util.UUID;

public final class PaymentId {
    private final UUID value;

    private PaymentId(UUID value) { this.value = Objects.requireNonNull(value); }

    public static PaymentId of(UUID value) { return new PaymentId(value); }
    public static PaymentId of(String value) { return new PaymentId(UUID.fromString(value)); }
    public static PaymentId generate() { return new PaymentId(UUID.randomUUID()); }

    public UUID getValue() { return value; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PaymentId)) return false;
        return Objects.equals(value, ((PaymentId) o).value);
    }
    @Override public int hashCode() { return Objects.hash(value); }
    @Override public String toString() { return value.toString(); }
}
```

---

## 4. Use Case / Interactor Pattern {#interactor}

The use case / interactor contains application-specific business rules.

```java
// src/main/java/com/example/payment/usecase/port/in/ProcessPaymentInputPort.java
package com.example.payment.usecase.port.in;

import com.example.payment.usecase.model.ProcessPaymentCommand;
import com.example.payment.usecase.model.PaymentResult;

/**
 * Input boundary — defines what the use case can do.
 * The controller depends on this interface, not on the use case class.
 */
public interface ProcessPaymentInputPort {
    PaymentResult process(ProcessPaymentCommand command);
}
```

```java
// src/main/java/com/example/payment/usecase/model/ProcessPaymentCommand.java
package com.example.payment.usecase.model;

import java.math.BigDecimal;
import java.util.Objects;

/**
 * Input data structure — carries data from the delivery mechanism to the use case.
 * Must be independent of HTTP, CLI, or any delivery mechanism.
 */
public class ProcessPaymentCommand {

    private final String customerId;
    private final String merchantId;
    private final BigDecimal amount;
    private final String currency;
    private final String paymentMethodToken;
    private final String idempotencyKey;

    public ProcessPaymentCommand(String customerId, String merchantId,
                                  BigDecimal amount, String currency,
                                  String paymentMethodToken, String idempotencyKey) {
        this.customerId = Objects.requireNonNull(customerId, "customerId required");
        this.merchantId = Objects.requireNonNull(merchantId, "merchantId required");
        this.amount = Objects.requireNonNull(amount, "amount required");
        this.currency = Objects.requireNonNull(currency, "currency required");
        this.paymentMethodToken = Objects.requireNonNull(paymentMethodToken, "paymentMethodToken required");
        this.idempotencyKey = Objects.requireNonNull(idempotencyKey, "idempotencyKey required");

        if (amount.compareTo(BigDecimal.ZERO) <= 0)
            throw new IllegalArgumentException("Amount must be positive");
    }

    public String getCustomerId() { return customerId; }
    public String getMerchantId() { return merchantId; }
    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
    public String getPaymentMethodToken() { return paymentMethodToken; }
    public String getIdempotencyKey() { return idempotencyKey; }
}
```

```java
// src/main/java/com/example/payment/usecase/model/PaymentResult.java
package com.example.payment.usecase.model;

import java.math.BigDecimal;
import java.time.Instant;

/**
 * Output data structure — carries data from use case to presenter/controller.
 * Must be independent of any delivery mechanism.
 */
public class PaymentResult {

    private final String paymentId;
    private final String status;
    private final BigDecimal amount;
    private final String currency;
    private final String externalTransactionId;
    private final String failureReason;
    private final Instant processedAt;

    private PaymentResult(Builder builder) {
        this.paymentId = builder.paymentId;
        this.status = builder.status;
        this.amount = builder.amount;
        this.currency = builder.currency;
        this.externalTransactionId = builder.externalTransactionId;
        this.failureReason = builder.failureReason;
        this.processedAt = builder.processedAt;
    }

    public boolean isSuccess() { return "COMPLETED".equals(status); }
    public boolean isFailure() { return "FAILED".equals(status); }

    public String getPaymentId() { return paymentId; }
    public String getStatus() { return status; }
    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
    public String getExternalTransactionId() { return externalTransactionId; }
    public String getFailureReason() { return failureReason; }
    public Instant getProcessedAt() { return processedAt; }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String paymentId;
        private String status;
        private BigDecimal amount;
        private String currency;
        private String externalTransactionId;
        private String failureReason;
        private Instant processedAt;

        public Builder paymentId(String paymentId) { this.paymentId = paymentId; return this; }
        public Builder status(String status) { this.status = status; return this; }
        public Builder amount(BigDecimal amount) { this.amount = amount; return this; }
        public Builder currency(String currency) { this.currency = currency; return this; }
        public Builder externalTransactionId(String id) { this.externalTransactionId = id; return this; }
        public Builder failureReason(String reason) { this.failureReason = reason; return this; }
        public Builder processedAt(Instant processedAt) { this.processedAt = processedAt; return this; }
        public PaymentResult build() { return new PaymentResult(this); }
    }
}
```

---

## 5. Output Ports (Boundaries) {#boundaries}

```java
// src/main/java/com/example/payment/usecase/port/out/SavePaymentOutputPort.java
package com.example.payment.usecase.port.out;

import com.example.payment.domain.entity.Payment;

public interface SavePaymentOutputPort {
    void save(Payment payment);
}
```

```java
// src/main/java/com/example/payment/usecase/port/out/LoadPaymentOutputPort.java
package com.example.payment.usecase.port.out;

import com.example.payment.domain.entity.*;
import java.util.Optional;

public interface LoadPaymentOutputPort {
    Optional<Payment> loadById(PaymentId id);
    Optional<Payment> loadByIdempotencyKey(String idempotencyKey);
}
```

```java
// src/main/java/com/example/payment/usecase/port/out/PaymentGatewayOutputPort.java
package com.example.payment.usecase.port.out;

import com.example.payment.domain.entity.Money;

/**
 * Output port (boundary) for external payment processing.
 * The use case depends on this interface, not on Stripe/PayPal/etc.
 */
public interface PaymentGatewayOutputPort {

    PaymentGatewayResult charge(String customerId, String paymentMethodToken, Money amount);

    PaymentGatewayResult refund(String externalTransactionId, Money amount);

    record PaymentGatewayResult(
        boolean success,
        String transactionId,
        String errorCode,
        String errorMessage
    ) {
        public static PaymentGatewayResult success(String transactionId) {
            return new PaymentGatewayResult(true, transactionId, null, null);
        }

        public static PaymentGatewayResult failure(String errorCode, String errorMessage) {
            return new PaymentGatewayResult(false, null, errorCode, errorMessage);
        }
    }
}
```

```java
// src/main/java/com/example/payment/usecase/port/out/NotificationOutputPort.java
package com.example.payment.usecase.port.out;

import com.example.payment.domain.entity.Payment;

public interface NotificationOutputPort {
    void notifyPaymentCompleted(Payment payment);
    void notifyPaymentFailed(Payment payment);
    void notifyRefundProcessed(Payment payment);
}
```

---

## 6. The Use Case Interactor {#use-cases}

```java
// src/main/java/com/example/payment/usecase/ProcessPaymentUseCase.java
package com.example.payment.usecase;

import com.example.payment.domain.entity.*;
import com.example.payment.usecase.model.*;
import com.example.payment.usecase.port.in.ProcessPaymentInputPort;
import com.example.payment.usecase.port.out.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
public class ProcessPaymentUseCase implements ProcessPaymentInputPort {

    private static final Logger log = LoggerFactory.getLogger(ProcessPaymentUseCase.class);

    private final LoadPaymentOutputPort loadPayment;
    private final SavePaymentOutputPort savePayment;
    private final PaymentGatewayOutputPort paymentGateway;
    private final NotificationOutputPort notification;

    public ProcessPaymentUseCase(
            LoadPaymentOutputPort loadPayment,
            SavePaymentOutputPort savePayment,
            PaymentGatewayOutputPort paymentGateway,
            NotificationOutputPort notification) {
        this.loadPayment = loadPayment;
        this.savePayment = savePayment;
        this.paymentGateway = paymentGateway;
        this.notification = notification;
    }

    @Override
    @Transactional
    public PaymentResult process(ProcessPaymentCommand command) {
        log.info("Processing payment for customer={}, amount={} {}",
            command.getCustomerId(), command.getAmount(), command.getCurrency());

        // Idempotency check
        Optional<Payment> existing = loadPayment.loadByIdempotencyKey(command.getIdempotencyKey());
        if (existing.isPresent()) {
            log.info("Duplicate request detected, returning existing result for key={}",
                command.getIdempotencyKey());
            return toResult(existing.get());
        }

        // Create payment entity
        Money amount = Money.of(command.getAmount(), command.getCurrency());
        Payment payment = Payment.create(
            command.getCustomerId(),
            command.getMerchantId(),
            amount
        );

        // Start processing
        payment.startProcessing();
        savePayment.save(payment);

        // Call external payment gateway
        PaymentGatewayOutputPort.PaymentGatewayResult gatewayResult = paymentGateway.charge(
            command.getCustomerId(),
            command.getPaymentMethodToken(),
            amount
        );

        // Update payment based on gateway result
        if (gatewayResult.success()) {
            payment.complete(gatewayResult.transactionId());
            savePayment.save(payment);
            notification.notifyPaymentCompleted(payment);
            log.info("Payment completed: paymentId={}, transactionId={}",
                payment.getId(), gatewayResult.transactionId());
        } else {
            payment.fail(gatewayResult.errorMessage());
            savePayment.save(payment);
            notification.notifyPaymentFailed(payment);
            log.warn("Payment failed: paymentId={}, reason={}",
                payment.getId(), gatewayResult.errorMessage());
        }

        return toResult(payment);
    }

    private PaymentResult toResult(Payment payment) {
        return PaymentResult.builder()
            .paymentId(payment.getId().toString())
            .status(payment.getStatus().name())
            .amount(payment.getAmount().getAmount())
            .currency(payment.getAmount().getCurrency())
            .externalTransactionId(payment.getExternalTransactionId())
            .failureReason(payment.getFailureReason())
            .processedAt(payment.getProcessedAt())
            .build();
    }
}
```

```java
// src/main/java/com/example/payment/usecase/RefundPaymentUseCase.java
package com.example.payment.usecase;

import com.example.payment.domain.entity.*;
import com.example.payment.usecase.model.*;
import com.example.payment.usecase.port.in.RefundPaymentInputPort;
import com.example.payment.usecase.port.out.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class RefundPaymentUseCase implements RefundPaymentInputPort {

    private final LoadPaymentOutputPort loadPayment;
    private final SavePaymentOutputPort savePayment;
    private final PaymentGatewayOutputPort paymentGateway;
    private final NotificationOutputPort notification;

    public RefundPaymentUseCase(LoadPaymentOutputPort loadPayment,
                                 SavePaymentOutputPort savePayment,
                                 PaymentGatewayOutputPort paymentGateway,
                                 NotificationOutputPort notification) {
        this.loadPayment = loadPayment;
        this.savePayment = savePayment;
        this.paymentGateway = paymentGateway;
        this.notification = notification;
    }

    @Override
    @Transactional
    public RefundResult refund(RefundCommand command) {
        PaymentId paymentId = PaymentId.of(command.getPaymentId());

        Payment payment = loadPayment.loadById(paymentId)
            .orElseThrow(() -> new PaymentNotFoundException("Payment not found: " + paymentId));

        if (!payment.isRefundable()) {
            throw new PaymentNotRefundableException(
                "Payment " + paymentId + " is not refundable in status: " + payment.getStatus());
        }

        Money refundAmount = Money.of(command.getAmount(), command.getCurrency());

        // Validate refund amount
        if (refundAmount.isGreaterThan(payment.getRemainingRefundableAmount())) {
            throw new ExcessiveRefundException(
                "Refund amount " + refundAmount + " exceeds remaining " +
                payment.getRemainingRefundableAmount());
        }

        // Process refund via gateway
        PaymentGatewayOutputPort.PaymentGatewayResult gatewayResult =
            paymentGateway.refund(payment.getExternalTransactionId(), refundAmount);

        if (!gatewayResult.success()) {
            throw new RefundFailedException("Refund failed: " + gatewayResult.errorMessage());
        }

        // Update domain entity
        payment.refund(refundAmount);
        savePayment.save(payment);
        notification.notifyRefundProcessed(payment);

        return RefundResult.builder()
            .paymentId(payment.getId().toString())
            .refundedAmount(refundAmount.getAmount())
            .currency(refundAmount.getCurrency())
            .newStatus(payment.getStatus().name())
            .build();
    }
}
```

```java
// src/main/java/com/example/payment/usecase/port/in/RefundPaymentInputPort.java
package com.example.payment.usecase.port.in;

import com.example.payment.usecase.model.*;

public interface RefundPaymentInputPort {
    RefundResult refund(RefundCommand command);
}
```

```java
// src/main/java/com/example/payment/usecase/model/RefundCommand.java
package com.example.payment.usecase.model;

import java.math.BigDecimal;

public class RefundCommand {
    private final String paymentId;
    private final BigDecimal amount;
    private final String currency;
    private final String reason;

    public RefundCommand(String paymentId, BigDecimal amount, String currency, String reason) {
        this.paymentId = paymentId;
        this.amount = amount;
        this.currency = currency;
        this.reason = reason;
    }

    public String getPaymentId() { return paymentId; }
    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
    public String getReason() { return reason; }
}
```

```java
// src/main/java/com/example/payment/usecase/model/RefundResult.java
package com.example.payment.usecase.model;

import java.math.BigDecimal;

public class RefundResult {
    private final String paymentId;
    private final BigDecimal refundedAmount;
    private final String currency;
    private final String newStatus;

    private RefundResult(Builder b) {
        this.paymentId = b.paymentId;
        this.refundedAmount = b.refundedAmount;
        this.currency = b.currency;
        this.newStatus = b.newStatus;
    }

    public String getPaymentId() { return paymentId; }
    public BigDecimal getRefundedAmount() { return refundedAmount; }
    public String getCurrency() { return currency; }
    public String getNewStatus() { return newStatus; }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String paymentId;
        private BigDecimal refundedAmount;
        private String currency;
        private String newStatus;

        public Builder paymentId(String v) { this.paymentId = v; return this; }
        public Builder refundedAmount(BigDecimal v) { this.refundedAmount = v; return this; }
        public Builder currency(String v) { this.currency = v; return this; }
        public Builder newStatus(String v) { this.newStatus = v; return this; }
        public RefundResult build() { return new RefundResult(this); }
    }
}
```

---

## 7. Interface Adapters (Controllers) {#interface-adapters}

```java
// src/main/java/com/example/payment/adapter/in/web/PaymentController.java
package com.example.payment.adapter.in.web;

import com.example.payment.adapter.in.web.request.*;
import com.example.payment.adapter.in.web.response.*;
import com.example.payment.usecase.model.*;
import com.example.payment.usecase.port.in.*;
import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    private final ProcessPaymentInputPort processPayment;
    private final RefundPaymentInputPort refundPayment;

    public PaymentController(ProcessPaymentInputPort processPayment,
                              RefundPaymentInputPort refundPayment) {
        this.processPayment = processPayment;
        this.refundPayment = refundPayment;
    }

    @PostMapping
    public ResponseEntity<PaymentResponse> processPayment(
            @Valid @RequestBody ProcessPaymentRequest request) {

        ProcessPaymentCommand command = new ProcessPaymentCommand(
            request.getCustomerId(),
            request.getMerchantId(),
            request.getAmount(),
            request.getCurrency(),
            request.getPaymentMethodToken(),
            request.getIdempotencyKey() != null
                ? request.getIdempotencyKey()
                : UUID.randomUUID().toString()
        );

        PaymentResult result = processPayment.process(command);

        PaymentResponse response = PaymentResponse.from(result);
        HttpStatus status = result.isSuccess() ? HttpStatus.CREATED : HttpStatus.UNPROCESSABLE_ENTITY;
        return ResponseEntity.status(status).body(response);
    }

    @PostMapping("/{paymentId}/refund")
    public ResponseEntity<RefundResponse> refundPayment(
            @PathVariable String paymentId,
            @Valid @RequestBody RefundRequest request) {

        RefundCommand command = new RefundCommand(
            paymentId,
            request.getAmount(),
            request.getCurrency(),
            request.getReason()
        );

        RefundResult result = refundPayment.refund(command);
        return ResponseEntity.ok(RefundResponse.from(result));
    }
}
```

```java
// src/main/java/com/example/payment/adapter/in/web/request/ProcessPaymentRequest.java
package com.example.payment.adapter.in.web.request;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public class ProcessPaymentRequest {

    @NotBlank
    private String customerId;

    @NotBlank
    private String merchantId;

    @NotNull
    @Positive
    private BigDecimal amount;

    @NotBlank
    @Size(min = 3, max = 3)
    private String currency;

    @NotBlank
    private String paymentMethodToken;

    private String idempotencyKey;

    // Getters and setters
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public String getMerchantId() { return merchantId; }
    public void setMerchantId(String merchantId) { this.merchantId = merchantId; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
    public String getPaymentMethodToken() { return paymentMethodToken; }
    public void setPaymentMethodToken(String token) { this.paymentMethodToken = token; }
    public String getIdempotencyKey() { return idempotencyKey; }
    public void setIdempotencyKey(String key) { this.idempotencyKey = key; }
}
```

```java
// src/main/java/com/example/payment/adapter/in/web/response/PaymentResponse.java
package com.example.payment.adapter.in.web.response;

import com.example.payment.usecase.model.PaymentResult;
import java.math.BigDecimal;
import java.time.Instant;

public class PaymentResponse {

    private String paymentId;
    private String status;
    private BigDecimal amount;
    private String currency;
    private String transactionId;
    private String errorMessage;
    private Instant processedAt;

    public static PaymentResponse from(PaymentResult result) {
        PaymentResponse response = new PaymentResponse();
        response.paymentId = result.getPaymentId();
        response.status = result.getStatus();
        response.amount = result.getAmount();
        response.currency = result.getCurrency();
        response.transactionId = result.getExternalTransactionId();
        response.errorMessage = result.getFailureReason();
        response.processedAt = result.getProcessedAt();
        return response;
    }

    public String getPaymentId() { return paymentId; }
    public String getStatus() { return status; }
    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
    public String getTransactionId() { return transactionId; }
    public String getErrorMessage() { return errorMessage; }
    public Instant getProcessedAt() { return processedAt; }
}
```

---

## 8. Gateway Pattern for External Services {#gateway}

```java
// src/main/java/com/example/payment/adapter/out/payment/StripePaymentGatewayAdapter.java
package com.example.payment.adapter.out.payment;

import com.example.payment.domain.entity.Money;
import com.example.payment.usecase.port.out.PaymentGatewayOutputPort;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

@Component
public class StripePaymentGatewayAdapter implements PaymentGatewayOutputPort {

    private static final Logger log = LoggerFactory.getLogger(StripePaymentGatewayAdapter.class);

    private final RestTemplate restTemplate;
    private final String apiKey;
    private final String baseUrl;

    public StripePaymentGatewayAdapter(
            RestTemplate restTemplate,
            @Value("${stripe.api-key}") String apiKey,
            @Value("${stripe.base-url:https://api.stripe.com}") String baseUrl) {
        this.restTemplate = restTemplate;
        this.apiKey = apiKey;
        this.baseUrl = baseUrl;
    }

    @Override
    public PaymentGatewayResult charge(String customerId, String paymentMethodToken, Money amount) {
        log.info("Charging via Stripe: customer={}, amount={}", customerId, amount);

        try {
            // Convert amount to cents (Stripe uses smallest currency unit)
            long amountInCents = amount.getAmount()
                .multiply(new java.math.BigDecimal("100"))
                .longValueExact();

            StripeChargeRequest request = new StripeChargeRequest(
                amountInCents,
                amount.getCurrency().toLowerCase(),
                paymentMethodToken,
                "Payment for customer " + customerId
            );

            StripeChargeResponse response = restTemplate.postForObject(
                baseUrl + "/v1/charges",
                request,
                StripeChargeResponse.class
            );

            if (response != null && "succeeded".equals(response.status())) {
                return PaymentGatewayResult.success(response.id());
            } else {
                return PaymentGatewayResult.failure("CHARGE_FAILED", "Stripe charge failed");
            }
        } catch (Exception e) {
            log.error("Stripe charge failed", e);
            return PaymentGatewayResult.failure("GATEWAY_ERROR", e.getMessage());
        }
    }

    @Override
    public PaymentGatewayResult refund(String externalTransactionId, Money amount) {
        log.info("Refunding via Stripe: transactionId={}, amount={}", externalTransactionId, amount);

        try {
            long amountInCents = amount.getAmount()
                .multiply(new java.math.BigDecimal("100"))
                .longValueExact();

            StripeRefundRequest request = new StripeRefundRequest(
                externalTransactionId,
                amountInCents
            );

            StripeRefundResponse response = restTemplate.postForObject(
                baseUrl + "/v1/refunds",
                request,
                StripeRefundResponse.class
            );

            if (response != null && "succeeded".equals(response.status())) {
                return PaymentGatewayResult.success(response.id());
            } else {
                return PaymentGatewayResult.failure("REFUND_FAILED", "Stripe refund failed");
            }
        } catch (Exception e) {
            log.error("Stripe refund failed", e);
            return PaymentGatewayResult.failure("GATEWAY_ERROR", e.getMessage());
        }
    }

    record StripeChargeRequest(long amount, String currency, String source, String description) {}
    record StripeChargeResponse(String id, String status, String failure_message) {}
    record StripeRefundRequest(String charge, long amount) {}
    record StripeRefundResponse(String id, String status) {}
}
```

---

## 9. Persistence Adapter

```java
// src/main/java/com/example/payment/adapter/out/persistence/PaymentPersistenceAdapter.java
package com.example.payment.adapter.out.persistence;

import com.example.payment.domain.entity.*;
import com.example.payment.usecase.port.out.*;
import org.springframework.stereotype.Component;

import java.util.Optional;

@Component
public class PaymentPersistenceAdapter implements SavePaymentOutputPort, LoadPaymentOutputPort {

    private final SpringDataPaymentRepository repository;
    private final PaymentPersistenceMapper mapper;

    public PaymentPersistenceAdapter(SpringDataPaymentRepository repository,
                                      PaymentPersistenceMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public void save(Payment payment) {
        PaymentJpaEntity entity = mapper.toJpaEntity(payment);
        repository.save(entity);
    }

    @Override
    public Optional<Payment> loadById(PaymentId id) {
        return repository.findById(id.getValue())
            .map(mapper::toDomain);
    }

    @Override
    public Optional<Payment> loadByIdempotencyKey(String idempotencyKey) {
        return repository.findByIdempotencyKey(idempotencyKey)
            .map(mapper::toDomain);
    }
}
```

```java
// src/main/java/com/example/payment/adapter/out/persistence/PaymentJpaEntity.java
package com.example.payment.adapter.out.persistence;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "payments")
public class PaymentJpaEntity {

    @Id
    private UUID id;

    @Column(nullable = false)
    private String customerId;

    @Column(nullable = false)
    private String merchantId;

    @Column(nullable = false)
    private BigDecimal amount;

    @Column(nullable = false)
    private BigDecimal refundedAmount;

    @Column(nullable = false)
    private String currency;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private String status;

    @Column(unique = true)
    private String idempotencyKey;

    private String externalTransactionId;
    private String failureReason;

    @Column(nullable = false)
    private Instant createdAt;

    private Instant processedAt;

    // Getters and setters omitted for brevity — standard getters/setters
    public UUID getId() { return id; }
    public void setId(UUID id) { this.id = id; }
    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public String getMerchantId() { return merchantId; }
    public void setMerchantId(String merchantId) { this.merchantId = merchantId; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    public BigDecimal getRefundedAmount() { return refundedAmount; }
    public void setRefundedAmount(BigDecimal refundedAmount) { this.refundedAmount = refundedAmount; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String getIdempotencyKey() { return idempotencyKey; }
    public void setIdempotencyKey(String key) { this.idempotencyKey = key; }
    public String getExternalTransactionId() { return externalTransactionId; }
    public void setExternalTransactionId(String id) { this.externalTransactionId = id; }
    public String getFailureReason() { return failureReason; }
    public void setFailureReason(String reason) { this.failureReason = reason; }
    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }
    public Instant getProcessedAt() { return processedAt; }
    public void setProcessedAt(Instant processedAt) { this.processedAt = processedAt; }
}
```

---

## 10. Testing Each Layer in Isolation {#testing}

```java
// src/test/java/com/example/payment/usecase/ProcessPaymentUseCaseTest.java
package com.example.payment.usecase;

import com.example.payment.domain.entity.*;
import com.example.payment.usecase.model.*;
import com.example.payment.usecase.port.out.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ProcessPaymentUseCaseTest {

    @Mock
    private LoadPaymentOutputPort loadPayment;

    @Mock
    private SavePaymentOutputPort savePayment;

    @Mock
    private PaymentGatewayOutputPort paymentGateway;

    @Mock
    private NotificationOutputPort notification;

    @InjectMocks
    private ProcessPaymentUseCase useCase;

    private ProcessPaymentCommand validCommand;

    @BeforeEach
    void setUp() {
        validCommand = new ProcessPaymentCommand(
            "customer-123",
            "merchant-456",
            new BigDecimal("100.00"),
            "USD",
            "pm_card_visa",
            "idem-key-001"
        );
    }

    @Test
    void shouldProcessPaymentSuccessfully() {
        // Given
        when(loadPayment.loadByIdempotencyKey("idem-key-001"))
            .thenReturn(Optional.empty());
        when(paymentGateway.charge(anyString(), anyString(), any()))
            .thenReturn(PaymentGatewayOutputPort.PaymentGatewayResult.success("txn-stripe-001"));

        // When
        PaymentResult result = useCase.process(validCommand);

        // Then
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getExternalTransactionId()).isEqualTo("txn-stripe-001");
        assertThat(result.getStatus()).isEqualTo("COMPLETED");

        verify(savePayment, times(2)).save(any()); // once for PROCESSING, once for COMPLETED
        verify(notification).notifyPaymentCompleted(any());
        verify(notification, never()).notifyPaymentFailed(any());
    }

    @Test
    void shouldReturnExistingResultForDuplicateIdempotencyKey() {
        // Given
        Payment existingPayment = Payment.create("customer-123", "merchant-456",
            Money.of("100.00", "USD"));
        existingPayment.startProcessing();
        existingPayment.complete("txn-existing-001");

        when(loadPayment.loadByIdempotencyKey("idem-key-001"))
            .thenReturn(Optional.of(existingPayment));

        // When
        PaymentResult result = useCase.process(validCommand);

        // Then
        assertThat(result.getStatus()).isEqualTo("COMPLETED");
        verify(paymentGateway, never()).charge(anyString(), anyString(), any());
        verify(savePayment, never()).save(any());
    }

    @Test
    void shouldFailPaymentWhenGatewayDeclines() {
        // Given
        when(loadPayment.loadByIdempotencyKey(anyString()))
            .thenReturn(Optional.empty());
        when(paymentGateway.charge(anyString(), anyString(), any()))
            .thenReturn(PaymentGatewayOutputPort.PaymentGatewayResult
                .failure("CARD_DECLINED", "Your card was declined"));

        // When
        PaymentResult result = useCase.process(validCommand);

        // Then
        assertThat(result.isFailure()).isTrue();
        assertThat(result.getFailureReason()).contains("declined");
        verify(notification).notifyPaymentFailed(any());
    }
}
```

```java
// Test domain entity in isolation — no mocking needed!
// src/test/java/com/example/payment/domain/entity/PaymentTest.java
package com.example.payment.domain.entity;

import com.example.payment.domain.exception.PaymentException;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

class PaymentTest {

    @Test
    void shouldCreatePaymentInPendingStatus() {
        Payment payment = Payment.create("customer-1", "merchant-1",
            Money.of("50.00", "USD"));

        assertThat(payment.getStatus()).isEqualTo(Payment.Status.PENDING);
        assertThat(payment.getId()).isNotNull();
        assertThat(payment.getRefundedAmount()).isEqualTo(Money.zero("USD"));
    }

    @Test
    void shouldTransitionThroughCompleteLifecycle() {
        Payment payment = Payment.create("customer-1", "merchant-1",
            Money.of("100.00", "USD"));

        payment.startProcessing();
        assertThat(payment.getStatus()).isEqualTo(Payment.Status.PROCESSING);

        payment.complete("txn-001");
        assertThat(payment.getStatus()).isEqualTo(Payment.Status.COMPLETED);
        assertThat(payment.getExternalTransactionId()).isEqualTo("txn-001");
    }

    @Test
    void shouldHandlePartialRefund() {
        Payment payment = createCompletedPayment("100.00");

        payment.refund(Money.of("30.00", "USD"));

        assertThat(payment.getStatus()).isEqualTo(Payment.Status.PARTIALLY_REFUNDED);
        assertThat(payment.getRefundedAmount()).isEqualTo(Money.of("30.00", "USD"));
        assertThat(payment.getRemainingRefundableAmount()).isEqualTo(Money.of("70.00", "USD"));
    }

    @Test
    void shouldHandleFullRefund() {
        Payment payment = createCompletedPayment("100.00");

        payment.refund(Money.of("100.00", "USD"));

        assertThat(payment.getStatus()).isEqualTo(Payment.Status.REFUNDED);
    }

    @Test
    void shouldNotAllowRefundExceedingOriginalAmount() {
        Payment payment = createCompletedPayment("100.00");

        assertThatThrownBy(() -> payment.refund(Money.of("150.00", "USD")))
            .isInstanceOf(PaymentException.class);
    }

    @Test
    void shouldNotAllowStartingAlreadyProcessingPayment() {
        Payment payment = Payment.create("c", "m", Money.of("10.00", "USD"));
        payment.startProcessing();

        assertThatThrownBy(payment::startProcessing)
            .isInstanceOf(PaymentException.class);
    }

    private Payment createCompletedPayment(String amount) {
        Payment payment = Payment.create("customer-1", "merchant-1", Money.of(amount, "USD"));
        payment.startProcessing();
        payment.complete("txn-001");
        return payment;
    }
}
```

```java
// src/test/java/com/example/payment/adapter/in/web/PaymentControllerTest.java
package com.example.payment.adapter.in.web;

import com.example.payment.usecase.model.PaymentResult;
import com.example.payment.usecase.port.in.ProcessPaymentInputPort;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Map;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(PaymentController.class)
class PaymentControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private ProcessPaymentInputPort processPayment;

    @MockBean
    private com.example.payment.usecase.port.in.RefundPaymentInputPort refundPayment;

    @Test
    void shouldReturn201WhenPaymentSucceeds() throws Exception {
        PaymentResult successResult = PaymentResult.builder()
            .paymentId("pay-001")
            .status("COMPLETED")
            .amount(new BigDecimal("100.00"))
            .currency("USD")
            .externalTransactionId("txn-001")
            .processedAt(Instant.now())
            .build();

        when(processPayment.process(any())).thenReturn(successResult);

        Map<String, Object> body = Map.of(
            "customerId", "customer-123",
            "merchantId", "merchant-456",
            "amount", 100.00,
            "currency", "USD",
            "paymentMethodToken", "pm_card_visa"
        );

        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(body)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("COMPLETED"))
            .andExpect(jsonPath("$.transactionId").value("txn-001"));
    }

    @Test
    void shouldReturn422WhenPaymentFails() throws Exception {
        PaymentResult failedResult = PaymentResult.builder()
            .paymentId("pay-002")
            .status("FAILED")
            .amount(new BigDecimal("100.00"))
            .currency("USD")
            .failureReason("Card declined")
            .build();

        when(processPayment.process(any())).thenReturn(failedResult);

        Map<String, Object> body = Map.of(
            "customerId", "customer-123",
            "merchantId", "merchant-456",
            "amount", 100.00,
            "currency", "USD",
            "paymentMethodToken", "pm_card_declined"
        );

        mockMvc.perform(post("/api/v1/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(body)))
            .andExpect(status().isUnprocessableEntity())
            .andExpect(jsonPath("$.errorMessage").value("Card declined"));
    }
}
```

---

## 11. Comparison with Hexagonal Architecture {#comparison}

| Aspect | Clean Architecture | Hexagonal (Ports & Adapters) |
|---|---|---|
| Core metaphor | Concentric rings | Hexagon with ports |
| Layer count | 4 rings | Domain + Adapters |
| Input/Output distinction | Use case has separate input/output ports | Explicit primary/secondary ports |
| Presenter | Explicit presenter concept | Often combined with controller |
| Entity vs Aggregate | Enterprise entities | Aggregates (DDD aligned) |
| Dependency rule | Strictly inward | Domain at center |
| Primary difference | Presenter separates formatting from use case | Same separation, less prescribed structure |

Both approaches share the same fundamental principle: **business logic must not depend on frameworks or external systems**.

---

## 12. Migration from 3-Layer to Clean Architecture {#migration}

### Step 1: Identify Business Logic in Service Layer

```java
// BEFORE: Traditional service layer (mixes business logic with infrastructure)
@Service
public class OldPaymentService {

    @Autowired
    private PaymentRepository paymentRepository;  // JPA directly

    @Autowired
    private StripeClient stripeClient;  // Stripe SDK directly

    @Autowired
    private EmailService emailService;

    @Transactional
    public PaymentDTO processPayment(PaymentRequestDTO request) {
        // Business logic mixed with infrastructure concerns
        PaymentEntity entity = new PaymentEntity();
        entity.setCustomerId(request.getCustomerId());
        entity.setAmount(request.getAmount());
        entity.setStatus("PENDING");
        paymentRepository.save(entity);

        try {
            String chargeId = stripeClient.createCharge(
                request.getAmount().multiply(BigDecimal.valueOf(100)).longValue(),
                request.getCurrency(),
                request.getToken()
            );
            entity.setExternalId(chargeId);
            entity.setStatus("COMPLETED");
        } catch (StripeException e) {
            entity.setStatus("FAILED");
            entity.setFailureReason(e.getMessage());
        }

        paymentRepository.save(entity);
        emailService.sendPaymentConfirmation(entity.getCustomerId(), entity.getAmount());

        return PaymentDTO.from(entity);
    }
}
```

### Step 2: Extract Domain Entity

Move core business logic into a pure domain object (as shown in the `Payment` entity above).

### Step 3: Define Ports (Interfaces)

Replace direct dependencies with interfaces:
- `StripeClient` → `PaymentGatewayOutputPort`
- `PaymentRepository` → `SavePaymentOutputPort` + `LoadPaymentOutputPort`
- `EmailService` → `NotificationOutputPort`

### Step 4: Create Use Case

Move orchestration logic into `ProcessPaymentUseCase` (as shown above).

### Step 5: Create Adapters

Implement ports in the infrastructure layer:
- `StripePaymentGatewayAdapter implements PaymentGatewayOutputPort`
- `PaymentPersistenceAdapter implements SavePaymentOutputPort, LoadPaymentOutputPort`
- `EmailNotificationAdapter implements NotificationOutputPort`

---

## Summary

| Layer | Contains | Depends On |
|---|---|---|
| Domain Entities | Business rules, value objects | Nothing |
| Use Cases | Interactors, input/output ports | Domain only |
| Interface Adapters | Controllers, presenters, gateways | Use Cases |
| Frameworks | Spring, JPA, HTTP, Stripe SDK | Everything |

### The Three Tests of Clean Architecture

1. **The Dependency Rule Test**: Can you compile the domain and use case layers without Spring, JPA, or any framework? They should compile and test in isolation.
2. **The Independence Test**: Can you swap Stripe for PayPal without touching use case code?
3. **The Testability Test**: Can you unit test use cases without starting a Spring context or database?

---

## Next Part Preview

**Part 073: Advanced Reactive Patterns** — build on Spring WebFlux fundamentals with advanced operators like `groupBy`, `window`, `buffer`, backpressure strategies, reactive transactions, and build a complete real-time data pipeline that handles backpressure gracefully.
