
## Why are b-trees better than binary search trees for databases?
- While a BST works beautifully in RAM, the data load process from HDD or SSD is very slow.
- You see, on seconday storages like HDD, data is stored in blocks. An OS does not read data from a disk byte-by-byte. Instead, they read or write data in terms of blocks. A block is usually of size 4KB or 8KB. 
- Since a BST node has just one key and 2 child pointers, so its tiny. If we store a BST node onto the disk, so much of block space would go to waste. We will be wasting an entire 4KB block space just to store few bytes of data. Thats problem number one.
- Second, if we go with BST, the height of the tree would be more. So, number of node reads would be more. That would mean, more number of block reads.
- In a B-Tree, a node contains large number of keys so that a single node occupies as much space as possible of a single block. Ideally, the node should fit perfectly in a block. So, with just one block read, we are reading hundreds/thousands of keys at once. So, the number of block reads would be a lot less.

### The Power of Fan-Out (shorter trees)
- The maximum number of children coming out from a single node is called fan-out (denoted as M). A binary tree has a strict fan-out of 2. A B-tree can have a fan-out of hundreds or thousands.
- So, the larger the number of keys in a node of B-Tree, the more the children. The more children means shorter tree height compared to BST. Hence, a larger fan-out means shorter tree height.
- With a fan-out of 2, the BST can only double its capacity at each level: $2^{l}$.
- With a fan-out of 100, the B-tree multiplies its capacity by 100 at each level: $100^l$
- Because of large fan-out of B-trees, it spreads horizontally a lot, so, the tree height remains shorter than BST.
- Lets suppose we are storing 1 million rows
  - Balanced BST: $\log_2(1,000,000) \approx 20$ levels deep.
  - B-Tree (with an order of $m=200$): $\log_{100}(1,000,000) \approx 3$ levels deep.

### 20 Disk Reads vs. 3 Disk Reads
- If databases, if we want to optimize reads/writes, we have to minimize disk accesses.
- In BST we will have to visit max 20 nodes, till we reach our target node if the key exists in the leaf node. That means max 20 block reads.
- In B-Tree we will have to visit max 3 nodes, till we reach out target node if the key exists in the leaf node. That means max 3 block reads.

### Balancing
- Usually, we go with auto-incremented primary key. 1,2,3,4,... so on. If we decide to use BST to store this, we might end up with right skewed BST. Which will basically turn into a linked list.
- B-tree is self-balancing. Balance is made via splitting a node and push the median upwards. It builds the tree bottom-to-up. Every leaf node is at the same level. 

## How different queries map to the tree
### Point Lookups (`WHERE id = 105`)
- Easy. Just go through the tree and find the key.

### Range Scans (`WHERE age BETWEEN 20 AND 30`)
- Since B+ tree's leaf nodes are linked, so:
  - The database just needs to reach the first leaf node containing the first range key (`20`)
  - Once we reach the first leaf node, we can start reading horizontally across the leaf nodes using the linked list pointers. Once we find a key greater than `30`, we can stop.

### Sorting (`ORDER BY age ASC`)
- If you query `SELECT * FROM users ORDER BY age`, sorting millions of rows in RAM is incredibly expensive.
- However, if a **covering** index exists on `age`, the database completely skips sorting. Because the B+ Tree leaf nodes are already physically sorted on disk, the database simply starts at the leftmost leaf node and reads straight across.
- This is true only when we are selecting columns that are all in the index. In case of sorting by PK, clustered index will be used. In case of sorting by non-PK column, we need a covering index to sort efficiently.
- In PostgreSQL, the table data is stored on heap file. So, the database still has to go to heap file. For large result sets, postgres may choose to ignore the index entirely for sorting and just do sequential heap file scan + sort in memory.

### Prefix Scans
- A prefix search is when you search for all values that start with a given string.
```sql
SELECT * FROM users WHERE name LIKE 'Jo%';
```
- If we have an index on column `name`, the index tree will store the keys lexicographically sorted.
- Since the keys are sorted, we can easily find out which keys in the B+ tree node start with `Jo`, and include that in result set.
- This is essentially a range scan across nodes:
  - The database finds the first key which has `Jo` in it.
  - Then reads across the leaf nodes horizontally using the linked list - just like range scans.
  - The database stops when it hits a key that no longer starts with `Jo`.
- Prefix scans definitely have an advantage of the index. But middle and suffix scans doesn't have that. They can't use the B+ tree index. 

## Why do databases prefer B+ trees instead of classic B-trees?
- In classic B-trees, every node - both internal and leaf - can store actual data. So, if we are doing point lookup, we find the key in an internal node, get the data and we are done.
- In B+ tree, internal nodes only store keys. They are used for navigation purpose only. All actual data lives in the leaf nodes. The leaf nodes are also linked together as a linked list.
  
