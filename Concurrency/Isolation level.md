
# Transactions
Imagine you're transferring ₹1,000 from Alice to Bob.

You need to do two things:

1. Subtract ₹1,000 from Alice.
2. Add ₹1,000 to Bob.

Without a transaction, this could happen:
```
Alice: ₹5,000
Bob:   ₹2,000

Subtract ₹1,000 from Alice
→ Alice: ₹4,000

💥 Server crashes

Add ₹1,000 to Bob never happens
→ Bob: ₹2,000

```
- Now ₹1,000 has disappeared. That's obviously bad.
- This is where transactions come into the picture.
- The database guarantees that the changes are done together and never a half-finished transfer.

## What is a database transaction?
A transaction is a collection of one or more SQL queries treated as a single, indivisible unit of work.

```sql
BEGIN;
  -- Step 1: Check balance
  SELECT balance FROM accounts WHERE account_id = 1;

  -- Step 2: Debit sender
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;

  -- Step 3: Credit receiver
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;
```

### Transaction Lifecycle
Normally
```
BEGIN
  ↓
READ / WRITE
  ↓
READ / WRITE
  ↓
COMMIT

```
If something goes wrong:
```
BEGIN
  ↓
READ / WRITE
  ↓
ERROR
  ↓
ROLLBACK
```


### Core Transaction Control Commands:
- `BEGIN / START TRANSACTION`: Explicitly defines the start of a transaction block.
- `COMMIT`: Tells the database engine to permanently apply and persist all changes made during the transaction.
- `ROLLBACK / ABORT`: Cancels the entire transaction, discarding all uncommitted in-memory or dirty changes and returning data to its initial state.


## ACID properties
In order to ensure transactions are reliable and predictable and execute correctly, databases provide ACID properties:
1. Atomicity
2. Consistency
3. Isolation
4. Durability


# 1. Atomicity
Atomicity ensures that all queries within a transaction either completely succeed or completely fail.

```
Normal Flow:
[Debit Account 1] ──► [Credit Account 2] ──► [COMMIT] (Success)

Failure Scenario (Without Atomicity):
[Debit Account 1] ──► [System Crash / DB Error] ──► $100 Vanishes Into Thin Air!

Failure Scenario (With Atomicity):
[Debit Account 1] ──► [System Crash / DB Error] ──► [Automatic ROLLBACK] (Balance Restored)
```

- The Problem It Solves: If a database crashes midway through transferring money between accounts, a partial update would debit Account 1 without crediting Account 2.
- The Guarantee: If any query fails or a crash occurs before COMMIT, the database engine traverses its undo mechanism to revert all partial modifications made by that transaction.


# 2. Isolation
- Two transactions can run concurrently. Concurrency is desirable as it increases the throughput of the system. There is no compromise on this one. 
- Since two transactions are running concurrently and can work on the same data, they may run into race conditions, leading to interfering with each others work and producing wrong results.
- So, in order to ensure they do not interfere with each other, we need to have some isolation between the two transactions, so that they execute correctly.
- Isolation in ACID means that two concurrent transactions should not interfere with each others operations, ensuring no race conditions.
- Without isolation, we end up with certain issues because of concurrent execution of transactions.

## Issues with concurrency of transactions

1. Dirty Reads - T1 reads a value written by T2, before T2 commits.
2. Lost update problem (write-write problem) - T1 and T2 both read the same value, both compute a new value based on it, and one write overwrites the other without either seeing the other's change. A lost update happens during an application-level `read-modify-write` cycle when two transactions read the same data at the same time and both try to update it based on what they saw.
3. Non-repeatable read problem - T1 reads a row twice, gets different values, because T2 committed a change to that row in between 
4. Phantom read problem - T1 runs the same range query twice, gets a different set of rows, because T2 inserted/deleted rows matching the predicate.
5. Write Skew
6. Dirty Writes

### 5. Write Skew
- Write skew occurs when two concurrent transactions read the same data, check a business rule, and then update different rows. Because they modify separate records, both transactions commit without a write conflict—but their combined changes violate the business rule.

#### Example: Doctors on call
- Business requirement: At least one doctor must always be on call.
- Initial state: Both Alice and Bob are on call. Both of them want to check if another one is on-call, so that he/she can go off-call.

