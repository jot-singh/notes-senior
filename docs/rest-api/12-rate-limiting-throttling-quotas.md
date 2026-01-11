# Rate Limiting, Throttling, and Quotas

## What You'll Learn
How to protect your API from abuse, ensure fair resource allocation, and maintain service quality under load. You'll understand different rate limiting algorithms, implementation strategies, and how to communicate limits to clients effectively.

## Why This Matters
Without rate limiting, a single misbehaving client can take down your entire API. A bug causing infinite retry loops, a DDoS attack, or a misconfigured integration can overwhelm your infrastructure. Rate limiting protects infrastructure, ensures fair usage, prevents abuse, and enables tiered pricing models. It's the difference between a $100/month AWS bill and a $100,000 surprise. Good rate limiting is invisible to normal users but essential for reliability.

## Rate Limiting Algorithms

### 1. Token Bucket

**How It Works**:
- Bucket holds tokens (max capacity)
- Tokens refill at fixed rate
- Each request consumes 1 token
- Request allowed if tokens available
- Allows bursts up to bucket size

**Visualization**:
```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Time  Tokens  Action
0s    100     [Request] → 99 tokens
0.1s  100     [Request] → 99 (refilled to 100, then consumed)
0.5s  100     [50 requests in burst] → 50 tokens
1s    60      (refilled 10 tokens)
5s    100     (refilled to max)
```

**Implementation** (Redis + Lua):
```lua
-- token_bucket.lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Calculate tokens to add
local elapsed = now - last_refill
local tokens_to_add = elapsed * refill_rate
tokens = math.min(capacity, tokens + tokens_to_add)

-- Check if request allowed
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return {1, tokens}  -- Allowed
else
    return {0, tokens}  -- Denied
end
```

**Java Implementation**:
```java
public class TokenBucketRateLimiter {
    private final int capacity;
    private final double refillRate;  // tokens per second
    private double tokens;
    private long lastRefill;
    
    public TokenBucketRateLimiter(int capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefill = System.nanoTime();
    }
    
    public synchronized boolean tryConsume(int tokensRequested) {
        refill();
        
        if (tokens >= tokensRequested) {
            tokens -= tokensRequested;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefill) / 1_000_000_000.0;  // seconds
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefill = now;
    }
    
    public synchronized int getAvailableTokens() {
        refill();
        return (int) tokens;
    }
}
```

**Pros**:
- Allows bursts (good UX)
- Smooth average rate
- Simple to understand

**Cons**:
- Large bursts can still overwhelm
- Memory per client

### 2. Leaky Bucket

**How It Works**:
- Requests enter bucket (queue)
- Processed at fixed rate
- Bucket overflows if full (reject)
- Smooths traffic spikes

**Visualization**:
```
    Requests
       ↓↓↓
    [======]  ← Bucket (queue)
       ↓
    Fixed rate out
```

**Use Case**: Protecting downstream services with strict rate requirements

**Pros**:
- Perfectly smooth output
- Protects downstream

**Cons**:
- Adds latency (queueing)
- No burst allowance
- Less intuitive

### 3. Fixed Window

**How It Works**:
- Count requests in time window (e.g., per minute)
- Reset counter at window boundary

```
Window 1 (00:00-00:59): 100 requests allowed
Window 2 (01:00-01:59): 100 requests allowed

00:59 → 100 requests ✓
01:00 → Counter resets
01:00 → 100 requests ✓

Problem: 200 requests in 1 second (00:59-01:00) 💥
```

**Implementation**:
```java
public boolean isAllowed(String clientId) {
    String key = "ratelimit:" + clientId + ":" + getCurrentWindow();
    long count = redis.incr(key);
    
    if (count == 1) {
        redis.expire(key, 60);  // Window size
    }
    
    return count <= LIMIT;
}

private long getCurrentWindow() {
    return System.currentTimeMillis() / 60000;  // Minute windows
}
```

**Pros**:
- Simple
- Memory efficient
- Predictable resets

**Cons**:
- Boundary burst issue
- Uneven distribution

### 4. Sliding Window Log

