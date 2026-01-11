# Distributed Tracing

## Overview
Distributed tracing is a method for tracking requests as they flow through distributed systems. It provides visibility into the entire lifecycle of a request, helping identify performance bottlenecks, failures, and dependencies across microservices.

---

## Why Distributed Tracing?

### The Problem

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEBUGGING WITHOUT TRACING                         │
│                                                                      │
│   User reports: "Order checkout is slow"                            │
│                                                                      │
│   Request flow (unknown):                                            │
│   Client → ??? → ??? → ??? → ??? → Response                         │
│                                                                      │
│   Questions:                                                         │
│   • Which service is slow?                                          │
│   • What's the call sequence?                                       │
│   • How long does each step take?                                   │
│   • Are there any errors?                                           │
│   • How many downstream calls are made?                             │
│                                                                      │
│   Traditional logs:                                                  │
│   [Order-Service] 10:30:00 Processing order 123                     │
│   [User-Service]  10:30:00 Fetching user 456                        │
│   [Payment-Svc]   10:30:01 Processing payment                       │
│   [Inventory-Svc] 10:30:00 Checking stock                           │
│                                                                      │
│   Problem: Logs are disconnected, no correlation!                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEBUGGING WITH TRACING                            │
│                                                                      │
│   Trace ID: abc-123-xyz                                             │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ API Gateway [Span 1]                                   100ms│   │
│   │ ├─────────────────────────────────────────────────────────┤│   │
│   │ │ Order Service [Span 2]                            80ms  ││   │
│   │ │ ├───────────────────────────────────────────────────────┤│   │
│   │ │ │ User Service [Span 3]                    15ms         ││   │
│   │ │ ├───────────────────────────────────────────────────────┤│   │
│   │ │ │ Inventory Service [Span 4]               20ms         ││   │
│   │ │ │ └── Database Query [Span 5]        10ms               ││   │
│   │ │ ├───────────────────────────────────────────────────────┤│   │
│   │ │ │ Payment Service [Span 6]                 35ms ← SLOW! ││   │
│   │ │ │ └── Payment Gateway [Span 7]       30ms               ││   │
│   │ │ └───────────────────────────────────────────────────────┘│   │
│   │ └─────────────────────────────────────────────────────────┘│   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   Finding: Payment Gateway external call is the bottleneck          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### Trace, Span, and Context

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TRACING CONCEPTS                             │
│                                                                      │
│   TRACE: End-to-end journey of a request                            │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Trace ID: abc-123-xyz                                       │   │
│   │                                                              │   │
│   │   SPAN: A unit of work (service call, DB query, etc.)       │   │
│   │   ┌───────────────────────────────────────────────────────┐ │   │
│   │   │ Span ID: span-001                                     │ │   │
│   │   │ Parent ID: null (root span)                           │ │   │
│   │   │ Operation: HTTP GET /orders                           │ │   │
│   │   │ Service: api-gateway                                  │ │   │
│   │   │ Duration: 100ms                                       │ │   │
│   │   │ Tags: {http.method: GET, http.status: 200}           │ │   │
│   │   │ Logs: [{timestamp, event: "received request"}]        │ │   │
│   │   └───────────────────────────────────────────────────────┘ │   │
│   │         │                                                    │   │
│   │         ▼ Parent-Child Relationship                         │   │
│   │   ┌───────────────────────────────────────────────────────┐ │   │
│   │   │ Span ID: span-002                                     │ │   │
│   │   │ Parent ID: span-001                                   │ │   │
│   │   │ Operation: processOrder                               │ │   │
│   │   │ Service: order-service                                │ │   │
│   │   │ Duration: 80ms                                        │ │   │
│   │   └───────────────────────────────────────────────────────┘ │   │
│   │                                                              │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   CONTEXT PROPAGATION:                                               │
│   Headers passed between services:                                   │
│   • traceparent: 00-abc123xyz-span001-01                            │
│   • tracestate: vendor=value                                        │
│   • baggage: user-id=123, tenant-id=abc                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### OpenTelemetry Data Model

