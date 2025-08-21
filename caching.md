1. What is Caching? What are its goals?
2. Why use caching?
3. Why caching is scalable, but database it not?
4. Types of caching (based on location)
5. Caching strategies
6. Cache Invalidation Strategies
7. Cache Eviction Policies
8. Cache Performance Metrices
9. Caching in Web (HTTP Caching)
10. Real-World examples
11. Benefits & Problems in caching
12. 

Caching is about trading freshness for speed & efficiency.
The hard parts are invalidating correctly, choosing what to cache, and handling edge cases under load.


Von Neumann Bottleneck: Caching helps overcome this bottleneck by serving data faster than traditional memory access routes.




# What is Caching? What are its goals?
- Caching is a technique used to temporarily store copies of data in high-speed storage layers (such as RAM) to reduce the time taken to access data.
- Its goal is to improve performance by reducing latency and expensive operations (like database queries, network calls, heavy calculations, etc).

# Why use caching?
1. Improved Performance: Retrieving data from cache is always faster than retrieving data from slower sources like disk, database, remote API. 
    - Imagine fetching user profile data:
      - From cache (Redis in-memory): ~0.2ms
      - From database (PostgreSQL, disk involved): ~20 ms
      - Difference: 100x faster.
2. Reduced loan on backend systems: Caching reduces the number of requests that need to be processed by the backend, freeing up resources for other operations.
3. Increased Scalability: Caches help in handling a large number of read requests, making the system more scalable.
4. Cost Efficiency:
   - Fewer database queries or API calls → lower compute and bandwidth costs.
   - Example: Instead of paying for every call to an external paid API, results can be cached for reuse.
5. Resilience and Availability:
  - If the database or API is temporarily down, cached data can still serve requests.
  - Example: A social media app can still show cached timelines even if the backend is experiencing issues.
6. Enhanced User Experience: Faster response times lead to a better user experience, particularly for web and mobile applications.
      

# Why caching is scalable but database is not?

In case of DBs, there are many steps involved between receiving a query to returning results: parsing, planning, executing, managing transactions, and often performing disk I/O, which is slow. As data grows in the database or number of concurrent requests increase, these become heavier, leading to slower responses. 

But in the case of Cache, its a fast memory and all data is stored as key-value store. So, no matter how big the cache storage is, the lookup will always be O(1). This makes retrieval extremely fast and predictable, even under high load.


# Types of caching (based on location)

1. In-Memory Cache
2. Distributed Cache
3. Client-Side Cache
4. Database Cache
5. CDN
6. Reverse Proxy Cache



## 1. In-Memory Cache

- An in-memory cache stores data directly in RAM (main memory) instead of on disk.
- Typically used for hot data (frequently accessed, high-performance needs).
- It is the fastest type of cache.
- Volatility: Data is lost if the process or machine restarts (unless backed up or persisted separately).
- Structure: Usually key–value store (hash map, dictionary, etc.).

### Types of In-Memory Cache
1. Local (In-Process) Cache
   - Data is stored inside the same memory space as the application process.
   - Examples - Java: Caffeine, HashMap
   - Fastest possible (no network calls, just memory lookup)
   - Risk of out-of-memory if cache grows too large.
   - Cache is private, per app.
2. Distributed In-Memory Cache
   - A separate service (cluster of cache servers) accessible over the network.
   - Shared by multiple application servers.
   - Examples:
        - Redis (supports persistence, advanced data structures).
        - Memcached (lightweight, simple key–value).
        - Hazelcast / Ehcache (for Java applications).
     
   - Scales horizontally (add more nodes, shard keys).
   - Large capacity (since memory is spread across servers).
   - Slightly slower than local cache (network call involved). 

## 2. Distributed Cache

- A distributed cache is a caching system that runs on multiple servers (a cluster) and stores data in RAM across those servers.
- Applications connect to this cache cluster over the network.
- All app instances share the same cache → ensures consistency across the system.
- Examples: Redis, Memcached, Hazelcast, Apache Ignite, Amazon ElastiCache
- Think of it as a shared memory space for the whole system, unlike local cache which is private per app instance.

### Architectures of Distributed Cache

#### Centralized Cache (Single Node)

- One cache server, all app servers connect to it.
- Simple but a single point of failure (SPOF).
- Used in small deployments.

#### Replicated Cache (Full Copy per Node)
- Every cache node has a full copy of the cache data.
- High availability but memory overhead (data duplicated everywhere).
- Works well for read-heavy workloads.

