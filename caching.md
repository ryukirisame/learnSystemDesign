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
2. Reduced load on backend systems: Caching reduces the number of requests that need to be processed by the backend, freeing up resources for other operations.
3. Increased Scalability: Caches help in handling a large number of read requests, making the system more scalable.
4. Cost Efficiency:
   - Fewer database queries or API calls → lower compute and bandwidth costs.
   - Example: Instead of paying for every call to an external paid API, results can be cached for reuse.
5. Resilience and Availability:
  - If the database or API is temporarily down, cached data can still serve requests.
  - Example: A social media app can still show cached timelines even if the backend is experiencing issues.
6. Enhanced User Experience: Faster response times lead to a better user experience, particularly for web and mobile applications.

# Benefits
1. Reduced latency
2. Cuts down the bandwidth
3. Reduced load on the server
      

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

# Cache Penetration
- A client requests for nonexistent data. Every request will miss from cache and hit the DB.

## Solutions
- Null Caching: If DB returns null data, cache null with short TTL.
- Bloom Filters.

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

<img width="777" height="256" alt="image" src="https://github.com/user-attachments/assets/d6b89afc-a05f-499b-9565-439331e641da" />

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

- Write Back doesn't require invalidation of cache entries, as the cache itself is the source of truth during the write process.
- Write Back caching is ideal for write-heavy scenarios where write operations need to be fast and frequent, but immediate consistency with the database is not critical, such as logging systems and social media feeds.
- Example:  Imagine a collaborative document editing application that allows multiple users to make changes to a document simultaneously. When users make changes, those changes are first saved to the cache, allowing the application to respond quickly and provide a smooth editing experience. When certain conditions are met (e.g., the number of changes reaches a certain threshold), the application writes the cached changes back to the data store, updating the document with the latest changes from all users. This approach minimizes the number of write operations to the data store and reduces the load on the storage system, improving the overall performance of the application.


<img width="1456" height="860" alt="image" src="https://github.com/user-attachments/assets/10532688-6510-4af4-a13a-3ad86a02c782" />



# Cache Invalidation Strategies

- Cache invalidation is the process of removing or updating outdated data from a cache to ensure that only the most recent and accurate information is stored.
- It ensures cache consitency.

## 1. Time-to-Live (TTL) / Expiry

- Each cache entry has a fixed expiration time.
- After TTL expires, the entry is automatically removed or marked stale.
- When a request is received for the content, the cache checks the time-to-live value and serves the cached content only if the value hasn’t expired. If the value has expired, the cache fetches the latest version of the content from the origin server and caches it.
- Cons: Can serve stale data until TTL expires.
- If TTL is too short → more DB hits. If too long → stale data risk.

## 2. Manual/Programmatic Invalidation (Explicit)

- In this approach, the code responsible for data modification is also tasked with invalidating the corresponding cache. This can be accomplished by issuing an invalidation command whenever data is altered.
```
DB.update(user=123, name="Alice")
Cache.delete(user=123)
```
- Pros: Data freshness guaranteed.
- Cons: If the programmer forgets to invalidate the cache, the stale data will be served.
- Example: Imagine an e-commerce application. When an administrator updates a product’s price, the price update code also issues a cache invalidation command for that specific product.
  
## 3. Write-Through with Invalidation

- Since, in write-through the write operation is done on cache as well as DB synchronously, so cache always contains the latest data -> No invalidation needed.
- However, if another system updates the DB directly (bypassing the cache), then cache can become stale and will need invalidation. So, this strategy only works if all writes go through cache layer.

## 4. Write-Behind (Write-Back) with Invalidation
- Since, in write-behind strategy, data is always written on cache (and DB is updated asynchronously in the background), there is no need of invalidation because cache acts as the source of truth and always has the latest data.

## 5. Event-Based Invalidation

