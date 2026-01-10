# Java Concurrency

## Concept Overview

Java concurrency enables multiple threads to execute simultaneously, utilizing multi-core processors for parallel execution and maintaining responsiveness in applications performing long-running operations. Concurrency becomes essential when building high-performance servers handling thousands of concurrent requests, responsive user interfaces that remain interactive during background tasks, or data processing pipelines that maximize CPU utilization.

The Java concurrency model provides low-level threading primitives (Thread, Runnable) alongside high-level abstractions (ExecutorService, CompletableFuture, parallel streams) that handle thread management complexity. The java.util.concurrent package, introduced in Java 5, revolutionized concurrent programming by providing battle-tested implementations of common patterns: thread pools, concurrent collections, synchronization utilities, and atomic variables.

Concurrency solves several critical problems: improving throughput by utilizing available CPU cores, maintaining application responsiveness by offloading work to background threads, and handling naturally concurrent scenarios like serving multiple simultaneous users. However, concurrency introduces complexity through shared mutable state, requiring careful synchronization to prevent race conditions, deadlocks, and data corruption.

Use Java concurrency when building web servers requiring simultaneous request handling, data processing jobs benefiting from parallel execution, real-time systems needing responsive event handling, or applications performing multiple independent I/O operations. Understanding when to apply concurrency—and when single-threaded solutions suffice—separates senior developers from junior ones.

## Core Mechanics & Implementation

### Threads and Runnable

Threads represent independent execution paths within a program. Java provides two primary ways to create threads: extending Thread class or implementing Runnable interface. The Runnable approach is preferred as it allows separating task logic from thread management and enables implementing multiple interfaces.

```java
// Runnable approach (preferred)
public class DataProcessor implements Runnable {
    private final String data;
    
    public DataProcessor(String data) {
        this.data = data;
    }
    
    @Override
    public void run() {
        // Thread execution logic
        System.out.println("Processing " + data + " on thread: " 
            + Thread.currentThread().getName());
        processData(data);
    }
    
    private void processData(String data) {
        // Actual processing logic
    }
}

// Creating and starting threads
Thread thread1 = new Thread(new DataProcessor("dataset1"));
Thread thread2 = new Thread(new DataProcessor("dataset2"));

thread1.start();  // Starts new thread, calls run() method
thread2.start();

// Lambda syntax (Java 8+) for simple tasks
Thread thread3 = new Thread(() -> {
    System.out.println("Lambda task running");
});
thread3.start();

// Thread extending approach (less flexible)
public class WorkerThread extends Thread {
    @Override
    public void run() {
        System.out.println("Worker thread executing");
    }
}

new WorkerThread().start();
```

Thread lifecycle states include NEW (created but not started), RUNNABLE (executing or ready to execute), BLOCKED (waiting for monitor lock), WAITING (waiting indefinitely for another thread), TIMED_WAITING (waiting for specified time), and TERMINATED (execution completed). Understanding these states helps diagnose thread issues and optimize performance.

```java
// Thread lifecycle management
Thread thread = new Thread(() -> {
    try {
        System.out.println("Thread starting");
        Thread.sleep(1000);  // TIMED_WAITING state
        System.out.println("Thread completing");
    } catch (InterruptedException e) {
        System.out.println("Thread interrupted");
        Thread.currentThread().interrupt();  // Restore interrupted status
    }
});

System.out.println("State: " + thread.getState());  // NEW
thread.start();
System.out.println("State: " + thread.getState());  // RUNNABLE

thread.join();  // Wait for thread to complete
System.out.println("State: " + thread.getState());  // TERMINATED
```

### Synchronization and Locks

Synchronization prevents race conditions when multiple threads access shared mutable state. The `synchronized` keyword provides mutual exclusion, allowing only one thread to execute a synchronized block at a time. Every object has an intrinsic lock (monitor) used by synchronized blocks.

```java
public class Counter {
    private int count = 0;
    
    // Synchronized method - acquires lock on 'this'
    public synchronized void increment() {
        count++;  // Read-modify-write, must be atomic
    }
    
    public synchronized int getCount() {
        return count;
    }
    
    // Synchronized block - explicit lock object
    private final Object lock = new Object();
    
    public void incrementWithBlock() {
        synchronized(lock) {
            count++;
        }
    }
    
    // Fine-grained locking - synchronize only critical section
    public void processData() {
        // Non-synchronized operations
        String data = prepareData();
        
        // Synchronize only the update
        synchronized(this) {
            updateSharedState(data);
        }
        
        // More non-synchronized work
        postProcess();
    }
}

// Using the thread-safe counter
Counter counter = new Counter();

// Create multiple threads incrementing same counter
List<Thread> threads = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    Thread thread = new Thread(() -> {
        for (int j = 0; j < 1000; j++) {
            counter.increment();
        }
    });
    threads.add(thread);
    thread.start();
}

// Wait for all threads
for (Thread thread : threads) {
    thread.join();
}

System.out.println("Final count: " + counter.getCount());  // 10000 (correct)
```

ReentrantLock provides explicit locking with additional features: try-locking with timeout, interruptible lock acquisition, and fairness policies. Use ReentrantLock when you need these advanced features or clearer lock scope control.

```java
public class BankAccount {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void deposit(double amount) {
        lock.lock();  // Acquire lock
        try {
            balance += amount;
        } finally {
            lock.unlock();  // Always unlock in finally block
        }
    }
    
    public boolean withdraw(double amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
    
    // Try-lock with timeout
    public boolean withdrawWithTimeout(double amount, long timeoutMs) 
            throws InterruptedException {
        if (lock.tryLock(timeoutMs, TimeUnit.MILLISECONDS)) {
            try {
                if (balance >= amount) {
                    balance -= amount;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Couldn't acquire lock
    }
    
    // Check if operation can proceed without blocking
    public boolean quickCheck() {
        if (lock.tryLock()) {  // Non-blocking attempt
            try {
                return balance > 0;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Lock held by another thread
    }
}
```

ReadWriteLock allows multiple concurrent readers or one exclusive writer, optimizing scenarios with frequent reads and infrequent writes.

