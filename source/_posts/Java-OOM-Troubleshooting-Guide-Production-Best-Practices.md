---
title: Java OOM Troubleshooting Guide - Production Best Practices
date: 2025-06-14 17:50:06
tags: [java,oom]
categories: [java]
---

## Types of OutOfMemoryError in Production Environments

### Java Heap Space OOM

The most common OOM error occurs when the JVM cannot allocate objects in the heap due to insufficient memory.

```java
// Example code that can cause heap OOM
public class HeapOOMExample {
    private static List<byte[]> memoryEater = new ArrayList<>();
    
    public static void main(String[] args) {
        try {
            while (true) {
                // Continuously allocate 1MB arrays
                memoryEater.add(new byte[1024 * 1024]);
                System.out.println("Allocated arrays: " + memoryEater.size());
            }
        } catch (OutOfMemoryError e) {
            System.err.println("java.lang.OutOfMemoryError: Java heap space");
            throw e;
        }
    }
}
```

**Production Case Study**: An e-commerce application experienced heap OOM during Black Friday sales due to caching user sessions without proper expiration policies.

```java
// Problematic session cache implementation
public class SessionManager {
    private static final Map<String, UserSession> sessions = new ConcurrentHashMap<>();
    
    public void createSession(String sessionId, UserSession session) {
        // Problem: No expiration mechanism
        sessions.put(sessionId, session);
    }
    
    // Fixed version with expiration
    private static final Map<String, SessionWithTimestamp> sessionsFixed = new ConcurrentHashMap<>();
    
    public void createSessionFixed(String sessionId, UserSession session) {
        sessionsFixed.put(sessionId, new SessionWithTimestamp(session, System.currentTimeMillis()));
        cleanupExpiredSessions();
    }
}
```

### PermGen/Metaspace OOM

Occurs when the permanent generation (Java 7 and earlier) or Metaspace (Java 8+) runs out of memory.

```java
// Dynamic class generation causing Metaspace OOM
public class MetaspaceOOMExample {
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        
        for (int i = 0; i < 100000; i++) {
            CtClass cc = pool.makeClass("GeneratedClass" + i);
            cc.addMethod(CtNewMethod.make("public void test() {}", cc));
            
            // Loading classes without proper cleanup
            Class<?> clazz = cc.toClass();
            System.out.println("Created class: " + clazz.getName());
        }
    }
}
```

**Interview Insight**: *"How would you differentiate between heap OOM and Metaspace OOM in production logs?"*
- Heap OOM: `java.lang.OutOfMemoryError: Java heap space`
- Metaspace OOM: `java.lang.OutOfMemoryError: Metaspace`

### Direct Memory OOM

Occurs when off-heap memory allocated by NIO or unsafe operations exceeds limits.

```java
// Direct memory allocation example
public class DirectMemoryOOM {
    public static void main(String[] args) {
        List<ByteBuffer> buffers = new ArrayList<>();
        
        try {
            while (true) {
                // Allocate 1MB direct memory buffers
                ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024);
                buffers.add(buffer);
                System.out.println("Allocated direct buffers: " + buffers.size());
            }
        } catch (OutOfMemoryError e) {
            System.err.println("java.lang.OutOfMemoryError: Direct buffer memory");
        }
    }
}
```
**Production Case Study: Big Data Processing**
```
Scenario: Apache Kafka consumer processing large messages
Symptoms: Intermittent failures during peak message processing
Root Cause: NIO operations consuming direct memory without proper cleanup
Fix: Increased -XX:MaxDirectMemorySize and improved buffer management
```

## Generating Heap Dumps When OOM Occurs

### Automatic Heap Dump Generation

Configure JVM parameters to automatically generate heap dumps on OOM:

```bash
# JVM flags for automatic heap dump generation
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/opt/app/heapdumps/ \
     -XX:+PrintGCDetails \
     -XX:+PrintGCTimeStamps \
     -Xloggc:/opt/app/logs/gc.log \
     -jar your-application.jar
```

### Manual Heap Dump Generation

```bash
# Using jmap (requires application PID)
jmap -dump:format=b,file=heap-dump.hprof <PID>

# Using jcmd (Java 8+)
jcmd <PID> GC.run_finalization
jcmd <PID> VM.gc
jcmd <PID> GC.heap_dump /path/to/heap-dump.hprof

# Using kill signal (if configured)
kill -3 <PID>  # Generates thread dump, not heap dump
```

