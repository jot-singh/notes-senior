# Concurrency Patterns

## Overview
Concurrency patterns address common challenges in multi-threaded programming, including thread safety, synchronization, and efficient resource utilization.

---

## Producer-Consumer Pattern

### Intent
Decouple producers (that generate data) from consumers (that process data) using a shared buffer.

### Implementation

```java
// Using BlockingQueue
public class ProducerConsumer<T> {
    private final BlockingQueue<T> queue;
    private final int capacity;
    
    public ProducerConsumer(int capacity) {
        this.capacity = capacity;
        this.queue = new ArrayBlockingQueue<>(capacity);
    }
    
    public void produce(T item) throws InterruptedException {
        queue.put(item); // Blocks if queue is full
    }
    
    public T consume() throws InterruptedException {
        return queue.take(); // Blocks if queue is empty
    }
    
    public int size() {
        return queue.size();
    }
}

// Producer
public class Producer implements Runnable {
    private final ProducerConsumer<Task> buffer;
    private final String name;
    private volatile boolean running = true;
    
    public Producer(String name, ProducerConsumer<Task> buffer) {
        this.name = name;
        this.buffer = buffer;
    }
    
    @Override
    public void run() {
        int taskId = 0;
        while (running) {
            try {
                Task task = new Task(name + "-task-" + taskId++);
                buffer.produce(task);
                System.out.println(name + " produced: " + task.getId());
                Thread.sleep(100); // Simulate work
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void stop() {
        running = false;
    }
}

// Consumer
public class Consumer implements Runnable {
    private final ProducerConsumer<Task> buffer;
    private final String name;
    private volatile boolean running = true;
    
    public Consumer(String name, ProducerConsumer<Task> buffer) {
        this.name = name;
        this.buffer = buffer;
    }
    
    @Override
    public void run() {
        while (running) {
            try {
                Task task = buffer.consume();
                System.out.println(name + " processing: " + task.getId());
                task.execute();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public void stop() {
        running = false;
    }
}

// Usage
ProducerConsumer<Task> buffer = new ProducerConsumer<>(10);

List<Producer> producers = List.of(
    new Producer("P1", buffer),
    new Producer("P2", buffer)
);

List<Consumer> consumers = List.of(
    new Consumer("C1", buffer),
    new Consumer("C2", buffer),
    new Consumer("C3", buffer)
);

ExecutorService executor = Executors.newFixedThreadPool(5);
producers.forEach(executor::submit);
consumers.forEach(executor::submit);
```

---

## Thread Pool Pattern

### Intent
Manage a pool of worker threads to execute tasks, avoiding the overhead of creating new threads.

### Implementation

```java
// Custom Thread Pool
public class CustomThreadPool {
    private final int poolSize;
    private final BlockingQueue<Runnable> taskQueue;
    private final List<WorkerThread> workers;
    private volatile boolean isShutdown = false;
    
    public CustomThreadPool(int poolSize, int queueCapacity) {
        this.poolSize = poolSize;
        this.taskQueue = new LinkedBlockingQueue<>(queueCapacity);
        this.workers = new ArrayList<>();
        
        for (int i = 0; i < poolSize; i++) {
            WorkerThread worker = new WorkerThread("Worker-" + i);
            workers.add(worker);
            worker.start();
        }
    }
    
    public void submit(Runnable task) {
        if (isShutdown) {
            throw new IllegalStateException("Thread pool is shutdown");
        }
        try {
            taskQueue.put(task);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    public <T> Future<T> submit(Callable<T> callable) {
        FutureTask<T> futureTask = new FutureTask<>(callable);
        submit(futureTask);
        return futureTask;
    }
    
    public void shutdown() {
        isShutdown = true;
        for (WorkerThread worker : workers) {
            worker.interrupt();
        }
    }
    
    public void awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        long deadline = System.currentTimeMillis() + unit.toMillis(timeout);
        for (WorkerThread worker : workers) {
            long remaining = deadline - System.currentTimeMillis();
            if (remaining > 0) {
                worker.join(remaining);
            }
        }
    }
    
    private class WorkerThread extends Thread {
        public WorkerThread(String name) {
            super(name);
        }
        
        @Override
        public void run() {
            while (!isShutdown || !taskQueue.isEmpty()) {
                try {
                    Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
                    if (task != null) {
                        task.run();
                    }
                } catch (InterruptedException e) {
                    if (isShutdown) break;
                } catch (Exception e) {
                    System.err.println(getName() + " caught exception: " + e.getMessage());
                }
            }
        }
    }
}

// Usage
CustomThreadPool pool = new CustomThreadPool(4, 100);

for (int i = 0; i < 20; i++) {
    final int taskId = i;
    pool.submit(() -> {
        System.out.println(Thread.currentThread().getName() + " executing task " + taskId);
        Thread.sleep(100);
    });
}

pool.shutdown();
pool.awaitTermination(5, TimeUnit.SECONDS);
```

