# System Design (High-Level Design)

Comprehensive notes on distributed systems, scalability patterns, and production architectures used at scale.

## Overview

System design encompasses the architecture, components, and trade-offs involved in building large-scale distributed systems. These notes cover fundamental concepts, proven patterns, and real-world implementations from companies like Google, Amazon, Netflix, and Facebook.

## Topics Covered

### 1. [Load Balancing](01-load-balancing.md)
- Load balancing algorithms (Round Robin, Least Connections, Consistent Hashing)
- Health checking mechanisms
- Layer 4 vs Layer 7 load balancing
- AWS ELB, nginx, HAProxy implementations
- Session persistence and sticky sessions

### 2. [Caching Strategies](02-caching-strategies.md)
- Cache-aside, write-through, write-behind patterns
- Cache eviction policies (LRU, LFU, TTL)
- Multi-level caching architectures
- Redis, Memcached implementations
- Cache invalidation strategies

### 3. [Database Scaling Strategies](03-database-scaling-strategies.md)
- Vertical vs horizontal scaling
- Read replicas and replication strategies
- Database sharding (hash-based, range-based, geographic)
- Consistent hashing for sharding
- Partitioning strategies
- Real-world case studies (Instagram, Pinterest, Uber)

### 4. [CDN and Content Delivery](04-cdn-content-delivery.md)
- CDN architecture and request flow
- Edge computing and Lambda@Edge
- Cache control headers and invalidation
- Image optimization at the edge
- CloudFlare, CloudFront, Fastly comparison

### 5. [CAP Theorem](05-cap-theorem.md)
- Consistency, Availability, Partition Tolerance trade-offs
- CP systems (MongoDB, HBase, Spanner)
- AP systems (Cassandra, DynamoDB, Riak)
- Eventual consistency patterns
- Vector clocks and CRDTs
- Real-world trade-off decisions

### 6. [Message Queues and Asynchronous Processing](06-message-queues.md)
- Message queue fundamentals
- Delivery guarantees (at-most-once, at-least-once, exactly-once)
- RabbitMQ: exchanges, queues, bindings
- Apache Kafka: topics, partitions, consumer groups
- AWS SQS: standard vs FIFO queues
- Dead letter queues and error handling

### 7. [Rate Limiting](07-rate-limiting.md)
- Rate limiting algorithms and strategies
- Token bucket, leaky bucket, fixed window
- Distributed rate limiting
- API throttling and quota management

### 8. [API Gateway](08-api-gateway.md)
- API gateway patterns and responsibilities
- Request routing and load balancing
- Authentication and authorization
- Rate limiting and caching

### 9. [Service Discovery](09-service-discovery.md)
- Service registry patterns
- Client-side vs server-side discovery
- Health checking and heartbeats
- Consul, Eureka, etcd implementations

### 10. [System Design Interview Questions](10-system-design-interview-questions.md)
- Common interview patterns
- Design Twitter, Instagram, URL shortener
- Capacity estimation techniques
- Trade-off analysis

### 11. [Distributed Systems Fundamentals](11-distributed-systems-fundamentals.md)
- Consistency models (strong, eventual, causal)
- Consensus algorithms (Paxos, Raft)
- Logical clocks (Lamport, Vector clocks)
- Quorum-based replication
- Gossip protocols
- Eight fallacies of distributed computing

### 12. [High Availability and Fault Tolerance](12-high-availability-fault-tolerance.md)
- Availability metrics and the nines
- SLA, SLO, SLI definitions
- Redundancy patterns (active-active, active-passive)
- Circuit breakers and bulkhead patterns
- Disaster recovery and backup strategies
- Chaos engineering principles

### 13. [Monitoring and Observability](13-monitoring-observability.md)
- Three pillars: metrics, logs, traces
- Distributed tracing implementation
- SLI/SLO-based alerting
- Context propagation and correlation IDs
- Observability tools (Prometheus, Grafana, Jaeger)

### 14. [Security in Distributed Systems](14-security-distributed-systems.md)
- Authentication patterns (JWT, OAuth 2.0, API keys)
- Authorization (RBAC, ABAC)
- Encryption in transit and at rest
- DDoS protection and rate limiting
- Security best practices and anti-patterns

### 15. [Distributed Transactions](15-distributed-transactions.md)
- Two-phase commit (2PC) protocol
- Saga pattern (choreography vs orchestration)
- Try-Cancel/Confirm (TCC) pattern
- Compensation and rollback strategies
- XA transactions and distributed coordinators

