---
title: Redis Caching Patterns and Consistency Assurance
date: 2025-06-10 18:47:55
tags: [redis]
categories: [redis]
---
## Introduction

Redis serves as a high-performance in-memory data structure store, commonly used as a cache, database, and message broker. Understanding caching patterns and consistency mechanisms is crucial for building scalable, reliable systems.

**🎯 Interview Insight**: *Interviewers often ask about the trade-offs between performance and consistency. Be prepared to discuss CAP theorem implications and when to choose eventual consistency over strong consistency.*

### Why Caching Matters
- **Reduced Latency**: Sub-millisecond response times for cached data
- **Decreased Database Load**: Offloads read operations from primary databases
- **Improved Scalability**: Handles higher concurrent requests
- **Cost Efficiency**: Reduces expensive database operations
- 
### Key Benefits of Redis Caching
- **Performance**: Sub-millisecond latency for most operations
- **Scalability**: Handles millions of requests per second
- **Flexibility**: Rich data structures (strings, hashes, lists, sets, sorted sets)
- **Persistence**: Optional durability with RDB/AOF
- **High Availability**: Redis Sentinel and Cluster support

## Core Caching Patterns

### 1. Cache-Aside (Lazy Loading)

The application manages the cache directly, loading data on cache misses.

{% mermaid sequenceDiagram %}
    participant App as Application
    participant Cache as Redis Cache
    participant DB as Database
    
    App->>Cache: GET user:123
    Cache-->>App: Cache Miss (null)
    App->>DB: SELECT * FROM users WHERE id=123
    DB-->>App: User data
    App->>Cache: SET user:123 {user_data} EX 3600
    Cache-->>App: OK
    App-->>App: Return user data
{% endmermaid %}

**Implementation Example:**
```python
import redis
import json
import time

class CacheAsidePattern:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
        self.cache_ttl = 3600  # 1 hour
    
    def get_user(self, user_id):
        cache_key = f"user:{user_id}"
        
        # Try cache first
        cached_data = self.redis_client.get(cache_key)
        if cached_data:
            return json.loads(cached_data)
        
        # Cache miss - fetch from database
        user_data = self.fetch_user_from_db(user_id)
        if user_data:
            # Store in cache with TTL
            self.redis_client.setex(
                cache_key, 
                self.cache_ttl, 
                json.dumps(user_data)
            )
        
        return user_data
    
    def update_user(self, user_id, user_data):
        # Update database
        self.update_user_in_db(user_id, user_data)
        
        # Invalidate cache
        cache_key = f"user:{user_id}"
        self.redis_client.delete(cache_key)
        
        return user_data
```
**Pros:**
- Simple to implement and understand
- Cache only contains requested data
- Resilient to cache failures

**Cons:**
- Cache miss penalty (extra database call)
- Potential cache stampede issues
- Data staleness between updates

**💡 Interview Insight**: *Discuss cache stampede scenarios: multiple requests hitting the same missing key simultaneously. Solutions include distributed locking or probabilistic refresh.*

### 2. Write-Through

Data is written to both cache and database simultaneously.

{% mermaid sequenceDiagram %}
    participant App as Application
    participant Cache as Redis Cache
    participant DB as Database
    
    App->>Cache: SET key data
    Cache->>DB: UPDATE data
    DB-->>Cache: Success
    Cache-->>App: Success
    
    Note over App,DB: Read requests served directly from cache
{% endmermaid %}

**Implementation Example:**
```python
class WriteThroughPattern:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
    
    def save_user(self, user_id, user_data):
        cache_key = f"user:{user_id}"
        
        try:
            # Write to database first
            self.save_user_to_db(user_id, user_data)
            
            # Then write to cache
            self.redis_client.set(cache_key, json.dumps(user_data))
            
            return True
        except Exception as e:
            # Rollback cache if database write fails
            self.redis_client.delete(cache_key)
            raise e
    
    def get_user(self, user_id):
        cache_key = f"user:{user_id}"
        cached_data = self.redis_client.get(cache_key)
        
        if cached_data:
            return json.loads(cached_data)
        
        # This should rarely happen in write-through
        return self.fetch_user_from_db(user_id)

```
**Pros:**
- Data consistency between cache and database
- Fast read performance
- No cache miss penalty for written data

**Cons:**
- Higher write latency
- Writes to cache even for data that may never be read
- More complex error handling


### 3. Write-Behind (Write-Back)

Data is written to cache immediately and to the database asynchronously.

{% mermaid sequenceDiagram %}
    participant App as Application
    participant Cache as Redis Cache
    participant Queue as Write Queue
    participant DB as Database
    
    App->>Cache: SET user:123 {updated_data}
    Cache-->>App: OK (immediate)
    Cache->>Queue: Enqueue write operation
    Queue->>DB: Async UPDATE users
    DB-->>Queue: Success
{% endmermaid %}
**Implementation Example:**

```python
import asyncio
from queue import Queue
import threading

class WriteBehindPattern:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
        self.write_queue = Queue()
        self.start_background_writer()
    
    def save_user(self, user_id, user_data):
        cache_key = f"user:{user_id}"
        
        # Immediate cache update
        self.redis_client.set(cache_key, json.dumps(user_data))
        
        # Queue for database write
        self.write_queue.put({
            'action': 'update',
            'user_id': user_id,
            'data': user_data,
            'timestamp': time.time()
        })
        
        return user_data
    
    def start_background_writer(self):
        def worker():
            while True:
                try:
                    item = self.write_queue.get(timeout=1)
                    self.process_write(item)
                    self.write_queue.task_done()
                except:
                    continue
        
        thread = threading.Thread(target=worker, daemon=True)
        thread.start()
    
    def process_write(self, item):
        try:
            self.save_user_to_db(item['user_id'], item['data'])
        except Exception as e:
            # Implement retry logic or dead letter queue
            self.handle_write_failure(item, e)
```

**Pros:**
- Excellent write performance
- Reduced database load
- Can batch writes for efficiency

**Cons:**
- Risk of data loss on cache failure
- Complex failure handling
- Eventual consistency only

**🎯 Interview Insight**: *Write-behind offers better write performance but introduces complexity and potential data loss risks. Discuss scenarios where this pattern is appropriate (high write volume, acceptable eventual consistency,some data loss is acceptable, like analytics or logging systems).*

### 4. Refresh-Ahead

Proactively refresh cache entries before they expire.

