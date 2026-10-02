# Part 046: Event Sourcing and CQRS

## Overview

Event Sourcing stores the state of an aggregate as a sequence of events rather than the current state. CQRS separates read and write models for scalability. Together they enable full audit trails, temporal queries, and eventually consistent read models. This part builds a complete Bank Account system using these patterns.

---

## 1. CQRS Concept

```
TRADITIONAL (single model):
  POST /accounts/1/deposit  → (read DB + write DB same model)
  GET  /accounts/1         → (same DB table)

CQRS (separate models):
  Command Side:
    POST /accounts/1/deposit  → Command → Aggregate → Events → EventStore

  Query Side:
    GET  /accounts/1          → Read Model (denormalized view) → Response
    (Updated asynchronously from events via projections)
```

---

## 2. Project Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/eventstore
    username: postgres
    password: secret
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
  redis:
    host: localhost
    port: 6379
```

---

## 3. Events

### 3.1 Base Domain Event

```java
package com.example.eventsourcing.event;

import java.time.Instant;
import java.util.UUID;

public abstract class DomainEvent {

    private final String eventId;
    private final String aggregateId;
    private final String aggregateType;
    private final long sequenceNumber;
    private final Instant occurredAt;
    private final String eventType;

    protected DomainEvent(
            String aggregateId,
            String aggregateType,
            long sequenceNumber
    ) {
        this.eventId = UUID.randomUUID().toString();
        this.aggregateId = aggregateId;
        this.aggregateType = aggregateType;
        this.sequenceNumber = sequenceNumber;
        this.occurredAt = Instant.now();
        this.eventType = this.getClass().getSimpleName();
    }

    public String getEventId() { return eventId; }
    public String getAggregateId() { return aggregateId; }
    public String getAggregateType() { return aggregateType; }
    public long getSequenceNumber() { return sequenceNumber; }
    public Instant getOccurredAt() { return occurredAt; }
    public String getEventType() { return eventType; }
}
```

### 3.2 Bank Account Events

```java
package com.example.eventsourcing.event.account;

import com.example.eventsourcing.event.DomainEvent;
import java.math.BigDecimal;

public class AccountOpenedEvent extends DomainEvent {

    private final String ownerName;
    private final String ownerEmail;
    private final BigDecimal initialDeposit;

    public AccountOpenedEvent(
            String aggregateId,
            long sequenceNumber,
            String ownerName,
            String ownerEmail,
            BigDecimal initialDeposit
    ) {
        super(aggregateId, "BankAccount", sequenceNumber);
        this.ownerName = ownerName;
        this.ownerEmail = ownerEmail;
        this.initialDeposit = initialDeposit;
    }

    public String getOwnerName() { return ownerName; }
    public String getOwnerEmail() { return ownerEmail; }
    public BigDecimal getInitialDeposit() { return initialDeposit; }
}
```

```java
package com.example.eventsourcing.event.account;

import com.example.eventsourcing.event.DomainEvent;
import java.math.BigDecimal;

public class MoneyDepositedEvent extends DomainEvent {

    private final BigDecimal amount;
    private final String description;
    private final BigDecimal balanceAfter;

    public MoneyDepositedEvent(
            String aggregateId,
            long sequenceNumber,
            BigDecimal amount,
            String description,
            BigDecimal balanceAfter
    ) {
        super(aggregateId, "BankAccount", sequenceNumber);
        this.amount = amount;
        this.description = description;
        this.balanceAfter = balanceAfter;
    }

    public BigDecimal getAmount() { return amount; }
    public String getDescription() { return description; }
    public BigDecimal getBalanceAfter() { return balanceAfter; }
}
```

```java
package com.example.eventsourcing.event.account;

import com.example.eventsourcing.event.DomainEvent;
import java.math.BigDecimal;

public class MoneyWithdrawnEvent extends DomainEvent {

    private final BigDecimal amount;
    private final String description;
    private final BigDecimal balanceAfter;

    public MoneyWithdrawnEvent(
            String aggregateId,
            long sequenceNumber,
            BigDecimal amount,
            String description,
            BigDecimal balanceAfter
    ) {
        super(aggregateId, "BankAccount", sequenceNumber);
        this.amount = amount;
        this.description = description;
        this.balanceAfter = balanceAfter;
    }

    public BigDecimal getAmount() { return amount; }
    public String getDescription() { return description; }
    public BigDecimal getBalanceAfter() { return balanceAfter; }
}
```

```java
package com.example.eventsourcing.event.account;

import com.example.eventsourcing.event.DomainEvent;
import java.math.BigDecimal;

public class TransferInitiatedEvent extends DomainEvent {

    private final String targetAccountId;
    private final BigDecimal amount;
    private final String transferId;

    public TransferInitiatedEvent(
            String aggregateId,
            long sequenceNumber,
            String targetAccountId,
            BigDecimal amount,
            String transferId
    ) {
        super(aggregateId, "BankAccount", sequenceNumber);
        this.targetAccountId = targetAccountId;
        this.amount = amount;
        this.transferId = transferId;
    }

    public String getTargetAccountId() { return targetAccountId; }
    public BigDecimal getAmount() { return amount; }
    public String getTransferId() { return transferId; }
}
```

```java
package com.example.eventsourcing.event.account;

import com.example.eventsourcing.event.DomainEvent;

public class AccountClosedEvent extends DomainEvent {

    private final String reason;

    public AccountClosedEvent(String aggregateId, long sequenceNumber, String reason) {
        super(aggregateId, "BankAccount", sequenceNumber);
        this.reason = reason;
    }

    public String getReason() { return reason; }
}
```

---

## 4. Aggregate

```java
package com.example.eventsourcing.aggregate;

import com.example.eventsourcing.event.DomainEvent;
import com.example.eventsourcing.event.account.*;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

public class BankAccountAggregate {

    private String id;
    private String ownerName;
    private String ownerEmail;
    private BigDecimal balance = BigDecimal.ZERO;
    private AccountStatus status;
    private long version = 0;  // Optimistic concurrency control

    // Uncommitted events (to be persisted)
    private final List<DomainEvent> pendingEvents = new ArrayList<>();

    public enum AccountStatus {
        OPEN, CLOSED, SUSPENDED
    }

    // Private constructor - use factory methods
    private BankAccountAggregate() {}

    // ==================== Command Handlers ====================
    // Commands validate business rules and generate events

