
# Isolation
- Isolation in ACID means that two concurrent transactions should not interfere with each others operations, ensuring no race conditions.
- Without isolation, we end up with certain issues because of concurrent execution of transactions.

## Issues with concurrency of transactions
1. Dirty Reads - T1 reads a value written by T2, before T2 commits.
2. Lost update problem (write-write problem) - T1 and T2 both read the same value, both compute a new value based on it, and one write overwrites the other without either seeing the other's change. A lost update happens during an application-level `read-modify-write` cycle when two transactions read the same data at the same time and both try to update it based on what they saw.
3. Non-repeatable read problem - T1 reads a row twice, gets different values, because T2 committed a change to that row in between 
4. Phantom read problem - T1 runs the same range query twice, gets a different set of rows, because T2 inserted/deleted rows matching the predicate.

Before we learn about how to eliminate these issues, lets first understand dirty writes.
## Dirty Writes vs Lost Updates
### Dirty Writes (Overwriting Uncommitted Data)
- Let's consider two transactions T1 and T2.
- Theres a value X = 100.
- T1 wants to update it to 110 and T2 want to update it to 120.
- T1 updates x to 110. Not committed yet.
- T2 runs and updates X to 120 and committs.
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

In order to get rid of the issues, we have certain levels of isolation.
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
  - Locking: Before a read statement starts executing, a shared lock is acquired by it and released immediately after the `SELECT` statement finishes (not held till commit of transaction). Exclusive locks are still held by writers and is held till end of transaction.
  - MVCC: Generates a fresh snapshot of the entire database at the start of every SQL statement. Snapshot is discarded once statement finishes. If $T_2$ updates and commits between Query 1 and Query 2 in $T_1$, Query 2 generates a fresh snapshot and sees $T_2$'s commit.
- This fixes the dirty reads problem.
- This doesn't fix non-repeatable read problem because of the nature of the non-repeatable read itself. Even if another transaction commits, the reading transaction will get different values for the same query if the update and commit happened between two reads.
- Taking a fresh snapshot for each statement is the exact reason Phantom Reads happen in `READ COMMITTED`.
- This level also allows lost update problem.
- **Anomalies Solved**: Eliminates Dirty Reads
- **Remaining Anomalies**: Non-Repeatable Reads, Lost Updates, Phantom Reads, Write Skew.

## Repeatable Read
- This isolation level guarantees that a transaction will see the same data throughout its execution. 
- So, if a transaction reads a row once, every subsequent read of that same row within the transaction will return the exact same data.
- Implementation:
  - Locking: Shared lock is taken incrementally as each row is read, held until end of transaction. So, we get same data for each row throughout the transaction. Exclusive lock on writes are held till end of transaction to prevent dirty writes as usual. Any concurrent `UPDATE/DELETE` trying to acquire an $X$-lock is blocked. 
  - MVCC: A snapshot of the entire database is taken at the beginning of the transaction and this snapshot is used throughout the transaction. Please note: only committed version of each rows are part of this snapshot, uncommitted versions not allowed.
- This prevents dirty reads because the transaction only sees committed values, and eliminates non-repeatable reads because it guarantees the entire transaction sees the same thing throughout.
- This level also eliminates lost updates.
  - Locking: $T_1$ and $T_2$ both hold $S$-locks. When either attempts to upgrade to an $X$-lock to perform the write, a deadlock is triggered and the engine terminates one transaction. 
  - MVCC: When a transaction wants to update a row, it will first check if that row's committed version is the same as the transactions snapshot version or not. If someone else committed a change to that exact row in the meantime, that means the current transaction might overwrite data committed by that other transaction - leading to lost update problem. So, MVCC aborts the current transaction.
  ```
  Time   Transaction A (Snapshot @ T1)              Transaction B (Snapshot @ T1)
  ────────────────────────────────────────────────────────────────────────────────────────────
  T1     BEGIN TRANSACTION ISOLATION                BEGIN TRANSACTION ISOLATION 
         LEVEL REPEATABLE READ;                     LEVEL REPEATABLE READ;
  
  T2     SELECT balance FROM accounts               SELECT balance FROM accounts 
         WHERE id = 1;                              WHERE id = 1;
         (Reads 100)                                (Reads 100)
  
  T3     UPDATE accounts SET balance = 150 
         WHERE id = 1;
         (Acquires X-Lock on row 1)
  
  T4                                                UPDATE accounts SET balance = 120 
                                                    WHERE id = 1;
                                                    [BLOCKED - Waiting for Txn A's lock]
  
  T5     COMMIT; 
         (Txn A's 150 committed successfully)
  
  T6                                                [Txn B Unblocks & Evaluates Row State]
                                                    ERROR: could not serialize access 
                                                    due to concurrent update
                                                    (Txn B is ABORTED by Postgres!)
  ```
  Here, transaction B at T6 checks whether row id = 1 was modified by some other transaction after its snapshot and aborts.
  
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