### 16. [CQRS and Event Sourcing](16-cqrs-event-sourcing.md)
- Command Query Responsibility Segregation
- Event Store implementation
- Event sourcing fundamentals
- Projections and read models
- Snapshotting and event versioning

### 17. [Back-of-Envelope Calculations](17-back-of-envelope-calculations.md)
- Capacity estimation techniques
- QPS and throughput calculations
- Storage and bandwidth requirements
- Latency numbers every programmer should know
- Real-world examples (Twitter, URL shortener)

## Core Concepts

### Scalability
- **Horizontal Scaling**: Adding more servers to distribute load
- **Vertical Scaling**: Adding more resources (CPU, RAM) to existing servers
- **Elastic Scaling**: Automatically scaling based on demand

### Reliability
- **Fault Tolerance**: System continues operating despite component failures
- **Redundancy**: Multiple copies of components for backup
- **Graceful Degradation**: Reduced functionality instead of complete failure

### Availability
- **High Availability**: System operational and accessible (99.9% uptime)
- **SLA (Service Level Agreement)**: Guaranteed uptime commitments
- **Disaster Recovery**: Plans and systems for catastrophic failures

### Performance
- **Latency**: Time to process a single request
- **Throughput**: Number of requests processed per time unit
- **Response Time**: Latency + queuing + processing time

### Consistency
- **Strong Consistency**: All nodes see the same data simultaneously
- **Eventual Consistency**: Nodes converge to same state eventually
- **Read-Your-Writes**: Users see their own writes immediately

## Design Patterns

### Circuit Breaker
Prevents cascading failures by failing fast when downstream services are unhealthy.

### Bulkhead
Isolates resources to prevent failures from affecting entire system.

### Rate Limiting
Controls request rate to protect services from overload.

### Retry with Exponential Backoff
Automatically retries failed requests with increasing delays.

### Saga Pattern
Manages distributed transactions through compensating actions.

## Learning Path

1. **Fundamentals**: Start with load balancing and caching strategies to understand traffic distribution and performance optimization.

2. **Data Layer**: Study database scaling strategies and CAP theorem to understand data distribution and consistency trade-offs.

3. **Async Communication**: Learn message queues to decouple services and handle asynchronous workflows.

4. **Content Delivery**: Explore CDN concepts for global content distribution and edge computing.

5. **Integration**: Combine these patterns to design complete systems (e.g., design Twitter, design URL shortener).

## Real-World Examples

### Netflix
- **CDN**: Open Connect servers in ISP data centers
- **Chaos Engineering**: Simian Army for fault injection
- **Microservices**: Hundreds of services with API gateway
- **Caching**: Multi-level caching (EVCache)

### Amazon
- **DynamoDB**: AP system with eventual consistency
- **SQS**: Managed message queue for async processing
- **CloudFront**: Global CDN with Lambda@Edge
- **Auto Scaling**: Dynamic resource allocation

### Twitter
- **Timeline Service**: Fanout-on-write for fast reads
- **Manhattan**: Distributed key-value store
- **Firehose**: Real-time event streaming
- **Caching**: Extensive use of Redis

### Uber
- **Ringpop**: Consistent hashing for request routing
- **Schemaless**: Sharded MySQL with NoSQL interface
- **DISCO**: Service discovery and health checking
- **Kafka**: Event streaming for real-time data

## Key Metrics

### Availability Metrics
- **99.9% (three nines)**: 8.76 hours downtime per year
- **99.99% (four nines)**: 52.56 minutes downtime per year
- **99.999% (five nines)**: 5.26 minutes downtime per year

### Performance Metrics
- **P50 Latency**: 50th percentile response time
- **P95 Latency**: 95th percentile response time
- **P99 Latency**: 99th percentile response time (tail latency)

### Throughput Metrics
- **RPS**: Requests per second
- **QPS**: Queries per second
- **TPS**: Transactions per second

## Interview Preparation

System design interviews typically cover:
- Designing scalable systems (Twitter, Instagram, Netflix)
- Understanding trade-offs (CAP theorem, consistency vs availability)
- Capacity estimation and back-of-the-envelope calculations
- Discussing real-world constraints and bottlenecks

## Further Reading

- *Designing Data-Intensive Applications* by Martin Kleppmann
- *System Design Interview* by Alex Xu
- AWS Architecture Blog
- Google Cloud Architecture Center
- Netflix Tech Blog
- High Scalability Blog
