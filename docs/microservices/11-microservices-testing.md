# Microservices Testing

## Overview
Testing microservices is more complex than testing monoliths due to distributed nature, network boundaries, and service dependencies. A comprehensive testing strategy includes unit tests, integration tests, contract tests, and end-to-end tests, following the testing pyramid principle.

---

## Testing Pyramid for Microservices

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TESTING PYRAMID                                   │
│                                                                      │
│                          ▲                                          │
│                         /│\                                         │
│                        / │ \          E2E Tests                     │
│                       /  │  \         (Few, Slow, Expensive)        │
│                      /   │   \        Test full user journeys       │
│                     /    │    \                                     │
│                    ───────────────                                  │
│                   /      │      \     Component/Integration         │
│                  /       │       \    Tests (Medium)                │
│                 /        │        \   Test service with deps        │
│                /         │         \                                │
│               /          │          \                               │
│              ─────────────────────────                              │
│             /            │            \   Contract Tests            │
│            /             │             \  (API compatibility)       │
│           ─────────────────────────────────                         │
│          /               │               \   Unit Tests             │
│         /                │                \  (Many, Fast, Cheap)    │
│        /                 │                 \ Test business logic    │
│       ─────────────────────────────────────────                     │
│                                                                      │
│   Coverage Goals:                                                   │
│   • Unit Tests: 70-80%                                              │
│   • Integration Tests: 15-20%                                       │
│   • Contract Tests: 5-10%                                           │
│   • E2E Tests: 1-5%                                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Unit Testing

### Testing Business Logic

```java
// Service class
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PricingService pricingService;
    private final InventoryClient inventoryClient;
    
    public Order createOrder(OrderRequest request) {
        // Validate
        validateRequest(request);
        
        // Calculate price
        BigDecimal totalPrice = pricingService.calculateTotal(request.getItems());
        
        // Create order
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(request.getItems())
            .totalPrice(totalPrice)
            .status(OrderStatus.PENDING)
            .build();
        
        return orderRepository.save(order);
    }
    
    private void validateRequest(OrderRequest request) {
        if (request.getItems().isEmpty()) {
            throw new InvalidOrderException("Order must have at least one item");
        }
    }
}
```

```java
// Unit test with Mockito
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private PricingService pricingService;
    
    @Mock
    private InventoryClient inventoryClient;
    
    @InjectMocks
    private OrderService orderService;
    
    @Captor
    private ArgumentCaptor<Order> orderCaptor;
    
    @Test
    void createOrder_ShouldCreatePendingOrder() {
        // Arrange
        OrderRequest request = OrderRequest.builder()
            .customerId("customer-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();
        
        BigDecimal expectedPrice = new BigDecimal("99.99");
        when(pricingService.calculateTotal(any())).thenReturn(expectedPrice);
        when(orderRepository.save(any())).thenAnswer(inv -> {
            Order order = inv.getArgument(0);
            order.setId("order-123");
            return order;
        });
        
        // Act
        Order result = orderService.createOrder(request);
        
        // Assert
        assertThat(result.getId()).isNotNull();
        assertThat(result.getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(result.getTotalPrice()).isEqualTo(expectedPrice);
        
        verify(orderRepository).save(orderCaptor.capture());
        Order savedOrder = orderCaptor.getValue();
        assertThat(savedOrder.getCustomerId()).isEqualTo("customer-123");
    }
    
    @Test
    void createOrder_WithEmptyItems_ShouldThrowException() {
        // Arrange
        OrderRequest request = OrderRequest.builder()
            .customerId("customer-123")
            .items(Collections.emptyList())
            .build();
        
        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InvalidOrderException.class)
            .hasMessage("Order must have at least one item");
        
        verifyNoInteractions(orderRepository, pricingService);
    }
    
    @Test
    void createOrder_WhenPricingFails_ShouldPropagateException() {
        // Arrange
        OrderRequest request = OrderRequest.builder()
            .customerId("customer-123")
            .items(List.of(new OrderItem("product-1", 2)))
            .build();
        
        when(pricingService.calculateTotal(any()))
            .thenThrow(new PricingException("Pricing service unavailable"));
        
        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(PricingException.class);
    }
}
```