```java
public class CachedData {
    private Map<String, String> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    
    // Multiple threads can read simultaneously
    public String get(String key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    // Only one thread can write, blocks all readers
    public void put(String key, String value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
    
    // Upgrade from read to write (careful - potential deadlock)
    public String computeIfAbsent(String key) {
        readLock.lock();
        try {
            String value = cache.get(key);
            if (value != null) {
                return value;
            }
        } finally {
            readLock.unlock();
        }
        
        // Acquire write lock outside read lock to avoid deadlock
        writeLock.lock();
        try {
            // Check again - another thread might have added it
            String value = cache.get(key);
            if (value == null) {
                value = expensiveComputation(key);
                cache.put(key, value);
            }
            return value;
        } finally {
            writeLock.unlock();
        }
    }
    
    private String expensiveComputation(String key) {
        return "computed-" + key;
    }
}
```

### Thread Pools and ExecutorService

Thread pools reuse a fixed number of threads for executing tasks, avoiding thread creation overhead and controlling resource usage. ExecutorService manages thread pools, task submission, and lifecycle.

```java
// Fixed thread pool - reuses fixed number of threads
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit tasks for execution
for (int i = 0; i < 10; i++) {
    final int taskId = i;
    executor.submit(() -> {
        System.out.println("Task " + taskId + " running on " 
            + Thread.currentThread().getName());
        performWork(taskId);
    });
}

// Shutdown executor gracefully
executor.shutdown();  // No new tasks accepted
try {
    // Wait for existing tasks to complete
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();  // Force shutdown
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
}

// Different executor types
ExecutorService cachedPool = Executors.newCachedThreadPool();  
// Creates threads as needed, reuses idle ones

ExecutorService singleThread = Executors.newSingleThreadExecutor();  
// Single worker thread, tasks execute sequentially

ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
// Supports delayed and periodic task execution

// Scheduled task examples
scheduler.schedule(() -> {
    System.out.println("Runs once after 5 seconds");
}, 5, TimeUnit.SECONDS);

scheduler.scheduleAtFixedRate(() -> {
    System.out.println("Runs every 10 seconds");
}, 0, 10, TimeUnit.SECONDS);  // initial delay, period

scheduler.scheduleWithFixedDelay(() -> {
    System.out.println("Runs 10 seconds after previous completion");
}, 0, 10, TimeUnit.SECONDS);  // initial delay, delay between completions
```

Callables return values and throw checked exceptions, unlike Runnable. Future represents the result of an asynchronous computation, allowing retrieval when complete.

```java
// Callable returns a value
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

ExecutorService executor = Executors.newFixedThreadPool(2);

// Submit returns Future
Future<Integer> future = executor.submit(task);

// Do other work while task executes
System.out.println("Task submitted, doing other work");

try {
    // Get result, blocks until available
    Integer result = future.get();  // Blocks
    System.out.println("Result: " + result);
    
    // Get with timeout
    Integer result2 = future.get(5, TimeUnit.SECONDS);
    
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
} catch (ExecutionException e) {
    // Task threw exception
    Throwable cause = e.getCause();
    System.err.println("Task failed: " + cause);
} catch (TimeoutException e) {
    // Task didn't complete in time
    future.cancel(true);  // Cancel task
}

// Check future status
if (future.isDone()) {
    System.out.println("Task completed");
}

if (future.isCancelled()) {
    System.out.println("Task was cancelled");
}

executor.shutdown();
```

### CompletableFuture

CompletableFuture enables functional-style asynchronous programming with composition, combining multiple async operations through chainable methods.

```java
// Create CompletableFuture
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // Runs asynchronously in ForkJoinPool.commonPool()
    return fetchDataFromDatabase();
});

// Chain operations
CompletableFuture<String> processed = future
    .thenApply(data -> {
        // Transform result (runs in same thread as previous stage)
        return data.toUpperCase();
    })
    .thenApply(data -> {
        // Further transformation
        return "Processed: " + data;
    });

// Handle completion
processed.thenAccept(result -> {
    // Consume result (no return value)
    System.out.println(result);
});

// Exception handling
CompletableFuture<String> withErrorHandling = future
    .thenApply(data -> data.toUpperCase())
    .exceptionally(ex -> {
        // Handle exception, return fallback value
        System.err.println("Error: " + ex.getMessage());
        return "DEFAULT";
    });

// Combine multiple futures
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = future1.thenCombine(future2, (s1, s2) -> {
    return s1 + " " + s2;
});

combined.thenAccept(System.out::println);  // "Hello World"

// Wait for all futures
CompletableFuture<Void> allOf = CompletableFuture.allOf(future1, future2);
allOf.join();  // Blocks until all complete

// Wait for any future
CompletableFuture<Object> anyOf = CompletableFuture.anyOf(future1, future2);
Object result = anyOf.join();  // Result from first completing future

// Complex workflow example
CompletableFuture<String> workflow = CompletableFuture
    .supplyAsync(() -> fetchUserId())
    .thenCompose(userId -> fetchUserProfile(userId))  // Flat map
    .thenCompose(profile -> fetchUserOrders(profile.getId()))
    .thenApply(orders -> formatOrders(orders))
    .exceptionally(ex -> {
        logError(ex);
        return "Error fetching data";
    });

String finalResult = workflow.join();  // Block and get result
```

### Concurrent Collections

Java provides thread-safe collections optimized for concurrent access, avoiding external synchronization and offering better performance than synchronized wrappers.

