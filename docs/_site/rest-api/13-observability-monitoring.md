# Observability & Monitoring

## What You'll Learn
How to make your API observable through metrics, logs, and traces. You'll understand the three pillars of observability, how to implement distributed tracing, define SLOs, and build effective monitoring systems that detect issues before users notice.

## Why This Matters
You can't fix what you can't see. In distributed systems, failures are subtle: slow queries, memory leaks, cascading timeouts. Without observability, you're debugging blind, relying on user complaints. Good observability means detecting 95th percentile latency spikes before they become outages, finding the one slow service in a 20-service call chain, and understanding user impact. Observability is essential for reliability, performance, and incident response.

## The Three Pillars

### 1. Metrics (Aggregated Numbers)

**What**: Time-series data; counts, rates, distributions.
**When**: Real-time dashboards, alerting, capacity planning.
**Tools**: Prometheus, Datadog, CloudWatch.

**Example Metrics**:
```
http_requests_total{method="POST", endpoint="/orders", status="201"} 1543
http_request_duration_seconds{method="POST", endpoint="/orders", quantile="0.95"} 0.245
http_requests_in_flight{endpoint="/orders"} 23
api_errors_total{type="validation", endpoint="/orders"} 45
```

### 2. Logs (Discrete Events)

**What**: Text records of events; structured or unstructured.
**When**: Debugging, audit trails, error analysis.
**Tools**: ELK Stack, Splunk, Loki.

**Example Log**:
```json
{
  "timestamp": "2026-01-11T10:30:15.123Z",
  "level": "INFO",
  "service": "order-service",
  "request_id": "req-abc123",
  "trace_id": "trace-xyz789",
  "user_id": "user-456",
  "method": "POST",
  "path": "/orders",
  "status": 201,
  "duration_ms": 245,
  "message": "Order created successfully",
  "order_id": "ord-789"
}
```

### 3. Traces (Request Journeys)

**What**: End-to-end request flow through services.
**When**: Understanding dependencies, finding bottlenecks.
**Tools**: Jaeger, Zipkin, OpenTelemetry.

**Example Trace**:
```
Trace ID: trace-xyz789

├─ POST /orders [API Gateway] 450ms
   ├─ validateToken [Auth Service] 25ms
   ├─ createOrder [Order Service] 380ms
      ├─ checkInventory [Inventory Service] 120ms
      │  └─ queryDB [PostgreSQL] 95ms
      ├─ reserveInventory [Inventory Service] 80ms
      ├─ calculatePrice [Pricing Service] 45ms
      └─ saveOrder [PostgreSQL] 110ms
   └─ publishEvent [Kafka] 15ms
```

## Key Metrics for APIs

### Golden Signals (Google SRE)

**1. Latency**: How long requests take
```
http_request_duration_seconds{quantile="0.5"}  # p50 (median)
http_request_duration_seconds{quantile="0.95"}  # p95
http_request_duration_seconds{quantile="0.99"}  # p99
```

**2. Traffic**: How many requests
```
rate(http_requests_total[1m])  # requests per second
```

**3. Errors**: How many failures
```
rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])
```

**4. Saturation**: How full is the system
```
process_cpu_usage
process_memory_usage_bytes / process_memory_max_bytes
jvm_threads_current / jvm_threads_max
db_connections_active / db_connections_max
```

### RED Method (Request-centric)

- **Rate**: Requests per second
- **Errors**: Error rate (count or %)
- **Duration**: Latency distribution

**Prometheus Query**:
```promql
# Rate (req/s per endpoint)
sum(rate(http_requests_total[5m])) by (endpoint, method)

# Error rate
sum(rate(http_requests_total{status=~"[45].*"}[5m])) by (endpoint)
  / sum(rate(http_requests_total[5m])) by (endpoint)

# Duration (p95 latency)
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (endpoint, le)
)
```

### USE Method (Resource-centric)

- **Utilization**: % of resource used
- **Saturation**: Queue depth, backlog
- **Errors**: Error count

For: CPU, memory, disk, network, DB connections.

## Distributed Tracing

### W3C Trace Context

**Headers**:
```http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
trace state: congo=t61rcWkgMzE
```

