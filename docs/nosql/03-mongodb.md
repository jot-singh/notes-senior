# MongoDB: Document Database for High Availability and Scale

## What You'll Learn

- MongoDB's architecture and replication mechanisms
- Sharding strategies for horizontal scalability
- Document modeling and indexing best practices
- Transactions and consistency guarantees
- Production deployment patterns and performance optimization

## Why This Matters

MongoDB powers applications at Fortune 500 companies including eBay (50+ billion documents), Adobe (40+ TB), and Bosch (IoT data). It combines the flexibility of JSON documents with ACID transactions, making it suitable for both rapid development and mission-critical systems. MongoDB's replica sets provide automatic failover with 99.995% availability, while sharding enables linear scalability to petabytes of data. Understanding MongoDB is essential for building modern applications that require flexible schemas, horizontal scaling, and strong consistency guarantees.

## Architecture Overview

### Replica Sets

A replica set is a group of MongoDB servers that maintain the same dataset, providing redundancy and high availability.

```mermaid
graph TB
    subgraph "Replica Set"
        PRIMARY[Primary<br/>Read/Write]
        SECONDARY1[Secondary 1<br/>Read-only]
        SECONDARY2[Secondary 2<br/>Read-only]
        ARBITER[Arbiter<br/>Voting only]
    end
    
    CLIENT[Application] -->|Write| PRIMARY
    CLIENT -->|Read| PRIMARY
    CLIENT -.->|Read<br/>readPreference| SECONDARY1
    CLIENT -.->|Read<br/>readPreference| SECONDARY2
    
    PRIMARY -->|Replication| SECONDARY1
    PRIMARY -->|Replication| SECONDARY2
    
    PRIMARY -.->|Heartbeat| ARBITER
    SECONDARY1 -.->|Heartbeat| ARBITER
    SECONDARY2 -.->|Heartbeat| ARBITER
    
    style PRIMARY fill:#4CAF50
    style SECONDARY1 fill:#2196F3
    style SECONDARY2 fill:#2196F3
    style ARBITER fill:#FFC107
```

**Components**:
- **Primary**: Accepts all writes, replicates to secondaries
- **Secondary**: Maintains copy of data, can serve reads
- **Arbiter**: Participates in elections but doesn't store data

**Automatic Failover**:
```mermaid
sequenceDiagram
    participant App
    participant Primary
    participant Secondary1
    participant Secondary2
    
    App->>Primary: Write operations
    Primary->>Secondary1: Replicate
    Primary->>Secondary2: Replicate
    
    Note over Primary: Primary fails
    
    Secondary1->>Secondary2: Election vote
    Secondary2->>Secondary1: Vote granted
    
    Note over Secondary1: Becomes new Primary
    
    App->>Secondary1: Write operations
    Secondary1->>Secondary2: Replicate
```

Election happens in ~12 seconds. New primary elected by majority vote.

### Sharded Cluster

Sharding distributes data across multiple replica sets for horizontal scalability.

```mermaid
graph TB
    subgraph "Application Tier"
        APP1[App Server 1]
        APP2[App Server 2]
        APP3[App Server 3]
    end
    
    subgraph "Query Routers"
        MONGOS1[mongos 1]
        MONGOS2[mongos 2]
    end
    
    subgraph "Config Servers"
        CONFIG1[Config 1]
        CONFIG2[Config 2]
        CONFIG3[Config 3]
    end
    
    subgraph "Shard 1"
        S1P[Primary]
        S1S1[Secondary]
        S1S2[Secondary]
    end
    
    subgraph "Shard 2"
        S2P[Primary]
        S2S1[Secondary]
        S2S2[Secondary]
    end
    
    subgraph "Shard 3"
        S3P[Primary]
        S3S1[Secondary]
        S3S2[Secondary]
    end
    
    APP1 --> MONGOS1
    APP2 --> MONGOS2
    APP3 --> MONGOS1
    
    MONGOS1 --> CONFIG1
    MONGOS2 --> CONFIG1
    
    CONFIG1 --> CONFIG2
    CONFIG2 --> CONFIG3
    
    MONGOS1 --> S1P
    MONGOS1 --> S2P
    MONGOS1 --> S3P
    
    S1P --> S1S1
    S1P --> S1S2
    
    S2P --> S2S1
    S2P --> S2S2
    
    S3P --> S3S1
    S3P --> S3S2
```

