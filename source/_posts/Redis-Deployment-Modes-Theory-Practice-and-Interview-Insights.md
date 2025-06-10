---
title: 'Redis Deployment Modes: Theory, Practice, and Interview Insights'
date: 2025-06-10 16:21:06
tags: [redis]
categories: [redis]
---

## Introduction

Redis supports multiple deployment modes, each designed for different use cases, scalability requirements, and availability needs. Understanding these modes is crucial for designing robust, scalable systems.

**🎯 Common Interview Question**: *"How do you decide which Redis deployment mode to use for a given application?"*

**Answer Framework**: Consider these factors:
- **Data size**: Single instance practical limits (~25GB operational recommendation)
- **Availability requirements**: RTO/RPO expectations
- **Read/write patterns**: Read-heavy vs write-heavy workloads
- **Geographic distribution**: Single vs multi-region
- **Operational complexity**: Team expertise and maintenance overhead

## Standalone Redis

### Overview
Standalone Redis is the simplest deployment mode where a single Redis instance handles all operations. It's ideal for development, testing, and small-scale applications.

### Architecture

{% mermaid graph TB %}
    A[Client Applications] --> B[Redis Instance]
    B --> C[Disk Storage]
    
    style B fill:#ff9999
    style A fill:#99ccff
    style C fill:#99ff99
{% endmermaid %}

### Configuration Example

```redis
# redis.conf for standalone
port 6379
bind 127.0.0.1
maxmemory 2gb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
```

### Best Practices

1. **Memory Management**
   - Set `maxmemory` to 75% of available RAM
   - Choose appropriate eviction policy based on use case
   - Monitor memory fragmentation ratio

2. **Persistence Configuration**
   - Use AOF for critical data (better durability)
   - RDB for faster restarts and backups
   - Consider hybrid persistence for optimal balance

3. **Security**
   - Enable AUTH with strong passwords
   - Use TLS for client connections
   - Bind to specific interfaces, avoid 0.0.0.0

### Limitations and Use Cases

**Limitations:**
- Single point of failure
- Limited by single machine resources
- No automatic failover

**Optimal Use Cases:**
- Development and testing environments
- Applications with < 25GB data (to avoid RDB performance impact)
- Non-critical applications where downtime is acceptable
- Cache-only scenarios with acceptable data loss

**🎯 Interview Insight**: *"When would you NOT use standalone Redis?"*
Answer: When you need high availability (>99.9% uptime), **data sizes exceed 25GB** (RDB operations impact performance), or when application criticality requires zero data loss guarantees.

### RDB Operation Impact Analysis

**Critical Production Insight**: The **25GB threshold** is where RDB operations start significantly impacting online business:

{% mermaid graph LR %}
    A[BGSAVE Command] --> B["fork() syscall"]
    B --> C[Copy-on-Write Memory]
    C --> D[Memory Usage Spike]
    D --> E[Potential OOM]
    
    F[Write Operations] --> G[COW Page Copies]
    G --> H[Increased Latency]
    H --> I[Client Timeouts]
    
    style D fill:#ff9999
    style E fill:#ff6666
    style H fill:#ffcc99
    style I fill:#ff9999
{% endmermaid %}

**Real-world Impact at 25GB+:**
- **Memory spike**: Up to 2x memory usage during fork
- **Latency impact**: P99 latencies can spike from 1ms to 100ms+
- **CPU impact**: Fork operation can freeze Redis for 100ms-1s
- **I/O saturation**: Large RDB writes competing with normal operations

**Mitigation Strategies:**
1. **Disable automatic RDB**: Use `save ""` and only manual BGSAVE during low traffic
2. **AOF-only persistence**: More predictable performance impact
3. **Slave-based backups**: Perform RDB operations on slave instances
4. **Memory optimization**: Use compression, optimize data structures

## Redis Replication (Master-Slave)

### Overview
Redis replication creates exact copies of the master instance on one or more slave instances. It provides read scalability and basic redundancy.

### Architecture

{% mermaid graph TB %}
    A[Client - Writes] --> B[Redis Master]
    C[Client - Reads] --> D[Redis Slave 1]
    E[Client - Reads] --> F[Redis Slave 2]
    
    B --> D
    B --> F
    
    B --> G[Disk Storage Master]
    D --> H[Disk Storage Slave 1]
    F --> I[Disk Storage Slave 2]
    
    style B fill:#ff9999
    style D fill:#ffcc99
    style F fill:#ffcc99
{% endmermaid %}

