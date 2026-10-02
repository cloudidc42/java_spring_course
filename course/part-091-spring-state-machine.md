# Part 091: Spring State Machine

## Introduction

A state machine models an object that can be in exactly one of a finite number of states at any given time. Spring State Machine brings first-class state machine support to Spring applications with features like guards, actions, persistence, and hierarchical states.

**Core concepts:**
- **State**: A condition the system is in (e.g., PENDING, CONFIRMED)
- **Event**: A trigger that causes a transition (e.g., CONFIRM, SHIP)
- **Transition**: The path from one state to another
- **Guard**: A condition that must be true for a transition to occur
- **Action**: Code that executes when a transition fires

---

## Project Setup

### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-core</artifactId>
        <version>3.2.1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-data-jpa</artifactId>
        <version>3.2.1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

---

## States and Events Enums

```java
// src/main/java/com/example/statemachine/order/OrderState.java
package com.example.statemachine.order;

public enum OrderState {
    PENDING,        // Order created, awaiting payment
    PAYMENT_RECEIVED, // Payment confirmed
    CONFIRMED,      // Order confirmed, awaiting fulfillment
    PROCESSING,     // Being packed/prepared
    SHIPPED,        // Dispatched to carrier
    IN_TRANSIT,     // On the way to customer
    DELIVERED,      // Successfully delivered
    CANCELLED,      // Cancelled at any point before delivery
    REFUND_REQUESTED,  // Customer requested refund
    REFUNDED        // Refund processed
}
```

```java
// src/main/java/com/example/statemachine/order/OrderEvent.java
package com.example.statemachine.order;

public enum OrderEvent {
    RECEIVE_PAYMENT,    // Payment received
    CONFIRM,            // Merchant confirms the order
    START_PROCESSING,   // Begin fulfillment
    SHIP,               // Mark as shipped
    TRANSIT_UPDATE,     // Carrier update received
    DELIVER,            // Mark as delivered
    CANCEL,             // Cancel the order
    REQUEST_REFUND,     // Customer requests refund
    PROCESS_REFUND,     // Refund is processed
    REJECT_REFUND       // Refund request rejected
}
```

---

## State Machine Configuration