### Testing with JUnit 5 Parameterized Tests

```java
@ParameterizedTest
@CsvSource({
    "0, INVALID",
    "1, BRONZE",
    "5, BRONZE",
    "10, SILVER",
    "25, GOLD",
    "50, PLATINUM"
})
void calculateTier_ShouldReturnCorrectTier(int orderCount, CustomerTier expected) {
    CustomerTier result = customerService.calculateTier(orderCount);
    assertThat(result).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("provideInvalidRequests")
void validateRequest_WithInvalidData_ShouldThrow(OrderRequest request, String expectedMessage) {
    assertThatThrownBy(() -> orderService.validateRequest(request))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining(expectedMessage);
}

private static Stream<Arguments> provideInvalidRequests() {
    return Stream.of(
        Arguments.of(new OrderRequest(null, List.of()), "Customer ID required"),
        Arguments.of(new OrderRequest("c1", null), "Items required"),
        Arguments.of(new OrderRequest("c1", List.of()), "At least one item required")
    );
}
```

---

## Integration Testing

### Spring Boot Integration Tests

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class OrderIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @LocalServerPort
    private int port;
    
    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }
    
    @Test
    void createOrder_ShouldPersistAndReturn() {
        // Arrange
        OrderRequest request = OrderRequest.builder()
            .customerId("customer-123")
            .items(List.of(new OrderItem("product-1", 2, BigDecimal.TEN)))
            .build();
        
        // Act
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", request, Order.class);
        
        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getId()).isNotNull();
        
        // Verify persistence
        Optional<Order> persisted = orderRepository.findById(response.getBody().getId());
        assertThat(persisted).isPresent();
    }
}
```

### Testcontainers for Database Integration

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("integration-test")
class OrderRepositoryIntegrationTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void save_ShouldPersistOrder() {
        Order order = Order.builder()
            .customerId("customer-123")
            .status(OrderStatus.PENDING)
            .totalPrice(new BigDecimal("99.99"))
            .build();
        
        Order saved = orderRepository.save(order);
        
        assertThat(saved.getId()).isNotNull();
        
        Optional<Order> found = orderRepository.findById(saved.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getCustomerId()).isEqualTo("customer-123");
    }
    
    @Test
    void findByCustomerId_ShouldReturnOrders() {
        // Arrange
        orderRepository.saveAll(List.of(
            Order.builder().customerId("c1").status(OrderStatus.PENDING).build(),
            Order.builder().customerId("c1").status(OrderStatus.COMPLETED).build(),
            Order.builder().customerId("c2").status(OrderStatus.PENDING).build()
        ));
        
        // Act
        List<Order> orders = orderRepository.findByCustomerId("c1");
        
        // Assert
        assertThat(orders).hasSize(2);
        assertThat(orders).allMatch(o -> o.getCustomerId().equals("c1"));
    }
}
```

### Testcontainers for Kafka

```java
@SpringBootTest
@Testcontainers
class OrderEventIntegrationTest {
    
    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Autowired
    private KafkaConsumer<String, OrderEvent> consumer;
    
    @Test
    void publishOrderEvent_ShouldBeConsumed() throws Exception {
        // Arrange
        OrderEvent event = new OrderEvent("order-123", OrderEventType.CREATED);
        
        // Act
        kafkaTemplate.send("orders", event.getOrderId(), event).get();
        
        // Assert
        ConsumerRecords<String, OrderEvent> records = KafkaTestUtils.getRecords(
            consumer, Duration.ofSeconds(10));
        
        assertThat(records.count()).isGreaterThan(0);
        OrderEvent received = records.iterator().next().value();
        assertThat(received.getOrderId()).isEqualTo("order-123");
    }
}
```

---

