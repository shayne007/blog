---
title: 'Redis Data Types and Data Structures: Complete Guide'
date: 2025-06-09 21:40:29
tags: [redis]
categories: [redis]
---

Redis is an in-memory data structure store that serves as a database, cache, and message broker. Understanding its data types and their underlying implementations is crucial for optimal performance and design decisions in production systems.

## String Data Type and SDS Implementation

### What is SDS (Simple Dynamic String)?

Redis implements strings using Simple Dynamic String (SDS) instead of traditional C strings. This design choice addresses several limitations of C strings and provides additional functionality.

{% mermaid graph TD %}
    A[SDS Structure] --> B[len: used length]
    A --> C[alloc: allocated space]
    A --> D[flags: type info]
    A --> E[buf: character array]
    
    F[C String] --> G[null-terminated]
    F --> H[no length info]
    F --> I[buffer overflow risk]
{% endmermaid %}

### SDS vs C String Comparison

| Feature | C String | SDS |
|---------|----------|-----|
| Length tracking | O(n) strlen() | O(1) access |
| Memory safety | Buffer overflow risk | Safe append operations |
| Binary safety | Null-terminated only | Can store binary data |
| Memory efficiency | Fixed allocation | Dynamic resizing |

### SDS Implementation Details

```c
struct sdshdr {
    int len;      // used length
    int free;     // available space
    char buf[];   // character array
};
```

**Key advantages:**
- **O(1) length operations**: No need to traverse the string
- **Buffer overflow prevention**: Automatic memory management
- **Binary safe**: Can store any byte sequence
- **Space pre-allocation**: Reduces memory reallocations

### String's Internal encoding/representation
- **RAW**
Used for strings longer than 44 bytes (in Redis 6.0+, previously 39 bytes)
Stores the string data in a separate memory allocation
Uses a standard SDS (Simple Dynamic String) structure
More memory overhead due to separate allocation, but handles large strings efficiently
- **EMBSTR (Embedded String)**
Used for strings 44 bytes or shorter that cannot be represented as integers
Embeds the string data directly within the Redis object structure in a single memory allocation
More memory-efficient than RAW for short strings
Read-only - if you modify an EMBSTR, Redis converts it to RAW encoding
- **INT**
Used when the string value can be represented as a 64-bit signed integer
Examples: "123", "-456", "0"
Stores the integer directly in the Redis object structure
Most memory-efficient encoding for numeric strings
Redis can perform certain operations (like INCR) directly on the integer without conversion

### String Use Cases and Examples

#### User Session Caching
```bash
# Store user session data
SET user:session:12345 '{"user_id":12345,"username":"john","role":"admin"}'
EXPIRE user:session:12345 3600

# Retrieve session
GET user:session:12345
```

#### Atomic Counters
Page view counter
```bash
INCR page:views:homepage
INCRBY user:points:123 50
```
Rate limiting
```python
def is_rate_limited(user_id, limit=100, window=3600):
    key = f"rate_limit:{user_id}"
    current = r.incr(key)
    if current == 1:
        r.expire(key, window)
    return current > limit
```

#### Distributed Locks with SETNX
```python
import redis
import time
import uuid

def acquire_lock(redis_client, lock_key, timeout=10):
    identifier = str(uuid.uuid4())
    end = time.time() + timeout
    
    while time.time() < end:
        if redis_client.set(lock_key, identifier, nx=True, ex=timeout):
            return identifier
        time.sleep(0.001)
    return False

def release_lock(redis_client, lock_key, identifier):
    pipe = redis_client.pipeline(True)
    while True:
        try:
            pipe.watch(lock_key)
            if pipe.get(lock_key) == identifier:
                pipe.multi()
                pipe.delete(lock_key)
                pipe.execute()
                return True
            pipe.unwatch()
            break
        except redis.WatchError:
            pass
    return False
```

**Interview Question**: *Why does Redis use SDS instead of C strings?*
**Answer**: SDS provides O(1) length operations, prevents buffer overflows, supports binary data, and offers efficient memory management through pre-allocation strategies, making it superior for database operations.

## List Data Type Implementation

### Evolution of List Implementation

Redis lists have evolved through different implementations:

{% mermaid graph LR %}
    A[Redis < 3.2] --> B[Doubly Linked List + Ziplist]
    B --> C[Redis >= 3.2] 
    C --> D[Quicklist Only]
    
    E[Quicklist] --> F[Linked List of Ziplists]
    E --> G[Memory Efficient]
    E --> H[Cache Friendly]
{% endmermaid %}

### Quicklist Structure
Quicklist combines the benefits of linked lists and ziplists:
- **Linked list of ziplists**: Each node contains a compressed ziplist
- **Configurable compression**: LZ4 compression for memory efficiency
- **Balanced performance**: Good for both ends, operations, and memory usage

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
    unsigned int count;     // Elements in ziplist
} quicklistNode;
```

### List Use Cases and Examples

#### Message Queue Implementation
```python
# Producer
def send_message(redis_client, queue_name, message):
    redis_client.lpush(queue_name, json.dumps(message))

# Consumer
def consume_messages(redis_client, queue_name):
    while True:
        message = redis_client.brpop(queue_name, timeout=1)
        if message:
            process_message(json.loads(message[1]))
```

#### Latest Articles List
```bash
# Add new article (keep only latest 100)
LPUSH latest:articles:tech "article:12345"
LTRIM latest:articles:tech 0 99

# Get the latest 10 articles
LRANGE latest:articles:tech 0 9

# Timeline implementation
LPUSH user:timeline:123 "post:456"
LRANGE user:timeline:123 0 19  # Get latest 20 posts
```

#### Activity Feed
```python
def add_activity(user_id, activity):
    key = f"feed:{user_id}"
    redis_client.lpush(key, json.dumps(activity))
    redis_client.ltrim(key, 0, 999)  # Keep latest 1000 activities
    redis_client.expire(key, 86400 * 7)  # Expire in 7 days
