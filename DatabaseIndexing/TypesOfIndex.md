
# Clustered Indexes 
- When the table data itself is stored in the leaf nodes of index's B+ tree. There's no separate "table file" and "index file". The B+ tree that serves as the index also stores the table as well.
- This is particularly true for MySQL's InnoDB engine.
- Since we can store the table only in one way at a time, we can have only one clustered index for a table.
- Which column of the table is used as the key for B+ clustered index tree?
  - If we have a primary key, that's used.
  - If no primary key, InnoDB picks the first `UNIQUE NOT NULL` column.
  - If neither exists, InnoDB silently generates a hidden 6-byte `ROW_ID` and clusters on that.
- Why "clustered"?
  - Because records with adjacent key values are physically stored next to each other on the disk. So its like all the adjacent records form a cluster, and are not scattered.
  
# Secondary (Non-Clustered) Indexes 
- Let's suppose that we fire a query that searches for a record not by primary key but by some other column. For example: searching a user by name instead of user_id. For this query, the clustered index cannot help us. And hence the database would have to scan the entire table to search for the users with that name. This would be very costly.
- To prevent this, we create a Secondary Index: `CREATE INDEX idx_name ON users(name)`;
- A separate B+ tree would be created with `name` as the key for each node.
- The secondary index sorts its keys alphabetically by the `name` column.
- The leaf nodes store the clustered index key as the payload instead of the entire record itself. This works as a pointer back to the clustered index tree where the actual table is stored.

### The "double lookup" problem
- Say our query is: `SELECT * FROM users WHERE email = 'sid@example.com';`
- With a secondary index on `email`:
  1. Traverse the secondary index B+ tree on `email` -> reach the leaf -> get the primary key (say, id = 4521)
  2. Traverse the clustered index B+ tree on `id` -> reach the leaf -> get the full row.
- That's two B+ tree traversals instead of one.  

# Covering Indexes 
- This index helps us avoid the double lookup problem that we had in secondary indexes.
- So, the idea is: If the query only needs those columns that are already present in the secondary index itself, database can skip traversing the clustered index entirely. This type of index is called covering index.
- ```sql
  -- Index: (email)
  -- Query needs only 'id' (which is embedded as the PK reference) and 'email'
  SELECT id FROM users WHERE email = 'sid@example.com';  -- covered, no lookup to clustered index
  ```
- A covering index is an index which contains all the data the query needs. The actual table is never actually read at all.

## Example
- Suppose we have a table:
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,         -- Clustered Index
    email VARCHAR(255),
    status VARCHAR(50),
    created_at TIMESTAMP
);
```
### Case A: Standard Secondary Index (Not Covering)
We create a standard secondary index:
```sql
CREATE INDEX idx_email ON users(email);
```
- This creates a B+ tree where the key is email. Each node contains many emails sorted lexicographically.
- The leaves of this B+ tree will contain `id` as payload because that's the PK for this table.
  
Now consider this query:
```sql
SELECT id, email, status FROM users WHERE email = 'alice@example.com';
```

- Execution Flow:
  1. Engine searches `idx_email` B+ tree for `'alice@example.com'`.
  2. Finds leaf node containing `email: 'alice@example.com'` and `id: 101`.
  3. The query demands `status`, which is not in `idx_email`.
  4. Engine must perform a Key Lookup into the primary key B+ tree using `id: 101` to read the `status` column from disk.

### Case B: Upgrading to a Covering Index
To make our query "covered", we have two options:

#### Option 1: Composite Index
```sql
CREATE INDEX idx_email_status ON users(email, status);
```
- Once we do this, every node of the B+ index tree will contain both email and status as key. Please note that the order of `email` and `status` matters. `email` comes first, then `status`. An index on `(status, email)` is different than an index on `(email, status)`.
-  The leaf node already contains `id`.
-  So, now our index contains `email`, `status` and `id`, which are required for the query. So, this index is now a covering index for this query.

#### Option 2: Index with `INCLUDE` (PostgreSQL, SQL Server, MySQL 8.0+)
```sql
CREATE INDEX idx_email_inc_status ON users(email) INCLUDE (status);
```
- Each node of the B+ tree of the index contains only `email` as the key.
- `status` only contains in the leaf nodes together with `id` as extra payload.
-  Advantage is that non-leaf nodes will require less storage for keys and hence we will be able to fit more keys into a single node/block, and hence our tree will branch out more, so the height of the tree will reduce and hence it will mean less block reads.
-  