**Components**:
- **mongos**: Query router directing operations to appropriate shards
- **Config Servers**: Store cluster metadata and configuration
- **Shards**: Replica sets storing subset of data

## Data Model

### Document Structure

MongoDB stores data as BSON (Binary JSON) documents with flexible schemas.

```javascript
// User document with embedded addresses and preferences
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "user_id": 12345,
  "username": "johndoe",
  "email": "john@example.com",
  "profile": {
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": ISODate("1990-05-15"),
    "avatar_url": "https://cdn.example.com/avatars/12345.jpg"
  },
  "addresses": [
    {
      "type": "home",
      "street": "123 Main St",
      "city": "San Francisco",
      "state": "CA",
      "zip": "94102",
      "country": "US",
      "is_default": true
    },
    {
      "type": "work",
      "street": "456 Market St",
      "city": "San Francisco",
      "state": "CA",
      "zip": "94103",
      "country": "US",
      "is_default": false
    }
  ],
  "preferences": {
    "newsletter": true,
    "notifications": {
      "email": true,
      "sms": false,
      "push": true
    },
    "language": "en",
    "timezone": "America/Los_Angeles"
  },
  "roles": ["customer", "premium"],
  "created_at": ISODate("2023-01-15T10:30:00Z"),
  "updated_at": ISODate("2024-01-10T14:22:00Z"),
  "last_login": ISODate("2024-01-10T14:22:00Z"),
  "login_count": 342
}
```

### Document Relationships

MongoDB supports three patterns for modeling relationships.

#### Embedded Documents (One-to-Few)

Embed related data when relationship is one-to-few and data is frequently accessed together.

```javascript
// ✅ Good: Blog post with comments (< 100 comments typical)
{
  "_id": ObjectId("..."),
  "title": "MongoDB Best Practices",
  "author": "John Doe",
  "content": "...",
  "comments": [
    {
      "user": "Jane Smith",
      "text": "Great article!",
      "date": ISODate("2024-01-10T10:00:00Z"),
      "likes": 5
    },
    {
      "user": "Bob Johnson",
      "text": "Very helpful, thanks!",
      "date": ISODate("2024-01-10T11:30:00Z"),
      "likes": 3
    }
  ],
  "tags": ["mongodb", "nosql", "databases"],
  "published_at": ISODate("2024-01-10T09:00:00Z")
}
```

**Benefits**: Single query retrieves all data. Atomic updates.  
**Limitations**: 16MB document size limit. Array growth makes updates slower.

#### References (One-to-Many)

Use references when relationship is one-to-many or data grows unbounded.

```javascript
// Orders collection
{
  "_id": ObjectId("..."),
  "order_number": "ORD-2024-00123",
  "customer_id": ObjectId("507f1f77bcf86cd799439011"),  // Reference
  "items": [
    {
      "product_id": ObjectId("..."),  // Reference
      "quantity": 2,
      "price": 29.99
    }
  ],
  "total": 59.98,
  "status": "shipped",
  "created_at": ISODate("2024-01-10T10:00:00Z")
}

// Query with population (application-level join)
const order = await db.orders.findOne({ order_number: "ORD-2024-00123" });
const customer = await db.users.findOne({ _id: order.customer_id });
const productIds = order.items.map(item => item.product_id);
const products = await db.products.find({ _id: { $in: productIds } }).toArray();
```

#### Extended Reference (Denormalization)

Embed frequently accessed fields while maintaining reference for complete data.

```javascript
// ✅ Best practice: Extended reference with denormalization
{
  "_id": ObjectId("..."),
  "order_number": "ORD-2024-00123",
  // Denormalized customer data (frequently accessed)
  "customer": {
    "id": ObjectId("507f1f77bcf86cd799439011"),
    "name": "John Doe",
    "email": "john@example.com"
  },
  "items": [
    {
      "product_id": ObjectId("..."),
      "name": "Wireless Mouse",  // Denormalized
      "price": 29.99,
      "quantity": 2
    }
  ],
  "total": 59.98,
  "status": "shipped"
}

// Most queries need only order data (single query)
// Full customer details fetched only when needed
```

## CRUD Operations

### Insert

