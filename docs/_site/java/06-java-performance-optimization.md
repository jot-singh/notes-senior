# Java Performance Optimization

## Concept Overview

Performance optimization involves identifying bottlenecks and applying targeted improvements to meet latency, throughput, and resource utilization goals. Effective optimization requires measurement-driven decisions, understanding JVM behavior, and avoiding premature optimization that sacrifices maintainability.

## Core Mechanics & Implementation

### Profiling and Measurement

```bash
# JVM built-in profiling
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
     -XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=app.jfr \
     -jar application.jar

# Async profiler (low overhead)
./profiler.sh -d 30 -f flamegraph.html <pid>

# JMH for microbenchmarks
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class StringBenchmark {
    private List<String> strings;
    
    @Setup
    public void setup() {
        strings = IntStream.range(0, 1000)
            .mapToObj(i -> "string" + i)
            .collect(Collectors.toList());
    }
    
    @Benchmark
    public String concatenation() {
        String result = "";
        for (String s : strings) {
            result += s;  // Bad
        }
        return result;
    }
    
    @Benchmark
    public String stringBuilder() {
        StringBuilder sb = new StringBuilder();
        for (String s : strings) {
            sb.append(s);  // Good
        }
        return sb.toString();
    }
}
```

### Memory Optimization

```java
// Avoid unnecessary object creation
// Bad: creates new Boolean each time
public Boolean isActive(String status) {
    return new Boolean(status.equals("ACTIVE"));
}

// Good: uses cached instances
public Boolean isActive(String status) {
    return Boolean.valueOf(status.equals("ACTIVE"));
}

// Object pooling for expensive objects
public class ConnectionPool {
    private final BlockingQueue<Connection> pool;
    
    public ConnectionPool(int size) {
        pool = new ArrayBlockingQueue<>(size);
        for (int i = 0; i < size; i++) {
            pool.offer(createConnection());
        }
    }
    
    public Connection acquire() throws InterruptedException {
        return pool.take();
    }
    
    public void release(Connection conn) {
        pool.offer(conn);
    }
}

// Use primitives over wrappers in hot paths
// Bad: boxing overhead
public long sumWrappers(List<Long> numbers) {
    Long sum = 0L;
    for (Long n : numbers) {
        sum += n;  // Unbox, add, box
    }
    return sum;
}

// Good: primitives
public long sumPrimitives(long[] numbers) {
    long sum = 0;
    for (long n : numbers) {
        sum += n;
    }
    return sum;
}
```

### Collection Performance

```java
// Choose right collection for access pattern
// Random access: ArrayList O(1), LinkedList O(n)
// Insert/remove middle: LinkedList O(1) with iterator, ArrayList O(n)
// Contains check: HashSet O(1), List O(n)

// Pre-size collections when size known
List<User> users = new ArrayList<>(expectedSize);
Map<String, User> userMap = new HashMap<>(expectedSize, 0.75f);

// Use EnumSet/EnumMap for enum keys
EnumSet<Day> weekend = EnumSet.of(Day.SATURDAY, Day.SUNDAY);
EnumMap<Status, Handler> handlers = new EnumMap<>(Status.class);

// Stream vs loop - measure in context
// Streams add overhead but enable parallel processing
long sum = numbers.parallelStream()
    .filter(n -> n > 0)
    .mapToLong(Long::longValue)
    .sum();
```

### String Optimization

```java
// String interning for repeated strings
String key = input.intern();  // Returns canonical instance

// StringBuilder for concatenation
StringBuilder sb = new StringBuilder(256);  // Pre-size if known
for (String part : parts) {
    sb.append(part);
}

// String.format is slow - use concatenation or StringBuilder
// Slow
String message = String.format("User %s has %d orders", name, count);
// Faster
String message = "User " + name + " has " + count + " orders";

// Use char[] for sensitive data
char[] password = getPassword();
try {
    authenticate(password);
} finally {
    Arrays.fill(password, '\0');  // Clear from memory
}
```

### I/O Optimization