```

**Interview Question**: *How would you implement a reliable message queue using Redis lists?*
**Answer**: Use BRPOPLPUSH for atomic move operations between queues, implement acknowledgment patterns with backup queues, and use Lua scripts for complex atomic operations.

## Set Data Type Implementation

### Dual Implementation Strategy

Redis sets use different underlying structures based on data characteristics:

{% mermaid flowchart TD %}
    A[Redis Set] --> B{Data Type Check}
    B -->|All Integers| C{Size Check}
    B -->|Mixed Types| D[Hash Table]
    
    C -->|Small| E[IntSet]
    C -->|Large| D
    
    E --> F[Memory Compact]
    E --> G[Sorted Array]
    D --> H["Fast O(1) Operations"]
    D --> I[Higher Memory Usage]
{% endmermaid %}

### IntSet Implementation

```c
typedef struct intset {
    uint32_t encoding;  // INTSET_ENC_INT16/32/64
    uint32_t length;    // Number of elements
    int8_t contents[];  // Sorted array of integers
} intset;
```

### IntSet vs Hash Table
- **IntSet**: Used when all elements are integers and the set size is small
- **Hash Table**: Used for larger sets or sets containing non-integer values

### Set Use Cases and Examples

#### Tag System
```bash
# Add tags to articles
SADD article:123:tags "redis" "database" "nosql"
SADD article:456:tags "redis" "caching" "performance"

# Find articles with a specific tag
SINTER article:123:tags article:456:tags  # Common tags
SUNION article:123:tags article:456:tags  # All tags
```
```python
class TagSystem:
    def __init__(self):
        self.redis = redis.Redis()
    
    def add_tags(self, item_id, tags):
        """Add tags to an item"""
        key = f"item:tags:{item_id}"
        return self.redis.sadd(key, *tags)
    
    def get_tags(self, item_id):
        """Get all tags for an item"""
        key = f"item:tags:{item_id}"
        return self.redis.smembers(key)
    
    def find_items_with_all_tags(self, tags):
        """Find items that have ALL specified tags"""
        tag_keys = [f"tag:items:{tag}" for tag in tags]
        return self.redis.sinter(*tag_keys)
    
    def find_items_with_any_tags(self, tags):
        """Find items that have ANY of the specified tags"""
        tag_keys = [f"tag:items:{tag}" for tag in tags]
        return self.redis.sunion(*tag_keys)

# Usage
tag_system = TagSystem()

# Tag some articles
tag_system.add_tags("article:1", ["python", "redis", "database"])
tag_system.add_tags("article:2", ["python", "web", "flask"])

# Find articles with both "python" and "redis"
matching_articles = tag_system.find_items_with_all_tags(["python", "redis"])
```

#### User Interests and Recommendations
```python
def add_user_interest(user_id, interest):
    redis_client.sadd(f"user:{user_id}:interests", interest)

def find_similar_users(user_id):
    user_interests = f"user:{user_id}:interests"
    similar_users = []
    
    # Find users with common interests
    for other_user in get_all_users():
        if other_user != user_id:
            common_interests = redis_client.sinter(
                user_interests, 
                f"user:{other_user}:interests"
            )
            if len(common_interests) >= 3:  # Threshold
                similar_users.append(other_user)
    
    return similar_users
```
#### Social Features: Mutual Friends

```python
def find_mutual_friends(user1_id, user2_id):
    """Find mutual friends between two users"""
    friends1_key = f"user:friends:{user1_id}"
    friends2_key = f"user:friends:{user2_id}"
    
    return r.sinter(friends1_key, friends2_key)

def suggest_friends(user_id, limit=10):
    """Suggest friends based on mutual connections"""
    user_friends_key = f"user:friends:{user_id}"
    user_friends = r.smembers(user_friends_key)
    
    suggestions = set()
    
    for friend_id in user_friends:
        friend_friends_key = f"user:friends:{friend_id}"
        # Get friends of friends, excluding current user and existing friends
        potential_friends = r.sdiff(friend_friends_key, user_friends_key, user_id)
        suggestions.update(potential_friends)
    
    return list(suggestions)[:limit]
```
#### Online Users Tracking
```bash
# Track online users
SADD online:users "user:123" "user:456"
SREM online:users "user:789"

# Check if user is online
SISMEMBER online:users "user:123"

# Get all online users
SMEMBERS online:users
```

**Interview Question**: *When would Redis choose IntSet over Hash Table for sets?*
**Answer**: IntSet is chosen when all elements are integers and the set size is below the configured threshold (default 512 elements), providing better memory efficiency and acceptable O(log n) performance.

## Sorted Set (ZSet) Implementation

### Hybrid Data Structure

Sorted Sets use a combination of two data structures for optimal performance:

{% mermaid graph TD %}
    A[Sorted Set] --> B[Skip List]
    A --> C[Hash Table]
    
    B --> D["Range queries O(log n)"]
    B --> E[Ordered iteration]
    
    C --> F["Score lookup O(1)"]
    C --> G["Member existence O(1)"]
    
    H[Small ZSets] --> I[Ziplist]
    I --> J[Memory efficient]
    I --> K[Linear scan acceptable]
{% endmermaid %}
### Skip List Structure

```c
typedef struct zskiplistNode {
    sds ele;                          // Member
    double score;                     // Score
    struct zskiplistNode *backward;   // Previous node
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;           // Distance to next node
    } level[];
} zskiplistNode;
```
### Skip List Advantages
- **Probabilistic data structure**: Average O(log n) complexity
- **Range query friendly**: Efficient ZRANGE operations
- **Memory efficient**: Less overhead than balanced trees
- **Simple implementation**: Easier to maintain than AVL/Red-Black trees

### ZSet Use Cases and Examples

#### Leaderboard System
```python
class GameLeaderboard:
    def __init__(self, game_id):
        self.redis = redis.Redis()
        self.leaderboard_key = f"game:leaderboard:{game_id}"
    
    def update_score(self, player_id, score):
        """Update player's score (higher score = better rank)"""
        return self.redis.zadd(self.leaderboard_key, {player_id: score})
    
    def get_top_players(self, count=10):
        """Get top N players with their scores"""
        return self.redis.zrevrange(
            self.leaderboard_key, 0, count-1, withscores=True
        )
    
    def get_player_rank(self, player_id):
        """Get player's current rank (0-based)"""
        rank = self.redis.zrevrank(self.leaderboard_key, player_id)
        return rank + 1 if rank is not None else None
    
    def get_players_in_range(self, min_score, max_score):
        """Get players within score range"""
        return self.redis.zrangebyscore(
            self.leaderboard_key, min_score, max_score, withscores=True
        )

# Usage
leaderboard = GameLeaderboard("tetris")

# Update scores
leaderboard.update_score("player1", 15000)
leaderboard.update_score("player2", 12000)
leaderboard.update_score("player3", 18000)

# Get top 5 players
top_players = leaderboard.get_top_players(5)
print(f"Top players: {top_players}")

# Get specific player rank
rank = leaderboard.get_player_rank("player1")
print(f"Player1 rank: {rank}")
```

#### Time-based Delayed Queue
```python
import time
import json

