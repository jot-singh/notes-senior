# Testing, Contracts, and Automation

## What You'll Learn
Comprehensive testing strategies for REST APIs covering unit tests, integration tests, contract tests, performance tests, and test automation. You'll understand the testing pyramid, when to use each test type, and how to prevent breaking changes in distributed systems.

## Why This Matters
Untested APIs break in production. Manual testing doesn't scale. Contract violations break client integrations. Performance regressions affect user experience. As a senior developer, you're responsible for building confidence in releases through automated testing. Contract tests prevent microservices from breaking each other. Performance tests catch issues before they hit production. Good test automation means deploying multiple times per day safely.

## Testing Pyramid for APIs

### The Pyramid Structure

```
        /\         E2E Tests (Few)
       /  \        - Critical user journeys
      /    \       - Cross-service flows
     /------\      - Expensive, slow, brittle
    /        \    
   /  Contract \   Contract Tests (Some)
  /    Tests    \  - API boundaries
 /--------------\ - Provider/consumer
/                \ 
/  Integration    \ Integration Tests (More)
/     Tests       \ - DB, external services
/------------------\ - Service boundaries
/                    \
/    Unit Tests      \ Unit Tests (Most)
/                    \ - Business logic
/____________________\ - Fast, isolated
```

### 1. Unit Tests

**Scope**: Single function/class; no I/O
**Speed**: Milliseconds
**Purpose**: Test business logic in isolation

**Example** (Java with JUnit):
```java
@Test
void shouldCalculateOrderTotal() {
    // Arrange
    Order order = new Order();
    order.addItem(new OrderItem("item-1", 10.00, 2));  // $20
    order.addItem(new OrderItem("item-2", 5.00, 3));   // $15
    order.setTaxRate(0.10);  // 10% tax
    
    // Act
    BigDecimal total = order.calculateTotal();
    
    // Assert
    assertEquals(new BigDecimal("38.50"), total);  // 35 + 3.50 tax
}

@Test
void shouldRejectInvalidStatusTransition() {
    // Arrange
    Order order = new Order(OrderStatus.SHIPPED);
    
    // Act & Assert
    assertThrows(InvalidStateException.class, () -> {
        order.transitionTo(OrderStatus.PENDING);
    });
}
```

**Best Practices**:
- Mock external dependencies (DB, HTTP clients, queues)
- Test edge cases and error conditions
- One assertion per test (mostly)
- Fast: entire suite < 1 minute

### 2. Integration Tests

**Scope**: Multiple components; real I/O
**Speed**: Seconds
**Purpose**: Test integration with databases, external services

**Example with Testcontainers** (Spring Boot):
```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void shouldPersistAndRetrieveOrder() {
        // Arrange
        Order order = new Order("cust-123", OrderStatus.PENDING);
        
        // Act
        Order saved = orderRepository.save(order);
        Order retrieved = orderRepository.findById(saved.getId()).orElseThrow();
        
        // Assert
        assertNotNull(retrieved.getId());
        assertEquals("cust-123", retrieved.getCustomerId());
        assertEquals(OrderStatus.PENDING, retrieved.getStatus());
    }
    
    @Test
    void shouldFindOrdersByCustomer() {
        // Arrange
        orderRepository.save(new Order("cust-123", OrderStatus.PENDING));
        orderRepository.save(new Order("cust-123", OrderStatus.SHIPPED));
        orderRepository.save(new Order("cust-456", OrderStatus.PENDING));
        
        // Act
        List<Order> orders = orderRepository.findByCustomerId("cust-123");
        
        // Assert
        assertEquals(2, orders.size());
    }
}
```

**External Service Testing**:
```java
@Test
void shouldCallPaymentGateway() {
    // Use WireMock to stub external API
    stubFor(post(urlEqualTo("/payments"))
        .willReturn(aResponse()
            .withStatus(200)
            .withHeader("Content-Type", "application/json")
            .withBody("{\"transactionId\": \"txn-123\", \"status\": \"approved\"}")));
    
    PaymentResult result = paymentService.charge(new ChargeRequest(100.00, "USD"));
    
    assertEquals("txn-123", result.getTransactionId());
    assertEquals(PaymentStatus.APPROVED, result.getStatus());
    
    verify(postRequestedFor(urlEqualTo("/payments"))
        .withHeader("Authorization", matching("Bearer .*")));
}
```

