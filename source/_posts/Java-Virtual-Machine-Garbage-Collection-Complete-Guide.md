---
title: Java Virtual Machine Garbage Collection - Complete Guide
date: 2025-06-19 20:00:34
tags: [java,garbage collection]
categories: [java]
---

## Memory Management Fundamentals

Java's automatic memory management through garbage collection is one of its key features that differentiates it from languages like C and C++. The JVM automatically handles memory allocation and deallocation, freeing developers from manual memory management while preventing memory leaks and dangling pointer issues.

### Memory Layout Overview

The JVM heap is divided into several regions, each serving specific purposes in the garbage collection process:

{% mermaid flowchart TB %}
    subgraph "JVM Memory Structure"
        subgraph "Heap Memory"
            subgraph "Young Generation"
                Eden["Eden Space"]
                S0["Survivor 0"]
                S1["Survivor 1"]
            end
            
            subgraph "Old Generation"
                OldGen["Old Generation (Tenured)"]
            end
            
            MetaSpace["Metaspace (Java 8+)"]
        end
        
        subgraph "Non-Heap Memory"
            PC["Program Counter"]
            Stack["Java Stacks"]
            Native["Native Method Stacks"]
            Direct["Direct Memory"]
        end
    end
{% endmermaid %}

**Interview Insight**: *"Can you explain the difference between heap and non-heap memory in JVM?"*

**Answer**: Heap memory stores object instances and arrays, managed by GC. Non-heap includes method area (storing class metadata), program counter registers, and stack memory (storing method calls and local variables). Only heap memory is subject to garbage collection.

## GC Roots and Object Reachability

### Understanding GC Roots

GC Roots are the starting points for garbage collection algorithms to determine object reachability. An object is considered "reachable" if there's a path from any GC Root to that object.

**Primary GC Roots include:**

- **Local Variables**: Variables in currently executing methods
- **Static Variables**: Class-level static references
- **JNI References**: Objects referenced from native code
- **Monitor Objects**: Objects used for synchronization
- **Thread Objects**: Active thread instances
- **Class Objects**: Loaded class instances in Metaspace

{% mermaid flowchart TD %}
    subgraph "GC Roots"
        LV["Local Variables"]
        SV["Static Variables"]
        JNI["JNI References"]
        TO["Thread Objects"]
    end
    
    subgraph "Heap Objects"
        A["Object A"]
        B["Object B"]
        C["Object C"]
        D["Object D (Unreachable)"]
    end
    
    LV --> A
    SV --> B
    A --> C
    B --> C
    
    style D fill:#ff6b6b
    style A fill:#51cf66
    style B fill:#51cf66
    style C fill:#51cf66
{% endmermaid %}

### Object Reachability Algorithm

The reachability analysis works through a mark-and-sweep approach:

1. **Mark Phase**: Starting from GC Roots, mark all reachable objects
2. **Sweep Phase**: Reclaim memory of unmarked (unreachable) objects

```java
// Example: Object Reachability
public class ReachabilityExample {
    private static Object staticRef;  // GC Root
    
    public void demonstrateReachability() {
        Object localRef = new Object();     // GC Root (local variable)
        Object chainedObj = new Object();
        
        // Creating reference chain
        localRef.someField = chainedObj;    // chainedObj is reachable
        
        // Breaking reference chain
        localRef = null;                    // chainedObj becomes unreachable
    }
}
```

**Interview Insight**: *"How does JVM determine if an object is eligible for garbage collection?"*

**Answer**: JVM uses reachability analysis starting from GC Roots. If an object cannot be reached through any path from GC Roots, it becomes eligible for GC. This is more reliable than reference counting as it handles circular references correctly.

## Object Reference Types

Java provides different reference types that interact with garbage collection in distinct ways:

### Strong References
Default reference type that prevents garbage collection:

```java
Object obj = new Object();  // Strong reference
// obj will not be collected while this reference exists
```

