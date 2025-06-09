---
title: 'Redis Data Types and Data Structures: Complete Guide'
date: 2025-06-09 21:40:29
tags: [redis]
categories: [redis]
---

## Introduction

Redis (Remote Dictionary Server) is an in-memory data structure store that supports various data types. Understanding the underlying data structures is crucial for optimal performance and memory usage.

{% mermaid flowchart TD %}
    A[Redis Data Types] --> B[String]
    A --> C[Hash]
    A --> D[List]
    A --> E[Set]
    A --> F[Sorted Set]
    A --> G[Bitmap]
    A --> H[HyperLogLog]
    A --> I[Stream]
    A --> J[Geospatial]
    
    B --> B1[Simple Dynamic String - SDS]
    C --> C1[Hash Table / Ziplist]
    D --> D1[Ziplist / Quicklist]
    E --> E1[Hash Table / Intset]
    F --> F1[Skiplist + Hash Table / Ziplist]
{% endmermaid %}

**🎯 Interview Insight**: Always mention that Redis uses different underlying data structures based on the size and type of data to optimize memory and performance.

---

## String

### Underlying Data Structure: Simple Dynamic String (SDS)

Redis strings are built on Simple Dynamic Strings, not C strings. SDS provides several advantages:

- **Length caching**: O(1) length operation
- **Buffer overrun protection**: Prevents buffer overflow
- **Binary safe**: Can store any binary data
- **Space pre-allocation**: Reduces memory reallocations

### Structure Layout
```c
struct sdshdr {
    int len;        // String length
    int free;       // Available space
    char buf[];     // Character array
}
```

### Use Cases & Best Practices

#### 1. Caching
```bash
# Session storage
SET user:1001:session "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
EXPIRE user:1001:session 3600

# Cache with compression
SET article:123 "compressed_json_data" EX 1800
```

#### 2. Counters
```bash
# Page views
INCR page:home:views
INCRBY user:1001:score 50

# Rate limiting
SET rate_limit:user:1001 1 EX 60 NX
```

#### 3. Distributed Locks
```bash
# Acquire lock
SET lock:resource:123 "unique_token" EX 30 NX

# Release lock (Lua script)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### Memory Optimization Tips

```bash
# Use appropriate data types for numbers
SET counter 42           # Stored as string "42"
SET counter:int 42       # Better: use INCR for integers

# Compress large strings
SET large:data "gzip_compressed_data"
```

**🎯 Interview Insight**: Explain that Redis optimizes integer strings (like "123") by storing them as actual integers when possible, saving memory.

---

## Hash

### Underlying Data Structures

Redis hashes use two different encodings based on configuration:

1. **Ziplist** (for small hashes)
2. **Hash Table** (for larger hashes)

{% mermaid flowchart LR %}
    A[Hash] --> B{Size Check}
    B -->|Small| C[Ziplist Encoding]
    B -->|Large| D[Hash Table Encoding]
    
    C --> C1[Sequential Storage]
    C --> C2[Memory Efficient]
    
    D --> D1[O（1） Access]
    D --> D2[Hash Collision Handling]
{% endmermaid %}

### Configuration Thresholds
```bash
# redis.conf settings
hash-max-ziplist-entries 512    # Max fields in ziplist
hash-max-ziplist-value 64      # Max value size in ziplist
```

### Use Cases & Best Practices

#### 1. User Profiles
```bash
# User data storage
HSET user:1001 name "John Doe" email "john@example.com" age 30
HMGET user:1001 name email
HINCRBY user:1001 login_count 1
```

#### 2. Object Storage
```bash
# Product catalog
HSET product:123 name "Laptop" price 999.99 stock 50 category "electronics"
HGETALL product:123
```

#### 3. Configuration Storage
```bash
# Application settings
HSET app:config db_host "localhost" db_port 5432 cache_ttl 3600
HGET app:config db_host
```

### Performance Considerations

```bash
# Efficient batch operations
HMSET user:1001 field1 value1 field2 value2 field3 value3

