# Part 095: Testing Patterns and Anti-Patterns

## Introduction

Writing tests is easy. Writing tests that provide value, stay maintainable, and give you confidence to refactor — that's a skill. This part covers the patterns that make test suites trustworthy and the anti-patterns that make them a burden.

---

## FIRST Principles

| Principle | Meaning | How to achieve |
|---|---|---|
| **Fast** | Tests run quickly | No I/O, no sleep, mocked dependencies |
| **Isolated** | Tests don't affect each other | No shared mutable state, proper setup/teardown |
| **Repeatable** | Same result every run | No random data, no timestamps, no external services |
| **Self-validating** | Pass/fail without manual inspection | Clear assertions, not just "no exception" |
| **Timely** | Written alongside (or before) production code | TDD or test-first |

---

## Test Anti-Patterns

### Anti-Pattern 1: Mystery Guest

```java
// BAD: "Mystery Guest" - test depends on external setup not visible in test
@Test
void testOrderCalculation() {
    // Where does "order-123" come from? What's in it?
    Order order = orderRepository.findById("order-123");
    BigDecimal total = orderService.calculateTotal(order);
    assertThat(total).isEqualTo(new BigDecimal("199.99"));
}

// GOOD: Everything needed is explicit in the test
@Test
void orderTotalIsCorrectWithTwoItems() {
    Order order = Order.builder()
        .item(new OrderItem("product-a", 2, new BigDecimal("49.99")))
        .item(new OrderItem("product-b", 1, new BigDecimal("99.99")))
        .build();

    BigDecimal total = orderService.calculateTotal(order);

    assertThat(total).isEqualByComparingTo(new BigDecimal("199.97"));
}
```

### Anti-Pattern 2: Over-Mocking

```java
// BAD: Mocking everything - testing nothing meaningful
@Test
void testCreateUser_overMocked() {
    UserRepository mockRepo = mock(UserRepository.class);
    PasswordEncoder mockEncoder = mock(PasswordEncoder.class);
    EmailService mockEmail = mock(EmailService.class);
    UserValidator mockValidator = mock(UserValidator.class);

    when(mockValidator.validate(any())).thenReturn(true);
    when(mockEncoder.encode(any())).thenReturn("encoded");
    when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));

    UserService service = new UserService(mockRepo, mockEncoder, mockEmail, mockValidator);
    User result = service.createUser("alice", "password123");

    assertThat(result).isNotNull();
    // This test tells us nothing about actual business logic
}

// GOOD: Mock only external boundaries; use real collaborators
@Test
void createUserHashesPasswordAndSendsWelcomeEmail() {
    UserRepository mockRepo = mock(UserRepository.class);
    when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
    when(mockRepo.existsByUsername(any())).thenReturn(false);

    // Use real encoder - it IS business logic
    PasswordEncoder realEncoder = new BCryptPasswordEncoder();
    EmailService mockEmail = mock(EmailService.class);

    UserService service = new UserService(mockRepo, realEncoder, mockEmail);
    User result = service.createUser("alice", "password123");

    assertThat(result.getPasswordHash()).isNotEqualTo("password123");
    assertThat(result.getPasswordHash()).startsWith("$2a$");
    verify(mockEmail).sendWelcome("alice");
}
```

### Anti-Pattern 3: Slow Tests (I/O in Unit Tests)

```java
// BAD: Unit test makes HTTP call
@Test
void testPaymentProcessing() throws Exception {
    // Real HTTP call - slow, brittle, requires network
    HttpResponse response = httpClient.post("https://payments.example.com/charge",
        Map.of("amount", "9999"));
    assertThat(response.statusCode()).isEqualTo(200);
}

// GOOD: Mock the HTTP client; test the logic around it
@Test
void paymentProcessingRetries3TimesOnFailure() {
    PaymentGateway mockGateway = mock(PaymentGateway.class);
    when(mockGateway.charge(any()))
        .thenThrow(new TransientException("timeout"))
        .thenThrow(new TransientException("timeout"))
        .thenReturn(PaymentResult.success("txn-123"));

    PaymentService service = new PaymentService(mockGateway, retryPolicy(3));
    PaymentResult result = service.processPayment(Money.usd(99.99));

    assertThat(result.isSuccessful()).isTrue();
    verify(mockGateway, times(3)).charge(any());
}
```

