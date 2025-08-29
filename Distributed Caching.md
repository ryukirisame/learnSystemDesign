


# Thrashing
Thrashing happens when the cache spends most of its time evicting and loading data instead of serving requests efficiently. The result:
- Cache hit ratio drops close to zero.
- Performance is worse than having no cache, because of extra overhead (eviction + storage updates).

## Causes of Thrashing

### 1. Insufficient Cache Storage

Thrashing can happen due to many reasons, one of them is when the cache capacity is less. Imagine a scenario when the cache can hold just one entry. In that case the following happens:
1. User A requests a data.
2. Its a cache miss -> Hit DB -> Cache that data (lets say X)
3. User B requests a data.
4. Its a cache miss -> Hit DB -> Time to cache this data (say Y)
    1.  But the cache has a capacity of 1. And the cache already has data X.
    2.  So, we need to evict X.
    3.  Evict X.
    4.  Cache Y.
5. Again User A requests for the same data X.
6. Its a cache miss -> Hit DB -> Time to cache X

This goes on and on. The cache never really serves any data. Hit ratio = 0%. This is worse than having no cache at all, as a new step is added that is adding no value to the entire process. 


### 2. Poor Eviction Policy
- Thrashing may occur due to poor cache eviction policy as well.
- Example:
    - Using Random Replacement when access pattern clearly favors LRU (Least Recently Used) or LFU (Least Frequently Used).
    - Evicting useful data too early.

### 3. Workload Characteristics
- If the user access pattern is random and not predictable, then we cannot predict which items to cache and which ones to evict. 
- Caching depends heavily on **predictability in access patterns**. If user behavior is completely random (no repetition, no locality), then:
    - Every request becomes a cache miss.
    - The cache will keep evicting old items to make space for new ones, but since none of them are reused, those evictions are pointless. 



### 4. Cache Pollution

### 5. Distributed Cache Thrashing

## How Modern Caches Handle Randomness

- If we cache everytime a cache miss occurs, it may evict a hot item. And this newly cache may not be used that frequently at all. This reduces the hit ratio.
- "Not every item deserves to go into cache".

Solutions:
### 1. Admission Policies (Don't Cache Everything)
- 


## How to Prevent Thrashing
- Right-Sized Cache
- Better eviction policies













