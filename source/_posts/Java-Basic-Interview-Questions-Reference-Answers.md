---
title: Java Basic Interview Questions-Reference Answers
date: 2025-07-28 23:57:51
tags: [java]
categories: [java]
---

## Core Java Concepts

### What's the difference between `==` and `.equals()` in Java?

**Reference Answer:**
- `==` compares references (memory addresses) for objects and values for primitives
- `.equals()` compares the actual content/state of objects
- By default, `.equals()` uses `==` (reference comparison) unless overridden
- When overriding `.equals()`, you must also override `.hashCode()` to maintain the contract: if two objects are equal according to `.equals()`, they must have the same hash code
- String pool example: `"hello" == "hello"` is true due to string interning, but `new String("hello") == new String("hello")` is false

```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

s1 == s2;        // true (same reference in string pool)
s1 == s3;        // false (different references)
s1.equals(s3);   // true (same content)
```

### Explain the Java memory model and garbage collection.

**Reference Answer:**
**Memory Areas:**
- **Heap**: Object storage, divided into Young Generation (Eden, S0, S1) and Old Generation
- **Stack**: Method call frames, local variables, partial results
- **Method Area/Metaspace**: Class metadata, constant pool
- **PC Register**: Current executing instruction
- **Native Method Stack**: Native method calls

**Garbage Collection Process:**
1. Objects created in Eden space
2. When Eden fills, minor GC moves surviving objects to Survivor space
3. After several GC cycles, long-lived objects promoted to Old Generation
4. Major GC cleans Old Generation (more expensive)

**Common GC Algorithms:**
- **Serial GC**: Single-threaded, suitable for small applications
- **Parallel GC**: Multi-threaded, good for throughput
- **G1GC**: Low-latency, good for large heaps
- **ZGC/Shenandoah**: Ultra-low latency collectors

### What are the differences between abstract classes and interfaces?

**Reference Answer:**
| Aspect | Abstract Class | Interface |
|--------|---------------|-----------|
| Inheritance | Single inheritance | Multiple inheritance |
| Methods | Can have concrete methods | All methods abstract (before Java 8) |
| Variables | Can have instance variables | Only public static final variables |
| Constructor | Can have constructors | Cannot have constructors |
| Access Modifiers | Any access modifier | Public by default |

**Modern Java (8+) additions:**
- Interfaces can have default and static methods
- Private methods in interfaces (Java 9+)

**When to use:**
- **Abstract Class**: When you have common code to share and "is-a" relationship
- **Interface**: When you want to define a contract and "can-do" relationship

## Concurrency and Threading

### How does the `volatile` keyword work?

**Reference Answer:**
**Purpose:** Ensures visibility of variable changes across threads and prevents instruction reordering.

**Memory Effects:**
- Reads/writes to volatile variables are directly from/to main memory
- Creates a happens-before relationship
- Prevents compiler optimizations that cache variable values

**When to use:**
- Simple flags or state variables
- Single writer, multiple readers scenarios
- Not sufficient for compound operations (like increment)

```java
public class VolatileExample {
    private volatile boolean flag = false;
    
    // Thread 1
    public void setFlag() {
        flag = true; // Immediately visible to other threads
    }
    
    // Thread 2
    public void checkFlag() {
        while (!flag) {
            // Will see the change immediately
        }
    }
}
```

**Limitations:** Doesn't provide atomicity for compound operations. Use `AtomicBoolean`, `AtomicInteger`, etc., for atomic operations.

### Explain different ways to create threads and their trade-offs.

**Reference Answer:**

**1. Extending Thread class:**
```java
class MyThread extends Thread {
    public void run() { /* implementation */ }
}
new MyThread().start();
```
- **Pros**: Simple, direct control
- **Cons**: Single inheritance limitation, tight coupling