```python
import asyncio
from datetime import datetime, timedelta

class RefreshAheadCache:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        self.refresh_threshold = 0.8  # Refresh when 80% of TTL is reached
    
    async def get_with_refresh_ahead(self, key: str):
        data = self.redis.get(key)
        ttl = self.redis.ttl(key)
        
        if data and ttl > 0:
            # Check if refresh is needed
            if ttl < (self.cache_ttl * (1 - self.refresh_threshold)):
                # Trigger async refresh
                asyncio.create_task(self.refresh_cache_entry(key))
            
            return json.loads(data)
        
        # Cache miss or expired
        return await self.load_and_cache(key)
    
    async def refresh_cache_entry(self, key: str):
        fresh_data = await self.fetch_fresh_data(key)
        if fresh_data:
            self.redis.setex(key, self.cache_ttl, json.dumps(fresh_data))
```

## Consistency Models and Strategies

### 1. Strong Consistency with Distributed Locks

Ensures all reads receive the most recent write. Implemented using distributed locks or transactions.

```python
import time
import uuid

class DistributedLock:
    def __init__(self, redis_client, key, timeout=10):
        self.redis = redis_client
        self.key = f"lock:{key}"
        self.timeout = timeout
        self.identifier = str(uuid.uuid4())
    
    def acquire(self):
        end = time.time() + self.timeout
        while time.time() < end:
            if self.redis.setnx(self.key, self.identifier):
                self.redis.expire(self.key, self.timeout)
                return True
            time.sleep(0.001)
        return False
    
    def release(self):
        # Lua script ensures atomicity
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        return self.redis.eval(lua_script, 1, self.key, self.identifier)

# Usage in cache update
def update_user_with_lock(user_id, user_data):
    lock = DistributedLock(redis_client, f"user:{user_id}")
    
    if lock.acquire():
        try:
            # Update database
            update_user_in_db(user_id, user_data)
            
            # Update cache
            cache_key = f"user:{user_id}"
            redis_client.set(cache_key, json.dumps(user_data))
            
        finally:
            lock.release()
    else:
        raise Exception("Could not acquire lock")
```

**🎯 Interview Insight**: *Strong consistency comes with performance costs. Discuss scenarios where it's necessary (financial transactions, inventory management) vs. where eventual consistency is acceptable (user profiles, social media posts).*

### 2. Eventual Consistency

Updates propagate through the system over time, allowing temporary inconsistencies.

```python
import time
from threading import Thread
from queue import Queue

class EventualConsistencyHandler:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        self.update_queue = Queue()
        self.worker_thread = Thread(target=self._process_updates, daemon=True)
        self.worker_thread.start()
    
    def update_user_async(self, user_id: int, updates: Dict):
        # Immediate cache update for read performance
        cache_key = f"user:{user_id}"
        current_cached = self.redis.get(cache_key)
        
        if current_cached:
            current_data = json.loads(current_cached)
            updated_data = {**current_data, **updates}
            self.redis.setex(cache_key, 3600, json.dumps(updated_data))
        
        # Queue database update
        self.update_queue.put((user_id, updates, time.time()))
    
    def _process_updates(self):
        while True:
            try:
                user_id, updates, timestamp = self.update_queue.get(timeout=1)
                
                # Process database update
                self.update_database_with_retry(user_id, updates, timestamp)
                
                # Verify cache consistency
                self._verify_consistency(user_id)
                
            except Exception as e:
                # Handle failed updates (DLQ, alerts, etc.)
                self.handle_update_failure(user_id, updates, e)
```

### 3. Read-Your-Writes Consistency

Guarantees that a user will see their own writes immediately.

```python
class ReadYourWritesCache:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        self.user_versions = {}  # Track user-specific versions
    
    def write_user_data(self, user_id: int, data: Dict, session_id: str):
        # Increment version for this user
        version = self.redis.incr(f"user_version:{user_id}")
        
        # Store data with version
        cache_key = f"user:{user_id}"
        versioned_data = {**data, "_version": version, "_updated_by": session_id}
        
        # Write to cache and database
        self.redis.setex(cache_key, 3600, json.dumps(versioned_data))
        self.update_database(user_id, data)
        
        # Track version for this session
        self.user_versions[session_id] = version
    
    def read_user_data(self, user_id: int, session_id: str) -> Dict:
        cache_key = f"user:{user_id}"
        cached_data = self.redis.get(cache_key)
        
        if cached_data:
            data = json.loads(cached_data)
            cached_version = data.get("_version", 0)
            expected_version = self.user_versions.get(session_id, 0)
            
            # Ensure user sees their own writes
            if cached_version >= expected_version:
                return data
        
        # Fallback to database for consistency
        return self.fetch_from_database(user_id)
```

## Advanced Patterns

### 1. Cache Warming

Pre-populate cache with frequently accessed data.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class CacheWarmer:
    def __init__(self, redis_client, batch_size=100):
        self.redis = redis_client
        self.batch_size = batch_size
    
    async def warm_user_cache(self, user_ids: List[int]):
        """Warm cache for multiple users concurrently"""
        
        async def warm_single_user(user_id: int):
            try:
                user_data = await self.fetch_user_from_db(user_id)
                if user_data:
                    cache_key = f"user:{user_id}"
                    self.redis.setex(
                        cache_key,
                        3600,
                        json.dumps(user_data)
                    )
                    return True
            except Exception as e:
                print(f"Failed to warm cache for user {user_id}: {e}")
                return False
        
        # Process in batches to avoid overwhelming the system
        for i in range(0, len(user_ids), self.batch_size):
            batch = user_ids[i:i + self.batch_size]
            tasks = [warm_single_user(uid) for uid in batch]
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            success_count = sum(1 for r in results if r is True)
            print(f"Warmed {success_count}/{len(batch)} cache entries")
            
            # Small delay between batches
            await asyncio.sleep(0.1)
    
    def warm_on_startup(self):
        """Warm cache with most accessed data on application startup"""
        popular_users = self.get_popular_user_ids()
        asyncio.run(self.warm_user_cache(popular_users))
```

### 2. Multi-Level Caching

Implement multiple cache layers for optimal performance.

{% mermaid graph TD %}
    A[Application] --> B[L1 Cache - Local Memory]
    B --> C[L2 Cache - Redis]
    C --> D[L3 Cache - CDN]
    D --> E[Database]
    
    style B fill:#e1f5fe
    style C fill:#f3e5f5
    style D fill:#e8f5e8
    style E fill:#fff3e0
{% endmermaid %}

```python
from functools import lru_cache
import time

