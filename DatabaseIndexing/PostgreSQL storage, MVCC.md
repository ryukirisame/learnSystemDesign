# PostgreSQL Storage & Index Model
- In databases like MySQL(InnoDB), a clustered index means the table data itself is stored inside the B+ tree. The leaf nodes contains the table rows itself as payload.
- PostgreSQL does not use a clustered B+ tree to store table data. Instead, it stores all rows of a table in a separate heap file.
- This heap file is divided into fixed-size pages (8KB by default), and those pages can be spread across disk blocks wherever free space exists. When a new row is inserted, it is written into any page that has free space — there is no ordering guarantee.

## How indexes work in Postgres
- Every index — whether on the primary key or any other column — is a separate B+ tree structure stored apart from the heap.
- The leaf nodes of these B+ trees do not contain actual row data. Instead, they store a `ctid` (a tuple identifier), which is a pair of `(page number, row offset within that page)`.
- To fetch a row using an index, Postgres must:
  - Traverse the B+ tree to find the matching `ctid`.
  - Load the heap page identified by the ctid into memory.
  - Seek to the row offset within that page and read the row.
- This is always a two-hop access — index lookup, then heap fetch. 


## How storing table data in a heap file is beneficial
### MVCC(Multi-Version Concurrency Control)
- When we update a row, the writer doesn't make changes to the old row. Instead, it creates a new version of the row and writes it to the file. The old version is marked as expired. It is not deleted immediately. Later, through a background process, it is deleted to claim space.
- The advantage of this is writers do not block readers. and vice versa. Since locking is expensive and complex, this design gets rid of locking and provides concurrent access to the same record by a reader and writer both.
- In databases that use clustered indexing - like MySQL InnoDB, they also have MVCC. Storing data in a heap makes implementing MVCC much simpler and cheaper, because new row versions can land anywhere in the heap. In a clustered index, inserting a new row version in sorted order is structurally harder and more expensive.

## `CLUSTER` command
- If we want to physically order the heap pages to match an index, we can use this command. This will sort the pages of the file according to a specific index.
- `CLUSTER my_table USING my_index;`
- This rewrites the entire table so that rows are physically sorted in index order — great for range scans.
- NOTE: It's a one-time operation. New inserts and updates go wherever there's free space, and the physical ordering drifts over time. You have to re-run CLUSTER periodically to get the benefit back.

## Range query trade-off
- Now, since we are storing rows randomly in the heap file we may think range queries will suffer. Well, this is how postgreSQL handles it:
  -  For small ranges, the two-step process to access rows is almost negligible.
  -  For large ranges, we can go with covering indexes (index-only scans). This way, we will get the required data in the index itself. No need to access the table file.
  -  In cases where our query doesn't have a covering index, and row scatter is severe, database will choose to skip the first step entirely and perform linear scan on the table file. This will be cheaper.

# The Heap File
- The heap is just a file where the actual table rows are stored. That's it.
- A row is simply placed wherever there's a free space. 
- Initially the file will be empty - no pages. PostgreSQL will add pages to it as we insert rows.
- When all existing pages are full, postgres will extend the file by appending new page. 
- Physically it will look something like this:
```
Table "orders" on disk
├── Page 1  [ row, row, empty, row, row ]
├── Page 2  [ row, empty, empty, row, row ]
├── Page 3  [ row, row, row, empty, empty ]
└── ...
```
- Each page will be 8KB by default. In our case, one page can fit 5 table rows.


# MVCC (Multi-Version Concurrency Control) in PostgreSQL
- Imagine theres a reader and a writer, both wanting to operate on the same row at the same time.
  - Reader: `SELECT * FROM orders WHERE id = 5` — takes a few milliseconds to run
  - Writer: `UPDATE orders SET status = 'shipped' WHERE id = 5` — happens mid-way through the read

- If we allow both of them to work on the same row, the reader might see corrupted data.
- One solution would be: locking the row. The writer will obtain a lock on the row, the reader will wait. This will work, but performance might suffer a lot in a busy database.
- The idea of MVCC is to keep multiple versions of the same row. When a writer updates a row, it doesn't overwrite the old one - it creates a new version alongside it.
- Any reader which started before the writer, simply reads the old version. Any reader that started after the writer was done (after COMMIT), reads the new version.
- This allows a reader and writer to access the same record without blocking each other.

## How it works
When we run `UPDATE`:
- Instead of updating the old row, database will create a new row entirely.
- It will mark the old row as expired.
- The old row is not deleted yet - it will stay on the disk.
- Because old versions are never immediately deleted, we will have dead tuples accumulating over time.
- PostgreSQL solves this with a background process called `VACUUM`, which scans the heap file, identifies expired rows that are not in use by any transaction, and marks that space as reusable - This does not shrink the file size though. The space stays with the file.
- In order to truly reclaim space - we need to run `VACUUM FULL` command. This will rewrite the entire table into a new file, compacting and shrinking the file size. The only problem is, it will lock the table for a while. So, this is not recommended on a live busy table.
- In short: the file grows as data grows, but only explicitly shrinks with VACUUM FULL. Normal VACUUM just reclaims space for reuse internally.


# MVCC in InnoDB (Multi-Version Concurrency Control)

## The Problem MVCC Solves

Imagine two things happening at the same time on the same row:

- **Alice (reader):** `SELECT * FROM orders WHERE id = 5`
- **Bob (writer):** `UPDATE orders SET status = 'shipped' WHERE id = 5`

If Bob overwrites the row while Alice is reading it, Alice might see corrupted, half-written data.

The naive fix is **locking** — Bob locks the row, Alice waits. This works but kills performance in a busy database. Readers and writers are constantly blocking each other.