```javascript
// Insert single document
const result = await db.users.insertOne({
  username: "johndoe",
  email: "john@example.com",
  created_at: new Date()
});
console.log(result.insertedId);

// Insert multiple documents
const results = await db.users.insertMany([
  { username: "jane", email: "jane@example.com" },
  { username: "bob", email: "bob@example.com" }
], { ordered: false });  // Continue on error
console.log(results.insertedIds);
```

### Query

```javascript
// Find all documents
const users = await db.users.find({}).toArray();

// Find with filter
const activeUsers = await db.users.find({
  status: "active",
  created_at: { $gte: new Date("2024-01-01") }
}).toArray();

// Find with projection (select specific fields)
const userEmails = await db.users.find(
  { status: "active" },
  { projection: { email: 1, username: 1, _id: 0 } }
).toArray();

// Find one
const user = await db.users.findOne({ username: "johndoe" });

// Complex query with operators
const results = await db.products.find({
  $and: [
    { price: { $gte: 100, $lte: 500 } },
    { category: { $in: ["electronics", "computers"] } },
    { stock: { $gt: 0 } }
  ]
}).sort({ price: -1 }).limit(20).toArray();

// Query embedded documents
const orders = await db.orders.find({
  "customer.email": "john@example.com",
  "items.quantity": { $gte: 5 }
}).toArray();

// Query array elements
const posts = await db.posts.find({
  tags: { $all: ["mongodb", "nosql"] }  // Contains both tags
}).toArray();
```

### Update

```javascript
// Update single document
await db.users.updateOne(
  { username: "johndoe" },
  { 
    $set: { 
      email: "newemail@example.com",
      updated_at: new Date()
    },
    $inc: { login_count: 1 }
  }
);

// Update multiple documents
await db.users.updateMany(
  { status: "inactive", last_login: { $lt: new Date("2023-01-01") } },
  { $set: { status: "archived" } }
);

// Update with operators
await db.products.updateOne(
  { _id: productId },
  {
    $inc: { stock: -1, sold_count: 1 },  // Decrement stock, increment sold
    $push: { 
      purchase_history: {
        user_id: userId,
        date: new Date(),
        quantity: 1
      }
    },
    $set: { last_purchased: new Date() }
  }
);

// Update array elements
await db.posts.updateOne(
  { _id: postId, "comments.user_id": userId },
  { $set: { "comments.$.edited": true } }  // $ positional operator
);

// Upsert (update or insert)
await db.users.updateOne(
  { username: "newuser" },
  { 
    $set: { email: "new@example.com" },
    $setOnInsert: { created_at: new Date() }  // Only on insert
  },
  { upsert: true }
);

// Replace entire document
await db.users.replaceOne(
  { _id: userId },
  { 
    username: "johndoe",
    email: "john@example.com",
    created_at: new Date()
  }
);
```

### Delete

```javascript
// Delete single document
await db.users.deleteOne({ username: "johndoe" });

// Delete multiple documents
await db.users.deleteMany({ 
  status: "archived",
  created_at: { $lt: new Date("2020-01-01") }
});

// Find and delete (returns deleted document)
const deletedUser = await db.users.findOneAndDelete(
  { username: "johndoe" }
);
```

## Aggregation Framework

MongoDB's aggregation pipeline processes documents through stages.

```javascript
// Basic aggregation: Group and count
const stats = await db.orders.aggregate([
  { $match: { status: "completed" } },
  { 
    $group: {
      _id: "$customer_id",
      total_orders: { $sum: 1 },
      total_spent: { $sum: "$total" },
      avg_order_value: { $avg: "$total" }
    }
  },
  { $sort: { total_spent: -1 } },
  { $limit: 10 }
]).toArray();

// Complex pipeline: Join-like operation with $lookup
const ordersWithCustomers = await db.orders.aggregate([
  {
    $match: {
      created_at: { $gte: new Date("2024-01-01") }
    }
  },
  {
    $lookup: {
      from: "users",
      localField: "customer_id",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },  // Convert array to object
  {
    $project: {
      order_number: 1,
      total: 1,
      "customer.name": 1,
      "customer.email": 1
    }
  }
]).toArray();

// Time-series aggregation: Daily sales report
const dailySales = await db.orders.aggregate([
  {
    $match: {
      created_at: { 
        $gte: new Date("2024-01-01"),
        $lt: new Date("2024-02-01")
      }
    }
  },
  {
    $group: {
      _id: { 
        $dateToString: { format: "%Y-%m-%d", date: "$created_at" }
      },
      order_count: { $sum: 1 },
      total_revenue: { $sum: "$total" },
      avg_order_value: { $avg: "$total" },
      unique_customers: { $addToSet: "$customer_id" }
    }
  },
  {
    $project: {
      date: "$_id",
      order_count: 1,
      total_revenue: 1,
      avg_order_value: 1,
      unique_customers: { $size: "$unique_customers" }
    }
  },
  { $sort: { date: 1 } }
]).toArray();

// Faceted search: Multiple aggregations in one query
const analytics = await db.products.aggregate([
  {
    $facet: {
      "by_category": [
        { $group: { _id: "$category", count: { $sum: 1 } } },
        { $sort: { count: -1 } }
      ],
      "price_ranges": [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 50, 100, 500, 1000, 5000],
            default: "5000+",
            output: { count: { $sum: 1 }, avg_price: { $avg: "$price" } }
          }
        }
      ],
      "top_sellers": [
        { $sort: { sold_count: -1 } },
        { $limit: 10 },
        { $project: { name: 1, sold_count: 1, price: 1 } }
      ]
    }
  }
]).toArray();
```

