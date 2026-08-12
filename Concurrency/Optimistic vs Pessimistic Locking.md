# Optimistic Locking
- This type of locking assumes that conflicts are rare. So, each transaction can read freely. But we must verify before writing.
- This is not actually a lock as in traditional sense.
- The idea is this: Each row will include a token number (version, timestamp etc). Whenever a transaction reads a row, it will also read this version number. All is fine till now.
  - But when the same transaction tries to update the row, it will have to verify if the token number it read matches the token number stored in the table or not.
  - If the token number matches that means the row was not updated since the transaction read the row. So, the transaction can proceed with updating the row safely.
  - If the token number does not match, that means some other transaction has updated the row. So, in this case, the update fails and the client must retry again.
- More than one transaction can read the same row at the same time. That is totally fine.
  
<img width="900" alt="image" src="https://github.com/user-attachments/assets/200fc1d7-4645-4235-aa5c-d4a67ffffb29" />

## Codes
### Schema - adding a `version` column
```sql
CREATE TABLE accounts (
    id       SERIAL PRIMARY KEY,
    balance  INTEGER NOT NULL,
    version  INTEGER NOT NULL DEFAULT 1
);
```
### Making sure the version matches
```sql
-- You read version=1. Now you try to write:
UPDATE accounts
SET    balance = 150,
       version = version + 1
WHERE  id = 42
  AND  version = 1;   -- <-- this is the guard

-- If 0 rows updated → someone else changed it → retry
```

### JPA code
- Add the `@Version` annotation to a field in your entitiy. Hibernate will automatically manage incrementing this field on every update.
- If another transaction modified the row first, the version in the database will no longer match, returning `0` modified rows. Hibernate then throws an `OptimisticLockException`
```java
@Entity
public class Account {
    @Id
    private Long id;
    
    private int balance;

    @Version          // ← Hibernate manages this automatically
    private int version;
}

// Hibernate throws OptimisticLockException if the version mismatches
try {
    account.setBalance(account.getBalance() + 50);
    entityManager.merge(account);
    entityManager.getTransaction().commit();
} catch (OptimisticLockException e) {
    // Re-read and retry
}
```




# Pessimistic Locking

- Assumes conflicts are likely. A transaction explicitly acquires a lock on a row before performing any operation.
- All other transactions that try to lock the same row must wait until the lock is released (on commit or rollback).
- The lock is held for the entire duration of the transaction.
- The lock is managed by the database itself.

## Two types of locks

### 1. Exclusive Lock (X-lock)
Acquired via: `SELECT ... FOR UPDATE` (explicitly acquires X), `UPDATE`, `DELETE`, `INSERT`

- Only one transaction can hold X at a time.
- Blocks all other transactions trying to acquire any lock on the row. (both `FOR SHARE` and `FOR UPDATE` requests will wait).
- Plain `SELECT` (no locking clause) is NOT blocked — it reads a consistent snapshot via MVCC without acquiring a lock.
- Use when: you intend to write to the row.

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 42 FOR UPDATE;
-- X lock acquired. All lock requests on row 42 now block.
UPDATE accounts SET balance = 150 WHERE id = 42;
COMMIT; -- X lock released here
```

### 2. Shared Lock (S)
Acquired via: `SELECT ... FOR SHARE`

- Multiple transactions can hold S locks on the same row simultaneously.
- If a transaction tries to acquire an exclusive lock on a row which already has a shared lock, the transaction will have to wait until all shared locks are released.
- Does NOT block other FOR SHARE readers. Meaning, other transactions trying to acquire a shared lock on the same row can proceed.
- Plain `SELECT` (no locking clause) is also NOT blocked — MVCC again.
- Use when: you need to read a row and pin it(freeze it temporarily) so nobody can modify it. Even the same transaction cannot modify.
- The main idea is to block writers. Any write operation (UPDATE, DELETE, or INSERT affecting the row) implicitly requires an X-lock. Because an X-lock cannot co-exist with an S-lock, any transaction attempting a write will have to wait until all existing shared locks on that row are released.

```sql
BEGIN;
-- Pin the parent row so it cannot be deleted while we insert the child
SELECT * FROM users WHERE id = 7 FOR SHARE;
INSERT INTO orders (user_id, amount) VALUES (7, 250);
COMMIT;
```


## Compatibility
<img width="730" height="170" alt="image" src="https://github.com/user-attachments/assets/6c562abb-d1f8-434d-925c-5a7b7a8bf846" />


## Lock upgrade 
- Lock upgrading is when a transaction changes its existing lock on a row from a weaker type to a stronger type — specifically from Shared (S) to Exclusive (X).
- At first we acquire only S lock on a row to read only. But mid-transaction if we change our mind and want to write also. Now we will need an X lock on the same row we already hold an S lock on.
- In this case, if no other transaction holds a S lock on the row, the lock is upgraded immediately.
- However, if there are other transactions that hold a S lock on the row, the current transaction will have to wait until all S locks are released on the row.
  
### Deadlock scenario
- If two transactions both hold S locks on a row and both try to upgrade to X, they deadlock — each waits for the other to release its S lock.
```
Row 42 granted set: [Txn A: S, Txn B: S]