class MultiLevelCache:
    def __init__(self):
        # L1: Local memory cache (LRU)
        self.l1_cache = {}
        self.l1_access_times = {}
        self.l1_max_size = 1000
        
        # L2: Redis cache
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        
        # L3: Persistent cache (database cache table)
        self.db_cache = DatabaseCacheLayer()
    
    def get(self, key: str) -> Optional[any]:
        # L1 Cache check
        if key in self.l1_cache:
            self.l1_access_times[key] = time.time()
            return self.l1_cache[key]
        
        # L2 Cache check (Redis)
        l2_data = self.redis.get(key)
        if l2_data:
            value = json.loads(l2_data)
            self._store_in_l1(key, value)
            return value
        
        # L3 Cache check (Database cache)
        l3_data = self.db_cache.get(key)
        if l3_data:
            # Populate upper levels
            self.redis.setex(key, 3600, json.dumps(l3_data))
            self._store_in_l1(key, l3_data)
            return l3_data
        
        # Cache miss - fetch from origin
        return None
    
    def set(self, key: str, value: any, ttl: int = 3600):
        # Store in all levels
        self._store_in_l1(key, value)
        self.redis.setex(key, ttl, json.dumps(value))
        self.db_cache.set(key, value, ttl)
    
    def _store_in_l1(self, key: str, value: any):
        # Implement LRU eviction
        if len(self.l1_cache) >= self.l1_max_size:
            self._evict_lru()
        
        self.l1_cache[key] = value
        self.l1_access_times[key] = time.time()
    
    def _evict_lru(self):
        # Remove least recently used item
        lru_key = min(self.l1_access_times, key=self.l1_access_times.get)
        del self.l1_cache[lru_key]
        del self.l1_access_times[lru_key]
```

**🎯 Interview Insight**: *Multi-level caching questions often focus on cache coherence. Discuss strategies for maintaining consistency across levels and the trade-offs between complexity and performance.*

## Data Invalidation Strategies

### 1. TTL-Based Expiration

```python
class TTLInvalidationStrategy:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        
        # Different TTL strategies for different data types
        self.ttl_config = {
            'user_profile': 3600,      # 1 hour
            'user_preferences': 86400,  # 24 hours
            'session_data': 1800,       # 30 minutes
            'product_catalog': 300,     # 5 minutes
            'real_time_data': 60        # 1 minute
        }
    
    def set_with_appropriate_ttl(self, key: str, value: any, data_type: str):
        ttl = self.ttl_config.get(data_type, 3600)  # Default 1 hour
        
        # Add jitter to prevent thundering herd
        jitter = random.randint(-60, 60)  # ±1 minute
        final_ttl = max(ttl + jitter, 60)  # Minimum 1 minute
        
        self.redis.setex(key, final_ttl, json.dumps(value))
```

### 2. Event-Driven Invalidation

```python
import pika
import json

class EventDrivenInvalidation:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        
        # Set up exchange and queue
        self.channel.exchange_declare(
            exchange='cache_invalidation',
            exchange_type='topic'
        )
    
    def invalidate_user_cache(self, user_id: int, event_type: str):
        """Invalidate cache based on user events"""
        
        patterns_to_invalidate = {
            'user_updated': [f"user:{user_id}", f"user_profile:{user_id}"],
            'user_preferences_changed': [f"user_prefs:{user_id}"],
            'user_deleted': [f"user:*:{user_id}", f"session:*:{user_id}"],
        }
        
        keys_to_invalidate = patterns_to_invalidate.get(event_type, [])
        
        for pattern in keys_to_invalidate:
            if '*' in pattern:
                # Handle wildcard patterns
                matching_keys = self.redis.keys(pattern)
                if matching_keys:
                    self.redis.delete(*matching_keys)
            else:
                self.redis.delete(pattern)
        
        # Publish invalidation event
        self.channel.basic_publish(
            exchange='cache_invalidation',
            routing_key=f'user.{event_type}',
            body=json.dumps({
                'user_id': user_id,
                'event_type': event_type,
                'timestamp': time.time(),
                'invalidated_keys': keys_to_invalidate
            })
        )
    
    def setup_invalidation_listener(self):
        """Listen for cache invalidation events"""
        
        def callback(ch, method, properties, body):
            try:
                event = json.loads(body)
                print(f"Cache invalidation event: {event}")
                # Additional processing if needed
                
            except Exception as e:
                print(f"Error processing invalidation event: {e}")
        
        queue = self.channel.queue_declare(queue='cache_invalidation_processor')
        self.channel.queue_bind(
            exchange='cache_invalidation',
            queue=queue.method.queue,
            routing_key='user.*'
        )
        
        self.channel.basic_consume(
            queue=queue.method.queue,
            on_message_callback=callback,
            auto_ack=True
        )
        
        self.channel.start_consuming()
```

### 3. Cache Tags and Dependencies

```python
class TagBasedInvalidation:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379, db=0)
    
    def set_with_tags(self, key: str, value: any, tags: List[str], ttl: int = 3600):
        """Store data with associated tags for bulk invalidation"""
        
        # Store the actual data
        self.redis.setex(key, ttl, json.dumps(value))
        
        # Associate key with tags
        for tag in tags:
            tag_key = f"tag:{tag}"
            self.redis.sadd(tag_key, key)
            self.redis.expire(tag_key, ttl + 300)  # Tags live longer than data
    
    def invalidate_by_tag(self, tag: str):
        """Invalidate all cache entries associated with a tag"""
        
        tag_key = f"tag:{tag}"
        
        # Get all keys associated with this tag
        keys_to_invalidate = self.redis.smembers(tag_key)
        
        if keys_to_invalidate:
            # Delete all associated keys
            self.redis.delete(*keys_to_invalidate)
            
            # Clean up tag associations
            for key in keys_to_invalidate:
                self._remove_key_from_all_tags(key.decode())
        
        # Remove the tag itself
        self.redis.delete(tag_key)
    
    def _remove_key_from_all_tags(self, key: str):
        """Remove a key from all tag associations"""
        
        # This could be expensive - consider background cleanup
        tag_pattern = "tag:*"
        for tag_key in self.redis.scan_iter(match=tag_pattern):
            self.redis.srem(tag_key, key)

# Usage example
cache = TagBasedInvalidation()

# Store user data with tags
user_data = {"name": "John", "department": "Engineering"}
cache.set_with_tags(
    key="user:123",
    value=user_data,
    tags=["user", "department:engineering", "active_users"]
)