**How It Works**:
- Store timestamp of each request
- Count requests in last N seconds
- Remove old entries

```
Limit: 10 requests per minute

Timestamps: [10:00:15, 10:00:18, 10:00:22, ...]

At 10:01:00:
- Remove entries < 10:00:00
- Count remaining entries
- Allow if count < 10
```

**Implementation** (Redis sorted set):
```java
public boolean isAllowed(String clientId) {
    String key = "ratelimit:" + clientId;
    long now = System.currentTimeMillis();
    long windowStart = now - 60_000;  // 1 minute
    
    // Remove old entries
    redis.zremrangeByScore(key, 0, windowStart);
    
    // Count current requests
    long count = redis.zcard(key);
    
    if (count < LIMIT) {
        redis.zadd(key, now, UUID.randomUUID().toString());
        redis.expire(key, 60);
        return true;
    }
    return false;
}
```

**Pros**:
- No boundary issue
- Precise
- Fair

**Cons**:
- Memory intensive (stores all timestamps)
- Expensive at high scale

### 5. Sliding Window Counter (Hybrid)

**How It Works**:
- Combine fixed window efficiency with sliding window fairness
- Estimate current window using previous and current

```
Previous window (00:00-00:59): 80 requests
Current window  (01:00-01:59): 30 requests

At 01:30 (50% into current window):
Estimate = (previous * 50%) + current
         = (80 * 0.5) + 30
         = 70 requests

Allow if 70 < 100 ✓
```

**Implementation**:
```java
public boolean isAllowed(String clientId) {
    long now = System.currentTimeMillis();
    long currentWindow = now / 60000;
    long previousWindow = currentWindow - 1;
    
    String currentKey = "ratelimit:" + clientId + ":" + currentWindow;
    String previousKey = "ratelimit:" + clientId + ":" + previousWindow;
    
    long currentCount = redis.incr(currentKey);
    if (currentCount == 1) {
        redis.expire(currentKey, 120);
    }
    
    long previousCount = redis.get(previousKey);
    
    // Calculate weight of previous window
    double elapsedInCurrentWindow = (now % 60000) / 60000.0;
    double previousWeight = 1.0 - elapsedInCurrentWindow;
    
    double estimatedCount = previousCount * previousWeight + currentCount;
    
    return estimatedCount <= LIMIT;
}
```

**Pros**:
- Memory efficient
- Smooth rate limiting
- No boundary burst

**Cons**:
- Approximation (not exact)
- Slightly complex

### Algorithm Comparison

| Algorithm | Burst Friendly | Memory | Precision | Complexity |
|-----------|----------------|--------|-----------|------------|
| Token Bucket | Yes | Low | Good | Medium |
| Leaky Bucket | No | Low | Perfect | Medium |
| Fixed Window | No | Very Low | Poor | Low |
| Sliding Log | No | High | Perfect | High |
| Sliding Counter | Moderate | Low | Good | Medium |

**Recommendation**: Token Bucket for most APIs; Sliding Window Counter for strict fairness.

## Response Headers

### Standard Rate Limit Headers (IETF Draft)

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 73
RateLimit-Reset: 1673451600
```

**Fields**:
- `RateLimit-Limit`: Max requests in window
- `RateLimit-Remaining`: Requests left
- `RateLimit-Reset`: Unix timestamp when limit resets

### When Limit Exceeded

```http
HTTP/1.1 429 Too Many Requests
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1673451600
Retry-After: 60
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limit-exceeded",
  "title": "Rate Limit Exceeded",
  "status": 429,
  "detail": "API rate limit of 100 requests per minute exceeded",
  "retry_after": 60,
  "limit": 100,
  "window": "1 minute"
}
```

**Client Handling**:
```javascript
async function makeRequest(url) {
  const response = await fetch(url);
  
  // Check rate limit headers
  const remaining = parseInt(response.headers.get('RateLimit-Remaining'));
  const reset = parseInt(response.headers.get('RateLimit-Reset'));
  
  if (remaining < 10) {
    console.warn(`Only ${remaining} requests remaining`);
  }
  
  if (response.status === 429) {
    const retryAfter = parseInt(response.headers.get('Retry-After'));
    console.log(`Rate limited. Retry after ${retryAfter} seconds`);
    await sleep(retryAfter * 1000);
    return makeRequest(url);  // Retry
  }
  
  return response;
}
```

## Multi-Dimensional Rate Limiting

### Dimensions

**Per User**:
```
Key: ratelimit:user:{userId}
Limit: 1000/hour
```

**Per API Key**:
```
Key: ratelimit:apikey:{keyId}
Limit: 10000/hour (paid tier)
```

**Per IP**:
```
Key: ratelimit:ip:{ipAddress}
Limit: 100/minute (anti-abuse)
```

**Per Tenant** (multi-tenant SaaS):
```
Key: ratelimit:tenant:{tenantId}
Limit: 50000/day
```

**Per Endpoint**:
```
Key: ratelimit:user:{userId}:endpoint:POST:/orders
Limit: 100/minute
```

### Tiered Limits

```java
public class TieredRateLimiter {
    
