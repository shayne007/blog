---
title: 'MySQL Database Sharding: Complete Guide'
date: 2025-06-08 17:38:40
tags: [mysql]
categories: [mysql]
---

## Introduction

Database sharding is a horizontal scaling technique that distributes data across multiple database instances. As applications grow and face increasing data volumes and user loads, traditional vertical scaling (adding more CPU, RAM, or storage) becomes insufficient and cost-prohibitive. Sharding addresses this by partitioning data horizontally across multiple database servers, allowing for linear scalability and improved performance.

**Key Interview Question**: *"When would you consider implementing database sharding over other scaling solutions?"*

The decision to implement sharding typically occurs when:
- Single database performance degrades despite optimization
- Data volume exceeds single server capacity
- Read/write throughput requirements exceed single instance limits
- Geographic distribution of users requires localized data access
- Compliance requirements mandate data locality

## Understanding Database Sharding

### What is Sharding?

Sharding partitions a large database into smaller, more manageable pieces called "shards." Each shard contains a subset of the total data and operates as an independent database. The collection of shards together represents the complete dataset.

### Sharding vs. Other Scaling Techniques

**Vertical Scaling (Scale Up)**
- Increases hardware resources on a single server
- Limited by hardware constraints
- Single point of failure
- Eventually becomes cost-prohibitive

**Read Replicas**
- Multiple read-only copies of the master database
- Improves read performance but doesn't help with write scaling
- All writes still go to the master

**Sharding (Horizontal Scaling)**
- Distributes both reads and writes across multiple servers
- Theoretically unlimited scalability
- Eliminates single points of failure
- Introduces complexity in application logic

**Interview Insight**: Candidates should understand that sharding is typically the last resort due to its complexity. Always explore vertical scaling, read replicas, caching, and query optimization first.

## Sharding Strategies

### 1. Range-Based Sharding

Data is partitioned based on ranges of a specific column value, typically a primary key or timestamp.

```sql
-- Example: User data sharded by user ID ranges
-- Shard 1: user_id 1-10000
-- Shard 2: user_id 10001-20000
-- Shard 3: user_id 20001-30000

SELECT * FROM users WHERE user_id BETWEEN 10001 AND 20000;
-- Routes to Shard 2
```

**Advantages:**
- Simple to understand and implement
- Range queries are efficient
- Easy to add new shards for new ranges

**Disadvantages:**
- Potential for hotspots if data distribution is uneven
- Difficult to rebalance existing shards
- Sequential IDs can create write hotspots

### 2. Hash-Based Sharding

Data is distributed using a hash function applied to a sharding key.

```python
# Example hash-based sharding logic
def get_shard(user_id, num_shards):
    return hash(user_id) % num_shards

# user_id 12345 -> hash(12345) % 4 = shard_2
```

**Advantages:**
- Even data distribution
- No hotspots with good hash function
- Predictable shard routing

**Disadvantages:**
- Range queries require checking all shards
- Difficult to add/remove shards (resharding required)
- Hash function changes affect all data

### 3. Directory-Based Sharding

A lookup service maintains a mapping of sharding keys to specific shards.

```sql
-- Sharding directory table
CREATE TABLE shard_directory (
    shard_key VARCHAR(255) PRIMARY KEY,
    shard_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example mappings
INSERT INTO shard_directory VALUES 
('user_region_us_east', 1),
('user_region_us_west', 2),
('user_region_europe', 3);
```

**Advantages:**
- Flexible shard assignment
- Easy to rebalance and migrate data
- Supports complex sharding logic

**Disadvantages:**
- Additional lookup overhead
- Directory service becomes a potential bottleneck
- More complex to implement and maintain

### 4. Geographic Sharding

Data is partitioned based on geographic location, often for compliance or performance reasons.

```sql
-- Users table with geographic sharding
-- US Shard
CREATE TABLE users_us (
    user_id INT PRIMARY KEY,
    name VARCHAR(255),
    region ENUM('US') DEFAULT 'US'
);

-- EU Shard  
CREATE TABLE users_eu (
    user_id INT PRIMARY KEY,
    name VARCHAR(255),
    region ENUM('EU') DEFAULT 'EU'
);
```

**Interview Question**: *"How would you handle a user who moves from one geographic region to another in a geographically sharded system?"*

**Answer**: This requires careful planning including data migration procedures, temporary dual-write strategies during migration, and handling of cross-shard relationships. Consider implementing a migration workflow that can move user data between shards while maintaining data consistency.

## Implementation Approaches

### Application-Level Sharding

The application handles shard routing, query distribution, and result aggregation.

```python
class ShardManager:
    def __init__(self, shards):
        self.shards = shards
    
    def get_connection(self, shard_key):
        shard_id = self.calculate_shard(shard_key)
        return self.shards[shard_id].get_connection()
    
    def calculate_shard(self, key):
        return hash(key) % len(self.shards)
    
    def execute_query(self, shard_key, query):
        conn = self.get_connection(shard_key)
        return conn.execute(query)
    
    def execute_cross_shard_query(self, query):
        results = []
        for shard in self.shards:
            result = shard.execute(query)
            results.extend(result)
        return self.aggregate_results(results)
```

**Advantages:**
- Full control over sharding logic
- Can optimize for specific use cases
- No additional infrastructure components

**Disadvantages:**
- Increases application complexity
- Requires handling connection pooling per shard
- Cross-shard operations become complex

### Middleware/Proxy-Based Sharding

A middleware layer handles shard routing transparently to the application.

Popular solutions include:
- **ProxySQL**: MySQL-compatible proxy with sharding capabilities
- **Vitess**: Kubernetes-native MySQL sharding solution
- **MySQL Router**: Official MySQL proxy with limited sharding support

