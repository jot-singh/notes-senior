# API Gateway Pattern

## What You'll Learn

API Gateway is a server that acts as the single entry point for all client requests in a microservices architecture. It handles request routing, composition, protocol translation, authentication, rate limiting, and monitoring. This note covers API Gateway patterns, implementations (Kong, AWS API Gateway, NGINX), and production considerations.

## Why This Matters

In microservices architectures, clients would need to know about dozens or hundreds of service endpoints. Each service might use different protocols (HTTP, gRPC, WebSocket). Cross-cutting concerns like authentication would be duplicated across services. API Gateway solves these problems by providing a unified interface. Netflix's Zuul handles billions of requests daily. Amazon API Gateway processes trillions of requests. Understanding API Gateway is essential for microservices design.

## Core Responsibilities

### Request Routing

API Gateway routes requests to appropriate backend services based on URL paths, headers, or other criteria. A request to `/users/*` routes to the user service, `/orders/*` to the order service.

```javascript
// Node.js: Simple API Gateway routing
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

// Route to user service
app.use('/api/users', createProxyMiddleware({
    target: 'http://user-service:3001',
    changeOrigin: true,
    pathRewrite: {
        '^/api/users': '/users'
    },
    onProxyReq: (proxyReq, req, res) => {
        // Add tracing headers
        proxyReq.setHeader('X-Request-ID', req.headers['x-request-id'] || generateId());
        proxyReq.setHeader('X-Forwarded-For', req.ip);
    },
    onError: (err, req, res) => {
        console.error('Proxy error:', err);
        res.status(503).json({ error: 'Service unavailable' });
    }
}));

// Route to order service
app.use('/api/orders', createProxyMiddleware({
    target: 'http://order-service:3002',
    changeOrigin: true,
    pathRewrite: {
        '^/api/orders': '/orders'
    }
}));

// Route to product service with load balancing
const productServers = [
    'http://product-service-1:3003',
    'http://product-service-2:3003',
    'http://product-service-3:3003'
];

let currentServer = 0;

app.use('/api/products', createProxyMiddleware({
    target: () => {
        // Round-robin load balancing
        const server = productServers[currentServer];
        currentServer = (currentServer + 1) % productServers.length;
        return server;
    },
    changeOrigin: true,
    pathRewrite: {
        '^/api/products': '/products'
    }
}));

app.listen(8080, () => {
    console.log('API Gateway listening on port 8080');
});
```

### Request Aggregation (Backend for Frontend)

API Gateway can aggregate multiple backend service calls into a single response, reducing client round-trips.

```java
// Java Spring: Request aggregation
@RestController
@RequestMapping("/api/gateway")
public class AggregationGateway {
    
    @Autowired
    private WebClient webClient;
    
    @GetMapping("/user/{userId}/dashboard")
    public Mono<UserDashboard> getUserDashboard(@PathVariable String userId) {
        // Call multiple services in parallel
        Mono<User> userMono = webClient.get()
            .uri("http://user-service/users/{id}", userId)
            .retrieve()
            .bodyToMono(User.class);
        
        Mono<List<Order>> ordersMono = webClient.get()
            .uri("http://order-service/users/{id}/orders", userId)
            .retrieve()
            .bodyToFlux(Order.class)
            .collectList();
        
        Mono<List<Notification>> notificationsMono = webClient.get()
            .uri("http://notification-service/users/{id}/notifications", userId)
            .retrieve()
            .bodyToFlux(Notification.class)
            .collectList();
        
        // Combine results
        return Mono.zip(userMono, ordersMono, notificationsMono)
            .map(tuple -> {
                User user = tuple.getT1();
                List<Order> orders = tuple.getT2();
                List<Notification> notifications = tuple.getT3();
                
                return new UserDashboard(user, orders, notifications);
            })
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(e -> {
                // Fallback response on error
                return Mono.just(new UserDashboard());
            });
    }
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class UserDashboard {
        private User user;
        private List<Order> recentOrders;
        private List<Notification> notifications;
    }
}
```

### Authentication and Authorization

