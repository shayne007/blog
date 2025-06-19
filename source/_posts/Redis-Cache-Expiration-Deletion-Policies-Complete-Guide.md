---
title: Redis Cache Expiration Deletion Policies - Complete Guide
date: 2025-06-19 15:25:00
tags: [redis]
categories: [redis]
---

## Overview of Cache Expiration Strategies

Redis implements multiple expiration deletion strategies to efficiently manage memory and ensure optimal performance. Understanding these mechanisms is crucial for building scalable, high-performance applications.

**Interview Insight**: *"How does Redis handle expired keys?"* - Redis uses a combination of lazy deletion and active deletion strategies. It doesn't immediately delete expired keys but employs intelligent algorithms to balance performance and memory usage.

## Core Expiration Deletion Policies

### Lazy Deletion (Passive Expiration)

Lazy deletion is the primary mechanism where expired keys are only removed when they are accessed.

**How it works**:
- When a client attempts to access a key, Redis checks if it has expired
- If expired, the key is immediately deleted and `NULL` is returned
- No background scanning or proactive deletion occurs

```python
# Example: Lazy deletion in action
import redis
import time

r = redis.Redis()

# Set a key with 2-second expiration
r.setex('temp_key', 2, 'temporary_value')

# Key exists initially
print(r.get('temp_key'))  # b'temporary_value'

# Wait for expiration
time.sleep(3)

# Key is deleted only when accessed (lazy deletion)
print(r.get('temp_key'))  # None
```

**Advantages**:
- Minimal CPU overhead
- No background processing required
- Perfect for frequently accessed keys

**Disadvantages**:
- Memory waste if expired keys are never accessed
- Unpredictable memory usage patterns

### Active Deletion (Proactive Scanning)

Redis periodically scans and removes expired keys to prevent memory bloat.

**Algorithm Details**:
1. Redis runs expiration cycles approximately 10 times per second
2. Each cycle samples 20 random keys from the expires dictionary
3. If more than 25% are expired, repeat the process
4. Maximum execution time per cycle is limited to prevent blocking

{% mermaid flowchart TD %}
    A[Start Expiration Cycle] --> B[Sample 20 Random Keys]
    B --> C{More than 25% expired?}
    C -->|Yes| D[Delete Expired Keys]
    D --> E{Time limit reached?}
    E -->|No| B
    E -->|Yes| F[End Cycle]
    C -->|No| F
    F --> G[Wait ~100ms]
    G --> A
{% endmermaid %}

**Configuration Parameters**:
```bash
# Redis configuration for active expiration
hz 10                    # Frequency of background tasks (10 Hz = 10 times/second)
active-expire-effort 1   # CPU effort for active expiration (1-10)
```

### Timer-Based Deletion

While Redis doesn't implement traditional timer-based deletion, you can simulate it using sorted sets:

```python
import redis
import time
import threading

class TimerCache:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.timer_key = "expiration_timer"
        
    def set_with_timer(self, key, value, ttl):
        """Set key-value with custom timer deletion"""
        expire_time = time.time() + ttl
        
        # Store the actual data
        self.redis_client.set(key, value)
        
        # Add to timer sorted set
        self.redis_client.zadd(self.timer_key, {key: expire_time})
        
    def cleanup_expired(self):
        """Background thread to clean expired keys"""
        current_time = time.time()
        expired_keys = self.redis_client.zrangebyscore(
            self.timer_key, 0, current_time
        )
        
        if expired_keys:
            # Remove expired keys
            for key in expired_keys:
                self.redis_client.delete(key.decode())
            
            # Remove from timer set
            self.redis_client.zremrangebyscore(self.timer_key, 0, current_time)

# Usage example
cache = TimerCache()
cache.set_with_timer('user:1', 'John Doe', 60)  # 60 seconds TTL
```

### Delay Queue Deletion

Implement a delay queue pattern for complex expiration scenarios:

```python
import redis
import json
import time
from datetime import datetime, timedelta

class DelayQueueExpiration:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.queue_key = "delay_expiration_queue"
        
    def schedule_deletion(self, key, delay_seconds):
        """Schedule key deletion after specified delay"""
        execution_time = time.time() + delay_seconds
        task = {
            'key': key,
            'scheduled_time': execution_time,
            'action': 'delete'
        }
        
        self.redis_client.zadd(
            self.queue_key, 
            {json.dumps(task): execution_time}
        )
    
    def process_delayed_deletions(self):
        """Process pending deletions"""
        current_time = time.time()
        
        # Get tasks ready for execution
        ready_tasks = self.redis_client.zrangebyscore(
            self.queue_key, 0, current_time, withscores=True
        )
        
        for task_json, score in ready_tasks:
            task = json.loads(task_json)
            
            # Execute deletion
            self.redis_client.delete(task['key'])
            
            # Remove from queue
            self.redis_client.zrem(self.queue_key, task_json)
            
            print(f"Deleted key: {task['key']} at {datetime.now()}")

# Usage
delay_queue = DelayQueueExpiration()
delay_queue.schedule_deletion('temp_data', 300)  # Delete after 5 minutes
```

**Interview Insight**: *"What's the difference between active and passive expiration?"* - Passive (lazy) expiration only occurs when keys are accessed, while active expiration proactively scans and removes expired keys in background cycles to prevent memory bloat.

## Redis Expiration Policies (Eviction Policies)

When Redis reaches memory limits, it employs eviction policies to free up space:

### Available Eviction Policies

```bash
# Configuration in redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

**Policy Types**:

1. **noeviction** (default)
   - No keys are evicted
   - Write operations return errors when memory limit reached
   - Use case: Critical data that cannot be lost

2. **allkeys-lru**
   - Removes least recently used keys from all keys
   - Use case: General caching scenarios

3. **allkeys-lfu**
   - Removes least frequently used keys
   - Use case: Applications with distinct access patterns

4. **volatile-lru**
   - Removes LRU keys only from keys with expiration set
   - Use case: Mixed persistent and temporary data

5. **volatile-lfu**
   - Removes LFU keys only from keys with expiration set

6. **allkeys-random**
   - Randomly removes keys
   - Use case: When access patterns are unpredictable

7. **volatile-random**
   - Randomly removes keys with expiration set

8. **volatile-ttl**
   - Removes keys with shortest TTL first
   - Use case: Time-sensitive data prioritization

### Policy Selection Guide

{% mermaid flowchart TD %}
    A[Memory Pressure] --> B{All data equally important?}
    B -->|Yes| C[allkeys-lru/lfu]
    B -->|No| D{Temporary vs Persistent data?}
    D -->|Mixed| E[volatile-lru/lfu]
    D -->|Time-sensitive| F[volatile-ttl]
    C --> G[High access pattern variance?]
    G -->|Yes| H[allkeys-lfu]
    G -->|No| I[allkeys-lru]
{% endmermaid %}

## Master-Slave Cluster Expiration Mechanisms

### Replication of Expiration

In Redis clusters, expiration handling follows specific patterns:

**Master-Slave Expiration Flow**:
1. Only masters perform active expiration
2. Masters send explicit `DEL` commands to slaves
3. Slaves don't independently expire keys (except for lazy deletion)

{% mermaid sequenceDiagram %}
    participant M as Master
    participant S1 as Slave 1
    participant S2 as Slave 2
    participant C as Client
    
    Note over M: Active expiration cycle
    M->>M: Check expired keys
    M->>S1: DEL expired_key
    M->>S2: DEL expired_key
    
    C->>S1: GET expired_key
    S1->>S1: Lazy expiration check
    S1->>C: NULL (key expired)
{% endmermaid %}

### Cluster Configuration for Expiration

```bash
# Master configuration
bind 0.0.0.0
port 6379
maxmemory 1gb
maxmemory-policy allkeys-lru
hz 10

# Slave configuration
bind 0.0.0.0
port 6380
slaveof 127.0.0.1 6379
slave-read-only yes
```

**Production Example - Redis Sentinel with Expiration**:

```python
import redis.sentinel

# Sentinel configuration for high availability
sentinels = [('localhost', 26379), ('localhost', 26380), ('localhost', 26381)]
sentinel = redis.sentinel.Sentinel(sentinels)

