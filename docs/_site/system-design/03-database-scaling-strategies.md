# Database Scaling Strategies

## What You'll Learn

Database scaling strategies enable systems to handle increasing data volumes and query loads while maintaining performance and availability. This note covers vertical and horizontal scaling, replication, sharding, partitioning, and distributed database architectures used in production systems at companies like Facebook, Netflix, and Amazon.

## Why This Matters

Databases are often the bottleneck in distributed systems. As traffic grows from thousands to millions of requests per second, a single database server cannot handle the load. Understanding how to scale databases is critical for building systems that serve billions of users. Twitter handles 500 million tweets per day. Facebook stores petabytes of data. These scale challenges require sophisticated database scaling strategies.

## Vertical Scaling (Scale-Up)

Vertical scaling increases the capacity of a single database server by adding more CPU, RAM, or storage. This is the simplest scaling approach because it requires no architectural changes to the application. You simply provision a larger server and migrate the database.

### Advantages

Vertical scaling provides immediate performance improvements without application changes. All data remains on a single server, avoiding distributed system complexities. ACID transactions work normally. There's no data consistency concerns. Application code remains simple because it connects to a single database endpoint.

Hardware upgrades can provide significant performance gains. Moving from 32GB to 256GB of RAM allows the entire working set to fit in memory. Upgrading to NVMe SSDs reduces I/O latency from milliseconds to microseconds. Adding more CPU cores increases query parallelism.

### Limitations

Vertical scaling has hard limits. Physical hardware has maximum specifications. Even the largest cloud instances have finite resources. AWS RDS supports instances up to 768GB RAM and 96 vCPUs. Beyond this, you cannot scale further vertically.

Cost increases non-linearly. Doubling capacity often more than doubles cost. A server with 256GB RAM costs more than twice as much as one with 128GB RAM. At extreme scales, vertical scaling becomes prohibitively expensive.

Vertical scaling creates a single point of failure. If the database server fails, the entire system goes down. While backups and failover mechanisms exist, there's always downtime during recovery. Mission-critical systems require high availability that single-server architectures cannot provide.

### When to Use Vertical Scaling

Vertical scaling works well for small to medium applications where a single server can handle the load. If your database handles under 10,000 queries per second and your dataset fits on a single large server, vertical scaling is the simplest solution.

Use vertical scaling when your application isn't designed for distributed databases. Refactoring to support horizontal scaling requires significant engineering effort. If vertical scaling meets your needs for the foreseeable future, the simplicity is valuable.

## Horizontal Scaling (Scale-Out)

Horizontal scaling distributes data and load across multiple database servers. Instead of adding resources to a single server, you add more servers to form a database cluster. This approach enables nearly unlimited scalability but introduces distributed system challenges.

### Read Replicas

Read replicas create copies of the primary database that handle read queries. The primary database handles all writes and replicates changes to replicas asynchronously or synchronously. Applications route read queries to replicas, offloading the primary database.

```java
// Java example: Read/Write splitting with Spring
@Configuration
public class DatabaseConfiguration {
    
    @Bean
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://primary-db:3306/myapp")
            .build();
    }
    
    @Bean
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:mysql://replica-db:3306/myapp")
            .build();
    }
    
    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource routing = new RoutingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DatabaseType.PRIMARY, primaryDataSource());
        targetDataSources.put(DatabaseType.REPLICA, replicaDataSource());
        
        routing.setTargetDataSources(targetDataSources);
        routing.setDefaultTargetDataSource(primaryDataSource());
        
        return routing;
    }
}

@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    // Writes go to primary
    @Transactional(readOnly = false)
    public User createUser(User user) {
        DatabaseContextHolder.set(DatabaseType.PRIMARY);
        return userRepository.save(user);
    }
    
    // Reads go to replica
    @Transactional(readOnly = true)
    public User findUser(Long id) {
        DatabaseContextHolder.set(DatabaseType.REPLICA);
        return userRepository.findById(id).orElse(null);
    }
}
```