    public RateLimit getLimitForClient(String clientId) {
        Client client = clientRepository.findById(clientId);
        
        return switch (client.getTier()) {
            case FREE -> new RateLimit(100, Duration.ofMinutes(1));
            case BASIC -> new RateLimit(1000, Duration.ofMinutes(1));
            case PREMIUM -> new RateLimit(10000, Duration.ofMinutes(1));
            case ENTERPRISE -> new RateLimit(100000, Duration.ofMinutes(1));
        };
    }
}
```

### Weighted Endpoints

**Expensive operations cost more**:

```java
@RateLimit(cost = 1)
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable String id) { ... }

@RateLimit(cost = 10)  // Costs 10 tokens
@PostMapping("/reports/generate")
public Report generateReport(@RequestBody ReportRequest request) { ... }

@RateLimit(cost = 100)  // Very expensive
@PostMapping("/bulk-import")
public void bulkImport(@RequestBody List<Order> orders) { ... }
```

## Implementation Patterns

### Middleware/Filter

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    @Autowired
    private RateLimiter rateLimiter;
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        String clientId = extractClientId(request);
        
        if (!rateLimiter.tryAcquire(clientId)) {
            response.setStatus(429);
            response.setHeader("RateLimit-Limit", "100");
            response.setHeader("RateLimit-Remaining", "0");
            response.setHeader("RateLimit-Reset", String.valueOf(getResetTime()));
            response.setHeader("Retry-After", "60");
            response.setContentType("application/problem+json");
            response.getWriter().write(createErrorResponse());
            return;
        }
        
        filterChain.doFilter(request, response);
    }
}
```

### API Gateway

**Kong Configuration**:
```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 100
      hour: 5000
      policy: redis
      redis_host: redis.example.com
      redis_port: 6379
      fault_tolerant: true
```

**AWS API Gateway**:
```json
{
  "throttle": {
    "rateLimit": 1000,
    "burstLimit": 2000
  },
  "quota": {
    "limit": 50000,
    "period": "DAY"
  }
}
```

## Monitoring & Analytics

**Metrics to Track**:
- Rate limit hit rate (% of 429 responses)
- Per-client usage distribution
- Peak usage times
- Quota utilization

**Dashboard**:
```
Top Consumers:
1. client-abc: 95% of quota (47,500 / 50,000)
2. client-xyz: 80% of quota (40,000 / 50,000)

Rate Limit Violations (last hour):
- client-def: 234 requests rejected
- IP 1.2.3.4: 1,045 requests rejected (potential attack)

Quota Exhaustion Alerts:
- client-ghi will exhaust daily quota in 2 hours
```

## Best Practices

1. **Communicate clearly**: Document limits in API docs
2. **Be generous**: Set limits higher than normal usage
3. **Provide headers**: Always include rate limit headers
4. **Graceful degradation**: Return 429, not 5xx
5. **Per-client tracking**: Don't punish all for one abuser
6. **Priority lanes**: Critical operations bypass limits
7. **Burst tolerance**: Allow reasonable bursts (token bucket)
8. **Monitoring**: Alert on unusual patterns
9. **Self-service**: Let clients see their usage in dashboard