```java
// Buffer I/O operations
try (BufferedReader reader = new BufferedReader(
        new FileReader(file), 8192)) {  // 8KB buffer
    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
    }
}

// NIO for large files
try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocateDirect(64 * 1024);  // Direct buffer
    while (channel.read(buffer) != -1) {
        buffer.flip();
        processBuffer(buffer);
        buffer.clear();
    }
}

// Batch database operations
try (PreparedStatement stmt = conn.prepareStatement(
        "INSERT INTO users (name, email) VALUES (?, ?)")) {
    for (User user : users) {
        stmt.setString(1, user.getName());
        stmt.setString(2, user.getEmail());
        stmt.addBatch();
        
        if (++count % 1000 == 0) {
            stmt.executeBatch();  // Execute in batches
        }
    }
    stmt.executeBatch();
}
```

### Caching Strategies

```java
// Simple in-memory cache with Caffeine
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(10))
    .recordStats()
    .build();

User user = cache.get(userId, id -> userRepository.findById(id));

// Spring caching
@Service
public class UserService {
    @Cacheable(value = "users", key = "#id")
    public User findById(String id) {
        return userRepository.findById(id);
    }
    
    @CacheEvict(value = "users", key = "#user.id")
    public void update(User user) {
        userRepository.save(user);
    }
}
```

## Best Practices

- **Measure first**: Profile before optimizing, identify actual bottlenecks
- **Set goals**: Define latency/throughput targets (p99 < 100ms)
- **Optimize hotspots**: Focus on code paths that dominate execution time
- **Minimize allocations**: Reduce GC pressure in hot paths
- **Cache appropriately**: Cache expensive computations, invalidate correctly
- **Use appropriate data structures**: Match collection type to access pattern
- **Batch I/O**: Minimize round trips to database/network
- **Avoid premature optimization**: Keep code maintainable

## Interview Questions

**Q1: How do you identify performance bottlenecks?**

Profile with JFR, async-profiler, or YourKit. Analyze flame graphs for CPU time, allocation profilers for memory. Check GC logs for memory pressure. Use APM tools (New Relic, Datadog) in production. Identify hotspots consuming most time/memory.

**Q2: Explain the difference between ArrayList and LinkedList performance.**

ArrayList: O(1) random access, O(n) insert/remove middle, contiguous memory (cache-friendly). LinkedList: O(n) random access, O(1) insert/remove with iterator, scattered memory (cache-unfriendly). ArrayList better for most use cases; LinkedList only for frequent mid-list modifications.

**Q3: How does string concatenation affect performance?**

String is immutable; concatenation creates new objects. In loops, each `+=` creates StringBuilder, appends, creates new String. Use StringBuilder explicitly for multiple concatenations. String.format() is slower than concatenation. Compiler optimizes single-line concatenation.

**Q4: What is object pooling and when to use it?**

Reuse expensive objects instead of creating new ones. Use for database connections, thread pools, socket connections, large byte arrays. Implement with BlockingQueue or libraries like Apache Commons Pool. Adds complexity—only use when object creation is measured bottleneck.

**Q5: How to reduce GC pressure?**

Minimize allocations in hot paths, reuse objects, use primitives over wrappers, pre-size collections, avoid unnecessary autoboxing, use object pools for expensive objects. Off-heap storage for large data. Monitor with GC logs.

**Q6: Explain caching strategies and invalidation.**

Cache expensive computations or I/O results. Strategies: TTL (time-based expiry), LRU (size-based eviction), write-through/write-behind. Invalidation: explicit eviction, TTL expiry, event-driven. Cache-aside pattern: check cache, load from source if miss, populate cache.

**Q7: What JVM flags improve performance?**

`-XX:+UseG1GC` for balanced throughput/latency. `-Xms/-Xmx` same value avoids resize. `-XX:MaxGCPauseMillis` for latency targets. `-XX:+UseStringDeduplication` reduces memory. `-XX:+AlwaysPreTouch` pre-allocates heap. `-XX:+UseNUMA` for multi-socket systems.

**Q8: How to optimize database access?**

Use connection pooling, batch operations, prepared statements. Enable query caching, optimize indexes. Fetch only needed columns. Use pagination for large results. Consider read replicas. Cache frequently accessed data. Profile queries with EXPLAIN.
