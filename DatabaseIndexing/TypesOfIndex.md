
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
  
# Secondary Indexes
- Let's suppose that we fire a query that searches for a record not by primary key but by some other column. For example: searching a user by name instead of user_id. For this query, the clustered index cannot help us. And hence the database would have to scan the entire table to search for the users with that name. This would be very costly.
- To prevent this, we create a Secondary Index: `CREATE INDEX idx_name ON users(name)`;
- A separate B+ tree would be created with `name` as the key for each node.
- The secondary index sorts its keys alphabetically by the `name` column.
- The leaf nodes store the clustered index key as the payload instead of the entire record itself. This works as a pointer back to the clustered index tree where the actual table is stored.

## The "double lookup" problem
- Say our query is: `SELECT * FROM users WHERE email = 'sid@example.com';`
- With a secondary index on `email`:
  1. Traverse the secondary index B+ tree on `email` -> reach the leaf -> get the primary key (say, id = 4521)
  2. Traverse the clustered index B+ tree on `id` -> reach the leaf -> get the full row.
- That's two B+ tree traversals instead of one.  