```java
// Span Attributes (Tags)
Span span = tracer.spanBuilder("processOrder")
    .setSpanKind(SpanKind.INTERNAL)
    .startSpan();

// Standard semantic conventions
span.setAttribute(SemanticAttributes.HTTP_METHOD, "POST");
span.setAttribute(SemanticAttributes.HTTP_URL, "http://order-service/orders");
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);
span.setAttribute(SemanticAttributes.DB_SYSTEM, "postgresql");
span.setAttribute(SemanticAttributes.DB_STATEMENT, "SELECT * FROM orders");

// Custom attributes
span.setAttribute("order.id", orderId);
span.setAttribute("order.amount", amount);
span.setAttribute("customer.tier", "premium");

// Span Events (Logs)
span.addEvent("Order validated");
span.addEvent("Payment initiated", Attributes.of(
    AttributeKey.stringKey("payment.method"), "credit_card"
));

// Span Status
span.setStatus(StatusCode.OK);
span.setStatus(StatusCode.ERROR, "Payment failed");

span.end();
```

---

## OpenTelemetry Implementation

### Setup and Configuration

```xml
<!-- Maven dependencies -->
<dependencies>
    <!-- OpenTelemetry API -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api</artifactId>
        <version>1.32.0</version>
    </dependency>
    
    <!-- OpenTelemetry SDK -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk</artifactId>
        <version>1.32.0</version>
    </dependency>
    
    <!-- Auto-instrumentation -->
    <dependency>
        <groupId>io.opentelemetry.instrumentation</groupId>
        <artifactId>opentelemetry-instrumentation-bom</artifactId>
        <version>1.32.0</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
    
    <!-- Exporter - Jaeger -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-jaeger</artifactId>
        <version>1.32.0</version>
    </dependency>
    
    <!-- Exporter - OTLP (Recommended) -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
        <version>1.32.0</version>
    </dependency>
</dependencies>
```

```java
// OpenTelemetry SDK Configuration
@Configuration
public class OpenTelemetryConfig {
    
    @Bean
    public OpenTelemetry openTelemetry() {
        // Resource - identifies the service
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "order-service",
                ResourceAttributes.SERVICE_VERSION, "1.0.0",
                ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production"
            )));
        
        // OTLP Exporter
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .setTimeout(Duration.ofSeconds(10))
            .build();
        
        // Span Processor
        SpanProcessor spanProcessor = BatchSpanProcessor.builder(spanExporter)
            .setMaxQueueSize(2048)
            .setMaxExportBatchSize(512)
            .setScheduleDelay(Duration.ofSeconds(5))
            .build();
        
        // Sampler - control how many traces to collect
        Sampler sampler = Sampler.parentBased(
            Sampler.traceIdRatioBased(0.1)  // Sample 10% of traces
        );
        
        // Tracer Provider
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(spanProcessor)
            .setSampler(sampler)
            .build();
        
        // Context Propagators
        ContextPropagators propagators = ContextPropagators.create(
            TextMapPropagator.composite(
                W3CTraceContextPropagator.getInstance(),
                W3CBaggagePropagator.getInstance()
            )
        );
        
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(propagators)
            .buildAndRegisterGlobal();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("order-service", "1.0.0");
    }
}
```

### Manual Instrumentation

```java
@Service
@Slf4j
public class OrderService {
    
    private final Tracer tracer;
    private final InventoryServiceClient inventoryClient;
    private final PaymentServiceClient paymentClient;
    
    public Order createOrder(CreateOrderRequest request) {
        // Create root span
        Span span = tracer.spanBuilder("createOrder")
            .setSpanKind(SpanKind.INTERNAL)
            .setAttribute("order.customer_id", request.getCustomerId())
            .setAttribute("order.item_count", request.getItems().size())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            span.addEvent("Validating order");
            validateOrder(request);
            
            span.addEvent("Checking inventory");
            checkInventory(request.getItems());
            
            span.addEvent("Processing payment");
            processPayment(request);
            
            Order order = saveOrder(request);
            
            span.setAttribute("order.id", order.getId());
            span.setAttribute("order.total", order.getTotal().doubleValue());
            span.setStatus(StatusCode.OK);
            
            return order;
            
        } catch (ValidationException e) {
            span.setStatus(StatusCode.ERROR, "Validation failed: " + e.getMessage());
            span.recordException(e);
            throw e;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, "Order creation failed");
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    private void checkInventory(List<OrderItem> items) {
        Span span = tracer.spanBuilder("checkInventory")
            .setSpanKind(SpanKind.CLIENT)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            for (OrderItem item : items) {
                // Create child span for each item check
                Span itemSpan = tracer.spanBuilder("checkItemStock")
                    .setAttribute("product.id", item.getProductId())
                    .setAttribute("quantity.requested", item.getQuantity())
                    .startSpan();
                
                try (Scope itemScope = itemSpan.makeCurrent()) {
                    boolean available = inventoryClient.checkStock(
                        item.getProductId(), 
                        item.getQuantity()
                    );
                    itemSpan.setAttribute("stock.available", available);
                } finally {
                    itemSpan.end();
                }
            }
        } finally {
            span.end();
        }
    }
}
```