Txn A wants to upgrade S → X
→ must wait for Txn B's S to release

Txn B wants to upgrade S → X
→ must wait for Txn A's S to release

← both are waiting for each other. Neither will ever release.
← DEADLOCK.
```
- The database detects this cycle and kills one of the transactions with an error, forcing the other to proceed.
- Fix: if you know you'll write, use `FOR UPDATE` from the start, never `FOR SHARE → upgrade`.


## Lock lifetime
Held for the entire duration of the transaction, not just the statement. Released on COMMIT or ROLLBACK.

---

## When to use which

There are two layers:

### Low level — automatic (atomic operations)
- `UPDATE`, `DELETE`, and `INSERT` acquire exclusive locks indirectly as part of their execution. You never ask for them — the database just does it to protect the write itself.

- Use when: the entire `read-modify-write` logic fits inside a single SQL statement. No gap, no race condition, no manual locking needed.

```sql
-- ATOMIC: read and write happen as one unit inside SQL
-- DB acquires and releases X lock automatically
UPDATE accounts
SET balance = balance + 50   -- no gap between read and write
WHERE id = 42;

DELETE FROM accounts WHERE id = 42;

INSERT INTO accounts (id, balance) VALUES (42, 100);
```
- All the individual UPDATE, DELETE AND INSERT statement above are atomic in nature.
  
### High level — explicit (non-atomic operations)
- `SELECT FOR UPDATE` is a deliberate, application-level tool. Use it when your `read-modify-write` cycle spans multiple statements with application logic in between. You are telling the database: "I am about to make a decision based on this data — hold it for me until I am done."

- Use when: you need to read first, do something in application code, make a decision, and then write. That gap between read and write is where race conditions live — `FOR UPDATE` closes it.

```sql
-- NON-ATOMIC: logic lives in application code
BEGIN;

SELECT balance FROM accounts    -- read
WHERE id = 42
FOR UPDATE;                     -- X lock held across the gap

-- application code runs here:
-- check if balance > minimum threshold
-- calculate new balance
-- apply business rules
-- make a decision

UPDATE accounts                 -- write (X lock already held, no wait)
SET balance = 150
WHERE id = 42;

COMMIT;                         -- X lock released here
```

### The race condition FOR UPDATE prevents

```sql
-- WITHOUT SELECT FOR UPDATE (dangerous gap):
-- Txn A reads balance = 100
-- Txn B reads balance = 100        ← both read the same stale value
-- Txn A writes balance = 150
-- Txn B writes balance = 150       ← overwrites A, 50 is lost

-- WITH SELECT FOR UPDATE (safe):
-- Txn A: FOR UPDATE → X lock acquired, reads balance = 100
-- Txn B: FOR UPDATE → blocked, waits
-- Txn A: writes balance = 150, commits, releases X lock
-- Txn B: unblocked, reads fresh balance = 150, writes 200 ✓
```

### The general rule

> If the entire `read-modify-write logic` can be expressed in a single SQL statement, let the database handle locking automatically.
> If the logic requires reading first, doing something in application code, and then writing — that gap is where race conditions live, and `SELECT FOR UPDATE` is how you close it.

---

## Code examples

### Exclusive lock — full pattern
```sql
BEGIN;
SELECT * FROM accounts WHERE id = 42 FOR UPDATE;
-- X lock held — all other lock requests on row 42 block here
-- application logic runs here safely
UPDATE accounts SET balance = balance + 50 WHERE id = 42;
COMMIT; -- X lock released
```

### Shared lock — pinning a parent row
```sql
BEGIN;
-- Pin the parent row so it cannot be deleted while we insert the child
SELECT * FROM users WHERE id = 7 FOR SHARE;
INSERT INTO orders (user_id, amount) VALUES (7, 250);
COMMIT;
```

### FOR UPDATE variations

| Modifier    | Behavior                                        | Use case             |
|-------------|-------------------------------------------------|----------------------|
| (default)   | Waits indefinitely for the lock                 | Standard writes      |
| NOWAIT      | Fails immediately with error if row is locked   | Fail-fast scenarios  |
| SKIP LOCKED | Skips locked rows, returns only available ones  | Job queues, workers  |

```sql
-- Standard (blocking — waits indefinitely)
SELECT * FROM inventory WHERE item_id = 101 FOR UPDATE;

-- Fail immediately if locked
SELECT * FROM inventory WHERE item_id = 101 FOR UPDATE NOWAIT;

-- Skip locked rows (ideal for job queues)
-- The query skips any locked rows and only returns available, unlocked rows. This is highly useful for implementing message queues.
SELECT * FROM inventory WHERE status = 'pending' FOR UPDATE SKIP LOCKED;
```