```yaml
# Example Vitess configuration
keyspaces:
  - name: user_data
    sharded: true
    vindexes:
      hash:
        type: hash
    tables:
      - name: users
        column_vindexes:
          - column: user_id
            name: hash
```

**Advantages:**
- Transparent to application
- Centralized shard management
- Built-in connection pooling and load balancing

**Disadvantages:**
- Additional infrastructure complexity
- Potential single point of failure
- Learning curve for specific tools

### Database-Level Sharding

Some databases provide built-in sharding capabilities.

**MySQL Cluster (NDB)**
- Automatic data distribution
- Built-in redundancy
- Different storage engine with limitations

**MySQL with Partitioning**
- Table-level partitioning within single instance
- Not true sharding but can help with some use cases

```sql
-- MySQL table partitioning example
CREATE TABLE users (
    user_id INT,
    name VARCHAR(255),
    created_at DATE
) PARTITION BY RANGE(user_id) (
    PARTITION p1 VALUES LESS THAN (10000),
    PARTITION p2 VALUES LESS THAN (20000),
    PARTITION p3 VALUES LESS THAN (30000)
);
```

## Best Practices

### Choosing the Right Sharding Key

The sharding key is crucial for system performance and maintainability.

**Characteristics of a Good Sharding Key:**
- High cardinality (many unique values)
- Even distribution of access patterns
- Rarely changes or never changes
- Present in most queries
- Allows for efficient routing

**Common Interview Question**: *"What would you use as a sharding key for a social media application?"*

**Answer**: User ID is often the best choice because:
- High cardinality (millions of users)
- Present in most queries (posts, likes, follows)
- Immutable once assigned
- Enables user-centric data locality

However, consider the trade-offs:
- Cross-user analytics become complex
- Friend relationships span shards
- Popular users might create hotspots

### Data Modeling for Sharded Systems

**Denormalization Strategy**
```sql
-- Instead of normalized tables across shards
-- Users table (shard by user_id)
-- Posts table (shard by user_id) 
-- Comments table (shard by user_id)

-- Consider denormalized approach
CREATE TABLE user_timeline (
    user_id INT,
    post_id INT,
    post_content TEXT,
    post_timestamp TIMESTAMP,
    comment_count INT,
    like_count INT,
    -- Denormalized data for efficient queries
    author_name VARCHAR(255),
    author_avatar_url VARCHAR(500)
);
```

**Avoiding Cross-Shard Joins**
- Denormalize frequently joined data
- Use application-level joins when necessary
- Consider data duplication for read performance
- Implement eventual consistency patterns

### Connection Management

```python
class ShardConnectionPool:
    def __init__(self, shard_configs):
        self.pools = {}
        for shard_id, config in shard_configs.items():
            self.pools[shard_id] = mysql.connector.pooling.MySQLConnectionPool(
                pool_name=f"shard_{shard_id}",
                pool_size=config['pool_size'],
                **config['connection_params']
            )
    
    def get_connection(self, shard_id):
        return self.pools[shard_id].get_connection()
```

**Best Practices:**
- Maintain separate connection pools per shard
- Monitor pool utilization and adjust sizes
- Implement circuit breakers for failed shards
- Use connection health checks

### Transaction Management

**Single-Shard Transactions**
```python
def transfer_within_shard(shard_key, from_account, to_account, amount):
    conn = get_shard_connection(shard_key)
    try:
        conn.begin()
        # Debit from_account
        conn.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", 
                    (amount, from_account))
        # Credit to_account  
        conn.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", 
                    (amount, to_account))
        conn.commit()
    except Exception as e:
        conn.rollback()
        raise e
```

**Cross-Shard Transactions**
Implement distributed transaction patterns like Two-Phase Commit (2PC) or Saga pattern:

```python
def transfer_cross_shard(from_shard_key, to_shard_key, from_account, to_account, amount):
    # Saga pattern implementation
    steps = [
        ("debit", from_shard_key, from_account, amount),
        ("credit", to_shard_key, to_account, amount)
    ]
    
    completed_steps = []
    try:
        for step_type, shard_key, account, amt in steps:
            execute_step(step_type, shard_key, account, amt)
            completed_steps.append((step_type, shard_key, account, amt))
    except Exception as e:
        # Compensate completed steps
        for step in reversed(completed_steps):
            compensate_step(step)
        raise e
```

## Challenges and Solutions

### Cross-Shard Queries

**Challenge**: Aggregating data across multiple shards efficiently.

**Solutions:**
1. **Application-Level Aggregation**
```python
def get_user_stats_across_shards(user_id_list):
    shard_queries = defaultdict(list)
    
    # Group users by shard
    for user_id in user_id_list:
        shard_id = calculate_shard(user_id)
        shard_queries[shard_id].append(user_id)
    
    # Execute parallel queries
    results = []
    with ThreadPoolExecutor() as executor:
        futures = []
        for shard_id, user_ids in shard_queries.items():
            future = executor.submit(query_shard_users, shard_id, user_ids)
            futures.append(future)
        
        for future in futures:
            results.extend(future.result())
    
    return aggregate_user_stats(results)
```

2. **Materialized Views/ETL**
- Pre-aggregate data in separate analytical databases
- Use ETL processes to combine shard data
- Implement near real-time data pipelines

### Rebalancing and Resharding

**Challenge**: Adding new shards or rebalancing existing ones without downtime.

**Solutions:**

1. **Consistent Hashing**
```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, nodes=None, replicas=150):
        self.replicas = replicas
        self.ring = {}
        self.sorted_keys = []
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def add_node(self, node):
        for i in range(self.replicas):
            key = self.hash(f"{node}:{i}")
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
    
    def get_node(self, key):
        if not self.ring:
            return None
        
        hash_key = self.hash(key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        if idx == len(self.sorted_keys):
            idx = 0
        return self.ring[self.sorted_keys[idx]]
```

