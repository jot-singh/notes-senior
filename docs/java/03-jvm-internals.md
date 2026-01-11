# JVM Internals

## Concept Overview

The Java Virtual Machine (JVM) executes Java bytecode, providing platform independence, automatic memory management, and runtime optimizations. Understanding JVM internals enables performance tuning, memory leak diagnosis, and production troubleshooting critical for senior developers.

The JVM architecture consists of three main components: Class Loader Subsystem loads bytecode, Runtime Data Areas store program data, and Execution Engine executes instructions. The JVM specification allows different implementations (HotSpot, OpenJ9, GraalVM) with varying performance characteristics.

## Core Mechanics & Implementation

### JVM Memory Structure

```
┌─────────────────────────────────────────────────────────────┐
│                        JVM Memory                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Heap (Shared)                      │   │
│  │  ┌─────────────────┐  ┌───────────────────────────┐  │   │
│  │  │   Young Gen     │  │       Old Gen             │  │   │
│  │  │ ┌─────┬───────┐ │  │                           │  │   │
│  │  │ │Eden │S0 │S1 │ │  │   Tenured Objects         │  │   │
│  │  │ └─────┴───────┘ │  │                           │  │   │
│  │  └─────────────────┘  └───────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────┐  ┌─────────────────────────────────┐  │
│  │ Metaspace        │  │  Thread Stacks (per thread)    │  │
│  │ (Class metadata) │  │  ┌─────┐ ┌─────┐ ┌─────┐       │  │
│  └──────────────────┘  │  │Stack│ │Stack│ │Stack│       │  │
│                        │  └─────┘ └─────┘ └─────┘       │  │
│                        └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Heap**: Shared memory for all objects and arrays. Divided into Young Generation (Eden + Survivor spaces) and Old Generation. Objects start in Eden, survive collections to Survivor spaces, eventually promote to Old Gen.

**Metaspace**: Stores class metadata (class structures, method data). Replaced PermGen in Java 8, uses native memory with auto-growth.

**Stack**: Per-thread memory storing method frames with local variables and operand stack. Fixed size per thread (default ~1MB).

**Program Counter**: Per-thread register tracking current instruction.

**Native Method Stack**: Stores native method calls (JNI).

### Class Loading

```java
// Class loader hierarchy
// Bootstrap ClassLoader -> Extension/Platform ClassLoader -> Application ClassLoader

public class ClassLoadingDemo {
    public static void main(String[] args) {
        // Check class loaders
        System.out.println(String.class.getClassLoader());     // null (Bootstrap)
        System.out.println(ClassLoadingDemo.class.getClassLoader()); // AppClassLoader
        
        // Custom class loader
        ClassLoader customLoader = new ClassLoader() {
            @Override
            protected Class<?> findClass(String name) throws ClassNotFoundException {
                byte[] bytecode = loadBytecode(name);
                return defineClass(name, bytecode, 0, bytecode.length);
            }
        };
    }
}
```

Class loading phases: Loading (read bytecode) → Linking (verify, prepare, resolve) → Initialization (execute static initializers).

### Garbage Collection

**GC Algorithms**:
- **Serial GC**: Single-threaded, stop-the-world. Use for small heaps.
- **Parallel GC**: Multi-threaded throughput collector. Default in Java 8.
- **G1 GC**: Region-based, predictable pause times. Default in Java 9+.
- **ZGC/Shenandoah**: Ultra-low latency (<10ms pauses), large heaps.

```bash
# GC selection flags
-XX:+UseSerialGC
-XX:+UseParallelGC
-XX:+UseG1GC
-XX:+UseZGC
-XX:+UseShenandoahGC

# G1 tuning
-XX:MaxGCPauseMillis=200        # Target pause time
-XX:G1HeapRegionSize=16m        # Region size
-XX:InitiatingHeapOccupancyPercent=45  # When to start marking
```

**GC Roots**: Objects that GC cannot collect, used as starting points for reachability analysis:
- Local variables in active threads
- Static variables
- JNI references
- Active threads

### JIT Compilation

The JVM interprets bytecode initially, then JIT-compiles hot methods to native code for performance.

```bash
# JIT flags
-XX:+TieredCompilation           # Use tiered compilation (default)
-XX:CompileThreshold=10000       # Invocations before compile
-XX:+PrintCompilation            # Log compilations
-XX:-UseCompressedOops           # Disable compressed pointers (>32GB heap)
```

**Tiered Compilation Levels**:
- Level 0: Interpreter
- Level 1-3: C1 compiler (fast compilation)
- Level 4: C2 compiler (optimized code)

## Best Practices & Patterns

### JVM Tuning Guidelines

```bash
# Production JVM settings
java \
  -Xms4g -Xmx4g \              # Fixed heap size (avoid resize pauses)
  -XX:+UseG1GC \               # G1 for balanced throughput/latency
  -XX:MaxGCPauseMillis=200 \   # Target pause time
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/heap.hprof \
  -Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=10m \
  -jar application.jar