# Invalidate all engineering department data
cache.invalidate_by_tag("department:engineering")
```

**🎯 Interview Insight**: *Tag-based invalidation is a sophisticated pattern. Discuss the trade-offs between granular control and storage overhead. Mention alternatives like dependency graphs for complex invalidation scenarios.*

## Performance Optimization

### 1. Connection Pooling and Pipelining

```python
import redis.connection
from redis import Redis
from redis.connection import ConnectionPool

class OptimizedRedisClient:
    def __init__(self):
        # Connection pool for better resource management
        self.pool = ConnectionPool(
            host='localhost',
            port=6379,
            db=0,
            max_connections=20,
            socket_connect_timeout=5,
            socket_timeout=5,
            retry_on_timeout=True
        )
        self.redis = Redis(connection_pool=self.pool)
    
    def batch_get_users(self, user_ids: List[int]) -> Dict[int, Dict]:
        """Efficiently fetch multiple users using pipelining"""
        
        pipe = self.redis.pipeline()
        
        # Queue multiple commands
        cache_keys = [f"user:{uid}" for uid in user_ids]
        for key in cache_keys:
            pipe.get(key)
        
        # Execute all commands at once
        results = pipe.execute()
        
        # Process results
        user_data = {}
        for i, result in enumerate(results):
            if result:
                user_data[user_ids[i]] = json.loads(result)
        
        return user_data
    
    def batch_set_users(self, user_data_map: Dict[int, Dict]):
        """Efficiently store multiple users using pipelining"""
        
        pipe = self.redis.pipeline()
        
        for user_id, data in user_data_map.items():
            cache_key = f"user:{user_id}"
            pipe.setex(cache_key, 3600, json.dumps(data))
        
        # Execute all commands
        pipe.execute()
```

### 2. Memory Optimization

```python
class MemoryOptimizedCache:
    def __init__(self):
        self.redis = Redis(host='localhost', port=6379, db=0)
    
    def store_user_efficiently(self, user_id: int, user_data: Dict):
        """Use Redis hashes for memory efficiency with structured data"""
        
        hash_key = f"user:{user_id}"
        
        # Store as hash instead of JSON string
        # This is more memory efficient for structured data
        mapping = {
            'name': user_data.get('name', ''),
            'email': user_data.get('email', ''),
            'created_at': str(user_data.get('created_at', '')),
            'is_active': '1' if user_data.get('is_active') else '0'
        }
        
        self.redis.hset(hash_key, mapping=mapping)
        self.redis.expire(hash_key, 3600)
    
    def get_user_efficiently(self, user_id: int) -> Optional[Dict]:
        """Retrieve user data from hash"""
        
        hash_key = f"user:{user_id}"
        user_hash = self.redis.hgetall(hash_key)
        
        if not user_hash:
            return None
        
        # Convert back to proper types
        return {
            'name': user_hash.get(b'name', b'').decode(),
            'email': user_hash.get(b'email', b'').decode(),
            'created_at': user_hash.get(b'created_at', b'').decode(),
            'is_active': user_hash.get(b'is_active') == b'1'
        }
    
    def compress_large_data(self, key: str, data: any):
        """Compress large data before storing"""
        import gzip
        
        json_data = json.dumps(data)
        compressed_data = gzip.compress(json_data.encode())
        
        # Store with compression flag
        self.redis.hset(f"compressed:{key}", mapping={
            'data': compressed_data,
            'compressed': '1'
        })
    
    def get_compressed_data(self, key: str) -> Optional[any]:
        """Retrieve and decompress data"""
        import gzip
        
        result = self.redis.hgetall(f"compressed:{key}")
        if not result:
            return None
        
        if result.get(b'compressed') == b'1':
            compressed_data = result.get(b'data')
            json_data = gzip.decompress(compressed_data).decode()
            return json.loads(json_data)
        
        return None
```

### 3. Hot Key Detection and Mitigation

```python
import threading
from collections import defaultdict, deque
import time

class HotKeyDetector:
    def __init__(self, threshold=100, window_seconds=60):
        self.redis = Redis(host='localhost', port=6379, db=0)
        self.threshold = threshold
        self.window_seconds = window_seconds
        
        # Track key access patterns
        self.access_counts = defaultdict(deque)
        self.lock = threading.RLock()
        
        # Hot key mitigation strategies
        self.hot_keys = set()
        self.local_cache = {}  # Local caching for hot keys
    
    def track_access(self, key: str):
        """Track key access for hot key detection"""
        current_time = time.time()
        
        with self.lock:
            # Add current access
            self.access_counts[key].append(current_time)
            
            # Remove old accesses outside the window
            cutoff_time = current_time - self.window_seconds
            while (self.access_counts[key] and 
                   self.access_counts[key][0] < cutoff_time):
                self.access_counts[key].popleft()
            
            # Check if key is hot
            if len(self.access_counts[key]) > self.threshold:
                if key not in self.hot_keys:
                    self.hot_keys.add(key)
                    self._handle_hot_key(key)
    
    def get_with_hot_key_handling(self, key: str):
        """Get data with hot key optimization"""
        self.track_access(key)
        
        # If it's a hot key, try local cache first
        if key in self.hot_keys:
            local_data = self.local_cache.get(key)
            if local_data and local_data['expires'] > time.time():
                return local_data['value']
        
        # Get from Redis
        data = self.redis.get(key)
        
        # Cache locally if hot key
        if key in self.hot_keys and data:
            self.local_cache[key] = {
                'value': data,
                'expires': time.time() + 30  # Short local cache TTL
            }
        
        return data
    
    def _handle_hot_key(self, key: str):
        """Implement hot key mitigation strategies"""
        
        # Strategy 1: Add local caching
        print(f"Hot key detected: {key} - enabling local caching")
        
        # Strategy 2: Create multiple copies with random distribution
        original_data = self.redis.get(key)
        if original_data:
            for i in range(3):  # Create 3 copies
                copy_key = f"{key}:copy:{i}"
                self.redis.setex(copy_key, 300, original_data)  # 5 min TTL
        
        # Strategy 3: Use read replicas (if available)
        # This would involve routing reads to replica nodes
    
    def get_distributed_hot_key(self, key: str):
        """Get hot key data using distribution strategy"""
        if key not in self.hot_keys:
            return self.redis.get(key)
        
        # Random selection from copies
        import random
        copy_index = random.randint(0, 2)
        copy_key = f"{key}:copy:{copy_index}"
        
        data = self.redis.get(copy_key)
        if not data:
            # Fallback to original
            data = self.redis.get(key)
        
        return data
