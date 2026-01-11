# Service Discovery

## What You'll Learn

Service discovery enables services to find and communicate with each other dynamically in distributed systems. This note covers service discovery patterns, implementations (Consul, Eureka, etcd), health checking, and production considerations for microservices architectures.

## Why This Matters

In microservices architectures with hundreds of services across thousands of instances, hardcoding service locations is impossible. Services scale up and down dynamically. Instances fail and restart with new IP addresses. Service discovery solves this by maintaining a registry of available services and their locations. Netflix's Eureka handles discovery for thousands of service instances. Kubernetes uses etcd for service discovery. Understanding service discovery is fundamental to building scalable microservices.

## Service Discovery Patterns

### Client-Side Discovery

Clients query a service registry to discover available instances and choose one using a load balancing algorithm. Netflix Ribbon implements this pattern.

```java
// Java: Client-side discovery with Eureka
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {
    
    @Bean
    @LoadBalanced  // Enable client-side load balancing
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

@Service
public class OrderService {
    
    @Autowired
    @LoadBalanced
    private RestTemplate restTemplate;
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    public User getUserById(String userId) {
        // RestTemplate automatically discovers USER-SERVICE instances
        // and load balances between them
        return restTemplate.getForObject(
            "http://USER-SERVICE/users/" + userId,
            User.class
        );
    }
    
    public List<ServiceInstance> getAvailableUserServices() {
        // Get all instances of USER-SERVICE
        return discoveryClient.getInstances("USER-SERVICE");
    }
    
    public User getUserWithCustomLoadBalancing(String userId) {
        List<ServiceInstance> instances = 
            discoveryClient.getInstances("USER-SERVICE");
        
        if (instances.isEmpty()) {
            throw new ServiceUnavailableException("No USER-SERVICE instances");
        }
        
        // Custom load balancing (e.g., random)
        ServiceInstance instance = instances.get(
            new Random().nextInt(instances.size())
        );
        
        String url = instance.getUri() + "/users/" + userId;
        return restTemplate.getForObject(url, User.class);
    }
}
```

### Server-Side Discovery

Clients make requests to a load balancer which queries the service registry and forwards requests to available instances. AWS ELB with ECS implements this pattern.

```javascript
// Node.js: Server-side discovery with Consul
const express = require('express');
const Consul = require('consul');
const axios = require('axios');

const app = express();
const consul = new Consul({
    host: 'consul-server',
    port: 8500
});

// Register service with Consul
async function registerService() {
    const serviceName = 'api-gateway';
    const serviceId = `${serviceName}-${process.env.HOSTNAME}`;
    
    await consul.agent.service.register({
        id: serviceId,
        name: serviceName,
        address: process.env.SERVICE_ADDRESS,
        port: parseInt(process.env.SERVICE_PORT || '3000'),
        tags: ['api', 'gateway', 'v1'],
        check: {
            http: `http://${process.env.SERVICE_ADDRESS}:3000/health`,
            interval: '10s',
            timeout: '5s',
            deregisterfailuresafter: '1m'
        }
    });
    
    console.log(`Registered service: ${serviceId}`);
}

// Discover service instances
async function discoverService(serviceName) {
    try {
        const result = await consul.health.service({
            service: serviceName,
            passing: true  // Only healthy instances
        });
        
        return result.map(entry => ({
            id: entry.Service.ID,
            address: entry.Service.Address,
            port: entry.Service.Port,
            tags: entry.Service.Tags
        }));
    } catch (error) {
        console.error(`Service discovery failed for ${serviceName}:`, error);
        return [];
    }
}

// Load balance across instances (round-robin)
const serviceCounters = {};

async function callService(serviceName, path) {
    const instances = await discoverService(serviceName);
    
    if (instances.length === 0) {
        throw new Error(`No healthy instances of ${serviceName}`);
    }
    
    // Round-robin load balancing
    if (!serviceCounters[serviceName]) {
        serviceCounters[serviceName] = 0;
    }
    
    const index = serviceCounters[serviceName] % instances.length;
    serviceCounters[serviceName]++;
    
    const instance = instances[index];
    const url = `http://${instance.address}:${instance.port}${path}`;
    
    try {
        const response = await axios.get(url, {
            timeout: 5000
        });
        return response.data;
    } catch (error) {
        // Try another instance on failure
        if (instances.length > 1) {
            const nextIndex = (index + 1) % instances.length;
            const fallbackInstance = instances[nextIndex];
            const fallbackUrl = `http://${fallbackInstance.address}:${fallbackInstance.port}${path}`;
            
            const response = await axios.get(fallbackUrl, {
                timeout: 5000
            });
            return response.data;
        }
        throw error;
    }
}

