# Part 098: Performance Testing with JMeter, Gatling, and k6

## เนื้อหาในส่วนนี้
- Performance Testing Concepts (Load, Stress, Spike, Soak)
- JMeter for REST API load testing
- Gatling with Scala/Java DSL
- k6 for modern load testing
- Analyzing results and bottlenecks
- Continuous performance testing in CI/CD

---

## 1. Performance Testing Types

```
Load Test:     Simulate expected traffic (e.g., 100 concurrent users)
Stress Test:   Find the breaking point (ramp up until failure)
Spike Test:    Sudden traffic spike (10 → 500 → 10 users instantly)
Soak Test:     Sustained load over time (6-24 hours for memory leaks)
Volume Test:   Large amounts of data (millions of records)

Key Metrics:
- Response Time (p50, p95, p99 percentiles)
- Throughput (requests/second, transactions/second)
- Error Rate (% of failed requests)
- Concurrent Users (virtual users)
- CPU/Memory during load
```

---

## 2. Gatling Load Testing (Java DSL)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.10.5</version>
    <scope>test</scope>
</dependency>

<plugin>
    <groupId>io.gatling</groupId>
    <artifactId>gatling-maven-plugin</artifactId>
    <version>4.9.6</version>
    <configuration>
        <simulationClass>com.myapp.perf.OrderSimulation</simulationClass>
    </configuration>
</plugin>
```

```java
import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class OrderSimulation extends Simulation {
    
    // HTTP config
    private final HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json")
        .userAgentHeader("Gatling/3.10")
        .shareConnections();  // Reuse connections (like real browsers)
    
    // Test data feeder (CSV)
    private final FeederBuilder<String> userFeeder = 
        csv("test-users.csv").circular();
    // test-users.csv:
    // username,password
    // user1@test.com,Password1!
    // user2@test.com,Password2!
    
    // Scenario: authenticate and create orders
    private final ScenarioBuilder orderScenario = scenario("Order Creation")
        .feed(userFeeder)
        
        // Step 1: Login
        .exec(http("Login")
            .post("/api/auth/login")
            .body(StringBody("""
                {"email": "#{username}", "password": "#{password}"}
                """))
            .check(status().is(200))
            .check(jsonPath("$.accessToken").saveAs("token"))
        )
        .pause(1, 3)  // Think time between actions
        
        // Step 2: Browse products
        .exec(http("List Products")
            .get("/api/products?page=0&size=20")
            .header("Authorization", "Bearer #{token}")
            .check(status().is(200))
            .check(jsonPath("$.content[0].id").saveAs("productId"))
        )
        .pause(2, 5)
        
        // Step 3: Get product details
        .exec(http("Product Detail")
            .get("/api/products/#{productId}")
            .header("Authorization", "Bearer #{token}")
            .check(status().is(200))
        )
        .pause(1, 2)
        
        // Step 4: Create order
        .exec(http("Create Order")
            .post("/api/orders")
            .header("Authorization", "Bearer #{token}")
            .body(StringBody("""
                {
                    "items": [{"productId": "#{productId}", "quantity": 1}],
                    "shippingAddress": "123 Main St, Bangkok 10110"
                }
                """))
            .check(status().is(201))
            .check(jsonPath("$.orderId").saveAs("orderId"))
        );
    
    // Scenario: read-heavy (product browsing)
    private final ScenarioBuilder browseScenario = scenario("Product Browsing")
        .exec(http("Browse Products")
            .get("/api/products?category=electronics&page=0&size=20")
            .check(status().is(200))
        )
        .pause(1)
        .exec(http("Search Products")
            .get("/api/products/search?q=laptop")
            .check(status().is(200))
        );
    
    // Load profiles
    {
        setUp(
            // Ramp up: 10 users over 60s, then hold 50 users for 5 minutes
            orderScenario.injectOpen(
                rampUsers(10).during(30),       // Warm up
                constantUsersPerSec(5).during(300)  // Steady state
            ),
            
            // Browse scenario: higher concurrency
            browseScenario.injectOpen(
                constantUsersPerSec(20).during(300)
            )
        )
        .protocols(httpProtocol)
        .assertions(
            global().responseTime().percentile(95).lt(500),  // p95 < 500ms
            global().successfulRequests().percent().gt(99.0), // 99% success
            forAll().failedRequests().percent().lt(1.0)       // < 1% errors
        );
    }
}
```

---

## 3. Gatling Scenarios for Complex Flows

```java
public class ECommerceSimulation extends Simulation {
    
    private final HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");
    
    // Spike test: sudden 10x traffic spike
    private final ScenarioBuilder flashSaleScenario = scenario("Flash Sale Spike")
        .exec(http("Get Flash Sale Product")
            .get("/api/flash-sale/current")
            .check(status().is(200))
            .check(jsonPath("$.productId").saveAs("flashProductId"))
        )
        .exec(http("Purchase Flash Sale Item")
            .post("/api/flash-sale/purchase")
            .body(StringBody("""
                {"productId": "#{flashProductId}", "quantity": 1}
                """))
            // Acceptable to get 200 (success) or 409 (sold out)
            .check(status().in(200, 409))
        );
    
    // Soak test profile (24-hour simulation condensed)
    private final ScenarioBuilder soakScenario = scenario("Soak Test")
        .forever().on(
            exec(http("Health Check")
                .get("/actuator/health/readiness")
                .check(status().is(200))
            )
            .pause(10)
        );
    
