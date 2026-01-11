# Rate Limiting and Throttling

## What You'll Learn

Rate limiting controls the number of requests a client can make to protect services from overload, prevent abuse, and ensure fair resource distribution. This note covers rate limiting algorithms, implementation strategies, distributed rate limiting, and production patterns used at companies like Twitter, Stripe, and GitHub.

## Why This Matters

Without rate limiting, services are vulnerable to overload and abuse. A single misbehaving client can overwhelm servers with millions of requests. DDoS attacks can take down services. Rate limiting protects infrastructure, ensures fair usage, and provides predictable performance. Twitter limits API calls to 900 requests per 15 minutes. Stripe limits API calls based on subscription tier. Understanding rate limiting is essential for building production-ready APIs.

## Rate Limiting Algorithms

### Token Bucket

The token bucket algorithm maintains a bucket with fixed capacity. Tokens are added at a constant rate. Each request consumes one token. If no tokens are available, the request is rejected.

This algorithm allows bursts up to the bucket capacity while enforcing average rate over time. A bucket with 100 tokens refilling at 10 tokens/second allows 100 requests immediately, then 10 requests/second.

```python
# Python: Token bucket implementation
import time
import threading

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        """
        capacity: Maximum tokens in bucket
        refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = threading.Lock()
    
    def _refill(self):
        """Add tokens based on time elapsed"""
        now = time.time()
        time_elapsed = now - self.last_refill
        
        # Calculate tokens to add
        tokens_to_add = time_elapsed * self.refill_rate
        
        # Update bucket
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
    
    def consume(self, tokens=1):
        """Try to consume tokens, return True if successful"""
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            
            return False
    
    def get_wait_time(self, tokens=1):
        """Get seconds to wait before tokens available"""
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens:
                return 0
            
            tokens_needed = tokens - self.tokens
            return tokens_needed / self.refill_rate

# Usage
bucket = TokenBucket(capacity=100, refill_rate=10)

# Allow burst of 100 requests
for i in range(100):
    assert bucket.consume() == True

# 101st request fails
assert bucket.consume() == False

# Wait for refill
time.sleep(1)  # 10 tokens refilled

# Can make 10 more requests
for i in range(10):
    assert bucket.consume() == True
```

```java
// Java: Thread-safe token bucket
public class TokenBucket {
    private final long capacity;
    private final long refillRate;  // tokens per second
    private long tokens;
    private long lastRefillTimestamp;
    private final Object lock = new Object();
    
    public TokenBucket(long capacity, long refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefillTimestamp = System.nanoTime();
    }
    
    private void refill() {
        long now = System.nanoTime();
        long elapsedNanos = now - lastRefillTimestamp;
        
        // Convert nanoseconds to seconds
        double elapsedSeconds = elapsedNanos / 1_000_000_000.0;
        
        // Calculate tokens to add
        long tokensToAdd = (long) (elapsedSeconds * refillRate);
        
        if (tokensToAdd > 0) {
            tokens = Math.min(capacity, tokens + tokensToAdd);
            lastRefillTimestamp = now;
        }
    }
    
    public boolean tryConsume(long tokensToConsume) {
        synchronized (lock) {
            refill();
            
            if (tokens >= tokensToConsume) {
                tokens -= tokensToConsume;
                return true;
            }
            
            return false;
        }
    }
    
    public long getWaitTimeMillis(long tokensNeeded) {
        synchronized (lock) {
            refill();
            
            if (tokens >= tokensNeeded) {
                return 0;
            }
            
            long tokensShort = tokensNeeded - tokens;
            return (tokensShort * 1000) / refillRate;
        }
    }
    
    public long availableTokens() {
        synchronized (lock) {
            refill();
            return tokens;
        }
    }
}

// Usage in API controller
@RestController
public class ApiController {
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    private static final long RATE_LIMIT = 100;  // 100 requests per minute
    
    @GetMapping("/api/resource")
    public ResponseEntity<?> getResource(
            @RequestHeader("X-API-Key") String apiKey) {
        
        TokenBucket bucket = buckets.computeIfAbsent(
            apiKey,
            k -> new TokenBucket(RATE_LIMIT, RATE_LIMIT / 60)
        );
        
        if (!bucket.tryConsume(1)) {
            long waitMillis = bucket.getWaitTimeMillis(1);
            
            return ResponseEntity
                .status(HttpStatus.TOO_MANY_REQUESTS)
                .header("X-RateLimit-Limit", String.valueOf(RATE_LIMIT))
                .header("X-RateLimit-Remaining", "0")
                .header("X-RateLimit-Reset", 
                    String.valueOf(System.currentTimeMillis() + waitMillis))
                .header("Retry-After", String.valueOf(waitMillis / 1000))
                .body("Rate limit exceeded");
        }
        
        return ResponseEntity.ok()
            .header("X-RateLimit-Limit", String.valueOf(RATE_LIMIT))
            .header("X-RateLimit-Remaining", 
                String.valueOf(bucket.availableTokens()))
            .body(/* response data */);
    }
}
```