API Gateway centralizes authentication, validating tokens and enforcing access policies before routing to backend services.

```python
# Python FastAPI: JWT authentication at gateway
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
import httpx
from datetime import datetime, timedelta

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

class AuthenticationMiddleware:
    def __init__(self, app):
        self.app = app
    
    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            request = Request(scope, receive)
            
            # Skip auth for public endpoints
            if request.url.path in ["/health", "/login", "/register"]:
                await self.app(scope, receive, send)
                return
            
            # Verify JWT token
            auth_header = request.headers.get("Authorization")
            if not auth_header or not auth_header.startswith("Bearer "):
                response = {"error": "Missing or invalid authorization header"}
                await send_json_response(send, 401, response)
                return
            
            token = auth_header.split(" ")[1]
            
            try:
                payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
                user_id = payload.get("user_id")
                
                # Add user context to request
                scope["user_id"] = user_id
                scope["user_role"] = payload.get("role")
                
            except jwt.ExpiredSignatureError:
                await send_json_response(send, 401, {"error": "Token expired"})
                return
            except jwt.InvalidTokenError:
                await send_json_response(send, 401, {"error": "Invalid token"})
                return
        
        await self.app(scope, receive, send)

async def send_json_response(send, status_code, body):
    import json
    await send({
        'type': 'http.response.start',
        'status': status_code,
        'headers': [[b'content-type', b'application/json']],
    })
    await send({
        'type': 'http.response.body',
        'body': json.dumps(body).encode('utf-8'),
    })

# Protected route with role-based authorization
@app.get("/api/admin/users")
async def get_all_users(
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=[ALGORITHM]
        )
        
        if payload.get("role") != "admin":
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        
        # Proxy to user service
        async with httpx.AsyncClient() as client:
            response = await client.get("http://user-service:3000/users")
            return response.json()
            
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# API key authentication
API_KEYS = {
    "key123": {"client": "mobile-app", "rate_limit": 1000},
    "key456": {"client": "web-app", "rate_limit": 5000}
}

@app.middleware("http")
async def api_key_middleware(request: Request, call_next):
    if request.url.path.startswith("/api/public"):
        api_key = request.headers.get("X-API-Key")
        
        if not api_key or api_key not in API_KEYS:
            return JSONResponse(
                status_code=401,
                content={"error": "Invalid API key"}
            )
        
        # Add client context
        request.state.client = API_KEYS[api_key]["client"]
        request.state.rate_limit = API_KEYS[api_key]["rate_limit"]
    
    response = await call_next(request)
    return response
```

### Protocol Translation

API Gateway can translate between different protocols (HTTP to gRPC, REST to SOAP, HTTP to WebSocket).

```go
// Go: HTTP to gRPC translation
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    
    "google.golang.org/grpc"
    pb "your-package/proto"
)

type GRPCGateway struct {
    userClient pb.UserServiceClient
    orderClient pb.OrderServiceClient
}

func NewGRPCGateway() *GRPCGateway {
    // Connect to gRPC services
    userConn, err := grpc.Dial(
        "user-service:50051",
        grpc.WithInsecure(),
    )
    if err != nil {
        log.Fatal(err)
    }
    
    orderConn, err := grpc.Dial(
        "order-service:50052",
        grpc.WithInsecure(),
    )
    if err != nil {
        log.Fatal(err)
    }
    
    return &GRPCGateway{
        userClient: pb.NewUserServiceClient(userConn),
        orderClient: pb.NewOrderServiceClient(orderConn),
    }
}

// HTTP endpoint that calls gRPC backend
func (g *GRPCGateway) GetUser(w http.ResponseWriter, r *http.Request) {
    userId := r.URL.Query().Get("id")
    
    // Call gRPC service
    ctx := context.Background()
    resp, err := g.userClient.GetUser(ctx, &pb.GetUserRequest{
        UserId: userId,
    })
    
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Convert gRPC response to JSON
    user := map[string]interface{}{
        "id": resp.Id,
        "name": resp.Name,
        "email": resp.Email,
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (g *GRPCGateway) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var orderReq struct {
        UserId string `json:"user_id"`
        Items []struct {
            ProductId string `json:"product_id"`
            Quantity int32 `json:"quantity"`
        } `json:"items"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&orderReq); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Convert to gRPC request
    items := make([]*pb.OrderItem, len(orderReq.Items))
    for i, item := range orderReq.Items {
        items[i] = &pb.OrderItem{
            ProductId: item.ProductId,
            Quantity: item.Quantity,
        }
    }
    
    ctx := context.Background()
    resp, err := g.orderClient.CreateOrder(ctx, &pb.CreateOrderRequest{
        UserId: orderReq.UserId,
        Items: items,
    })
    
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Return JSON response
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "order_id": resp.OrderId,
        "status": resp.Status,
    })
}
```

### Rate Limiting and Throttling

API Gateway enforces rate limits to protect backend services.

```typescript
// TypeScript: Rate limiting middleware
import express from 'express';
import Redis from 'ioredis';

