---
title: MVCC Detailed Guide
date: 2025-06-07 20:12:50
tags: [mysql]
categories: [mysql]
---

# What is MVCC(Multi-Version Concurrency Control)?

MVCC is a concurrency control method that allows multiple transactions to access the same data simultaneously without blocking each other. Instead of using locks for reads, MVCC maintains multiple versions of data and shows each transaction a consistent snapshot based on when the transaction started.

# Why MVCC Matters

## Traditional Locking Problems
Without MVCC, databases face the **readers-writers problem**:
- **Readers block writers**: Transactions reading data prevent others from modifying it
- **Writers block readers**: Transactions modifying data prevent others from reading it
- **Performance bottleneck**: High contention leads to poor concurrency

## MVCC Benefits
- **Non-blocking reads**: Readers never block writers and vice versa
- **Consistent snapshots**: Each transaction sees a consistent view of data
- **Higher concurrency**: Multiple transactions can work simultaneously
- **ACID compliance**: Maintains isolation without sacrificing performance

# Core MVCC Components

## Hidden Columns in InnoDB

Every InnoDB table row contains hidden system columns:

```
| User Data | DB_TRX_ID | DB_ROLL_PTR | DB_ROW_ID |
|-----------|-----------|-------------|-----------|
| name, age | 12345     | 0x8A2B...   | 67890     |
```

### DB_TRX_ID (Transaction ID)
- **Size**: 6 bytes
- **Purpose**: Identifies which transaction last modified this row
- **Behavior**: Updated every time a row is inserted or updated
- **Uniqueness**: Globally unique, monotonically increasing

### DB_ROLL_PTR (Rollback Pointer)
- **Size**: 7 bytes
- **Purpose**: Points to the undo log record for this row's previous version
- **Structure**: Contains undo log segment ID and offset
- **Function**: Forms the backbone of the version chain

### DB_ROW_ID (Row ID)
- **Size**: 6 bytes
- **Purpose**: Auto-incrementing row identifier
- **When used**: Only when table has no primary key or unique index
- **Note**: Not directly related to MVCC, but part of InnoDB's row format

## Version Chains and Undo Log

### Version Chain Structure

When a row is modified multiple times, MVCC creates a version chain:

```
Current Row (TRX_ID: 103)
    ↓ (DB_ROLL_PTR)
Version 2 (TRX_ID: 102) ← Undo Log Entry
    ↓ (roll_ptr)
Version 1 (TRX_ID: 101) ← Undo Log Entry
    ↓ (roll_ptr)
Original (TRX_ID: 100) ← Undo Log Entry
```

### Detailed Example

Let's trace a row through multiple modifications:

#### Initial State
```sql
-- Transaction 100 inserts row
INSERT INTO users (name, age) VALUES ('Alice', 25);
```

**Row State:**
```
| name: Alice | age: 25 | DB_TRX_ID: 100 | DB_ROLL_PTR: NULL |
```

#### First Update
```sql
-- Transaction 101 updates age
UPDATE users SET age = 26 WHERE name = 'Alice';
```

**After Update:**
- **Current row**: `| name: Alice | age: 26 | DB_TRX_ID: 101 | DB_ROLL_PTR: 0x8A2B |`
- **Undo log entry**: `| operation: UPDATE | old_age: 25 | roll_ptr: NULL |`

#### Second Update
```sql
-- Transaction 102 updates name
UPDATE users SET name = 'Alicia' WHERE name = 'Alice';
```

**After Update:**
- **Current row**: `| name: Alicia | age: 26 | DB_TRX_ID: 102 | DB_ROLL_PTR: 0x8C3D |`
- **New undo entry**: `| operation: UPDATE | old_name: Alice | roll_ptr: 0x8A2B |`
- **Previous undo entry**: `| operation: UPDATE | old_age: 25 | roll_ptr: NULL |`

### Undo Log Types

#### INSERT Undo Log
```
| Type: INSERT | Table ID | Primary Key Values | Transaction ID |
```
- **Purpose**: Rolling back INSERT operations
- **Content**: Only primary key needed (for deletion)
- **Cleanup**: Purged immediately after transaction commits

#### UPDATE Undo Log
```
| Type: UPDATE | Table ID | Primary Key | Changed Columns | Old Values | roll_ptr |
```
- **Purpose**: Rolling back UPDATE operations and MVCC reads
- **Content**: Original values of modified columns
- **Cleanup**: Purged when no active transaction needs this version