**2. Implementing Runnable:**
```java
class MyTask implements Runnable {
    public void run() { /* implementation */ }
}
new Thread(new MyTask()).start();
```
- **Pros**: Better design, can extend other classes
- **Cons**: Still creates OS threads

**3. ExecutorService:**
```java
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> { /* task */ });
```
- **Pros**: Thread pooling, resource management
- **Cons**: More complex, need proper shutdown

**4. CompletableFuture:**
```java
CompletableFuture.supplyAsync(() -> { /* computation */ })
    .thenApply(result -> { /* transform */ });
```
- **Pros**: Asynchronous composition, functional style
- **Cons**: Learning curve, can be overkill for simple tasks

**5. Virtual Threads (Java 19+):**
```java
Thread.startVirtualThread(() -> { /* task */ });
```
- **Pros**: Lightweight, millions of threads possible
- **Cons**: New feature, limited adoption

### What's the difference between `synchronized` and `ReentrantLock`?

**Reference Answer:**

| Feature | synchronized | ReentrantLock |
|---------|-------------|---------------|
| Type | Intrinsic/implicit lock | Explicit lock |
| Acquisition | Automatic | Manual (lock/unlock) |
| Fairness | No fairness guarantee | Optional fairness |
| Interruptibility | Not interruptible | Interruptible |
| Try Lock | Not available | Available |
| Condition Variables | wait/notify | Multiple Condition objects |
| Performance | JVM optimized | Slightly more overhead |

**ReentrantLock Example:**
```java
private final ReentrantLock lock = new ReentrantLock(true); // fair lock

public void performTask() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock(); // Must be in finally block
    }
}

public boolean tryPerformTask() {
    if (lock.tryLock()) {
        try {
            // critical section
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false;
}
```

## Collections and Data Structures

### How does HashMap work internally?

**Reference Answer:**
**Internal Structure:**
- Array of buckets (Node<K,V>[] table)
- Each bucket can contain a linked list or red-black tree
- Default initial capacity: 16, load factor: 0.75

**Hash Process:**
1. Calculate hash code of key using `hashCode()`
2. Apply hash function: `hash(key) = key.hashCode() ^ (key.hashCode() >>> 16)`
3. Find bucket: `index = hash & (capacity - 1)`

**Collision Resolution:**
- **Chaining**: Multiple entries in same bucket form linked list
- **Treeification** (Java 8+): When bucket size ≥ 8, convert to red-black tree
- **Untreeification**: When bucket size ≤ 6, convert back to linked list

**Resizing:**
- When size > capacity × load factor, capacity doubles
- All entries rehashed to new positions
- Expensive operation, can cause performance issues

**Poor hashCode() Impact:**
If `hashCode()` always returns same value, all entries go to one bucket, degrading performance to O(n) for operations.

```java
// Simplified internal structure
static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

### When would you use ConcurrentHashMap vs Collections.synchronizedMap()?

**Reference Answer:**

**Collections.synchronizedMap():**
- Wraps existing map with synchronized methods
- **Synchronization**: Entire map locked for each operation
- **Performance**: Poor in multi-threaded scenarios
- **Iteration**: Requires external synchronization
- **Fail-fast**: Iterators can throw ConcurrentModificationException

**ConcurrentHashMap:**
- **Synchronization**: Segment-based locking (Java 7) or CAS operations (Java 8+)
- **Performance**: Excellent concurrent read performance
- **Iteration**: Weakly consistent iterators, no external sync needed
- **Fail-safe**: Iterators reflect state at creation time
- **Atomic operations**: putIfAbsent(), replace(), computeIfAbsent()

```java
// ConcurrentHashMap example
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.putIfAbsent("key", 1);
map.computeIfPresent("key", (k, v) -> v + 1);

