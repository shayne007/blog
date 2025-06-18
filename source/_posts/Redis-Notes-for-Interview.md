---
title: Redis Notes for Interview
author: Charlie
date: 2025-06-01 16:20:52
tags: [redis]
categories: [redis]
---

## Redis Fundamentals

### Key Concepts
- **In-memory database**
- **Data structures**: string, hash, set, sorted set, list, geo, hyperloglog
- **Cluster modes**: master-slave, sentinel, cluster sharding
- **Performance optimization through caching**

### Why Use Redis?

**Performance**: When encountering SQL queries that take a long time to execute and have results that don't change frequently, it's ideal to store the results in cache. This allows subsequent requests to read from cache, enabling rapid response times.

**Concurrency**: Under high concurrency, if all requests directly access the database, connection exceptions will occur. Redis serves as a buffer, allowing requests to access Redis first instead of directly hitting the database. (MySQL supports ~1,500 QPS, while Redis supports 20,000-50,000 QPS)

### Redis Use Cases
- Caching
- Flash sales/spike traffic handling
- Distributed locking

## Redis Single-Threading Model

### Key Concepts
- **Thread tasks**: command processing, I/O handling, persistence, data synchronization
- **Version 6.0+**: configurable multi-threading support
- **epoll + reactor pattern**

### High Performance Reasons

**Memory operations**: All data operations occur in memory

**I/O Model on Linux**: Uses epoll combined with reactor pattern
- **epoll fundamentals**: Manages multiple socket file descriptors
  - **Red-black tree** structure maintains all monitored file descriptors
  - **Doubly linked list** maintains the ready list
  - **Interrupt mechanism** adds ready file descriptors
  - **Key APIs**: `epoll_create`, `epoll_ctl`, `epoll_wait`
  - **Advantages over poll/select**: More efficient for large numbers of connections

- **Reactor pattern**:
  - **Reactor (Dispatcher)**: Calls epoll to get available file descriptors and dispatches events
  - **Acceptor**: Handles connection creation events
  - **Handler**: Processes I/O read/write events

## Redis Data Types and Underlying Structures

### String
- **Implementation**: SDS (Simple Dynamic String)
- **Use cases**: User info caching, counters, distributed locks (SETNX)

### Hash
- **Implementation**: ziplist (small data) + hashtable (large data)
- **Use cases**: Storing objects

### List
- **Implementation**: Doubly linked list or ziplist
- **Use cases**: Message queues, latest articles list

### Set
- **Implementation**: intset (integer set) or hashtable
- **Use cases**: Tag systems, mutual friends (SINTER for intersection)

### Sorted Set (ZSet)
- **Implementation**: ziplist + skiplist (skip list)
- **Use cases**: Leaderboards (ZADD/ZRANGE), delayed queues

## Redis Data Persistence

### Key Concepts
- **AOF (Append Only File)**
- **RDB (Redis Database)**
- **Hybrid mode**

### RDB (Redis Database Backup)
- **Mechanism**: Periodically generates binary snapshot files of memory data
- **Process**: Forks child process to avoid blocking main thread
- **Frequency**: Executes `BGSAVE` every 5+ minutes
- **Drawback**: Potential data loss between snapshots

### AOF (Append Only File)
- **Mechanism**: Records all write operation commands
- **Sync strategies**:
  - **Every second** (`appendfsync everysec`) - Default, good balance
  - **Every modification** (`appendfsync always`) - Safest but slowest
  - **No sync** (`appendfsync no`) - Fastest but risky
- **Rewrite mechanism**: Compacts AOF file by removing redundant commands

### Comparison
- **RDB**: Fast recovery, smaller files, but potential data loss during failures
- **AOF**: Better data integrity, but larger files and slower recovery

## Redis Cluster Deployment Modes

### Master-Slave Replication
- **Read operations**: Both master and slave nodes can handle reads
- **Write operations**: Only master handles writes, then syncs to slaves
- **Benefits**: Read scaling, basic fault tolerance

