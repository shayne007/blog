---
title: 'MySQL Logs: Binlog, Redo Log, and Undo Log'
date: 2025-06-09 02:23:31
tags: [mysql]
categories: [mysql]
---


MySQL's logging mechanisms are fundamental to its reliability, performance, and replication capabilities. Understanding the three primary logs—binary log (binlog), redo log, and undo log—is crucial for database administrators and developers working with MySQL at scale.

## Overview and Architecture

MySQL employs a multi-layered logging architecture where each log serves specific purposes:

- **Redo Log (InnoDB)**: Ensures crash recovery and durability (ACID compliance)
- **Undo Log (InnoDB)**: Enables transaction rollback and MVCC (Multi-Version Concurrency Control)
- **Binary Log (Server Level)**: Facilitates replication and point-in-time recovery

{% mermaid graph TB %}
    subgraph "MySQL Server"
        subgraph "Server Layer"
            SQL[SQL Layer]
            BL[Binary Log]
        end
        
        subgraph "InnoDB Storage Engine"
            BP[Buffer Pool]
            RL[Redo Log]
            UL[Undo Log]
            DF[Data Files]
        end
    end
    
    Client[Client Application] --> SQL
    SQL --> BL
    SQL --> BP
    BP --> RL
    BP --> UL
    BP --> DF
    
    BL --> Slave[Replica Server]
    RL --> Recovery[Crash Recovery]
    UL --> MVCC[MVCC Reads]
    
    style RL fill:#e1f5fe
    style UL fill:#f3e5f5
    style BL fill:#e8f5e8
{% endmermaid %}

These logs work together to provide MySQL's ACID guarantees while supporting high-availability architectures through replication.

## Redo Log: Durability and Crash Recovery

### Core Concepts

The redo log is InnoDB's crash recovery mechanism that ensures committed transactions survive system failures. It operates on the Write-Ahead Logging (WAL) principle, where changes are logged before being written to data files.

**Key Characteristics:**
- Physical logging of page-level changes
- Circular buffer structure with configurable size
- Synchronous writes for committed transactions
- Critical for MySQL's durability guarantee

### Technical Implementation

The redo log consists of multiple files (typically `ib_logfile0`, `ib_logfile1`) that form a circular buffer. When InnoDB modifies a page, it first writes the change to the redo log, then marks the page as "dirty" in the buffer pool for eventual flushing to disk.

{% mermaid graph LR %}
    subgraph "Redo Log Circular Buffer"
        LF1[ib_logfile0]
        LF2[ib_logfile1]
        LF1 --> LF2
        LF2 --> LF1
    end
    
    subgraph "Write Process"
        Change[Data Change] --> WAL[Write to Redo Log]
        WAL --> Mark[Mark Page Dirty]
        Mark --> Flush[Background Flush to Disk]
    end
    
    LSN1[LSN: 12345]
    LSN2[LSN: 12346] 
    LSN3[LSN: 12347]
    
    Change --> LSN1
    LSN1 --> LSN2
    LSN2 --> LSN3
    
    style LF1 fill:#e1f5fe
    style LF2 fill:#e1f5fe
{% endmermaid %}

**Log Sequence Number (LSN):** A monotonically increasing number that uniquely identifies each redo log record. LSNs are crucial for recovery operations and determining which changes need to be applied during crash recovery.

### Configuration and Monitoring

```sql
-- Monitor redo log activity and health
SHOW ENGINE INNODB STATUS\G

-- Key metrics to watch:
-- Log sequence number: Current LSN
-- Log flushed up to: Last flushed LSN  
-- Pages flushed up to: Last checkpoint LSN
-- Last checkpoint at: Checkpoint LSN

-- Check for redo log waits (performance bottleneck indicator)
SHOW GLOBAL STATUS LIKE 'Innodb_log_waits';
-- Should be 0 or very low in healthy systems

-- Diagnostic script for redo log issues
SELECT 
    'Log Waits' as Metric,
    variable_value as Value,
    CASE 
        WHEN CAST(variable_value AS UNSIGNED) > 100 THEN 'CRITICAL - Increase redo log size'
        WHEN CAST(variable_value AS UNSIGNED) > 10 THEN 'WARNING - Monitor closely'
        ELSE 'OK'
    END as Status
FROM performance_schema.global_status 
WHERE variable_name = 'Innodb_log_waits';
```