# Get master and slave connections
master = sentinel.master_for('mymaster', socket_timeout=0.1)
slave = sentinel.slave_for('mymaster', socket_timeout=0.1)

# Write to master with expiration
master.setex('session:user:1', 3600, 'session_data')

# Read from slave (expiration handled consistently)
session_data = slave.get('session:user:1')
```

**Interview Insight**: *"How does Redis handle expiration in a cluster?"* - In Redis clusters, only master nodes perform active expiration. When a master expires a key, it sends explicit DEL commands to all slaves to maintain consistency.

## Durability and Expired Keys

### RDB Persistence

Expired keys are handled during RDB operations:

```bash
# RDB configuration
save 900 1      # Save if at least 1 key changed in 900 seconds
save 300 10     # Save if at least 10 keys changed in 300 seconds
save 60 10000   # Save if at least 10000 keys changed in 60 seconds

# Expired keys are not saved to RDB files
rdbcompression yes
rdbchecksum yes
```

### AOF Persistence

AOF handles expiration through explicit commands:

```bash
# AOF configuration
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Expired keys generate explicit DEL commands in AOF
no-appendfsync-on-rewrite no
```

**Example AOF entries for expiration**:
```
*2
$6
SELECT
$1
0
*3
$3
SET
$8
temp_key
$5
value
*3
$6
EXPIRE
$8
temp_key
$2
60
*2
$3
DEL
$8
temp_key
```

## Optimization Strategies
### Memory-Efficient Configuration

```bash
# redis.conf optimizations
maxmemory 2gb
maxmemory-policy allkeys-lru

# Active deletion tuning
hz 10  # Background task frequency
active-expire-cycle-lookups-per-loop 20
active-expire-cycle-fast-duration 1000