```python
# Python example: Read/Write splitting with SQLAlchemy
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, scoped_session

class DatabaseRouter:
    def __init__(self):
        # Primary database handles writes
        self.primary_engine = create_engine(
            'postgresql://primary-db:5432/myapp',
            pool_size=20,
            max_overflow=40
        )
        
        # Replica handles reads
        self.replica_engine = create_engine(
            'postgresql://replica-db:5432/myapp',
            pool_size=50,
            max_overflow=100
        )
        
        self.primary_session = scoped_session(
            sessionmaker(bind=self.primary_engine)
        )
        
        self.replica_session = scoped_session(
            sessionmaker(bind=self.replica_engine)
        )
    
    def get_write_session(self):
        """Get session for write operations"""
        return self.primary_session
    
    def get_read_session(self):
        """Get session for read operations"""
        return self.replica_session

# Usage
db_router = DatabaseRouter()

# Write operation
def create_user(user_data):
    session = db_router.get_write_session()
    user = User(**user_data)
    session.add(user)
    session.commit()
    return user

# Read operation
def get_user(user_id):
    session = db_router.get_read_session()
    return session.query(User).filter(User.id == user_id).first()
```

Read replicas work well for read-heavy workloads. If 90% of queries are reads, read replicas can handle that load. You can add multiple replicas to scale read capacity linearly. However, there's replication lag between primary and replicas, typically milliseconds to seconds. Applications must tolerate reading slightly stale data.

### Replication Strategies

**Asynchronous Replication**: The primary database doesn't wait for replicas to acknowledge writes. Writes complete quickly, providing low latency. However, if the primary fails before replication completes, recent writes may be lost. This is suitable for non-critical data where some loss is acceptable.

**Synchronous Replication**: The primary waits for at least one replica to acknowledge the write before committing. This ensures no data loss if the primary fails, but increases write latency. Each write takes at least one network round-trip. This is suitable for critical data where consistency matters more than latency.

**Semi-Synchronous Replication**: The primary waits for at least one replica but not all replicas. This balances consistency and performance, ensuring at least one copy exists without waiting for all replicas.

## Database Sharding

Sharding horizontally partitions data across multiple database servers, with each server (shard) containing a subset of the data. Unlike read replicas which contain full copies, shards contain disjoint data sets. This distributes both read and write load across servers.

### Sharding Strategies

**Range-Based Sharding**: Data is partitioned based on ranges of a shard key. Users with IDs 1-1,000,000 go to shard 1, IDs 1,000,001-2,000,000 go to shard 2, and so on. This is simple to implement but can create hotspots if data isn't uniformly distributed. If user IDs are sequential and new users cluster in high ID ranges, recent shards receive more traffic.

**Hash-Based Sharding**: A hash function maps shard keys to shards. Hash(user_id) % num_shards determines which shard stores each user. This distributes data uniformly but makes range queries difficult. Finding all users with IDs between 1,000 and 2,000 requires querying all shards.

**Geographic Sharding**: Data is partitioned by geographic region. European users go to European shards, American users to American shards. This reduces latency by keeping data close to users and helps with data sovereignty regulations. However, it can create imbalanced load if regions have different user populations.

**Directory-Based Sharding**: A lookup service maps entities to shards. When accessing a user, query the directory to find which shard contains that user. This provides flexibility to move data between shards but adds a dependency on the directory service and an extra query.

```java
// Java example: Hash-based sharding
public class ShardedDatabaseManager {
    
    private static final int NUM_SHARDS = 4;
    private final List<DataSource> shards;
    
    public ShardedDatabaseManager() {
        this.shards = new ArrayList<>();
        
        // Initialize shard connections
        for (int i = 0; i < NUM_SHARDS; i++) {
            DataSource ds = DataSourceBuilder.create()
                .url("jdbc:mysql://shard-" + i + ":3306/myapp")
                .build();
            shards.add(ds);
        }
    }
    
    private int getShardIndex(String shardKey) {
        // Use consistent hashing for better distribution
        int hashCode = shardKey.hashCode();
        return Math.abs(hashCode) % NUM_SHARDS;
    }
    
    public DataSource getShardForUser(String userId) {
        int shardIndex = getShardIndex(userId);
        return shards.get(shardIndex);
    }
    
    public User getUserById(String userId) throws SQLException {
        DataSource shard = getShardForUser(userId);
        
        try (Connection conn = shard.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "SELECT * FROM users WHERE id = ?")) {
            
            stmt.setString(1, userId);
            ResultSet rs = stmt.executeQuery();
            
            if (rs.next()) {
                return mapResultSetToUser(rs);
            }
            return null;
        }
    }
    
    public void saveUser(User user) throws SQLException {
        DataSource shard = getShardForUser(user.getId());
        
        try (Connection conn = shard.getConnection();
             PreparedStatement stmt = conn.prepareStatement(
                 "INSERT INTO users (id, name, email) VALUES (?, ?, ?) " +
                 "ON DUPLICATE KEY UPDATE name = ?, email = ?")) {
            
            stmt.setString(1, user.getId());
            stmt.setString(2, user.getName());
            stmt.setString(3, user.getEmail());
            stmt.setString(4, user.getName());
            stmt.setString(5, user.getEmail());
            
            stmt.executeUpdate();
        }
    }
    
    // Query across all shards for global operations
    public List<User> getAllUsersWithEmail(String email) throws SQLException {
        List<User> users = new ArrayList<>();
        
        // Must query all shards
        for (DataSource shard : shards) {
            try (Connection conn = shard.getConnection();
                 PreparedStatement stmt = conn.prepareStatement(
                     "SELECT * FROM users WHERE email = ?")) {
                
                stmt.setString(1, email);
                ResultSet rs = stmt.executeQuery();
                
                while (rs.next()) {
                    users.add(mapResultSetToUser(rs));
                }
            }
        }
        
        return users;
    }
    
    private User mapResultSetToUser(ResultSet rs) throws SQLException {
        User user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setEmail(rs.getString("email"));
        return user;
    }
}
```