- When a significant event occurs, such as a data update, a system can dispatch a message or signal to the cache, indicating that certain data has been modified. The cache can then proceed to invalidate the relevant data. This approach requires a messaging/event infrastructure.
- In a real-time chat application, when a user sends a message, the messaging server sends an event to all participants in the conversation, signaling an update. This triggers the cache to be invalidated for that specific conversation.
- Needs pub/sub infra (kafka, Redis Streams, RabbitMQ).
- Event delivery must be reliable (otherwise stale data persists).
- This is often used in distributed systems where multiple cache nodes must stay in sync (multi-region apps, CDNs).

## 6. Lazy Invalidation (Stale-While-Revalidate)

- In this strategy, the cache isn’t immediately invalidated after an update. Instead, it’s marked as “invalid,” and new data is fetched only when someone attempts to access the invalid data.
- Allows serving slightly stale data while refreshing in the background.
- Example:
    -  TTL expired at t=600s.
    -  User request at t=601s → stale entry served immediately.
    -  Background thread refreshes entry from DB.
- Pros: Low latency, no cache miss penalty. Also, good for data that can tolerate slightly stale data. 
- Cons: Not suitable for critical data (like balances)


# Difference between cache invalidation and eviction

- In cache invalidation, cache is removed **or** marked as invalid when the data in the DB changes. Its purpose is to ensure that cache does not serve stale (outdated) data. 
- In cache eviction, data is removed form cache due to memory constraints, not because it's outdated. Its purpose is to manage/optimize cache storage for frequently used data.


# Cache Eviction/Replacement Policies

https://blog.algomaster.io/p/7-cache-eviction-strategies


# Cache Performance Metrics

### 1. **Hit Rate**

* **Definition**: The percentage of requests served from cache (instead of going to the database/backend).

* **Formula**:

  $$
  \text{Hit Rate} = \frac{\text{Cache Hits}}{\text{Total Requests}}
  $$

* **Example**:

  * 1000 total requests
  * 800 served from cache
  * Hit rate = 80%

* **Why Important?**

  * Higher hit rate → better cache utilization.
  * Low hit rate → cache is underperforming (wrong data, wrong eviction policy, too small cache size).

### 2. **Miss Rate**

* **Definition**: The percentage of requests **not** found in cache and served from backend.

* **Formula**:

 ```
Miss Rate=1−Hit Rate
```

* **Example**:
  If hit rate is 80%, miss rate = 20%.

* **Why Important?**

  * A high miss rate means the cache isn’t helping much. Maybe you’re caching the wrong things.



### 3. **Eviction Rate**

* **Definition**: How often items are evicted from the cache (because of TTL expiry or memory pressure).

* **Why Important?**

  * A high eviction rate might mean cache size is too small.
  * If popular keys are evicted too frequently, hit rate will drop.

* **Example**:

  * Cache size = 1000 keys
  * You insert 10,000 keys in a short time
  * Eviction rate will be high (depends on policy like LRU/FIFO).



### 4. **Latency (Cache Access Time)**

* **Definition**: Time to fetch from cache.

  * Memory cache (e.g., in-process) → nanoseconds to microseconds
  * Distributed cache (e.g., Redis over network) → microseconds to milliseconds

* **Why Important?**

  * Cache should be much faster than DB.
  * If cache latency is close to DB latency → defeats the purpose.



### 5. **Staleness / Freshness**

* **Definition**: How up-to-date cached data is compared to the source of truth (DB).

* **Why Important?**

  * High staleness → users see outdated data.
  * Too aggressive invalidation → defeats purpose of caching.
  * Balance freshness vs performance.

* **Example**:

  * If product price changes in DB but cache shows old price for 30s, that’s staleness.



### 6. **Throughput**

So basically, the cache system is also a server. Hence, it has a limit on how much it can serve at a time. So, the number of requests/sec the cache server can serve is called throughput.
  
A cache system is a server (like Redis, Memcached, or even a CDN edge server). Just like any server:
-  It has CPU limits (processing requests).
-  It has memory or disk I/O limits (storing/reading cached objects).
-  It has network limits (bandwidth).
Because of those limits, it can only serve **X requests per second** reliably.
That “X” is what we call throughput.