## Contract Testing with Pact

### Consumer-Driven Contract Testing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CONTRACT TESTING FLOW                             │
│                                                                      │
│   Consumer (Order Service)           Provider (Product Service)     │
│                                                                      │
│   1. Define expectations                                            │
│   ┌────────────────┐                                                │
│   │ Consumer Test  │ ──► Pact File (Contract)                       │
│   │ with Pact DSL  │                                                │
│   └────────────────┘            │                                   │
│                                  │                                   │
│   2. Publish to Pact Broker     │                                   │
│                                  ▼                                   │
│                        ┌─────────────────┐                          │
│                        │   Pact Broker   │                          │
│                        └─────────────────┘                          │
│                                  │                                   │
│   3. Verify contract            │                                   │
│                                  ▼                                   │
│                        ┌────────────────┐                           │
│                        │ Provider Test  │                           │
│                        │ Verification   │                           │
│                        └────────────────┘                           │
│                                                                      │
│   Key Principle: Consumer defines expected behavior                 │
│                  Provider must satisfy all contracts                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Consumer Test (Pact)

```java
// pom.xml dependency
// <dependency>
//     <groupId>au.com.dius.pact.consumer</groupId>
//     <artifactId>junit5</artifactId>
//     <version>4.6.4</version>
//     <scope>test</scope>
// </dependency>

@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service", port = "8080")
class ProductClientContractTest {
    
    @Pact(provider = "product-service", consumer = "order-service")
    public V4Pact getProductPact(PactDslWithProvider builder) {
        return builder
            .given("product with ID P001 exists")
            .uponReceiving("a request for product P001")
                .path("/products/P001")
                .method("GET")
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .stringType("id", "P001")
                    .stringType("name", "Test Product")
                    .decimalType("price", 99.99)
                    .integerType("quantity", 100)
                )
            .toPact(V4Pact.class);
    }
    
    @Pact(provider = "product-service", consumer = "order-service")
    public V4Pact getProductNotFoundPact(PactDslWithProvider builder) {
        return builder
            .given("product with ID P999 does not exist")
            .uponReceiving("a request for non-existent product P999")
                .path("/products/P999")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(new PactDslJsonBody()
                    .stringType("error", "Product not found")
                )
            .toPact(V4Pact.class);
    }
    
    @Test
    @PactTestFor(pactMethod = "getProductPact")
    void getProduct_ShouldReturnProduct(MockServer mockServer) {
        // Arrange
        ProductClient client = new ProductClient(mockServer.getUrl());
        
        // Act
        Product result = client.getProduct("P001");
        
        // Assert
        assertThat(result.getId()).isEqualTo("P001");
        assertThat(result.getName()).isEqualTo("Test Product");
        assertThat(result.getPrice()).isEqualByComparingTo(new BigDecimal("99.99"));
    }
    
    @Test
    @PactTestFor(pactMethod = "getProductNotFoundPact")
    void getProduct_WhenNotFound_ShouldThrowException(MockServer mockServer) {
        ProductClient client = new ProductClient(mockServer.getUrl());
        
        assertThatThrownBy(() -> client.getProduct("P999"))
            .isInstanceOf(ProductNotFoundException.class);
    }
}
```

