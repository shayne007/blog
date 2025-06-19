---
title: Redis Cache Eviction Policies - Complete Guide
date: 2025-06-18 20:51:32
tags: [redis]
categories: [redis]
---

## Overview of Redis Memory Management

Redis is an in-memory data structure store that requires careful memory management to maintain optimal performance. When Redis approaches its memory limit, it must decide which keys to remove to make space for new data. This process is called **memory eviction**.

{% mermaid flowchart TD %}
    A[Redis Instance] --> B{Memory Usage Check}
    B -->|Below maxmemory| C[Accept New Data]
    B -->|At maxmemory| D[Apply Eviction Policy]
    D --> E[Select Keys to Evict]
    E --> F[Remove Selected Keys]
    F --> G[Accept New Data]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#bbf,stroke:#333,stroke-width:2px
    style E fill:#fbb,stroke:#333,stroke-width:2px
{% endmermaid %}

**Interview Insight**: *Why is memory management crucial in Redis?*
- Redis stores all data in RAM for fast access
- Uncontrolled memory growth can lead to system crashes
- Proper eviction prevents OOM (Out of Memory) errors
- Maintains predictable performance characteristics

## Redis Memory Eviction Policies

Redis offers 8 different eviction policies, each serving different use cases:

### LRU-Based Policies

#### allkeys-lru
Evicts the least recently used keys across all keys in the database.

```bash
# Configuration
CONFIG SET maxmemory-policy allkeys-lru

# Example scenario
SET user:1001 "John Doe"     # Time: T1
GET user:1001               # Access at T2
SET user:1002 "Jane Smith"  # Time: T3
# If memory is full, user:1002 is more likely to be evicted
```

**Best Practice**: Use when you have a natural access pattern where some data is accessed more frequently than others.

#### volatile-lru
Evicts the least recently used keys only among keys with an expiration set.

```bash
# Setup
SET session:abc123 "user_data" EX 3600  # With expiration
SET config:theme "dark"                 # Without expiration

# Only session:abc123 is eligible for LRU eviction
```

**Use Case**: Session management where you want to preserve configuration data.

### LFU-Based Policies

#### allkeys-lfu
Evicts the least frequently used keys across all keys.

```bash
# Example: Access frequency tracking
SET product:1 "laptop"      # Accessed 100 times
SET product:2 "mouse"       # Accessed 5 times
SET product:3 "keyboard"    # Accessed 50 times

# product:2 (mouse) would be evicted first due to lowest frequency
```

#### volatile-lfu
Evicts the least frequently used keys only among keys with expiration.

**Interview Insight**: *When would you choose LFU over LRU?*
- LFU is better for data with consistent access patterns
- LRU is better for data with temporal locality
- LFU prevents cache pollution from occasional bulk operations

### Random Policies

#### allkeys-random
Randomly selects keys for eviction across all keys.

```python
# Simulation of random eviction
import random

keys = ["user:1", "user:2", "user:3", "config:db", "session:xyz"]
evict_key = random.choice(keys)
print(f"Evicting: {evict_key}")
```

#### volatile-random
Randomly selects keys for eviction only among keys with expiration.

**When to Use Random Policies**:
- When access patterns are completely unpredictable
- For testing and development environments
- When you need simple, fast eviction decisions

### TTL-Based Policy

#### volatile-ttl
Evicts keys with expiration, prioritizing those with shorter remaining TTL.

```bash
# Example scenario
SET cache:data1 "value1" EX 3600  # Expires in 1 hour
SET cache:data2 "value2" EX 1800  # Expires in 30 minutes
SET cache:data3 "value3" EX 7200  # Expires in 2 hours

# cache:data2 will be evicted first (shortest TTL)
```

### No Eviction Policy

#### noeviction
Returns errors when memory limit is reached instead of evicting keys.

```bash
CONFIG SET maxmemory-policy noeviction

# When memory is full:
SET new_key "value"
# Error: OOM command not allowed when used memory > 'maxmemory'
```

**Use Case**: Critical systems where data loss is unacceptable.

## Memory Limitation Strategies

### Why Limit Cache Memory?

{% mermaid flowchart LR %}
    A[Unlimited Memory] --> B[System Instability]
    A --> C[Unpredictable Performance]
    A --> D[Resource Contention]
    
    E[Limited Memory] --> F[Predictable Behavior]
    E --> G[System Stability]
    E --> H[Better Resource Planning]
    
    style A fill:#fbb,stroke:#333,stroke-width:2px
    style E fill:#bfb,stroke:#333,stroke-width:2px
{% endmermaid %}