|Time|Transaction 1 (Alice leaves)                            |Transaction 2 (Bob leaves)                            |What Happens                                |
|----|--------------------------------------------------------|------------------------------------------------------|--------------------------------------------|
|t1​ |SELECT count(*) WHERE on_call = true;                   |—                                                     |Alice sees 2 doctors on call. Safe to leave.|
|t2​ |—                                                       |SELECT count(*) WHERE on_call = true;                 |Bob sees 2 doctors on call. Safe to leave.  |
|t3​ |UPDATE doctors SET on_call = false WHERE name = 'Alice';|—                                                     |Alice modifies Row A.                       |
|t4​ |—                                                       |UPDATE doctors SET on_call = false WHERE name = 'Bob';|Bob modifies Row B.                         |
|t5​ |COMMIT;                                                 |COMMIT;                                               |Both succeed.                               |

- Result: 0 doctors are on call. The business rule is broken.
- The fundamental difference between lost update problem and write skew is that, in lost update, two transactions end up overwriting the same row data, while in write skew, two transactions update different rows in the end.
- How to prevent write skew:
  - Alice must lock all the rows involved first before she makes any change. So that bob cannot read-modify-write at the same time. She must acquire exclusive lock for that.
     - `SELECT * FROM doctors WHERE on_call = true FOR UPDATE;`
  - Use serializable isolation level.  


### Dirty Writes vs Lost Updates
### 6. Dirty Writes (Overwriting Uncommitted Data)
- Let's consider two transactions T1 and T2.
- Theres a value X = 100.
- T1 wants to update it to 110 and T2 want to update it to 120.
- T1 updates x to 110. Not committed yet.
- T2 runs and updates X to 120 and commits.
- T1 resumes. It can either commit or fail and rollback.
  - If it commits: T1 will commit thinking that it wrote the value 110. This leads to consistency & durability violation because T1 was the last transaction to commit, yet its value was not persisted.
  - If it rollbacks, the value of X would be 100, as that was the value of X when the T1 started. So x = 100, completely erases T2's work, again violating durability even after T2 committed. 

- How databases handle this?
  - All relational database engines (PostgreSQL, MySQL, Oracle, SQL Server) prevent Dirty Writes at every isolation level (including READ UNCOMMITTED) by requiring exclusive row-level locks on writes until transaction completion.

### Lost Updates (Overwriting Committed Data via Stale Reads)
- A lost update happens when two transactions read the exact same committed data, calculate a new value locally in application memory, and then write their updates back to the database. The second write clobbers the first write, making it as if the first transaction never happened.

```
Time    Transaction A (T1)                     Transaction B (T2)
────────────────────────────────────────────────────────────────────────────
T1      SELECT balance FROM accounts;          
        (Reads balance = $100)
T2                                             SELECT balance FROM accounts;
                                               (Reads balance = $100)
T3      -- App calculates: 100 + 50 = 150      -- App calculates: 100 + 20 = 120
T4      UPDATE accounts SET balance = 150;     
T5      COMMIT;                                
        (T1's write is successfully committed)
T6                                             UPDATE accounts SET balance = 120;
                                               (Overwrites T1's committed $150!)
T7                                             COMMIT;

```
- Both transactions committed successfully, but T1's deposit of $\$50$ was completely erased because T2 operated on a stale snapshot.
- How databases handle this:
  - Atomic SQL updates: `UPDATE accounts SET balance = balance + 50 WHERE id = 1;`
  - Optimistic and Pessimistic locking.

In order to get rid of the issues due to concurrency, we have levels of isolation for transactions.

## Isolation levels
1. Read Uncommitted
2. Read Committed
3. Repeatable Read
4. Serializable

### Implementation Paradigms
- Databases implement/enforce the above isolation levels in two different ways:
  - Pessimistic Locking(2PL)
  - MVCC  
- Note: The standard isolation levels was designed with only Locking in mind. MVCC came later which tried to adhere to the standard in a different way.