**Key Configuration Parameters:**

```sql
-- Optimal production settings
innodb_log_file_size = 2G          -- Size of each redo log file
innodb_log_files_in_group = 2      -- Number of redo log files  
innodb_flush_log_at_trx_commit = 1 -- Full ACID compliance
innodb_log_buffer_size = 64M       -- Buffer for high concurrency
```

**Performance Tuning Guidelines:**

1. **Log File Sizing**: Size the total redo log space to handle 60-90 minutes of peak write activity. Larger logs reduce checkpoint frequency but increase recovery time.

2. **Flush Strategy**: The `innodb_flush_log_at_trx_commit` parameter controls durability vs. performance:
   - `1` (default): Full ACID compliance, flush and sync on each commit
   - `2`: Flush on commit, sync every second (risk: 1 second of transactions on OS crash)
   - `0`: Flush and sync every second (risk: 1 second of transactions on MySQL crash)

### Interview Deep Dive: Checkpoint Frequency vs Recovery Time

**Common Question**: "Explain the relationship between checkpoint frequency and redo log size. How does this impact recovery time?"

{% mermaid graph LR %}
    subgraph "Small Redo Logs"
        SRL1[Frequent Checkpoints] --> SRL2[Less Dirty Pages]
        SRL2 --> SRL3[Fast Recovery]
        SRL1 --> SRL4[More I/O Overhead]
        SRL4 --> SRL5[Slower Performance]
    end
    
    subgraph "Large Redo Logs"  
        LRL1[Infrequent Checkpoints] --> LRL2[More Dirty Pages]
        LRL2 --> LRL3[Slower Recovery]
        LRL1 --> LRL4[Less I/O Overhead]
        LRL4 --> LRL5[Better Performance]
    end
    
    style SRL3 fill:#e8f5e8
    style SRL5 fill:#ffebee
    style LRL3 fill:#ffebee
    style LRL5 fill:#e8f5e8
{% endmermaid %}

**Answer Framework:**
- Checkpoint frequency is inversely related to redo log size
- Small logs: fast recovery, poor performance during high writes
- Large logs: slow recovery, better steady-state performance
- Sweet spot: size logs for 60-90 minutes of peak write activity
- Monitor `Innodb_log_waits` to detect undersized logs

## Undo Log: Transaction Rollback and MVCC

### Fundamental Role

Undo logs serve dual purposes: enabling transaction rollback and supporting MySQL's MVCC implementation for consistent reads. They store the inverse operations needed to undo changes made by transactions.

**MVCC Implementation:**
When a transaction reads data, InnoDB uses undo logs to reconstruct the appropriate version of the data based on the transaction's read view, enabling non-blocking reads even while other transactions are modifying the same data.

### Undo Log Structure and MVCC Showcase

{% mermaid graph TB %}
    subgraph "Transaction Timeline"
        T1[Transaction 1<br/>Read View: LSN 100]
        T2[Transaction 2<br/>Read View: LSN 200]  
        T3[Transaction 3<br/>Read View: LSN 300]
    end
    
    subgraph "Data Versions via Undo Chain"
        V1[Row Version 1<br/>LSN 100<br/>Value: 'Alice']
        V2[Row Version 2<br/>LSN 200<br/>Value: 'Bob']
        V3[Row Version 3<br/>LSN 300<br/>Value: 'Charlie']
        
        V3 --> V2
        V2 --> V1
    end
    
    T1 --> V1
    T2 --> V2
    T3 --> V3
    
    style V1 fill:#f3e5f5
    style V2 fill:#f3e5f5  
    style V3 fill:#f3e5f5
{% endmermaid %}

