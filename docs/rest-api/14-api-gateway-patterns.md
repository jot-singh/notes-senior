# API Gateway Patterns

## What You'll Learn
How API gateways sit at the edge of your architecture, handling cross-cutting concerns like authentication, rate limiting, routing, and protocol translation. You'll understand gateway responsibilities, Backend for Frontend patterns, and how to avoid gateway anti-patterns.

## Why This Matters
Gateways are the front door to your microservices. They centralize common functionality (auth, rate limiting, logging) so services focus on business logic. Without a gateway, every service duplicates auth, CORS, rate limiting—maintenance nightmare and security risk. Good gateway design improves security, reduces latency, and simplifies client integration. Bad gateway design creates bottlenecks, tight coupling, and single points of failure. Understanding gateway patterns is essential for distributed system architecture.

## Gateway Responsibilities

### 1. Routing & Load Balancing

**Path-based Routing**:
```nginx
location /orders {
    proxy_pass http://order-service;
}

location /customers {
    proxy_pass http://customer-service;
}

location /payments {
    proxy_pass http://payment-service;
}
```

**Header-based Routing** (Versioning):
```nginx
map $http_api_version $backend {
    "1" order-service-v1;
    "2" order-service-v2;
    default order-service-v2;
}

server {
    location /orders {
        proxy_pass http://$backend;
    }
}
```

**Canary Routing** (10% traffic to new version):
```yaml
# Kong configuration
routes:
  - paths: ["/orders"]
    service: order-service
    plugins:
      - name: canary
        config:
          percentage: 10
          upstream: order-service-v2
```

### 2. Authentication & Authorization

