
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
- `COMMIT`: Tells the database engine to permanently apply and persist all changes made during the transaction.
- `ROLLBACK / ABORT`: Cancels the entire transaction, discarding all uncommitted in-memory or dirty changes and returning data to its initial state.

## 1. Atomicity
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