```sql
-- Demonstrate MVCC in action
-- Terminal 1: Start long-running transaction
START TRANSACTION;
SELECT * FROM users WHERE id = 1;  -- Returns: name = 'Alice'
-- Don't commit yet - keep transaction open

-- Terminal 2: Update the same row
UPDATE users SET name = 'Bob' WHERE id = 1;
COMMIT;

-- Terminal 1: Read again - still sees 'Alice' due to MVCC
SELECT * FROM users WHERE id = 1;  -- Still returns: name = 'Alice'

-- Terminal 3: New transaction sees latest data
START TRANSACTION;
SELECT * FROM users WHERE id = 1;  -- Returns: name = 'Bob'
```

### Management and Troubleshooting

```sql
-- Comprehensive undo log diagnostic script
-- 1. Check for long-running transactions
SELECT 
    trx_id,
    trx_started,
    trx_mysql_thread_id,
    TIMESTAMPDIFF(MINUTE, trx_started, NOW()) as duration_minutes,
    trx_rows_locked,
    trx_rows_modified,
    LEFT(trx_query, 100) as query_snippet
FROM information_schema.innodb_trx 
WHERE trx_started < NOW() - INTERVAL 5 MINUTE
ORDER BY trx_started;

-- 2. Monitor undo tablespace usage
SELECT 
    tablespace_name,
    file_name,
    ROUND(file_size/1024/1024, 2) as size_mb,
    ROUND(allocated_size/1024/1024, 2) as allocated_mb
FROM information_schema.files 
WHERE tablespace_name LIKE '%undo%';

-- 3. Check purge thread activity
SELECT 
    variable_name,
    variable_value
FROM performance_schema.global_status 
WHERE variable_name IN (
    'Innodb_purge_trx_id_age',
    'Innodb_purge_undo_no'
);
```

**Best Practices:**
1. **Transaction Hygiene**: Keep transactions short to prevent undo log accumulation
2. **Undo Tablespace Management**: Use dedicated undo tablespaces (`innodb_undo_tablespaces = 4`)
3. **Purge Thread Tuning**: Configure `innodb_purge_threads = 4` for better cleanup performance

## Binary Log: Replication and Recovery

### Architecture and Purpose

The binary log operates at the MySQL server level (above storage engines) and records all statements that modify data. It's essential for replication and point-in-time recovery operations.

**Logging Formats:**
- **Statement-Based (SBR)**: Logs SQL statements
- **Row-Based (RBR)**: Logs actual row changes (recommended)
- **Mixed**: Automatically switches between statement and row-based logging

### Replication Mechanics

{% mermaid sequenceDiagram %}
    participant App as Application
    participant Master as Master Server
    participant BinLog as Binary Log
    participant Slave as Slave Server
    participant RelayLog as Relay Log
    
    App->>Master: INSERT/UPDATE/DELETE
    Master->>BinLog: Write binary log event
    Master->>App: Acknowledge transaction
    
    Slave->>BinLog: Request new events (I/O Thread)
    BinLog->>Slave: Send binary log events
    Slave->>RelayLog: Write to relay log
    
    Note over Slave: SQL Thread processes relay log
    Slave->>Slave: Apply changes to slave database
    
    Note over Master,Slave: Asynchronous replication
{% endmermaid %}

### Configuration and Format Comparison

```sql
-- View current binary log files
SHOW BINARY LOGS;

-- Examine binary log contents
SHOW BINLOG EVENTS IN 'mysql-bin.000002' LIMIT 5;

-- Compare different formats:
-- Statement-based logging
SET SESSION binlog_format = 'STATEMENT';
UPDATE users SET last_login = NOW() WHERE active = 1;
-- Logs: UPDATE users SET last_login = NOW() WHERE active = 1

-- Row-based logging (recommended)
SET SESSION binlog_format = 'ROW';
UPDATE users SET last_login = NOW() WHERE active = 1;
-- Logs: Actual row changes with before/after images
```

**Production Configuration:**