### Configuration

**Master Configuration:**
```redis
# master.conf
port 6379
bind 0.0.0.0
requirepass masterpassword123
masterauth slavepassword123
```

**Slave Configuration:**
```redis
# slave.conf
port 6380
bind 0.0.0.0
slaveof 192.168.1.100 6379
masterauth masterpassword123
requirepass slavepassword123
slave-read-only yes
```

### Replication Process Flow

{% mermaid sequenceDiagram %}
    participant M as Master
    participant S as Slave
    participant C as Client
    
    Note over S: Initial Connection
    S->>M: PSYNC replicationid offset
    M->>S: +FULLRESYNC runid offset
    M->>S: RDB snapshot
    Note over S: Load RDB data
    M->>S: Replication backlog commands
    
    Note over M,S: Ongoing Replication
    C->>M: SET key value
    M->>S: SET key value
    C->>S: GET key
    S->>C: value
{% endmermaid %}

### Best Practices

1. **Network Optimization**
   - Use `repl-diskless-sync yes` for fast networks
   - Configure `repl-backlog-size` based on network latency
   - Monitor replication lag with `INFO replication`

2. **Slave Configuration**
   - Set `slave-read-only yes` to prevent accidental writes
   - Use `slave-priority` for failover preferences
   - Configure appropriate `slave-serve-stale-data` behavior

3. **Monitoring Key Metrics**
   - Replication offset difference
   - Last successful sync time
   - Number of connected slaves

### Production Showcase

```bash
#!/bin/bash
# Production deployment script for master-slave setup

# Start master
redis-server /etc/redis/master.conf --daemonize yes

# Wait for master to be ready
redis-cli ping

# Start slaves
for i in {1..2}; do
    redis-server /etc/redis/slave${i}.conf --daemonize yes
done

# Verify replication
redis-cli -p 6379 INFO replication
```

**🎯 Interview Question**: *"How do you handle slave promotion in a master-slave setup?"*

**Answer**: Manual promotion involves:
1. Stop writes to current master
2. Ensure slave is caught up (`LASTSAVE` comparison)
3. Execute `SLAVEOF NO ONE` on chosen slave
4. Update application configuration to point to new master
5. Configure other slaves to replicate from new master

**Limitation**: No automatic failover - requires manual intervention or external tooling.

## Redis Sentinel

### Overview
Redis Sentinel provides high availability for Redis through automatic failover, monitoring, and configuration management. It's the recommended solution for automatic failover in non-clustered environments.

### Architecture

{% mermaid graph TB %}
    subgraph "Redis Instances"
        M[Redis Master]
        S1[Redis Slave 1]
        S2[Redis Slave 2]
    end
    
    subgraph "Sentinel Cluster"
        SE1[Sentinel 1]
        SE2[Sentinel 2]
        SE3[Sentinel 3]
    end
    
    subgraph "Applications"
        A1[App Instance 1]
        A2[App Instance 2]
    end
    
    M --> S1
    M --> S2
    
    SE1 -.-> M
    SE1 -.-> S1
    SE1 -.-> S2
    SE2 -.-> M
    SE2 -.-> S1
    SE2 -.-> S2
    SE3 -.-> M
    SE3 -.-> S1
    SE3 -.-> S2
    
    A1 --> SE1
    A2 --> SE2
    
    style M fill:#ff9999
    style S1 fill:#ffcc99
    style S2 fill:#ffcc99
    style SE1 fill:#99ccff
    style SE2 fill:#99ccff
    style SE3 fill:#99ccff
{% endmermaid %}

### Sentinel Configuration

```redis
# sentinel.conf
port 26379
bind 0.0.0.0

# Monitor master named "mymaster"
sentinel monitor mymaster 192.168.1.100 6379 2
sentinel auth-pass mymaster masterpassword123

# Failover configuration
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1

# Notification scripts
sentinel notification-script mymaster /path/to/notify.sh
sentinel client-reconfig-script mymaster /path/to/reconfig.sh
```

### Failover Process

