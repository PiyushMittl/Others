# XA Transactions in MySQL and PostgreSQL

## Overview
XA Transactions are distributed transactions that follow the X/Open XA standard for two-phase commit (2PC) protocol, allowing multiple separate databases or resources to participate in a single atomic transaction.

## What is Two-Phase Commit (2PC)?
The 2PC protocol ensures atomicity across distributed systems:
- **Phase 1 (Prepare)**: Transaction manager asks all participants to prepare (validate and lock resources)
- **Phase 2 (Commit/Rollback)**: If all participants are ready, commit; otherwise, rollback all

![How Row Locking Works](https://github.com/PiyushMittl/Others/blob/main/db-distributedtransation-blog/XAtransactions-blog/images/1.xa-2pc-protocol.png)

*Figure 1: Two-Phase Commit Protocol - Transaction Manager coordinates with multiple database participants*

## Understanding Row Locking in XA Transactions

![How Row Locking Works](https://github.com/PiyushMittl/Others/blob/main/db-distributedtransation-blog/XAtransactions-blog/images/2.xa-understanding-xa.png)

*Figure 2: Row Locking Example - Two transactions (T1 and T2) competing for the same row with lock wait behavior*

<details>
<summary>📊 Click here if diagram above is not visible (Text Version)</summary>

**Key Points:**
- 🔒 = Row Lock Active
- ⏳ = Transaction Waiting
- ✅ = Lock Released
- T2 is blocked from t5 to t7 (during T1's PREPARE and COMMIT phases)

</details>

### How Row Locking Works - Example with Two Transactions

When multiple transactions try to modify the same row, databases use **row-level locking** to ensure data consistency. Here's how it works:

#### Key Concepts:
1. **Uncommitted Versions**: Each transaction maintains its own version of modified data
2. **Exclusive Lock**: When a transaction updates a row, it acquires an exclusive lock that prevents others from modifying it
3. **Read Committed Isolation**: Other transactions see only the last committed value, not uncommitted changes
4. **Lock Wait**: Transactions wait for locks to be released before proceeding

#### Why This Matters in XA Transactions:

In distributed XA transactions, these locks are held **until XA COMMIT or XA ROLLBACK**, which means:
- **Longer lock duration**: From XA PREPARE to XA COMMIT (two phases)
- **Higher risk of deadlocks**: Multiple resources waiting on each other
- **Performance impact**: Other transactions may wait longer for locks to be released
- **Resource contention**: Critical in high-concurrency scenarios

#### Isolation Levels:
The example assumes **READ COMMITTED** isolation level. Other levels (READ UNCOMMITTED, REPEATABLE READ, SERIALIZABLE) may change read behavior, but locking concepts remain the same.

## Common Use Cases
- **Multi-database transactions**: Updating MySQL and PostgreSQL databases in a single transaction
- **Microservices**: Maintaining data consistency across multiple services
- **Banking systems**: Ensuring atomic money transfers across accounts in different databases
- **E-commerce**: Order processing with inventory, payment, and shipping systems


## MySQL XA Transactions
### Key Features:
- Implements two-phase commit protocol
- Allows coordination of transactions across multiple databases
- XID format: `gtrid[,bqual[,formatID]]` where gtrid is global transaction ID

### Important Limitations:
- XA transactions cannot use `SAVEPOINT`
- Cannot be used with replication filters
- Temporary tables are not supported in XA transactions
- Locks are held until `XA COMMIT` or `XA ROLLBACK`


### Basic Syntax:
```sql
-- Start XA transaction
XA START 'xid';

-- Execute operations
INSERT INTO table VALUES (...);
UPDATE table SET ...;

-- End XA transaction
XA END 'xid';

-- Prepare transaction (phase 1)
XA PREPARE 'xid';

-- Commit transaction (phase 2)
XA COMMIT 'xid';

-- Or rollback
XA ROLLBACK 'xid';

-- One-phase commit (for single resource optimization)
XA COMMIT 'xid' ONE PHASE;
```

### Example:
```sql
-- 1. XA START - No locks yet
XA START 'xid_001';

-- 2. SQL Operations - LOCKS ACQUIRED HERE ⬅️
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Row lock on accounts table, row id=1 is now LOCKED

SELECT * FROM orders WHERE account_id = 1 FOR UPDATE;
-- Additional locks on orders table rows

-- 3. XA END - LOCKS STILL HELD ⬅️
XA END 'xid_001';
-- Transaction boundaries marked, but locks NOT released

-- 4. XA PREPARE - LOCKS STILL HELD ⬅️
XA PREPARE 'xid_001';
-- Transaction prepared, written to binary log, but locks STILL held

-- 5. XA COMMIT - LOCKS RELEASED HERE ⬅️
XA COMMIT 'xid_001';
-- Finally, all locks are released
```

### Practical Example:

![How Row Locking Works](https://github.com/PiyushMittl/Others/blob/main/db-distributedtransation-blog/XAtransactions-blog/images/3.xa-completedexample.png)

#### Example Scenario (Initial Balance = 1000):
**Step-by-Step Flow:**

1. **Transaction T1 Begins**: `UPDATE accounts SET balance = balance - 100 WHERE id = 1;`
   - T1's internal value: 900 (uncommitted)
   - Row lock acquired by T1
   - Other transactions cannot update this row

2. **Transaction T2 Attempts to Read**: `SELECT balance FROM accounts WHERE id = 1;`
   - T2 sees: 1000 (last committed value)
   - T2 does NOT see T1's uncommitted change (900)

3. **Transaction T2 Attempts to Update**: `UPDATE accounts SET balance = balance - 50 WHERE id = 1;`
   - **T2 is BLOCKED** - waiting for T1 to commit or rollback
   - T2 enters a wait state

4. **Transaction T1 Commits**: 
   - New committed value: 900
   - Lock released
   - T1's changes are now visible to all transactions

5. **Transaction T2 Resumes**:
   - T2 now works with the latest committed value (900)
   - T2 calculates: 900 - 50 = 850
   - T2 acquires the lock and creates uncommitted version (850)

6. **Transaction T2 Commits**:
   - Final committed balance: 850
   - Lock released


**Scenario: Two transactions updating the same account (Initial balance = 1000)**

**Transaction T1: Deduct 100 from account**
```sql
-- Terminal/Connection 1
XA START 'trx_t1_001';

UPDATE accounts 
SET balance = balance - 100 
WHERE id = 1;

XA END 'trx_t1_001';

-- Prepare phase (locks are held)
XA PREPARE 'trx_t1_001';
```

**Transaction T2: Deduct 50 from same account (runs concurrently)**
```sql
-- Terminal/Connection 2
XA START 'trx_t2_001';

-- This UPDATE will WAIT if T1 hasn't committed yet
-- T2 is blocked until T1 releases the lock
UPDATE accounts 
SET balance = balance - 50 
WHERE id = 1;

XA END 'trx_t2_001';

-- Prepare phase
XA PREPARE 'trx_t2_001';
```

**Commit both transactions (coordinated by transaction manager)**
```sql
-- Commit T1 (lock released, T2 can now proceed)
XA COMMIT 'trx_t1_001';

-- Commit T2
XA COMMIT 'trx_t2_001';
```

**Verify final balance**
```sql
SELECT id, balance FROM accounts WHERE id = 1;
-- Result: balance = 850 (1000 - 100 - 50)
```

**Example: Distributed transaction across two databases**

![How Row Locking Works](https://github.com/PiyushMittl/Others/blob/main/db-distributedtransation-blog/XAtransactions-blog/images/4.xa-tm-distributed.png)
*Figure 3: Distributed XA Transaction - Money transfer coordinated across two separate databases*

**Database 1 - Transfer money out**
```sql
XA START 'transfer_001', 'db1', 1;

UPDATE accounts 
SET balance = balance - 100 
WHERE id = 1;

XA END 'transfer_001', 'db1', 1;
XA PREPARE 'transfer_001', 'db1', 1;
```

**Database 2 - Transfer money in**
```sql
XA START 'transfer_001', 'db2', 1;

UPDATE accounts 
SET balance = balance + 100 
WHERE id = 2;

XA END 'transfer_001', 'db2', 1;
XA PREPARE 'transfer_001', 'db2', 1;
```

**Commit or Rollback both (atomicity guaranteed)**

![How Row Locking Works](https://github.com/PiyushMittl/Others/blob/main/db-distributedtransation-blog/XAtransactions-blog/images/5.xa-mysql-xa-lifecycle.png)
*Figure 4: MySQL XA Lock Lifecycle - Locks acquired during SQL operations and held until XA COMMIT*

**Understanding when locks are acquired and released**

```sql
-- 1. XA START - No locks yet
XA START 'xid_001';

-- 2. SQL Operations - LOCKS ACQUIRED HERE ⬅️
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Row lock on accounts table, row id=1 is now LOCKED

SELECT * FROM orders WHERE account_id = 1 FOR UPDATE;
-- Additional locks on orders table rows

-- 3. XA END - LOCKS STILL HELD ⬅️
XA END 'xid_001';
-- Transaction boundaries marked, but locks NOT released

-- 4. XA PREPARE - LOCKS STILL HELD ⬅️
XA PREPARE 'xid_001';
-- Transaction prepared, written to binary log, but locks STILL held

-- 5. XA COMMIT - LOCKS RELEASED HERE ⬅️
XA COMMIT 'xid_001';
-- Finally, all locks are released
```

**Key Points:**
- Locks are held from the first SQL operation until `XA COMMIT` or `XA ROLLBACK`
- Between `XA PREPARE` and `XA COMMIT`, locks remain active (critical period)
- Other transactions trying to access locked rows will wait during this entire period
- This is why XA transactions can impact performance in high-concurrency scenarios

### Recovery Commands:
```sql
-- List prepared XA transactions
XA RECOVER;

-- Format with details
XA RECOVER FORMAT='SQL';
```



## PostgreSQL XA Transactions
### Key Features:
- Supported via PREPARE TRANSACTION command
- More limited compared to full XA implementation
- Prepared transactions survive server crashes
- Each prepared transaction consumes resources until resolved

### Important Limitations:
- Cannot use `PREPARE TRANSACTION` inside a function
- Must resolve prepared transactions promptly to avoid resource locks
- No automatic cleanup of orphaned prepared transactions

### Basic Syntax:
```sql
-- Start transaction
BEGIN;

-- Execute operations
INSERT INTO table VALUES (...);
UPDATE table SET ...;

-- Prepare transaction (phase 1)
PREPARE TRANSACTION 'transaction_id';

-- Later, commit (phase 2)
COMMIT PREPARED 'transaction_id';

-- Or rollback
ROLLBACK PREPARED 'transaction_id';
```

### Practical Example:

**Scenario: Simple account balance update (Initial balance = 1000)**

**Transaction T1: Deduct 100 from account**
```sql
-- Terminal/Connection 1
BEGIN;

UPDATE accounts 
SET balance = balance - 100 
WHERE id = 1;

-- Prepare transaction (phase 1)
PREPARE TRANSACTION 'trx_t1_pg_001';
```

**Transaction T2: Deduct 50 from same account**
```sql
-- Terminal/Connection 2
BEGIN;

-- This UPDATE will WAIT if T1 hasn't been committed
UPDATE accounts 
SET balance = balance - 50 
WHERE id = 1;

-- Prepare transaction (phase 1)
PREPARE TRANSACTION 'trx_t2_pg_001';
```

**Commit both transactions (coordinated by transaction manager)**
```sql
-- Commit T1 (lock released)
COMMIT PREPARED 'trx_t1_pg_001';

-- Commit T2
COMMIT PREPARED 'trx_t2_pg_001';
```

**Verify final balance**
```sql
SELECT id, balance FROM accounts WHERE id = 1;
-- Result: balance = 850 (1000 - 100 - 50)
```

**Example: Handling a failed transaction with rollback**

**Start a transaction**
```sql
BEGIN;

UPDATE accounts 
SET balance = balance - 200 
WHERE id = 1;

-- Prepare transaction
PREPARE TRANSACTION 'trx_rollback_test';
```

**Check prepared transactions**
```sql
SELECT gid, prepared, owner, database 
FROM pg_prepared_xacts 
WHERE gid = 'trx_rollback_test';
```

**Decide to rollback instead of commit**
```sql
-- Something went wrong, rollback the prepared transaction
ROLLBACK PREPARED 'trx_rollback_test';

-- Balance remains unchanged
SELECT id, balance FROM accounts WHERE id = 1;
```

### Lock Lifecycle in PostgreSQL Prepared Transactions:

![How Row Locking Works](https://github.com/PiyushMittl/Others/blob/main/db-distributedtransation-blog/XAtransactions-blog/images/6.xa-pg-prepared-lifecycle.png)
*Figure 5: PostgreSQL Prepared Transaction Lock Lifecycle - Locks persist even after connection closes*

**Understanding when locks are acquired and released**

```sql
-- 1. BEGIN - No locks yet
BEGIN;

-- 2. SQL Operations - LOCKS ACQUIRED HERE ⬅️
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Row lock on accounts table, row id=1 is now LOCKED

SELECT * FROM orders WHERE account_id = 1 FOR UPDATE;
-- Additional locks on orders table rows

-- 3. PREPARE TRANSACTION - LOCKS STILL HELD ⬅️
PREPARE TRANSACTION 'trx_pg_001';
-- Transaction prepared, persisted to disk, but locks STILL held
-- Connection can now be closed, but locks persist!

-- 4. COMMIT PREPARED - LOCKS RELEASED HERE ⬅️
COMMIT PREPARED 'trx_pg_001';
-- Finally, all locks are released

-- Alternative: ROLLBACK PREPARED - LOCKS RELEASED HERE ⬅️
-- ROLLBACK PREPARED 'trx_pg_001';
-- Locks released, changes discarded
```

**Key Points:**
- Locks are held from the first SQL operation until `COMMIT PREPARED` or `ROLLBACK PREPARED`
- After `PREPARE TRANSACTION`, locks survive even if the database connection is closed
- Prepared transactions can persist across server restarts (locks remain!)
- This is why it's critical to resolve prepared transactions promptly
- Orphaned prepared transactions can cause serious blocking issues

### Configuration:
```sql
-- Check current setting
SHOW max_prepared_transactions;

-- In postgresql.conf, set:
-- max_prepared_transactions = 100  # or desired value
-- Requires server restart
```

### Recovery Commands:
```sql
-- List prepared transactions
SELECT * FROM pg_prepared_xacts;

-- View details
SELECT gid, prepared, owner, database FROM pg_prepared_xacts;
```

## Comparison

| Feature | MySQL | PostgreSQL |
|---------|-------|------------|
| XA Standard | Full XA support | Partial (PREPARE TRANSACTION) |
| Configuration | Enabled by default | Requires `max_prepared_transactions` |
| XID Format | gtrid[,bqual[,formatID]] | Simple string identifier |
| Recovery View | `XA RECOVER` | `pg_prepared_xacts` |
| Performance Impact | Moderate overhead | Higher overhead, locks held longer |

## Best Practices
1. **Always have timeout mechanisms**: Prevent hanging transactions
2. **Monitor prepared transactions**: Use recovery views to detect orphaned transactions
3. **Use transaction managers**: Leverage JTA (Java) or similar frameworks rather than manual XA
4. **Keep transactions short**: Minimize lock duration to avoid blocking
5. **Test failure scenarios**: Ensure proper handling of network failures, crashes, and timeouts
6. **Resource cleanup**: Implement automated cleanup for orphaned prepared transactions
7. **Consider alternatives**: Eventual consistency patterns (Saga, Event Sourcing) may be more suitable for some use cases

## Transaction Manager Integration
Both databases typically work with transaction managers like:
- **Java**: JTA (Java Transaction API), Atomikos, Narayana
- **Spring**: `@Transactional` with JTA
- **.NET**: System.Transactions, DTС (Distributed Transaction Coordinator)
- **Python**: Various XA-capable libraries and frameworks

## When NOT to Use XA Transactions
- **High throughput systems**: 2PC adds significant latency
- **Microservices at scale**: Consider eventual consistency patterns instead
- **Simple use cases**: Single-database transactions are more efficient
- **Long-running processes**: Risk of deadlocks and resource exhaustion

## Further Reading
- X/Open XA Specification
- MySQL Documentation: XA Transactions
- PostgreSQL Documentation: Two-Phase Transactions
- Patterns for distributed transactions: Saga, Event Sourcing, CQRS