### Anti-Pattern 4: Test Only the Happy Path

```java
// BAD: Only tests success
@Test
void testLogin() {
    User user = userService.login("alice", "correct-password");
    assertThat(user).isNotNull();
}

// GOOD: Tests behavior boundaries
@Test
void loginWithCorrectCredentialsReturnsUser() {
    User user = userService.login("alice", "correct-password");
    assertThat(user.getUsername()).isEqualTo("alice");
}

@Test
void loginWithWrongPasswordThrowsAuthenticationException() {
    assertThatThrownBy(() -> userService.login("alice", "wrong-password"))
        .isInstanceOf(BadCredentialsException.class);
}

@Test
void loginWithLockedAccountThrowsLockedException() {
    userRepository.lockAccount("alice");
    assertThatThrownBy(() -> userService.login("alice", "correct-password"))
        .isInstanceOf(AccountLockedException.class);
}

@Test
void loginWithNonExistentUserThrowsUserNotFoundException() {
    assertThatThrownBy(() -> userService.login("nobody", "any-password"))
        .isInstanceOf(UsernameNotFoundException.class);
}
```

---

## Given-When-Then (BDD) Structure

```java
// src/test/java/com/example/testing/OrderServiceTest.java
package com.example.testing;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@DisplayName("Order Service")
class OrderServiceTest {

    private OrderRepository orderRepository;
    private DiscountService discountService;
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderRepository = mock(OrderRepository.class);
        discountService = mock(DiscountService.class);
        orderService = new OrderService(orderRepository, discountService);
    }

    @Nested
    @DisplayName("When placing a new order")
    class WhenPlacingNewOrder {

        @Test
        @DisplayName("Should save the order and return order ID")
        void shouldSaveOrderAndReturnId() {
            // GIVEN
            Order order = buildOrder("customer-1", "product-1", 2, new BigDecimal("49.99"));
            when(orderRepository.save(any())).thenReturn(order.withId("order-123"));
            when(discountService.getDiscount("customer-1")).thenReturn(BigDecimal.ZERO);

            // WHEN
            String orderId = orderService.placeOrder(order);

            // THEN
            assertThat(orderId).isEqualTo("order-123");
            verify(orderRepository).save(order);
        }

        @Test
        @DisplayName("Should apply customer discount to total")
        void shouldApplyCustomerDiscount() {
            // GIVEN
            Order order = buildOrder("vip-customer", "product-1", 1, new BigDecimal("100.00"));
            when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));
            when(discountService.getDiscount("vip-customer"))
                .thenReturn(new BigDecimal("0.15")); // 15% discount

            // WHEN
            orderService.placeOrder(order);

            // THEN
            verify(orderRepository).save(argThat(savedOrder ->
                savedOrder.getFinalAmount().compareTo(new BigDecimal("85.00")) == 0
            ));
        }

        @ParameterizedTest(name = "quantity={0}, unit price={1}, expected={2}")
        @DisplayName("Should calculate correct totals")
        @CsvSource({
            "1,  100.00, 100.00",
            "3,   25.00,  75.00",
            "10,   9.99,  99.90",
            "2,   49.99,  99.98"
        })
        void shouldCalculateCorrectTotals(int qty, String unitPrice, String expectedTotal) {
            // GIVEN
            Order order = buildOrder("customer-1", "product-1", qty, new BigDecimal(unitPrice));
            when(discountService.getDiscount(any())).thenReturn(BigDecimal.ZERO);
            when(orderRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

            // WHEN
            orderService.placeOrder(order);

            // THEN
            verify(orderRepository).save(argThat(saved ->
                saved.getSubtotal().compareTo(new BigDecimal(expectedTotal)) == 0
            ));
        }
    }

    @Nested
    @DisplayName("When order quantity is invalid")
    class WhenQuantityIsInvalid {

        @Test
        @DisplayName("Should throw exception for zero quantity")
        void shouldThrowForZeroQuantity() {
            // GIVEN
            Order order = buildOrder("customer-1", "product-1", 0, new BigDecimal("10.00"));

            // WHEN + THEN
            assertThatThrownBy(() -> orderService.placeOrder(order))
                .isInstanceOf(InvalidOrderException.class)
                .hasMessage("Quantity must be positive");
        }

        @Test
        @DisplayName("Should throw exception for negative quantity")
        void shouldThrowForNegativeQuantity() {
            Order order = buildOrder("customer-1", "product-1", -1, new BigDecimal("10.00"));

            assertThatThrownBy(() -> orderService.placeOrder(order))
                .isInstanceOf(InvalidOrderException.class);
        }
    }

    private Order buildOrder(String customerId, String productId,
                               int quantity, BigDecimal unitPrice) {
        return Order.builder()
            .customerId(customerId)
            .productId(productId)
            .quantity(quantity)
            .unitPrice(unitPrice)
            .build();
    }
}
```