```sql
-- High-availability binary log setup
[mysqld]
# Enable binary logging with GTID
log-bin = mysql-bin
server-id = 1
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON

# Performance and retention
sync_binlog = 1
expire_logs_days = 7
max_binlog_size = 1G
binlog_cache_size = 2M
```

### Interview Scenario: Replication Lag Analysis

**Common Question**: "A production database suddenly slowed down with replication lag. How would you diagnose?"

```sql
-- Step-by-step diagnostic approach
-- 1. Check overall replication status
SHOW SLAVE STATUS\G
-- Key metrics: Seconds_Behind_Master, Master_Log_File vs Relay_Master_Log_File

-- 2. Identify bottleneck location
SELECT 
    'I/O Thread Performance' as check_type,
    IF(Master_Log_File = Relay_Master_Log_File, 'OK', 'I/O LAG') as status
-- Add actual SHOW SLAVE STATUS parsing logic here

-- 3. Check for problematic queries on slave
SELECT 
    schema_name,
    digest_text,
    count_star,
    avg_timer_wait/1000000000 as avg_seconds
FROM performance_schema.events_statements_summary_by_digest 
WHERE avg_timer_wait > 1000000000  -- > 1 second
ORDER BY avg_timer_wait DESC 
LIMIT 10;

-- 4. Monitor slave thread performance
SELECT 
    thread_id,
    name,
    processlist_state,
    processlist_time
FROM performance_schema.threads 
WHERE name LIKE '%slave%';
```

**Optimization Solutions:**
- Enable parallel replication: `slave_parallel_workers = 4`
- Optimize slow queries on slave
- Consider read/write splitting
- Network optimization between master and slave

## Transaction Commit Flow Integration

Understanding how these logs interact during transaction commits is crucial for troubleshooting and optimization:

{% mermaid flowchart TD %}
    Start([Transaction Begins]) --> Changes[Execute DML Statements]
    Changes --> UndoWrite[Write Undo Records]
    UndoWrite --> RedoWrite[Write Redo Log Records]
    RedoWrite --> Prepare[Prepare Phase]
    
    Prepare --> BinLogCheck{Binary Logging Enabled?}
    BinLogCheck -->|Yes| BinLogWrite[Write to Binary Log]
    BinLogCheck -->|No| RedoCommit[Write Redo Commit Record]
    
    BinLogWrite --> BinLogSync[Sync Binary Log<br/>「if sync_binlog=1」]
    BinLogSync --> RedoCommit
    
    RedoCommit --> RedoSync[Sync Redo Log<br/>「if innodb_flush_log_at_trx_commit=1」]
    RedoSync --> Complete([Transaction Complete])
    
    Complete --> UndoPurge[Mark Undo for Purge<br/>「Background Process」]
    
    style UndoWrite fill:#f3e5f5
    style RedoWrite fill:#e1f5fe
    style BinLogWrite fill:#e8f5e8
    style RedoCommit fill:#e1f5fe
{% endmermaid %}

### Group Commit Optimization

**Interview Insight**: "How does MySQL's group commit feature improve performance with binary logging enabled?"

Group commit allows multiple transactions to be fsynced together, reducing I/O overhead:

```sql
-- Monitor group commit efficiency
SHOW GLOBAL STATUS LIKE 'Binlog_commits';
SHOW GLOBAL STATUS LIKE 'Binlog_group_commits';
-- Higher ratio of group_commits to commits indicates better efficiency
```

## Crash Recovery and Point-in-Time Recovery

### Recovery Process Flow

{% mermaid graph TB %}
    subgraph "Crash Recovery Process"
        Crash[System Crash] --> Start[MySQL Restart]
        Start --> ScanRedo[Scan Redo Log from<br/>Last Checkpoint]
        ScanRedo --> RollForward[Apply Committed<br/>Transactions]
        RollForward --> ScanUndo[Scan Undo Logs for<br/>Uncommitted Transactions]
        ScanUndo --> RollBack[Rollback Uncommitted<br/>Transactions]
        RollBack --> BinLogSync[Synchronize with<br/>Binary Log Position]
        BinLogSync --> Ready[Database Ready]
    end
    
    style ScanRedo fill:#e1f5fe
    style ScanUndo fill:#f3e5f5
{% endmermaid %}