```

**🎯 Interview Insight**: *Hot key problems are common in production. Discuss identification techniques (monitoring access patterns), mitigation strategies (local caching, key distribution), and prevention approaches (better key design, load balancing).*

## Monitoring and Troubleshooting

### 1. Performance Monitoring

```python
import time
import logging
from functools import wraps

class RedisMonitor:
    def __init__(self):
        self.redis = Redis(host='localhost', port=6379, db=0)
        self.metrics = {
            'hits': 0,
            'misses': 0,
            'errors': 0,
            'total_requests': 0,
            'total_latency': 0,
            'slow_queries': 0
        }
        self.slow_query_threshold = 0.1  # 100ms
        
        # Setup logging
        self.logger = logging.getLogger('redis_monitor')
        handler = logging.StreamHandler()
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)
    
    def monitor_operation(self, operation_name='redis_op'):
        """Decorator to monitor Redis operations"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                start_time = time.time()
                try:
                    result = func(*args, **kwargs)
                    
                    # Track hit/miss
                    if result is not None:
                        self.metrics['hits'] += 1
                    else:
                        self.metrics['misses'] += 1
                    
                    return result
                    
                except Exception as e:
                    self.metrics['errors'] += 1
                    self.logger.error(f"Redis operation failed: {operation_name} - {e}")
                    raise
                    
                finally:
                    # Track latency
                    end_time = time.time()
                    latency = end_time - start_time
                    self.metrics['total_requests'] += 1
                    self.metrics['total_latency'] += latency
                    
                    # Log slow queries
                    if latency > self.slow_query_threshold:
                        self.metrics['slow_queries'] += 1
                        self.logger.warning(
                            f"Slow Redis operation: {operation_name} - {latency:.3f}s"
                        )
            
            return wrapper
        return decorator
    
    @monitor_operation('get')
    def monitored_get(self, key: str):
        return self.redis.get(key)
    
    @monitor_operation('set')
    def monitored_set(self, key: str, value: str, ex: int = None):
        return self.redis.set(key, value, ex=ex)
    
    def get_performance_stats(self) -> Dict:
        """Get current performance statistics"""
        total_requests = self.metrics['total_requests']
        if total_requests == 0:
            return {'error': 'No requests recorded'}
        
        hit_rate = self.metrics['hits'] / total_requests * 100
        avg_latency = self.metrics['total_latency'] / total_requests * 1000  # ms
        error_rate = self.metrics['errors'] / total_requests * 100
        
        return {
            'hit_rate': f"{hit_rate:.2f}%",
            'miss_rate': f"{100 - hit_rate:.2f}%",
            'error_rate': f"{error_rate:.2f}%",
            'avg_latency_ms': f"{avg_latency:.2f}",
            'total_requests': total_requests,
            'slow_queries': self.metrics['slow_queries'],
            'slow_query_rate': f"{self.metrics['slow_queries'] / total_requests * 100:.2f}%"
        }
    
    def get_redis_info(self) -> Dict:
        """Get Redis server information"""
        info = self.redis.info()
        
        return {
            'version': info.get('redis_version'),
            'uptime': info.get('uptime_in_seconds'),
            'connected_clients': info.get('connected_clients'),
            'used_memory': info.get('used_memory_human'),
            'used_memory_peak': info.get('used_memory_peak_human'),
            'keyspace_hits': info.get('keyspace_hits'),
            'keyspace_misses': info.get('keyspace_misses'),
            'expired_keys': info.get('expired_keys'),
            'evicted_keys': info.get('evicted_keys')
        }
    
    def health_check(self) -> Dict:
        """Comprehensive health check"""
        try:
            # Test basic connectivity
            start_time = time.time()
            ping_result = self.redis.ping()
            ping_latency = (time.time() - start_time) * 1000
            
            # Get memory info
            info = self.redis.info('memory')
            used_memory_pct = (info['used_memory'] / info['maxmemory'] * 100 
                             if info.get('maxmemory', 0) > 0 else 0)
            
            # Check for concerning patterns
            warnings = []
            if ping_latency > 10:  # 10ms
                warnings.append(f"High ping latency: {ping_latency:.2f}ms")
            
            if used_memory_pct > 80:
                warnings.append(f"High memory usage: {used_memory_pct:.1f}%")
            
            if self.metrics['errors'] > self.metrics['total_requests'] * 0.01:  # >1% error rate
                warnings.append("High error rate detected")
            
            return {
                'status': 'healthy' if not warnings else 'warning',
                'ping_latency_ms': f"{ping_latency:.2f}",
                'memory_usage_pct': f"{used_memory_pct:.1f}%",
                'warnings': warnings,
                'performance_stats': self.get_performance_stats()
            }
            
        except Exception as e:
            return {
                'status': 'unhealthy',
                'error': str(e)
            }

# Usage example
monitor = RedisMonitor()

# Use monitored operations
data = monitor.monitored_get("user:123")
monitor.monitored_set("user:123", json.dumps({"name": "John"}), ex=3600)

# Check performance
print(monitor.get_performance_stats())
print(monitor.health_check())
```

### 2. Advanced Debugging and Profiling

```python
class RedisDebugger:
    def __init__(self):
        self.redis = Redis(host='localhost', port=6379, db=0)
        self.command_history = deque(maxlen=1000)  # Keep last 1000 commands
    
    def debug_key_access_pattern(self, key_pattern: str, duration: int = 60):
        """Monitor access patterns for keys matching a pattern"""
        
        print(f"Monitoring key pattern: {key_pattern} for {duration} seconds")
        
        # Use Redis MONITOR command (use with caution in production)
        pubsub = self.redis.pubsub()
        
        access_stats = defaultdict(int)
        start_time = time.time()
        
        try:
            # Note: MONITOR is expensive and should not be used in production
            # This is for debugging purposes only
            with self.redis.monitor() as monitor:
                for command in monitor.listen():
                    if time.time() - start_time > duration:
                        break
                    
                    if command['command']:
                        cmd_parts = command['command'].split()
                        if len(cmd_parts) >= 2:
                            operation = cmd_parts[0].upper()
                            key = cmd_parts[1]
                            
                            if key_pattern in key:
                                access_stats[f"{operation}:{key}"] += 1
                                
        except KeyboardInterrupt:
            pass
        
        # Analyze patterns
        print("\nAccess Pattern Analysis:")
        for pattern, count in sorted(access_stats.items(), key=lambda x: x[1], reverse=True):
            print(f"{pattern}: {count} accesses")
        
        return access_stats
    
    def analyze_memory_usage(self, sample_size: int = 100):
        """Analyze memory usage of different key patterns"""
        
        memory_stats = {}
        
        # Get random sample of keys
        keys = []
        for key in self.redis.scan_iter(count=sample_size):
            keys.append(key.decode())
        
        print(f"Analyzing memory usage for {len(keys)} keys...")
        
        for key in keys:
            try:
                # Get memory usage for this key
                memory_usage = self.redis.memory_usage(key)
                key_type = self.redis.type(key).decode()
                
                pattern = self._extract_key_pattern(key)
                
                if pattern not in memory_stats:
                    memory_stats[pattern] = {
                        'total_memory': 0,
                        'count': 0,
                        'avg_memory': 0,
                        'type': key_type
                    }
                
                memory_stats[pattern]['total_memory'] += memory_usage
                memory_stats[pattern]['count'] += 1
                memory_stats[pattern]['avg_memory'] = (
                    memory_stats[pattern]['total_memory'] / 
                    memory_stats[pattern]['count']
                )
                
            except Exception as e:
                print(f"Error analyzing key {key}: {e}")
        
        # Sort by total memory usage
        sorted_stats = sorted(
            memory_stats.items(), 
            key=lambda x: x[1]['total_memory'], 
            reverse=True
        )
        
        print("\nMemory Usage Analysis:")
        print(f"{'Pattern':<30} {'Type':<10} {'Count':<8} {'Total (bytes)':<15} {'Avg (bytes)':<12}")
        print("-" * 85)
        
        for pattern, stats in sorted_stats:
            print(f"{pattern:<30} {stats['type']:<10} {stats['count']:<8} "
                  f"{stats['total_memory']:<15} {stats['avg_memory']:<12.1f}")
        
        return memory_stats
    
    def _extract_key_pattern(self, key: str) -> str:
        """Extract pattern from key (e.g., user:123 -> user:*"""
        parts = key.split(':')
        if len(parts) > 1:
            # Replace numeric parts with *
            pattern_parts = []
            for part in parts:
                if part.isdigit():
                    pattern_parts.append('*')
                else:
                    pattern_parts.append(part)
            return ':'.join(pattern_parts)
        return key
    
    def find_large_keys(self, threshold_bytes: int = 1024) -> List[Dict]:
        """Find keys that consume more memory than threshold"""
        
        large_keys = []
        
        for key in self.redis.scan_iter():
            try:
                key_str = key.decode()
                memory_usage = self.redis.memory_usage(key)
                
                if memory_usage > threshold_bytes:
                    key_info = {
                        'key': key_str,
                        'memory_bytes': memory_usage,
                        'type': self.redis.type(key).decode(),
                        'ttl': self.redis.ttl(key)
                    }
                    
                    # Additional info based on type
                    key_type = key_info['type']
                    if key_type == 'string':
                        key_info['length'] = self.redis.strlen(key)
                    elif key_type == 'list':
                        key_info['length'] = self.redis.llen(key)
                    elif key_type == 'set':
                        key_info['length'] = self.redis.scard(key)
                    elif key_type == 'hash':
                        key_info['length'] = self.redis.hlen(key)
                    elif key_type == 'zset':
                        key_info['length'] = self.redis.zcard(key)
                    
                    large_keys.append(key_info)
                    
            except Exception as e:
                print(f"Error checking key {key}: {e}")
        
        # Sort by memory usage
        large_keys.sort(key=lambda x: x['memory_bytes'], reverse=True)
        
        print(f"\nFound {len(large_keys)} keys larger than {threshold_bytes} bytes:")
        for key_info in large_keys[:10]:  # Show top 10
            print(f"Key: {key_info['key']}")
            print(f"  Memory: {key_info['memory_bytes']} bytes")
            print(f"  Type: {key_info['type']}")
            print(f"  Length: {key_info.get('length', 'N/A')}")
            print(f"  TTL: {key_info['ttl']} seconds")
            print()
        
        return large_keys
    
    def connection_pool_stats(self):
        """Get connection pool statistics"""
        if hasattr(self.redis, 'connection_pool'):
            pool = self.redis.connection_pool
            return {
                'created_connections': pool.created_connections,
                'available_connections': len(pool._available_connections),
                'in_use_connections': len(pool._in_use_connections),
                'max_connections': pool.max_connections
            }
        return {'error': 'Connection pool info not available'}

# Usage example
debugger = RedisDebugger()

# Analyze memory usage
memory_stats = debugger.analyze_memory_usage(sample_size=500)

# Find large keys
large_keys = debugger.find_large_keys(threshold_bytes=10240)  # 10KB threshold

# Check connection pool
pool_stats = debugger.connection_pool_stats()
print(f"Connection pool stats: {pool_stats}")
```

**🎯 Interview Insight**: *Debugging questions often focus on production issues. Discuss tools like Redis MONITOR (and its performance impact), MEMORY USAGE command, and the importance of having proper monitoring in place before issues occur.*

## Production Best Practices

### 1. High Availability Setup

{% mermaid graph TB %}
    subgraph "Redis Sentinel Cluster"
        S1[Sentinel 1]
        S2[Sentinel 2] 
        S3[Sentinel 3]
    end
    
    subgraph "Redis Instances"
        M[Master]
        R1[Replica 1]
        R2[Replica 2]
    end
    
    subgraph "Application Layer"
        A1[App Instance 1]
        A2[App Instance 2]
        A3[App Instance 3]
    end
    
    S1 -.-> M
    S1 -.-> R1
    S1 -.-> R2
    S2 -.-> M
    S2 -.-> R1
    S2 -.-> R2
    S3 -.-> M
    S3 -.-> R1
    S3 -.-> R2
    
    A1 --> S1
    A2 --> S2
    A3 --> S3
    
    M --> R1
    M --> R2
    
    style M fill:#ff6b6b
    style R1 fill:#4ecdc4
    style R2 fill:#4ecdc4
    style S1 fill:#ffe66d
    style S2 fill:#ffe66d
    style S3 fill:#ffe66d
{% endmermaid %}

```python
import redis.sentinel

class HighAvailabilityRedisClient:
    def __init__(self):
        # Redis Sentinel configuration
        self.sentinels = [
            ('sentinel1.example.com', 26379),
            ('sentinel2.example.com', 26379),
            ('sentinel3.example.com', 26379)
        ]
        
        self.sentinel = redis.sentinel.Sentinel(
            self.sentinels,
            socket_timeout=0.5,
            socket_connect_timeout=0.5
        )
        
        self.master_name = 'mymaster'
        self.master = None
        self.slaves = []
        
        self._initialize_connections()
    
    def _initialize_connections(self):
        """Initialize master and slave connections"""
        try:
            # Get master connection
            self.master = self.sentinel.master_for(
                self.master_name,
                socket_timeout=0.5,
                socket_connect_timeout=0.5,
                retry_on_timeout=True,
                db=0
            )
            
            # Get slave connections for read operations
            self.slave = self.sentinel.slave_for(
                self.master_name,
                socket_timeout=0.5,
                socket_connect_timeout=0.5,
                retry_on_timeout=True,
                db=0
            )
            
            print("Redis HA connections initialized successfully")
            
        except Exception as e:
            print(f"Failed to initialize Redis HA connections: {e}")
            raise
    
    def get(self, key: str, use_slave: bool = True):
        """Get data, optionally from slave for read scaling"""
        try:
            if use_slave and self.slave:
                return self.slave.get(key)
            else:
                return self.master.get(key)
        except redis.ConnectionError:
            # Failover handling
            self._handle_connection_error()
            # Retry with master
            return self.master.get(key)
    
    def set(self, key: str, value: str, ex: int = None):
        """Set data (always use master for writes)"""
        try:
            return self.master.set(key, value, ex=ex)
        except redis.ConnectionError:
            self._handle_connection_error()
            return self.master.set(key, value, ex=ex)
    
    def _handle_connection_error(self):
        """Handle connection errors and potential failover"""
        print("Redis connection error detected, reinitializing connections...")
        try:
            self._initialize_connections()
        except Exception as e:
            print(f"Failed to reinitialize connections: {e}")
            raise
    
    def health_check(self) -> Dict:
        """Check health of Redis cluster"""
        health_status = {
            'master_available': False,
            'slaves_available': 0,
            'sentinel_status': [],
            'overall_status': 'unhealthy'
        }
        
        # Check master
        try:
            self.master.ping()
            health_status['master_available'] = True
        except:
            pass
        
        # Check slaves
        try:
            self.slave.ping()
            health_status['slaves_available'] = 1  # Simplified
        except:
            pass
        
        # Check sentinels
        for sentinel_host, sentinel_port in self.sentinels:
            try:
                sentinel_conn = redis.Redis(host=sentinel_host, port=sentinel_port)
                sentinel_conn.ping()
                health_status['sentinel_status'].append({
                    'host': sentinel_host,
                    'port': sentinel_port,
                    'status': 'healthy'
                })
            except:
                health_status['sentinel_status'].append({
                    'host': sentinel_host,
                    'port': sentinel_port,
                    'status': 'unhealthy'
                })
        
        # Determine overall status
        healthy_sentinels = sum(1 for s in health_status['sentinel_status'] 
                              if s['status'] == 'healthy')
        
        if (health_status['master_available'] and 
            healthy_sentinels >= 2):  # Quorum
            health_status['overall_status'] = 'healthy'
        elif healthy_sentinels >= 2:
            health_status['overall_status'] = 'degraded'
        
        return health_status
```

### 2. Security Best Practices

```python
import hashlib
import hmac
import ssl
from cryptography.fernet import Fernet

class SecureRedisClient:
    def __init__(self):
        # SSL/TLS configuration
        self.redis = redis.Redis(
            host='redis.example.com',
            port=6380,  # TLS port
            password='your-strong-password',
            ssl=True,
            ssl_cert_reqs=ssl.CERT_REQUIRED,
            ssl_ca_certs='/path/to/ca-cert.pem',
            ssl_certfile='/path/to/client-cert.pem',
            ssl_keyfile='/path/to/client-key.pem'
        )
        
        # Encryption for sensitive data
        self.encryption_key = Fernet.generate_key()
        self.cipher = Fernet(self.encryption_key)
        
        # Rate limiting
        self.rate_limiter = RateLimiter()
    
    def set_encrypted(self, key: str, value: str, ex: int = None):
        """Store encrypted data"""
        # Encrypt sensitive data
        encrypted_value = self.cipher.encrypt(value.encode())
        
        # Add integrity check
        checksum = hashlib.sha256(value.encode()).hexdigest()
        
        data_with_checksum = {
            'data': encrypted_value.decode(),
            'checksum': checksum
        }
        
        return self.redis.set(key, json.dumps(data_with_checksum), ex=ex)
    
    def get_encrypted(self, key: str) -> Optional[str]:
        """Retrieve and decrypt data"""
        encrypted_data = self.redis.get(key)
        if not encrypted_data:
            return None
        
        try:
            data_dict = json.loads(encrypted_data)
            encrypted_value = data_dict['data'].encode()
            stored_checksum = data_dict['checksum']
            
            # Decrypt
            decrypted_value = self.cipher.decrypt(encrypted_value).decode()
            
            # Verify integrity
            computed_checksum = hashlib.sha256(decrypted_value.encode()).hexdigest()
            if not hmac.compare_digest(stored_checksum, computed_checksum):
                raise ValueError("Data integrity check failed")
            
            return decrypted_value
            
        except Exception as e:
            print(f"Failed to decrypt data: {e}")
            return None
    
    def secure_session_management(self, session_id: str, user_id: int, 
                                 session_data: Dict, ttl: int = 3600):
        """Secure session management with Redis"""
        
        # Create secure session key
        session_key = f"session:{hashlib.sha256(session_id.encode()).hexdigest()}"
        
        # Session data with security metadata
        secure_session_data = {
            'user_id': user_id,
            'created_at': time.time(),
            'ip_address': session_data.get('ip_address'),
            'user_agent_hash': hashlib.sha256(
                session_data.get('user_agent', '').encode()
            ).hexdigest(),
            'data': session_data.get('data', {})
        }
        
        # Store encrypted session
        self.set_encrypted(session_key, json.dumps(secure_session_data), ex=ttl)
        
        # Track active sessions for user
        user_sessions_key = f"user_sessions:{user_id}"
        self.redis.sadd(user_sessions_key, session_key)
        self.redis.expire(user_sessions_key, ttl)
        
        return session_key
    
    def validate_session(self, session_id: str, ip_address: str, 
                        user_agent: str) -> Optional[Dict]:
        """Validate session with security checks"""
        
        session_key = f"session:{hashlib.sha256(session_id.encode()).hexdigest()}"
        
        session_data_str = self.get_encrypted(session_key)
        if not session_data_str:
            return None
        
        try:
            session_data = json.loads(session_data_str)
            
            # Security validations
            user_agent_hash = hashlib.sha256(user_agent.encode()).hexdigest()
            
            if session_data.get('user_agent_hash') != user_agent_hash:
                print("Session validation failed: User agent mismatch")
                self.invalidate_session(session_id)
                return None
            
            # Optional: IP address validation (be careful with load balancers)
            if session_data.get('ip_address') != ip_address:
                print("Session validation failed: IP address changed")
                # You might want to require re-authentication instead of invalidating
            
            return session_data
            
        except Exception as e:
            print(f"Session validation error: {e}")
            return None
    
    def invalidate_session(self, session_id: str):
        """Securely invalidate a session"""
        session_key = f"session:{hashlib.sha256(session_id.encode()).hexdigest()}"
        
        # Get user ID before deleting session
        session_data_str = self.get_encrypted(session_key)
        if session_data_str:
            try:
                session_data = json.loads(session_data_str)
                user_id = session_data.get('user_id')
                
                # Remove from user's active sessions
                if user_id:
                    user_sessions_key = f"user_sessions:{user_id}"
                    self.redis.srem(user_sessions_key, session_key)
                
            except Exception as e:
                print(f"Error during session cleanup: {e}")
        
        # Delete the session
        self.redis.delete(session_key)

class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def is_allowed(self, identifier: str, limit: int, window: int) -> bool:
        """Sliding window rate limiter"""
        current_time = int(time.time())
        window_start = current_time - window
        
        key = f"rate_limit:{identifier}"
        
        # Remove old entries
        self.redis.zremrangebyscore(key, 0, window_start)
        
        # Count current requests
        current_requests = self.redis.zcard(key)
        
        if current_requests >= limit:
            return False
        
        # Add current request
        self.redis.zadd(key, {str(current_time): current_time})
        self.redis.expire(key, window)
        
        return True
```

**🎯 Interview Insight**: *Security questions often cover data encryption, session management, and rate limiting. Discuss the balance between security and performance, and mention compliance requirements (GDPR, HIPAA) that might affect caching strategies.*

### 3. Operational Excellence

```python
class RedisOperationalExcellence:
    def __init__(self):
        self.redis = Redis(host='localhost', port=6379, db=0)
        self.backup_location = '/var/backups/redis'
        
    def automated_backup(self):
        """Automated backup with rotation"""
        import subprocess
        from datetime import datetime
        
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        backup_file = f"{self.backup_location}/redis_backup_{timestamp}.rdb"
        
        try:
            # Trigger background save
            self.redis.bgsave()
            
            # Wait for background save to complete
            while self.redis.lastsave() == self.redis.lastsave():
                time.sleep(1)
            
            # Copy RDB file
            subprocess.run([
                'cp', '/var/lib/redis/dump.rdb', backup_file
            ], check=True)
            
            # Compress backup
            subprocess.run([
                'gzip', backup_file
            ], check=True)
            
            # Cleanup old backups (keep last 7 days)
            self._cleanup_old_backups()
            
            print(f"Backup completed: {backup_file}.gz")
            
        except Exception as e:
            print(f"Backup failed: {e}")
            # Send alert to monitoring system
            self._send_alert("Redis backup failed", str(e))
    
    def _cleanup_old_backups(self):
        """Remove backups older than 7 days"""
        import os
        import glob
        from datetime import datetime, timedelta
        
        cutoff_date = datetime.now() - timedelta(days=7)
        pattern = f"{self.backup_location}/redis_backup_*.rdb.gz"
        
        for backup_file in glob.glob(pattern):
            file_time = datetime.fromtimestamp(os.path.getctime(backup_file))
            if file_time < cutoff_date:
                os.remove(backup_file)
                print(f"Removed old backup: {backup_file}")
    
    def capacity_planning_analysis(self) -> Dict:
        """Analyze Redis usage for capacity planning"""
        info = self.redis.info()
        
        # Memory analysis
        used_memory = info['used_memory']
        used_memory_peak = info['used_memory_peak']
        max_memory = info.get('maxmemory', 0)
        
        # Connection analysis
        connected_clients = info['connected_clients']
        
        # Key analysis
        total_keys = sum(info.get(f'db{i}', {}).get('keys', 0) for i in range(16))
        
        # Performance metrics
        ops_per_sec = info.get('instantaneous_ops_per_sec', 0)
        
        # Calculate trends (simplified - in production, use time series data)
        memory_growth_rate = self._calculate_memory_growth_rate()
        
        recommendations = []
        
        # Memory recommendations
        if max_memory > 0:
            memory_usage_pct = (used_memory / max_memory) * 100
            if memory_usage_pct > 80:
                recommendations.append("Memory usage is high - consider scaling up")
        
        # Connection recommendations
        if connected_clients > 1000:
            recommendations.append("High connection count - review connection pooling")
        
        # Performance recommendations
        if ops_per_sec > 100000:
            recommendations.append("High operation rate - consider read replicas")
        
        return {
            'memory': {
                'used_bytes': used_memory,
                'used_human': info['used_memory_human'],
                'peak_bytes': used_memory_peak,
                'peak_human': info['used_memory_peak_human'],
                'max_bytes': max_memory,
                'usage_percentage': (used_memory / max_memory * 100) if max_memory > 0 else 0,
                'growth_rate_mb_per_day': memory_growth_rate
            },
            'connections': {
                'current': connected_clients,
                'max_input': info.get('maxclients', 'unlimited')
            },
            'keys': {
                'total': total_keys,
                'expired': info.get('expired_keys', 0),
                'evicted': info.get('evicted_keys', 0)
            },
            'performance': {
                'ops_per_second': ops_per_sec,
                'keyspace_hits': info.get('keyspace_hits', 0),
                'keyspace_misses': info.get('keyspace_misses', 0),
                'hit_rate': self._calculate_hit_rate(info)
            },
            'recommendations': recommendations
        }
    
    def _calculate_memory_growth_rate(self) -> float:
        """Calculate memory growth rate (simplified)"""
        # In production, this would analyze historical data
        # For demo purposes, return a placeholder
        return 50.0