```javascript
// Node.js example: Geographic sharding with connection pooling
const mysql = require('mysql2/promise');

class GeographicShardManager {
    constructor() {
        // Create connection pools for each geographic region
        this.shards = {
            'US': mysql.createPool({
                host: 'us-db.example.com',
                port: 3306,
                database: 'myapp',
                user: 'app_user',
                password: process.env.DB_PASSWORD,
                connectionLimit: 50,
                waitForConnections: true,
                queueLimit: 0
            }),
            
            'EU': mysql.createPool({
                host: 'eu-db.example.com',
                port: 3306,
                database: 'myapp',
                user: 'app_user',
                password: process.env.DB_PASSWORD,
                connectionLimit: 50,
                waitForConnections: true,
                queueLimit: 0
            }),
            
            'ASIA': mysql.createPool({
                host: 'asia-db.example.com',
                port: 3306,
                database: 'myapp',
                user: 'app_user',
                password: process.env.DB_PASSWORD,
                connectionLimit: 50,
                waitForConnections: true,
                queueLimit: 0
            })
        };
    }
    
    getShardForRegion(region) {
        return this.shards[region] || this.shards['US'];
    }
    
    async getUserById(userId, region) {
        const pool = this.getShardForRegion(region);
        const [rows] = await pool.execute(
            'SELECT * FROM users WHERE id = ?',
            [userId]
        );
        return rows[0];
    }
    
    async createUser(user, region) {
        const pool = this.getShardForRegion(region);
        const [result] = await pool.execute(
            'INSERT INTO users (id, name, email, region) VALUES (?, ?, ?, ?)',
            [user.id, user.name, user.email, region]
        );
        return result.insertId;
    }
    
    // Cross-shard query - expensive operation
    async searchUsersByEmail(email) {
        const results = [];
        
        // Query all shards in parallel
        const promises = Object.entries(this.shards).map(
            async ([region, pool]) => {
                const [rows] = await pool.execute(
                    'SELECT * FROM users WHERE email = ?',
                    [email]
                );
                return rows;
            }
        );
        
        const allResults = await Promise.all(promises);
        
        // Flatten results from all shards
        return allResults.flat();
    }
    
    async close() {
        // Close all connection pools
        await Promise.all(
            Object.values(this.shards).map(pool => pool.end())
        );
    }
}

// Usage
const shardManager = new GeographicShardManager();

// Query user in specific region
const user = await shardManager.getUserById('user123', 'EU');

// Cross-shard search (expensive)
const users = await shardManager.searchUsersByEmail('test@example.com');
```

### Challenges with Sharding

**Cross-Shard Queries**: Queries spanning multiple shards require querying each shard and aggregating results. Finding all users with a specific email requires checking all shards. This is expensive and increases latency. Design schemas to minimize cross-shard queries.

**Cross-Shard Transactions**: Distributed transactions across shards are complex and slow. Two-phase commit protocols ensure consistency but add significant latency. Most systems avoid cross-shard transactions by designing schemas where related data lives in the same shard.

**Rebalancing**: Adding or removing shards requires redistributing data. If you start with 4 shards and grow to 8, roughly half the data must move. This is operationally complex and can impact performance during rebalancing. Consistent hashing minimizes but doesn't eliminate this problem.

**Hot Shards**: Some shards may receive more traffic than others. A celebrity user with millions of followers creates a hot shard if all their data lives on one shard. Careful shard key selection and monitoring are required to detect and mitigate hot shards.