### Weak References
Allow garbage collection even when references exist:

```java
import java.lang.ref.WeakReference;

WeakReference<Object> weakRef = new WeakReference<>(new Object());
Object obj = weakRef.get();  // May return null if collected

// Common use case: Cache implementation
public class WeakCache<K, V> {
    private Map<K, WeakReference<V>> cache = new HashMap<>();
    
    public V get(K key) {
        WeakReference<V> ref = cache.get(key);
        return (ref != null) ? ref.get() : null;
    }
}
```

### Soft References
More aggressive than weak references, collected only when memory is low:

```java
import java.lang.ref.SoftReference;

SoftReference<LargeObject> softRef = new SoftReference<>(new LargeObject());
// Collected only when JVM needs memory
```

### Phantom References
Used for cleanup operations, cannot retrieve the object:

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(obj, queue);
// Used for resource cleanup notification
```

**Interview Insight**: *"When would you use WeakReference vs SoftReference?"*

**Answer**: Use WeakReference for cache entries that can be recreated easily (like parsed data). Use SoftReference for memory-sensitive caches where you want to keep objects as long as possible but allow collection under memory pressure.

## Generational Garbage Collection

### The Generational Hypothesis

Most objects die young - this fundamental observation drives generational GC design:

{% mermaid flowchart LR %}
    subgraph "Object Lifecycle"
        A["Object Creation"] --> B["Short-lived Objects (90%+)"]
        A --> C["Long-lived Objects (<10%)"]
        B --> D["Die in Young Generation"]
        C --> E["Promoted to Old Generation"]
    end
{% endmermaid %}

### Young Generation Structure

**Eden Space**: Where new objects are allocated
**Survivor Spaces (S0, S1)**: Hold objects that survived at least one GC cycle

```java
// Example: Object allocation flow
public class AllocationExample {
    public void demonstrateAllocation() {
        // Objects allocated in Eden space
        for (int i = 0; i < 1000; i++) {
            Object obj = new Object();  // Allocated in Eden
            
            if (i % 100 == 0) {
                // Some objects may survive longer
                longLivedList.add(obj);  // May get promoted to Old Gen
            }
        }
    }
}
```

### Minor GC Process

1. **Allocation**: New objects go to Eden
2. **Eden Full**: Triggers Minor GC
3. **Survival**: Live objects move to Survivor space
4. **Age Increment**: Survivor objects get age incremented
5. **Promotion**: Old enough objects move to Old Generation

{% mermaid sequenceDiagram %}
    participant E as Eden Space
    participant S0 as Survivor 0
    participant S1 as Survivor 1
    participant O as Old Generation
    
    E->>S0: First GC: Move live objects
    Note over S0: Age = 1
    E->>S0: Second GC: New objects to S0
    S0->>S1: Move aged objects
    Note over S1: Age = 2
    S1->>O: Promotion (Age >= threshold)
{% endmermaid %}

### Major GC and Old Generation

Old Generation uses different algorithms optimized for long-lived objects:

- **Concurrent Collection**: Minimize application pause times
- **Compaction**: Reduce fragmentation
- **Different Triggers**: Based on Old Gen occupancy or allocation failure

**Interview Insight**: *"Why is Minor GC faster than Major GC?"*

**Answer**: Minor GC only processes Young Generation (smaller space, most objects are dead). Major GC processes entire heap or Old Generation (larger space, more live objects), often requiring more complex algorithms like concurrent marking or compaction.

## Garbage Collection Algorithms

### Mark and Sweep

The fundamental GC algorithm:

**Mark Phase**: Identify live objects starting from GC Roots
**Sweep Phase**: Reclaim memory from dead objects

{% mermaid flowchart TD %}
    subgraph "Mark Phase"
        A["Start from GC Roots"] --> B["Mark Reachable Objects"]
        B --> C["Traverse Reference Graph"]
    end
    
    subgraph "Sweep Phase"
        D["Scan Heap"] --> E["Identify Unmarked Objects"]
        E --> F["Reclaim Memory"]
    end
    
    C --> D
{% endmermaid %}

**Advantages**: Simple, handles circular references
**Disadvantages**: Stop-the-world pauses, fragmentation

### Copying Algorithm

Used primarily in Young Generation:

```java
// Conceptual representation
public class CopyingGC {
    private Space fromSpace;
    private Space toSpace;
    