{% mermaid sequenceDiagram %}
    participant S1 as Sentinel 1
    participant S2 as Sentinel 2
    participant S3 as Sentinel 3
    participant M as Master
    participant SL as Slave
    participant A as Application
    
    Note over S1,S3: Normal Monitoring
    S1->>M: PING
    M--xS1: No Response
    S1->>S2: Master seems down
    S1->>S3: Master seems down
    
    Note over S1,S3: Quorum Check
    S2->>M: PING
    M--xS2: No Response
    S3->>M: PING
    M--xS3: No Response
    
    Note over S1,S3: Failover Decision
    S1->>S2: Start failover?
    S2->>S1: Agreed
    S1->>SL: SLAVEOF NO ONE
    S1->>A: New master notification
{% endmermaid %}

### Best Practices

1. **Quorum Configuration**
   - Use odd number of sentinels (3, 5, 7)
   - Set quorum to majority (e.g., 2 for 3 sentinels)
   - Deploy sentinels across different failure domains

2. **Timing Parameters**
   - `down-after-milliseconds`: 5-30 seconds based on network conditions
   - `failover-timeout`: 2-3x down-after-milliseconds
   - `parallel-syncs`: Usually 1 to avoid overwhelming new master

3. **Client Integration**
```python
import redis.sentinel

# Python client example
sentinels = [('localhost', 26379), ('localhost', 26380), ('localhost', 26381)]
sentinel = redis.sentinel.Sentinel(sentinels, socket_timeout=0.1)

# Discover master
master = sentinel.master_for('mymaster', socket_timeout=0.1)
slave = sentinel.slave_for('mymaster', socket_timeout=0.1)

# Use connections
master.set('key', 'value')
value = slave.get('key')
```

### Production Monitoring Script

```bash
#!/bin/bash
# Sentinel health check script

SENTINEL_PORT=26379
MASTER_NAME="mymaster"

# Check sentinel status
for port in 26379 26380 26381; do
    echo "Checking Sentinel on port $port"
    redis-cli -p $port SENTINEL masters | grep -A 20 $MASTER_NAME
    echo "---"
done

# Check master discovery
redis-cli -p $SENTINEL_PORT SENTINEL get-master-addr-by-name $MASTER_NAME
```

**🎯 Interview Question**: *"How does Redis Sentinel handle split-brain scenarios?"*

**Answer**: Sentinel prevents split-brain through:
1. **Quorum requirement**: Only majority can initiate failover
2. **Epoch mechanism**: Each failover gets unique epoch number
3. **Leader election**: Only one sentinel leads failover process
4. **Configuration propagation**: All sentinels must agree on new configuration

**Key Point**: Even if network partitions occur, only the partition with quorum majority can perform failover, preventing multiple masters.

## Redis Cluster

### Overview
Redis Cluster provides horizontal scaling and high availability through data sharding across multiple nodes. It's designed for applications requiring both high performance and large data sets.

### Architecture

{% mermaid graph TB %}
    subgraph "Redis Cluster"
        subgraph "Shard 1"
            M1[Master 1<br/>Slots 0-5460]
            S1[Slave 1]
        end
        
        subgraph "Shard 2"
            M2[Master 2<br/>Slots 5461-10922]
            S2[Slave 2]
        end
        
        subgraph "Shard 3"
            M3[Master 3<br/>Slots 10923-16383]
            S3[Slave 3]
        end
    end
    
    M1 --> S1
    M2 --> S2
    M3 --> S3
    
    M1 -.-> M2
    M1 -.-> M3
    M2 -.-> M3
    
    A[Application] --> M1
    A --> M2
    A --> M3
    
    style M1 fill:#ff9999
    style M2 fill:#ff9999
    style M3 fill:#ff9999
    style S1 fill:#ffcc99
    style S2 fill:#ffcc99
    style S3 fill:#ffcc99
{% endmermaid %}

### Hash Slot Distribution

Redis Cluster uses consistent hashing with 16,384 slots:

{% mermaid graph LR %}
    A[Key] --> B[CRC16]
    B --> C[% 16384]
    C --> D[Hash Slot]
    D --> E[Node Assignment]
    
    F[Example: user:1000] --> G[CRC16 = 31949]
    G --> H[31949 % 16384 = 15565]
    H --> I[Slot 15565 → Node 3]
{% endmermaid %}

### Cluster Configuration