### Auto-Instrumentation

```bash
# Download OpenTelemetry Java Agent
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v1.32.0/opentelemetry-javaagent.jar

# Run application with agent
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.traces.exporter=otlp \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.metrics.exporter=otlp \
     -Dotel.logs.exporter=otlp \
     -jar order-service.jar
```

```yaml
# Docker Compose with agent
services:
  order-service:
    image: order-service:latest
    environment:
      JAVA_TOOL_OPTIONS: "-javaagent:/opentelemetry-javaagent.jar"
      OTEL_SERVICE_NAME: order-service
      OTEL_TRACES_EXPORTER: otlp
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_METRICS_EXPORTER: otlp
      OTEL_LOGS_EXPORTER: otlp
    volumes:
      - ./opentelemetry-javaagent.jar:/opentelemetry-javaagent.jar
```

### Context Propagation

```java
// HTTP Client - Inject context into headers
@Component
public class TracingRestTemplate {
    
    private final RestTemplate restTemplate;
    private final TextMapPropagator propagator;
    
    public TracingRestTemplate(OpenTelemetry openTelemetry) {
        this.restTemplate = new RestTemplate();
        this.propagator = openTelemetry.getPropagators().getTextMapPropagator();
        
        // Add interceptor to inject trace context
        this.restTemplate.setInterceptors(List.of(new TracingInterceptor()));
    }
    
    private class TracingInterceptor implements ClientHttpRequestInterceptor {
        @Override
        public ClientHttpResponse intercept(
                HttpRequest request, byte[] body, 
                ClientHttpRequestExecution execution) throws IOException {
            
            // Inject current context into headers
            propagator.inject(Context.current(), request.getHeaders(), 
                (headers, key, value) -> headers.add(key, value));
            
            return execution.execute(request, body);
        }
    }
}

// HTTP Server - Extract context from headers
@Component
public class TracingFilter implements Filter {
    
    private final TextMapPropagator propagator;
    private final Tracer tracer;
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest request = (HttpServletRequest) req;
        
        // Extract context from incoming headers
        Context extractedContext = propagator.extract(
            Context.current(),
            request,
            new TextMapGetter<HttpServletRequest>() {
                @Override
                public Iterable<String> keys(HttpServletRequest carrier) {
                    return Collections.list(carrier.getHeaderNames());
                }
                
                @Override
                public String get(HttpServletRequest carrier, String key) {
                    return carrier.getHeader(key);
                }
            }
        );
        
        // Create span with extracted context as parent
        Span span = tracer.spanBuilder(request.getMethod() + " " + request.getRequestURI())
            .setParent(extractedContext)
            .setSpanKind(SpanKind.SERVER)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            chain.doFilter(req, res);
            span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 
                ((HttpServletResponse) res).getStatus());
        } finally {
            span.end();
        }
    }
}
```

### Baggage (Cross-Cutting Data)

```java
// Add baggage (propagated to all downstream services)
@Service
public class OrderService {
    
    public Order createOrder(CreateOrderRequest request, String userId) {
        // Add user context to baggage
        Baggage baggage = Baggage.builder()
            .put("user.id", userId)
            .put("tenant.id", request.getTenantId())
            .put("request.source", "web")
            .build();
        
        try (Scope scope = baggage.makeCurrent()) {
            // All downstream calls will include this baggage
            return processOrder(request);
        }
    }
}

// Read baggage in downstream service
@Service
public class InventoryService {
    
    public void checkStock(String productId) {
        // Get baggage from context
        String userId = Baggage.current().getEntryValue("user.id");
        String tenantId = Baggage.current().getEntryValue("tenant.id");
        
        log.info("Checking stock for user {} in tenant {}", userId, tenantId);
    }
}
```