```java
// ConcurrentHashMap - thread-safe map with fine-grained locking
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("key1", 100);
map.putIfAbsent("key2", 200);  // Atomic operation

// Atomic compute operations
map.compute("key1", (key, value) -> {
    return value == null ? 1 : value + 1;
});

map.computeIfAbsent("key3", key -> expensiveComputation(key));

map.merge("key1", 10, (oldValue, newValue) -> oldValue + newValue);

// Iterate safely - weakly consistent iterator
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
    // Other threads can modify map during iteration
}

// CopyOnWriteArrayList - iteration without synchronization
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("item1");
list.add("item2");

// Multiple readers iterate without blocking
for (String item : list) {
    System.out.println(item);
    // Modifications create new underlying array
}

// Good for many reads, few writes
// Writes are expensive (copies entire array)

// ConcurrentLinkedQueue - non-blocking thread-safe queue
ConcurrentLinkedQueue<Task> queue = new ConcurrentLinkedQueue<>();

queue.offer(new Task("task1"));  // Add to tail
Task task = queue.poll();         // Remove from head

// BlockingQueue - blocking operations for producer-consumer
BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<>(10);

// Producer thread
new Thread(() -> {
    try {
        blockingQueue.put("item");  // Blocks if queue full
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();

// Consumer thread
new Thread(() -> {
    try {
        String item = blockingQueue.take();  // Blocks if queue empty
        processItem(item);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}).start();
```

### Atomic Variables

Atomic classes provide lock-free thread-safe operations on single variables using compare-and-swap (CAS) instructions.

```java
// AtomicInteger - thread-safe integer operations
AtomicInteger counter = new AtomicInteger(0);

// Atomic operations
counter.incrementAndGet();  // ++counter
counter.getAndIncrement();  // counter++
counter.addAndGet(5);       // counter += 5
counter.getAndAdd(5);       // temp = counter; counter += 5; return temp

// Compare and set
int expected = 10;
int newValue = 20;
boolean updated = counter.compareAndSet(expected, newValue);
// Updates only if current value equals expected

// Example: thread-safe counter without locks
public class Statistics {
    private final AtomicLong successCount = new AtomicLong(0);
    private final AtomicLong failureCount = new AtomicLong(0);
    private final AtomicReference<Instant> lastUpdate = 
        new AtomicReference<>(Instant.now());
    
    public void recordSuccess() {
        successCount.incrementAndGet();
        lastUpdate.set(Instant.now());
    }
    
    public void recordFailure() {
        failureCount.incrementAndGet();
        lastUpdate.set(Instant.now());
    }
    
    public long getSuccessCount() {
        return successCount.get();
    }
    
    public long getFailureCount() {
        return failureCount.get();
    }
    
    // Complex atomic operation
    public long incrementSuccessIfNotMax(long maxValue) {
        long current;
        long next;
        do {
            current = successCount.get();
            if (current >= maxValue) {
                return current;
            }
            next = current + 1;
        } while (!successCount.compareAndSet(current, next));
        return next;
    }
}
```

### Thread Coordination

Java provides utilities for coordinating thread execution: CountDownLatch for waiting on multiple operations, CyclicBarrier for coordinating multiple threads at a synchronization point, Semaphore for controlling access to resources, and Phaser for advanced coordination scenarios.

```java
// CountDownLatch - wait for N operations to complete
CountDownLatch latch = new CountDownLatch(3);

ExecutorService executor = Executors.newFixedThreadPool(3);

for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        performInitialization();
        latch.countDown();  // Decrement count
    });
}

// Wait for all initialization to complete
latch.await();  // Blocks until count reaches zero
System.out.println("All initialization complete");

// CyclicBarrier - synchronize N threads at a point
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All threads reached barrier");
});

for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        performPhase1();
        barrier.await();  // Wait for all threads
        performPhase2();
        // Barrier resets, can be reused
    });
}

// Semaphore - limit concurrent access
Semaphore semaphore = new Semaphore(2);  // Allow 2 concurrent accesses

for (int i = 0; i < 10; i++) {
    executor.submit(() -> {
        try {
            semaphore.acquire();  // Acquire permit
            accessLimitedResource();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            semaphore.release();  // Release permit
        }
    });
}

// Phaser - flexible multi-phase coordination
Phaser phaser = new Phaser(1);  // Register main thread

for (int i = 0; i < 3; i++) {
    phaser.register();  // Register each worker
    
    executor.submit(() -> {
        // Phase 0
        performPhase0();
        phaser.arriveAndAwaitAdvance();  // Wait for all
        
        // Phase 1
        performPhase1();
        phaser.arriveAndAwaitAdvance();
        
        // Phase 2
        performPhase2();
        phaser.arriveAndDeregister();  // Done with all phases
    });
}

phaser.arriveAndDeregister();  // Main thread done registering
```

## Best Practices & Patterns

### Prefer High-Level Concurrency Utilities

Use ExecutorService instead of creating threads directly. Let thread pools manage thread lifecycle, resource pooling, and task queuing. Use CompletableFuture for async operations requiring composition. Leverage concurrent collections instead of synchronized wrappers.

```java
// Bad - manual thread management
for (Task task : tasks) {
    new Thread(() -> task.execute()).start();
}

// Good - thread pool manages threads
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

for (Task task : tasks) {
    executor.submit(() -> task.execute());
}

executor.shutdown();
```

### Minimize Shared Mutable State

Design immutable objects that can be safely shared without synchronization. Prefer thread-local variables for thread-specific data. Use message passing (queues) instead of shared memory when possible.

```java
// Thread-safe through immutability
public final class UserProfile {
    private final String userId;
    private final String name;
    private final Instant created;
    
    public UserProfile(String userId, String name, Instant created) {
        this.userId = userId;
        this.name = name;
        this.created = created;
    }
    
    // Only getters, no setters - immutable
    public String getUserId() { return userId; }
    public String getName() { return name; }
    public Instant getCreated() { return created; }
}

// ThreadLocal for thread-specific data
public class RequestContext {
    private static final ThreadLocal<String> requestId = 
        ThreadLocal.withInitial(() -> UUID.randomUUID().toString());
    
    public static String getRequestId() {
        return requestId.get();
    }
    
    public static void setRequestId(String id) {
        requestId.set(id);
    }
    
    public static void clear() {
        requestId.remove();  // Prevent memory leaks
    }
}
```

### Proper Lock Ordering

Always acquire multiple locks in a consistent order to prevent deadlocks. Document lock ordering requirements. Consider using tryLock with timeout for deadlock recovery.

```java
public class BankTransfer {
    public boolean transfer(Account from, Account to, double amount) {
        // Acquire locks in consistent order (by account ID)
        Account first = from.getId() < to.getId() ? from : to;
        Account second = from.getId() < to.getId() ? to : from;
        
        synchronized(first) {
            synchronized(second) {
                if (from.getBalance() >= amount) {
                    from.withdraw(amount);
                    to.deposit(amount);
                    return true;
                }
                return false;
            }
        }
    }
}
```