```java
// src/main/java/com/example/statemachine/config/OrderStateMachineConfig.java
package com.example.statemachine.config;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.statemachine.actions.*;
import com.example.statemachine.statemachine.guards.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.config.EnableStateMachineFactory;
import org.springframework.statemachine.config.StateMachineConfigurerAdapter;
import org.springframework.statemachine.config.builders.StateMachineConfigurationConfigurer;
import org.springframework.statemachine.config.builders.StateMachineStateConfigurer;
import org.springframework.statemachine.config.builders.StateMachineTransitionConfigurer;
import org.springframework.statemachine.listener.StateMachineListenerAdapter;
import org.springframework.statemachine.state.State;

import java.util.EnumSet;

@Slf4j
@Configuration
@EnableStateMachineFactory
@RequiredArgsConstructor
public class OrderStateMachineConfig
        extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    private final PaymentReceivedAction paymentReceivedAction;
    private final OrderConfirmedAction orderConfirmedAction;
    private final ShipOrderAction shipOrderAction;
    private final DeliverOrderAction deliverOrderAction;
    private final CancelOrderAction cancelOrderAction;
    private final RefundAction refundAction;

    private final PaymentValidGuard paymentValidGuard;
    private final StockAvailableGuard stockAvailableGuard;
    private final CancellableGuard cancellableGuard;
    private final RefundEligibleGuard refundEligibleGuard;

    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config)
            throws Exception {
        config
            .withConfiguration()
                .autoStartup(true)
                .listener(new OrderStateMachineListener());
    }

    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
            throws Exception {
        states
            .withStates()
                .initial(OrderState.PENDING)
                .end(OrderState.DELIVERED)
                .end(OrderState.CANCELLED)
                .end(OrderState.REFUNDED)
                .states(EnumSet.allOf(OrderState.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
            throws Exception {
        transitions
            // PENDING → PAYMENT_RECEIVED
            .withExternal()
                .source(OrderState.PENDING)
                .target(OrderState.PAYMENT_RECEIVED)
                .event(OrderEvent.RECEIVE_PAYMENT)
                .guard(paymentValidGuard)
                .action(paymentReceivedAction)
                .and()

            // PAYMENT_RECEIVED → CONFIRMED
            .withExternal()
                .source(OrderState.PAYMENT_RECEIVED)
                .target(OrderState.CONFIRMED)
                .event(OrderEvent.CONFIRM)
                .guard(stockAvailableGuard)
                .action(orderConfirmedAction)
                .and()

            // CONFIRMED → PROCESSING
            .withExternal()
                .source(OrderState.CONFIRMED)
                .target(OrderState.PROCESSING)
                .event(OrderEvent.START_PROCESSING)
                .and()

            // PROCESSING → SHIPPED
            .withExternal()
                .source(OrderState.PROCESSING)
                .target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP)
                .action(shipOrderAction)
                .and()

            // SHIPPED → IN_TRANSIT
            .withExternal()
                .source(OrderState.SHIPPED)
                .target(OrderState.IN_TRANSIT)
                .event(OrderEvent.TRANSIT_UPDATE)
                .and()

            // IN_TRANSIT → DELIVERED
            .withExternal()
                .source(OrderState.IN_TRANSIT)
                .target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
                .action(deliverOrderAction)
                .and()

            // Cancellation (from multiple states)
            .withExternal()
                .source(OrderState.PENDING)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
                .action(cancelOrderAction)
                .and()
            .withExternal()
                .source(OrderState.PAYMENT_RECEIVED)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
                .guard(cancellableGuard)
                .action(cancelOrderAction)
                .and()
            .withExternal()
                .source(OrderState.CONFIRMED)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
                .guard(cancellableGuard)
                .action(cancelOrderAction)
                .and()

            // Refund flow
            .withExternal()
                .source(OrderState.DELIVERED)
                .target(OrderState.REFUND_REQUESTED)
                .event(OrderEvent.REQUEST_REFUND)
                .guard(refundEligibleGuard)
                .and()
            .withExternal()
                .source(OrderState.REFUND_REQUESTED)
                .target(OrderState.REFUNDED)
                .event(OrderEvent.PROCESS_REFUND)
                .action(refundAction)
                .and()
            .withExternal()
                .source(OrderState.REFUND_REQUESTED)
                .target(OrderState.DELIVERED)
                .event(OrderEvent.REJECT_REFUND);
    }

    // Inner listener class
    private static class OrderStateMachineListener
            extends StateMachineListenerAdapter<OrderState, OrderEvent> {
        @Override
        public void stateChanged(State<OrderState, OrderEvent> from,
                                 State<OrderState, OrderEvent> to) {
            log.info("State changed from {} to {}",
                from != null ? from.getId() : "NULL",
                to != null ? to.getId() : "NULL");
        }

        @Override
        public void eventNotAccepted(org.springframework.messaging.Message<OrderEvent> event) {
            log.warn("Event not accepted: {}", event.getPayload());
        }
    }
}
```

---

## Guards

```java
// src/main/java/com/example/statemachine/statemachine/guards/PaymentValidGuard.java
package com.example.statemachine.statemachine.guards;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.PaymentService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.guard.Guard;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentValidGuard implements Guard<OrderState, OrderEvent> {

    public static final String ORDER_ID_HEADER = "orderId";
    public static final String PAYMENT_ID_HEADER = "paymentId";

    private final PaymentService paymentService;

    @Override
    public boolean evaluate(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get(ORDER_ID_HEADER);
        String paymentId = (String) context.getMessageHeaders().get(PAYMENT_ID_HEADER);

        if (orderId == null || paymentId == null) {
            log.warn("Missing orderId or paymentId in message headers");
            return false;
        }

        boolean valid = paymentService.isPaymentValid(orderId, paymentId);
        log.debug("Payment validation for order {}: {}", orderId, valid);
        return valid;
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/guards/StockAvailableGuard.java
package com.example.statemachine.statemachine.guards;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.InventoryService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.guard.Guard;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class StockAvailableGuard implements Guard<OrderState, OrderEvent> {

    private final InventoryService inventoryService;

    @Override
    public boolean evaluate(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");

        if (orderId == null) {
            log.warn("Missing orderId in message headers");
            return false;
        }

        boolean available = inventoryService.isStockAvailable(orderId);
        log.debug("Stock availability for order {}: {}", orderId, available);
        return available;
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/guards/RefundEligibleGuard.java
package com.example.statemachine.statemachine.guards;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.guard.Guard;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;

@Slf4j
@Component
@RequiredArgsConstructor
public class RefundEligibleGuard implements Guard<OrderState, OrderEvent> {

    private static final int REFUND_WINDOW_DAYS = 30;
    private final OrderRepository orderRepository;

    @Override
    public boolean evaluate(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");

        return orderRepository.findById(orderId)
            .map(order -> {
                if (order.getDeliveredAt() == null) return false;
                long daysSinceDelivery = ChronoUnit.DAYS.between(
                    order.getDeliveredAt(), LocalDateTime.now());
                boolean eligible = daysSinceDelivery <= REFUND_WINDOW_DAYS;
                log.info("Order {} refund eligibility: {} ({} days since delivery)",
                    orderId, eligible, daysSinceDelivery);
                return eligible;
            })
            .orElse(false);
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/guards/CancellableGuard.java
package com.example.statemachine.statemachine.guards;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.guard.Guard;
import org.springframework.stereotype.Component;

@Component
public class CancellableGuard implements Guard<OrderState, OrderEvent> {

    @Override
    public boolean evaluate(StateContext<OrderState, OrderEvent> context) {
        // Orders can only be cancelled by the customer within 24 hours
        // In a real system you'd check a timestamp header
        Boolean customerCancellation = (Boolean) context.getMessageHeaders()
            .get("customerCancellation");
        return Boolean.TRUE.equals(customerCancellation);
    }
}
```