**Production Reasons**:
- **System Stability**: Prevents Redis from consuming all available RAM
- **Performance Predictability**: Maintains consistent response times
- **Multi-tenancy**: Allows multiple services to coexist
- **Cost Control**: Manages infrastructure costs effectively

### Basic Memory Configuration

```bash
# Set maximum memory limit (512MB)
CONFIG SET maxmemory 536870912

# Set eviction policy
CONFIG SET maxmemory-policy allkeys-lru

# Check current memory usage
INFO memory
```

### Using Lua Scripts for Advanced Memory Control

#### Limiting Key-Value Pairs

```lua
-- limit_keys.lua: Limit total number of keys
local max_keys = tonumber(ARGV[1])
local current_keys = redis.call('DBSIZE')

if current_keys >= max_keys then
    -- Get random key and delete it
    local keys = redis.call('RANDOMKEY')
    if keys then
        redis.call('DEL', keys)
        return "Evicted key: " .. keys
    end
end

-- Add the new key
redis.call('SET', KEYS[1], ARGV[2])
return "Key added successfully"
```

```bash
# Usage
EVAL "$(cat limit_keys.lua)" 1 "new_key" 1000 "new_value"
```

#### Limiting Value Size

```lua
-- limit_value_size.lua: Reject large values
local max_size = tonumber(ARGV[2])
local value = ARGV[1]
local value_size = string.len(value)

if value_size > max_size then
    return redis.error_reply("Value size " .. value_size .. " exceeds limit " .. max_size)
end

redis.call('SET', KEYS[1], value)
return "OK"
```

```bash
# Usage: Limit values to 1KB
EVAL "$(cat limit_value_size.lua)" 1 "my_key" "my_value" 1024
```

#### Memory-Aware Key Management

```lua
-- memory_aware_set.lua: Check memory before setting
local key = KEYS[1]
local value = ARGV[1]
local memory_threshold = tonumber(ARGV[2])

-- Get current memory usage
local memory_info = redis.call('MEMORY', 'USAGE', 'SAMPLES', '0')
local used_memory = memory_info['used_memory']
local max_memory = memory_info['maxmemory']

if max_memory > 0 and used_memory > (max_memory * memory_threshold / 100) then
    -- Trigger manual cleanup
    local keys_to_check = redis.call('RANDOMKEY')
    if keys_to_check then
        local key_memory = redis.call('MEMORY', 'USAGE', keys_to_check)
        if key_memory > 1000 then  -- If key uses more than 1KB
            redis.call('DEL', keys_to_check)
        end
    end
end

redis.call('SET', key, value)
return "Key set with memory check"
```

## Practical Cache Eviction Solutions

### Big Object Evict First Strategy

This strategy prioritizes evicting large objects to free maximum memory quickly.

```python
# Python implementation for big object eviction
import redis
import json

class BigObjectEvictionRedis:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.size_threshold = 10240  # 10KB threshold
    
    def set_with_size_check(self, key, value):
        # Calculate value size
        value_size = len(str(value).encode('utf-8'))
        
        # Store size metadata
        self.redis.hset(f"{key}:meta", "size", value_size)
        self.redis.hset(f"{key}:meta", "created", int(time.time()))
        
        # Set the actual value
        self.redis.set(key, value)
        
        # Track large objects
        if value_size > self.size_threshold:
            self.redis.sadd("large_objects", key)
    
    def evict_large_objects(self, target_memory_mb):
        large_objects = self.redis.smembers("large_objects")
        freed_memory = 0
        target_bytes = target_memory_mb * 1024 * 1024
        
        # Sort by size (largest first)
        objects_with_size = []
        for obj in large_objects:
            size = self.redis.hget(f"{obj}:meta", "size")
            if size:
                objects_with_size.append((obj, int(size)))
        
        objects_with_size.sort(key=lambda x: x[1], reverse=True)
        
        for obj, size in objects_with_size:
            if freed_memory >= target_bytes:
                break
                
            self.redis.delete(obj)
            self.redis.delete(f"{obj}:meta")
            self.redis.srem("large_objects", obj)
            freed_memory += size
            
        return freed_memory

# Usage example
r = redis.Redis()
big_obj_redis = BigObjectEvictionRedis(r)

# Set some large objects
big_obj_redis.set_with_size_check("large_data:1", "x" * 50000)
big_obj_redis.set_with_size_check("large_data:2", "y" * 30000)

# Evict to free 100MB
freed = big_obj_redis.evict_large_objects(100)
print(f"Freed {freed} bytes")
```