### Exception Handling in Threads

Uncaught exceptions in threads terminate the thread silently. Set UncaughtExceptionHandler to handle exceptions. Use ExecutorService for automatic exception handling through Future.

```java
// Set default exception handler
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> {
    System.err.println("Exception in thread " + thread.getName());
    throwable.printStackTrace();
    // Log to monitoring system
});

// Per-thread exception handler
Thread thread = new Thread(() -> {
    throw new RuntimeException("Error!");
});
thread.setUncaughtExceptionHandler((t, e) -> {
    System.err.println("Thread " + t.getName() + " failed: " + e.getMessage());
});
thread.start();

// ExecutorService captures exceptions in Future
ExecutorService executor = Executors.newFixedThreadPool(2);
Future<?> future = executor.submit(() -> {
    throw new RuntimeException("Task failed");
});

try {
    future.get();  // Exception thrown here
} catch (ExecutionException e) {
    Throwable cause = e.getCause();
    // Handle exception from task
}
```

### Volatile for Simple Flags

Use `volatile` for simple boolean flags and references that don't need atomic compound operations. Volatile ensures visibility across threads without synchronization overhead.

```java
public class TaskController {
    private volatile boolean running = true;
    
    public void run() {
        while (running) {  // Sees updates from other threads
            performWork();
        }
    }
    
    public void stop() {
        running = false;  // Visible to all threads
    }
}
```

### Producer-Consumer Pattern

Use BlockingQueue for clean producer-consumer implementation. The queue handles synchronization and blocking behavior.

```java
public class LogProcessor {
    private final BlockingQueue<LogEntry> queue = 
        new ArrayBlockingQueue<>(1000);
    private final ExecutorService executor = 
        Executors.newFixedThreadPool(3);
    
    public void start() {
        // Start consumer threads
        for (int i = 0; i < 3; i++) {
            executor.submit(this::processLogs);
        }
    }
    
    // Producer
    public void log(LogEntry entry) {
        try {
            queue.put(entry);  // Blocks if queue full
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    // Consumer
    private void processLogs() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                LogEntry entry = queue.take();  // Blocks if empty
                processEntry(entry);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

## Real-World Applications

### Web Server Request Handling

Web servers use thread pools to handle concurrent HTTP requests. Each request executes in a separate thread from the pool, enabling simultaneous processing of thousands of requests.

```java
@RestController
public class OrderController {
    private final OrderService orderService;
    private final ExecutorService asyncExecutor = 
        Executors.newFixedThreadPool(10);
    
    @PostMapping("/orders")
    public CompletableFuture<OrderResponse> createOrder(
            @RequestBody OrderRequest request) {
        // Return immediately, process asynchronously
        return CompletableFuture.supplyAsync(() -> {
            Order order = orderService.createOrder(request);
            sendConfirmationEmail(order);
            updateInventory(order);
            return new OrderResponse(order);
        }, asyncExecutor);
    }
    
    @GetMapping("/orders/{id}")
    public DeferredResult<Order> getOrder(@PathVariable String id) {
        DeferredResult<Order> result = new DeferredResult<>(5000L);
        
        CompletableFuture.supplyAsync(() -> orderService.findById(id))
            .whenComplete((order, ex) -> {
                if (ex != null) {
                    result.setErrorResult(ex);
                } else if (order.isPresent()) {
                    result.setResult(order.get());
                } else {
                    result.setErrorResult(new OrderNotFoundException());
                }
            });
        
        return result;
    }
}
```

### Parallel Data Processing

Process large datasets in parallel using parallel streams or explicit thread pools. Divide work into independent chunks processed concurrently.

```java
public class DataAnalyzer {
    private final ExecutorService executor;
    
    public DataAnalyzer(int threadCount) {
        this.executor = Executors.newFixedThreadPool(threadCount);
    }
    
    public Map<String, Statistics> analyzeData(List<DataRecord> records) {
        // Partition data
        int chunkSize = records.size() / Runtime.getRuntime().availableProcessors();
        List<List<DataRecord>> partitions = partition(records, chunkSize);
        
        // Process partitions in parallel
        List<Future<Map<String, Statistics>>> futures = new ArrayList<>();
        for (List<DataRecord> partition : partitions) {
            Future<Map<String, Statistics>> future = executor.submit(() -> 
                processPartition(partition)
            );
            futures.add(future);
        }
        
        // Merge results
        Map<String, Statistics> combined = new ConcurrentHashMap<>();
        for (Future<Map<String, Statistics>> future : futures) {
            try {
                Map<String, Statistics> partial = future.get();
                partial.forEach((key, stats) -> 
                    combined.merge(key, stats, Statistics::merge)
                );
            } catch (Exception e) {
                throw new RuntimeException("Analysis failed", e);
            }
        }
        
        return combined;
    }
    
    private Map<String, Statistics> processPartition(List<DataRecord> records) {
        Map<String, Statistics> result = new HashMap<>();
        for (DataRecord record : records) {
            Statistics stats = result.computeIfAbsent(
                record.getCategory(), 
                k -> new Statistics()
            );
            stats.update(record);
        }
        return result;
    }
}
```

### Real-Time Event Processing

Process events from multiple sources concurrently, maintaining throughput and responsiveness.

```java
public class EventProcessor {
    private final BlockingQueue<Event> eventQueue = 
        new LinkedBlockingQueue<>(10000);
    private final ExecutorService processors = 
        Executors.newFixedThreadPool(5);
    private final ScheduledExecutorService monitor = 
        Executors.newScheduledThreadPool(1);
    
    public void start() {
        // Start event processors
        for (int i = 0; i < 5; i++) {
            processors.submit(this::processEvents);
        }
        
        // Monitor queue depth
        monitor.scheduleAtFixedRate(() -> {
            int queueSize = eventQueue.size();
            if (queueSize > 8000) {
                System.err.println("High queue depth: " + queueSize);
            }
        }, 0, 10, TimeUnit.SECONDS);
    }
    
