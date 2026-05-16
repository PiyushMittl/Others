XA Transactions in MySQL and PostgreSQL

XA Transactions are distributed transactions that follow the X/Open XA standard for two-phase commit (2PC) protocol, allowing multiple separate databases or resources to participate in a single atomic transaction.


MySQL XA Transactions
Key Features:
•	Supported in InnoDB storage engine (from MySQL 5.0.3+)
•	Implements two-phase commit protocol
•	Allows coordination of transactions across multiple databases


Basic Syntax:
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


Recovery Commands:
-- List prepared XA transactions
XA RECOVER;



PostgreSQL XA Transactions
Key Features:
•	Supported via PREPARE TRANSACTION command
•	Must enable max_prepared_transactions in config
•	More limited compared to full XA implementation
Basic Syntax:

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

Both are typically used with transaction managers (like Java's JTA) for distributed transactions across multiple databases or resources.