### 3. Contract Tests

**What**: Consumer-driven contracts; verify API compatibility without E2E tests.

**Problem Without Contract Tests**:
```
Order Service changes response:
- Old: {"customer_id": "123"}
- New: {"customerId": "123"}  // Breaking change!

Shipping Service (consumer) breaks silently
Discovered in production 💥
```

**Solution: Pact Contract Testing**

**Consumer Side** (Shipping Service tests):
```java
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "OrderService")
class OrderServiceContractTest {
    
    @Pact(consumer = "ShippingService")
    public RequestResponsePact createOrderPact(PactDslWithProvider builder) {
        return builder
            .given("order ord-123 exists")
            .uponReceiving("a request for order details")
            .path("/orders/ord-123")
            .method("GET")
            .willRespondWith()
            .status(200)
            .body(new PactDslJsonBody()
                .stringType("id", "ord-123")
                .stringType("customerId", "cust-456")  // Expect customerId!
                .stringType("status", "shipped")
                .numberType("total", 99.99))
            .toPact();
    }
    
    @Test
    @PactTestFor
    void testGetOrder(MockServer mockServer) {
        OrderClient client = new OrderClient(mockServer.getUrl());
        Order order = client.getOrder("ord-123");
        
        assertEquals("ord-123", order.getId());
        assertEquals("cust-456", order.getCustomerId());  // This field must exist
    }
}
```

**Provider Side** (Order Service verification):
```java
@Provider("OrderService")
@PactFolder("pacts")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderServiceContractVerificationTest {
    
    @LocalServerPort
    private int port;
    
    @BeforeEach
    void setup(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }
    
    @State("order ord-123 exists")
    void orderExists() {
        // Setup test data
        orderRepository.save(new Order("ord-123", "cust-456", OrderStatus.SHIPPED, 99.99));
    }
}
```

**If Order Service changes `customerId` to `customer_id`**:
- Provider verification fails
- CI blocks deployment
- Breaking change caught before production! ✅

### 4. API Tests (REST Assured)

**Black-box testing against running service**:

```java
@Test
void shouldCreateOrder() {
    given()
        .contentType("application/json")
        .header("Authorization", "Bearer " + token)
        .body("""
            {
                "customerId": "cust-123",
                "items": [{"productId": "prod-456", "quantity": 2}]
            }
            """)
    .when()
        .post("/orders")
    .then()
        .statusCode(201)
        .header("Location", matchesPattern(".*/orders/ord-.*"))
        .body("id", notNullValue())
        .body("status", equalTo("pending"))
        .body("total", greaterThan(0f));
}

@Test
void shouldReturn422ForInvalidOrder() {
    given()
        .contentType("application/json")
        .header("Authorization", "Bearer " + token)
        .body("{\"items\": []}")
    .when()
        .post("/orders")
    .then()
        .statusCode(422)
        .contentType("application/problem+json")
        .body("type", containsString("/errors/validation-error"))
        .body("status", equalTo(422))
        .body("errors", hasSize(greaterThan(0)));
}
```

### 5. End-to-End Tests

**Scope**: Full system; all services running
**Speed**: Minutes
**Purpose**: Critical user journeys

**Example** (Playwright for API):
```javascript
test('complete order flow', async ({ request }) => {
  // 1. Create customer
  const customerResponse = await request.post('/customers', {
    data: { email: 'test@example.com', name: 'Test User' }
  });
  const customer = await customerResponse.json();
  
  // 2. Create order
  const orderResponse = await request.post('/orders', {
    data: {
      customerId: customer.id,
      items: [{ productId: 'prod-123', quantity: 1 }]
    },
    headers: { 'Idempotency-Key': crypto.randomUUID() }
  });
  expect(orderResponse.status()).toBe(201);
  const order = await orderResponse.json();
  
  // 3. Process payment
  const paymentResponse = await request.post(`/orders/${order.id}/payment`, {
    data: { method: 'credit_card', token: 'tok_visa' }
  });
  expect(paymentResponse.ok()).toBeTruthy();
  
  // 4. Verify order status
  const statusResponse = await request.get(`/orders/${order.id}`);
  const updatedOrder = await statusResponse.json();
  expect(updatedOrder.status).toBe('paid');
});
```

