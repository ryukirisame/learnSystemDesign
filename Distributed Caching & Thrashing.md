
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
    - `npm install redis` (Node.js)
    - `pip install redis` (Python)
    - `<dependency>jedis</dependency>` (Java)
- Cache Client = The object you create from that library
    - `const client = redis.createClient()`
    - `client = redis.Redis()`
    - `Jedis jedis = new Jedis()`

#### 3. Replication
To make the system more reliable, some distributed caches replicate data across multiple nodes. If one node goes down, the data is still available on another.

#### 4. Sharding
Dataset is split into smaller and manageable pieces called shards. Each shard is stored on different cache node. It helps distribute the data evenly and allows the cache to scale horizontally.

# Dedicated Cache Servers vs Co-located Cache
Where should the cache live? That's what we are going to see now.

## 1. Dedicated Cache Servers (Global Cache)

<img width="700"  alt="image" src="https://github.com/user-attachments/assets/534bb746-4f26-47ce-8530-5e4ae1c68f52" />

- Dedicated cache servers are standalone machines or virtual instances used only for caching.
- They are separate from the application servers and are optimized for caching.
- So basically, there will be a cluster of nodes which will store only cache data.
- Since all application servers share the same cache cluster, they see the same cached values, avoiding inconsistencies across nodes.
- Compute and cache layers can be scaled independently, giving flexibility under high read workloads.
- If lets say Application Server A fails, the cache can still be accessible by other application servers unlike in the case of Co-located cache.
- Since a network call needs to be made to access the cache, so slighly slower than Co-located cache.
- High cost, since we need dedicated servers.
- Ex: Redis, Memcached

## 2. Co-located Cache (In-Memory Cache)

<img width="700"  alt="image" src="https://github.com/user-attachments/assets/f8233ea4-f981-417d-aa5d-0b786b7c5918" />

- Co-located cache means running the cache and the application on the same server.
- In this setup, the application and the cache share the same hardware resources, such as CPU, memory, and network interfaces.
- Since both the cache and the application are on the same server, accessing the cache is very quick, as no network call needs to be made. This is ideal for applications where every millisecond counts, like real-time gaming or high-frequency trading platforms.
- Co-locating the cache with the application can be more cost-effective, especially for small to medium-sized applications.
- Simple: No separate infra to manage.
- If Server A fails, Cache A will also be unavailable.
- Data sync problem: one server’s cache may be stale, others fresh.
- Scalability issues: We can only have same number of cache as the application servers. We cannot scale cache independently. If you need more cache without more application servers, you cannot do that.
- If a server restarts -> Cache also restarts.
- The cache and the application use the same server resources like CPU memory, and I/O. This can slow things down under high load, if both demand significant resources.
- Ex: Caffeine, Guava 

### Data Fragmentation Issue in Co-located Cache

- Example: E-commerce site with 3 app servers (A, B, C)
- Now users browse products:
    -  User 1’s request goes to Server A → it fetches Product 101 from DB → caches it locally.
    -  Later, User 2’s request goes to Server B → it also fetches Product 101 from DB → caches it locally.
    -  Then, User 3’s request goes to Server C → same story, Product 101 cached locally again.    
- End result:
    - Product 101 exists in 3 separate caches (A, B, C).
    - That’s 3× memory usage for the same data.
- With dedicated cache servers, this duplication wouldn’t happen: Product 101 would live in cache once, and all app servers would reuse it.

### Data Sync Problem in Co-located Cache

- Example: Social media site with 2 app servers (A, B)
- User accesses profile via Server A -> Server A caches: `user:123 → { name: "Alice" }`.
- User updates their name via Server B:
    -  Server B updates the DB: `name = "Alicia"`.
    -  Server B may also update its own cache → now it has `{ name: "Alicia" }`.
    -  But Server A’s cache still says `{ name: "Alice" }`.
- Next request comes to Server A:
    - Server A serves data from its local cache → user still sees `{ name: "Alice" }` (stale).
- End result:
    - Different servers return different answers for the same user.
    - Data is inconsistent across caches.
- So the core problem is: When the source of truth (DB) changes, how do we make sure all copies of that data in different caches are invalidated and updated.
- One solution is to use Sticky Sessions: A user always gets routed to the same Server. But this strategy has its own challenges like what if some users are more active than other? In that case one server will get more load than others.
- With dedicated cache servers, this wouldn’t happen:
    - The cache cluster stores an item once. (Not considering any replicas here)
    - All application servers see the same data. 


If you need your system to handle a lot of users, keep resources separate, and have the budget for it, using dedicated cache servers is likely the better option.

But if your app is smaller, you want to save money, or need it to be super fast, putting the cache and app on the same server can work well.


## 3. Hybrid Approach (Multi-level Cache)

- L1 (local, co-located) → fastest access.
- L2 (dedicated, distributed) → consistent, scalable source.
- Example: Netflix EVCache (local + Memcached).
- L1 avoids network hops for hot data.
- L2 ensures consistency & scalability.
- Local misses fall back to the distributed cache, reducing DB load.
- Reduces load on distributed cache.
- Complexity increases — cache invalidation, coherence, and memory management are harder.

# Data Distribution Strategies
### Consistent Hashing 
- The most common approach where keys are mapped to nodes using a hash function. 
- When nodes are added or removed, only a small portion of data needs to be redistributed. This minimizes cache invalidation during scaling operations.
  
### Hash-based Partitioning
- Simple modulo-based distribution where `node = hash(key) % number_of_nodes`.
- While easy to implement, adding or removing nodes requires significant data redistribution.

### Range-based Partitioning
- Data is distributed based on key ranges assigned to different nodes.
- This works well when you can predict access patterns but can lead to hotspots.


# How Does Distributed Caching Work?
To get data from the cache, application provides the key to the client library. The client library uses this key to find and query the node which has the data. If the data is present (a cache hit), it’s returned to the application. If not (a cache miss), the data is fetched from the primary data store (e.g., a database), and it can be cached for future use.

















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