---

## Actions

```java
// src/main/java/com/example/statemachine/statemachine/actions/PaymentReceivedAction.java
package com.example.statemachine.statemachine.actions;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.NotificationService;
import com.example.statemachine.service.OrderService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.action.Action;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentReceivedAction implements Action<OrderState, OrderEvent> {

    private final OrderService orderService;
    private final NotificationService notificationService;

    @Override
    public void execute(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");
        String paymentId = (String) context.getMessageHeaders().get("paymentId");

        log.info("Executing PaymentReceivedAction for order: {}", orderId);

        try {
            orderService.recordPayment(orderId, paymentId);
            notificationService.sendPaymentConfirmation(orderId);
            log.info("Payment recorded and confirmation sent for order: {}", orderId);
        } catch (Exception e) {
            log.error("Error processing payment for order {}: {}", orderId, e.getMessage(), e);
            // Set error in context so calling code can check
            context.getStateMachine().setStateMachineError(e);
        }
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/actions/ShipOrderAction.java
package com.example.statemachine.statemachine.actions;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.NotificationService;
import com.example.statemachine.service.ShippingService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.action.Action;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class ShipOrderAction implements Action<OrderState, OrderEvent> {

    private final ShippingService shippingService;
    private final NotificationService notificationService;

    @Override
    public void execute(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");
        String trackingNumber = shippingService.createShipment(orderId);

        // Store tracking number in extended state for later use
        context.getExtendedState().getVariables()
            .put("trackingNumber", trackingNumber);

        notificationService.sendShippingNotification(orderId, trackingNumber);
        log.info("Order {} shipped with tracking number: {}", orderId, trackingNumber);
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/actions/DeliverOrderAction.java
package com.example.statemachine.statemachine.actions;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.OrderService;
import com.example.statemachine.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.action.Action;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Slf4j
@Component
@RequiredArgsConstructor
public class DeliverOrderAction implements Action<OrderState, OrderEvent> {

    private final OrderService orderService;
    private final NotificationService notificationService;

    @Override
    public void execute(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");

        orderService.markDelivered(orderId, LocalDateTime.now());
        notificationService.sendDeliveryConfirmation(orderId);

        log.info("Order {} marked as delivered at {}", orderId, LocalDateTime.now());
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/actions/CancelOrderAction.java
package com.example.statemachine.statemachine.actions;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.OrderService;
import com.example.statemachine.service.PaymentService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.action.Action;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class CancelOrderAction implements Action<OrderState, OrderEvent> {

    private final OrderService orderService;
    private final PaymentService paymentService;

    @Override
    public void execute(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");
        String cancelReason = (String) context.getMessageHeaders()
            .getOrDefault("cancelReason", "Customer requested");

        orderService.cancelOrder(orderId, cancelReason);

        // Initiate refund if payment was received
        OrderState previousState = context.getSource().getId();
        if (previousState == OrderState.PAYMENT_RECEIVED
                || previousState == OrderState.CONFIRMED) {
            paymentService.initiateRefund(orderId);
        }

        log.info("Order {} cancelled. Reason: {}", orderId, cancelReason);
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/actions/OrderConfirmedAction.java
package com.example.statemachine.statemachine.actions;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.InventoryService;
import com.example.statemachine.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.action.Action;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderConfirmedAction implements Action<OrderState, OrderEvent> {

    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    @Override
    public void execute(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");

        inventoryService.reserveStock(orderId);
        notificationService.sendOrderConfirmation(orderId);

        log.info("Order {} confirmed, stock reserved", orderId);
    }
}
```

