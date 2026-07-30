
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
  