---

## Read-Write Lock Pattern

### Intent
Allow concurrent read access while ensuring exclusive write access.

### Implementation

```java
// Thread-safe cache with read-write lock
public class ReadWriteCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();
    
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    public V getOrCompute(K key, Function<K, V> computeFunction) {
        // Try read lock first
        readLock.lock();
        try {
            V value = cache.get(key);
            if (value != null) {
                return value;
            }
        } finally {
            readLock.unlock();
        }
        
        // Need write lock to compute and store
        writeLock.lock();
        try {
            // Double-check after acquiring write lock
            V value = cache.get(key);
            if (value == null) {
                value = computeFunction.apply(key);
                cache.put(key, value);
            }
            return value;
        } finally {
            writeLock.unlock();
        }
    }
    
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
    
    public V remove(K key) {
        writeLock.lock();
        try {
            return cache.remove(key);
        } finally {
            writeLock.unlock();
        }
    }
    
    public void clear() {
        writeLock.lock();
        try {
            cache.clear();
        } finally {
            writeLock.unlock();
        }
    }
    
    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
    
    public Map<K, V> snapshot() {
        readLock.lock();
        try {
            return new HashMap<>(cache);
        } finally {
            readLock.unlock();
        }
    }
}
```

---

## Future Pattern

### Intent
Represent a result that will be available in the future, allowing asynchronous operations.

### Implementation

```java
// Custom Future implementation
public class CustomFuture<T> {
    private T result;
    private Exception exception;
    private volatile boolean done = false;
    private final CountDownLatch latch = new CountDownLatch(1);
    
    public void complete(T result) {
        this.result = result;
        this.done = true;
        latch.countDown();
    }
    
    public void completeExceptionally(Exception exception) {
        this.exception = exception;
        this.done = true;
        latch.countDown();
    }
    
    public T get() throws InterruptedException, ExecutionException {
        latch.await();
        if (exception != null) {
            throw new ExecutionException(exception);
        }
        return result;
    }
    
    public T get(long timeout, TimeUnit unit) 
            throws InterruptedException, ExecutionException, TimeoutException {
        if (!latch.await(timeout, unit)) {
            throw new TimeoutException("Future timed out");
        }
        if (exception != null) {
            throw new ExecutionException(exception);
        }
        return result;
    }
    
    public boolean isDone() {
        return done;
    }
}

// Async service using CompletableFuture
public class AsyncOrderService {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public CompletableFuture<Order> createOrderAsync(OrderRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // Validate
            validateRequest(request);
            return request;
        }, executor)
        .thenCompose(req -> checkInventoryAsync(req))
        .thenCompose(req -> processPaymentAsync(req))
        .thenApply(req -> createOrder(req))
        .exceptionally(ex -> {
            log.error("Order creation failed", ex);
            throw new OrderException("Failed to create order", ex);
        });
    }
    
    public CompletableFuture<OrderDetails> getOrderDetailsAsync(String orderId) {
        CompletableFuture<Order> orderFuture = getOrderAsync(orderId);
        CompletableFuture<Customer> customerFuture = orderFuture
            .thenCompose(order -> getCustomerAsync(order.getCustomerId()));
        CompletableFuture<List<Product>> productsFuture = orderFuture
            .thenCompose(order -> getProductsAsync(order.getProductIds()));
        
        return orderFuture.thenCombine(customerFuture, (order, customer) -> 
            new OrderDetails(order, customer, null))
            .thenCombine(productsFuture, (details, products) -> 
                details.withProducts(products));
    }
    
    // Multiple async calls with allOf
    public CompletableFuture<Dashboard> getDashboardAsync(String userId) {
        CompletableFuture<UserProfile> profileFuture = getUserProfileAsync(userId);
        CompletableFuture<List<Order>> ordersFuture = getRecentOrdersAsync(userId);
        CompletableFuture<List<Notification>> notificationsFuture = 
            getNotificationsAsync(userId);
        CompletableFuture<Stats> statsFuture = getUserStatsAsync(userId);
        
        return CompletableFuture.allOf(
            profileFuture, ordersFuture, notificationsFuture, statsFuture
        ).thenApply(v -> new Dashboard(
            profileFuture.join(),
            ordersFuture.join(),
            notificationsFuture.join(),
            statsFuture.join()
        ));
    }
}
```