---

## Test Doubles Classification

```java
// src/test/java/com/example/testing/doubles/TestDoublesDemo.java
package com.example.testing.doubles;

import java.util.ArrayList;
import java.util.List;

/**
 * Five types of test doubles:
 */
public class TestDoublesDemo {

    interface EmailSender {
        void send(String to, String subject, String body);
        List<String> getSentEmails();
    }

    interface PaymentGateway {
        PaymentResult charge(String cardNumber, double amount);
        boolean isAvailable();
    }

    // 1. DUMMY: Passed but never actually used
    static class DummyEmailSender implements EmailSender {
        @Override
        public void send(String to, String subject, String body) {
            // Do nothing - this dependency isn't exercised in this test
        }
        @Override
        public List<String> getSentEmails() { return List.of(); }
    }

    // 2. STUB: Returns pre-programmed answers
    static class StubPaymentGateway implements PaymentGateway {
        @Override
        public PaymentResult charge(String cardNumber, double amount) {
            // Always returns the same canned response
            return new PaymentResult(true, "txn-stub-123", null);
        }
        @Override
        public boolean isAvailable() { return true; }
    }

    // 3. SPY: Records interactions for later verification
    static class SpyEmailSender implements EmailSender {
        private final List<EmailRecord> sent = new ArrayList<>();

        @Override
        public void send(String to, String subject, String body) {
            sent.add(new EmailRecord(to, subject, body));
        }

        @Override
        public List<String> getSentEmails() {
            return sent.stream().map(r -> r.to).toList();
        }

        public boolean wasSentTo(String email) {
            return sent.stream().anyMatch(r -> r.to.equals(email));
        }

        public int getSentCount() { return sent.size(); }

        record EmailRecord(String to, String subject, String body) {}
    }

    // 4. MOCK: Pre-programmed with expectations (verify interactions)
    // Usually created with Mockito: mock(PaymentGateway.class)
    // Difference from spy: mock verifies ALL expectations; spy is more lenient

    // 5. FAKE: Has working implementation, but simplified (unsuitable for production)
    static class FakeEmailSender implements EmailSender {
        private final List<String> inbox = new ArrayList<>();

        @Override
        public void send(String to, String subject, String body) {
            inbox.add(to + "|" + subject + "|" + body);
        }

        @Override
        public List<String> getSentEmails() { return List.copyOf(inbox); }

        // Additional helper for tests
        public boolean hasEmailForRecipient(String email) {
            return inbox.stream().anyMatch(e -> e.startsWith(email + "|"));
        }
    }
}
```

---

## Sociable vs Solitary Unit Tests

```java
// src/test/java/com/example/testing/sociable/SociableVsSolitaryTest.java
package com.example.testing.sociable;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

/**
 * SOLITARY test: mocks ALL collaborators
 * Good for: complex logic with many paths
 * Bad for: testing that collaborators work together correctly
 */
@ExtendWith(MockitoExtension.class)
class SolitaryUserServiceTest {

    @Mock UserRepository userRepository;
    @Mock PasswordEncoder passwordEncoder;
    @Mock EmailValidator emailValidator;

    @InjectMocks UserService userService;

    @Test
    void registerUser_solitaryStyle() {
        when(emailValidator.isValid("alice@example.com")).thenReturn(true);
        when(userRepository.existsByEmail("alice@example.com")).thenReturn(false);
        when(passwordEncoder.encode("password")).thenReturn("$2a$hashed");
        when(userRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        User result = userService.register("alice@example.com", "password");

        assertThat(result.getEmail()).isEqualTo("alice@example.com");
    }
}

/**
 * SOCIABLE test: uses real collaborators where they don't involve I/O
 * Good for: testing that multiple classes work together correctly
 * Bad for: slow if collaborators have I/O
 */
class SociableUserServiceTest {

    @Test
    void registerUser_sociableStyle() {
        // Use real implementations for pure logic
        EmailValidator realValidator = new EmailValidator();     // pure logic
        PasswordEncoder realEncoder = new BCryptPasswordEncoder(); // pure logic

        // Only mock the I/O boundary
        UserRepository mockRepo = mock(UserRepository.class);
        when(mockRepo.existsByEmail(any())).thenReturn(false);
        when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));

        UserService service = new UserService(mockRepo, realEncoder, realValidator);
        User result = service.register("alice@example.com", "password123");

        // Can test real encoding happened
        assertThat(result.getPasswordHash()).startsWith("$2a$");
        assertThat(realEncoder.matches("password123", result.getPasswordHash())).isTrue();
    }
}
```