**Format**: `version-trace_id-parent_id-trace_flags`
- `00`: Version
- `0af7...319c`: Trace ID (128-bit)
- `b7ad...3331`: Span ID (64-bit)
- `01`: Sampled flag

### Implementation (Spring Boot + OpenTelemetry)

**1. Add Dependencies**:
```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

**2. Configuration**:
```yaml
otel:
  service:
    name: order-service
  exporter:
    otlp:
      endpoint: http://jaeger:4317
  traces:
    sampler: probability
    sampler-arg: 0.1  # Sample 10% of traces
```

**3. Automatic Instrumentation**:
```java
// HTTP requests automatically traced
@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        // Span created automatically
        return orderService.getOrder(id);
    }
}
```

**4. Manual Spans**:
```java
@Service
public class OrderService {
    
    @Autowired
    private Tracer tracer;
    
    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("createOrder")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("order.customer_id", request.getCustomerId());
            span.setAttribute("order.item_count", request.getItems().size());
            
            // Business logic
            Order order = processOrder(request);
            
            span.setAttribute("order.id", order.getId());
            span.setStatus(StatusCode.OK);
            return order;
            
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**5. Propagate Context**:
```java
@Service
public class InventoryClient {
    
    public boolean checkInventory(String productId, int quantity) {
        // Trace context automatically propagated in headers
        return restTemplate.getForObject(
            "http://inventory-service/check?productId={productId}&quantity={quantity}",
            Boolean.class,
            productId,
            quantity
        );
    }
}
```

### Trace Sampling

**Why**: Tracing every request is expensive at scale.

**Strategies**:

1. **Probabilistic**: Sample X% of traces
   ```
   Sample 1% of requests
   At 10,000 req/s = 100 traces/s
   ```

2. **Rate Limiting**: Max N traces per second
   ```
   Max 100 traces/s regardless of traffic
   ```

3. **Adaptive**: Increase sampling for errors
   ```
   Success: 1% sampling
   4xx errors: 10% sampling
   5xx errors: 100% sampling (always trace)
   Slow requests (>1s): 100% sampling
   ```

4. **Debug Mode**: Force sampling via header
   ```http
   X-Force-Trace: 1
   ```

## Structured Logging

### Log Format

```java
@Slf4j
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(
        @RequestBody CreateOrderRequest request,
        @RequestHeader("X-Request-ID") String requestId
    ) {
        MDC.put("request_id", requestId);
        MDC.put("customer_id", request.getCustomerId());
        
        log.info("Creating order", 
            kv("item_count", request.getItems().size()),
            kv("total_amount", request.getTotal()));
        
        try {
            Order order = orderService.createOrder(request);
            
            log.info("Order created successfully",
                kv("order_id", order.getId()),
                kv("status", order.getStatus()));
            
            return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(order);
                
        } catch (ValidationException e) {
            log.warn("Order validation failed",
                kv("errors", e.getErrors()));
            throw e;
        } catch (Exception e) {
            log.error("Order creation failed", e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

**Output (JSON)**:
```json
{
  "timestamp": "2026-01-11T10:30:15.123Z",
  "level": "INFO",
  "logger": "com.example.OrderController",
  "message": "Order created successfully",
  "request_id": "req-abc123",
  "trace_id": "trace-xyz789",
  "customer_id": "cust-456",
  "order_id": "ord-789",
  "status": "pending",
  "thread": "http-nio-8080-exec-5"
}
```

### What to Log

**DO Log**:
- Request ID, trace ID
- User/tenant ID (not PII)
- Resource IDs (order ID, product ID)
- Operation outcomes (created, updated, failed)
- Error details (sanitized)
- Performance (duration, size)

**DON'T Log**:
- Passwords, tokens, API keys
- Credit card numbers, SSNs
- Full request/response bodies (too large)
- High-frequency events (every DB query)

### Log Levels

- **ERROR**: Requires immediate attention
- **WARN**: Unexpected but handled
- **INFO**: Normal operations (create, update, delete)
- **DEBUG**: Detailed flow (disabled in prod)
- **TRACE**: Very detailed (never in prod)

## SLIs, SLOs, and SLAs

### Service Level Indicator (SLI)

**What**: Quantitative measure of service level.

**Examples**:
- Availability: % of successful requests
- Latency: % of requests < 500ms
- Throughput: requests per second
- Correctness: % of correct responses

### Service Level Objective (SLO)

**What**: Target for SLI; internal goal.

**Examples**:
```
Availability SLO: 99.9% of requests succeed
  = 43 minutes downtime per month

Latency SLO: 95% of requests < 200ms
            99% of requests < 500ms

Error Budget: 0.1% (1 - 99.9%)
  = 43,800 failed requests per month (at 1M req/day)
```

### Service Level Agreement (SLA)

**What**: Contractual commitment to customers.

**Example**:
```
SLA: 99.95% uptime
  If < 99.95%: 10% credit
  If < 99.00%: 25% credit
```

**Relationship**: SLA ≤ SLO (set SLO higher to avoid SLA violations)

### Error Budget

**Concept**: Allowed failures before violating SLO.

**Calculation**:
```
SLO: 99.9% success
Error budget: 0.1% failures

At 1,000,000 requests/month:
Allowed failures: 1,000 requests

Current month:
- Day 15: 400 failures used (40% of budget)
- Pace: 800 failures/month (within budget ✓)
```

**Policy**:
```
If error budget > 0:
  ✓ Deploy new features
  ✓ Take risks
  ✓ Optimize for velocity

If error budget exhausted:
  ❌ Freeze feature deployments
  ✓ Focus on reliability
  ✓ Pay down tech debt
```

## Alerting

### Alert Design

**Good Alert**:
```yaml
alert: HighErrorRate
expr: |
  sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))
    > 0.05