const redis = new Redis({
    host: 'localhost',
    port: 6379
});

interface RateLimitConfig {
    windowSeconds: number;
    maxRequests: number;
}

// Different limits for different tiers
const RATE_LIMITS: Record<string, RateLimitConfig> = {
    'free': { windowSeconds: 3600, maxRequests: 100 },
    'pro': { windowSeconds: 3600, maxRequests: 1000 },
    'enterprise': { windowSeconds: 3600, maxRequests: 10000 }
};

async function rateLimitMiddleware(
    req: express.Request,
    res: express.Response,
    next: express.NextFunction
) {
    const apiKey = req.headers['x-api-key'] as string;
    
    if (!apiKey) {
        return res.status(401).json({ error: 'API key required' });
    }
    
    // Get user tier (from database or cache)
    const userTier = await getUserTier(apiKey);
    const config = RATE_LIMITS[userTier] || RATE_LIMITS['free'];
    
    const key = `rate_limit:${apiKey}:${Math.floor(Date.now() / 1000 / config.windowSeconds)}`;
    
    // Increment request count
    const count = await redis.incr(key);
    
    if (count === 1) {
        await redis.expire(key, config.windowSeconds);
    }
    
    // Set rate limit headers
    res.setHeader('X-RateLimit-Limit', config.maxRequests.toString());
    res.setHeader('X-RateLimit-Remaining', Math.max(0, config.maxRequests - count).toString());
    res.setHeader('X-RateLimit-Reset', 
        ((Math.floor(Date.now() / 1000 / config.windowSeconds) + 1) * config.windowSeconds).toString()
    );
    
    if (count > config.maxRequests) {
        return res.status(429).json({
            error: 'Rate limit exceeded',
            retryAfter: config.windowSeconds
        });
    }
    
    next();
}

async function getUserTier(apiKey: string): Promise<string> {
    // Check cache first
    const cached = await redis.get(`user_tier:${apiKey}`);
    if (cached) return cached;
    
    // Fetch from database
    // const tier = await db.query('SELECT tier FROM users WHERE api_key = ?', [apiKey]);
    
    // Cache for 1 hour
    // await redis.setex(`user_tier:${apiKey}`, 3600, tier);
    
    return 'free';
}
```

### Response Caching

API Gateway can cache responses to reduce backend load and improve latency.

```python
# Python: Response caching at gateway
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse
import redis
import hashlib
import json
from datetime import timedelta

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def generate_cache_key(request: Request) -> str:
    """Generate cache key from request"""
    key_parts = [
        request.method,
        str(request.url),
        request.headers.get("Authorization", ""),
    ]
    key_string = ":".join(key_parts)
    return f"cache:{hashlib.md5(key_string.encode()).hexdigest()}"