// Safe iteration without external synchronization
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    // No ConcurrentModificationException
}
```

**Use ConcurrentHashMap when:**
- High concurrent access
- More reads than writes
- Need atomic operations
- Want better performance

## Design Patterns and Architecture

### Implement the Singleton pattern and discuss its problems.

**Reference Answer:**

**1. Eager Initialization:**
```java
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    
    private EagerSingleton() {}
    
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```
- **Pros**: Thread-safe, simple
- **Cons**: Creates instance even if never used

**2. Lazy Initialization (Thread-unsafe):**
```java
public class LazySingleton {
    private static LazySingleton instance;
    
    private LazySingleton() {}
    
    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton(); // Race condition!
        }
        return instance;
    }
}
```

**3. Thread-safe Lazy (Double-checked locking):**
```java
public class ThreadSafeSingleton {
    private static volatile ThreadSafeSingleton instance;
    
    private ThreadSafeSingleton() {}
    
    public static ThreadSafeSingleton getInstance() {
        if (instance == null) {
            synchronized (ThreadSafeSingleton.class) {
                if (instance == null) {
                    instance = new ThreadSafeSingleton();
                }
            }
        }
        return instance;
    }
}
```

**4. Enum Singleton (Recommended):**
```java
public enum EnumSingleton {
    INSTANCE;
    
    public void doSomething() {
        // business logic
    }
}
```

**Problems with Singleton:**
- **Testing**: Difficult to mock, global state
- **Coupling**: Tight coupling throughout application
- **Scalability**: Global bottleneck
- **Serialization**: Need special handling
- **Reflection**: Can break private constructor
- **Classloader**: Multiple instances with different classloaders

### Explain dependency injection and inversion of control.

**Reference Answer:**

**Inversion of Control (IoC):**
Principle where control of object creation and lifecycle is transferred from the application code to an external framework.

**Dependency Injection (DI):**
A technique to implement IoC where dependencies are provided to an object rather than the object creating them.

**Types of DI:**

**1. Constructor Injection:**
```java
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**2. Setter Injection:**
```java
public class UserService {
    private UserRepository userRepository;
    
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**3. Field Injection:**
```java
public class UserService {
    @Inject
    private UserRepository userRepository;
}
```

**Benefits:**
- **Testability**: Easy to inject mock dependencies
- **Flexibility**: Change implementations without code changes
- **Decoupling**: Reduces tight coupling between classes
- **Configuration**: Centralized dependency configuration

**Without DI:**
```java
public class UserService {
    private UserRepository userRepository = new DatabaseUserRepository(); // Tight coupling
}
```

**With DI:**
```java
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) { // Loose coupling
        this.userRepository = userRepository;
    }
}
```

## Performance and Optimization

### How would you identify and resolve a memory leak in a Java application?

**Reference Answer:**

**Identification Tools:**
1. **JVisualVM**: Visual profiler, heap dumps
2. **JProfiler**: Commercial profiler
3. **Eclipse MAT**: Memory Analyzer Tool
4. **JConsole**: Built-in monitoring
5. **Application metrics**: OutOfMemoryError frequency

**Detection Signs:**
- Gradual memory increase over time
- OutOfMemoryError exceptions
- Increasing GC frequency/duration
- Application slowdown

**Analysis Process:**

**1. Heap Dump Analysis:**
```bash
jcmd <pid> GC.run_finalization
jcmd <pid> VM.gc
jmap -dump:format=b,file=heapdump.hprof <pid>
```

**2. Common Leak Scenarios:**

**Static Collections:**
```java
public class LeakyClass {
    private static List<Object> cache = new ArrayList<>(); // Never cleared
    
    public void addToCache(Object obj) {
        cache.add(obj); // Memory leak!
    }
}
```

**Listener Registration:**
```java
public class EventPublisher {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener); // If not removed, leak!
    }
    
    public void removeListener(EventListener listener) {
        listeners.remove(listener); // Often forgotten
    }
}
```

**ThreadLocal Variables:**
```java
public class ThreadLocalLeak {
    private static ThreadLocal<ExpensiveObject> threadLocal = new ThreadLocal<>();
    
