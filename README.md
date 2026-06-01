# Resources
1. https://github.com/ashishps1/awesome-system-design-resources?tab=readme-ov-file
2. https://github.com/donnemartin/system-design-primer?tab=readme-ov-file#content-delivery-network




# Roadmap 1:
## Phase 1: The Absolute Foundations

Before building massive distributed networks, you need to understand how data moves and how basic servers handle load.

* **Vertical vs. Horizontal Scaling:** * *Vertical (Scaling Up):* Adding more RAM/CPU to an existing machine (has a hard limit).
* *Horizontal (Scaling Out):* Adding more machines to your pool (the backbone of modern system design).


* **The CAP Theorem:** The ultimate rule of distributed systems. In the event of a network partition (**P**), you must choose between Consistency (**C**) or Availability (**A**). You cannot have both.
* **PACELC Theorem:** An extension of CAP. If there is a **P**artition, choose **A**vailability or **C**onsistency; **E**lse, choose **L**atency or **C**onsistency.
* **Load Balancing:** Distributing incoming network traffic across multiple servers.
* *Algorithms:* Round Robin, Least Connections, IP Hash.
* *Layers:* Layer 4 (Transport layer, TCP/UDP) vs. Layer 7 (Application layer, HTTP/HTTPS).

## Phase 2: Client-Server Communication & APIs

How do different parts of your system talk to each other?

* **API Protocols & Styles:**
* **REST:** Stateless, resource-oriented, uses standard HTTP methods. Great for standard CRUD.
* **GraphQL:** Client specifies exactly what data it needs. Solves over-fetching/under-fetching.
* **gRPC:** High-performance, low-latency framework using Protocol Buffers over HTTP/2. Ideal for microservice-to-microservice communication.


* **Real-Time Communication:**
* **WebSockets:** Full-duplex, persistent connection for real-time data (e.g., chat apps).
* **Server-Sent Events (SSE):** One-way real-time streaming from server to client.
* **Long Polling:** The client repeatedly requests data from the server, holding the connection open until new data is available.

## Phase 3: Data Storage & Management

Choosing the right database and data layer layout is often the most critical decision in a system design interview or project.

### 1. Relational (SQL) vs. Non-Relational (NoSQL)

* **SQL (e.g., PostgreSQL, MySQL):** Strict schema, ACID compliance, excellent for complex joins and transactional integrity.
* **NoSQL:**
* *Key-Value (Redis, DynamoDB):* Fast caching, session storage.
* *Document (MongoDB):* Flexible schema, JSON-like storage.
* *Wide-Column (Cassandra, ScyllaDB):* Massive write throughput, great for time-series or analytics.
* *Graph (Neo4j):* Excellent for highly interconnected data (social networks, fraud detection).

### 2. Scaling the Data Layer

* **Replication:** Copying data across multiple nodes (Leader-Follower or Leaderless) to ensure high availability and read scalability.
* **Sharding (Horizontal Partitioning):** Breaking a large database into smaller, faster, more manageable pieces (shards) based on a shard key.
* **Indexes:** How databases speed up read queries (B-Trees, LSM Trees), and the trade-off of slower writes.

## Phase 4: Performance & Scalability Design Patterns

How do you make a system fast and resilient when millions of users hit it at once?

* **Caching:** Reducing database load by storing frequently accessed data in memory.
* *Strategies:* Cache-Aside, Write-Through, Write-Behind.
* *Eviction Policies:* LRU (Least Recently Used), LFU (Least Frequently Used).


* **Asynchronous Processing & Message Queues:** Decoupling heavy tasks from the main application flow.
* *Tools:* RabbitMQ, Apache Kafka, AWS SQS.
* *Concepts:* Publish/Subscribe pattern, event-driven architecture, guaranteeing "at-least-once" vs. "exactly-once" delivery.


* **CDN (Content Delivery Network):** Geographically distributed groups of servers that cache static content (images, video, HTML) closer to the user to reduce latency.


## Phase 5: Advanced Distributed Systems Concepts

When you reach massive scale, normal networking rules break down. You need to handle distributed state.

* **Consistent Hashing:** A technique used to distribute requests across a dynamic set of servers (like routers or cache nodes) minimizing re-mapping when nodes are added or removed.
* **Distributed Transactions:** Managing data consistency across multiple databases or microservices.
* *Patterns:* Two-Phase Commit (2PC), Saga Pattern (orchestration vs. choreography).


* **Rate Limiting:** Protecting your servers from being overwhelmed or subjected to DDoS attacks.
* *Algorithms:* Token Bucket, Leaky Bucket, Fixed Window, Sliding Window.


## Phase 6: Observability, Security, & Resilience

Building it isn't enough; you have to keep it alive and secure.

* **Resilience Patterns:**
* *Circuit Breaker:* Automatically stopping requests to a failing service to give it time to recover.
* *Bulkhead:* Isolating resources (like thread pools) so a failure in one area doesn't bring down the entire system.


* **Observability:**
* *Metrics:* CPU, memory, error rates, request latency (p99, p95, p50).
* *Logging & Distributed Tracing:* Tracking a single user request as it travels through dozens of microservices (e.g., Jaeger, Zipkin).


* **Security:** Rate limiting, SSL/TLS encryption in transit, encryption at rest, and OAuth2/JWT for authentication.



Are you preparing for a specific system design interview, or are you looking to apply this directly to a project you're currently building?
