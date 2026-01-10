# Java Memory Model

## Concept Overview

The Java Memory Model (JMM) defines how threads interact through memory and what behaviors are allowed in concurrent programs. It specifies when changes made by one thread become visible to others, enabling both optimizations and correct concurrent programming.

## Core Mechanics & Implementation

### Memory Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Main Memory                              │
│                  (Shared Variables)                          │
└─────────────────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
    │ Thread 1 │    │ Thread 2 │    │ Thread 3 │
    │  Cache   │    │  Cache   │    │  Cache   │
    │ (Working │    │ (Working │    │ (Working │
    │  Memory) │    │  Memory) │    │  Memory) │
    └─────────┘    └─────────┘    └─────────┘
```

Each thread has working memory (cache) that may hold copies of variables. Changes may not immediately propagate to main memory or other threads' caches without proper synchronization.

### Visibility Problem

```java
// Without synchronization, thread 2 may never see update
public class VisibilityProblem {
    private boolean running = true;  // Not volatile
    
    public void stop() {
        running = false;  // Write in thread 1
    }
    
    public void run() {
        while (running) {  // Read in thread 2 - may never see false!
            doWork();
        }
    }
}

// Fix with volatile
public class FixedVisibility {
    private volatile boolean running = true;
    
    public void stop() {
        running = false;  // Volatile write - visible to all threads
    }
    
    public void run() {
        while (running) {  // Volatile read - sees latest value
            doWork();
        }
    }
}
```

### Happens-Before Relationship

Happens-before defines when writes are guaranteed visible to subsequent reads.

```java
// 1. Program Order: actions in thread happen-before subsequent actions
int x = 1;  // Action A
int y = 2;  // Action B - A happens-before B in same thread

// 2. Monitor Lock: unlock happens-before subsequent lock
synchronized(lock) {
    x = 1;
}  // Unlock happens-before...

synchronized(lock) {  // ...this lock acquisition
    int y = x;  // Guaranteed to see x = 1
}

// 3. Volatile: write happens-before subsequent read
volatile int flag;
int data;

// Thread 1
data = 42;      // Write data
flag = 1;       // Volatile write

// Thread 2
if (flag == 1) {     // Volatile read
    int x = data;    // Guaranteed to see 42
}

// 4. Thread Start: start() happens-before any action in started thread
data = 42;
thread.start();  // In new thread, guaranteed to see data = 42

// 5. Thread Join: actions in thread happen-before join() returns
thread.join();
int x = data;  // Sees all writes from thread
```

### Volatile Keyword

```java
public class VolatileExample {
    private volatile int counter = 0;
    private volatile boolean ready = false;
    
    // Volatile guarantees:
    // 1. Visibility - writes immediately visible to all threads
    // 2. Ordering - prevents reordering across volatile access
    
    // Does NOT guarantee atomicity!
    public void increment() {
        counter++;  // NOT atomic! Read-modify-write race possible
    }
    
    // Correct usage: single writer
    public void publish() {
        // Non-volatile writes before volatile write
        data = computeData();
        moreData = computeMore();
        ready = true;  // Volatile write - all above visible to readers
    }
    
    public void consume() {
        if (ready) {  // Volatile read
            // All writes before ready=true are visible
            use(data);
            use(moreData);
        }
    }
}
```

### Synchronization Mechanisms

```java
// synchronized - visibility + atomicity + mutual exclusion
public class SynchronizedCounter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;  // Atomic - only one thread at a time
    }
    
    public synchronized int get() {
        return count;  // Sees latest value
    }
}

// Lock - explicit synchronization
public class LockCounter {
    private final Lock lock = new ReentrantLock();
    private int count = 0;
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }
}

// Atomic variables - lock-free operations
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();  // Atomic CAS operation
    }
    
    public int get() {
        return count.get();
    }
}
```

### Double-Checked Locking

```java
// Broken without volatile (before Java 5 fixes)
public class Singleton {
    private static Singleton instance;  // Must be volatile!
    