{% mermaid graph TB %}
    subgraph "Point-in-Time Recovery"
        Backup[Full Backup] --> RestoreData[Restore Data Files]
        RestoreData --> ApplyBinLog[Apply Binary Logs<br/>to Target Time]
        ApplyBinLog --> Recovered[Database Recovered<br/>to Specific Point]
    end
    
    style ApplyBinLog fill:#e8f5e8
{% endmermaid %}

### Point-in-Time Recovery Example

```sql
-- Practical PITR scenario
-- 1. Record current position before problematic operation
SHOW MASTER STATUS;
-- Example: File: mysql-bin.000003, Position: 1547

-- 2. After accidental data loss (e.g., DROP TABLE)
-- Recovery process (command line):

-- Stop MySQL and restore from backup
-- mysql < full_backup_before_incident.sql

-- Apply binary logs up to just before the problematic statement
-- mysqlbinlog --stop-position=1500 mysql-bin.000003 | mysql

-- Skip the problematic statement and continue
-- mysqlbinlog --start-position=1600 mysql-bin.000003 | mysql
```

## Environment-Specific Configurations

### Production-Grade Configuration

```sql
-- High-Performance Production Template
[mysqld]
# ============================================
# REDO LOG CONFIGURATION
# ============================================
innodb_log_file_size = 2G
innodb_log_files_in_group = 2
innodb_flush_log_at_trx_commit = 1  # Full ACID compliance
innodb_log_buffer_size = 64M

# ============================================
# UNDO LOG CONFIGURATION  
# ============================================
innodb_undo_tablespaces = 4
innodb_undo_logs = 128
innodb_purge_threads = 4

# ============================================
# BINARY LOG CONFIGURATION
# ============================================
log-bin = mysql-bin
server-id = 1
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
sync_binlog = 1
expire_logs_days = 7
binlog_cache_size = 2M

# ============================================
# GENERAL PERFORMANCE SETTINGS
# ============================================
innodb_buffer_pool_size = 8G  # 70-80% of RAM
innodb_buffer_pool_instances = 8
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
```

### Interview Scenario: Financial Application Design

**Question**: "How would you design a MySQL setup for a financial application that cannot lose any transactions?"

{% mermaid graph TB %}
    subgraph "Financial Grade Setup"
        App[Application] --> LB[Load Balancer]
        LB --> Master[Master DB]
        Master --> Sync1[Synchronous Slave 1]
        Master --> Sync2[Synchronous Slave 2]
        
        subgraph "Master Configuration"
            MC1[innodb_flush_log_at_trx_commit = 1]
            MC2[sync_binlog = 1] 
            MC3[Large redo logs for performance]
            MC4[GTID enabled]
        end
        
        subgraph "Monitoring"
            Mon1[Transaction timeout < 30s]
            Mon2[Undo log size alerts]
            Mon3[Replication lag < 1s]
        end
    end
    
    style Master fill:#e8f5e8
    style Sync1 fill:#e1f5fe
    style Sync2 fill:#e1f5fe
{% endmermaid %}

**Answer Framework:**
- **Durability**: `innodb_flush_log_at_trx_commit = 1` and `sync_binlog = 1`
- **Consistency**: Row-based binary logging with GTID
- **Availability**: Semi-synchronous replication
- **Performance**: Larger redo logs to handle synchronous overhead
- **Monitoring**: Aggressive alerting on log-related metrics

## Monitoring and Alerting

### Comprehensive Health Check Script