2. **Live Migration Strategy**
```python
def migrate_shard_data(source_shard, target_shard, migration_key_range):
    # 1. Start dual-write to both shards
    enable_dual_write(source_shard, target_shard, migration_key_range)
    
    # 2. Copy existing data
    copy_data_batch(source_shard, target_shard, migration_key_range)
    
    # 3. Verify data consistency
    verify_data_consistency(source_shard, target_shard, migration_key_range)
    
    # 4. Switch reads to target shard
    switch_reads(target_shard, migration_key_range)
    
    # 5. Stop dual-write, switch writes to target
    switch_writes(target_shard, migration_key_range)
    
    # 6. Clean up source shard data
    cleanup_source_data(source_shard, migration_key_range)
```

### Hotspots and Load Balancing

**Interview Question**: *"How would you handle a situation where one shard is receiving significantly more traffic than others?"*

**Solutions:**

1. **Hotspot Detection**
```python
class HotspotMonitor:
    def __init__(self):
        self.shard_metrics = defaultdict(lambda: {
            'queries_per_second': 0,
            'cpu_usage': 0,
            'connection_count': 0
        })
    
    def detect_hotspots(self, threshold_multiplier=2.0):
        avg_qps = sum(m['queries_per_second'] for m in self.shard_metrics.values()) / len(self.shard_metrics)
        
        hotspots = []
        for shard_id, metrics in self.shard_metrics.items():
            if metrics['queries_per_second'] > avg_qps * threshold_multiplier:
                hotspots.append(shard_id)
        
        return hotspots
```

2. **Load Balancing Strategies**
- **Split hot shards**: Divide heavily loaded shard ranges
- **Read replicas**: Add read replicas for hot shards
- **Caching**: Implement application-level caching for hot data
- **Request throttling**: Rate limit requests to hot shards

## Performance Considerations

### Query Optimization for Sharded Systems

**Efficient Query Patterns:**
```sql
-- Good: Single shard query with shard key
SELECT * FROM users WHERE user_id = 12345;

-- Good: Single shard range query
SELECT * FROM posts WHERE user_id = 12345 AND created_at > '2023-01-01';

-- Avoid: Cross-shard queries without shard key
SELECT COUNT(*) FROM users WHERE age > 25;

-- Better: Use application-level aggregation
-- Query each shard separately and combine results
```

**Indexing Strategy:**
```sql
-- Ensure shard key is part of compound indexes
CREATE INDEX idx_user_posts ON posts(user_id, created_at, post_type);

-- Include shard key in all WHERE clauses
SELECT * FROM posts 
WHERE user_id = 12345 -- Shard key
  AND post_type = 'public' 
  AND created_at > '2023-01-01';
```

### Caching Strategies

**Multi-Level Caching:**
```python
class ShardedCache:
    def __init__(self):
        self.l1_cache = {}  # Application memory cache
        self.l2_cache = redis.Redis()  # Shared Redis cache
    
    def get(self, key):
        # Try L1 cache first
        if key in self.l1_cache:
            return self.l1_cache[key]
        
        # Try L2 cache
        value = self.l2_cache.get(key)
        if value:
            self.l1_cache[key] = value
            return value
        
        # Fallback to database
        shard_key = extract_shard_key(key)
        value = query_shard(shard_key, key)
        
        # Cache the result
        self.l2_cache.setex(key, 3600, value)
        self.l1_cache[key] = value
        
        return value
```

## Monitoring and Maintenance

### Key Metrics to Monitor

**Per-Shard Metrics:**
- Query response time (P50, P95, P99)
- Queries per second
- Connection pool utilization
- Disk I/O and CPU usage
- Error rates and timeouts

**Cross-Shard Metrics:**
- Query distribution across shards
- Cross-shard query frequency
- Data migration progress
- Replication lag (if using replicas)

**Monitoring Implementation:**
```python
class ShardMonitor:
    def __init__(self):
        self.metrics_collector = MetricsCollector()
    
    def collect_shard_metrics(self):
        for shard_id in self.shards:
            metrics = {
                'shard_id': shard_id,
                'timestamp': time.time(),
                'active_connections': self.get_active_connections(shard_id),
                'queries_per_second': self.get_qps(shard_id),
                'avg_response_time': self.get_avg_response_time(shard_id),
                'error_rate': self.get_error_rate(shard_id)
            }
            self.metrics_collector.send(metrics)
    
    def check_shard_health(self):
        unhealthy_shards = []
        for shard_id in self.shards:
            try:
                conn = self.get_connection(shard_id)
                conn.execute("SELECT 1")
            except Exception as e:
                unhealthy_shards.append((shard_id, str(e)))
        return unhealthy_shards
```

### Backup and Recovery

**Shard-Level Backups:**
```bash
#!/bin/bash
# Backup script for individual shards

SHARD_ID=$1
BACKUP_DIR="/backups/shard_${SHARD_ID}"
DATE=$(date +%Y%m%d_%H%M%S)

# Create consistent backup
mysqldump --single-transaction \
          --routines \
          --triggers \
          --host=${SHARD_HOST} \
          --user=${SHARD_USER} \
          --password=${SHARD_PASS} \
          ${SHARD_DATABASE} > ${BACKUP_DIR}/backup_${DATE}.sql

# Compress backup
gzip ${BACKUP_DIR}/backup_${DATE}.sql

# Upload to cloud storage
aws s3 cp ${BACKUP_DIR}/backup_${DATE}.sql.gz \
          s3://db-backups/shard_${SHARD_ID}/
```

**Point-in-Time Recovery:**
```python
def restore_shard_to_point_in_time(shard_id, target_timestamp):
    # 1. Find appropriate backup before target time
    backup_file = find_backup_before_timestamp(shard_id, target_timestamp)
    
    # 2. Restore from backup
    restore_from_backup(shard_id, backup_file)
    
    # 3. Apply binary logs up to target timestamp
    apply_binary_logs(shard_id, backup_file.timestamp, target_timestamp)
    
    # 4. Verify data integrity
    verify_shard_integrity(shard_id)
```