    public void setThreadLocalValue() {
        threadLocal.set(new ExpensiveObject()); // Clear when done!
    }
    
    public void cleanup() {
        threadLocal.remove(); // Essential in long-lived threads
    }
}
```

**Resolution Strategies:**
- Use weak references where appropriate
- Implement proper cleanup in finally blocks
- Clear collections when no longer needed
- Remove listeners in lifecycle methods
- Use try-with-resources for automatic cleanup
- Monitor object creation patterns

### What are some JVM tuning parameters you've used?

**Reference Answer:**

**Heap Memory:**
```bash
-Xms2g          # Initial heap size
-Xmx8g          # Maximum heap size
-XX:NewRatio=3  # Old/Young generation ratio
-XX:MaxMetaspaceSize=256m  # Metaspace limit
```

**Garbage Collection:**
```bash
# G1GC (recommended for large heaps)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m

# Parallel GC (good throughput)
-XX:+UseParallelGC
-XX:ParallelGCThreads=8

# ZGC (ultra-low latency)
-XX:+UseZGC
-XX:+UnlockExperimentalVMOptions
```

**GC Logging:**
```bash
-Xlog:gc*:gc.log:time,tags
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=100M
```

**Performance Monitoring:**
```bash
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dumps/
```

**JIT Compilation:**
```bash
-XX:+TieredCompilation
-XX:CompileThreshold=10000
-XX:+PrintCompilation
```

**Common Tuning Scenarios:**
- **High throughput**: Parallel GC, larger heap
- **Low latency**: G1GC or ZGC, smaller pause times
- **Memory constrained**: Smaller heap, compressed OOPs
- **CPU intensive**: More GC threads, tiered compilation

## Modern Java Features

### Explain streams and when you'd use them vs traditional loops.

**Reference Answer:**

**Stream Characteristics:**
- **Functional**: Declarative programming style
- **Lazy**: Operations executed only when terminal operation called
- **Immutable**: Original collection unchanged
- **Chainable**: Fluent API for operation composition

**Stream Example:**
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

// Traditional loop
List<String> result = new ArrayList<>();
for (String name : names) {
    if (name.length() > 4) {
        result.add(name.toUpperCase());
    }
}

// Stream approach
List<String> result = names.stream()
    .filter(name -> name.length() > 4)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

**When to use Streams:**
- **Data transformation pipelines**
- **Complex filtering/mapping operations**
- **Parallel processing** (`.parallelStream()`)
- **Functional programming style preferred**
- **Readability** over performance for complex operations

**When to use Traditional Loops:**
- **Simple iterations** without transformations
- **Performance critical** tight loops
- **Early termination** needed
- **State modification** during iteration
- **Index-based** operations

**Performance Considerations:**
```java
// Stream overhead for simple operations
list.stream().forEach(System.out::println); // Slower
list.forEach(System.out::println);           // Faster

// Streams excel at complex operations
list.stream()
    .filter(complex_predicate)
    .map(expensive_transformation)
    .sorted()
    .limit(10)
    .collect(Collectors.toList()); // More readable than equivalent loop
```

### What are records in Java 14+ and when would you use them?

**Reference Answer:**

**Records Definition:**
Records are immutable data carriers that automatically generate boilerplate code.

**Basic Record:**
```java
public record Person(String name, int age, String email) {}

// Automatically generates:
// - Constructor: Person(String name, int age, String email)
// - Accessors: name(), age(), email()
// - equals(), hashCode(), toString()
// - All fields are private final
```

**Custom Methods:**
```java
public record Point(double x, double y) {
    // Custom constructor with validation
    public Point {
        if (x < 0 || y < 0) {
            throw new IllegalArgumentException("Coordinates must be positive");
        }
    }
    
    // Additional methods
    public double distanceFromOrigin() {
        return Math.sqrt(x * x + y * y);
    }
    