---

## Integration Test Scope Decisions

```java
// src/test/java/com/example/testing/integration/RepositoryIntegrationTest.java
package com.example.testing.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Repository integration test: tests the adapter against a real database.
 * Uses @DataJpaTest to load only JPA components (no web layer).
 */
@DataJpaTest
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    OrderJpaRepository orderJpaRepository;

    @Test
    void saveAndRetrieveOrder() {
        OrderEntity entity = OrderEntity.builder()
            .customerId("customer-1")
            .amount(new BigDecimal("99.99"))
            .status("PENDING")
            .build();

        OrderEntity saved = orderJpaRepository.save(entity);

        assertThat(saved.getId()).isNotNull();
        assertThat(orderJpaRepository.findById(saved.getId()))
            .isPresent()
            .get()
            .extracting(OrderEntity::getAmount)
            .isEqualTo(new BigDecimal("99.99"));
    }

    @Test
    void findByCustomerIdReturnsAllOrders() {
        orderJpaRepository.save(buildEntity("customer-1", "PENDING"));
        orderJpaRepository.save(buildEntity("customer-1", "SHIPPED"));
        orderJpaRepository.save(buildEntity("customer-2", "PENDING"));

        assertThat(orderJpaRepository.findByCustomerId("customer-1")).hasSize(2);
        assertThat(orderJpaRepository.findByCustomerId("customer-2")).hasSize(1);
        assertThat(orderJpaRepository.findByCustomerId("customer-3")).isEmpty();
    }

    private OrderEntity buildEntity(String customerId, String status) {
        return OrderEntity.builder()
            .customerId(customerId)
            .amount(new BigDecimal("50.00"))
            .status(status)
            .build();
    }
}
```

### REST Layer Integration Test

```java
// src/test/java/com/example/testing/integration/OrderControllerIntegrationTest.java
package com.example.testing.integration;

import com.example.testing.OrderService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.util.Map;

import static org.hamcrest.Matchers.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * @WebMvcTest: loads only web layer, mocks service beans.
 * Fast alternative to @SpringBootTest for controller testing.
 */
@WebMvcTest(OrderController.class)
class OrderControllerIntegrationTest {

    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean OrderService orderService;

    @Test
    @WithMockUser(roles = "USER")
    void createOrderReturns201WithOrderId() throws Exception {
        when(orderService.createOrder(any()))
            .thenReturn(Order.withId("order-created-123"));

        String requestBody = objectMapper.writeValueAsString(Map.of(
            "customerId", "customer-1",
            "productId", "product-1",
            "quantity", 2,
            "unitPrice", "49.99"
        ));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody)
                .with(csrf()))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.orderId").value("order-created-123"))
            .andExpect(header().exists("Location"));
    }

    @Test
    @WithMockUser(roles = "USER")
    void createOrderWithInvalidDataReturns400() throws Exception {
        String invalidBody = objectMapper.writeValueAsString(Map.of(
            "customerId", "",    // blank - invalid
            "quantity", -1       // negative - invalid
        ));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidBody)
                .with(csrf()))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors", hasSize(greaterThan(0))));
    }

    @Test
    void createOrderWithoutAuthReturns401() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}")
                .with(csrf()))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void getOrderReturns404WhenNotFound() throws Exception {
        when(orderService.getOrder("non-existent"))
            .thenThrow(new OrderNotFoundException("non-existent"));

        mockMvc.perform(get("/api/orders/non-existent"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value(containsString("non-existent")));
    }
}
```

---

## Test Coverage That Matters