**Node Configuration:**
```redis
# cluster-node.conf
port 7000
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000
appendonly yes
```

**Cluster Setup Script:**
```bash
#!/bin/bash
# Create 6-node cluster (3 masters, 3 slaves)

# Start nodes
for port in 7000 7001 7002 7003 7004 7005; do
    redis-server --port $port --cluster-enabled yes \
                 --cluster-config-file nodes-${port}.conf \
                 --cluster-node-timeout 5000 \
                 --appendonly yes --daemonize yes
done

# Create cluster
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 \
                           127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
                           --cluster-replicas 1
```

### Data Distribution and Client Routing

{% mermaid sequenceDiagram %}
    participant C as Client
    participant N1 as Node 1
    participant N2 as Node 2
    participant N3 as Node 3
    
    C->>N1: GET user:1000
    Note over N1: Check slot ownership
    alt Key belongs to N1
        N1->>C: value
    else Key belongs to N2
        N1->>C: MOVED 15565 192.168.1.102:7001
        C->>N2: GET user:1000
        N2->>C: value
    end
{% endmermaid %}

### Advanced Operations

**Resharding Example:**
```bash
# Move 1000 slots from node 1 to node 4
redis-cli --cluster reshard 127.0.0.1:7000 \
          --cluster-from 1a2b3c4d... \
          --cluster-to 5e6f7g8h... \
          --cluster-slots 1000
```

**Adding New Nodes:**
```bash
# Add new master
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# Add new slave
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7000 --cluster-slave
```

### Client Implementation Best Practices

```python
import redis.cluster

# Python cluster client
startup_nodes = [
    {"host": "127.0.0.1", "port": "7000"},
    {"host": "127.0.0.1", "port": "7001"},
    {"host": "127.0.0.1", "port": "7002"}
]

cluster = redis.cluster.RedisCluster(
    startup_nodes=startup_nodes,
    decode_responses=True,
    skip_full_coverage_check=True,
    health_check_interval=30
)

# Hash tags for multi-key operations
cluster.mset({
    "user:{1000}:name": "Alice",
    "user:{1000}:email": "alice@example.com",
    "user:{1000}:age": "30"
})
```

### Limitations and Considerations

1. **Multi-key Operations**: Limited to same hash slot
2. **Lua Scripts**: All keys must be in same slot
3. **Database Selection**: Only database 0 supported
4. **Client Complexity**: Requires cluster-aware clients

**🎯 Interview Question**: *"How do you handle hotspot keys in Redis Cluster?"*

**Answer Strategies**:
1. **Hash tags**: Distribute related hot keys across slots
2. **Client-side caching**: Cache frequently accessed data
3. **Read replicas**: Use slave nodes for read operations
4. **Application-level sharding**: Pre-shard at application layer
5. **Monitoring**: Use `redis-cli --hotkeys` to identify patterns

## Deployment Architecture Comparison

### Feature Matrix

| Feature | Standalone | Replication | Sentinel | Cluster |
|---------|------------|-------------|----------|---------|
| **High Availability** | ❌ | ❌ | ✅ | ✅ |
| **Automatic Failover** | ❌ | ❌ | ✅ | ✅ |
| **Horizontal Scaling** | ❌ | ❌ | ❌ | ✅ |
| **Read Scaling** | ❌ | ✅ | ✅ | ✅ |
| **Operational Complexity** | Low | Low | Medium | High |
| **Multi-key Operations** | ✅ | ✅ | ✅ | Limited |
| **Max Data Size** | Single Node | Single Node | Single Node | Multi-Node |

### Decision Flow Chart

{% mermaid flowchart TD %}
    A[Start: Redis Deployment Decision] --> B{Data Size > 25GB?}
    B -->|Yes| C{Can tolerate RDB impact?}
    C -->|No| D[Consider Redis Cluster]
    C -->|Yes| E{High Availability Required?}
    B -->|No| E
    E -->|No| F{Read Scaling Needed?}
    F -->|Yes| G[Master-Slave Replication]
    F -->|No| H[Standalone Redis]
    E -->|Yes| I{Automatic Failover Needed?}
    I -->|Yes| J[Redis Sentinel]
    I -->|No| G
    
    style D fill:#ff6b6b
    style J fill:#4ecdc4
    style G fill:#45b7d1
    style H fill:#96ceb4
{% endmermaid %}