for: 5m
labels:
  severity: critical
annotations:
  summary: "Error rate above 5% for 5 minutes"
  description: "Current error rate: {{ $value }}%"
  runbook: "https://wiki.example.com/runbooks/high-error-rate"
```

**Principles**:
1. **Actionable**: Someone can fix it
2. **Urgent**: Requires immediate attention
3. **Real impact**: Users affected
4. **Specific**: Clear what's wrong
5. **Includes runbook**: How to fix

### Alert Fatigue Prevention

**Anti-patterns**:
- Alert on everything ❌
- No context ❌
- False positives ❌
- Noisy aggregations ❌

**Solutions**:
- Alert on SLO violations, not symptoms
- Include dashboards and traces in alerts
- Tune thresholds based on historical data
- Aggregate similar alerts
- Escalation policies

## Dashboards

### API Overview Dashboard

```
┏━━━━━━━━━━━━━━━━━━━━┓  ┏━━━━━━━━━━━━━━━━━━━━┓
┃ Request Rate      ┃  ┃ Error Rate        ┃
┃ 1,245 req/s        ┃  ┃ 0.12% (⇑ 0.03%)  ┃
┗━━━━━━━━━━━━━━━━━━━━┛  ┗━━━━━━━━━━━━━━━━━━━━┛

┏━━━━━━━━━━━━━━━━━━━━┓  ┏━━━━━━━━━━━━━━━━━━━━┓
┃ p50 Latency       ┃  ┃ p95 Latency       ┃
┃ 45ms              ┃  ┃ 180ms             ┃
┗━━━━━━━━━━━━━━━━━━━━┛  ┗━━━━━━━━━━━━━━━━━━━━┛

Latency Heatmap (last hour):
[Visualization showing latency distribution over time]

Top Endpoints by Request Count:
1. GET  /orders          45% (560 req/s)
2. POST /orders          25% (311 req/s)
3. GET  /products        15% (187 req/s)

Error Breakdown by Status:
500: 8 req/s  (■■■■■■■■■■■■■■■■■■■■ 67%)
422: 3 req/s  (■■■■■■■■ 25%)
404: 1 req/s  (■■ 8%)
```

## Tooling

### Stack Example

- **Metrics**: Prometheus + Grafana
- **Logs**: Fluentd → Elasticsearch → Kibana
- **Traces**: OpenTelemetry → Jaeger
- **Alerts**: Prometheus Alertmanager → PagerDuty
- **APM**: Datadog / New Relic (all-in-one)

### Cost Considerations

**Strategies**:
- Sample high-volume traces (1-10%)
- Retain logs for 7-30 days
- Aggregate old metrics (1h → 5m → 1m resolution)
- Index only searchable fields
- Separate hot/cold storage
