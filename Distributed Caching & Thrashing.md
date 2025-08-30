
# Distributed Caching

- When our dataset is small, it's usually enough to keep all the cache data on one server.
- But as our data grows, a single cache will not be enough to handle millions of users and massive datasets.
- In such scenarios, we distribute the cache data across multiple servers.
- Distributed caching is a technique where cache data is stored across multiple nodes (servers) instead of being confined to a single machine.
- This allows the cache to scale horizontally and accommodate the needs of large-scale applications.

## Components of Distributed Caching

#### 1. Cache Nodes
These are the individual servers where the cache data is stored. Each node is a part of the overall cache cluster.

#### 2. Client Library/Cache Client
- Client library is a software component/library/SDK that provides an interface for applications to interact with distributed cache systems like Redis, Memcached, or Apache Ignite. This library handles the logic of connecting to cache nodes, distributing data, and retrieving cached data.
- Cache client is the actual running instance or object that connects and talks to the cache server.
- Client Library = The software package you install
    - npm install redis (Node.js)
    - pip install redis (Python)
    - <dependency>jedis</dependency> (Java)
- Cache Client = The object you create from that library
    - const client = redis.createClient()
    - client = redis.Redis()
    - Jedis jedis = new Jedis()


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
- This is when low-value or one-time-use data fills up the cache and evicts hot data (frequently used items).
- So imagine a scenario when a large data enters the cache and it's going to be used just for once. Just to put this data into cache, a lot of other items must have been evicted which were frequently used. This reduces hit ratio.
- Example: A nightly ETL job scans through 10GB of historical data (used once), evicting 1GB of user session data (accessed thousands of times daily).


## How Modern Caches Handle Randomness

- If we cache everytime a cache miss occurs, it may evict a hot item. And this newly cache may not be used that frequently at all. This reduces the hit ratio.
- "Not every item deserves to go into cache".

Solutions:
### 1. Admission Policies (Don't Cache Everything)

- Admission policies decide whether a new item **should** enter the cache when there's limited space. They act as gatekeepers to prevent cache pollution.

#### Common Admission Policy Types:
1. Threshold-Based Admission 
- Only admit items that have been accessed X times
- Example: "Only cache items accessed ≥ 3 times"
- Pros: Prevents one-hit wonders, simple to implement
- Cons: May miss items that become hot after first access

2. Probabilistic Admission
- Admit items with some probability (e.g., 50% chance)
- Can be based on cache load, item size, etc.
- Pros: Simple, provides natural filtering
- Cons: May randomly reject useful items
  
3. Size-Based Admission
- Reject items larger than a certain size
- Example: "Don't cache objects > 1MB"
- Pros: Prevents large items from dominating cache
- Cons: Size doesn't correlate with utility

4. Frequency-Based Admission (like TinyLFU  used in Caffeine)
- Compare item frequencies before admitting
- Only admit if new item is "more worthy" than victim
- Pros: Makes optimal trade-offs
- Cons: Requires frequency tracking

### 2. Eviction Policies Beyond LRU
- LRU (Least Recently Used) alone is bad for random workloads.
- LFU is better in this case. It keeps frequently accessed items longer.



## How to Prevent Thrashing

### 1. Right-Sized Cache

### 2. Better eviction policies

### 3. Segregation
Not all requests are equal. Some are hot traffic (frequently used) and some are cold (rarely requested).

1. Hot Traffic - Interactive queries
    - Example: User profile lookups, shopping cart items.
    - Small set of items accessed repeatedly.
    - Perfect candidates for caching.

2. Cold Traffic - Batch jobs / scans
    - Example: Analytics queries, ETL pipelines, recommendation model training.
    - Access a huge amount of data once or twice.
    - Poor candidates for caching.
    - If they share the same cache → they evict hot items used by real users.



- This mixing leads to cache pollution → hot interactive data gets pushed out by cold batch data → thrashing.
- So the idea is to keep the two types of data in separate caches.
- This could be achieved by:
  - Physically separate caches - one cache cluster for hot data and other cluster for cold data.
  - Logically separate caches - Same cache cluster, but partition memory into regions: 70% memory for interactive traffic (hot) and 30% for batch jobs (cold)
- Industry Examples
    - Netflix: Separate caches for user data vs analytics
    - Facebook: Different cache tiers for different content types