# Avoid large hashes (>1000 fields)
# Better: Split into multiple hashes
HSET user:1001:profile name "John" email "john@example.com"
HSET user:1001:prefs theme "dark" lang "en"
```

**🎯 Interview Insight**: Mention that ziplist encoding provides significant memory savings (up to 10x) for small hashes, but switching to hash table occurs automatically when thresholds are exceeded.

---

## List

### Underlying Data Structures

Redis lists evolved through different implementations:

1. **Ziplist** (Redis < 3.2, for small lists)
2. **Linked List** (Redis < 3.2, for large lists)
3. **Quicklist** (Redis >= 3.2, hybrid approach)

{% mermaid flowchart TD %}
    A[List Evolution] --> B[Redis < 3.2]
    A --> C[Redis >= 3.2]
    
    B --> B1[Ziplist - Small Lists]
    B --> B2[Linked List - Large Lists]
    
    C --> C1[Quicklist]
    C1 --> C2[Doubly Linked List of Ziplists]
    C2 --> C3[Balanced Memory vs Performance]
{% endmermaid %}

### Quicklist Structure
```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;    // Total elements
    unsigned long len;      // Number of nodes
} quicklist;

typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;      // Ziplist
    unsigned int sz;        // Ziplist size
} quicklistNode;
```

### Use Cases & Best Practices

#### 1. Message Queues
```bash
# Producer
LPUSH queue:emails "email1@example.com"
LPUSH queue:emails "email2@example.com"

# Consumer
BRPOP queue:emails 0  # Blocking pop
```

#### 2. Activity Feeds
```bash
# Add new activity
LPUSH user:1001:feed "User liked post 123"
LTRIM user:1001:feed 0 99  # Keep latest 100 items

# Get recent activities
LRANGE user:1001:feed 0 9  # Get latest 10
```

#### 3. Undo/Redo Functionality
```bash
# Save state
LPUSH user:1001:undo_stack "state_data"

# Undo operation
LPOP user:1001:undo_stack
```

### Performance Optimization

```bash
# Efficient pagination
LRANGE articles:recent 0 19    # First page (0-19)
LRANGE articles:recent 20 39   # Second page (20-39)

# Avoid LINDEX on large lists (O(n) operation)
# Better: Use LRANGE for multiple elements
```

**🎯 Interview Insight**: Explain that LPUSH/LPOP and RPUSH/RPOP are O(1) operations, while LINDEX and LINSERT are O(n). This makes Redis lists perfect for stacks and queues but not for random access.

---

## Set

### Underlying Data Structures

Sets use two different encodings:

1. **Intset** (for sets containing only integers)
2. **Hash Table** (for other cases)

{% mermaid flowchart LR %}
    A[Set] --> B{All Integers?}
    B -->|Yes & Small| C[Intset Encoding]
    B -->|No or Large| D[Hash Table Encoding]
    
    C --> C1[Sorted Array]
    C --> C2[Binary Search]
    C --> C3[Memory Efficient]
    
    D --> D1[Hash Table]
    D --> D2[O（1） Operations]
    D --> D3[Any Data Type]
{% endmermaid %}

### Configuration
```bash
# redis.conf
set-max-intset-entries 512  # Max elements in intset
```

### Use Cases & Best Practices

#### 1. Unique Visitors Tracking
```bash
# Add visitors
SADD page:home:visitors user:1001 user:1002 user:1003

# Count unique visitors
SCARD page:home:visitors

# Check if user visited
SISMEMBER page:home:visitors user:1001
```

#### 2. Tag Systems
```bash
# Article tags
SADD article:123:tags "python" "redis" "database"
SADD article:456:tags "python" "flask" "web"

# Find articles with common tags
SINTER article:123:tags article:456:tags  # Returns: python
```

#### 3. Social Features
```bash
# Following/Followers
SADD user:1001:following user:1002 user:1003
SADD user:1002:followers user:1001

# Mutual friends
SINTER user:1001:following user:1002:following
```

### Set Operations Showcase

```bash
# Union - All unique elements
SADD set1 "a" "b" "c"
SADD set2 "c" "d" "e"
SUNION set1 set2  # Result: a, b, c, d, e

# Intersection - Common elements
SINTER set1 set2  # Result: c

