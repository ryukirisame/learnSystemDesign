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
- In databases that use clustered indexing, writers will have to lock readers if they want to work on the same record.

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


# MVCC (Multi-Version Concurrency Control)
- Imagine theres a reader and a writer, both wanting to operate on the same row at the same time.
  - Reader: `SELECT * FROM orders WHERE id = 5` — takes a few milliseconds to run
  - Writer: `UPDATE orders SET status = 'shipped' WHERE id = 5` — happens mid-way through the read

- If we allow both of them to work on the same row, the reader might see corrupted data.
- One solution would be: locking the row. The writer will obtain a lock on the row, the reader will wait. This will work, but performance might suffer a lot in a busy database.
- The idea of MVCC is to keep multiple versions of the same row. When a writer updates a row, it doesn't overwrite the old one - it creates a new version alongside it.
- Any reader which started before the writer, simply reads the old version. Any reader that started after the writer was done (after COMMIT), reads the new version - because by this time, the `ctid` of the row would be updated to the new version - so naturally new readers will end up routed to the new version.
- This allows a reader and writer to access the same record without blocking each other.

## How it works
When we run `UPDATE`:
- Instead of updating the old row, database will create a new row entirely.
- It will mark the old row as expired.
- The old row is not deleted yet - it will stay on the disk.
- Because old versions are never immediately deleted, we will have dead tuples accumulating over time.
- PostgreSQL solves this with a background process called VACUUM, which scans the heap file, identifies expired rows that are not in use by any transaction, and deletes it. 
- 