    public static BankAccountAggregate open(
            String ownerName,
            String ownerEmail,
            BigDecimal initialDeposit
    ) {
        if (ownerName == null || ownerName.isBlank()) {
            throw new IllegalArgumentException("Owner name is required");
        }
        if (initialDeposit.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Initial deposit cannot be negative");
        }

        BankAccountAggregate account = new BankAccountAggregate();
        String id = UUID.randomUUID().toString();

        // Apply the event (validates and changes state)
        account.applyEvent(new AccountOpenedEvent(
            id, 1, ownerName, ownerEmail, initialDeposit
        ));

        return account;
    }

    public void deposit(BigDecimal amount, String description) {
        validateOpen();
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        if (amount.compareTo(new BigDecimal("1000000")) > 0) {
            throw new IllegalArgumentException("Single deposit cannot exceed 1,000,000");
        }

        BigDecimal newBalance = balance.add(amount);
        applyEvent(new MoneyDepositedEvent(id, nextVersion(), amount, description, newBalance));
    }

    public void withdraw(BigDecimal amount, String description) {
        validateOpen();
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount.compareTo(balance) > 0) {
            throw new IllegalStateException(
                "Insufficient funds. Balance: " + balance + ", Requested: " + amount
            );
        }

        BigDecimal newBalance = balance.subtract(amount);
        applyEvent(new MoneyWithdrawnEvent(id, nextVersion(), amount, description, newBalance));
    }

    public void initiateTransfer(String targetAccountId, BigDecimal amount) {
        validateOpen();
        if (amount.compareTo(balance) > 0) {
            throw new IllegalStateException("Insufficient funds for transfer");
        }

        String transferId = UUID.randomUUID().toString();
        // Deduct from source
        BigDecimal newBalance = balance.subtract(amount);
        applyEvent(new TransferInitiatedEvent(id, nextVersion(), targetAccountId, amount, transferId));
        applyEvent(new MoneyWithdrawnEvent(id, nextVersion(), amount,
            "Transfer to " + targetAccountId, newBalance));
    }

    public void close(String reason) {
        if (status == AccountStatus.CLOSED) {
            throw new IllegalStateException("Account is already closed");
        }
        if (balance.compareTo(BigDecimal.ZERO) > 0) {
            throw new IllegalStateException("Cannot close account with positive balance");
        }

        applyEvent(new AccountClosedEvent(id, nextVersion(), reason));
    }

    // ==================== Event Handlers ====================
    // Event handlers change state (no validation here - events already happened)

    private void on(AccountOpenedEvent event) {
        this.id = event.getAggregateId();
        this.ownerName = event.getOwnerName();
        this.ownerEmail = event.getOwnerEmail();
        this.balance = event.getInitialDeposit();
        this.status = AccountStatus.OPEN;
    }

    private void on(MoneyDepositedEvent event) {
        this.balance = event.getBalanceAfter();
    }

    private void on(MoneyWithdrawnEvent event) {
        this.balance = event.getBalanceAfter();
    }

    private void on(TransferInitiatedEvent event) {
        // State change handled by MoneyWithdrawnEvent
    }

    private void on(AccountClosedEvent event) {
        this.status = AccountStatus.CLOSED;
    }

    // ==================== Infrastructure ====================

    // Apply event: update state + add to pending events
    private void applyEvent(DomainEvent event) {
        handleEvent(event);
        pendingEvents.add(event);
        this.version++;
    }

    // Replay event (from EventStore - no adding to pending)
    public void replayEvent(DomainEvent event) {
        handleEvent(event);
        this.version++;
    }

    private void handleEvent(DomainEvent event) {
        if (event instanceof AccountOpenedEvent e) on(e);
        else if (event instanceof MoneyDepositedEvent e) on(e);
        else if (event instanceof MoneyWithdrawnEvent e) on(e);
        else if (event instanceof TransferInitiatedEvent e) on(e);
        else if (event instanceof AccountClosedEvent e) on(e);
        else throw new IllegalArgumentException("Unknown event type: " + event.getEventType());
    }

    // Reconstitute aggregate from events (replay)
    public static BankAccountAggregate reconstitute(List<DomainEvent> events) {
        if (events.isEmpty()) {
            throw new IllegalArgumentException("Cannot reconstitute from empty event list");
        }

        BankAccountAggregate account = new BankAccountAggregate();
        events.forEach(account::replayEvent);
        return account;
    }

    private void validateOpen() {
        if (status != AccountStatus.OPEN) {
            throw new IllegalStateException("Account is not open. Status: " + status);
        }
    }

    private long nextVersion() {
        return version + 1;
    }

    // Getters
    public String getId() { return id; }
    public String getOwnerName() { return ownerName; }
    public String getOwnerEmail() { return ownerEmail; }
    public BigDecimal getBalance() { return balance; }
    public AccountStatus getStatus() { return status; }
    public long getVersion() { return version; }
    public List<DomainEvent> getPendingEvents() { return Collections.unmodifiableList(pendingEvents); }
    public void clearPendingEvents() { pendingEvents.clear(); }
}
```

---

## 5. EventStore Implementation

### 5.1 Event Store Entity

```java
package com.example.eventsourcing.store;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(
    name = "event_store",
    indexes = {
        @Index(name = "idx_event_store_aggregate",
               columnList = "aggregate_id, sequence_number",
               unique = true),
        @Index(name = "idx_event_store_type",
               columnList = "aggregate_type, aggregate_id"),
        @Index(name = "idx_event_store_occurred",
               columnList = "occurred_at")
    }
)
public class StoredEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 36)
    private String eventId;

    @Column(name = "aggregate_id", nullable = false, length = 36)
    private String aggregateId;

    @Column(name = "aggregate_type", nullable = false, length = 100)
    private String aggregateType;

    @Column(name = "sequence_number", nullable = false)
    private long sequenceNumber;

    @Column(name = "event_type", nullable = false, length = 200)
    private String eventType;

    @Column(name = "event_data", nullable = false, columnDefinition = "TEXT")
    private String eventData;  // JSON serialized event

    @Column(name = "metadata", columnDefinition = "TEXT")
    private String metadata;  // Optional: correlation ID, causation ID, user ID

    @Column(name = "occurred_at", nullable = false)
    private Instant occurredAt;

    public StoredEvent() {}

    public StoredEvent(String eventId, String aggregateId, String aggregateType,
                       long sequenceNumber, String eventType,
                       String eventData, Instant occurredAt) {
        this.eventId = eventId;
        this.aggregateId = aggregateId;
        this.aggregateType = aggregateType;
        this.sequenceNumber = sequenceNumber;
        this.eventType = eventType;
        this.eventData = eventData;
        this.occurredAt = occurredAt;
    }

    // Getters
    public Long getId() { return id; }
    public String getEventId() { return eventId; }
    public String getAggregateId() { return aggregateId; }
    public String getAggregateType() { return aggregateType; }
    public long getSequenceNumber() { return sequenceNumber; }
    public String getEventType() { return eventType; }
    public String getEventData() { return eventData; }
    public String getMetadata() { return metadata; }
    public void setMetadata(String metadata) { this.metadata = metadata; }
    public Instant getOccurredAt() { return occurredAt; }
}
```

### 5.2 Event Store Repository

```java
package com.example.eventsourcing.store;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