# Difference - Elements in set1 but not in set2
SDIFF set1 set2   # Result: a, b
```

**🎯 Interview Insight**: Emphasize that intset encoding can save significant memory for integer sets, and Redis automatically chooses the optimal encoding based on data characteristics.

---

## Sorted Set (ZSet)

### Underlying Data Structures

Sorted Sets use a sophisticated dual data structure approach:

1. **Hash Table** - Maps members to scores (O(1) member lookup)
2. **Skip List** - Maintains sorted order (O(log n) range operations)

For small sorted sets, **Ziplist** encoding is used instead.

{% mermaid flowchart TD %}
    A[Sorted Set] --> B{Size Check}
    B -->|Small| C[Ziplist Encoding]
    B -->|Large| D[Skip List + Hash Table]
    
    D --> D1[Skip List]
    D --> D2[Hash Table]
    
    D1 --> D3[Sorted Range Queries]
    D1 --> D4[O（log n） Insert/Delete]
    
    D2 --> D5[O（1） Score Lookup]
    D2 --> D6[O（1） Member Check]
{% endmermaid %}

### Skip List Structure
```c
typedef struct zskiplistNode {
    sds ele;                    // Member
    double score;               // Score
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;     // Number of nodes to next
    } level[];
} zskiplistNode;
```

### Use Cases & Best Practices

#### 1. Leaderboards
```bash
# Gaming leaderboard
ZADD game:leaderboard 1500 "player1" 1200 "player2" 1800 "player3"

# Top 10 players
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# Player rank
ZREVRANK game:leaderboard "player1"

# Update score
ZINCRBY game:leaderboard 100 "player1"
```

#### 2. Time-based Data
```bash
# Recent articles (using timestamp as score)
ZADD articles:recent 1640995200 "article:123" 1640995300 "article:124"

# Get articles from last hour
ZRANGEBYSCORE articles:recent $(date -d "1 hour ago" +%s) +inf
```

#### 3. Priority Queues
```bash
# Task queue with priorities
ZADD task:queue 1 "low_priority_task" 5 "high_priority_task" 3 "medium_task"

# Process highest priority task
ZPOPMAX task:queue
```

### Advanced Operations

```bash
# Range by score
ZRANGEBYSCORE game:leaderboard 1000 2000 WITHSCORES

# Count elements in score range
ZCOUNT game:leaderboard 1000 2000

# Remove by rank
ZREMRANGEBYRANK game:leaderboard 0 -11  # Keep only top 10

# Lexicographical operations (when scores are equal)
ZRANGEBYLEX myset "[a" "[z"
```

**🎯 Interview Insight**: Explain why Redis uses both skip list and hash table - skip list for range operations and hash table for direct member access. This dual structure makes sorted sets extremely versatile.

---

## Bitmap

### Underlying Data Structure: String with Bit Operations

Bitmaps in Redis are actually strings that support bit-level operations. Each bit can represent a boolean state for a specific ID or position.

{% mermaid flowchart LR %}
    A[Bitmap] --> B[String Representation]
    B --> C[Bit Position 0]
    B --> D[Bit Position 1]
    B --> E[Bit Position 2]
    B --> F[... Bit Position N]
    
    C --> C1[User ID 1]
    D --> D1[User ID 2]
    E --> E1[User ID 3]
    F --> F1[User ID N+1]
{% endmermaid %}

### Use Cases & Best Practices

#### 1. User Activity Tracking
```bash
# Daily active users (bit position = user ID)
SETBIT daily_active:2024-01-15 1001 1  # User 1001 was active
SETBIT daily_active:2024-01-15 1002 1  # User 1002 was active

# Check if user was active
GETBIT daily_active:2024-01-15 1001

# Count active users
BITCOUNT daily_active:2024-01-15
```

#### 2. Feature Flags
```bash
# Feature availability (bit position = feature ID)
SETBIT user:1001:features 0 1  # Feature 0 enabled
SETBIT user:1001:features 2 1  # Feature 2 enabled

# Check feature access
GETBIT user:1001:features 0
```

#### 3. A/B Testing
```bash
# Test group assignment
SETBIT experiment:feature_x:group_a 1001 1
SETBIT experiment:feature_x:group_b 1002 1

# Users in both experiments
BITOP AND result experiment:feature_x:group_a experiment:other_experiment
```

### Bitmap Operations

```bash
# Bitwise operations
BITOP AND result key1 key2        # Intersection
BITOP OR result key1 key2         # Union
BITOP XOR result key1 key2        # Exclusive OR
BITOP NOT result key1             # Complement

# Find first bit
BITPOS daily_active:2024-01-15 1  # First active user
BITPOS daily_active:2024-01-15 0  # First inactive user
```

### Memory Efficiency

```bash
# Memory usage example
# Traditional set for 1 million users: ~32MB
# Bitmap for 1 million users: ~125KB (if sparse, much less)