class DelayedJobQueue:
    def __init__(self, queue_name):
        self.redis = redis.Redis()
        self.queue_key = f"delayed_jobs:{queue_name}"
    
    def schedule_job(self, job_data, delay_seconds):
        """Schedule a job to run after delay_seconds"""
        execute_at = time.time() + delay_seconds
        job_id = f"job:{int(time.time() * 1000000)}"  # Microsecond timestamp
        
        # Store job data
        job_key = f"job_data:{job_id}"
        self.redis.setex(job_key, delay_seconds + 3600, json.dumps(job_data))
        
        # Schedule execution
        return self.redis.zadd(self.queue_key, {job_id: execute_at})
    
    def get_ready_jobs(self, limit=10):
        """Get jobs ready to be executed"""
        now = time.time()
        ready_jobs = self.redis.zrangebyscore(
            self.queue_key, 0, now, start=0, num=limit
        )
        
        if ready_jobs:
            # Remove from queue atomically
            pipe = self.redis.pipeline()
            for job_id in ready_jobs:
                pipe.zrem(self.queue_key, job_id)
            pipe.execute()
        
        return ready_jobs
    
    def get_job_data(self, job_id):
        """Retrieve job data"""
        job_key = f"job_data:{job_id}"
        data = self.redis.get(job_key)
        return json.loads(data) if data else None

# Usage
delayed_queue = DelayedJobQueue("email_notifications")

# Schedule an email to be sent in 1 hour
delayed_queue.schedule_job({
    "type": "email",
    "to": "user@example.com",
    "template": "reminder",
    "data": {"name": "John"}
}, 3600)
```

#### Trending Content
```bash
# Track content popularity with time decay
ZADD trending:articles 1609459200 "article:123"
ZADD trending:articles 1609462800 "article:456"

# Get trending articles (recent first)
ZREVRANGE trending:articles 0 9 WITHSCORES

# Clean old entries
ZREMRANGEBYSCORE trending:articles 0 (current_timestamp - 86400)
```

**Interview Question**: *Why does Redis use both skip list and hash table for sorted sets?*
**Answer**: Skip list enables efficient range operations and ordered traversal in O(log n), while hash table provides O(1) score lookups and member existence checks. This dual structure optimizes for all sorted set operations.

## Hash Data Type Implementation

### Adaptive Data Structure

Redis hashes optimize memory usage through conditional implementation:

{% mermaid graph TD %}
    A[Redis Hash] --> B{Small hash?}
    B -->|Yes| C[Ziplist]
    B -->|No| D[Hash Table]
    
    C --> E[Sequential key-value pairs]
    C --> F[Memory efficient]
    C --> G["O(n) field access"]
    
    D --> H[Separate chaining]
    D --> I["O(1) average access"]
    D --> J[Higher memory overhead]
{% endmermaid %}

### Configuration Thresholds
```bash
# Hash conversion thresholds
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
```

### Hash Use Cases and Examples

#### Object Storage
```python
class UserProfileManager:
    def __init__(self):
        self.redis = redis.Redis()
    
    def save_profile(self, user_id, profile_data):
        """Save user profile as hash"""
        key = f"user:profile:{user_id}"
        return self.redis.hset(key, mapping=profile_data)
    
    def get_profile(self, user_id):
        """Get complete user profile"""
        key = f"user:profile:{user_id}"
        return self.redis.hgetall(key)
    
    def update_field(self, user_id, field, value):
        """Update single profile field"""
        key = f"user:profile:{user_id}"
        return self.redis.hset(key, field, value)
    
    def get_field(self, user_id, field):
        """Get single profile field"""
        key = f"user:profile:{user_id}"
        return self.redis.hget(key, field)
    
    def increment_counter(self, user_id, counter_name, amount=1):
        """Increment a counter field"""
        key = f"user:profile:{user_id}"
        return self.redis.hincrby(key, counter_name, amount)

# Usage
profile_manager = UserProfileManager()

# Save user profile
profile_manager.save_profile("user:123", {
    "name": "Alice Johnson",
    "email": "alice@example.com",
    "age": "28",
    "city": "San Francisco",
    "login_count": "0",
    "last_login": str(int(time.time()))
})

# Update specific field
profile_manager.update_field("user:123", "city", "New York")

# Increment login counter
profile_manager.increment_counter("user:123", "login_count")
```

#### Shopping Cart
```python
def add_to_cart(user_id, product_id, quantity):
    cart_key = f"cart:{user_id}"
    redis_client.hset(cart_key, product_id, quantity)
    redis_client.expire(cart_key, 86400 * 7)  # 7 days

def get_cart_items(user_id):
    return redis_client.hgetall(f"cart:{user_id}")

def update_cart_quantity(user_id, product_id, quantity):
    if quantity <= 0:
        redis_client.hdel(f"cart:{user_id}", product_id)
    else:
        redis_client.hset(f"cart:{user_id}", product_id, quantity)
```

#### Configuration Management
```bash
# Application configuration
HSET app:config:prod "db_host" "prod-db.example.com"
HSET app:config:prod "cache_ttl" "3600"
HSET app:config:prod "max_connections" "100"

# Feature flags
HSET features:flags "new_ui" "enabled"
HSET features:flags "beta_feature" "disabled"
```

**Interview Question**: *When would you choose Hash over String for storing objects?*
**Answer**: Use Hash when you need to access individual fields frequently without deserializing the entire object, when the object has many fields, or when you want to use Redis hash-specific operations like HINCRBY for counters within objects.

## Advanced Data Types

### Bitmap: Space-Efficient Boolean Arrays
Bitmaps in Redis are strings that support bit-level operations. Each bit can represent a Boolean state for a specific ID or position.

{% mermaid graph LR %}
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

#### Use Case 1: User Activity Tracking
```bash
# Daily active users (bit position = user ID)
SETBIT daily_active:2024-01-15 1001 1  # User 1001 was active
SETBIT daily_active:2024-01-15 1002 1  # User 1002 was active

# Check if user was active
GETBIT daily_active:2024-01-15 1001

# Count active users
BITCOUNT daily_active:2024-01-15
```
```python
class UserActivityTracker:
    def __init__(self):
        self.redis = redis.Redis()
    
    def mark_daily_active(self, user_id, date):
        """Mark user as active on specific date"""
        # Use day of year as bit position
        day_of_year = date.timetuple().tm_yday - 1  # 0-based
        key = f"daily_active:{date.year}"
        return self.redis.setbit(key, day_of_year * 1000000 + user_id, 1)
    
    def is_active_on_date(self, user_id, date):
        """Check if user was active on specific date"""
        day_of_year = date.timetuple().tm_yday - 1
        key = f"daily_active:{date.year}"
        return bool(self.redis.getbit(key, day_of_year * 1000000 + user_id))
    
    def count_active_users(self, date):
        """Count active users on specific date (simplified)"""
        key = f"daily_active:{date.year}"
        return self.redis.bitcount(key)