Think of it like:
- Latency = How fast it answers one request.
- Throughput = How many requests it can answer per second.


For example:
- A Redis cache might respond in 0.5 ms per request (low latency).
- But if you send 1 million requests/sec, it will choke because throughput capacity might be only 300k requests/sec.

#### What affects throughput?

- Cache type
    - Memory caches (Redis, Memcached) → very high throughput (hundreds of thousands ops/sec).
    - Disk caches (SSD-based, CDN edge caches) → lower throughput due to I/O.
- Data size per request (large objects take more time).
- Network bandwidth (especially for CDN throughput).
- Concurrency (single-threaded Redis vs. sharded Redis cluster).


### 7. **Byte Hit Ratio**

* Cache Hit Ratio only tells you the percentage of requests served from cache.
* Byte Hit Ratio tells you the percentage of data volume (bytes) served from cache.
* In other words:
    *  Hit ratio answers: “How often do I avoid going to the origin?”
    *  Byte hit ratio answers: “How much data (traffic) do I save from going to the origin?”
* Why is this needed?
    * Not all requests are equal in size.
    * A 1 KB CSS file and a 10 MB video file are both single requests.
    * If your cache serves the 1 KB file but not the 10 MB video, your hit ratio might look good, but in terms of bandwidth saved, it’s almost useless.
    * That’s why byte hit ratio is often a more realistic measure of caching effectiveness.
* Formula
    * <img width="500"  alt="image" src="https://github.com/user-attachments/assets/ca59dc4d-2294-4dce-a382-200829c8408d" />
* Example
     * Suppose in one minute you get these requests:
         * `style.css` → 1 KB (cached) ✅
         *  `logo.png` → 100 KB (cached) ✅
         *  `video.mp4` → 100 MB (not cached, from origin) ❌
    * Hit Ratio
        * Total requests = 3
        * Cache hits = 2
        * Hit Ratio = 2/3 = 66.7%
    * Byte Hit Ratio
        * Bytes served from cache = 1 KB + 100 KB = 101 KB
        * Total bytes requested = 1 KB + 100 KB + 100 MB ≈ 100,101 KB
        * ByteHitRatio = (101/100101)*100 ~ 0.1%
    * So, even though your hit ratio is 67%, your cache is basically saving nothing in terms of bandwidth.
* Where is this useful?
    *  CDNs (Content Delivery Networks) → They optimize for byte hit ratio since video/image traffic dominates.
    *  Large-scale systems with heavy media → Byte hit ratio is more important than simple hit ratio.
    *  Cost savings → Bandwidth savings = $$ saved in cloud infra.

### 8. **Cost per Hit**

* "Cost per hit" measures how much it costs (in infrastructure/resources) to serve one cache hit.
* Unlike hit ratio or byte hit ratio (which are effectiveness metrics), cost per hit is an efficiency metric — it tells you whether the gains of caching are worth the resources spent.
* Why do we need this?
    * Caching isn’t free:
        * Maintaining RAM caches (Redis, Memcached) is expensive.
        * SSD caches cost less but have slower lookups.
        * More cache servers = higher cost.   
    * A system with 90% hit ratio sounds great… but if you’re spending $10,000/month on caching to save only $8,000/month of origin cost, it’s a net loss.
    * You must ensure the cost per hit < cost of DB hit saved.
* Trade-offs: Adding more cache nodes improves hit ratio but also increases cost per hit.
* Business justification: Is caching saving us money or wasting it?
#### Example

Let’s say you run a video streaming platform:
- Cache cluster costs: $5,000/month (Redis + infra)
- Cache serves: 500M hits/month
- <img width="400"  alt="image" src="https://github.com/user-attachments/assets/4454b0e2-8e10-4559-bc99-21ae8f1a219a" />
- That’s 1 cent per 1,000 hits → super efficient ✅


But suppose you misconfigured and most large videos bypass cache, leaving only small assets in cache:
- Still $5,000/month cost
- Only 50M hits/month served
- <img width="400"  alt="image" src="https://github.com/user-attachments/assets/b0c5eb0a-7360-48b6-9f3f-3c2fb1e5a440" />
- Now each hit costs 10x more, and caching may no longer be justified unless it’s giving latency benefits.