### Production-Ready Heap Dump Script

```bash
#!/bin/bash
# heap-dump-collector.sh

APP_PID=$(pgrep -f "your-application.jar")
DUMP_DIR="/opt/app/heapdumps"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DUMP_FILE="${DUMP_DIR}/heap-dump-${TIMESTAMP}.hprof"

if [ -z "$APP_PID" ]; then
    echo "Application not running"
    exit 1
fi

echo "Generating heap dump for PID: $APP_PID"
jmap -dump:format=b,file="$DUMP_FILE" "$APP_PID"

if [ $? -eq 0 ]; then
    echo "Heap dump generated: $DUMP_FILE"
    # Compress the dump file to save space
    gzip "$DUMP_FILE"
    echo "Heap dump compressed: ${DUMP_FILE}.gz"
else
    echo "Failed to generate heap dump"
    exit 1
fi
```

## Analyzing Problems with MAT and VisualVM

### Memory Analyzer Tool (MAT) Analysis

**Step-by-step MAT Analysis Process:**

1. **Load the heap dump** into MAT
2. **Automatic Leak Suspects Report** - MAT automatically identifies potential memory leaks
3. **Dominator Tree Analysis** - Shows objects and their retained heap sizes
4. **Histogram View** - Groups objects by class and shows instance counts

```java
// Example of analyzing a suspected memory leak
public class LeakyClass {
    private List<String> dataList = new ArrayList<>();
    private static List<LeakyClass> instances = new ArrayList<>();
    
    public LeakyClass() {
        // Problem: Adding to static list without removal
        instances.add(this);
    }
    
    public void addData(String data) {
        dataList.add(data);
    }
    
    // Missing cleanup method
    public void cleanup() {
        instances.remove(this);
        dataList.clear();
    }
}
```

**MAT Analysis Results Interpretation:**

```
Leak Suspects Report:
┌─────────────────────────────────────────────────────────────┐
│ Problem Suspect 1                                           │
│ 45,234 instances of "LeakyClass"                           │
│ Accumulated objects in cluster have total size 89.2 MB     │
│ Keywords: java.util.ArrayList                               │
└─────────────────────────────────────────────────────────────┘
```

### VisualVM Analysis Approach

**Real-time Monitoring Setup:**

```bash
# Enable JMX for VisualVM connection
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     -jar your-application.jar
```

**VisualVM Analysis Workflow:**

1. **Connect to running application**
2. **Monitor heap usage patterns**
3. **Perform heap dumps during high memory usage**
4. **Analyze object retention paths**

## Common Causes of OOM and Solutions

### Memory Leaks

**Listener/Observer Pattern Leaks:**

```java
// Problematic code
public class EventPublisher {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
        // Problem: No removal mechanism
    }
    
    // Fixed version
    public void removeListener(EventListener listener) {
        listeners.remove(listener);
    }
}

// Proper cleanup in listener implementations
public class MyEventListener implements EventListener {
    private EventPublisher publisher;
    
    public MyEventListener(EventPublisher publisher) {
        this.publisher = publisher;
        publisher.addListener(this);
    }
    
    public void cleanup() {
        publisher.removeListener(this);
    }
}
```

**Interview Question**: *"How would you identify and fix a listener leak in production?"*

**Answer approach**: Monitor heap dumps for increasing listener collections, implement weak references, or ensure proper cleanup in lifecycle methods.

### Connection Pool Leaks

```java
// Database connection leak example
public class DatabaseService {
    private DataSource dataSource;
    
    // Problematic method
    public List<User> getUsers() throws SQLException {
        Connection conn = dataSource.getConnection();
        PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users");
        ResultSet rs = stmt.executeQuery();
        
        List<User> users = new ArrayList<>();
        while (rs.next()) {
            users.add(new User(rs.getString("name"), rs.getString("email")));
        }
        
        // Problem: Resources not closed properly
        return users;
    }
    
    // Fixed version with try-with-resources
    public List<User> getUsersFixed() throws SQLException {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users");
             ResultSet rs = stmt.executeQuery()) {
            
            List<User> users = new ArrayList<>();
            while (rs.next()) {
                users.add(new User(rs.getString("name"), rs.getString("email")));
            }
            return users;
        }
    }
}
```

### Oversized Objects