# Real-time analytics with bitmaps
def track_feature_usage(user_id, feature_id):
    """Track which features each user has used"""
    key = f"user:features:{user_id}"
    r.setbit(key, feature_id, 1)

def get_user_features(user_id):
    """Get all features used by user"""
    key = f"user:features:{user_id}"
    # This would need additional logic to extract set bits
    return r.get(key)
```
#### Use Case 2: Feature Flags
```bash
# Feature availability (bit position = feature ID)
SETBIT user:1001:features 0 1  # Feature 0 enabled
SETBIT user:1001:features 2 1  # Feature 2 enabled

# Check feature access
GETBIT user:1001:features 0
```

#### Use Case 3:  A/B Testing
```bash
# Test group assignment
SETBIT experiment:feature_x:group_a 1001 1
SETBIT experiment:feature_x:group_b 1002 1

# Users in both experiments
BITOP AND result experiment:feature_x:group_a experiment:other_experiment
```

**🎯 Interview Insight**: Bitmaps are extremely memory-efficient for representing large, sparse boolean datasets. One million users can be represented in just 125KB instead of several megabytes with other data structures.

### HyperLogLog: Probabilistic Counting

HyperLogLog uses probabilistic algorithms to estimate cardinality (unique count) with minimal memory usage.

{% mermaid graph TD %}
    A[HyperLogLog] --> B[Hash Function]
    B --> C[Leading Zeros Count]
    C --> D[Bucket Assignment]
    D --> E[Cardinality Estimation]
    
    E --> E1[Standard Error: 0.81%]
    E --> E2[Memory Usage: 12KB]
    E --> E3[Max Cardinality: 2^64]
{% endmermaid %}

#### Algorithm Principle
```bash
# Simplified algorithm:
# 1. Hash each element
# 2. Count leading zeros in binary representation
# 3. Use bucket system for better accuracy
# 4. Apply harmonic mean for final estimation
```

#### Use Case 1: Unique Visitors

```python
class UniqueVisitorCounter:
    def __init__(self):
        self.redis = redis.Redis()
    
    def add_visitor(self, page_id, visitor_id):
        """Add visitor to unique count"""
        key = f"unique_visitors:{page_id}"
        return self.redis.pfadd(key, visitor_id)
    
    def get_unique_count(self, page_id):
        """Get approximate unique visitor count"""
        key = f"unique_visitors:{page_id}"
        return self.redis.pfcount(key)
    
    def merge_counts(self, destination, *source_pages):
        """Merge unique visitor counts from multiple pages"""
        source_keys = [f"unique_visitors:{page}" for page in source_pages]
        dest_key = f"unique_visitors:{destination}"
        return self.redis.pfmerge(dest_key, *source_keys)

# Usage - can handle millions of unique items with ~12KB memory
visitor_counter = UniqueVisitorCounter()

# Track visitors (can handle duplicates efficiently)
for i in range(1000000):
    visitor_counter.add_visitor("homepage", f"user_{i % 50000}")  # Many duplicates

# Get unique count (approximate, typically within 1% error)
unique_count = visitor_counter.get_unique_count("homepage")
print(f"Approximate unique visitors: {unique_count}")
```

#### Use Case 2: Unique Event Counting
```bash
# Track unique events
PFADD events:login user:1001 user:1002 user:1003
PFADD events:purchase user:1001 user:1004

# Count unique users who performed any action
PFMERGE events:total events:login events:purchase
PFCOUNT events:total
```

#### Use Case 3: Real-time Analytics
```bash
# Hourly unique visitors
PFADD stats:$(date +%Y%m%d%H):unique visitor_id_1 visitor_id_2

# Daily aggregation
for hour in {00..23}; do
    PFMERGE stats:$(date +%Y%m%d):unique stats:$(date +%Y%m%d)${hour}:unique
done
```

#### Accuracy vs. Memory Trade-off

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

### Stream: Radix Tree + Consumer Groups

Redis Streams use a radix tree (compressed trie) to store entries efficiently, with additional structures for consumer group management.

{% mermaid graph TD %}
    A[Stream] --> B[Radix Tree]
    A --> C[Consumer Groups]
    
    B --> B1[Stream Entries]
    B --> B2[Time-ordered IDs]
    B --> B3[Field-Value Pairs]
    
    C --> C1[Consumer Group State]
    C --> C2[Pending Entries List - PEL]
    C --> C3[Consumer Last Delivered ID]
{% endmermaid %}

#### Stream Entry Structure
```bash
# Stream ID format: timestamp-sequence
# Example: 1640995200000-0
#          |-------------|--- |
#          timestamp(ms)     sequence
```

#### Use Case 1: Event Sourcing
```bash
# Add events to stream
XADD user:1001:events * action "login" timestamp 1640995200 ip "192.168.1.1"
XADD user:1001:events * action "purchase" item "laptop" amount 999.99

# Read events
XRANGE user:1001:events - + COUNT 10
```

#### Use Case 2: Message Queues with Consumer Groups
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

#### Use Case 3: Real-time Data Processing
```bash
# IoT sensor data
XADD sensors:temperature * sensor_id "temp001" value 23.5 location "room1"
XADD sensors:humidity * sensor_id "hum001" value 45.2 location "room1"