    public void collect() {
        // Copy live objects from 'from' to 'to' space
        for (Object obj : fromSpace.getLiveObjects()) {
            toSpace.copy(obj);
            updateReferences(obj);
        }
        
        // Swap spaces
        Space temp = fromSpace;
        fromSpace = toSpace;
        toSpace = temp;
        
        // Clear old space
        temp.clear();
    }
}
```

**Advantages**: No fragmentation, fast allocation
**Disadvantages**: Requires double memory, inefficient for high survival rates

### Mark-Compact Algorithm

Combines marking with compaction:

1. **Mark**: Identify live objects
2. **Compact**: Move live objects to eliminate fragmentation

{% mermaid flowchart LR %}
    subgraph "Before Compaction"
        A["Live"] --> B["Dead"] --> C["Live"] --> D["Dead"] --> E["Live"]
    end
{% endmermaid %}
{% mermaid flowchart LR %}
    subgraph "After Compaction"
        F["Live"] --> G["Live"] --> H["Live"] --> I["Free Space"]
    end
{% endmermaid %}

**Interview Insight**: *"Why doesn't Young Generation use Mark-Compact algorithm?"*

**Answer**: Young Generation has high mortality rate (90%+ objects die), making copying algorithm more efficient. Mark-Compact is better for Old Generation where most objects survive and fragmentation is a concern.

### Incremental and Concurrent Algorithms

**Incremental GC**: Breaks GC work into small increments
**Concurrent GC**: Runs GC concurrently with application threads

```java
// Tri-color marking for concurrent GC
public enum ObjectColor {
    WHITE,  // Not visited
    GRAY,   // Visited but children not processed
    BLACK   // Visited and children processed
}

