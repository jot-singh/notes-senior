# Monitoring and Observability

## What You'll Learn

- Difference between monitoring and observability
- The three pillars: metrics, logs, and traces
- Implementing distributed tracing
- Alerting strategies and SLI/SLO/SLA
- Observability tools and patterns

## Why This Matters

You cannot improve what you cannot measure. In distributed systems spanning thousands of servers across multiple regions, understanding system behavior is critical. When outages happen, every minute costs money and damages reputation. Netflix loses $7.2M per hour of downtime, Amazon $13.5M. Effective observability lets you detect issues before customers notice, diagnose root causes quickly, and understand system behavior under load. Companies with mature observability practices reduce mean time to recovery (MTTR) from hours to minutes.

## Monitoring vs Observability

**Monitoring** answers known questions about system state. You define metrics and thresholds in advance. "Is CPU above 80%?" "Is error rate above 1%?"

**Observability** lets you ask arbitrary questions without predicting failure modes. You can explore system behavior and understand why failures happen.

```python
# Monitoring: Predefined question
if cpu_utilization > 80:
    alert("High CPU usage")

# Observability: Explore arbitrary questions
trace_id = "abc-123"
# Why did this specific request take 5 seconds?
# What other requests happened at the same time?
# Which service caused the latency?
logs = query_logs(trace_id=trace_id)
traces = query_traces(trace_id=trace_id)
metrics = query_metrics(time_range=logs[0].timestamp)
```

## The Three Pillars

### 1. Metrics

Numeric measurements collected over time. Aggregated data suitable for dashboards and alerts.

**Types of Metrics**:

**Counters**: Monotonically increasing values (requests served, errors encountered)

**Gauges**: Point-in-time values that can go up or down (CPU usage, memory, queue depth)

**Histograms**: Distribution of values (request latency percentiles)

**Timers**: Specialized histograms for measuring duration

```java
// Metrics with Micrometer
@RestController
public class ApiController {
    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final Timer requestTimer;
    private final Gauge activeConnections;
    
    public ApiController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // Counter: Total requests
        this.requestCounter = Counter.builder("api.requests.total")
            .tag("endpoint", "/api/users")
            .description("Total number of API requests")
            .register(meterRegistry);
        
        // Timer: Request latency
        this.requestTimer = Timer.builder("api.requests.duration")
            .tag("endpoint", "/api/users")
            .description("API request duration")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);
        
        // Gauge: Active connections
        this.activeConnections = Gauge.builder("api.connections.active", 
                                              connectionPool, 
                                              ConnectionPool::getActiveCount)
            .register(meterRegistry);
    }
    
    @GetMapping("/api/users")
    public ResponseEntity<List<User>> getUsers() {
        requestCounter.increment();
        
        return requestTimer.record(() -> {
            List<User> users = userService.findAll();
            return ResponseEntity.ok(users);
        });
    }
}
```

**Key Metrics to Track**:

```javascript
// Application metrics
const metrics = {
    // Request metrics
    requestRate: 'requests per second',
    errorRate: 'errors per second',
    latency: {
        p50: 'median response time',
        p95: '95th percentile response time',
        p99: '99th percentile response time'
    },
    
    // Resource metrics
    cpu: 'CPU utilization percentage',
    memory: 'Memory usage in bytes',
    diskIO: 'Disk I/O operations per second',
    networkIO: 'Network bytes in/out per second',
    
    // Business metrics
    activeUsers: 'Currently active users',
    revenue: 'Revenue per minute',
    conversionRate: 'Percentage of successful transactions',
    
    // Infrastructure metrics
    queueDepth: 'Messages waiting in queue',
    cacheHitRate: 'Cache hits / total requests',
    databaseConnections: 'Active database connections'
};
```

### 2. Logs

Discrete events with context. Essential for debugging and understanding specific requests.

**Structured Logging**:

```java
// Bad: Unstructured logging
log.info("User logged in: " + userId + " from IP " + ipAddress);

// Good: Structured logging
log.info("User login",
    kv("userId", userId),
    kv("ipAddress", ipAddress),
    kv("userAgent", userAgent),
    kv("loginMethod", "oauth"),
    kv("timestamp", Instant.now())
);
```

**Log Levels**:

```python
import logging

logger = logging.getLogger(__name__)

# ERROR: Something failed, requires immediate attention
logger.error("Failed to process payment", 
             extra={"user_id": user_id, "amount": amount, "error": str(e)})

# WARN: Something unexpected, but system still functioning
logger.warning("API rate limit approaching", 
               extra={"current_rate": rate, "limit": limit})

# INFO: Important business events
logger.info("Order placed", 
            extra={"order_id": order_id, "user_id": user_id, "total": total})

# DEBUG: Detailed information for debugging (disabled in production)
logger.debug("Cache miss", 
             extra={"key": cache_key, "ttl": ttl})
```

**Log Aggregation Pattern**:

```javascript
// Centralized logging with correlation IDs
class Logger {
    constructor(serviceName) {
        this.serviceName = serviceName;
    }
    
    info(message, context = {}) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            level: 'INFO',
            service: this.serviceName,
            message: message,
            traceId: context.traceId || this.getTraceId(),
            spanId: context.spanId || this.getSpanId(),
            userId: context.userId,
            ...context
        };
        
        // Send to centralized logging (ELK, Splunk, Datadog)
        this.sendToLogAggregator(logEntry);
    }
    
    getTraceId() {
        // Extract from request context or generate
        return RequestContext.getTraceId() || this.generateTraceId();
    }
    
    generateTraceId() {
        return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    }
}

// Usage
const logger = new Logger('user-service');
logger.info('User registration', {
    userId: '12345',
    email: 'user@example.com',
    source: 'web'
});
```

### 3. Distributed Tracing

Tracks requests across multiple services. Critical for understanding latency and dependencies in microservices.

```java
// OpenTelemetry distributed tracing
@Service
public class OrderService {
    private final Tracer tracer;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    
    public OrderService(Tracer tracer, 
                       PaymentService paymentService,
                       InventoryService inventoryService) {
        this.tracer = tracer;
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
    
    public Order placeOrder(OrderRequest request) {
        // Create span for this operation
        Span span = tracer.spanBuilder("placeOrder")
            .setAttribute("user.id", request.getUserId())
            .setAttribute("order.amount", request.getAmount())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Check inventory - creates child span automatically
            boolean inStock = inventoryService.checkStock(request.getItems());
            span.addEvent("Inventory checked", 
                         Attributes.of(AttributeKey.booleanKey("in_stock"), inStock));
            
            if (!inStock) {
                span.setStatus(StatusCode.ERROR, "Out of stock");
                throw new OutOfStockException();
            }
            
            // Process payment - creates child span
            Payment payment = paymentService.processPayment(
                request.getUserId(), 
                request.getAmount()
            );
            span.addEvent("Payment processed",
                         Attributes.of(AttributeKey.stringKey("payment.id"), 
                                      payment.getId()));
            
            // Create order
            Order order = createOrder(request, payment);
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

**Trace Visualization**:

```
Request ID: abc-123-def-456
Total Duration: 250ms

Order Service (placeOrder) [0ms - 250ms]
  │
  ├─ Inventory Service (checkStock) [10ms - 60ms]
  │   │
  │   └─ Database Query (SELECT inventory) [15ms - 55ms]
  │
  └─ Payment Service (processPayment) [70ms - 240ms]
      │
      ├─ Payment Gateway API [80ms - 220ms]  ⚠️ Slow!
      │
      └─ Database Insert (payment record) [225ms - 235ms]
```

## Alerting Strategies

### Alert Fatigue Prevention

```python
class AlertManager:
    def __init__(self):
        self.alert_history = {}
        self.suppression_window = 300  # 5 minutes
    
    def should_alert(self, alert_name, severity, value):
        """Prevent alert fatigue with intelligent suppression"""
        
        # Check if we recently alerted
        if alert_name in self.alert_history:
            last_alert = self.alert_history[alert_name]
            time_since = time.time() - last_alert['timestamp']
            
            # Suppress duplicate alerts within window
            if time_since < self.suppression_window:
                # Unless severity increased
                if severity <= last_alert['severity']:
                    return False
        
        # Check if value is significantly different
        if alert_name in self.alert_history:
            last_value = self.alert_history[alert_name]['value']
            # Require 20% change to re-alert
            if abs(value - last_value) / last_value < 0.2:
                return False
        
        # Record this alert
        self.alert_history[alert_name] = {
            'timestamp': time.time(),
            'severity': severity,
            'value': value
        }
        
        return True
    
    def create_alert(self, name, description, severity, runbook_url):
        """Create actionable alert"""
        return {
            'name': name,
            'description': description,
            'severity': severity,  # CRITICAL, WARNING, INFO
            'runbook': runbook_url,
            'dashboard': f"https://grafana.com/d/{name}",
            'timestamp': time.time()
        }