    public void submitEvent(Event event) {
        if (!eventQueue.offer(event)) {
            // Queue full, handle overflow
            handleQueueOverflow(event);
        }
    }
    
    private void processEvents() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                List<Event> batch = new ArrayList<>();
                eventQueue.drainTo(batch, 100);  // Batch processing
                
                if (!batch.isEmpty()) {
                    processBatch(batch);
                } else {
                    Event event = eventQueue.poll(100, TimeUnit.MILLISECONDS);
                    if (event != null) {
                        processEvent(event);
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void shutdown() {
        processors.shutdown();
        monitor.shutdown();
    }
}
```

### Background Task Execution

Execute background tasks without blocking main application flow. Schedule periodic maintenance, cleanup, or data synchronization.

```java
public class TaskScheduler {
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(2);
    
    public void scheduleBackgroundTasks() {
        // Periodic cache cleanup
        scheduler.scheduleAtFixedRate(() -> {
            try {
                cleanupExpiredCacheEntries();
            } catch (Exception e) {
                logger.error("Cache cleanup failed", e);
            }
        }, 0, 5, TimeUnit.MINUTES);
        
        // Daily report generation
        LocalTime targetTime = LocalTime.of(2, 0);  // 2 AM
        long initialDelay = calculateInitialDelay(targetTime);
        
        scheduler.scheduleAtFixedRate(() -> {
            try {
                generateDailyReport();
            } catch (Exception e) {
                logger.error("Report generation failed", e);
            }
        }, initialDelay, TimeUnit.DAYS.toSeconds(1), TimeUnit.SECONDS);
    }
    
    private long calculateInitialDelay(LocalTime targetTime) {
        LocalDateTime now = LocalDateTime.now();
        LocalDateTime nextRun = now.with(targetTime);
        if (now.compareTo(nextRun) > 0) {
            nextRun = nextRun.plusDays(1);
        }
        return Duration.between(now, nextRun).getSeconds();
    }
}
```

## Common Pitfalls & Anti-patterns

### Race Conditions

Multiple threads accessing shared mutable state without proper synchronization causes race conditions, leading to data corruption and unpredictable behavior.

```java
// Bad - race condition
public class UnsafeCounter {
    private int count = 0;
    
    public void increment() {
        count++;  // Read, modify, write - not atomic!
    }
}

// Two threads calling increment() might:
// Thread 1: read count (0)
// Thread 2: read count (0)
// Thread 1: write count (1)
// Thread 2: write count (1)
// Result: count = 1 (should be 2)

// Good - synchronized
public class SafeCounter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
}

// Better - atomic variable
public class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();
    }
}
```

### Deadlocks

Circular wait conditions cause deadlocks where threads wait indefinitely for each other's locks.

```java
// Bad - potential deadlock
public class DeadlockExample {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    public void method1() {
        synchronized(lock1) {
            synchronized(lock2) {
                // Work
            }
        }
    }
    
    public void method2() {
        synchronized(lock2) {  // Different order!
            synchronized(lock1) {
                // Work
            }
        }
    }
    
    // Thread 1 calls method1(), acquires lock1
    // Thread 2 calls method2(), acquires lock2
    // Thread 1 waits for lock2
    // Thread 2 waits for lock1
    // DEADLOCK!
}

// Good - consistent lock ordering
public class NoDeadlock {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    public void method1() {
        synchronized(lock1) {
            synchronized(lock2) {
                // Work
            }
        }
    }
    
    public void method2() {
        synchronized(lock1) {  // Same order
            synchronized(lock2) {
                // Work
            }
        }
    }
}
```

### Thread Leaks

Creating threads without proper management causes resource exhaustion. Always use thread pools and ensure proper shutdown.

```java
// Bad - unbounded thread creation
public void handleRequests() {
    while (true) {
        Request request = getNextRequest();
        new Thread(() -> processRequest(request)).start();
        // Threads accumulate, system runs out of resources
    }
}

// Good - thread pool with bounded size
public class RequestHandler {
    private final ExecutorService executor = 
        Executors.newFixedThreadPool(100);
    
    public void handleRequests() {
        while (running) {
            Request request = getNextRequest();
            executor.submit(() -> processRequest(request));
        }
    }
    
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}
```

### Improper Volatile Usage

Using `volatile` for compound operations doesn't guarantee atomicity. Volatile only ensures visibility, not atomicity.

```java
// Bad - volatile doesn't make this thread-safe
public class VolatileMisuse {
    private volatile int count = 0;
    
    public void increment() {
        count++;  // Still a race condition! Read-modify-write not atomic
    }
}

// Good - use atomic variable or synchronization
public class CorrectAtomic {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();
    }
}
```

### Ignoring Interruption

Not properly handling InterruptedException allows threads to continue when they should stop.

```java
// Bad - swallowing interruption
public void process() {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        // Ignored! Thread doesn't know it was interrupted
    }
}

// Good - restore interrupt status
public void process() {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();  // Restore flag
        // Clean up and exit
    }
}

// Better - propagate exception
public void process() throws InterruptedException {
    Thread.sleep(1000);
    // Let caller decide how to handle
}
```

## Advanced Topics & Edge Cases

### Memory Visibility and Happens-Before

The Java Memory Model defines when writes by one thread become visible to others. Synchronization establishes happens-before relationships guaranteeing visibility.

Actions creating happens-before relationships: releasing a lock happens-before acquiring same lock, writing volatile variable happens-before reading it, thread completion happens-before join() returns, starting a thread happens-before any actions in that thread.

```java
public class VisibilityExample {
    private int value = 0;
    private volatile boolean ready = false;
    
    // Writer thread
    public void writer() {
        value = 42;         // Write 1
        ready = true;       // Write 2 (volatile)
    }
    
    // Reader thread
    public void reader() {
        while (!ready) { }  // Read volatile
        System.out.println(value);  // Guaranteed to see 42
    }
    