### Leaky Bucket

The leaky bucket algorithm processes requests at a constant rate. Requests are added to a queue (bucket). Requests leak out (are processed) at a fixed rate. If the queue is full, new requests are rejected.

This algorithm smooths traffic spikes by enforcing constant output rate. Unlike token bucket which allows bursts, leaky bucket provides perfectly smooth rate.

```javascript
// Node.js: Leaky bucket implementation
class LeakyBucket {
    constructor(capacity, leakRate) {
        this.capacity = capacity;
        this.leakRate = leakRate;  // requests processed per second
        this.queue = [];
        this.processing = false;
    }
    
    async addRequest(request) {
        if (this.queue.length >= this.capacity) {
            throw new Error('Bucket full - rate limit exceeded');
        }
        
        return new Promise((resolve, reject) => {
            this.queue.push({ request, resolve, reject });
            this.startProcessing();
        });
    }
    
    startProcessing() {
        if (this.processing) {
            return;
        }
        
        this.processing = true;
        this.processNext();
    }
    
    async processNext() {
        if (this.queue.length === 0) {
            this.processing = false;
            return;
        }
        
        const { request, resolve, reject } = this.queue.shift();
        
        try {
            const result = await this.handleRequest(request);
            resolve(result);
        } catch (error) {
            reject(error);
        }
        
        // Wait before processing next request (leak rate)
        const delayMs = 1000 / this.leakRate;
        setTimeout(() => this.processNext(), delayMs);
    }
    
    async handleRequest(request) {
        // Process the actual request
        return { status: 'success', data: request };
    }
}

// Usage with Express
const express = require('express');
const app = express();

const buckets = new Map();
const BUCKET_CAPACITY = 100;
const LEAK_RATE = 10;  // 10 requests/second

app.use((req, res, next) => {
    const clientId = req.ip;
    
    if (!buckets.has(clientId)) {
        buckets.set(clientId, new LeakyBucket(BUCKET_CAPACITY, LEAK_RATE));
    }
    
    const bucket = buckets.get(clientId);
    
    bucket.addRequest(req)
        .then(() => next())
        .catch(error => {
            res.status(429).json({
                error: 'Rate limit exceeded',
                retryAfter: Math.ceil(bucket.queue.length / LEAK_RATE)
            });
        });
});
```

### Fixed Window Counter

Fixed window divides time into fixed intervals (windows). Track request count per window. Reset count when window expires. Reject requests exceeding limit in current window.

Simple to implement but can allow bursts at window boundaries. If limit is 100 requests per minute, a client can make 100 requests at 0:59 and 100 at 1:00, totaling 200 requests in 2 seconds.