```

### Memory Sizing

- Set `-Xms` equal to `-Xmx` to avoid resize pauses
- Heap should be 50-70% of available RAM (leave room for Metaspace, native memory)
- Young Gen typically 1/3 to 1/4 of heap
- Monitor and adjust based on GC logs

### Monitoring Commands

```bash
# JVM process info
jps -lvm                        # List Java processes

# Memory and GC info
jstat -gc <pid> 1000            # GC stats every second
jstat -gcutil <pid>             # GC utilization percentages

# Thread dump
jstack <pid> > thread_dump.txt

# Heap dump
jmap -dump:format=b,file=heap.hprof <pid>

# Heap histogram
jmap -histo <pid> | head -20
```

## Common Pitfalls

### Memory Leaks

```java
// Common leak: unbounded cache
public class LeakyCache {
    private static Map<String, Object> cache = new HashMap<>();  // Never cleared!
    
    public void addToCache(String key, Object value) {
        cache.put(key, value);  // Grows forever
    }
}

// Fix: Use bounded cache or weak references
private static Map<String, Object> cache = new LinkedHashMap<>(100, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 100;
    }
};

// Or use WeakHashMap for keys that can be GC'd
private static Map<String, Object> cache = new WeakHashMap<>();
```

### OutOfMemoryError Types

```
java.lang.OutOfMemoryError: Java heap space     → Increase -Xmx or fix leak
java.lang.OutOfMemoryError: Metaspace           → Increase -XX:MaxMetaspaceSize
java.lang.OutOfMemoryError: GC overhead limit   → Too much time in GC, fix leak
java.lang.OutOfMemoryError: unable to create thread → Reduce -Xss or increase ulimit
```

## Interview Questions

**Q1: Explain the difference between stack and heap memory.**

Stack is per-thread, stores method frames with local variables and references. Fixed size, LIFO order, automatically managed. Heap is shared across threads, stores all objects and arrays, managed by garbage collector. Stack access is faster but limited in size; heap is larger but requires GC.

**Q2: What are the different types of garbage collectors?**

Serial GC: single-threaded, small heaps. Parallel GC: multi-threaded throughput, default Java 8. G1 GC: region-based, predictable pauses, default Java 9+. ZGC/Shenandoah: ultra-low latency (<10ms), large heaps. Choose based on latency requirements and heap size.

**Q3: Explain class loading mechanism.**

Bootstrap loader loads core Java classes (rt.jar). Platform/Extension loader loads extension classes. Application loader loads classpath classes. Custom loaders can load from custom sources. Parent delegation: child asks parent first, loads only if parent can't find class.

**Q4: What causes OutOfMemoryError?**

Heap space: objects exceed heap limit—increase heap or fix leak. Metaspace: too many classes loaded—increase Metaspace or fix class loader leak. GC overhead: spending >98% time in GC—fix memory leak. Unable to create thread: too many threads—reduce stack size or increase OS limits.

**Q5: How does JIT compilation work?**

JVM interprets bytecode initially, profiling execution. Hot methods (frequently called) are compiled to native code. Tiered compilation: C1 compiles quickly with basic optimizations, C2 produces highly optimized code for hottest methods. Optimizations include inlining, escape analysis, loop unrolling.

**Q6: What are GC roots?**

Objects that GC cannot collect: local variables in active stack frames, static variables, JNI references, active threads, synchronized monitors. GC traces from roots marking reachable objects; unreachable objects are collected.

**Q7: Explain G1 garbage collector.**

G1 divides heap into equal-sized regions, collects garbage-first regions (most garbage). Concurrent marking identifies live objects. Mixed collections reclaim old gen regions incrementally. Targets pause time via `-XX:MaxGCPauseMillis`. Evacuates live objects to new regions, compacting memory.

**Q8: How to diagnose memory leaks?**

Enable heap dumps on OOM. Take heap dumps during runtime with jmap. Analyze with tools (Eclipse MAT, VisualVM). Look for growing collections, retained objects, dominator trees. Common causes: unbounded caches, listeners not removed, static collections, ThreadLocal not cleared.

**Q9: What is the difference between Young and Old generation?**

Young Gen holds new objects in Eden space. Minor GC collects Eden, survivors move to Survivor spaces. After surviving multiple GCs, objects promote to Old Gen. Old Gen holds long-lived objects, collected less frequently (Major/Full GC). Young Gen collection is fast; Old Gen collection is slower.

**Q10: Explain escape analysis.**

JIT optimization determining if objects escape method scope. Non-escaping objects can be stack-allocated (avoiding heap), scalar-replaced (fields as local variables), or have synchronization eliminated. Reduces GC pressure and improves performance for short-lived objects.

**Q11: What JVM flags would you use for production?**

Fixed heap (`-Xms=-Xmx`), appropriate GC (G1 for general, ZGC for low latency), heap dump on OOM, GC logging, container awareness (`-XX:+UseContainerSupport`). Monitor and tune based on application characteristics and SLAs.

**Q12: How does Metaspace differ from PermGen?**

PermGen was fixed-size heap area for class metadata, causing OOM when full. Metaspace uses native memory, auto-grows (configurable max), class metadata released when classloader is GC'd. Metaspace eliminates PermGen tuning issues but can still leak with custom classloaders.