**Centralized Token Validation**:
```java
// Gateway validates JWT once; services trust gateway
@Component
public class AuthenticationFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange.getRequest());
        
        return validateToken(token)
            .flatMap(claims -> {
                // Add user context to headers for downstream services
                ServerHttpRequest request = exchange.getRequest().mutate()
                    .header(\"X-User-ID\", claims.getSubject())
                    .header(\"X-User-Roles\", String.join(\",\", claims.getRoles()))
                    .header(\"X-Tenant-ID\", claims.getTenantId())
                    .build();
                
                return chain.filter(exchange.mutate().request(request).build());
            })
            .onErrorResume(e -> {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }\n}\n```\n\n**Services Trust Gateway**:\n```java\n// No JWT validation needed; trust gateway headers\n@GetMapping(\"/orders/{id}\")\npublic Order getOrder(\n    @PathVariable String id,\n    @RequestHeader(\"X-User-ID\") String userId,\n    @RequestHeader(\"X-Tenant-ID\") String tenantId\n) {\n    // Already authenticated by gateway\n    return orderService.getOrder(id, userId, tenantId);\n}\n```\n\n### 3. Rate Limiting\n\nSee module 12 for algorithms.\n\n**Gateway-Level Limits**:\n```yaml\nplugins:\n  - name: rate-limiting\n    config:\n      minute: 1000\n      policy: redis\n      redis_host: redis.example.com\n```\n\n### 4. Request/Response Transformation\n\n**Legacy API Compatibility**:\n```javascript\n// Transform old API format to new\nfunction transformRequest(request) {\n  // Old: {customer_id: \"123\"}\n  // New: {customerId: \"123\"}\n  return {\n    ...request,\n    customerId: request.customer_id,\n    customer_id: undefined\n  };\n}\n\nfunction transformResponse(response) {\n  // New: {customerId: \"123\"}\n  // Old: {customer_id: \"123\"}\n  return {\n    ...response,\n    customer_id: response.customerId,\n    customerId: undefined\n  };\n}\n```\n\n**GraphQL to REST**:\n```javascript\n// Gateway translates GraphQL query to multiple REST calls\nquery {\n  order(id: \"ord-123\") {\n    id\n    customer { name, email }\n    items { product { name, price } }\n  }\n}\n\n// \u2192 Gateway makes 3 calls:\n// GET /orders/ord-123\n// GET /customers/{customerId}\n// GET /products/{productId} (for each item)\n```\n\n### 5. Caching\n\n**Edge Caching**:\n```nginx\nproxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m;\n\nlocation /products {\n    proxy_cache api_cache;\n    proxy_cache_valid 200 10m;\n    proxy_cache_key \"$request_uri\";\n    proxy_cache_methods GET HEAD;\n    add_header X-Cache-Status $upstream_cache_status;\n    \n    proxy_pass http://product-service;\n}\n```\n\n### 6. Observability\n\n**Correlation IDs**:\n```java\npublic Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {\n    String requestId = UUID.randomUUID().toString();\n    \n    ServerHttpRequest request = exchange.getRequest().mutate()\n        .header(\"X-Request-ID\", requestId)\n        .header(\"X-Forwarded-For\", getClientIP(exchange))\n        .build();\n    \n    // Log at gateway\n    log.info(\"Request started\",\n        kv(\"request_id\", requestId),\n        kv(\"method\", request.getMethod()),\n        kv(\"path\", request.getPath()));\n    \n    return chain.filter(exchange.mutate().request(request).build())\n        .doFinally(signal -> {\n            log.info(\"Request completed\",\n                kv(\"request_id\", requestId),\n                kv(\"status\", exchange.getResponse().getStatusCode()),\n                kv(\"duration_ms\", getDuration()));\n        });\n}\n```\n\n## Backend for Frontend (BFF)\n\n### Concept\n\nDifferent clients (web, mobile, IoT) have different needs:\n- **Mobile**: Battery-conscious, limited bandwidth, simple payloads\n- **Web**: Rich features, large screens, complex data\n- **IoT**: Minimal payloads, specific fields\n\n**Solution**: Separate gateway per client type.\n\n### Architecture\n\n```\n\u250c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2510\n\u2502  Web Client   \u2502\n\u2514\u2500\u2500\u2500\u2500\u2500\u252c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518\n     \u2502\n     \u2193\n\u250c\u2500\u2500\u2500\u2500\u2500\u2534\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2510      \u250c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2510      \u250c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2510\n\u2502  Web BFF    \u2502      \u2502  Mobile BFF   \u2502      \u2502   IoT BFF     \u2502\n\u2502 (Node.js)  \u2502      \u2502  (Node.js)    \u2502      \u2502  (Go)         \u2502\n\u2514\u2500\u2500\u2500\u252c\u2500\u2500\u252c\u2500\u2500\u252c\u2500\u2500\u2500\u2518      \u2514\u2500\u2500\u2500\u252c\u2500\u2500\u252c\u2500\u2500\u252c\u2500\u2500\u2500\u2500\u2500\u2518      \u2514\u2500\u2500\u2500\u252c\u2500\u2500\u2500\u252c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518\n    \u2502  \u2502  \u2502            \u2502  \u2502  \u2502                \u2502   \u2502\n    \u2502  \u2502  \u2514\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u253c\u2500\u2500\u253c\u2500\u2500\u253c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u253c\u2500\u2500\u2500\u253c\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2510\n    \u2502  \u2502              \u2502  \u2502  \u2502                \u2502   \u2502         \u2502\n    \u2193  \u2193              \u2193  \u2193  \u2193                \u2193   \u2193         \u2502\n\u250c\u2500\u2500\u2500\u2500\u2500\u2534\u2500\u2500\u2500\u2500\u2510  \u250c\u2500\u2500\u2500\u2500\u2500\u2534\u2500\u2500\u2500\u2500\u2510  \u250c\u2500\u2500\u2500\u2500\u2500\u2534\u2500\u2500\u2500\u2500\u2500\u2510  \u250c\u2500\u2500\u2500\u2500\u2500\u2534\u2500\u2500\u2500\u2500\u2500\u2510     \u2502\n\u2502  Orders  \u2502  \u2502 Customers \u2502  \u2502  Products \u2502  \u2502  Payments \u2502     \u2502\n\u2502 Service \u2502  \u2502  Service   \u2502  \u2502  Service  \u2502  \u2502  Service  \u2502     \u2502\n\u2514\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518  \u2514\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518  \u2514\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518  \u2514\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518     \u2502\n                                                          \u2502\n                     Microservices                        \u2502\n                                                          \u2514\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2500\u2518\n```\n\n### Mobile BFF Example\n\n**Web Endpoint** (rich data):\n```json\nGET /orders/ord-123 (Web BFF)\n{\n  \"id\": \"ord-123\",\n  \"customer\": {\n    \"id\": \"cust-456\",\n    \"name\": \"John Doe\",\n    \"email\": \"john@example.com\",\n    \"phone\": \"+1234567890\",\n    \"address\": { /* full address */ },\n    \"loyaltyPoints\": 1250\n  },\n  \"items\": [\n    {\n      \"product\": { /* full product details */ },\n      \"quantity\": 2,\n      \"subtotal\": 50.00\n    }\n  ],\n  \"shipments\": [ /* array of shipments */ ],\n  \"invoices\": [ /* array of invoices */ ]\n}\n```\n\n**Mobile Endpoint** (minimal data):\n```json\nGET /orders/ord-123 (Mobile BFF)\n{\n  \"id\": \"ord-123\",\n  \"status\": \"shipped\",\n  \"total\": 50.00,\n  \"itemCount\": 2,\n  \"trackingUrl\": \"https://tracking.example.com/...\"\n}\n```\n\n**Mobile BFF Implementation**:\n```javascript\napp.get('/orders/:id', async (req, res) => {\n  // Call services in parallel\n  const [order, customer, trackingInfo] = await Promise.all([\n    orderService.getOrder(req.params.id),\n    customerService.getCustomer(order.customerId),\n    shippingService.getTracking(order.trackingNumber)\n  ]);\n  \n  // Return minimal mobile-optimized response\n  res.json({\n    id: order.id,\n    status: order.status,\n    total: order.total,\n    itemCount: order.items.length,\n    trackingUrl: trackingInfo.url,\n    // No nested objects, no extra fields\n  });\n});\n```\n\n## API Composition\n\n**Problem**: Client needs data from multiple services\n\n```\nClient wants order details:\n- Order info (Order Service)\n- Customer info (Customer Service)\n- Product details (Product Service)\n- Shipping status (Shipping Service)\n\nWithout gateway: 4 HTTP calls from client\nWith gateway: 1 HTTP call; gateway makes 4 internal calls\n```\n\n**Implementation**:\n```java\n@GetMapping(\"/orders/{id}/complete\")\npublic Mono<CompleteOrderView> getCompleteOrder(@PathVariable String id) {\n    return orderService.getOrder(id)\n        .flatMap(order -> {\n            // Parallel calls to multiple services\n            Mono<Customer> customer = customerService.getCustomer(order.getCustomerId());\n            Mono<List<Product>> products = getProductsForOrder(order);\n            Mono<Shipment> shipment = shippingService.getShipment(order.getShipmentId());\n            \n            return Mono.zip(customer, products, shipment)\n                .map(tuple -> new CompleteOrderView(\n                    order,\n                    tuple.getT1(),  // customer\n                    tuple.getT2(),  // products\n                    tuple.getT3()   // shipment\n                ));\n        });\n}\n```\n\n## Circuit Breaking\n\n**Problem**: Failing service causes cascading failures\n\n**Solution**: Fail fast; return cached/default response\n\n```java\n@CircuitBreaker(name = \"orderService\", fallbackMethod = \"getOrderFallback\")\npublic Order getOrder(String id) {\n    return restTemplate.getForObject(\n        \"http://order-service/orders/\" + id,\n        Order.class\n    );\n}\n\npublic Order getOrderFallback(String id, Exception e) {\n    // Return cached order if available\n    return cacheService.get(\"order:\" + id)\n        .orElseGet(() -> Order.unavailable(id));\n}\n```\n\n**States**:\n```\nClosed (normal) \u2192 requests flow through\n   \u2193 (failure threshold reached)\nOpen \u2192 requests fail fast (no calls to service)\n   \u2193 (after timeout)\nHalf-Open \u2192 try a few requests\n   \u2193 (success) \u2502\n Closed      \u2502 (failure)\n              \u2193\n             Open\n```\n\n## Service Mesh vs Gateway\n\n### When to Use What\n\n| Concern | Gateway | Service Mesh |\n|---------|---------|-------------|\n| Client \u2192 Services | \u2713 | |\n| Service \u2192 Service | | \u2713 |\n| Authentication | \u2713 | \u2713 (mTLS) |\n| Rate Limiting | \u2713 | |\n| Protocol Translation | \u2713 | |\n| Retry/Circuit Breaking | \u2713 | \u2713 |\n| Observability | \u2713 | \u2713 |\n| Load Balancing | \u2713 | \u2713 |\n\n**Often Used Together**: Gateway for north-south traffic, mesh for east-west\n\n## Best Practices\n\n1. **Keep Gateway Thin**: Business logic belongs in services\n2. **Stateless**: No session state; scale horizontally\n3. **Fail Fast**: Timeouts and circuit breakers everywhere\n4. **Observability**: Log, trace, meter all traffic\n5. **Security**: Centralize auth; validate at edge\n6. **Resilience**: Deploy across AZs; health checks\n7. **Avoid Over-Aggregation**: Tight coupling; prefer client-side composition when reasonable\n\n## Anti-Patterns\n\n\u274c **Gateway as Orchestrator**: Complex business workflows\n\u274c **Shared State**: Database in gateway\n\u274c **Tight Coupling**: Gateway knows service internals\n\u274c **Single Gateway**: No separation of concerns\n\u274c **No Timeout**: Requests hang forever\n\u274c **No Fallback**: One service down = all down