    // Static factory method
    public static Point origin() {
        return new Point(0, 0);
    }
}
```

**When to Use Records:**
- **Data Transfer Objects** (DTOs)
- **Configuration objects**
- **API response/request models**
- **Value objects** in domain modeling
- **Tuple-like** data structures
- **Database result mapping**

**Example Use Cases:**

**API Response:**
```java
public record UserResponse(Long id, String username, String email, LocalDateTime createdAt) {}

// Usage
return users.stream()
    .map(user -> new UserResponse(user.getId(), user.getUsername(), 
                                 user.getEmail(), user.getCreatedAt()))
    .collect(Collectors.toList());
```

**Configuration:**
```java
public record DatabaseConfig(String url, String username, String password, 
                           int maxConnections, Duration timeout) {}
```

**Limitations:**
- Cannot extend other classes (can implement interfaces)
- All fields are implicitly final
- Cannot declare instance fields beyond record components
- Less flexibility than regular classes

**Records vs Classes:**
- **Use Records**: Immutable data, minimal behavior
- **Use Classes**: Mutable state, complex behavior, inheritance needed

## System Design Integration

### How would you design a thread-safe cache with TTL (time-to-live)?

**Reference Answer:**

**Design Requirements:**
- Thread-safe concurrent access
- Automatic expiration based on TTL
- Efficient cleanup of expired entries
- Good performance for reads and writes

**Implementation:**

```java
public class TTLCache<K, V> {
    private static class CacheEntry<V> {
        final V value;
        final long expirationTime;
        
        CacheEntry(V value, long ttlMillis) {
            this.value = value;
            this.expirationTime = System.currentTimeMillis() + ttlMillis;
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() > expirationTime;
        }
    }
    
    private final ConcurrentHashMap<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    private final ScheduledExecutorService cleanupExecutor;
    private final long defaultTTL;
    