### Small Object Evict First Strategy

Useful when you want to preserve large, expensive-to-recreate objects.

```lua
-- small_object_evict.lua
local function get_object_size(key)
    return redis.call('MEMORY', 'USAGE', key) or 0
end

local function evict_small_objects(count)
    local all_keys = redis.call('KEYS', '*')
    local small_keys = {}
    
    for i, key in ipairs(all_keys) do
        local size = get_object_size(key)
        if size < 1000 then  -- Less than 1KB
            table.insert(small_keys, {key, size})
        end
    end
    
    -- Sort by size (smallest first)
    table.sort(small_keys, function(a, b) return a[2] < b[2] end)
    
    local evicted = 0
    for i = 1, math.min(count, #small_keys) do
        redis.call('DEL', small_keys[i][1])
        evicted = evicted + 1
    end
    
    return evicted
end

return evict_small_objects(tonumber(ARGV[1]))
```

### Low-Cost Evict First Strategy

Evicts data that's cheap to regenerate or reload.

```python
class CostBasedEviction:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.cost_factors = {
            'cache:': 1,      # Low cost - can regenerate
            'session:': 5,    # Medium cost - user experience impact
            'computed:': 10,  # High cost - expensive computation
            'external:': 8    # High cost - external API call
        }
    
    def set_with_cost(self, key, value, custom_cost=None):
        # Determine cost based on key prefix
        cost = custom_cost or self._calculate_cost(key)
        
        # Store with cost metadata
        pipe = self.redis.pipeline()
        pipe.set(key, value)
        pipe.hset(f"{key}:meta", "cost", cost)
        pipe.hset(f"{key}:meta", "timestamp", int(time.time()))
        pipe.execute()
    
    def _calculate_cost(self, key):
        for prefix, cost in self.cost_factors.items():
            if key.startswith(prefix):
                return cost
        return 3  # Default medium cost
    
    def evict_low_cost_items(self, target_count):
        # Get all keys with metadata
        pattern = "*:meta"
        meta_keys = self.redis.keys(pattern)
        
        items_with_cost = []
        for meta_key in meta_keys:
            original_key = meta_key.replace(':meta', '')
            cost = self.redis.hget(meta_key, 'cost')
            if cost:
                items_with_cost.append((original_key, int(cost)))
        
        # Sort by cost (lowest first)
        items_with_cost.sort(key=lambda x: x[1])
        
        evicted = 0
        for key, cost in items_with_cost[:target_count]:
            self.redis.delete(key)
            self.redis.delete(f"{key}:meta")
            evicted += 1
        
        return evicted

# Usage
cost_eviction = CostBasedEviction(redis.Redis())
cost_eviction.set_with_cost("cache:user:1001", user_data)
cost_eviction.set_with_cost("computed:analytics:daily", expensive_computation)
cost_eviction.evict_low_cost_items(10)
```

### Cold Data Evict First Strategy

```python
import time
from datetime import datetime, timedelta

class ColdDataEviction:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.access_tracking_key = "access_log"
    
    def get_with_tracking(self, key):
        # Record access
        now = int(time.time())
        self.redis.zadd(self.access_tracking_key, {key: now})
        
        # Get value
        return self.redis.get(key)
    
    def set_with_tracking(self, key, value):
        now = int(time.time())
        
        # Set value and track access
        pipe = self.redis.pipeline()
        pipe.set(key, value)
        pipe.zadd(self.access_tracking_key, {key: now})
        pipe.execute()
    
    def evict_cold_data(self, days_threshold=7, max_evict=100):
        """Evict data not accessed within threshold days"""
        cutoff_time = int(time.time()) - (days_threshold * 24 * 3600)
        
        # Get cold keys (accessed before cutoff time)
        cold_keys = self.redis.zrangebyscore(
            self.access_tracking_key, 
            0, 
            cutoff_time,
            start=0,
            num=max_evict
        )
        
        evicted_count = 0
        if cold_keys:
            pipe = self.redis.pipeline()
            for key in cold_keys:
                pipe.delete(key)
                pipe.zrem(self.access_tracking_key, key)
                evicted_count += 1
                
            pipe.execute()
        
        return evicted_count
    
    def get_access_stats(self):
        """Get access statistics"""
        now = int(time.time())
        day_ago = now - 86400
        week_ago = now - (7 * 86400)
        
        recent_keys = self.redis.zrangebyscore(self.access_tracking_key, day_ago, now)
        weekly_keys = self.redis.zrangebyscore(self.access_tracking_key, week_ago, now)
        total_keys = self.redis.zcard(self.access_tracking_key)
        
        return {
            'total_tracked_keys': total_keys,
            'accessed_last_day': len(recent_keys),
            'accessed_last_week': len(weekly_keys),
            'cold_keys': total_keys - len(weekly_keys)
        }

# Usage example
cold_eviction = ColdDataEviction(redis.Redis())

# Use with tracking
cold_eviction.set_with_tracking("user:1001", "user_data")
value = cold_eviction.get_with_tracking("user:1001")

# Evict data not accessed in 7 days
evicted = cold_eviction.evict_cold_data(days_threshold=7)
print(f"Evicted {evicted} cold data items")

# Get statistics
stats = cold_eviction.get_access_stats()
print(f"Access stats: {stats}")
```