## Real-World Examples

### E-commerce Platform Sharding

**Scenario**: An e-commerce platform with millions of users and orders.

**Sharding Strategy:**
```sql
-- Shard by user_id for user-centric data
-- Shard 1: user_id % 4 = 0
-- Shard 2: user_id % 4 = 1  
-- Shard 3: user_id % 4 = 2
-- Shard 4: user_id % 4 = 3

-- Users table (sharded by user_id)
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255),
    created_at TIMESTAMP
);

-- Orders table (sharded by user_id for co-location)
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,  -- Shard key
    total_amount DECIMAL(10,2),
    status ENUM('pending', 'completed', 'cancelled'),
    created_at TIMESTAMP,
    INDEX idx_user_orders (user_id, created_at)
);

-- Order items (sharded by user_id via order relationship)
CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2)
);
```

**Challenges Addressed:**
- Product catalog remains unsharded (reference data)
- Order analytics aggregated via ETL processes
- Cross-user features (recommendations) use separate service

### Social Media Platform Sharding

**Scenario**: Social media platform with user feeds, posts, and relationships.

**Multi-Dimensional Sharding:**
```python
class SocialMediaSharding:
    def __init__(self):
        self.user_shards = 8  # User data sharded by user_id
        self.timeline_shards = 16  # Timeline data sharded by user_id
        self.content_shards = 4  # Content sharded by content_id
    
    def get_user_shard(self, user_id):
        return f"user_shard_{user_id % self.user_shards}"
    
    def get_timeline_shard(self, user_id):
        return f"timeline_shard_{user_id % self.timeline_shards}"
    
    def get_content_shard(self, content_id):
        return f"content_shard_{content_id % self.content_shards}"
```

**Feed Generation Strategy:**
```python
def generate_user_feed(user_id):
    # 1. Get user's following list (from user shard)
    following_list = get_user_following(user_id)
    
    # 2. Fetch recent posts from followed users (distributed query)
    recent_posts = []
    for followed_user_id in following_list:
        content_shard = get_content_shard_for_user(followed_user_id)
        posts = fetch_recent_posts(content_shard, followed_user_id, limit=10)
        recent_posts.extend(posts)
    
    # 3. Rank and personalize feed
    ranked_feed = rank_posts(recent_posts, user_id)
    
    # 4. Cache generated feed
    cache_user_feed(user_id, ranked_feed)
    
    return ranked_feed
```

## Interview Insights

### Common Interview Questions and Answers

**Q: "How do you handle database schema changes in a sharded environment?"**

**A:** Schema changes in sharded systems require careful planning:

1. **Backward-compatible changes first**: Add new columns with default values, create new indexes
2. **Rolling deployment**: Apply changes to one shard at a time to minimize downtime
3. **Application compatibility**: Ensure application can handle both old and new schemas during transition
4. **Automated tooling**: Use migration tools that can apply changes across all shards consistently

```python
def deploy_schema_change(migration_script):
    for shard_id in get_all_shards():
        try:
            # Apply migration to shard
            apply_migration(shard_id, migration_script)
            
            # Verify migration success
            verify_schema(shard_id)
            
            # Update deployment status
            mark_shard_migrated(shard_id)
            
        except Exception as e:
            # Rollback and alert
            rollback_migration(shard_id)
            alert_migration_failure(shard_id, e)
            break
```

**Q: "What are the trade-offs between different sharding strategies?"**

**A:** Each strategy has specific trade-offs:

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| Range-based | Simple, efficient range queries | Hotspots, hard to rebalance | Time-series data, sequential access |
| Hash-based | Even distribution, no hotspots | No range queries, resharding complex | User data, even access patterns |
| Directory-based | Flexible, easy rebalancing | Lookup overhead, complexity | Dynamic requirements, frequent rebalancing |
| Geographic | Compliance, latency optimization | Cross-region complexity | Global applications, data locality requirements |

**Q: "How would you test a sharded database system?"**

**A:** Comprehensive testing strategy includes:

1. **Unit Testing**: Test shard routing logic, connection management
2. **Integration Testing**: Test cross-shard operations, transaction handling
3. **Load Testing**: Simulate realistic traffic patterns across shards
4. **Failure Testing**: Test behavior with shard failures, network partitions
5. **Migration Testing**: Test resharding and rebalancing procedures

```python
class ShardTestSuite:
    def test_shard_routing(self):
        # Test that queries route to correct shards
        for user_id in range(1000):
            expected_shard = calculate_expected_shard(user_id)
            actual_shard = shard_router.get_shard(user_id)
            assert expected_shard == actual_shard
    
    def test_cross_shard_transaction(self):
        # Test distributed transaction handling
        result = transfer_between_shards(
            from_shard=1, to_shard=2, 
            amount=100, user1=123, user2=456
        )
        assert result.success
        assert verify_balance_consistency()
    
    def test_shard_failure_handling(self):
        # Simulate shard failure and test fallback
        with mock_shard_failure(shard_id=2):
            response = query_with_fallback(user_id=456)
            assert response.from_replica or response.cached
```

**Q: "When would you not recommend sharding?"**

**A:** Avoid sharding when:

- Current database size is manageable (< 100GB)
- Query patterns don't align with sharding keys
- Application heavily relies on complex joins and transactions
- Team lacks expertise in distributed systems
- Alternative solutions (caching, read replicas, optimization) haven't been fully explored

**Red flags for sharding:**
- Premature optimization without clear bottlenecks
- Complex reporting requirements across all data
- Strong consistency requirements for all operations
- Limited operational resources for maintaining distributed system