---

## Semaphore Pattern

### Intent
Control access to a shared resource by limiting the number of concurrent accesses.

### Implementation

```java
// Connection pool using Semaphore
public class ConnectionPool {
    private final Semaphore semaphore;
    private final BlockingQueue<Connection> pool;
    private final int maxConnections;
    
    public ConnectionPool(String url, int maxConnections) {
        this.maxConnections = maxConnections;
        this.semaphore = new Semaphore(maxConnections, true);
        this.pool = new LinkedBlockingQueue<>(maxConnections);
        
        // Initialize connections
        for (int i = 0; i < maxConnections; i++) {
            pool.offer(createConnection(url));
        }
    }
    
    public Connection acquire() throws InterruptedException {
        semaphore.acquire();
        return pool.poll();
    }
    
    public Connection acquire(long timeout, TimeUnit unit) 
            throws InterruptedException, TimeoutException {
        if (!semaphore.tryAcquire(timeout, unit)) {
            throw new TimeoutException("Could not acquire connection within timeout");
        }
        return pool.poll();
    }
    
    public void release(Connection connection) {
        if (connection != null) {
            pool.offer(connection);
            semaphore.release();
        }
    }
    
    public int availableConnections() {
        return semaphore.availablePermits();
    }
    
    private Connection createConnection(String url) {
        // Create database connection
        return DriverManager.getConnection(url);
    }
}

// Rate limiter using Semaphore
public class RateLimiter {
    private final Semaphore semaphore;
    private final int permitsPerSecond;
    private final ScheduledExecutorService scheduler;
    
    public RateLimiter(int permitsPerSecond) {
        this.permitsPerSecond = permitsPerSecond;
        this.semaphore = new Semaphore(permitsPerSecond);
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        
        // Replenish permits every second
        scheduler.scheduleAtFixedRate(() -> {
            int permitsToAdd = permitsPerSecond - semaphore.availablePermits();
            if (permitsToAdd > 0) {
                semaphore.release(permitsToAdd);
            }
        }, 1, 1, TimeUnit.SECONDS);
    }
    
    public boolean tryAcquire() {
        return semaphore.tryAcquire();
    }
    
    public void acquire() throws InterruptedException {
        semaphore.acquire();
    }
    
    public void shutdown() {
        scheduler.shutdown();
    }
}
```

---

## Active Object Pattern

### Intent
Decouple method execution from method invocation for objects that each reside in their own thread of control.

### Implementation

```java
// Active Object
public class ActiveObject {
    private final BlockingQueue<Runnable> taskQueue;
    private final Thread workerThread;
    private volatile boolean running = true;
    
    public ActiveObject() {
        this.taskQueue = new LinkedBlockingQueue<>();
        this.workerThread = new Thread(this::processQueue, "ActiveObject");
        this.workerThread.start();
    }
    
    private void processQueue() {
        while (running) {
            try {
                Runnable task = taskQueue.poll(100, TimeUnit.MILLISECONDS);
                if (task != null) {
                    task.run();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    public <T> CompletableFuture<T> invoke(Callable<T> callable) {
        CompletableFuture<T> future = new CompletableFuture<>();
        taskQueue.offer(() -> {
            try {
                T result = callable.call();
                future.complete(result);
            } catch (Exception e) {
                future.completeExceptionally(e);
            }
        });
        return future;
    }
    
    public void invoke(Runnable runnable) {
        taskQueue.offer(runnable);
    }
    
    public void shutdown() {
        running = false;
        workerThread.interrupt();
    }
}

// Example: Thread-safe logger using Active Object
public class AsyncLogger {
    private final ActiveObject activeObject;
    private final PrintWriter writer;
    private final DateTimeFormatter formatter;
    
    public AsyncLogger(String filename) throws IOException {
        this.activeObject = new ActiveObject();
        this.writer = new PrintWriter(new FileWriter(filename, true), true);
        this.formatter = DateTimeFormatter.ISO_LOCAL_DATE_TIME;
    }
    
    public void log(String level, String message) {
        activeObject.invoke(() -> {
            String timestamp = LocalDateTime.now().format(formatter);
            writer.printf("[%s] [%s] %s%n", timestamp, level, message);
        });
    }
    
    public void info(String message) { log("INFO", message); }
    public void warn(String message) { log("WARN", message); }
    public void error(String message) { log("ERROR", message); }
    
    public CompletableFuture<Void> flush() {
        return activeObject.invoke(() -> {
            writer.flush();
            return null;
        });
    }
    
    public void close() {
        activeObject.invoke(writer::close);
        activeObject.shutdown();
    }
}
```

