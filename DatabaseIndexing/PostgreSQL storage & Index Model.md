# PostgreSQL Storage & Index Model
- PostgreSQL does not use a clustered B+ tree to store table data. Instead, it stores all rows in a separate heap file.
- This heap file is divided into fixed-size pages (8KB by default), and those pages can be spread across disk blocks wherever free space exists. When a new row is inserted, it is written into any page that has free space — there is no ordering guarantee.

## How indexes work in Postgres
- Every index — whether on the primary key or any other column — is a separate B+ tree structure stored apart from the heap.
- The leaf nodes of these B+ trees do not contain actual row data. Instead, they store a ctid (a tuple identifier), which is a pair of (page number, row offset within that page).
- To fetch a row using an index, Postgres must:
  - Traverse the B+ tree to find the matching `ctid`.
  - Load the heap page identified by the ctid into memory.
  - Seek to the row offset within that page and read the row.
- This is always a two-hop access — index lookup, then heap fetch. 