# For user ID 1000000
SETBIT users:active 1000000 1  # Uses ~125KB total
```

**🎯 Interview Insight**: Bitmaps are extremely memory-efficient for representing large sparse boolean datasets. One million users can be represented in just 125KB instead of several megabytes with other data structures.

---

## HyperLogLog

### Underlying Data Structure: Probabilistic Counting

HyperLogLog uses probabilistic algorithms to estimate cardinality (unique count) with minimal memory usage.

{% mermaid flowchart TD %}
    A[HyperLogLog] --> B[Hash Function]
    B --> C[Leading Zeros Count]
    C --> D[Bucket Assignment]
    D --> E[Cardinality Estimation]
    
    E --> E1[Standard Error: 0.81%]
    E --> E2[Memory Usage: 12KB]
    E --> E3[Max Cardinality: 2^64]
{% endmermaid %}

### Algorithm Principle
```bash
# Simplified algorithm:
# 1. Hash each element
# 2. Count leading zeros in binary representation
# 3. Use bucket system for better accuracy
# 4. Apply harmonic mean for final estimation
```

### Use Cases & Best Practices

#### 1. Unique Visitors
```bash
# Add page visitors
PFADD page:home:unique_visitors user:1001 user:1002 user:1001

# Count unique visitors (approximate)
PFCOUNT page:home:unique_visitors

# Merge multiple HyperLogLogs
PFMERGE daily:unique_visitors page:home:unique_visitors page:about:unique_visitors
```

#### 2. Unique Event Counting
```bash
# Track unique events
PFADD events:login user:1001 user:1002 user:1003
PFADD events:purchase user:1001 user:1004

# Count unique users who performed any action
PFMERGE events:total events:login events:purchase
PFCOUNT events:total
```

#### 3. Real-time Analytics
```bash
# Hourly unique visitors
PFADD stats:$(date +%Y%m%d%H):unique visitor_id_1 visitor_id_2

# Daily aggregation
for hour in {00..23}; do
    PFMERGE stats:$(date +%Y%m%d):unique stats:$(date +%Y%m%d)${hour}:unique
done
```

### Accuracy vs. Memory Trade-off

```bash
# HyperLogLog: 12KB for any cardinality up to 2^64
# Set: 1GB+ for 10 million unique elements
# Error rate: 0.81% standard error

# Example comparison:
# Counting 10M unique users:
# - Set: ~320MB memory, 100% accuracy
# - HyperLogLog: 12KB memory, 99.19% accuracy
```

**🎯 Interview Insight**: HyperLogLog trades a small amount of accuracy (0.81% standard error) for tremendous memory savings. It's perfect for analytics where approximate counts are acceptable.

---

## Stream

### Underlying Data Structure: Radix Tree + Consumer Groups

Redis Streams use a radix tree (compressed trie) to store entries efficiently, with additional structures for consumer group management.

{% mermaid flowchart TD %}
    A[Stream] --> B[Radix Tree]
    A --> C[Consumer Groups]
    
    B --> B1[Stream Entries]
    B --> B2[Time-ordered IDs]
    B --> B3[Field-Value Pairs]
    
    C --> C1[Consumer Group State]
    C --> C2[Pending Entries List - PEL]
    C --> C3[Consumer Last Delivered ID]
{% endmermaid %}

### Stream Entry Structure
```bash
# Stream ID format: timestamp-sequence
# Example: 1640995200000-0
#          |-------------|--- |
#          timestamp(ms)     sequence
```

### Use Cases & Best Practices

#### 1. Event Sourcing
```bash
# Add events to stream
XADD user:1001:events * action "login" timestamp 1640995200 ip "192.168.1.1"
XADD user:1001:events * action "purchase" item "laptop" amount 999.99

# Read events
XRANGE user:1001:events - + COUNT 10
```

#### 2. Message Queues with Consumer Groups
```bash
# Create consumer group
XGROUP CREATE mystream mygroup $ MKSTREAM

# Add messages
XADD mystream * task "process_order" order_id 12345

# Consume messages
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >

# Acknowledge processing
XACK mystream mygroup 1640995200000-0
```

#### 3. Real-time Data Processing
```bash
# IoT sensor data
XADD sensors:temperature * sensor_id "temp001" value 23.5 location "room1"
XADD sensors:humidity * sensor_id "hum001" value 45.2 location "room1"

# Read latest data
XREAD COUNT 10 STREAMS sensors:temperature sensors:humidity $ $
```

### Stream Operations

```bash
# Range queries
XRANGE mystream 1640995200000 1640998800000  # Time range
XREVRANGE mystream + - COUNT 10               # Latest 10 entries