### Technical Deep-Dive Questions

**Q: "Explain how you would implement consistent hashing for shard rebalancing."**

**A:** Consistent hashing minimizes data movement during resharding:

```python
class ConsistentHashingShardManager:
    def __init__(self, initial_shards, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        
        for shard in initial_shards:
            self.add_shard(shard)
    
    def hash_function(self, key):
        return int(hashlib.md5(str(key).encode()).hexdigest(), 16)
    
    def add_shard(self, shard_id):
        # Add multiple virtual nodes for each physical shard
        for i in range(self.virtual_nodes):
            virtual_key = f"{shard_id}:{i}"
            hash_key = self.hash_function(virtual_key)
            self.ring[hash_key] = shard_id
            bisect.insort(self.sorted_keys, hash_key)
    
    def remove_shard(self, shard_id):
        # Remove all virtual nodes for this shard
        keys_to_remove = []
        for hash_key, shard in self.ring.items():
            if shard == shard_id:
                keys_to_remove.append(hash_key)
        
        for key in keys_to_remove:
            del self.ring[key]
            self.sorted_keys.remove(key)
    
    def get_shard(self, data_key):
        if not self.ring:
            return None
        
        hash_key = self.hash_function(data_key)
        idx = bisect.bisect_right(self.sorted_keys, hash_key)
        if idx == len(self.sorted_keys):
            idx = 0
        return self.ring[self.sorted_keys[idx]]
    
    def get_affected_keys_for_new_shard(self, new_shard_id):
        # Determine which keys need to be moved to new shard
        old_ring = self.ring.copy()
        old_sorted_keys = self.sorted_keys.copy()
        
        self.add_shard(new_shard_id)
        
        affected_keys = []
        # Sample key space to find affected ranges
        for sample_key in range(0, 2**32, 1000):  # Sample every 1000
            old_shard = self._get_shard_from_ring(sample_key, old_ring, old_sorted_keys)
            new_shard = self.get_shard(sample_key)
            
            if old_shard != new_shard and new_shard == new_shard_id:
                affected_keys.append(sample_key)
        
        return affected_keys
```

**Q: "How do you handle foreign key relationships in a sharded environment?"**

**A:** Foreign key relationships require special handling in sharded systems:

1. **Co-location Strategy**: Keep related data in the same shard
```sql
-- Both users and orders use user_id as shard key
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(255)
) SHARD BY user_id;

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,  -- Foreign key, same shard key
    total_amount DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
) SHARD BY user_id;
```

2. **Denormalization Approach**: Duplicate reference data
```sql
-- Instead of foreign key to products table
CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    -- Denormalized product data
    product_name VARCHAR(255),
    product_price DECIMAL(10,2),
    user_id INT  -- Shard key
) SHARD BY user_id;
```

3. **Application-Level Referential Integrity**
```python
class ShardedReferentialIntegrity:
    def create_order_with_items(self, user_id, order_data, items_data):
        # Validate references before creating
        self.validate_user_exists(user_id)
        self.validate_products_exist([item['product_id'] for item in items_data])
        
        shard = self.get_shard(user_id)
        try:
            shard.begin_transaction()
            
            # Create order
            order_id = shard.insert_order(order_data)
            
            # Create order items with denormalized product data
            for item in items_data:
                product_info = self.get_product_info(item['product_id'])
                item_data = {
                    **item,
                    'order_id': order_id,
                    'product_name': product_info['name'],
                    'product_price': product_info['price']
                }
                shard.insert_order_item(item_data)
            
            shard.commit()
            return order_id
            
        except Exception as e:
            shard.rollback()
            raise e
```

**Q: "Describe your approach to handling eventual consistency in a sharded system."**

**A:** Eventual consistency management requires multiple strategies:

1. **Event-Driven Architecture**
```python
class EventDrivenConsistency:
    def __init__(self):
        self.event_bus = EventBus()
        self.event_handlers = {}
    
    def publish_user_update(self, user_id, updated_fields):
        event = {
            'event_type': 'user_updated',
            'user_id': user_id,
            'fields': updated_fields,
            'timestamp': time.time(),
            'event_id': uuid.uuid4()
        }
        self.event_bus.publish('user_events', event)
    
    def handle_user_update(self, event):
        # Update denormalized user data across relevant shards
        affected_shards = self.find_shards_with_user_data(event['user_id'])
        
        for shard_id in affected_shards:
            try:
                self.update_denormalized_user_data(shard_id, event)
                self.mark_event_processed(shard_id, event['event_id'])
            except Exception as e:
                # Retry mechanism for failed updates
                self.schedule_retry(shard_id, event, delay=60)
```

2. **Read-After-Write Consistency**
```python
class ReadAfterWriteConsistency:
    def __init__(self):
        self.write_cache = {}  # Track recent writes per user
        self.cache_ttl = 300   # 5 minutes
    
    def write_user_data(self, user_id, data):
        shard = self.get_shard(user_id)
        result = shard.update_user(user_id, data)
        
        # Cache the write for read consistency
        self.write_cache[user_id] = {
            'data': data,
            'timestamp': time.time(),
            'version': result.version
        }
        
        return result
    
    def read_user_data(self, user_id):
        # Check if we have recent write data
        if user_id in self.write_cache:
            cache_entry = self.write_cache[user_id]
            if time.time() - cache_entry['timestamp'] < self.cache_ttl:
                return cache_entry['data']
        
        # Read from appropriate shard
        shard = self.get_shard(user_id)
        return shard.get_user(user_id)
```

