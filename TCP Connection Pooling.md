# TCP Connection Pooling

- Let's suppose that we have our application server which talks to another fast server or database or redis. Each time a request comes, the application server talks to the fast server via opening a new TCP connection. This opening (and closing) of TCP connection becomes a bottleneck at a large scale. Even though Redis is fast, our DB is fast, the network call to them becomes an area of bottleneck. 

- How does TCP connection work?
  - We have a 3-way handshake first before any bit of data can be sent. This 3-Way handshake takes time.
  - If the database or server uses encrypted connection (HTTPS/TLS), then on top of the 3-way handshake, it requires TLS handshake as well.  

## Optimization
- If we want to optimize the network call, we can use TCP connection pooling.
- Instead of establishing a TCP connection for each and every request, which includes the handshakes, we will keep a collection of pre-handshaked(already established) TCP connections cached into a pool.
- These connections are created once during startup (or on demand) and reused.
- Whenever a client request comes, it can ask the pool for one of the available connection. The pool will then assign one of the available connections to the request.
- The request gains control over the connection and starts transmitting data. Once the job is done, the request hands over the connection back to the pool.
- What we have essentially done is, we have eliminated the overhead of establishing and closing a TCP connection, hencing optimizing network calls to the db/server/redis etc.

## Things to watch out
- We need to tune the connection pool size properly:
  - If the pool size is too small, requests will queue up waiting for a connection.
  - If the pool size is too big, it will first of all consume a lot of memory on the application server side plus, it can overwhelm the db/redis.
  - Databases have a hard cap — Postgres defaults to `max_connections = 100`, Redis to `maxclients = 10,000`. If you have `20` pods of application server each with a pool size of `50` connections, that's `20 × 50 = 1000` connections to Postgres — postgres performance will drop. That's why we need a multiplexer infront of the postgres. // LETS RESEARCH ON THIS LATER. pgbouncer
- If a connection is idle for too long, it will be dropped by the router/firewalls.
  - To prevent that, the connection pool will send a keep-alive heartbeat signals or lightweight queries like `SELECT 1` periodically to the db/redis to ensure the cached connections don't get silently dropped by network firewalls while sitting idle.

## Questions
- How is a thread pool different from a connection pool?
  - Thread pool is a collection of process threads, they work with CPU.
  - Connection pool is a collection of already-established TCP connections, they are used for network calls, not for CPU processing.  