### Aggregation Pipeline Operators

```javascript
// $unwind: Deconstruct array field
await db.orders.aggregate([
  { $unwind: "$items" },  // One document per item
  {
    $group: {
      _id: "$items.product_id",
      total_quantity: { $sum: "$items.quantity" },
      total_revenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
    }
  }
]);

// $addFields: Add computed fields
await db.products.aggregate([
  {
    $addFields: {
      discount_price: { $multiply: ["$price", 0.9] },
      in_stock: { $gt: ["$stock", 0] }
    }
  }
]);

// $replaceRoot: Promote embedded document
await db.orders.aggregate([
  { $unwind: "$items" },
  { $replaceRoot: { newRoot: "$items" } }  // Items become top-level documents
]);
```

## Indexing

### Index Types

```javascript
// Single field index
await db.users.createIndex({ email: 1 });  // 1 = ascending, -1 = descending

// Compound index (order matters!)
await db.orders.createIndex({ customer_id: 1, created_at: -1 });

// Multikey index (array fields)
await db.products.createIndex({ tags: 1 });

// Text index (full-text search)
await db.articles.createIndex({ 
  title: "text", 
  content: "text" 
}, { 
  weights: { title: 10, content: 1 }  // Title 10x more important
});

// Query text index
const results = await db.articles.find({
  $text: { $search: "mongodb sharding performance" }
}, {
  score: { $meta: "textScore" }
}).sort({ score: { $meta: "textScore" } }).toArray();

// Geospatial index (2dsphere for Earth-like sphere)
await db.stores.createIndex({ location: "2dsphere" });

// Query nearby locations
const nearbyStores = await db.stores.find({
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-122.4194, 37.7749]  // [longitude, latitude]
      },
      $maxDistance: 5000  // meters
    }
  }
}).toArray();

// Unique index
await db.users.createIndex({ email: 1 }, { unique: true });

// Partial index (index subset of documents)
await db.orders.createIndex(
  { customer_id: 1, created_at: -1 },
  { 
    partialFilterExpression: { 
      status: { $in: ["pending", "processing"] } 
    }
  }
);

// TTL index (auto-delete expired documents)
await db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }  // Delete after 1 hour
);

// Wildcard index (index all fields)
await db.products.createIndex({ "$**": 1 });  // Use sparingly!
```

### Index Strategies

```javascript
// Compound index: ESR rule (Equality, Sort, Range)
// ✅ Good: Equality first, then sort, then range
await db.orders.createIndex({ 
  customer_id: 1,    // Equality
  created_at: -1,    // Sort
  total: 1           // Range
});

// Query efficiently uses this index
await db.orders.find({
  customer_id: customerId,        // Equality
  total: { $gte: 100 }            // Range
}).sort({ created_at: -1 });      // Sort

// Covered queries: Index contains all fields
await db.users.createIndex({ email: 1, name: 1, status: 1 });

// This query is "covered" (no document fetch needed)
await db.users.find(
  { email: "john@example.com" },
  { projection: { email: 1, name: 1, status: 1, _id: 0 } }
);
```

### Index Analysis