## Production Considerations

### Hardware Sizing Guidelines

**CPU Requirements:**
- Standalone/Replication: 2-4 cores
- Sentinel: 1-2 cores per sentinel
- Cluster: 4-8 cores per node

**Memory Guidelines:**
```
Total RAM = (Dataset Size × 1.5) + OS overhead
Example: 100GB dataset = 150GB + 16GB = 166GB total RAM
```

**Network Considerations:**
- Replication: 1Gbps minimum for large datasets
- Cluster: Low latency (<1ms) between nodes
- Client connections: Plan for connection pooling

### Security Best Practices

```redis
# Production security configuration
bind 127.0.0.1 10.0.0.0/8
protected-mode yes
requirepass your-secure-password-here
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG_b9f8e7a6d2c1"

# TLS configuration
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```

### Backup and Recovery Strategy

```bash
#!/bin/bash
# Comprehensive backup script

REDIS_HOST="localhost"
REDIS_PORT="6379"
BACKUP_DIR="/var/backups/redis"
DATE=$(date +%Y%m%d_%H%M%S)

# Create RDB backup
redis-cli -h $REDIS_HOST -p $REDIS_PORT BGSAVE
sleep 5

# Wait for background save to complete
while [ $(redis-cli -h $REDIS_HOST -p $REDIS_PORT LASTSAVE) -eq $LASTSAVE ]; do
    sleep 1
done

# Copy files
cp /var/lib/redis/dump.rdb $BACKUP_DIR/dump_$DATE.rdb
cp /var/lib/redis/appendonly.aof $BACKUP_DIR/aof_$DATE.aof

# Compress and upload to S3
tar -czf $BACKUP_DIR/redis_backup_$DATE.tar.gz $BACKUP_DIR/*_$DATE.*
aws s3 cp $BACKUP_DIR/redis_backup_$DATE.tar.gz s3://redis-backups/
```

## Monitoring and Operations

### Key Performance Metrics

```bash
#!/bin/bash
# Redis monitoring script

redis-cli INFO all | grep -E "(used_memory_human|connected_clients|total_commands_processed|keyspace_hits|keyspace_misses|role|master_repl_offset)"

# Cluster-specific monitoring
if redis-cli CLUSTER NODES &>/dev/null; then
    echo "=== Cluster Status ==="
    redis-cli CLUSTER NODES
    redis-cli CLUSTER INFO
fi
```

### Alerting Thresholds

| Metric | Warning | Critical |
|--------|---------|----------|
| **Memory Usage** | >80% | >90% |
| **Hit Ratio** | <90% | <80% |
| **Connected Clients** | >80% max | >95% max |
| **Replication Lag** | >10s | >30s |
| **Cluster State** | degraded | fail |

### Troubleshooting Common Issues

**Memory Fragmentation:**
```bash
# Check fragmentation ratio
redis-cli INFO memory | grep mem_fragmentation_ratio

# If ratio > 1.5, consider:
# 1. Restart Redis during maintenance window
# 2. Enable active defragmentation
CONFIG SET activedefrag yes
```

**Slow Queries:**
```bash
# Enable slow log
CONFIG SET slowlog-log-slower-than 10000
CONFIG SET slowlog-max-len 128

# Check slow queries
SLOWLOG GET 10
```

**🎯 Interview Question**: *"How do you handle Redis memory pressure in production?"*

**Comprehensive Answer**:
1. **Immediate actions**: Check `maxmemory-policy`, verify no memory leaks
2. **Short-term**: Scale vertically, optimize data structures, enable compression
3. **Long-term**: Implement data archiving, consider clustering, optimize application usage patterns
4. **Monitoring**: Set up alerts for memory usage, track key expiration patterns

## Conclusion

Choosing the right Redis deployment mode depends on your specific requirements for availability, scalability, and operational complexity. Start simple with standalone or replication for smaller applications, progress to Sentinel for high availability needs, and adopt Cluster for large-scale, horizontally distributed systems.

**Final Interview Insight**: The key to Redis success in production is not just choosing the right deployment mode, but also implementing proper monitoring, backup strategies, and operational procedures. Always plan for failure scenarios and test your disaster recovery procedures regularly.

Remember: **"The best Redis deployment is the simplest one that meets your requirements."**