    // Volatile write happens-before volatile read
    // All writes before volatile write visible after volatile read
}
```

### Lock-Free Algorithms

Compare-and-swap (CAS) enables lock-free data structures avoiding blocking. These algorithms remain responsive even under contention but require careful design.

```java
public class LockFreeStack<T> {
    private static class Node<T> {
        final T value;
        Node<T> next;
        
        Node(T value) {
            this.value = value;
        }
    }
    
    private final AtomicReference<Node<T>> head = 
        new AtomicReference<>();
    
    public void push(T value) {
        Node<T> newNode = new Node<>(value);
        Node<T> oldHead;
        do {
            oldHead = head.get();
            newNode.next = oldHead;
        } while (!head.compareAndSet(oldHead, newNode));
    }
    
    public T pop() {
        Node<T> oldHead;
        Node<T> newHead;
        do {
            oldHead = head.get();
            if (oldHead == null) {
                return null;
            }
            newHead = oldHead.next;
        } while (!head.compareAndSet(oldHead, newHead));
        return oldHead.value;
    }
}
```

### Fork/Join Framework

Fork/Join framework implements work-stealing for divide-and-conquer algorithms. Tasks split into subtasks, distributed among threads, and results combined.

```java
public class ParallelSum extends RecursiveTask<Long> {
    private final long[] array;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 10000;
    
    public ParallelSum(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }
    
    @Override
    protected Long compute() {
        int length = end - start;
        
        if (length <= THRESHOLD) {
            // Small enough, compute directly
            return computeDirectly();
        }
        
        // Split into subtasks
        int mid = start + length / 2;
        ParallelSum left = new ParallelSum(array, start, mid);
        ParallelSum right = new ParallelSum(array, mid, end);
        
        // Fork left subtask
        left.fork();
        
        // Compute right subtask in current thread
        long rightResult = right.compute();
        
        // Wait for left subtask
        long leftResult = left.join();
        
        return leftResult + rightResult;
    }
    
    private long computeDirectly() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += array[i];
        }
        return sum;
    }
}

// Usage
long[] array = new long[1000000];
ForkJoinPool pool = new ForkJoinPool();
long result = pool.invoke(new ParallelSum(array, 0, array.length));
```

### Virtual Threads (Java 21+)

Virtual threads (Project Loom) enable millions of concurrent threads with minimal overhead, simplifying concurrent programming for I/O-bound applications.

```java
// Traditional platform threads - expensive
ExecutorService traditional = Executors.newFixedThreadPool(100);

// Virtual threads - lightweight, millions possible
ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();

// Create virtual thread directly
Thread.startVirtualThread(() -> {
    performIOOperation();
});

// Virtual threads ideal for I/O-bound tasks
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            // Blocking I/O operation
            String data = httpClient.get(url);
            processData(data);
        });
    }
}  // All tasks complete before closing
```

## Interview Questions

**Q1: Explain the difference between synchronized and ReentrantLock.**

`synchronized` provides implicit locking through monitor objects with simpler syntax requiring no manual unlock. The JVM automatically releases locks when exiting synchronized blocks, even with exceptions. Use synchronized for straightforward mutual exclusion without advanced features.

`ReentrantLock` offers explicit locking with additional capabilities: try-lock operations with timeout allowing deadlock recovery, interruptible lock acquisition letting threads respond to interruption while waiting, fairness policies ensuring longest-waiting thread acquires lock next, and multiple condition variables enabling complex coordination.

Example of ReentrantLock advantages:
```java
ReentrantLock lock = new ReentrantLock();
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Got lock within timeout
    } finally {
        lock.unlock();
    }
} else {
    // Timeout, handle appropriately
}
```

Both are reentrant (same thread can reacquire), but ReentrantLock requires explicit unlock in finally blocks to prevent lock leaks.

**Q2: What is a race condition? Provide an example and how to fix it.**

Race conditions occur when multiple threads access shared mutable state concurrently, and the program's correctness depends on execution order. Results become unpredictable as thread scheduling varies.

Classic example: check-then-act pattern:
```java
// Race condition
if (map.containsKey(key)) {  // Check
    value = map.get(key);     // Act
    // Another thread might remove key between check and act!
}

// Fixed with atomic operation
value = map.computeIfPresent(key, (k, v) -> v);
```

Counter increment demonstrates the problem:
```java
count++;  // Actually: temp = count; temp++; count = temp;
```

Two threads might both read count=0, increment to 1, and write 1, losing an increment.

Fix with synchronization (atomic operation), AtomicInteger, or concurrent collections. The solution ensures read-modify-write happens atomically without interleaving.

**Q3: Explain deadlock and how to prevent it.**

Deadlock occurs when threads wait circularly for locks, none progressing. Four conditions must hold: mutual exclusion (locks are exclusive), hold and wait (threads hold locks while requesting others), no preemption (locks can't be forcibly taken), and circular wait (circular chain of lock dependencies).

Prevention strategies: eliminate circular wait through lock ordering (always acquire locks in consistent order), use try-lock with timeout for deadlock recovery, employ lock-free algorithms avoiding locks entirely, or reduce lock scope minimizing hold time.

Example prevention through ordering:
```java
public boolean transfer(Account from, Account to, double amount) {
    // Order locks by ID to prevent circular wait
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    
    synchronized(first) {
        synchronized(second) {
            // Transfer logic
        }
    }
}
```

**Q4: What is the purpose of volatile keyword?**

`volatile` ensures visibility of variable changes across threads without synchronization. Writing a volatile variable immediately flushes to main memory, and reading always retrieves from main memory rather than thread cache.

Use volatile for simple flags or references where reads and writes are independent:
```java
private volatile boolean running = true;

// Writer thread
public void stop() {
    running = false;  // Immediately visible to all threads
}