## Consistent Hashing for Sharding

Consistent hashing maps both shard keys and servers to points on a circle. Each key is assigned to the first server found moving clockwise from the key's position. This minimizes data movement when servers are added or removed.

When a server is removed, only keys between the removed server and the previous server need reassignment. With traditional hash-based sharding (hash % N), removing a server requires redistributing almost all keys. With consistent hashing, only 1/N keys move on average.

Virtual nodes improve distribution. Instead of mapping each server to one point on the circle, map each server to multiple virtual nodes. A server with more capacity gets more virtual nodes. This ensures more uniform load distribution.

```python
# Python: Consistent hashing implementation
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, num_virtual_nodes=150):
        self.num_virtual_nodes = num_virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        self.nodes = set()
    
    def _hash(self, key):
        """Generate hash for key"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_node(self, node):
        """Add a physical node with virtual replicas"""
        self.nodes.add(node)
        
        # Create virtual nodes for better distribution
        for i in range(self.num_virtual_nodes):
            virtual_key = f"{node}:vnode:{i}"
            hash_value = self._hash(virtual_key)
            
            self.ring[hash_value] = node
            bisect.insort(self.sorted_keys, hash_value)
    
    def remove_node(self, node):
        """Remove a physical node and its virtual replicas"""
        if node not in self.nodes:
            return
        
        self.nodes.remove(node)
        
        # Remove all virtual nodes
        for i in range(self.num_virtual_nodes):
            virtual_key = f"{node}:vnode:{i}"
            hash_value = self._hash(virtual_key)
            
            if hash_value in self.ring:
                del self.ring[hash_value]
                self.sorted_keys.remove(hash_value)
    
    def get_node(self, key):
        """Get node responsible for key"""
        if not self.ring:
            return None
        
        hash_value = self._hash(key)
        
        # Find first node clockwise from hash
        idx = bisect.bisect_right(self.sorted_keys, hash_value)
        
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def get_nodes(self, key, count=3):
        """Get N nodes for replication"""
        if count > len(self.nodes):
            count = len(self.nodes)
        
        hash_value = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_value)
        
        nodes_found = set()
        result = []
        
        # Walk clockwise until we have count unique physical nodes
        for i in range(len(self.sorted_keys)):
            ring_idx = (idx + i) % len(self.sorted_keys)
            node = self.ring[self.sorted_keys[ring_idx]]
            
            if node not in nodes_found:
                nodes_found.add(node)
                result.append(node)
                
                if len(result) == count:
                    break
        
        return result

# Usage example
hash_ring = ConsistentHashRing(num_virtual_nodes=150)

# Add database shards
for i in range(4):
    hash_ring.add_node(f"shard-{i}")

# Route keys to shards
user_id = "user12345"
shard = hash_ring.get_node(user_id)
print(f"User {user_id} -> {shard}")

# For replication, get multiple nodes
replica_shards = hash_ring.get_nodes(user_id, count=3)
print(f"Replicas: {replica_shards}")

# Add new shard - minimal redistribution
hash_ring.add_node("shard-4")

# Key may route to different shard now
new_shard = hash_ring.get_node(user_id)
print(f"After adding shard: {user_id} -> {new_shard}")
```

## Database Partitioning

Partitioning divides a large table into smaller pieces within a single database server. Unlike sharding which distributes across servers, partitioning organizes data within one database for manageability and performance.

### Horizontal Partitioning

Horizontal partitioning splits tables by rows. Each partition contains a subset of rows based on a partition key. This is similar to sharding but within a single database instance.

```sql
-- PostgreSQL: Range partitioning on timestamp
CREATE TABLE orders (
    order_id BIGINT,
    user_id BIGINT,
    order_date TIMESTAMP,
    total DECIMAL(10,2),
    status VARCHAR(50)
) PARTITION BY RANGE (order_date);

-- Create partitions for each month
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Queries automatically route to appropriate partition
SELECT * FROM orders WHERE order_date >= '2024-02-15';
-- Only queries orders_2024_02 and orders_2024_03
```

Horizontal partitioning improves query performance by scanning only relevant partitions. Querying recent orders only scans recent partition tables. It also simplifies maintenance operations like archiving old data. Drop the entire partition table instead of running expensive DELETE queries.

### Vertical Partitioning

Vertical partitioning splits tables by columns. Frequently accessed columns go in one table, rarely accessed columns in another. This is also called column-family partitioning.

