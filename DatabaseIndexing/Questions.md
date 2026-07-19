
## Why are b-trees better than binary search trees for databases?
- While a BST works beautifully in RAM, the data load process from HDD or SSD is very slow.
- You see, on seconday storages like HDD, data is stored in blocks. An OS does not read data from a disk byte-by-byte. Instead, they read or write data in terms of blocks. A block is usually of size 4KB or 8KB. 
- Since a BST node has just one key and 2 child pointers, so its tiny. If we store a BST node onto the disk, so much of block space would go to waste. We will be wasting an entire 4KB block space just to store few bytes of data. Thats problem number one.
- Second, if we go with BST, the height of the tree would be more. So, number of node reads would be more. That would mean, more number of block reads.
- In a B-Tree, a node contains large number of keys so that a single node occupies as much space as possible of a single block. Ideally, the node should fit perfectly in a block. So, with just one block read, we are reading hundreds/thousands of keys at once. So, the number of block reads would be a lot less.

### The Power of Fan-Out (shorted trees)
- The maximum number of children coming out from a single node is called fan-out (denoted as M). A binary tree has a strict fan-out of 2. A binary tree can have a fan-out of hundreds or thousands.
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
- B-tree is self-balancing. It builds the tree bottom-to-up. Every leaf node is at the same level. 