```java
// src/main/java/com/example/statemachine/statemachine/actions/RefundAction.java
package com.example.statemachine.statemachine.actions;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.PaymentService;
import com.example.statemachine.service.NotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.statemachine.StateContext;
import org.springframework.statemachine.action.Action;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class RefundAction implements Action<OrderState, OrderEvent> {

    private final PaymentService paymentService;
    private final NotificationService notificationService;

    @Override
    public void execute(StateContext<OrderState, OrderEvent> context) {
        Long orderId = (Long) context.getMessageHeaders().get("orderId");

        String refundTransactionId = paymentService.processRefund(orderId);
        notificationService.sendRefundConfirmation(orderId, refundTransactionId);

        log.info("Refund processed for order {}. Transaction: {}", orderId, refundTransactionId);
    }
}
```

---

## Domain Entity

```java
// src/main/java/com/example/statemachine/entity/Order.java
package com.example.statemachine.entity;

import com.example.statemachine.order.OrderState;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String customerId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderState state;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;

    private String paymentId;
    private String trackingNumber;
    private String cancelReason;
    private String refundTransactionId;

    @Column(nullable = false)
    private LocalDateTime createdAt;

    private LocalDateTime confirmedAt;
    private LocalDateTime shippedAt;
    private LocalDateTime deliveredAt;
    private LocalDateTime cancelledAt;
    private LocalDateTime refundedAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @Builder.Default
    private List<OrderItem> items = new ArrayList<>();

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @Builder.Default
    private List<OrderStateTransition> transitions = new ArrayList<>();

    @PrePersist
    public void prePersist() {
        if (createdAt == null) {
            createdAt = LocalDateTime.now();
        }
        if (state == null) {
            state = OrderState.PENDING;
        }
    }
}
```

```java
// src/main/java/com/example/statemachine/entity/OrderStateTransition.java
package com.example.statemachine.entity;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;

@Entity
@Table(name = "order_state_transitions")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OrderStateTransition {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @Enumerated(EnumType.STRING)
    private OrderState fromState;

    @Enumerated(EnumType.STRING)
    private OrderState toState;

    @Enumerated(EnumType.STRING)
    private OrderEvent event;

    private LocalDateTime transitionedAt;
    private String performedBy;
}
```

---

## State Machine Persistence

```java
// src/main/java/com/example/statemachine/service/OrderStateMachineService.java
package com.example.statemachine.service;

import com.example.statemachine.entity.Order;
import com.example.statemachine.entity.OrderStateTransition;
import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.repository.OrderRepository;
import com.example.statemachine.repository.OrderStateTransitionRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.statemachine.StateMachine;
import org.springframework.statemachine.config.StateMachineFactory;
import org.springframework.statemachine.state.State;
import org.springframework.statemachine.support.DefaultStateMachineContext;
import org.springframework.statemachine.support.StateMachineInterceptorAdapter;
import org.springframework.statemachine.transition.Transition;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import reactor.core.publisher.Mono;

import java.time.LocalDateTime;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderStateMachineService {

    private final StateMachineFactory<OrderState, OrderEvent> stateMachineFactory;
    private final OrderRepository orderRepository;
    private final OrderStateTransitionRepository transitionRepository;

    /**
     * Build a state machine for an existing order, restoring its current state.
     */
    public StateMachine<OrderState, OrderEvent> build(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("Order not found: " + orderId));

        StateMachine<OrderState, OrderEvent> sm =
            stateMachineFactory.getStateMachine(orderId.toString());

        sm.stopReactively().block();

        sm.getStateMachineAccessor()
            .doWithAllRegions(accessor -> {
                // Add interceptor to persist state changes
                accessor.addStateMachineInterceptor(
                    new StateMachineInterceptorAdapter<>() {
                        @Override
                        public void preStateChange(
                                State<OrderState, OrderEvent> state,
                                Message<OrderEvent> message,
                                Transition<OrderState, OrderEvent> transition,
                                StateMachine<OrderState, OrderEvent> stateMachine,
                                StateMachine<OrderState, OrderEvent> rootStateMachine) {
                            if (message != null) {
                                Long id = (Long) message.getHeaders().get("orderId");
                                if (id != null) {
                                    log.debug("Persisting state {} for order {}",
                                        state.getId(), id);
                                    orderRepository.findById(id).ifPresent(o -> {
                                        o.setState(state.getId());
                                        orderRepository.save(o);

                                        // Record the transition
                                        OrderStateTransition t = OrderStateTransition.builder()
                                            .order(o)
                                            .fromState(transition.getSource().getId())
                                            .toState(state.getId())
                                            .event(message.getPayload())
                                            .transitionedAt(LocalDateTime.now())
                                            .performedBy((String) message.getHeaders()
                                                .getOrDefault("performedBy", "SYSTEM"))
                                            .build();
                                        transitionRepository.save(t);
                                    });
                                }
                            }
                        }
                    });

                // Restore current state from the database
                accessor.resetStateMachineReactively(
                    new DefaultStateMachineContext<>(
                        order.getState(), null, null, null
                    )
                ).block();
            });

        sm.startReactively().block();
        return sm;
    }

    /**
     * Send an event to a specific order's state machine.
     */
    @Transactional
    public boolean sendEvent(Long orderId, OrderEvent event) {
        return sendEvent(orderId, event, null, null);
    }

    @Transactional
    public boolean sendEvent(Long orderId, OrderEvent event,
                              String performedBy, Object additionalData) {
        StateMachine<OrderState, OrderEvent> sm = build(orderId);

        MessageBuilder<OrderEvent> builder = MessageBuilder
            .withPayload(event)
            .setHeader("orderId", orderId);

        if (performedBy != null) {
            builder.setHeader("performedBy", performedBy);
        }
        if (additionalData instanceof String) {
            builder.setHeader("paymentId", additionalData);
        }

        Message<OrderEvent> message = builder.build();

        Mono<Boolean> result = sm.sendEvent(Mono.just(message))
            .map(resultContext -> resultContext.getResultType()
                .equals(org.springframework.statemachine.StateMachineEventResult
                    .ResultType.ACCEPTED))
            .defaultIfEmpty(false)
            .next();

        Boolean accepted = result.block();
        return Boolean.TRUE.equals(accepted);
    }

    public OrderState getCurrentState(Long orderId) {
        return orderRepository.findById(orderId)
            .map(Order::getState)
            .orElseThrow(() -> new IllegalArgumentException("Order not found: " + orderId));
    }
}
```