**When to Use**: Critical paths only (checkout, signup, password reset)

## Performance Testing

### Load Testing with k6

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '1m', target: 100 },  // Ramp to 100
    { duration: '5m', target: 100 },  // Stay at 100
    { duration: '1m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
    errors: ['rate<0.05'],             // Business errors < 5%
  },
};

export default function () {
  const payload = JSON.stringify({
    customerId: `cust-${__VU}`,
    items: [{ productId: 'prod-123', quantity: 1 }]
  });
  
  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.API_TOKEN}`,
      'Idempotency-Key': `${__VU}-${__ITER}`,
    },
  };
  
  const response = http.post('https://api.example.com/orders', payload, params);
  
  const success = check(response, {
    'status is 201': (r) => r.status === 201,
    'has order id': (r) => r.json('id') !== undefined,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  errorRate.add(!success);
  
  sleep(1);  // Think time
}
```

**Run**:
```bash
k6 run --vus 100 --duration 10m load-test.js
```

**Metrics to Track**:
- Latency percentiles (p50, p95, p99)
- Throughput (requests/sec)
- Error rate
- Resource utilization (CPU, memory)

### Chaos Testing

**Test resilience by injecting failures**:

```javascript
// Test retry logic
test('should retry on 503', async () => {
  let attempts = 0;
  
  server.use(
    http.post('/orders', () => {
      attempts++;
      if (attempts < 3) {
        return new HttpResponse(null, { status: 503 });
      }
      return HttpResponse.json({ id: 'ord-123' }, { status: 201 });
    })
  );
  
  const result = await orderClient.createOrder(orderData);
  
  expect(attempts).toBe(3);
  expect(result.id).toBe('ord-123');
});

// Test timeout handling
test('should timeout after 5 seconds', async () => {
  server.use(
    http.post('/orders', async () => {
      await delay(10000);  // 10s delay
      return HttpResponse.json({ id: 'ord-123' });
    })
  );
  
  await expect(orderClient.createOrder(orderData))
    .rejects.toThrow('Request timeout');
});
```

## Test Automation

### CI Pipeline

```yaml
# .github/workflows/api-tests.yml
name: API Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Unit Tests
        run: ./mvnw test
      
      - name: Integration Tests
        run: ./mvnw verify -P integration-tests
      
      - name: Contract Tests (Consumer)
        run: ./mvnw test -P contract-tests
      
      - name: Publish Pacts
        if: github.ref == 'refs/heads/main'
        run: ./mvnw pact:publish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
  
  contract-verification:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Verify Provider Contracts
        run: ./mvnw test -P pact-verification
  
  performance:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Run Load Tests
        run: k6 run --out json=results.json load-test.js
      
      - name: Check Thresholds
        run: |
          if grep -q '"thresholds":{.*"ok":false' results.json; then
            echo "Performance thresholds failed"
            exit 1
          fi
```

### Schema Validation

**Validate requests/responses against OpenAPI**:

```java
@Test
void shouldMatchOpenAPISchema() {
    given()
        .filter(new OpenApiValidationFilter("openapi.yaml"))
        .contentType("application/json")
        .body(orderRequest)
    .when()
        .post("/orders")
    .then()
        .statusCode(201);
    
    // Filter automatically validates request and response against spec
}
```

## Best Practices

1. **Test Isolation**: Each test should be independent
2. **Data Cleanup**: Reset state between tests
3. **Parallel Execution**: Tests should be thread-safe
4. **Meaningful Assertions**: Test behavior, not implementation
5. **Fast Feedback**: Unit tests in seconds, full suite in minutes
6. **Flake-Free**: No random failures; deterministic tests
7. **Coverage Metrics**: Aim for 80%+ for business logic
8. **Contract-First**: Generate tests from OpenAPI specs