##  Example Flow with Metrics

Let’s say you’re building **an e-commerce product catalog**:

* Cache = Redis
* Backend = MySQL

### Scenario:

* 10,000 requests/min
* 7,500 served from Redis (hits)
* 2,500 go to MySQL (misses)

Metrics:

* **Hit Rate** = 75%
* **Miss Rate** = 25%
* Average Redis latency = 1ms
* Average MySQL latency = 40ms
* Effective latency (weighted avg) = 0.75 × 1 + 0.25 × 40 = \~10.25ms

👉 With cache, avg response time is 10ms.
👉 Without cache, avg response time would be 40ms.
👉 Cache saved \~75% load on DB.


# Difference between CDN and Caching

- Caching is a general concept of storing data temporarily for faster future access. It can exist at many layers: application cache, database cache, browser cache, OS cache.
- While, CDN is a specialized implemenation of caching. It is a distributed cache with the specific goal of reducing latency by bringing web content physically closer to users.
- All CDNs use caching, but not all caches are CDNs.

# HTTP Caching (Web Caching)
HTTP caching is about storing HTTP responses (HTML, CSS, JS, images, API responses, etc.) so that future requests can be served faster without always hitting the origin server.

#### Responses can be cached at the following possible locations: 
1. Browser Cache → Lives on the client machine. Private to the user.
2. Proxy Server
3. Reverse Proxy Server
4. CDNs

- The closer the client and cache are, the faster the response will be. The most typical example is when the browser itself stores a cache for browser requests.

## Types of HTTP Cache

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/779a60ac-3bb5-4abc-ab7c-2a362bd4720d" />

1. Private - Accessible just by the client. Used to store sensitive/personalized information. Browsers cache are private.
2. Shared - Accessible by many users. 
    1. Proxy Caches 
    2. Managed Caches
   
HTTP cache headers are more like **suggestions/directives** than **commands**. Any cache can choose to ignore them. So, even if the server sends the `cache-control` headers, its upto the receiver, if they want to respect the header or not. 

