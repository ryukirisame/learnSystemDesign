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

3. Early Revalidation / Soft TTL