#### Partitioned / Sharded Cache

- Data is split across multiple nodes (each node stores only a portion).
- Scaling horizontally is easy → just add more nodes.
- Requires consistent hashing or similar algorithm to decide where data lives.
- Common in Redis Cluster, Memcached.

#### Hybrid (Partitioned + Replicated)

- Data is _sharded_ across nodes, but each shard has a replica for fault tolerance.
- Balances scalability and high availability.

## 3. Client-Side Cache

A client-side cache is when the end-user’s device (browser, mobile app, desktop app, etc.) stores data locally so that it doesn’t have to fetch it again from the server on every request.

## 4. Database Cache

A database cache is a layer (in-memory or distributed) that stores results of database queries or objects retrieved from DB so future requests don’t have to hit the database directly.
This improves latency, throughput, and scalability.

## 5. CDN


# Cache Stampede / Dogpile Effect / Thundering Herd Problem

A cache stampede happens when a popular cached item expires (or is missing), and then many concurrent requests hit the backend (DB, API, etc.) at the same time to recompute or reload that value.

- Instead of a single request repopulating the cache, hundreds or thousands may hit the database simultaneously.
- This negates the purpose of caching and can even cause the backend to crash under sudden load.

- We can have two scenarios:
  1. Multiple simultaneous requests hit the same cached key.
  2. Multiple simultaneous requests hit different cached key.

## Example Scenarios
1. Manual bulk warming / preloading
- Example: At app startup, you preload the top 1M product pages into the cache with a TTL=5 min.
- Since they’re all written at once, they will all expire at the same moment → stampede risk.

2. Hot keys with synchronized access
- Example: A trending product (product:123) gets cached at t=0 with TTL=300s.
- Millions of users hit it within that window.
- At t=300s, everyone sees a miss at the same instant.


## Why Does It Happen?

- Fixed TTL Expiry → When many clients request the same data right after it expires.
- Cold Cache / Cache Miss → First access after startup or eviction triggers many requests.
- Thundering Herd Problem → Multiple workers/threads waking up to recompute at the same time.

## Problem it causes
- Sudden spike in database load.
- Increased latency for users.
- Possible downtime if DB is overwhelmed.


## Solutions

1. Cache Locking (Mutex / Locking Mechanism)  (For scenario 1)
- When a cache miss occurs, only the first process/thread recomputes and repopulates the cache.
- Other concurrent requests either wait for the process to finish (and serve the newly cached value), or serve stale data.

2. Randomized Expiry (For scenario 2)
- Problem: If you set a fixed TTL (say 60s) on a popular cache key, all clients may get a cache miss at exactly the same second → everyone hits the DB together → stampede.
- Working:
  - Instead of all cache entries expiring at the same fixed time, you add a small random offset to each key’s TTL.
  - Example:
    - Normally: All items expire at 60s.
    - With randomized expiry: Each item expires at 60 ± random(0–10s) → some expire at 55s, some at 63s, some at 68s.
- Effect: Expirations are spread out over time, hence preventing simultaneous recomputations.


# Caching Strategies

There are several caching strategies, depending on what a system needs - whether the focus is on optimizing for **read-heavy** workloads, **write-heavy** operations, or ensuring **data consistency**.

## Read Through


<img width="777" height="246" alt="image" src="https://github.com/user-attachments/assets/789af15a-8d65-4b38-be7d-ba279428aa58" />

- In this strategy, Cache sits "in-between" the app and the DB.
- The application always asks the cache system for data.
- If its a cache hit, the cache system returns the data to application immediately. However, if its a cache miss, its the responsibility of the cache system itself to bring data from DB, store it in the cache and then serve it to the application.
- This approach simplifies application logic because the application does not need to handle the logic for fetching and updating the cache. The cache itself handles both reading from the database and storing the requested data automatically.
- Since this is a "pull method", so, only that data is stored in the cache that is actually requested by the application. So, it saves cache storage.
- Pros: For cache hits, Read Through provides low-latency data access.
- Cons: for cache misses, there is a potential delay while the cache queries the database and stores the data. This can result in higher latency during initial reads.
- To prevent the cache from serving stale data, a time-to-live (TTL) can be added to cached entries. TTL automatically expires the data after a specified duration, allowing it to be reloaded from the database when needed.
- Read Through caching is best suited for read-heavy applications where data is accessed frequently but updated less often, such as content delivery systems (CDNs), social media feeds, or user profiles.
- Example: Hazelcast