## Algorithm Deep Dive

### LRU Implementation Details

Redis uses an approximate LRU algorithm for efficiency:

{% mermaid flowchart TD %}
    A[Key Access] --> B[Update LRU Clock]
    B --> C{Memory Full?}
    C -->|No| D[Operation Complete]
    C -->|Yes| E[Sample Random Keys]
    E --> F[Calculate LRU Score]
    F --> G[Select Oldest Key]
    G --> H[Evict Key]
    H --> I[Operation Complete]
    
    style E fill:#bbf,stroke:#333,stroke-width:2px
    style F fill:#fbb,stroke:#333,stroke-width:2px
{% endmermaid %}

**Interview Question**: *Why doesn't Redis use true LRU?*
- True LRU requires maintaining a doubly-linked list of all keys
- This would consume significant memory overhead
- Approximate LRU samples random keys and picks the best candidate
- Provides good enough results with much better performance

### LFU Implementation Details

Redis LFU uses a probabilistic counter that decays over time:

```python
# Simplified LFU counter simulation
import time
import random

class LFUCounter:
    def __init__(self):
        self.counter = 0
        self.last_access = time.time()
    
    def increment(self):
        # Probabilistic increment based on current counter
        # Higher counters increment less frequently
        probability = 1.0 / (self.counter * 10 + 1)
        if random.random() < probability:
            self.counter += 1
        self.last_access = time.time()
    
    def decay(self, decay_time_minutes=1):
        # Decay counter over time
        now = time.time()
        minutes_passed = (now - self.last_access) / 60
        
        if minutes_passed > decay_time_minutes:
            decay_amount = int(minutes_passed / decay_time_minutes)
            self.counter = max(0, self.counter - decay_amount)
            self.last_access = now

# Example usage
counter = LFUCounter()
for _ in range(100):
    counter.increment()
print(f"Counter after 100 accesses: {counter.counter}")
```

## Choosing the Right Eviction Policy

### Decision Matrix

{% mermaid flowchart TD %}
    A[Choose Eviction Policy] --> B{Data has TTL?}
    B -->|Yes| C{Preserve non-expiring data?}
    B -->|No| D{Access pattern known?}
    
    C -->|Yes| E[volatile-lru/lfu/ttl]
    C -->|No| F[allkeys-lru/lfu]
    
    D -->|Temporal locality| G[allkeys-lru]
    D -->|Frequency based| H[allkeys-lfu]
    D -->|Unknown/Random| I[allkeys-random]
    
    J{Can tolerate data loss?} --> K[No eviction]
    J -->|Yes| L[Choose based on pattern]
    
    style E fill:#bfb,stroke:#333,stroke-width:2px
    style G fill:#bbf,stroke:#333,stroke-width:2px
    style H fill:#fbb,stroke:#333,stroke-width:2px
{% endmermaid %}

### Use Case Recommendations

| Use Case | Recommended Policy | Reason |
|----------|-------------------|--------|
| Web session store | `volatile-lru` | Sessions have TTL, preserve config data |
| Cache layer | `allkeys-lru` | Recent data more likely to be accessed |
| Analytics cache | `allkeys-lfu` | Popular queries accessed frequently |
| Rate limiting | `volatile-ttl` | Remove expired limits first |
| Database cache | `allkeys-lfu` | Hot data accessed repeatedly |

### Production Configuration Example

```bash
# redis.conf production settings
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# Monitor memory usage
redis-cli --latency-history -i 1
redis-cli INFO memory | grep used_memory_human
```

## Performance Monitoring and Tuning

### Key Metrics to Monitor

