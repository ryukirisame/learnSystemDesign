
# Relational Database ACID Transactions

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

### Core Transaction Control Commands:
- `BEGIN / START TRANSACTION`: Explicitly defines the start of a transaction block.
- `COMMIT`: Tells the database engine to permanently apply and persist all changes made during the transaction into the disk.
- `ROLLBACK / ABORT`: Cancels the entire transaction, discarding all uncommitted in-memory or dirty changes and returning data to its initial state.