# Read latest data
XREAD COUNT 10 STREAMS sensors:temperature sensors:humidity $ $
```

**🎯 Interview Insight**: Streams provide at-least-once delivery guarantees through the Pending Entries List (PEL), making them suitable for reliable message processing unlike simple pub/sub.

### Geospatial: Sorted Set with Geohash

Redis geospatial features are built on top of sorted sets, using geohash as scores to enable spatial queries.

{% mermaid graph LR %}
    A[Geospatial] --> B[Sorted Set Backend]
    B --> C[Geohash as Score]
    C --> D[Spatial Queries]
    
    D --> D1[GEORADIUS]
    D --> D2[GEODIST]
    D --> D3[GEOPOS]
    D --> D4[GEOHASH]
{% endmermaid %}

#### Geohash Encoding
```bash
# Latitude/Longitude -> Geohash -> 52-bit integer
# Example: (37.7749, -122.4194) -> 9q8yy -> score for sorted set
```

#### Use Case 1: Location Services
```python
class LocationService:
    def __init__(self):
        self.redis = redis.Redis()
    
    def add_location(self, location_set, name, longitude, latitude):
        """Add a location to the geospatial index"""
        return self.redis.geoadd(location_set, longitude, latitude, name)
    
    def find_nearby(self, location_set, longitude, latitude, radius_km, unit='km'):
        """Find locations within radius"""
        return self.redis.georadius(
            location_set, longitude, latitude, radius_km, unit=unit,
            withdist=True, withcoord=True, sort='ASC'
        )
    
    def find_nearby_member(self, location_set, member_name, radius_km, unit='km'):
        """Find locations near an existing member"""
        return self.redis.georadiusbymember(
            location_set, member_name, radius_km, unit=unit,
            withdist=True, sort='ASC'
        )
    
    def get_distance(self, location_set, member1, member2, unit='km'):
        """Get distance between two members"""
        result = self.redis.geodist(location_set, member1, member2, unit=unit)
        return float(result) if result else None

# Usage - Restaurant finder
location_service = LocationService()

# Add restaurants
restaurants = [
    ("Pizza Palace", -122.4194, 37.7749),    # San Francisco
    ("Burger Barn", -122.4094, 37.7849),
    ("Sushi Spot", -122.4294, 37.7649),
]

for name, lon, lat in restaurants:
    location_service.add_location("restaurants:sf", name, lon, lat)

# Find restaurants within 2km of a location
nearby = location_service.find_nearby(
    "restaurants:sf", -122.4194, 37.7749, 2, 'km'
)
print(f"Nearby restaurants: {nearby}")
```


#### Use Case 2: Delivery Services
```bash
# Add delivery drivers
GEOADD drivers -122.4094 37.7849 "driver:1001"
GEOADD drivers -122.4294 37.7649 "driver:1002"

# Find nearby drivers
GEORADIUS drivers -122.4194 37.7749 5 km WITHCOORD ASC

# Update driver location
GEOADD drivers -122.4150 37.7800 "driver:1001"
```

#### Use Case 3: Store Locator
```bash
# Add store locations
GEOADD stores -122.4194 37.7749 "store:sf_downtown"
GEOADD stores -122.4094 37.7849 "store:sf_mission"

# Find stores by area
GEORADIUSBYMEMBER stores "store:sf_downtown" 10 km WITHCOORD
```

**🎯 Interview Insight**: Redis geospatial commands are syntactic sugar over sorted set operations. Understanding this helps explain why you can mix geospatial and sorted set commands on the same key.

---
## Choosing the Right Data Type

### Decision Matrix

{% mermaid flowchart TD %}
    A[Data Requirements] --> B{Single Value?}
    B -->|Yes| C{Need Expiration?}
    C -->|Yes| D[String with TTL]
    C -->|No| E{Counter/Numeric?}
    E -->|Yes| F[String with INCR]
    E -->|No| D
    
    B -->|No| G{Key-Value Pairs?}
    G -->|Yes| H{Large Object?}
    H -->|Yes| I[Hash]
    H -->|No| J{Frequent Field Updates?}
    J -->|Yes| I
    J -->|No| K[String with JSON]
    
    G -->|No| L{Ordered Collection?}
    L -->|Yes| M{Need Scores/Ranking?}
    M -->|Yes| N[Sorted Set]
    M -->|No| O[List]
    
    L -->|No| P{Unique Elements?}
    P -->|Yes| Q{Set Operations Needed?}
    Q -->|Yes| R[Set]
    Q -->|No| S{Memory Critical?}
    S -->|Yes| T[Bitmap/HyperLogLog]
    S -->|No| R
    
    P -->|No| O
{% endmermaid %}

### Use Case Mapping Table

| **Use Case** | **Primary Data Type** | **Alternative** | **Why This Choice** |
|--------------|----------------------|-----------------|-------------------|
| User Sessions | String | Hash | TTL support, simple storage |
| Shopping Cart | Hash | String (JSON) | Atomic field updates |
| Message Queue | List | Stream | FIFO ordering, blocking ops |
| Leaderboard | Sorted Set | - | Score-based ranking |
| Tags/Categories | Set | - | Unique elements, set operations |
| Real-time Analytics | Bitmap/HyperLogLog | - | Memory efficiency |
| Activity Feed | List | Stream | Chronological ordering |
| Friendship Graph | Set | - | Intersection operations |
| Rate Limiting | String | Hash | Counter with expiration |
| Geographic Search | Geospatial | - | Location-based queries |

### Performance Characteristics

| **Operation** | **String** | **List** | **Set** | **Sorted Set** | **Hash** |
|---------------|------------|----------|---------|----------------|----------|
| Get/Set Single | O(1) | O(1) ends | O(1) | O(log N) | O(1) |
| Range Query | N/A | O(N) | N/A | O(log N + M) | N/A |
| Add Element | O(1) | O(1) ends | O(1) | O(log N) | O(1) |
| Remove Element | N/A | O(N) | O(1) | O(log N) | O(1) |
| Size Check | O(1) | O(1) | O(1) | O(1) | O(1) |

## Best Practices and Interview Insights

### Memory Optimization Strategies

1. **Choose appropriate data types**: Use the most memory-efficient type for your use case
2. **Configure thresholds**: Tune ziplist/intset thresholds based on your data patterns
3. **Use appropriate key naming**: Consistent, predictable key patterns
4. **Implement key expiration**: Use TTL to prevent memory leaks

### Common Anti-Patterns to Avoid

```python
# ❌ Bad: Using inappropriate data type
r.set("user:123:friends", json.dumps(["friend1", "friend2", "friend3"]))

# ✅ Good: Use Set for unique collections
r.sadd("user:123:friends", "friend1", "friend2", "friend3")

# ❌ Bad: Large objects as single keys
r.set("user:123", json.dumps(large_user_object))

# ✅ Good: Use Hash for structured data
r.hset("user:123", mapping={
    "name": "John Doe",
    "email": "john@example.com",
    "age": "30"
})

# ❌ Bad: Sequential key access
for i in range(1000):
    r.get(f"key:{i}")

# ✅ Good: Use pipeline for batch operations
pipe = r.pipeline()
for i in range(1000):
    pipe.get(f"key:{i}")
