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
      

