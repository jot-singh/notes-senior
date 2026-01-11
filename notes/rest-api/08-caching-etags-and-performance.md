# Caching, ETags, and Performance

## What You'll Learn
How to leverage HTTP caching to reduce latency, server load, and costs. You'll understand caching headers, validation strategies, ETags for concurrency control, and performance optimization techniques.

## Why This Matters
Every request that hits your origin servers costs CPU, database queries, and money. Good caching can eliminate 80-95% of requests, delivering sub-100ms responses from CDN edge. Bad caching causes stale data bugs or cache invalidation storms. Understanding HTTP caching turns your API from expensive and slow to cheap and fast. For high-traffic APIs, caching is the difference between $1000/month and $100,000/month in infrastructure costs.

## Caching Layers

### 1. Client Cache (Browser/App)
**Location**: User's device
**Control**: `Cache-Control` headers
**Scope**: Single user
**Speed**: Instant (0ms)

**Example**: Mobile app caches user profile for 5 minutes
```http
GET /users/me
→ 200 OK
Cache-Control: private, max-age=300

// Next request within 5 min: served from app cache, no network
```

### 2. CDN/Proxy Cache
**Location**: Edge servers (Cloudflare, Fastly, AWS CloudFront)
**Control**: `Cache-Control`, `Vary`
**Scope**: All users in region
**Speed**: 10-50ms

**Example**: Product catalog cached at edge
```http
GET /products/shoe-123
→ 200 OK
Cache-Control: public, max-age=3600, s-maxage=7200

// Cached at CDN for 2 hours (s-maxage)
// 1M requests/hour = 1 origin request per 2 hours
```

### 3. Server Cache (Redis/Memcached)
**Location**: Backend infrastructure
**Control**: Application logic
**Scope**: All users
**Speed**: 1-10ms

**Example**: Database query results cached
```java
@Cacheable(value = "orders", key = "#orderId")
public Order getOrder(String orderId) {
    // Only hits DB on cache miss
    return orderRepository.findById(orderId);
}
```

## HTTP Cache Headers

### Cache-Control Directives

**`max-age=<seconds>`**: How long response is fresh
```http
Cache-Control: max-age=3600
// Fresh for 1 hour, then stale
```

**`s-maxage=<seconds>`**: Shared cache (CDN) freshness
```http
Cache-Control: max-age=300, s-maxage=3600
// Browsers cache 5 min, CDN caches 1 hour
```

**`public`**: Cacheable by any cache (CDN, proxies)
```http
Cache-Control: public, max-age=3600
// Safe for CDN: no user-specific data
```

**`private`**: Only client cache (not CDN)
```http
Cache-Control: private, max-age=300
// User-specific data (profile, orders)
```

**`no-cache`**: Must revalidate before use (not "don't cache"!)
```http
Cache-Control: no-cache
// Cached but must validate with If-None-Match before serving
```

**`no-store`**: Never cache (sensitive data)
```http
Cache-Control: no-store
// Credit card data, passwords, OTPs
```

**`must-revalidate`**: Don't serve stale; always validate after expiry
```http
Cache-Control: max-age=3600, must-revalidate
```

**`stale-while-revalidate=<seconds>`**: Serve stale while fetching fresh
```http
Cache-Control: max-age=3600, stale-while-revalidate=60
// After 1h: serve stale immediately, fetch fresh in background
// User sees instant response, next user gets fresh
```

**`stale-if-error=<seconds>`**: Serve stale if origin errors
```http
Cache-Control: max-age=600, stale-if-error=86400
// If origin down, serve up to 24h stale data
// Resilience during outages
```

### Validation Headers

#### ETag (Entity Tag)

**What**: Opaque identifier for resource version

**Strong ETag**: Byte-for-byte identical
```http
ETag: "v2.5.1"
ETag: "sha256:a3d5e9..."
```

**Weak ETag**: Semantically equivalent
```http
ETag: W/"v2.5"
// Minor formatting changes don't change weak ETag
```

**Generation Strategies**:
1. **Hash content**: `MD5(responseBody)` or `SHA256(content)`
2. **Version number**: `updated_at` timestamp or row version
3. **Composite**: `{id}-{updated_at}`

**Example**:
```java
String etag = "\"" + order.getId() + "-" + order.getVersion() + "\"";
return ResponseEntity
    .ok()
    .eTag(etag)
    .body(order);
```

#### Last-Modified

**What**: Timestamp of last modification
```http
Last-Modified: Wed, 11 Jan 2026 10:30:00 GMT
```

**Less precise than ETag**: 1-second granularity

### Conditional Requests

#### GET with If-None-Match (ETag validation)

**Client Side**:
```http
// First request
GET /orders/123
→ 200 OK
ETag: "v5"
Cache-Control: max-age=300
{"id": "123", "status": "pending"}

// 5 minutes later (stale), revalidate:
GET /orders/123
If-None-Match: "v5"

// If unchanged:
→ 304 Not Modified
ETag: "v5"
// No body, client uses cached version

// If changed:
→ 200 OK
ETag: "v6"
{"id": "123", "status": "shipped"}  // New data
```

**Bandwidth Savings**: 304 response is ~200 bytes vs ~5KB full response

#### GET with If-Modified-Since

```http
GET /orders/123
If-Modified-Since: Wed, 11 Jan 2026 10:00:00 GMT

→ 304 Not Modified  (if not modified since)
→ 200 OK with body  (if modified)
```

## Concurrency Control with ETags

### Optimistic Locking

**Problem**: Two clients update same resource simultaneously
```
Time    Client A                Client B
10:00   GET /orders/123         GET /orders/123
        (status: pending)       (status: pending)
10:01   PATCH (set shipped)     
10:02                           PATCH (set cancelled)
        ✓ Success                ✓ Success
        Result: cancelled (B overwrites A's change!)
```

**Solution**: Conditional updates with If-Match
```
Time    Client A                          Client B
10:00   GET /orders/123                   GET /orders/123
        ETag: "v5"                         ETag: "v5"
        
10:01   PATCH /orders/123
        If-Match: "v5"
        (set shipped)
        → 200 OK, ETag: "v6"
        
10:02                                     PATCH /orders/123
                                          If-Match: "v5"
                                          (set cancelled)
                                          → 412 Precondition Failed
                                          ETag: "v6"
                                          
        Client B must refetch and retry
```

**Implementation**:
```java
@PatchMapping("/orders/{id}")
public ResponseEntity<Order> updateOrder(
    @PathVariable String id,
    @RequestHeader("If-Match") String ifMatch,
    @RequestBody OrderUpdate update
) {
    Order order = orderService.get(id);
    String currentETag = "\"" + order.getVersion() + "\"";
    
    if (!currentETag.equals(ifMatch)) {
        return ResponseEntity
            .status(HttpStatus.PRECONDITION_FAILED)
            .eTag(currentETag)
            .build();
    }
    
    Order updated = orderService.update(id, update);
    return ResponseEntity
        .ok()
        .eTag("\"" + updated.getVersion() + "\"")
        .body(updated);
}
```