### Sentinel Mode
- **Purpose**: Automatic failover when master fails
- **Key functions**:
  - **Monitoring**: Continuously checks master/slave health
  - **Election**: Selects new master when current master fails
  - **Notification**: Informs clients of topology changes

- **Failure detection**:
  - **Subjective down**: Single sentinel marks master as down
  - **Objective down**: Majority of sentinels agree master is down
  
- **Master selection criteria**:
  1. Slave priority configuration
  2. Replication progress (most up-to-date)
  3. Smallest slave ID

### Cluster Sharding
- **Purpose**: Handles large datasets (>25GB) and utilizes multi-core CPUs
- **Hash slots**: Uses 16,384 fixed hash slots for data distribution
- **Benefits**: 
  - Horizontal scaling
  - Automatic failover
  - No single point of failure

## Caching Patterns and Consistency

### Key Concepts
- **Cache-Aside pattern**
- **Read Through, Write Through, Write Back**
- **Cache invalidation strategies**

### Cache-Aside Pattern
- **Read**: Check cache first; if miss, query database and populate cache
- **Write**: Update database first, then delete cache

### Ensuring Cache-Database Consistency

**Delayed Double Delete**:
1. Update database
2. Delete cache immediately
3. Wait brief period (100-500ms)
4. Delete cache again
5. **Trade-off**: Lower cache hit rate for better consistency

**Fallback strategies**:
- Set cache expiration times
- Use message queues for asynchronous synchronization

## Cache Problems: Penetration, Breakdown, Avalanche

### Cache Avalanche
- **Problem**: Many cache keys expire simultaneously or Redis instance crashes
- **Solutions**:
  - Random expiration times
  - Circuit breaker and rate limiting
  - High-availability cluster deployment
  - Service degradation

### Cache Penetration
- **Problem**: Queries for non-existent data bypass cache and hit database
- **Solutions**:
  - Cache null/default values with short TTL
  - **Bloom Filter**:
    - Data structure: Bit array + multiple hash functions
    - Write: Hash element multiple times, set corresponding bits to 1
    - Query: If all hash positions are 1, element might exist (false positives possible)
  - Input validation at application layer

### Cache Breakdown (Hotspot Invalid)
- **Problem**: Single popular cache key expires, causing traffic spike to database
- **Solutions**:
  - Never expire hot data
  - Use distributed locks to prevent multiple database hits
  - Pre-warming cache before expiration

## Distributed Locking with Redis

### Key Concepts
- **SETNX**: Only one client can successfully set the same key
- **Expiration time**: Prevents deadlocks
- **Lock renewal**: Extends lock duration for long-running operations

### Lock Retry on Failure
- **Wait time determination**: Based on 99th percentile business execution time
- **Implementation approaches**:
  - Sleep-based retry
  - Event-driven (listen for DEL events)
  - Lua script for atomic timeout retry

### Expiration Time Management
- **Why needed**: Prevents deadlocks when lock holder crashes
- **Setting strategy**: 
  - Analyze 99% of business operations completion time
  - Set 2x safety margin (e.g., if 99% complete in 1s, set 2s expiration)
  - For critical operations, consider 10s or 1 minute

### Lock Renewal
- **When needed**: Business operations exceed expiration time
- **Implementation**:
  - Reset expiration before current expiration
  - Daemon thread periodically checks and renews
  - **Redisson watchdog mechanism**: Automatic renewal

### Lock Release
- **Verification**: Always verify lock ownership before release
- **Prevention**: Avoid releasing locks held by other threads
- **Implementation**: Use Lua script for atomic check-and-delete

### Advanced Patterns
- **Redlock Algorithm**: Distributed consensus for multiple Redis instances
- **SingleFlight Pattern**: Prevents cache stampede by allowing only one request for the same resource

## Best Practices Summary

1. **Choose appropriate data structures** based on use case
2. **Implement proper persistence strategy** (RDB + AOF hybrid recommended)
3. **Design for high availability** with clustering and replication
4. **Handle cache problems proactively** with proper expiration and fallback strategies
5. **Use distributed locks carefully** with proper timeout and renewal mechanisms
6. **Monitor and optimize performance** regularly