### Provider Verification Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("product-service")
@PactBroker(
    host = "pact-broker.internal",
    authentication = @PactBrokerAuth(token = "${PACT_BROKER_TOKEN}")
)
class ProductServiceProviderTest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private ProductRepository productRepository;
    
    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }
    
    @State("product with ID P001 exists")
    void setupProduct() {
        Product product = Product.builder()
            .id("P001")
            .name("Test Product")
            .price(new BigDecimal("99.99"))
            .quantity(100)
            .build();
        productRepository.save(product);
    }
    
    @State("product with ID P999 does not exist")
    void setupNoProduct() {
        productRepository.deleteById("P999");
    }
}
```

---

## Component Testing

### Testing with WireMock

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)  // Random port
class OrderServiceComponentTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private WireMockServer wireMockServer;
    
    @Test
    void createOrder_ShouldCallInventoryAndPayment() {
        // Stub inventory service
        stubFor(post(urlEqualTo("/inventory/reserve"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"reserved": true, "reservationId": "res-123"}
                    """)));
        
        // Stub payment service
        stubFor(post(urlEqualTo("/payments"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"paymentId": "pay-123", "status": "SUCCESS"}
                    """)));
        
        // Create order
        OrderRequest request = new OrderRequest("customer-1", 
            List.of(new OrderItem("product-1", 2)));
        
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", request, Order.class);
        
        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        
        // Verify external calls were made
        verify(postRequestedFor(urlEqualTo("/inventory/reserve"))
            .withRequestBody(containing("product-1")));
        verify(postRequestedFor(urlEqualTo("/payments")));
    }
    
    @Test
    void createOrder_WhenInventoryFails_ShouldReturnError() {
        // Stub inventory service failure
        stubFor(post(urlEqualTo("/inventory/reserve"))
            .willReturn(aResponse()
                .withStatus(503)
                .withBody("""
                    {"error": "Service unavailable"}
                    """)));
        
        OrderRequest request = new OrderRequest("customer-1", 
            List.of(new OrderItem("product-1", 2)));
        
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
            "/api/orders", request, ErrorResponse.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.SERVICE_UNAVAILABLE);
        
        // Verify payment was not called
        verify(0, postRequestedFor(urlEqualTo("/payments")));
    }
}
```

### Testing with Test Slices

```java
// WebMvcTest - Controller layer only
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private OrderService orderService;
    
    @Test
    void createOrder_ShouldReturnCreated() throws Exception {
        Order order = Order.builder()
            .id("order-123")
            .status(OrderStatus.PENDING)
            .build();
        
        when(orderService.createOrder(any())).thenReturn(order);
        
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerId": "customer-123",
                        "items": [{"productId": "p1", "quantity": 2}]
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("order-123"))
            .andExpect(jsonPath("$.status").value("PENDING"));
    }
    
    @Test
    void createOrder_WithInvalidData_ShouldReturnBadRequest() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerId": "",
                        "items": []
                    }
                    """))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }
}

// DataJpaTest - Repository layer only
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {
    
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void findByStatus_ShouldReturnFilteredOrders() {
        orderRepository.saveAll(List.of(
            Order.builder().status(OrderStatus.PENDING).build(),
            Order.builder().status(OrderStatus.COMPLETED).build()
        ));
        
        List<Order> pending = orderRepository.findByStatus(OrderStatus.PENDING);
        
        assertThat(pending).hasSize(1);
    }
}
```

---

## End-to-End Testing

### E2E with RestAssured

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
@ActiveProfiles("e2e")
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderE2ETest {
    
    private static String orderId;
    
    @BeforeAll
    static void setup() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = 8080;
    }
    
    @Test
    @Order(1)
    void createOrder() {
        orderId = given()
            .contentType(ContentType.JSON)
            .body("""
                {
                    "customerId": "e2e-customer",
                    "items": [
                        {"productId": "prod-1", "quantity": 2}
                    ]
                }
                """)
        .when()
            .post("/api/orders")
        .then()
            .statusCode(201)
            .body("status", equalTo("PENDING"))
            .extract()
            .path("id");
        
        assertThat(orderId).isNotNull();
    }
    
    @Test
    @Order(2)
    void getOrder() {
        given()
        .when()
            .get("/api/orders/{id}", orderId)
        .then()
            .statusCode(200)
            .body("id", equalTo(orderId))
            .body("status", equalTo("PENDING"));
    }
    
    @Test
    @Order(3)
    void processPayment() {
        given()
            .contentType(ContentType.JSON)
            .body("""
                {"amount": 100.00}
                """)
        .when()
            .post("/api/orders/{id}/payment", orderId)
        .then()
            .statusCode(200)
            .body("paymentStatus", equalTo("SUCCESS"));
    }
    
    @Test
    @Order(4)
    void verifyOrderCompleted() {
        // Wait for async processing
        await().atMost(10, TimeUnit.SECONDS)
            .pollInterval(1, TimeUnit.SECONDS)
            .until(() -> {
                String status = given()
                    .when()
                        .get("/api/orders/{id}", orderId)
                    .then()
                        .extract()
                        .path("status");
                return "COMPLETED".equals(status);
            });
    }
}
```

### Cucumber BDD Tests

```gherkin
# features/order.feature
Feature: Order Management
  As a customer
  I want to place orders
  So that I can purchase products

  Scenario: Create a new order
    Given the customer "customer-123" exists
    And the product "product-1" is in stock with quantity 10
    When the customer creates an order with:
      | productId  | quantity |
      | product-1  | 2        |
    Then the order should be created with status "PENDING"
    And the inventory should be reduced by 2

  Scenario: Order with insufficient inventory
    Given the product "product-2" is in stock with quantity 1
    When the customer creates an order with:
      | productId  | quantity |
      | product-2  | 5        |
    Then the order should fail with error "Insufficient inventory"