```javascript
// Explain query execution
const explanation = await db.orders.find({
  customer_id: customerId,
  created_at: { $gte: new Date("2024-01-01") }
}).explain("executionStats");

console.log(explanation.executionStats);
// totalDocsExamined: Number of documents scanned
// totalKeysExamined: Number of index keys scanned
// executionTimeMillis: Query execution time
// stage: IXSCAN (index scan) vs COLLSCAN (collection scan)

// ✅ Good query (using index)
// totalDocsExamined: 100
// totalKeysExamined: 100
// stage: "IXSCAN"

// ❌ Bad query (no index)
// totalDocsExamined: 1000000
// totalKeysExamined: 0
// stage: "COLLSCAN"

// View all indexes
await db.orders.getIndexes();

// Drop index
await db.orders.dropIndex("customer_id_1_created_at_-1");
```

## Transactions

MongoDB supports multi-document ACID transactions.

```javascript
const session = client.startSession();

try {
  await session.withTransaction(async () => {
    // Transfer funds between accounts
    await db.accounts.updateOne(
      { _id: fromAccountId },
      { $inc: { balance: -amount } },
      { session }
    );
    
    await db.accounts.updateOne(
      { _id: toAccountId },
      { $inc: { balance: amount } },
      { session }
    );
    
    // Log transaction
    await db.transactions.insertOne({
      from: fromAccountId,
      to: toAccountId,
      amount: amount,
      timestamp: new Date()
    }, { session });
  }, {
    readPreference: 'primary',
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' }
  });
  
  console.log("Transaction committed");
} catch (error) {
  console.error("Transaction aborted:", error);
} finally {
  await session.endSession();
}
```

### Transaction Best Practices

```javascript
// ✅ Keep transactions short
// ✅ Limit to related operations
// ✅ Use appropriate write concern

// ❌ Anti-pattern: Long-running transaction
await session.withTransaction(async () => {
  // Don't perform expensive computations
  // Don't make external API calls
  // Don't query large datasets
});

// ✅ Better: Move logic outside transaction
const orderData = await prepareOrderData();  // Outside transaction
const inventoryCheck = await checkInventory();  // Outside transaction

if (inventoryCheck.available) {
  await session.withTransaction(async () => {
    // Only critical writes inside transaction
    await db.orders.insertOne(orderData, { session });
    await db.inventory.updateOne(
      { _id: productId },
      { $inc: { quantity: -orderData.quantity } },
      { session }
    );
  });
}
```

## Replication and Read Preferences

### Read Preferences

```javascript
const { MongoClient, ReadPreference } = require('mongodb');

// Primary (default): Read from primary only
const users = await db.collection('users')
  .find({}, { readPreference: ReadPreference.PRIMARY })
  .toArray();

// PrimaryPreferred: Primary if available, else secondary
const products = await db.collection('products')
  .find({}, { readPreference: ReadPreference.PRIMARY_PREFERRED })
  .toArray();

// Secondary: Read from secondary only (stale reads acceptable)
const analytics = await db.collection('pageviews')
  .find({}, { readPreference: ReadPreference.SECONDARY })
  .toArray();

// SecondaryPreferred: Secondary if available, else primary
const recommendations = await db.collection('recommendations')
  .find({}, { readPreference: ReadPreference.SECONDARY_PREFERRED })
  .toArray();

// Nearest: Lowest network latency (multi-region deployments)
const content = await db.collection('articles')
  .find({}, { readPreference: ReadPreference.NEAREST })
  .toArray();
```

### Write Concerns

```javascript
// Write concern: Acknowledgment level for writes

// w: 1 (default): Acknowledge from primary only
await db.users.insertOne(
  { username: "john" },
  { writeConcern: { w: 1 } }
);

// w: "majority": Acknowledge from majority of replica set
await db.orders.insertOne(
  { order_id: "ORD-123", total: 99.99 },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
);

// w: 0: No acknowledgment (fire-and-forget)
await db.logs.insertOne(
  { message: "User logged in", timestamp: new Date() },
  { writeConcern: { w: 0 } }
);

// j: true: Wait for journal sync (durability)
await db.payments.insertOne(
  { amount: 1000, status: "completed" },
  { writeConcern: { w: "majority", j: true } }
);
```

## Sharding

### Shard Key Selection

The shard key determines data distribution. Choose carefully—it cannot be changed easily.

