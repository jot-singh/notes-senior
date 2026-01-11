# NoSQL Databases

Comprehensive notes on NoSQL databases with focus on highly available and scalable systems.

## Overview

NoSQL databases provide horizontal scalability, flexible schemas, and high availability for modern distributed applications. These notes cover fundamental concepts, specific database implementations, and production best practices.

## Contents

### 1. [NoSQL Fundamentals](01-nosql-fundamentals.md)
- NoSQL categories (Document, Key-Value, Column-Family, Graph)
- CAP and PACELC theorems
- Consistency models (Strong, Eventual, Causal)
- When to choose NoSQL vs SQL
- Data modeling strategies
- Real-world architecture patterns

**Key Topics**: CAP theorem, consistency levels, denormalization, query-driven design

### 2. [Apache Cassandra](02-cassandra.md)
- Peer-to-peer ring architecture with no single point of failure
- Partitioning, replication, and tunable consistency
- Data modeling for write-heavy workloads
- Time-series data patterns with bucketing
- Production deployment and hardware recommendations
- Performance optimization (prepared statements, async queries)
- Netflix use case: 200K+ nodes, petabytes of data

**Key Topics**: Virtual nodes, consistency levels, wide-column model, compaction strategies

### 3. [MongoDB](03-mongodb.md)
- Replica sets for high availability with automatic failover
- Sharding for horizontal scalability
- Document model with flexible schemas
- ACID transactions across multiple documents
- Aggregation framework for complex queries
- Index strategies and query optimization
- eBay use case: 1B+ product listings

**Key Topics**: Replica sets, sharding strategies, aggregation pipeline, read/write concerns

## Quick Comparison

| Feature | Cassandra | MongoDB |
|---------|-----------|---------|
| **Type** | Column-Family | Document Store |
| **CAP** | AP (Availability + Partition Tolerance) | CP (Consistency + Partition Tolerance) |
| **Consistency** | Tunable per operation | Strong with replica sets |
| **Transactions** | Lightweight transactions | Multi-document ACID |
| **Best For** | Time-series, write-heavy | Flexible schemas, complex queries |
| **Scaling** | Linear horizontal scaling | Horizontal sharding |
| **Query Language** | CQL (SQL-like) | MongoDB Query Language (JSON-like) |
| **Use Cases** | IoT, metrics, logs | E-commerce, content management |

## When to Use Each Database

### Use Cassandra When:
- Write-heavy workloads (millions of writes/second)
- Time-series data (metrics, IoT sensor data, logs)
- Need 99.999% availability with no single point of failure
- Geographic distribution across multiple datacenters
- Linear scalability is critical
- Example: Netflix viewing history, Apple iCloud

### Use MongoDB When:
- Flexible, evolving schemas
- Complex document structures with embedded data
- Need ACID transactions
- Rich query capabilities (aggregations, joins)
- Strong consistency requirements
- Rapid application development
- Example: eBay product catalog, Adobe user profiles

### Use Both When:
- Different use cases in same application
- MongoDB for transactional data, Cassandra for analytics
- MongoDB for user profiles, Cassandra for activity logs
- Example: E-commerce with MongoDB for orders, Cassandra for clickstream

## Common Patterns

### Time-Series Data
```javascript
// Cassandra: Partition by sensor + time bucket
CREATE TABLE sensor_data (
    sensor_id UUID,
    bucket text,          -- 'YYYY-MM-DD-HH'
    timestamp timestamp,
    temperature double,
    PRIMARY KEY ((sensor_id, bucket), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

// MongoDB: Bucketed documents with TTL
{
  _id: "sensor123:2024-01-10:10",
  sensor_id: "sensor123",
  hour: "2024-01-10:10",
  readings: [
    { timestamp: ISODate("..."), temp: 23.5 },
    // Up to 60 readings per hour
  ]
}
```

### Denormalization
```javascript
// Both databases favor denormalization for performance

// Cassandra: Multiple tables per query pattern
CREATE TABLE orders_by_user (...);
CREATE TABLE orders_by_status (...);
CREATE TABLE orders_by_date (...);

// MongoDB: Embedded documents
{
  order_id: "ORD-123",
  customer: {
    id: 12345,
    name: "John Doe",  // Denormalized
    email: "john@example.com"
  },
  items: [...]
}
```

### High Availability
```javascript
// Cassandra: Replication factor across datacenters
CREATE KEYSPACE production
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'us_east': 3,
    'us_west': 3,
    'eu_west': 2
};

// MongoDB: Replica set across availability zones
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-us-east-1a" },
    { _id: 1, host: "mongo-us-east-1b" },
    { _id: 2, host: "mongo-us-east-1c" }
  ]
});
```

## Performance Benchmarks

**Cassandra**:
- Writes: 1M+ ops/sec per cluster
- Reads: Sub-10ms p99 latency (local DC)
- Scalability: Linear with nodes
- Best: Time-series writes, append-only logs

**MongoDB**:
- Writes: 100K+ ops/sec per replica set
- Reads: Sub-5ms p99 with proper indexes
- Scalability: Horizontal with sharding
- Best: Complex queries, transactions

## Learning Path

1. **Start**: [NoSQL Fundamentals](01-nosql-fundamentals.md) - Understand CAP theorem, consistency models
2. **Deep Dive**: Choose based on use case:
   - Write-heavy, time-series → [Cassandra](02-cassandra.md)
   - Flexible schema, transactions → [MongoDB](03-mongodb.md)
3. **Practice**: Set up local clusters, model your data, run benchmarks
4. **Production**: Study operational best practices, monitoring, and scaling

## Additional Resources

### Cassandra
- [DataStax Academy](https://academy.datastax.com/) - Free courses
- [Cassandra Summit](https://www.cassandrasummit.org/) - Annual conference
- Tools: nodetool, cqlsh, DataStax DevCenter

### MongoDB
- [MongoDB University](https://university.mongodb.com/) - Free courses
- [MongoDB World](https://www.mongodb.com/world) - Annual conference
- Tools: mongosh, MongoDB Compass, MongoDB Atlas

### Books
- *Cassandra: The Definitive Guide* by Jeff Carpenter
- *MongoDB: The Definitive Guide* by Shannon Bradshaw
- *Designing Data-Intensive Applications* by Martin Kleppmann

---

**Related Topics**:
- [Microservices](../microservices/) - Service data patterns
- [System Design](../system-design/) - Distributed systems architecture
- [DBMS](../dbms/) - Relational database concepts for comparison