@Repository
public interface EventStoreRepository extends JpaRepository<StoredEvent, Long> {

    List<StoredEvent> findByAggregateIdOrderBySequenceNumberAsc(String aggregateId);

    List<StoredEvent> findByAggregateIdAndSequenceNumberGreaterThanOrderBySequenceNumberAsc(
        String aggregateId, long afterVersion
    );

    Optional<StoredEvent> findTopByAggregateIdOrderBySequenceNumberDesc(String aggregateId);

    boolean existsByAggregateId(String aggregateId);

    @Query("""
        SELECT e FROM StoredEvent e
        WHERE e.aggregateType = :type
          AND e.occurredAt BETWEEN :from AND :to
        ORDER BY e.occurredAt ASC
        """)
    List<StoredEvent> findByAggregateTypeAndTimeRange(
        @Param("type") String type,
        @Param("from") Instant from,
        @Param("to") Instant to
    );

    @Query("SELECT COUNT(e) FROM StoredEvent e WHERE e.aggregateId = :id")
    long countByAggregateId(@Param("id") String aggregateId);

    // For global event ordering (projections)
    List<StoredEvent> findByIdGreaterThanOrderByIdAsc(Long afterId);
}
```

### 5.3 Event Store Service

```java
package com.example.eventsourcing.store;

import com.example.eventsourcing.event.DomainEvent;
import com.example.eventsourcing.event.account.*;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Map;

@Service
public class EventStore {

    private static final Logger log = LoggerFactory.getLogger(EventStore.class);

    @Autowired
    private EventStoreRepository repository;

    private final ObjectMapper objectMapper;

    // Event type registry for deserialization
    private static final Map<String, Class<? extends DomainEvent>> EVENT_TYPES = Map.of(
        "AccountOpenedEvent", AccountOpenedEvent.class,
        "MoneyDepositedEvent", MoneyDepositedEvent.class,
        "MoneyWithdrawnEvent", MoneyWithdrawnEvent.class,
        "TransferInitiatedEvent", TransferInitiatedEvent.class,
        "AccountClosedEvent", AccountClosedEvent.class
    );

    public EventStore() {
        this.objectMapper = new ObjectMapper();
        this.objectMapper.registerModule(new JavaTimeModule());
        this.objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Transactional
    public void appendEvents(
            String aggregateId,
            long expectedVersion,
            List<DomainEvent> events
    ) {
        // Optimistic concurrency: check current version
        long currentVersion = getCurrentVersion(aggregateId);

        if (currentVersion != expectedVersion) {
            throw new OptimisticConcurrencyException(
                String.format(
                    "Concurrency conflict on aggregate %s: expected version %d, got %d",
                    aggregateId, expectedVersion, currentVersion
                )
            );
        }

        events.forEach(event -> {
            StoredEvent stored = new StoredEvent(
                event.getEventId(),
                event.getAggregateId(),
                event.getAggregateType(),
                event.getSequenceNumber(),
                event.getEventType(),
                serialize(event),
                event.getOccurredAt()
            );

            repository.save(stored);
            log.debug("Stored event: {} for aggregate: {}", event.getEventType(), aggregateId);
        });
    }

    @Transactional(readOnly = true)
    public List<DomainEvent> loadEvents(String aggregateId) {
        return repository.findByAggregateIdOrderBySequenceNumberAsc(aggregateId)
            .stream()
            .map(this::deserialize)
            .toList();
    }

    @Transactional(readOnly = true)
    public List<DomainEvent> loadEventsSince(String aggregateId, long afterVersion) {
        return repository
            .findByAggregateIdAndSequenceNumberGreaterThanOrderBySequenceNumberAsc(
                aggregateId, afterVersion
            )
            .stream()
            .map(this::deserialize)
            .toList();
    }

    public long getCurrentVersion(String aggregateId) {
        return repository.findTopByAggregateIdOrderBySequenceNumberDesc(aggregateId)
            .map(StoredEvent::getSequenceNumber)
            .orElse(0L);
    }

    public boolean exists(String aggregateId) {
        return repository.existsByAggregateId(aggregateId);
    }

    private String serialize(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize event: " + event.getEventType(), e);
        }
    }

    private DomainEvent deserialize(StoredEvent stored) {
        Class<? extends DomainEvent> eventClass = EVENT_TYPES.get(stored.getEventType());
        if (eventClass == null) {
            throw new IllegalArgumentException("Unknown event type: " + stored.getEventType());
        }

        try {
            return objectMapper.readValue(stored.getEventData(), eventClass);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(
                "Failed to deserialize event: " + stored.getEventType(), e
            );
        }
    }
}
```

```java
package com.example.eventsourcing.store;

public class OptimisticConcurrencyException extends RuntimeException {
    public OptimisticConcurrencyException(String message) {
        super(message);
    }
}
```

---

## 6. Aggregate Repository

```java
package com.example.eventsourcing.repository;

import com.example.eventsourcing.aggregate.BankAccountAggregate;
import com.example.eventsourcing.event.DomainEvent;
import com.example.eventsourcing.store.EventStore;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Repository
public class BankAccountRepository {

    @Autowired
    private EventStore eventStore;

    @Transactional
    public void save(BankAccountAggregate account) {
        List<DomainEvent> pendingEvents = account.getPendingEvents();
        if (pendingEvents.isEmpty()) return;

        long expectedVersion = account.getVersion() - pendingEvents.size();

        eventStore.appendEvents(account.getId(), expectedVersion, pendingEvents);
        account.clearPendingEvents();
    }