```python
# Python: Fixed window counter with Redis
import redis
import time

class FixedWindowCounter:
    def __init__(self, redis_client, window_size_seconds=60):
        self.redis = redis_client
        self.window_size = window_size_seconds
    
    def is_allowed(self, client_id, limit):
        """Check if request is allowed"""
        # Calculate current window
        now = int(time.time())
        window = now // self.window_size
        
        key = f"rate_limit:{client_id}:{window}"
        
        # Increment counter
        count = self.redis.incr(key)
        
        # Set expiration on first request in window
        if count == 1:
            self.redis.expire(key, self.window_size * 2)
        
        return count <= limit
    
    def get_remaining(self, client_id, limit):
        """Get remaining requests in current window"""
        now = int(time.time())
        window = now // self.window_size
        key = f"rate_limit:{client_id}:{window}"
        
        count = int(self.redis.get(key) or 0)
        return max(0, limit - count)
    
    def get_reset_time(self):
        """Get timestamp when current window resets"""
        now = int(time.time())
        window = now // self.window_size
        return (window + 1) * self.window_size

# Usage
redis_client = redis.Redis(host='localhost', port=6379, db=0)
limiter = FixedWindowCounter(redis_client, window_size_seconds=60)

client_id = "user123"
limit = 100

if limiter.is_allowed(client_id, limit):
    # Process request
    print("Request allowed")
    print(f"Remaining: {limiter.get_remaining(client_id, limit)}")
else:
    # Reject request
    print("Rate limit exceeded")
    print(f"Reset at: {limiter.get_reset_time()}")
```

### Sliding Window Log

Sliding window log tracks timestamp of each request. For each new request, remove timestamps outside the window and count remaining requests. Allow request if count is below limit.

Provides accurate rate limiting without boundary burst issues. However, requires storing all request timestamps, consuming more memory.

```java
// Java: Sliding window log
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class SlidingWindowLog {
    private final Map<String, Queue<Long>> requestLogs;
    private final long windowSizeMillis;
    
    public SlidingWindowLog(long windowSizeSeconds) {
        this.requestLogs = new ConcurrentHashMap<>();
        this.windowSizeMillis = windowSizeSeconds * 1000;
    }
    
    public synchronized boolean isAllowed(String clientId, int limit) {
        long now = Instant.now().toEpochMilli();
        long windowStart = now - windowSizeMillis;
        
        // Get or create request log for client
        Queue<Long> log = requestLogs.computeIfAbsent(
            clientId,
            k -> new LinkedList<>()
        );
        
        // Remove timestamps outside window
        while (!log.isEmpty() && log.peek() < windowStart) {
            log.poll();
        }
        
        // Check if within limit
        if (log.size() < limit) {
            log.offer(now);
            return true;
        }
        
        return false;
    }
    
    public synchronized int getRemaining(String clientId, int limit) {
        long now = Instant.now().toEpochMilli();
        long windowStart = now - windowSizeMillis;
        
        Queue<Long> log = requestLogs.get(clientId);
        if (log == null) {
            return limit;
        }
        
        // Remove old timestamps
        while (!log.isEmpty() && log.peek() < windowStart) {
            log.poll();
        }
        
        return Math.max(0, limit - log.size());
    }
    
    public long getOldestTimestamp(String clientId) {
        Queue<Long> log = requestLogs.get(clientId);
        if (log == null || log.isEmpty()) {
            return 0;
        }
        return log.peek();
    }
    
    // Cleanup old entries periodically
    public void cleanup() {
        long now = Instant.now().toEpochMilli();
        long windowStart = now - windowSizeMillis;
        
        requestLogs.forEach((clientId, log) -> {
            synchronized (this) {
                while (!log.isEmpty() && log.peek() < windowStart) {
                    log.poll();
                }
                
                // Remove empty logs
                if (log.isEmpty()) {
                    requestLogs.remove(clientId);
                }
            }
        });
    }
}
```

### Sliding Window Counter

Sliding window counter combines fixed window and sliding window log. Maintains counters for current and previous windows. Calculates weighted count based on time within current window.

Provides good accuracy with lower memory usage than sliding window log. Approximation is close enough for most use cases.