```javascript
// Enable sharding on database
sh.enableSharding("ecommerce");

// ✅ Good shard key: High cardinality, evenly distributed
sh.shardCollection("ecommerce.orders", { 
  customer_id: "hashed"  // Hashed sharding for even distribution
});

// ✅ Good for range queries: Compound shard key
sh.shardCollection("ecommerce.events", {
  user_id: 1,
  timestamp: 1
});

// ❌ Bad shard key: Low cardinality (few unique values)
sh.shardCollection("ecommerce.orders", { status: 1 });  // Only 5-10 values

// ❌ Bad shard key: Monotonically increasing (hot shard)
sh.shardCollection("ecommerce.logs", { _id: 1 });  // ObjectId increases over time
```

### Shard Key Characteristics

| Characteristic | Description | Example |
|----------------|-------------|---------|
| **High Cardinality** | Many unique values | user_id, email, hashed fields |
| **Low Frequency** | Values appear rarely | user_id (each user once per doc) |
| **Non-Monotonic** | Values don't increase over time | hashed user_id, random UUID |
| **Query Isolation** | Queries target single shard | customer_id for customer queries |

```mermaid
graph TB
    subgraph "Hashed Sharding (Even Distribution)"
        H1[Hash customer_id]
        H2[Chunk 1: hash 0-33%]
        H3[Chunk 2: hash 33-66%]
        H4[Chunk 3: hash 66-100%]
        
        H1 --> H2
        H1 --> H3
        H1 --> H4
    end
    
    subgraph "Range Sharding (Query Optimization)"
        R1[user_id + timestamp]
        R2[Shard 1: users A-M]
        R3[Shard 2: users N-Z]
        
        R1 --> R2
        R1 --> R3
    end
```

### Zone Sharding (Geographic Distribution)

```javascript
// Assign geographic zones to shards
sh.addShardTag("shard01", "US_EAST");
sh.addShardTag("shard02", "US_WEST");
sh.addShardTag("shard03", "EU");

// Define zone ranges
sh.addTagRange(
  "ecommerce.orders",
  { customer_country: "US", state: MinKey },
  { customer_country: "US", state: "M" },
  "US_EAST"
);

sh.addTagRange(
  "ecommerce.orders",
  { customer_country: "US", state: "M" },
  { customer_country: "US", state: MaxKey },
  "US_WEST"
);

sh.addTagRange(
  "ecommerce.orders",
  { customer_country: "GB", state: MinKey },
  { customer_country: "FR", state: MaxKey },
  "EU"
);

// Enable sharding with compound key
sh.shardCollection("ecommerce.orders", {
  customer_country: 1,
  state: 1,
  customer_id: 1
});
```

## Production Deployment

### Connection String and Options

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb://user:pass@host1:27017,host2:27017,host3:27017/dbname?" +
  "replicaSet=rs0&" +
  "w=majority&" +
  "readPreference=primaryPreferred&" +
  "maxPoolSize=100&" +
  "minPoolSize=10&" +
  "maxIdleTimeMS=300000&" +
  "serverSelectionTimeoutMS=5000&" +
  "retryWrites=true&" +
  "retryReads=true";

const client = new MongoClient(uri, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000,
  compressors: ['zlib', 'snappy'],
  zlibCompressionLevel: 6
});

await client.connect();
```

### Replica Set Configuration

```javascript
// Initialize replica set
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },  // Preferred primary
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1 }
  ]
});

// Add hidden member (for backups, doesn't become primary)
rs.add({
  host: "mongo4:27017",
  priority: 0,
  hidden: true,
  votes: 0
});

// Add delayed member (disaster recovery, 1 hour behind)
rs.add({
  host: "mongo5:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 3600,
  votes: 0
});

// View replica set status
rs.status();

// View replica set configuration
rs.conf();
```

### Hardware Recommendations

**Production MongoDB Server** (per node):
- **CPU**: 16+ cores (concurrent operations)
- **RAM**: 64-256GB (working set + OS cache)
- **Storage**: NVMe SSD, RAID 10 for redundancy
- **Network**: 10Gbps for replication and queries
- **WiredTiger Cache**: 50% of RAM (or RAM - 1GB - 256MB, whichever is greater)

**Replica Set**: Minimum 3 members across availability zones  
**Sharded Cluster**: 3 config servers, 2+ mongos routers, 3+ shards

## Performance Optimization

### Connection Pooling

```python
from pymongo import MongoClient