#### 1. Internal nodes can fit more keys in B+ trees
- Since internal nodes in B+ tree store only keys (no data), they are small. This means, we can fit a lot more keys in a single node. Which means the tree will widen more and its height will be shorter. This directly translates to lesser block reads.

#### 2. Range scans are massively better in B+ trees
- Since the leaf nodes are linked, we can easily do a range scan in B+ trees.
- In B-trees, we will need to traverse up and down multiple times.

#### 3. Better cache and disk efficiency
- Since internal nodes of B+ trees are light, we can easily fit the internal nodes in memory as cache. They also work as index. So, it will be blazing fast to query the index.
- In classic B trees, internal nodes also contain data. So, it would be difficult to cache them in memory. 

## What happens when we have a secondary index and there are multiple records that matches a key

Say you have a users table with a primary key `id` and a secondary index on `age`. Multiple users can have the same age — so age is a non-unique column.
```
users: id=1 age=25, id=2 age=25, id=3 age=30, id=4 age=25
```
### How the secondary index handles duplicates
- In a B+ tree secondary index, the leaf nodes store (indexed key, pointer to row) pairs. When multiple rows share the same key, the index simply stores multiple entries with the same key — one entry per matching row.
```
Leaf nodes of secondary index on age:
... | (25, ptr→id=1) | (25, ptr→id=2) | (25, ptr→id=4) | (30, ptr→id=3) | ...
```
- All three age=25 entries sit side by side in the leaf nodes because the leaf nodes are sorted — same keys cluster together naturally.

### What happens during a query
```sql
SELECT * FROM users WHERE age = 25;
```
1. Traverse the B+ tree to find the first leaf entry with `age = 25`
2. Read across the linked leaf nodes collecting all entries where `age = 25`
3. For each entry, follow the pointer to fetch the actual row - double lookup 
4. Stop when you hit `age = 26` or higher

So it's essentially a range scan on the secondary index — just a range where start and end are the same value.

```
Postgres:  index leaf → ctid → heap page (random I/O into heap)
InnoDB:    index leaf → PK   → clustered index leaf (another B+ tree traversal)
```

- If `age = 25` matches 3 rows, that's 3 double lookups — fine. But if age = 25 matches 100,000 rows, that's 100,000 individual heap fetches in Postgres or 100,000 clustered index traversals in InnoDB. At that point the double lookup cost becomes enormous.
- This is exactly why sometimes the query planner says - "you know what, forget the index entirely, I'll just do a full sequential scan."
```
Few matching rows    →  index is worth it, double lookup cost is small
Many matching rows   →  index overhead exceeds full scan cost, skip the index
```
- This is called selectivity.
- A column is highly selective if most values are unique (like id or email) - index is always worth it.
- A column is low selective if many rows share the same value (like age, gender or status) - index may be ignored for broad queries.
- A good rule of thumb when designing indexes:
  - Index columns that are highly selective. Secondary indexes on low-selectivity columns can actually hurt performance for broad queries because the double lookup cost per row adds up faster than a simple sequential scan.

### Selectivity
- Selectivity is simply a measure of how unique the values in a column are.
- It's expressed as a ratio:
```
selectivity = number of distinct values / total number of rows
```
- A value close to 1 means almost every row has a unique value — highly selective. A value close to 0 means most rows share the same value — low selectivity.
```
users table — 1,000,000 rows

email       → 1,000,000 distinct values → selectivity = 1.0   (perfectly unique)
id          → 1,000,000 distinct values → selectivity = 1.0
country     → 195 distinct values       → selectivity = 0.0002 (very low)
age         → ~80 distinct values       → selectivity = 0.00008
gender      → 2 distinct values         → selectivity = 0.000002
is_active   → 2 distinct values         → selectivity = 0.000002
```
#### Why it matters for indexes
- When you query `WHERE email = 'john@example.com'`, the database knows this will return exactly 1 row. The double lookup cost is trivial — absolutely worth using the index.
- When you query `WHERE country = 'USA'`, the database knows this might return 300,000 rows out of 1,000,000. Now it has to do 300,000 double lookups. At that point a full sequential scan — one linear pass through the heap — is cheaper.
- The query planner uses selectivity statistics to make this decision automatically. In Postgres you can see these statistics in `pg_stats`:
```sql
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'users';
```

#### The practical rule
```
High selectivity  (close to 1)  →  index is very useful
Low selectivity   (close to 0)  →  index may be ignored or even harmful
```