**Large Collection Handling:**

```java
// Problematic approach for large datasets
public class DataProcessor {
    public void processLargeDataset() {
        List<String> allData = new ArrayList<>();
        
        // Problem: Loading entire dataset into memory
        try (BufferedReader reader = Files.newBufferedReader(Paths.get("large-file.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                allData.add(line);
            }
        }
        
        // Process all data at once
        processData(allData);
    }
    
    // Streaming approach to handle large datasets
    public void processLargeDatasetStreaming() throws IOException {
        try (Stream<String> lines = Files.lines(Paths.get("large-file.txt"))) {
            lines.parallel()
                 .filter(line -> !line.isEmpty())
                 .map(this::transformLine)
                 .forEach(this::processLine);
        }
    }
}
```
**Large Object Example:**
```java
// Memory-intensive operation
public class ReportGenerator {
    public List<CustomerReport> generateMonthlyReports() {
        List<CustomerReport> reports = new ArrayList<>();
        // Loading 100K+ customer records into memory
        List<Customer> allCustomers = customerRepository.findAll();
        
        for (Customer customer : allCustomers) {
            reports.add(generateReport(customer)); // Each report ~50KB
        }
        return reports; // Total: ~5GB in memory
    }
}
```

**Optimized Solution:**
```java
// Stream-based processing
public class OptimizedReportGenerator {
    @Autowired
    private CustomerRepository customerRepository;
    
    public void generateMonthlyReports(ReportWriter writer) {
        customerRepository.findAllByStream()
            .map(this::generateReport)
            .forEach(writer::writeReport);
        // Memory usage: ~50KB per iteration
    }
}
```
### Container Resource Constraints
```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          limits:
            memory: "2Gi"  # Container limit
          requests:
            memory: "1Gi"
        env:
        - name: JAVA_OPTS
          value: "-Xmx1536m"  # JVM heap should be ~75% of container limit
```

**Interview Insight:** *"How do you size JVM heap in containerized environments? What's the relationship between container memory limits and JVM heap size?"*

## Tuning Object Promotion and GC Parameters

### Understanding Object Lifecycle

{% mermaid graph TD %}
    A[Object Creation] --> B[Eden Space]
    B --> C{Minor GC}
    C -->|Survives| D[Survivor Space S0]
    D --> E{Minor GC}
    E -->|Survives| F[Survivor Space S1]
    F --> G{Age Threshold?}
    G -->|Yes| H[Old Generation]
    G -->|No| I[Back to Survivor]
    C -->|Dies| J[Garbage Collected]
    E -->|Dies| J
    H --> K{Major GC}
    K --> L[Garbage Collected or Retained]
{% endmermaid %}

### GC Tuning Parameters

**G1GC Configuration for Production:**

```bash
# G1GC tuning for 8GB heap
java -Xms8g -Xmx8g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=16m \
     -XX:G1NewSizePercent=20 \
     -XX:G1MaxNewSizePercent=30 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:G1MixedGCCountTarget=8 \
     -XX:G1MixedGCLiveThresholdPercent=85 \
     -jar your-application.jar
```

**Parallel GC Configuration:**

```bash
# ParallelGC tuning for high-throughput applications
java -Xms4g -Xmx4g \
     -XX:+UseParallelGC \
     -XX:ParallelGCThreads=8 \
     -XX:NewRatio=3 \
     -XX:SurvivorRatio=8 \
     -XX:MaxTenuringThreshold=15 \
     -XX:PretenureSizeThreshold=1048576 \
     -jar your-application.jar
```

### Object Promotion Threshold Tuning

```java
// Monitor object age distribution
public class ObjectAgeMonitoring {
    
    // JVM flag to print tenuring distribution
    // -XX:+PrintTenuringDistribution
    
    public static void demonstrateObjectPromotion() {
        List<byte[]> shortLived = new ArrayList<>();
        List<byte[]> longLived = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            // Short-lived objects (should stay in young generation)
            byte[] temp = new byte[1024];
            shortLived.add(temp);
            
            if (i % 10 == 0) {
                // Long-lived objects (will be promoted to old generation)
                longLived.add(new byte[1024]);
            }
            
            // Clear short-lived objects periodically
            if (i % 100 == 0) {
                shortLived.clear();
            }
        }
    }
}
```

### GC Performance Monitoring