3. **Saga Pattern for Distributed Transactions**
```python
class SagaOrchestrator:
    def __init__(self):
        self.saga_store = SagaStateStore()
    
    def execute_cross_shard_operation(self, saga_id, steps):
        saga_state = {
            'saga_id': saga_id,
            'steps': steps,
            'completed_steps': [],
            'status': 'running'
        }
        
        self.saga_store.save_saga_state(saga_state)
        
        try:
            for step_index, step in enumerate(steps):
                self.execute_step(saga_id, step_index, step)
                saga_state['completed_steps'].append(step_index)
                self.saga_store.update_saga_state(saga_state)
            
            saga_state['status'] = 'completed'
            self.saga_store.update_saga_state(saga_state)
            
        except Exception as e:
            # Compensate completed steps
            self.compensate_saga(saga_id, saga_state['completed_steps'])
            saga_state['status'] = 'compensated'
            self.saga_store.update_saga_state(saga_state)
            raise e
    
    def compensate_saga(self, saga_id, completed_steps):
        for step_index in reversed(completed_steps):
            try:
                self.execute_compensation(saga_id, step_index)
            except Exception as e:
                # Log compensation failure - may need manual intervention
                self.log_compensation_failure(saga_id, step_index, e)
```

### Advanced Sharding Patterns

**Q: "How would you implement multi-tenant sharding where each tenant's data needs to be isolated?"**

**A:** Multi-tenant sharding requires additional isolation considerations:

```python
class MultiTenantShardManager:
    def __init__(self):
        self.tenant_shard_mapping = {}
        self.shard_tenant_mapping = defaultdict(set)
    
    def assign_tenant_to_shard(self, tenant_id, shard_preference=None):
        if shard_preference and self.has_capacity(shard_preference):
            assigned_shard = shard_preference
        else:
            assigned_shard = self.find_optimal_shard(tenant_id)
        
        self.tenant_shard_mapping[tenant_id] = assigned_shard
        self.shard_tenant_mapping[assigned_shard].add(tenant_id)
        
        # Create tenant-specific database/schema
        self.create_tenant_schema(assigned_shard, tenant_id)
        
        return assigned_shard
    
    def get_tenant_connection(self, tenant_id):
        shard_id = self.tenant_shard_mapping.get(tenant_id)
        if not shard_id:
            raise TenantNotFoundError(f"Tenant {tenant_id} not assigned to any shard")
        
        # Return connection with tenant context
        conn = self.get_shard_connection(shard_id)
        conn.execute(f"USE tenant_{tenant_id}_db")
        return conn
    
    def migrate_tenant(self, tenant_id, target_shard):
        source_shard = self.tenant_shard_mapping[tenant_id]
        
        # 1. Create tenant schema on target shard
        self.create_tenant_schema(target_shard, tenant_id)
        
        # 2. Copy tenant data
        self.copy_tenant_data(source_shard, target_shard, tenant_id)
        
        # 3. Enable dual-write mode
        self.enable_dual_write(tenant_id, source_shard, target_shard)
        
        # 4. Switch reads to target shard
        self.tenant_shard_mapping[tenant_id] = target_shard
        
        # 5. Verify consistency and cleanup
        if self.verify_tenant_data_consistency(tenant_id, source_shard, target_shard):
            self.cleanup_tenant_data(source_shard, tenant_id)
            self.shard_tenant_mapping[source_shard].remove(tenant_id)
            self.shard_tenant_mapping[target_shard].add(tenant_id)
```

**Multi-tenant Schema Patterns:**

1. **Schema-per-Tenant**
```sql
-- Each tenant gets their own database schema
CREATE DATABASE tenant_123_db;
USE tenant_123_db;

CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    total_amount DECIMAL(10,2)
);
```

2. **Shared Schema with Tenant ID**
```sql
-- Shared tables with tenant_id column
CREATE TABLE users (
    tenant_id INT,
    user_id INT,
    name VARCHAR(255),
    email VARCHAR(255),
    PRIMARY KEY (tenant_id, user_id),
    INDEX idx_tenant_users (tenant_id)
);

-- Row-level security policies
CREATE VIEW tenant_users AS
SELECT user_id, name, email
FROM users
WHERE tenant_id = GET_CURRENT_TENANT_ID();
```

### Performance Optimization Strategies

**Q: "How do you optimize query performance across shards?"**

**A:** Multi-faceted approach to query optimization:

1. **Query Routing Optimization**
```python
class QueryOptimizer:
    def __init__(self):
        self.query_stats = QueryStatistics()
        self.shard_metadata = ShardMetadata()
    
    def optimize_query_plan(self, query, params):
        # Analyze query to determine optimal execution strategy
        query_analysis = self.analyze_query(query)
        
        if query_analysis.is_single_shard_query():
            return self.execute_single_shard(query, params)
        elif query_analysis.can_be_parallelized():
            return self.execute_parallel_query(query, params)
        else:
            return self.execute_sequential_query(query, params)
    
    def execute_parallel_query(self, query, params):
        # Execute query on multiple shards concurrently
        with ThreadPoolExecutor(max_workers=len(self.shards)) as executor:
            futures = []
            for shard_id in self.get_relevant_shards(query):
                future = executor.submit(self.execute_on_shard, shard_id, query, params)
                futures.append((shard_id, future))
            
            results = []
            for shard_id, future in futures:
                try:
                    result = future.result(timeout=30)  # 30 second timeout
                    results.append((shard_id, result))
                except TimeoutError:
                    self.log_slow_shard_query(shard_id, query)
                    # Continue without this shard's results
                    continue
            
            return self.merge_shard_results(results)
```