---

## Tracing Backends

### Jaeger

```yaml
# docker-compose.yml
services:
  jaeger:
    image: jaegertracing/all-in-one:1.50
    ports:
      - "6831:6831/udp"   # Jaeger thrift
      - "16686:16686"     # UI
      - "14268:14268"     # HTTP collector
      - "14250:14250"     # gRPC collector
      - "4317:4317"       # OTLP gRPC
      - "4318:4318"       # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: true
```

```java
// Jaeger exporter configuration
@Bean
public SpanExporter jaegerExporter() {
    return JaegerGrpcSpanExporter.builder()
        .setEndpoint("http://jaeger:14250")
        .setTimeout(Duration.ofSeconds(10))
        .build();
}
```

### Zipkin

```yaml
# docker-compose.yml
services:
  zipkin:
    image: openzipkin/zipkin:2.24
    ports:
      - "9411:9411"
```

```java
// Zipkin exporter configuration
@Bean
public SpanExporter zipkinExporter() {
    return ZipkinSpanExporter.builder()
        .setEndpoint("http://zipkin:9411/api/v2/spans")
        .build();
}
```

### OpenTelemetry Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 512
  
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024
    spike_limit_mib: 256
  
  attributes:
    actions:
      - key: environment
        value: production
        action: insert

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  prometheus:
    endpoint: 0.0.0.0:8889
  
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes]
      exporters: [jaeger, logging]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

```yaml
# docker-compose.yml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.88.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
      - "8889:8889"     # Prometheus metrics
```

---

## Spring Cloud Sleuth / Micrometer Tracing

### Setup (Spring Boot 3.x)

```xml
<!-- Spring Boot 3.x uses Micrometer Tracing -->
<dependencies>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-otlp</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling in dev
    propagation:
      type: w3c  # W3C Trace Context
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### Automatic Instrumentation

```java
// Spring Cloud Sleuth automatically instruments:
// - REST controllers
// - RestTemplate / WebClient
// - Feign clients
// - Kafka listeners/templates
// - JDBC queries
// - Scheduled tasks

@RestController
public class OrderController {
    
    private final OrderService orderService;
    private final RestTemplate restTemplate;  // Auto-instrumented
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        // Span automatically created for this endpoint
        return orderService.getOrder(id);
    }
}

// Custom span with @NewSpan
@Service
public class OrderService {
    
    @NewSpan("processOrder")
    public Order processOrder(@SpanTag("orderId") String orderId) {
        // New span created with orderId tag
        return doProcess(orderId);
    }
    
    // Continue existing span
    @ContinueSpan
    public void validateOrder(@SpanTag("orderId") String orderId) {
        // Adds tag to current span, doesn't create new one
    }
}
```

---

## Sampling Strategies

### Types of Sampling

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SAMPLING STRATEGIES                             │
│                                                                      │
│   1. HEAD-BASED SAMPLING (Decision at trace start)                  │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                                                              │   │
│   │   Request → Sample? → Yes → Trace entire request            │   │
│   │               │                                              │   │
│   │               └─→ No → Don't trace                          │   │
│   │                                                              │   │
│   │   Types:                                                     │   │
│   │   • Probability (10% of traces)                             │   │
│   │   • Rate limiting (100 traces/sec)                          │   │
│   │   • Always on/off                                           │   │
│   │                                                              │   │
│   │   Pros: Simple, predictable overhead                        │   │
│   │   Cons: May miss interesting traces                         │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   2. TAIL-BASED SAMPLING (Decision after trace completes)           │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                                                              │   │
│   │   Request → Collect all spans → Analyze → Keep/Discard      │   │
│   │                                                              │   │
│   │   Criteria:                                                  │   │
│   │   • Keep if error occurred                                  │   │
│   │   • Keep if latency > threshold                             │   │
│   │   • Keep if specific attribute present                      │   │
│   │                                                              │   │
│   │   Pros: Keeps interesting traces                            │   │
│   │   Cons: Complex, needs buffering, more resources            │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// Head-based sampling configuration
Sampler sampler = Sampler.parentBased(
    Sampler.traceIdRatioBased(0.1)  // 10% base rate
);

// Custom sampler
public class CustomSampler implements Sampler {
    
    @Override
    public SamplingResult shouldSample(
            Context parentContext,
            String traceId,
            String name,
            SpanKind spanKind,
            Attributes attributes,
            List<LinkData> parentLinks) {
        
        // Always sample errors
        if (attributes.get(AttributeKey.booleanKey("error")) == Boolean.TRUE) {
            return SamplingResult.recordAndSample();
        }
        
        // Always sample specific operations
        if (name.contains("payment") || name.contains("checkout")) {
            return SamplingResult.recordAndSample();
        }
        
        // Sample 10% of other traces
        if (Math.random() < 0.1) {
            return SamplingResult.recordAndSample();
        }
        
        return SamplingResult.drop();
    }
    
    @Override
    public String getDescription() {
        return "CustomSampler";
    }
}
```