So basically, it makes the shared caches divided into two categories: 
1. Managed Caches: almost always respect headers (because you can control them and can configure them)
2. Proxy Caches: typically DO respect headers too (they're "good citizens"), but **may** choose not to, as they are not under our(application owners) control.

The cache that we(the application owner) can control and configure are Managed Cache. For example: CDNs like cloudflare. **WE** configure Cloudflare's cache rules through their dashboard. We can set custom rules like "cache all images for 30 days" even if our server sends different headers. Everything is under our control (the server/service owner)

As for the caches who are not under our control, they are free to choose to override their own behavior. For example: proxy servers, ISP caches. They could ignore the cache headers and can enforce their own caching policies.

   

## Caching headers 
Whenever the server responds to the client, it sends the http headers along with the response. 

The caching behaviors of browsers and shared caches is controlled by the following headers:
1. `Expires`
2. `Pragma`
3. `Cache-Control`

- `Expires` and `Pragma` existed before HTTP 1.1 and they are not used as much. They are used here and there for backward compatibility.
- `Cache-Control` was introduced in HTTP 1.1 and it is the preferred header for caching.

### 1. `Expires`
- Syntax: `Expires: <day-name>, <day> <month> <year> <hour>:<minute>:<second> GMT`
- The HTTP Expires response header contains the absolute date/time after which the response is considered expired.
- Note: If there is a `Cache-Control` header with the `max-age` or `s-maxage` directive in the response, the `Expires` header is ignored.
- Clocks have to be in sync.
- Can't be more than a year.
- If you provide wrong date format, or invalid value, the response will be considered stale. So, this directive is very error-prone.

### 2. `Pragma`
- It has just one possible value: `no-cache`
- Example: `Pragma: no-cache`
- To prevent caching.
- `Cache-Control` should be preferred over this.

### 3. `Cache-Control`
- It's a multi-valued header. So, it can have multiple values/directives and they determine the caching behavior.

#### Possible values/directives for `Cache-Control` header:
1. `Cache-Control: Private`
- It means the content can be cached by private cache (browser) ONLY.
- The response will only be cached in the client/browser.

2. `Cache-Control: Public`
- It means the content can be cached by ANY cache: private and shared
- Content can be cached at any location: Browser, Proxy, Reverse Proxy, CDNs, ISP etc.

3. `Cache-Control: no-store`
- It means "Don't cache at all (for sensitive data)".
- So, everytime client makes a request for a resource, the resource will be served from the origin server directly.

4. `Cache-Control: no-cache`
- It means the content can be cached, but for the client to re-use it, it must first re-validate from the server.
- Re-validation is done using `E-Tag` header.


5. `Cache-Control: max-age=<seconds>`
- The content can be cached for the given number of seconds.
- Applies to both: Private and Shared cache.
- Example: `Cache-Control: max-age=3600, public` It means the value can be cached for 3600 seconds at any public cache.

6. `Cache-Control: s-max-age=<seconds>`
- The prefix 's' stands for shared.
- It is the same as `max-age`, but gives a caching duration for 'shared' caches.
- In case of `Cache-Control: max-age=600, s-max-age=3600` it means private caches can cache the content for 600s and shared caches can cache the content for 3600s.

7. `Cache-Control: max-age=<seconds>, must-revalidate`
- This response directive indicates that the response can be stored in caches and can be reused while fresh. If the response becomes stale, it must be validated with the origin server before reuse.
- Typically, must-revalidate is used with max-age.
- This directive prevents serving stale content: HTTP allows caches to reuse stale responses when they are disconnected from the origin server. `must-revalidate` is a way to prevent this from happening - either the stored response is revalidated with the origin server or a 504 (Gateway Timeout) response is generated.
- Applies to all: Private and Public caches

8. `Cache-Control: max-age=<seconds>, proxy-revalidate`
- Same as `must-revalidate` but specifically for shared caches.
- Example: `Cache-Control: max-age=600, proxy-revalidate` : Since `must-revalidate` is not present, the client is allowed to serve stale content if the server is not reachable. But, the proxy can't. Because, proxy must revalidate.

#### We can mix directives to achieve different caching behaviors
`Cache-Control: max-age=3600, s-max-age=600, public, must-revalidate`

### Validator Headers 

https://github.com/ryukirisame/learnSystemDesign/blob/main/CDN.md#how-to-optimize-re-fetch-in-pull-cdns

- These headers are identifiers that help determine whether a cached resource is still the same as the version on the server. They enable conditional requests to check if content has changed without downloading the entire resource.
- The server sends one or more validation headers in the response, which are used by the client to make a conditional request to the server.

#### Headers: 
1. `ETag` 
2. `If-None-Match`
3. `Last-Modified`
4. `If-Modified-Since`

#### What if `ETag` or `Last-Modified` headers are not present in server response? 
That means no conditional requests can be made. So, the server will always respond with the content when asked about a resource.


## Heuristic Caching
When the server do not provide caching headers in the response, the cache falls back to a method called **heuristic caching** in which the cache tries to **guess** "for how long the content should be cached based on whatever information is available".

### Heuristic Caching Behavior

**The 10% Rule** - This is the most common heuristic. If a `Last-Modified` header is present, browsers typically cache the resource for about 10% of its age:
- If a file was last modified 100 days ago, cache it for ~10 days
- If modified 1 day ago, cache for ~2.4 hours

**Without Last-Modified** - Behavior becomes much less predictable and varies significantly between browsers:
- Some browsers might cache for a very short time (minutes)
- Others might not cache at all
- Some may cache based on content type (images vs HTML vs CSS)

### Browser Variations

Different browsers implement different heuristics:
- **Chrome/Edge** - Generally follow the 10% rule when Last-Modified exists
- **Firefox** - Similar but with different fallback behaviors
- **Safari** - Has its own heuristic algorithms