```bash
# Comprehensive GC logging (Java 11+)
java -Xlog:gc*:gc.log:time,tags \
     -XX:+UnlockExperimentalVMOptions \
     -XX:+UseEpsilonGC \  # For testing only
     -jar your-application.jar

# GC analysis script
#!/bin/bash
# gc-analyzer.sh

GC_LOG_FILE="gc.log"

echo "=== GC Performance Analysis ==="
echo "Total GC events:"
grep -c "GC(" "$GC_LOG_FILE"

echo "Average pause time:"
grep "GC(" "$GC_LOG_FILE" | awk -F',' '{sum+=$2; count++} END {print sum/count "ms"}'

echo "Memory before/after GC:"
grep -E "->.*\(" "$GC_LOG_FILE" | tail -10
```

## Advanced OOM Prevention Strategies

### Memory-Efficient Data Structures

```java
// Using primitive collections to reduce memory overhead
public class MemoryEfficientStorage {
    
    // Instead of HashMap<Integer, Integer>
    private TIntIntHashMap primitiveMap = new TIntIntHashMap();
    
    // Instead of List<Integer>
    private TIntArrayList primitiveList = new TIntArrayList();
    
    // Object pooling for frequently created objects
    private final ObjectPool<StringBuilder> stringBuilderPool = 
        new GenericObjectPool<>(new StringBuilderFactory());
    
    public String processData(List<String> data) {
        StringBuilder sb = null;
        try {
            sb = stringBuilderPool.borrowObject();
            
            for (String item : data) {
                sb.append(item).append(",");
            }
            
            return sb.toString();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            if (sb != null) {
                stringBuilderPool.returnObject(sb);
            }
        }
    }
}
```

### Circuit Breaker Pattern for Memory Protection

```java
// Circuit breaker to prevent memory exhaustion
public class MemoryCircuitBreaker {
    private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
    private final double memoryThreshold = 0.85; // 85% of heap
    private volatile boolean circuitOpen = false;
    
    public boolean isMemoryAvailable() {
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        double usedPercentage = (double) heapUsage.getUsed() / heapUsage.getMax();
        
        if (usedPercentage > memoryThreshold) {
            circuitOpen = true;
            return false;
        }
        
        if (circuitOpen && usedPercentage < (memoryThreshold - 0.1)) {
            circuitOpen = false;
        }
        
        return !circuitOpen;
    }
    
    public <T> T executeWithMemoryCheck(Supplier<T> operation) {
        if (!isMemoryAvailable()) {
            throw new RuntimeException("Circuit breaker open: Memory usage too high");
        }
        
        return operation.get();
    }
}
```

## Production Monitoring and Alerting

### JVM Metrics Collection

```java
// Custom JVM metrics collector
@Component
public class JVMMetricsCollector {
    
    private final MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
    private final List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
    
    @Scheduled(fixedRate = 30000) // Every 30 seconds
    public void collectMetrics() {
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        // Log heap metrics
        double heapUsedPercent = (double) heapUsage.getUsed() / heapUsage.getMax() * 100;
        
        if (heapUsedPercent > 80) {
            log.warn("High heap usage: {}%", String.format("%.2f", heapUsedPercent));
        }
        
        // Log GC metrics
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            long collections = gcBean.getCollectionCount();
            long time = gcBean.getCollectionTime();
            
            log.info("GC [{}]: Collections={}, Time={}ms", 
                    gcBean.getName(), collections, time);
        }
    }
}
```

### Alerting Configuration

```yaml
# Prometheus alerting rules for OOM prevention
groups:
  - name: jvm-memory-alerts
    rules:
      - alert: HighHeapUsage
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High JVM heap usage detected"
          description: "Heap usage is above 85% for more than 2 minutes"
      
      - alert: HighGCTime
        expr: rate(jvm_gc_collection_seconds_sum[5m]) > 0.1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "High GC time detected"
          description: "Application spending more than 10% time in GC"
      
      - alert: FrequentGC
        expr: rate(jvm_gc_collection_seconds_count[5m]) > 2
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Frequent GC cycles"
          description: "More than 2 GC cycles per second"
```

## Interview Questions and Expert Insights

### Core Technical Questions

**Q: "Explain the difference between memory leak and memory pressure in Java applications."**