@app.middleware("http")
async def cache_middleware(request: Request, call_next):
    # Only cache GET requests
    if request.method != "GET":
        return await call_next(request)
    
    # Check if cacheable endpoint
    cacheable_paths = ["/api/products", "/api/categories", "/api/public"]
    if not any(request.url.path.startswith(path) for path in cacheable_paths):
        return await call_next(request)
    
    # Check cache
    cache_key = generate_cache_key(request)
    cached_response = redis_client.get(cache_key)
    
    if cached_response:
        data = json.loads(cached_response)
        return JSONResponse(
            content=data["body"],
            status_code=data["status_code"],
            headers={
                **data["headers"],
                "X-Cache": "HIT"
            }
        )
    
    # Cache miss - call backend
    response = await call_next(request)
    
    # Cache successful responses
    if 200 <= response.status_code < 300:
        body = b""
        async for chunk in response.body_iterator:
            body += chunk
        
        cache_data = {
            "body": json.loads(body.decode()),
            "status_code": response.status_code,
            "headers": dict(response.headers)
        }
        
        # Cache for 5 minutes
        redis_client.setex(
            cache_key,
            300,
            json.dumps(cache_data)
        )
        
        return JSONResponse(
            content=cache_data["body"],
            status_code=response.status_code,
            headers={
                **cache_data["headers"],
                "X-Cache": "MISS"
            }
        )
    
    return response
```

## Circuit Breaker Pattern

API Gateway implements circuit breaker to prevent cascading failures when backend services are down.

```java
// Java: Circuit breaker with Resilience4j
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.time.Duration;

@Component
public class ResilientApiGateway {
    
    private final WebClient webClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    public ResilientApiGateway() {
        this.webClient = WebClient.builder().build();
        
        // Configure circuit breaker
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .slidingWindowSize(10)
            .failureRateThreshold(50)  // Open if 50% fail
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .slowCallRateThreshold(50)
            .build();
        
        this.circuitBreakerRegistry = CircuitBreakerRegistry.of(config);
    }
    
    public Mono<String> callUserService(String userId) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("user-service");
        
        return Mono.defer(() -> webClient.get()
            .uri("http://user-service/users/{id}", userId)
            .retrieve()
            .bodyToMono(String.class))
            .transform(CircuitBreakerOperator.of(circuitBreaker))
            .onErrorResume(e -> {
                // Fallback response
                return Mono.just("{\"error\": \"User service unavailable\"}");
            });
    }
    
    public Mono<String> callOrderService(String orderId) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("order-service");
        
        return Mono.defer(() -> webClient.get()
            .uri("http://order-service/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(String.class))
            .transform(CircuitBreakerOperator.of(circuitBreaker))
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(e -> {
                // Check circuit state
                CircuitBreaker.State state = circuitBreaker.getState();
                
                if (state == CircuitBreaker.State.OPEN) {
                    return Mono.just("{\"error\": \"Order service circuit open\"}");
                }
                
                return Mono.just("{\"error\": \"Order service error\"}");
            });
    }
}
```

## API Gateway Implementations

### Kong API Gateway

Kong is an open-source API Gateway built on NGINX. It provides plugins for authentication, rate limiting, logging, and more.

```yaml
# Kong configuration
services:
  - name: user-service
    url: http://user-service:3000
    routes:
      - name: user-route
        paths:
          - /api/users
        methods:
          - GET
          - POST
        plugins:
          - name: rate-limiting
            config:
              minute: 100
              policy: local
          - name: jwt
            config:
              key_claim_name: kid
              secret_is_base64: false
          - name: cors
            config:
              origins:
                - "*"
              methods:
                - GET
                - POST
              headers:
                - Authorization
                - Content-Type

  - name: order-service
    url: http://order-service:3001
    routes:
      - name: order-route
        paths:
          - /api/orders
        plugins:
          - name: request-transformer
            config:
              add:
                headers:
                  - X-Service: order-service
          - name: response-transformer
            config:
              add:
                headers:
                  - X-Gateway: kong

plugins:
  - name: prometheus
    config:
      per_consumer: true
```

### AWS API Gateway

AWS API Gateway is a fully managed service for creating, publishing, and managing APIs.

```javascript
// AWS CDK: API Gateway configuration
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Construct } from 'constructs';