**MVCC's idea:** instead of locking, keep multiple versions of the same row. Bob doesn't overwrite the old row — he creates a new version alongside it. Alice simply reads the old version. Nobody waits for anybody.


## How InnoDB Implements MVCC

InnoDB uses two things:

- **Main file** — always contains the latest version of the row
- **Undo log** — stores old versions of rows that have been updated

When a writer updates a row:
1. It copies the current row into the undo log (preserving the old version)
2. It updates the main file with the new version
3. It adds a pointer from the new version in the main file back to the old version in the undo log

This chain of old versions is called the **version chain**.

<img width="1472" height="250" alt="image" src="https://github.com/user-attachments/assets/18b03bd7-81f9-48d3-a5d4-9783ebe21853" />
<img width="1472" height="250" alt="image" src="https://github.com/user-attachments/assets/ee04b39a-eb65-4ead-b530-f608ae003fe6" />



## What is a Snapshot?

When a transaction begins, the database hands it a **snapshot** — simply a number representing the latest committed transaction at that moment.

The snapshot means: *"I only trust data created by transactions up to this number. Anything created after this is invisible to me."*

**Example:** Transaction 11 starts when the latest commit was transaction 10. Its snapshot is 10. It will only read row versions created by transaction 10 or earlier.



## Full Walkthrough Example

### Starting state

```
Main file:
id=5  status='pending'  (created by txn 10)

Undo log: empty
```

### Step 1: Transaction 11 (reader) begins

It receives snapshot = 10. It hasn't read anything yet — it just has this cutoff number in its pocket.

### Step 2: Transaction 12 (writer) updates the row

Before touching the main file, InnoDB saves the current row to the undo log:

```
Undo log:
id=5  status='pending'  (created by txn 10)
```

Then it updates the main file:

```
Main file:
id=5  status='shipped'  (created by txn 12)  ──ptr──▶  Undo log:
                                                        id=5  status='pending'  (created by txn 10)
```

Transaction 12 commits.

### Step 3: Transaction 11 reads id=5

It goes to the main file and sees `status='shipped'` created by txn 12.

It checks its snapshot: *"Is txn 12 within my snapshot of 10?"* No — 12 > 10. Too new. Invisible.

It follows the pointer to the undo log and finds `status='pending'` created by txn 10.

It checks: *"Is txn 10 within my snapshot of 10?"* Yes — 10 ≤ 10. Visible. ✅

**Transaction 11 reads `status='pending'`** — the world as it was when it started.

### Step 4: Transaction 15 (a future reader) reads id=5

It started after txn 12 committed, so its snapshot is 12 or higher.

It goes to the main file and sees `status='shipped'` created by txn 12.

It checks: *"Is txn 12 within my snapshot?"* Yes. ✅

**Transaction 15 reads `status='shipped'`** — directly from the main file. No undo log needed.



## What if a Reader and Writer Hit the Same Row at the Same Time?

This is where MVCC really shines. The writer does not wait for the reader, and the reader does not wait for the writer.

The writer always follows this order:
1. Save old row to undo log **first**
2. Update main file **second**

So at any given moment, either:
- The main file still has the old row → reader reads it directly ✅
- The main file has the new row, undo log has the old row → reader follows the pointer ✅

There is no in-between state where the reader sees garbage. The old version is always accessible — either in the main file or in the undo log.

Think of it like this: imagine you are reading a document and someone wants to edit it. Instead of grabbing the paper from your hands, they first make a photocopy and file it away, then edit the original. At no point are you holding a half-edited document.



## What About True Simultaneous Access (Same Millisecond)?

At the exact hardware level, two processes cannot read and write the same memory or disk location simultaneously. InnoDB handles this with a **latch** — an extremely brief, low-level mechanism held for just microseconds while a page is being physically read or written.

This is different from a lock:

| | Latch | Lock |
|---|---|---|
| Duration | Microseconds | Milliseconds to seconds |
| Purpose | Physical page access | Protecting a row during a transaction |
| Eliminated by MVCC? | No — unavoidable physics | Yes |

Think of it like a revolving door. Two people cannot pass through at the exact same instant — one goes first, the other waits half a second. Nobody considers that "blocking" in any meaningful sense.

So MVCC eliminates **meaningful waiting** (row-level locks). The microsecond latch is just physics — it exists at the OS/hardware level regardless of what the database does.



## The Only Case Where Real Waiting Happens

**Writer vs Writer.** If two transactions want to update the same row simultaneously, one must wait for the other to commit or rollback. You cannot have two conflicting new versions of the same row created at once.

```
Reader  +  Reader  →  no waiting
Reader  +  Writer  →  no waiting  (MVCC handles it)
Writer  +  Writer  →  one waits  (lock required)
```



## Cleanup — The Undo Log Doesn't Grow Forever

Old versions in the undo log are kept as long as any active transaction might need them. Once no running transaction has a snapshot old enough to need a version, InnoDB purges it from the undo log automatically in the background.

This is simpler than Postgres's approach — in Postgres, dead row versions accumulate inside the heap file itself and require a separate VACUUM process to clean up. InnoDB keeps the main file clean by design, at the cost of maintaining the undo log separately.



## Summary

- MVCC keeps multiple versions of a row so readers and writers don't block each other
- InnoDB stores the **latest version** in the main file and **old versions** in the undo log, linked by pointers
- Every transaction gets a **snapshot** — a cutoff number — and only sees row versions created at or before that point
- Readers walk back through the undo log version chain until they find a version that fits their snapshot
- The only real blocking is **writer vs writer**, and a microsecond-level **latch** for physical page access

<img width="1472" height="440" alt="image" src="https://github.com/user-attachments/assets/d03f1e92-ce5d-4626-98d7-86f803622b0f" />