```python
# monitoring_script.py
import redis
import time

def monitor_eviction_performance(redis_client):
    info = redis_client.info('stats')
    memory_info = redis_client.info('memory')
    
    metrics = {
        'evicted_keys': info.get('evicted_keys', 0),
        'keyspace_hits': info.get('keyspace_hits', 0),
        'keyspace_misses': info.get('keyspace_misses', 0),
        'used_memory': memory_info.get('used_memory', 0),
        'used_memory_peak': memory_info.get('used_memory_peak', 0),
        'mem_fragmentation_ratio': memory_info.get('mem_fragmentation_ratio', 0)
    }
    
    # Calculate hit ratio
    total_requests = metrics['keyspace_hits'] + metrics['keyspace_misses']
    hit_ratio = metrics['keyspace_hits'] / total_requests if total_requests > 0 else 0
    
    metrics['hit_ratio'] = hit_ratio
    
    return metrics

# Usage
r = redis.Redis()
while True:
    stats = monitor_eviction_performance(r)
    print(f"Hit Ratio: {stats['hit_ratio']:.2%}, Evicted: {stats['evicted_keys']}")
    time.sleep(10)
```

### Alerting Thresholds

```yaml
# alerts.yml (Prometheus/Grafana style)
alerts:
  - name: redis_hit_ratio_low
    condition: redis_hit_ratio < 0.90
    severity: warning
    
  - name: redis_eviction_rate_high
    condition: rate(redis_evicted_keys[5m]) > 100
    severity: critical
    
  - name: redis_memory_usage_high
    condition: redis_used_memory / redis_maxmemory > 0.90
    severity: warning
```

## Interview Questions and Answers

### Advanced Interview Questions

**Q: How would you handle a scenario where your cache hit ratio drops significantly after implementing LRU eviction?**

**A**: This suggests the working set is larger than available memory. Solutions:
1. Increase memory allocation if possible
2. Switch to LFU if there's a frequency-based access pattern
3. Implement application-level partitioning
4. Use Redis Cluster for horizontal scaling
5. Optimize data structures (use hashes for small objects)

**Q: Explain the trade-offs between different sampling sizes in Redis LRU implementation.**

**A**: 
- Small samples (3-5): Fast eviction, less accurate LRU approximation
- Large samples (10+): Better LRU approximation, higher CPU overhead
- Default (5): Good balance for most use cases
- Monitor `evicted_keys` and `keyspace_misses` to tune

**Q: How would you implement a custom eviction policy for a specific business requirement?**

**A**: Use Lua scripts or application-level logic:
```lua
-- Custom: Evict based on business priority
local function business_priority_evict()
    local keys = redis.call('KEYS', '*')
    local priorities = {}
    
    for i, key in ipairs(keys) do
        local priority = redis.call('HGET', key .. ':meta', 'business_priority')
        if priority then
            table.insert(priorities, {key, tonumber(priority)})
        end
    end
    
    table.sort(priorities, function(a, b) return a[2] < b[2] end)
    
    if #priorities > 0 then
        redis.call('DEL', priorities[1][1])
        return priorities[1][1]
    end
    return nil
end
```

## Best Practices Summary

### Configuration Best Practices

1. **Set appropriate maxmemory**: 80% of available RAM for dedicated Redis instances
2. **Choose policy based on use case**: LRU for temporal, LFU for frequency patterns
3. **Monitor continuously**: Track hit ratios, eviction rates, and memory usage
4. **Test under load**: Verify eviction behavior matches expectations

### Application Integration Best Practices

1. **Graceful degradation**: Handle cache misses gracefully
2. **TTL strategy**: Set appropriate expiration times
3. **Key naming**: Use consistent patterns for better policy effectiveness
4. **Size awareness**: Monitor and limit large values

### Operational Best Practices

1. **Regular monitoring**: Set up alerts for key metrics
2. **Capacity planning**: Plan for growth and peak loads
3. **Testing**: Regularly test eviction scenarios
4. **Documentation**: Document policy choices and rationale

## External Resources

- [Redis Memory Optimization Guide](https://redis.io/docs/manual/eviction/)
- [Redis Configuration Best Practices](https://redis.io/docs/manual/config/)
- [Redis Monitoring Tools](https://redis.io/docs/manual/admin/)
- [Performance Tuning Guide](https://redis.io/docs/manual/optimization/)

This comprehensive guide provides the foundation for implementing effective memory eviction strategies in Redis production environments. The combination of theoretical understanding and practical implementation examples ensures robust cache management that scales with your application needs.