# Usage
alert_mgr = AlertManager()

if error_rate > 0.05:  # 5% error rate
    if alert_mgr.should_alert('high_error_rate', 'CRITICAL', error_rate):
        alert = alert_mgr.create_alert(
            name='high_error_rate',
            description=f'Error rate is {error_rate:.1%}, threshold is 5%',
            severity='CRITICAL',
            runbook_url='https://wiki.company.com/runbooks/high-error-rate'
        )
        send_alert(alert)
```

### SLI/SLO-Based Alerting

```java
public class SLOMonitor {
    private static final double TARGET_AVAILABILITY = 0.999;  // 99.9%
    private final MetricsRegistry metrics;
    
    public void checkSLO() {
        long windowStart = Instant.now().minus(Duration.ofHours(1)).toEpochMilli();
        long windowEnd = Instant.now().toEpochMilli();
        
        // Calculate SLI (Service Level Indicator)
        long totalRequests = metrics.counter("requests.total", windowStart, windowEnd);
        long successfulRequests = metrics.counter("requests.successful", windowStart, windowEnd);
        
        double availability = (double) successfulRequests / totalRequests;
        
        // Calculate error budget consumption
        double errorBudget = 1 - TARGET_AVAILABILITY;  // 0.1%
        double actualErrors = 1 - availability;
        double budgetConsumed = actualErrors / errorBudget;
        
        // Alert if burning error budget too fast
        if (budgetConsumed > 0.5) {  // Used 50% of monthly budget in 1 hour
            Alert alert = new Alert()
                .severity(Severity.CRITICAL)
                .title("Error Budget Burn Rate Too High")
                .description(String.format(
                    "Consumed %.1f%% of error budget in 1 hour. " +
                    "Current availability: %.3f%%, Target: %.1f%%",
                    budgetConsumed * 100,
                    availability * 100,
                    TARGET_AVAILABILITY * 100
                ))
                .runbook("https://runbooks.company.com/error-budget-burn");
            
            alertManager.send(alert);
        }
    }
}
```

## Observability Patterns

### Context Propagation

```javascript
// Express middleware for request tracing
const tracingMiddleware = (req, res, next) => {
    // Extract or generate trace ID
    const traceId = req.headers['x-trace-id'] || generateTraceId();
    const spanId = generateSpanId();
    
    // Store in request context
    req.context = {
        traceId,
        spanId,
        parentSpanId: req.headers['x-span-id'],
        startTime: Date.now()
    };
    
    // Add to response headers
    res.setHeader('X-Trace-Id', traceId);
    res.setHeader('X-Span-Id', spanId);
    
    // Log request
    logger.info('Request started', {
        traceId,
        spanId,
        method: req.method,
        path: req.path,
        userAgent: req.headers['user-agent']
    });
    
    // Capture response
    res.on('finish', () => {
        const duration = Date.now() - req.context.startTime;
        
        logger.info('Request completed', {
            traceId,
            spanId,
            statusCode: res.statusCode,
            duration
        });
        
        // Record metrics
        requestDuration.record(duration, {
            method: req.method,
            path: req.path,
            status: res.statusCode
        });
    });
    
    next();
};

// Propagate context to downstream services
const callDownstreamService = async (url, data, context) => {
    const response = await fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-Trace-Id': context.traceId,
            'X-Span-Id': generateSpanId(),
            'X-Parent-Span-Id': context.spanId
        },
        body: JSON.stringify(data)
    });
    
    return response.json();
};
```

### Health Check Endpoint

```python
from flask import Flask, jsonify
import time

app = Flask(__name__)