    {
        // Spike test
        setUp(
            flashSaleScenario.injectOpen(
                atOnceUsers(5),           // Baseline
                nothingFor(10),
                atOnceUsers(500),         // Spike!
                nothingFor(30),
                rampUsers(10).during(60)  // Recovery
            )
        ).protocols(httpProtocol)
         .assertions(
            global().responseTime().max().lt(2000),  // Even during spike < 2s
            global().successfulRequests().percent().gt(95.0)
         );
    }
}
```

---

## 4. k6 Load Testing Script

```javascript
// order-load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const BASE_URL = 'http://localhost:8080';

// Custom metrics
const orderCreationTime = new Trend('order_creation_time', true);
const errorRate = new Rate('error_rate');

// Load stages
export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Warm up
    { duration: '2m',  target: 50 },   // Ramp up
    { duration: '5m',  target: 50 },   // Steady state
    { duration: '1m',  target: 100 },  // Peak load
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
    order_creation_time: ['p(95)<1000'],
  },
};

// Setup: create test users (runs once)
export function setup() {
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: 'loadtest@example.com',
    password: 'LoadTest123!'
  }), { headers: { 'Content-Type': 'application/json' } });
  
  return { token: loginRes.json('accessToken') };
}

// Main virtual user flow
export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };
  
  group('Browse and Order', () => {
    // Step 1: List products
    group('List Products', () => {
      const res = http.get(`${BASE_URL}/api/products?size=20`, { headers });
      check(res, {
        'products returned 200': (r) => r.status === 200,
        'has products': (r) => r.json('content').length > 0,
      });
      errorRate.add(res.status !== 200);
      sleep(1);
    });
    
    // Step 2: Create order
    group('Create Order', () => {
      const start = Date.now();
      const res = http.post(`${BASE_URL}/api/orders`, JSON.stringify({
        items: [{ productId: 'PROD-001', quantity: 1 }],
        shippingAddress: '123 Test St'
      }), { headers });
      
      orderCreationTime.add(Date.now() - start);
      check(res, {
        'order created': (r) => r.status === 201,
        'has orderId': (r) => r.json('orderId') !== undefined,
      });
      errorRate.add(res.status !== 201);
    });
  });
  
  sleep(Math.random() * 2 + 1);  // 1-3s think time
}

// Teardown
export function teardown(data) {
  console.log('Test completed');
}
```

```bash
# Run k6
k6 run order-load-test.js

# With dashboard
k6 run --out dashboard order-load-test.js

# With InfluxDB for Grafana
k6 run --out influxdb=http://localhost:8086/k6 order-load-test.js

# Stress test: find breaking point
k6 run --vus 200 --duration 60s order-load-test.js
```

---

## 5. Analyzing Results and Bottlenecks

```java
// Add custom metrics to Spring Boot endpoints
import io.micrometer.core.instrument.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final Timer orderCreationTimer;
    private final Counter orderErrorCounter;
    private final DistributionSummary orderValueSummary;
    
    public OrderController(MeterRegistry registry) {
        this.orderCreationTimer = Timer.builder("order.creation.time")
            .description("Time to create an order")
            .percentiles(0.5, 0.95, 0.99)
            .sla(Duration.ofMillis(100), Duration.ofMillis(500), Duration.ofSeconds(1))
            .register(registry);
            
        this.orderErrorCounter = Counter.builder("order.errors.total")
            .tag("type", "creation")
            .register(registry);
            
        this.orderValueSummary = DistributionSummary.builder("order.value")
            .description("Order value in USD")
            .register(registry);
    }
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        return orderCreationTimer.record(() -> {
            try {
                Order order = orderService.createOrder(request);
                orderValueSummary.record(order.getTotalAmount().doubleValue());
                return ResponseEntity.status(HttpStatus.CREATED).body(order);
            } catch (Exception e) {
                orderErrorCounter.increment();
                throw e;
            }
        });
    }
}

import java.time.Duration;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
```

---

## 6. CI/CD Performance Gate

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  push:
    branches: [main, develop]

jobs:
  perf-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
      redis:
        image: redis:7
        ports: ["6379:6379"]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Build and start app
        run: |
          ./mvnw package -DskipTests
          java -jar target/app.jar &
          sleep 30  # Wait for startup
          curl --retry 10 --retry-delay 3 http://localhost:8080/actuator/health
      
      - name: Run Gatling tests
        run: ./mvnw gatling:test -Dgatling.simulationClass=com.myapp.perf.SmokeSimulation
      
      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
            --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | \
            sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update && sudo apt-get install k6
      
      - name: Run k6 smoke test
        run: k6 run --vus 5 --duration 30s k6/smoke-test.js
      
      - name: Publish Gatling Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: gatling-report
          path: target/gatling/
```

---

## สรุป Part 098

| Tool | Best For | Language | Output |
|------|---------|----------|--------|
| Gatling | Complex scenarios, HTML reports | Java/Scala | HTML dashboard |
| k6 | Developer-friendly, CI/CD | JavaScript | JSON/InfluxDB |
| JMeter | GUI, enterprise, legacy | GUI/XML | Reports |
| Locust | Python teams, distributed | Python | Web UI |

**Performance Testing Checklist:**
- [ ] Define SLA (p95 < 500ms, 99.9% uptime)
- [ ] Baseline with 10 users
- [ ] Load test with expected users
- [ ] Stress test to find breaking point
- [ ] Soak test for 6+ hours
- [ ] Identify and fix bottlenecks
- [ ] Add to CI/CD pipeline

---

**Part 099:** Microservices Security - Zero Trust, mTLS, service-to-service auth