## Cache Aside / Lazy Loading

<img width="692" height="439" alt="image" src="https://github.com/user-attachments/assets/bb4f9a9b-efc4-439e-b9f7-759960793000" />

- In this strategy, its the responsibility of application itself (not the cache system) for deciding when to read from DB and when to update the cache.
- Cache sits "aside" the database - the app controls both: cache and DB.
- The application first checks the cache for data. If the data exists in cache (cache hit), it’s returned to the application.
- If the data isn't found in cache (cache miss), the application retrieves it from the database (or the primary data store), then loads it into the cache for subsequent requests.
- To avoid stale data, we can set a time-to-live (TTL) for cached data. Once the TTL expires, the data is automatically removed from the cache.
- Cache Aside is perfect for systems where the reads are frequent, writes are relatively infrequent. For example, in an e-commerce website, product data (like prices, descriptions, or stock status) is often read much more frequently than it's updated.
- Example: Memcached, Redis (in most common usage)


## Write Through

<img width="777" height="256" alt="image" src="https://github.com/user-attachments/assets/57973356-6805-4d60-bd41-6272ea352705" />

- In read-through, the cache automatically loads data from the database on a read miss.
- In write-through, the cache automatically writes data to the database whenever you update the cache.
- Your application only talks to the cache (never directly to the DB).
- The cache acts as a transparent proxy for both reads and writes.
- Every write operation is executed on both the cache and the database at the same time. This is a synchronous process, meaning both the cache and the database are updated as part of the same operation, ensuring that there is no delay in data propagation.
- This approach ensures that the cache and the database remain synchronized and the read requests from the cache will always return the latest data, avoiding the risk of serving stale data.
- In a Write Through caching strategy, cache expiration policies (such as TTL) are generally not necessary. However, if you are concerned about cache memory usage, you can implement a TTL policy to remove infrequently accessed data after a certain time period.
- Pros: This strategy ensures data consitency between the cache and the database.
- Cons: Writes are slower than plain DB writes (because every write has to go through cache + DB). Also, if DB is down, writes can fail.
- Write Through is ideal for consistency-critical systems, such as financial applications or online transaction processing systems, where the cache and database must always have the latest data.
  

## Write Around

<img width="665" height="468" alt="image" src="https://github.com/user-attachments/assets/19b3e969-9580-4faf-9f6c-989da37289eb" />

- Write Around is a caching strategy where data is written directly to the database, bypassing the cache.
- The cache is only updated when the data is requested later during a read operation, at which point the Cache Aside strategy is used to load the data into the cache.
- This approach ensures that only frequently accessed data resides in the cache, preventing it from being polluted by data that may not be accessed again soon.
- Writes are relatively faster because they only target the database and don’t incur the overhead of writing to the cache.
- TTL can be used to ensure that data does not remain in the cache indefinitely. Once the TTL expires, the data is removed from the cache, forcing the system to retrieve it from the database again if needed.
- Write Around caching is best used in write-heavy systems where data is frequently written or updated, but not immediately or frequently read such as logging systems.


## Write Back / Write Behind

- In Write-Back, when the application writes data:
    - It only writes to the cache.
    - The actual database update is deferred (asynchronously written “behind the scenes”).
### Example Flow:
- Step 1: Application writes
  - User updates their profile.
  - Instead of writing to the DB:
      - The update goes into the cache.
      - Cache marks this entry as dirty (needs DB sync).
      - Application instantly gets a “success” response.    
- Step 2: Later DB sync
    -  Cache has a background thread / write queue.
    -  After a certain time interval or batch size, it flushes dirty entries to the database.
- Step 3: Reads
    - Reads always go to cache.
    - Since writes are reflected in cache immediately, reads get the latest value (even before DB is updated).

### Pros

1. Low latency on writes (faster than Write-Through and Write-Around), as writes are completed quickly in the cache, and the database updates are delayed or batched.
2. Good for write-heavy workloads (batching reduces DB load).
3. High cache hit ratio → since all writes are in cache, reads are almost always served from cache.

### Cons

- There is a risk of data loss if the cache fails before the data has been written to the database.
- This can be mitigated by using persistent caching solutions like Redis with AOF (Append Only File), which logs every write operation to disk, ensuring data durability even if the cache crashes.


---



