```python
# Python: Sliding window counter with Redis
import redis
import time
import math

class SlidingWindowCounter:
    def __init__(self, redis_client, window_size_seconds=60):
        self.redis = redis_client
        self.window_size = window_size_seconds
    
    def is_allowed(self, client_id, limit):
        """Check if request allowed using sliding window"""
        now = time.time()
        current_window = int(now) // self.window_size
        previous_window = current_window - 1
        
        # Time elapsed in current window
        elapsed_time_in_current = now - (current_window * self.window_size)
        
        # Weight for previous window
        previous_weight = 1 - (elapsed_time_in_current / self.window_size)
        
        # Get counts
        current_key = f"rate_limit:{client_id}:{current_window}"
        previous_key = f"rate_limit:{client_id}:{previous_window}"
        
        current_count = int(self.redis.get(current_key) or 0)
        previous_count = int(self.redis.get(previous_key) or 0)
        
        # Calculate weighted count
        weighted_count = (previous_count * previous_weight) + current_count
        
        if weighted_count < limit:
            # Increment current window counter
            pipe = self.redis.pipeline()
            pipe.incr(current_key)
            pipe.expire(current_key, self.window_size * 2)
            pipe.execute()
            
            return True
        
        return False
    
    def get_remaining(self, client_id, limit):
        """Get approximate remaining requests"""
        now = time.time()
        current_window = int(now) // self.window_size
        previous_window = current_window - 1
        
        elapsed_time_in_current = now - (current_window * self.window_size)
        previous_weight = 1 - (elapsed_time_in_current / self.window_size)
        
        current_key = f"rate_limit:{client_id}:{current_window}"
        previous_key = f"rate_limit:{client_id}:{previous_window}"
        
        current_count = int(self.redis.get(current_key) or 0)
        previous_count = int(self.redis.get(previous_key) or 0)
        
        weighted_count = (previous_count * previous_weight) + current_count
        
        return max(0, limit - math.ceil(weighted_count))

# Usage in Flask
from flask import Flask, request, jsonify

app = Flask(__name__)
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
limiter = SlidingWindowCounter(redis_client, window_size_seconds=60)

@app.before_request
def rate_limit():
    client_id = request.headers.get('X-API-Key') or request.remote_addr
    limit = 100  # 100 requests per minute
    
    if not limiter.is_allowed(client_id, limit):
        response = jsonify({
            'error': 'Rate limit exceeded',
            'limit': limit,
            'window': '60s'
        })
        response.status_code = 429
        response.headers['X-RateLimit-Limit'] = str(limit)
        response.headers['X-RateLimit-Remaining'] = '0'
        response.headers['Retry-After'] = '60'
        return response
    
    # Add rate limit headers to response
    @app.after_request
    def add_rate_limit_headers(response):
        remaining = limiter.get_remaining(client_id, limit)
        response.headers['X-RateLimit-Limit'] = str(limit)
        response.headers['X-RateLimit-Remaining'] = str(remaining)
        return response
```

## Distributed Rate Limiting

Single-server rate limiting doesn't work with multiple application servers. Need distributed coordination to enforce global rate limits.

### Redis-Based Rate Limiting

Use Redis as centralized store for rate limit counters. All application servers check/update Redis. Redis atomic operations ensure accuracy.