    @Transactional(readOnly = true)
    public Optional<BankAccountAggregate> findById(String accountId) {
        if (!eventStore.exists(accountId)) {
            return Optional.empty();
        }

        List<DomainEvent> events = eventStore.loadEvents(accountId);
        if (events.isEmpty()) return Optional.empty();

        return Optional.of(BankAccountAggregate.reconstitute(events));
    }

    public BankAccountAggregate getById(String accountId) {
        return findById(accountId)
            .orElseThrow(() -> new IllegalArgumentException(
                "Account not found: " + accountId
            ));
    }
}
```

---

## 7. Commands

```java
package com.example.eventsourcing.command;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import java.math.BigDecimal;

public sealed interface AccountCommand
    permits AccountCommand.OpenAccount,
            AccountCommand.Deposit,
            AccountCommand.Withdraw,
            AccountCommand.Transfer,
            AccountCommand.CloseAccount {

    record OpenAccount(
        @NotBlank String ownerName,
        @Email @NotBlank String ownerEmail,
        @NotNull @DecimalMin("0.00") BigDecimal initialDeposit
    ) implements AccountCommand {}

    record Deposit(
        @NotBlank String accountId,
        @NotNull @DecimalMin("0.01") BigDecimal amount,
        String description
    ) implements AccountCommand {}

    record Withdraw(
        @NotBlank String accountId,
        @NotNull @DecimalMin("0.01") BigDecimal amount,
        String description
    ) implements AccountCommand {}

    record Transfer(
        @NotBlank String sourceAccountId,
        @NotBlank String targetAccountId,
        @NotNull @DecimalMin("0.01") BigDecimal amount
    ) implements AccountCommand {}

    record CloseAccount(
        @NotBlank String accountId,
        @NotBlank String reason
    ) implements AccountCommand {}
}
```

---

## 8. Command Handler (Write Side)

```java
package com.example.eventsourcing.handler;

import com.example.eventsourcing.aggregate.BankAccountAggregate;
import com.example.eventsourcing.command.AccountCommand;
import com.example.eventsourcing.projection.AccountProjectionUpdater;
import com.example.eventsourcing.repository.BankAccountRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BankAccountCommandHandler {

    private static final Logger log = LoggerFactory.getLogger(BankAccountCommandHandler.class);

    @Autowired
    private BankAccountRepository accountRepository;

    @Autowired
    private AccountProjectionUpdater projectionUpdater;

    @Transactional
    public String handle(AccountCommand.OpenAccount command) {
        log.info("Opening account for: {}", command.ownerName());

        BankAccountAggregate account = BankAccountAggregate.open(
            command.ownerName(),
            command.ownerEmail(),
            command.initialDeposit()
        );

        accountRepository.save(account);
        projectionUpdater.updateFromEvents(account.getId());

        log.info("Account opened: {}", account.getId());
        return account.getId();
    }

    @Transactional
    public void handle(AccountCommand.Deposit command) {
        log.info("Depositing {} to account: {}", command.amount(), command.accountId());

        BankAccountAggregate account = accountRepository.getById(command.accountId());
        account.deposit(command.amount(), command.description());

        accountRepository.save(account);
        projectionUpdater.updateFromEvents(command.accountId());
    }

    @Transactional
    public void handle(AccountCommand.Withdraw command) {
        log.info("Withdrawing {} from account: {}", command.amount(), command.accountId());

        BankAccountAggregate account = accountRepository.getById(command.accountId());
        account.withdraw(command.amount(), command.description());

        accountRepository.save(account);
        projectionUpdater.updateFromEvents(command.accountId());
    }

    @Transactional
    public void handle(AccountCommand.Transfer command) {
        log.info("Transferring {} from {} to {}",
            command.amount(), command.sourceAccountId(), command.targetAccountId());

        BankAccountAggregate source = accountRepository.getById(command.sourceAccountId());
        BankAccountAggregate target = accountRepository.getById(command.targetAccountId());

        source.initiateTransfer(command.targetAccountId(), command.amount());
        target.deposit(command.amount(), "Transfer from " + command.sourceAccountId());

        accountRepository.save(source);
        accountRepository.save(target);

        projectionUpdater.updateFromEvents(command.sourceAccountId());
        projectionUpdater.updateFromEvents(command.targetAccountId());
    }

    @Transactional
    public void handle(AccountCommand.CloseAccount command) {
        log.info("Closing account: {}", command.accountId());

        BankAccountAggregate account = accountRepository.getById(command.accountId());
        account.close(command.reason());

        accountRepository.save(account);
        projectionUpdater.updateFromEvents(command.accountId());
    }
}
```

---

## 9. Projections (Read Models)

### 9.1 Account Summary Projection

```java
package com.example.eventsourcing.readmodel;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "account_summaries")
public class AccountSummary {

    @Id
    private String accountId;

    private String ownerName;
    private String ownerEmail;

    @Column(precision = 20, scale = 2)
    private BigDecimal balance;

    @Enumerated(EnumType.STRING)
    private AccountStatus status;

    private int totalTransactions;
    private Instant openedAt;
    private Instant lastTransactionAt;
    private long version;

    public enum AccountStatus { OPEN, CLOSED, SUSPENDED }

    public AccountSummary() {}