```yaml
# OpenTelemetry Collector - Tail-based sampling
processors:
  tail_sampling:
    decision_wait: 10s  # Wait for trace to complete
    num_traces: 100000  # Buffer size
    expected_new_traces_per_sec: 1000
    policies:
      # Always sample errors
      - name: error-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      
      # Sample slow traces
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 1000
      
      # Sample specific operations
      - name: string-policy
        type: string_attribute
        string_attribute:
          key: http.url
          values: ["/api/payments", "/api/checkout"]
      
      # Probabilistic for everything else
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

service:
  pipelines:
    traces:
      processors: [tail_sampling, batch]
```

---

## Correlating Logs and Traces

### Adding Trace Context to Logs

```java
// Logback configuration (logback-spring.xml)
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} 
                [traceId=%X{traceId} spanId=%X{spanId}] - %msg%n
            </pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

```java
// Manual MDC for custom spans
@Service
public class OrderService {
    
    private final Tracer tracer;
    
    public void processOrder(String orderId) {
        Span span = tracer.spanBuilder("processOrder").startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Add trace context to MDC
            MDC.put("traceId", span.getSpanContext().getTraceId());
            MDC.put("spanId", span.getSpanContext().getSpanId());
            MDC.put("orderId", orderId);
            
            log.info("Processing order");  // Will include trace context
            
            doProcess(orderId);
            
        } finally {
            MDC.clear();
            span.end();
        }
    }
}
```

### Structured Logging with Traces

```java
// JSON structured logging
@Configuration
public class LoggingConfig {
    
    @Bean
    public LoggingEventCompositeJsonEncoder jsonEncoder() {
        LoggingEventCompositeJsonEncoder encoder = new LoggingEventCompositeJsonEncoder();
        // Include trace context in JSON logs
        encoder.setProviders(List.of(
            new TimestampJsonProvider(),
            new LogLevelJsonProvider(),
            new ThreadNameJsonProvider(),
            new LoggerNameJsonProvider(),
            new MessageJsonProvider(),
            new MdcJsonProvider()  // Includes traceId, spanId
        ));
        return encoder;
    }
}

// Output:
// {
//   "timestamp": "2024-01-15T10:30:00.000Z",
//   "level": "INFO",
//   "logger": "com.example.OrderService",
//   "message": "Processing order",
//   "traceId": "abc123xyz",
//   "spanId": "def456",
//   "orderId": "order-123"
// }
```

---

## Best Practices

### 1. Meaningful Span Names

```java
// Bad: Generic names
tracer.spanBuilder("process").startSpan();
tracer.spanBuilder("call").startSpan();

// Good: Descriptive names
tracer.spanBuilder("OrderService.createOrder").startSpan();
tracer.spanBuilder("HTTP GET /api/orders/{id}").startSpan();
tracer.spanBuilder("SELECT orders WHERE customer_id = ?").startSpan();
```

### 2. Rich Attributes

```java
// Add business context
Span span = tracer.spanBuilder("processPayment")
    .setAttribute("payment.method", "credit_card")
    .setAttribute("payment.amount", amount)
    .setAttribute("payment.currency", "USD")
    .setAttribute("customer.id", customerId)
    .setAttribute("customer.tier", "premium")
    .setAttribute("order.id", orderId)
    .startSpan();
