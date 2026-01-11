# Idempotency & Concurrency Control

## What You'll Learn
How to design APIs that handle retries safely, prevent duplicate operations, and manage concurrent updates. You'll understand idempotency keys, retry strategies, and concurrency control patterns.

## Why This Matters
Networks are unreliable. Clients will retry failed requests. Without idempotency, retries cause duplicates: double charges, duplicate orders, repeated emails. In distributed systems, "exactly-once" is a myth—you achieve at-most-once (lose data) or at-least-once (potential duplicates). With idempotency, at-least-once becomes effectively exactly-once. Concurrency bugs (lost updates, race conditions) are subtle and often discovered in production. These patterns are essential for reliable systems.

## Idempotency Fundamentals

### Definition

**Idempotent Operation**: Applying it multiple times has same effect as applying once.

**Mathematical**: `f(f(x)) = f(x)`

**API Context**: Calling same endpoint with same parameters multiple times produces same result and side effects.

### HTTP Method Idempotency

| Method | Idempotent | Safe | Description |
|--------|------------|------|-------------|
| GET | Yes | Yes | Read; no side effects |
| PUT | Yes | No | Replace; same result every time |
| DELETE | Yes | No | Remove; deleting twice = still deleted |
| PATCH | No* | No | Update; depends on implementation |
| POST | No | No | Create/action; each call may create new |

*PATCH can be designed to be idempotent

### Why POST Isn't Idempotent

```http
POST /orders
{"item": "book", "quantity": 1}

Call 1 → Order ID: ord-1
Call 2 → Order ID: ord-2  // Duplicate!
Call 3 → Order ID: ord-3  // Another duplicate!
```

**Problem**: Network timeout after submitting payment
```
Client                  Server                  Payment Gateway
  |─POST /orders───────>|                      |
  |                     |─charge card──────>|
  |                     |<───$100 charged──|
  |  X  (timeout)      |                      |
  |                     (order created)       |
  |─POST /orders───────>|  (retry)             |
  |                     |─charge card──────>|
  |                     |<───$100 charged──|
  |<────200 OK──────|                      |
  
Result: Customer charged $200, only got 1 order!
```

## Idempotency Keys

### Pattern

Client generates unique key per operation; includes in header.

```http
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{"item": "book", "quantity": 1}
```

**Server Logic**:
1. Check if key exists in store (Redis, DB)
2. If exists: return cached response (idempotent)
3. If new: process request, store key + response, return response

### Implementation

```java
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(
    @RequestHeader("Idempotency-Key") String idempotencyKey,
    @RequestBody CreateOrderRequest request
) {
    // Check cache
    CachedResponse cached = idempotencyStore.get(idempotencyKey);
    if (cached != null) {
        if (cached.isProcessing()) {
            return ResponseEntity
                .status(HttpStatus.CONFLICT)
                .body("Request is being processed");
        }
        // Return cached result
        return ResponseEntity
            .status(cached.getStatus())
            .body(cached.getBody());
    }
    
    // Mark as processing (prevent concurrent requests with same key)
    idempotencyStore.setProcessing(idempotencyKey);
    
    try {
        // Process request
        Order order = orderService.create(request);
        
        // Cache result (TTL: 24 hours)
        idempotencyStore.set(
            idempotencyKey, 
            new CachedResponse(HttpStatus.CREATED, order),
            Duration.ofHours(24)
        );
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(order);
            
    } catch (Exception e) {
        // Remove processing flag
        idempotencyStore.delete(idempotencyKey);
        throw e;
    }
}
```

### Key Generation (Client)

**Option 1: UUID v4** (random)
```javascript
const idempotencyKey = crypto.randomUUID();
// 550e8400-e29b-41d4-a716-446655440000
```

**Option 2: Hash of request** (deterministic)
```javascript
const idempotencyKey = sha256(JSON.stringify({
    operation: 'create_order',
    userId: user.id,
    items: request.items,
    timestamp: Date.now()
}));
```

**Recommendation**: UUID v4; simpler, no hash collisions

### Scope & TTL

**Scope**: Per user + operation type
```
Key: idempotency:user-123:create-order:550e8400-e29b-41d4-a716-446655440000
```

**TTL**: 24-72 hours
- Long enough for reasonable retry window
- Short enough to avoid unbounded storage
- Document in API: "Keys expire after 24h"

### Edge Cases

**Different request body, same key**:
```http
Call 1:
Idempotency-Key: abc-123
{"amount": 100}
→ Creates order for $100

Call 2:
Idempotency-Key: abc-123
{"amount": 200}  // Different body!
→ Returns cached $100 order (ignores new body)
```

**Decision**: Return cached result (strict idempotency) OR reject with 422 (detect misuse).

**Recommendation**: Return cached + warning header
```http
X-Idempotency-Replayed: true
```

## Retry Strategies

### Exponential Backoff with Jitter

**Formula**: `delay = min(max_delay, base * 2^attempt) + random(0, jitter)`

```javascript
function calculateBackoff(attempt, baseMs = 100, maxMs = 30000) {
    const exponential = Math.min(maxMs, baseMs * Math.pow(2, attempt));
    const jitter = Math.random() * exponential * 0.1;  // 10% jitter
    return exponential + jitter;
}

// Attempt 0: 100ms + jitter
// Attempt 1: 200ms + jitter
// Attempt 2: 400ms + jitter
// Attempt 3: 800ms + jitter
// Attempt 4: 1600ms + jitter
// ...
// Attempt 10: 30000ms + jitter (capped)
```

**Why Jitter**: Prevent thundering herd (all clients retry at same time)

### Retry Budget

**Limit retries**: Avoid infinite loops
```javascript
const MAX_RETRIES = 3;
let attempt = 0;

while (attempt < MAX_RETRIES) {
    try {
        const response = await fetch('/orders', {
            headers: {'Idempotency-Key': idempotencyKey}
        });
        return response;
    } catch (error) {
        if (!isRetryable(error) || attempt >= MAX_RETRIES) {
            throw error;
        }
        await sleep(calculateBackoff(attempt));
        attempt++;
    }
}
```

### Which Errors to Retry

**Retryable** (transient failures):
- Network errors (timeout, connection refused)
- 429 Too Many Requests
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout

**Non-Retryable** (client errors, won't succeed):
- 400 Bad Request (malformed)
- 401 Unauthorized (invalid auth)
- 403 Forbidden (lacks permission)
- 404 Not Found
- 422 Unprocessable Entity (validation)

```javascript
function isRetryable(error) {
    const retryableStatusCodes = [429, 500, 502, 503, 504];
    return (
        error.code === 'ECONNRESET' ||
        error.code === 'ETIMEDOUT' ||
        (error.response && retryableStatusCodes.includes(error.response.status))
    );
}
```

### Respect Retry-After

```javascript
if (response.status === 429) {
    const retryAfter = response.headers.get('Retry-After');
    if (retryAfter) {
        await sleep(parseInt(retryAfter) * 1000);
        // Then retry
    }
}
```