# Consumer group management
XGROUP CREATE mystream group1 0              # Create group
XINFO GROUPS mystream                        # Group info
XPENDING mystream group1                     # Pending messages

# Stream maintenance
XTRIM mystream MAXLEN ~ 1000                 # Keep ~1000 entries
XDEL mystream 1640995200000-0                # Delete specific entry
```

**🎯 Interview Insight**: Streams provide at-least-once delivery guarantees through the Pending Entries List (PEL), making them suitable for reliable message processing unlike simple pub/sub.

---

## Geospatial

### Underlying Data Structure: Sorted Set with Geohash

Redis geospatial features are built on top of sorted sets, using geohash as scores to enable spatial queries.

{% mermaid flowchart LR %}
    A[Geospatial] --> B[Sorted Set Backend]
    B --> C[Geohash as Score]
    C --> D[Spatial Queries]
    
    D --> D1[GEORADIUS]
    D --> D2[GEODIST]
    D --> D3[GEOPOS]
    D --> D4[GEOHASH]
{% endmermaid %}

### Geohash Encoding
```bash
# Latitude/Longitude -> Geohash -> 52-bit integer
# Example: (37.7749, -122.4194) -> 9q8yy -> score for sorted set
```

### Use Cases & Best Practices

#### 1. Location Services
```bash
# Add locations
GEOADD locations -122.4194 37.7749 "San Francisco"
GEOADD locations -74.0060 40.7128 "New York"
GEOADD locations -87.6298 41.8781 "Chicago"

# Find nearby locations
GEORADIUS locations -122.4194 37.7749 100 km WITHDIST WITHCOORD

# Distance between locations
GEODIST locations "San Francisco" "New York" km
```

#### 2. Delivery Services
```bash
# Add delivery drivers
GEOADD drivers -122.4094 37.7849 "driver:1001"
GEOADD drivers -122.4294 37.7649 "driver:1002"

# Find nearby drivers
GEORADIUS drivers -122.4194 37.7749 5 km WITHCOORD ASC

# Update driver location
GEOADD drivers -122.4150 37.7800 "driver:1001"
```

#### 3. Store Locator
```bash
# Add store locations
GEOADD stores -122.4194 37.7749 "store:sf_downtown"
GEOADD stores -122.4094 37.7849 "store:sf_mission"

# Find stores by area
GEORADIUSBYMEMBER stores "store:sf_downtown" 10 km WITHCOORD
```

### Advanced Geospatial Operations

```bash
# Get coordinates
GEOPOS locations "San Francisco"

# Get geohash
GEOHASH locations "San Francisco"

# Since it's built on sorted sets, you can use:
ZRANGE locations 0 -1              # All locations
ZREM locations "San Francisco"     # Remove location
ZCARD locations                    # Count locations
```

**🎯 Interview Insight**: Redis geospatial commands are syntactic sugar over sorted set operations. Understanding this helps explain why you can mix geospatial and sorted set commands on the same key.

---

## Memory Optimization

### Encoding Optimizations

Redis automatically chooses optimal encodings based on data characteristics:

{% mermaid flowchart TD %}
    A[Memory Optimization] --> B[Automatic Encoding Selection]
    A --> C[Configuration Tuning]
    A --> D[Data Structure Design]
    
    B --> B1[ziplist for small collections]
    B --> B2[intset for integer sets]
    B --> B3[embstr for small strings]
    
    C --> C1[hash-max-ziplist-entries]
    C --> C2[set-max-intset-entries]
    C --> C3[zset-max-ziplist-entries]
    
    D --> D1[Key naming patterns]
    D --> D2[Appropriate data types]
    D --> D3[Expiration policies]
{% endmermaid %}

### Configuration Optimization

```bash
# redis.conf optimizations
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Memory usage analysis
MEMORY USAGE mykey
MEMORY DOCTOR
INFO memory
```

### Best Practices

#### 1. Key Design
```bash
# Bad: Long descriptive keys
SET user:profile:information:personal:name:first "John"

# Good: Short, structured keys
SET u:1001:fname "John"

# Use hash for related data
HSET u:1001 fname "John" lname "Doe" email "john@example.com"
```

#### 2. Data Type Selection
```bash
# For counters
INCR counter           # Better than SET counter "1"

# For boolean flags
SETBIT flags 1001 1    # Better than SET flag:1001 "true"

# For unique counting
PFADD unique_visitors user:1001  # Better than SADD for large sets
```

#### 3. Memory Monitoring
```bash
# Check memory usage
MEMORY STATS
MEMORY USAGE mykey SAMPLES 5