public class ConcurrentMarking {
    public void concurrentMark() {
        // Mark roots as gray
        for (Object root : gcRoots) {
            root.color = GRAY;
            grayQueue.add(root);
        }
        
        // Process gray objects concurrently
        while (!grayQueue.isEmpty() && !shouldYield()) {
            Object obj = grayQueue.poll();
            for (Object child : obj.getReferences()) {
                if (child.color == WHITE) {
                    child.color = GRAY;
                    grayQueue.add(child);
                }
            }
            obj.color = BLACK;
        }
    }
}
```

## Garbage Collectors Evolution

### Serial GC (-XX:+UseSerialGC)

**Characteristics**: Single-threaded, stop-the-world
**Best for**: Small applications, client-side applications
**JVM Versions**: All versions

```bash
# JVM flags for Serial GC
java -XX:+UseSerialGC -Xmx512m MyApplication
```

**Use Case Example**:
```java
// Small desktop application
public class CalculatorApp {
    public static void main(String[] args) {
        // Serial GC sufficient for small heap sizes
        SwingUtilities.invokeLater(() -> new Calculator().setVisible(true));
    }
}
```

### Parallel GC (-XX:+UseParallelGC)

**Characteristics**: Multi-threaded, throughput-focused
**Best for**: Batch processing, throughput-sensitive applications
**Default**: Java 8 (server-class machines)

```bash
# Parallel GC configuration
java -XX:+UseParallelGC -XX:ParallelGCThreads=4 -Xmx2g MyBatchJob
```

**Production Example**:
```java
// Data processing application
public class DataProcessor {
    public void processBatch(List<Record> records) {
        // High throughput processing
        records.parallelStream()
               .map(this::transform)
               .collect(Collectors.toList());
    }
}
```

### CMS GC (-XX:+UseConcMarkSweepGC) [Deprecated in Java 14]
**Phases**:
1. Initial Mark (STW)
2. Concurrent Mark
3. Concurrent Preclean
4. Remark (STW)
5. Concurrent Sweep

**Characteristics**: Concurrent, low-latency focused
**Best for**: Web applications requiring low pause times

```bash
# CMS configuration (legacy)
java -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode -Xmx4g WebApp
```

### G1 GC (-XX:+UseG1GC)

**Characteristics**: Low-latency, region-based, predictable pause times
**Best for**: Large heaps (>4GB), latency-sensitive applications
**Default**: Java 9+

```bash
# G1 GC tuning
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:G1HeapRegionSize=16m -Xmx8g
```

**Region-based Architecture**:
{% mermaid flowchart TB %}
    subgraph "G1 Heap Regions"
        subgraph "Young Regions"
            E1["Eden 1"]
            E2["Eden 2"]
            S1["Survivor 1"]
        end
        
        subgraph "Old Regions"
            O1["Old 1"]
            O2["Old 2"]
            O3["Old 3"]
        end
        
        subgraph "Special Regions"
            H["Humongous"]
            F["Free"]
        end
    end
{% endmermaid %}

**Interview Insight**: *"When would you choose G1 over Parallel GC?"*

**Answer**: Choose G1 for applications requiring predictable low pause times (<200ms) with large heaps (>4GB). Use Parallel GC for batch processing where throughput is more important than latency.

### ZGC (-XX:+UseZGC) [Java 11+]

**Characteristics**: Ultra-low latency (<10ms), colored pointers
**Best for**: Applications requiring consistent low latency

```bash
# ZGC configuration
java -XX:+UseZGC -XX:+UseTransparentHugePages -Xmx32g LatencyCriticalApp
```

### Shenandoah GC (-XX:+UseShenandoahGC) [Java 12+]

**Characteristics**: Low pause times, concurrent collection
**Best for**: Applications with large heaps requiring consistent performance
```bash
# Shenandoah configuration
-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=adaptive
```
### Collector Comparison
**Collector Comparison Table**:

| Collector | Java Version | Best Heap Size | Pause Time | Throughput | Use Case |
|---|---|---|---|---|---|
| Serial | All | < 100MB | High | Low | Single-core, client apps |
| Parallel | All (default 8) | < 8GB | Medium-High | High | Multi-core, batch processing |
| G1 | 7+ (default 9+) | > 4GB | Low-Medium | Medium-High | Server applications |
| ZGC | 11+ | > 8GB | Ultra-low | Medium | Latency-critical applications |
| Shenandoah | 12+ | > 8GB | Ultra-low | Medium | Real-time applications |

## GC Tuning Parameters and Best Practices

### Heap Sizing Parameters

```bash
# Basic heap configuration
-Xms2g          # Initial heap size
-Xmx8g          # Maximum heap size
-XX:NewRatio=3  # Old/Young generation ratio
-XX:SurvivorRatio=8  # Eden/Survivor ratio
```

### Young Generation Tuning

```bash
# Young generation specific tuning
-Xmn2g                    # Set young generation size
-XX:MaxTenuringThreshold=7 # Promotion threshold
-XX:TargetSurvivorRatio=90 # Survivor space target utilization
```

**Real-world Example**:
```java
// Web application tuning scenario
public class WebAppTuning {
    /*
     * Application characteristics:
     * - High request rate
     * - Short-lived request objects
     * - Some cached data
     * 
     * Tuning strategy:
     * - Larger young generation for short-lived objects
     * - G1GC for predictable pause times
     * - Monitoring allocation rate
     */
}

// JVM flags:
// -XX:+UseG1GC -Xmx4g -XX:MaxGCPauseMillis=100 
// -XX:G1HeapRegionSize=8m -XX:NewRatio=2
```

### Monitoring and Logging

```bash
# GC logging (Java 8)
-Xloggc:gc.log -XX:+PrintGCDetails -XX:+PrintGCTimeStamps

