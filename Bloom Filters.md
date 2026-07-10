
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








     