```sql
-- Complete MySQL logs health check
SELECT 'REDO LOG METRICS' as section, '' as metric, '' as value, '' as status
UNION ALL
SELECT 
    '',
    'Log Waits (should be 0)' as metric,
    variable_value as value,
    CASE 
        WHEN CAST(variable_value AS UNSIGNED) = 0 THEN '✓ EXCELLENT'
        WHEN CAST(variable_value AS UNSIGNED) < 10 THEN '⚠ WATCH'
        ELSE '✗ CRITICAL - Increase redo log size'
    END as status
FROM performance_schema.global_status 
WHERE variable_name = 'Innodb_log_waits'

UNION ALL
SELECT 'UNDO LOG METRICS' as section, '' as metric, '' as value, '' as status
UNION ALL
SELECT 
    '',
    'Long Running Transactions (>5 min)' as metric,
    COUNT(*) as value,
    CASE 
        WHEN COUNT(*) = 0 THEN '✓ GOOD'
        WHEN COUNT(*) < 5 THEN '⚠ MONITOR'
        ELSE '✗ CRITICAL - Kill long transactions'
    END as status
FROM information_schema.innodb_trx 
WHERE trx_started < NOW() - INTERVAL 5 MINUTE

UNION ALL
SELECT 'BINARY LOG METRICS' as section, '' as metric, '' as value, '' as status
UNION ALL
SELECT 
    '',
    'Binary Logging Status' as metric,
    @@log_bin as value,
    CASE 
        WHEN @@log_bin = 1 THEN '✓ ENABLED'
        ELSE '⚠ DISABLED'
    END as status

UNION ALL
SELECT 
    '',
    'Binlog Format' as metric,
    @@binlog_format as value,
    CASE 
        WHEN @@binlog_format = 'ROW' THEN '✓ RECOMMENDED'
        WHEN @@binlog_format = 'MIXED' THEN '⚠ ACCEPTABLE'
        ELSE '⚠ STATEMENT-BASED'
    END as status;
```

### Key Alert Thresholds

Establish monitoring for:
- **Redo log waits** > 100/second
- **Slave lag** > 30 seconds  
- **Long-running transactions** > 1 hour
- **Binary log disk usage** > 80%
- **Undo tablespace growth** > 20% per hour

### Real-Time Monitoring Dashboard

```sql
-- Create monitoring view for continuous observation
CREATE OR REPLACE VIEW mysql_logs_dashboard AS
SELECT 
    NOW() as check_time,
    
    -- Redo Log Metrics
    (SELECT variable_value FROM performance_schema.global_status 
     WHERE variable_name = 'Innodb_log_waits') as redo_log_waits,
    
    -- Undo Log Metrics  
    (SELECT COUNT(*) FROM information_schema.innodb_trx 
     WHERE trx_started < NOW() - INTERVAL 5 MINUTE) as long_transactions,
    
    -- Binary Log Metrics
    (SELECT variable_value FROM performance_schema.global_status 
     WHERE variable_name = 'Binlog_bytes_written') as binlog_bytes_written,
    
    -- Buffer Pool Hit Ratio
    ROUND(
        (1 - (
            (SELECT variable_value FROM performance_schema.global_status 
             WHERE variable_name = 'Innodb_buffer_pool_reads') / 
            (SELECT variable_value FROM performance_schema.global_status 
             WHERE variable_name = 'Innodb_buffer_pool_read_requests')
        )) * 100, 2
    ) as buffer_pool_hit_ratio;

-- Use the dashboard
SELECT * FROM mysql_logs_dashboard;
```

## Conclusion

MySQL's logging architecture provides a robust foundation for transaction processing, crash recovery, and high-availability deployments. Key takeaways:

1. **Redo logs** ensure durability through Write-Ahead Logging - size them for 60-90 minutes of peak writes
2. **Undo logs** enable MVCC and rollbacks - keep transactions short to prevent growth
3. **Binary logs** facilitate replication and PITR - use ROW format with GTID for modern deployments

The key to successful MySQL log management lies in understanding your workload's specific requirements and balancing durability, consistency, and performance. Regular monitoring of log metrics and proactive tuning ensure these critical systems continue to provide reliable service as your database scales.

Remember: in production environments, always test configuration changes in staging first, and maintain comprehensive monitoring to detect issues before they impact your applications.