# Create client with connection pool
client = MongoClient(
    "mongodb://host1,host2,host3/",
    replicaSet="rs0",
    maxPoolSize=100,     # Max connections
    minPoolSize=10,      # Maintain minimum connections
    maxIdleTimeMS=45000, # Close idle connections after 45s
    waitQueueTimeoutMS=5000  # Wait max 5s for available connection
)

db = client['ecommerce']

# Connections automatically managed by pool
def get_user(user_id):
    return db.users.find_one({"_id": user_id})

# ❌ Don't create new client per request
def bad_pattern(user_id):
    client = MongoClient("mongodb://...")  # Creates new connection pool!
    result = client.db.users.find_one({"_id": user_id})
    client.close()
    return result
```

### Query Performance

```javascript
// ✅ Use projection to fetch only needed fields
const users = await db.users.find(
  { status: "active" },
  { projection: { name: 1, email: 1, _id: 0 } }
).toArray();

// ✅ Use limit to reduce data transfer
const recentOrders = await db.orders.find({
  customer_id: customerId
}).sort({ created_at: -1 }).limit(10).toArray();

// ✅ Use cursor for large result sets (streaming)
const cursor = db.products.find({});
for await (const product of cursor) {
  processProduct(product);  // Process one at a time
}

// ❌ Don't load everything into memory
const allProducts = await db.products.find({}).toArray();  // OOM if millions of docs
```

### Bulk Operations

```javascript
// ✅ Bulk writes (much faster than individual writes)
const bulkOps = [];

for (const user of users) {
  bulkOps.push({
    updateOne: {
      filter: { _id: user._id },
      update: { $set: { status: "active" } },
      upsert: true
    }
  });
}

const result = await db.users.bulkWrite(bulkOps, { ordered: false });
console.log(`Modified: ${result.modifiedCount}, Inserted: ${result.upsertedCount}`);

// ❌ Don't use loops with individual operations
for (const user of users) {
  await db.users.updateOne(
    { _id: user._id },
    { $set: { status: "active" } }
  );  // Network round-trip for each operation
}
```

### Aggregation Optimization

```javascript
// ✅ Use indexes in early stages
const pipeline = [
  { $match: { status: "completed", created_at: { $gte: startDate } } },  // Uses index
  { $lookup: { ... } },
  { $group: { ... } }
];

// ✅ Filter before expensive operations
const pipeline = [
  { $match: { category: "electronics" } },  // Reduce dataset first
  { $lookup: { from: "reviews", ... } },    // Then join
  { $unwind: "$reviews" }
];

// ✅ Use $project to drop unneeded fields early
const pipeline = [
  { $match: { ... } },
  { $project: { name: 1, price: 1, category: 1 } },  // Drop large fields
  { $group: { ... } }
];

// ❌ Don't use $lookup on unindexed fields
const pipeline = [
  { $lookup: {
    from: "reviews",
    localField: "unindexed_field",  // Slow!
    foreignField: "product_id",
    as: "reviews"
  }}
];
```

## Monitoring and Operations

### Key Metrics

```javascript
// Database statistics
db.stats();
// dataSize: Total size of data
// indexSize: Total size of indexes
// storageSize: Total storage allocated

// Collection statistics
db.orders.stats();

// Current operations
db.currentOp({
  active: true,
  secs_running: { $gte: 5 }  // Long-running queries
});

// Kill slow query
db.killOp(opId);

// Server status
db.serverStatus();
// connections: Current and available
// opcounters: Operations per second
// network: Bytes in/out
// wiredTiger: Cache hit ratio, pages evicted

// Profiler: Log slow queries
db.setProfilingLevel(1, { slowms: 100 });  // Log queries > 100ms

// View profiled queries
db.system.profile.find().sort({ ts: -1 }).limit(10);
```

### Monitoring Stack

```yaml
# MongoDB Monitoring Agent (Cloud Manager / Ops Manager)
# Tracks:
# - Query performance (slow queries, index usage)
# - Hardware metrics (CPU, RAM, disk I/O)
# - Replication lag
# - Oplog window
# - Index efficiency

# Prometheus + Grafana
# MongoDB exporter exposes metrics:
# - mongodb_connections{type="current|available"}
# - mongodb_opcounters_total{type="insert|query|update|delete"}
# - mongodb_replication_lag_seconds
# - mongodb_wiredtiger_cache_bytes{type="total|dirty|used"}
```

### Backup Strategies

```bash
# mongodump: Logical backup (BSON format)
mongodump --uri="mongodb://..." --out=/backup/$(date +%Y%m%d)

