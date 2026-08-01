
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