```java
// src/test/java/com/example/testing/coverage/CoverageExplained.java
package com.example.testing.coverage;

/**
 * Coverage metrics and what they actually mean:
 *
 * Line Coverage: % of lines executed
 * Branch Coverage: % of if/switch branches taken
 * Mutation Coverage: % of artificial bugs (mutations) caught by tests
 *
 * A test suite with 100% line coverage can still miss real bugs:
 */
public class CoverageExplained {

    // This method has 100% line coverage but poor branch coverage:
    public String classify(int score) {
        String grade;
        if (score >= 90) {
            grade = "A";
        } else if (score >= 80) {
            grade = "B";
        } else if (score >= 70) {
            grade = "C";
        } else {
            grade = "F";
        }
        return grade;
    }

    // BAD test: hits all lines but misses most branches
    void badCoverageTest() {
        assert classify(95).equals("A"); // Only tests score >= 90
        // Lines covered: all. Branches covered: 1 of 4 true branches.
    }

    // GOOD tests: cover all branches and boundaries
    void goodCoverageTests() {
        // Each grade boundary
        assert classify(100).equals("A");
        assert classify(90).equals("A");  // boundary
        assert classify(89).equals("B");  // just below boundary
        assert classify(80).equals("B");
        assert classify(79).equals("C");
        assert classify(70).equals("C");
        assert classify(69).equals("F");
        assert classify(0).equals("F");
        assert classify(-1).equals("F");  // edge case
    }
}
```

---

## Living Documentation with Cucumber

```gherkin
# src/test/resources/features/loan-application.feature
Feature: Loan Application Processing

  Background:
    Given the applicant "alice@example.com" exists in the system
    And the credit bureau reports a score of 720 for "alice@example.com"

  Scenario: Successful loan application
    When "alice@example.com" applies for a loan of $10,000 for 60 months
    Then the application status should be "SUBMITTED"
    And a confirmation email should be sent to "alice@example.com"

  Scenario: Fast-track approval for excellent credit
    Given the credit bureau reports a score of 800 for "alice@example.com"
    And the credit bureau reports a DTI of 0.10 for "alice@example.com"
    When "alice@example.com" applies for a loan of $20,000 for 36 months
    Then the application status should be "APPROVED"
    And an approval email should be sent to "alice@example.com"

  Scenario: Rejection for low credit score
    Given the credit bureau reports a score of 500 for "alice@example.com"
    When "alice@example.com" applies for a loan of $10,000 for 60 months
    Then the application status should be "SUBMITTED"
    When an underwriter reviews the application
    Then the application status should be "REJECTED"
    And a rejection email should be sent to "alice@example.com"

  Scenario Outline: Loan amount validation
    When "alice@example.com" applies for a loan of $<amount> for <months> months
    Then the application should fail with "<error>"

    Examples:
      | amount | months | error                          |
      | 500    | 24     | Minimum loan amount is $1,000  |
      | 10000  | 6      | term must be between 12 and 360|
      | 10000  | 400    | term must be between 12 and 360|
```

```java
// src/test/java/com/example/testing/cucumber/LoanApplicationSteps.java
package com.example.testing.cucumber;

import io.cucumber.java.en.*;
import io.cucumber.spring.CucumberContextConfiguration;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@CucumberContextConfiguration
@SpringBootTest
public class LoanApplicationSteps {

    @Autowired ApplyForLoanUseCase applyForLoanUseCase;
    @Autowired ApplicantRepository applicantRepository;
    @Autowired FakeCreditScoreProvider fakeCreditProvider;
    @Autowired SpyEmailSender spyEmailSender;

    private LoanApplication currentApplication;
    private Exception lastException;

    @Given("the applicant {string} exists in the system")
    public void applicantExists(String email) {
        applicantRepository.save(Applicant.builder()
            .email(email)
            .name("Test User")
            .build());
    }

    @Given("the credit bureau reports a score of {int} for {string}")
    public void creditBureauReportsScore(int score, String email) {
        fakeCreditProvider.setScore(email, score);
    }

    @Given("the credit bureau reports a DTI of {double} for {string}")
    public void creditBureauReportsDTI(double dti, String email) {
        fakeCreditProvider.setDTI(email, dti);
    }

    @When("{string} applies for a loan of ${int} for {int} months")
    public void appliesForLoan(String email, int amount, int months) {
        Applicant applicant = applicantRepository.findByEmail(email).orElseThrow();
        try {
            var command = new ApplyForLoanUseCase.ApplyForLoanCommand(
                applicant.getId(),
                BigDecimal.valueOf(amount),
                "USD",
                months,
                "Personal use"
            );
            currentApplication = applyForLoanUseCase.apply(command);
        } catch (Exception e) {
            lastException = e;
        }
    }

    @Then("the application status should be {string}")
    public void applicationStatusShouldBe(String expectedStatus) {
        assertThat(currentApplication).isNotNull();
        assertThat(currentApplication.getStatus().name()).isEqualTo(expectedStatus);
    }

    @Then("a confirmation email should be sent to {string}")
    public void confirmationEmailSentTo(String email) {
        assertThat(spyEmailSender.wasSentTo(email)).isTrue();
        assertThat(spyEmailSender.getLastSubjectFor(email))
            .contains("Application Received");
    }

    @Then("the application should fail with {string}")
    public void applicationShouldFailWith(String errorMessage) {
        assertThat(lastException).isNotNull();
        assertThat(lastException.getMessage()).contains(errorMessage);
    }
}
```