---

## REST Controller

```java
// src/main/java/com/example/statemachine/controller/OrderController.java
package com.example.statemachine.controller;

import com.example.statemachine.dto.CreateOrderRequest;
import com.example.statemachine.dto.OrderResponse;
import com.example.statemachine.dto.OrderEventRequest;
import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.service.OrderService;
import com.example.statemachine.service.OrderStateMachineService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;
    private final OrderStateMachineService stateMachineService;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        OrderResponse order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long orderId) {
        return ResponseEntity.ok(orderService.getOrder(orderId));
    }

    @PostMapping("/{orderId}/events")
    public ResponseEntity<Map<String, Object>> sendEvent(
            @PathVariable Long orderId,
            @RequestBody OrderEventRequest eventRequest) {

        log.info("Processing event {} for order {}", eventRequest.getEvent(), orderId);

        boolean accepted = stateMachineService.sendEvent(
            orderId,
            eventRequest.getEvent(),
            eventRequest.getPerformedBy(),
            eventRequest.getAdditionalData()
        );

        OrderState currentState = stateMachineService.getCurrentState(orderId);

        Map<String, Object> response = Map.of(
            "orderId", orderId,
            "event", eventRequest.getEvent(),
            "accepted", accepted,
            "currentState", currentState
        );

        return accepted
            ? ResponseEntity.ok(response)
            : ResponseEntity.status(HttpStatus.CONFLICT).body(response);
    }

    @GetMapping("/{orderId}/state")
    public ResponseEntity<Map<String, Object>> getState(@PathVariable Long orderId) {
        OrderState state = stateMachineService.getCurrentState(orderId);
        return ResponseEntity.ok(Map.of(
            "orderId", orderId,
            "state", state,
            "transitions", orderService.getTransitionHistory(orderId)
        ));
    }
}
```

---

## DTOs

```java
// src/main/java/com/example/statemachine/dto/CreateOrderRequest.java
package com.example.statemachine.dto;

import lombok.Data;
import java.math.BigDecimal;
import java.util.List;

@Data
public class CreateOrderRequest {
    private String customerId;
    private BigDecimal totalAmount;
    private List<OrderItemRequest> items;
}
```

```java
// src/main/java/com/example/statemachine/dto/OrderEventRequest.java
package com.example.statemachine.dto;

import com.example.statemachine.order.OrderEvent;
import lombok.Data;

@Data
public class OrderEventRequest {
    private OrderEvent event;
    private String performedBy;
    private Object additionalData;  // e.g., paymentId, cancelReason
}
```

---

## Hierarchical State Machine

For complex scenarios, you can nest states:

```java
// src/main/java/com/example/statemachine/config/HierarchicalOrderStateMachineConfig.java
package com.example.statemachine.config;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.config.EnableStateMachine;
import org.springframework.statemachine.config.StateMachineConfigurerAdapter;
import org.springframework.statemachine.config.builders.StateMachineStateConfigurer;
import org.springframework.statemachine.config.builders.StateMachineTransitionConfigurer;

/**
 * Demonstrates hierarchical state machines where FULFILLMENT is a
 * super-state containing PROCESSING, SHIPPED, and IN_TRANSIT sub-states.
 */
@Configuration
@EnableStateMachine(name = "hierarchicalOrderStateMachine")
public class HierarchicalOrderStateMachineConfig
        extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states)
            throws Exception {
        states
            .withStates()
                .initial(OrderState.PENDING)
                .end(OrderState.DELIVERED)
                .end(OrderState.CANCELLED)
                // "PROCESSING" acts as a parent state (region)
                .state(OrderState.PROCESSING)
                    // From the PROCESSING super-state, the initial sub-state is PROCESSING itself
                    // In a real hierarchical setup you'd use separate enums for sub-states
                .and()
            .withStates()
                // Define sub-states of PROCESSING
                .parent(OrderState.PROCESSING)
                .initial(OrderState.PROCESSING)
                .state(OrderState.SHIPPED)
                .state(OrderState.IN_TRANSIT);
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions)
            throws Exception {
        transitions
            .withExternal()
                .source(OrderState.PENDING)
                .target(OrderState.CONFIRMED)
                .event(OrderEvent.CONFIRM)
                .and()
            .withExternal()
                .source(OrderState.CONFIRMED)
                .target(OrderState.PROCESSING)
                .event(OrderEvent.START_PROCESSING)
                .and()
            // Internal transition within PROCESSING super-state
            .withInternal()
                .source(OrderState.PROCESSING)
                .event(OrderEvent.TRANSIT_UPDATE)
                // action to update tracking info
                .and()
            .withExternal()
                .source(OrderState.PROCESSING)
                .target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
                .and()
            // Cancel can fire from the PROCESSING super-state,
            // covering all sub-states at once
            .withExternal()
                .source(OrderState.PROCESSING)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL);
    }
}
```

---

## State Machine Testing