# mongorestore: Restore from dump
mongorestore --uri="mongodb://..." /backup/20240110/

# Filesystem snapshot (faster, requires replica set)
# 1. Lock secondary node
db.fsyncLock()

# 2. Take filesystem snapshot (LVM, EBS snapshot, etc.)

# 3. Unlock secondary
db.fsyncUnlock()

# Cloud Manager: Continuous backup with point-in-time recovery
# - Automated backups every 6-12 hours
# - Oplog captured continuously
# - Restore to any point within retention window
```

## Best Practices

**Design Schema for Queries**: Model data based on access patterns. Embed related data accessed together. Use references for large or rarely accessed data.

**Index Strategically**: Create indexes for all query patterns. Use compound indexes following ESR rule (Equality, Sort, Range). Monitor index usage and remove unused indexes.

**Use Appropriate Read/Write Concerns**: Use `w: "majority"` for critical writes. Use secondary reads for analytics to reduce primary load.

**Connection Pooling**: Reuse connections. Set appropriate pool size (start with 100).

**Avoid Large Documents**: Keep documents under 16MB. Use GridFS for large files.

**Limit Array Growth**: Don't let embedded arrays grow unbounded. Use separate collections for one-to-many relationships.

**Use Transactions Sparingly**: Keep transactions short. Limit to related operations. Most use cases don't need multi-document transactions.

**Monitor Replication Lag**: Lag > 10 seconds indicates issues. Check network, disk I/O, or increase secondaries.

**Plan Shard Key**: Choose high-cardinality, evenly distributed shard key. Cannot be changed without migrating data.

**Regular Backups**: Automated backups with tested restore procedures. Keep multiple restore points.

## Anti-Patterns

❌ **Massive Documents**: Don't embed unbounded arrays. 16MB limit and poor performance.

```javascript
// ❌ Bad
{
  _id: userId,
  orders: [ /* 10,000 orders */ ]  // Document grows indefinitely
}

// ✅ Good
// Separate collection for orders
```

❌ **Unnecessary Indexes**: Each index slows writes. Remove unused indexes.

```javascript
// Check index usage
db.orders.aggregate([{ $indexStats: {} }]);

// If ops: 0, drop the index
db.orders.dropIndex("unused_field_1");
```

❌ **No Shard Key in Queries**: Queries without shard key scan all shards (scatter-gather).

```javascript
// ❌ Bad (sharded by customer_id)
db.orders.find({ order_id: orderId });  // Scans all shards

// ✅ Good
db.orders.find({ customer_id: customerId, order_id: orderId });  // Targets single shard
```

❌ **SELECT * Equivalent**: Fetching all fields wastes bandwidth.

```javascript
// ❌ Bad
db.users.find({});

// ✅ Good
db.users.find({}, { projection: { name: 1, email: 1 } });
```

❌ **Long Transactions**: Transactions hold locks and resources.

❌ **Ignoring Replication Lag**: Writing to primary, immediately reading from secondary may return stale data.

## Real-World Use Case: eBay

eBay uses MongoDB for multiple services:

**Product Catalog**: 1+ billion product listings
- **Schema**: Flexible documents accommodate varied product attributes
- **Sharding**: Sharded by seller_id for isolation
- **Indexes**: Text indexes for search, compound indexes for category browsing
- **Performance**: Sub-50ms p99 latency

**User Sessions**: Active shopping sessions
- **TTL Indexes**: Auto-expire after 24 hours
- **Write Concern**: w: 1 for high throughput
- **Read Preference**: Primary for consistency

**Metadata Search**: Item descriptions, titles, attributes
- **Text Indexes**: Full-text search across fields
- **Aggregation**: Faceted search results
- **Caching**: Redis + MongoDB for hot data

**Architecture**:
- 50+ sharded clusters
- 10+ nodes per replica set
- Multi-region deployment (US, EU, Asia)
- 1M+ operations/second
- 99.99% availability SLA

---

**Related Notes**:
- [NoSQL Fundamentals](01-nosql-fundamentals.md)
- [Cassandra Deep Dive](02-cassandra.md)
- [Microservices Data Patterns](../microservices/)