2. **Intelligent Caching**
```python
class ShardedCacheManager:
    def __init__(self):
        self.local_cache = {}
        self.distributed_cache = RedisCluster()
        self.cache_stats = CacheStatistics()
    
    def get_with_cache(self, cache_key, query_func, ttl=3600):
        # L1: Local cache
        if cache_key in self.local_cache:
            self.cache_stats.record_hit('local')
            return self.local_cache[cache_key]
        
        # L2: Distributed cache
        cached_value = self.distributed_cache.get(cache_key)
        if cached_value:
            self.cache_stats.record_hit('distributed')
            self.local_cache[cache_key] = cached_value
            return cached_value
        
        # L3: Database query
        self.cache_stats.record_miss()
        value = query_func()
        
        # Cache the result
        self.distributed_cache.setex(cache_key, ttl, value)
        self.local_cache[cache_key] = value
        
        return value
    
    def invalidate_pattern(self, pattern):
        # Invalidate cache entries matching pattern
        keys_to_delete = self.distributed_cache.keys(pattern)
        if keys_to_delete:
            self.distributed_cache.delete(*keys_to_delete)
        
        # Clear local cache entries
        local_keys_to_delete = [k for k in self.local_cache.keys() if fnmatch.fnmatch(k, pattern)]
        for key in local_keys_to_delete:
            del self.local_cache[key]
```

3. **Connection Pool Optimization**
```python
class OptimizedConnectionPool:
    def __init__(self, shard_configs):
        self.pools = {}
        self.pool_stats = defaultdict(lambda: {'active': 0, 'idle': 0, 'wait_time': 0})
        
        for shard_id, config in shard_configs.items():
            self.pools[shard_id] = self.create_optimized_pool(shard_id, config)
    
    def create_optimized_pool(self, shard_id, config):
        # Dynamic pool sizing based on shard load
        base_size = config.get('base_pool_size', 10)
        max_size = config.get('max_pool_size', 50)
        
        # Adjust pool size based on historical usage
        avg_concurrent_queries = self.get_avg_concurrent_queries(shard_id)
        optimal_size = min(max_size, max(base_size, int(avg_concurrent_queries * 1.2)))
        
        return mysql.connector.pooling.MySQLConnectionPool(
            pool_name=f"optimized_shard_{shard_id}",
            pool_size=optimal_size,
            pool_reset_session=True,
            autocommit=True,
            **config['connection_params']
        )
    
    def get_connection_with_monitoring(self, shard_id):
        start_time = time.time()
        
        try:
            conn = self.pools[shard_id].get_connection()
            wait_time = time.time() - start_time
            
            self.pool_stats[shard_id]['wait_time'] += wait_time
            self.pool_stats[shard_id]['active'] += 1
            
            return ConnectionWrapper(conn, shard_id, self.pool_stats)
            
        except mysql.connector.pooling.PoolError as e:
            # Pool exhausted - consider scaling up
            self.alert_pool_exhaustion(shard_id)
            raise e
```

### Disaster Recovery and High Availability

**Q: "How do you design disaster recovery for a sharded MySQL environment?"**

**A:** Comprehensive disaster recovery strategy:

1. **Multi-Region Shard Replication**
```python
class DisasterRecoveryManager:
    def __init__(self):
        self.primary_region = "us-east-1"
        self.backup_regions = ["us-west-2", "eu-west-1"]
        self.replication_lag_threshold = 5  # seconds
    
    def setup_cross_region_replication(self, shard_id):
        primary_shard = self.get_shard(self.primary_region, shard_id)
        
        for backup_region in self.backup_regions:
            backup_shard = self.get_shard(backup_region, shard_id)
            
            # Configure MySQL replication
            self.configure_replication(
                master=primary_shard,
                slave=backup_shard,
                replication_mode='GTID'
            )
            
            # Monitor replication health
            self.monitor_replication_lag(primary_shard, backup_shard)
    
    def failover_to_backup_region(self, failed_region, backup_region):
        affected_shards = self.get_shards_in_region(failed_region)
        
        for shard_id in affected_shards:
            try:
                # Promote backup shard to primary
                backup_shard = self.get_shard(backup_region, shard_id)
                self.promote_to_primary(backup_shard)
                
                # Update shard routing
                self.update_shard_routing(shard_id, backup_region)
                
                # Notify applications of failover
                self.notify_failover(shard_id, failed_region, backup_region)
                
            except Exception as e:
                self.log_failover_error(shard_id, e)
                # Continue with other shards
```

2. **Automated Backup Strategy**
```python
class ShardBackupManager:
    def __init__(self):
        self.backup_schedule = BackupScheduler()
        self.storage_backends = {
            'local': LocalStorage('/backups'),
            's3': S3Storage('db-backups-bucket'),
            'gcs': GCSStorage('db-backups-bucket')
        }
    
    def create_consistent_backup(self, shard_id):
        shard = self.get_shard(shard_id)
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        
        # Create consistent point-in-time backup
        backup_info = {
            'shard_id': shard_id,
            'timestamp': timestamp,
            'gtid_position': shard.get_gtid_position(),
            'binlog_position': shard.get_binlog_position()
        }
        
        # Physical backup using Percona XtraBackup
        backup_path = f"/tmp/backup_{shard_id}_{timestamp}"
        self.execute_xtrabackup(shard, backup_path)
        
        # Upload to multiple storage backends
        for storage_name, storage in self.storage_backends.items():
            try:
                storage.upload(backup_path, f"shard_{shard_id}/{timestamp}")
                backup_info[f'{storage_name}_uploaded'] = True
            except Exception as e:
                self.log_backup_upload_error(storage_name, shard_id, e)
                backup_info[f'{storage_name}_uploaded'] = False
        
        # Store backup metadata
        self.store_backup_metadata(backup_info)
        
        return backup_info
    
    def restore_from_backup(self, shard_id, target_timestamp, target_shard=None):
        # Find appropriate backup
        backup_info = self.find_backup_before_timestamp(shard_id, target_timestamp)
        
        if not backup_info:
            raise BackupNotFoundError(f"No backup found for shard {shard_id} before {target_timestamp}")
        
        target_shard = target_shard or self.get_shard(shard_id)
        
        # Download and restore backup
        backup_path = self.download_backup(backup_info)
        self.restore_xtrabackup(target_shard, backup_path)
        
        # Apply point-in-time recovery if needed
        if target_timestamp > backup_info['timestamp']:
            self.apply_binlog_recovery(
                target_shard, 
                backup_info['binlog_position'],
                target_timestamp
            )
        
        return True
```