class HealthChecker:
    def __init__(self):
        self.checks = []
    
    def register_check(self, name, check_fn, critical=True):
        self.checks.append({
            'name': name,
            'check': check_fn,
            'critical': critical
        })
    
    def run_checks(self):
        results = {}
        overall_status = 'healthy'
        
        for check in self.checks:
            start = time.time()
            try:
                is_healthy, message = check['check']()
                duration = time.time() - start
                
                results[check['name']] = {
                    'status': 'healthy' if is_healthy else 'unhealthy',
                    'message': message,
                    'duration_ms': int(duration * 1000),
                    'critical': check['critical']
                }
                
                if not is_healthy and check['critical']:
                    overall_status = 'unhealthy'
                    
            except Exception as e:
                results[check['name']] = {
                    'status': 'unhealthy',
                    'message': str(e),
                    'critical': check['critical']
                }
                if check['critical']:
                    overall_status = 'unhealthy'
        
        return {
            'status': overall_status,
            'checks': results,
            'timestamp': time.time()
        }

health_checker = HealthChecker()

# Register checks
health_checker.register_check(
    'database', 
    lambda: check_database_connection(),
    critical=True
)

health_checker.register_check(
    'redis',
    lambda: check_redis_connection(),
    critical=True
)

health_checker.register_check(
    'external_api',
    lambda: check_external_api(),
    critical=False  # Non-critical dependency
)

@app.route('/health')
def health():
    result = health_checker.run_checks()
    status_code = 200 if result['status'] == 'healthy' else 503
    return jsonify(result), status_code
```

## Observability Tools

### Tool Comparison

| Tool | Purpose | Strengths | Use Case |
|------|---------|-----------|----------|
| Prometheus | Metrics | Pull-based, powerful queries | Time-series metrics, alerting |
| Grafana | Visualization | Beautiful dashboards | Metrics visualization |
| ELK Stack | Logs | Full-text search, powerful | Centralized logging |
| Jaeger | Tracing | Distributed tracing | Microservices tracing |
| Datadog | All-in-one | Integrated solution | Complete observability |
| New Relic | APM | Application performance | Performance monitoring |

### Prometheus Metrics Example

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'api-service'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:8080']
    
  - job_name: 'database'
    scrape_interval: 30s
    static_configs:
      - targets: ['localhost:9090']
```

```java
// Expose Prometheus metrics
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "order-service")
            .commonTags("environment", environment)
            .commonTags("instance", instanceId);
    }
}

// PromQL queries
// Request rate
rate(http_requests_total[5m])

// Error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

// P95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

// CPU usage by pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name)
```

## Best Practices

**Instrument Everything**: Add metrics, logs, and traces to all code paths. You can always reduce collection later.

**Use Correlation IDs**: Track requests across services with trace IDs. Essential for debugging distributed systems.

**Monitor Business Metrics**: Technical metrics (CPU, latency) matter, but business metrics (revenue, conversions) matter more.

**Set Meaningful Alerts**: Alert on symptoms (high latency, errors) not causes (high CPU). Let on-call engineers sleep.

**Build Dashboards for Roles**: Different dashboards for operators (infrastructure health), developers (application metrics), and executives (business KPIs).

**Practice Incident Response**: Regular game days where you simulate outages and practice using observability tools.

## Anti-Patterns

❌ **Alert on Everything**: Too many alerts cause fatigue. Alert only on actionable problems.

❌ **No Runbooks**: Alerts without remediation steps waste time during incidents.

❌ **Ignoring Tail Latency**: P50 looks fine but P99 is terrible. Most users experience tail latency.

❌ **Vanity Metrics**: Tracking metrics that don't drive decisions. Every dashboard should inform action.

❌ **No Context in Logs**: Logging "Error occurred" without request ID, user ID, or stack trace.

## Real-World Examples

**Google**: Invented SRE practices. Uses Borgmon for monitoring, Dapper for tracing. Error budgets drive deployment decisions.

**Netflix**: Built custom observability tools (Atlas for metrics, Mantis for streaming). Uses observability to validate chaos experiments.

**Uber**: Built Jaeger for distributed tracing. Processes 2000+ traces/second. Open-sourced for the community.

**Amazon**: CloudWatch provides unified observability. X-Ray for distributed tracing. Alarms based on anomaly detection.

## Related Topics

- [High Availability & Fault Tolerance](12-high-availability-fault-tolerance.md)
- [Distributed Systems Fundamentals](11-distributed-systems-fundamentals.md)
- [Load Balancing](01-load-balancing.md)
- [Rate Limiting](07-rate-limiting.md)