    // Getters and setters
    public String getAccountId() { return accountId; }
    public void setAccountId(String accountId) { this.accountId = accountId; }
    public String getOwnerName() { return ownerName; }
    public void setOwnerName(String ownerName) { this.ownerName = ownerName; }
    public String getOwnerEmail() { return ownerEmail; }
    public void setOwnerEmail(String ownerEmail) { this.ownerEmail = ownerEmail; }
    public BigDecimal getBalance() { return balance; }
    public void setBalance(BigDecimal balance) { this.balance = balance; }
    public AccountStatus getStatus() { return status; }
    public void setStatus(AccountStatus status) { this.status = status; }
    public int getTotalTransactions() { return totalTransactions; }
    public void setTotalTransactions(int count) { this.totalTransactions = count; }
    public Instant getOpenedAt() { return openedAt; }
    public void setOpenedAt(Instant openedAt) { this.openedAt = openedAt; }
    public Instant getLastTransactionAt() { return lastTransactionAt; }
    public void setLastTransactionAt(Instant ts) { this.lastTransactionAt = ts; }
    public long getVersion() { return version; }
    public void setVersion(long version) { this.version = version; }
}
```

```java
package com.example.eventsourcing.readmodel;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "transaction_history",
    indexes = {
        @Index(name = "idx_txn_account", columnList = "account_id"),
        @Index(name = "idx_txn_occurred", columnList = "occurred_at")
    }
)
public class TransactionEntry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "account_id", nullable = false)
    private String accountId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TransactionType type;

    @Column(precision = 15, scale = 2, nullable = false)
    private BigDecimal amount;

    @Column(precision = 15, scale = 2, nullable = false)
    private BigDecimal balanceAfter;

    private String description;
    private String counterpartAccountId;

    @Column(nullable = false)
    private Instant occurredAt;

    public enum TransactionType { DEPOSIT, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT }

    public TransactionEntry() {}

    // Getters and setters
    public Long getId() { return id; }
    public String getAccountId() { return accountId; }
    public void setAccountId(String accountId) { this.accountId = accountId; }
    public TransactionType getType() { return type; }
    public void setType(TransactionType type) { this.type = type; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    public BigDecimal getBalanceAfter() { return balanceAfter; }
    public void setBalanceAfter(BigDecimal balanceAfter) { this.balanceAfter = balanceAfter; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public String getCounterpartAccountId() { return counterpartAccountId; }
    public void setCounterpartAccountId(String id) { this.counterpartAccountId = id; }
    public Instant getOccurredAt() { return occurredAt; }
    public void setOccurredAt(Instant occurredAt) { this.occurredAt = occurredAt; }
}
```

### 9.2 Projection Updater

```java
package com.example.eventsourcing.projection;

import com.example.eventsourcing.event.DomainEvent;
import com.example.eventsourcing.event.account.*;
import com.example.eventsourcing.readmodel.AccountSummary;
import com.example.eventsourcing.readmodel.TransactionEntry;
import com.example.eventsourcing.store.EventStore;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
public class AccountProjectionUpdater {

    private static final Logger log = LoggerFactory.getLogger(AccountProjectionUpdater.class);

    @Autowired
    private EventStore eventStore;

    @Autowired
    private AccountSummaryRepository summaryRepository;

    @Autowired
    private TransactionEntryRepository transactionRepository;

    @Transactional
    public void updateFromEvents(String accountId) {
        List<DomainEvent> events = eventStore.loadEvents(accountId);
        rebuildProjection(accountId, events);
    }

    private void rebuildProjection(String accountId, List<DomainEvent> events) {
        AccountSummary summary = summaryRepository.findById(accountId)
            .orElse(new AccountSummary());

        for (DomainEvent event : events) {
            switch (event) {
                case AccountOpenedEvent e -> {
                    summary.setAccountId(e.getAggregateId());
                    summary.setOwnerName(e.getOwnerName());
                    summary.setOwnerEmail(e.getOwnerEmail());
                    summary.setBalance(e.getInitialDeposit());
                    summary.setStatus(AccountSummary.AccountStatus.OPEN);
                    summary.setOpenedAt(e.getOccurredAt());
                    summary.setTotalTransactions(0);
                    summary.setVersion(e.getSequenceNumber());
                }
                case MoneyDepositedEvent e -> {
                    summary.setBalance(e.getBalanceAfter());
                    summary.setTotalTransactions(summary.getTotalTransactions() + 1);
                    summary.setLastTransactionAt(e.getOccurredAt());
                    summary.setVersion(e.getSequenceNumber());

                    TransactionEntry txn = new TransactionEntry();
                    txn.setAccountId(accountId);
                    txn.setType(TransactionEntry.TransactionType.DEPOSIT);
                    txn.setAmount(e.getAmount());
                    txn.setBalanceAfter(e.getBalanceAfter());
                    txn.setDescription(e.getDescription());
                    txn.setOccurredAt(e.getOccurredAt());
                    transactionRepository.save(txn);
                }
                case MoneyWithdrawnEvent e -> {
                    summary.setBalance(e.getBalanceAfter());
                    summary.setTotalTransactions(summary.getTotalTransactions() + 1);
                    summary.setLastTransactionAt(e.getOccurredAt());
                    summary.setVersion(e.getSequenceNumber());

                    TransactionEntry txn = new TransactionEntry();
                    txn.setAccountId(accountId);
                    txn.setType(TransactionEntry.TransactionType.WITHDRAWAL);
                    txn.setAmount(e.getAmount());
                    txn.setBalanceAfter(e.getBalanceAfter());
                    txn.setDescription(e.getDescription());
                    txn.setOccurredAt(e.getOccurredAt());
                    transactionRepository.save(txn);
                }
                case AccountClosedEvent e -> {
                    summary.setStatus(AccountSummary.AccountStatus.CLOSED);
                    summary.setVersion(e.getSequenceNumber());
                }
                default -> log.debug("Skipping event in projection: {}", event.getEventType());
            }
        }

        summaryRepository.save(summary);
        log.debug("Projection updated for account: {}", accountId);
    }
}

interface AccountSummaryRepository extends JpaRepository<AccountSummary, String> {}
interface TransactionEntryRepository extends JpaRepository<TransactionEntry, Long> {
    List<TransactionEntry> findByAccountIdOrderByOccurredAtDesc(String accountId);
}
```

---

## 10. Query Side (Read)

```java
package com.example.eventsourcing.query;

import com.example.eventsourcing.readmodel.AccountSummary;
import com.example.eventsourcing.readmodel.TransactionEntry;
import com.example.eventsourcing.projection.AccountSummaryRepository;
import com.example.eventsourcing.projection.TransactionEntryRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.PageRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

@Service
public class AccountQueryService {

    @Autowired
    private AccountSummaryRepository summaryRepository;

    @Autowired
    private TransactionEntryRepository transactionRepository;

    @Transactional(readOnly = true)
    public Optional<AccountSummary> getAccountSummary(String accountId) {
        return summaryRepository.findById(accountId);
    }

    @Transactional(readOnly = true)
    public List<TransactionEntry> getTransactionHistory(String accountId, int limit) {
        return transactionRepository.findByAccountIdOrderByOccurredAtDesc(accountId)
            .stream()
            .limit(limit)
            .toList();
    }

    @Transactional(readOnly = true)
    public List<AccountSummary> getAllAccounts() {
        return summaryRepository.findAll();
    }
}
```

---

## 11. REST Controller (CQRS Endpoints)

```java
package com.example.eventsourcing.controller;

import com.example.eventsourcing.command.AccountCommand;
import com.example.eventsourcing.handler.BankAccountCommandHandler;
import com.example.eventsourcing.query.AccountQueryService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.net.URI;
import java.util.Map;

@RestController
@RequestMapping("/api/accounts")
public class BankAccountController {

    @Autowired
    private BankAccountCommandHandler commandHandler;

    @Autowired
    private AccountQueryService queryService;

    // ===== COMMAND endpoints (Write side) =====

    @PostMapping
    public ResponseEntity<?> openAccount(@Valid @RequestBody AccountCommand.OpenAccount command) {
        String accountId = commandHandler.handle(command);
        return ResponseEntity.created(URI.create("/api/accounts/" + accountId))
            .body(Map.of("accountId", accountId));
    }

    @PostMapping("/{accountId}/deposit")
    public ResponseEntity<?> deposit(
            @PathVariable String accountId,
            @RequestParam BigDecimal amount,
            @RequestParam(defaultValue = "Deposit") String description
    ) {
        commandHandler.handle(new AccountCommand.Deposit(accountId, amount, description));
        return ResponseEntity.ok(Map.of("status", "success"));
    }

    @PostMapping("/{accountId}/withdraw")
    public ResponseEntity<?> withdraw(
            @PathVariable String accountId,
            @RequestParam BigDecimal amount,
            @RequestParam(defaultValue = "Withdrawal") String description
    ) {
        commandHandler.handle(new AccountCommand.Withdraw(accountId, amount, description));
        return ResponseEntity.ok(Map.of("status", "success"));
    }

    @PostMapping("/{accountId}/transfer")
    public ResponseEntity<?> transfer(
            @PathVariable String accountId,
            @RequestParam String targetAccountId,
            @RequestParam BigDecimal amount
    ) {
        commandHandler.handle(new AccountCommand.Transfer(accountId, targetAccountId, amount));
        return ResponseEntity.ok(Map.of("status", "success"));
    }

    @DeleteMapping("/{accountId}")
    public ResponseEntity<?> closeAccount(
            @PathVariable String accountId,
            @RequestParam String reason
    ) {
        commandHandler.handle(new AccountCommand.CloseAccount(accountId, reason));
        return ResponseEntity.ok(Map.of("status", "closed"));
    }

    // ===== QUERY endpoints (Read side) =====

    @GetMapping("/{accountId}")
    public ResponseEntity<?> getAccount(@PathVariable String accountId) {
        return queryService.getAccountSummary(accountId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/{accountId}/transactions")
    public ResponseEntity<?> getTransactions(
            @PathVariable String accountId,
            @RequestParam(defaultValue = "20") int limit
    ) {
        return ResponseEntity.ok(
            queryService.getTransactionHistory(accountId, limit)
        );
    }

    @GetMapping
    public ResponseEntity<?> getAllAccounts() {
        return ResponseEntity.ok(queryService.getAllAccounts());
    }

    // Temporal query: what was the state at a point in time?
    @GetMapping("/{accountId}/state-at")
    public ResponseEntity<?> getStateAt(
            @PathVariable String accountId,
            @RequestParam String timestamp
    ) {
        // Replay events up to the given timestamp
        // (Implementation: filter events by occurredAt <= timestamp)
        return ResponseEntity.ok(Map.of(
            "accountId", accountId,
            "note", "Temporal query - replay events up to: " + timestamp
        ));
    }
}
```

---

## 12. Saga Pattern for Distributed Transactions

```java
package com.example.eventsourcing.saga;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

// Saga coordinates a multi-step distributed transaction
// Each step can be compensated (reversed) if a later step fails
public class TransferSaga {

    private final String sagaId;
    private final String sourceAccountId;
    private final String targetAccountId;
    private final BigDecimal amount;
    private SagaState state;
    private Instant startedAt;
    private String failureReason;

    public enum SagaState {
        STARTED,
        SOURCE_DEBITED,        // Money taken from source
        TARGET_CREDITED,       // Money added to target
        COMPLETED,             // Success
        COMPENSATING,          // Rollback in progress
        COMPENSATED,           // Rolled back successfully
        FAILED                 // Permanent failure
    }

    public TransferSaga(String sourceAccountId, String targetAccountId, BigDecimal amount) {
        this.sagaId = UUID.randomUUID().toString();
        this.sourceAccountId = sourceAccountId;
        this.targetAccountId = targetAccountId;
        this.amount = amount;
        this.state = SagaState.STARTED;
        this.startedAt = Instant.now();
    }

    // Saga steps
    public void onSourceDebited() {
        if (state != SagaState.STARTED) {
            throw new IllegalStateException("Invalid state transition: " + state);
        }
        state = SagaState.SOURCE_DEBITED;
    }

    public void onTargetCredited() {
        if (state != SagaState.SOURCE_DEBITED) {
            throw new IllegalStateException("Invalid state transition: " + state);
        }
        state = SagaState.TARGET_CREDITED;
    }

    public void onCompleted() {
        state = SagaState.COMPLETED;
    }

    // Compensation
    public void onTargetCreditFailed(String reason) {
        this.failureReason = reason;
        state = SagaState.COMPENSATING;
        // Must refund source account
    }

    public void onSourceRefunded() {
        if (state != SagaState.COMPENSATING) {
            throw new IllegalStateException("Not in compensating state");
        }
        state = SagaState.COMPENSATED;
    }

    public String getSagaId() { return sagaId; }
    public String getSourceAccountId() { return sourceAccountId; }
    public String getTargetAccountId() { return targetAccountId; }
    public BigDecimal getAmount() { return amount; }
    public SagaState getState() { return state; }
    public Instant getStartedAt() { return startedAt; }
    public String getFailureReason() { return failureReason; }
}
```

```java
package com.example.eventsourcing.saga;

import com.example.eventsourcing.aggregate.BankAccountAggregate;
import com.example.eventsourcing.command.AccountCommand;
import com.example.eventsourcing.handler.BankAccountCommandHandler;
import com.example.eventsourcing.repository.BankAccountRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

@Service
public class TransferSagaOrchestrator {

    private static final Logger log = LoggerFactory.getLogger(TransferSagaOrchestrator.class);

    @Autowired
    private BankAccountRepository accountRepository;

    @Autowired
    private BankAccountCommandHandler commandHandler;

    @Transactional
    public TransferSaga executeTransfer(
            String sourceId, String targetId, BigDecimal amount
    ) {
        TransferSaga saga = new TransferSaga(sourceId, targetId, amount);
        log.info("Starting transfer saga: {}", saga.getSagaId());

        try {
            // Step 1: Debit source
            BankAccountAggregate source = accountRepository.getById(sourceId);
            source.withdraw(amount, "Transfer saga: " + saga.getSagaId());
            accountRepository.save(source);
            saga.onSourceDebited();
            log.info("Saga {}: Source debited", saga.getSagaId());

            // Step 2: Credit target
            try {
                BankAccountAggregate target = accountRepository.getById(targetId);
                target.deposit(amount, "Transfer saga: " + saga.getSagaId());
                accountRepository.save(target);
                saga.onTargetCredited();
                log.info("Saga {}: Target credited", saga.getSagaId());

            } catch (Exception e) {
                log.error("Saga {}: Target credit failed, compensating...", saga.getSagaId());
                saga.onTargetCreditFailed(e.getMessage());

                // Compensating transaction: refund source
                BankAccountAggregate sourceRefund = accountRepository.getById(sourceId);
                sourceRefund.deposit(amount, "Refund saga: " + saga.getSagaId());
                accountRepository.save(sourceRefund);
                saga.onSourceRefunded();

                log.info("Saga {}: Compensated - source refunded", saga.getSagaId());
                return saga;
            }

            saga.onCompleted();
            log.info("Saga {}: Transfer completed successfully", saga.getSagaId());
            return saga;

        } catch (Exception e) {
            log.error("Saga {}: Failed - {}", saga.getSagaId(), e.getMessage());
            throw e;
        }
    }
}
```

---

## 13. Testing Event-Sourced Aggregates

```java
package com.example.eventsourcing.test;

import com.example.eventsourcing.aggregate.BankAccountAggregate;
import com.example.eventsourcing.event.DomainEvent;
import com.example.eventsourcing.event.account.AccountOpenedEvent;
import com.example.eventsourcing.event.account.MoneyDepositedEvent;
import com.example.eventsourcing.event.account.MoneyWithdrawnEvent;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.*;

class BankAccountAggregateTest {

    @Test
    void shouldOpenAccountWithInitialDeposit() {
        BankAccountAggregate account = BankAccountAggregate.open(
            "John Doe", "john@example.com", new BigDecimal("1000.00")
        );

        assertThat(account.getOwnerName()).isEqualTo("John Doe");
        assertThat(account.getOwnerEmail()).isEqualTo("john@example.com");
        assertThat(account.getBalance()).isEqualByComparingTo("1000.00");
        assertThat(account.getStatus()).isEqualTo(BankAccountAggregate.AccountStatus.OPEN);
        assertThat(account.getVersion()).isEqualTo(1);

        List<DomainEvent> events = account.getPendingEvents();
        assertThat(events).hasSize(1);
        assertThat(events.get(0)).isInstanceOf(AccountOpenedEvent.class);
    }

    @Test
    void shouldDepositMoney() {
        BankAccountAggregate account = BankAccountAggregate.open(
            "John Doe", "john@example.com", new BigDecimal("1000.00")
        );
        account.clearPendingEvents();

        account.deposit(new BigDecimal("500.00"), "Paycheck");

        assertThat(account.getBalance()).isEqualByComparingTo("1500.00");
        assertThat(account.getPendingEvents()).hasSize(1);
        assertThat(account.getPendingEvents().get(0)).isInstanceOf(MoneyDepositedEvent.class);
    }

    @Test
    void shouldRejectWithdrawalExceedingBalance() {
        BankAccountAggregate account = BankAccountAggregate.open(
            "John Doe", "john@example.com", new BigDecimal("100.00")
        );

        assertThatThrownBy(() ->
            account.withdraw(new BigDecimal("200.00"), "Test")
        )
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("Insufficient funds");
    }

    @Test
    void shouldReconstitutStateFromEvents() {
        // Build events directly
        AccountOpenedEvent opened = new AccountOpenedEvent(
            "acc-001", 1, "Jane Doe", "jane@example.com", new BigDecimal("500.00")
        );
        MoneyDepositedEvent deposited = new MoneyDepositedEvent(
            "acc-001", 2, new BigDecimal("200.00"), "Bonus", new BigDecimal("700.00")
        );
        MoneyWithdrawnEvent withdrawn = new MoneyWithdrawnEvent(
            "acc-001", 3, new BigDecimal("100.00"), "Bills", new BigDecimal("600.00")
        );

        // Reconstitute aggregate from events
        BankAccountAggregate account = BankAccountAggregate.reconstitute(
            List.of(opened, deposited, withdrawn)
        );

        assertThat(account.getId()).isEqualTo("acc-001");
        assertThat(account.getOwnerName()).isEqualTo("Jane Doe");
        assertThat(account.getBalance()).isEqualByComparingTo("600.00");
        assertThat(account.getVersion()).isEqualTo(3);
        assertThat(account.getPendingEvents()).isEmpty(); // No new events
    }

    @Test
    void shouldPreventClosingAccountWithPositiveBalance() {
        BankAccountAggregate account = BankAccountAggregate.open(
            "John Doe", "john@example.com", new BigDecimal("100.00")
        );

        assertThatThrownBy(() ->
            account.close("Test closure")
        )
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("Cannot close account with positive balance");
    }

    @Test
    void shouldCloseAccountWithZeroBalance() {
        BankAccountAggregate account = BankAccountAggregate.open(
            "John Doe", "john@example.com", BigDecimal.ZERO
        );
        account.close("Customer request");

        assertThat(account.getStatus()).isEqualTo(BankAccountAggregate.AccountStatus.CLOSED);
        assertThat(account.getPendingEvents()).hasSize(2); // Opened + Closed
    }

    @Test
    void shouldMaintainEventSequenceNumbers() {
        BankAccountAggregate account = BankAccountAggregate.open(
            "John Doe", "john@example.com", new BigDecimal("1000.00")
        );
        account.deposit(new BigDecimal("500.00"), "dep1");
        account.deposit(new BigDecimal("300.00"), "dep2");
        account.withdraw(new BigDecimal("200.00"), "with1");

        List<DomainEvent> events = account.getPendingEvents();
        assertThat(events).hasSize(4);

        // Sequence numbers should be consecutive
        for (int i = 0; i < events.size(); i++) {
            assertThat(events.get(i).getSequenceNumber()).isEqualTo(i + 1);
        }
    }
}
```

---

## 14. Snapshot Pattern (Performance Optimization)

```java
package com.example.eventsourcing.snapshot;

import com.example.eventsourcing.aggregate.BankAccountAggregate;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "aggregate_snapshots")
public class AggregateSnapshot {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String aggregateType;

    @Column(nullable = false)
    private long version;  // The event version at snapshot time

    @Column(columnDefinition = "TEXT", nullable = false)
    private String snapshotData;  // JSON state

    @Column(nullable = false)
    private Instant createdAt;

    public AggregateSnapshot() {}

    public AggregateSnapshot(String aggregateId, String aggregateType,
                              long version, String snapshotData) {
        this.aggregateId = aggregateId;
        this.aggregateType = aggregateType;
        this.version = version;
        this.snapshotData = snapshotData;
        this.createdAt = Instant.now();
    }

    public Long getId() { return id; }
    public String getAggregateId() { return aggregateId; }
    public String getAggregateType() { return aggregateType; }
    public long getVersion() { return version; }
    public String getSnapshotData() { return snapshotData; }
    public Instant getCreatedAt() { return createdAt; }
}
```

```java
package com.example.eventsourcing.snapshot;

import com.example.eventsourcing.aggregate.BankAccountAggregate;
import com.example.eventsourcing.event.DomainEvent;
import com.example.eventsourcing.store.EventStore;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

@Repository
interface SnapshotRepository extends JpaRepository<AggregateSnapshot, Long> {
    Optional<AggregateSnapshot> findTopByAggregateIdOrderByVersionDesc(String aggregateId);
}

@Service
class SnapshotService {

    private static final int SNAPSHOT_THRESHOLD = 50; // Snapshot every 50 events

    @Autowired
    private SnapshotRepository snapshotRepository;

    @Autowired
    private EventStore eventStore;

    @Autowired
    private ObjectMapper objectMapper;

    public BankAccountAggregate loadWithSnapshot(String accountId) {
        Optional<AggregateSnapshot> snapshot = snapshotRepository
            .findTopByAggregateIdOrderByVersionDesc(accountId);

        if (snapshot.isPresent()) {
            AggregateSnapshot snap = snapshot.get();
            // Load only events AFTER the snapshot version
            List<DomainEvent> events = eventStore.loadEventsSince(accountId, snap.getVersion());

            // Reconstitute from snapshot + remaining events
            BankAccountAggregate account = deserializeSnapshot(snap);
            events.forEach(account::replayEvent);
            return account;
        }

        // No snapshot: load all events
        List<DomainEvent> events = eventStore.loadEvents(accountId);
        return BankAccountAggregate.reconstitute(events);
    }

    public void createSnapshotIfNeeded(BankAccountAggregate account) {
        long eventCount = eventStore.getCurrentVersion(account.getId());
        Optional<AggregateSnapshot> existing = snapshotRepository
            .findTopByAggregateIdOrderByVersionDesc(account.getId());

        long snapshotVersion = existing.map(AggregateSnapshot::getVersion).orElse(0L);
        long eventsSinceSnapshot = eventCount - snapshotVersion;

        if (eventsSinceSnapshot >= SNAPSHOT_THRESHOLD) {
            saveSnapshot(account);
        }
    }

    private void saveSnapshot(BankAccountAggregate account) {
        try {
            // Simple snapshot state
            var state = new java.util.HashMap<String, Object>();
            state.put("id", account.getId());
            state.put("ownerName", account.getOwnerName());
            state.put("ownerEmail", account.getOwnerEmail());
            state.put("balance", account.getBalance().toPlainString());
            state.put("status", account.getStatus().name());

            String json = objectMapper.writeValueAsString(state);
            snapshotRepository.save(new AggregateSnapshot(
                account.getId(), "BankAccount", account.getVersion(), json
            ));
        } catch (Exception e) {
            throw new RuntimeException("Failed to create snapshot", e);
        }
    }

    private BankAccountAggregate deserializeSnapshot(AggregateSnapshot snap) {
        // In practice, restore state directly without replaying all events
        // This is a simplified example
        try {
            var state = objectMapper.readValue(snap.getSnapshotData(), java.util.Map.class);
            // Reconstitute aggregate state from snapshot data
            // (simplified - real implementation would set private fields directly)
            throw new UnsupportedOperationException("Implement based on your aggregate structure");
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize snapshot", e);
        }
    }
}
```

---

## Summary

| Concept | Role | Key Class |
|---------|------|-----------|
| Command | Intent to change state | `AccountCommand` (sealed interface) |
| Event | Record that something happened | `DomainEvent` subclasses |
| Aggregate | Business entity with consistency boundary | `BankAccountAggregate` |
| EventStore | Append-only event log | `EventStore`, `StoredEvent` |
| Repository | Load/save aggregates via events | `BankAccountRepository` |
| Projection | Denormalized read model | `AccountSummary`, `TransactionEntry` |
| Command Handler | Validates + dispatches commands | `BankAccountCommandHandler` |
| Query Service | Reads from read models | `AccountQueryService` |
| Saga | Multi-step distributed transaction with compensation | `TransferSaga` |
| Snapshot | Performance optimization for event replay | `AggregateSnapshot` |

**CQRS Flow:**
```
REST (POST) → Command → CommandHandler → Aggregate → Events → EventStore → Projection update
REST (GET)  → QueryService → ReadModel (AccountSummary, TransactionEntry)
```

**Event Sourcing Benefits:**
- Full audit trail: every state change is recorded
- Temporal queries: reconstruct state at any point in time
- Event replay: rebuild any read model from scratch
- Debugging: replay production events to reproduce bugs
- Loose coupling: new projections can be built from existing events

---

## Next Part Preview

**Part 047: Spring Cloud and Microservices** — We'll explore service discovery with Eureka, API Gateway with Spring Cloud Gateway, load balancing, circuit breakers with Resilience4j, distributed configuration with Spring Cloud Config, and distributed tracing with Zipkin/Micrometer.
