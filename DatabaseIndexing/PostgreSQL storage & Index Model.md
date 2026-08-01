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