// API Gateway routes
app.get('/api/users/:id', async (req, res) => {
    try {
        const user = await callService('user-service', `/users/${req.params.id}`);
        res.json(user);
    } catch (error) {
        res.status(503).json({ error: 'User service unavailable' });
    }
});

app.get('/api/orders/:id', async (req, res) => {
    try {
        const order = await callService('order-service', `/orders/${req.params.id}`);
        res.json(order);
    } catch (error) {
        res.status(503).json({ error: 'Order service unavailable' });
    }
});

app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
});

// Cleanup on shutdown
process.on('SIGTERM', async () => {
    const serviceId = `api-gateway-${process.env.HOSTNAME}`;
    await consul.agent.service.deregister(serviceId);
    process.exit(0);
});

registerService().then(() => {
    app.listen(3000, () => {
        console.log('API Gateway listening on port 3000');
    });
});
```

## Service Registry Implementations

### Consul

HashiCorp Consul provides service discovery, health checking, and key-value storage. It uses a gossip protocol for cluster membership and Raft for consistency.

```python
# Python: Service registration with Consul
import consul
import socket
import os
import signal
import sys
from flask import Flask, jsonify

app = Flask(__name__)
consul_client = consul.Consul(host='consul-server', port=8500)

SERVICE_NAME = 'product-service'
SERVICE_ID = f"{SERVICE_NAME}-{socket.gethostname()}"
SERVICE_PORT = int(os.getenv('PORT', '5000'))
SERVICE_ADDRESS = socket.gethostbyname(socket.gethostname())

def register_service():
    """Register service with Consul"""
    consul_client.agent.service.register(
        name=SERVICE_NAME,
        service_id=SERVICE_ID,
        address=SERVICE_ADDRESS,
        port=SERVICE_PORT,
        tags=['python', 'flask', 'v1.0'],
        check=consul.Check.http(
            f'http://{SERVICE_ADDRESS}:{SERVICE_PORT}/health',
            interval='10s',
            timeout='5s',
            deregister='30s'
        ),
        meta={
            'version': '1.0.0',
            'environment': os.getenv('ENVIRONMENT', 'production')
        }
    )
    print(f'Registered {SERVICE_ID} with Consul')

def deregister_service():
    """Deregister service from Consul"""
    consul_client.agent.service.deregister(SERVICE_ID)
    print(f'Deregistered {SERVICE_ID} from Consul')

def discover_services(service_name, tag=None):
    """Discover healthy instances of a service"""
    _, services = consul_client.health.service(
        service_name,
        tag=tag,
        passing=True
    )
    
    instances = []
    for service in services:
        instances.append({
            'id': service['Service']['ID'],
            'address': service['Service']['Address'],
            'port': service['Service']['Port'],
            'tags': service['Service']['Tags'],
            'meta': service['Service']['Meta']
        })
    
    return instances

@app.route('/health')
def health():
    """Health check endpoint"""
    return jsonify({'status': 'healthy', 'service': SERVICE_NAME})

@app.route('/products/<product_id>')
def get_product(product_id):
    """Get product by ID"""
    return jsonify({
        'id': product_id,
        'name': f'Product {product_id}',
        'instance': SERVICE_ID
    })

@app.route('/discover/<service_name>')
def discover(service_name):
    """Discover instances of a service"""
    instances = discover_services(service_name)
    return jsonify({
        'service': service_name,
        'instances': instances
    })

def signal_handler(sig, frame):
    """Handle shutdown signals"""
    print('Shutting down...')
    deregister_service()
    sys.exit(0)

if __name__ == '__main__':
    # Register signal handlers
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)
    
    # Register with Consul
    register_service()
    
    # Start Flask app
    app.run(host='0.0.0.0', port=SERVICE_PORT)
```

### Netflix Eureka

Eureka is a REST-based service registry for resilient mid-tier load balancing and failover.

```yaml
# application.yml: Eureka Server configuration
server:
  port: 8761

