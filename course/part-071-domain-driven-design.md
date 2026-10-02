# Part 071: Domain-Driven Design with Spring Boot

Domain-Driven Design (DDD) is an approach to software development that focuses on understanding the business domain and reflecting that understanding in code. This part covers tactical and strategic DDD patterns implemented with Spring Boot.

---

## Table of Contents

1. [DDD Building Blocks Overview](#ddd-building-blocks)
2. [Value Objects](#value-objects)
3. [Entities](#entities)
4. [Aggregates and Aggregate Roots](#aggregates)
5. [Domain Services](#domain-services)
6. [Repositories](#repositories)
7. [Factories](#factories)
8. [Domain Events](#domain-events)
9. [Bounded Context and Context Map](#bounded-context)
10. [Application vs Domain vs Infrastructure Layer](#layers)
11. [Domain Model vs Persistence Model](#model-separation)
12. [Anti-Corruption Layer](#acl)
13. [Hexagonal Architecture](#hexagonal)
14. [Real Example: Order Management](#order-example)

---

## 1. DDD Building Blocks Overview {#ddd-building-blocks}

DDD provides a vocabulary and set of patterns for modeling complex business domains.

**Strategic Patterns:**
- Bounded Context — a boundary within which a domain model is consistent
- Ubiquitous Language — shared language between developers and domain experts
- Context Map — relationships between bounded contexts

**Tactical Patterns:**
- Entity — object with identity that persists over time
- Value Object — immutable object defined by its attributes
- Aggregate — cluster of domain objects treated as a unit
- Domain Service — stateless business logic that doesn't belong to an entity
- Repository — abstraction for data access
- Factory — encapsulates creation logic
- Domain Event — records something that happened in the domain

---

## 2. Value Objects {#value-objects}

Value objects are immutable and have no identity — equality is based on their values.

```java
// src/main/java/com/example/order/domain/model/Money.java
package com.example.order.domain.model;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Currency;
import java.util.Objects;

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

    public static Money of(BigDecimal amount, Currency currency) {
        return new Money(amount, currency);
    }

    public static Money of(String amount, String currencyCode) {
        return new Money(new BigDecimal(amount), Currency.getInstance(currencyCode));
    }

    public static Money ofUSD(BigDecimal amount) {
        return new Money(amount, Currency.getInstance("USD"));
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Result would be negative");
        }
        return new Money(result, this.currency);
    }

    public Money multiply(int multiplier) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(multiplier)), this.currency);
    }

    public Money multiply(BigDecimal factor) {
        return new Money(this.amount.multiply(factor), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isLessThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) < 0;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Currency mismatch: " + this.currency + " vs " + other.currency);
        }
    }

    public BigDecimal getAmount() { return amount; }
    public Currency getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return Objects.equals(amount, money.amount) &&
               Objects.equals(currency, money.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return currency.getSymbol() + amount.toPlainString();
    }
}
```

```java
// src/main/java/com/example/order/domain/model/Address.java
package com.example.order.domain.model;

import java.util.Objects;

public final class Address {

    private final String street;
    private final String city;
    private final String state;
    private final String postalCode;
    private final String country;

    public Address(String street, String city, String state,
                   String postalCode, String country) {
        this.street = requireNonBlank(street, "street");
        this.city = requireNonBlank(city, "city");
        this.state = requireNonBlank(state, "state");
        this.postalCode = requireNonBlank(postalCode, "postalCode");
        this.country = requireNonBlank(country, "country");
    }

    private static String requireNonBlank(String value, String field) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException(field + " cannot be blank");
        }
        return value;
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getState() { return state; }
    public String getPostalCode() { return postalCode; }
    public String getCountry() { return country; }

    public String format() {
        return street + ", " + city + ", " + state + " " + postalCode + ", " + country;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Address)) return false;
        Address address = (Address) o;
        return Objects.equals(street, address.street) &&
               Objects.equals(city, address.city) &&
               Objects.equals(state, address.state) &&
               Objects.equals(postalCode, address.postalCode) &&
               Objects.equals(country, address.country);
    }

    @Override
    public int hashCode() {
        return Objects.hash(street, city, state, postalCode, country);
    }
}
```

```java
// src/main/java/com/example/order/domain/model/CustomerId.java
package com.example.order.domain.model;

import java.util.Objects;
import java.util.UUID;

public final class CustomerId {

    private final UUID value;

    private CustomerId(UUID value) {
        this.value = Objects.requireNonNull(value, "CustomerId value cannot be null");
    }

    public static CustomerId of(UUID value) {
        return new CustomerId(value);
    }

    public static CustomerId of(String value) {
        return new CustomerId(UUID.fromString(value));
    }

    public static CustomerId generate() {
        return new CustomerId(UUID.randomUUID());
    }

    public UUID getValue() { return value; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CustomerId)) return false;
        CustomerId that = (CustomerId) o;
        return Objects.equals(value, that.value);
    }

    @Override
    public int hashCode() { return Objects.hash(value); }

    @Override
    public String toString() { return value.toString(); }
}
```

---

## 3. Entities {#entities}

Entities have identity — two entities with the same data but different IDs are different.

```java
// src/main/java/com/example/order/domain/model/OrderItem.java
package com.example.order.domain.model;

import java.util.Objects;
import java.util.UUID;

public class OrderItem {

    private final OrderItemId id;
    private final ProductId productId;
    private final String productName;
    private int quantity;
    private Money unitPrice;

    public OrderItem(OrderItemId id, ProductId productId,
                     String productName, int quantity, Money unitPrice) {
        this.id = Objects.requireNonNull(id);
        this.productId = Objects.requireNonNull(productId);
        this.productName = Objects.requireNonNull(productName);
        setQuantity(quantity);
        this.unitPrice = Objects.requireNonNull(unitPrice);
    }

    public static OrderItem create(ProductId productId, String productName,
                                    int quantity, Money unitPrice) {
        return new OrderItem(
            OrderItemId.generate(),
            productId,
            productName,
            quantity,
            unitPrice
        );
    }

    public void changeQuantity(int newQuantity) {
        setQuantity(newQuantity);
    }

    private void setQuantity(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive: " + quantity);
        }
        this.quantity = quantity;
    }

    public Money getTotalPrice() {
        return unitPrice.multiply(quantity);
    }

    public OrderItemId getId() { return id; }
    public ProductId getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public Money getUnitPrice() { return unitPrice; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderItem)) return false;
        OrderItem item = (OrderItem) o;
        return Objects.equals(id, item.id);
    }

    @Override
    public int hashCode() { return Objects.hash(id); }
}
```

```java
// src/main/java/com/example/order/domain/model/OrderItemId.java
package com.example.order.domain.model;

import java.util.Objects;
import java.util.UUID;

public final class OrderItemId {
    private final UUID value;

    private OrderItemId(UUID value) {
        this.value = Objects.requireNonNull(value);
    }

    public static OrderItemId of(UUID value) { return new OrderItemId(value); }
    public static OrderItemId of(String value) { return new OrderItemId(UUID.fromString(value)); }
    public static OrderItemId generate() { return new OrderItemId(UUID.randomUUID()); }

    public UUID getValue() { return value; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderItemId)) return false;
        return Objects.equals(value, ((OrderItemId) o).value);
    }
    @Override public int hashCode() { return Objects.hash(value); }
    @Override public String toString() { return value.toString(); }
}
```

---

## 4. Aggregates and Aggregate Roots {#aggregates}

An aggregate is a cluster of domain objects that must be treated as a single unit for data changes. The aggregate root is the entry point.

```java
// src/main/java/com/example/order/domain/model/Order.java
package com.example.order.domain.model;

import java.time.Instant;
import java.util.*;

public class Order {

    public enum Status {
        PENDING, CONFIRMED, PAID, SHIPPED, DELIVERED, CANCELLED
    }

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private Status status;
    private Address shippingAddress;
    private Instant createdAt;
    private Instant updatedAt;
    private final List<DomainEvent> domainEvents;

    // Private constructor — use factory methods
    private Order(OrderId id, CustomerId customerId, Address shippingAddress) {
        this.id = Objects.requireNonNull(id);
        this.customerId = Objects.requireNonNull(customerId);
        this.shippingAddress = Objects.requireNonNull(shippingAddress);
        this.items = new ArrayList<>();
        this.status = Status.PENDING;
        this.createdAt = Instant.now();
        this.updatedAt = Instant.now();
        this.domainEvents = new ArrayList<>();
    }

    // Factory method — encapsulates creation logic
    public static Order create(CustomerId customerId, Address shippingAddress) {
        OrderId id = OrderId.generate();
        Order order = new Order(id, customerId, shippingAddress);
        order.domainEvents.add(new OrderCreatedEvent(id, customerId, Instant.now()));
        return order;
    }

    // Reconstitution constructor for loading from persistence
    public static Order reconstitute(OrderId id, CustomerId customerId,
                                      Address shippingAddress, Status status,
                                      List<OrderItem> items, Instant createdAt,
                                      Instant updatedAt) {
        Order order = new Order(id, customerId, shippingAddress);
        order.status = status;
        order.items.addAll(items);
        order.createdAt = createdAt;
        order.updatedAt = updatedAt;
        order.domainEvents.clear(); // No events when reconstituting
        return order;
    }

    // ===== Domain Behavior =====

    public void addItem(ProductId productId, String productName,
                        int quantity, Money unitPrice) {
        ensureNotCancelledOrDelivered();

        // Check if product already in order
        Optional<OrderItem> existing = items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst();

        if (existing.isPresent()) {
            existing.get().changeQuantity(existing.get().getQuantity() + quantity);
        } else {
            OrderItem newItem = OrderItem.create(productId, productName, quantity, unitPrice);
            items.add(newItem);
        }
        this.updatedAt = Instant.now();
    }

    public void removeItem(OrderItemId itemId) {
        ensureNotCancelledOrDelivered();
        boolean removed = items.removeIf(item -> item.getId().equals(itemId));
        if (!removed) {
            throw new ItemNotFoundException("Item not found: " + itemId);
        }
        this.updatedAt = Instant.now();
    }

    public void confirm() {
        if (status != Status.PENDING) {
            throw new InvalidOrderStateException(
                "Cannot confirm order in status: " + status);
        }
        if (items.isEmpty()) {
            throw new InvalidOrderStateException("Cannot confirm empty order");
        }
        this.status = Status.CONFIRMED;
        this.updatedAt = Instant.now();
        domainEvents.add(new OrderConfirmedEvent(id, customerId, getTotalAmount(), Instant.now()));
    }

    public void pay(Money amountPaid) {
        if (status != Status.CONFIRMED) {
            throw new InvalidOrderStateException(
                "Cannot pay order in status: " + status);
        }
        Money total = getTotalAmount();
        if (amountPaid.isLessThan(total)) {
            throw new InsufficientPaymentException(
                "Payment " + amountPaid + " is less than order total " + total);
        }
        this.status = Status.PAID;
        this.updatedAt = Instant.now();
        domainEvents.add(new OrderPaidEvent(id, amountPaid, Instant.now()));
    }

    public void ship(String trackingNumber) {
        if (status != Status.PAID) {
            throw new InvalidOrderStateException(
                "Cannot ship order in status: " + status);
        }
        this.status = Status.SHIPPED;
        this.updatedAt = Instant.now();
        domainEvents.add(new OrderShippedEvent(id, trackingNumber, Instant.now()));
    }

    public void deliver() {
        if (status != Status.SHIPPED) {
            throw new InvalidOrderStateException(
                "Cannot deliver order in status: " + status);
        }
        this.status = Status.DELIVERED;
        this.updatedAt = Instant.now();
        domainEvents.add(new OrderDeliveredEvent(id, Instant.now()));
    }

    public void cancel(String reason) {
        if (status == Status.DELIVERED || status == Status.CANCELLED) {
            throw new InvalidOrderStateException(
                "Cannot cancel order in status: " + status);
        }
        this.status = Status.CANCELLED;
        this.updatedAt = Instant.now();
        domainEvents.add(new OrderCancelledEvent(id, reason, Instant.now()));
    }

    public void changeShippingAddress(Address newAddress) {
        if (status == Status.SHIPPED || status == Status.DELIVERED) {
            throw new InvalidOrderStateException(
                "Cannot change address for shipped or delivered orders");
        }
        this.shippingAddress = Objects.requireNonNull(newAddress);
        this.updatedAt = Instant.now();
    }

    // ===== Query Methods =====

    public Money getTotalAmount() {
        return items.stream()
            .map(OrderItem::getTotalPrice)
            .reduce(Money.ofUSD(java.math.BigDecimal.ZERO), Money::add);
    }

    public int getTotalItemCount() {
        return items.stream().mapToInt(OrderItem::getQuantity).sum();
    }

    private void ensureNotCancelledOrDelivered() {
        if (status == Status.CANCELLED || status == Status.DELIVERED) {
            throw new InvalidOrderStateException(
                "Cannot modify order in status: " + status);
        }
    }

    // ===== Domain Events =====

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        domainEvents.clear();
    }

    // ===== Getters =====

    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    public Status getStatus() { return status; }
    public Address getShippingAddress() { return shippingAddress; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}
```

```java
// src/main/java/com/example/order/domain/model/OrderId.java
package com.example.order.domain.model;

import java.util.Objects;
import java.util.UUID;

public final class OrderId {
    private final UUID value;

    private OrderId(UUID value) {
        this.value = Objects.requireNonNull(value);
    }

    public static OrderId of(UUID value) { return new OrderId(value); }
    public static OrderId of(String value) { return new OrderId(UUID.fromString(value)); }
    public static OrderId generate() { return new OrderId(UUID.randomUUID()); }

    public UUID getValue() { return value; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderId)) return false;
        return Objects.equals(value, ((OrderId) o).value);
    }
    @Override public int hashCode() { return Objects.hash(value); }
    @Override public String toString() { return value.toString(); }
}
```

---

## 5. Domain Services {#domain-services}

Domain services contain business logic that doesn't fit naturally in a single entity.

```java
// src/main/java/com/example/order/domain/service/ShippingCostCalculator.java
package com.example.order.domain.service;

import com.example.order.domain.model.*;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

/**
 * Domain Service: calculates shipping cost based on order and address.
 * This logic doesn't belong to Order or Address alone.
 */
public interface ShippingCostCalculator {
    Money calculate(Order order, Address destination);
}
```

```java
// src/main/java/com/example/order/domain/service/StandardShippingCostCalculator.java
package com.example.order.domain.service;

import com.example.order.domain.model.*;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

@Service
public class StandardShippingCostCalculator implements ShippingCostCalculator {

    private static final Money FREE_SHIPPING_THRESHOLD = Money.ofUSD(new BigDecimal("50.00"));
    private static final Money STANDARD_RATE = Money.ofUSD(new BigDecimal("5.99"));
    private static final Money EXPRESS_INTERNATIONAL_RATE = Money.ofUSD(new BigDecimal("29.99"));

    @Override
    public Money calculate(Order order, Address destination) {
        // Free shipping for large orders within the same country
        if (isSameCountry(destination) && order.getTotalAmount().isGreaterThan(FREE_SHIPPING_THRESHOLD)) {
            return Money.ofUSD(BigDecimal.ZERO);
        }

        if (isInternational(destination)) {
            return EXPRESS_INTERNATIONAL_RATE;
        }

        return STANDARD_RATE;
    }

    private boolean isSameCountry(Address destination) {
        return "US".equals(destination.getCountry());
    }

    private boolean isInternational(Address destination) {
        return !isSameCountry(destination);
    }
}
```

```java
// src/main/java/com/example/order/domain/service/DiscountService.java
package com.example.order.domain.service;

import com.example.order.domain.model.*;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

public interface DiscountService {
    Money calculateDiscount(Order order, Customer customer);
}
```

```java
// src/main/java/com/example/order/domain/service/LoyaltyDiscountService.java
package com.example.order.domain.service;

import com.example.order.domain.model.*;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

@Service
public class LoyaltyDiscountService implements DiscountService {

    @Override
    public Money calculateDiscount(Order order, Customer customer) {
        BigDecimal discountRate = determineDiscountRate(customer);

        if (discountRate.compareTo(BigDecimal.ZERO) == 0) {
            return Money.ofUSD(BigDecimal.ZERO);
        }

        return order.getTotalAmount().multiply(discountRate);
    }

    private BigDecimal determineDiscountRate(Customer customer) {
        int orderCount = customer.getCompletedOrderCount();

        if (orderCount >= 50) return new BigDecimal("0.15"); // 15% VIP
        if (orderCount >= 20) return new BigDecimal("0.10"); // 10% Gold
        if (orderCount >= 5)  return new BigDecimal("0.05"); // 5% Silver
        return BigDecimal.ZERO;
    }
}
```

---

## 6. Repositories {#repositories}

Repositories abstract data access. Domain code depends on the interface, not the implementation.

```java
// src/main/java/com/example/order/domain/repository/OrderRepository.java
package com.example.order.domain.repository;

import com.example.order.domain.model.*;

import java.util.List;
import java.util.Optional;

/**
 * Repository interface lives in the DOMAIN layer.
 * Implementation lives in the INFRASTRUCTURE layer.
 */
public interface OrderRepository {

    void save(Order order);

    Optional<Order> findById(OrderId id);

    List<Order> findByCustomerId(CustomerId customerId);

    List<Order> findByStatus(Order.Status status);

    void delete(OrderId id);

    boolean exists(OrderId id);
}
```

---

## 7. Domain Events {#domain-events}

Domain events capture things that happened in the domain.

```java
// src/main/java/com/example/order/domain/event/DomainEvent.java
package com.example.order.domain.event;

import java.time.Instant;
import java.util.UUID;

public interface DomainEvent {
    UUID getEventId();
    Instant getOccurredOn();
    String getEventType();
}
```

```java
// src/main/java/com/example/order/domain/event/OrderCreatedEvent.java
package com.example.order.domain.event;

import com.example.order.domain.model.*;

import java.time.Instant;
import java.util.UUID;

public class OrderCreatedEvent implements DomainEvent {

    private final UUID eventId;
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Instant occurredOn;

    public OrderCreatedEvent(OrderId orderId, CustomerId customerId, Instant occurredOn) {
        this.eventId = UUID.randomUUID();
        this.orderId = orderId;
        this.customerId = customerId;
        this.occurredOn = occurredOn;
    }

    @Override public UUID getEventId() { return eventId; }
    @Override public Instant getOccurredOn() { return occurredOn; }
    @Override public String getEventType() { return "order.created"; }

    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
}
```

```java
// src/main/java/com/example/order/domain/event/OrderConfirmedEvent.java
package com.example.order.domain.event;

import com.example.order.domain.model.*;

import java.time.Instant;
import java.util.UUID;

public class OrderConfirmedEvent implements DomainEvent {

    private final UUID eventId;
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money totalAmount;
    private final Instant occurredOn;

    public OrderConfirmedEvent(OrderId orderId, CustomerId customerId,
                                Money totalAmount, Instant occurredOn) {
        this.eventId = UUID.randomUUID();
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.occurredOn = occurredOn;
    }

    @Override public UUID getEventId() { return eventId; }
    @Override public Instant getOccurredOn() { return occurredOn; }
    @Override public String getEventType() { return "order.confirmed"; }

    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public Money getTotalAmount() { return totalAmount; }
}
```

```java
// src/main/java/com/example/order/domain/event/OrderPaidEvent.java
package com.example.order.domain.event;

import com.example.order.domain.model.*;
import java.time.Instant;
import java.util.UUID;

public class OrderPaidEvent implements DomainEvent {

    private final UUID eventId;
    private final OrderId orderId;
    private final Money amountPaid;
    private final Instant occurredOn;

    public OrderPaidEvent(OrderId orderId, Money amountPaid, Instant occurredOn) {
        this.eventId = UUID.randomUUID();
        this.orderId = orderId;
        this.amountPaid = amountPaid;
        this.occurredOn = occurredOn;
    }

    @Override public UUID getEventId() { return eventId; }
    @Override public Instant getOccurredOn() { return occurredOn; }
    @Override public String getEventType() { return "order.paid"; }

    public OrderId getOrderId() { return orderId; }
    public Money getAmountPaid() { return amountPaid; }
}
```

### Publishing Domain Events with Spring

```java
// src/main/java/com/example/order/infrastructure/event/SpringDomainEventPublisher.java
package com.example.order.infrastructure.event;

import com.example.order.domain.event.DomainEvent;
import com.example.order.domain.model.Order;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Component;

@Component
public class SpringDomainEventPublisher {

    private final ApplicationEventPublisher eventPublisher;

    public SpringDomainEventPublisher(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void publishEventsFrom(Order order) {
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
        order.clearDomainEvents();
    }
}
```

```java
// src/main/java/com/example/order/infrastructure/event/OrderEventHandler.java
package com.example.order.infrastructure.event;

import com.example.order.domain.event.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;

@Component
public class OrderEventHandler {

    private static final Logger log = LoggerFactory.getLogger(OrderEventHandler.class);

    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Order created: orderId={}, customerId={}",
            event.getOrderId(), event.getCustomerId());
        // Could trigger: send welcome email, create audit log, etc.
    }

    @Async
    @EventListener
    public void onOrderConfirmed(OrderConfirmedEvent event) {
        log.info("Order confirmed: orderId={}, total={}",
            event.getOrderId(), event.getTotalAmount());
        // Could trigger: send confirmation email, notify warehouse, etc.
    }

    @Async
    @EventListener
    public void onOrderPaid(OrderPaidEvent event) {
        log.info("Order paid: orderId={}, amount={}",
            event.getOrderId(), event.getAmountPaid());
        // Could trigger: start fulfillment, update inventory, etc.
    }
}
```

---

## 8. Bounded Context and Context Map {#bounded-context}

A bounded context defines a boundary within which a particular domain model is consistently defined. The Ubiquitous Language applies within this context.

```
Order Management Context:
  - Order, OrderItem, OrderId
  - Customer (minimal representation)
  - Product (minimal representation — just ID and price)

Product Catalog Context:
  - Product (full model with descriptions, categories, images)
  - Category, Brand, Inventory

Customer Context:
  - Customer (full model with personal info, preferences)
  - Address, LoyaltyTier

Payment Context:
  - Payment, PaymentMethod, Transaction
  - Order (minimal — just ID and amount)
```

---

## 9. Application vs Domain vs Infrastructure Layer {#layers}

```
src/main/java/com/example/order/
├── domain/                     ← Domain Layer (pure Java, no frameworks)
│   ├── model/                  ← Entities, Value Objects, Aggregates
│   │   ├── Order.java
│   │   ├── OrderItem.java
│   │   ├── Money.java
│   │   └── Address.java
│   ├── repository/             ← Repository interfaces
│   │   └── OrderRepository.java
│   ├── service/                ← Domain Services
│   │   └── ShippingCostCalculator.java
│   └── event/                  ← Domain Events
│       ├── DomainEvent.java
│       └── OrderCreatedEvent.java
│
├── application/                ← Application Layer (use cases, orchestration)
│   ├── usecase/
│   │   ├── CreateOrderUseCase.java
│   │   ├── ConfirmOrderUseCase.java
│   │   └── PayOrderUseCase.java
│   ├── dto/                    ← Application DTOs (input/output)
│   │   ├── CreateOrderRequest.java
│   │   └── OrderResponse.java
│   └── exception/
│       └── OrderNotFoundException.java
│
└── infrastructure/             ← Infrastructure Layer (Spring, JPA, etc.)
    ├── persistence/
    │   ├── entity/             ← JPA entities
    │   │   └── OrderJpaEntity.java
    │   ├── repository/         ← JPA repositories
    │   │   └── OrderJpaRepository.java
    │   └── mapper/             ← Domain ↔ Persistence mappers
    │       └── OrderMapper.java
    ├── web/                    ← REST controllers
    │   └── OrderController.java
    └── event/                  ← Event publishers/handlers
        └── OrderEventHandler.java
```

### Application Use Case

```java
// src/main/java/com/example/order/application/usecase/CreateOrderUseCase.java
package com.example.order.application.usecase;

import com.example.order.application.dto.*;
import com.example.order.domain.model.*;
import com.example.order.domain.repository.OrderRepository;
import com.example.order.infrastructure.event.SpringDomainEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CreateOrderUseCase {

    private final OrderRepository orderRepository;
    private final SpringDomainEventPublisher eventPublisher;

    public CreateOrderUseCase(OrderRepository orderRepository,
                               SpringDomainEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public OrderResponse execute(CreateOrderRequest request) {
        // Create the aggregate using its factory method
        Order order = Order.create(
            CustomerId.of(request.getCustomerId()),
            new Address(
                request.getStreet(),
                request.getCity(),
                request.getState(),
                request.getPostalCode(),
                request.getCountry()
            )
        );

        // Add items
        for (CreateOrderRequest.ItemRequest item : request.getItems()) {
            order.addItem(
                ProductId.of(item.getProductId()),
                item.getProductName(),
                item.getQuantity(),
                Money.ofUSD(item.getUnitPrice())
            );
        }

        // Persist the aggregate
        orderRepository.save(order);

        // Publish domain events after successful persistence
        eventPublisher.publishEventsFrom(order);

        return OrderResponse.from(order);
    }
}
```

```java
// src/main/java/com/example/order/application/dto/CreateOrderRequest.java
package com.example.order.application.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

public class CreateOrderRequest {

    @NotNull
    private UUID customerId;

    @NotBlank
    private String street;

    @NotBlank
    private String city;

    @NotBlank
    private String state;

    @NotBlank
    private String postalCode;

    @NotBlank
    private String country;

    @NotEmpty
    @Valid
    private List<ItemRequest> items;

    public static class ItemRequest {
        @NotNull
        private UUID productId;

        @NotBlank
        private String productName;

        @Min(1)
        private int quantity;

        @NotNull
        @Positive
        private BigDecimal unitPrice;

        // Getters and setters
        public UUID getProductId() { return productId; }
        public void setProductId(UUID productId) { this.productId = productId; }
        public String getProductName() { return productName; }
        public void setProductName(String productName) { this.productName = productName; }
        public int getQuantity() { return quantity; }
        public void setQuantity(int quantity) { this.quantity = quantity; }
        public BigDecimal getUnitPrice() { return unitPrice; }
        public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
    }

    // Getters and setters
    public UUID getCustomerId() { return customerId; }
    public void setCustomerId(UUID customerId) { this.customerId = customerId; }
    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getState() { return state; }
    public void setState(String state) { this.state = state; }
    public String getPostalCode() { return postalCode; }
    public void setPostalCode(String postalCode) { this.postalCode = postalCode; }
    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }
    public List<ItemRequest> getItems() { return items; }
    public void setItems(List<ItemRequest> items) { this.items = items; }
}
```

```java
// src/main/java/com/example/order/application/dto/OrderResponse.java
package com.example.order.application.dto;

import com.example.order.domain.model.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

public class OrderResponse {

    private UUID id;
    private UUID customerId;
    private String status;
    private BigDecimal totalAmount;
    private String currency;
    private List<ItemResponse> items;
    private Instant createdAt;

    public static OrderResponse from(Order order) {
        OrderResponse response = new OrderResponse();
        response.id = order.getId().getValue();
        response.customerId = order.getCustomerId().getValue();
        response.status = order.getStatus().name();
        response.totalAmount = order.getTotalAmount().getAmount();
        response.currency = order.getTotalAmount().getCurrency().getCurrencyCode();
        response.items = order.getItems().stream()
            .map(ItemResponse::from)
            .collect(Collectors.toList());
        response.createdAt = order.getCreatedAt();
        return response;
    }

    public static class ItemResponse {
        private UUID id;
        private UUID productId;
        private String productName;
        private int quantity;
        private BigDecimal unitPrice;
        private BigDecimal totalPrice;

        public static ItemResponse from(OrderItem item) {
            ItemResponse response = new ItemResponse();
            response.id = item.getId().getValue();
            response.productId = item.getProductId().getValue();
            response.productName = item.getProductName();
            response.quantity = item.getQuantity();
            response.unitPrice = item.getUnitPrice().getAmount();
            response.totalPrice = item.getTotalPrice().getAmount();
            return response;
        }

        public UUID getId() { return id; }
        public UUID getProductId() { return productId; }
        public String getProductName() { return productName; }
        public int getQuantity() { return quantity; }
        public BigDecimal getUnitPrice() { return unitPrice; }
        public BigDecimal getTotalPrice() { return totalPrice; }
    }

    public UUID getId() { return id; }
    public UUID getCustomerId() { return customerId; }
    public String getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public String getCurrency() { return currency; }
    public List<ItemResponse> getItems() { return items; }
    public Instant getCreatedAt() { return createdAt; }
}
```

---

## 10. Domain Model vs Persistence Model {#model-separation}

```java
// src/main/java/com/example/order/infrastructure/persistence/entity/OrderJpaEntity.java
package com.example.order.infrastructure.persistence.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.*;

@Entity
@Table(name = "orders")
public class OrderJpaEntity {

    @Id
    @Column(name = "id", columnDefinition = "uuid")
    private UUID id;

    @Column(name = "customer_id", nullable = false, columnDefinition = "uuid")
    private UUID customerId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private String status;

    @Column(name = "shipping_street")
    private String shippingStreet;

    @Column(name = "shipping_city")
    private String shippingCity;

    @Column(name = "shipping_state")
    private String shippingState;

    @Column(name = "shipping_postal_code")
    private String shippingPostalCode;

    @Column(name = "shipping_country")
    private String shippingCountry;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true,
               fetch = FetchType.EAGER)
    private List<OrderItemJpaEntity> items = new ArrayList<>();

    // Getters and setters
    public UUID getId() { return id; }
    public void setId(UUID id) { this.id = id; }
    public UUID getCustomerId() { return customerId; }
    public void setCustomerId(UUID customerId) { this.customerId = customerId; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String getShippingStreet() { return shippingStreet; }
    public void setShippingStreet(String shippingStreet) { this.shippingStreet = shippingStreet; }
    public String getShippingCity() { return shippingCity; }
    public void setShippingCity(String shippingCity) { this.shippingCity = shippingCity; }
    public String getShippingState() { return shippingState; }
    public void setShippingState(String shippingState) { this.shippingState = shippingState; }
    public String getShippingPostalCode() { return shippingPostalCode; }
    public void setShippingPostalCode(String code) { this.shippingPostalCode = code; }
    public String getShippingCountry() { return shippingCountry; }
    public void setShippingCountry(String country) { this.shippingCountry = country; }
    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(Instant updatedAt) { this.updatedAt = updatedAt; }
    public List<OrderItemJpaEntity> getItems() { return items; }
    public void setItems(List<OrderItemJpaEntity> items) { this.items = items; }
}
```

```java
// src/main/java/com/example/order/infrastructure/persistence/entity/OrderItemJpaEntity.java
package com.example.order.infrastructure.persistence.entity;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.util.UUID;

@Entity
@Table(name = "order_items")
public class OrderItemJpaEntity {

    @Id
    @Column(name = "id", columnDefinition = "uuid")
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private OrderJpaEntity order;

    @Column(name = "product_id", nullable = false, columnDefinition = "uuid")
    private UUID productId;

    @Column(name = "product_name", nullable = false)
    private String productName;

    @Column(name = "quantity", nullable = false)
    private int quantity;

    @Column(name = "unit_price", nullable = false)
    private BigDecimal unitPrice;

    @Column(name = "currency", nullable = false)
    private String currency;

    // Getters and setters
    public UUID getId() { return id; }
    public void setId(UUID id) { this.id = id; }
    public OrderJpaEntity getOrder() { return order; }
    public void setOrder(OrderJpaEntity order) { this.order = order; }
    public UUID getProductId() { return productId; }
    public void setProductId(UUID productId) { this.productId = productId; }
    public String getProductName() { return productName; }
    public void setProductName(String name) { this.productName = name; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal price) { this.unitPrice = price; }
    public String getCurrency() { return currency; }
    public void setCurrency(String currency) { this.currency = currency; }
}
```

---

## 11. Anti-Corruption Layer with Mapper {#acl}

The mapper translates between the domain model and the persistence model.

```java
// src/main/java/com/example/order/infrastructure/persistence/mapper/OrderMapper.java
package com.example.order.infrastructure.persistence.mapper;

import com.example.order.domain.model.*;
import com.example.order.infrastructure.persistence.entity.*;
import org.springframework.stereotype.Component;

import java.util.Currency;
import java.util.stream.Collectors;

@Component
public class OrderMapper {

    /**
     * Convert domain Order to JPA entity for persistence
     */
    public OrderJpaEntity toJpaEntity(Order order) {
        OrderJpaEntity entity = new OrderJpaEntity();
        entity.setId(order.getId().getValue());
        entity.setCustomerId(order.getCustomerId().getValue());
        entity.setStatus(order.getStatus().name());

        Address addr = order.getShippingAddress();
        entity.setShippingStreet(addr.getStreet());
        entity.setShippingCity(addr.getCity());
        entity.setShippingState(addr.getState());
        entity.setShippingPostalCode(addr.getPostalCode());
        entity.setShippingCountry(addr.getCountry());
        entity.setCreatedAt(order.getCreatedAt());
        entity.setUpdatedAt(order.getUpdatedAt());

        // Map items
        for (OrderItem item : order.getItems()) {
            OrderItemJpaEntity itemEntity = toItemJpaEntity(item, entity);
            entity.getItems().add(itemEntity);
        }

        return entity;
    }

    private OrderItemJpaEntity toItemJpaEntity(OrderItem item, OrderJpaEntity orderEntity) {
        OrderItemJpaEntity entity = new OrderItemJpaEntity();
        entity.setId(item.getId().getValue());
        entity.setOrder(orderEntity);
        entity.setProductId(item.getProductId().getValue());
        entity.setProductName(item.getProductName());
        entity.setQuantity(item.getQuantity());
        entity.setUnitPrice(item.getUnitPrice().getAmount());
        entity.setCurrency(item.getUnitPrice().getCurrency().getCurrencyCode());
        return entity;
    }

    /**
     * Convert JPA entity back to domain Order
     */
    public Order toDomain(OrderJpaEntity entity) {
        Address shippingAddress = new Address(
            entity.getShippingStreet(),
            entity.getShippingCity(),
            entity.getShippingState(),
            entity.getShippingPostalCode(),
            entity.getShippingCountry()
        );

        java.util.List<OrderItem> items = entity.getItems().stream()
            .map(this::toItemDomain)
            .collect(Collectors.toList());

        return Order.reconstitute(
            OrderId.of(entity.getId()),
            CustomerId.of(entity.getCustomerId()),
            shippingAddress,
            Order.Status.valueOf(entity.getStatus()),
            items,
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }

    private OrderItem toItemDomain(OrderItemJpaEntity entity) {
        return new OrderItem(
            OrderItemId.of(entity.getId()),
            ProductId.of(entity.getProductId()),
            entity.getProductName(),
            entity.getQuantity(),
            Money.of(entity.getUnitPrice(), Currency.getInstance(entity.getCurrency()))
        );
    }

    /**
     * Update existing JPA entity from domain Order (for merge/update scenarios)
     */
    public void updateJpaEntity(OrderJpaEntity jpaEntity, Order order) {
        jpaEntity.setStatus(order.getStatus().name());
        jpaEntity.setUpdatedAt(order.getUpdatedAt());

        Address addr = order.getShippingAddress();
        jpaEntity.setShippingStreet(addr.getStreet());
        jpaEntity.setShippingCity(addr.getCity());
        jpaEntity.setShippingState(addr.getState());
        jpaEntity.setShippingPostalCode(addr.getPostalCode());
        jpaEntity.setShippingCountry(addr.getCountry());

        // Update items — simplistic approach: clear and re-add
        jpaEntity.getItems().clear();
        for (OrderItem item : order.getItems()) {
            jpaEntity.getItems().add(toItemJpaEntity(item, jpaEntity));
        }
    }
}
```

---

## 12. Repository Implementation

```java
// src/main/java/com/example/order/infrastructure/persistence/repository/JpaOrderRepository.java
package com.example.order.infrastructure.persistence.repository;

import com.example.order.domain.model.*;
import com.example.order.domain.repository.OrderRepository;
import com.example.order.infrastructure.persistence.entity.OrderJpaEntity;
import com.example.order.infrastructure.persistence.mapper.OrderMapper;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.*;
import java.util.stream.Collectors;

@Repository
public class JpaOrderRepositoryAdapter implements OrderRepository {

    private final SpringDataOrderRepository springDataRepo;
    private final OrderMapper mapper;

    public JpaOrderRepositoryAdapter(SpringDataOrderRepository springDataRepo,
                                      OrderMapper mapper) {
        this.springDataRepo = springDataRepo;
        this.mapper = mapper;
    }

    @Override
    public void save(Order order) {
        Optional<OrderJpaEntity> existing = springDataRepo.findById(order.getId().getValue());
        if (existing.isPresent()) {
            mapper.updateJpaEntity(existing.get(), order);
            springDataRepo.save(existing.get());
        } else {
            springDataRepo.save(mapper.toJpaEntity(order));
        }
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return springDataRepo.findById(id.getValue())
            .map(mapper::toDomain);
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return springDataRepo.findByCustomerId(customerId.getValue())
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findByStatus(Order.Status status) {
        return springDataRepo.findByStatus(status.name())
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public void delete(OrderId id) {
        springDataRepo.deleteById(id.getValue());
    }

    @Override
    public boolean exists(OrderId id) {
        return springDataRepo.existsById(id.getValue());
    }
}
```

```java
// src/main/java/com/example/order/infrastructure/persistence/repository/SpringDataOrderRepository.java
package com.example.order.infrastructure.persistence.repository;

import com.example.order.infrastructure.persistence.entity.OrderJpaEntity;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.UUID;

public interface SpringDataOrderRepository extends JpaRepository<OrderJpaEntity, UUID> {
    List<OrderJpaEntity> findByCustomerId(UUID customerId);
    List<OrderJpaEntity> findByStatus(String status);
}
```

---

## 13. Hexagonal Architecture (Ports and Adapters) {#hexagonal}

```java
// Primary Port — input to the application
// src/main/java/com/example/order/application/port/in/CreateOrderPort.java
package com.example.order.application.port.in;

import com.example.order.application.dto.*;

public interface CreateOrderPort {
    OrderResponse createOrder(CreateOrderRequest request);
}
```

```java
// Secondary Port — output from the application
// src/main/java/com/example/order/application/port/out/SaveOrderPort.java
package com.example.order.application.port.out;

import com.example.order.domain.model.Order;

public interface SaveOrderPort {
    void save(Order order);
}
```

```java
// src/main/java/com/example/order/application/port/out/LoadOrderPort.java
package com.example.order.application.port.out;

import com.example.order.domain.model.*;
import java.util.Optional;

public interface LoadOrderPort {
    Optional<Order> loadById(OrderId id);
}
```

```java
// src/main/java/com/example/order/application/port/out/NotifyCustomerPort.java
package com.example.order.application.port.out;

import com.example.order.domain.model.*;

public interface NotifyCustomerPort {
    void notifyOrderCreated(Order order);
}
```

```java
// Use Case implementing the primary port
// src/main/java/com/example/order/application/usecase/CreateOrderService.java
package com.example.order.application.usecase;

import com.example.order.application.dto.*;
import com.example.order.application.port.in.CreateOrderPort;
import com.example.order.application.port.out.*;
import com.example.order.domain.model.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CreateOrderService implements CreateOrderPort {

    private final SaveOrderPort saveOrderPort;
    private final NotifyCustomerPort notifyCustomerPort;

    public CreateOrderService(SaveOrderPort saveOrderPort,
                               NotifyCustomerPort notifyCustomerPort) {
        this.saveOrderPort = saveOrderPort;
        this.notifyCustomerPort = notifyCustomerPort;
    }

    @Override
    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        Order order = Order.create(
            CustomerId.of(request.getCustomerId()),
            new Address(request.getStreet(), request.getCity(),
                        request.getState(), request.getPostalCode(), request.getCountry())
        );

        for (var item : request.getItems()) {
            order.addItem(
                ProductId.of(item.getProductId()),
                item.getProductName(),
                item.getQuantity(),
                Money.ofUSD(item.getUnitPrice())
            );
        }

        saveOrderPort.save(order);
        notifyCustomerPort.notifyOrderCreated(order);

        return OrderResponse.from(order);
    }
}
```

---

## 14. REST Controller

```java
// src/main/java/com/example/order/infrastructure/web/OrderController.java
package com.example.order.infrastructure.web;

import com.example.order.application.dto.*;
import com.example.order.application.port.in.CreateOrderPort;
import com.example.order.application.usecase.ConfirmOrderUseCase;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final CreateOrderPort createOrderPort;
    private final ConfirmOrderUseCase confirmOrderUseCase;

    public OrderController(CreateOrderPort createOrderPort,
                           ConfirmOrderUseCase confirmOrderUseCase) {
        this.createOrderPort = createOrderPort;
        this.confirmOrderUseCase = confirmOrderUseCase;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        OrderResponse response = createOrderPort.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @PostMapping("/{orderId}/confirm")
    public ResponseEntity<OrderResponse> confirmOrder(@PathVariable UUID orderId) {
        OrderResponse response = confirmOrderUseCase.execute(orderId);
        return ResponseEntity.ok(response);
    }
}
```

---

## 15. Domain Model Tests

```java
// src/test/java/com/example/order/domain/model/OrderTest.java
package com.example.order.domain.model;

import org.junit.jupiter.api.*;
import java.math.BigDecimal;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class OrderTest {

    private CustomerId customerId;
    private Address shippingAddress;
    private ProductId productId;

    @BeforeEach
    void setUp() {
        customerId = CustomerId.generate();
        shippingAddress = new Address("123 Main St", "Springfield", "IL", "62701", "US");
        productId = ProductId.generate();
    }

    @Test
    void shouldCreateOrderInPendingStatus() {
        Order order = Order.create(customerId, shippingAddress);
        assertThat(order.getStatus()).isEqualTo(Order.Status.PENDING);
        assertThat(order.getId()).isNotNull();
        assertThat(order.getCustomerId()).isEqualTo(customerId);
        assertThat(order.getItems()).isEmpty();
    }

    @Test
    void shouldAddItemToOrder() {
        Order order = Order.create(customerId, shippingAddress);
        order.addItem(productId, "Laptop", 2, Money.ofUSD(new BigDecimal("999.99")));

        assertThat(order.getItems()).hasSize(1);
        assertThat(order.getTotalAmount())
            .isEqualTo(Money.ofUSD(new BigDecimal("1999.98")));
    }

    @Test
    void shouldConsolidateItemsWithSameProduct() {
        Order order = Order.create(customerId, shippingAddress);
        order.addItem(productId, "Laptop", 1, Money.ofUSD(new BigDecimal("999.99")));
        order.addItem(productId, "Laptop", 2, Money.ofUSD(new BigDecimal("999.99")));

        assertThat(order.getItems()).hasSize(1);
        assertThat(order.getItems().get(0).getQuantity()).isEqualTo(3);
    }

    @Test
    void shouldConfirmOrderWithItems() {
        Order order = Order.create(customerId, shippingAddress);
        order.addItem(productId, "Laptop", 1, Money.ofUSD(new BigDecimal("999.99")));

        order.confirm();

        assertThat(order.getStatus()).isEqualTo(Order.Status.CONFIRMED);
    }

    @Test
    void shouldNotConfirmEmptyOrder() {
        Order order = Order.create(customerId, shippingAddress);

        assertThatThrownBy(order::confirm)
            .isInstanceOf(InvalidOrderStateException.class)
            .hasMessageContaining("empty order");
    }

    @Test
    void shouldNotCancelDeliveredOrder() {
        Order order = createDeliveredOrder();

        assertThatThrownBy(() -> order.cancel("Changed mind"))
            .isInstanceOf(InvalidOrderStateException.class);
    }

    @Test
    void shouldPublishDomainEventsOnStateChange() {
        Order order = Order.create(customerId, shippingAddress);
        order.addItem(productId, "Laptop", 1, Money.ofUSD(new BigDecimal("999.99")));
        order.confirm();

        assertThat(order.getDomainEvents()).hasSize(2);
        assertThat(order.getDomainEvents().get(0))
            .isInstanceOf(OrderCreatedEvent.class);
        assertThat(order.getDomainEvents().get(1))
            .isInstanceOf(OrderConfirmedEvent.class);
    }

    @Test
    void shouldCalculateTotalCorrectly() {
        Order order = Order.create(customerId, shippingAddress);
        order.addItem(productId, "Laptop", 2, Money.ofUSD(new BigDecimal("999.99")));
        order.addItem(ProductId.generate(), "Mouse", 3, Money.ofUSD(new BigDecimal("29.99")));

        Money expected = Money.ofUSD(new BigDecimal("2089.95")); // 2*999.99 + 3*29.99
        assertThat(order.getTotalAmount()).isEqualTo(expected);
    }

    private Order createDeliveredOrder() {
        Order order = Order.create(customerId, shippingAddress);
        order.addItem(productId, "Laptop", 1, Money.ofUSD(new BigDecimal("999.99")));
        order.confirm();
        order.pay(Money.ofUSD(new BigDecimal("999.99")));
        order.ship("TRACK-123");
        order.deliver();
        return order;
    }
}
```

```java
// src/test/java/com/example/order/domain/model/MoneyTest.java
package com.example.order.domain.model;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;

class MoneyTest {

    @Test
    void shouldAddMoneyWithSameCurrency() {
        Money a = Money.ofUSD(new BigDecimal("10.00"));
        Money b = Money.ofUSD(new BigDecimal("5.50"));
        assertThat(a.add(b)).isEqualTo(Money.ofUSD(new BigDecimal("15.50")));
    }

    @Test
    void shouldThrowWhenSubtractingMoreThanAvailable() {
        Money a = Money.ofUSD(new BigDecimal("5.00"));
        Money b = Money.ofUSD(new BigDecimal("10.00"));
        assertThatThrownBy(() -> a.subtract(b))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void shouldThrowWhenAddingDifferentCurrencies() {
        Money usd = Money.ofUSD(new BigDecimal("10.00"));
        Money eur = Money.of("10.00", "EUR");
        assertThatThrownBy(() -> usd.add(eur))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Currency mismatch");
    }

    @Test
    void shouldBeEqualWhenAmountAndCurrencyMatch() {
        Money a = Money.ofUSD(new BigDecimal("10.00"));
        Money b = Money.ofUSD(new BigDecimal("10.00"));
        assertThat(a).isEqualTo(b);
        assertThat(a.hashCode()).isEqualTo(b.hashCode());
    }
}
```

---

## Summary

| Concept | Where | Role |
|---|---|---|
| Value Object | Domain layer | Immutable, no identity (Money, Address) |
| Entity | Domain layer | Has identity, mutable behavior (OrderItem) |
| Aggregate Root | Domain layer | Controls access to cluster (Order) |
| Domain Service | Domain layer | Cross-entity business logic |
| Repository Interface | Domain layer | Data access abstraction |
| Domain Event | Domain layer | Records things that happened |
| Use Case | Application layer | Orchestrates domain objects |
| Repository Impl | Infrastructure layer | JPA/DB implementation |
| Mapper/ACL | Infrastructure layer | Translates between models |
| Controller | Infrastructure layer | HTTP adapter |

### Key Principles

- Domain model is pure Java — no Spring annotations, no JPA annotations
- Domain objects enforce invariants — invalid state is impossible
- Repositories have interfaces in domain, implementations in infrastructure
- Domain events decouple domain logic from side effects
- Mappers translate between domain and persistence models

---

## Next Part Preview

**Part 072: Clean Architecture with Spring Boot** — takes DDD concepts further with Clean Architecture's strict dependency rule: business logic in the center, frameworks at the edges. You'll implement the Interactor pattern, Input/Output ports, and learn how to test each layer in complete isolation.