```

```java
// Step definitions
public class OrderSteps {
    
    private TestRestTemplate restTemplate = new TestRestTemplate();
    private ResponseEntity<?> lastResponse;
    private String customerId;
    
    @Given("the customer {string} exists")
    public void customerExists(String customerId) {
        this.customerId = customerId;
        // Setup customer in database
    }
    
    @Given("the product {string} is in stock with quantity {int}")
    public void productInStock(String productId, int quantity) {
        // Setup inventory
    }
    
    @When("the customer creates an order with:")
    public void createOrder(DataTable dataTable) {
        List<OrderItem> items = dataTable.asMaps().stream()
            .map(row -> new OrderItem(row.get("productId"), 
                Integer.parseInt(row.get("quantity"))))
            .toList();
        
        OrderRequest request = new OrderRequest(customerId, items);
        lastResponse = restTemplate.postForEntity("/api/orders", request, Order.class);
    }
    
    @Then("the order should be created with status {string}")
    public void orderCreatedWithStatus(String status) {
        assertThat(lastResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        Order order = (Order) lastResponse.getBody();
        assertThat(order.getStatus().name()).isEqualTo(status);
    }
}
```

---

## Testing Asynchronous Operations

```java
@SpringBootTest
class AsyncOrderServiceTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void processOrderAsync_ShouldCompleteEventually() {
        // Arrange
        Order order = orderRepository.save(
            Order.builder().status(OrderStatus.PENDING).build());
        
        // Act
        orderService.processOrderAsync(order.getId());
        
        // Assert with Awaitility
        await()
            .atMost(Duration.ofSeconds(30))
            .pollInterval(Duration.ofMillis(500))
            .untilAsserted(() -> {
                Order updated = orderRepository.findById(order.getId()).orElseThrow();
                assertThat(updated.getStatus()).isEqualTo(OrderStatus.COMPLETED);
            });
    }
    
    @Test
    void publishEvent_ShouldBeConsumed() throws Exception {
        // Arrange
        CountDownLatch latch = new CountDownLatch(1);
        OrderEventListener listener = new OrderEventListener() {
            @Override
            public void onOrderCreated(OrderCreatedEvent event) {
                latch.countDown();
            }
        };
        
        // Act
        orderService.createOrder(new OrderRequest("c1", List.of()));
        
        // Assert
        boolean received = latch.await(5, TimeUnit.SECONDS);
        assertThat(received).isTrue();
    }
}
```

---

## Test Data Management

### Test Fixtures

```java
public class OrderTestFixtures {
    
    public static OrderRequest validOrderRequest() {
        return OrderRequest.builder()
            .customerId("customer-123")
            .items(List.of(
                new OrderItem("product-1", 2, new BigDecimal("50.00")),
                new OrderItem("product-2", 1, new BigDecimal("25.00"))
            ))
            .shippingAddress(Address.builder()
                .street("123 Test St")
                .city("Test City")
                .zipCode("12345")
                .build())
            .build();
    }
    