# GC logging (Java 9+)
-Xlog:gc*:gc.log:time,tags

# Additional monitoring
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintStringDeduplicationStatistics (G1)
```

### Production Tuning Checklist

**Memory Allocation**:
```java
// Monitor allocation patterns
public class AllocationMonitoring {
    public void trackAllocationRate() {
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        
        long beforeGC = memoryBean.getHeapMemoryUsage().getUsed();
        // ... application work
        long afterGC = memoryBean.getHeapMemoryUsage().getUsed();
        
        long allocatedBytes = calculateAllocationRate(beforeGC, afterGC);
    }
}
```

**GC Overhead Analysis**:
```java
// Acceptable GC overhead typically < 5%
public class GCOverheadCalculator {
    public double calculateGCOverhead(List<GCEvent> events, long totalTime) {
        long gcTime = events.stream()
                           .mapToLong(GCEvent::getDuration)
                           .sum();
        return (double) gcTime / totalTime * 100;
    }
}
```

## Advanced GC Concepts

### Escape Analysis and TLAB

**Thread Local Allocation Buffers (TLAB)** optimize object allocation:

```java
public class TLABExample {
    public void demonstrateTLAB() {
        // Objects allocated in thread-local buffer
        for (int i = 0; i < 1000; i++) {
            Object obj = new Object();  // Fast TLAB allocation
        }
    }
    
    // Escape analysis may eliminate allocation entirely
    public void noEscapeAllocation() {
        StringBuilder sb = new StringBuilder();  // May be stack-allocated
        sb.append("Hello");
        return sb.toString();  // Object doesn't escape method
    }
}
```

### String Deduplication (G1)

```bash
# Enable string deduplication
-XX:+UseG1GC -XX:+UseStringDeduplication
```

```java
// String deduplication example
public class StringDeduplication {
    public void demonstrateDeduplication() {
        List<String> strings = new ArrayList<>();
        
        // These strings have same content but different instances
        for (int i = 0; i < 1000; i++) {
            strings.add(new String("duplicate content"));  // Candidates for deduplication
        }
    }
}
```

### Compressed OOPs

```bash
# Enable compressed ordinary object pointers (default on 64-bit with heap < 32GB)
-XX:+UseCompressedOops
-XX:+UseCompressedClassPointers
```

## Interview Questions and Advanced Scenarios

### Scenario-Based Questions

**Question**: *"Your application experiences long GC pauses during peak traffic. How would you diagnose and fix this?"*

**Answer**:
1. **Analysis**: Enable GC logging, analyze pause times and frequency
2. **Identification**: Check if Major GC is causing long pauses
3. **Solutions**:
   - Switch to G1GC for predictable pause times
   - Increase heap size to reduce GC frequency
   - Tune young generation size
   - Consider object pooling for frequently allocated objects

```java
// Example diagnostic approach
public class GCDiagnostics {
    public void diagnoseGCIssues() {
        // Monitor GC metrics
        List<GarbageCollectorMXBean> gcBeans = 
            ManagementFactory.getGarbageCollectorMXBeans();
            
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            System.out.printf("GC Name: %s, Collections: %d, Time: %d ms%n",
                gcBean.getName(),
                gcBean.getCollectionCount(),
                gcBean.getCollectionTime());
        }
    }
}
```

**Question**: *"Explain the trade-offs between throughput and latency in GC selection."*

**Answer**:
- **Throughput-focused**: Parallel GC maximizes application processing time
- **Latency-focused**: G1/ZGC minimizes pause times but may reduce overall throughput
- **Choice depends on**: Application requirements, SLA constraints, heap size

### Memory Leak Detection

```java
// Common memory leak patterns
public class MemoryLeakExamples {
    private static Set<Object> cache = new HashSet<>();  // Static collection
    