## 1. Read Uncommitted
- No isolation whatsoever. A transaction can read uncommitted changes made by some other concurrent transaction.
- Because there is no isolation, so we have all the issues present at this level.
- Implementation:
  - Locking: Read queries do not acquire any lock. Only writers acquire exclusive locks till the end of transaction to prevent dirty writes. Exclusive locks for writes are a given thing. No matter which isolation level. The database must not allow two writers to write on the same row concurrently to prevent data corruption.
  - MVCC: MVCC doesn't even implement this level. Reads simply use latest data regardless of commit status (Postgres treats this as Read Committed).  

## Read Committed
- In this isolation level, a transaction only reads data that was committed before the individual statement began.
- Implementation:
  - Locking: Before a read statement starts executing, a shared lock is acquired by it and released immediately (query-level lock, not transaction level) after the `SELECT` statement finishes (not held till commit of transaction). Exclusive locks are still held by writers and is held till end of transaction. 
  - MVCC: Generates a fresh snapshot of the entire database at the start of every SQL statement. Snapshot is discarded once statement finishes. If $T_2$ updates and commits between Query 1 and Query 2 in $T_1$, Query 2 generates a fresh snapshot and sees $T_2$'s commit. (query-level snapshot, not transaction level)
- This fixes the dirty reads problem.
- This doesn't fix non-repeatable read problem because of the nature of the non-repeatable read itself. Even if another transaction commits, the reading transaction will get different values for the same query if the update and commit happened between two reads.
```sql
-- T1                                  -- T2
BEGIN;
SELECT balance FROM accounts
  WHERE id = 1;        -- returns 100

                                        BEGIN;
                                        UPDATE accounts SET balance = 50
                                          WHERE id = 1;
                                        COMMIT;

SELECT balance FROM accounts
  WHERE id = 1;        -- returns 50, different from first read!
COMMIT;
```

- Taking a fresh snapshot for each statement is the exact reason Phantom Reads happen in `READ COMMITTED`.
- This level also allows lost update problem.
```sql
-- T1 (web request A: withdraw 30)      -- T2 (web request B: withdraw 20)
BEGIN;                                  BEGIN;
SELECT balance FROM accounts
  WHERE id=1;  -- reads 100
                                         SELECT balance FROM accounts
                                           WHERE id=1;  -- reads 100
UPDATE accounts SET balance = 70
  WHERE id=1;  -- 100 - 30
COMMIT;
                                         UPDATE accounts SET balance = 80
                                           WHERE id=1;  -- 100 - 20 (based on stale read!)
                                         COMMIT;
-- Final balance: 80. Should be 50 (100 - 30 - 20). T1's withdrawal is "lost".
```
  - Fix at Read Committed level: use `UPDATE accounts SET balance = balance - 30 WHERE id=1` (atomic, single-statement — the DB serializes the read+write internally), or `SELECT ... FOR UPDATE` to take a row lock before computing the new value.
- **Anomalies Solved**: Eliminates Dirty Reads
- **Remaining Anomalies**: Non-Repeatable Reads, Lost Updates, Phantom Reads, Write Skew.



## Repeatable Read
- This isolation level guarantees that a transaction will see the same data throughout its execution. 
- So, if a transaction reads a row once, every subsequent read of that same row within the transaction will return the exact same data.
- Implementation:
  - Locking: Shared lock is taken incrementally as each row is read, held until end of transaction. So, we get same data for each row throughout the transaction. Exclusive lock on writes are held till end of transaction to prevent dirty writes as usual. Any concurrent `UPDATE/DELETE` trying to acquire an $X$-lock is blocked. (transaction-level locks, not query-level locks)
  - MVCC: A snapshot of the entire database is taken at the beginning of the transaction and this snapshot is used throughout the transaction. Please note: only committed version of each rows are part of this snapshot, uncommitted versions not allowed. (transaction-level snapshot, not query-level snapshot)
