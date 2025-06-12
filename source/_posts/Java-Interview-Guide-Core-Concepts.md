---
title: Java Interview Guide - Core Concepts
date: 2025-06-12 15:44:19
tags: [java]
categories: [java]
---

## Java Fundamentals

### Collection Framework

**Key Concepts:** Thread safety, iterators, duplicates, ordered collections, key-value pairs

#### List Implementations

**Vector**
- Thread-safe with strong consistency
- 2x capacity expansion
- `Collections.synchronizedList()` requires manual synchronization for iterators

**ArrayList**
- Array-based implementation
- Inefficient head insertion, efficient tail insertion
- 1.5x capacity expansion
- Pre-specify capacity to reduce expansion overhead
- Uses `transient` for object array to avoid serializing empty slots

**LinkedList**
- Doubly-linked list implementation
- Slowest for middle insertions
- Use Iterator for traversal
- `remove(Integer)` removes element, `remove(int)` removes by index

**Iterator Safety**
- Enhanced for-loop uses iterator internally
- Concurrent modification throws `ConcurrentModificationException`
- Use iterator's `remove()` method instead of collection's

#### Map Implementations

**Hashtable**
- Thread-safe using `synchronized`
- No null keys/values allowed

**HashMap**
- Not thread-safe, allows null keys/values
- Array + linked list structure
- Hash collision resolution via chaining
- Default capacity: 16, load factor: 0.75
- JDK 1.8: Converts to red-black tree when chain length > 8 and capacity > 64
- Multi-threading can cause infinite loops during rehashing

```java
// Hash calculation for efficient indexing
index = (n - 1) & hash(key)

// Hash function reduces collisions
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**LinkedHashMap**
- HashMap + doubly-linked list
- Maintains insertion/access order
- Used for LRU cache implementation

**TreeMap**
- Red-black tree based
- O(log n) time complexity
- Sorted keys via Comparator/Comparable

### I/O Models

**Java I/O Hierarchy**
- Base classes: `InputStream/Reader`, `OutputStream/Writer`
- Decorator pattern implementation

**BIO (Blocking I/O)**
- Synchronous blocking model
- One thread per connection
- Suitable for low-concurrency scenarios
- Thread pool provides natural rate limiting

**NIO (Non-blocking I/O)**
- I/O multiplexing model (not traditional NIO)
- Uses `select/epoll` system calls
- Single thread monitors multiple file descriptors
- Core components: Buffer, Channel, Selector

**AIO (Asynchronous I/O)**
- Event-driven with callbacks
- Introduced in Java 7 as NIO.2
- Direct return without blocking
- Limited adoption in practice

### Thread Pool

**Thread Creation Methods**
1. Runnable interface
2. Thread class
3. Callable + FutureTask
4. Thread pools (recommended)

**Core Parameters**
- `corePoolSize`: Core thread count
- `maximumPoolSize`: Maximum thread count
- `keepAliveTime`: Idle thread survival time
- `workQueue`: Task queue (must be bounded)
- `rejectedExecutionHandler`: Rejection policy

**Execution Flow**
1. Create threads up to core size
2. Queue tasks when core threads busy
3. Expand to max size when queue full
4. Apply rejection policy when max reached
5. Shrink to core size after keepAliveTime

**Optimal Thread Count**
```
Optimal Threads = CPU Cores × [1 + (I/O Time / CPU Time)]
```

**Best Practices**
- Graceful shutdown
- Uncaught exception handling
- Separate pools for dependent tasks
- Proper rejection policy implementation

### ThreadLocal

**Use Cases**
- Thread isolation with cross-method data sharing
- Database session management (Hibernate)
- Request context propagation (user ID, session)
- HTTP request instances

**Implementation**
```
Thread → ThreadLocalMap<ThreadLocal, Object> → Entry(key, value)
```

**Memory Management**
- Entry key uses weak reference
- Automatic cleanup of null keys
- Must use `private static final` modifiers

**Best Practices**
- Always call `remove()` in finally blocks
- Handle thread pool reuse scenarios
- Use `afterExecute` hook for cleanup

## JVM (Java Virtual Machine)

### Memory Structure

**Components**
- Class Loader
- Runtime Data Area
- Execution Engine
- Native Interface

**Runtime Data Areas**

**Thread-Shared**
- **Method Area**: Class metadata, constants, static variables
- **Heap**: Object instances, string pool (JDK 7+)

**Thread-Private**
- **VM Stack**: Stack frames with local variables, operand stack
- **Native Method Stack**: For native method calls
- **Program Counter**: Current bytecode instruction pointer

**Key Changes**
- **JDK 8**: Metaspace replaced PermGen (uses native memory)
- **JDK 7**: String pool moved to heap for better GC efficiency

### Class Loading

**Process**
1. **Loading**: Read .class files, create Class objects
2. **Verification**: Validate bytecode integrity
3. **Preparation**: Allocate memory, set default values for static variables
4. **Resolution**: Convert symbolic references to direct references
5. **Initialization**: Execute static blocks and variable assignments

**Class Loaders**
- **Bootstrap**: JDK core classes
- **Extension/Platform**: JDK extensions
- **Application**: CLASSPATH classes

**Parent Delegation**
- Child loaders delegate to parent first
- Prevents duplicate loading
- Ensures class uniqueness and security

**Custom Class Loaders**
- Extend ClassLoader, override `findClass()`
- Tomcat breaks delegation for web app isolation

### Garbage Collection

**Memory Regions**
- **Young Generation**: Eden + Survivor (From/To)
- **Old Generation**: Long-lived objects
- **Metaspace**: Class metadata (JDK 8+)

**GC Root Objects**
- Local variables in method stacks
- Static variables in loaded classes
- JNI references in native stacks
- Active Java threads

**Reference Types**
- **Strong**: Never collected while referenced
- **Soft**: Collected before OOM
- **Weak**: Collected at next GC
- **Phantom**: For cleanup notification only

**Collection Algorithms**
- **Young Generation**: Copy algorithm (efficient, no fragmentation)
- **Old Generation**: Mark-Sweep or Mark-Compact

**Garbage Collectors**

**Serial/Parallel**
- Single/multi-threaded
- Suitable for small applications or batch processing

**CMS (Concurrent Mark Sweep)**
- Low-latency for old generation
- Concurrent collection with application threads

**G1 (Garbage First)**
- Region-based collection
- Predictable pause times
- Default in JDK 9+, suitable for large heaps (>8GB)

**ZGC/Shenandoah**
- Ultra-low latency (<10ms)
- Supports TB-scale heaps
- Ideal for cloud-native applications

**Tuning Parameters**
```bash
# Heap sizing
-Xms4g -Xmx4g                    # Initial = Max heap
-Xmn2g                           # Young generation size
-XX:SurvivorRatio=8              # Eden:Survivor ratio

# GC logging
-XX:+PrintGCDetails -Xloggc:gc.log

# Metaspace
-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=512M

# OOM debugging
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/dump.hprof
```

**OOM Troubleshooting**
1. Generate heap dumps on OOM
2. Analyze with MAT (Memory Analyzer Tool) or VisualVM
3. Common causes: memory leaks, oversized objects, insufficient heap space
4. Tune object promotion thresholds and GC parameters

## Performance Optimization Tips

1. **Collections**: Pre-size collections, use appropriate implementations
2. **Strings**: Use StringBuilder for concatenation, intern frequently used strings
3. **Thread Pools**: Tune core/max sizes based on workload characteristics
4. **GC**: Choose appropriate collector, monitor GC logs, tune heap ratios
5. **Memory**: Avoid memory leaks, use object pools for expensive objects