---

## Barrier Pattern

### Intent
Synchronize multiple threads at a common point before proceeding.

### Implementation

```java
// Parallel data processing with CyclicBarrier
public class ParallelProcessor {
    private final int numWorkers;
    private final CyclicBarrier barrier;
    private final ExecutorService executor;
    private final List<DataChunk> chunks;
    private final AtomicReference<ProcessingResult> result;
    
    public ParallelProcessor(int numWorkers) {
        this.numWorkers = numWorkers;
        this.executor = Executors.newFixedThreadPool(numWorkers);
        this.chunks = new CopyOnWriteArrayList<>();
        this.result = new AtomicReference<>();
        
        // Barrier with merge action
        this.barrier = new CyclicBarrier(numWorkers, this::mergeResults);
    }
    
    public void process(List<Data> data) throws InterruptedException {
        // Split data into chunks
        List<List<Data>> partitions = partition(data, numWorkers);
        
        // Submit workers
        for (int i = 0; i < numWorkers; i++) {
            final int workerId = i;
            final List<Data> partition = partitions.get(i);
            executor.submit(() -> {
                try {
                    // Process partition
                    DataChunk chunk = processChunk(partition);
                    chunks.add(chunk);
                    
                    // Wait for all workers
                    barrier.await();
                    
                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
    }
    
    private void mergeResults() {
        // Called when all threads reach barrier
        ProcessingResult merged = chunks.stream()
            .reduce(ProcessingResult.empty(), ProcessingResult::merge);
        result.set(merged);
        chunks.clear();
    }
    
    private DataChunk processChunk(List<Data> data) {
        // Process data chunk
        return new DataChunk(data.stream()
            .map(this::processItem)
            .toList());
    }
}

// Phaser example - more flexible than CyclicBarrier
public class PhasedComputation {
    private final Phaser phaser;
    private final List<ComputationTask> tasks;
    
    public PhasedComputation(int numTasks) {
        this.phaser = new Phaser(1); // Register self
        this.tasks = new ArrayList<>();
        
        for (int i = 0; i < numTasks; i++) {
            phaser.register();
            tasks.add(new ComputationTask(i, phaser));
        }
    }
    
    public void execute() {
        ExecutorService executor = Executors.newFixedThreadPool(tasks.size());
        
        for (ComputationTask task : tasks) {
            executor.submit(task);
        }
        
        // Wait for all phases
        while (!phaser.isTerminated()) {
            int phase = phaser.arriveAndAwaitAdvance();
            System.out.println("Phase " + phase + " completed");
        }
        
        executor.shutdown();
    }
    
    private class ComputationTask implements Runnable {
        private final int id;
        private final Phaser phaser;
        
        ComputationTask(int id, Phaser phaser) {
            this.id = id;
            this.phaser = phaser;
        }
        
        @Override
        public void run() {
            // Phase 1: Initialize
            initialize();
            phaser.arriveAndAwaitAdvance();
            
            // Phase 2: Compute
            compute();
            phaser.arriveAndAwaitAdvance();
            
            // Phase 3: Finalize
            finalize();
            phaser.arriveAndDeregister();
        }
    }
}
```

---

## Double-Checked Locking

### Intent
Reduce locking overhead while ensuring thread-safe lazy initialization.

```java
// Classic double-checked locking
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// Using Holder pattern (preferred)
public class SingletonHolder {
    private SingletonHolder() {}
    
    private static class Holder {
        static final SingletonHolder INSTANCE = new SingletonHolder();
    }
    
    public static SingletonHolder getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## Comparison Table

| Pattern | Use Case | Key Classes |
|---------|----------|-------------|
| **Producer-Consumer** | Async task processing | BlockingQueue |
| **Thread Pool** | Manage worker threads | ExecutorService |
| **Read-Write Lock** | Concurrent reads, exclusive writes | ReadWriteLock |
| **Future** | Async results | CompletableFuture |
| **Semaphore** | Limit concurrent access | Semaphore |
| **Active Object** | Thread-safe object | BlockingQueue + Thread |
| **Barrier** | Synchronization point | CyclicBarrier, Phaser |

---

## Related Topics
- [[01-lld-fundamentals]] - SOLID principles
- [[02-creational-patterns]] - Singleton pattern
- [[04-behavioral-patterns]] - Command pattern