```javascript
// Node.js: Distributed rate limiting with Redis
const Redis = require('ioredis');

class DistributedRateLimiter {
    constructor(redisClient) {
        this.redis = redisClient;
    }
    
    async tokenBucket(clientId, capacity, refillRate) {
        const key = `rate_limit:token_bucket:${clientId}`;
        const now = Date.now();
        
        // Lua script for atomic token bucket operation
        const script = `
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1]) or capacity
            local last_refill = tonumber(bucket[2]) or now
            
            -- Refill tokens
            local elapsed = (now - last_refill) / 1000
            local tokens_to_add = elapsed * refill_rate
            tokens = math.min(capacity, tokens + tokens_to_add)
            
            -- Try to consume token
            if tokens >= 1 then
                tokens = tokens - 1
                redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
                redis.call('EXPIRE', key, 3600)
                return 1
            else
                return 0
            end
        `;
        
        const result = await this.redis.eval(
            script,
            1,
            key,
            capacity,
            refillRate,
            now
        );
        
        return result === 1;
    }
    
    async slidingWindowCounter(clientId, limit, windowSeconds) {
        const now = Math.floor(Date.now() / 1000);
        const currentWindow = Math.floor(now / windowSeconds);
        const previousWindow = currentWindow - 1;
        
        const currentKey = `rate_limit:window:${clientId}:${currentWindow}`;
        const previousKey = `rate_limit:window:${clientId}:${previousWindow}`;
        
        // Lua script for atomic sliding window
        const script = `
            local current_key = KEYS[1]
            local previous_key = KEYS[2]
            local limit = tonumber(ARGV[1])
            local window_size = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local current_window = tonumber(ARGV[4])
            
            local current_count = tonumber(redis.call('GET', current_key) or 0)
            local previous_count = tonumber(redis.call('GET', previous_key) or 0)
            
            -- Calculate position in current window
            local elapsed = now - (current_window * window_size)
            local previous_weight = 1 - (elapsed / window_size)
            
            -- Calculate weighted count
            local weighted_count = (previous_count * previous_weight) + current_count
            
            if weighted_count < limit then
                redis.call('INCR', current_key)
                redis.call('EXPIRE', current_key, window_size * 2)
                return 1
            else
                return 0
            end
        `;
        
        const result = await this.redis.eval(
            script,
            2,
            currentKey,
            previousKey,
            limit,
            windowSeconds,
            now,
            currentWindow
        );
        
        return result === 1;
    }
}

// Usage with Express middleware
const express = require('express');
const app = express();

const redis = new Redis({
    host: 'localhost',
    port: 6379
});

const limiter = new DistributedRateLimiter(redis);

app.use(async (req, res, next) => {
    const clientId = req.headers['x-api-key'] || req.ip;
    
    const allowed = await limiter.slidingWindowCounter(
        clientId,
        100,  // 100 requests
        60    // per 60 seconds
    );
    
    if (!allowed) {
        return res.status(429).json({
            error: 'Rate limit exceeded'
        });
    }
    
    next();
});
```

## Rate Limiting Strategies

### Per-User Rate Limiting

Limit requests per user/API key. Prevents single user from monopolizing resources.

### Per-IP Rate Limiting

Limit requests per IP address. Protects against DDoS and scraping. Less accurate than per-user (shared IPs, VPNs).

### Tiered Rate Limiting

Different limits for different user tiers. Free tier: 100 requests/hour. Pro tier: 1000 requests/hour. Enterprise: unlimited.

### Endpoint-Specific Limits

Different limits for different endpoints. Expensive operations have lower limits. Read operations have higher limits than writes.

## Best Practices

✅ **Return Clear Error Messages**: Include limit, remaining, and reset time in 429 responses.

✅ **Add Rate Limit Headers**: Include X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset headers.

✅ **Use Retry-After Header**: Tell clients when to retry with Retry-After header.

✅ **Implement Graceful Degradation**: Instead of hard blocking, slow down requests gradually.

✅ **Monitor Rate Limit Hits**: Track how often users hit limits. Adjust if too restrictive or too lenient.

✅ **Use Distributed Coordination**: For multi-server deployments, use Redis or similar for global limits.

✅ **Document Limits**: Clearly document rate limits in API documentation.

✅ **Provide Burst Allowance**: Allow short bursts with token bucket algorithm.

## Anti-Patterns

❌ **No Rate Limiting**: Leaving APIs unprotected invites abuse and overload.

❌ **Fixed Window Only**: Allows burst at window boundaries. Use sliding window instead.

❌ **Per-Server Limits**: With multiple servers, per-server limits don't protect overall capacity.

❌ **No User Feedback**: Don't silently reject. Provide clear error messages and retry guidance.

❌ **Same Limit for All Endpoints**: Different endpoints have different costs. Use endpoint-specific limits.

❌ **Blocking Legitimate Traffic**: Too restrictive limits hurt legitimate users. Monitor and adjust.