eureka:
  instance:
    hostname: eureka-server
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 60000
    renewal-percent-threshold: 0.85
```

```java
// Java: Eureka Client configuration
@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

// application.yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
    registry-fetch-interval-seconds: 5
  instance:
    instance-id: ${spring.application.name}:${random.value}
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 10
    prefer-ip-address: true
    metadata-map:
      version: 1.0.0
      region: us-east-1

spring:
  application:
    name: user-service
```

### etcd

etcd is a distributed key-value store used by Kubernetes for service discovery and configuration.

```go
// Go: Service registration with etcd
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    
    clientv3 "go.etcd.io/etcd/client/v3"
)

type ServiceRegistry struct {
    client *clientv3.Client
    lease  clientv3.Lease
    leaseID clientv3.LeaseID
}

func NewServiceRegistry(endpoints []string) (*ServiceRegistry, error) {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout: 5 * time.Second,
    })
    if err != nil {
        return nil, err
    }
    
    return &ServiceRegistry{
        client: client,
        lease:  clientv3.NewLease(client),
    }, nil
}

func (r *ServiceRegistry) Register(ctx context.Context, serviceName, serviceAddr string, ttl int64) error {
    // Create a lease
    leaseResp, err := r.lease.Grant(ctx, ttl)
    if err != nil {
        return err
    }
    r.leaseID = leaseResp.ID
    
    // Register service with lease
    key := fmt.Sprintf("/services/%s/%s", serviceName, serviceAddr)
    _, err = r.client.Put(ctx, key, serviceAddr, clientv3.WithLease(r.leaseID))
    if err != nil {
        return err
    }
    
    // Keep lease alive
    keepAliveChan, err := r.lease.KeepAlive(ctx, r.leaseID)
    if err != nil {
        return err
    }
    
    // Process keep-alive responses
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case ka := <-keepAliveChan:
                if ka == nil {
                    log.Println("Keep-alive channel closed")
                    return
                }
            }
        }
    }()
    
    log.Printf("Registered service: %s at %s", serviceName, serviceAddr)
    return nil
}

func (r *ServiceRegistry) Discover(ctx context.Context, serviceName string) ([]string, error) {
    prefix := fmt.Sprintf("/services/%s/", serviceName)
    resp, err := r.client.Get(ctx, prefix, clientv3.WithPrefix())
    if err != nil {
        return nil, err
    }
    
    var addresses []string
    for _, kv := range resp.Kvs {
        addresses = append(addresses, string(kv.Value))
    }
    
    return addresses, nil
}

func (r *ServiceRegistry) Watch(ctx context.Context, serviceName string) clientv3.WatchChan {
    prefix := fmt.Sprintf("/services/%s/", serviceName)
    return r.client.Watch(ctx, prefix, clientv3.WithPrefix())
}

func (r *ServiceRegistry) Deregister(ctx context.Context, serviceName, serviceAddr string) error {
    key := fmt.Sprintf("/services/%s/%s", serviceName, serviceAddr)
    _, err := r.client.Delete(ctx, key)
    if err != nil {
        return err
    }
    
    // Revoke lease
    _, err = r.lease.Revoke(ctx, r.leaseID)
    return err
}

func (r *ServiceRegistry) Close() error {
    return r.client.Close()
}

// Usage example
func main() {
    ctx := context.Background()
    
    registry, err := NewServiceRegistry([]string{"localhost:2379"})
    if err != nil {
        log.Fatal(err)
    }
    defer registry.Close()
    
    // Register service
    err = registry.Register(ctx, "user-service", "192.168.1.10:8080", 10)
    if err != nil {
        log.Fatal(err)
    }
    
    // Discover services
    addresses, err := registry.Discover(ctx, "user-service")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Discovered services:", addresses)
    
    // Watch for changes
    watchChan := registry.Watch(ctx, "user-service")
    go func() {
        for watchResp := range watchChan {
            for _, event := range watchResp.Events {
                fmt.Printf("Event: %s %s %s\n", 
                    event.Type, event.Kv.Key, event.Kv.Value)
            }
        }
    }()
    
    // Keep running
    select {}
}
```

## Health Checking

Service discovery systems continuously check service health to route traffic only to healthy instances.

```javascript
// Node.js: Advanced health checks
const express = require('express');
const app = express();