results = pipe.execute()
```

### Key Design Patterns

#### Hierarchical Key Naming
```python
# Good key naming conventions
"user:12345:profile"           # User profile data
"user:12345:settings"          # User settings
"user:12345:sessions:abc123"   # User session data
"cache:article:567:content"    # Cached article content
"queue:email:high_priority"    # High priority email queue
"counter:page_views:2024:01"   # Monthly page view counter
```

#### Atomic Operations with Lua Scripts
```python
# Atomic rate limiting with sliding window
rate_limit_script = """
local key = KEYS[1]
local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local current_time = tonumber(ARGV[3])

-- Remove expired entries
redis.call('ZREMRANGEBYSCORE', key, 0, current_time - window)

-- Count current requests
local current_requests = redis.call('ZCARD', key)

if current_requests < limit then
    -- Add current request
    redis.call('ZADD', key, current_time, current_time)
    redis.call('EXPIRE', key, window)
    return {1, limit - current_requests - 1}
else
    return {0, 0}
end
"""

def check_rate_limit(user_id, limit=100, window=3600):
    """Sliding window rate limiter"""
    key = f"rate_limit:{user_id}"
    current_time = int(time.time())
    
    result = r.eval(rate_limit_script, 1, key, window, limit, current_time)
    return {
        "allowed": bool(result[0]),
        "remaining": result[1]
    }
```

## Advanced Implementation Details

### Memory Layout Optimization

{% mermaid graph TD %}
    A[Redis Object] --> B[Encoding Type]
    A --> C[Reference Count]
    A --> D[LRU Info]
    A --> E[Actual Data]
    
    B --> F[String: RAW/INT/EMBSTR]
    B --> G[List: ZIPLIST/LINKEDLIST/QUICKLIST]
    B --> H[Set: INTSET/HASHTABLE]
    B --> I[ZSet: ZIPLIST/SKIPLIST]
    B --> J[Hash: ZIPLIST/HASHTABLE]
{% endmermaid %}

### Configuration Tuning for Production

```bash
# Redis configuration optimizations
# Memory optimization
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Performance tuning
tcp-backlog 511
timeout 0
tcp-keepalive 300
maxclients 10000
maxmemory-policy allkeys-lru
```

### Monitoring and Debugging

```python
class RedisMonitor:
    def __init__(self):
        self.redis = redis.Redis()
    
    def analyze_key_patterns(self):
        """Analyze key distribution and memory usage"""
        info = {}
        
        # Get overall memory info
        memory_info = self.redis.info('memory')
        info['total_memory'] = memory_info['used_memory_human']
        
        # Sample keys for pattern analysis
        sample_keys = self.redis.randomkey() for _ in range(100)
        patterns = {}
        
        for key in sample_keys:
            if key:
                key_type = self.redis.type(key)
                pattern = key.split(':')[0] if ':' in key else 'no_pattern'
                
                if pattern not in patterns:
                    patterns[pattern] = {'count': 0, 'types': {}}
                
                patterns[pattern]['count'] += 1
                patterns[pattern]['types'][key_type] = \
                    patterns[pattern]['types'].get(key_type, 0) + 1
        
        info['key_patterns'] = patterns
        return info
    
    def get_large_keys(self, threshold_mb=1):
        """Find keys consuming significant memory"""
        large_keys = []
        
        # This would require MEMORY USAGE command (Redis 4.0+)
        # Implementation would scan keys and check memory usage
        
        return large_keys
    
    def check_data_type_efficiency(self, key):
        """Analyze if current data type is optimal"""
        key_type = self.redis.type(key)
        key_size = self.redis.memory_usage(key) if hasattr(self.redis, 'memory_usage') else 0
        
        analysis = {
            'type': key_type,
            'memory_usage': key_size,
            'recommendations': []
        }
        
        if key_type == 'string':
            # Check if it's JSON that could be a hash
            try:
                value = self.redis.get(key)
                if value and value.startswith(('{', '[')):
                    analysis['recommendations'].append(
                        "Consider using Hash if you need to update individual fields"
                    )
            except:
                pass
        
        return analysis
```

## Real-World Architecture Patterns

### Microservices Data Patterns

```python
class UserServiceRedisLayer:
    """Redis layer for user microservice"""
    
    def __init__(self):
        self.redis = redis.Redis()
        self.cache_ttl = 3600  # 1 hour
    
    def cache_user_profile(self, user_id, profile_data):
        """Cache user profile with optimized structure"""
        # Use hash for structured data
        profile_key = f"user:profile:{user_id}"
        self.redis.hset(profile_key, mapping=profile_data)
        self.redis.expire(profile_key, self.cache_ttl)
        
        # Cache frequently accessed fields separately
        self.redis.setex(f"user:name:{user_id}", self.cache_ttl, profile_data['name'])
        self.redis.setex(f"user:email:{user_id}", self.cache_ttl, profile_data['email'])
    
    def get_user_summary(self, user_id):
        """Get essential user info with fallback strategy"""
        # Try cache first
        name = self.redis.get(f"user:name:{user_id}")
        email = self.redis.get(f"user:email:{user_id}")
        
        if name and email:
            return {'name': name.decode(), 'email': email.decode()}
        
        # Fallback to full profile
        profile = self.redis.hgetall(f"user:profile:{user_id}")
        if profile:
            return {k.decode(): v.decode() for k, v in profile.items()}
        
        return None  # Cache miss, need to fetch from database

class NotificationService:
    """Redis-based notification system"""
    
    def __init__(self):
        self.redis = redis.Redis()
    
    def queue_notification(self, user_id, notification_type, data, priority='normal'):
        """Queue notification with priority"""
        queue_key = f"notifications:{priority}"
        
        notification = {
            'user_id': user_id,
            'type': notification_type,
            'data': json.dumps(data),
            'created_at': int(time.time())
        }
        
        if priority == 'high':
            # Use list for FIFO queue
            self.redis.lpush(queue_key, json.dumps(notification))
        else:
            # Use sorted set for delayed delivery
            deliver_at = time.time() + 300  # 5 minutes delay
            self.redis.zadd(queue_key, {json.dumps(notification): deliver_at})
    
    def get_user_notifications(self, user_id, limit=10):
        """Get recent notifications for user"""
        key = f"user:notifications:{user_id}"
        notifications = self.redis.lrange(key, 0, limit - 1)
        return [json.loads(n) for n in notifications]
    
    def mark_notifications_read(self, user_id, notification_ids):
        """Mark specific notifications as read"""
        read_key = f"user:notifications:read:{user_id}"
        self.redis.sadd(read_key, *notification_ids)
        self.redis.expire(read_key, 86400 * 30)  # Keep for 30 days
