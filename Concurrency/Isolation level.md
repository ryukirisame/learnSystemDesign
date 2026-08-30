
# Isolation
- Isolation in ACID means that two concurrent transactions should not interfere with each others operations.
- Without isolation, we end up with certain issues because of concurrent execution of transactions.

## Issues with concurrency of transactions
1. Dirty Reads - T1 reads a value written by T2, before T2 commits.
2. Lost update problem (write-write problem) - T1 and T2 both read the same value, both compute a new value based on it, and one write overwrites the other without either seeing the other's change
3. Non-repeatable read problem - T1 reads a row twice, gets different values, because T2 committed a change to that row in between 
4. Phantom read problem - T1 runs the same range query twice, gets a different set of rows, because T2 inserted/deleted rows matching the predicate.

In order to get rid of these issues, we have certain levels of isolation.
## Isolation levels
1. Read Uncommitted
2. Read Committed
3. Repeatable Read
4. Serializable

## Read Uncommitted
- No isolation whatsoever. A transaction can read uncommitted changes made by some other concurrent transaction.
- Because there is no isolation, so we have all the issues present at this level.

## Read Committed
- This isolation level allows only committed changes to be read by a transaction.
- This fixes the dirty reads problem.
- This doesn't fix non-repeatable read problem because of the nature of the non-repeatable read itself. Even if another transaction commits, the reading transaction will get different values for the same query if the update and commit happened between two reads.
- This level also allows phantom reads and lost update problems.

## Repeatable Read
- This isolation level guarantees that a transaction will see the same data throughout its execution, even if other transactions commit changes to the data.
- So, if a transaction reads a row once, every subsequent read of that same row within the transaction will return the exact same data, even if concurrent transactions modify and commit changes to it in the meantime.
- This prevents dirty reads because the transaction only sees committed values, and eliminates non-repeatable reads because it guarantees the entire transaction sees the same thing throughout.
- However, phantom reads are still possible.

|Time|Transaction 1 (Generate Report)     |Transaction 2 (Price Update)                      |READ COMMITTED        |REPEATABLE READ       |
|----|------------------------------------|--------------------------------------------------|----------------------|----------------------|
|t1​ |SELECT price FROM items WHERE id = 1|—                                                 |Sees $100             |Sees $100             |
|t2​ |—                                   |UPDATE items SET price = 150 WHERE id = 1; COMMIT;|Committed to DB       |Committed to DB       |
|t3​ |SELECT price FROM items WHERE id = 1|—                                                 |Sees $150 (Fuzzy Read)|Sees $100 (Consistent)|
|t4​ |COMMIT;                             |—                                                 |—                     |—                     |