// Reader thread
while (running) {
    // Will see running=false promptly
}
```

Volatile doesn't provide atomicity for compound operations. `count++` with volatile count isn't thread-safe—the read-modify-write still has a race condition. Use AtomicInteger or synchronization for compound operations.

Volatile also prevents instruction reordering around it, establishing happens-before relationships critical for correct memory visibility in concurrent code.

**Q5: Explain ThreadLocal and when to use it.**

ThreadLocal provides independent variable copies per thread, isolating thread-specific data without synchronization. Each thread accessing a ThreadLocal sees its own value, independent of other threads.

Use ThreadLocal for storing thread-specific context (user session, transaction ID), avoiding parameter passing through deep call stacks, or implementing per-thread buffers/caches.

```java
private static final ThreadLocal<SimpleDateFormat> formatter = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String formatDate(Date date) {
    return formatter.get().format(date);  // Each thread has own formatter
}
```

Caution: ThreadLocal can cause memory leaks in thread pools where threads live long but ThreadLocal values should be cleaned up. Always call `remove()` when done:

```java
try {
    RequestContext.setUserId(userId);
    processRequest();
} finally {
    RequestContext.clear();  // Prevent leak
}
```

**Q6: What is the difference between Callable and Runnable?**

`Runnable` represents tasks without return values and cannot throw checked exceptions. The `run()` method returns void, limiting use to side-effects or setting shared state.

`Callable<V>` represents tasks returning values of type V and can throw checked exceptions. The `call()` method returns V and declares `throws Exception`, enabling error propagation.

```java
// Runnable - no return value
Runnable task = () -> {
    System.out.println("Running task");
    // Cannot return value or throw checked exception
};

// Callable - returns value
Callable<Integer> callable = () -> {
    return expensiveComputation();  // Returns Integer
};

ExecutorService executor = Executors.newFixedThreadPool(2);

executor.submit(task);  // Returns Future<?>, value is null

Future<Integer> future = executor.submit(callable);
Integer result = future.get();  // Retrieve result
```

Use Callable when you need task results or must handle checked exceptions. ExecutorService wraps results in Future allowing asynchronous retrieval.

**Q7: Explain CompletableFuture and its advantages.**

CompletableFuture enables functional-style asynchronous programming with method chaining for composing async operations. Unlike Future's blocking `get()`, CompletableFuture supports non-blocking callbacks triggered on completion.

Advantages include: non-blocking completion handlers, exception handling through `exceptionally()` and `handle()`, combining multiple futures with `thenCombine()` and `allOf()`, transformation with `thenApply()`, and flat-mapping with `thenCompose()`.

```java
CompletableFuture<User> userFuture = CompletableFuture
    .supplyAsync(() -> fetchUser(userId))
    .thenCompose(user -> fetchOrders(user.getId()))  // Flat map
    .thenApply(orders -> enrichOrders(orders))
    .exceptionally(ex -> {
        log.error("Failed to fetch data", ex);
        return Collections.emptyList();
    });

userFuture.thenAccept(orders -> displayOrders(orders));  // Non-blocking
```

CompletableFuture supports async execution in custom executors, timeouts, and combining multiple independent operations, making it ideal for building reactive async pipelines.

**Q8: What are the different types of thread pools?**

Java provides several ExecutorService implementations for different scenarios:

**FixedThreadPool**: Fixed number of threads, bounded queue. Use for predictable loads with defined concurrency level. Threads reused for multiple tasks.

**CachedThreadPool**: Unbounded threads created on demand, reuses idle threads. Use for short-lived async tasks. Threads terminate after 60 seconds idle.

**SingleThreadExecutor**: Single worker thread executing tasks sequentially. Use for sequential task execution with asynchronous submission.

**ScheduledThreadPool**: Supports delayed and periodic task execution. Use for scheduling recurring tasks or delayed execution.

**WorkStealingPool**: Fork/Join pool using available processors, implements work-stealing. Use for parallel computation with divide-and-conquer.

```java
// Fixed - bounded concurrency
ExecutorService fixed = Executors.newFixedThreadPool(10);

// Cached - scales with load
ExecutorService cached = Executors.newCachedThreadPool();

// Single - sequential execution
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled - periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);

// Work-stealing - parallel computation
ExecutorService workStealing = Executors.newWorkStealingPool();
```

Choose based on workload characteristics: predictable load uses fixed, variable load uses cached, sequential needs single, periodic uses scheduled.

**Q9: Explain ConcurrentHashMap and how it differs from Hashtable.**

ConcurrentHashMap provides thread-safe map implementation with fine-grained locking, allowing multiple concurrent updates. It partitions data into segments, locking only affected segments during updates, enabling high concurrency.

Hashtable synchronizes every method on the entire map, allowing only one thread at a time. This becomes a severe bottleneck under concurrent access.

ConcurrentHashMap never throws ConcurrentModificationException during iteration—iterators are weakly consistent, reflecting map state at some point since creation. Hashtable's iterators fail-fast, throwing exception if map modified during iteration.

```java
// ConcurrentHashMap - high concurrency
ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
concurrentMap.put("key", 1);           // Non-blocking for different keys
concurrentMap.computeIfAbsent("key2", k -> expensiveComputation());

// Atomic operations
concurrentMap.merge("key", 1, (old, val) -> old + val);

// Hashtable - poor concurrency
Hashtable<String, Integer> hashtable = new Hashtable<>();
hashtable.put("key", 1);  // Locks entire table

// Every operation synchronized - bottleneck
```

ConcurrentHashMap also provides bulk operations (forEach, search, reduce) utilizing parallelism and atomic compute methods unavailable in Hashtable.

**Q10: What is the Fork/Join framework?**

Fork/Join framework implements work-stealing for divide-and-conquer parallel algorithms. Tasks recursively split into smaller subtasks (fork), processed in parallel, then results combined (join).

Work-stealing means idle threads steal tasks from busy threads' queues, balancing load automatically. This maximizes CPU utilization compared to static work distribution.

Extend RecursiveTask<V> for tasks returning values or RecursiveAction for tasks without return. Implement `compute()` to split work if large enough, otherwise compute directly.

```java
public class ParallelMergeSort extends RecursiveAction {
    private final int[] array;
    private final int start, end;
    private static final int THRESHOLD = 100;
    
    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            Arrays.sort(array, start, end);  // Direct computation
            return;
        }
        
        int mid = (start + end) / 2;
        ParallelMergeSort left = new ParallelMergeSort(array, start, mid);
        ParallelMergeSort right = new ParallelMergeSort(array, mid, end);
        
        invokeAll(left, right);  // Fork both, wait for completion
        merge(array, start, mid, end);
    }
}

