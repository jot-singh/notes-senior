# API Versioning Strategies

## Overview
API versioning is essential for evolving microservices without breaking existing clients. This note covers versioning strategies, backward compatibility, deprecation policies, and best practices for managing API changes in a microservices ecosystem.

---

## Why API Versioning?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    API VERSIONING CHALLENGES                         │
│                                                                      │
│   Without Versioning:                                               │
│                                                                      │
│   ┌─────────────┐     v1.0      ┌─────────────┐                    │
│   │  Mobile App │─────────────►│ Order API   │                     │
│   │  (Old)      │               │             │                     │
│   └─────────────┘               └─────────────┘                     │
│                                       │                             │
│   API Change (breaking)               │                             │
│                                       ▼                             │
│   ┌─────────────┐     v2.0      ┌─────────────┐                    │
│   │  Mobile App │──────X───────►│ Order API   │  ← Mobile breaks!  │
│   │  (Old)      │    (Breaks)   │             │                     │
│   └─────────────┘               └─────────────┘                     │
│                                                                      │
│   With Versioning:                                                  │
│                                                                      │
│   ┌─────────────┐     /v1/*     ┌─────────────┐                    │
│   │  Mobile App │─────────────►│ Order API   │                     │
│   │  (Old)      │               │   v1 + v2   │  ← Both supported  │
│   └─────────────┘               └─────────────┘                     │
│                                       ▲                             │
│   ┌─────────────┐     /v2/*           │                             │
│   │  Web App    │─────────────────────┘                             │
│   │  (New)      │                                                   │
│   └─────────────┘                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Versioning Strategies

### 1. URL Path Versioning

```
GET /api/v1/orders
GET /api/v2/orders
```

```java
// Spring Boot implementation
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderV1Response> getOrder(@PathVariable String id) {
        Order order = orderService.getOrder(id);
        return ResponseEntity.ok(OrderV1Response.from(order));
    }
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderV2Response> getOrder(@PathVariable String id) {
        Order order = orderService.getOrder(id);
        return ResponseEntity.ok(OrderV2Response.from(order));  // Enhanced response
    }
}

// Response DTOs
public record OrderV1Response(
    String id,
    String customerName,
    double totalAmount,
    String status
) {
    public static OrderV1Response from(Order order) {
        return new OrderV1Response(
            order.getId(),
            order.getCustomer().getName(),
            order.getTotalAmount().doubleValue(),
            order.getStatus().name()
        );
    }
}

public record OrderV2Response(
    String id,
    CustomerInfo customer,    // Nested object instead of flat
    MoneyAmount total,        // Structured money type
    OrderStatus status,
    List<OrderItem> items,    // Added field
    Instant createdAt,        // Added field
    Map<String, String> metadata
) {
    public static OrderV2Response from(Order order) {
        // Enhanced mapping
    }
}
```

**Pros:**
- Clear and explicit versioning
- Easy to implement and understand
- Cacheable (different URLs)
- Works with all HTTP clients

**Cons:**
- URL pollution
- Duplicated controller code
- Hard to maintain many versions

---

### 2. Header Versioning

```
GET /api/orders
Accept: application/vnd.company.order.v1+json
Accept: application/vnd.company.order.v2+json
```

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping(value = "/{id}", produces = "application/vnd.company.order.v1+json")
    public ResponseEntity<OrderV1Response> getOrderV1(@PathVariable String id) {
        return ResponseEntity.ok(OrderV1Response.from(orderService.getOrder(id)));
    }
    
    @GetMapping(value = "/{id}", produces = "application/vnd.company.order.v2+json")
    public ResponseEntity<OrderV2Response> getOrderV2(@PathVariable String id) {
        return ResponseEntity.ok(OrderV2Response.from(orderService.getOrder(id)));
    }
}

// Or using custom header
@GetMapping("/{id}")
public ResponseEntity<?> getOrder(
        @PathVariable String id,
        @RequestHeader(value = "X-API-Version", defaultValue = "1") int version) {
    
    Order order = orderService.getOrder(id);
    
    return switch (version) {
        case 1 -> ResponseEntity.ok(OrderV1Response.from(order));
        case 2 -> ResponseEntity.ok(OrderV2Response.from(order));
        default -> ResponseEntity.badRequest().body("Unsupported API version");
    };
}
```

**Pros:**
- Clean URLs
- Follows HTTP standards (content negotiation)
- Single endpoint per resource

**Cons:**
- Less discoverable
- Harder to test in browser
- Can break caching

---

### 3. Query Parameter Versioning

```
GET /api/orders?version=1
GET /api/orders?version=2
```

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping("/{id}")
    public ResponseEntity<?> getOrder(
            @PathVariable String id,
            @RequestParam(value = "version", defaultValue = "1") int version) {
        
        Order order = orderService.getOrder(id);
        
        return switch (version) {
            case 1 -> ResponseEntity.ok(OrderV1Response.from(order));
            case 2 -> ResponseEntity.ok(OrderV2Response.from(order));
            default -> ResponseEntity.badRequest()
                .body(new ErrorResponse("Unsupported version: " + version));
        };
    }
}
```

**Pros:**
- Optional versioning (default fallback)
- Easy to implement
- Works with all clients

**Cons:**
- Can be missed in requests
- May affect caching
- Less RESTful

---

## Comparison of Strategies

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VERSIONING STRATEGY COMPARISON                    │
│                                                                      │
│   Strategy      │ Clarity │ Caching │ REST │ Adoption │ Maintenance │
│   ──────────────┼─────────┼─────────┼──────┼──────────┼────────────│
│   URL Path      │  High   │  Good   │ Med  │  High    │  Medium    │
│   Header        │  Medium │  Poor   │ High │  Medium  │  Low       │
│   Query Param   │  Medium │  Poor   │ Low  │  Medium  │  Low       │
│                                                                      │
│   Recommendation:                                                   │
│   • Public APIs: URL Path (most explicit)                          │
│   • Internal APIs: Header or URL Path                              │
│   • Legacy systems: Query Parameter (easy adoption)                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Gateway Versioning

### Kong API Gateway

```yaml
# kong.yml
services:
  - name: order-service-v1
    url: http://order-service-v1:8080
    routes:
      - name: order-v1-route
        paths:
          - /api/v1/orders
        strip_path: true
        
  - name: order-service-v2
    url: http://order-service-v2:8080
    routes:
      - name: order-v2-route
        paths:
          - /api/v2/orders
        strip_path: true

# Traffic splitting for gradual migration
plugins:
  - name: canary
    service: order-service-v1
    config:
      percentage: 90
      upstream_host: order-service-v2:8080
```

### Spring Cloud Gateway

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-v1
          uri: lb://ORDER-SERVICE-V1
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=2
            
        - id: order-service-v2
          uri: lb://ORDER-SERVICE-V2
          predicates:
            - Path=/api/v2/orders/**
          filters:
            - StripPrefix=2
            
        # Header-based routing
        - id: order-service-v2-header
          uri: lb://ORDER-SERVICE-V2
          predicates:
            - Path=/api/orders/**
            - Header=X-API-Version, 2
          filters:
            - StripPrefix=1
            
        # Default to v1
        - id: order-service-default
          uri: lb://ORDER-SERVICE-V1
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

```java
// Custom version routing filter
@Component
public class VersionRoutingFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String version = exchange.getRequest().getHeaders()
            .getFirst("X-API-Version");
        
        if (version == null) {
            version = exchange.getRequest().getQueryParams()
                .getFirst("version");
        }
        
        if (version == null) {
            version = "1";  // Default
        }
        
        // Add version to attributes for downstream filters
        exchange.getAttributes().put("api-version", version);
        
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() {
        return -1;
    }
}
```

---

## Backward Compatibility

### Additive Changes (Non-Breaking)

```java
// V1 Response
public record OrderV1Response(
    String id,
    String customerName,
    double totalAmount
) {}

// V2 Response - Backward compatible (additive)
public record OrderV2Response(
    String id,
    String customerName,
    double totalAmount,
    String currency,      // New optional field
    List<String> tags,    // New optional field
    Instant createdAt     // New optional field
) {}
```

### Breaking Changes

```java
// Changes that BREAK backward compatibility:

// 1. Removing fields
// V1: { "customerName": "John" }
// V2: { "customer": { "name": "John" } }  // Field renamed/restructured

// 2. Changing field types
// V1: { "amount": 100.50 }
// V2: { "amount": "100.50" }  // Type changed from number to string

// 3. Changing required fields
// V1: { "orderId": "123" }  // orderId was optional
// V2: { "orderId": "123" }  // orderId is now required

// 4. Changing enum values
// V1: status can be "PENDING", "COMPLETED"
// V2: status can be "PENDING", "PROCESSING", "COMPLETED"  // New value added (may break)
```

### Response Transformation

```java
@Service
public class ResponseTransformer {
    
    private final ObjectMapper objectMapper;
    
    public String transformResponse(Object response, int targetVersion) {
        if (targetVersion == 1 && response instanceof OrderV2Response v2) {
            // Transform V2 to V1 format
            OrderV1Response v1 = new OrderV1Response(
                v2.id(),
                v2.customer().name(),
                v2.total().amount().doubleValue()
            );
            return objectMapper.writeValueAsString(v1);
        }
        return objectMapper.writeValueAsString(response);
    }
}

// Using aspect for automatic transformation
@Aspect
@Component
public class VersionTransformAspect {
    
    @Around("@annotation(ApiVersioned)")
    public Object transformResponse(ProceedingJoinPoint pjp) throws Throwable {
        Object result = pjp.proceed();
        
        HttpServletRequest request = getCurrentRequest();
        int version = getApiVersion(request);
        
        return responseTransformer.transform(result, version);
    }
}
```

---

## Deprecation Strategy

### Deprecation Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                    API DEPRECATION LIFECYCLE                         │
│                                                                      │
│   Phase 1: Announcement (3+ months before)                          │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  • Document deprecation in API docs                          │   │
│   │  • Add deprecation headers to responses                      │   │
│   │  • Notify API consumers                                      │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                          │                                          │
│                          ▼                                          │
│   Phase 2: Warning Period (1-3 months)                              │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  • Log deprecation warnings                                  │   │
│   │  • Track usage of deprecated endpoints                       │   │
│   │  • Reach out to heavy users                                  │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                          │                                          │
│                          ▼                                          │
│   Phase 3: Sunset (End of support)                                  │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  • Return 410 Gone or 301 Redirect                          │   │
│   │  • Remove from documentation                                 │   │
│   │  • Archive code (don't delete immediately)                   │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Deprecation Headers

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderV1Response> getOrder(@PathVariable String id) {
        Order order = orderService.getOrder(id);
        
        return ResponseEntity.ok()
            .header("Deprecation", "true")
            .header("Sunset", "Sat, 31 Dec 2024 23:59:59 GMT")
            .header("Link", "</api/v2/orders/" + id + ">; rel=\"successor-version\"")
            .header("X-API-Warn", "299 - \"API v1 is deprecated. Use v2.\"")
            .body(OrderV1Response.from(order));
    }
}

// Deprecation interceptor
@Component
public class DeprecationInterceptor implements HandlerInterceptor {
    
    private final DeprecationConfig config;
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler, ModelAndView modelAndView) {
        
        String path = request.getRequestURI();
        
        if (config.isDeprecated(path)) {
            DeprecationInfo info = config.getDeprecationInfo(path);
            
            response.setHeader("Deprecation", "true");
            response.setHeader("Sunset", info.getSunsetDate().toString());
            
            if (info.getSuccessorUrl() != null) {
                response.setHeader("Link", 
                    "<" + info.getSuccessorUrl() + ">; rel=\"successor-version\"");
            }
            
            // Log usage for tracking
            deprecationMetrics.recordUsage(path, request.getHeader("X-Client-ID"));
        }
    }
}
```

### Deprecation Configuration

```yaml
# deprecation-config.yml
deprecations:
  - path: /api/v1/**
    deprecated: true
    sunsetDate: 2024-12-31
    successorPath: /api/v2/**
    message: "API v1 will be retired on December 31, 2024"
    
  - path: /api/v2/orders/legacy/**
    deprecated: true
    sunsetDate: 2024-06-30
    message: "Legacy order endpoints deprecated"
```

---

## OpenAPI Documentation

```yaml
# openapi.yml
openapi: 3.0.3
info:
  title: Order Service API
  version: "2.0"
  description: |
    Order management API.
    
    ## API Versioning
    This API uses URL path versioning. Current versions:
    - **v2** (current): Full feature set
    - **v1** (deprecated): Sunset date 2024-12-31
    
    ## Deprecation Policy
    Deprecated APIs will be supported for minimum 6 months.

servers:
  - url: https://api.example.com/api/v2
    description: Production (v2)
  - url: https://api.example.com/api/v1
    description: Production (v1 - Deprecated)

paths:
  /orders/{id}:
    get:
      summary: Get order by ID
      operationId: getOrder
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Order found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderV2'
                
  # Deprecated endpoint documentation
  /v1/orders/{id}:
    get:
      deprecated: true
      summary: Get order by ID (DEPRECATED)
      description: |
        **DEPRECATED**: This endpoint will be removed on 2024-12-31.
        Use `/v2/orders/{id}` instead.
      operationId: getOrderV1

components:
  schemas:
    OrderV2:
      type: object
      properties:
        id:
          type: string
        customer:
          $ref: '#/components/schemas/Customer'
        total:
          $ref: '#/components/schemas/Money'
        status:
          $ref: '#/components/schemas/OrderStatus'
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        createdAt:
          type: string
          format: date-time
          
    OrderV1:
      deprecated: true
      type: object
      description: "DEPRECATED: Use OrderV2"
      properties:
        id:
          type: string
        customerName:
          type: string
        totalAmount:
          type: number
```

---

## Semantic Versioning for APIs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SEMANTIC VERSIONING                               │
│                                                                      │
│   MAJOR.MINOR.PATCH                                                 │
│     │     │     │                                                   │
│     │     │     └── Bug fixes (backward compatible)                 │
│     │     │         No API changes                                  │
│     │     │                                                         │
│     │     └──────── New features (backward compatible)              │
│     │               Additive changes only                           │
│     │                                                               │
│     └────────────── Breaking changes                                │
│                     Require version bump in URL                     │
│                                                                      │
│   Examples:                                                         │
│   • 1.0.0 → 1.0.1: Fixed null handling in response                 │
│   • 1.0.1 → 1.1.0: Added 'createdAt' field to response             │
│   • 1.1.0 → 2.0.0: Changed 'amount' from number to object          │
│                                                                      │
│   URL Versioning Convention:                                        │
│   • /api/v1/...  = MAJOR version 1.x.x                             │
│   • /api/v2/...  = MAJOR version 2.x.x                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Multi-Version Support

### Code Organization

```
src/main/java/com/example/
├── controller/
│   ├── v1/
│   │   └── OrderControllerV1.java
│   └── v2/
│       └── OrderControllerV2.java
├── dto/
│   ├── v1/
│   │   ├── OrderRequestV1.java
│   │   └── OrderResponseV1.java
│   └── v2/
│       ├── OrderRequestV2.java
│       └── OrderResponseV2.java
├── mapper/
│   ├── OrderMapperV1.java
│   └── OrderMapperV2.java
└── service/
    └── OrderService.java           # Shared service logic
```

### Shared Service with Versioned DTOs

```java
// Core service (version-agnostic)
@Service
public class OrderService {
    
    public Order createOrder(String customerId, List<OrderItemInput> items) {
        // Business logic is version-independent
        validateCustomer(customerId);
        Order order = new Order(customerId, items);
        return orderRepository.save(order);
    }
    
    public Order getOrder(String orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}

// V1 Controller
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    
    private final OrderService orderService;
    private final OrderMapperV1 mapper;
    
    @PostMapping
    public ResponseEntity<OrderResponseV1> createOrder(
            @RequestBody @Valid OrderRequestV1 request) {
        
        Order order = orderService.createOrder(
            request.customerName(),  // V1 uses customerName
            mapper.toOrderItems(request.items())
        );
        
        return ResponseEntity.created(URI.create("/api/v1/orders/" + order.getId()))
            .body(mapper.toResponse(order));
    }
}

// V2 Controller
@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    
    private final OrderService orderService;
    private final OrderMapperV2 mapper;
    
    @PostMapping
    public ResponseEntity<OrderResponseV2> createOrder(
            @RequestBody @Valid OrderRequestV2 request) {
        
        Order order = orderService.createOrder(
            request.customer().id(),  // V2 uses customer object
            mapper.toOrderItems(request.items())
        );
        
        return ResponseEntity.created(URI.create("/api/v2/orders/" + order.getId()))
            .body(mapper.toResponse(order));
    }
}
```

---

## Testing Multiple Versions

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderApiVersionTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void v1AndV2_ShouldReturnSameData_InDifferentFormats() {
        // Create order via V2 (latest)
        OrderRequestV2 requestV2 = new OrderRequestV2(
            new Customer("c1", "John Doe"),
            List.of(new OrderItemV2("p1", 2, new Money(BigDecimal.TEN, "USD")))
        );
        
        ResponseEntity<OrderResponseV2> v2Response = restTemplate.postForEntity(
            "/api/v2/orders", requestV2, OrderResponseV2.class);
        String orderId = v2Response.getBody().id();
        
        // Fetch via V1
        ResponseEntity<OrderResponseV1> v1Response = restTemplate.getForEntity(
            "/api/v1/orders/{id}", OrderResponseV1.class, orderId);
        
        // Fetch via V2
        ResponseEntity<OrderResponseV2> v2GetResponse = restTemplate.getForEntity(
            "/api/v2/orders/{id}", OrderResponseV2.class, orderId);
        
        // Assert both return same core data
        assertThat(v1Response.getBody().id()).isEqualTo(v2GetResponse.getBody().id());
        assertThat(v1Response.getBody().customerName())
            .isEqualTo(v2GetResponse.getBody().customer().name());
        assertThat(v1Response.getBody().totalAmount())
            .isEqualTo(v2GetResponse.getBody().total().amount().doubleValue());
    }
    
    @Test
    void deprecatedV1_ShouldIncludeDeprecationHeaders() {
        ResponseEntity<OrderResponseV1> response = restTemplate.getForEntity(
            "/api/v1/orders/{id}", OrderResponseV1.class, "order-123");
        
        assertThat(response.getHeaders().get("Deprecation")).contains("true");
        assertThat(response.getHeaders().get("Sunset")).isNotNull();
    }
}
```

---

## Best Practices

### 1. Design for Evolution

```java
// Good: Flexible structure
public record OrderResponse(
    String id,
    Map<String, Object> attributes,  // Extensible
    List<Link> _links                 // HATEOAS
) {}

// Good: Use wrapper types
public record Money(
    BigDecimal amount,
    String currency
) {}

// Instead of: double amount
```

### 2. Version Everything

```yaml
# Include version in error responses
error:
  code: ORDER_NOT_FOUND
  message: Order not found
  apiVersion: "2.0"
  timestamp: "2024-01-15T10:30:00Z"
```

### 3. Monitor Version Usage

```java
@Component
public class ApiVersionMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordApiCall(String version, String endpoint) {
        meterRegistry.counter("api.calls",
            "version", version,
            "endpoint", endpoint
        ).increment();
    }
    
    public void recordDeprecatedUsage(String version, String clientId) {
        meterRegistry.counter("api.deprecated.usage",
            "version", version,
            "clientId", clientId
        ).increment();
    }
}
```

---

## Interview Questions

### Conceptual Questions
1. **What are the different API versioning strategies?**
2. **What constitutes a breaking change in an API?**
3. **How do you handle deprecation of API versions?**
4. **What is semantic versioning and how does it apply to APIs?**
5. **How do you maintain backward compatibility?**

### Design Questions
1. **Design a versioning strategy for a public API with millions of users.**
2. **How would you migrate clients from v1 to v2 of your API?**
3. **Design an API gateway that supports multiple API versions.**
4. **How do you handle versioning in event-driven microservices?**
5. **Design a deprecation notification system for API consumers.**

### Practical Questions
1. **How do you document multiple API versions?**
2. **How do you test backward compatibility?**
3. **How do you track usage of deprecated endpoints?**
4. **What's your strategy for maintaining multiple versions of code?**
5. **How do you handle database schema changes with API versioning?**

---

## Key Takeaways

1. **URL path versioning is most explicit** and widely adopted for public APIs
2. **Header versioning is cleaner** but less discoverable
3. **Avoid breaking changes** - prefer additive changes
4. **Deprecate gracefully** - give consumers 6+ months notice
5. **Document versions clearly** in OpenAPI specs
6. **Monitor version usage** to track migration progress
7. **Keep business logic version-agnostic** - version only DTOs and controllers
8. **Test all supported versions** in your CI/CD pipeline
9. **Use semantic versioning** for clear change communication
10. **Plan for sunsetting** - don't support old versions indefinitely

---

*Previous: [Microservices Testing](11-microservices-testing.md) | Back to [Topics Index](../../topics-index.md)*