    public static Singleton getInstance() {
        if (instance == null) {                // First check
            synchronized (Singleton.class) {
                if (instance == null) {        // Second check
                    instance = new Singleton();  // Problem: may see partial object
                }
            }
        }
        return instance;
    }
}

// Fixed with volatile
public class SingletonFixed {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  // Safe with volatile
                }
            }
        }
        return instance;
    }
}

// Better: Initialization-on-demand holder
public class SingletonHolder {
    private SingletonHolder() {}
    
    private static class Holder {
        static final SingletonHolder INSTANCE = new SingletonHolder();
    }
    
    public static SingletonHolder getInstance() {
        return Holder.INSTANCE;  // Thread-safe, lazy
    }
}
```

### Memory Barriers

```java
// Volatile read: LoadLoad + LoadStore barrier
// Volatile write: StoreStore + StoreLoad barrier

private volatile boolean flag;
private int data;

// Thread 1
data = 42;           // Store
flag = true;         // Volatile Store
// StoreStore: data write completes before flag write
// StoreLoad: flag write visible before any subsequent load

// Thread 2
if (flag) {          // Volatile Load
    // LoadLoad: flag load completes before data load
    // LoadStore: flag load completes before any subsequent store
    int x = data;    // Load - guaranteed to see 42
}
```

## Common Pitfalls

```java
// 1. Assuming visibility without synchronization
private int count = 0;  // Not volatile!
// Thread 1 increments, Thread 2 may never see updates

// 2. Thinking volatile is enough for compound operations
private volatile int count = 0;
count++;  // NOT atomic! Race condition exists

// 3. Lock on mutable object
private Object lock = new Object();
synchronized(lock) {
    lock = new Object();  // Now synchronizing on different object!
}

// 4. Synchronizing on different locks
synchronized(lock1) { count++; }  // Thread 1
synchronized(lock2) { count++; }  // Thread 2 - no synchronization!
```

## Interview Questions

**Q1: What is the Java Memory Model?**

JMM defines how threads interact through memory, specifying when writes become visible to reads. It allows CPU/compiler optimizations while providing rules for correct concurrent programming. Key concept is happens-before relationship establishing visibility guarantees.

**Q2: What is happens-before relationship?**

Guarantees that writes before action A are visible after action B. Established by: program order within thread, synchronized block exit→entry, volatile write→read, thread start/join, final field initialization. Transitive: if A→B and B→C, then A→C.

**Q3: What does volatile guarantee?**

Visibility: writes immediately visible to all threads. Ordering: prevents reordering across volatile access. Does NOT guarantee atomicity. Use for flags, publication of immutable objects, single-writer patterns. Compound operations still need synchronization.

**Q4: Why is double-checked locking broken without volatile?**

Without volatile, object reference may be written before constructor completes (reordering). Thread may see non-null reference to partially constructed object. Volatile prevents this reordering by establishing happens-before between write and read.

**Q5: What is the difference between visibility and atomicity?**

Visibility: when writes become visible to other threads. Volatile provides visibility. Atomicity: operations execute as single unit, not interleaved. synchronized provides both. Volatile counter increment is visible but not atomic (race condition).

**Q6: How does synchronized provide memory visibility?**

Unlock happens-before subsequent lock on same monitor. All writes before unlock are visible after lock acquisition. Also provides mutual exclusion (atomicity). Memory barriers ensure write flush and read refresh.

**Q7: What are memory barriers?**

CPU instructions preventing reordering across barrier. LoadLoad: loads before complete before loads after. StoreStore: stores before complete before stores after. Volatile read: LoadLoad+LoadStore. Volatile write: StoreStore+StoreLoad.

**Q8: When to use volatile vs synchronized?**

Volatile: single flag variables, publication of immutable objects, status indicators, when only visibility needed. Synchronized: compound operations (check-then-act, read-modify-write), multiple variables updated together, when atomicity needed.