# Memory sampling for LRU/LFU
maxmemory-samples 5
```

### Expiration Time Configuration Optimization

**Hierarchical TTL Strategy**:

```python
class TTLManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        
        # Define TTL hierarchy
        self.ttl_config = {
            'hot_data': 300,      # 5 minutes - frequently accessed
            'warm_data': 1800,    # 30 minutes - moderately accessed
            'cold_data': 3600,    # 1 hour - rarely accessed
            'session_data': 7200, # 2 hours - user sessions
            'cache_data': 86400   # 24 hours - general cache
        }
    
    def set_with_smart_ttl(self, key, value, data_type='cache_data'):
        """Set key with intelligent TTL based on data type"""
        ttl = self.ttl_config.get(data_type, 3600)
        
        # Add jitter to prevent thundering herd
        import random
        jitter = random.randint(-ttl//10, ttl//10)
        final_ttl = ttl + jitter
        
        return self.redis.setex(key, final_ttl, value)
    
    def adaptive_ttl(self, key, access_frequency):
        """Adjust TTL based on access patterns"""
        base_ttl = 3600  # 1 hour base
        
        if access_frequency > 100:  # Hot key
            return base_ttl // 4    # 15 minutes
        elif access_frequency > 10: # Warm key
            return base_ttl // 2    # 30 minutes
        else:                       # Cold key
            return base_ttl * 2     # 2 hours

# Usage example
ttl_manager = TTLManager(redis.Redis())
ttl_manager.set_with_smart_ttl('user:profile:123', user_data, 'hot_data')
```



## Production Use Cases

### High-Concurrent Idempotent Scenarios
In idempotent(/aɪˈdempətənt/) operations, cache expiration must prevent duplicate processing while maintaining consistency.

```python
import redis
import uuid
import time
import hashlib

class IdempotentCache:
    def __init__(self):
        self.redis = redis.Redis()
        self.default_ttl = 300  # 5 minutes
    
    def generate_idempotent_key(self, operation, params):
        """Generate unique key for operation"""
        # Create hash from operation and parameters
        content = f"{operation}:{str(sorted(params.items()))}"
        return f"idempotent:{hashlib.md5(content.encode()).hexdigest()}"
    
    def execute_idempotent(self, operation, params, executor_func):
        """Execute operation with idempotency guarantee"""
        idempotent_key = self.generate_idempotent_key(operation, params)
        
        # Check if operation already executed
        result = self.redis.get(idempotent_key)
        if result:
            return json.loads(result)
        
        # Use distributed lock to prevent concurrent execution
        lock_key = f"lock:{idempotent_key}"
        lock_acquired = self.redis.set(lock_key, "1", nx=True, ex=60)
        
        if not lock_acquired:
            # Wait and check again
            time.sleep(0.1)
            result = self.redis.get(idempotent_key)
            if result:
                return json.loads(result)
            raise Exception("Operation in progress")
        
        try:
            # Execute the actual operation
            result = executor_func(params)
            
            # Cache the result
            self.redis.setex(
                idempotent_key, 
                self.default_ttl, 
                json.dumps(result)
            )
            
            return result
        finally:
            # Release lock
            self.redis.delete(lock_key)

# Usage example
def process_payment(params):
    # Simulate payment processing
    return {"status": "success", "transaction_id": str(uuid.uuid4())}

idempotent_cache = IdempotentCache()
result = idempotent_cache.execute_idempotent(
    "payment", 
    {"amount": 100, "user_id": "123"}, 
    process_payment
)
```

### Hot Key Scenarios

**Problem**: Managing frequently accessed keys that can overwhelm Redis.

```python
import redis
import random
import threading
from collections import defaultdict

class HotKeyManager:
    def __init__(self):
        self.redis = redis.Redis()
        self.access_stats = defaultdict(int)
        self.hot_key_threshold = 1000  # Requests per minute
        
    def get_with_hot_key_protection(self, key):
        """Get value with hot key protection"""
        self.access_stats[key] += 1
        
        # Check if key is hot
        if self.access_stats[key] > self.hot_key_threshold:
            return self._handle_hot_key(key)
        
        return self.redis.get(key)
    
    def _handle_hot_key(self, hot_key):
        """Handle hot key with multiple strategies"""
        strategies = [
            self._local_cache_strategy,
            self._replica_strategy,
            self._fragmentation_strategy
        ]
        
        # Choose strategy based on key characteristics
        return random.choice(strategies)(hot_key)
    
    def _local_cache_strategy(self, key):
        """Use local cache for hot keys"""
        local_cache_key = f"local:{key}"
        
        # Check local cache first (simulate with Redis)
        local_value = self.redis.get(local_cache_key)
        if local_value:
            return local_value
        
        # Get from main cache and store locally
        value = self.redis.get(key)
        if value:
            # Short TTL for local cache
            self.redis.setex(local_cache_key, 60, value)
        
        return value
    
    def _replica_strategy(self, key):
        """Create multiple replicas of hot key"""
        replica_count = 5
        replica_key = f"{key}:replica:{random.randint(1, replica_count)}"
        
        # Try to get from replica
        value = self.redis.get(replica_key)
        if not value:
            # Get from master and update replica
            value = self.redis.get(key)
            if value:
                self.redis.setex(replica_key, 300, value)  # 5 min TTL
        
        return value
    
    def _fragmentation_strategy(self, key):
        """Fragment hot key into smaller pieces"""
        # For large objects, split into fragments
        fragments = []
        fragment_index = 0
        
        while True:
            fragment_key = f"{key}:frag:{fragment_index}"
            fragment = self.redis.get(fragment_key)
            
            if not fragment:
                break
                
            fragments.append(fragment)
            fragment_index += 1
        
        if fragments:
            return b''.join(fragments)
        
        return self.redis.get(key)

# Usage example
hot_key_manager = HotKeyManager()
value = hot_key_manager.get_with_hot_key_protection('popular_product:123')
```

### Pre-Loading and Predictive Caching

```python
class PredictiveCacheManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        
    def preload_related_data(self, primary_key, related_keys_func, short_ttl=300):
        """
        Pre-load related data with shorter TTL
        Useful for pagination, related products, etc.
        """
        # Get related keys that might be accessed soon
        related_keys = related_keys_func(primary_key)
        
        pipeline = self.redis.pipeline()
        for related_key in related_keys:
            # Check if already cached
            if not self.redis.exists(related_key):
                # Pre-load with shorter TTL
                related_data = self._fetch_data(related_key)
                pipeline.setex(related_key, short_ttl, related_data)
        
        pipeline.execute()
    
    def cache_with_prefetch(self, key, value, ttl=3600, prefetch_ratio=0.1):
        """
        Cache data and trigger prefetch when TTL is near expiration
        """
        self.redis.setex(key, ttl, value)
        
        # Set a prefetch trigger at 90% of TTL
        prefetch_ttl = int(ttl * prefetch_ratio)
        prefetch_key = f"prefetch:{key}"
        self.redis.setex(prefetch_key, ttl - prefetch_ttl, "trigger")
        
    def check_and_prefetch(self, key, refresh_func):
        """Check if prefetch is needed and refresh in background"""
        prefetch_key = f"prefetch:{key}"
        if not self.redis.exists(prefetch_key):
            # Prefetch trigger expired - refresh in background
            threading.Thread(
                target=self._background_refresh,
                args=(key, refresh_func)
            ).start()
    
    def _background_refresh(self, key, refresh_func):
        """Refresh data in background before expiration"""
        try:
            new_value = refresh_func()
            current_ttl = self.redis.ttl(key)
            if current_ttl > 0:
                # Extend current key TTL and set new value
                self.redis.setex(key, current_ttl + 3600, new_value)
        except Exception as e:
            # Log error but don't fail main request
            print(f"Background refresh failed for {key}: {e}")

# Example usage for e-commerce
def get_related_product_keys(product_id):
    """Return keys for related products, reviews, recommendations"""
    return [
        f"product:{product_id}:reviews",
        f"product:{product_id}:recommendations", 
        f"product:{product_id}:similar",
        f"category:{get_category(product_id)}:featured"
    ]

# Pre-load when user views a product
predictive_cache = PredictiveCacheManager(redis_client)
predictive_cache.preload_related_data(
    f"product:{product_id}",
    get_related_product_keys,
    short_ttl=600  # 10 minutes for related data
)
```

## Performance Monitoring and Metrics

### Expiration Monitoring

```python
import redis
import time
import json

class ExpirationMonitor:
    def __init__(self):
        self.redis = redis.Redis()
        
    def get_expiration_stats(self):
        """Get comprehensive expiration statistics"""
        info = self.redis.info()
        
        stats = {
            'expired_keys': info.get('expired_keys', 0),
            'evicted_keys': info.get('evicted_keys', 0),
            'keyspace_hits': info.get('keyspace_hits', 0),
            'keyspace_misses': info.get('keyspace_misses', 0),
            'used_memory': info.get('used_memory', 0),
            'maxmemory': info.get('maxmemory', 0),
            'memory_usage_percentage': 0
        }
        
        if stats['maxmemory'] > 0:
            stats['memory_usage_percentage'] = (
                stats['used_memory'] / stats['maxmemory'] * 100
            )
        
        # Calculate hit ratio
        total_requests = stats['keyspace_hits'] + stats['keyspace_misses']
        if total_requests > 0:
            stats['hit_ratio'] = stats['keyspace_hits'] / total_requests * 100
        else:
            stats['hit_ratio'] = 0
            
        return stats
    
    def analyze_key_expiration_patterns(self, pattern="*"):
        """Analyze expiration patterns for keys matching pattern"""
        keys = self.redis.keys(pattern)
        expiration_analysis = {
            'total_keys': len(keys),
            'keys_with_ttl': 0,
            'keys_without_ttl': 0,
            'avg_ttl': 0,
            'ttl_distribution': {}
        }
        
        ttl_values = []
        
        for key in keys:
            ttl = self.redis.ttl(key)
            
            if ttl == -1:  # No expiration set
                expiration_analysis['keys_without_ttl'] += 1
            elif ttl >= 0:  # Has expiration
                expiration_analysis['keys_with_ttl'] += 1
                ttl_values.append(ttl)
                
                # Categorize TTL
                if ttl < 300:  # < 5 minutes
                    category = 'short_term'
                elif ttl < 3600:  # < 1 hour
                    category = 'medium_term'
                else:  # >= 1 hour
                    category = 'long_term'
                
                expiration_analysis['ttl_distribution'][category] = \
                    expiration_analysis['ttl_distribution'].get(category, 0) + 1
        
        if ttl_values:
            expiration_analysis['avg_ttl'] = sum(ttl_values) / len(ttl_values)
        
        return expiration_analysis

# Usage
monitor = ExpirationMonitor()
stats = monitor.get_expiration_stats()
print(f"Hit ratio: {stats['hit_ratio']:.2f}%")
print(f"Memory usage: {stats['memory_usage_percentage']:.2f}%")
```
### Configuration Checklist

```bash
# Memory management
maxmemory 2gb
maxmemory-policy allkeys-lru

# Expiration tuning
hz 10
active-expire-effort 1

# Persistence (affects expiration)
save 900 1
appendonly yes
appendfsync everysec

# Monitoring
latency-monitor-threshold 100
```

## Interview Questions and Expert Answers

**Q: How does Redis handle expiration in a master-slave setup, and what happens during failover?**

A: In Redis replication, only the master performs expiration logic. When a key expires on the master (either through lazy or active expiration), the master sends an explicit `DEL` command to all slaves. Slaves never expire keys independently - they wait for the master's instruction.

During failover, the promoted slave becomes the new master and starts handling expiration. However, there might be temporary inconsistencies because:
1. The old master might have expired keys that weren't yet replicated
2. Clock differences can cause timing variations
3. Some keys might appear "unexpired" on the new master

Production applications should handle these edge cases by implementing fallback mechanisms and not relying solely on Redis for strict expiration timing.

**Q: What's the difference between eviction and expiration, and how do they interact?**

A: Expiration is time-based removal of keys that have reached their TTL, while eviction is memory-pressure-based removal when Redis reaches its memory limit.

They interact in several ways:
- Eviction policies like `volatile-lru` only consider keys with expiration set
- Active expiration reduces memory pressure, potentially avoiding eviction
- The `volatile-ttl` policy evicts keys with the shortest remaining TTL first
- Proper TTL configuration can reduce eviction frequency and improve cache performance

**Q: How would you optimize Redis expiration for a high-traffic e-commerce site?**

A: For high-traffic e-commerce, I'd implement a multi-tier expiration strategy:

1. **Product Catalog**: Long TTL (4-24 hours) with background refresh
2. **Inventory Counts**: Short TTL (1-5 minutes) with real-time updates
3. **User Sessions**: Medium TTL (30 minutes) with sliding expiration
4. **Shopping Carts**: Longer TTL (24-48 hours) with cleanup processes
5. **Search Results**: Staggered TTL (15-60 minutes) with jitter to prevent thundering herd

Key optimizations:
- Use `allkeys-lru` eviction for cache-heavy workloads
- Implement predictive pre-loading for related products
- Add jitter to TTL values to prevent simultaneous expiration
- Monitor hot keys and implement replication strategies
- Use pipeline operations for bulk TTL updates

The goal is balancing data freshness, memory usage, and system performance while handling traffic spikes gracefully.

## External References and Resources

- [Redis Expiration Documentation](https://redis.io/commands/expire/)
- [Redis Memory Optimization Guide](https://redis.io/topics/memory-optimization)
- [Redis Cluster Specification](https://redis.io/topics/cluster-spec)
- [Redis Persistence Demystified](https://redis.io/topics/persistence)
- [High Performance Redis Patterns](https://redis.io/topics/patterns)

## Key Takeaways

Redis expiration deletion policies are crucial for maintaining optimal performance and memory usage in production systems. The combination of lazy deletion, active expiration, and memory eviction policies provides flexible options for different use cases.

Success in production requires understanding the trade-offs between memory usage, CPU overhead, and data consistency, especially in distributed environments. Monitoring expiration efficiency and implementing appropriate TTL strategies based on access patterns is essential for maintaining high-performance Redis deployments.

The key is matching expiration strategies to your specific use case: use longer TTLs with background refresh for stable data, shorter TTLs for frequently changing data, and implement sophisticated hot key handling for high-traffic scenarios.