    public void potentialLeak() {
        // Listeners not removed
        someComponent.addListener(event -> {});
        
        // ThreadLocal not cleaned
        ThreadLocal<ExpensiveObject> threadLocal = new ThreadLocal<>();
        threadLocal.set(new ExpensiveObject());
        // threadLocal.remove(); // Missing cleanup
    }
    
    // Proper cleanup
    public void properCleanup() {
        try {
            // Use try-with-resources
            try (AutoCloseable resource = createResource()) {
                // Work with resource
            }
        } catch (Exception e) {
            // Handle exception
        }
    }
}
```

## Production Best Practices

### Monitoring and Alerting

```java
// JMX-based GC monitoring
public class GCMonitor {
    private final List<GarbageCollectorMXBean> gcBeans;
    
    public GCMonitor() {
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
    }
    
    public void setupAlerts() {
        // Alert if GC overhead > 5%
        // Alert if pause times > SLA limits
        // Monitor allocation rate trends
    }
    
    public GCMetrics collectMetrics() {
        return new GCMetrics(
            getTotalGCTime(),
            getGCFrequency(),
            getLongestPause(),
            getAllocationRate()
        );
    }
}
```

### Capacity Planning

```java
// Capacity planning calculations
public class CapacityPlanning {
    public HeapSizeRecommendation calculateHeapSize(
            long allocationRate, 
            int targetGCFrequency,
            double survivorRatio) {
        
        // Rule of thumb: Heap size should accommodate 
        // allocation rate * GC interval * safety factor
        long recommendedHeap = allocationRate * targetGCFrequency * 3;
        
        return new HeapSizeRecommendation(
            recommendedHeap,
            calculateYoungGenSize(recommendedHeap, survivorRatio),
            calculateOldGenSize(recommendedHeap, survivorRatio)
        );
    }
}
```

### Performance Testing

```java
// GC performance testing framework
public class GCPerformanceTest {
    public void runGCStressTest() {
        // Measure allocation patterns
        AllocationProfiler profiler = new AllocationProfiler();
        
        // Simulate production load
        for (int iteration = 0; iteration < 1000; iteration++) {
            simulateWorkload();
            
            if (iteration % 100 == 0) {
                profiler.recordMetrics();
            }
        }
        
        // Analyze results
        profiler.generateReport();
    }
    
    private void simulateWorkload() {
        // Create realistic object allocation patterns
        List<Object> shortLived = createShortLivedObjects();
        Object longLived = createLongLivedObject();
        
        // Process data
        processData(shortLived, longLived);
    }
}
```

## Conclusion and Future Directions

Java's garbage collection continues to evolve with new collectors like ZGC and Shenandoah pushing the boundaries of low-latency collection. Understanding GC fundamentals, choosing appropriate collectors, and proper tuning remain critical for production Java applications.

**Key Takeaways**:
- Choose GC based on application requirements (throughput vs latency)
- Monitor and measure before optimizing
- Understand object lifecycle and allocation patterns
- Use appropriate reference types for memory-sensitive applications
- Regular capacity planning and performance testing

**Future Trends**:
- Ultra-low latency collectors (sub-millisecond pauses)
- Better integration with container environments
- Machine learning-assisted GC tuning
- Region-based collectors becoming mainstream

The evolution of GC technology continues to make Java more suitable for a wider range of applications, from high-frequency trading systems requiring microsecond latencies to large-scale data processing systems prioritizing throughput.

## External References

- [Oracle JVM Tuning Guide](https://docs.oracle.com/en/java/javase/17/gctuning/)
- [G1 Garbage Collector Documentation](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-g1-garbage-collector1.html)
- [ZGC Documentation](https://wiki.openjdk.org/display/zgc/Main)
- [Shenandoah GC](https://wiki.openjdk.org/display/shenandoah/Main)
- [GC Algorithms Implementations](https://github.com/openjdk/jdk)
- [Java Memory Model Specification](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4)