    public TTLCache(long defaultTTLMillis, long cleanupIntervalMillis) {
        this.defaultTTL = defaultTTLMillis;
        this.cleanupExecutor = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "TTLCache-Cleanup");
            t.setDaemon(true);
            return t;
        });
        
        // Schedule periodic cleanup
        cleanupExecutor.scheduleAtFixedRate(this::cleanup, 
            cleanupIntervalMillis, cleanupIntervalMillis, TimeUnit.MILLISECONDS);
    }
    
    public void put(K key, V value) {
        put(key, value, defaultTTL);
    }
    
    public void put(K key, V value, long ttlMillis) {
        cache.put(key, new CacheEntry<>(value, ttlMillis));
    }
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null || entry.isExpired()) {
            cache.remove(key); // Clean up expired entry
            return null;
        }
        return entry.value;
    }
    
    public boolean containsKey(K key) {
        return get(key) != null;
    }
    
    public void remove(K key) {
        cache.remove(key);
    }
    
    public void clear() {
        cache.clear();
    }
    
    public int size() {
        cleanup(); // Clean expired entries first
        return cache.size();
    }
    
    private void cleanup() {
        cache.entrySet().removeIf(entry -> entry.getValue().isExpired());
    }
    
    public void shutdown() {
        cleanupExecutor.shutdown();
        try {
            if (!cleanupExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                cleanupExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            cleanupExecutor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

**Usage Example:**
```java
// Create cache with 5-minute default TTL, cleanup every minute
TTLCache<String, UserData> userCache = new TTLCache<>(5 * 60 * 1000, 60 * 1000);

// Store with default TTL
userCache.put("user123", userData);

// Store with custom TTL (10 minutes)
userCache.put("session456", sessionData, 10 * 60 * 1000);

// Retrieve
UserData user = userCache.get("user123");
```

**Alternative Approaches:**
- **Caffeine Cache**: Production-ready with advanced features
- **Guava Cache**: Google's caching library
- **Redis**: External cache for distributed systems
- **Chronicle Map**: Off-heap storage for large datasets

### Explain how you'd handle database connections in a high-traffic application.

**Reference Answer:**

**Connection Pooling Strategy:**

**1. HikariCP Configuration (Recommended):**
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("user");
        config.setPassword("password");
        
        // Pool sizing
        config.setMaximumPoolSize(20);              // Max connections
        config.setMinimumIdle(5);                   // Min idle connections
        config.setConnectionTimeout(30000);         // 30 seconds timeout
        config.setIdleTimeout(600000);              // 10 minutes idle timeout
        config.setMaxLifetime(1800000);             // 30 minutes max lifetime
        
        // Performance tuning
        config.setLeakDetectionThreshold(60000);    // 1 minute leak detection
        config.setCachePrepStmts(true);
        config.setPrepStmtCacheSize(250);
        config.setPrepStmtCacheSqlLimit(2048);
        
        return new HikariDataSource(config);
    }
}
```

**2. Connection Pool Sizing:**
```
connections = ((core_count * 2) + effective_spindle_count)

For CPU-intensive: core_count * 2
For I/O-intensive: higher multiplier (3-4x)
Monitor and adjust based on actual usage
```

**3. Transaction Management:**
```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Transactional(readOnly = true)
    public User findById(Long id) {
        return userRepository.findById(id);
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void updateUserAsync(Long id, UserData data) {
        // Runs in separate transaction
        User user = userRepository.findById(id);
        user.update(data);
        userRepository.save(user);
    }
    
    @Transactional(timeout = 30) // 30 seconds timeout
    public void bulkOperation(List<User> users) {
        users.forEach(userRepository::save);
    }
}
```

**4. Read/Write Splitting:**
```java
@Configuration
public class DatabaseRoutingConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("write", writeDataSource());
        targetDataSources.put("read", readDataSource());
        
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(writeDataSource());
        
        return routingDataSource;
    }
    
    @Bean
    public DataSource writeDataSource() {
        // Master database configuration
        return createDataSource("jdbc:postgresql://master:5432/mydb");
    }
    
    @Bean
    public DataSource readDataSource() {
        // Replica database configuration
        return createDataSource("jdbc:postgresql://replica:5432/mydb");
    }
}

public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "read" : "write";
    }
}
```

**5. Monitoring and Health Checks:**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection connection = dataSource.getConnection()) {
            if (connection.isValid(2)) { // 2 second timeout
                return Health.up()
                    .withDetail("database", "Available")
                    .withDetail("active-connections", getActiveConnections())
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withDetail("database", "Unavailable")
                .withException(e)
                .build();
        }
        return Health.down().withDetail("database", "Connection invalid").build();
    }
    
    private int getActiveConnections() {
        if (dataSource instanceof HikariDataSource) {
            return ((HikariDataSource) dataSource).getHikariPoolMXBean().getActiveConnections();
        }
        return -1;
    }
}
```

**6. Best Practices for High Traffic:**

**Connection Management:**
- Always use connection pooling
- Set appropriate timeouts
- Monitor pool metrics
- Use read replicas for read-heavy workloads

**Query Optimization:**
- Use prepared statements
- Implement proper indexing
- Cache frequently accessed data
- Use batch operations for bulk updates

**Resilience Patterns:**
- Circuit breaker for database failures
- Retry logic with exponential backoff
- Graceful degradation when database unavailable
- Database failover strategies

**Performance Monitoring:**
```java
@EventListener
public void handleConnectionPoolMetrics(ConnectionPoolMetricsEvent event) {
    logger.info("Active connections: {}, Idle: {}, Waiting: {}", 
        event.getActive(), event.getIdle(), event.getWaiting());
    
    if (event.getActive() > event.getMaxPool() * 0.8) {
        alertingService.sendAlert("High database connection usage");
    }
}
```

This comprehensive approach ensures database connections are efficiently managed in high-traffic scenarios while maintaining performance and reliability.