---

## Approval Testing

```java
// src/test/java/com/example/testing/approval/InvoiceApprovalTest.java
package com.example.testing.approval;

import org.approvaltests.Approvals;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;

/**
 * Approval testing: compare output to a pre-approved "golden file".
 * Great for complex text/HTML/JSON outputs where writing assertions is tedious.
 * First run: creates the approved file. Subsequent runs: compare to it.
 */
class InvoiceApprovalTest {

    private final InvoiceGenerator generator = new InvoiceGenerator();

    @Test
    void invoiceFormatIsCorrect() {
        Invoice invoice = Invoice.builder()
            .invoiceNumber("INV-2024-001")
            .customerName("Acme Corporation")
            .customerEmail("billing@acme.com")
            .issueDate(LocalDate.of(2024, 1, 15))
            .dueDate(LocalDate.of(2024, 2, 15))
            .items(List.of(
                new InvoiceItem("Consulting Services", 5, new BigDecimal("150.00")),
                new InvoiceItem("Software License", 1, new BigDecimal("999.00"))
            ))
            .build();

        String output = generator.generateText(invoice);

        // On first run: creates InvoiceApprovalTest.invoiceFormatIsCorrect.approved.txt
        // On subsequent runs: compares against that file
        Approvals.verify(output);
    }

    @Test
    void invoiceJsonOutputIsStable() {
        Invoice invoice = buildSampleInvoice();
        String json = generator.generateJson(invoice);

        Approvals.verifyJson(json);  // pretty-prints before comparison
    }
}
```

---

## Refactoring Poorly-Tested Code

### Before: Untestable Code

```java
// BEFORE: Hard to test due to static calls, new operators, and side effects
public class OrderProcessor {

    public void processOrder(String orderId) {
        // Static call - cannot be mocked
        Order order = Database.getOrder(orderId);

        // 'new' inside business logic - cannot inject test doubles
        PaymentProcessor processor = new PaymentProcessor("prod-api-key");

        // Side effect mixed with business logic
        if (order.getTotal() > 1000) {
            EmailSender.sendEmail(order.getCustomerEmail(), "Large order notification");
        }

        boolean paid = processor.charge(order.getCustomerEmail(), order.getTotal());

        if (paid) {
            // Side effect
            Database.updateOrderStatus(orderId, "PAID");
            Logger.log("Order " + orderId + " paid");
        } else {
            // Side effect
            Database.updateOrderStatus(orderId, "PAYMENT_FAILED");
        }
    }
}
```

### After: Testable Code

```java
// AFTER: Properly structured for testing
public class OrderProcessor {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;

    // Constructor injection - all dependencies can be replaced in tests
    public OrderProcessor(OrderRepository orderRepository,
                          PaymentGateway paymentGateway,
                          NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
    }

    public ProcessingResult processOrder(String orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        if (order.getTotal().compareTo(new BigDecimal("1000")) > 0) {
            notificationService.notifyLargeOrder(order);
        }

        PaymentResult result = paymentGateway.charge(
            order.getCustomerEmail(), order.getTotal()
        );

        if (result.isSuccessful()) {
            order.markAsPaid(result.getTransactionId());
            orderRepository.save(order);
            return ProcessingResult.success(result.getTransactionId());
        } else {
            order.markAsPaymentFailed(result.getErrorCode());
            orderRepository.save(order);
            return ProcessingResult.failure(result.getErrorCode());
        }
    }
}
```

