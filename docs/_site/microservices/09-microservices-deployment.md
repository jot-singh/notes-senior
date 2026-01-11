# Microservices Deployment

## Overview
Deploying microservices requires orchestrating multiple independent services, managing their lifecycles, ensuring zero-downtime updates, and handling the complexity of distributed systems. This note covers containerization, orchestration, CI/CD, and deployment strategies.

---

## Containerization with Docker

### Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# Copy only dependency files first (better caching)
COPY gradle gradle
COPY gradlew build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon

# Copy source and build
COPY src src
RUN ./gradlew bootJar --no-daemon

# Extract layered JAR
RUN java -Djarmode=layertools -jar build/libs/*.jar extract

# Runtime stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copy layers in order of change frequency
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Production-Ready Configuration

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine

# Security: Run as non-root
RUN addgroup -S app && adduser -S app -G app

# Install debugging tools (optional for production)
RUN apk add --no-cache curl

WORKDIR /app

# Copy application
COPY --chown=app:app target/*.jar app.jar

USER app

# JVM tuning for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:InitialRAMPercentage=50.0 \
    -XX:+UseG1GC \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/tmp/heapdump.hprof \
    -Djava.security.egd=file:/dev/./urandom"

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s \
    CMD curl -f http://localhost:8080/actuator/health/liveness || exit 1

EXPOSE 8080

ENTRYPOINT exec java $JAVA_OPTS -jar app.jar
```

### Docker Compose for Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/orders
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka:8761/eureka
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    ports:
      - "8082:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATA_MONGODB_URI=mongodb://mongo:27017/products
    depends_on:
      - mongo
    networks:
      - microservices-network

  api-gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
    depends_on:
      - eureka
    networks:
      - microservices-network

  eureka:
    build:
      context: ./eureka-server
      dockerfile: Dockerfile
    ports:
      - "8761:8761"
    networks:
      - microservices-network

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo:
    image: mongo:6
    volumes:
      - mongo-data:/data/db
    networks:
      - microservices-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 3

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - microservices-network

networks:
  microservices-network:
    driver: bridge

volumes:
  postgres-data:
  mongo-data:
```

---

## Kubernetes Deployment

### Deployment Manifest

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: order-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
        - name: order-service
          image: myregistry/order-service:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: kubernetes
            - name: JAVA_OPTS
              value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-db-secret
                  key: password
          envFrom:
            - configMapRef:
                name: order-service-config
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30
          volumeMounts:
            - name: config-volume
              mountPath: /config
              readOnly: true
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
      volumes:
        - name: config-volume
          configMap:
            name: order-service-config
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: order-service
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: order-service
```

### Service and Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: order-service

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /products(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
```

### ConfigMap and Secrets

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-service:5432/orders"
  SPRING_KAFKA_BOOTSTRAP_SERVERS: "kafka-service:9092"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,prometheus,info"
  LOGGING_LEVEL_ROOT: "INFO"
  
---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-db-secret
  namespace: production
type: Opaque
data:
  username: b3JkZXItdXNlcg==  # base64 encoded
  password: c2VjcmV0LXBhc3N3b3Jk  # base64 encoded
```

### Horizontal Pod Autoscaler

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

---

## CI/CD Pipeline

### GitHub Actions Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
      
      - name: Run tests
        run: ./gradlew test
      
      - name: Generate test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: '**/build/test-results/test/*.xml'
          reporter: java-junit
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./build/reports/jacoco/test/jacocoTestReport.xml

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          
      - name: Run OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          path: '.'
          format: 'HTML'

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name staging-cluster
      
      - name: Deploy to staging
        run: |
          kubectl set image deployment/order-service \
            order-service=${{ needs.build-and-push.outputs.image-tag }} \
            -n staging
          kubectl rollout status deployment/order-service -n staging

  deploy-production:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name production-cluster
      
      - name: Deploy with canary
        run: |
          # Deploy canary (10% traffic)
          kubectl apply -f k8s/canary-deployment.yaml
          
          # Wait and verify
          sleep 60
          
          # Check error rate
          ERROR_RATE=$(kubectl exec -n monitoring prometheus-0 -- \
            promtool query instant 'rate(http_server_requests_seconds_count{status=~"5.."}[5m])')
          
          if [ "$ERROR_RATE" -gt "0.01" ]; then
            echo "Canary failed, rolling back"
            kubectl rollout undo deployment/order-service-canary -n production
            exit 1
          fi
          
          # Promote to full deployment
          kubectl set image deployment/order-service \
            order-service=${{ needs.build-and-push.outputs.image-tag }} \
            -n production
          kubectl rollout status deployment/order-service -n production
```

---

## Deployment Strategies

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                             │
│                                                                      │
│   Before Switch:                                                    │
│                                                                      │
│   ┌──────────────────┐     100%     ┌──────────────────┐           │
│   │   Load Balancer  │─────────────►│  Blue (v1.0)     │           │
│   └──────────────────┘              │  ACTIVE          │           │
│                                     └──────────────────┘           │
│                                                                      │
│                                     ┌──────────────────┐           │
│                                     │  Green (v1.1)    │           │
│                                     │  STANDBY         │           │
│                                     └──────────────────┘           │
│                                                                      │
│   After Switch:                                                     │
│                                                                      │
│   ┌──────────────────┐              ┌──────────────────┐           │
│   │   Load Balancer  │              │  Blue (v1.0)     │           │
│   └──────────────────┘              │  STANDBY         │           │
│          │                          └──────────────────┘           │
│          │                                                          │
│          │         100%             ┌──────────────────┐           │
│          └─────────────────────────►│  Green (v1.1)    │           │
│                                     │  ACTIVE          │           │
│                                     └──────────────────┘           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```yaml
# blue-green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: green
  template:
    metadata:
      labels:
        app: order-service
        version: green
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.1.0

---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
    version: blue  # Switch to 'green' for cutover
  ports:
    - port: 80
      targetPort: 8080
```

### Canary Deployment

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CANARY DEPLOYMENT                                 │
│                                                                      │
│   Phase 1: Initial (10% to canary)                                  │
│                                                                      │
│   ┌──────────────────┐    90%      ┌──────────────────┐            │
│   │   Load Balancer  │────────────►│  Stable (v1.0)   │            │
│   └──────────────────┘             │  3 replicas      │            │
│          │                         └──────────────────┘            │
│          │                                                          │
│          │   10%                   ┌──────────────────┐            │
│          └────────────────────────►│  Canary (v1.1)   │            │
│                                    │  1 replica       │            │
│                                    └──────────────────┘            │
│                                                                      │
│   Phase 2: Progressive (50% to canary)                              │
│                                                                      │
│   ┌──────────────────┐    50%      ┌──────────────────┐            │
│   │   Load Balancer  │────────────►│  Stable (v1.0)   │            │
│   └──────────────────┘             │  2 replicas      │            │
│          │                         └──────────────────┘            │
│          │                                                          │
│          │   50%                   ┌──────────────────┐            │
│          └────────────────────────►│  Canary (v1.1)   │            │
│                                    │  2 replicas      │            │
│                                    └──────────────────┘            │
│                                                                      │
│   Phase 3: Full promotion (100% to new version)                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```yaml
# Istio canary with traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
  namespace: production
spec:
  hosts:
    - order-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: canary
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
  namespace: production
spec:
  host: order-service
  subsets:
    - name: stable
      labels:
        version: v1.0.0
    - name: canary
      labels:
        version: v1.1.0
```

### Rolling Deployment

```yaml
# Rolling deployment (default Kubernetes)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods over desired count
      maxUnavailable: 0  # Zero downtime
  template:
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:1.1.0
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

## Graceful Shutdown

### Spring Boot Configuration

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        readiness:
          include: readinessState,db,kafka
        liveness:
          include: livenessState
```

```java
@Component
@Slf4j
public class GracefulShutdownHandler {
    
    @PreDestroy
    public void onShutdown() {
        log.info("Application shutting down gracefully...");
        
        // Deregister from service discovery
        // Complete in-flight requests
        // Close database connections
        // Flush caches
    }
    
    @EventListener(ContextClosedEvent.class)
    public void onContextClosed(ContextClosedEvent event) {
        log.info("Context closed, performing cleanup...");
    }
}
```

### Kubernetes PreStop Hook

```yaml
spec:
  containers:
    - name: order-service
      lifecycle:
        preStop:
          exec:
            # Wait for load balancer to remove pod from rotation
            command: ["sh", "-c", "sleep 10"]
      # Termination grace period
      terminationGracePeriodSeconds: 60
```

---

## Health Checks

### Spring Boot Actuator

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    private final DataSource dataSource;
    private final KafkaTemplate kafkaTemplate;
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        boolean healthy = true;
        
        // Check database
        try {
            dataSource.getConnection().isValid(1);
            details.put("database", "UP");
        } catch (SQLException e) {
            healthy = false;
            details.put("database", "DOWN: " + e.getMessage());
        }
        
        // Check Kafka
        try {
            kafkaTemplate.getProducerFactory().createProducer().metrics();
            details.put("kafka", "UP");
        } catch (Exception e) {
            healthy = false;
            details.put("kafka", "DOWN: " + e.getMessage());
        }
        
        return healthy 
            ? Health.up().withDetails(details).build()
            : Health.down().withDetails(details).build();
    }
}
```

### Kubernetes Probes

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES HEALTH PROBES                          │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    STARTUP PROBE                             │   │
│   │                                                              │   │
│   │   Purpose: Check if app has started                          │   │
│   │   When: During container startup only                        │   │
│   │   Failure: Container is restarted                            │   │
│   │   Use for: Slow-starting applications                        │   │
│   │                                                              │   │
│   │   startupProbe:                                              │   │
│   │     httpGet:                                                 │   │
│   │       path: /actuator/health/liveness                        │   │
│   │     failureThreshold: 30                                     │   │
│   │     periodSeconds: 10                                        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼ (Once passed)                         │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    LIVENESS PROBE                            │   │
│   │                                                              │   │
│   │   Purpose: Check if app is alive                             │   │
│   │   When: Throughout container lifetime                        │   │
│   │   Failure: Container is restarted                            │   │
│   │   Use for: Detecting deadlocks, infinite loops               │   │
│   │                                                              │   │
│   │   livenessProbe:                                             │   │
│   │     httpGet:                                                 │   │
│   │       path: /actuator/health/liveness                        │   │
│   │     periodSeconds: 15                                        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    READINESS PROBE                           │   │
│   │                                                              │   │
│   │   Purpose: Check if app can serve traffic                    │   │
│   │   When: Throughout container lifetime                        │   │
│   │   Failure: Pod removed from Service endpoints               │   │
│   │   Use for: Checking dependencies (DB, cache, etc.)          │   │
│   │                                                              │   │
│   │   readinessProbe:                                            │   │
│   │     httpGet:                                                 │   │
│   │       path: /actuator/health/readiness                       │   │
│   │     periodSeconds: 10                                        │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Helm Charts

```yaml
# Chart.yaml
apiVersion: v2
name: order-service
description: Order Service Helm Chart
version: 1.0.0
appVersion: "1.0.0"

---
# values.yaml
replicaCount: 3

image:
  repository: myregistry/order-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.example.com
      paths:
        - path: /orders
          pathType: Prefix

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: kubernetes
  JAVA_OPTS: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

secrets:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-db-secret
        key: password

---
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            {{- range .Values.secrets }}
            - name: {{ .name }}
              valueFrom:
                secretKeyRef:
                  name: {{ .valueFrom.secretKeyRef.name }}
                  key: {{ .valueFrom.secretKeyRef.key }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## Interview Questions

### Conceptual Questions
1. **Compare Blue-Green vs Canary deployment strategies.**
2. **What are the three Kubernetes health probes and when to use each?**
3. **How do you achieve zero-downtime deployments?**
4. **Explain the importance of graceful shutdown in microservices.**
5. **What are the benefits of containerizing microservices?**

### Design Questions
1. **Design a CI/CD pipeline for a microservices application.**
2. **How would you implement automated rollback in Kubernetes?**
3. **Design a deployment strategy for a critical payment service.**
4. **How do you handle database migrations in microservices deployments?**
5. **Design a multi-region deployment architecture.**

### Practical Questions
1. **How do you troubleshoot a failing Kubernetes deployment?**
2. **What metrics would you monitor during a canary deployment?**
3. **How do you handle secrets in Kubernetes?**
4. **How do you configure resource limits for Java applications in containers?**
5. **What's your strategy for container image security scanning?**

---

## Key Takeaways

1. **Multi-stage Docker builds** reduce image size and improve security
2. **Container resource limits** are essential - use `UseContainerSupport` for JVM
3. **Health probes**: startup for slow apps, liveness for restarts, readiness for traffic
4. **Graceful shutdown** prevents request failures during deployments
5. **Rolling updates** with `maxUnavailable: 0` ensure zero downtime
6. **Canary deployments** reduce risk by gradual traffic shifting
7. **HPA** provides automatic scaling based on metrics
8. **Helm** simplifies deployment configuration management
9. **CI/CD automation** ensures consistent, repeatable deployments
10. **Security scanning** should be part of every deployment pipeline

---

*Previous: [Inter-Service Communication](08-inter-service-communication.md) | Next: [Configuration Management](10-configuration-management.md)*