#### DELETE Undo Log
```
| Type: DELETE | Table ID | Complete Row Data | roll_ptr |
```
- **Purpose**: Rolling back DELETE operations
- **Content**: Entire row data
- **Behavior**: Row is marked as deleted but not physically removed

## Read View Mechanism

### Read View Structure

A Read View is a snapshot of active transactions at a specific point in time:

```cpp
struct ReadView {
    trx_id_t    m_low_limit_id;     // Highest TRX_ID + 1 at creation time
    trx_id_t    m_up_limit_id;      // Lowest active TRX_ID at creation time
    trx_list_t  m_ids;             // List of active transaction IDs
    trx_id_t    m_creator_trx_id;   // Transaction ID that created this view
};
```

### Read View Fields Explained

#### m_low_limit_id (High Water Mark)
- **Definition**: Next transaction ID to be assigned
- **Rule**: Any TRX_ID ≥ m_low_limit_id is invisible (not yet started)

#### m_up_limit_id (Low Water Mark)
- **Definition**: Smallest active transaction ID when Read View was created
- **Rule**: Any TRX_ID < m_up_limit_id is visible (committed before snapshot)

#### m_ids (Active Transaction List)
- **Definition**: List of all active (uncommitted) transaction IDs
- **Rule**: Any TRX_ID in this list is invisible (uncommitted)

#### m_creator_trx_id
- **Definition**: ID of the transaction that created this Read View
- **Rule**: Changes made by this transaction are always visible to itself

### Visibility Algorithm

For each row version, MVCC determines visibility using this logic:

```python
def is_visible(row_trx_id, read_view):
    # Rule 1: Own changes are always visible
    if row_trx_id == read_view.m_creator_trx_id:
        return True
    
    # Rule 2: Future transactions are invisible
    if row_trx_id >= read_view.m_low_limit_id:
        return False
    
    # Rule 3: Very old transactions are visible
    if row_trx_id < read_view.m_up_limit_id:
        return True
    
    # Rule 4: Check if transaction was active
    if row_trx_id in read_view.m_ids:
        return False  # Was active, so invisible
    else:
        return True   # Was committed, so visible
```

### Detailed Visibility Example

**Scenario Setup:**
- Active transactions: 100, 102, 105
- Next TRX_ID to assign: 106
- Current transaction: 103 (reading data)

**Read View for Transaction 103:**
```
m_creator_trx_id: 103
m_up_limit_id: 100    (lowest active)
m_low_limit_id: 106   (next to assign)
m_ids: [100, 102, 105] (active transactions)
```

**Visibility Tests:**
- **TRX_ID 99**: Visible (< m_up_limit_id, committed before snapshot)
- **TRX_ID 100**: Invisible (in m_ids, still active)
- **TRX_ID 101**: Visible (not in m_ids, committed)
- **TRX_ID 102**: Invisible (in m_ids, still active)
- **TRX_ID 103**: Visible (own transaction)
- **TRX_ID 104**: Visible (not in m_ids, committed)
- **TRX_ID 105**: Invisible (in m_ids, still active)
- **TRX_ID 106**: Invisible (≥ m_low_limit_id, future transaction)

## Isolation Levels and Read Views

### READ COMMITTED
- **Read View Creation**: New Read View for **every** SELECT statement
- **Behavior**: Sees all changes committed before each individual statement
- **Result**: Can see different data within the same transaction (non-repeatable reads)

```sql
-- Transaction A
START TRANSACTION;
SELECT age FROM users WHERE name = 'Alice'; -- Returns 25

-- Transaction B commits: UPDATE users SET age = 26 WHERE name = 'Alice';

SELECT age FROM users WHERE name = 'Alice'; -- Returns 26 (different result!)
COMMIT;
```

### REPEATABLE READ
- **Read View Creation**: Single Read View at **first** SELECT statement
- **Behavior**: Consistent snapshot throughout the entire transaction
- **Result**: Same data for all reads within the transaction

```sql
-- Transaction A
START TRANSACTION;
SELECT age FROM users WHERE name = 'Alice'; -- Returns 25, creates Read View

-- Transaction B commits: UPDATE users SET age = 26 WHERE name = 'Alice';

SELECT age FROM users WHERE name = 'Alice'; -- Still returns 25 (consistent!)
COMMIT;
```