```java
// Tests for the refactored code
class OrderProcessorTest {

    private OrderRepository mockRepo;
    private PaymentGateway mockGateway;
    private NotificationService mockNotification;
    private OrderProcessor processor;

    @BeforeEach
    void setUp() {
        mockRepo = mock(OrderRepository.class);
        mockGateway = mock(PaymentGateway.class);
        mockNotification = mock(NotificationService.class);
        processor = new OrderProcessor(mockRepo, mockGateway, mockNotification);
    }

    @Test
    void successfulPaymentMarkOrderAsPaid() {
        Order order = Order.builder()
            .id("order-1")
            .customerEmail("alice@example.com")
            .total(new BigDecimal("150.00"))
            .build();

        when(mockRepo.findById("order-1")).thenReturn(Optional.of(order));
        when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(mockGateway.charge(any(), any()))
            .thenReturn(PaymentResult.success("txn-abc"));

        ProcessingResult result = processor.processOrder("order-1");

        assertThat(result.isSuccessful()).isTrue();
        assertThat(result.getTransactionId()).isEqualTo("txn-abc");
        verify(mockRepo).save(argThat(o -> o.getStatus() == OrderStatus.PAID));
    }

    @Test
    void largeOrderTriggersNotification() {
        Order order = Order.builder()
            .id("order-2")
            .customerEmail("big@corp.com")
            .total(new BigDecimal("1500.00"))
            .build();

        when(mockRepo.findById("order-2")).thenReturn(Optional.of(order));
        when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(mockGateway.charge(any(), any()))
            .thenReturn(PaymentResult.success("txn-xyz"));

        processor.processOrder("order-2");

        verify(mockNotification).notifyLargeOrder(order);
    }

    @Test
    void failedPaymentMarkOrderAsPaymentFailed() {
        Order order = Order.builder()
            .id("order-3")
            .customerEmail("broke@example.com")
            .total(new BigDecimal("200.00"))
            .build();

        when(mockRepo.findById("order-3")).thenReturn(Optional.of(order));
        when(mockRepo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(mockGateway.charge(any(), any()))
            .thenReturn(PaymentResult.failure("INSUFFICIENT_FUNDS"));

        ProcessingResult result = processor.processOrder("order-3");

        assertThat(result.isSuccessful()).isFalse();
        assertThat(result.getErrorCode()).isEqualTo("INSUFFICIENT_FUNDS");
        verify(mockRepo).save(argThat(o -> o.getStatus() == OrderStatus.PAYMENT_FAILED));
    }

    @Test
    void orderNotFoundThrowsException() {
        when(mockRepo.findById("missing")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> processor.processOrder("missing"))
            .isInstanceOf(OrderNotFoundException.class)
            .hasMessageContaining("missing");
    }
}
```

---

## Summary

| Anti-Pattern | Problem | Solution |
|---|---|---|
| Mystery Guest | Setup hidden from test reader | Make all data explicit in test |
| Over-mocking | Tests verify nothing real | Mock only I/O boundaries |
| Slow tests | Developers skip them | No network/disk in unit tests |
| Happy-path only | Misses real bugs | Test boundaries, errors, edge cases |
| Testing implementation | Tests break on refactor | Test behavior, not internals |
| Fragile ordering | Tests fail in different order | Isolate state between tests |
| `assertTrue(result != null)` | Not self-validating | `assertThat(result).isNotNull()` |

### Key Takeaways
- Sociable tests (real collaborators, mocked I/O) find more bugs than solitary tests alone
- Given-When-Then structure makes intent clear and test failures readable
- Mocks are best for I/O boundaries (DB, HTTP, email); use fakes for complex dependencies
- Coverage matters at the business logic level — 100% coverage of trivial getters is noise
- Cucumber keeps tests as living documentation that non-developers can read

---

## Next Part Preview

**Part 096: Spring Data REST** covers auto-exposed repositories as REST APIs, projections, event handlers, HATEOAS links, and custom search endpoints — with a complete blog API built in minimal code.