```

### 3. Error Recording

```java
try {
    processPayment();
} catch (PaymentException e) {
    span.setStatus(StatusCode.ERROR, "Payment failed");
    span.recordException(e, Attributes.of(
        AttributeKey.stringKey("error.type"), e.getClass().getSimpleName(),
        AttributeKey.stringKey("error.code"), e.getErrorCode()
    ));
    throw e;
}
```

### 4. Sensitive Data Handling

```java
// Don't include sensitive data in traces
// Bad
span.setAttribute("customer.credit_card", "4111111111111111");
span.setAttribute("user.password", password);

// Good - use redaction
span.setAttribute("customer.credit_card_last4", "1111");
span.setAttribute("user.email_domain", email.split("@")[1]);
```

---

## Troubleshooting with Traces

### Common Patterns to Look For

```
┌─────────────────────────────────────────────────────────────────────┐
│                   TRACE ANALYSIS PATTERNS                            │
│                                                                      │
│   1. WATERFALL ANALYSIS                                              │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Service A ████████████████████████████████████  100ms       │   │
│   │ └─Service B ██████████                          20ms        │   │
│   │ └─Service C ████████████████████████            50ms ← Slow │   │
│   │   └─Database ██████████████████                 40ms        │   │
│   │ └─Service D ██████                              10ms        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   2. PARALLEL vs SEQUENTIAL                                          │
│   Sequential (inefficient):                                          │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ A ████████ B ████████ C ████████               Total: 30ms  │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   Parallel (better):                                                 │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ A ████████                                                   │   │
│   │ B ████████                                     Total: 10ms   │   │
│   │ C ████████                                                   │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   3. ERROR PROPAGATION                                               │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Gateway ████████████████ ERROR                              │   │
│   │ └─Order ██████████████ ERROR                                │   │
│   │   └─Payment ████████ ERROR ← Root cause                     │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   4. RETRY DETECTION                                                 │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Client call ████████████████████████████████████            │   │
│   │ └─Attempt 1 ████ FAIL                                       │   │
│   │ └─Attempt 2 ████ FAIL                                       │   │
│   │ └─Attempt 3 ████████████████ SUCCESS                        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Conceptual Questions
1. **What is distributed tracing and why is it important?**
2. **Explain the concepts of trace, span, and context propagation.**
3. **What's the difference between head-based and tail-based sampling?**
4. **How does W3C Trace Context work?**
5. **What is OpenTelemetry and how does it differ from vendor-specific solutions?**

### Design Questions
1. **How would you implement tracing for a high-throughput system?**
2. **Design a sampling strategy for a production system.**
3. **How do you correlate logs, metrics, and traces?**
4. **How would you handle sensitive data in traces?**
5. **Design a trace analysis system to detect anomalies.**

### Practical Questions
1. **How do you instrument a legacy application with tracing?**
2. **What's the performance overhead of distributed tracing?**
3. **How do you troubleshoot missing spans in a trace?**
4. **How do you set up trace-based alerting?**
5. **How do you handle tracing in async/event-driven systems?**

---

## Key Takeaways

1. **Distributed tracing connects the dots** across microservices
2. **Every trace needs a unique ID** propagated via headers (W3C Trace Context)
3. **Spans represent units of work** with timing, attributes, and relationships
4. **OpenTelemetry is the standard** - use it for vendor-neutral instrumentation
5. **Auto-instrumentation reduces effort** but manual spans add business context
6. **Sampling is essential** for high-volume systems
7. **Tail-based sampling captures interesting traces** but requires more resources
8. **Correlate traces with logs and metrics** for complete observability
9. **Don't include sensitive data** in trace attributes
10. **Use traces for debugging and performance optimization**

---

*Previous: [Service Mesh](05-service-mesh.md) | Next: [Circuit Breaker Pattern](07-circuit-breaker-pattern.md)*