```

### E-commerce Platform Integration

```python
class EcommerceCacheLayer:
    """Comprehensive Redis integration for e-commerce"""
    
    def __init__(self):
        self.redis = redis.Redis()
    
    def cache_product_catalog(self, category_id, products):
        """Cache product listings with multiple access patterns"""
        # Main product list
        catalog_key = f"catalog:{category_id}"
        product_ids = [p['id'] for p in products]
        self.redis.delete(catalog_key)
        self.redis.lpush(catalog_key, *product_ids)
        self.redis.expire(catalog_key, 3600)
        
        # Cache individual products
        for product in products:
            product_key = f"product:{product['id']}"
            self.redis.hset(product_key, mapping=product)
            self.redis.expire(product_key, 7200)
            
            # Add to price-sorted index
            price_index = f"category:{category_id}:by_price"
            self.redis.zadd(price_index, {product['id']: float(product['price'])})
    
    def implement_inventory_tracking(self):
        """Real-time inventory management"""
        def reserve_inventory(product_id, quantity, user_id):
            """Atomically reserve inventory"""
            lua_script = """
            local product_key = 'inventory:' .. KEYS[1]
            local reservation_key = 'reservations:' .. KEYS[1]
            local user_reservation = KEYS[1] .. ':' .. ARGV[2]
            
            local current_stock = tonumber(redis.call('GET', product_key) or 0)
            local requested = tonumber(ARGV[1])
            
            if current_stock >= requested then
                redis.call('DECRBY', product_key, requested)
                redis.call('HSET', reservation_key, user_reservation, requested)
                redis.call('EXPIRE', reservation_key, 900)  -- 15 minutes
                return {1, current_stock - requested}
            else
                return {0, current_stock}
            end
            """
            
            result = self.redis.eval(lua_script, 1, product_id, quantity, user_id)
            return {
                'success': bool(result[0]),
                'remaining_stock': result[1]
            }
        
        return reserve_inventory
    
    def implement_recommendation_engine(self):
        """Collaborative filtering with Redis"""
        def track_user_interaction(user_id, product_id, interaction_type, score=1.0):
            """Track user-product interactions"""
            # User's interaction history
            user_key = f"user:interactions:{user_id}"
            self.redis.zadd(user_key, {f"{interaction_type}:{product_id}": score})
            
            # Product's interaction summary
            product_key = f"product:interactions:{product_id}"
            self.redis.zincrby(product_key, score, interaction_type)
            
            # Similar users (simplified)
            similar_users_key = f"similar:users:{user_id}"
            # Logic to find and cache similar users would go here
        
        def get_recommendations(user_id, limit=10):
            """Get product recommendations for user"""
            user_interactions = self.redis.zrevrange(f"user:interactions:{user_id}", 0, -1)
            
            # Simple collaborative filtering
            recommendations = set()
            for interaction in user_interactions[:5]:  # Use top 5 interactions
                interaction_type, product_id = interaction.decode().split(':', 1)
                
                # Find users who also interacted with this product
                similar_pattern = f"*:{interaction_type}:{product_id}"
                # This is simplified - real implementation would be more sophisticated
            
            return list(recommendations)[:limit]
        
        return track_user_interaction, get_recommendations

# Session management for web applications
class SessionManager:
    """Redis-based session management"""
    
    def __init__(self, session_timeout=3600):
        self.redis = redis.Redis()
        self.timeout = session_timeout
    
    def create_session(self, user_id, session_data):
        """Create new user session"""
        session_id = str(uuid.uuid4())
        session_key = f"session:{session_id}"
        
        session_info = {
            'user_id': str(user_id),
            'created_at': str(int(time.time())),
            'last_accessed': str(int(time.time())),
            **session_data
        }
        
        # Store session data
        self.redis.hset(session_key, mapping=session_info)
        self.redis.expire(session_key, self.timeout)
        
        # Track active sessions for user
        user_sessions_key = f"user:sessions:{user_id}"
        self.redis.sadd(user_sessions_key, session_id)
        self.redis.expire(user_sessions_key, self.timeout)
        
        return session_id
    
    def get_session(self, session_id):
        """Retrieve session data"""
        session_key = f"session:{session_id}"
        session_data = self.redis.hgetall(session_key)
        
        if session_data:
            # Update last accessed time
            self.redis.hset(session_key, 'last_accessed', str(int(time.time())))
            self.redis.expire(session_key, self.timeout)
            
            return {k.decode(): v.decode() for k, v in session_data.items()}
        
        return None
    
    def invalidate_session(self, session_id):
        """Invalidate specific session"""
        session_key = f"session:{session_id}"
        session_data = self.redis.hgetall(session_key)
        
        if session_data:
            user_id = session_data[b'user_id'].decode()
            user_sessions_key = f"user:sessions:{user_id}"
            
            # Remove from user's active sessions
            self.redis.srem(user_sessions_key, session_id)
            
            # Delete session
            self.redis.delete(session_key)
            
            return True
        return False
```

## Production Deployment Considerations

### High Availability Patterns

```python
class RedisHighAvailabilityClient:
    """Redis client with failover and connection pooling"""
    
    def __init__(self, sentinel_hosts, service_name, db=0):
        from redis.sentinel import Sentinel
        
        self.sentinel = Sentinel(sentinel_hosts, socket_timeout=0.1)
        self.service_name = service_name
        self.db = db
        self._master = None
        self._slaves = []
    
    def get_master(self):
        """Get master connection with automatic failover"""
        try:
            if not self._master:
                self._master = self.sentinel.master_for(
                    self.service_name, 
                    socket_timeout=0.1,
                    db=self.db
                )
            return self._master
        except Exception:
            self._master = None
            raise
    
    def get_slave(self):
        """Get slave connection for read operations"""
        try:
            return self.sentinel.slave_for(
                self.service_name,
                socket_timeout=0.1,
                db=self.db
            )
        except Exception:
            # Fallback to master if no slaves available
            return self.get_master()
    
    def execute_read(self, operation, *args, **kwargs):
        """Execute read operation on slave with master fallback"""
        try:
            slave = self.get_slave()
            return getattr(slave, operation)(*args, **kwargs)
        except Exception:
            master = self.get_master()
            return getattr(master, operation)(*args, **kwargs)
    
    def execute_write(self, operation, *args, **kwargs):
        """Execute write operation on master"""
        master = self.get_master()
        return getattr(master, operation)(*args, **kwargs)

# Usage example
ha_redis = RedisHighAvailabilityClient([
    ('sentinel1.example.com', 26379),
    ('sentinel2.example.com', 26379),
    ('sentinel3.example.com', 26379)
], 'mymaster')

