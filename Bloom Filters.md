
# The Problem
- You try to sign up on instagram/ try to change your username.
- The system immediately tells you if the username is taken or not.
- How does the system checks for the existence of that username in the system?
- Approach 1:
  - Each time you type a username, a request goes to the backend. The backend searches through the database table for the existence of that username.
  - Time Complexity would be: O(n)
  - At scale, this would be very slow.
- Approach 2:
  - Instead of scanning the database table, we store all existing usernames in a hashmap/hashset.
  - Lookup for the username would be: O(1)
  - However, the space complexity would be: O(n).
- Approach 3: Bloom Filters
  - This balances the time and space complexity.

# Bloom Filters
- It is a space-efficient, probabilistic data structure used to test the existence of an element in the dataset.
- It always tells one of the two things:
  - "Definitely not": The definitely 100% does not exist in the dataset.
  - "Probably yes": Means, the element may or may not exist in the dataset.
- Bloom Filters may give you false positives: Bloom filters may say the data exists, but in reality it does not. This is fine for many use-cases.
- Bloom Filter will never give you false negatives: Bloom filters will never say the element does not exist which actually exists in the system. This is a guarantee provided by bloom filters.

## Working
There are two most important components in bloom filters:
1. Bit Array: A fixed size array of 1-bit. So, each index can store either 0 or 1. We could also call each indices as buckets/slots. Initally each bit is set to 0.
2. Hash Functions: A set of $k$ independent has functions. Every hash function takes the same input and gives a position/bucket no./slot no. in the bit array.

### Adding an element to bloom filter
- Feed the element into each of the $k$ hash functions.
- Each function outputs the position/index in the bit array.
- Set the bits at all of those positions to 1.

### Checking for an element
- Feed the eleemnt into each of the same $k$ hash functions to get the target indices.
- Check all those positions in the bit array.
- If any of the bits is 0: The element 100% does not exist in the system. Because if it had been added, all those bits would have been flipped to 1.
- If all of the bits are 1: The item probably exists. The element may or may not exist in the system. It's possible that a combination of other items flipped those exact bits by coincidence. This is a false positive case.

## Example

- **Bit Array Size (m)**: 10 bits (indices 0 to 9)
- **Number of hash functions (k)**: 2 (h1 and h2)
- Initial State

```
Index: [ 0  1  2  3  4  5  6  7  8  9 ]
Bits:  [ 0  0  0  0  0  0  0  0  0  0 ]
```
- Adding "Alice"
  - Hash "Alice" using our two functions.
  - h1("Alice") % 10 = 2
  - h2("Alice") % 10 = 5
  - Set the indices 2 and 5 in the bit array.

```
Bits:  [ 0  0  1  0  0  1  0  0  0  0 ]
               ^        ^
```

- Adding "Bob"
  - Hash "Bob" using our two hash functions.
  - h1("Bob") % 10 = 5
  - h2("Bob") % 10 = 8
  - Set the indices 5 and 8 in the bit array. Index 5 is already set by Alice.

```
Bits:  [ 0  0  1  0  0  1  0  0  1  0 ]
                        ^        ^
```

- Query "Alice" (True Positive Scenario)
  - h1("Alice") % 10 = 2
  - h2("Alice") % 10 = 5
  - We will check positions 2 and 5. Both are 1.
  - Result: "Alice" **probably** already exists in the system.

- Query "Charlie" (True Negative Scenario)
  - h1("Charlie") % 10 = 3
  - h2("Charlie") % 10 = 8
  - Check positions 3 and 8. Index 8 is set, but 3 is not set.
  - Result: "Charlie" is 100% not in the system.

- Query "Eve" (False Positive Scenario)
  - What would happen if we look up "Eve" who we actually never added in the system? Let's see
  - h1("Eve") % 10 = 2
  - h2("Eve") % 10 = 8
  - Both positions are set. They were previously set by Alice and Bob.
  - Result: "Eve" probably is in the system, even though she isn't.
  - This is the false positive case.

## Tuning up the parameters
- Imagine, our 10-sized bit array all fill up: all gets set to 1 as we keep adding new elements to bloom filter.
- For every query, the bloom filter will most probably give false positive. This is because the number of elements were large, and the size of array was small.
- Thats why in real-world scenario we tune up the following parameters:
  - $m$: The size of the bit array.
  - $k$: The number of hash functions.
- If your bit array is too small ($m$ is low) or you insert too many items ($n$ is high), the array fills up with 1s, and the false positive rate skyrockets toward 100%. Software engineers use specific mathematical formulas to calculate the optimal size of $m$ and $k$ based on their acceptable false-positive tolerance (e.g., 1%).

## Real World Applications
- Because Bloom filters are incredibly fast ($O(k)$ time complexity for both inserts and lookups) and use a fraction of the memory a Hash Set would require, they are widely used in massive scale systems.
- Databases (Cassandra, RocksDB, Bigtable): Before a database goes through the heavy lifting of reading a file from the hard drive to look for a specific row, it checks a Bloom filter. If the filter says "definitely not," the database skips the disk read entirely, saving massive amounts of I/O.
- Web Browsers (Google Chrome): Chrome used to use Bloom filters to cross-check URLs against a local list of malicious websites. If a URL flagged a "probably yes," Chrome would then make a secure API call to Google's servers to double-check.
- Content Platforms (Medium): Medium has used Bloom filters to track which articles have already been recommended to a specific user, ensuring you rarely see duplicates on your feed without needing to store your entire reading history in an active cache memory.

## Limitations
- The standard bloom filter does not support deleting elements.
- If you try to delete "Bob" by flipping bits 5 and 8 back to 0, you accidentally delete part of "Alice" too (since she relies on bit 5). To handle deletions, you would need a more complex variant called a Counting Bloom Filter, which uses numbers instead of raw binary bits.




     