```sql
-- Split users table into hot and cold columns
CREATE TABLE users_hot (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    last_login TIMESTAMP
);

CREATE TABLE users_cold (
    user_id BIGINT PRIMARY KEY,
    address TEXT,
    bio TEXT,
    preferences JSON,
    created_at TIMESTAMP
);
```

Most queries access username, email, and last_login. These fit in fewer disk blocks, improving cache efficiency. Large rarely-accessed columns like bio and preferences don't pollute the cache.

## Real-World Case Studies

### Instagram Sharding

Instagram stores billions of photos and user records across thousands of PostgreSQL shards. They use a custom sharding solution called Instagram ID that embeds shard information in the ID itself. Each ID is 64 bits: 41 bits for timestamp, 13 bits for shard ID, and 10 bits for auto-increment sequence.

This design allows Instagram to determine the shard from the ID without a lookup service. Given a photo ID, they extract the shard ID bits and route to the correct shard. They partition photos and users into the same shard, enabling efficient queries of a user's photos without cross-shard queries.

### Pinterest Sharding

Pinterest uses a sharded MySQL architecture with consistent hashing. They partition pins, boards, and users across shards using a directory service. When a user creates a pin, Pinterest assigns it to a shard based on capacity and replication requirements.

They maintain multiple replicas per shard across availability zones. Read queries go to the nearest replica. Write queries go to the primary, which replicates to all secondaries asynchronously. This provides high availability and geographic distribution.

### Uber's Schemaless

Uber built Schemaless, a sharded MySQL datastore that presents a NoSQL interface. Data is stored as JSON blobs in MySQL tables sharded across hundreds of servers. They use consistent hashing to distribute data.

Schemaless provides append-only semantics where each write creates a new version. This simplifies replication and enables time-travel queries. Deleted records are tombstoned rather than actually deleted, maintaining history.

## Comparison: Scaling Strategies

| Strategy | Read Scalability | Write Scalability | Complexity | Consistency | Use Case |
|----------|------------------|-------------------|------------|-------------|----------|
| Vertical Scaling | Low | Low | Very Low | Strong | Small applications |
| Read Replicas | High | Low | Medium | Eventual | Read-heavy workloads |
| Sharding | Very High | Very High | Very High | Varies | Massive scale |
| Partitioning | Medium | Medium | Low | Strong | Large tables |

## Best Practices

✅ **Start Simple**: Use vertical scaling and read replicas before sharding. Sharding adds significant complexity. Only shard when absolutely necessary.

✅ **Choose Shard Keys Carefully**: The shard key determines data distribution and query patterns. Choose a key that distributes data uniformly and minimizes cross-shard queries. User ID often works well.

✅ **Monitor Shard Health**: Track query latency, connection pool utilization, and disk usage per shard. Detect hot shards early and rebalance if needed.

✅ **Plan for Rebalancing**: Design systems to add shards without downtime. Use consistent hashing or directory-based sharding to minimize data movement.

✅ **Co-locate Related Data**: Store related entities in the same shard. If querying users and their orders together, shard both by user ID so they live on the same shard.

✅ **Use Connection Pooling**: Opening database connections is expensive. Use connection pools with appropriate sizing. Pool size should match expected concurrency.

✅ **Implement Circuit Breakers**: When a shard fails, fail fast rather than queuing requests. Use circuit breakers to prevent cascading failures.

✅ **Test Failover Scenarios**: Regularly test replica promotion and shard failover. Ensure automated failover works correctly before you need it in production.

## Anti-Patterns

❌ **Premature Sharding**: Sharding before you need it adds complexity without benefits. Use simpler scaling strategies first. Shard when a single database can't handle the load.

❌ **Poor Shard Key Selection**: Sharding by timestamp creates hot shards where all writes go to the most recent shard. Shard by user ID or entity ID instead.

❌ **Ignoring Cross-Shard Queries**: Designing schemas without considering query patterns leads to expensive cross-shard queries. Design schemas where common queries access single shards.

❌ **No Monitoring**: Operating sharded databases without monitoring leads to undetected failures and performance issues. Monitor each shard individually and aggregate metrics.

❌ **Manual Sharding Logic**: Hardcoding shard routing logic in application code is fragile. Use libraries or frameworks that abstract sharding complexity.

❌ **Synchronous Replication to All Replicas**: Waiting for all replicas to acknowledge writes adds latency. Use asynchronous or semi-synchronous replication instead.