export class ApiGatewayStack extends Construct {
    constructor(scope: Construct, id: string) {
        super(scope, id);
        
        // Create Lambda functions
        const userFunction = new lambda.Function(this, 'UserFunction', {
            runtime: lambda.Runtime.NODEJS_18_X,
            handler: 'index.handler',
            code: lambda.Code.fromAsset('lambda/user')
        });
        
        const orderFunction = new lambda.Function(this, 'OrderFunction', {
            runtime: lambda.Runtime.NODEJS_18_X,
            handler: 'index.handler',
            code: lambda.Code.fromAsset('lambda/order')
        });
        
        // Create API Gateway
        const api = new apigateway.RestApi(this, 'MyApi', {
            restApiName: 'My Service API',
            description: 'API Gateway for microservices',
            deployOptions: {
                stageName: 'prod',
                throttlingBurstLimit: 5000,
                throttlingRateLimit: 10000,
                loggingLevel: apigateway.MethodLoggingLevel.INFO,
                dataTraceEnabled: true,
                metricsEnabled: true
            }
        });
        
        // Add usage plan and API key
        const usagePlan = api.addUsagePlan('UsagePlan', {
            name: 'Standard',
            throttle: {
                rateLimit: 1000,
                burstLimit: 2000
            },
            quota: {
                limit: 10000,
                period: apigateway.Period.DAY
            }
        });
        
        const apiKey = api.addApiKey('ApiKey');
        usagePlan.addApiKey(apiKey);
        
        // User resource
        const users = api.root.addResource('users');
        users.addMethod('GET', new apigateway.LambdaIntegration(userFunction), {
            authorizationType: apigateway.AuthorizationType.IAM,
            apiKeyRequired: true,
            requestParameters: {
                'method.request.querystring.id': true
            },
            requestValidator: new apigateway.RequestValidator(this, 'RequestValidator', {
                restApi: api,
                validateRequestParameters: true
            })
        });
        
        // Order resource with caching
        const orders = api.root.addResource('orders');
        orders.addMethod('GET', new apigateway.LambdaIntegration(orderFunction), {
            authorizationType: apigateway.AuthorizationType.IAM,
            requestParameters: {
                'method.request.header.Authorization': true
            },
            methodResponses: [{
                statusCode: '200',
                responseParameters: {
                    'method.response.header.Content-Type': true
                }
            }],
            cacheKeyParameters: ['method.request.querystring.id'],
            cachingEnabled: true,
            cacheTtl: 300  // 5 minutes
        });
        
        // Add CORS
        users.addCorsPreflight({
            allowOrigins: ['*'],
            allowMethods: ['GET', 'POST', 'OPTIONS'],
            allowHeaders: ['Content-Type', 'Authorization']
        });
    }
}
```

## Best Practices

✅ **Single Entry Point**: All external traffic goes through API Gateway, never directly to services.

✅ **Timeout Configuration**: Set appropriate timeouts for backend calls. Don't let slow services block the gateway.

✅ **Circuit Breakers**: Implement circuit breakers for each backend service to prevent cascading failures.

✅ **Response Caching**: Cache responses for frequently accessed, slowly changing data.

✅ **Request Validation**: Validate requests at the gateway to reduce invalid traffic to backend services.

✅ **Centralized Logging**: Log all requests through gateway for monitoring and debugging.

✅ **Health Checks**: Implement health check endpoints and monitor gateway health.

✅ **Versioning**: Support API versioning (URL path, header, or query parameter).

✅ **Security**: Implement authentication, authorization, and encryption at the gateway.

✅ **Rate Limiting**: Protect backend services with rate limiting at the gateway.

## Anti-Patterns

❌ **Business Logic in Gateway**: Keep gateway focused on routing and cross-cutting concerns. Business logic belongs in services.

❌ **Single Point of Failure**: Deploy multiple gateway instances with load balancing. Don't make gateway a bottleneck.

❌ **No Fallbacks**: Always provide fallback responses when backend services fail.

❌ **Chatty Aggregation**: Don't make too many backend calls for single request. Can timeout and cause latency.

❌ **Ignoring Monitoring**: Gateway is critical infrastructure. Monitor performance, errors, and latency metrics.

❌ **Hard-Coded Service URLs**: Use service discovery instead of hard-coding backend URLs.