## MVCC Read Process (Step by Step)

### When a SELECT Statement Executes:

#### Step 1: Create or Reuse Read View
```sql
SELECT name, age FROM users WHERE user_id = 1;
```
- **READ COMMITTED**: Create new Read View
- **REPEATABLE READ**: Use existing Read View or create if first read

#### Step 2: Locate Current Row Version
- Use index or table scan to find the row
- Current row has latest TRX_ID and ROLL_PTR

#### Step 3: Apply Visibility Rules
- Check if current version is visible using Read View
- If visible, return this version
- If not visible, follow the version chain

#### Step 4: Traverse Version Chain
```
Current Row (TRX_ID: 105) → Not visible
    ↓ (follow ROLL_PTR)
Version in Undo (TRX_ID: 103) → Not visible
    ↓ (follow roll_ptr)
Version in Undo (TRX_ID: 101) → Visible! Return this version
```

#### Step 5: Return Appropriate Version
- Return the first visible version found
- If no visible version exists, row doesn't exist for this transaction

## MVCC Write Operations

### INSERT Operations
1. **Create new row** with current transaction's TRX_ID
2. **No undo log needed** for MVCC (only for rollback)
3. **Row immediately visible** to the inserting transaction
4. **Invisible to others** until transaction commits

### UPDATE Operations
1. **Create undo log entry** with original values
2. **Update current row** with new values and TRX_ID
3. **Link to previous version** via ROLL_PTR
4. **Original version remains** accessible via undo log

### DELETE Operations
1. **Mark row as deleted** (set delete flag)
2. **Create undo log entry** with complete row data
3. **Row remains physically present** but marked deleted
4. **Appears deleted to new transactions** but still visible to older ones

## Purge Process

### Why Purge is Needed
- Undo logs grow indefinitely without cleanup
- Old versions become unnecessary when no transaction needs them
- Storage space must be reclaimed

### Purge Thread Operation
1. **Identify purgeable versions**: No active transaction needs them
2. **Remove undo log entries**: Free up undo tablespace
3. **Physical row deletion**: Remove rows marked for deletion
4. **Index cleanup**: Remove deleted entries from secondary indexes

### Purge Lag Issues
When purge falls behind:
- **Undo tablespace growth**: Disk space consumption increases
- **Version chain length**: Longer chains slow down reads
- **Memory pressure**: More versions kept in buffer pool

## Performance Implications

### MVCC Benefits
- **High concurrency**: No read-write blocking
- **Consistent reads**: Snapshot isolation without locks
- **Predictable performance**: No lock contention delays

### MVCC Costs
- **Storage overhead**: Multiple versions consume space
- **Version traversal**: Long chains increase read latency
- **Purge overhead**: Background cleanup uses resources
- **Undo log I/O**: Additional disk operations for version chains

### Optimization Strategies
1. **Monitor purge lag**: Ensure purge keeps up with modifications
2. **Tune undo tablespace**: Size appropriately for workload
3. **Minimize long transactions**: Reduce version chain lengths
4. **Index optimization**: Reduce version traversal overhead

## Common MVCC Scenarios

### Phantom Reads Prevention
```sql
-- Transaction 1 (REPEATABLE READ)
START TRANSACTION;
SELECT COUNT(*) FROM orders WHERE amount > 1000; -- Returns 5

-- Transaction 2 inserts new row
INSERT INTO orders (amount) VALUES (1500);
COMMIT;

-- Transaction 1 continues
SELECT COUNT(*) FROM orders WHERE amount > 1000; -- Still returns 5
COMMIT;
```

### Consistent Backup
```sql
-- Long-running backup transaction
START TRANSACTION WITH CONSISTENT SNAPSHOT;
-- Takes hours to complete, but sees consistent point-in-time data
mysqldump --single-transaction ...
COMMIT;
```

### Read-Write Workload
```sql
-- Reader transaction
START TRANSACTION;
SELECT * FROM accounts WHERE account_id = 1; -- Non-blocking read

-- Writer transaction (concurrent)
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1; -- Non-blocking write
COMMIT;

-- Reader continues with original snapshot
SELECT * FROM accounts WHERE account_id = 1; -- Still sees original balance
COMMIT;
```

This comprehensive understanding of MVCC explains how MySQL achieves high concurrency while maintaining data consistency, making it essential knowledge for database administrators and developers working with high-performance applications.