**Expert Answer**: Memory leak refers to objects that are no longer needed but still referenced, preventing garbage collection. Memory pressure occurs when application legitimately needs more memory than available. Leaks show constant growth in heap dumps, while pressure shows high but stable memory usage with frequent GC.

**Q: "How would you troubleshoot an application that has intermittent OOM errors?"**

**Systematic Approach**:
1. Enable heap dump generation on OOM
2. Monitor GC logs for patterns
3. Use application performance monitoring (APM) tools
4. Implement memory circuit breakers
5. Analyze heap dumps during both normal and high-load periods

**Q: "What's the impact of different GC algorithms on OOM behavior?"**

**Comparison Table**:

| GC Algorithm | OOM Behavior | Best Use Case |
|--------------|--------------|---------------|
| Serial GC | Quick OOM detection | Small applications |
| Parallel GC | High throughput before OOM | Batch processing |
| G1GC | Predictable pause times | Large heaps (>4GB) |
| ZGC | Ultra-low latency | Real-time applications |

### Advanced Troubleshooting Scenarios

**Scenario**: "Application runs fine for hours, then suddenly throws OOM. Heap dump shows high memory usage but no obvious leaks."

**Investigation Strategy**:

```java
// Implement memory pressure monitoring
public class MemoryPressureDetector {
    private final Queue<Long> memorySnapshots = new ConcurrentLinkedQueue<>();
    private final int maxSnapshots = 100;
    
    @Scheduled(fixedRate = 60000) // Every minute
    public void takeSnapshot() {
        long usedMemory = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        
        memorySnapshots.offer(usedMemory);
        if (memorySnapshots.size() > maxSnapshots) {
            memorySnapshots.poll();
        }
        
        // Detect memory pressure trends
        if (memorySnapshots.size() >= 10) {
            List<Long> recent = new ArrayList<>(memorySnapshots);
            Collections.reverse(recent);
            
            // Check if memory is consistently increasing
            boolean increasingTrend = true;
            for (int i = 1; i < Math.min(10, recent.size()); i++) {
                if (recent.get(i) <= recent.get(i-1)) {
                    increasingTrend = false;
                    break;
                }
            }
            
            if (increasingTrend) {
                log.warn("Detected increasing memory pressure trend");
                // Trigger proactive measures
                triggerGarbageCollection();
                generateHeapDump();
            }
        }
    }
    
    private void triggerGarbageCollection() {
        System.gc(); // Use with caution in production
    }
}
```

## Best Practices Summary

### Development Phase
- Implement proper resource management with try-with-resources
- Use weak references for caches and listeners
- Design with memory constraints in mind
- Implement object pooling for frequently created objects

### Testing Phase
- Load test with realistic data volumes
- Monitor memory usage patterns during extended runs
- Test GC behavior under various loads
- Validate heap dump analysis procedures

### Production Phase
- Enable automatic heap dump generation
- Implement comprehensive monitoring and alerting
- Have heap dump analysis procedures documented
- Maintain GC logs for performance analysis

### Emergency Response
- Automated heap dump collection on OOM
- Circuit breaker patterns to prevent cascading failures
- Memory pressure monitoring and proactive alerting
- Documented escalation procedures for memory issues

## External References and Tools

### Essential Tools
- **Eclipse MAT**: https://www.eclipse.org/mat/
- **VisualVM**: https://visualvm.github.io/
- **JProfiler**: https://www.ej-technologies.com/products/jprofiler/
- **Async Profiler**: https://github.com/jvm-profiling-tools/async-profiler

### Monitoring Solutions
- **Micrometer**: https://micrometer.io/
- **Prometheus JVM Metrics**: https://github.com/prometheus/client_java
- **APM Tools**: New Relic, AppDynamics, Datadog

### Documentation
- **Oracle JVM Tuning Guide**: https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/
- **G1GC Documentation**: https://docs.oracle.com/javase/8/docs/technotes/guides/vm/G1.html
- **JVM Flags Reference**: https://chriswhocodes.com/hotspot_options_jdk11.html

### Best Practices Resources
- **Java Performance Tuning**: "Java Performance: The Definitive Guide" by Scott Oaks
- **GC Tuning**: "Optimizing Java" by Benjamin J. Evans
- **Memory Management**: "Java Memory Management" by Kiran Kumar

Remember: OOM troubleshooting is both art and science. Combine systematic analysis with deep understanding of your application's memory patterns. Always test memory optimizations in staging environments before production deployment.