### Security Considerations

**Q: "What security measures should be implemented in a sharded MySQL environment?"**

**A:** Multi-layered security approach:

1. **Network Security**
```python
class ShardSecurityManager:
    def __init__(self):
        self.vpc_config = VPCConfiguration()
        self.firewall_rules = FirewallManager()
        self.encryption_manager = EncryptionManager()
    
    def setup_network_security(self):
        # VPC configuration for shard isolation
        for region in self.regions:
            vpc = self.vpc_config.create_vpc(
                region=region,
                cidr_block="10.0.0.0/16",
                enable_dns_hostnames=True
            )
            
            # Private subnets for database shards
            for az_index, availability_zone in enumerate(self.get_availability_zones(region)):
                subnet = self.vpc_config.create_private_subnet(
                    vpc=vpc,
                    cidr_block=f"10.0.{az_index + 1}.0/24",
                    availability_zone=availability_zone
                )
                
                # Security groups for shard access
                self.create_shard_security_group(vpc, subnet)
    
    def create_shard_security_group(self, vpc, subnet):
        security_group = self.firewall_rules.create_security_group(
            name=f"shard-sg-{subnet.id}",
            vpc=vpc,
            rules=[
                # MySQL port access only from application tier
                {
                    'protocol': 'tcp',
                    'port': 3306,
                    'source': 'application-sg',
                    'description': 'MySQL access from application servers'
                },
                # Replication port for cross-region replication
                {
                    'protocol': 'tcp', 
                    'port': 3307,
                    'source': 'replication-sg',
                    'description': 'MySQL replication traffic'
                },
                # Monitoring access
                {
                    'protocol': 'tcp',
                    'port': 9104,
                    'source': 'monitoring-sg', 
                    'description': 'MySQL exporter for monitoring'
                }
            ]
        )
        return security_group
```

2. **Authentication and Authorization**
```sql
-- Shard-specific user management
-- Create dedicated users for different access patterns

-- Application read-write user
CREATE USER 'app_rw'@'%' IDENTIFIED BY 'secure_password_123';
GRANT SELECT, INSERT, UPDATE, DELETE ON shard_db.* TO 'app_rw'@'%';

-- Application read-only user  
CREATE USER 'app_ro'@'%' IDENTIFIED BY 'secure_password_456';
GRANT SELECT ON shard_db.* TO 'app_ro'@'%';

-- Replication user
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'replication_password_789';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- Monitoring user
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor_password_abc';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'monitor'@'%';

-- Backup user
CREATE USER 'backup'@'localhost' IDENTIFIED BY 'backup_password_def';
GRANT SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER ON *.* TO 'backup'@'localhost';
```

3. **Encryption Implementation**
```python
class ShardEncryptionManager:
    def __init__(self):
        self.kms_client = KMSClient()
        self.encryption_keys = {}
    
    def setup_shard_encryption(self, shard_id):
        # Generate shard-specific encryption key
        key_id = self.kms_client.create_key(
            description=f"Encryption key for shard {shard_id}",
            key_usage='ENCRYPT_DECRYPT'
        )
        
        self.encryption_keys[shard_id] = key_id
        
        # Configure MySQL encryption at rest
        shard = self.get_shard(shard_id)
        shard.execute(f"""
            SET GLOBAL default_table_encryption = ON;
            SET GLOBAL table_encryption_privilege_check = ON;
        """)
        
        # Configure binlog encryption
        shard.execute(f"""
            SET GLOBAL binlog_encryption = ON;
            SET GLOBAL binlog_rotate_encryption_master_key_at_startup = ON;
        """)
        
        return key_id
    
    def encrypt_sensitive_data(self, shard_id, data):
        key_id = self.encryption_keys[shard_id]
        return self.kms_client.encrypt(key_id, data)
    
    def decrypt_sensitive_data(self, shard_id, encrypted_data):
        key_id = self.encryption_keys[shard_id]
        return self.kms_client.decrypt(key_id, encrypted_data)
```

## Conclusion

Database sharding is a powerful scaling technique that enables applications to handle massive datasets and high-throughput workloads. However, it introduces significant complexity that must be carefully managed through proper planning, implementation, and operational practices.

### Key Takeaways

**When to Consider Sharding:**
- Single database performance becomes a bottleneck despite optimization
- Data volume exceeds single server capacity
- Geographic distribution requirements
- Compliance and data locality needs

**Success Factors:**
- Choose the right sharding strategy for your access patterns
- Implement comprehensive monitoring and alerting
- Plan for failure scenarios and disaster recovery
- Maintain operational expertise in distributed systems
- Start simple and evolve complexity gradually

**Common Pitfalls to Avoid:**
- Premature sharding before exploring alternatives
- Poor sharding key selection leading to hotspots
- Insufficient testing of failure scenarios
- Neglecting operational complexity
- Inadequate monitoring and observability

### Final Interview Advice

When discussing sharding in interviews, demonstrate:

1. **Understanding of Trade-offs**: Show that you understand sharding is not a silver bullet and comes with significant complexity
2. **Practical Experience**: Discuss real-world challenges you've faced and how you solved them
3. **Operational Thinking**: Consider monitoring, maintenance, and disaster recovery from the start
4. **Gradual Approach**: Advocate for incremental adoption rather than big-bang migrations
5. **Alternative Awareness**: Mention other scaling techniques and when they might be more appropriate

The key to successful sharding lies not just in the technical implementation, but in the operational discipline and organizational readiness to manage distributed data systems effectively.