    public static Order pendingOrder() {
        return Order.builder()
            .id("order-123")
            .customerId("customer-123")
            .status(OrderStatus.PENDING)
            .totalPrice(new BigDecimal("125.00"))
            .createdAt(Instant.now())
            .build();
    }
    
    public static Order completedOrder() {
        Order order = pendingOrder();
        order.setStatus(OrderStatus.COMPLETED);
        order.setCompletedAt(Instant.now());
        return order;
    }
}

// Usage in tests
@Test
void processOrder_ShouldUpdateStatus() {
    Order order = OrderTestFixtures.pendingOrder();
    when(orderRepository.findById("order-123")).thenReturn(Optional.of(order));
    
    orderService.processOrder("order-123");
    
    verify(orderRepository).save(argThat(o -> 
        o.getStatus() == OrderStatus.PROCESSING));
}
```

### Test Data Builders

```java
public class OrderBuilder {
    
    private String id = UUID.randomUUID().toString();
    private String customerId = "default-customer";
    private OrderStatus status = OrderStatus.PENDING;
    private List<OrderItem> items = new ArrayList<>();
    private BigDecimal totalPrice = BigDecimal.ZERO;
    
    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }
    
    public OrderBuilder withId(String id) {
        this.id = id;
        return this;
    }
    
    public OrderBuilder withCustomer(String customerId) {
        this.customerId = customerId;
        return this;
    }
    
    public OrderBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }
    
    public OrderBuilder withItem(String productId, int quantity, BigDecimal price) {
        this.items.add(new OrderItem(productId, quantity, price));
        this.totalPrice = this.totalPrice.add(price.multiply(BigDecimal.valueOf(quantity)));
        return this;
    }
    
    public Order build() {
        return Order.builder()
            .id(id)
            .customerId(customerId)
            .status(status)
            .items(items)
            .totalPrice(totalPrice)
            .build();
    }
}

// Usage
Order order = OrderBuilder.anOrder()
    .withCustomer("vip-customer")
    .withItem("product-1", 2, new BigDecimal("50"))
    .withItem("product-2", 1, new BigDecimal("30"))
    .withStatus(OrderStatus.PENDING)
    .build();
```

---

## Interview Questions

### Conceptual Questions
1. **Explain the testing pyramid for microservices.**
2. **What is contract testing and why is it important?**
3. **How do you test service-to-service communication?**
4. **What's the difference between component and integration tests?**
5. **How do you handle test data in microservices testing?**

### Design Questions
1. **Design a testing strategy for a payment microservice.**
2. **How would you implement contract testing across 20+ services?**
3. **Design an E2E testing approach for a distributed system.**
4. **How do you test saga transactions?**
5. **Design a test data management strategy for microservices.**

### Practical Questions
1. **How do you test async message processing?**
2. **How do you mock external dependencies in integration tests?**
3. **What's your approach to testing circuit breakers?**
4. **How do you handle flaky tests in distributed systems?**
5. **How do you test database migrations?**

---

## Key Takeaways

1. **Follow the testing pyramid** - more unit tests, fewer E2E tests
2. **Use contract testing** to ensure API compatibility between services
3. **Testcontainers** provide real dependencies in integration tests
4. **WireMock** enables stubbing external services
5. **Test slices** (@WebMvcTest, @DataJpaTest) speed up testing
6. **Awaitility** simplifies testing async operations
7. **Use fixtures and builders** for consistent test data
8. **Test failure scenarios** - circuit breakers, timeouts, retries
9. **Consumer-driven contracts** prevent breaking changes
10. **E2E tests should be minimal** - expensive and slow

---

*Previous: [Configuration Management](10-configuration-management.md) | Next: [API Versioning Strategies](12-api-versioning-strategies.md)*