let isHealthy = true;
let isReady = true;

// Database connection check
async function checkDatabase() {
    try {
        await db.query('SELECT 1');
        return true;
    } catch (error) {
        console.error('Database check failed:', error);
        return false;
    }
}

// External service check
async function checkExternalServices() {
    try {
        const services = ['user-service', 'order-service'];
        const checks = services.map(async service => {
            const instances = await discoverService(service);
            return instances.length > 0;
        });
        
        const results = await Promise.all(checks);
        return results.every(r => r);
    } catch (error) {
        console.error('External service check failed:', error);
        return false;
    }
}

// Liveness probe - is the service running?
app.get('/health/live', (req, res) => {
    if (isHealthy) {
        res.status(200).json({ status: 'alive' });
    } else {
        res.status(503).json({ status: 'unhealthy' });
    }
});

// Readiness probe - is the service ready to receive traffic?
app.get('/health/ready', async (req, res) => {
    const dbHealthy = await checkDatabase();
    const servicesHealthy = await checkExternalServices();
    
    isReady = dbHealthy && servicesHealthy;
    
    if (isReady) {
        res.status(200).json({
            status: 'ready',
            checks: {
                database: dbHealthy,
                externalServices: servicesHealthy
            }
        });
    } else {
        res.status(503).json({
            status: 'not ready',
            checks: {
                database: dbHealthy,
                externalServices: servicesHealthy
            }
        });
    }
});

// Detailed health check
app.get('/health', async (req, res) => {
    const startTime = Date.now();
    
    const checks = {
        database: await checkDatabase(),
        externalServices: await checkExternalServices(),
        memory: process.memoryUsage().heapUsed < 500 * 1024 * 1024, // < 500MB
        uptime: process.uptime() > 10 // Running for at least 10 seconds
    };
    
    const allHealthy = Object.values(checks).every(c => c);
    const responseTime = Date.now() - startTime;
    
    res.status(allHealthy ? 200 : 503).json({
        status: allHealthy ? 'healthy' : 'unhealthy',
        timestamp: new Date().toISOString(),
        checks,
        metrics: {
            responseTime: `${responseTime}ms`,
            uptime: `${Math.floor(process.uptime())}s`,
            memory: `${Math.floor(process.memoryUsage().heapUsed / 1024 / 1024)}MB`
        }
    });
});

// Graceful shutdown
process.on('SIGTERM', () => {
    console.log('SIGTERM received, starting graceful shutdown');
    isHealthy = false;
    isReady = false;
    
    // Wait for existing requests to complete
    setTimeout(() => {
        process.exit(0);
    }, 30000); // 30 seconds grace period
});
```

## DNS-Based Service Discovery

Kubernetes uses DNS for service discovery. Services are accessible via DNS names like `service-name.namespace.svc.cluster.local`.

```yaml
# Kubernetes Service definition
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
spec:
  selector:
    app: user-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      containers:
        - name: user-service
          image: user-service:v1
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

## Best Practices

✅ **Health Checks**: Implement both liveness and readiness probes. Liveness detects crashes, readiness detects overload.

✅ **Graceful Shutdown**: Deregister from service registry before shutting down. Allow time for in-flight requests to complete.

✅ **Service Metadata**: Include version, environment, and region in service metadata for advanced routing.

✅ **Heartbeats**: Send regular heartbeats to service registry to maintain registration.

✅ **Cache Service Lists**: Cache discovered service lists with TTL to reduce registry load.

✅ **Retry Logic**: Implement retries with exponential backoff when service discovery fails.

✅ **Load Balancing**: Implement client-side load balancing with service discovery for better distribution.

✅ **Monitor Registry**: Monitor service registry health and availability. It's critical infrastructure.

## Anti-Patterns

❌ **Hard-Coded Service URLs**: Never hard-code service addresses. Always use service discovery.

❌ **No Health Checks**: Services without health checks can send traffic to unhealthy instances.

❌ **Ignoring Deregistration**: Not deregistering on shutdown leaves stale entries in the registry.

❌ **Single Registry Instance**: Service registry should be highly available. Use clusters, not single instances.

❌ **Caching Forever**: Don't cache discovered services indefinitely. Use reasonable TTLs.

❌ **No Fallbacks**: Always have fallback logic when service discovery fails.