# Read from slave, write to master
user_data = ha_redis.execute_read('hgetall', 'user:123')
ha_redis.execute_write('hset', 'user:123', 'last_login', str(int(time.time())))
```

### Performance Monitoring

```python
class RedisPerformanceMonitor:
    """Monitor Redis performance metrics"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
        self.metrics = {}
    
    def collect_metrics(self):
        """Collect comprehensive Redis metrics"""
        info = self.redis.info()
        
        return {
            'memory': {
                'used_memory': info['used_memory'],
                'used_memory_human': info['used_memory_human'],
                'used_memory_peak': info['used_memory_peak'],
                'fragmentation_ratio': info.get('mem_fragmentation_ratio', 0)
            },
            'performance': {
                'ops_per_sec': info.get('instantaneous_ops_per_sec', 0),
                'keyspace_hits': info.get('keyspace_hits', 0),
                'keyspace_misses': info.get('keyspace_misses', 0),
                'hit_rate': self._calculate_hit_rate(info)
            },
            'connections': {
                'connected_clients': info.get('connected_clients', 0),
                'rejected_connections': info.get('rejected_connections', 0)
            },
            'persistence': {
                'rdb_last_save_time': info.get('rdb_last_save_time', 0),
                'aof_enabled': info.get('aof_enabled', 0)
            }
        }
    
    def _calculate_hit_rate(self, info):
        """Calculate cache hit rate"""
        hits = info.get('keyspace_hits', 0)
        misses = info.get('keyspace_misses', 0)
        total = hits + misses
        
        return (hits / total * 100) if total > 0 else 0
    
    def detect_performance_issues(self, metrics):
        """Detect common performance issues"""
        issues = []
        
        # High memory fragmentation
        if metrics['memory']['fragmentation_ratio'] > 1.5:
            issues.append("High memory fragmentation detected")
        
        # Low hit rate
        if metrics['performance']['hit_rate'] < 80:
            issues.append("Low cache hit rate - consider key expiration strategy")
        
        # High connection count
        if metrics['connections']['connected_clients'] > 1000:
            issues.append("High connection count - consider connection pooling")
        
        return issues
```

## Key Takeaways and Interview Excellence

### Essential Concepts to Master

1. **Data Structure Selection**: Understanding when and why to use each Redis data type
2. **Memory Optimization**: How Redis optimizes memory usage through encoding strategies
3. **Performance Characteristics**: Big O complexity of operations across data types
4. **Real-world Applications**: Practical use cases and implementation patterns
5. **Production Considerations**: Scaling, monitoring, and high availability

### Critical Interview Questions and Expert Answers

**Q: "How does Redis decide between ziplist and skiplist for sorted sets?"**

**Expert Answer**: Redis uses configuration thresholds (`zset-max-ziplist-entries` and `zset-max-ziplist-value`) to decide. When elements ≤ 128 and values ≤ 64 bytes, it uses ziplist for memory efficiency. Beyond these thresholds, it switches to skiplist + hashtable for better performance. This dual approach optimizes for both memory usage (small sets) and operation speed (large sets).

**Q: "Explain the trade-offs between using Redis Hash vs storing JSON strings."**

**Expert Answer**: Hash provides atomic field operations (HSET, HINCRBY), memory efficiency for small objects, and partial updates without deserializing entire objects. JSON strings offer simplicity, better compatibility across languages, and easier complex queries. Choose Hash for frequently updated structured data, JSON for read-heavy or complex nested data.

**Q: "How would you implement a distributed rate limiter using Redis?"**

**Expert Answer**: Use sliding window with sorted sets or fixed window with strings. Sliding window stores timestamps as scores in sorted set, removes expired entries, counts current requests. Fixed window uses string counters with expiration. Lua scripts ensure atomicity. Consider token bucket for burst handling.

**Q: "What are the memory implications of Redis data type choices?"**

**Expert Answer**: Strings have 20+ bytes overhead, Lists use ziplist (compact) vs quicklist (flexible), Sets use intset (integers only) vs hashtable, Sorted Sets use ziplist vs skiplist, Hashes use ziplist vs hashtable. Understanding these encodings is crucial for memory optimization in production.

### Redis Command Cheat Sheet for Data Types

```bash
# String Operations
SET key value [EX seconds] [NX|XX]
GET key
INCR key / INCRBY key increment
MSET key1 value1 key2 value2
MGET key1 key2

# List Operations  
LPUSH key value1 [value2 ...]
RPOP key
LRANGE key start stop
BLPOP key [key ...] timeout
LTRIM key start stop

# Set Operations
SADD key member1 [member2 ...]
SMEMBERS key
SINTER key1 [key2 ...]
SUNION key1 [key2 ...]
SDIFF key1 [key2 ...]

# Sorted Set Operations
ZADD key score1 member1 [score2 member2 ...]
ZRANGE key start stop [WITHSCORES]
ZRANGEBYSCORE key min max
ZRANK key member
ZINCRBY key increment member

# Hash Operations
HSET key field1 value1 [field2 value2 ...]
HGET key field
HGETALL key
HINCRBY key field increment
HDEL key field1 [field2 ...]

# Advanced Data Types
SETBIT key offset value
BITCOUNT key [start end]
PFADD key element [element ...]
PFCOUNT key [key ...]
GEOADD key longitude latitude member
GEORADIUS key longitude latitude radius unit
XADD stream-key ID field1 value1 [field2 value2 ...]
XREAD [COUNT count] STREAMS key [key ...] ID [ID ...]
```

## Conclusion

Redis's elegance lies in its thoughtful data type design and implementation strategies. Each data type addresses specific use cases while maintaining excellent performance characteristics. The key to mastering Redis is understanding not just what each data type does, but why it's implemented that way and when to use it.

For production systems, consider data access patterns, memory constraints, and scaling requirements when choosing data types. Redis's flexibility allows for creative solutions, but with great power comes the responsibility to choose wisely.

The combination of simple operations, rich data types, and high performance makes Redis an indispensable tool for modern application architecture. Whether you're building caches, message queues, real-time analytics, or complex data structures, Redis provides the foundation for scalable, efficient solutions.

## External References

- [Redis Official Documentation](https://redis.io/documentation)
- [Redis Data Types Deep Dive](https://redis.io/topics/data-types)
- [Redis Memory Optimization](https://redis.io/topics/memory-optimization)
- [Redis Persistence](https://redis.io/topics/persistence)
- [Redis Sentinel](https://redis.io/topics/sentinel)
- [Redis Cluster](https://redis.io/topics/cluster-tutorial)
- [Redis Streams](https://redis.io/topics/streams-intro)
- [Redis Modules](https://redis.io/modules)