- This prevents dirty reads because the transaction only sees committed values, and eliminates non-repeatable reads because it guarantees the entire transaction sees the same thing throughout.
- This level also eliminates lost updates.
  - Locking: $T_1$ and $T_2$ both hold $S$-locks. When either attempts to upgrade to an $X$-lock to perform the write, a deadlock is triggered and the engine terminates one transaction. 
  - MVCC: When a transaction wants to update a row, it will first check if that row's committed version is the same as the transactions snapshot version or not. If someone else committed a change to that exact row in the meantime, that means the current transaction might overwrite data committed by that other transaction - leading to lost update problem. So, MVCC aborts the current transaction.
  ```
    How MVCC Prevents Lost Updates (Repeatable Read)

    1. Write-First Protection (Row Locks):
       - Even in MVCC, write statements (`UPDATE`, `DELETE`) immediately acquire row-level Exclusive (X) locks held until transaction termination (`COMMIT`/`ROLLBACK`).
       - If T1 writes first, any concurrent write by T2 on that row is blocked, ensuring T1's in-flight work cannot be overwritten.
    
    2. Read-First Protection (First-Committer-Wins):
       - If T1 only reads a row (holding no locks) and T2 commits an update to that row in the meantime, T1's snapshot view becomes stale.
       - When T1 eventually issues its `UPDATE`, the engine inspects the row's commit metadata against T1's snapshot.
       - Detecting that the row was modified after the snapshot began, the engine aborts T1 immediately with a serialization error instead of overwriting T2's committed data.
  ```    


|Step|Transaction 1 (T1​)                            |Transaction 2 (T2​)                            |Engine Action & Row State                                                                                       |
|----|-----------------------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------|
|t1​ |BEGIN;                                         |BEGIN;                                         |Both transactions start with their own snapshots.                                                               |
|t2​ |UPDATE accounts SET balance = 150 WHERE id = 1;|—                                              |T1​ acquires an Exclusive (X) lock on row 1. Value in RAM: 150 (uncommitted).                                   |
|t3​ |—                                              |UPDATE accounts SET balance = 120 WHERE id = 1;|T2​ is blocked. The lock manager suspends T2​ waiting for T1​'s X-lock.                                         |
|t4​ |COMMIT;                                        |(waiting...)                                   |T1​ commits. Lock on row 1 is released. Value in DB: 150.                                                       |
|t5​ |—                                              |(unblocks, evaluates row)                      |T2​ wakes up, detects row was updated by T1​ after its snapshot, and aborts with serialization error (Postgres).|


|Step|Transaction 1 (T1​)                            |Transaction 2 (T2​)                            |Engine Action & Row State                                                                                       |
|----|-----------------------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------|
|t1​ |BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;|—                                              |Snapshot 1 established.                                                                                         |
|t2​ |SELECT balance FROM accounts WHERE id = 1;(Reads 100, holds no locks)|BEGIN;                                         |T1​ calculates in application memory (100 + 50 = 150).                                                          |
|t3​ |(app thinking in background...)                |UPDATE accounts SET balance = 120 WHERE id = 1;COMMIT;|T2​ updates and commits successfully. New disk version created with xmin = T2. Committed balance: 120.          |
|t4​ |UPDATE accounts SET balance = 150 WHERE id = 1;|—                                              |T1​ fails immediately. Engine reads row header, sees xmin = T2 committed after T1​'s snapshot, and aborts T1​:ERROR: could not serialize access due to concurrent update.|
|t5​ |COMMIT;                                        |—                                              |Never reached. Transaction was marked aborted at t4​.                                                           |


- The standard says phantom reads **can** exist in repeatable reads because it considered lock based implementation. But when MVCC came, because of its snapshot nature, each transaction sees only those rows that were part of the snapshot. Any rows that were inserted/delete later by some other transaction are not part of the snapshot. So, MVCC ends up naturally over-delivering and eliminates phantom reads at repeatable read isolation level only.
- Remaining Issues: Write Skew

|Time|Transaction 1 (Generate Report)     |Transaction 2 (Price Update)                      |READ COMMITTED        |REPEATABLE READ       |
|----|------------------------------------|--------------------------------------------------|----------------------|----------------------|
|t1​ |SELECT price FROM items WHERE id = 1|—                                                 |Sees $100             |Sees $100             |
|t2​ |—                                   |UPDATE items SET price = 150 WHERE id = 1; COMMIT;|Committed to DB       |Committed to DB       |
|t3​ |SELECT price FROM items WHERE id = 1|—                                                 |Sees $150 (Fuzzy Read)|Sees $100 (Consistent)|
|t4​ |COMMIT;                             |—                                                 |—                     |—                     |

## Serializable
- This is the highest isolation level.
- It guarantees that the outcome of executing concurrent transactions is mathematically equivalent to executing them one at a time - sequentially/serially.
- It eliminates all the concurrency issues.