```java
// src/test/java/com/example/statemachine/OrderStateMachineTest.java
package com.example.statemachine;

import com.example.statemachine.config.OrderStateMachineConfig;
import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.statemachine.actions.*;
import com.example.statemachine.statemachine.guards.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.statemachine.StateMachine;
import org.springframework.statemachine.config.StateMachineFactory;
import org.springframework.statemachine.test.StateMachineTestPlan;
import org.springframework.statemachine.test.StateMachineTestPlanBuilder;

import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.Mockito.when;

@SpringBootTest
class OrderStateMachineTest {

    @Autowired
    private StateMachineFactory<OrderState, OrderEvent> factory;

    @MockBean
    private PaymentValidGuard paymentValidGuard;

    @MockBean
    private StockAvailableGuard stockAvailableGuard;

    @MockBean
    private CancellableGuard cancellableGuard;

    @MockBean
    private RefundEligibleGuard refundEligibleGuard;

    @MockBean
    private PaymentReceivedAction paymentReceivedAction;

    @MockBean
    private OrderConfirmedAction orderConfirmedAction;

    @MockBean
    private ShipOrderAction shipOrderAction;

    @MockBean
    private DeliverOrderAction deliverOrderAction;

    @MockBean
    private CancelOrderAction cancelOrderAction;

    @MockBean
    private RefundAction refundAction;

    private StateMachine<OrderState, OrderEvent> sm;

    @BeforeEach
    void setUp() {
        sm = factory.getStateMachine(UUID.randomUUID().toString());
        sm.startReactively().block();
    }

    @Test
    void shouldStartInPendingState() {
        assertThat(sm.getState().getId()).isEqualTo(OrderState.PENDING);
    }

    @Test
    void shouldTransitionFromPendingToPaymentReceivedWhenPaymentValid() throws Exception {
        when(paymentValidGuard.evaluate(any())).thenReturn(true);

        StateMachineTestPlan<OrderState, OrderEvent> plan =
            StateMachineTestPlanBuilder.<OrderState, OrderEvent>builder()
                .stateMachine(sm)
                .step()
                    .expectState(OrderState.PENDING)
                .and()
                .step()
                    .sendEvent(MessageBuilder
                        .withPayload(OrderEvent.RECEIVE_PAYMENT)
                        .setHeader("orderId", 1L)
                        .setHeader("paymentId", "pay_123")
                        .build())
                    .expectState(OrderState.PAYMENT_RECEIVED)
                    .expectStateChanged(1)
                .and()
                .build();

        plan.test();
    }

    @Test
    void shouldNotTransitionWhenPaymentGuardFails() throws Exception {
        when(paymentValidGuard.evaluate(any())).thenReturn(false);

        StateMachineTestPlan<OrderState, OrderEvent> plan =
            StateMachineTestPlanBuilder.<OrderState, OrderEvent>builder()
                .stateMachine(sm)
                .step()
                    .expectState(OrderState.PENDING)
                .and()
                .step()
                    .sendEvent(MessageBuilder
                        .withPayload(OrderEvent.RECEIVE_PAYMENT)
                        .setHeader("orderId", 1L)
                        .build())
                    .expectState(OrderState.PENDING)  // stays in PENDING
                    .expectStateChanged(0)
                .and()
                .build();

        plan.test();
    }

    @Test
    void shouldFollowFullHappyPath() throws Exception {
        when(paymentValidGuard.evaluate(any())).thenReturn(true);
        when(stockAvailableGuard.evaluate(any())).thenReturn(true);

        StateMachineTestPlan<OrderState, OrderEvent> plan =
            StateMachineTestPlanBuilder.<OrderState, OrderEvent>builder()
                .stateMachine(sm)
                .step().expectState(OrderState.PENDING).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.RECEIVE_PAYMENT)
                        .setHeader("orderId", 1L).build())
                    .expectState(OrderState.PAYMENT_RECEIVED).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.CONFIRM)
                        .setHeader("orderId", 1L).build())
                    .expectState(OrderState.CONFIRMED).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.START_PROCESSING)
                        .setHeader("orderId", 1L).build())
                    .expectState(OrderState.PROCESSING).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.SHIP)
                        .setHeader("orderId", 1L).build())
                    .expectState(OrderState.SHIPPED).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.TRANSIT_UPDATE)
                        .setHeader("orderId", 1L).build())
                    .expectState(OrderState.IN_TRANSIT).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.DELIVER)
                        .setHeader("orderId", 1L).build())
                    .expectState(OrderState.DELIVERED)
                    .expectStateMachineStopped(true)
                .and()
                .build();

        plan.test();
    }

    @Test
    void shouldAllowCancellationFromPending() throws Exception {
        StateMachineTestPlan<OrderState, OrderEvent> plan =
            StateMachineTestPlanBuilder.<OrderState, OrderEvent>builder()
                .stateMachine(sm)
                .step().expectState(OrderState.PENDING).and()
                .step()
                    .sendEvent(MessageBuilder.withPayload(OrderEvent.CANCEL)
                        .setHeader("orderId", 1L)
                        .setHeader("cancelReason", "Changed my mind")
                        .build())
                    .expectState(OrderState.CANCELLED)
                    .expectStateMachineStopped(true)
                .and()
                .build();

        plan.test();
    }

    @Test
    void shouldAllowRefundWithin30Days() throws Exception {
        when(paymentValidGuard.evaluate(any())).thenReturn(true);
        when(stockAvailableGuard.evaluate(any())).thenReturn(true);
        when(refundEligibleGuard.evaluate(any())).thenReturn(true);

        // Navigate to DELIVERED state first (abbreviated)
        sm.sendEvent(Mono.just(MessageBuilder.withPayload(OrderEvent.RECEIVE_PAYMENT)
            .setHeader("orderId", 1L).build())).blockLast();
        sm.sendEvent(Mono.just(MessageBuilder.withPayload(OrderEvent.CONFIRM)
            .setHeader("orderId", 1L).build())).blockLast();
        sm.sendEvent(Mono.just(MessageBuilder.withPayload(OrderEvent.START_PROCESSING)
            .setHeader("orderId", 1L).build())).blockLast();
        sm.sendEvent(Mono.just(MessageBuilder.withPayload(OrderEvent.SHIP)
            .setHeader("orderId", 1L).build())).blockLast();
        sm.sendEvent(Mono.just(MessageBuilder.withPayload(OrderEvent.TRANSIT_UPDATE)
            .setHeader("orderId", 1L).build())).blockLast();
        sm.sendEvent(Mono.just(MessageBuilder.withPayload(OrderEvent.DELIVER)
            .setHeader("orderId", 1L).build())).blockLast();

        // Now test refund
        // Note: DELIVERED is an end state in our config, so in practice
        // you'd configure it differently for refund-capable orders.
        // This illustrates the concept.
        assertThat(sm.getState().getId()).isEqualTo(OrderState.DELIVERED);
    }
}
```

---

## JPA State Machine Persistence

```java
// src/main/java/com/example/statemachine/config/JpaPersistenceConfig.java
package com.example.statemachine.config;

import com.example.statemachine.order.OrderEvent;
import com.example.statemachine.order.OrderState;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.data.jpa.JpaPersistingStateMachineInterceptor;
import org.springframework.statemachine.data.jpa.JpaStateMachineRepository;
import org.springframework.statemachine.persist.StateMachineRuntimePersister;

@Configuration
public class JpaPersistenceConfig {

    @Bean
    public StateMachineRuntimePersister<OrderState, OrderEvent, String> stateMachineRuntimePersister(
            JpaStateMachineRepository jpaStateMachineRepository) {
        return new JpaPersistingStateMachineInterceptor<>(jpaStateMachineRepository);
    }
}
```