ForkJoinPool pool = new ForkJoinPool();
pool.invoke(new ParallelMergeSort(array, 0, array.length));
```

Fork/Join works best for CPU-intensive recursive problems like sorting, tree traversal, or parallel aggregations.

**Q11: Explain happens-before relationships in Java Memory Model.**

Happens-before defines when writes by one thread are guaranteed visible to reads by another thread. Without happens-before relationship, reordering and caching might make writes invisible to other threads.

Key happens-before rules:
- Program order: each action happens-before subsequent actions in same thread
- Monitor lock: unlock happens-before subsequent lock acquisition on same monitor
- Volatile: write to volatile happens-before subsequent reads
- Thread start: thread start happens-before any action in started thread
- Thread termination: actions in thread happen-before detecting termination via join()

These establish memory visibility guarantees:
```java
class Example {
    private int x = 0;
    private volatile boolean ready = false;
    
    // Thread 1
    public void writer() {
        x = 42;          // Write 1
        ready = true;    // Write 2 (volatile)
    }
    
    // Thread 2
    public void reader() {
        if (ready) {     // Read volatile
            System.out.println(x);  // Guaranteed see 42
        }
    }
}
```

Volatile write happens-before subsequent volatile read, and all writes before volatile write happen-before that write, establishing visibility chain.

**Q12: What are the pitfalls of using synchronized blocks excessively?**

Excessive synchronization causes performance bottlenecks by serializing parallel work. Synchronized blocks allow only one thread at a time, wasting CPU cores and reducing throughput.

Holding locks too long blocks other threads, reducing responsiveness. Synchronizing more than necessary limits scalability—coarse-grained locks are easier to reason about but kill concurrency.

Over-synchronization increases deadlock risk as more locks interact. Complex lock dependencies make reasoning difficult.

```java
// Bad - coarse-grained locking
public class Cache {
    private final Map<String, Object> map = new HashMap<>();
    
    public synchronized Object get(String key) {
        return map.get(key);  // All gets serialized!
    }
    
    public synchronized void put(String key, Object value) {
        map.put(key, value);
    }
}

// Better - fine-grained concurrent collection
public class Cache {
    private final ConcurrentHashMap<String, Object> map = 
        new ConcurrentHashMap<>();
    
    public Object get(String key) {
        return map.get(key);  // Multiple concurrent gets
    }
    
    public void put(String key, Object value) {
        map.put(key, value);  // Concurrent updates to different keys
    }
}
```

Use concurrent collections, lock-free algorithms, or fine-grained locking instead of coarse synchronization for better scalability.

**Q13: Explain thread starvation and how to prevent it.**

Thread starvation occurs when threads can't acquire resources (locks, CPU time) needed for progress. Low-priority threads might never execute if high-priority threads continuously run, or unfair lock scheduling might favor certain threads.

Prevention strategies:
- Use fair locks ensuring longest-waiting thread acquires lock next
- Avoid unbounded priority differences between threads
- Set reasonable thread priorities (don't use MIN_PRIORITY)
- Use ExecutorService managing fairness automatically
- Limit lock hold time so other threads get chances

```java
// Fair lock prevents starvation
ReentrantLock fairLock = new ReentrantLock(true);  // Fair mode

public void fairAccess() {
    fairLock.lock();
    try {
        // Longest-waiting thread gets lock next
    } finally {
        fairLock.unlock();
    }
}
```

Monitor thread behavior—if some threads consistently show zero progress while others run, investigate starvation. Thread pools with bounded queues and fair scheduling help prevent starvation in concurrent applications.

**Q14: What is the purpose of CountDownLatch and how does it differ from CyclicBarrier?**

CountDownLatch coordinates threads waiting for operations to complete. Initialize with count, threads call `await()` blocking until count reaches zero via `countDown()` calls. CountDownLatch is single-use—once count reaches zero, it cannot reset.

CyclicBarrier coordinates N threads to wait at a synchronization point. All threads call `await()`, blocking until all reach the barrier, then proceed together. CyclicBarrier resets after release, enabling repeated use for multi-phase operations.

```java
// CountDownLatch - wait for N operations
CountDownLatch latch = new CountDownLatch(3);

// Workers count down
executor.submit(() -> {
    initialize();
    latch.countDown();
});

// Main thread waits
latch.await();  // Blocks until count = 0

// CyclicBarrier - synchronize N threads
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All reached barrier");
});

// Each thread waits for others
for (int phase = 0; phase < 5; phase++) {
    doWork();
    barrier.await();  // Reusable across phases
}
```

Use CountDownLatch when one thread waits for multiple operations to complete. Use CyclicBarrier when multiple threads coordinate at synchronization points repeatedly.

**Q15: Explain the concept of work-stealing in concurrent programming.**

Work-stealing optimizes parallel execution by letting idle threads steal work from busy threads' queues. Each thread has a work queue, processing tasks locally. When a thread's queue empties, it steals tasks from another thread's queue tail (while owner takes from head), balancing load dynamically.

This differs from work-sharing where a central scheduler distributes work, potentially becoming a bottleneck. Work-stealing is decentralized—threads self-balance without coordination overhead.

Fork/Join framework implements work-stealing:
```java
ForkJoinPool pool = new ForkJoinPool(4);  // 4 worker threads

pool.submit(new RecursiveTask<Long>() {
    protected Long compute() {
        if (smallEnough) {
            return computeDirectly();
        }
        
        // Split work
        Task left = new Task();
        Task right = new Task();
        
        left.fork();  // Add to local queue
        long rightResult = right.compute();  // Compute in current thread
        long leftResult = left.join();  // Other thread might steal left
        
        return leftResult + rightResult;
    }
});
```

When threads fork subtasks, they add to their queue. If another thread is idle, it steals these subtasks, utilizing all CPU cores efficiently without manual load balancing. Work-stealing maximizes throughput for divide-and-conquer algorithms with varying subtask sizes.

---

These comprehensive notes cover Java concurrency with production-grade depth, including real-world patterns, common pitfalls, advanced topics, and thorough interview preparation for senior developers.