# Identify memory hogs
redis-cli --bigkeys
redis-cli --memkeys
```

**🎯 Interview Insight**: Explain that Redis encoding transitions are transparent but can cause performance spikes during conversion. Understanding thresholds helps design applications that avoid frequent transitions.

---

## Performance Considerations

### Time Complexity by Operation

| Data Type | Operation | Time Complexity | Notes |
|-----------|-----------|----------------|-------|
| String | GET/SET | O(1) | |
| Hash | HGET/HSET | O(1) | O(n) for HGETALL |
| List | LPUSH/RPUSH | O(1) | |
| List | LINDEX | O(n) | Avoid on large lists |
| Set | SADD/SREM | O(1) | |
| Set | SINTER | O(n*m) | n = smallest set size |
| ZSet | ZADD/ZREM | O(log n) | |
| ZSet | ZRANGE | O(log n + m) | m = result size |

### Pipelining and Batching

```bash
# Without pipelining (multiple round trips)
SET key1 value1
SET key2 value2
SET key3 value3

# With pipelining (single round trip)
redis-cli --pipe <<EOF
SET key1 value1
SET key2 value2
SET key3 value3
EOF

# Lua scripting for atomic operations
EVAL "
  redis.call('SET', KEYS[1], ARGV[1])
  redis.call('INCR', KEYS[2])
  return redis.call('GET', KEYS[2])
" 2 mykey counter myvalue
```

### Connection Pooling

```python
# Python example with connection pooling
import redis

# Connection pool
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=20,
    retry_on_timeout=True
)

r = redis.Redis(connection_pool=pool)
```

### Monitoring and Profiling

```bash
# Monitor commands in real-time
MONITOR

# Slow query log
CONFIG SET slowlog-log-slower-than 10000  # 10ms
SLOWLOG GET 10

# Client connections
CLIENT LIST
CLIENT TRACKING ON

# Performance stats
INFO stats
INFO commandstats
```

**🎯 Interview Insight**: Always mention that Redis is single-threaded for command execution, so blocking operations can affect overall performance. Understanding this helps explain why pipelining and non-blocking operations are crucial.

---

## Common Interview Questions & Answers

### 1. "How does Redis achieve such high performance?"

**Key Points:**
- Single-threaded command execution eliminates lock contention
- In-memory storage with optimized data structures
- Efficient network I/O with epoll/kqueue
- Smart encoding selection based on data characteristics
- Pipelining support reduces network round trips

### 2. "Explain the trade-offs between Redis data types"

**Answer Framework:**
- **Memory vs. Access Pattern**: Hash vs. String for objects
- **Accuracy vs. Memory**: HyperLogLog vs. Set for counting
- **Simplicity vs. Features**: List vs. Stream for queues
- **Query Flexibility vs. Memory**: Sorted Set's dual structure

### 3. "How would you design a real-time leaderboard?"

**Solution:**
```bash
# Use Sorted Set with score as points
ZADD leaderboard 1500 "player1" 1200 "player2"

# Real-time updates
ZINCRBY leaderboard 100 "player1"

# Get rankings
ZREVRANGE leaderboard 0 9 WITHSCORES  # Top 10
ZREVRANK leaderboard "player1"        # Player rank
```

### 4. "How do you handle Redis memory limitations?"

**Strategies:**
- Use appropriate data types and encodings
- Implement expiration policies
- Use Redis Cluster for horizontal scaling
- Monitor memory usage and optimize keys
- Consider data archival strategies

### 5. "Explain Redis persistence options and their trade-offs"

**RDB vs AOF:**
- **RDB**: Point-in-time snapshots, compact, faster restarts
- **AOF**: Command logging, better durability, larger files
- **Hybrid**: RDB + AOF for best of both worlds

---

## Conclusion

Understanding Redis data types and their underlying data structures is crucial for:

- **Optimal Performance**: Choosing the right data type for your use case
- **Memory Efficiency**: Leveraging Redis's encoding optimizations
- **Scalability**: Designing systems that work well with Redis's architecture
- **Reliability**: Understanding persistence and replication implications

Remember that Redis's power comes from its simplicity and the careful engineering of its data structures. Each data type is optimized for specific access patterns and use cases.

**Final Interview Tip**: Always relate theoretical knowledge to practical scenarios. Demonstrate understanding by explaining not just *what* Redis does, but *why* it makes those design choices and *when* to use each feature.