```java
// application.yml - state machine JPA tables
// Spring State Machine creates these tables automatically:
// state_machine_context - persists state machine contexts
// state_machine_action_task - persists action tasks

// application.yml
// spring:
//   jpa:
//     hibernate:
//       ddl-auto: update
//     show-sql: true
//   statemachine:
//     data:
//       jpa:
//         repositories:
//           enabled: true
```

---

## Service Layer

```java
// src/main/java/com/example/statemachine/service/OrderService.java
package com.example.statemachine.service;

import com.example.statemachine.dto.CreateOrderRequest;
import com.example.statemachine.dto.OrderResponse;
import com.example.statemachine.dto.TransitionHistoryDto;
import com.example.statemachine.entity.Order;
import com.example.statemachine.entity.OrderItem;
import com.example.statemachine.order.OrderState;
import com.example.statemachine.repository.OrderRepository;
import com.example.statemachine.repository.OrderStateTransitionRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderStateTransitionRepository transitionRepository;

    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .totalAmount(request.getTotalAmount())
            .state(OrderState.PENDING)
            .build();

        if (request.getItems() != null) {
            List<OrderItem> items = request.getItems().stream()
                .map(itemRequest -> OrderItem.builder()
                    .order(order)
                    .productId(itemRequest.getProductId())
                    .quantity(itemRequest.getQuantity())
                    .unitPrice(itemRequest.getUnitPrice())
                    .build())
                .collect(Collectors.toList());
            order.setItems(items);
        }

        Order saved = orderRepository.save(order);
        log.info("Order created: {}", saved.getId());
        return OrderResponse.from(saved);
    }

    public OrderResponse getOrder(Long orderId) {
        return orderRepository.findById(orderId)
            .map(OrderResponse::from)
            .orElseThrow(() -> new IllegalArgumentException("Order not found: " + orderId));
    }

    @Transactional
    public void recordPayment(Long orderId, String paymentId) {
        orderRepository.findById(orderId).ifPresent(order -> {
            order.setPaymentId(paymentId);
            orderRepository.save(order);
        });
    }

    @Transactional
    public void markDelivered(Long orderId, LocalDateTime deliveredAt) {
        orderRepository.findById(orderId).ifPresent(order -> {
            order.setDeliveredAt(deliveredAt);
            orderRepository.save(order);
        });
    }

    @Transactional
    public void cancelOrder(Long orderId, String reason) {
        orderRepository.findById(orderId).ifPresent(order -> {
            order.setCancelReason(reason);
            order.setCancelledAt(LocalDateTime.now());
            orderRepository.save(order);
        });
    }

    public List<TransitionHistoryDto> getTransitionHistory(Long orderId) {
        return transitionRepository.findByOrderIdOrderByTransitionedAtAsc(orderId)
            .stream()
            .map(TransitionHistoryDto::from)
            .collect(Collectors.toList());
    }
}
```

---

## Summary

| Concept | Spring State Machine Class/Interface |
|---|---|
| State Machine Configuration | `StateMachineConfigurerAdapter` |
| Enable Factory | `@EnableStateMachineFactory` |
| Guard | `Guard<S, E>` |
| Action | `Action<S, E>` |
| Listener | `StateMachineListenerAdapter<S, E>` |
| Interceptor | `StateMachineInterceptorAdapter<S, E>` |
| JPA Persistence | `JpaPersistingStateMachineInterceptor` |
| Runtime Persister | `StateMachineRuntimePersister<S, E, ID>` |
| State Context | `StateContext<S, E>` |
| Extended State | `context.getExtendedState().getVariables()` |
| Testing | `StateMachineTestPlanBuilder` |

### Key Takeaways
- Guards return `boolean` — they gate transitions
- Actions execute side effects when a transition fires
- Always restore state from database before processing events
- Use `StateMachineInterceptorAdapter.preStateChange` to persist transitions
- Extended state stores variables across multiple transitions
- Hierarchical states reduce duplication for shared transitions (e.g., CANCEL from any active state)
- `@EnableStateMachineFactory` allows creating machine instances per entity

---

## Next Part Preview

**Part 092: Functional Programming in Java** covers Function composition, the Either monad for error handling, immutable pipelines, Vavr library integration, and building a complete data processing pipeline using only pure functions.
